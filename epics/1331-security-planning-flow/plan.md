# Harden the security-planning flow — implementation plan

- **Epic:** vergil-project/.github#1331
- **Planning issue:** vergil-project/vergil-claude-plugin#828 (review history
  is in its comments)
- **Status:** approved

> **For agentic workers:** each task is one issue and one PR in the repo named
> in its heading, worked with `issue-implement`; deployment tasks are worked
> with `issue-deploy`. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** the tools refuse the unsafe moves in the security-planning flow
(retarget from an issue the advisory does not track, a sync that closes an
advisory early, a hold that fails open), and `epic-create` and `epic-review`
say what to do around those refusals.

**Architecture:** three vergil-tooling tasks (retarget, sync guard, one
fail-closed hold rule), a vergil-tooling release, then the skill text that
names the new commands. The skill text that needs no new tool behavior ships
first, in parallel with the tooling.

**Tech Stack:** Python 3 and pytest (vergil-tooling), Markdown skill text
(vergil-claude-plugin), MkDocs pages (vergil-tooling, vergil-claude-plugin,
docs).

**Spec:** [`spec.md`](spec.md) in this directory. Executors read both.

## Global Constraints

- Public text stays in general terms: issue titles and bodies, commit
  messages, PR bodies, report-ready notes and docs describe mechanisms, and
  refer to findings by issue ref and GHSA ID alone (spec §6.4, §8).
- Where a check can be mechanical, it is; the skill text says what to do
  when a tool refuses (spec §1).
- Every refusal names what it refused and the command that would succeed,
  and nothing is written before a refusal (spec §7).
- No silent failures: a failed GitHub read fails the command loudly, never
  becomes "not tracking", "not held" or a close (spec §6.2, §6.3, §7).
- Agents never release: the vergil-tooling and plugin releases are
  human-gated preconditions of their deployment tasks (spec §8).
- Validation runs only through `vrg-container-run -- vrg-validate` in the
  task repo's worktree. Commit with `vrg-commit`; hand off with
  `vrg-pr-workflow report-ready`.

## Review Focus

1. **An interrupted retarget, re-run with identical arguments**, after it
   rewrote the tracking lines and failed while unmarking any dropped issue,
   or after it finished but reported an error. It finishes (or writes
   nothing); it is not refused by the `--from` check. Pinned in Task 2.
2. **An intake-route split of an item whose advisory also tracks another
   open issue.** The route succeeds and the other issue stays tracked.
   Pinned in Task 2.
3. **An advisory a human closed with `close`, whose tracked issue is a
   closed planning item already drained into its archive.** The sync leaves
   it `closed` and does not report it held. Pinned in Task 3.
4. **A fix task closed as a duplicate, the advisory's only open tracked
   issue before that.** The advisory stays `draft` and is reported held.
   Pinned in Task 3.
5. **A planning issue with a `Security advisory:` line for an advisory the
   listing does not return.** With the `security` label: held, in the queue
   and in the docs-PR close. Without it: a broken pair, reported, not held.
   Pinned in Task 4.

---

## Delivery table

| Task | Repo | Kind | What | Blocked by |
| --- | --- | --- | --- | --- |
| 1 | .github | task (#1332) | Spec and plan (this PR) | — |
| 2 | vergil-tooling | task (#4718) | `retarget --from`, the open-issue drop check, intake-route callers | — |
| 3 | vergil-tooling | task (#4719) | Sync close guard and `held:` report | — |
| 4 | vergil-tooling | task (#3996) | One fail-closed planning hold rule | 2 |
| 5 | .github | deployment (#1360) | vergil-tooling release with Tasks 2–4 attested | 2, 3, 4 |
| 6 | vergil-claude-plugin | task (#1178) | Skill part 1: step 0 check, wording rule, gate, not-planned stop, hold text | — |
| 7 | vergil-claude-plugin | task (#1179) | Skill part 2: `--from` hand-offs and the unreadable-advisory note | 5, 6 |
| 8 | docs | task (#141) | Intake page: retarget names `--from` | 5 |
| 9 | .github | deployment (#1361) | Plugin release with Task 7 attested | 7 |
| — | vergil-tooling | bookend (#4697) | Documentation review | 2, 3, 4, 6, 7, 8 |
| — | .github | bookend (#1333) | Retrospective | all |

Tasks 2, 3 and 6 start at once. The window in which a retarget command
without `--from` is refused runs from Task 5 to Task 9 (spec §6.7); Tasks 7
and 8 are the next work after Task 5.

**Filing.** Tasks 2, 3 and 5–9 are filed at epic-create's filing step, each
in its repo and linked under #1331, with its `Blocked-by:` refs. Task 4 is
vergil-project/vergil-tooling#3996, routed out of the deferred queue and
linked under #1331 then, with its body's scope note pointing at spec §6.3.

**Advisory hand-off.** At the filing step, each advisory tracking the
planning issue is retargeted, once, to every task that fixes it (spec §6.5):

| Advisory | Fix tasks |
| --- | --- |
| GHSA-4crp-43r6-45pm | 6 |
| GHSA-9xmr-vx6j-rh7f | 6 |
| GHSA-6pr8-7hv7-h9rw | 2, 7 |
| GHSA-4rjw-x8xw-5742 | 2, 3, 6, 7 |
| GHSA-xhm6-fmj7-pgjc | 3 |
| GHSA-3q5q-26ff-26j4 | 3 |
| GHSA-3xmc-jfc4-w489 | 4 |
| GHSA-p49g-jghc-747h | 4, 6 |
| GHSA-j8xj-8jwq-wh82 | 6, 7 |
| GHSA-53xv-77w3-fpvx | 6 |
| GHSA-3fxw-j6mr-3mq5 | 6 |
| GHSA-qrrh-m82w-78vg | 3, 5 |
| GHSA-4338-qqxp-rc2f | 3 |
| GHSA-f6pf-7r3g-xfgv | 2 |
| GHSA-c6j6-qxqp-w348 | 2, 7 |
| GHSA-h6f6-38fj-w6g4 | 5 |
| GHSA-642m-8772-xh5j | 2 |
| GHSA-w5xf-7p4w-fp67 | 3 |

---

### Task 2: `retarget --from`, the open-issue drop check, intake-route callers (vergil-tooling)

**Files:**

- Modify: `src/vergil_tooling/lib/security_records.py` (`retarget`; new `pair`)
- Modify: `src/vergil_tooling/bin/vrg_security_record.py` (`retarget` parser, `_retarget`, `_finish_pair`)
- Modify: `src/vergil_tooling/lib/intake_route.py` (`_retarget`, its three callers, the several-advisories split message, `_retarget_commands`)
- Modify: `docs/site/docs/guides/security-records.md`, `docs/site/docs/reference/dev/security-record.md`, `docs/site/docs/reference/cli-tools-overview.md` (every retarget command gains `--from`; the two refusals described)
- Test: `tests/vergil_tooling/test_security_records.py`, `tests/vergil_tooling/test_vrg_security_record.py`, `tests/vergil_tooling/test_intake_route.py`

**Interfaces:**

- Produces: `security_records.retarget(home: str, ghsa_id: str, targets: Sequence[IssueRef], *, source: IssueRef, drop: Sequence[IssueRef] = (), renamed: Sequence[IssueRef] = ()) -> tuple[str, ...]`. `source` is keyword-only and required.
- Produces: `security_records.pair(home: str, ghsa_id: str, issue: IssueRef) -> None`: the first pairing (mark `issue`, write the advisory's first `Tracking issue:` line). It raises `RetargetError` before any write when the advisory already has a tracking line, unless that line is `issue` alone (a resumed `create`, which then only finishes the mark). `_finish_pair` calls it instead of `retarget`.
- Produces: `security_records.pair_check(home: str, ghsa_id: str, issue: IssueRef | None) -> None`: the same refusal, read-only. `_finish_pair` calls it first, before it creates, marks or links any issue (with `issue=None` when the issue is not yet known: then any existing tracking line refuses); `pair` calls it too.
- Produces: CLI `vrg-security-record retarget <ghsa> --from <ref> --to <ref> [--to …] [--drop <ref> …]`; `--from` required, `--drop` repeatable.
- Produces: `intake_route._retarget_commands(ghsas, source: IssueRef, targets: Mapping[str, Sequence[IssueRef]]) -> str`: one command per advisory, carrying `--from <source>` and one `--to` per ref of the same target set the internal call used for that advisory. For a transfer, `source` is the pre-transfer ref, which is still on the tracking line.

**Rules** (spec §6.1). Both checks run before the first write (before any target is marked), and a refusal raises `RetargetError` naming the advisory's current tracking refs and the command that would succeed (for the source check: the same `--to` set with `--from` a current tracking ref):

- **Source check:** accepted when `source` is on a current `Tracking issue:` line; or when the tracking lines are already exactly the targets (as a set) and `source` is tied to that move: either its body still names `ghsa_id` (an interrupted run), or it carries the comment `unmark_issue` posts (`_moved_comment`) naming exactly these targets (a completed run). In the second case the run only finishes the unmarks; it moves nothing. Every other source refuses.
- **Unmark order:** the source is unmarked last, after every other dropped issue.
- **Drop check:** the dropped set is (current tracking refs ∪ issues whose body names `ghsa_id`) minus targets, minus `renamed`. Each dropped issue that is open, other than `source`, refuses unless it is in `drop`. A dropped ref missing from the `security_issues` listing has its state read directly; a failed read refuses. The drop-check refusal names, as the command that would succeed, the keep form (the same targets plus one `--to` per open dropped issue), and says that `--drop` is a human decision; it never prints a `--drop` form.
- **Internal callers** pass the routed item as `source`, and as targets the advisory's current tracking refs minus the routed item plus the new issue or issues, so a route never meets the drop check.

- [ ] **Step 1: Write the failing tests**
  - `test_retarget_refuses_untracked_source`: an advisory tracking `A`; `source=B` raises `RetargetError` naming `A` and suggesting `--from A` with the same `--to` set, and no mark, description write or unmark call is made.
  - `test_retarget_refuses_dropping_open_issue`: tracking `A` (open) and `M`; `source=M, targets=(T,)` raises, naming `A`, with the suggested command `--from M --to A --to T` and no `--drop` in the message; with `drop=(A,)` it succeeds.
  - `test_retarget_rerun_after_interrupted_unmark`: tracking lines already `(T,)`, `M`'s body still names the advisory; `source=M, targets=(T,)` unmarks `M` and succeeds (Review Focus 1).
  - `test_retarget_rerun_after_interrupted_other_unmark`: tracking lines already `(T,)`, `X` already unmarked, `M` (the source) still marked, as the source-last order leaves it; `source=M, targets=(T,), drop=(X,)` unmarks `M` and succeeds.
  - `test_retarget_rerun_after_completed_run`: everything done, `M` carrying the moved-to comment naming `(T,)`; the identical re-run succeeds and writes nothing.
  - `test_retarget_refuses_unrelated_source_when_targets_match`: tracking lines already `(T,)`; `source=B`, unmarked and with no moved-to comment (or one naming other targets), refuses.
  - `test_retarget_unmarks_source_last`: with `M` and `X` dropped, `X`'s unmark comes before `M`'s.
  - `test_retarget_source_may_be_dropped`: `source=M` is dropped without `drop`.
  - `test_drop_check_reads_unlisted_issue_state`: a tracked open issue without the `security` label (absent from the listing) refuses to be dropped; a failed state read refuses.
  - `test_cli_retarget_requires_from`: the parser exits 2 without `--from`.
  - `test_create_pairs_new_advisory`, `test_create_attach_pairs`, `test_create_resume_finishes_pair`: `create`, `create --issue` and `create --resume` each end with the advisory tracking the issue and the issue marked.
  - `test_pair_refuses_existing_tracking`: an advisory tracking `A`; `pair(…, B)` raises before any write.
  - `test_create_resume_onto_tracked_advisory_writes_nothing`: `create --resume G` (and `create --issue B --resume G`) where `G` tracks `A` exits non-zero with no issue created, marked or linked.
  - `test_split_keeps_other_tracked_issues`: an item `I` whose advisory also tracks open `A`; a split into `T1, T2` calls `retarget` with `source=I` and targets `(A, T1, T2)`, and succeeds (Review Focus 2).
  - `test_transfer_and_recreate_pass_source`: each caller passes the routed item as `source`.
  - `test_recovery_commands_carry_from`: `_retarget_commands` and the several-advisories split message include `--from <item>`.
  - `test_recovery_command_keeps_other_tracked_issues`: a failed transfer of `I` whose advisory also tracks open `A` prints `--from <I's old ref> --to <A> --to <new ref>`, which the drop check accepts.
- [ ] **Step 2: Run** `vrg-container-run -- vrg-validate`; expect the new tests to fail, because `retarget` takes no `source` and `pair` does not exist; a new test that passes here is not exercising the new checks.
- [ ] **Step 3: Implement** the rules above in `retarget`, the CLI and `intake_route`. The `retarget` docstring states both checks and the re-run case.
- [ ] **Step 4: Update the three docs pages:** every retarget example names `--from`; the security-records guide says what each refusal means and that `--drop` names an open issue deliberately left.
- [ ] **Step 4b: Refactor:** fold `_retarget_commands`, `_transfer_recovery` and the several-advisories split message onto one command builder. Tests stay green.
- [ ] **Step 5: Run** `vrg-container-run -- vrg-validate`; expect PASS.
- [ ] **Step 6: Commit** `vrg-commit --type feat --scope security-record --message "require retarget's source and refuse to drop an open tracked issue"`, then report-ready.

### Task 3: Sync close guard and `held:` report (vergil-tooling)

**Files:**

- Modify: `src/vergil_tooling/lib/epics.py` (new `queue_or_archive_kind`)
- Modify: `src/vergil_tooling/lib/security_records.py` (`SecurityIssue`, `_SECURITY_ISSUES_JQ`, `pending_syncs`, new `guard_check`, `records_pass`; `security_issues` gains no reads)
- Modify: `src/vergil_tooling/bin/vrg_security_record.py` (`sync` help: who runs it)
- Modify: `docs/site/docs/guides/security-records.md` (the sync section: the guard, `held:`, and that the human or the user agent runs `sync`)
- Test: `tests/vergil_tooling/test_security_records.py`, `tests/vergil_tooling/test_vrg_epic_audit.py`

**Interfaces:**

- Produces: `epics.queue_or_archive_kind(epic: IssueRef) -> str | None`: `"intake"`, `"planning"` or `"deferred"` for a live queue epic or a queue archive epic (`epics.is_archive`, which counts the `archive` label or a stamped archive title alone, plus the kind from its label or title prefix), else None.
- Produces: `SecurityIssue` gains `state_reason: str | None` (GitHub's REST `state_reason`, lower-cased, read by the existing listing through `_SECURITY_ISSUES_JQ`; None while open). It gains no parent field: `security_issues`, which `retarget` and `close_record` also call, makes no new reads.
- Produces: `security_records.GuardResult` (frozen dataclass): `held: list[tuple[str, tuple[IssueRef, ...]]]` (GHSA ID and the refs that fire the guard) and `errors: list[tuple[str, IssueRef, str]]` (GHSA ID, the ref whose read failed, the scrubbed error).
- Produces: `security_records.guard_check(classification: Classification, issues: Sequence[SecurityIssue]) -> GuardResult`. Only for each `draft` paired advisory whose tracked issues are all closed, it reads each tracked issue's native parent (`epics.native_parent_of`, then `queue_or_archive_kind`). `pending_syncs` takes its result and lists no change for an advisory in `held` or `errors`.

**Rules** (spec §6.2). The guard fires when any tracked issue is a queue item, or closed with a `state_reason` other than `completed` (a closed issue with a null `state_reason` fires it). `pending_syncs` never lists a `closed` target for an advisory the guard fires on, and never moves a `closed` advisory to `draft` because of it. `records_pass` prints `- held: <GHSA> (<refs>): retarget it to a fix task, or close it with vrg-security-record close` per held advisory, and counts holds in the summary apart from broken pairs; a hold does not fail the pass. A failed parent read is reported for that advisory only (`- error: <GHSA> (<ref>): …`), the advisory is left unchanged, the rest are synced, and with `apply` the pass then exits 1, as it does for a broken pair.

- [ ] **Step 1: Write the failing tests**
  - `test_queue_or_archive_kind`: live planning epic, planning archive epic, a planning archive that has lost its `archive` label (stamped title only), development epic, a non-epic.
  - `test_guard_holds_queue_item`: tracked issue closed as completed under a planning epic; no change listed, one held.
  - `test_guard_holds_drained_queue_item`: the same item under a planning archive epic.
  - `test_guard_holds_non_completed_reasons`: parametrized over `not_planned`, `duplicate`, an unknown reason and a closed issue with a null `state_reason` (Review Focus 4).
  - `test_guard_leaves_closed_advisory_alone`: an advisory already `closed`, tracking a closed, drained planning item; no change, not held (Review Focus 3).
  - `test_completed_fix_tasks_close_advisory`: every tracked issue a development-epic task closed as completed; the advisory moves to `closed`.
  - `test_records_pass_reports_held_without_failing`: output has the `- held:` line and the count; exit 0 with `apply`.
  - `test_parent_read_error_is_per_advisory`: one advisory's parent read raises `GitHubAPIError`; that advisory is reported as an error and unchanged, another advisory still syncs, and `apply` exits 1.
  - `test_security_issues_makes_no_parent_reads`: `security_issues` calls no parent or label read.
- [ ] **Step 2: Run** `vrg-container-run -- vrg-validate`; expect the new tests to fail, because `queue_or_archive_kind`, `guard_check` and `state_reason` do not exist; a new test that passes here is not exercising the guard.
- [ ] **Step 3: Implement** the rules above.
- [ ] **Step 4: Update** the `sync` help and the security-records guide's sync section.
- [ ] **Step 4b: Refactor:** build `queue_or_archive_kind` on the existing `queue_kind`, `standing_kind_of` and `is_archive` rather than a parallel classifier. Tests stay green.
- [ ] **Step 5: Run** `vrg-container-run -- vrg-validate`; expect PASS.
- [ ] **Step 6: Commit** `vrg-commit --type feat --scope security-record --message "hold an advisory open while a queue item or a non-completed close is all that ended it"`, then report-ready.

### Task 4: One fail-closed planning hold rule (vergil-tooling, #3996)

Blocked by 2 (its hints name `--from`).

**Files:**

- Modify: `src/vergil_tooling/lib/queue_view.py` (new `advisory_hold`; `tracking_advisories`, `_tracking`, `planning_items` and `_held_note` use it)
- Modify: `src/vergil_tooling/lib/planning_close.py` (`find` uses `advisory_hold`; the skip reason)
- Modify: `docs/site/docs/guides/security-records.md` (the planning hold: one rule, unreadable advisories, unlabelled lines)
- Modify: `docs/site/docs/reference/dev/security-record.md` (the reproduced `held:` note, around line 394)
- Test: `tests/vergil_tooling/test_queue_view.py`, `tests/vergil_tooling/test_planning_close.py`, `tests/vergil_tooling/test_vrg_queue.py`

**Interfaces:**

- Produces: `queue_view.AdvisoryHold` (frozen dataclass): `tracking: tuple[str, ...]` (advisories whose `Tracking issue:` lines name the issue), `unreadable: tuple[str, ...]` (GHSA IDs a `security`-labelled body names that the listing does not return), `broken: str | None` (an unlabelled body line with nothing tracking the issue). `held` is `bool(tracking or unreadable)`.
- Produces: `queue_view.advisory_hold(ref: IssueRef, body: str, labels: set[str], advisories: Sequence[Advisory]) -> AdvisoryHold`, pure, the one rule. `planning_items` calls it with the per-org listing it already reads; `planning_close.find` with one listing of its own.

**Rules** (spec §6.3). `planning_items` shows a held item `in-progress` with the held note in place of `ready-to-close`; the note names each tracking GHSA, then each unreadable one as "cannot be read: re-run with a token that can read the repository's advisories, or have the human check it", and ends with `vrg-security-record retarget <GHSA> --from <M> --to <task>`. A `broken` hold is shown as `broken pair: …` in the note and does not demote the item. `planning_close.find` skips on `held` with the same wording.

- [ ] **Step 1: Write the failing tests**
  - `test_advisory_hold_tracking_line_holds`: a tracking advisory holds whatever the body or label.
  - `test_advisory_hold_unreadable_labelled_holds`: labelled body names a GHSA absent from the listing; `unreadable` has it, `held` is True (Review Focus 5).
  - `test_advisory_hold_unreadable_unlabelled_is_broken`: the same body without the label; `held` False, `broken` set (Review Focus 5).
  - `test_planning_items_holds_unreadable`: the queue shows the item `in-progress` with the cannot-be-read note.
  - `test_planning_close_skips_unreadable`: `find` returns a skip naming the GHSA and the cannot-be-read wording.
  - `test_held_note_names_from`: the note's command carries `--from <M>`.
- [ ] **Step 2: Run** `vrg-container-run -- vrg-validate`; expect the new tests to fail, because `advisory_hold` does not exist and the view holds on listing absence as not tracking; a new test that passes here is not exercising the new rule.
- [ ] **Step 3: Implement** `advisory_hold` and route both callers through it; keep `tracking_advisories` as a thin wrapper if other callers use it.
- [ ] **Step 4: Update** the security-records guide's planning-hold section, and the CLI reference's reproduced `held:` note to the new wording with `--from`.
- [ ] **Step 4b: Refactor:** remove the duplicate tracking logic between `tracking_advisories` and `_tracking`; keep `tracking_advisories` only as a wrapper if a caller outside this task still uses it. Tests stay green.
- [ ] **Step 5: Run** `vrg-container-run -- vrg-validate`; expect PASS.
- [ ] **Step 6: Commit** `vrg-commit --type fix --scope queue --message "make the planning hold one rule that fails closed on an unreadable advisory"`, then report-ready.

### Task 5: vergil-tooling release with Tasks 2–4 (deployment, .github)

Blocked by 2, 3, 4.

- **Precondition (human-attested):** a vergil-tooling release cut after Tasks 2, 3 and 4 merged is published, and its version is named in a comment.
- **Procedure (agent-safe):**
  1. Confirm the release notes list the PRs of Tasks 2, 3 and 4.
  2. Update the host's vergil-tooling to that release, then confirm `vrg-security-record retarget --help` shows `--from` and `--drop`, and `vrg-security-record sync --help` names who runs it.
  3. Run the read-only `vrg-epic-audit` preview and record its security-records summary line, which now carries the held count.
  4. **One-time check of advisories closed before the guard existed.** List the `closed` advisories (`vrg-security-record list --state closed`). For each tracked issue, read `vrg-gh issue view <N> --repo <repo> --json stateReason,parent` and the parent's labels. An advisory is **found** when a tracked issue is a queue item (live or archived parent) or closed with a reason other than completed, and no tracked issue carries a `Security record <GHSA> closed:` comment (the `close` command's). Give the human the found GHSA IDs in the terminal only, and record each on its advisory with `vrg-security-record note`.
  5. **The human decides each found advisory:** reopen it by retargeting it to an open fix task (a follow-up task filed under this epic if none exists; the next `sync` then moves it back to `draft`, since an open tracked issue always wins), or confirm the close with `vrg-security-record close <GHSA> --reason …` (the reason is posted publicly, so it stays neutral). The agent carries out each decision.
- **SUCCESS:** every check passes, recorded with the release version, and every found advisory is either reopened and tracked by a named fix task or confirmed closed. The public result comment says only that the one-time check is done, never a count. From here until Task 9, a retarget command without `--from` is refused; Tasks 7 and 8 are the next work.

### Task 6: Skill part 1 — step 0 check, wording rule, gate, not-planned stop, hold text (vergil-claude-plugin)

**Files:**

- Modify: `skills/epic-create/SKILL.md` (step 0; step 1's gate and not-planned close; the step 8 neutrality bullet moves to step 0; the states table, the latest-note description and step 10's fallback)
- Modify: `skills/epic-review/SKILL.md` (planning mode, against the same hold text)
- Modify: `skills/epic-retrospective/SKILL.md` (the wording rule for a security epic)
- Modify: `docs/paad-review-doctrine.md` (§2.7, the rejected-finding record)

This is skill text with no tests in the repo; `vrg-validate`'s Markdown checks prove its form.

- [ ] **Step 1: Step 0 security check** (spec §6.4). A run is a security run when the planning issue carries `security`, has a `Security advisory:` line, or `vrg-security-record list` shows its ref under `tracks:`. Any disagreement between the three is a broken pair: stop and ask the human. A security run records a `Security run: yes` marker line, alone on its line, in its step 0 progress comment; it is never cleared. A resumed run reads the latest trusted marker before posting anything, so it stays a security run after step 8's retargets have removed the label, the line and the tracking. When the marker is present, the three-way check and its broken-pair stop are skipped: the run resumes at its recorded stage, and an interrupted step 8 re-runs its identical retargets.
- [ ] **Step 2: The public-wording rule at step 0**, listing every public artifact spec §6.4 names, with specifics going to the advisory through `vrg-security-record note`. Remove the step 8 bullet it replaces and point step 8 at step 0.
- [ ] **Step 3: The human gate when the design is approved**, before any step 1 outcome publishes anything: the single-task exit, the not-planned exit and the brainstorm-approved comment before step 2. The question, the bare brainstorm-approved comment of a security run, and the neutral "paused for a design-placement decision" stop on no.
- [ ] **Step 4: The not-planned close** (spec §6.5): for a security run the agent does not close; it stops and names the two paths in order (retarget to a fix task; or close the planning issue as not planned with a neutral comment, then `vrg-security-record close`).
- [ ] **Step 5: The hold as it is today** (spec §6.6) in the states table, the latest-note description and step 10's fallback; then check `epic-review`'s planning mode against the same text and close any gap.
- [ ] **Step 5b: `epic-retrospective`:** an epic is a security epic when its planning issue carries a trusted `Security run: yes` marker or any of its tasks carries `security`; its retrospective follows the same public-wording rule, with a link to epic-create step 0.
- [ ] **Step 5c: Doctrine §2.7, rejected findings** (a human decision, made in this epic's design review): the record is a plain `vrg-security-record create`, then its intake issue is closed as not planned, then `vrg-security-record close <GHSA> --reason "<neutral reason>"`. The sync guard (Task 3) leaves a closed advisory alone, where it would hold one whose only tracked issue is a not-planned queue item.
- [ ] **Step 6: Run** `vrg-container-run -- vrg-validate`; expect PASS.
- [ ] **Step 7: Commit** `vrg-commit --type feat --scope epic-create --message "check for a security run at step 0 and gate its first publication on the human"`, then report-ready.

### Task 7: Skill part 2 — `--from` hand-offs and the unreadable-advisory note (vergil-claude-plugin)

Blocked by 5, 6.

**Files:**

- Modify: `skills/epic-create/SKILL.md` (step 1's single-task close; step 8's retarget; the states table, the latest-note description and step 10's held bullet)
- Modify: `skills/epic-review/SKILL.md` (planning mode, against the new hold wording)
- Modify: `docs/site/docs/skills/index.md` (its retarget command, around line 111)

- [ ] **Step 1: Step 1, single task** (spec §6.5): after the human gate (Task 6 Step 3), file the task, retarget each advisory to it with `--from <M>`, then close the planning issue.
- [ ] **Step 2: Step 8:** one retarget per advisory, once, after the last fix task is filed, with `--from <M>` and every fix task; GHSA IDs from `vrg-security-record list` (`tracks:` names M), never the body lines; a later fix task added `--from` a current task with the complete set; on a failure partway through (a step error), fix the cause and re-run the identical command; on a refusal (source or drop check), stop and ask the human, and never add `--drop` without the human's decision. Replace "the scheduled sweep closes an advisory" with spec §6.5's account.
- [ ] **Step 3: The new hold wording everywhere Task 6 placed hold text** (the states table, the latest-note description and step 10's held bullet): the cannot-be-read note and its next step (spec §6.3) and Task 4's `held:` wording with `--from`; then re-check `epic-review`'s planning mode against it.
- [ ] **Step 4: Update** the skills index's retarget command.
- [ ] **Step 5: Run** `vrg-container-run -- vrg-validate`; expect PASS.
- [ ] **Step 6: Commit** `vrg-commit --type feat --scope epic-create --message "hand advisories on with retarget --from"`, then report-ready.

### Task 8: Intake page names `--from` (docs)

Blocked by 5.

**Files:**

- Modify: `docs/site/docs/management/intake.md` (the security-item paragraph, around line 104)

- [ ] **Step 1:** The hand-retarget sentence names the full command, `vrg-security-record retarget <GHSA> --from <item> --to <task>`, and links the guide's retarget section.
- [ ] **Step 2: Run** `vrg-container-run -- vrg-validate`; expect PASS.
- [ ] **Step 3: Commit** `vrg-commit --type docs --scope intake --message "name retarget's source in the security-item hand-off"`, then report-ready.

### Task 9: Plugin release with Task 7 (deployment, .github)

Blocked by 7.

- **Precondition (human-attested):** a vergil-claude-plugin release cut
  after Tasks 6 and 7 merged is published, and its version is named in a
  comment.
- **Procedure (agent-safe):**
  1. Confirm the release notes list the PRs of Tasks 6 and 7.
  2. Update the installed plugin (`vrg-host plugins`) and confirm the
     installed `epic-create` skill's step 8 names `--from`.
- **SUCCESS:** both checks pass, recorded with the release version. The
  `--from` window is closed.
