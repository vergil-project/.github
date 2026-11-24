# sub-issue-capacity — implementation plan

- **Epic:** vergil-project/.github#1609
- **Planning issue:** vergil-project/vergil-tooling#3840 (review history is in its comments)
- **Status:** draft

> **For agentic workers:** each task is one issue and one PR in the repo named
> in its heading, worked with `issue-implement`. Operational tasks
> (deployment, validation) are worked with `issue-deploy` and
> `issue-validate`. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** An epic of any kind grows past GitHub's 100-sub-issue limit as a
chain of containers behind one chain layer, so no caller ever meets the limit
and no item lands outside its epic.

**Architecture:**

- **vergil-tooling** adds `lib/epic_chain.py`, which owns walking, growing,
  healing and closing a chain. `lib/epics.py`'s public readers become
  chain-aware, its raw writers become private, and every caller that links,
  moves, unlinks or reads children goes through the chain layer.
- Two behaviour changes the chain depends on ship first, on their own:
  **closed is closed** (no tooling path reopens an epic) and an **atomic
  `vrg-epic-move`**.
- The audit and the epic sweep report and heal chain drift, and the sweep's
  queue cap becomes a nudge instead of a refusal.

**Tech Stack:** Python 3 and pytest, GitHub GraphQL sub-issue API
(vergil-tooling); Markdown (site docs).

**Spec:** [`spec.md`](spec.md) in this directory. Executors read both.

## Global Constraints

- **Container capacity:** a container holds at most **99 items**; the 100th
  slot of a non-tail container is its continuation (spec §4.1).
- **The count is all children, open and closed**, read from the sub-issue
  total, less one when the container holds a continuation (spec §4.3).
- **Continuation identity:** an issue in the head's repo, labels `epic` and
  `continuation` and no kind label, title `<head title> (continuation <k>)`
  with `k` from 2, body lines `Continues: <owner>/<repo>#<head>` and
  `Container: <k>`, each alone on its line (spec §4.1).
- **`Container:` and the title number are display only;** position is the
  native chain's, and no reader checks the number (spec §4.1).
- **Trust:** a continuation is followed only when it has the label, a trusted
  body, a `Continues:` line naming the head being walked, and a native link
  to the container before it; anything else is chain drift, raised by name
  (spec §4.1).
- **First fit:** link into the first container in chain order with fewer than
  99 items; roll over only when every container holds 99 (spec §4.3).
- **Rollover order:** find-or-create the continuation, link it into the
  tail's last slot, then link the item (spec §4.3).
- **Closed is closed:** no tooling path reopens an epic; `link_into` and
  `move_into` refuse a closed head by name (spec §4.2).
- **`close_chain` closes the head first,** then every continuation
  (spec §4.6).
- **No raw writer or raw `parent`/`subIssues` selection outside
  `lib/epics.py` and `lib/epic_chain.py`** once Task 11 lands (spec §4.2).
- **The sweep keeps its cap of 95 open children** as the review nudge, but
  no longer refuses at it (spec §4.7).
- **Archives are unchanged:** `ensure_writable_standing_archive` and its
  segment cap of 95 are not modified (spec §2, non-goals).
- **No silent failures:** every refusal, heal and rollover prints one line
  saying what it did (spec §5).

## Review Focus

1. **A continuation's ref pasted as `--epic`** (copied from GitHub's UI). It
   resolves to the head with a line saying so, and every kind check runs
   against the head. Pinned in Task 6
   (`test_resolve_epic_ref_maps_continuation_to_head`).
2. **A head already at 100 children with no continuation** (the state #438
   was in). The first link heals it and rolls over, and no item is left
   unlinked. Pinned in Task 5c
   (`test_link_into_heals_head_at_100_without_continuation`).
3. **The last open task of an epic sits in a continuation.** Closing a task
   in the head does not roll the epic up; closing that last task does, and
   closes the whole chain. Pinned in Task 6
   (`test_rollup_waits_for_open_item_in_continuation`).
4. **A continuation closed by hand in an open chain.** Readers still list its
   items, writers skip it, the audit notes it, and nothing reopens it.
   Pinned in Task 5b (`test_link_into_skips_closed_continuation`) and
   Task 9a (`test_audit_notes_closed_continuation_in_open_chain`).
5. **A create-then-link command at the depth limit.** `vrg-issue-create`
   refuses before creating, so no issue is orphaned. Pinned in Task 7
   (`test_issue_create_depth_refusal_creates_nothing`).

## Delivery table

Each line gives the task, its repo and kind, what it delivers, and what it
is blocked by. All implementation tasks are in vergil-tooling.

- **1** (.github, task #1610): spec and plan, this PR.
- **2** (vergil-tooling): closed is closed: no reopen in `add_child`,
  `intake_route` or `proxies.create_proxy`; `ensure_standing_epic` refuses a
  closed one; the audit's reopen fix removed.
- **3** (vergil-tooling): `vrg-epic-move` moves atomically with
  `reparent_child`.
- **4** (.github, validation): a live probe of GitHub's nesting limit and its
  error texts, fixing the depth constant.
- **5a** (vergil-tooling): `epic_chain` read side, the `continuation` label,
  the chain-drift error and the in-memory sub-issue fake.
- **5b** (vergil-tooling): `epic_chain` write side: first-fit `link_into`,
  `move_into`, `unlink_from`, rollover, find-or-create with orphan adoption,
  the depth refusal, the capacity pre-check and the rollover messages.
  Blocked by 5a, 4.
- **5c** (vergil-tooling): race heals (limit error, full tail, duplicate
  continuations) and `close_chain`. Blocked by 5b.
- **6** (vergil-tooling): `epics.py` chain-aware readers, ref resolution,
  rollup, drain and the lookups that must not find a continuation. Blocked
  by 2, 5c.
- **7** (vergil-tooling): the CLI writers (`vrg-epic-link`,
  `vrg-issue-create`, `vrg-intake-create`, `vrg-security-record`,
  `vrg-epic-move`) and `ops_report` through the chain layer, with the
  capacity pre-check. Blocked by 3, 6.
- **8** (vergil-tooling): `intake_route` through the chain layer, with
  transfer settling moved into it. Blocked by 6.
- **9a** (vergil-tooling): audit read side: continuation skips, chain-aware
  children and `misplaced_adhoc`, drift and orphan reporting, container
  count. Blocked by 6.
- **9b** (vergil-tooling): sweep writes: the orphan pass through the chain
  with the nudge, the heals, orphan adoption with a rollup re-run, and
  `close_drift`/`close_proxies` through `close_chain`. Blocked by 9a.
- **10** (vergil-tooling): the remaining readers (`queue_view`, `roadmap`,
  `planning_close`, `security_records`, `merge_review/request`,
  `vrg-submit-pr`, `epic_docs`). Blocked by 6.
- **11** (vergil-tooling): the raw layer made private, with a source-scan
  test that keeps it so. Blocked by 7, 8, 9b, 10.
- **12** (.github, deployment): the vergil-tooling release carrying 2–11,
  installed on the host and every VM. Blocked by 11.
- **13** (.github, validation): #1384's 14 tasks re-homed into its chain, the
  first live rollover of a development epic. Blocked by 12.

### Bookends

- **doc-review** (vergil-tooling, `Plan task: doc-review`): blocked by 13.
  Its sweep covers vergil-tooling's
  `docs/site/docs/standards/github-issues.md` (the 100-sub-issue cap, the
  orphan pass, archives' 95 cap beside the chain) and
  `docs/site/docs/reference/` CLI pages for the changed commands; the docs
  repo's summary of the epic model; and any vergil-claude-plugin skill that
  tells an agent what to do at the cap (`epic-implement`,
  `issue-implement`), each a spawned per-repo doc task.
- **retrospective** (.github, `Plan task: retrospective`): blocked by
  everything else.

Notes:

- **No fixture runs** and no `dev-` image validation: the change touches no
  language support, container image, shared workflow or release tooling.
- **No follow-on brainstorm task:** no enabling chain is known. Collecting
  empty continuations (spec §2, non-goals) is accrued as an intake item only
  if a real chain ever needs it.
- **Develop stays correct between tasks.** Until Task 11, the raw writers
  remain public and keep working; no chain exists before the release, so a
  reader made chain-aware before its writer is harmless. Each task must
  still pass the full suite on its own.
- **One release.** Task 12 installs a release carrying every implementation
  task. Tasks 2 and 3 may reach an earlier release by the normal cadence;
  that is fine, as each stands alone.
- **Open question Q1 (for the human): a closed continuation with no
  successor.** Spec §4.2 says writers never link into a closed
  continuation, and that when its successor is missing, "rollover links the
  next continuation into the last open container instead". But a closed
  continuation fills the 100th slot of the container before it, so when
  every open container already holds 99 items there is no open container
  with a free slot to link the next continuation into. The spec does not
  settle this. Options: (a) the writer refuses, naming the closed
  continuation and saying a person may reopen it or unlink it (closed is
  closed holds; the epic is stuck until a human acts); (b) the writer
  unlinks the closed, empty continuation to free the slot, refusing as in
  (a) if it still holds items; (c) linking into a closed container is
  allowed for the continuation link alone. Task 5b's test is written once
  the human rules, and the spec is corrected to match.
- **Docs travel with behaviour.** Each task that changes documented
  behaviour updates `docs/site/docs/standards/github-issues.md` in the same
  PR (named in its Files); the doc-review bookend checks the whole.

---

### Task 2: closed is closed (vergil-tooling)

Spec §4.2 (closed is closed), goal 8.

**Files:**

- Modify: `src/vergil_tooling/lib/epics.py` (`add_child`, `ensure_standing_epic`, `_find_epic_by_title`)
- Modify: `src/vergil_tooling/lib/intake_route.py:1545-1560` (the reopen before a re-parent)
- Modify: `src/vergil_tooling/lib/proxies.py:330-375` (`create_proxy`), `src/vergil_tooling/bin/vrg_epic_create.py:189` (the reopened report)
- Modify: `src/vergil_tooling/lib/epic_audit.py` (remove `reopen_epics_with_open_children`), `src/vergil_tooling/bin/vrg_epic_audit.py:300-330`
- Modify: `src/vergil_tooling/bin/vrg_epic_link.py` (docstring: the reopen-on-late-child rule)
- Modify: `docs/site/docs/standards/github-issues.md` (every mention of reopen-on-late-child and the audit's reopen fix)
- Test: `tests/vergil_tooling/test_epics.py`, `test_intake_route.py`, `test_proxies.py`, `test_epic_audit.py`, `test_vrg_epic_audit.py`

**Interfaces:**

- Produces: `epics.ClosedEpicError(ValueError)` with attribute `epic: IssueRef`
  and message `epic <slug> is closed; closed epics are not reopened by the
  tooling`. Later tasks raise it from `link_into` and `move_into`.
- Produces: `epics.ClosedStandingEpicError(ClosedEpicError)`, raised by
  `ensure_standing_epic` when no open epic has the canonical title and a
  closed one does; its message adds `a person may reopen it by hand`.

- [ ] **Step 1: Write the failing tests**

  - `test_add_child_refuses_closed_epic`: with the epic's state `CLOSED`,
    `add_child` raises `ClosedEpicError` naming `org/.github#40`, and no
    `issue reopen` and no `addSubIssue` call is made. Replaces
    `test_add_child_reopens_closed_epic_before_linking`.
  - `test_ensure_standing_epic_reuses_open_beside_closed`: an open and a
    closed epic share the canonical title; the open one is returned, nothing
    is created.
  - `test_ensure_standing_epic_refuses_when_only_closed`: only a closed one
    exists; raises `ClosedStandingEpicError` naming it; nothing is created.
  - `test_ensure_standing_epic_creates_when_none`: neither exists; created as
    today.
  - `test_route_refuses_closed_target_epic` (intake_route): routing into a
    closed epic exits non-zero naming it; no `issue reopen` call.
  - `test_create_proxy_refuses_closed_proxy` (proxies): a closed proxy whose
    `Primary:` line names the primary raises `ClosedEpicError`; no reopen.
  - `test_audit_reports_closed_epic_open_child_without_reopen`
    (vrg-epic-audit `--close`): the violation is printed for the human and
    no `issue reopen` call is made.

- [ ] **Step 2: Run them to verify they fail**

  Run: `vrg-container-run -- pytest tests/vergil_tooling/test_epics.py tests/vergil_tooling/test_intake_route.py tests/vergil_tooling/test_proxies.py tests/vergil_tooling/test_vrg_epic_audit.py -k "closed or reopen" -v`
  Expected: the new tests FAIL (reopen still happens).

- [ ] **Step 3: Implement**

  `add_child` checks the state and raises; `_find_epic_by_title` gains a
  `states` parameter (`"open"` default, `"all"` for `ensure_standing_epic`);
  remove `reopen_epics_with_open_children` and its CLI wiring, keeping
  `closed_epic_open_child` as a report. `ProxyResult.reopened` is removed.

- [ ] **Step 4: Run the tests and the full gate**

  Run: `vrg-container-run -- vrg-validate`
  Expected: PASS.

- [ ] **Step 5: Commit**: `vrg-commit --type feat --scope epics --message "make closed epics final: refuse to link, move or reuse a closed epic instead of reopening it"`.

### Task 3: atomic `vrg-epic-move` (vergil-tooling)

Spec §4.5 (`vrg-epic-move` stops orphaning).

**Files:**

- Modify: `src/vergil_tooling/bin/vrg_epic_move.py:90-93`
- Test: `tests/vergil_tooling/test_vrg_epic_move.py`

**Interfaces:**

- Consumes: `epics.reparent_child(new_parent, task)`; `epics.add_child` for a
  task with no current parent.

- [ ] **Step 1: Write the failing tests**

  - `test_move_uses_one_reparent`: a task under epic A moved to B makes one
    `addSubIssue` call with `replaceParent: true` and no `removeSubIssue`.
  - `test_failed_move_leaves_task_in_place`: the reparent raises; the command
    exits 1 with the error, and no `removeSubIssue` was called.
  - `test_move_unparented_task_links_it`: no current parent → `add_child`.

- [ ] **Step 2: Run them to verify they fail**

  Run: `vrg-container-run -- pytest tests/vergil_tooling/test_vrg_epic_move.py -v`
  Expected: the first two FAIL.

- [ ] **Step 3: Implement**: replace the remove-then-add with `reparent_child`
  when `current is not None`.

- [ ] **Step 4: Validate**: `vrg-container-run -- vrg-validate` passes.

- [ ] **Step 5: Commit**: `vrg-commit --type fix --scope epic-move --message "move a task between epics in one atomic re-parent"`.

### Task 4: probe GitHub's nesting limit (validation, .github)

Spec §4.5 (the depth headroom), §4.4 (GitHub's limit error). No blockers.

- **Precondition (machine probe):** `vrg-whoami --mode` prints `user`.
- **Procedure:**
  1. In vergil-project/.github, create ten issues titled `Probe: sub-issue
     depth <n>` (n = 1…10), each body saying it is a throwaway for epic
     vergil-project/.github#1609 Task 4.
  2. Link each `n+1` under `n` with `vrg-epic-link`, in order, until a link
     fails. Record the last depth that succeeded, counting the top issue as
     depth 1, and GitHub's error text verbatim.
  3. Capture the 100-child error text: if any open issue in the org then has
     100 sub-issues (`vrg-gh issue view <N> --json subIssuesSummary`), try
     linking probe issue 10 under it and record the error; a failed link
     writes nothing. If none has 100, record that the text is unobserved.
  4. Unlink every probe issue and close each as not planned.
- **SUCCESS:** a result comment carrying the lines `Max depth: <d>` (levels
  including the top issue), `Depth error: <text>` and `Limit error: <text>`
  (or `Limit error: unobserved`). Task 5b reads them; `unobserved` means 5b
  matches GitHub's documented wording and Task 13 confirms it live.

### Task 5a: chain read side and the sub-issue fake (vergil-tooling)

Spec §4.1, §4.2 (`chain_of`, `head_of`, `chain_child_states`,
`chain_contains`).

**Files:**

- Create: `src/vergil_tooling/lib/epic_chain.py`
- Create: `tests/vergil_tooling/subissue_fake.py`
- Modify: `src/vergil_tooling/data/labels.json` (the `continuation` label)
- Modify: `src/vergil_tooling/lib/epics.py` (`CONTINUATION_LABEL`, `is_continuation`)
- Test: `tests/vergil_tooling/test_epic_chain.py`

**Interfaces:**

- Produces in `epics`: `CONTINUATION_LABEL = "continuation"`;
  `is_continuation(labels: Collection[str]) -> bool`.
- Produces in `epic_chain`:
  - `ITEM_CAP = 99`
  - `class ChainDriftError(RuntimeError)` with `container: IssueRef` and
    `rule: str` (one of `label`, `untrusted-body`, `continues-mismatch`,
    `not-native-child`, `two-continuations`), message naming both.
  - `@dataclass(frozen=True) class Container: ref: IssueRef; state: str;
    item_count: int; continuation: IssueRef | None`
  - `containers(head: IssueRef) -> list[Container]` (head first)
  - `chain_of(head: IssueRef) -> list[IssueRef]`
  - `head_of(container: IssueRef) -> IssueRef`
  - `chain_child_states(head: IssueRef) -> list[epics.ChildState]`
    (container order, continuations excluded)
  - `chain_contains(head: IssueRef, task: IssueRef) -> bool`
  - `parse_continues(body: str) -> IssueRef | None`
- Produces in the fake: `FakeSubIssueGraph` with `add_issue(ref, *, title,
  labels=(), body="", state="OPEN") -> IssueRef`, `link(parent, child)`,
  `close(ref)`, and `install(monkeypatch)`, which answers `github.graphql`
  for `epics`' sub-issue, parent and mutation queries and `github.read_output`
  for node ids, state, title and labels. `addSubIssue` fails at 100 children
  with `LIMIT_ERROR` and past `MAX_DEPTH` with `DEPTH_ERROR` (module
  constants, from Task 4's result); a trusted body is served through
  `trust_stubs.FakeItems`.

- [ ] **Step 1: Write the fake and the failing tests**

  - `test_chain_of_single_head`: a head with 3 items → `[head]`.
  - `test_chain_of_follows_continuations`: head (99 items + C2), C2 (99 + C3),
    C3 (5) → `[head, C2, C3]`; `chain_child_states(head)` has 203 items, no
    continuation among them, head's items first.
  - `test_head_of_continuation_walks_parents`: `head_of(C3) == head`;
    `head_of(head) == head`.
  - `test_continuation_with_wrong_continues_is_drift`: C2's `Continues:`
    names another head → `ChainDriftError(rule="continues-mismatch")`.
  - `test_continuation_untrusted_body_is_drift`: rule `untrusted-body`.
  - `test_two_continuations_under_one_container_is_drift`: rule
    `two-continuations` (5c heals it; readers raise).
  - `test_closed_continuation_is_walked`: C2 closed under an open head; its
    items are listed and no error is raised.
  - `test_item_count_excludes_continuation`: head with 99 items + C2 →
    `item_count == 99`; read from the sub-issue total, not a listing.
  - `test_chain_contains_item_in_continuation`.

- [ ] **Step 2: Run them to verify they fail**

  Run: `vrg-container-run -- pytest tests/vergil_tooling/test_epic_chain.py -v`
  Expected: FAIL (module does not exist).

- [ ] **Step 3: Implement** the read side. `head_of` on an issue with no
  native parent and a trusted `Continues:` line returns that head and prints
  `chain drift: <slug> is an orphaned continuation of <head>` to stderr
  (adoption is 5b's).

- [ ] **Step 4: Validate**: `vrg-container-run -- vrg-validate` passes.

- [ ] **Step 5: Commit**: `vrg-commit --type feat --scope epic-chain --message "add the chain layer's read side and the continuation label"`.

### Task 5b: chain write side and rollover (vergil-tooling)

Spec §4.2 (`link_into`, `move_into`, `unlink_from`), §4.3, §4.5, §4.7
(rollover messages). Blocked by 5a and Task 4.

**Files:**

- Modify: `src/vergil_tooling/lib/epic_chain.py`
- Test: `tests/vergil_tooling/test_epic_chain.py`

**Interfaces:**

- Consumes: Task 5a's read side; `epics.ClosedEpicError` (Task 2);
  `epics.add_child`/`reparent_child`/`remove_child` (still public until 11).
- Produces:
  - `MAX_DEPTH: int` (Task 4's `Max depth`)
  - `class ChainDepthError(RuntimeError)`: message `epic <slug> cannot grow:
    <items> items in <containers> containers, at GitHub's nesting limit of
    <MAX_DEPTH>; split the epic`.
  - `check_capacity(head: IssueRef) -> None`: raises `ClosedEpicError` or
    `ChainDepthError` when a link would fail for either; reads only.
  - `link_into(head: IssueRef, task: IssueRef) -> IssueRef` (the container
    linked into)
  - `move_into(head: IssueRef, task: IssueRef) -> IssueRef`
  - `unlink_from(head: IssueRef, task: IssueRef) -> None` (raises
    `ValueError` naming both when the task is not in the chain)
  - `find_or_create_continuation(head: IssueRef, tail: IssueRef) -> IssueRef`
  - `find_orphans(head: IssueRef) -> list[IssueRef]` (open
    `continuation`-labelled issues in the head's repo whose trusted
    `Continues:` names *head* and that `chain_of(head)` does not reach)

- [ ] **Step 1: Write the failing tests**

  - `test_link_into_first_fit`: head 99 items + C2 with 40 → the item goes
    into C2; head 98 items + C2 with 40 → into the head.
  - `test_link_into_reuses_freed_slot_before_growing`: head 99 + C2 99, then
    one head item moved out → the next link goes into the head, no C3.
  - `test_link_into_rolls_over_at_99`: head 99 → C2 created with the exact
    title, labels and body lines, linked as the head's 100th child, then the
    item linked into C2 (assert the call order).
  - `test_link_into_skips_closed_continuation`: head 99 items + C2 closed
    (5 items) + C3 open (10 items) → the item goes into C3; C2 is not
    written to and stays closed.
  - `test_link_into_closed_continuation_without_successor`: head 99 items +
    C2 closed with no successor. The expected behaviour is **open question
    Q1** (below the delivery table); the test is written once it is ruled.
  - `test_link_into_refuses_closed_head`: `ClosedEpicError`, nothing written.
  - `test_find_or_create_reuses_existing_child_continuation` and
    `test_find_or_create_adopts_orphan`: an orphan naming the head is linked
    into the tail's last slot and its `Container:` line and title rewritten;
    nothing created.
  - `test_orphan_failing_trust_is_not_adopted`: a `continuation`-labelled
    issue with an untrusted body raises `ChainDriftError`; a new
    continuation is not created over it either.
  - `test_depth_refusal_before_any_write`: a chain at `MAX_DEPTH` with every
    container full → `ChainDepthError` with the exact message; no issue
    created, nothing linked.
  - `test_github_depth_error_is_reworded`: the fake's `DEPTH_ERROR` comes
    first → `ChainDepthError`.
  - `test_development_rollover_is_loud`: stdout has `epic <head> rolled over
    into continuation <C2> at 99 children; consider splitting it`, and the
    same text is commented on the head.
  - `test_queue_rollover_prints_review_nudge`: an intake head → stdout has
    `queue <head> rolled over into continuation <C2>; run epic-review
    intake`; no comment on the head.
  - `test_move_into_is_one_reparent` and `test_unlink_from_finds_container`.

- [ ] **Step 2: Run them to verify they fail**

  Run: `vrg-container-run -- pytest tests/vergil_tooling/test_epic_chain.py -v`
  Expected: the new tests FAIL.

- [ ] **Step 3: Implement** the writers. Depth is the head's own depth
  (`native_parent_of` walk) plus the container count; the check runs before
  find-or-create.

- [ ] **Step 4: Validate**: `vrg-container-run -- vrg-validate` passes.

- [ ] **Step 5: Commit**: `vrg-commit --type feat --scope epic-chain --message "link, move and unlink through the chain with first-fit rollover, orphan adoption and the depth refusal"`.

### Task 5c: race heals and `close_chain` (vergil-tooling)

Spec §4.4, §4.6 (`close_chain`). Blocked by 5b.

**Files:**

- Modify: `src/vergil_tooling/lib/epic_chain.py`
- Test: `tests/vergil_tooling/test_epic_chain.py`

**Interfaces:**

- Produces:
  - `LINK_RETRIES = 3`
  - `heal_full_tail(head: IssueRef, *, own_item: IssueRef | None = None) -> bool`
  - `heal_duplicate_continuations(head: IssueRef) -> bool`
  - `close_chain(head: IssueRef) -> list[IssueRef]` (closed, head first)
  - `class ChainLimitError(RuntimeError)`: message naming GitHub's limit of
    100, the container and the item, raised when retries run out.

- [ ] **Step 1: Write the failing tests**

  - `test_link_into_heals_head_at_100_without_continuation`: the head holds
    100 items and no continuation; `link_into` creates C2, moves the head's
    newest item into it with `replaceParent`, links C2 into the freed slot,
    then links the new item (into C2), and prints one heal line. At no step
    is any item without a parent (assert on the fake's graph after every
    mutation).
  - `test_full_tail_heal_moves_own_item`: after the writer's own link fills
    slot 100, the moved item is its own.
  - `test_limit_error_retries_then_succeeds`: the first link raises the
    fake's `LIMIT_ERROR` (a racing writer); the writer re-reads, rolls over
    and succeeds.
  - `test_limit_error_retries_exhausted_names_limit`: `ChainLimitError` after
    `LIMIT_ERROR` three times, naming 100, the container and the item.
  - `test_duplicate_continuations_keep_lower`: C2a (#201) and C2b (#205)
    under the head; items of #205 moved to #201, #205 unlinked, then closed
    as not planned with a comment naming #201 — assert that order.
  - `test_close_chain_head_first`: closes head, C2, C3 in that order.
  - `test_close_chain_skips_already_closed`.

- [ ] **Step 2: Run them to verify they fail**

  Run: `vrg-container-run -- pytest tests/vergil_tooling/test_epic_chain.py -k "heal or limit or duplicate or close_chain" -v`
  Expected: FAIL.

- [ ] **Step 3: Implement**: `link_into` reads the tail back after each link
  and calls `heal_full_tail` and `heal_duplicate_continuations`; a
  `LIMIT_ERROR` triggers the re-read and retry, never a raw surface.

- [ ] **Step 4: Validate**: `vrg-container-run -- vrg-validate` passes.

- [ ] **Step 5: Commit**: `vrg-commit --type feat --scope epic-chain --message "heal racing rollovers and close a chain head first"`.

### Task 6: chain-aware `epics.py` (vergil-tooling)

Spec §4.1 (lookups, ref resolution), §4.2 (public readers), §4.6. Blocked by
2 and 5c.

**Files:**

- Modify: `src/vergil_tooling/lib/epics.py` (`child_states`,
  `child_states_with_blockers`, `native_children`, `all_children_closed`,
  `parent_of`, `native_parent_of`, `qualify_epic_ref`, `resolve_epic_ref`,
  `rollup`, `drain_standing_child`, `apply_standing_drain`,
  `_find_epic_by_title`, `standing_kind_of`, `queue_kind`)
- Modify: `docs/site/docs/standards/github-issues.md` (a new "Chained epics"
  section: the chain, the label and lines, first fit, the messages, the
  depth refusal)
- Test: `tests/vergil_tooling/test_epics.py`

**Interfaces:**

- Consumes: `epic_chain.chain_child_states`, `head_of`, `find_orphans`,
  `find_or_create_continuation`, `close_chain`.
- Produces: the readers keep their signatures and answer for the chain. The
  per-container raw reads are renamed `_container_child_states(container)`
  and `_container_parent(task)` and used only by `epic_chain`.
  `resolve_epic_ref` prints `<continuation> is a continuation of <head>;
  using <head>` to stderr when it maps one.

- [ ] **Step 1: Write the failing tests** (on `FakeSubIssueGraph`)

  - `test_resolve_epic_ref_maps_continuation_to_head`: `--epic` naming C2
    resolves to the head with the stderr line; a kind check on the result
    sees the head's labels.
  - `test_parent_of_item_in_continuation_is_head`.
  - `test_child_states_spans_chain`.
  - `test_rollup_waits_for_open_item_in_continuation`: all head items closed,
    one C2 item open → closing a head item does not close the epic; closing
    the C2 item closes head then C2.
  - `test_rollup_continuation_close_is_noop`: closing C2 itself rolls nothing.
  - `test_rollup_adopts_orphan_before_judging`: an orphan of the head holding
    an open item → adopted, epic stays open.
  - `test_rollup_refuses_with_unadoptable_orphan`: an orphan with an
    untrusted body → no close, the orphan reported.
  - `test_drain_from_continuation_uses_head_kind`: a closed item in an
    intake chain's C2 drains to the intake archive.
  - `test_standing_lookup_never_finds_continuation`: an intake head and its
    continuation (same title prefix) → `ensure_standing_epic` returns the
    head; `standing_kind_of(continuation labels)` is `None`.

- [ ] **Step 2: Run them to verify they fail**

  Run: `vrg-container-run -- pytest tests/vergil_tooling/test_epics.py -v`
  Expected: the new tests FAIL; the existing ones still pass.

- [ ] **Step 3: Implement.** The reflink fallback in `child_states` and
  `parent_of` is kept, applied to the head.

- [ ] **Step 4: Validate**: `vrg-container-run -- vrg-validate` passes.

- [ ] **Step 5: Commit**: `vrg-commit --type feat --scope epics --message "read epics, parents and rollups across the whole chain"`.

### Task 7: the CLI writers through the chain (vergil-tooling)

Spec §4.2 (writer callers), §4.5 (create-then-link checks first,
`vrg-epic-move`). Blocked by 3 and 6.

**Files:**

- Modify: `src/vergil_tooling/bin/vrg_epic_link.py:61`,
  `vrg_issue_create.py:189`, `vrg_intake_create.py:75`,
  `vrg_security_record.py:237,343,389` (`_find_resumed_issue` included),
  `vrg_epic_move.py`
- Modify: `src/vergil_tooling/lib/ops_report.py:300-302`
- Test: the matching `tests/vergil_tooling/test_vrg_*.py`, `test_ops_report.py`

**Interfaces:**

- Consumes: `epic_chain.check_capacity`, `link_into`, `move_into`,
  `chain_contains`, `chain_child_states`.

- [ ] **Step 1: Write the failing tests**

  - `test_issue_create_depth_refusal_creates_nothing`: `check_capacity`
    raises `ChainDepthError` → exit 1 with its message, and
    `github.create_issue` is never called.
  - `test_issue_create_closed_epic_refused_before_create`.
  - `test_intake_create_rolls_over_full_queue`: an intake head at 99 → the
    item lands in C2 and the queue nudge is printed.
  - `test_security_record_resume_finds_item_in_continuation`: the resumed
    intake issue sits in C2 and is found.
  - `test_epic_link_into_full_epic_rolls_over`.
  - `test_epic_move_into_chain`: moved with `move_into`; `already under epic`
    is decided with `chain_contains`.
  - `test_ops_report_links_through_chain`.

- [ ] **Step 2: Run them to verify they fail**

  Run: `vrg-container-run -- pytest tests/vergil_tooling/test_vrg_issue_create.py tests/vergil_tooling/test_vrg_intake_create.py tests/vergil_tooling/test_vrg_security_record.py tests/vergil_tooling/test_vrg_epic_link.py tests/vergil_tooling/test_vrg_epic_move.py tests/vergil_tooling/test_ops_report.py -v`
  Expected: the new tests FAIL.

- [ ] **Step 3: Implement**: `check_capacity` runs before every create; the
  link-failure hint that suggested `vrg-epic-move` names the chain error
  instead.

- [ ] **Step 4: Validate**: `vrg-container-run -- vrg-validate` passes.

- [ ] **Step 5: Commit**: `vrg-commit --type feat --scope epics --message "link, move and create through the chain layer, checking capacity before creating"`.

### Task 8: `intake_route` through the chain (vergil-tooling)

Spec §4.2 (settling after a transfer, writer callers). Blocked by 6.

**Files:**

- Modify: `src/vergil_tooling/lib/intake_route.py` (lines 924, 947, 1218,
  1549-1556, 1613-1615; `_observe`, `_settle_parent`, `_clear_duplicate`,
  `_relink_after_failed_transfer`)
- Modify: `src/vergil_tooling/lib/epic_chain.py` (settling operations)
- Test: `tests/vergil_tooling/test_intake_route.py`, `test_epic_chain.py`

**Interfaces:**

- Produces in `epic_chain`:
  - `observe_parent(head: IssueRef, task: IssueRef) -> tuple[IssueRef | None, list[IssueRef]]`
    (the task's parent field, and the containers of *head*'s chain whose
    listing holds it)
  - `clear_stale_listing(head: IssueRef, task: IssueRef, keep: IssueRef) -> None`

- [ ] **Step 1: Write the failing tests**

  - `test_route_item_from_continuation`: an intake item in C2 routed to
    planning is unlinked from C2 and linked into the planning chain.
  - `test_route_into_full_planning_rolls_over`.
  - `test_settle_after_transfer_finds_listing_in_continuation`: the stale
    listing (the #4004 case) is in C2 and is cleared there.
  - `test_relink_after_failed_transfer_uses_chain`.
  - The existing #3839/#4004/#4291 regression tests pass unchanged.

- [ ] **Step 2: Run them to verify they fail**

  Run: `vrg-container-run -- pytest tests/vergil_tooling/test_intake_route.py -v`
  Expected: the new tests FAIL.

- [ ] **Step 3: Implement**: `intake_route` makes no raw read or write.

- [ ] **Step 4: Validate**: `vrg-container-run -- vrg-validate` passes.

- [ ] **Step 5: Commit**: `vrg-commit --type feat --scope intake-route --message "route and settle intake items across chained queues"`.

### Task 9a: the audit's read side (vergil-tooling)

Spec §4.1 (consumers skip continuations), §4.2 (no raw queries), §4.7
(reporting). Blocked by 6.

**Files:**

- Modify: `src/vergil_tooling/lib/epic_audit.py` (`list_home_epics` consumers:
  `epic_documents`, `closed_epic_open_child`, `proxy_link_drift`,
  `misplaced_adhoc`; `_OPEN_ISSUES_QUERY`; `epic_children`; new
  `chain_drift`)
- Modify: `src/vergil_tooling/lib/epic_docs.py` (`is_skipped`)
- Modify: `src/vergil_tooling/bin/vrg_epic_audit.py` (container count with
  `--epic`; drift sections)
- Test: `tests/vergil_tooling/test_epic_audit.py`, `test_vrg_epic_audit.py`, `test_epic_docs.py`

**Interfaces:**

- Produces: `epic_audit.ChainDrift` dataclass (`head: IssueRef`, `kind: str`
  one of `drift`, `full-tail`, `duplicate`, `closed-head-open-continuation`,
  `orphan`, `closed-continuation`; `containers: tuple[IssueRef, ...]`;
  `rule: str`), and `chain_drift(home: str) -> list[ChainDrift]`. 9b heals
  from this list.

- [ ] **Step 1: Write the failing tests**

  - `test_continuation_not_reported_missing_documents`.
  - `test_continuation_not_a_closed_epic_with_open_child`.
  - `test_continuation_not_a_primary_in_proxy_drift`.
  - `test_misplaced_adhoc_sees_task_in_adhoc_continuation`: the task's raw
    parent is the ad-hoc C2; it maps to the ad-hoc head.
  - `test_epic_children_spans_chain_and_proxies`.
  - `test_audit_epic_shows_container_count`: `--epic` on a 3-container chain
    prints `containers: 3`.
  - `test_audit_reports_each_drift_kind`: one fixture per `ChainDrift.kind`.
  - `test_audit_notes_closed_continuation_in_open_chain`: reported as a note,
    exit 0.
  - `test_audit_lists_orphan_continuation`.

- [ ] **Step 2: Run them to verify they fail**

  Run: `vrg-container-run -- pytest tests/vergil_tooling/test_epic_audit.py tests/vergil_tooling/test_vrg_epic_audit.py tests/vergil_tooling/test_epic_docs.py -v`
  Expected: the new tests FAIL.

- [ ] **Step 3: Implement**: `_OPEN_ISSUES_QUERY` drops its `parent`
  selection; `misplaced_adhoc` gets each parent from the chain layer
  (`epics.parent_of`, now chain-aware).

- [ ] **Step 4: Validate**: `vrg-container-run -- vrg-validate` passes.

- [ ] **Step 5: Commit**: `vrg-commit --type feat --scope epic-audit --message "audit chained epics: skip continuations, read across the chain and report chain drift"`.

### Task 9b: the sweep's writes (vergil-tooling)

Spec §4.6 (every close closes the chain), §4.7 (nudge, heals, orphans).
Blocked by 9a.

**Files:**

- Modify: `src/vergil_tooling/lib/epic_audit.py` (`_QueueCapacity`,
  `StandingEpicFull`, `attach_orphans`, the re-links at line 1197,
  `close_drift`, `close_proxies`; new `heal_chains`)
- Modify: `src/vergil_tooling/bin/vrg_epic_audit.py`
- Modify: `docs/site/docs/standards/github-issues.md` (the orphan pass: the
  nudge replaces the stop; the chain heals)
- Test: `tests/vergil_tooling/test_epic_audit.py`, `test_vrg_epic_audit.py`

**Interfaces:**

- Consumes: `ChainDrift`, `chain_drift` (9a); `epic_chain.link_into`,
  `heal_full_tail`, `heal_duplicate_continuations`, `close_chain`,
  `find_orphans`, `find_or_create_continuation`; `epics.rollup`.
- Produces: `heal_chains(drift: list[ChainDrift]) -> list[str]` (one line per
  heal).

- [ ] **Step 1: Write the failing tests**

  - `test_orphan_pass_past_95_links_and_nudges`: the intake head has 95 open
    children → the orphans are linked through the chain, exit 0, and the
    report has `intake queue <ref> holds 95 open items; run epic-review
    intake`. Replaces the `StandingEpicFull` stop test.
  - `test_sweep_heals_full_tail`, `test_sweep_heals_duplicates`,
    `test_sweep_closes_open_continuations_of_closed_head`.
  - `test_sweep_adopts_orphan_and_reruns_rollup`: the orphan held the last
    open item, now closed → after adoption the rollup closes the chain.
  - `test_sweep_reports_chained_development_epics`: each open development or
    proxy epic with a continuation is a report line, not a failure.
  - `test_close_drift_closes_whole_chain` and
    `test_close_proxies_closes_whole_chain`.

- [ ] **Step 2: Run them to verify they fail**

  Run: `vrg-container-run -- pytest tests/vergil_tooling/test_epic_audit.py tests/vergil_tooling/test_vrg_epic_audit.py -v`
  Expected: the new tests FAIL.

- [ ] **Step 3: Implement**: `StandingEpicFull` is removed; `_QueueCapacity`
  only counts, for the nudge.

- [ ] **Step 4: Validate**: `vrg-container-run -- vrg-validate` passes.

- [ ] **Step 5: Commit**: `vrg-commit --type feat --scope epic-audit --message "sweep through chained epics: nudge past the queue cap, heal chains and close them whole"`.

### Task 10: the remaining readers (vergil-tooling)

Spec §4.2 (readers), §4.1 (lookups). Blocked by 6.

**Files:**

- Modify: `src/vergil_tooling/lib/queue_view.py`, `roadmap.py`,
  `planning_close.py`, `security_records.py`, `merge_review/request.py`
  (`_fetch_task`, `_open_sub_issues`, `_primary_open_tasks` and their raw
  queries), `src/vergil_tooling/lib/pr_workflow/batch_rules.py` (the
  batch "same epic" rule `vrg-submit-pr` runs)
- Test: the matching test files

**Interfaces:**

- Consumes: the chain-aware `epics` readers (Task 6); `epics.is_continuation`.

- [ ] **Step 1: Write the failing tests**, one per module, each with an item
  in a continuation:
  - `test_queue_view_lists_items_in_continuation`, and the planning view
    does not list the continuation as an item;
  - `test_roadmap_skips_continuations`;
  - `test_planning_close_reads_chain`;
  - `test_security_records_reads_chain`;
  - `test_merge_review_open_tasks_span_chain`: a task in C2 gets its epic's
    open tasks from the whole chain, and `_fetch_task` reports the head as
    its parent;
  - `test_submit_pr_batch_same_epic_across_containers`.

- [ ] **Step 2: Run them to verify they fail**

  Run: `vrg-container-run -- pytest tests/vergil_tooling -k "continuation or chain" -v`
  Expected: the new tests FAIL.

- [ ] **Step 3: Implement**: each module's own `parent`/`subIssues` GraphQL
  selection is replaced with an `epics` reader.

- [ ] **Step 4: Validate**: `vrg-container-run -- vrg-validate` passes.

- [ ] **Step 5: Commit**: `vrg-commit --type feat --scope epics --message "read queues, roadmaps, planning closes, advisories and merge-review scopes across the chain"`.

### Task 11: lock the raw layer (vergil-tooling)

Spec §4.2 (raw writers private; no raw queries outside the chain layer).
Blocked by 7, 8, 9b, 10.

**Files:**

- Modify: `src/vergil_tooling/lib/epics.py` (`add_child` → `_add_child`,
  `remove_child` → `_remove_child`, `reparent_child` → `_reparent_child`;
  their uses in the archive drain stay, as `epics`-internal)
- Modify: `src/vergil_tooling/lib/epic_chain.py`
- Create: `tests/vergil_tooling/test_chain_layer_boundary.py`

- [ ] **Step 1: Write the failing test**

  `test_no_raw_subissue_access_outside_chain_layer`: tokenizes every `.py`
  under `src/vergil_tooling` except `lib/epics.py` and `lib/epic_chain.py`
  (`tokenize`, so comments are skipped) and asserts that no `NAME` token is
  `add_child`, `remove_child` or `reparent_child` (or their private forms),
  and that no `STRING` token matches the GraphQL selections
  `\bsubIssues\s*[({]`, `\bparent\s*\{\s*number`, `\baddSubIssue\s*\(`,
  `\bremoveSubIssue\s*\(` or the REST path `/sub_issues`. Docstrings and
  messages that merely name these (today `intake_route.py:1565-1580`,
  `epic_audit.py:1599`, `planning_close.py:158`) do not match the
  selection patterns; the test asserts that too, with those three lines as
  negative fixtures. A failure lists each file and line.

- [ ] **Step 2: Run it to verify it fails** (on the public names), then
  rename.

  Run: `vrg-container-run -- pytest tests/vergil_tooling/test_chain_layer_boundary.py -v`
  Expected: FAIL, then PASS after the renames.

- [ ] **Step 3: Validate**: `vrg-container-run -- vrg-validate` passes.

- [ ] **Step 4: Commit**: `vrg-commit --type refactor --scope epics --message "make the raw sub-issue layer private to epics and epic_chain"`.

### Task 12: the vergil-tooling release, installed (deployment, .github)

Spec §4.8 (once released and installed). Blocked by 11.

- **Precondition (human-attested):** a vergil-tooling release carrying
  Tasks 2, 3, 5a, 5b, 5c, 6, 7, 8, 9a, 9b, 10 and 11 is published, its
  version and tag named.
- **Procedure:**
  1. `vrg-release-record check <ref>` exits 0 for each of those tasks. For a
     task with no record (exit 4), `vrg-git merge-base --is-ancestor
     <merge-sha> <tag-sha>` in vergil-tooling exits 0 instead.
  2. Confirm the release's `install` stage ran: it installs the release on
     the Mac that ran it and runs `vrg-vm update --all` on every VM.
     **Human-run steps (need a human identity):** on any other host,
     `vrg-tooling-install <version>`; `vrg-vm update --all` only if a VM
     was not reached by the install stage.
  3. `python -c "import vergil_tooling.lib.epic_chain"` succeeds in the
     installed environment, and `vrg-epic-audit --help` runs.
- **SUCCESS:** every check passes, recorded with the version and tag.

### Task 13: re-home #1384's tasks, the live rollover (validation, .github)

Spec §4.8, goals 1, 3, 5 and 7. Blocked by 12.

- **Precondition (machine probe):** vergil-project/.github#1384 is open
  (`vrg-gh issue view 1384 --repo vergil-project/.github --json state`). Its
  sub-issue total was 96 on 2026-11-24; record the current total.
- **Procedure (if #1384 is open):**
  1. Move each of the 14 tasks listed on the planning issue
     (vergil-project/vergil-tooling#3840, comment of 2026-11-24) with
     `vrg-epic-move --epic vergil-project/.github#1384 --task <ref>`.
  2. Observe the rollover: once the head holds 99 items, the next move
     prints the development-epic rollover line, and the head gets the same
     comment.
  3. `vrg-epic-audit --epic vergil-project/.github#1384` lists all 14, shows
     `containers: 2`, and reports no chain drift.
  4. The epic sweep's next run (`ops.yml`, or `vrg-epic-audit` in its sweep
     mode) reports #1384 as a chained development epic, not a failure.
  5. Tell the #1384 driver the hand tracking can be dropped; note it on
     #1384.
- **Fallback (if #1384 has closed):** the 14 tasks stay where they are
  (closed is closed). Make the live check on a throwaway development epic
  instead: `vrg-epic-create --repo vergil-project/.github --title "Epic:
  chain rollover probe"`, file 100 throwaway tasks under it with
  `vrg-issue-create --epic …` (titles `Probe: rollover <n>`), observe steps
  2 and 3 against it, then close every task as not planned and let the
  rollup close the chain, confirming head-first.
- **SUCCESS:** the rollover line, the head comment and the audit output are
  recorded in the result comment, with no item left outside the chain.

## Evolution during execution
