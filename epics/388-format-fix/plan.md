# Opt-in formatter fix mode Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add an opt-in `vrg-validate --fix-format` mode that runs every audited
formatter in write mode before the checks, with hints that teach it, and add
rumdl to the dev containers so Markdown line length can be fixed too.

**Architecture:** Each formatter is declared once as a `Formatter` record in the
language registry (`lib/languages.py`). A pure formatter's `check` feeds the
LINT stage; its `fix` feeds a new pre-check fix stage. Mixed tools get fix-only
companions that drive a soft hint. A new `lib/format_fix.py` runs fixers and
reports which files they changed. The driver (`bin/vrg_validate.py`) orders
all fix stages before all check stages when the flag is given.

**Tech Stack:** Python 3.12+ (vergil-tooling), pytest, Docker (vergil-containers),
rumdl (Rust Markdown linter/formatter, installed with npm).

**Spec:** `epics/388-format-fix/spec.md` (this directory).

## Global Constraints

- Plain `vrg-validate` (no flag) never modifies files; its stage order stays
  `common -> install -> lint -> typecheck -> test -> audit -> custom`.
- `--fix-format` never stages or commits.
- `--fix-format` with `CI` or `GITHUB_ACTIONS` set: exit 2.
- `--fix-format` with `--check typecheck|test|audit`: exit 2.
- With `--fix-format`, order is
  `install -> format-common -> format-lint -> common -> lint -> typecheck -> test -> audit -> custom`.
- Fix stages are `fail_defer`; a fixer failure is never hidden.
- The LINT command multiset for every language is unchanged, except Java's
  split of `spotless:check checkstyle:check` into two commands.
- No change to `vergil-actions`, CI evidence reports or branch protection.
- Formatters run only in the dev container.
- All validation runs through `vrg-container-run -- vrg-validate` (in this repo,
  transparently `uv run vrg-validate`). Run tests with
  `vrg-container-run -- vrg-validate --check test`. Do not run pytest, ruff or
  other tools directly.
- Commits use `vrg-commit`; git uses `vrg-git`. Each task is its own issue,
  worktree and PR (`issue-implement`).
- Firm hint text, exactly:
  `formatting failures can be fixed with: vrg-container-run -- vrg-validate --fix-format`
- Soft hint text, exactly:
  `<gate command> failed; some of its findings may be auto-fixable with: vrg-container-run -- vrg-validate --fix-format`

## Review Focus

1. **A tracked file deleted in the working tree** is listed by
   `git ls-files --cached` but cannot be read. The snapshot must record it as
   absent, not crash or report it as rewritten. The test is in Task 4.
2. **No Markdown files in scope** (no `docs/site/`, no `README.md`, no
   `epics/`). `markdownlint --fix` and `rumdl fmt` with no paths must never be
   invoked: markdownlint with no file arguments reads stdin, and rumdl formats
   the current directory. The tests are in Task 4 and Task 6.
3. **A file that was already modified** before the fix (dirty tree) and is then
   rewritten by a formatter must still appear in the change report. The test is
   in Task 4.
4. **A repo with no `primary-language`** (only common checks). `--fix-format`
   runs only `format-common`, with no install and no `format-lint`. The test is
   in Task 4.
5. **A fixer creates a file that did not exist before** (a path appears in the
   after-snapshot only). It counts as changed. The test is in Task 4.

---

## Task map

| # | Repo | Kind | Title | Blocked by |
|---|---|---|---|---|
| 1 | vergil-containers | task | Install rumdl in the dev images | — |
| 2 | vergil-containers | deployment | Publish dev images containing rumdl | 1 |
| 3 | vergil-tooling | task | Formatter registry and LINT derivation | — |
| 4 | vergil-tooling | task | `--fix-format` flag, fix stages and change report | 3 |
| 5 | vergil-tooling | task | Firm and soft formatting hints, plus docs | 4 |
| 6 | vergil-tooling | task | rumdl reflow companion for strict Markdown | 2, 4 |
| 7 | vergil-tooling | task | Ruby and Go fix-only companions (verified) | 4 |
| 8 | vergil-tooling | validation | Cross-language `--fix-format` validation | 5, 6, 7 |

---

### Task 1: Install rumdl in the dev images (vergil-containers)

**Files:**

- Modify: `docker/common/node-markdownlint.dockerfile` (the `npm install -g` line)
- Modify: `README.md` (tool list, next to `markdownlint-cli`)
- Modify: `CLAUDE.md` (shared tooling list, next to `markdownlint-cli`)

**Interfaces:**

- Produces: a `rumdl` executable on `PATH` in every dev image that includes
  the common Node layer.

rumdl installs with npm (`npm install -g rumdl`, per the rumdl README). It goes
in the same `npm install -g` step as `markdownlint-cli`. Under this repo's pin
policy (`docker/pins/pins.yml` header), package-manager tools float, so no
version and no `pins.yml` entry are added.

- [ ] **Step 1: Add rumdl to the npm install line**

```dockerfile
RUN npm install -g npm@latest && \
    npm install -g markdownlint-cli rumdl && \
    npm cache clean --force
```

- [ ] **Step 2: Regenerate and build one image locally**

Run: `docker/generate.sh && docker/build.sh python`
Expected: the build succeeds.

- [ ] **Step 3: Smoke-check the binary in the built image**

Run: `docker run --rm <built python image tag printed by build.sh> rumdl --version`
Expected: prints `rumdl <version>`.

- [ ] **Step 4: Update the README and CLAUDE.md tool lists**

Add `- **rumdl** — Markdown formatter used by vrg-validate --fix-format for line-length reflow`
next to the `markdownlint-cli` lines.

- [ ] **Step 5: Validate and commit**

Run: `vrg-container-run -- vrg-validate`
Expected: PASS (hadolint covers the dockerfile).

```bash
vrg-git add docker/common/node-markdownlint.dockerfile README.md CLAUDE.md
vrg-commit --type feat --scope images --message "install rumdl alongside markdownlint-cli"
```

---

### Task 2: Publish dev images containing rumdl (deployment, vergil-containers)

Create with `vrg-issue-create --kind deployment --blocked-by vergil-project/vergil-containers#<Task 1>`.

**Precondition (human-attested):** a vergil-containers release containing
Task 1 has been cut and its images published. The agent never cuts the release.

**Procedure:**

1. From a vergil-tooling checkout, pull the current dev image and run:
   `vrg-container-run -- rumdl --version`
2. Expected: prints a version. If `rumdl: command not found`, the published
   image predates Task 1: record `Outcome: FAILURE` and stop.

**Acceptance:** `rumdl --version` succeeds inside vergil-tooling's dev
container. Record `Outcome: SUCCESS` with the version output.

---

### Task 3: Formatter registry and LINT derivation (vergil-tooling)

**Files:**

- Modify: `src/vergil_tooling/lib/languages.py` (add `Formatter`, `Language.formatters`, per-language formatter tuples, LINT derivation, `language_formatters()`, C++ clang-format helper, shared placeholder expansion)
- Test: `tests/vergil_tooling/test_languages.py`

**Interfaces:**

- Produces:
  - `languages.Formatter` (frozen dataclass): `name: str`, `fix: list[str]`,
    `check: list[str] | None = None`, `soft_hint_for: list[str] | None = None`.
    `soft_hint_for` is the argv of the mixed gate command whose failure
    triggers the soft hint. (The spec's sketch typed it `str`; an argv list is
    what the driver compares against failing commands.)
  - `Language.formatters: tuple[Formatter, ...] = ()`
  - `languages.language_formatters(language: str | None, *, cpp_std: str | None = None, cpp_stdlib: str | None = None) -> list[Formatter]`,
    with `check`, `fix` and `soft_hint_for` placeholder-expanded exactly like
    `language_commands`. It returns `[]` for an unknown or `None` language.

- [ ] **Step 1: Write the failing tests**

Add to `tests/vergil_tooling/test_languages.py`:

```python
from vergil_tooling.lib.languages import Formatter, language_formatters

# The LINT multiset before epic #388, with Java's single command split in two.
_EXPECTED_LINT_CHANGES = {
    "java": (
        [["./mvnw", "spotless:check", "checkstyle:check", "-B"]],
        [["./mvnw", "spotless:check", "-B"], ["./mvnw", "checkstyle:check", "-B"]],
    ),
}


@pytest.mark.parametrize("lang", sorted(supported_languages()))
def test_every_formatter_check_is_in_lint(lang: str) -> None:
    lint = language_commands(lang, CheckKind.LINT)
    for fmt in language_formatters(lang):
        if fmt.check is not None:
            assert fmt.check in lint, f"{lang}: {fmt.name} check missing from LINT"


@pytest.mark.parametrize("lang", sorted(supported_languages()))
def test_formatter_checks_run_first_in_lint(lang: str) -> None:
    lint = language_commands(lang, CheckKind.LINT)
    checks = [f.check for f in language_formatters(lang) if f.check is not None]
    assert lint[: len(checks)] == checks


@pytest.mark.parametrize("lang", sorted(supported_languages()))
def test_every_formatter_has_a_fix_and_valid_hint_target(lang: str) -> None:
    lint = language_commands(lang, CheckKind.LINT)
    for fmt in language_formatters(lang):
        assert fmt.fix, f"{lang}: {fmt.name} has an empty fix"
        if fmt.check is None:
            assert fmt.soft_hint_for in lint, f"{lang}: {fmt.name} names no LINT gate"


def test_python_formatters() -> None:
    fmts = {f.name: f for f in language_formatters("python")}
    assert fmts["ruff format"].check == ["ruff", "format", "--check", "src/", "tests/"]
    assert fmts["ruff format"].fix == ["ruff", "format", "src/", "tests/"]
    sort = fmts["ruff import sort"]
    assert sort.check is None
    assert sort.fix == ["ruff", "check", "--select", "I", "--fix", "src/", "tests/"]
    assert sort.soft_hint_for == ["ruff", "check", "src/", "tests/"]


def test_java_lint_is_split() -> None:
    assert language_commands("java", CheckKind.LINT) == [
        ["./mvnw", "spotless:check", "-B"],
        ["./mvnw", "checkstyle:check", "-B"],
    ]
    (spotless,) = language_formatters("java")
    assert spotless.fix == ["./mvnw", "spotless:apply", "-B"]


def test_rust_and_typescript_formatters() -> None:
    (cargo,) = language_formatters("rust")
    assert cargo.check == ["cargo", "fmt", "--all", "--", "--check"]
    assert cargo.fix == ["cargo", "fmt", "--all"]
    (prettier,) = language_formatters("typescript")
    assert prettier.check is not None
    assert prettier.check[:3] == ["prettier", "--check", "."]
    assert prettier.fix[:3] == ["prettier", "--write", "."]
    assert prettier.check[3:] == prettier.fix[3:]  # same --config, expanded


def test_cpp_clang_format_check_and_fix_walk_same_files() -> None:
    (clang,) = language_formatters("cpp")
    assert clang.check is not None
    check_script, fix_script = clang.check[2], clang.fix[2]
    assert "--dry-run --Werror" in check_script
    assert " -i " in fix_script
    assert check_script.split("| xargs")[0] == fix_script.split("| xargs")[0]
    assert "{configs}" not in fix_script


def test_language_formatters_unknown_language() -> None:
    assert language_formatters(None) == []
    assert language_formatters("cobol") == []
```

Also replace any existing assertions on the Python and Java LINT lists that
encode the old order or the old Java argv, so they match the new order:
format check first for Python, and the split for Java.

- [ ] **Step 2: Run the tests to verify they fail**

Run: `vrg-container-run -- vrg-validate --check test`
Expected: FAIL with `ImportError: cannot import name 'Formatter'`.

- [ ] **Step 3: Implement the record, the expansion helper and the derivation**

In `languages.py`, after `EcosystemInfo`:

```python
@dataclass(frozen=True)
class Formatter:
    """A formatter declared once: its gate check (if pure) and its write-mode fix.

    ``check`` is the pure gate argv, which also runs in LINT. ``None`` marks a
    fix-only companion to a mixed gate (one that also reports findings no
    formatter can fix). ``soft_hint_for`` is that mixed gate's argv, so a failure
    of it prints the soft ``--fix-format`` hint (epic vergil-project/.github#388).
    """

    name: str
    fix: list[str]
    check: list[str] | None = None
    soft_hint_for: list[str] | None = None


def _format_checks(formatters: tuple[Formatter, ...]) -> list[list[str]]:
    return [f.check for f in formatters if f.check is not None]
```

Add to `Language`: `formatters: tuple[Formatter, ...] = ()`.

Define the per-language tuples above `_REGISTRY` and build LINT from them:

```python
_PYTHON_FORMATTERS = (
    Formatter(
        name="ruff format",
        check=["ruff", "format", "--check", "src/", "tests/"],
        fix=["ruff", "format", "src/", "tests/"],
    ),
    Formatter(
        name="ruff import sort",
        fix=["ruff", "check", "--select", "I", "--fix", "src/", "tests/"],
        soft_hint_for=["ruff", "check", "src/", "tests/"],
    ),
)

_PRETTIER_CONFIG = "{configs}/typescript/prettier.config.json"
_TYPESCRIPT_FORMATTERS = (
    Formatter(
        name="prettier",
        check=["prettier", "--check", ".", "--config", _PRETTIER_CONFIG],
        fix=["prettier", "--write", ".", "--config", _PRETTIER_CONFIG],
    ),
)

_RUST_FORMATTERS = (
    Formatter(
        name="cargo fmt",
        check=["cargo", "fmt", "--all", "--", "--check"],
        fix=["cargo", "fmt", "--all"],
    ),
)

_JAVA_FORMATTERS = (
    Formatter(
        name="spotless",
        check=["./mvnw", "spotless:check", "-B"],
        fix=["./mvnw", "spotless:apply", "-B"],
    ),
)


def _cpp_clang_format(mode: str) -> list[str]:
    """clang-format over the C++ tree; check and fix share one file walk."""
    return [
        "sh",
        "-c",
        "find . -type d \\( -name build -o -name build-sanitize "
        "-o -name .worktrees \\) -prune -o "
        '-type f \\( -name "*.cpp" -o -name "*.cxx" -o -name "*.cc" '
        '-o -name "*.hpp" -o -name "*.hxx" -o -name "*.hh" '
        '-o -name "*.h" \\) -print0 '
        f"| xargs -0 -r clang-format {mode} "
        "--style=file:{configs}/cpp/.clang-format",
    ]


_CPP_FORMATTERS = (
    Formatter(
        name="clang-format",
        check=_cpp_clang_format("--dry-run --Werror"),
        fix=_cpp_clang_format("-i"),
    ),
)
```

Note: the C++ string mixes an f-string segment with a literal `{configs}`
segment. Only the `f"| xargs …{mode} "` piece is an f-string, so `{configs}`
stays a placeholder.

Registry edits:

- python: `formatters=_PYTHON_FORMATTERS`; LINT
  `[*_format_checks(_PYTHON_FORMATTERS), ["ruff", "check", "src/", "tests/"], [ruff json report argv unchanged]]`.
- typescript: `formatters=_TYPESCRIPT_FORMATTERS`; LINT
  `[*_format_checks(_TYPESCRIPT_FORMATTERS), [eslint sh -c argv unchanged]]`.
- rust: `formatters=_RUST_FORMATTERS`; LINT
  `[*_format_checks(_RUST_FORMATTERS), ["cargo", "clippy", "--", "-D", "warnings"]]`.
- java: `formatters=_JAVA_FORMATTERS`; LINT
  `[*_format_checks(_JAVA_FORMATTERS), ["./mvnw", "checkstyle:check", "-B"]]`.
- cpp: `formatters=_CPP_FORMATTERS`; LINT
  `[*_format_checks(_CPP_FORMATTERS), <run-clang-tidy argv unchanged>, <cppcheck argv unchanged>]`,
  removing the old inline clang-format `sh -c` entry. Keep its comment block
  above `_cpp_clang_format`.
- ruby, go: unchanged in this task (Task 7).

Refactor placeholder expansion out of `language_commands` so both functions
share it:

```python
def _expander(cpp_std: str | None, cpp_stdlib: str | None) -> Callable[[str], str]:
    from vergil_tooling.lib.config import DEFAULT_CPP_STD, DEFAULT_CPP_STDLIB

    replacements = {
        "{configs}": str(files("vergil_tooling.configs")),
        "{std}": cpp_std if cpp_std is not None else DEFAULT_CPP_STD,
        "{stdlib}": cpp_stdlib if cpp_stdlib is not None else DEFAULT_CPP_STDLIB,
    }

    def _expand(arg: str) -> str:
        for token, value in replacements.items():
            arg = arg.replace(token, value)
        return arg

    return _expand


def language_formatters(
    language: str | None,
    *,
    cpp_std: str | None = None,
    cpp_stdlib: str | None = None,
) -> list[Formatter]:
    """Return a language's formatters with placeholders expanded.

    Expansion matches :func:`language_commands`, so a formatter's ``check`` is
    equal to the LINT argv it appears as. ``[]`` for an unknown/``None`` language.
    """
    entry = _REGISTRY.get(language) if language is not None else None
    if entry is None:
        return []
    expand = _expander(cpp_std, cpp_stdlib)

    def _argv(cmd: list[str] | None) -> list[str] | None:
        return None if cmd is None else [expand(a) for a in cmd]

    return [
        Formatter(
            name=f.name,
            fix=[expand(a) for a in f.fix],
            check=_argv(f.check),
            soft_hint_for=_argv(f.soft_hint_for),
        )
        for f in entry.formatters
    ]
```

`language_commands` keeps its Python-TEST special case, then uses
`expand = _expander(cpp_std, cpp_stdlib)` in place of its inline `_expand`.
Add `Callable` to the `TYPE_CHECKING` imports from `collections.abc`.

- [ ] **Step 4: Run the tests to verify they pass**

Run: `vrg-container-run -- vrg-validate --check test`
Expected: PASS, with 100% coverage.

- [ ] **Step 5: Full validation and commit**

Run: `vrg-container-run -- vrg-validate`
Expected: PASS.

```bash
vrg-git add src/vergil_tooling/lib/languages.py tests/vergil_tooling/test_languages.py
vrg-commit --type feat --scope validate --message "declare formatters once as check/fix pairs in the language registry"
```

---

### Task 4: `--fix-format` flag, fix stages and change report (vergil-tooling)

**Files:**

- Create: `src/vergil_tooling/lib/format_fix.py` (snapshot, fixer runner, report)
- Modify: `src/vergil_tooling/bin/vrg_validate.py` (flag, refusals, stage ordering)
- Modify: `src/vergil_tooling/bin/validate_common.py` (add `markdown_fixers()`)
- Test: `tests/vergil_tooling/test_format_fix.py` (new), `tests/vergil_tooling/test_vrg_validate.py`, `tests/vergil_tooling/test_validate_common.py`

**Interfaces:**

- Consumes: `language_formatters()` and `Formatter` (Task 3).
- Produces:
  - `format_fix.Fixer = tuple[str, list[str]]` (label, argv)
  - `format_fix.snapshot(repo_root: Path) -> dict[str, str | None]`, mapping a
    repo-relative path to its sha256 hex, or `None` if the path is listed but
    absent.
  - `format_fix.run_fixers(fixers: list[Fixer], repo_root: Path) -> FixResult`
  - `format_fix.FixResult` (dataclass): `changed: dict[str, list[str]]`
    (label → sorted paths), `failures: list[str]` (messages).
  - `format_fix.render_report(result: FixResult) -> str`
  - `validate_common.markdown_fixers(repo_root: Path) -> list[Fixer]`
  - `vrg_validate._build_stages(..., fix_format: bool = False)`

- [ ] **Step 1: Write the failing `format_fix` tests**

Create `tests/vergil_tooling/test_format_fix.py`:

```python
"""Tests for vergil_tooling.lib.format_fix."""

from __future__ import annotations

import subprocess
from pathlib import Path
from unittest.mock import patch

import pytest

from vergil_tooling.lib import format_fix
from vergil_tooling.lib.format_fix import FixResult, render_report, run_fixers, snapshot


def _git_repo(tmp_path: Path) -> Path:
    subprocess.run(["git", "init", "-q", str(tmp_path)], check=True)  # noqa: S607
    return tmp_path


def _commit_all(repo: Path) -> None:
    subprocess.run(["git", "-C", str(repo), "add", "-A"], check=True)  # noqa: S607
    subprocess.run(  # noqa: S607
        ["git", "-C", str(repo), "-c", "user.email=t@t", "-c", "user.name=t",
         "commit", "-qm", "init"],
        check=True,
    )


def test_snapshot_hashes_tracked_and_untracked(tmp_path: Path) -> None:
    repo = _git_repo(tmp_path)
    (repo / "a.py").write_text("x = 1\n")
    _commit_all(repo)
    (repo / "new.md").write_text("# hi\n")
    snap = snapshot(repo)
    assert set(snap) == {"a.py", "new.md"}
    assert all(isinstance(v, str) and len(v) == 64 for v in snap.values())


def test_snapshot_records_deleted_tracked_file_as_absent(tmp_path: Path) -> None:
    repo = _git_repo(tmp_path)
    (repo / "gone.py").write_text("x = 1\n")
    _commit_all(repo)
    (repo / "gone.py").unlink()
    assert snapshot(repo) == {"gone.py": None}


def test_snapshot_respects_gitignore(tmp_path: Path) -> None:
    repo = _git_repo(tmp_path)
    (repo / ".gitignore").write_text("build/\n")
    (repo / "build").mkdir()
    (repo / "build" / "out.o").write_text("bin")
    assert "build/out.o" not in snapshot(repo)


def test_snapshot_listing_failure_raises(tmp_path: Path) -> None:
    with pytest.raises(format_fix.FormatFixError, match="git ls-files"):
        snapshot(tmp_path)  # not a git repo


def test_run_fixers_attributes_changes_per_fixer(tmp_path: Path) -> None:
    repo = _git_repo(tmp_path)
    (repo / "a.py").write_text("x=1\n")
    (repo / "b.md").write_text("# b\n")
    _commit_all(repo)
    (repo / "a.py").write_text("x=2\n")  # already dirty before fixing

    def fake_run(cmd: list[str], *, check: bool) -> int:  # noqa: ARG001
        if cmd[0] == "fmt-py":
            (repo / "a.py").write_text("x = 2\n")
        if cmd[0] == "fmt-md":
            (repo / "created.md").write_text("new\n")
        return 0

    with patch("vergil_tooling.lib.format_fix.progress.run", side_effect=fake_run):
        result = run_fixers([("py", ["fmt-py"]), ("md", ["fmt-md"])], repo)
    assert result.changed == {"py": ["a.py"], "md": ["created.md"]}
    assert result.failures == []


def test_run_fixers_records_nonzero_exit(tmp_path: Path) -> None:
    repo = _git_repo(tmp_path)
    with patch("vergil_tooling.lib.format_fix.progress.run", return_value=1):
        result = run_fixers([("ruff format", ["ruff", "format"])], repo)
    assert result.failures == ["ruff format exited 1"]


def test_run_fixers_missing_binary_names_the_tool(tmp_path: Path) -> None:
    repo = _git_repo(tmp_path)
    with patch(
        "vergil_tooling.lib.format_fix.progress.run",
        side_effect=FileNotFoundError(2, "No such file", "rumdl"),
    ):
        result = run_fixers([("rumdl fmt", ["rumdl", "fmt", "x.md"])], repo)
    assert len(result.failures) == 1
    assert "rumdl" in result.failures[0]
    assert "not found in this dev container" in result.failures[0]


def test_render_report_lists_files_by_fixer() -> None:
    result = FixResult(changed={"ruff format": ["a.py", "b.py"], "md": ["c.md"]}, failures=[])
    assert render_report(result) == (
        "format: rewrote 3 files\n  ruff format: a.py, b.py\n  md: c.md"
    )


def test_render_report_no_changes() -> None:
    assert render_report(FixResult(changed={}, failures=[])) == "format: no changes"
```

- [ ] **Step 2: Run the tests to verify they fail**

Run: `vrg-container-run -- vrg-validate --check test`
Expected: FAIL with `ModuleNotFoundError: No module named 'vergil_tooling.lib.format_fix'`.

- [ ] **Step 3: Implement `format_fix.py`**

```python
"""Run formatters in write mode and report which files they changed.

Backs ``vrg-validate --fix-format`` (epic vergil-project/.github#388). Fixing is
explicit and never stages or commits: rewritten files are left as ordinary
working-tree changes for the caller to commit.
"""

from __future__ import annotations

import hashlib
import subprocess
from dataclasses import dataclass, field
from typing import TYPE_CHECKING

from vergil_tooling.lib import progress

if TYPE_CHECKING:
    from pathlib import Path

Fixer = tuple[str, list[str]]


class FormatFixError(Exception):
    """The change snapshot could not be taken."""


@dataclass
class FixResult:
    changed: dict[str, list[str]] = field(default_factory=dict)
    failures: list[str] = field(default_factory=list)


def snapshot(repo_root: Path) -> dict[str, str | None]:
    """Hash every tracked and non-ignored untracked file (``None`` = absent)."""
    proc = subprocess.run(  # noqa: S603
        ["git", "ls-files", "-z", "--cached", "--others", "--exclude-standard"],  # noqa: S607
        cwd=repo_root,
        capture_output=True,
        check=False,
    )
    if proc.returncode != 0:
        msg = f"git ls-files failed: {proc.stderr.decode(errors='replace').strip()}"
        raise FormatFixError(msg)
    hashes: dict[str, str | None] = {}
    for raw in proc.stdout.split(b"\0"):
        if not raw:
            continue
        rel = raw.decode()
        path = repo_root / rel
        hashes[rel] = hashlib.sha256(path.read_bytes()).hexdigest() if path.is_file() else None
    return hashes


def _diff(before: dict[str, str | None], after: dict[str, str | None]) -> list[str]:
    return sorted(p for p in before.keys() | after.keys() if before.get(p) != after.get(p))


def run_fixers(fixers: list[Fixer], repo_root: Path) -> FixResult:
    """Run each fixer in order, attributing changed files to the fixer that changed them."""
    result = FixResult()
    before = snapshot(repo_root)
    for label, argv in fixers:
        print(f"Running (format): {' '.join(argv)}")
        try:
            rc = progress.run(argv, check=False)
        except FileNotFoundError:
            result.failures.append(
                f"{label}: '{argv[0]}' not found in this dev container; "
                "update to a dev image that provides it"
            )
            continue
        if rc != 0:
            result.failures.append(f"{label} exited {rc}")
        after = snapshot(repo_root)
        changed = _diff(before, after)
        if changed:
            result.changed[label] = changed
        before = after
    return result


def render_report(result: FixResult) -> str:
    total = sum(len(paths) for paths in result.changed.values())
    if total == 0:
        return "format: no changes"
    lines = [f"format: rewrote {total} files"]
    lines += [f"  {label}: {', '.join(paths)}" for label, paths in result.changed.items()]
    return "\n".join(lines)
```

C++'s `sh -c` fixer can't raise `FileNotFoundError` for a missing
`clang-format`. The shell exits 127, which is recorded as `clang-format exited 127`.
That is acceptable: it is still a named failure.

- [ ] **Step 4: Run the `format_fix` tests to verify they pass**

Run: `vrg-container-run -- vrg-validate --check test`
Expected: the `test_format_fix.py` tests PASS.

- [ ] **Step 5: Write the failing `markdown_fixers` tests**

Add to `tests/vergil_tooling/test_validate_common.py`:

```python
from vergil_tooling.bin.validate_common import markdown_fixers


def test_markdown_fixers_both_passes(tmp_path: Path) -> None:
    _write_minimal_config(tmp_path)  # existing helper in this module, or add one
    (tmp_path / "README.md").write_text("# r\n")
    (tmp_path / "epics" / "1-x").mkdir(parents=True)
    (tmp_path / "epics" / "1-x" / "spec.md").write_text("# s\n")
    fixers = markdown_fixers(tmp_path)
    labels = [label for label, _ in fixers]
    assert labels == ["markdownlint --fix (strict)", "markdownlint --fix (prose)"]
    strict_argv = fixers[0][1]
    assert strict_argv[:2] == ["markdownlint", "--fix"]
    assert strict_argv[-1] == str(tmp_path / "README.md")
    assert "markdownlint.yaml" in strict_argv[3]
    assert "markdownlint-prose.yaml" in fixers[1][1][3]


def test_markdown_fixers_empty_scope_yields_nothing(tmp_path: Path) -> None:
    _write_minimal_config(tmp_path)
    assert markdown_fixers(tmp_path) == []
```

If `test_validate_common.py` has no config-writing helper, add
`_write_minimal_config` using the same `vergil.toml` body as
`test_vrg_validate._write_config`.

- [ ] **Step 6: Implement `markdown_fixers`**

In `validate_common.py`:

```python
def markdown_fixers(repo_root: Path) -> list[tuple[str, list[str]]]:
    """``--fix-format`` Markdown fixers, one per non-empty markdownlint pass.

    Same file discovery and packaged configs as the markdownlint checks in
    :func:`main`, so the fix targets exactly what the gate checks. A pass with
    no files yields no fixer: markdownlint with no paths reads stdin.
    """
    cfg = read_config(repo_root)
    strict_md = _find_strict_markdown_files(repo_root, ignore=cfg.markdownlint.ignore)
    prose_md = _find_prose_markdown_files(repo_root, ignore=cfg.markdownlint.ignore)
    configs = files("vergil_tooling.configs")
    fixers: list[tuple[str, list[str]]] = []
    if strict_md:
        fixers.append((
            "markdownlint --fix (strict)",
            ["markdownlint", "--fix", "--config", str(configs / "markdownlint.yaml"), *strict_md],
        ))
    if prose_md:
        fixers.append((
            "markdownlint --fix (prose)",
            ["markdownlint", "--fix", "--config", str(configs / "markdownlint-prose.yaml"), *prose_md],
        ))
    return fixers
```

- [ ] **Step 7: Write the failing driver tests**

Add to `tests/vergil_tooling/test_vrg_validate.py`:

```python
def test_fix_format_full_run_order() -> None:
    with (
        patch(_MOD + ".language_commands") as m_cmds,
        patch(_MOD + "._find_custom_validator", return_value=None),
    ):
        m_cmds.side_effect = lambda lang, kind, **_kw: [["echo", kind.value]]
        stages = _build_stages(None, "python", Path("/tmp/r"), fix_format=True)  # noqa: S108
    assert [s.name for s in stages] == [
        "install", "format-common", "format-lint",
        "common", "lint", "typecheck", "test", "audit",
    ]
    assert all(s.mode == "fail_defer" for s in stages if s.name != "install")


def test_fix_format_without_language_runs_only_common_fix() -> None:
    with patch(_MOD + "._find_custom_validator", return_value=None):
        stages = _build_stages(None, None, Path("/tmp/r"), fix_format=True)  # noqa: S108
    assert [s.name for s in stages] == ["format-common", "common"]


def test_fix_format_scoped_to_common() -> None:
    stages = _build_stages("common", "python", Path("/tmp/r"), fix_format=True)  # noqa: S108
    assert [s.name for s in stages] == ["format-common", "common"]


def test_fix_format_scoped_to_lint() -> None:
    with patch(_MOD + ".language_commands", side_effect=lambda l, k, **_kw: [["echo", k.value]]):
        stages = _build_stages("lint", "python", Path("/tmp/r"), fix_format=True)  # noqa: S108
    assert [s.name for s in stages] == ["install", "format-lint", "lint"]


def test_plain_run_order_unchanged() -> None:
    with (
        patch(_MOD + ".language_commands") as m_cmds,
        patch(_MOD + "._find_custom_validator", return_value=None),
    ):
        m_cmds.side_effect = lambda lang, kind, **_kw: [["echo", kind.value]]
        stages = _build_stages(None, "python", Path("/tmp/r"))  # noqa: S108
    assert [s.name for s in stages][:3] == ["common", "install", "lint"]


@pytest.mark.parametrize("check", ["typecheck", "test", "audit"])
def test_fix_format_rejects_non_fixable_check(check: str) -> None:
    assert main(["--fix-format", "--check", check]) == 2


@pytest.mark.parametrize("var", ["CI", "GITHUB_ACTIONS"])
def test_fix_format_refused_in_ci(var: str, monkeypatch: pytest.MonkeyPatch) -> None:
    monkeypatch.setenv(var, "true")
    assert main(["--fix-format"]) == 2


def test_format_stage_raises_on_fixer_failure(tmp_path: Path) -> None:
    from vergil_tooling.lib.format_fix import FixResult

    with patch(_MOD + ".run_fixers", return_value=FixResult(failures=["ruff format exited 2"])):
        stage = _format_stage("format-lint", lambda: [("ruff format", ["ruff", "format"])], tmp_path)
        with pytest.raises(ValidationError, match="ruff format exited 2"):
            stage.fn(None)
```

Import `_format_stage` from the driver alongside the existing imports.
If an existing test unsets CI, mirror it. Otherwise make the existing
autouse fixture also `monkeypatch.delenv("CI", raising=False)` and
`monkeypatch.delenv("GITHUB_ACTIONS", raising=False)`, so the suite does not
behave differently when it runs in CI.

- [ ] **Step 8: Run the driver tests to verify they fail**

Run: `vrg-container-run -- vrg-validate --check test`
Expected: FAIL with `TypeError: _build_stages() got an unexpected keyword argument 'fix_format'`.

- [ ] **Step 9: Implement the driver changes**

In `vrg_validate.py`:

```python
from vergil_tooling.bin.validate_common import markdown_fixers
from vergil_tooling.lib.format_fix import Fixer, render_report, run_fixers
from vergil_tooling.lib.languages import CheckKind, language_commands, language_formatters

_FIX_FORMAT_CHECKS = (None, "common", "lint")
```

Add the argument and refusals in `main`, after the container guard:

```python
    parser.add_argument(
        "--fix-format",
        action="store_true",
        help=(
            "Run every formatter in write mode before the checks. Rewrites files "
            "in place; never stages or commits. Local only."
        ),
    )
    ...
    if args.fix_format and (os.environ.get("CI") or os.environ.get("GITHUB_ACTIONS")):
        print(
            "ERROR: --fix-format is local-only; CI validation is check-only.",
            file=sys.stderr,
        )
        return 2
    if args.fix_format and args.check not in _FIX_FORMAT_CHECKS:
        print(
            f"ERROR: --fix-format has nothing to fix for --check {args.check}; "
            "use it with a full run, --check common or --check lint.",
            file=sys.stderr,
        )
        return 2
```

Pass `fix_format=args.fix_format` to `_build_stages`.

Add the stage builder:

```python
def _format_stage(label: str, fixers: Callable[[], list[Fixer]], repo_root: Path) -> Stage:
    def fn(_ctx: object) -> None:
        result = run_fixers(fixers(), repo_root)
        print(render_report(result))
        if result.failures:
            raise ValidationError("; ".join(result.failures))

    return Stage(label, fn, mode="fail_defer")
```

`fixers` is a callable, so file discovery runs when the stage runs, after
install, not at build time.

Restructure `_build_stages` (keep its existing body as the source of the
command lists):

```python
def _build_stages(
    check: str | None,
    language: str | None,
    repo_root: Path,
    *,
    cpp_std: str | None = None,
    cpp_stdlib: str | None = None,
    test_parallel: bool = True,
    xdist_available: bool = False,
    fix_format: bool = False,
) -> list[Stage]:
    common_stage = _common_stage(repo_root) if check in (None, "common") else None
    install_stage, language_stages = _language_stages(
        check, language,
        cpp_std=cpp_std, cpp_stdlib=cpp_stdlib,
        test_parallel=test_parallel, xdist_available=xdist_available,
    )
    custom_stage = _custom_stage(repo_root) if check is None else None

    stages: list[Stage] = []
    if not fix_format:
        stages += [s for s in (common_stage, install_stage) if s is not None]
    else:
        if install_stage is not None:
            stages.append(install_stage)
        if common_stage is not None:
            stages.append(
                _format_stage("format-common", lambda: markdown_fixers(repo_root), repo_root)
            )
        lint_selected = any(s.name == "lint" for s in language_stages)
        if lint_selected:
            stages.append(
                _format_stage(
                    "format-lint",
                    lambda: [
                        (f.name, f.fix)
                        for f in language_formatters(
                            language, cpp_std=cpp_std, cpp_stdlib=cpp_stdlib
                        )
                    ],
                    repo_root,
                )
            )
        if common_stage is not None:
            stages.append(common_stage)
    stages += language_stages
    if custom_stage is not None:
        stages.append(custom_stage)
    return stages
```

Extract `_common_stage(repo_root) -> Stage`, `_custom_stage(repo_root) -> Stage | None`
(returns `None` when no custom validator exists) and
`_language_stages(...) -> tuple[Stage | None, list[Stage]]` from the existing
body without changing their behavior. `_language_stages` returns the install
stage, or `None`, separately from the lint/typecheck/test/audit stages.
Add `Callable` to the `TYPE_CHECKING` imports.

- [ ] **Step 10: Run all tests to verify they pass**

Run: `vrg-container-run -- vrg-validate --check test`
Expected: PASS, with 100% coverage.

- [ ] **Step 11: Real-tool check in this repo**

In the worktree, add a deliberately misformatted line to a scratch copy:
`printf 'x=1\n' > src/vergil_tooling/_scratch_fmt.py`, then run
`vrg-container-run -- vrg-validate --fix-format --check lint`.
Expected: `format: rewrote 1 files` naming `_scratch_fmt.py` under
`ruff format`, and the file now contains `x = 1`. Delete the scratch file.
Do not commit it.

- [ ] **Step 12: Full validation and commit**

Run: `vrg-container-run -- vrg-validate`
Expected: PASS.

```bash
vrg-git add src/vergil_tooling/lib/format_fix.py src/vergil_tooling/bin/vrg_validate.py \
  src/vergil_tooling/bin/validate_common.py tests/vergil_tooling/test_format_fix.py \
  tests/vergil_tooling/test_vrg_validate.py tests/vergil_tooling/test_validate_common.py
vrg-commit --type feat --scope validate --message "add opt-in --fix-format that runs formatters before the checks"
```

---

### Task 5: Firm and soft formatting hints, plus docs (vergil-tooling)

**Files:**

- Modify: `src/vergil_tooling/bin/vrg_validate.py` (record failing commands, print hints)
- Modify: `src/vergil_tooling/bin/validate_common.py` (report a markdownlint failure)
- Modify: `docs/site/docs/guides/validation-matrix.md`, `docs/site/docs/reference/cli-tools-overview.md`, `docs/site/docs/reference/lint/markdown-standards.md`, `CLAUDE.md` (Validation section)
- Test: `tests/vergil_tooling/test_vrg_validate.py`, `tests/vergil_tooling/test_validate_common.py`

**Interfaces:**

- Consumes: `language_formatters()` (Task 3); `_build_stages`, `_command_stage` (Task 4).
- Produces:
  - `_command_stage(label, cmds, *, mode, failed: list[list[str]] | None = None)`,
    which appends each failing argv to `failed`.
  - `validate_common.main(argv=None, *, failed_tools: list[str] | None = None) -> int`,
    which appends `"markdownlint"` when either markdownlint pass fails.
  - `vrg_validate.format_hints(failed: list[list[str]], failed_tools: list[str], formatters: list[Formatter]) -> list[str]`

- [ ] **Step 1: Write the failing tests**

```python
from vergil_tooling.bin.vrg_validate import FIRM_HINT, format_hints
from vergil_tooling.lib.languages import Formatter

_RUFF_FMT = Formatter(name="ruff format", check=["ruff", "format", "--check"], fix=["ruff", "format"])
_RUFF_SORT = Formatter(
    name="ruff import sort",
    fix=["ruff", "check", "--select", "I", "--fix"],
    soft_hint_for=["ruff", "check", "src/", "tests/"],
)


def test_firm_hint_for_pure_formatter_failure() -> None:
    hints = format_hints([["ruff", "format", "--check"]], [], [_RUFF_FMT, _RUFF_SORT])
    assert hints == [FIRM_HINT]
    assert FIRM_HINT == (
        "formatting failures can be fixed with: "
        "vrg-container-run -- vrg-validate --fix-format"
    )


def test_soft_hint_for_mixed_gate_failure() -> None:
    hints = format_hints([["ruff", "check", "src/", "tests/"]], [], [_RUFF_FMT, _RUFF_SORT])
    assert hints == [
        "ruff check src/ tests/ failed; some of its findings may be auto-fixable "
        "with: vrg-container-run -- vrg-validate --fix-format"
    ]


def test_soft_hint_for_markdownlint() -> None:
    hints = format_hints([], ["markdownlint"], [])
    assert hints == [
        "markdownlint failed; some of its findings may be auto-fixable "
        "with: vrg-container-run -- vrg-validate --fix-format"
    ]


def test_no_hint_for_unrelated_failure() -> None:
    assert format_hints([["mypy", "src"]], ["shellcheck"], [_RUFF_FMT, _RUFF_SORT]) == []


def test_command_stage_records_failed_commands() -> None:
    failed: list[list[str]] = []
    with patch(_MOD + ".progress.run", side_effect=[0, 1]):
        stage = _command_stage("lint", [["ok"], ["bad"]], mode="fail_defer", failed=failed)
        with pytest.raises(ValidationError):
            stage.fn(None)
    assert failed == [["bad"]]


def test_main_prints_hint_after_failing_run(
    tmp_path: Path, capsys: pytest.CaptureFixture[str], monkeypatch: pytest.MonkeyPatch
) -> None:
    _write_config(tmp_path, "python")
    monkeypatch.chdir(tmp_path)

    def fake_pipeline(*_a: object, **_kw: object) -> int:
        _RECORDED_FAILURES.append(["ruff", "format", "--check", "src/", "tests/"])
        return 1

    with (
        patch(_MOD + ".git.repo_root", return_value=tmp_path),
        patch(_MOD + ".progress.run_pipeline", side_effect=fake_pipeline),
    ):
        assert main([]) == 1
    assert FIRM_HINT in capsys.readouterr().err
```

`_RECORDED_FAILURES` is the module-level failure list described in Step 3.
Import it from the driver for this test.

And in `test_validate_common.py`: a test that patches the markdownlint
`subprocess.run` to return code 1, calls `main(failed_tools=tools)`, and
asserts `tools == ["markdownlint"]`. Use this module's existing patching
pattern for the markdownlint calls.

- [ ] **Step 2: Run the tests to verify they fail**

Run: `vrg-container-run -- vrg-validate --check test`
Expected: FAIL with `ImportError: cannot import name 'FIRM_HINT'`.

- [ ] **Step 3: Implement**

In `vrg_validate.py`:

```python
_FIX_CMD = "vrg-container-run -- vrg-validate --fix-format"
FIRM_HINT = f"formatting failures can be fixed with: {_FIX_CMD}"

# Failing argvs and common-check tool names recorded during one run, read after
# the pipeline to print --fix-format hints (epic vergil-project/.github#388).
_RECORDED_FAILURES: list[list[str]] = []
_RECORDED_TOOL_FAILURES: list[str] = []


def _soft_hint(gate: str) -> str:
    return f"{gate} failed; some of its findings may be auto-fixable with: {_FIX_CMD}"


def format_hints(
    failed: list[list[str]], failed_tools: list[str], formatters: list[Formatter]
) -> list[str]:
    hints: list[str] = []
    if any(f.check is not None and f.check in failed for f in formatters):
        hints.append(FIRM_HINT)
    hints += [
        _soft_hint(" ".join(f.soft_hint_for))
        for f in formatters
        if f.check is None and f.soft_hint_for is not None and f.soft_hint_for in failed
    ]
    if "markdownlint" in failed_tools:
        hints.append(_soft_hint("markdownlint"))
    return hints
```

`_command_stage` gains `failed: list[list[str]] | None = None` and does
`if failed is not None: failed.append(list(cmd))` on each non-zero exit.
`_language_stages` passes `failed=_RECORDED_FAILURES`. The common stage calls
`_run_common_checks(repo_root, _RECORDED_TOOL_FAILURES)`, which forwards
`failed_tools=` to `validate_common.main`.

In `main`, clear both lists before building stages. After `run_pipeline`
returns non-zero, print each hint from
`format_hints(_RECORDED_FAILURES, _RECORDED_TOOL_FAILURES, language_formatters(language, cpp_std=cpp_std, cpp_stdlib=cpp_stdlib))`
to stderr, then return the pipeline's exit code unchanged.

In `validate_common.main`, add the keyword-only `failed_tools` parameter. When
`markdownlint_rc != 0`, append `"markdownlint"` (if `failed_tools` is not
`None`) before the existing early return.

- [ ] **Step 4: Run the tests to verify they pass**

Run: `vrg-container-run -- vrg-validate --check test`
Expected: PASS, with 100% coverage.

- [ ] **Step 5: Write the docs**

- `validation-matrix.md`: add a "Fixing formatting" section. It covers what
  `--fix-format` does; the order (fixes first, then the normal run); that it
  never stages or commits and is refused in CI; the per-language fix table
  from spec §4; and the firm/soft hint wording.
- `cli-tools-overview.md`: one line under `vrg-validate` for `--fix-format`.
- `markdown-standards.md`: `--fix-format` runs `markdownlint --fix`. Say that
  line-length reflow arrives with rumdl (Task 6 updates this), and that long
  headings stay manual.
- `CLAUDE.md` Validation section: one paragraph. "On formatting failures,
  `vrg-container-run -- vrg-validate --fix-format` rewrites files with the
  formatters, then validates. Commit the rewritten files with your work."

- [ ] **Step 6: Full validation and commit**

Run: `vrg-container-run -- vrg-validate`
Expected: PASS (markdownlint covers the docs).

```bash
vrg-git add src/vergil_tooling/bin/vrg_validate.py src/vergil_tooling/bin/validate_common.py \
  tests/vergil_tooling/test_vrg_validate.py tests/vergil_tooling/test_validate_common.py \
  docs/site/docs/guides/validation-matrix.md docs/site/docs/reference/cli-tools-overview.md \
  docs/site/docs/reference/lint/markdown-standards.md CLAUDE.md
vrg-commit --type feat --scope validate --message "hint --fix-format on formatter failures and document it"
```

---

### Task 6: rumdl reflow companion for strict Markdown (vergil-tooling)

Blocked by Task 2 (rumdl in the published dev image) and Task 4.

**Files:**

- Create: `src/vergil_tooling/configs/rumdl.toml`
- Modify: `src/vergil_tooling/bin/validate_common.py` (`markdown_fixers`)
- Modify: `docs/site/docs/reference/lint/markdown-standards.md`
- Test: `tests/vergil_tooling/test_validate_common.py`, `tests/vergil_tooling/test_rumdl_convergence.py` (new)

**Interfaces:**

- Consumes: `markdown_fixers()` (Task 4).
- Produces: `markdown_fixers()` returns, for the strict pass,
  `("rumdl fmt (strict)", [...])` followed by `("markdownlint --fix (strict)", [...])`.
  The rumdl entry lists only strict files that have an over-long line
  (`_has_overlong_line`), and is omitted when none does.

**Scope note:** the prose pass's config disables MD013 entirely
(`markdownlint-prose.yaml`), so there is nothing to reflow there. rumdl runs on
the **strict** pass only. The prose pass keeps `markdownlint --fix` alone.

**Churn note (spec §5.3):** rumdl's reflow can rewrap paragraphs that already
pass MD013, so rumdl is given only files with a line MD013 would flag. A file
with no such line is never touched. The scan mirrors the strict config: fenced
code blocks and table rows are exempt, and, as in markdownlint's default
non-strict MD013, a line is flagged only when it has whitespace beyond column
100 (a long bare URL is not flagged).

- [ ] **Step 1: Write the rumdl config, mirroring `markdownlint.yaml`**

`src/vergil_tooling/configs/rumdl.toml`:

```toml
# rumdl config for `vrg-validate --fix-format` (epic vergil-project/.github#388).
#
# Mirrors configs/markdownlint.yaml (the strict Markdown gate) and adds MD013
# reflow, which markdownlint cannot do. markdownlint remains the gate; rumdl is
# a fix-only companion. Keep in sync with markdownlint.yaml.

[global]
disable = ["MD060"]

[MD013]
line-length = 100
tables = false
code-blocks = false
reflow = true

[MD024]
siblings-only = true
```

- [ ] **Step 2: Verify the config keys against the installed rumdl**

Run: `vrg-container-run -- rumdl check --config src/vergil_tooling/configs/rumdl.toml README.md`
Expected: no unknown-key or config-parse warnings. If rumdl reports an
unknown key, look up the correct key in `rumdl config --help` or
<https://github.com/rvben/rumdl/blob/main/docs/md013.md>, fix the file, and
re-run. Record the rumdl version used in the commit body.

- [ ] **Step 3: Write the failing tests**

In `test_validate_common.py`, update `test_markdown_fixers_both_passes` so
`README.md` has an over-long line (write
`"# r\n\n" + "word " * 30 + "\n"` instead of `"# r\n"`), then assert:

```python
    assert labels == [
        "rumdl fmt (strict)",
        "markdownlint --fix (strict)",
        "markdownlint --fix (prose)",
    ]
    rumdl_argv = fixers[0][1]
    assert rumdl_argv[:2] == ["rumdl", "fmt"]
    assert rumdl_argv[2] == "--config"
    assert rumdl_argv[3].endswith("rumdl.toml")
    assert rumdl_argv[-1] == str(tmp_path / "README.md")
```

`test_markdown_fixers_empty_scope_yields_nothing` stays as is. It now also
proves rumdl is never called with no paths (Review Focus 2).

Add the selection tests:

```python
from vergil_tooling.bin.validate_common import _has_overlong_line

_LONG_PROSE = ("word " * 30).strip()  # 149 characters, spaces throughout
_FENCE = "`" * 3


def test_rumdl_gets_only_files_with_overlong_lines(tmp_path: Path) -> None:
    _write_minimal_config(tmp_path)
    (tmp_path / "README.md").write_text(f"# r\n\n{_LONG_PROSE}\n")
    site = tmp_path / "docs" / "site" / "docs"
    site.mkdir(parents=True)
    (site / "clean.md").write_text("# c\n\nshort line\n")
    fixers = dict(markdown_fixers(tmp_path))
    assert fixers["rumdl fmt (strict)"][4:] == [str(tmp_path / "README.md")]
    assert str(site / "clean.md") in fixers["markdownlint --fix (strict)"]


def test_no_overlong_lines_means_no_rumdl(tmp_path: Path) -> None:
    _write_minimal_config(tmp_path)
    (tmp_path / "README.md").write_text("# r\n\nshort line\n")
    labels = [label for label, _ in markdown_fixers(tmp_path)]
    assert labels == ["markdownlint --fix (strict)"]


def test_overlong_line_exemptions(tmp_path: Path) -> None:
    doc = tmp_path / "d.md"
    doc.write_text(f"{_FENCE}text\n{_LONG_PROSE}\n{_FENCE}\n")
    assert not _has_overlong_line(doc)  # fenced code is exempt
    doc.write_text(f"| a | b |\n| --- | --- |\n| {_LONG_PROSE} | x |\n")
    assert not _has_overlong_line(doc)  # table rows are exempt
    doc.write_text("see https://example.com/" + "x" * 120 + "\n")
    assert not _has_overlong_line(doc)  # no whitespace beyond column 100
    doc.write_text(f"# h\n\n{_LONG_PROSE}\n")
    assert _has_overlong_line(doc)


def test_overlong_scan_read_error_is_raised(tmp_path: Path) -> None:
    with pytest.raises(FileNotFoundError):
        _has_overlong_line(tmp_path / "missing.md")
```

The read-error test pins "no silent failures": an unreadable file fails the
`format-common` stage instead of being quietly skipped. (The file list comes
from discovery, so this only happens on a race or permissions problem.)

Create `tests/vergil_tooling/test_rumdl_convergence.py`. This is a real-tool
test: it requires `rumdl` and `markdownlint` in the dev container and **fails**
(never skips) when they are absent.

```python
"""rumdl reflow + markdownlint --fix converge to a markdownlint-clean file (epic #388)."""

from __future__ import annotations

import subprocess
from importlib.resources import files
from pathlib import Path

_CONFIGS = files("vergil_tooling.configs")
_LONG = ("word " * 60).strip()  # ~300 characters in one paragraph line
_FENCE = "`" * 3

_FIXTURE = (
    f"# Title\n\n{_LONG}\n\n"
    f"- a list item that is short\n- {_LONG}\n\n"
    f"> {_LONG}\n\n"
    f"| col | col |\n| --- | --- |\n| {_LONG} | x |\n\n"
    f"{_FENCE}text\n{_LONG}\n{_FENCE}\n"
)


def _run(argv: list[str]) -> subprocess.CompletedProcess[str]:
    return subprocess.run(argv, capture_output=True, text=True, check=False)  # noqa: S603


def test_rumdl_then_markdownlint_fix_passes_markdownlint(tmp_path: Path) -> None:
    doc = tmp_path / "doc.md"
    doc.write_text(_FIXTURE)
    strict = str(_CONFIGS / "markdownlint.yaml")
    assert _run(["markdownlint", "--config", strict, str(doc)]).returncode != 0
    fmt = _run(["rumdl", "fmt", "--config", str(_CONFIGS / "rumdl.toml"), str(doc)])
    assert fmt.returncode == 0, fmt.stderr
    _run(["markdownlint", "--fix", "--config", strict, str(doc)])
    final = _run(["markdownlint", "--config", strict, str(doc)])
    assert final.returncode == 0, final.stdout + final.stderr


def test_fix_is_idempotent(tmp_path: Path) -> None:
    doc = tmp_path / "doc.md"
    doc.write_text(_FIXTURE)
    cfg = str(_CONFIGS / "rumdl.toml")
    _run(["rumdl", "fmt", "--config", cfg, str(doc)])
    once = doc.read_text()
    _run(["rumdl", "fmt", "--config", cfg, str(doc)])
    assert doc.read_text() == once
```

- [ ] **Step 4: Run the tests to verify they fail**

Run: `vrg-container-run -- vrg-validate --check test`
Expected: `test_markdown_fixers_both_passes` FAILS (labels mismatch), and the
selection tests FAIL with `ImportError` (`_has_overlong_line`). The
convergence tests PASS or FAIL on the tools alone. If
`test_rumdl_then_markdownlint_fix_passes_markdownlint` fails, read its output.
A markdownlint rule tripped by rumdl's output is the loop that spec §7 warns
about. Fix `rumdl.toml`. If the conflict cannot be fixed by config, stop and
report it as a blocker; do not weaken the test.

- [ ] **Step 5: Implement**

In `validate_common.py`, add the scan:

```python
_MD013_LINE_LENGTH = 100  # keep in sync with configs/markdownlint.yaml MD013


def _has_overlong_line(path: Path) -> bool:
    """True if *path* has a line the strict MD013 config would flag.

    Mirrors configs/markdownlint.yaml: fenced code blocks and table rows are
    exempt, and (markdownlint's non-strict default) a line is flagged only
    when there is whitespace beyond the limit. Used to give rumdl only files
    that need reflow, because rumdl can rewrap already-compliant paragraphs
    (epic vergil-project/.github#388). Read errors propagate.
    """
    fence: str | None = None
    for line in path.read_text(encoding="utf-8").splitlines():
        stripped = line.lstrip()
        if fence is not None:
            if stripped.startswith(fence):
                fence = None
            continue
        if stripped.startswith(("```", "~~~")):
            fence = stripped[:3]
            continue
        if stripped.startswith("|"):
            continue
        if len(line) > _MD013_LINE_LENGTH and any(
            c.isspace() for c in line[_MD013_LINE_LENGTH:]
        ):
            return True
    return False
```

In `markdown_fixers`, before the strict markdownlint fixer:

```python
    reflow_md = [f for f in strict_md if _has_overlong_line(Path(f))]
    if reflow_md:
        fixers.append((
            "rumdl fmt (strict)",
            ["rumdl", "fmt", "--config", str(configs / "rumdl.toml"), *reflow_md],
        ))
```

- [ ] **Step 6: Convergence on real docs**

Run in this worktree: `vrg-container-run -- vrg-validate --fix-format --check common`
Expected: the run passes and prints `format: no changes`. The repo's docs
already pass the markdownlint gate, so no strict file has an over-long line and
rumdl is never invoked. Any change here means the scan or `markdownlint --fix`
is touching clean files: stop and fix that before going on.

Then append one over-long prose paragraph (spaces throughout, over 100
characters) to one page under `docs/site/docs/`, and re-run the same command.
Expected: the run passes, and the change report lists only that page, under
`rumdl fmt (strict)`. Review the diff: only that file's line wrapping should
change. Revert it with `vrg-git checkout -- docs/site`, and record both runs'
change reports in the PR notes.

- [ ] **Step 7: Update docs**

`markdown-standards.md`: `--fix-format` runs `rumdl fmt` (MD013 reflow at 100
columns; code blocks, tables, headings and reference definitions left alone)
and then `markdownlint --fix` on the strict scope. `epics/` gets only
`markdownlint --fix`, because MD013 is off there. markdownlint remains the
gate. Point to follow-on `vergil-project/.github#392`.

- [ ] **Step 8: Full validation and commit**

Run: `vrg-container-run -- vrg-validate`
Expected: PASS.

```bash
vrg-git add src/vergil_tooling/configs/rumdl.toml src/vergil_tooling/bin/validate_common.py \
  tests/vergil_tooling/test_validate_common.py tests/vergil_tooling/test_rumdl_convergence.py \
  docs/site/docs/reference/lint/markdown-standards.md
vrg-commit --type feat --scope validate --message "reflow strict Markdown with rumdl under --fix-format"
```

Confirm that `pyproject.toml`'s package-data configuration ships
`configs/*.toml`. `cliff.toml` already lives there, so it should. If it
doesn't, add it.

---

### Task 7: Ruby and Go fix-only companions (vergil-tooling)

Blocked by Task 4. This task ships **only what it verifies** (spec §7).

**Files:**

- Modify: `src/vergil_tooling/lib/languages.py` (ruby and go `formatters=`)
- Test: `tests/vergil_tooling/test_languages.py`

**Interfaces:**

- Consumes: `Formatter`, `language_formatters` (Task 3).
- Produces: the ruby and go formatter tuples. Their LINT lists are unchanged.

vergil-project has no Ruby or Go consuming repo, so both checks run in a
**throwaway local repo** (spec §9): a new scratch directory outside any
existing repo, with `git init`, a `vergil.toml` naming the language as
`primary-language`, the build files the language's `install` stage needs, and
one source file. First confirm the clean baseline:
`vrg-container-run -- vrg-validate --check lint` passes before any defect is
added. Record every file you created in the PR notes; Task 8 reuses the recipe.

- [ ] **Step 1: Verify Ruby in a throwaway Ruby repo**

In the Ruby repo (`Gemfile` with `rubocop`, and `Gemfile.lock` from
`vrg-container-run -- bundle install`), introduce only Layout offences (for example, wrong indentation and
trailing whitespace). Run
`vrg-container-run -- bundle exec rubocop` and expect a failure that lists only
`Layout/*` cops. Then run `vrg-container-run -- bundle exec rubocop --fix-layout`
followed by `vrg-container-run -- bundle exec rubocop`, and expect exit 0.
Record the commands and output in the PR notes. If it fails, Ruby gets no
companion: drop Steps 3–4's Ruby part and record the gap.

- [ ] **Step 2: Verify Go in a throwaway Go repo**

In the Go repo (`go.mod` and a `main.go`), add a `.golangci.yml` (golangci-lint
v2 format) that enables `gofmt` under `formatters:`. Real Go repos that adopt
`--fix-format` need the same, which the docs must say. Then confirm `vrg-container-run -- golangci-lint fmt --help`
works (v2), introduce a gofmt-only defect, check that
`vrg-container-run -- golangci-lint run ./...` fails on it, and check that
`golangci-lint fmt` followed by `golangci-lint run ./...` passes. Record the
evidence. If it fails, Go gets no companion: drop the Go part and record the gap.

- [ ] **Step 3: Write the failing tests** (for each verified language)

```python
def test_ruby_layout_companion() -> None:
    (fmt,) = language_formatters("ruby")
    assert fmt.check is None
    assert fmt.fix == ["bundle", "exec", "rubocop", "--fix-layout"]
    assert fmt.soft_hint_for == ["bundle", "exec", "rubocop"]


def test_go_fmt_companion() -> None:
    (fmt,) = language_formatters("go")
    assert fmt.check is None
    assert fmt.fix == ["golangci-lint", "fmt"]
    assert fmt.soft_hint_for == ["golangci-lint", "run", "./..."]
```

Run: `vrg-container-run -- vrg-validate --check test`
Expected: FAIL with `ValueError: not enough values to unpack`.

- [ ] **Step 4: Implement**

```python
_RUBY_FORMATTERS = (
    Formatter(
        name="rubocop --fix-layout",
        fix=["bundle", "exec", "rubocop", "--fix-layout"],
        soft_hint_for=["bundle", "exec", "rubocop"],
    ),
)

_GO_FORMATTERS = (
    Formatter(
        name="golangci-lint fmt",
        fix=["golangci-lint", "fmt"],
        soft_hint_for=["golangci-lint", "run", "./..."],
    ),
)
```

Set `formatters=` on the ruby and go entries. Their LINT lists are unchanged.
The Task 3 registry tests now also cover these languages.

- [ ] **Step 5: Run the tests, validate and commit**

Run: `vrg-container-run -- vrg-validate`
Expected: PASS.

```bash
vrg-git add src/vergil_tooling/lib/languages.py tests/vergil_tooling/test_languages.py
vrg-commit --type feat --scope validate --message "add verified Ruby and Go fix-only formatter companions"
```

---

### Task 8: Cross-language `--fix-format` validation (validation, vergil-tooling)

Create with `vrg-issue-create --kind validation` blocked by Tasks 5, 6 and 7.

**Precondition (self-check):** Tasks 5–7 are merged to vergil-tooling
`develop`. Task 2 is closed (`Outcome: SUCCESS`).

**Targets (spec §9).** vergil-project has no consuming repo for TypeScript,
Rust, Java, Ruby or Go, and only a skeleton for C++, so every language except
Python uses a **throwaway local repo**. Permanent test-only repos are the
subject of a follow-on brainstorm under epic #388.

| Language | Target | Minimal contents besides `vergil.toml` |
|---|---|---|
| Python | `vergil-project/vergil-tooling` (scratch branch) | — |
| TypeScript | throwaway | `package.json` and `package-lock.json` (for `npm ci`), `tsconfig.json`, `src/index.ts` |
| Rust | throwaway | `Cargo.toml`, `Cargo.lock`, `src/main.rs` |
| Java | throwaway | Maven wrapper (`mvnw`, `.mvn/`), `pom.xml` with the spotless and checkstyle plugins, one class |
| C++ | throwaway, from the existing C++ skeleton | the skeleton's files |
| Ruby | throwaway, reuse Task 7's | `Gemfile`, `Gemfile.lock`, one `.rb` file |
| Go | throwaway, reuse Task 7's | `go.mod`, `main.go`, `.golangci.yml` enabling `gofmt` |
| Markdown | every target | `README.md` |

A throwaway repo is a new scratch directory outside any existing repo, with
`git init`, a `vergil.toml` naming the language as `primary-language`, and the
files above. Before adding any defect, confirm the baseline:
`vrg-container-run -- vrg-validate --check common` and `--check lint` both pass.
If a language's baseline cannot be made to pass its lint checks, record that
and its output as a FAILURE for that language; do not change the registry to
get there. Record each repo's exact files in the results comment, so the
recipe can be reused.

**Procedure (per target; the throwaway repos and the scratch branch are discarded afterwards):**

1. `export VRG_DOCKER_INSTALL_TAG=develop` so the container installs
   vergil-tooling `develop`.
2. Introduce one formatting-only defect for the language's formatter, plus an
   over-long paragraph line in `README.md`.
3. Run `vrg-container-run -- vrg-validate --check common`, then the same
   with `--check lint`. (A throwaway repo has no meaningful typecheck, test or
   audit stages, so only the two stages `--fix-format` touches are run.)
   Expected: non-zero, and the right
   hint (firm for Python, TypeScript, Rust, Java and C++; soft for Ruby, Go and
   markdownlint).
4. Run `vrg-container-run -- vrg-validate --fix-format --check common`, then
   the same with `--check lint`. Expected: the change reports name the defect
   files and nothing else.
5. Re-run step 3's two commands. Expected: both pass.

**Acceptance:** every language's pair clears its defect, or a failing pair is
recorded with its output and a follow-up task is filed to remove it.
`Outcome: SUCCESS` only if every pair that shipped passes.
