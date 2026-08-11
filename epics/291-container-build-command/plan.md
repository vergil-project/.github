# `[container].build-command` Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add a `[container].build-command` hook (plus a `build-cache-files`
companion key) that bakes a repo's non-apt dependencies into its dev/CI container
the same way `[container].system-packages` bakes apt packages.

**Architecture:** Generalize the existing per-repo cache-build setup step
(`container_cache.py`) to run one repo-declared shell command after the
vergil-tooling install and before the language warmup, install its artifacts
outside the bind-mounted `/workspace`, and fold the command's declared input files
into the image cache hash. A new `vrg-*` speller emits the command for CI, where a
new fail-closed (no-retry) vergil-actions composite step runs it on test jobs.

**Tech Stack:** Python 3.14 (vergil-tooling), pytest, TOML config; Bash composite
GitHub Actions (vergil-actions); Docker cache-build.

**Direct precedent:** epic vergil-project/.github#272 (`system-packages`). Mirror
its shape everywhere; this plan calls out only the deltas.

## Global Constraints

- **Single reader per key.** `container_build_command(repo_root)` is the only
  reader of `build-command`; `container_build_cache_files(repo_root)` the only
  reader of `build-cache-files`. Local bake and CI both go through them — never
  re-parse `vergil.toml`.
- **Fail-closed, no silent skip.** A non-zero `build-command` fails the build
  (local) and the job (CI). No `except: pass`, no degraded image.
- **Out-of-workspace contract.** The command must install *outside* `/workspace`
  (e.g. `npm install -g`); workspace-path artifacts are masked by the runtime
  bind-mount (`container.py:179`). This is a documented contract, not enforced by
  tooling.
- **Order: after the vergil-tooling install, before warmup — in BOTH paths.** So
  the command's environment cannot diverge local-vs-CI.
- **Empty/absent ⇒ byte-identical to today.** No key ⇒ no setup fragment, no extra
  hash input.
- **Test-runtime only in CI.** The CI step runs on test jobs, never lint/typecheck.
- **Repo tooling:** commit with `vrg-commit`; git via `vrg-git`; validate with
  `vrg-container-run -- vrg-validate`. Targeted pytest runs go through the dev
  container (`vrg-container-run -- uv run pytest …`).

## Task → issue/repo map

Each task below is filed (workflow step 9) as one GitHub issue, in the repo where
its PR lands, linked under epic vergil-project/.github#291.

| Task | Repo | Kind | Depends on |
|---|---|---|---|
| 1 Config surface | vergil-tooling | task | — |
| 2 Local bake + cache key | vergil-tooling | task | 1 |
| 3 CI speller entry point | vergil-tooling | task | 1 |
| 4 vergil-actions CI step | vergil-actions | task | 3 (released) |
| 5 Cold-rebuild validation | vergil-tooling | validation | 2, 4 |

**Sequencing (mirrors #272):** vergil-tooling (1→2, 1→3) lands and is released;
then vergil-actions (4) consumes the speller; validation (5) runs last. Tasks 1–3
can be three PRs or folded into fewer at the implementer's discretion, but 2 and 3
both depend on 1's accessors.

---

### Task 1: Config surface — `build-command` + `build-cache-files`

**Files:**
- Modify: `src/vergil_tooling/lib/config.py` (`_KNOWN_KEYS`, `ContainerConfig`,
  the `container_raw` parse block ~566–583, accessors ~684–703)
- Test: `tests/lib/test_config.py`

**Interfaces:**
- Produces:
  - `ContainerConfig.build_command: str | None` (default `None`)
  - `ContainerConfig.build_cache_files: list[str]` (default `[]`)
  - `container_build_command(repo_root: Path) -> str | None`
  - `container_build_cache_files(repo_root: Path) -> list[str]`

- [ ] **Step 1: Write the failing tests**

```python
# tests/lib/test_config.py
def test_container_build_command_parsed(tmp_path):
    (tmp_path / "vergil.toml").write_text(
        '[container]\n'
        'env-prefixes = []\n'
        'build-command = "npm install -g @coderline/alphatab"\n'
        'build-cache-files = ["melete-render/package-lock.json"]\n'
    )
    from vergil_tooling.lib.config import (
        container_build_command,
        container_build_cache_files,
    )
    assert container_build_command(tmp_path) == "npm install -g @coderline/alphatab"
    assert container_build_cache_files(tmp_path) == ["melete-render/package-lock.json"]


def test_container_build_command_defaults_when_absent(tmp_path):
    (tmp_path / "vergil.toml").write_text('[container]\nenv-prefixes = []\n')
    from vergil_tooling.lib.config import (
        container_build_command,
        container_build_cache_files,
    )
    assert container_build_command(tmp_path) is None
    assert container_build_cache_files(tmp_path) == []


def test_container_build_command_defaults_when_no_toml(tmp_path):
    from vergil_tooling.lib.config import (
        container_build_command,
        container_build_cache_files,
    )
    assert container_build_command(tmp_path) is None
    assert container_build_cache_files(tmp_path) == []


def test_container_build_command_must_be_string(tmp_path):
    (tmp_path / "vergil.toml").write_text(
        '[container]\nenv-prefixes = []\nbuild-command = ["a", "b"]\n'
    )
    from vergil_tooling.lib.config import read_config, ConfigError
    import pytest
    with pytest.raises(ConfigError, match="build-command must be a string"):
        read_config(tmp_path)


def test_container_build_cache_files_must_be_list_of_strings(tmp_path):
    (tmp_path / "vergil.toml").write_text(
        '[container]\nenv-prefixes = []\nbuild-cache-files = "lock"\n'
    )
    from vergil_tooling.lib.config import read_config, ConfigError
    import pytest
    with pytest.raises(ConfigError, match="build-cache-files must be a list of strings"):
        read_config(tmp_path)
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `vrg-container-run -- uv run pytest tests/lib/test_config.py -k build_command -v`
Expected: FAIL (ImportError / AttributeError — accessors and fields not defined).

- [ ] **Step 3: Extend `_KNOWN_KEYS` and `ContainerConfig`**

```python
# _KNOWN_KEYS["container"]:
"container": frozenset({"env-prefixes", "system-packages", "build-command", "build-cache-files"}),

# ContainerConfig dataclass — add fields:
@dataclass
class ContainerConfig:
    env_prefixes: list[str]
    system_packages: list[str]
    # A single shell command run in the container setup step after the
    # vergil-tooling install, before warmup (epic vergil-project/.github#291).
    # Must install artifacts OUTSIDE /workspace (masked by the runtime mount).
    build_command: str | None
    # Repo-relative files the build-command reads; folded into the image cache
    # hash so a dependency bump rebuilds. Default [].
    build_cache_files: list[str]
```

- [ ] **Step 4: Parse + validate in the `container_raw` block**

```python
    container_raw = raw.get("container")
    if container_raw is not None:
        env_prefixes = container_raw.get("env-prefixes")
        if env_prefixes is None:
            msg = f"{source}: [container] missing required field 'env-prefixes'"
            raise ConfigError(msg)
        if not isinstance(env_prefixes, list) or not all(isinstance(p, str) for p in env_prefixes):
            msg = f"{source}: [container].env-prefixes must be a list of strings"
            raise ConfigError(msg)
        system_packages = container_raw.get("system-packages", [])
        if not isinstance(system_packages, list) or not all(
            isinstance(p, str) for p in system_packages
        ):
            msg = f"{source}: [container].system-packages must be a list of strings"
            raise ConfigError(msg)
        build_command = container_raw.get("build-command")
        if build_command is not None and not isinstance(build_command, str):
            msg = f"{source}: [container].build-command must be a string"
            raise ConfigError(msg)
        build_cache_files = container_raw.get("build-cache-files", [])
        if not isinstance(build_cache_files, list) or not all(
            isinstance(p, str) for p in build_cache_files
        ):
            msg = f"{source}: [container].build-cache-files must be a list of strings"
            raise ConfigError(msg)
        container = ContainerConfig(
            env_prefixes=env_prefixes,
            system_packages=system_packages,
            build_command=build_command,
            build_cache_files=build_cache_files,
        )
    else:
        container = ContainerConfig(
            env_prefixes=[], system_packages=[], build_command=None, build_cache_files=[]
        )
```

- [ ] **Step 5: Add the accessors** (next to `container_system_packages`)

```python
def container_build_command(repo_root: Path) -> str | None:
    """Return ``[container].build-command`` from vergil.toml, or ``None``.

    The single reader of the key; the local cache build and the CI setup step
    both resolve the command through this. Must install outside /workspace.
    """
    try:
        cfg = read_config(repo_root)
    except FileNotFoundError:
        return None
    return cfg.container.build_command


def container_build_cache_files(repo_root: Path) -> list[str]:
    """Return ``[container].build-cache-files`` from vergil.toml, or ``[]``.

    Repo-relative files the build-command reads; folded into the image cache
    hash so a dependency bump rebuilds.
    """
    try:
        cfg = read_config(repo_root)
    except FileNotFoundError:
        return []
    return cfg.container.build_cache_files
```

- [ ] **Step 6: Run tests to verify they pass**

Run: `vrg-container-run -- uv run pytest tests/lib/test_config.py -k build_command -v`
Expected: PASS (all five).

- [ ] **Step 7: Update every other `ContainerConfig(...)` construction**

Search: `vrg-git grep -n "ContainerConfig("` — any construction missing the two new
fields fails typecheck. Add `build_command=None, build_cache_files=[]` to each.

- [ ] **Step 8: Full validation + commit**

Run: `vrg-container-run -- vrg-validate`
Then:
```bash
vrg-commit --type feat --scope config \
  --message "add [container].build-command + build-cache-files keys" \
  --body "New optional [container] keys (epic vergil-project/.github#291): build-command (a single shell string) and build-cache-files (its input files). Adds ContainerConfig fields, strict validation mirroring system-packages, and the single-reader accessors container_build_command / container_build_cache_files."
```

---

### Task 2: Local path — bake the command + fold cache-key inputs

**Files:**
- Modify: `src/vergil_tooling/lib/container_cache.py`
  (`_build_cached_image` ~249–348; `cache_sensitive_files` ~188–191)
- Test: `tests/lib/test_container_cache.py`

**Interfaces:**
- Consumes: `container_build_command`, `container_build_cache_files` (Task 1).
- Produces: a `setup` string of the form
  `<apt> && <uv tool install …> && <build-command> && <warmup>` (build-command
  slotted after the vergil-tooling install); `cache_sensitive_files` additionally
  returns the declared `build-cache-files` paths that exist.

- [ ] **Step 1: Write the failing tests**

```python
# tests/lib/test_container_cache.py
def test_build_command_slotted_after_install_before_warmup(tmp_path, monkeypatch):
    # A consumer repo (not self): setup = apt && uv_install && build && warmup
    from vergil_tooling.lib import container_cache as cc
    monkeypatch.setattr(cc, "_is_self_repo", lambda root: False)
    monkeypatch.setattr(cc, "_warmup_command", lambda lang: "uv sync --frozen")
    monkeypatch.setattr(cc, "vrg_install_tag", lambda root: "v2.1")
    from vergil_tooling.lib import config as cfg
    monkeypatch.setattr(cfg, "container_system_packages", lambda root: [])
    monkeypatch.setattr(cfg, "container_build_command", lambda root: "npm install -g x")
    setup = cc._compose_setup(tmp_path, "python")  # extracted pure helper (Step 3)
    assert setup.index("uv tool install") < setup.index("npm install -g x")
    assert setup.index("npm install -g x") < setup.index("uv sync --frozen")


def test_build_command_absent_is_byte_identical(tmp_path, monkeypatch):
    from vergil_tooling.lib import container_cache as cc
    from vergil_tooling.lib import config as cfg
    monkeypatch.setattr(cc, "_is_self_repo", lambda root: False)
    monkeypatch.setattr(cc, "_warmup_command", lambda lang: "uv sync --frozen")
    monkeypatch.setattr(cc, "vrg_install_tag", lambda root: "v2.1")
    monkeypatch.setattr(cfg, "container_system_packages", lambda root: [])
    monkeypatch.setattr(cfg, "container_build_command", lambda root: None)
    setup = cc._compose_setup(tmp_path, "python")
    assert "&&" in setup  # uv_install && warmup
    assert setup == "uv tool install --quiet 'vergil-tooling @ git+" \
        "https://github.com/vergil-project/vergil-tooling@v2.1' && uv sync --frozen"


def test_build_cache_files_included_when_present(tmp_path, monkeypatch):
    (tmp_path / "vergil.toml").write_text("[container]\nenv-prefixes=[]\n")
    (tmp_path / "lock.json").write_text("{}")
    from vergil_tooling.lib import container_cache as cc
    from vergil_tooling.lib import config as cfg
    monkeypatch.setattr(cfg, "container_build_cache_files", lambda root: ["lock.json", "absent.json"])
    files = cc.cache_sensitive_files(tmp_path, "python")
    assert (tmp_path / "lock.json") in files
    assert (tmp_path / "absent.json") not in files  # only existing files


def test_editing_build_cache_file_changes_hash(tmp_path):
    from vergil_tooling.lib.container_cache import compute_cache_hash
    f = tmp_path / "lock.json"
    f.write_text("v1")
    h1 = compute_cache_hash([f])
    f.write_text("v2")
    h2 = compute_cache_hash([f])
    assert h1 != h2
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `vrg-container-run -- uv run pytest tests/lib/test_container_cache.py -k "build_command or build_cache" -v`
Expected: FAIL (`_compose_setup` not defined; `cache_sensitive_files` ignores build-cache-files).

- [ ] **Step 3: Extract `_compose_setup` and slot the build-command**

Refactor the setup construction out of `_build_cached_image` into a pure helper so
it is unit-testable, and insert the build-command after the install:

```python
def _compose_setup(repo_root: Path, lang: str) -> str:
    from vergil_tooling.lib.config import container_build_command, container_system_packages

    warmup = _warmup_command(lang)
    self_repo = _is_self_repo(repo_root)
    if self_repo:
        base = warmup or "true"
    else:
        tag = vrg_install_tag(repo_root)
        uv_install = f"uv tool install --quiet 'vergil-tooling @ git+{_VRG_GIT_URL}@{tag}'"
        base = f"{uv_install} && {warmup}" if warmup else uv_install

    # Repo build-command: after the vergil-tooling install, before warmup, so the
    # command's environment matches CI (epic #291). Must install outside /workspace.
    build = container_build_command(repo_root)
    if build:
        if self_repo:
            base = f"{build} && {warmup}" if warmup else build
        else:
            base = f"{uv_install} && {build}" + (f" && {warmup}" if warmup else "")

    apt = apt_install_command(container_system_packages(repo_root), container_platform())
    return f"{apt} && {base}" if apt else base
```

Replace the inline `setup = …` logic in `_build_cached_image` with
`setup = _compose_setup(repo_root, lang)`, and add the build-command to the
provisioning banner:

```python
    build = container_build_command(repo_root)
    if build:
        print(f"  Build:   {build}")
```

- [ ] **Step 4: Fold build-cache-files into `cache_sensitive_files`**

```python
def cache_sensitive_files(repo_root: Path, lang: str) -> list[Path]:
    """Return paths of cache-sensitive files that exist in *repo_root*.

    Includes the language cache set plus any [container].build-cache-files the
    repo declares (epic #291), so a build-command's inputs rebuild the image.
    """
    from vergil_tooling.lib.config import container_build_cache_files

    names = _CACHE_FILES.get(lang, _DEFAULT_CACHE_FILES)
    paths = [repo_root / n for n in names if (repo_root / n).is_file()]
    for rel in container_build_cache_files(repo_root):
        p = repo_root / rel
        if p.is_file() and p not in paths:
            paths.append(p)
    return paths
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `vrg-container-run -- uv run pytest tests/lib/test_container_cache.py -k "build_command or build_cache" -v`
Expected: PASS.

- [ ] **Step 6: Full validation + commit**

Run: `vrg-container-run -- vrg-validate`
Then:
```bash
vrg-commit --type feat --scope container \
  --message "bake [container].build-command into the cached image" \
  --body "Run the declared build-command in the cache-build setup step after the vergil-tooling install and before warmup (epic #291), and fold [container].build-cache-files into the cache hash so a dependency bump rebuilds. Extracts _compose_setup as a pure, tested helper; absent build-command is byte-identical to today. Fail-closed via the existing setup-step failure path."
```

---

### Task 3: CI speller entry point — `vrg-container-build-command`

**Files:**
- Create: `src/vergil_tooling/bin/vrg_container_build_command.py`
- Modify: `pyproject.toml` (`[project.scripts]`)
- Test: `tests/bin/test_vrg_container_build_command.py`

**Interfaces:**
- Consumes: `container_build_command` (Task 1).
- Produces: console script `vrg-container-build-command`; `--script` prints the
  command (empty output when none declared), default prints the command or nothing.

- [ ] **Step 1: Write the failing test**

```python
# tests/bin/test_vrg_container_build_command.py
def test_script_mode_prints_command(tmp_path, capsys):
    (tmp_path / "vergil.toml").write_text(
        '[container]\nenv-prefixes=[]\nbuild-command = "npm install -g x"\n'
    )
    from vergil_tooling.bin.vrg_container_build_command import main
    rc = main(["--script", "--repo-root", str(tmp_path)])
    assert rc == 0
    assert capsys.readouterr().out.strip() == "npm install -g x"


def test_script_mode_empty_when_absent(tmp_path, capsys):
    (tmp_path / "vergil.toml").write_text('[container]\nenv-prefixes=[]\n')
    from vergil_tooling.bin.vrg_container_build_command import main
    rc = main(["--script", "--repo-root", str(tmp_path)])
    assert rc == 0
    assert capsys.readouterr().out.strip() == ""
```

- [ ] **Step 2: Run test to verify it fails**

Run: `vrg-container-run -- uv run pytest tests/bin/test_vrg_container_build_command.py -v`
Expected: FAIL (ModuleNotFoundError).

- [ ] **Step 3: Write the entry point** (mirror `vrg_container_system_packages.py`)

```python
"""Print a repo's declared [container].build-command, for CI consumption.

--script prints the command verbatim (empty when none declared). The single
speller shared with the local cache build (epic vergil-project/.github#291).
"""
from __future__ import annotations

import argparse
import sys
from pathlib import Path

from vergil_tooling.lib.config import container_build_command


def main(argv: list[str] | None = None) -> int:
    parser = argparse.ArgumentParser(description="Print declared [container].build-command.")
    parser.add_argument(
        "--script", action="store_true",
        help="print the command for CI to run (empty when none declared)",
    )
    parser.add_argument("--repo-root", default=".", help="repo root (default: CWD)")
    args = parser.parse_args(argv)

    command = container_build_command(Path(args.repo_root))
    if command:
        print(command)
    return 0


if __name__ == "__main__":
    sys.exit(main())
```

- [ ] **Step 4: Register the console script**

```toml
# pyproject.toml [project.scripts], next to vrg-container-system-packages:
vrg-container-build-command = "vergil_tooling.bin.vrg_container_build_command:main"
```

- [ ] **Step 5: Run test to verify it passes**

Run: `vrg-container-run -- uv run pytest tests/bin/test_vrg_container_build_command.py -v`
Expected: PASS.

- [ ] **Step 6: Full validation + commit**

Run: `vrg-container-run -- vrg-validate`
Then:
```bash
vrg-commit --type feat --scope container \
  --message "add vrg-container-build-command speller for CI" \
  --body "New console script emitting [container].build-command via the single container_build_command reader (epic #291), so CI runs the identical command without parsing vergil.toml. --script mode prints the command (empty when none declared)."
```

---

### Task 4: CI path — new vergil-actions composite step (fail-closed, no retry)

**Repo:** vergil-actions. **Depends on** Task 3 being released and installed in CI
(the `Install vergil-tooling` step must provide `vrg-container-build-command`).

**Files:**
- Create: `actions/shared/setup/build-command/action.yml`
- Create: `actions/shared/setup/build-command/install.sh`
- Create: `actions/shared/setup/build-command/tests/install.test.sh`
- Modify: `.github/workflows/ci-test.yml` (unit job, after system-packages)
- Create: `docs/site/docs/actions/shared-setup-build-command.md` (+ mkdocs nav)

- [ ] **Step 1: Write the failing test harness** (mirror system-packages tests)

```bash
# actions/shared/setup/build-command/tests/install.test.sh
# PATH-stub vrg-container-build-command; assert:
#  - a declared command is executed (marker file created) → exit 0
#  - a failing command exits non-zero with NO retry (stub asserts single call)
#  - no command declared → "skipping" → exit 0
```

- [ ] **Step 2: Run it to verify it fails**

Run: `bash actions/shared/setup/build-command/tests/install.test.sh`
Expected: FAIL (install.sh absent).

- [ ] **Step 3: Write `install.sh`** (fail-closed, NO retry — the key delta from #272)

```bash
#!/usr/bin/env bash
#
# Run the command a repo declares in [container].build-command.
#
# Reads it from `vrg-container-build-command --script` — the single speller
# shared with the local dev-container cache build (epic vergil-project/.github#291).
# Fail-closed and NOT retried: unlike apt (mirror flake), an arbitrary build
# command that fails is a real failure, not a transient one.
#
# Test-runtime dependency: callers wire this onto jobs that execute the repo's
# tests only, never lint/typecheck (spec §3.3). Runs after Install vergil-tooling.
set -euo pipefail

script="$(vrg-container-build-command --script)"
if [ -z "${script}" ]; then
  echo "No [container].build-command declared; skipping."
  exit 0
fi

echo "Running [container].build-command: ${script}"
bash -c "${script}"
```

- [ ] **Step 4: Write `action.yml`** (thin wrapper, mirror system-packages)

```yaml
name: Run repo build-command
description: >-
  Run the command a repo declares in [container].build-command, read through
  vrg-container-build-command. Test-runtime dependency: runs only on jobs that
  execute the repo's tests, after Install vergil-tooling. Fail-closed, no retry
  (epic vergil-project/.github#291). Logic lives in install.sh (tests/install.test.sh).
runs:
  using: composite
  steps:
    - name: Run declared build-command
      shell: bash
      run: bash "${GITHUB_ACTION_PATH}/install.sh"
```

- [ ] **Step 5: Wire into `ci-test.yml`** — unit job, after system-packages (line ~75)

```yaml
      - name: Install repo system packages
        uses: ./actions/shared/setup/system-packages

      # Test-runtime dependency only (epic #291): run the declared
      # [container].build-command after vergil-tooling + system-packages, before
      # tests. Fail-closed, no retry.
      - name: Run repo build-command
        uses: ./actions/shared/setup/build-command
```

- [ ] **Step 6: Run the test harness to verify it passes**

Run: `bash actions/shared/setup/build-command/tests/install.test.sh`
Expected: PASS (execute / fail-no-retry / skip).

- [ ] **Step 7: Write the action doc + nav, then validate + commit**

Add `docs/site/docs/actions/shared-setup-build-command.md` (mirror
`shared-setup-system-packages.md`: what it reads, the single-speller note, the
fail-closed no-retry contract, test-runtime scoping) and its mkdocs nav entry.

Run: `vrg-container-run -- vrg-validate`
Then:
```bash
vrg-commit --type feat --scope actions \
  --message "add shared/setup/build-command CI step (fail-closed, no retry)" \
  --body "New composite action running [container].build-command via vrg-container-build-command --script on test jobs, after Install vergil-tooling and system-packages (epic vergil-project/.github#291). Fail-closed with no retry — an arbitrary build command that fails is a real failure, unlike an apt mirror flake. Logic in install.sh with a stubbed test harness; wired into ci-test.yml's unit job; action doc added."
```

---

### Task 5: Cold-rebuild validation (operational — `validation`)

**Repo:** vergil-tooling. **Kind:** `validation`. **Blocked-by:** Tasks 2 and 4.
Not PR-workable — run via `issue-validate`, record `Outcome: SUCCESS` as a comment.

**Precondition self-check:** the build-command mechanism (Task 2) is merged to
develop and the vergil-actions step (Task 4) is released; Docker is available. If
unmet, comment "blocked: preconditions not met" and stop.

**Procedure (clean-tree, out-of-workspace proof):**

1. In a scratch repo (or a disposable branch of a repo) declare a real
   out-of-workspace command:
   ```toml
   [container]
   env-prefixes = []
   build-command = "npm install -g @coderline/alphatab"
   build-cache-files = []
   ```
2. Force a cold rebuild: `clean_branch_images` for the branch (or remove the
   cached tag), then `vrg-container-run -- true` to trigger the bake. Confirm the
   provisioning banner shows the `Build:` line and the build succeeds.
3. **Clean-tree check:** ensure no host `node_modules` under the workspace
   (`vrg-git clean -ndx` shows none relevant), then:
   ```bash
   vrg-container-run -- node -e 'require.resolve("@coderline/alphatab")'
   ```
   Expected: resolves (exit 0) — proving the dependency is image-resident, not
   host-tree pollution.
4. **Cache-key check:** add a `build-cache-files` entry, edit that file, run
   `vrg-container-run`, and confirm a rebuild is triggered (banner reappears).
5. **Fail-closed check:** set `build-command = "exit 3"`; confirm the cache build
   fails loudly (non-zero) with the command output, and no image is committed.

**Acceptance (record as a comment):**
- `Outcome: SUCCESS` iff steps 2–5 all pass: cold build runs the command; the dep
  resolves on a clean tree; a build-cache-file edit rebuilds; a failing command
  fails closed. Any failure ⇒ leave open, record the failing step.

---

## Self-Review

**Spec coverage:**
- §3.1 config surface → Task 1. ✓
- §3.2 local bake, order-after-install, self-repo, fail-closed, byte-identical →
  Task 2. ✓
- §3.3 CI new step, single speller, fail-closed no-retry, test-jobs-only → Tasks 3
  (speller) + 4 (step). ✓ (O4 resolved: new step, not generalization.)
- §3.4 `build-cache-files` cache-key → Task 2 (fold) + Task 1 (key). ✓
- §3.5 trust model → docs in Task 4 + the key docs (doc-review bookend #2756). ✓
- §4 out-of-workspace contract → Global Constraints + Task 5 clean-tree proof. ✓
- §6 acceptance → Tasks 2/4 tests + Task 5 validation. ✓
- Site-docs narrative sweep → **not** an impl task; handled by the
  documentation-review bookend vergil-tooling#2756 (spawns per-repo siblings). The
  impl tasks carry only code-adjacent docs (the action doc, docstrings, key
  comments).

**Placeholder scan:** none — every step has concrete code, commands, and expected
results.

**Type consistency:** `container_build_command` / `container_build_cache_files`
and the `ContainerConfig.build_command` / `build_cache_files` fields are named
identically across Tasks 1–3 and the CI speller. `_compose_setup(repo_root, lang)`
is defined in Task 2 and referenced only there. ✓
