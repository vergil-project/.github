# checkpointed-epic-implement: a compaction-safe, clear-friendly epic-implement — design

- **Epic:** vergil-project/.github#1596
- **Planning issue:** vergil-project/.github#1544 (review history is in its comments)
- **Status:** approved

## 1. Intent

`epic-implement` drives an epic's tasks in one orchestrating main session. Each
task's work runs in a subagent, but the orchestrator accumulates every task's
results, gate outcomes and rulings, so its context grows with the epic. Once
agents submit and finalize their own PRs (epic vergil-project/.github#1384)
and `epic-implement` continues from one task to the next
(vergil-project/vergil-claude-plugin#1206), one run covers more tasks and the
growth is worse.

The goal is to keep the orchestrator on the minimal context it needs, for
better reasoning on long runs and lower token cost. Recoverability after an
interrupted session comes as a side benefit.

`epic-create` met the same goal with checkpointed stages and a recommended
`/clear` at each one (epic vergil-project/.github#1546). That model does not
carry over, for two reasons:

- **`epic-implement` has no fixed sequence.** It loops over a changing
  frontier of tasks, so a fixed, ordered stage checklist (`vrg-checkpoint`)
  has no stable "next stage" to record.
- **No one is there to clear.** Under #1384 the run is meant to go unattended,
  stopping for the human only on a ruling, an approval, a failure or a
  release. An agent cannot run `/clear` or `/compact` itself: both are
  commands the user types
  ([commands](https://code.claude.com/docs/en/commands.md)). The one context
  reset that happens unattended is **auto-compaction**, near the context
  limit ([model config](https://code.claude.com/docs/en/model-config.md#set-the-auto-compact-window)).

So this epic does not add checkpoints. It makes the run **safe to compact or
clear at any task boundary**, keeps the orchestrator's context growing slowly,
and recommends a clear at the human gates, the only points where a human is
present to run one.

## 2. Goals, non-goals, anti-goals

### Goals

- Nothing the run needs lives only in the orchestrating session. A clear or an
  auto-compaction at any task boundary loses no decision, ruling or security
  detail.
- After a compaction, or on re-invocation after a clear, the run rebuilds its
  position from the durable record, not from a summary.
- The orchestrator's context grows slowly: subagents do the work and report
  briefly.
- At each human gate, once the gate is fully recorded, the run recommends a
  clear with the commands ready to paste, in the shape of `epic-create`'s
  checkpoint pause.

### Non-goals

- A stage checklist or any other stored cursor for `epic-implement`.
  `vrg-checkpoint` gains no workflow.
- A compaction hook (`PreCompact`, or `SessionStart` with source `compact`).
  A plugin hook fires in every session, not only in `epic-implement` runs.
- An outer driver that starts a fresh headless session per frontier pass.
- Forcing a clear, or pausing an unattended run for one.
- Changing the `adhoc` mode beyond what the shared guidelines say of it.

### Anti-goals

- A security detail lost to a clear or a compaction, or one written anywhere
  public to avoid losing it.
- A clear recommended while something at the gate exists only in the session.
- A resumed or compacted run that acts on a remembered state instead of the
  re-derived one (a task dispatched twice, a ruling applied twice).

## 3. Approaches

- **A. Guidelines in the skill, plus durable security details** (chosen). The
  `epic-implement` skill states how the run keeps its state durable, behaves
  after a compaction, keeps its context thin and recommends a clear at the
  gate. The one piece of state that today reaches the orchestrator only in a
  run's final message (a security item's details) moves to the task's local
  state directory. The cost is skill text in two skills; no tool changes.
- **B. A `vrg-checkpoint` block for `epic-implement`.** Rejected: the block
  records a fixed, ordered stage list, and the run's frontier is neither fixed
  nor ordered.
- **C. A fresh session per frontier pass** (an outer driver around
  `claude -p`). Rejected as out of proportion: it is a new execution
  architecture, overlapping #1384's unattended goal, for a problem the
  guidelines and compaction already address.

For the security details, two options were weighed:

- **Write them to the state directory** (chosen). Safe against an unattended
  auto-compaction, which is the reset that actually happens.
- **Only never recommend a clear while a security item is unruled.** Smaller,
  but a compaction before the gate would still lose the details.

## 4. Design

### 4.1 Durable state only

The skill states the rule: everything the run needs to continue is on GitHub
(the epic, its tasks and their comments, the plan) or in a local state
directory
([doctrine §1.5](https://github.com/vergil-project/vergil-claude-plugin/blob/develop/docs/paad-review-doctrine.md#15-state-directories)).
Most of it already is: task states, the plan, batch markers, merge-review
directories and ruling requests, non-review items under `Awaiting a ruling`,
ruling lines, documentation items on the documentation-review bookend, and
the session's own follow-ups in comments on the epic.

The exception today is a **security item raised by an `issue-implement`
run** as a non-review item: only its count is posted, and its details reach
the orchestrator only in the run's final message. §4.2 moves them to the
state directory. Security items from merge review
(`merge-review/security-<ID>.md`) and from `issue-validate` and
`issue-deploy` (`operational/<N>/`) already live in state directories; the
builder confirms this and closes any other gap of the same kind it finds.

### 4.2 Security details in the state directory

A run dispatched by `epic-implement` that raises a security non-review item:

- writes its details to `security-<i>.md` in the task's state directory,
  `build/.vergil-state/issue-implement/<N>/` in the task repo's main clone,
  under §1.5's rules for creating and checking it, and records the file's
  hash;
- names the file and its hash in its final report, never the details.

At the gate, `epic-implement` shows the human the directory's listing, and
once the human confirms it, checks each file against its hash and reads it to
walk the item, in the terminal only. The ruling is relayed as today. The file
is deleted once the item's ruling is recorded; the builder fixes the exact
point and the naming of `<i>` so that it does not collide with the `record`
ruling's existing `security-<i>.md` files in the same directory.

A run with the human present (not dispatched by `epic-implement`) keeps
today's behaviour: the details stay in the conversation and are ruled on
there.

### 4.3 Compaction-safe

After an auto-compaction, the summary is not a source of state. The run
re-derives its position from step 2 (the epic, `vrg-epic-audit`, the plan)
and the step 3 re-entry checks (batch markers, merge-review directories,
ruling replies, the state directories) before it dispatches anything. The
same holds for a re-invocation after a clear, which the skill already
supports.

The skill does not detect a compaction. The rule is that at every task
boundary, before a dispatch, the run works from the re-derived state, so it
is right whether or not a compaction happened.

### 4.4 A thin orchestrator

- Each task runs in a subagent, as today.
- A subagent's final report is short: the outcome, the refs, and pointers to
  durable records (comments, state-directory files), not the run's working
  detail.
- The orchestrator does not read task bodies, diffs or logs it does not need
  to decide the next step; the trust-layer pre-dispatch read already discards
  its output.

### 4.5 The clear recommendation at the gate

When the run stops at a human gate (step 4), and once everything at the gate
is recorded (each item is on GitHub or in a state directory, per §4.1), the
gate's summary ends with a recommendation to clear before continuing, with
the commands ready to paste:

```text
/clear
/vergil:epic-implement <epic-ref>
```

In the `adhoc` mode, the second command is the one that started the run
(`/vergil:epic-implement adhoc [--repo <owner>/<repo>]`). The human clears or
continues in the same session; either way the run re-derives its state
(§4.3). The run never pauses solely to recommend a clear.

The wording follows the boundary that #1206 sets: by then the gate is where
the run stops for a ruling, an approval, a failure or a release, not where
PRs wait for the human to submit them.

### 4.6 Shared with epic-create

What `epic-implement` shares with `epic-create`'s checkpoint model is the
pause's shape (a summary, then the two commands) and the resume rule (a
continued session and a resumed one run the same code, from the durable
record). It does not share the stage checklist.

## 5. Delivery outline

The plan sets the tasks. The expected shape, all in
`vergil-project/vergil-claude-plugin`:

1. `issue-implement`: security details of a dispatched run's non-review items
   go to the state directory (§4.2). Not blocked by #1206.
2. `epic-implement`: the guidelines of §4.1 and §4.3–§4.5, and the gate's
   reading of the security files (§4.2). Blocked by task 1 and by
   vergil-project/vergil-claude-plugin#1206, since both edit the skill's gate
   and stop sections and the guidelines describe the run as #1206 leaves it.
3. The bookends: the documentation review, in the plugin repo, and the
   retrospective, in `.github`.

No deployment or validation task: the change is skill text that the
pipeline's checks cover.

## 6. Left to the builder

- The exact wording of each guideline and where it sits in the skill.
- The naming of the security files and when each is deleted (§4.2).
- Whether any other state reaches the orchestrator only in a run's final
  message, and its fix (§4.1).
