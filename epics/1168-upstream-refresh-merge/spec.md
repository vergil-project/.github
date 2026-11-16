# Upstream-refresh PRs merge themselves — design

- **Epic:** vergil-project/.github#1168
- **Planning issue:** vergil-project/.github#1067 (review history is in its comments)
- **Status:** draft, in design review

## 1. Intent

The scheduled upstream-refresh job should carry its PR all the way to a merge
into `develop`, with no human involved, and should count as done only once that
merge has happened. Epic vergil-project/.github#1064 built that behaviour for
the nightly `github-config` job: a mechanized PR on a protected `automation/`
branch that the job watches until it merges. This epic applies the same
standard to upstream-refresh, the next most important job that opens PRs.

Upstream-refresh runs every night in two repos:

- **vergil-containers.** Its hook (`docker/pins/upstream-refresh.sh`) moves
  the `ARG *_VERSION=` pins of the auto-managed tools to their newest
  releases and regenerates `docker/pins/CATALOG.md`.
- **vergil-python.** Its hook (`scripts/dev/upstream-refresh.sh`) moves each
  CPython series in `runtime.toml` to its newest patch.

Both are how upstream fixes, security patches included, enter the fleet.
Today each refresh waits on `develop`'s doorstep until a human merges it.

## 2. Why

### 2.1 The refresh stops at an open PR

`vrg-upstream-refresh` keeps one PR open on the fixed `upstream-refresh`
branch: it opens it, force-pushes new content into it, and closes it once
nothing has drifted. It never merges. The PR body says so: "A human reviews
and merges; nothing auto-merges." The human does not, in practice, review
these PRs. CI is the gate, as it is for fleet sync (epic #1064, spec §2.3),
and merging them by hand is toil. Refreshes that are not merged pile up
unseen. On 2026-11-15 a CPython refresh had been open on vergil-python for
six days (vergil-project/.github#1145).

### 2.2 The fleet standard already exists

Epic #1064 made the mechanized-PR building block
(`vergil_tooling.lib.mechanized_pr`) the fleet standard for every automated
job that opens a PR (its spec §6.2, G4). It named upstream-refresh as the
first job to move. Upstream-refresh is a mechanized job in exactly that sense:
its content comes from a reviewed, merged hook script, with no AI agent
involved.

### 2.3 The one-hour token does not cover vergil-containers' CI

The building block waits up to 40 minutes (`DEFAULT_DEADLINE_SECS = 2400`),
because the automation App's installation token lasts one hour and cannot be
renewed inside a running process. That is enough for fleet sync, whose PRs
merge in about 5 minutes.

It is not enough for vergil-containers. Its upstream-refresh PRs' CI took
32 to 40 minutes in every run from 2026-11-12 to 2026-11-15 (`CI`,
`pull_request` event, on the `upstream-refresh` branch). Those times exclude
the hook's run time. A branch that falls behind `develop` reruns CI from the
start. vergil-python's refresh CI takes about 2 minutes.

So the job must be able to wait longer than one token's life. The same limit
blocks the next jobs to move: the vergil-containers lifecycle PRs run the
same CI, and `vrg-update-deps` takes about 80 minutes. This epic therefore
solves it once, in the building block.

## 3. Goals, non-goals, anti-goals

### Goals

- G1. The building block can wait on a PR it opened across more than one
  token's life, without the tool ever holding the App's private key.
- G2. `vrg-upstream-refresh` opens its PR through the building block. Its
  run succeeds only when the PR has merged into `develop`, or when there was
  nothing to change.
- G3. `ops-upstream-refresh.yml` runs as the `automation` identity, under
  the same guards as `github-config`: `develop` only, scoped tokens, and the
  App's bot as commit author.
- G4. In vergil-python and vergil-containers, a real refresh merges with no
  human involved, and the run is green. vergil-containers proves the path
  past one hour.

### Non-goals

- The vergil-containers language lifecycle job (vergil-project/.github#1128):
  one PR per family, merged in order. It is the next epic, and it builds on
  G1.
- Releasing automatically once a refresh merges (the second half of
  vergil-project/.github#1145). This is a high-priority follow-on. It depends
  on release rights for the automation identity, which epic #1064 listed as a
  non-goal, and on a per-repo lock for single-threaded processes
  (vergil-project/.github#1088).
- Moving `vrg-update-deps` into automation. It runs inside the human-gated
  `vrg-release`, and its human-credential run is not bound by the token's
  life.
- Changing the human-run PR tools (`vrg-release`, `vrg-update-deps`,
  `vrg-submit-pr --finalize`). They already wait for and merge their own PRs
  through `pr_merge.wait_and_merge`.
- Moving the jobs that mint the App token but open no PR (epic-sweep,
  epic-rollup, branch-sweep, dependabot-audit, `cd.yml`, `cd-release.yml`)
  onto the `automation` mode, and the `develop`-only credentials environment
  (vergil-project/vergil-actions#1185).
- Changing either repo's hook, or what it refreshes.
- Token renewal inside a process (§5, approach B).

### Anti-goals

Outcomes that count as failure even if the feature works:

- The App's private key reaching the environment of `vrg-upstream-refresh`,
  or of the repo hook it runs.
- A refresh merging from any branch's code other than `develop`'s.
- A silent failure. A refresh PR that cannot merge must leave a red run that
  names the PR. A run must not go green while its PR is still open.
- More than one open refresh PR per repo, or a nightly run that piles a new
  PR on top of an earlier one instead of replacing it.
- An await step that can be pointed at an arbitrary PR and merge it.

## 4. Current state (facts the design builds on)

- **`lib/upstream_refresh.py` and `bin/vrg_upstream_refresh.py`.**
  - The CLI takes `--repo`, `--base` (default: the repo's default branch)
    and `--dry-run`. It refuses agent modes, and requires a clean checkout
    whose `HEAD` is `origin/<base>`.
  - The hook (`[upstream-refresh].hook` in `vergil.toml`) runs with
    `bash -c` in a temporary linked worktree detached at the base. It gets
    `$VRG_UPSTREAM_REPORT`, a file whose non-empty lines become the PR
    body's notes.
  - A hook that leaves untracked files is refused ("edit tracked files
    only").
  - `sync()` keeps one PR on `upstream-refresh` (it also adopts
    `upstream-refresh-*`). It pushes with `--force-with-lease` only when the
    tree differs, comments "Refreshed", closes extra PRs, and closes the PR
    once nothing has drifted.
- **`ops-upstream-refresh.yml`** (vergil-actions).
  - It mints one App token with `contents: write` and
    `pull-requests: write`, and checks out the run's ref with that token.
  - It sets the git identity to `github-actions[bot]`, installs
    vergil-tooling, and runs `vrg-upstream-refresh --base "$BASE"`, where
    `BASE` is `github.ref_name`.
  - It has no ref guard, no `VRG_AUTOMATION_APP`, and no `timeout-minutes`.
    Its concurrency group is `upstream-refresh`.
  - The callers are vergil-containers' and vergil-python's `ops.yml`, both
    at `@v2.1`.
- **The building block** (`lib/mechanized_pr.py`, epic #1064).
  `open_sync_pr(repo, clone, *, job, title, body)`:
  1. closes the open same-repo PRs under `automation/<job>/` and deletes
     those branches;
  2. returns `in_sync` when the clone is clean;
  3. otherwise commits on `automation/<job>/<run-id>` (the run ID is
     `$GITHUB_RUN_ID`, or a timestamp and random hex) and pushes;
  4. opens the PR against `develop`, turns on squash auto-merge pinned to
     the pushed head, and reads it back;
  5. waits through `pr_merge.wait_until_merged(strategy="squash",
     deadline_secs=…)`.

  The wait merges a ready PR itself and updates a branch that falls behind,
  up to `_MAX_BRANCH_UPDATES = 5`. It raises on a closed PR, a draft, a
  conflict or a failed check, and on the deadline (`MergeDeadlineError`).
  Any failure surfaces as `PRNotMergedError(pr_url, reason)`. The deadline
  comes from `$VRG_MECHANIZED_PR_DEADLINE`, default 2400 seconds. The
  result is `SyncResult(status, pr_url, closed)`.
- **The `github-config` job pattern** (`ops-github-config.yml`).
  1. A token-free first step fails unless `github.ref` is
     `refs/heads/develop` and `vars.VRG_AUTOMATION_APP` is non-empty. It
     reads the variable as `AUTOMATION_APP`.
  2. The App token is minted with explicit permissions, and the checkout
     uses `ref: develop`.
  3. Then come `gh auth setup-git`, and a git identity of `<app-slug>[bot]`
     with the bot user's noreply address.
  4. `VRG_AUTOMATION_APP` is set only on the step that runs the tool. Set
     job-wide, it broke the install step (vergil-project/vergil-actions#1192).
  5. The job has `timeout-minutes: 60` and the concurrency group
     `${{ github.repository }}/github-config`.
- **The protected prefix.** Since epic #1064, every repo's desired state has
  a branch ruleset on `refs/heads/automation/**`. Only the automation App and
  the repository admin role may create or push those branches. The repo
  setting `allow_auto_merge` is on, and `delete_branch_on_merge` is on.
  `develop` requires passing, up-to-date status checks and no approvals.
- **The `automation` identity mode** (`lib/identity_mode.py`,
  `lib/automation_identity`). A process resolves to `automation` only when
  `VRG_AUTOMATION_APP` is set and `GH_TOKEN` is verified as that App's
  installation token. Epic #1064's rule: every identity check on a path the
  automation identity runs states explicitly what it does with `automation`.
- **Open refresh PRs** exist today on the fixed `upstream-refresh` branch in
  both repos. The new mechanism does not look at that branch.

## 5. Approaches

How the job waits past the token's one-hour life:

- **A. Split open from wait, with a fresh token for each wait round
  (chosen).** The building block gains a separate, resumable await entry
  point. The workflow opens the PR in one step, then runs a fixed number of
  await steps, minting a new App token before each one. The private key
  never leaves `actions/create-github-app-token`. The costs are some
  repetition in the workflow's YAML, and a total wait capped at the number
  of rounds. The human picked this, and treats the cap as a feature: a
  refresh that has not merged after the last round should fail loudly.
- **B. The tool renews its own token.** The workflow passes the App's
  private key to the tool, which mints a fresh installation token when the
  current one nears expiry. This has no cap and simpler YAML. But the key
  would sit in the environment of the process that runs the repo's hook,
  widening exactly the exposure vergil-actions#1185 means to narrow.
  Rejected.
- **C. Stop at auto-merge, and report merges separately.** This contradicts
  the intent: the job would go green with its PR still open, as epic
  #1064's live rollout found for PRs that fell behind `develop`. Rejected.

How upstream-refresh opens its PR:

- **Rebase it onto the building block (chosen).** The fixed branch, the
  force-push and the close-if-stale logic are replaced by the building
  block's per-run branches and its "close the earlier run" step. That
  matches the fleet standard, and the protected prefix keeps agents away
  from the branch.
- **Keep the fixed branch and add a wait.** This leaves the branch outside
  `automation/**`, where any identity with push rights can write to it, and
  keeps two PR mechanisms in the fleet. Rejected.

## 6. Design

### 6.1 The building block: open, then await

`lib/mechanized_pr.py` gains a second entry point. It also lets a caller
stop after opening.

- **Open without waiting.** A caller can ask `open_sync_pr` to open the PR
  and turn on auto-merge, and then return without waiting. The result
  status for that case is `opened`, carrying the PR URL. The existing
  behaviour (open, then wait) stays the default, and `github-config`'s
  `update` is unchanged.
- **Await an open PR.** The new function takes the repo, the job name and a
  PR URL, and waits on that PR through the same `pr_merge.wait_until_merged`
  loop, with the same deadline rule (`$VRG_MECHANIZED_PR_DEADLINE`, default
  2400 seconds). It returns one of these outcomes:
  - **merged:** the PR has merged, either during this wait or before it
    started. That second case makes a re-run of a round harmless.
  - **pending:** the deadline passed while the PR was still open, not a
    draft, not conflicted and with no failed check. It is a normal outcome,
    not an error.
  - **failed:** everything the wait treats as terminal today (closed, draft,
    conflict, failed check, the branch-update cap). This raises
    `PRNotMergedError` with the PR URL and the reason, as it does now.
- **The await refuses a PR that is not its own job's.** Before waiting, it
  checks that the PR's base is `develop`, its head is in the same repo
  (not a fork), and its head branch is under `automation/<job>/`. Otherwise
  it fails without touching the PR. Only the automation App and the admin
  can push those branches (§4), so the await can only ever merge content
  the automation produced.
- **The branch-update cap applies per await call.** Each round may update a
  behind branch up to `_MAX_BRANCH_UPDATES` times. (Builder's call: a
  run-wide cap would mean passing state between steps for little gain,
  since the number of rounds already bounds the total.)
- **Deadline overrun.** `wait_until_merged` raises `MergeDeadlineError`
  today. The await reports that as **pending**, and the open-and-wait
  default reports it as a failure exactly as it does now. No other error is
  reclassified.

### 6.2 `vrg-upstream-refresh` on the building block

- **Where the hook runs.** The tool takes a fresh clone of `develop`
  (`develop_clone.clone_develop`) and runs the hook there, with the same
  `bash -c` invocation and the same `$VRG_UPSTREAM_REPORT` contract. The
  scratch linked worktree, the "clean checkout at `origin/<base>`" check,
  and `--base` are removed. The base is always `develop`, as with every
  mechanized PR. The tool then works the same from any directory, and never
  touches the caller's checkout.
- **The hook contract is unchanged.** Untracked files left by the hook are
  still refused, before anything reaches the building block, because the
  building block commits whatever `git status` shows. The changed-paths list
  and the hook's notes still make up the PR body. Its intro changes from
  "a human merges" to say that the job merges the PR itself once CI passes.
- **Opening the PR.** The job name is `upstream-refresh`, so its branches are
  `automation/upstream-refresh/<run-id>`. The title stays
  `chore(upstream): refresh upstream inputs`. Everything in `sync()` that
  manages the fixed branch is removed: `REFRESH_BRANCH`, adopting
  `upstream-refresh-*`, the `--force-with-lease` push, the
  "unchanged/refreshed" in-place update, and closing extra PRs. A run that
  finds nothing to change still closes an earlier run's open PR, because
  the building block's first step does that.
- **The command line** (the flags' exact spelling is the builder's call):
  - by default the tool opens the PR, then waits in the same process (a
    human's local run, and any caller that fits in one token);
  - an open-only mode opens the PR and records its URL for the next step.
    In Actions it writes the URL to `$GITHUB_OUTPUT`; it always prints it;
  - an await mode takes a PR URL and runs one await round (§6.1);
  - `--dry-run` stays: it runs the hook and prints the PR body, pushing
    nothing.
- **Exit codes**, stated in `--help`:
  - **0:** merged, or nothing to change;
  - **2:** not merged, naming the PR and the reason. This covers a
    terminal failure, and **pending** in the final round;
  - a distinct code for **pending** in a non-final round, so the workflow
    can tell "wait another round" apart from "stop". The await mode is told
    whether it is the final round. A pending result in the final round is a
    failure, exit 2, so a run never ends green with its PR open.
  - Any other failure (the hook failed, the clone failed, a refused
    identity) exits as the tool's errors do today, non-zero with the
    reason.
- **Identity.** The tool accepts `human` and `automation`, and refuses the
  agent modes, as it does today. Its identity check names `automation`
  explicitly (epic #1064's rule).
- **Logging.** Every run prints:
  - the PR it closed, if any;
  - the PR it opened, or "in sync";
  - each await round's outcome, with the PR URL and the time waited.

  The log alone must be enough to tell what happened, without opening
  GitHub.

### 6.3 `ops-upstream-refresh.yml`

The reusable workflow takes on the `github-config` pattern (§4), with
multiple await rounds:

1. **Guard step**, before any token exists: fail unless `github.ref` is
   `refs/heads/develop` and `vars.VRG_AUTOMATION_APP` is non-empty, reading
   the variable as `AUTOMATION_APP`. A failing step, never a job-level
   `if:`, so a run from another ref shows up red instead of being skipped.
2. **Open:**
   - mint an App token scoped to the calling repo, with `contents: write`
     and `pull-requests: write`, plus `workflows: write` only if a hook can
     touch `.github/workflows/` (neither does today, so it is left out);
   - check out `ref: develop`;
   - install vergil-tooling;
   - run `gh auth setup-git`;
   - set the git identity to `<app-slug>[bot]`;
   - run the tool in open-only mode, with `VRG_AUTOMATION_APP` set on that
     step only.

   If the run is in sync, the job ends green here.
3. **Await rounds:** three rounds. Each round:
   - mints a new App token, with `pull-requests: write` and
     `contents: write` (the branch update writes to the branch);
   - runs the tool's await mode on the PR from step 2, with
     `VRG_AUTOMATION_APP` and the new token on that step only.

   A round runs only when the PR is still pending after the previous round.
   The third round is the final one. A terminal failure in any round fails
   the job at once, and no later round runs.
4. **Time cap:** `timeout-minutes` covers the open step and three 40-minute
   rounds with margin (about 150 minutes). The plan fixes the exact number.
5. **Concurrency:** `group: ${{ github.repository }}/upstream-refresh`,
   `cancel-in-progress: false`.
6. **The header comment** loses "it never merges; a human merges", and
   explains the rounds and why the private key stays in the token action.

Three rounds give about 120 minutes of waiting. That covers vergil-containers'
CI (32 to 40 minutes) with one branch update and its CI re-run, with room to
spare. (Builder's call: three, rather than a configurable count. An input can
be added later if a job needs more.)

The callers' `ops.yml` files in vergil-containers and vergil-python, and their
`vergil.toml` comments, lose "nothing is auto-merged; a human merges".
Neither caller's job wiring needs to change otherwise, since both pass the
same two secrets at `@v2.1`. Whether the caller job needs any new permission
is for the builder to confirm against the reusable workflow.

### 6.4 Rollout

Each step stays inert until the next one switches it on:

1. **Tooling.** vergil-tooling releases the building-block change (§6.1) and
   the new `vrg-upstream-refresh` (§6.2). The released ops workflow still
   runs the old CLI flags, and nothing calls the new modes. So the release
   changes no scheduled behaviour unless the old command line stops working.
   The plan must either keep the old invocation working until step 2, or
   ship steps 1 and 2 so that no nightly run sits between them. The plan
   decides which.
2. **Workflow.** vergil-actions releases the new `ops-upstream-refresh.yml`
   (§6.3). From its next run, each caller runs the new flow.
3. **One-time cleanup.** The open PRs on the old fixed `upstream-refresh`
   branch in both repos are closed, and their branches deleted. Either the
   human does this, or a deployment task does it with a closing comment
   that points to the new mechanism. The new tool carries no transitional
   code for the old branch name.
4. **Live validation** (§7).

### 6.5 Local runs

A human running `vrg-upstream-refresh` locally gets open-and-wait in one
process, under the human's own credentials. Those are not bound to one hour,
and `$VRG_MECHANIZED_PR_DEADLINE` can raise the wait. The `automation/**`
ruleset admits the admin role, so the human's push is accepted. As with
`update` (epic #1064, spec §6.5), the tool works in its own temporary clone
and never touches the human's checkout.

## 7. Validation

A validation task, run after the vergil-actions release, proves G4 live:

- **vergil-python:** a dispatched (or nightly) run either finds nothing to
  change and exits 0, or opens an `automation/upstream-refresh/<run-id>` PR
  as `<org>-automation[bot]`, which merges within the run. The run is green,
  and `develop` carries the commit.
- **vergil-containers:** a run opens a refresh PR and merges it, and the run
  is green. **The path past one hour must run live at least once:** a merge
  in an await round after a pending round. Epic #1064's retrospective notes
  that its branch-update path never ran live. vergil-containers' CI usually
  finishes inside one 40-minute round, so the validation must force a
  pending round, for example with a shorter per-round deadline on a
  dispatched run. How is the plan's call.
- **No legacy PRs:** neither repo has an open PR on `upstream-refresh`.
- **The guard:** a dispatch from a branch other than `develop` fails at the
  guard step, before any token is minted.

The log lines of §6.2 are the evidence, along with the PR's author and
merge, and the job's conclusion.

## 8. What's left to the builder

- The exact flag names and the "pending" exit code (§6.2).
- How the await rounds are written in YAML (repeated steps or a matrix of
  fixed size, for example), as long as each round mints its own token, a
  terminal failure stops the rounds, and no key reaches the tool.
- The open step's output plumbing to the await rounds.
- The job's exact `timeout-minutes`.
- How the validation forces a pending round (§7).
- How steps 1 and 2 of the rollout avoid a broken nightly run (§6.4).

## 9. Follow-ons this epic leaves

- **Lifecycle PRs** (vergil-project/.github#1128): one PR per family, merged
  in order, past one hour through §6.1's await. It needs its own planning
  issue once this epic's planning issue closes, because that issue
  (vergil-project/.github#1067) also named the lifecycle job.
- **Release after refresh** (vergil-project/.github#1145, second half): high
  priority, blocked on automation release rights and the release lock
  (vergil-project/.github#1088).
- **Logging the resolved identity mode** (vergil-project/vergil-tooling#4433,
  intake) applies to this job too.
