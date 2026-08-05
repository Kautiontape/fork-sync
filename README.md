# fork-sync

A reusable GitHub Actions workflow that detects new upstream releases for a
forked repo, optionally attempts to carry the fork's patches onto the new
release, and sends a notification.

## Modes

- **`rebase`** — replays the fork's carried patches (commits unique to the
  deploy branch, not already present upstream) on top of the new upstream tag
  via `git cherry-pick`, then opens a PR against the deploy branch.
- **`merge`** — merges the new upstream tag directly into the deploy branch
  (`git merge --no-ff`) on a new branch, then opens a PR against the deploy
  branch.
- **`notify`** — no branch work at all; just checks whether a new upstream
  tag has appeared since the last one recorded, and reports it. Useful for
  repos where syncing is manual but you still want a heads-up.

For `rebase`/`merge`, "already synced up to" is tracked by a marker file
(`base_marker`, default `.ktn-base`) committed on the deploy branch. **This
file must already exist on the deploy branch before the first run** — it's a
bootstrap contract, not something the workflow creates for you. If it's
missing, `detect` fails loudly (by design) rather than silently guessing a
starting point. Commit an initial marker (e.g. the tag you last synced to,
or the earliest tag you want carried patches replayed from) before enabling
`rebase`/`merge` mode on a repo.

For `notify`, the last-seen tag is tracked by a file, `LAST_TAG`, on an
orphan branch, `fork-sync-state`, in the calling repo — created via the Git
Data API with no parent commit and no connection to the repo's real history,
since there's no deploy branch to commit a marker to. This branch is created
and updated automatically; no bootstrap step needed.

In all modes, detection compares against the newest upstream tag whose name
matches `tag_pattern` (a `grep -E` pattern). Upstream tag names are treated
as untrusted input: anything empty or containing characters outside
`[A-Za-z0-9._-]` (or starting with `-`) is rejected before it's used as a
branch name or shell token.

### Rejected versions stay rejected

If a sync PR for a given tag was already opened and then closed/rejected
(not just merged), `fork-sync` will not re-open it — any prior PR for
`sync/<tag>` (open, merged, or closed) causes that tag to be skipped on
later runs. To re-attempt a version you previously rejected, delete the
`sync/<tag>` branch and its closed PR, or bump the marker/state past it.

## Inputs

| Input           | Required | Type    | Default     | Description                                                                                               |
| --------------- | -------- | ------- | ----------- | --------------------------------------------------------------------------------------------------------- |
| `upstream`      | yes      | string  | —           | Upstream repo as `owner/repo`.                                                                            |
| `mode`          | yes      | string  | —           | One of `rebase`, `merge`, `notify`.                                                                       |
| `tag_pattern`   | yes      | string  | —           | `grep -E` pattern used to pick the newest matching upstream tag.                                          |
| `base_marker`   | no       | string  | `.ktn-base` | Path to the marker file that records the last-synced tag (rebase/merge only, must pre-exist — see above). |
| `deploy_branch` | no       | string  | `ktn`       | Branch that PRs target and that the marker file lives on.                                                 |
| `dry_run`       | no       | boolean | `false`     | For rebase/merge: do the sync locally but don't push or open a PR.                                        |

## Secrets

| Secret          | Required | Description                                                                                                                                   |
| --------------- | -------- | --------------------------------------------------------------------------------------------------------------------------------------------- |
| `NTFY_URL`      | yes      | Push-notification endpoint the `notify` job posts to via `curl`. Supplied as an **org-level secret** — never hard-code it in a workflow file. |
| `SYNC_PR_TOKEN` | no       | PAT used for the `attempt` job's push and PR creation instead of the default token — see below.                                               |

### Why `SYNC_PR_TOKEN`

Pull requests created with the default `github.token` don't trigger other
workflows' `pull_request` checks (a deliberate GitHub anti-recursion
safeguard) — the sync PR would sit unchecked until someone manually
closes/reopens it. Sync branches can also contain changes to
`.github/workflows/*.yml`, which the default token is never allowed to
push, even with `contents: write`.

If a repo needs checks to run automatically on sync PRs, or expects synced
patches to touch workflow files, add `SYNC_PR_TOKEN` as a secret: a
fine-grained PAT scoped to that repo with **Contents: write, Pull requests:
write, Workflows: write**. When present, `attempt` uses it for both the
branch push and `gh pr create`. When absent, the workflow still works, but
the PR body gets an added warning and the ntfy detail is suffixed with
`(checks will not auto-run — close/reopen the PR to trigger them)`.

## Notification job

The `notify` job runs on the org's self-hosted runner
(`runs-on: [self-hosted, ktn]`) because it's the one that can reach the
private `NTFY_URL` endpoint; GitHub-hosted runners cannot. It only depends on
`curl` — do not assume `gh` or other tooling is present on that runner. It
needs no repo permissions (`permissions: {}`) and runs whenever a new tag
was detected, even if `attempt` failed or was skipped — a crashed sync is
reported as a failure notification, never silently mistaken for "just a
release announcement."

## Permissions

Every caller needs to grant **`contents: write, pull-requests: write`**,
regardless of `mode`. This isn't mode-conditional: GitHub validates each
job's declared `permissions:` inside `fork-sync` against what the caller
granted at dispatch time, for every job in the file, before any `if:`
condition is evaluated — so even in `notify` mode, where the `attempt` job
never actually runs, its declared need for `pull-requests: write` still has
to be satisfiable by the caller, or the whole run fails immediately with
`startup_failure` ("invalid workflow file"). Under-granting is the most
common way to break this workflow; there is no way to "downgrade" the grant
per mode.

Each job inside `fork-sync` still declares its own minimal `permissions:`
internally (`detect: contents: read`, `attempt: contents: write,
pull-requests: write`, `record: contents: write`, `notify: {}`) — so while
every caller has to grant the same ceiling, each job's actual token is
capped well below it.

## Caller template

Add a thin workflow like this to each fork that wants to use `fork-sync`:

```yaml
name: fork-sync
on:
  schedule:
    - cron: '0 9 * * *'
  workflow_dispatch:

concurrency:
  { group: 'fork-sync-${{ github.repository }}', cancel-in-progress: false }

jobs:
  sync:
    permissions:
      contents: write
      pull-requests: write # required for all modes, see Permissions above
    uses: Kautiontape/fork-sync/.github/workflows/fork-sync.yml@main
    with:
      upstream: <owner>/<repo>
      mode: rebase # or: merge | notify
      tag_pattern: '^v[0-9]+\.'
    secrets: inherit
```

`secrets: inherit` passes the org-level `NTFY_URL` secret (and `SYNC_PR_TOKEN`,
if configured) through automatically — no per-repo secret configuration
needed beyond adding `SYNC_PR_TOKEN` itself where wanted. The `concurrency`
block prevents overlapping runs (e.g. a manual dispatch racing the schedule)
from stepping on the same branch/marker.

## Smoke test

`.github/workflows/smoke-test.yml` runs `fork-sync` in `notify` mode against
`actualbudget/actual`, dispatched manually (`workflow_dispatch`). It's a
quick way to confirm the reusable workflow and notification path still work
without touching any fork's deploy branch.
