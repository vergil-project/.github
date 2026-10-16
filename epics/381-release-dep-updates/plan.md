# Release-integrated dependency updates: implementation plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Every `vrg-release` ends by upgrading the repo's dependencies to the
newest versions their declared bounds allow. A failed update leaves an open,
labelled PR and a fix task, and the next release refuses to start until that PR
is dealt with. Repos can add bespoke update logic through a hook. Every run
reports what the bounds are holding back.

**Architecture:** `vrg-update-deps` keeps its stage pipeline and Updater
protocol (`lib/update_deps/`). Updaters gain held-back report entries. A new
hook-runner updater runs a repo-owned command, and a new `images` updater bumps
exact image tags. The driver no longer raises on validation, merge or updater
failures. It records the failure on its context, keeps the labelled PR open,
and files a fix task. A plain-function entry point, `run_update()`, lets
`vrg-release` run the same stages inside its own pipeline without nesting
`progress.run_pipeline`, which holds a single module-global session. The release
gains a `fail_defer` `update-deps` stage and a preflight gate on open
`dependency-update` PRs. vergil-vm and vergil-python each ship a hook.

**Tech Stack:** Python 3.12+ (`vergil-tooling`) with `pytest` and 100% branch
coverage; `packaging` (new runtime dependency) for PEP 440 specifiers; bash and
stdlib Python for the repo hooks; the Docker Registry v2 HTTP API; the PyPI JSON
API; the vergil apt index.

**Spec:** `epics/381-release-dep-updates/spec.md` (same directory). Read it
before starting any task. Section references (§N) below point into it.

## Global Constraints

- Validation is exactly `vrg-container-run -- vrg-validate`. In vergil-tooling
  it expands to `uv run vrg-validate`. Never run linters or formatters
  standalone. For the TDD red/green loop, run focused tests with
  `vrg-container-run -- uv run pytest <path> -v`.
- **100% branch coverage is enforced** in vergil-tooling, on every
  `[ci].versions` interpreter (3.12, 3.13, 3.14) in PR CI. Every new line and
  branch needs a test, so avoid unreachable branches.
- Git and GitHub only through `vrg-git` and `vrg-gh`. Commit with
  `vrg-git add <paths>` then
  `vrg-commit --type <t> --scope <s> --message <m> [--body <b>]`.
- One PR per task. Each task is filed and lands in the repo named in its
  heading (placement law). No PR `Closes` an issue across repos.
- The label name is exactly `dependency-update` (§5.2).
- The deps branch is exactly `deps/update-<new-develop-version>`, for example
  `deps/update-2.1.243` (§5.1). The fix branch is
  `chore/<N>-dep-update-<version>` (§5.2).
- The fix-task title is exactly `Fix failed dependency update for <version>`
  (§5.2).
- Failure output in a PR body or issue is at most the last 200 lines, and at
  most 20,000 characters, of the failing output, plus the reproduce command
  (§5.2). A rendered PR body must stay under GitHub's 65,536-character limit.
- The `[dependency-update]` table has exactly one key, `hook`, a non-empty
  string (§6.1).
- A held-back entry is one JSON object per line: `source`, `name`, `current`
  and `latest` are required strings, and `note` is an optional string (§7.2).
- Fail loud. No fallback may turn a failed lookup, hook or parse into "no
  change" (§3.7).
- Tooling carries no repo-specific logic (§3.5). vergil-vm and vergil-python
  logic lives only in their own hooks.
- Site docs (`docs/site/…`) are **not** edited by these tasks. The epic's
  documentation-review bookend sweeps them, and marks the 2026-06-10 spec as
  partly superseded (§10).

## Review Focus

These are inputs or conditions the spec implies but no task's ordinary tests
would exercise, most likely to bite first. Each one has a pinned test in the
task named.

1. **An enormous PR body.** With 10,000 lines of validation output and 2,000
   held-back entries, the PR body must still render under 65,536 characters
   (the failure tail is capped, and the held-back table is capped at 200 rows
   with an "…and N more" line). Otherwise `gh pr create` is rejected and no
   PR exists for the gate to see. Tests: Task 1
   (`test_render_markdown_caps_rows`) and Task 2
   (`test_pr_body_stays_under_github_limit`).
2. **A second off-cycle `vrg-update-deps` on the same develop version.** The
   branch `deps/update-<v>` already exists. The run must fail loud, naming the
   branch, rather than reusing or clobbering it. Test: Task 2
   (`test_preflight_existing_deps_branch_fails_loud`).
3. **`vrg-release --resume` with a pending deps PR.** A resumed release is
   mid-flight, and the gate must not strand it. The gate runs only on a fresh
   start. Test: Task 4 (`test_preflight_resume_skips_pending_deps_gate`).
4. **Hook report lines that are almost right.** These include a trailing blank
   line (accepted), an unknown key, a numeric `current`, a missing `latest`,
   and invalid JSON (each rejected loud, naming the line number). Test: Task 1
   (`test_parse_report_lines_*`).
5. **Real-world `pyproject.toml` shapes.** These are `{include-group = …}`
   entries in dependency groups, URL requirements, and extras with markers
   (`foo[bar]>=1; python_version < "3.13"`). The first two are skipped and the
   third is parsed by name. Test: Task 5
   (`test_direct_requirements_shapes`).

---

### Task 1: Report plumbing, `[dependency-update]` config, and the hook runner [`vergil-tooling`]

Covers spec §6.1, §6.2, §7.1, §7.2 and the hook-runner row of §9.

**Files:**

- Create: `src/vergil_tooling/lib/update_deps/report.py`
- Create: `src/vergil_tooling/lib/update_deps/updaters/repo_hook.py`
- Modify: `src/vergil_tooling/lib/update_deps/updater.py` (add `held_back` to `UpdateResult`)
- Modify: `src/vergil_tooling/lib/update_deps/orchestrator.py` (register `RepoHookUpdater` last)
- Modify: `src/vergil_tooling/lib/config.py` (`[dependency-update]` table)
- Test: `tests/vergil_tooling/test_update_deps_report.py` (new)
- Test: `tests/vergil_tooling/test_update_deps_repo_hook.py` (new)
- Test: `tests/vergil_tooling/test_config.py`
- Test: `tests/vergil_tooling/test_update_deps_orchestrator.py` (registry names)

**Interfaces:**

- Consumes: `progress.run(cmd)` (raises `subprocess.CalledProcessError`
  carrying output), `git.read_output(*args)`, `config.read_config(root)`.
- Produces:
  - `report.HeldBack(source: str, name: str, current: str, latest: str, note: str | None = None)`
    (frozen dataclass)
  - `report.to_line(entry: HeldBack) -> str`
  - `report.parse_report_lines(text: str, *, origin: str) -> list[HeldBack]`
    (raises `UpdateDepsError`)
  - `report.collect(results: Sequence[UpdateResult]) -> list[HeldBack]`
  - `report.render_markdown(entries: Sequence[HeldBack], *, max_rows: int = 200) -> str`
  - `report.EMPTY_REPORT: str`
  - `UpdateResult.held_back: list[HeldBack]` (default empty)
  - `config.DependencyUpdateConfig(hook: str | None = None)`;
    `VergilConfig.dependency_update`; `config.dependency_update_hook(repo_root: Path) -> str | None`
  - `updaters.repo_hook.RepoHookUpdater` with `name = "hook"`
  - `DEFAULT_REGISTRY` names: `["python", "vergil", "hook"]`

- [ ] **Step 1: Failing tests for the report module**

```python
# tests/vergil_tooling/test_update_deps_report.py
from __future__ import annotations

import json

import pytest

from vergil_tooling.lib.update_deps import report
from vergil_tooling.lib.update_deps.context import UpdateDepsError
from vergil_tooling.lib.update_deps.report import HeldBack
from vergil_tooling.lib.update_deps.updater import UpdateResult

_E = HeldBack(source="python", name="example-pkg", current="1.9.4", latest="2.1.0",
              note="pyproject: example-pkg>=1.0,<2")


def test_to_line_round_trips() -> None:
    line = report.to_line(_E)
    assert json.loads(line) == {
        "source": "python", "name": "example-pkg", "current": "1.9.4",
        "latest": "2.1.0", "note": "pyproject: example-pkg>=1.0,<2",
    }
    assert report.parse_report_lines(line + "\n", origin="t") == [_E]


def test_to_line_omits_absent_note() -> None:
    entry = HeldBack(source="s", name="n", current="1", latest="2")
    assert "note" not in json.loads(report.to_line(entry))


def test_parse_report_lines_accepts_blank_lines() -> None:
    text = "\n" + report.to_line(_E) + "\n\n"
    assert report.parse_report_lines(text, origin="t") == [_E]


@pytest.mark.parametrize(
    ("line", "why"),
    [
        ("{not json", "not valid JSON"),
        ("[1, 2]", "not a JSON object"),
        ('{"source": "s", "name": "n", "current": "1"}', "missing required key 'latest'"),
        ('{"source": "s", "name": "n", "current": 1, "latest": "2"}', "'current' must be a string"),
        ('{"source": "s", "name": "n", "current": "1", "latest": "2", "note": 3}',
         "'note' must be a string"),
        ('{"source": "s", "name": "n", "current": "1", "latest": "2", "extra": "x"}',
         "unknown key 'extra'"),
    ],
)
def test_parse_report_lines_rejects_malformed(line: str, why: str) -> None:
    text = report.to_line(_E) + "\n" + line + "\n"
    with pytest.raises(UpdateDepsError, match=why) as info:
        report.parse_report_lines(text, origin="scripts/dev/update-deps.sh")
    assert "line 2" in str(info.value)
    assert "scripts/dev/update-deps.sh" in str(info.value)


def test_collect_flattens_in_result_order() -> None:
    a = HeldBack(source="a", name="a", current="1", latest="2")
    results = [
        UpdateResult(updater="x", changed=False, summary="", commit_message="", held_back=[a]),
        UpdateResult(updater="y", changed=False, summary="", commit_message=""),
        UpdateResult(updater="z", changed=True, summary="", commit_message="", held_back=[_E]),
    ]
    assert report.collect(results) == [a, _E]


def test_render_markdown_empty_says_so() -> None:
    assert report.render_markdown([]) == report.EMPTY_REPORT
    assert "No dependencies are held back" in report.EMPTY_REPORT


def test_render_markdown_table_escapes_pipes() -> None:
    entry = HeldBack(source="python", name="p", current="1", latest="2", note="a|b")
    text = report.render_markdown([entry])
    assert "| python | p | 1 | 2 | a\\|b |" in text


def test_render_markdown_caps_rows() -> None:
    entries = [HeldBack(source="s", name=f"n{i}", current="1", latest="2") for i in range(2000)]
    text = report.render_markdown(entries)
    assert text.count("\n| s | n") == 200
    assert "…and 1800 more" in text
    assert len(text) < 20_000
```

- [ ] **Step 2: Run them. Expect FAIL (no `report` module)**

Run: `vrg-container-run -- uv run pytest tests/vergil_tooling/test_update_deps_report.py -v`
Expected: FAIL with `ModuleNotFoundError: ... update_deps.report`.

- [ ] **Step 3: Add `held_back` to `UpdateResult`**

In `src/vergil_tooling/lib/update_deps/updater.py`, extend the dataclass. The
import goes under `TYPE_CHECKING` to avoid a cycle, because `report` imports
`UpdateResult` for typing only:

```python
if TYPE_CHECKING:
    from collections.abc import Sequence

    from vergil_tooling.lib.update_deps.context import UpdateDepsContext
    from vergil_tooling.lib.update_deps.report import HeldBack


@dataclass
class UpdateResult:
    """What an updater changed (or didn't) in one run."""

    updater: str
    changed: bool
    summary: str
    commit_message: str
    warnings: list[str] = field(default_factory=list)
    # Dependencies newer than the declared bounds allow (spec §7). Reported,
    # never bumped.
    held_back: list[HeldBack] = field(default_factory=list)
```

- [ ] **Step 4: Write `report.py`**

```python
# src/vergil_tooling/lib/update_deps/report.py
"""The held-back report: dependencies newer than their declared bounds allow.

Spec §7 (epic vergil-project/.github#381). Built-in updaters produce entries
directly; a repo hook appends JSON lines to ``$VRG_DEPS_REPORT``. A malformed
line fails the run loud rather than being dropped.
"""

from __future__ import annotations

import json
from dataclasses import dataclass
from typing import TYPE_CHECKING

from vergil_tooling.lib.update_deps.context import UpdateDepsError

if TYPE_CHECKING:
    from collections.abc import Sequence

    from vergil_tooling.lib.update_deps.updater import UpdateResult

EMPTY_REPORT = "No dependencies are held back by their declared bounds."

_REQUIRED = ("source", "name", "current", "latest")
_OPTIONAL = ("note",)


@dataclass(frozen=True)
class HeldBack:
    """One dependency whose newest release is outside its declared bounds."""

    source: str
    name: str
    current: str
    latest: str
    note: str | None = None


def to_line(entry: HeldBack) -> str:
    """Serialize *entry* as one JSON line (no trailing newline)."""
    data = {"source": entry.source, "name": entry.name, "current": entry.current,
            "latest": entry.latest}
    if entry.note is not None:
        data["note"] = entry.note
    return json.dumps(data, sort_keys=False)


def _reject(origin: str, lineno: int, why: str) -> UpdateDepsError:
    return UpdateDepsError(
        phase="update",
        command=origin,
        message=f"Malformed held-back report from {origin}, line {lineno}: {why}.",
    )


def parse_report_lines(text: str, *, origin: str) -> list[HeldBack]:
    """Parse JSON-lines report *text*; blank lines are ignored, anything else must be valid."""
    entries: list[HeldBack] = []
    for lineno, raw in enumerate(text.splitlines(), start=1):
        if not raw.strip():
            continue
        try:
            data = json.loads(raw)
        except json.JSONDecodeError as exc:
            raise _reject(origin, lineno, f"not valid JSON ({exc.msg})") from exc
        if not isinstance(data, dict):
            raise _reject(origin, lineno, "not a JSON object")
        for key in data:
            if key not in _REQUIRED and key not in _OPTIONAL:
                raise _reject(origin, lineno, f"unknown key '{key}'")
        for key in _REQUIRED:
            if key not in data:
                raise _reject(origin, lineno, f"missing required key '{key}'")
        for key in (*_REQUIRED, *_OPTIONAL):
            if key in data and not isinstance(data[key], str):
                raise _reject(origin, lineno, f"'{key}' must be a string")
        entries.append(HeldBack(**data))
    return entries


def collect(results: Sequence[UpdateResult]) -> list[HeldBack]:
    """Every result's held-back entries, in registry order."""
    return [entry for result in results for entry in result.held_back]


def _cell(value: str | None) -> str:
    return (value or "").replace("|", "\\|")


def render_markdown(entries: Sequence[HeldBack], *, max_rows: int = 200) -> str:
    """Render *entries* as a Markdown table, capped at *max_rows* rows."""
    if not entries:
        return EMPTY_REPORT
    lines = [
        "| Source | Dependency | Current | Latest | Note |",
        "|---|---|---|---|---|",
    ]
    lines.extend(
        f"| {_cell(e.source)} | {_cell(e.name)} | {_cell(e.current)} | "
        f"{_cell(e.latest)} | {_cell(e.note)} |"
        for e in entries[:max_rows]
    )
    if len(entries) > max_rows:
        lines.append("")
        lines.append(f"…and {len(entries) - max_rows} more.")
    return "\n".join(lines)
```

- [ ] **Step 5: Run the report tests. Expect PASS**

Run: `vrg-container-run -- uv run pytest tests/vergil_tooling/test_update_deps_report.py -v`
Expected: PASS.

- [ ] **Step 6: Failing config tests**

Append to `tests/vergil_tooling/test_config.py`:

```python
_DEPS_BASE = """\
[project]
repository-type = "infrastructure"
versioning-scheme = "semver"
branching-model = "library-release"
release-model = "tagged-release"

[ci]
versions = ["latest"]
integration-tests = false

[publish]
release = true
docs = false

[dependencies]
vergil = "v2.1"
"""


def _write_cfg(tmp_path, extra: str):
    (tmp_path / "vergil.toml").write_text(_DEPS_BASE + extra)
    return tmp_path


def test_dependency_update_absent_defaults_to_no_hook(tmp_path) -> None:
    cfg = config.read_config(_write_cfg(tmp_path, ""))
    assert cfg.dependency_update.hook is None
    assert config.dependency_update_hook(tmp_path) is None


def test_dependency_update_hook_parsed(tmp_path) -> None:
    root = _write_cfg(tmp_path, '\n[dependency-update]\nhook = "scripts/dev/update-deps.sh"\n')
    assert config.dependency_update_hook(root) == "scripts/dev/update-deps.sh"


@pytest.mark.parametrize(
    ("table", "match"),
    [
        ('\n[dependency-update]\nhook = ""\n', "must be a non-empty string"),
        ('\n[dependency-update]\nhook = "   "\n', "must be a non-empty string"),
        ("\n[dependency-update]\nhook = 3\n", "must be a non-empty string"),
        ("\n[dependency-update]\n", "missing required field 'hook'"),
        ('dependency-update = "x"\n', r"\[dependency-update\] must be a table"),
    ],
)
def test_dependency_update_invalid(tmp_path, table: str, match: str) -> None:
    # A bare top-level key must precede every table header, so prepend it.
    text = table + _DEPS_BASE if not table.startswith("\n") else _DEPS_BASE + table
    (tmp_path / "vergil.toml").write_text(text)
    with pytest.raises(config.ConfigError, match=match):
        config.read_config(tmp_path)


def test_dependency_update_unknown_key_warns(tmp_path, capsys) -> None:
    root = _write_cfg(tmp_path, '\n[dependency-update]\nhook = "h"\nschedule = "x"\n')
    config.read_config(root)
    assert "unrecognized key 'schedule' in [dependency-update]" in capsys.readouterr().err
```

(If `test_config.py` lacks the `config` and `pytest` imports, add
`import pytest` and `from vergil_tooling.lib import config`.)

- [ ] **Step 7: Run. Expect FAIL (no `dependency_update` attribute)**

Run: `vrg-container-run -- uv run pytest tests/vergil_tooling/test_config.py -k dependency_update -v`
Expected: FAIL.

- [ ] **Step 8: Implement the config table**

In `src/vergil_tooling/lib/config.py`:

1. Add `"dependency-update"` to `_KNOWN_SECTIONS`, and
   `"dependency-update": frozenset({"hook"}),` to `_KNOWN_KEYS`.
2. Add the dataclass after `ValidationConfig`:

```python
@dataclass
class DependencyUpdateConfig:
    # [dependency-update] (epic vergil-project/.github#381, spec §6.1): a
    # repo-owned command vrg-update-deps runs in the dev container after the
    # built-in updaters. None when the table is absent. Distinct from
    # [dependencies], which declares *what* the repo depends on.
    hook: str | None = None
```

**3.** Add the field to `VergilConfig`, after `package`:

```python
    dependency_update: DependencyUpdateConfig = field(default_factory=DependencyUpdateConfig)
```

**4.** Add the parser next to `_parse_test_config`:

```python
def _parse_dependency_update(
    raw: dict[str, Any], source: str = CONFIG_FILE
) -> DependencyUpdateConfig:
    table = raw.get("dependency-update")
    if table is None:
        return DependencyUpdateConfig()
    if not isinstance(table, dict):
        msg = f"{source}: [dependency-update] must be a table"
        raise ConfigError(msg)
    if "hook" not in table:
        msg = f"{source}: [dependency-update] missing required field 'hook'"
        raise ConfigError(msg)
    hook = table["hook"]
    if not isinstance(hook, str) or not hook.strip():
        msg = f"{source}: [dependency-update].hook must be a non-empty string"
        raise ConfigError(msg)
    return DependencyUpdateConfig(hook=hook)
```

**5.** In `_parse_raw_config`'s `return VergilConfig(...)`, add
   `dependency_update=_parse_dependency_update(raw, source),`.
**6.** Add the accessor next to `validation_container_command`:

```python
def dependency_update_hook(repo_root: Path) -> str | None:
    """Return ``[dependency-update].hook`` from vergil.toml, or None when unset.

    ``FileNotFoundError`` and ``ConfigError`` propagate: callers check for
    vergil.toml first, and a malformed table must fail loud.
    """
    return read_config(repo_root).dependency_update.hook
```

- [ ] **Step 9: Run the config tests. Expect PASS**

Run: `vrg-container-run -- uv run pytest tests/vergil_tooling/test_config.py -v`
Expected: PASS (new tests and existing ones).

- [ ] **Step 10: Failing hook-runner tests**

```python
# tests/vergil_tooling/test_update_deps_repo_hook.py
from __future__ import annotations

import subprocess
from pathlib import Path

import pytest

from vergil_tooling.lib.update_deps import report
from vergil_tooling.lib.update_deps.context import UpdateDepsContext, UpdateDepsError
from vergil_tooling.lib.update_deps.report import HeldBack
from vergil_tooling.lib.update_deps.updaters.repo_hook import RepoHookUpdater

_MOD = "vergil_tooling.lib.update_deps.updaters.repo_hook"


def _ctx(root: Path) -> UpdateDepsContext:
    return UpdateDepsContext(repo="o/r", repo_root=root, worktree_path=root)


def _setup(monkeypatch, tmp_path, *, hook="scripts/dev/update-deps.sh", heads=("a", "a"),
           status="", run=None):
    monkeypatch.setattr(_MOD + ".config.dependency_update_hook", lambda root: hook)
    head_iter = iter(heads)

    def fake_read(*args):
        if args == ("rev-parse", "HEAD"):
            return next(head_iter)
        if args == ("status", "--porcelain"):
            return status
        raise AssertionError(args)

    monkeypatch.setattr(_MOD + ".git.read_output", fake_read)
    calls: list[list[str]] = []

    def fake_run(cmd):
        calls.append(cmd)
        if run is not None:
            run(tmp_path)
        return 0

    monkeypatch.setattr(_MOD + ".progress.run", fake_run)
    return calls


def test_applies_only_with_hook_configured(monkeypatch, tmp_path) -> None:
    (tmp_path / "vergil.toml").write_text("")
    monkeypatch.setattr(_MOD + ".config.dependency_update_hook", lambda root: None)
    assert RepoHookUpdater().applies(_ctx(tmp_path)) is False
    monkeypatch.setattr(_MOD + ".config.dependency_update_hook", lambda root: "h.sh")
    assert RepoHookUpdater().applies(_ctx(tmp_path)) is True


def test_applies_false_without_vergil_toml(tmp_path) -> None:
    assert RepoHookUpdater().applies(_ctx(tmp_path)) is False


def test_runs_hook_in_container_with_report_path(monkeypatch, tmp_path) -> None:
    calls = _setup(monkeypatch, tmp_path, hook="scripts/dev/update-deps.sh --fast")
    RepoHookUpdater().apply(_ctx(tmp_path))
    assert calls == [[
        "vrg-container-run", "--", "env",
        "VRG_DEPS_REPORT=/workspace/.vergil/deps-report.jsonl",
        "scripts/dev/update-deps.sh", "--fast",
    ]]


def test_hook_that_edits_reports_change(monkeypatch, tmp_path) -> None:
    _setup(monkeypatch, tmp_path, status=" M opentofu/x.hcl")
    result = RepoHookUpdater().apply(_ctx(tmp_path))
    assert result.changed is True
    assert result.commit_message == "chore(deps): repo hook"
    assert result.updater == "hook"


def test_hook_that_does_nothing_reports_no_change(monkeypatch, tmp_path) -> None:
    _setup(monkeypatch, tmp_path, status="")
    result = RepoHookUpdater().apply(_ctx(tmp_path))
    assert result.changed is False


def test_hook_report_entries_collected_and_file_removed(monkeypatch, tmp_path) -> None:
    entry = HeldBack(source="vergil-vm", name="lima", current="2.0.0", latest="3.0.1")

    def write_report(root: Path) -> None:
        (root / ".vergil" / "deps-report.jsonl").write_text(report.to_line(entry) + "\n")

    _setup(monkeypatch, tmp_path, run=write_report)
    result = RepoHookUpdater().apply(_ctx(tmp_path))
    assert result.held_back == [entry]
    assert not (tmp_path / ".vergil" / "deps-report.jsonl").exists()


def test_hook_nonzero_exit_fails_loud_with_output(monkeypatch, tmp_path) -> None:
    def boom(root: Path) -> None:
        raise subprocess.CalledProcessError(2, ["h"], output="tofu: registry unreachable")

    _setup(monkeypatch, tmp_path, run=boom)
    with pytest.raises(UpdateDepsError, match="hook failed") as info:
        RepoHookUpdater().apply(_ctx(tmp_path))
    assert "registry unreachable" in (info.value.detail or "")


def test_hook_that_commits_fails_head_check(monkeypatch, tmp_path) -> None:
    _setup(monkeypatch, tmp_path, heads=("aaaa1111", "bbbb2222"))
    with pytest.raises(UpdateDepsError, match="moved HEAD"):
        RepoHookUpdater().apply(_ctx(tmp_path))


def test_hook_malformed_report_line_fails_loud(monkeypatch, tmp_path) -> None:
    def bad(root: Path) -> None:
        (root / ".vergil" / "deps-report.jsonl").write_text("{oops\n")

    _setup(monkeypatch, tmp_path, run=bad)
    with pytest.raises(UpdateDepsError, match="line 1"):
        RepoHookUpdater().apply(_ctx(tmp_path))
```

- [ ] **Step 11: Run. Expect FAIL (no `repo_hook` module)**

Run: `vrg-container-run -- uv run pytest tests/vergil_tooling/test_update_deps_repo_hook.py -v`
Expected: FAIL with `ModuleNotFoundError`.

- [ ] **Step 12: Write the hook runner**

```python
# src/vergil_tooling/lib/update_deps/updaters/repo_hook.py
"""Repo-hook updater: run ``[dependency-update].hook`` in the dev container.

Spec §6.2 (epic vergil-project/.github#381). Last in the registry, so the hook
sees lockfiles the built-in updaters already refreshed. The hook may edit files
in the worktree; it must not commit or otherwise move HEAD. Held-back entries
are appended to ``$VRG_DEPS_REPORT``, a file inside the worktree (``.vergil/``
is gitignored fleet-wide) so the container sees it at ``/workspace/...``.
"""

from __future__ import annotations

import shlex
import subprocess
from typing import TYPE_CHECKING

from vergil_tooling.lib import config, git, progress
from vergil_tooling.lib.update_deps import report
from vergil_tooling.lib.update_deps.context import UpdateDepsError
from vergil_tooling.lib.update_deps.updater import UpdateResult

if TYPE_CHECKING:
    from pathlib import Path

    from vergil_tooling.lib.update_deps.context import UpdateDepsContext

_REPORT_REL = ".vergil/deps-report.jsonl"
_COMMIT_MESSAGE = "chore(deps): repo hook"


def _base(ctx: UpdateDepsContext) -> Path:
    return ctx.worktree_path if ctx.worktree_path is not None else ctx.repo_root


class RepoHookUpdater:
    """Run the repo-owned dependency-update hook (tier 2 of the spec §4 taxonomy)."""

    name = "hook"

    def applies(self, ctx: UpdateDepsContext) -> bool:
        base = _base(ctx)
        if not (base / "vergil.toml").is_file():
            return False
        return config.dependency_update_hook(base) is not None

    def apply(self, ctx: UpdateDepsContext) -> UpdateResult:
        base = _base(ctx)
        hook = config.dependency_update_hook(base)
        if hook is None:  # applies() gates this; re-checked so a race fails loud
            raise UpdateDepsError(
                phase="update", command="hook", message="[dependency-update].hook is not set."
            )
        report_path = base / _REPORT_REL
        report_path.parent.mkdir(parents=True, exist_ok=True)
        report_path.write_text("")
        cmd = [
            "vrg-container-run", "--", "env",
            f"VRG_DEPS_REPORT=/workspace/{_REPORT_REL}",
            *shlex.split(hook),
        ]
        head_before = git.read_output("rev-parse", "HEAD")
        try:
            progress.run(cmd)
        except subprocess.CalledProcessError as exc:
            raise UpdateDepsError(
                phase="update",
                command=" ".join(cmd),
                message=f"Repository hook failed (exit {exc.returncode}): {hook}",
                detail=(exc.stderr or exc.output or ""),
            ) from exc
        head_after = git.read_output("rev-parse", "HEAD")
        if head_after != head_before:
            raise UpdateDepsError(
                phase="update",
                command=hook,
                message=(
                    f"Repository hook moved HEAD ({head_before[:8]} -> {head_after[:8]}). "
                    "Hooks may edit files but must not commit; the driver commits."
                ),
            )
        entries = report.parse_report_lines(report_path.read_text(), origin=hook)
        report_path.unlink()
        changed = bool(git.read_output("status", "--porcelain").strip())
        return UpdateResult(
            updater=self.name,
            changed=changed,
            summary=f"repo hook `{hook}`" + ("" if changed else " (no changes)"),
            commit_message=_COMMIT_MESSAGE,
            held_back=entries,
        )
```

- [ ] **Step 13: Register it last**

In `src/vergil_tooling/lib/update_deps/orchestrator.py`:

```python
from vergil_tooling.lib.update_deps.updaters.repo_hook import RepoHookUpdater

# The repo hook is always last: it sees what the built-in updaters produced (spec §6.2).
DEFAULT_REGISTRY: list[Updater] = [PythonUvUpdater(), VergilUpdater(), RepoHookUpdater()]
```

Update `test_default_registry_names` in
`tests/vergil_tooling/test_update_deps_orchestrator.py` to expect
`["python", "vergil", "hook"]`.

- [ ] **Step 14: Run every update-deps and config test. Expect PASS**

Run: `vrg-container-run -- uv run pytest tests/vergil_tooling -k "update_deps or config" -v`
Expected: PASS.

- [ ] **Step 15: Full validation, then commit**

Run: `vrg-container-run -- vrg-validate`. Expected: PASS with 100% coverage.

```bash
vrg-git add src/vergil_tooling/lib/update_deps/report.py \
  src/vergil_tooling/lib/update_deps/updaters/repo_hook.py \
  src/vergil_tooling/lib/update_deps/updater.py \
  src/vergil_tooling/lib/update_deps/orchestrator.py \
  src/vergil_tooling/lib/config.py \
  tests/vergil_tooling/test_update_deps_report.py \
  tests/vergil_tooling/test_update_deps_repo_hook.py \
  tests/vergil_tooling/test_config.py \
  tests/vergil_tooling/test_update_deps_orchestrator.py
vrg-commit --type feat --scope update-deps \
  --message "add held-back report, [dependency-update] config and the repo-hook updater"
```

---

### Task 2: Driver failure handling, labelled PR, fix task, and `run_update()` [`vergil-tooling`]

Covers spec §5.1 (branch name, no-op), §5.2 (label, failure paths, failure
output, fix task) and the driver row of §9. It also removes the stale
`Ref #1379` from the PR body.

**Files:**

- Create: `src/vergil_tooling/lib/update_deps/failure.py`
- Modify: `src/vergil_tooling/lib/update_deps/context.py` (`DepsFailure`, `DEPS_LABEL`, new fields, `has_work`)
- Modify: `src/vergil_tooling/lib/update_deps/preflight.py` (branch `deps/update-<version>`)
- Modify: `src/vergil_tooling/lib/update_deps/pr.py` (label, body, merge failure, fix task)
- Modify: `src/vergil_tooling/lib/update_deps/orchestrator.py` (record-don't-raise stages, `fix-task` and `status` stages, `run_update()`)
- Modify: `src/vergil_tooling/lib/github.py` (`create_pr(..., labels=...)`)
- Modify: `src/vergil_tooling/data/labels.json` (add `dependency-update`)
- Test: `tests/vergil_tooling/test_update_deps_failure.py` (new)
- Test: `tests/vergil_tooling/test_update_deps_preflight.py`
- Test: `tests/vergil_tooling/test_update_deps_pr.py`
- Test: `tests/vergil_tooling/test_update_deps_orchestrator.py`
- Test: `tests/vergil_tooling/test_github.py` (or wherever `create_pr` is tested; grep `def test_create_pr`)

**Interfaces:**

- Consumes: `report.collect`, `report.render_markdown` (Task 1);
  `version.show(repo_root) -> str`; `github.create_pr`, `github.edit_pr_body`,
  `github.failed_check_names(pr) -> list[str]`; `pr_merge.wait_and_merge`;
  the `vrg-ensure-label` and `vrg-issue-create` CLIs.
- Produces:
  - `context.DEPS_LABEL = "dependency-update"`
  - `context.DepsFailure(kind: Literal["update", "validation", "merge"], message: str, output_tail: str, reproduce: str)`
  - `UpdateDepsContext.version: str | None`, `.failure: DepsFailure | None`,
    `.fix_issue_url: str | None`, property `.has_work -> bool`
  - `failure.tail_output(text: str, *, lines: int = 200, max_chars: int = 20_000) -> str`
  - `failure.failure_markdown(f: DepsFailure) -> str`
  - `failure.failure_summary(ctx: UpdateDepsContext) -> str`
  - `pr.FIX_TASK_PREFIX = "Fix task: "` (the line the release gate parses, Task 4)
  - `pr.fix_task_title(version: str) -> str`
  - `orchestrator.run_update(*, repo_root: Path, registry: list[Updater] | None = None) -> UpdateDepsContext`
  - `github.create_pr(*, base, title, body_file, head=None, labels: Sequence[str] = ()) -> str`
  - Stage names: `preflight, update, validate, prepare-pr, merge, fix-task, finalize, status`

- [ ] **Step 1: Failing tests for `failure.py`**

```python
# tests/vergil_tooling/test_update_deps_failure.py
from __future__ import annotations

from pathlib import Path

from vergil_tooling.lib.update_deps import failure
from vergil_tooling.lib.update_deps.context import DepsFailure, UpdateDepsContext


def test_tail_output_keeps_last_lines() -> None:
    text = "\n".join(f"line {i}" for i in range(1000))
    tail = failure.tail_output(text, lines=200)
    assert tail.splitlines()[0].startswith("[… 800 earlier lines omitted …]")
    assert tail.splitlines()[-1] == "line 999"
    assert "line 799" not in tail
    assert "line 800" in tail


def test_tail_output_short_text_unchanged() -> None:
    assert failure.tail_output("a\nb") == "a\nb"


def test_tail_output_caps_characters() -> None:
    text = "\n".join("x" * 1000 for _ in range(150))  # 150 lines, ~150k chars
    tail = failure.tail_output(text)
    assert len(tail) <= 20_000 + 100  # cap + the omission marker line
    assert tail.endswith("x" * 1000)


def test_failure_markdown_has_tail_and_reproduce() -> None:
    f = DepsFailure(kind="validation", message="Validation failed.",
                    output_tail="E   assert 1 == 2", reproduce="vrg-container-run -- vrg-validate")
    text = failure.failure_markdown(f)
    assert "## Failure: validation" in text
    assert "```text\nE   assert 1 == 2\n```" in text
    assert "`vrg-container-run -- vrg-validate`" in text


def test_failure_summary_names_pr_and_fix_task() -> None:
    ctx = UpdateDepsContext(repo="o/r", repo_root=Path("/r"), version="2.1.243")
    ctx.failure = DepsFailure(kind="merge", message="checks failed", output_tail="", reproduce="x")
    ctx.pr_url = "https://github.com/o/r/pull/9"
    ctx.fix_issue_url = "https://github.com/o/r/issues/10"
    text = failure.failure_summary(ctx)
    assert "2.1.243" in text and "merge" in text
    assert "pull/9" in text and "issues/10" in text


def test_failure_summary_update_kind_has_no_pr() -> None:
    ctx = UpdateDepsContext(repo="o/r", repo_root=Path("/r"), version="2.1.243")
    ctx.failure = DepsFailure(kind="update", message="hook failed", output_tail="", reproduce="x")
    assert "No PR was opened" in failure.failure_summary(ctx)
```

- [ ] **Step 2: Run. Expect FAIL (no `DepsFailure` and no `failure` module)**

Run: `vrg-container-run -- uv run pytest tests/vergil_tooling/test_update_deps_failure.py -v`
Expected: FAIL.

- [ ] **Step 3: Extend `context.py`**

```python
from typing import TYPE_CHECKING, Literal

# Every dependency-update PR carries this label from creation (spec §5.2); the
# vrg-release preflight gate refuses while any open PR has it (spec §5.3).
DEPS_LABEL = "dependency-update"

FailureKind = Literal["update", "validation", "merge"]


@dataclass
class DepsFailure:
    """A recorded (not raised) failure; the status stage surfaces it loud.

    ``update`` — an updater or hook failed before validation: no PR.
    ``validation`` — the single vrg-validate run failed: PR stays open.
    ``merge`` — remote CI failed or the merge aborted: PR stays open.
    """

    kind: FailureKind
    message: str
    output_tail: str
    reproduce: str
```

Add these fields to `UpdateDepsContext`:

```python
    # The develop version the update is the first change on (spec §5.1).
    version: str | None = None
    failure: DepsFailure | None = None
    fix_issue_url: str | None = None

    @property
    def has_work(self) -> bool:
        """True when there is a diff that goes through validation and a PR.

        A pre-validation (``update``) failure has no PR, even if an earlier
        updater already committed (spec §5.2).
        """
        return self.any_changes and (self.failure is None or self.failure.kind != "update")
```

- [ ] **Step 4: Write `failure.py`**

```python
# src/vergil_tooling/lib/update_deps/failure.py
"""Failure output for dependency-update PRs, fix tasks and release comments.

Spec §5.2: carry the last ~200 lines of output plus the reproduce command, never
the full output. GitHub rejects bodies over 65,536 characters, and a rejected
body would leave no PR for the release gate to see.
"""

from __future__ import annotations

from typing import TYPE_CHECKING

if TYPE_CHECKING:
    from vergil_tooling.lib.update_deps.context import DepsFailure, UpdateDepsContext


def tail_output(text: str, *, lines: int = 200, max_chars: int = 20_000) -> str:
    """The last *lines* lines of *text*, at most *max_chars* characters, with an omission marker."""
    all_lines = text.splitlines()
    kept = all_lines[-lines:]
    omitted = len(all_lines) - len(kept)
    body = "\n".join(kept)
    if len(body) > max_chars:
        body = body[-max_chars:]
        omitted_note = "[… output truncated to the last 20000 characters …]"
        return f"{omitted_note}\n{body}"
    if omitted:
        return f"[… {omitted} earlier lines omitted …]\n{body}"
    return body


def failure_markdown(failure: DepsFailure) -> str:
    """A Markdown section describing *failure* for a PR body or issue."""
    return "\n".join([
        f"## Failure: {failure.kind}",
        "",
        failure.message,
        "",
        "Last lines of output:",
        "",
        "```text",
        failure.output_tail,
        "```",
        "",
        f"Reproduce on the branch: `{failure.reproduce}`",
    ])


def failure_summary(ctx: UpdateDepsContext) -> str:
    """One-paragraph summary for the status stage and the release tracking issue."""
    f = ctx.failure
    if f is None:  # callers check first; fail loud rather than print "None"
        msg = "failure_summary called without a recorded failure"
        raise ValueError(msg)
    head = f"Dependency update for {ctx.version} failed ({f.kind}): {f.message}"
    if f.kind == "update":
        return f"{head} No PR was opened."
    parts = [head, f"The PR stays open: {ctx.pr_url}."]
    if ctx.fix_issue_url:
        parts.append(f"Fix task: {ctx.fix_issue_url}.")
    parts.append("The next vrg-release is gated until it lands.")
    return " ".join(parts)
```

- [ ] **Step 5: Run the failure tests. Expect PASS**

Run: `vrg-container-run -- uv run pytest tests/vergil_tooling/test_update_deps_failure.py -v`
Expected: PASS.

- [ ] **Step 6: Failing preflight tests (branch name, collision)**

In `tests/vergil_tooling/test_update_deps_preflight.py`, change the expected
branch in the existing success test from `chore/dep-update-<date>` to
`deps/update-2.1.243` (patch `_MOD + ".version.show"` to return `"2.1.243"`),
and assert `ctx.version == "2.1.243"`. Then add:

```python
def test_preflight_existing_deps_branch_fails_loud(monkeypatch) -> None:
    _patch_happy_path(monkeypatch)  # the existing helper, or inline the same patches
    monkeypatch.setattr(_MOD + ".version.show", lambda root: "2.1.243")

    def collide(root, *, branch, base):
        raise ManagedWorktreeError(f"branch {branch} already exists")

    monkeypatch.setattr(_MOD + ".create_worktree", collide)
    with pytest.raises(UpdateDepsError, match=r"deps/update-2\.1\.243 already exists") as info:
        preflight(repo_root=Path("/tmp/r"))  # noqa: S108
    assert "dependency-update PR" in str(info.value)
```

(If the file has no `_patch_happy_path` helper, extract the existing success
test's patches into one in this step.)

- [ ] **Step 7: Run. Expect FAIL**

Run: `vrg-container-run -- uv run pytest tests/vergil_tooling/test_update_deps_preflight.py -v`
Expected: FAIL (the branch is still date-based).

- [ ] **Step 8: Update `preflight.py`**

Remove `_today()` and the `datetime` import. Import `version` from
`vergil_tooling.lib`, and replace the branch block:

```python
    develop_version = version.show(repo_root)
    # deps/ is deliberately not an issue-branch prefix (lib/branch_names.py):
    # only this driver commits here. Fixes go on chore/<N>-dep-update-<version>
    # (spec §5.1, §5.2). Version-keyed, because repos release several times a day.
    branch = f"deps/update-{develop_version}"
    try:
        worktree_path = create_worktree(repo_root, branch=branch, base="develop")
    except ManagedWorktreeError as exc:
        raise UpdateDepsError(
            phase="preflight",
            command=f"git worktree add {branch}",
            message=(
                f"{branch} already exists ({exc}). An earlier dependency update for "
                f"{develop_version} is still in flight; land or close its "
                "dependency-update PR, then delete the branch."
            ),
        ) from exc
```

Set `version=develop_version` on the returned `UpdateDepsContext`.

- [ ] **Step 9: Run the preflight tests. Expect PASS**

Run: `vrg-container-run -- uv run pytest tests/vergil_tooling/test_update_deps_preflight.py -v`
Expected: PASS.

- [ ] **Step 10: Failing tests for `create_pr` labels and the PR helpers**

Add a `create_pr` test next to the existing ones (find them with
`grep -rn "def test_create_pr" tests/`):

```python
def test_create_pr_passes_labels(monkeypatch) -> None:
    seen: list[tuple[str, ...]] = []
    monkeypatch.setattr(github, "read_output", lambda *a: seen.append(a) or "https://x/pull/1")
    github.create_pr(base="develop", title="t", body_file="/b", labels=["dependency-update"])
    assert seen[0][-2:] == ("--label", "dependency-update")
```

Replace the body tests in `tests/vergil_tooling/test_update_deps_pr.py` and add
the new behaviour:

```python
from vergil_tooling.lib.update_deps import pr
from vergil_tooling.lib.update_deps.context import DepsFailure, UpdateDepsContext
from vergil_tooling.lib.update_deps.report import HeldBack
from vergil_tooling.lib.update_deps.updater import UpdateResult

_MOD = "vergil_tooling.lib.update_deps.pr"


def _ctx() -> UpdateDepsContext:
    ctx = UpdateDepsContext(repo="o/r", repo_root=Path("/r"), version="2.1.243",
                            branch="deps/update-2.1.243", worktree_path=Path("/r/.wt"))
    ctx.results = [UpdateResult(updater="python", changed=True, summary="uv lock --upgrade",
                                commit_message="c", held_back=[
                                    HeldBack(source="python", name="p", current="1", latest="2")])]
    return ctx


def test_pr_body_has_updates_and_report_and_no_stale_ref() -> None:
    body = pr.build_pr_body(_ctx())
    assert "**python** — uv lock --upgrade" in body
    assert "## Held back by declared bounds" in body
    assert "| python | p | 1 | 2 |" in body
    assert "#1379" not in body


def test_pr_body_includes_failure_and_fix_task() -> None:
    ctx = _ctx()
    ctx.failure = DepsFailure(kind="validation", message="Validation failed.",
                              output_tail="boom", reproduce="vrg-container-run -- vrg-validate")
    ctx.fix_issue_url = "https://github.com/o/r/issues/10"
    body = pr.build_pr_body(ctx)
    assert "## Failure: validation" in body
    assert "\nFix task: https://github.com/o/r/issues/10\n" in body


def test_pr_body_stays_under_github_limit() -> None:
    ctx = _ctx()
    ctx.results[0].held_back = [
        HeldBack(source="python", name=f"pkg{i}", current="1.0.0", latest="2.0.0",
                 note="pyproject: pkg>=1,<2") for i in range(2000)
    ]
    huge = "\n".join("E" * 300 for _ in range(10_000))
    from vergil_tooling.lib.update_deps.failure import tail_output
    ctx.failure = DepsFailure(kind="validation", message="m", output_tail=tail_output(huge),
                              reproduce="r")
    assert len(pr.build_pr_body(ctx)) < 65_536


def test_prepare_pr_ensures_label_and_labels_pr(monkeypatch) -> None:
    runs: list[list[str]] = []
    monkeypatch.setattr(_MOD + ".progress.run", lambda cmd: runs.append(cmd) or 0)
    monkeypatch.setattr(_MOD + ".git.run", lambda *a: None)
    created: dict[str, object] = {}

    def fake_create(**kw):
        created.update(kw)
        return "https://github.com/o/r/pull/9"

    monkeypatch.setattr(_MOD + ".github.create_pr", fake_create)
    ctx = _ctx()
    pr.prepare_pr(ctx)
    assert runs[0][:5] == ["vrg-ensure-label", "--repo", "o/r", "--label", "dependency-update"]
    assert created["labels"] == ["dependency-update"]
    assert created["title"] == "chore(deps): dependency update for 2.1.243"
    assert ctx.pr_url == "https://github.com/o/r/pull/9"


def test_record_merge_failure_lists_failed_checks(monkeypatch) -> None:
    monkeypatch.setattr(_MOD + ".github.failed_check_names", lambda url: ["test / 3.12"])
    edits: list[str] = []
    monkeypatch.setattr(_MOD + ".github.edit_pr_body", lambda url, *, body: edits.append(body))
    ctx = _ctx()
    ctx.pr_url = "https://github.com/o/r/pull/9"
    pr.record_merge_failure(ctx, "required checks failed")
    assert ctx.failure is not None and ctx.failure.kind == "merge"
    assert "test / 3.12" in ctx.failure.output_tail
    assert ctx.failure.reproduce == "https://github.com/o/r/pull/9/checks"
    assert "## Failure: merge" in edits[0]


def test_file_fix_task_creates_adhoc_issue_and_links_pr(monkeypatch) -> None:
    seen: dict[str, object] = {}

    def fake_run(cmd, **kw):
        seen["cmd"] = cmd
        return subprocess.CompletedProcess(
            cmd, 0,
            stdout="Created https://github.com/o/r/issues/10, linked under epic o/.github#5.\n",
            stderr="")

    monkeypatch.setattr(_MOD + ".subprocess.run", fake_run)
    edits: list[str] = []
    monkeypatch.setattr(_MOD + ".github.edit_pr_body", lambda url, *, body: edits.append(body))
    ctx = _ctx()
    ctx.pr_url = "https://github.com/o/r/pull/9"
    ctx.failure = DepsFailure(kind="validation", message="m", output_tail="t", reproduce="r")
    pr.file_fix_task(ctx)
    cmd = seen["cmd"]
    assert cmd[:5] == ["vrg-issue-create", "--epic", "adhoc", "--repo", "o/r"]
    assert cmd[cmd.index("--title") + 1] == "Fix failed dependency update for 2.1.243"
    assert ctx.fix_issue_url == "https://github.com/o/r/issues/10"
    assert "Fix task: https://github.com/o/r/issues/10" in edits[0]


def test_file_fix_task_fails_loud_on_error(monkeypatch) -> None:
    monkeypatch.setattr(_MOD + ".subprocess.run", lambda cmd, **kw: subprocess.CompletedProcess(
        cmd, 1, stdout="", stderr="no installation"))
    ctx = _ctx()
    ctx.pr_url = "u"
    ctx.failure = DepsFailure(kind="validation", message="m", output_tail="t", reproduce="r")
    with pytest.raises(UpdateDepsError, match="vrg-issue-create failed"):
        pr.file_fix_task(ctx)


def test_file_fix_task_fails_loud_on_unparseable_output(monkeypatch) -> None:
    monkeypatch.setattr(_MOD + ".subprocess.run", lambda cmd, **kw: subprocess.CompletedProcess(
        cmd, 0, stdout="ok\n", stderr=""))
    ctx = _ctx()
    ctx.pr_url = "u"
    ctx.failure = DepsFailure(kind="validation", message="m", output_tail="t", reproduce="r")
    with pytest.raises(UpdateDepsError, match="could not find the created issue URL"):
        pr.file_fix_task(ctx)
```

Also add the fix-task body test:

```python
def test_fix_task_body_explains_branch_and_reproduce() -> None:
    ctx = _ctx()
    ctx.pr_url = "https://github.com/o/r/pull/9"
    ctx.failure = DepsFailure(kind="validation", message="m", output_tail="boom",
                              reproduce="vrg-container-run -- vrg-validate")
    body = pr.build_fix_task_body(ctx)
    assert "https://github.com/o/r/pull/9" in body
    assert "chore/<this issue number>-dep-update-2.1.243" in body
    assert "origin/deps/update-2.1.243" in body
    assert "boom" in body
```

- [ ] **Step 11: Run. Expect FAIL**

Run: `vrg-container-run -- uv run pytest tests/vergil_tooling/test_update_deps_pr.py tests/vergil_tooling/test_github.py -k "pr or label" -v`
Expected: FAIL.

- [ ] **Step 12: `github.create_pr` labels**

```python
def create_pr(
    *,
    base: str,
    title: str,
    body_file: str,
    head: str | None = None,
    labels: Sequence[str] = (),
) -> str:
    """Create a pull request and return its URL.

    (keep the existing *head* paragraph)

    *labels* are applied at creation (``--label``, repeatable), so the PR is never
    visible unlabelled — the dependency-update gate depends on that (spec §5.2).
    """
    args = ["pr", "create", "--base", base, "--title", title, "--body-file", body_file]
    if head is not None:
        args += ["--head", head]
    for label in labels:
        args += ["--label", label]
    return read_output(*args)
```

(Import `Sequence` under `TYPE_CHECKING` from `collections.abc` if
`github.py` does not already import it.)

- [ ] **Step 13: Add the label to the registry**

In `src/vergil_tooling/data/labels.json`, after `observability`:

```json
    {"name": "dependency-update", "color": "0366d6", "description": "Mechanized dependency update; an open one gates the next release"},
```

- [ ] **Step 14: Rewrite `pr.py`**

```python
"""PR, fix-task and worktree helpers for vrg-update-deps."""

from __future__ import annotations

import os
import re
import subprocess
import tempfile
from pathlib import Path
from typing import TYPE_CHECKING

from vergil_tooling.lib import git, github, pr_merge, progress
from vergil_tooling.lib.managed_worktree import remove_worktree
from vergil_tooling.lib.update_deps import report
from vergil_tooling.lib.update_deps.context import DEPS_LABEL, DepsFailure, UpdateDepsError
from vergil_tooling.lib.update_deps.failure import failure_markdown, tail_output

if TYPE_CHECKING:
    from vergil_tooling.lib.update_deps.context import UpdateDepsContext

# The release gate parses this line out of the PR body (spec §5.3).
FIX_TASK_PREFIX = "Fix task: "
_LABEL_COLOR = "0366d6"
_LABEL_DESCRIPTION = "Mechanized dependency update; an open one gates the next release"
_CREATED_RE = re.compile(r"Created (https://github\.com/\S+/issues/\d+),")


def fix_task_title(version: str) -> str:
    return f"Fix failed dependency update for {version}"


def _pr_title(ctx: UpdateDepsContext) -> str:
    return f"chore(deps): dependency update for {ctx.version}"


def build_pr_body(ctx: UpdateDepsContext) -> str:
    """Build the PR body: updates, warnings, held-back report, and any failure."""
    lines = ["Mechanized dependency update (`vrg-update-deps`).", "", "## Updated", ""]
    lines.extend(f"- **{r.updater}** — {r.summary}" for r in ctx.results if r.changed)
    warnings = [w for r in ctx.results for w in r.warnings]
    if warnings:
        lines.extend(["", "## Warnings", ""])
        lines.extend(f"- {w}" for w in warnings)
    lines.extend(["", "## Held back by declared bounds", "",
                  report.render_markdown(report.collect(ctx.results))])
    if ctx.failure is not None:
        lines.extend(["", failure_markdown(ctx.failure)])
    if ctx.fix_issue_url is not None:
        lines.extend(["", f"{FIX_TASK_PREFIX}{ctx.fix_issue_url}"])
    return "\n".join(lines) + "\n"


def _ensure_label(repo: str) -> None:
    progress.run([
        "vrg-ensure-label", "--repo", repo, "--label", DEPS_LABEL,
        "--color", _LABEL_COLOR, "--description", _LABEL_DESCRIPTION,
    ])


def prepare_pr(ctx: UpdateDepsContext) -> None:
    """Push the worktree branch and open the labelled PR to develop."""
    if ctx.branch is None:
        raise UpdateDepsError(
            phase="prepare-pr",
            command="prepare_pr",
            message="No branch on context — preflight did not run.",
        )
    git.run("push", "-u", "origin", ctx.branch)
    _ensure_label(ctx.repo)
    with tempfile.NamedTemporaryFile("w", suffix=".md", delete=False) as handle:
        handle.write(build_pr_body(ctx))
        body_path = Path(handle.name)
    try:
        ctx.pr_url = github.create_pr(
            base="develop", title=_pr_title(ctx), body_file=str(body_path), labels=[DEPS_LABEL]
        )
    finally:
        body_path.unlink(missing_ok=True)
    print(f"Opened PR: {ctx.pr_url}")


def merge_pr(ctx: UpdateDepsContext) -> None:
    """Wait for checks and merge; a merge abort is recorded, not raised (spec §5.2)."""
    if ctx.pr_url is None:
        raise UpdateDepsError(
            phase="merge",
            command="merge_pr",
            message="No PR URL on context — prepare-pr did not run.",
        )
    try:
        pr_merge.wait_and_merge(ctx.pr_url, strategy="merge")
    except pr_merge.MergeAbortError as exc:
        record_merge_failure(ctx, str(exc))


def record_merge_failure(ctx: UpdateDepsContext, message: str) -> None:
    """Record a remote-CI or merge failure and put it in the PR body."""
    pr_url = ctx.pr_url or ""
    failed = github.failed_check_names(pr_url)
    output = message + ("\n\nFailed checks:\n" + "\n".join(f"- {n}" for n in failed)
                        if failed else "")
    ctx.failure = DepsFailure(
        kind="merge",
        message="The dependency-update PR could not merge.",
        output_tail=tail_output(output),
        reproduce=f"{pr_url}/checks",
    )
    github.edit_pr_body(pr_url, body=build_pr_body(ctx))


def build_fix_task_body(ctx: UpdateDepsContext) -> str:
    failure = ctx.failure
    if failure is None:
        msg = "build_fix_task_body called without a recorded failure"
        raise ValueError(msg)
    return "\n".join([
        f"The mechanized dependency update for `{ctx.version}` failed. Its PR stays "
        f"open, labelled `{DEPS_LABEL}`, and gates the next `vrg-release`: {ctx.pr_url}",
        "",
        "## How to fix",
        "",
        f"Cut the fix branch from the failed update, so it starts from its commits: "
        f"`chore/<this issue number>-dep-update-{ctx.version}` from "
        f"`origin/deps/update-{ctx.version}`. Fix, then hand off as usual. When the fix "
        "PR merges, GitHub marks the dependency-update PR merged and the gate clears.",
        "",
        failure_markdown(failure),
    ]) + "\n"


def file_fix_task(ctx: UpdateDepsContext) -> None:
    """File the fix task under the repo's ad-hoc epic and link it from the PR body."""
    if ctx.version is None or ctx.pr_url is None:
        raise UpdateDepsError(phase="fix-task", command="file_fix_task",
                              message="No version or PR on context.")
    with tempfile.NamedTemporaryFile("w", suffix=".md", delete=False) as handle:
        handle.write(build_fix_task_body(ctx))
        body_path = Path(handle.name)
    cmd = ["vrg-issue-create", "--epic", "adhoc", "--repo", ctx.repo,
           "--title", fix_task_title(ctx.version), "--body-file", str(body_path)]
    try:
        result = subprocess.run(cmd, check=False, capture_output=True, text=True)  # noqa: S603
    finally:
        body_path.unlink(missing_ok=True)
    if result.returncode != 0:
        raise UpdateDepsError(phase="fix-task", command=" ".join(cmd),
                              message="vrg-issue-create failed; no fix task was filed.",
                              detail=result.stderr)
    match = _CREATED_RE.search(result.stdout)
    if match is None:
        raise UpdateDepsError(phase="fix-task", command=" ".join(cmd),
                              message="vrg-issue-create succeeded but could not find the "
                                      "created issue URL in its output.",
                              detail=result.stdout)
    ctx.fix_issue_url = match.group(1)
    github.edit_pr_body(ctx.pr_url, body=build_pr_body(ctx))
    print(f"Filed fix task: {ctx.fix_issue_url}")


def cleanup_worktree(ctx: UpdateDepsContext) -> None:
    """Return to the root checkout and remove the managed worktree + local branch.

    On a failure the branch is already on origin (the PR's head), so deleting the
    local copy loses nothing.
    """
    if ctx.worktree_path is None:
        return
    os.chdir(ctx.repo_root)
    remove_worktree(ctx.worktree_path)
    if ctx.branch is not None:
        git.run("branch", "-D", ctx.branch)
```

- [ ] **Step 15: Run the pr and github tests. Expect PASS**

Run: `vrg-container-run -- uv run pytest tests/vergil_tooling/test_update_deps_pr.py tests/vergil_tooling/test_github.py -v`
Expected: PASS. Delete the old `merge_pr` test that expected `UpdateDepsError`
on `MergeAbortError`, and replace it with one asserting `record_merge_failure`
was called.

- [ ] **Step 16: Failing orchestrator tests**

Replace `test_validate_stage_aborts_on_red` and add:

```python
def test_build_stages_names_and_modes() -> None:
    assert [(s.name, s.mode) for s in build_stages()] == [
        ("preflight", "fail_fast"), ("update", "fail_fast"), ("validate", "fail_fast"),
        ("prepare-pr", "fail_fast"), ("merge", "fail_fast"), ("fix-task", "fail_fast"),
        ("finalize", "fail_defer"), ("status", "fail_defer"),
    ]


def test_updater_error_is_recorded_and_stops_updaters(monkeypatch) -> None:
    class _Boom:
        name = "boom"
        def applies(self, ctx): return True
        def apply(self, ctx):
            raise UpdateDepsError(phase="update", command="hook", message="hook failed",
                                  detail="trace")

    monkeypatch.setattr(_MOD + ".git.run", lambda *a: None)
    state, ctx = _state()
    state.registry = [_Changed(), _Boom(), _NoChange()]
    run_updaters_stage(state)
    assert ctx.failure is not None and ctx.failure.kind == "update"
    assert "hook failed" in ctx.failure.message
    assert [r.updater for r in ctx.results] == ["changed"]
    assert ctx.has_work is False


def test_validation_failure_is_recorded_not_raised(monkeypatch) -> None:
    def _red() -> None:
        raise UpdateDepsError(phase="validate", command="vrg-container-run -- vrg-validate",
                              message="Validation failed.", detail="E assert 1 == 2")

    monkeypatch.setattr(_MOD + ".validate.run_validation", _red)
    state, ctx = _state()
    ctx.any_changes = True
    validate_stage(state)
    assert ctx.failure is not None and ctx.failure.kind == "validation"
    assert "E assert 1 == 2" in ctx.failure.output_tail
    assert ctx.failure.reproduce == "vrg-container-run -- vrg-validate"


def test_validation_failure_still_opens_pr_but_skips_merge(monkeypatch) -> None:
    calls: list[str] = []
    monkeypatch.setattr(_MOD + ".pr.prepare_pr", lambda ctx: calls.append("prepare"))
    monkeypatch.setattr(_MOD + ".pr.merge_pr", lambda ctx: calls.append("merge"))
    monkeypatch.setattr(_MOD + ".pr.file_fix_task", lambda ctx: calls.append("fix"))
    state, ctx = _state()
    ctx.any_changes = True
    ctx.failure = DepsFailure(kind="validation", message="m", output_tail="t", reproduce="r")
    prepare_pr_stage(state)
    merge_stage(state)
    fix_task_stage(state)
    assert calls == ["prepare", "fix"]


def test_update_failure_opens_no_pr_and_files_no_task(monkeypatch) -> None:
    calls: list[str] = []
    monkeypatch.setattr(_MOD + ".pr.prepare_pr", lambda ctx: calls.append("prepare"))
    monkeypatch.setattr(_MOD + ".pr.file_fix_task", lambda ctx: calls.append("fix"))
    monkeypatch.setattr(_MOD + ".validate.run_validation", lambda: calls.append("validate"))
    state, ctx = _state()
    ctx.any_changes = True
    ctx.failure = DepsFailure(kind="update", message="m", output_tail="t", reproduce="r")
    validate_stage(state)
    prepare_pr_stage(state)
    fix_task_stage(state)
    assert calls == []


def test_finalize_prints_report_on_noop(monkeypatch, capsys) -> None:
    monkeypatch.setattr(_MOD + ".pr.cleanup_worktree", lambda ctx: None)
    state, ctx = _state()
    ctx.results = [UpdateResult(updater="python", changed=False, summary="", commit_message="",
                                held_back=[HeldBack(source="python", name="p", current="1",
                                                    latest="2")])]
    finalize_stage(state)
    out = capsys.readouterr().out
    assert "Held back by declared bounds:" in out
    assert "| python | p | 1 | 2 |" in out
    assert "No updates found" in out


def test_status_stage_raises_on_recorded_failure() -> None:
    state, ctx = _state()
    ctx.version = "2.1.243"
    ctx.failure = DepsFailure(kind="update", message="hook failed", output_tail="t", reproduce="r")
    with pytest.raises(UpdateDepsError, match="2.1.243"):
        status_stage(state)


def test_status_stage_quiet_on_success() -> None:
    state, _ = _state()
    status_stage(state)  # no raise


def test_run_update_runs_stages_in_order_and_always_cleans_up(monkeypatch) -> None:
    order: list[str] = []
    ctx = UpdateDepsContext(repo="o/r", repo_root=Path("/tmp/r"))  # noqa: S108
    monkeypatch.setattr(_MOD + ".preflight", lambda *, repo_root: ctx)  # noqa: ARG005
    for name in ("run_updaters_stage", "validate_stage", "prepare_pr_stage", "merge_stage",
                 "fix_task_stage"):
        monkeypatch.setattr(f"{_MOD}.{name}", lambda s, n=name: order.append(n))
    monkeypatch.setattr(_MOD + ".finalize_stage", lambda s: order.append("finalize"))
    assert run_update(repo_root=Path("/tmp/r")) is ctx  # noqa: S108
    assert order == ["run_updaters_stage", "validate_stage", "prepare_pr_stage",
                     "merge_stage", "fix_task_stage", "finalize"]


def test_run_update_cleans_up_when_a_stage_raises(monkeypatch) -> None:
    order: list[str] = []
    ctx = UpdateDepsContext(repo="o/r", repo_root=Path("/tmp/r"))  # noqa: S108
    monkeypatch.setattr(_MOD + ".preflight", lambda *, repo_root: ctx)  # noqa: ARG005

    def boom(state):
        raise RuntimeError("git commit failed")

    monkeypatch.setattr(_MOD + ".run_updaters_stage", boom)
    monkeypatch.setattr(_MOD + ".finalize_stage", lambda s: order.append("finalize"))
    with pytest.raises(RuntimeError):
        run_update(repo_root=Path("/tmp/r"))  # noqa: S108
    assert order == ["finalize"]
```

Import `DepsFailure`, `HeldBack`, `fix_task_stage`, `status_stage` and
`run_update` at the top of the test file.

- [ ] **Step 17: Run. Expect FAIL**

Run: `vrg-container-run -- uv run pytest tests/vergil_tooling/test_update_deps_orchestrator.py -v`
Expected: FAIL.

- [ ] **Step 18: Rewrite the orchestrator stages**

In `src/vergil_tooling/lib/update_deps/orchestrator.py` (keep `UpdateDepsState`,
`_require_ctx` and `preflight_stage`), add `import subprocess` and these
imports:

```python
from vergil_tooling.lib.update_deps import pr, report, validate
from vergil_tooling.lib.update_deps.context import DepsFailure, UpdateDepsContext, UpdateDepsError
from vergil_tooling.lib.update_deps.failure import failure_summary, tail_output
```

(This replaces the existing `from vergil_tooling.lib.update_deps import pr, validate`.)

Then replace the stage functions:

```python
def run_updaters_stage(state: UpdateDepsState) -> None:
    """Run each applicable updater; record (don't raise) the first failure (spec §5.2)."""
    ctx = _require_ctx(state)
    selected = select_updaters(state.registry, only=state.only, skip=state.skip)
    for updater in applicable_updaters(ctx, registry=selected):
        try:
            result = updater.apply(ctx)
        except UpdateDepsError as exc:
            ctx.failure = DepsFailure(kind="update", message=f"{updater.name}: {exc}",
                                      output_tail=tail_output(exc.detail or ""),
                                      reproduce=exc.command)
            return
        except subprocess.CalledProcessError as exc:
            ctx.failure = DepsFailure(kind="update", message=f"{updater.name}: {exc}",
                                      output_tail=tail_output(exc.stderr or exc.output or ""),
                                      reproduce=" ".join(map(str, exc.cmd)))
            return
        ctx.results.append(result)
        if result.changed:
            ctx.any_changes = True
            git.run("add", "-A")
            git.run("commit", "-m", result.commit_message)


def validate_stage(state: UpdateDepsState) -> None:
    ctx = _require_ctx(state)
    if not ctx.has_work:
        print("No dependency changes to validate — skipping validation.")
        return
    try:
        validate.run_validation()
    except UpdateDepsError as exc:
        ctx.failure = DepsFailure(kind="validation", message=str(exc),
                                  output_tail=tail_output(exc.detail or ""),
                                  reproduce=exc.command)


def prepare_pr_stage(state: UpdateDepsState) -> None:
    """Open the labelled PR whenever there is work, including after a red validation."""
    ctx = _require_ctx(state)
    if ctx.has_work:
        pr.prepare_pr(ctx)


def merge_stage(state: UpdateDepsState) -> None:
    ctx = _require_ctx(state)
    if ctx.has_work and ctx.failure is None:
        pr.merge_pr(ctx)


def fix_task_stage(state: UpdateDepsState) -> None:
    ctx = _require_ctx(state)
    if ctx.failure is not None and ctx.failure.kind != "update":
        pr.file_fix_task(ctx)


def finalize_stage(state: UpdateDepsState) -> None:
    ctx = _require_ctx(state)
    # Every run reports what the bounds hold back, a no-op run included (spec §7.1).
    print("Held back by declared bounds:")
    print(report.render_markdown(report.collect(ctx.results)))
    if ctx.failure is not None:
        print(f"Dependency update did not land ({ctx.failure.kind}).")
    elif not ctx.any_changes:
        print("No updates found — removing worktree.")
    else:
        print(f"Dependency update merged: {ctx.pr_url}")
    pr.cleanup_worktree(ctx)


def status_stage(state: UpdateDepsState) -> None:
    """Surface a recorded failure as a non-zero exit, after cleanup has run."""
    ctx = _require_ctx(state)
    if ctx.failure is None:
        return
    raise UpdateDepsError(phase="status", command="vrg-update-deps",
                          message=failure_summary(ctx), detail=ctx.failure.output_tail)


def build_stages() -> list[Stage]:
    """The vrg-update-deps pipeline, in execution order."""
    return [
        Stage("preflight", preflight_stage, mode="fail_fast"),
        Stage("update", run_updaters_stage, mode="fail_fast"),
        Stage("validate", validate_stage, mode="fail_fast"),
        Stage("prepare-pr", prepare_pr_stage, mode="fail_fast"),
        Stage("merge", merge_stage, mode="fail_fast"),
        Stage("fix-task", fix_task_stage, mode="fail_fast"),
        Stage("finalize", finalize_stage, mode="fail_defer"),
        Stage("status", status_stage, mode="fail_defer"),
    ]


def run_update(*, repo_root: Path, registry: list[Updater] | None = None) -> UpdateDepsContext:
    """Run the pipeline as plain calls, for callers already inside a pipeline.

    vrg-release's update-deps stage uses this rather than nesting
    ``progress.run_pipeline``, which holds one module-global session. Cleanup
    always runs. A recorded failure is returned on the context, not raised: the
    caller decides how to surface it. Preflight errors and unexpected exceptions
    propagate.
    """
    state = UpdateDepsState(repo_root=repo_root)
    if registry is not None:
        state.registry = list(registry)
    preflight_stage(state)
    ctx = _require_ctx(state)
    try:
        for stage_fn in (run_updaters_stage, validate_stage, prepare_pr_stage,
                         merge_stage, fix_task_stage):
            stage_fn(state)
    finally:
        finalize_stage(state)
    return ctx
```

Move `from pathlib import Path` out of `TYPE_CHECKING` only if ruff asks; it
is used in an annotation only, so it can stay. Note that `run_update` calls the
stage functions through module globals, so the monkeypatched tests above
intercept them.

- [ ] **Step 19: Run every update-deps test. Expect PASS**

Run: `vrg-container-run -- uv run pytest tests/vergil_tooling -k update_deps -v`
Expected: PASS. Fix `test_vrg_update_deps.py` if it asserts the old stage list.

- [ ] **Step 20: Full validation, then commit**

Run: `vrg-container-run -- vrg-validate`. Expected: PASS with 100% coverage.

```bash
vrg-git add src/vergil_tooling/lib/update_deps src/vergil_tooling/lib/github.py \
  src/vergil_tooling/data/labels.json tests/vergil_tooling
vrg-commit --type feat --scope update-deps \
  --message "keep failed dependency updates open as labelled PRs with a fix task"
```

---

### Task 3: `update-deps` release stage and `--without-deps` [`vergil-tooling`]

Covers spec §5.1 (stage, mode, root sync, tracking, `--without-deps`) and §5.2
item 3 (record in the tracking issue and in `publish-status`).

**Files:**

- Create: `src/vergil_tooling/lib/release/update_deps_stage.py`
- Modify: `src/vergil_tooling/lib/release/context.py` (deps fields)
- Modify: `src/vergil_tooling/lib/release/orchestrator.py` (stage, details, publish-status)
- Modify: `src/vergil_tooling/bin/vrg_release.py` (`--without-deps`)
- Test: `tests/vergil_tooling/test_release_update_deps_stage.py` (new)
- Test: `tests/vergil_tooling/test_release_orchestrator.py`
- Test: `tests/vergil_tooling/test_vrg_release.py`

**Interfaces:**

- Consumes: `orchestrator.run_update(*, repo_root) -> UpdateDepsContext`,
  `failure.failure_summary`, `failure.failure_markdown`, `report.collect`,
  `report.render_markdown` (Tasks 1 and 2).
- Produces:
  - `ReleaseContext.without_deps: bool = False`, `.deps_report: str | None`,
    `.deps_pr_url: str | None`, `.deps_fix_issue_url: str | None`,
    `.deps_failure: str | None`
  - `ReleaseState.without_deps: bool = False`
  - `update_deps_stage.update_deps(ctx: ReleaseContext) -> None`
  - Release stage order: `… consumer-refresh, update-deps, publish-status`

- [ ] **Step 1: Failing tests for the stage function**

```python
# tests/vergil_tooling/test_release_update_deps_stage.py
from __future__ import annotations

from pathlib import Path

import pytest

from vergil_tooling.lib.release.context import ReleaseContext, ReleaseError
from vergil_tooling.lib.release.update_deps_stage import update_deps
from vergil_tooling.lib.update_deps.context import DepsFailure, UpdateDepsContext
from vergil_tooling.lib.update_deps.report import HeldBack
from vergil_tooling.lib.update_deps.updater import UpdateResult

_MOD = "vergil_tooling.lib.release.update_deps_stage"


def _rctx() -> ReleaseContext:
    return ReleaseContext(repo="o/r", version="2.1.242", repo_root=Path("/r"),
                          version_override=None)


def _deps(**kw) -> UpdateDepsContext:
    d = UpdateDepsContext(repo="o/r", repo_root=Path("/r"), version="2.1.243")
    d.results = [UpdateResult(updater="python", changed=True, summary="s", commit_message="c",
                              held_back=[HeldBack(source="python", name="p", current="1",
                                                  latest="2")])]
    for k, v in kw.items():
        setattr(d, k, v)
    return d


def _patch(monkeypatch, deps: UpdateDepsContext | Exception) -> list[tuple[str, ...]]:
    gits: list[tuple[str, ...]] = []
    monkeypatch.setattr(_MOD + ".os.chdir", lambda p: None)
    monkeypatch.setattr(_MOD + ".git.run", lambda *a: gits.append(a))

    def fake_run_update(*, repo_root):
        if isinstance(deps, Exception):
            raise deps
        return deps

    monkeypatch.setattr(_MOD + ".run_update", fake_run_update)
    return gits


def test_without_deps_skips_everything(monkeypatch) -> None:
    gits = _patch(monkeypatch, AssertionError("must not run"))
    ctx = _rctx()
    ctx.without_deps = True
    update_deps(ctx)
    assert gits == []
    assert ctx.deps_report == "Skipped (--without-deps)."


def test_syncs_root_develop_before_running(monkeypatch) -> None:
    gits = _patch(monkeypatch, _deps(pr_url="https://x/pull/9"))
    update_deps(_rctx())
    assert gits == [("fetch", "origin", "develop"), ("merge", "--ff-only", "origin/develop")]


def test_success_records_report_and_pr(monkeypatch) -> None:
    _patch(monkeypatch, _deps(pr_url="https://x/pull/9", any_changes=True))
    ctx = _rctx()
    update_deps(ctx)
    assert "| python | p | 1 | 2 |" in (ctx.deps_report or "")
    assert ctx.deps_pr_url == "https://x/pull/9"
    assert ctx.deps_failure is None


def test_noop_records_report_without_pr(monkeypatch) -> None:
    _patch(monkeypatch, _deps())
    ctx = _rctx()
    update_deps(ctx)
    assert ctx.deps_pr_url is None
    assert ctx.deps_report is not None


def test_failure_raises_release_error_with_report_and_tail(monkeypatch) -> None:
    d = _deps(pr_url="https://x/pull/9", fix_issue_url="https://x/issues/10", any_changes=True)
    d.failure = DepsFailure(kind="validation", message="Validation failed.",
                            output_tail="E boom", reproduce="vrg-container-run -- vrg-validate")
    _patch(monkeypatch, d)
    ctx = _rctx()
    with pytest.raises(ReleaseError, match="2.1.243") as info:
        update_deps(ctx)
    assert "E boom" in (info.value.detail or "")
    assert "| python | p |" in (info.value.detail or "")
    assert ctx.deps_fix_issue_url == "https://x/issues/10"
    assert ctx.deps_failure is not None and "issues/10" in ctx.deps_failure


def test_sync_failure_propagates(monkeypatch) -> None:
    monkeypatch.setattr(_MOD + ".os.chdir", lambda p: None)

    def boom(*a):
        raise RuntimeError("not a fast-forward")

    monkeypatch.setattr(_MOD + ".git.run", boom)
    with pytest.raises(RuntimeError, match="fast-forward"):
        update_deps(_rctx())
```

- [ ] **Step 2: Run. Expect FAIL (no module)**

Run: `vrg-container-run -- uv run pytest tests/vergil_tooling/test_release_update_deps_stage.py -v`
Expected: FAIL.

- [ ] **Step 3: Add the context fields**

In `src/vergil_tooling/lib/release/context.py`, after `promote`:

```python
    # update-deps stage (epic vergil-project/.github#381, spec §5.1). Skipped
    # with --without-deps. deps_failure is the human summary when the update
    # did not land; publish-status repeats it.
    without_deps: bool = False
    deps_report: str | None = None
    deps_pr_url: str | None = None
    deps_fix_issue_url: str | None = None
    deps_failure: str | None = None
```

- [ ] **Step 4: Write the stage function**

```python
# src/vergil_tooling/lib/release/update_deps_stage.py
"""The vrg-release ``update-deps`` stage (epic vergil-project/.github#381, spec §5.1).

Runs after consumer-refresh, once the release is tagged, promoted and published,
so nothing about the release depends on it (mode fail_defer). Develop is already
on the next patch version, so the update is the first change on it.
"""

from __future__ import annotations

import os
from typing import TYPE_CHECKING

from vergil_tooling.lib import git
from vergil_tooling.lib.release.context import ReleaseError
from vergil_tooling.lib.update_deps import report
from vergil_tooling.lib.update_deps.failure import failure_markdown, failure_summary
from vergil_tooling.lib.update_deps.orchestrator import run_update

if TYPE_CHECKING:
    from vergil_tooling.lib.release.context import ReleaseContext


def update_deps(ctx: ReleaseContext) -> None:
    if ctx.without_deps:
        print("Skipping dependency update (--without-deps).")
        ctx.deps_report = "Skipped (--without-deps)."
        return
    # vrg-update-deps preflight needs the root checkout on a clean develop in sync
    # with origin; the back-merge just moved origin/develop. A failed sync raises
    # and the stage fails (deferred) with nothing updated.
    os.chdir(ctx.repo_root)
    git.run("fetch", "origin", "develop")
    git.run("merge", "--ff-only", "origin/develop")
    deps = run_update(repo_root=ctx.repo_root)
    ctx.deps_report = report.render_markdown(report.collect(deps.results))
    ctx.deps_pr_url = deps.pr_url
    ctx.deps_fix_issue_url = deps.fix_issue_url
    if deps.failure is not None:
        ctx.deps_failure = failure_summary(deps)
        raise ReleaseError(
            phase="update-deps",
            command="vrg-update-deps",
            message=ctx.deps_failure,
            detail=f"{ctx.deps_report}\n\n{failure_markdown(deps.failure)}",
        )
```

- [ ] **Step 5: Run the stage tests. Expect PASS**

Run: `vrg-container-run -- uv run pytest tests/vergil_tooling/test_release_update_deps_stage.py -v`
Expected: PASS.

- [ ] **Step 6: Failing orchestrator tests**

In `tests/vergil_tooling/test_release_orchestrator.py`, insert
`("update-deps", "fail_defer")` before `publish-status` in
`test_build_stages_order_and_modes`, and add:

```python
def test_update_deps_stage_between_consumer_refresh_and_publish_status() -> None:
    names = [s.name for s in build_stages()]
    assert names[-3:] == ["consumer-refresh", "update-deps", "publish-status"]


def test_phase_details_update_deps_with_pr() -> None:
    ctx = _ctx()  # the file's existing ReleaseContext factory
    ctx.deps_report = "REPORT"
    ctx.deps_pr_url = "https://x/pull/9"
    text = _phase_details(ctx, "update-deps")
    assert "REPORT" in text and "https://x/pull/9" in text


def test_phase_details_update_deps_noop() -> None:
    ctx = _ctx()
    ctx.deps_report = "REPORT"
    assert "No dependency changes; no PR opened." in _phase_details(ctx, "update-deps")


def test_publish_status_reports_deps_failure() -> None:
    ctx = _ctx()
    ctx.deps_failure = "Dependency update for 2.1.243 failed (validation)"
    state = ReleaseState(version_override=None, repo_root=Path("/r"), promote=True, ctx=ctx)
    with pytest.raises(ReleaseError, match="2.1.243 failed"):
        _publish_status_stage(state)


def test_publish_status_reports_both_failures() -> None:
    ctx = _ctx()
    ctx.deferred_publish_failures = ["docs"]
    ctx.deps_failure = "Dependency update for 2.1.243 failed (merge)"
    state = ReleaseState(version_override=None, repo_root=Path("/r"), promote=True, ctx=ctx)
    with pytest.raises(ReleaseError) as info:
        _publish_status_stage(state)
    assert "docs" in str(info.value) and "2.1.243 failed" in str(info.value)


def test_preflight_stage_copies_without_deps() -> None:
    state = ReleaseState(version_override=None, repo_root=Path("/tmp/repo"),  # noqa: S108
                         promote=True, without_deps=True)
    ctx = _ctx()
    with patch(_MOD + ".preflight", return_value=ctx):
        _preflight_stage(state)
    assert ctx.without_deps is True
```

`_ctx()` is the file's existing `ReleaseContext` factory. Add `_phase_details`
to the import from `vergil_tooling.lib.release.orchestrator`.

- [ ] **Step 7: Run. Expect FAIL**

Run: `vrg-container-run -- uv run pytest tests/vergil_tooling/test_release_orchestrator.py -v`
Expected: FAIL.

- [ ] **Step 8: Wire the stage**

In `src/vergil_tooling/lib/release/orchestrator.py`:

1. Import `from vergil_tooling.lib.release.update_deps_stage import update_deps`.
2. Add `without_deps: bool = False` to `ReleaseState`, and in
   `_preflight_stage` add `ctx.without_deps = state.without_deps`.
3. Insert the stage between `consumer-refresh` and `publish-status`:

```python
        # First change on the new develop version (epic .github#381, spec §5.1).
        Stage("update-deps", _tracked("update-deps", update_deps), mode="fail_defer"),
```

**4.** In `_phase_details`, before `return`:

```python
    elif phase == "update-deps":
        lines.append("**Held back by declared bounds**")
        lines.append("")
        lines.append(ctx.deps_report or "")
        lines.append("")
        if ctx.deps_pr_url:
            lines.append(f"Dependency-update PR: {ctx.deps_pr_url}")
        elif not ctx.without_deps:
            lines.append("No dependency changes; no PR opened.")
```

**5.** Replace `_publish_status_stage` so it reports either failure or both:

```python
def _publish_status_stage(state: ReleaseState) -> None:
    """Terminal fail_defer: surface deferred publish and dependency-update failures as exit 1.

    (keep the existing docstring body)
    """
    ctx = state.ctx
    if ctx is None:
        return
    problems: list[str] = []
    if ctx.deferred_publish_failures:
        jobs = ", ".join(ctx.deferred_publish_failures)
        problems.append(
            f"Release v{ctx.version} is published (tag + GitHub Release), but these "
            f"CD publish jobs did not succeed: {jobs}. Re-run the CD publish to "
            "deliver the artifacts — do NOT use vrg-release --resume (the version "
            "is already tagged)."
        )
    if ctx.deps_failure:
        problems.append(ctx.deps_failure)
    if problems:
        raise ReleaseError(phase="publish-status", command="publish-status",
                           message=" ".join(problems))
```

Check the existing publish-status tests still pass. The publish-only message is
unchanged.

- [ ] **Step 9: `--without-deps` on the CLI**

In `src/vergil_tooling/bin/vrg_release.py` `parse_args`, after `--no-promote`:

```python
    parser.add_argument(
        "--without-deps",
        action="store_true",
        default=False,
        help="Skip the post-release dependency update (the update-deps stage).",
    )
```

And pass `without_deps=args.without_deps` into `ReleaseState(...)`. In
`tests/vergil_tooling/test_vrg_release.py`, add a test that `main([...,
"--without-deps"])` builds a `ReleaseState` with `without_deps=True`, using the
same patching the file already uses for `--no-promote`.

- [ ] **Step 10: Run the release tests. Expect PASS**

Run: `vrg-container-run -- uv run pytest tests/vergil_tooling -k "release" -v`
Expected: PASS.

- [ ] **Step 11: Full validation, then commit**

Run: `vrg-container-run -- vrg-validate`. Expected: PASS with 100% coverage.

```bash
vrg-git add src/vergil_tooling/lib/release src/vergil_tooling/bin/vrg_release.py tests/vergil_tooling
vrg-commit --type feat --scope release \
  --message "run the dependency update as the last vrg-release stage"
```

---

### Task 4: Release gate on open `dependency-update` PRs and `--allow-pending-deps` [`vergil-tooling`]

Covers spec §5.3.

**Files:**

- Modify: `src/vergil_tooling/lib/release/preflight.py`
- Modify: `src/vergil_tooling/lib/release/context.py` (`bypassed_deps_prs`)
- Modify: `src/vergil_tooling/lib/release/orchestrator.py` (`ReleaseState.allow_pending_deps`, prepare details)
- Modify: `src/vergil_tooling/bin/vrg_release.py` (`--allow-pending-deps`)
- Test: `tests/vergil_tooling/test_release_preflight.py`
- Test: `tests/vergil_tooling/test_release_orchestrator.py`
- Test: `tests/vergil_tooling/test_vrg_release.py`

**Interfaces:**

- Consumes: `context.DEPS_LABEL`, `pr.FIX_TASK_PREFIX` (Task 2);
  `github.read_json(*args)`.
- Produces:
  - `preflight.PendingDepsPr(url: str, fix_task: str | None)` (frozen dataclass)
  - `preflight.pending_deps_prs(repo: str) -> list[PendingDepsPr]`
  - `preflight.preflight(..., allow_pending_deps: bool = False)`
  - `ReleaseContext.bypassed_deps_prs: list[str]`
  - `ReleaseState.allow_pending_deps: bool = False`

- [ ] **Step 1: Failing gate tests**

Add to `tests/vergil_tooling/test_release_preflight.py`:

```python
from vergil_tooling.lib.release.preflight import PendingDepsPr, pending_deps_prs

_PRS = [
    {"url": "https://github.com/o/r/pull/9",
     "body": "…\n\nFix task: https://github.com/o/r/issues/10\n"},
    {"url": "https://github.com/o/r/pull/11", "body": "no fix task yet"},
]


def test_pending_deps_prs_parses_fix_task() -> None:
    with patch(_MOD + ".github.read_json", return_value=_PRS) as m:
        prs = pending_deps_prs("o/r")
    assert prs == [
        PendingDepsPr(url="https://github.com/o/r/pull/9", fix_task="https://github.com/o/r/issues/10"),
        PendingDepsPr(url="https://github.com/o/r/pull/11", fix_task=None),
    ]
    args = m.call_args.args
    assert ("--label", "dependency-update") == args[args.index("--label"):args.index("--label") + 2]
    assert "open" in args


def _fresh_patches(pending):
    root_wt = Path("/tmp/repo/.worktrees/release-2.1.0")  # noqa: S108
    return (
        patch(_MOD + "._check_host_prerequisites"),
        patch(_MOD + ".check_gh_auth", return_value="o/r"),
        patch(_MOD + "._read_and_validate_config"),
        patch(_MOD + "._check_branch_and_tree"),
        patch(_MOD + ".version.show", return_value="2.1.0"),
        patch(_MOD + "._check_version_not_tagged"),
        patch(_MOD + "._check_no_existing_tracking_issue"),
        patch(_MOD + "._acquire_release_worktree", return_value=root_wt),
        patch(_MOD + ".os.chdir"),
        patch(_MOD + ".pending_deps_prs", return_value=pending),
    )


def test_preflight_refuses_on_pending_deps_pr() -> None:
    pending = [PendingDepsPr(url="https://github.com/o/r/pull/9",
                             fix_task="https://github.com/o/r/issues/10")]
    with contextlib.ExitStack() as stack:
        for p in _fresh_patches(pending):
            stack.enter_context(p)
        with pytest.raises(ReleaseError, match="dependency-update") as info:
            preflight(version_override=None, repo_root=Path("/tmp/repo"))  # noqa: S108
    msg = str(info.value)
    assert "pull/9" in msg and "issues/10" in msg
    assert "--allow-pending-deps" in msg


def test_preflight_allow_pending_deps_records_bypass() -> None:
    pending = [PendingDepsPr(url="https://github.com/o/r/pull/9", fix_task=None)]
    with contextlib.ExitStack() as stack:
        for p in _fresh_patches(pending):
            stack.enter_context(p)
        ctx = preflight(version_override=None, repo_root=Path("/tmp/repo"),  # noqa: S108
                        allow_pending_deps=True)
    assert ctx.bypassed_deps_prs == ["https://github.com/o/r/pull/9"]


def test_preflight_no_pending_deps_passes() -> None:
    with contextlib.ExitStack() as stack:
        for p in _fresh_patches([]):
            stack.enter_context(p)
        ctx = preflight(version_override=None, repo_root=Path("/tmp/repo"))  # noqa: S108
    assert ctx.bypassed_deps_prs == []


def test_preflight_resume_skips_pending_deps_gate() -> None:
    wt = Path("/tmp/repo/.worktrees/release-2.1.0")  # noqa: S108
    with (
        patch(_MOD + "._check_host_prerequisites"),
        patch(_MOD + ".check_gh_auth", return_value="o/r"),
        patch(_MOD + "._read_and_validate_config"),
        patch(_MOD + "._check_branch_and_tree"),
        patch(_MOD + "._acquire_release_worktree", return_value=wt),
        patch(_MOD + ".os.chdir"),
        patch(_MOD + ".pending_deps_prs") as m_pending,
    ):
        preflight(version_override=None, repo_root=Path("/tmp/repo"),  # noqa: S108
                  resume=True, resume_version="2.1.0", resume_issue_number=7)
    m_pending.assert_not_called()
```

Add `import contextlib` at the top. Existing fresh-start tests in this file now
reach the gate, so add `patch(_MOD + ".pending_deps_prs", return_value=[])` to
each of their `with` blocks.

- [ ] **Step 2: Run. Expect FAIL**

Run: `vrg-container-run -- uv run pytest tests/vergil_tooling/test_release_preflight.py -v`
Expected: FAIL.

- [ ] **Step 3: Implement the gate**

In `src/vergil_tooling/lib/release/context.py`, after `deps_failure`:

```python
    # Open dependency-update PRs this release bypassed with --allow-pending-deps
    # (spec §5.3); named in the prepare-stage tracking comment.
    bypassed_deps_prs: list[str] = field(default_factory=list)
```

In `src/vergil_tooling/lib/release/preflight.py`:

```python
import re
from dataclasses import dataclass

from vergil_tooling.lib.update_deps.context import DEPS_LABEL
from vergil_tooling.lib.update_deps.pr import FIX_TASK_PREFIX

_FIX_TASK_RE = re.compile(rf"^{re.escape(FIX_TASK_PREFIX)}(\S+)\s*$", re.MULTILINE)


@dataclass(frozen=True)
class PendingDepsPr:
    url: str
    fix_task: str | None


def pending_deps_prs(repo: str) -> list[PendingDepsPr]:
    """Open PRs labelled ``dependency-update`` in *repo*, with their fix task if linked."""
    raw = github.read_json(
        "pr", "list", "--repo", repo, "--state", "open", "--label", DEPS_LABEL,
        "--json", "url,body", "--limit", "100",
    )
    if not isinstance(raw, list):
        msg = f"gh pr list returned {type(raw).__name__}, expected a list"
        raise ReleaseError(phase="preflight", command="gh pr list", message=msg)
    found: list[PendingDepsPr] = []
    for item in raw:
        if not isinstance(item, dict):
            msg = "gh pr list returned a non-object entry"
            raise ReleaseError(phase="preflight", command="gh pr list", message=msg)
        match = _FIX_TASK_RE.search(str(item.get("body") or ""))
        found.append(PendingDepsPr(url=str(item["url"]), fix_task=match.group(1) if match else None))
    return found


def _check_pending_deps(repo: str, *, allow: bool) -> list[str]:
    """Refuse while a dependency-update PR is open, unless *allow* (spec §5.3)."""
    pending = pending_deps_prs(repo)
    if not pending:
        return []
    if allow:
        return [p.url for p in pending]
    described = "; ".join(
        p.url + (f" (fix task: {p.fix_task})" if p.fix_task else "") for p in pending
    )
    raise ReleaseError(
        phase="preflight",
        command=f"gh pr list --label {DEPS_LABEL}",
        message=(
            f"Open dependency-update PR(s) block this release: {described}. Clear it by "
            "landing the fix task, merging the PR, or closing it deliberately. For an "
            "urgent release, pass --allow-pending-deps (the bypass is logged on the "
            "tracking issue)."
        ),
    )
```

In `preflight(...)`, add the keyword `allow_pending_deps: bool = False`. Inside
the existing `if not resume:` block, after `_check_no_existing_tracking_issue`,
add `bypassed = _check_pending_deps(repo, allow=allow_pending_deps)`, with
`bypassed: list[str] = []` initialized before the block. After building `ctx`,
set `ctx.bypassed_deps_prs = bypassed`.

Check for an import cycle: `lib/update_deps/preflight.py` imports
`release.preflight.check_gh_auth`, and `release.preflight` now imports
`update_deps.context` and `update_deps.pr`. Neither of those imports
`release.preflight`, so there is no cycle. If ruff or mypy reports one, move
`FIX_TASK_PREFIX` into `update_deps/context.py` and import it from there in
both places.

- [ ] **Step 4: Run the preflight tests. Expect PASS**

Run: `vrg-container-run -- uv run pytest tests/vergil_tooling/test_release_preflight.py -v`
Expected: PASS.

- [ ] **Step 5: Failing tests for the bypass comment and CLI flag**

In `tests/vergil_tooling/test_release_orchestrator.py`:

```python
def test_phase_details_prepare_names_bypassed_deps_prs() -> None:
    ctx = _ctx()
    ctx.bypassed_deps_prs = ["https://github.com/o/r/pull/9"]
    text = _phase_details(ctx, "prepare")
    assert "--allow-pending-deps" in text
    assert "https://github.com/o/r/pull/9" in text


def test_preflight_stage_passes_allow_pending_deps() -> None:
    state = ReleaseState(version_override=None, repo_root=Path("/tmp/repo"),  # noqa: S108
                         promote=True, allow_pending_deps=True)
    with patch(_MOD + ".preflight", return_value=_ctx()) as m_preflight:
        _preflight_stage(state)
    assert m_preflight.call_args.kwargs["allow_pending_deps"] is True
```

The existing `test_preflight_stage_populates_ctx` asserts the exact `preflight`
keyword set, so add `allow_pending_deps=False` to its
`assert_called_once_with`. In
`tests/vergil_tooling/test_vrg_release.py`, add a test that
`--allow-pending-deps` reaches `ReleaseState(allow_pending_deps=True)`.

- [ ] **Step 6: Run. Expect FAIL**

Run: `vrg-container-run -- uv run pytest tests/vergil_tooling/test_release_orchestrator.py tests/vergil_tooling/test_vrg_release.py -v`
Expected: FAIL.

- [ ] **Step 7: Wire state, details and CLI**

`ReleaseState`: add `allow_pending_deps: bool = False`. In `_preflight_stage`,
pass `allow_pending_deps=state.allow_pending_deps` to `preflight(...)`. In
`_phase_details`, inside the `prepare` branch:

```python
        if ctx.bypassed_deps_prs:
            prs = ", ".join(ctx.bypassed_deps_prs)
            lines.append(
                f"**Bypassed pending dependency-update PR(s)** with "
                f"`--allow-pending-deps`: {prs}"
            )
```

In `vrg_release.py`:

```python
    parser.add_argument(
        "--allow-pending-deps",
        action="store_true",
        default=False,
        help=(
            "Release even though a dependency-update PR is open (urgent fixes "
            "only); the bypass is logged on the tracking issue."
        ),
    )
```

Pass `allow_pending_deps=args.allow_pending_deps` into `ReleaseState`.

- [ ] **Step 8: Run the release tests. Expect PASS**

Run: `vrg-container-run -- uv run pytest tests/vergil_tooling -k release -v`
Expected: PASS.

- [ ] **Step 9: Full validation, then commit**

Run: `vrg-container-run -- vrg-validate`. Expected: PASS with 100% coverage.

```bash
vrg-git add src/vergil_tooling/lib/release src/vergil_tooling/bin/vrg_release.py tests/vergil_tooling
vrg-commit --type feat --scope release \
  --message "gate vrg-release on open dependency-update PRs, with --allow-pending-deps"
```

---

### Task 5: Python held-back report [`vergil-tooling`]

Covers spec §7.3 (the `python` row) and §8.1.

**Files:**

- Create: `src/vergil_tooling/lib/update_deps/pypi.py`
- Modify: `src/vergil_tooling/lib/update_deps/updaters/python_uv.py`
- Modify: `pyproject.toml` (add `packaging>=24.0` to `[project] dependencies`), then `uv.lock`
- Test: `tests/vergil_tooling/test_update_deps_pypi.py` (new)
- Test: `tests/vergil_tooling/test_update_deps_python_uv.py`

**Interfaces:**

- Consumes: `report.HeldBack` (Task 1).
- Produces:
  - `pypi.Fetch = Callable[[str], bytes]`
  - `pypi.latest_version(name: str, *, fetch: Fetch = _fetch) -> str` (raises `UpdateDepsError`)
  - `python_uv.direct_requirements(pyproject: dict) -> list[tuple[Requirement, str]]`
    (each requirement plus the table it came from)
  - `python_uv.locked_versions(lock: dict) -> dict[str, str]` (canonical name to version)
  - `python_uv.held_back(pyproject: dict, lock: dict, *, latest: Callable[[str], str]) -> list[HeldBack]`
  - `PythonUvUpdater(latest: Callable[[str], str] = pypi.latest_version)`

- [ ] **Step 1: Add the dependency**

Edit `pyproject.toml` `[project] dependencies` to
`["rich>=13.0", "pyyaml>=6.0", "pyelftools>=0.31", "packaging>=24.0"]`, then
run `vrg-container-run -- uv lock` so `uv.lock` records it.

- [ ] **Step 2: Failing PyPI tests**

```python
# tests/vergil_tooling/test_update_deps_pypi.py
from __future__ import annotations

import json
import urllib.error

import pytest

from vergil_tooling.lib.update_deps import pypi
from vergil_tooling.lib.update_deps.context import UpdateDepsError


def test_latest_version_reads_info_version() -> None:
    urls: list[str] = []

    def fetch(url: str) -> bytes:
        urls.append(url)
        return json.dumps({"info": {"version": "2.1.0"}}).encode()

    assert pypi.latest_version("Example_Pkg", fetch=fetch) == "2.1.0"
    assert urls == ["https://pypi.org/pypi/example-pkg/json"]


def test_latest_version_fails_loud_on_http_error() -> None:
    def fetch(url: str) -> bytes:
        raise urllib.error.HTTPError(url, 404, "Not Found", None, None)  # type: ignore[arg-type]

    with pytest.raises(UpdateDepsError, match="example-pkg"):
        pypi.latest_version("example-pkg", fetch=fetch)


def test_latest_version_fails_loud_on_bad_payload() -> None:
    with pytest.raises(UpdateDepsError, match="no info.version"):
        pypi.latest_version("p", fetch=lambda url: b"{}")
```

- [ ] **Step 3: Run. Expect FAIL**

Run: `vrg-container-run -- uv run pytest tests/vergil_tooling/test_update_deps_pypi.py -v`
Expected: FAIL.

- [ ] **Step 4: Write `pypi.py`**

```python
# src/vergil_tooling/lib/update_deps/pypi.py
"""Latest-release lookups on PyPI for the python held-back report (spec §8.1)."""

from __future__ import annotations

import json
import urllib.error
import urllib.request
from collections.abc import Callable

from packaging.utils import canonicalize_name

from vergil_tooling.lib.update_deps.context import UpdateDepsError

Fetch = Callable[[str], bytes]


def _fetch(url: str) -> bytes:
    with urllib.request.urlopen(url, timeout=30) as resp:  # noqa: S310 - fixed https base URL
        data: bytes = resp.read()
        return data


def latest_version(name: str, *, fetch: Fetch = _fetch) -> str:
    """The latest release of *name* on PyPI (``info.version``). Fails loud on any error."""
    url = f"https://pypi.org/pypi/{canonicalize_name(name)}/json"
    try:
        payload = json.loads(fetch(url))
    except (urllib.error.URLError, OSError, ValueError) as exc:
        raise UpdateDepsError(phase="update", command=f"GET {url}",
                              message=f"PyPI lookup failed for {canonicalize_name(name)}: {exc}") from exc
    version = payload.get("info", {}).get("version") if isinstance(payload, dict) else None
    if not isinstance(version, str) or not version:
        raise UpdateDepsError(phase="update", command=f"GET {url}",
                              message=f"PyPI response for {name} has no info.version.")
    return version
```

(`urllib.error.HTTPError` is a subclass of `URLError`.)

- [ ] **Step 5: Run the PyPI tests. Expect PASS**

Run: `vrg-container-run -- uv run pytest tests/vergil_tooling/test_update_deps_pypi.py -v`
Expected: PASS.

- [ ] **Step 6: Failing held-back tests**

Add to `tests/vergil_tooling/test_update_deps_python_uv.py`:

```python
from vergil_tooling.lib.update_deps.report import HeldBack
from vergil_tooling.lib.update_deps.updaters import python_uv

_PYPROJECT = {
    "build-system": {"requires": ["setuptools>=68", "wheel"]},
    "project": {"dependencies": ["rich>=13.0", "pyyaml>=6.0,<7",
                                 'foo[bar]>=1,<2; python_version < "3.13"']},
    "dependency-groups": {"dev": ["pytest<9", {"include-group": "lint"}],
                          "lint": ["ruff"]},
    "tool": {"uv": {"constraint-dependencies": ["msgpack>=1.2.1,<1.3"]}},
}
_LOCK = {"package": [
    {"name": "rich", "version": "14.0.0"}, {"name": "pyyaml", "version": "6.0.2"},
    {"name": "foo", "version": "1.4.0"}, {"name": "pytest", "version": "8.4.1"},
    {"name": "ruff", "version": "0.12.0"}, {"name": "msgpack", "version": "1.2.3"},
    {"name": "transitive-only", "version": "1.0.0"},
]}
_LATEST = {"rich": "14.0.0", "pyyaml": "7.0.0", "foo": "2.0.0", "pytest": "9.0.1",
           "ruff": "0.13.0", "msgpack": "1.3.0", "setuptools": "80.0.0", "wheel": "0.45.0"}


def test_direct_requirements_shapes() -> None:
    reqs = python_uv.direct_requirements({
        "project": {"dependencies": ['foo[bar]>=1; python_version < "3.13"',
                                     "local @ https://example.invalid/local.whl"]},
        "dependency-groups": {"dev": [{"include-group": "x"}, "pytest"]},
    })
    names = [r.name for r, _ in reqs]
    assert names == ["foo", "pytest"]  # URL requirement and include-group skipped


def test_locked_versions_canonicalizes() -> None:
    assert python_uv.locked_versions({"package": [{"name": "PyYAML", "version": "6.0.2"}]}) == {
        "pyyaml": "6.0.2"}


def test_held_back_reports_only_direct_deps_outside_bounds() -> None:
    entries = python_uv.held_back(_PYPROJECT, _LOCK, latest=lambda n: _LATEST[n])
    assert entries == [
        HeldBack(source="python", name="pyyaml", current="6.0.2", latest="7.0.0",
                 note="project.dependencies: pyyaml<7,>=6.0"),
        HeldBack(source="python", name="foo", current="1.4.0", latest="2.0.0",
                 note='project.dependencies: foo[bar]<2,>=1; python_version < "3.13"'),
        HeldBack(source="python", name="pytest", current="8.4.1", latest="9.0.1",
                 note="dependency-groups.dev: pytest<9"),
        HeldBack(source="python", name="msgpack", current="1.2.3", latest="1.3.0",
                 note="tool.uv.constraint-dependencies: msgpack<1.3,>=1.2.1"),
    ]
    assert all(e.name != "transitive-only" for e in entries)


def test_held_back_unlocked_build_requirement() -> None:
    entries = python_uv.held_back(
        {"build-system": {"requires": ["setuptools<70"]}}, {"package": []},
        latest=lambda n: "80.0.0")
    assert entries == [HeldBack(source="python", name="setuptools", current="(not locked)",
                                latest="80.0.0", note="build-system.requires: setuptools<70")]


def test_held_back_dedupes_same_requirement() -> None:
    pp = {"project": {"dependencies": ["a<2"]}, "dependency-groups": {"dev": ["a<2"]}}
    entries = python_uv.held_back(pp, {"package": [{"name": "a", "version": "1"}]},
                                  latest=lambda n: "2")
    assert len(entries) == 1


def test_apply_attaches_held_back(monkeypatch, tmp_path) -> None:
    (tmp_path / "pyproject.toml").write_text('[project]\nname="x"\ndependencies=["a<2"]\n')
    (tmp_path / "uv.lock").write_text('[[package]]\nname = "a"\nversion = "1.0"\n')
    monkeypatch.setattr(python_uv.progress, "run", lambda cmd: 0)
    monkeypatch.setattr(python_uv.git, "read_output", lambda *a: "")
    ctx = UpdateDepsContext(repo="o/r", repo_root=tmp_path, worktree_path=tmp_path)
    result = python_uv.PythonUvUpdater(latest=lambda n: "2.0").apply(ctx)
    assert [e.name for e in result.held_back] == ["a"]
```

(Ensure `UpdateDepsContext` is imported in the test file.)

The expected `note` strings use `str(Requirement)`, which `packaging`
normalizes. If the installed `packaging` orders specifiers differently, use the
string `str(Requirement(...))` produces and keep the assertion exact.

- [ ] **Step 7: Run. Expect FAIL**

Run: `vrg-container-run -- uv run pytest tests/vergil_tooling/test_update_deps_python_uv.py -v`
Expected: FAIL.

- [ ] **Step 8: Implement the held-back logic in `python_uv.py`**

```python
"""Python language-library updater: ``uv lock --upgrade`` plus the held-back report."""

from __future__ import annotations

import tomllib
from typing import TYPE_CHECKING, Any

from packaging.requirements import Requirement
from packaging.utils import canonicalize_name
from packaging.version import Version

from vergil_tooling.lib import git, progress
from vergil_tooling.lib.update_deps import pypi
from vergil_tooling.lib.update_deps.report import HeldBack
from vergil_tooling.lib.update_deps.updater import UpdateResult

if TYPE_CHECKING:
    from collections.abc import Callable
    from pathlib import Path

    from vergil_tooling.lib.update_deps.context import UpdateDepsContext

_COMMIT_MESSAGE = "chore(deps): uv lock --upgrade"


def _strings(values: Any) -> list[str]:
    # Dependency groups may hold {include-group = "..."} tables; only strings are requirements.
    return [v for v in (values or []) if isinstance(v, str)]


def direct_requirements(pyproject: dict[str, Any]) -> list[tuple[Requirement, str]]:
    """Directly declared requirements (spec §8.1), each with the table it came from.

    URL requirements are skipped: they have no index release to compare against.
    """
    sources: list[tuple[str, list[str]]] = [
        ("project.dependencies", _strings(pyproject.get("project", {}).get("dependencies"))),
    ]
    for group, values in (pyproject.get("dependency-groups") or {}).items():
        sources.append((f"dependency-groups.{group}", _strings(values)))
    sources.append(("build-system.requires",
                    _strings(pyproject.get("build-system", {}).get("requires"))))
    sources.append(("tool.uv.constraint-dependencies",
                    _strings(pyproject.get("tool", {}).get("uv", {}).get("constraint-dependencies"))))
    found: list[tuple[Requirement, str]] = []
    for where, values in sources:
        for text in values:
            req = Requirement(text)
            if req.url is None:
                found.append((req, where))
    return found


def locked_versions(lock: dict[str, Any]) -> dict[str, str]:
    return {canonicalize_name(p["name"]): p["version"] for p in lock.get("package", [])
            if "version" in p}


def held_back(
    pyproject: dict[str, Any], lock: dict[str, Any], *, latest: Callable[[str], str]
) -> list[HeldBack]:
    """Direct requirements whose latest release is outside their own specifier."""
    locked = locked_versions(lock)
    entries: list[HeldBack] = []
    seen: set[tuple[str, str]] = set()
    for req, where in direct_requirements(pyproject):
        name = canonicalize_name(req.name)
        if not req.specifier or (name, str(req.specifier)) in seen:
            continue
        seen.add((name, str(req.specifier)))
        newest = latest(name)
        if req.specifier.contains(Version(newest), prereleases=True):
            continue
        entries.append(HeldBack(source="python", name=name,
                                current=locked.get(name, "(not locked)"), latest=newest,
                                note=f"{where}: {req}"))
    return entries


def _base(ctx: UpdateDepsContext) -> Path:
    return ctx.worktree_path if ctx.worktree_path is not None else ctx.repo_root


class PythonUvUpdater:
    """Upgrade Python dependencies within their declared constraints."""

    name = "python"

    def __init__(self, latest: Callable[[str], str] = pypi.latest_version) -> None:
        self._latest = latest

    def applies(self, ctx: UpdateDepsContext) -> bool:
        root = ctx.repo_root
        return (root / "pyproject.toml").is_file() and (root / "uv.lock").is_file()

    def apply(self, ctx: UpdateDepsContext) -> UpdateResult:
        progress.run(["vrg-container-run", "--", "uv", "lock", "--upgrade"])
        dirty = bool(git.read_output("status", "--porcelain", "uv.lock").strip())
        base = _base(ctx)
        pyproject = tomllib.loads((base / "pyproject.toml").read_text())
        lock = tomllib.loads((base / "uv.lock").read_text())
        return UpdateResult(
            updater=self.name,
            changed=dirty,
            summary="uv lock --upgrade" if dirty else "uv lock --upgrade (no changes)",
            commit_message=_COMMIT_MESSAGE,
            held_back=held_back(pyproject, lock, latest=self._latest),
        )
```

`applies` keeps checking `ctx.repo_root`, which is the existing behaviour, so
the existing tests keep passing. `DEFAULT_REGISTRY` constructs
`PythonUvUpdater()` with the real PyPI lookup.

- [ ] **Step 9: Run. Expect PASS**

Run: `vrg-container-run -- uv run pytest tests/vergil_tooling/test_update_deps_python_uv.py tests/vergil_tooling/test_update_deps_pypi.py -v`
Expected: PASS.

- [ ] **Step 10: Full validation, then commit**

Run: `vrg-container-run -- vrg-validate`. Expected: PASS. `pip-audit` and
`pip-licenses` accept `packaging`, which is Apache-2.0/BSD.

```bash
vrg-git add pyproject.toml uv.lock src/vergil_tooling/lib/update_deps tests/vergil_tooling
vrg-commit --type feat --scope update-deps \
  --message "report direct Python dependencies held back by their declared bounds"
```

---

### Task 6: `vergil` updater: normalize `action.yml` refs and bump the vergil-python runtime from the index [`vergil-tooling`]

Covers spec §7.3 (the `vergil` row) and §8.2.

**Files:**

- Create: `src/vergil_tooling/lib/update_deps/vergil_index.py`
- Modify: `src/vergil_tooling/lib/update_deps/updaters/vergil_eco.py`
- Modify: `src/vergil_tooling/lib/release/package_index.py` (public `deb_package_names`, `rpm_package_names`)
- Modify: `src/vergil_tooling/lib/package/python_builder.py` (public `SHARED_SUITE`)
- Test: `tests/vergil_tooling/test_update_deps_vergil_index.py` (new)
- Test: `tests/vergil_tooling/test_update_deps_vergil_eco.py`
- Test: `tests/vergil_tooling/test_package_index.py` (or the existing `package_index` test file; grep `deb_has`)

**Interfaces:**

- Consumes: `report.HeldBack` (Task 1); `package_index._fetch`,
  `orgs.for_vendor("vergil").base_url`, `matrix.select_targets(pkg)` (each
  `Target` has `.fmt`, `.suite`, `.arch`, `.rpm_arch`), `config.read_config`.
- Produces:
  - `package_index.deb_package_names(base_url: str, suite: str, arch: str, fetch: Fetch) -> set[str]`
  - `package_index.rpm_package_names(base_url: str, el: str, rpm_arch: str, fetch: Fetch) -> set[str]`
  - `python_builder.SHARED_SUITE: str`
  - `vergil_index.runtime_indexes(pkg: PackageConfig) -> list[tuple[str, str, str]]`
    (the `(fmt, suite, arch)` indexes the build and install-test resolve
    `vergil-python<runtime>` from: the shared deb suite for both arches, plus
    every selected target)
  - `vergil_index.served_runtimes(pkg: PackageConfig, *, fetch: Fetch = package_index._fetch) -> set[str]`
    (runtimes served by **every** index in `runtime_indexes`; raises
    `UpdateDepsError` when empty)
  - `vergil_index.served_runtimes_for(base: Path) -> set[str]` (reads `[package]` from `base/vergil.toml`)
  - `vergil_index.newest_patch(current: str, served: set[str]) -> str | None`
  - `vergil_index.newer_minor(current: str, served: set[str]) -> str | None`
  - `vergil_eco.normalize_refs(base, target)` now also rewrites every `action.yml`/`action.yaml`
  - `vergil_eco.bump_runtime(base: Path, served: set[str]) -> tuple[str | None, HeldBack | None]`
    (the new runtime if bumped, and the newer-minor entry)
  - `VergilUpdater(served_runtimes: Callable[[Path], set[str]] = vergil_index.served_runtimes_for)`

**Why every index (spec §8.2, alignment decision):** non-native cells install
the runtime from the shared deb suite (`python_builder.py:135-138`). A native
cell installs it from its own target's suite, which may be rpm. Install-test
then installs the product on every selected target, each of which resolves
`Depends: vergil-python<runtime>` from its own index. A runtime missing from any
one of these would merge green and fail the next release. The rpm package name
is assumed to match the deb name (`vergil-python<runtime>`, set by
vergil-python's `[package].name`). Before relying on that, confirm it once by
running `rpm_package_names` against a live rpm index in a REPL.

- [ ] **Step 1: Failing index tests**

```python
# tests/vergil_tooling/test_update_deps_vergil_index.py
from __future__ import annotations

import gzip
from pathlib import Path

import pytest

from vergil_tooling.lib.config import PackageConfig, PackagePythonConfig
from vergil_tooling.lib.package import matrix, python_builder
from vergil_tooling.lib.update_deps import vergil_index
from vergil_tooling.lib.update_deps.context import UpdateDepsError

_MOD = "vergil_tooling.lib.update_deps.vergil_index"
_PKG = PackageConfig(builder="python", vendor="vergil", summary="s", smoke="x",
                     python=PackagePythonConfig(runtime="3.14.8"))


def _deb(*names: str) -> bytes:
    return "\n\n".join(f"Package: {n}\nVersion: 1\nArchitecture: all" for n in names).encode()


def _rpm(*names: str) -> tuple[bytes, bytes]:
    repomd = (b'<repomd xmlns="http://linux.duke.edu/metadata/repo"><data type="primary">'
              b'<location href="repodata/primary.xml.gz"/></data></repomd>')
    primary = ('<metadata xmlns="http://linux.duke.edu/metadata/common">'
               + "".join(f'<package><name>{n}</name><version ver="1" rel="1"/></package>'
                         for n in names)
               + "</metadata>")
    return repomd, gzip.compress(primary.encode())


def _fetch(deb_names: tuple[str, ...], rpm_names: tuple[str, ...]):
    repomd, primary = _rpm(*rpm_names)

    def fetch(url: str) -> bytes:
        if url.endswith("/Packages"):
            return _deb(*deb_names)
        if url.endswith("repomd.xml"):
            return repomd
        if url.endswith("primary.xml.gz"):
            return primary
        raise AssertionError(url)

    return fetch


def test_runtime_indexes_cover_shared_suite_and_every_target() -> None:
    indexes = vergil_index.runtime_indexes(_PKG)
    assert ("deb", python_builder.SHARED_SUITE, "amd64") in indexes
    assert ("deb", python_builder.SHARED_SUITE, "arm64") in indexes
    for t in matrix.select_targets(_PKG):
        expected = (t.fmt, t.suite, t.arch if t.fmt == "deb" else t.rpm_arch)
        assert expected in indexes
    assert len(indexes) == len(set(indexes))


def test_served_runtimes_requires_every_index(monkeypatch) -> None:
    monkeypatch.setattr(_MOD + ".runtime_indexes",
                        lambda pkg: [("deb", "noble", "amd64"), ("rpm", "el9", "x86_64")])
    fetch = _fetch(("vergil-python3.14.8", "vergil-python3.14.9", "vergil-tooling"),
                   ("vergil-python3.14.8",))  # rpm not yet published for 3.14.9
    assert vergil_index.served_runtimes(_PKG, fetch=fetch) == {"3.14.8"}


def test_served_runtimes_empty_fails_loud(monkeypatch) -> None:
    monkeypatch.setattr(_MOD + ".runtime_indexes", lambda pkg: [("deb", "noble", "amd64")])
    with pytest.raises(UpdateDepsError, match="no vergil-python runtime"):
        vergil_index.served_runtimes(_PKG, fetch=lambda url: b"")


def test_served_runtimes_for_reads_package(monkeypatch) -> None:
    class _Cfg:
        package = _PKG

    monkeypatch.setattr(_MOD + ".config.read_config", lambda base: _Cfg())
    monkeypatch.setattr(_MOD + ".served_runtimes", lambda pkg: {"3.14.9"})
    assert vergil_index.served_runtimes_for(Path("/r")) == {"3.14.9"}


def test_served_runtimes_for_without_package_fails_loud(monkeypatch) -> None:
    class _Cfg:
        package = None

    monkeypatch.setattr(_MOD + ".config.read_config", lambda base: _Cfg())
    with pytest.raises(UpdateDepsError, match=r"\[package\]"):
        vergil_index.served_runtimes_for(Path("/r"))


@pytest.mark.parametrize(
    ("current", "served", "expected"),
    [
        ("3.14.8", {"3.14.8", "3.14.9", "3.14.10", "3.15.1"}, "3.14.10"),
        ("3.14.10", {"3.14.8", "3.14.10"}, None),
        ("3.14.8", {"3.15.0"}, None),
    ],
)
def test_newest_patch(current, served, expected) -> None:
    assert vergil_index.newest_patch(current, served) == expected


def test_newer_minor() -> None:
    assert vergil_index.newer_minor("3.14.8", {"3.14.9", "3.15.0", "3.15.2"}) == "3.15.2"
    assert vergil_index.newer_minor("3.14.8", {"3.14.9"}) is None
```

Add the listing tests to the package-index test file. Reuse the `_rpm` helper
shape above, or import it from a shared test helper:

```python
def test_deb_package_names_lists_every_package() -> None:
    text = b"Package: a\nVersion: 1\n\nPackage: b\nVersion: 2\n"
    assert package_index.deb_package_names("https://x", "noble", "amd64",
                                           lambda url: text) == {"a", "b"}


def test_rpm_package_names_lists_every_package() -> None:
    repomd = (b'<repomd xmlns="http://linux.duke.edu/metadata/repo"><data type="primary">'
              b'<location href="repodata/primary.xml.gz"/></data></repomd>')
    primary = gzip.compress(
        b'<metadata xmlns="http://linux.duke.edu/metadata/common">'
        b'<package><name>a</name><version ver="1" rel="1"/></package>'
        b'<package><name>b</name><version ver="2" rel="1"/></package></metadata>')

    def fetch(url: str) -> bytes:
        return repomd if url.endswith("repomd.xml") else primary

    assert package_index.rpm_package_names("https://x", "el9", "x86_64", fetch) == {"a", "b"}


def test_rpm_package_names_unpublished_repo_is_empty() -> None:
    assert package_index.rpm_package_names("https://x", "el9", "x86_64", lambda url: b"") == set()
```

- [ ] **Step 2: Run. Expect FAIL**

Run: `vrg-container-run -- uv run pytest tests/vergil_tooling/test_update_deps_vergil_index.py -v`
Expected: FAIL.

- [ ] **Step 3: Expose the index helpers**

In `src/vergil_tooling/lib/release/package_index.py`:

1. Extract the stanza parse that `deb_has` does into
   `_stanzas(text) -> list[dict[str, str]]`, and rewrite `deb_has` on top of it
   (no behaviour change).
2. Extract the body of `rpm_has` that locates and decompresses the primary
   metadata into
   `_rpm_primary(base_url, el, rpm_arch, fetch) -> ET.Element | None`, which
   returns `None` where `rpm_has` returns `False` today (missing `repomd.xml`
   or primary file). It still raises `ReleaseError` when `repomd.xml` has no
   primary location. Rewrite `rpm_has` on top of it (no behaviour change).
3. Add the two listing helpers:

```python
def deb_package_names(base_url: str, suite: str, arch: str, fetch: Fetch) -> set[str]:
    """Every ``Package:`` name in the suite's ``binary-<arch>/Packages`` index."""
    text = fetch(_deb_url(base_url, suite, arch)).decode("utf-8")
    return {s["Package"] for s in _stanzas(text) if "Package" in s}


def rpm_package_names(base_url: str, el: str, rpm_arch: str, fetch: Fetch) -> set[str]:
    """Every package name in ``<el>/<rpm_arch>``'s primary metadata (empty if unpublished)."""
    meta = _rpm_primary(base_url, el, rpm_arch, fetch)
    if meta is None:
        return set()
    names: set[str] = set()
    for pkg in meta:
        name_el = next((c for c in pkg if _local(c.tag) == "name"), None)
        if name_el is not None and name_el.text:
            names.add(name_el.text)
    return names
```

In `src/vergil_tooling/lib/package/python_builder.py`, add below
`_SHARED_SUITE`:

```python
# Public alias: vrg-update-deps resolves served runtimes against this suite.
SHARED_SUITE = _SHARED_SUITE
```

- [ ] **Step 4: Write `vergil_index.py`**

```python
# src/vergil_tooling/lib/update_deps/vergil_index.py
"""vergil-python runtimes the live vergil package repository serves (spec §8.2).

The package build installs ``vergil-python<runtime>`` from the shared deb suite
(non-native cells) or the target's own suite (native cells), and install-test
resolves it from every selected target's index. A runtime is therefore a bump
candidate only when **every** one of those indexes serves it. A version missing
from any of them would merge green and then fail the next release.
"""

from __future__ import annotations

import re
from typing import TYPE_CHECKING

from vergil_tooling.lib import config
from vergil_tooling.lib.package import matrix, orgs, python_builder
from vergil_tooling.lib.package.targets import ARCHES
from vergil_tooling.lib.release import package_index
from vergil_tooling.lib.update_deps.context import UpdateDepsError

if TYPE_CHECKING:
    from pathlib import Path

    from vergil_tooling.lib.config import PackageConfig

_RUNTIME_RE = re.compile(r"^vergil-python(\d+\.\d+\.\d+)$")


def _key(v: str) -> tuple[int, ...]:
    return tuple(int(p) for p in v.split("."))


def runtime_indexes(pkg: PackageConfig) -> list[tuple[str, str, str]]:
    """``(fmt, suite, arch)`` of every index the build or install-test reads the runtime from.

    ``arch`` is the deb arch for deb indexes and the rpm arch for rpm ones.
    """
    indexes = [("deb", python_builder.SHARED_SUITE, arch) for arch in ARCHES]
    for t in matrix.select_targets(pkg):
        entry = (t.fmt, t.suite, t.arch if t.fmt == "deb" else t.rpm_arch)
        if entry not in indexes:
            indexes.append(entry)
    return indexes


def served_runtimes(
    pkg: PackageConfig,
    *,
    fetch: package_index.Fetch = package_index._fetch,  # noqa: SLF001
) -> set[str]:
    base = orgs.for_vendor("vergil").base_url
    per_index: list[set[str]] = []
    for fmt, suite, arch in runtime_indexes(pkg):
        if fmt == "deb":
            names = package_index.deb_package_names(base, suite, arch, fetch)
        else:
            names = package_index.rpm_package_names(base, suite, arch, fetch)
        per_index.append({m.group(1) for n in names if (m := _RUNTIME_RE.match(n))})
    served = set.intersection(*per_index)
    if not served:
        raise UpdateDepsError(
            phase="update",
            command=f"list vergil-python runtimes in {base}",
            message="The vergil package index lists no vergil-python runtime served by every "
                    "build and install-test index; refusing to treat a failed lookup as "
                    "'no change'.",
        )
    return served


def served_runtimes_for(base: Path) -> set[str]:
    """``served_runtimes`` for the ``[package]`` block in ``base/vergil.toml``."""
    pkg = config.read_config(base).package
    if pkg is None:
        raise UpdateDepsError(phase="update", command="served_runtimes_for",
                              message="[package] is missing, so there is no runtime to resolve.")
    return served_runtimes(pkg)


def newest_patch(current: str, served: set[str]) -> str | None:
    major_minor = _key(current)[:2]
    same = [v for v in served if _key(v)[:2] == major_minor and _key(v) > _key(current)]
    return max(same, key=_key) if same else None


def newer_minor(current: str, served: set[str]) -> str | None:
    newer = [v for v in served if _key(v)[:2] > _key(current)[:2]]
    return max(newer, key=_key) if newer else None
```

- [ ] **Step 5: Run the index tests. Expect PASS**

Run: `vrg-container-run -- uv run pytest tests/vergil_tooling/test_update_deps_vergil_index.py tests/vergil_tooling -k "package_index or vergil_index" -v`
Expected: PASS.

- [ ] **Step 6: Failing updater tests**

Add to `tests/vergil_tooling/test_update_deps_vergil_eco.py`:

```python
def test_normalize_refs_covers_action_yml(tmp_path) -> None:
    action = tmp_path / "actions" / "ci" / "x" / "action.yml"
    action.parent.mkdir(parents=True)
    action.write_text("runs:\n  steps:\n    - uses: vergil-project/vergil-actions/actions/y@v2.0\n")
    skipped = tmp_path / ".venv" / "lib" / "action.yml"
    skipped.parent.mkdir(parents=True)
    skipped.write_text("    - uses: vergil-project/vergil-actions/actions/y@v2.0\n")
    changed = vergil_eco.normalize_refs(tmp_path, "v2.1")
    assert changed == [action]
    assert "@v2.1" in action.read_text()
    assert "@v2.0" in skipped.read_text()


_TOML = '[package.python]\n# comment\nruntime = "3.14.8"\n'


def test_bump_runtime_rewrites_to_newest_served_patch(tmp_path) -> None:
    (tmp_path / "vergil.toml").write_text(_TOML)
    new, entry = vergil_eco.bump_runtime(tmp_path, {"3.14.8", "3.14.9"})
    assert new == "3.14.9"
    assert entry is None
    assert 'runtime = "3.14.9"' in (tmp_path / "vergil.toml").read_text()


def test_bump_runtime_never_picks_unserved(tmp_path) -> None:
    (tmp_path / "vergil.toml").write_text(_TOML)
    new, _ = vergil_eco.bump_runtime(tmp_path, {"3.14.8"})
    assert new is None
    assert (tmp_path / "vergil.toml").read_text() == _TOML


def test_bump_runtime_reports_newer_minor(tmp_path) -> None:
    (tmp_path / "vergil.toml").write_text(_TOML)
    _, entry = vergil_eco.bump_runtime(tmp_path, {"3.14.8", "3.15.1"})
    assert entry == HeldBack(source="vergil", name="vergil-python", current="3.14.8",
                             latest="3.15.1",
                             note="[package.python] runtime: a new minor is a judgement call")


def test_bump_runtime_absent_is_noop(tmp_path) -> None:
    (tmp_path / "vergil.toml").write_text('[dependencies]\nvergil = "v2.1"\n')
    assert vergil_eco.bump_runtime(tmp_path, set()) == (None, None)


def test_apply_skips_index_when_no_runtime(tmp_path) -> None:
    (tmp_path / "vergil.toml").write_text('[dependencies]\nvergil = "v2.1"\n')

    def must_not_call(base) -> set[str]:
        raise AssertionError("index lookup without [package.python]")

    ctx = UpdateDepsContext(repo="o/r", repo_root=tmp_path, worktree_path=tmp_path)
    result = vergil_eco.VergilUpdater(served_runtimes=must_not_call).apply(ctx)
    assert result.changed is False


def test_apply_combines_runtime_bump(tmp_path) -> None:
    (tmp_path / "vergil.toml").write_text('[dependencies]\nvergil = "v2.1"\n\n' + _TOML)
    ctx = UpdateDepsContext(repo="o/r", repo_root=tmp_path, worktree_path=tmp_path)
    seen: list[Path] = []
    updater = vergil_eco.VergilUpdater(served_runtimes=lambda base: seen.append(base) or {"3.14.9"})
    result = updater.apply(ctx)
    assert seen == [tmp_path]
    assert result.changed is True
    assert "vergil-python runtime 3.14.9" in result.commit_message
```

Import `HeldBack`, `UpdateDepsContext` and `pathlib.Path` in the test file if needed.

- [ ] **Step 7: Run. Expect FAIL**

Run: `vrg-container-run -- uv run pytest tests/vergil_tooling/test_update_deps_vergil_eco.py -v`
Expected: FAIL.

- [ ] **Step 8: Implement in `vergil_eco.py`**

Add `import re` and `import tomllib`; import `HeldBack` and `vergil_index`; and
under `TYPE_CHECKING` import `from collections.abc import Callable, Iterator`.

```python
# Directories never scanned for action files: tool caches and nested checkouts.
_SKIP_DIRS = frozenset({".git", ".venv", ".worktrees", "node_modules", ".tox"})
_RUNTIME_LINE_RE = r'^(runtime\s*=\s*")({old})(")'


def _ref_files(base: Path) -> Iterator[Path]:
    workflows = base / ".github" / "workflows"
    if workflows.is_dir():
        yield from sorted([*workflows.glob("*.yml"), *workflows.glob("*.yaml")])
    actions = [
        p for name in ("action.yml", "action.yaml") for p in base.rglob(name)
        if not _SKIP_DIRS.intersection(p.relative_to(base).parts)
    ]
    yield from sorted(actions)


def normalize_refs(base: Path, target: str) -> list[Path]:
    """Rewrite drifting ``uses: vergil-*@vX.Y`` refs in workflows and action files."""
    changed: list[Path] = []
    for path in _ref_files(base):
        text = path.read_text()
        new = _REF_RE.sub(lambda m: m.group(1) + target, text)
        if new != text:
            path.write_text(new)
            changed.append(path)
    return changed


def bump_runtime(base: Path, served: set[str]) -> tuple[str | None, HeldBack | None]:
    """Bump ``[package.python] runtime`` to the newest served patch; report a newer minor."""
    path = base / "vergil.toml"
    text = path.read_text()
    current = tomllib.loads(text).get("package", {}).get("python", {}).get("runtime")
    if not isinstance(current, str):
        return None, None
    entry = None
    minor = vergil_index.newer_minor(current, served)
    if minor is not None:
        entry = HeldBack(source="vergil", name="vergil-python", current=current, latest=minor,
                         note="[package.python] runtime: a new minor is a judgement call")
    patch = vergil_index.newest_patch(current, served)
    if patch is None:
        return None, entry
    pattern = re.compile(_RUNTIME_LINE_RE.format(old=re.escape(current)), re.MULTILINE)
    new_text, count = pattern.subn(lambda m: m.group(1) + patch + m.group(3), text)
    if count != 1:
        raise UpdateDepsError(phase="update", command="bump_runtime",
                              message=f"Expected exactly one runtime = \"{current}\" line in "
                                      f"vergil.toml, found {count}.")
    path.write_text(new_text)
    return patch, entry


def _declares_runtime(base: Path) -> bool:
    data = tomllib.loads((base / "vergil.toml").read_text())
    return isinstance(data.get("package", {}).get("python", {}).get("runtime"), str)
```

Import `UpdateDepsError` from `context`. In `VergilUpdater`:

```python
    def __init__(
        self, served_runtimes: Callable[[Path], set[str]] = vergil_index.served_runtimes_for
    ) -> None:
        self._served_runtimes = served_runtimes

    def _runtime(self, base: Path) -> tuple[str | None, list[HeldBack]]:
        if not _declares_runtime(base):
            return None, []
        new, entry = bump_runtime(base, self._served_runtimes(base))
        return new, [entry] if entry is not None else []
```

At the end of both the bump branch and the normalize branch of `apply()`, call
`runtime, held = self._runtime(base)` and fold it into the result:

```python
        parts = [f"normalize vergil refs to {target}"] if changed else []
        if runtime is not None:
            parts.append(f"vergil-python runtime {runtime}")
        return UpdateResult(
            updater=self.name,
            changed=changed or runtime is not None,
            summary="; ".join(parts) if parts else f"vergil refs already at {target}",
            commit_message="chore(deps): " + ("; ".join(parts) if parts else "vergil refs"),
            held_back=held,
        )
```

For the bump branch, keep `bump vergil to {target}` as the first part. Make sure
the existing commit-message tests still pass: if one asserts the old exact
message `chore(deps): normalize vergil ecosystem refs (v2.1)`, update it to the
new format in this step.

- [ ] **Step 9: Run. Expect PASS**

Run: `vrg-container-run -- uv run pytest tests/vergil_tooling -k "vergil_eco or vergil_index or package_index" -v`
Expected: PASS.

- [ ] **Step 10: Full validation, then commit**

Run: `vrg-container-run -- vrg-validate`. Expected: PASS.

```bash
vrg-git add src/vergil_tooling/lib tests/vergil_tooling
vrg-commit --type feat --scope update-deps \
  --message "normalize action.yml refs and bump the vergil-python runtime from the package index"
```

---

### Task 7: Remove the `@v2.1` pin from `repo_init.py` templates [`vergil-tooling`]

Covers spec §8.4, first bullet.

**Files:**

- Modify: `src/vergil_tooling/lib/repo_init.py`
- Test: `tests/vergil_tooling/test_repo_init.py` (grep for `render_ci_workflow` tests)

**Interfaces:**

- Produces: `repo_init.default_vergil_series() -> str` (for example `"v2.1"`
  from installed `vergil-tooling` 2.1.242); `render_epic_rollup_workflow(vergil_version: str) -> str`.

- [ ] **Step 1: Failing tests**

```python
def test_default_vergil_series_from_installed_version(monkeypatch) -> None:
    monkeypatch.setattr(repo_init.metadata, "version", lambda name: "9.4.17")
    assert repo_init.default_vergil_series() == "v9.4"


def test_templates_use_context_series_not_literal(monkeypatch) -> None:
    ctx = _ci_ctx()  # the file's existing factory for render_ci_workflow tests
    ctx.vergil_version = "v9.4"
    rendered = (repo_init.render_ci_workflow(ctx) + repo_init.render_cd_workflow(ctx)
                + repo_init.render_ops_workflow(ctx)
                + repo_init.render_epic_rollup_workflow("v9.4"))
    assert "@v2.1\n" not in rendered
    assert "ci-quality.yml@v9.4\n" in rendered
    assert "ops-epic-rollup.yml@v9.4\n" in rendered


def test_context_default_series_tracks_tooling(monkeypatch) -> None:
    monkeypatch.setattr(repo_init.metadata, "version", lambda name: "3.0.1")
    assert repo_init.RepoInitContext().vergil_version == "v3.0"
```

(If `RepoInitContext()` requires arguments, construct it the way the existing
tests do. If no `_ci_ctx` factory exists, build the context the way the
existing `render_ci_workflow` tests do.)

- [ ] **Step 2: Run. Expect FAIL**

Run: `vrg-container-run -- uv run pytest tests/vergil_tooling/test_repo_init.py -k "series or templates_use" -v`
Expected: FAIL.

- [ ] **Step 3: Implement**

```python
from importlib import metadata


def default_vergil_series() -> str:
    """``vX.Y`` of the installed vergil-tooling: new repos pin the series that made them."""
    major, minor = metadata.version("vergil-tooling").split(".")[:2]
    return f"v{major}.{minor}"
```

- Change the field to
  `vergil_version: str = field(default_factory=default_vergil_series)`.
- Change `vv_default = deps.get("vergil", "v2.1")` to
  `deps.get("vergil", default_vergil_series())`.
- In `render_ci_workflow`, `render_cd_workflow` and `render_ops_workflow`,
  replace each literal `@v2.1\n` in a `uses:` line with `@{ctx.vergil_version}\n`,
  turning the string into an f-string. Find them with
  `grep -n '@v2\.1\\n' src/vergil_tooling/lib/repo_init.py`.
- Change `render_epic_rollup_workflow()` to
  `render_epic_rollup_workflow(vergil_version: str)` and use it the same way.
  Update its caller (around line 1421) to pass `ctx.vergil_version`.
- Leave comments that cite a historical version, such as
  `# Requested by ci-security.yml@v2.1.3: …`, unchanged. They are history,
  not pins.

- [ ] **Step 4: Run all repo_init tests. Expect PASS**

Run: `vrg-container-run -- uv run pytest tests/vergil_tooling/test_repo_init.py -v`
Expected: PASS. Any existing snapshot test that hard-codes `@v2.1` must set
`ctx.vergil_version = "v2.1"` explicitly rather than rely on the default.

- [ ] **Step 5: Full validation, then commit**

Run: `vrg-container-run -- vrg-validate`. Expected: PASS.

```bash
vrg-git add src/vergil_tooling/lib/repo_init.py tests/vergil_tooling/test_repo_init.py
vrg-commit --type refactor --scope repo-init \
  --message "derive the vergil series for new-repo templates from the installed tooling"
```

---

### Task 8: `images` updater [`vergil-tooling`]

Covers spec §7.3 (the `images` row) and §8.3.

**Files:**

- Create: `src/vergil_tooling/lib/update_deps/registry.py`
- Create: `src/vergil_tooling/lib/update_deps/updaters/images.py`
- Modify: `src/vergil_tooling/lib/update_deps/orchestrator.py` (registry: `python, vergil, images, hook`)
- Test: `tests/vergil_tooling/test_update_deps_registry.py` (new)
- Test: `tests/vergil_tooling/test_update_deps_images.py` (new)
- Test: `tests/vergil_tooling/test_update_deps_orchestrator.py` (registry names)

**Interfaces:**

- Consumes: `report.HeldBack` (Task 1); `git.read_output("ls-files")`.
- Produces:
  - `registry.ImageRef(registry: str, repository: str, tag: str, text: str)`
    (frozen; `text` is the ref as written)
  - `registry.parse_image_ref(text: str) -> ImageRef | None`
  - `registry.Opener = Callable[[urllib.request.Request], tuple[int, dict[str, str], bytes]]`
  - `registry.list_tags(ref: ImageRef, *, opener: Opener = _open) -> list[str]` (raises `UpdateDepsError`)
  - `images.find_refs(base: Path, files: list[str]) -> list[tuple[Path, ImageRef]]`
  - `images.plan(ref: ImageRef, tags: list[str]) -> tuple[str | None, HeldBack | None]`
    (new tag to bump to, and the report entry)
  - `images.ImagesUpdater(list_tags: Callable[[ImageRef], list[str]] = registry.list_tags)`
    with `name = "images"`

- [ ] **Step 1: Failing registry tests**

```python
# tests/vergil_tooling/test_update_deps_registry.py
from __future__ import annotations

import json

import pytest

from vergil_tooling.lib.update_deps import registry
from vergil_tooling.lib.update_deps.context import UpdateDepsError
from vergil_tooling.lib.update_deps.registry import ImageRef


@pytest.mark.parametrize(
    ("text", "expected"),
    [
        ("ghcr.io/aquasecurity/trivy:0.71.0",
         ImageRef("ghcr.io", "aquasecurity/trivy", "0.71.0", "ghcr.io/aquasecurity/trivy:0.71.0")),
        ("python:3.14-slim",
         ImageRef("registry-1.docker.io", "library/python", "3.14-slim", "python:3.14-slim")),
        ("hadolint/hadolint:v2.12.0",
         ImageRef("registry-1.docker.io", "hadolint/hadolint", "v2.12.0", "hadolint/hadolint:v2.12.0")),
        ("localhost:5000/x:1.0.0", ImageRef("localhost:5000", "x", "1.0.0", "localhost:5000/x:1.0.0")),
    ],
)
def test_parse_image_ref(text, expected) -> None:
    assert registry.parse_image_ref(text) == expected


@pytest.mark.parametrize("text", [
    "ubuntu:latest", "ubuntu", "alpine@sha256:abc", "ubuntu:24.04@sha256:abc",
    "ghcr.io/vergil-project/prod-base:1.2.3", "${{ matrix.image }}", "$BASE_IMAGE",
    "ghcr.io/x/${{ inputs.y }}:1.0.0",
])
def test_parse_image_ref_skips(text) -> None:
    assert registry.parse_image_ref(text) is None


def _opener(responses):
    calls = []

    def opener(req):
        calls.append((req.full_url, req.headers.get("Authorization")))
        return responses.pop(0)

    return opener, calls


def test_list_tags_anonymous_token_and_pagination() -> None:
    ref = ImageRef("ghcr.io", "aquasecurity/trivy", "0.71.0", "t")
    opener, calls = _opener([
        (401, {"WWW-Authenticate": 'Bearer realm="https://ghcr.io/token",service="ghcr.io",'
                                   'scope="repository:aquasecurity/trivy:pull"'}, b""),
        (200, {}, json.dumps({"token": "T"}).encode()),
        (200, {"Link": '</v2/aquasecurity/trivy/tags/list?n=1000&last=0.71.0>; rel="next"'},
         json.dumps({"tags": ["0.70.0", "0.71.0"]}).encode()),
        (200, {}, json.dumps({"tags": ["0.74.0"]}).encode()),
    ])
    assert registry.list_tags(ref, opener=opener) == ["0.70.0", "0.71.0", "0.74.0"]
    assert calls[1][0] == ("https://ghcr.io/token?service=ghcr.io"
                           "&scope=repository%3Aaquasecurity%2Ftrivy%3Apull")
    assert calls[2][1] == "Bearer T"
    assert calls[3][0] == "https://ghcr.io/v2/aquasecurity/trivy/tags/list?n=1000&last=0.71.0"


def test_list_tags_fails_loud_on_error_status() -> None:
    ref = ImageRef("ghcr.io", "a/b", "1.0.0", "t")
    opener, _ = _opener([(500, {}, b"boom")])
    with pytest.raises(UpdateDepsError, match="ghcr.io/a/b"):
        registry.list_tags(ref, opener=opener)


def test_list_tags_fails_loud_on_unsupported_challenge() -> None:
    ref = ImageRef("ghcr.io", "a/b", "1.0.0", "t")
    opener, _ = _opener([(401, {"WWW-Authenticate": "Basic realm=x"}, b"")])
    with pytest.raises(UpdateDepsError, match="Bearer"):
        registry.list_tags(ref, opener=opener)
```

- [ ] **Step 2: Run. Expect FAIL**

Run: `vrg-container-run -- uv run pytest tests/vergil_tooling/test_update_deps_registry.py -v`
Expected: FAIL.

- [ ] **Step 3: Write `registry.py`**

```python
# src/vergil_tooling/lib/update_deps/registry.py
"""Image-ref parsing and anonymous Docker Registry v2 tag listing (spec §8.3).

The opener is injected so tests never touch the network. Any lookup failure
fails loud (spec §3.7).
"""

from __future__ import annotations

import json
import re
import urllib.error
import urllib.parse
import urllib.request
from collections.abc import Callable
from dataclasses import dataclass

from vergil_tooling.lib.update_deps.context import UpdateDepsError

Opener = Callable[[urllib.request.Request], tuple[int, dict[str, str], bytes]]

_DOCKER_HUB = "registry-1.docker.io"
_SKIP_PREFIXES = ("ghcr.io/vergil-project/",)
_REF_RE = re.compile(r"^(?P<name>[a-z0-9][a-z0-9._/:-]*?):(?P<tag>[A-Za-z0-9_][A-Za-z0-9._-]{0,127})$")
_CHALLENGE_RE = re.compile(r'(\w+)="([^"]*)"')
_NEXT_RE = re.compile(r'<([^>]+)>;\s*rel="next"')


@dataclass(frozen=True)
class ImageRef:
    registry: str
    repository: str
    tag: str
    text: str


def parse_image_ref(text: str) -> ImageRef | None:
    """Parse a tagged image ref; None for refs this updater skips (spec §8.3 table)."""
    if "$" in text or "@" in text or text.startswith(_SKIP_PREFIXES):
        return None
    match = _REF_RE.match(text)
    if match is None or match["tag"] == "latest":
        return None
    name = match["name"]
    first, _, rest = name.partition("/")
    if rest and ("." in first or ":" in first or first == "localhost"):
        reg, repo = first, rest
    else:
        reg, repo = _DOCKER_HUB, name if "/" in name else f"library/{name}"
    return ImageRef(reg, repo, match["tag"], text)


def _open(req: urllib.request.Request) -> tuple[int, dict[str, str], bytes]:
    try:
        with urllib.request.urlopen(req, timeout=30) as resp:  # noqa: S310 - https registry URLs
            return resp.status, dict(resp.headers.items()), resp.read()
    except urllib.error.HTTPError as exc:
        return exc.code, dict(exc.headers.items()), exc.read()


def _fail(ref: ImageRef, why: str) -> UpdateDepsError:
    return UpdateDepsError(phase="update", command=f"list tags {ref.registry}/{ref.repository}",
                           message=f"Tag lookup failed for {ref.registry}/{ref.repository}: {why}")


def _token(ref: ImageRef, challenge: str, opener: Opener) -> str:
    if not challenge.startswith("Bearer "):
        raise _fail(ref, f"unsupported auth challenge (need Bearer): {challenge!r}")
    params = dict(_CHALLENGE_RE.findall(challenge))
    realm = params.pop("realm", "")
    query = urllib.parse.urlencode({k: params[k] for k in ("service", "scope") if k in params})
    status, _, body = opener(urllib.request.Request(f"{realm}?{query}"))
    if status != 200:
        raise _fail(ref, f"token endpoint returned HTTP {status}")
    token = json.loads(body).get("token") or json.loads(body).get("access_token")
    if not isinstance(token, str):
        raise _fail(ref, "token endpoint returned no token")
    return token


def list_tags(ref: ImageRef, *, opener: Opener = _open) -> list[str]:
    url = f"https://{ref.registry}/v2/{ref.repository}/tags/list?n=1000"
    headers: dict[str, str] = {}
    tags: list[str] = []
    retried = False
    while True:
        status, resp_headers, body = opener(urllib.request.Request(url, headers=headers))
        if status == 401 and not retried:
            challenge = resp_headers.get("WWW-Authenticate") or resp_headers.get("www-authenticate", "")
            headers = {"Authorization": f"Bearer {_token(ref, challenge, opener)}"}
            retried = True
            continue
        if status != 200:
            raise _fail(ref, f"HTTP {status}")
        page = json.loads(body).get("tags") or []
        tags.extend(t for t in page if isinstance(t, str))
        link = resp_headers.get("Link") or resp_headers.get("link")
        nxt = _NEXT_RE.search(link) if link else None
        if nxt is None:
            return tags
        url = urllib.parse.urljoin(f"https://{ref.registry}", nxt.group(1))
```

- [ ] **Step 4: Run the registry tests. Expect PASS**

Run: `vrg-container-run -- uv run pytest tests/vergil_tooling/test_update_deps_registry.py -v`
Expected: PASS.

- [ ] **Step 5: Failing images-updater tests**

```python
# tests/vergil_tooling/test_update_deps_images.py
from __future__ import annotations

import pytest

from vergil_tooling.lib.update_deps.context import UpdateDepsContext, UpdateDepsError
from vergil_tooling.lib.update_deps.registry import ImageRef, parse_image_ref
from vergil_tooling.lib.update_deps.report import HeldBack
from vergil_tooling.lib.update_deps.updaters import images


def _ref(text: str) -> ImageRef:
    ref = parse_image_ref(text)
    assert ref is not None
    return ref


def test_plan_exact_bumps_within_major_including_zero() -> None:
    new, entry = images.plan(_ref("ghcr.io/aquasecurity/trivy:0.71.0"),
                             ["0.70.0", "0.71.0", "0.74.0", "0.74.0-amd64", "1.0.0"])
    assert new == "0.74.0"
    assert entry == HeldBack(source="images", name="ghcr.io/aquasecurity/trivy",
                             current="0.71.0", latest="1.0.0",
                             note="newer major; never bumped automatically")


def test_plan_exact_keeps_v_prefix() -> None:
    new, entry = images.plan(_ref("hadolint/hadolint:v2.12.0"), ["v2.12.0", "v2.14.1", "2.99.0"])
    assert new == "v2.14.1"
    assert entry is None


def test_plan_exact_already_newest() -> None:
    assert images.plan(_ref("a/b:1.2.3"), ["1.2.3", "1.2.2"]) == (None, None)


def test_plan_series_reports_newer_series_never_bumps() -> None:
    new, entry = images.plan(_ref("python:3.14-slim"),
                             ["3.13-slim", "3.14-slim", "3.15-slim", "3.15", "3.15-alpine"])
    assert new is None
    assert entry == HeldBack(source="images", name="python", current="3.14-slim",
                             latest="3.15-slim", note="newer series; a judgement call")


def test_plan_series_bare_numbers() -> None:
    _, entry = images.plan(_ref("ubuntu:24.04"), ["22.04", "24.04", "26.04", "24.04.1"])
    assert entry is not None and entry.latest == "26.04"


def test_plan_other_tag_shapes_skipped() -> None:
    assert images.plan(_ref("a/b:stable"), ["stable", "1.0.0"]) == (None, None)


def test_find_refs_scans_dockerfiles_workflows_and_action_inputs(tmp_path) -> None:
    (tmp_path / "docker").mkdir()
    (tmp_path / "docker" / "Dockerfile").write_text(
        "ARG BASE=x\nFROM --platform=$BUILDPLATFORM python:3.14-slim AS build\nFROM $BASE\n")
    wf = tmp_path / ".github" / "workflows"
    wf.mkdir(parents=True)
    (wf / "ci.yml").write_text(
        "jobs:\n  a:\n    container: hadolint/hadolint:v2.12.0\n"
        "  b:\n    container: ${{ matrix.image }}\n"
        "    services:\n      db:\n        image: \"postgres:16\"\n")
    act = tmp_path / "actions" / "trivy" / "action.yml"
    act.parent.mkdir(parents=True)
    act.write_text("inputs:\n  trivy-image:\n    default: \"ghcr.io/aquasecurity/trivy:0.71.0\"\n"
                   "  format:\n    default: \"sarif:2.1.0\"\n")
    files = ["docker/Dockerfile", ".github/workflows/ci.yml", "actions/trivy/action.yml"]
    found = [(p.relative_to(tmp_path).as_posix(), r.text) for p, r in images.find_refs(tmp_path, files)]
    assert found == [
        ("docker/Dockerfile", "python:3.14-slim"),
        (".github/workflows/ci.yml", "hadolint/hadolint:v2.12.0"),
        (".github/workflows/ci.yml", "postgres:16"),
        ("actions/trivy/action.yml", "ghcr.io/aquasecurity/trivy:0.71.0"),
    ]


def test_apply_rewrites_exact_ref_and_collects_report(monkeypatch, tmp_path) -> None:
    act = tmp_path / "action.yml"
    act.write_text("inputs:\n  trivy-image:\n    default: \"ghcr.io/aquasecurity/trivy:0.71.0\"\n")
    monkeypatch.setattr(images.git, "read_output", lambda *a: "action.yml")
    tags = {"aquasecurity/trivy": ["0.71.0", "0.74.0", "1.0.0"]}
    ctx = UpdateDepsContext(repo="o/r", repo_root=tmp_path, worktree_path=tmp_path)
    result = images.ImagesUpdater(list_tags=lambda ref: tags[ref.repository]).apply(ctx)
    assert result.changed is True
    assert "trivy:0.74.0" in act.read_text()
    assert [e.latest for e in result.held_back] == ["1.0.0"]
    assert result.commit_message == "chore(deps): bump image tags"


def test_apply_lookup_failure_is_loud(monkeypatch, tmp_path) -> None:
    (tmp_path / "Dockerfile").write_text("FROM a/b:1.0.0\n")
    monkeypatch.setattr(images.git, "read_output", lambda *a: "Dockerfile")

    def boom(ref):
        raise UpdateDepsError(phase="update", command="x", message="Tag lookup failed")

    ctx = UpdateDepsContext(repo="o/r", repo_root=tmp_path, worktree_path=tmp_path)
    with pytest.raises(UpdateDepsError, match="Tag lookup failed"):
        images.ImagesUpdater(list_tags=boom).apply(ctx)


def test_applies_when_any_scanned_file_exists(monkeypatch, tmp_path) -> None:
    ctx = UpdateDepsContext(repo="o/r", repo_root=tmp_path, worktree_path=tmp_path)
    monkeypatch.setattr(images.git, "read_output", lambda *a: "")
    assert images.ImagesUpdater().applies(ctx) is False
    monkeypatch.setattr(images.git, "read_output", lambda *a: "README.md\nDockerfile")
    assert images.ImagesUpdater().applies(ctx) is True
```

- [ ] **Step 6: Run. Expect FAIL**

Run: `vrg-container-run -- uv run pytest tests/vergil_tooling/test_update_deps_images.py -v`
Expected: FAIL.

- [ ] **Step 7: Write the updater**

```python
# src/vergil_tooling/lib/update_deps/updaters/images.py
"""Container image updater (spec §8.3).

Exact ``x.y.z`` tags bump to the newest tag with the same major (major 0
included); a newer major is reported. Series tags (``3.14-slim``, ``24.04``)
float and are only reported. ``:latest``, digests, expressions and
``ghcr.io/vergil-project/*`` are skipped.
"""

from __future__ import annotations

import re
from collections.abc import Callable
from pathlib import PurePosixPath
from typing import TYPE_CHECKING

import yaml

from vergil_tooling.lib import git
from vergil_tooling.lib.update_deps import registry
from vergil_tooling.lib.update_deps.registry import ImageRef, parse_image_ref
from vergil_tooling.lib.update_deps.report import HeldBack
from vergil_tooling.lib.update_deps.updater import UpdateResult

if TYPE_CHECKING:
    from pathlib import Path

    from vergil_tooling.lib.update_deps.context import UpdateDepsContext

_EXACT_RE = re.compile(r"^(v?)(\d+)\.(\d+)\.(\d+)$")
_SERIES_RE = re.compile(r"^(\d+)(?:\.(\d+))?((?:-[A-Za-z0-9._]+)?)$")
_FROM_RE = re.compile(r"^\s*FROM\s+(?:--platform=\S+\s+)?(\S+)", re.IGNORECASE | re.MULTILINE)
_WF_RE = re.compile(r"""^\s*(?:image|container):\s*["']?([^\s"'#]+)""", re.MULTILINE)


def _is_dockerfile(path: str) -> bool:
    name = PurePosixPath(path).name
    return name == "Dockerfile" or name.startswith("Dockerfile.") or name.lower().endswith(".dockerfile")


def _is_workflow(path: str) -> bool:
    return path.startswith(".github/workflows/") and path.endswith((".yml", ".yaml"))


def _is_action(path: str) -> bool:
    return PurePosixPath(path).name in ("action.yml", "action.yaml")


def _scanned(path: str) -> bool:
    return _is_dockerfile(path) or _is_workflow(path) or _is_action(path)


def _action_image_defaults(text: str) -> list[str]:
    data = yaml.safe_load(text) or {}
    inputs = data.get("inputs") if isinstance(data, dict) else None
    if not isinstance(inputs, dict):
        return []
    return [str(spec["default"]) for name, spec in inputs.items()
            if str(name).endswith("image") and isinstance(spec, dict)
            and isinstance(spec.get("default"), str)]


def find_refs(base: Path, files: list[str]) -> list[tuple[Path, ImageRef]]:
    found: list[tuple[Path, ImageRef]] = []
    for rel in files:
        if not _scanned(rel):
            continue
        path = base / rel
        text = path.read_text()
        if _is_dockerfile(rel):
            candidates = _FROM_RE.findall(text)
        elif _is_workflow(rel):
            candidates = _WF_RE.findall(text)
        else:
            candidates = _action_image_defaults(text)
        for candidate in candidates:
            ref = parse_image_ref(candidate)
            if ref is not None:
                found.append((path, ref))
    return found


def _display(ref: ImageRef) -> str:
    return ref.text.rsplit(":", 1)[0]


def plan(ref: ImageRef, tags: list[str]) -> tuple[str | None, HeldBack | None]:
    exact = _EXACT_RE.match(ref.tag)
    if exact is not None:
        prefix = exact[1]
        cur = tuple(int(x) for x in exact.groups()[1:])
        versions = [(tuple(int(x) for x in m.groups()[1:]), t) for t in tags
                    if (m := _EXACT_RE.match(t)) and m[1] == prefix]
        same = [v for v in versions if v[0][0] == cur[0] and v[0] > cur]
        higher = [v for v in versions if v[0][0] > cur[0]]
        new = max(same)[1] if same else None
        entry = (HeldBack(source="images", name=_display(ref), current=ref.tag,
                          latest=max(higher)[1], note="newer major; never bumped automatically")
                 if higher else None)
        return new, entry
    series = _SERIES_RE.match(ref.tag)
    if series is None:
        return None, None
    width = 2 if series[2] is not None else 1
    suffix = series[3]
    cur_key = tuple(int(x) for x in series.groups()[:width])
    newer = []
    for t in tags:
        m = _SERIES_RE.match(t)
        if m is None or m[3] != suffix or (m[2] is not None) != (width == 2):
            continue
        key = tuple(int(x) for x in m.groups()[:width])
        if key > cur_key:
            newer.append((key, t))
    if not newer:
        return None, None
    return None, HeldBack(source="images", name=_display(ref), current=ref.tag,
                          latest=max(newer)[1], note="newer series; a judgement call")


def _base(ctx: UpdateDepsContext) -> Path:
    return ctx.worktree_path if ctx.worktree_path is not None else ctx.repo_root


def _tracked_files() -> list[str]:
    return [line for line in git.read_output("ls-files").splitlines() if line]


class ImagesUpdater:
    name = "images"

    def __init__(self, list_tags: Callable[[ImageRef], list[str]] = registry.list_tags) -> None:
        self._list_tags = list_tags

    def applies(self, ctx: UpdateDepsContext) -> bool:  # noqa: ARG002
        return any(_scanned(f) for f in _tracked_files())

    def apply(self, ctx: UpdateDepsContext) -> UpdateResult:
        base = _base(ctx)
        refs = find_refs(base, _tracked_files())
        held: list[HeldBack] = []
        bumped: list[str] = []
        cache: dict[tuple[str, str], list[str]] = {}
        for path, ref in refs:
            key = (ref.registry, ref.repository)
            if key not in cache:
                cache[key] = self._list_tags(ref)
            new_tag, entry = plan(ref, cache[key])
            if entry is not None and entry not in held:
                held.append(entry)
            if new_tag is not None:
                new_text = f"{_display(ref)}:{new_tag}"
                path.write_text(path.read_text().replace(ref.text, new_text))
                bumped.append(f"{ref.text} → {new_text}")
        return UpdateResult(
            updater=self.name,
            changed=bool(bumped),
            summary="; ".join(bumped) if bumped else "image tags already newest within their major",
            commit_message="chore(deps): bump image tags",
            held_back=held,
        )
```

`git.read_output("ls-files")` runs in the cwd, which is the worktree after
preflight. Note that `str.replace` changes every occurrence of `ref.text` in the
file. That is intended: two copies of the same pinned ref must not diverge.

- [ ] **Step 8: Register it**

`DEFAULT_REGISTRY = [PythonUvUpdater(), VergilUpdater(), ImagesUpdater(), RepoHookUpdater()]`.
Update `test_default_registry_names` to `["python", "vergil", "images", "hook"]`.

- [ ] **Step 9: Run. Expect PASS**

Run: `vrg-container-run -- uv run pytest tests/vergil_tooling -k "images or registry or orchestrator" -v`
Expected: PASS.

- [ ] **Step 10: Full validation, then commit**

Run: `vrg-container-run -- vrg-validate`. Expected: PASS.

```bash
vrg-git add src/vergil_tooling/lib/update_deps tests/vergil_tooling
vrg-commit --type feat --scope update-deps \
  --message "add the images updater: bump exact tags within their major, report newer series"
```

---

### Task 9: Deployment: release the dependency-update tooling [`vergil-tooling`, `--kind deployment`]

**Blocked by:** Tasks 1–8.

This is an operational task, run with `issue-deploy`, not `issue-implement`.

- **Precondition (human-attested, never performed by the agent):** the human
  has cut a vergil-tooling release containing Tasks 1–8 with `vrg-release`, and
  upgraded the Mac host install
  (`uv tool install --reinstall 'vergil-tooling @ git+https://github.com/vergil-project/vergil-tooling@v2.1'`).
  That release ran on the *previous* tooling, so it has no `update-deps` stage.
  This is expected.
- **Procedure (agent-safe):**
  1. In the agent VM, `vrg-vm update` (or confirm the packaged
     `vergil-tooling` version is at least the attested release) and run
     `vrg-release --help`. Confirm `--without-deps` and `--allow-pending-deps`
     are listed. `--help` exits before the identity check.
  2. Provision the label in the three repos the validation task exercises:
     `vrg-ensure-label --repo vergil-project/vergil-tooling --sync`, and the
     same for `vergil-project/vergil-vm` and `vergil-project/vergil-python`.
     Confirm with
     `vrg-gh label list --repo <repo> --search dependency-update`.
- **Acceptance:** both checks pass, recorded as `Outcome: SUCCESS` with the
  version observed.

Create it with:

```bash
vrg-issue-create --epic vergil-project/.github#381 --repo vergil-project/vergil-tooling \
  --kind deployment --title "Deploy release-integrated dependency updates" \
  --blocked-by vergil-project/vergil-tooling#<T1> … --blocked-by vergil-project/vergil-tooling#<T8>
```

---

### Task 10: vergil-vm hook and `vrg-vm-init.sh` pin removal [`vergil-vm`]

Covers spec §6.3 (vergil-vm), §7.3 (the vergil-vm hook row) and §8.4, second
bullet.

**Blocked by:** Task 9.

**Files:**

- Create: `scripts/dev/update-deps.sh` (the hook)
- Create: `scripts/dev/update-deps-lib.sh` (pure functions, sourced)
- Create: `tests/check-update-deps-hook.sh`
- Modify: `tests/run-host-tests.sh` (add the check)
- Modify: `vergil.toml` (`[dependency-update]`)
- Modify: `scripts/vrg-vm-init.sh` (read the series from `vergil.toml`)

**Interfaces:**

- Consumes: the hook contract (spec §6.2): `$VRG_DEPS_REPORT`, run from the
  worktree root inside the dev container, edit files only, exit 0 on success.
- Produces: `update-deps-lib.sh` functions `semver_newest_same_major <current> <candidates…>`,
  `semver_major <v>`, `set_nerdctl_version <file> <version>`,
  `report_line <source> <name> <current> <latest> [note]`.

- [ ] **Step 1: Precondition check, tofu in the container**

Run: `vrg-container-run -- tofu version`
Expected: `OpenTofu v1.x`. vergil-containers installs it in every base image
(`docker/common/opentofu.dockerfile`). If it is missing, stop and add
`[container].build-command` in this task, per spec §6.3.

- [ ] **Step 2: Determine the lockfile platform set**

The committed `.terraform.lock.hcl` files carry `h1:` hashes for several
platforms. To find the exact set, in a scratch copy of one module run
`tofu providers lock` at the *current* provider version, varying `-platform=`
across `linux_amd64 linux_arm64 darwin_amd64 darwin_arm64 windows_amd64`. The
right set is the one whose `h1:` set matches the committed file exactly. Write
that list into `PLATFORMS` in Step 5. Do not guess: a smaller set silently drops
hashes, which breaks `tofu init` on a dropped platform.

- [ ] **Step 3: Failing check script**

```bash
#!/usr/bin/env bash
# tests/check-update-deps-hook.sh — unit checks for the pure functions behind the
# dependency-update hook (epic vergil-project/.github#381). Offline; no tofu, no network.
set -euo pipefail
ROOT="$(cd "$(dirname "$0")/.." && pwd)"
# shellcheck source=../scripts/dev/update-deps-lib.sh
. "${ROOT}/scripts/dev/update-deps-lib.sh"
fail() { echo "FAIL: $*" >&2; exit 1; }

[ "$(semver_newest_same_major 2.3.3 2.3.3 2.4.0 2.5.1 3.0.0 2.5.1-rc1)" = "2.5.1" ] \
  || fail "newest same major"
[ "$(semver_newest_same_major 2.5.1 2.3.3 2.5.1)" = "2.5.1" ] || fail "already newest"
[ "$(semver_major 3.0.0)" = "3" ] || fail "major"

tmp="$(mktemp -d)"; trap 'rm -rf "$tmp"' EXIT
printf '      NERDCTL_VERSION=2.3.3\n      x="${NERDCTL_VERSION}"\n' > "$tmp/skel"
set_nerdctl_version "$tmp/skel" 2.5.1
grep -q '^      NERDCTL_VERSION=2.5.1$' "$tmp/skel" || fail "nerdctl rewrite"
grep -q 'x="${NERDCTL_VERSION}"' "$tmp/skel" || fail "nerdctl rewrite touched other lines"

line="$(report_line vergil-vm nerdctl 2.5.1 3.0.0 'newer major')"
[ "$line" = '{"source": "vergil-vm", "name": "nerdctl", "current": "2.5.1", "latest": "3.0.0", "note": "newer major"}' ] \
  || fail "report_line: $line"
line="$(report_line vergil-vm lima 2.0.0 3.1.0)"
[ "$line" = '{"source": "vergil-vm", "name": "lima", "current": "2.0.0", "latest": "3.1.0"}' ] \
  || fail "report_line without note: $line"
echo "check-update-deps-hook: OK"
```

Run: `bash tests/check-update-deps-hook.sh`
Expected: FAIL (the library does not exist).

- [ ] **Step 4: Write `scripts/dev/update-deps-lib.sh`**

```bash
#!/usr/bin/env bash
# scripts/dev/update-deps-lib.sh — pure helpers for scripts/dev/update-deps.sh.
# Sourced, never executed. No network, no tofu.

# semver_major <x.y.z> -> x
semver_major() { printf '%s\n' "${1%%.*}"; }

# semver_newest_same_major <current> <candidate>... -> newest stable x.y.z sharing
# current's major (current itself if nothing newer). Pre-releases are ignored.
semver_newest_same_major() {
  local cur="$1"; shift
  local major; major="$(semver_major "$cur")"
  printf '%s\n' "$cur" "$@" \
    | grep -E "^${major}\.[0-9]+\.[0-9]+$" \
    | sort -t. -k1,1n -k2,2n -k3,3n | tail -1
}

# set_nerdctl_version <file> <version> — rewrite the NERDCTL_VERSION= assignment only.
set_nerdctl_version() {
  sed -i.bak -E "s/^([[:space:]]*NERDCTL_VERSION=)[0-9]+\.[0-9]+\.[0-9]+$/\1$2/" "$1"
  rm -f "$1.bak"
}

# report_line <source> <name> <current> <latest> [note] — one held-back JSON line.
# Values are fixed version strings and names from this hook; none contain quotes.
report_line() {
  if [ -n "${5:-}" ]; then
    printf '{"source": "%s", "name": "%s", "current": "%s", "latest": "%s", "note": "%s"}\n' \
      "$1" "$2" "$3" "$4" "$5"
  else
    printf '{"source": "%s", "name": "%s", "current": "%s", "latest": "%s"}\n' "$1" "$2" "$3" "$4"
  fi
}
```

Run: `bash tests/check-update-deps-hook.sh`
Expected: `check-update-deps-hook: OK`.

- [ ] **Step 5: Write the hook `scripts/dev/update-deps.sh`**

```bash
#!/usr/bin/env bash
# scripts/dev/update-deps.sh — vergil-vm's dependency-update hook
# (epic vergil-project/.github#381, spec §6.3). Run by vrg-update-deps inside the
# dev container from the worktree root, with $VRG_DEPS_REPORT set. Edits files
# only; never commits. Any failure exits non-zero (no silent "no change").
set -euo pipefail
: "${VRG_DEPS_REPORT:?set by vrg-update-deps (or export it for a manual run)}"
ROOT="$(cd "$(dirname "$0")/../.." && pwd)"
# shellcheck source=update-deps-lib.sh
. "${ROOT}/scripts/dev/update-deps-lib.sh"
PLATFORMS=(linux_amd64 linux_arm64 darwin_amd64 darwin_arm64)  # set from Step 2

latest_github_release() { # owner/repo -> tag without the leading v
  local loc
  loc="$(curl -fsSI "https://github.com/$1/releases/latest" | tr -d '\r' \
    | sed -n 's/^[Ll]ocation: .*\/tag\/v\{0,1\}//p')"
  [ -n "$loc" ] || { echo "update-deps: cannot resolve latest release of $1" >&2; exit 1; }
  printf '%s\n' "$loc"
}

# 1. OpenTofu providers, within the existing ~> ranges.
for m in "${ROOT}"/opentofu/modules/*/vm "${ROOT}"/opentofu/modules/*/volume; do
  tofu -chdir="$m" init -backend=false -upgrade -input=false >/dev/null
  tofu -chdir="$m" providers lock "${PLATFORMS[@]/#/-platform=}" >/dev/null
  rm -rf "$m/.terraform"
done

# 2. nerdctl in each cloud-init skeleton, within the same major; then regenerate.
skels=("${ROOT}"/opentofu/modules/*/vm/cloud-init.yaml.skel)
current="$(sed -n -E 's/^[[:space:]]*NERDCTL_VERSION=([0-9.]+)$/\1/p' "${skels[0]}")"
[ -n "$current" ] || { echo "update-deps: NERDCTL_VERSION not found" >&2; exit 1; }
latest="$(latest_github_release containerd/nerdctl)"
target="$(semver_newest_same_major "$current" "$latest")"
if [ "$target" != "$current" ]; then
  for s in "${skels[@]}"; do set_nerdctl_version "$s" "$target"; done
  bash "${ROOT}/scripts/build-cloud-init.sh"
fi
if [ "$(semver_major "$latest")" != "$(semver_major "$current")" ]; then
  report_line vergil-vm nerdctl "$target" "$latest" "newer major; never bumped automatically" \
    >> "$VRG_DEPS_REPORT"
fi

# 3. Report-only: Lima major and NodeSource major.
lima_min="$(sed -n -E 's/^minimumLimaVersion: "([0-9.]+)"$/\1/p' "${ROOT}/templates/agent.yaml.skel")"
lima_latest="$(latest_github_release lima-vm/lima)"
if [ "$(semver_major "$lima_latest")" -gt "$(semver_major "$lima_min")" ]; then
  report_line vergil-vm lima "$lima_min" "$lima_latest" "minimumLimaVersion in templates/agent.yaml.skel" \
    >> "$VRG_DEPS_REPORT"
fi
node_cur="$(sed -n -E 's/.*deb\.nodesource\.com\/setup_([0-9]+)\.x.*/\1/p' \
  "${ROOT}/templates/provision/10-base.sh" | head -1)"
node_lts="$(curl -fsS https://nodejs.org/dist/index.json \
  | python3 -c 'import json,sys; print(next(r["version"].lstrip("v").split(".")[0] for r in json.load(sys.stdin) if r["lts"]))')"
if [ "$node_lts" -gt "$node_cur" ]; then
  report_line vergil-vm nodesource "$node_cur" "$node_lts" "newest Node LTS major" >> "$VRG_DEPS_REPORT"
fi
```

Then `chmod +x scripts/dev/update-deps.sh`. If `python3` is not in the dev
container (check with `vrg-container-run -- python3 --version`), parse
`index.json` with `jq` instead (the CI images ship `jq`; check with
`vrg-container-run -- jq --version`).

- [ ] **Step 6: Wire the hook and the check**

Add to `vergil.toml`:

```toml
[dependency-update]
hook = "scripts/dev/update-deps.sh"
```

In `tests/run-host-tests.sh`, add `check-update-deps-hook` to the `for t in …`
list.

- [ ] **Step 7: Live dry run in the container**

```bash
vrg-container-run -- env VRG_DEPS_REPORT=/workspace/.vergil/deps-report.jsonl scripts/dev/update-deps.sh
cat .vergil/deps-report.jsonl
vrg-git status --porcelain
bash tests/run-host-tests.sh
```

Expected: exit 0, and every report line is valid JSON. `git status` shows only
lockfile, skeleton or `cloud-init.yaml` changes, and no `.terraform/`.
`run-host-tests.sh` passes, including the cloud-init freshness check. Then
**discard the dependency edits** (`vrg-git restore opentofu`), because bumping
them is the release stage's job, not this PR's. Remove the report file.

- [ ] **Step 8: Remove the pin from `vrg-vm-init.sh`**

Replace the hard-coded install near line 109:

```bash
VERGIL_SERIES="$(sed -n -E 's/^vergil = "(v[0-9]+\.[0-9]+)"$/\1/p' "${REPO_ROOT}/vergil.toml")"
[ -n "$VERGIL_SERIES" ] || { echo "vrg-vm-init: [dependencies].vergil not found in vergil.toml" >&2; exit 1; }
# shellcheck disable=SC2016
limactl shell "$INSTANCE" -- bash -c \
    "export PATH=\"\$HOME/.local/bin:\$PATH\" && uv tool install \"vergil-tooling @ git+https://github.com/vergil-project/vergil-tooling@${VERGIL_SERIES}\"" \
    2>&1 | tail -3
```

Use whatever variable the script already uses for its repo root. Look near
the top of `scripts/vrg-vm-init.sh`, and if there is none, define
`REPO_ROOT="$(cd "$(dirname "$0")/.." && pwd)"` there.

- [ ] **Step 9: Validate, then commit**

Run: `vrg-container-run -- vrg-validate`. Expected: PASS, with shellcheck clean
and no unrecognized-key warning for `[dependency-update]`, since Task 9's
tooling is deployed.

```bash
vrg-git add scripts/dev tests/check-update-deps-hook.sh tests/run-host-tests.sh vergil.toml scripts/vrg-vm-init.sh
vrg-commit --type feat --scope deps \
  --message "add the dependency-update hook and read the vergil series in vrg-vm-init"
```

---

### Task 11: vergil-python hook [`vergil-python`]

Covers spec §6.3 (vergil-python) and §7.3 (the vergil-python hook row).

**Blocked by:** Task 9.

**Files:**

- Create: `scripts/dev/update_deps.py` (stdlib-only logic)
- Create: `scripts/dev/update-deps.sh` (the hook entry point)
- Create: `tests/test_update_deps.py` (stdlib `unittest`)
- Create: `tests/check-update-deps.sh` (runs the unit tests)
- Modify: `vergil.toml` (`[dependency-update]`)

**Interfaces:**

- Consumes: the hook contract (spec §6.2); `runtime.toml` and
  `vergil.toml [package]` as read by `packaging/build.sh`.
- Produces (in `update_deps.py`):
  `parse_sums(text: str, tag: str) -> dict[str, dict[str, str]]` (CPython
  version to `{"x86_64": sha, "aarch64": sha}`, complete pairs only),
  `pick(current: str, versions: Iterable[str]) -> tuple[str, str | None]`
  (the target patch in the current minor, and a newer stable minor if any),
  `render_runtime(text: str, cpython: str, tag: str, sums: dict[str, str]) -> str`,
  `rewrite_package(text: str, old_cpython: str, old_tag: str, new_cpython: str, new_tag: str) -> str`,
  `main(fetch=…) -> int`.

- [ ] **Step 1: Failing unit tests**

```python
# tests/test_update_deps.py — stdlib unittest for scripts/dev/update_deps.py.
import os
import sys
import tempfile
import unittest
from pathlib import Path

sys.path.insert(0, str(Path(__file__).resolve().parents[1] / "scripts" / "dev"))
import update_deps as ud  # noqa: E402

_TAG = "20261101"
_SUMS = "\n".join([
    f"aaa  cpython-3.14.9+{_TAG}-x86_64-unknown-linux-gnu-install_only_stripped.tar.gz",
    f"bbb  cpython-3.14.9+{_TAG}-aarch64-unknown-linux-gnu-install_only_stripped.tar.gz",
    f"ccc  cpython-3.15.1+{_TAG}-x86_64-unknown-linux-gnu-install_only_stripped.tar.gz",
    f"ddd  cpython-3.15.1+{_TAG}-aarch64-unknown-linux-gnu-install_only_stripped.tar.gz",
    f"eee  cpython-3.16.0a1+{_TAG}-x86_64-unknown-linux-gnu-install_only_stripped.tar.gz",
    f"fff  cpython-3.13.12+{_TAG}-x86_64-unknown-linux-gnu-install_only_stripped.tar.gz",
    f"ggg  cpython-3.14.9+{_TAG}-x86_64-unknown-linux-gnu-install_only.tar.gz",
])
_RUNTIME = '# header\n\ncpython = "3.14.8"\npbs_tag = "20261003"\n\n[sha256]\nx86_64 = "old1"\naarch64 = "old2"\n'
_VERGIL = ('[dependencies]\nvergil = "v2.1"\n\n[package]\nbuilder = "staged"\n'
           'name = "vergil-python3.14.8"\nversion = "3.14.8+20261003"\n'
           'summary = "Pinned CPython 3.14.8 (python-build-standalone 20261003) for vergil products"\n'
           "smoke = \"/opt/vergil/python/3.14.8/bin/python3.14 -c 'import ssl'\"\n\n"
           '[package.staged]\nbuild-command = "packaging/build.sh"\n')


class UpdateDepsTest(unittest.TestCase):
    def test_parse_sums_complete_pairs_stable_only(self):
        sums = ud.parse_sums(_SUMS, _TAG)
        self.assertEqual(sums["3.14.9"], {"x86_64": "aaa", "aarch64": "bbb"})
        self.assertIn("3.15.1", sums)
        self.assertNotIn("3.16.0a1", sums)
        self.assertNotIn("3.13.12", sums)  # only one arch

    def test_pick_patch_and_newer_minor(self):
        self.assertEqual(ud.pick("3.14.8", ["3.14.9", "3.15.1"]), ("3.14.9", "3.15.1"))
        self.assertEqual(ud.pick("3.14.9", ["3.14.9"]), ("3.14.9", None))

    def test_pick_fails_when_current_minor_missing(self):
        with self.assertRaises(SystemExit):
            ud.pick("3.14.8", ["3.15.1"])

    def test_render_runtime_keeps_comments(self):
        out = ud.render_runtime(_RUNTIME, "3.14.9", _TAG, {"x86_64": "aaa", "aarch64": "bbb"})
        self.assertTrue(out.startswith("# header\n"))
        self.assertIn('cpython = "3.14.9"', out)
        self.assertIn(f'pbs_tag = "{_TAG}"', out)
        self.assertIn('x86_64 = "aaa"', out)
        self.assertIn('aarch64 = "bbb"', out)

    def test_rewrite_package_only_touches_package_table(self):
        out = ud.rewrite_package(_VERGIL, "3.14.8", "20261003", "3.14.9", _TAG)
        self.assertIn('name = "vergil-python3.14.9"', out)
        self.assertIn(f'version = "3.14.9+{_TAG}"', out)
        self.assertIn(f"CPython 3.14.9 (python-build-standalone {_TAG})", out)
        self.assertIn("/opt/vergil/python/3.14.9/bin/python3.14", out)
        self.assertIn('vergil = "v2.1"', out)

    def test_main_end_to_end_with_fake_fetch(self):
        with tempfile.TemporaryDirectory() as d:
            root = Path(d)
            (root / "runtime.toml").write_text(_RUNTIME)
            (root / "vergil.toml").write_text(_VERGIL)
            report = root / "report.jsonl"
            report.write_text("")

            def fetch(url):
                if url.endswith("/releases/latest"):
                    return f"https://github.com/astral-sh/python-build-standalone/releases/tag/{_TAG}"
                return _SUMS

            os.environ["VRG_DEPS_REPORT"] = str(report)
            self.assertEqual(ud.main(root=root, fetch=fetch), 0)
            self.assertIn('cpython = "3.14.9"', (root / "runtime.toml").read_text())
            self.assertIn("vergil-python3.14.9", (root / "vergil.toml").read_text())
            self.assertIn('"latest": "3.15.1"', report.read_text())


if __name__ == "__main__":
    unittest.main()
```

`tests/check-update-deps.sh`:

```bash
#!/usr/bin/env bash
# tests/check-update-deps.sh — offline unit tests for the dependency-update hook.
set -euo pipefail
ROOT="$(cd "$(dirname "$0")/.." && pwd)"
uv run --no-project --quiet python3 -m unittest discover -s "${ROOT}/tests" -p 'test_update_deps.py' -v
```

Run: `vrg-container-run -- bash tests/check-update-deps.sh`
Expected: FAIL (`ModuleNotFoundError: update_deps`).

- [ ] **Step 2: Write `scripts/dev/update_deps.py`**

```python
"""vergil-python's dependency-update logic (epic vergil-project/.github#381, spec §6.3).

Resolves the newest python-build-standalone (PBS) release, takes the newest
CPython patch of the *current* minor it ships, and rewrites runtime.toml and
vergil.toml [package] together (packaging/build.sh fails if they disagree). A
newer stable CPython minor is reported, never bumped. Stdlib only; any failure
exits non-zero.
"""

from __future__ import annotations

import json
import os
import re
import sys
import tomllib
import urllib.request
from collections.abc import Callable, Iterable
from pathlib import Path

_PBS = "astral-sh/python-build-standalone"
_ARCHES = {"x86_64": "x86_64-unknown-linux-gnu", "aarch64": "aarch64-unknown-linux-gnu"}
_ASSET_RE = re.compile(
    r"^([0-9a-f]+)\s+cpython-(\d+\.\d+\.\d+)\+(\d+)-(\S+)-install_only_stripped\.tar\.gz$"
)

Fetch = Callable[[str], str]


def _key(v: str) -> tuple[int, ...]:
    return tuple(int(p) for p in v.split("."))


def _die(msg: str) -> None:
    print(f"update-deps: {msg}", file=sys.stderr)
    raise SystemExit(1)


def parse_sums(text: str, tag: str) -> dict[str, dict[str, str]]:
    by_version: dict[str, dict[str, str]] = {}
    triples = {v: k for k, v in _ARCHES.items()}
    for line in text.splitlines():
        m = _ASSET_RE.match(line.strip())
        if m is None or m[3] != tag or m[4] not in triples:
            continue
        by_version.setdefault(m[2], {})[triples[m[4]]] = m[1]
    return {v: s for v, s in by_version.items() if set(s) == set(_ARCHES)}


def pick(current: str, versions: Iterable[str]) -> tuple[str, str | None]:
    versions = list(versions)
    same = [v for v in versions if _key(v)[:2] == _key(current)[:2]]
    if not same:
        _die(f"PBS release ships no CPython {'.'.join(current.split('.')[:2])}.x")
    newer = [v for v in versions if _key(v)[:2] > _key(current)[:2]]
    return max(same, key=_key), (max(newer, key=_key) if newer else None)


def render_runtime(text: str, cpython: str, tag: str, sums: dict[str, str]) -> str:
    out = re.sub(r'^cpython = "[^"]*"$', f'cpython = "{cpython}"', text, count=1, flags=re.M)
    out = re.sub(r'^pbs_tag = "[^"]*"$', f'pbs_tag = "{tag}"', out, count=1, flags=re.M)
    for arch, sha in sums.items():
        out = re.sub(rf'^{arch} = "[^"]*"$', f'{arch} = "{sha}"', out, count=1, flags=re.M)
    return out


def rewrite_package(text: str, old_cpython: str, old_tag: str, new_cpython: str, new_tag: str) -> str:
    start = text.index("\n[package]\n") + 1
    end_match = re.compile(r"^\[(?!package\]$)", re.M).search(text, start + len("[package]\n"))
    end = end_match.start() if end_match else len(text)
    block = text[start:end].replace(old_cpython, new_cpython).replace(old_tag, new_tag)
    return text[:start] + block + text[end:]


def _fetch(url: str) -> str:
    req = urllib.request.Request(url, method="GET")
    with urllib.request.urlopen(req, timeout=30) as resp:  # noqa: S310
        if url.endswith("/releases/latest"):
            return resp.geturl()  # the redirect target names the tag
        return resp.read().decode()


def main(*, root: Path, fetch: Fetch = _fetch) -> int:
    report_path = os.environ.get("VRG_DEPS_REPORT")
    if not report_path:
        _die("VRG_DEPS_REPORT is not set (vrg-update-deps sets it)")
    runtime_text = (root / "runtime.toml").read_text()
    runtime = tomllib.loads(runtime_text)
    current, old_tag = runtime["cpython"], runtime["pbs_tag"]
    final = fetch(f"https://github.com/{_PBS}/releases/latest")
    tag = final.rstrip("/").rsplit("/", 1)[-1]
    if not re.fullmatch(r"\d{8}", tag):
        _die(f"unexpected latest PBS tag {tag!r} from {final}")
    sums = parse_sums(fetch(f"https://github.com/{_PBS}/releases/download/{tag}/SHA256SUMS"), tag)
    target, minor = pick(current, sums)
    if (target, tag) != (current, old_tag):
        (root / "runtime.toml").write_text(render_runtime(runtime_text, target, tag, sums[target]))
        vergil = (root / "vergil.toml").read_text()
        (root / "vergil.toml").write_text(rewrite_package(vergil, current, old_tag, target, tag))
    if minor is not None:
        with open(report_path, "a", encoding="utf-8") as fh:
            fh.write(json.dumps({"source": "vergil-python", "name": "cpython", "current": target,
                                 "latest": minor, "note": "new CPython minor; a judgement call"}) + "\n")
    return 0


if __name__ == "__main__":
    sys.exit(main(root=Path(__file__).resolve().parents[2]))
```

`rewrite_package` assumes `[package]` is not the first line of `vergil.toml`.
That holds today, because `[project]` comes first. The test pins it.

- [ ] **Step 3: Run the unit tests. Expect PASS**

Run: `vrg-container-run -- bash tests/check-update-deps.sh`
Expected: PASS.

- [ ] **Step 4: Hook entry point and config**

`scripts/dev/update-deps.sh`:

```bash
#!/usr/bin/env bash
# scripts/dev/update-deps.sh — vergil-python's dependency-update hook (epic .github#381).
set -euo pipefail
ROOT="$(cd "$(dirname "$0")/../.." && pwd)"
exec uv run --no-project --quiet python3 "${ROOT}/scripts/dev/update_deps.py"
```

Then `chmod +x scripts/dev/update-deps.sh`. Add to `vergil.toml`:

```toml
[dependency-update]
hook = "scripts/dev/update-deps.sh"
```

- [ ] **Step 5: Live dry run**

```bash
vrg-container-run -- env VRG_DEPS_REPORT=/workspace/.vergil/deps-report.jsonl scripts/dev/update-deps.sh
vrg-git diff --stat
```

Expected: exit 0. Either no diff (already newest), or a consistent change to
`runtime.toml` and `vergil.toml`. If there is a diff, check that
`packaging/build.sh`'s name and version guard would pass: run
`VRG_STAGING_ROOT=$(mktemp -d) VRG_TARGET_ARCH=amd64 vrg-container-run -- packaging/build.sh`
and expect `staged CPython …`. Then discard the dependency edits
(`vrg-git restore runtime.toml vergil.toml`), keeping only the
`[dependency-update]` addition. Re-apply that addition if the restore removed
it.

- [ ] **Step 6: Validate, then commit**

Run: `vrg-container-run -- vrg-validate`. Expected: PASS.

```bash
vrg-git add scripts/dev tests/test_update_deps.py tests/check-update-deps.sh vergil.toml
vrg-commit --type feat --scope deps \
  --message "add the dependency-update hook for the PBS runtime pin"
```

---

### Task 12: Validation: a live release cycle with the dependency-update stage [`vergil-tooling`, `--kind validation`]

**Blocked by:** Tasks 10 and 11.

This is an operational task, run with `issue-validate`. Releases are human
actions, so the agent records results the human attests, and checks what is
visible on GitHub. Covers the end-to-end bullet of spec §12.

- **Precondition (probe):** Task 9 is closed `SUCCESS`; the `dependency-update`
  label exists in all three repos.
- **Procedure:**
  1. In **vergil-tooling**, the human runs `vrg-release`. The agent checks the
     release tracking issue. It needs an `update-deps` completion (or failure)
     comment carrying a "Held back by declared bounds" section. There must also
     be either a merged `deps/update-<next>` PR labelled `dependency-update`, or
     the line "No dependency changes; no PR opened."
  2. Same in **vergil-vm**. The PR, if any, contains `chore(deps): repo hook`,
     and the report shows any Lima, NodeSource or nerdctl entries the hook
     emitted.
  3. Same in **vergil-python**. If a newer PBS existed, the PR changes
     `runtime.toml` and `vergil.toml` together, and its CI (the package build)
     is green.
  4. **Gate trip:** in one repo, the human opens a throwaway PR labelled
     `dependency-update`, then runs `vrg-release`. Preflight must refuse,
     naming the PR. Then `vrg-release --allow-pending-deps` must proceed, and
     the prepare comment on the tracking issue must name the bypassed PR. The
     human closes the throwaway PR.
- **Acceptance:** every step above observed, with links, recorded as
  `Outcome: SUCCESS`. Any deviation is `FAILURE` with the evidence, and the
  task stays open.

---

## Task filing map (epic-create step 9)

| Plan task | Repo | Kind | Blocked by |
|---|---|---|---|
| 1 Report, config, hook runner | vergil-tooling | task | – |
| 2 Driver failure handling and `run_update()` | vergil-tooling | task | 1 |
| 3 `update-deps` release stage | vergil-tooling | task | 2 |
| 4 Release gate | vergil-tooling | task | 2 |
| 5 Python held-back report | vergil-tooling | task | 1 |
| 6 `vergil` updater: action.yml and runtime | vergil-tooling | task | 1 |
| 7 `repo_init.py` pin removal | vergil-tooling | task | – |
| 8 `images` updater | vergil-tooling | task | 1 |
| 9 Deploy | vergil-tooling | deployment | 1–8 |
| 10 vergil-vm hook | vergil-vm | task | 9 |
| 11 vergil-python hook | vergil-python | task | 9 |
| 12 Live release validation | vergil-tooling | validation | 10, 11 |

Tasks 2, 5, 6 and 8 each touch `orchestrator.py` or a shared test file.
Implement them in table order, or rebase onto the previous one, to avoid merge
conflicts. Tasks 3 and 4 both edit `release/orchestrator.py`, `release/context.py`
and `vrg_release.py`, so land 3 before 4.
