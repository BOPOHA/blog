---
layout: post
title: "Back to School for DevOps: When 65 GiB Free Is Not Enough"
date: 2026-09-20
tags: [devops, fedora, rhel, btrfs, electron, capacity-planning, rpm]
---

I have used Fedora on my primary machine long enough to see it grow up with
me. The rescue initramfs on this ThinkPad is dated **26 December 2020**—the
original Fedora 33 installation. The same installation is now running Fedora
44, with 7.2.x kernels in `/boot`.

That longevity is one reason I remain a RHEL/Fedora enthusiast. It also makes
the machine a useful reminder that infrastructure knowledge is not replaced by
writing YAML. A Senior DevOps role still needs filesystem, storage, packaging,
and failure-analysis skills. The declarative layer is valuable; it is not the
whole system.

## The incident

I was building Electron 41 as an RPM from a prepared Chromium/Electron source
artifact. This is not a small application build:

| Measurement | Value |
| --- | ---: |
| Prepared source artifact | 7.6 GiB compressed |
| Ninja work graph | about 45,437 actions |
| RPM build tree during the build | about 43 GiB |
| Filesystem | 476 GiB Btrfs `/home` |

The build had already compiled more than 20,000 Ninja actions when Clang
failed to open a dependency file:

```text
error opening '...blob_bytes_provider.o.d': No space left on device
```

The obvious check appeared to contradict the error:

```text
/home: 476 GiB total, 411 GiB used, 65 GiB available
```

This is the point where a superficial response—"delete a few logs and retry"—
is not enough. `df -h` reports filesystem-level free space. It does not
describe all of Btrfs's chunk-allocation state.

## Btrfs data space is not Btrfs metadata space

The useful command was:

```bash
sudo btrfs filesystem usage -T /home
```

It showed the real condition:

```text
Device allocated:   475.35 GiB
Device unallocated:   1.00 MiB

Metadata total:       8.01 GiB
Metadata used:        7.50 GiB
```

The device had almost no unallocated chunks left. Electron/Chromium creates an
enormous number of object files, generated sources, dependency files, and
ThinLTO cache entries. Btrfs needed another metadata chunk for those new file
entries, but could not allocate one from the device even though `df` still
reported 65 GiB free.

This is not an inode problem. On Btrfs, the familiar `df -i` output is often
not useful for this diagnosis. It is a chunk-allocation problem.

## Recovering space without destroying the build

The expensive part of the build was worth preserving. I did not remove the
RPM `BUILD` directory or restart `%prep`; doing so would discard tens of
thousands of completed Ninja actions.

First, I measured the system instead of deleting directories at random:

```bash
sudo journalctl --disk-usage
podman system df
du -xhd1 ~/.cache ~/.local/share ~/.config | sort -h
```

The significant findings were:

- `journald` retained about 3.8 GiB;
- Podman held about 49 GiB under `~/.local/share/containers`;
- caches under `~/.cache` consumed 38 GiB; and
- application profiles under `~/.config` were not automatically safe to
  delete—Lens, Freelens, Chrome, and Slack can contain real user state.

I bounded the journal, pruned genuinely unused Podman images only after
reviewing them, and removed selected disposable caches. Then I ran a targeted
Btrfs balance:

```bash
sudo btrfs balance start -dusage=10 -musage=50 /home
```

The balance relocated only 10 of 479 chunks. That small operation was enough
to return about 9 GiB to Btrfs as *unallocated* device space.

## The result

After cleanup and the targeted balance:

| Measurement | Before | After |
| --- | ---: | ---: |
| Free space reported by `df` | 65 GiB | 145 GiB |
| Btrfs data used | 380.38 GiB | 322.58 GiB |
| Btrfs metadata used | 7.50 GiB | 6.52 GiB |
| Device unallocated | 1 MiB | 8.97 GiB |
| Podman storage | about 49 GiB | 7.4 GiB |

The most important row is not the 145 GiB of ordinary free space. It is the
8.97 GiB of unallocated device space. Btrfs can now allocate new data or
metadata chunks, so the build can resume safely:

```bash
cd ~/rpmbuild/BUILD/electron41-41.10.0-build/electron41-source-41.10.0/src
ninja -C out/Release electron electron:electron_dist_zip electron:node_headers
```

Ninja resumes only incomplete work. The separate RPM packaging phase can then
run without `%prep`, which would otherwise delete the incremental build tree.

## Operational lessons

1. **Capacity is multidimensional.** Disk bytes, Btrfs chunk allocation,
   metadata headroom, inodes, quotas, and snapshot retention are different
   constraints.
2. **Measure before cleaning.** `podman system df`, `journalctl --disk-usage`,
   `du`, and `btrfs filesystem usage` answer different questions.
3. **Do not treat cache as universally disposable.** Browser and IDE caches
   usually are; application profiles, container volumes, SDKs, wallets, and
   Kubernetes credentials may not be.
4. **Preserve incremental work.** A failed large build is often recoverable.
   Fix the missing resource, then resume Ninja instead of restarting the
   pipeline.
5. **Balance Btrfs deliberately.** A whole-filesystem balance is not routine
   maintenance. Use a targeted balance after substantial deletion or when
   unallocated space approaches zero and metadata is under pressure.

For large local builds, I now check this before starting:

```bash
df -h /home
sudo btrfs filesystem usage -T /home
podman system df
```

The lesson is pleasantly old-fashioned: platform automation is only as good
as the operator's model of the platform underneath it. YAML does not make a
full Btrfs metadata chunk become writable.
