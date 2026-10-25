# Multi-series Python Runtime — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** vergil-python publishes one `vergil-python<x.y>` package per listed series (3.14 and 3.15), each kept current by a scheduled `vrg-upstream-refresh` PR, and every release republishes every series under a version that carries the upstream inputs plus the repo release.

**Architecture:** vergil-tooling turns `[package]` into a list, threads the package name through the matrix, build, install-test, runtime check, index and release wait, and computes every package version through one helper. A new language-neutral `vrg-upstream-refresh` command runs a repo's `[upstream-refresh].hook` and opens one PR or exits quietly. A reusable vergil-actions workflow runs it on a schedule. vergil-python (series list, per-series hook, leading-edge add) and vergil-containers' tool bumps both use it.

**Tech Stack:** Python 3.12+ (vergil-tooling: pytest, `lib/git.py`, `lib/github.py`), stdlib Python and bash (vergil-python hook and `build.sh`: unittest), GitHub Actions reusable workflows (vergil-actions), nFPM `.deb`/`.rpm`.

**Spec:** `epics/472-multi-series-runtime/spec.md`

## Global Constraints

- A single `[package]` table must parse, build, release and index exactly as today. Cell ids, artifact names and identities stay unchanged for a single-package repo.
- With more than one `[[package]]` entry, every entry sets an explicit, unique `name`, and every entry has the same `vendor`.
- Full package version for an entry with `upstream-version`: `<upstream-version>.<X.Y.Z>`, where `X.Y.Z` is the repo's `VERSION`. Example: `3.15.1+20261021.1.0.5`. It is computed only by `naming.package_version`.
- Version order: a newer CPython patch sorts higher, and at the same inputs a newer release sorts higher. `3.15.1+20261021.1.0.5` must sort above `3.15.1+20261021` under **both** dpkg and rpm.
- `vrg-upstream-refresh` never merges, never runs dependency updaters or a release, and refuses agent identity modes (`user`, `audit`).
- `vrg-update-deps` never runs an `[upstream-refresh]` hook, and `vrg-upstream-refresh` never runs a `[dependency-update]` hook.
- The refresh branch is `upstream-refresh`, never `deps/update-*`.
- The vergil-python hook leaves a series PBS no longer ships at its pin, with the note `vergil-python<x.y>: no longer built upstream (latest PBS <tag> ships no <x.y> patch); kept at <cpython>+<pbs_tag>`. It never searches older PBS releases.
- Leading edge only: add a window version only when it is newer than the newest listed series and the latest PBS release ships it for both arches. Never back-fill, never remove.
- `packages.toml` `[retention]` becomes `keep = 1` (`lines` stays 2) before the first two-series vergil-python release.
- All git goes through `vrg-git`, commits through `vrg-commit`, and validation through `vrg-container-run -- vrg-validate`. Agents stop at `vrg-pr-workflow report-ready`. Releases are human-gated preconditions, attested in deployment tasks and never cut by an agent.

## Review Focus

1. **A single-package repo after T1–T4** (vergil-tooling itself, `packages`). Matrix ids, the manifest's artifact list, `planned()` identities and the release wait must be byte-for-byte what they were. Tests: `test_single_package_cells_unchanged` (T3) and `test_single_package_identities_unchanged` (T2).
2. **Two entries where one name is a dash-prefix of the other** (`foo`, `foo-bar`). Install-test and index matching must each pick exactly their own files. Tests: `test_select_artifact_exact_name` (T3) and `test_match_exact_name_prefix` (T4).
3. **A hook that leaves untracked files behind.** `vrg-upstream-refresh` must fail with a message naming them, not commit them. Test: `test_untracked_files_refused` (T6).
4. **An open refresh PR whose content already matches the new run.** No force-push and no comment, so the PR doesn't churn every night. Test: `test_open_pr_same_content_no_push` (T6).
5. **One series' PBS asset missing or its checksum unreadable.** The hook exits non-zero and writes neither file, so there is no partial rewrite across series. Test: `test_one_series_failure_writes_nothing` (P1).

---

## Epic task map

File each task in the repo where its PR lands, under `vergil-project/.github#472` (`vrg-issue-create --epic vergil-project/.github#472 --repo …`):

| Row | Plan part | Repo | Kind | Blocked-by |
|---|---|---|---|---|
| T1 | Multi-package config | vergil-tooling | task | — |
| T2 | Version helper, `upstream-version`, identities, served-runtime parser | vergil-tooling | task | T1 |
| T3 | Package dimension: matrix, manifest, build, install-test, check-runtime | vergil-tooling | task | T2 |
| T4 | Index matching and release wait across packages | vergil-tooling | task | T3 |
| T5 | update-deps and version resolution over the packages list | vergil-tooling | task | T1 |
| T6 | `vrg-upstream-refresh` command and `[upstream-refresh]` config | vergil-tooling | task | — |
| A1 | `ops-upstream-refresh.yml` reusable workflow | vergil-actions | task | — |
| K1 | `packages.toml` `keep = 1` plus size projection | packages | task | — |
| D1 | Release vergil-tooling (T1–T6) | vergil-tooling | deployment | T1, T2, T3, T4, T5, T6 |
| D2 | Release vergil-actions (A1), and confirm App `pull-requests: write` on vergil-python and vergil-containers | vergil-actions | deployment | A1 |
| P1 | Series list, per-series `build.sh` and per-series refresh hook | vergil-python | task | D1 |
| P2 | Leading-edge add from the window | vergil-python | task | P1 |
| P3 | Nightly upstream refresh in `ops.yml` | vergil-python | task | P1, D2 |
| C1 | `bump-tools` on upstream refresh | vergil-containers | task | D1, D2 |
| D3 | Release vergil-python (first two-series release) | vergil-python | deployment | P1, P2, K1 |
| V1 | End-to-end validation | vergil-python | validation | D3, P3, C1 |

D1, D2 and D3 are human-run releases. D3's precondition, attested by the human, is that K1 has merged to `packages` develop, which is where `publish-index.yml` runs. Otherwise the first two-series release could push the index over `SIZE_FAIL`.

---

## Part T — vergil-tooling

### Task T1: Multi-package config

**Files:**

- Modify: `src/vergil_tooling/lib/config.py` (`_parse_package_config` ~837, `VergilConfig` ~399, repo checks ~1164)
- Modify every `cfg.package` / `config.package` reader that is not migrated in T2–T5, so it uses `packages`: `lib/ci_evidence.py:785`, `lib/github_config.py:727`, `lib/repo_init.py` (package presence), `lib/release/tracking.py:136`
- Modify: `docs/site/docs/reference/package-config.md` (the "Structure" section: `[[package]]`, its rules, and the overlay restriction)
- Test: `tests/vergil_tooling/test_config_package.py` (new, or extend the existing package-config test module)

**Interfaces:**

- Produces:
  - `VergilConfig.packages: tuple[PackageConfig, ...]`, which is empty when there is no `[package]`.
  - `VergilConfig.package` becomes a property. It returns `None` for zero entries and the sole entry for one. For more than one, it raises `ConfigError("vergil.toml declares <n> [[package]] entries; read cfg.packages")`.
  - `config.single_vendor(cfg: VergilConfig) -> str | None` returns the shared vendor (`None` with no packages).

- [ ] **Step 1: Write the failing tests:**
  - `test_single_table_is_one_entry`: `[package]` gives `len(cfg.packages) == 1`, and `cfg.package is cfg.packages[0]`.
  - `test_array_of_tables_parses`: two `[[package]]` staged entries named `a` and `b` give `[p.name for p in cfg.packages] == ["a", "b"]`.
  - `test_multi_requires_names`: a second entry without `name` raises `ConfigError` matching `"every [[package]] entry must set name"`.
  - `test_multi_unique_names`: a duplicate `name` raises a message matching `"duplicate [[package]] name 'a'"`.
  - `test_multi_single_vendor`: vendors `vergil` and `other` raise a message matching `"all [[package]] entries must share one vendor"`.
  - `test_multi_refuses_overlay`: two entries plus `packaging/nfpm.overlay.yaml` raise a message matching `"packaging/nfpm.overlay.yaml applies only to a single [package]"`.
  - `test_package_property_refuses_multi`: two entries, accessing `cfg.package`, raises `ConfigError` matching `"read cfg.packages"`.
  - `test_no_package_empty`: no table gives `cfg.packages == ()`, and `cfg.package is None`.
- [ ] **Step 2: Run them and confirm they fail.** Run `vrg-container-run -- uv run pytest tests/vergil_tooling/test_config_package.py -v`. Expected: FAIL.
- [ ] **Step 3: Implement.**
  - `[package]` may be a table or a list of tables. Each one goes through the existing `_parse_package_config`.
  - The multi-entry rules run after parsing.
  - `_check_package_overlay` and `_check_package_python_lock` run per entry. The overlay check refuses when there is more than one entry.
  - Migrate the listed readers: presence checks become `bool(cfg.packages)`, and `tracking.py` uses `single_vendor`.
- [ ] **Step 4: Run them and confirm they pass.** Then run `vrg-container-run -- uv run pytest tests/vergil_tooling -q`. Expected: all pass. Any caller that still reads `.package` on a multi-entry fixture fails loudly here.
- [ ] **Step 5: Update the `package-config.md` Structure section.**
- [ ] **Step 6: Validate.** Run `vrg-container-run -- vrg-validate`. Expected: green, 100% coverage.
- [ ] **Step 7: Commit.** `vrg-commit --type feat --scope package --message "accept [[package]] entries for several packages per repo"`

### Task T2: One version helper, `upstream-version`, identities, served-runtime parser

**Files:**

- Modify: `src/vergil_tooling/lib/config.py` (new `upstream-version` key on `PackageConfig`)
- Modify: `src/vergil_tooling/lib/package/naming.py` (new `package_version`)
- Modify: `src/vergil_tooling/lib/package/build.py:178`, `lib/package/index/published.py:65-73,118-122` (`planned`, refusal advice) and `lib/release/package_index.py:329`, so each uses the helper
- Modify: `src/vergil_tooling/lib/update_deps/vergil_index.py:47` (`_SERVED_RE`)
- Modify: `docs/site/docs/reference/package-config.md` (keys table: `upstream-version`)
- Test: `tests/vergil_tooling/test_package_naming.py`, `tests/vergil_tooling/test_package_published.py`, `tests/vergil_tooling/test_update_deps_vergil_index.py`, `tests/vergil_tooling/test_package_version_order.py` (new)

**Interfaces:**

- Consumes: T1 `cfg.packages`.
- Produces:
  - `PackageConfig.upstream_version: str | None` (TOML `upstream-version`). It matches the same pattern as `version`, and is mutually exclusive with `version`: `ConfigError` `"[package] sets both version and upstream-version"`.
  - `naming.package_version(pkg: PackageConfig, repo_version: str) -> str`:
    - `f"{pkg.upstream_version}.{repo_version}"` when `upstream_version` is set;
    - otherwise `pkg.version` when set;
    - otherwise `repo_version`.
  - `published.planned(repo_root: Path, cfg: VergilConfig, version: str) -> list[Identity]` covers every entry, in entry order, using `package_version`.
  - `published.check_release(repo_root, cfg, product, version, gh=...)` now takes the config, not one `PackageConfig`. Update both callers: `bin/vrg_package.py:312` and `lib/release/preflight.py:603`.

- [ ] **Step 1: Write the failing tests:**
  - `test_package_version_upstream_join`: `upstream_version="3.15.1+20261021"`, repo `1.0.5` gives `"3.15.1+20261021.1.0.5"`.
  - `test_package_version_override_and_default`: `version="2.0"` gives `"2.0"`, and neither key set gives the repo version.
  - `test_upstream_version_and_version_exclusive`: `ConfigError`.
  - `test_single_package_identities_unchanged`: the vergil-tooling-shaped config gives the same `planned()` list as before the change. Pin the expected identities literally.
  - `test_planned_multi_entries`: two staged entries with `upstream-version` give identities for both names, each at `<inputs>.1.0.5-1`.
  - `test_served_parser_accepts_release_suffix`: `_served_key("3.15.1+20261021.1.0.5", …)` orders above `_served_key("3.15.1+20261021", …)`. It still raises on `"3.15.1"`.
  - `test_version_order_dpkg` and `test_version_order_rpm` (skip with a reason when `dpkg`/`rpm` isn't installed; the dev container must have both, so CI runs them):
    - `3.15.1+20261021.1.0.5` > `3.15.1+20261021`
    - `3.15.1+20261021.1.0.10` > `3.15.1+20261021.1.0.9`
    - `3.15.2+20261101.1.0.5` > `3.15.1+20261021.1.0.6`
    - use `dpkg --compare-versions` and `rpmdev-vercmp` (or `rpm --eval '%{lua: print(rpm.vercmp(...))}'`) with `-1` revisions appended.
- [ ] **Step 2: Run them and confirm they fail.** Run `vrg-container-run -- uv run pytest tests/vergil_tooling/test_package_naming.py tests/vergil_tooling/test_package_published.py tests/vergil_tooling/test_update_deps_vergil_index.py tests/vergil_tooling/test_package_version_order.py -v`. Expected: FAIL.
- [ ] **Step 3: Implement.**
  - Replace the three `pkg.version or version` sites with `naming.package_version`.
  - `_SERVED_RE` becomes `^(\d+\.\d+\.\d+)\+(\d+)(?:\.\d+\.\d+\.\d+)?$`. `_served_key` orders by `(patch, pbs, release)`, with a missing release as `()`.
  - Rewrite the `check_unpublished` advice to read: "bump the package version: a new upstream-version, or a new repo release for an upstream-version package".
- [ ] **Step 4: Run them and confirm they pass.** Same command. Expected: PASS. If the dev image lacks `rpm` or `dpkg`, the order tests skip, which is not acceptable. In that case, file a vergil-containers task to add the missing tool, and say so in the PR notes.
- [ ] **Step 5: Validate.** Run `vrg-container-run -- vrg-validate`. Expected: green.
- [ ] **Step 6: Commit.** `vrg-commit --type feat --scope package --message "compute every package version through one helper, with upstream-version"`

### Task T3: Package dimension in matrix, manifest, build, install-test, check-runtime

**Files:**

- Modify: `src/vergil_tooling/lib/package/matrix.py` (`BuildCell`, `TestCell`, `resolve`, `to_json`, `manifest`)
- Modify: `src/vergil_tooling/lib/package/build.py` (`run_build`), `lib/package/staged.py` (env), `lib/package/install_test.py` (`select_artifact`, `run_install_test`), `lib/package/runtime.py` (`check_consistent` per name)
- Modify: `src/vergil_tooling/bin/vrg_package.py` (`_cmd_matrix`, `_cmd_check_runtime`, the install-test command)
- Modify: `docs/site/docs/reference/package-config.md` (matrix ids, the staged env vars `VRG_PACKAGE_NAME`/`VRG_PACKAGE_VERSION`, the manifest shape, and the "One runtime per release" section, which becomes per package)
- Test: `tests/vergil_tooling/test_package_matrix.py`, `test_package_build.py`, `test_package_staged.py`, `test_package_install_test.py`, `test_vrg_package.py`

**Interfaces:**

- Consumes: T1 `cfg.packages` and T2 `naming.package_version`.
- Produces:
  - `BuildCell.package: str` and `TestCell.package: str`, the package name.
  - `matrix.resolve_all(repo_root: Path, packages: Sequence[PackageConfig], *, tier: Tier = Tier.FULL) -> Matrix`:
    - one package: ids exactly as today;
    - several: every id is `f"{name}--{id}"` (a double dash, which can't occur in a valid id today).
  - `matrix.to_json(m: Matrix, *, runtimes: Mapping[str, str | None]) -> dict`: each build cell carries `"runtime": runtimes[cell.package]` when that value isn't `None`.
  - `matrix.manifest(m: Matrix, *, runtimes: Mapping[str, str | None]) -> dict`:
    - `{"artifacts": [{"fmt", "arch", "suite", "name"}], "runtimes": {name: runtime}}`;
    - for a single package, keep the top-level `"runtime"` key as today, alongside the new keys.
  - Staged build env adds `VRG_PACKAGE_NAME` (the name) and `VRG_PACKAGE_VERSION` (the full version, from `package_version`).
  - `runtime.check_consistent_by_name(runtimes: Mapping[str, str | None], packages: list[tuple[str, str, tuple[str, ...]]], where: str) -> None`. The tuple is (filename, package name, depends). A name whose runtime is `None` (staged) is not checked. A file whose name isn't in `runtimes` raises.

- [ ] **Step 1: Write the failing tests:**
  - `test_single_package_cells_unchanged`: the vergil-tooling-shaped config gives build ids `["shared-amd64", "shared-arm64"]` and the test ids it has today (pin them literally), each with `package == "vergil-tooling"`.
  - `test_multi_package_cells_qualified`: two staged entries `vergil-python3.14` and `vergil-python3.15` give build ids `vergil-python3.14--shared-amd64`, …, with no duplicates.
  - `test_manifest_names_artifacts`: the multi manifest has every artifact carrying `name`, and `runtimes == {"vergil-python3.14": None, "vergil-python3.15": None}`.
  - `test_run_build_selects_entry_by_cell`: `run_build(cell_id="vergil-python3.15--shared-amd64")` builds with `ctx.name == "vergil-python3.15"`.
  - `test_staged_env_name_and_version`: the build command sees `VRG_PACKAGE_NAME=vergil-python3.15` and `VRG_PACKAGE_VERSION=3.15.1+20261021.1.0.5`. Assert it by having the fixture command echo them into a file.
  - `test_select_artifact_exact_name`: files for `foo` and `foo-bar` in one dir. `select_artifact(…, "foo", …)` returns only `foo`'s rpm and deb.
  - `test_install_test_uses_entry_smoke`: the multi entries' test cell runs that entry's `smoke`.
  - `test_check_runtime_by_name`: a staged-only manifest passes; a python package whose depends name another series raises `PackageError`.
- [ ] **Step 2: Run them and confirm they fail.** Run `vrg-container-run -- uv run pytest tests/vergil_tooling/test_package_matrix.py tests/vergil_tooling/test_package_build.py tests/vergil_tooling/test_package_staged.py tests/vergil_tooling/test_package_install_test.py tests/vergil_tooling/test_vrg_package.py -v`. Expected: FAIL.
- [ ] **Step 3: Implement.**
  - `_cmd_matrix` resolves one runtime per python-builder entry with `python_builder.release_runtime`, and passes `runtimes`.
  - `run_build` and `run_install_test` find the cell across `resolve_all`, then pick the entry by `cell.package`.
  - `select_artifact` filters candidates by `published.parse_filename(f).name == name`.
  - `_cmd_check_runtime` reads `runtimes` when present, falling back to the top-level `runtime` for old manifests, and calls `check_consistent_by_name`.
- [ ] **Step 4: Run them and confirm they pass.** Same command. Expected: PASS.
- [ ] **Step 5: Check the vergil-actions workflows.** In `vergil-actions/.github/workflows/ci-package.yml` and `cd-release.yml`, confirm that cells are used only through `matrix.id` / `--cell` and the per-cell `runtime`. If any step assumes one package (for example one manifest runtime, or a fixed artifact name), file a vergil-actions task under the epic, blocked by T3, and list it in the PR notes.
- [ ] **Step 6: Update the `package-config.md` sections.**
- [ ] **Step 7: Validate.** Run `vrg-container-run -- vrg-validate`. Expected: green.
- [ ] **Step 8: Commit.** `vrg-commit --type feat --scope package --message "thread the package name through matrix, build, install-test and runtime checks"`

### Task T4: Index matching and release wait across packages

**Files:**

- Modify: `src/vergil_tooling/lib/package/index/collect.py` (`ManifestEntry`, `_manifest_entries`, `pattern`, `_match`)
- Modify: `src/vergil_tooling/lib/release/package_index.py` (`_wait`, `_pick_target`, `_revision`), `lib/release/context.py` (`package_index_url`)
- Test: `tests/vergil_tooling/test_package_index_collect.py`, `tests/vergil_tooling/test_release_package_index.py`

**Interfaces:**

- Consumes: T3's manifest shape, and T2's `package_version`.
- Produces:
  - `ManifestEntry.name: str | None`, which is `None` for old manifests. With a name, `pattern()` matches `re.escape(name)` exactly.
  - `ReleaseContext.package_index_urls: tuple[str, ...]` replaces `package_index_url`; update its readers. The wait succeeds only when every entry's identity is visible.

- [ ] **Step 1: Write the failing tests:**
  - `test_match_two_packages_same_arch`: a new manifest with two names and four files per name pairs each entry with exactly its own file. There's no "has 2 packages" error.
  - `test_match_exact_name_prefix`: names `foo` and `foo-bar` each match only their own files.
  - `test_old_manifest_still_matches`: a manifest without `name` keys and a single package matches as today.
  - `test_wait_all_packages_visible`: two packages, where the index serves only one. The wait keeps polling and doesn't succeed, and once both are served it records two URLs.
- [ ] **Step 2: Run them and confirm they fail.** Run `vrg-container-run -- uv run pytest tests/vergil_tooling/test_package_index_collect.py tests/vergil_tooling/test_release_package_index.py -v`. Expected: FAIL.
- [ ] **Step 3: Implement.** `_wait` loops over `cfg.packages`, using `package_version(pkg, ctx.version)` and each entry's first deb target.
- [ ] **Step 4: Run them and confirm they pass.** Same command. Expected: PASS.
- [ ] **Step 5: Validate.** Run `vrg-container-run -- vrg-validate`. Expected: green.
- [ ] **Step 6: Commit.** `vrg-commit --type feat --scope package-index --message "match index artifacts and wait on every package by name"`

### Task T5: update-deps and version resolution over the packages list

**Files:**

- Modify: `src/vergil_tooling/lib/update_deps/vergil_index.py` (`served_runtime_for`, `runtime_indexes`), `lib/update_deps/updaters/vergil_eco.py` (`_declared_runtime`, `bump_runtime`), `lib/versions/extensions/python.py:84,109`
- Test: `tests/vergil_tooling/test_update_deps_vergil_eco.py`, `tests/vergil_tooling/test_versions_python_extension.py`

**Interfaces:**

- Consumes: T1 `cfg.packages`.
- Produces: `python_packages(cfg: VergilConfig) -> tuple[PackageConfig, ...]` (in `lib/package/runtime.py`), the python-builder entries. Every reader of a runtime pin goes through it.
  - None: behaviour as when there is no package today.
  - One: exactly today's behaviour.
  - More than one python-builder entry **with a pin**: `UpdateDepsError` / `VersionsError` `"several [[package]] entries pin a runtime; pin it on one entry"`. That case is deliberately left unsupported until someone needs it.

- [ ] **Step 1: Write the failing tests:**
  - `test_staged_only_multi_has_no_runtime`: two staged entries make the vergil updater do nothing runtime-related.
  - `test_single_python_pin_unchanged`: today's pinned fixture behaves identically.
  - `test_two_pins_refused`: the error message above.
  - `test_resolution_narrows_with_python_entry`: the extension narrows to `runtime_published` when any python-builder entry exists.
- [ ] **Step 2: Run them and confirm they fail.** Run `vrg-container-run -- uv run pytest tests/vergil_tooling/test_update_deps_vergil_eco.py tests/vergil_tooling/test_versions_python_extension.py -v`. Expected: FAIL.
- [ ] **Step 3: Implement** through `python_packages`.
- [ ] **Step 4: Run them and confirm they pass.** Same command. Expected: PASS.
- [ ] **Step 5: Validate.** Run `vrg-container-run -- vrg-validate`. Expected: green.
- [ ] **Step 6: Commit.** `vrg-commit --type feat --scope update-deps --message "read runtime pins from the packages list"`

### Task T6: `vrg-upstream-refresh`

**Files:**

- Modify: `src/vergil_tooling/lib/config.py` (`[upstream-refresh]`)
- Create: `src/vergil_tooling/lib/upstream_refresh.py`
- Create: `src/vergil_tooling/bin/vrg_upstream_refresh.py`
- Modify: `pyproject.toml` (`[project.scripts]`: `vrg-upstream-refresh = "vergil_tooling.bin.vrg_upstream_refresh:main"`)
- Create: `docs/site/docs/reference/upstream-refresh-config.md`. Add a row to `docs/site/docs/reference/cli-tools-overview.md`, add a nav entry if the nav is explicit, and add one line to `dependency-update-config.md` saying the two hooks are separate.
- Test: `tests/vergil_tooling/test_upstream_refresh.py`, `tests/vergil_tooling/test_vrg_upstream_refresh.py`

**Interfaces:**

- Produces (config): `UpstreamRefreshConfig(hook: str)` and `VergilConfig.upstream_refresh: UpstreamRefreshConfig | None`. `hook` is a required non-empty string, and unknown keys are refused.
- Produces (lib):
  - `REFRESH_BRANCH = "upstream-refresh"`, `REPORT_ENV = "VRG_UPSTREAM_REPORT"`, `COMMIT_SUBJECT = "chore(upstream): refresh upstream inputs"`.
  - `run_hook(repo_root: Path, hook: str, report: Path) -> list[str]`:
    - runs `bash -c hook` from the root, with `REPORT_ENV=<report>`;
    - returns the report's non-empty lines (the notes);
    - non-zero exit raises `UpstreamRefreshError(f"[upstream-refresh].hook failed (exit {n}): {hook}")`.
  - `changed_paths(repo_root: Path) -> list[str]`: modified tracked files. Untracked files raise `UpstreamRefreshError("hook left untracked files: <paths>")`.
  - `render_body(notes: list[str], paths: list[str]) -> str`: a fixed intro ("Automated upstream refresh. A human reviews and merges; nothing auto-merges."), then `## Changed files` and `## Notes` as bullets. The notes section is omitted when there are none.
  - `sync(repo: str, base: str, repo_root: Path, notes: list[str], paths: list[str]) -> str` returns one of `"opened"`, `"refreshed"`, `"unchanged"`, `"closed"` or `"nothing"`. Its rules are in Step 3.
- Produces (CLI): `vrg-upstream-refresh [--repo OWNER/NAME] [--base BRANCH] [--dry-run]`.
  - `--repo` defaults to `GITHUB_REPOSITORY`, then the origin remote. `--base` defaults to the repo's default branch.
  - Exit 0 on any outcome, 1 on error, and 1 in an agent identity mode, with `vrg-upstream-refresh opens PRs and is not run by agents`.
  - `--dry-run` runs the hook and prints the body without pushing.

- [ ] **Step 1: Write the failing lib tests.** Use a temp git repo, and patch `lib.github` calls:
  - `test_no_change_no_pr_exits_quietly`: the hook edits nothing, there's no open PR, and the result is `"nothing"` with no GitHub write calls.
  - `test_no_change_closes_stale_pr`: no change, and an open PR on `upstream-refresh`, gives `"closed"`. The PR is closed with comment `Nothing to refresh on <base>; closing.` and its branch is deleted.
  - `test_change_opens_pr`: one modified file and no PR give `"opened"`. One commit with `COMMIT_SUBJECT` is pushed to `upstream-refresh`, and the PR title equals `COMMIT_SUBJECT`.
  - `test_change_refreshes_pr`: an open PR with different content gives `"refreshed"`: a force-push, the body updated, and one comment.
  - `test_open_pr_same_content_no_push`: an open PR whose head tree equals the new commit's tree gives `"unchanged"`, with no push and no comment.
  - `test_untracked_files_refused`: the hook creates `junk.txt`, and `UpstreamRefreshError` names it.
  - `test_hook_failure_raises`: exit 3 gives a message containing `exit 3`.
  - `test_notes_in_body`: two report lines appear as two bullets under `## Notes`.
- [ ] **Step 2: Run them and confirm they fail.** Run `vrg-container-run -- uv run pytest tests/vergil_tooling/test_upstream_refresh.py -v`. Expected: FAIL.
- [ ] **Step 3: Implement `lib/upstream_refresh.py`.** Git goes through `lib/git.py`, and GitHub through `lib/github.py`. Rules:
  - The repo must be clean on `base` before the hook runs.
  - With changes, the commit is on a detached HEAD from `base`.
  - Compare the trees of the new commit and the existing `origin/upstream-refresh` (`git rev-parse <ref>^{tree}`) to decide between `refreshed` and `unchanged`.
  - Push with `--force-with-lease` to `refs/heads/upstream-refresh`.
- [ ] **Step 4: Run them and confirm they pass.** Same command. Expected: PASS.
- [ ] **Step 5: Write the failing CLI and config tests:**
  - `test_cli_refuses_agent_mode` (patch `identity_mode` to USER): exit 1 and the message above.
  - `test_cli_requires_config`: no `[upstream-refresh]` gives exit 1 with `vergil.toml has no [upstream-refresh] section`.
  - `test_cli_dry_run_no_push`.
  - `test_config_upstream_refresh_parses` and `test_config_upstream_refresh_unknown_key`.
- [ ] **Step 6: Run them, confirm they fail, implement the CLI and config, then confirm they pass.** Run `vrg-container-run -- uv run pytest tests/vergil_tooling/test_vrg_upstream_refresh.py tests/vergil_tooling/test_config_package.py -v`.
- [ ] **Step 7: Write the reference page:**
  - purpose, and how upstream refresh differs from a dependency update;
  - the hook contract (edits tracked files only, `VRG_UPSTREAM_REPORT` notes, exit code);
  - the outcomes;
  - the identity rule;
  - the reusable workflow (A1).
- [ ] **Step 8: Validate.** Run `vrg-container-run -- vrg-validate`. Expected: green.
- [ ] **Step 9: Commit.** `vrg-commit --type feat --scope upstream-refresh --message "add vrg-upstream-refresh: run a repo's refresh hook and open one PR"`

---

## Part A — vergil-actions

### Task A1: `ops-upstream-refresh.yml`

**Files:**

- Create: `.github/workflows/ops-upstream-refresh.yml`
- Modify: the vergil-actions docs page that lists the ops workflows (next to `ops-epic-sweep.yml`)

**Interfaces:**

- Produces: a reusable workflow, `on: workflow_call`, with secrets `APP_CLIENT_ID` and `APP_PRIVATE_KEY` (required) and no inputs. One job:
  - `permissions: contents: write, pull-requests: write`;
  - `concurrency: group: upstream-refresh, cancel-in-progress: false`;
  - `container: image: ghcr.io/vergil-project/prod-base:latest`.

- [ ] **Step 1: Write the workflow.** Follow `ops-epic-sweep.yml` for the token and the vergil-tooling install, and vergil-containers' `bump-tools.yml` for the checkout. The steps:
  1. `actions/create-github-app-token@v3`.
  2. `actions/checkout@v6` with the token.
  3. `git config --global --add safe.directory "$GITHUB_WORKSPACE"`.
  4. Set the git identity to `github-actions[bot]` / `41898282+github-actions[bot]@users.noreply.github.com`, as `bump-tools.yml` does.
  5. Install vergil-tooling exactly as `ops-epic-sweep.yml` does.
  6. Run `vrg-upstream-refresh --base "$BASE"`, with `GH_TOKEN` set to the App token and `BASE: ${{ github.event.repository.default_branch }}` passed through env, never inline.
- [ ] **Step 2: Validate.** Run `vrg-container-run -- vrg-validate`. Expected: green; actionlint is clean.
- [ ] **Step 3: Commit.** `vrg-commit --type feat --scope ops --message "add the ops-upstream-refresh reusable workflow"`

---

## Part K — packages

### Task K1: `keep = 1` and a size projection

**Files:**

- Modify: `packages.toml` (`[retention]`: `keep = 1`, update the comment, keep `lines = 2`)

- [ ] **Step 1: Make the edit.**
- [ ] **Step 2: Measure.**
  - Sum the `.deb`/`.rpm` asset sizes of the newest release of each product with `vrg-gh release view <tag> --repo <product> --json assets`.
  - Project the worst case as: 2 lines × 1 release × (vergil-python's per-series size × 2 series), plus vergil-tooling × 2 lines, plus `packages`.
  - Put the figures and the result against `SIZE_WARN` (750 MB) and `SIZE_FAIL` (900 MB) in the PR notes. If the projection is over `SIZE_WARN`, say so prominently.
- [ ] **Step 3: Validate.** Run `vrg-container-run -- vrg-validate`. Expected: green, and `vrg-package index --check-config` passes if validation runs it.
- [ ] **Step 4: Commit.** `vrg-commit --type chore --scope index --message "keep one release per line in the package index"`

---

## Part P — vergil-python

### Task P1: Series list, per-series build and refresh hook

**Files:**

- Modify: `runtime.toml` (a list of `[[series]]` tables: `series`, `cpython`, `pbs_tag`, `[series.sha256]` with `x86_64`/`aarch64`)
- Modify: `vergil.toml`:
  - two `[[package]]` entries, `vergil-python3.14` and `vergil-python3.15`, each with `builder = "staged"`, `vendor`, `upstream-version`, `summary`, `smoke` and `[package.staged] build-command = "packaging/build.sh"`;
  - remove `[dependency-update]`;
  - add `[upstream-refresh] hook = "scripts/dev/upstream-refresh.sh"`.
- Modify: `packaging/build.sh` (select the series by `VRG_PACKAGE_NAME`; per-entry guards)
- Move: `scripts/dev/update_deps.py` → `scripts/dev/upstream_refresh.py`, `scripts/dev/update-deps.sh` → `scripts/dev/upstream-refresh.sh`, `tests/test_update_deps.py` → `tests/test_upstream_refresh.py`, and `tests/check-update-deps.sh` → `tests/check-upstream-refresh.sh` (with `vrg-git mv`)
- Modify: `tests/test_build.py`
- Modify: `docs/runtime-package-pattern.md` (one package per listed series; the patch floats within each; upstream refresh replaces the dependency-update hook; add and remove; the version format)
- Modify: `CLAUDE.md` if it names the old hook

**Interfaces:**

- Consumes:
  - D1: vergil-tooling with `[[package]]`, `upstream-version` and `[upstream-refresh]`, plus `VRG_PACKAGE_NAME`/`VRG_PACKAGE_VERSION` in the staged build env.
  - The hook contract from T6 (`VRG_UPSTREAM_REPORT`).
- Produces (in `scripts/dev/upstream_refresh.py`, stdlib only):
  - `load_series(text: str) -> list[dict]`: the parsed `[[series]]` entries, validated with `check_series`.
  - `refresh(root: Path, fetch: Fetch) -> list[str]`: rewrites both files and returns the notes.
  - `render_runtime(entries: list[dict]) -> str`.
  - `rewrite_packages(text: str, entries: list[dict]) -> str`: rewrites each `[[package]]` whose `name` is `vergil-python<series>` (`upstream-version` and `summary`), and checks `smoke`.
  - `main(*, root: Path, fetch: Fetch = fetch) -> int`.

- [ ] **Step 1: Write the failing hook tests** in `tests/test_upstream_refresh.py` (unittest, offline, fixture SHA256SUMS through an injected `fetch`):
  - `test_refreshes_each_series_independently`: the latest PBS ships 3.14.9 and 3.15.2. Both entries move to them, and the checksums and `upstream-version` change for each.
  - `test_series_no_longer_built_keeps_pin_with_note`: the latest PBS ships only 3.15.x. The 3.14 entry is unchanged, and the notes contain the exact "no longer built upstream" line from Global Constraints.
  - `test_one_series_failure_writes_nothing`: the 3.15 aarch64 checksum is missing. `main` exits non-zero, and both files are byte-identical to before.
  - `test_package_series_mismatch_fails`: a `[[package]]` exists for 3.13 with no `[[series]]` entry, so the hook fails before writing.
  - `test_never_changes_repo_version`: `VERSION` is untouched, and no `upstream-version` contains it.
- [ ] **Step 2: Write the failing build tests** in `tests/test_build.py`:
  - `test_stages_requested_series`: with `VRG_PACKAGE_NAME=vergil-python3.14`, the result is under `/opt/vergil/python/3.14/`. Use the existing fixture approach for the download.
  - `test_unknown_package_name_fails`: `VRG_PACKAGE_NAME=vergil-python3.13` makes build.sh exit non-zero, with a message naming the missing series.
  - `test_version_must_carry_inputs`: `VRG_PACKAGE_VERSION=3.14.9+20261101.1.0.5` while `runtime.toml` says `3.14.8+20261013` makes build.sh fail.
  - `test_every_series_has_one_package`: a series with no `[[package]]`, or the reverse, fails.
- [ ] **Step 3: Run them and confirm they fail.** Run `vrg-container-run -- bash tests/check-upstream-refresh.sh` and `vrg-container-run -- bash tests/check-build.sh`. Expected: FAIL.
- [ ] **Step 4: Implement.**
  - `build.sh`'s `read_toml` helper gains a mode that selects the `[[series]]` and `[[package]]` entries by series. The guards compare:
    - name with `vergil-python<series>`;
    - `upstream-version` with `<cpython>+<pbs_tag>`;
    - `VRG_PACKAGE_VERSION`, which must start with `<cpython>+<pbs_tag>.`;
    - smoke with the interpreter path.
  - The hook keeps the existing `/releases/latest` lookup and `SHA256SUMS` parsing, applied per series. It reads the series' current pin before rewriting anything, and writes notes to `$VRG_UPSTREAM_REPORT`.
- [ ] **Step 5: Run them and confirm they pass.** Same commands. Expected: PASS.
- [ ] **Step 6: Seed the 3.14 entry.**
  - Write the 3.14 `[[series]]` at `3.14.8` / `20261013`, with that release's checksums from `https://github.com/astral-sh/python-build-standalone/releases/download/20261013/SHA256SUMS`.
  - Run `vrg-container-run -- env VRG_UPSTREAM_REPORT=/tmp/notes bash scripts/dev/upstream-refresh.sh` to bring both series current. Commit the result.
- [ ] **Step 7: Update `docs/runtime-package-pattern.md`.** Remove "one series at a time", the "frozen" paragraph and the pointer to the archived planning issue.
- [ ] **Step 8: Validate.** Run `vrg-container-run -- vrg-validate`. Expected: green. The PR's CI builds and install-tests both series, so check that CI ran four build cells.
- [ ] **Step 9: Commit.** `vrg-commit --type feat --scope runtime --message "build one package per listed series, refreshed by an upstream-refresh hook"`

### Task P2: Leading-edge add from the window

**Files:**

- Modify: `scripts/dev/upstream_refresh.py`
- Modify: `tests/test_upstream_refresh.py`
- Modify: `docs/runtime-package-pattern.md` (the add section)

**Interfaces:**

- Consumes: P1's `refresh`, `render_runtime` and `rewrite_packages`.
- Produces:
  - `WINDOW_URL = "https://raw.githubusercontent.com/vergil-project/vergil-containers/{ref}/docker/versions/window.json"` and `WINDOW_ENV = "VRG_VERSION_WINDOW"`.
  - `load_window(root: Path, fetch: Fetch) -> list[str]`:
    - reads the `in_policy` versions of `languages.python.families[key == "python"].versions`;
    - `ref` comes from `vergil.toml` `[dependencies].vergil`;
    - `WINDOW_ENV`, when set, is a local path read instead;
    - `schema != 1`, or a missing key, raises through `_die`.
  - `leading_edge_candidates(listed: list[str], window: list[str]) -> list[str]`: the window versions newer than the newest listed series, oldest first.
  - `new_entry(series: str, cpython: str, tag: str, sums: dict[str, str]) -> tuple[dict, str]`: the `[[series]]` dict and the `[[package]]` TOML block. That block's summary and smoke match the existing entries' wording.

- [ ] **Step 1: Write the failing tests:**
  - `test_adds_leading_edge_series`: the fixture window has 3.12–3.16, the listed series are 3.14 and 3.15, and the latest PBS ships 3.16.0. A 3.16 entry is added to both files, and the note `added series 3.16 (CPython 3.16.0, PBS <tag>)` is written.
  - `test_never_backfills`: the same window without 3.16. No entry is added, and 3.12 and 3.13 are never added.
  - `test_window_version_pbs_lacks_is_held_back`: the window has 3.16 but PBS ships no 3.16. Nothing is added, and the note is `held back: series 3.16 is in the window but PBS <tag> does not ship it`.
  - `test_window_env_override`: `VRG_VERSION_WINDOW` points at a fixture, and no network fetch happens.
  - `test_bad_window_schema_fails`: `schema: 2` exits non-zero, and nothing is written.
- [ ] **Step 2: Run them and confirm they fail.** Run `vrg-container-run -- bash tests/check-upstream-refresh.sh`. Expected: FAIL.
- [ ] **Step 3: Implement.** `refresh` calls the add after the per-series refresh, inside the same all-or-nothing write.
- [ ] **Step 4: Run them and confirm they pass.** Same command. Expected: PASS.
- [ ] **Step 5: Validate.** Run `vrg-container-run -- vrg-validate`. Expected: green.
- [ ] **Step 6: Commit.** `vrg-commit --type feat --scope runtime --message "add a new leading-edge series once the version window has it"`

### Task P3: Nightly upstream refresh

**Files:**

- Modify: `.github/workflows/ops.yml` (add a job that calls `vergil-project/vergil-actions/.github/workflows/ops-upstream-refresh.yml@v2.1` with the two App secrets; the file already has a daily schedule)

- [ ] **Step 1: Add the job,** after the existing `ops-github-config` job, on the same schedule.
- [ ] **Step 2: Validate.** Run `vrg-container-run -- vrg-validate`. Expected: green.
- [ ] **Step 3: Commit.** `vrg-commit --type ci --scope ops --message "run the upstream refresh nightly"`

The push of this change touches `.github/workflows/**`, so `report-ready` may report that the agent push is refused. That's expected, and the human's `vrg-submit-pr` pushes it.

---

## Part C — vergil-containers

### Task C1: `bump-tools` on upstream refresh

**Files:**

- Create: `docker/pins/upstream-refresh.sh` (runs `python3 docker/pins/resolve_latest.py --write`, then `python3 docker/pins/generate_catalog.py`; edits files only; writes no notes)
- Modify: `vergil.toml` (`[upstream-refresh] hook = "docker/pins/upstream-refresh.sh"`)
- Modify: `.github/workflows/bump-tools.yml` (replace the job with a call to `ops-upstream-refresh.yml@v2.1`; keep the `cron: '30 6 * * 1'` schedule and `workflow_dispatch`)
- Modify: any doc that describes the bump PR's `bot/bump-tools-*` branch

- [ ] **Step 1: Make the changes.** The inline drift-detection and PR bash is deleted.
- [ ] **Step 2: Check the script by hand.** Run `vrg-container-run -- bash docker/pins/upstream-refresh.sh`, and confirm `git status` shows only modified pin and catalog files (or nothing). Then restore the tree with `vrg-git checkout -- .`.
- [ ] **Step 3: Validate.** Run `vrg-container-run -- vrg-validate`. Expected: green.
- [ ] **Step 4: Commit.** `vrg-commit --type ci --scope bump-tools --message "run the tool bumps through vrg-upstream-refresh"`
- [ ] **Step 5: Put the follow-up in the PR notes.** Once merged, any open `bot/bump-tools-*` PR must be closed by hand, because the new command manages only the `upstream-refresh` branch.

---

## Part D — Deployments (human-gated releases)

Each one is created with `vrg-issue-create --kind deployment …` and run with `issue-deploy`. The release itself is a **human-attested precondition**.

- **D1 — vergil-tooling release.** Precondition: the human has released vergil-tooling with T1–T6 merged, so the `v2.1` tag carries them. Deploy step: confirm that `vrg-upstream-refresh --help` runs from a fresh `uv tool install` at `v2.1`, and that `vrg-package matrix` against vergil-tooling itself prints today's cell ids.
- **D2 — vergil-actions release and App permissions.** Preconditions:
  - the human has released vergil-actions with A1;
  - the human attests that the per-org App installation grants **Contents: write** and **Pull requests: write** on vergil-python and vergil-containers.
  
  Deploy step: confirm `ops-upstream-refresh.yml` exists at `v2.1`.
- **D3 — vergil-python release.** Preconditions:
  - P1 and P2 are merged;
  - K1 is merged to `packages` develop;
  - the human has released vergil-python.
  
  Deploy step: confirm that the index serves `vergil-python3.14` and `vergil-python3.15` at `<inputs>.<release>` versions, with 3.14 at 3.14.9 or later, and that `publish-index` reported no `SIZE_WARN`. If it did warn, record it and file intake.

## Part V — Validation

### V1: End-to-end

Created with `--kind validation`, blocked by D3, P3 and C1, and run with `issue-validate`. The checks:

1. On an agent VM, `apt-get install vergil-python3.14 vergil-python3.15` installs both side by side. Each `/opt/vergil/python/<x.y>/bin/python<x.y> -c 'import ssl; print(ssl.OPENSSL_VERSION)'` runs, and `apt policy` shows the new-format version as the candidate over any old-format one.
2. A manual `workflow_dispatch` of vergil-python's `ops.yml` either opens or refreshes the `upstream-refresh` PR, or exits quietly with "nothing" when there's nothing to refresh. Either outcome is recorded with its run URL.
3. A manual `workflow_dispatch` of vergil-containers' `bump-tools.yml` does the same.
4. A normal `vrg-release` of vergil-python (D3's) didn't touch `runtime.toml`. Its tracking issue shows the `update-deps` stage ran no repo hook.
