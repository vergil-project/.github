# Security records: several advisories per issue

- **Epic:** vergil-project/.github#573
- **Planning issue:** vergil-project/.github#571
- **Umbrella:** vergil-project/.github#500 (child A′). Revises epic
  vergil-project/.github#517 (child A). Children C (vergil-project/.github#553)
  and D (vergil-project/.github#568) depend on this epic for their
  security-finding handling.
- **Status:** approved design (2026-10-27). The human skipped the brainstorm
  questions, so the design is the agent's; decisions marked *my call* were
  approved with the spec.
- **Review history:** the comments on the planning issue,
  vergil-project/.github#571

## 1. Intent

Epic #517 gave every security finding a **pair**: a private draft advisory in
the org's `.github` holding the details, and a neutral `security` issue
pointing at it. Its tooling assumes each issue names **one** advisory.

One piece of work (a planning issue under design review, a task under
agentic-review) can find **several** security problems, and each needs its own
advisory. The tooling breaks on the second:

- `parse_advisory_ref` reads only the first `Security advisory:` line, and the
  planning hold, intake routing and the sweep all rely on it;
- `unmark_issue` acts only when the first line names its advisory, and when it
  does it drops the `security` label even if other advisory lines remain;
- the `epic-create` step 9 handoff names a single advisory.

So with two findings on one issue, the order of retargets decides whether a
stale line is left behind, and the label can vanish while an advisory still
tracks the issue.

This epic makes **several advisories per issue** work everywhere, and gives
children C and D a recipe each that uses the existing commands. The planning
issue's starting direction also proposed a dedicated issue per finding; that
was a proposed mechanism, not a requirement, and the design review found it
unnecessary once an issue can carry several advisories.

## 2. Goals, non-goals, anti-goals

### Goals

- An issue may name any number of advisories, and every tool reads, marks,
  unmarks, checks, holds and routes all of them correctly.
- `epic-create` step 9 hands off **every** advisory a planning issue names.
- C and D each have a recipe, stated here, that uses only existing commands.

### Non-goals

- New commands, flags, queues, holds or routing rules.
- Changing where advisories live, the output rule, the sweep's state sync, or
  the App permissions (#517 §4.1–4.3, §4.8).
- Writing C's or D's skill text. This epic states the contract (§4.4).

### Anti-goals (failures even if everything technically works)

- Any security detail in a public record.
- A stale advisory line, or a `security` label removed while an advisory line
  remains, whatever order operations run in.
- A public pointer from shipped work to a record that is still open, which
  would lead an observer from "an unfixed problem exists" to the diff it is
  in (§4.4).

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
  - `retarget` posts `Security record moved to <refs>` on every issue it
    unmarks.
- `lib/queue_view.py` `planning_items` holds a planning issue back from
  `ready-to-close` while the advisory its first line names still tracks it.
- `lib/intake_route.py`: split and recreate retarget the first-line advisory
  before the original closes; transfer retargets it after the move.
- `bin/vrg_security_record.py` `show <issue>` parses one advisory.
- `vrg-security-record create` without `--issue` creates a record issue in
  the intake of `epics.queue_home(<affected repo>)`; `--issue <ref>` (attach
  mode) adds the line and label to an existing issue.
- Live records today: GHSA-r5m3-gp2j-fqmh on vergil-tooling#3524 (an ordinary
  task, one advisory) and the closed test advisory GHSA-hgp4-f6fc-fjcg. C and
  D have not run. §4.5 checks this rather than assuming it.

## 4. Design

### 4.1 Several advisories per issue

An issue names any number of advisories, one `Security advisory:` line each.

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
- `show <issue>` prints every pair the issue names, in body order.

### 4.2 Holds and routing

- **Planning hold** (`vrg-queue planning`, #517 §4.4b): a planning issue is
  held from `ready-to-close` while **any** advisory it names still tracks it.
  The note names how many, by GHSA ID only.
- **Transfer and recreate** retarget **every** advisory the item names to the
  new issue.
- **Split** of an item naming more than one advisory is refused (*my call*):
  which advisory goes to which task can't be inferred, so the message lists
  the GHSA IDs and says to `retarget` each by hand before splitting. An item
  naming one advisory splits as today.

### 4.3 `epic-create` step 9

For **each** advisory the planning issue names, once its fix task(s) are
filed, run `vrg-security-record retarget <GHSA-id> --to <task> [--to <task>
…]`. Post `Tasks filed: yes` only after every retarget succeeds. The planning
hold (§4.2) keeps the planning issue open until then.

A finding that the revised design resolves by itself (no task implements it,
for example because the spec now removes a mechanism) is retargeted to the
epic's **documentation task**, whose PR publishes the revised spec, but
**only** when the weakness exists only in the design and no shipped code has
it. A finding about shipped behaviour always goes to a fix task that changes
the code.

### 4.4 Recipes for C and D

- **Child C** (#553, spec §3.8, design review):
  - accepted finding: `create --issue <planning issue>`; step 9 hands it off
    (§4.3);
  - rejected or dropped finding: plain `create`, which gives it its own
    intake issue.
- **Child D** (#568, spec §4.11, agentic-review), which records at collection,
  before it knows whether a finding will be fixed:
  - at collection, record each `new` or `repair` security finding with plain
    `create`. A `known` finding already has its record, so it is not recorded
    again;
  - fixed on the branch: `vrg-intake-route <record> split --into <task>`,
    which retargets the advisory to the task and closes the record, so the
    advisory closes when the task does;
  - not fixed: nothing more. The record stays an ordinary intake item, with
    no pointer to the task.
  - D does **not** attach to the task at collection (*my call*): handing an
    unfixed finding off later would make `retarget` post a public
    `Security record moved to <record>` comment on the task, pointing from
    the shipped diff to the open record.
- This epic posts these recipes on C's and D's planning issues
  (vergil-project/.github#469, vergil-project/vergil-claude-plugin#677); their
  own documentation tasks revise their specs.

### 4.5 Migration

One advisory line per issue is a valid case of the new form, so nothing needs
converting. The end-to-end validation task confirms it on the org's live
records (`vrg-security-record list` and the sweep's broken-pair report) and
records the counts; any inconsistent record is repaired by hand under that
task. No migration code (*my call*).

### 4.6 Documentation

- vergil-tooling security-records guide: several advisories per issue, the
  step 9 handoff per advisory and the docs-task rule (§4.3), the recipes
  (§4.4), the migration check.
- vergil-tooling CLI reference: `show` with several pairs, the split refusal,
  the planning-hold note.
- vergil-claude-plugin: `epic-create` step 9 and 10 text, and the skills
  catalogue where it describes them.

## 5. Constraints

- Public records show refs and counts only (umbrella §3.1; #517 §4.2).
- Lookups use REST listings, never search.
- No silent failures: every refusal names the refs involved and what to do
  next.
- This epic's own records must not contain any real finding's details.

## 6. Left to the builder

- The exact wording of refusals and the planning-hold note.

## 7. Open questions for the human

None.
