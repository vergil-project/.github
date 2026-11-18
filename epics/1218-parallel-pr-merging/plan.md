# Parallel PR merging — implementation plan

- **Epic:** vergil-project/.github#1218
- **Planning issue:** vergil-project/.github#1213 (review history is in its
  comments)
- **Status:** approved

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
  - 1: a tool or usage error;
  - 3: develop is healthy, but not every PR merged;
  - 4: the validate returned non-zero for any reason (with `--wait-cd`, also
    a CD failure);
  - 5: full success, but the chained release failed.

  4 takes precedence over 3. For submit, a 1 from finalize always wins
  (spec §6.2).
- **The batch deadline** is `$VRG_FINALIZE_DEADLINE` seconds, default `7200`.
  A non-positive or non-integer value is a configuration error before any
  waiting, like `$VRG_CD_TIMEOUT`.
- **The lock** is an exclusive `fcntl.flock` on
  `<main clone>/build/.vergil-state/checkout.lock`. It is not re-entrant.
  It covers only local sync, cleanup, provision, validate, release preflight's
  sync, and update-deps' fast-forward and preflight sync (spec §6.4).
  Close-refs runs before it is taken.
- **Refactor before commit.** Every code task ends with a refactor pass over
  what it touched (stale docstrings, leftover helpers, duplicated logic),
  then a re-run of `vrg-container-run -- vrg-validate`, before its commit
  step. The tasks below do not repeat this step.
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
   CI in the watch, and the validate fails. Submit exits 4, names the failed
   submit, and does not release. Pinned in Task 13b
   (`test_exit_mixed_outcomes_validate_failed`).

---

## Delivery table

| Task | Repo | Kind | What | Blocked by |
| --- | --- | --- | --- | --- |
| 1 | .github | task (#1219) | Spec and plan (this PR) | — |
| 2 | vergil-tooling | task | `lib/cd_runs.py` and release's develop-CD gate | — |
| 3 | vergil-tooling | task | `vrg-release` installs by default | 2 |
| D1 | .github | deployment | Release 1 (Tasks 2–3) attested and installed | 2, 3 |
| 4 | vergil-tooling | task | Split the CI gates ruleset; producibility covers both | — |
| 5 | vergil-tooling | task | Audit clears classic protection's `strict` on develop | 4 |
| 6 | vergil-actions | task | vergil-actions' own CD concurrency group per branch | — |
| 7 | vergil-tooling | task | `confirm-main` re-dispatches a superseded main CD run | 2 |
| 20 | vergil-actions | task | Ruleset docs describe the develop and main rulesets | 4 |
| D2 | .github | deployment | Release 2 (Tasks 4, 5, 7; actions Tasks 6, 20) applied fleet-wide | D1, 4, 5, 6, 7, 20 |
| 8 | vergil-tooling | task | `lib/checkout_lock.py`; cleanup and update-deps take it | 2, 3 |
| 9 | vergil-tooling | task | Release preflight fast-forwards under the lock | 8 |
| 10 | vergil-tooling | task | `lib/auto_merge.py`; mechanized PRs use it | — |
| 11 | vergil-tooling | task | Provenance counts auto-merge events | — |
| 18 | vergil-tooling | task | `vrg-pr-workflow batch-check --pr` | 2, 3 |
| 12c | vergil-tooling | task | Finalize reports CD; `--wait-cd` opts in | 2, 3, 8, 18 |
| 12a | vergil-tooling | task | `lib/finalize_watch.py`: the watch phase | 10, 11 |
| 12b | vergil-tooling | task | `vrg-finalize-pr` rebuilt on the watch and the lock | 8, 9, 12a, 12c, 18 |
| 13a | vergil-tooling | task | `vrg-submit-pr`: one path, no rebase, develop only, auto-merge | 3, 10, 11 |
| 13b | vergil-tooling | task | `vrg-submit-pr`: finalize by default, flags and exit status | 12b, 13a |
| 14 | vergil-claude-plugin | task | merge-review: whole-branch review of an open PR | — |
| 15 | vergil-claude-plugin | task | pr-watch for the auto-merge flow | 14, 18 |
| 16 | vergil-claude-plugin | task | epic-create: the dev-image epic rule | — |
| 19 | vergil-claude-plugin | task | Skills and the agent-merge hook describe the auto-merge flow | 15 |
| 17 | vergil-tooling | task | Supersede the governance and batch-pipeline specs; update the permission model | 11, 12b, 13b |
| D3 | .github | deployment | Release 3 (Tasks 8–13b, 17, 18; plugin 14–16, 19) attested and installed | D2, 8–19 |
| V1 | .github | validation | Live parallel batch, concurrent finalize, auto-merge after a push | D3 |
| F | fixtures | fixture runs | python-app, python-lib, rust-app, rust-lib, ts-lib, ts-app | D3 |
| — | vergil-tooling | bookend (#4529) | Documentation review | 2–20 |
| — | .github | bookend (#1220) | Retrospective | all |

Notes on the table:

- **Ordering by shared files.** The Blocked-by edges keep changes to the same
  file apart.
  - **`bin/vrg_finalize_pr.py`** is changed by Tasks 2, 3, 8, 18, 12c and
    12b, in that order. Task 12a only adds a new module.
  - **`bin/vrg_submit_pr.py`** is changed by Tasks 3, 13a and 13b, in that
    order.
- **Interim flags.** Task 13b retires finalize's `--skip-post-checks` and
  `--install`. Task 12b keeps both, so that submit's cascade works in
  between.
- **The retrospective** records that epic #1168's spec still says develop
  requires up-to-date checks (spec §6.9).
- **Task 3** keeps submit's and finalize's existing `--install` working until
  Tasks 12b and 13b replace those commands' flags.
- **Release 3 ships as one release** (spec §6.12 step 3). No task in it is
  deployed on its own. Release 1 (Tasks 2 and 3) ships and is validated on
  its own by D1, ahead of everything else.
- **No plugin skill passes a removed flag today.** Task 15 rewrites pr-watch's
  text, and Task 17 and the documentation-review sweep cover the docs.
- **The advisories tracked by #1213** are retargeted when the tasks are filed:
  - GHSA-78xw-g79w-7f62 to Tasks 11, 12a, 13a and 17;
  - GHSA-gcff-v4p4-8rcr to Tasks 11 and 17;
  - GHSA-35qq-6wpw-3gfw to Task 17;
  - GHSA-fpm4-wh9m-2cm4 to Task 11;
  - GHSA-36g5-mf7w-77cr, now design-only because finalize no longer restores
    auto-merge, to the documentation task (#1219);
  - GHSA-rpx5-46r2-f2qm to Task 12a;
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
  - `test_develop_cd_gate_superseded_uses_newer`;
  - `test_develop_cd_gate_skipped_on_resume`;
  - `test_develop_cd_gate_other_cancel_refuses` (a cancellation that is not a supersede).
- [ ] **Step 3: Run** `vrg-container-run -- vrg-validate`; expect the new tests to fail.
- [ ] **Step 4: Implement** the module and the move. Then call
  `check_develop_cd` in `preflight()` inside the existing `if not resume:`
  block, after `_check_branch_and_tree()`, using local `HEAD`. A resume
  adopts a release that already passed the gate.
- [ ] **Step 5: Run** `vrg-container-run -- vrg-validate`; expect PASS, with finalize's existing CD tests unchanged.
- [ ] **Step 6: Commit** `vrg-commit --type feat --scope release --message "require a green develop CD run before cutting a release"`, then report-ready.

### Task 3: `vrg-release` installs by default (vergil-tooling)

**Files:**

- Modify:
  - `src/vergil_tooling/bin/vrg_release.py` (parser and the install branch
    around line 105);
  - `bin/vrg_submit_pr.py` and `bin/vrg_finalize_pr.py`: the lines that build
    a `vrg-release` command (about 775 and 1442 in submit; 357 and 1944 in
    finalize), and the hint strings that name one (finalize's dry-run line at
    about 219 and its rerun hint at about 1953).
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
- **One source per command.** Each place builds its `vrg-release` command as
  a tuple from one helper (`_release_cmd(install: bool) -> tuple[str, ...]`
  in each file), and every hint that names the command is printed from that
  same tuple (`shlex.join`). A hint can then never drift from what runs.

- [ ] **Step 1: Write the failing tests:**
  - `test_install_runs_by_default`;
  - `test_no_install_skips`;
  - `test_install_flag_is_fatal` (exit 1, exact message);
  - `test_install_skipped_after_failed_release`;
  - in the submit and finalize tests, `test_release_cascade_passes_no_install` and `test_install_cascade_runs_plain_release`;
  - `test_rerun_hint_matches_command` and `test_dry_run_hint_matches_command` in the finalize tests.
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
  - `test_drift_report_names_ruleset_split` (the report names the legacy `CI gates` ruleset being replaced by the pair);
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
  `_supersede_group` identifies as superseded is dispatched again once, but
  only when its **release job** was among the superseded jobs, or every job
  was.
  - **Only non-release jobs superseded** (for example the docs job, whose
    `cd-docs` group develop shares): no re-dispatch. Today's handling stays:
    the release job is verified, and the cancelled job is recorded as
    "publish deferred".
  - The dispatch is `gh workflow run cd.yml --ref main`. Before dispatching,
    confirm records the superseded run's ID and the dispatch time.
  - **Finding the new run.** Confirm polls for a run on `main` with the same
    head SHA whose ID differs from the superseded one, whose event is
    `workflow_dispatch`, and which was created at or after the dispatch time.
    That run is then watched. The cancelled run is never watched again.
  - A second supersede is a `ReleaseError`, with the resume command.
  - `confirm_develop` treats a supersede as a warning, as finalize does.

- [ ] **Step 1: Write the failing tests:**
  - `test_confirm_main_redispatches_superseded_once`;
  - `test_docs_only_supersede_no_redispatch` (the release job green, only the docs job superseded: no dispatch, publish deferred);
  - `test_redispatch_ignores_cancelled_run` (until the dispatched run registers, the only matching run is the cancelled one; confirm keeps polling and does not report a second supersede);
  - `test_confirm_main_second_supersede_fails`;
  - `test_confirm_main_other_cancel_fails`;
  - `test_confirm_develop_supersede_warns`.
- [ ] **Step 2: Run** `vrg-container-run -- vrg-validate`; expect FAIL.
- [ ] **Step 3: Implement.**
- [ ] **Step 4: Run** `vrg-container-run -- vrg-validate`; expect PASS.
- [ ] **Step 5: Commit** `vrg-commit --type feat --scope release --message "re-dispatch a main CD run superseded before it started"`, then report-ready.

### Task 20: Ruleset docs describe the develop and main rulesets (vergil-actions)

**Files:** modify `docs/site/docs/ci-gates/repository-rulesets.md` (about
lines 13, 40 and 47), and `required-checks.md` and `security-scanning.md`
where they name the `CI gates` ruleset.

**Interfaces:**

- Consumes: Task 4's names, `CI gates (develop)` and `CI gates (main)`.

- [ ] **Step 1: Rewrite the rulesets page.** Describe two rulesets:
  `develop`'s with "strict required status checks: no", and `main`'s with
  "yes (branch must be up to date)". Give the reason in one paragraph, linking
  the spec: `vergil-project/.github` `epics/1218-parallel-pr-merging/spec.md`
  §2.
- [ ] **Step 2: Update the links** in `required-checks.md` and
  `security-scanning.md` to name the ruleset each page means.
- [ ] **Step 3: Note the Scorecard change.** Where the docs discuss the
  OpenSSF Scorecard's Branch-Protection check, note that the up-to-date item
  for `develop` is gone, and that this is accepted.
- [ ] **Step 4: Run** `vrg-container-run -- vrg-validate`; expect PASS,
  including the docs build.
- [ ] **Step 5: Commit** `vrg-commit --type docs --scope ci-gates --message "describe the separate develop and main CI gates rulesets"`, then report-ready.

### Task D2: Release 2 applied fleet-wide (deployment, .github)

Blocked by D1, 4, 5, 6, 7, 20.

- **Precondition (human-attested):**
  - a vergil-tooling release containing Tasks 4, 5 and 7 is published, and
    installed on the host and VMs;
  - the vergil-actions rolling tag carries Task 6, and Task 20's docs are
    published;
  - both versions are named in a comment.
- **Human-run steps:** run the GitHub config audit with apply for the fleet,
  via the `ops.yml` workflow dispatch or the documented host command.
- **Procedure (agent-safe):**
  1. Read the config audit's report for the org (`vrg-github-repo-config`'s
     audit output, which reads each repo's rulesets; the agent has no raw
     `gh api` access). Confirm for every repo:
     - `CI gates (develop)` exists with strict `false`;
     - `CI gates (main)` exists with strict `true`, where the repo has
       `main`;
     - no `CI gates` ruleset remains.

     If the report does not show strict per ruleset, the human runs the
     ruleset read and posts its output for the agent to check.
  2. The audit's next report shows no `missing_ci_gates_ruleset` and no
     `branch_protection.develop.strict` item.
  3. In vergil-actions, `cd.yml` at the tag the latest runs used reads
     `group: cd-${{ github.ref_name }}`. A `develop` CD run and a `main` CD
     run that overlapped in time both completed, and neither was cancelled.
     If no overlap has happened yet, record that, and the check passes on
     the file alone.
- **SUCCESS:** every repo passes step 1, and steps 2 and 3 hold. The comment
  lists any repo that failed and why.

### Task 8: `lib/checkout_lock.py`; cleanup and update-deps take it (vergil-tooling)

**Files:**

- Create: `src/vergil_tooling/lib/checkout_lock.py`
- Modify: `bin/vrg_finalize_pr.py` (the `--cleanup-only` entry point only);
  `lib/release/update_deps_stage.py` (the `merge --ff-only` at line 50)
- Test: `tests/vergil_tooling/test_checkout_lock.py`, plus the two callers' tests

**Interfaces:**

- Produces: `checkout_lock(main_root: Path, *, purpose: str) -> ContextManager[None]`.
  - It creates `build/.vergil-state/` with the state-directory checks (real
    directories, mode 700).
  - **Order:**
    1. open `checkout.lock` without truncating it;
    2. try `fcntl.LOCK_EX | fcntl.LOCK_NB`;
    3. if that fails, read the holder line, print
       `Waiting for the checkout lock (held by pid <pid>: <purpose>)…` once,
       then block on `LOCK_EX`;
    4. only once it holds the lock, truncate the file and write
       `<pid> <purpose>`.

    A waiter therefore never overwrites the holder's record.
  - It holds the lock until exit.
- **Nesting is a bug.** A nested acquire by the same process raises
  `CheckoutLockError("checkout lock already held by this process")`.
- **Taken only at command-level entry points:** the `--cleanup-only` path and
  update-deps' fast-forward here, and finalize's settle phase in Task 12b.
  Stages such as `_stage_cleanup` never take it. Their docstrings say the
  caller must hold it.
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
(`_check_branch_and_tree`, about lines 617–646) and
`src/vergil_tooling/lib/update_deps/preflight.py` (about lines 142–153); test
`tests/vergil_tooling/test_release_preflight.py`,
`tests/vergil_tooling/test_update_deps_preflight.py`.

**Interfaces:**

- Consumes: Task 8's `checkout_lock`; Task 2's `check_develop_cd`.
- Produces: `_check_branch_and_tree()`, under
  `checkout_lock(purpose="vrg-release preflight")`, does the following:
  1. refuses a dirty tree, as today;
  2. fetches `--tags --force origin develop`, as today, so the
     already-tagged check sees remote tags;
  3. if local `develop` is an ancestor of `origin/develop`, fast-forwards it
     and prints `Fast-forwarded develop <old8>..<new8>`;
  4. otherwise refuses with
     `Local develop (<sha8>) has diverged from origin/develop (<sha8>); resolve it first.`

  It then releases the lock. `check_develop_cd` then runs on the new `HEAD`.
- **Update-deps' preflight.** When the release calls it, update-deps'
  preflight does the same: under the lock, it fast-forwards a local develop
  that is behind `origin/develop`, instead of refusing unless the two are
  equal. Run standalone, its behaviour is unchanged.
  - **The mechanism:** `run_update(…, fast_forward: bool = False)` passes the
    flag to the preflight. Only `lib/release/update_deps_stage.py` passes
    `fast_forward=True`. `vrg-update-deps` never does.

- [ ] **Step 1: Write the failing tests:**
  - `test_preflight_fast_forwards`;
  - `test_preflight_up_to_date_no_change`;
  - `test_preflight_diverged_refuses`;
  - `test_preflight_dirty_refuses`;
  - `test_preflight_holds_lock_only_for_sync` (the lock is free when `check_develop_cd` runs);
  - `test_preflight_fetch_keeps_tags`;
  - `test_update_deps_preflight_fast_forwards_from_release`;
  - `test_update_deps_preflight_standalone_unchanged`;
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
  - the module docstring's whole numbered list (about lines 17–35, including
    the lines added by `d75d93a2f7`) no longer says develop requires
    up-to-date branches, or that a branch update brings an older-base PR up
    to date on `develop`.
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
  - **It wraps `AutoMergeError`** with `_fail("enabling auto-merge", repo,
    exc)`, so its callers still see only `MechanizedPRError`, as
    `open_sync_pr`'s docstring promises.

- [ ] **Step 1: Write the failing tests:**
  - `test_enables_and_reads_back`;
  - `test_already_mergeable_merges_directly_with_method_and_head`;
  - `test_other_refusal_raises`;
  - `test_readback_mismatch_raises`;
  - `test_mechanized_uses_helper_squash`;
  - `test_mechanized_auto_merge_refusal_is_mechanized_error`;
  - `test_mechanized_already_mergeable_merges_directly` (the accepted behaviour change, spec §6.7).
- [ ] **Step 2: Run** `vrg-container-run -- vrg-validate`; expect FAIL.
- [ ] **Step 3: Implement.** Record the exact GraphQL refusal string the
  helper matches in a module constant, with the GitHub docs URL in a comment.
- [ ] **Step 4: Run** `vrg-container-run -- vrg-validate`; expect PASS.
- [ ] **Step 5: Commit** `vrg-commit --type refactor --scope auto-merge --message "share the auto-merge helper between submit and mechanized PRs"`, then report-ready.

### Task 11: Provenance counts auto-merge events (vergil-tooling)

**Files:** modify `src/vergil_tooling/lib/pr_provenance.py` (`_EVENT_MAP`,
lines 78–83, and `_collect_actions`); create
`tests/vergil_tooling/fixtures/pr_provenance/` (recorded timeline payloads);
test `tests/vergil_tooling/test_pr_provenance.py`.

**Interfaces:**

- Produces:
  - `_EVENT_MAP` maps every auto-merge **enable** event name recorded in
    Step 1 (expected to include `auto_merge_enabled`, `auto_squash_enabled`
    and `auto_rebase_enabled`) to `auto_merge_enabled`, which `evaluate`
    treats as forbidden for agent roles.
  - **Fail-closed fallback.** Any other timeline event matching `auto_*_enabled`
    maps to `auto_merge_enabled` too, so a renamed or new enable event by an
    agent identity is a violation rather than ignored.
  - **Disable events are not mapped.** An agent turning auto-merge off is not
    a provenance violation. Finalize reports any PR with auto-merge off as an
    exception anyway (Task 12a).
  - `check_pr(pr: str) -> ProvenanceResult` is unchanged in signature.
    Tasks 12a and 13a call it.

- [ ] **Step 1: Record the real event names. This step is human-run:**
  `vrg-gh` denies the agent `pr create`, auto-merge and `gh api`.
  1. **The human** creates a scratch PR in a fixture repo and enables
     auto-merge on it once with SQUASH, then (after disabling it) once with
     REBASE.
  2. **The human** saves `gh api repos/<o>/<r>/issues/<n>/timeline --paginate`
     output after each enable, into the agent's worktree under the fixtures
     directory.
  3. **The agent** commits the fixtures and lists the event names in the PR
     body.

  Until the fixtures exist, the agent stops with "blocked: awaiting the
  human-recorded timeline fixtures". It never writes synthetic payloads in
  their place.
- [ ] **Step 2: Write the failing tests:**
  - `test_recorded_squash_enable_is_violation_for_agent`;
  - `test_recorded_rebase_enable_is_violation_for_agent`;
  - `test_disable_is_not_a_violation`;
  - `test_unknown_auto_enable_fails_closed` (a synthetic `auto_frobnicate_enabled` by an agent is a violation);
  - `test_human_auto_merge_enabled_allowed`.

  Each test uses a recorded fixture, not a synthetic name, except the
  unknown-event and disable tests.
- [ ] **Step 2a: Run** `vrg-container-run -- vrg-validate`; expect FAIL.
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

- Consumes: Task 10's `auto_merge_enabled` (read only: the watch never turns
  auto-merge on); Task 11's `check_pr`.
- Produces:
  - `Outcome = Literal["merged", "failed: CI", "failed: conflict", "failed: closed", "failed: auto-merge off", "failed: provenance", "failed: not a develop PR", "timed out"]`;
  - `@dataclass(frozen=True) class PRResult: pr: str; outcome: Outcome; detail: str; head: str; provenance_violation: bool = False`.
    `detail` names the failed check, or the provenance actions.
  - `deadline_secs() -> int`, reading `$VRG_FINALIZE_DEADLINE`.
  - `watch(prs: Sequence[str], *, allow_provenance_violation: bool, poll_interval: float = 15.0) -> list[PRResult]`.
- **What `watch` does:**
  1. **Start-up.** For each listed PR, in this order:
     - **Develop PRs only.** A PR whose base is not `develop`, or whose head
       is `release/*`, ends at once as `failed: not a develop PR`, untouched.
     - **Already merged.** A merged PR ends at once as `merged`. Provenance
       still runs on it. A violation sets `provenance_violation=True` and is
       printed loudly, and no disable call is made. Task 12b treats such a
       run as not a full success.
     - **Provenance.** On an open PR, a violation, unless `allow_provenance_violation`,
       turns auto-merge off (GraphQL `disablePullRequestAutoMerge`) and ends
       the PR at once as `failed: provenance`.
     - **Auto-merge off.** An open PR with auto-merge off ends at once as
       `failed: auto-merge off`. It is reported as an exception for the human
       and never turned back on.
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
  - `test_merged_pr_with_violation_flagged_not_disabled` (no disable call; outcome `merged`, `provenance_violation=True`);
  - `test_ci_failure_names_check`;
  - `test_conflict`;
  - `test_closed`;
  - `test_auto_merge_turned_off_midwatch`;
  - `test_auto_merge_off_at_start_is_exception` (reported, never turned on: no enable call is made);
  - `test_startup_violation_disables_and_fails`;
  - `test_release_branch_not_watched`;
  - `test_non_develop_base_not_watched`;
  - `test_watch_never_enables_auto_merge` (no `enable_auto_merge` import or GraphQL enable call in the module);
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
  2. If anything merged: close-refs, with no lock. Then
     `checkout_lock(purpose="vrg-finalize-pr settle")` around sync, cleanup,
     provision and validate.
  3. After the lock is released: the CD report, then `vrg-release
     [--no-install]` if `--release` and full success.
- **Full success** means every PR merged, none with `provenance_violation`,
  the validate is green, and, with `--wait-cd`, CD is green. A merged PR with
  a violation makes the run exit 3, and no release runs.
- **Report.** Each unmerged PR gets a next-step line: "hand it to an agent,
  then rerun `vrg-finalize-pr <pr>`". For `failed: auto-merge off` and
  `failed: provenance`, the line is "investigate before re-enabling
  auto-merge".
- **Exit codes** per Global Constraints. Any non-zero validate is 4, and the
  report carries the validate's log path and its last 40 lines.
- **Removed flags.** `--strategy` is fatal, with a message naming the new
  default.
- **Kept for now, until Task 13b:**
  - **`--skip-post-checks`** means "watch only, no settle phase". Submit's
    batch cascade and `lib/pr_workflow/batch.py` still call it.
  - **`--install`** is still accepted and maps to a plain `vrg-release`
    after full success, because submit's single-PR `--install` cascade
    (`_chain_finalize`) still passes it.

  Task 13b removes both, together with their callers.

- [ ] **Step 1: Write the failing tests:**
  - `test_settle_once_after_mixed_batch`;
  - `test_nothing_merged_skips_settle`;
  - `test_exit_3_when_one_failed`;
  - `test_exit_4_validate_failed_precedes_3`;
  - `test_exit_4_validate_could_not_run_reports_log`;
  - `test_settle_under_real_lock` (a real `checkout_lock`, not a mock: the
    settle steps run once, with no nested-acquire error);
  - `test_exit_5_release_failed`;
  - `test_release_only_on_full_success`;
  - `test_chained_release_runs_after_lock_released`;
  - `test_report_lists_merged_and_unmerged_with_reasons`;
  - `test_report_names_next_step_per_unmerged_pr`;
  - `test_report_broken_develop_names_head_and_commits`;
  - `test_strategy_flag_fatal`;
  - `test_install_still_chains_plain_release` (submit's single-PR `--install` cascade works after this task);
  - `test_merged_with_violation_blocks_release` (exit 3, no release);
  - `test_close_refs_runs_outside_lock`;
  - `test_skip_post_checks_watches_without_settling`;
  - `test_all_selects_worktree_prs`;
  - `test_cleanup_only_settles_without_watch` (sync, cleanup, provision and validate under the lock, then the CD report; a failed validate is 4);
  - `test_interactive_no_open_prs_offers_cleanup_only`;
  - `test_no_merge_call_anywhere` (no `pr_merge.wait_and_merge` import remains).
- [ ] **Step 2: Run** `vrg-container-run -- vrg-validate`; expect FAIL.
- [ ] **Step 3: Implement.** Delete the tests of removed stages along with
  the stages. Keep `--cleanup-only` on its own path, as the "settle develop
  with no PRs" mode (spec §6.3): the settle steps with no watch, under the
  lock, with the same exit codes.
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
- **Deadline first.** Unless `--no-finalize` is given, submit calls
  `finalize_watch.deadline_secs()` before the per-PR loop, so a malformed
  `$VRG_FINALIZE_DEADLINE` exits 1 before any PR opens.
- **Default.** One `vrg-finalize-pr <opened…> [--release] [--no-install]
  [--allow-provenance-violation]`.
  - **Nothing opened.** When `summary.opened` is empty, finalize is not run
    at all. Submit prints the failed submits and exits 3. An empty list never
    reaches finalize's interactive pick.
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
  - `test_all_submits_failed_skips_finalize`;
  - `test_release_skipped_after_failed_submit`;
  - `test_no_finalize_prints_command`;
  - `test_exit_precedence_tool_error_wins` (finalize 1 plus a failed submit → 1);
  - `test_exit_mixed_outcomes_validate_failed` (a failed submit, a CI failure and a failed validate → 4, the failed submit named, no release);
  - `test_deadline_checked_before_any_pr_opens` (a malformed `$VRG_FINALIZE_DEADLINE` exits 1 before the first push; skipped with `--no-finalize`);
  - `test_exit_failed_submit_at_least_3`;
  - `test_exit_finalize_4_wins_over_3`;
  - `test_removed_and_invalid_flag_combinations_fatal` (parametrized);
  - `test_dry_run_plans_without_side_effects`;
  - `test_single_confirmation_for_selection` (one plan and one confirmation for one PR or many; none with `--yes`).
- [ ] **Step 2: Run** `vrg-container-run -- vrg-validate`; expect FAIL.
- [ ] **Step 3: Implement.** Remove the per-PR `--skip-post-checks` loops and
  their post-checks step. Change `_chain_finalize` to pass `--release` (and
  `--no-install`) instead of `--install`. Then make finalize's
  `--skip-post-checks` and `--install` fatal (`test_skip_post_checks_fatal`
  and `test_finalize_install_fatal` in the finalize tests, replacing Task
  12b's `test_skip_post_checks_watches_without_settling` and
  `test_install_still_chains_plain_release`).
- [ ] **Step 4: Run** `vrg-container-run -- vrg-validate`; expect PASS.
- [ ] **Step 5: Refactor.** Update the module docstring and the printed
  hints. Run `vrg-container-run -- vrg-validate` again; expect PASS.
- [ ] **Step 6: Commit** `vrg-commit --type feat --scope submit --message "finalize by default; --no-finalize opts out"`, then report-ready.

### Task 18: `vrg-pr-workflow batch-check --pr` (vergil-tooling)

**Files:**

- Modify: `src/vergil_tooling/lib/pr_workflow/batch_rules.py`, which gains
  `check_closing_refs`, moved out of finalize's `_batch_issues` and
  `_make_batch_check` (`bin/vrg_finalize_pr.py` about lines 1284–1360);
- Modify: `src/vergil_tooling/bin/vrg_pr_workflow.py` (a new `batch-check`
  subcommand);
- Modify: `bin/vrg_finalize_pr.py`, whose merge path calls the moved function
  until Task 12b deletes that path.
- Test: `tests/vergil_tooling/pr_workflow/test_batch_rules.py`,
  `tests/vergil_tooling/test_vrg_pr_workflow.py`

**Interfaces:**

- Produces:
  - `check_closing_refs(pr_body: str, issues: list[str], *, repo: str) -> None`,
    raising `BatchRuleError` naming the mismatch when the body's `Closes`
    refs differ from the batch's issues;
  - `vrg-pr-workflow batch-check --pr <url> [--head <sha>]`. It reads the PR's
    body and base, and runs `check_commit_per_issue(issues, base=<base>,
    head=<sha or the PR head>)` and `check_closing_refs`. It does **not** run
    `check_hard_rules`.
  - **Exit status:** 0 when the PR passes or is not a batch PR; 1 with the
    rule's message otherwise.
  - It runs on a submitted, frozen branch: it reads only, and makes no freeze
    check.

- [ ] **Step 1: Write the failing tests:**
  - `test_check_closing_refs_match`;
  - `test_check_closing_refs_missing_issue`;
  - `test_check_closing_refs_extra_issue`;
  - `test_batch_check_passes_non_batch_pr`;
  - `test_batch_check_fixup_commit_fails`;
  - `test_batch_check_uses_given_head`;
  - `test_batch_check_allowed_when_frozen`;
  - `test_finalize_merge_path_uses_moved_check` (finalize's behaviour is unchanged until Task 12b).
- [ ] **Step 2: Run** `vrg-container-run -- vrg-validate`; expect FAIL.
- [ ] **Step 3: Implement.**
- [ ] **Step 4: Run** `vrg-container-run -- vrg-validate`; expect PASS.
- [ ] **Step 5: Commit** `vrg-commit --type feat --scope pr-workflow --message "add batch-check for a submitted batch PR"`, then report-ready.

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
  - **Batch PRs.** Before any push to a batch PR, run Task 18's
    `vrg-pr-workflow batch-check --pr <url> --head <sha>` on the head about to
    be pushed, and refuse to push on a non-zero exit.
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

### Task 19: Skills and the agent-merge hook describe the auto-merge flow (vergil-claude-plugin)

**Files:** modify `skills/fixture-run/SKILL.md` (about line 275),
`skills/epic-implement/SKILL.md` (about line 312),
`skills/issue-implement/SKILL.md` (about line 821) and
`hooks/scripts/block-agent-merge.sh` (about line 58).

- [ ] **Step 1: Correct the skill text** so each describes the new flow (spec
  §1, §6.2, §6.3):
  - the human's `vrg-submit-pr` opens the PRs with auto-merge on and
    finalizes by default;
  - `vrg-finalize-pr` watches and settles, and never merges;
  - a batch PR rebase-merges through auto-merge.
- [ ] **Step 2: Correct the hook message** to: "Merging is a human action:
  the human's `vrg-submit-pr` turns auto-merge on." Keep the hook's behaviour
  unchanged.
- [ ] **Step 3: Search the plugin** for any other description of a separate
  finalize merge (`grep -rn "finalize" skills hooks agents`), and correct
  each one, or list it in the PR body with why it is still accurate.
- [ ] **Step 4: Run** `vrg-container-run -- vrg-validate`; expect PASS.
- [ ] **Step 5: Commit** `vrg-commit --type docs --scope skills --message "describe submit-time auto-merge in skills and the agent-merge hook"`, then report-ready.

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
  on merge serialization. Note that the OpenSSF Scorecard's Branch-Protection
  check will no longer list "up-to-date branches required on develop", and
  that this is accepted (spec §6.1).
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

Blocked by D2, 8–19.

- **Precondition (human-attested):** a vergil-tooling release containing
  Tasks 8–13b, 17 and 18, and a vergil-claude-plugin release containing
  Tasks 14–16 and 19, are published, their versions named in a comment.
- **Human-run steps:** `vrg-host upgrade` and `vrg-host plugins` on the host;
  `vrg-vm update --all`.
- **Procedure (agent-safe):**
  1. `vrg-submit-pr --help` lists `--no-finalize`, `--no-install` and
     `--allow-provenance-violation`, and not `--finalize`.
  2. `vrg-finalize-pr --help` lists `--wait-cd` and not `--strategy`.
  3. `vrg-finalize-pr --no-wait-cd` exits 1 with Task 12c's message.
  4. The installed pr-watch skill mentions auto-merge and the routine rebase.
  5. `vrg-pr-workflow batch-check --help` succeeds and shows `--pr` and
     `--head`.
  6. The installed `block-agent-merge.sh` message describes submit-time
     auto-merge.
  7. Both releases' notes list their tasks.
- **SUCCESS:** all seven hold, recorded with both versions.

### Task V1: Live parallel batch (validation, .github)

Blocked by D3.

- **Precondition (machine probe):** D3 is closed with `Outcome: SUCCESS`.
  `vrg-finalize-pr --help` lists `--wait-cd`.
- **Procedure:** the human runs the submits. The agent prepares the branches
  and records what happens.
  1. In one repo, prepare three independent, reported-ready branches that
     touch different files, plus a second batch of two.
  2. The human runs `vrg-submit-pr --select <first three issues> --yes`.
     Record that every PR opened with auto-merge on, and that no `BEHIND`
     update happened.
  3. While it watches, the human runs
     `vrg-submit-pr --select <other two issues> --yes` from another terminal.
     Record whether its settle phase printed the lock-wait line. The lock's
     serialization is proven by Task 8's two-process tests, so a wait here
     is observed, not required.
  4. On one open PR, push a commit as the agent identity pr-watch uses.
     Record whether GitHub kept auto-merge enabled (spec §10).
  5. Record both finalize reports and their exit codes, and that develop
     validated green.
- **SUCCESS:**
  - all five PRs merged without `BEHIND`;
  - whether the second settle waited for the lock is recorded;
  - both runs exited 0, except that the run watching step 4's PR may exit 3
    with that PR *failed: auto-merge off*;
  - step 4's result is recorded either way. If GitHub turned auto-merge off
    on the push, the comment says so, the human turns it back on by hand,
    and the result is proposed to the human as a follow-on finding. That
    case does not fail V1.

### Fixture runs (F)

Seed with `fixture-run vergil-project/.github#1218 python-app,python-lib,rust-app,rust-lib,ts-lib,ts-app --validates "vergil-tooling release 3 of epic #1218 (auto-merge submit, batch finalize, install-by-default release)" --after <D3 ref>`, once D3 is filed. The change reaches release tooling and the PR flow every fixture uses (spec §6.11).

## Evolution during execution

### 2026-11-17: Two follow-up tasks, a new blocker for Task 15, a new V1 step, and Task 11's recorded fixtures

**What changed:**

- **A new code task, vergil-project/vergil-tooling#4584:** `vrg-merge-review-request` carries every batch issue's scope for a fix to an open batch PR. It has no blockers, and D3 covers it.
- **Task 15 (vergil-project/vergil-claude-plugin#1121) now also carries `Blocked-by: vergil-project/vergil-tooling#4584`.**
- **A new code task, vergil-project/vergil-tooling#4585:** the release gate's ancestor-run refusal no longer reports a 0s wait. It carries `Blocked-by: vergil-project/vergil-tooling#4561` (Task 2) and `Blocked-by: vergil-project/vergil-tooling#4567` (Task 9), and D3 covers it.
- **V1 (vergil-project/.github#1255) gains a step and an acceptance criterion:** it checks GitHub's refusal text when auto-merge is turned on for a PR that is already mergeable. The change is a [human scope change on #1255](https://github.com/vergil-project/.github/issues/1255#issuecomment-6952835283).
- **Task 11 (vergil-project/vergil-tooling#4569) Step 1's fixtures were recorded by the human** on the throwaway PR vergil-project/vergil-fixture-python-lib#20. The real enable events are `auto_squash_enabled` and `auto_rebase_enabled`, and the disable event is `auto_merge_disabled`.
- The delivery table and every task section above are unchanged. The plan froze when its first tasks merged (vergil-project/vergil-actions#1273 and vergil-project/vergil-claude-plugin#1122).

**Why:**

- **#4584:** the merge review of vergil-project/vergil-claude-plugin#1120 (Task 14) found that the whole-branch review of a fix to a batch PR diffs the whole branch under one issue's scope. The sibling issues' commits then read as out-of-scope work, so such a review could never end `eligible`. Task 14 keeps the spec §6.8 whole-branch rule, and Task 15 must not rely on fixes to batch PRs reaching `eligible` until #4584 lands. Filed on [the human's `file` ruling of R1](https://github.com/vergil-project/vergil-claude-plugin/issues/1120#issuecomment-6952696352).
- **#4585:** in `lib/release/preflight.py`, the ancestor-run check judges the run with a zero timeout, so its refusal says the run is "still not complete after 0s". The gate still refuses safely, but the message misleads. Tasks 2 and 9 change the same file, so #4585 runs after both. Raised as [follow-up F2](https://github.com/vergil-project/.github/issues/1218#issuecomment-6952754109).
- **V1's new step:** `lib/auto_merge.py` (Task 10, vergil-project/vergil-tooling#4568) treats GitHub's refusal text for an already-mergeable PR as the signal to merge directly. GitHub does not document that text, and no V1 step submitted a PR whose required checks were already green. Raised as [follow-up F1](https://github.com/vergil-project/.github/issues/1218#issuecomment-6952573508).
- **Task 11's fixtures:** Step 1 is human-run, because `vrg-gh` denies the agent auto-merge and `gh api`. The recorded names replace the plan's expected list, which had also included `auto_merge_enabled` as an enable event.
- The human ruled on F1, F2 and [the F3 proposal](https://github.com/vergil-project/.github/issues/1218#issuecomment-6952786586) (which filed this entry's task, vergil-project/.github#1264) in [the rulings on the epic](https://github.com/vergil-project/.github/issues/1218#issuecomment-6952880275).

### 2026-11-17: Five follow-up tasks, a release constraint, Release 3 pieces in an earlier release, and a CD report change

**What changed:**

- **A new code task, vergil-project/vergil-claude-plugin#1131:** the `merge-review` skill and the `merge-reviewer` agent use the new `vrg-merge-review-request --pr` batch fields. It carries `Blocked-by: vergil-project/vergil-tooling#4584`. Until it lands, a fix to a batch PR still cannot end `eligible`.
- **Task 15 (vergil-project/vergil-claude-plugin#1121) now also carries `Blocked-by: vergil-project/vergil-claude-plugin#1131`.** D3 covers it.
- **A release constraint between Tasks 13a and 13b:** 13a (vergil-project/vergil-tooling#4574) and 13b (vergil-project/vergil-tooling#4575) ship in the same vergil-tooling release, and no vergil-tooling release is cut after 13a merges and before 13b merges. No code works around the gap between them.
- **Release 3 pieces shipped early in vergil-tooling v2.1.322**, the release that carried Releases 1 and 2: vergil-project/vergil-tooling#4568 (Task 10), #4569 (Task 11), #4572 (Task 12a) and #4584. They had merged before the cut.
- **A new code task, vergil-project/vergil-tooling#4648:** `vrg-pr-await` waits for the PR to merge. It exits 0 when the PR merges, and 1, with a named reason, when it cannot merge.
- **A new code task, vergil-project/vergil-claude-plugin#1142:** pr-watch adopts the new `vrg-pr-await` contract. It carries `Blocked-by: vergil-project/vergil-tooling#4648` and `Blocked-by: vergil-project/vergil-claude-plugin#1121` (Task 15).
- **Task 12c (vergil-project/vergil-tooling#4571) changed its CD report:** a repo with no CD workflow gets `CD: no CD workflow in this repo`, with no warning.
- **A new code task, vergil-project/vergil-tooling#4662:** `--allow-provenance-violation` covers only an open PR, never one that has already merged with a violation. It carries `Blocked-by: vergil-project/vergil-tooling#4573` (Task 12b), and Task 13b (vergil-project/vergil-tooling#4575) now also carries `Blocked-by: vergil-project/vergil-tooling#4662`.
- **A new code task, vergil-project/vergil-claude-plugin#1147:** pr-watch waits on `vrg-pr-await` in the background. It carries `Blocked-by: vergil-project/vergil-claude-plugin#1142`.
- #4648, #1142, #4662 and #1147 ship in Release 3, and D3 (vergil-project/.github#1254) covers them.
- The delivery table and every task section above are unchanged.

**Why:**

- **#1131:** #4584 added the batch fields to the request, but nothing reads them until the plugin passes `--pr` on every step of an open-PR run and the reviewer judges each commit against its own issue's scope. The merge review of #4584 raised this as follow-up N0.1, and the human [ruled it `file`](https://github.com/vergil-project/vergil-tooling/issues/4584#issuecomment-6953363979): a task of its own, blocked by #4584 and not folded into Task 15. Task 15 must not rely on a fix to a batch PR reaching `eligible` before #1131 lands.
- **The 13a and 13b constraint:** from Task 13a on, submit turns auto-merge on, but until 13b its `--finalize`, `--release` and `--install` chain still calls the old finalize, which can fail if GitHub merges the PR during its wait. The merge review of 13a raised this as R1, and the human [ruled it `keep`](https://github.com/vergil-project/vergil-tooling/issues/4574#issuecomment-6961476729): no interim special case in code, with the gap closed by release discipline instead. The rule held: 13a merged after v2.1.322 was cut, and v2.1.323 carries both 13a and 13b.
- **The early Release 3 pieces:** they had merged before v2.1.322 was cut, so that release carried them. No behaviour regressed: #4572 and #4584 were not used yet, and #4568 and #4569 change only mechanized PRs and agent provenance. The plan said Release 3 ships as one release, so the retrospective should record the exception. It was [recorded for the sweep](https://github.com/vergil-project/vergil-tooling/issues/4529#issuecomment-6961412667).
- **#4648 and #1142:** `vrg-pr-await` treated a merged PR as an error, but under auto-merge at submit a PR merging during a watch is the normal way it lands, so pr-watch had to read exit 1 two ways. Raised as [follow-up F5](https://github.com/vergil-project/.github/issues/1218#issuecomment-6962241909), which the human [ruled `file`](https://github.com/vergil-project/.github/issues/1218#issuecomment-6962563891): `vrg-pr-await` follows the new flow, and nothing depends on its exit code in a roundabout way.
- **Task 12c's CD report:** a repo with no CD workflow would have warned `CD: unknown` on every finalize, where it used to pass quietly. The merge review of #4571 raised this as R1, and the human [ruled it `change`](https://github.com/vergil-project/vergil-tooling/issues/4571#issuecomment-6962616352). Any other lookup failure keeps the warning.
- **#4662:** spec §6.3 was ambiguous about what `--allow-provenance-violation` overrides in finalize's watch phase. Task 12b applied it to an already-merged PR too, so with the flag such a run could reach full success and release. Raised as [follow-up F7](https://github.com/vergil-project/.github/issues/1218#issuecomment-6963763578), which the human ruled `file` in the same comment: a PR that has already merged with a violation always blocks full success and any chained release.
- **#1147:** #4648 makes `vrg-pr-await` wait up to `$VRG_FINALIZE_DEADLINE` (7200s by default), but pr-watch ran it in the foreground, where a Bash call is capped at 600s, so a CI run longer than 10 minutes had its wait killed. Raised as [follow-up F6](https://github.com/vergil-project/.github/issues/1218#issuecomment-6963745209), which the human [ruled `file`](https://github.com/vergil-project/.github/issues/1218#issuecomment-6963807664): it is still one wait, with no polling.

### 2026-11-18: Release 3 cut before the documentation sweep

**What changed:**

- **Release 3 was cut once all of its code had merged:** vergil-tooling v2.1.323 and vergil-claude-plugin v2.1.76. The documentation-review sweep (vergil-project/vergil-tooling#4529) and its per-repo doc tasks had not merged yet.
- **The epic's one security item:** security: 1 findings: 0 fix, 0 record, 1 reject ([ruling](https://github.com/vergil-project/.github/issues/1218#issuecomment-6977495350)).

**Why:**

- **The early cut:** the human [gave the Release 3 green light](https://github.com/vergil-project/.github/issues/1218#issuecomment-6978492910) once all Release 3 code had merged, without waiting for the documentation sweep.
