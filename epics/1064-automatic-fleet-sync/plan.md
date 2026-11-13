# Automatic fleet sync through repo-config update Implementation Plan

- **Epic:** vergil-project/.github#1064
- **Planning issue:** vergil-project/.github#909 (review history is in its comments)
- **Status:** approved

> **For agentic workers:** each task is one PR, implemented with the
> `issue-implement` skill (operational tasks with `issue-deploy` or
> `issue-validate`). Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** each repo's nightly ops job runs `vrg-github-repo-config update`
as the org automation App. When the repo's managed files have drifted from
the released master copies, it opens one auto-merging PR on a fresh,
push-protected `automation/` branch. The ad-hoc `vrg-*-sync` tools are
retired.

**Architecture:** vergil-tooling gains four things:

- an `automation` identity mode, switched on by the `VRG_AUTOMATION_APP` environment variable, which names
  the App and verified against the token;
- a fresh shallow clone of the repo's `develop`, in which the managed-file
  checks and their new fixers run;
- a mechanized-PR library that commits and pushes from that clone with
  plain git, opens the PR and turns on auto-merge;
- an `update` subcommand that joins them.

vergil-actions scopes every automation token, serializes the ops jobs, and
finally switches `github-config` from `audit` to `update`. The human-gated
rollout steps are operational tasks.

**Tech stack:** Python 3 (vergil-tooling `src/vergil_tooling`, pytest under
`tests/vergil_tooling`), GitHub REST and GraphQL through `lib/github.py`,
and GitHub Actions YAML (vergil-actions, vergil-containers).

**Spec:** [`epics/1064-automatic-fleet-sync/spec.md`](spec.md)

## Global constraints

- The automation identity is used only for mechanized work, never for
  anything an AI agent does (spec §3).
- In this epic only the `github-config` ops job sets `VRG_AUTOMATION_APP`.
  Every other job resolves exactly as it does today.
- The `VRG_AUTOMATION_APP` environment variable and the org-level Actions variable are both named
  `VRG_AUTOMATION_APP`, and both hold the automation App's slug.
- Automation branches are named `automation/<job>/<run-id>`. This epic's job
  name is `repo-config-sync`.
- The automation-prefix ruleset covers `refs/heads/automation/**`. Its
  bypass actors are the automation App (`Integration`) and the admin role
  (`RepositoryRole` 5). It restricts creation, update and non-fast-forward,
  never deletion.
- The desired state sets `allow_auto_merge=True`. Sync PRs are
  squash-merged.
- At most one sync PR is open per repo for ops runs. No locking is added
  beyond the per-job Actions concurrency groups.
- Under the automation identity, an unreadable `bypass_actors` field is
  fatal. It is never skipped.
- `apply` keeps today's behaviour and prints a deprecation notice.
- All file work, checks, fixes, commit and push, happens with plain git
  in a fresh shallow clone of `develop` in a temporary directory. No GitHub
  API is used for file contents or commits.
- No silent failures: every failure is a non-zero exit with the reason
  named.
- Required configuration is asserted, not re-checked. The human's token
  holds full rights, `admin:org` included, and a human run reads the org
  variable through the org endpoint. Every managed repo is org-owned and
  configured at org level. A missing right or setting fails with the API's
  error. No task adds a probe, fallback or special case for configuration
  that this plan requires.
- A nightly ops run that goes red for a day or two during the rollout is
  accepted and ignored. No step exists only to keep nightly runs green.
  For example, vergil-tooling's own nightly `audit` is red from Task 5's
  merge until Task D3.
- The `github-config` and `bypass-probe` jobs, the two jobs this epic gives
  write rights to, run only from `develop`. Each checks out `ref: develop`
  and fails when dispatched from any other ref. Other jobs are unchanged.
- The only validation command is `vrg-container-run -- vrg-validate`. Every
  "run the tests" step below means that command.
- Commits use `vrg-commit`, and handoff uses `vrg-pr-workflow
  report-ready`. Changes under `.github/workflows/**` are pushed by the
  human's `vrg-submit-pr`.

## Review focus

1. **A repo with drift in several managed files and a failing no-fixer
   check at once.** Expected: one PR with every fixable file, then exit 1
   naming the no-fixer check. Test in Task 6.
2. **A ruleset that reads back with an empty bypass list while its desired
   list is non-empty, under the automation identity.** Expected: fatal, with
   nothing converged and no PR opened. Test in Task 5.
3. **`VRG_AUTOMATION_APP` set, but the token belongs to a different App or to
   `GITHUB_TOKEN`.** Expected: resolution fails, naming both identities. Test
   in Task 1.
4. **An earlier run's sync PR still open, and the drift now gone.**
   Expected: that PR is closed, its branch is deleted, and the result is "in
   sync". Test in Task 4.
5. **The hook-guard shim with correct bytes but not executable.** Expected:
   it is reported as drift and fixed to mode 0755. Test in Task 3a.

---

## Delivery

| Task | Repo | What | Blocked by |
| --- | --- | --- | --- |
| A1 | vergil-actions | Scope every automation token, add per-job concurrency, rewrite `configuration.md` | — |
| C1 | vergil-containers | Scope the automation tokens in `ops.yml` | — |
| D1 | vergil-actions (deployment) | Release the token scoping and confirm consumers pick it up | A1, C1 |
| D2 | .github (deployment) | Assert every org's App configuration, grant Administration and Workflows write, set `VRG_AUTOMATION_APP` | D1 |
| A1b | vergil-actions | The bypass probe (dispatch-only, `develop`-only) | D2 |
| V1 | .github (validation) | An automation token reads `bypass_actors` correctly | A1b |
| 1 | vergil-tooling | The `automation` identity mode | V1 |
| 2 | vergil-tooling | `audit` and `update` run in a fresh clone of `develop` | — |
| 3a | vergil-tooling | Fixers (shared with their checks): shim, CLAUDE.md, Claude settings | 2 |
| 3b | vergil-tooling | Fixers (shared with their checks): `.gitignore`, `dependabot.yml`, workflow ref pins | 3a |
| 4 | vergil-tooling | The mechanized-PR library | 2 |
| 5 | vergil-tooling | Desired state: auto-merge, the automation-prefix ruleset, fatal bypass reads | 1, V1 |
| 6 | vergil-tooling | `vrg-github-repo-config update`; `apply` deprecated | 1, 3a, 3b, 4, 5 |
| 8 | vergil-tooling | `vrg-repo-status` shows automation PRs as `automation` | — |
| D3 | vergil-tooling (deployment) | Release the tooling; the human's same-day `update` sweep | 6, 8 |
| A2 | vergil-actions | Ops switch: `github-config` runs `update` as automation | D3 |
| D4 | vergil-actions (deployment) | Release the ops switch | A2 |
| V2 | vergil-actions (validation) | Live validation on switch day | D4 |
| 9 | vergil-tooling | Retire the sync tools | V2 |

Advisory hand-off: each advisory is retargeted to the task that fixes it.

| Task | Advisories |
| --- | --- |
| 1 | GHSA-ggw3-hcf8-cqxh |
| 4 | GHSA-596v-vphr-cpqq |
| 5 | GHSA-mv3r-qfr4-7fr6 |
| A2 | GHSA-2422-4hw9-c34c, GHSA-52rq-x5gv-6jg2 |
| D2 | GHSA-jp69-53fx-f5j8 |

### Task A1: Scope every automation token, add per-job concurrency, rewrite the App docs

**Files:**

- Modify: `.github/workflows/ops-github-config.yml`, `ops-epic-sweep.yml`,
  `ops-epic-rollup.yml`, `ops-upstream-refresh.yml`, `cd.yml`,
  `cd-release.yml`
- Modify: `docs/site/docs/configuration.md`,
  `docs/site/docs/workflows/ops-github-config.md` (each job's scoped token,
  the concurrency groups, and `gh auth setup-git`)

**Interfaces:**

- Produces: every `actions/create-github-app-token` step in vergil-actions
  carries explicit `permission-*` inputs, and every job in
  `ops-github-config.yml` has a `concurrency:` group named
  `${{ github.repository }}/<job>` with `cancel-in-progress: false`.

- [ ] **Step 1: Inventory.** List every `create-github-app-token` step and
  the API calls its job makes. Record each step's needed permissions in the
  PR body. Scope `github-config` now to Administration read and Contents
  read, which is what `audit` reads. Task A2 widens it. No Issues
  permission: the repo-config code makes no Issues call.
- [ ] **Step 2: Scope each step.** Add `permission-*` inputs matching the
  inventory. Leave `branch-sweep` and `dependabot-audit`, which are already
  scoped, unchanged.
- [ ] **Step 3: Add concurrency.** Add the per-job `concurrency:` groups to
  `branch-sweep`, `github-config` and `dependabot-audit`.
- [ ] **Step 3b: Give git the job's token.** In the `github-config` job, add
  a `gh auth setup-git` step (with `GH_TOKEN` set to the App token) before
  the tool runs. Then the `git clone` and `git push` that the tooling runs
  authenticate as the App.
- [ ] **Step 4: Rewrite `configuration.md`.**
  - The permission table becomes the asserted configuration for every org's
    automation App: today's rows, plus Administration Read & write and
    Workflows Read & write.
  - Remove "Administration stays read-only" and the claim that only one
    org's App needs Workflows.
  - Add the `VRG_AUTOMATION_APP` org variable to the setup steps.
  - Add a section on token scoping: each job requests only what it needs.
- [ ] **Step 5: Validate, then commit** (`vrg-commit --type ci --scope ops`).
  The human's `vrg-submit-pr` pushes the branch.

### Task C1: Scope the automation tokens in vergil-containers `ops.yml`

**Files:** Modify `.github/workflows/ops.yml`.

- [ ] **Step 1:** Give each `create-github-app-token` step explicit
  `permission-*` inputs for what its job calls (the lifecycle jobs: Contents
  write, Pull requests write; the issue-filing token: Issues write).
- [ ] **Step 2: Validate, then commit** (`vrg-commit --type ci --scope ops`).

### Task D1 (deployment): Release the token scoping

Kind `deployment`, filed in vergil-actions and blocked by A1 and C1. Its
precondition is human-attested: vergil-actions is released with A1, and
vergil-containers has C1 merged.

- **Procedure:** dispatch one consumer's ops workflow and one epic-sweep run.
- **Acceptance:**
  - both runs are green;
  - their logs show tokens minted with the scoped permission set, and the
    `github-config` token shows exactly Administration read and Contents
    read.

### Task D2 (deployment): Configure every org's automation App

Kind `deployment`, filed in .github and blocked by D1. Every step is a
human action; the agent records the result.

- **Procedure:** for each org,
  1. search the org's code for `create-github-app-token` and
     `APP_PRIVATE_KEY`. Scope every minting step that has no `permission-*`
     inputs before going on, and record the search result on this task;
  2. compare the automation App's permissions with `configuration.md`'s
     table, and correct any difference;
  3. grant Administration Read & write, and Workflows Read & write;
  4. accept the change on the installation;
  5. set the org-level Actions variable `VRG_AUTOMATION_APP` to the App's
     slug.
- **Acceptance:**
  - each org's code search is recorded, with no unscoped minting step left;
  - each org's App permission page matches the table;
  - `vrg-gh api orgs/<org>/actions/variables/VRG_AUTOMATION_APP` returns
    the slug, with `visibility: all`, or `selected` with a list that covers
    every managed repo.

### Task A1b: The bypass probe

**Files:** Modify `.github/workflows/ops-github-config.yml`. It is
blocked by D2, so the probe's Administration-write token can be minted
from the moment it ships.

- [ ] **Step 1:** Add a `bypass-probe` job with these properties:
  - it runs only when `github.event_name == 'workflow_dispatch'`;
  - it fails unless `github.ref == 'refs/heads/develop'`, and checks out
    `ref: develop`;
  - it mints a token with Administration write and Metadata read;
  - it reads the repo's "Tag protection" ruleset;
  - it prints one line per bypass actor (`<actor_type> <actor_id>`), then
    `bypass_actors entries: <n>`;
  - it prints `VRG_AUTOMATION_APP set: yes` or `no`, read from
    `vars.VRG_AUTOMATION_APP`, with no value printed;
  - it prints `login: <login>` from GraphQL `viewer { login }` with the
    token, the premise of Task 1's identity check.

  Nightly scheduled runs skip it. Task A2 removes it.
- [ ] **Step 2: Validate, then commit** (`vrg-commit --type ci --scope ops`).

### Task V1 (validation): An automation token reads `bypass_actors`

Kind `validation`, filed in .github and blocked by A1b. Its precondition
is human-attested: vergil-actions is released with A1b.

- **Procedure:** dispatch the ops workflow in one public repo and in one
  private repo (in each org that has a private managed repo), and read the
  `bypass-probe` job's log (Task A1b).
- **Acceptance:** each log shows `RepositoryRole 5`,
  `bypass_actors entries: 1`, `VRG_AUTOMATION_APP set: yes`, and
  `login: <slug>[bot]` for the org's automation App.
- **On failure:** the result is recorded and the epic stops for a human
  decision (spec §6.3). Task 1, Task 5 and everything after them wait. Tasks
  2, 3a, 3b, 4 and 8 do not depend on the premise and may proceed.

### Task 1: The `automation` identity mode

**Files:**

- Modify: `src/vergil_tooling/lib/identity_mode.py`
- Create: `src/vergil_tooling/lib/automation_identity.py`
- Test: `tests/vergil_tooling/test_identity_mode.py`,
  `tests/vergil_tooling/test_automation_identity.py`

**Interfaces:**

- Produces:
  - `IdentityMode.AUTOMATION` (`"automation"`) and `is_automation() -> bool`.
  - `automation_identity.AUTOMATION_ENV = "VRG_AUTOMATION_APP"` and
    `ORG_VARIABLE = "VRG_AUTOMATION_APP"`.
  - `automation_app_slug(org: str) -> str`: the environment variable if set, otherwise
    the org variable read through the API. Raises `AutomationIdentityError`
    when it is unset or empty.
  - `automation_app_id(slug: str) -> int`, from `GET apps/<slug>`.
  - `verify_token_is_app(slug: str) -> None`. GraphQL `viewer { login }`
    must equal `f"{slug}[bot]"`. Transient errors are retried through
    `_run_with_retry`; anything else raises `AutomationIdentityError`,
    naming the expected and actual login.
  - `class AutomationIdentityError(RuntimeError)`.
- Consumes: `github.graphql`, `github.read_json`.

- [ ] **Step 1: Write the failing tests.**
  - `test_env_and_matching_token_resolves_automation`
  - `test_no_env_resolves_as_today`: each of today's signal cases,
    unchanged.
  - `test_env_with_other_app_token_raises`: the login is
    `vergil-project-user[bot]`, and the message names both logins.
  - `test_env_with_github_token_raises`: the login is
    `github-actions[bot]`.
  - `test_env_with_agent_env_mode_raises`: `VRG_IDENTITY_MODE=user` plus
    `VRG_AUTOMATION_APP`; the message says the signals contradict.
  - `test_lookup_failure_raises_after_retry`
  - `test_no_env_makes_no_api_call`
  - `test_automation_app_slug_reads_org_variable_without_env`
  - `test_automation_app_slug_missing_raises`
  - `test_automation_app_id_parses_id`
- [ ] **Step 2: Run the tests and see them fail.**
- [ ] **Step 3: Implement.** In `resolve()`, check `VRG_AUTOMATION_APP` first. With
  it set, call `verify_token_is_app`, then refuse if any agent
  signal (`any_agent_signal()`) is also present, and otherwise return
  `AUTOMATION`. Without it, the existing order is unchanged. `is_agent()`
  stays false for `AUTOMATION`, and `is_human()` is false too. Every
  identity check on `update`'s path names `AUTOMATION` explicitly (Task 6).
  **Trust:** the App name in `VRG_AUTOMATION_APP` is trusted because ops
  copies it from the org-level Actions variable, which only an org admin
  can set. Outside ops, a process that sets the environment variable by
  hand still holds only its own token, so GitHub bounds what it can do. On
  an agent VM, the agent identity signals make resolution refuse, and
  `update` refuses agent modes. Write this into the module docstring.
- [ ] **Step 4: Run the tests and see them pass.**
- [ ] **Step 5: Commit** (`vrg-commit --type feat --scope identity`).

### Task 2: `audit` and `update` run in a fresh clone of `develop`

**Files:**

- Create: `src/vergil_tooling/lib/develop_clone.py`
- Modify: `src/vergil_tooling/bin/vrg_github_repo_config.py` (the
  local-file half of `audit`, `diff` and `apply`)
- Test: `tests/vergil_tooling/test_develop_clone.py`,
  `tests/vergil_tooling/test_vrg_github_repo_config.py`

**Interfaces:**

- Produces: `clone_develop(repo: str) -> ContextManager[Path]`. It makes a
  shallow clone (`--depth 1 --branch develop`) into a
  `tempfile.TemporaryDirectory`, yields its path, and removes it on exit.
  It runs plain `git clone`, which authenticates through git's credential
  helper: `gh auth setup-git` in ops (Task A1), and the human's own `gh`
  login locally. No token appears in argv or logs.
  A failed clone raises `DevelopCloneError`, naming the repo.
- Consumes: `audit_local_config(repo_root: Path)` and every `_check_*`,
  unchanged. They already read a directory.

- [ ] **Step 1: Write the failing tests.**
  - `test_audit_reads_develop_not_cwd`: the CLI audits the clone, and a
    dirty cwd checkout does not change the result.
  - `test_audit_runs_without_checkout`: `--repo other/repo` from an
    unrelated cwd still audits files. Today's "skipped with a warning" path
    is removed.
  - `test_clone_removed_after_run`
  - `test_clone_failure_exits_2_naming_repo`
- [ ] **Step 2: Run the tests and see them fail.**
- [ ] **Step 3: Implement.** The CLI runs
  `audit_local_config(clone)` inside `with clone_develop(repo) as clone:`,
  in place of the cwd check.
- [ ] **Step 4: Run the tests and see them pass.**
- [ ] **Step 5: Commit** (`vrg-commit --type feat --scope repo-config`).

### Task 3a: Fixers for the shim, CLAUDE.md and Claude settings

**Files:**

- Create: `src/vergil_tooling/lib/repo_config_fixes.py`
- Modify: `src/vergil_tooling/lib/repo_config.py` (`_check_hook_guard_shim`,
  `_check_claude_md`, `_check_claude_settings`)
- Test: `tests/vergil_tooling/test_repo_config_fixes.py`

**Interfaces:**

- Produces:
  - One fixer per managed file, each `fix_<name>(root: Path) -> bool`. It
    writes the file in place in the clone, and returns whether it changed
    anything:
    - `fix_hook_guard_shim`
    - `fix_claude_md`
    - `fix_claude_settings`
  - `class CannotFix(Exception)`, raised by a fixer when its check fails
    but no fix can be computed (for example, CLAUDE.md markers that are
    missing or malformed). It carries the check's name and reason.
  - The registry `FIXERS: list[Callable[[Path], bool]]`.

  Each `_check_*` and its fixer share one definition of the correct file.
  The check reports drift exactly when the fixer would change something
  or raise `CannotFix`.
- Consumes: the existing renderers (`render_hook_guard_shim`,
  `load_claude_md_template`, `claude_md_marker_lines`,
  `_load_settings_template`) and the retired applicators' logic in
  `bin/vrg_hook_guard_shim_sync.py`, `bin/vrg_claude_md_sync.py` and
  `bin/vrg_claude_settings_sync.py`.

- [ ] **Step 1: Write the failing tests.**
  - `test_shim_not_executable_is_drift_and_fix_sets_0755`
  - `test_shim_missing_is_created_0755`
  - `test_claude_md_region_replaced_outside_text_kept`
  - `test_claude_md_without_markers_raises_cannot_fix`: the check still
    reports it, and `fix_claude_md` raises `CannotFix`.
  - `test_claude_settings_written_equal_to_template`
  - `test_in_sync_root_changes_nothing`: every fixer returns `False`.
  - `test_check_and_fixer_agree`: parametrised over every entry in `FIXERS`,
    so Task 3b's fixers are covered too. For each fixture directory, the check
    reports drift if and only if the fixer changes something or raises. Task
    3b adds fixtures for its checks.
- [ ] **Step 2: Run the tests and see them fail.**
- [ ] **Step 3: Implement**, porting each retired applicator's write logic.
  The shim check also requires the executable bit.
- [ ] **Step 4: Run the tests and see them pass.**
- [ ] **Step 5: Commit** (`vrg-commit --type feat --scope repo-config`).

### Task 3b: Fixers for `.gitignore`, `dependabot.yml` and workflow ref pins

**Files:**

- Modify: `src/vergil_tooling/lib/repo_config_fixes.py`,
  `src/vergil_tooling/lib/repo_config.py` (`_check_gitignore`,
  `_check_dependabot_yml`, `_check_workflow_refs`)
- Test: `tests/vergil_tooling/test_repo_config_fixes.py`

**Interfaces:**

- Produces, each added to `FIXERS` with the Task 3a contract:
  - `fix_gitignore`;
  - `fix_dependabot_yml`;
  - `fix_workflow_refs`, which rewrites every workflow file whose
    `vergil-*` pins differ.
- Consumes: `lib/dependabot.render`, `lib/vergil_refs`, the gitignore data
  files, and `bin/vrg_gitignore_sync.py` / `bin/vrg_dependabot_sync.py`
  logic.

- [ ] **Step 1: Write the failing tests.**
  - `test_gitignore_block_replaced_user_lines_kept`
  - `test_dependabot_rendered_exactly`
  - `test_workflow_pins_rewritten_to_vergil_toml_version`
- [ ] **Step 2: Run the tests and see them fail.**
- [ ] **Step 3: Implement.**
- [ ] **Step 4: Run the tests and see them pass.**
- [ ] **Step 5: Commit** (`vrg-commit --type feat --scope repo-config`).

### Task 4: The mechanized-PR library

**Files:**

- Create: `src/vergil_tooling/lib/mechanized_pr.py`
- Modify: `src/vergil_tooling/lib/github.py` (`create_pr` gains `repo`)
- Test: `tests/vergil_tooling/test_mechanized_pr.py`

**Interfaces:**

- Produces:
  - `AUTOMATION_BRANCH_PREFIX = "automation/"`.
  - `@dataclass(frozen=True) class SyncResult: status: Literal["in_sync",
    "opened"]; pr_url: str | None; closed: list[str]`.
  - `open_sync_pr(repo: str, clone: Path, *, job: str, title: str, body:
    str) -> SyncResult`. `clone` is a `clone_develop` checkout (Task 2)
    with the job's changes already written in it.
  - `branch_name(job: str, run_id: str) -> str`, giving
    `automation/<job>/<run-id>`. `run_id` is `$GITHUB_RUN_ID` when set,
    otherwise a UTC `%Y%m%dT%H%M%SZ` stamp plus 6 hex characters.
  - `class MechanizedPRError(RuntimeError)`.
- Consumes: `lib.git` (status, checkout, commit, push, run with `-C <clone>`;
  authentication is git's credential helper, Task 2), and `github.delete_if_exists`
  (a branch already gone counts as deleted), and
  `github.create_pr`, `graphql`, `read_json` and `delete`. `create_pr` gains a
  `repo: str` argument that is passed as `--repo`, and every PR list and
  close call here passes `--repo` too, so nothing depends on the cwd.

- [ ] **Step 1: Write the failing tests**, against a local bare repo
  standing in for `origin` and a mocked GitHub API.
  - `test_clean_clone_closes_earlier_pr_and_reports_in_sync`
  - `test_changes_pushed_on_new_branch_and_pr_opened`: the branch's parent
    is the clone's `develop` commit, and the shim's executable bit
    survives the commit.
  - `test_earlier_open_pr_closed_before_new_one_opened`
  - `test_branch_without_pr_under_own_prefix_deleted`: a branch a failed
    earlier attempt pushed without a PR is deleted, and a re-run with the
    same run ID then pushes cleanly.
  - `test_auto_merge_enabled_squash_with_expected_head`: GraphQL
    `enablePullRequestAutoMerge` gets `mergeMethod: SQUASH` and
    `expectedHeadOid` equal to the pushed commit.
  - `test_auto_merge_not_confirmed_raises`: `autoMergeRequest` is null on
    read-back.
  - `test_push_failure_raises_naming_step`
  - `test_pr_opened_for_named_repo_from_unrelated_cwd`: every `gh` PR call
    (create, list, close) passes `--repo <repo>`.
  - `test_only_own_job_prefix_closed`: a PR on `automation/other-job/…` is
    left alone.
  - `test_fork_pr_on_automation_prefix_ignored`: a cross-repository PR
    from `automation/repo-config-sync/x` is neither closed nor touched.
- [ ] **Step 2: Run the tests and see them fail.**
- [ ] **Step 3: Implement.**
  1. First, delete every branch in this repo under `automation/<job>/`,
     closing its PR first if one is open. Only same-repo PRs count
     (`head.repo.full_name == repo`): a PR from a fork is never closed.
  2. Then, with plain git in the clone: `git status`, then `checkout -b`,
     `commit`, and `push origin <branch>`.
  3. Then `create_pr`, the GraphQL auto-merge, and the read-back.
- [ ] **Step 4: Run the tests and see them pass.**
- [ ] **Step 5: Commit** (`vrg-commit --type feat --scope automation`).

### Task 5: Desired state: auto-merge, the automation-prefix ruleset, fatal bypass reads

**Files:**

- Modify: `src/vergil_tooling/lib/github_config.py`
  (`desired_repo_settings`, `compute_desired_state`, `_diff_rulesets`)
- Modify: `src/vergil_tooling/bin/vrg_github_repo_config.py`
  (`_audit_repo`, `_apply_repo`: passing the mode and the App ID)
- Modify: `src/vergil_tooling/lib/repo_init.py` (`step_github_config`, the
  third `compute_desired_state` caller). Repo-init always runs as the human,
  so it passes `skip_bypass=False` and resolves the App ID the way a human
  run does.
- Test: `tests/vergil_tooling/test_github_config_lib.py`,
  `tests/vergil_tooling/test_repo_init.py`
  (`test_new_repo_gets_automation_ruleset_with_both_bypass_actors`)

**Interfaces:**

- Produces:
  - `desired_automation_branch_ruleset(app_id: int | None) -> DesiredRuleset`:
    - name `"Automation branches"`, target `branch`;
    - `ref_include=["refs/heads/automation/**"]`;
    - rules `creation`, `update` and `non_fast_forward`;
    - bypass actors `{app_id, "Integration", "always"}` and
      `{5, "RepositoryRole", "always"}`. `None` means skipped, which is
      allowed only when `skip_bypass` is set.
  - `compute_desired_state(config, *, visibility, is_org, skip_bypass:
    bool, automation_app_id: int | None)`, replacing `app_mode`.
    `skip_bypass` is true only for a non-automation App token: today's
    ops `audit`.
  - `BypassUnreadable`, a `NotChecked` entry recorded when a desired
    non-empty bypass list reads back empty **under the automation identity**
    (`identity_mode.is_automation()`). For a human run, an empty list is
    ordinary drift, and the run converges it.
  - `allow_auto_merge=True` in `desired_repo_settings`.
- Consumes: `identity_mode.is_automation`, `github.is_app_mode`, and
  `automation_identity.automation_app_slug` / `automation_app_id`
  (Task 1).

- [ ] **Step 1: Write the failing tests.**
  - `test_desired_settings_allow_auto_merge`
  - `test_automation_ruleset_shape`: the exact dict above, and no
    `deletion` rule.
  - `test_empty_bypass_read_is_unreadable_under_automation`
  - `test_empty_bypass_read_skipped_for_non_automation_app`
  - `test_human_run_converges_emptied_bypass_list`: a human run on a
    ruleset whose bypass list reads back `[]` reports drift and writes the
    desired list.
  - `test_unreadable_bypass_stops_apply_before_any_write`
- [ ] **Step 2: Run the tests and see them fail.**
- [ ] **Step 3: Implement.** The CLI derives `skip_bypass =
  github.is_app_mode() and not identity_mode.is_automation()`. It derives
  the App ID via `automation_app_id(automation_app_slug(owner))` whenever
  `skip_bypass` is false. Update the `_resolve_config` docstring, whose
  trust note ("only a human runs `apply`") no longer covers `update`.
- [ ] **Step 4: Run the tests and see them pass.**
- [ ] **Step 5: Commit** (`vrg-commit --type feat --scope repo-config`).

### Task 6: `vrg-github-repo-config update`; `apply` deprecated

**Files:**

- Modify: `src/vergil_tooling/bin/vrg_github_repo_config.py`
- Modify: `docs/site/docs/reference/config-audit.md` (the `update` and
  `apply` sections)
- Test: `tests/vergil_tooling/test_vrg_github_repo_config.py`

**Interfaces:**

- Produces: `vrg-github-repo-config update --repo R [--config PATH]`. Exit
  codes:
  - **0:** in sync, or a sync PR was opened with auto-merge confirmed;
  - **1:** drift that could not be fixed, with any PR still opened;
  - **2:** something could not be audited or converged.
- Consumes: `clone_develop`, `audit_local_config`, `FIXERS`, `open_sync_pr`, `compute_desired_state` and
  the identity functions (Tasks 1–5).

- [ ] **Step 1: Write the failing tests.**
  - `test_update_refuses_agent_modes`: `user` and `audit` exit 2, naming
    the mode.
  - `test_update_runs_for_human_and_automation`
  - `test_update_opens_one_pr_for_all_fixable_drift`
  - `test_update_opens_pr_then_exits_1_for_no_fixer_check` (review focus
    1)
  - `test_update_unreadable_github_setting_opens_nothing`
  - `test_update_in_sync_exits_0_no_pr`
  - `test_apply_unchanged_and_prints_deprecation`: stderr names `update`;
    the behaviour is identical to `apply` after Tasks 2 and 5, apart from the
    notice.
- [ ] **Step 2: Run the tests and see them fail.**
- [ ] **Step 3: Implement** `_update_repo`:
  1. identity gate;
  2. GitHub-side convergence (shared with `apply`);
  3. `with clone_develop(repo) as clone:`;
  4. every fixer in `FIXERS` on the clone. Each `CannotFix` is collected,
     and does not stop the other fixers;
  5. `open_sync_pr(repo, clone, job="repo-config-sync", …)`;
  6. the exit code: `audit_local_config(clone)` on the fixed clone. Any
     failure left, or any `CannotFix`, is exit 1, even with a PR opened.

  Update `config-audit.md`.
- [ ] **Step 4: Run the tests and see them pass.**
- [ ] **Step 5: Commit** (`vrg-commit --type feat --scope repo-config`).

### Task 8: `vrg-repo-status` shows automation PRs as `automation`

`vrg-repo-status` (epic vergil-project/.github#1049) has shipped. This task
extends it.

**Deliverable:** PRs whose branch is under `automation/` show with the type
`automation`, kept apart from development PRs wherever the tool groups or
counts PRs. The docs list the new type. How the tool classifies and counts
it is left to the builder, following the tool's existing branch-based types
(`deps/update-*`, `release/post-*`).

- [ ] **Step 1:** Write a failing test: a PR on
  `automation/repo-config-sync/123` shows as `automation`, apart from
  development PRs.
- [ ] **Step 2:** Implement it, update the docs, and see the test pass.
- [ ] **Step 3: Commit** (`vrg-commit --type feat --scope repo-status`).

### Task D3 (deployment): Release the tooling; the same-day sweep

Kind `deployment`, filed in vergil-tooling and blocked by 6 and 8. Its
precondition is human-attested: vergil-tooling is released with Tasks 1–6 and 8.

- **Procedure:** the human runs `vrg-github-repo-config update --repo
  <repo>` for every managed repo in each org, the same day.
- **Acceptance:**
  - `vrg-github-repo-config audit` shows no GitHub-side drift on any repo;
  - each repo's file state is either in sync, or covered by an open sync PR
    with auto-merge on;
  - each repo shows `allow_auto_merge` on, and the "Automation branches"
    ruleset with both bypass actors;
  - any sync PRs the sweep opened are recorded.

### Task A2: Ops switch: `github-config` runs `update` as automation

**Files:** Modify `.github/workflows/ops-github-config.yml` (the
`github-config` job), and `docs/site/docs/workflows/ops-github-config.md`
(the job runs `update` as the automation identity with a write-scoped
token, only from `develop`; the probe job is gone).

- [ ] **Step 1:** Scope the token to Administration write, Contents write,
  Pull requests write, Workflows write and Metadata read.
- [ ] **Step 2:** Set `env: VRG_AUTOMATION_APP: ${{ vars.VRG_AUTOMATION_APP }}`,
  and add a first step that fails the job when it is empty.
- [ ] **Step 2b: Run only from `develop`.**
  - The first step fails the job unless `github.ref == 'refs/heads/develop'`.
  - `actions/checkout` pins `ref: develop`.

  Every fleet repo's default branch is `develop`, so the automation job
  never runs code from any other branch.
- [ ] **Step 2c: Set the git identity**, following the pattern in
  `actions/create-github-app-token`'s README. Set `user.name` to
  `<app-slug>[bot]`, from the token step's `app-slug` output. Set
  `user.email` to `<id>+<app-slug>[bot]@users.noreply.github.com`, with
  `<id>` from `gh api /users/<app-slug>[bot] --jq .id`. Sync commits then
  show as the automation App.
- [ ] **Step 3:** Replace `audit` with `vrg-github-repo-config update --repo
  "${{ github.repository }}"`, and rename the step. Remove the
  `bypass-probe` job (Task A1b).
- [ ] **Step 4: Validate, then commit** (`vrg-commit --type ci --scope ops`).
  The human's `vrg-submit-pr` pushes the branch.

### Task D4 (deployment): Release the ops switch

Kind `deployment`, filed in vergil-actions and blocked by A2. Its
precondition is human-attested: vergil-actions is released with A2.

- **Acceptance:** the rolling line tag points at the release.

### Task V2 (validation): Live validation on switch day

Kind `validation`, filed in vergil-actions and blocked by D4.

- **Procedure:** dispatch one repo's ops workflow, then every repo's, or
  let the night run.
- **Acceptance (spec §6.7, step 6):**
  - the first run is green;
  - its `github-config` log shows the `automation` mode resolved;
  - any sync PR opens on `automation/repo-config-sync/<run-id>` with
    auto-merge on, and merges once CI is green;
  - the run reports no bypass drift and no `BypassUnreadable`, and its log
    shows both rulesets' bypass actors read back;
  - after the fleet-wide run, every failure is triaged and either fixed
    forward or answered by reverting the switch, with the outcome recorded.

### Task 9: Retire the sync tools

**Files:**

- Delete:
  - the `bin/` scripts: `vrg_fleet_sync.py`, `vrg_gitignore_sync.py`,
    `vrg_dependabot_sync.py`, `vrg_claude_settings_sync.py`,
    `vrg_claude_md_sync.py`, `vrg_hook_guard_shim_sync.py`,
    `vrg_ci_sync.py`;
  - the `lib/` modules `fleet_sweep.py` and `ci_yml_applicator.py`;
  - their tests.
- Modify: `pyproject.toml` (the entry points), and
  `docs/site/docs/reference/config-audit.md` and `cli-tools-overview.md`
  (remove the sync-tool sections).

- [ ] **Step 1:** Search `src/`, `tests/` and `docs/` with `rg` for the seven command
  names (`vrg-fleet-sync`, `vrg-gitignore-sync`, `vrg-dependabot-sync`,
  `vrg-claude-settings-sync`, `vrg-claude-md-sync`, `vrg-hook-guard-shim-sync`,
  `vrg-ci-sync`) and for the deleted modules. Fix every hit outside the deleted
  files, remediation messages and docstrings included (for example
  `bin/vrg_container_docs.py`, `lib/gitignore.py`, `lib/dependabot.py`, and
  the guides and reference pages). Change the managed `.gitignore` block's
  header text from naming `vrg-gitignore-sync` to naming
  `vrg-github-repo-config update`, and update the gitignore data files and
  tests to match. Every repo then drifts once, and the nightly `update`
  syncs the new header fleet-wide.
- [ ] **Step 2:** Delete them and remove the entry points and docs.
- [ ] **Step 3: Validate, then commit** (`vrg-commit --type refactor
  --scope sync`).

## Evolution during execution

### 2026-11-13: the `epic-sweep` token, D2's permission list, and a catch-all exception

**What changed:**

- **Task A1 (vergil-project/vergil-actions#1153) no longer scopes the
  `epic-sweep` token.** By the human's ruling, that part is split off
  through a scope change on #1153. Every other token is scoped as planned.
- **Two tasks are added:**
  - **vergil-project/vergil-tooling#4313:** move the advisory-state sync
    out of `vrg-epic-audit --close`. It has no blockers.
  - **vergil-project/vergil-actions#1159:** scope the `epic-sweep` token
    to Issues write, Pull requests read and Contents read. It is blocked
    by #4313 and a vergil-tooling release that carries it.
- **Task D2 (vergil-project/.github#1083) is also blocked by #1159.**
- **Task D2's asserted App permissions also include Repository security
  advisories: Read & write.** This came through a scope change on #1083.
  A1's rewritten `configuration.md` table lists it too.
- **One task is added:** vergil-project/vergil-tooling#4311 narrows the
  `except Exception: return False` in `_cwd_matches_repo`
  (`bin/vrg_github_repo_config.py`). It has no blockers.

**Why:**

- **The `epic-sweep` token.** Task A1's inventory found that the
  security-records pass of `vrg-epic-audit --close` updates advisory
  states in the org `.github` repo. That needs Repository security
  advisories: write, and `actions/create-github-app-token` has no input
  for it. A scoped sweep token would lose it, so the token stays unscoped
  for now. Once D2 grants Administration write, an unscoped org-wide token
  would carry it, which is what spec §6.4's scoping rule forbids. So the
  token must be scoped, through #4313 and #1159, before D2.
- **D2's permission list.** The plan's list left out a permission the
  nightly sweep already uses in every org.
- **The catch-all exception.** The run for Task 2 (vergil-tooling#4301)
  found it. It predates the epic and conflicts with the no-silent-failures
  rule. The human approved filing it on the epic.

The updated order for the rollout chain:

| Task | Repo | What | Blocked by |
| --- | --- | --- | --- |
| 4313 | vergil-tooling | Move the advisory sync out of `--close` | — |
| 1159 | vergil-actions | Scope the `epic-sweep` token | 4313 (released) |
| D2 | .github (deployment) | As planned, plus the advisories permission | D1, 1159 |
| 4311 | vergil-tooling | Narrow the catch-all in `_cwd_matches_repo` | — |
