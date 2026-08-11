# fork-sync

A reusable GitHub Actions workflow that detects new upstream releases for a
forked repo, optionally attempts to carry the fork's patches onto the new
release, and sends a notification.

## Modes

- **`rebase`** — replays the fork's carried patches (commits unique to the
  deploy branch, not already present upstream) on top of the new upstream tag
  via `git cherry-pick`, then opens a PR against the deploy branch. That PR is
  **not** merged with the merge button — see
  [Promoting a rebase sync](#promoting-a-rebase-sync).
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
`SYNC_PR_TOKEN` (when set) is also the token `token_check` uses to monitor
its own expiry — see below.

## Notification job

The `notify` job runs on the org's self-hosted runner
(`runs-on: [self-hosted, ktn]`) because it's the one that can reach the
private `NTFY_URL` endpoint; GitHub-hosted runners cannot. It only depends on
`curl` — do not assume `gh` or other tooling is present on that runner. It
needs no repo permissions (`permissions: {}`) and runs whenever a new tag
was detected, even if `attempt` failed or was skipped — a crashed sync is
reported as a failure notification, never silently mistaken for "just a
release announcement."

## Failure catch-all

`report_failure` runs whenever any of `detect`/`attempt`/`notify`/`record`
failed, regardless of whether `notify` itself already ran (a `detect` crash,
for instance, means `notify` never got the chance). It sends one high-priority
ntfy ping with a link to the failed run. Like `notify`, it needs the
self-hosted runner to reach `NTFY_URL`.

**Known limit**: if the self-hosted runner itself is down, _neither_ `notify`
nor `report_failure` can send anything — there's no code-level fix for that
inside this workflow. The backstops are (1) GitHub's own run-failure email to
the repo's watchers, and (2) the monthly heartbeat ping below: if the runner
has been down long enough, the heartbeat itself stops arriving, which is the
signal to go check.

## Token expiry monitoring, keepalive, and the monthly heartbeat

Two more jobs, `token_check` and `token_notify`, run on **every** invocation
of `fork-sync` — regardless of `mode`, regardless of whether a new upstream
tag was found. This is deliberate: a `SYNC_PR_TOKEN` expiring during a quiet
stretch with no releases would otherwise never be reported, since that path
never touches `detect`'s `has_new` branch or the `notify` job at all.

- **PAT expiry warnings.** `token_check` calls `gh api -i user` with
  whatever token `attempt` would use (`SYNC_PR_TOKEN` if set, else the
  default token) and reads the `github-authentication-token-expiration`
  response header (format confirmed empirically: `YYYY-MM-DD HH:MM:SS UTC`,
  e.g. `2027-08-06 04:39:13 UTC`). ≤30 days left → a default-priority
  warning; ≤7 days → high priority. With a weekly schedule that's ~4
  warnings before a 30-day window closes, regardless of whether the PAT was
  issued with a 1-year or shorter expiry.
  - If the token returns **no** expiration header at all (a classic token,
    the default `GITHUB_TOKEN`, or a fine-grained PAT explicitly set to
    "No expiration"), `token_check` sends **one** informational notice the
    first time and then stays quiet — it can't monitor what GitHub doesn't
    expose, and repeating that same notice weekly forever would just be
    noise. It remembers having sent it via a marker file on `fork-sync-state`.
  - If the authenticated call itself fails outright (bad/revoked token),
    that's reported immediately as its own high-priority alert rather than
    being folded into the "no expiration header" case.
- **Keepalive.** GitHub auto-disables scheduled workflows in public repos
  after **60 days with no repository activity** — a real risk for an idle
  fork that rarely gets new upstream releases. On every run, `token_check`
  writes a timestamp to a `HEARTBEAT` file on `fork-sync-state`, using the
  same git-data/contents-API approach `record` uses (creating the branch
  parentless if it doesn't exist yet). A weekly cron therefore always keeps
  the repo well inside the 60-day window.
  - **Safety**: this write goes **only** to the orphan `fork-sync-state`
    branch, never to a deploy branch — a stray commit on a deploy branch
    (e.g. `ktn`) would be a production deploy for callers whose build
    triggers on `push` to that branch. Confirmed each caller's build trigger
    before relying on this: `actual` and `trilium` build on
    `push: branches: [ktn]`; `feishin` builds on tags only. An orphan-branch
    push triggers none of them, and it needs to stay that way.
- **Monthly "still alive" ping.** On the first scheduled run of each month
  (`github.event_name == 'schedule'` and day-of-month ≤ 7 — one Monday
  always falls in that window for a weekly cron), `token_check` also queues
  a low-priority ntfy message reporting the last-seen upstream tag and days
  until token expiry. This is deliberately gated to `schedule` only, not
  manual `workflow_dispatch` runs (including this repo's own smoke test) —
  it's meant to answer "is the _scheduled_ automation still alive," not "did
  someone click the button recently." The logic: if a monthly ping stops
  arriving, the automation is dead somewhere upstream of this workflow
  (auto-disabled, runner down, token dead) — that silence **is** the alert.
  Costs about 4 low-priority pings a month across the whole fleet of forks.

`token_check` (ubuntu-latest) only decides what needs sending and returns it
as a JSON array output; `token_notify` (self-hosted, same reachability
reason as `notify`) is the job that actually `curl`s each queued message to
`NTFY_URL`.

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

## Promoting a rebase sync

`.github/workflows/fork-sync-promote.yml` is the integration step for
`rebase`-mode syncs, dispatched by hand once the sync branch's checks are
green. It is a separate reusable workflow with its own thin caller.

A rebase sync branch is the deploy branch _rebuilt_ onto a newer upstream tag,
so it has to **replace** the deploy branch wholesale. Merging it would weave
both histories together, and every carried patch would then appear twice in
the next sync's carry list. The sync PR also conflicts on the marker file by
construction — both sides changed it from different bases — so GitHub's merge
button is unusable anyway. That is deliberate, not a bug to work around.

Promote instead: it verifies the branch's check runs, force-pushes it over the
deploy branch with a lease, waits for GitHub's indirect-merge detection to
flip the sync PR to merged (the head commits become reachable from the base),
then deletes the sync branch.

### Inputs

| Input           | Required | Type    | Default     | Description                                        |
| --------------- | -------- | ------- | ----------- | -------------------------------------------------- |
| `tag`           | yes      | string  | —           | Upstream tag whose `sync/<tag>` branch to promote. |
| `base_marker`   | no       | string  | `.ktn-base` | Marker file to cross-check against `tag`.          |
| `deploy_branch` | no       | string  | `ktn`       | Branch to replace.                                 |
| `dry_run`       | no       | boolean | `false`     | Verify and report only — push nothing.             |

Grant `contents: write`. `NTFY_URL` is required; `SYNC_PR_TOKEN` is optional
but **effectively required** — a push made with `github.token` never starts
workflow runs, so the deploy branch's build and deploy would silently not
fire. Without it the run logs a `::warning::` saying exactly that, and you
have to trigger the deploy by hand.

Use the **same concurrency group as the repo's `fork-sync` caller**
(`fork-sync-${{ github.repository }}`) so a promote can never race a running
sync attempt. A top-level `concurrency:` block inside the reusable workflow
would be ignored — it only takes effect in the caller.

```yaml
name: fork-sync promote
on:
  workflow_dispatch:
    inputs:
      tag: { type: string, required: true }
      dry_run: { type: boolean, default: false }

concurrency:
  { group: 'fork-sync-${{ github.repository }}', cancel-in-progress: false }

permissions:
  contents: write

jobs:
  promote:
    uses: Kautiontape/fork-sync/.github/workflows/fork-sync-promote.yml@main
    with:
      tag: ${{ inputs.tag }}
      dry_run: ${{ inputs.dry_run }}
    secrets: inherit
```

### Outcomes

| `outcome`          | Exit | ntfy priority | When                                                      |
| ------------------ | ---- | ------------- | --------------------------------------------------------- |
| `promoted`         | 0    | default       | Deploy branch replaced.                                   |
| `already-promoted` | 0    | low           | Nothing to do — this tag is already on the deploy branch. |
| `dry-promote`      | 0    | low           | `dry_run: true` and the branch is promotable.             |
| (none — refused)   | 1    | high          | Any guard below fired.                                    |

`already-promoted` is the duplicate-dispatch case: a successful promote
deletes the sync branch, so a second dispatch finds nothing. Rather than
treat that as a failure, the workflow cross-checks the deploy branch's marker
— if it already reads the requested tag, the work is genuinely done, so it
exits 0 with a quiet ping instead of paging. A missing sync branch whose tag
does **not** match the marker is still a hard error.

### Refusals

Every guard runs before anything is pushed, in this order:

| Message                                        | Meaning                                                                                                                                 |
| ---------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------- |
| `unsafe tag: '<tag>'`                          | Same hostile-input rule as `detect` — the tag reaches git refs and shell strings.                                                       |
| `branch sync/<tag> not found ... records 'X'`  | No sync branch, and the marker disagrees — a typo'd tag, or the sync never ran. (When the marker agrees, see `already-promoted` above.) |
| `<marker> on sync/<tag> reads 'X', expected …` | The branch was built for a different tag — wrong `tag` input.                                                                           |
| `no check runs on <branch>@<sha>`              | The gate never fired. The branch predates a push-triggered gate, or was pushed with `github.token`. Promoting is refused.               |
| `N check run(s) still in progress`             | Wait for the gate, then re-dispatch.                                                                                                    |
| `not green: <name> -> <conclusion>`            | A check failed. `continue-on-error` jobs report success, so a caller's non-blocking jobs stay non-blocking here too.                    |

Because `notify` runs on `!cancelled()` and reads no outputs from a failed
`promote`, every refusal above sends the same high-priority `rotating_light`
ping titled `<repo>: <tag> promote FAILED` — it cannot say which guard fired,
so read the run log before treating one as an incident. The one formerly
benign case, a duplicate dispatch, is now `already-promoted` and no longer
pages at all.

## Smoke test

`.github/workflows/smoke-test.yml` runs `fork-sync` in `notify` mode against
`actualbudget/actual`, dispatched manually (`workflow_dispatch`). It's a
quick way to confirm the reusable workflow and notification path still work
without touching any fork's deploy branch.
