# Binary package publishing — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking. In the Vergil framework each **Task** below becomes one linked GitHub issue and one PR under epic `vergil-project/.github#356`, implemented via `issue-implement`. Each **operational task** (OP/DEP/VAL) is a `deployment`- or `validation`-kind issue run with `issue-deploy` / `issue-validate`, never `issue-implement`.

**Goal:** Vergil releases build signed `.deb`/`.rpm` packages, publish them into a per-org package repository on GitHub Pages, and vergil-tooling itself is installed from that repository on Lima and cloud VMs.

**Architecture:**

- **vergil-tooling** gains a `[package]` config section, a target registry, two builders (`staged` and `python`), nFPM packaging, install tests and index generation, all behind one CLI, `vrg-package` (Tasks T1–T9). It also switches VM provisioning to `apt` (T10).
- **vergil-actions** wires `vrg-package` into a PR gate (`ci-package.yml`), a split `cd-release.yml`, and a reusable `publish-index.yml` (A1–A3).
- **Two new repos** carry the first products: `vergil-project/packages` (the keyring product, plus the index) and `vergil-project/vergil-python` (the runtime) (P1, P2).
- **vergil-tooling adopts packaging** for itself (T11).
- **Operational tasks** stand up trust roots, deploy each bootstrap layer, and validate a cold rebuild.

**Tech Stack:** Python 3.12+ (`vergil_tooling`; strict mypy; pytest with a 100% coverage gate), nFPM, pyelftools, GnuPG, `rpm`/`rpmsign`, `createrepo_c`, `dpkg-deb`, GitHub Actions (arm64 runners, environments, Pages deploy API, `repository_dispatch`), uv, python-build-standalone (PBS).

**Spec:** `epics/356-binary-package-publishing/spec.md`. Read it alongside this plan; section references (§) point there.

## Global Constraints

- **Validation** is `vrg-container-run -- vrg-validate` only (the `[validation]` override expands it to `uv run vrg-validate` in vergil-tooling). Never run ad-hoc linters.
- **100% line and branch coverage** on all new or changed Python. mypy strict. The test layout is flat: `tests/vergil_tooling/test_<module>.py`.
- **One task = one branch = one PR**, `feature/<issue>-<slug>`, in a worktree under `.worktrees/`. Use `vrg-git`/`vrg-gh`/`vrg-commit` only; agents stop at `vrg-pr-workflow report-ready`.
- **Config errors are `ConfigError` raised from `lib/config.py`** (`raise ConfigError(msg)` with `msg = f"{source}: [package].<key> ..."`). Everything else in `lib/package/` raises `PackageError`. Nothing is skipped silently; the only deferred path is index publishing.
- **Default target set (verbatim, §5.1):** `ubuntu/24.04/{amd64,arm64}`, `ubuntu/26.04/{amd64,arm64}` → `.deb`; `rhel/9/{amd64,arm64}`, `rhel/10/{amd64,arm64}` → `.rpm`. Ubuntu suites: `noble` (24.04), `resolute` (26.04). EL suites: `el9`, `el10`.
- **Runners:** amd64 → `ubuntu-24.04`, arm64 → `ubuntu-24.04-arm`. The shared build image is `ubuntu:24.04`.
- **Install layout (§6.3):** runtime `/opt/<vendor>/python/<X.Y.Z>/`; product venv `/opt/<vendor>/<name>/venv/`; shims `/usr/bin/<cmd>`.
- **Package revision:** `1` for shared builds, `1~<suite>` (deb) or `1.<suite>` (rpm) for `native` builds.
- **Runtime dependency format:** deb `vergil-python<X.Y.Z> (>= <pbs-version>)`; rpm `vergil-python<X.Y.Z> >= <pbs-version>`.
- **Signing environments:**
  - `package-signing`, restricted to `main`, in every repo that releases packages. Used only by `cd-release`'s `package-sign` job.
  - `index-signing`, restricted to `develop`, in `vergil-project/packages` only. Used only by `publish-index`, which runs on the default branch because `repository_dispatch` and `schedule` always do.
  - Both hold `PACKAGE_SIGNING_KEY` (ASCII-armored secret subkey export) and `PACKAGE_SIGNING_PASSPHRASE`.
- **Attestation verification (verbatim, §7.2):** `--signer-workflow vergil-project/vergil-actions/.github/workflows/cd-release.yml --source-ref refs/heads/main`.
- **Retention defaults:** `keep = 3`, `lines = 2`. **Size guard:** warn above 750,000,000 bytes; fail above 900,000,000 bytes.
- **The vergil org repository base URL** is `https://vergil-project.github.io/packages`.

## Review Focus

These are inputs the spec implies but no headline test exercises. Each line has its test added to the owning task.

1. **Re-releasing a product without changing its package version.** For example, `vergil-python` is re-released with the same `name`/`version` but different bytes. The index must fail loudly, naming both releases, and never pick one at random. Owner: T6, test `test_duplicate_name_version_arch_with_different_bytes_is_fatal`.
2. **A release with a partial package set.** Its `packages-manifest.json` lists a `(fmt, arch)` that has no attached file. The index must fail and name the missing pair. Releases *without* a manifest are pre-packaging history and are ignored, with a printed note. Owner: T6, tests `test_partial_release_is_fatal` and `test_release_without_manifest_is_ignored`.
3. **A VM whose identity version names a line with no published package yet** (e.g. `v2.2` before the first packaged 2.2 release). Provisioning must fail with a message naming the line and the repository URL. It must not install whatever version is newest. Owner: T10, test `test_line_with_no_candidate_fails_with_actionable_message`.
4. **A matrix that resolves to zero targets** (`exclude = ["*/*/*"]`, or `targets` globs that match nothing). This is a config error, never an empty no-op build. Owner: T1, test `test_exclude_everything_is_an_error`.
5. **A VM that previously had a `uv` install** (legacy, or a dev install) and then gets the packaged install. The `uv` copy in `~/.local/bin` would shadow `/usr/bin`, so the packaged path must remove it. Owner: T10, test `test_packaged_install_removes_legacy_uv_install`.

## Sequencing and release gates

The pipeline bootstraps itself, so several tasks wait on a **human release** of an earlier layer. Releases are human actions. Each one is recorded as a human-attested precondition in the first operational task that needs it.

```text
OP1 (human: keys, environments, repos)
 │
 ├─ T1 ─┬─ T2 ─┬─ T4 (needs T3)
 │      │      └─ T5 (needs T3)
 │      ├─ T6 ─ T7
 │      ├─ T8
 │      └─ T3 (needs OP1) ─┬─ T9
 │                         └─ T10
 │   ── human release: vergil-tooling (T1–T9) ──
 ├─ A1 ─ A2 ; A3          (vergil-actions)
 │   ── human release: vergil-actions ──
 ├─ P1 (packages repo) → DEP1 (human release P1; index live with keyring)
 ├─ P2 (vergil-python)  → DEP2 (human release P2; runtime indexed)
 ├─ T11 (vergil-tooling adopts [package])
 │   ── human release: vergil-tooling (T10, T11) ──
 └─ DEP3 (packaged vergil-tooling indexed; vrg-vm update --all) → VAL1 (cold rebuild)
```

P1 and P2 are filed in repos that OP1 creates, so **OP1's final step files them** under the epic.

---

## OP1 (deployment, human-attested): trust roots, environments and repos

**Repo:** vergil-tooling (the issue lives here; the work is human-performed). Kind: `deployment`. Blocked by: nothing.

**Procedure** (every step is human-performed and attested in the SUCCESS comment; agents never handle the primary key):

- [ ] **1.** Generate the org primary key offline. It is certify-only, with a 10-year expiry:
  `gpg --quick-generate-key "vergil-project packages <packages@vergil-project.invalid>" ed25519 cert 10y`.
- [ ] **2.** Add a signing subkey (2-year expiry): `gpg --quick-add-key <PRIMARY_FPR> ed25519 sign 2y`.
- [ ] **3.** Export the public key (primary + subkey) to a file, `vergil.asc`:
  `gpg --armor --export <PRIMARY_FPR> > vergil.asc`.
- [ ] **4.** Export **only** the signing subkey's secret (the trailing `!` exports just that subkey):
  `gpg --armor --export-secret-subkeys <SUBKEY_FPR>! > subkey.asc`.
- [ ] **5.** Back up the primary key offline (password manager or offline media). Then delete it from any networked machine.
- [ ] **6.** Create the repos `vergil-project/packages` and `vergil-project/vergil-python`, both public, via `vrg-github-repo-init`. In `vergil-project/packages`, set **Settings → Pages → Source = GitHub Actions**.
- [ ] **7.** In each of `vergil-tooling`, `vergil-python` and `packages`:
  - create the environment `package-signing`, with deployment branches restricted to `main`;
  - add the secrets `PACKAGE_SIGNING_KEY` (the contents of `subkey.asc`) and `PACKAGE_SIGNING_PASSPHRASE`.

  In `packages` **also**: create the environment `index-signing` (deployment branches restricted to `develop`) with the same two secrets.
- [ ] **8.** Confirm that the org GitHub App (`APP_CLIENT_ID`/`APP_PRIVATE_KEY`) is installed on `vergil-project/packages` with `contents: write`, which `repository_dispatch` needs. Confirm the two secrets are available to `vergil-tooling` and `vergil-python`.
- [ ] **9.** Post the SUCCESS comment. It must include the **40-hex primary fingerprint** (T3 pins it) and the public key's armored text (P1 commits it as `keys/vergil.asc`).
- [ ] **10.** File P1 in `vergil-project/packages` and P2 in `vergil-project/vergil-python` under the epic, using the P1/P2 bodies below:
  `vrg-issue-create --epic vergil-project/.github#356 --repo vergil-project/<repo> --title …`.

**Acceptance:** all of the above attested, plus the fingerprint posted.

---

## Task T1: target registry, `[package]` config, validation, `vrg-package matrix`

**Repo:** vergil-tooling. Blocked by: nothing.

**Files:**

- Create: `src/vergil_tooling/lib/package/__init__.py` (defines `PackageError`)
- Create: `src/vergil_tooling/lib/package/targets.py`
- Create: `src/vergil_tooling/lib/package/matrix.py`
- Create: `src/vergil_tooling/lib/package/naming.py`
- Create: `src/vergil_tooling/bin/vrg_package.py`
- Modify: `src/vergil_tooling/lib/config.py`. Add `"package"` to `_KNOWN_SECTIONS` (L72-87) and `_KNOWN_KEYS` (L89-103). Add the `PackageConfig` dataclasses near `TestConfig` (L207). Add `_parse_package_config`. Wire the `package=` field in `_parse_raw_config` (L729-742) and into `VergilConfig` (L229-248). Add an overlay check in `read_config` (L745).
- Modify: `pyproject.toml`. Add `vrg-package = "vergil_tooling.bin.vrg_package:main"` to `[project.scripts]`.
- Test: `tests/vergil_tooling/test_package_targets.py`, `test_package_matrix.py`, `test_package_naming.py`, `test_vrg_package.py`, and additions to `test_config.py`.

**Interfaces:**

- Produces:
  - `lib.package.PackageError(Exception)`.
  - `targets.Target(distro, version, arch, glibc: tuple[int, int], image, suite)` with properties `key -> "distro/version/arch"`, `fmt -> "deb"|"rpm"` and `rpm_arch -> "x86_64"|"aarch64"`. Also `targets.all_targets() -> list[Target]` and `targets.match(pattern: str) -> list[Target]`.
  - `config.PackageConfig`, `config.PackagePythonConfig`, `config.PackageStagedConfig` (fields below); `VergilConfig.package: PackageConfig | None`.
  - `matrix.BuildCell(id, arch, runner, image, fmts: tuple[str, ...], targets: tuple[str, ...], native: bool, suite: str | None)`.
  - `matrix.TestCell(id, target, arch, runner, image, fmt, suite, native: bool)`.
  - `matrix.Matrix(build: tuple[BuildCell, ...], test: tuple[TestCell, ...])`.
  - `matrix.select_targets(pkg) -> list[Target]`, `matrix.resolve(pkg) -> Matrix`, `matrix.to_json(m) -> dict`, `matrix.manifest(m) -> dict`.
  - `naming.package_name(repo_root: Path, pkg: PackageConfig) -> str`.
  - CLI: `vrg-package matrix [--github-output] [--manifest PATH]`.

- [ ] **Step 1: Failing test, the registry.**

```python
# tests/vergil_tooling/test_package_targets.py
from vergil_tooling.lib.package import targets

def test_registry_is_the_full_two_by_two() -> None:
    keys = [t.key for t in targets.all_targets()]
    assert keys == [
        "rhel/10/amd64", "rhel/10/arm64", "rhel/9/amd64", "rhel/9/arm64",
        "ubuntu/24.04/amd64", "ubuntu/24.04/arm64", "ubuntu/26.04/amd64", "ubuntu/26.04/arm64",
    ]

def test_format_suite_and_rpm_arch() -> None:
    t = targets.REGISTRY["rhel/9/arm64"]
    assert (t.fmt, t.suite, t.rpm_arch, t.glibc) == ("rpm", "el9", "aarch64", (2, 34))
    u = targets.REGISTRY["ubuntu/24.04/amd64"]
    assert (u.fmt, u.suite, u.image) == ("deb", "noble", "ubuntu:24.04")

def test_match_globs() -> None:
    assert [t.key for t in targets.match("rhel/*/arm64")] == ["rhel/10/arm64", "rhel/9/arm64"]
    assert targets.match("debian/*/*") == []
```

- [ ] **Step 2: Run it and verify FAIL.**
  `uv run pytest tests/vergil_tooling/test_package_targets.py -v` fails on a missing module.

- [ ] **Step 3: Implement the registry.**

```python
# src/vergil_tooling/lib/package/__init__.py
"""Binary OS packaging (epic vergil-project/.github#356)."""

class PackageError(Exception):
    """A packaging failure. Always fatal to the command that raised it."""
```

```python
# src/vergil_tooling/lib/package/targets.py
"""Registry of supported package targets (spec §5.1)."""

from __future__ import annotations

from dataclasses import dataclass
from fnmatch import fnmatchcase


@dataclass(frozen=True)
class Target:
    distro: str
    version: str
    arch: str
    glibc: tuple[int, int]
    image: str
    suite: str

    @property
    def key(self) -> str:
        return f"{self.distro}/{self.version}/{self.arch}"

    @property
    def fmt(self) -> str:
        return "deb" if self.distro == "ubuntu" else "rpm"

    @property
    def rpm_arch(self) -> str:
        return {"amd64": "x86_64", "arm64": "aarch64"}[self.arch]


_ARCHES = ("amd64", "arm64")
# (version, suite, glibc, image). Verify glibc with `ldd --version` in each image (Step 4).
_UBUNTU = (
    ("24.04", "noble", (2, 39), "ubuntu:24.04"),
    ("26.04", "resolute", (2, 42), "ubuntu:26.04"),
)
_RHEL = (
    ("9", "el9", (2, 34), "registry.access.redhat.com/ubi9/ubi"),
    ("10", "el10", (2, 39), "registry.access.redhat.com/ubi10/ubi"),
)


def _build() -> dict[str, Target]:
    reg: dict[str, Target] = {}
    for distro, rows in (("ubuntu", _UBUNTU), ("rhel", _RHEL)):
        for version, suite, glibc, image in rows:
            for arch in _ARCHES:
                t = Target(distro, version, arch, glibc, image, suite)
                reg[t.key] = t
    return reg


REGISTRY: dict[str, Target] = _build()


def all_targets() -> list[Target]:
    return sorted(REGISTRY.values(), key=lambda t: t.key)


def match(pattern: str) -> list[Target]:
    return [t for t in all_targets() if fnmatchcase(t.key, pattern)]
```

- [ ] **Step 4: Verify the glibc values against the real images, then run the tests and verify PASS.**
  Run this for each image:
  `docker run --rm <image> sh -c 'ldd --version | head -1'`
  Images: `ubuntu:24.04`, `ubuntu:26.04`, `registry.access.redhat.com/ubi9/ubi` and `registry.access.redhat.com/ubi10/ubi`. Correct any `_UBUNTU`/`_RHEL` glibc tuple that differs (the `rhel/9` assertion in Step 1 pins 2.34), then re-run the tests.

- [ ] **Step 5: Failing tests, `[package]` config parsing.** Append these to `tests/vergil_tooling/test_config.py`, reusing its `_VALID_TOML`:

```python
_PKG_PY = _VALID_TOML + """
[package]
builder = "python"
vendor = "vergil"
summary = "Shared tooling"
smoke = "vrg-whoami --mode"

[package.python]
runtime = "3.14.4"
"""

def test_package_python_parses(tmp_path: Path) -> None:
    (tmp_path / "vergil.toml").write_text(_PKG_PY)
    pkg = read_config(tmp_path).package
    assert pkg is not None
    assert (pkg.builder, pkg.vendor, pkg.python.runtime if pkg.python else None) == ("python", "vergil", "3.14.4")
    assert (pkg.exclude, pkg.targets, pkg.native, pkg.noarch) == ([], None, [], False)

def test_no_package_section_is_none(tmp_path: Path) -> None:
    (tmp_path / "vergil.toml").write_text(_VALID_TOML)
    assert read_config(tmp_path).package is None

@pytest.mark.parametrize(
    ("extra", "match"),
    [
        ('exclude = ["x/y/z"]', r"\[package\]\.exclude pattern 'x/y/z' matches no target"),
        ('exclude = ["*/*/*"]', r"\[package\] selects no targets"),
        ('targets = ["ubuntu/*/*"]\nexclude = ["rhel/*/*"]', r"\[package\]: set at most one of exclude, targets"),
        ('native = ["rhel/*/*"]\ntargets = ["ubuntu/*/*"]', r"\[package\]\.native pattern 'rhel/\*/\*' matches no selected target"),
        ('builder = "cmake"', r"\[package\]\.builder must be one of python, staged"),
    ],
)
def test_package_errors(tmp_path: Path, extra: str, match: str) -> None:
    body = _PKG_PY.replace('builder = "python"\n', "") if extra.startswith("builder") else _PKG_PY
    (tmp_path / "vergil.toml").write_text(body.replace("[package]\n", f"[package]\n{extra}\n"))
    with pytest.raises(ConfigError, match=match):
        read_config(tmp_path)

def test_exclude_everything_is_an_error(tmp_path: Path) -> None:  # Review Focus 4
    (tmp_path / "vergil.toml").write_text(_PKG_PY.replace("[package]\n", '[package]\nexclude = ["*/*/*"]\n'))
    with pytest.raises(ConfigError, match=r"selects no targets"):
        read_config(tmp_path)

def test_python_builder_requires_runtime(tmp_path: Path) -> None:
    (tmp_path / "vergil.toml").write_text(_PKG_PY.replace('\n[package.python]\nruntime = "3.14.4"\n', ""))
    with pytest.raises(ConfigError, match=r"\[package\.python\]\.runtime is required"):
        read_config(tmp_path)

def test_runtime_must_be_exact_patch(tmp_path: Path) -> None:
    (tmp_path / "vergil.toml").write_text(_PKG_PY.replace('"3.14.4"', '"3.14"'))
    with pytest.raises(ConfigError, match=r"runtime must be an exact CPython patch"):
        read_config(tmp_path)

def test_cross_builder_subtable_is_an_error(tmp_path: Path) -> None:
    (tmp_path / "vergil.toml").write_text(_PKG_PY + '\n[package.staged]\nbuild-command = "x"\n')
    with pytest.raises(ConfigError, match=r"\[package\.staged\] is only valid with builder = \"staged\""):
        read_config(tmp_path)

_PKG_STAGED = _VALID_TOML + """
[package]
builder = "staged"
vendor = "vergil"
name = "vergil-archive-keyring"
summary = "Keyring"
smoke = "true"
noarch = true
"""

def test_staged_needs_command_or_overlay_files(tmp_path: Path) -> None:
    (tmp_path / "vergil.toml").write_text(_PKG_STAGED)
    with pytest.raises(ConfigError, match=r"builder = \"staged\" needs \[package\.staged\]\.build-command or overlay contents"):
        read_config(tmp_path)

def test_staged_with_overlay_contents_is_valid(tmp_path: Path) -> None:
    (tmp_path / "vergil.toml").write_text(_PKG_STAGED)
    (tmp_path / "packaging").mkdir()
    (tmp_path / "packaging" / "nfpm.overlay.yaml").write_text("contents:\n  - src: keys/vergil.asc\n    dst: /usr/share/keyrings/vergil-archive-keyring.asc\n")
    pkg = read_config(tmp_path).package
    assert pkg is not None and pkg.name == "vergil-archive-keyring" and pkg.noarch

def test_staged_requires_name(tmp_path: Path) -> None:
    (tmp_path / "vergil.toml").write_text(_PKG_STAGED.replace('name = "vergil-archive-keyring"\n', "") + '\n[package.staged]\nbuild-command = "x"\n')
    with pytest.raises(ConfigError, match=r"\[package\]\.name is required for builder = \"staged\""):
        read_config(tmp_path)

def test_overlay_must_be_a_mapping(tmp_path: Path) -> None:
    (tmp_path / "vergil.toml").write_text(_PKG_PY)
    (tmp_path / "packaging").mkdir()
    (tmp_path / "packaging" / "nfpm.overlay.yaml").write_text("- not a mapping\n")
    with pytest.raises(ConfigError, match=r"nfpm.overlay.yaml must be a YAML mapping"):
        read_config(tmp_path)

def test_overlay_may_not_set_identity_keys(tmp_path: Path) -> None:
    (tmp_path / "vergil.toml").write_text(_PKG_PY)
    (tmp_path / "packaging").mkdir()
    (tmp_path / "packaging" / "nfpm.overlay.yaml").write_text("name: other\n")
    with pytest.raises(ConfigError, match=r"overlay may not set 'name'"):
        read_config(tmp_path)
```

- [ ] **Step 6: Run and verify FAIL.**

- [ ] **Step 7: Implement the config section.** In `lib/config.py`:

```python
# _KNOWN_SECTIONS: add "package"
# _KNOWN_KEYS: add
"package": frozenset({
    "builder", "vendor", "name", "version", "summary", "smoke",
    "exclude", "targets", "native", "noarch", "python", "staged",
}),

_PACKAGE_BUILDERS = ("python", "staged")
_RUNTIME_RE = re.compile(r"^3\.\d+\.\d+$")
_VERSION_RE = re.compile(r"^[0-9][0-9A-Za-z.+~]*$")
_OVERLAY_PATH = "packaging/nfpm.overlay.yaml"
_OVERLAY_ALLOWED = frozenset({"contents", "scripts", "overrides", "provides", "conflicts", "replaces", "recommends", "suggests"})


@dataclass
class PackagePythonConfig:
    runtime: str
    commands: list[str] | None = None


@dataclass
class PackageStagedConfig:
    build_command: str | None = None


@dataclass
class PackageConfig:
    builder: str
    vendor: str
    summary: str
    smoke: str
    name: str | None = None
    version: str | None = None
    exclude: list[str] = field(default_factory=list)
    targets: list[str] | None = None
    native: list[str] = field(default_factory=list)
    noarch: bool = False
    python: PackagePythonConfig | None = None
    staged: PackageStagedConfig | None = None


def _str_list(value: Any, key: str, source: str) -> list[str]:
    if not isinstance(value, list) or not all(isinstance(v, str) for v in value):
        msg = f"{source}: [package].{key} must be a list of strings (got {value!r})"
        raise ConfigError(msg)
    return list(value)


def _req_str(raw: dict[str, Any], key: str, source: str, table: str = "package") -> str:
    value = raw.get(key)
    if not isinstance(value, str) or not value:
        msg = f"{source}: [{table}].{key} is required and must be a non-empty string"
        raise ConfigError(msg)
    return value


def _parse_package_config(raw: dict[str, Any], source: str = CONFIG_FILE) -> PackageConfig | None:
    p = raw.get("package")
    if p is None:
        return None
    builder = p.get("builder")
    if builder not in _PACKAGE_BUILDERS:
        msg = f"{source}: [package].builder must be one of {', '.join(_PACKAGE_BUILDERS)} (got {builder!r})"
        raise ConfigError(msg)
    pkg = PackageConfig(
        builder=builder,
        vendor=_req_str(p, "vendor", source),
        summary=_req_str(p, "summary", source),
        smoke=_req_str(p, "smoke", source),
        name=p.get("name"),
        version=p.get("version"),
        exclude=_str_list(p.get("exclude", []), "exclude", source),
        targets=_str_list(p["targets"], "targets", source) if "targets" in p else None,
        native=_str_list(p.get("native", []), "native", source),
        noarch=p.get("noarch", False),
    )
    if not isinstance(pkg.noarch, bool):
        msg = f"{source}: [package].noarch must be a boolean (got {pkg.noarch!r})"
        raise ConfigError(msg)
    if pkg.version is not None and not (isinstance(pkg.version, str) and _VERSION_RE.match(pkg.version)):
        msg = f"{source}: [package].version must match {_VERSION_RE.pattern} (got {pkg.version!r})"
        raise ConfigError(msg)
    if "python" in p and builder != "python":
        msg = f'{source}: [package.python] is only valid with builder = "python"'
        raise ConfigError(msg)
    if "staged" in p and builder != "staged":
        msg = f'{source}: [package.staged] is only valid with builder = "staged"'
        raise ConfigError(msg)
    if builder == "python":
        py = p.get("python", {})
        if "runtime" not in py:
            msg = f"{source}: [package.python].runtime is required for builder = \"python\""
            raise ConfigError(msg)
        if not (isinstance(py["runtime"], str) and _RUNTIME_RE.match(py["runtime"])):
            msg = f"{source}: [package.python].runtime must be an exact CPython patch like 3.14.4 (got {py['runtime']!r})"
            raise ConfigError(msg)
        cmds = py.get("commands")
        pkg.python = PackagePythonConfig(
            runtime=py["runtime"],
            commands=_str_list(cmds, "python.commands", source) if cmds is not None else None,
        )
    else:
        if not isinstance(pkg.name, str) or not pkg.name:
            msg = f'{source}: [package].name is required for builder = "staged"'
            raise ConfigError(msg)
        st = p.get("staged", {})
        cmd = st.get("build-command")
        if cmd is not None and not isinstance(cmd, str):
            msg = f"{source}: [package.staged].build-command must be a string (got {cmd!r})"
            raise ConfigError(msg)
        pkg.staged = PackageStagedConfig(build_command=cmd)
    # Late import: matrix depends on PackageConfig.
    from vergil_tooling.lib.package import matrix  # noqa: PLC0415

    matrix.select_targets(pkg, source=source)  # raises ConfigError
    return pkg


def _check_package_overlay(repo_root: Path, pkg: PackageConfig, source: str) -> None:
    path = repo_root / _OVERLAY_PATH
    overlay: dict[str, Any] = {}
    if path.is_file():
        loaded = yaml.safe_load(path.read_text()) or {}
        if not isinstance(loaded, dict):
            msg = f"{source}: {_OVERLAY_PATH} must be a YAML mapping"
            raise ConfigError(msg)
        for key in loaded:
            if key not in _OVERLAY_ALLOWED:
                msg = f"{source}: {_OVERLAY_PATH}: overlay may not set {key!r} (allowed: {', '.join(sorted(_OVERLAY_ALLOWED))})"
                raise ConfigError(msg)
        overlay = loaded
    if pkg.builder == "staged" and pkg.staged is not None and pkg.staged.build_command is None and not overlay.get("contents"):
        msg = f'{source}: builder = "staged" needs [package.staged].build-command or overlay contents in {_OVERLAY_PATH}'
        raise ConfigError(msg)
```

  Add `package: PackageConfig | None = None` to `VergilConfig`, and `package=_parse_package_config(raw, source)` in `_parse_raw_config`. In `read_config`, after parsing: `if cfg.package is not None: _check_package_overlay(repo_root, cfg.package, str(config_path))`. (`yaml` is already a runtime dependency.)

- [ ] **Step 8: Failing tests, the matrix.**

```python
# tests/vergil_tooling/test_package_matrix.py
from vergil_tooling.lib.config import PackageConfig, PackagePythonConfig
from vergil_tooling.lib.package import matrix

def _pkg(**kw: object) -> PackageConfig:
    base = dict(builder="python", vendor="vergil", summary="s", smoke="true",
                python=PackagePythonConfig(runtime="3.14.4"))
    base.update(kw)
    return PackageConfig(**base)  # type: ignore[arg-type]

def test_default_is_two_shared_cells_and_eight_tests() -> None:
    m = matrix.resolve(_pkg())
    assert [(c.id, c.runner, c.image, c.fmts) for c in m.build] == [
        ("shared-amd64", "ubuntu-24.04", "ubuntu:24.04", ("deb", "rpm")),
        ("shared-arm64", "ubuntu-24.04-arm", "ubuntu:24.04", ("deb", "rpm")),
    ]
    assert len(m.test) == 8
    assert {t.runner for t in m.test if t.arch == "arm64"} == {"ubuntu-24.04-arm"}

def test_exclude_drops_rhel_arm() -> None:
    m = matrix.resolve(_pkg(exclude=["rhel/*/arm64"]))
    assert [(c.id, c.fmts) for c in m.build] == [("shared-amd64", ("deb", "rpm")), ("shared-arm64", ("deb",))]
    assert len(m.test) == 6

def test_native_gets_its_own_cell_in_its_os() -> None:
    m = matrix.resolve(_pkg(native=["ubuntu/26.04/*"]))
    native = [c for c in m.build if c.native]
    assert [(c.id, c.image, c.fmts, c.suite) for c in native] == [
        ("native-ubuntu-26.04-amd64", "ubuntu:26.04", ("deb",), "resolute"),
        ("native-ubuntu-26.04-arm64", "ubuntu:26.04", ("deb",), "resolute"),
    ]
    shared_amd = next(c for c in m.build if c.id == "shared-amd64")
    assert "ubuntu/26.04/amd64" not in shared_amd.targets
    assert all(t.native for t in m.test if t.target.startswith("ubuntu/26.04"))

def test_noarch_builds_once_on_amd64() -> None:
    m = matrix.resolve(_pkg(noarch=True))
    assert [c.id for c in m.build] == ["shared-noarch"]
    assert len(m.test) == 8

def test_json_and_manifest_shapes() -> None:
    m = matrix.resolve(_pkg(exclude=["rhel/*/arm64"]))
    j = matrix.to_json(m)
    assert set(j) == {"build", "test"} and j["build"][0]["id"] == "shared-amd64"
    assert matrix.manifest(m) == {"artifacts": [
        {"fmt": "deb", "arch": "amd64", "suite": None}, {"fmt": "rpm", "arch": "amd64", "suite": None},
        {"fmt": "deb", "arch": "arm64", "suite": None},
    ]}
```

- [ ] **Step 9: Run and verify FAIL.**

- [ ] **Step 10: Implement `matrix.py`.**

```python
# src/vergil_tooling/lib/package/matrix.py
"""Resolve [package] into explicit build and test cells (spec §5.3)."""

from __future__ import annotations

from dataclasses import asdict, dataclass
from fnmatch import fnmatchcase
from typing import TYPE_CHECKING, Any

from vergil_tooling.lib.package import targets as tg

if TYPE_CHECKING:
    from vergil_tooling.lib.config import PackageConfig

RUNNERS = {"amd64": "ubuntu-24.04", "arm64": "ubuntu-24.04-arm"}
SHARED_IMAGE = "ubuntu:24.04"


@dataclass(frozen=True)
class BuildCell:
    id: str
    arch: str
    runner: str
    image: str
    fmts: tuple[str, ...]
    targets: tuple[str, ...]
    native: bool
    suite: str | None


@dataclass(frozen=True)
class TestCell:
    __test__ = False
    id: str
    target: str
    arch: str
    runner: str
    image: str
    fmt: str
    suite: str
    native: bool


@dataclass(frozen=True)
class Matrix:
    build: tuple[BuildCell, ...]
    test: tuple[TestCell, ...]


def _config_error(msg: str) -> Exception:
    from vergil_tooling.lib.config import ConfigError  # noqa: PLC0415

    return ConfigError(msg)


def select_targets(pkg: PackageConfig, *, source: str = "vergil.toml") -> list[tg.Target]:
    if pkg.targets is not None and pkg.exclude:
        raise _config_error(f"{source}: [package]: set at most one of exclude, targets")
    if pkg.targets is not None:
        chosen: dict[str, tg.Target] = {}
        for pat in pkg.targets:
            hits = tg.match(pat)
            if not hits:
                raise _config_error(f"{source}: [package].targets pattern {pat!r} matches no target")
            chosen.update({t.key: t for t in hits})
        selected = sorted(chosen.values(), key=lambda t: t.key)
    else:
        for pat in pkg.exclude:
            if not tg.match(pat):
                raise _config_error(f"{source}: [package].exclude pattern {pat!r} matches no target")
        selected = [t for t in tg.all_targets() if not any(fnmatchcase(t.key, p) for p in pkg.exclude)]
    if not selected:
        raise _config_error(f"{source}: [package] selects no targets")
    for pat in pkg.native:
        if not any(fnmatchcase(t.key, pat) for t in selected):
            raise _config_error(f"{source}: [package].native pattern {pat!r} matches no selected target")
    return selected


def resolve(pkg: PackageConfig) -> Matrix:
    selected = select_targets(pkg)
    is_native = {t.key: any(fnmatchcase(t.key, p) for p in pkg.native) for t in selected}
    build: list[BuildCell] = []
    shared = [t for t in selected if not is_native[t.key]]
    if pkg.noarch:
        fmts = tuple(sorted({t.fmt for t in shared}))
        build.append(BuildCell("shared-noarch", "amd64", RUNNERS["amd64"], SHARED_IMAGE, fmts,
                               tuple(t.key for t in shared), False, None))
    else:
        for arch in ("amd64", "arm64"):
            ts = [t for t in shared if t.arch == arch]
            if ts:
                build.append(BuildCell(f"shared-{arch}", arch, RUNNERS[arch], SHARED_IMAGE,
                                       tuple(sorted({t.fmt for t in ts})), tuple(t.key for t in ts), False, None))
    for t in selected:
        if is_native[t.key]:
            build.append(BuildCell(f"native-{t.distro}-{t.version}-{t.arch}", t.arch, RUNNERS[t.arch], t.image,
                                   (t.fmt,), (t.key,), True, t.suite))
    test = tuple(
        TestCell(f"test-{t.distro}-{t.version}-{t.arch}", t.key, t.arch, RUNNERS[t.arch], t.image, t.fmt,
                 t.suite, is_native[t.key])
        for t in selected
    )
    return Matrix(tuple(build), test)


def to_json(m: Matrix) -> dict[str, Any]:
    return {"build": [asdict(c) for c in m.build], "test": [asdict(c) for c in m.test]}


def manifest(m: Matrix) -> dict[str, Any]:
    arts: list[dict[str, Any]] = []
    for c in m.build:
        arch = "all" if c.id == "shared-noarch" else c.arch
        arts.extend({"fmt": f, "arch": arch, "suite": c.suite} for f in c.fmts)
    return {"artifacts": arts}
```

  (`select_targets`'s `ConfigError` messages are prefixed with `source` when called from config, and with the default `vergil.toml` from the CLI.)

- [ ] **Step 11: Failing tests, naming and the CLI.**

```python
# tests/vergil_tooling/test_package_naming.py
def test_python_name_comes_from_pyproject(tmp_path: Path) -> None:
    (tmp_path / "pyproject.toml").write_text('[project]\nname = "vergil-tooling"\n')
    assert naming.package_name(tmp_path, _pkg()) == "vergil-tooling"

def test_explicit_name_wins(tmp_path: Path) -> None:
    assert naming.package_name(tmp_path, _pkg(name="x")) == "x"

def test_python_without_pyproject_name_is_fatal(tmp_path: Path) -> None:
    (tmp_path / "pyproject.toml").write_text("[project]\n")
    with pytest.raises(PackageError, match=r"pyproject.toml has no \[project\]\.name"):
        naming.package_name(tmp_path, _pkg())
```

```python
# tests/vergil_tooling/test_vrg_package.py
def test_matrix_prints_json(tmp_path: Path, monkeypatch: pytest.MonkeyPatch, capsys: pytest.CaptureFixture[str]) -> None:
    (tmp_path / "vergil.toml").write_text(_PKG_PY_TOML)  # same text as test_config._PKG_PY
    monkeypatch.chdir(tmp_path)
    assert vrg_package.main(["matrix"]) == 0
    out = json.loads(capsys.readouterr().out)
    assert out["enabled"] is True and len(out["build"]) == 2

def test_matrix_github_output_and_manifest(tmp_path: Path, monkeypatch: pytest.MonkeyPatch) -> None:
    (tmp_path / "vergil.toml").write_text(_PKG_PY_TOML)
    gho = tmp_path / "gho"
    monkeypatch.chdir(tmp_path)
    monkeypatch.setenv("GITHUB_OUTPUT", str(gho))
    assert vrg_package.main(["matrix", "--github-output", "--manifest", "m.json"]) == 0
    lines = gho.read_text().splitlines()
    assert lines[0] == "enabled=true" and lines[1].startswith("build=[") and lines[2].startswith("test=[")
    assert json.loads((tmp_path / "m.json").read_text())["artifacts"]

def test_matrix_without_package_section_is_disabled(tmp_path: Path, monkeypatch: pytest.MonkeyPatch, capsys: pytest.CaptureFixture[str]) -> None:
    (tmp_path / "vergil.toml").write_text(_VALID_TOML_TEXT)
    monkeypatch.chdir(tmp_path)
    assert vrg_package.main(["matrix"]) == 0
    assert json.loads(capsys.readouterr().out) == {"enabled": False, "build": [], "test": []}

def test_config_error_returns_1(tmp_path: Path, monkeypatch: pytest.MonkeyPatch) -> None:
    (tmp_path / "vergil.toml").write_text(_PKG_PY_TOML.replace("[package]\n", '[package]\nexclude = ["*/*/*"]\n'))
    monkeypatch.chdir(tmp_path)
    assert vrg_package.main(["matrix"]) == 1
```

- [ ] **Step 12: Run and verify FAIL.**

- [ ] **Step 13: Implement `naming.py` and `bin/vrg_package.py`.**

```python
# src/vergil_tooling/lib/package/naming.py
from __future__ import annotations

import tomllib
from pathlib import Path
from typing import TYPE_CHECKING

from vergil_tooling.lib.package import PackageError

if TYPE_CHECKING:
    from vergil_tooling.lib.config import PackageConfig


def package_name(repo_root: Path, pkg: PackageConfig) -> str:
    if pkg.name:
        return pkg.name
    data = tomllib.loads((repo_root / "pyproject.toml").read_text())
    name = data.get("project", {}).get("name")
    if not isinstance(name, str) or not name:
        msg = "pyproject.toml has no [project].name; set [package].name"
        raise PackageError(msg)
    return name
```

```python
# src/vergil_tooling/bin/vrg_package.py
"""vrg-package — build, test and index binary OS packages (epic .github#356)."""

from __future__ import annotations

import argparse
import json
import os
import sys
from pathlib import Path

from vergil_tooling.lib import config
from vergil_tooling.lib.package import PackageError, matrix


def _cmd_matrix(args: argparse.Namespace) -> int:
    cfg = config.read_config(Path.cwd())
    if cfg.package is None:
        payload: dict[str, object] = {"enabled": False, "build": [], "test": []}
        m = None
    else:
        m = matrix.resolve(cfg.package)
        payload = {"enabled": True, **matrix.to_json(m)}
    if args.manifest and m is not None:
        Path(args.manifest).write_text(json.dumps(matrix.manifest(m), indent=2) + "\n")
    if args.github_output:
        with Path(os.environ["GITHUB_OUTPUT"]).open("a") as fh:
            fh.write(f"enabled={'true' if payload['enabled'] else 'false'}\n")
            fh.write(f"build={json.dumps(payload['build'])}\n")
            fh.write(f"test={json.dumps(payload['test'])}\n")
    else:
        print(json.dumps(payload, indent=2))
    return 0


def parse_args(argv: list[str] | None = None) -> argparse.Namespace:
    parser = argparse.ArgumentParser(prog="vrg-package", description=__doc__)
    sub = parser.add_subparsers(dest="command", required=True)
    p = sub.add_parser("matrix", help="Resolve [package] into build/test cells (JSON)")
    p.add_argument("--github-output", action="store_true", help="Write enabled/build/test to $GITHUB_OUTPUT")
    p.add_argument("--manifest", default="", help="Also write the release artifact manifest to this path")
    p.set_defaults(func=_cmd_matrix)
    return parser.parse_args(argv)


def main(argv: list[str] | None = None) -> int:
    args = parse_args(argv)
    try:
        return int(args.func(args))
    except (config.ConfigError, PackageError) as exc:
        print(f"ERROR: {exc}", file=sys.stderr)
        return 1


if __name__ == "__main__":
    sys.exit(main())
```

  Later tasks add subparsers to `parse_args` the same way: `build` (T2), `install-test` (T5), `index` (T7).

- [ ] **Step 14: Run all the new tests and verify PASS.** Then `vrg-container-run -- vrg-validate` should be green with 100% coverage.

- [ ] **Step 15: Commit.**
  `vrg-commit --type feat --scope package --message "target registry, [package] config and vrg-package matrix (#<T1>)"`

---

## Task T2: builder interface, `staged` builder, nFPM packaging, glibc guard, `vrg-package build`

**Repo:** vergil-tooling. Blocked by: T1.

**Files:**

- Create: `src/vergil_tooling/lib/package/build.py` (context, result, builder registry, orchestration)
- Create: `src/vergil_tooling/lib/package/staged.py`
- Create: `src/vergil_tooling/lib/package/nfpm.py` (config rendering, overlay merge, maintainer-script check, running nFPM)
- Create: `src/vergil_tooling/lib/package/elf.py` (glibc floor guard)
- Modify: `src/vergil_tooling/bin/vrg_package.py`. Add the `build` subcommand.
- Modify: `pyproject.toml`. Add `pyelftools>=0.31` to `dependencies`.
- Test: `tests/vergil_tooling/test_package_build.py`, `test_package_staged.py`, `test_package_nfpm.py`, `test_package_elf.py`, and additions to `test_vrg_package.py`.

**Interfaces:**

- Consumes: `matrix.BuildCell`, `matrix.resolve`, `naming.package_name`, `config.PackageConfig`.
- Produces:
  - `build.BuildContext(repo_root: Path, pkg: PackageConfig, cell: BuildCell, name: str, version: str, staging_root: Path, out_dir: Path)`.
  - `build.BuildResult(contents: list[dict[str, Any]], depends: dict[str, list[str]])`, where `depends` is keyed `"deb"`/`"rpm"`.
  - `build.BUILDERS: dict[str, Callable[[BuildContext], BuildResult]]`.
  - `build.contents_from_tree(root: Path, own_below: str) -> list[dict[str, Any]]`. It owns every directory strictly below `own_below`; `own_below` itself (e.g. the shared `/opt/vergil`) is never owned.
  - `build.package_release(cell: BuildCell, fmt: str) -> str`.
  - `build.run_build(repo_root: Path, cell_id: str, version: str, out_dir: Path, staging_root: Path) -> list[Path]`.
  - `nfpm.render(ctx: BuildContext, result: BuildResult, fmt: str, overlay: dict[str, Any]) -> dict[str, Any]`.
  - `nfpm.check_maintainer_scripts(repo_root: Path, overlay: dict[str, Any]) -> None`.
  - `nfpm.package(config: dict[str, Any], fmt: str, out_dir: Path) -> Path`.
  - `elf.glibc_requirements(path: Path) -> set[tuple[int, int]]`; `elf.check_glibc_floor(root: Path, floor: tuple[int, int]) -> None` (raises `PackageError` listing every violating file).
  - CLI: `vrg-package build --cell ID --version V [--out DIR] [--staging DIR]`.

- [ ] **Step 1: Failing tests, `contents_from_tree` and the package release.**

```python
# tests/vergil_tooling/test_package_build.py
def test_contents_owns_dirs_below_prefix_files_and_symlinks(tmp_path: Path) -> None:
    root = tmp_path / "stage"
    (root / "opt/vergil/x/bin").mkdir(parents=True)
    (root / "opt/vergil/x/bin/tool").write_text("#!/bin/sh\n")
    (root / "opt/vergil/x/bin/tool").chmod(0o755)
    (root / "etc/vergil").mkdir(parents=True)
    (root / "etc/vergil/x.conf").write_text("a=1\n")
    (root / "opt/vergil/x/bin/link").symlink_to("tool")
    got = build.contents_from_tree(root, own_below="/opt/vergil")
    assert {"dst": "/opt/vergil/x", "type": "dir"} in got
    assert {"dst": "/opt/vergil/x/bin", "type": "dir"} in got
    assert not any(e.get("dst") in ("/opt", "/opt/vergil", "/etc", "/etc/vergil") and e.get("type") == "dir" for e in got)
    tool = next(e for e in got if e["dst"] == "/opt/vergil/x/bin/tool")
    assert tool["file_info"]["mode"] == 0o755
    assert {"src": "tool", "dst": "/opt/vergil/x/bin/link", "type": "symlink"} in got

def test_package_release_revision() -> None:
    shared = BuildCell("shared-amd64", "amd64", "r", "i", ("deb", "rpm"), (), False, None)
    native = BuildCell("native-ubuntu-26.04-amd64", "amd64", "r", "i", ("deb",), (), True, "resolute")
    native_rpm = BuildCell("native-rhel-10-amd64", "amd64", "r", "i", ("rpm",), (), True, "el10")
    assert build.package_release(shared, "deb") == "1"
    assert build.package_release(native, "deb") == "1~resolute"
    assert build.package_release(native_rpm, "rpm") == "1.el10"
```

- [ ] **Step 2: Run and verify FAIL.**

- [ ] **Step 3: Implement the core of `build.py`.**

```python
# src/vergil_tooling/lib/package/build.py
"""Builder interface and build orchestration (spec §6)."""

from __future__ import annotations

import os
import shutil
from collections.abc import Callable
from dataclasses import dataclass, field
from pathlib import Path
from typing import Any

from vergil_tooling.lib.config import PackageConfig, read_config
from vergil_tooling.lib.package import PackageError, elf, matrix, naming, nfpm
from vergil_tooling.lib.package import targets as tg
from vergil_tooling.lib.package.matrix import BuildCell


@dataclass(frozen=True)
class BuildContext:
    repo_root: Path
    pkg: PackageConfig
    cell: BuildCell
    name: str
    version: str
    staging_root: Path
    out_dir: Path


@dataclass
class BuildResult:
    contents: list[dict[str, Any]]
    depends: dict[str, list[str]] = field(default_factory=lambda: {"deb": [], "rpm": []})


Builder = Callable[[BuildContext], BuildResult]
BUILDERS: dict[str, Builder] = {}


def register(kind: str) -> Callable[[Builder], Builder]:
    def deco(fn: Builder) -> Builder:
        BUILDERS[kind] = fn
        return fn

    return deco


def contents_from_tree(root: Path, own_below: str) -> list[dict[str, Any]]:
    out: list[dict[str, Any]] = []
    prefix = own_below.rstrip("/")
    for dirpath, dirnames, filenames in os.walk(root):
        dirnames.sort()
        rel_dir = "/" + str(Path(dirpath).relative_to(root)).lstrip(".")
        rel_dir = rel_dir.rstrip("/") or "/"
        if rel_dir.startswith(prefix + "/"):
            out.append({"dst": rel_dir, "type": "dir"})
        for name in sorted(filenames + [d for d in dirnames if (Path(dirpath) / d).is_symlink()]):
            path = Path(dirpath) / name
            dst = (rel_dir.rstrip("/") + "/" + name)
            if path.is_symlink():
                out.append({"src": os.readlink(path), "dst": dst, "type": "symlink"})
            else:
                out.append({"src": str(path), "dst": dst, "file_info": {"mode": path.stat().st_mode & 0o7777}})
    return out


def package_release(cell: BuildCell, fmt: str) -> str:
    if not cell.native:
        return "1"
    return f"1~{cell.suite}" if fmt == "deb" else f"1.{cell.suite}"


def glibc_floor(cell: BuildCell) -> tuple[int, int]:
    return min(tg.REGISTRY[k].glibc for k in cell.targets)


def run_build(repo_root: Path, cell_id: str, version: str, out_dir: Path, staging_root: Path) -> list[Path]:
    cfg = read_config(repo_root)
    if cfg.package is None:
        msg = "vergil.toml has no [package] section"
        raise PackageError(msg)
    pkg = cfg.package
    cells = {c.id: c for c in matrix.resolve(pkg).build}
    if cell_id not in cells:
        msg = f"unknown build cell {cell_id!r} (known: {', '.join(cells)})"
        raise PackageError(msg)
    cell = cells[cell_id]
    if staging_root.exists():
        shutil.rmtree(staging_root)
    staging_root.mkdir(parents=True)
    out_dir.mkdir(parents=True, exist_ok=True)
    ctx = BuildContext(repo_root, pkg, cell, naming.package_name(repo_root, pkg),
                       pkg.version or version, staging_root, out_dir)
    result = BUILDERS[pkg.builder](ctx)
    elf.check_glibc_floor(staging_root, glibc_floor(cell))
    overlay = nfpm.load_overlay(repo_root)
    nfpm.check_maintainer_scripts(repo_root, overlay)
    if not result.contents and not overlay.get("contents"):
        msg = f"build produced an empty package for {ctx.name} (cell {cell_id})"
        raise PackageError(msg)
    return [nfpm.package(nfpm.render(ctx, result, fmt, overlay), fmt, out_dir) for fmt in cell.fmts]
```

  `staged` and (in T4) `python` register via `@register(...)` and are imported in `bin/vrg_package.py` so the registry is populated. (Note: `contents_from_tree` treats symlinks-to-directories as entries, never descending into them.)

- [ ] **Step 4: Run and verify PASS.**

- [ ] **Step 5: Failing tests, the glibc guard.**

```python
# tests/vergil_tooling/test_package_elf.py
import os, sys
from pathlib import Path

import pytest

from vergil_tooling.lib.package import PackageError, elf

def test_non_elf_has_no_requirements(tmp_path: Path) -> None:
    f = tmp_path / "x.txt"
    f.write_text("hello")
    assert elf.glibc_requirements(f) == set()

def test_real_interpreter_requires_some_glibc() -> None:
    reqs = elf.glibc_requirements(Path(os.path.realpath(sys.executable)))
    assert reqs and all(r >= (2, 2) for r in reqs)

def test_floor_violation_lists_every_file(tmp_path: Path, monkeypatch: pytest.MonkeyPatch) -> None:
    for n in ("a.so", "b.so", "ok.so"):
        (tmp_path / n).write_bytes(b"\x7fELF")
    fake = {"a.so": {(2, 38)}, "b.so": {(2, 35), (2, 17)}, "ok.so": {(2, 17)}}
    monkeypatch.setattr(elf, "glibc_requirements", lambda p: fake[p.name])
    with pytest.raises(PackageError, match=r"(?s)GLIBC floor 2\.34.*a\.so needs 2\.38.*b\.so needs 2\.35"):
        elf.check_glibc_floor(tmp_path, (2, 34))

def test_floor_ok(tmp_path: Path, monkeypatch: pytest.MonkeyPatch) -> None:
    (tmp_path / "ok.so").write_bytes(b"\x7fELF")
    monkeypatch.setattr(elf, "glibc_requirements", lambda p: {(2, 17)})
    elf.check_glibc_floor(tmp_path, (2, 34))
```

  (Validation runs in the Linux dev container, so `sys.executable` is an ELF binary there.)

- [ ] **Step 6: Run, verify FAIL, then implement `elf.py`.**

```python
# src/vergil_tooling/lib/package/elf.py
"""glibc floor guard (spec §6.1 step 4)."""

from __future__ import annotations

import re
from pathlib import Path

from elftools.common.exceptions import ELFError
from elftools.elf.elffile import ELFFile
from elftools.elf.gnuversions import GNUVerNeedSection

from vergil_tooling.lib.package import PackageError

_GLIBC = re.compile(r"GLIBC_(\d+)\.(\d+)(?:\.\d+)?")


def glibc_requirements(path: Path) -> set[tuple[int, int]]:
    with path.open("rb") as fh:
        if fh.read(4) != b"\x7fELF":
            return set()
        fh.seek(0)
        try:
            elffile = ELFFile(fh)
        except ELFError as exc:
            msg = f"{path}: unreadable ELF ({exc})"
            raise PackageError(msg) from exc
        out: set[tuple[int, int]] = set()
        for section in elffile.iter_sections():
            if isinstance(section, GNUVerNeedSection):
                for _verneed, auxiter in section.iter_versions():
                    for aux in auxiter:
                        m = _GLIBC.fullmatch(aux.name)
                        if m:
                            out.add((int(m[1]), int(m[2])))
        return out


def check_glibc_floor(root: Path, floor: tuple[int, int]) -> None:
    bad: list[str] = []
    for path in sorted(p for p in root.rglob("*") if p.is_file() and not p.is_symlink()):
        reqs = glibc_requirements(path)
        if reqs and max(reqs) > floor:
            need = max(reqs)
            bad.append(f"  {path.relative_to(root)} needs {need[0]}.{need[1]}")
    if bad:
        msg = (f"GLIBC floor {floor[0]}.{floor[1]} exceeded (a dependency was compiled against a newer glibc; "
               "use a manylinux wheel or a [package].native build):\n" + "\n".join(bad))
        raise PackageError(msg)
```

  Add a test that a corrupt ELF (`b"\x7fELF" + b"\x00" * 8`) raises `PackageError` with "unreadable ELF". Run again and verify PASS.

- [ ] **Step 7: Failing tests, nFPM rendering, the overlay, and the maintainer-script check.**

```python
# tests/vergil_tooling/test_package_nfpm.py
def _ctx(tmp_path: Path, **pkgkw: object) -> BuildContext:
    cell = BuildCell("shared-amd64", "amd64", "r", "i", ("deb", "rpm"), ("ubuntu/24.04/amd64",), False, None)
    return BuildContext(tmp_path, _pkg(**pkgkw), cell, "vergil-tooling", "2.1.240", tmp_path / "s", tmp_path / "o")

def test_render_identity_and_depends(tmp_path: Path) -> None:
    res = BuildResult(contents=[{"dst": "/opt/vergil/vergil-tooling", "type": "dir"}],
                      depends={"deb": ["vergil-python3.14.4 (>= 3.14.4+20261001)"], "rpm": ["vergil-python3.14.4 >= 3.14.4+20261001"]})
    cfg = nfpm.render(_ctx(tmp_path), res, "rpm", {})
    assert cfg["name"] == "vergil-tooling" and cfg["version"] == "2.1.240" and cfg["release"] == "1"
    assert cfg["version_schema"] == "none" and cfg["arch"] == "amd64" and cfg["platform"] == "linux"
    assert cfg["depends"] == ["vergil-python3.14.4 >= 3.14.4+20261001"]
    assert cfg["contents"] == res.contents

def test_noarch_renders_all(tmp_path: Path) -> None:
    ctx = _ctx(tmp_path, noarch=True)
    assert nfpm.render(ctx, BuildResult(contents=[]), "deb", {})["arch"] == "all"

def test_overlay_appends_contents_and_depends(tmp_path: Path) -> None:
    overlay = {"contents": [{"src": "x", "dst": "/etc/x"}],
               "overrides": {"deb": {"depends": ["adduser"]}}, "scripts": {"postinstall": "packaging/postinst.sh"}}
    cfg = nfpm.render(_ctx(tmp_path), BuildResult(contents=[{"dst": "/a", "type": "dir"}]), "deb", overlay)
    assert cfg["contents"][-1]["dst"] == "/etc/x"
    assert cfg["depends"] == ["adduser"]
    assert cfg["scripts"]["postinstall"] == str(tmp_path / "packaging/postinst.sh")

def test_overlay_contents_src_is_repo_relative(tmp_path: Path) -> None:
    cfg = nfpm.render(_ctx(tmp_path), BuildResult(contents=[]), "deb", {"contents": [{"src": "keys/k.asc", "dst": "/k"}]})
    assert cfg["contents"][0]["src"] == str(tmp_path / "keys/k.asc")

def test_raw_systemctl_in_maintainer_script_is_fatal(tmp_path: Path) -> None:
    (tmp_path / "packaging").mkdir()
    (tmp_path / "packaging/postinst.sh").write_text("#!/bin/sh\nsystemctl enable --now x.service\n")
    with pytest.raises(PackageError, match=r"postinst\.sh:2: raw 'systemctl'"):
        nfpm.check_maintainer_scripts(tmp_path, {"scripts": {"postinstall": "packaging/postinst.sh"}})

def test_helper_idioms_are_allowed(tmp_path: Path) -> None:
    (tmp_path / "packaging").mkdir()
    (tmp_path / "packaging/postinst.sh").write_text(
        "#!/bin/sh\ndeb-systemd-helper enable x.service\ndeb-systemd-invoke start x.service\n"
        "[ -x /usr/lib/systemd/systemd-update-helper ] && /usr/lib/systemd/systemd-update-helper install-system-units x.service\n")
    nfpm.check_maintainer_scripts(tmp_path, {"scripts": {"postinstall": "packaging/postinst.sh"}})

def test_package_runs_nfpm_and_returns_the_artifact(tmp_path: Path, monkeypatch: pytest.MonkeyPatch) -> None:
    calls: list[list[str]] = []
    def fake_run(cmd: list[str], **kw: object) -> subprocess.CompletedProcess[str]:
        calls.append(cmd)
        (tmp_path / "o").mkdir(exist_ok=True)
        (tmp_path / "o" / "vergil-tooling_2.1.240-1_amd64.deb").write_bytes(b"x")
        return subprocess.CompletedProcess(cmd, 0, "", "")
    monkeypatch.setattr(nfpm.shutil, "which", lambda _: "/usr/bin/nfpm")
    monkeypatch.setattr(nfpm.subprocess, "run", fake_run)
    got = nfpm.package({"name": "vergil-tooling", "version": "2.1.240", "release": "1", "arch": "amd64"}, "deb", tmp_path / "o")
    assert got.name == "vergil-tooling_2.1.240-1_amd64.deb"
    assert calls[0][:2] == ["nfpm", "package"] and "--packager" in calls[0]

def test_missing_nfpm_is_fatal(monkeypatch: pytest.MonkeyPatch, tmp_path: Path) -> None:
    monkeypatch.setattr(nfpm.shutil, "which", lambda _: None)
    with pytest.raises(PackageError, match=r"nfpm not found on PATH"):
        nfpm.package({}, "deb", tmp_path)
```

- [ ] **Step 8: Run, verify FAIL, then implement `nfpm.py`.**

```python
# src/vergil_tooling/lib/package/nfpm.py
"""nFPM config rendering and invocation (spec §5.2, §6.5)."""

from __future__ import annotations

import re
import shutil
import subprocess
from pathlib import Path
from typing import TYPE_CHECKING, Any

import yaml

from vergil_tooling.lib.package import PackageError

if TYPE_CHECKING:
    from vergil_tooling.lib.package.build import BuildContext, BuildResult

OVERLAY_PATH = "packaging/nfpm.overlay.yaml"
_RAW_SYSTEMCTL = re.compile(r"(?<![\w/-])systemctl\b")


def load_overlay(repo_root: Path) -> dict[str, Any]:
    path = repo_root / OVERLAY_PATH
    if not path.is_file():
        return {}
    data = yaml.safe_load(path.read_text()) or {}
    return dict(data)  # shape already validated by config._check_package_overlay


def render(ctx: BuildContext, result: BuildResult, fmt: str, overlay: dict[str, Any]) -> dict[str, Any]:
    from vergil_tooling.lib.package.build import package_release  # noqa: PLC0415

    contents = list(result.contents)
    for entry in overlay.get("contents", []):
        e = dict(entry)
        if "src" in e and e.get("type") != "symlink" and not str(e["src"]).startswith("/"):
            e["src"] = str(ctx.repo_root / e["src"])
        contents.append(e)
    depends = list(result.depends.get(fmt, []))
    depends += list(overlay.get("overrides", {}).get(fmt, {}).get("depends", []))
    cfg: dict[str, Any] = {
        "name": ctx.name,
        "arch": "all" if ctx.pkg.noarch else ctx.cell.arch,
        "platform": "linux",
        "version": ctx.version,
        "version_schema": "none",
        "release": package_release(ctx.cell, fmt),
        "maintainer": f"{ctx.pkg.vendor} packages <packages@{ctx.pkg.vendor}.invalid>",
        "description": ctx.pkg.summary,
        "vendor": ctx.pkg.vendor,
        "contents": contents,
        "depends": depends,
    }
    if "scripts" in overlay:
        cfg["scripts"] = {k: str(ctx.repo_root / v) for k, v in overlay["scripts"].items()}
    for key in ("provides", "conflicts", "replaces", "recommends", "suggests"):
        if key in overlay:
            cfg[key] = overlay[key]
    return cfg


def check_maintainer_scripts(repo_root: Path, overlay: dict[str, Any]) -> None:
    for rel in overlay.get("scripts", {}).values():
        path = repo_root / rel
        for lineno, line in enumerate(path.read_text().splitlines(), start=1):
            stripped = line.split("#", 1)[0]
            if _RAW_SYSTEMCTL.search(stripped):
                msg = (f"{rel}:{lineno}: raw 'systemctl' in a maintainer script — use deb-systemd-helper/"
                       "deb-systemd-invoke (Ubuntu) or /usr/lib/systemd/systemd-update-helper (RHEL) (spec §6.5)")
                raise PackageError(msg)


def package(config: dict[str, Any], fmt: str, out_dir: Path) -> Path:
    if shutil.which("nfpm") is None:
        msg = "nfpm not found on PATH (CI installs it via actions/shared/setup/nfpm)"
        raise PackageError(msg)
    out_dir.mkdir(parents=True, exist_ok=True)
    cfg_path = out_dir / f"nfpm-{fmt}.yaml"
    cfg_path.write_text(yaml.safe_dump(config, sort_keys=False))
    before = set(out_dir.iterdir())
    subprocess.run(["nfpm", "package", "--config", str(cfg_path), "--packager", fmt, "--target", str(out_dir)],
                   check=True, capture_output=True, text=True)
    produced = [p for p in set(out_dir.iterdir()) - before if p.suffix == f".{fmt}"]
    if len(produced) != 1:
        msg = f"nfpm produced {len(produced)} .{fmt} files in {out_dir}, expected exactly 1"
        raise PackageError(msg)
    return produced[0]
```

  The `_RAW_SYSTEMCTL` lookbehind excludes `deb-systemd-…` (hyphen) and `/usr/lib/systemd/systemd-update-helper` (a slash path, not the word `systemctl`). Run again and verify PASS.

- [ ] **Step 9: Failing tests, the `staged` builder.**

```python
# tests/vergil_tooling/test_package_staged.py
def test_build_command_receives_env_and_populates_root(tmp_path: Path) -> None:
    (tmp_path / "packaging").mkdir()
    script = tmp_path / "packaging/build.sh"
    script.write_text('#!/bin/sh\nset -e\nmkdir -p "$VRG_STAGING_ROOT/opt/vergil/python/3.14.4/bin"\n'
                      'echo "$VRG_TARGET_ARCH" > "$VRG_STAGING_ROOT/opt/vergil/python/3.14.4/bin/arch"\n')
    script.chmod(0o755)
    ctx = _staged_ctx(tmp_path, build_command="packaging/build.sh", name="vergil-python3.14.4")
    res = staged.build_staged(ctx)
    assert (ctx.staging_root / "opt/vergil/python/3.14.4/bin/arch").read_text().strip() == "amd64"
    assert {"dst": "/opt/vergil/python", "type": "dir"} in res.contents
    assert {"dst": "/opt/vergil/python/3.14.4", "type": "dir"} in res.contents
    assert not any(e["dst"] == "/opt/vergil" and e.get("type") == "dir" for e in res.contents)

def test_failing_command_is_fatal(tmp_path: Path) -> None:
    ctx = _staged_ctx(tmp_path, build_command="exit 3", name="x")
    with pytest.raises(PackageError, match=r"build-command failed \(exit 3\)"):
        staged.build_staged(ctx)

def test_overlay_only_returns_no_contents(tmp_path: Path) -> None:
    ctx = _staged_ctx(tmp_path, build_command=None, name="vergil-archive-keyring")
    assert staged.build_staged(ctx).contents == []
```

  Here `_staged_ctx` builds a `BuildContext` with `pkg.builder="staged"`, `pkg.vendor="vergil"`, the given name and command, `cell=shared-amd64`, and `staging_root=tmp_path/"s"` (already created).

- [ ] **Step 10: Run, verify FAIL, then implement `staged.py`.**

```python
# src/vergil_tooling/lib/package/staged.py
"""The `staged` builder (spec §6.6)."""

from __future__ import annotations

import os
import subprocess

from vergil_tooling.lib.package import PackageError
from vergil_tooling.lib.package.build import BuildContext, BuildResult, contents_from_tree, register


@register("staged")
def build_staged(ctx: BuildContext) -> BuildResult:
    staged = ctx.pkg.staged
    cmd = staged.build_command if staged else None
    if cmd:
        env = {**os.environ, "VRG_STAGING_ROOT": str(ctx.staging_root), "VRG_TARGET_ARCH": ctx.cell.arch}
        proc = subprocess.run(["bash", "-c", cmd], cwd=ctx.repo_root, env=env, check=False)
        if proc.returncode != 0:
            msg = f"[package.staged].build-command failed (exit {proc.returncode}): {cmd}"
            raise PackageError(msg)
    if not any(ctx.staging_root.iterdir()):
        return BuildResult(contents=[])
    return BuildResult(contents=contents_from_tree(ctx.staging_root, own_below=f"/opt/{ctx.pkg.vendor}"))
```

  Every directory strictly below `/opt/<vendor>` is owned, so `/opt/vergil/python` is owned by each runtime package. That is safe because dpkg and rpm both reference-count shared directories. `/opt/vergil` itself is never owned. Run and verify PASS.

- [ ] **Step 11: Failing test, then implement the `build` subcommand.**

```python
def test_build_subcommand_wires_run_build(tmp_path: Path, monkeypatch: pytest.MonkeyPatch, capsys: pytest.CaptureFixture[str]) -> None:
    seen: dict[str, object] = {}
    def fake(repo_root: Path, cell_id: str, version: str, out_dir: Path, staging_root: Path) -> list[Path]:
        seen.update(cell=cell_id, version=version, out=out_dir)
        return [out_dir / "a.deb"]
    monkeypatch.setattr(vrg_package.build, "run_build", fake)
    monkeypatch.chdir(tmp_path)
    assert vrg_package.main(["build", "--cell", "shared-amd64", "--version", "2.1.240", "--out", "dist"]) == 0
    assert seen == {"cell": "shared-amd64", "version": "2.1.240", "out": tmp_path / "dist"}
    assert "a.deb" in capsys.readouterr().out
```

  In `vrg_package.py`: `from vergil_tooling.lib.package import build, staged  # noqa: F401  (registers builders)`. Add a `build` parser with `--cell` (required), `--version` (required), `--out` (default `dist/packages`) and `--staging` (default `.vergil/package-staging`). `_cmd_build` resolves the paths against `Path.cwd()`, calls `build.run_build`, and prints each artifact path. Add an integration-style test of `run_build` too: a staged overlay-only product in `tmp_path`, with `nfpm.package` monkeypatched, asserting it's called once per format in the cell. Also add `test_unknown_cell_is_fatal` and `test_empty_package_is_fatal` (a staged product whose command produces nothing and has no overlay contents).

- [ ] **Step 12: `vrg-container-run -- vrg-validate` green; commit.**
  `vrg-commit --type feat --scope package --message "staged builder, nFPM packaging, glibc guard, vrg-package build (#<T2>)"`

---

## Task T3: repository trust bootstrap (org registry, fingerprint pin, apt/dnf setup)

**Repo:** vergil-tooling. Blocked by: T1, OP1 (needs the attested fingerprint).

**Files:**

- Create: `src/vergil_tooling/lib/package/orgs.py`
- Create: `src/vergil_tooling/lib/package/repo_setup.py`
- Test: `tests/vergil_tooling/test_package_orgs.py`, `test_package_repo_setup.py`

**Interfaces:**

- Produces:
  - `orgs.OrgRepo(vendor, github_org, base_url, fingerprint, keyring_package)` and `orgs.ORGS: dict[str, OrgRepo]`. `orgs.for_vendor(vendor) -> OrgRepo` raises `PackageError` for an unknown vendor.
  - `repo_setup.Run = Callable[..., subprocess.CompletedProcess[str]]`, called as `run(*argv)` and raising `CalledProcessError` on a nonzero exit. `Transport.run` already satisfies this.
  - `repo_setup.parse_primary_fingerprint(colons: str) -> str`.
  - `repo_setup.bootstrap(run: Run, org: OrgRepo, fmt: str, suite: str, *, sudo: bool) -> None`. It installs prerequisites, fetches and verifies the key, writes bootstrap sources, installs `org.keyring_package`, and removes the bootstrap files.
  - `repo_setup.local_run(*argv: str) -> CompletedProcess[str]`.

- [ ] **Step 1: Failing tests.**

```python
# tests/vergil_tooling/test_package_repo_setup.py
_COLONS = """\
pub:u:255:22:ABCDEF0123456789:1700000000:2015000000::u:::cC:::::ed25519:::0:
fpr:::::::::0123456789ABCDEF0123456789ABCDEF01234567:
uid:u::::1700000000::HASH::vergil-project packages <packages@vergil-project.invalid>::::::::::0:
sub:u:255:22:1111222233334444:1700000000:1763000000:::::s:::::ed25519::
fpr:::::::::9999888877776666555544443333222211110000:
"""
_ORG = OrgRepo("vergil", "vergil-project", "https://vergil-project.github.io/packages",
               "0123456789ABCDEF0123456789ABCDEF01234567", "vergil-archive-keyring")

def test_primary_fingerprint_is_the_fpr_after_pub() -> None:
    assert repo_setup.parse_primary_fingerprint(_COLONS) == "0123456789ABCDEF0123456789ABCDEF01234567"

def test_no_pub_is_fatal() -> None:
    with pytest.raises(PackageError, match=r"no primary key"):
        repo_setup.parse_primary_fingerprint("sub:...\n")

def _run(colons: str = _COLONS) -> MagicMock:
    m = MagicMock()
    m.side_effect = lambda *a, **k: subprocess.CompletedProcess(a, 0, colons if "--with-colons" in a else "", "")
    return m

def _flat(run: MagicMock) -> list[str]:
    return [" ".join(c.args) for c in run.call_args_list]

def test_apt_bootstrap_sequence() -> None:
    run = _run()
    repo_setup.bootstrap(run, _ORG, "deb", "noble", sudo=True)
    cmds = _flat(run)
    assert any("curl -fsSL https://vergil-project.github.io/packages/keys/vergil.asc" in c for c in cmds)
    assert any("/etc/apt/sources.list.d/vergil-bootstrap.sources" in c for c in cmds)
    assert any("Suites: noble" in c for c in cmds)
    assert any(c.startswith("sudo apt-get install -y vergil-archive-keyring") for c in cmds)
    assert cmds[-1].startswith("sudo rm -f /etc/apt/sources.list.d/vergil-bootstrap.sources")

def test_dnf_bootstrap_sequence() -> None:
    run = _run()
    repo_setup.bootstrap(run, _ORG, "rpm", "el9", sudo=False)
    cmds = _flat(run)
    assert any("baseurl=https://vergil-project.github.io/packages/rpm/el$releasever/$basearch" in c for c in cmds)
    assert any("repo_gpgcheck=1" in c for c in cmds)
    assert any(c.startswith("dnf install -y vergil-archive-keyring") for c in cmds)

def test_fingerprint_mismatch_is_fatal_before_any_source_is_written() -> None:
    run = _run(_COLONS.replace("0123456789ABCDEF0123456789ABCDEF01234567", "F" * 40))
    with pytest.raises(PackageError, match=r"fingerprint mismatch"):
        repo_setup.bootstrap(run, _ORG, "deb", "noble", sudo=True)
    assert not any("sources.list.d" in c for c in _flat(run))

def test_vergil_org_is_pinned() -> None:
    org = orgs.for_vendor("vergil")
    assert re.fullmatch(r"[0-9A-F]{40}", org.fingerprint)
    assert org.base_url == "https://vergil-project.github.io/packages"

def test_unknown_vendor_is_fatal() -> None:
    with pytest.raises(PackageError, match=r"unknown package vendor 'lmf'"):
        orgs.for_vendor("lmf")
```

- [ ] **Step 2: Run and verify FAIL.**

- [ ] **Step 3: Implement `orgs.py`.** Set the fingerprint to the **exact 40-hex primary fingerprint from OP1's SUCCESS comment**:

```python
# src/vergil_tooling/lib/package/orgs.py
"""Org package repositories and their pinned trust roots (spec §7.4, §9)."""

from __future__ import annotations

from dataclasses import dataclass

from vergil_tooling.lib.package import PackageError


@dataclass(frozen=True)
class OrgRepo:
    vendor: str
    github_org: str
    base_url: str
    fingerprint: str
    keyring_package: str


ORGS: dict[str, OrgRepo] = {
    "vergil": OrgRepo(
        vendor="vergil",
        github_org="vergil-project",
        base_url="https://vergil-project.github.io/packages",
        fingerprint="<40-HEX FROM OP1 SUCCESS COMMENT>",  # replace with the attested value in this step
        keyring_package="vergil-archive-keyring",
    ),
}


def for_vendor(vendor: str) -> OrgRepo:
    if vendor not in ORGS:
        msg = f"unknown package vendor {vendor!r} (known: {', '.join(sorted(ORGS))})"
        raise PackageError(msg)
    return ORGS[vendor]
```

  The `<40-HEX …>` marker must not survive this step; `test_vergil_org_is_pinned` fails until it's replaced. Adding LMF later means one more `ORGS` entry; that's LMF's own task.

- [ ] **Step 4: Implement `repo_setup.py`.**

```python
# src/vergil_tooling/lib/package/repo_setup.py
"""Bootstrap trust in an org package repository (spec §9; reused by install tests and builders)."""

from __future__ import annotations

import subprocess
from collections.abc import Callable

from vergil_tooling.lib.package import PackageError
from vergil_tooling.lib.package.orgs import OrgRepo

Run = Callable[..., subprocess.CompletedProcess[str]]


def local_run(*argv: str) -> subprocess.CompletedProcess[str]:
    return subprocess.run(list(argv), check=True, capture_output=True, text=True)


def parse_primary_fingerprint(colons: str) -> str:
    seen_pub = False
    for line in colons.splitlines():
        fields = line.split(":")
        if fields[0] == "pub":
            seen_pub = True
        elif fields[0] == "fpr" and seen_pub:
            return fields[9]
    msg = "no primary key in the published key file"
    raise PackageError(msg)


def _sh(run: Run, script: str, sudo: bool) -> None:
    run(*(["sudo"] if sudo else []), "bash", "-c", script)


def bootstrap(run: Run, org: OrgRepo, fmt: str, suite: str, *, sudo: bool) -> None:
    s = ["sudo"] if sudo else []
    key_tmp = f"/tmp/{org.vendor}-bootstrap.asc"
    if fmt == "deb":
        run(*s, "apt-get", "update")
        run(*s, "apt-get", "install", "-y", "ca-certificates", "curl", "gnupg")
    else:
        run(*s, "dnf", "install", "-y", "ca-certificates", "curl", "gnupg2")
    run("curl", "-fsSL", f"{org.base_url}/keys/{org.vendor}.asc", "-o", key_tmp)
    got = parse_primary_fingerprint(run("gpg", "--show-keys", "--with-colons", key_tmp).stdout)
    if got != org.fingerprint:
        msg = f"{org.vendor} package key fingerprint mismatch: published {got}, pinned {org.fingerprint}"
        raise PackageError(msg)
    if fmt == "deb":
        src = f"/etc/apt/sources.list.d/{org.vendor}-bootstrap.sources"
        key = f"/etc/apt/keyrings/{org.vendor}-bootstrap.asc"
        _sh(run, f"install -D -m 0644 {key_tmp} {key} && cat > {src} <<'EOF'\n"
                 f"Types: deb\nURIs: {org.base_url}/deb\nSuites: {suite}\nComponents: main\nSigned-By: {key}\nEOF", sudo)
        run(*s, "apt-get", "update")
        run(*s, "apt-get", "install", "-y", org.keyring_package)
        run(*s, "rm", "-f", src, key)
    else:
        src = f"/etc/yum.repos.d/{org.vendor}-bootstrap.repo"
        key = f"/etc/pki/rpm-gpg/RPM-GPG-KEY-{org.vendor}-bootstrap"
        _sh(run, f"install -D -m 0644 {key_tmp} {key} && cat > {src} <<'EOF'\n"
                 f"[{org.vendor}-bootstrap]\nname={org.vendor} packages (bootstrap)\n"
                 f"baseurl={org.base_url}/rpm/el$releasever/$basearch\n"
                 f"gpgcheck=1\nrepo_gpgcheck=1\ngpgkey=file://{key}\nenabled=1\nEOF", sudo)
        run(*s, "dnf", "install", "-y", org.keyring_package)
        run(*s, "rm", "-f", src, key)
```

  The `rm -f` is the final call on both paths, which the apt test asserts. Run again and verify PASS.

- [ ] **Step 5: `vrg-container-run -- vrg-validate` green; commit.**
  `vrg-commit --type feat --scope package --message "org repository registry and fingerprint-pinned trust bootstrap (#<T3>)"`

---

## Task T4: the `python` builder

**Repo:** vergil-tooling. Blocked by: T2, T3.

**Files:**

- Create: `src/vergil_tooling/lib/package/python_builder.py`
- Modify: `src/vergil_tooling/bin/vrg_package.py`. Import `python_builder` so it registers.
- Test: `tests/vergil_tooling/test_package_python_builder.py`

**Interfaces:**

- Consumes: `build.register`, `build.BuildContext`/`BuildResult`/`contents_from_tree`, `repo_setup.bootstrap`/`local_run`, `orgs.for_vendor`.
- Produces:
  - `python_builder.build_python(ctx) -> BuildResult` (registered as `"python"`).
  - `python_builder.runtime_dir(vendor, runtime) -> str`.
  - `python_builder.shim_commands(repo_root, pkg) -> list[str]`.
  - `python_builder.runtime_depends(runtime, pbs_version) -> dict[str, list[str]]`.

- [ ] **Step 1: Failing tests.**

```python
# tests/vergil_tooling/test_package_python_builder.py
def test_runtime_depends_formats() -> None:
    assert python_builder.runtime_depends("3.14.4", "3.14.4+20261001") == {
        "deb": ["vergil-python3.14.4 (>= 3.14.4+20261001)"],
        "rpm": ["vergil-python3.14.4 >= 3.14.4+20261001"],
    }

def test_shims_default_to_all_project_scripts(tmp_path: Path) -> None:
    (tmp_path / "pyproject.toml").write_text('[project]\nname="t"\n[project.scripts]\nvrg-a="m:a"\nvrg-b="m:b"\n')
    assert python_builder.shim_commands(tmp_path, _pkg()) == ["vrg-a", "vrg-b"]

def test_shims_subset_must_exist(tmp_path: Path) -> None:
    (tmp_path / "pyproject.toml").write_text('[project]\nname="t"\n[project.scripts]\nvrg-a="m:a"\n')
    with pytest.raises(PackageError, match=r"\[package\.python\]\.commands: 'vrg-z' is not in \[project\.scripts\]"):
        python_builder.shim_commands(tmp_path, _pkg(python=PackagePythonConfig(runtime="3.14.4", commands=["vrg-z"])))

def test_missing_lock_is_fatal(tmp_path: Path) -> None:
    (tmp_path / "pyproject.toml").write_text('[project]\nname="t"\n')
    with pytest.raises(PackageError, match=r"uv\.lock is required"):
        python_builder.build_python(_py_ctx(tmp_path))

def test_build_sequence(tmp_path: Path, monkeypatch: pytest.MonkeyPatch) -> None:
    (tmp_path / "pyproject.toml").write_text('[project]\nname="vergil-tooling"\n[project.scripts]\nvrg-a="m:a"\n')
    (tmp_path / "uv.lock").write_text("version = 1\n")
    calls: list[tuple[str, ...]] = []
    monkeypatch.setattr(python_builder.repo_setup, "bootstrap", lambda *a, **k: calls.append(("bootstrap",)))
    def fake_run(*argv: str, **kw: object) -> subprocess.CompletedProcess[str]:
        calls.append(argv)
        out = "3.14.4+20261001-1" if argv[0] in ("dpkg-query", "rpm") else ""
        return subprocess.CompletedProcess(argv, 0, out, "")
    monkeypatch.setattr(python_builder, "_run", fake_run)
    venv_root = tmp_path / "fake-opt"
    monkeypatch.setattr(python_builder, "_OPT", str(venv_root))
    (venv_root / "vergil/vergil-tooling/venv/bin").mkdir(parents=True)
    (venv_root / "vergil/vergil-tooling/venv/bin/vrg-a").write_text("#!x\n")
    res = python_builder.build_python(_py_ctx(tmp_path))
    flat = [" ".join(c) for c in calls]
    assert flat[0] == "bootstrap"
    assert any(c.startswith("apt-get install -y vergil-python3.14.4") for c in flat)
    assert any(c.startswith("uv venv --python /opt/vergil/python/3.14.4/bin/python3.14") for c in flat)
    assert any("uv sync --frozen --no-dev --no-editable --compile-bytecode" in c for c in flat)
    assert res.depends["deb"] == ["vergil-python3.14.4 (>= 3.14.4+20261001)"]
    assert {"src": "/opt/vergil/vergil-tooling/venv/bin/vrg-a", "dst": "/usr/bin/vrg-a", "type": "symlink"} in res.contents
    assert (_py_ctx(tmp_path).staging_root / "opt/vergil/vergil-tooling/venv/bin/vrg-a").exists()
```

  `_py_ctx` creates a `BuildContext` with the python `_pkg()`, `cell=shared-amd64` (targets including `ubuntu/24.04/amd64`), `name="vergil-tooling"`, and `staging_root=tmp_path/"s"` (created). Note that the shared cell builds in `ubuntu:24.04`, so the builder bootstraps with `fmt="deb", suite="noble"`. A `native` RHEL cell uses `fmt="rpm"` with its suite. Add `test_native_rhel_cell_uses_dnf` to cover that.

- [ ] **Step 2: Run and verify FAIL.**

- [ ] **Step 3: Implement.**

```python
# src/vergil_tooling/lib/package/python_builder.py
"""The `python` builder (spec §6.1)."""

from __future__ import annotations

import os
import shutil
import subprocess
import tomllib
from pathlib import Path
from typing import TYPE_CHECKING

from vergil_tooling.lib.package import PackageError, orgs, repo_setup
from vergil_tooling.lib.package import targets as tg
from vergil_tooling.lib.package.build import BuildContext, BuildResult, contents_from_tree, register

if TYPE_CHECKING:
    from vergil_tooling.lib.config import PackageConfig

_OPT = "/opt"  # monkeypatched in tests; the real build writes the venv at its final path


def _run(*argv: str, env: dict[str, str] | None = None, cwd: Path | None = None) -> subprocess.CompletedProcess[str]:
    return subprocess.run(list(argv), check=True, capture_output=True, text=True, env=env, cwd=cwd)


def runtime_dir(vendor: str, runtime: str) -> str:
    return f"/opt/{vendor}/python/{runtime}"


def runtime_depends(runtime: str, pbs_version: str) -> dict[str, list[str]]:
    name = f"vergil-python{runtime}"
    return {"deb": [f"{name} (>= {pbs_version})"], "rpm": [f"{name} >= {pbs_version}"]}


def shim_commands(repo_root: Path, pkg: PackageConfig) -> list[str]:
    scripts = tomllib.loads((repo_root / "pyproject.toml").read_text()).get("project", {}).get("scripts", {})
    wanted = pkg.python.commands if pkg.python and pkg.python.commands is not None else sorted(scripts)
    for cmd in wanted:
        if cmd not in scripts:
            msg = f"[package.python].commands: {cmd!r} is not in [project.scripts]"
            raise PackageError(msg)
    return list(wanted)


@register("python")
def build_python(ctx: BuildContext) -> BuildResult:
    assert ctx.pkg.python is not None  # guaranteed by config validation
    if not (ctx.repo_root / "uv.lock").is_file():
        msg = "uv.lock is required for builder = \"python\" (build from the lock only)"
        raise PackageError(msg)
    runtime = ctx.pkg.python.runtime
    first = tg.REGISTRY[ctx.cell.targets[0]]
    fmt, suite = (first.fmt, first.suite) if ctx.cell.native else ("deb", "noble")
    repo_setup.bootstrap(repo_setup.local_run, orgs.for_vendor("vergil"), fmt, suite, sudo=False)
    rt_pkg = f"vergil-python{runtime}"
    if fmt == "deb":
        _run("apt-get", "install", "-y", rt_pkg)
        installed = _run("dpkg-query", "-W", "-f=${Version}", rt_pkg).stdout.strip()
    else:
        _run("dnf", "install", "-y", rt_pkg)
        installed = _run("rpm", "-q", "--qf", "%{VERSION}-%{RELEASE}", rt_pkg).stdout.strip()
    pbs_version = installed.rsplit("-", 1)[0]
    interpreter = f"{runtime_dir('vergil', runtime)}/bin/python{runtime.rsplit('.', 1)[0]}"
    product = f"{_OPT}/{ctx.pkg.vendor}/{ctx.name}"
    venv = f"{product}/venv"
    _run("uv", "venv", "--python", interpreter, venv)
    _run("uv", "sync", "--frozen", "--no-dev", "--no-editable", "--compile-bytecode",
         env={**os.environ, "UV_PROJECT_ENVIRONMENT": venv, "UV_PYTHON": interpreter}, cwd=ctx.repo_root)
    dest = ctx.staging_root / "opt" / ctx.pkg.vendor / ctx.name
    shutil.copytree(product, dest, symlinks=True)
    contents = contents_from_tree(ctx.staging_root, own_below=f"/opt/{ctx.pkg.vendor}")
    final_venv = f"/opt/{ctx.pkg.vendor}/{ctx.name}/venv"
    contents += [{"src": f"{final_venv}/bin/{c}", "dst": f"/usr/bin/{c}", "type": "symlink"}
                 for c in shim_commands(ctx.repo_root, ctx.pkg)]
    return BuildResult(contents=contents, depends=runtime_depends(runtime, pbs_version))
```

Notes for the implementer:

- In tests `_OPT` points at a temp dir and the asserted strings use the real `/opt` paths for the interpreter.
- In CI the build runs as root in the build container, so `/opt` is writable.
- The runtime package always comes from the **vergil** org (D7), whatever the product's vendor.

- [ ] **Step 4: Run and verify PASS; validate; commit.**
  `vrg-commit --type feat --scope package --message "python builder: venv from lock on the pinned runtime (#<T4>)"`

---

## Task T5: `vrg-package install-test`

**Repo:** vergil-tooling. Blocked by: T2, T3.

**Files:**

- Create: `src/vergil_tooling/lib/package/install_test.py`
- Modify: `src/vergil_tooling/bin/vrg_package.py`. Add the `install-test` subcommand.
- Test: `tests/vergil_tooling/test_package_install_test.py`

**Interfaces:**

- Consumes: `matrix.resolve`/`TestCell`, `naming.package_name`, `repo_setup.bootstrap`/`local_run`, `orgs.for_vendor`.
- Produces:
  - `install_test.select_artifact(artifacts: Path, name: str, cell: TestCell, noarch: bool) -> Path`.
  - `install_test.run_install_test(repo_root: Path, cell_id: str, artifacts: Path, report: Path, run: Run = local_run) -> None`. The report JSON is `{"cell", "target", "artifact", "units": [...], "smoke": "pass", "residue": []}`.
  - CLI: `vrg-package install-test --cell ID --artifacts DIR --report PATH`.

- [ ] **Step 1: Failing tests, artifact selection.**

```python
def test_selects_shared_deb_for_its_arch(tmp_path: Path) -> None:
    for f in ("t_2.1.240-1_amd64.deb", "t_2.1.240-1_arm64.deb", "t-2.1.240-1.x86_64.rpm"):
        (tmp_path / f).write_bytes(b"x")
    cell = TestCell("test-ubuntu-24.04-amd64", "ubuntu/24.04/amd64", "amd64", "r", "ubuntu:24.04", "deb", "noble", False)
    assert install_test.select_artifact(tmp_path, "t", cell, noarch=False).name == "t_2.1.240-1_amd64.deb"

def test_selects_native_suffix(tmp_path: Path) -> None:
    for f in ("t_2.1.240-1_amd64.deb", "t_2.1.240-1~resolute_amd64.deb"):
        (tmp_path / f).write_bytes(b"x")
    cell = TestCell("x", "ubuntu/26.04/amd64", "amd64", "r", "ubuntu:26.04", "deb", "resolute", True)
    assert install_test.select_artifact(tmp_path, "t", cell, noarch=False).name == "t_2.1.240-1~resolute_amd64.deb"

def test_rpm_noarch(tmp_path: Path) -> None:
    (tmp_path / "k-1.0.0-1.noarch.rpm").write_bytes(b"x")
    cell = TestCell("x", "rhel/9/arm64", "arm64", "r", "ubi9", "rpm", "el9", False)
    assert install_test.select_artifact(tmp_path, "k", cell, noarch=True).name == "k-1.0.0-1.noarch.rpm"

def test_zero_or_many_matches_is_fatal(tmp_path: Path) -> None:
    cell = TestCell("x", "rhel/9/amd64", "amd64", "r", "ubi9", "rpm", "el9", False)
    with pytest.raises(PackageError, match=r"expected exactly one artifact for t on rhel/9/amd64, found 0"):
        install_test.select_artifact(tmp_path, "t", cell, noarch=False)
```

  The selection globs are: deb shared `{name}_*-1_{arch}.deb`; deb native `{name}_*-1~{suite}_{arch}.deb`; rpm shared `{name}-*-1.{rpm_arch}.rpm`; rpm native `{name}-*-1.{suite}.{rpm_arch}.rpm`. noarch uses `all` for deb and `noarch` for rpm.

- [ ] **Step 2: Failing tests, the install-test sequence.**

```python
def test_sequence_python_product_deb(tmp_path: Path, monkeypatch: pytest.MonkeyPatch) -> None:
    repo = _repo_with_python_package(tmp_path)  # vergil.toml [package] builder=python, smoke="vrg-whoami --mode"
    arts = tmp_path / "arts"; arts.mkdir()
    (arts / "vergil-tooling_2.1.240-1_amd64.deb").write_bytes(b"x")
    boot: list[tuple[str, str]] = []
    monkeypatch.setattr(install_test.repo_setup, "bootstrap", lambda run, org, fmt, suite, sudo: boot.append((fmt, suite)))
    calls: list[str] = []
    def run(*argv: str) -> subprocess.CompletedProcess[str]:
        calls.append(" ".join(argv))
        out = "/usr/bin/vrg-whoami\n/opt/vergil/vergil-tooling/venv/bin/python\n" if argv[:2] == ("dpkg", "-L") else ""
        if argv[0] == "test" and argv[1] == "-e":
            return subprocess.CompletedProcess(argv, 1, "", "")  # residue check: nothing left
        return subprocess.CompletedProcess(argv, 0, out, "")
    rep = tmp_path / "r.json"
    install_test.run_install_test(repo, "test-ubuntu-24.04-amd64", arts, rep, run=run)
    assert boot == [("deb", "noble")]
    assert any(c.startswith("apt-get install -y ") and c.endswith("vergil-tooling_2.1.240-1_amd64.deb") for c in calls)
    assert f"{install_test.CLEAN_ENV} bash -c vrg-whoami --mode" in calls
    assert f"{install_test.CLEAN_ENV} bash -c command -v vrg-whoami" in calls
    assert any(c == "apt-get purge -y vergil-tooling" for c in calls)
    assert json.loads(rep.read_text())["residue"] == []

def test_residue_is_fatal(tmp_path: Path, monkeypatch: pytest.MonkeyPatch) -> None:
    # same setup as above, but `test -e /opt/vergil/vergil-tooling` returns 0
    with pytest.raises(PackageError, match=r"left behind after removal: /opt/vergil/vergil-tooling"):
        ...

def test_units_are_verified(tmp_path: Path, monkeypatch: pytest.MonkeyPatch) -> None:
    # dpkg -L lists /usr/lib/systemd/system/x.service → expect "apt-get install -y systemd" then
    # "systemd-analyze verify /usr/lib/systemd/system/x.service" in calls, and report["units"] == [that path]
    ...

def test_staged_product_does_not_bootstrap_repo(tmp_path: Path, monkeypatch: pytest.MonkeyPatch) -> None:
    # builder = "staged" → repo_setup.bootstrap never called; installs the local file only
    ...

def test_smoke_runs_with_sanitized_path(tmp_path: Path, monkeypatch: pytest.MonkeyPatch) -> None:
    # The uv copy of vergil-tooling that drives the test lives in ~/.local/bin; the smoke must
    # never resolve it. Assert the exact CLEAN_ENV prefix, that it contains no ".local", and that
    # a shim resolving anywhere but /usr/bin/<cmd> (fake `command -v` stdout "/root/.local/bin/vrg-whoami")
    # raises PackageError "shim vrg-whoami resolves to /root/.local/bin/vrg-whoami, expected /usr/bin/vrg-whoami".
    ...
```

  `install_test.CLEAN_ENV` is the module constant:
  `"env -i HOME=/root PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"`.
  It is split with `shlex.split` when it's passed to `run`. The smoke and the shim checks always run under it (spec §8.1).

  Write the three elided bodies out in full in the same style as the first test; each asserts what its comment states. The `run` fake returns exit 1 for `test -e` unless that path is listed as residue. The sequence is fixed, as follows.

  1. Prerequisites:
     - deb: `apt-get update`
     - rpm: no-op
  2. Repo bootstrap for `builder == "python"` only:
     - `repo_setup.bootstrap(run, orgs.for_vendor("vergil"), cell.fmt, cell.suite, sudo=False)`
  3. Install:
     - deb: `apt-get install -y <abs path>`
     - rpm: `dnf install -y <abs path>`
  4. List files:
     - deb: `dpkg -L <name>`
     - rpm: `rpm -ql <name>`
  5. Units: if any listed path ends in `.service`, `.timer` or `.socket` under a `systemd/system/` directory:
     - deb: `apt-get install -y systemd`
     - rpm: `dnf install -y systemd`
     - then `systemd-analyze verify <unit>` for each unit
  6. Smoke and shims, all under `CLEAN_ENV`:
     - run `bash -c <smoke>`;
     - for each listed `/usr/bin/<cmd>`, run `bash -c "command -v <cmd>"` and require the output to be exactly `/usr/bin/<cmd>`.
  7. Remove:
     - deb: `apt-get purge -y <name>`
     - rpm: `dnf remove -y <name>`
  8. Residue check: for `/opt/<vendor>/<name>`, and for every listed path under `/usr/bin/`, run `test -e <path>`. Exit 0 means residue. Any residue is fatal, listing every path.
  9. Write the report.

- [ ] **Step 3: Run, verify FAIL, implement `install_test.py` to that sequence, run, verify PASS.** `run(*argv)` raising `CalledProcessError` propagates. Wrap the smoke call so a failure raises `PackageError(f"smoke command failed on {cell.target}: {smoke}")`. A non-zero exit from `test -e` is *expected* and means "absent", so call it through a helper that catches `CalledProcessError` → `False`. (`local_run` uses `check=True`, which is why the helper exists.)

- [ ] **Step 4: Add the `install-test` subparser and a CLI wiring test (pattern of T2 Step 11).** Validate and commit.
  `vrg-commit --type feat --scope package --message "vrg-package install-test: install, units, smoke, clean removal (#<T5>)"`

---

## Task T6: index part 1: collect, verify, retain

**Repo:** vergil-tooling. Blocked by: T1.

**Files:**

- Create: `src/vergil_tooling/lib/package/index/__init__.py`
- Create: `src/vergil_tooling/lib/package/index/config.py` (`packages.toml`)
- Create: `src/vergil_tooling/lib/package/index/collect.py`
- Create: `src/vergil_tooling/lib/package/index/retention.py`
- Test: `tests/vergil_tooling/test_package_index_config.py`, `test_package_index_collect.py`, `test_package_index_retention.py`

**Interfaces:**

- Produces:
  - `index.config.IndexConfig(vendor: str, products: list[str], keep: int, lines: int)` and `index.config.load(path: Path) -> IndexConfig`.
  - `index.collect.Artifact(product, tag, path: Path, fmt, name, version, release, arch, depends: tuple[str, ...], sha256)` with `full_version -> f"{version}-{release}"`.
  - `index.collect.collect(cfg, workdir: Path, gh: Run = local_run) -> list[Artifact]`. It lists stable releases, skips releases without a manifest, downloads, checks manifest completeness, and reads metadata.
  - `index.collect.verify(artifacts, keyfile: Path, gh: Run = local_run) -> None`. Attestation for all, plus the rpm signature.
  - `index.retention.select(artifacts, keep, lines) -> list[Artifact]`. Per-line retention plus the dependency closure, and the duplicate check.

- [ ] **Step 1: Failing tests, `packages.toml`.**

```python
def test_load_defaults(tmp_path: Path) -> None:
    p = tmp_path / "packages.toml"
    p.write_text('vendor = "vergil"\nproducts = ["vergil-project/vergil-tooling"]\n')
    assert index_config.load(p) == IndexConfig("vergil", ["vergil-project/vergil-tooling"], keep=3, lines=2)

@pytest.mark.parametrize(("body", "match"), [
    ('products = ["a/b"]\n', r"packages\.toml: vendor is required"),
    ('vendor = "v"\nproducts = []\n', r"products must be a non-empty list"),
    ('vendor = "v"\nproducts = ["a/b"]\n[retention]\nkeep = 0\n', r"retention\.keep must be an integer >= 1"),
])
def test_load_errors(tmp_path: Path, body: str, match: str) -> None:
    p = tmp_path / "packages.toml"; p.write_text(body)
    with pytest.raises(PackageError, match=match):
        index_config.load(p)
```

- [ ] **Step 2: Run, verify FAIL, implement `index/config.py`** (tomllib plus the checks above), then verify PASS.

- [ ] **Step 3: Failing tests, retention.**

```python
def _a(product: str, tag: str, name: str, version: str, arch: str = "amd64", fmt: str = "deb",
       depends: tuple[str, ...] = (), sha: str = "s") -> Artifact:
    return Artifact(product, tag, Path(f"/x/{name}_{version}_{arch}.{fmt}"), fmt, name, version, "1", arch, depends, sha + version)

def test_keep_per_line_newest_two_lines() -> None:
    tags = ["v2.0.1", "v2.0.2", "v2.1.1", "v2.1.2", "v2.1.3", "v2.1.4", "v2.2.0"]
    arts = [_a("o/t", t, "t", t[1:]) for t in tags]
    kept = {a.tag for a in retention.select(arts, keep=3, lines=2)}
    assert kept == {"v2.1.2", "v2.1.3", "v2.1.4", "v2.2.0"}

def test_dependency_closure_keeps_old_runtime() -> None:
    rt_old = _a("o/py", "v1.0.0", "vergil-python3.14.3", "3.14.3+20260801")
    rt_new = _a("o/py", "v1.1.0", "vergil-python3.14.4", "3.14.4+20261001")
    rt_newer = _a("o/py", "v1.2.0", "vergil-python3.14.5", "3.14.5+20261201")
    rt_newest = _a("o/py", "v1.3.0", "vergil-python3.14.6", "3.14.6+20270101")
    app = _a("o/t", "v2.1.4", "t", "2.1.4", depends=("vergil-python3.14.3",))
    kept = retention.select([rt_old, rt_new, rt_newer, rt_newest, app], keep=1, lines=2)
    assert rt_old in kept  # evicted by line retention (lines=2 keeps 1.3, 1.2) but kept by closure

def test_duplicate_name_version_arch_with_different_bytes_is_fatal() -> None:  # Review Focus 1
    a = _a("o/py", "v1.0.0", "vergil-python3.14.4", "3.14.4+20261001", sha="aaa")
    b = _a("o/py", "v1.0.1", "vergil-python3.14.4", "3.14.4+20261001", sha="bbb")
    with pytest.raises(PackageError, match=r"vergil-python3\.14\.4 3\.14\.4\+20261001-1 amd64 \(deb\) differs between o/py@v1\.0\.0 and o/py@v1\.0\.1"):
        retention.select([a, b], keep=3, lines=2)

def test_identical_duplicate_is_collapsed() -> None:
    a = _a("o/py", "v1.0.0", "p", "1", sha="same")
    b = _a("o/py", "v1.0.1", "p", "1", sha="same")
    assert len([x for x in retention.select([a, b], keep=3, lines=2) if x.name == "p"]) == 1
```

  Duplicate identity is `(fmt, name, full_version, arch)`. Equal `sha256` collapses to the first; a different `sha256` is fatal. The closure rule: for every dependency name of a retained artifact that some collected artifact provides, retain the **newest** `full_version` of that name **per (fmt, arch)**, comparing versions with `packaging.version`-free logic. Use `tuple(int(x) if x.isdigit() else x for x in re.split(r"[.+~-]", v))`; numeric parts only matter here. Dependency names not provided by any collected artifact are ignored (OS packages, or another org's packages).

- [ ] **Step 4: Run, verify FAIL, implement `retention.py`, verify PASS.**

- [ ] **Step 5: Failing tests, collect.**

```python
def test_release_without_manifest_is_ignored(tmp_path: Path, capsys: pytest.CaptureFixture[str]) -> None:  # Review Focus 2
    gh = _fake_gh(releases={"o/t": [("v2.0.0", [])]})  # no assets at all
    assert collect.collect(IndexConfig("vergil", ["o/t"], 3, 2), tmp_path, gh=gh) == []
    assert "o/t@v2.0.0: no packages-manifest.json (pre-packaging release) — skipped" in capsys.readouterr().out

def test_partial_release_is_fatal(tmp_path: Path) -> None:  # Review Focus 2
    manifest = {"artifacts": [{"fmt": "deb", "arch": "amd64", "suite": None}, {"fmt": "deb", "arch": "arm64", "suite": None}]}
    gh = _fake_gh(releases={"o/t": [("v2.1.0", ["packages-manifest.json", "t_2.1.0-1_amd64.deb"])]}, manifest=manifest)
    with pytest.raises(PackageError, match=r"o/t@v2\.1\.0 is missing deb/arm64"):
        collect.collect(IndexConfig("vergil", ["o/t"], 3, 2), tmp_path, gh=gh)

def test_only_stable_tags_and_metadata(tmp_path: Path) -> None:
    # releases: v2.1.0 (complete), develop-v2.1.0 (ignored), v2.2.0-rc1 (ignored)
    # metadata via fake `dpkg-deb -f` / `rpm -qp` outputs → Artifact fields, sha256 of the file bytes
    ...

def test_verify_pins_signer_workflow_and_ref(tmp_path: Path) -> None:
    calls: list[str] = []
    gh = lambda *a: (calls.append(" ".join(a)), subprocess.CompletedProcess(a, 0, "digests signatures OK", ""))[1]
    art = _a("o/t", "v2.1.0", "t", "2.1.0", fmt="rpm")
    collect.verify([art], tmp_path / "k.asc", gh=gh)
    assert any(c.startswith("gh attestation verify ") and
               "--repo o/t --signer-workflow vergil-project/vergil-actions/.github/workflows/cd-release.yml "
               "--source-ref refs/heads/main" in c for c in calls)
    assert any(c.startswith("rpmkeys --checksig ") for c in calls)

def test_verify_failure_is_fatal(tmp_path: Path) -> None:
    def gh(*a: str) -> subprocess.CompletedProcess[str]:
        raise subprocess.CalledProcessError(1, a, "", "no matching attestations")
    with pytest.raises(PackageError, match=r"attestation verification failed for .*: no matching attestations"):
        collect.verify([_a("o/t", "v2.1.0", "t", "2.1.0")], tmp_path / "k.asc", gh=gh)

def test_unsigned_rpm_is_fatal(tmp_path: Path) -> None:
    # gh fake: attestation ok; `rpmkeys --checksig` stdout "digests SIGNATURES NOT OK" → PackageError "rpm signature check failed"
    ...
```

Write the elided bodies in full. The `_fake_gh` helper is a `Run` fake:

- It answers `gh release list --repo R --limit 200 --json tagName,isDraft,isPrerelease` with JSON.
- It answers `gh release view TAG --repo R --json assets` with `{"assets": [{"name": ...}]}`.
- On `gh release download TAG --repo R --pattern NAME --dir D` it writes fixture files into `D`. The manifest is `json.dumps(manifest)`; package files get placeholder bytes.
- It answers `dpkg-deb -f FILE Package Version Architecture Depends` and `rpm -qp --qf ... FILE` / `rpm -qp --requires FILE` with deterministic metadata derived from the filename.

- [ ] **Step 6: Run, verify FAIL, implement `collect.py`, verify PASS.** These are the implementation rules:
  - **Stable tags only:** `re.fullmatch(r"v\d+\.\d+\.\d+", tag)`, skipping drafts and prereleases.
  - **Completeness:** each manifest `(fmt, arch, suite)` must match exactly one downloaded file. The arch is `all`/`noarch` when the manifest says `all`, and the suite suffix rule is as in T5.
  - **Deb metadata:** `dpkg-deb -f F Package Version Architecture Depends`. `Version` is `X-R`, which splits at the last `-` into `version`/`release`.
  - **Rpm metadata:**
    - `rpm -qp --qf '%{NAME}\n%{VERSION}\n%{RELEASE}\n%{ARCH}\n' F`
    - `rpm -qp --requires F`, keeping only names: the first token per line, dropping `rpmlib(` and `/`-prefixed entries.
    - Map `x86_64→amd64`, `aarch64→arm64`, `noarch→all` so retention compares arches uniformly.
  - **Verification:** `gh attestation verify F --repo <product> --signer-workflow <verbatim> --source-ref refs/heads/main`. For rpm, run `rpmkeys --import <keyfile>` once, then `rpmkeys --checksig F` and require `signatures OK` in stdout.

- [ ] **Step 7: Validate; commit.**
  `vrg-commit --type feat --scope package --message "index: collect stable releases, verify provenance, per-line retention (#<T6>)"`

---

## Task T7: index part 2: apt/dnf metadata, signing, size guard, `vrg-package index`

**Repo:** vergil-tooling. Blocked by: T6.

**Files:**

- Create: `src/vergil_tooling/lib/package/index/apt.py`
- Create: `src/vergil_tooling/lib/package/index/rpm.py`
- Create: `src/vergil_tooling/lib/package/index/sign.py`
- Create: `src/vergil_tooling/lib/package/index/site.py`
- Modify: `src/vergil_tooling/bin/vrg_package.py`. Add the `index` subcommand.
- Test: `tests/vergil_tooling/test_package_index_apt.py`, `test_package_index_rpm.py`, `test_package_index_sign.py`, `test_package_index_site.py`

**Interfaces:**

- Consumes: `collect.Artifact`, `collect.collect`/`verify`, `retention.select`, `index_config.load`, `targets.REGISTRY`.
- Produces:
  - `apt.write(site: Path, artifacts: list[Artifact], suites: list[str], control: Callable[[Path], str]) -> list[Path]`, which returns each suite's `Release` path.
  - `rpm.write(site: Path, artifacts: list[Artifact], base_url: str, els: list[str], run: Run) -> list[Path]`, which returns each `repomd.xml` path.
  - `sign.import_key(run: Run, key_env: str, pass_env: str) -> None`, `sign.clearsign(run, src, dst)`, `sign.detach(run, src, dst)`.
  - `site.size_guard(site: Path, warn: int = 750_000_000, fail: int = 900_000_000) -> int`.
  - `site.build_site(cfg, keys_dir: Path, out: Path, workdir: Path, run: Run) -> None` (end to end).
  - CLI: `vrg-package index --config packages.toml --keys keys --out _site [--work .vergil/index-work]`.

- [ ] **Step 1: Failing tests, apt.**

```python
def test_packages_stanza_has_hashes_and_filename(tmp_path: Path) -> None:
    deb = tmp_path / "in" / "t_2.1.0-1_amd64.deb"; deb.parent.mkdir(); deb.write_bytes(b"abc")
    art = Artifact("o/t", "v2.1.0", deb, "deb", "t", "2.1.0", "1", "amd64", (), hashlib.sha256(b"abc").hexdigest())
    control = lambda p: "Package: t\nVersion: 2.1.0-1\nArchitecture: amd64\n"
    releases = apt.write(tmp_path / "site", [art], ["noble", "resolute"], control)
    pk = (tmp_path / "site/deb/dists/noble/main/binary-amd64/Packages").read_text()
    assert "Filename: pool/t/t_2.1.0-1_amd64.deb\n" in pk and "Size: 3\n" in pk
    assert f"SHA256: {hashlib.sha256(b'abc').hexdigest()}\n" in pk
    assert (tmp_path / "site/deb/pool/t/t_2.1.0-1_amd64.deb").read_bytes() == b"abc"
    assert (tmp_path / "site/deb/dists/resolute/main/binary-amd64/Packages").read_text() == pk
    rel = releases[0].read_text()
    assert "Suite: noble\n" in rel and "Architectures: amd64 arm64\n" in rel and "main/binary-amd64/Packages.gz" in rel

def test_native_suffixed_deb_only_in_its_suite(tmp_path: Path) -> None:
    # artifact release "1~resolute" → listed in resolute's Packages only; shared "1" → both
    ...

def test_all_arch_listed_in_every_binary_dir(tmp_path: Path) -> None:
    # Architecture: all → appears in binary-amd64 and binary-arm64
    ...
```

  The `Packages` stanza is the full control text (from `dpkg-deb -f F` with no field list, i.e. all fields) followed by `Filename:`, `Size:`, `MD5sum:`, `SHA1:` and `SHA256:`. Stanzas are separated by blank lines, sorted by `(Package, Version)`. Write `Packages` and `Packages.gz` (deterministic: `gzip.GzipFile(mtime=0)`). The `Release` fields are `Origin`/`Label` (vendor), `Suite`, `Codename`, `Architectures: amd64 arm64`, `Components: main`, `Date` (RFC 2822 UTC), and the `MD5Sum`/`SHA256` lists of each `main/binary-*/Packages{,.gz}` with sizes. Pool path: `deb/pool/<name>/<file>`.

- [ ] **Step 2: Run, verify FAIL, implement `apt.py`, verify PASS.**

- [ ] **Step 3: Failing tests, rpm.**

```python
def test_rpm_repodata_per_el_and_arch_with_pool_baseurl(tmp_path: Path) -> None:
    rpm_file = tmp_path / "in" / "t-2.1.0-1.x86_64.rpm"; rpm_file.parent.mkdir(); rpm_file.write_bytes(b"r")
    art = Artifact("o/t", "v2.1.0", rpm_file, "rpm", "t", "2.1.0", "1", "amd64", (), "s")
    calls: list[list[str]] = []
    def run(*argv: str) -> subprocess.CompletedProcess[str]:
        calls.append(list(argv))
        outdir = Path(argv[argv.index("--outputdir") + 1])
        (outdir / "repodata").mkdir(parents=True, exist_ok=True)
        (outdir / "repodata" / "repomd.xml").write_text("<repomd/>")
        return subprocess.CompletedProcess(argv, 0, "", "")
    out = rpm.write(tmp_path / "site", [art], "https://v.github.io/packages", ["el9", "el10"], run)
    assert (tmp_path / "site/rpm/pool/t-2.1.0-1.x86_64.rpm").exists()
    assert {p.relative_to(tmp_path / "site").as_posix() for p in out} == {
        "rpm/el9/x86_64/repodata/repomd.xml", "rpm/el9/aarch64/repodata/repomd.xml",
        "rpm/el10/x86_64/repodata/repomd.xml", "rpm/el10/aarch64/repodata/repomd.xml"}
    assert all("--baseurl" in c and "https://v.github.io/packages/rpm/pool/" in c for c in calls)
```

  Implementation: for each `(el, rpm_arch)`, make a temp dir containing **symlinks** to the selected pool files. Selection: matching arch (or noarch); shared release `1`, or native `1.<el>` for that el only. Then run `createrepo_c --baseurl <base>/rpm/pool/ --outputdir <tmp> <tmp>` and copy `<tmp>/repodata` to `site/rpm/<el>/<arch>/repodata`. Files are flat in `rpm/pool/`, so `baseurl + href` resolves to `rpm/pool/<file>` (the §7.3 verification item). An empty `(el, arch)` still gets valid empty repodata.

- [ ] **Step 4: Run, verify FAIL, implement `rpm.py`, verify PASS.**

- [ ] **Step 5: Failing tests, signing and size guard.**

```python
def test_import_key_requires_env(monkeypatch: pytest.MonkeyPatch) -> None:
    monkeypatch.delenv("PACKAGE_SIGNING_KEY", raising=False)
    with pytest.raises(PackageError, match=r"PACKAGE_SIGNING_KEY is not set"):
        sign.import_key(MagicMock(), "PACKAGE_SIGNING_KEY", "PACKAGE_SIGNING_PASSPHRASE")

def test_clearsign_and_detach_use_loopback(tmp_path: Path, monkeypatch: pytest.MonkeyPatch) -> None:
    monkeypatch.setenv("PACKAGE_SIGNING_PASSPHRASE", "pw")
    run = MagicMock(return_value=subprocess.CompletedProcess([], 0, "", ""))
    sign.clearsign(run, tmp_path / "Release", tmp_path / "InRelease")
    sign.detach(run, tmp_path / "repomd.xml", tmp_path / "repomd.xml.asc")
    flat = [" ".join(c.args) for c in run.call_args_list]
    assert all("--pinentry-mode loopback" in c and "--passphrase-file" in c for c in flat)
    assert "--clearsign" in flat[0] and "--detach-sign --armor" in flat[1]

def test_size_guard(tmp_path: Path, capsys: pytest.CaptureFixture[str]) -> None:
    (tmp_path / "a").write_bytes(b"x" * 100)
    assert site.size_guard(tmp_path, warn=50, fail=200) == 100
    assert "WARNING: package site is 100 bytes" in capsys.readouterr().err
    with pytest.raises(PackageError, match=r"package site is 100 bytes, over the 90-byte limit"):
        site.size_guard(tmp_path, warn=50, fail=90)
```

  `sign` writes the passphrase from the env into a `0600` temp file for `--passphrase-file` and deletes it in a `finally`. `import_key` pipes the env key to `gpg --batch --import` via a temp file, deleted in the same way.

- [ ] **Step 6: Run, verify FAIL, implement `sign.py` and `site.size_guard`, verify PASS.**

- [ ] **Step 7: Failing test, then implement `site.build_site` and the `index` subcommand.**
  `build_site` steps:
  1. Load the config.
  2. `collect`.
  3. `verify` (keyfile `keys/<vendor>.asc`).
  4. `retention.select`.
  5. Wipe `out`.
  6. `apt.write`, with suites drawn from the registry's Ubuntu targets: `["noble", "resolute"]`.
  7. `rpm.write`, with els from the registry: `["el9", "el10"]`.
  8. `sign.import_key`; `clearsign` each `Release` → `InRelease`; `detach` each `Release` → `Release.gpg`; `detach` each `repomd.xml` → `repomd.xml.asc`.
  9. Copy `keys/<vendor>.asc` → `out/keys/<vendor>.asc`.
  10. Write `out/index.html`: a minimal page with the apt/dnf setup instructions and the key fingerprint.
  11. `size_guard(out)`.

  Test it with all collaborators monkeypatched, asserting the call order and that a size-guard failure propagates as a nonzero CLI exit.

- [ ] **Step 8: Validate; commit.**
  `vrg-commit --type feat --scope package --message "index: apt and dnf metadata, signing, size guard, vrg-package index (#<T7>)"`

---

## Task T8: the `package / evidence` gate in rulesets and evidence harvest

**Repo:** vergil-tooling. Blocked by: T1.

**Files:**

- Modify: `src/vergil_tooling/lib/github_config.py`. Add `_EVIDENCE_GATE_PREFIXES` (L557-575) and `_EVIDENCE_GATE_ORDER`, plus the required-check construction (L322-343).
- Test: `tests/vergil_tooling/test_github_config.py`, `tests/vergil_tooling/test_ci_evidence.py`

**Interfaces:**

- Produces: when the target repo's `vergil.toml` has `[package]`, `desired_ci_gates_ruleset` includes the required check `package / evidence`. `required_evidence_gates` then includes `package`, so `vrg-ci-evidence harvest` downloads `ci-evidence-package`.

- [ ] **Step 1: Read `github_config.py` L300-360 and L550-580** to see how `_lang_has_check` receives the repo's config, then write failing tests that mirror the existing `test / evidence` tests:

```python
def test_package_gate_required_when_package_section_present(...) -> None:
    # build the same fixture the existing test/evidence test uses, plus a [package] section
    checks = <call desired_ci_gates_ruleset exactly as the neighbouring test does>
    assert "package / evidence" in checks

def test_package_gate_absent_without_package_section(...) -> None:
    assert "package / evidence" not in checks

def test_required_evidence_gates_include_package(...) -> None:
    assert "package" in <required_evidence_gates(...) as the neighbouring test calls it>
```

  Copy the neighbouring tests' fixture construction and call verbatim; only the `[package]` section and the assertions differ.

- [ ] **Step 2: Run, verify FAIL.**

- [ ] **Step 3: Implement.** Append `("package /", "package")` to `_EVIDENCE_GATE_PREFIXES` (before the `("version /", None)` entry) and `"package"` to `_EVIDENCE_GATE_ORDER`. Where the required checks are built, add:

```python
if cfg.package is not None:  # use the config variable name the surrounding code uses
    required.append("package / evidence")
```

  Here `required` stands for the list the surrounding code appends to.

- [ ] **Step 4: Run and verify PASS; validate; commit.**
  `vrg-commit --type feat --scope ci --message "package / evidence gate: required with [package], harvested at release (#<T8>)"`

---

## Task T9: `vrg-release` deferred `package-index` stage

**Repo:** vergil-tooling. Blocked by: T1, T3.

**Files:**

- Create: `src/vergil_tooling/lib/release/package_index.py`
- Modify: `src/vergil_tooling/lib/release/orchestrator.py`. In `build_stages()` (L113-144), insert `Stage("package-index", _tracked("package-index", package_index.wait_for_index), mode="fail_defer")` immediately before `consumer-refresh`. Add a `_phase_details` branch (L198-236).
- Test: `tests/vergil_tooling/test_release_package_index.py`, plus additions to `test_release_orchestrator.py`.

**Interfaces:**

- Consumes: `config.read_config`, `matrix.select_targets`, `naming.package_name`, `orgs.for_vendor`, `ReleaseContext` (`ctx.repo_root`, `ctx.version`, `ctx.deferred_publish_failures`).
- Produces:
  - `package_index.deb_has(base_url, suite, arch, name, full_version, fetch) -> bool`.
  - `package_index.rpm_has(base_url, el, rpm_arch, name, version, release, fetch) -> bool`.
  - `package_index.wait_for_index(ctx, *, fetch=_fetch, sleep=time.sleep, timeout=1200, interval=30) -> None`.

- [ ] **Step 1: Failing tests.**

```python
_PACKAGES = "Package: vergil-tooling\nVersion: 2.1.240-1\nArchitecture: amd64\n\nPackage: other\nVersion: 1-1\n"

def test_deb_has() -> None:
    fetch = lambda url: _PACKAGES.encode() if url.endswith("/deb/dists/noble/main/binary-amd64/Packages") else b""
    assert package_index.deb_has("https://b", "noble", "amd64", "vergil-tooling", "2.1.240-1", fetch)
    assert not package_index.deb_has("https://b", "noble", "amd64", "vergil-tooling", "2.1.241-1", fetch)

def test_rpm_has_reads_primary_via_repomd() -> None:
    repomd = b'<repomd xmlns="http://linux.duke.edu/metadata/repo"><data type="primary"><location href="repodata/abc-primary.xml.gz"/></data></repomd>'
    primary = gzip.compress(b'<metadata><package><name>t</name><version epoch="0" ver="2.1.240" rel="1"/></package></metadata>')
    fetch = lambda url: repomd if url.endswith("repomd.xml") else primary
    assert package_index.rpm_has("https://b", "el9", "x86_64", "t", "2.1.240", "1", fetch)

def test_no_package_section_is_a_noop(release_ctx: ReleaseContext) -> None:
    package_index.wait_for_index(release_ctx, fetch=_boom, sleep=_boom)

def test_times_out_and_defers(release_ctx_with_package: ReleaseContext) -> None:
    clock = iter(range(0, 10_000, 30))
    with pytest.raises(ReleaseError, match=r"vergil-tooling 2\.1\.240-1 not visible in .* after 1200s"):
        package_index.wait_for_index(release_ctx_with_package, fetch=lambda u: b"", sleep=lambda s: None,
                                     now=lambda: next(clock))
    assert "package-index" in release_ctx_with_package.deferred_publish_failures

def test_appears_after_polling(release_ctx_with_package: ReleaseContext) -> None:
    answers = iter([b"", _PACKAGES.encode()])
    package_index.wait_for_index(release_ctx_with_package, fetch=lambda u: next(answers), sleep=lambda s: None)
```

  Build `release_ctx` and `release_ctx_with_package` the way `test_release_handoff.py` builds its `ReleaseContext`, writing `vergil.toml` (with or without the T1 `[package]` text) plus `pyproject.toml` into `tmp_path`, with `version="2.1.240"`. `_boom` raises `AssertionError`.

- [ ] **Step 2: Run, verify FAIL.**

- [ ] **Step 3: Implement `package_index.py`.**
  - `wait_for_index` reads the config and returns if there's no `[package]`.
  - Otherwise it picks the **first deb target** from `select_targets`, or the first rpm target if there are no deb targets. The full version is `<pkg.version or ctx.version>-1`.
  - It polls `deb_has`/`rpm_has` (with the `noarch` arch mapping) every `interval` until `timeout`.
  - On timeout it appends `"package-index"` to `ctx.deferred_publish_failures` **before** raising `ReleaseError(phase="package-index", command="poll package index", message=…, detail="re-run the publish-index workflow in <org>/packages; the release itself stands")`.
  - `fetch` defaults to `urllib.request.urlopen(url, timeout=30).read()`, and maps HTTP 404 to `b""`. Any other error propagates.
  - `now` defaults to `time.monotonic`.

  Because `consumer_refresh` already holds when `ctx.deferred_publish_failures` is non-empty (`handoff.py:15`), the refresh then waits on this stage with no further change.

- [ ] **Step 4: Orchestrator wiring test.** Assert that `_stage_names()` has `"package-index"` directly before `"consumer-refresh"`, and that `_phase_details("package-index", ctx)` returns a non-empty string. Implement it, run, verify PASS, validate, commit.
  `vrg-commit --type feat --scope release --message "deferred package-index stage; consumer refresh waits for the index (#<T9>)"`

---

## Task T10: VM provisioning installs vergil-tooling from the package repository

**Repo:** vergil-tooling. Blocked by: T3.

**Files:**

- Create: `src/vergil_tooling/lib/vm_packages.py`
- Modify: `src/vergil_tooling/lib/vm_guest.py`. `install_tooling` (L263-269) and `update_tooling` (L272-294) dispatch on the ref kind; keep `_uv_tool_install` for dev installs.
- Modify: `src/vergil_tooling/bin/vrg_vm.py`. Print the dev-mode banner in `_update_over_transport` (L1186-1205), `_st_install_tooling` (L627-629), `_cs_tooling` (L931), and at every `exec_session(` call site (grep for it).
- Test: `tests/vergil_tooling/test_vm_packages.py`, plus additions to `test_vm_guest.py` and `test_vrg_vm.py`.

**Interfaces:**

- Consumes: `repo_setup.bootstrap`, `orgs.for_vendor`, the `Transport` protocol (`run`, `pipe`).
- Produces:
  - `vm_packages.classify_ref(ref: str) -> Literal["line", "exact", "dev"]`.
  - `vm_packages.apt_pin(ref: str) -> str` (preferences-file text).
  - `vm_packages.packaged_install(transport, ref) -> None`.
  - `vm_packages.dev_install(transport, ref) -> None`.
  - `vm_packages.dev_ref(transport) -> str | None`.
  - `vm_packages.DEV_BANNER = "DEV tooling (ref {ref}) — not the packaged install"`.

- [ ] **Step 1: Failing tests.**

```python
@pytest.mark.parametrize(("ref", "kind"), [("v2.1", "line"), ("v2.1.226", "exact"), ("develop", "dev"),
                                           ("feature/123-x", "dev"), ("2.1", "dev"), ("v2", "dev")])
def test_classify(ref: str, kind: str) -> None:
    assert vm_packages.classify_ref(ref) == kind

def test_apt_pin_text() -> None:
    assert vm_packages.apt_pin("v2.1") == "Package: vergil-tooling\nPin: version 2.1.*\nPin-Priority: 1001\n"
    assert vm_packages.apt_pin("v2.1.226") == "Package: vergil-tooling\nPin: version 2.1.226-1\nPin-Priority: 1001\n"

def test_packaged_install_sequence(monkeypatch: pytest.MonkeyPatch) -> None:
    boot: list[tuple[str, str]] = []
    monkeypatch.setattr(vm_packages.repo_setup, "bootstrap", lambda run, org, fmt, suite, sudo: boot.append((fmt, suite)))
    t = _transport_answering({"cat /etc/os-release": "VERSION_CODENAME=noble\n",
                              "apt-cache policy vergil-tooling": "vergil-tooling:\n  Installed: (none)\n  Candidate: 2.1.240-1\n",
                              "uv tool list": ""})
    vm_packages.packaged_install(t, "v2.1")
    flat = _flat(t)
    assert boot == [("deb", "noble")]
    assert any("/etc/apt/preferences.d/vergil-tooling" in p for p in _piped(t))
    assert "sudo apt-get install -y vergil-tooling" in flat

def test_exact_installs_exact_version(monkeypatch: pytest.MonkeyPatch) -> None:
    # ... apt-cache policy candidate 2.1.226-1 → "sudo apt-get install -y vergil-tooling=2.1.226-1"
    ...

def test_line_with_no_candidate_fails_with_actionable_message(monkeypatch: pytest.MonkeyPatch) -> None:  # Review Focus 3
    t = _transport_answering({"cat /etc/os-release": "VERSION_CODENAME=noble\n",
                              "apt-cache policy vergil-tooling": "vergil-tooling:\n  Installed: (none)\n  Candidate: (none)\n",
                              "uv tool list": ""})
    monkeypatch.setattr(vm_packages.repo_setup, "bootstrap", lambda *a, **k: None)
    with pytest.raises(SystemExit):
        vm_packages.packaged_install(t, "v2.2")
    # stderr names the line and the repository
    ...

def test_packaged_install_removes_legacy_uv_install(monkeypatch: pytest.MonkeyPatch) -> None:  # Review Focus 5
    t = _transport_answering({"cat /etc/os-release": "VERSION_CODENAME=noble\n",
                              "apt-cache policy vergil-tooling": "  Candidate: 2.1.240-1\n",
                              "uv tool list": "vergil-tooling v2.1.200\n- vrg-git\n"})
    monkeypatch.setattr(vm_packages.repo_setup, "bootstrap", lambda *a, **k: None)
    vm_packages.packaged_install(t, "v2.1")
    flat = _flat(t)
    assert any("uv tool uninstall vergil-tooling" in c for c in flat)
    assert any("rm -f ~/.config/vergil/tooling-dev-ref" in c for c in flat)

def test_unsupported_codename_is_fatal(monkeypatch: pytest.MonkeyPatch) -> None:
    # VERSION_CODENAME=jammy → SystemExit, message lists supported suites noble, resolute
    ...

def test_dev_install_records_ref_and_uses_uv() -> None:
    t = _transport_answering({})
    vm_packages.dev_install(t, "develop")
    assert any("uv tool install" in c and "@develop" in c for c in _flat(t))
    assert any("tooling-dev-ref" in p for p in _piped(t))
```

  `_transport_answering(map)` is a `MagicMock` whose `run.side_effect` returns `CompletedProcess(args, 0, stdout)`, where `stdout` is the first map value whose key is a substring of `" ".join(args)`, else `""`. `_flat(t)` joins the `run` calls, and `_piped(t)` lists the `pipe` command strings. Write out the elided test bodies in full.

- [ ] **Step 2: Run, verify FAIL.**

- [ ] **Step 3: Implement `vm_packages.py`.**

```python
# src/vergil_tooling/lib/vm_packages.py
"""Install vergil-tooling on a VM from the org package repository (spec §9)."""

from __future__ import annotations

import re
import sys
from typing import Literal

from vergil_tooling.lib.package import orgs, repo_setup
from vergil_tooling.lib.package import targets as tg
from vergil_tooling.lib.vm_transport import Transport

_LINE = re.compile(r"v(\d+)\.(\d+)")
_EXACT = re.compile(r"v(\d+)\.(\d+)\.(\d+)")
_DEV_REF_FILE = "~/.config/vergil/tooling-dev-ref"
_PIN_FILE = "/etc/apt/preferences.d/vergil-tooling"
DEV_BANNER = "DEV tooling (ref {ref}) — not the packaged install"


def classify_ref(ref: str) -> Literal["line", "exact", "dev"]:
    if _EXACT.fullmatch(ref):
        return "exact"
    if _LINE.fullmatch(ref):
        return "line"
    return "dev"


def apt_pin(ref: str) -> str:
    version = f"{ref[1:]}-1" if classify_ref(ref) == "exact" else f"{ref[1:]}.*"
    return f"Package: vergil-tooling\nPin: version {version}\nPin-Priority: 1001\n"


def _die(msg: str) -> None:
    print(f"ERROR: {msg}", file=sys.stderr)
    raise SystemExit(1)


def _remove_uv_copy(transport: Transport) -> None:
    listed = transport.run("bash", "-c", 'export PATH="$HOME/.local/bin:$PATH"; uv tool list 2>/dev/null || true').stdout
    if re.search(r"^vergil-tooling\b", listed, re.MULTILINE):
        transport.run("bash", "-c", 'export PATH="$HOME/.local/bin:$PATH"; uv tool uninstall vergil-tooling')
    transport.run("bash", "-c", f"rm -f {_DEV_REF_FILE}")


def packaged_install(transport: Transport, ref: str) -> None:
    os_release = transport.run("bash", "-c", "cat /etc/os-release").stdout
    m = re.search(r"^VERSION_CODENAME=(\S+)", os_release, re.MULTILINE)
    suites = sorted({t.suite for t in tg.all_targets() if t.fmt == "deb"})
    if not m or m[1] not in suites:
        _die(f"VM OS codename {m[1] if m else '(unknown)'} is not a supported package target ({', '.join(suites)})")
    assert m is not None
    org = orgs.for_vendor("vergil")
    repo_setup.bootstrap(transport.run, org, "deb", m[1], sudo=True)
    transport.pipe(f"sudo tee {_PIN_FILE} >/dev/null", apt_pin(ref))
    transport.run("sudo", "apt-get", "update")
    policy = transport.run("bash", "-c", "apt-cache policy vergil-tooling").stdout
    cand = re.search(r"Candidate:\s*(\S+)", policy)
    if not cand or cand[1] == "(none)":
        _die(f"no packaged vergil-tooling matching {ref} in {org.base_url} — release it first, "
             f"or use 'vrg-vm update --tag <git-ref>' for a dev install")
    spec = f"vergil-tooling={ref[1:]}-1" if classify_ref(ref) == "exact" else "vergil-tooling"
    transport.run("sudo", "apt-get", "install", "-y", spec)
    _remove_uv_copy(transport)


def dev_install(transport: Transport, ref: str) -> None:
    from vergil_tooling.lib.vm_guest import _uv_tool_install  # noqa: PLC0415

    _uv_tool_install(transport, f"vergil-tooling @ git+https://github.com/vergil-project/vergil-tooling@{ref}",
                     reinstall=True)
    transport.run("bash", "-c", f"mkdir -p $(dirname {_DEV_REF_FILE})")
    transport.pipe(f"cat > {_DEV_REF_FILE}", f"{ref}\n")


def dev_ref(transport: Transport) -> str | None:
    out = transport.run("bash", "-c", f"cat {_DEV_REF_FILE} 2>/dev/null || true").stdout.strip()
    return out or None
```

  Order matters in `packaged_install`: `_remove_uv_copy` runs **after** a successful apt install, so a failed packaged install never leaves the VM with no tooling at all. Adjust `test_packaged_install_removes_legacy_uv_install` to assert that order.

- [ ] **Step 4: Rewire `vm_guest.py`.**
  - `install_tooling(transport, tag)` → `vm_packages.dev_install(transport, tag)` if `classify_ref(tag) == "dev"`, else `vm_packages.packaged_install(transport, tag)`. Then write the tag file exactly as today.
  - `update_tooling` keeps its tag resolution (explicit, then the tag file, then the fallback) and dispatches the same way. An explicit tag is still not persisted, so a plain `vrg-vm update` afterwards resolves to the persisted identity version (always `line`/`exact`), and `packaged_install` removes the dev copy.
  - Update the existing `TestInstallTooling`/`TestUpdateTooling` tests (`test_vm_guest.py` L391, L417), which currently expect a `uv tool install` for a release tag. Make them assert the dispatch instead, by monkeypatching `vm_packages.packaged_install`/`dev_install`.

- [ ] **Step 5: Add the dev banner.** In each `vrg_vm.py` site listed under Files, after the transport is available, add:

```python
ref = vm_packages.dev_ref(transport)
if ref:
    print(vm_packages.DEV_BANNER.format(ref=ref))
```

  Add a `test_vrg_vm.py` test for `_update_over_transport` asserting that the banner prints when `dev_ref` returns `"develop"` and is absent when it returns `None`.

- [ ] **Step 6: Run all the VM tests, validate, commit.**
  `vrg-commit --type feat --scope vm --message "VMs install vergil-tooling from the package repository; explicit dev installs (#<T10>)"`

---

## Task A1: vergil-actions `ci-package.yml` and the setup actions

**Repo:** vergil-actions. Blocked by: a **human release of vergil-tooling** containing T1, T2, T4, T5 and T8 (attested in the PR notes: "vergil-tooling vX.Y.Z includes vrg-package build/install-test").

**Files:**

- Create: `actions/package/setup/action.yml` (prerequisites, uv, vergil-tooling, nfpm inside any build/test container)
- Create: `actions/package/setup/tests/detect.test.sh`
- Create: `.github/workflows/ci-package.yml`
- Modify: `actions/ci/evidence/emit/action.yml` L12. Add `package` to the documented gate list.
- Modify: `.github/workflows/smoke-setup-vergil.yml`. Run `detect.test.sh` in the `fixtures` job.

**Interfaces:**

- Consumes: `vrg-package matrix --github-output`, `vrg-package build`, `vrg-package install-test`.
- Produces: the reusable workflow `ci-package.yml` (no inputs), whose gate job `evidence` becomes `package / evidence` under a caller job keyed `package`. Build artifacts are named `package-build-<cell id>` and evidence partials `ci-evidence-package-reports-<cell id>`. The final `ci-evidence-package` comes from `emit` with `gate: package`.

- [ ] **Step 1: Write `actions/package/setup/action.yml`.**

```yaml
name: Package toolchain setup
description: >-
  Install prerequisites, uv, vergil-tooling and nFPM inside an arbitrary build or
  install-test container (ubuntu or UBI). Self-repo installs vergil-tooling from
  the checkout; other repos from [dependencies].vergil.
inputs:
  nfpm:
    description: Install nFPM (build cells only)
    default: "false"
runs:
  using: composite
  steps:
    - name: OS prerequisites
      shell: bash
      run: |
        set -euo pipefail
        if command -v apt-get >/dev/null; then
          apt-get update && apt-get install -y --no-install-recommends ca-certificates curl git tar gzip
        else
          dnf install -y ca-certificates curl-minimal git tar gzip
        fi
        git config --global --add safe.directory "$GITHUB_WORKSPACE"
    - name: uv
      shell: bash
      env:
        UV_VERSION: "0.9.2"   # pin; bump deliberately
      run: |
        set -euo pipefail
        curl -LsSf "https://astral.sh/uv/${UV_VERSION}/install.sh" | env UV_INSTALL_DIR=/usr/local/bin sh
    - name: vergil-tooling
      shell: bash
      run: |
        set -euo pipefail
        bash "${{ github.action_path }}/detect.sh" > /tmp/vrg-spec
        uv tool install --python 3.14 "$(cat /tmp/vrg-spec)"
        echo "$HOME/.local/bin" >> "$GITHUB_PATH"
    - name: nFPM
      if: inputs.nfpm == 'true'
      shell: bash
      env:
        NFPM_VERSION: "2.43.0"
        NFPM_SHA256_X86_64: "<from the release's checksums.txt>"
        NFPM_SHA256_ARM64: "<from the release's checksums.txt>"
      run: |
        set -euo pipefail
        case "$(uname -m)" in
          x86_64)  a=x86_64; sum="$NFPM_SHA256_X86_64" ;;
          aarch64) a=arm64;  sum="$NFPM_SHA256_ARM64" ;;
          *) echo "unsupported arch $(uname -m)" >&2; exit 1 ;;
        esac
        f="nfpm_${NFPM_VERSION}_Linux_${a}.tar.gz"
        curl -fsSLo "/tmp/$f" "https://github.com/goreleaser/nfpm/releases/download/v${NFPM_VERSION}/$f"
        echo "${sum}  /tmp/$f" | sha256sum -c -
        tar -xzf "/tmp/$f" -C /usr/local/bin nfpm
```

  **Pin step (part of this task):** pick the current nFPM and uv releases, then replace the version and the two `NFPM_SHA256_*` values with the real entries from that release's `checksums.txt`. The `<from …>` markers must not survive this step. Also create `actions/package/setup/detect.sh`:

- if `pyproject.toml` declares `name = "vergil-tooling"`, print `.` (the self-repo checkout);
- otherwise print `vergil-tooling @ git+https://github.com/vergil-project/vergil-tooling@<[dependencies].vergil>`, read with `python3 -c 'import tomllib,…'` (uv's Python is available after the uv step: run `uv run --no-project python3 -c …`).

  `detect.test.sh` covers both branches, in the style of `actions/shared/setup/vergil/tests/derive.test.sh`.

- [ ] **Step 2: Write `.github/workflows/ci-package.yml`.**

```yaml
name: CI Package

on:
  workflow_call: {}

permissions:
  contents: read

jobs:
  matrix:
    runs-on: ubuntu-latest
    outputs:
      build: ${{ steps.m.outputs.build }}
      test: ${{ steps.m.outputs.test }}
    steps:
      - uses: actions/checkout@v6
      - uses: ./actions/shared/setup/vergil
      - id: m
        shell: bash
        run: |
          set -euo pipefail
          vrg-package matrix --github-output
          grep -q '^enabled=true$' "$GITHUB_OUTPUT" || { echo "::error::ci-package called but vergil.toml has no [package]"; exit 1; }

  build:
    name: build / ${{ matrix.id }}
    needs: matrix
    runs-on: ${{ matrix.runner }}
    container: ${{ matrix.image }}
    strategy:
      fail-fast: false
      matrix:
        include: ${{ fromJSON(needs.matrix.outputs.build) }}
    steps:
      - uses: actions/checkout@v6
      - uses: ./actions/package/setup
        with:
          nfpm: "true"
      - shell: bash
        run: vrg-package build --cell "${{ matrix.id }}" --version "$(cat VERSION)" --out dist/packages
      - uses: actions/upload-artifact@v4
        with:
          name: package-build-${{ matrix.id }}
          path: dist/packages/*.deb
          if-no-files-found: ignore
      - uses: actions/upload-artifact@v4
        with:
          name: package-build-${{ matrix.id }}-rpm
          path: dist/packages/*.rpm
          if-no-files-found: ignore

  install-test:
    name: install-test / ${{ matrix.id }}
    needs: [matrix, build]
    runs-on: ${{ matrix.runner }}
    container: ${{ matrix.image }}
    strategy:
      fail-fast: false
      matrix:
        include: ${{ fromJSON(needs.matrix.outputs.test) }}
    steps:
      - uses: actions/checkout@v6
      - uses: ./actions/package/setup
      - uses: actions/download-artifact@v4
        with:
          pattern: package-build-*
          merge-multiple: true
          path: dist/packages
      - shell: bash
        run: |
          mkdir -p reports
          vrg-package install-test --cell "${{ matrix.id }}" --artifacts dist/packages --report "reports/${{ matrix.id }}.json"
      - if: always()
        uses: actions/upload-artifact@v4
        with:
          name: ci-evidence-package-reports-${{ matrix.id }}
          path: reports/
          if-no-files-found: warn

  evidence:
    needs: [build, install-test]
    if: always()
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v6
      - name: Assert all required legs succeeded
        shell: bash
        env:
          RESULTS: build=${{ needs.build.result }} install-test=${{ needs.install-test.result }}
        run: |
          for r in $RESULTS; do
            [ "${r#*=}" = success ] || { echo "::error::${r%%=*} did not succeed (${r#*=})"; exit 1; }
          done
      - uses: actions/download-artifact@v4
        with:
          pattern: ci-evidence-package-reports-*
          merge-multiple: true
          path: evidence-in
      - uses: ./actions/ci/evidence/emit
        with:
          gate: package
          tools: nfpm,vrg-package
          path: evidence-in
```

  Match `emit`'s real input names by reading `actions/ci/evidence/emit/action.yml` and adjusting the last step's `with:` to them. The `ci-test.yml` gate at L113-171 is the model.

- [ ] **Step 3: Validate locally.** Run `vrg-container-run -- vrg-validate`; actionlint, shellcheck and yamllint must be green. Run `bash actions/package/setup/tests/detect.test.sh`.

- [ ] **Step 4: Live check.** Push the branch. In a scratch branch of vergil-tooling, add a caller job pinned to this branch:

  ```yaml
  package:
    uses: vergil-project/vergil-actions/.github/workflows/ci-package.yml@<branch>
  ```

  Confirm the build legs go green on both runners. Install-test legs for `python` products need DEP2; until then, confirm that the matrix/build legs work and that install-test fails **loudly** at bootstrap (the expected state). Record the run URL in the PR notes, then discard the scratch branch.

- [ ] **Step 5: Commit.**
  `vrg-commit --type feat --scope package --message "ci-package reusable workflow and package toolchain setup (#<A1>)"`

---

## Task A2: split `cd-release.yml`: package build, signing environment, attach, dispatch

**Repo:** vergil-actions. Blocked by: A1.

**Files:**

- Modify: `.github/workflows/cd-release.yml`. Add the jobs `package-matrix`, `package-build` and `package-sign`; make `release` depend on them; append the package artifacts to the release; dispatch after tagging. Add the optional secrets `APP_CLIENT_ID` and `APP_PRIVATE_KEY`.

**Interfaces:**

- Consumes: `vrg-package matrix --github-output --manifest`, `vrg-package build`, the `package-signing` environment secrets.
- Produces:
  - Release assets: the signed `.rpm`s, the `.deb`s, and `packages-manifest.json`.
  - A `repository_dispatch` of `event_type: package-released` to `<owner>/packages` with `client_payload.product = <owner/repo>`.
  - Repos without `[package]` see no change: the package jobs are skipped and `release` runs as today.

- [ ] **Step 1: Add the jobs before `release`.**

```yaml
  package-matrix:
    runs-on: ubuntu-latest
    outputs:
      enabled: ${{ steps.m.outputs.enabled }}
      build: ${{ steps.m.outputs.build }}
    steps:
      - uses: actions/checkout@v6
      - uses: ./actions/shared/setup/vergil
      - id: m
        shell: bash
        run: |
          vrg-package matrix --github-output --manifest packages-manifest.json
      - if: steps.m.outputs.enabled == 'true'
        uses: actions/upload-artifact@v4
        with:
          name: package-manifest
          path: packages-manifest.json

  package-build:
    name: package-build / ${{ matrix.id }}
    needs: package-matrix
    if: needs.package-matrix.outputs.enabled == 'true'
    runs-on: ${{ matrix.runner }}
    container: ${{ matrix.image }}
    strategy:
      fail-fast: true
      matrix:
        include: ${{ fromJSON(needs.package-matrix.outputs.build) }}
    steps:
      - uses: actions/checkout@v6
      - uses: ./actions/package/setup
        with:
          nfpm: "true"
      - shell: bash
        run: vrg-package build --cell "${{ matrix.id }}" --version "$(vrg-version show)" --out dist/packages
      - uses: actions/upload-artifact@v4
        with:
          name: package-unsigned-${{ matrix.id }}
          path: dist/packages/*.*[mb]   # .deb and .rpm

  package-sign:
    needs: [package-matrix, package-build]
    if: needs.package-matrix.outputs.enabled == 'true'
    runs-on: ubuntu-latest
    environment: package-signing
    permissions:
      contents: read
      id-token: write
      attestations: write
    steps:
      - uses: actions/download-artifact@v4
        with:
          pattern: package-unsigned-*
          merge-multiple: true
          path: dist/packages
      - name: Sign rpms
        shell: bash
        env:
          PACKAGE_SIGNING_KEY: ${{ secrets.PACKAGE_SIGNING_KEY }}
          PACKAGE_SIGNING_PASSPHRASE: ${{ secrets.PACKAGE_SIGNING_PASSPHRASE }}
        run: |
          set -euo pipefail
          [ -n "$PACKAGE_SIGNING_KEY" ] || { echo "::error::package-signing environment has no PACKAGE_SIGNING_KEY"; exit 1; }
          sudo apt-get update && sudo apt-get install -y rpm gnupg
          printf '%s' "$PACKAGE_SIGNING_KEY" | gpg --batch --import
          umask 077; printf '%s' "$PACKAGE_SIGNING_PASSPHRASE" > /tmp/pp
          keyid=$(gpg --list-secret-keys --with-colons | awk -F: '/^ssb/ && $12 ~ /s/ {print $5; exit}')
          shopt -s nullglob
          for f in dist/packages/*.rpm; do
            rpmsign --define "_gpg_name $keyid" \
              --define "_gpg_sign_cmd_extra_args --batch --pinentry-mode loopback --passphrase-file /tmp/pp" \
              --addsign "$f"
          done
          rm -f /tmp/pp
          gpg --armor --export > /tmp/pub.asc && sudo rpmkeys --import /tmp/pub.asc
          for f in dist/packages/*.rpm; do rpmkeys --checksig "$f" | grep -q 'signatures OK'; done
      - uses: actions/attest-build-provenance@v4
        with:
          subject-path: dist/packages/*
      - uses: actions/upload-artifact@v4
        with:
          name: package-signed
          path: dist/packages/
```

- [ ] **Step 2: Change `release`.**
  - `needs: [package-matrix, package-sign]`
  - The gating condition: when packaging is enabled, `package-sign` must have succeeded. A failed `package-build` makes `package-sign` *skipped*, and that must **not** let `release` run.

```yaml
    if: >-
      ${{ always() &&
          needs.package-matrix.result == 'success' &&
          (needs.package-matrix.outputs.enabled != 'true' || needs.package-sign.result == 'success') }}
```

  Add these steps before `Resolve release artifacts`:

```yaml
      - if: needs.package-matrix.outputs.enabled == 'true'
        uses: actions/download-artifact@v4
        with:
          name: package-signed
          path: dist-packages
      - if: needs.package-matrix.outputs.enabled == 'true'
        uses: actions/download-artifact@v4
        with:
          name: package-manifest
          path: dist-packages
```

  In `Resolve release artifacts`, append `dist-packages/*` (space-separated) when `ENABLED == 'true'` (pass `needs.package-matrix.outputs.enabled` in via `env`). After `Tag and release`, add:

```yaml
      - name: Dispatch package index
        if: needs.package-matrix.outputs.enabled == 'true' && steps.tag_check.outputs.exists == 'false'
        id: app
        continue-on-error: true   # deferred: the release stands; vrg-release's package-index stage reports a miss
        uses: actions/create-github-app-token@v3
        with:
          app-id: ${{ secrets.APP_CLIENT_ID }}
          private-key: ${{ secrets.APP_PRIVATE_KEY }}
          owner: ${{ github.repository_owner }}
          repositories: packages
      - if: steps.app.outcome == 'success'
        continue-on-error: true
        shell: bash
        env:
          GH_TOKEN: ${{ steps.app.outputs.token }}
          OWNER: ${{ github.repository_owner }}
          PRODUCT: ${{ github.repository }}
        run: |
          gh api "repos/$OWNER/packages/dispatches" -f event_type=package-released -f "client_payload[product]=$PRODUCT"
```

  Add `APP_CLIENT_ID` and `APP_PRIVATE_KEY` (`required: false`) to `workflow_call.secrets`.

- [ ] **Step 3: Validate** (actionlint). **Live check:** run a no-`[package]` consumer's CD on a scratch branch pinned to this branch and confirm `release` behaves exactly as before. Record the run in the PR notes. The packaged path is exercised by DEP1.

- [ ] **Step 4: Commit.**
  `vrg-commit --type feat --scope release --message "cd-release: package build, main-only signing environment, attach, dispatch (#<A2>)"`

---

## Task A3: `publish-index.yml` reusable workflow

**Repo:** vergil-actions. Blocked by: a human release of vergil-tooling containing T6 and T7.

**Files:**

- Create: `.github/workflows/publish-index.yml`

**Interfaces:**

- Consumes: `vrg-package index`, the `index-signing` environment of the calling repo, and the Pages environment `github-pages`.
- Produces: a deployed Pages site for the calling `<org>/packages` repo.

- [ ] **Step 1: Write the workflow.**

```yaml
name: Publish package index

on:
  workflow_call: {}

permissions:
  contents: read

concurrency:
  group: publish-index
  cancel-in-progress: false

jobs:
  build-index:
    runs-on: ubuntu-latest
    environment: index-signing   # develop-only: dispatch/schedule run on the default branch (spec §7.4)
    permissions:
      contents: read
      attestations: read
    steps:
      - uses: actions/checkout@v6
      - uses: ./actions/shared/setup/vergil
      - name: Index tools
        shell: bash
        run: sudo apt-get update && sudo apt-get install -y createrepo-c rpm dpkg-dev gnupg
      - name: Build, verify, sign
        shell: bash
        env:
          GH_TOKEN: ${{ github.token }}
          PACKAGE_SIGNING_KEY: ${{ secrets.PACKAGE_SIGNING_KEY }}
          PACKAGE_SIGNING_PASSPHRASE: ${{ secrets.PACKAGE_SIGNING_PASSPHRASE }}
        run: vrg-package index --config packages.toml --keys keys --out _site
      - uses: actions/upload-pages-artifact@v3
        with:
          path: _site

  deploy:
    needs: build-index
    runs-on: ubuntu-latest
    environment:
      name: github-pages
      url: ${{ steps.d.outputs.page_url }}
    permissions:
      pages: write
      id-token: write
    steps:
      - id: d
        uses: actions/deploy-pages@v4
```

  The `concurrency` group serializes index rebuilds.

- [ ] **Step 2: Validate** (actionlint). The live check happens in DEP1.

- [ ] **Step 3: Commit.**
  `vrg-commit --type feat --scope package --message "publish-index reusable workflow (#<A3>)"`

---

## Task P1: `vergil-project/packages`: keyring product and index config

**Repo:** vergil-project/packages (filed by OP1 step 10). Blocked by: OP1, and a human release of vergil-actions containing A1–A3.

**Files (all new):**

- `VERSION` (`1.0.0`)
- `vergil.toml`
- `packages.toml`
- `keys/vergil.asc`: the public key from OP1's SUCCESS comment, verbatim.
- `packaging/nfpm.overlay.yaml`
- `packaging/postinst.sh`, `packaging/postrm.sh` (deb: write and remove the codename-specific apt source)
- `.github/workflows/ci.yml`, `.github/workflows/cd.yml`, `.github/workflows/publish-index.yml`
- `README.md`: consumer instructions, and the fingerprint.

- [ ] **Step 1: `vergil.toml`.**

```toml
[project]
repository-type = "library"
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

[package]
builder = "staged"
vendor = "vergil"
name = "vergil-archive-keyring"
summary = "Signing key and repository configuration for the vergil-project package repository"
smoke = "test -s /usr/share/keyrings/vergil-archive-keyring.asc"
noarch = true
```

- [ ] **Step 2: `packages.toml`.**

```toml
vendor = "vergil"
products = [
  "vergil-project/packages",
  "vergil-project/vergil-python",
  "vergil-project/vergil-tooling",
]

[retention]
keep = 3
lines = 2
```

- [ ] **Step 3: The overlay and maintainer scripts.**

```yaml
# packaging/nfpm.overlay.yaml
contents:
  - src: keys/vergil.asc
    dst: /usr/share/keyrings/vergil-archive-keyring.asc
    file_info: {mode: 0644}
  - src: keys/vergil.asc
    dst: /etc/pki/rpm-gpg/RPM-GPG-KEY-vergil
    packager: rpm
    file_info: {mode: 0644}
  - src: packaging/vergil.repo
    dst: /etc/yum.repos.d/vergil.repo
    packager: rpm
    type: config|noreplace
scripts:
  postinstall: packaging/postinst.sh
  postremove: packaging/postrm.sh
```

  `packaging/vergil.repo`:

```ini
[vergil]
name=vergil-project packages
baseurl=https://vergil-project.github.io/packages/rpm/el$releasever/$basearch
gpgcheck=1
repo_gpgcheck=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-vergil
enabled=1
```

  `packaging/postinst.sh` writes the Ubuntu source only on dpkg systems, because one `.deb` serves both codenames:

```sh
#!/bin/sh
set -e
if command -v dpkg >/dev/null 2>&1 && [ -d /etc/apt/sources.list.d ]; then
  . /etc/os-release
  cat > /etc/apt/sources.list.d/vergil.sources <<EOF
Types: deb
URIs: https://vergil-project.github.io/packages/deb
Suites: ${VERSION_CODENAME}
Components: main
Signed-By: /usr/share/keyrings/vergil-archive-keyring.asc
EOF
fi
```

  `packaging/postrm.sh`:

```sh
#!/bin/sh
set -e
case "$1" in
  remove|purge|0) rm -f /etc/apt/sources.list.d/vergil.sources ;;
esac
```

- [ ] **Step 4: Workflows.**
  - `ci.yml` calls `quality` (language `shell`, as vergil-actions does) and `package: uses: vergil-project/vergil-actions/.github/workflows/ci-package.yml@v2.1`.
  - `cd.yml` mirrors vergil-tooling's `cd.yml` release job, plus `secrets: inherit`, so the dispatch can use `APP_*`. It also needs `permissions` that include `attestations: write` and `id-token: write`.
  - `publish-index.yml`:

```yaml
name: Publish package index
on:
  repository_dispatch:
    types: [package-released]
  workflow_dispatch: {}
  schedule:
    - cron: "17 4 * * 1"   # weekly reconcile (spec §7.2)
permissions:
  contents: read
  pages: write
  id-token: write
  attestations: read
jobs:
  index:
    uses: vergil-project/vergil-actions/.github/workflows/publish-index.yml@v2.1
    secrets: inherit
```

- [ ] **Step 5: Validate** (`vrg-container-run -- vrg-validate`). The PR's `package / evidence` must be green across all 8 test cells. That proves the keyring installs and removes cleanly on every target, with no repo bootstrap needed (staged product).

- [ ] **Step 6: Commit.**
  `vrg-commit --type feat --scope packages --message "keyring product and index configuration (#<P1>)"`

---

## Task P2: `vergil-project/vergil-python`: the runtime package

**Repo:** vergil-project/vergil-python (filed by OP1 step 10). Blocked by: OP1, and a human release of vergil-actions containing A1 and A2.

**Files (all new):** `VERSION` (`1.0.0`), `vergil.toml`, `runtime.toml`, `packaging/build.sh`, `docs/runtime-package-pattern.md`, `.github/workflows/{ci,cd}.yml`, `README.md`.

- [ ] **Step 1: Choose the pin.** Take the newest python-build-standalone release that ships CPython 3.14.x `install_only_stripped` for both `x86_64-unknown-linux-gnu` and `aarch64-unknown-linux-gnu`. Record its tag (`YYYYMMDD`), the CPython patch, and both SHA-256 values from the release's `SHA256SUMS`.

```toml
# runtime.toml
cpython = "3.14.N"        # the chosen patch
pbs_tag = "YYYYMMDD"      # the chosen PBS release tag
[sha256]
x86_64  = "…"             # from SHA256SUMS
aarch64 = "…"
```

  Write the real values; no `N`/`YYYYMMDD`/`…` may remain.

- [ ] **Step 2: `vergil.toml` `[package]`.** The other sections are as in P1.

```toml
[package]
builder = "staged"
vendor = "vergil"
name = "vergil-python3.14.N"
version = "3.14.N+YYYYMMDD"
summary = "Pinned CPython 3.14.N (python-build-standalone YYYYMMDD) for vergil products"
smoke = "/opt/vergil/python/3.14.N/bin/python3.14 -c 'import ssl, sqlite3, ctypes, zlib; print(ssl.OPENSSL_VERSION)'"

[package.staged]
build-command = "packaging/build.sh"
```

  Use the same real values as `runtime.toml`.

- [ ] **Step 3: `packaging/build.sh`.**

```bash
#!/usr/bin/env bash
# Fetch, verify and lay out the pinned PBS runtime (spec §6.2, §6.6).
set -euo pipefail
cd "$(dirname "$0")/.."
read_toml() { uv run --no-project python3 -c "import tomllib,sys;d=tomllib.load(open('$1','rb'));print(eval(sys.argv[1],{},{'d':d}))" "$2"; }
cpython=$(read_toml runtime.toml "d['cpython']")
tag=$(read_toml runtime.toml "d['pbs_tag']")
name=$(read_toml vergil.toml "d['package']['name']")
version=$(read_toml vergil.toml "d['package']['version']")
[ "$name" = "vergil-python${cpython}" ] || { echo "vergil.toml name $name != vergil-python${cpython}" >&2; exit 1; }
[ "$version" = "${cpython}+${tag}" ] || { echo "vergil.toml version $version != ${cpython}+${tag}" >&2; exit 1; }
case "$VRG_TARGET_ARCH" in
  amd64) triple=x86_64-unknown-linux-gnu; key=x86_64 ;;
  arm64) triple=aarch64-unknown-linux-gnu; key=aarch64 ;;
  *) echo "unsupported VRG_TARGET_ARCH=$VRG_TARGET_ARCH" >&2; exit 1 ;;
esac
sum=$(read_toml runtime.toml "d['sha256']['$key']")
f="cpython-${cpython}+${tag}-${triple}-install_only_stripped.tar.gz"
curl -fsSLo "/tmp/$f" "https://github.com/astral-sh/python-build-standalone/releases/download/${tag}/${f}"
echo "${sum}  /tmp/$f" | sha256sum -c -
dest="$VRG_STAGING_ROOT/opt/vergil/python/${cpython}"
mkdir -p "$dest"
tar -xzf "/tmp/$f" -C "$dest" --strip-components=1   # archive root is python/
minor="${cpython%.*}"
cat > "$dest/lib/python${minor}/EXTERNALLY-MANAGED" <<'EOF'
[externally-managed]
Error=This interpreter is the shared vergil runtime. Install into a product venv, never into the runtime.
EOF
```

- [ ] **Step 4: `docs/runtime-package-pattern.md`.** Write it from spec §6.2 and §4.1:
  - why a runtime is its own package;
  - side-by-side per patch;
  - exact CPython pin with floating PBS rebuilds;
  - never on `PATH`, plus the PEP 668 marker;
  - how to adopt a new patch (one PR: `runtime.toml` + `vergil.toml` name/version/smoke);
  - why the pattern applies to Ruby, Perl and the JVM, but not to native-binary languages;
  - the version-collision rule: never re-release an unchanged `name`/`version` (the index rejects it).

- [ ] **Step 5: Workflows** as in P1, minus `publish-index.yml`. `cd.yml` passes `secrets: inherit`.

- [ ] **Step 6: Validate.** `package / evidence` must be green on all 8 cells, which proves the smoke import of `ssl`/`sqlite3`/`ctypes`/`zlib` on Ubuntu 24.04/26.04 and RHEL 9/10, both architectures. The glibc guard also runs over the PBS tree in each build cell.

- [ ] **Step 7: Commit.**
  `vrg-commit --type feat --scope runtime --message "vergil-python runtime package (#<P2>)"`

---

## DEP1 (deployment): package repository live with the keyring

**Repo:** vergil-project/packages. Kind: `deployment`. Blocked by: P1.

- **Precondition (human-attested):** P1 is released (`vrg-release` in `vergil-project/packages`).
- [ ] Confirm the `publish-index` run that the release dispatched succeeded. Run `workflow_dispatch` if the dispatch was missed.
- [ ] Ubuntu: in `docker run --rm ubuntu:24.04` and `ubuntu:26.04` (on the matching arch where available), follow the README bootstrap. Then `apt-get update` with the keyring's own source must succeed (signature verified), and `apt-cache policy vergil-archive-keyring` must show a candidate.
- [ ] RHEL: in `registry.access.redhat.com/ubi9/ubi` and `ubi10/ubi`, run `dnf install` of the keyring from the repository with `repo_gpgcheck=1`. This is the live check of the §7.3 `xml:base` pool layout.
- [ ] **SUCCESS** records the four container results. **FAILURE** leaves the task open with the failing output.

## DEP2 (deployment): runtime indexed

**Repo:** vergil-project/vergil-python. Kind: `deployment`. Blocked by: P2, DEP1.

- **Precondition (human-attested):** P2 is released.
- [ ] Confirm the index includes `vergil-python3.14.N` for amd64 and arm64 (deb and rpm).
- [ ] In clean `ubuntu:24.04`, `ubuntu:26.04`, `ubi9` and `ubi10` containers, bootstrap via the keyring, then `apt`/`dnf install vergil-python3.14.N` and run the smoke command. All of them pass.

## Task T11: vergil-tooling adopts `[package]`

**Repo:** vergil-tooling. Blocked by: DEP2, and a human release of vergil-actions containing A1–A2.

**Files:**

- Modify: `vergil.toml`. Add the `[package]` section from spec §5.2 (`builder = "python"`, `vendor = "vergil"`, `summary`, `smoke = "vrg-whoami --mode"`, `[package.python] runtime = "<the 3.14.N from DEP2>"`).
- Modify: `.github/workflows/ci.yml`. Add a `package:` job calling `ci-package.yml@v2.1`.
- Modify: `.github/workflows/cd.yml`. The `release` job adds `secrets: inherit`.
- Modify: `docs/site/docs/reference/` gets a new `package-config.md` documenting `[package]`, the targets and the overlay contract; add it to `docs/site/mkdocs.yml` nav.

- [ ] **Step 1:** Make the changes above. `uv.lock` already exists.
- [ ] **Step 2:** `vrg-container-run -- vrg-validate` green.
- [ ] **Step 3:** The PR's `package / evidence` is green on all 8 cells. The python builder bootstraps the live vergil repository, installs `vergil-python3.14.N`, builds the venv from `uv.lock`, and the install test runs `vrg-whoami --mode` from `/usr/bin` and then removes cleanly.
- [ ] **Step 4:** Commit.
  `vrg-commit --type feat --scope package --message "publish vergil-tooling as a signed .deb/.rpm (#<T11>)"`

## DEP3 (deployment): packaged vergil-tooling live; existing VMs switched

**Repo:** vergil-tooling. Kind: `deployment`. Blocked by: T9, T10, T11.

- **Precondition (human-attested):** vergil-tooling is released with T9–T11. That release's `package-index` stage reports the version visible.
- [ ] Confirm the index has `vergil-tooling <version>-1` for all 4 artifacts.
- [ ] Run `vrg-vm update --all`. Every running box reports the packaged version. `which vrg-git` resolves to `/usr/bin/vrg-git`. No box shows the DEV banner, and `uv tool list` no longer shows vergil-tooling.
- [ ] On one box, run `vrg-vm update --tag develop`: the DEV banner appears. Then a plain `vrg-vm update` returns the box to the packaged install, with no banner.

## VAL1 (validation): cold rebuild

**Repo:** vergil-tooling. Kind: `validation`. Blocked by: DEP3.

- [ ] Create a **fresh Lima arm64 VM** (`vrg-vm create`). Provisioning installs vergil-tooling via apt. `vrg-whoami --mode` works, and `dpkg -S /usr/bin/vrg-whoami` names `vergil-tooling`.
- [ ] Do the same on a **fresh cloud amd64 VM**.
- [ ] Install vergil-tooling from the live repository into clean `ubi9` and `ubi10` containers, amd64 and arm64, then run `vrg-whoami --mode`.
- [ ] Negative check: run `apt-get install` against a tampered `InRelease` (modify one byte in a local mirror copy). It must fail signature verification.
- [ ] **SUCCESS** records all results; on any failure the task stays open.

---

## Self-review notes

- **Spec coverage:**

| Spec section | Task(s) |
|---|---|
| §5.1–§5.4 | T1 |
| §6.1 | T4 |
| §6.2 | P2 |
| §6.3 | T4, P2 |
| §6.4 | T2 (`package_release`) and the `--version` wiring in A1/A2 |
| §6.5 | T2 (script check), T5 (unit verify) |
| §6.6 | T2 |
| §7.1 | P1 |
| §7.2 | T6, T7, A3 |
| §7.3 | T7 (live check in DEP1) |
| §7.4 | OP1, A2, A3 |
| §8.1 | A1, T5, T8 |
| §8.2 | A2 |
| §8.3 | T9 |
| §9 | T3, T10 |
| §11 | each task's tests |
| §12 | OP1, DEP1–3, VAL1 |

  §4.3 (LMF) is deliberately not a task here; LMF adds an `ORGS` entry and a `packages` repo in its own epic.

- **Spec corrections surfaced while planning.** These were approved at alignment and are **applied to spec.md**, together with alignment [1] (the `index-signing` environment for the develop-branch index job) and [2] (the sanitized smoke environment):
  1. `package / evidence` is **required only in repos with `[package]`**, following the existing per-repo `_lang_has_check` ruleset pattern (`github_config.py:322-343`). It is not "trivially passing everywhere": rulesets are computed per repo, and the release-time harvester only recognizes registered gates (`ci_evidence.py:536`, `github_config.py:557-575`).
  2. `vrg-validate` checks the overlay's **shape** (mapping, allowed keys), not nFPM's own config check. nFPM isn't in the dev container; nFPM's validation runs at build time in CI.
  3. New `[package]` keys: `version` (an explicit package version, needed by `vergil-python`, whose package version is `<cpython>+<pbs>`, not the repo's semver) and `noarch` (the keyring is architecture-independent; building it per arch would collide in the index).
  4. Each release attaches `packages-manifest.json`. The index uses it to tell pre-packaging releases (ignored) from partial ones (fatal).
  5. apt metadata is generated in Python (`apt.py`) rather than by `apt-ftparchive`. That gives per-suite selection for `native` builds without view-directory hacks; `createrepo_c` is still used for dnf.
  6. The keyring `.deb` writes its apt source in `postinst`, from `/etc/os-release`. One `.deb` serves both Ubuntu codenames, so a static `Suites:` line can't.
- **Type consistency:** `BuildCell`/`TestCell`/`Matrix` (T1) are used unchanged in T2/T4/T5. `BuildContext`/`BuildResult` (T2) feed T4. `OrgRepo`/`bootstrap` (T3) feed T4/T5/T10. `Artifact` (T6) feeds T7. `Run` (T3) is the subprocess seam everywhere.
