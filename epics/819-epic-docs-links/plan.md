# Epic issues link their spec, plan and retrospective: Implementation Plan

- **Epic:** vergil-project/.github#819
- **Planning issue:** vergil-project/.github#815 (review history is in its
  comments)
- **Status:** draft, in design review

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Every epic body opens with a standard Documents section linking its spec, plan and retrospective, or a note saying why one does not exist. The skills keep it current, the audit checks it, and every org's existing epics are backfilled.

**Architecture:** A new `lib/epic_docs.py` in vergil-tooling owns the section's format, the entry rules and the document discovery. `vrg-epic-docs` (`set`, `backfill`) and a new `vrg-epic-audit` check both use it. The plugin skills call `vrg-epic-docs set` at epic creation and at each docs handoff. Deployment tasks then run the backfill org by org.

**Tech Stack:** Python 3.14 (vergil-tooling, pytest, the `lib/github.py` wrappers), Markdown skills (vergil-claude-plugin).

**Spec:** `epics/819-epic-docs-links/spec.md`

## Global Constraints

- Markers: `<!-- vrg-epic-docs -->` and `<!-- /vrg-epic-docs -->`; heading line `**Documents**`; entries in the order Spec, Plan, Retrospective (spec §2).
- The section is the first thing in the body (spec §2).
- Link text is the repo-relative path (`<repo>/<path>` for another repo); the target is `https://github.com/<owner>/<repo>/blob/<default-branch>/<path>`, with the default branch queried from GitHub (spec §2).
- Notes, verbatim: `pending`; `not written (closed as not planned)`; `not written (predates the spec and plan convention)`; `not written (predates the retrospective convention)`; `not written` (spec §3).
- Entry rules are read top to bottom, first match wins. The cutoffs compare the epic's **creation** date: spec and plan `2026-07-12`, retrospective `2026-07-27` (spec §3).
- Skips: `epics.is_standing` labels, the `standing` label, titles starting `Epic (standing):` or `[archived]` (spec §5.2).
- No silent failures: every API error is named and fails the run, apart from a contents-API 404, which means "absent" (CLAUDE.md).
- Agents never cut a release or run `vrg-submit-pr`. vergil-tooling validation is `vrg-container-run -- vrg-validate` only.

## Review Focus

1. **Half a section in the body** (only one marker, or END before BEGIN, for example after a hand edit): `upsert` raises and nothing is edited. Pinned in Task 1.
2. **CRLF line endings** (GitHub's web editor saves bodies with `\r\n`): markers and legacy lines are still found, and the rest of the body is kept byte for byte. Pinned in Task 1.
3. **A legacy-looking line inside a fenced code block:** it is not removed. Pinned in Task 1.
4. **An empty or missing body:** the result is the section alone. Pinned in Task 1.
5. **A non-404 API error partway through a backfill:** that epic is named as failed, the run carries on, and the exit status is 1. Pinned in Task 2.

## Task order and placement

| # | Task | Repo (where it closes) | Kind | Blocked by |
|---|---|---|---|---|
| 1 | `epic_docs` library and `vrg-epic-docs set` | vergil-tooling | task | — |
| 2 | `vrg-epic-docs backfill` | vergil-tooling | task | 1 |
| 3 | `vrg-epic-audit` Epic documents check, and the site docs | vergil-tooling | task | 1 |
| D1 | Backfill vergil-project | vergil-project/.github | deployment | 2, 3 |
| 4 | Skills write the section (vergil-project/vergil-claude-plugin#837) | vergil-claude-plugin | task | D1 |
| D2–D6 | Backfill logical-minds-foundry, mq-rest-admin-project, mnemosys-project, diogenes-project, vergils-nemesis (one task each) | vergil-project/.github | deployment | D1 |
| D7 | Confirm no epics under the `wphillipmoore` account | vergil-project/.github | deployment | D1 |

Task 4 waits for D1 so that the skills never call a `vrg-epic-docs` that is not installed yet. The existing bookends follow: the documentation review (vergil-project/vergil-tooling#3895), then the retrospective (vergil-project/.github#821).

---

### Task 1: `epic_docs` library and `vrg-epic-docs set`

**Files:**

- Create: `src/vergil_tooling/lib/epic_docs.py`, `src/vergil_tooling/bin/vrg_epic_docs.py`
- Modify: `pyproject.toml` (register `vrg-epic-docs = "vergil_tooling.bin.vrg_epic_docs:main"` next to `vrg-epic-audit`)
- Test: `tests/vergil_tooling/test_epic_docs.py`, `tests/vergil_tooling/test_vrg_epic_docs.py`

**Interfaces:**

- Produces, in `lib/epic_docs.py`:
  - `DOCS = ("spec", "plan", "retrospective")`, `BEGIN`, `END`, and the five note constants `PENDING`, `NOT_PLANNED`, `PREDATES_SPEC_PLAN`, `PREDATES_RETRO`, `NOT_WRITTEN`
  - `@dataclass(frozen=True) class DocLink: owner: str; repo: str; path: str; branch: str`, with `.text(home_repo: str) -> str` and `.url -> str`
  - `Entry = DocLink | str` (a `str` entry is one of the five notes)
  - `class EpicDocsFormatError(ValueError)`
  - `render(entries: Mapping[str, Entry], home_repo: str) -> str`: the whole block, markers included
  - `parse(body: str) -> dict[str, Entry] | None`: `None` when there is no section; raises `EpicDocsFormatError` when it is malformed
  - `strip_legacy(body: str) -> tuple[str, list[str]]`: the body without legacy lines, plus the lines removed
  - `upsert(body: str, block: str) -> tuple[str, list[str]]`: replaces the marked region, or prepends the block and a blank line; also runs `strip_legacy`
  - `default_branch(owner: str, repo: str) -> str`
  - `read_body(owner: str, repo: str, number: int) -> str` and `write_body(owner: str, repo: str, number: int, body: str) -> None`, through `github.read_json` / `github.write_json` on `repos/{o}/{r}/issues/{n}`
- `vrg-epic-docs set --epic <owner>/<repo>#<N> [--spec P] [--plan P] [--retrospective P]`, where `P` is `<path>` (home repo) or `<repo>:<path>`

- [ ] **Step 1: Write the failing tests** in `test_epic_docs.py`:
  - `test_render_order_and_markers`: the output starts with `BEGIN`, then `**Documents**`, then lines `- Spec: …`, `- Plan: …`, `- Retrospective: …`, and ends with `END`.
  - `test_render_link_home_and_other_repo`: a home link's text is `` `epics/819-x/spec.md` `` and its URL `https://github.com/vergil-project/.github/blob/develop/epics/819-x/spec.md`; another repo's text is `` `vergil-vm/docs/specs/a.md` ``.
  - `test_parse_roundtrip`: `parse(render(e)) == e` for a mix of links and each of the five notes.
  - `test_parse_none_without_section`.
  - `test_parse_rejects_unknown_note_and_missing_entry`: raises `EpicDocsFormatError`.
  - `test_upsert_prepends_when_absent`: the result is `block + "\n\n" + body`.
  - `test_upsert_replaces_in_place_keeps_rest`: text below the section is unchanged.
  - `test_upsert_half_section_raises` (Review Focus 1): only `BEGIN`, only `END`, or `END` before `BEGIN` all raise `EpicDocsFormatError`.
  - `test_strip_legacy_documents_list`: the #558 form (a `Documents:` line plus three bullets) is removed and reported.
  - `test_strip_legacy_standalone_lines`: `` Spec: `epics/550-x/spec.md` (published by the documentation task) ``, and `Plan:` / `Retrospective:` lines, are removed.
  - `test_strip_legacy_keeps_code_fence_lines` (Review Focus 3).
  - `test_crlf_body_preserved` (Review Focus 2): a `\r\n` body round-trips through `upsert`, with only the section region and legacy lines changed.
  - `test_upsert_empty_body` (Review Focus 4): `upsert("", block)[0] == block`.
- [ ] **Step 2:** Run `vrg-container-run -- vrg-validate`. Expected: the new tests fail (module missing).
- [ ] **Step 3:** Implement `lib/epic_docs.py` with the interfaces above. Normalize to `\n` for matching, and write back with the body's original line ending. `strip_legacy` tracks fence state (lines starting with three backticks) and never removes inside a fence.
- [ ] **Step 4: Write the failing tests** in `test_vrg_epic_docs.py`, mocking `read_body`, `write_body` and `default_branch`:
  - `test_set_seeds_pending_section`: on a body without a section, `set --epic o/.github#1` writes all three entries as `pending`.
  - `test_set_updates_named_entries_only`: `--spec epics/1-x/spec.md` changes Spec and keeps Plan and Retrospective.
  - `test_set_other_repo_path`: `--spec vergil-vm:docs/specs/a.md` gives a `DocLink` with `repo="vergil-vm"`, and the default branch is queried for `vergil-vm`.
  - `test_set_idempotent_no_write`: when the body is already correct, `write_body` is not called.
  - `test_set_does_not_check_existence`: no contents-API call is made.
- [ ] **Step 5:** Implement `bin/vrg_epic_docs.py` with an argparse `set` subcommand (Task 2 adds `backfill`). Its `--help` follows the other `vrg-*` tools.
- [ ] **Step 6:** Run `vrg-container-run -- vrg-validate`. Expected: PASS.
- [ ] **Step 7:** `vrg-commit --type feat --scope epic-docs --message "epic Documents section library and vrg-epic-docs set"`

### Task 2: `vrg-epic-docs backfill`

**Files:**

- Modify: `src/vergil_tooling/lib/epic_docs.py`, `src/vergil_tooling/bin/vrg_epic_docs.py`
- Test: `tests/vergil_tooling/test_epic_docs.py`, `tests/vergil_tooling/test_vrg_epic_docs.py`

**Interfaces:**

- Consumes: Task 1's `DocLink`, `Entry`, notes, `render`, `upsert`, `read_body`, `write_body`, `default_branch`.
- Produces, in `lib/epic_docs.py`:
  - `SPEC_PLAN_CUTOFF = "2026-07-12"`, `RETRO_CUTOFF = "2026-07-27"`
  - `is_skipped(title: str, labels: Iterable[str]) -> bool`
  - `entry_for(doc: str, link: DocLink | None, *, state: str, state_reason: str | None, created_at: str) -> Entry`
  - `class Ambiguous(Exception)`, carrying the candidates
  - `discover(owner: str, home_repo: str, number: int, body: str, branch: str) -> dict[str, DocLink | None]`: raises `Ambiguous`
  - `file_exists(owner: str, repo: str, path: str, branch: str) -> bool`: contents API; a 404 is `False`, any other error propagates
  - `list_epics(home: str) -> list[dict]`: every `epic`-labelled issue, open and closed, with number, title, state, stateReason, createdAt, labels and body, minus `is_skipped`

- [ ] **Step 1: Write the failing tests:**
  - `test_is_skipped`: true for labels `{"epic","ad-hoc"}`, `{"epic","standing"}` and titles `Epic (standing): Ad-hoc maintenance` and `[archived] Epic: x`; false for `{"epic"}` with `Epic: x` or `Sync .gitignore to baseline`.
  - `test_entry_for_table`, parametrized, one row each: link → the `DocLink`; open → `PENDING`; closed `NOT_PLANNED` with a missing retro → `NOT_PLANNED` (this beats the date rows); closed, created `2026-07-01`, spec missing → `PREDATES_SPEC_PLAN`; closed, created `2026-07-16`, retro missing → `PREDATES_RETRO` (the #156 case); closed, created `2026-08-25`, all missing → `NOT_WRITTEN` (the #322 case).
  - `test_discover_epic_dir`: `epics/819-x/` holding `spec.md`, `plan.md` and `audit-findings.md` gives spec and plan links, retro `None`.
  - `test_discover_body_mentioned_dir`: no `epics/<N>-*/`, but the body mentions an existing `epics/553-x/` → that directory is used.
  - `test_discover_member_repo_spec`: a body mentioning `vergil-vm/docs/specs/a.md`, which exists → spec `DocLink(repo="vergil-vm", …)`.
  - `test_discover_two_dirs_ambiguous`: raises `Ambiguous`.
  - `test_backfill_dry_run_no_writes`: prints each epic's block with `add`, `change` or `no change`, each removed legacy line, and a summary; no `write_body` call; exit 0.
  - `test_backfill_apply_writes_changed_only`.
  - `test_backfill_ambiguous_exit_1`: that epic is named and left unchanged, the others are applied, exit 1.
  - `test_backfill_api_error_continues` (Review Focus 5): a non-404 error on one epic is named, the next epic is still processed, exit 1.
  - `test_backfill_repo_flag`: `--repo o/private` reads epics from `o/private`.
- [ ] **Step 2:** Run `vrg-container-run -- vrg-validate`. Expected: the new tests fail.
- [ ] **Step 3:** Implement the functions, then the `backfill (--org ORG | --repo OWNER/REPO) [--apply]` subcommand. `--org` resolves to `<org>/.github`. Writes go inside `github.target_org(owner)`. Discovery runs on the original body, before legacy lines are stripped.
- [ ] **Step 4:** Run `vrg-container-run -- vrg-validate`. Expected: PASS.
- [ ] **Step 5:** `vrg-commit --type feat --scope epic-docs --message "vrg-epic-docs backfill"`

### Task 3: `vrg-epic-audit` Epic documents check, and the site docs

**Files:**

- Modify: `src/vergil_tooling/lib/epic_audit.py` (factor the closed-epic listing out of `closed_epic_open_child` into `_closed_epics(home: str) -> list[dict]`; add the check and its render section), `src/vergil_tooling/bin/vrg_epic_audit.py` (wire it in)
- Modify: `docs/site/docs/standards/github-issues.md` (§Epics, §Creating an epic, §Compliance invariants), `docs/site/docs/reference/cli-tools-overview.md` (`vrg-epic-docs`, and the new audit section)
- Test: `tests/vergil_tooling/test_epic_audit.py`, `tests/vergil_tooling/test_vrg_epic_audit.py`

**Interfaces:**

- Consumes: `epic_docs.parse`, `epic_docs.is_skipped`, `epic_docs.file_exists`.
- Produces: `epic_documents(org: str, *, home: str | None = None) -> list[EpicDocsFinding]`, where `EpicDocsFinding(number: int, state: str, problem: str)`, rendered under `## Epic documents`. Report-only: the exit status is unchanged.

- [ ] **Step 1: Write the failing tests:**
  - `test_epic_documents_missing_section`: an open epic and a closed epic without a section → one finding each.
  - `test_epic_documents_malformed`: `parse` raises → finding.
  - `test_epic_documents_closed_link_404`: closed epic, `file_exists` false → finding naming the path.
  - `test_epic_documents_closed_pending`: closed epic with a `pending` entry → finding.
  - `test_epic_documents_open_link_not_checked`: open epic → `file_exists` is not called.
  - `test_epic_documents_skips`: standing and `[archived]` epics yield nothing.
  - `test_closed_epic_open_child_unchanged`: the existing behaviour holds after the refactor (the existing tests pass unchanged).
  - `test_render_epic_documents_section`.
- [ ] **Step 2:** Run `vrg-container-run -- vrg-validate`. Expected: the new tests fail.
- [ ] **Step 3:** Implement the check and the refactor.
- [ ] **Step 4:** Update the two site pages: the section's format and notes, the entry rules with both cutoffs, when the skills write it, `vrg-epic-docs set` and `backfill`, and the new invariant.
- [ ] **Step 5:** Run `vrg-container-run -- vrg-validate`. Expected: PASS (tests, markdown lint, docs build).
- [ ] **Step 6:** `vrg-commit --type feat --scope epic-audit --message "check every epic's Documents section"`

### Task D1: Backfill vergil-project (deployment)

Operational: no PR. Run with `issue-deploy`.

**Precondition (human-attested):** a vergil-tooling release that contains Tasks 1–3 is cut and installed. The agent never cuts it.

**Procedure:** spec §8 steps 1–6 for `vergil-project`. Check the dry-run summary against the spec §8 table (77 epics as of 2026-11-02, allowing for epics created or closed since). The removed legacy lines must include those on #51, #500, #550, #558 and #819.

**Acceptance:** `vrg-epic-audit --repo vergil-project/.github` shows no Epic documents findings. Spot-check #40, #156, #171, #558 and #819 against spec §3, and record the results.

### Task 4: Skills write the section (vergil-claude-plugin)

Relink vergil-project/vergil-claude-plugin#837 under the epic as this task (`vrg-epic-link`).

**Files:**

- Modify: `skills/epic-create/SKILL.md` (step 2, step 8), `skills/epic-retrospective/SKILL.md` (close-out step 3), `skills/migrate-repo/SKILL.md` (wherever it creates an epic)

- [ ] **Step 1:** In `epic-create` step 2, after `vrg-epic-create`, add: run `vrg-epic-docs set --epic <owner>/<repo>#<N>` to seed the Documents section. The epic body written by hand carries no document lines.
- [ ] **Step 2:** In `epic-create` step 8, at the `report-ready` handoff, add: run `vrg-epic-docs set --epic <ref> --spec epics/<N>-<slug>/spec.md --plan epics/<N>-<slug>/plan.md`. The links resolve once the PR merges, and that window is accepted.
- [ ] **Step 3:** In `epic-retrospective` close-out, at the handoff, add: run `vrg-epic-docs set --epic <ref> --retrospective epics/<N>-<slug>/retrospective.md`, taking `<slug>` from the epic's existing `epics/<N>-*/` directory.
- [ ] **Step 4:** In `migrate-repo`, add the same `set` call after each epic it creates.
- [ ] **Step 5:** `vrg-container-run -- vrg-validate`. Expected: PASS.
- [ ] **Step 6:** `vrg-commit --type feat --scope skills --message "epic-create and epic-retrospective keep the epic Documents section current"`

### Tasks D2–D6: Backfill each other org (deployment)

One task per org: logical-minds-foundry, mq-rest-admin-project, mnemosys-project, diogenes-project, vergils-nemesis. Each is operational, run with `issue-deploy`, and blocked by D1.

**Procedure:** spec §8 steps 1–6 for that org, including the private-repo check (step 2).

**Acceptance:** `vrg-epic-audit --repo <org>/.github` shows no Epic documents findings. The record lists the dry-run summary and every decision settled with `set`.

### Task D7: Confirm no epics under `wphillipmoore` (deployment)

Operational, blocked by D1. List the account's repos (public and private) and confirm that none carries an `epic`-labelled issue. Record the repo list and the result. If any does, stop and report: the account has no epic home, so the human decides.
