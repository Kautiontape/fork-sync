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
(`base_marker`, default `.ktn-base`) committed on the deploy branch. For
`notify`, it's tracked by a file, `LAST_TAG`, on an orphan branch,
`fork-sync-state`, in the calling repo — created via the Git Data API with no
parent commit and no connection to the repo's real history, since there's no
deploy branch to commit a marker to.

In all modes, detection compares against the newest upstream tag whose name
matches `tag_pattern` (a `grep -E` pattern).

## Inputs

| Input           | Required | Type    | Default     | Description                                                                   |
| --------------- | -------- | ------- | ----------- | ----------------------------------------------------------------------------- |
| `upstream`      | yes      | string  | —           | Upstream repo as `owner/repo`.                                                |
| `mode`          | yes      | string  | —           | One of `rebase`, `merge`, `notify`.                                           |
| `tag_pattern`   | yes      | string  | —           | `grep -E` pattern used to pick the newest matching upstream tag.              |
| `base_marker`   | no       | string  | `.ktn-base` | Path to the marker file that records the last-synced tag (rebase/merge only). |
| `deploy_branch` | no       | string  | `ktn`       | Branch that PRs target and that the marker file lives on.                     |
| `dry_run`       | no       | boolean | `false`     | For rebase/merge: do the sync locally but don't push or open a PR.            |

## Secrets

| Secret     | Required | Description                                                                                                                                   |
| ---------- | -------- | --------------------------------------------------------------------------------------------------------------------------------------------- |
| `NTFY_URL` | yes      | Push-notification endpoint the `notify` job posts to via `curl`. Supplied as an **org-level secret** — never hard-code it in a workflow file. |

## Notification job

The `notify` job runs on the org's self-hosted runner
(`runs-on: [self-hosted, ktn]`) because it's the one that can reach the
private `NTFY_URL` endpoint; GitHub-hosted runners cannot. It only depends on
`curl` — do not assume `gh` or other tooling is present on that runner.

## Permissions

The caller's `GITHUB_TOKEN` needs different permissions depending on `mode`,
since all state (the `base_marker` file and the `notify`-mode `LAST_TAG` file)
is written via `contents`, never via Actions variables/secrets (the default
token can't write those under any permission grant):

| Mode             | Required permissions                      |
| ---------------- | ----------------------------------------- |
| `rebase`/`merge` | `contents: write`, `pull-requests: write` |
| `notify`         | `contents: write`                         |

## Caller template

Add a thin workflow like this to each fork that wants to use `fork-sync`:

```yaml
name: fork-sync
on:
  schedule:
    - cron: '0 9 * * *'
  workflow_dispatch:

jobs:
  sync:
    permissions:
      contents: write
      pull-requests: write # omit for notify mode
    uses: Kautiontape/fork-sync/.github/workflows/fork-sync.yml@main
    with:
      upstream: <owner>/<repo>
      mode: rebase # or: merge | notify
      tag_pattern: '^v[0-9]+\.'
    secrets: inherit
```

`secrets: inherit` passes the org-level `NTFY_URL` secret through
automatically — no per-repo secret configuration needed.

## Smoke test

`.github/workflows/smoke-test.yml` runs `fork-sync` in `notify` mode against
`actualbudget/actual`, dispatched manually (`workflow_dispatch`). It's a
quick way to confirm the reusable workflow and notification path still work
without touching any fork's deploy branch.
