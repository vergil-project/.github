# Automatic fleet sync through repo-config update Implementation Plan

- **Epic:** vergil-project/.github#1064
- **Planning issue:** vergil-project/.github#909 (review history is in its comments)
- **Status:** draft, in design review

> **For agentic workers:** each task is one PR, implemented with the
> `issue-implement` skill (operational tasks with `issue-deploy` or
> `issue-validate`). Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** each repo's nightly ops job runs `vrg-github-repo-config update`
as the org automation App. When the repo's managed files have drifted from
the released master copies, it opens one auto-merging PR on a fresh,
push-protected `automation/` branch. The ad-hoc `vrg-*-sync` tools are
retired.

**Architecture:** vergil-tooling gains four things:
- an `automation` identity mode, switched on by an env marker that names
  the App and verified against the token;
- a file-tree view of a repo's `develop` tip that the managed-file checks
  and their new fixers share;
- a mechanized-PR library that writes commits through the Git Data API and
  turns on auto-merge;
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
- In this epic only the `github-config` ops job sets the automation marker.
  Every other job resolves exactly as it does today.
- The marker env var and the org-level Actions variable are both named
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
- No silent failures: every failure is a non-zero exit with the reason
  named.
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
3. **The marker set, but the token belongs to a different App or to
   `GITHUB_TOKEN`.** Expected: resolution fails, naming both identities. Test
   in Task 1.
4. **An earlier run's sync PR still open, and the drift now gone.**
   Expected: that PR is closed, its branch is deleted, and the result is "in
   sync". Test in Task 4.
5. **The hook-guard shim with correct bytes but mode 100644.** Expected: it
   is reported as drift and fixed to 100755. Test in Task 3a.

---

## Delivery

| Task | Repo | What | Blocked by |
| --- | --- | --- | --- |
| A1 | vergil-actions | Scope every automation token, add per-job concurrency, rewrite `configuration.md` | — |
| C1 | vergil-containers | Scope the automation tokens in `ops.yml` | — |
| D1 | vergil-actions (deployment) | Release the token scoping and confirm consumers pick it up | A1, C1 |
| D2 | .github (deployment) | Assert every org's App configuration, grant Administration and Workflows write, set `VRG_AUTOMATION_APP` | D1 |
| V1 | .github (validation) | An automation token reads `bypass_actors` correctly | D2 |
| 1 | vergil-tooling | The `automation` identity mode | — |
| 2a | vergil-tooling | `FileTree`: a repo's files and modes at a ref | — |
| 2b | vergil-tooling | Managed-file checks read a `FileTree` of `develop` | 2a |
| 3a | vergil-tooling | Expected-file definitions and fixers: shim, CLAUDE.md, Claude settings | 2b |
| 3b | vergil-tooling | Expected-file definitions and fixers: `.gitignore`, `dependabot.yml`, workflow ref pins | 2b |
| 4 | vergil-tooling | The mechanized-PR library | 2a |
| 5 | vergil-tooling | Desired state: auto-merge, the automation-prefix ruleset, fatal bypass reads | 1, V1 |
| 6 | vergil-tooling | `vrg-github-repo-config update`; `apply` deprecated | 1, 3a, 3b, 4, 5 |
| 7 | vergil-tooling | Branch-sweep cleans finished automation branches | 5 |
| 8 | vergil-tooling | `vrg-repo-status` shows automation PRs as `automation` | — (epic #1049 shipped) |
| D3 | vergil-tooling (deployment) | Release the tooling; the human's same-day `update` sweep | 6, 7, 8 |
| A2 | vergil-actions | Ops switch: `github-config` runs `update` as automation | D3 |
| D4 | vergil-actions (deployment) | Release the ops switch | A2 |
| V2 | vergil-actions (validation) | Live validation on switch day | D4 |
| 9 | vergil-tooling | Retire the sync tools | V2 |

Advisory hand-off: GHSA-mv3r-qfr4-7fr6 is fixed by Task 5 (the
automation-prefix ruleset).

### Task A1: Scope every automation token, add per-job concurrency, rewrite the App docs

**Files:**

- Modify: `.github/workflows/ops-github-config.yml`, `ops-epic-sweep.yml`,
  `ops-epic-rollup.yml`, `ops-upstream-refresh.yml`, `cd.yml`,
  `cd-release.yml`
- Modify: `docs/site/docs/configuration.md`

**Interfaces:**

- Produces: every `actions/create-github-app-token` step in vergil-actions
  carries explicit `permission-*` inputs, and every job in
  `ops-github-config.yml` has a `concurrency:` group named
  `${{ github.repository }}/<job>` with `cancel-in-progress: false`.

- [ ] **Step 1: Inventory.** List every `create-github-app-token` step and
  the API calls its job makes. Record each step's needed permissions in the
  PR body. `github-config` keeps today's unscoped token until Task A2:
  scope it to Administration read, Contents read and Issues read, which is
  what `audit` reads.
- [ ] **Step 2: Scope each step.** Add `permission-*` inputs matching the
  inventory. Leave `branch-sweep` and `dependabot-audit`, which are already
  scoped, unchanged.
- [ ] **Step 3: Add concurrency.** Add the per-job `concurrency:` groups to
  `branch-sweep`, `github-config` and `dependabot-audit`.
- [ ] **Step 3b: Add the bypass probe for Task V1.** Add a `bypass-probe`
  job to `ops-github-config.yml` with these properties:
  - it runs only when `github.event_name == 'workflow_dispatch'`;
  - it mints a token with Administration write;
  - it reads the repo's "Tag protection" ruleset;
  - it prints only `bypass_actors entries: <n>`.

  Before Task D2 it fails at token minting, which is expected. Nightly
  scheduled runs skip it. Task A2 removes it.
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
  - their logs show tokens minted with the scoped permission set.

### Task D2 (deployment): Configure every org's automation App

Kind `deployment`, filed in .github and blocked by D1. Every step is a
human action; the agent records the result.

- **Procedure:** for each org,
  1. compare the automation App's permissions with `configuration.md`'s
     table, and correct any difference;
  2. grant Administration Read & write, and Workflows Read & write;
  3. accept the change on the installation;
  4. set the org-level Actions variable `VRG_AUTOMATION_APP` to the App's
     slug.
- **Acceptance:**
  - each org's App permission page matches the table;
  - `vrg-gh api orgs/<org>/actions/variables/VRG_AUTOMATION_APP` returns
    the slug.

### Task V1 (validation): An automation token reads `bypass_actors`

Kind `validation`, filed in .github and blocked by D2.

- **Procedure:** in one repo, dispatch the ops workflow, and read the
  `bypass-probe` job's log (Task A1, step 3b).
- **Acceptance:** the log shows `bypass_actors entries: 1`; the Tag
  protection ruleset lists only `RepositoryRole` 5.
- **On failure:** the result is recorded and the epic stops for a human
  decision (spec §6.3). Tasks 5 onward stay blocked.

### Task 1: The `automation` identity mode

**Files:**

- Modify: `src/vergil_tooling/lib/identity_mode.py`
- Create: `src/vergil_tooling/lib/automation_identity.py`
- Test: `tests/vergil_tooling/test_identity_mode.py`,
  `tests/vergil_tooling/test_automation_identity.py`

**Interfaces:**

- Produces:
  - `IdentityMode.AUTOMATION` (`"automation"`) and `is_automation() -> bool`.
  - `automation_identity.MARKER_ENV = "VRG_AUTOMATION_APP"` and
    `ORG_VARIABLE = "VRG_AUTOMATION_APP"`.
  - `automation_app_slug(org: str) -> str`: the marker if set, otherwise
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
  - `test_marker_and_matching_token_resolves_automation`
  - `test_no_marker_resolves_as_today`: each of today's signal cases,
    unchanged.
  - `test_marker_with_other_app_token_raises`: the login is
    `vergil-project-user[bot]`, and the message names both logins.
  - `test_marker_with_github_token_raises`: the login is
    `github-actions[bot]`.
  - `test_marker_with_agent_env_mode_raises`: `VRG_IDENTITY_MODE=user` plus
    the marker; the message says the signals contradict.
  - `test_lookup_failure_raises_after_retry`
  - `test_no_marker_makes_no_api_call`
  - `test_automation_app_slug_reads_org_variable_without_marker`
  - `test_automation_app_slug_missing_raises`
  - `test_automation_app_id_parses_id`
- [ ] **Step 2: Run the tests and see them fail.**
- [ ] **Step 3: Implement.** In `resolve()`, check the marker first. With
  the marker set, call `verify_token_is_app`, then refuse if any agent
  signal (`any_agent_signal()`) is also present, and otherwise return
  `AUTOMATION`. Without it, the existing order is unchanged. `is_agent()`
  stays false for `AUTOMATION`, and `is_human()` is false too. Every
  identity check on `update`'s path names `AUTOMATION` explicitly (Task 6).
- [ ] **Step 4: Run the tests and see them pass.**
- [ ] **Step 5: Commit** (`vrg-commit --type feat --scope identity`).

### Task 2a: `FileTree`, a repo's files and modes at a ref

**Files:**

- Create: `src/vergil_tooling/lib/file_tree.py`
- Test: `tests/vergil_tooling/test_file_tree.py`

**Interfaces:**

- Produces:
  - `@dataclass(frozen=True) class TreeFile: path: str; content: bytes;
    mode: str`, where `mode` is one of `"100644"`, `"100755"`.
  - `class FileTree`, with:
    - `get(path: str) -> TreeFile | None`;
    - `paths() -> frozenset[str]`;
    - `ref_sha: str`, the commit SHA it was read at.
  - `fetch_tree(repo: str, ref: str) -> FileTree`. It reads the recursive
    Git tree once, and fetches blob contents lazily on `get`.
  - `FileTree.from_files(files: Iterable[TreeFile], ref_sha: str) ->
    FileTree`, for tests and for callers.
- Consumes: `github.read_json`.

- [ ] **Step 1: Write the failing tests.**
  - `test_fetch_tree_reads_modes`: a tree response with a `100755` entry
    gives `TreeFile.mode == "100755"`.
  - `test_get_missing_returns_none`
  - `test_fetch_tree_truncated_raises`: `"truncated": true` raises
    `FileTreeError`.
  - `test_blob_fetched_once`
  - `test_symlink_and_submodule_entries_rejected_on_get`: `get` on a
    `120000` or `160000` entry raises `FileTreeError`.
- [ ] **Step 2: Run the tests and see them fail.**
- [ ] **Step 3: Implement**, with `GET repos/<repo>/git/trees/<ref>?recursive=1`
  and `GET repos/<repo>/git/blobs/<sha>` (base64).
- [ ] **Step 4: Run the tests and see them pass.**
- [ ] **Step 5: Commit** (`vrg-commit --type feat --scope repo-config`).

### Task 2b: Managed-file checks read a `FileTree` of `develop`

**Files:**

- Modify: `src/vergil_tooling/lib/repo_config.py` (every `_check_*`, and
  `audit_local_config`)
- Modify: `src/vergil_tooling/bin/vrg_github_repo_config.py` (`audit`,
  `diff`, `apply`: the local-file half)
- Test: `tests/vergil_tooling/test_repo_config.py`,
  `tests/vergil_tooling/test_vrg_github_repo_config.py`

**Interfaces:**

- Produces: `audit_files(tree: FileTree) -> ConfigDiff`, which replaces
  `audit_local_config(repo_root: Path)`. Every `_check_*(tree, items)`
  takes the tree.
- Consumes: `fetch_tree` (Task 2a).

- [ ] **Step 1: Convert the existing tests** to build trees with
  `FileTree.from_files`, keeping every assertion. Add:
  - `test_audit_reads_develop_not_cwd`: the CLI fetches `develop` for
    `--repo`, and a dirty working tree does not change the result;
  - `test_audit_runs_without_checkout`: `--repo other/repo` from an
    unrelated cwd still audits files. Today's "skipped with a warning" path
    is removed.
- [ ] **Step 2: Run the tests and see them fail.**
- [ ] **Step 3: Implement.** Port each check onto the tree.
  `_gitignore_language` and `_check_workflow_refs` read `vergil.toml` from
  the tree. The CLI calls `audit_files(fetch_tree(repo, "develop"))`.
- [ ] **Step 4: Run the tests and see them pass.**
- [ ] **Step 5: Commit** (`vrg-commit --type refactor --scope repo-config`).

### Task 3a: Expected-file definitions and fixers for the shim, CLAUDE.md and Claude settings

**Files:**

- Create: `src/vergil_tooling/lib/repo_config_fixes.py`
- Modify: `src/vergil_tooling/lib/repo_config.py` (`_check_hook_guard_shim`,
  `_check_claude_md`, `_check_claude_settings`)
- Test: `tests/vergil_tooling/test_repo_config_fixes.py`

**Interfaces:**

- Produces: one function per managed file, each
  `expected_<name>(tree: FileTree) -> TreeFile | None`, where `None` means
  the check takes no position:
  - `expected_hook_guard_shim`
  - `expected_claude_md`
  - `expected_claude_settings`

  Each `_check_*` compares the file in the tree with its `expected_*`, so a
  check and its fix share one definition. Also produces the registry
  `FIXERS: list[Callable[[FileTree], TreeFile | None]]`.
- Consumes: the existing renderers (`render_hook_guard_shim`,
  `load_claude_md_template`, `claude_md_marker_lines`,
  `_load_settings_template`) and the retired applicators' logic in
  `bin/vrg_hook_guard_shim_sync.py`, `bin/vrg_claude_md_sync.py` and
  `bin/vrg_claude_settings_sync.py`.

- [ ] **Step 1: Write the failing tests.**
  - `test_shim_wrong_mode_is_drift_and_fix_sets_100755`
  - `test_shim_missing_is_created_100755`
  - `test_claude_md_region_replaced_outside_text_kept`
  - `test_claude_md_without_markers_has_no_fix`: the check still reports
    it; `expected_claude_md` returns `None`, so it is a no-fixer failure.
  - `test_claude_settings_written_equal_to_template`
  - `test_in_sync_tree_yields_no_fixes`
- [ ] **Step 2: Run the tests and see them fail.**
- [ ] **Step 3: Implement**, porting each retired applicator's write logic.
- [ ] **Step 4: Run the tests and see them pass.**
- [ ] **Step 5: Commit** (`vrg-commit --type feat --scope repo-config`).

### Task 3b: Expected-file definitions and fixers for `.gitignore`, `dependabot.yml` and workflow ref pins

**Files:**

- Modify: `src/vergil_tooling/lib/repo_config_fixes.py`,
  `src/vergil_tooling/lib/repo_config.py` (`_check_gitignore`,
  `_check_dependabot_yml`, `_check_workflow_refs`)
- Test: `tests/vergil_tooling/test_repo_config_fixes.py`

**Interfaces:**

- Produces, each added to `FIXERS`:
  - `expected_gitignore`;
  - `expected_dependabot_yml`;
  - `expected_workflow_refs(tree) -> list[TreeFile]`, one entry per
    workflow file whose `vergil-*` pins differ.
- Consumes: `lib/dependabot.render`, `lib/vergil_refs`, the gitignore data
  files, and `bin/vrg_gitignore_sync.py` / `bin/vrg_dependabot_sync.py`
  logic.

- [ ] **Step 1: Write the failing tests.**
  - `test_gitignore_block_replaced_user_lines_kept`
  - `test_dependabot_rendered_exactly`
  - `test_workflow_pins_rewritten_to_vergil_toml_version`
  - `test_workflow_file_mode_preserved`
- [ ] **Step 2: Run the tests and see them fail.**
- [ ] **Step 3: Implement.**
- [ ] **Step 4: Run the tests and see them pass.**
- [ ] **Step 5: Commit** (`vrg-commit --type feat --scope repo-config`).

### Task 4: The mechanized-PR library

**Files:**

- Create: `src/vergil_tooling/lib/mechanized_pr.py`
- Test: `tests/vergil_tooling/test_mechanized_pr.py`

**Interfaces:**

- Produces:
  - `AUTOMATION_BRANCH_PREFIX = "automation/"`.
  - `@dataclass(frozen=True) class MechanizedChange: job: str; base: str;
    files: list[TreeFile]; title: str; body: str`.
  - `@dataclass(frozen=True) class SyncResult: status: Literal["in_sync",
    "opened"]; pr_url: str | None; closed: list[str]`.
  - `sync_pr(repo: str, change: MechanizedChange, *, base_tree: FileTree)
    -> SyncResult`.
  - `branch_name(job: str, run_id: str) -> str`, giving
    `automation/<job>/<run-id>`. `run_id` is `$GITHUB_RUN_ID` when set,
    otherwise a UTC `%Y%m%dT%H%M%SZ` stamp plus 6 hex characters.
  - `class MechanizedPRError(RuntimeError)`.
- Consumes: `github.read_json`, `post_json_once`, `graphql`, `delete`, and
  `TreeFile` / `FileTree` (Task 2a).

- [ ] **Step 1: Write the failing tests.**
  - `test_no_change_closes_earlier_pr_and_deletes_branch`
  - `test_change_opens_pr_from_base_tip_on_new_branch`: the commit's
    parent is `base_tree.ref_sha`, and the tree entries carry the given
    modes.
  - `test_earlier_open_pr_closed_before_new_one_opened`
  - `test_auto_merge_enabled_squash_with_expected_head`: GraphQL
    `enablePullRequestAutoMerge` gets `mergeMethod: SQUASH` and
    `expectedHeadOid` equal to the new commit.
  - `test_auto_merge_not_confirmed_raises`: `autoMergeRequest` is null on
    read-back.
  - `test_api_failure_mid_way_raises_naming_step`
  - `test_only_own_job_prefix_closed`: a PR on `automation/other-job/…` is
    left alone.
- [ ] **Step 2: Run the tests and see them fail.**
- [ ] **Step 3: Implement**, through the Git Data API: blobs, then a tree
  with `base_tree`, then a commit, then `POST git/refs`. Then
  `POST pulls`, and the GraphQL auto-merge, then the read-back. Earlier PRs
  are the open PRs whose head starts with `automation/<job>/`.
- [ ] **Step 4: Run the tests and see them pass.**
- [ ] **Step 5: Commit** (`vrg-commit --type feat --scope automation`).

### Task 5: Desired state: auto-merge, the automation-prefix ruleset, fatal bypass reads

**Files:**

- Modify: `src/vergil_tooling/lib/github_config.py`
  (`desired_repo_settings`, `compute_desired_state`, `_diff_rulesets`)
- Modify: `src/vergil_tooling/bin/vrg_github_repo_config.py`
  (`_audit_repo`, `_apply_repo`: passing the mode and the App ID)
- Test: `tests/vergil_tooling/test_github_config_lib.py`

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
  - `class BypassUnreadable(NotChecked)`, raised when a desired non-empty
    bypass list reads back empty and `skip_bypass` is false.
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
  - `test_human_run_compares_bypass_actors`
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
- Consumes: `audit_files`, `FIXERS`, `sync_pr`, `compute_desired_state` and
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
    behaviour is byte-identical otherwise.
- [ ] **Step 2: Run the tests and see them fail.**
- [ ] **Step 3: Implement** `_update_repo`:
  1. identity gate;
  2. GitHub-side convergence (shared with `apply`);
  3. `fetch_tree(repo, "develop")`;
  4. `audit_files`;
  5. the `FIXERS` outputs, gathered into one `MechanizedChange` (job
     `repo-config-sync`), then `sync_pr`;
  6. the exit code.

  Update `config-audit.md`.
- [ ] **Step 4: Run the tests and see them pass.**
- [ ] **Step 5: Commit** (`vrg-commit --type feat --scope repo-config`).

### Task 7: Branch-sweep cleans finished automation branches

**Files:** Modify `src/vergil_tooling/lib/branch_sweep.py`
(`ruleset_rule_types`, `decide`). Test in
`tests/vergil_tooling/test_branch_sweep.py`.

- [ ] **Step 1: Write the failing tests.**
  - `test_finished_automation_branch_is_deleted`: a merged or closed PR on
    an `automation/` branch targeted only by the "Automation branches"
    ruleset is deleted.
  - `test_open_automation_pr_branch_kept`
  - `test_other_ruleset_protected_branch_still_skipped`
- [ ] **Step 2: Run the tests and see them fail.**
- [ ] **Step 3: Implement.** A branch is still exempt when any ruleset
  targets it, except when the only ruleset targeting it is "Automation
  branches". The exemption for every other ruleset, the rolling-tag branch
  ruleset included, is unchanged.
- [ ] **Step 4: Run the tests and see them pass.**
- [ ] **Step 5: Commit** (`vrg-commit --type fix --scope branch-sweep`).

### Task 8: `vrg-repo-status` shows automation PRs as `automation`

Requires epic vergil-project/.github#1049 to have shipped `lib/pending_release.py`.

**Files:** Modify `src/vergil_tooling/lib/pending_release.py` (classify),
and `docs/site/docs/reference/cli-tools-overview.md`. Test in
`tests/vergil_tooling/test_pending_release.py`.

- [ ] **Step 1: Write the failing test**
  `test_automation_prefix_classified_automation`: the branch
  `automation/repo-config-sync/123` gives type `automation`, checked before
  the conventional-commit fallback.
- [ ] **Step 2: Run the test and see it fail.**
- [ ] **Step 3: Implement**, and add the type to the TYPE column's docs.
- [ ] **Step 4: Run the test and see it pass.**
- [ ] **Step 5: Commit** (`vrg-commit --type feat --scope repo-status`).

### Task D3 (deployment): Release the tooling; the same-day sweep

Kind `deployment`, filed in vergil-tooling and blocked by 6, 7 and 8. Its
precondition is human-attested: vergil-tooling is released with Tasks 1–8.

- **Procedure:** the human runs `vrg-github-repo-config update --repo
  <repo>` for every managed repo in each org, the same day.
- **Acceptance:**
  - `vrg-github-repo-config audit` exits 0 on every repo;
  - each repo shows `allow_auto_merge` on, and the "Automation branches"
    ruleset with both bypass actors;
  - any sync PRs the sweep opened are recorded.

### Task A2: Ops switch: `github-config` runs `update` as automation

**Files:** Modify `.github/workflows/ops-github-config.yml` (the
`github-config` job).

- [ ] **Step 1:** Scope the token to Administration write, Contents write,
  Pull requests write, Workflows write and Metadata read.
- [ ] **Step 2:** Set `env: VRG_AUTOMATION_APP: ${{ vars.VRG_AUTOMATION_APP }}`,
  and add a first step that fails the job when it is empty.
- [ ] **Step 3:** Replace `audit` with `vrg-github-repo-config update --repo
  "${{ github.repository }}"`, and rename the step. Remove the
  `bypass-probe` job (Task A1, step 3b).
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
  - `bypass_actors` converges under the automation identity;
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

- [ ] **Step 1:** Confirm nothing imports the deleted modules outside the
  deleted files: `rg` over `src/` and `tests/`.
- [ ] **Step 2:** Delete them and remove the entry points and docs.
- [ ] **Step 3: Validate, then commit** (`vrg-commit --type refactor
  --scope sync`).
