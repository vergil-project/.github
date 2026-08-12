# Shared SARIF-gate exception allowlist Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Let the shared `evaluate_findings` SARIF gate honor a documented, version-controlled exception allowlist in `vergil.toml`, applied uniformly across the CodeQL / Semgrep / Trivy CLIs, with fail-on-stale hygiene.

**Architecture:** A small isolated loader parses `[[security.sarif-exception]]` from `vergil.toml`; `evaluate_findings` gains an `exceptions` argument, scopes exceptions to the current scanner by `tool.driver.name`, skips matched findings, and fails on stale (in-scope, zero-match) exceptions; the three scanner CLIs load the exceptions and pass them in. Spec: `epics/94-sarif-exceptions/spec.md` (vergil-project/.github#94).

**Tech Stack:** Python 3.12, `tomllib`, `fnmatch`; `vergil-tooling` (`src/vergil_tooling/`), pytest, `vrg-validate`.

**Where this is implemented:** `vergil-project/vergil-tooling` (this plan doc lives in the epic home; do the work in a vergil-tooling worktree). File paths below are relative to the vergil-tooling repo root.

## Global Constraints

- `tool`, `rule`, `reason`, `issue` are **required** on every exception; `path` is **optional**. A table missing a required field, or with `tool` not in `{codeql, semgrep, trivy}`, is a **config error** (fail loud).
- Exceptions are matched **only** against the scanner whose `tool.driver.name` maps to the exception's `tool` (tool-identity scoping) — never by `tool.driver.rules`.
- A finding is excepted iff `rule` == finding rule id **and** (`path` is absent **or** its glob matches the finding's file). Matching runs **after** the severity filter.
- **Fail on stale:** an in-scope exception that matches zero findings fails the gate.
- **Report, never silent:** applied exceptions (with match counts) and stale exceptions are emitted and land in the job summary.
- Existing callers of `evaluate_findings` must keep working unchanged until wired (the new `exceptions` arg defaults to none).
- Validation gate: `vrg-container-run -- vrg-validate`. TDD inner loop: `uv run pytest <path>::<test> -v`. Commit with `vrg-commit`.

---

### Task 1: Exception schema + loader

**Files:**

- Create: `src/vergil_tooling/lib/sarif_exceptions.py`
- Test: `tests/vergil_tooling/test_sarif_exceptions.py`

**Interfaces:**

- Produces: `SarifException(tool: str, rule: str, path: str | None, reason: str, issue: str)` (frozen dataclass); `load_sarif_exceptions(repo_root: Path) -> list[SarifException]`.

- [ ] **Step 1: Write the failing tests**

```python
# tests/vergil_tooling/test_sarif_exceptions.py
from __future__ import annotations

import pytest

from vergil_tooling.lib.sarif_exceptions import SarifException, load_sarif_exceptions

_VALID = """
[[security.sarif-exception]]
tool = "codeql"
rule = "py/overly-permissive-file"
path = "clients/app_requester.py"
reason = "non-sensitive metrics .prom must be group-readable"
issue = "logical-minds-foundry/mq-resiliency-lab-for-linux#491"

[[security.sarif-exception]]
tool = "trivy"
rule = "CVE-2025-0001"
reason = "no fix available; not reachable in our usage"
issue = "vergil-project/x#7"
"""


def _write(tmp_path, text):
    (tmp_path / "vergil.toml").write_text(text, encoding="utf-8")
    return tmp_path


def test_load_parses_required_and_optional_fields(tmp_path):
    excs = load_sarif_exceptions(_write(tmp_path, _VALID))
    assert excs == [
        SarifException("codeql", "py/overly-permissive-file", "clients/app_requester.py",
                       "non-sensitive metrics .prom must be group-readable",
                       "logical-minds-foundry/mq-resiliency-lab-for-linux#491"),
        SarifException("trivy", "CVE-2025-0001", None,
                       "no fix available; not reachable in our usage", "vergil-project/x#7"),
    ]


def test_missing_vergil_toml_yields_no_exceptions(tmp_path):
    assert load_sarif_exceptions(tmp_path) == []


def test_no_section_yields_no_exceptions(tmp_path):
    assert load_sarif_exceptions(_write(tmp_path, "[project]\n")) == []


def test_missing_required_field_is_config_error(tmp_path):
    bad = '[[security.sarif-exception]]\ntool="codeql"\nrule="r"\nreason="x"\n'  # no issue
    with pytest.raises(ValueError, match="issue"):
        load_sarif_exceptions(_write(tmp_path, bad))


def test_unknown_tool_is_config_error(tmp_path):
    bad = '[[security.sarif-exception]]\ntool="grype"\nrule="r"\nreason="x"\nissue="o/r#1"\n'
    with pytest.raises(ValueError, match="tool"):
        load_sarif_exceptions(_write(tmp_path, bad))
```

- [ ] **Step 2: Run to verify it fails**

Run: `uv run pytest tests/vergil_tooling/test_sarif_exceptions.py -v`
Expected: FAIL — `ModuleNotFoundError: vergil_tooling.lib.sarif_exceptions`.

- [ ] **Step 3: Implement the loader**

```python
# src/vergil_tooling/lib/sarif_exceptions.py
"""Documented exceptions for the SARIF gate, parsed from vergil.toml (#94)."""

from __future__ import annotations

import tomllib
from dataclasses import dataclass
from typing import TYPE_CHECKING

if TYPE_CHECKING:
    from pathlib import Path

TOOLS = frozenset({"codeql", "semgrep", "trivy"})
_REQUIRED = ("tool", "rule", "reason", "issue")


@dataclass(frozen=True)
class SarifException:
    tool: str
    rule: str
    path: str | None
    reason: str
    issue: str


def load_sarif_exceptions(repo_root: Path) -> list[SarifException]:
    """Parse [[security.sarif-exception]] tables from repo_root/vergil.toml.

    Missing file or section -> []. A table missing a required field, or with an
    unknown tool, is a config error (fail loud) -- an undocumented exception is
    not an exception.
    """
    toml_path = repo_root / "vergil.toml"
    if not toml_path.is_file():
        return []
    with toml_path.open("rb") as handle:
        raw = tomllib.load(handle)
    entries = raw.get("security", {}).get("sarif-exception", [])
    out: list[SarifException] = []
    for i, entry in enumerate(entries):
        missing = [k for k in _REQUIRED if not entry.get(k)]
        if missing:
            msg = f"security.sarif-exception[{i}] missing required field(s): {', '.join(missing)}"
            raise ValueError(msg)
        if entry["tool"] not in TOOLS:
            msg = f"security.sarif-exception[{i}] tool must be one of {sorted(TOOLS)}, got {entry['tool']!r}"
            raise ValueError(msg)
        out.append(
            SarifException(
                tool=entry["tool"],
                rule=entry["rule"],
                path=entry.get("path"),
                reason=entry["reason"],
                issue=entry["issue"],
            )
        )
    return out
```

- [ ] **Step 4: Run to verify it passes**

Run: `uv run pytest tests/vergil_tooling/test_sarif_exceptions.py -v`
Expected: PASS (5 passed).

- [ ] **Step 5: Commit**

```bash
vrg-git add src/vergil_tooling/lib/sarif_exceptions.py tests/vergil_tooling/test_sarif_exceptions.py
vrg-commit --type feat --scope sarif --message "add SarifException schema + vergil.toml loader (#94)"
```

---

### Task 2: Exception matching, tool-identity scoping, and fail-on-stale in `evaluate_findings`

**Files:**

- Modify: `src/vergil_tooling/lib/sarif.py`
- Test: `tests/vergil_tooling/test_sarif.py`

**Interfaces:**

- Consumes: `SarifException` (Task 1).
- Produces: `EvaluationResult` gains `applied: list[AppliedException]` and `stale: list[SarifException]`; `AppliedException(exception: SarifException, count: int)`; `evaluate_findings(sarif_data, severity_filter=None, exceptions=None)` — `passed` is now `(no findings) and (no stale)`.

- [ ] **Step 1: Write the failing tests**

```python
# add to tests/vergil_tooling/test_sarif.py
from vergil_tooling.lib.sarif import AppliedException, evaluate_findings
from vergil_tooling.lib.sarif_exceptions import SarifException


def _sarif(driver: str, rule: str, uri: str, level: str = "error") -> dict:
    return {
        "runs": [{
            "tool": {"driver": {"name": driver}},
            "results": [{
                "ruleId": rule, "level": level,
                "message": {"text": "m"},
                "locations": [{"physicalLocation": {
                    "artifactLocation": {"uri": uri}, "region": {"startLine": 1}}}],
            }],
        }],
    }


def test_exception_skips_matching_finding_and_records_applied():
    exc = SarifException("codeql", "py/x", "clients/app.py", "why", "o/r#1")
    res = evaluate_findings([_sarif("CodeQL", "py/x", "clients/app.py")], exceptions=[exc])
    assert res.findings == []
    assert res.passed is True
    assert res.applied == [AppliedException(exc, 1)]
    assert res.stale == []


def test_exception_glob_path_matches():
    exc = SarifException("codeql", "py/x", "clients/*.py", "why", "o/r#1")
    res = evaluate_findings([_sarif("CodeQL", "py/x", "clients/app.py")], exceptions=[exc])
    assert res.findings == [] and res.passed is True


def test_optional_path_matches_rule_wide():
    exc = SarifException("trivy", "CVE-1", None, "why", "o/r#1")
    res = evaluate_findings([_sarif("Trivy", "CVE-1", "go.mod")], exceptions=[exc])
    assert res.findings == [] and res.passed is True and res.applied[0].count == 1


def test_other_tool_exception_is_out_of_scope_not_applied_not_stale():
    # a semgrep exception must not be applied to, or judged stale by, the CodeQL run
    exc = SarifException("semgrep", "py/x", None, "why", "o/r#1")
    res = evaluate_findings([_sarif("CodeQL", "py/x", "clients/app.py")], exceptions=[exc])
    assert res.findings != []          # the CodeQL finding still gates
    assert res.passed is False
    assert res.applied == [] and res.stale == []


def test_in_scope_zero_match_is_stale_and_fails():
    exc = SarifException("codeql", "py/gone", "clients/app.py", "why", "o/r#1")
    res = evaluate_findings([_sarif("CodeQL", "py/other", "clients/app.py")], exceptions=[exc])
    assert res.stale == [exc]
    assert res.passed is False         # stale exception fails the gate


def test_exception_only_removes_severity_gated_findings():
    # a note-level result is already below the gate; an exception for it is stale
    exc = SarifException("codeql", "py/x", None, "why", "o/r#1")
    res = evaluate_findings([_sarif("CodeQL", "py/x", "a.py", level="note")], exceptions=[exc])
    assert res.findings == []           # note is not gated
    assert res.stale == [exc]           # so the exception matched nothing -> stale
```

- [ ] **Step 2: Run to verify it fails**

Run: `uv run pytest tests/vergil_tooling/test_sarif.py -k "exception or stale or scope or glob or optional" -v`
Expected: FAIL — `cannot import name 'AppliedException'` / `evaluate_findings() got an unexpected keyword argument 'exceptions'`.

- [ ] **Step 3: Implement matching + scoping + stale in `sarif.py`**

Add near the top of `src/vergil_tooling/lib/sarif.py`:

```python
import fnmatch

from vergil_tooling.lib.sarif_exceptions import SarifException

# Map a SARIF tool.driver.name (lowercased) to our exception `tool` id. Verify the
# real driver.name strings against a live scan of each tool before relying on them.
_DRIVER_NAME_TO_TOOL = {"codeql": "codeql", "semgrep": "semgrep", "trivy": "trivy"}


@dataclass(frozen=True)
class AppliedException:
    exception: SarifException
    count: int


def _scanner_tool(sarif_data: list[dict[str, Any]]) -> str | None:
    """The exception `tool` id for this SARIF, from the first recognized driver name."""
    for data in sarif_data:
        for run in data.get("runs", []):
            name = run.get("tool", {}).get("driver", {}).get("name", "")
            tool = _DRIVER_NAME_TO_TOOL.get(name.lower())
            if tool:
                return tool
    return None


def _excepts(exc: SarifException, finding: SarifFinding) -> bool:
    if exc.rule != finding.rule_id:
        return False
    return exc.path is None or fnmatch.fnmatch(finding.file, exc.path)
```

Extend `EvaluationResult`:

```python
@dataclass(frozen=True)
class EvaluationResult:
    findings: list[SarifFinding] = field(default_factory=list)
    passed: bool = True
    applied: list[AppliedException] = field(default_factory=list)
    stale: list[SarifException] = field(default_factory=list)
```

Rewrite `evaluate_findings` (keep the existing finding-extraction; add scoping + exception handling):

```python
def evaluate_findings(
    sarif_data: list[dict[str, Any]],
    severity_filter: set[str] | None = None,
    exceptions: list[SarifException] | None = None,
) -> EvaluationResult:
    """Filter findings by severity, applying documented exceptions scoped to this
    scanner (by tool.driver.name). An in-scope exception that matches no finding is
    stale and fails the gate (#94)."""
    if severity_filter is None:
        severity_filter = {"warning", "error"}
    tool = _scanner_tool(sarif_data)
    in_scope = [e for e in (exceptions or []) if e.tool == tool]
    counts = [0] * len(in_scope)

    findings: list[SarifFinding] = []
    for data in sarif_data:
        for run in data.get("runs", []):
            for result in run.get("results", []):
                level = result.get("level", "warning")
                if level not in severity_filter:
                    continue
                rule_id = result.get("ruleId", "unknown")
                message = result.get("message", {}).get("text", "")
                file_path = ""
                line_num = 0
                locations = result.get("locations", [])
                if locations:
                    phys = locations[0].get("physicalLocation", {})
                    file_path = phys.get("artifactLocation", {}).get("uri", "")
                    line_num = phys.get("region", {}).get("startLine", 0)
                finding = SarifFinding(
                    rule_id=rule_id, message=message, level=level,
                    file=file_path, line=line_num,
                )
                idx = next((i for i, e in enumerate(in_scope) if _excepts(e, finding)), None)
                if idx is not None:
                    counts[idx] += 1
                    continue
                findings.append(finding)

    applied = [AppliedException(in_scope[i], counts[i]) for i in range(len(in_scope)) if counts[i]]
    stale = [in_scope[i] for i in range(len(in_scope)) if counts[i] == 0]
    passed = not findings and not stale
    return EvaluationResult(findings=findings, passed=passed, applied=applied, stale=stale)
```

- [ ] **Step 4: Run to verify it passes**

Run: `uv run pytest tests/vergil_tooling/test_sarif.py -v`
Expected: PASS (existing + new tests). If a pre-existing test constructs `EvaluationResult(findings=..., passed=...)` positionally, it still works — the new fields default empty.

- [ ] **Step 5: Report applied/stale in `format_summary`**

Append to `format_summary` before the final `return`, and cover it with a test:

```python
# in format_summary(result), after the findings table is built (and in the passed branch too):
    if result.applied:
        lines += ["", "### Exceptions applied", "",
                  "| Tool | Rule | Path | Count | Issue |",
                  "|------|------|------|-------|-------|"]
        for a in result.applied:
            e = a.exception
            lines.append(f"| {e.tool} | {e.rule} | {e.path or '*'} | {a.count} | {e.issue} |")
    if result.stale:
        lines += ["", "### Stale exceptions (remove these — they match nothing)", ""]
        lines += [f"- `{e.tool}` `{e.rule}` ({e.path or '*'}) — {e.issue}" for e in result.stale]
```

Note: `format_summary`'s early `if result.passed:` return must be updated so an
all-excepted run still shows its applied table. Restructure to build `lines`
first, then return. Add:

```python
def test_format_summary_lists_applied_and_stale():
    exc = SarifException("codeql", "py/x", None, "why", "o/r#1")
    res = evaluate_findings([_sarif("CodeQL", "py/x", "a.py")], exceptions=[exc])
    text = format_summary(res)
    assert "Exceptions applied" in text and "py/x" in text
```

Run: `uv run pytest tests/vergil_tooling/test_sarif.py -v` → PASS.

- [ ] **Step 6: Commit**

```bash
vrg-git add src/vergil_tooling/lib/sarif.py tests/vergil_tooling/test_sarif.py
vrg-commit --type feat --scope sarif --message "evaluate_findings: apply exceptions, tool-identity scope, fail on stale (#94)"
```

---

### Task 3: Wire the three scanner CLIs

**Files:**

- Modify: `src/vergil_tooling/bin/vrg_sarif_evaluate.py`, `src/vergil_tooling/bin/vrg_semgrep_scan.py`, `src/vergil_tooling/bin/vrg_trivy_scan.py`
- Test: `tests/vergil_tooling/test_vrg_sarif_evaluate.py` (+ mirror for the other two if they have suites)

**Interfaces:**

- Consumes: `load_sarif_exceptions` (Task 1), `evaluate_findings(..., exceptions=...)` and `EvaluationResult.stale` (Task 2).

- [ ] **Step 1: Write the failing test (CodeQL CLI)**

```python
# tests/vergil_tooling/test_vrg_sarif_evaluate.py — add
def test_exception_from_vergil_toml_clears_the_gate(tmp_path, monkeypatch, capsys):
    (tmp_path / "vergil.toml").write_text(
        '[[security.sarif-exception]]\ntool="codeql"\nrule="VULN-1"\n'
        'reason="reviewed, accepted"\nissue="o/r#1"\n', encoding="utf-8")
    sarif = tmp_path / "out.sarif"
    sarif.write_text(json.dumps({"runs": [{
        "tool": {"driver": {"name": "CodeQL"}},
        "results": [{"ruleId": "VULN-1", "level": "error", "message": {"text": "m"},
                     "locations": [{"physicalLocation": {"artifactLocation": {"uri": "a.py"}}}]}],
    }]}), encoding="utf-8")
    monkeypatch.chdir(tmp_path)
    from vergil_tooling.bin.vrg_sarif_evaluate import main
    assert main([str(sarif)]) == 0            # excepted -> gate passes
```

Run: `uv run pytest tests/vergil_tooling/test_vrg_sarif_evaluate.py::test_exception_from_vergil_toml_clears_the_gate -v`
Expected: FAIL — exit 1 (exception not loaded yet).

- [ ] **Step 2: Wire each CLI**

In each of the three `bin/*.py`, load exceptions from the repo root (the CI cwd) and pass them, then emit stale as errors. The edit is the same shape in all three; apply it to each. For `vrg_sarif_evaluate.py`:

```python
# add imports
from pathlib import Path
from vergil_tooling.lib.sarif_exceptions import load_sarif_exceptions

# replace the evaluate call:
exceptions = load_sarif_exceptions(Path.cwd())
result = evaluate_findings(sarif_data, severity_filter, exceptions=exceptions)

# after the existing `for finding in result.findings: emit_error(...)` loop, add:
for applied in result.applied:
    e = applied.exception
    emit_warning(f"exception applied: [{e.tool}/{e.rule}] {applied.count} finding(s) — {e.issue}")
for e in result.stale:
    emit_error(f"stale exception (matches nothing, remove it): [{e.tool}/{e.rule}] {e.issue}")
```

For `vrg_semgrep_scan.py` and `vrg_trivy_scan.py`, the call is `evaluate_findings(sarif_data, exceptions=exceptions)` (they pass no `severity_filter`); add the same `load_sarif_exceptions(Path.cwd())` line and the same applied/stale emit loops after their finding loop. `result.passed` already drives their `return 0 if result.passed else 1`, so stale failures propagate with no further change.

- [ ] **Step 3: Run to verify it passes**

Run: `uv run pytest tests/vergil_tooling/test_vrg_sarif_evaluate.py -v`
Expected: PASS. Existing CLI tests still pass (no `vergil.toml` in their tmp dir → `load_sarif_exceptions` returns `[]` → unchanged behavior).

- [ ] **Step 4: Validate the whole tree**

Run: `vrg-container-run -- vrg-validate`
Expected: PASS (lint, typecheck, tests, coverage).

- [ ] **Step 5: Commit**

```bash
vrg-git add src/vergil_tooling/bin/vrg_sarif_evaluate.py src/vergil_tooling/bin/vrg_semgrep_scan.py src/vergil_tooling/bin/vrg_trivy_scan.py tests/vergil_tooling/test_vrg_sarif_evaluate.py
vrg-commit --type feat --scope sarif --message "wire codeql/semgrep/trivy CLIs to load + report SARIF exceptions (#94)"
```

---

### Task 4 (verification): confirm real `tool.driver.name` strings

**Files:** none (verification + a possible one-line map fix in `lib/sarif.py`).

- [ ] **Step 1:** From a real CI run (or a sample SARIF of each scanner), read `runs[].tool.driver.name` for CodeQL, Semgrep, and Trivy.
- [ ] **Step 2:** Confirm each lowercased name is a key in `_DRIVER_NAME_TO_TOOL` (`codeql` / `semgrep` / `trivy`). If a scanner reports a different string (e.g. `"Trivy Vulnerability Scanner"`), add/adjust the mapping key and a unit test asserting that name maps.
- [ ] **Step 3:** `vrg-container-run -- vrg-validate`, then `vrg-commit --type fix --scope sarif --message "map real scanner driver names (#94)"` if a change was needed.

---

## Self-Review

**Spec coverage** (spec §→task):

- §3.1 config surface (required `tool/rule/reason/issue`, optional `path`, enum) → Task 1.
- §3.2 matching + tool-identity scoping (`tool.driver.name`, not `driver.rules`) → Task 2 (`_scanner_tool`, `_excepts`) + Task 4 (real names).
- §3.3 skip / report / fail-on-stale, applied after severity → Task 2 (evaluate + `passed`) + Task 2 Step 5 (report) + Task 3 (CLI emit).
- §3.4 boundaries: `lib/sarif.py`, dedicated loader, all three CLIs → Tasks 1–3. (Loader is a dedicated `sarif_exceptions.py` reading the same `vergil.toml` — honors "reuse the config file" without bloating the typed `VergilConfig`.)
- §4 testing (glob, rule-wide, scoping, validation, stale, severity ordering, report, per-CLI) → Task 1 + Task 2 + Task 3 tests.
- §5 non-goals — nothing to implement; the plan adds no governance/expiry.
- §6 tradeoffs — informational.
- §7 first consumer — separate cross-org task, not in this plan.

**Placeholder scan:** none — every code step is complete; Task 4 is explicit verification with a concrete map to fix, not deferred work.

**Type consistency:** `SarifException(tool, rule, path|None, reason, issue)`, `AppliedException(exception, count)`, `evaluate_findings(sarif_data, severity_filter=None, exceptions=None)`, and `EvaluationResult(findings, passed, applied, stale)` are used identically across Tasks 1–3. `load_sarif_exceptions(repo_root)` is defined in Task 1 and consumed unchanged in Task 3.
