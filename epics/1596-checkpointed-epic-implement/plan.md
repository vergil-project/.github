# checkpointed-epic-implement — implementation plan

- **Epic:** vergil-project/.github#1596
- **Planning issue:** vergil-project/.github#1544 (review history is in its comments)
- **Status:** approved

> **For agentic workers:** each task is one issue and one PR in the repo named
> in its heading, worked with `issue-implement`. Steps use checkbox (`- [ ]`)
> syntax for tracking.

**Goal:** An `epic-implement` run can be compacted or cleared at any task
boundary and continue from the durable record alone, with nothing it needs
held only in the orchestrating session, and it recommends a clear at each
human gate.

**Architecture:** Skill text only, in `vergil-project/vergil-claude-plugin`.

- `issue-implement` and the paad review doctrine: a run dispatched by
  `epic-implement` keeps the details of each security non-review item in the
  task's state directory, with a `Kept:` line, instead of in its final
  message (spec §4.2).
- `epic-implement`: the durable-state, re-derivation, thin-orchestrator and
  clear-at-the-gate guidelines (spec §4.1, §4.3–§4.5), the gate's reading of
  the kept files, and the orchestrator's own security follow-ups kept in a
  new `epic-implement` state directory (spec §4.2).

No tool changes: `vrg-checkpoint` gains no workflow.

**Tech Stack:** Markdown skills and docs (vergil-claude-plugin).

**Spec:** [`spec.md`](spec.md) in this directory. Executors read both.

## Global Constraints

- **No stored cursor:** no `vrg-checkpoint` workflow, no compaction hook, no
  outer driver, no forced clear and no pause made only to recommend one
  (spec §2, non-goals).
- **A security detail is never written anywhere public,** and is shown in
  the terminal only (spec §2, anti-goals; doctrine §1.7).
- **A kept file's hash has one source:** the `Kept: <file> sha256:<hash>`
  line of a trusted comment, read through `vrg-issue-read`: the
  `Awaiting a ruling` comment that carries the item's count, or a held
  item's own `Kept:` comment (Task 2, step 1). Never the final report, never
  a file beside the details file (spec §4.2).
- **A kept file is deleted only once its item's ruling is recorded;** a
  failed record keeps it, with its `Kept:` line (spec §4.2).
- **The kept-file name** is `security-<ID>.md`, `<ID>` the item's
  `Awaiting a ruling` ID (§1.7's numbering, for example `N0.S1`). A `record`
  ruling's existing `security-<i>.md` files use a bare number, so the two
  never collide. A `record` ruling on a kept item uses the kept file as its
  details file, as `operational/<N>/` already does, and writes no second one
  (the naming spec §6 leaves to the builder, decided here).
- **The new state-directory kind:** `epic-implement`, at
  `build/.vergil-state/epic-implement/<N>/` in the main clone of the epic's
  home repo, `<N>` the epic's number, following doctrine §1.5's
  `upstream-report` and `operational` rules (spec §4.2). The skill creates
  and checks it by those rules as written; vergil-tooling's shared
  state-directory checker does not know the kind, and adding it is an
  intake follow-up outside this epic.
- **The gate's clear recommendation** ends the gate's summary with exactly:

  ```text
  /clear
  /vergil:epic-implement <epic-ref>
  ```

  and, in the `adhoc` mode, the command that started the run in place of the
  second line (spec §4.5).
- **A run with the human present** (not dispatched by `epic-implement`)
  keeps today's behaviour: details in the conversation, ruled on there
  (spec §4.2).

## Review Focus

The skills have no test harness. Each line below is pinned by the step that
states it in the skill; the reviewer checks the text against it.

1. **A clear between a dispatched run's `Awaiting a ruling` comment and the
   gate.** The next session finds the kept file from the directory listing
   (confirmed by the human) and the `Kept:` line, and walks the item as the
   dispatching session would have. Pinned in Task 3, step 3.
2. **A kept file that fails its check:** its hash differs from its `Kept:`
   line, it has no trusted `Kept:` line, or a `Kept:` line names a file that
   is not there. Each is a stop for the human, nothing read or passed on.
   Pinned in Task 2, step 2, and Task 3, step 3.
3. **A failed `vrg-security-record`.** The file and its `Kept:` line stay;
   the next run checks for a trusted `Recorded: <ID>` line before recording
   again, and records once. Pinned in Task 2, step 3.
4. **A compacted or resumed run meets a ruling it already carried out.**
   The re-derivation skips every `Awaiting a ruling` ID that is ruled (a
   trusted `Human ruling:` or `Recorded:` line, or for a security item a
   trusted security counts line covering it), so no ruling is applied twice
   and no task dispatched twice. Pinned in Task 3, step 2.
5. **A kept file in a batch member's directory.** `vrg-pr-workflow
   batch-extract` refuses while a `security-*.md` sits in
   `issue-implement/<N>/`; the refusal is loud and says to record first.
   The skill names this so the run records before it extracts. Pinned in
   Task 2, step 1.

## Delivery table

Each line gives the task, its repo and kind, what it delivers, and what it
is blocked by.

- **1** (.github, task #1597): spec and plan, this PR.
- **2** (vergil-claude-plugin): `issue-implement` keeps a dispatched run's
  security details in the task's state directory with a `Kept:` line, and
  the doctrine's §1.5 and §1.7 are revised to match, with the new
  `epic-implement` kind. Blocked by
  vergil-project/vergil-claude-plugin#1206.
- **3** (vergil-claude-plugin): `epic-implement`'s guidelines, the gate's
  reading of the kept files, and the orchestrator's own security follow-ups
  in the `epic-implement` state directory. Blocked by 2 and by
  vergil-project/vergil-claude-plugin#1206.
- **doc-review** (vergil-claude-plugin, documentation review). Blocked by 3.
  Its sweep covers the plugin's `docs/site/docs/skills/` pages for
  `epic-implement` and `issue-implement`, and the docs repo's description
  of the implementation workflow.
- **retrospective** (.github). Blocked by every other task.

Notes:

- **One plugin release carries 2 and 3 together.** With Task 2 alone, a
  dispatched run names its kept file in its final report, but the released
  `epic-implement` still expects the details there, so the gate would have
  no way to present the item. So Task 2 is blocked by #1206 too, and
  starts only once Task 3 can follow it at once: the two land close
  together, and the release that carries Task 2 waits only until Task 3
  has merged, not open-endedly. Task 2's issue says so; no deployment task
  is needed, since the change is skill text the pipeline's checks cover
  (spec §5).
- **Implementation waits for epic vergil-project/.github#1384,** through
  #1206, which Task 3 is blocked by.
- **No fixture runs, no `dev-` image validation, no deployment or
  validation task:** the change touches no language support, container
  image, shared workflow or release tooling (spec §5).
- **Filing.** Every task above is filed at epic-create's filing step, in its
  repo, under #1596, with its `Blocked-by:` refs and its `Plan task:` key.

---

### Task 2: dispatched runs keep security details durable (vergil-claude-plugin)

Spec §4.1, §4.2. Blocked by vergil-project/vergil-claude-plugin#1206
(delivery table notes).

**Files:**

- Modify: `skills/issue-implement/SKILL.md`
- Modify: `docs/paad-review-doctrine.md`

The skill has no test harness. Its check is the review and
`vrg-container-run -- vrg-validate`.

- [ ] **Step 1: Rewrite the non-review security rule** in
  `issue-implement`'s opening section ("Creating an issue needs the human's
  approval", today "its details stay in the conversation (under
  `epic-implement`, in your final message)"). A run dispatched by
  `epic-implement` that raises a security non-review item:
  - writes its details to `security-<ID>.md` in
    `build/.vergil-state/issue-implement/<N>/` of the task repo's main
    clone, after a `test ! -e` check, under doctrine §1.5's rules for
    creating and checking the directory;
  - posts the file's hash as a `Kept: security-<ID>.md sha256:<hash>` line
    in the trusted `Awaiting a ruling` comment that carries the item's
    count;
  - names the file in its final report, never the details.

  While a merge-review ruling exchange is open, the item itself is held, as
  today, and posted under `Awaiting a ruling` after the response. Its file
  is still written at once, and its `Kept:` line is posted at once in a
  trusted comment of its own that carries only the item's ID and that line,
  so the item is durable across the stop; the later `Awaiting a ruling`
  comment names the same ID.

  With the human present, the rule is unchanged. Name the
  `batch-extract` interaction (Review Focus 5): a kept file blocks
  extraction until the item is recorded.
- [ ] **Step 2: Rewrite the rulings run's handling** of the same section
  ("It takes each item, and any security details, only from its prompt").
  For a kept item, the details come from the kept file, not the prompt:
  first check the file against its trusted `Kept:` line, read through
  `vrg-issue-read`; a differing hash, a missing line or a missing file is a
  stop (Review Focus 2). A `record` ruling on a kept item passes the kept
  file as `--details-file` and writes no second file; the existing
  `security-<i>.md` path stays for an item with no kept file.
- [ ] **Step 3: Deletion and the failed record.** The kept file is deleted
  once the item's ruling is recorded (`Recorded: <ID>`, or the counts line
  for a ruling other than `record`). A failed record keeps the file and its
  `Kept:` line and reports the item as still waiting; a later run checks for
  a trusted `Recorded: <ID>` before recording again (Review Focus 3). Update
  the `waiting: non-review items` row of the outcome table: the final report
  lists the items and names each kept file, never security details.
- [ ] **Step 4: Revise the doctrine** (spec §4.2):
  - §1.5's table, `issue-implement` row: add a dispatched run's kept
    `security-<ID>.md`;
  - §1.5's table: add the `epic-implement` row (path, repo and contents as
    Global Constraints give them);
  - §1.5's "Kept details files" bullet: add both kinds to its list of
    records that carry a kept file's hash;
  - §1.5's `issue-implement` bullet: replace "nothing at that level is
    kept" with the kept file's rule (kept with its `Kept:` line until its
    ruling is recorded, then deleted; `<N>/` deleted once it holds no file
    and no `merge-review/`);
  - §1.5: add an `epic-implement` bullet following the `operational` one;
  - §1.5's **Only your own files**: a dispatched run accepts a kept file in
    `issue-implement/<N>/` only when its prompt relays the human's
    confirmation of that directory's listing (as for `operational/<N>/`)
    and the file matches a trusted `Kept:` line; otherwise it stops;
  - §1.5, for the `issue-implement` and `epic-implement` kinds: a run that
    holds no hash of the directory in its own context (a new session, or
    after a compaction) validates it as a new session does: it checks each
    file against its trusted `Kept:` line, stops on any entry with no
    trusted `Kept:` line, and only then takes a fresh directory hash;
  - §1.7's numbering: a security ID's `N0.S<i>` continues past every
    `N0.S<i>` in a trusted `Awaiting a ruling` comment and every kept
    `security-N0.S<i>.md`, as well as past the existing sources, so no ID
    is reused;
  - §1.7's rule that non-review comments are never posted while a ruling
    exchange is open: add its one exception, the held item's own `Kept:`
    comment (Step 1);
  - §1.7's "Under `epic-implement`" paragraph and its "One channel per kind
    of item" non-review bullet: a dispatched run's final message names its
    kept files and carries no security details, and the rulings run takes
    them from the kept file, not from its prompt (spec §4.2);
  - §1.7's private-store paragraph ("Anywhere else there is no private
    store"): name both new stores, and keep "the details stay in the
    conversation" for a run with the human present only;
  - §1.7's **Recording** paragraph: a kept file is the details file for a
    code task too, and a failed record keeps it rather than handing the
    details back in a final message.
- [ ] **Step 5: Validate and commit:** `vrg-container-run -- vrg-validate`
  passes; `vrg-commit --type feat --scope issue-implement --message "keep a
  dispatched run's security details in the task's state directory"`.

### Task 3: compaction-safe, clear-friendly epic-implement (vergil-claude-plugin)

Spec §4.1–§4.6. Blocked by Task 2 and by
vergil-project/vergil-claude-plugin#1206: both edit the gate and stop
sections, and the guidelines describe the run as #1206 leaves it.

**Files:**

- Modify: `skills/epic-implement/SKILL.md`

The skill has no test harness. Its check is the review and
`vrg-container-run -- vrg-validate`.

- [ ] **Step 1: Durable state only** (spec §4.1). Extend the opening's
  "stateless by design" paragraph into the rule: everything the run needs
  is on GitHub or in a local state directory, and being there is not enough
  on its own, since the run reads each record back when it re-derives.
  List the records (task states, the plan, batch markers, merge-review
  directories and ruling requests, `Awaiting a ruling` items, ruling lines,
  the documentation-review bookend's items, the epic's follow-up comments,
  and the kept files of Task 2 and step 5).
- [ ] **Step 2: Re-derivation** (spec §4.3). In step 2 ("Reconstruct
  state"), add the pending non-review items: for every task of the epic (or
  of the ad-hoc queue) that carries a trusted `Awaiting a ruling` comment
  or a held item's `Kept:` comment, closed tasks included, as §1.7 lets a
  rulings run act on a closed task, and for the epic issue itself (in the
  `adhoc` mode, every ad-hoc epic the run writes records to), which holds
  the orchestrator's own follow-ups (step 5), read the trusted comments
  with `vrg-issue-read` and collect every such ID that is not ruled
  (Review Focus 4). An ID is ruled
  when a trusted `Human ruling: <ID>` or `Recorded: <ID>` line names it, or,
  for a security item, when a trusted security counts line covers it, the
  test merge-review's non-review gate already uses. State when the full
  re-derivation runs: before the first dispatch of each frontier pass, on a
  re-invocation, before presenting a gate, and whenever the run is unsure.
  In step 3, before each dispatch within a pass, keep only the per-task
  checks (still open and ready, plus the existing batch, trust and
  merge-review checks). State that a compaction summary is never a source of
  state, and that the skill does not detect a compaction.
- [ ] **Step 3: The gate is built from the record** (spec §4.2, §4.3). In
  step 4, the non-review bullet presents the items from step 2's list, not
  from runs' final reports. For a kept security item: show the human the
  listing of the task's `issue-implement/<N>/` (pruning `merge-review/` as
  doctrine §1.5 says); once confirmed, check each file against its trusted
  `Kept:` line and read it to walk the item, in the terminal only (Review
  Focus 1, 2). Every prompt that dispatches a run of a task with a kept
  file (the rulings run, a continuation after `waiting: non-review items`,
  a resume after a merge-review reply) relays the human's confirmation of
  that listing, as the gate already does for `operational/<N>/`. The
  rulings run's prompt names the kept file and carries no security
  detail. Each item the gate presents names its durable source.
  Find every item that reaches the gate only through a final report and
  make it regenerable; the known one is a deployment stopped at `waiting:
  human-run steps`, re-dispatched to produce its precondition results again.
  Make any other such item regenerable in the same way, and name each in
  the PR's notes.
- [ ] **Step 3a: Confirm every security-detail path** (spec §4.1, §6).
  Check each place a security detail can arise in a run that
  `epic-implement` drives (merge review's `merge-review/security-<ID>.md`,
  `operational/<N>/`, Task 2's kept files, and step 5's) and confirm each
  keeps its details in a state directory and none lives only in the
  session. Fix any gap found in this task; one that cannot be fixed here
  is a follow-up filed with the human's approval, named in the PR's notes.
- [ ] **Step 4: A thin orchestrator** (spec §4.4). In step 3, each dispatch
  prompt asks for a short final report: the outcome, the refs, and pointers
  to durable records, not the run's working detail. The orchestrator reads
  no task body, diff or log it does not need to choose the next step.
- [ ] **Step 5: The orchestrator's own security follow-ups** (spec §4.2).
  In step 5 ("Escalate on problems"), replace "its details kept in this
  conversation" with the `epic-implement` state directory: the details go to
  `security-<ID>.md` in `build/.vergil-state/epic-implement/<N>/` of the
  epic's home repo's main clone, after a `test ! -e` check; the hash goes in
  a `Kept:` line of the epic's `Awaiting a ruling` comment that carries the
  count; the gate reads it as step 3 says. The main session carries out the
  ruling itself: for a `record` ruling it runs `vrg-security-record` with
  the kept file as the details file and posts `Recorded: <ID>` on the epic;
  any other ruling is recorded by the counts line, as today. The file is
  deleted once the ruling is recorded; a failed record keeps the file and
  its `Kept:` line, and the item stays pending, so a later gate checks for
  a trusted `Recorded: <ID>` before recording again. In the `adhoc` mode,
  `<N>` is the epic its records go to.
- [ ] **Step 6: The clear recommendation** (spec §4.5, §4.6). At the end of
  step 4, once every item at the gate is on GitHub or in a state directory,
  the summary ends with the two commands of Global Constraints, and says
  that clearing and continuing run the same re-derivation. The `adhoc`
  mode's step 6 uses the command that started the run. Add no pause of its
  own.
- [ ] **Step 7: Validate and commit:** `vrg-container-run -- vrg-validate`
  passes; `vrg-commit --type feat --scope epic-implement --message "make
  epic-implement safe to compact or clear at any task boundary"`.

## Evolution during execution

None yet.
