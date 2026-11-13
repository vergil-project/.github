# epic-review and epic-implement adhoc mode — design

- **Epic:** vergil-project/.github#1077
- **Planning issue:** vergil-project/.github#1076 (review history is in its comments)
- **Status:** draft

## 1. Intent

Work that enters the fleet's epic/task model moves through a short, fixed
lifecycle:

1. A raw idea is captured into **intake**. Its destination is unknown.
2. Reviewing intake sends each item to exactly one place: **ad hoc** (a
   one-off task under a repo's ad-hoc epic), **planning** (it needs a full
   epic), **deferred** (on hold until a condition is met), or **dropped**.
3. Reviewing **deferred** drops each item or revives it into intake, planning
   or ad hoc.
4. Reviewing **planning** picks what `epic-create` expands next into a
   development epic, and defers or drops the rest.
5. Buildable work lives in only two places: a **development epic**, worked
   by `epic-implement`, and the **ad-hoc epics**, one per repo, whose tasks
   are small and standalone.

Steps 2 to 4 each have their own skill (`intake-review`, `deferred-review`,
`planning-review`). The three are almost identical in procedure and differ
only in the per-item check and outcomes. Step 5's ad-hoc half has no skill. It
is worked today by a driver given an ad-hoc prompt that loops
`issue-implement` over the tasks. That driver skips the rules that apply only
under `epic-implement`: one run went to merge review with non-review items
unsettled (vergil-project/vergil-tooling#3976), and the review doctrine
accepts rulings relayed in a dispatch prompt only from `epic-implement`
(vergil-project/vergil-claude-plugin#1005).

This epic gives the lifecycle two commands with non-overlapping verbs:

| | intake | planning | deferred | ad hoc | development epic |
|---|---|---|---|---|---|
| `epic-review` | reviews | reviews | reviews | refuses | refuses |
| `epic-implement` | refuses | refuses | refuses | implements (org-wide) | implements (as today) |

**Review** applies to the queues whose items still need a decision about
where they go. **Implement** applies to the epics whose items are already
placed and buildable. Reviewing an ad-hoc or development epic means nothing,
and neither does implementing a queue.

## 2. Goals

1. A new `epic-review` skill that reviews the intake, planning and deferred
   queues and replaces `intake-review`, `planning-review` and
   `deferred-review`.
2. Drop is an outcome in every `epic-review` mode.
3. An `adhoc` mode in `epic-implement` that works the org's ad-hoc tasks
   across every repo as one pass, under the same gates, ruling relay and
   batching rules as a development epic.
4. Each skill refuses the other's cases with a one-line pointer to the right
   command, and does nothing else.
5. A `vrg-queue adhoc` view in vergil-tooling that lists the org's open
   ad-hoc tasks, so the `adhoc` mode collects its work the same way the
   queue reviews do.
6. Deliver vergil-project/vergil-claude-plugin#1005 (container-epic routing)
   and vergil-project/vergil-claude-plugin#1007 (the drop-rule text).

## 3. Non-goals

- Unattended or scheduled running of either command
  (vergil-project/.github#710, vergil-project/.github#379).
- Changes to `epic-create`, or to the planning-issue lifecycle and its queue
  states (vergil-project/.github#950).
- Changes to `vrg-intake-route`'s routes or refusals, including
  vergil-project/vergil-tooling#4291 and the intake epic's sub-issue limit
  (vergil-project/vergil-tooling#3840).
- New batching rules. The `adhoc` mode uses `issue-implement`'s existing
  [What to batch](https://github.com/vergil-project/vergil-claude-plugin/blob/develop/skills/issue-implement/SKILL.md#what-to-batch)
  guidelines as they stand.
- A review of the ad-hoc queue. Its items were placed by a review already.
  A task found to be stale while it is worked is handled the way a stale
  task in a development epic is today.

## 4. Anti-goals

- **A wrapper over the old skills.** `epic-review` that dispatches to
  `intake-review` and the others keeps three copies of the same procedure.
  The point is one procedure, written once.
- **A second implement driver.** A separate ad-hoc skill would have to
  reproduce, or share by reference, the gate, ruling-relay and batching rules
  that `epic-implement` holds. Those are the rules the ad-hoc prompt skipped,
  and the review doctrine would need a change to accept relayed rulings from
  a second driver.
- **Routing across verbs.** `epic-implement intake` does not run the intake
  review on the human's behalf (as #1005 first proposed). It refuses and
  points to `epic-review intake`, so each verb means one thing.

## 5. Design

### 5.1 Identifying an epic's kind

Both skills accept either a keyword or an epic ref (`<owner>/<repo>#<N>`,
`#<N>`, or a URL). A ref's kind comes from its labels:

| Labels | Kind |
|---|---|
| `epic` + `intake` | intake queue |
| `epic` + `planning` | planning queue |
| `epic` + `deferred` | deferred queue |
| `epic` + `ad-hoc` | one repo's ad-hoc epic |
| `epic`, none of the above | development epic |

An ad-hoc epic ref given to `epic-implement` works that one repo's ad-hoc
tasks: the `adhoc` mode limited to one repo. A ref that is not an `epic` at
all is refused as not an epic. A keyword with no ref works the current org's
queue home (`<owner>/.github`); `--repo <owner>/<repo>` selects a self-homed
private repo's queues, as `vrg-queue --repo` does today.

### 5.2 `epic-review`

Invocation: `/vergil:epic-review <intake|planning|deferred>` or
`/vergil:epic-review <epic-ref>`. With no argument, it shows all three queue
counts (`vrg-queue`) and asks which to review.

**Shared procedure**, written once:

1. **Preflight:** the USER agent check, since every mode can close issues.
2. **Collect** the queue with `vrg-queue <kind>`. A missing queue epic is an
   empty queue, not a failure: report it and stop, as each skill does today.
3. **Present a batch of 4.** For each item, check its current state before
   recommending anything: read the item and its comments, search for an
   overlapping or already-closed issue or epic, and do the mode's own check
   (below). Write one short paraphrased paragraph per item ending in one
   recommended outcome.
4. **One confirmation per batch.** Apply nothing before the reply.
5. **Apply** with `vrg-intake-route`, using fully qualified refs. The agent
   closes dropped and split items itself. A refusal is information: report
   it and ask for a different outcome, and never work around it by hand. A
   scope change on a routed one-off goes to the human through
   `vrg-ruling draft-scope`, as today.
6. **Next batch**, then the mode's tallies when the queue is done or the
   human stops. Unreviewed items stay where they are, so there is nothing to
   resume.

**Per-mode sections** hold only what differs:

| Mode | Order | Mode check | Outcomes |
|---|---|---|---|
| intake | oldest first | is it already fixed (read the code for bugs) | drop, defer, one-off, split, planning, leave |
| planning | ranked with the human; `ready-to-close` and `in-progress` items surfaced first | blockers; the core rule (plan only what will be implemented soon) | pick (hand to `epic-create`), defer, drop, leave |
| deferred | least recently deferred first | actually check every `Revisit when:` condition against its source, and never fabricate a check | keep, refresh, revive (intake, planning or one-off), drop |

Each mode's existing rules carry over in substance: the planning core rule
and its discouragement of large pick lists, deferred's evidence-first report
of what was checked, and intake's preference for an existing finite epic
over the ad-hoc bucket.

**Drop in every mode.** Drop and defer of a planning item are refused only
while one of its development epics is open, which is what `vrg-intake-route`
enforces since vergil-project/vergil-tooling#4170. The skill text states that
rule and no older one (#1007).

**Refusals:** given an ad-hoc or development epic, it prints one line
(`<ref> is an ad-hoc epic; its tasks are worked with /vergil:epic-implement
adhoc` or `<ref> is a development epic; it is worked with
/vergil:epic-implement <ref>`) and stops.

**Triggers:** the description carries the old skills' trigger phrases
("review intake", "process the intake queue", "groom the backlog", "review
planning", "what should we plan next", "review deferred", "what's
deferred", "revisit the backlog", and the "triage" phrasings), so the
existing habits still land on it.

**Retirement:** `skills/intake-review/`, `skills/planning-review/` and
`skills/deferred-review/` are deleted in the same change. Every reference to
them in the plugin (other skills, the review doctrine, the paad policy, the
README, the site docs) is updated to `epic-review <mode>`.

### 5.3 `epic-implement adhoc`

Invocation: `/vergil:epic-implement adhoc`, or with one repo's ad-hoc epic
ref to limit it to that repo. The development-epic path is unchanged.

1. **Collect** the open tasks across every ad-hoc epic in the queue home
   with `vrg-queue adhoc`.
2. **Summarize** them for the human, grouped by repo: each task's ref, age,
   kind and a one-line paraphrase. Ad-hoc tasks run in the order the
   human sets, not a plan's dependency order. They have no plan, so the
   runnable frontier is every open task with no open `Blocked-by:`
   dependency.
3. **Propose a strategy:** which tasks to batch together per repo (under
   `issue-implement`'s batching guidelines), which to run alone, which to
   leave for later, and the order, oldest first within a repo unless there
   is a reason to differ. Flag any task whose state suggests it is stale or
   larger than one PR, with a recommendation; moving it is the human's call.
4. **One confirmation** of the whole strategy.
5. **Dispatch** by kind label exactly as the development path does
   (`issue-implement`, `issue-validate`, `issue-deploy`), as sub-agents
   under the Agent prompt contract, with the batch-marker check before each
   code-task dispatch.
6. **Gate and stop** exactly as the development path does: batch the
   non-review items and rulings for the human once, relay rulings to stopped
   runs, and stop. The skill is stateless: re-running it re-derives the
   frontier from GitHub.

What it does not do: the development path's plan reading, plan evolution
log, terminal handoff and retrospective drafting. Ad-hoc epics have no plan
and never close.

**Refusals:** given an intake, planning or deferred epic (or keyword), it
prints `<ref> is the <kind> queue; review it with /vergil:epic-review
<kind>` and stops.

The review doctrine's rule that relayed rulings are accepted only from
`epic-implement` then covers ad-hoc runs with no change to the doctrine.

### 5.4 `vrg-queue adhoc`

A fourth queue in `vrg-queue`: the open tasks under every open `Epic (ad
hoc): <repo>` epic (`epic` + `ad-hoc` labels) in the queue home, grouped by
repo and oldest first within each. Text and `--json` output follow the other
queues' shape, with the item's repo, its kind label and its open
`Blocked-by:` refs added. `--repo` selects a self-homed private repo's
ad-hoc epic, as for the other queues. `vrg-queue` with no queue named still
shows intake, planning and deferred; `adhoc` is shown only when named, since
it is not a review queue.

Failure handling follows the existing queues: a missing ad-hoc epic for one
repo is not an error (that repo has no ad-hoc work), while a failed read of
an epic that exists is a real failure that stops the command with its cause,
never a silently short list.

### 5.5 Tooling messages

vergil-tooling names the old skills in user-facing text: the refusals in
`lib/epics.py` that say to route an item "with intake-review first" or
revive it "with deferred-review first", the next-step hints in
`lib/epic_audit.py`, the queue epics' generated bodies, and
`vrg-intake-create`'s help. These change to `epic-review <mode>`.

## 6. Constraints

- **Release order.** The plugin's `adhoc` mode calls `vrg-queue adhoc`, so
  the tooling release that adds it ships before the plugin release that uses
  it (vergil-project/.github#932's concern). The tooling message change
  points to a skill that exists only after the plugin release. The plan
  orders the two releases so no released version gives advice that cannot be
  followed.
- **Private work stays private.** Both skills keep the existing
  private-repo rules: a self-homed private repo's queues and ad-hoc epic are
  reviewed and worked only with its own `--repo`.
- **Placement law.** Every task lands its PR in the repo it is filed in.

## 7. Left to the builder

- The exact layout of `epic-review`'s SKILL.md (one file, or one file plus
  per-mode reference files), within the plugin's skill conventions.
- The exact wording of the refusal lines and the `vrg-queue adhoc` text
  columns.
- Whether `epic-implement`'s `adhoc` mode lives inline in its SKILL.md or in
  a referenced section file, provided the shared gate, relay and batching
  text is not duplicated.
- The operational tasks: whether a plugin deployment and a live validation
  of both commands are needed, and their granularity (decided in the plan).
