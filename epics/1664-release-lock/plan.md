# release-lock: a release lock for `develop` — implementation plan

- **Epic:** vergil-project/.github#1664
- **Planning issue:** vergil-project/.github#1662 (review history is in its
  comments)
- **Status:** draft

> **For agentic workers:** each implementation task is one PR, run with
> `issue-implement`; operational tasks are run with `issue-deploy` or
> `issue-validate`. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** a release runs against a quiescent `develop`: once it cuts, no PR
merges into `develop` except the release's own, agents keep submitting and
running CI throughout, and nothing (a crash, a lock break or a cleanup sweep)
strands a PR unarmed or removes the release's worktree.

**Architecture:** three steps, in order. Phase 0 fixes the incident in
vergil-project/vergil-tooling#5187 by itself: cleanup never removes a
`release/*` worktree while a release tracking issue is open. Phase 1 moves
every tool that gets a PR into `develop` onto one PR path,
`lib/pr_path.py`, which judges, arms and (for a clean-status PR) merges, and
reduces `vrg-finalize-pr` to watching and cleaning up. Phase 2 builds the
release lock on that path: a ref, `refs/vergil/release-lock`, taken by
create-only push and updated by compare-and-swap; an arming-ref handshake in
the path; a disarm sweep before the cut and a re-arm sweep at the end; stale
reclaim, a human break and the merge watch's waiting.

**Tech Stack:** Python 3 (vergil-tooling, pytest), git refs pushed through
the relay transport, GitHub GraphQL auto-merge mutations, Markdown skills
(vergil-claude-plugin), Zensical site docs.

**Spec:** [`spec.md`](spec.md) in this directory. Section references (§5.4)
are to it.

## Global Constraints

- All lock state lives in refs: `refs/vergil/release-lock`,
  `refs/vergil/release-lock-history`,
  `refs/vergil/release-lock-arming/<pr>` (§5.2, §5.4). Never an issue or a
  repo variable.
- Taking a ref is a create-only push, `--force-with-lease=<ref>:`; updating
  or deleting one is a compare-and-swap push,
  `--force-with-lease=<ref>:<expected-sha>` (§3.5, §5.2).
- Lock and arming refs are read with **no** trusted-author check (§5.10). A
  lock that cannot be read or parsed is **held**, never absent (§5.2, §6).
- Timing constants, all in `lib/release_lock/model.py`, and nowhere else:
  `HEARTBEAT_INTERVAL_SECS = 60`, `STALE_AFTER_SECS = 900`,
  `ARMING_EXPIRY_SECS = 300`, `ARM_CALL_BOUND_SECS = 60` (well below the
  expiry, §5.4), `ARMING_WAIT_BOUND_SECS = 900`,
  `BROKEN_WATCH_POLL_SECS = 300`. The normal watch poll stays what it is
  today. (§7 leaves these to the builder; the plan fixes them so every task
  uses the same values.)
- Merge methods: `MERGE` for a same-repo `release/post-*` head with a
  trusted author and for a trusted-author PR labelled `dependency-update`;
  `REBASE` for a batch PR (trusted linkage, as submit does today); `SQUASH`
  otherwise. Worked out from the PR, never from the caller (§5.4).
- The one PR path accepts only a same-repo, non-draft PR into `develop`
  whose branch has a trusted author (`finalize_watch.confirmed_head`'s rule)
  and which passes the provenance check (§5.1).
- Only the disarm sweep turns auto-merge off (§5.1). No other code does.
- After Task 9 no module outside `lib/pr_path.py` calls
  `enablePullRequestAutoMerge`, `gh pr merge` or a wait-and-merge against
  `develop`. The release PR into `main` (`lib/release/merge.py`, `--admin`,
  pinned to the head) is the one exception (§5.4).
- `vrg-release` and `vrg-release-lock break` are human only, fail-closed
  against every agent identity and agent signal, as `bin/vrg_release.py` is
  today. The agent identity may read the lock, arm any PR the path accepts
  and reclaim a stale lock; it never takes or breaks one (§6).
- No silent failures: a failed disarm, re-arm, ref write or issue read stops
  the step with an error naming what failed (§6).
- Every vergil-tooling task ends with `vrg-container-run -- vrg-validate`
  green and a commit with `vrg-commit`; every new or changed behavior is
  test-first (write the test, see it fail, implement, see it pass).
- Releases (vergil-tooling and vergil-claude-plugin) are human actions; a
  deployment task attests them as preconditions and never cuts one.
  `vrg-host upgrade`, `vrg-host plugins` and `vrg-vm update --all` are
  human-run steps.

## Review Focus

1. **Two releases at once.** Two humans (or one human in two shells) start
   `vrg-release` together. Expected: exactly one create-only push wins; the
   other refuses, naming the holder, and has disarmed nothing (Task 23,
   `test_second_release_refused_before_disarm`).
2. **A PR going green during the quiesce.** A PR's checks pass between the
   lock and the disarm sweep. Expected: it either merged before the cut and
   is in the release, or is disarmed and merges at the unlock; never merged
   after the cut (Task 19, `test_arm_racing_lock_never_arms_under_lock`;
   Task 23, `test_merge_between_gates_and_disarm_rejudges_new_head`).
3. **An unreadable or corrupt lock.** The lock ref points at a commit whose
   file is not valid JSON. Expected: everything treats it as held and
   reports it; nothing deletes or overwrites it except `break`
   (Task 17, `test_unparseable_lock_reads_as_held`; Task 22,
   `test_reclaim_refuses_unparseable_lock`).
4. **A laptop that sleeps mid-release.** The heartbeat goes stale while the
   tracking issue is open. Expected: no watch reclaims it (an open tracking
   issue keeps it the human's); `--resume` after wake finds its own lock and
   goes on (Task 18, `test_stale_heartbeat_open_issue_not_stale`; Task 21,
   `test_own_write_cas_failure_retried`).
5. **An agent's watch past its deadline during a long release.** Expected:
   the watch ends with the `held by release` outcome and its own exit code,
   not a timeout or `failed: auto-merge off`, and names the release, version,
   stage and health (Task 26, `test_deadline_while_held_exits_held`).

---

## Delivery

Every task is filed under the epic, vergil-project/.github#1664, in the repo
its PR lands in. The plan task ID is the `Plan task:` key `file-tasks`
writes.

| Task | Repo | Kind | What | Blocked by |
|---|---|---|---|---|
| 1 | `.github` | task | Spec and plan (vergil-project/.github#1665) | — |
| **Phase 0** | | | | |
| 2 | `vergil-tooling` | task | Cleanup never removes a `release/*` worktree while a release tracking issue is open; every removal logged; "release worktree vanished" error | — |
| 3 | `vergil-tooling` | deployment | A vergil-tooling release carries Task 2, installed on the host and VMs | 2 |
| **Phase 1** | | | | |
| 4 | `vergil-tooling` | task | Provenance relaxation for arming and the path's clean-status merge; audit actor check follows it | — |
| 5 | `vergil-tooling` | task | The one PR path (`lib/pr_path.py`), the `MERGE` method; `vrg-submit-pr` arms through it | 4 |
| 6 | `vergil-tooling` | task | Push mode: `vrg-submit-pr --push` judges a head before pushing it to an open PR's branch | 5 |
| 7 | `vergil-tooling` | task | Mechanized automation (`mechanized_pr`) opens and arms through the path | 5 |
| 8 | `vergil-tooling` | task | `vrg-update-deps` and the release's `update-deps` stage arm through the path; their direct merge retired | 5 |
| 9 | `vergil-tooling` | task | The release's back-merge arms through the path; `wait_and_merge` kept for `main` only; a guard test for the one path | 7, 8 |
| 10 | `vergil-tooling` | task | `vrg-finalize-pr` reduced: no auto-merge turn-off, no provenance or gate re-checks | 6 |
| 11 | `vergil-tooling` | deployment | A vergil-tooling release carries Tasks 4–10, installed on the host and VMs | 4, 5, 6, 7, 8, 9, 10 |
| 12 | `vergil-claude-plugin` | task | The pr-watch skill pushes its fixes with `vrg-submit-pr --push` | 11 |
| 13 | `vergil-claude-plugin` | deployment | A plugin release carries Task 12, installed on the host and VMs | 12 |
| 14 | `vergil-tooling` | task | `vrg-git` refuses a plain push to an open develop PR's branch, naming `vrg-submit-pr --push` | 13 |
| 15 | `vergil-tooling` | deployment | A vergil-tooling release carries Task 14, installed on the host and VMs | 14 |
| 16 | `vergil-tooling` | validation | Phase 1 exit: every caller goes through the one path, live | 15 |
| **Phase 2** | | | | |
| 17 | `vergil-tooling` | task | Lock refs: model, create-only and compare-and-swap writes, history, arming refs | 16 |
| 18 | `vergil-tooling` | task | Staleness; `vrg-release-lock status`; the lock in `vrg-repo-status` | 17 |
| 19 | `vergil-tooling` | task | The arming gate in the path, the holder's pass, `held for release` in submit and the mechanized jobs | 17 |
| 20 | `vergil-tooling` | task | The disarm and re-arm sweeps | 19 |
| 21 | `vergil-tooling` | task | The lock owner: serialised writes and the background heartbeat | 17 |
| 22 | `vergil-tooling` | task | Reclaim and `vrg-release-lock break` | 18, 20 |
| 23 | `vergil-tooling` | task | `vrg-release`: pre-check, lock, quiesce and cut; the early-exit unlock path | 21, 22 |
| 24 | `vergil-tooling` | task | `vrg-release`: the `unlock` and `rearm` stages, the stray-merge checks and the abandon step | 23 |
| 25 | `vergil-tooling` | task | `vrg-release`: `--resume` and `--supersede` under the lock | 24 |
| 26 | `vergil-tooling` | task | The merge watch under the lock (`vrg-finalize-pr`, `vrg-pr-await`) | 19, 22 |
| 27 | `vergil-tooling` | task | Cleanup safety moves onto the lock | 18 |
| 28 | `vergil-tooling` | task | The backstop audit: invariant breaches, drafts, stale locks, outsider PRs | 18 |
| 29 | `vergil-tooling` | deployment | A vergil-tooling release carries Tasks 17–28, installed on the host and VMs | 17, 18, 19, 20, 21, 22, 23, 24, 25, 26, 27, 28 |
| 30 | `vergil-claude-plugin` | task | Skills re-run the watch on its `held by release` exit | 29 |
| 31 | `vergil-claude-plugin` | deployment | A plugin release carries Task 30, installed on the host and VMs | 30 |
| 32 | `vergil-tooling` | validation | A live vergil-tooling release under the lock, with an agent PR in flight | 29, 31 |
| 33 | fixtures | fixture runs | One run on each of the six fixtures against the Task 29 release | 29 |
| doc-review | `vergil-tooling` | task | Documentation review: `Documentation review: release-lock` | 32, 33 |
| retrospective | `.github` | retrospective | `Retrospective: release-lock` | all others |

### Phase 1 exit criteria

Phase 2 starts only when all of these hold; Task 16 records them, and Task
17, the root of phase 2, is blocked by it:

1. Every caller §5.4 names (submit, pr-watch fixes, `vrg-finalize-pr`,
   `mechanized_pr`, standalone `vrg-update-deps`, the release's
   `update-deps` stage and back-merge) arms or merges a PR into `develop`
   only through `lib/pr_path.py`. (`vrg-pr-await` and the re-arm sweeps join
   in phase 2, Tasks 20 and 26.)
2. The guard test of Task 9 passes: no module outside `lib/pr_path.py`
   calls `enablePullRequestAutoMerge`, `gh pr merge` or a wait-and-merge
   against `develop`.
3. `vrg-finalize-pr` never turns auto-merge off (Task 10).
4. The pr-watch skill pushes with `vrg-submit-pr --push` and `vrg-git`
   refuses a plain push to an open develop PR's branch (Tasks 12, 14).
5. All of it is released and installed, and Task 16 has run green.

### Operational tasks

- **Task 3** ships the incident fix ahead of everything, since the human
  must otherwise wait out agents by hand before every release. Nothing
  downstream needs it deployed; its closure is the signal that #5187 cannot
  recur.
- **Tasks 11, 13 and 15** sequence a cross-repo cut-over: the plugin's
  pr-watch cannot use `--push` until a release carries it (11), and `vrg-git`
  must not refuse a plain push until the plugin that stops making one is
  installed (13). Each deployment's procedure starts with
  `vrg-release-record check <owner>/<repo>#<N>` for each task it ships.
- **Task 29** ships phase 2; **Task 31** ships the skill rule that depends
  on its exit code.
- **Task 33** is a fixture run (`fixture-run`): the lock changes
  `vrg-release` and the release's language-specific `update-deps` stage,
  and the one PR path changes every submit, so every language and both
  shapes are affected and all six fixtures are seeded:

  ```bash
  fixture-run vergil-project/.github#1664 \
    python-lib,python-app,rust-lib,rust-app,ts-lib,ts-app \
    --validates "the release lock and the one PR path, vergil-tooling as released by Task 29" \
    --after <Task 29 ref>
  ```

  Each run's three tasks are keyed `Plan task: 33 <fixture>`.

### Notes for `file-tasks`

- **#5187.** vergil-project/vergil-tooling#5187 is the intake report Task 2
  fixes. Once Task 2 is filed, close #5187 as completed with the comment
  `Tracked as <Task 2 ref> under vergil-project/.github#1664.` Task 2's body
  refs it.
- **Advisories.** Six advisories track the planning issue. Before
  `Tasks filed: yes`, retarget each from vergil-project/.github#1662 to the
  task or tasks that fix it (`vrg-security-record retarget … --from … --to
  …`, one after another), choosing the tasks from each advisory's own notes.
  All fix tasks are in `vergil-project`, so no advisory moves.
- **The follow-up intakes** raised in the review rounds
  (vergil-project/.github#1674, vergil-project/.github#1681) are not tasks of
  this epic.

---

## Phase 0

### Task 2: Cleanup safety keyed on an open release tracking issue (vergil-tooling)

Spec §5.8, phase 0 of §5.

**Files:**

- Modify: `src/vergil_tooling/bin/vrg_finalize_pr.py` (the cleanup around
  `_CLEARABLE_STATES`, lines ~955–1060)
- Modify: `src/vergil_tooling/lib/worktrees.py` (add the protection
  predicate)
- Modify: `src/vergil_tooling/lib/release/preflight.py`
  (`_trusted_release_issues` is reused; no behavior change)
- Modify: `src/vergil_tooling/lib/release/orchestrator.py` (the vanished
  check before each stage up to `teardown-worktree`)
- Modify: `src/vergil_tooling/lib/release/finalize.py`,
  `src/vergil_tooling/lib/release/supersede.py` (log their sanctioned
  removals)
- Test: `tests/vergil_tooling/test_vrg_finalize_pr.py`,
  `tests/vergil_tooling/test_worktrees.py`,
  `tests/vergil_tooling/test_release_orchestrator.py`

**Interfaces:**

- Produces: `worktrees.release_protection(repo: str) -> ReleaseProtection`,
  a frozen dataclass `ReleaseProtection(active: bool, reason: str)`; `active`
  while any open, trusted release tracking issue exists. A failed read
  raises `worktrees.ProtectionReadError`, which cleanup treats as protected.
  Task 27 replaces its key with the lock.
- Produces: `worktrees.log_removal(path: Path, branch: str, reason: str) ->
  None`, one line on stderr and one in the activity log
  (`lib/activity_log.py`): `removed worktree <path> (branch <branch>):
  <reason>`.
- Produces: `release.orchestrator.ReleaseWorktreeVanished(ReleaseError)`.

- [ ] **Step 1: Write the failing tests**
  - `test_cleanup_skips_release_worktree_while_release_open`: a merged
    `release/2.1.9` worktree, `release_protection` active → not removed;
    the output says `kept .worktrees/release-2.1.9: a release is running`.
  - `test_cleanup_removes_release_worktree_when_no_release_open`: same
    worktree, protection inactive → removed and logged.
  - `test_cleanup_treats_failed_protection_read_as_protected`: the read
    raises → no `release/*` worktree removed, the error is printed, and the
    exit is non-zero.
  - `test_every_removal_is_logged`: a merged feature worktree's removal
    writes the `removed worktree …` line with the reason `PR merged`.
  - `test_release_fails_when_worktree_vanished_before_teardown`: the
    worktree path is missing before `confirm-main` → `ReleaseWorktreeVanished`
    naming the path.
  - `test_no_vanished_check_after_teardown`: `ctx.worktree_path is None`
    after `teardown-worktree` → no error.
- [ ] **Step 2: Run them and see them fail.**
  `vrg-container-run -- pytest tests/vergil_tooling/test_vrg_finalize_pr.py
  tests/vergil_tooling/test_worktrees.py
  tests/vergil_tooling/test_release_orchestrator.py -k
  "release_worktree or removal or vanished or teardown"` → FAIL.
- [ ] **Step 3: Implement.** Cleanup that judges what is removable (the
  finalize cleanup and any sweep of worktrees that calls it) skips every
  `release/*` worktree while protection is active. The release's own
  removals (`teardown-worktree`, supersede's `remove_release_branch`) are not
  routed through the protection and call `log_removal`.
- [ ] **Step 4: Run the tests and see them pass; run
  `vrg-container-run -- vrg-validate`; commit.** The PR body carries
  `Ref vergil-project/vergil-tooling#5187`.

### Task 3: Release the cleanup fix (deployment)

- Precondition (human-attested): a vergil-tooling release containing Task 2
  exists; `vrg-release-record check vergil-project/vergil-tooling#<Task 2>`
  exits 0.
- Human-run steps: `vrg-host upgrade`; `vrg-vm update --all`.
- Acceptance: `vrg-finalize-pr --help` on the host and one VM shows the
  installed version is the release's or later.

---

## Phase 1: the one PR path

### Task 4: Provenance relaxation for the one path (vergil-tooling)

Spec §5.1 (the relaxed rule), §5.9 (the audit follows it).

**Files:**

- Modify: `src/vergil_tooling/lib/pr_provenance.py` (`_own_pr`,
  `evaluate`, `_merge_close`)
- Modify: `src/vergil_tooling/lib/agent_audit.py` (`_actor_findings`,
  `_provenance_findings`)
- Test: `tests/vergil_tooling/test_pr_provenance.py`,
  `tests/vergil_tooling/test_agent_audit.py`, new fixtures under
  `tests/vergil_tooling/fixtures/pr_provenance/`

**Interfaces:**

- Produces: `pr_provenance.evaluate(actions, *, author, same_repo: bool,
  branch_trusted: bool) -> ProvenanceResult`. The two new keyword arguments
  are required; every caller passes them from the PR snapshot.

- [ ] **Step 1: Write the failing tests** (fixtures as JSON timelines)
  - `test_user_agent_enable_on_others_trusted_pr_allowed`
    (`others_pr_auto_merge_enable.json`): the user agent enables auto-merge
    on a same-repo PR authored by the automation App → no violation.
  - `test_user_agent_clean_merge_on_others_trusted_pr_allowed`
    (`others_pr_clean_merge.json`): a direct merge with its `closed` event,
    same case → no violation.
  - `test_enable_on_fork_pr_still_violation`: same enable on a fork PR →
    violation.
  - `test_enable_on_untrusted_branch_still_violation`: same-repo branch with
    an untrusted author → violation.
  - `test_other_forbidden_actions_unchanged`: every existing violation
    fixture still yields its violation.
  - `test_audit_actor_accepts_trusted_enable_on_others_pr` and
    `test_audit_actor_flags_enable_on_fork_pr` in `test_agent_audit.py`.
- [ ] **Step 2: Run them and see them fail.**
- [ ] **Step 3: Implement** the rule exactly as §5.1 states it: the
  relaxation covers an auto-merge enable and a clean-status direct merge
  with its merge-close event, by a trusted identity, on a same-repo PR whose
  branch has a trusted author. Nothing else changes.
- [ ] **Step 4: Pass, validate, commit.**

### Task 5: The one PR path (vergil-tooling)

Spec §5.1, §5.4.

**Files:**

- Create: `src/vergil_tooling/lib/pr_path.py`
- Modify: `src/vergil_tooling/lib/auto_merge.py` (`MergeMethod` gains
  `"MERGE"`; `enable_auto_merge` and `_merge_now` become private to the path,
  renamed `_enable` and `_merge_now`)
- Modify: `src/vergil_tooling/bin/vrg_submit_pr.py` (arms through
  `pr_path.arm`)
- Test: `tests/vergil_tooling/test_pr_path.py` (new),
  `tests/vergil_tooling/test_auto_merge.py`,
  `tests/vergil_tooling/test_vrg_submit_pr.py`

**Interfaces:**

- Produces, in `lib/pr_path.py`:

  ```python
  ArmOutcome = Literal["armed", "merged", "held", "refused"]

  @dataclass(frozen=True)
  class ArmResult:
      outcome: ArmOutcome
      pr_url: str
      head: str
      method: MergeMethod | None
      detail: str            # the reason for "refused"; "held for release <v>" for "held"

  class PRPathError(RuntimeError): ...

  def accepts(pr_url: str) -> str | None: ...        # None when accepted, else why not
  def merge_method(pr_url: str) -> MergeMethod: ...   # raises PRPathError when it cannot tell
  def arm(pr_url: str) -> ArmResult: ...
  ```

  `arm` reads the PR at its current head, runs `accepts`, the provenance
  check and the agent gates (`agent_gates.evaluate`) at that head, works out
  the method, and enables auto-merge pinned to the head; GitHub's refusal
  for a clean-status PR becomes a direct merge at that head with that
  method (outcome `merged`). A PR it does not accept or that fails a check
  is `refused` and left unarmed. `"held"` is produced from Task 19 on.
- Consumes: `finalize_watch.confirmed_head`, `pr_provenance.check_pr`,
  `agent_gates.evaluate`, `update_deps.context.DEPS_LABEL`, submit's batch
  linkage reader.

- [ ] **Step 1: Write the failing tests** in `test_pr_path.py`
  - `test_method_merge_for_trusted_post_release_head`: head
    `release/post-2.1.9`, trusted author → `MERGE`.
  - `test_method_merge_for_trusted_dependency_update_pr` → `MERGE`.
  - `test_method_rebase_for_batch_pr` → `REBASE`.
  - `test_method_squash_otherwise` → `SQUASH`.
  - `test_method_refuses_post_release_head_from_untrusted_author`:
    `PRPathError` naming why.
  - `test_arm_refuses_fork_pr`, `test_arm_refuses_draft_pr`,
    `test_arm_refuses_failing_gates`: `outcome == "refused"`, no GraphQL
    mutation sent.
  - `test_arm_clean_status_merges_directly_at_head`: the enable is refused
    as clean → one merge call with `--match-head-commit <head>` and the
    worked-out method; `outcome == "merged"`.
  - `test_arm_is_pinned_to_current_head`.
- [ ] **Step 2: Run them and see them fail.**
- [ ] **Step 3: Implement** `lib/pr_path.py`; move `vrg-submit-pr`'s arming
  call onto `pr_path.arm`, mapping `refused` to its existing failure output.
- [ ] **Step 4: Pass**, including the existing `test_vrg_submit_pr.py`
  suite unchanged except where it patched `auto_merge.enable_auto_merge`
  (patch `pr_path.arm` instead). **Validate, commit.**

### Task 6: Push mode for an open PR's branch (vergil-tooling)

Spec §5.4 (every push to an open PR's branch goes through the path).

**Files:**

- Modify: `src/vergil_tooling/lib/pr_path.py` (add `push_to_open_pr`)
- Modify: `src/vergil_tooling/bin/vrg_submit_pr.py` (add `--push`)
- Test: `tests/vergil_tooling/test_pr_path.py`,
  `tests/vergil_tooling/test_vrg_submit_pr.py`

**Interfaces:**

- Produces: `pr_path.push_to_open_pr(branch: str, *, head: str,
  lease: str | None) -> PushResult`, with `PushResult(pushed: bool,
  pr_url: str, detail: str)`. It finds the branch's open PR into `develop`,
  runs the provenance check and the agent gates against `head` before
  pushing, and pushes with `--force-with-lease=<branch>:<lease>` when
  `lease` is given (a rebased head), a plain push otherwise. A head that
  fails a check is not pushed.
- Produces: `vrg-submit-pr --push [--force-with-lease]`, run in the
  branch's worktree; exit 0 pushed, non-zero refused with the reason.

- [ ] **Step 1: Failing tests:**
  `test_push_mode_refuses_head_failing_gates` (no `git push` issued),
  `test_push_mode_pushes_passing_head`,
  `test_push_mode_rebased_head_uses_lease`,
  `test_push_mode_refuses_branch_with_no_open_develop_pr`,
  `test_push_mode_leaves_auto_merge_on` (no disable mutation).
- [ ] **Step 2: Fail; Step 3: implement; Step 4: pass, validate, commit.**

### Task 7: Mechanized automation through the path (vergil-tooling)

Spec §5.4 (`mechanized_pr` converges on the path).

**Files:**

- Modify: `src/vergil_tooling/lib/mechanized_pr.py` (`_enable_auto_merge`
  and its direct merge removed; `open_sync_pr` calls `pr_path.arm`; it keeps
  `_wait_until_merged`, which waits and never merges)
- Test: `tests/vergil_tooling/test_mechanized_pr.py`,
  `tests/vergil_tooling/test_vrg_github_repo_config.py`

- [ ] **Step 1: Failing tests:** `test_sync_pr_arms_through_pr_path`
  (`pr_path.arm` called once with the PR URL),
  `test_sync_pr_refused_arm_fails_job` (`refused` → `MechanizedPRError`
  naming the reason), `test_sync_pr_never_merges_itself` (no merge call in
  the module; waits until GitHub merges).
- [ ] **Step 2–4: fail, implement, pass, validate, commit.**

### Task 8: Update-deps through the path (vergil-tooling)

Spec §3.2 rows 3–4, §5.4.

**Files:**

- Modify: `src/vergil_tooling/lib/update_deps/pr.py` (`merge_pr` becomes
  `arm_and_wait(ctx) -> None`: `pr_path.arm`, then
  `pr_merge.wait_until_merged`)
- Modify: `src/vergil_tooling/lib/update_deps/orchestrator.py`,
  `src/vergil_tooling/lib/release/update_deps_stage.py` (call the renamed
  function)
- Modify: `src/vergil_tooling/lib/pr_merge.py` (`wait_and_merge` loses its
  last develop caller; delete it if nothing else calls it, else leave it with
  a `main`-only assertion)
- Test: `tests/vergil_tooling/test_update_deps_pr.py`,
  `tests/vergil_tooling/test_release_update_deps_stage.py`

- [ ] **Step 1: Failing tests:** `test_deps_pr_armed_with_merge_method`
  (the path works out `MERGE` from the `dependency-update` label),
  `test_deps_pr_waits_and_never_merges`, `test_release_deps_stage_uses_path`.
- [ ] **Step 2–4: fail, implement, pass, validate, commit.**

### Task 9: The back-merge through the path, and the guard test (vergil-tooling)

Spec §5.4 (the back-merge; `wait_and_merge` retired for `develop`; the
release PR into `main` the one direct merge).

**Files:**

- Modify: `src/vergil_tooling/lib/release/bump.py` (`back_merge_and_bump`
  arms its `release/post-X.Y.Z` PR with `pr_path.arm` and waits with
  `pr_merge.wait_until_merged`)
- Modify: `src/vergil_tooling/lib/release/merge.py` (`wait_and_merge`
  asserts its base is `main`)
- Create: `tests/vergil_tooling/test_one_pr_path_guard.py`
- Test: `tests/vergil_tooling/test_release_bump.py`,
  `tests/vergil_tooling/test_release_bump_resume.py`,
  `tests/vergil_tooling/test_release_merge.py`

- [ ] **Step 1: Failing tests:**
  - `test_back_merge_arms_through_path_with_merge_commit`.
  - `test_back_merge_resume_rearms_open_post_pr`.
  - `test_release_wait_and_merge_refuses_develop_base`.
  - `test_no_module_outside_pr_path_arms_or_merges_develop`: an AST scan of
    `src/vergil_tooling/` for the string `enablePullRequestAutoMerge`, a
    `gh pr merge` argument list and calls to `wait_and_merge`; the allowed
    sites are `lib/pr_path.py`, `lib/auto_merge.py` (private to the path) and
    `lib/release/merge.py` (`main` only). Any other hit fails, naming the
    file and line. `bin/vrg_gh.py`'s deny-list strings are allowed by name.
- [ ] **Step 2–4: fail, implement, pass, validate, commit.**

### Task 10: `vrg-finalize-pr` reduced to its primary job (vergil-tooling)

Spec §5, phase 1; §3.3. Only the auto-merge turn-off and the provenance and
gate re-checks are removed; every other stage (close-refs, the CD check,
the advisory sync, cleanup) stays.

**Files:**

- Modify: `src/vergil_tooling/lib/finalize_watch.py` (remove
  `_disable_auto_merge`'s callers, `recheck_gates` and the per-head
  provenance re-check; `_disable_auto_merge` itself moves to Task 20's
  sweeps, so delete it here)
- Modify: `src/vergil_tooling/bin/vrg_finalize_pr.py` (module docstring and
  the outcome table; `failed: provenance` and `failed: gates` outcomes go)
- Test: `tests/vergil_tooling/test_finalize_watch.py`,
  `tests/vergil_tooling/test_vrg_finalize_pr.py`

- [ ] **Step 1: Failing tests:** `test_watch_never_disables_auto_merge`
  (no `disablePullRequestAutoMerge` in any path of the watch, for a new
  head, a provenance violation and a gate failure),
  `test_watch_new_head_not_rechecked`,
  `test_watch_keeps_close_refs_cd_and_advisory_stages`.
- [ ] **Step 2: Fail; Step 3: implement, deleting the tests of the removed
  behavior; Step 4: pass, validate, commit.**

### Task 11: Release phase 1's tooling (deployment)

- Precondition (human-attested): a vergil-tooling release containing Tasks
  4–10; `vrg-release-record check` exits 0 for each.
- Human-run steps: `vrg-host upgrade`; `vrg-vm update --all`.
- Acceptance: `vrg-submit-pr --help` lists `--push` on the host and one VM.

### Task 12: pr-watch pushes through the path (vergil-claude-plugin)

**Files:**

- Modify: `skills/pr-watch/SKILL.md` step 4, "Push" (~line 288): every fix
  push becomes `vrg-submit-pr --push`, with `--force-with-lease` after a
  rebase or a fold; a refused push is a stop for the human, with its reason
- Modify: `skills/issue-implement/SKILL.md` (~line 620, the push that
  updates an open PR). Its push before report-ready (~line 501) is to a
  branch with no PR yet and stays a `vrg-git push`

- [ ] **Step 1:** Edit the skills; no `vrg-git push` to an open PR's branch
  remains (`grep -rn "vrg-git push" skills/` shows only pushes before a PR
  exists).
- [ ] **Step 2:** `vrg-container-run -- vrg-validate`; commit.

### Task 13: Release the plugin (deployment)

- Precondition (human-attested): a vergil-claude-plugin release containing
  Task 12; `vrg-release-record check` exits 0.
- Human-run steps: `vrg-host plugins`; `vrg-vm update --all`.
- Acceptance: the installed pr-watch skill on the host and one VM contains
  `vrg-submit-pr --push`.

### Task 14: `vrg-git` refuses a plain push to an open PR's branch (vergil-tooling)

Spec §5.4.

**Files:**

- Modify: `src/vergil_tooling/bin/vrg_git.py` (beside the frozen-push
  refusal)
- Test: `tests/vergil_tooling/test_vrg_git.py`

- [ ] **Step 1: Failing tests:** `test_push_to_open_develop_pr_branch_refused`
  (the message names `vrg-submit-pr --push`),
  `test_push_of_branch_with_no_pr_allowed`,
  `test_push_to_branch_with_closed_pr_allowed`,
  `test_pr_lookup_failure_refuses` (fail closed, naming the error),
  `test_submit_push_mode_not_refused` (the path's own push passes, by the
  same internal marker the frozen-push refusal uses for the relay).
- [ ] **Step 2–4: fail, implement, pass, validate, commit.**

### Task 15: Release Task 14 (deployment)

- Precondition (human-attested): a vergil-tooling release containing Task
  14; `vrg-release-record check` exits 0.
- Human-run steps: `vrg-host upgrade`; `vrg-vm update --all`.

### Task 16: Phase 1 exit (validation)

- Precondition: Tasks 11, 13 and 15 closed.
- Procedure:
  1. Confirm criteria 1–4 of the phase 1 exit criteria against
     vergil-tooling `origin/develop`: `test_one_pr_path_guard.py` passes in
     CI on the head, and `grep -rn "vrg-git push" skills/` in
     vergil-claude-plugin shows no push to an open PR's branch.
  2. Live: one agent PR submitted with `vrg-submit-pr` merges by auto-merge;
     one pr-watch fix pushed with `vrg-submit-pr --push` merges; a plain
     `vrg-git push` to that PR's branch is refused.
  3. Live (human-attested, the human's identity): one standalone
     `vrg-update-deps` run and one scheduled `vrg-github-repo-config` sync
     PR each merge through auto-merge, with the method the path worked out
     (merge commit and squash), cited by PR.
- Acceptance: each step's evidence cited; `Outcome: SUCCESS`.

---

## Phase 2: the release lock

### Task 17: Lock refs (vergil-tooling)

Spec §5.2, §5.4 (arming refs), §5.10.

**Files:**

- Create: `src/vergil_tooling/lib/release_lock/__init__.py`,
  `model.py`, `refs.py`, `history.py`, `arming.py`
- Modify: `src/vergil_tooling/lib/pr_workflow/github_transport.py` (factor
  out the one-file-commit builder so both use it; the relay's own
  force-push stays as it is)
- Test: `tests/vergil_tooling/test_release_lock_refs.py`,
  `test_release_lock_model.py`, `test_release_lock_arming.py`

**Interfaces:**

- Produces, `model.py`: the timing constants (Global Constraints);

  ```python
  Health = Literal["running", "broken"]

  @dataclass(frozen=True)
  class LockRecord:
      holder_identity: str
      holder_host: str
      version: str
      worktree: str              # "" until preflight creates it
      tracking_issue: str        # "" until prepare creates it; "<owner>/<repo>#<n>"
      stage: str
      health: Health
      broken_stage: str          # "" unless health == "broken"
      started_at: str            # ISO 8601 UTC
      heartbeat_at: str

  def to_json(rec: LockRecord) -> str: ...
  def from_json(text: str) -> LockRecord: ...      # raises LockParseError
  ```

- Produces, `refs.py`:

  ```python
  @dataclass(frozen=True)
  class LockState:
      sha: str | None               # None when the ref is absent
      record: LockRecord | None     # None when absent or unreadable
      error: str                    # non-empty when present but unreadable

  def read_lock(repo: str) -> LockState: ...        # unreadable → sha set, record None
  def take_lock(repo: str, rec: LockRecord) -> str | None: ...      # create-only; new sha, or None if held
  def update_lock(repo: str, rec: LockRecord, *, expected: str) -> str | None: ...  # CAS; None on mismatch
  def delete_lock(repo: str, *, expected: str) -> bool: ...         # CAS
  ```

  `LockState.held` is a property: `sha is not None`.
- Produces, `history.py`: `append(repo: str, entry: HistoryEntry) -> None`
  (compare-and-swap, retried on a concurrent append), with
  `HistoryEntry(action: Literal["break", "reclaim", "takeover", "retake"],
  old: str, actor: str, at: str, reason: str, rearmed: tuple[str, ...])`.
- Produces, `arming.py`: `create(repo, pr: int) -> str | None`,
  `renew(repo, pr, *, expected: str) -> str | None`,
  `delete(repo, pr, *, expected: str) -> bool`,
  `list_arming(repo) -> dict[int, str]` (PR number → sha).
- Any transport failure other than a lease mismatch raises
  `release_lock.RefWriteError`; a lease mismatch is the `None`/`False`
  return.

- [ ] **Step 1: Failing tests** (against a bare repo fixture served as
  `origin`):
  - `test_take_lock_create_only_second_take_returns_none`.
  - `test_update_lock_cas_mismatch_returns_none`.
  - `test_delete_lock_cas_mismatch_returns_false_and_keeps_ref`.
  - `test_unparseable_lock_reads_as_held` (`held` true, `record` None,
    `error` names the parse failure).
  - `test_lock_read_has_no_trusted_author_check` (a commit authored by an
    unlisted identity reads normally).
  - `test_history_append_concurrent_both_kept`.
  - `test_arming_create_only_and_renew_cas`.
  - `test_transport_failure_raises_not_returns_none`.
  - `test_model_round_trip` and `test_model_rejects_unknown_health`.
- [ ] **Step 2–4: fail, implement, pass, validate, commit.**

### Task 18: Staleness, `vrg-release-lock status` and `vrg-repo-status` (vergil-tooling)

Spec §5.6 (stale; status), §5.6 (the open tracking issue of the version).

**Files:**

- Create: `src/vergil_tooling/lib/release_lock/stale.py`,
  `src/vergil_tooling/bin/vrg_release_lock.py` (the `status` subcommand;
  Task 22 adds `break`)
- Modify: `pyproject.toml` (the `vrg-release-lock` entry point),
  `src/vergil_tooling/bin/vrg_repo_status.py`
- Test: `tests/vergil_tooling/test_release_lock_stale.py`,
  `test_vrg_release_lock.py`, `test_vrg_repo_status.py`

**Interfaces:**

- Produces: `stale.judge(repo: str, state: LockState, *, now: datetime)
  -> Staleness`, `Staleness(stale: bool, live: bool, reason: str,
  tracking_issue: str)`. Rules, in order: absent → not stale; unreadable →
  not stale, reason `unreadable`; `running` with a heartbeat younger than
  `STALE_AFTER_SECS` → live, never stale; names a tracking issue → stale iff
  that issue is closed; names none → look up an open trusted release
  tracking issue of the lock's version (reusing
  `preflight._trusted_release_issues`); if one exists, it is the lock's
  issue and the lock is not stale; otherwise stale iff the heartbeat is
  older than `STALE_AFTER_SECS`. A failed issue read raises; it never means
  stale.
- Produces: `vrg-release-lock status [--json]`, any identity. Text: one
  line per field plus `stale: yes|no (<reason>)`, or `no release lock held`.
  JSON: the record's fields plus `held`, `stale`, `live`, `reason`,
  `heartbeat_age_secs`, `held_prs` (open, non-draft develop PRs the path
  accepts with auto-merge off). Exit 0 on a read, 1 on a failure.
- Produces: `vrg-repo-status` prints a `Release lock:` section with the same
  fields as the text form and the PRs it holds unarmed, beside everything
  it shows today.

- [ ] **Step 1: Failing tests:** `test_running_fresh_never_stale_even_with_closed_issue`,
  `test_broken_with_open_issue_not_stale`,
  `test_closed_issue_stale`,
  `test_no_issue_old_heartbeat_stale`,
  `test_no_issue_but_open_tracking_issue_of_version_not_stale`,
  `test_stale_heartbeat_open_issue_not_stale`,
  `test_issue_read_failure_raises`,
  `test_status_json_fields`, `test_status_no_lock`,
  `test_repo_status_shows_lock_section`.
- [ ] **Step 2–4: fail, implement, pass, validate, commit.**

### Task 19: The arming gate and the holder's pass (vergil-tooling)

Spec §5.4 (the arming gate, the holder's pass), §5.5 (submit and
scheduled jobs while held).

**Files:**

- Modify: `src/vergil_tooling/lib/pr_path.py` (`arm` gains the handshake
  and `holder_pass`)
- Modify: `src/vergil_tooling/bin/vrg_submit_pr.py` (reports `held for
  release <version>; the release re-arms it` and goes on to its watch)
- Modify: `src/vergil_tooling/lib/mechanized_pr.py` (`held` is success;
  the PR is left to the unlock sweep, not replaced)
- Test: `tests/vergil_tooling/test_pr_path.py`,
  `test_vrg_submit_pr.py`, `test_mechanized_pr.py`

**Interfaces:**

- Changes: `pr_path.arm(pr_url: str, *, holder_pass: str | None = None)
  -> ArmResult`. `holder_pass` is the lock sha the calling release last
  wrote. When it equals the current lock sha, `arm` skips the handshake.
- The handshake, inside `arm`: `arming.create` → `read_lock` → lock free:
  `arming.renew`, then enable (or merge) bounded by `ARM_CALL_BOUND_SECS`,
  retries included, then `arming.delete`; lock held by another holder:
  `arming.delete`, outcome `held`, detail `held for release <version>`;
  renew finds the ref gone: abort, outcome `refused`, detail `arming ref
  expired`; `create` returns `None` (another caller is arming): poll the
  PR's auto-merge state up to `ARM_CALL_BOUND_SECS`; armed or merged →
  `armed`/`merged`; still unarmed → `PRPathError`.
- Unreadable lock: outcome `held`, detail names the read error.

- [ ] **Step 1: Failing tests:**
  - `test_arm_lock_free_writes_then_deletes_arming_ref`.
  - `test_arm_lock_held_leaves_pr_unarmed_reports_held`.
  - `test_arm_racing_lock_never_arms_under_lock`: the lock is taken between
    `create` and `read_lock` → `held`, no enable sent.
  - `test_arm_holder_pass_matching_skips_handshake`.
  - `test_arm_holder_pass_stale_sha_is_ordinary_call` (→ `held`).
  - `test_arm_create_collision_waits_for_other_caller` and
    `test_arm_create_collision_still_unarmed_raises`.
  - `test_arm_renew_finds_ref_gone_aborts`.
  - `test_arm_call_past_bound_stops_without_retry`.
  - `test_submit_reports_held_and_continues_to_watch`.
  - `test_mechanized_held_is_success`.
- [ ] **Step 2–4: fail, implement, pass, validate, commit.**

### Task 20: The disarm and re-arm sweeps (vergil-tooling)

Spec §5.1 (only the disarm sweep turns auto-merge off; sweeps skip drafts),
§5.3 steps 1.3–1.4 and 2.2, §5.4 (expiry; the second sweep).

**Files:**

- Create: `src/vergil_tooling/lib/release_lock/sweeps.py`
- Test: `tests/vergil_tooling/test_release_lock_sweeps.py`

**Interfaces:**

- Produces:

  ```python
  def wait_for_arming_refs(repo: str, *, clock: Callable[[], float] = time.monotonic) -> list[int]: ...
      # waits until every arming ref is gone; deletes (CAS) a value seen unchanged
      # for ARMING_EXPIRY_SECS, measured from this call's own first sighting of that
      # value; returns the PRs whose refs it deleted; past ARMING_WAIT_BOUND_SECS
      # raises SweepError naming the refs
  def disarm_all(repo: str) -> list[str]: ...
      # turns auto-merge off on every open PR into develop (drafts included: they
      # cannot be armed, so there is nothing to turn off) and confirms each is off
  def quiesce(repo: str) -> QuiesceResult: ...
      # wait_for_arming_refs → disarm_all → if any ref was deleted, disarm_all again →
      # list the arming refs and every open PR's auto-merge state once more and
      # disarm any found armed
  def rearm_all(repo: str, *, holder_pass: str | None = None) -> RearmResult: ...
      # offers every open, non-draft PR into develop with auto-merge off to
      # pr_path.arm; idempotent
  ```

  `RearmResult(armed: list[str], merged: list[str], refused: list[tuple[str,
  str]])`. Any failed disarm or GitHub call raises `SweepError`; a refused
  arm is reported, never raised.

- [ ] **Step 1: Failing tests:**
  `test_wait_deletes_only_value_unchanged_for_expiry`,
  `test_wait_renewal_restarts_expiry_clock`,
  `test_wait_past_bound_raises_naming_refs`,
  `test_quiesce_second_disarm_after_expired_ref_deleted`,
  `test_quiesce_final_check_disarms_late_arm`,
  `test_disarm_failure_raises`,
  `test_rearm_skips_drafts`, `test_rearm_idempotent`,
  `test_rearm_reports_refused_outsider_pr`,
  `test_rearm_clean_status_pr_merges`.
- [ ] **Step 2–4: fail, implement, pass, validate, commit.**

### Task 21: The lock owner (vergil-tooling)

Spec §5.2 (the background heartbeat; all writes through one owner; a CAS
failure against the holder's own write retried), §5.6 (a broken or
reclaimed holder stops).

**Files:**

- Create: `src/vergil_tooling/lib/release_lock/holder.py`
- Test: `tests/vergil_tooling/test_release_lock_holder.py`

**Interfaces:**

- Produces:

  ```python
  class LockLost(ReleaseError): ...     # the lock was broken, reclaimed or taken over

  class LockOwner:
      @classmethod
      def take(cls, repo: str, rec: LockRecord) -> "LockOwner | None": ...   # None: held
      @classmethod
      def adopt(cls, repo: str, state: LockState) -> "LockOwner": ...
      @property
      def sha(self) -> str: ...           # the holder's pass for pr_path.arm
      def update(self, **fields: str) -> None: ...   # stage, health, broken_stage, tracking_issue, worktree
      def start_heartbeat(self) -> None: ...
      def stop_heartbeat(self) -> None: ...
      def release(self) -> None: ...      # stop_heartbeat, then delete_lock(expected=sha)
  ```

  One lock serialises every write. A CAS mismatch is re-read: if the ref
  still holds a record the owner wrote (same holder, version and start
  time; the mismatch came from its own heartbeat write racing), retry; if
  it is absent or someone else's, raise `LockLost` on the next `update`
  (the heartbeat thread records the loss and stops; it never re-takes).

- [ ] **Step 1: Failing tests:** `test_heartbeat_refreshes_during_blocking_wait`,
  `test_writes_serialised_no_lost_update`,
  `test_own_write_cas_failure_retried`,
  `test_broken_lock_raises_lock_lost_on_next_update`,
  `test_heartbeat_stops_on_loss_and_never_retakes`,
  `test_release_stops_heartbeat_before_delete`.
- [ ] **Step 2–4: fail, implement, pass, validate, commit.**

### Task 22: Reclaim and `vrg-release-lock break` (vergil-tooling)

Spec §5.6.

**Files:**

- Create: `src/vergil_tooling/lib/release_lock/reclaim.py`
- Modify: `src/vergil_tooling/bin/vrg_release_lock.py` (`break`)
- Test: `tests/vergil_tooling/test_release_lock_reclaim.py`,
  `test_vrg_release_lock.py`

**Interfaces:**

- Produces: `reclaim.reclaim_if_stale(repo: str, *, actor: str) ->
  ReclaimResult | None`: judges with `stale.judge`; not stale → `None`;
  stale → `delete_lock(expected=state.sha)`; a lost race → `None`; won →
  `rearm_all`, then `history.append(action="reclaim", rearmed=…)`, then a
  comment on the tracking issue when the lock names one.
- Produces: `reclaim.break_lock(repo: str, *, actor: str, reason: str) ->
  ReclaimResult`: records the history entry and tracking-issue comment
  first, then deletes (CAS) and runs `rearm_all`.
- Produces: `vrg-release-lock break --reason "<why>"`, refused under every
  agent identity and agent signal (the same check `bin/vrg_release.py`
  uses); `--reason` required and non-empty.

- [ ] **Step 1: Failing tests:** `test_reclaim_not_stale_does_nothing`,
  `test_reclaim_stale_deletes_rearms_records`,
  `test_reclaim_lost_race_returns_none`,
  `test_reclaim_refuses_unparseable_lock`,
  `test_reclaim_broken_open_issue_refused`,
  `test_break_refused_for_agent_identity`,
  `test_break_refused_with_agent_signal`,
  `test_break_requires_reason`,
  `test_break_records_before_delete_then_rearms`.
- [ ] **Step 2–4: fail, implement, pass, validate, commit.**

### Task 23: Pre-check, lock, quiesce and cut (vergil-tooling)

Spec §5.3 step 1, §5.2 (the worktree, the tracking issue, health), §5.6
(a new release reclaims a stale lock).

**Files:**

- Modify: `src/vergil_tooling/lib/release/preflight.py` (`preflight`:
  the pre-check, lock and quiesce before `_release_contents`)
- Modify: `src/vergil_tooling/lib/release/context.py` (`ReleaseContext`
  gains `lock: LockOwner | None`)
- Modify: `src/vergil_tooling/lib/release/tracking.py`
  (`create_tracking_issue` calls `ctx.lock.update(tracking_issue=…)`
  immediately after creating the issue)
- Modify: `src/vergil_tooling/lib/release/orchestrator.py` (before each
  stage `ctx.lock.update(stage=…)`; at the first recorded failure, deferred
  ones included, `update(health="broken", broken_stage=…)`; heartbeat
  started at the lock, stopped at `unlock`)
- Test: `tests/vergil_tooling/test_release_preflight.py`,
  `test_release_orchestrator.py`, `test_release_prepare.py`

**Interfaces:**

- Consumes: `LockOwner`, `sweeps.quiesce`, `sweeps.rearm_all`,
  `reclaim.reclaim_if_stale`, `pending_deps_prs`, `_check_pending_deps`,
  `check_develop_cd`.
- Order in `preflight` (§5.3 step 1): sync `develop` under the checkout
  lock and run the pending-deps and develop CD gates as pre-checks
  (refusal: exit with nothing locked), record the judged head;
  `reclaim_if_stale`; `LockOwner.take` (held → refuse naming holder,
  version and tracking issue); `quiesce`; sync again; always re-run the
  pending-deps gate; if the head moved, re-run the head-judging gates on the
  new head; create the worktree and `update(worktree=…)`; then
  `_release_contents` records `cut`.
- The early-exit unlock path: every exit after the lock is taken and before
  the tracking issue exists (a refusal or an exception) runs
  `ctx.lock.release()` and then, if `disarm_all` has started,
  `rearm_all(holder_pass=None)`, in a `finally` scoped to that window; a
  failure in it is reported with both errors, never swallowed.

- [ ] **Step 1: Failing tests:**
  - `test_precheck_refusal_locks_nothing`.
  - `test_second_release_refused_before_disarm`.
  - `test_stale_lock_reclaimed_before_take`.
  - `test_quiesce_runs_before_cut`.
  - `test_pending_deps_gate_always_rerun_after_disarm`.
  - `test_merge_between_gates_and_disarm_rejudges_new_head`.
  - `test_unmoved_head_cut_without_rerunning_head_gates`.
  - `test_refusal_after_lock_runs_unlock_path_and_rearms`.
  - `test_exception_before_tracking_issue_runs_unlock_path`.
  - `test_tracking_issue_written_into_lock_immediately`.
  - `test_stage_and_health_written_per_stage`.
  - `test_deferred_failure_marks_broken_when_recorded`.
  - `test_one_release_check_kept_as_second_guard`.
- [ ] **Step 2–4: fail, implement, pass, validate, commit.**

### Task 24: Unlock, rearm, stray merges and the abandon step (vergil-tooling)

Spec §5.3 step 2, §5.4 (the stray-merge checks), §5.8 (the abandon step's
removals are sanctioned and logged).

**Files:**

- Modify: `src/vergil_tooling/lib/release/orchestrator.py` (stages
  `unlock` and `rearm` after `release-records`, before `close-tracking`; the
  stray-merge check before `merge-release` and before `unlock`)
- Modify: `src/vergil_tooling/lib/release/handoff.py` (`_NEEDS_ALL`
  gains `unlock` and `rearm`)
- Modify: `src/vergil_tooling/lib/release/bump.py`,
  `update_deps_stage.py` (pass `holder_pass=ctx.lock.sha` to the path)
- Create: `src/vergil_tooling/lib/release/abandon.py`
- Create: `src/vergil_tooling/lib/release/stray.py`
- Test: `tests/vergil_tooling/test_release_orchestrator.py`,
  `test_release_handoff.py`, `test_release_abandon.py` (new),
  `test_release_stray.py` (new)

**Interfaces:**

- Produces: `stray.stray_merges(ctx: ReleaseContext) -> list[str]`: PRs
  merged into `develop` after `cut` other than the holder's back-merge and
  dependency-update PRs (first-parent walk of `cut..origin/develop`).
- Produces: `abandon.abandon_release(ctx: ReleaseContext, *, reason: str)
  -> None`: closes the release PR into `main`, removes `release/X.Y.Z`
  (branch and worktree, logged with `worktrees.log_removal`), closes the
  tracking issue with an `abandoned` comment naming `reason`, then the full
  unlock path.
- Stage `unlock`: `ctx.lock.release()`. Stage `rearm`: `rearm_all()`; a
  refused PR is listed on the tracking issue, not a failure; a `SweepError`
  is. Neither stage reads the lock after `unlock`.

- [ ] **Step 1: Failing tests:**
  `test_unlock_and_rearm_run_after_records_before_close`,
  `test_unlock_held_back_by_earlier_failure`,
  `test_rearm_failure_keeps_tracking_issue_open`,
  `test_holder_pass_lets_back_merge_and_deps_arm_under_lock`,
  `test_standalone_update_deps_held_under_lock`,
  `test_stray_merge_before_merge_release_abandons`
  (the PR is named; the release PR, branch, worktree and tracking issue are
  closed or removed; the lock is gone; PRs are re-armed),
  `test_stray_merge_before_unlock_reported_and_continues`,
  `test_abandon_removal_logged`.
- [ ] **Step 2–4: fail, implement, pass, validate, commit.**

### Task 25: `--resume` and `--supersede` under the lock (vergil-tooling)

Spec §5.3 (resume, re-take, supersede takeover), §5.6 (a lost holder
stops; the human resumes or supersedes).

**Files:**

- Modify: `src/vergil_tooling/lib/release/resume.py`,
  `src/vergil_tooling/lib/release/supersede.py`,
  `src/vergil_tooling/lib/release/preflight.py` (`_resume_context`)
- Test: `tests/vergil_tooling/test_release_resume.py`,
  `test_release_supersede.py`

**Interfaces:**

- Resume before `unlock`: lock names this identity and tracking issue →
  `LockOwner.adopt` (host ignored), health back to `running`; lock absent →
  `LockOwner.take`, `history.append(action="retake")`, `quiesce`; then, at a
  stage before `merge-release`, `stray_merges` after the disarm, and if any,
  `abandon_release` and tell the human to run a fresh release; lock held by
  another release → refuse naming it. Resume at `unlock` or later takes and
  adopts no lock.
- Supersede: lock held by the superseded release and `broken` or
  heartbeat-stale → `update_lock` compare-and-swap takeover with
  `tracking_issue=""` and the new holder, `history.append(action="takeover")`,
  comments on both tracking issues; the new tracking issue is written into
  the lock before the old one is closed; `running` and fresh → refuse naming
  it; no lock → `take` plus `quiesce`.

- [ ] **Step 1: Failing tests:** `test_resume_adopts_own_lock_on_other_host`,
  `test_resume_retakes_absent_lock_quiesces_and_records`,
  `test_resume_before_merge_release_stray_check_after_disarm_abandons`,
  `test_resume_refuses_lock_of_other_release`,
  `test_resume_at_unlock_takes_no_lock`,
  `test_supersede_takes_over_broken_lock_without_rearm`,
  `test_supersede_refuses_running_fresh_lock`,
  `test_supersede_lock_never_names_closed_issue`,
  `test_supersede_with_no_lock_takes_fresh_and_quiesces`,
  `test_lost_holder_stops_with_clear_error`.
- [ ] **Step 2–4: fail, implement, pass, validate, commit.**

### Task 26: The merge watch under the lock (vergil-tooling)

Spec §5.5, §5.7 layers 2–3, §5.4 (`vrg-pr-await`'s contract change).

**Files:**

- Modify: `src/vergil_tooling/lib/finalize_watch.py`,
  `src/vergil_tooling/lib/pr_await.py`,
  `src/vergil_tooling/bin/vrg_finalize_pr.py`,
  `src/vergil_tooling/bin/vrg_pr_await.py` (docstring: the watch's own
  re-arm through the path is now allowed)
- Test: `tests/vergil_tooling/test_finalize_watch.py`,
  `test_pr_await.py`, `test_vrg_finalize_pr.py`, `test_vrg_pr_await.py`

**Interfaces:**

- Produces: outcome `held by release` and the constant
  `EXIT_HELD_BY_RELEASE` (the next unused exit code of each command,
  documented in its `--help`), shared from `lib/finalize_watch.py`.
- Per poll, before judging auto-merge off: `read_lock`. Held and live →
  report `held by release <version> (stage <stage>, <health>)`, keep
  polling at the normal pace. Held and broken or heartbeat-stale → report
  the broken release, poll every `BROKEN_WATCH_POLL_SECS`, and call
  `reclaim_if_stale`. Not held and the PR open, unarmed → `pr_path.arm`.
  Deadline (`VRG_FINALIZE_DEADLINE`) passes while held → `held by release`.

- [ ] **Step 1: Failing tests:**
  `test_held_pr_never_reported_auto_merge_off`,
  `test_watch_reports_release_version_stage_health`,
  `test_broken_release_slows_poll`,
  `test_watch_reclaims_stale_lock`,
  `test_watch_does_not_reclaim_broken_with_open_issue`,
  `test_watch_rearms_own_pr_when_unlocked`,
  `test_deadline_while_held_exits_held`,
  `test_pr_await_rearm_through_path`.
- [ ] **Step 2–4: fail, implement, pass, validate, commit.**

### Task 27: Cleanup safety on the lock (vergil-tooling)

Spec §5.8.

**Files:**

- Modify: `src/vergil_tooling/lib/worktrees.py` (`release_protection`
  keyed on the lock: protects the worktree the lock names and every
  `release/*` worktree while any lock is held; an unreadable lock protects)
- Test: `tests/vergil_tooling/test_worktrees.py`,
  `test_vrg_finalize_pr.py`

- [ ] **Step 1: Failing tests:** `test_cleanup_skips_worktree_lock_names`,
  `test_cleanup_skips_release_worktrees_while_lock_held`,
  `test_cleanup_unreadable_lock_protects`,
  `test_cleanup_removes_release_worktree_with_no_lock`,
  `test_release_own_teardown_not_blocked`.
- [ ] **Step 2–4: fail, implement, pass, validate, commit.**

### Task 28: The backstop audit (vergil-tooling)

Spec §5.9.

**Files:**

- Modify: `src/vergil_tooling/lib/agent_audit.py`,
  `src/vergil_tooling/bin/vrg_agent_audit.py`
- Test: `tests/vergil_tooling/test_agent_audit.py`

- [ ] **Step 1: Failing tests:**
  `test_audit_flags_accepted_pr_unarmed_with_no_lock`,
  `test_audit_quiet_on_unarmed_pr_while_lock_held`,
  `test_audit_flags_draft_pr_into_develop`,
  `test_audit_flags_stale_lock`,
  `test_audit_reports_outsider_prs_on_own_line_not_breach`,
  `test_audit_never_arms` (no mutation issued).
- [ ] **Step 2–4: fail, implement, pass, validate, commit.**

### Task 29: Release phase 2's tooling (deployment)

- Precondition (human-attested): a vergil-tooling release containing Tasks
  17–28; `vrg-release-record check` exits 0 for each. That release is itself
  cut by the previous installed version, so it is the last release without
  a lock.
- Human-run steps: `vrg-host upgrade`; `vrg-vm update --all`.
- Acceptance: `vrg-release-lock status` runs on the host and one VM and
  prints `no release lock held` for vergil-tooling.

### Task 30: Skills re-run the watch on `held by release` (vergil-claude-plugin)

Spec §5.5 (the only skill rule), §5.7 layer 3.

**Files:**

- Modify: `skills/issue-implement/SKILL.md` (submit and watch),
  `skills/pr-watch/SKILL.md`

- [ ] **Step 1:** Add the one rule: on the watch's `held by release` exit,
  report the release it names and re-run the same watch command; never turn
  auto-merge on by hand, never treat it as a failure.
- [ ] **Step 2:** `vrg-container-run -- vrg-validate`; commit.

### Task 31: Release the plugin (deployment)

- Precondition (human-attested): a vergil-claude-plugin release containing
  Task 30; `vrg-release-record check` exits 0.
- Human-run steps: `vrg-host plugins`; `vrg-vm update --all`.

### Task 32: A live release under the lock (validation)

- Preconditions: Tasks 29 and 31 closed. Human-attested: the human runs the
  next vergil-tooling `vrg-release` with the installed Task 29 version while
  at least one agent PR into `develop` is in flight (CI running), and a
  second agent PR is submitted during the release.
- Procedure (the agent observes and records):
  1. `vrg-release-lock status --json` during the release shows the holder,
     version, tracking issue and a fresh heartbeat; `vrg-repo-status` shows
     the lock and the held PRs.
  2. The in-flight PR's auto-merge is off after the quiesce; the second
     PR's submit reported `held for release <version>`; neither merged
     between the cut and `unlock` (first-parent log of `develop`).
  3. The back-merge merged with a merge commit during the release.
  4. After `rearm`, both PRs are armed and merge; no lock remains; the
     release worktree was never removed before `teardown-worktree`
     (activity log).
  5. The agents' watches reported the release while held and ended
     merged, or with `held by release` and were re-run.
- Acceptance: each step's evidence cited; `Outcome: SUCCESS`.

### Task 33: Fixture runs

Filed by `fixture-run` as above, one trio per fixture, each blocked by Task
29.

### Documentation review and retrospective

- **Documentation review** (`vergil-tooling`, `Plan task: doc-review`): sweep
  the vergil-tooling site docs (`docs/site/`) for the release guide (the
  lock, `--resume`, `--supersede`, `vrg-release-lock status` and `break`,
  stale locks), the PR workflow guide (the one PR path, `--push`, `held for
  release`, the watch's exit), `vrg-repo-status` and the agent audit's
  findings; spawn per-repo doc tasks for vergil-claude-plugin and the `docs`
  repo where they need changes, each under the epic.
- **Retrospective** (`.github`, `--kind retrospective`,
  `Plan task: retrospective`): `epic-retrospective`, once every other task is
  closed.
