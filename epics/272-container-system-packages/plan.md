# Repo-specific system dependencies in dev containers — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.
>
> **Epic:** vergil-project/.github#272 · **Spec:** `epics/272-container-system-packages/spec.md`

**Goal:** Let a repo declare Debian system packages in `vergil.toml`
(`[container].system-packages`) and have them installed into both the local
cached dev image and CI test jobs, without touching the shared base images.

**Architecture:** One reader (`container_system_packages` in `config.py`, exposed
to CI via a `vrg-container-system-packages` accessor) and one shared install-command
speller (`apt_install_command`). The local path bakes the packages into the
per-branch cached image by extending the existing `_build_cached_image` `setup`
step; the CI path installs them per-run on the test job via a composite action
that calls the accessor. Fail-closed on a missing package; bounded retry in CI.

**Tech Stack:** Python 3.12+ (`tomllib`, `hashlib`, `subprocess`); GitHub Actions
composite actions (bash); pytest.

## Global Constraints

- **Scope: package names only.** A list of Debian package names installed with
  `apt-get install -y --no-install-recommends` from the base image's existing apt
  sources. No third-party sources, keys, `add-apt-repository`, or scripts.
- **Key name:** `system-packages`, under the existing `[container]` table.
- **One reader, one speller.** All install paths read the key through
  `config.container_system_packages` and spell the apt command through
  `apt_install_command`. No ad-hoc shell/`grep` parsing of `vergil.toml`.
- **Fail-closed, loud.** A package with no installation candidate for the build
  arch fails the build/step immediately, naming the package and the architecture.
  No silent skip.
- **Test-runtime contract.** System packages are a test-runtime dependency: baked
  into the single local image (used for all checks) but installed in CI **only on
  test jobs**. Repos must not rely on a system package during lint or typecheck.
- **Empty/absent list ⇒ byte-identical to today** (no apt step, no new behaviour).
- **Adoption gating (sequencing):** tooling (accessor + local bake) → vergil-actions
  (CI step) → only then any repo declares `system-packages`.
- Portability (macOS + Linux host), shellcheck-clean bash, no repo-specific logic
  in shared code.

## File Structure

**Repo A — `vergil-project/vergil-tooling`** (PR 1, implementation issue A):

- Modify `src/vergil_tooling/lib/config.py` — `ContainerConfig.system_packages`
  field, validation, `container_system_packages(repo_root)` accessor.
- Modify `src/vergil_tooling/lib/container_cache.py` — `apt_install_command()`
  speller; prepend it to the `setup` string in `_build_cached_image`.
- Create `src/vergil_tooling/bin/vrg_container_system_packages.py` — accessor CLI.
- Modify `pyproject.toml` — register the `vrg-container-system-packages` console script.
- Tests: `tests/vergil_tooling/test_config.py`,
  `tests/vergil_tooling/test_container_cache.py`,
  `tests/vergil_tooling/test_vrg_container_system_packages.py` (new).

**Repo B — `vergil-project/vergil-actions`** (PR 2, implementation issue B):

- Create `actions/shared/setup/system-packages/action.yml` — composite action:
  read via accessor, apt-install with bounded retry + fail-closed.
- Modify `.github/workflows/ci-test.yml` — add the step to the `unit` job, after
  `Install vergil-tooling`, before `Run unit tests`.
- Test: a fixture-repo workflow / action test exercising install + fail-closed +
  retry (per repo's existing action-test conventions).

**Docs** are delivered by the epic's documentation-review bookend (#2724), not
this plan; see "Docs" at the end.

---

### Task 1: Parse `[container].system-packages` (vergil-tooling)

**Files:**

- Modify: `src/vergil_tooling/lib/config.py`
  (`_KNOWN_KEYS["container"]`, `ContainerConfig`, container parse block ~498–509,
  add `container_system_packages` near `container_env_prefixes` ~609–615)
- Test: `tests/vergil_tooling/test_config.py`

**Interfaces:**

- Produces:
  - `ContainerConfig.system_packages: list[str]` (default `[]`)
  - `config.container_system_packages(repo_root: Path) -> list[str]`
    (returns `[]` when there is no `vergil.toml`; propagates `ConfigError`)

- [ ] **Step 1: Write the failing tests** (append to `test_config.py`, mirroring
  the existing `env-prefixes` tests):

```python
def test_read_config_container_system_packages(tmp_path: Path) -> None:
    toml = _VALID_TOML + '[container]\nenv-prefixes = ["MQ_"]\nsystem-packages = ["lilypond"]\n'
    (tmp_path / "vergil.toml").write_text(toml)
    cfg = read_config(tmp_path)
    assert cfg.container.system_packages == ["lilypond"]


def test_read_config_container_no_system_packages_defaults_empty(tmp_path: Path) -> None:
    toml = _VALID_TOML + '[container]\nenv-prefixes = ["MQ_"]\n'
    (tmp_path / "vergil.toml").write_text(toml)
    cfg = read_config(tmp_path)
    assert cfg.container.system_packages == []


def test_read_config_container_no_container_section_system_packages_empty(tmp_path: Path) -> None:
    (tmp_path / "vergil.toml").write_text(_VALID_TOML)
    cfg = read_config(tmp_path)
    assert cfg.container.system_packages == []


def test_read_config_container_system_packages_not_list(tmp_path: Path) -> None:
    toml = _VALID_TOML + '[container]\nenv-prefixes = ["MQ_"]\nsystem-packages = "lilypond"\n'
    (tmp_path / "vergil.toml").write_text(toml)
    with pytest.raises(ConfigError, match=r"\[container\]\.system-packages must be a list"):
        read_config(tmp_path)


def test_read_config_container_system_packages_not_strings(tmp_path: Path) -> None:
    toml = _VALID_TOML + '[container]\nenv-prefixes = ["MQ_"]\nsystem-packages = [1, 2]\n'
    (tmp_path / "vergil.toml").write_text(toml)
    with pytest.raises(ConfigError, match=r"\[container\]\.system-packages must be a list"):
        read_config(tmp_path)


def test_container_system_packages_accessor(tmp_path: Path) -> None:
    toml = _VALID_TOML + '[container]\nenv-prefixes = []\nsystem-packages = ["lilypond", "fluidsynth"]\n'
    (tmp_path / "vergil.toml").write_text(toml)
    assert container_system_packages(tmp_path) == ["lilypond", "fluidsynth"]


def test_container_system_packages_no_file(tmp_path: Path) -> None:
    assert container_system_packages(tmp_path) == []
```

Add `container_system_packages` to the imports at the top of the test module
alongside `container_env_prefixes`.

- [ ] **Step 2: Run the tests to verify they fail**

Run: `uv run pytest tests/vergil_tooling/test_config.py -k system_packages -v`
Expected: FAIL (ImportError on `container_system_packages` / `AttributeError` on
`system_packages`).

- [ ] **Step 3: Implement**

In `config.py`:

- Add `system-packages` to the container key set:

```python
    "container": frozenset({"env-prefixes", "system-packages"}),
```

- Add the field to `ContainerConfig`:

```python
@dataclass
class ContainerConfig:
    env_prefixes: list[str]
    system_packages: list[str]
```

- In the container parse block (where `env_prefixes` is validated), parse and
  validate `system-packages`, then construct `ContainerConfig(...)` with both
  fields in **both** branches (present and absent):

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
        container = ContainerConfig(env_prefixes=env_prefixes, system_packages=system_packages)
    else:
        container = ContainerConfig(env_prefixes=[], system_packages=[])
```

- Add the accessor beside `container_env_prefixes`:

```python
def container_system_packages(repo_root: Path) -> list[str]:
    """Return ``[container].system-packages`` from vergil.toml, or ``[]``.

    The single reader of the key; the local cache build and the CI setup step
    both resolve packages through this. Debian ``apt`` package names.
    """
    try:
        cfg = read_config(repo_root)
    except FileNotFoundError:
        return []
    return cfg.container.system_packages
```

- [ ] **Step 4: Run the tests to verify they pass**

Run: `uv run pytest tests/vergil_tooling/test_config.py -v`
Expected: PASS (all, including pre-existing container tests — the new field is
keyword-constructed everywhere, so existing `ContainerConfig(env_prefixes=[...])`
constructions in tests must be updated to pass `system_packages=[]`; grep and fix).

- [ ] **Step 5: Commit**

```bash
vrg-git add src/vergil_tooling/lib/config.py tests/vergil_tooling/test_config.py
vrg-commit --type feat --scope config \
  --message "parse [container].system-packages" \
  --body "Add ContainerConfig.system_packages + container_system_packages accessor; the single reader of the key for both the local cache build and CI. Epic vergil-project/.github#272."
```

---

### Task 2: Install-command speller + bake into the cached image (vergil-tooling)

**Files:**

- Modify: `src/vergil_tooling/lib/container_cache.py`
  (new `apt_install_command`; extend `_build_cached_image` `setup` construction ~234–244)
- Test: `tests/vergil_tooling/test_container_cache.py`

**Interfaces:**

- Consumes: `config.container_system_packages` (Task 1); `container_platform`
  (already imported in `container.py`).
- Produces:
  - `container_cache.apt_install_command(packages: list[str], platform_label: str) -> str`
    — returns the shell snippet that installs the packages fail-closed, or `""`
    for an empty list. **This is the single speller** reused by the CI accessor
    (Task 3).

- [ ] **Step 1: Write the failing tests** (append to `test_container_cache.py`):

```python
from vergil_tooling.lib.container_cache import apt_install_command, compute_cache_hash


def test_apt_install_command_empty_is_blank() -> None:
    assert apt_install_command([], "linux/arm64") == ""


def test_apt_install_command_updates_then_installs_each_package() -> None:
    cmd = apt_install_command(["lilypond", "fluidsynth"], "linux/arm64")
    assert "apt-get update" in cmd
    assert "--no-install-recommends" in cmd
    # per-package install so the failing package is named
    assert "lilypond" in cmd and "fluidsynth" in cmd


def test_apt_install_command_fail_closed_names_package_and_arch() -> None:
    cmd = apt_install_command(["boguspkg"], "linux/arm64")
    assert "boguspkg" in cmd
    assert "linux/arm64" in cmd
    assert "not installable" in cmd
    assert "exit 1" in cmd


def test_cache_hash_changes_when_system_packages_change(tmp_path: Path) -> None:
    # The §3.6 invariant: vergil.toml is cache-sensitive, so editing the package
    # list changes the hash and forces a rebuild.
    base = _VALID_VERGIL_TOML  # module-level minimal valid config used by other tests
    a = tmp_path / "a.toml"
    b = tmp_path / "b.toml"
    a.write_text(base + '[container]\nenv-prefixes = []\nsystem-packages = ["lilypond"]\n')
    b.write_text(base + '[container]\nenv-prefixes = []\nsystem-packages = ["lilypond", "fluidsynth"]\n')
    assert compute_cache_hash([a]) != compute_cache_hash([b])
```

If `test_container_cache.py` has no minimal-valid-toml constant, add one (copy the
shape from `test_config.py`'s `_VALID_TOML`).

- [ ] **Step 2: Run the tests to verify they fail**

Run: `uv run pytest tests/vergil_tooling/test_container_cache.py -k "apt_install or system_packages" -v`
Expected: FAIL (`apt_install_command` not defined).

- [ ] **Step 3: Implement the speller** in `container_cache.py`:

```python
import shlex  # add to imports


def apt_install_command(packages: list[str], platform_label: str) -> str:
    """Return a shell snippet installing *packages* fail-closed, or ``""``.

    Installs one package at a time so a missing candidate names the offending
    package; on failure it prints a message with the package and *platform_label*
    and exits non-zero (no silent skip). Debian ``apt`` names, from the base
    image's existing sources, ``--no-install-recommends``.
    """
    if not packages:
        return ""
    parts = ["apt-get update"]
    for pkg in packages:
        q = shlex.quote(pkg)
        parts.append(
            f"{{ apt-get install -y --no-install-recommends {q} "
            f"|| {{ echo \"system package {q} is not installable on "
            f"{platform_label} (apt: no installation candidate)\" >&2; exit 1; }} }}"
        )
    return " && ".join(parts)
```

- [ ] **Step 4: Prepend the snippet in `_build_cached_image`.** After `warmup` is
  computed and before `setup` is assembled, resolve packages and prepend:

```python
    warmup = _warmup_command(lang)
    from vergil_tooling.lib.config import container_system_packages
    from vergil_tooling.lib.container import container_platform

    apt = apt_install_command(container_system_packages(repo_root), container_platform())

    if self_repo:
        setup = warmup or "true"
    else:
        tag = vrg_install_tag(repo_root)
        uv_install = f"uv tool install --quiet 'vergil-tooling @ git+{_VRG_GIT_URL}@{tag}'"
        setup = f"{uv_install} && {warmup}" if warmup else uv_install

    if apt:
        setup = f"{apt} && {setup}"
```

Extend the provisioning banner to print `Packages: <names>` when `apt` is
non-empty (mirroring the existing `Warmup:` line), so a system-package install is
visible in the build log.

- [ ] **Step 5: Run the tests to verify they pass**

Run: `uv run pytest tests/vergil_tooling/test_container_cache.py -v`
Expected: PASS.

- [ ] **Step 6: Commit**

```bash
vrg-git add src/vergil_tooling/lib/container_cache.py tests/vergil_tooling/test_container_cache.py
vrg-commit --type feat --scope container \
  --message "bake declared system-packages into the cached dev image" \
  --body "apt_install_command spells a fail-closed per-package apt install; _build_cached_image prepends it to setup when [container].system-packages is declared. Cache key already covers it via vergil.toml. Epic vergil-project/.github#272."
```

---

### Task 3: `vrg-container-system-packages` accessor (vergil-tooling)

**Files:**

- Create: `src/vergil_tooling/bin/vrg_container_system_packages.py`
- Modify: `pyproject.toml` (console_scripts, next to the other `vrg-container-*`)
- Test: `tests/vergil_tooling/test_vrg_container_system_packages.py`

**Interfaces:**

- Consumes: `config.container_system_packages` (Task 1); `container_cache.apt_install_command`
  and `container.container_platform` (Task 2).
- Produces: CLI `vrg-container-system-packages`:
  - `--install-script`: prints the exact shell snippet from `apt_install_command`
    (the single speller CI executes), or nothing when the list is empty. **This is
    the mode CI consumes** (spec §3.3).
  - default: prints the package list, one per line (empty output when none). This
    is a **deliberate human-facing inspection affordance** ("what would this repo
    install?"), not a second automated consumer path — kept intentionally (spec
    §3.1).

- [ ] **Step 1: Write the failing tests** (new file):

```python
from pathlib import Path

from vergil_tooling.bin import vrg_container_system_packages as cli


def _write(tmp_path: Path, packages: str) -> Path:
    (tmp_path / "vergil.toml").write_text(_VALID_TOML + f"[container]\nenv-prefixes = []\nsystem-packages = {packages}\n")
    return tmp_path


def test_list_mode_prints_one_per_line(tmp_path, capsys, monkeypatch):
    _write(tmp_path, '["lilypond", "fluidsynth"]')
    monkeypatch.chdir(tmp_path)
    assert cli.main([]) == 0
    assert capsys.readouterr().out.split() == ["lilypond", "fluidsynth"]


def test_list_mode_empty_prints_nothing(tmp_path, capsys, monkeypatch):
    _write(tmp_path, "[]")
    monkeypatch.chdir(tmp_path)
    assert cli.main([]) == 0
    assert capsys.readouterr().out.strip() == ""


def test_install_script_mode_emits_apt(tmp_path, capsys, monkeypatch):
    _write(tmp_path, '["lilypond"]')
    monkeypatch.chdir(tmp_path)
    assert cli.main(["--install-script"]) == 0
    out = capsys.readouterr().out
    assert "apt-get install -y --no-install-recommends" in out and "lilypond" in out
```

(Reuse a `_VALID_TOML` constant as in the other bin tests, or import a shared
fixture.)

- [ ] **Step 2: Run to verify it fails**

Run: `uv run pytest tests/vergil_tooling/test_vrg_container_system_packages.py -v`
Expected: FAIL (module does not exist).

- [ ] **Step 3: Implement the CLI:**

```python
"""Print a repo's declared [container].system-packages, for CI consumption.

Default: the package list, one per line. --install-script: the exact apt
install snippet (the single speller shared with the local cache build).
"""

from __future__ import annotations

import argparse
from pathlib import Path

from vergil_tooling.lib.config import container_system_packages
from vergil_tooling.lib.container import container_platform
from vergil_tooling.lib.container_cache import apt_install_command


def main(argv: list[str] | None = None) -> int:
    parser = argparse.ArgumentParser(description="Print declared [container].system-packages.")
    parser.add_argument(
        "--install-script",
        action="store_true",
        help="print the apt install snippet instead of the package list",
    )
    parser.add_argument("--repo-root", default=".", help="repo root (default: CWD)")
    args = parser.parse_args(argv)

    root = Path(args.repo_root)
    packages = container_system_packages(root)
    if args.install_script:
        script = apt_install_command(packages, container_platform())
        if script:
            print(script)
    else:
        for pkg in packages:
            print(pkg)
    return 0
```

- [ ] **Step 4: Register the console script** in `pyproject.toml`, beside the
  other `vrg-container-*` entries:

```toml
vrg-container-system-packages = "vergil_tooling.bin.vrg_container_system_packages:main"
```

- [ ] **Step 5: Run tests + re-sync so the entry point resolves**

Run: `uv sync --group dev && uv run pytest tests/vergil_tooling/test_vrg_container_system_packages.py -v`
Expected: PASS; `uv run vrg-container-system-packages --help` works.

- [ ] **Step 6: Commit**

```bash
vrg-git add src/vergil_tooling/bin/vrg_container_system_packages.py pyproject.toml tests/vergil_tooling/test_vrg_container_system_packages.py
vrg-commit --type feat --scope container \
  --message "add vrg-container-system-packages accessor for CI" \
  --body "CLI printing the declared package list, or --install-script emitting the shared apt speller for the CI setup step. Epic vergil-project/.github#272."
```

- [ ] **Step 7: Validate the whole vergil-tooling change**

Run: `vrg-container-run -- vrg-validate`
Expected: PASS. This is the end of implementation issue A (PR 1).

---

### Task 4: CI setup step in vergil-actions (test job only, retry, fail-closed)

**Files:**

- Create: `actions/shared/setup/system-packages/action.yml`
- Modify: `.github/workflows/ci-test.yml` (the `unit` job)
- Test: per the repo's action-test convention (fixture repo / workflow test)

**Interfaces:**

- Consumes: `vrg-container-system-packages` (Task 3), on `PATH` after the existing
  `./actions/shared/setup/vergil` step.

- [ ] **Step 1: Write the composite action** `actions/shared/setup/system-packages/action.yml`:

```yaml
name: Install repo system packages
description: >-
  Install the Debian packages a repo declares in [container].system-packages,
  read through vrg-container-system-packages. Test-runtime dependency: this runs
  only on jobs that execute the repo's tests. Bounded retry around apt (mirror
  flake); fail-closed with a package+arch message on a missing candidate.

runs:
  using: composite
  steps:
    - name: Install declared system packages
      shell: bash
      run: |
        set -euo pipefail
        script="$(vrg-container-system-packages --install-script)"
        if [ -z "${script}" ]; then
          echo "No [container].system-packages declared; skipping."
          exit 0
        fi
        attempts=3
        for i in $(seq 1 "${attempts}"); do
          if bash -c "${script}"; then
            exit 0
          fi
          echo "system-package install attempt ${i}/${attempts} failed" >&2
          [ "${i}" -lt "${attempts}" ] && sleep 5
        done
        echo "::error::system-package install failed after ${attempts} attempts" >&2
        exit 1
```

Note: the fail-closed message (package + arch) is emitted by the snippet itself;
the retry wrapper adds only the attempt bookkeeping. A genuinely missing package
exhausts the retries and exits non-zero with the snippet's message visible — the
correct fail-closed outcome.

- [ ] **Step 2: Wire it into the `unit` job** of `.github/workflows/ci-test.yml`,
  between `Install vergil-tooling` and `Run unit tests`:

```yaml
      - name: Install vergil-tooling
        uses: ./actions/shared/setup/vergil

      - name: Install repo system packages
        uses: ./actions/shared/setup/system-packages

      - name: Install dependencies
        if: inputs.language == 'python'
        run: uv sync --group dev --frozen

      - name: Run unit tests
        run: vrg-validate --check test
```

Do **not** add it to `ci-quality.yml`'s lint/typecheck jobs — the test-runtime
contract keeps it test-only.

- [ ] **Step 3: Add an action test** following the repo's existing convention for
  `actions/shared/*` (a fixture repo declaring `system-packages = ["<a real
  Debian-main pkg>"]`, plus a negative fixture declaring a bogus name asserting a
  non-zero exit with the "not installable" message, and a retry-path test that
  simulates a transient failure succeeding on a later attempt — e.g. a stub
  `apt-get` on `PATH` that fails once). Match how the repo tests other composite
  actions; if it uses `bats`/shell harnesses, add cases there.

- [ ] **Step 4: Validate**

Run the repo's validation (`vrg-container-run -- vrg-validate`) plus `actionlint`
via the standard pipeline.
Expected: PASS. This is the end of implementation issue B (PR 2).

- [ ] **Step 5: Commit**

```bash
vrg-git add actions/shared/setup/system-packages/action.yml .github/workflows/ci-test.yml <test files>
vrg-commit --type feat --scope ci \
  --message "install declared system-packages on CI test jobs" \
  --body "Composite action reads [container].system-packages via vrg-container-system-packages and apt-installs on the unit job with bounded retry and fail-closed behaviour. Test-runtime only. Epic vergil-project/.github#272."
```

---

## Docs (bookend #2724, not this plan)

The spec's docs acceptance — documenting the `[container].system-packages` key,
its Debian coupling, the trust model, and the fail-closed behaviour — is the
epic's **documentation-review** bookend (`vergil-tooling#2724`), a multi-repo
sweep. It updates the `[container]` config reference and the container-model guide
in `vergil-tooling/docs/site`, and spawns a same-repo doc task in `vergil-actions`
if the CI step needs operator-facing docs. Not implemented here.

## Epic implementation issues (filed in step 9)

- **Issue A — `vergil-project/vergil-tooling`:** Tasks 1–3 (config, bake, accessor).
  One PR.
- **Issue B — `vergil-project/vergil-actions`:** Task 4 (CI setup step). One PR.
  **Blocked-by** Issue A (needs the released accessor; adoption gating, spec §5).
- The cold-rebuild **validation** task (`vergil-tooling#2725`) is blocked-by both A
  and B and run with `issue-validate` after they merge/release.

## Self-Review

- **Spec coverage.** §3.1 config → Task 1. §3.2 local bake + fail-closed → Task 2.
  §3.3 CI step (test-only, retry, accessor, fail-closed) → Task 4 (accessor from
  Task 3). §3.4 naming → Global Constraints + Task 1 (docs line deferred to #2724).
  §3.5 cross-arch fail-closed → Task 2 `apt_install_command` + Task 4 propagation.
  §3.6 cache key → Task 2 hash-invariant test. §3.7 trust → Global Constraints
  (no code surface beyond names). §5 sequencing → "Epic implementation issues"
  (A → B blocked-by, adoption gating). §6 acceptance → covered across Tasks 1–4 +
  #2725 validation. §7 testing → each task's tests + #2725.
- **Placeholder scan.** No TBD/TODO; every code step carries real code. The only
  deferred specifics are the `vergil-actions` action-test harness (Task 4 Step 3),
  bounded by "match the repo's existing convention" because that convention is
  repo-local — an implementer reads the neighbouring `actions/shared/*` tests.
- **Type consistency.** `container_system_packages(repo_root) -> list[str]`,
  `apt_install_command(packages, platform_label) -> str`, and
  `vrg-container-system-packages [--install-script]` are used with the same
  signatures in every task that consumes them.
