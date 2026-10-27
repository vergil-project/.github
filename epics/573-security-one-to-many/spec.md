# Security records for one-to-many discoveries

- **Epic:** vergil-project/.github#573
- **Planning issue:** vergil-project/.github#571
- **Umbrella:** vergil-project/.github#500 (child A′). Revises epic
  vergil-project/.github#517 (child A). Children C (vergil-project/.github#553)
  and D (vergil-project/.github#568) depend on this epic for their
  security-finding handling.
- **Status:** draft (2026-10-27). The human skipped the brainstorm questions,
  so the design below is the agent's, built on the planning issue's starting
  direction. Decisions the human has not confirmed are marked *my call*.
- **Review history:** the comments on the planning issue,
  vergil-project/.github#571

## 1. Intent

Epic #517 gave every security finding a **pair**: a private draft advisory in
the org's `.github` holding the details, and a neutral `security` issue
pointing at it. Its tooling assumes each issue names **one** advisory.

Real use has found the gap. Whatever the agent is working on when it finds
security problems (a planning issue under design review, a task under
agentic-review), it is working on one issue, and that one issue can produce
**several** findings. Child C's design (spec §3.8) attaches every accepted
finding to the planning issue with `vrg-security-record create --issue`, and
today's code breaks on the second one:

- `parse_advisory_ref` reads only the first `Security advisory:` line, and the
  planning hold, intake routing and the sweep all rely on it;
- `unmark_issue` acts only when the first line names its advisory, and when it
  does it drops the `security` label even if other advisory lines remain;
- the `epic-create` step 9 handoff names a single advisory.

So with two findings on one issue, the order of retargets decides whether a
stale line is left behind, and the label can vanish while an advisory still
tracks the issue. The sweep would then report a broken pair every night.

This epic makes many findings per piece of work a first-class case. Each
finding stays individually manageable, the work that found it keeps a visible
relation to it, and nothing is routed or closed out from under it.

## 2. Goals, non-goals, anti-goals

### Goals

- An issue may name any number of advisories, and every tool reads, marks,
  unmarks, checks and routes all of them correctly.
- Every finding made while working an issue gets its **own** record issue,
  which records the issue that found it. The relation is queryable from either
  end.
- A finding held for the work that found it isn't routed separately by intake
  review, and a planning issue can't close or be deferred while findings it
  holds have not been handed to fix tasks.
- `epic-create` hands each held finding to its fix task(s) with existing
  routing machinery.
- Children C and D each have one clear recipe, stated here, for the commands
  they run.

### Non-goals

- Changing where advisories live, the output rule, the sweep's state sync, or
  the App permissions. All of #517's §4.1–4.3 and §4.8 stand except where this
  spec names a change.
- Writing C's or D's skill text. This epic states the contract; C and D build
  their skills on it (§4.9).
- Enforcing the about-a-day lifetime.
- Search-based lookups. Every lookup stays on REST listings, as in #517.

### Anti-goals (failures even if everything technically works)

- Any security detail in a public record, including the new `Found in:` line
  and the new queue notes. They carry issue refs and counts only.
- A `Found in:` line in a public issue that names a **private** repo's issue.
  That would reveal the private repo and its work.
- An advisory tracking an issue whose closing does not mean the problem is
  resolved or handed off.
- A stale advisory line, or a `security` label removed while an advisory line
  remains, whatever order operations run in.
- A held finding parked for months because its origin was deferred.

## 3. Verified facts this design rests on

From the code (vergil-tooling `develop` at `c19192d0ae`):

- `lib/security_records.py`:
  - `parse_advisory_ref` returns the first match of the `Security advisory:`
    pattern, or None.
  - `unmark_issue` returns without changes unless that first match is its
    GHSA ID. When it acts, it removes only its own line but always drops the
    `security` label.
  - `mark_issue` appends a line only when that exact line is absent, so it
    already handles a second advisory correctly.
  - `security_issues` and `classify` keep one `ghsa_id` per issue.
- `lib/queue_view.py` `planning_items` holds a planning issue back from
  `ready-to-close` when the advisory its first line names still tracks it.
- `lib/intake_route.py`:
  - split and recreate retarget the advisory named by the first line before
    the original closes;
  - transfer retargets it after the move, with `renamed=` for the old number;
  - the planning route leaves the advisory alone, so a security item routed
    into planning is itself a planning issue that tracks its advisory
    (#517 §4.4b).
- `vrg-security-record create --issue <ref>` (attach mode) adds the line and
  label to an existing issue. Without `--issue` it creates a neutral issue in
  the intake of `epics.queue_home(<affected repo>)`.
- Live records today, from #517's retrospective: GHSA-r5m3-gp2j-fqmh on
  vergil-tooling#3524 (an ordinary task, one advisory), and the closed test
  advisory GHSA-hgp4-f6fc-fjcg. C and D have not run, so no issue is known to
  name two advisories, and no planning issue is known to have had an advisory
  attached. §4.8 checks this rather than assuming it.

## 4. Design

### 4.1 The relational model

Four kinds of thing, and the relations between them:

| Relation | Cardinality | Recorded as |
|---|---|---|
| advisory → issues that **track** it | one to many | `Tracking issue:` lines in the advisory (unchanged) |
| issue → advisories it **names** | one to many (**new**: was at most one) | `Security advisory:` lines in the issue body, one per advisory |
| record issue → the issue it was **found in** | many to one | a `Found in: <owner>/<repo>#<N>` line in the record issue's body (**new**) |
| advisory → fix tasks | one to many | the tracking relation above, after the handoff (unchanged) |

**The tracking rule** (*my call*). An advisory is tracked only by issues
whose closing means the problem is resolved, or decided:

- its **record issue**, the neutral issue created with it, until a fix is
  planned; or
- the **fix task(s)** it was handed to (`retarget`, unchanged).

The issue being worked when the finding was made, the **origin**, is not a
tracker unless its own work fixes the finding (§4.4). It relates to its
findings through their `Found in:` lines.

This is how the planning issue's two directions fit together. Managing a
finding is one to one: one advisory, one record issue. Fixing it is one to
many: one advisory, any number of fix tasks, and a fix task may fix several
advisories, which is why an issue must be able to name several (§4.2).

The relation is **stored once**, on the record issue. The origin's body is
never edited (*my call*). GitHub's own cross-reference ("mentioned this
issue") shows each record issue on the origin's timeline without our writing
anything, and `vrg-security-record list --found-in <origin>` (§4.3) answers
the reverse query.

### 4.2 Several advisories per issue

Every reader and writer of the `Security advisory:` line handles any number
of them:

- `parse_advisory_ref` is replaced by `parse_advisory_refs`, which returns all
  of the GHSA IDs an issue body names, in order, without duplicates. Every
  caller moves to it.
- `unmark_issue(ref, ghsa_id, …)` acts when **any** line names `ghsa_id`. It
  removes that line only, and drops the `security` label only when **no**
  advisory line remains. The comment-then-one-PATCH order and its idempotence
  stay as they are.
- `mark_issue` is unchanged; it already appends without duplicating.
- `security_issues` returns every GHSA ID per issue (`ghsa_ids`), and gains
  the issue's `found_in` ref (§4.3).
- `classify` checks every line of every issue both ways:
  - a `security` issue with no advisory line is broken;
  - a line naming an advisory that doesn't exist, or that doesn't track the
    issue, is broken;
  - each advisory's tracking lines must name issues that name it back.
- `retarget` needs no change in shape: it already finds dropped issues by
  their line naming this advisory, which now means any line.

### 4.3 Record issues and `Found in:`

`vrg-security-record create` gains `--found-in <owner>/<repo>#<N>`, the
origin. It is not allowed with `--issue`.

- The tool checks the origin exists and is open, and refuses otherwise.
- It creates the record issue as today and adds the line
  `Found in: <owner>/<repo>#<N>` to its body. The line is exact and alone on
  its line, like the other reference lines.
- `--resume` carries `--found-in` through: the printed resume command includes
  it, and a resumed create adds the line if it's missing. Resume looks for the
  record issue in the **same** queue home that `create` chose under the
  placement rule below, never a separately derived one. Otherwise a record
  placed in an origin's home would be missed and created a second time.

**Placement** (*my call*). The record issue lives in a queue home's intake,
as today. Never in the affected code repo, and `retarget` stays the hop to
the fix task. Placing it where the fix lands doesn't work: the fix task may
not exist yet, and may end up in a different repo or in several. The queue
home is chosen so a public issue never names private work:

- if the **affected repo** is private: `queue_home(affected)`, the #517 rule;
- otherwise, if the **origin** is in a private repo: the origin's queue home;
- otherwise: `queue_home(affected)`, which is the org's `.github`.

A record issue in a private home may name a public origin, so nothing private
reaches a public page.

When the origin and the affected repo are **both private and different
repos**, `--found-in` is refused (*my call*). A record in one private repo
must not name another private repo's work: they may belong to different
clients. The refusal says to use plain `create` and to put the relation in
the advisory details, which are private.

**Queries.**

- `vrg-security-record list --found-in <ref>` lists the record issues found in
  `<ref>`, with their state and GHSA IDs, never advisory text.
- `show` prints the `Found in:` ref with the pair.

The origin's ref in `Found in:` is neutral by the same reasoning as the GHSA
ID in #517 §4.1. It says that the origin's work produced a security record,
and nothing about what the record is.

**Trade-off, accepted.** GitHub's cross-reference puts each record issue's
neutral title on a public origin's timeline. A reader can then line up "a
record was found here" with the origin's public text, such as a spec under
design review. This is accepted because:

- the spec already states the requirement publicly ("write the requirement,
  never the weakness");
- the umbrella's rules (§3.1) already allow public counts;
- a record is meant to live about a day.

### 4.4 Choosing the recipe

Whoever finds a security problem while working an issue picks exactly one of
three recipes:

| The finding… | Command | Tracked by |
|---|---|---|
| will be fixed by **this issue's own** work (D: fixed on the task's branch) | `create --issue <this issue>` | this issue, alongside any other advisories it names |
| belongs to this work, but its fix isn't planned yet (C: accepted in design review) | `create --found-in <this issue>` | its own record issue, held for this issue (§4.5) |
| is **not** this work's (rejected, out of scope, or D's unaddressed) | `create` | its own record issue, routed by intake review as today |

**Attach mode** (`--issue`) stays, restricted (*my call*):

- It is **refused for a planning item** (an issue whose parent is a planning
  epic). A planning issue's work never fixes anything, so under the tracking
  rule it can't be the tracker of a new finding. The refusal message names
  `--found-in` as the recipe to use.
- The #517 audit's use (attaching to an existing issue that *is* the problem)
  and D's use (attaching to the task that fixes it) are both kept.

A security item routed into planning keeps its own advisory, as #517 §4.4b
already allows: that planning issue is the record issue, not an origin.

### 4.5 The intake hold

A record issue is **held** when it is in intake, carries `security` and
names an advisory, and its `Found in:` origin is **open**. It belongs to the
origin's work, so intake review must not route it on its own.

- `vrg-queue intake` lists held items in a separate **held** group, with the
  note `held: found in <ref>`. The `intake-review` skill skips that group.
- `vrg-intake-route` refuses every route of a held item except **split**,
  naming the origin. Split is the handoff (§4.6). `--release` lets any route
  through, for an origin that decides the finding isn't its own after all;
  the item's `Found in:` line stays as provenance. There is no route-less
  release back to ordinary intake (*my call*). An origin that changes its
  mind routes the item there and then (defer, planning, one-off or drop),
  so nothing needs to remember a released state.
- When the origin closes, the hold lapses and the item is an ordinary intake
  item, with no tool action. A finding is never lost by its origin closing.
  It goes back into the queue, visible, oldest first, and the sweep keeps
  pairing it as before.

### 4.6 Planning: hold, handoff, defer

**The planning hold** gains a second reason. `vrg-queue planning` never shows
a planning issue as `ready-to-close` while:

- an advisory still tracks it (unchanged from #517, now over **every**
  advisory its body names); or
- it **holds** any open, `security`-labelled record issue found in it
  (**new**).

The note names the reason by refs only: `held: <k> security record(s) found
in this issue not yet handed to fix tasks — see vrg-security-record list
--found-in <ref>`.

**The handoff** at `epic-create` step 9 covers both kinds:

- for each record issue held for the planning issue, once its fix task(s)
  are filed: `vrg-intake-route <record> split --into <task> [--into <task>
  …] --reason "fix planned under <epic>"`. Split already retargets the
  advisory to the tasks before it closes the record issue, and stops with the
  record open on any failure;
- a finding the design **resolves by itself**, so that no task implements
  it (the revised spec removes the mechanism, or forbids the weak shape), is
  split into the epic's **documentation task**. The docs PR publishes the
  revised spec, so its merge is when the weakness is resolved, and the
  advisory closes with that task through the sweep. It is never dropped: a
  drop would record a fixed finding as "not planned";
- for each advisory the planning issue itself names (a security item that
  went through planning): `vrg-security-record retarget`, as today, once per
  advisory.

`Tasks filed: yes` is posted only after every handoff succeeds.

**Defer guard.** `vrg-intake-route <planning issue> defer` refuses while the
planning issue holds any record issue, naming the count and the `list
--found-in` command (*my call*). Deferring it would park security findings
for as long as the deferral lasts. The human releases or hands off each one
first.

**Split of an item naming several advisories** is refused (*my call*),
because which advisory goes to which task can't be inferred. The message
says to `retarget` each advisory by hand first. A record issue names exactly
one advisory, so the handoff never hits this. Transfer and recreate move the
whole issue, so they retarget **every** advisory it names to the new number.

### 4.7 The sweep

The sweep's pass changes only through §4.2: it reads every line, checks each
pair both ways, and syncs each advisory from all the issues tracking it,
exactly as today. `Found in:` is provenance, not a pair link, and the sweep
doesn't check it (*my call*). A `Found in:` naming a missing issue is harmless
(the hold just lapses), and checking it would add org-wide reads for no
safety gain.

### 4.8 Migration

There is no schema change to migrate: one advisory line per issue is a valid
case of the new form, and `Found in:` is optional. What could exist in a
shape the new rules reject:

1. a planning issue that an advisory was **attached** to (not routed in);
2. an issue naming two or more advisories that today's code has already left
   inconsistent (a stale line, or a missing label).

The end-to-end validation task checks both across the org's live records
with `vrg-security-record list` and the sweep's broken-pair report, and
records the counts. If any record is found, it's converted by hand under the
validation task:

- for (1), create a record issue with `--found-in`, then `retarget` the
  advisory to it;
- for (2), mark or unmark the issue by hand until the sweep reports it clean.

No migration code is written (*my call*), since §3 expects zero records.

### 4.9 Consumers

- **`epic-create` skill** (vergil-claude-plugin): step 9 and step 10 as in
  §4.6, replacing the single-advisory text from #517.
- **`intake-review` skill**: skip the held group (§4.5).
- **Child C** (#553, spec §3.8): accepted findings use
  `create --found-in <planning issue>`, not `--issue`. Rejected or dropped
  findings use plain `create`, unchanged. The step 9 handoff becomes the split
  of §4.6, including the split into the documentation task for a finding the
  design resolves by itself. This epic posts the required change on C's planning issue (#469);
  C's own documentation task revises its spec.
- **Child D** (#568, spec §4.10): no command changes. Findings fixed on the
  branch keep `create --issue <task>`, which now holds any number of findings
  correctly. Unaddressed findings keep plain `create`. This epic posts a note
  on D's planning issue confirming that.
- **`vrg-queue`, `vrg-intake-route`, the sweep:** as in §4.5–4.7.

### 4.10 Documentation

- vergil-tooling **security-records guide**: the relational model and the
  tracking rule (§4.1), the three recipes (§4.4), the intake hold (§4.5), the
  planning hold, handoff and defer guard (§4.6), and the migration check.
- vergil-tooling **CLI reference**: `create --found-in`, the attach-mode
  refusal, `list --found-in`, `vrg-queue intake`'s held group, and
  `vrg-intake-route --release` with the new refusals.
- vergil-claude-plugin: `epic-create` and `intake-review` (§4.9), and the
  skills catalogue where it describes them.

## 5. Constraints

- Public records show refs and counts only (umbrella §3.1; #517 §4.2's output
  rule).
- Lookups use REST listings, never search.
- No silent failures: every refusal names the refs involved and what to do
  next.
- Agents use `vrg-git` and `vrg-gh`. All new behaviour lives in the
  sanctioned tools (`vrg-security-record`, `vrg-intake-route`, `vrg-queue`).
- This epic's own records must not contain any real finding's details.

## 6. Left to the builder

- How `vrg-queue planning` finds record issues found in each planning item
  (one `security_issues` listing per run is the obvious way) and whether it
  needs caching.
- The exact wording of refusals and notes, within §5.
- Whether `list --found-in` and the hold share one helper.

## 7. Open questions for the human

None blocking. Each *my call* above is open to the human's review.
