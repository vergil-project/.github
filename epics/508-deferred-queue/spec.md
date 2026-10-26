# Deferred queue: park evaluated items apart from intake and planning

- **Epic:** vergil-project/.github#508
- **Planning issue:** vergil-project/.github#487
- **Status:** Draft design (2026-10-26), before pushback
- **Brainstorm source:** superpowers brainstorming session, 2026-10-26
- **Implemented in:** vergil-tooling, vergil-claude-plugin, then a migration
  sweep of every queue home

## 1. Summary

A **deferred** item is one that has been evaluated and deliberately put on
hold because some outside condition is not yet met. For example, the MkDocs
move (#436) waits until either MkDocs 2.0 or Zensical reaches GA. Today
"deferred" is only a comment (`Deferred <date>: <reason>`). The item stays in
the intake or planning queue next to items nobody has evaluated yet, so:

- **The queues are cluttered.** Most of the vergil-project intake queue is
  deferrals, and a new item cannot be spotted among them.
- **Nothing brings an item back.** The reason and its trigger are prose that
  nothing rereads.

This epic adds a third **queue epic**, `Epic (deferred): <name>`, alongside
intake and planning in every queue home. **Being in the deferred epic is what
makes an issue deferred.** Every deferral records why it is waiting and what to
check to know when it can come back. Items come back through **review**: a new
`deferred-review` skill re-checks each item. A new `planning-review` skill does
the same job for the planning queue. A one-time **migration sweep** moves
everything deferred today into the new epics, so intake and planning hold only
live work.

The deferred queue becomes the real backlog. It is not a graveyard: a deferral
is **temporary by definition**. If the reason were permanent, the item would be
closed.

## 2. Goals, non-goals, anti-goals

### Goals

- Deferred items are visible on their own and apart from unprocessed (intake)
  and to-be-planned (planning) items.
- Every deferral states its reason and its revival conditions.
- All three queues have a review skill that processes them with the human.
- After the sweep, intake and planning hold no deferred items in any queue
  home.

### Non-goals

- **Scheduled reviews.** The human plans start/end-of-day, week, month and
  quarter routines that will prompt interactive runs of the review skills.
  That is a separate project. This epic makes the reviews exist and work when
  run by hand.
- **Structured or machine-checked revival criteria.** Conditions are prose,
  such as "Once MkDocs 2.0 goes GA, re-evaluate this." No date parsing, no
  condition DSL, no automatic revival. A reviewing agent reads the conditions
  and makes the judgment call with the human.
- **Deferring or suspending development epics or their tasks** (see §3.2).

### Anti-goals

- A deferral without a stated reason and revival conditions. That is how
  today's queues filled up with items nobody could evaluate.
- The deferred queue turning into a place where items go to die. Deferred
  review must actually move items out when their conditions are met.

## 3. Design

### 3.1 Container epics and development epics

Two kinds of epic matter here:

- **Container epics** are perpetual standing epics that hold unrelated,
  standalone issues: **intake**, **planning**, **deferred** and **ad hoc**.
  Their items move between them independently.
- **Development epics** are expanded, finite epics whose tasks depend on each
  other and on the epic's goal.

### 3.2 Which items can be deferred

Only items in a **container epic** can be deferred: an intake, planning or
ad-hoc child. A task in a **development epic** is refused, because deferring
one task of an epic in effect defers or suspends the whole epic. That is a
separate, larger problem that this epic explicitly does not solve. An epic
itself cannot be deferred either. An unparented issue is refused too; the audit
attaches orphans to intake, and they are deferred from there.

The tooling enforces this rule. It is not left to the agent's judgment: the
refusal names the item's parent and says why.

A **planning item mid-`epic-create`** (one with the `in-progress` label) is a
special case:

- If an `Epic:` marker is recorded, a development epic already exists. The
  defer is refused, because that would be suspending an epic.
- Otherwise the `in-progress` label is removed as part of the defer.

### 3.3 The deferred queue epic

Add a `DEFERRED` entry to the standing-epic kinds in `lib/epics.py`, next to
`ADHOC`, `INTAKE` and `PLANNING`:

| Field | Value |
| --- | --- |
| key / label | `deferred` (added to `data/labels.json`) |
| sentinel | `deferred` |
| title | `Epic (deferred): <name>` |
| archive | `Archive (deferred): <name> — <YYYY>-Qn`, labels `archive` + `deferred` |

It follows the existing queue-home rule (`queue_home`): a public repo's items
share `<org>/.github`'s deferred epic, named for the org, and a self-homed
private repo has its own. It also inherits everything the other queue kinds
get:

- Quarterly archives and the closed-child drain (`drain_standing_org`,
  `drain_standing_child`).
- The visibility-flip relocation in the audit.
- Exclusion from the strategic roadmap.

There is **no per-item label**. Membership is the state, exactly as with intake
and planning.

**In and out only through the route.** `vrg-epic-move` and
`vrg-issue-create` refuse the `deferred` sentinel and a deferred epic's `#N` as
a target. `vrg-epic-move` also refuses to move a task *out of* a deferred epic.
Both refusals point to `vrg-intake-route`. That way no item becomes deferred
without its deferral comment, and none is revived silently.
*(Builder's call, recorded here.)*

The live queue epics are not covered by the 100-sub-issue cap overflow that
archives have. A deferred queue that size is out of scope for now.

### 3.4 The deferral record

`defer` takes two required arguments, `--reason` and `--revisit-when`, and
posts one comment:

```text
Deferred <YYYY-MM-DD>: <reason>
Revisit when: <conditions>
Deferred from: <owner>/<repo>#<N>
```

- **reason:** why the item is waiting.
- **conditions:** what to check to know when it can come back. This can be a
  date ("re-research by 2027-01-26"), events ("Zensical 1.0 ships",
  "vergil-python#50 ships"), or both. It is free prose.
- **Deferred from:** the container epic the item left, so deferred review knows
  where a revived item came from. The tool writes this line; it is omitted on a
  refresh, where the item doesn't move. *(Builder's call.)*

The line prefixes are **fixed** so the queue view and reviewers can find the
latest deferral. The content is never parsed for meaning.

An optional `--date YYYY-MM-DD` overrides the stamped date. The migration
sweep (§3.9) uses it to keep each item's original deferral date. *(Builder's
call.)*

### 3.5 Routing: `vrg-intake-route`

`vrg-intake-route` stays the one command for moving items between queues. Its
eligibility widens from "intake child only" to the route table below. It keeps
its name to avoid churning every skill and doc. *(Builder's call; a rename is
out of scope.)*

| Item is in | Allowed routes |
| --- | --- |
| intake | drop, **defer** (now moves the item), one-off, planning, split |
| planning | **defer**, **drop**: both refused when an `Epic:` marker is recorded (§3.2) |
| ad hoc | **defer**: refused while an open PR would close the item, since §3.7 would strand that work |
| deferred | **defer** (refresh: new comment, no move), **intake** (new), planning, one-off, drop, split |
| development epic, or unparented | refused (§3.2); `planning` keeps accepting an unparented issue for the `epic-create` free-text path |

- **defer** reparents the item from its container epic to its queue home's
  deferred epic, posts the §3.4 comment, and removes `in-progress` where §3.2
  allows it. Every existing guard still holds: same org only, and a private
  item is never routed into a public repo or epic.
- **intake** (new route) reparents a deferred item back to its home's intake
  epic, for when a revived item needs a fresh look rather than a known
  destination.
- **one-off from deferred** covers a revived ad-hoc item going back under its
  repo's ad-hoc epic, using the existing transfer-or-reparent mechanics.
- **drop** keeps the existing comment-and-close behavior. The comment wording
  should name the queue it was dropped from, not always "intake review".

### 3.6 Viewing: `vrg-queue deferred`

`vrg-queue` gains a `deferred` view with the same output shape, `--repo` and
`--json` as the others. Every item is `waiting`. Two differences from intake:

- **The note is the latest deferral**, meaning its reason and `Revisit when:`
  conditions. It is not just the first line of the latest comment.
- **Ordering is least recently deferred first:** the oldest latest-`Deferred`
  date comes first. A refreshed item sinks to the bottom. A **keep** writes
  nothing, so a kept item stays near the top until its conditions change or it
  is refreshed. That is intended: a review normally goes through the whole
  deferred queue, so the order is a reading aid, not a rotation.
  *(Builder's call.)*

`vrg-queue` with no queue name, or any summary that lists queues, should now
report all three.

### 3.7 Not PR-workable

A deferred item is not buildable. `queue_kind` / `queue_item_kind` return
`"deferred"`, and `vrg-pr-workflow report-ready` and `vrg-submit-pr` refuse it
with the shared refusal message: "…it is deferred until its revival conditions
are met. Revive it with deferred-review first." `vrg-epic-audit` treats deferred
children like other queue items: they are not PR-tracked tasks, and it reports
them with "Run deferred-review."

### 3.8 The review skills (vergil-claude-plugin)

All three reviews share one idea: **a review processes a queue item by item,
with the human making the calls.** For any item, "leave it here and look again
next time" is a valid outcome. It is the human's call, and it fits things that
will be ready within days or a week. Each review uses `intake-review`'s pattern:
list the queue, check each item against the current state before recommending,
present items in batches with one confirmation per batch, and apply each
decision with `vrg-intake-route`.

- **`intake-review` (updated).** Defer now moves the item out of intake and
  needs both a reason and revival conditions. The skill's "defer keeps its age
  and comes up first next time" behavior goes away. Leaving an item in intake
  is the "look again next time" outcome.
- **`planning-review` (new).** Its main job is to decide, with the human, **what
  to plan next**. It ranks the planning queue by priority and importance and
  picks one or more items to hand to `epic-create`. Along the way it can defer
  an item whose blockers are significant, drop one that is no longer needed, or
  leave one for next time. It carries the planning doctrine: **plan an epic only
  when you mean to implement it soon.** A plan left sitting for months goes
  stale and needs a conceptual rebase onto everything built in the meantime, so
  a pile of planned-but-unbuilt epics is something to avoid. It also surfaces
  `in-progress` items that have stalled. It should start simple; the human
  expects it to evolve with use.
- **`deferred-review` (new).** For each item it rereads the latest deferral
  comment, actually checks the conditions against the world (releases, issue
  states, dates, upstream status), and recommends one of these:
  - **keep:** no change.
  - **refresh:** re-defer with updated conditions.
  - **revive:** to intake, planning or one-off.
  - **drop.**

  It reports what it checked, so the human can trust the recommendation.
- **`epic-create` (updated).** Its entry points accept an issue in the deferred
  queue by routing it to planning first, as they do for intake. Its "don't
  invent states like deferred" note now points at the defer route instead.

### 3.9 Migration sweep

A one-time, interactive pass, run after the tooling ships and is installed. It
covers **every queue home in every active org** (each org's `.github`, plus
self-homed private repos):

1. **Find candidates:**
   - Intake items whose latest disposition is a `Deferred <date>:` comment.
   - Planning items deferred in prose, such as #165 and #436. Finding these
     takes judgment.
2. **Move each one** with `vrg-intake-route … defer --date <original date>`.
   The reason comes from the
   existing comment. Where the old comment gave only a reason, the agent drafts
   the `Revisit when:` conditions.
3. **Flag items whose "deferral" was really something else** for the human
   instead of moving them silently. For example, vergil-tooling#1593 and #1594
   are waiting to become tasks of planning item #384.
4. **Confirm in batches** with the human, as in intake review.

**Done when** `vrg-queue intake` and `vrg-queue planning` show no deferred
items in any queue home.

The sweep is an operational step with no code PR. Its result is recorded as a
comment on its task.

## 4. Sequence

1. **vergil-tooling:** the deferred kind, label, routes, view, refusals and
   audit treatment, released.
2. **Deployment:** the release is installed where the review skills and the
   sweep run. The release itself is human-gated.
3. **vergil-claude-plugin:** the three review skills and the `epic-create`
   update, released. This needs only the tooling to be merged and released, so
   it can overlap with step 2.
4. **Migration sweep** across all queue homes.
5. **Docs review** (vergil-tooling#3669), then the **retrospective**
   (#515).

## 5. Left to the builder

- The exact wording of the refusal messages, comments and skill prose.
- The batch size and presentation in the two new skills.
- How `vrg-queue` with no argument presents the three queues.
- Whether the sweep is driven from a skill or by an agent following the task's
  procedure.
