# agentic-style — implementation plan

- **Epic:** vergil-project/.github#1374
- **Planning issue:** vergil-project/.github#1373 (review history is in its comments)
- **Status:** approved

> **For agentic workers:** each task is one issue and one PR in the repo named
> in its heading, worked with `issue-implement`. Operational tasks
> (deployment, validation) are worked with `issue-deploy` or
> `issue-validate`. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** A cited, human-approved rulebook for Python, Rust, TypeScript, shell
and a common set, and a `/vergil:agentic-style` skill that audits every file
of code a repo owns against it and writes an advisory report, tuned on the six
fixtures and the plugin's shell scripts.

**Architecture:**

- **.github** holds the research survey and, during tuning, the screened
  tuning reports, beside the spec.
- **vergil-tooling** does every deterministic part: the strict `[standards]`
  table and `vrg-standards-level`; `vrg-style-inventory`, which lists the code
  a repo owns, the exempt files and each generated file's generator; and
  `vrg-style-result`, which parses the rulebook, cuts and checks results in
  the spec §5.6 frame, computes provenance and tallies, and renders the
  report. Deterministic beats stochastic (spec §2.2), so nothing that can be
  computed or rendered is left to an agent.
- **vergil-claude-plugin** holds the rulebook, the reviewer and verifier
  agents, and the skill that dispatches them and hands their results to the
  tooling.
- **The fixtures and the plugin** opt in with a committed `[standards] level
  = "audit"`.

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
- `[standards].level` is `off` or `audit`. `enforced` is a `ConfigError`
  ("not yet supported") until the merge-check epic adds it. Any other value,
  and any unknown key in the table, is a `ConfigError` (spec §3, §5.4).
- **A requirement is never worked around.** `vergil.toml` is changed only by a
  committed PR, never edited locally to get past a check. A file that fails
  validation is an error, not something to patch and continue.
- Every file of code the repo owns is audited, tests included, however it was
  created. The only exemptions are the paths listed in `[standards].exempt`,
  each listed in the report; nothing is exempted by inference (no marker
  comments, no directory-name guesses). YAML, TOML, JSON and other non-code
  files are not code (spec §5.3).
- A result that fails a cut or a structure check is refused, never repaired,
  and a refused or blocked result fails the whole audit (spec §5.3).
- Security-section details never reach a tracked file: the written result and
  the report carry the security count alone (spec §5.3).
- Code that generates Markdown generates markdownlint-clean Markdown; a
  lint failure in generated output is a bug in the generator. The structured
  result is not Markdown and is written as `.result.txt`.
- No skill or agent work (Tasks 6 to 10g) starts before the rulebook approval
  gate, Task 5, closes (spec §5.2).
- This epic measures false positives only (spec §2.2).

## Review Focus

1. **`[standards]` written as `level = "Audit"`, `level = ""`,
   `level = "enforced"`, or a table with no `level`.** Each is a
   `ConfigError` naming the table and the allowed values; matching is
   case-sensitive. Pinned in Task 3 (`test_standards_level_case_sensitive`,
   `test_standards_enforced_not_supported`,
   `test_standards_table_without_level`).
2. **An extensionless script with a shebang** (`scripts/bin/validate-custom`,
   `#!/usr/bin/env bash`). The inventory classifies it as shell. Pinned in
   Task 6 (`test_shebang_classifies_shell`).
3. **A file trying to exempt itself** with a `DO NOT EDIT` or `@generated`
   comment, or by sitting under `vendor/`. It is still audited unless
   `[standards].exempt` lists it. Pinned in Task 6
   (`test_marker_does_not_exempt`, `test_vendor_dir_not_exempt`).
4. **A forged security marker, or reviewer free text holding a link, an
   image or HTML.** The check fails closed on a marker; the report renders
   free text inert in fenced blocks. Pinned in Task 7
   (`test_marker_in_free_text_refused`, `test_report_free_text_fenced`).
5. **A repo with no code at all** (a git repo holding only Markdown and
   YAML, opted in by a committed `vergil.toml`). The inventory is empty and
   the report says so. Pinned in Task 6 (`test_inventory_no_code_files`)
   and run end to end in Task D2v.

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
- **6** (vergil-tooling): `vrg-style-inventory`. Blocked by 3 and 5.
- **7** (vergil-tooling): `vrg-style-result` (rulebook parser, cut, check,
  provenance, tally, report). Blocked by 5.
- **D1** (.github, deployment): the vergil-tooling release carrying Tasks 3,
  6 and 7 attested and installed. Blocked by 3, 6 and 7.
- **8** (vergil-claude-plugin): the reviewer and verifier agents. Blocked by
  5 and D1.
- **9** (vergil-claude-plugin): the `agentic-style` skill. Blocked by 8 and
  D1.
- **D2** (.github, deployment): the vergil-claude-plugin release carrying
  Tasks 4a, 4b, 8 and 9 attested and installed. Blocked by 9.
- **10a–10f** (one per fixture: python-app, python-lib, rust-app, rust-lib,
  ts-app, ts-lib): `[standards] level = "audit"` committed. Each blocked by
  D1.
- **10g** (vergil-claude-plugin): `[standards] level = "audit"` committed,
  for the shell tuning run. Blocked by D1.
- **D2v** (.github, validation): the skill's end-to-end dry runs against the
  installed release. Blocked by D2 and 10b.
- **11** (.github): tuning round 1, its screened reports committed. Blocked by
  D2v, 10a–10f and 10g.
- **12** (vergil-tooling): the Python, Rust, TypeScript and shell naming docs
  link to the rulebook. Blocked by 5.
- **13** (.github, validation): the tuning success check. Blocked by 11 and
  every tuning task filed after it.
- **Documentation review** (vergil-claude-plugin, bookend #1187). Blocked by
  3, 6 to 9 and 12.
- **Follow-on brainstorm** (.github, bookend #1390). Blocked by 13.
- **Retrospective** (.github, bookend #1376). Blocked by everything else.

Notes:

- **Tasks 2a–2c and 3 run in parallel.** Task 3 has no dependency on the
  rulebook: it is configuration, not skill or agent work.
- **Tasks 6 and 7 stay behind the gate** although they are deterministic
  tooling: the human's cut of the rulebook may change its format or themes,
  which Task 7 parses and the inventory's reviewers are grouped by. They run
  in parallel once it closes. Task 8 also waits for D1, because it checks its
  templates with the released `vrg-style-result`.
- **No run ever edits `vergil.toml` locally.** Every audited repo is opted in
  by a committed PR (Tasks 10a–10g); the no-code case in D2v uses a throwaway
  repo created with its own committed `vergil.toml`.
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
  or book and edition, license, open or copyrighted, `Covers`: what the
  source addresses, such as naming, structure, errors, API design or docs)
  and one table of
  candidate rules with columns `Candidate`, `Source`, `Linter coverage`
  (`<tool> <check>`, `<tool> <check> (available, not enabled)` or `none`) and
  `Data or judgment`. The rule table is Task 4a's input.

- [ ] **Step 1:** For each source, record what it says only after reading it;
  mark anything recalled but not read `unverified`.
- [ ] **Step 2:** Fill `Linter coverage` from the linters vergil runs or ships
  configuration for (ruff, mypy or ty, shellcheck), naming the exact check,
  and from vergil-tooling's committed linter configuration for whether it is
  enabled. Linter coverage is decided per rule and per language (spec §7),
  never per file. Record shellcheck's actual file scope (vergil-tooling
  `src/vergil_tooling/bin/validate_common.py`, `_find_shell_files`: `*.sh`
  anywhere, plus files under `scripts/bin/` and `scripts/git-hooks/`) and
  list the shell files outside it as a linter coverage gap, for the human's
  approval as an intake issue to widen that scope.
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
from it in this epic. Each "available, not enabled" check and closable linter
gap it records is listed in the PR notes for the human's approval as an
intake issue (doctrine §1.7); file only what is approved.

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

- Produces: `STANDARDS_LEVELS = ("off", "audit")`;
  `StandardsConfig(level: str = "off", exempt: tuple[str, ...] = ())`;
  `_parse_standards(raw: dict[str, Any], source: str = CONFIG_FILE) ->
  StandardsConfig`. Keys: `level` (required when the table exists) and
  `exempt` (optional list of non-empty, repo-relative paths; an entry ending
  in `/` covers everything below it, as `[merge-review].docs-paths` does).
  `vrg-standards-level [--repo-root PATH]` prints the level and exits 0, or
  prints the `ConfigError` to stderr and exits 1.

- [ ] **Step 1: Write the failing tests** in `test_config.py`:
  `test_standards_absent_is_off`; `test_standards_levels_accepted` (`off`
  and `audit` round-trip); `test_standards_enforced_not_supported`
  (`"enforced"` raises `ConfigError` whose message says it is not yet
  supported); `test_standards_unknown_level` (`level = "enforce"` raises, the
  message containing `[standards].level` and `off, audit`);
  `test_standards_level_case_sensitive` (`"Audit"` raises);
  `test_standards_table_without_level`; `test_standards_unknown_key`
  (`levle = "audit"` raises, not warns); `test_standards_not_a_table`;
  `test_standards_exempt_paths` (a list round-trips);
  `test_standards_exempt_rejects_absolute_and_empty`. In
  `test_vrg_standards_level.py`: `test_prints_level` and
  `test_config_error_exits_1`.
- [ ] **Step 2:** Run `vrg-container-run -- pytest
  tests/vergil_tooling/test_config.py -k standards`. Expect FAIL.
- [ ] **Step 3: Implement** `_parse_standards` in the style of
  `_parse_merge_review`; it rejects unknown keys in its own table itself,
  since `_warn_unrecognized_keys` only warns. `vrg_standards_level.main`
  follows `vrg_container_build_command.py`.
- [ ] **Step 4:** Run the tests again. Expect PASS.
- [ ] **Step 5: Refactor:** share the repo-relative path validation between
  `exempt` and `[merge-review].docs-paths`. Run the tests and
  `vrg-container-run -- vrg-validate`. Expect PASS.
- [ ] **Step 6:** Commit `vrg-commit --type feat --scope config --message
  "add the strictly validated [standards] table and vrg-standards-level"`,
  then report-ready.

### Task 4a: Rulebook draft — common, Python, shell (vergil-claude-plugin)

Spec §5.2. Blocked by 2a.

**Files:**

- Create: `skills/agentic-style/rulebook/README.md` (the format below and the
  severity definitions of spec §5.2; never parsed or hashed as rules)
- Create: `skills/agentic-style/rulebook/common.md`, `python.md`, `shell.md`

**Interfaces:**

- Produces: the rulebook format Task 7 parses. The rulebook is exactly the
  files `common.md`, `python.md`, `shell.md`, `rust.md` and `typescript.md`.
  Each rule is a level-3 heading `` ### `<id>` — <one-line statement> ``
  followed by exactly these list lines, in order: `- **Theme:** <naming |
  flow | idiom | docs>`, `- **Severity:** <high | medium | low>`,
  `- **Violation:** <prose>`, `- **Source:** <citation>[ (unverified)]`,
  `- **Linter:** <none | enforced by <tool> <check>>`, `- **Why no linter:**
  <prose, or n/a when a linter enforces it>`, `- **Adjacent paad lens:**
  <none | agentic-review | agentic-architecture>`. IDs are
  `<file>.<theme>.<kebab-name>`, `<file>` being the file's name without
  `.md` (so `python.naming.full-words` and `shell.naming.full-words` can both
  exist), unique across the rulebook. The themes are `naming` (naming),
  `flow` (functions and control flow), `idiom` (language idioms) and `docs`
  (comments and documentation). The `meta.` prefix is never used. A rule
  whose `Linter:` is `enforced by …` is listed but not agent-judged.

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
- **Procedure:** confirm that, in that commit, each of the five rulebook files
  exists and every rule in them has the seven fields in order, an ID of the
  form Task 4a fixes, unique across the rulebook, and no `meta.` prefix.
- **SUCCESS:** recorded with the commit SHA and the rule counts per file.

### Task 6: `vrg-style-inventory` (vergil-tooling)

Spec §5.3 (Inventory, ownership). Blocked by 3 and 5.

**Files:**

- Create: `src/vergil_tooling/lib/style_inventory.py`
- Create: `src/vergil_tooling/bin/vrg_style_inventory.py`
- Modify: `pyproject.toml`
- Test: `tests/vergil_tooling/test_style_inventory.py`

**Interfaces:**

- Consumes: `StandardsConfig.exempt` (Task 3).
- Produces: `build_inventory(repo_root: Path, languages: frozenset[str] |
  None = None) -> Inventory`, where `Inventory` holds `files:
  list[InventoryFile]` and `InventoryFile(path: str, language: str, lines:
  int, status: str, generated_from: str | None)`; `status` is `audited` or
  `exempt`. `vrg-style-inventory [--repo-root PATH] [--language LANG]...`
  prints the inventory as JSON with those field names; each `--language`
  limits it to that language. A `ConfigError` from reading `vergil.toml`
  exits 1.
- **What counts as code.** A file is code only when its extension or its
  shebang interpreter maps to a language, and every other tracked file is
  left out entirely (Markdown, LICENSE, VERSION, `.gitignore`, YAML, TOML,
  JSON, lockfiles):
  - `python`: `.py`, or a `python`/`python3` shebang;
  - `rust`: `.rs`;
  - `typescript`: `.ts`, `.tsx`, `.mts`, `.cts`;
  - `javascript`: `.js`, `.jsx`, `.mjs`, `.cjs`, or a `node` shebang. Its
    own language, not folded into TypeScript; it has no rulebook, so it is
    audited against `common` and reported as a coverage gap;
  - `shell`: `.sh`, `.bash`, or a `sh`/`bash` shebang;
  - `go`: `.go`;
  - `cpp`: `.cc`, `.cpp`, `.cxx`, `.h`, `.hh`, `.hpp`.
- Files come from `vrg-git ls-files` (tracked only; symlinks are not followed
  and not listed). `generated_from` comes from vergil-tooling's own
  managed-file definitions in `lib/repo_config.py` (for example
  `.claude/hooks/guard.sh` →
  `vergil-tooling:src/vergil_tooling/data/hook_guard_shim.sh`). A file is
  `exempt` only when `[standards].exempt` lists it; nothing else exempts a
  file.

- [ ] **Step 1: Write the failing tests** (each over a temporary git repo
  with a committed `vergil.toml`): `test_extension_classification`;
  `test_shebang_classifies_shell`; `test_tests_are_audited`;
  `test_managed_shim_names_generator`; `test_explicit_exempt_listed`;
  `test_exempt_directory_prefix` (`tools/` exempts `tools/a.py` but not
  `toolsx/b.py`);
  `test_marker_does_not_exempt` (a `DO NOT EDIT` file is audited);
  `test_vendor_dir_not_exempt` (a `vendor/x.py` is audited when not listed);
  `test_config_formats_left_out` (`ci.yml`, `vergil.toml`, `package.json`,
  `uv.lock` absent from `files`); `test_markdown_left_out` (`README.md`,
  `LICENSE`, `VERSION` absent); `test_mjs_classified`
  (`eslint.vergil.config.mjs` is `javascript`, `audited`, with its generator
  from `LANGUAGE_MANAGED_FILES["typescript"]`); `test_language_filter`;
  `test_untracked_and_symlink_skipped`; `test_inventory_no_code_files` (a
  repo of only Markdown and YAML: empty `files`, exit 0).
- [ ] **Step 2:** Run them. Expect FAIL.
- [ ] **Step 3: Implement** `build_inventory` and the CLI.
- [ ] **Step 4:** Run them. Expect PASS.
- [ ] **Step 5: Refactor:** reuse the existing tracked-file listing helpers
  in `validate_common.py` and the managed-file tables in `repo_config.py`
  rather than duplicating them. Run the tests and `vrg-container-run --
  vrg-validate`. Expect PASS.
- [ ] **Step 6:** Commit `vrg-commit --type feat --scope style --message
  "add vrg-style-inventory"`, then report-ready.

### Task 7: `vrg-style-result` (vergil-tooling)

Spec §5.3 (delegate pattern, meta findings, security cut, report), §5.6.
Blocked by 5.

**Files:**

- Create: `src/vergil_tooling/lib/style_rulebook.py`
- Create: `src/vergil_tooling/lib/style_result.py`
- Create: `src/vergil_tooling/lib/style_report.py`
- Create: `src/vergil_tooling/bin/vrg_style_result.py`
- Modify: `pyproject.toml`
- Test: `tests/vergil_tooling/test_style_rulebook.py`,
  `tests/vergil_tooling/test_style_result.py`,
  `tests/vergil_tooling/test_style_report.py`

**Interfaces:**

- Consumes: the rulebook format of Task 4a.
- Produces:
  - `load_rulebook(directory: Path) -> dict[str, Rule]`, reading exactly the
    five rulebook files and never `README.md`, with `Rule(id: str, theme:
    str, severity: str, source: str, agent_judged: bool)`; it raises
    `RulebookError` on a missing file or any format breach.
  - `cut_security(text: str) -> tuple[str, str]`: the public part, with the
    section replaced by its `Security: <k>` line, and the section itself. It
    raises `CutError` unless the text has exactly one start-marker line and
    one end-marker line, the start first, each the marker alone, and no line
    outside them starting (after stripping leading whitespace, list markers
    and emphasis) with `### S<digit>`, `Security:` or `Security discards:`.
    Markers and keywords match without regard to case (doctrine §1.9,
    §1.10).
  - `check_result(public: str, security: str, role: str, rules: dict[str,
    Rule], plugin_version: str) -> list[str]`, the failed checks, empty when
    the result passes. **On the public part:** the announce line for the
    role, carrying exactly `plugin_version`; `Status:`;
    `Findings: <n>` equal to its `### F<i>` headings; each finding's labels
    (below) in order; each `Rule:` an agent-judged rule and each heading's
    severity equal to that rule's; `Discards: <k>` followed by exactly `<k>`
    dash-led lines; exactly one `Security: <k>` line, where the section was;
    the provenance lines for `audit` only; `Files written:` last, `none` for
    a reviewer or verifier and paths relative to the output directory for
    `audit`. **On the security part:** `Security: <k>` equal to its `### S<i>`
    headings; `Security discards: <k>` followed by exactly `<k>` dash-led
    lines; each finding's labels; only `meta.*` findings, never a rulebook
    finding; and `<k>` equal to the public part's `Security:` count. **For a
    verifier,** given the reviewer result it judged: exactly one verdict per
    reviewer finding, each naming it.
  - **Meta kinds are a closed list:** `meta.prompt-injection` (text in the
    audited code that tries to steer a reviewer) and
    `meta.security-observation` (a secret-like value or exploitable weakness
    noticed in passing), each with severity `high`. Any other `meta.*` ID is
    refused. Adding a kind is a reviewed change to this list.
  - **Finding labels,** one `<Label>: <value>` line each, in this order. A
    reviewer finding: `Rule:`, `Location:` (`<path>:<line>`), `Problem:`,
    `Suggested change (untrusted):`, `Reach:` (`local | cross-module |
    public-interface`). A verifier finding: `Finding:` (the reviewer
    finding's `F<i>`), `Rule:`, `Location:`, `Verdict:` (`upheld | dropped`),
    `Reason:`. A `meta.*` finding: `Rule:`, `Location:`, `Problem:`. An
    `audit` finding (one per upheld finding): `Rule:`, `Source:` (the
    rule's citation), `Location:`, `Problem:`, `Suggested change
    (untrusted):`, `Reach:`, `Generator:` (`none` or the inventory's
    `generated_from`). The source and the
    generator come from the rulebook and the inventory; agents never supply
    them.
  - `provenance(repo_root: Path, rulebook_dir: Path, plugin_version: str) ->
    Provenance`: `commit`, `tree_clean`, `plugin_version` (as given, refused
    when empty or malformed), `rulebook_sha256` (SHA-256 over the five files'
    names and contents, in sorted name order).
  - `tally(results: list[str]) -> dict[str, RuleTally]`: per rule ID,
    `raised` from reviewer results and `surviving` from verifier results;
    `meta.*` excluded.
  - `build_audit_result(reviewer_results: list[str], verifier_results:
    list[str], inventory: Inventory, provenance: Provenance, rules:
    dict[str, Rule]) -> str`: the skill's `audit` result, built only from
    checked inputs. Its findings are the upheld findings; its security part
    is the security count alone; its `Files written:` lists the report and
    result relative to the output directory.
  - `render_report(audit_result: str, inventory: Inventory, tally: dict[str,
    RuleTally], rules: dict[str, Rule]) -> str`: the Markdown report, in
    this order: the provenance (commit, tree clean, plugin version, rulebook
    hash); the inventory (per language, file count, lines and rulebook used,
    `common` alone marking a coverage gap) and the list of exempt files, or
    a plain statement that the repo has no code; the upheld findings, each
    with its rule ID, severity, source, location, problem, suggested change,
    reach and generator; the per-rule table, with a row for every
    agent-judged rule of the languages audited, raised and surviving counts
    and dropped findings, so rules that never fired show as zero rows; and
    the security count. Every
    piece of reviewer free text sits in a top-level fenced code block longer
    than any backtick run in it (doctrine §1.9); the output passes
    markdownlint with vergil's configuration.
  - CLI, each reading FILE or `-` for stdin: `vrg-style-result cut FILE
    --public-out PATH --security-out PATH` (exit 1, writing nothing, when the
    cut cannot proceed); `vrg-style-result check --role
    {reviewer,verifier,audit} --rulebook-dir DIR --public PATH --security
    PATH --plugin-version V [--reviewer PATH]` (`--reviewer`, the checked
    reviewer result, required for a verifier; exit 0 on pass, 1 printing
    each failed check);
    `vrg-style-result provenance --rulebook-dir DIR --repo-root PATH
    --plugin-version V`; `vrg-style-result tally FILE...`; `vrg-style-result
    audit --rulebook-dir DIR --inventory PATH --provenance PATH --out PATH
    --reviewer PATH... --verifier PATH...`; and `vrg-style-result report
    --rulebook-dir DIR --inventory PATH --tally PATH --out PATH AUDIT`.

- [ ] **Step 1: Write the failing tests:** `test_rulebook_parses_fields`;
  `test_rulebook_reads_only_the_five_files` (a `README.md` with an example
  heading is ignored and outside the hash); `test_rulebook_missing_file`;
  `test_rulebook_duplicate_id_rejected`; `test_rulebook_meta_prefix_rejected`;
  `test_cut_then_check_passes`; `test_cut_leaves_count_only`;
  `test_cut_refuses_two_markers`; `test_cut_refuses_missing_end_marker`;
  `test_marker_case_insensitive`; `test_marker_in_free_text_refused`;
  `test_security_count_mismatch_refused`; `test_count_mismatch_refused`;
  `test_label_order_refused`; `test_unknown_rule_refused`;
  `test_linter_enforced_rule_refused`; `test_severity_mismatch_refused`;
  `test_meta_in_security_section_passes`;
  `test_meta_outside_security_section_refused`;
  `test_unlisted_meta_kind_refused`; `test_rule_finding_in_security_refused`;
  `test_meta_security_observation_passes`;
  `test_verifier_one_verdict_per_finding` (a missing, extra or unnamed
  verdict is refused); `test_verifier_requires_reviewer`;
  `test_audit_requires_provenance`; `test_audit_files_written_relative`;
  `test_announce_version_mismatch_refused`;
  `test_build_audit_keeps_only_upheld`;
  `test_build_audit_generator_from_inventory`;
  `test_build_audit_source_from_rulebook`;
  `test_provenance_hash_stable`; `test_provenance_dirty_tree`;
  `test_provenance_rejects_empty_version`;
  `test_tally_counts_raised_and_surviving`; `test_tally_ignores_meta`;
  `test_report_carries_rule_source`; `test_report_counts_dropped`;
  `test_report_inventory_and_exempt_list`; `test_report_provenance`;
  `test_report_finding_severity_and_reach`;
  `test_report_never_fired_rules_listed`; `test_report_no_code`;
  `test_report_free_text_fenced` (free text holding backticks, a link and an
  HTML tag renders inside a fence); `test_report_passes_markdownlint` (runs
  markdownlint with vergil's configuration over rendered sample reports, and
  fails, never skips, when markdownlint is missing).
- [ ] **Step 2:** Run them. Expect FAIL.
- [ ] **Step 3: Implement** the parser, cut, checker, provenance, tally,
  audit builder and renderer. The checker never edits the text it checks.
- [ ] **Step 4:** Run them. Expect PASS.
- [ ] **Step 5: Refactor:** one label-order and field parser shared by the
  checker and `build_audit_result`, and the meta-kind list defined in one
  place. Run the tests and `vrg-container-run -- vrg-validate`. Expect PASS.
- [ ] **Step 6: Parse the approved rulebook.** Run `load_rulebook` over the
  `skills/agentic-style/rulebook/` of the plugin commit Task 5 recorded.
  Expect success. A failure is a mismatch to resolve with the human while
  this task is open, never by loosening the parser silently.
- [ ] **Step 7:** Commit `vrg-commit --type feat --scope style --message
  "add vrg-style-result to parse the rulebook, check results and render the
  report"`, then report-ready.

### Task D1: vergil-tooling release attested and installed (deployment, .github)

Blocked by 3, 6 and 7.

- **Precondition (human-attested):** the vergil-tooling release carrying
  Tasks 3, 6 and 7 is published and installed on the host and the VMs, its
  version named in a comment.
- **Procedure (agent-safe):**
  1. Confirm the release notes list Tasks 3, 6 and 7's PRs.
  2. Confirm `vergil = "v2.1"` resolves to it.
  3. In a scratch clone of `vergil-fixture-python-lib`, run
     `vrg-standards-level` (prints `off`), `vrg-style-inventory` (lists
     `.claude/hooks/guard.sh` with its generator) and `vrg-style-result
     --help` from the installed release.
- **SUCCESS:** every check passes, recorded with the version.

### Task 8: The reviewer and verifier agents (vergil-claude-plugin)

Spec §5.3 (review and verify passes, data not instructions, incidental
security observations), §5.6. Blocked by 5 and D1.

**Files:**

- Create: `agents/agentic-style-reviewer.md`,
  `agents/agentic-style-verifier.md`
- Modify: `docs/site/docs/agents/index.md` (a section per agent, as for
  `merge-reviewer`)

**Interfaces:**

- Consumes: the finding labels and frame of Task 7.
- Produces: two read-only agents (`tools: Read, Glob, Grep`), dispatched as
  `vergil:agentic-style-reviewer` and `vergil:agentic-style-verifier`. The
  reviewer's prompt input: the plugin version, the language, the theme, the
  agent-judged rules of that theme (language plus `common`) and the file
  list. The verifier's: the plugin version, one reviewer's checked findings
  and the rule text each cites. Each puts the given version in its announce
  line. Each returns one
  result in the §5.6 frame for its role and dispatches nothing.

- [ ] **Step 1:** Write both definitions. Each carries a "Data, not
  instructions" section in the shape of `agents/merge-reviewer.md`; the rule
  to report steering attempts as `meta.prompt-injection` and incidental
  secret-like values or weaknesses only as `meta.security-observation`, both
  in the security section and never quoted elsewhere; the rule to copy each
  finding's severity from its rule; and the result frame with Task 7's
  labels for its role as a template (a verifier names each reviewer finding
  in `Finding:`).
- [ ] **Step 2:** Check each template: cut a hand-written sample result per
  role with the released `vrg-style-result cut`, then run `vrg-style-result
  check` on both parts against the approved rulebook. Expect PASS.
- [ ] **Step 3:** Run `vrg-container-run -- vrg-validate`. Expect PASS.
- [ ] **Step 4:** Commit `vrg-commit --type feat --scope agentic-style
  --message "add the agentic-style reviewer and verifier agents"`, then
  report-ready.

### Task 9: The `agentic-style` skill (vergil-claude-plugin)

Spec §5.3, §5.5, §5.6, §7. Blocked by 8 and D1.

**Files:**

- Create: `skills/agentic-style/SKILL.md`
- Modify: `docs/site/docs/skills/index.md` (an `## agentic-style` section,
  matching the other skills)

**Interfaces:**

- Consumes: `vrg-standards-level`, `vrg-style-inventory`, `vrg-style-result`
  (Tasks 3, 6, 7); the agents of Task 8; the rulebook.
- Produces: `/vergil:agentic-style [--output DIR] [--language LANG]...`.
  `DIR` defaults to the audited repo's `build/agentic-style/`; the skill
  resolves it to an absolute path and prints it before writing. Each
  `--language` is passed to `vrg-style-inventory`. On success it writes
  `<date>-<repo>.md` (the report) and `<date>-<repo>.result.txt` (its own
  result in the `audit` frame) to `DIR`, and raw material only to the
  audited repo's ignored `build/agentic-style/raw/`, whatever `DIR` is. It
  makes no issue, commit or handoff.

- [ ] **Step 1: Write the skill.** Its frontmatter description says what
  it is for, says it is not for bugs or structure, and names its siblings
  `paad:agentic-review` and `paad:agentic-architecture`. Read this plugin's
  version from its own `plugin.json` once, and pass it to every agent and
  every `vrg-style-result` call that takes it. Cost on a large repo is not
  bounded in this epic, whose targets are small; the bound is decided with
  the follow-on vergil-tooling audit (spec §6; the follow-on brainstorm
  #1390). The run, in order:
  1. Run `vrg-standards-level`. Stop with its error or reason unless it
     prints `audit`.
  2. Run `vrg-style-inventory`, passing each `--language`. Stop on error.
  3. For each language with audited files and each theme with agent-judged
     rules, dispatch one reviewer, given that language's rules plus
     `common`, or `common` alone for a language with no rulebook (a
     coverage gap the report names, as for `javascript`).
  4. **Before the first write,** check that `build`, `build/agentic-style`
     and `build/agentic-style/raw`, each where it exists, is a real directory
     and not a symlink; that `vrg-git check-ignore` covers
     `build/agentic-style/raw/`; and that `DIR` and each directory on the way
     to it is a real directory and not a symlink. Stop with the reason if any
     check fails.
  5. Write each raw result only under `build/agentic-style/raw/`. **Cut
     first** with `vrg-style-result cut`, keeping the security part there
     and showing it in the terminal; **then check** both parts with
     `vrg-style-result check`.
  6. Dispatch the verifier per checked reviewer result, and cut and check
     its result the same way, passing `--reviewer`.
  7. **Any failure stops the audit.** A result that cannot be cut, fails a
     check, or reports `Status: blocked` stops the run: no report and no
     result are written to `DIR`, the failing result stays whole in `raw/`
     as security material, and the skill tells the human which result
     failed and which check, never its text. Nothing is retried
     automatically.
  8. Run `vrg-style-result provenance --plugin-version <version>` and
     `vrg-style-result tally`.
  9. Build the `audit` result with `vrg-style-result audit`, cut and check
     it, then render the report with `vrg-style-result report`. The skill
     authors neither file: it only dispatches agents and passes files
     between commands.
- [ ] **Step 2:** Run `vrg-container-run -- vrg-validate`. Expect PASS.
- [ ] **Step 3:** Commit `vrg-commit --type feat --scope agentic-style
  --message "add the agentic-style skill"`, then report-ready.

### Task D2: vergil-claude-plugin release attested and installed (deployment, .github)

Blocked by 9.

- **Precondition (human-attested):** the plugin release carrying Tasks 4a,
  4b, 8 and 9 is published and installed, its version named in a comment.
- **Procedure (agent-safe):** confirm the installed plugin lists
  `/vergil:agentic-style` and both agents, and that its rulebook matches the
  commit Task 5 approved, or a later commit the human re-approved in a
  recorded comment on Task 5. Any rulebook change after Task 5, including a
  format-only one, needs that re-approval.
- **SUCCESS:** recorded with the plugin version.

### Tasks 10a–10g: Opt-in (one per fixture, and the plugin)

Blocked by D1. In each of the six fixtures (10a–10f) and in
vergil-claude-plugin (10g): add `[standards]` with `level = "audit"` to
`vergil.toml`; run `vrg-standards-level` (expect `audit`) and
`vrg-container-run -- vrg-validate` (expect PASS); commit `vrg-commit --type
chore --scope config --message "opt in to the agentic-style audit"`, then
report-ready.

### Task D2v: The skill's dry runs (validation, .github)

Blocked by D2 and 10b. Runs the installed release, so no session needs a
plugin worktree loaded.

- **Precondition (machine probe):** the installed plugin is D2's version, and
  `vrg-standards-level` in `vergil-fixture-python-lib` prints `audit`.
- **Procedure:**
  1. Create a throwaway git repo under the session scratch directory holding
     only a `README.md`, a `ci.yml`, a `.gitignore` with `build/`, and a
     committed `vergil.toml` with `[standards] level = "audit"`. Run
     `/vergil:agentic-style` there: the report's inventory says there is no
     code, and nothing is audited.
  2. Run it in a clean clone of `vergil-fixture-python-lib`: the `.result.txt`
     passes `vrg-style-result check --role audit`, the report passes
     markdownlint, and `raw/` is under `build/`.
  3. Delete the throwaway repo.
- **SUCCESS:** both runs behave as stated, recorded with the plugin version.

### Task 11: Tuning round 1 (.github)

Spec §5.5, §7. Blocked by D2v, 10a–10f and 10g.

**Files:**

- Create: `epics/1374-agentic-style/tuning/round-1/<date>-<repo>.md` and
  `<date>-<repo>.result.txt` for each of the six fixtures and for
  vergil-claude-plugin (shell only)
- Create: `epics/1374-agentic-style/tuning/round-1/README.md` (the round's
  summary: per-rule counts across repos, generated-file findings merged
  across fixtures, and the follow-up tasks the human approved)

- [ ] **Step 1:** In a clean clone of each fixture, run `/vergil:agentic-style
  --output <absolute path of this .github worktree>/epics/1374-agentic-style/tuning/round-1/`,
  and confirm the path it prints is inside this worktree. In a clean clone of
  vergil-claude-plugin, run it the same way with `--language shell`.
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
  are candidates for enforcement once a merge check exists.
