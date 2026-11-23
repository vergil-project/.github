# checkpointed-epic-create: a restartable, staged epic-create — design

- **Epic:** vergil-project/.github#1546
- **Planning issue:** vergil-project/.github#1493 (review history is in its comments)
- **Status:** draft, in design review

## 1. Intent

`epic-create` runs in one long main session. Its review rounds already run in
fresh subagents, but the orchestrating session itself accumulates everything:
the brainstorm, every round's findings walk, the plan, and every task filed.
On small epics that comes to 300,000–400,000 tokens. On large ones (70–80
tasks, complex designs, many decisions) the session gets close to
auto-compaction.

Most of that context is no longer needed. By the time a later decision is
made, the earlier ones are already recorded in planning-issue comments, issue
state and committed documents. Once a run's state is durable, the session
should be able to drop its context and pick the run up again from the record
alone.

This epic makes `epic-create` an explicit sequence of **stages**. Each stage
ends at a **checkpoint** where the run is fully recoverable. At every
checkpoint the skill pauses, summarizes what the stage did, and recommends a
`/clear` followed by `/vergil:epic-create <planning issue>`. The human decides
whether to clear. A resumed run finds the stage to enter from a stage
checklist in the planning issue's body, and rebuilds what it needs from the
recorded evidence.

The motivation is reasoning quality and token cost on long runs.
Recoverability after an interrupted or idle session comes as a side benefit.
`epic-implement` adopts the same mechanism next, under
vergil-project/.github#1544.

## 2. Goals, non-goals, anti-goals

### Goals

- Every stage boundary of `epic-create` is a checkpoint from which a fresh
  session, after `/clear`, resumes with no loss of decisions or work.
- Re-running `epic-create <planning issue>` at any point is idempotent: a stage
  that completed is not redone, and a stage interrupted partway is safe to
  re-enter.
- A fresh session is **told** which stage to enter by a tool, not left to infer
  it from prose.
- The mechanism is generic enough for `epic-implement` (#1544) to adopt
  without a second tool.

### Non-goals

- Forcing a clear. The skill recommends it; the human decides, usually by the
  size of the context.
- Checkpoints inside the brainstorm. The brainstorm is one unit of work: an
  interrupted brainstorm restarts.
- Resuming a run on a different machine. The docs worktree's commits are
  local until report-ready pushes the branch.
- Changing `vrg-release`'s own skew guard. That is
  vergil-project/.github#1545.
- Changing the review doctrine's round procedure. Rounds already checkpoint
  through their round comments (§4.5).

### Anti-goals

These are failures even if the mechanism technically works.

- A resume that silently skips work, or silently redoes work that created
  something (a second epic, duplicate tasks).
- A dead-end refusal. When the recorded state and the current tooling
  disagree, the human is present by definition, and the run must offer a way
  forward (§4.4).
- A resumed session acting on untrusted content. The stage checklist is read
  only through the trust layer (§4.7).

## 3. Approaches

Three ways to record where a run stands were considered.

- **A. A tool-owned stage checklist in the planning issue's body** (chosen).
  This is the resumable-release model (vergil-project/vergil-tooling
  `docs/specs/2026-06-11-resumable-release-design.md`). A command owns a
  delimited checklist block, ticks a stage only once its work is durable, and
  names the next stage. The cursor is exact, and the same command serves #1544.
  The cost is one more `vergil-tooling` command and changes in two repos.
- **B. Infer the stage from existing evidence, plugin only.** The skill would
  gain a resume table mapping markers, round comments and commits to stages.
  This is the cheapest to build, but it leaves the judgment-heavy inference to a
  fresh session, which is the step this epic exists to make reliable. Some
  stage ends, such as "the human stopped spec pushback", leave no evidence of
  their own.
- **C. A tool that computes the stage from evidence, with no stored cursor.**
  Nothing stored can drift. It meets the same gap as B, though: the human's
  decisions to end a stage are not observable, so it would need markers anyway,
  at which point it is A without the checklist.

The human chose A, with one change from the release model: a mismatch between
the recorded checklist and the current stage list is reported and reconciled,
not refused (§4.4).

## 4. Design

### 4.1 The command

A new `vergil-tooling` command, `vrg-checkpoint` (the name is the builder's
call), manages the stage checklist of one tracking issue for one named
workflow. `epic-create` is the first workflow. `epic-implement` adds the second
under #1544.

Its operations:

- **`init <ref> --workflow <name>`:** write the checklist block for the
  workflow's current stage list into the issue body, all stages unticked, or
  with the stages named by `--tick` already ticked (§4.8). It refuses if a
  block is already present.
- **`status <ref> --workflow <name>`:** print the stages with their ticks
  and the **next stage**, the first unticked one. `--json` gives the same
  data for the skill. This is how a fresh session finds where it is.
- **`tick <ref> --workflow <name> <stage>`:** tick one stage. It refuses a
  stage that is not in the block, and refuses to tick a stage while an
  earlier one is unticked, because stages are linear.
- **`reconcile <ref> --workflow <name> [--tick <stage>]…`:** rewrite the
  block to the current stage list after skew (§4.4).
- **`reset <ref> --workflow <name>`:** remove the workflow's block, so the
  next `status` reports it missing. It removes every closed block of the
  workflow, each from its begin marker to its end marker, malformed or
  duplicate ones
  included, and succeeds, writing nothing, when there is none. A begin
  marker with no end marker is refused as malformed, with nothing written,
  since where that block ends is unknown. Abandoning a skeleton ends with
  it (§4.3).

`--workflow` is required on every operation, so one issue body can carry
the blocks of several workflows, each rewritten only by its own operations.

Each write rewrites only the delimited block. Every other part of the body is
left as it is, including the `Planning:` lines, the security lines, and the
human-written description.

Each workflow's stage list is defined once, in `vergil-tooling`, and the block
is generated from it, so the recorded names and order cannot drift from the
tool. The existing release checklist module
(`vergil_tooling/lib/release/checklist.py`: render, parse, upsert, tick) is
already caller-agnostic except for its block markers and its lenient parse
(§4.2). Generalizing it, so that
`vrg-release` and the new command share one implementation, is the builder's
call.

### 4.2 The block

```markdown
<!-- vrg-checkpoint:epic-create -->
- [x] bootstrap
- [x] brainstorm
- [x] spec-pushback
- [ ] spec-review
- [ ] plan
- [ ] plan-pushback
- [ ] alignment
- [ ] file-tasks
- [ ] docs-pr
<!-- /vrg-checkpoint:epic-create -->
```

- A stage line holds only the checkbox and the stage name. Any other line
  inside the block, or any text after a name, makes the block malformed:
  `status` and every write report it with its own exit code, distinct from
  skew, and change nothing. The one exception is `reset`, which removes
  a malformed closed block (§4.1). Per-stage detail belongs in progress comments.
  This strict parse is a `vrg-checkpoint` requirement. The release
  checklist's parser skips lines it does not match, and `vrg-release` keeps
  that behaviour unless the builder decides otherwise, so sharing the module
  must not change what `vrg-release --resume` accepts.
- It is a real GitHub task list, so a human can tick a box in the web UI to
  record a stage done by hand. The next `status` treats it as done.
- The block is public, and its stage names are neutral, so a security run
  (step 0's public-wording rule) can carry it unchanged.

### 4.3 The stages of epic-create

The planning issue's step 0 (resolve the entry point, the security check, the
`in-progress` label) runs on every invocation and is not a stage. It always
ends with `status`; there is no separate first-run path. When `status`
reports the block missing, the skill runs the evidence inference of §4.8. On
a planning issue with no evidence of any stage, that inference ticks nothing,
and the skill runs `init` without asking the human: that is a first run. The
run then enters the stage `status` names, after posting a "Resuming at
`<stage>`" line in a progress comment on a resume.

A planning issue already closed is handled before the label is added. If
`status` names `docs-pr` and the documentation task is closed as completed,
the docs PR's merge has closed it: the skill adds no label, posts the
`Planned as` comment if it is missing, and ticks `docs-pr`. Any other closed
planning issue stops the run for the human.

An open planning issue with every stage ticked has been handed off. The
skill runs `vrg-queue planning`. If it shows the item `ready-to-close` (the
docs PR merged but did not close it), the skill closes it as step 10 says.
Otherwise it reports the item's note (handed off and awaiting the merge, or
the `held:` note) and exits, changing nothing.

Each stage below lists the **evidence** its checkpoint leaves (what a later
stage reads back) and what **re-entry** does when the stage was interrupted
before its tick:

- **`bootstrap`.** Creates the epic's skeleton from the planning issue alone,
  before any design work: the epic, with a placeholder body drawn from the
  planning issue and its `Planning:` line; the documentation task, which the
  docs PR closes; the epic's Documents section, seeded with
  `vrg-epic-docs set`, all three entries pending; and the docs worktree on
  the documentation task's branch. Nothing else: `bootstrap` is kept
  minimal, and every other task is filed in `file-tasks`. The slug is chosen
  here, from the planning issue's title, and is not changed later. Ends when
  all of these exist and the progress comment carrying the `Epic:` and
  `Docs task:` markers is posted. Evidence: those markers, the documentation
  task under the epic, the Documents section in the epic body, and the
  branch. A marker naming an issue closed as not planned is an abandoned
  skeleton's, and is not evidence. Re-entry: each creation checks its
  evidence first and creates only what is missing.
- **`brainstorm`.** Ends when the design is approved and the spec draft is
  committed at `epics/<N>-<slug>/spec.md` on the documentation task's
  branch, and the epic body is written from it. **Writing the epic body** is
  a one-time step, separate from the epic body sync: it replaces the
  placeholder body with one drawn from the spec, keeps the `Planning:` line,
  leaves the Documents section to `vrg-epic-docs`, and follows step 0's
  public-wording rule. The epic body sync, unchanged, runs at every later
  spec or plan change. Evidence: the spec commit and the written body.
  Re-entry: if the spec is already committed, only the body is written;
  otherwise the brainstorm restarts. It creates no issue, so a restart is
  safe.
  - **The early exits clean up the skeleton.** Three outcomes end a run
    early: the work is a single task, it is not worth doing, or it must be
    deferred or dropped. Each first runs one shared step, **abandoning the
    skeleton**: the skill closes the epic as not planned first, then the
    documentation task as not planned, each with a comment pointing to the
    outcome, removes the docs worktree and its branch, and last runs
    `reset` on the planning issue. The epic goes
    first because the epic rollup closes an epic as completed once all its
    children are closed. Only then does it
    finish the exit: file the single task and close the planning issue,
    close the planning issue as not planned, or route it with
    `vrg-intake-route … defer|drop`, which refuses while the epic is open.
    `epic-review planning`, deferring or dropping a planning item that has
    a skeleton, abandons the skeleton the same way before it routes the
    item.
  - **A revived item starts afresh.** A planning item revived after its
    skeleton was abandoned has no block, so it starts at `bootstrap` and
    creates a new epic and a new documentation task, posting both markers,
    which are then the latest. The old `Epic:` and `Docs task:` markers name
    issues closed as not planned, so `bootstrap` ignores them. If step 0
    finds that the epic the latest trusted `Epic:` marker names is closed as
    not planned while the block still exists (an abandon interrupted before
    its `reset`, or an item deferred by another route), the skill shows the
    human and, once confirmed, finishes the abandon step, each part
    skipping what is already done, then starts at `bootstrap`.
  - **The closing bookends are filed in `file-tasks`.** The
    documentation-review task belongs in the repo where most of its sweep
    lands, which the design decides, and the retrospective task has no
    reason to exist earlier. Each carries a fixed key in place of a plan
    task ID (§4.5), `doc-review` and `retrospective`, so re-entry finds
    them like any other task.
  - **Operational and follow-on tasks are plan tasks.** Deployment and
    validation tasks, the default cold-rebuild validation, and any
    follow-on brainstorm bookend depend on the design, so each appears in
    the plan with a plan task ID and is filed in `file-tasks`. The skill's
    seeding of them at epic creation is removed.
  - **A security run keeps the skeleton neutral.** `bootstrap` publishes
    before the step 1 human gate, so the epic's title and placeholder body
    and the documentation task's title say nothing about the design, under
    step 0's public-wording rule. The human gate still comes before the spec is
    written. A "No" at the gate keeps the neutral skeleton as it is, posts
    the neutral pause comment, and publishes nothing further; a later "Yes"
    continues the stage.
- **`spec-pushback`.** Ends when the human stops the stage. Evidence: the
  round comments (§4.5) and the round commits. Re-entry: when the latest
  round has its round comment, re-entry starts at that round's step 7,
  asking the human whether to run another round or stop the stage. Only on
  the human's answer does it dispatch the next round, or tick the stage.
  Re-entry never dispatches a round the human has not asked for.
- **`spec-review`.** Ends when the human's spec edits are committed and the
  human says the review is done. Evidence: the commit, or a progress comment
  recording that there were no edits. Re-entry: show the spec and continue
  the review.
- **`plan`.** Ends when `plan.md` is committed. Evidence: the plan commit.
  Re-entry: rerun `writing-plans`, which is automated.
- **`plan-pushback`** and **`alignment`.** As `spec-pushback`.
- **`file-tasks`.** Ends when every task in the plan is filed, the
  documentation-review and retrospective tasks exist, every
  advisory is retargeted, and `Tasks filed: yes` is posted. Evidence: the
  filed tasks, each carrying its plan task ID (§4.5). Re-entry: file only
  the tasks that are missing.
- **`docs-pr`.** Ends when report-ready has recorded the handoff and the
  handoff comment is posted. Evidence: the report-ready record and the
  `Planned as` comment. Re-entry: if the docs PR has merged, finish as for a
  closed planning issue (above); otherwise re-run report-ready, which only
  rewrites the prose.

The epic body sync, the Documents section, the security-run retargets and the
planning-issue close keep their current rules. They move into the stage whose
work triggers them; the stages do not change what they do.

**A dirty worktree on resume** belongs to the stage `status` names. Before
the stage continues, the skill shows the human the diff, and the human
either commits it as that stage's commit or discards it. In a review stage
this is the doctrine's rule for an interrupted round
([§2.4](https://github.com/vergil-project/vergil-claude-plugin/blob/develop/docs/paad-review-doctrine.md#24-output-checks-and-broken-rounds)):
the diff is that round's commit, with its round comment. In `spec-review`
it is the human's spec edits; in `plan`, a partial `plan.md`.

**The order at a checkpoint:** make the stage's work durable (commit, file,
comment), post the stage's progress comment, then `tick`. A tick is the last
write, so a session lost between the work and the tick leaves the stage
unticked, and re-entry redoes or completes it. As in the release model,
redoing beats skipping, and every re-entry above is safe to repeat.

The existing markers (`Epic:`, `Docs task:`, `Tasks filed: yes`), the round
comments and the planning queue's three states (`waiting`, `in-progress`,
`ready-to-close`) stay as they are. `vrg-queue planning` does not read the
block.

### 4.4 Skew

A tooling release can land in the middle of a run, and a run can sit for days.
So the recorded block may not match the workflow's current stage list.

- `status` detects it (different names, a different order, or a missing or
  extra stage), prints the difference, and exits with its own exit code,
  distinct from success, failure and an untrusted body.
- The skill shows the human the difference and asks how to proceed. Then it
  runs `reconcile`, which rewrites the block to the current list:
  - a stage whose name matches keeps its tick;
  - a removed stage is dropped and reported;
  - a new stage is unticked unless the human says it is done, which the skill
    passes as `--tick <stage>`. A new stage that sits before the cursor is
    always the human's call: run it, or tick it.
- `reconcile` records what it changed in a comment on the issue, so the log
  shows it.

There is no refusal without a way forward, and no silent remapping: every
change to the cursor is shown to the human before it is made.

### 4.5 Inside a stage

**Review rounds.** A round is already a checkpoint: its commit and its round
comment are durable, and the doctrine rebuilds the stage and the round count
from the latest trusted `Design review: <stage> round <n>` comment
([paad review doctrine §2.6](https://github.com/vergil-project/vergil-claude-plugin/blob/develop/docs/paad-review-doctrine.md#26-the-round-comment)).
So the skill pauses with the checkpoint summary (§4.6) after every round's
comment, not only at the end of the stage. The doctrine's step 7 question
(another round, or stop the stage) comes after the pause, on continuing or
on re-entry. A round interrupted before its
comment is re-run, after the doctrine's dirty-worktree handling
([§2.4](https://github.com/vergil-project/vergil-claude-plugin/blob/develop/docs/paad-review-doctrine.md#24-output-checks-and-broken-rounds)).
No change to the doctrine is needed. The round's findings walk is in the main
session until its comment posts, and that is accepted: a round is short enough
to redo.

**Task filing.** A large plan files tens of tasks, so a clear halfway through
must be safe. Each filed task carries a stable key, its plan task ID (for
example `Plan task: 13a`, the builder's call on the exact form). On re-entry the
skill lists the tasks already under the epic and under its proxy epics, reads
their keys through the trust layer, and files only the plan's tasks that are
missing. A task that `vrg-issue-create` created but failed to link is
resolved before any checkpoint pause: the skill stops, and the task is linked
under its epic or recorded in a progress comment, so a re-entry never files
it a second time. Security retargets keep their own idempotency (re-run the identical
command). The skill pauses for the checkpoint summary only at the end of the
stage, not after every task.

### 4.6 The checkpoint pause

At every checkpoint (each stage end, and each review round) the skill stops
and gives the human:

- a short summary of what the stage or round did, and the decisions made in
  it;
- the next stage, or the next round;
- a recommendation to reset, with the two commands ready to paste:

  ```text
  /clear
  /vergil:epic-create <owner>/<repo>#<M>
  ```

The human then either runs them or tells the skill to continue in the same
session. Continuing enters the next stage as a resume would, from `status`, so
both paths run the same code.

### 4.7 Trust

The block lives in the planning issue's body, which a resumed session reads
only through `vrg-issue-read`. An untrusted body (exit 3) or a read failure
(exit 1) stops the run for the human, as it does today. `vrg-checkpoint` reads
the body through the same trust layer and refuses an untrusted one with its own
exit code. A human's tick in the web UI is a body edit by a trusted identity,
so it stays trusted. A tick by an outsider makes the body untrusted, and the run
stops.

### 4.8 Runs already in flight

A planning issue whose run started before this ships has no block. On the
first resume after the upgrade, `status` reports it missing, as for every
first run (§4.3). The skill then
works out which stages are complete from the existing evidence (markers, round
comments, commits, filed tasks), shows the human its proposal, and runs `init`
with `--tick` for each stage the human confirms. From then on the run is
ordinary.

A run caught partway through `file-tasks` has tasks filed with no key. Before
it files anything, the skill matches each task already under the epic and
its proxy epics to a plan task ID (or to `doc-review` or `retrospective`),
shows the human the
matches, and adds the key to each task the human confirms. Only then does
re-entry file what is missing.

## 5. Delivery outline

The plan sets the tasks. The expected shape:

1. `vergil-tooling`: `vrg-checkpoint` with the `epic-create` stage list, tests,
   and its docs.
2. `vergil-claude-plugin`: `epic-create` restructured into the stages of §4.3,
   with `status` on entry, a tick at each checkpoint, the checkpoint pause,
   skew handling, plan task IDs in filed tasks, and the in-flight adoption of
   §4.8.
3. A release of each repo, and a deployment task that installs both. Some of
   its steps (`vrg-host plugins`, `vrg-vm update --all`) are human-run.
4. A validation task: a live planning run that clears and resumes at several
   checkpoints, at least one inside `spec-pushback` and one inside
   `file-tasks`, and finishes with nothing duplicated or lost.
5. The bookends already filed: the documentation review
   (vergil-project/vergil-claude-plugin#1293) and the retrospective
   (vergil-project/.github#1548).

## 6. Left to the builder

- The command's name, its exit codes, and its JSON shape.
- Whether to generalize the release checklist module or write a sibling.
- The exact form of the plan task ID line, and where the plan records the IDs
  (its delivery table already lists tasks by ID).
- The wording of the checkpoint summary.
