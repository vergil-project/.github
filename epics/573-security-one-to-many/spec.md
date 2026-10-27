# Security records for one-to-many discoveries

- **Epic:** vergil-project/.github#573
- **Planning issue:** vergil-project/.github#571
- **Umbrella:** vergil-project/.github#500 (child A′). Revises epic
  vergil-project/.github#517 (child A). Children C (vergil-project/.github#553)
  and D (vergil-project/.github#568) depend on this epic for their
  security-finding handling.
- **Status:** draft (2026-10-27), in design review. The human skipped the
  brainstorm questions, so the design is the agent's, built on the planning
  issue's starting direction. Decisions the human has not confirmed are
  marked *my call*.
- **Review history:** the comments on the planning issue,
  vergil-project/.github#571

## 1. Intent

Epic #517 gave every security finding a **pair**: a private draft advisory in
the org's `.github` holding the details, and a neutral `security` issue
pointing at it. Its tooling assumes each issue names **one** advisory.

One piece of work (a planning issue under design review, a task under
agentic-review) can find **several** security problems. Each needs its own
advisory, and the tooling breaks on the second:

- `parse_advisory_ref` reads only the first `Security advisory:` line, and the
  planning hold, intake routing and the sweep all rely on it;
- `unmark_issue` acts only when the first line names its advisory, and when it
  does it drops the `security` label even if other advisory lines remain;
- the `epic-create` step 9 handoff names a single advisory.

This epic fixes that, and gives each finding found during a piece of work its
own record issue that says which work found it, so the work can hand each one
to the task that fixes it.

## 2. Goals, non-goals, anti-goals

### Goals

- An issue may name any number of advisories, and every tool reads, marks,
  unmarks, checks and routes all of them correctly.
- Each finding found while working an issue has its own record issue naming
  that work, and the work can list them.
- The work hands each finding to its fix task with existing routing
  machinery.
- C and D have one recipe each, stated here.

### Non-goals

- Any new queue, hold, or routing rule. Intake review is a human-led pass;
  it is told what a `Found in:` line means (§4.5), and that is all.
- Changing where advisories live, the output rule, the sweep's state sync, or
  the App permissions (#517 §4.1–4.3, §4.8).
- Writing C's or D's skill text. This epic states the contract (§4.6).
- Enforcing the about-a-day lifetime.

### Anti-goals (failures even if everything technically works)

- Any security detail in a public record, including the `Found in:` line.
- A stale advisory line, or a `security` label removed while an advisory line
  remains, whatever order operations run in.
- A public marker that points an observer at public text describing a
  weakness that is live in shipped code (§4.4).

## 3. Verified facts this design rests on

From the code (vergil-tooling `develop` at `c19192d0ae`):

- `lib/security_records.py`:
  - `parse_advisory_ref` returns the first match of the `Security advisory:`
    pattern, or None.
  - `unmark_issue` returns without changes unless that first match is its
    GHSA ID; when it acts it removes only its own line but always drops the
    `security` label.
  - `mark_issue` appends a line only when that exact line is absent, so it
    already handles a second advisory.
  - `security_issues` and `classify` keep one `ghsa_id` per issue.
- `lib/queue_view.py` `planning_items` holds a planning issue back from
  `ready-to-close` while the advisory its first line names still tracks it.
- `lib/intake_route.py`: split and recreate retarget the first-line advisory
  before the original closes; transfer retargets it after the move.
- `vrg-security-record create` without `--issue` creates the record issue in
  the intake of `epics.queue_home(<affected repo>)`; `--issue <ref>` (attach
  mode) adds the line and label to an existing issue instead.
- Live records today: GHSA-r5m3-gp2j-fqmh on vergil-tooling#3524 (an ordinary
  task, one advisory) and the closed test advisory GHSA-hgp4-f6fc-fjcg. C and
  D have not run. §4.8 checks this rather than assuming it.

## 4. Design

### 4.1 The model

| Relation | Cardinality | Recorded as |
|---|---|---|
| advisory → issues that track it | one to many | `Tracking issue:` lines in the advisory (unchanged) |
| issue → advisories it names | one to many (**new**: was at most one) | one `Security advisory:` line per advisory in the issue body |
| record issue → the work that found it | many to one | a `` Found in: `<owner>/<repo>#<N>` `` line in the record issue's body (**new**) |

**Lifecycle of one finding found during a piece of work:**

1. **Recorded:** `vrg-security-record create --found-in <the work>` makes the
   advisory and its record issue (§4.3).
2. **Handed off:** once the fix is planned (C) or done (D), the work runs
   `vrg-intake-route <record> split --into <fix task> [--into …]`. Split
   already retargets the advisory to the task(s) and then closes the record
   issue, and stops with the record open on any failure.
3. **Closed:** the sweep closes the advisory once every issue tracking it is
   closed (unchanged).

A finding that is not the work's own (rejected, out of scope, or not fixed)
is recorded with plain `create` and routed by intake review like any other
item. A record created with `--found-in` whose work ends without a handoff is
simply an open intake item: nothing is closed or lost.

### 4.2 Several advisories per issue

- `parse_advisory_ref` is replaced by `parse_advisory_refs`, which returns
  every GHSA ID a body names, in order, without duplicates. Every caller moves
  to it.
- `unmark_issue(ref, ghsa_id, …)` acts when **any** line names `ghsa_id`,
  removes that line only, and drops the `security` label only when **no**
  advisory line remains. Its comment-then-one-PATCH order and idempotence stay.
- `security_issues` returns every GHSA ID per issue, and `classify` checks
  every line both ways.
- `retarget` keeps its shape; "the issue names this advisory" becomes a
  membership test.
- The `vrg-queue planning` hold (#517 §4.4b) covers **every** advisory the
  planning issue names.
- Transfer and recreate retarget **every** advisory the item names. Split of
  an item naming more than one advisory is refused (*my call*): which advisory
  goes to which task can't be inferred, so the message says to `retarget` each
  by hand. A record issue names one advisory, so the handoff never hits this.
- `show <issue>` prints every pair the issue names, in body order.

### 4.3 `create --found-in`

- `--found-in <owner>/<repo>#<N>` names the work that found the finding. Not
  allowed with `--issue`.
- Allowed only when `queue_home(<affected repo>)` equals the queue home of
  the origin's repo; otherwise refused, pointing to plain `create`. The record
  issue lives in that one home (*my call*). This keeps a public record from
  naming private work and a private record from naming another private repo's,
  and resume finds the issue in the same home by construction.
- A fresh create checks the origin exists and is open. `--resume` skips that
  check (the origin may have closed since the failure) and adds the
  `Found in:` line if it is missing.
- The line is written as inline code, `` Found in: `<ref>` `` (*my call*), so
  GitHub draws no cross-reference on the origin's timeline. The relation is
  read with `vrg-security-record list --found-in <ref>` (states and GHSA IDs
  only, never advisory text).

### 4.4 Where a handoff goes

- **To the fix task(s)** whose work changes the code. A finding about
  **shipped** behaviour always goes to a fix task.
- **To the epic's documentation task**, only for a finding whose weakness
  exists **only in the design** (no shipped code has it) and that the revised
  spec resolves by itself, for example by removing a mechanism. The docs PR
  publishes the revised spec, so its merge is when the weakness is resolved.
  It is never dropped, which would record a fixed finding as "not planned".

### 4.5 Skill text

- **`epic-create` step 9:** for each record from
  `vrg-security-record list --found-in <planning issue>`, split it into its
  fix task(s) (§4.4); for each advisory the planning issue itself names (a
  security item that went through planning), `retarget` it as today. Post
  `Tasks filed: yes` only after every handoff succeeds.
- **`intake-review`:** an item whose `Found in:` names an open issue belongs
  to that work; leave it for the work to hand off, unless the human decides
  otherwise.

### 4.6 Consumers

- **Child C** (#553, spec §3.8): accepted findings use
  `create --found-in <planning issue>`; rejected or dropped findings use plain
  `create`. Step 9 hands them off (§4.5).
- **Child D** (#568, spec §4.11): every finding is recorded at collection with
  `create --found-in <task>`. Fixed on the branch: split the record into the
  task. Not fixed: the record stays an open intake item.
- This epic posts the contract on C's and D's planning issues
  (vergil-project/.github#469, vergil-project/vergil-claude-plugin#677); their
  own documentation tasks revise their specs.

### 4.7 The sweep

Changes only through §4.2. `Found in:` is not a pair link and the sweep
doesn't check it.

### 4.8 Migration

One advisory line per issue is a valid case of the new form, and `Found in:`
is optional, so nothing needs converting. The end-to-end validation task
confirms it on the org's live records (`vrg-security-record list` and the
sweep's broken-pair report) and records the counts; any inconsistent record
found is repaired by hand under that task. No migration code (*my call*).

### 4.9 Documentation

- vergil-tooling security-records guide: the model and lifecycle (§4.1), the
  handoff rule (§4.4), several advisories per issue, the migration check.
- vergil-tooling CLI reference: `create --found-in`, `list --found-in`,
  `show` with several pairs, the split refusal.
- vergil-claude-plugin: `epic-create` and `intake-review` (§4.5), and the
  skills catalogue where it describes them.

## 5. Constraints

- Public records show refs and counts only (umbrella §3.1; #517 §4.2).
- Lookups use REST listings, never search.
- No silent failures: every refusal names the refs involved and what to do
  next.
- All behaviour lives in the sanctioned tools.
- This epic's own records must not contain any real finding's details.

## 6. Left to the builder

- Whether `list --found-in` reads `security_issues` or only the queue home.
- The exact wording of refusals.

## 7. Open questions for the human

None. A `--found-in` record issue lives in the queue home's intake, where
`create` already puts every record issue; `Found in:` and the
`intake-review` instruction (§4.5) keep it with its work. A separate home for
record issues was considered and rejected, since it would add a queue.
