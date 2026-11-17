# Parallel PR merging — implementation plan

- **Epic:** vergil-project/.github#1218
- **Planning issue:** vergil-project/.github#1213 (review history is in its
  comments)
- **Status:** draft, in design review

> **For agentic workers:** each task is one issue and one PR in the repo named
> in its heading, worked with `issue-implement`. Deployment tasks are worked
> with `issue-deploy`, validation tasks with `issue-validate`. Steps use
> checkbox (`- [ ]`) syntax for tracking.

**Goal:** a batch of ready PRs merges into `develop` in parallel. Submit opens
every PR with auto-merge on. One finalize run watches the batch, then syncs,
cleans up and validates `develop` once. The release gets its own develop-CD
gate and installs by default.

**Architecture:** the work lands in three releases, in the order of spec §6.12.

1. Two standalone `vrg-release` changes: the develop-CD gate and install by
   default.
2. The ruleset split, applied fleet-wide by the config audit, with the CD
   concurrency fix.
3. The merge rework, released together. New `lib/` modules own the pieces the
   commands share:
   - `checkout_lock` (the `flock`);
   - `auto_merge` (enable, read back, merge directly when already mergeable);
   - `cd_runs` (find, classify and watch a CD run);
   - `finalize_watch` (the watch phase).

   `vrg-finalize-pr` and `vrg-submit-pr` are rebuilt on them. pr-watch and
   merge-review learn the post-submit flow.

**Tech Stack:** Python 3 (vergil-tooling), pytest, `gh` through
`vergil_tooling.lib.github`, `fcntl.flock`, GitHub Actions YAML
(vergil-actions), Markdown skills (vergil-claude-plugin).

**Spec:** [`spec.md`](spec.md) in this directory. Executors read both.

## Global Constraints

- **`develop` drops the up-to-date rule in every repo; `main` keeps it**
  (spec §6.1). The split rulesets are named `CI gates (develop)` and
  `CI gates (main)`.
- **Merge methods:** auto-merge uses squash for a single-issue PR and rebase
  for a batch PR, pinned to the pushed head (`expectedHeadOid`) (spec §6.2).
  Submit opens PRs against `develop` only.
- **Finalize's exit codes are 0, 1, 3, 4 and 5** (spec §6.3):
  - 0: full success;
  - 1: a tool or usage error, or a validate that could not run;
  - 3: develop is healthy, but not every PR merged;
  - 4: a validate gate failed (with `--wait-cd`, also a CD failure);
  - 5: full success, but the chained release failed.

  4 takes precedence over 3. For submit, a 1 from finalize always wins
  (spec §6.2).
- **The batch deadline** is `$VRG_FINALIZE_DEADLINE` seconds, default `7200`.
  A non-positive or non-integer value is a configuration error before any
  waiting, like `$VRG_CD_TIMEOUT`.
- **The lock** is an exclusive `fcntl.flock` on
  `<main clone>/build/.vergil-state/checkout.lock`. It is not re-entrant.
  It covers only local sync, cleanup, provision, validate, release preflight's
  sync, and update-deps' fast-forward (spec §6.4).
- **Removed flags are fatal errors** that name the new default (spec §6.10):
  - `vrg-submit-pr`: `--finalize`, `--install`;
  - `vrg-release`: `--install`;
  - `vrg-finalize-pr`: `--install`, `--no-wait-cd`, `--skip-post-checks`,
    `--strategy`.
- **New flags:** `vrg-submit-pr --no-finalize`, `--no-install`,
  `--allow-provenance-violation`; `vrg-finalize-pr --wait-cd`, `--no-install`;
  `vrg-release --no-install`.
- **CD workflows keep `cancel-in-progress: false`** (spec §6.5).
- **Fail loudly:** no swallowed exceptions, no fallback that hides an error
  (repo engineering principles).
- **Process.** Validation runs only through `vrg-container-run --
  vrg-validate` in the task repo's worktree. Commit with `vrg-commit`; hand
  off with `vrg-pr-workflow report-ready`. Releases are human-gated; no agent
  cuts one.

## Review Focus

1. **A PR that merged before finalize started, listed next to PRs still
   running.** It counts as merged at once, the watch continues for the rest,
   and the settle phase runs once at the end. Pinned in Task 12a
   (`test_already_merged_counts`) and Task 12b
   (`test_settle_once_after_mixed_batch`).
2. **A finalize run killed while it holds the lock.** The next run acquires
   the lock at once and syncs again. Pinned in Task 8
   (`test_lock_released_when_holder_dies`, with a child process that takes
   the lock and is killed).
3. **`origin/develop` moving between finalize's validate and the chained
   release.** Release preflight fast-forwards and cuts from the new head
   instead of refusing. Pinned in Task 9 (`test_preflight_fast_forwards`) and
   Task 12b (`test_chained_release_runs_after_lock_released`).
4. **A batch PR whose required checks are already green when submit turns
   auto-merge on.** GitHub refuses auto-merge, and the helper merges directly
   with the rebase method and the head pin. Pinned in Task 10
   (`test_already_mergeable_merges_directly_with_method_and_head`).
5. **Mixed outcomes in one submit:** one PR's submit fails, another PR fails
   CI in the watch, and the validate could not run. Submit exits 1, names the
   failed submit, and does not release. Pinned in Task 13b
   (`test_exit_precedence_tool_error_wins`).

---

## Delivery table

| Task | Repo | Kind | What | Blocked by |
| --- | --- | --- | --- | --- |
| 1 | .github | task (#1219) | Spec and plan (this PR) | — |
| 2 | vergil-tooling | task | `lib/cd_runs.py` and release's develop-CD gate | — |
| 3 | vergil-tooling | task | `vrg-release` installs by default | — |
| D1 | .github | deployment | Release 1 (Tasks 2–3) attested and installed | 2, 3 |
| 4 | vergil-tooling | task | Split the CI gates ruleset; producibility covers both | — |
| 5 | vergil-tooling | task | Audit clears classic protection's `strict` on develop | 4 |
| 6 | vergil-actions | task | vergil-actions' own CD concurrency group per branch | — |
| 7 | vergil-tooling | task | `confirm-main` re-dispatches a superseded main CD run | 2 |
| D2 | .github | deployment | Release 2 (Tasks 4, 5, 7; actions Task 6) applied fleet-wide | D1, 4, 5, 6, 7 |
| 8 | vergil-tooling | task | `lib/checkout_lock.py`; cleanup and update-deps take it | — |
| 9 | vergil-tooling | task | Release preflight fast-forwards under the lock | 8 |
| 10 | vergil-tooling | task | `lib/auto_merge.py`; mechanized PRs use it | — |
| 11 | vergil-tooling | task | Provenance counts auto-merge events | — |
| 12c | vergil-tooling | task | Finalize reports CD; `--wait-cd` opts in | 2 |
| 12a | vergil-tooling | task | `lib/finalize_watch.py`: the watch phase | 10, 11 |
| 12b | vergil-tooling | task | `vrg-finalize-pr` rebuilt on the watch and the lock | 8, 9, 12a, 12c |
| 13a | vergil-tooling | task | `vrg-submit-pr`: one path, no rebase, develop only, auto-merge | 10, 11 |
| 13b | vergil-tooling | task | `vrg-submit-pr`: finalize by default, flags and exit status | 12b, 13a |
| 14 | vergil-claude-plugin | task | merge-review: whole-branch review of an open PR | — |
| 15 | vergil-claude-plugin | task | pr-watch for the auto-merge flow | 14 |
| 16 | vergil-claude-plugin | task | epic-create: the dev-image epic rule | — |
| 17 | vergil-tooling | task | Supersede the governance and batch-pipeline specs; update the permission model | 11, 12b, 13b |
| D3 | .github | deployment | Release 3 (Tasks 8–13b, 17; plugin 14–16) attested and installed | D2, 8–17 |
| V1 | .github | validation | Live parallel batch, concurrent finalize, auto-merge after a push | D3 |
| F | fixtures | fixture runs | python-app, python-lib, rust-app, rust-lib | D3 |
| — | vergil-tooling | bookend (#4529) | Documentation review | 2–17 |
| — | .github | bookend (#1220) | Retrospective | all |

Notes on the table:

- **Ordering by shared files.** Tasks 12c, 12a and 12b all change
  `bin/vrg_finalize_pr.py` and run in that order. Tasks 13a and 13b both
  change `bin/vrg_submit_pr.py`. Task 13b also retires finalize's
  `--skip-post-checks`, which Task 12b keeps so that submit's cascade works
  in between.
- **The retrospective** records that epic #1168's spec still says develop
  requires up-to-date checks (spec §6.9).
- **Task 3** keeps submit's and finalize's existing `--install` working until
  Tasks 12b and 13b replace those commands' flags.
- **Release 3 ships as one release** (spec §6.12 step 3). No task in it is
  deployed on its own.
- **No plugin skill passes a removed flag today.** Task 15 rewrites pr-watch's
  text, and Task 17 and the documentation-review sweep cover the docs.
- **The advisories tracked by #1213** are retargeted when the tasks are filed:
  - GHSA-78xw-g79w-7f62 to Tasks 11, 12a, 13a and 17;
  - GHSA-gcff-v4p4-8rcr to Tasks 11 and 17;
  - GHSA-35qq-6wpw-3gfw to Task 17;
  - GHSA-vv7q-29w3-225r, a design-only accepted risk stated by the spec, to
    the documentation task (#1219).

---

### Task 2: `lib/cd_runs.py` and release's develop-CD gate (vergil-tooling)

**Files:**

- Create: `src/vergil_tooling/lib/cd_runs.py`
- Modify:
  - `src/vergil_tooling/bin/vrg_finalize_pr.py`: move `_list_cd_runs`,
    `_match_cd_run`, `_view_cd_run`, `_supersede_group`, `CDRunProblem` and
    `CDRunSuperseded` out, and import them;
  - `src/vergil_tooling/lib/release/preflight.py` (call the gate);
  - `src/vergil_tooling/lib/release/confirm.py` (share the job
    classification).
- Test: `tests/vergil_tooling/test_cd_runs.py`,
  `tests/vergil_tooling/test_release_preflight.py`,
  `tests/vergil_tooling/test_vrg_finalize_pr.py` (imports only)

**Interfaces:**

- Produces, in `vergil_tooling.lib.cd_runs`:
  - the moved names, unchanged in behaviour;
  - `RunState = Literal["queued", "in_progress", "success", "failed", "timed_out", "superseded", "cancelled", "none"]`;
  - `find_run(repo: str, branch: str, head_sha: str, *, dispatch_timeout: float) -> dict | None`;
  - `watch_run(run_id: str, *, timeout: float) -> RunState`, which waits
    through `queued` and `pending` as well as `in_progress`;
  - `classify_jobs(jobs: list[dict]) -> tuple[list[str], list[str]]`, which
    returns `(blocking_failures, deferred_failures)` using
    `confirm._collect_deferred_publish`'s rule. That rule moves here, and
    confirm imports it.
- Produces, in `lib/release/preflight.py`:
  `check_develop_cd(repo: str, head_sha: str) -> None`, raising
  `ReleaseError(phase="preflight", …)` with the run URL.
  - A failed or timed-out run raises.
  - Deferred failures print a warning.
  - With no run registered within the dispatch window, it falls back to the
    latest completed run on an ancestor of `head_sha`, which must be green.
  - A superseded run is skipped in favour of the newer run.

- [ ] **Step 1: Write the failing tests** in `test_cd_runs.py`, with a fake `gh` runner:
  - `test_watch_waits_through_queued` (statuses `queued`, `pending`, `in_progress`, then `completed/success` → `success`);
  - `test_watch_timeout_is_timed_out`;
  - `test_classify_docs_publish_is_deferred` (a failed docs-publish job is in `deferred_failures` only).
- [ ] **Step 2: Write the failing tests** in `test_release_preflight.py`:
  - `test_develop_cd_gate_failed_run_refuses` (message contains the run URL);
  - `test_develop_cd_gate_queued_run_waits`;
  - `test_develop_cd_gate_no_run_uses_green_ancestor`;
  - `test_develop_cd_gate_no_run_red_ancestor_refuses`;
  - `test_develop_cd_gate_deferred_failure_warns`;
  - `test_develop_cd_gate_superseded_uses_newer`.
- [ ] **Step 3: Run** `vrg-container-run -- vrg-validate`; expect the new tests to fail.
- [ ] **Step 4: Implement** the module and the move. Then call
  `check_develop_cd` in `preflight()` right after `_check_branch_and_tree()`,
  using local `HEAD`.
- [ ] **Step 5: Run** `vrg-container-run -- vrg-validate`; expect PASS, with finalize's existing CD tests unchanged.
- [ ] **Step 6: Commit** `vrg-commit --type feat --scope release --message "require a green develop CD run before cutting a release"`, then report-ready.

### Task 3: `vrg-release` installs by default (vergil-tooling)

**Files:**

- Modify:
  - `src/vergil_tooling/bin/vrg_release.py` (parser and the install branch
    around line 105);
  - `bin/vrg_submit_pr.py` and `bin/vrg_finalize_pr.py`, only at the lines
    that build a `vrg-release` command (about 334, 775 and 1442 in submit;
    357 and 1944 in finalize).
- Test: `tests/vergil_tooling/test_vrg_release.py`,
  `tests/vergil_tooling/test_vrg_submit_pr.py`,
  `tests/vergil_tooling/test_vrg_finalize_pr.py`

**Interfaces:**

- Produces: `vrg-release [--no-install]`.
  - The install runs after a clean release when
    `[publish].consumer-refresh` is set.
  - With none set, it prints today's "nothing to run" line and is not an
    error.
  - `--install` exits 1 with
    `vrg-release: --install was removed; installing is now the default (use --no-install to skip it)`.
- **Interim, until Tasks 12b and 13b:** submit's and finalize's `--install`
  runs plain `vrg-release`, and their `--release` without `--install` runs
  `vrg-release --no-install`. Today's behaviour of both commands is
  unchanged.

- [ ] **Step 1: Write the failing tests:**
  - `test_install_runs_by_default`;
  - `test_no_install_skips`;
  - `test_install_flag_is_fatal` (exit 1, exact message);
  - `test_install_skipped_after_failed_release`;
  - in the submit and finalize tests, `test_release_cascade_passes_no_install` and `test_install_cascade_runs_plain_release`.
- [ ] **Step 2: Run** `vrg-container-run -- vrg-validate`; expect FAIL.
- [ ] **Step 3: Implement.**
- [ ] **Step 4: Run** `vrg-container-run -- vrg-validate`; expect PASS.
- [ ] **Step 5: Commit** `vrg-commit --type feat --scope release --message "install by default; --no-install skips it"`, then report-ready.

### Task D1: Release 1 attested and installed (deployment, .github)

Blocked by 2, 3.

- **Precondition (human-attested):** a vergil-tooling release containing
  Tasks 2 and 3 is published, its version named in a comment.
- **Human-run steps:** `vrg-host upgrade` on the host; `vrg-vm update --all`
  for the VMs.
- **Procedure (agent-safe):**
  1. `vrg-release --help` lists `--no-install` and not `--install`.
  2. `vrg-release --install` exits 1 with Task 3's message, and nothing
     changes.
  3. The release's notes list Tasks 2 and 3.
- **SUCCESS:** all three hold, recorded with the version.

### Task 4: Split the CI gates ruleset; producibility covers both (vergil-tooling)

**Files:**

- Modify: `src/vergil_tooling/lib/github_config.py`:
  - `desired_ci_gates_ruleset` becomes `desired_ci_gates_rulesets`, returning
    two rulesets;
  - `_CI_GATES_RULESET_NAME` becomes `CI_GATES_RULESET_NAMES`;
  - `_diff_producibility`;
  - the callers at about lines 684, 864 and 921.
- Modify: `src/vergil_tooling/lib/repo_init.py` (about line 1752) and
  `src/vergil_tooling/bin/vrg_github_repo_config.py` (about line 669).
- Test: `tests/vergil_tooling/test_github_config_lib.py`,
  `tests/vergil_tooling/test_vrg_github_repo_config.py`

**Interfaces:**

- Produces:
  - `CI_GATES_DEVELOP = "CI gates (develop)"` and
    `CI_GATES_MAIN = "CI gates (main)"`;
  - `desired_ci_gates_rulesets(...) -> tuple[DesiredRuleset, ...]`, with the
    same parameters as today. It returns the develop ruleset with
    `strict_required_status_checks_policy: False`, plus the main ruleset with
    `True` when the repo has a `main` branch model. `docs-single-branch`
    repos get the develop one only.
- **The legacy ruleset.** The shared `"CI gates"` ruleset is no longer
  desired, so the audit's existing apply order (write desired, then delete
  undesired) replaces it.
- **Producibility.** `_diff_producibility` checks each name in
  `CI_GATES_RULESET_NAMES`. A desired CI ruleset with no actual counterpart,
  or the reverse, is a diff item
  `rulesets.<name>.missing_ci_gates_ruleset`, never a silent return.

- [ ] **Step 1: Write the failing tests:**
  - `test_ci_gates_split_develop_not_strict_main_strict`;
  - `test_docs_single_branch_gets_develop_only`;
  - `test_legacy_ci_gates_is_undesired`;
  - `test_producibility_checks_both_rulesets`;
  - `test_producibility_missing_ruleset_is_error`;
  - `test_repo_init_creates_both`;
  - `test_repo_config_reports_both`.
- [ ] **Step 2: Run** `vrg-container-run -- vrg-validate`; expect FAIL.
- [ ] **Step 3: Implement.** Grep for every reference to the old name and
  function, so none is left.
- [ ] **Step 4: Run** `vrg-container-run -- vrg-validate`; expect PASS.
- [ ] **Step 5: Commit** `vrg-commit --type feat --scope github-config --message "split CI gates into develop and main rulesets; develop drops the up-to-date rule"`, then report-ready.

### Task 5: Audit clears classic protection's `strict` on develop (vergil-tooling)

**Files:** modify `src/vergil_tooling/lib/github_config.py` (classic
protection read and apply); test `tests/vergil_tooling/test_github_config_lib.py`.

**Interfaces:**

- Consumes: Task 4's names.
- Produces:
  - **Drift.** Classic protection on `develop` with
    `required_status_checks.strict == true` is a diff item
    `branch_protection.develop.strict`.
  - **Apply.** The fix sets `strict` to `false` and leaves every other
    classic setting unchanged. `main` is never touched.

- [ ] **Step 1: Write the failing tests:**
  - `test_classic_strict_on_develop_is_drift`;
  - `test_apply_clears_only_strict`;
  - `test_classic_strict_on_main_untouched`;
  - `test_no_classic_protection_no_item`.
- [ ] **Step 2: Run** `vrg-container-run -- vrg-validate`; expect FAIL.
- [ ] **Step 3: Implement.**
- [ ] **Step 4: Run** `vrg-container-run -- vrg-validate`; expect PASS.
- [ ] **Step 5: Commit** `vrg-commit --type feat --scope github-config --message "clear classic protection's up-to-date flag on develop"`, then report-ready.

### Task 6: vergil-actions' own CD concurrency group per branch (vergil-actions)

**Files:** modify `.github/workflows/cd.yml` (lines 11–13); test with the repo's
existing workflow lint (`actionlint` through `vrg-validate`).

- [ ] **Step 1: Verify the precondition the spec names** (§6.5). Read
  `cd.yml`, `cd-release.yml` and the release action. Record in the PR body
  whether anything in the `main` release path must be serialized with a
  `develop` run, for example a tag push racing develop's version bump.
  - If something must, stop and propose to the human that Task 7's backstop
    covers `cd`, with this task closed as not planned.
- [ ] **Step 2: Change** the group to `cd-${{ github.ref_name }}`, keeping
  `cancel-in-progress: false`. Leave the reusable `cd-docs.yml` and
  `cd-release.yml` groups unchanged.
- [ ] **Step 3: Run** `vrg-container-run -- vrg-validate`; expect PASS.
- [ ] **Step 4: Commit** `vrg-commit --type ci --scope cd --message "key the CD concurrency group by branch"`, then report-ready. The delta touches `.github/workflows/`, so the agent push is refused; report-ready records it anyway.

### Task 7: `confirm-main` re-dispatches a superseded main CD run (vergil-tooling)

**Files:** modify `src/vergil_tooling/lib/release/confirm.py` (`_watch_cd`,
`_poll_for_run`); test `tests/vergil_tooling/test_release_confirm.py`.

**Interfaces:**

- Consumes: Task 2's `cd_runs` (`find_run`, `_supersede_group`).
- Produces: in `confirm_main`, a `main` run that concluded `cancelled` and that
  `_supersede_group` identifies as superseded is dispatched again once.
  - The dispatch is `gh workflow run cd.yml --ref main`.
  - The new run is then watched.
  - A second supersede is a `ReleaseError`, with the resume command.
  - `confirm_develop` treats a supersede as a warning, as finalize does.

- [ ] **Step 1: Write the failing tests:**
  - `test_confirm_main_redispatches_superseded_once`;
  - `test_confirm_main_second_supersede_fails`;
  - `test_confirm_main_other_cancel_fails`;
  - `test_confirm_develop_supersede_warns`.
- [ ] **Step 2: Run** `vrg-container-run -- vrg-validate`; expect FAIL.
- [ ] **Step 3: Implement.**
- [ ] **Step 4: Run** `vrg-container-run -- vrg-validate`; expect PASS.
- [ ] **Step 5: Commit** `vrg-commit --type feat --scope release --message "re-dispatch a main CD run superseded before it started"`, then report-ready.

### Task D2: Release 2 applied fleet-wide (deployment, .github)

Blocked by D1, 4, 5, 6, 7.

- **Precondition (human-attested):**
  - a vergil-tooling release containing Tasks 4, 5 and 7 is published, and
    installed on the host and VMs;
  - the vergil-actions rolling tag carries Task 6;
  - both versions are named in a comment.
- **Human-run steps:** run the GitHub config audit with apply for the fleet,
  via the `ops.yml` workflow dispatch or the documented host command.
- **Procedure (agent-safe):**
  1. For every repo in the org, read the rulesets:
     - `CI gates (develop)` exists with strict `false`;
     - `CI gates (main)` exists with strict `true`, where the repo has
       `main`;
     - no `CI gates` ruleset remains.
  2. The audit's next report shows no `missing_ci_gates_ruleset` and no
     `branch_protection.develop.strict` item.
  3. In vergil-actions, the latest `main` and `develop` CD runs show the
     groups `cd-main` and `cd-develop`.
- **SUCCESS:** every repo passes step 1, and steps 2 and 3 hold. The comment
  lists any repo that failed and why.

### Task 8: `lib/checkout_lock.py`; cleanup and update-deps take it (vergil-tooling)

**Files:**

- Create: `src/vergil_tooling/lib/checkout_lock.py`
- Modify: `bin/vrg_finalize_pr.py` (`--cleanup-only` path and
  `_stage_cleanup`); `lib/release/update_deps_stage.py` (the
  `merge --ff-only` at line 50)
- Test: `tests/vergil_tooling/test_checkout_lock.py`, plus the two callers' tests

**Interfaces:**

- Produces: `checkout_lock(main_root: Path, *, purpose: str) -> ContextManager[None]`.
  - It creates `build/.vergil-state/` with the state-directory checks (real
    directories, mode 700).
  - It opens `checkout.lock` and writes `<pid> <purpose>`.
  - While it blocks, it prints
    `Waiting for the checkout lock (held by pid <pid>: <purpose>)…` once.
  - It holds `fcntl.LOCK_EX` until exit.
- **Nesting is a bug.** A nested acquire by the same process raises
  `CheckoutLockError("checkout lock already held by this process")`.
- **trivy stays as it is** unless it can use the helper unchanged.

- [ ] **Step 1: Write the failing tests:**
  - `test_lock_serializes_two_processes` (the second blocks until the first exits);
  - `test_lock_released_when_holder_dies` (a child takes the lock and is killed; acquiring succeeds at once);
  - `test_wait_message_names_holder`;
  - `test_nested_acquire_raises`;
  - `test_state_dir_symlink_refused`;
  - `test_cleanup_only_takes_lock`;
  - `test_update_deps_fast_forward_takes_lock`.
- [ ] **Step 2: Run** `vrg-container-run -- vrg-validate`; expect FAIL.
- [ ] **Step 3: Implement.**
- [ ] **Step 4: Run** `vrg-container-run -- vrg-validate`; expect PASS.
- [ ] **Step 5: Commit** `vrg-commit --type feat --scope checkout-lock --message "serialize local develop sync and cleanup with a checkout lock"`, then report-ready.

### Task 9: Release preflight fast-forwards under the lock (vergil-tooling)

**Files:** modify `src/vergil_tooling/lib/release/preflight.py`
(`_check_branch_and_tree`, about lines 617–646); test
`tests/vergil_tooling/test_release_preflight.py`.

**Interfaces:**

- Consumes: Task 8's `checkout_lock`; Task 2's `check_develop_cd`.
- Produces: `_check_branch_and_tree()`, under
  `checkout_lock(purpose="vrg-release preflight")`, does the following:
  1. refuses a dirty tree, as today;
  2. fetches `origin develop`;
  3. if local `develop` is an ancestor of `origin/develop`, fast-forwards it
     and prints `Fast-forwarded develop <old8>..<new8>`;
  4. otherwise refuses with
     `Local develop (<sha8>) has diverged from origin/develop (<sha8>); resolve it first.`

  It then releases the lock. `check_develop_cd` then runs on the new `HEAD`.

- [ ] **Step 1: Write the failing tests:**
  - `test_preflight_fast_forwards`;
  - `test_preflight_up_to_date_no_change`;
  - `test_preflight_diverged_refuses`;
  - `test_preflight_dirty_refuses`;
  - `test_preflight_holds_lock_only_for_sync` (the lock is free when `check_develop_cd` runs);
  - `test_cd_gate_checks_new_head`.
- [ ] **Step 2: Run** `vrg-container-run -- vrg-validate`; expect FAIL.
- [ ] **Step 3: Implement.**
- [ ] **Step 4: Run** `vrg-container-run -- vrg-validate`; expect PASS.
- [ ] **Step 5: Commit** `vrg-commit --type feat --scope release --message "fast-forward develop in preflight instead of refusing"`, then report-ready.

### Task 10: `lib/auto_merge.py`; mechanized PRs use it (vergil-tooling)

**Files:**

- Create: `src/vergil_tooling/lib/auto_merge.py`
- Modify: `src/vergil_tooling/lib/mechanized_pr.py`:
  - `_enable_auto_merge` (line 337) is replaced by the helper;
  - the module docstring (lines 24–35) no longer says develop requires
    up-to-date branches.
- Test: `tests/vergil_tooling/test_auto_merge.py`,
  `tests/vergil_tooling/test_mechanized_pr.py`

**Interfaces:**

- Produces:
  - `MergeMethod = Literal["SQUASH", "REBASE"]`;
  - `@dataclass(frozen=True) class AutoMergeResult: merged_now: bool`;
  - `enable_auto_merge(pr_url: str, *, head: str, method: MergeMethod) -> AutoMergeResult`.
- **What it does.** It enables auto-merge through GraphQL
  `enablePullRequestAutoMerge` with `expectedHeadOid=head` and reads the
  setting back.
  - **Already mergeable.** When GitHub refuses because the PR is already
    clean (a "clean status" refusal), it merges directly with `method` and
    `--match-head-commit head`, and returns `merged_now=True`.
  - **Any other refusal,** or a read-back that does not match, raises
    `AutoMergeError` naming the PR.
- Produces: `auto_merge_enabled(pr_url: str) -> bool`, reading
  `autoMergeRequest`.
- `mechanized_pr` calls `enable_auto_merge(..., method="SQUASH")`, so its
  behaviour is unchanged.

- [ ] **Step 1: Write the failing tests:**
  - `test_enables_and_reads_back`;
  - `test_already_mergeable_merges_directly_with_method_and_head`;
  - `test_other_refusal_raises`;
  - `test_readback_mismatch_raises`;
  - `test_mechanized_uses_helper_squash`.
- [ ] **Step 2: Run** `vrg-container-run -- vrg-validate`; expect FAIL.
- [ ] **Step 3: Implement.** Record the exact GraphQL refusal string the
  helper matches in a module constant, with the GitHub docs URL in a comment.
- [ ] **Step 4: Run** `vrg-container-run -- vrg-validate`; expect PASS.
- [ ] **Step 5: Commit** `vrg-commit --type refactor --scope auto-merge --message "share the auto-merge helper between submit and mechanized PRs"`, then report-ready.

### Task 11: Provenance counts auto-merge events (vergil-tooling)

**Files:** modify `src/vergil_tooling/lib/pr_provenance.py` (`_EVENT_MAP`,
lines 78–83); test `tests/vergil_tooling/test_pr_provenance.py`.

**Interfaces:**

- Produces:
  - `_EVENT_MAP` adds `"auto_merge_enabled": "auto_merge_enabled"` and
    `"auto_merge_disabled": "auto_merge_disabled"`.
  - `evaluate` treats both as forbidden for agent roles.
  - `check_pr(pr: str) -> ProvenanceResult` is unchanged in signature.
    Tasks 12a and 13a call it.

- [ ] **Step 1: Write the failing tests:**
  - `test_agent_auto_merge_enabled_is_violation`;
  - `test_agent_auto_merge_disabled_is_violation`;
  - `test_human_auto_merge_enabled_allowed`.
- [ ] **Step 2: Run** `vrg-container-run -- vrg-validate`; expect FAIL.
- [ ] **Step 3: Implement.**
- [ ] **Step 4: Run** `vrg-container-run -- vrg-validate`; expect PASS.
- [ ] **Step 5: Commit** `vrg-commit --type feat --scope provenance --message "treat agent auto-merge events as forbidden actions"`, then report-ready.

### Task 12c: Finalize reports CD; `--wait-cd` opts in (vergil-tooling)

**Files:**

- Modify: `bin/vrg_finalize_pr.py`: the parser (`--no-wait-cd` at about line
  297) and `_stage_cd_check` (line 1800);
- Modify: `lib/release/finalize.py` (`_FINALIZE_CMD`, which drops
  `--no-wait-cd`).
- Test: `tests/vergil_tooling/test_vrg_finalize_pr.py`,
  `tests/vergil_tooling/test_release_finalize.py`

**Interfaces:**

- Consumes: Task 2's `cd_runs`.
- Produces:
  - **The default.** `_stage_cd_check` finds the run for the synced head and
    prints `CD: <state> <url>` without waiting. With no run registered, it
    prints `CD: no CD run for this head`.
  - **`--wait-cd`** gives today's blocking watch, with failures classified
    by `classify_jobs`.
  - **`--no-wait-cd`** exits 1 with
    `vrg-finalize-pr: --no-wait-cd was removed; not waiting on CD is now the default (use --wait-cd to wait)`.

- [ ] **Step 1: Write the failing tests:**
  - `test_cd_reported_not_awaited_by_default`;
  - `test_wait_cd_blocks`;
  - `test_no_wait_cd_fatal`;
  - `test_no_cd_run_reported`;
  - `test_release_cleanup_call_has_no_no_wait_cd`.
- [ ] **Step 2: Run** `vrg-container-run -- vrg-validate`; expect FAIL.
- [ ] **Step 3: Implement.**
- [ ] **Step 4: Run** `vrg-container-run -- vrg-validate`; expect PASS.
- [ ] **Step 5: Commit** `vrg-commit --type feat --scope finalize --message "report develop CD instead of waiting; --wait-cd opts in"`, then report-ready.

### Task 12a: `lib/finalize_watch.py`: the watch phase (vergil-tooling)

**Files:** create `src/vergil_tooling/lib/finalize_watch.py`; test
`tests/vergil_tooling/test_finalize_watch.py`.

**Interfaces:**

- Consumes: Task 10's `enable_auto_merge`, `auto_merge_enabled`; Task 11's
  `check_pr`.
- Produces:
  - `Outcome = Literal["merged", "failed: CI", "failed: conflict", "failed: closed", "failed: auto-merge off", "failed: provenance", "timed out"]`;
  - `@dataclass(frozen=True) class PRResult: pr: str; outcome: Outcome; detail: str; head: str`.
    `detail` names the failed check, or the provenance actions.
  - `deadline_secs() -> int`, reading `$VRG_FINALIZE_DEADLINE`.
  - `watch(prs: Sequence[str], *, allow_provenance_violation: bool, poll_interval: float = 15.0) -> list[PRResult]`.
- **What `watch` does:**
  1. **Start-up.** For each listed PR, it runs provenance. Then, if the PR is
     open with auto-merge off, it restores auto-merge with the PR's method:
     rebase if the PR's commits name several issues
     (`batch_rules.is_batch`), squash otherwise. It prints
     `Restored auto-merge on <pr> at <head8>`.
  2. **Polling.** It polls every open PR until each is terminal or the
     deadline passes.
     - Provenance runs again whenever a PR's head changes.
     - A violation turns auto-merge off (GraphQL
       `disablePullRequestAutoMerge`) and gives `failed: provenance`, unless
       `allow_provenance_violation`.
  3. **Terminal states:**
     - `MERGED` gives merged;
     - a required check concluded failure gives `failed: CI`;
     - `mergeStateStatus == DIRTY` gives `failed: conflict`;
     - `CLOSED` gives `failed: closed`;
     - auto-merge going off mid-watch gives `failed: auto-merge off`;
     - the deadline gives `timed out`.

- [ ] **Step 1: Write the failing tests**, with a fake GitHub that scripts each PR's state sequence:
  - `test_already_merged_counts`;
  - `test_ci_failure_names_check`;
  - `test_conflict`;
  - `test_closed`;
  - `test_auto_merge_turned_off_midwatch`;
  - `test_restore_at_start_prints_head`;
  - `test_restore_already_mergeable_merges`;
  - `test_provenance_rechecked_on_new_head`;
  - `test_provenance_violation_disables_auto_merge`;
  - `test_allow_provenance_violation`;
  - `test_deadline_times_out_pending_only`;
  - `test_deadline_env_malformed_raises_before_polling`;
  - `test_one_poll_per_interval_for_ten_prs` (call count bound).
- [ ] **Step 2: Run** `vrg-container-run -- vrg-validate`; expect FAIL.
- [ ] **Step 3: Implement.**
- [ ] **Step 4: Run** `vrg-container-run -- vrg-validate`; expect PASS.
- [ ] **Step 5: Commit** `vrg-commit --type feat --scope finalize --message "add the batch watch phase"`, then report-ready.

### Task 12b: `vrg-finalize-pr` rebuilt on the watch and the lock (vergil-tooling)

**Files:**

- Modify: `src/vergil_tooling/bin/vrg_finalize_pr.py`:
  - `main`, the parser and `build_stages`;
  - remove `_stage_provenance`, `_stage_merge`, `_resolve_strategy` and
    `_run_finalize_batch`;
  - add the settle phase and the report.
- Test: `tests/vergil_tooling/test_vrg_finalize_pr.py`

**Interfaces:**

- Consumes: Task 12a's `watch`, `PRResult`; Task 8's `checkout_lock`; Task
  12c's CD stage; Task 9 (the chained release fast-forwards itself).
- **Input.** `vrg-finalize-pr <pr>[,<pr>…] | --all | (interactive)
  [--release] [--no-install] [--wait-cd] [--allow-provenance-violation]
  [--yes] [--dry-run] | --cleanup-only`.
- **The run.**
  1. `watch`.
  2. If anything merged: `checkout_lock(purpose="vrg-finalize-pr settle")`
     around close-refs, sync, cleanup, provision and validate.
  3. After the lock is released: the CD report, then `vrg-release
     [--no-install]` if `--release` and full success.
- **Exit codes** per Global Constraints. A validate whose
  `vrg-container-run` failed before the gate ran (exit status other than the
  gate's own failure) maps to 1, with `develop health unknown` in the report.
- **Removed flags.** `--strategy` and `--install` are fatal, each with a
  message naming the new default.
- **`--skip-post-checks` stays, for now.** It means "watch only, no settle
  phase". Submit's batch cascade and `lib/pr_workflow/batch.py` still call it
  until Task 13b removes them, together with this flag.

- [ ] **Step 1: Write the failing tests:**
  - `test_settle_once_after_mixed_batch`;
  - `test_nothing_merged_skips_settle`;
  - `test_exit_3_when_one_failed`;
  - `test_exit_4_validate_gate_failed_precedes_3`;
  - `test_exit_1_validate_could_not_run`;
  - `test_exit_5_release_failed`;
  - `test_release_only_on_full_success`;
  - `test_chained_release_runs_after_lock_released`;
  - `test_report_lists_merged_and_unmerged_with_reasons`;
  - `test_report_broken_develop_names_head_and_commits`;
  - `test_removed_flags_fatal` (parametrized over `--strategy` and `--install`);
  - `test_skip_post_checks_watches_without_settling`;
  - `test_all_selects_worktree_prs`;
  - `test_no_merge_call_anywhere` (no `pr_merge.wait_and_merge` import remains).
- [ ] **Step 2: Run** `vrg-container-run -- vrg-validate`; expect FAIL.
- [ ] **Step 3: Implement.** Delete the tests of removed stages along with
  the stages. Keep `--cleanup-only` on its own path.
- [ ] **Step 4: Run** `vrg-container-run -- vrg-validate`; expect PASS.
- [ ] **Step 5: Refactor.** Update the module docstring, which is the
  command's reference text. Run `vrg-container-run -- vrg-validate` again;
  expect PASS.
- [ ] **Step 6: Commit** `vrg-commit --type feat --scope finalize --message "finalize watches a batch, then settles develop once"`, then report-ready.

### Task 13a: `vrg-submit-pr`: one path, no rebase, develop only, auto-merge (vergil-tooling)

**Files:**

- Modify: `src/vergil_tooling/bin/vrg_submit_pr.py`:
  - `_run_template_mode`, `_run_cli_mode` and `_run_relay` all route to one
    `_submit_all(selection) -> SubmitSummary`;
  - remove `_run_submit_batch`'s rebase and its merge-commit refusal;
  - `_target_branch` refuses any base other than develop.
- Test: `tests/vergil_tooling/test_vrg_submit_pr.py`

**Interfaces:**

- Consumes: Task 10's `enable_auto_merge`; Task 11's `check_pr`.
- Produces:
  - `@dataclass(frozen=True) class SubmitSummary: opened: list[str]; failed: list[tuple[str, str]]`.
    `failed` holds `(branch or PR, reason)` pairs.
  - **The per-PR loop:**
    1. ready gate and freeze check;
    2. `_push_branch`;
    3. `_create_pr`;
    4. `check_pr`: a violation, unless `--allow-provenance-violation`, fails
       this PR and leaves auto-merge off;
    5. `enable_auto_merge(method=…)`: REBASE when `batch_rules.is_batch`,
       SQUASH otherwise;
    6. `record_submission`.
  - **Base other than develop.** `--base` or a recorded base naming another
    branch exits 1 with
    `vrg-submit-pr: PRs are opened against develop only; releases go through vrg-release`.
  - **The submit-time batch-PR check stays as it is** (`_recheck_batch`):
    `check_hard_rules` and `check_commit_per_issue`.
  - The cascade flags still work through the old per-PR finalize chain until
    Task 13b.

- [ ] **Step 1: Write the failing tests:**
  - `test_single_and_batch_share_submit_all`;
  - `test_no_rebase_at_submit`;
  - `test_base_main_refused`;
  - `test_auto_merge_squash_single`;
  - `test_auto_merge_rebase_batch`;
  - `test_provenance_violation_fails_pr_others_continue`;
  - `test_auto_merge_refusal_fails_pr_others_continue`;
  - `test_relay_mode_uses_submit_all`.
- [ ] **Step 2: Run** `vrg-container-run -- vrg-validate`; expect FAIL.
- [ ] **Step 3: Implement.**
- [ ] **Step 4: Run** `vrg-container-run -- vrg-validate`; expect PASS.
- [ ] **Step 5: Commit** `vrg-commit --type feat --scope submit --message "open every PR through one path with auto-merge on"`, then report-ready.

### Task 13b: `vrg-submit-pr`: finalize by default, flags and exit status (vergil-tooling)

**Files:**

- Modify: `src/vergil_tooling/bin/vrg_submit_pr.py`:
  - the parser, around lines 191–256;
  - `_chain_finalize`;
  - the cascade normalization at 123–140;
  - the batch and relay cascades, at about 764–790 and 1421–1452.
- Modify: `src/vergil_tooling/lib/pr_workflow/batch.py`, which drops the
  per-PR finalize loop and `finalize_post_checks_step`.
- Modify: `src/vergil_tooling/bin/vrg_finalize_pr.py`, where
  `--skip-post-checks` becomes fatal.
- Test: `tests/vergil_tooling/test_vrg_submit_pr.py`,
  `tests/vergil_tooling/pr_workflow/test_batch.py`,
  `tests/vergil_tooling/test_vrg_finalize_pr.py`

**Interfaces:**

- Consumes: Task 13a's `SubmitSummary`; Task 12b's CLI and exit codes.
- **Default.** One `vrg-finalize-pr <opened…> [--release] [--no-install]
  [--allow-provenance-violation]`.
  - `--release` is dropped when `summary.failed` is non-empty, and the
    summary prints `Release skipped: <n> PR(s) failed to submit`.
  - After finalize's report, submit prints the failed submits.
- **`--no-finalize`** prints
  `Not watching. To finalize these PRs: vrg-finalize-pr <pr>,<pr>,…`.
- **Exit:**
  - 1 if finalize returned 1;
  - otherwise the higher of finalize's code and 3, when anything failed to
    submit;
  - otherwise finalize's code.

  With `--no-finalize`, submit exits 3 if anything failed to submit.
- **Fatal flags.** `--finalize` and `--install` exit 1, each naming the new
  default. So do `--release` with `--no-finalize`, and `--no-install`
  without `--release` (`--no-install requires --release`).

- [ ] **Step 1: Write the failing tests:**
  - `test_finalize_by_default_one_run_over_opened`;
  - `test_release_skipped_after_failed_submit`;
  - `test_no_finalize_prints_command`;
  - `test_exit_precedence_tool_error_wins`;
  - `test_exit_failed_submit_at_least_3`;
  - `test_exit_finalize_4_wins_over_3`;
  - `test_removed_and_invalid_flag_combinations_fatal` (parametrized);
  - `test_dry_run_plans_without_side_effects`.
- [ ] **Step 2: Run** `vrg-container-run -- vrg-validate`; expect FAIL.
- [ ] **Step 3: Implement.** Remove the per-PR `--skip-post-checks` loops and
  their post-checks step. Then make finalize's `--skip-post-checks` fatal
  (`test_skip_post_checks_fatal` in the finalize tests, replacing Task 12b's
  `test_skip_post_checks_watches_without_settling`).
- [ ] **Step 4: Run** `vrg-container-run -- vrg-validate`; expect PASS.
- [ ] **Step 5: Refactor.** Update the module docstring and the printed
  hints. Run `vrg-container-run -- vrg-validate` again; expect PASS.
- [ ] **Step 6: Commit** `vrg-commit --type feat --scope submit --message "finalize by default; --no-finalize opts out"`, then report-ready.

### Task 14: merge-review: whole-branch review of an open PR (vergil-claude-plugin)

**Files:** modify `skills/merge-review/SKILL.md`; test with the plugin's skill
lint through `vrg-validate`.

- [ ] **Step 1: Check the tooling.** Confirm whether
  `vrg-merge-review-request --issue <ref>` runs against a task whose PR is
  open and submitted, with no stored earlier step. A fresh first step needs
  no `--fix-base`.
  - If the tooling refuses (for example, it checks the freeze or the
    report-ready state), stop and propose a vergil-tooling sub-task (14t) to
    the human. Do not change the tooling here.
- [ ] **Step 2: Add the entry** "Fix to an open PR". It is called by pr-watch
  before a push that is not a clean rebase. It covers:
  - a fresh review state (`--ensure-dir`);
  - a whole-branch review against `origin/develop`, with the same reviewer,
    outcomes and ruling flow;
  - the closing comment on the task issue carrying the eligibility line;
  - no report-ready.
- [ ] **Step 3: Run** `vrg-container-run -- vrg-validate`; expect PASS.
- [ ] **Step 4: Commit** `vrg-commit --type feat --scope merge-review --message "review a fix to an open PR as a whole branch"`, then report-ready.

### Task 15: pr-watch for the auto-merge flow (vergil-claude-plugin)

**Files:** modify `skills/pr-watch/SKILL.md`.

- [ ] **Step 1: Rewrite the skill** for the auto-merge flow (spec §6.8):
  - **Routine rebase.** A conflict caused by another merge is rebased onto
    `origin/develop` and force-pushed with no review.
  - **Batch PRs.** Before any push to a batch PR, run the batch-PR check
    (`check_commit_per_issue` plus the closing-references check, through the
    same `vrg-pr-workflow` command submit's check uses), and refuse on a
    violation.
  - **Other fixes.** Any other fix goes through Task 14's entry, and is
    pushed only when eligible.
  - **After a fix,** tell the human to rerun `vrg-finalize-pr <PR>`.
  - **Review comments.** A human review comment after submit is addressed but
    does not hold the merge.
  - **Corrected text.** The notes that say the human merges every PR, and
    that agents never turn auto-merge on, are corrected.
  - **Reasons.** The finalize report's unmerged reasons map to actions.
- [ ] **Step 2: Run** `vrg-container-run -- vrg-validate`; expect PASS.
- [ ] **Step 3: Commit** `vrg-commit --type feat --scope pr-watch --message "watch PRs that auto-merge"`, then report-ready.

### Task 16: epic-create: the dev-image epic rule (vergil-claude-plugin)

**Files:** modify `skills/epic-create/SKILL.md` ("Operational tasks").

- [ ] **Step 1: Add the rule** (spec §6.5): an epic that depends on freshly
  published `dev-` images states it in its plan and seeds a validation task
  for the CD build, since finalize no longer waits on CD.
- [ ] **Step 2: Run** `vrg-container-run -- vrg-validate`; expect PASS.
- [ ] **Step 3: Commit** `vrg-commit --type docs --scope epic-create --message "seed a CD-build validation when an epic needs fresh dev images"`, then report-ready.

### Task 17: Supersede the governance and batch-pipeline specs; update the permission model (vergil-tooling)

**Files:** modify `docs/specs/2026-05-11-org-governance-design.md`,
`docs/specs/2026-06-17-batch-pr-pipeline-design.md` and
`docs/specs/2026-05-29-agent-permission-model-design.md`.

- [ ] **Step 1: Governance spec.** Add a dated supersession note to "Why
  'Require Up to Date' Is Retained", pointing at
  `vergil-project/.github` `epics/1218-parallel-pr-merging/spec.md`. Drop the
  up-to-date line from "Rules for `develop`", and update the risks-table row
  on merge serialization.
- [ ] **Step 2: Batch-pipeline spec.** Add a dated supersession note on its
  zero-waste serial design. Its merge-queue research stands.
- [ ] **Step 3: Permission-model spec.** The provenance mitigation now names:
  - the checks at submit and in finalize's watch phase;
  - the actions the check reads, including the auto-merge events;
  - the windows it runs in;
  - an accepted gap covering everything else, to revisit if any agent
    identity gains server-side rights to act on PRs (spec §6.7, §6.9).
- [ ] **Step 4: Run** `vrg-container-run -- vrg-validate`; expect PASS.
- [ ] **Step 5: Commit** `vrg-commit --type docs --scope specs --message "supersede the up-to-date rationale; update the provenance mitigation"`, then report-ready.

### Task D3: Release 3 attested and installed (deployment, .github)

Blocked by D2, 8–17.

- **Precondition (human-attested):** a vergil-tooling release containing
  Tasks 8–13b and 17, and a vergil-claude-plugin release containing Tasks
  14–16, are published, their versions named in a comment.
- **Human-run steps:** `vrg-host upgrade` and `vrg-host plugins` on the host;
  `vrg-vm update --all`.
- **Procedure (agent-safe):**
  1. `vrg-submit-pr --help` lists `--no-finalize`, `--no-install` and
     `--allow-provenance-violation`, and not `--finalize`.
  2. `vrg-finalize-pr --help` lists `--wait-cd` and not `--strategy`.
  3. `vrg-finalize-pr --no-wait-cd` exits 1 with Task 12c's message.
  4. The installed pr-watch skill mentions auto-merge and the routine rebase.
  5. Both releases' notes list their tasks.
- **SUCCESS:** all five hold, recorded with both versions.

### Task V1: Live parallel batch (validation, .github)

Blocked by D3.

- **Precondition (machine probe):** D3 is closed with `Outcome: SUCCESS`.
  `vrg-finalize-pr --help` lists `--wait-cd`.
- **Procedure:** the human runs the submits. The agent prepares the branches
  and records what happens.
  1. In one repo, prepare three independent, reported-ready branches that
     touch different files, plus a second batch of two.
  2. The human runs `vrg-submit-pr --all --yes` on the first batch. Record
     that every PR opened with auto-merge on, and that no `BEHIND` update
     happened.
  3. While it watches, the human submits the second batch from another
     terminal. Record that its settle phase printed the lock-wait line, then
     validated.
  4. On one open PR, push a commit as the agent identity pr-watch uses.
     Record whether GitHub kept auto-merge enabled (spec §10).
  5. Record both finalize reports and their exit codes, and that develop
     validated green.
- **SUCCESS:**
  - all five PRs merged without `BEHIND`;
  - the second settle waited for the lock;
  - both runs exited 0;
  - step 4's result is recorded either way. If auto-merge turned off, the
    next finalize restored it.

### Fixture runs (F)

Seed with `fixture-run vergil-project/.github#1218 python-app,python-lib,rust-app,rust-lib --validates "vergil-tooling release 3 of epic #1218 (auto-merge submit, batch finalize, install-by-default release)" --after <D3 ref>`, once D3 is filed. The change reaches release tooling and the PR flow every fixture uses (spec §6.11).
