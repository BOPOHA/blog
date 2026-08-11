---
layout: post
title: "scomp-agent: My tmux Sessions on the Phone, Without SSH and Without Open Ports"
date: 2026-08-11
tags: [scomp, tmux, screen, go, terminal, devops, security]
---

I start long jobs in tmux. Then I go away from the laptop. Then I want to know
if the job is finished. For many years, my procedure was this: find a laptop,
connect a VPN, open SSH, attach to tmux, and read one line. That is five steps
to read one line.

The phone is always in my pocket. The terminal session is always on a server.
I wrote **scomp** to connect these two things. This post is about
[scomp-agent](https://github.com/smart-bit-me/scomp-agent), the small Go binary
that runs next to your sessions.

## What the agent does

The agent runs on the machine that has your tmux or GNU Screen sessions. It does
these steps:

1. It finds the sessions that already exist.
2. It opens one outbound WebSocket connection to the relay,
   `wss://link.scomp.me/agent`.
3. It shows a link and a QR code.
4. You open the link on the phone, and you see your sessions.

The machine needs no inbound port, no public IP, and no SSH key on the phone.
The connection goes out from the machine, so a host in a private network works
too. Your firewall team will like this. Your firewall team does not like
anything, so this is a big result.

```sh
./bin/scomp            # connect, discover sessions, print the QR code
./bin/scomp --no-qr    # print only the link
```

The agent supports Linux and macOS, on amd64 and arm64.

## No polling, by design

The simple design asks tmux "what is new?" every second. That works, and it also
uses CPU for nothing, 86,400 times a day. The agent does not poll, and it never
did:

- **GNU Screen:** the agent watches the socket directory with filesystem events.
  A new socket is a new session.
- **tmux:** the agent attaches as a read-only control-mode client, and it
  ignores the window size. tmux sends events when sessions change.
- **PTYs** attach lazily. The agent opens a session only when a client asks for
  it.

Each session keeps a 1 MiB ring buffer of output. When you open a session on the
phone, you see the recent scrollback immediately. The agent also tracks the
DECCKM cursor-key mode, so the arrow keys work correctly in `vim` and `less`.
(Yes, I use vim on the phone. No, I cannot exit it either.)

## The safe default is read-only

A remote terminal is a powerful tool. A remote terminal that can type is a
remote shell with your permissions. For that reason, the agent has three input
modes, and the agent enforces them, not the relay:

| Mode | What the remote client can do |
| --- | --- |
| `readonly` (default) | Watch sessions. All remote input is dropped. |
| `approved-only` | Send only Enter, Tab, Escape, Ctrl+C, and `y`/`n`. |
| `full` | Type anything, like a local keyboard. |

```sh
scomp                     # readonly: monitor only
scomp --mode approved-only  # answer "Continue? [y/n]" from the phone
scomp --mode full         # full control: use this deliberately
```

An unknown mode value is an error. It does not fall back to `full`. I prefer a
program that refuses to start to a program that guesses.

`approved-only` is my favourite mode. A deploy script asks `Continue? [y/n]`
while I am in a queue for coffee. I press `y` on the phone. I do not get the
power to type `rm -rf /` by mistake. This is the 21st-century version of "press
any key to continue": now any key is not permitted.

## Identity and pairing

On the first start, the agent makes a random UID and an Ed25519 key in
`~/.config/scomp/`, with file mode `0600`. Each relay connection proves
possession of that key with a challenge-response handshake. Thus, a client that
knows only the public UID cannot impersonate a machine that is already pinned.

To pair a machine with your account, you type the PIN that the phone shows into
the agent process on the machine. You must be at the machine to do this. That
is the point.

Back up `~/.config/scomp/` if the machine identity must survive a migration.
Key rotation is manual at this time.

## What scomp is not

I want to be clear about the trust model. The relay sees the terminal traffic in
plaintext. It is not end-to-end encrypted, and the relay is part of the trusted
computing base. The [security policy](https://github.com/smart-bit-me/scomp-agent/blob/main/SECURITY.md)
says this in the first paragraph, because it is better to read it there than to
discover it later.

If you need end-to-end encryption for a terminal, use SSH. If you need to see
your build log from the bus, use scomp. Both are correct tools for different
problems.

## Try it

- Download a release:
  [github.com/smart-bit-me/scomp-agent/releases](https://github.com/smart-bit-me/scomp-agent/releases/latest)
- Read the documentation: [scomp.me/en/docs](https://scomp.me/en/docs/)
- See what people do with it: [scomp.me/en/use-cases](https://scomp.me/en/use-cases/)

The agent is free software under the AGPL-3.0. Pull requests are welcome.
Bug reports are welcome too, and they come with my standard first answer: "Did
you try to detach and attach again?"
