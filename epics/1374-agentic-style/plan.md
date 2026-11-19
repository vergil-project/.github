# agentic-style — implementation plan

- **Epic:** vergil-project/.github#1374
- **Planning issue:** vergil-project/.github#1373 (review history is in its comments)
- **Status:** draft, in design review

> **For agentic workers:** each task is one issue and one PR in the repo named
> in its heading, worked with `issue-implement`. Operational tasks
> (deployment, validation) are worked with `issue-deploy` or
> `issue-validate`. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** A cited, human-approved rulebook for Python, Rust, TypeScript, shell
and a common set, and a `/vergil:agentic-style` skill that audits every file
of code a repo owns against it and writes an advisory report, tuned on the six
fixtures.

**Architecture:**

- **.github** holds the research survey and, during tuning, the screened
  tuning reports, beside the spec.
- **vergil-tooling** does every deterministic part: the strict `[standards]`
  table and `vrg-standards-level`; `vrg-style-inventory`, which lists the code
  a repo owns, the exempt files and each generated file's generator; and
  `vrg-style-result`, which parses the rulebook and checks and cuts results in
  the spec §5.6 frame. Deterministic beats stochastic (spec §2.2), so nothing
  that can be computed is left to an agent.
- **vergil-claude-plugin** holds the rulebook, the reviewer and verifier
  agents, and the skill that runs them and assembles the report.
- **The fixtures** opt in with `[standards] level = "audit"`.

**Tech Stack:** Python 3 and pytest (vergil-tooling), Markdown skills and
agent definitions (vergil-claude-plugin), Markdown documents (.github).

**Spec:** [`spec.md`](spec.md) in this directory. Executors read both.

## Global Constraints

- Every rule cites a checkable source; a book citation not checked against
  the text is marked `unverified` (spec §3, §7).
- No copyrighted book text is quoted at length, in the rulebook, the survey or
  any prompt (spec §2.3, §3).
- The linter always wins, language by language: a rule a linter vergil runs or
  ships configuration for can check in a language is marked "enforced by
  `<check>`" there and is not agent-judged (spec §7).
- `[standards].level` is exactly one of `off`, `audit`, `enforced`; anything
  else, and any unknown key in the table, is a `ConfigError` (spec §3, §5.4).
- Every file of code the repo owns is audited, tests included, however it was
  created; only vendored code, third-party tool output and lockfiles are
  exempt, each listed; YAML, TOML and JSON are not code (spec §5.3).
- A result that fails a structure check is refused, never repaired (spec
  §5.3).
- Security-section details never reach a tracked file: the written result and
  the report carry the security count alone (spec §5.3).
- No skill or agent work (Tasks 6 to 10) starts before the rulebook approval
  gate, Task 5, closes (spec §5.2).
- This epic measures false positives only (spec §2.2).

## Review Focus

1. **`[standards]` written as `level = "Audit"`, `level = ""`, or a table with
   no `level`.** Each is a `ConfigError` naming the table and the allowed
   values; matching is case-sensitive. Pinned in Task 3
   (`test_standards_level_case_sensitive`, `test_standards_table_without_level`).
2. **An extensionless script with a shebang** (`scripts/bin/validate-custom`,
   `#!/usr/bin/env bash`). The inventory classifies it as shell. Pinned in
   Task 6 (`test_shebang_classifies_shell`).
3. **A forged security marker inside a finding's free text** (a suggested
   change containing `=== paad security section ===`). The check fails closed
   and the result is refused. Pinned in Task 7
   (`test_marker_in_free_text_refused`).
4. **A finding whose heading severity differs from its rule's.** Refused, the
   message naming the rule and both severities. Pinned in Task 7
   (`test_severity_mismatch_refused`).
5. **A repo with no code at all** (a git repo holding only Markdown and
   YAML). The skill writes a report whose inventory says so and audits
   nothing, rather than failing or inventing work. Pinned in Task 6
   (`test_inventory_no_code_files`) and checked in Task 9's dry run.

---

## Delivery table

Each line gives the task, its repo and kind, what it delivers, and what it is
blocked by.

- **1** (.github, task #1375): spec and plan, this PR.
- **2a** (.github): survey of the common, cross-language sources, Python and
  shell.
- **2b** (.github): survey of Rust and TypeScript.
- **2c** (.github): survey of Go and C++.
- **3** (vergil-tooling): the strict `[standards]` table and
  `vrg-standards-level`.
- **4a** (vergil-claude-plugin): rulebook draft, `common`, `python` and
  `shell`. Blocked by 2a.
- **4b** (vergil-claude-plugin): rulebook draft, `rust` and `typescript`.
  Blocked by 2b and 4a.
- **5** (.github, validation): **the rulebook approval gate.** Blocked by 4a
  and 4b.
- **6** (vergil-tooling): `vrg-style-inventory`. Blocked by 5.
- **7** (vergil-tooling): `vrg-style-result` (rulebook parser, frame check,
  security cut). Blocked by 5.
- **8** (vergil-claude-plugin): the reviewer and verifier agents. Blocked by
  5 and D1.
- **D1** (.github, deployment): the vergil-tooling release carrying Tasks 3,
  6 and 7 attested and installed. Blocked by 3, 6 and 7.
- **9** (vergil-claude-plugin): the `agentic-style` skill and its docs page.
  Blocked by 8 and D1.
- **D2** (.github, deployment): the vergil-claude-plugin release carrying
  Tasks 4a, 4b, 8 and 9 attested and installed. Blocked by 9.
- **10a–10f** (one per fixture: python-app, python-lib, rust-app, rust-lib,
  ts-app, ts-lib): `[standards] level = "audit"` in its `vergil.toml`. Each
  blocked by D1.
- **11** (.github): tuning round 1, its screened reports committed. Blocked by
  D2 and 10a–10f.
- **12** (vergil-tooling): the Python, Rust, TypeScript and shell naming docs
  link to the rulebook. Blocked by 5.
- **13** (.github, validation): the tuning success check. Blocked by 11 and
  every tuning task filed after it.
- **Documentation review** (vergil-claude-plugin, bookend #1187). Blocked by
  3, 6 to 9 and 12.
- **Follow-on brainstorm** (.github, bookend #1390). Blocked by 13.
- **Retrospective** (.github, bookend #1376). Blocked by everything else.

Notes:

- **Tasks 2a–2c and 3 run in parallel.** Task 3 has no
  dependency on the rulebook: it is configuration, not skill or agent work.
- **Tasks 6, 7 and 8 wait for the gate** (spec §5.2). Tasks 6 and 7 then run
  in parallel; Task 8 also waits for D1, because it checks its templates with
  the released `vrg-style-result`.
- **No fixture runs are seeded.** This epic changes no language support,
  container image, shared CI workflow or release tooling, so `fixture-run`'s
  rule does not apply. Tasks 10a–10f are ordinary one-line config PRs in the
  fixtures.
- **Further tuning rounds and the fixes they find are filed during Task 11,
  each on the human's approval** (doctrine §1.7): a rulebook or prompt change
  is a plugin task, a fixture's own finding a fixture task, a generated file's
  finding a vergil-tooling task against its generator (spec §5.5), and a
  linter gap an intake issue (spec §7). Each later round is a .github task
  shaped like Task 11. Task 13 is blocked by all of them.
- **Filing.** Every task above is filed at epic-create's filing step, in its
  repo, linked under #1374, with its `Blocked-by:` refs. The security
  advisories tracking the planning issue are retargeted then to the tasks
  that implement their requirements: Tasks 7, 8 and 9.

---

### Task 2a: Survey — common sources, Python, shell (.github)

Spec §5.1.

**Files:**

- Create: `epics/1374-agentic-style/survey/README.md` (the survey's shape,
  shared by 2a–2c, and the cross-language sources)
- Create: `epics/1374-agentic-style/survey/python.md`
- Create: `epics/1374-agentic-style/survey/shell.md`

**Interfaces:**

- Produces: per language, one table of candidate sources (title, author, URL
  or book and edition, license, open or copyrighted) and one table of
  candidate rules with columns `Candidate`, `Source`, `Linter coverage`
  (`<tool> <check>`, `<tool> <check> (available, not enabled)` or `none`) and
  `Data or judgment`. The rule table is Task 4a's input.

- [ ] **Step 1:** For each source, record what it says only after reading it;
  mark anything recalled but not read `unverified`.
- [ ] **Step 2:** Fill `Linter coverage` from the linters vergil runs or ships
  configuration for (ruff, mypy or ty, shellcheck, shfmt), naming the exact
  check, and from vergil-tooling's committed linter configuration for
  whether it is enabled.
- [ ] **Step 3:** Cover Conway's naming rules as adapted in vergil-tooling
  `docs/site/docs/standards/development/{python,shell}/naming-conventions.md`,
  each with its linter coverage.
- [ ] **Step 4:** Run `vrg-container-run -- vrg-validate`. Expect PASS.
- [ ] **Step 5:** Commit `vrg-commit --type docs --scope epic-1374 --message
  "survey common, Python and shell coding standards"`, then report-ready.

### Task 2b: Survey — Rust, TypeScript (.github)

As Task 2a, for `survey/rust.md` and `survey/typescript.md`, with clippy,
rustfmt, ESLint and `tsc` strict as the linters, and the Rust and TypeScript
naming-conventions docs.

### Task 2c: Survey — Go, C++ (.github)

As Task 2a, for `survey/go.md` and `survey/cpp.md`, with the linters vergil
runs for each (golangci-lint, clang-tidy). Research only: no rulebook follows
from it in this epic.

### Task 3: The strict `[standards]` table and `vrg-standards-level` (vergil-tooling)

Spec §3, §5.4.

**Files:**

- Modify: `src/vergil_tooling/lib/config.py` (`_KNOWN_SECTIONS`,
  `_KNOWN_KEYS`, a `StandardsConfig` dataclass, `_parse_standards`, a
  `standards` field on `VergilConfig`, wired in `_parse_raw_config`)
- Create: `src/vergil_tooling/bin/vrg_standards_level.py`
- Modify: `pyproject.toml` (the `vrg-standards-level` entry point)
- Modify: `docs/site/docs/guides/consuming-repo-setup.md` (a `[standards]`
  section)
- Test: `tests/vergil_tooling/test_config.py`,
  `tests/vergil_tooling/test_vrg_standards_level.py`

**Interfaces:**

- Produces: `STANDARDS_LEVELS = ("off", "audit", "enforced")`;
  `StandardsConfig(level: str = "off")`;
  `_parse_standards(raw: dict[str, Any], source: str = CONFIG_FILE) ->
  StandardsConfig`; `vrg-standards-level [--repo-root PATH]`, which prints
  the level and exits 0, or prints the `ConfigError` to stderr and exits 1.

- [ ] **Step 1: Write the failing tests** in `test_config.py`:
  `test_standards_absent_is_off` (`StandardsConfig().level == "off"`);
  `test_standards_levels_accepted` (each of the three values round-trips);
  `test_standards_unknown_level` (`level = "enforce"` raises `ConfigError`
  whose message contains `[standards].level` and `off, audit, enforced`);
  `test_standards_level_case_sensitive` (`"Audit"` raises);
  `test_standards_table_without_level` (`[standards]` alone raises);
  `test_standards_unknown_key` (`[standards] levle = "audit"` raises, not
  warns); `test_standards_not_a_table` (`standards = "audit"` raises). In
  `test_vrg_standards_level.py`: `test_prints_level` and
  `test_config_error_exits_1`.
- [ ] **Step 2:** Run `vrg-container-run -- pytest
  tests/vergil_tooling/test_config.py -k standards`. Expect FAIL.
- [ ] **Step 3: Implement** `_parse_standards` in the style of
  `_parse_merge_review`; it rejects unknown keys in its own table itself,
  since `_warn_unrecognized_keys` only warns. `vrg_standards_level.main`
  follows `vrg_container_build_command.py`.
- [ ] **Step 4:** Run the tests again. Expect PASS. Run
  `vrg-container-run -- vrg-validate`. Expect PASS.
- [ ] **Step 5:** Commit `vrg-commit --type feat --scope config --message
  "add the strictly validated [standards] table and vrg-standards-level"`,
  then report-ready.

### Task 4a: Rulebook draft — common, Python, shell (vergil-claude-plugin)

Spec §5.2. Blocked by 2a.

**Files:**

- Create: `skills/agentic-style/rulebook/README.md` (the format below and the
  severity definitions of spec §5.2)
- Create: `skills/agentic-style/rulebook/common.md`, `python.md`, `shell.md`

**Interfaces:**

- Produces: the rulebook format Task 7 parses. Each rule is a level-3
  heading `` ### `<id>` — <one-line statement> `` followed by exactly these
  list lines, in order: `- **Theme:** <naming | functions-and-control-flow |
  language-idioms | comments-and-documentation>`, `- **Severity:** <high |
  medium | low>`, `- **Violation:** <prose>`, `- **Source:** <citation>[
  (unverified)]`, `- **Linter:** <none | enforced by <tool> <check>>`,
  `- **Why no linter:** <prose, or n/a when a linter enforces it>`,
  `- **Adjacent paad lens:** <none | agentic-review | agentic-architecture>`.
  IDs are `<theme-prefix>.<kebab-name>`, unique across all files; the
  `meta.` prefix is never used. A rule whose `Linter:` is `enforced by …` is
  listed but not agent-judged.

- [ ] **Step 1:** Draft about 15 agent-judged rules for `python` and for
  `shell` and about 10 for `common` from the Task 2a tables, in our own words,
  listing every Conway naming rule whatever its linter coverage.
- [ ] **Step 2:** For each "available, not enabled" or closable gap in the
  survey, list it in the PR notes for the human's approval to file as an
  intake issue (doctrine §1.7). File only what is approved.
- [ ] **Step 3:** Run `vrg-container-run -- vrg-validate`. Expect PASS.
- [ ] **Step 4:** Commit `vrg-commit --type feat --scope agentic-style
  --message "draft the common, Python and shell rulebooks"`, then
  report-ready. The human's cut happens in this PR's review.

### Task 4b: Rulebook draft — Rust, TypeScript (vergil-claude-plugin)

As Task 4a, for `rust.md` and `typescript.md` from the Task 2b tables, in the
format Task 4a's README fixes. Blocked by 2b and 4a.

### Task 5: The rulebook approval gate (validation, .github)

Spec §5.2. Blocked by 4a and 4b.

- **Precondition (human-attested):** the human has cut the rulebook, verified
  every surviving book citation or left it marked `unverified`, and approves
  it, in a comment naming the plugin commit that holds the approved rulebook.
- **Procedure:** confirm every rule in that commit's
  `skills/agentic-style/rulebook/*.md` has the seven fields in order, a unique
  ID and no `meta.` prefix.
- **SUCCESS:** recorded with the commit SHA and the rule counts per file.

### Task 6: `vrg-style-inventory` (vergil-tooling)

Spec §5.3 (Inventory, ownership). Blocked by 5.

**Files:**

- Create: `src/vergil_tooling/lib/style_inventory.py`
- Create: `src/vergil_tooling/bin/vrg_style_inventory.py`
- Modify: `pyproject.toml`
- Test: `tests/vergil_tooling/test_style_inventory.py`

**Interfaces:**

- Produces: `build_inventory(repo_root: Path, languages: frozenset[str] |
  None = None) -> Inventory`, where `Inventory` holds `files:
  list[InventoryFile]` and `InventoryFile(path: str, language: str, lines:
  int, status: str, generated_from: str | None)`; `status` is `audited` or
  `exempt`. `vrg-style-inventory [--repo-root PATH] [--language LANG]...`
  prints the inventory as JSON with those field names; each `--language`
  limits it to that language.
- **What counts as code.** A file is code only when its extension or its
  shebang interpreter maps to a language, and every other tracked file is
  left out entirely (Markdown, LICENSE, VERSION, `.gitignore`, YAML, TOML,
  JSON):
  - `python`: `.py`, or a `python`/`python3` shebang;
  - `rust`: `.rs`;
  - `typescript`: `.ts`, `.tsx`, `.mts`, `.cts`;
  - `javascript`: `.js`, `.jsx`, `.mjs`, `.cjs`, or a `node` shebang. Its
    own language, not folded into TypeScript; it has no rulebook, so it is
    audited against `common` and reported as a coverage gap;
  - `shell`: `.sh`, `.bash`, or a `sh`/`bash` shebang;
  - `go`: `.go`;
  - `cpp`: `.cc`, `.cpp`, `.cxx`, `.h`, `.hh`, `.hpp`.
- Files come from `vrg-git ls-files` (tracked only; symlinks skipped).
  `generated_from` comes from vergil-tooling's own managed-file definitions
  in `lib/repo_config.py` (for example `.claude/hooks/guard.sh` →
  `vergil-tooling:src/vergil_tooling/data/hook_guard_shim.sh`). Exempt:
  lockfiles, paths under a `vendor/` or `third_party/` directory, and files
  carrying a third-party generated marker (`@generated`, `DO NOT EDIT`)
  that vergil does not manage.

- [ ] **Step 1: Write the failing tests** (each over a temporary git repo):
  `test_extension_classification`; `test_shebang_classifies_shell`;
  `test_tests_are_audited`; `test_managed_shim_names_generator`;
  `test_lockfile_exempt`; `test_vendor_dir_exempt`;
  `test_config_formats_left_out` (`ci.yml`, `vergil.toml`, `package.json`
  absent from `files`); `test_markdown_left_out` (`README.md`, `LICENSE`,
  `VERSION` absent); `test_mjs_classified` (`eslint.vergil.config.mjs` is
  `javascript`, `audited`, with its generator from
  `LANGUAGE_MANAGED_FILES["typescript"]`); `test_language_filter` (only the
  named languages listed); `test_untracked_and_symlink_skipped`;
  `test_inventory_no_code_files` (a repo of only Markdown and YAML: empty
  `files`, exit 0).
- [ ] **Step 2:** Run them. Expect FAIL.
- [ ] **Step 3: Implement** `build_inventory` and the CLI.
- [ ] **Step 4:** Run them. Expect PASS. Run `vrg-container-run --
  vrg-validate`. Expect PASS.
- [ ] **Step 5:** Commit `vrg-commit --type feat --scope style --message
  "add vrg-style-inventory"`, then report-ready.

### Task 7: `vrg-style-result` (vergil-tooling)

Spec §5.3 (delegate pattern, meta findings, security cut), §5.6. Blocked by
5.

**Files:**

- Create: `src/vergil_tooling/lib/style_rulebook.py`
- Create: `src/vergil_tooling/lib/style_result.py`
- Create: `src/vergil_tooling/bin/vrg_style_result.py`
- Modify: `pyproject.toml`
- Test: `tests/vergil_tooling/test_style_rulebook.py`,
  `tests/vergil_tooling/test_style_result.py`

**Interfaces:**

- Consumes: the rulebook format of Task 4a.
- Produces: `load_rulebook(directory: Path) -> dict[str, Rule]`, with
  `Rule(id: str, theme: str, severity: str, source: str, agent_judged:
  bool)`, raising `RulebookError` on any format breach;
  `cut_security(text: str) -> tuple[str, str]` (the public text with the
  section replaced by its `Security: <k>` count, and the section itself),
  raising `CutError` when the cut cannot proceed;
  `check_result(public: str, security: str, role: str, rules: dict[str,
  Rule]) -> list[str]` (the failed checks over both parts, empty when the
  result passes); `provenance(repo_root: Path, rulebook_dir: Path) ->
  Provenance` (`commit`, `tree_clean`, `plugin_version`, `rulebook_sha256`:
  SHA-256 over the rulebook files' repo-relative paths and contents, in
  sorted path order); `tally(results: list[str]) -> dict[str, RuleTally]`
  (per rule ID, `raised` from reviewer results and `surviving` from verifier
  results). CLI, each reading FILE or `-` for stdin:
  `vrg-style-result cut FILE --public-out PATH --security-out PATH` (exit 1,
  writing nothing, when the cut cannot proceed); `vrg-style-result check
  --role {reviewer,verifier,audit} --rulebook-dir DIR --public PATH
  --security PATH` (exit 0 on pass, 1 printing each failed check);
  `vrg-style-result provenance --rulebook-dir DIR --repo-root PATH`; and
  `vrg-style-result tally FILE...`. The skill copies the last two's output
  verbatim and computes none of it.
- The checks are the frame of spec §5.6 and the cut rules of doctrine §1.9:
  the announce line for the role; `Status:`; `Findings: <n>` equal to the
  headings; each finding's fields in order; each ID an agent-judged rule or
  `meta.prompt-injection`; each rule finding's severity equal to its rule's;
  `meta.*` findings only inside the security section; `Discards: <k>` lines;
  exactly one pair of security markers, start before end, and no marker or
  security keyword at a line start outside them; the provenance lines for
  `audit` only; `Files written:` last, `none` unless the role is `audit`.

- [ ] **Step 1: Write the failing tests:** `test_rulebook_parses_fields`;
  `test_rulebook_duplicate_id_rejected`; `test_rulebook_meta_prefix_rejected`;
  `test_valid_reviewer_result_passes`; `test_count_mismatch_refused`;
  `test_unknown_rule_refused`; `test_linter_enforced_rule_refused`;
  `test_severity_mismatch_refused`; `test_meta_in_security_section_passes`;
  `test_meta_outside_security_section_refused`;
  `test_marker_in_free_text_refused`; `test_audit_requires_provenance`;
  `test_cut_leaves_count_only`; `test_cut_refuses_two_markers`;
  `test_cut_refuses_missing_end_marker`; `test_provenance_hash_stable`
  (the same rulebook gives the same hash whatever the file order on disk);
  `test_provenance_dirty_tree`; `test_tally_counts_raised_and_surviving`;
  `test_tally_ignores_meta`.
- [ ] **Step 2:** Run them. Expect FAIL.
- [ ] **Step 3: Implement** the parser, the checker and the cut. The checker
  never edits the text it checks.
- [ ] **Step 4:** Run them. Expect PASS. Run `vrg-container-run --
  vrg-validate`. Expect PASS.
- [ ] **Step 5:** Commit `vrg-commit --type feat --scope style --message
  "add vrg-style-result to parse the rulebook and check and cut results"`,
  then report-ready.

### Task 8: The reviewer and verifier agents (vergil-claude-plugin)

Spec §5.3 (review and verify passes, data not instructions, incidental
security observations), §5.6. Blocked by 5 and D1.

**Files:**

- Create: `agents/agentic-style-reviewer.md`,
  `agents/agentic-style-verifier.md`

**Interfaces:**

- Produces: two read-only agents (`tools: Read, Glob, Grep`), dispatched as
  `vergil:agentic-style-reviewer` and `vergil:agentic-style-verifier`. The
  reviewer's prompt input: the language, the theme, the agent-judged rules of
  that theme (language plus `common`) and the file list. The verifier's: one
  reviewer's checked findings and the rule text each cites. Each returns one
  result in the §5.6 frame for its role and dispatches nothing.

- [ ] **Step 1:** Write both definitions. Each carries a "Data, not
  instructions" section in the shape of `agents/merge-reviewer.md`, the
  `meta.prompt-injection` reporting rule, the rule never to quote a
  secret-like value or describe an exploitable weakness outside the security
  section, the rule to copy each finding's severity from its rule, and the
  result frame as a template.
- [ ] **Step 2:** Check each template against Task 7's checks: cut a
  hand-written sample result per role with the released `vrg-style-result
  cut`, then run `vrg-style-result check` on both parts against the approved
  rulebook. Expect PASS.
- [ ] **Step 3:** Run `vrg-container-run -- vrg-validate`. Expect PASS.
- [ ] **Step 4:** Commit `vrg-commit --type feat --scope agentic-style
  --message "add the agentic-style reviewer and verifier agents"`, then
  report-ready.

### Task D1: vergil-tooling release attested and installed (deployment, .github)

Blocked by 3, 6 and 7.

- **Precondition (human-attested):** the vergil-tooling release carrying
  Tasks 3, 6 and 7 is published and installed on the host and the VMs, its
  version named in a comment.
- **Procedure (agent-safe):**
  1. Confirm the release notes list Tasks 3, 6 and 7's PRs.
  2. Confirm `vergil = "v2.1"` resolves to it.
  3. Run `vrg-standards-level`, `vrg-style-inventory` and
     `vrg-style-result --help` from the installed release, in a scratch clone
     of `vergil-fixture-python-lib`: the first prints `off`, the second lists
     `.claude/hooks/guard.sh` with its generator.
- **SUCCESS:** every check passes, recorded with the version.

### Task 9: The `agentic-style` skill (vergil-claude-plugin)

Spec §5.3, §5.5, §5.6, §7. Blocked by 8 and D1.

**Files:**

- Create: `skills/agentic-style/SKILL.md`
- Create: `docs/site/docs/skills/agentic-style.md`; modify
  `docs/site/docs/skills/index.md` to list it

**Interfaces:**

- Consumes: `vrg-standards-level`, `vrg-style-inventory`, `vrg-style-result`
  (Tasks 3, 6, 7); the agents of Task 8; the rulebook.
- Produces: `/vergil:agentic-style [--output DIR] [--language LANG]...`.
  `DIR` defaults to the audited repo's `build/agentic-style/`; the skill
  resolves it to an absolute path and prints it before writing. Each
  `--language` is passed to `vrg-style-inventory`. It writes
  `<date>-<repo>.md` (the report) and `<date>-<repo>.result.md` (its own
  result in the `audit` frame) to `DIR`, and raw material only to the
  audited repo's ignored `build/agentic-style/raw/`, whatever `DIR` is. It
  makes no issue, commit or handoff.

- [ ] **Step 1: Write the skill.** In order:
  1. Run `vrg-standards-level`; stop with its reason unless `audit` or
     `enforced`.
  2. Run `vrg-style-inventory`, passing each `--language`.
  3. For each language with audited files and each theme with agent-judged
     rules, dispatch one reviewer, given that language's rules plus
     `common`, or `common` alone for a language with no rulebook (a
     coverage gap the report names, as for `javascript`).
  4. Write each raw result only under `build/agentic-style/raw/`. **Cut
     first:** `vrg-style-result cut`, keeping the security part there and
     showing it in the terminal. **Then check** both parts with
     `vrg-style-result check`. A result that cannot be cut, or that fails a
     check, is kept whole in `raw/` as security material; the report
     records only that it was refused and which check failed, never its
     text.
  5. Dispatch the verifier per checked reviewer result, and cut and check
     its result the same way.
  6. Run `vrg-style-result provenance` and `vrg-style-result tally` and copy
     their output verbatim.
  7. Build the report (inventory, exempt files, coverage gaps, findings with
     every field of spec §5.3, the per-rule table from the tally, the
     provenance, refused results, the security count) and the `audit`
     result; cut and check the `audit` result before writing it.
- [ ] **Step 2: Dry runs,** each with a local `level = "audit"` edit that is
  not committed: a scratch git repo of only Markdown and YAML (the report's
  inventory says there is no code, and nothing is audited); this `.github`
  repo (exactly one audited file, `.claude/hooks/guard.sh`, with its
  generator named); and `vergil-fixture-python-lib` (its `audit` result
  passes `vrg-style-result check`).
- [ ] **Step 3:** Run `vrg-container-run -- vrg-validate`. Expect PASS.
- [ ] **Step 4:** Commit `vrg-commit --type feat --scope agentic-style
  --message "add the agentic-style skill"`, then report-ready.

### Task D2: vergil-claude-plugin release attested and installed (deployment, .github)

Blocked by 9.

- **Precondition (human-attested):** the plugin release carrying Tasks 4a,
  4b, 8 and 9 is published and installed, its version named in a comment.
- **Procedure (agent-safe):** confirm the installed plugin lists
  `/vergil:agentic-style` and both agents, and that its rulebook matches the
  commit Task 5 approved.
- **SUCCESS:** recorded with the plugin version.

### Tasks 10a–10f: Fixture opt-in (one per fixture)

Blocked by D1. In each fixture: add `[standards]` with `level = "audit"` to
`vergil.toml`; run `vrg-standards-level` (expect `audit`) and
`vrg-container-run -- vrg-validate` (expect PASS); commit `vrg-commit --type
chore --scope config --message "opt in to the agentic-style audit"`, then
report-ready.

### Task 11: Tuning round 1 (.github)

Spec §5.5, §7. Blocked by D2 and 10a–10f.

**Files:**

- Create: `epics/1374-agentic-style/tuning/round-1/<date>-<repo>.md` and
  `<date>-<repo>.result.md` for each of the six fixtures and for
  vergil-tooling (shell only)
- Create: `epics/1374-agentic-style/tuning/round-1/README.md` (the round's
  summary: per-rule counts across repos, generated-file findings merged
  across fixtures, and the follow-up tasks the human approved)

- [ ] **Step 1:** In each fixture's clone, run `/vergil:agentic-style
  --output <absolute path of this .github worktree>/epics/1374-agentic-style/tuning/round-1/`,
  and confirm the path it prints is inside this worktree. In vergil-tooling's
  clone, with a local `level = "audit"` edit that is not committed, run it
  the same way with `--language shell`.
- [ ] **Step 2:** Screen every report and result with the doctrine's
  screening (§1.6) before staging; a match stops the round for the human.
- [ ] **Step 3:** Walk the human through the findings; file the follow-ups
  they approve (see the delivery-table notes).
- [ ] **Step 4:** Run `vrg-container-run -- vrg-validate`. Expect PASS.
- [ ] **Step 5:** Commit `vrg-commit --type docs --scope epic-1374 --message
  "record agentic-style tuning round 1"`, then report-ready.

### Task 12: Naming docs link to the rulebook (vergil-tooling)

Spec §7 (Naming docs). Blocked by 5.

**Files:**

- Modify: `docs/site/docs/standards/development/{python,rust,typescript,shell}/naming-conventions.md`

- [ ] **Step 1:** Replace each doc's variable naming rules with a short
  statement that the agentic-style rulebook is the authority and a link to
  its rules in the plugin's `skills/agentic-style/rulebook/<lang>.md`,
  keeping the language's own casing baseline. Leave the Go and C++ docs
  unchanged.
- [ ] **Step 2:** Run `vrg-container-run -- vrg-validate`. Expect PASS.
- [ ] **Step 3:** Commit `vrg-commit --type docs --scope standards
  --message "point the naming conventions at the agentic-style rulebook"`,
  then report-ready.

### Task 13: Tuning success check (validation, .github)

Spec §5.5. Blocked by 11 and every tuning task filed after it.

- **Precondition (human-attested):** the human judges each fixture's latest
  report mostly worth acting on.
- **Procedure:** from the latest round's README, record per fixture whether
  its report is clean, and per rule whether it fired, survived verification,
  or never fired (not yet judged, spec §7).
- **SUCCESS:** recorded with the round number and the clean fixtures, which
  are candidates for `enforced` once a merge check exists.
