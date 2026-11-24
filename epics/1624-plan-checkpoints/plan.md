# plan-checkpoints — implementation plan

- **Epic:** vergil-project/.github#1624
- **Planning issue:** vergil-project/.github#1623 (review history is in its comments)
- **Status:** approved

> **For agentic workers:** each task is one issue and one PR in the repo named
> in its heading, worked with `issue-implement`. Steps use checkbox (`- [ ]`)
> syntax for tracking.

**Goal:** A plan names the points where a long `epic-implement` run is
expected to reset, and `epic-implement` recommends `/clear` and a
re-invocation at those points and, on one rule, at other human stops, never
losing anything the run needs.

**Architecture:** Guidance only, in `vergil-claude-plugin`:

- **`epic-create`** writes a Reset points section into every plan (step 6)
  and checks it while walking each plan pushback and alignment round
  (step 7), as it already holds the plan to "Task sizing".
- **`epic-implement`** reads the section when it rebuilds state (step 2),
  treats a release or human-run deployment task an Evolution entry adds as a
  named reset point, states the whole-gate reset rule in its gate (step 4),
  records a batch's rationale at dispatch (step 3), and keeps the reset rule
  out of the `adhoc` mode.

No tool, hook or format changes (spec §4.3).

**Tech Stack:** Markdown skills and the plugin's MkDocs site
(`docs/site/docs/skills/index.md`).

**Spec:** [`spec.md`](spec.md) in this directory. Executors read both.

## Global Constraints

- **The section's name is Reset points**, never Checkpoints, in both skills
  and the site docs (spec §4.1, "The name"). `epic-implement`'s text never
  uses "checkpoint" for a reset.
- **The recommendation is `/clear`, never `/compact`** (spec §4.2), with the
  two commands ready to paste:

  ```text
  /clear
  /vergil:epic-implement <epic-ref>
  ```

- **A reset point sits at a human stop** the run reaches anyway: a release
  or a human-run deployment step is the typical kind. A phase boundary with
  no human action is never one (spec §4.1).
- **The rule applies to the whole gate** (spec §4.2): every item the run is
  blocked on there is completed and recorded, no dispatched run is in
  flight, and every finished run's final report is taken in and recorded.
- **A declined or deferred item is not completed**: it blocks the reset at
  that gate and every later gate until ruled or run, and the agent says the
  reset is skipped and names the item (spec §4.2).
- **Session-only state blocks a reset**: unrecorded pasted evidence, and a
  non-review or follow-up security item's details. A merge-review security
  item's `merge-review/security-<ID>.md` counts as recorded (spec §4.2).
- **The `adhoc` mode gets no reset rule**; only the batching rationale
  applies in both modes (spec §2, non-goals).
- **The human decides every reset.** Each one is a recommendation, and the
  human may continue in the session instead (spec §4.2).

## Review Focus

1. **A gate where the human declines one item** (a human-run deployment step
   not run) and completes the rest. The agent must not recommend a reset, and
   must say which item holds it, at this gate and every later one. Pinned in
   Task 2b, step 3.
2. **A gate opened by a per-repo handoff while another repo's run is still
   in flight.** No reset until that run reports and its report is recorded.
   Pinned in Task 2b, step 3.
3. **A plan written before this epic**, with no Reset points section.
   `epic-implement` falls back to the judgment rule alone, and
   `epic-create`'s check does not demand the section of a plan it is not
   reviewing. Pinned in Task 2b, step 2.
4. **An adhoc run reaching the shared gate text.** The reset rule must not
   reach it, while the batching rationale does. Pinned in Task 2b, step 4.
5. **A batch stop after a reset**, when the human asks why the issues were
   batched. The fresh run answers from the rationale line recorded at
   dispatch, not from a reconstruction. Pinned in Task 2b, step 4.

## Reset points

This plan has no planned human stop at which a run is expected to reset.
Both implementation tasks are guidance edits that agents submit and merge
themselves under epic vergil-project/.github#1384, and no task here needs a
release or a human-run deployment step: nothing downstream needs the plugin
change deployed.

## Delivery table

Each line gives the task, its repo and kind, what it delivers, and what it
is blocked by.

- **1** (.github, task #1625): spec and plan, this PR.
- **2a** (vergil-claude-plugin): `epic-create`'s Reset points section in
  plan writing (step 6) and its check in plan pushback and alignment
  (step 7), with the site docs' `epic-create` section. Blocked by
  vergil-project/.github#1384.
- **2b** (vergil-claude-plugin): `epic-implement`'s reading of the section,
  the Evolution-entry reset points, the whole-gate reset rule, the
  batching rationale at dispatch and the `adhoc` exclusion, with the site
  docs' `epic-implement` section. Blocked by vergil-project/.github#1384.

### Bookends

- **Documentation review** (vergil-claude-plugin, `Plan task: doc-review`).
  Blocked by 2a, 2b. Its sweep starts from the items recorded on it and
  covers the plugin's `docs/site/docs/skills/index.md` and the docs repo's
  description of the epic lifecycle, if it describes `epic-implement`'s
  gate.
- **Retrospective** (.github, `Plan task: retrospective`). Blocked by
  everything else.

Notes:

- **Blocked by epic #1384, not by one of its tasks.** The gates this design
  resets at are the ones a #1384 run still stops at (spec §5), so 2a and 2b
  carry `Blocked-by: vergil-project/.github#1384`, the epic issue, which
  closes when its last task does.
- **2a and 2b are independent.** Either alone is harmless: a plan with the
  section is read by an `epic-implement` that ignores it, and an
  `epic-implement` that reads the section falls back to judgment when a plan
  lacks it. They touch different skills and different site-doc sections,
  and `epic-implement` may batch them into one PR on its own judgment.
- **No operational tasks.** No release, deployment or validation task: the
  change is guidance, exercised by the next epic planned and run with it
  (spec §5). The plugin release that ships it is the fleet's ordinary
  release, which no task here waits on.
- **No fixture runs** and no `dev-` image validation: the change touches no
  language support, container image, shared workflow or release tooling.
- **No follow-on brainstorm task:** no enabling chain is known. Tuning the
  guidance from experience (spec §1) accrues as follow-up issues.

---

### Task 2a: `epic-create` plans for reset points (vergil-claude-plugin)

Spec §4.1, §6.

**Files:**

- Modify: `skills/epic-create/SKILL.md` (step 6, `plan`; step 7,
  `plan-pushback` and `alignment`; "Design review rounds" step 4 only if
  the walk's wording lives there)
- Modify: `docs/site/docs/skills/index.md` (the `## epic-create` section)

**Interfaces:**

- Produces: a plan section headed `## Reset points`, the term "reset point"
  and its definition (a planned human stop after which everything the run
  needs is durable), which Task 2b's `epic-implement` text reads by that
  heading.

- [ ] **Step 1: Add the section to plan writing (step 6).** After the
  sentence on operational and follow-on tasks, require a `## Reset points`
  section in every plan: each planned human stop at which a run is expected
  to reset, with the plan tasks it follows and what the human does there
  (typically the `vrg-release` commands a deployment task, or a later task,
  needs as a human-gated precondition, or human-run deployment steps). A plan
  with no such stop says so in one line. State that a reset point is a stop
  the run makes anyway, never a phase boundary with no human action, and
  that the section is not called Checkpoints because `epic-create` uses that
  word for its own stages. Its layout and position in the plan are the
  writer's call within that.
- [ ] **Step 2: Add the check to plan review (step 7).** Beside the
  sentence that holds the plan to "Task sizing" in each plan pushback
  round, add: in each plan pushback round **and each alignment round**,
  check that the plan has the section, that each named reset point sits at
  a real human stop in the delivery table, and raise any planned release or
  human-run step the section leaves out.
- [ ] **Step 3: Update the site docs.** In the `## epic-create` section of
  `docs/site/docs/skills/index.md`, add one short paragraph: every plan
  carries a Reset points section naming the human stops where an
  `epic-implement` run is expected to reset, checked in plan pushback and
  alignment, and link it to `epic-implement`'s reset recommendation.
- [ ] **Step 4: Check the wording.** Run
  `vrg-git grep -n -i 'checkpoint' -- skills/epic-create/SKILL.md` and
  confirm no new line uses "checkpoint" for a reset point. Run
  `vrg-git grep -n 'Reset points' -- skills/epic-create/SKILL.md
  docs/site/docs/skills/index.md`. Expected: hits in step 6, step 7 and the
  site docs.
- [ ] **Step 5: Validate.** Run `vrg-container-run -- vrg-validate`.
  Expected: pass.
- [ ] **Step 6: Commit** with `vrg-commit --type feat --scope epic-create`,
  message naming the Reset points section and its review check.

### Task 2b: `epic-implement` recommends resets (vergil-claude-plugin)

Spec §4.2, §4.3, §2 (non-goals), §6.

**Files:**

- Modify: `skills/epic-implement/SKILL.md` (step 2, reconstruct state;
  step 3, batching; step 4, the gate; "The `adhoc` mode", its step 6 and
  "What the mode does not do"; Notes)
- Modify: `docs/site/docs/skills/index.md` (the `## epic-implement`
  section)

**Interfaces:**

- Consumes: the plan's `## Reset points` section and the term "reset point"
  (Task 2a). A plan without the section is valid.
- Produces: nothing another task reads.

- [ ] **Step 1: Read the reset points (step 2).** In step 2's first
  item, where the plan is read, add that the run reads the plan's
  `## Reset points` section with the rest of the plan; a plan written before
  this epic has none, and the run then relies on the judgment rule alone.
- [ ] **Step 2: Reset points added during execution (step 2).** In the
  "Plans evolve append-only" paragraph, add: a release or human-run
  deployment task an `## Evolution during execution` entry adds is a named
  reset point, as if the section listed it, and the entry that adds it says
  so. State the pre-epic fallback from step 1 here too if the paragraph is
  where a reader looks for it (Review Focus 3).
- [ ] **Step 3: The reset rule (step 4).** Add a subsection to step 4,
  after the gate's item list and before "PRs left for the human, per
  repository", headed to name the reset (for example "Recommend a reset
  once the gate is recorded"). It states, from spec §4.2:
  - the whole-gate rule and its three conditions (Global Constraints), and
    the two commands to paste;
  - a declined or deferred item blocks the reset at this gate and every
    later gate until ruled or run; a named reset point is then skipped, and
    the agent says so at the gate, naming the item that holds it (Review
    Focus 1);
  - a run still in flight blocks the reset, since its final report comes
    back only to the dispatching session, including a gate a per-repo
    handoff opens while another repo's tasks are in flight (Review Focus 2);
  - the up-front form, allowed only when every item leaves its own durable
    record and needs nothing pasted back, nothing at the gate is held only
    in the session, and no run is in flight; otherwise the reset is
    recommended only once the human reports back and everything is
    recorded;
  - the reset verifies nothing itself: a missed or failed release surfaces
    at the dependent task's own precondition check;
  - at a named reset point the agent always recommends the reset once the
    conditions hold; at any other human stop it may, on its own judgment;
    the human decides either way;
  - the session-only list: unrecorded pasted evidence and a non-review or
    follow-up security item's details block it; a merge-review security
    item's `merge-review/security-<ID>.md` does not;
  - `/clear`, never `/compact`, with the reason (a compacted summary can
    carry stale state into the run);
  - the rule applies to the development path only, not to the `adhoc`
    mode.

  Then change "When the human returns and says continue (or re-invokes this
  skill)" so it reads as the two paths the reset leaves open: continuing in
  the session, or the cleared re-invocation.
- [ ] **Step 4: The batching rationale and the `adhoc` mode.** In step 3's
  "Batching ready tasks" paragraph, add: when dispatching a batch, record in
  one line why the issues were batched together, screened against doctrine
  §1.6 first, linked as the skill's sweep-items paragraph links it (a
  security-class reason posted only as "security-related", its details kept
  as a non-review security item's are), in a comment on the epic issue (in
  the `adhoc` mode, on the epic its records go to, linking the skill's
  "Where an adhoc run writes its records" section): the comment
  that already carries the run's items, or a new one. In step 4's batch-stop
  item, change "answer from what you knew when you batched them" to answer
  from that recorded line (Review Focus 5). In "The `adhoc` mode", step 6
  ("Gate and stop"), add that the gate runs without step 4's reset rule;
  in "What the mode does not do", add the reset rule (Review Focus 4).
- [ ] **Step 5: Update the site docs.** In the `## epic-implement` section
  of `docs/site/docs/skills/index.md`, change "re-invoking after a lost or
  compacted session" so it does not present compaction as a recommended
  reset, and add one short paragraph: at a recorded gate, and always at the
  plan's reset points, the skill recommends `/clear` and a re-invocation,
  never while anything is held only in the session or a run is in flight,
  and not in the `adhoc` mode; a batch's rationale is recorded at dispatch.
- [ ] **Step 6: Check the wording.** Run
  `vrg-git grep -n -i -E 'checkpoint|compact' --
  skills/epic-implement/SKILL.md docs/site/docs/skills/index.md`. Expected:
  no line uses "checkpoint" for a reset, and every "compact" line either
  says never to recommend it or describes recovery after an automatic
  compaction.
- [ ] **Step 7: Validate.** Run `vrg-container-run -- vrg-validate`.
  Expected: pass.
- [ ] **Step 8: Commit** with `vrg-commit --type feat --scope
  epic-implement`, message naming the reset recommendation and the batching
  rationale.
