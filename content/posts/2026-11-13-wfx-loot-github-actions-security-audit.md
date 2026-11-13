---
layout: post
title: "wfx-loot: Auditing GitHub Actions Across a Personal Account and Multiple Organizations"
date: 2026-11-13
description: "A practical investigation of the wfx-loot CI secret-stealing campaign, with GitHub CLI scripts for repository discovery, incident triage, and verified permission changes."
tags: [security, github, github-actions, devops, incident-response]
draft: true
---

A compromised GitHub credential can turn a routine CI job into a credential
collection system. The attacker does not need an application vulnerability if
the stolen credential already permits changes to workflow files.

On November 12, 2026, Aikido published its investigation of **wfx-loot**, a
campaign that used stolen GitHub credentials to inject malicious workflows
across thousands of accounts. I used that report to review my public
repositories, then extended a preventive permission change to repositories
in multiple organizations.

The investigation and the permission change answer different questions:
**Was there evidence of an intrusion?** And **how much authority should a CI
job receive by default?** Keeping those questions separate makes the findings
more useful and prevents a hardening change from being mistaken for incident
remediation.

## What happened

According to Aikido's November 12 report, 4,837 compromised accounts opened
probe pull requests in 61,764 repositories, and at least 1,835 repositories
ran the payload. These are the report's figures at publication, rather than
an independently verified count or a live total.

The observed sequence began on November 10:

1. Open a PR titled `docs: dependency audit notes` from a `wfx-docs` branch
   to test write access.
2. Inject a shell payload into an existing workflow, introduce a temporary
   audit workflow, or add an npm installation hook.
3. Let automation execute the payload.
4. Revert the change, sometimes within a minute, and delete the workflow run.

That last step changes how the incident must be investigated. A clean workflow
on the default branch today does not establish that it was clean yesterday.
Likewise, an empty Actions history does not establish that nothing ran.

The payload collects workflow secrets, tokens, environment variables, and
credential files. The report also describes capabilities for exchanging OIDC
tokens, accessing cloud roles, and establishing persistence through cloud
identity changes. It reports HTTP exfiltration to two addresses:

```text
194.150.166[.]244:8080/exfil
18.143.168[.]96:8080/exfil
```

Not every capability was enabled in the observed deployment. In particular,
the absence of a `wfx-loot-*` artifact or a `WFX-BEGIN` step summary is weak
evidence: those delivery channels were disabled in the reported configuration.

The report lists these useful investigation leads:

| Evidence | Examples |
| --- | --- |
| Probe branch or PR | `wfx-docs`; `docs: dependency audit notes` |
| Commit messages | `ci: refresh build cache`; `chore: add ci helper`; `chore: add dependency audit workflow` |
| Cleanup messages | `ci: restore`; `ci: cleanup` |
| Temporary branches | `chore/` followed by eight lowercase letters or digits; `ci-check-` followed by four digits |
| Files | `.ci-*.sh`; `.github/workflows/build-audit-*.yml` |
| Payload configuration | `WF_CALLBACK`; `WF_CTOKEN`; `WF_SECRETS` |

Generic branch names, audit filenames, or cleanup messages are leads to review,
not proof of compromise. Inspect the content, author, timing, and execution
history together.

There is also an installation-path concern. Aikido identifies
`digitalelvis/neuronai-studio` version `3.14.3` and Go module
`github.com/onuragtas/docker-env` versions `v1.0.278` and `v1.0.280`.
For the Go versions, the reported payload is in workflow files; importing a Go
module does not execute those workflows. Exposure therefore depends on what
was installed or executed, rather than the package name alone.

## What I found in my repositories

The initial audit covered public repositories and their existing branches,
including forks and archived projects. It reviewed selected CI and
dependency files, recent reachable commits, historical commit diffs, PR
metadata, retained repository events, and Actions history.

I found **no matching campaign evidence in the inspected data**. That finding
has limits: deleted runs, removed branches, unreachable commits, and missing
event history can leave gaps. The review did not establish the security of
developer machines or cloud accounts.

Two missing workflow run numbers initially needed explanation. They predated
the November campaign, and I had deleted those older runs while cleaning up
build history. This is a useful example of why a numbering gap should prompt
an investigation rather than an immediate conclusion.

The permission review produced a concrete improvement. Across the repositories in scope, some needed a change and others already
had the desired settings. The change set
default workflow tokens to read-only and disabled Actions creating or
approving pull requests. Each changed repository was checked again afterward.

## Start with an explicit repository inventory

The commands below assume Bash, `gh`, and `jq`. Authenticate with the intended
account and confirm which account is active:

```bash
gh auth status
gh api user --jq .login
gh api 'user/orgs?per_page=100' --paginate --jq '.[].login'
```

Organization membership does not automatically grant access to every private
repository or permission to change settings. Your token may also need
organization SSO authorization. Investigate an access error; do not silently
count an inaccessible repository as checked.

Save the following as `github-actions-inventory.sh`.

```bash
#!/usr/bin/env bash
# Read-only. Personal public repositories + all visible organization repositories.
set -euo pipefail
EXCLUDE_ORG=${EXCLUDE_ORG:-example-excluded-org}
tmpdir=$(mktemp -d)
trap 'rm -rf "$tmpdir"' EXIT

login=$(gh api user --jq .login)
gh api "users/$login/repos?per_page=100" --paginate \
  --jq '.[] | select(.private == false) | .full_name' > "$tmpdir/repos"
gh api 'user/orgs?per_page=100' --paginate \
  --jq '.[].login' > "$tmpdir/orgs"

while IFS= read -r org; do
  [[ ${org,,} == ${EXCLUDE_ORG,,} ]] && continue
  gh api "orgs/$org/repos?type=all&per_page=100" --paginate \
    --jq '.[].full_name' >> "$tmpdir/repos"
done < "$tmpdir/orgs"

# Output only after every discovery request succeeded.
LC_ALL=C sort -u "$tmpdir/repos"
```
It lists the authenticated user's public repositories and every repository
visible to that credential in their organizations, including private
repositories and forks. It paginates the API requests and excludes one
organization by a case-insensitive name comparison.

```bash
# Set this deliberately. The exclusion is also honored by the other scripts.
export EXCLUDE_ORG=example-excluded-org

bash github-actions-inventory.sh > repos.txt
wc -l repos.txt
cat repos.txt
```

`example-excluded-org` is a fictional name. Replace it with the organization
you intend to exclude. Organization names and portfolio counts have been
omitted from this case study.

Review `repos.txt` before doing anything that writes to GitHub. Remove owners
or repositories outside your scope. The scripts use this saved list so that
the mutation step does not rediscover a different set of targets.

If you also need your personal private repositories, replace the personal
repository discovery command in the script with:

```bash
gh api 'user/repos?affiliation=owner&visibility=all&per_page=100' \
  --paginate --jq '.[].full_name'
```

Keep the resulting inventory and evidence directory private: repository names
and historical metadata can themselves be sensitive.

## Collect evidence before changing anything

Save the following as `github-actions-triage.sh`.

```bash
#!/usr/bin/env bash
# Read-only metadata triage. Does not execute repository code or fetch secret values.
set -euo pipefail
repos_file=${1:?Usage: bash github-actions-triage.sh repos.txt [output-dir]}
output_dir=${2:-github-actions-triage}
SINCE=${SINCE:-2026-10-01T00:00:00Z}
EXCLUDE_ORG=${EXCLUDE_ORG:-example-excluded-org}
mkdir -p "$output_dir"

while IFS= read -r repo; do
  [[ -z $repo ]] && continue
  owner=${repo%%/*}
  [[ ${owner,,} == ${EXCLUDE_ORG,,} ]] && continue
  [[ $repo =~ ^[A-Za-z0-9_.-]+/[A-Za-z0-9_.-]+$ ]] || exit 2
  dir="$output_dir/$repo"
  mkdir -p "$dir"
  printf '\nChecking %s\n' "$repo"
  gh api "repos/$repo/actions/permissions/workflow" > "$dir/permissions.json"
  gh api "repos/$repo/branches?per_page=100" --paginate --slurp \
    --jq 'add' > "$dir/branches.json"
  gh api "repos/$repo/pulls?state=all&per_page=100" --paginate --slurp \
    --jq 'add' > "$dir/pulls.json"
  gh api "repos/$repo/actions/workflows?per_page=100" --paginate --slurp \
    --jq '[.[].workflows[]]' > "$dir/workflows.json"
  gh api "repos/$repo/actions/runs?created=%3E%3D${SINCE}&per_page=100" \
    --paginate --slurp --jq '[.[].workflow_runs[]]' > "$dir/runs.json"
  gh api "repos/$repo/events?per_page=100" --paginate --slurp \
    --jq 'add' > "$dir/events.json"

  # Query every current branch, using its SHA to avoid URL-encoding branch names.
  : > "$dir/commits.jsonl"
  while IFS= read -r sha; do
    gh api "repos/$repo/commits?sha=$sha&since=$SINCE&per_page=100" \
      --paginate --jq '.[] | {sha, message: .commit.message}' \
      >> "$dir/commits.jsonl"
  done < <(jq -r '.[].commit.sha' "$dir/branches.json")

  jq -r '.[] | select(.name | test("^wfx-docs$|^chore/[a-z0-9]{8}$|^ci-check-[0-9]{4}$"; "i")) | "REVIEW BRANCH: " + .name' \
    "$dir/branches.json"
  jq -r '.[] | select(.title == "docs: dependency audit notes" or .head.ref == "wfx-docs") | "REVIEW PR: " + .html_url' \
    "$dir/pulls.json"
  jq -r 'select(.message | test("ci: refresh build cache|chore: add ci helper|chore: add dependency audit workflow|ci: restore|ci: cleanup"; "i")) | "REVIEW COMMIT: " + .sha + " " + .message' \
    "$dir/commits.jsonl"
  jq -r '.[] | select(.path | test("build-audit-|security-audit|ci-check"; "i")) | "REVIEW WORKFLOW: " + .path' \
    "$dir/workflows.json"
done < "$repos_file"

printf '\nMetadata saved in %s. No matches does not establish a clean account.\n' "$output_dir"
```
It collects current branches, open and closed PRs, registered workflows,
retained Actions runs, repository events, and commit messages since a chosen
date on every current branch. It also records current workflow permissions.

```bash
# Include the October activity discussed in Aikido's report.
export SINCE=2026-10-01T00:00:00Z
bash github-actions-triage.sh repos.txt evidence
```

The script prints `REVIEW` candidates and saves JSON for closer inspection.
It does not execute repository code, install dependencies, or fetch secret
values. **It is metadata triage, not a complete malware scanner.** No printed
matches means only that these particular metadata checks found no matches.
An API failure stops the script; treat that run as incomplete.

For a suspicious commit, inspect its changed files and diff:

```bash
repo=YOUR_ACCOUNT/YOUR_REPOSITORY
sha=COMMIT_SHA_TO_REVIEW

gh api "repos/$repo/commits/$sha" \
  --jq '{message: .commit.message, files: [.files[] | {filename, status, patch}]}'
```

GitHub can omit or truncate large patches. For a fuller review, fetch history
without checking out or executing the project:

```bash
gh repo clone "$repo" review.git -- --mirror

git --git-dir=review.git log --all --since=2026-10-01 \
  --format='%h %cI %s' -- .github/workflows package.json .vscode

# Inspect a chosen commit, including changes later reverted.
git --git-dir=review.git show "$sha" -- .github/workflows package.json .vscode
```

A mirror fetches advertised branches and tags and can require substantial
space for large repositories. It cannot recover every deleted, unreachable
commit. Read the files as evidence; do not run an installation hook or open an
untrusted project in an IDE that automatically executes workspace tasks.

Review current workflows and historical diffs for the callback addresses,
configuration markers, unexpected `toJSON(secrets)` exports, new hidden shell
scripts, and changed `postinstall` hooks. Also inspect non-default branches.
For the October overlap described in the report, review unexpected audit
workflows contacting `193.32.204[.]199` and hidden VS Code tasks that execute
`public/fonts/fa-solid-400.woff2` as JavaScript.

To review retained run numbering, group runs by workflow, never across the
whole repository:

```bash
jq 'group_by(.workflow_id) | map({
  workflow_id: .[0].workflow_id,
  path: .[0].path,
  runs: (sort_by(.run_number) | map({run_number, created_at, head_sha}))
})' "evidence/$repo/runs.json"
```

The chosen date window can hide earlier runs. GitHub also limits some filtered
run queries to 1,000 results; busy repositories need narrower date windows.
Repository events have bounded retention and coverage. Preserve suspicious
SHAs, PR URLs, diffs, and timestamps before deleting anything.

## Apply a smaller default token permission

For one repository, the proposed settings change is:

```diff
 {
-  "default_workflow_permissions": "write",
-  "can_approve_pull_request_reviews": true
+  "default_workflow_permissions": "read",
+  "can_approve_pull_request_reviews": false
 }
```

This is a GitHub repository setting, not a workflow YAML edit. Read-only here
means the repository contents and packages default described by GitHub; it
does not make the runner, network, or filesystem read-only.

Save the following as `github-actions-harden.sh`.

```bash
#!/usr/bin/env bash
# Dry run by default. --apply changes only the two repository settings below.
set -euo pipefail
repos_file=${1:?Usage: bash github-actions-harden.sh repos.txt [--apply]}
mode=${2:---dry-run}
[[ $mode == --dry-run || $mode == --apply ]] || exit 2
EXCLUDE_ORG=${EXCLUDE_ORG:-example-excluded-org}
LOG_FILE=${LOG_FILE:-github-actions-permissions.jsonl}
desired='{"default_workflow_permissions":"read","can_approve_pull_request_reviews":false}'

while IFS= read -r repo; do
  [[ -z $repo ]] && continue
  owner=${repo%%/*}
  [[ ${owner,,} == ${EXCLUDE_ORG,,} ]] && continue
  [[ $repo =~ ^[A-Za-z0-9_.-]+/[A-Za-z0-9_.-]+$ ]] || exit 2
  endpoint="repos/$repo/actions/permissions/workflow"
  before=$(gh api "$endpoint")
  jq -cn --arg repo "$repo" --arg mode "$mode" --argjson before "$before" \
    '{repo: $repo, mode: $mode, before: $before}' >> "$LOG_FILE"
  if jq -e '.default_workflow_permissions == "read" and .can_approve_pull_request_reviews == false' <<< "$before" > /dev/null; then
    printf '%s: already correct\n' "$repo"
    continue
  fi
  if [[ $mode == --dry-run ]]; then
    printf '%s: would set read-only default and disable PR approval\n' "$repo"
    continue
  fi
  gh api "$endpoint" --method PUT --input - <<< "$desired"
  # GitHub can briefly return stale values after a successful update.
  verified=false
  for attempt in 1 2 3 4 5; do
    after=$(gh api "$endpoint")
    if jq -e '.default_workflow_permissions == "read" and .can_approve_pull_request_reviews == false' <<< "$after" > /dev/null; then
      verified=true
      break
    fi
    sleep 2
  done
  jq -cn --arg repo "$repo" --argjson after "$after" --argjson verified "$verified" \
    '{repo: $repo, after: $after, verified: $verified}' >> "$LOG_FILE"
  [[ $verified == true ]] || { printf 'Verification failed: %s\n' "$repo" >&2; exit 1; }
  printf '%s: updated and verified\n' "$repo"
done < "$repos_file"
```
Start with its dry run:

```bash
bash github-actions-harden.sh repos.txt
```

After reviewing the target list and workflows that publish releases or create
PRs, apply the two settings:

```bash
bash github-actions-harden.sh repos.txt --apply
```

The script saves before/after records to `github-actions-permissions.jsonl`,
skips repositories already configured correctly, and verifies each update.
It retries verification because GitHub can briefly return stale settings
immediately after a successful write. A failure stops the batch; completed
updates remain applied. Rerunning is safe because correct repositories are
skipped.

The underlying write is small:

```bash
gh api "repos/$repo/actions/permissions/workflow" --method PUT \
  -f default_workflow_permissions=read \
  -F can_approve_pull_request_reviews=false
```

These examples change repository settings only. They do not change organization
defaults for newly created repositories, modify workflow files, or revoke a
personal access token.

## Preserve legitimate publishing permissions

An explicit workflow or job `permissions` block still controls the token
permissions GitHub allows for that run. A release job can request write access
while a PR build stays read-only:

```yaml
# Relevant excerpts from a workflow, not a complete runnable file.
permissions:
  contents: read

jobs:
  build:
    # Build and test with read-only permissions.
    # ...

  deploy:
    needs: build
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
    permissions:
      contents: write
    # Deployment steps go here.
    # ...
```

Keep publishing credentials in the job that needs them. Disable checkout
credential persistence where it is unnecessary, pin action references to
verified full commit SHAs, and protect branches and release tags. For required
checks, bind the expected source application where GitHub supports it.

Review workflows that use `pull_request_target`, consume artifacts from
untrusted runs, or run on self-hosted machines. Avoid passing untrusted PR
code into a job that has secrets or privileged tokens. Fork PR tokens are
normally restricted by GitHub, but same-repository PRs and privileged triggers
need their own review.

Disabling the PR setting can affect automation that creates or approves PRs.
Explicit token write permissions do not override that disabled capability;
review such automation before applying the setting to its repository.

## What to do if the payload actually ran

Reducing the default `GITHUB_TOKEN` permission does **not** neutralize a stolen
credential that can edit workflows. It also does not remove cloud persistence
or make an exposed secret safe again.

If execution is confirmed, preserve the evidence and stop the affected
automation. Revoke the credential used for the intrusion and rotate credentials
available to the affected execution. Inspect cloud audit logs, role trust
policies, newly created access keys, and federated identity credentials.
Isolate and rebuild compromised self-hosted runners or developer machines
after determining the exposure. Review releases and packages published during
the affected interval.

My review supported a preventive permission change, rather than emergency
rotation. A confirmed payload run would require incident response with a much
wider scope than these repository settings.

## References and freshness

Research date: **November 13, 2026**. The primary campaign report below was
published November 12. I attempted a
[Google search restricted to the last 48 hours](https://www.google.com/search?q=wfx-loot&tbs=qdr:d2),
but this network received a CAPTCHA challenge, so I could not verify its
results. The documentation links support the operational guidance; they are
not additional news reports from that time window.

1. [Aikido: wfx-loot campaign pushes CI secret stealer to tens of thousands of repositories](https://www.aikido.dev/blog/compromised-github-accounts-push-ci-secret-stealer-to-tens-of-thousands-of-repositories) — campaign sequence, indicators, and reported impact.
2. [GitHub REST API: Actions permissions](https://docs.github.com/en/rest/actions/permissions#setting-default-workflow-permissions-for-a-repository) — reading and changing the two repository settings.
3. [GitHub: Authentication in a workflow](https://docs.github.com/en/actions/reference/authentication-in-a-workflow) — default token permissions and workflow/job overrides.
4. [GitHub: Security hardening for GitHub Actions](https://docs.github.com/en/actions/security-for-github-actions/security-guides/security-hardening-for-github-actions) — minimum permissions, action pinning, and credential handling.
5. [GitHub REST API: Repository events](https://docs.github.com/en/rest/activity/events#list-repository-events) — event collection and its limitations.
