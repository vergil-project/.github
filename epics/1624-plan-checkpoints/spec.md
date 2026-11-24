# plan-checkpoints: plan-driven reset points for epic-implement — design

- **Epic:** vergil-project/.github#1624
- **Planning issue:** vergil-project/.github#1623 (review history is in its comments)
- **Status:** approved

## 1. Intent

A long `epic-implement` run keeps one orchestrating session open across an
epic's tasks, and that session's context grows with every task's results,
gate outcomes and rulings. On a large epic it has passed 600,000 tokens, a
significant share of the fleet's token cost.

`epic-implement` is already stateless by design: a fresh invocation rebuilds
the run's position from GitHub, the plan and the local state directories.
What is missing is not recovery but the habit of resetting. This epic adds
that habit at the points where a human is already involved, so a run's
context is bounded by the stretch between two human stops rather than by the
whole epic.

This is a deliberately small first pass. Which stops deserve a reset is not
yet known from experience. The design gives the agent room for judgment, and
the guidance is expected to be tuned as runs show where resets pay off.

### Why now

Epic vergil-project/.github#1384 lets agents submit and finalize their own
PRs into `develop`, so a run stops for the human far less often. The main
stop that remains is a **release**: agents are not yet allowed to cut one,
so a run that needs merged work released pauses and asks the human to run
the `vrg-release` commands. That pause is already a human interaction, and
it is the natural place to reset.

## 2. Goals, non-goals, anti-goals

### Goals

- A plan names, in advance, the points where a run is expected to reset.
- `epic-implement` recommends a reset (`/clear`, then re-invoking the skill)
  at those points, and may recommend one at other human stops, on one rule.
- A reset never loses anything the run needs.

### Non-goals

- Resets at points where no human is involved (an unattended run is not
  interrupted).
- An outer driver that starts a fresh session per phase.
- Any change to how `epic-implement` recovers its position: that already
  works.
- New tooling or a machine-readable reset point format.
- The `adhoc` mode of `epic-implement`. Its runs are human-driven and
  already cleared and re-invoked by hand as a matter of routine, so the reset
  rule applies only to the development path. Where the adhoc mode reuses the
  development path's gate step, the rule is stated so that it does not reach
  adhoc runs. The one exception is the batching rationale (§4.2), a dispatch
  rule that applies in both modes.

### Anti-goals

- **Resetting too often.** Re-bootstrapping a large, complex epic costs time
  and I/O of its own. A reset after every PR would cost more than it saves.
- **A reset that loses evidence.** When the human pastes a command's output
  so the agent can record it, that output exists only in the session until
  the agent records it. A reset between the two loses it.

## 3. Approaches

1. **`epic-implement` alone.** At any human stop, once everything from it is
   recorded, the agent may recommend a reset. The plan is unchanged. Smallest,
   but `epic-create` takes no part and nothing records where resets are
   expected.
2. **The plan names the expected resets; `epic-implement` adds judgment.**
   `epic-create` writes a Reset points section into every plan and its plan
   review checks it; `epic-implement` reads it, recommends a reset at each
   named reset point, and may recommend one at other human stops under the
   same rule. Skill guidance and prose only.
3. **Machine-readable reset points.** A fixed format in the plan, read by a
   tool (for example `vrg-epic-audit` reporting the next reset point). Precise,
   but it builds machinery before the guidance has been learned.

**Chosen: approach 2.** It makes the plan and the orchestrator work together,
as the planning issue asks, while staying small enough to learn from.
Approach 1 leaves out the plan; approach 3 is premature.

## 4. Design

### 4.1 The plan's Reset points section (`epic-create`)

- **Plan writing (step 6).** Every plan `epic-create` writes carries a
  **Reset points** section. It names each planned human stop at which a run is
  expected to reset, each with the plan tasks it follows and what the human
  does there. The typical reset point is a release: the human runs the
  `vrg-release` commands that a deployment task (or a later task) needs as a
  human-gated precondition. Human-run deployment steps are the other common
  kind. A plan with no such stop says so in the section, in one line.
- **What a reset point is.** A point where the run has to stop for a human
  action anyway, and after which everything the run needs is durable. It is
  not a phase boundary chosen for its own sake: a reset point with no human
  action at it would interrupt an unattended run for nothing.
- **Plan pushback and alignment (step 7).** While walking the human through
  each plan pushback round and each alignment round, the session checks that
  the plan has the section and that each named reset point sits at a real
  human stop in the delivery table, and raises any planned release or
  human-run step the section leaves out. Alignment rounds can still change
  the plan, so the check runs there too. This mirrors how the walk already
  holds the plan to "Task sizing".
- The section is prose. Its exact layout is the builder's call (§6), within
  the plan's existing structure.
- **The name.** The section is called Reset points, not Checkpoints, because
  `epic-create` already uses "checkpoint" for the end of one of its own
  stages (`vrg-checkpoint` and its checkpoint pauses). The two are unrelated,
  and the skills keep the words apart.

### 4.2 Recommending a reset (`epic-implement`)

- **Reading the reset points.** When step 2 rebuilds state, it reads the
  plan's Reset points section with the rest of the plan. A plan written before
  this epic has none; the run then relies on the judgment rule alone.
- **Reset points added during execution.** The plan changes during execution
  only through its `## Evolution during execution` entries. A release or
  human-run deployment task that an Evolution entry adds is a named reset
  point, as if the Reset points section listed it, and the entry that adds
  it says so.
- **The reset rule.** At a gate where the human must act (run releases, run
  human-run deployment steps, rule on items, or any other "human, do X"
  request), the gate asks for the actions as it does today. The rule applies
  to the **whole gate**: every item the run is blocked on there, not one
  action alone. Once the human has completed every item, the agent has
  recorded everything that came from them (pasted output, a deployment
  outcome, a ruling relayed), **and** no run the session dispatched is still
  in flight, with every finished run's final report taken in and recorded
  (sweep items posted on the bookend, non-review items relayed or listed),
  the agent recommends a reset instead of continuing in the same session:

  ```text
  /clear
  /vergil:epic-implement <epic-ref>
  ```

  An item the human declines or defers (a deployment step not run, a
  ruling put off) is not completed: it blocks the reset at that gate and at
  every later gate until it is ruled or run. A named reset point is then
  skipped, and the agent says at the gate that the reset is skipped and
  which item holds it, so the human knows resets have lapsed.

  Where every item at the gate leaves its own durable record and needs
  nothing pasted back (a release recorded by `vrg-release`, say), nothing
  at the gate is held only in the session, and no run the session dispatched
  is in flight, the gate can say so up front:
  complete these items, then clear and re-invoke rather than saying
  continue. If any item at the gate fails either condition, the reset is
  recommended only after the human reports back and everything is recorded.
  The reset verifies nothing itself: the fresh run rebuilds task state as it
  does today, and a release that was missed or failed surfaces when the task
  that depends on it checks its own precondition.
- **Named reset points and judgment.** At a reset point the plan names, the
  agent always recommends the reset, once the rule's conditions hold. At any
  other human stop it may recommend one under the same rule, on its own
  judgment. In both cases it is a recommendation: the human decides, and may
  continue in the session instead.
- **Never with session-only state.** The agent never recommends a reset while
  the session holds something not recorded elsewhere: evidence the human
  pasted and the agent has not yet recorded, or details shown only in the
  session: the details of a non-review or follow-up security item, which
  exist only in the conversation. A merge-review security item's details are
  kept on disk in the task's `merge-review/security-<ID>.md`, so they count
  as recorded and do not block a reset. A run still in flight also blocks a
  reset: its final report, with its non-review items, their security
  details and its sweep items, comes back only to the session that
  dispatched it. The agent finishes recording first, or, where an item may
  only live in the session, does not recommend the reset at that stop.
- **The batching rationale is recorded at dispatch.** At a batch stop the
  human may ask why the issues were batched, and the answer must come from
  what was known when they were batched. So when the agent dispatches a
  batch, it records in one line why the issues were batched together, in a
  comment on the epic issue (in the `adhoc` mode, on the epic its records go
  to): the comment that already carries the run's items if there is one, or
  a new comment if not. The line is screened against doctrine §1.6 first,
  as a sweep item is: a security-class reason is posted only as
  "security-related", and its details are kept as a non-review security
  item's details are. A fresh run after a reset answers from that record
  rather than reconstructing a reason. Those details are the one exception
  to the session-only rule above: they do not block a reset, and after a
  reset a line posted as "security-related" is answered as that alone. The
  human at a batch stop needs the kind of reason, not its details, so losing
  them in a reset is accepted.
- **`/clear`, never `/compact`.** A clear gives a clean rebuild from the
  durable record. A compacted summary can carry stale or wrong state into the
  run.

### 4.3 What does not change

- How `epic-implement` rebuilds its position, its gate contents, its batching
  and its dispatch rules, except that a batch's rationale is now recorded
  when it is dispatched (§4.2).
- The `epic-create` stage model and its own checkpoint pauses.
- No tool, hook or format changes.

## 5. Delivery outline

- Both changes are guidance edits in `vergil-claude-plugin`: the
  `epic-create` skill (§4.1) and the `epic-implement` skill (§4.2), with the
  site documentation that describes them.
- The work builds on epic vergil-project/.github#1384, and its
  implementation tasks start once that epic is finished, since the gates
  this design resets at are the ones a #1384 run still stops at.
- No release, deployment or validation task is needed to prove it: the
  change is guidance, exercised by the next epic planned and run with it.
  Whether the plugin release that ships it needs a deployment task is the
  plan's call.

## 6. Left to the builder

- The Reset points section's exact layout and position in the plan, and the
  wording `epic-create` uses to ask for it.
- The wording of the reset recommendation at each kind of gate, and where in
  `epic-implement`'s step 4 the rule is stated.
- Whether the two skill changes land as one PR or two.
