# Automatic fleet sync through repo-config update — design

- **Epic:** vergil-project/.github#1064
- **Planning issue:** vergil-project/.github#909 (review history is in its comments)
- **Status:** draft

## 1. Intent

A change to a managed file's master copy in vergil-tooling (the consumer
CLAUDE.md block, the Claude settings, a `.gitignore` template, the
`dependabot.yml` render, the hook-guard shim) should reach every repo in the
fleet with no human driving it. Once the change ships in a vergil-tooling
release, each repo's nightly ops job notices that its files have drifted from
the released master copies. It opens one pull request with the fixes and
turns on GitHub auto-merge, and the PR merges into `develop` when CI passes.
The human hears about it only when something breaks.

The epic delivers this by putting the fleet sync inside a new
`vrg-github-repo-config update` command, a superset of `apply`, which is
deprecated. In doing so it takes the first step into new
territory: an **automation identity** whose rights go beyond the agent's,
starting with merging PRs.

## 2. Why

### 2.1 The fleet sync is manual toil

vergil-tooling carries a family of ad-hoc sync tools: `vrg-fleet-sync`
(targets `gitignore`, `dependabot`, `claude-settings`, `claude-md` and
`hook-guard-shim`), the per-target applicators `vrg-gitignore-sync`,
`vrg-dependabot-sync`, `vrg-claude-settings-sync`, `vrg-claude-md-sync` and
`vrg-hook-guard-shim-sync`, and `vrg-ci-sync`. All of them run on the shared
sweep engine `lib/fleet_sweep.py`. No workflow or skill calls them.

A sync today works like this. The human points the tool at local clones. For
each repo that has drifted, the tool files an issue, cuts a feature branch,
commits, and runs `vrg-pr-workflow report-ready`. The human then runs
`vrg-submit-pr --finalize` across every repo. The human does not review
these PRs (except in rare cases). Merging them is pure labor, and the labor
grows with the size of the fleet.

### 2.2 The audit already knows the drift

`vrg-github-repo-config audit` already checks most of the same files
(`lib/repo_config.py`, `audit_local_config`). It reports drift as errors
and stops there. `apply` converges only the GitHub-side state (rulesets,
repo settings, Actions settings). The checking logic and the fixing logic
live in two different tools.

### 2.3 Why mechanized PRs can merge on CI alone

The rule that a human merges every PR exists because PR content was
AI-generated. A sync PR is produced entirely by mechanized code, with no
agent involved. Its risk profile is that of the code that produced it, and
that code was itself reviewed and released. CI is the gate. That gate is the
one the human applies today, by hand.

## 3. The identity invariant

There are three identities, each with its own rights:

| Identity | What it is | Rights |
|---|---|---|
| Agent (`user`, `audit` modes) | AI-driven work, under the `*-vergil-user` / `*-vergil-audit` identities | The narrowest. Stops at `report-ready`; never submits or merges. |
| Automation | The org automation GitHub App (`<org>-automation`, e.g. `vergil-project-automation`) | A subset of the human's rights, broader than the agent's. |
| Human | The human's own credentials | Everything. |

**Invariant:** the automation identity is used **only for mechanized
work, with no AI agent involved.** Any work an AI agent does, automated or
not, runs under the agent identities. The word "automation" is reserved
for this identity and this meaning. The safety of every right this epic
grants the automation identity rests on this invariant.

The automation identity's rights are expected to grow toward the human's
over time (releases and other administrative work come later). This epic
grants the first set: pushing branches, changing workflow files, opening
and refreshing PRs, turning on auto-merge, and converging repo
administration (§6.4).

## 4. Goals, non-goals, anti-goals

**Goals**

- G1. A new `vrg-github-repo-config update` command fixes managed-file drift as well as the
  GitHub-side state.
- G2. Each repo's nightly ops job runs `update` instead of `audit`, as the
  automation identity, and its file fixes arrive as one auto-merging PR.
- G3. A new `automation` identity mode exists in the tooling, detected from
  the automation App's credential, with explicit rights.
- G4. The mechanized-PR flow is a reusable building block that later
  mechanized jobs can use.
- G5. The ad-hoc sync tools and their engine are removed, along with their
  tests.

**Non-goals**

- Moving the other PR-opening jobs (`vrg-update-deps`, upstream-refresh,
  the vergil-containers lifecycle job) onto the building block. That is
  the follow-on brainstorm (vergil-project/.github#1067).
- Alerting the human when an automation PR stays unmerged. That belongs
  to vergil-project/.github#471 and to the repo-status planning work that
  runs in parallel (§9).
- A wider review of identity-mode resolution, including how the human
  default falls through to ambient credentials. That belongs to
  vergil-project/.github#593.
- Releases, or any other new rights for the automation identity beyond
  those listed in §3.

**Anti-goals** (outcomes that count as failure even if the feature works)

- The automation identity ever carrying AI-authored content.
- A nightly run that opens a new PR (or issue) every night for the same
  drift, instead of keeping one PR per repo.
- A silent failure. A sync that cannot be made, or a check that cannot be
  fixed, must leave a visible signal: a red run or an open PR.
- Keeping the old sync tools "just in case" next to the new mechanism.

## 5. Current state (facts the design builds on)

- **Ops wiring.** Every repo's `.github/workflows/ops.yml` calls
  `vergil-actions/.github/workflows/ops-github-config.yml`. Its
  `github-config` job mints an App installation token with no
  permission or repository limits, so the token covers the calling repo
  only, with every permission the installation holds. The job checks out
  the repo and runs
  `vrg-github-repo-config audit --repo "${{ github.repository }}"`. One run
  covers one repo, its own.
- **Tooling version.** The ops job installs vergil-tooling through the
  `setup/vergil` action, at the version pinned in the repo's `vergil.toml`.
  The master copies the job applies are therefore those of that **release**.
- **Master copies** live in `src/vergil_tooling/data/` (`claude_md_consumer.md`,
  `claude_settings.json`, `hook_guard_shim.sh`, `gitignore/*`).
  `dependabot.yml` is rendered in code (`lib/dependabot.py`).
- **Local checks** (`lib/repo_config.py`): `vergil.toml` valid, hook-guard
  shim, CLAUDE.md marker region, CLAUDE.md length, AGENTS.md absent,
  Claude settings, `vergil-*` workflow ref pins, the `.gitignore` managed
  block, `dependabot.yml`, required workflows. Today `apply` reports these
  but never fixes them.
- **The App's documented permissions** (vergil-actions
  `docs/site/docs/configuration.md`): Metadata read, Administration
  read, Contents read/write, Pull requests read/write, Issues read/write,
  Dependabot alerts read. The App also holds Workflows read/write, which only
  vergil-actions' own `cd.yml` uses today. The doc states as policy that
  Administration stays read-only.
- **Other jobs already run as the automation App** and resolve to `human`
  today: `vrg-epic-audit --close` (epic-sweep), `vrg-upstream-refresh`,
  `vrg-branch-sweep`, `vrg-dependabot-audit` and epic-rollup. Release and docs
  CI pass `GITHUB_TOKEN`, which is also an App installation token (`ghs_`),
  to vrg tools.
- **Managed-file checks read the working tree** (`audit_local_config(Path.cwd())`)
  and are skipped when the cwd is not a checkout of `--repo`.
- **App tokens cannot read ruleset `bypass_actors`** at today's permissions.
  `compute_desired_state(app_mode=True)` skips that comparison and reports it
  as not checked.
- **vergil-tooling's own CI and ops** install vergil-tooling from its own
  checkout (`uv sync`), not from a release.
- **Rulesets** (`lib/github_config.py`): `develop` requires 0 approvals and
  passing status checks (strict, up-to-date), with no bypass actors. The
  repo settings set `allow_auto_merge=False` and `delete_branch_on_merge=True`.
- **Identity modes** (`lib/identity_mode.py`): `human`, `user`, `audit`.
  Inside an Actions job holding an App token, none of the resolution
  signals are present, so the mode resolves to `human`. "Human" is not
  asserted anywhere. It is the fall-through to whatever credentials the
  environment already holds. The agent modes are the special case.
- **Prior art for mechanized PRs.**
  - `lib/upstream_refresh.py`: one PR on a fixed branch, refreshed in place,
    superseded PRs closed.
  - The vergil-containers lifecycle `publish.py`: deterministic commits, and
    PRs closed when their change is no longer reported.
  - `vrg-update-deps` preflight: merged / in-flight / stale classification.
  - `lib/pr_merge.wait_and_merge`.
- **Issue linkage** is enforced when vergil-tooling builds a PR body, not by
  CI (the `vrg-pr-issue-linkage` gate was retired). A PR with no linked
  issue does not fail CI for that reason.

## 6. Design

### 6.1 The `automation` identity mode

- Add an `automation` mode to `lib/identity_mode.py`.
- It is **detected from the credential**, not declared. A process whose
  GitHub credential is an installation token of the org automation App
  resolves to `automation`.
- The agents are GitHub Apps too. So the check must establish *which*
  App minted the token (the automation App, as configured for the org), not
  merely that the token belongs to some App. A token from the agent Apps
  never resolves to `automation`.
- **Scope of the check.** Only a credential positively identified as the
  automation App resolves to `automation`. Every other credential resolves
  exactly as it does today: an explicit agent signal resolves to the agent
  mode, and anything else, including `GITHUB_TOKEN` in release and docs
  CI, falls through to `human`. The lookup runs only when an App
  installation token is present and no explicit mode signal is set, so
  ordinary human and agent calls make no extra network request.
- **Fail loudly** when an App installation token is present but the lookup
  that identifies its App fails. The resolver never guesses. An explicit
  mode signal (the environment variable or the mode file) that contradicts
  a credential identified as the automation App is an error too; neither
  side wins.
- How the check works (for example, asking GitHub which App the
  installation token belongs to, or comparing a configured App ID) is left
  to the builder. It must not depend on a flag or environment variable
  that any process could set.
- **Rights in the tooling's own checks.** Some `vrg-*` tools check the mode
  themselves ("refuse unless human", "refuse if agent"). These are separate
  from GitHub's permissions: GitHub grants permissions per App and per
  token (§6.4), and the tooling cannot widen or narrow them. The tooling's
  checks admit `automation` for the operations this epic needs (§3). They
  also keep admitting **the jobs that already run as the automation App**
  (§5: epic-sweep, epic-rollup, upstream-refresh, branch-sweep,
  dependabot-audit). Those jobs resolve as `human` today, and they must
  not start being refused once they resolve as `automation`. Every identity
  check in vergil-tooling is reviewed and states explicitly what it does
  with `automation`; a check that is silent about `automation` is a defect.
  Beyond those, the tooling's checks admit `automation` to nothing by
  default.
- The `human` fall-through stays as it is (§4 non-goals).

### 6.2 The mechanized-PR building block

This is a library module in vergil-tooling for mechanized jobs. It does
not use the agent PR workflow: no issue, no `vrg-pr-workflow` state, no
relay ref, no `vrg-submit-pr`, no `vrg-finalize-pr`. It is called with
the job's identity (its branch name), the target base (`develop`), the
change (the files to write), and the PR title and body. It then:

1. Computes the change against the current tip of the base. If there is no
   change, it closes any open PR the job has on its branch (the drift went
   away), deletes the branch, and reports "in sync".
2. If there is a change, it commits it on the job's fixed branch, built
   fresh from the current base tip each run, and force-pushes only when the
   resulting tree differs from what the branch already holds. If an
   identical branch is already open, the run does nothing.
3. Opens the PR if none is open on that branch; otherwise updates its title
   and body.
4. Turns on GitHub auto-merge for the PR, using the repo's squash method.
5. Returns. It does not wait for CI or for the merge.

Exactly one PR is open per job per repo. Rebuilding from the base tip each
night also handles a PR that has fallen behind `develop`. GitHub auto-merge
does not update a branch that is behind, and `develop` requires branches
to be up to date, so such a PR waits until the next night's rebuild. A
one-day delay is accepted.

**Branch naming is a shared interface.** Every mechanized job's branch
uses a prefix that marks it as automation (for this job, a name of the
shape `automation/<job>`, e.g. `automation/repo-config-sync`). The
repo-status work (§9) relies on the prefix to tell automated PRs from agent
development PRs. The exact name is fixed in the plan and recorded so both
epics use it.

This is the reusable building block (G4). The prior art in §5 is the
source to draw on. The follow-on brainstorm moves the other jobs onto it.

### 6.3 The new `update` command fixes managed files

- **`update` is a new subcommand** of `vrg-github-repo-config`. It is a
  superset of `apply`: the same GitHub-side convergence, plus the managed-file
  sync below. Both share their code paths.
- **`apply` is deprecated, and its behavior is unchanged.** It still
  converges only the GitHub side, and still reports file drift without
  fixing it. Its new identity check (below) is `update`'s alone, and it
  prints a deprecation notice pointing at `update`. Removing `apply` is a
  later change once nothing calls it; this epic only deprecates it. The one
  thing `apply` picks up is the shared desired-state change (§6.4,
  auto-merge), because that is a change to the desired state, not to the
  command.
- Because a new command carries the new behavior, releasing the tooling
  changes nothing in the fleet. The fleet switches only when ops is changed
  to call `update` (§6.4, §6.7).
- Each local check that has a fixer gains an applicator, in the same module
  family as the check. The applicator writes the file to match the released
  master copy:
  - the hook-guard shim;
  - the CLAUDE.md marker region;
  - the Claude settings;
  - the `.gitignore` managed block;
  - `dependabot.yml`;
  - the `vergil-*` workflow ref pins.

  The check and its applicator share one definition of "correct", so they
  cannot disagree.
- **The checks evaluate the `develop` tip on GitHub, not the working tree.**
  The fixable checks and the ones with no fixer both read the base the
  PR targets. That makes `audit`, `update` and the PR agree, and lets
  `update --repo X` sync a repo with no local checkout. The check and
  applicator code works on a file tree fetched from GitHub. `audit` follows
  the same rule, so a human's feature-branch edits never mask or invent
  drift on `develop`.
- **Identity.** `update` refuses to run in the agent modes (`user`, `audit`)
  and says why. It runs for `human` and `automation`.
- **`bypass_actors`.** Once the App holds Administration write, the plan
  first checks whether it can read ruleset `bypass_actors`. If it can, an
  App-run `update` audits and converges them like every other field. If it
  cannot, an App-run `update` does not write that field, because it never
  writes what it cannot verify, and reports it as not checked, as
  `audit` does today. That one field stays converged by a human `update`.
- `update` runs the GitHub-side convergence as it does today. It then
  collects every file fix for the repo into **one change** and hands it to
  the building block (§6.2), giving one PR per repo however many files
  drifted.
- Checks with no fixer still fail the run: a missing or invalid
  `vergil.toml`, a CLAUDE.md over the length limit, an AGENTS.md that
  shouldn't exist, missing required workflows. A file fix that cannot be
  computed fails the run too.
- `update` exits 0 when the GitHub side converged and the file state is
  in sync, or a sync PR is open with auto-merge on. It exits non-zero
  when anything could not be converged, fixed or audited.
- `audit` and `diff` keep their read-only meaning.
- The applicators port what is still useful from the per-target sync
  applicators. `vrg-ci-sync`'s matrix-input stripping has no audit check
  behind it; it was a one-time migration and is not ported.

### 6.4 Fleet changes

- **App permissions.** The automation App gains **Administration:
  read/write** (so `update` can converge rulesets and settings). It already
  holds **Workflows: read/write**. That permission is now used for syncs that
  change files under `.github/workflows/`, and the doc says so. The human
  changes the App's settings in each org, and each installation must accept
  the new permission. vergil-actions'
  `configuration.md` drops its "Administration stays read-only" policy and
  documents the new permissions and why, citing §3.
- **Token scoping (rule).** GitHub grants an App's permissions to every
  token minted from it unless the token asks for less, so once the App holds
  Administration write, any job minting an unscoped token holds it too.
  Therefore **every workflow that mints an automation App token requests
  only the permissions that job needs** (the `permission-*` inputs of
  `actions/create-github-app-token`), as `branch-sweep` and
  `dependabot-audit` already do. Only the `github-config` job that runs
  `update` requests Administration write. The epic inventories every place
  in the fleet that mints an automation token (vergil-actions' reusable
  workflows and any repo-local workflow) and scopes each one before the App
  is granted Administration write (§6.7, step 2).
- **Auto-merge.** The repo config desired state sets
  `allow_auto_merge=True`. The first `update` after the release turns it on
  fleet-wide. Agents still cannot turn on auto-merge for a PR, because
  `vrg-gh` and the plugin hooks block it, so the agent boundary holds.
- **Ops runs `update`.** `ops-github-config.yml`'s `github-config` job runs
  `vrg-github-repo-config update`, with a token scoped by the rule above.
- **The trust note in `_resolve_config`.** Its claim that only a human runs
  `apply` does not hold for `update`, so it is rewritten. In an ops run, the
  config comes from `develop` (merged, CI-gated content). The local-copy
  fallback still applies only when the cwd matches the target repo.

### 6.5 Local runs

A human running `update` locally gets the same behavior: the same one PR,
with auto-merge on. Once it merges, the human runs `vrg-finalize-pr` on the
merged PR to bring the local `develop` up to date; `vrg-finalize-pr` already
handles a PR that has already been merged. The building block does not
touch any local branch or worktree.

### 6.6 Retirement

Removed, with their tests, entry points and docs:

- `vrg-fleet-sync`;
- `vrg-gitignore-sync`, `vrg-dependabot-sync`, `vrg-claude-settings-sync`,
  `vrg-claude-md-sync` and `vrg-hook-guard-shim-sync`;
- `vrg-ci-sync` and `lib/ci_yml_applicator.py`;
- `lib/fleet_sweep.py`.

Applicator code that §6.3 reuses moves into the repo-config module family
before the old files go. They are removed only once `update` covers every
target they handled, except the `ci.yml` matrix stripping (§6.3).

### 6.7 Rollout

The rollout ships as patch releases in the current line. The new command
keeps each step inert until the next one switches it on:

1. **Tooling.** vergil-tooling releases the `automation` mode, the building
   block, and `update`. Ops still runs `audit`, so nothing changes in the
   fleet.
2. **App permissions** (a human action), once every automation token in
   the fleet is scoped (§6.4) and those workflow changes are released. The
   human grants Administration
   write to the automation App in each org, and each installation accepts
   it.
3. **Prove it on one repo.** Run `update` as the automation App on one
   repo, by a manual dispatch, and confirm that its sync PR opens with
   auto-merge on and merges once CI is green. Also settle the
   `bypass_actors` check (§6.3).
4. **Ops switch.** vergil-actions releases the change that makes ops run
   `update` instead of `audit`. From the next night on, every repo on that
   line syncs.
5. **Retirement** (§6.6) follows once `update` is live.

Steps 2 and 3 are operational tasks (deployment and validation), not code
PRs. Nothing runs `update` as the automation App before step 2 is done.

## 7. Failure model

| Failure | Signal |
|---|---|
| `update` cannot converge the GitHub side, audit a setting, or compute a fix | The nightly ops run goes red. |
| A check with no fixer fails | The nightly ops run goes red. |
| The sync PR's CI fails, conflicts, or is refused | The PR stays open and unmerged. The next night refreshes it. |
| An App token is present but its App cannot be identified, or the mode signals contradict it | The run goes red; it never guesses (§6.1). |

No issues are filed for any of these. An open, unmerged automation PR is as
easy to alert on as a red run. Alerting itself is out of scope (§4).

## 8. Constraints

- No silent failures (the fleet engineering principle): every path in
  §7 leaves a red run or an open PR, and nothing is swallowed.
- Trigger: a **released** vergil-tooling (the pinned version in each repo).
  Target: `develop`, where the managed files take effect. The target repo
  needs no release for the sync to count. The one exception is
  vergil-tooling itself. Its ops job runs its own `develop` code, which is
  also where the master copies live, so it syncs its own files to
  unreleased master copies. That is accepted, because nothing leaves the
  repo.
- One PR per job per repo, at most.
- Agents keep their current boundary: no submit, no merge, no auto-merge.
- Public logs and PR bodies carry no secrets.

## 9. Related work

- **Repo-status planning (in parallel).** A generic repo-status tool will
  replace `vrg-worktree-status`. It shows what has merged to `develop`
  and is waiting for `main`, and it keeps automated PRs apart from agent
  development PRs. It relies on the automation branch prefix in §6.2.
- vergil-project/.github#471: noticing automation PRs and red ops runs.
- vergil-project/.github#593: identity-mode resolution across the gated
  tools.
- vergil-project/.github#1067: follow-on brainstorm. It moves
  `vrg-update-deps`, upstream-refresh and the lifecycle job onto the
  building block.

## 10. Left to the builder

- How `automation` detection works (§6.1), within its stated rules.
- Where the building block lives and what its API looks like.
- The exact branch name, PR title and body, and commit message format, as
  long as the branch prefix is fixed and recorded (§6.2).
- The commit author identity for sync commits, and whether commits are
  made deterministic (the lifecycle job's pattern).
- The exact permission set each scoped token requests, within the §6.4 rule.
- How the code tasks are split within each rollout step (§6.7).
