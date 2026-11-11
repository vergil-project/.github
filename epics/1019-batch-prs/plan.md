# Batched same-repo PRs Implementation Plan

- **Epic:** vergil-project/.github#1019
- **Planning issue:** vergil-project/.github#1017 (review history is in its comments)
- **Status:** approved

> **For agentic workers:** each task is one PR, implemented with the
> `issue-implement` skill (operational tasks with `issue-deploy` or
> `issue-validate`). Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** one agent implements up to 8 small same-epic, same-repo issues on
one branch, one commit per issue, and hands them off as one PR that is
rebase-merged and closes every issue.

**Architecture:** the handoff chain (workflow state, report-ready, submit,
finalize, the epic sweep) carries an ordered issue list. Two shared checks in
one new module, the hard rules and the commit-per-issue invariant, run at
batch start, report-ready, submit and finalize. `vrg-git` lifts its local-only
denials on unfrozen feature branches, and `vrg-commit` gains `--fixup`. The
skills change last, in the spec's delivery order.

**Tech stack:** Python 3 (vergil-tooling `src/vergil_tooling`, pytest under
`tests/vergil_tooling`), Markdown skills (vergil-claude-plugin `skills/`).

**Spec:** [`epics/1019-batch-prs/spec.md`](spec.md)

## Global constraints

- Single-issue PRs behave exactly as they do today, squash merge included.
- A batch has at least 2 and at most 8 issues.
- Every issue in a batch is in the PR's repo, has the same parent epic (a
  finite epic or the repo's ad-hoc epic), is not operational, and is not
  directly Blocked-by an open deployment task.
- Commit-per-issue invariant: counting from the base, exactly one commit per
  issue, in the listed order, each subject ending in a space and `(#<N>)` for its issue,
  and no merge commits.
- No silent fallback: every failed check refuses and names the reason.
- vrg-git protects origin, not the local checkout. The push guards and
  rulesets are unchanged.
- The only validation command is `vrg-container-run -- vrg-validate`; every
  "run the tests" step below means that command.
- Commits use `vrg-commit`, and handoff uses `vrg-pr-workflow report-ready`.

## Review focus

1. **A duplicate issue in the list** (`--issue 10 --issue 10`): refused at
   batch start, naming the duplicate. Test in Task 2a.
2. **Commits out of order against the list** (issue 11's commit before issue
   10's): refused, naming the first mismatched commit and the issue it
   should carry. Test in Task 2a.
3. **A commit subject with no issue ref, or someone else's ref:** refused,
   naming the commit. Test in Task 2a.
4. **A merge commit on a batch branch at finalize** (GitHub's update-branch
   button): refused, with the rebase-onto-`develop` fix named. Test in Task 5.
5. **A one-issue run through every changed path** (report-ready, submit,
   finalize, the sweep): byte-for-byte today's behaviour (one linkage line,
   squash). A test in each of Tasks 3, 4, 5 and 6.

---

## Delivery

| Task | Repo | What | Blocked by |
| --- | --- | --- | --- |
| 1a | vergil-tooling | vrg-git: lift local denials on unfrozen feature branches | — |
| 1b | vergil-tooling | vrg-commit `--fixup` | — |
| 2a | vergil-tooling | Batch rules module: hard rules and commit-per-issue | — |
| 2b | vergil-tooling | `vrg-pr-workflow batch-start` / `batch-extract` (marker) | 2a |
| 3 | vergil-tooling | Workflow state and report-ready carry an issue list | 2a, 2b |
| 4 | vergil-tooling | Submit: per-issue checks and one `Closes` line per issue | 2a, 3 |
| 5 | vergil-tooling | Finalize: detect a batch, re-check, rebase-merge | 2a, 6 |
| 6 | vergil-tooling | Epic sweep reads every linkage line | — |
| 7 | vergil-tooling | Worktree convention: the batch form | — |
| 8a | vergil-claude-plugin | issue-implement accepts a list | 1a, 1b, 2b, 3 |
| 8b | vergil-claude-plugin | merge-review in a batch | 1b |
| 8c | vergil-claude-plugin | pr-watch folds fixups on a batch PR | 1a, 1b |
| D1 | vergil-tooling (deployment) | Release and install the tooling and plugin | 1a–8c |
| V1 | vergil-tooling (validation) | Live batch end to end | D1 |
| 9 | vergil-claude-plugin | epic-implement batches and resumes batches | V1 |

Existing bookends: the spec and plan, vergil-project/.github#1020; paad
feedback, vergil-project/.github#1028 (no blocker); documentation review,
vergil-project/vergil-tooling#4198 (blocked by 9); retrospective,
vergil-project/.github#1021 (last).

When the tasks are filed, these advisories are retargeted to the tasks
that fix them. The task's merge review checks each against that task's
tests.

| Task | Advisories |
| --- | --- |
| 1a | GHSA-mrc8-6fmh-mxvw, GHSA-p76r-qxvq-xcr2, GHSA-72jp-cp9w-xrc4 |
| 2a | GHSA-x56h-m46c-crjh |
| 2b | GHSA-f9wp-49wg-ff36 |

---

### Task 1a: vrg-git lifts its local-only denials on unfrozen feature branches

**Files:**

- Modify: `src/vergil_tooling/bin/vrg_git.py` (`_ALLOWED_SIMPLE`, `_DENIED`,
  `_FLAG_DENY`, `_check_denied_flags`, `_run`)
- Test: `tests/vergil_tooling/test_vrg_git.py`

**Interfaces:**

- Produces: `vrg-git reset …`, `vrg-git clean …`, `vrg-git checkout -- .`
  and `vrg-git rebase -i --autosquash <base>`, allowed on a branch that is
  neither protected nor frozen.
- Consumes: `_is_protected_branch()` and `freeze.check_worktree(root,
  action=…)`, both existing.

- [ ] **Step 1: Write the failing tests**
  - `test_reset_allowed_on_feature_branch`: `reset --hard <sha>` on an
    unfrozen `feature/*` branch reaches git.
  - `test_reset_refused_on_protected_branch`: on `develop`, it exits 1 and
    stderr names `protected`.
  - `test_reset_refused_on_frozen_branch`: with a ready
    `.vergil/pr-workflow.json`, it exits 1 with `freeze.check_worktree`'s
    message.
  - The same three tests for `clean -fd`, for `checkout -- .`, and for
    `checkout .` without `--`.
  - `test_rebase_interactive_allowed_only_with_autosquash`:
    `rebase -i --autosquash origin/develop` reaches git with
    `GIT_SEQUENCE_EDITOR=true`; `rebase -i origin/develop` exits 1 naming
    `--autosquash`.
  - `test_rebase_autosquash_refused_on_frozen_branch`.
  - `test_reset_refused_on_unreadable_state`: a malformed
    `.vergil/pr-workflow.json` makes `reset`, `clean`, `checkout -- .` and
    `rebase -i --autosquash` exit 1, naming the read error.
- [ ] **Step 2: Run the tests and see them fail.**
- [ ] **Step 3: Implement.** Move `reset` and `clean` from `_DENIED` into
  `_ALLOWED_SIMPLE`. Add `_check_local_rewrite(subcmd: str, args: list[str]) -> str |
  None`, called from `_run` for `reset`, `clean`, every `checkout` that
  takes a pathspec without switching branch (`checkout .`, `checkout --
  <path>`, `checkout -- .`/`*`), and `rebase -i`. It refuses on `_is_protected_branch()`, and on a frozen
  worktree with `freeze.check_worktree(root, action="rewrite this branch")`.
  It fails closed: when the state file exists but `check.read_error` is set,
  it refuses, naming the read error and the recovery (fix or remove
  `.vergil/pr-workflow.json`, then re-run report-ready). Commit and push
  keep their existing warn-and-proceed. In
  `_check_denied_flags`, allow `-i`/`--interactive` only when `--autosquash`
  is also present, and drop the `checkout` `.`/`*` denial in favour of
  `_check_local_rewrite`. Update `_print_help` and the module docstring to
  say vrg-git protects origin, not the local checkout.
- [ ] **Step 4: Run the tests and see them pass.**
- [ ] **Step 5: Commit** (`vrg-commit --type feat --scope git`).

### Task 1b: vrg-commit `--fixup`

**Files:**

- Modify: `src/vergil_tooling/bin/vrg_commit.py`
- Test: `tests/vergil_tooling/test_vrg_commit.py`

**Interfaces:**

- Produces: `vrg-commit --fixup <commit>`, which makes a `fixup! <subject>`
  commit of `<commit>`. `--type`, `--scope` and `--message` are not required
  with it, and are refused alongside it.

- [ ] **Step 1: Write the failing tests**
  - `test_fixup_creates_fixup_commit`: the subject is `fixup!`, a space, then the
    target's subject.
  - `test_fixup_refuses_commit_not_on_branch`: the target must be reachable
    from HEAD and not from `origin/develop`. Otherwise exit 1, naming the
    commit.
  - `test_fixup_refuses_with_type_or_message`.
  - `test_fixup_refused_on_frozen_branch`, through the existing freeze check.
- [ ] **Step 2: Run the tests and see them fail.**
- [ ] **Step 3: Implement.** Add `--fixup COMMIT` to the parser as
  mutually exclusive with `--type`/`--scope`/`--message`/`--body`. Build the
  commit with `git commit --fixup=<commit>` through the same identity and
  freeze path as a normal commit.
- [ ] **Step 4: Run the tests and see them pass.**
- [ ] **Step 5: Commit** (`vrg-commit --type feat --scope commit`).

### Task 2a: Batch rules module

**Files:**

- Create: `src/vergil_tooling/lib/pr_workflow/batch_rules.py`
- Test: `tests/vergil_tooling/pr_workflow/test_batch_rules.py`

**Interfaces:**

- Produces:
  - `class BatchRuleError(WorkflowError)`, with a message naming the rule and
    the issue or commit.
  - `MAX_BATCH = 8`, `MIN_BATCH = 2`.
  - `check_hard_rules(issues: list[str], *, repo: str) -> None`. `issues` are
    canonical `#N` refs in order. It checks: count 2–8; no duplicate; each in
    `repo`; each a task, not an epic or archive (`epics.non_task_kind`) and
    not a queue item (`epics.queue_item_kind`), the same per-issue refusals
    report-ready and submit run; each opened by an author that
    `vrg-trusted-authors` lists (a leading `app/` stripped), failing closed
    if that command fails; one shared `epics.parent_of` that `epics.is_epic` and whose
    `epics.queue_kind` is `None` (finite or ad-hoc); not
    `epics.is_operational`; no `epics.blockers_of` entry that is open and has
    `epics.operational_kind == "deployment"`.
  - `check_commit_per_issue(issues: list[str], *, base: str, head: str) ->
    None`. It reads `git rev-list --reverse <base>..<head>` and the subjects,
    and checks: no merge commits, `len == len(issues)`, and commit *i*'s
    subject ends with a space and `(#<issues[i] number>)`.
  - `is_batch(issues: list[str]) -> bool`, which is `len(issues) > 1`.

- [ ] **Step 1: Write the failing tests.** GitHub reads are mocked at
  `epics.*`, and git runs on a temporary repo.
  - `test_hard_rules_accept_two_same_epic_tasks`
  - `test_hard_rules_refuse_one_issue`, `…_nine_issues`, `…_duplicate`
    (Review focus 1)
  - `test_hard_rules_refuse_mixed_parents`, `…_queue_parent`,
    `…_unparented`, `…_other_repo`
  - `test_hard_rules_refuse_operational_issue`
  - `test_hard_rules_refuse_non_task_issue` and
    `test_hard_rules_refuse_queue_item`
  - `test_hard_rules_refuse_untrusted_author` and
    `test_hard_rules_refuse_when_trusted_authors_fails`
  - `test_hard_rules_refuse_on_read_failure`
  - `test_hard_rules_refuse_open_deployment_blocker` and
    `test_hard_rules_accept_closed_deployment_blocker`
  - `test_commits_accept_one_per_issue_in_order`
  - `test_commits_refuse_out_of_order` (Review focus 2) and
    `test_commits_refuse_missing_or_foreign_ref` (Review focus 3). Each
    message names the commit's short SHA.
  - `test_commits_refuse_merge_commit`, `…_extra_commit`, `…_fixup_left`
- [ ] **Step 2: Run the tests and see them fail.**
- [ ] **Step 3: Implement** the three functions with the signatures above,
  reusing the `epics` helpers named and `lib.git` for the git reads.
  `check_hard_rules` fails closed: a GitHub read that fails (a parent, a
  label, a blocker, the trusted-author list) refuses, naming what could not
  be read. It never skips a rule.
- [ ] **Step 4: Run the tests and see them pass.**
- [ ] **Step 5: Commit** (`vrg-commit --type feat --scope pr-workflow`).

### Task 2b: The batch marker, through `vrg-pr-workflow`

**Files:**

- Modify: `src/vergil_tooling/bin/vrg_pr_workflow.py` (`parse_args`, new
  `cmd_batch_start`, `cmd_batch_extract`)
- Create: `src/vergil_tooling/lib/pr_workflow/batch_marker.py`
- Test: `tests/vergil_tooling/pr_workflow/test_batch_marker.py`,
  `tests/vergil_tooling/test_vrg_pr_workflow.py`

**Interfaces:**

- Consumes: `batch_rules.check_hard_rules`, and `git.main_worktree_root()`.
- Produces:
  - `marker_path(root: Path, issue: str) -> Path`, which is
    `build/.vergil-state/issue-implement/<N>/batch`.
  - `write_markers(root: Path, issues: list[str], *, base_sha: str) ->
    None`, which writes JSON `{"issues": ["#10", "#11"], "base_sha":
    "<sha>"}` to every member's marker. `base_sha` is `origin/develop`'s
    commit when the batch starts, and is the batch's fixed base from then
    on. It creates each level as a real directory with mode 700.
  - `read_marker(root: Path, issue: str) -> Marker | None`, where
    `Marker` has `issues: list[str]` and `base_sha: str`.
  - `extract(root: Path, issue: str) -> list[str]`, which first refuses,
    naming each file and the record step, while any
    `issue-implement/<N>/merge-review/security-*.md` exists. Otherwise it
    removes the issue's whole `issue-implement/<N>/` directory, rewrites each remaining
    member's list without it, and returns the remaining list. When fewer
    than 2 members remain, it removes the last member's marker too, so a
    marker always means a batch of 2 or more.
  - `remove_markers(root: Path, issues: list[str]) -> None`.
  - CLI: `vrg-pr-workflow batch-start --issue N --issue M …` runs
    `check_hard_rules` and then `write_markers`, refusing before writing
    anything. If any member already has a marker, it succeeds without
    writing only when every member's marker holds exactly this list (a
    resume), and refuses otherwise, naming the conflicting issue. `vrg-pr-workflow batch-extract --issue N` runs `extract`.
    `vrg-pr-workflow batch-show --issue N` prints the marker's list and
    `base_sha`, or exits 1 when there is none. All three also take a fully qualified
    `--issue owner/repo#N`, and exit 2 when the current clone is not that
    repo's, so "wrong clone" is never read as "no marker".

- [ ] **Step 1: Write the failing tests**
  - `test_batch_start_writes_marker_for_every_member`, with the same
    `base_sha` in each.
  - `test_batch_start_refuses_and_writes_nothing_on_rule_break`
  - `test_batch_start_accepts_identical_existing_batch`: a resume, so
    nothing changes.
  - `test_batch_start_refuses_different_list_over_existing_markers`
  - `test_extract_removes_whole_state_dir_and_rewrites_lists`, including a
    `merge-review/` subdirectory present.
  - `test_extract_refuses_while_a_security_details_file_exists`: the file
    survives, and the message names the record step.
  - `test_extract_to_one_member_removes_last_marker`
  - `test_batch_show_exits_1_without_marker`
  - `test_batch_show_exits_2_in_another_repos_clone`
- [ ] **Step 2: Run the tests and see them fail.**
- [ ] **Step 3: Implement** the module and the three subcommands.
- [ ] **Step 4: Run the tests and see them pass.**
- [ ] **Step 5: Commit** (`vrg-commit --type feat --scope pr-workflow`).

### Task 3: Workflow state and report-ready carry an issue list

**Files:**

- Modify: `src/vergil_tooling/lib/pr_workflow/state.py`,
  `src/vergil_tooling/lib/pr_workflow/engine.py`,
  `src/vergil_tooling/lib/pr_workflow/submission.py` (`read_pr_fields`),
  `src/vergil_tooling/bin/vrg_pr_workflow.py` (`parse_args`,
  `cmd_report_ready`)
- Test: `tests/vergil_tooling/pr_workflow/test_state.py`,
  `tests/vergil_tooling/test_vrg_pr_workflow.py`

**Interfaces:**

- Produces:
  - `SCHEMA_VERSION` stays `2`. `WorkflowState` gains an optional `issues:
    list[str] | None`, ordered, written to the JSON only for a batch, and a
    property `all_issues -> list[str]` that returns `issues` or `[issue]`.
    `WorkflowState.issue` stays the first issue. A single-issue record is
    byte-for-byte unchanged, and existing records read as they do today.
  - `engine.init_state(*, issues: list[str], …)`.
  - `read_pr_fields(...)` also returns `"issues"`, the comma-joined
    `all_issues`, so it is always present (one issue for a single-issue
    record).
  - `report-ready --issue` repeats (`action="append"`).
- Consumes: `batch_rules.check_hard_rules`, `check_commit_per_issue`,
  `is_batch`, and `batch_marker.remove_markers` (Task 2b).

- [ ] **Step 1: Write the failing tests**
  - `test_state_round_trips_issue_list`
  - `test_state_single_issue_json_unchanged`: no `issues` key is written.
  - `test_state_without_issues_reads_as_single`: `all_issues == [issue]`.
  - `test_report_ready_single_issue_unchanged` (Review focus 5)
  - `test_report_ready_batch_records_ordered_issues`
  - `test_report_ready_batch_refuses_rule_break` and
    `test_report_ready_batch_refuses_commit_mismatch`. Each records nothing
    and pushes nothing.
  - `test_report_ready_runs_existing_per_issue_refusals_on_every_issue`
    (cross-repo, non-task, operational, queue).
  - `test_report_ready_rerun_refuses_a_different_issue_list`: a batch
    recorded for `#10, #11` and re-run with `--issue 10` exits non-zero,
    naming both lists, and records nothing.
  - `test_report_ready_batch_removes_markers_on_success`,
    `test_report_ready_batch_removes_markers_when_relay_push_warns`, and
    `test_report_ready_batch_keeps_markers_on_refusal`.
- [ ] **Step 2: Run the tests and see them fail.**
- [ ] **Step 3: Implement.** In `cmd_report_ready`, normalise each `--issue`
  and run the four `_reject_*` checks on each. Replace the re-run guard's
  `str(args.issue) != state.issue` with a comparison of the whole ordered
  list against the recorded `all_issues`, refusing and naming both lists
  when they differ. When `is_batch`, run
  `check_hard_rules` and `check_commit_per_issue(base=state.base,
  head="HEAD")` before any push. Record `issues` in the state and the relay
  record. Once the local state write succeeds (report-ready's point of no
  return, whatever the relay push does), call
  `batch_marker.remove_markers(git.main_worktree_root(), issues)`.
- [ ] **Step 4: Run the tests and see them pass.**
- [ ] **Step 5: Commit** (`vrg-commit --type feat --scope pr-workflow`).

### Task 4: Submit runs per-issue checks and writes one `Closes` line per issue

**Files:**

- Modify: `src/vergil_tooling/lib/pr_body.py` (`build_pr_body`),
  `src/vergil_tooling/bin/vrg_submit_pr.py` (the three body build sites:
  `_run_cli_mode`, `_open_pr`, the template mode; and
  `_metadata_from_state`, the relay and named-branch path's metadata)
- Test: `tests/vergil_tooling/test_pr_body.py`,
  `tests/vergil_tooling/test_vrg_submit_pr.py`

**Interfaces:**

- Produces: `build_pr_body(*, summary: str, links: list[tuple[str, str]],
  notes: str) -> str`, where `links` is `[(linkage, issue_ref), …]`, one `-
  <linkage> <ref>` line each under `## Issue Linkage`. A single-element list
  renders exactly today's body.
- Consumes: the state's `issues` (Task 3), `batch_rules`.

- [ ] **Step 1: Write the failing tests**
  - `test_build_pr_body_single_link_unchanged` (Review focus 5)
  - `test_build_pr_body_one_line_per_issue`
  - `test_submit_batch_closes_every_issue`: every issue is a managed task,
    so `Closes` each.
  - `test_submit_batch_rechecks_rules_and_commits_before_create`: on a
    mismatch, no `gh pr create` is called.
  - `test_submit_cli_mode_stays_single_issue`
  - `test_relay_submit_batch_closes_every_issue`: the named-branch path
    (`_relay_open`) writes one `Closes` line per issue.
  - `test_relay_submit_batch_rechecks_after_fetch`: on the relay path, the
    branch and base are fetched before the commit check, and a mismatch
    opens no PR.
  - `test_open_pr_requires_issues`: metadata without `issues` raises.
  - `test_serial_batch_checks_the_rebased_head`: after the `--select`
    rebase, the check runs on the head the PR opens at, and a commit that
    went empty in the rebase is refused.
- [ ] **Step 2: Run the tests and see them fail.**
- [ ] **Step 3: Implement.** Add `_checked_links(issue_refs: list[str],
  requested: str) -> list[tuple[str, str]]`, which runs the four
  `_reject_if_*` checks and `_resolve_linkage` per issue. Use it at the
  template and relay build sites. `_metadata_from_state` returns `issues`
  from `state.all_issues`, as `read_pr_fields` does, and `_open_pr` reads
  `metadata["issues"]` with no fallback to `issue`. On the relay path, run
  `vrg-git fetch origin <branch> <target>` before the commit check. For a batch, call `check_hard_rules` and
  `check_commit_per_issue` before `_create_pr`, on the head the PR opens at
  (`expected_head` in `_open_pr` and `_push_create_record`, which is the
  rebased head in the serial batch mode), against `origin/<target>` after a
  fetch of the target. The CLI emergency mode keeps one `--issue`.
- [ ] **Step 4: Run the tests and see them pass.**
- [ ] **Step 5: Commit** (`vrg-commit --type feat --scope submit-pr`).

### Task 5: Finalize detects a batch PR, re-checks it and rebase-merges

**Files:**

- Modify: `src/vergil_tooling/bin/vrg_finalize_pr.py` (`_resolve_strategy`,
  `_stage_merge`, `_conflict_resolution_msg`), `src/vergil_tooling/lib/pr_merge.py`
  (`wait_and_merge`, `_update_or_abort`), `src/vergil_tooling/lib/github.py`
  (new `update_branch_rebase`; `merge` gains `match_head`)
- Test: `tests/vergil_tooling/test_vrg_finalize_pr.py`,
  `tests/vergil_tooling/test_pr_merge.py`

**Interfaces:**

- Consumes: `github.pr_closing_references` (existing),
  `linkage.extract_tracking_refs` (Task 6),
  `batch_rules.check_commit_per_issue`.
- Produces: `_resolve_strategy(branch: str, override: str | None, *,
  batch: bool) -> str`. `batch` gives `"rebase"`, and a `squash` override on
  a batch exits non-zero naming the anti-goal.
- Produces: `github.update_branch_rebase(pr: str, *, expected_head: str) ->
  None`, which calls GraphQL `updatePullRequestBranch` with `updateMethod:
  REBASE` and `expectedHeadOid`. Also `pr_merge.wait_and_merge(pr, *,
  strategy, batch_check: Callable[[], str] | None = None)`. `batch_check`
  fetches the PR head every time it runs, checks it, and returns the SHA it
  checked. With `batch_check`, a `BEHIND` PR is updated with
  `update_branch_rebase`, never `update_branch`, and `batch_check` runs
  after every update and immediately before the merge call. The merge then
  runs `gh pr merge --rebase --match-head-commit <that sha>`
  (`github.merge` gains `match_head: str | None`). A head mismatch counts as
  a re-check, within the existing update cap.

- [ ] **Step 0: Confirm the API.** Check GitHub's GraphQL docs that
  `updatePullRequestBranch` accepts `updateMethod: REBASE`. If it does not,
  stop and ask the human. The fallback is to refuse a `BEHIND` batch PR and
  print the rebase and force-with-lease commands.
- [ ] **Step 1: Write the failing tests**
  - `test_behind_batch_pr_updates_by_rebase_not_merge`: `update_branch` is
    never called, and `update_branch_rebase` is.
  - `test_batch_check_runs_after_each_update_and_before_merge`
  - `test_batch_merge_pins_the_checked_head`: the merge call carries
    `--match-head-commit <checked sha>`, and a head moved after the check
    is re-checked, never merged unchecked.
  - `test_behind_single_issue_pr_still_uses_update_branch` (Review focus 5)
  - `test_single_issue_pr_still_squashes` (Review focus 5)
  - `test_batch_pr_rebase_merges`: the body has 2 `Closes` lines.
  - `test_pr_with_one_close_and_one_ref_is_not_a_batch`
  - `test_single_close_pr_with_manual_development_link_still_squashes`
  - `test_batch_pr_refuses_squash_override`
  - `test_rebase_merged_batch_records_the_tip_for_the_cd_check`: the SHA
    `_stage_merge` records is the new tip of `develop`, which the CD run is
    keyed on.
  - `test_batch_pr_refuses_commit_mismatch_before_merge`, with no merge call.
  - `test_batch_pr_refuses_merge_commit_naming_rebase` (Review focus 4)
  - `test_batch_conflict_message_says_rebase_not_merge`
- [ ] **Step 2: Run the tests and see them fail.**
- [ ] **Step 3: Implement.** In `_stage_merge`, decide "batch" from the PR
  body (`github.read_json("pr", "view", pr, "--json", "body")`): a batch has
  more than one `Closes` line, read with `linkage.extract_tracking_refs`
  (Task 6). Those lines, in body order, are the list for the commit check.
  Only for a batch, cross-check them against
  `github.pr_closing_references(pr)`, and refuse, naming both, when a
  `Closes` issue is missing from it. Manual Development links never make a
  PR a batch. For a batch, run
  `check_commit_per_issue(issues, base=origin/<base>, head=origin/<head>)`
  after fetching. Pass it to `wait_and_merge` as `batch_check`, with
  `strategy="rebase"`. `_conflict_resolution_msg` takes
  `batch: bool` and, for a batch, prints `git rebase origin/<base>` and
  `git push --force-with-lease origin <head>` in place of the merge steps.
- [ ] **Step 4: Run the tests and see them pass.**
- [ ] **Step 5: Commit** (`vrg-commit --type feat --scope finalize-pr`).

### Task 6: The epic sweep reads every linkage line

**Files:**

- Modify: `src/vergil_tooling/lib/linkage.py`,
  `src/vergil_tooling/lib/epic_audit.py` (`task_drift`)
- Test: `tests/vergil_tooling/test_linkage.py`,
  `tests/vergil_tooling/test_epic_audit.py`

**Interfaces:**

- Produces: `extract_tracking_refs(text: str) -> list[tuple[str, str]]`,
  every linkage as `(keyword, ref)`, with `keyword` either `"Ref"` or
  `"Closes"` and `ref` either `"#N"` or `"owner/repo#N"`, in body order. `extract_tracking_ref` keeps
  its single-linkage contract for its other callers.

- [ ] **Step 1: Write the failing tests**
  - `test_extract_tracking_refs_returns_all_in_order`
  - `test_task_drift_checks_every_issue_of_a_batch_pr`
  - `test_task_drift_ignores_ref_lines_in_a_multi_linkage_body`
  - `test_task_drift_single_issue_unchanged` (Review focus 5)
- [ ] **Step 2: Run the tests and see them fail.**
- [ ] **Step 3: Implement.** In `task_drift`, drop the `except ValueError:
  continue` skip. A body with one linkage line is handled as today. In a
  body with more than one, act on each `Closes` line, and never on a `Ref`
  line. `extract_tracking_refs` returns `(keyword, ref)` pairs so the caller
  can tell them apart.
- [ ] **Step 4: Run the tests and see them pass.**
- [ ] **Step 5: Commit** (`vrg-commit --type fix --scope epic-audit`).

### Task 7: Worktree convention, the batch form

**Files:**

- Modify: `docs/specs/worktree-convention.md`

- [ ] **Step 1:** Add the batch rule: one worktree and one branch per batch,
  `.worktrees/issue-<N>-batch-<slug>/` on `feature/<N>-batch-<slug>`, named
  for its first issue.
- [ ] **Step 2:** Add a batch form of the Agent prompt contract that names
  every issue in order, and the batch marker (`vrg-pr-workflow
  batch-start`).
- [ ] **Step 3:** Validate, then commit (`vrg-commit --type docs --scope
  worktrees`).

### Task 8a: issue-implement accepts a list

**Files:**

- Modify: `skills/issue-implement/SKILL.md` (Preflight, Implement, Merge
  review, Record the PR metadata and hand off)

- [ ] **Step 1:** Add a "Batch mode" section. It opens with the spec §5.1
  guidelines: what to batch, what never to batch, the cap of 4 (up to 8
  when every item is tiny), and the Order rule, which applies to a list a
  human hands over directly. Then the steps:
  1. Run Preflight steps 2–5 on **every** listed issue, the trusted-author
     check included, and start nothing if any issue fails. Then run
     `vrg-pr-workflow batch-start --issue …`, and stop on refusal.
  2. Use one batch worktree, created at the marker's `base_sha`. **No
     rebase onto `develop` mid-batch:** in batch mode, the "Resolving
     conflicts with the base branch" section does not apply until handoff.
     A conflict with a moved `develop` is resolved at submit or by pr-watch,
     never by rewriting finished issues. The first issue's fold base and
     merge-review base is `base_sha`. **Resume:** when the batch already exists
     (`batch-start` accepted an identical list, or the batch worktree is
     present), enter the existing batch worktree and continue at the first
     issue that is not finished. An issue is finished only when its merge
     review has closed: its closing merge-review comment, with the
     eligibility line, is on the issue from a trusted author, and its commit
     is folded. The commit's shape alone never counts. Resume that issue
     from the state its `merge-review/` directory holds, or from the start
     of its review if there is none. Make
     Preflight step 6's merge-review resume batch-aware: for an issue with a
     batch marker, it resumes the batch in the batch worktree, never the
     single issue.
  3. For each issue: first commit with its ref, later work as `vrg-commit
     --fixup <first>`, validate, merge-review (Task 8b), then fold with
     `vrg-git rebase -i --autosquash <previous issue's commit>`.
  4. Stop at an issue boundary on the §5.4 conditions. A batch stop is a
     **non-review item** (review doctrine §1.7): list it under `Awaiting a
     ruling`, naming the item, the reason and the choices (extract,
     continue, hand off what is done, unbatch), and end with the existing
     `waiting: non-review items`. The ruling arrives the way non-review
     rulings do: from the human in the session, or in epic-implement's
     dispatch prompt.
  5. On a human extraction: `vrg-git reset --hard <prev>`, `vrg-git clean
     -fd`, `vrg-pr-workflow batch-extract --issue <N>`, the approved
     comment, and a `Blocked-by:` link for a collision.
  6. A batch left with one issue hands off as a single issue.
  7. Restate the Merge review section's rule "Every `report-ready` follows a
     merge review that ended `eligible` or `not-eligible` at the `HEAD` it
     reports" for a batch: each issue's merge review ends at that issue's
     unfolded head, and the fold that follows only combines that issue's
     own commits, so no review is re-run after a fold.
  8. Hand off with `report-ready` and every `--issue`, in order. The notes
     carry each issue's merge-review eligibility line, prefixed with its
     ref. report-ready removes the markers (Task 3), and the skill deletes
     each issue's `merge-review/` as it does today.
- [ ] **Step 2:** Validate, then commit (`vrg-commit --type feat --scope
  issue-implement`).

### Task 8b: merge-review in a batch

**Files:**

- Modify: `skills/merge-review/SKILL.md` (the review request, the fix step,
  the check request)

- [ ] **Step 1:** In a batch, both `--base` uses take the previous issue's
  commit, or for the first issue the marker's `base_sha` (never a moving
  `origin/develop`), and the fix is `vrg-commit --fixup <the issue's first
  commit>`.
- [ ] **Step 2:** Validate, then commit (`vrg-commit --type feat --scope
  merge-review`).

### Task 8c: pr-watch folds fixups on a batch PR

**Files:**

- Modify: `skills/pr-watch/SKILL.md` (Monitor & reconcile)

- [ ] **Step 1:** On a PR that closes more than one issue, a CI or review fix
  is `vrg-commit --fixup <the owning issue's commit>`, then `vrg-git rebase
  -i --autosquash origin/<base>`, then `vrg-git push --force-with-lease`. A
  fold that conflicts, or a fix that belongs to no single issue, stops for
  the human. Single-issue PRs are unchanged.
- [ ] **Step 2:** Validate, then commit (`vrg-commit --type feat --scope
  pr-watch`).

### Task D1 (deployment): Release and install the tooling and plugin

Kind `deployment`, filed in vergil-tooling and blocked by 1a–8c. Its
precondition is human-attested: vergil-tooling and vergil-claude-plugin are
released with these changes. Procedure: `vrg-host` and `vrg-vm update --all`
install them. Acceptance: `vrg-pr-workflow batch-start --help` and
`vrg-commit --fixup --help` work on the host and a VM.

### Task V1 (validation): A live batch end to end

Kind `validation`, filed in vergil-tooling and blocked by D1. Procedure: pick
2–4 small ready vergil-tooling issues under one epic, and run one
`issue-implement` call with the list through report-ready, then the human's
`vrg-submit-pr` and `vrg-finalize-pr`. Acceptance (spec §8):

- one PR, with one PR CI run and one post-merge CD run;
- one commit per issue on `develop`, each with its own changelog entry under
  its own type;
- every issue closed;
- a single-issue PR in the same period still squash-merges.

### Task 9: epic-implement batches and resumes batches

**Files:**

- Modify: `skills/epic-implement/SKILL.md` (§3 Work the runnable frontier,
  §4 Batch at the gate)

- [ ] **Step 1:** In §3, epic-implement may batch independent ready tasks
  under the spec §5.1 guidelines. It names each batch, with its issues in
  order, in the plan it already shows; there is no separate approval step.
  It dispatches a batch with the batch form of the Agent prompt contract
  (Task 7). After a ruling on a batch item, it re-dispatches the list that
  `batch-show` prints for the first remaining member, never a list it holds
  in memory (an extraction may have shortened it).
  Before **every** code-task dispatch, not only when it rebuilds state in a
  new session, it runs `vrg-pr-workflow batch-show --issue <owner/repo#N>` in
  the task repo's main clone (as its merge-review lookup does). A task with a
  marker is re-dispatched as its whole batch, never alone.
- [ ] **Step 2:** In §4, present a batch stop with the run's other
  non-review items. The human extracts the item, continues, hands off what
  is done, or unbatches (extraction for every item), and asks why the issues
  were batched, to refine the guidelines. Relay the ruling in the dispatch
  prompt of one new run of the same list, as §1.7 already does for
  non-review rulings.
- [ ] **Step 3:** Validate, then commit (`vrg-commit --type feat --scope
  epic-implement`).
