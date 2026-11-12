# Automatic fleet sync through repo-config update — design

- **Epic:** vergil-project/.github#1064
- **Planning issue:** vergil-project/.github#909 (review history is in its comments)
- **Status:** draft, in design review

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
and replacing PRs, turning on auto-merge, and converging repo
administration (§6.4).

## 4. Goals, non-goals, anti-goals

**Goals**

- G1. A new `vrg-github-repo-config update` command fixes managed-file drift as well as the
  GitHub-side state.
- G2. Each repo's nightly ops job runs `update` instead of `audit`, as the
  automation identity, and its file fixes arrive as one auto-merging PR.
- G3. A new `automation` identity mode exists in the tooling, detected from
  the automation App's credential, with explicit rights.
- G4. The mechanized-PR flow is a reusable building block and the fleet
  standard for every automated job that opens a PR (§6.2).
- G5. The ad-hoc sync tools and their engine are removed, along with their
  tests.
- G6. The repo-status tool (from epic vergil-project/.github#1049) shows PRs
  on the automation prefix as automated (§9).

**Non-goals**

- Moving the other PR-opening jobs (`vrg-update-deps`, upstream-refresh,
  the vergil-containers lifecycle job) onto the building block, and moving
  any job other than `github-config` (including the release workflows)
  onto the `automation` identity mode. That is the follow-on brainstorm
  (vergil-project/.github#1067).
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
  Dependabot alerts read. The vergil-project App also holds Workflows read/write, which
  vergil-actions' own `cd.yml` uses; the doc does not list it for other orgs. The doc states as policy that
  Administration stays read-only.
- **Other jobs already run as the automation App** and resolve to `human`
  today: `vrg-epic-audit --close` (epic-sweep), `vrg-upstream-refresh`,
  `vrg-branch-sweep`, `vrg-dependabot-audit` and epic-rollup. Release and docs
  CI pass `GITHUB_TOKEN`, which is also an App installation token (`ghs_`),
  to vrg tools.
- **Managed-file checks read the working tree** (`audit_local_config(Path.cwd())`)
  and are skipped when the cwd is not a checkout of `--repo`.
- **App tokens cannot read ruleset `bypass_actors`** at today's permissions.
  `compute_desired_state(app_mode=True)` sets the desired field to `None`, so
  it is skipped: printed, but with no effect on the exit code. Separately, a
  setting that cannot be read (`NotChecked`) makes `audit` exit 2 and makes
  `apply` refuse to converge anything.
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
  CI, falls through to `human`.
- **Where the lookup runs.** A job that runs as the automation identity
  sets a marker in its environment saying it holds the automation App
  token. **In this epic only the `github-config` ops job, which runs
  `update`, sets it.** Every other job that mints an automation token today
  (the other ops jobs, epic-sweep, epic-rollup, upstream-refresh, and the
  release workflows `cd.yml` and `cd-release.yml`) does not set it. Those
  jobs keep resolving exactly as they do today, and are migrated later (§4,
  vergil-project/.github#1067). The lookup runs only when the marker is
  present, and it
  verifies that the credential really belongs to the automation App. The
  marker alone grants nothing: a marker without the App's credential fails
  the verification, and the run fails. Jobs that use `GITHUB_TOKEN`, and
  ordinary human and agent calls, never set the marker, so they make no
  extra network request and gain no new way to fail. The lookup retries
  transient errors before it fails.
- **Fail loudly** when the marker is present but the lookup that
  identifies the App fails, or identifies a different App. The resolver never guesses. An explicit
  mode signal (the environment variable or the mode file) that contradicts
  a credential identified as the automation App is an error too; neither
  side wins.
- **Which App is the automation App** is configuration, not code. Each org
  has exactly one automation App, and its name is set once, as an
  **org-level Actions variable**, when the org's jobs are set up, next to
  the App's secrets. It is a variable, not a secret, because the name is not
  sensitive and a local human run must be able to read it back. Workflows
  pass it to their jobs. A local human run reads it through the API (org
  admin). Only an org admin can change it. The tooling turns the name into
  the App's numeric ID at run time through GitHub's public App lookup by
  name. Both the identity check and the automation-prefix ruleset's bypass
  actor (§6.2) use that ID. A missing variable, or a lookup that fails, fails
  the run. This is the first ruleset bypass actor that differs per org:
  today's bypass actors (the admin role, the GitHub Actions App) have the
  same ID everywhere.
- How the check works (for example, asking GitHub which App the
  installation token belongs to and comparing it with the configured App)
  is left to the builder. The marker only decides whether the verification runs.
  The verification itself must not depend on a flag or environment variable
  that any process could set.
- **Rights in the tooling's own checks.** Some `vrg-*` tools check the mode
  themselves ("refuse unless human", "refuse if agent"). These are separate
  from GitHub's permissions: GitHub grants permissions per App and per
  token (§6.4), and the tooling cannot widen or narrow them. The tooling's
  checks admit `automation` for exactly the operations the `github-config`
  job performs through `update` (§3), and to nothing else. Because no other
  job sets the marker, no existing job changes mode in this epic. Every
  identity check that `update`'s code path passes through states explicitly
  what it does with `automation`; a check on that path that is silent about
  `automation` is a defect.
- The `human` fall-through stays as it is (§4 non-goals).

### 6.2 The mechanized-PR building block

This is a library module in vergil-tooling for mechanized jobs. It does
not use the agent PR workflow: no issue, no `vrg-pr-workflow` state, no
relay ref, no `vrg-submit-pr`, no `vrg-finalize-pr`. It is called with
the job's name, the target base (`develop`), the change (the files to
write), and the PR title and body. It then:

1. Computes the change against the current tip of the base.
2. Closes any open PR the job left from an earlier run and deletes its
   branch. A new run always replaces it, never builds on it.
3. If there is no change, reports "in sync" and stops.
4. If there is a change, creates a **new branch for this run** from the
   current base tip, named with the automation prefix, the job name and a
   unique run identifier (for example `automation/repo-config-sync/<run-id>`),
   commits the change there, and opens the PR.
5. Turns on GitHub auto-merge for the PR, using the repo's squash method.
6. Returns. It does not wait for CI or for the merge.

At most one PR is open per job per repo. Ops runs of a job never overlap
(§6.4, concurrency). A local human run that happens to overlap an ops run is
accepted. The PR mechanism already catches it: one PR merges, and the other
conflicts or is replaced by the next run, as with two developers making the
same change. The building block adds no locking or close-the-others step
of its own. The branch is deleted when the
PR merges, by the repo's delete-on-merge setting, or when a later run
closes the PR. Because every run starts from the current base tip, a PR
that has fallen behind `develop` is replaced by the next night's run.
GitHub auto-merge does not update a branch that is behind, and `develop`
requires branches to be up to date, so such a change waits a day. That
delay is accepted.

**Only the automation App writes automation branches.** The desired state
gains a branch ruleset on the automation prefix (`refs/heads/automation/**`).
Its bypass actors are the automation App and the repository admin role.
Only they may create those branches or push to them (including
force-push). Every other identity, including every agent App, is
refused. The ruleset does **not** restrict deletion. Automation branches
are ordinary temporary branches, not protected long-lived ones like
`develop` and `main`: delete-on-merge removes them, and `vrg-branch-sweep`
(already run nightly by ops) cleans up any that are left. Branch-sweep
skips every branch a ruleset targets today, so it is changed so that this
ruleset no longer exempts finished automation branches. So the content of an
automation PR is what the automation, or the human admin, pushed, and never
an agent's. The admin role is included as a rule: an identity-restricting
ruleset never locks out the admin, so the human can always repair things
from the CLI, and a human `update` (§6.5) works the same as the App's. The ruleset's bypass actor is converged like
every other `bypass_actors` field (§6.3).

**Branch naming is a shared interface.** Every mechanized job's branch
uses the automation prefix, and the ruleset above protects that prefix.
The repo-status tool uses it to tell automated PRs from agent
development PRs (§9). The exact prefix and naming scheme are fixed in the
plan.

**This is the fleet standard for mechanized PRs** (G4). Every automated job
that opens a PR, now and in the future, uses it: a fresh branch per run
under the protected automation prefix, one PR at a time, auto-merge, and
deletion on merge. That includes `vrg-update-deps` and its dependency
branches, upstream-refresh, and the vergil-containers lifecycle job. This
epic defines the standard and its first user. Moving the existing jobs
onto it is the follow-on brainstorm (vergil-project/.github#1067). No new
automated job opens a PR any other way. The prior art in §5 is the source
to draw on.

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
  opens no PRs and syncs no files. The fleet switches only when ops is
  changed to call `update` (§6.4, §6.7). The one fleet-visible change is the
  auto-merge setting, which a human sweep handles (§6.7, step 4).
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
  cannot disagree. "Correct" covers the git file mode as well as the bytes.
  The file tree the checks read, and the change handed to the building block,
  carry each file's mode. The hook-guard shim must be mode 100755: its check
  treats any other mode as drift, and its applicator writes 100755, as the
  retired `vrg-hook-guard-shim-sync` did.
- **The checks evaluate the `develop` tip on GitHub, not the working tree.**
  The fixable checks and the ones with no fixer both read the base the
  PR targets. That makes `audit`, `update` and the PR agree, and lets
  `update --repo X` sync a repo with no local checkout. The check and
  applicator code works on a file tree fetched from GitHub. `audit` follows
  the same rule, so a human's feature-branch edits never mask or invent
  drift on `develop`.
- **Identity.** `update` refuses to run in the agent modes (`user`, `audit`)
  and says why. It runs for `human` and `automation`.
- **`bypass_actors`.** The automation App holds Administration write on
  every repo in its org, so an App-run `update` reads, audits and converges
  ruleset `bypass_actors` like every other field, including the
  automation-prefix ruleset's (§6.2). The `app_mode` skip of that field
  (§5) does not apply to the automation identity. If the field cannot be
  read, that is a fatal error: the run fails and converges nothing. It is
  never skipped. GitHub returns an empty list rather than an error to a
  token that cannot see the field, so **unreadable is detected, not
  assumed.** A ruleset whose desired bypass list is non-empty, and which
  reads back with an empty one, counts as unreadable. The tag-protection and
  automation-prefix rulesets (§6.2) both have non-empty bypass lists, so the
  check always has something to test. Whether GitHub exposes `bypass_actors` to an App token with
  Administration write is confirmed before any code relies on it (§6.7, step 3).
  If it does not, the epic stops there for a human decision.
- `update` runs the GitHub-side convergence as it does today. It then
  collects every file fix for the repo into **one change** and hands it to
  the building block (§6.2), giving one PR per repo however many files
  drifted.
- Checks with no fixer still fail the run: a missing or invalid
  `vergil.toml`, a CLAUDE.md over the length limit, an AGENTS.md that
  shouldn't exist, missing required workflows. A file fix that cannot be
  computed fails the run too. Such a failure does not hold back the fixes
  that can be made: `update` still opens the sync PR for the fixable drift,
  then exits non-zero for the rest. A GitHub-side setting that cannot be
  read (§6.3, `bypass_actors`) is different: it stops the run before
  anything is converged or opened.
- `update` exits 0 when the GitHub side converged and the file state is
  in sync, or a sync PR was opened this run and auto-merge is confirmed on
  for it. It exits non-zero
  when anything could not be converged, fixed or audited.
- `audit` and `diff` keep their read-only meaning.
- The applicators port what is still useful from the per-target sync
  applicators. `vrg-ci-sync`'s matrix-input stripping has no audit check
  behind it; it was a one-time migration and is not ported.

### 6.4 Fleet changes

- **App permissions.** Every org's automation App is meant to be
  configured identically, and this epic asserts what that configuration is:
  the permissions already documented, plus **Administration: read/write**
  (so `update` can converge rulesets and settings) and **Workflows:
  read/write** (so syncs can change files under `.github/workflows/`). The
  human checks each org's App against that list, corrects any difference,
  and each installation accepts the change (§6.7, step 2). vergil-actions'
  `configuration.md` is out of date: its permission table has no Workflows
  row, and it says only one org's App needs Workflows. It is rewritten to
  state the asserted configuration. vergil-actions'
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
  `allow_auto_merge=True`. The human's sweep right after the tooling
  release turns it on fleet-wide (§6.7, step 4). Agents still cannot turn on auto-merge for a PR, because
  `vrg-gh` and the plugin hooks block it, so the agent boundary holds.
- **Ops runs `update`.** `ops-github-config.yml`'s `github-config` job runs
  `vrg-github-repo-config update`, with a token scoped by the rule above.
- **Concurrency.** Each job in `ops-github-config.yml` gets its own
  per-repo `concurrency:` group (for example `<repo>/github-config`), which
  queues rather than cancels. A job never overlaps itself, whether started
  by the schedule or by dispatch, and the jobs stay independent of each
  other.
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

1. **Token scoping.** vergil-actions releases the change that scopes every
   automation App token in the fleet (§6.4). It is independent of the
   tooling and changes no behaviour.
2. **App permissions** (a human action), once step 1 is released. The human
   checks each org's automation App against the asserted configuration
   (§6.4), grants Administration write, and grants Workflows write where it
   is missing. Each installation accepts the change. The human also sets
   each org's automation-App name variable (§6.1).
3. **Confirm the premise** (a check, before any code relies on it). With an
   automation App token, read a ruleset whose bypass list is known to be
   non-empty (tag protection lists the admin role) and confirm that the
   list reads back correctly (§6.3). If it does not, the epic stops here
   for a human decision.
4. **Tooling, then a human sweep the same day.** vergil-tooling releases the
   `automation` mode, the building block, and `update`. Ops still runs
   `audit`. The release also carries the desired-state change
   `allow_auto_merge=True` (§6.4), which `audit` and `vrg-release`'s preflight
   read too. So from the release on, every repo whose auto-merge setting is
   still off fails `audit`. Right after the release, the human runs `update`
   (or `apply`) across the fleet, which turns the setting on everywhere.
   Allowing auto-merge as a repo setting is harmless by itself, and agents
   cannot turn it on for a PR. A sweep with `update` may open the first sync
   PRs early, under the human's identity; that is harmless. The red
   window is limited to the hours between the release and the sweep.
5. **Ops switch.** vergil-actions releases the change that makes ops run
   `update` instead of `audit`, and that sets the automation marker on the
   `github-config` job (§6.1). The marker ships in this release and no
   earlier one, so the job runs as `automation` only after the grant
   (step 2) and the premise check (step 3). This switches the whole fleet at once.
   There is no canary mechanism: the fleet is small, and a failed sync
   breaks nothing; at worst a repo fails to update.
6. **Validate it live.** On the day of the switch, the human dispatches one
   repo's ops workflow and confirms that:
   - the run is green;
   - its sync PR, if there is drift, opens on an automation branch with
     auto-merge on, and merges once CI is green;
   - the App converges `bypass_actors` (§6.3), including the
     automation-prefix ruleset's.

   Then the human dispatches ops for every repo, or lets the night's runs
   go, and triages what breaks. A failure is either fixed forward or the
   ops switch is reverted to `audit` while it is fixed.
7. **Retirement** (§6.6) follows once `update` is live.

Steps 2 and 3 are operational tasks (deployment and validation), and so is
step 6. None of them is a code PR. Nothing runs `update` as the automation
App before step 2 is done.

## 7. Failure model

| Failure | Signal |
|---|---|
| `update` cannot converge the GitHub side, audit a setting, or compute a fix | The nightly ops run goes red. |
| A check with no fixer fails | The nightly ops run goes red. |
| The sync PR's CI fails, conflicts, or is refused | The PR stays open and unmerged. The next night closes it and opens a fresh one. |
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
- One PR per job per repo, at most, for ops runs (§6.2, §6.4).
- Agents keep their current boundary: no submit, no merge, no auto-merge.
- Public logs and PR bodies carry no secrets.

## 9. Related work

- **Repo-status (epic vergil-project/.github#1049) comes first.** Its
  repo-status tool replaces `vrg-worktree-status`. It shows what has
  merged to `develop` and is waiting for `main`, and it gives each row a
  type, with branch-based types today only for `deps/update-*` and
  `release/post-*`. That epic is implemented before this one starts, so
  this epic builds on it as shipped. This epic adds a small change to the
  repo-status tool: it recognizes the automation prefix (§6.2) and shows
  those PRs as automated, apart from agent development PRs. When the
  follow-on (vergil-project/.github#1067) moves other jobs, such as
  `vrg-update-deps`, onto the automation prefix, their branch-based types
  move with them.
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
