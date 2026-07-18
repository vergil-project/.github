# macOS host dependency management (`vrg-host`) — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Ship `vrg-host`, a host-side CLI that enumerates the Virgil-critical macOS host dependencies from a tooling-owned manifest, reports installed-vs-latest gaps and per-dependency health, upgrades the ones that can be upgraded automatically, and pins components as tracked technical debt.

**Architecture:** A declarative TOML manifest shipped in `vergil_tooling.data` describes each dependency (install method, upgrade capability, latest-signal kind, auxiliary checks, optional pin). A `host_deps` lib module loads/validates it into dataclasses; a `host_methods` registry maps each `method` to a handler that knows how to read installed/latest versions and upgrade; a `host_status` module classifies each dependency on two axes (version state × auxiliary checks) and rolls up severity; the `vrg_host` bin exposes `status`/`upgrade`/`pin`/`unpin`. Pin semantics inherit epic #155 verbatim.

**Tech Stack:** Python 3.12, stdlib only for the runtime path — `tomllib` for the manifest (no new dependency), `importlib.resources` for packaged-data loading, `argparse` for the CLI, `subprocess` for tool probes. Tests use `pytest` with stubbed `subprocess` (no real host mutation).

## Global Constraints

- **Python 3.12; runtime path is stdlib-only.** Manifest parsed with `tomllib`; no PyYAML or other new runtime dependency. (Verbatim decision resolving spec §7.1 "YAML/TOML — settled at plan time".)
- **No silent failures.** A probe that cannot answer is surfaced explicitly (`latest-unknown` WARN / `latest-unmanaged` INFO / unsatisfied check ERROR) — never treated as OK. (Spec §8; tooling-wide rule.)
- **`vrg-host` runs unprivileged and never silently invokes `sudo`.** (Spec §10.)
- **The manifest is tooling-owned and never repo-overridable.** `custom` shell strings are trusted, tooling-authored content, loaded only from `vergil_tooling.data`. (Spec §10 security invariant.)
- **Pin doctrine is epic #155's, verbatim** — schema fields `constraint`, `inducing_release`, `deterministic`, `reason`, `state` (`active`/`under-evaluation`/`freed`), `tracking_issue`; a pin with no `reason` is refused. (Spec §5, §9.)
- **macOS-only.** No cross-platform host logic (that is #909). Tools installed as `vrg-*` console scripts under `[project.scripts]` in `pyproject.toml`.
- **Exit code = worst severity across all dependencies and axes:** `0` OK, `1` warnings, `2` errors. (Spec §8.)

---

## File Structure

- `src/vergil_tooling/data/host_dependencies.toml` — the declarative manifest (single source of truth for what the host requires).
- `src/vergil_tooling/lib/host_deps.py` — dataclasses (`HostDependency`, `AuxCheck`, `HostPin`), TOML loader, validation.
- `src/vergil_tooling/lib/host_methods.py` — method-handler registry (`brew-formula`, `brew-cask`, `uv-tool`, `custom`) implementing installed/latest/upgrade.
- `src/vergil_tooling/lib/host_status.py` — version-state + aux-check classification, severity roll-up, human-readable rendering.
- `src/vergil_tooling/lib/host_pins.py` — pin read/write against the manifest, re-evaluation-due detection (#155 trigger).
- `src/vergil_tooling/bin/vrg_host.py` — the `vrg-host` CLI (`status`/`upgrade`/`pin`/`unpin`), `main()`/`parse_args()`.
- `tests/vergil_tooling/test_host_deps.py`, `test_host_methods.py`, `test_host_status.py`, `test_host_pins.py`, `test_vrg_host.py` — unit tests, `subprocess` stubbed.
- `pyproject.toml` — add `vrg-host = "vergil_tooling.bin.vrg_host:main"` under `[project.scripts]`.

Each task below is scoped to become a single GitHub implementation task filed under epic vergil-project/.github#167.

---

## Task 1: Manifest schema + TOML loader

**Files:**
- Create: `src/vergil_tooling/lib/host_deps.py`
- Create: `src/vergil_tooling/data/host_dependencies.toml` (minimal seed; full curation is Task 7)
- Test: `tests/vergil_tooling/test_host_deps.py`

**Interfaces:**
- Produces:
  - `@dataclass(frozen=True) AuxCheck(name: str, probe: list[str], remediation: list[str] | None)`
  - `@dataclass(frozen=True) HostPin(constraint: str, inducing_release: str, deterministic: bool, reason: str, state: str, tracking_issue: str | None, pinned_date: str)` — `pinned_date` (ISO `YYYY-MM-DD`) is the one deliberate host extension to #155's schema, so `status` can render pin age (spec §5, §9).
  - `@dataclass(frozen=True) HostDependency(name: str, why: str, method: str, upgrade: str, latest: str, probes: dict[str, list[str]], checks: tuple[AuxCheck, ...], pin: HostPin | None)` where `upgrade ∈ {"auto","manual"}`, `latest ∈ {"probe","unmanaged"}`, `method ∈ {"brew-formula","brew-cask","uv-tool","custom"}`.
  - `load_manifest() -> tuple[HostDependency, ...]`
  - `ManifestError(Exception)`

- [ ] **Step 1: Write the failing test**

```python
# tests/vergil_tooling/test_host_deps.py
from vergil_tooling.lib import host_deps

def test_load_manifest_parses_seed_entries():
    deps = host_deps.load_manifest()
    by_name = {d.name: d for d in deps}
    assert "uv" in by_name
    assert by_name["uv"].method == "uv-tool"
    assert by_name["uv"].upgrade == "auto"

def test_unknown_method_raises(tmp_path, monkeypatch):
    bad = '[[dependency]]\nname="x"\nwhy="w"\nmethod="wat"\n'
    monkeypatch.setattr(host_deps, "_read_manifest_text", lambda: bad)
    import pytest
    with pytest.raises(host_deps.ManifestError, match="unknown method"):
        host_deps.load_manifest()

def test_manifest_read_only_from_package_not_cwd(tmp_path, monkeypatch):
    """Security invariant (spec §10): the manifest is never repo/CWD-overridable.

    Plant a hostile manifest in the CWD; loading must ignore it entirely and
    still read the packaged data. This test is the regression tripwire — if a
    future change adds a repo-override path, it breaks and forces the security
    decision back into the open.
    """
    (tmp_path / "host_dependencies.toml").write_text(
        '[[dependency]]\nname="evil"\nwhy="x"\nmethod="custom"\n'
    )
    monkeypatch.chdir(tmp_path)
    names = {d.name for d in host_deps.load_manifest()}
    assert "evil" not in names            # CWD file ignored
    assert "uv" in names                  # packaged manifest still used
    # And the resolved source is inside the package, never a loose path:
    from importlib import resources
    ref = resources.files("vergil_tooling.data").joinpath("host_dependencies.toml")
    assert "vergil_tooling/data" in str(ref)
```

- [ ] **Step 2: Run test to verify it fails**

Run: `uv run pytest tests/vergil_tooling/test_host_deps.py -v`
Expected: FAIL (`AttributeError: module 'vergil_tooling.lib.host_deps' has no attribute 'load_manifest'`).

- [ ] **Step 3: Write minimal implementation**

```python
# src/vergil_tooling/lib/host_deps.py
"""Load and validate the tooling-owned macOS host-dependency manifest."""
from __future__ import annotations

import tomllib
from dataclasses import dataclass
from importlib import resources

_METHODS = {"brew-formula", "brew-cask", "uv-tool", "custom"}
_UPGRADE = {"auto", "manual"}
_LATEST = {"probe", "unmanaged"}
_PIN_STATES = {"active", "under-evaluation", "freed"}


class ManifestError(Exception):
    """The host-dependency manifest is malformed."""


@dataclass(frozen=True)
class AuxCheck:
    name: str
    probe: list[str]
    remediation: list[str] | None


@dataclass(frozen=True)
class HostPin:
    constraint: str
    inducing_release: str
    deterministic: bool
    reason: str
    state: str
    tracking_issue: str | None
    pinned_date: str  # ISO YYYY-MM-DD; host extension to #155 for age rendering


@dataclass(frozen=True)
class HostDependency:
    name: str
    why: str
    method: str
    upgrade: str
    latest: str
    probes: dict[str, list[str]]
    checks: tuple[AuxCheck, ...]
    pin: HostPin | None


def _read_manifest_text() -> str:
    ref = resources.files("vergil_tooling.data").joinpath("host_dependencies.toml")
    return ref.read_text(encoding="utf-8")


def _parse_pin(raw: dict) -> HostPin:
    if not raw.get("reason"):
        raise ManifestError(f"pin without a reason: {raw!r}")
    state = raw.get("state", "active")
    if state not in _PIN_STATES:
        raise ManifestError(f"unknown pin state: {state}")
    return HostPin(
        constraint=raw["constraint"],
        inducing_release=raw["inducing_release"],
        deterministic=bool(raw.get("deterministic", False)),
        reason=raw["reason"],
        state=state,
        tracking_issue=raw.get("tracking_issue"),
        pinned_date=raw["pinned_date"],
    )


def _parse_dep(raw: dict) -> HostDependency:
    method = raw.get("method")
    if method not in _METHODS:
        raise ManifestError(f"unknown method {method!r} for {raw.get('name')!r}")
    upgrade = raw.get("upgrade", "auto")
    if upgrade not in _UPGRADE:
        raise ManifestError(f"unknown upgrade capability {upgrade!r}")
    latest = raw.get("latest", "probe")
    if latest not in _LATEST:
        raise ManifestError(f"unknown latest kind {latest!r}")
    checks = tuple(
        AuxCheck(c["name"], c["probe"], c.get("remediation"))
        for c in raw.get("check", [])
    )
    pin = _parse_pin(raw["pin"]) if "pin" in raw else None
    return HostDependency(
        name=raw["name"],
        why=raw.get("why", ""),
        method=method,
        upgrade=upgrade,
        latest=latest,
        probes=raw.get("probes", {}),
        checks=checks,
        pin=pin,
    )


def load_manifest() -> tuple[HostDependency, ...]:
    data = tomllib.loads(_read_manifest_text())
    return tuple(_parse_dep(d) for d in data.get("dependency", []))
```

```toml
# src/vergil_tooling/data/host_dependencies.toml
# Tooling-owned. NEVER repo-overridable (see epic #167 spec §10).
[[dependency]]
name = "uv"
why = "Python build/runtime tool manager; installs the vrg-* host tools."
method = "uv-tool"
upgrade = "auto"
latest = "probe"

[[dependency]]
name = "gh"
why = "GitHub CLI; underpins vrg-gh and all PR/issue workflows."
method = "brew-formula"
upgrade = "auto"
latest = "probe"
```

- [ ] **Step 4: Run test to verify it passes**

Run: `uv run pytest tests/vergil_tooling/test_host_deps.py -v`
Expected: PASS (both tests).

- [ ] **Step 5: Commit**

```bash
vrg-git add src/vergil_tooling/lib/host_deps.py src/vergil_tooling/data/host_dependencies.toml tests/vergil_tooling/test_host_deps.py
vrg-commit --type feat --scope host --message "add host-dependency manifest schema and TOML loader"
```

---

## Task 2: Method-handler registry

**Files:**
- Create: `src/vergil_tooling/lib/host_methods.py`
- Test: `tests/vergil_tooling/test_host_methods.py`

**Interfaces:**
- Consumes: `HostDependency` (Task 1).
- Produces:
  - Sentinel `UNMANAGED = object()` for "no latest feed".
  - `installed_version(dep: HostDependency) -> str | None` (None ⇒ missing).
  - `latest_version(dep: HostDependency) -> str | None | object` (`UNMANAGED` when `dep.latest == "unmanaged"`; `None` ⇒ probe failed).
  - `upgrade(dep: HostDependency) -> None` (raises `UpgradeUnsupported` for `manual`).
  - `UpgradeUnsupported(Exception)`.
- All shell-out via a single `_run(cmd: list[str]) -> str | None` helper (returns None on non-zero/OSError) so tests stub one point.

- [ ] **Step 1: Write the failing test**

```python
# tests/vergil_tooling/test_host_methods.py
from vergil_tooling.lib import host_methods as hm
from vergil_tooling.lib.host_deps import HostDependency

def _dep(**kw):
    base = dict(name="gh", why="", method="brew-formula", upgrade="auto",
               latest="probe", probes={}, checks=(), pin=None)
    base.update(kw)
    return HostDependency(**base)

def test_latest_unmanaged_returns_sentinel():
    dep = _dep(latest="unmanaged")
    assert hm.latest_version(dep) is hm.UNMANAGED

def test_installed_version_none_when_probe_fails(monkeypatch):
    monkeypatch.setattr(hm, "_run", lambda cmd: None)
    assert hm.installed_version(_dep()) is None

def test_manual_upgrade_raises():
    import pytest
    with pytest.raises(hm.UpgradeUnsupported):
        hm.upgrade(_dep(upgrade="manual"))
```

- [ ] **Step 2: Run test to verify it fails**

Run: `uv run pytest tests/vergil_tooling/test_host_methods.py -v`
Expected: FAIL (module/attributes missing).

- [ ] **Step 3: Write minimal implementation**

```python
# src/vergil_tooling/lib/host_methods.py
"""Install-method handlers: installed/latest/upgrade per HostDependency."""
from __future__ import annotations

import subprocess

from vergil_tooling.lib.host_deps import HostDependency

UNMANAGED = object()


class UpgradeUnsupported(Exception):
    """upgrade() called on a manual (observe-only) dependency."""


def _run(cmd: list[str]) -> str | None:
    try:
        out = subprocess.run(cmd, capture_output=True, text=True, check=True)  # noqa: S603
    except (subprocess.CalledProcessError, OSError):
        return None
    return out.stdout.strip()


def _custom(dep: HostDependency, key: str) -> list[str] | None:
    cmd = dep.probes.get(key)
    return cmd or None


def installed_version(dep: HostDependency) -> str | None:
    if dep.method in ("brew-formula", "brew-cask"):
        return _run(["brew", "list", "--versions", dep.name]) or None
    if dep.method == "uv-tool":
        return _run(["uv", "tool", "list"])  # parsed for dep.name by caller-side helper
    if dep.method == "custom":
        cmd = _custom(dep, "installed_version")
        return _run(cmd) if cmd else None
    return None


def latest_version(dep: HostDependency):
    if dep.latest == "unmanaged":
        return UNMANAGED
    if dep.method in ("brew-formula", "brew-cask"):
        return _run(["brew", "info", "--json=v2", dep.name])
    if dep.method == "uv-tool":
        return _run(["uv", "tool", "list", "--outdated"])
    if dep.method == "custom":
        cmd = _custom(dep, "latest_version")
        return _run(cmd) if cmd else None
    return None


def upgrade(dep: HostDependency) -> None:
    if dep.upgrade == "manual":
        raise UpgradeUnsupported(dep.name)
    if dep.method in ("brew-formula", "brew-cask"):
        _run(["brew", "upgrade", dep.name])
    elif dep.method == "uv-tool":
        _run(["uv", "tool", "upgrade", dep.name])
    elif dep.method == "custom":
        cmd = _custom(dep, "upgrade")
        if cmd:
            _run(cmd)
```

> Note: brew/uv output *parsing* (extracting a bare version from `brew info --json=v2` / `uv tool list`) is refined during Task 7 enumeration against real fixtures; Task 2 establishes the dispatch and the stub seam. Keep parsing helpers pure and separately tested.

- [ ] **Step 4: Run test to verify it passes**

Run: `uv run pytest tests/vergil_tooling/test_host_methods.py -v`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
vrg-git add src/vergil_tooling/lib/host_methods.py tests/vergil_tooling/test_host_methods.py
vrg-commit --type feat --scope host --message "add install-method handler registry"
```

---

## Task 3: `vrg-host status` — classification, reporting, entry point

**Files:**
- Create: `src/vergil_tooling/lib/host_status.py`
- Create: `src/vergil_tooling/bin/vrg_host.py`
- Modify: `pyproject.toml` (add `vrg-host` under `[project.scripts]`, alphabetical position near `vrg-hook-guard`)
- Test: `tests/vergil_tooling/test_host_status.py`, `tests/vergil_tooling/test_vrg_host.py`

**Interfaces:**
- Consumes: `HostDependency` (T1), `installed_version`/`latest_version`/`UNMANAGED` (T2).
- Produces:
  - `VERSION_STATES` = `{"missing","current","behind","pinned","latest-unmanaged","latest-unknown"}`.
  - `classify(dep, installed, latest, check_results) -> tuple[str, int]` returning `(state, severity)` where severity `0/1/2` = OK/WARN/ERROR; worst of version-state and aux-check severities.
  - `Report` dataclass and `render(reports) -> str`.
  - `main(argv=None) -> int` in `vrg_host.py` returning worst severity as exit code.

- [ ] **Step 1: Write the failing test**

```python
# tests/vergil_tooling/test_host_status.py
from vergil_tooling.lib import host_status as hs
from vergil_tooling.lib import host_methods as hm
from vergil_tooling.lib.host_deps import HostDependency, AuxCheck

def _dep(**kw):
    base = dict(name="gh", why="", method="brew-formula", upgrade="auto",
               latest="probe", probes={}, checks=(), pin=None)
    base.update(kw); return HostDependency(**base)

def test_missing_is_error():
    state, sev = hs.classify(_dep(), installed=None, latest="2.0", check_results=[])
    assert (state, sev) == ("missing", 2)

def test_behind_is_warn():
    _, sev = hs.classify(_dep(), installed="1.0", latest="2.0", check_results=[])
    assert sev == 1

def test_latest_unmanaged_is_info_not_warn():
    state, sev = hs.classify(_dep(latest="unmanaged"), installed="1.0",
                             latest=hm.UNMANAGED, check_results=[])
    assert state == "latest-unmanaged" and sev == 0

def test_latest_unknown_is_warn():
    state, sev = hs.classify(_dep(), installed="1.0", latest=None, check_results=[])
    assert state == "latest-unknown" and sev == 1

def test_unsatisfied_check_forces_error():
    dep = _dep(checks=(AuxCheck("numpy", ["true"], ["fix"]),))
    _, sev = hs.classify(dep, installed="1.0", latest="1.0",
                         check_results=[("numpy", False)])
    assert sev == 2
```

- [ ] **Step 2: Run test to verify it fails**

Run: `uv run pytest tests/vergil_tooling/test_host_status.py -v`
Expected: FAIL (module missing).

- [ ] **Step 3: Write minimal implementation**

```python
# src/vergil_tooling/lib/host_status.py
"""Classify each host dependency (version state × aux checks) and render."""
from __future__ import annotations

from dataclasses import dataclass

from vergil_tooling.lib import host_methods as hm
from vergil_tooling.lib.host_deps import HostDependency

OK, WARN, ERROR = 0, 1, 2


def classify(dep: HostDependency, installed, latest, check_results) -> tuple[str, int]:
    check_sev = ERROR if any(not ok for _, ok in check_results) else OK
    if installed is None:
        return "missing", max(ERROR, check_sev)
    if dep.pin is not None and dep.pin.state == "active":
        return "pinned", max(WARN if _behind(installed, dep.pin.constraint) else OK, check_sev)
    if latest is hm.UNMANAGED:
        return "latest-unmanaged", max(OK, check_sev)
    if latest is None:
        return "latest-unknown", max(WARN, check_sev)
    if installed != latest:
        return "behind", max(WARN, check_sev)
    return "current", max(OK, check_sev)


def _behind(installed: str, latest: str) -> bool:
    # Heterogeneous version schemes: treat "differs" as behind unless a
    # tool-specific comparator is registered (refined in Task 7).
    return installed != latest


@dataclass(frozen=True)
class Report:
    dep: HostDependency
    installed: object
    latest: object
    state: str
    severity: int
    check_results: list


def render(reports: list[Report]) -> str:
    rows = []
    for r in reports:
        tag = {0: "OK", 1: "WARN", 2: "ERR"}[r.severity]
        rows.append(f"[{tag:4}] {r.dep.name:16} {str(r.installed):12} -> {r.state}")
    return "\n".join(rows)
```

```python
# src/vergil_tooling/bin/vrg_host.py
"""vrg-host: observe and manage Virgil-critical macOS host dependencies."""
from __future__ import annotations

import argparse
import sys

from vergil_tooling.lib import host_deps, host_methods, host_status
from vergil_tooling.lib.host_status import Report


def _run_check(check) -> tuple[str, bool]:
    ok = host_methods._run(check.probe) is not None
    return check.name, ok


def _status(_args) -> int:
    reports: list[Report] = []
    worst = 0
    for dep in host_deps.load_manifest():
        installed = host_methods.installed_version(dep)
        latest = host_methods.latest_version(dep)
        checks = [_run_check(c) for c in dep.checks]
        state, sev = host_status.classify(dep, installed, latest, checks)
        reports.append(Report(dep, installed, latest, state, sev, checks))
        worst = max(worst, sev)
    print(host_status.render(reports))
    return worst


def parse_args(argv: list[str] | None = None) -> argparse.Namespace:
    parser = argparse.ArgumentParser(prog="vrg-host", description=__doc__)
    sub = parser.add_subparsers(dest="cmd")
    sub.add_parser("status", help="report installed-vs-latest gaps and health")
    args = parser.parse_args(argv)
    if args.cmd is None:
        args.cmd = "status"
    return args


def main(argv: list[str] | None = None) -> int:
    args = parse_args(argv)
    if args.cmd == "status":
        return _status(args)
    return 0


if __name__ == "__main__":
    sys.exit(main())
```

- [ ] **Step 4: Add the entry point and run the CLI test**

Add to `pyproject.toml` `[project.scripts]`: `vrg-host = "vergil_tooling.bin.vrg_host:main"`.

```python
# tests/vergil_tooling/test_vrg_host.py
from vergil_tooling.bin import vrg_host

def test_status_exit_code_reflects_worst(monkeypatch, capsys):
    from vergil_tooling.lib.host_deps import HostDependency
    dep = HostDependency("gh","", "brew-formula","auto","probe",{},(),None)
    monkeypatch.setattr(vrg_host.host_deps, "load_manifest", lambda: (dep,))
    monkeypatch.setattr(vrg_host.host_methods, "installed_version", lambda d: None)
    monkeypatch.setattr(vrg_host.host_methods, "latest_version", lambda d: "2.0")
    assert vrg_host.main(["status"]) == 2  # missing -> ERROR
```

Run: `uv run pytest tests/vergil_tooling/test_host_status.py tests/vergil_tooling/test_vrg_host.py -v`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
vrg-git add src/vergil_tooling/lib/host_status.py src/vergil_tooling/bin/vrg_host.py pyproject.toml tests/vergil_tooling/test_host_status.py tests/vergil_tooling/test_vrg_host.py
vrg-commit --type feat --scope host --message "add vrg-host status with two-axis classification"
```

---

## Task 4: Auxiliary requirement checks + gcloud numpy (#166 absorption)

**Files:**
- Modify: `src/vergil_tooling/bin/vrg_host.py` (a `--fix-checks` path / remediation wiring)
- Modify: `src/vergil_tooling/data/host_dependencies.toml` (gcloud `custom` entry with the numpy check)
- Test: `tests/vergil_tooling/test_vrg_host.py`

**Interfaces:**
- Consumes: `AuxCheck` (T1), `_run_check` (T3).
- Produces: `remediate_checks(dep) -> list[str]` (names of checks whose remediation ran), invoked by `upgrade` (T5) and a `status --fix` flag.

- [ ] **Step 1: Write the failing test** — gcloud's numpy check is present and its remediation is the documented pip install.

```python
def test_gcloud_entry_has_numpy_check():
    from vergil_tooling.lib import host_deps
    gcloud = {d.name: d for d in host_deps.load_manifest()}["gcloud"]
    names = [c.name for c in gcloud.checks]
    assert "numpy-in-interpreter" in names
```

- [ ] **Step 2: Run to verify it fails** (`gcloud` not yet in the seed).
Run: `uv run pytest tests/vergil_tooling/test_vrg_host.py::test_gcloud_entry_has_numpy_check -v` → FAIL (KeyError 'gcloud').

- [ ] **Step 3: Add the gcloud entry** (the #166 fix, generalized):

```toml
[[dependency]]
name = "gcloud"
why = "Google Cloud CLI; IAP tunnel for vrg-vm cloud sessions."
method = "custom"
upgrade = "auto"
latest = "probe"
[dependency.probes]
installed_version = ["gcloud", "version", "--format=value(Google Cloud SDK)"]
latest_version = ["gcloud", "components", "list", "--filter=id=core", "--format=value(latest_version_string)"]
upgrade = ["gcloud", "components", "update", "--quiet"]
[[dependency.check]]
name = "numpy-in-interpreter"
probe = ["bash", "-c", "\"$(gcloud info --format='value(basic.python_location)')\" -c 'import numpy'"]
remediation = ["bash", "-c", "\"$(gcloud info --format='value(basic.python_location)')\" -m pip install numpy"]
```

- [ ] **Step 4: Run to verify it passes.** Run the test → PASS. Then add a remediation unit test with a stubbed `_run` proving `remediate_checks` invokes the remediation only for unsatisfied checks.

- [ ] **Step 5: Commit**

```bash
vrg-git add src/vergil_tooling/bin/vrg_host.py src/vergil_tooling/data/host_dependencies.toml tests/vergil_tooling/test_vrg_host.py
vrg-commit --type feat --scope host --message "add auxiliary checks + gcloud numpy check (closes #166)"
```

> On merge, link and close vergil-project/.github#166 as superseded (spec §13).

---

## Task 5: `vrg-host upgrade` — auto/manual, pin-respecting, no escalation

**Files:**
- Modify: `src/vergil_tooling/bin/vrg_host.py`
- Test: `tests/vergil_tooling/test_vrg_host.py`

**Interfaces:**
- Consumes: `upgrade`/`UpgradeUnsupported` (T2), pin state (T1).
- Produces: `_upgrade(args) -> int`; `upgrade` subcommand accepting `deps...` and `--all`.

- [ ] **Step 1: Write the failing test**

```python
def test_upgrade_all_skips_manual_and_reports(monkeypatch, capsys):
    from vergil_tooling.lib.host_deps import HostDependency
    auto = HostDependency("gh","", "brew-formula","auto","probe",{},(),None)
    manual = HostDependency("macos","", "custom","manual","probe",{},(),None)
    monkeypatch.setattr(vrg_host.host_deps, "load_manifest", lambda: (auto, manual))
    calls = []
    monkeypatch.setattr(vrg_host.host_methods, "upgrade", lambda d: calls.append(d.name))
    rc = vrg_host.main(["upgrade", "--all"])
    out = capsys.readouterr().out
    assert calls == ["gh"]           # only auto upgraded
    assert "macos" in out and "manual" in out.lower()  # skipped, explicitly listed
    assert rc == 0
```

- [ ] **Step 2: Run to verify it fails.** → FAIL (no `upgrade` subcommand).

- [ ] **Step 3: Implement** `_upgrade`: resolve target deps (named or `--all`); for each, refuse if `pin.state == "active"` and target crosses the constraint (unless `--force`); call `host_methods.upgrade`, catching `UpgradeUnsupported` to collect manual skips; print the skipped-manual list; never invoke `sudo` (a handler that returns a privilege error is reported, not retried elevated). Add the `upgrade` subparser with `deps` (nargs="*") and `--all`/`--force`.

- [ ] **Step 4: Run to verify it passes.** → PASS. Add a test that a pinned dep is refused without `--force`.

- [ ] **Step 5: Commit**

```bash
vrg-commit --type feat --scope host --message "add vrg-host upgrade (auto/manual, pin-respecting)"
```

---

## Task 6: `vrg-host pin` / `unpin` + #155 lifecycle surfacing

**Files:**
- Create: `src/vergil_tooling/lib/host_pins.py`
- Modify: `src/vergil_tooling/bin/vrg_host.py`, `src/vergil_tooling/lib/host_status.py` (render pins as debt)
- Test: `tests/vergil_tooling/test_host_pins.py`

**Interfaces:**
- Consumes: `HostPin`/`HostDependency` (T1).
- Produces:
  - `write_pin(name, pin: HostPin) -> None`, `clear_pin(name) -> None` (rewrite the manifest TOML in place, preserving other entries).
  - `is_reevaluation_due(pin: HostPin, current_leading: str) -> bool` (#155 trigger: leading edge past `inducing_release`).
  - pin-debt rendering in `status` (constraint, latest held-back-from, reason, tracking issue, age).

- [ ] **Step 1: Write the failing test**

```python
# tests/vergil_tooling/test_host_pins.py
from vergil_tooling.lib import host_pins
from vergil_tooling.lib.host_deps import HostPin

def test_pin_requires_reason():
    import pytest
    with pytest.raises(ValueError, match="reason"):
        host_pins.make_pin(constraint="<=1.x", inducing_release="2.0",
                           deterministic=True, reason="", tracking_issue=None)

def test_reevaluation_due_when_leading_past_inducing():
    pin = HostPin("<=1.x", "2.0", True, "breaks us", "active", None)
    assert host_pins.is_reevaluation_due(pin, current_leading="2.1") is True
    assert host_pins.is_reevaluation_due(pin, current_leading="2.0") is False
```

- [ ] **Step 2: Run to verify it fails.** → FAIL (module missing).

- [ ] **Step 3: Implement** `host_pins.make_pin` (raises `ValueError` on empty reason — the justification gate; **stamps `pinned_date`** with today's date in ISO `YYYY-MM-DD`), `write_pin`/`clear_pin` (load TOML, splice the `[dependency.pin]` table for the named dep, write back), `is_reevaluation_due`. Wire `pin`/`unpin` subcommands (`pin` requires `--reason`; argparse enforces, and `make_pin` re-checks). Extend `render` to show active pins as debt lines — including **age** derived from `pinned_date` — and mark re-evaluation-due pins loudly.

- [ ] **Step 4: Run to verify it passes.** → PASS. Add a round-trip test: `write_pin` then `load_manifest` returns the pin; `clear_pin` removes it.

- [ ] **Step 5: Commit**

```bash
vrg-commit --type feat --scope host --message "add vrg-host pin/unpin with #155 lifecycle surfacing"
```

---

## Task 7: Enumerate & curate the authoritative manifest (spec §11)

**Files:**
- Modify: `src/vergil_tooling/data/host_dependencies.toml` (full curation)
- Modify: `src/vergil_tooling/lib/host_methods.py` (real brew/uv/gcloud output parsers + fixtures)
- Test: `tests/vergil_tooling/test_host_methods.py` (parser tests against recorded fixtures)

This is the investigative task. For each candidate host tool, determine and record in the manifest: Virgil-critical (in/out per spec §4)? `method`? `auto`/`manual`? `latest = probe` or `unmanaged`? which `check`s? The working set: gcloud (done T4), Docker (engine/Desktop — `manual`, note the OSS-runtime opportunity per spec §4), Lima, uv, gh, shellcheck, hadolint, yamllint, actionlint, the `vrg-*` tools, and macOS itself (`manual`, `latest = probe` via `softwareupdate -l` parsing, no upgrade). Harden the T2 handler parsers against **recorded real output fixtures** for each `method` (a captured `brew info --json=v2`, `uv tool list`, `gcloud components list`), so version extraction is tested without a live host. Each handler must **normalize** installed and latest to a bare, comparable version (spec §8): a fixture test must prove `"gh version 2.40.0"` and `"2.40.0"` compare **equal** (no false `behind`), and that an install *newer* than latest classifies `current`, not `behind`. Replace the T3 `_behind` "differs" stand-in with the normalized comparator per method; "differs" remains only the fallback when no comparator is registered.

- [ ] **Step 1:** Capture real command output fixtures under `tests/vergil_tooling/fixtures/host/` (one per method).
- [ ] **Step 2:** Write parser + normalization tests: the bare installed/latest version is extracted from each fixture, `"gh version 2.40.0"` normalizes equal to `"2.40.0"` (no false `behind`), and installed-newer-than-latest classifies `current`.
- [ ] **Step 3:** Implement the parsers to pass; fill the manifest entry per enumerated tool.
- [ ] **Step 4:** Run the full host test suite: `uv run pytest tests/vergil_tooling/test_host_*.py -v` → PASS.
- [ ] **Step 5:** Commit: `vrg-commit --type feat --scope host --message "enumerate and curate the host-dependency manifest"`.

---

## Validation (every task)

Run the repo's single validation entry point before reporting a task ready:

```bash
vrg-container-run -- vrg-validate
```

(Expands to `uv run vrg-validate` here via the `[validation]` override in `vergil.toml`.)

## Self-Review (spec coverage)

- §6 surface: `status` (T3), `upgrade` (T5), `pin`/`unpin` (T6). ✓
- §7 descriptor model + method handlers: T1 (schema/loader), T2 (handlers), T7 (curation). ✓
- §7.4 auxiliary checks + #166: T4. ✓
- §8 two-axis classification, `latest-unmanaged` vs `latest-unknown`, exit codes: T3. ✓
- §9 pin mechanism (#155 schema, justification gate, re-evaluation trigger, debt rendering): T6. ✓
- §10 invariants: manifest trust (T1 loads only from packaged data; no repo override path exists), unprivileged/no-escalation (T5). ✓
- §11 enumeration: T7. ✓
- §12 scheduling / §14 bookends: out of scope (follow-on #169) / handled by epic bookends. ✓

## Deferred / follow-on

- Real version comparators for non-semver schemes (T3 `_behind` is "differs" until T7 registers per-tool comparators where needed).
- Mechanizing the pin re-evaluation *trigger* as an automated sweep (spec §9) may become a later task or fold into the #169 follow-on; the manual `is_reevaluation_due` surfacing (T6) ships now.
