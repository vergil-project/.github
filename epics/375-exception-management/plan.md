# Exception management — Implementation Plan

> **For agentic workers:** each `## Task` below becomes one GitHub task issue
> under epic `vergil-project/.github#375`, implemented with `issue-implement`
> (one task = one PR, in the repo named on the task). Within a task, use
> superpowers:subagent-driven-development or superpowers:executing-plans; steps
> use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Every exception (inline suppression or rule disable) in a
Vergil-managed repo is documented in a register, agents ask before adding one,
and a deterministic `vrg-exceptions audit` gate fails anything undocumented.

**Architecture:** A new `vergil_tooling.lib.exceptions` package: a pure model,
a register parser, a `RepoTree` abstraction (working tree or git ref), one
plug-in per exception mechanism, a pure audit engine, a managed-block module for
ruff's fleet ignores, and diff/report renderers. A `vrg-exceptions` CLI exposes
`audit`/`sync`/`report`/`diff`; `vrg-validate` runs `audit` as a common check;
`vrg-submit-pr` prints the diff in its preview. The agent side is a consumer
CLAUDE.md rule plus a `vergil:exception-request` skill and a PostToolUse hook in
vergil-claude-plugin.

**Tech Stack:** Python 3.12+ (`vergil_tooling`; `tomllib`, `tokenize`,
`hashlib`, `fnmatch`), strict mypy + ty, pytest with a 100% line+branch
coverage gate, ruff, git plumbing (`ls-files`, `ls-tree`, `show`); bash + `jq`
for the plugin hook.

**Spec:** `epics/375-exception-management/spec.md`. Read it alongside this
plan; section references (§) point there.

## Global Constraints

- Python floor 3.12; CI runs 3.12, 3.13, 3.14 and requires 100% line **and
  branch** coverage on each (`--cov-fail-under=100`, `--cov-branch`).
- Type checks: `mypy` (strict, `src/`) and `ty check src tests` must pass.
- The only validation command is `vrg-container-run -- vrg-validate` (expands to
  `uv run vrg-validate` in vergil-tooling via `vergil.toml`).
- Git and GitHub only through `vrg-git` / `vrg-gh` / `vrg-commit`; agents stop at
  `vrg-pr-workflow report-ready` (never `vrg-submit-pr`).
- Every `bin/` module: `main()` first, decorated with `@cli_main`
  (`vergil_tooling.lib.cli`), registered in `[project.scripts]`.
- No silent failures: every error path prints an actionable message via
  `vergil_tooling.lib.output.emit_error` and exits non-zero. No bare `except`.
- Register file name: `vergil-exceptions.toml` (repo root). IDs: `EXC-NNN`, at
  least three digits — the only ID namespace. Fleet register entries carry no ID;
  they are identified by tool and rule (§5.3). Line-level exceptions are always
  per repo; the fleet register has no `[[inline]]` entries.
- Inline marker: a **separate** comment `# vrg-exc: <ID>` after the tool's own
  suppression on the same line, or alone on the line immediately above (§5.2).
- `issue` is required iff `expires` is set; `expires` is required for `trivy`
  entries; `scope` is required on `[[rule]]` and repo-wide is written `["**"]`.
- Expired entries and stale managed blocks are **reported, never failed**
  (§6.2, §6.3).
- `diff` reads both sides from git refs, never the working tree; an unresolvable
  ref is a hard error (§9).
- New code avoids suppressions where an idiom exists: plug-in method
  implementations carry `@typing.override` (ruff's `ARG002` skips overrides),
  narrowing uses `typing.cast`, not `assert`. This epic should not add the very
  exceptions it exists to manage.
- Until T9-final (the end of vergil-tooling's "kill the exceptions" series)
  lands, vergil-tooling has no register, so the audit is inactive here; new code
  in T1–T8 and T10 may add suppressions the usual way, and the series purges or
  documents them.
- Every suppression must name its rules: a blanket suppression fails the audit
  whatever its marker says (spec §6.1 item 10).

## Review Focus

1. **Suppression text inside strings.** vergil-tooling's own tests and docs
   contain text like `"# noqa: S603"` inside string literals. Expected: only
   real comments count. Python plug-ins find comments with `tokenize`, never by
   line regex (T2 test `test_noqa_inside_string_is_not_a_site`).
2. **Blanket suppressions.** `# noqa` with no codes, or `# type: ignore` with no
   code, suppress everything and cannot be tied to a rule. Expected: a hard
   finding `blanket` naming the file and line, whatever its marker says (spec
   §6.1 item 10; T1 audit test `test_blanket_site_fails`; T2 plug-in tests emit
   `rules=()`).
3. **A malformed or unreadable register.** Expected: a hard error naming
   `vergil-exceptions.toml` and the TOML parse message — never "not adopted",
   never a pass (T1 `test_malformed_toml_is_error`, T5 CLI test).
4. **One comment suppressing two rules covered by different entries.**
   `# noqa: S603, S607  # vrg-exc: EXC-014` where EXC-014 lists only `S603`.
   Expected: a `rule-not-covered` finding naming `S607` (T1 audit test).
5. **Non-UTF-8 source file.** Expected: a hard error naming the file, not a
   crash traceback and not a silent skip (T1 `test_worktree_non_utf8_is_error`).

## Sequencing

```text
T1 core ──▶ T2 python plug-ins ──┐
   │                             ├──▶ T5 CLI + vrg-validate ──▶ T6 diff + submit-pr preview
   ├──▶ T3 other plug-ins ───────┤                         └──▶ T8 report + docs page
   └──▶ T4 fleet register + ruff managed block ┘
T7a (vergil-tooling: consumer rule + standards docs)   — independent
T7b (vergil-claude-plugin: skill + hook)               — independent
T10 repo-init seeds the register + tags the generated nosemgrep ◀── T1, T3, T4
T9 kill-the-exceptions in vergil-tooling (joint)  ◀── T1–T6, T7a, T7b
   └──▶ purge tasks (filed by T9) ──▶ T9-final: document the remainder, enable the audit
D1 deployment: release vergil-tooling + plugin   ◀── T1–T8, T10, T7a, T7b
Follow-on: revisit the generated secrets: inherit — not blocking
```

Bookends already filed: documentation `.github#376`, docs review
`vergil-tooling#3230`, follow-on brainstorms `.github#378` and `.github#379`,
retrospective `.github#377`.

---

## Task T1: core model, register parser, repo trees, markers and audit engine

**Repo:** vergil-tooling. Blocked by: nothing. Spec: §4, §5.1, §5.2, §6.1, §6.2,
§11.

**Files:**

- Create: `src/vergil_tooling/lib/exceptions/__init__.py` (empty docstring module)
- Create: `src/vergil_tooling/lib/exceptions/model.py`
- Create: `src/vergil_tooling/lib/exceptions/register.py`
- Create: `src/vergil_tooling/lib/exceptions/tree.py`
- Create: `src/vergil_tooling/lib/exceptions/markers.py`
- Create: `src/vergil_tooling/lib/exceptions/plugin.py`
- Create: `src/vergil_tooling/lib/exceptions/audit.py`
- Modify: `src/vergil_tooling/lib/git.py` (add `read_raw`)
- Test: `tests/vergil_tooling/test_exceptions_model.py`,
  `test_exceptions_register.py`, `test_exceptions_tree.py`,
  `test_exceptions_markers.py`, `test_exceptions_audit.py`
- Test helper: `tests/vergil_tooling/exceptions_fakes.py` (`FakeTree`, `FakePlugin`)

**Interfaces (produced — later tasks rely on these exact names):**

```python
# model.py
Level = Literal["rule", "inline"]
@dataclass(frozen=True) class Entry: id: str; level: Level; tool: str; rules: tuple[str, ...]; added: date; reason: str; scope: tuple[str, ...] = (); expires: date | None = None; issue: str | None = None
@dataclass(frozen=True) class Register: entries: tuple[Entry, ...]; def get(self, entry_id: str) -> Entry | None
@dataclass(frozen=True) class Site: tool: str; path: str; line: int; rules: tuple[str, ...]; exc_id: str | None   # rules == () means blanket
@dataclass(frozen=True) class ConfigDisable: tool: str; rule: str; scope: tuple[str, ...]; source: str; managed: bool = False
@dataclass(frozen=True) class Finding: code: str; message: str; path: str | None = None; line: int | None = None
@dataclass(frozen=True) class Notice: code: str; message: str
@dataclass(frozen=True) class AuditResult: findings: tuple[Finding, ...]; notices: tuple[Notice, ...]; ok -> bool (property)

# register.py
REGISTER_FILE = "vergil-exceptions.toml"
class RegisterError(Exception): source: str; problems: tuple[str, ...]
FLEET_LABEL_PREFIX = "fleet:"   # a fleet entry's Entry.id is the label "fleet:<tool>:<rule>[+<rule>…]"
def parse_register(text: str, *, source: str, known_tools: frozenset[str], fleet: bool = False) -> Register
def register_from_data(raw: dict[str, Any], *, source: str, known_tools: frozenset[str], fleet: bool = False, extra_tables: frozenset[str] = frozenset()) -> Register

# tree.py
class RepoTree(Protocol): def files(self) -> list[str]; def read(self, path: str) -> str | None
class WorkTree: def __init__(self, root: Path) -> None
class GitTree: def __init__(self, ref: str, *, repo_root: Path) -> None
class TreeReadError(Exception)

# markers.py
MARKER_RE: re.Pattern[str]
def marker_id(line: str, previous: str | None) -> str | None

# plugin.py
class Plugin(ABC): name: str; def detect(self, tree: RepoTree) -> bool; def sites(self, tree: RepoTree) -> list[Site]; def config_disables(self, tree: RepoTree) -> list[ConfigDisable]; def dead_sites(self, root: Path, sites: list[Site]) -> list[Site]  (default: [])
class UncoveredDetector(Protocol): name: str; def detect(self, tree: RepoTree) -> bool

# audit.py
@dataclass(frozen=True) class Collected: sites: tuple[Site, ...]; disables: tuple[ConfigDisable, ...]; dead: tuple[Site, ...]; uncovered: tuple[str, ...]
def collect(tree: RepoTree, plugins: Sequence[Plugin], uncovered: Sequence[UncoveredDetector], *, root: Path | None) -> Collected
def audit(collected: Collected, register: Register, fleet: Register, *, today: date, extra_findings: Sequence[Finding] = (), extra_notices: Sequence[Notice] = ()) -> AuditResult

# git.py
def read_raw(*args: str) -> str   # like read_output but stdout NOT stripped
```

Finding codes (stable strings, used by CLI output and tests): `missing-id`,
`unknown-id`, `rule-not-covered`, `level-mismatch`, `blanket`, `dead`,
`undocumented-disable`, `stale-entry`, `unused-entry`. Notice codes:
`expired`, `expiring`, `promotion-candidate`, `uncovered`.

- [ ] **Step 1: Write the failing model and register tests**

```python
# tests/vergil_tooling/test_exceptions_register.py
from datetime import date

import pytest

from vergil_tooling.lib.exceptions.register import RegisterError, parse_register

TOOLS = frozenset({"ruff", "mypy", "trivy"})


def _parse(text: str):
    return parse_register(text, source="vergil-exceptions.toml", known_tools=TOOLS)


def test_parses_rule_and_inline_entries() -> None:
    reg = _parse(
        '[[rule]]\nid = "EXC-001"\ntool = "ruff"\nrules = ["S101"]\nscope = ["tests/**"]\n'
        'reason = "pytest asserts"\nadded = 2026-10-20\n\n'
        '[[inline]]\nid = "EXC-014"\ntool = "ruff"\nrules = ["S603"]\nreason = "fixed argv"\n'
        "added = 2026-10-20\n"
    )
    rule = reg.get("EXC-001")
    assert rule is not None
    assert rule.level == "rule"
    assert rule.scope == ("tests/**",)
    assert rule.added == date(2026, 10, 20)
    inline = reg.get("EXC-014")
    assert inline is not None
    assert inline.level == "inline"
    assert inline.scope == ()


def test_empty_file_is_an_empty_register() -> None:
    assert _parse("").entries == ()


def test_malformed_toml_is_error() -> None:
    with pytest.raises(RegisterError) as exc:
        _parse("[[rule]\n")
    assert exc.value.source == "vergil-exceptions.toml"
    assert "TOML" in exc.value.problems[0]


@pytest.mark.parametrize(
    ("body", "problem"),
    [
        ('id = "X-1"', "id 'X-1' must match EXC-NNN"),
        ('id = "EXC-001"\ntool = "nope"', "unknown tool 'nope'"),
        ('id = "EXC-001"\ntool = "ruff"\nrules = []', "rules must be a non-empty list"),
        ('id = "EXC-001"\ntool = "ruff"\nrules = ["S1"]\nreason = "r"\nadded = 2026-10-20',
         "[[rule]] requires scope"),
        ('id = "EXC-001"\ntool = "ruff"\nrules = ["S1"]\nscope = ["**"]\nadded = 2026-10-20',
         "reason is required"),
        ('id = "EXC-001"\ntool = "ruff"\nrules = ["S1"]\nscope = ["**"]\nreason = "r"',
         "added must be a date"),
        ('id = "EXC-001"\ntool = "ruff"\nrules = ["S1"]\nscope = ["**"]\nreason = "r"\n'
         'added = 2026-10-20\nexpires = 2027-01-01', "issue is required when expires is set"),
        ('id = "EXC-001"\ntool = "ruff"\nrules = ["S1"]\nscope = ["**"]\nreason = "r"\n'
         'added = 2026-10-20\nissue = "o/r#1"', "issue is only allowed when expires is set"),
        ('id = "EXC-001"\ntool = "trivy"\nrules = ["CVE-1"]\nscope = ["**"]\nreason = "r"\n'
         "added = 2026-10-20", "trivy entries require expires"),
        ('id = "EXC-001"\ntool = "ruff"\nrules = ["S1"]\nscope = ["**"]\nreason = "r"\n'
         'added = 2026-10-20\ncolour = "red"', "unknown key 'colour'"),
        ('id = "EXC-001"\ntool = "ruff"\nrules = ["S1"]\nscope = ["**"]\nreason = "r"\n'
         'added = 2026-10-20\nfleet = "VRG-004"', "unknown key 'fleet'"),
    ],
)
def test_rule_entry_validation(body: str, problem: str) -> None:
    with pytest.raises(RegisterError) as exc:
        _parse(f"[[rule]]\n{body}\n")
    assert any(problem in p for p in exc.value.problems)


def test_inline_rejects_scope() -> None:
    with pytest.raises(RegisterError) as exc:
        _parse('[[inline]]\nid = "EXC-002"\ntool = "ruff"\nrules = ["S1"]\nscope = ["**"]\n'
               'reason = "r"\nadded = 2026-10-20\n')
    assert any("[[inline]] does not take scope" in p for p in exc.value.problems)


def test_duplicate_ids_rejected() -> None:
    one = '[[inline]]\nid = "EXC-001"\ntool = "ruff"\nrules = ["S1"]\nreason = "r"\nadded = 2026-10-20\n'
    with pytest.raises(RegisterError) as exc:
        _parse(one + one)
    assert any("duplicate id 'EXC-001'" in p for p in exc.value.problems)


def test_unknown_top_level_table_rejected() -> None:
    with pytest.raises(RegisterError) as exc:
        _parse('[[rules]]\nid = "EXC-001"\n')
    assert any("unknown table 'rules'" in p for p in exc.value.problems)


def _parse_fleet(text: str):
    return parse_register(text, source="fleet", known_tools=TOOLS, fleet=True)


def test_fleet_entries_are_keyed_by_tool_and_rule() -> None:
    reg = _parse_fleet('[[rule]]\ntool = "ruff"\nrules = ["D400", "D415"]\nscope = ["**"]\n'
                       'reason = "r"\nadded = 2026-10-20\n')
    assert reg.get("fleet:ruff:D400+D415") is not None


@pytest.mark.parametrize(
    ("text", "problem"),
    [
        ('[[rule]]\nid = "EXC-001"\ntool = "ruff"\nrules = ["D400"]\nscope = ["**"]\nreason = "r"\n'
         "added = 2026-10-20\n", "fleet entries take no id"),
        ('[[inline]]\ntool = "ruff"\nrules = ["D400"]\nreason = "r"\nadded = 2026-10-20\n',
         "unknown table 'inline'"),
        ('[[rule]]\ntool = "ruff"\nrules = ["D400"]\nscope = ["**"]\nreason = "r"\nadded = 2026-10-20\n'
         '[[rule]]\ntool = "ruff"\nrules = ["D400"]\nscope = ["tests/**"]\nreason = "r"\nadded = 2026-10-20\n',
         "duplicate fleet rule 'ruff D400'"),
    ],
)
def test_fleet_validation(text: str, problem: str) -> None:
    with pytest.raises(RegisterError) as exc:
        _parse_fleet(text)
    assert any(problem in p for p in exc.value.problems)
```

- [ ] **Step 2: Run to verify failure**

Run: `vrg-container-run -- uv run pytest tests/vergil_tooling/test_exceptions_register.py -q --no-cov`
Expected: FAIL — `ModuleNotFoundError: vergil_tooling.lib.exceptions`.

- [ ] **Step 3: Implement `model.py` and `register.py`**

```python
# src/vergil_tooling/lib/exceptions/model.py
"""Value types for exception management (epic vergil-project/.github#375)."""

from __future__ import annotations

from dataclasses import dataclass
from typing import TYPE_CHECKING, Literal

if TYPE_CHECKING:
    from datetime import date

Level = Literal["rule", "inline"]


@dataclass(frozen=True)
class Entry:
    """One register entry: a documented rule-level or line-level exception."""

    id: str
    level: Level
    tool: str
    rules: tuple[str, ...]
    added: date
    reason: str
    scope: tuple[str, ...] = ()
    expires: date | None = None
    issue: str | None = None


@dataclass(frozen=True)
class Register:
    entries: tuple[Entry, ...]

    def get(self, entry_id: str) -> Entry | None:
        return next((e for e in self.entries if e.id == entry_id), None)


@dataclass(frozen=True)
class Site:
    """An inline suppression found in source. ``rules == ()`` is a blanket."""

    tool: str
    path: str
    line: int
    rules: tuple[str, ...]
    exc_id: str | None


@dataclass(frozen=True)
class ConfigDisable:
    """A rule turned off in tool configuration for ``scope`` (globs)."""

    tool: str
    rule: str
    scope: tuple[str, ...]
    source: str
    managed: bool = False


@dataclass(frozen=True)
class Finding:
    code: str
    message: str
    path: str | None = None
    line: int | None = None


@dataclass(frozen=True)
class Notice:
    code: str
    message: str


@dataclass(frozen=True)
class AuditResult:
    findings: tuple[Finding, ...]
    notices: tuple[Notice, ...]

    @property
    def ok(self) -> bool:
        return not self.findings
```

```python
# src/vergil_tooling/lib/exceptions/register.py
"""Parse and validate ``vergil-exceptions.toml`` and the fleet register (spec §5)."""

from __future__ import annotations

import re
import tomllib
from datetime import date
from typing import Any, cast

from vergil_tooling.lib.exceptions.model import Entry, Level, Register

REGISTER_FILE = "vergil-exceptions.toml"
#: A fleet entry has no ID; its ``Entry.id`` is this prefix plus ``<tool>:<rule>[+<rule>…]``,
#: a display label that no ``# vrg-exc:`` marker can name (markers take ``EXC-NNN``).
FLEET_LABEL_PREFIX = "fleet:"

_ID_RE = re.compile(r"^EXC-\d{3,}$")
_ISSUE_RE = re.compile(r"^[\w.-]+/[\w.-]+#\d+$")
_KEYS = frozenset({"id", "tool", "rules", "scope", "reason", "added", "expires", "issue"})


class RegisterError(Exception):
    """A register that cannot be parsed or fails validation."""

    def __init__(self, source: str, problems: list[str]) -> None:
        self.source = source
        self.problems = tuple(problems)
        super().__init__(f"{source}: " + "; ".join(problems))


def parse_register(text: str, *, source: str, known_tools: frozenset[str], fleet: bool = False) -> Register:
    try:
        raw = tomllib.loads(text)
    except tomllib.TOMLDecodeError as exc:
        raise RegisterError(source, [f"invalid TOML: {exc}"]) from exc
    return register_from_data(raw, source=source, known_tools=known_tools, fleet=fleet)


def register_from_data(
    raw: dict[str, Any],
    *,
    source: str,
    known_tools: frozenset[str],
    fleet: bool = False,
    extra_tables: frozenset[str] = frozenset(),
) -> Register:
    """Validate already-parsed TOML. *fleet* selects the fleet register's rules
    (``[[rule]]`` only, no ``id``; spec §5.3). *extra_tables* are keys owned by the
    caller (the fleet register's ``revision``/``managed``/``history``, Task T4)."""
    problems: list[str] = []
    entries: list[Entry] = []
    levels: tuple[Level, ...] = ("rule",) if fleet else ("rule", "inline")
    for table in raw:
        if table not in levels and table not in extra_tables:
            expected = " or ".join(f"[[{level}]]" for level in levels)
            problems.append(f"unknown table '{table}' (expected {expected})")
    for level in levels:
        items = raw.get(level, [])
        if not isinstance(items, list):
            problems.append(f"'{level}' must be an array of tables ([[{level}]])")
            continue
        for index, item in enumerate(items):
            entry = _entry(level, index, item, fleet, known_tools, problems)
            if entry is not None:
                entries.append(entry)
    if fleet:
        claimed: set[tuple[str, str]] = set()
        for entry in entries:
            for rule in entry.rules:
                if (entry.tool, rule) in claimed:
                    problems.append(f"duplicate fleet rule '{entry.tool} {rule}'")
                claimed.add((entry.tool, rule))
    else:
        seen: set[str] = set()
        for entry in entries:
            if entry.id in seen:
                problems.append(f"duplicate id '{entry.id}'")
            seen.add(entry.id)
    if problems:
        raise RegisterError(source, problems)
    return Register(entries=tuple(entries))


def _str_list(value: object) -> tuple[str, ...] | None:
    if isinstance(value, list) and value and all(isinstance(v, str) and v for v in value):
        return tuple(value)
    return None


def _entry(
    level: Level,
    index: int,
    item: Any,
    fleet: bool,
    known_tools: frozenset[str],
    problems: list[str],
) -> Entry | None:
    where = f"[[{level}]] #{index + 1}"
    start = len(problems)
    if not isinstance(item, dict):
        problems.append(f"{where}: must be a table")
        return None
    for key in item:
        if key not in _KEYS:
            problems.append(f"{where}: unknown key '{key}'")
    entry_id = item.get("id")
    if fleet:
        if entry_id is not None:
            problems.append(f"{where}: fleet entries take no id (they are identified by tool and rule)")
    elif not isinstance(entry_id, str) or not _ID_RE.match(entry_id):
        problems.append(f"{where}: id '{entry_id}' must match EXC-NNN")
    else:
        where = entry_id
    tool = item.get("tool")
    if tool not in known_tools:
        problems.append(f"{where}: unknown tool '{tool}'")
    rules = _str_list(item.get("rules"))
    if rules is None:
        problems.append(f"{where}: rules must be a non-empty list of strings")
    scope: tuple[str, ...] = ()
    if level == "rule":
        parsed_scope = _str_list(item.get("scope"))
        if parsed_scope is None:
            problems.append(f"{where}: [[rule]] requires scope (a non-empty list of globs; repo-wide is [\"**\"])")
        else:
            scope = parsed_scope
    elif "scope" in item:
        problems.append(f"{where}: [[inline]] does not take scope")
    reason = item.get("reason")
    if reason is None:
        problems.append(f"{where}: reason is required")
    elif not isinstance(reason, str) or not reason.strip():
        problems.append(f"{where}: reason must be non-empty text")
    added = item.get("added")
    if not isinstance(added, date):
        problems.append(f"{where}: added must be a date (YYYY-MM-DD)")
    expires = item.get("expires")
    if expires is not None and not isinstance(expires, date):
        problems.append(f"{where}: expires must be a date (YYYY-MM-DD)")
    issue = item.get("issue")
    if expires is not None and issue is None:
        problems.append(f"{where}: issue is required when expires is set")
    if expires is None and issue is not None:
        problems.append(f"{where}: issue is only allowed when expires is set")
    if issue is not None and (not isinstance(issue, str) or not _ISSUE_RE.match(issue)):
        problems.append(f"{where}: issue must look like owner/repo#N")
    if tool == "trivy" and expires is None:
        problems.append(f"{where}: trivy entries require expires (CVE suppressions are temporary)")
    if len(problems) > start:
        return None
    # Every field below was validated above (no new problems were recorded).
    rules = cast("tuple[str, ...]", rules)
    label = f"{FLEET_LABEL_PREFIX}{tool}:{'+'.join(rules)}" if fleet else cast("str", entry_id)
    return Entry(
        id=label,
        level=level,
        tool=cast("str", tool),
        rules=rules,
        added=cast("date", added),
        reason=cast("str", reason),
        scope=scope,
        expires=expires,
        issue=issue,
    )
```

> Note: vergil-tooling's ruff `select` has no `C90`/`PL` rules, so do not add
> complexity `noqa` comments — they would be dead (and fail the audit once T9
> adopts it). `S101` is selected for `src/`, hence `cast` rather than `assert`.

- [ ] **Step 4: Run the register tests to verify they pass**

Run: `vrg-container-run -- uv run pytest tests/vergil_tooling/test_exceptions_register.py -q --no-cov`
Expected: PASS.

- [ ] **Step 5: Write failing tests for `markers.py`, `tree.py` and `git.read_raw`**

```python
# tests/vergil_tooling/test_exceptions_markers.py
from vergil_tooling.lib.exceptions.markers import marker_id


def test_same_line_marker() -> None:
    assert marker_id("x = 1  # noqa: S603  # vrg-exc: EXC-014", None) == "EXC-014"


def test_non_exc_id_is_returned_for_the_audit_to_reject() -> None:
    # Only EXC-NNN IDs exist; any other ID is read so the audit can fail it as unknown-id
    # rather than reporting the site as having no marker at all.
    assert marker_id("x = 1  # noqa: S603  # vrg-exc: VRG-007", None) == "VRG-007"


def test_line_above_marker_must_be_comment_only() -> None:
    assert marker_id("# shellcheck disable=SC2086", "# vrg-exc: EXC-020") == "EXC-020"
    assert marker_id("# shellcheck disable=SC2086", "x=1  # vrg-exc: EXC-020") is None


def test_trailing_text_form_is_not_a_marker() -> None:
    # spec §5.2: the ID must be its own comment
    assert marker_id("return x  # type: ignore[return-value]  vrg-exc: EXC-002", None) is None


def test_no_marker() -> None:
    assert marker_id("x = 1  # noqa: S603", None) is None


def test_markdown_marker() -> None:
    line = "<!-- markdownlint-disable-next-line MD013 --> <!-- vrg-exc: EXC-005 -->"
    assert marker_id(line, None) == "EXC-005"
    assert marker_id("<!-- markdownlint-disable MD033 -->", "<!-- vrg-exc: EXC-006 -->") == "EXC-006"
```

```python
# tests/vergil_tooling/test_exceptions_tree.py
import subprocess
from pathlib import Path

import pytest

from vergil_tooling.lib.exceptions.tree import GitTree, TreeReadError, WorkTree


def _git(root: Path, *args: str) -> None:
    subprocess.run(["git", "-C", str(root), *args], check=True, capture_output=True)  # noqa: S603, S607


@pytest.fixture
def repo(tmp_path: Path) -> Path:
    _git(tmp_path, "init", "-q", "-b", "main")
    _git(tmp_path, "config", "user.email", "t@example.com")
    _git(tmp_path, "config", "user.name", "t")
    (tmp_path / "a.py").write_text("\n\nx = 1\n", encoding="utf-8")
    _git(tmp_path, "add", "a.py")
    _git(tmp_path, "commit", "-q", "-m", "init")
    return tmp_path


def test_worktree_lists_tracked_and_untracked(repo: Path) -> None:
    (repo / "b.py").write_text("y = 2\n", encoding="utf-8")
    tree = WorkTree(repo)
    assert tree.files() == ["a.py", "b.py"]
    assert tree.read("a.py") == "\n\nx = 1\n"   # leading blank lines preserved
    assert tree.read("missing.py") is None


def test_worktree_non_utf8_is_error(repo: Path) -> None:
    (repo / "bad.py").write_bytes(b"\xff\xfe")
    with pytest.raises(TreeReadError, match="bad.py"):
        WorkTree(repo).read("bad.py")


def test_gittree_reads_ref_not_worktree(repo: Path) -> None:
    (repo / "a.py").write_text("changed\n", encoding="utf-8")
    tree = GitTree("main", repo_root=repo)
    assert tree.files() == ["a.py"]
    assert tree.read("a.py") == "\n\nx = 1\n"
    assert tree.read("missing.py") is None


def test_gittree_unknown_ref_is_error(repo: Path) -> None:
    with pytest.raises(TreeReadError, match="no-such-ref"):
        GitTree("no-such-ref", repo_root=repo)
```

- [ ] **Step 6: Run to verify failure**

Run: `vrg-container-run -- uv run pytest tests/vergil_tooling/test_exceptions_markers.py tests/vergil_tooling/test_exceptions_tree.py -q --no-cov`
Expected: FAIL — modules missing.

- [ ] **Step 7: Implement `git.read_raw`, `markers.py`, `tree.py`**

```python
# src/vergil_tooling/lib/git.py — add after read_output
def read_raw(*args: str) -> str:
    """Run a git command and return stdout exactly as produced (not stripped).

    For reading file content (``git show <ref>:<path>``), where stripping would
    shift line numbers.
    """
    env = _git_env(args)
    try:
        result = subprocess.run(  # noqa: S603
            ("git", *args),  # noqa: S607
            check=True,
            text=True,
            capture_output=True,
            env=env,
        )
    except subprocess.CalledProcessError as exc:
        if exc.stderr:
            print(exc.stderr, end="", file=sys.stderr)
        raise
    return result.stdout
```

Add a test in `tests/vergil_tooling/test_git.py` following that file's existing
mocking style for `read_output`: `read_raw` returns stdout unstripped
(`"\n\nx\n"` stays `"\n\nx\n"`) and prints stderr then re-raises on
`CalledProcessError`.

```python
# src/vergil_tooling/lib/exceptions/markers.py
"""Find the ``# vrg-exc: <ID>`` marker that ties a suppression to its entry (spec §5.2)."""

from __future__ import annotations

import re

#: A separate comment naming a register ID: ``# vrg-exc: ID`` (or, in Markdown,
#: ``<!-- vrg-exc: ID -->``). Must start its own comment. Valid IDs are ``EXC-NNN``;
#: any ``PREFIX-NNN`` is captured so the audit reports a wrong ID as ``unknown-id``.
MARKER_RE = re.compile(r"(?:#|<!--)\s*vrg-exc:\s*([A-Z]+-\d+)\b")
_COMMENT_ONLY_RE = re.compile(r"^\s*(?:#|<!--)\s*vrg-exc:\s*([A-Z]+-\d+)\b\s*(?:-->)?\s*$")


def marker_id(line: str, previous: str | None) -> str | None:
    """Return the ID on *line*, else on a comment-only *previous* line, else None."""
    match = MARKER_RE.search(line)
    if match:
        return match.group(1)
    if previous is not None:
        above = _COMMENT_ONLY_RE.match(previous)
        if above:
            return above.group(1)
    return None
```

```python
# src/vergil_tooling/lib/exceptions/tree.py
"""Read a repository either from the working tree or from a git ref (spec §9)."""

from __future__ import annotations

import subprocess
from typing import TYPE_CHECKING, Protocol

from vergil_tooling.lib import git

if TYPE_CHECKING:
    from pathlib import Path


class TreeReadError(Exception):
    """A file or ref that cannot be read."""


class RepoTree(Protocol):
    def files(self) -> list[str]: ...

    def read(self, path: str) -> str | None: ...


class WorkTree:
    """The checked-out tree: tracked plus untracked-not-ignored files."""

    def __init__(self, root: Path) -> None:
        self.root = root

    def files(self) -> list[str]:
        out = git.read_raw("-C", str(self.root), "ls-files", "--cached", "--others", "--exclude-standard")
        return sorted({line for line in out.splitlines() if line})

    def read(self, path: str) -> str | None:
        target = self.root / path
        if not target.is_file():
            return None
        try:
            return target.read_text(encoding="utf-8")
        except UnicodeDecodeError as exc:
            msg = f"{path}: not valid UTF-8 ({exc.reason}); cannot scan it for exceptions"
            raise TreeReadError(msg) from exc


class GitTree:
    """The tree at a git ref, read from objects — never the working tree."""

    def __init__(self, ref: str, *, repo_root: Path) -> None:
        self.ref = ref
        self.root = repo_root
        try:
            out = git.read_raw("-C", str(repo_root), "ls-tree", "-r", "--name-only", ref)
        except subprocess.CalledProcessError as exc:
            msg = f"cannot resolve git ref '{ref}' (fetch it first?)"
            raise TreeReadError(msg) from exc
        self._files = sorted(line for line in out.splitlines() if line)
        self._known = set(self._files)

    def files(self) -> list[str]:
        return list(self._files)

    def read(self, path: str) -> str | None:
        if path not in self._known:
            return None
        try:
            return git.read_raw("-C", str(self.root), "show", f"{self.ref}:{path}")
        except subprocess.CalledProcessError as exc:
            msg = f"{self.ref}:{path}: cannot read from git"
            raise TreeReadError(msg) from exc
        except UnicodeDecodeError as exc:
            msg = f"{self.ref}:{path}: not valid UTF-8; cannot scan it for exceptions"
            raise TreeReadError(msg) from exc
```

- [ ] **Step 8: Run the marker and tree tests to verify they pass**

Run: `vrg-container-run -- uv run pytest tests/vergil_tooling/test_exceptions_markers.py tests/vergil_tooling/test_exceptions_tree.py tests/vergil_tooling/test_git.py -q --no-cov`
Expected: PASS.

- [ ] **Step 9: Write the test fakes and failing audit tests**

```python
# tests/vergil_tooling/exceptions_fakes.py
from __future__ import annotations

from typing import TYPE_CHECKING

from vergil_tooling.lib.exceptions.plugin import Plugin

if TYPE_CHECKING:
    from pathlib import Path

    from vergil_tooling.lib.exceptions.model import ConfigDisable, Site
    from vergil_tooling.lib.exceptions.tree import RepoTree


class FakeTree:
    def __init__(self, files: dict[str, str]) -> None:
        self._files = files

    def files(self) -> list[str]:
        return sorted(self._files)

    def read(self, path: str) -> str | None:
        return self._files.get(path)


class FakePlugin(Plugin):
    name = "fake"

    def __init__(self, sites: list[Site] | None = None, disables: list[ConfigDisable] | None = None,
                 dead: list[Site] | None = None, present: bool = True) -> None:
        self._sites = sites or []
        self._disables = disables or []
        self._dead = dead or []
        self._present = present

    def detect(self, tree: RepoTree) -> bool:
        return self._present

    def sites(self, tree: RepoTree) -> list[Site]:
        return list(self._sites)

    def config_disables(self, tree: RepoTree) -> list[ConfigDisable]:
        return list(self._disables)

    def dead_sites(self, root: Path, sites: list[Site]) -> list[Site]:
        return list(self._dead)
```

```python
# tests/vergil_tooling/test_exceptions_audit.py
from datetime import date
from pathlib import Path

from tests.vergil_tooling.exceptions_fakes import FakePlugin, FakeTree
from vergil_tooling.lib.exceptions.audit import Collected, audit, collect
from vergil_tooling.lib.exceptions.model import ConfigDisable, Entry, Register, Site

TODAY = date(2026, 11, 1)
ADDED = date(2026, 10, 20)
EMPTY = Register(entries=())


def _inline(eid: str, *rules: str, expires: date | None = None) -> Entry:
    return Entry(id=eid, level="inline", tool="fake", rules=rules, added=ADDED, reason="r",
                 expires=expires, issue="o/r#1" if expires else None)


def _rule(eid: str, rule: str, *scope: str) -> Entry:
    return Entry(id=eid, level="rule", tool="fake", rules=(rule,), added=ADDED, reason="r", scope=scope)


def _site(exc_id: str | None, *rules: str, line: int = 3) -> Site:
    return Site(tool="fake", path="src/a.py", line=line, rules=rules, exc_id=exc_id)


def _codes(result) -> list[str]:
    return sorted(f.code for f in result.findings)


def _run(sites=(), disables=(), dead=(), entries=(), fleet=EMPTY, uncovered=()):
    collected = Collected(sites=tuple(sites), disables=tuple(disables), dead=tuple(dead),
                          uncovered=tuple(uncovered))
    return audit(collected, Register(entries=tuple(entries)), fleet, today=TODAY)


def test_documented_site_passes() -> None:
    result = _run(sites=[_site("EXC-001", "S603")], entries=[_inline("EXC-001", "S603")])
    assert result.ok


def test_missing_id_fails() -> None:
    assert _codes(_run(sites=[_site(None, "S603")])) == ["missing-id"]


def test_unknown_id_fails() -> None:
    assert _codes(_run(sites=[_site("EXC-009", "S603")])) == ["unknown-id"]


def test_rule_not_covered_names_the_rule() -> None:
    result = _run(sites=[_site("EXC-014", "S603", "S607")], entries=[_inline("EXC-014", "S603")])
    assert _codes(result) == ["rule-not-covered"]
    assert "S607" in result.findings[0].message


def test_blanket_site_fails() -> None:
    assert _codes(_run(sites=[_site("EXC-001")], entries=[_inline("EXC-001", "S603")])) == ["blanket"]


def test_site_pointing_at_rule_entry_is_level_mismatch() -> None:
    result = _run(sites=[_site("EXC-001", "S101")], entries=[_rule("EXC-001", "S101", "tests/**")],
                  disables=[ConfigDisable("fake", "S101", ("tests/**",), "cfg")])
    assert _codes(result) == ["level-mismatch"]


def test_inline_site_cannot_reference_a_fleet_entry() -> None:
    # Line-level exceptions are always per repo (spec §5.2): a non-EXC ID, or a fleet
    # entry's label, is unknown.
    fleet = Register(entries=(Entry(id="fleet:fake:S603", level="rule", tool="fake", rules=("S603",),
                                    added=ADDED, reason="r", scope=("**",)),))
    assert _codes(_run(sites=[_site("VRG-007", "S603")], fleet=fleet)) == ["unknown-id"]


def test_dead_site_fails() -> None:
    site = _site("EXC-001", "PLR2004")
    result = _run(sites=[site], dead=[site], entries=[_inline("EXC-001", "PLR2004")])
    assert _codes(result) == ["dead"]


def test_undocumented_disable_fails() -> None:
    assert _codes(_run(disables=[ConfigDisable("fake", "S101", ("tests/**",), "cfg")])) == [
        "undocumented-disable"]


def test_disable_covered_by_rule_entry_with_matching_scope() -> None:
    result = _run(disables=[ConfigDisable("fake", "S101", ("tests/**",), "cfg")],
                  entries=[_rule("EXC-001", "S101", "tests/**")])
    assert result.ok


def test_wider_disable_than_entry_scope_fails() -> None:
    result = _run(disables=[ConfigDisable("fake", "S101", ("tests/**", "examples/**"), "cfg")],
                  entries=[_rule("EXC-001", "S101", "tests/**")])
    assert _codes(result) == ["undocumented-disable"]


FLEET_D400 = Register(entries=(Entry(id="fleet:fake:D400", level="rule", tool="fake", rules=("D400",),
                                     added=ADDED, reason="r", scope=("**",)),))


def test_managed_disable_covered_by_fleet_rule() -> None:
    result = _run(disables=[ConfigDisable("fake", "D400", ("**",), "managed block", managed=True)],
                  fleet=FLEET_D400)
    assert result.ok


def test_repo_disable_needs_a_repo_entry_even_when_the_fleet_has_the_rule() -> None:
    # A fleet default reaches a repo by package or managed block; a repo that disables
    # the rule in its own config owns that exception (spec §5.1: no fleet field).
    result = _run(disables=[ConfigDisable("fake", "D400", ("**",), "pyproject.toml")], fleet=FLEET_D400)
    assert _codes(result) == ["undocumented-disable"]


def test_stale_rule_entry_fails() -> None:
    assert _codes(_run(entries=[_rule("EXC-001", "S101", "tests/**")])) == ["stale-entry"]


def test_unused_inline_entry_fails() -> None:
    assert _codes(_run(entries=[_inline("EXC-001", "S603")])) == ["unused-entry"]


def test_expired_is_notice_not_failure() -> None:
    result = _run(sites=[_site("EXC-001", "S603")],
                  entries=[_inline("EXC-001", "S603", expires=date(2026, 10, 1))])
    assert result.ok
    assert [n.code for n in result.notices] == ["expired"]


def test_expiring_within_30_days_is_notice() -> None:
    result = _run(sites=[_site("EXC-001", "S603")],
                  entries=[_inline("EXC-001", "S603", expires=date(2026, 11, 20))])
    assert [n.code for n in result.notices] == ["expiring"]


def test_promotion_candidate_notice() -> None:
    sites = [_site("EXC-001", "S603", line=n) for n in (1, 2, 3)]
    result = _run(sites=sites, entries=[_inline("EXC-001", "S603")])
    assert result.ok
    assert [n.code for n in result.notices] == ["promotion-candidate"]
    assert "3 sites" in result.notices[0].message


def test_uncovered_mechanism_notice() -> None:
    result = _run(uncovered=["eslint"])
    assert [n.code for n in result.notices] == ["uncovered"]


def test_collect_runs_dead_only_with_root() -> None:
    site = _site(None, "S1")
    plugin = FakePlugin(sites=[site], dead=[site])
    tree = FakeTree({})
    assert collect(tree, [plugin], [], root=None).dead == ()
    assert collect(tree, [plugin], [], root=Path("/r")).dead == (site,)


def test_collect_skips_absent_plugins_and_reports_uncovered() -> None:
    class Detector:
        name = "eslint"

        def detect(self, tree: object) -> bool:
            return True

    collected = collect(FakeTree({}), [FakePlugin(sites=[_site(None, "S1")], present=False)],
                        [Detector()], root=None)
    assert collected.sites == ()
    assert collected.uncovered == ("eslint",)
```

- [ ] **Step 10: Run to verify failure**

Run: `vrg-container-run -- uv run pytest tests/vergil_tooling/test_exceptions_audit.py -q --no-cov`
Expected: FAIL — `plugin`/`audit` modules missing.

- [ ] **Step 11: Implement `plugin.py` and `audit.py`**

```python
# src/vergil_tooling/lib/exceptions/plugin.py
"""The plug-in interface every exception mechanism implements (spec §7)."""

from __future__ import annotations

from abc import ABC, abstractmethod
from typing import TYPE_CHECKING, Protocol

if TYPE_CHECKING:
    from pathlib import Path

    from vergil_tooling.lib.exceptions.model import ConfigDisable, Site
    from vergil_tooling.lib.exceptions.tree import RepoTree


class Plugin(ABC):
    name: str

    @abstractmethod
    def detect(self, tree: RepoTree) -> bool:
        """True when the mechanism is present in the repo at all."""

    @abstractmethod
    def sites(self, tree: RepoTree) -> list[Site]:
        """Inline suppressions, with their ``vrg-exc`` IDs."""

    @abstractmethod
    def config_disables(self, tree: RepoTree) -> list[ConfigDisable]:
        """Rules turned off in configuration, with their scopes."""

    def dead_sites(self, root: Path, sites: list[Site]) -> list[Site]:
        """Sites that suppress a rule that is not enabled or suppress nothing.

        Working tree only. The default is "cannot tell" (spec §6.1 item 5:
        only where the tool can report it).
        """
        del root, sites
        return []


class UncoveredDetector(Protocol):
    """Detects a mechanism no plug-in covers yet, so it is reported (§6.2)."""

    name: str

    def detect(self, tree: RepoTree) -> bool: ...
```

```python
# src/vergil_tooling/lib/exceptions/audit.py
"""Combine sites, config disables and registers into findings (spec §6)."""

from __future__ import annotations

from collections import Counter
from dataclasses import dataclass
from datetime import timedelta
from typing import TYPE_CHECKING

from vergil_tooling.lib.exceptions.model import AuditResult, Finding, Notice

if TYPE_CHECKING:
    from collections.abc import Sequence
    from datetime import date
    from pathlib import Path

    from vergil_tooling.lib.exceptions.model import ConfigDisable, Entry, Register, Site
    from vergil_tooling.lib.exceptions.plugin import Plugin, UncoveredDetector
    from vergil_tooling.lib.exceptions.tree import RepoTree

EXPIRY_WARNING = timedelta(days=30)
PROMOTION_THRESHOLD = 3


@dataclass(frozen=True)
class Collected:
    sites: tuple[Site, ...]
    disables: tuple[ConfigDisable, ...]
    dead: tuple[Site, ...]
    uncovered: tuple[str, ...]


def collect(
    tree: RepoTree,
    plugins: Sequence[Plugin],
    uncovered: Sequence[UncoveredDetector],
    *,
    root: Path | None,
) -> Collected:
    """Run every present plug-in over *tree*. Dead detection needs *root*."""
    sites: list[Site] = []
    disables: list[ConfigDisable] = []
    dead: list[Site] = []
    for plugin in plugins:
        if not plugin.detect(tree):
            continue
        plugin_sites = plugin.sites(tree)
        sites.extend(plugin_sites)
        disables.extend(plugin.config_disables(tree))
        if root is not None:
            dead.extend(plugin.dead_sites(root, plugin_sites))
    names = tuple(d.name for d in uncovered if d.detect(tree))
    return Collected(tuple(sites), tuple(disables), tuple(dead), names)


def _overlaps(entry: Entry, disable: ConfigDisable) -> bool:
    """Same tool and rule with any scope in common — the entry is not stale."""
    return (
        entry.level == "rule"
        and entry.tool == disable.tool
        and disable.rule in entry.rules
        and bool(set(disable.scope) & set(entry.scope))
    )


def _covers(entry: Entry, disable: ConfigDisable) -> bool:
    return (
        entry.level == "rule"
        and entry.tool == disable.tool
        and disable.rule in entry.rules
        and set(disable.scope) <= set(entry.scope)
    )


def audit(
    collected: Collected,
    register: Register,
    fleet: Register,
    *,
    today: date,
    extra_findings: Sequence[Finding] = (),
    extra_notices: Sequence[Notice] = (),
) -> AuditResult:
    findings: list[Finding] = list(extra_findings)
    notices: list[Notice] = list(extra_notices)
    dead = set(collected.dead)
    used: Counter[str] = Counter()

    for site in collected.sites:
        where = {"path": site.path, "line": site.line}
        label = f"{site.tool} suppression"
        if not site.rules:
            if site.exc_id is not None:
                used[site.exc_id] += 1  # the entry is referenced; the blanket is the problem
            findings.append(Finding("blanket", f"{label} names no rule; suppress specific rules only", **where))
            continue
        if site.exc_id is None:
            findings.append(Finding("missing-id", f"{label} of {', '.join(site.rules)} has no '# vrg-exc: <ID>' "
                                    "marker; use /vergil:exception-request", **where))
            continue
        entry = register.get(site.exc_id)
        if entry is None:
            findings.append(Finding("unknown-id", f"{site.exc_id} is not in vergil-exceptions.toml (inline "
                                    "markers take the repo's EXC-NNN IDs only)", **where))
            continue
        used[entry.id] += 1
        if entry.level != "inline":
            findings.append(Finding("level-mismatch", f"{entry.id} is a [[rule]] entry; inline sites need an "
                                    "[[inline]] entry", **where))
            continue
        missing = [r for r in site.rules if entry.tool != site.tool or r not in entry.rules]
        if missing:
            findings.append(Finding("rule-not-covered", f"{entry.id} does not cover {', '.join(missing)} "
                                    f"({entry.tool}: {', '.join(entry.rules)})", **where))
        if site in dead:
            findings.append(Finding("dead", f"suppression of {', '.join(site.rules)} is dead (rule not "
                                    "enabled, or nothing to suppress); remove it", **where))

    for disable in collected.disables:
        # A managed-block rule is documented by the fleet; anything else in repo config is the repo's own.
        pool = fleet.entries if disable.managed else register.entries
        if not any(_covers(e, disable) for e in pool):
            findings.append(Finding("undocumented-disable", f"{disable.tool} rule {disable.rule} is disabled for "
                                    f"{', '.join(disable.scope)} in {disable.source} with no [[rule]] entry "
                                    "covering that scope"))

    for entry in register.entries:
        if entry.level == "rule" and not any(_overlaps(entry, d) for d in collected.disables):
            findings.append(Finding("stale-entry", f"{entry.id} documents {entry.tool} {', '.join(entry.rules)} "
                                    f"for {', '.join(entry.scope)} but no config disables it; remove the entry"))
        if entry.level == "inline" and used[entry.id] == 0:
            findings.append(Finding("unused-entry", f"{entry.id} is referenced by no suppression; remove it"))
        if entry.expires is not None:
            if entry.expires < today:
                notices.append(Notice("expired", f"{entry.id} expired {entry.expires} ({entry.issue})"))
            elif entry.expires - today <= EXPIRY_WARNING:
                notices.append(Notice("expiring", f"{entry.id} expires {entry.expires} ({entry.issue})"))
        if entry.level == "inline" and used[entry.id] >= PROMOTION_THRESHOLD:
            notices.append(Notice("promotion-candidate", f"{entry.id} covers {used[entry.id]} sites; consider "
                                  "a narrowly scoped [[rule]] exception instead"))

    notices.extend(Notice("uncovered", f"{name} exceptions are present but not audited yet")
                   for name in collected.uncovered)
    return AuditResult(findings=tuple(findings), notices=tuple(notices))
```

Also add `tests/vergil_tooling/test_exceptions_model.py` with one test that
`AuditResult(findings=(), notices=()).ok` is `True` and becomes `False` with one
`Finding`, and that `Register.get` returns `None` for an unknown ID.

- [ ] **Step 12: Run all T1 tests, then full validation**

Run: `vrg-container-run -- uv run pytest tests/vergil_tooling/test_exceptions_*.py -q --no-cov`
Expected: PASS.
Run: `vrg-container-run -- vrg-validate`
Expected: PASS, coverage 100%.

- [ ] **Step 13: Commit**

```bash
vrg-git add src/vergil_tooling/lib/exceptions src/vergil_tooling/lib/git.py tests/vergil_tooling
vrg-commit --type feat --scope exceptions --message "add exception register, repo trees and audit engine" --body "Ref #<task>"
```

---

## Task T2: Python plug-ins — ruff, mypy, ty, coverage

**Repo:** vergil-tooling. Blocked by: T1. Spec: §5.2, §6.1 items 5–6, §7.

**Files:**

- Create: `src/vergil_tooling/lib/exceptions/plugins/__init__.py` (registry)
- Create: `src/vergil_tooling/lib/exceptions/plugins/_python.py` (comment scanner)
- Create: `src/vergil_tooling/lib/exceptions/plugins/_config.py` (TOML config reader)
- Create: `src/vergil_tooling/lib/exceptions/plugins/ruff.py`, `mypy.py`, `ty.py`, `coverage.py`
- Test: `tests/vergil_tooling/test_exceptions_plugins_python.py`

**Interfaces:**

- Consumes (T1): `Plugin`, `UncoveredDetector`, `Site`, `ConfigDisable`,
  `RepoTree`, `TreeReadError`, `marker_id`.
- Produces:

```python
# plugins/__init__.py
PLUGINS: tuple[Plugin, ...]                 # T3 appends its plug-ins here
UNCOVERED: tuple[UncoveredDetector, ...]    # T3 fills this
def known_tools() -> frozenset[str]         # {p.name for p in PLUGINS} | {"trivy", ...} — every name a register may use

# plugins/_python.py
@dataclass(frozen=True) class Comment: path: str; line: int; text: str; previous: str | None
def python_comments(tree: RepoTree) -> list[Comment]   # every COMMENT token in every *.py file

# plugins/_config.py
def read_toml(tree: RepoTree, path: str) -> dict[str, Any] | None   # None if absent; TreeReadError if malformed
```

Plug-in names (the `tool` values a register uses): `ruff`, `mypy`, `ty`,
`coverage`.

- [ ] **Step 1: Write the failing tests**

```python
# tests/vergil_tooling/test_exceptions_plugins_python.py
import json
import subprocess
from pathlib import Path
from unittest import mock

import pytest

from tests.vergil_tooling.exceptions_fakes import FakeTree
from vergil_tooling.lib.exceptions.model import ConfigDisable, Site
from vergil_tooling.lib.exceptions.plugins import known_tools
from vergil_tooling.lib.exceptions.plugins._python import python_comments
from vergil_tooling.lib.exceptions.plugins.coverage import CoveragePlugin
from vergil_tooling.lib.exceptions.plugins.mypy import MypyPlugin
from vergil_tooling.lib.exceptions.plugins.ruff import RuffPlugin
from vergil_tooling.lib.exceptions.plugins.ty import TyPlugin
from vergil_tooling.lib.exceptions.tree import TreeReadError

SRC = (
    "import subprocess\n"
    "\n"
    "def f(cmd: list[str]) -> None:\n"
    "    subprocess.run(cmd)  # noqa: S603, S607  # vrg-exc: EXC-014\n"
    "    x = 1  # noqa\n"
    '    s = "# noqa: S603"\n'
    "    # vrg-exc: EXC-015\n"
    "    y: str = 1  # type: ignore[assignment]\n"
    "    z = 2  # type: ignore\n"
    "    w = 3  # ty: ignore[invalid-assignment]  # vrg-exc: EXC-016\n"
    "    if x:  # pragma: no cover  # vrg-exc: EXC-017\n"
    "        pass\n"
)


def test_python_comments_are_tokens_not_strings() -> None:
    comments = python_comments(FakeTree({"src/a.py": SRC, "README.md": "# noqa"}))
    texts = [c.text for c in comments]
    assert all("s = " not in t for t in texts)
    assert comments[0].path == "src/a.py"
    assert comments[0].line == 4


def test_noqa_inside_string_is_not_a_site() -> None:
    sites = RuffPlugin().sites(FakeTree({"src/a.py": SRC}))
    assert [s.line for s in sites] == [4, 5]


def test_ruff_sites() -> None:
    sites = RuffPlugin().sites(FakeTree({"src/a.py": SRC}))
    assert sites[0] == Site("ruff", "src/a.py", 4, ("S603", "S607"), "EXC-014")
    assert sites[1] == Site("ruff", "src/a.py", 5, (), None)   # blanket


def test_noqa_codes_stop_before_prose() -> None:
    sites = RuffPlugin().sites(FakeTree({"a.py": "x = 1  # noqa: S603  vrg-exc: EXC-001\n"}))
    assert sites == [Site("ruff", "a.py", 1, ("S603",), None)]   # trailing-text form is not a marker


def test_ruff_file_level_noqa() -> None:
    sites = RuffPlugin().sites(FakeTree({"a.py": "# ruff: noqa: E501  # vrg-exc: EXC-001\nx = 1\n"}))
    assert sites == [Site("ruff", "a.py", 1, ("E501",), "EXC-001")]


def test_unparseable_python_is_error() -> None:
    with pytest.raises(TreeReadError, match="bad.py"):
        RuffPlugin().sites(FakeTree({"bad.py": 'x = """\n'}))


def test_ruff_config_disables() -> None:
    pyproject = (
        "[tool.ruff.lint]\n"
        'select = ["E", "S"]\n'
        'ignore = ["E501"]\n'
        'extend-ignore = ["S311"]\n'
        "[tool.ruff.lint.per-file-ignores]\n"
        '"tests/**" = ["S101", "ARG"]\n'
    )
    disables = RuffPlugin().config_disables(FakeTree({"pyproject.toml": pyproject}))
    assert ConfigDisable("ruff", "E501", ("**",), "pyproject.toml [tool.ruff.lint] ignore") in disables
    assert ConfigDisable("ruff", "S311", ("**",), "pyproject.toml [tool.ruff.lint] extend-ignore") in disables
    assert ConfigDisable("ruff", "S101", ("tests/**",), "pyproject.toml [tool.ruff.lint] per-file-ignores") in disables
    assert ConfigDisable("ruff", "ARG", ("tests/**",), "pyproject.toml [tool.ruff.lint] per-file-ignores") in disables


def test_ruff_reads_ruff_toml() -> None:
    disables = RuffPlugin().config_disables(FakeTree({"ruff.toml": '[lint]\nignore = ["E501"]\n'}))
    assert disables == [ConfigDisable("ruff", "E501", ("**",), "ruff.toml [lint] ignore")]


def test_malformed_pyproject_is_error() -> None:
    with pytest.raises(TreeReadError, match="pyproject.toml"):
        RuffPlugin().config_disables(FakeTree({"pyproject.toml": "[tool.ruff\n"}))


def test_ruff_dead_sites_uses_ruf100(tmp_path: Path) -> None:
    site = Site("ruff", "src/a.py", 4, ("PLR2004",), "EXC-014")
    other = Site("ruff", "src/a.py", 9, ("S603",), "EXC-015")
    report = [{"code": "RUF100", "filename": str(tmp_path / "src/a.py"), "location": {"row": 4}},
              {"code": "S603", "filename": str(tmp_path / "src/a.py"), "location": {"row": 9}}]
    done = subprocess.CompletedProcess(args=[], returncode=0, stdout=json.dumps(report), stderr="")
    with mock.patch("vergil_tooling.lib.exceptions.plugins.ruff.subprocess.run", return_value=done) as run:
        assert RuffPlugin().dead_sites(tmp_path, [site, other]) == [site]
    argv = run.call_args.args[0]
    assert argv[:2] == ["ruff", "check"]
    assert "--extend-select=RUF100" in argv
    assert "--exit-zero" in argv


def test_ruff_dead_sites_failure_is_error(tmp_path: Path) -> None:
    bad = subprocess.CompletedProcess(args=[], returncode=2, stdout="", stderr="boom")
    with (mock.patch("vergil_tooling.lib.exceptions.plugins.ruff.subprocess.run", return_value=bad),
          pytest.raises(TreeReadError, match="ruff check failed"),):
        RuffPlugin().dead_sites(tmp_path, [])


def test_mypy_sites_and_blanket() -> None:
    sites = MypyPlugin().sites(FakeTree({"src/a.py": SRC}))
    assert sites == [
        Site("mypy", "src/a.py", 8, ("assignment",), "EXC-015"),   # marker on the line above
        Site("mypy", "src/a.py", 9, (), None),
    ]


def test_mypy_config_disables() -> None:
    pyproject = (
        "[tool.mypy]\n"
        'disable_error_code = ["import-untyped"]\n'
        "[[tool.mypy.overrides]]\n"
        'module = ["yaml.*"]\n'
        "ignore_missing_imports = true\n"
        "[[tool.mypy.overrides]]\n"
        'module = "legacy"\n'
        "ignore_errors = true\n"
    )
    disables = MypyPlugin().config_disables(FakeTree({"pyproject.toml": pyproject}))
    assert disables == [
        ConfigDisable("mypy", "import-untyped", ("**",), "pyproject.toml [tool.mypy] disable_error_code"),
        ConfigDisable("mypy", "ignore_missing_imports", ("module:yaml.*",), "pyproject.toml [[tool.mypy.overrides]]"),
        ConfigDisable("mypy", "ignore_errors", ("module:legacy",), "pyproject.toml [[tool.mypy.overrides]]"),
    ]


def test_ty_sites_and_config() -> None:
    tree = FakeTree({"src/a.py": SRC, "pyproject.toml": '[tool.ty.rules]\npossibly-unbound = "ignore"\n'})
    assert TyPlugin().sites(tree) == [Site("ty", "src/a.py", 10, ("invalid-assignment",), "EXC-016")]
    assert TyPlugin().config_disables(tree) == [
        ConfigDisable("ty", "possibly-unbound", ("**",), "pyproject.toml [tool.ty.rules]")]


def test_coverage_sites_and_config() -> None:
    pyproject = (
        "[tool.coverage.run]\n"
        'omit = ["src/gen/*"]\n'
        "[tool.coverage.report]\n"
        'exclude_lines = ["pragma: no cover", "if TYPE_CHECKING:"]\n'
    )
    tree = FakeTree({"src/a.py": SRC, "pyproject.toml": pyproject})
    assert CoveragePlugin().sites(tree) == [Site("coverage", "src/a.py", 11, ("no-cover",), "EXC-017")]
    assert CoveragePlugin().config_disables(tree) == [
        ConfigDisable("coverage", "omit", ("src/gen/*",), "pyproject.toml [tool.coverage.run] omit"),
        ConfigDisable("coverage", "if TYPE_CHECKING:", ("**",), "pyproject.toml [tool.coverage.report] exclude_lines"),
    ]


def test_detect_requires_python_files() -> None:
    assert RuffPlugin().detect(FakeTree({"a.py": ""}))
    assert not RuffPlugin().detect(FakeTree({"a.sh": ""}))


def test_known_tools_include_python_plugins() -> None:
    assert {"ruff", "mypy", "ty", "coverage"} <= known_tools()
```

- [ ] **Step 2: Run to verify failure**

Run: `vrg-container-run -- uv run pytest tests/vergil_tooling/test_exceptions_plugins_python.py -q --no-cov`
Expected: FAIL — `plugins` package missing.

- [ ] **Step 3: Implement the shared helpers**

```python
# src/vergil_tooling/lib/exceptions/plugins/_python.py
"""Real comments in Python source, found with ``tokenize`` (never by line regex)."""

from __future__ import annotations

import io
import tokenize
from dataclasses import dataclass
from typing import TYPE_CHECKING

from vergil_tooling.lib.exceptions.tree import TreeReadError

if TYPE_CHECKING:
    from vergil_tooling.lib.exceptions.tree import RepoTree


@dataclass(frozen=True)
class Comment:
    path: str
    line: int
    text: str
    previous: str | None


def python_files(tree: RepoTree) -> list[str]:
    return [p for p in tree.files() if p.endswith(".py")]


def python_comments(tree: RepoTree) -> list[Comment]:
    comments: list[Comment] = []
    for path in python_files(tree):
        text = tree.read(path)
        if text is None:
            continue
        lines = text.splitlines()
        try:
            tokens = list(tokenize.generate_tokens(io.StringIO(text).readline))
        except (tokenize.TokenError, SyntaxError) as exc:
            msg = f"{path}: cannot tokenize Python source ({exc}); fix the syntax error first"
            raise TreeReadError(msg) from exc
        for tok in tokens:
            if tok.type == tokenize.COMMENT:
                row = tok.start[0]
                previous = lines[row - 2] if row > 1 else None
                comments.append(Comment(path=path, line=row, text=lines[row - 1], previous=previous))
    return comments
```

`Comment.text` is the whole physical line, so `marker_id(c.text, c.previous)`
sees a same-line `# vrg-exc:` that follows the tool's directive.

```python
# src/vergil_tooling/lib/exceptions/plugins/_config.py
"""Read TOML config files from a ``RepoTree``; malformed config is an error."""

from __future__ import annotations

import tomllib
from typing import TYPE_CHECKING, Any

from vergil_tooling.lib.exceptions.tree import TreeReadError

if TYPE_CHECKING:
    from vergil_tooling.lib.exceptions.tree import RepoTree


def read_toml(tree: RepoTree, path: str) -> dict[str, Any] | None:
    text = tree.read(path)
    if text is None:
        return None
    try:
        return tomllib.loads(text)
    except tomllib.TOMLDecodeError as exc:
        msg = f"{path}: invalid TOML ({exc})"
        raise TreeReadError(msg) from exc


def table(data: dict[str, Any] | None, *keys: str) -> dict[str, Any]:
    """Walk nested tables; a missing level yields ``{}``."""
    node: Any = data or {}
    for key in keys:
        node = node.get(key, {}) if isinstance(node, dict) else {}
    return node if isinstance(node, dict) else {}


def str_list(value: object) -> list[str]:
    if isinstance(value, str):
        return [value]
    if isinstance(value, list):
        return [v for v in value if isinstance(v, str)]
    return []
```

- [ ] **Step 4: Implement the four plug-ins**

```python
# src/vergil_tooling/lib/exceptions/plugins/ruff.py
"""ruff: ``# noqa``, ``# ruff: noqa``, and ignore/per-file-ignores config."""

from __future__ import annotations

import json
import re
import subprocess
from pathlib import Path
from typing import TYPE_CHECKING

from vergil_tooling.lib.exceptions.markers import marker_id
from vergil_tooling.lib.exceptions.model import ConfigDisable, Site
from vergil_tooling.lib.exceptions.plugin import Plugin
from vergil_tooling.lib.exceptions.plugins._config import read_toml, str_list, table
from vergil_tooling.lib.exceptions.plugins._python import python_comments, python_files
from vergil_tooling.lib.exceptions.tree import TreeReadError

if TYPE_CHECKING:
    from vergil_tooling.lib.exceptions.tree import RepoTree

# Codes are case-sensitive and always carry digits, so trailing prose such as
# "vrg-exc" can never be mistaken for a code.
_NOQA_RE = re.compile(r"#\s*(?:ruff:\s*)?(?i:noqa)(?::\s*(?P<codes>[A-Z]+[0-9]+(?:[\s,]+[A-Z]+[0-9]+)*))?")
_CONFIG_FILES = (("pyproject.toml", ("tool", "ruff")), ("ruff.toml", ()), (".ruff.toml", ()))


class RuffPlugin(Plugin):
    name = "ruff"

    def detect(self, tree: RepoTree) -> bool:
        return bool(python_files(tree))

    def sites(self, tree: RepoTree) -> list[Site]:
        sites: list[Site] = []
        for c in python_comments(tree):
            match = _NOQA_RE.search(c.text)
            if match is None:
                continue
            codes = tuple(re.split(r"[\s,]+", match.group("codes").strip())) if match.group("codes") else ()
            sites.append(Site(self.name, c.path, c.line, codes, marker_id(c.text, c.previous)))
        return sites

    def config_disables(self, tree: RepoTree) -> list[ConfigDisable]:
        out: list[ConfigDisable] = []
        for path, prefix in _CONFIG_FILES:
            data = read_toml(tree, path)
            if data is None:
                continue
            base = table(data, *prefix)
            lint = table(base, "lint")
            where = f"{path} [{'.'.join((*prefix, 'lint'))}]"
            for key in ("ignore", "extend-ignore"):
                for rule in str_list(lint.get(key, base.get(key))):
                    out.append(ConfigDisable(self.name, rule, ("**",), f"{where} {key}"))
            per_file = table(lint, "per-file-ignores") or table(base, "per-file-ignores")
            for glob, rules in per_file.items():
                for rule in str_list(rules):
                    out.append(ConfigDisable(self.name, rule, (glob,), f"{where} per-file-ignores"))
        return out

    def dead_sites(self, root: Path, sites: list[Site]) -> list[Site]:
        result = subprocess.run(  # noqa: S603
            ["ruff", "check", "--extend-select=RUF100", "--output-format=json", "--no-fix", "--exit-zero", "."],  # noqa: S607
            cwd=root,
            capture_output=True,
            text=True,
            check=False,
        )
        if result.returncode != 0:
            msg = f"ruff check failed while detecting dead noqa (exit {result.returncode}): {result.stderr.strip()}"
            raise TreeReadError(msg)
        dead_at = {
            (Path(item["filename"]).resolve().relative_to(root.resolve()).as_posix(), item["location"]["row"])
            for item in json.loads(result.stdout or "[]")
            if item.get("code") == "RUF100"
        }
        return [s for s in sites if (s.path, s.line) in dead_at]
```

```python
# src/vergil_tooling/lib/exceptions/plugins/mypy.py
"""mypy: ``# type: ignore[...]`` and disable_error_code / overrides config.

Dead ``type: ignore`` comments are already a mypy error under ``strict``
(``warn_unused_ignores``), which the TYPECHECK stage runs, so this plug-in does
not re-run mypy for dead detection.
"""

from __future__ import annotations

import re
from typing import TYPE_CHECKING

from vergil_tooling.lib.exceptions.markers import marker_id
from vergil_tooling.lib.exceptions.model import ConfigDisable, Site
from vergil_tooling.lib.exceptions.plugin import Plugin
from vergil_tooling.lib.exceptions.plugins._config import read_toml, str_list, table
from vergil_tooling.lib.exceptions.plugins._python import python_comments, python_files

if TYPE_CHECKING:
    from vergil_tooling.lib.exceptions.tree import RepoTree

_IGNORE_RE = re.compile(r"#\s*type:\s*ignore(?:\[(?P<codes>[^\]]*)\])?")
_OVERRIDE_FLAGS = ("ignore_missing_imports", "ignore_errors")


class MypyPlugin(Plugin):
    name = "mypy"

    def detect(self, tree: RepoTree) -> bool:
        return bool(python_files(tree))

    def sites(self, tree: RepoTree) -> list[Site]:
        sites: list[Site] = []
        for c in python_comments(tree):
            match = _IGNORE_RE.search(c.text)
            if match is None:
                continue
            raw = match.group("codes")
            codes = tuple(code.strip() for code in raw.split(",") if code.strip()) if raw else ()
            sites.append(Site(self.name, c.path, c.line, codes, marker_id(c.text, c.previous)))
        return sites

    def config_disables(self, tree: RepoTree) -> list[ConfigDisable]:
        data = read_toml(tree, "pyproject.toml")
        mypy = table(data, "tool", "mypy")
        out = [
            ConfigDisable(self.name, code, ("**",), "pyproject.toml [tool.mypy] disable_error_code")
            for code in str_list(mypy.get("disable_error_code"))
        ]
        overrides = mypy.get("overrides", [])
        for override in overrides if isinstance(overrides, list) else []:
            modules = tuple(f"module:{m}" for m in str_list(override.get("module")))
            for code in str_list(override.get("disable_error_code")):
                out.append(ConfigDisable(self.name, code, modules, "pyproject.toml [[tool.mypy.overrides]]"))
            for flag in _OVERRIDE_FLAGS:
                if override.get(flag) is True:
                    out.append(ConfigDisable(self.name, flag, modules, "pyproject.toml [[tool.mypy.overrides]]"))
        return out
```

```python
# src/vergil_tooling/lib/exceptions/plugins/ty.py
"""ty: ``# ty: ignore[...]`` and ``[tool.ty.rules] <rule> = "ignore"``."""

from __future__ import annotations

import re
from typing import TYPE_CHECKING

from vergil_tooling.lib.exceptions.markers import marker_id
from vergil_tooling.lib.exceptions.model import ConfigDisable, Site
from vergil_tooling.lib.exceptions.plugin import Plugin
from vergil_tooling.lib.exceptions.plugins._config import read_toml, table
from vergil_tooling.lib.exceptions.plugins._python import python_comments, python_files

if TYPE_CHECKING:
    from vergil_tooling.lib.exceptions.tree import RepoTree

_IGNORE_RE = re.compile(r"#\s*ty:\s*ignore(?:\[(?P<codes>[^\]]*)\])?")


class TyPlugin(Plugin):
    name = "ty"

    def detect(self, tree: RepoTree) -> bool:
        return bool(python_files(tree))

    def sites(self, tree: RepoTree) -> list[Site]:
        sites: list[Site] = []
        for c in python_comments(tree):
            match = _IGNORE_RE.search(c.text)
            if match is None:
                continue
            raw = match.group("codes")
            codes = tuple(code.strip() for code in raw.split(",") if code.strip()) if raw else ()
            sites.append(Site(self.name, c.path, c.line, codes, marker_id(c.text, c.previous)))
        return sites

    def config_disables(self, tree: RepoTree) -> list[ConfigDisable]:
        rules = table(read_toml(tree, "pyproject.toml"), "tool", "ty", "rules")
        return [
            ConfigDisable(self.name, rule, ("**",), "pyproject.toml [tool.ty.rules]")
            for rule, level in rules.items()
            if level == "ignore"
        ]
```

```python
# src/vergil_tooling/lib/exceptions/plugins/coverage.py
"""coverage.py: ``# pragma: no cover`` / ``no branch``, ``omit`` and ``exclude_lines``.

A bare pragma names no rule, so its rule ID is ``no-cover`` / ``no-branch``
(spec §7). The default ``pragma: no cover`` exclude pattern is the inline
mechanism itself, not an exception.
"""

from __future__ import annotations

import re
from typing import TYPE_CHECKING

from vergil_tooling.lib.exceptions.markers import marker_id
from vergil_tooling.lib.exceptions.model import ConfigDisable, Site
from vergil_tooling.lib.exceptions.plugin import Plugin
from vergil_tooling.lib.exceptions.plugins._config import read_toml, str_list, table
from vergil_tooling.lib.exceptions.plugins._python import python_comments, python_files

if TYPE_CHECKING:
    from vergil_tooling.lib.exceptions.tree import RepoTree

_PRAGMA_RE = re.compile(r"#\s*pragma:\s*no\s+(?P<kind>cover|branch)")
_MECHANISM_PATTERNS = frozenset({"pragma: no cover", "pragma: no branch", r"#\s*(pragma|PRAGMA)[:\s]?\s*(no|NO)\s*(cover|COVER)"})


class CoveragePlugin(Plugin):
    name = "coverage"

    def detect(self, tree: RepoTree) -> bool:
        return bool(python_files(tree))

    def sites(self, tree: RepoTree) -> list[Site]:
        sites: list[Site] = []
        for c in python_comments(tree):
            match = _PRAGMA_RE.search(c.text)
            if match is None:
                continue
            rule = f"no-{match.group('kind')}"
            sites.append(Site(self.name, c.path, c.line, (rule,), marker_id(c.text, c.previous)))
        return sites

    def config_disables(self, tree: RepoTree) -> list[ConfigDisable]:
        data = read_toml(tree, "pyproject.toml")
        out = [
            ConfigDisable(self.name, "omit", (glob,), "pyproject.toml [tool.coverage.run] omit")
            for glob in str_list(table(data, "tool", "coverage", "run").get("omit"))
        ]
        report = table(data, "tool", "coverage", "report")
        for key in ("exclude_lines", "exclude_also"):
            for pattern in str_list(report.get(key)):
                if pattern not in _MECHANISM_PATTERNS:
                    out.append(ConfigDisable(self.name, pattern, ("**",), f"pyproject.toml [tool.coverage.report] {key}"))
        return out
```

```python
# src/vergil_tooling/lib/exceptions/plugins/__init__.py
"""Registry of exception-mechanism plug-ins (spec §7)."""

from __future__ import annotations

from vergil_tooling.lib.exceptions.plugin import Plugin, UncoveredDetector
from vergil_tooling.lib.exceptions.plugins.coverage import CoveragePlugin
from vergil_tooling.lib.exceptions.plugins.mypy import MypyPlugin
from vergil_tooling.lib.exceptions.plugins.ruff import RuffPlugin
from vergil_tooling.lib.exceptions.plugins.ty import TyPlugin

PLUGINS: tuple[Plugin, ...] = (RuffPlugin(), MypyPlugin(), TyPlugin(), CoveragePlugin())
UNCOVERED: tuple[UncoveredDetector, ...] = ()


def known_tools() -> frozenset[str]:
    """Every ``tool`` value a register entry may use."""
    return frozenset(p.name for p in PLUGINS)
```

- [ ] **Step 5: Run the tests to verify they pass**

Run: `vrg-container-run -- uv run pytest tests/vergil_tooling/test_exceptions_plugins_python.py -q --no-cov`
Expected: PASS.

- [ ] **Step 6: Smoke-check against vergil-tooling itself**

Run: `vrg-container-run -- uv run python -c "from pathlib import Path; from vergil_tooling.lib.exceptions.tree import WorkTree; from vergil_tooling.lib.exceptions.plugins import PLUGINS; t=WorkTree(Path('.')); print({p.name: len(p.sites(t)) for p in PLUGINS})"`
Expected: roughly `{'ruff': ~480, 'mypy': 9, 'ty': 0, 'coverage': 61}` — matching the
spec §2 counts (exact numbers drift with develop; order of magnitude must match).
A large mismatch means comments are being missed or string text is being counted.

- [ ] **Step 7: Full validation and commit**

Run: `vrg-container-run -- vrg-validate` — Expected: PASS, coverage 100%.

```bash
vrg-git add src/vergil_tooling/lib/exceptions/plugins tests/vergil_tooling/test_exceptions_plugins_python.py
vrg-commit --type feat --scope exceptions --message "add ruff, mypy, ty and coverage exception plug-ins" --body "Ref #<task>"
```

---

## Task T3: common-check and security plug-ins, plus uncovered-mechanism detection

**Repo:** vergil-tooling. Blocked by: T2 (registry). Spec: §5.2, §6.2, §7.

**Files:**

- Create: `src/vergil_tooling/lib/exceptions/plugins/_text.py` (line scanner shared by non-Python tools)
- Create: `plugins/markdownlint.py`, `shellcheck.py`, `yamllint.py`, `hadolint.py`,
  `actionlint.py`, `semgrep.py`, `trivy.py`, `uncovered.py` (all under
  `src/vergil_tooling/lib/exceptions/plugins/`)
- Modify: `src/vergil_tooling/lib/exceptions/plugins/__init__.py` (register them; fill `UNCOVERED`)
- Test: `tests/vergil_tooling/test_exceptions_plugins_text.py`

**Interfaces:**

- Consumes (T1/T2): `Plugin`, `Site`, `ConfigDisable`, `marker_id`, `read_toml`,
  `table`, `str_list`, `PLUGINS`, `UNCOVERED`, `TreeReadError`.
- Produces:

```python
# plugins/_text.py
@dataclass(frozen=True) class Line: path: str; number: int; text: str; previous: str | None
def lines_of(tree: RepoTree, paths: Iterable[str], *, skip_md_fences: bool = False) -> Iterator[Line]
def shell_scripts(tree: RepoTree) -> list[str]   # *.sh, *.bash, and extensionless files with a sh/bash shebang

# plugins/uncovered.py
@dataclass(frozen=True) class RegexDetector: name: str; suffixes: tuple[str, ...]; pattern: re.Pattern[str]; def detect(self, tree) -> bool
DETECTORS: tuple[RegexDetector, ...]
```

Plug-in names: `markdownlint`, `shellcheck`, `yamllint`, `hadolint`,
`actionlint`, `semgrep`, `trivy`.

Behaviour decisions this task makes concrete (spec §5.2 and §7 leave them to
planning):

- **Comment detection for non-Python files** is line-based: a directive counts
  only when its comment marker starts the line or follows whitespace
  (`(^|\s)#`). Markdown directives inside fenced code blocks (```` ``` ```` or
  `~~~`) are skipped, so docs and specs that *show* a directive are not sites.
- **Markdown markers** use `<!-- vrg-exc: EXC-NNN -->` (T1 `MARKER_RE` accepts it).
- **`.trivyignore` grouping:** a comment-only `# vrg-exc: EXC-NNN` line applies to
  every following non-comment, non-blank entry until the next blank line. This
  keeps vergil-containers' grouped-by-carrier layout (spec §5.2 "line above"),
  without a marker per CVE. The entry's `rules` must still list every CVE in the
  group, so adding a CVE to a group still edits the register.
- **`nosemgrep` markers** go on the line above (spec §5.2): semgrep reads the
  rest of a `nosemgrep:` comment as its rule-ID list.
- **semgrep `--exclude-rule`** flags in `.github/workflows/*.yml` and shell
  scripts are config disables with scope `["**"]` (spec §7). The vergil-actions
  semgrep action has no exclusion input, so a direct flag is the only per-repo
  route; no repo uses one today. The fleet's `DEFAULT_EXCLUDED_RULES` are not
  scanned here — they are fleet defaults (T4).
- **Shellcheck in workflows:** `# shellcheck disable=` lines inside
  `.github/workflows/*.yml` `run:` blocks are scanned too (actionlint runs
  shellcheck over them).
- **Path exclusions** (`vergil.toml` `[markdownlint].ignore`, `.semgrepignore`)
  are config disables with rule `*` and the excluded path as scope.

- [ ] **Step 1: Write the failing tests**

```python
# tests/vergil_tooling/test_exceptions_plugins_text.py
from tests.vergil_tooling.exceptions_fakes import FakeTree
from vergil_tooling.lib.exceptions.model import ConfigDisable, Site
from vergil_tooling.lib.exceptions.plugins import PLUGINS, UNCOVERED, known_tools
from vergil_tooling.lib.exceptions.plugins._text import shell_scripts
from vergil_tooling.lib.exceptions.plugins.actionlint import ActionlintPlugin
from vergil_tooling.lib.exceptions.plugins.hadolint import HadolintPlugin
from vergil_tooling.lib.exceptions.plugins.markdownlint import MarkdownlintPlugin
from vergil_tooling.lib.exceptions.plugins.semgrep import SemgrepPlugin
from vergil_tooling.lib.exceptions.plugins.shellcheck import ShellcheckPlugin
from vergil_tooling.lib.exceptions.plugins.trivy import TrivyPlugin
from vergil_tooling.lib.exceptions.plugins.yamllint import YamllintPlugin

MD = (
    "# Title\n"
    "<!-- markdownlint-disable-next-line MD013 --> <!-- vrg-exc: EXC-005 -->\n"
    "long line\n"
    "```markdown\n"
    "<!-- markdownlint-disable MD033 -->\n"
    "```\n"
    "<!-- markdownlint-disable -->\n"
)


def test_markdownlint_sites_skip_fences() -> None:
    assert MarkdownlintPlugin().sites(FakeTree({"docs/a.md": MD})) == [
        Site("markdownlint", "docs/a.md", 2, ("MD013",), "EXC-005"),
        Site("markdownlint", "docs/a.md", 7, (), None),
    ]


def test_markdownlint_vergil_toml_ignore_is_disable() -> None:
    tree = FakeTree({"vergil.toml": '[markdownlint]\nignore = ["docs/legacy"]\n', "a.md": ""})
    assert MarkdownlintPlugin().config_disables(tree) == [
        ConfigDisable("markdownlint", "*", ("docs/legacy",), "vergil.toml [markdownlint] ignore")]


def test_shell_scripts_detected_by_extension_and_shebang() -> None:
    tree = FakeTree({"scripts/a.sh": "", "scripts/run": "#!/usr/bin/env bash\n", "bin/x": "#!/usr/bin/python3\n"})
    assert shell_scripts(tree) == ["scripts/a.sh", "scripts/run"]


def test_shellcheck_sites_including_workflows() -> None:
    tree = FakeTree({
        "scripts/a.sh": '# shellcheck disable=SC2086,SC2034  # vrg-exc: EXC-020\necho $x\nx="# shellcheck disable=SC1"\n',
        ".github/workflows/ci.yml": "jobs:\n  a:\n    steps:\n      - run: |\n          # shellcheck disable=SC2046\n",
        ".shellcheckrc": "disable=SC1091\n",
    })
    plugin = ShellcheckPlugin()
    assert plugin.sites(tree) == [
        Site("shellcheck", ".github/workflows/ci.yml", 5, ("SC2046",), None),
        Site("shellcheck", "scripts/a.sh", 1, ("SC2086", "SC2034"), "EXC-020"),
    ]
    assert plugin.config_disables(tree) == [ConfigDisable("shellcheck", "SC1091", ("**",), ".shellcheckrc")]


def test_yamllint_sites() -> None:
    tree = FakeTree({"a.yml": "a: 1  # yamllint disable-line rule:line-length  # vrg-exc: EXC-030\n# yamllint disable\n"})
    assert YamllintPlugin().sites(tree) == [
        Site("yamllint", "a.yml", 1, ("line-length",), "EXC-030"),
        Site("yamllint", "a.yml", 2, (), None),
    ]


def test_hadolint_sites_and_config() -> None:
    tree = FakeTree({"Dockerfile": "# hadolint ignore=DL3008,DL3009\nRUN apt-get install x\n",
                     ".hadolint.yaml": "ignored:\n  - DL3006\n"})
    assert HadolintPlugin().sites(tree) == [Site("hadolint", "Dockerfile", 1, ("DL3008", "DL3009"), None)]
    assert HadolintPlugin().config_disables(tree) == [ConfigDisable("hadolint", "DL3006", ("**",), ".hadolint.yaml ignored")]


def test_actionlint_config_ignores() -> None:
    cfg = "paths:\n  .github/workflows/release.yml:\n    ignore:\n      - 'shellcheck reported issue.+SC2129'\n"
    tree = FakeTree({".github/actionlint.yaml": cfg, ".github/workflows/release.yml": ""})
    assert ActionlintPlugin().config_disables(tree) == [
        ConfigDisable("actionlint", "shellcheck reported issue.+SC2129", (".github/workflows/release.yml",),
                      ".github/actionlint.yaml paths.ignore")]


def test_semgrep_sites_any_language_and_semgrepignore() -> None:
    tree = FakeTree({
        ".github/workflows/cd.yml": ("    # vrg-exc: EXC-001\n"
                                     "    secrets: inherit  # nosemgrep: yaml.r.secrets-inherit\n"),
        "web/a.ts": "eval(x) // nosemgrep\n",
        ".semgrepignore": "# comment\nvendor/\n",
    })
    plugin = SemgrepPlugin()
    assert plugin.sites(tree) == [
        Site("semgrep", ".github/workflows/cd.yml", 2, ("yaml.r.secrets-inherit",), "EXC-001"),
        Site("semgrep", "web/a.ts", 1, (), None),
    ]
    assert plugin.config_disables(tree) == [ConfigDisable("semgrep", "*", ("vendor/",), ".semgrepignore")]


def test_semgrep_exclude_rule_flags_are_config_disables() -> None:
    # spec §7: a per-repo --exclude-rule disables the rule repo-wide, so it needs a [[rule]] entry.
    tree = FakeTree({
        ".github/workflows/scan.yml": "      - run: vrg-semgrep-scan --exclude-rule a.b.c --exclude-rule=d.e\n",
        "scripts/scan.sh": "semgrep scan \\\n  --exclude-rule f.g\n",
        "docs/x.md": "pass `--exclude-rule z` to drop a rule\n",
    })
    assert SemgrepPlugin().config_disables(tree) == [
        ConfigDisable("semgrep", "a.b.c", ("**",), ".github/workflows/scan.yml:1 --exclude-rule"),
        ConfigDisable("semgrep", "d.e", ("**",), ".github/workflows/scan.yml:1 --exclude-rule"),
        ConfigDisable("semgrep", "f.g", ("**",), "scripts/scan.sh:2 --exclude-rule"),
    ]


def test_trivyignore_groups_share_the_marker_above() -> None:
    text = (
        "# Go asn1 carrier — not reachable\n"
        "# vrg-exc: EXC-031\n"
        "CVE-2026-1\n"
        "CVE-2026-2 exp:2027-01-01\n"
        "\n"
        "CVE-2026-3\n"
    )
    assert TrivyPlugin().sites(FakeTree({".trivyignore": text})) == [
        Site("trivy", ".trivyignore", 3, ("CVE-2026-1",), "EXC-031"),
        Site("trivy", ".trivyignore", 4, ("CVE-2026-2",), "EXC-031"),
        Site("trivy", ".trivyignore", 6, ("CVE-2026-3",), None),
    ]


def test_trivy_config_disables() -> None:
    tree = FakeTree({"trivy.yaml": "vulnerability:\n  ignore-unfixed: true\n", ".trivy/ignore-policy.rego": "package x\n",
                     ".trivyignore": ""})
    assert TrivyPlugin().config_disables(tree) == [
        ConfigDisable("trivy", "ignore-unfixed", ("**",), "trivy.yaml vulnerability.ignore-unfixed"),
        ConfigDisable("trivy", "rego-policy", (".trivy/ignore-policy.rego",), ".trivy/ignore-policy.rego"),
    ]


def test_uncovered_detectors() -> None:
    tree = FakeTree({"web/a.ts": "// eslint-disable-next-line no-console\n", "src/x.cpp": "int a; // NOLINT\n",
                     ".ansible-lint": "skip_list:\n  - yaml\n"})
    names = sorted(d.name for d in UNCOVERED if d.detect(tree))
    assert names == ["ansible-lint", "clang-tidy", "eslint"]


def test_registry_and_known_tools() -> None:
    names = [p.name for p in PLUGINS]
    assert names == ["ruff", "mypy", "ty", "coverage", "markdownlint", "shellcheck", "yamllint",
                     "hadolint", "actionlint", "semgrep", "trivy"]
    assert set(names) == known_tools()
```

- [ ] **Step 2: Run to verify failure**

Run: `vrg-container-run -- uv run pytest tests/vergil_tooling/test_exceptions_plugins_text.py -q --no-cov`
Expected: FAIL — modules missing.

- [ ] **Step 3: Implement the line scanner**

```python
# src/vergil_tooling/lib/exceptions/plugins/_text.py
"""Line-based scanning for tools whose files are not Python."""

from __future__ import annotations

import re
from dataclasses import dataclass
from typing import TYPE_CHECKING

if TYPE_CHECKING:
    from collections.abc import Iterable, Iterator

    from vergil_tooling.lib.exceptions.tree import RepoTree

_FENCE_RE = re.compile(r"^\s*(```|~~~)")
_SHEBANG_RE = re.compile(r"^#!.*\b(ba)?sh\b")


@dataclass(frozen=True)
class Line:
    path: str
    number: int
    text: str
    previous: str | None


def lines_of(tree: RepoTree, paths: Iterable[str], *, skip_md_fences: bool = False) -> Iterator[Line]:
    for path in paths:
        text = tree.read(path)
        if text is None:
            continue
        previous: str | None = None
        in_fence = False
        for number, line in enumerate(text.splitlines(), start=1):
            if skip_md_fences and _FENCE_RE.match(line):
                in_fence = not in_fence
                previous = line
                continue
            if not in_fence:
                yield Line(path, number, line, previous)
            previous = line


def by_suffix(tree: RepoTree, *suffixes: str) -> list[str]:
    return [p for p in tree.files() if p.endswith(suffixes)]


def shell_scripts(tree: RepoTree) -> list[str]:
    out: list[str] = []
    for path in tree.files():
        if path.endswith((".sh", ".bash")):
            out.append(path)
        elif "." not in path.rsplit("/", 1)[-1]:
            head = (tree.read(path) or "").split("\n", 1)[0]
            if _SHEBANG_RE.match(head):
                out.append(path)
    return out


def split_rules(raw: str | None, sep: str = r"[\s,]+") -> tuple[str, ...]:
    if not raw:
        return ()
    return tuple(r for r in re.split(sep, raw.strip()) if r)
```

- [ ] **Step 4: Implement the plug-ins**

In every plug-in module here and in T2, add `from typing import override` and
decorate each `detect`/`sites`/`config_disables`/`dead_sites` implementation with
`@override` (the listings below show it only where an argument is unused, which
is where it matters for ruff `ARG002`).

```python
# src/vergil_tooling/lib/exceptions/plugins/markdownlint.py
"""markdownlint: ``<!-- markdownlint-disable* -->`` and ``vergil.toml`` path ignores.

Rule disables in the shipped ``markdownlint.yaml`` are fleet defaults (Task T4);
repos cannot override them because ``vrg-validate`` passes the bundled config.
"""

from __future__ import annotations

import re
from typing import TYPE_CHECKING

from vergil_tooling.lib.exceptions.markers import marker_id
from vergil_tooling.lib.exceptions.model import ConfigDisable, Site
from vergil_tooling.lib.exceptions.plugin import Plugin
from vergil_tooling.lib.exceptions.plugins._config import read_toml, str_list, table
from vergil_tooling.lib.exceptions.plugins._text import by_suffix, lines_of, split_rules

if TYPE_CHECKING:
    from vergil_tooling.lib.exceptions.tree import RepoTree

_DISABLE_RE = re.compile(r"<!--\s*markdownlint-disable(?:-next-line|-line|-file)?(?P<rules>(?:\s+[\w-]+)*)\s*-->")


class MarkdownlintPlugin(Plugin):
    name = "markdownlint"

    def detect(self, tree: RepoTree) -> bool:
        return bool(by_suffix(tree, ".md"))

    def sites(self, tree: RepoTree) -> list[Site]:
        sites: list[Site] = []
        for line in lines_of(tree, by_suffix(tree, ".md"), skip_md_fences=True):
            match = _DISABLE_RE.search(line.text)
            if match:
                rules = split_rules(match.group("rules"))
                sites.append(Site(self.name, line.path, line.number, rules, marker_id(line.text, line.previous)))
        return sites

    def config_disables(self, tree: RepoTree) -> list[ConfigDisable]:
        ignore = table(read_toml(tree, "vergil.toml"), "markdownlint").get("ignore")
        return [ConfigDisable(self.name, "*", (path,), "vergil.toml [markdownlint] ignore") for path in str_list(ignore)]
```

```python
# src/vergil_tooling/lib/exceptions/plugins/shellcheck.py
"""shellcheck: ``# shellcheck disable=`` (scripts and workflow run blocks), ``.shellcheckrc``."""

from __future__ import annotations

import re
from typing import TYPE_CHECKING

from vergil_tooling.lib.exceptions.markers import marker_id
from vergil_tooling.lib.exceptions.model import ConfigDisable, Site
from vergil_tooling.lib.exceptions.plugin import Plugin
from vergil_tooling.lib.exceptions.plugins._text import lines_of, shell_scripts, split_rules

if TYPE_CHECKING:
    from vergil_tooling.lib.exceptions.tree import RepoTree

_DISABLE_RE = re.compile(r"(?:^|\s)#\s*shellcheck\s+disable=(?P<rules>SC\d+(?:,SC\d+)*)")
_RC_RE = re.compile(r"^\s*disable=(?P<rules>SC\d+(?:,SC\d+)*)")


def _workflows(tree: RepoTree) -> list[str]:
    return [p for p in tree.files() if p.startswith(".github/workflows/") and p.endswith((".yml", ".yaml"))]


class ShellcheckPlugin(Plugin):
    name = "shellcheck"

    def detect(self, tree: RepoTree) -> bool:
        return bool(shell_scripts(tree) or _workflows(tree) or tree.read(".shellcheckrc") is not None)

    def sites(self, tree: RepoTree) -> list[Site]:
        sites: list[Site] = []
        for line in lines_of(tree, sorted(shell_scripts(tree) + _workflows(tree))):
            match = _DISABLE_RE.search(line.text)
            if match:
                rules = split_rules(match.group("rules"), sep=",")
                sites.append(Site(self.name, line.path, line.number, rules, marker_id(line.text, line.previous)))
        return sites

    def config_disables(self, tree: RepoTree) -> list[ConfigDisable]:
        out: list[ConfigDisable] = []
        for path in tree.files():
            if path.rsplit("/", 1)[-1] != ".shellcheckrc":
                continue
            directory = path.rsplit("/", 1)[0] if "/" in path else ""
            scope = (f"{directory}/**",) if directory else ("**",)
            for line in lines_of(tree, [path]):
                match = _RC_RE.match(line.text)
                if match:
                    out.extend(ConfigDisable(self.name, r, scope, path) for r in split_rules(match.group("rules"), sep=","))
        return out
```

```python
# src/vergil_tooling/lib/exceptions/plugins/yamllint.py
"""yamllint: ``# yamllint disable[-line] [rule:x ...]``.

Rule disables in the shipped ``yamllint.yaml`` are fleet defaults (Task T4).
"""

from __future__ import annotations

import re
from typing import TYPE_CHECKING

from vergil_tooling.lib.exceptions.markers import marker_id
from vergil_tooling.lib.exceptions.model import ConfigDisable, Site
from vergil_tooling.lib.exceptions.plugin import Plugin
from vergil_tooling.lib.exceptions.plugins._text import by_suffix, lines_of

if TYPE_CHECKING:
    from vergil_tooling.lib.exceptions.tree import RepoTree

_DISABLE_RE = re.compile(r"(?:^|\s)#\s*yamllint\s+disable(?:-line|-file)?(?P<rules>(?:\s+rule:[\w-]+)*)")


class YamllintPlugin(Plugin):
    name = "yamllint"

    def detect(self, tree: RepoTree) -> bool:
        return bool(by_suffix(tree, ".yml", ".yaml"))

    def sites(self, tree: RepoTree) -> list[Site]:
        sites: list[Site] = []
        for line in lines_of(tree, by_suffix(tree, ".yml", ".yaml")):
            match = _DISABLE_RE.search(line.text)
            if match:
                rules = tuple(r.removeprefix("rule:") for r in match.group("rules").split())
                sites.append(Site(self.name, line.path, line.number, rules, marker_id(line.text, line.previous)))
        return sites

    @override
    def config_disables(self, tree: RepoTree) -> list[ConfigDisable]:
        return []
```

```python
# src/vergil_tooling/lib/exceptions/plugins/hadolint.py
"""hadolint: ``# hadolint ignore=`` and ``.hadolint.yaml`` ``ignored``."""

from __future__ import annotations

import re
from typing import TYPE_CHECKING

import yaml

from vergil_tooling.lib.exceptions.markers import marker_id
from vergil_tooling.lib.exceptions.model import ConfigDisable, Site
from vergil_tooling.lib.exceptions.plugin import Plugin
from vergil_tooling.lib.exceptions.plugins._config import str_list
from vergil_tooling.lib.exceptions.plugins._text import lines_of, split_rules
from vergil_tooling.lib.exceptions.tree import TreeReadError

if TYPE_CHECKING:
    from vergil_tooling.lib.exceptions.tree import RepoTree

_IGNORE_RE = re.compile(r"^\s*#\s*hadolint\s+ignore=(?P<rules>DL\d+(?:,DL\d+)*)")
_CONFIGS = (".hadolint.yaml", ".hadolint.yml")


def _dockerfiles(tree: RepoTree) -> list[str]:
    return [p for p in tree.files() if p.rsplit("/", 1)[-1].startswith("Dockerfile")]


def read_yaml(tree: RepoTree, path: str) -> object:
    text = tree.read(path)
    if text is None:
        return None
    try:
        return yaml.safe_load(text)
    except yaml.YAMLError as exc:
        msg = f"{path}: invalid YAML ({exc})"
        raise TreeReadError(msg) from exc


class HadolintPlugin(Plugin):
    name = "hadolint"

    def detect(self, tree: RepoTree) -> bool:
        return bool(_dockerfiles(tree))

    def sites(self, tree: RepoTree) -> list[Site]:
        sites: list[Site] = []
        for line in lines_of(tree, _dockerfiles(tree)):
            match = _IGNORE_RE.search(line.text)
            if match:
                rules = split_rules(match.group("rules"), sep=",")
                sites.append(Site(self.name, line.path, line.number, rules, marker_id(line.text, line.previous)))
        return sites

    def config_disables(self, tree: RepoTree) -> list[ConfigDisable]:
        out: list[ConfigDisable] = []
        for path in _CONFIGS:
            data = read_yaml(tree, path)
            if isinstance(data, dict):
                out.extend(ConfigDisable(self.name, r, ("**",), f"{path} ignored") for r in str_list(data.get("ignored")))
        return out
```

```python
# src/vergil_tooling/lib/exceptions/plugins/actionlint.py
"""actionlint: ``paths.<glob>.ignore`` patterns in ``.github/actionlint.yaml``."""

from __future__ import annotations

from typing import TYPE_CHECKING

from vergil_tooling.lib.exceptions.model import ConfigDisable, Site
from vergil_tooling.lib.exceptions.plugin import Plugin
from vergil_tooling.lib.exceptions.plugins._config import str_list
from vergil_tooling.lib.exceptions.plugins.hadolint import read_yaml

if TYPE_CHECKING:
    from vergil_tooling.lib.exceptions.tree import RepoTree

_CONFIGS = (".github/actionlint.yaml", ".github/actionlint.yml")


class ActionlintPlugin(Plugin):
    name = "actionlint"

    def detect(self, tree: RepoTree) -> bool:
        return any(p.startswith(".github/workflows/") for p in tree.files())

    @override
    def sites(self, tree: RepoTree) -> list[Site]:
        return []

    def config_disables(self, tree: RepoTree) -> list[ConfigDisable]:
        out: list[ConfigDisable] = []
        for path in _CONFIGS:
            data = read_yaml(tree, path)
            paths = data.get("paths", {}) if isinstance(data, dict) else {}
            for glob, spec in (paths or {}).items():
                for pattern in str_list(spec.get("ignore") if isinstance(spec, dict) else None):
                    out.append(ConfigDisable(self.name, pattern, (glob,), f"{path} paths.ignore"))
        return out
```

```python
# src/vergil_tooling/lib/exceptions/plugins/semgrep.py
"""semgrep: ``nosemgrep`` comments in any language and ``.semgrepignore`` paths.

The fleet's ``DEFAULT_EXCLUDED_RULES`` (``lib/semgrep.py``) are fleet defaults
documented in the fleet register (Task T4). A per-repo ``--exclude-rule`` flag (the
only way a repo can add one: the vergil-actions semgrep action exposes no exclusion
input) in a workflow or shell script is a repo-wide config disable (spec §7).

``nosemgrep`` sites take their ``# vrg-exc:`` marker on the line above (spec §5.2):
semgrep reads the rest of a ``nosemgrep:`` comment as its rule-ID list, so a
trailing marker on the same line is not relied on.
"""

from __future__ import annotations

import re
from typing import TYPE_CHECKING

from vergil_tooling.lib.exceptions.markers import marker_id
from vergil_tooling.lib.exceptions.model import ConfigDisable, Site
from vergil_tooling.lib.exceptions.plugin import Plugin
from vergil_tooling.lib.exceptions.plugins._text import by_suffix, lines_of, shell_scripts, split_rules

if TYPE_CHECKING:
    from vergil_tooling.lib.exceptions.tree import RepoTree

_SUFFIXES = (".py", ".sh", ".bash", ".yml", ".yaml", ".js", ".ts", ".tsx", ".go", ".java", ".rs", ".rb", ".cpp", ".hpp", ".c", ".h")
_NOSEM_RE = re.compile(r"(?:^|\s)(?:#|//)\s*nosemgrep(?::\s*(?P<rules>[\w.-]+(?:\s*,\s*[\w.-]+)*))?")
_EXCLUDE_RE = re.compile(r"--exclude-rule(?:=|\s+)(?P<rule>[\w.-]+)")


class SemgrepPlugin(Plugin):
    name = "semgrep"

    def detect(self, tree: RepoTree) -> bool:
        return bool(by_suffix(tree, *_SUFFIXES))

    def sites(self, tree: RepoTree) -> list[Site]:
        sites: list[Site] = []
        for line in lines_of(tree, by_suffix(tree, *_SUFFIXES)):
            match = _NOSEM_RE.search(line.text)
            if match:
                rules = split_rules(match.group("rules"), sep=r"\s*,\s*")
                sites.append(Site(self.name, line.path, line.number, rules, marker_id(line.text, line.previous)))
        return sites

    def config_disables(self, tree: RepoTree) -> list[ConfigDisable]:
        out = [
            ConfigDisable(self.name, "*", (line.text.strip(),), ".semgrepignore")
            for line in lines_of(tree, [".semgrepignore"])
            if line.text.strip() and not line.text.lstrip().startswith("#")
        ]
        callers = [p for p in tree.files() if p.startswith(".github/workflows/") and p.endswith((".yml", ".yaml"))]
        for line in lines_of(tree, callers + shell_scripts(tree)):
            out.extend(
                ConfigDisable(self.name, match.group("rule"), ("**",), f"{line.path}:{line.number} --exclude-rule")
                for match in _EXCLUDE_RE.finditer(line.text)
            )
        return out
```

> Python files are scanned here with the line scanner, not `tokenize`: a
> `# nosemgrep` inside a Python string is rare and semgrep itself matches the
> raw comment text. Keep `.py` in `_SUFFIXES`.

```python
# src/vergil_tooling/lib/exceptions/plugins/trivy.py
"""trivy: ``.trivyignore`` CVE entries, ``trivy.yaml`` ignore-unfixed, Rego policies.

A comment-only ``# vrg-exc: ID`` line applies to the entries that follow it up
to the next blank line (entries are grouped by carrier).
"""

from __future__ import annotations

import re
from typing import TYPE_CHECKING

from vergil_tooling.lib.exceptions.markers import MARKER_RE
from vergil_tooling.lib.exceptions.model import ConfigDisable, Site
from vergil_tooling.lib.exceptions.plugin import Plugin
from vergil_tooling.lib.exceptions.plugins._text import lines_of
from vergil_tooling.lib.exceptions.plugins.hadolint import read_yaml

if TYPE_CHECKING:
    from vergil_tooling.lib.exceptions.tree import RepoTree

_ENTRY_RE = re.compile(r"^\s*(?P<id>[A-Za-z0-9][\w.:-]*)")


class TrivyPlugin(Plugin):
    name = "trivy"

    def detect(self, tree: RepoTree) -> bool:
        files = set(tree.files())
        return ".trivyignore" in files or "trivy.yaml" in files

    def sites(self, tree: RepoTree) -> list[Site]:
        sites: list[Site] = []
        group_id: str | None = None
        for line in lines_of(tree, [".trivyignore"]):
            stripped = line.text.strip()
            if not stripped:
                group_id = None
                continue
            if stripped.startswith("#"):
                marker = MARKER_RE.search(stripped)
                if marker:
                    group_id = marker.group(1)
                continue
            match = _ENTRY_RE.match(stripped)
            if match:
                sites.append(Site(self.name, line.path, line.number, (match.group("id"),), group_id))
        return sites

    def config_disables(self, tree: RepoTree) -> list[ConfigDisable]:
        out: list[ConfigDisable] = []
        data = read_yaml(tree, "trivy.yaml")
        vuln = data.get("vulnerability", {}) if isinstance(data, dict) else {}
        if isinstance(vuln, dict) and vuln.get("ignore-unfixed") is True:
            out.append(ConfigDisable(self.name, "ignore-unfixed", ("**",), "trivy.yaml vulnerability.ignore-unfixed"))
        out.extend(
            ConfigDisable(self.name, "rego-policy", (path,), path)
            for path in tree.files()
            if path.endswith(".rego") and path.startswith(".trivy/")
        )
        return out
```

```python
# src/vergil_tooling/lib/exceptions/plugins/uncovered.py
"""Mechanisms present in a repo that no plug-in audits yet — reported, never silent (§6.2)."""

from __future__ import annotations

import re
from dataclasses import dataclass
from typing import TYPE_CHECKING

if TYPE_CHECKING:
    from vergil_tooling.lib.exceptions.tree import RepoTree


@dataclass(frozen=True)
class RegexDetector:
    name: str
    suffixes: tuple[str, ...]
    pattern: re.Pattern[str]

    def detect(self, tree: RepoTree) -> bool:
        for path in tree.files():
            if path.endswith(self.suffixes) or path.rsplit("/", 1)[-1] in self.suffixes:
                text = tree.read(path) or ""
                if self.pattern.search(text):
                    return True
        return False


DETECTORS: tuple[RegexDetector, ...] = (
    RegexDetector("eslint", (".js", ".jsx", ".ts", ".tsx", ".mjs"), re.compile(r"eslint-disable")),
    RegexDetector("clang-tidy", (".c", ".cc", ".cpp", ".h", ".hpp"), re.compile(r"\bNOLINT")),
    RegexDetector("cppcheck", (".c", ".cc", ".cpp", ".h", ".hpp"), re.compile(r"cppcheck-suppress")),
    RegexDetector("golangci-lint", (".go",), re.compile(r"//\s*nolint")),
    RegexDetector("clippy", (".rs",), re.compile(r"#\[allow\(")),
    RegexDetector("java", (".java",), re.compile(r"@SuppressWarnings")),
    RegexDetector("rubocop", (".rb",), re.compile(r"rubocop:disable")),
    RegexDetector("codeql", (".py", ".js", ".ts", ".go", ".java", ".cpp"), re.compile(r"(codeql|lgtm)\[")),
    RegexDetector("ansible-lint", (".ansible-lint", ".ansible-lint.yml", ".ansible-lint.yaml"),
                  re.compile(r"^\s*(skip_list|exclude_paths|warn_list)\s*:", re.MULTILINE)),
)
```

Register them in `plugins/__init__.py`:

```python
PLUGINS: tuple[Plugin, ...] = (
    RuffPlugin(), MypyPlugin(), TyPlugin(), CoveragePlugin(),
    MarkdownlintPlugin(), ShellcheckPlugin(), YamllintPlugin(), HadolintPlugin(),
    ActionlintPlugin(), SemgrepPlugin(), TrivyPlugin(),
)
UNCOVERED: tuple[UncoveredDetector, ...] = DETECTORS
```

- [ ] **Step 5: Run the tests to verify they pass**

Run: `vrg-container-run -- uv run pytest tests/vergil_tooling/test_exceptions_plugins_text.py tests/vergil_tooling/test_exceptions_plugins_python.py -q --no-cov`
Expected: PASS.

- [ ] **Step 6: Full validation and commit**

Run: `vrg-container-run -- vrg-validate` — Expected: PASS, coverage 100%.

```bash
vrg-git add src/vergil_tooling/lib/exceptions/plugins tests/vergil_tooling/test_exceptions_plugins_text.py
vrg-commit --type feat --scope exceptions --message "add common-check, semgrep and trivy exception plug-ins" --body "Ref #<task>"
```

---

## Task T4: fleet register and the ruff managed block

**Repo:** vergil-tooling. Blocked by: T1, T2 (ruff plug-in). Spec: §4, §5.3, §6.3.

**Files:**

- Create: `src/vergil_tooling/configs/exceptions.toml` (the fleet register)
- Create: `src/vergil_tooling/lib/exceptions/fleet.py`
- Create: `src/vergil_tooling/lib/exceptions/managed.py`
- Modify: `src/vergil_tooling/lib/exceptions/plugins/ruff.py` (mark managed-block rules)
- Test: `tests/vergil_tooling/test_exceptions_fleet.py`, `test_exceptions_managed.py`.

**Interfaces:**

- Consumes (T1/T2): `register_from_data`, `Register`, `Finding`, `Notice`,
  `ConfigDisable`, `RuffPlugin`, `known_tools`, `gitignore.MANAGED_END`.
- Produces:

```python
# fleet.py
FLEET_ONLY_TOOLS: frozenset[str]   # tools whose fleet config ships in the package but have no plug-in
@dataclass(frozen=True) class Fleet: revision: int; ruff_rules: tuple[str, ...]; history: dict[int, str]; register: Register
def load_fleet() -> Fleet          # reads vergil_tooling.configs/exceptions.toml; RegisterError on any problem
def parse_fleet(text: str) -> Fleet

# managed.py
class ManagedBlockError(Exception)
@dataclass(frozen=True) class ManagedBlock: revision: int; digest: str; rules: tuple[str, ...]; begin: int; end: int  # 0-based line indexes of the fence lines
def body_digest(rules: Sequence[str]) -> str
def render_block(revision: int, rules: Sequence[str], indent: str = "    ") -> list[str]
def find_block(text: str) -> ManagedBlock | None
def check_block(block: ManagedBlock | None, fleet: Fleet, *, path: str) -> tuple[list[Finding], list[Notice]]
def sync_pyproject(text: str, fleet: Fleet) -> str
```

Finding codes added: `managed-edited`, `managed-unknown-revision`. Notice code
added: `fleet-behind` — also used for a **missing** block, which is behind at
revision 0 (spec §6.3).

**Fleet register format** (`configs/exceptions.toml`). Entries are `[[rule]]`
only and carry **no `id`**: a fleet default is identified by tool and rule, and a
`(tool, rule)` pair may appear in only one entry (spec §5.3). There are no
`[[inline]]` fleet entries — line-level exceptions always belong to the repo
holding the line.

```toml
# The fleet exception register (epic vergil-project/.github#375, spec §5.3).
# Entries document every rule Vergil disables for all repos. Bump
# `revision` and append to [[history]] whenever [managed.ruff].rules changes;
# a test enforces it.
revision = 1

[managed.ruff]
# Fleet ruff ignores delivered into each repo's pyproject.toml by managed block.
rules = []

[[history]]
revision = 1
sha256 = "<body_digest([]) — computed in Step 4>"

[[rule]]
tool   = "markdownlint"
rules  = ["MD060"]
scope  = ["**"]
reason = "<the rationale; find it in git log -S 'MD060' -- src/vergil_tooling/configs/markdownlint.yaml>"
added  = 2026-10-20
# … one entry per disabled rule, see the inventory below
```

> `reason` values must be real rationale, not placeholders: recover each one
> from the commit or issue that introduced the setting (`vrg-git log -S
> '<setting>' -- <file>`, then the referenced issue). Where no rationale exists,
> stop and ask the human for it — that is the exception-request rule applied to
> the fleet itself.

**Inventory of fleet exceptions to document** (from the shipped configs on
2026-10-15; the completeness tests in Step 1 keep it honest):

| Tool | Rule | Source |
|---|---|---|
| markdownlint | `MD060` | `configs/markdownlint.yaml` (`MD060: false`) |
| markdownlint | `MD013` (tables, code blocks) | `markdownlint.yaml` `MD013.tables/code_blocks: false` |
| markdownlint | `MD024` (siblings only) | `markdownlint.yaml` `no-duplicate-heading.siblings_only` |
| markdownlint | `MD013` scope `epics/**` | `configs/markdownlint-prose.yaml` (issue #2803) |
| yamllint | `document-start` | `configs/yamllint.yaml` |
| yamllint | `truthy` (keys) | `yamllint.yaml` `truthy.check-keys: false` |
| yamllint | `line-length` scope `.github/workflows/**` | `yamllint.yaml` `line-length.ignore` |
| semgrep | `yaml.github-actions.security.github-actions-mutable-action-tag.github-actions-mutable-action-tag` | `lib/semgrep.py` `DEFAULT_EXCLUDED_RULES` (#2513) |
| cppcheck | `missingIncludeSystem`, `unmatchedSuppression`, `checkersReport`, `missingInclude` | `configs/cpp/cppcheck-suppressions.txt` |
| clang-tidy | `modernize-use-trailing-return-type`, `readability-identifier-length`, `readability-magic-numbers`, `cppcoreguidelines-avoid-magic-numbers`, `cppcoreguidelines-pro-bounds-array-to-pointer-decay` | `configs/cpp/.clang-tidy` (negated checks) |
| gcovr | `exclude .*/build.*/.*`, `exclude .*/tests?/.*`, `exclude-unreachable-branches`, `exclude-throw-branches`, `gcov-ignore-errors source_not_found` | `configs/cpp/gcovr.cfg` (#3026) |
| eslint | every rule set to `"off"`/`0` | `configs/typescript/eslint.config.mjs` — enumerate by reading the file |
| ansible-lint | anything the bundled `configs/ansible-lint.yaml` skips (`profile: production` today; none) | — |

Related rules may share one entry (e.g. the four cppcheck suppressions), as
long as the reason covers all of them.

Not in the fleet register: the `nosemgrep` for `secrets-inherit` that
`lib/repo_init.py:877` generates. It is a line-level exception, so each repo
documents it in its own register — Task T10 makes `repo-init` seed that entry.

- [ ] **Step 1: Write the failing fleet tests**

```python
# tests/vergil_tooling/test_exceptions_fleet.py
import re
from importlib.resources import files

import pytest
import yaml

from vergil_tooling.lib.exceptions.fleet import load_fleet, parse_fleet
from vergil_tooling.lib.exceptions.managed import body_digest
from vergil_tooling.lib.exceptions.register import RegisterError
from vergil_tooling.lib.semgrep import DEFAULT_EXCLUDED_RULES

CONFIGS = files("vergil_tooling.configs")


def _documented(tool: str) -> set[str]:
    return {r for e in load_fleet().register.entries if e.tool == tool for r in e.rules}


def test_fleet_loads_and_history_matches_current_rules() -> None:
    fleet = load_fleet()
    assert fleet.history[fleet.revision] == body_digest(fleet.ruff_rules)
    assert max(fleet.history) == fleet.revision


def test_every_managed_ruff_rule_is_documented() -> None:
    fleet = load_fleet()
    assert set(fleet.ruff_rules) <= _documented("ruff")


def test_markdownlint_disables_documented() -> None:
    cfg = yaml.safe_load((CONFIGS / "markdownlint.yaml").read_text())
    disabled = {k for k, v in cfg.items() if v is False}
    assert disabled <= _documented("markdownlint")


def test_yamllint_disables_documented() -> None:
    cfg = yaml.safe_load((CONFIGS / "yamllint.yaml").read_text())
    disabled = {k for k, v in cfg["rules"].items() if v == "disable"}
    assert disabled <= _documented("yamllint")


def test_semgrep_default_exclusions_documented() -> None:
    assert set(DEFAULT_EXCLUDED_RULES) <= _documented("semgrep")


def test_cppcheck_suppressions_documented() -> None:
    text = (CONFIGS / "cpp" / "cppcheck-suppressions.txt").read_text()
    ids = {ln.strip() for ln in text.splitlines() if ln.strip() and not ln.startswith("#")}
    assert ids <= _documented("cppcheck")


def test_clang_tidy_negations_documented() -> None:
    text = (CONFIGS / "cpp" / ".clang-tidy").read_text()
    negated = set(re.findall(r"^\s*-([\w.-]+)", text, re.MULTILINE))
    assert negated <= _documented("clang-tidy")


def test_history_missing_current_revision_is_error() -> None:
    with pytest.raises(RegisterError, match="history"):
        parse_fleet('revision = 2\n[managed.ruff]\nrules = []\n[[history]]\nrevision = 1\nsha256 = "x"\n')


def test_fleet_entries_take_no_id() -> None:
    with pytest.raises(RegisterError, match="fleet entries take no id"):
        parse_fleet('revision = 1\n[managed.ruff]\nrules = []\n[[history]]\nrevision = 1\n'
                    f'sha256 = "{body_digest([])}"\n[[rule]]\nid = "EXC-001"\ntool = "ruff"\n'
                    'rules = ["D400"]\nscope = ["**"]\nreason = "r"\nadded = 2026-10-20\n')
```

- [ ] **Step 2: Write the failing managed-block tests**

```python
# tests/vergil_tooling/test_exceptions_managed.py
import tomllib

import pytest

from vergil_tooling.lib.exceptions.fleet import Fleet
from vergil_tooling.lib.exceptions.managed import (
    ManagedBlockError, body_digest, check_block, find_block, render_block, sync_pyproject,
)
from vergil_tooling.lib.exceptions.model import Register


def _fleet(revision: int, rules: tuple[str, ...], history: dict[int, tuple[str, ...]]) -> Fleet:
    return Fleet(revision=revision, ruff_rules=rules,
                 history={rev: body_digest(r) for rev, r in history.items()}, register=Register(entries=()))


FLEET2 = _fleet(2, ("D400", "D415"), {1: ("D400",), 2: ("D400", "D415")})

PYPROJECT = (
    "[project]\nname = 'x'\n\n"
    "[tool.ruff.lint]\n"
    'select = ["E"]\n'
    "extend-ignore = [\n"
    '    "E501",  # repo-owned\n'
    "]\n\n"
    "[tool.mypy]\nstrict = true\n"
)


def test_render_and_find_round_trip() -> None:
    text = "extend-ignore = [\n" + "\n".join(render_block(2, ["D400", "D415"])) + "\n]\n"
    block = find_block(text)
    assert block is not None
    assert (block.revision, block.rules) == (2, ("D400", "D415"))
    assert block.digest == body_digest(["D400", "D415"])


def test_find_block_absent() -> None:
    assert find_block(PYPROJECT) is None


def test_unterminated_block_is_error() -> None:
    with pytest.raises(ManagedBlockError, match="no closing"):
        find_block(render_block(1, ["D400"])[0] + "\n")


def test_sync_inserts_block_preserving_repo_entries() -> None:
    out = sync_pyproject(PYPROJECT, FLEET2)
    lint = tomllib.loads(out)["tool"]["ruff"]["lint"]
    assert lint["extend-ignore"] == ["E501", "D400", "D415"]
    assert "# repo-owned" in out
    assert out.startswith("[project]\nname = 'x'\n")          # rest of the file untouched
    assert out.endswith("[tool.mypy]\nstrict = true\n")


def test_sync_is_idempotent_and_upgrades_old_revision() -> None:
    old = sync_pyproject(PYPROJECT, _fleet(1, ("D400",), {1: ("D400",)}))
    new = sync_pyproject(old, FLEET2)
    assert new == sync_pyproject(new, FLEET2)
    block = find_block(new)
    assert block is not None
    assert block.revision == 2


def test_sync_adds_table_when_missing() -> None:
    out = sync_pyproject("[project]\nname = 'x'\n", FLEET2)
    assert tomllib.loads(out)["tool"]["ruff"]["lint"]["extend-ignore"] == ["D400", "D415"]


def test_sync_rewrites_single_line_array() -> None:
    text = '[tool.ruff.lint]\nextend-ignore = ["E501"]\n'
    out = sync_pyproject(text, FLEET2)
    assert tomllib.loads(out)["tool"]["ruff"]["lint"]["extend-ignore"] == ["E501", "D400", "D415"]


def test_check_current_block_is_clean() -> None:
    block = find_block(sync_pyproject(PYPROJECT, FLEET2))
    assert check_block(block, FLEET2, path="pyproject.toml") == ([], [])


def test_check_old_block_is_notice_not_finding() -> None:
    block = find_block(sync_pyproject(PYPROJECT, _fleet(1, ("D400",), {1: ("D400",)})))
    findings, notices = check_block(block, FLEET2, path="pyproject.toml")
    assert findings == []
    assert [n.code for n in notices] == ["fleet-behind"]


def test_check_hand_edited_block_fails() -> None:
    text = sync_pyproject(PYPROJECT, FLEET2).replace('    "D415",\n', "")
    findings, _ = check_block(find_block(text), FLEET2, path="pyproject.toml")
    assert [f.code for f in findings] == ["managed-edited"]


def test_check_unknown_revision_fails() -> None:
    text = sync_pyproject(PYPROJECT, FLEET2).replace(" rev 2 ", " rev 9 ")
    findings, _ = check_block(find_block(text), FLEET2, path="pyproject.toml")
    assert [f.code for f in findings] == ["managed-unknown-revision"]


def test_check_missing_block_with_fleet_rules_is_behind_not_failure() -> None:
    # spec §6.3: a missing block is behind at rev 0 — the repo still enforces the
    # fleet rules (stricter), so it is reported, never failed (no flag day).
    findings, notices = check_block(None, FLEET2, path="pyproject.toml")
    assert findings == []
    assert [n.code for n in notices] == ["fleet-behind"]
    assert "vrg-exceptions sync" in notices[0].message


def test_check_missing_block_with_empty_fleet_is_notice() -> None:
    findings, notices = check_block(None, _fleet(1, (), {1: ()}), path="pyproject.toml")
    assert findings == []
    assert [n.code for n in notices] == ["fleet-behind"]
```

Add to `tests/vergil_tooling/test_exceptions_plugins_python.py`:

```python
def test_ruff_marks_managed_block_rules() -> None:
    from vergil_tooling.lib.exceptions.managed import sync_pyproject
    from tests.vergil_tooling.test_exceptions_managed import FLEET2, PYPROJECT

    tree = FakeTree({"pyproject.toml": sync_pyproject(PYPROJECT, FLEET2)})
    managed = {d.rule: d.managed for d in RuffPlugin().config_disables(tree)}
    assert managed == {"E501": False, "D400": True, "D415": True}
```

- [ ] **Step 3: Run to verify failure**

Run: `vrg-container-run -- uv run pytest tests/vergil_tooling/test_exceptions_fleet.py tests/vergil_tooling/test_exceptions_managed.py -q --no-cov`
Expected: FAIL — modules missing.

- [ ] **Step 4: Implement `managed.py`**

```python
# src/vergil_tooling/lib/exceptions/managed.py
"""Fleet-managed blocks with revision + checksum stamps (spec §6.3).

v1 manages one block: the fleet ruff ignores inside ``[tool.ruff.lint]``
``extend-ignore`` in ``pyproject.toml``. The block is rewritten as text so the
rest of the file — formatting, comments, repo-owned entries — is untouched. The
render/parse/hash/check functions are tool-agnostic so later tools can reuse them
(follow-on vergil-project/.github#378).
"""

from __future__ import annotations

import hashlib
import re
import tomllib
from dataclasses import dataclass
from typing import TYPE_CHECKING

from vergil_tooling.lib.exceptions.model import Finding, Notice
from vergil_tooling.lib.gitignore import MANAGED_END

if TYPE_CHECKING:
    from collections.abc import Sequence

    from vergil_tooling.lib.exceptions.fleet import Fleet

_BEGIN_FMT = "# >>> vergil-managed: fleet-exceptions rev {rev} sha256:{digest} (do not edit) >>>"
_BEGIN_RE = re.compile(r"^\s*# >>> vergil-managed: fleet-exceptions rev (\d+) sha256:([0-9a-f]{64}) \(do not edit\) >>>\s*$")
_ITEM_RE = re.compile(r'^\s*"([^"]+)",?\s*$')
_TABLE_RE = re.compile(r"^\s*\[([^\[\]]+)\]\s*$")
_EXTEND_RE = re.compile(r"^(\s*)extend-ignore\s*=\s*\[(.*)$")


class ManagedBlockError(Exception):
    """A managed block that cannot be parsed or a file that cannot be synced safely."""


@dataclass(frozen=True)
class ManagedBlock:
    revision: int
    digest: str
    rules: tuple[str, ...]
    begin: int
    end: int


def body_digest(rules: Sequence[str]) -> str:
    return hashlib.sha256("\n".join(rules).encode("utf-8")).hexdigest()


def render_block(revision: int, rules: Sequence[str], indent: str = "    ") -> list[str]:
    begin = indent + _BEGIN_FMT.format(rev=revision, digest=body_digest(rules))
    return [begin, *(f'{indent}"{rule}",' for rule in rules), indent + MANAGED_END]


def find_block(text: str) -> ManagedBlock | None:
    lines = text.splitlines()
    begins = [(i, m) for i, line in enumerate(lines) if (m := _BEGIN_RE.match(line))]
    if not begins:
        return None
    if len(begins) > 1:
        msg = f"{len(begins)} fleet-exceptions managed blocks found; expected one"
        raise ManagedBlockError(msg)
    begin, match = begins[0]
    for end in range(begin + 1, len(lines)):
        if lines[end].strip() == MANAGED_END:
            body = lines[begin + 1 : end]
            rules = []
            for line in body:
                item = _ITEM_RE.match(line)
                if item is None:
                    msg = f"unexpected line inside managed block: {line.strip()!r}"
                    raise ManagedBlockError(msg)
                rules.append(item.group(1))
            return ManagedBlock(int(match.group(1)), match.group(2), tuple(rules), begin, end)
    msg = "fleet-exceptions managed block has no closing marker"
    raise ManagedBlockError(msg)


def check_block(block: ManagedBlock | None, fleet: Fleet, *, path: str) -> tuple[list[Finding], list[Notice]]:
    if block is None:
        # Behind at rev 0, never a failure (spec §6.3): the fleet rules are still enforced here.
        return [], [Notice("fleet-behind", f"{path} has no fleet-exceptions managed block (current rev "
                           f"{fleet.revision}); run 'vrg-exceptions sync'")]
    where = {"path": path, "line": block.begin + 1}
    expected = fleet.history.get(block.revision)
    if expected is None:
        return [Finding("managed-unknown-revision", f"managed block claims fleet rev {block.revision}, which this "
                        f"vergil-tooling does not know (current rev {fleet.revision})", **where)], []
    if body_digest(block.rules) != block.digest or block.digest != expected:
        return [Finding("managed-edited", "fleet-exceptions managed block was edited by hand; fleet exceptions "
                        "change in vergil-tooling, then 'vrg-exceptions sync'", **where)], []
    if block.revision < fleet.revision:
        return [], [Notice("fleet-behind", f"{path} fleet exceptions at rev {block.revision}, current is "
                           f"{fleet.revision}; run 'vrg-exceptions sync'")]
    return [], []
```

`sync_pyproject` (same module) — a small line-based editor. Algorithm:

1. Split `text` into lines (keep a trailing-newline flag).
2. Find the `[tool.ruff.lint]` table header line and its end (next `[...]`
   header or EOF). If absent, append `\n[tool.ruff.lint]\nextend-ignore = [\n<block>\n]\n`
   and go to step 6.
3. Inside the table, find `extend-ignore = [`. If absent, insert
   `extend-ignore = [`, the block lines, `]` right after the header line.
4. If the array closes on the same line (`extend-ignore = ["E501"]`), rewrite it
   to multi-line: one `    "<item>",` per existing item (parse the inline array
   with `tomllib.loads("x = [" + rest)`), then the block, then `]`.
5. Otherwise, if a managed block exists inside the array (`find_block`), replace
   lines `begin..end` with the new block; if not, insert the block immediately
   before the array's closing `]` line.
6. Re-join, then **verify**: `tomllib.loads` the result and assert
   `extend-ignore == repo_items + fleet.ruff_rules` (repo items = the array's
   items outside the block, in order). On any mismatch raise
   `ManagedBlockError("sync would corrupt pyproject.toml: …")` — never write a
   file that fails this check.

```python
def sync_pyproject(text: str, fleet: Fleet) -> str:
    block_lines = render_block(fleet.revision, fleet.ruff_rules)
    lines = text.splitlines()
    trailing = text.endswith("\n")
    header = next((i for i, ln in enumerate(lines) if (m := _TABLE_RE.match(ln)) and m.group(1).strip() == "tool.ruff.lint"), None)
    if header is None:
        lines += ["", "[tool.ruff.lint]", "extend-ignore = [", *block_lines, "]"]
        return _verified("\n".join(lines) + "\n", fleet, repo_items=[])
    table_end = next((i for i in range(header + 1, len(lines)) if _TABLE_RE.match(lines[i])), len(lines))
    found = next(((i, m) for i in range(header + 1, table_end) if (m := _EXTEND_RE.match(lines[i]))), None)
    if found is None:
        lines[header + 1 : header + 1] = ["extend-ignore = [", *block_lines, "]"]
        return _verified(_join(lines, trailing), fleet, repo_items=[])
    start, match = found
    rest = match.group(2)

    if "]" in rest:  # single-line array
        items = tomllib.loads("x = [" + rest)["x"]
        lines[start : start + 1] = ["extend-ignore = [", *(f'    "{i}",' for i in items), *block_lines, "]"]
        return _verified(_join(lines, trailing), fleet, repo_items=items)
    close = next(i for i in range(start + 1, table_end) if lines[i].strip().startswith("]"))
    existing = find_block("\n".join(lines[start:close + 1]))
    if existing is not None:
        b, e = start + existing.begin, start + existing.end
        lines[b : e + 1] = block_lines
    else:
        lines[close:close] = block_lines
    repo_items = _repo_items(lines, start)
    return _verified(_join(lines, trailing), fleet, repo_items=repo_items)
```

Write `_join(lines, trailing)` (re-adds the final newline), `_repo_items(lines,
start)` (the quoted items between `start` and the closing `]`, excluding the
managed block's lines), and `_verified(out, fleet, *, repo_items)` (step 6).
`close` uses `next(..., None)` too: an array with no closing `]` inside the table
raises `ManagedBlockError("extend-ignore array is not closed in [tool.ruff.lint]")`
(add a test).

Compute `body_digest([])` and paste it into `configs/exceptions.toml`
`[[history]] sha256` (`vrg-container-run -- uv run python -c "from
vergil_tooling.lib.exceptions.managed import body_digest; print(body_digest([]))"`).

- [ ] **Step 5: Implement `fleet.py`**

```python
# src/vergil_tooling/lib/exceptions/fleet.py
"""The fleet exception register shipped in ``vergil_tooling.configs`` (spec §5.3)."""

from __future__ import annotations

import tomllib
from dataclasses import dataclass
from importlib.resources import files

from vergil_tooling.lib.exceptions.model import Register
from vergil_tooling.lib.exceptions.plugins import known_tools
from vergil_tooling.lib.exceptions.register import RegisterError, register_from_data

FLEET_FILE = "exceptions.toml"
#: Tools Vergil configures centrally that have no audit plug-in (their config is
#: never in the repo, so only the fleet register documents them).
FLEET_ONLY_TOOLS = frozenset({"clang-tidy", "cppcheck", "gcovr", "eslint", "ansible-lint"})


@dataclass(frozen=True)
class Fleet:
    revision: int
    ruff_rules: tuple[str, ...]
    history: dict[int, str]
    register: Register


def parse_fleet(text: str) -> Fleet:
    source = f"vergil_tooling.configs/{FLEET_FILE}"
    try:
        raw = tomllib.loads(text)
    except tomllib.TOMLDecodeError as exc:
        raise RegisterError(source, [f"invalid TOML: {exc}"]) from exc
    register = register_from_data(raw, source=source, known_tools=known_tools() | FLEET_ONLY_TOOLS, fleet=True,
                                  extra_tables=frozenset({"revision", "managed", "history"}))
    revision = raw.get("revision")
    rules = raw.get("managed", {}).get("ruff", {}).get("rules")
    history = {h.get("revision"): h.get("sha256") for h in raw.get("history", []) if isinstance(h, dict)}
    problems: list[str] = []
    if not isinstance(revision, int) or revision < 1:
        problems.append("revision must be a positive integer")
    if not isinstance(rules, list) or not all(isinstance(r, str) for r in rules):
        problems.append("[managed.ruff] rules must be a list of strings")
    if revision not in history:
        problems.append(f"[[history]] has no entry for the current revision {revision}")
    if problems:
        raise RegisterError(source, problems)
    return Fleet(revision=revision, ruff_rules=tuple(rules), history=history, register=register)


def load_fleet() -> Fleet:
    return parse_fleet((files("vergil_tooling.configs") / FLEET_FILE).read_text(encoding="utf-8"))
```

(`[[history]]` entries are trusted only after the type checks above; add a test
for a non-string `sha256` → `RegisterError`.)

- [ ] **Step 6: Write `configs/exceptions.toml`** from the inventory table, with
real reasons recovered from git history (see the note above the table).

- [ ] **Step 7: Mark managed rules in the ruff plug-in**

In `RuffPlugin.config_disables`, for `pyproject.toml` only: call
`find_block(text)` on the raw file (`tree.read("pyproject.toml")`) and set
`managed=True` on each `extend-ignore` disable whose rule is in `block.rules`.
A `ManagedBlockError` from `find_block` propagates (the CLI reports it).

- [ ] **Step 8: Run the tests, full validation, commit**

Run: `vrg-container-run -- uv run pytest tests/vergil_tooling/test_exceptions_fleet.py tests/vergil_tooling/test_exceptions_managed.py tests/vergil_tooling/test_exceptions_plugins_python.py -q --no-cov` — Expected: PASS.
Run: `vrg-container-run -- vrg-validate` — Expected: PASS, coverage 100%.

```bash
vrg-git add src/vergil_tooling/configs/exceptions.toml src/vergil_tooling/lib/exceptions tests/vergil_tooling
vrg-commit --type feat --scope exceptions --message "add the fleet exception register and ruff managed block" --body "Ref #<task>"
```

---

## Task T5: `vrg-exceptions` CLI (`audit`, `sync`) and the `vrg-validate` common check

**Repo:** vergil-tooling. Blocked by: T1, T2, T3, T4. Spec: §6.4, §11.

**Files:**

- Create: `src/vergil_tooling/bin/vrg_exceptions.py`
- Modify: `pyproject.toml` `[project.scripts]` — add
  `vrg-exceptions = "vergil_tooling.bin.vrg_exceptions:main"` (alphabetical position)
- Modify: `src/vergil_tooling/bin/validate_common.py` (new last check; update the module docstring list)
- Test: `tests/vergil_tooling/test_vrg_exceptions.py`, extend `tests/vergil_tooling/test_validate_common.py`

**Interfaces:**

- Consumes: `REGISTER_FILE`, `parse_register`, `RegisterError`, `WorkTree`,
  `TreeReadError`, `collect`, `audit`, `PLUGINS`, `UNCOVERED`, `known_tools`,
  `load_fleet`, `find_block`, `check_block`, `sync_pyproject`, `ManagedBlockError`,
  `output.emit_error`, `output.is_ci`, `output.write_summary`.
- Produces:

```python
# bin/vrg_exceptions.py
@cli_main
def main(argv: list[str] | None = None) -> int     # subcommands: audit | sync | diff (T6) | report (T8)
def run_audit(root: Path, *, today: date) -> int    # 0 pass / not adopted, 1 findings or errors
def load_repo_register(root: Path) -> Register | None   # None when not adopted; RegisterError when invalid
```

Output contract (`audit`):

- Not adopted: prints `vrg-exceptions: not adopted (no vergil-exceptions.toml); skipping` and returns 0.
- Each finding: `emit_error(f"[{code}] {message}", file=path, line=line)` (GitHub
  annotation in CI, `error:` line locally).
- Each notice: `print(f"note: [{code}] {message}")`.
- Summary line: `vrg-exceptions: N finding(s), M notice(s)`; in CI also
  `write_summary` with a Markdown table of findings and a list of notices.
- `RegisterError`, `TreeReadError`, `ManagedBlockError`: each problem via
  `emit_error`, return 1.

- [ ] **Step 1: Write failing CLI tests**

```python
# tests/vergil_tooling/test_vrg_exceptions.py
from datetime import date
from pathlib import Path
from unittest import mock

import pytest

from vergil_tooling.bin import vrg_exceptions
from vergil_tooling.lib.exceptions.audit import Collected
from vergil_tooling.lib.exceptions.model import Site

TODAY = date(2026, 11, 1)
_MOD = "vergil_tooling.bin.vrg_exceptions."


def _collected(*sites: Site) -> Collected:
    return Collected(sites=sites, disables=(), dead=(), uncovered=())


def test_not_adopted_passes(tmp_path: Path, capsys: pytest.CaptureFixture[str]) -> None:
    assert vrg_exceptions.run_audit(tmp_path, today=TODAY) == 0
    assert "not adopted" in capsys.readouterr().out


def test_invalid_register_fails(tmp_path: Path, capsys: pytest.CaptureFixture[str]) -> None:
    (tmp_path / "vergil-exceptions.toml").write_text("[[rule]\n", encoding="utf-8")
    assert vrg_exceptions.run_audit(tmp_path, today=TODAY) == 1
    err = capsys.readouterr().err
    assert "vergil-exceptions.toml" in err
    assert "invalid TOML" in err


def test_findings_fail_with_location(tmp_path: Path, capsys: pytest.CaptureFixture[str]) -> None:
    (tmp_path / "vergil-exceptions.toml").write_text("", encoding="utf-8")
    site = Site("ruff", "src/a.py", 4, ("S603",), None)
    with mock.patch(_MOD + "collect", return_value=_collected(site)):
        assert vrg_exceptions.run_audit(tmp_path, today=TODAY) == 1
    captured = capsys.readouterr()
    assert "[missing-id]" in captured.err
    assert "src/a.py" in captured.err
    assert "1 finding(s)" in captured.out


def test_clean_audit_passes(tmp_path: Path) -> None:
    (tmp_path / "vergil-exceptions.toml").write_text("", encoding="utf-8")
    with mock.patch(_MOD + "collect", return_value=_collected()):
        assert vrg_exceptions.run_audit(tmp_path, today=TODAY) == 0


def test_tree_error_fails(tmp_path: Path, capsys: pytest.CaptureFixture[str]) -> None:
    from vergil_tooling.lib.exceptions.tree import TreeReadError

    (tmp_path / "vergil-exceptions.toml").write_text("", encoding="utf-8")
    with mock.patch(_MOD + "collect", side_effect=TreeReadError("bad.py: not valid UTF-8")):
        assert vrg_exceptions.run_audit(tmp_path, today=TODAY) == 1
    assert "bad.py" in capsys.readouterr().err


def test_sync_writes_block(tmp_path: Path) -> None:
    (tmp_path / "pyproject.toml").write_text("[tool.ruff.lint]\nselect = [\"E\"]\n", encoding="utf-8")
    with mock.patch(_MOD + "git.repo_root", return_value=tmp_path):
        assert vrg_exceptions.main(["sync"]) == 0
    assert "vergil-managed: fleet-exceptions" in (tmp_path / "pyproject.toml").read_text()


def test_sync_without_pyproject_is_noop(tmp_path: Path, capsys: pytest.CaptureFixture[str]) -> None:
    with mock.patch(_MOD + "git.repo_root", return_value=tmp_path):
        assert vrg_exceptions.main(["sync"]) == 0
    assert "no pyproject.toml" in capsys.readouterr().out


def test_audit_subcommand_uses_repo_root(tmp_path: Path) -> None:
    with (mock.patch(_MOD + "git.repo_root", return_value=tmp_path),
          mock.patch(_MOD + "run_audit", return_value=0) as run):
        assert vrg_exceptions.main(["audit"]) == 0
    assert run.call_args.args[0] == tmp_path
```

Add to `tests/vergil_tooling/test_validate_common.py`, following that file's
existing mocking pattern for the other checks: the exceptions audit runs as the
last check; a non-zero return from `vrg_exceptions.main(["audit"])` is returned
by `validate_common.main`.

- [ ] **Step 2: Run to verify failure**

Run: `vrg-container-run -- uv run pytest tests/vergil_tooling/test_vrg_exceptions.py -q --no-cov`
Expected: FAIL — module missing.

- [ ] **Step 3: Implement the CLI**

```python
# src/vergil_tooling/bin/vrg_exceptions.py
"""Audit, sync, diff and report a repository's exceptions (epic vergil-project/.github#375).

Every inline suppression and rule disable must be documented in
``vergil-exceptions.toml``; ``audit`` is the gate ``vrg-validate`` runs.
"""

from __future__ import annotations

import argparse
from datetime import UTC, datetime
from typing import TYPE_CHECKING

from vergil_tooling.lib import git
from vergil_tooling.lib.cli import cli_main
from vergil_tooling.lib.exceptions.audit import audit, collect
from vergil_tooling.lib.exceptions.fleet import load_fleet
from vergil_tooling.lib.exceptions.managed import ManagedBlockError, check_block, find_block, sync_pyproject
from vergil_tooling.lib.exceptions.plugins import PLUGINS, UNCOVERED, known_tools
from vergil_tooling.lib.exceptions.register import REGISTER_FILE, RegisterError, parse_register
from vergil_tooling.lib.exceptions.tree import TreeReadError, WorkTree
from vergil_tooling.lib.output import emit_error, is_ci, write_summary

if TYPE_CHECKING:
    from datetime import date
    from pathlib import Path

    from vergil_tooling.lib.exceptions.model import AuditResult, Finding, Notice, Register


@cli_main
def main(argv: list[str] | None = None) -> int:
    parser = argparse.ArgumentParser(prog="vrg-exceptions", description=__doc__.splitlines()[0])
    sub = parser.add_subparsers(dest="command", required=True)
    sub.add_parser("audit", help="fail on any undocumented, dead or stale exception")
    sub.add_parser("sync", help="write the fleet-exceptions managed block into pyproject.toml")
    args = parser.parse_args(argv)
    root = git.repo_root()
    if args.command == "audit":
        return run_audit(root, today=datetime.now(tz=UTC).date())
    return run_sync(root)


def load_repo_register(root: Path) -> Register | None:
    path = root / REGISTER_FILE
    if not path.is_file():
        return None
    return parse_register(path.read_text(encoding="utf-8"), source=REGISTER_FILE, known_tools=known_tools())


def run_audit(root: Path, *, today: date) -> int:
    try:
        register = load_repo_register(root)
        if register is None:
            print(f"vrg-exceptions: not adopted (no {REGISTER_FILE}); skipping")
            return 0
        fleet = load_fleet()
        collected = collect(WorkTree(root), PLUGINS, UNCOVERED, root=root)
        pyproject = root / "pyproject.toml"
        findings: list[Finding] = []
        notices: list[Notice] = []
        if pyproject.is_file():
            findings, notices = check_block(find_block(pyproject.read_text(encoding="utf-8")), fleet,
                                            path="pyproject.toml")
        result = audit(collected, register, fleet.register, today=today,
                       extra_findings=findings, extra_notices=notices)
    except RegisterError as exc:
        for problem in exc.problems:
            emit_error(f"{exc.source}: {problem}")
        return 1
    except (TreeReadError, ManagedBlockError) as exc:
        emit_error(str(exc))
        return 1
    _print(result)
    return 0 if result.ok else 1


def _print(result: AuditResult) -> None:
    for f in result.findings:
        emit_error(f"[{f.code}] {f.message}", file=f.path, line=f.line)
    for n in result.notices:
        print(f"note: [{n.code}] {n.message}")
    print(f"vrg-exceptions: {len(result.findings)} finding(s), {len(result.notices)} notice(s)")
    if is_ci():
        rows = "\n".join(f"| `{f.code}` | {f.path or ''}{f':{f.line}' if f.line else ''} | {f.message} |"
                         for f in result.findings)
        notes = "\n".join(f"- `{n.code}`: {n.message}" for n in result.notices)
        write_summary(f"## Exceptions audit\n\n| Code | Where | Message |\n|---|---|---|\n{rows}\n\n{notes}\n")


def run_sync(root: Path) -> int:
    pyproject = root / "pyproject.toml"
    if not pyproject.is_file():
        print("vrg-exceptions: no pyproject.toml; nothing to sync")
        return 0
    try:
        fleet = load_fleet()
        before = pyproject.read_text(encoding="utf-8")
        after = sync_pyproject(before, fleet)
    except (RegisterError, ManagedBlockError) as exc:
        emit_error(str(exc))
        return 1
    if after == before:
        print(f"vrg-exceptions: fleet exceptions already at rev {fleet.revision}")
        return 0
    pyproject.write_text(after, encoding="utf-8")
    print(f"vrg-exceptions: wrote fleet exceptions rev {fleet.revision} into pyproject.toml")
    return 0
```

- [ ] **Step 4: Wire into `validate_common.main`**, just before the final
`return 0`:

```python
    print("Running: exceptions audit")
    rc = vrg_exceptions.main(["audit"])
    if rc != 0:
        return rc
```

with `from vergil_tooling.bin import vrg_exceptions` at the top, and add
`8. vrg-exceptions audit when the repo has a vergil-exceptions.toml (epic
vergil-project/.github#375)` to the module docstring list.

- [ ] **Step 5: Run tests, full validation, commit**

Run: `vrg-container-run -- uv run pytest tests/vergil_tooling/test_vrg_exceptions.py tests/vergil_tooling/test_validate_common.py -q --no-cov` — Expected: PASS.
Run: `vrg-container-run -- vrg-validate` — Expected: PASS (prints "not adopted" for vergil-tooling), coverage 100%.
Also add `vrg-exceptions` to `docs/site/docs/reference/cli-tools-overview.md` and
`docs/site/docs/reference/index.md`, matching the neighbouring entries' format.

```bash
vrg-git add src/vergil_tooling/bin/vrg_exceptions.py src/vergil_tooling/bin/validate_common.py pyproject.toml tests/vergil_tooling docs/site/docs/reference
vrg-commit --type feat --scope exceptions --message "add vrg-exceptions audit and sync, run audit in vrg-validate" --body "Ref #<task>"
```

---

## Task T6: `vrg-exceptions diff` and the `vrg-submit-pr` preview section

**Repo:** vergil-tooling. Blocked by: T5. Spec: §9 (PR preview), pushback finding [4].

**Files:**

- Create: `src/vergil_tooling/lib/exceptions/diff.py`
- Modify: `src/vergil_tooling/bin/vrg_exceptions.py` (add `diff` subcommand)
- Modify: `src/vergil_tooling/bin/vrg_submit_pr.py` (`_open_pr`, around line 697)
- Test: `tests/vergil_tooling/test_exceptions_diff.py`, extend `tests/vergil_tooling/test_vrg_exceptions.py`
  and `tests/vergil_tooling/test_vrg_submit_pr.py`

**Interfaces:**

- Consumes: `GitTree`, `TreeReadError`, `collect`, `parse_register`,
  `REGISTER_FILE`, `PLUGINS`, `UNCOVERED`, `known_tools`, `Entry`, `Site`, `ConfigDisable`.
- Produces:

```python
# diff.py
@dataclass(frozen=True) class Snapshot: register: Register; sites: tuple[Site, ...]; disables: tuple[ConfigDisable, ...]
@dataclass(frozen=True) class ExceptionDiff: added_entries: tuple[Entry, ...]; removed_entries: tuple[Entry, ...]; changed_entries: tuple[tuple[Entry, Entry], ...]; added_sites: tuple[Site, ...]; removed_sites: tuple[Site, ...]; added_disables: tuple[ConfigDisable, ...]; removed_disables: tuple[ConfigDisable, ...]; empty -> bool (property)
def snapshot(tree: RepoTree) -> Snapshot                 # register may be absent → empty Register
def diff(base: Snapshot, head: Snapshot) -> ExceptionDiff
def render(d: ExceptionDiff) -> str                       # Markdown; "" when empty
def diff_refs(base_ref: str, head_ref: str, *, repo_root: Path) -> ExceptionDiff   # TreeReadError on a bad ref

# vrg_submit_pr.py
def _exceptions_preview(branch: str, base: str, *, push: bool) -> str   # SystemExit on failure
```

Rules:

- Sites compare **without line numbers** (lines shift with unrelated edits): a
  multiset of `(tool, path, rules, exc_id)`. Added = head − base, removed = base − head.
- An entry is "changed" when its ID exists on both sides with any field
  different; the renderer calls out a widened `scope` explicitly
  (`scope widened: + examples/**`).
- Snapshots use `collect(..., root=None)` — dead detection needs a working tree
  and is not part of the diff.
- A repo with no register on either side still diffs sites and disables: a new
  suppression is worth showing before adoption, too.

- [ ] **Step 1: Write failing tests**

```python
# tests/vergil_tooling/test_exceptions_diff.py
import subprocess
from datetime import date
from pathlib import Path

import pytest

from tests.vergil_tooling.exceptions_fakes import FakeTree
from vergil_tooling.lib.exceptions.diff import diff, diff_refs, render, snapshot
from vergil_tooling.lib.exceptions.tree import TreeReadError

REG_BASE = ('[[rule]]\nid = "EXC-001"\ntool = "ruff"\nrules = ["S101"]\nscope = ["tests/**"]\n'
            'reason = "pytest"\nadded = 2026-10-20\n')
PY = "x = 1  # noqa: S603  # vrg-exc: EXC-002\n"


def test_identical_trees_are_empty() -> None:
    snap = snapshot(FakeTree({"vergil-exceptions.toml": REG_BASE, "a.py": PY}))
    d = diff(snap, snap)
    assert d.empty
    assert render(d) == ""


def test_new_site_ignores_line_shift() -> None:
    base = snapshot(FakeTree({"a.py": PY}))
    head = snapshot(FakeTree({"a.py": "\n\n" + PY + "y = 2  # noqa: S607\n"}))
    d = diff(base, head)
    assert [(s.rules, s.exc_id) for s in d.added_sites] == [(("S607",), None)]
    assert d.removed_sites == ()


def test_widened_scope_is_called_out() -> None:
    head_reg = REG_BASE.replace('["tests/**"]', '["tests/**", "examples/**"]')
    d = diff(snapshot(FakeTree({"vergil-exceptions.toml": REG_BASE})),
             snapshot(FakeTree({"vergil-exceptions.toml": head_reg})))
    assert len(d.changed_entries) == 1
    assert "scope widened: + examples/**" in render(d)


def test_render_lists_added_entry_with_reason() -> None:
    d = diff(snapshot(FakeTree({})), snapshot(FakeTree({"vergil-exceptions.toml": REG_BASE})))
    text = render(d)
    assert "EXC-001" in text
    assert "pytest" in text


def _git(root: Path, *args: str) -> None:
    subprocess.run(["git", "-C", str(root), *args], check=True, capture_output=True)  # noqa: S603, S607


def test_diff_refs_reads_objects_not_worktree(tmp_path: Path) -> None:
    _git(tmp_path, "init", "-q", "-b", "develop")
    _git(tmp_path, "config", "user.email", "t@example.com")
    _git(tmp_path, "config", "user.name", "t")
    (tmp_path / "a.py").write_text("x = 1\n", encoding="utf-8")
    _git(tmp_path, "add", "a.py")
    _git(tmp_path, "commit", "-q", "-m", "base")
    _git(tmp_path, "checkout", "-q", "-b", "feature/x")
    (tmp_path / "a.py").write_text("x = 1  # noqa: S603\n", encoding="utf-8")
    _git(tmp_path, "commit", "-q", "-am", "head")
    _git(tmp_path, "checkout", "-q", "develop")          # worktree is on develop, like the Mac main worktree
    d = diff_refs("develop", "feature/x", repo_root=tmp_path)
    assert [s.rules for s in d.added_sites] == [("S603",)]


def test_diff_refs_bad_ref_is_error(tmp_path: Path) -> None:
    _git(tmp_path, "init", "-q")
    with pytest.raises(TreeReadError, match="origin/nope"):
        diff_refs("origin/nope", "HEAD", repo_root=tmp_path)
```

Add to `test_vrg_submit_pr.py`, in the style of `test_open_pr_relay_opens_without_pushing`:

```python
def test_open_pr_prints_exception_preview(capsys: pytest.CaptureFixture[str]) -> None:
    metadata = {"issue": "42", "title": "fix: bug", "summary": "s", "notes": "", "linkage": "Ref"}
    with (
        patch(_MOD + ".resolve_issue_ref", return_value="#42"),
        patch(_MOD + ".build_pr_body", return_value="BODY"),
        patch(_MOD + "._exceptions_preview", return_value="### Exceptions added or changed in this PR\n- x") as prev,
        patch(_MOD + ".github.create_pr", return_value="https://github.com/pr/9"),
    ):
        _open_pr("feature/x", "develop", metadata, push=False, assume_yes=True)
    prev.assert_called_once_with("feature/x", "develop", push=False)
    assert "Exceptions added or changed in this PR" in capsys.readouterr().out


def test_exceptions_preview_relay_fetches_and_diffs_origin_refs() -> None:
    with (
        patch(_MOD + ".git.run") as run,
        patch(_MOD + ".git.repo_root", return_value=Path("/r")),
        patch(_MOD + ".diff_refs") as diff_refs,
        patch(_MOD + ".render_diff", return_value=""),
    ):
        assert _exceptions_preview("feature/x", "develop", push=False) == \
            "=== Exceptions added or changed in this PR ===\n(none)"
    run.assert_called_once_with("fetch", "origin", "develop", "feature/x")
    assert diff_refs.call_args.args[:2] == ("origin/develop", "origin/feature/x")


def test_exceptions_preview_local_uses_local_head() -> None:
    with (
        patch(_MOD + ".git.run"),
        patch(_MOD + ".git.repo_root", return_value=Path("/r")),
        patch(_MOD + ".diff_refs") as diff_refs,
        patch(_MOD + ".render_diff", return_value="- EXC-001 added"),
    ):
        out = _exceptions_preview("feature/x", "develop", push=True)
    assert diff_refs.call_args.args[:2] == ("origin/develop", "feature/x")
    assert "EXC-001 added" in out


def test_exceptions_preview_bad_ref_aborts() -> None:
    from vergil_tooling.lib.exceptions.tree import TreeReadError

    with (
        patch(_MOD + ".git.run"),
        patch(_MOD + ".git.repo_root", return_value=Path("/r")),
        patch(_MOD + ".diff_refs", side_effect=TreeReadError("cannot resolve git ref 'origin/feature/x'")),
        pytest.raises(SystemExit, match="cannot resolve"),
    ):
        _exceptions_preview("feature/x", "develop", push=False)
```

- [ ] **Step 2: Run to verify failure**

Run: `vrg-container-run -- uv run pytest tests/vergil_tooling/test_exceptions_diff.py -q --no-cov` — Expected: FAIL.

- [ ] **Step 3: Implement `diff.py`**

```python
# src/vergil_tooling/lib/exceptions/diff.py
"""Exceptions added, removed or changed between two git refs (spec §9).

Both sides are read from git objects, never the working tree, so a branch that
only exists as ``origin/<branch>`` (worktree-free cloud submission) diffs the
same as a checked-out one.
"""

from __future__ import annotations

from collections import Counter
from dataclasses import dataclass, fields
from typing import TYPE_CHECKING

from vergil_tooling.lib.exceptions.audit import collect
from vergil_tooling.lib.exceptions.model import ConfigDisable, Entry, Register, Site
from vergil_tooling.lib.exceptions.plugins import PLUGINS, UNCOVERED, known_tools
from vergil_tooling.lib.exceptions.register import REGISTER_FILE, parse_register
from vergil_tooling.lib.exceptions.tree import GitTree

if TYPE_CHECKING:
    from pathlib import Path

    from vergil_tooling.lib.exceptions.tree import RepoTree


@dataclass(frozen=True)
class Snapshot:
    register: Register
    sites: tuple[Site, ...]
    disables: tuple[ConfigDisable, ...]


@dataclass(frozen=True)
class ExceptionDiff:
    added_entries: tuple[Entry, ...]
    removed_entries: tuple[Entry, ...]
    changed_entries: tuple[tuple[Entry, Entry], ...]
    added_sites: tuple[Site, ...]
    removed_sites: tuple[Site, ...]
    added_disables: tuple[ConfigDisable, ...]
    removed_disables: tuple[ConfigDisable, ...]

    @property
    def empty(self) -> bool:
        return not any(getattr(self, f.name) for f in fields(self))


def snapshot(tree: RepoTree) -> Snapshot:
    text = tree.read(REGISTER_FILE)
    register = (
        parse_register(text, source=REGISTER_FILE, known_tools=known_tools())
        if text is not None
        else Register(entries=())
    )
    collected = collect(tree, PLUGINS, UNCOVERED, root=None)
    return Snapshot(register, collected.sites, collected.disables)


def _site_key(s: Site) -> tuple[str, str, tuple[str, ...], str | None]:
    return (s.tool, s.path, s.rules, s.exc_id)


def _multiset_minus(a: tuple[Site, ...], b: tuple[Site, ...]) -> tuple[Site, ...]:
    remaining = Counter(_site_key(s) for s in b)
    out: list[Site] = []
    for site in a:
        key = _site_key(site)
        if remaining[key]:
            remaining[key] -= 1
        else:
            out.append(site)
    return tuple(out)


def diff(base: Snapshot, head: Snapshot) -> ExceptionDiff:
    base_ids = {e.id: e for e in base.register.entries}
    head_ids = {e.id: e for e in head.register.entries}
    return ExceptionDiff(
        added_entries=tuple(e for i, e in head_ids.items() if i not in base_ids),
        removed_entries=tuple(e for i, e in base_ids.items() if i not in head_ids),
        changed_entries=tuple((base_ids[i], e) for i, e in head_ids.items() if i in base_ids and base_ids[i] != e),
        added_sites=_multiset_minus(head.sites, base.sites),
        removed_sites=_multiset_minus(base.sites, head.sites),
        added_disables=tuple(d for d in head.disables if d not in base.disables),
        removed_disables=tuple(d for d in base.disables if d not in head.disables),
    )


def _entry_line(e: Entry) -> str:
    scope = f" scope {', '.join(e.scope)}" if e.scope else ""
    return f"- `{e.id}` {e.level} {e.tool} {', '.join(e.rules)}{scope} — {e.reason}"


def render(d: ExceptionDiff) -> str:
    if d.empty:
        return ""
    out: list[str] = []
    if d.added_entries:
        out += ["**New register entries**", *(_entry_line(e) for e in d.added_entries), ""]
    if d.changed_entries:
        out.append("**Changed register entries**")
        for old, new in d.changed_entries:
            out.append(_entry_line(new))
            widened = sorted(set(new.scope) - set(old.scope))
            if widened:
                out.append(f"  - scope widened: + {', '.join(widened)}")
        out.append("")
    if d.removed_entries:
        out += ["**Removed register entries**", *(f"- `{e.id}`" for e in d.removed_entries), ""]
    if d.added_sites:
        out.append("**New suppression sites**")
        out += [f"- {s.path}:{s.line} {s.tool} {', '.join(s.rules) or '(blanket)'} → {s.exc_id or 'NO ID'}"
                for s in d.added_sites]
        out.append("")
    if d.removed_sites:
        out.append(f"**Removed suppression sites:** {len(d.removed_sites)}")
        out.append("")
    if d.added_disables:
        out.append("**New config disables**")
        out += [f"- {x.tool} {x.rule} for {', '.join(x.scope)} ({x.source})" for x in d.added_disables]
        out.append("")
    if d.removed_disables:
        out.append(f"**Removed config disables:** {len(d.removed_disables)}")
    return "\n".join(out).rstrip() + "\n"


def diff_refs(base_ref: str, head_ref: str, *, repo_root: Path) -> ExceptionDiff:
    return diff(snapshot(GitTree(base_ref, repo_root=repo_root)), snapshot(GitTree(head_ref, repo_root=repo_root)))
```

- [ ] **Step 4: Add the `diff` subcommand** to `vrg_exceptions.main`:
`sub.add_parser("diff")` with required `--base` and `--head`; runs
`diff_refs(args.base, args.head, repo_root=git.repo_root())`, prints `render(...)`
or `vrg-exceptions: no exception changes between <base> and <head>`; on
`TreeReadError`/`RegisterError` → `emit_error`, return 1. Add a CLI test for each
of the three outcomes.

- [ ] **Step 5: Add the preview to `vrg-submit-pr`**

```python
# src/vergil_tooling/bin/vrg_submit_pr.py
from vergil_tooling.lib.exceptions.diff import diff_refs
from vergil_tooling.lib.exceptions.diff import render as render_diff
from vergil_tooling.lib.exceptions.register import RegisterError
from vergil_tooling.lib.exceptions.tree import TreeReadError


def _exceptions_preview(branch: str, base: str, *, push: bool) -> str:
    """The "Exceptions added or changed" preview section (epic #375, spec §9).

    The relay path (push=False) has no local branch, so both sides are fetched
    and read as ``origin/<ref>``; the local path diffs the local branch, which is
    not pushed until after the confirm. A ref that cannot be read aborts the
    submission rather than showing an empty section.
    """
    git.run("fetch", "origin", base, *(() if push else (branch,)))
    head = branch if push else f"origin/{branch}"
    try:
        d = diff_refs(f"origin/{base}", head, repo_root=git.repo_root())
    except (TreeReadError, RegisterError) as exc:
        msg = f"vrg-submit-pr: cannot compute the exceptions preview: {exc}"
        raise SystemExit(msg) from exc
    body = render_diff(d)
    return "=== Exceptions added or changed in this PR ===\n" + (body.rstrip() if body else "(none)")
```

In `_open_pr`, after `print(f"\n=== Body Preview ===\n{pr_body}")`, add:

```python
    print("\n" + _exceptions_preview(branch, base, push=push))
```

The relay path fetches `base` and `branch` in one call; the local path fetches
only `base` (its head is the local branch). The section always prints (`(none)` when the diff ran and found nothing), so a
missing section can never be mistaken for "no changes".

- [ ] **Step 6: Run tests, full validation, commit**

Run: `vrg-container-run -- uv run pytest tests/vergil_tooling/test_exceptions_diff.py tests/vergil_tooling/test_vrg_exceptions.py tests/vergil_tooling/test_vrg_submit_pr.py -q --no-cov` — Expected: PASS.
Run: `vrg-container-run -- vrg-validate` — Expected: PASS, coverage 100%.
Document `vrg-exceptions diff` and the new preview section in
`docs/site/docs/reference/cli-tools-overview.md` (the `vrg-exceptions` and
`vrg-submit-pr` entries).

```bash
vrg-git add src/vergil_tooling tests/vergil_tooling docs/site/docs/reference
vrg-commit --type feat --scope exceptions --message "add vrg-exceptions diff and the submit-pr exceptions preview" --body "Ref #<task>"
```

---

## Task T8: `vrg-exceptions report` and the published exceptions page

**Repo:** vergil-tooling. Blocked by: T5. Spec: §9 (docs page). May land after
the docs-build rework; nothing else depends on it.

**Files:**

- Create: `src/vergil_tooling/lib/exceptions/report.py`
- Modify: `src/vergil_tooling/bin/vrg_exceptions.py` (add `report` and `build_report`)
- Modify: `src/vergil_tooling/lib/docs.py` (`stage_docs` takes `exceptions_page`; `patch_nav` takes `exceptions`)
- Modify: `src/vergil_tooling/bin/vrg_docs_stage.py`, `src/vergil_tooling/bin/vrg_docs_patch_nav.py`
- Test: `tests/vergil_tooling/test_exceptions_report.py`, extend
  `tests/vergil_tooling/test_vrg_exceptions.py`, `tests/vergil_tooling/test_docs.py`,
  `tests/vergil_tooling/test_vrg_docs_stage.py`, `tests/vergil_tooling/test_vrg_docs_patch_nav.py`

**Interfaces:**

- Consumes: `Register`, `Collected`, `AuditResult`, `Fleet`, `load_repo_register`,
  `load_fleet`, `collect`, `audit`, `WorkTree`, `find_block`, `check_block`.
- Produces:

```python
# report.py
def render_report(register: Register, collected: Collected, fleet: Fleet, result: AuditResult, *, repo: str, block_revision: int | None) -> str
# bin/vrg_exceptions.py
def build_report(root: Path, *, repo: str, today: date) -> str | None   # None when not adopted; RegisterError/TreeReadError/ManagedBlockError propagate
# lib/docs.py (new keyword-only parameters, default off)
def stage_docs(*, ..., exceptions_page: str | None = None) -> int
def patch_nav(mkdocs_yml: Path, releases_dir: Path, *, exceptions: bool = False) -> None
```

Page content, in order (spec §9):

1. Title `# Exceptions` and a one-paragraph explanation linking to the standard.
2. **Rule-level exceptions** — table: ID, tool, rules, scope, reason.
3. **Line-level exceptions** — table: ID, tool, rules, sites (count), reason.
4. **Inherited fleet defaults** — table: tool, rules, scope, reason (fleet
   entries have no ID; spec §5.3), then "Fleet exceptions in this repo: rev N
   (current rev M)", or "no managed block (current rev M)".
5. **Expiring** — entries with `expires`, soonest first, with the tracking issue.
6. **Not yet audited** — the uncovered mechanisms.

- [ ] **Step 1: Write failing tests** — one per section above, using a small
  `Register`/`Collected`/`Fleet` built in the test (as in T1/T4 tests), asserting
  the section heading and a representative row; plus: expiring entries sorted
  soonest first; empty sections render `_None._` rather than disappearing.

```python
# tests/vergil_tooling/test_exceptions_report.py
from datetime import date

from vergil_tooling.lib.exceptions.audit import Collected
from vergil_tooling.lib.exceptions.fleet import Fleet
from vergil_tooling.lib.exceptions.managed import body_digest
from vergil_tooling.lib.exceptions.model import AuditResult, Entry, Register, Site
from vergil_tooling.lib.exceptions.report import render_report

ADDED = date(2026, 10, 20)
FLEET = Fleet(revision=2, ruff_rules=(), history={1: body_digest([]), 2: body_digest([])},
              register=Register(entries=(Entry(id="fleet:markdownlint:MD060", level="rule", tool="markdownlint",
                                               rules=("MD060",), added=ADDED, reason="tables", scope=("**",)),)))


def _report(entries: tuple[Entry, ...], sites: tuple[Site, ...] = (), uncovered: tuple[str, ...] = (),
            block_revision: int | None = 1) -> str:
    return render_report(Register(entries=entries), Collected(sites, (), (), uncovered), FLEET,
                         AuditResult((), ()), repo="vergil-project/x", block_revision=block_revision)


def test_rule_section_lists_scope_and_reason() -> None:
    text = _report((Entry(id="EXC-001", level="rule", tool="ruff", rules=("S101",), added=ADDED,
                          reason="pytest asserts", scope=("tests/**",)),))
    assert "## Rule-level exceptions" in text
    assert "| EXC-001 | ruff | S101 | tests/** | pytest asserts |" in text


def test_inline_section_counts_sites() -> None:
    entry = Entry(id="EXC-014", level="inline", tool="ruff", rules=("S603",), added=ADDED, reason="fixed argv")
    sites = tuple(Site("ruff", "a.py", n, ("S603",), "EXC-014") for n in (1, 2))
    assert "| EXC-014 | ruff | S603 | 2 | fixed argv |" in _report((entry,), sites)


def test_expiring_sorted_soonest_first() -> None:
    def cve(eid: str, exp: date) -> Entry:
        return Entry(id=eid, level="inline", tool="trivy", rules=("CVE-1",), added=ADDED, reason="r",
                     expires=exp, issue="o/r#1")
    text = _report((cve("EXC-002", date(2027, 3, 1)), cve("EXC-001", date(2027, 1, 1))))
    section = text[text.index("## Expiring"):]
    assert section.index("EXC-001") < section.index("EXC-002")


def test_fleet_section_lists_tool_and_rule_without_ids() -> None:
    text = _report(())
    assert "## Inherited fleet defaults" in text
    assert "| markdownlint | MD060 | ** | tables |" in text
    assert "fleet:" not in text                      # the internal label is never shown
    assert "rev 1 (current rev 2)" in text


def test_fleet_section_without_block() -> None:
    assert "no managed block (current rev 2)" in _report((), block_revision=None)


def test_uncovered_section() -> None:
    text = _report((), uncovered=("eslint",))
    assert "## Not yet audited" in text
    assert "- eslint" in text


def test_pipe_in_reason_is_escaped() -> None:
    entry = Entry(id="EXC-001", level="inline", tool="ruff", rules=("S603",), added=ADDED, reason="a | b")
    sites = (Site("ruff", "a.py", 1, ("S603",), "EXC-001"),)
    assert r"| a \| b |" in _report((entry,), sites)


def test_empty_sections_say_none() -> None:
    text = _report(())
    for heading in ("## Rule-level exceptions", "## Line-level exceptions", "## Expiring", "## Not yet audited"):
        assert text[text.index(heading):].split("\n## ", 1)[0].rstrip().endswith("_None._")
```

- [ ] **Step 2: Run to verify failure**

Run: `vrg-container-run -- uv run pytest tests/vergil_tooling/test_exceptions_report.py -q --no-cov`
Expected: FAIL — `ModuleNotFoundError: vergil_tooling.lib.exceptions.report`.

- [ ] **Step 3: Implement `report.py`**

```python
# src/vergil_tooling/lib/exceptions/report.py
"""Render a repo's exceptions as a Markdown docs page (spec §9)."""

from __future__ import annotations

from collections import Counter
from typing import TYPE_CHECKING

if TYPE_CHECKING:
    from collections.abc import Sequence

    from vergil_tooling.lib.exceptions.audit import Collected
    from vergil_tooling.lib.exceptions.fleet import Fleet
    from vergil_tooling.lib.exceptions.model import AuditResult, Register

STANDARD_URL = "https://vergil-project.github.io/vergil-tooling/standards/development/exceptions/"
NONE = "_None._"


def _cell(text: str) -> str:
    return text.replace("|", r"\|").replace("\n", " ")


def _table(header: Sequence[str], rows: Sequence[Sequence[str]]) -> list[str]:
    if not rows:
        return [NONE]
    out = ["| " + " | ".join(header) + " |", "|" + "---|" * len(header)]
    out += ["| " + " | ".join(_cell(c) for c in row) + " |" for row in rows]
    return out


def render_report(
    register: Register,
    collected: Collected,
    fleet: Fleet,
    result: AuditResult,
    *,
    repo: str,
    block_revision: int | None,
) -> str:
    counts = Counter(s.exc_id for s in collected.sites if s.exc_id is not None)
    rules = [e for e in register.entries if e.level == "rule"]
    inline = [e for e in register.entries if e.level == "inline"]
    # ISO dates sort correctly as text, which keeps the key a plain str for mypy.
    expiring = sorted((e for e in register.entries if e.expires is not None), key=lambda e: (str(e.expires), e.id))
    out = [
        "# Exceptions",
        "",
        f"Every exception to a quality or security gate in `{repo}` is documented in "
        f"`vergil-exceptions.toml` and checked by `vrg-exceptions audit` "
        f"([the exceptions standard]({STANDARD_URL})). The goal is zero.",
        "",
        "## Rule-level exceptions",
        "",
        *_table(("ID", "Tool", "Rules", "Scope", "Reason"),
                [(e.id, e.tool, ", ".join(e.rules), ", ".join(e.scope), e.reason) for e in rules]),
        "",
        "## Line-level exceptions",
        "",
        *_table(("ID", "Tool", "Rules", "Sites", "Reason"),
                [(e.id, e.tool, ", ".join(e.rules), str(counts[e.id]), e.reason) for e in inline]),
        "",
        "## Inherited fleet defaults",
        "",
        *_table(("Tool", "Rules", "Scope", "Reason"),
                [(e.tool, ", ".join(e.rules), ", ".join(e.scope), e.reason) for e in fleet.register.entries]),
        "",
        (f"Fleet exceptions in this repo: rev {block_revision} (current rev {fleet.revision})."
         if block_revision is not None else f"Fleet exceptions in this repo: no managed block (current rev "
         f"{fleet.revision})."),
        "",
        "## Expiring",
        "",
        *_table(("ID", "Expires", "Issue", "Reason"),
                [(e.id, str(e.expires), e.issue or "", e.reason) for e in expiring]),
        "",
        "## Not yet audited",
        "",
        *([f"- {name}" for name in collected.uncovered] or [NONE]),
    ]
    if result.findings:
        out += ["", f"> **Audit status:** {len(result.findings)} finding(s) — run `vrg-exceptions audit`."]
    return "\n".join(out) + "\n"
```

Run the report tests — Expected: PASS.

- [ ] **Step 4: Add `build_report` and the `report` subcommand (tests first)**

Tests in `tests/vergil_tooling/test_vrg_exceptions.py`, in the style of T5's
tests there (`tmp_path` as the root, `collect` mocked via `_MOD`):

```python
def test_report_not_adopted_prints_one_line(tmp_path: Path, capsys: pytest.CaptureFixture[str]) -> None:
    assert vrg_exceptions.run_report(tmp_path, repo="o/r", output=None, today=TODAY) == 0
    out = capsys.readouterr().out
    assert "not adopted" in out
    assert "# Exceptions" in out


def test_report_writes_page(tmp_path: Path) -> None:
    (tmp_path / "vergil-exceptions.toml").write_text(
        '[[inline]]\nid = "EXC-001"\ntool = "ruff"\nrules = ["S603"]\nreason = "fixed argv"\n'
        "added = 2026-10-20\n", encoding="utf-8")
    page = tmp_path / "page.md"
    with mock.patch(_MOD + "collect", return_value=_collected(Site("ruff", "a.py", 1, ("S603",), "EXC-001"))):
        assert vrg_exceptions.run_report(tmp_path, repo="o/r", output=page, today=TODAY) == 0
    text = page.read_text(encoding="utf-8")
    assert "| EXC-001 | ruff | S603 | 1 | fixed argv |" in text
    assert "no managed block" in text                 # tmp_path has no pyproject.toml


def test_report_malformed_register_fails(tmp_path: Path, capsys: pytest.CaptureFixture[str]) -> None:
    (tmp_path / "vergil-exceptions.toml").write_text("[[rule]\n", encoding="utf-8")
    assert vrg_exceptions.run_report(tmp_path, repo="o/r", output=None, today=TODAY) == 1
    assert "invalid TOML" in capsys.readouterr().err


def test_main_dispatches_report(tmp_path: Path) -> None:
    with mock.patch(_MOD + "git.repo_root", return_value=tmp_path), \
         mock.patch(_MOD + "run_report", return_value=0) as run:
        assert vrg_exceptions.main(["report", "--repo", "o/r"]) == 0
    assert run.call_args.kwargs["repo"] == "o/r"
```

Implementation in `bin/vrg_exceptions.py`:

```python
# in main(), beside the other subparsers
    report = sub.add_parser("report", help="render the exceptions docs page (Markdown)")
    report.add_argument("--repo", required=True, help="owner/name shown on the page")
    report.add_argument("--output", type=Path, default=None, help="write here instead of stdout")
# … and the dispatch
    if args.command == "report":
        return run_report(root, repo=args.repo, output=args.output, today=datetime.now(tz=UTC).date())


def build_report(root: Path, *, repo: str, today: date) -> str | None:
    """The docs page for *root*, or None when the repo has no register."""
    register = load_repo_register(root)
    if register is None:
        return None
    fleet = load_fleet()
    collected = collect(WorkTree(root), PLUGINS, UNCOVERED, root=root)
    pyproject = root / "pyproject.toml"
    block = find_block(pyproject.read_text(encoding="utf-8")) if pyproject.is_file() else None
    findings, notices = check_block(block, fleet, path="pyproject.toml")
    result = audit(collected, register, fleet.register, today=today, extra_findings=findings,
                   extra_notices=notices)
    return render_report(register, collected, fleet, result, repo=repo,
                         block_revision=block.revision if block is not None else None)


def run_report(root: Path, *, repo: str, output: Path | None, today: date) -> int:
    try:
        page = build_report(root, repo=repo, today=today)
    except RegisterError as exc:
        for problem in exc.problems:
            emit_error(f"{exc.source}: {problem}")
        return 1
    except (TreeReadError, ManagedBlockError) as exc:
        emit_error(str(exc))
        return 1
    if page is None:
        page = f"# Exceptions\n\n`{repo}` has not adopted the exception register (no {REGISTER_FILE}).\n"
        print(f"vrg-exceptions: not adopted (no {REGISTER_FILE})")
    if output is None:
        print(page, end="")
    else:
        output.write_text(page, encoding="utf-8")
    return 0
```

`Path` moves from the `TYPE_CHECKING` block to a runtime import (argparse uses
it). Add `from vergil_tooling.lib.exceptions.report import render_report`.

- [ ] **Step 5: Publish the page from the docs build (tests first)**

`lib/docs.py` stays pure: the caller renders the page and passes it in.

```python
# tests/vergil_tooling/test_docs.py — TestStageDocs
    def test_writes_exceptions_page_when_given(self, tmp_path: Path) -> None:
        docs_dir = tmp_path / "docs"
        docs_dir.mkdir()
        stage_docs(docs_dir=docs_dir, releases_dir=tmp_path / "none", changelog=None,
                   exceptions_page="# Exceptions\n")
        assert (docs_dir / "exceptions.md").read_text(encoding="utf-8") == "# Exceptions\n"

    def test_no_exceptions_page_by_default(self, tmp_path: Path) -> None:
        docs_dir = tmp_path / "docs"
        docs_dir.mkdir()
        stage_docs(docs_dir=docs_dir, releases_dir=tmp_path / "none", changelog=None)
        assert not (docs_dir / "exceptions.md").exists()

# TestPatchNav
    def test_adds_exceptions_entry_once(self, tmp_path: Path) -> None:
        mkdocs = tmp_path / "mkdocs.yml"
        mkdocs.write_text(_NAV_TEMPLATE)
        releases = tmp_path / "releases"
        releases.mkdir()
        patch_nav(mkdocs, releases, exceptions=True)
        patch_nav(mkdocs, releases, exceptions=True)
        content = mkdocs.read_text()
        assert content.count("- Exceptions: exceptions.md") == 1
        assert content.index("- Changelog:") < content.index("- Exceptions:") < content.index("- Release Notes:")

    def test_exceptions_entry_off_by_default(self, tmp_path: Path) -> None:
        mkdocs = tmp_path / "mkdocs.yml"
        mkdocs.write_text(_NAV_TEMPLATE)
        releases = tmp_path / "releases"
        releases.mkdir()
        patch_nav(mkdocs, releases)
        assert "Exceptions" not in mkdocs.read_text()
```

```python
# lib/docs.py — stage_docs: add the keyword parameter and, before `return`:
    exceptions_page: str | None = None,
    ...
    if exceptions_page is not None:
        (docs_dir / "exceptions.md").write_text(exceptions_page, encoding="utf-8")

# lib/docs.py — patch_nav(mkdocs_yml, releases_dir, *, exceptions: bool = False):
# after the release entries are spliced in, and before writing:
    if exceptions and not any(ln.strip() == "- Exceptions: exceptions.md" for ln in lines):
        notes = next((i for i, ln in enumerate(lines) if ln.lstrip().startswith("- Release Notes:")), None)
        if notes is not None:
            pad = lines[notes][: len(lines[notes]) - len(lines[notes].lstrip())]
            lines.insert(notes, f"{pad}- Exceptions: exceptions.md")
```

`patch_nav` returns early when there is no `releases/index.md` entry; move the
exceptions insertion above that early return so it applies either way, and add
a test with a nav that has `- Release Notes:` but no index line.

Callers:

- `bin/vrg_docs_stage.py`: before `stage_docs`, compute
  `page = vrg_exceptions.build_report(git.repo_root(), repo=repo, today=datetime.now(tz=UTC).date())`
  and pass `exceptions_page=page`. `RegisterError`, `TreeReadError` and
  `ManagedBlockError` are not caught there: a broken register fails the docs
  build loudly (`cli_main` reports it). Test: a repo with a register stages
  `exceptions.md`; a repo without one does not (mock `build_report`).
- `bin/vrg_docs_patch_nav.py`: pass
  `exceptions=(args.releases_dir.parent / "exceptions.md").is_file()` — the
  releases dir is `<docs>/releases`, so the page sits beside it. Test both values.

Run: `vrg-container-run -- uv run pytest tests/vergil_tooling/test_docs.py tests/vergil_tooling/test_vrg_docs_stage.py tests/vergil_tooling/test_vrg_docs_patch_nav.py tests/vergil_tooling/test_vrg_exceptions.py -q --no-cov`
Expected: PASS.

- [ ] **Step 6: Full validation and commit**

Run: `vrg-container-run -- vrg-validate` — Expected: PASS, coverage 100%.

```bash
vrg-git add src/vergil_tooling tests/vergil_tooling
vrg-commit --type feat --scope exceptions --message "publish a per-repo exceptions page from vrg-exceptions report" --body "Ref #<task>"
```

---

## Task T7a: consumer CLAUDE.md rule and standards docs

**Repo:** vergil-tooling. Blocked by: nothing (text only; it names
`vrg-exceptions` and `/vergil:exception-request`, which land in T5/T7b — merge
order does not matter because the rule is guidance). Spec: §8, §10.

**Files:**

- Modify: `src/vergil_tooling/data/claude_md_consumer.md` (new section)
- Create: `docs/site/docs/standards/development/exceptions.md` (the standard)
- Modify: `docs/site/docs/standards/development/python/testing-and-coverage.md`
  (`## Exceptions` section), `python/type-hints.md` (suppression bullets),
  `deprecation-warnings.md` (`## Warning suppression policy`) — each points to the new standard
- Modify: the standards nav (`docs/site/mkdocs.yml`) to list the new page
- Test: `tests/vergil_tooling/test_repo_config.py` (or whichever test covers
  `claude_md_consumer.md` rendering — find it with `grep -rn claude_md_consumer tests/`)

- [ ] **Step 1: Write the failing test** — assert the rendered consumer CLAUDE.md
  contains the heading `## Exceptions (lint, type, coverage and security suppressions)`
  and the string `/vergil:exception-request`.

- [ ] **Step 2: Run to verify failure** — `vrg-container-run -- uv run pytest <that test file> -q --no-cov` → FAIL.

- [ ] **Step 3: Add the section to `claude_md_consumer.md`:**

```markdown
## Exceptions (lint, type, coverage and security suppressions)

Never add, widen, or reuse an exception without explicit human approval. An
exception is any inline suppression (`# noqa`, `# type: ignore`,
`# pragma: no cover`, `# nosemgrep`, `# shellcheck disable=`,
`<!-- markdownlint-disable -->`, `.trivyignore` entries, …) or any rule disabled
in tool configuration. Use `/vergil:exception-request`: fix the code first; if a
fix is not practical, ask the human with the proposed register entry and wait for
an explicit yes. Every new site needs approval, even when it reuses an existing
entry. Widening a rule-level scope is an exception change too.

Approved exceptions are documented in `vergil-exceptions.toml` and tagged in code
with a separate `# vrg-exc: EXC-NNN` comment. `vrg-validate` runs
`vrg-exceptions audit`, which fails on any undocumented exception. See the
exceptions standard: <https://vergil-project.github.io/vergil-tooling/standards/development/exceptions/>.
```

(Verify the docs URL pattern against an existing link in the same template
before committing.)

- [ ] **Step 4: Write `standards/development/exceptions.md`** — the standard,
  condensed from spec §3–§10: what counts as an exception; the three levels;
  minimum scope (directories over files, repo-wide written `["**"]`); the
  register format and field rules (copy the §5.1 table); the marker form (§5.2,
  including the Markdown `<!-- vrg-exc: … -->` form and the `.trivyignore` group
  rule from T3); what the audit fails on and what it only reports (§6.1/§6.2);
  temporary-by-default and `expires`; how to request one (the skill); adoption is
  cleanup, not grandfathering (§10).

- [ ] **Step 5: Re-point the three existing docs.** Replace each local
  suppression rule with one sentence and a link to the new standard, keeping any
  tool-specific detail that is still true (e.g. the `pragma: no cover - <reason>`
  convention becomes "documented in `vergil-exceptions.toml`; the inline reason
  comment is optional").

- [ ] **Step 6: Validate and commit** — `vrg-container-run -- vrg-validate` → PASS.

```bash
vrg-git add src/vergil_tooling/data/claude_md_consumer.md docs/site tests/vergil_tooling
vrg-commit --type docs --scope exceptions --message "add the exceptions standard and the consumer agent rule" --body "Ref #<task>"
```

---

## Task T7b: `vergil:exception-request` skill and the suppression-detection hook

**Repo:** vergil-claude-plugin. Blocked by: nothing. Spec: §8.

**Files:**

- Create: `skills/exception-request/SKILL.md`
- Create: `hooks/scripts/detect-new-suppressions.sh`
- Create: `hooks/tests/detect-new-suppressions.test.sh`
- Modify: `hooks/hooks.json` (PostToolUse entry for `Edit|Write|MultiEdit`)
- Modify: `skills/issue-implement/SKILL.md`, `skills/epic-implement/SKILL.md` (cross-reference)
- Modify: `docs/site/docs/skills/index.md` (list the skill)

- [ ] **Step 1: Write the failing hook test** (`hooks/tests/detect-new-suppressions.test.sh`),
  table-driven in the style of `guard-audit-writes.test.sh` (parallel indexed
  arrays, bash 3.2-compatible). Each case pipes a PostToolUse JSON payload
  (`tool_name`, `tool_input.new_string` / `tool_input.content`, `cwd` of a temp
  dir containing `vergil.toml`) into the hook and asserts whether stdout contains
  `exception-request`:

| Case | tool_name | new text | expect |
|---|---|---|---|
| noqa added | Edit | `x = 1  # noqa: S603` | remind |
| type ignore added | Write | `y: int = ""  # type: ignore[assignment]` | remind |
| pragma added | Edit | `if x:  # pragma: no cover` | remind |
| nosemgrep added | Edit | `secrets: inherit  # nosemgrep: r` | remind |
| shellcheck added | Write | `# shellcheck disable=SC2086` | remind |
| markdownlint added | Edit | `<!-- markdownlint-disable MD013 -->` | remind |
| marker already present | Edit | `x  # noqa: S603  # vrg-exc: EXC-001` | silent |
| register edit | Edit (file_path `vergil-exceptions.toml`) | `[[inline]]` | silent |
| no suppression | Edit | `x = 1` | silent |
| unmanaged repo | Edit (cwd without `vergil.toml`) | `# noqa` | silent |

  "marker already present" is silent because a tagged site means the approval
  conversation already happened (the audit still checks the ID is real).

- [ ] **Step 2: Run to verify failure** — `bash hooks/tests/detect-new-suppressions.test.sh` → FAIL (hook missing).

- [ ] **Step 3: Write the hook**

```bash
#!/usr/bin/env bash
# detect-new-suppressions.sh — PostToolUse hook for Edit/Write/MultiEdit.
# When an edit introduces an exception (lint/type/coverage/security
# suppression) without a `vrg-exc:` marker, remind the agent that adding one
# needs explicit human approval via the exception-request skill
# (epic vergil-project/.github#375).
#
# Gated on managed-repo detection: no-op in repos that lack vergil.toml.
set -euo pipefail

input=$(cat)
cwd=$(echo "$input" | jq -r '.cwd // "."')

SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
# shellcheck source=/dev/null
source "$SCRIPT_DIR/lib/managed-repo-check.sh"
if ! is_managed_repo "$cwd"; then
  exit 0
fi

file_path=$(echo "$input" | jq -r '.tool_input.file_path // ""')
case "$file_path" in
  */vergil-exceptions.toml|vergil-exceptions.toml) exit 0 ;;
esac

new_text=$(echo "$input" | jq -r '[.tool_input.new_string, .tool_input.content, (.tool_input.edits // [] | .[].new_string)] | map(select(. != null)) | join("\n")')

pattern='(#[[:space:]]*(noqa|type:[[:space:]]*ignore|ty:[[:space:]]*ignore|pragma:[[:space:]]*no[[:space:]]+(cover|branch)|nosemgrep|shellcheck[[:space:]]+disable=|yamllint[[:space:]]+disable|hadolint[[:space:]]+ignore=))|(<!--[[:space:]]*markdownlint-disable)|(//[[:space:]]*nosemgrep)'

untagged=$(echo "$new_text" | grep -E "$pattern" | grep -v 'vrg-exc:' || true)
if [ -z "$untagged" ]; then
  exit 0
fi

jq -n '{
  hookSpecificOutput: {
    hookEventName: "PostToolUse",
    additionalContext: "STOP: this edit adds an exception (a lint/type/coverage/security suppression) with no vrg-exc marker. Adding, widening or reusing an exception needs explicit human approval. Revert it or run the exception-request skill (/vergil:exception-request) now: fix the code first, otherwise ask the human and wait for an explicit yes. vrg-validate will fail on an undocumented exception."
  }
}'
```

Register it in `hooks/hooks.json` under `PostToolUse` as a new matcher block:

```json
{
  "matcher": "Edit|Write|MultiEdit",
  "hooks": [
    {
      "type": "command",
      "command": "${CLAUDE_PLUGIN_ROOT}/hooks/scripts/detect-new-suppressions.sh",
      "statusMessage": "Checking for new exceptions..."
    }
  ]
}
```

- [ ] **Step 4: Run the hook test** — `bash hooks/tests/detect-new-suppressions.test.sh` → PASS.
  Also run shellcheck through `vrg-container-run -- vrg-validate`.

- [ ] **Step 5: Write `skills/exception-request/SKILL.md`**

```markdown
---
name: exception-request
description: Request human approval before adding, widening or reusing an exception — any lint, type, coverage or security suppression (`# noqa`, `# type: ignore`, `# pragma: no cover`, `# nosemgrep`, `# shellcheck disable=`, markdownlint disables, `.trivyignore` entries) or any rule disabled in tool config. Use whenever you are about to suppress a finding instead of fixing it, when the detect-new-suppressions hook fires, or when `vrg-exceptions audit` reports a missing or unknown `vrg-exc` ID.
---

# Exception request

## Rule

You may not add, widen, or reuse an exception on your own. Every new site needs
an explicit "yes" from the human — including a site that reuses an existing
register entry. Silence or an ambiguous reply is a "no".

## Workflow

1. **Fix first.** Try to make the finding go away by changing the code. Show
   the human what you tried, briefly.
2. **If a fix is not practical, ask.** Present, in this order:
   - tool and rule (e.g. `ruff S603`), and the file:line;
   - why fixing is not practical (one or two sentences);
   - the proposed level: **line** (one site) or **rule** (a rule disabled for a
     scope). Rule-level scope is the smallest set of *directories* that needs it;
     prefer directories over individual files; repo-wide (`["**"]`) needs a
     stated reason;
   - the proposed `vergil-exceptions.toml` entry, written out in full — a new
     `EXC-NNN` (next free number), or the existing entry it fits;
   - if the same rule already has line-level entries with several sites, say so:
     it may belong at rule level instead.
3. **Wait** for an explicit yes. Do not continue the task in the meantime.
4. **On yes:** write the entry with the wording you and the human agreed, and tag
   each site with a *separate* comment after the tool's own directive:
   `# noqa: S603  # vrg-exc: EXC-014` (Markdown:
   `<!-- vrg-exc: EXC-014 -->`). Put `# vrg-exc: EXC-014` alone on the line
   above instead for `nosemgrep` (semgrep reads the rest of that comment as rule
   IDs), for a `.trivyignore` group, and for any tool that rejects a second
   comment on the same line. Only `EXC-NNN` IDs exist. Every suppression names
   its rules. Run
   `vrg-container-run -- vrg-validate`; `vrg-exceptions audit` must pass.
5. **On no:** fix the code, or ask how the human wants to proceed.

## Never

- Never add a suppression with no `vrg-exc` marker "for now".
- Never edit a `vergil-managed` block — fleet exceptions change in vergil-tooling.
- Never extend an entry's `rules` or `scope` without asking.
- Never use a blanket suppression (`# noqa` / `# type: ignore` with no rule).
```

- [ ] **Step 6: Cross-reference** — in `skills/issue-implement/SKILL.md`, next to
  each "never suppress a gate" line (lines ~24, ~47, ~154), add: "If a finding
  truly cannot be fixed, use `/vergil:exception-request` — never suppress it
  unilaterally." Same in `skills/epic-implement/SKILL.md` where it describes
  validation. List the skill in `docs/site/docs/skills/index.md`.

- [ ] **Step 7: Validate and commit** — `vrg-container-run -- vrg-validate` → PASS.

```bash
vrg-git add skills hooks docs
vrg-commit --type feat --scope skills --message "add exception-request skill and new-suppression hook" --body "Ref #<task>"
```

---

## Task T9: "kill the exceptions" in vergil-tooling — inventory, mechanical cleanup, purge tasks

**Repo:** vergil-tooling. Blocked by: T1–T6, T7a, T7b. Spec: §10. **Run together
with the human**; every keep/fix/disable decision is theirs.

Spec §10 makes adoption a per-repo **"kill the exceptions"** series: a sequence
of tasks whose only job is to purge exceptions without losing functionality,
ending in one task that documents what is left and turns the gate on. In this
epic the series runs for vergil-tooling and is filed as sibling tasks under
epic #375: **T9** (this task), the **purge tasks** T9 files, and **T9-final**.
Other repos run their own series later, under their own epics.

**Files:**

- Modify: source files carrying dead or auto-fixable suppressions; nothing else.

- [ ] **Step 1: Inventory.** In the worktree, create an empty
  `vergil-exceptions.toml` (not committed), run
  `vrg-container-run -- uv run vrg-exceptions audit`, then delete the file. Group
  the findings by tool × rule × top-level directory into a table (expected ≈54
  ruff groups; S603/S607/S108 ≈75% of `noqa` sites; plus coverage, mypy,
  semgrep — including the generated `secrets-inherit` `nosemgrep` in
  `.github/workflows/cd.yml`).
- [ ] **Step 2: Mechanical cleanup** (no human decision needed): delete every
  `dead` suppression (`RUF100` output — e.g. `ANN001`, `PLR2004`, `PLC0415`,
  `BLE001`, which are not selected); run `ruff check --fix` and `ruff format`
  through `vrg-validate`'s toolchain; fix trivially fixable sites. Commit.
- [ ] **Step 3: Plan the purge with the human.** Walk the remaining groups,
  largest first. For each group, the human decides: **purge** (fix the code —
  file a purge task), **keep** (document it in T9-final, at line level or as a
  narrowly scoped `[[rule]]`), or **propose a fleet default** (a decision for
  vergil-tooling's fleet register, also recorded for T9-final). File one purge
  task per group or small batch of related groups:

  ```bash
  vrg-issue-create --epic vergil-project/.github#375 --repo vergil-project/vergil-tooling \
    --title "Kill the exceptions: <tool> <rule> in <dirs>" --body-file <tmp>
  ```

  Each body names the sites, states "remove every suppression in this group
  without changing behaviour; tests stay at 100% coverage", and records the
  human's decision. Then file **T9-final** (title
  `Kill the exceptions: document the remainder and enable the audit`) with the
  agreed keep list in its body, `--blocked-by` every purge task.
- [ ] **Step 4: Hand off** the cleanup PR with `report-ready`; its notes link
  the inventory table and the filed tasks.

**Each purge task** (filed in Step 3) is an ordinary `issue-implement` task:
fix the code so that the group's suppressions are no longer needed, delete
them, keep `vrg-validate` green. If a site turns out not to be fixable, stop and
ask the human — moving it to the keep list is their call.

**T9-final** (filed in Step 3; blocked by every purge task):

- Create `vergil-exceptions.toml` with the agreed entries: `[[inline]]` for kept
  line-level groups, narrowly scoped `[[rule]]` entries for kept rule disables
  (directories over files).
- Tag every remaining site with `# vrg-exc: EXC-NNN`. This includes the
  `nosemgrep` that repo-init generated in `.github/workflows/cd.yml`: the marker
  goes on the line above, and its entry carries the same reason T10 seeds
  (line-level exceptions are always per repo; spec §5.2).
- Run `vrg-exceptions sync` for the managed block, and get
  `vrg-container-run -- vrg-validate` green with the audit active.
- Hand off with `report-ready`. The PR's own exceptions preview (T6) lists every
  entry — the human reviews it before submitting. The `cd.yml` edit means the
  agent's push is refused (expected; see the handoff rules) and the human's
  `vrg-submit-pr` pushes it.

---

## Task T10: repo-init seeds the register and tags the generated `nosemgrep`

**Repo:** vergil-tooling. Blocked by: T1, T3, T4. Spec: §5.2, §6.4, §10.

`render_cd_workflow` (`lib/repo_init.py:868-878`) emits
`secrets: inherit  # nosemgrep: yaml.github-actions.security.secrets-inherit.secrets-inherit`
for a `[package]` repo that publishes releases. That is a line-level exception
in the new repo, so the new repo documents it: repo-init tags the line and seeds
`vergil-exceptions.toml` with the entry. Every new repo then starts with a
register, so the audit is active from day one (§6.4). Repos bootstrapped before
this task add the entry during their own adoption (§10); vergil-tooling's is in
T9-final.

**Files:**

- Modify: `src/vergil_tooling/lib/repo_init.py`
- Test: `tests/vergil_tooling/test_repo_init.py`

**Interfaces:**

- Consumes: `parse_register`, `REGISTER_FILE` (T1), `PLUGINS`, `UNCOVERED`,
  `known_tools` (T2/T3), `collect`, `audit` (T1), `load_fleet` (T4),
  `FakeTree` (T1 test helper).
- Produces:

```python
SECRETS_INHERIT_EXC_ID = "EXC-001"
def cd_inherits_secrets(ctx: RepoInitContext) -> bool   # ctx.publish_release and ctx.package — the one condition both renderers use
def render_exceptions_register(ctx: RepoInitContext, *, today: date) -> str
```

- [ ] **Step 1: Write the failing tests**

```python
# tests/vergil_tooling/test_repo_init.py — new tests
from datetime import date

from tests.vergil_tooling.exceptions_fakes import FakeTree
from vergil_tooling.lib.exceptions.audit import audit, collect
from vergil_tooling.lib.exceptions.fleet import load_fleet
from vergil_tooling.lib.exceptions.plugins import PLUGINS, UNCOVERED, known_tools
from vergil_tooling.lib.exceptions.register import parse_register
from vergil_tooling.lib.repo_init import render_cd_workflow, render_exceptions_register

TODAY = date(2026, 11, 1)


def _ctx(*, publish_release: bool, package: bool) -> RepoInitContext:
    ctx = RepoInitContext(org="vergil-project", name="vergil-vm")
    ctx.primary_language = "python"
    ctx.ci_versions = ["3.14"]
    ctx.publish_release = publish_release
    ctx.package = package
    return ctx


def _audit(files: dict[str, str]):
    register = parse_register(files["vergil-exceptions.toml"], source="vergil-exceptions.toml",
                              known_tools=known_tools())
    return audit(collect(FakeTree(files), PLUGINS, UNCOVERED, root=None), register, load_fleet().register,
                 today=TODAY)


def test_package_release_repo_gets_tagged_line_and_entry() -> None:
    ctx = _ctx(publish_release=True, package=True)
    cd = render_cd_workflow(ctx)
    lines = cd.splitlines()
    index = next(i for i, ln in enumerate(lines) if "secrets: inherit" in ln)
    assert lines[index - 1].strip() == "# vrg-exc: EXC-001"
    register = render_exceptions_register(ctx, today=TODAY)
    assert 'id      = "EXC-001"' in register
    assert "yaml.github-actions.security.secrets-inherit.secrets-inherit" in register
    result = _audit({".github/workflows/cd.yml": cd, "vergil-exceptions.toml": register})
    assert result.ok, result.findings


def test_other_repos_get_an_empty_register_that_passes() -> None:
    ctx = _ctx(publish_release=True, package=False)
    register = render_exceptions_register(ctx, today=TODAY)
    assert "[[inline]]" not in register
    assert parse_register(register, source="x", known_tools=known_tools()).entries == ()
    assert _audit({".github/workflows/cd.yml": render_cd_workflow(ctx), "vergil-exceptions.toml": register}).ok
```

And in the existing `TestStepScaffoldConfigFiles.test_creates_all_files`, after
the `.gitignore` assertion:

```python
        register = tmp_path / "vergil-exceptions.toml"
        assert register.exists()
        assert parse_register(register.read_text(), source="x", known_tools=known_tools()).entries == ()
```

(That context has `publish_release` False, so the register is the header only.)

Run: `vrg-container-run -- uv run pytest tests/vergil_tooling/test_repo_init.py -q --no-cov`
Expected: FAIL — `render_exceptions_register` is not defined.

- [ ] **Step 2: Implement**

```python
# lib/repo_init.py
SECRETS_INHERIT_EXC_ID = "EXC-001"

_REGISTER_HEADER = """\
# Exception register (vergil-project/.github#375). Every inline suppression and
# rule disable in this repo must have an entry here; `vrg-exceptions audit`
# (run by vrg-validate) fails on anything undocumented. Add entries only with
# explicit human approval (/vergil:exception-request).
"""


def cd_inherits_secrets(ctx: RepoInitContext) -> bool:
    """True when the generated cd.yml uses ``secrets: inherit`` (a [package] release repo)."""
    return ctx.publish_release and ctx.package


def render_exceptions_register(ctx: RepoInitContext, *, today: date) -> str:
    if not cd_inherits_secrets(ctx):
        return _REGISTER_HEADER
    return _REGISTER_HEADER + (
        "\n[[inline]]\n"
        f'id      = "{SECRETS_INHERIT_EXC_ID}"\n'
        'tool    = "semgrep"\n'
        'rules   = ["yaml.github-actions.security.secrets-inherit.secrets-inherit"]\n'
        'reason  = "cd-release needs secrets: inherit. GitHub passes environment secrets to a job in '
        "a cross-repo reusable workflow only under inherit; an explicit map leaves the package-sign "
        'job an empty PACKAGE_SIGNING_KEY (vergil-project/packages#6). The callee is first-party."\n'
        f"added   = {today.isoformat()}\n"
    )
```

In `render_cd_workflow`, replace the `if ctx.package:` condition with
`if cd_inherits_secrets(ctx):` (it is already inside `if ctx.publish_release:`)
and insert `"    # vrg-exc: EXC-001\n"` immediately before the
`"    secrets: inherit"` element (build it from `SECRETS_INHERIT_EXC_ID`). In
`step_scaffold_config_files`, after `.gitignore`:

```python
    # vergil-exceptions.toml — the exception register, so the audit is active
    # from the first commit (epic vergil-project/.github#375).
    today = datetime.datetime.now(tz=datetime.UTC).date()
    (wd / REGISTER_FILE).write_text(render_exceptions_register(ctx, today=today))
```

Import `REGISTER_FILE` from `vergil_tooling.lib.exceptions.register` and `date`
under `TYPE_CHECKING`. Update existing `render_cd_workflow` expectations in
`test_repo_init.py` for the added marker line.

- [ ] **Step 3: Run the tests, full validation, commit**

Run: `vrg-container-run -- uv run pytest tests/vergil_tooling/test_repo_init.py -q --no-cov` — Expected: PASS.
Run: `vrg-container-run -- vrg-validate` — Expected: PASS, coverage 100%.

```bash
vrg-git add src/vergil_tooling/lib/repo_init.py tests/vergil_tooling/test_repo_init.py
vrg-commit --type feat --scope repo-init --message "seed vergil-exceptions.toml and tag the generated secrets-inherit nosemgrep" --body "Ref #<task>"
```

---

## Follow-on task: revisit the generated `secrets: inherit`

**Repo:** vergil-tooling (filed with the other tasks). Not blocking. The
`nosemgrep` that T10 documents works around a GitHub limitation rather than
fixing a defect of ours: environment secrets reach a cross-repo reusable
workflow only under `secrets: inherit` (vergil-project/packages#6). Investigate
whether package signing can be restructured so no generated workflow needs
`inherit` — for example, signing in a same-repo job, or passing the key by
another route. If a design works, remove the suppression from repo-init and
from existing repos; if not, record why on the task and close it.

---

## Task D1 (deployment): release vergil-tooling and vergil-claude-plugin

**Kind:** `deployment`. **Repo:** vergil-tooling. Blocked by: T1–T8, T10, T7a, T7b.
Precondition (human-attested, never agent-performed): a vergil-tooling release
and a vergil-claude-plugin release containing the work have been cut.

**Procedure:**

1. In a consumer repo's dev container, `vrg-container-run -- vrg-exceptions audit`
   prints `not adopted` (proves the command ships and is on `PATH`).
2. In a managed repo, an agent edit adding `# noqa: S603` triggers the
   `exception-request` reminder (proves the hook ships).
3. `/vergil:exception-request` is listed among available skills.

**Acceptance:** all three observed; results recorded with the SUCCESS template.
