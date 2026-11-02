# Measuring agent cost Implementation Plan

- **Epic:** vergil-project/.github#816
- **Planning issue:** vergil-project/.github#802 (review history is in its
  comments)
- **Status:** draft

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Every sub-agent call in vergil and paad logs its cost in one
stanza, recorded in write-once comments that a tool rolls up and audits;
human rulings are posted on the issue by the human; and `issue-implement`
is split into a manager and workers so each activity's cost is measured
separately.

**Architecture:** Two new `vrg-` tools carry the mechanics, so skills hold
no parsing: `vrg-agent-cost` (record, report and audit, lint) and
`vrg-ruling` (request and read, with the posting-identity check). Two new
plugin docs carry the rules: `docs/agent-cost-logging.md` and
`docs/ruling-protocol.md`. paad gains the stanza as a generic convention.
The work lands in two waves: logging on today's flow, measured by a
baseline, then the ruling protocol and the manager split, measured by the
closing validation sample.

**Tech Stack:** vergil-tooling Python 3 with pytest and GitHub GraphQL;
vergil-claude-plugin Markdown skills and doctrine, checked by
`scripts/bin/validate-custom`; paad's Makefile checks and stdlib Python 3
scripts with `--self-test`.

**Spec:** [`epics/816-agent-cost-metrics/spec.md`](spec.md)

## Global Constraints

- **Tokens are the primary metric.** Time is secondary; sub-agent time is
  labelled as possibly including unseen permission waits (spec 2).
- **A value the notice does not give is `not-reported`.** Never estimated,
  never left out, never copied from elsewhere (spec 3).
- **The caller writes the stanza** from the completion notice; it never asks
  a sub-agent to report on itself (spec 3).
- **Each stanza is recorded once** (spec 3).
- **Write-once:** no agent edits or deletes a comment. A correction is a new
  comment that links to the one it supersedes (spec 7.1).
- **Rulings are never relayed through prompts.** The acting agent is told
  only that the human has responded on the issue (spec 7.2).
- **Security items** appear in public records as IDs and counts only;
  security fixes reach the responder only as state-directory paths with
  hashes (spec 7.2, 8.2).
- **paad work follows paad's workflow** (epic #699's plan, Global
  Constraints): edits in `preview/paad/` only; `make export && make test`
  before each commit; a 1–3 line `[Unreleased]` changelog entry; no version
  bump; git writes in the fork are the human's; nothing vergil-specific in
  paad; paad's vocabulary only.
- **vergil-tooling and vergil-claude-plugin work** uses `vrg-container-run
  -- vrg-validate` as its only validation, `vrg-commit`, and `vrg-pr-workflow
  report-ready`; agents never run `vrg-submit-pr`.
- **No silent failures** in either tool: a record that cannot be read or
  parsed is an error with its comment URL, never skipped.
- **The fleet checkout moves only with reviews held** (paad-policy §10):
  the human moves `fleet`, runs the gate, and reviews resume only once the
  gate passes and the doctrine matches the result syntax.

## Review Focus

1. **A stanza-shaped line in a comment that is not a cost record** (a code
   excerpt in a findings comment, or this epic's own test fixtures quoted in
   review): it must be ignored, not counted and not an error. Pinned in
   Task 4 (`test_report_ignores_stanzas_outside_cost_records`).
2. **A ruling reply posted from the agent's own login, or a session in
   human mode**: the protocol must refuse, before posting or reading.
   Pinned in Task 12 (`test_read_rejects_reply_by_agent_login`,
   `test_identity_check_refuses_human_mode`).
3. **A human ruling edited after posting**: it must not be acted on; the
   tool reports `edited` and the skill posts a fresh request. Pinned in
   Task 12 (`test_read_reports_edited_reply`).
4. **A malformed line inside a cost record** (a typo, a field out of order,
   a value with a space): an audit error naming the comment, never a
   skipped line. Pinned in Task 4 (`test_audit_errors_on_malformed_line`).
5. **Two manager runs on one issue**, including one dispatched and one
   interactive: each gets its own root, and a reused root is an audit
   error. Pinned in Task 4 (`test_audit_errors_on_duplicate_path`,
   `test_next_run_index`).

---

## Delivery

Each task below is one issue under the epic. Fork, probe, deployment,
baseline, validation and human tasks are filed in `vergil-project/.github`
and closed by comment (spec 14); the rest close by a same-repo PR. The
bookends already exist: the docs task vergil-project/.github#817, the
documentation review vergil-project/vergil-claude-plugin#877 and the
retrospective vergil-project/.github#818.

| Task | Issue (repo, kind) | Blocked by |
|---|---|---|
| 1 | Measurement probe (`.github`, closed by comment) | — |
| 2 | The logging rule doc (vergil-claude-plugin) | 1 |
| 3 | `vrg-agent-cost`: the stanza and `record` (vergil-tooling) | 1 |
| 4 | `vrg-agent-cost`: `report` and the audit (vergil-tooling) | 3 |
| 5 | `vrg-agent-cost lint` and its `vrg-validate` stage (vergil-tooling) | 2, 3 |
| 6 | paad: the stanza convention, agentic-review and the lint (`.github`, fork) | 1 |
| 7 | paad: the other dispatching skills (`.github`, fork; one issue and one commit with Task 6) | 6 |
| 8 | Logging and write-once on today's skills (vergil-claude-plugin) | 2, 3, 5 |
| 9 | Doctrine: the result's `Agent costs:` field (vergil-claude-plugin) | 6 |
| 10 | Deploy wave 1 (`.github`, deployment) | 2–9 |
| 11 | Baseline sample on today's flow (`.github`, validation) | 10 |
| 12 | `vrg-ruling` (vergil-tooling) | 3 |
| 13 | The ruling protocol doc (vergil-claude-plugin) | 12 |
| 14 | The manager split (vergil-claude-plugin) | 4, 11, 13 |
| 15 | Doctrine: mode 2 for the new roles and records (vergil-claude-plugin) | 11, 13 |
| 16 | `epic-retrospective`: the epic cost summary (vergil-claude-plugin) | 4 |
| 17 | Rename sweep: vergil-claude-plugin | 14 |
| 18 | Rename sweep: vergil-tooling | 14 |
| 19 | Rename sweep: the `docs` repo | 14 |
| 20 | Deploy wave 2 (`.github`, deployment) | 12–19 |
| 21 | Validation sample (`.github`, validation) | 20 |
| 22 | Offer the paad changes upstream (`.github`, human) | 6, 7 |

- **Two waves.** Tasks 1–11 put logging on today's flow and measure the
  baseline before the split (spec 12). Tasks 12–21 add the ruling protocol
  and the split. Task 12 can start once Task 3 has merged.
- **The checkpoint.** Task 11 ends wave 1: the human reviews the baseline
  and confirms before Tasks 14–21 start.
- **Tasks 6 and 7 are one fork issue and one commit**, so `preview/paad`
  goes from no skill on the convention to every dispatching skill at once
  (Task 6's lint fails only partial adoption).
- **The doctrine's rewrite is two tasks.** Spec 8.3 names one dedicated
  rewrite. Its result-field part (Task 9) and its write-once part (in
  Task 8) must land before the baseline, so the baseline carries
  per-specialist cost (spec 12) and is recorded write-once; its roles part
  (Task 15) must land with the split. All stay separate from the rename
  sweep, as the spec requires.
- **Merge together.** Tasks 14, 15 and 17 are merged in one sitting, and
  the plugin release that carries one carries all three, so no release has
  the new `issue-manager` with old-meaning references or an old mode 2.
- **#812** may be revived from Task 11's results (spec 12).
- **Security advisories.** When the spec and plan merge, the agent hands
  each advisory on #802 to its fix task: GHSA-2fjg-qxrj-fc2v (the
  responder's security contract) to Task 14, and GHSA-9q3r-h2xj-rgq3 (the
  posting identity) to Task 12. GHSA-3q8w-mvq4-w3fq was rejected and its
  intake issue closed.
- **Probe outcomes amend the spec** before dependent tasks start (spec 12):
  items 1–4 before Tasks 2–4, items 7–9 before Tasks 14 and 15.

## File map

| Repo | File | Responsibility |
| --- | --- | --- |
| vergil-claude-plugin | `docs/agent-cost-logging.md` | The logging rule, the stanza, the vocabulary, the records (Task 2) |
| vergil-claude-plugin | `docs/ruling-protocol.md` | The ruling protocol and write-once records (Task 13) |
| vergil-claude-plugin | `skills/epic-create/SKILL.md`, `skills/epic-implement/SKILL.md`, `skills/issue-implement/SKILL.md`, other dispatching skills | Cost records on today's flow (Task 8) |
| vergil-claude-plugin | `docs/paad-review-doctrine.md` §1.9, §3.4 | The `Agent costs:` field (Task 9) |
| vergil-claude-plugin | `docs/paad-review-doctrine.md` §1.7, §3 | Mode 2 for the new roles and records (Task 15) |
| vergil-claude-plugin | `skills/issue-manager/SKILL.md` (new), `skills/issue-implement/SKILL.md` (worker), `skills/issue-respond/SKILL.md` (new), `skills/issue-deploy/SKILL.md`, `skills/issue-validate/SKILL.md`, `skills/epic-implement/SKILL.md` | The manager split (Task 14) |
| vergil-claude-plugin | `skills/epic-retrospective/SKILL.md` | The epic cost summary (Task 16) |
| vergil-tooling | `src/vergil_tooling/lib/agent_cost.py` | Stanza grammar, parse, cost records (Task 3) |
| vergil-tooling | `src/vergil_tooling/lib/issue_comments.py` | Trusted comments with author, URL and edit state (Task 3) |
| vergil-tooling | `src/vergil_tooling/lib/agent_cost_report.py` | Roll-up and audit (Task 4) |
| vergil-tooling | `src/vergil_tooling/lib/agent_cost_lint.py` | The dispatch-site lint (Task 5) |
| vergil-tooling | `src/vergil_tooling/bin/vrg_agent_cost.py` | `record`, `report`, `lint` (Tasks 3–5) |
| vergil-tooling | `src/vergil_tooling/bin/validate_common.py` | The `agent-cost-lint` stage (Task 5) |
| vergil-tooling | `src/vergil_tooling/lib/rulings.py`, `src/vergil_tooling/bin/vrg_ruling.py` | `request`, `read`, the identity check (Task 12) |
| vergil-tooling | `pyproject.toml` | The two entry points (Tasks 3, 12) |
| vergil-tooling | `tests/vergil_tooling/test_agent_cost*.py`, `test_rulings.py`, `test_vrg_ruling.py`, `test_issue_comments.py` | Tests |
| vergil-tooling | `docs/site/docs/reference/cli-tools-overview.md` | The two tools documented (Tasks 3–5, 12) |
| paad fork | `preview/paad/skills/agentic-review/SKILL.md` | Stanzas in Review Metadata and in the result (Task 6) |
| paad fork | `scripts/check_agent_cost.py`, `Makefile` | The lint and its self-test (Task 6) |
| paad fork | `preview/paad/skills/{agentic-owasp,agentic-architecture,agentic-a11y,agentic-dedup,backlog,rethink}/SKILL.md` | Stanzas in their reports (Task 7) |
| paad fork | `preview/paad/skills/paad-help/SKILL.md`, `CHANGELOG.md` | The convention documented (Tasks 6, 7) |

## Formats (fixed here, used by every task)

The spec leaves the exact syntax to the builder. Several tasks parse it, so
this plan fixes it. Task 2's rule doc copies this section as its normative
text; Task 1's findings may only fill in `BREAKDOWN_FIELDS` and the notice
mapping.

**The stanza line** (one per call, or per turn of a continued call):

```text
agent-cost: v1 call=<path> [turn=<n>] tokens=<v> [<breakdown>=<v> …] tool_uses=<v> duration_ms=<v> [bail=<code>] [lost]
```

- `<v>` is a non-negative integer or `not-reported`; `<n>` is an integer
  of 2 or more.
- `<breakdown>` names come from `BREAKDOWN_FIELDS`, in that order, all or
  none; Task 1 sets it (empty if the notice gives only a total).
- A segment matches `[a-z0-9][a-z0-9.-]*`; `<path>` is segments joined by
  `/`.
- On a `lost` line every value is `not-reported`.

**The segment vocabulary** (a segment is a base name plus optional
suffixes):

| Base | Used for |
| --- | --- |
| `issue-manager.<k>` | a manager run's root, `k` ≥ 1; on today's flow (wave 1), `issue-implement.<k>`, `issue-deploy.<k>`, `issue-validate.<k>` the same way |
| `epic-implement`, `epic-create`, `epic-retrospective` and any other skill name | a session's root |
| `implement`, `deploy`, `validate` | the workers |
| `review-<n>`, `respond-<n>` | mode 2 rounds, `n` ≥ 1 |
| `spec-pushback-<n>`, `plan-pushback-<n>`, `alignment-<n>` | mode 1 rounds |
| `logic-correctness`, `error-handling`, `contract-integration`, `concurrency-state`, `security`, `spec-compliance`, `verifier` | agentic-review's lenses and Verifier |
| `analyst` | any other paad skill's analysis dispatch, and a stray analyst dispatch in agentic-review (allowed, never required) |
| `dispatch` | a vergil dispatch with no more specific name; the lint asks for a reason |

Suffixes: `.a`, `.b` for partitions of one lens (every instance
suffixed); `.<n>` (`n` ≥ 2) for a fresh agent taking over, which implies
`<base>` (or `<base>.<n-1>`) has a line.

**Bail codes:** `none`, `no-intent`, `no-boundary`, `no-surface`, `other`.
A `BAIL:` reason paad does not map to one of these is written `other`; its
prose stays in `Agents dispatched:`.

**A cost record** is a comment:

```text
Agent cost: <step>
[Supersedes: <comment URL>]
<stanza line>
…
```

`<step>` is one of `implement`, `deploy`, `validate`, `review round <n>`,
`respond round <n>`, `<stage> round <n>` (`spec pushback`, `plan pushback`,
`alignment`), `dispatch issue-manager.<k>`, or `dispatch <segment>`.

**A run record**, the first record of every run that roots its own paths
(`issue-manager`, and on today's flow `issue-implement`, `issue-deploy`,
`issue-validate`):

```text
Run: <root>.<k>
Dispatched by: epic-implement | interactive
```

`k` is one past the highest `k` for that root on the issue, or the value
the dispatch prompt passes.

**A superseding record** keeps the replaced record's first line and adds
`Supersedes: <comment URL>` as its second line.

**A ruling request**:

```text
Ruling request <k>: <n> items
<ID> [default: <answer>] <one-line question; options: <a> | <b> | …>
…
Reply on this issue: `approve` takes every default; `approve` then
`<ID>: <answer>` lines overrides some; `deny` then an `<ID>: <answer>`
line for every item answers each.
```

`k` is one past the highest request on the issue. Item IDs are the
finding IDs the item already has (`R2.3`, `R2.S1`), or `Q<k>.<i>` for any
other question. A security item carries `"security": true` in the items
file **and** an ID matching `^R\d+\.S\d+$`, and its question is exactly
`security item; details are held privately`. An item with only one of the
two marks, or with any other question, is refused.

**A ruling reply**, posted by the human login, after the request: the
first non-empty line is `approve` or `deny` (case-insensitive); every
other non-empty line is `<ID>: <answer>`, with `<answer>` one of that
item's options. A human comment after the request is a **ruling
attempt** when its first non-empty line is `approve` or `deny`, or when any
line starts with one of the request's IDs and a colon; any other comment is
ignored. The latest ruling attempt that parses is the ruling, and a later
one supersedes an earlier one. A ruling attempt that does not parse (an
answer to an unknown ID, an option not offered, a `deny` that leaves an
item unanswered), or an edited ruling attempt, is not a ruling.

**A pause** is a trusted agent comment carrying a `Paused:` line (wave 1),
or a ruling request (wave 2). Each counts as one human interaction.

---

### Task 1: Measurement probe

Establishes by experiment what the design assumes (spec 12, items 1–9).
Run from a main session started with the fleet checkout loaded (`vrg-vm
session`), so the delegate is available.

**Files:** none. Records go on the issue, one comment per item, with the
evidence quoted.

**Interfaces:**

- Produces: the notice field names and their mapping onto `tokens`,
  `tool_uses`, `duration_ms`; `BREAKDOWN_FIELDS`; whether tokens include
  children; whether a continued turn's tokens are cumulative; whether the
  main session's cost is readable and how; the answers on depth,
  nested continuation and the delegate from a sub-agent.

- [ ] **Step 1:** Dispatch a `general-purpose` sub-agent in the foreground,
  and one in the background, each running one `Read`. Record each
  completion notice's usage fields verbatim (item 1).
- [ ] **Step 2:** Dispatch a sub-agent that dispatches one sub-agent of its
  own and returns the child's notice verbatim. Compare the parent's tokens
  with the child's (item 2). Repeat with a child that does more work, so
  the difference is unambiguous.
- [ ] **Step 3:** From the notices and the session transcript's per-message
  usage, record what the token figure counts and whether the notice breaks
  it down; set `BREAKDOWN_FIELDS` (item 3).
- [ ] **Step 4:** Dispatch a sub-agent that ends its turn with a question;
  continue it twice with SendMessage. Record whether each continuation
  produces its own notice, and whether its tokens are cumulative or only
  the new turn's (item 4).
- [ ] **Step 5:** Record whether the main session can read its own cost,
  for example by summing per-message usage from its transcript JSONL, and
  the exact method (item 5).
- [ ] **Step 6:** Dispatch a sub-agent whose `Bash` call needs a permission
  the session does not pre-allow. Record what the human sees and whether
  the notice shows it (item 6).
- [ ] **Step 7:** Dispatch a sub-agent that dispatches `paad:paad-delegate`
  on `paad:agentic-review` against a small throwaway branch in a scratch
  worktree. Record whether the delegate is in the sub-agent's dispatchable
  types (item 9), whether its analysts run at that depth (item 7), and the
  notices at each level.
- [ ] **Step 8:** Have a dispatched sub-agent start a child that ends with
  a question, then end its own turn; continue the parent, and have it
  continue the child with SendMessage (item 8). Record the result.
- [ ] **Step 9:** Compare each finding with the spec. For any
  contradiction, open a spec amendment (a docs PR on `.github`) before the
  dependent tasks start (Delivery). Close the issue with a summary comment
  listing every value later tasks use.

### Task 2: The logging rule doc

**Files:**

- Create: `docs/agent-cost-logging.md`
- Modify: `docs/paad-review-doctrine.md` (one link from §1, no other
  change), `README.md` (docs list)

**Interfaces:**

- Consumes: Task 1's mapping and `BREAKDOWN_FIELDS`; the Formats section.
- Produces: the anchor `docs/agent-cost-logging.md`, which every skill's
  dispatch site links (Task 5's lint looks for that path); the list of
  records that imply a dispatch, under the heading `## Records that imply a
  dispatch` (Task 4 reads the same list from code; the doc and
  `IMPLIES_DISPATCH` in Task 4 must match, and Task 4 has a test that
  diffs them).

- [ ] **Step 1:** Write the doc, in this order: the rule (spec 4, its
  table); the stanza and its grammar (Formats, verbatim); the notice
  mapping and what `tokens` counts (Task 1); numbering, continued calls,
  lost calls and the manager root (spec 3); the segment vocabulary and bail
  codes (Formats); cost records and run records (spec 6, Formats);
  `## Records that imply a dispatch` (mode 1 round comments, mode 2
  findings comments, run records, the worker result comments of
  Task 14), each by its fixed first line, with the exceptions that a round
  comment carrying `Deviation: run interactively` implies nothing, a
  split comment's parts count once, and a superseded record implies
  nothing; where a skill with no issue
  prints its stanzas (spec 4); the main session's own cost (Task 1); the
  time label; write-once (spec 7.1, pointing at Task 13's doc for
  rulings).
- [ ] **Step 2:** Add the doctrine link and the README entry.
- [ ] **Step 3:** `vrg-container-run -- vrg-validate`. Expected: pass.
- [ ] **Step 4:** Commit (`docs(agent-cost): the logging rule`) and
  report-ready.

### Task 3: `vrg-agent-cost`: the stanza and `record`

**Files:**

- Create: `src/vergil_tooling/lib/agent_cost.py`,
  `src/vergil_tooling/lib/issue_comments.py`,
  `src/vergil_tooling/bin/vrg_agent_cost.py`
- Modify: `pyproject.toml` (`vrg-agent-cost =
  "vergil_tooling.bin.vrg_agent_cost:main"`),
  `docs/site/docs/reference/cli-tools-overview.md`
- Test: `tests/vergil_tooling/test_agent_cost.py`,
  `tests/vergil_tooling/test_issue_comments.py`,
  `tests/vergil_tooling/test_vrg_agent_cost.py`

**Interfaces:**

- Consumes: `BREAKDOWN_FIELDS` from Task 1.
- Produces:
  - `@dataclass(frozen=True) class Stanza: call: tuple[str, ...]; turn:
    int | None; tokens: int | None; breakdown: dict[str, int | None];
    tool_uses: int | None; duration_ms: int | None; bail: str | None;
    lost: bool` (`None` means `not-reported`);
  - `parse_stanza(line: str) -> Stanza` (raises `StanzaError(str)`),
    `format_stanza(s: Stanza) -> str`;
  - `@dataclass(frozen=True) class CostRecord: step: str; supersedes: str |
    None; stanzas: tuple[Stanza, ...]`;
    `parse_cost_record(body: str) -> CostRecord | None` (`None` when the
    first line is not `Agent cost:`; raises `StanzaError` naming the line
    when a body line does not parse);
  - `SEGMENT_RE`, `BAIL_CODES`, `BREAKDOWN_FIELDS`, `segment_known(seg:
    str) -> bool`;
  - `@dataclass(frozen=True) class Comment: url: str; author: str;
    created_at: str; edited: bool; body: str`;
    `trusted_comments(ref: IssueRef) -> list[Comment]` (GraphQL, oldest
    first, all pages, `edited` from `lastEditedAt`, authors filtered by
    `load_trusted_authors()`).
  - `vrg-agent-cost record --issue <ref> --step <step> --stanza-file
    <path> [--supersedes <url>]`.

- [ ] **Step 1: Write the failing tests** in `test_agent_cost.py`:
  - `test_parse_minimal`:
    `parse_stanza("agent-cost: v1 call=issue-manager.1/review-2/security tokens=1200 tool_uses=3 duration_ms=4500 bail=none")`
    gives `call == ("issue-manager.1", "review-2", "security")`, `tokens ==
    1200`, `bail == "none"`, `lost is False`;
  - `test_parse_not_reported`: `tokens=not-reported` gives `tokens is
    None`;
  - `test_parse_turn`: `turn=2` gives `turn == 2`; `turn=1` raises;
  - `test_parse_lost_requires_not_reported`: a `lost` line with
    `tokens=5` raises;
  - `test_parse_rejects`: out-of-order fields, a space in a value, an
    unknown bail code, `v2`, a trailing word, an uppercase segment each
    raise `StanzaError`;
  - `test_round_trip`: `format_stanza(parse_stanza(x)) == x` for each valid
    example above;
  - `test_segment_known`: `implement.2`, `logic-correctness.a`,
    `review-3`, `issue-manager.4` are known; `reviewer`, `review-0`,
    `issue-manager` are not;
  - `test_parse_cost_record`: a body `Agent cost: review round 2\n<two
    stanza lines>` parses with `step == "review round 2"` and two stanzas;
    a `Supersedes:` second line sets `supersedes`; a body whose first line
    is `Review cost: …` returns `None`; a body with one bad line raises
    naming it.
- [ ] **Step 2:** `vrg-container-run -- vrg-validate`. Expected: the new
  tests FAIL (module missing).
- [ ] **Step 3:** Implement `lib/agent_cost.py` to the Interfaces above and
  the Formats section; one compiled regex per field, no partial matches.
- [ ] **Step 4: Write the failing tests** in `test_issue_comments.py`,
  with GraphQL stubbed the way existing `lib/github.py` callers' tests
  stub it: two pages are joined oldest first; an untrusted author's comment
  is dropped; `lastEditedAt` set gives `edited is True`; a GraphQL error
  raises (never an empty list).
- [ ] **Step 5:** Implement `lib/issue_comments.py`.
- [ ] **Step 6: Write the failing tests** in `test_vrg_agent_cost.py` for
  `record`: it posts `Agent cost: <step>\n<lines>` through the existing
  comment-posting helper; a stanza file with a bad line exits 1 and posts
  nothing; an unknown segment exits 1; `--supersedes` adds the second line.
- [ ] **Step 7:** Implement `bin/vrg_agent_cost.py` with `record` and the
  entry point; document it in `cli-tools-overview.md`.
- [ ] **Step 8:** `vrg-container-run -- vrg-validate`. Expected: pass.
- [ ] **Step 9:** Commit (`feat(agent-cost): stanza parser and record
  command`) and report-ready.

### Task 4: `vrg-agent-cost`: `report` and the audit

**Files:**

- Create: `src/vergil_tooling/lib/agent_cost_report.py`
- Modify: `src/vergil_tooling/bin/vrg_agent_cost.py` (`report`),
  `docs/site/docs/reference/cli-tools-overview.md`
- Test: `tests/vergil_tooling/test_agent_cost_report.py`

**Interfaces:**

- Consumes: Task 3's `trusted_comments`, `parse_cost_record`,
  `segment_known`; Task 1's answers on children and turns.
- Produces:
  - `IMPLIES_DISPATCH: tuple[re.Pattern[str], ...]`, the first-line
    patterns of Task 2's list;
  - `@dataclass class Finding: level: Literal["error", "gap"]; url: str;
    message: str`;
  - `@dataclass class Report: calls: dict[tuple[str, ...], CallTotals];
    by_activity: dict[str, Totals]; human_interactions: int; findings:
    list[Finding]; pre_rule: bool`;
  - `build_report(comments: list[Comment]) -> Report`, `epic_report(epic:
    IssueRef) -> Report` (children plus the `Planning:` issue);
  - `next_run_index(comments: list[Comment], root: str) -> int`;
  - `vrg-agent-cost report (--issue <ref> | --epic <ref>) [--json]
    [--next-run-index <root>]`: exit 1 on any error finding, 0 otherwise.

- [ ] **Step 1: Write the failing tests**, each building comments in
  memory:
  - `test_rollup_sums_children` and `test_rollup_subtracts_children`
    (whichever Task 1 found; the other is deleted with a comment citing
    Task 1);
  - `test_turns_combined` (cumulative or summed, per Task 1);
  - `test_report_ignores_stanzas_outside_cost_records`: a findings comment
    quoting a stanza line, valid or not, adds nothing and no finding;
  - `test_audit_errors_on_malformed_line`: a cost record with one bad line
    gives one error carrying that comment's URL;
  - `test_audit_errors_on_missing_dispatch`: a `agentic-review round 2:
    findings` comment with no `Agent cost: review round 2` record is an
    error;
  - `test_superseded_record_implies_nothing`: a findings comment and its
    superseding comment give one implied dispatch;
  - `test_interactive_round_implies_nothing`: a round comment carrying
    `Deviation: run interactively` implies no dispatch;
  - `test_split_comment_counts_once`: parts `(part 1/2)` and `(part 2/2)`
    of one round comment imply one dispatch;
  - `test_audit_errors_on_orphan_parent`, `test_audit_errors_on_unknown_segment`,
    `test_audit_errors_on_duplicate_path`,
    `test_audit_errors_on_missing_predecessor` (`implement.2` without
    `implement`), `test_audit_errors_on_unmatched_manager` (`Run:
    issue-manager.2 / Dispatched by: epic-implement` with no
    `epic-implement/issue-manager.2` line),
    `test_audit_errors_on_edited_agent_record` (any record that implies a
    dispatch, a cost record, a run record or a ruling request);
  - `test_gaps_counted`: `not-reported` values and a `lost` line are gaps,
    exit 0;
  - `test_pre_rule`: an issue with no cost records reports `pre_rule`, no
    errors;
  - `test_next_run_index`: no `Run: issue-manager.<k>` records gives 1;
    records for 1 and 3 give 4; an `issue-implement` run does not count
    toward `issue-manager`;
  - `test_human_interactions`: counts each `Ruling request` record and
    each `Paused:` line in a trusted agent comment, per Formats, and the
    same issue in either shape gives the same count;
  - `test_implies_dispatch_matches_rule_doc`: the patterns match the first
    lines listed in a fixture copy of Task 2's section (the fixture is
    updated whenever the doc's list changes).
- [ ] **Step 2:** `vrg-container-run -- vrg-validate`. Expected: FAIL.
- [ ] **Step 3:** Implement `lib/agent_cost_report.py` and `report`. Human
  output: per path, own and total; per activity; human interactions; time
  labelled "may include unseen permission waits"; gaps and errors listed
  last. `--json` gives the same data.
- [ ] **Step 4:** `vrg-container-run -- vrg-validate`. Expected: pass.
- [ ] **Step 5:** Commit (`feat(agent-cost): roll-up and audit`) and
  report-ready.

### Task 5: `vrg-agent-cost lint` and its `vrg-validate` stage

**Files:**

- Create: `src/vergil_tooling/lib/agent_cost_lint.py`
- Modify: `src/vergil_tooling/bin/vrg_agent_cost.py` (`lint`),
  `src/vergil_tooling/bin/validate_common.py` (stage `agent-cost-lint`, run
  when `skills/*/SKILL.md` exists)
- Test: `tests/vergil_tooling/test_agent_cost_lint.py`

**Interfaces:**

- Produces: `lint_skills(skills_dir: Path) -> list[str]` (one `FAIL:` line
  per site); exception marker `<!-- agent-cost: exempt (<reason>) -->` on
  the site's line or the line above.

- [ ] **Step 1: Write the failing tests:** a site recognised by any of
  `Agent tool`, `subagent`, `sub-agent`, `dispatch` (word, any case) that
  neither links `agent-cost-logging.md` within its paragraph nor names
  where its stanza is recorded fails, with file and line; a site with the
  link passes; the marker with a reason passes and without one fails; an
  unreadable `SKILL.md` fails; `test_current_skills_all_flagged` runs on a
  fixture copy of today's plugin skills and expects every known dispatch
  site flagged (the list in the fixture).
- [ ] **Step 2:** `vrg-container-run -- vrg-validate`. Expected: FAIL.
- [ ] **Step 3:** Implement the lint and the stage (`Running:
  agent-cost-lint (<n> skills)`).
- [ ] **Step 4:** `vrg-container-run -- vrg-validate`. Expected: pass.
  (vergil-tooling holds no `skills/`, so the stage does not run here.)
- [ ] **Step 5:** Commit (`feat(agent-cost): dispatch-site lint`) and
  report-ready. vergil-claude-plugin's validation would fail on the stage
  without Task 8, so Task 10 releases this and merges Task 8 in one
  sitting.

### Task 6: paad: the stanza convention, agentic-review and the lint

**Files:**

- Create: `scripts/check_agent_cost.py`
- Modify: `Makefile` (`check-agent-cost` in `tree-checks`; its self-test in
  `self-tests`), `preview/paad/skills/agentic-review/SKILL.md`,
  `preview/paad/skills/pushback/SKILL.md` and
  `preview/paad/skills/alignment/SKILL.md` (their unattended results),
  `preview/paad/agents/paad-delegate.md` (the result format),
  `preview/paad/skills/paad-help/SKILL.md`, `CHANGELOG.md`

**Interfaces:**

- Produces: the convention text (paad's own words, no vergil names); in
  agentic-review's Review Metadata, one stanza line per specialist and
  Verifier dispatch; in its unattended result, the skill-specific field
  `Agent costs:` after `Verifier warnings:`, followed by the stanza lines
  (relative paths: `call=security`, `call=logic-correctness.b`,
  `call=verifier.2`); in every delegate result, including pushback's and
  alignment's, the field `Agent costs:` (`none` when nothing was
  dispatched); `check_agent_cost.py <skills_dir>`.

- [ ] **Step 1: Write the lint's self-tests first**, using
  `check-config`'s all-or-nothing pattern: a tree where no skill carries the
  `agent-cost:` convention is skipped with a `SKIP` line; in a tree where
  any skill carries it, a skill with a `subagent_type` line and no
  convention reference fails; a skill with both passes; a skill with
  neither passes; an unreadable file fails.
- [ ] **Step 2:** `python3 scripts/check_agent_cost.py --self-test`.
  Expected: FAIL.
- [ ] **Step 3:** Implement it in the style of `check_unattended.py`; wire
  the Makefile.
- [ ] **Step 4:** Write the convention in `paad-help`: the stanza grammar
  (Formats), relative paths, the caller writes the line from the
  completion notice, `not-reported` for a missing value, lens slugs and
  bail codes.
- [ ] **Step 5:** In agentic-review: after each specialist and the Verifier
  return, write its stanza from the completion notice; record them in
  Review Metadata and in the result's `Agent costs:` field; a re-dispatch
  takes `.2`; partitions `.a`/`.b`; a stray analyst dispatch is logged as
  `analyst`; bail codes from the `BAIL:` token. In pushback, alignment and
  the delegate's result format, add `Agent costs:` (`none`, or the lines
  for any analyst dispatched).
- [ ] **Step 6:** Do Task 7's steps in the same branch before going on;
  `make test` passes only once every dispatching skill in `preview/paad`
  carries the convention, and `plugins/paad` is skipped until paad promotes
  it.
- [ ] **Step 7:** `make export && make test`. Expected: `All checks
  passed.`
- [ ] **Step 8 (human):** Smoke: run agentic-review interactively on a
  small branch with the clone's preview tree; the agent checks the pasted
  Review Metadata has six specialist lines and one Verifier line, each
  parsing with the Formats grammar. Then dispatch it through the delegate;
  the result carries `Agent costs:` with the same lines.
- [ ] **Step 9 (human):** Commit Tasks 6 and 7 together on
  `unattended-mode` in paad's style and push. The agent closes the issue
  with the commit and the `make test` last line.

### Task 7: paad: the other dispatching skills

**Files:** `preview/paad/skills/{agentic-owasp,agentic-architecture,agentic-a11y,agentic-dedup,backlog,rethink}/SKILL.md`,
and any other skill `check_agent_cost.py` flags; `paad-help`,
`CHANGELOG.md`.

Done in Task 6's branch and commit, at its Step 6.

- [ ] **Step 1:** Run `make tree-checks TREE=preview/paad`. Expected:
  `check-agent-cost` FAIL listing each skill (partial adoption).
- [ ] **Step 2:** For each, add the stanza lines to the report's metadata
  section and final output (segment `analyst`, suffixed `.2`, `.3` per
  dispatch), without moving or changing an existing field.
- [ ] **Step 3:** `make tree-checks TREE=preview/paad`. Expected:
  `check-agent-cost` passes.
- [ ] **Step 4 (human):** Smoke one skill (rethink, the cheapest) and
  check its stanza lines parse. Then return to Task 6 Step 7.

### Task 8: Logging and write-once on today's skills

**Files:** `skills/epic-create/SKILL.md`, `skills/epic-implement/SKILL.md`,
`skills/issue-implement/SKILL.md`, `skills/issue-deploy/SKILL.md`,
`skills/issue-validate/SKILL.md`, every other skill Task 5's lint flags;
`docs/paad-review-doctrine.md` §1.7 and §3.5 (the unfinished-comment,
`--edit-last` and delete-and-repost paths), §2.6 (round comment: the cost
record) and §3.5 (`Review cost:` replaced by a cost record).

- [ ] **Step 1:** Validate this branch with the vergil-tooling release that
  carries Task 5 (Task 10 Step 1). Expected: `agent-cost-lint` FAIL, the
  sites listed.
- [ ] **Step 2:** At each site, after the completion notice: write the
  stanza and post it with `vrg-agent-cost record`, linking
  `docs/agent-cost-logging.md`. `epic-create`'s mode 1 rounds record
  `Agent cost: <stage> round <n>` (one line for the delegate; the
  delegate's own children come from Task 9's field once it is installed).
  `epic-implement` passes each dispatched skill its `k` (from `vrg-agent-cost
  report --next-run-index <skill>`) and records `Agent cost: dispatch
  <skill>.<k>`; each of today's `issue-implement`, `issue-deploy` and
  `issue-validate` runs posts its run record first and roots its own
  records at `<skill>.<k>`.
  `issue-implement`'s mode 2 records `Agent cost: review round <n>` in
  place of `Review cost:`, posted straight after the completion notice and
  before the findings comment, and the gate's summary takes its review
  total from `vrg-agent-cost report`.
- [ ] **Step 2a: Write-once.** Replace every edit and delete path in §1.7
  and §3.5, and in `issue-implement`, with superseding records (Formats):
  finishing a paused round posts a new comment that keeps the paused
  comment's first line and adds `Supersedes: <url>`; the `dismiss` repost
  does the same. No `--edit-last` or `--delete-last` remains in any skill
  or doctrine text.
- [ ] **Step 3:** Validate again against the same release. Expected:
  `agent-cost-lint` passes, and `vrg-git grep -n -e '--edit-last' -e
  '--delete-last' -- skills docs/paad-review-doctrine.md` prints nothing.
- [ ] **Step 4:** `vrg-container-run -- vrg-validate`. Expected: pass.
- [ ] **Step 5:** Commit (`feat(skills): log every dispatch's cost`) and
  report-ready.

### Task 9: Doctrine: the result's `Agent costs:` field

**Files:** `docs/paad-review-doctrine.md` §1.9 (the result format and the
output checks), §3.4 (mode 2's values), §2.4 (mode 1's values);
`docs/paad-policy.md` §10 (a line naming the field as part of the
contract).

- [ ] **Step 1:** Add `Agent costs:` to the result format after the
  skill-specific fields, required in every delegate result: `none` when
  nothing was dispatched, otherwise the stanza lines.
- [ ] **Step 2:** Add the output checks: each line matches the stanza
  grammar exactly (Formats); for agentic-review, at least one line for each
  lens in `Agents dispatched:` and at least one for the Verifier, with
  partition and `.<n>` suffixes and `analyst` lines allowed; for pushback
  and alignment, `Agent costs: none` with `paad:paad-analyst` listed in
  `Agent types dispatched:` fails. A malformed line or a lens with no line
  is a failed check, handled as any other. The caller prefixes each path
  with its own and records them in the round's cost record.
- [ ] **Step 3:** `vrg-container-run -- vrg-validate`. Expected: pass.
- [ ] **Step 4:** Commit (`docs(paad-doctrine): the result's agent costs`)
  and report-ready. Not merged until Task 10 says so.

### Task 10: Deploy wave 1

Operational; the order matters, because the result's syntax changes
(paad-policy §10).

- [ ] **Step 1 (human):** Merge Tasks 3, 4 and 5; release vergil-tooling;
  sync the consumer repos. The agent records the release. Steps 1 and 2
  happen in one sitting.
- [ ] **Step 2 (human):** The agent validates Task 8's branch against the
  released tooling (Task 8 Steps 1 and 3). Then merge Tasks 2 and 8 and
  release the plugin. The agent records it and checks `vrg-container-run --
  vrg-validate` passes in vergil-claude-plugin on develop.
- [ ] **Step 3 (human):** Hold reviews. Move `fleet` to the commit carrying
  Tasks 6 and 7, and record it. Update the #733 validation sets' expected
  results for the new field, then run the gate on the new commit.
- [ ] **Step 4 (human):** Once the gate passes, merge Task 9 and release
  the plugin. Reviews resume. The agent records each step's evidence and
  closes the issue.

### Task 11: Baseline sample on today's flow

- [ ] **Step 1:** Pick at least two docs-only and two code tasks from real
  queued work; record the list.
- [ ] **Step 2:** Run each through today's flow (`epic-implement` or
  `issue-implement`).
- [ ] **Step 3:** For each, run `vrg-agent-cost report --issue <ref>` and
  record the output, the audit result, and per-specialist cost and bail.
- [ ] **Step 4:** Post a summary on the issue: per task and per activity;
  docs-only against code; human interactions, counted per Formats; the
  paad fleet commit used. Note on #812 that it may be revived from these
  numbers.
- [ ] **Step 5 (human):** Review the baseline and confirm on the issue that
  wave 2 may start. Close the issue.

### Task 12: `vrg-ruling`

Carries GHSA-9q3r-h2xj-rgq3: the requirement is the posting-identity check
below; no detail goes in public records.

**Files:**

- Create: `src/vergil_tooling/lib/rulings.py`,
  `src/vergil_tooling/bin/vrg_ruling.py`
- Modify: `pyproject.toml` (`vrg-ruling =
  "vergil_tooling.bin.vrg_ruling:main"`),
  `docs/site/docs/reference/cli-tools-overview.md`
- Test: `tests/vergil_tooling/test_rulings.py`,
  `tests/vergil_tooling/test_vrg_ruling.py`

**Interfaces:**

- Consumes: Task 3's `trusted_comments`.
- Produces:
  - `@dataclass(frozen=True) class Item: id: str; question: str; options:
    tuple[str, ...]; default: str`;
  - `render_request(k: int, items: list[Item]) -> str`;
  - `@dataclass class Ruling: request: int; answers: dict[str, str]`;
    `read_ruling(comments: list[Comment], human: str) -> Ruling | Pending
    | NotARuling(reason: str)` (Pending when no ruling attempt follows the
    request);
  - `check_posting_identity() -> None` (raises `IdentityError` naming the
    fix);
  - `vrg-ruling request --issue <ref> --items-file <json>`: checks
    identity, posts the request, prints `k`;
  - `vrg-ruling read --issue <ref> --request <k>`: checks identity; exit 0
    printing the answers as JSON; exit 3 when no ruling attempt yet; exit
    4 printing the reason when the latest ruling attempt is not a ruling.

- [ ] **Step 1: Write the failing tests** in `test_rulings.py`:
  - `test_render_request` gives the Formats text, with `k` and every item;
  - `test_read_approve`: `approve` gives every default;
  - `test_read_approve_with_overrides`: `approve\nR2.3: file` changes one;
  - `test_read_deny_complete`: `deny` plus a line per item gives those
    answers;
  - `test_read_deny_incomplete_not_a_ruling`;
  - `test_read_unknown_id_not_a_ruling`, `test_read_bad_option_not_a_ruling`;
  - `test_read_ignores_replies_before_request`;
  - `test_read_ignores_non_ruling_comments`: a valid ruling followed by an
    unrelated human comment still reads as that ruling;
  - `test_read_later_ruling_supersedes`: two valid rulings give the later;
  - `test_read_rejects_reply_by_agent_login`: a reply from an agent login
    is ignored (pending if it is the only one);
  - `test_read_reports_edited_reply`: an edited human reply gives
    `NotARuling("edited")`;
  - `test_security_items_by_id_only`: `render_request` refuses an item
    with `security: true` and an ID not matching `^R\d+\.S\d+$`; an item
    with ID `R2.S1` and no flag; and a flagged `R2.S1` item whose question
    is not exactly `security item; details are held privately`. A flagged
    `R2.S1` with that question passes, and unflagged `R2.3` and `S2.3`
    items pass.
- [ ] **Step 2: Write the failing tests** for `check_posting_identity`:
  `test_identity_check_refuses_human_mode` (`identity_mode.resolve()`
  gives human); `test_identity_check_refuses_without_app_token`
  (`get_installation_token(require_installation=True)` gives `None`);
  `test_identity_check_refuses_login_not_agent` (the token's login is not
  in `load_trusted_authors().agents`); `test_identity_check_passes`. The
  token's login comes from GraphQL `viewer { login }` under the token; if
  Step 3 finds the API names it differently for an App, follow the API and
  record it in the module docstring.
- [ ] **Step 3:** `vrg-container-run -- vrg-validate`. Expected: FAIL.
- [ ] **Step 4:** Implement `lib/rulings.py` and `bin/vrg_ruling.py`.
  Every refusal names the fix: start the session in agent mode with App
  credentials.
- [ ] **Step 5:** `vrg-container-run -- vrg-validate`. Expected: pass.
- [ ] **Step 6:** Commit (`feat(rulings): ruling requests and replies`)
  and report-ready.

### Task 13: The ruling protocol doc

**Files:** Create `docs/ruling-protocol.md`; modify
`docs/agent-cost-logging.md` (link), `README.md`.

- [ ] **Step 1:** Write the doc: write-once records and superseding (spec
  7.1, Formats); the request, the reply grammar and reading (spec 7.2,
  Formats); never relayed through prompts; security items by ID only;
  `epic-implement` relays nothing; the per-session identity check and its
  fix; how to use `vrg-ruling`; what counts as a human interaction.
- [ ] **Step 2:** `vrg-container-run -- vrg-validate`. Expected: pass.
- [ ] **Step 3:** Commit (`docs(rulings): the ruling protocol`) and
  report-ready.

### Task 14: The manager split

Carries GHSA-2fjg-qxrj-fc2v: the requirement is the responder's security
contract below. Merged with Tasks 15 and 17.

**Files:**

- Create: `skills/issue-manager/SKILL.md` (from today's
  `skills/issue-implement/SKILL.md`), `skills/issue-respond/SKILL.md`
- Rewrite as workers: `skills/issue-implement/SKILL.md`,
  `skills/issue-deploy/SKILL.md`, `skills/issue-validate/SKILL.md` (and its
  lifecycle reference)
- Modify: `skills/epic-implement/SKILL.md`

**Interfaces:**

- Consumes: Tasks 4, 12, 13; Task 1 items 7–9.
- Produces: the marker line `Dispatched by issue-manager: <issue ref>`
  that every worker requires as its prompt's first line; each worker's
  result form (below); the worker result comments' first lines, `Worker
  result: <worker> <attempt>`, which Task 2's list names.

- [ ] **Step 1:** Write `issue-manager` from today's `issue-implement`:
  every trigger of the old three skills; preflight, worktree,
  preconditions; at start, `vrg-agent-cost report --next-run-index
  issue-manager` (or `k` from the dispatch prompt) and the run record; the kind
  dispatch; every human decision through `vrg-ruling`; the mode 2 loop per
  Task 15; a cost record per dispatch; resuming by SendMessage with only
  "the human has responded on the issue", a fresh suffixed worker with a
  `lost` line for the original when it cannot be continued; the task
  roll-up from `vrg-agent-cost report` before report-ready or closing,
  noting that a dispatched manager's own cost is recorded after it. No
  `--edit-last` or `--delete-last` survives from today's skill.
- [ ] **Step 2:** Write the workers. Each starts by checking the marker
  line and otherwise stops with "This skill runs only under
  `issue-manager`; run `issue-manager <issue>`." before any other step.
  Each states its input (from the prompt) and its result, ending with a
  fixed block: `Status: complete | needs-human`, `Commits:`,
  `Validation:`, `Proposed follow-ups:`, `Problems:`; the deployer and
  validator add `Outcome: SUCCESS | FAILURE` with evidence. Workers post
  nothing and close nothing.
- [ ] **Step 3:** Write `issue-respond`: fresh each round; fixes findings
  sorted `agent` and ruled `fix`; security fixes arrive only as
  state-directory paths with hashes, each checked before reading; reports
  security items by ID only; commits and re-validates.
- [ ] **Step 4:** `epic-implement`: dispatch only `issue-manager`, passing
  `k`; record `Agent cost: dispatch issue-manager.<k>`; batch what needs
  the human by pointing at each issue's ruling request; continue each
  manager after the human replies there; relay nothing.
- [ ] **Step 5:** Run Task 5's lint and `vrg-container-run --
  vrg-validate`. Expected: pass.
- [ ] **Step 6:** Commit (`feat(skills): split issue-implement into a
  manager and workers`) and report-ready.

### Task 15: Doctrine: mode 2 for the new roles and records

Merged with Tasks 14 and 17.

**Files:** `docs/paad-review-doctrine.md` §3 (title, §3.1 steps 1–7,
§3.5, §3.6–3.8). Write-once (§1.7, §3.5's edit and delete paths) already
landed in Task 8.

- [ ] **Step 1:** Rewrite mode 2 so the manager orchestrates, the reviewer
  is fresh each round, the responder fixes, every pause and ruling goes
  through the ruling protocol, the hold-back judgement stays with the
  manager, and security fixes reach the responder only as hashed paths.
- [ ] **Step 2:** `vrg-container-run -- vrg-validate`. Expected: pass.
- [ ] **Step 3:** Commit (`docs(paad-doctrine): mode 2 under issue-manager`)
  and report-ready.

### Task 16: `epic-retrospective`: the epic cost summary

- [ ] **Step 1:** Add a step: run `vrg-agent-cost report --epic <ref>`;
  post its output as a final comment on the epic, opening `Agent cost
  summary:` and stating that it excludes the retrospective task itself and
  that a straddling epic's summary is partial; the retrospective doc links
  the comment and repeats no numbers.
- [ ] **Step 2:** `vrg-container-run -- vrg-validate`. Expected: pass.
- [ ] **Step 3:** Commit (`feat(epic-retrospective): post the epic cost
  summary`) and report-ready.

### Tasks 17–19: Rename sweeps

Each finds references by search (`vrg-git grep -n 'issue-implement'`, and
`issue-deploy`, `issue-validate` as entry points), not from memory, and
leaves releases, dated specs and plans unchanged.

- [ ] **Task 17 (vergil-claude-plugin):** the other skills (`epic-create`,
  `deferred-review`, `intake-review`, `planning-review`, `pr-watch`,
  `steering-audit`), `README.md`, `CLAUDE.md`,
  `docs/development/skills-architecture.md`,
  `docs/development/starting-work-on-an-issue.md`, `docs/paad-policy.md`,
  `docs/site/docs/skills/index.md`. Merged with Tasks 14 and 15.
- [ ] **Task 18 (vergil-tooling):** `src/vergil_tooling/bin/vrg_submit_pr.py`
  (its refusal text, with its test), `docs/site/docs/guides/account-setup.md`,
  `docs/site/docs/guides/security-records.md`,
  `docs/site/docs/standards/github-issues.md`.
- [ ] **Task 19 (`docs`):** `docs/site/docs/management/tools-and-skills.md`,
  `docs/site/docs/management/workflow.md`.

Each: search, edit, `vrg-container-run -- vrg-validate` (pass), commit,
report-ready.

### Task 20: Deploy wave 2

- [ ] **Step 1 (human):** Merge Tasks 12 and 18; release vergil-tooling;
  sync consumers.
- [ ] **Step 2 (human):** Merge Tasks 13, 16, then 14, 15 and 17 together;
  release the plugin. Merge Task 19.
- [ ] **Step 3:** In a fresh agent session, run `vrg-ruling read` against
  a test request to confirm the identity check passes; record it. Close the
  issue.

### Task 21: Validation sample

- [ ] **Step 1:** Pick at least three docs-only and three code tasks from
  real queued work; record the list.
- [ ] **Step 2:** Run each through the new flow, rulings on the issue.
- [ ] **Step 3:** For each, `vrg-agent-cost report --issue <ref>`: no
  errors; gaps only for `lost` calls or values Task 1 showed unreadable.
  Check every dispatch left a stanza, paths nest, and rulings went through
  `vrg-ruling`.
- [ ] **Step 4:** Post the comparison: per task and activity, docs-only
  against code, per-specialist cost and bail, against Task 11's baseline,
  with the paad fleet commit used. Any failure is fixed before the epic
  closes. Close the issue.

### Task 22: Offer the paad changes upstream (human)

- [ ] **Step 1 (human):** Offer the Tasks 6 and 7 commit to `Ovid/paad` as PRs from
  the fork, in paad's terms. The agent records the PR refs on the issue
  without cross-org links. No task waits on them.
