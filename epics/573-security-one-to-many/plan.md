# Security records: several advisories per issue — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Let one issue name several security advisories, and make every tool
that reads, marks, unmarks, checks, holds or routes an advisory line handle all
of them.

**Architecture:** `lib/security_records.py` replaces the single-advisory
parser with `parse_advisory_refs`. `unmark_issue`, `SecurityIssue`, `classify`
and `retarget` move to many advisories per issue. Then the callers follow:
`vrg-security-record show`, `vrg-intake-route` (transfer, recreate, split) and
`vrg-queue planning` (the hold). The plugin's `epic-create` step 9 hands off
each advisory. No new command, flag, queue or hold.

**Tech Stack:** Python 3.12+ (pytest), the `gh` CLI via `lib/github.py`,
Markdown skill and docs.

**Spec:** `epics/573-security-one-to-many/spec.md`
(vergil-project/.github#573)

**Shape:** an epic-level plan. Each task below is **one issue and one PR in
the repo named in its heading**, worked in a worktree of that repo. Paths are
relative to that repo's root. The deployment and validation tasks are
operational: they have no code PR, and they close on `Outcome: SUCCESS`.

## Global Constraints

- The reference line is unchanged, exact and alone on its line:
  `Security advisory: <org>/.github <GHSA-id>`. An issue body may carry any
  number of them, one per advisory.
- An issue keeps the `security` label while **any** advisory line remains.
- The output rule stands (#517 §4.2): advisory `summary` and `description`
  reach stdout only through `show` and `list --details`. Every other output,
  including every new refusal and note, names only GHSA IDs, issue refs,
  labels and states.
- Lookups use REST listings, never search.
- No silent failures: every refusal names the refs and GHSA IDs involved and
  the next command to run.
- No new commands, flags, queues or holds (spec §2).
- Validation: `vrg-container-run -- vrg-validate`. Commit with `vrg-commit`.
  Hand off with `vrg-pr-workflow report-ready`. Report-ready notes and PR
  bodies carry no real finding's details.

## Review Focus

1. **Unmarking one of two advisories from an issue.** The label and the
   other line must stay. Pinned by
   `test_unmark_issue_keeps_label_while_another_advisory_remains` (Task 1).
2. **Retargets run in either order on an issue naming two advisories.** No
   stale line, and the label goes only after the last one. Pinned by
   `test_retarget_two_advisories_off_one_issue_in_either_order` (Task 1).
3. **A planning issue still naming one of two advisories** after the other
   was handed off. It must stay held. Pinned by
   `test_planning_held_while_any_named_advisory_tracks_it` (Task 2).
4. **A split of an intake item naming two advisories.** It must be refused
   before anything changes, with both GHSA IDs named. Pinned by
   `test_split_refuses_an_item_naming_several_advisories` (Task 2).
5. **A transfer whose second retarget fails.** The recovery message must
   name every advisory that is still not retargeted. Pinned by
   `test_transfer_retarget_failure_names_every_pending_advisory` (Task 2).

---

### Task 1: Several advisories per issue in the library, the sweep and `show` (vergil-tooling, implementation)

**Files:**

- Modify: `src/vergil_tooling/lib/security_records.py`:
  - add `parse_advisory_refs`;
  - `unmark_issue`, `SecurityIssue`, `security_issues`, `_issue_problem`,
    `classify` and `retarget`.
- Modify: `src/vergil_tooling/bin/vrg_security_record.py`: `show <issue>`
  (around line 381).
- Test: `tests/vergil_tooling/test_security_records.py`,
  `tests/vergil_tooling/test_vrg_security_record.py`,
  `tests/vergil_tooling/test_vrg_epic_audit.py`.

**Interfaces (produces):**

- `parse_advisory_refs(issue_body: str) -> tuple[str, ...]`: every GHSA ID
  named by an exact `Security advisory:` line, in body order, without
  duplicates. It uses the existing `_ADVISORY_RE`.
- `SecurityIssue.ghsa_ids: tuple[str, ...]` replaces `ghsa_id`. It is empty
  when the body has no line.
- `parse_advisory_ref` **stays** in this task, returning the first entry, so
  that Task 2's callers keep working. Task 2 removes it.

**Rules:**

- `unmark_issue(ref, ghsa_id, moved_to)` acts when `ghsa_id in
  parse_advisory_refs(body)`. It removes only that line, and drops the
  `security` label only when no advisory line is left after the removal. The
  comment-then-one-PATCH order is unchanged.
- `retarget`: the dropped-issue scan uses `ghsa_id in issue.ghsa_ids`.
- `classify`:
  - a `security` issue with no lines is broken;
  - **each** of its lines is checked (the named advisory exists and tracks
    the issue), and each failure is its own broken line;
  - an advisory is "named" when it is in any issue's `ghsa_ids`.
- `show <issue>` prints every pair the issue names, in body order, in the
  current per-pair format, with a blank line between pairs. It refuses an
  issue with no lines, as today. Before printing anything, it checks with
  one `list_advisories` call that every named advisory exists. If any is
  missing, it prints nothing and exits 1 with `advisory(ies) <GHSA>[, …] do
  not exist in <home>`.

- [ ] **Step 1: Write the failing tests:**
  - `test_parse_advisory_refs_reads_every_line_in_order`: a body naming
    GHSA-aaaa-aaaa-aaaa, then GHSA-bbbb-bbbb-bbbb, then the first again →
    `("GHSA-aaaa-aaaa-aaaa", "GHSA-bbbb-bbbb-bbbb")`;
  - `test_parse_advisory_refs_ignores_lookalike_text`: the existing
    lookalike cases → `()`;
  - `test_unmark_issue_keeps_label_while_another_advisory_remains`: two
    lines; unmark A → B's line and the label stay, A's line goes, and one
    comment is posted;
  - `test_unmark_issue_drops_label_with_the_last_line`;
  - `test_retarget_two_advisories_off_one_issue_in_either_order`: an issue
    naming A and B; retarget A then B to X (and B then A) → the issue has no
    lines and no label, and X has both;
  - `test_security_issues_returns_every_ghsa_id`;
  - `TestClassify.test_issue_naming_two_paired_advisories`: both paired, no
    broken lines;
  - `TestClassify.test_second_line_names_missing_advisory`: exactly one
    broken line, naming the issue and that GHSA ID;
  - `test_show_issue_prints_every_pair` (`test_vrg_security_record.py`);
  - `test_show_issue_refuses_when_a_named_advisory_is_missing`: names A
    (exists) and B (missing) → exit 1, stdout empty, and the message names
    B;
  - `test_sweep_closes_both_advisories_of_one_closed_issue`
    (`test_vrg_epic_audit.py`): one closed issue naming A and B, with two
    draft advisories that each track only it → output contains `2 advisory
    state(s) synced, 0 broken pair(s)`, and both are PATCHed to `closed`.
- [ ] **Step 2: Run them; expect FAIL.** Existing tests that read
  `SecurityIssue.ghsa_id` fail too; move them to `ghsa_ids`.
- [ ] **Step 3: Implement the rules above.**
- [ ] **Step 4: Run `vrg-container-run -- vrg-validate`; expect PASS.**
- [ ] **Step 5: Commit and run report-ready.**

---

### Task 2: Routing and the planning hold over every advisory (vergil-tooling, implementation)

Blocked by Task 1.

**Files:**

- Modify: `src/vergil_tooling/lib/intake_route.py`:
  - `_security_ghsa` becomes `_security_ghsas(...) -> tuple[str, ...]`, and
    `_item_advisory` becomes `_item_advisories(...)`;
  - transfer (`_one_off`), `_recreate` and the split branch of `_route`.
- Modify: `src/vergil_tooling/lib/queue_view.py`: `planning_items`,
  `_held_note`.
- Modify: `src/vergil_tooling/lib/security_records.py`: remove
  `parse_advisory_ref`.
- Test: `tests/vergil_tooling/test_intake_route.py`,
  `tests/vergil_tooling/test_queue_view.py`, and
  `tests/vergil_tooling/test_security_records.py`. Delete
  `test_parse_advisory_ref_*` and the `parse_advisory_ref` import; Task 1's
  `parse_advisory_refs` tests replace them.

**Interfaces:** consumes `parse_advisory_refs` (Task 1).

**Rules:**

- **Security item:** carries the `security` label and at least one line. A
  label without a line, or a line without a label, keeps today's
  broken-pair warning.
- **Transfer and recreate** retarget every advisory, in body order, to the
  new issue. Transfer keeps `renamed=(ref,)` for each.
  - On a failure, the error names every advisory not yet retargeted, with one
    `vrg-security-record retarget <GHSA> --to <new>` command each.
  - Recreate still leaves the original open on any failure.
- **Split** of an item naming more than one advisory is refused before any
  change, with:
  `<ref> names several advisories (<GHSA>, <GHSA>); retarget each by hand
  with vrg-security-record retarget, then split`.
  An item naming one advisory splits as today.
- **Planning hold:** an item is held while **any** advisory its body names
  still tracks it.
  - Advisories are still listed once per call, and only when some item names
    one.
  - The note becomes: `held: <k> advisory(ies) still track this planning
    issue (<GHSA>[, <GHSA>…]) — retarget each to its fix task(s) with
    vrg-security-record retarget`.

- [ ] **Step 1: Write the failing tests:**
  - `test_transfer_retargets_every_advisory`;
  - `test_transfer_retarget_failure_names_every_pending_advisory`: the
    second retarget fails → the message names B, with its command;
  - `test_recreate_retargets_every_advisory_before_closing`;
  - `test_split_refuses_an_item_naming_several_advisories`: no comment, no
    close and no retarget call; the message names both IDs;
  - `test_split_one_advisory_unchanged`;
  - `test_planning_held_while_any_named_advisory_tracks_it`: names A and B;
    only B still tracks it → held, and the note names B only;
  - `test_planning_ready_after_every_advisory_retargeted`.
- [ ] **Step 2: Run them; expect FAIL.**
- [ ] **Step 3: Implement the rules above, and delete `parse_advisory_ref`**
  and its tests. After this task no caller remains.
- [ ] **Step 4: Run `vrg-container-run -- vrg-validate`; expect PASS.**
- [ ] **Step 5: Commit and run report-ready.**

---

### Task 3: `epic-create` hands off every advisory (vergil-claude-plugin, implementation)

Blocked by Task 2 (the note wording).

**Files:**

- Modify: `skills/epic-create/SKILL.md`, step 9's security sub-bullet and
  step 10's held sub-bullet.
- Modify: `docs/site/docs/skills/index.md`, the epic-create paragraph
  (lines 64-68). It becomes "hands each advisory to its fix task(s) or, for
  a design-only finding, the documentation task, before posting `Tasks
  filed: yes`".

- [ ] Step 9: the planning issue's body has **one or more** `Security
  advisory:` lines. Once the fix task(s) are filed, retarget **each**
  advisory to the task(s) that fix it. The text contains the exact phrase
  `one vrg-security-record retarget <GHSA-id> --to … per advisory`, which
  Task 5 checks for. A finding the revised design resolves by
  itself goes to the epic's documentation task, but **only** when no shipped
  code has the weakness; a finding about shipped behaviour always goes to a
  fix task (spec §4.3). Post `Tasks filed: yes` only after every retarget
  succeeds.
- [ ] Step 10: quote Task 2's note, and say the item is held until every
  advisory is retargeted.
- [ ] Run `vrg-container-run -- vrg-validate`; expect PASS. Commit and run
  report-ready.

---

### Task 4: Security-records docs (vergil-tooling, implementation)

Blocked by Tasks 1 and 2.

**Files:**

- Modify: `docs/site/docs/guides/security-records.md`:
  - several advisories per issue;
  - the per-advisory step 9 handoff, and the docs-task rule (spec §4.3);
  - the recipes for design review and agentic review (spec §4.4), with the
    reason D records with plain `create`;
  - the migration check (spec §4.5).
- Modify: `docs/site/docs/reference/dev/security-record.md`:
  - `show` with several pairs;
  - the hold and routing paragraph (lines 215-222).
- Modify: `docs/site/docs/reference/cli-tools-overview.md`: the
  `vrg-intake-route` section (from line 251) gets the split refusal, and the
  `vrg-queue` section (lines 329-338) gets the new hold note.
- Modify: `docs/site/docs/standards/github-issues.md` (around line 616), the
  hold note.

- [ ] Grep `docs/site` for `held: advisory` and update every hit to Task 2's
  note.
- [ ] Write the changes. Every example uses placeholder GHSA IDs and the
  text "not a real finding".
- [ ] Run `vrg-container-run -- vrg-validate`; expect PASS. Commit and run
  report-ready.

---

### Task 5: Release and deploy (vergil-tooling, deployment)

Blocked by Tasks 1–4. **Preconditions (human-attested):**

- a vergil-tooling release containing Tasks 1, 2 and 4 has been cut, and the
  `v2.1` line resolves to it (name the version);
- a vergil-claude-plugin release containing Task 3 is published (name the
  version, cut **after** Task 3 merged);
- the sweep's tooling install resolves to that vergil-tooling release;
- the human has run `vrg-vm update` on the host for the agent VM. It is a
  host command, not an agent one.

- [ ] In the agent VM, check:
  - `/opt/vergil/vergil-tooling/venv/bin/python -c "import
    importlib.metadata as m, vergil_tooling.lib.security_records as s;
    print(m.version('vergil-tooling')); s.parse_advisory_refs"` prints the
    named release's version and exits 0;
  - the installed `epic-create` step 9 text contains `one
    vrg-security-record retarget <GHSA-id> --to … per advisory`.
- [ ] Record `Outcome: SUCCESS` with both versions.

---

### Task 6: End-to-end check and migration count (.github, validation)

Blocked by Task 5. **Precondition (human-attested):** the human will dispatch
`epic-sweep.yml` when asked.

- [ ] **Migration check first** (spec §4.5). Record:
  - the number of records `vrg-security-record list` shows;
  - the broken-pair count from a dispatched sweep's summary line.

  Repair any broken pair by hand under this task, then re-check.
- [ ] Create a test intake issue I and attach two records to it. Use `create
  --issue I` twice, with details text "PROBE-A′ not a real finding 1" and
  "… 2" and severity low. Expected: I has both lines and one `security`
  label.
- [ ] Create a plain intake issue X. Run `retarget <A> --to X`. Expected: I
  keeps B's line and the label, and X has A's.
- [ ] Run `retarget <B> --to X`. Expected: I has no lines and no label, and
  X has both.
- [ ] Drop I: `vrg-intake-route <I> drop --reason "end-to-end test issue for
  #573"`.
- [ ] Close X, then ask the human to dispatch the sweep. Expected: both
  advisories are closed, the run reports 0 broken pairs, and `vrg-gh run
  view <id> --log` does not contain `PROBE-A′`.
- [ ] Record `Outcome: SUCCESS` with the GHSA IDs, the run ID and the
  migration counts.

---

### Bookends (already filed)

- Documentation: vergil-project/.github#574 (this spec and plan).
- Documentation review: vergil-project/vergil-tooling#3725.
- Retrospective: vergil-project/.github#575.

**Planning-time action** (the `epic-create` session, after the docs PR is
handed off, with no task): post spec §4.4's recipes on C's and D's planning
issues (vergil-project/.github#469, vergil-project/vergil-claude-plugin#677).

## Self-Review

- **Spec coverage:**
  - §4.1 → Task 1;
  - §4.2 → Task 2;
  - §4.3 → Task 3;
  - §4.4 → the planning-time action and Task 4;
  - §4.5 → Task 6;
  - §4.6 → Tasks 3 and 4, plus the documentation-review bookend.
- **Types:** `parse_advisory_refs` and `SecurityIssue.ghsa_ids` are defined
  in Task 1 and used by those names in Task 2.
- **Proportion:** about 1.6 times the spec's length, with no function
  bodies. The extra length is the test names and the exact refusal and note
  wording, which Tasks 2 and 3 share.
