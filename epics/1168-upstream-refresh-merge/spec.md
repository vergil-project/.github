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
because the automation App's installation token lasts one hour and the job
has no way to get a new one: the workflow mints the token once, before the
tool starts. That is enough for fleet sync, whose PRs merge in about 5
minutes.

It is not enough for vergil-containers. Its upstream-refresh PRs' CI took
32 to 40 minutes in every run from 2026-11-12 to 2026-11-15 (`CI`,
`pull_request` event, on the `upstream-refresh` branch). Those times exclude
the hook's run time. A branch that falls behind `develop` reruns CI from the
start. vergil-python's refresh CI takes about 2 minutes.

So the job must be able to keep working after its first token has expired.
The job can never finish faster than its PR's CI, so the wait itself is not
the problem: the token's fixed life is. The same limit blocks the next jobs
to move: the vergil-containers lifecycle PRs run the same CI, and
`vrg-update-deps` takes about 80 minutes. This epic therefore solves it once,
in the building block, in Python, so every later job gets it without any
workflow logic of its own.

## 3. Goals, non-goals, anti-goals

### Goals

- G1. The building block renews its own automation token while it waits, so
  a mechanized PR can take longer than one token's life to merge. The logic
  lives in Python, and a workflow that uses it stays declarative.
- G1a. The key boundary (§6) holds: the App's private key never shares a
  runner with repo-supplied or third-party code.
- G2. `vrg-upstream-refresh` opens its PR through the building block. Its
  run succeeds only when the PR has merged into `develop`, or when there was
  nothing to change.
- G3. `ops-upstream-refresh.yml` runs as the `automation` identity, under
  the same guards as `github-config`: `develop` only, tokens scoped to the
  job's needs, and the App's bot as commit author.
- G4. In vergil-python and vergil-containers, a real refresh merges with no
  human involved, and the run is green. vergil-containers proves a renewal
  live: its PR merges after the token the run started with has been
  replaced and revoked.

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
- Moving `github-config` onto token renewal. Its PRs merge well inside one
  token, and it keeps minting its token in the workflow as today. It can move
  later with no change to the building block.
- Moving the jobs that mint the App token but open no PR (epic-sweep,
  epic-rollup, branch-sweep, dependabot-audit, `cd.yml`, `cd-release.yml`)
  onto the `automation` mode, and the `develop`-only credentials environment
  (vergil-project/vergil-actions#1185).
- Changing either repo's hook, or what it refreshes.

### Anti-goals

Outcomes that count as failure even if the feature works:

- The App's private key on a runner that runs repo-supplied or third-party
  code, or reaching any child process (`git`, `gh`), a log line, argv or a
  file.
- The hook holding a write token.
- A token left live after the tool has replaced it, or after the tool has
  ended by success, failure or cancellation. A hard kill is the one
  accepted exception (§6.1).
- Run logic for the wait (rounds, retries, state passed between steps) in
  the workflow's YAML.
- A refresh merging from any branch's code other than `develop`'s.
- A silent failure. A refresh PR that cannot merge must leave a red run that
  names the PR. A run must not go green while its PR is still open.
- More than one open refresh PR per repo, or a nightly run that piles a new
  PR on top of an earlier one instead of replacing it.

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
- **How the workflow mints its token today.** `actions/create-github-app-token`
  gives a token every permission the installation holds only when the step
  lists none. Once a step lists any `permission-*` input, the token gets
  exactly those. Epic #1064 scoped every automation token this way.
  `ops-upstream-refresh.yml` lists `contents` and `pull-requests` write;
  `github-config` also lists `workflows` write. Every in-scope org's
  automation App holds Workflows write (epic #1064, D2). The action revokes
  its tokens only when the job ends.

## 5. Approaches

How the job keeps working past the token's one-hour life:

- **A. Python renews the token (chosen).** The tool is given the App's
  client ID and private key, mints its own installation tokens, and renews
  them while it waits. All of the logic sits in the building block, in
  Python, under unit tests. The workflow stays declarative, with no run
  logic, and every later job (lifecycle, an automated `update-deps`) gets
  renewal with no workflow logic of its own. The key is in our tool's
  memory, so the job follows the key boundary (§6): the repo hook, and the
  third-party packages it loads, run in a separate job that never has the
  key. The `develop`-only guard keeps the key to `develop`'s code, as for
  any step that reads the App's secrets today.
- **B. Rounds in the workflow.** The workflow opens the PR in one step, then
  runs a fixed number of await steps, minting a new token before each one,
  with the outcome of each round passed to the next through step outputs
  and `if:` conditions. This keeps the key inside
  `actions/create-github-app-token`. The human rejected it: it puts run
  logic in YAML, which is not a programming language. It cannot be tested
  the way Python is, and it would have to be copied into every job that
  waits, which does not scale.
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

**The key boundary.** This is a rule for every mechanized job that holds
the automation App's private key, now and later. The key exists only in a
process that runs nothing but our own released code. It never shares a
machine with code that the repo supplies or that comes from a third party:
a repo hook, a generator, or any package such code installs or loads.

A job that needs both therefore runs as two GitHub Actions jobs, each on a
fresh runner:

- **prepare** runs that code with no secrets and a read-only
  `GITHUB_TOKEN`, and hands on **data only**: a patch and notes, as an
  artifact;
- **publish** holds the key, runs only our code, and treats what it
  receives as data. It never executes it, and never runs git inside a tree
  that `prepare` touched.

Separate steps in one job are not enough. Steps share the runner's
filesystem, so code in an earlier step could rewrite the tool, its
packages, or a git hook or config that a later step holding the key then
runs.

### 6.1 The building block: an App token that renews itself

`lib/mechanized_pr.py` (or a module beside it) gains an **App token
source**: an object that holds the App's credentials, mints installation
tokens for one repo, and hands out a current token on demand.

- **Inputs.** The App's client ID and private key, the target repo, and the
  permissions the job needs. For upstream-refresh the permissions are
  `contents: write`, `pull-requests: write`, `workflows: write` and
  `metadata: read`. Workflows write is needed even though no hook touches
  `.github/workflows/`: the wait's branch update merges `develop` into the
  PR branch, and GitHub refuses that merge from an App token without
  Workflows write whenever `develop` has changed a workflow file.
- **Minting.** The source signs a short-lived App JWT with the key, looks up
  the App's installation on the repo, and creates an installation token
  limited to that one repo and those permissions. The token's `expires_at`
  comes back with it.
- **Signing in-process, with a new dependency.** The JWT is signed inside
  the tool's process with `cryptography`, added to vergil-tooling's
  dependencies. The token source does not reuse the existing App mode in
  `lib/github.py` (`_load_app_config`, `_generate_jwt`,
  `get_installation_token`). That code reads the key from a file, signs by
  running `openssl` as a child process, and mints installation-wide tokens
  that are cached and never revoked. It stays as it is for the agent VMs.
  Publish refuses to run when that App mode is configured (`VRG_APP_ID` or
  `VRG_PRIVATE_KEY_PATH` set), so the two minting paths can never mix.
- **Renewal.** The source renews the token once it is older than a fixed
  interval, 30 minutes, well inside its one-hour life.
- **Where the token is used: every subprocess's environment.** The token
  source is registered with the shared call layer, so the token is read
  where every `gh` and `git` subprocess's environment is built (`_gh_env` in
  `lib/github.py` and `_git_env` in `lib/git.py`). Each call starts with the
  token that is current at that moment, renewing it first if it is due.
  - `git` gets it through the existing `_git_auth_env(token)` header form,
    which writes nothing to any git config file. That replaces the
    `gh auth setup-git` step the workflow drops (§6.3).
  - Calls made inside `lib/github.py`'s own loops, such as
    `wait_for_checks`, which can poll for over an hour, therefore pick up a
    renewed token on their next call.
  - A retry after HTTP 401 rebuilds the environment, so it gets a current
    token rather than repeating the expired one.
  - With no token source registered (`github-config`, `vrg-release`,
    `vrg-finalize-pr`, every human run), both functions behave exactly as
    they do today.

  A vergil-containers refresh, whose CI takes 32 to 40 minutes, renews at
  least once in a normal run. (Builder's call: the exact interval.
  It must stay well inside the hour, and short enough that a
  vergil-containers run renews.)
- **Revocation.** When the source replaces a token, it revokes the old one
  (`DELETE /installation/token`) at once. When the tool exits, it revokes the
  current one: on success, on failure, and when GitHub's cancel signal
  (SIGINT or SIGTERM) arrives. The tool's own deadline ends safely before
  the job's `timeout-minutes`, so the normal path always reaches that
  revocation. Only a hard kill (the runner lost, or the kill after the cancel
  grace period) skips it. The token is then left to expire within its hour,
  in the memory of a runner that ran no code but ours. A revocation that
  fails fails the run, naming the step.
- **Identity.** Each token the source mints is checked against
  `VRG_AUTOMATION_APP`, as `automation_identity` does today for `GH_TOKEN`.
  The mode resolves to `automation` only after the first token passes.
  A token from any other App fails the run.
- **Key hygiene.** The key reaches the tool only in the publish job, through
  an environment variable set only on the step that runs the tool. No
  repo-supplied or third-party code runs anywhere in that job (§6, the key
  boundary). The tool reads the key once, at startup, and removes the
  variable from its own environment before doing anything else, so its
  child processes (`git`, `gh`) are started without it. It never appears in
  a log line, an exception message, argv or a file. The client ID is not
  secret, but follows the same path.
- **The wait.** With a token source, the building block's wait-until-merged
  deadline is the caller's, not tied to one token. For upstream-refresh it
  is 120 minutes, counted from the moment publish starts, not from the
  start of the wait. The clone, the patch and the push are inside it, so
  publish always ends, and logs how the wait ended, a fixed margin before
  the job's `timeout-minutes`. That covers vergil-containers' CI, with one branch update
  and its CI re-run, with room to spare. It is a ceiling: the wait ends the
  moment the PR merges. The branch-update cap (`_MAX_BRANCH_UPDATES`) and
  every terminal outcome (closed, draft, conflict, failed check) are
  unchanged.
- **Without a token source,** the building block behaves exactly as today:
  it uses the ambient `GH_TOKEN` or the human's credentials, with the
  40-minute default deadline. `github-config`'s `update` is unchanged.

### 6.2 `vrg-upstream-refresh` on the building block

- **Two halves, prepare and publish** (§6, the key boundary).
  - **prepare** takes a fresh clone of `develop`
    (`develop_clone.clone_develop`) and runs the hook there, with the same
    `bash -c` invocation and the same `$VRG_UPSTREAM_REPORT` contract.
    It refuses untracked files left by the hook, as today ("edit tracked
    files only"). It writes the hook's tracked changes as a binary patch
    against the clone's `develop` commit, along with that commit's SHA, the
    changed-path list and the hook's notes, to an output directory. It
    holds no App credentials and needs no write token. It refuses to run
    if the App's key variable is present in its environment, so the
    boundary cannot be broken by a wiring mistake.
  - **publish** reads that output as data. It takes its own fresh clone of
    `develop` and applies the patch with `git apply`. It refuses a patch
    that does not apply cleanly, or that adds untracked paths outside the
    listed changes. It then hands the clone to the building block. It never
    runs the hook, and never runs git in the tree prepare worked in.
  - The scratch linked worktree, the "clean checkout at `origin/<base>`"
    check and `--base` are removed. The base is always `develop`, as with
    every mechanized PR. The tool never touches the caller's checkout.
- **The hook contract is unchanged.** The changed-paths list and the hook's
  notes still make up the PR body. Its intro changes from "a human merges"
  to say that the job merges the PR itself once CI passes. The hook gets no
  GitHub token and never the App's key: neither repo's hook uses one today.
  A future hook that needs to read GitHub gets a read-only token of its
  own, never a write token.
- **Opening the PR.** The job name is `upstream-refresh`, so its branches are
  `automation/upstream-refresh/<run-id>`. The title stays
  `chore(upstream): refresh upstream inputs`. Everything in `sync()` that
  manages the fixed branch is removed: `REFRESH_BRANCH`, adopting
  `upstream-refresh-*`, the `--force-with-lease` push, the
  "unchanged/refreshed" in-place update, and closing extra PRs. A run that
  finds nothing to change still closes an earlier run's open PR, because
  the building block's first step does that.
- **Commit author.** When it runs with a token source, the tool sets the
  clone's commit author and committer to the App's bot user,
  `<app-slug>[bot]`, with the bot user's noreply address, as `github-config`
  does in its workflow today. A human's local run commits as the human.
- **The command line.** The flags' spelling is the builder's call:
  - a prepare mode, writing to an output directory;
  - a publish mode, reading one, then opening the PR and waiting until it
    merges;
  - the default (no mode), which does both in one process. That is for a
    human's local run, where the human's own trust covers the hook. It is
    refused under Actions.

  `--repo` and `--dry-run` stay. `--dry-run` runs prepare and prints the PR
  body, pushing nothing.
- **Exit codes**, stated in `--help`:
  - **0:** merged, or nothing to change;
  - **2:** not merged, naming the PR and the reason, a terminal outcome or
    the deadline;
  - any other failure (the hook failed, the clone failed, a refused
    identity, a failed mint or revocation) exits non-zero with the reason,
    as the tool's errors do today.
- **Identity.** The tool accepts `human` and `automation`, and refuses the
  agent modes, as it does today. Its identity check names `automation`
  explicitly (epic #1064's rule).
- **Under Actions.** When `GITHUB_ACTIONS` is set:
  - publish refuses to run, before touching GitHub, unless it was given the
    App's credentials and resolves to `automation`. A workflow without them
    would otherwise resolve to `human` and run the merging flow with none
    of §6.3's guards;
  - prepare runs without credentials, and refuses if the key is present;
  - the combined default mode is refused.
- **Logging.** Every run prints:
  - the PR it closed, if any;
  - the PR it opened, or "in sync";
  - each token renewal and revocation, with the time (never the token);
  - how the wait ended, with the PR URL and the time waited.

  The log alone must be enough to tell what happened, without opening
  GitHub.

### 6.3 `ops-upstream-refresh.yml`

The reusable workflow has two jobs and no run logic: no `if:` chains, loops
or state passed between steps beyond one artifact.

- **`prepare`** (no secrets; `permissions: contents: read`):
  1. **Guard:** fail unless `github.ref` is `refs/heads/develop` and
     `vars.VRG_AUTOMATION_APP` is non-empty. It reads the variable as
     `AUTOMATION_APP`, as `github-config`'s guard does. A failing step,
     never a job-level `if:`, so a run from another ref shows up red
     instead of being skipped.
  2. **Checkout** of `ref: develop`, with `persist-credentials: false`, only
     to install the tooling.
  3. **Install vergil-tooling** (`setup/vergil`, as today).
  4. **Prepare:** `vrg-upstream-refresh` in prepare mode, writing its output
     directory.
  5. **Upload** that directory as the job's artifact.
- **`publish`** (`needs: prepare`; the App's secrets only on its last
  step):
  1. **Guard**, the same as prepare's.
  2. **Checkout** and **install**, as in prepare, on this job's own fresh
     runner.
  3. **Download** prepare's artifact.
  4. **Publish:** `vrg-upstream-refresh` in publish mode on the downloaded
     directory, with `VRG_AUTOMATION_APP` and the App's client ID and
     private key set on this step only.

  When prepare reports nothing to change, publish still runs. It closes an
  earlier run's open PR and reports "in sync" (§6.2).

`actions/create-github-app-token`, `gh auth setup-git` and the
git-identity step are removed: the tool now does each of them itself (§6.1,
§6.2). The workflow also gets:

- `timeout-minutes` on publish a little above the tool's 120-minute
  deadline (the plan fixes the number), so a hung run cannot hold the
  concurrency group for GitHub's six-hour default;
- `concurrency: group: ${{ github.repository }}/upstream-refresh`,
  `cancel-in-progress: false`, at the workflow level, so a run's prepare
  and publish never interleave with another run's;
- a header comment that loses "it never merges; a human merges", and
  explains the key boundary and the token renewal.

The callers' `ops.yml` files in vergil-containers and vergil-python, and their
`vergil.toml` comments, lose "nothing is auto-merged; a human merges". The
`language-lifecycle` job's comments in vergil-containers' `ops.yml` that
describe its token and pushes as working "as upstream-refresh's" do are
updated too. The callers' wiring is otherwise unchanged: both already pass
`APP_CLIENT_ID` and `APP_PRIVATE_KEY` at `@v2.1`. Whether the caller job
needs any new permission is for the builder to confirm against the reusable
workflow.

**Docs that state the old policy.** Each repo's change updates its own docs
in the same PR. These pages say today that a human merges refresh PRs, or
describe the fixed branch and its token:

- vergil-tooling: `docs/site/docs/reference/dependency-update-config.md`,
  `docs/site/docs/reference/upstream-refresh-config.md`, and
  `vrg-upstream-refresh`'s module docstring and `--help` text ("Never
  merges");
- vergil-actions: `docs/site/docs/workflows/ops-upstream-refresh.md`,
  `docs/site/docs/workflows/index.md`, and the upstream-refresh rows of
  `docs/site/docs/configuration.md`'s permission table;
- vergil-containers: `docs/site/docs/operations/version-management.md`.
  Past release notes are history and stay as they are.

This epic carries out the policy decided in vergil-project/.github#470
(unattended merge of automation-authored PRs under the automation
identity, closed). The documentation-review task sweeps for anything this
list misses.

### 6.4 Rollout

No step can merge a refresh before the guarded workflow is in place:

1. **Tooling.** vergil-tooling releases the token source (§6.1) and the new
   `vrg-upstream-refresh` (§6.2). Both callers install the newest release on
   the `v2.1` line, so from the next nightly the old
   `ops-upstream-refresh.yml` runs the new tool. That run fails loudly and
   merges nothing: `--base` no longer exists, and the tool refuses to run
   under Actions without the App's credentials (§6.2). At most one red
   nightly per repo between steps 1 and 2 is accepted. The old command line
   is not kept working.
2. **Workflow, with the one-time cleanup.** vergil-actions releases the new
   `ops-upstream-refresh.yml` (§6.3). From its next run, each caller runs the
   new flow. The same deployment step, before the next nightly run, closes
   the open PRs on the old fixed `upstream-refresh` branch in both repos and
   deletes their branches, with a closing comment that points to the new
   mechanism. So from the switch on there is never more than one open
   refresh PR per repo. The new tool carries no transitional code for the
   old branch name.
3. **Live validation** (§7).

### 6.5 Local runs

A human running `vrg-upstream-refresh` locally uses the default mode,
prepare and publish in one process, under the human's own credentials, with
no token source. The key boundary does not apply: no App key is involved. Those credentials
are not bound to one hour, and `$VRG_MECHANIZED_PR_DEADLINE` can raise the
wait. The `automation/**` ruleset admits the admin role, so the human's push
is accepted. As with `update` (epic #1064, spec §6.5), the tool works in its
own temporary clone and never touches the human's checkout.

## 7. Validation

A validation task, run after the vergil-actions release, proves G4 live:

- **vergil-python:** a dispatched (or nightly) run either finds nothing to
  change and exits 0, or opens an `automation/upstream-refresh/<run-id>` PR
  as `<org>-automation[bot]`, which merges within the run. The run is green,
  and `develop` carries the commit.
- **vergil-containers:** a run opens a refresh PR, renews its token at least
  once while it waits, and the PR merges. The run is green. Its log shows,
  in order: the renewal, the revocation of the replaced token, polls of the
  PR that succeed, and the merge seen by such a poll. The merge itself is no
  proof of renewal: auto-merge, turned on with the first token, lets GitHub
  merge on its own side. The polls are the proof. Each one ran after the old
  token was revoked, and a revoked token fails at once. A normal run does
  this, with no special settings, because its CI outlasts the renewal
  interval. A write made with a renewed token (a branch update, or the
  tool's own merge call) is covered by unit tests. If one happens during
  the validation, its log line is extra evidence. None is forced.
- **No legacy PRs:** neither repo has an open PR on `upstream-refresh`.
- **The guard:** a dispatch from a branch other than `develop` fails at the
  guard step.

The log lines of §6.2 are the evidence, along with the PR's author and
merge, and the job's conclusion.

## 8. What's left to the builder

- The token source's API, and where it lives in the module family.
- The names of the environment variables that carry the client ID and key
  (not `VRG_APP_ID` or `VRG_PRIVATE_KEY_PATH`, §6.1).
- The exact renewal interval (§6.1).
- The job's exact `timeout-minutes`.

## 9. Follow-ons this epic leaves

- **Lifecycle PRs** (vergil-project/.github#1128): one PR per family, merged
  in order, past one hour through §6.1's token source. It needs its own
  planning issue once this epic's planning issue closes, because that issue
  (vergil-project/.github#1067) also named the lifecycle job.
- **Release after refresh** (vergil-project/.github#1145, second half): high
  priority, blocked on automation release rights and the release lock
  (vergil-project/.github#1088).
- **The key boundary, fleet-wide** (vergil-project/.github#1180): apply §6's
  rule to every job that can reach the App's credentials.
- **`github-config` on the token source**, if its PRs ever need longer than
  one token.
- **Logging the resolved identity mode** (vergil-project/vergil-tooling#4433,
  intake) applies to this job too.
