# Centralized baseline `.gitignore` + self-policing ops audit — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Make the baseline `.gitignore` a centrally-owned single source of truth, enforced (with `ops.yml` presence-and-wiring) by the nightly config audit, and reconcile the fleet so drift is caught automatically.

**Architecture:** A packaged data asset `gitignore.baseline` becomes the one definition consumed by both `repo_init` scaffolding and a new local audit check. Two pure-local checks join `audit_local_config` (`_check_gitignore` superset + `_check_required_workflows` wiring), so they run — and are fatal — inside the existing nightly `ops.yml` config-audit job. `repo_init` gains `render_ops_workflow` (staggered per-repo cron). A one-time fleet sweep reconciles every repo; other orgs get their own rollout epics.

**Tech Stack:** Python 3.12–3.14, `importlib.resources` for packaged data, `hashlib` for deterministic cron staggering, pytest (100% coverage gate). Validation via `vrg-container-run -- vrg-validate`.

**Spec:** `epics/311-gitignore-baseline/spec.md` (read it alongside this plan).

## Global Constraints

- **Portability:** all changes must work on macOS and Linux (pure Python; no shell-specific behavior).
- **Coverage:** the repo enforces `--cov-fail-under=100`; every new line needs a test. Branch coverage (`--cov-branch`) can differ across 3.12/3.13/3.14 — PR-CI runs the coverage matrix, so avoid version-conditional branches.
- **No repo-specific logic:** the baseline and checks must work in any consuming repo.
- **Validation entry point:** `vrg-container-run -- vrg-validate` (transparently `uv run vrg-validate` here). Never run individual linters directly.
- **Reusable-workflow pins use the rolling `@v2.1` tag** (matches every other rendered workflow in `repo_init`).
- **Data-asset load idiom:** `importlib.resources.files("vergil_tooling.data").joinpath(<name>).read_text(encoding="utf-8")` (mirrors `_load_template`).

---

## File structure

| File | Responsibility | Change |
|---|---|---|
| `src/vergil_tooling/data/gitignore.baseline` | The single canonical baseline `.gitignore` (patterns + comments) | **create** |
| `pyproject.toml` (`[tool.setuptools.package-data]`, ~line 90) | Ship the baseline as package data | **modify** |
| `src/vergil_tooling/lib/repo_init.py` | `render_gitignore()` reads the asset; new `render_ops_workflow()` + `_ops_cron_minute()`; write `ops.yml` in `step_ci_cd_workflows` | **modify** |
| `src/vergil_tooling/lib/repo_config.py` | `_check_gitignore`, `_check_required_workflows`, `_load_gitignore_baseline`, `_gitignore_patterns`; register both in `audit_local_config` | **modify** |
| `tests/lib/test_repo_config.py` | Tests for both new checks | **modify/create** |
| `tests/lib/test_repo_init.py` | Tests for baseline read, `render_ops_workflow`, cron derivation, round-trip | **modify/create** |

All of the above lands as **one implementation PR in `vergil-tooling`** (epic task filed in step 9). The tasks below are its internal TDD units. The rollout and cross-org work follow as operational tasks after release.

---

### Task 1: Create the canonical baseline data asset

**Files:**

- Create: `src/vergil_tooling/data/gitignore.baseline`
- Modify: `pyproject.toml` (`[tool.setuptools.package-data]`)

**Interfaces:**

- Produces: the packaged file `vergil_tooling.data/gitignore.baseline`, read by Task 2 (`render_gitignore`) and Task 4 (`_check_gitignore`).

The content is the **integral** of the fleet's `.gitignore` files, canonicalized to one spelling per pattern (per spec §2, resolved O1). Start from the set below; before finalizing, run the integration check in Step 3 to fold in anything a fleet repo ignores that is missing here.

- [ ] **Step 1: Write the baseline file**

Create `src/vergil_tooling/data/gitignore.baseline` with exactly:

```gitignore
# Vergil baseline .gitignore — single source of truth (epic vergil-project/.github#311).
# Every managed repo's .gitignore must be a SUPERSET of the non-comment lines
# below (verbatim). Repos may add their own local entries. Do not edit a
# consuming repo to diverge — change this file in vergil-tooling and release.

# Editors
*.swp
*.swo
*~
.idea/
.vscode/

# OS
.DS_Store
Thumbs.db

# Environment / secrets
.env
.env.*

# Logs
*.log

# Vergil internals
.venv/
.worktrees/
.vergil/
.superpowers/
.claude/scheduled_tasks.lock

# Build / packaging output
build/
dist/
*.egg-info/

# Python bytecode / caches
__pycache__/
*.pyc
.pytest_cache/
.mypy_cache/
.ruff_cache/

# Coverage / validation / CI-gate evidence
.coverage
coverage.xml
junit.xml
pip-audit.json
licenses.json
quality-ruff.json
quality-mypy.xml

# Docs (mkdocs build output; mkdocs.yml lives at docs/site/)
docs/site/site/

# Node / TypeScript
node_modules/
*.tsbuildinfo

# Go (test/coverage output; binaries usually have no extension)
*.test
*.out

# Ruby
.bundle/
vendor/bundle/

# C/C++ object & archive output
*.o
*.obj
*.a
*.so
```

- [ ] **Step 2: Register it as package data**

In `pyproject.toml`, append `"data/gitignore.baseline"` to the `vergil_tooling = [...]` list under `[tool.setuptools.package-data]` (line ~90):

```toml
vergil_tooling = ["data/*.json", "data/*.md", "data/*.sh", "data/gitignore.baseline", "data/licenses/*.txt", "configs/*.yaml", "configs/*.toml", "configs/ruby/*.yml", "configs/cpp/.clang-format", "configs/cpp/.clang-tidy", "configs/cpp/*.txt", "configs/cpp/*.cfg", "configs/typescript/*.json", "configs/typescript/*.mjs"]
```

- [ ] **Step 3: Verify the integration (nothing currently-ignored is lost)**

Run this from the repo root to list every distinct non-comment `.gitignore` line across sibling checkouts, and eyeball for anything missing from the baseline:

Run: `find .. -maxdepth 3 -name .gitignore -not -path '*/.worktrees/*' -exec cat {} + | sed 's/#.*//' | sed 's/[[:space:]]*$//' | grep -v '^$' | sort -u`
Expected: every line printed is either already in the baseline or a deliberate repo-local extra. Add genuinely-universal missing lines to the baseline (canonical spelling).

- [ ] **Step 4: Commit**

```bash
git add src/vergil_tooling/data/gitignore.baseline pyproject.toml
git commit -m "feat(gitignore): add canonical baseline data asset (#311)"
```

---

### Task 2: `render_gitignore()` reads the asset

**Files:**

- Modify: `src/vergil_tooling/lib/repo_init.py:434` (`render_gitignore`)
- Test: `tests/lib/test_repo_init.py`

**Interfaces:**

- Consumes: `vergil_tooling.data/gitignore.baseline` (Task 1), via the existing `_load_data_file(filename: str) -> str` helper (`repo_init.py:284`).
- Produces: `render_gitignore() -> str` returning the baseline verbatim.

- [ ] **Step 1: Write the failing test**

In `tests/lib/test_repo_init.py`:

```python
from vergil_tooling.lib import repo_init


def test_render_gitignore_returns_packaged_baseline():
    rendered = repo_init.render_gitignore()
    # Reads the single source of truth, not a hardcoded string.
    assert ".venv/" in rendered
    assert "quality-ruff.json" in rendered
    assert "docs/site/site/" in rendered
    # It IS the packaged asset, byte-for-byte.
    import importlib.resources
    packaged = (
        importlib.resources.files("vergil_tooling.data")
        .joinpath("gitignore.baseline")
        .read_text(encoding="utf-8")
    )
    assert rendered == packaged
```

- [ ] **Step 2: Run test to verify it fails**

Run: `vrg-container-run -- uv run pytest tests/lib/test_repo_init.py::test_render_gitignore_returns_packaged_baseline -v`
Expected: FAIL — current `render_gitignore` returns a hardcoded string that omits `quality-ruff.json` / `docs/site/site/`.

- [ ] **Step 3: Replace the hardcoded body**

Replace `render_gitignore` (`repo_init.py:434`) with:

```python
def render_gitignore() -> str:
    """Render the baseline .gitignore from the packaged single source of truth.

    The baseline lives at ``vergil_tooling.data/gitignore.baseline`` so
    scaffolding and the audit check (`repo_config._check_gitignore`) share one
    definition and cannot diverge (epic vergil-project/.github#311).
    """
    return _load_data_file("gitignore.baseline")
```

Confirm `_load_data_file` (`repo_init.py:284`) reads from `vergil_tooling.data`; if it takes a bare filename and joins onto that package, no change is needed.

- [ ] **Step 4: Run test to verify it passes**

Run: `vrg-container-run -- uv run pytest tests/lib/test_repo_init.py::test_render_gitignore_returns_packaged_baseline -v`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add src/vergil_tooling/lib/repo_init.py tests/lib/test_repo_init.py
git commit -m "refactor(repo-init): render_gitignore reads the baseline asset (#311)"
```

---

### Task 3: `_load_gitignore_baseline` + `_gitignore_patterns` helpers

**Files:**

- Modify: `src/vergil_tooling/lib/repo_config.py`
- Test: `tests/lib/test_repo_config.py`

**Interfaces:**

- Produces:
  - `_load_gitignore_baseline() -> str` — reads the packaged asset.
  - `_gitignore_patterns(text: str) -> list[str]` — non-comment, non-blank, trailing-whitespace-trimmed lines. Consumed by Task 4.

- [ ] **Step 1: Write the failing test**

In `tests/lib/test_repo_config.py`:

```python
from vergil_tooling.lib import repo_config


def test_gitignore_patterns_strips_comments_blanks_and_trailing_ws():
    text = "# comment\n\n.venv/  \n  # indented comment\nbuild/\n"
    assert repo_config._gitignore_patterns(text) == [".venv/", "build/"]


def test_load_gitignore_baseline_has_required_patterns():
    patterns = repo_config._gitignore_patterns(repo_config._load_gitignore_baseline())
    for required in (".venv/", ".worktrees/", "quality-ruff.json", "docs/site/site/"):
        assert required in patterns
```

- [ ] **Step 2: Run test to verify it fails**

Run: `vrg-container-run -- uv run pytest tests/lib/test_repo_config.py -k gitignore_patterns -v`
Expected: FAIL — `AttributeError: module has no attribute '_gitignore_patterns'`.

- [ ] **Step 3: Add the helpers**

In `repo_config.py` (near `_load_settings_template`):

```python
def _load_gitignore_baseline() -> str:
    return (
        importlib.resources.files("vergil_tooling.data")
        .joinpath("gitignore.baseline")
        .read_text(encoding="utf-8")
    )


def _gitignore_patterns(text: str) -> list[str]:
    """Baseline pattern lines: non-comment, non-blank, trailing-ws trimmed.

    Comments and blanks in the baseline are documentation, not requirements
    (spec §2, resolved O2).
    """
    patterns: list[str] = []
    for raw in text.splitlines():
        line = raw.rstrip()
        if not line or line.lstrip().startswith("#"):
            continue
        patterns.append(line)
    return patterns
```

- [ ] **Step 4: Run test to verify it passes**

Run: `vrg-container-run -- uv run pytest tests/lib/test_repo_config.py -k gitignore_patterns -v`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add src/vergil_tooling/lib/repo_config.py tests/lib/test_repo_config.py
git commit -m "feat(github-config): add gitignore baseline loader + pattern parser (#311)"
```

---

### Task 4: `_check_gitignore` superset check + register in `audit_local_config`

**Files:**

- Modify: `src/vergil_tooling/lib/repo_config.py` (`audit_local_config:49`)
- Test: `tests/lib/test_repo_config.py`

**Interfaces:**

- Consumes: `_load_gitignore_baseline`, `_gitignore_patterns` (Task 3); `DiffItem` (from `github_config`, already imported).
- Produces: `_check_gitignore(repo_root: Path, items: list[DiffItem]) -> None`; registered in `audit_local_config`.

- [ ] **Step 1: Write the failing tests**

In `tests/lib/test_repo_config.py`:

```python
from pathlib import Path
from vergil_tooling.lib import repo_config


def _write(p: Path, text: str) -> None:
    p.write_text(text, encoding="utf-8")


def test_check_gitignore_superset_passes(tmp_path: Path):
    baseline = repo_config._load_gitignore_baseline()
    _write(tmp_path / ".gitignore", baseline + "\n# local\nmy-local-thing/\n")
    items: list = []
    repo_config._check_gitignore(tmp_path, items)
    assert items == []


def test_check_gitignore_missing_pattern_fails(tmp_path: Path):
    _write(tmp_path / ".gitignore", ".venv/\n")  # missing almost everything
    items: list = []
    repo_config._check_gitignore(tmp_path, items)
    fields = {i.field for i in items}
    expecteds = {i.expected for i in items}
    assert fields == {"local.gitignore"}
    assert ".worktrees/" in expecteds
    assert all(i.actual == "missing" for i in items)


def test_check_gitignore_absent_file_reports_all(tmp_path: Path):
    items: list = []
    repo_config._check_gitignore(tmp_path, items)
    required = repo_config._gitignore_patterns(repo_config._load_gitignore_baseline())
    assert len(items) == len(required)


def test_check_gitignore_trailing_whitespace_tolerated(tmp_path: Path):
    baseline = repo_config._load_gitignore_baseline()
    _write(tmp_path / ".gitignore", "\n".join(l + "   " for l in baseline.splitlines()))
    items: list = []
    repo_config._check_gitignore(tmp_path, items)
    assert items == []
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `vrg-container-run -- uv run pytest tests/lib/test_repo_config.py -k check_gitignore -v`
Expected: FAIL — `_check_gitignore` not defined.

- [ ] **Step 3: Implement the check and register it**

Add to `repo_config.py`:

```python
def _check_gitignore(repo_root: Path, items: list[DiffItem]) -> None:
    """Require the repo .gitignore to be a superset of the baseline.

    Every baseline pattern line (Task 3) must appear verbatim as a line in the
    repo's .gitignore (trailing whitespace trimmed on both sides). Repos may add
    any extra lines. Matching is verbatim by design — the baseline defines the
    one canonical spelling per pattern and the fleet is standardized to it
    (spec §2). (#311)
    """
    required = _gitignore_patterns(_load_gitignore_baseline())
    gitignore = repo_root / ".gitignore"
    if not gitignore.is_file():
        present: set[str] = set()
    else:
        present = {line.rstrip() for line in gitignore.read_text(encoding="utf-8").splitlines()}
    for pattern in required:
        if pattern not in present:
            items.append(
                DiffItem(field="local.gitignore", expected=pattern, actual="missing")
            )
```

Register it in `audit_local_config` (`repo_config.py:49`):

```python
    _check_claude_settings(repo_root, items, warnings)
    _check_workflow_refs(repo_root, items)
    _check_gitignore(repo_root, items)
    return ConfigDiff(items=items, warnings=warnings)
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `vrg-container-run -- uv run pytest tests/lib/test_repo_config.py -k check_gitignore -v`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add src/vergil_tooling/lib/repo_config.py tests/lib/test_repo_config.py
git commit -m "feat(github-config): audit .gitignore is a superset of the baseline (#311)"
```

---

### Task 5: `render_ops_workflow` + staggered cron, written by `repo_init`

**Files:**

- Modify: `src/vergil_tooling/lib/repo_init.py` (add `_ops_cron_minute`, `render_ops_workflow`; write in `step_ci_cd_workflows:1255`)
- Test: `tests/lib/test_repo_init.py`

**Interfaces:**

- Consumes: `RepoInitContext` (`org`, `name`).
- Produces: `_ops_cron_minute(org: str, name: str) -> int`; `render_ops_workflow(ctx: RepoInitContext) -> str`.

- [ ] **Step 1: Write the failing tests**

In `tests/lib/test_repo_init.py`:

```python
from vergil_tooling.lib import repo_init
from vergil_tooling.lib.repo_init import RepoInitContext


def test_ops_cron_minute_deterministic_and_in_range():
    a = repo_init._ops_cron_minute("vergil-project", "vergil-tooling")
    b = repo_init._ops_cron_minute("vergil-project", "vergil-tooling")
    assert a == b
    assert 0 <= a <= 59


def test_ops_cron_minute_varies_by_repo():
    minutes = {
        repo_init._ops_cron_minute("vergil-project", n)
        for n in ("vergil-tooling", "vergil-actions", "vergil-vm", "vergil-containers")
    }
    assert len(minutes) >= 2  # spread, not all identical


def test_render_ops_workflow_wires_audit_with_staggered_cron():
    ctx = RepoInitContext(org="vergil-project", name="vergil-actions")
    yaml = repo_init.render_ops_workflow(ctx)
    minute = repo_init._ops_cron_minute("vergil-project", "vergil-actions")
    assert f"- cron: '{minute} 6 * * *'" in yaml
    assert "ops-github-config.yml@v2.1" in yaml
    assert "workflow_dispatch:" in yaml
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `vrg-container-run -- uv run pytest tests/lib/test_repo_init.py -k ops -v`
Expected: FAIL — `_ops_cron_minute` / `render_ops_workflow` not defined.

- [ ] **Step 3: Implement (add near the other `render_*_workflow` functions)**

```python
import hashlib


def _ops_cron_minute(org: str, name: str) -> int:
    """Deterministic per-repo minute in [0,59] for the ops schedule.

    A stable hash of ``<org>/<name>`` spreads scheduled runs across the hour so
    the fleet does not stampede a single minute as it grows (spec §4, O3). Cron
    is static YAML, so "randomize" means "derive deterministically per repo."
    """
    digest = hashlib.sha256(f"{org}/{name}".encode()).digest()
    return digest[0] % 60


def render_ops_workflow(ctx: RepoInitContext) -> str:
    """Render .github/workflows/ops.yml — the daily config-audit caller.

    Scheduled minute is staggered per repo (`_ops_cron_minute`); the reusable
    workflow is pinned to the rolling @v2.1 tag (epic vergil-project/.github#311).
    """
    minute = _ops_cron_minute(ctx.org, ctx.name)
    return (
        "name: Ops\n"
        "\n"
        "on:\n"
        "  schedule:\n"
        f"    - cron: '{minute} 6 * * *'\n"
        "  workflow_dispatch:\n"
        "\n"
        "permissions:\n"
        "  contents: read\n"
        "  issues: write\n"
        "\n"
        "jobs:\n"
        "  github-config:\n"
        "    uses: vergil-project/vergil-actions/.github/workflows/ops-github-config.yml@v2.1\n"
        "    permissions:\n"
        "      contents: read\n"
        "      issues: write\n"
        "    secrets:\n"
        "      APP_CLIENT_ID: ${{ secrets.APP_CLIENT_ID }}\n"
        "      APP_PRIVATE_KEY: ${{ secrets.APP_PRIVATE_KEY }}\n"
    )
```

Write it in `step_ci_cd_workflows` (after the epic-rollup write, `repo_init.py:1271`):

```python
    (workflows_dir / "epic-rollup.yml").write_text(render_epic_rollup_workflow())
    (workflows_dir / "ops.yml").write_text(render_ops_workflow(ctx))
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `vrg-container-run -- uv run pytest tests/lib/test_repo_init.py -k ops -v`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add src/vergil_tooling/lib/repo_init.py tests/lib/test_repo_init.py
git commit -m "feat(repo-init): scaffold ops.yml with staggered per-repo cron (#311)"
```

---

### Task 6: `_check_required_workflows` (ops.yml presence-and-wiring) + register

**Files:**

- Modify: `src/vergil_tooling/lib/repo_config.py` (`audit_local_config:49`)
- Test: `tests/lib/test_repo_config.py`

**Interfaces:**

- Produces: `_check_required_workflows(repo_root: Path, items: list[DiffItem]) -> None`; registered in `audit_local_config`.

- [ ] **Step 1: Write the failing tests**

```python
_WIRED = "  github-config:\n    uses: vergil-project/vergil-actions/.github/workflows/ops-github-config.yml@v2.1\n"
_SCHED = "on:\n  schedule:\n    - cron: '7 6 * * *'\n  workflow_dispatch:\n"


def _ops(dir_: Path, body: str) -> None:
    wf = dir_ / ".github" / "workflows"
    wf.mkdir(parents=True, exist_ok=True)
    (wf / "ops.yml").write_text(body, encoding="utf-8")


def test_required_workflows_present_wired_scheduled_passes(tmp_path: Path):
    _ops(tmp_path, _SCHED + "\njobs:\n" + _WIRED)
    items: list = []
    repo_config._check_required_workflows(tmp_path, items)
    assert items == []


def test_required_workflows_absent_fails(tmp_path: Path):
    items: list = []
    repo_config._check_required_workflows(tmp_path, items)
    assert [i.field for i in items] == ["local.ops_workflow"]
    assert items[0].actual == "missing"


def test_required_workflows_present_but_not_wired_fails(tmp_path: Path):
    _ops(tmp_path, _SCHED + "\njobs:\n  x:\n    uses: vergil-project/vergil-actions/.github/workflows/ci.yml@v2.1\n")
    items: list = []
    repo_config._check_required_workflows(tmp_path, items)
    assert all(i.field == "local.ops_workflow" for i in items)
    assert any("does not wire" in i.actual for i in items)


def test_required_workflows_wired_but_unscheduled_fails(tmp_path: Path):
    _ops(tmp_path, "on:\n  workflow_dispatch:\n\njobs:\n" + _WIRED)
    items: list = []
    repo_config._check_required_workflows(tmp_path, items)
    assert all(i.field == "local.ops_workflow" for i in items)
    assert any("no schedule" in i.actual for i in items)
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `vrg-container-run -- uv run pytest tests/lib/test_repo_config.py -k required_workflows -v`
Expected: FAIL — not defined.

- [ ] **Step 3: Implement and register**

```python
def _check_required_workflows(repo_root: Path, items: list[DiffItem]) -> None:
    """Assert .github/workflows/ops.yml exists, wires the audit, and is scheduled.

    A WIRING validator: it verifies a *present* ops.yml (a) calls the reusable
    ops-github-config workflow and (b) carries a scheduled (cron) trigger, so a
    wired-but-unscheduled ops.yml can't silently never run nightly. It cannot
    detect a repo missing ops.yml entirely (no workflow -> no nightly run ->
    this check never fires there); that from-outside guarantee is deferred to
    follow-on C (#315). (#311)
    """
    ops = repo_root / ".github" / "workflows" / "ops.yml"
    if not ops.is_file():
        items.append(
            DiffItem(field="local.ops_workflow", expected="present", actual="missing")
        )
        return
    content = ops.read_text(encoding="utf-8")
    if "ops-github-config.yml" not in content:
        items.append(
            DiffItem(
                field="local.ops_workflow",
                expected="calls ops-github-config.yml",
                actual="ops.yml present but does not wire the config audit",
            )
        )
    if "cron:" not in content:
        items.append(
            DiffItem(
                field="local.ops_workflow",
                expected="scheduled (cron) trigger",
                actual="ops.yml present but has no schedule",
            )
        )
```

Register in `audit_local_config`, after `_check_gitignore`:

```python
    _check_gitignore(repo_root, items)
    _check_required_workflows(repo_root, items)
    return ConfigDiff(items=items, warnings=warnings)
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `vrg-container-run -- uv run pytest tests/lib/test_repo_config.py -k required_workflows -v`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add src/vergil_tooling/lib/repo_config.py tests/lib/test_repo_config.py
git commit -m "feat(github-config): audit ops.yml presence-and-wiring (#311)"
```

---

### Task 7: Round-trip + full validation

**Files:**

- Test: `tests/lib/test_repo_init.py`

- [ ] **Step 1: Write the round-trip test**

A freshly scaffolded repo must pass both new checks:

```python
def test_scaffolded_repo_passes_new_audit_checks(tmp_path: Path):
    from vergil_tooling.lib import repo_config
    (tmp_path / ".gitignore").write_text(repo_init.render_gitignore(), encoding="utf-8")
    ctx = RepoInitContext(org="vergil-project", name="demo")
    wf = tmp_path / ".github" / "workflows"
    wf.mkdir(parents=True, exist_ok=True)
    (wf / "ops.yml").write_text(repo_init.render_ops_workflow(ctx), encoding="utf-8")
    items: list = []
    repo_config._check_gitignore(tmp_path, items)
    repo_config._check_required_workflows(tmp_path, items)
    assert items == []
```

- [ ] **Step 2: Run it**

Run: `vrg-container-run -- uv run pytest tests/lib/test_repo_init.py::test_scaffolded_repo_passes_new_audit_checks -v`
Expected: PASS

- [ ] **Step 3: Run the FULL validation pipeline**

Run: `vrg-container-run -- vrg-validate`
Expected: PASS, including `--cov-fail-under=100`. If a `# pragma: no cover` is warranted (e.g. a packaging-error guard), match the existing style in `repo_config.py`.

- [ ] **Step 4: Reconcile this repo's own `.gitignore`**

`vergil-tooling`'s own `.gitignore` must be a superset of the new baseline. Verify and, if needed, rewrite it to carry every canonical baseline line (standardize spellings — do not append duplicates):

Run: `vrg-container-run -- uv run python -c "from pathlib import Path; from vergil_tooling.lib import repo_config; items=[]; repo_config._check_gitignore(Path('.'), items); print([i.expected for i in items])"`
Expected: `[]`. If not, edit `.gitignore` to add the missing canonical lines, then re-run.

- [ ] **Step 5: Commit**

```bash
git add tests/lib/test_repo_init.py .gitignore
git commit -m "test(github-config): round-trip scaffolded repo passes new checks (#311)"
```

- [ ] **Step 6: Hand off the PR** (agent stops here)

Record the PR with `vrg-pr-workflow report-ready --issue <impl-issue> --title ... --summary ... --notes ...`. **Do not** run `vrg-submit-pr` — the human submits and merges. A **`vergil-tooling` release under `v2.1`** must follow (human-gated) before any rollout task runs — it is the precondition that puts the new baseline on the rolling tag every repo tracks.

---

## Fleet rollout (operational tasks — filed after the tooling PR merges + releases)

These are **not** TDD code tasks; they are `deployment`-kind operational tasks (spec "Enforcement & fleet rollout"). Each is filed with `vrg-issue-create --kind deployment` under the epic, in the repo it targets. **Precondition self-check for every one (resolved O4):** run `vrg-github-repo-config audit` on the repo first; if it fails for pre-existing reasons, **fix that non-compliance as part of this task** before adding `ops.yml`. Never enable the nightly audit on a repo that will red for unrelated drift.

Each rollout task's body:

1. Verify readiness: `vrg-github-repo-config audit --repo <owner>/<repo>` (fix any pre-existing findings).
2. Reconcile `.gitignore`: rewrite to a superset of the baseline (standardize spellings; the check is verbatim). Use `vrg-github-repo-config diff`/the `repo-init adopt` managed-file regeneration where it applies.
3. Add `.github/workflows/ops.yml` (staggered cron) if absent — mirror `render_ops_workflow`.
4. Confirm a clean audit, then close on `Outcome: SUCCESS`.

**vergil-project targets:**

- `vergil-actions` — no `ops.yml` today: reconcile `.gitignore` + add `ops.yml`.
- `vergil-claude-plugin` — no `ops.yml` today: reconcile `.gitignore` + add `ops.yml`.
- `vergil-vm` — no `ops.yml` today: reconcile `.gitignore` + add `ops.yml`.
- `vergil-project/.github` — no `ops.yml` today: reconcile `.gitignore` + add `ops.yml` (treated as a managed member — verify O4 readiness).
- `vergil-containers` — has `ops.yml`: reconcile `.gitignore` only + restagger its cron minute.
- `vergil-tooling` — reconciled in Task 7 of the tooling PR; restagger its cron minute here if not done there.

---

## Cross-org follow-on rollout epics (this epic's closing work, before the retrospective)

Cross-org linking is banned, so each other org gets its **own** rollout epic in its own `.github`, tracked from #311's retrospective §5 (not linked). Stand these up via `vrg-epic-create --repo <org>/<repo>`; each is a set of `deployment` tasks identical in shape to the vergil-project rollout above.

- **`logical-minds-foundry`** — create + plan + **work aggressively**.
- **`mnemosys-project`** — create + plan + **work aggressively**.
- **`mq-rest-admin-project`** — create + plan, then **park**.

---

## Implementation issues to file (epic-create step 9)

File these under epic `vergil-project/.github#311`, each in the repo where its PR/outcome lands:

1. **`vergil-tooling`** — *impl* task: "Centralize baseline `.gitignore` + add `_check_gitignore`/`_check_required_workflows` + scaffold `ops.yml`" (this whole plan; one PR).
2. **`vergil-project/.github` release** — human-gated: release `vergil-tooling` under `v2.1` after (1) merges (gates all rollout — record as the deployment precondition, not agent-run).
3. **`deployment` tasks** (one per vergil-project target above), each `--kind deployment`, `--blocked-by` the impl task (and the release).
4. The three cross-org rollout epics are **not** filed as tasks here (cross-org); they are stood up as closing work and recorded in the retrospective.

---

## Self-review

**Spec coverage:**

- §1 baseline asset → Task 1; `render_gitignore` reads it → Task 2. ✓
- §2 superset semantics (verbatim, comments ignored, trailing-ws trim) → Tasks 3–4. ✓
- §3 both audit checks registered in `audit_local_config` → Tasks 4, 6. ✓
- §4 `repo_init` scaffolds `ops.yml` + staggered cron (O3) → Task 5. ✓
- §5 rolling `vX.Y` propagation → `@v2.1` pins in Tasks 1/5, release gate in Task 7/issue 2. ✓
- Enforcement & rollout + O4 readiness → Rollout section. ✓
- Cross-org epics → Cross-org section. ✓
- Follow-ons B (#313) / C (#315) → already seeded; not implemented here (correct). ✓
- O1 `docs/site/site/` → baseline content, Task 1. ✓  O2 comments → Task 3. ✓

**Placeholder scan:** no TBD/TODO; every code and test step carries real content. ✓

**Type consistency:** `DiffItem(field=, expected=, actual=)` matches existing usage; `_gitignore_patterns`/`_load_gitignore_baseline`/`_check_gitignore`/`_check_required_workflows`/`_ops_cron_minute`/`render_ops_workflow` names are used identically across tasks. ✓
