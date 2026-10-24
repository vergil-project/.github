# Multi-series runtimes and a fully automated lifecycle: implementation plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** `vergil-python` publishes and patches a runtime for every Python
version in the window. A scheduled, unattended workflow carries every
Python lifecycle change from detection to a published runtime, and alerts a
human only when a gate fails.

**Architecture:**

- **`vergil-tooling`** gains four things:
  - **Multi-package support.** `[[package]]`, a package dimension in the
    matrix and manifest, name-aware index matching, a `build`/`skip`
    classification, and per-package retention.
  - **An `automation` identity**, with an allowlist of workflows and jobs, a
    scope per operation, and a later fail-closed cutover for undeclared
    runners.
  - **An unattended-merge rule** for every PR the automation merges.
  - **`vrg-lifecycle`**, the driver that runs the chain.
- **`vergil-actions`** gains three things:
  - a package dimension in `ci-package.yml` and `cd-release.yml`;
  - declared `automation` in the existing ops workflows;
  - a new reusable `ops-lifecycle.yml`.
- **`vergil-containers`** gains three things:
  - a `merge` switch per language in the manifest;
  - lifecycle commits authored by the App, with the SHAs it pushed recorded;
  - the docker-publish concurrency fix, and the call to `ops-lifecycle.yml`.
- **`vergil-python`** gains three things:
  - a `[[series]]` list in `runtime.toml`;
  - a `[[package]]` per series in `vergil.toml`;
  - a hook that reconciles the series list against the released window,
    which `vrg-update-deps` now passes it.

**Tech Stack:**

- **`vergil-tooling`:** Python 3.12+, `pytest`, 100% branch coverage.
- **`vergil-actions`:** workflow YAML, with `actions/**/tests/*.test.sh`
  harnesses run by `smoke-*.yml`; actionlint runs through `vrg-validate`.
- **`vergil-containers`:** the stdlib + PyYAML scripts in `docker/versions/`,
  tested with `uv run --with pytest --with pyyaml python -m pytest -q`.
- **`vergil-python`:** bash plus stdlib Python, tested with `unittest` via
  `scripts/bin/validate-custom`.

**Spec:** `epics/443-multi-runtime-lifecycle/spec.md`, in the same directory.
Read it before starting any task. `§N` references point into it.

## Global Constraints

- **Validation:** run exactly `vrg-container-run -- vrg-validate`, from the
  task's worktree. For the TDD loop only, run focused tests:
  - `vergil-tooling`: `vrg-container-run -- uv run pytest <path> -v`;
  - `vergil-containers`: `uv run --with pytest --with pyyaml python -m pytest -q`
    in `docker/versions/`;
  - `vergil-python`: `uv run --no-project python3 -m unittest discover -s tests -p 'test_<x>.py' -v`.
- **Coverage:** `vergil-tooling` enforces 100% branch coverage.
- **Tools:** use `vrg-git` and `vrg-gh` only. Commit with `vrg-commit --type
  <t> --scope <s> --message <m>`.
- **Placement:** one PR per task, in the repo named in the task heading. No PR
  `Closes` an issue in another repo.
- **Fail loud** (§3 principle 6, §7.3). No skip, deferral, pending state or
  fallback may be silent. Each one is printed, written to
  `$GITHUB_STEP_SUMMARY` when that is set, and tracked as an issue past its
  bound.
- **No Python-specific branches in shared code** (§3 principle 8). That covers
  multi-package support, the automation identity, the merge rule and the
  driver. Python specifics live in `vergil-python`'s hook and in the
  `vergil-containers` manifest.
- **Single-package repos must not change** (§7.3). Their config, cell ids,
  manifests, overlay path and index output stay exactly as they are. The one
  deliberate exception is that an already-published `(name, version)` becomes
  a logged skip instead of a refusal (§4.3).
- **Auto-merge applies to Python only** (§6.4 step 2). Go and every other
  family stay `merge: "human"`.
- **Never fake a precondition.** Releases are human-gated until V1. A
  deployment task attests the release; it never cuts it.

## Review Focus

These are conditions the spec implies that no task's ordinary tests would
exercise, most likely first. Each has a pinned test in the task named.

1. **A package that is published for some targets but not others** (a release
   whose CD failed partway, say). That is neither `build` nor `skip`. It must
   fail loud and name the missing and present identities. Test: T5
   `test_classify_partial_publication_is_an_error`.
2. **A human or agent pushes onto an automation branch after the App did**
   (`lifecycle/*` or `deps/update-*`). The merge must be refused, and a push
   that lands after the check must make the merge itself fail. Tests: T7
   `test_refuses_pr_with_foreign_commit`, `test_merge_passes_match_head_commit`.
3. **`automation` declared where it doesn't belong:**
   - from an agent VM or the Mac (an agent signal is present);
   - off a runner;
   - from a workflow or job that isn't on the allowlist.

   Each must be refused, with the reason named. Tests: T1
   `test_automation_refused_with_agent_signal`,
   `test_automation_requires_ci_runner`,
   `test_scope_refused_for_unlisted_job`.
4. **The driver is re-run after the vergil-containers release succeeded but
   the vergil-python step failed.** It must resume at the vergil-python step.
   It must not cut a second vergil-containers release, because no new
   lifecycle PR has merged since the tag. Test: T10
   `test_rerun_after_containers_release_resumes_at_runtime`.
5. **Releases published before this epic** (no `name` in their manifest entry)
   must still be collected, matched and retained exactly as before. A frozen
   series that is missing from later releases must still be kept. Tests: T4
   `test_old_manifest_shape_still_matches`,
   `test_newest_version_of_every_name_is_kept`.

---

### Task T1: The `automation` identity, allowlist and scopes (opt-in) [`vergil-tooling`]

Adds `automation` as an identity that must be **declared** (§6.1). It doesn't
change how an undeclared runner resolves: that is the T11 cutover, which comes
after every ops workflow has migrated (§6.1 "Migration before cutover").

**Files:**

- Modify: `src/vergil_tooling/lib/identity_mode.py` (`IdentityMode`, `resolve`)
- Create: `src/vergil_tooling/lib/automation.py`
- Create: `src/vergil_tooling/data/automation_workflows.toml`
- Modify: `bin/vrg_release.py:23`, `bin/vrg_update_deps.py:22`,
  `bin/vrg_epic_audit.py:52`, `bin/vrg_submit_pr.py:100`,
  `bin/vrg_github_repo_init.py:45`, `bin/vrg_finalize_pr.py` (new gate),
  `bin/vrg_gh.py` (`_get_denied_pairs`, api rule), `bin/vrg_whoami.py`
- Test: `tests/vergil_tooling/test_identity_mode.py`,
  `test_automation.py` (new), `test_vrg_gh.py`, `test_vrg_release.py`,
  `test_vrg_update_deps.py`, `test_vrg_epic_audit.py`, `test_vrg_finalize_pr.py`,
  `test_vrg_submit_pr.py`, `test_vrg_whoami.py`

**Interfaces:**

- Produces, in `identity_mode.py`:
  - `IdentityMode.AUTOMATION = "automation"`, and `Signal.CI_RUNNER = "ci-runner"`.
  - `def on_ci_runner() -> bool`, which is `os.environ.get("GITHUB_ACTIONS") == "true"`.
  - `resolve()` accepts `VRG_IDENTITY_MODE=automation` **only** when
    `on_ci_runner()` is true and no mode file, `app.pem` or `VRG_APP_ID` is
    present. Otherwise it raises `IdentityModeError` naming the conflicting
    signal. It never falls back to another mode.
  - `is_human()` stays `mode is HUMAN`, and `is_agent()` stays `mode in
    {USER, AUDIT}`.
- Produces, in `automation.py`:

  ```python
  SCOPE_RELEASE = "release"; SCOPE_UPDATE_DEPS = "update-deps"; SCOPE_MERGE = "merge"
  SCOPE_LIFECYCLE = "lifecycle"; SCOPE_EPIC_SWEEP = "epic-sweep"; SCOPE_REPO_CONFIG = "repo-config"

  @dataclass(frozen=True)
  class AutomationGrant:
      workflow: str            # fnmatch pattern over GITHUB_WORKFLOW_REF (the *caller* workflow for reusable workflows)
      job: str                 # exact GITHUB_JOB (the job id inside the reusable workflow)
      scopes: frozenset[str]

  class AutomationRefused(Exception): ...           # message names mode, workflow ref, job and scope

  def load_grants() -> tuple[AutomationGrant, ...]  # packaged data file; malformed file -> ConfigError
  def is_authorised(scope: str) -> bool             # mode is AUTOMATION and a grant matches ref+job+scope
  def require(scope: str) -> None                   # raises AutomationRefused unless is_authorised
  ```

- `automation_workflows.toml` initial content (the org's ops callers):

  ```toml
  [[grant]]
  workflow = "vergil-project/*/.github/workflows/ops.yml@refs/heads/*"
  job = "epic-sweep"
  scopes = ["epic-sweep"]

  [[grant]]
  workflow = "vergil-project/*/.github/workflows/ops.yml@refs/heads/*"
  job = "github-config"
  scopes = ["repo-config"]

  [[grant]]
  workflow = "vergil-project/vergil-containers/.github/workflows/ops.yml@refs/heads/develop"
  job = "lifecycle"
  scopes = ["lifecycle", "release", "update-deps", "merge"]
  ```

- **Gates**:
  - `vrg-release` runs if `is_human() or is_authorised(SCOPE_RELEASE)`.
  - `vrg-update-deps` runs if `is_human() or is_authorised(SCOPE_UPDATE_DEPS)`.
  - `vrg-epic-audit --close` runs if `is_human() or
    is_authorised(SCOPE_EPIC_SWEEP) or _automated_sweep()`. `_automated_sweep`
    is the compatibility path, and T11 removes it.
  - `vrg-submit-pr` and `vrg-github-repo-init` refuse any identity that isn't
    HUMAN (`not is_human()` replaces `is_agent()`).
  - `vrg-finalize-pr` refuses any identity that isn't HUMAN, **except**
    `--cleanup-only` under `is_authorised(SCOPE_RELEASE)`. `vrg-release` calls
    it that way (`lib/release/finalize.py` `_FINALIZE_CMD`).
- **`vrg-gh` under AUTOMATION:**
  - `_get_denied_pairs` uses `_DENIED_AGENT`;
  - `gh api` is allowed for GET only, using the existing `_api_is_get`;
  - nothing is added back unless the caller sweep (Step 5) shows a need.
- **`vrg-whoami --explain`** reports `automation` together with the matching
  grant (or "no grant").

- [ ] **Step 1: Write the identity tests** in `test_identity_mode.py`:
  - `test_automation_resolves_on_ci_runner`: with `GITHUB_ACTIONS=true` and
    `VRG_IDENTITY_MODE=automation`, `current_mode() is IdentityMode.AUTOMATION`.
  - `test_automation_requires_ci_runner`: same env without `GITHUB_ACTIONS`
    raises `IdentityModeError`, and the message contains `"not a CI runner"`.
  - `test_automation_refused_with_agent_signal`: with an `app.pem` present,
    it raises, and the message names `app-pem`.
  - `test_undeclared_runner_still_resolves_human`: with `GITHUB_ACTIONS=true`
    and no mode set, the mode is HUMAN (unchanged until T11).
- [ ] **Step 2: Write the grant tests** in `test_automation.py`:
  - `test_scope_granted_for_listed_caller_and_job`:
    `GITHUB_WORKFLOW_REF=vergil-project/vergil-containers/.github/workflows/ops.yml@refs/heads/develop`
    with `GITHUB_JOB=lifecycle` gives `is_authorised("release")`.
  - `test_scope_refused_for_unlisted_job`: same ref, `GITHUB_JOB=build`.
    `require("release")` raises `AutomationRefused`, and the message contains
    the ref and the job.
  - `test_scope_refused_outside_grant_scopes`: `GITHUB_JOB=epic-sweep` with
    `require("release")` raises.
  - `test_not_authorised_unless_mode_is_automation`: as HUMAN with a matching
    ref, `is_authorised(...)` is False.
  - `test_malformed_grant_file_is_config_error`.
- [ ] **Step 3: Run** the two test files. Expected: FAIL, because the
  `automation` module and the enum member are missing.
- [ ] **Step 4: Implement `identity_mode` and `automation.py`,** plus the data
  file, packaged with `importlib.resources` the same way as
  `data/host_dependencies.toml`.
- [ ] **Step 5: Run the caller sweep** with `grep -rn identity_mode
  src/vergil_tooling`. Record every caller and what it decides under
  AUTOMATION in the PR body: allow, refuse, or scoped. Then write the gate
  tests:
  - `test_vrg_release_runs_under_authorised_automation`
  - `test_vrg_release_refuses_unauthorised_automation`, whose stderr names
    the scope
  - the same pair for `vrg-update-deps`
  - `test_epic_audit_close_under_epic_sweep_scope`
  - `test_epic_audit_close_still_accepts_sweep_env`, the compatibility path
  - `test_submit_pr_refuses_automation`
  - `test_finalize_pr_refuses_automation_without_cleanup_only`
  - `test_finalize_pr_cleanup_only_allowed_under_release_scope`
  - `test_vrg_gh_automation_uses_agent_deny_list`, where `pr merge` is
    denied
  - `test_vrg_gh_automation_api_get_only`
  - `test_whoami_explain_reports_grant`
- [ ] **Step 6: Implement the gates.** Then run
  `vrg-container-run -- vrg-validate`. Expected: PASS at 100% coverage.
- [ ] **Step 7: Commit.** `vrg-commit --type feat --scope identity --message
  "add the declared automation identity with workflow grants and scopes"`

### Task T2: `[[package]]` configuration [`vergil-tooling`]

**Files:**

- Modify: `src/vergil_tooling/lib/config.py` (`PackageConfig`, `VergilConfig`,
  `_parse_package_config`, `_check_package_overlay`,
  `_check_package_python_lock`, `read_config`)
- Modify: the `cfg.package` presence checks in `lib/ci_evidence.py:723` and
  `lib/github_config.py:727`, which become `bool(cfg.packages)`
- Test: `tests/vergil_tooling/test_config.py` (the package section from :1774)

**Interfaces:**

- Produces:
  - `PackageConfig.overlay_path: str = PACKAGE_OVERLAY_PATH`. The parser sets
    it to `f"packaging/{name}.nfpm.overlay.yaml"` in the array form.
  - `VergilConfig.packages: tuple[PackageConfig, ...] = ()`.
  - `VergilConfig.package` becomes a read-only property:
    - `None` when there are no packages;
    - the only package when there is exactly one;
    - with more than one, it raises `ConfigError("multi-package repo: read
      cfg.packages")`, so a reader that wasn't migrated fails loud.
  - `def find_package(cfg: VergilConfig, name: str) -> PackageConfig`, which
    raises `ConfigError` naming the known packages.
- Parsing rules (§4.1):
  - a `[package]` table becomes a 1-tuple;
  - a `[[package]]` array becomes an N-tuple;
  - in the array form every entry needs `name`, and names must be unique;
  - **error messages** name the entry as `[[package]] #<i> (<name>)`. The
    single form keeps its existing `[package]` messages byte for byte.
- `_check_package_overlay` and `_check_package_python_lock` run once per
  package against that package's `overlay_path`.

- [ ] **Step 1: Write the tests:**
  - `test_single_package_table_unchanged`: today's `_PKG_STAGED` text gives
    `cfg.packages == (cfg.package,)` with `overlay_path ==
    "packaging/nfpm.overlay.yaml"`.
  - `test_package_array_parses_each_entry`: two `[[package]]` entries named
    `vergil-python3.14` and `vergil-python3.15`, in order, each with its own
    version and smoke.
  - `test_package_array_requires_name`: the error contains
    `"[[package]] #1 (<unnamed>)"` and `"name"`.
  - `test_package_array_duplicate_name`: the error names both indices.
  - `test_package_array_overlay_path_per_name`.
  - `test_package_property_raises_for_multi`.
  - `test_find_package_unknown_name_lists_known`.
  - `test_overlay_check_runs_per_package`: a missing build-command and
    overlay for entry 2 errors, naming entry 2.
- [ ] **Step 2: Run** `vrg-container-run -- uv run pytest tests/vergil_tooling/test_config.py -v`. Expected: the new tests FAIL.
- [ ] **Step 3: Implement it.** Keep `_parse_package_config` returning a single
  `PackageConfig` for one table. Add `_parse_packages(raw, source) ->
  tuple[PackageConfig, ...]`, which dispatches on table versus list and
  raises `ConfigError` when both forms appear.
- [ ] **Step 4: Run** `vrg-container-run -- vrg-validate`. Expected: PASS. Every
  existing package test is unchanged.
- [ ] **Step 5: Commit.** `vrg-commit --type feat --scope package --message "accept a [[package]] array with per-package overlays"`

### Task T3: Package dimension in matrix, build, staged and nFPM [`vergil-tooling`]

**Files:**

- Modify: `src/vergil_tooling/lib/package/matrix.py`, `build.py`, `staged.py`,
  `nfpm.py`, and the `bin/vrg_package.py` `matrix` and `build` commands
- Test: `test_package_matrix.py`, `test_package_build.py`,
  `test_package_staged.py`, `test_package_nfpm.py`, `test_vrg_package.py`

**Interfaces:**

- Consumes T2's `cfg.packages`, `find_package` and `PackageConfig.overlay_path`.
- Produces:
  - `BuildCell.package: str` and `TestCell.package: str`. Both are the
    resolved package name (`naming.package_name`).
  - `def resolve_all(repo_root: Path, cfg: VergilConfig, *, tier: Tier =
    Tier.FULL) -> Matrix`. It concatenates `resolve(pkg, ...)` for each
    package.
  - **Cell ids:**
    - unchanged when there is one package;
    - with more than one, `f"{name}--{base_id}"` (for example
      `vergil-python3.15--shared-amd64`).
  - `resolve(pkg, *, tier, name: str, prefix: bool)`. The new parameters are
    keyword-only. Existing direct callers pass `prefix=False`.
  - `to_json(m, *, versions: Mapping[str, str], runtimes: Mapping[str, str |
    None])`. Each row gains `package` and `version`, plus `runtime` when the
    package has one. Single-package rows keep their existing keys and also
    gain `package` and `version`.
  - `manifest(m, *, versions, runtimes)`. Each entry gains `name` and
    `version`. With several runtimes the top-level `"runtime"` key becomes
    `"runtimes": {name: runtime}`. The single-package shape keeps
    `"runtime"`.
  - `run_build(...)`: same signature as today. It finds the cell in
    `resolve_all`, picks the package with `find_package(cfg, cell.package)`,
    and uses `version = pkg.version or version`.
  - `staged.py`'s env gains `VRG_PACKAGE_NAME=ctx.name` and
    `VRG_PACKAGE_VERSION=ctx.version`.
  - `nfpm.load_overlay(repo_root: Path, pkg: PackageConfig) -> dict[str, Any]`
    reads `pkg.overlay_path`.
  - `vrg-package matrix` computes `versions` (`pkg.version or vrg-version
    show`) and `runtimes` (`python_builder.release_runtime` per python-builder
    package).

- [ ] **Step 1: Write the tests:**
  - `test_single_package_cell_ids_unchanged`: today's ids exactly
    (`shared-noarch`, `shared-amd64`, `native-…`, `test-…`).
  - `test_multi_package_cell_ids_prefixed`.
  - `test_to_json_rows_carry_package_and_version`.
  - `test_manifest_entries_carry_name_and_version`.
  - `test_manifest_single_package_keeps_runtime_key` and
    `test_manifest_multi_runtime_map`.
  - `test_run_build_selects_package_by_cell`: two staged packages; building
    `b--shared-amd64` renders package `b` at its own version.
  - `test_staged_env_names_package_and_version`: the build command echoes
    `$VRG_PACKAGE_NAME:$VRG_PACKAGE_VERSION`, and the test asserts that output.
  - `test_load_overlay_per_package_path`.
- [ ] **Step 2: Run** the five test files. Expected: FAIL.
- [ ] **Step 3: Implement it.** Change `bin/vrg_package.py` `_cmd_matrix` to
  use `resolve_all`.
- [ ] **Step 4: Run** `vrg-container-run -- vrg-validate`. Expected: PASS.
- [ ] **Step 5: Commit.** `vrg-commit --type feat --scope package --message "add a package dimension to the matrix, manifest and builds"`

### Task T4: Name-aware index matching and per-package retention [`vergil-tooling`]

**Files:**

- Modify: `src/vergil_tooling/lib/package/index/collect.py` (`ManifestEntry`,
  `_manifest_entries`, `_match`), `retention.py` (`_by_lines`, `select`)
- Test: `test_package_index_collect.py`, `test_package_index_retention.py`,
  `test_package_index_site.py`

**Interfaces:**

- Consumes T3's manifest entry shape: `{"fmt", "arch", "suite", "name",
  "version"}`.
- Produces:
  - `ManifestEntry.name: str | None = None` and `ManifestEntry.version: str |
    None = None`.
  - **`pattern()` when `name` is set** anchors on the escaped name:
    - deb: `rf"{re.escape(name)}_.+-{rev}_{arch}\.deb"`
    - rpm: `rf"{re.escape(name)}-.+-{rev}\.{rpmarch}\.rpm"`

    When `name` is None the old pattern applies, unchanged.
  - `_match` keeps its rule of exactly one file per entry and no files left
    unclaimed.
  - **Retention:** `_by_lines` groups by `(product, name)` instead of
    `product`. Each name's lines come from the tags that carry that name.
    `select` therefore always keeps the newest version of every name,
    together with its release (§4.4).

- [ ] **Step 1: Write the tests:**
  - `test_two_packages_same_arch_match_by_name`: a manifest with
    `vergil-python3.14` and `vergil-python3.15` for deb/amd64 matches both
    files.
  - `test_old_manifest_shape_still_matches`: a fixture manifest with no
    `name` for a single-package release collects exactly as before.
  - `test_old_shape_two_packages_still_errors`: the error text
    `"has 2 packages for deb/amd64"` is kept.
  - `test_newest_version_of_every_name_is_kept`: product `vergil-python`
    with tags v1.0.4, which has 3.14 and 3.15, then v1.0.5 through v1.0.9,
    which have 3.15 only, and `keep=2`. `vergil-python3.14` from v1.0.4 is
    still selected.
  - `test_retire_still_drops_by_name`: the existing `[retention].retire`
    behaviour is unchanged.
  - `test_site_index_contains_both_series`.
- [ ] **Step 2: Run** the three test files. Expected: FAIL.
- [ ] **Step 3: Implement it.**
- [ ] **Step 4: Run** `vrg-container-run -- vrg-validate`. Expected: PASS.
- [ ] **Step 5: Commit.** `vrg-commit --type feat --scope package-index --message "match manifest entries by package name and retain per package"`

### Task T5: The `build`/`skip` classification in preflight and CD [`vergil-tooling`]

**Files:**

- Modify: `src/vergil_tooling/lib/package/index/published.py`,
  `lib/release/preflight.py` (`_check_packages_unpublished`),
  `lib/release/context.py`, and the `bin/vrg_package.py` `check-unpublished`
  and `matrix` commands
- Test: `test_package_index_published.py`, `test_release_preflight.py`
  (:1094-1164), `test_vrg_package.py`

**Interfaces:**

- Consumes T3's `resolve_all` and `manifest`.
- Produces, in `published.py`:

  ```python
  @dataclass(frozen=True)
  class PackagePlan:
      name: str
      version: str
      action: Literal["build", "skip"]
      identities: tuple[Identity, ...]

  def classify(repo_root: Path, cfg: VergilConfig, product: str, version: str,
               gh: Run = local_run) -> list[PackagePlan]
  def render_plan(plans: Sequence[PackagePlan]) -> str   # one line per package: "build  <name> <version>" / "skip   <name> <version> (published)"
  ```

- **The classification rule:**
  - **`skip`:** every identity for the package is already in
    `published(product, exclude_tag=f"v{version}")`.
  - **`build`:** none of them is.
  - **Some but not all:** raises `PackageError`, naming the identities
    present and missing (Review Focus 1).
- `check_release` is replaced by `classify`. It no longer raises just because
  an identity is published.
- `ReleaseContext.package_plan: list[PackagePlan] = field(default_factory=list)`.
  Preflight sets it and prints `render_plan`.
- **When every package is `skip`**, preflight prints `⚠ no packages to build
  in this release` and proceeds (§4.3).
- `vrg-package check-unpublished --version V [--repo R] --plan-out PATH`
  prints `render_plan`, writes the JSON list of plans, and appends to
  `$GITHUB_STEP_SUMMARY`.
- `vrg-package matrix --plan PATH` leaves out every cell whose `package` is
  `skip`. When nothing is left it writes `enabled=false` and `skipped=<names>`
  to `$GITHUB_OUTPUT`.

- [ ] **Step 1: Write the tests:**
  - `test_classify_unpublished_is_build`
  - `test_classify_published_is_skip`
  - `test_classify_partial_publication_is_an_error`
  - `test_classify_multi_mixed`: 3.14 build, 3.15 skip
  - `test_single_package_published_is_now_skip`, the documented behaviour
    change
  - `test_preflight_records_plan_and_prints_skips`
  - `test_preflight_all_skipped_warns_and_proceeds`
  - `test_check_unpublished_writes_plan_and_summary`
  - `test_matrix_plan_omits_skipped_cells`
  - `test_matrix_plan_all_skipped_disables`
- [ ] **Step 2: Run** the test files. Expected: FAIL.
- [ ] **Step 3: Implement it.** Update the `check_unpublished` docstring and
  the advice that names `[package].version`.
- [ ] **Step 4: Run** `vrg-container-run -- vrg-validate`. Expected: PASS.
- [ ] **Step 5: Commit.** `vrg-commit --type feat --scope package --message "classify packages as build or skip instead of refusing a published version"`

### Task T6: Per-package install-test, check-runtime and release index wait [`vergil-tooling`]

**Files:**

- Modify: `src/vergil_tooling/lib/package/install_test.py` (`run_install_test`),
  `bin/vrg_package.py` (`check-runtime`), `lib/package/runtime.py`,
  `lib/release/package_index.py` (`_wait`, `wait_for_index`),
  `lib/release/context.py`
- Test: `test_package_install_test.py`, `test_package_runtime.py`,
  `test_vrg_package.py`, `test_release_package_index.py`

**Interfaces:**

- Consumes T3's `TestCell.package` and `manifest["runtimes"]`, and T5's
  `ctx.package_plan`.
- Produces:
  - **`run_install_test`** resolves the package from `cell.package` and uses
    that package's `smoke` and residue path.
  - **`check-runtime`:**
    - reads `runtime` (single-package) or `runtimes` (multi);
    - groups artifacts by package name;
    - calls `runtime.check_consistent(runtime, packages, where)` once per name.
  - **The release index wait** loops over the `build` entries in
    `ctx.package_plan`, with one deadline for all of them.
  - `ReleaseContext.package_index_urls: list[str]`. `package_index_url` stays
    set for single-package repos, for the existing summary at
    `orchestrator.py:304`.

- [ ] **Step 1: Write the tests:**
  - `test_install_test_uses_cells_package_smoke`
  - `test_check_runtime_per_package_series`: two packages on different
    `vergil-python<x.y>` pass
  - `test_check_runtime_mismatch_within_one_package_fails`
  - `test_index_wait_waits_for_each_built_package`
  - `test_index_wait_skips_skipped_packages`
  - `test_index_wait_single_package_sets_url`
- [ ] **Step 2: Run** the tests. Expected: FAIL.
- [ ] **Step 3: Implement it.**
- [ ] **Step 4: Run** `vrg-container-run -- vrg-validate`. Expected: PASS.
- [ ] **Step 5: Commit.** `vrg-commit --type feat --scope package --message "install-test, runtime-check and index-wait per package"`

### Task D1: Deployment: release `vergil-tooling` with automation opt-in and multi-package [`vergil-tooling`, `--kind deployment`]

**Blocked by:** T1, T2, T3, T4, T5, T6.

**Precondition (human-attested):** a `vergil-tooling` release that contains
T1–T6 has been cut and promoted to `v2.1`.

**Procedure:**

1. Run `uv tool install` from the new tag.
2. `vrg-whoami --explain` on the Mac reports HUMAN.
3. Run `vrg-package matrix` inside a scratch checkout of `vergil-python` at
   `develop`. The cell ids are unchanged.

**Acceptance:**

- The `v2.1` tag points at the release.
- Step 3's output is identical to the output before this release.

### Task A1: Package dimension in `ci-package.yml` and `cd-release.yml` [`vergil-actions`]

**Blocked by:** D1.

**Files:**

- Modify: `.github/workflows/ci-package.yml` (build L52-66), and the
  `cd-release.yml` jobs `package-matrix` (L179-199) and `package-build`
  (L205-248)
- Modify: `actions/cd/release/tests/package-runtime.test.sh`
- Create: `actions/cd/release/tests/package-multi.test.sh`, and wire it into
  `smoke-cd-release.yml`

**Interfaces:**

- Consumes T3's matrix rows (`package`, `version`, `runtime`) and T5's
  `check-unpublished --plan-out` and `matrix --plan`.
- Produces:
  - **The build steps** in both workflows run
    `vrg-package build --cell "$CELL" --runtime "$RUNTIME" --version "$VERSION"`,
    with `VERSION: ${{ matrix.version }}`. This replaces `$(vrg-version show)`.
  - **`package-matrix`** runs
    `vrg-package check-unpublished --version "$version" --plan-out package-plan.json`,
    then `vrg-package matrix --github-output --manifest packages-manifest.json --plan package-plan.json`.
    It uploads `package-plan.json` together with the manifest.
  - **Artifact names** are unchanged (`package-build-${{ matrix.id }}`,
    `package-unsigned-${{ matrix.id }}`). They stay unique because the cell
    ids carry the package (T3).

- [ ] **Step 1: Write the multi-package test** in `package-multi.test.sh`.
  It runs the real `vrg-package` against an offline two-package fixture repo,
  then asserts that:
  - the matrix JSON has prefixed ids and per-row versions;
  - `--plan` with a skipped package removes that package's cells;
  - the workflow YAML (read with `yq`) passes `${{ matrix.version }}` to
    `--version` in both workflows.
- [ ] **Step 2: Run** `bash actions/cd/release/tests/package-multi.test.sh`. Expected: FAIL.
- [ ] **Step 3: Edit the workflows.**
- [ ] **Step 4: Run** both harnesses, then `vrg-container-run -- vrg-validate`
  (actionlint). Expected: PASS.
- [ ] **Step 5: Commit.** `vrg-commit --type feat --scope package --message "carry the package dimension and per-package version through CI and CD"`

### Task A2: Ops workflows declare `automation` [`vergil-actions`]

**Blocked by:** D1.

**Files:**

- Modify: `.github/workflows/ops-epic-sweep.yml` (the step at L75-85),
  `.github/workflows/ops-github-config.yml` (all three jobs)
- Create: `actions/ops/tests/automation-env.test.sh`, and wire it into a
  `smoke-ops.yml` (new)

**Interfaces:**

- Consumes T1's `VRG_IDENTITY_MODE=automation` and the `epic-sweep` and
  `github-config` grants.
- Produces:
  - Every step that runs a `vrg-*` tool sets `VRG_IDENTITY_MODE: automation`.
  - `ops-epic-sweep.yml` keeps `VRG_EPIC_SWEEP: "1"` until T11.
  - Job ids stay `epic-sweep` and `github-config`, matching the grants.
  - `branch-sweep` and `dependabot-audit` don't call identity-gated
    commands. They declare `automation` anyway, so the T11 cutover finds no
    undeclared runner.

- [ ] **Step 1: Write the test.** `automation-env.test.sh` uses `yq` to assert
  that every step in both workflows running a `vrg-` command sets
  `VRG_IDENTITY_MODE=automation`. It also checks that the `epic-sweep` and
  `github-config` job ids exist.
- [ ] **Step 2: Run** the test. Expected: FAIL.
- [ ] **Step 3: Edit the workflows.**
- [ ] **Step 4: Run** the harness and `vrg-container-run -- vrg-validate`. Expected: PASS.
- [ ] **Step 5: Commit.** `vrg-commit --type ci --scope ops --message "declare the automation identity in the ops workflows"`

### Task D2: Deployment: release `vergil-actions` [`vergil-actions`, `--kind deployment`]

**Blocked by:** A1, A2.

**Precondition (human-attested):** a `vergil-actions` release that contains
A1 and A2 has been cut and promoted to `v2.1`.

**Procedure:**

1. Trigger `ops.yml` (`workflow_dispatch`) in `vergil-tooling`, then in
   `vergil-containers`.
2. Open each `epic-sweep` and `github-config` job log and check that
   `vrg-whoami --explain` shows `automation` with its grant. Add that echo to
   both workflows in A2.
3. Run `vergil-tooling`'s CD once (its next release, or a `workflow_dispatch`
   dry run), and check that the package matrix is unchanged for a
   single-package repo.

**Acceptance:**

- Both ops runs are green, and each declares `automation`.
- `vergil-tooling`'s package CD output is unchanged.

### Task P1: Multi-series `vergil-python` (3.14 resumed, 3.15) [`vergil-python`]

**Blocked by:** D2.

**Files:**

- Modify: `runtime.toml`, which becomes `[[series]]`; `vergil.toml`, which
  becomes `[[package]]` × 2; `packaging/build.sh` (L45-65);
  `scripts/dev/update_deps.py`; `docs/runtime-package-pattern.md` (the
  sections "One package per series, side by side", "Changing the pin", and
  "Never re-release an unchanged name and version")
- Test: `tests/test_build.py`, `tests/test_update_deps.py`

**Interfaces:**

- Consumes T3's `VRG_PACKAGE_NAME`, T2's `[[package]]`, and T5's skip.
- Produces:
  - **`runtime.toml`** holds one `[[series]]` per series (`series`, `cpython`,
    `pbs_tag`, `[series.sha256]` `x86_64`, `aarch64`), newest first. The 3.14
    entry is filled in by running the hook, never typed by hand.
  - **`build.sh`** picks the `[[series]]` entry whose
    `vergil-python${series}` equals `$VRG_PACKAGE_NAME`, and fails if there is
    no such entry. Every existing guard (L53-106) applies to that entry.
  - **In `update_deps.py`:**
    - `def load_series(runtime_text: str) -> list[dict[str, Any]]`;
    - `def render_runtime(text: str, series: str, cpython: str, tag: str,
      sums: dict[str, str]) -> str`, which rewrites only the named
      `[[series]]` block;
    - `rewrite_package(text, series, ...)`, which rewrites only the
      `[[package]]` block whose `name` is `vergil-python{series}`;
    - `main` refreshes each series independently.
  - **New PBS builds** move every series to the newest stable PBS tag, even
    when the CPython patch hasn't changed (§5, the human's decision).

- [ ] **Step 1: Write the build tests:**
  - `test_build_selects_series_by_package_name`: with two series and
    `VRG_PACKAGE_NAME=vergil-python3.14`, it stages under
    `opt/vergil/python/3.14`.
  - `test_build_rejects_package_name_without_series`.
  - Every existing `test_build_rejects_*` test is ported to the two-series
    fixture.
- [ ] **Step 2: Write the update tests:**
  - `test_each_series_refreshes_independently`: 3.14 bumps a patch, and 3.15
    is current except for the tag.
  - `test_new_pbs_tag_alone_moves_every_series`.
  - `test_current_series_is_left_untouched`: byte-identical blocks.
  - `test_rewrite_package_touches_only_its_block`.
  - `test_missing_arch_for_one_series_fails_without_writing`.
- [ ] **Step 3: Run** `bash tests/check-build.sh && bash tests/check-update-deps.sh`. Expected: FAIL.
- [ ] **Step 4: Implement it.** Then run the hook locally
  (`VRG_DEPS_REPORT=$(mktemp) scripts/dev/update-deps.sh`) to fill in the
  3.14 series. Check the 3.14 checksums against PBS `SHA256SUMS`.
- [ ] **Step 5: Run** `vrg-container-run -- vrg-validate`. Expected: PASS.
- [ ] **Step 6: Commit.** `vrg-commit --type feat --scope runtime --message "publish every listed series; resume 3.14 patches"`

### Task D3: Deployment: release `vergil-python` with two series [`vergil-python`, `--kind deployment`]

**Blocked by:** P1.

**Precondition (human-attested):** a `vergil-python` release containing P1 has
been cut. It was released by hand, with its CD green.

**Procedure:**

1. The CD job summary's package plan lists `build` for `vergil-python3.14`,
   and `build` or `skip` for 3.15, depending on whether P1's refresh moved it.
2. On a scratch Ubuntu 24.04 VM: `apt-get install vergil-python3.14
   vergil-python3.15`. Both interpreters run their smoke commands.
3. `apt-cache policy vergil-python3.14` shows the newest 3.14 patch, not
   `3.14.8+20261013`.

**Acceptance:**

- Steps 1–3 are observed.
- Record the release URL and the plan output.

### Task T7: The unattended merge rule [`vergil-tooling`]

**Blocked by:** T1.

**Files:**

- Create: `src/vergil_tooling/lib/automation_merge.py`
- Modify: `lib/github.py` (`merge`, plus a new `pr_provenance_view`),
  `lib/pr_merge.py` (`wait_and_merge`)
- Test: `tests/vergil_tooling/test_automation_merge.py` (new),
  `test_pr_merge.py`, `test_github.py`

**Interfaces:**

- Consumes T1's `automation.require(SCOPE_MERGE)`.
- Produces:

  ```python
  @dataclass(frozen=True)
  class PrProvenance:
      url: str
      author: str                       # author.login
      cross_repository: bool            # isCrossRepository
      head_sha: str                     # headRefOid
      commit_authors: tuple[str, ...]   # commits[].authors[].login, every author of every commit

  class AutomationMergeRefused(Exception): ...   # names the PR and the failed condition

  def automation_logins() -> frozenset[str]       # from VRG_AUTOMATION_LOGIN (comma-separated); empty -> AutomationMergeRefused
  def check(pr: PrProvenance, *, expected_head_sha: str, logins: frozenset[str]) -> None
  def merge_automation_pr(pr: str, *, expected_head_sha: str, strategy: str) -> None
      # require(SCOPE_MERGE); check(github.pr_provenance_view(pr), ...); pr_merge.wait_and_merge(pr, strategy=strategy, match_head_commit=expected_head_sha)
  ```

- `github.pr_provenance_view(pr: str) -> PrProvenance` (`gh pr view --json
  url,author,isCrossRepository,headRefOid,commits`).
- `github.merge(pr: str, *, strategy: str, match_head_commit: str | None = None)`
  adds `--match-head-commit <sha>` when it is set.
- `pr_merge.wait_and_merge(..., match_head_commit: str | None = None)` passes it
  on to every merge attempt. `on_branch_updated` refreshes the expected SHA
  **only** for an update-branch the engine made itself, and re-runs `check`
  first.

- [ ] **Step 1: Write the tests:**
  - `test_accepts_app_authored_same_repo_pr`
  - `test_refuses_foreign_author`
  - `test_refuses_cross_repository`
  - `test_refuses_pr_with_foreign_commit`: one commit is authored by
    `wphillipmoore-vergil-user`
  - `test_refuses_head_sha_mismatch`
  - `test_missing_logins_env_refuses`
  - `test_merge_passes_match_head_commit`: the recorded gh argv contains
    `--match-head-commit <sha>`
  - `test_merge_requires_merge_scope`
  - `test_engine_rechecks_after_its_own_branch_update`
- [ ] **Step 2: Run** the tests. Expected: FAIL.
- [ ] **Step 3: Implement it.**
- [ ] **Step 4: Run** `vrg-container-run -- vrg-validate`. Expected: PASS.
- [ ] **Step 5: Commit.** `vrg-commit --type feat --scope merge --message "add the unattended merge rule: app author, same repo, app commits, pinned head"`

### Task T8: `vrg-update-deps` passes the window and merges under the rule [`vergil-tooling`]

**Blocked by:** T1, T7.

**Files:**

- Modify: `src/vergil_tooling/lib/update_deps/updaters/repo_hook.py` (:58-64),
  `lib/update_deps/pr.py` (`prepare_pr`, `merge_pr`),
  `lib/update_deps/context.py`
- Test: `test_update_deps_repo_hook.py`, `test_update_deps_pr.py`

**Interfaces:**

- Consumes T7's `merge_automation_pr`, and the existing window loader in
  `lib/versions/window.py`, the one `vrg-ci-resolve` uses to read
  `window.json` at `[dependencies].vergil`.
- Produces:
  - **The hook command** gains
    `VRG_DEPS_WINDOW=/workspace/.vergil/deps-window.json`. The file is the
    released window, written before the hook runs and deleted after. If the
    window can't be loaded, the run fails loud and the hook never runs.
  - **`prepare_pr`** records `ctx.pushed_sha` (the HEAD that was pushed).
  - **`merge_pr`**, when the mode is AUTOMATION, calls
    `merge_automation_pr(url, expected_head_sha=ctx.pushed_sha,
    strategy="merge")`. As HUMAN it behaves as today.
  - **Pending entries:** a held-back report entry whose `note` starts with
    `PENDING_PREFIX = "pending:"` (a constant in `lib/update_deps/report.py`)
    is copied into `ctx.pending` as well as the report. The driver reads it
    (T10).

  This changes hand-run behaviour, deliberately: the hook always gets the
  window. See the alignment note at the end of this plan.

- [ ] **Step 1: Write the tests:**
  - `test_hook_receives_released_window_file`
  - `test_window_load_failure_aborts_before_hook`
  - `test_pushed_sha_recorded`
  - `test_automation_merge_uses_rule_with_pushed_sha`
  - `test_human_merge_unchanged`
  - `test_pending_entries_collected`
- [ ] **Step 2: Run** the tests. Expected: FAIL.
- [ ] **Step 3: Implement it.**
- [ ] **Step 4: Run** `vrg-container-run -- vrg-validate`. Expected: PASS.
- [ ] **Step 5: Commit.** `vrg-commit --type feat --scope update-deps --message "pass the released window to the repo hook and merge under the automation rule"`

### Task T9: `vrg-release` runs on a runner [`vergil-tooling`]

**Blocked by:** T1, T7, T8.

**Files:**

- Modify: `src/vergil_tooling/lib/release/subprocess.py` (`watch_workflow`),
  `lib/release/confirm.py`, `lib/release/tracking.py`
  (`find_existing_tracking_issue`), `lib/release/merge.py`,
  `lib/release/orchestrator.py`
- Create: `src/vergil_tooling/lib/release/summary.py`
- Test: `test_release_subprocess.py`, `test_release_confirm.py`,
  `test_release_tracking.py`, `test_release_merge.py`, `test_release_summary.py`
  (new), `test_release_orchestrator.py`

**Interfaces:**

- Consumes T7's `merge_automation_pr` and T5's `ctx.package_plan`.
- Produces:
  - **Bounded watch:** `watch_workflow(repo, run_id, *, check_status=True,
    timeout_secs: int)` kills `gh run watch` at the deadline and raises
    `ReleaseError(phase=...)`. Callers pass `release_timeout_secs()`, which
    reads `VRG_RELEASE_WATCH_TIMEOUT` and defaults to `5400`.
  - **No search in the lock:** `find_existing_tracking_issue` lists open
    issues (`gh issue list --state open --json number,title,url --limit
    1000`) and matches the title exactly.
  - **Release and bump PRs under AUTOMATION:**
    - `release/merge.py` `wait_and_merge` uses `merge_automation_pr` with the
      SHA that `prepare` and `bump` pushed;
    - those stages record `ctx.release_pushed_sha` and `ctx.bump_pushed_sha`;
    - as HUMAN they behave as today.
  - **Job summary:** `summary.write_job_summary(ctx: ReleaseContext) -> None`
    appends the release URL, the package plan, CD run URLs and deferred
    failures to `$GITHUB_STEP_SUMMARY` when it is set, and does nothing
    otherwise. The `publish-status` stage calls it.
- **Runner requirements**, documented in the `vrg-release` docstring and
  asserted by preflight under AUTOMATION:
  - local branch `develop` checked out (not a detached HEAD);
  - full history and tags, i.e. `git rev-parse --is-shallow-repository` is
    `false`;
  - `GH_TOKEN` present.

  Each check fails with an actionable message.

- [ ] **Step 1: Write the tests:**
  - `test_watch_workflow_times_out_loud`
  - `test_tracking_lookup_does_not_use_search`
  - `test_release_merge_under_automation_pins_pushed_sha`
  - `test_release_merge_human_unchanged`
  - `test_job_summary_written_when_env_set`
  - `test_job_summary_noop_without_env`
  - `test_preflight_automation_rejects_shallow_clone`
  - `test_preflight_automation_rejects_detached_head`
  - `test_human_release_refused_while_automation_release_in_flight`: an
    origin `release/<v>` branch made by an AUTOMATION run blocks a HUMAN
    preflight, through the existing worktree/branch lock (§6.3 "The reverse
    direction")
- [ ] **Step 2: Run** the tests. Expected: FAIL.
- [ ] **Step 3: Implement it.**
- [ ] **Step 4: Run** `vrg-container-run -- vrg-validate`. Expected: PASS.
- [ ] **Step 5: Commit.** `vrg-commit --type feat --scope release --message "let vrg-release run unattended on a runner with bounded waits and pinned merges"`

### Task T10: `vrg-lifecycle`, the driver [`vergil-tooling`]

**Blocked by:** T5, T7, T8, T9.

**Files:**

- Create: `src/vergil_tooling/bin/vrg_lifecycle.py`, and register it in
  `pyproject.toml` `[project.scripts]`
- Create: `src/vergil_tooling/lib/lifecycle/__init__.py`, `orchestrator.py`,
  `trigger.py`, `lock.py`, `issues.py`
- Test: `tests/vergil_tooling/test_lifecycle_orchestrator.py`,
  `test_lifecycle_trigger.py`, `test_lifecycle_lock.py`,
  `test_lifecycle_issues.py`, `test_vrg_lifecycle.py`

**Interfaces:**

- Consumes:
  - T1's `require(SCOPE_LIFECYCLE)`;
  - T7's `merge_automation_pr`;
  - T5's `classify`;
  - T8's `ctx.pending`;
  - the existing `vrg-release` and `vrg-update-deps` CLIs, run as
    subprocesses inside clones.
- **CLI:**

  ```text
  vrg-lifecycle run --manifest-repo OWNER/REPO --runtime-repo OWNER/REPO
                    --workdir DIR --pushed PATH [--dry-run]
  ```

  - `--pushed` is the `{branch: sha}` JSON that `publish.py --pushed-out`
    writes (C1).
  - `--dry-run` prints every decision and makes no merge, release or PR.
  - Repos are parameters, so nothing in the code names Python.
- **Produces** `orchestrator.run(opts: RunOptions) -> RunResult`, with the
  stages in order:
  1. **`merge-lifecycle`.** Merges every open PR on a `lifecycle/` head whose
     family has `merge: "auto"` in the manifest repo's
     `docker/versions/manifest.yml`. It reads the family with the existing
     `parse_branch` convention `lifecycle/<family>-<kind>-<version>`. The SHA
     comes from `--pushed`. A PR with no recorded SHA is refused.
  2. **`wait-develop-cd`.** For each merge commit, waits for `cd.yml` on
     `develop` with `head_sha` equal to the merge commit to finish with
     conclusion `success`, bounded by `release_timeout_secs()`.
  3. **`release-manifest-repo`.** Runs when
     `trigger.lifecycle_merged_since_last_release(repo) -> bool`: some PR with
     a `lifecycle/` head and base `develop` has `mergedAt` later than the
     newest stable release's `publishedAt`. Release tags live on `main`, so
     ancestry from `develop` isn't used. It then runs `vrg-release` in the
     clone.
  4. **`update-runtime`.** Runs `vrg-update-deps` in the runtime repo clone,
     and collects `pending`.
  5. **`release-runtime`.** Runs when `classify(...)` for the runtime repo has
     at least one `build` and develop is clean. It then runs `vrg-release`.
- **Lock:** `lock.in_flight(repo: str) -> str | None` returns the reason, or
  None. It looks for a `release/*` branch on origin, an open
  `release: <v>` issue (T9's lookup), or an open PR on a `deps/update-*` or
  `lifecycle/*` head **that fails** `automation_merge.check`. A PR that
  passes the check is the driver's own, and is resumed. A reason means
  `deferred` for that repo.
- **Results:**

  ```python
  StageOutcome = Literal["done", "nothing-to-do", "deferred", "pending", "failed"]
  @dataclass(frozen=True)
  class StageResult: stage: str; repo: str; outcome: StageOutcome; detail: str
  @dataclass(frozen=True)
  class RunResult: stages: tuple[StageResult, ...]
  def exit_code(r: RunResult) -> int   # 1 if any "failed", else 0
  ```

- **Issues:** `issues.sync(results: RunResult, *, run_url: str, now: datetime,
  pending_days: int, deferred_days: int) -> None`.
  - Key: one issue per `(repo, stage, outcome)`, titled `lifecycle: <repo>
    <stage> <outcome>`, with label `lifecycle-automation`, filed in that repo.
  - `failed` opens or refreshes the issue.
  - `pending` and `deferred` open or refresh it as well. Once the issue's
    `createdAt` is older than `pending_days` (`VRG_LIFECYCLE_PENDING_DAYS`,
    default `14`) or `deferred_days` (`VRG_LIFECYCLE_DEFERRED_DAYS`, default
    `3`), it is labelled `overdue` and the result becomes `failed`.
  - A key that is absent from this run's results is closed with a comment
    linking the run.
  - Every outcome is written to `$GITHUB_STEP_SUMMARY`.

- [ ] **Step 1: Write the trigger and lock tests:**
  - `test_trigger_true_after_lifecycle_merge`
  - `test_trigger_false_with_only_bump_and_deps_merges`
  - `test_lock_defers_on_foreign_release_branch`
  - `test_lock_resumes_own_deps_pr`
  - `test_lock_defers_on_foreign_deps_pr`
- [ ] **Step 2: Write the orchestrator tests**, with fake subprocess and gh
  runners:
  - `test_full_chain_order`
  - `test_go_family_pr_never_merged`, for `merge: "human"`
  - `test_pr_without_pushed_sha_refused`
  - `test_develop_cd_failure_stops_before_release`
  - `test_rerun_after_containers_release_resumes_at_runtime`
  - `test_runtime_all_skip_no_release`
  - `test_pending_series_is_green_and_recorded`
  - `test_dry_run_makes_no_mutation`
  - `test_requires_lifecycle_scope`
- [ ] **Step 3: Write the issues tests:**
  - `test_failure_issue_opened_then_closed_on_recovery`
  - `test_pending_becomes_overdue_and_fails`
  - `test_summary_lists_every_outcome`
- [ ] **Step 4: Run** all five files. Expected: FAIL.
- [ ] **Step 5: Implement it.** The orchestrator uses the `progress` stage
  model that `vrg-release` uses.
- [ ] **Step 6: Run** `vrg-container-run -- vrg-validate`. Expected: PASS.
- [ ] **Step 7: Commit.** `vrg-commit --type feat --scope lifecycle --message "add vrg-lifecycle, the unattended lifecycle driver"`

### Task P2: The hook reconciles the series list against the window [`vergil-python`]

**Blocked by:** P1, D4.

**Files:**

- Modify: `scripts/dev/update_deps.py`, `scripts/dev/update-deps.sh`, and
  `docs/runtime-package-pattern.md` ("Changing the pin")
- Test: `tests/test_update_deps.py`

**Interfaces:**

- Consumes T8's `VRG_DEPS_WINDOW` (`window.json` schema 1: the
  `languages.python.families[key=="python"].versions[].version`) and T8's
  `pending:` prefix.
- Produces:
  - `def window_series(window_text: str) -> list[str]`. It fails loud on a
    missing family or a schema other than 1.
  - `def add_series(runtime_text: str, vergil_text: str, series: str, cpython:
    str, tag: str, sums: dict[str, str]) -> tuple[str, str]`.
  - `def drop_series(runtime_text: str, vergil_text: str, series: str) ->
    tuple[str, str]`.
  - `main` reconciles first, then refreshes:
    - adds every window series that is missing and has a PBS build;
    - writes a `pending: no PBS build for CPython <x.y> yet` report line for
      each one without a build;
    - drops every listed series that is not in the window.
  - An unset `VRG_DEPS_WINDOW` is an error. T8 always sets it.

- [ ] **Step 1: Write the tests:**
  - `test_window_adds_missing_series_with_build`: a window with
    3.12–3.15 and a list with 3.14 and 3.15 adds 3.12 and 3.13, newest
    first.
  - `test_window_series_without_pbs_build_is_pending`: the report line
    starts with `pending:`, and nothing is written for that series.
  - `test_series_not_in_window_is_dropped`.
  - `test_add_then_rerun_is_noop`.
  - `test_unset_window_fails_loud`.
  - `test_window_schema_mismatch_fails_loud`.
- [ ] **Step 2: Run** the tests. Expected: FAIL.
- [ ] **Step 3: Implement it.**
- [ ] **Step 4: Run** `vrg-container-run -- vrg-validate`. Expected: PASS.
- [ ] **Step 5: Commit.** `vrg-commit --type feat --scope runtime --message "reconcile the series list against the released window"`

### Task D4: Deployment: release `vergil-tooling` with the merge rule, unattended release and driver [`vergil-tooling`, `--kind deployment`]

**Blocked by:** T7, T8, T9, T10.

**Precondition (human-attested):** a `vergil-tooling` release containing
T7–T10 has been cut and promoted to `v2.1`.

**Procedure:**

1. `vrg-lifecycle run --dry-run ...` on the Mac is refused, because the mode
   is not AUTOMATION.
2. Running `vrg-update-deps` by hand in `vergil-tooling` still opens and merges
   a deps PR exactly as before (or reports nothing to update).

**Acceptance:**

- Both observations are recorded.

### Task C1: Per-family `merge` switch, App-authored commits and pushed SHAs [`vergil-containers`]

**Blocked by:** D4.

**Files:**

- Modify: `docker/versions/manifest.yml` (every family), `docker/versions/manifest.py`
  (`_FAMILY_KEYS`, `Family`, `_parse_family`, `dump_manifest`),
  `docker/versions/lifecycle/publish.py` (`GitCli.publish_branch`, `main`
  argparse, the PR body text)
- Test: `docker/versions/test_manifest.py`, `test_apply.py`, `test_runtime.py`,
  `test_publish.py`

**Interfaces:**

- Produces:
  - `Family.merge: Literal["auto", "human"]`. The key is required and
    validated, and unknown values raise `ManifestError`.
  - `python: merge: "auto"`. Every other family gets `merge: "human"`.
  - `merge` is never written to `window.json`, so its schema is unchanged.
- **`publish.py --prs`:**
  - requires `LIFECYCLE_GIT_AUTHOR_NAME` and `LIFECYCLE_GIT_AUTHOR_EMAIL`, and
    exits 1 naming the missing variable;
  - authors and commits each branch as that identity;
  - with `--pushed-out PATH`, writes `{branch: sha}` for every branch it
    pushed, refreshed branches included.
- **PR body:**
  - for a `merge: "auto"` family: "This PR is merged automatically by the
    lifecycle workflow when CI is green. A human is alerted only on failure."
  - for a `human` family: the existing text, unchanged.
- `publish.py` itself still never merges. `test_sync_never_touches_auto_merge`
  stays as it is; the driver merges (T10).

- [ ] **Step 1: Write the tests:**
  - `test_family_merge_required`
  - `test_family_merge_rejects_unknown`
  - `test_python_auto_go_human_in_repo_manifest`
  - `test_window_json_has_no_merge_key`
  - `test_publish_requires_author_env`
  - `test_publish_commits_as_configured_author`
  - `test_pushed_out_lists_every_pushed_branch`
  - `test_auto_family_pr_body_says_automatic`
  - `test_human_family_pr_body_unchanged`
- [ ] **Step 2: Run** `uv run --with pytest --with pyyaml python -m pytest -q`
  in `docker/versions`. Expected: FAIL.
- [ ] **Step 3: Implement it.** Then run `python3 docker/versions/generate.py`
  and check that no generated file changes, apart from any whose input
  intentionally includes `merge`.
- [ ] **Step 4: Run** `vrg-container-run -- vrg-validate`. Expected: PASS.
- [ ] **Step 5: Commit.** `vrg-commit --type feat --scope versions --message "add a per-family merge switch and app-authored lifecycle commits"`

### Task C2: The docker-publish cancel race [`vergil-containers`]

**Blocked by:** none.

**Files:**

- Modify: `.github/workflows/cd-docker-publish.yml` (L28-30) and `ops.yml`
  (the `nightly-rebuild-*` callers)
- Test: `docker/versions/test_workflows.py` (new; reads the YAML with PyYAML)

**Interfaces:**

- Produces:
  - A new `workflow_call` input `trigger: string`, default `"release"`.
  - The concurrency group becomes
    `docker-publish-${{ inputs.image-prefix }}-${{ inputs.trigger }}`.
  - `cancel-in-progress: ${{ inputs.trigger == 'nightly' }}`.
  - The `nightly-rebuild-*` jobs pass `trigger: nightly`, and `cd.yml` passes
    the default.

- [ ] **Step 1: Write the tests:**
  - `test_release_and_nightly_use_different_groups`
  - `test_release_group_never_cancels`
  - `test_nightly_callers_pass_trigger_nightly`
- [ ] **Step 2: Run** the tests. Expected: FAIL.
- [ ] **Step 3: Edit the workflows.**
- [ ] **Step 4: Run** `vrg-container-run -- vrg-validate`. Expected: PASS.
- [ ] **Step 5: Commit.** `vrg-commit --type fix --scope ci --message "keep release image publishes out of the nightly rebuild's cancel group"`

### Task A3: The reusable `ops-lifecycle.yml` [`vergil-actions`]

**Blocked by:** D4.

**Files:**

- Create: `.github/workflows/ops-lifecycle.yml`
- Create: `actions/ops/tests/ops-lifecycle.test.sh`, wired into `smoke-ops.yml`

**Interfaces:**

- Consumes:
  - T10's `vrg-lifecycle run` flags;
  - T1's grant for `vergil-project/vergil-containers/.github/workflows/ops.yml@refs/heads/develop`
    with job `lifecycle`;
  - T7's `VRG_AUTOMATION_LOGIN`.
- **Produces** a `workflow_call` with:
  - inputs: `manifest-repo` and `runtime-repo` (strings), `dry-run` (boolean,
    default false);
  - secrets: `APP_CLIENT_ID` and `APP_PRIVATE_KEY`.
- **Job id `lifecycle`:**
  - `runs-on: ubuntu-latest`, with **no container**, because
    `vrg-update-deps` needs Docker (§6.2);
  - `timeout-minutes: 300`;
  - `concurrency: {group: ops-lifecycle, cancel-in-progress: false}`.
- **Steps:**
  1. Mint the App token, with `owner: ${{ github.repository_owner }}` and
     `repositories` listing the two repos.
  2. Install uv and vergil-tooling (`./actions/shared/setup/vergil`).
  3. Download the `lifecycle-pushed` artifact.
  4. Set the env:
     - `GH_TOKEN` from the App token;
     - `VRG_IDENTITY_MODE: automation`;
     - `VRG_AUTOMATION_LOGIN: ${{ steps.app-token.outputs.app-slug }}[bot]`;
     - git author name and email set to the App bot,
       `<app-id>+<slug>[bot]@users.noreply.github.com`.
  5. Run `vrg-lifecycle run --manifest-repo ... --runtime-repo ...
     --workdir "$RUNNER_TEMP/lifecycle" --pushed lifecycle-pushed.json`, plus
     `--dry-run` when that input is set.

- [ ] **Step 1: Write the test.** `ops-lifecycle.test.sh` uses `yq` to assert:
  - the job id is `lifecycle`;
  - there is no `container:` key;
  - the concurrency is set with no cancellation;
  - `timeout-minutes` is set;
  - `VRG_IDENTITY_MODE` is `automation`;
  - the token step lists exactly the two input repos.
- [ ] **Step 2: Run** the test. Expected: FAIL.
- [ ] **Step 3: Write the workflow.**
- [ ] **Step 4: Run** the harness and `vrg-container-run -- vrg-validate`. Expected: PASS.
- [ ] **Step 5: Commit.** `vrg-commit --type feat --scope ops --message "add the reusable unattended lifecycle workflow"`

### Task D5: Deployment: release `vergil-actions` with `ops-lifecycle.yml` [`vergil-actions`, `--kind deployment`]

**Blocked by:** A3.

**Precondition (human-attested):** a `vergil-actions` release containing A3 has
been cut and promoted to `v2.1`.

**Procedure:**

1. `ops-lifecycle.yml` resolves at `@v2.1`. A `gh workflow view` from a
   caller, or actionlint on C3's caller, succeeds.

**Acceptance:**

- The workflow is reachable at `v2.1`.

### Task C3: Call `ops-lifecycle.yml` from `ops.yml` [`vergil-containers`]

**Blocked by:** C1, D5.

**Files:**

- Modify: `.github/workflows/ops.yml`, where the `language-lifecycle` step 8
  gains `--pushed-out` and the author env; and add a new `lifecycle` job
- Test: `docker/versions/test_workflows.py` (from C2)

**Interfaces:**

- Consumes C1's `--pushed-out` and author env, and A3's `ops-lifecycle.yml@v2.1`.
- **Produces:**
  - In `language-lifecycle`, step 8 passes `--pushed-out "${OUT}/pushed.json"`
    and sets `LIFECYCLE_GIT_AUTHOR_*` to the App bot identity.
  - It uploads the artifact `lifecycle-pushed`, writing `{}` when no branch was
    pushed.
  - The new job `lifecycle` has `needs: language-lifecycle` and `uses:
    vergil-project/vergil-actions/.github/workflows/ops-lifecycle.yml@v2.1`.
  - It runs only on the default branch, with `manifest-repo:
    vergil-project/vergil-containers` and `runtime-repo:
    vergil-project/vergil-python`.
  - **The live switch:** `dry-run: ${{ vars.LIFECYCLE_LIVE != 'true' }}`.
    Until a human sets the repo variable `LIFECYCLE_LIVE=true` (D6), every
    run is a dry run.

- [ ] **Step 1: Write the tests:**
  - `test_lifecycle_job_needs_language_lifecycle`
  - `test_lifecycle_job_runs_only_on_default_branch`
  - `test_pushed_artifact_always_uploaded`
  - `test_lifecycle_dry_run_unless_live_variable_true`
- [ ] **Step 2: Run** the tests. Expected: FAIL.
- [ ] **Step 3: Edit `ops.yml`.**
- [ ] **Step 4: Run** `vrg-container-run -- vrg-validate`. Expected: PASS.
- [ ] **Step 5: Commit.** `vrg-commit --type feat --scope ops --message "run the unattended lifecycle chain after the lifecycle check"`

### Task D6: Deployment: release `vergil-containers` and `vergil-python` [`vergil-containers`, `--kind deployment`]

**Blocked by:** C1, C2, C3, P2.

**Precondition (human-attested):**

- A `vergil-containers` release containing C1–C3 has been cut and promoted.
- A `vergil-python` release containing P2 has been cut **with `vrg-release
  --without-deps`**. Without that flag, the release's closing `update-deps`
  stage would do the 3.12/3.13 backfill by hand and take V1's first real run
  away from it.
- The `package-signing` environment in `vergil-python` has **no required
  reviewers** (§8), checked with `gh api
  repos/vergil-project/vergil-python/environments --jq '.environments[] |
  {name, protection_rules}'`.
- The App installation covers contents, pull requests, issues and actions
  (read) on both repos.

**Procedure:**

1. With `LIFECYCLE_LIVE` unset, run `workflow_dispatch` on `vergil-containers`
   `ops.yml`. The `lifecycle` job runs as a dry run (C3).
2. Its job summary plans `update-runtime` (adding 3.12 and 3.13) and
   `release-runtime` for `vergil-python`. It makes no merge, release or PR.
3. The human sets the repo variable `LIFECYCLE_LIVE=true` on
   `vergil-containers`. That's the switch that starts V1.

**Acceptance:**

- The dry-run output shows the planned stages and makes no mutation.
- `LIFECYCLE_LIVE=true` is set, with the time it was set recorded.

### Task V1: Validation: the first unattended chain run, backfilling 3.12 and 3.13 [`vergil-containers`, `--kind validation`]

**Blocked by:** D6.

**Precondition:** D6 is closed and the human attestations are recorded on D6.

**Procedure:** let the scheduled `ops.yml` run, or dispatch it, and do not
intervene. Then observe:

1. **`update-runtime`.** `vrg-update-deps` in `vergil-python` adds 3.12 and
   3.13 (P2 reconcile). The `deps/update-*` PR is authored by the App and
   merged under the rule (T7 and T8).
2. **`release-runtime`.**
   - `vrg-release` releases `vergil-python` unattended.
   - The package plan shows `build` for 3.12 and 3.13, and `skip` for any
     series that didn't change.
   - The CD is green and the index wait succeeds.
3. **The runtime PRs.** The next run's `language-lifecycle` opens
   `runtime_published` PRs for 3.12 and 3.13. These are lifecycle PRs for the
   `python` family.
4. **`merge-lifecycle`** merges them, `wait-develop-cd` is green, and
   `release-manifest-repo` releases `vergil-containers` unattended and
   promotes `v2.1`.
5. **The runtimes are visible.** `curl` of `window.json` at `v2.1` shows
   `runtime_published: true` for 3.12–3.15.
6. **Nothing fails silently.** No `lifecycle-automation` issue is left open.
   Each job summary lists every stage outcome.
7. **Patches land on 3.14 and 3.15.** On the first PBS release after V1
   starts, a later run moves every series. This is observed if a PBS build
   lands during the validation window; otherwise it is recorded as not yet
   observed.

**Acceptance:**

- Steps 1–6 are observed without human action, with run URLs recorded.
- Step 7 is observed or explicitly recorded as pending, with its tracking
  noted.
- **If any step needed a human,** that step's gap is filed as a follow-on
  under `.github#443` with the exact fallback ("prepared, a human acts"),
  per §6.5 and principle 6. V1 is then still closed as SUCCESS only if every
  gap is filed and the remaining steps passed.

### Task T11: Fail-closed runner cutover [`vergil-tooling`]

**Blocked by:** D2, V1.

**Files:**

- Modify: `src/vergil_tooling/lib/identity_mode.py` (`resolve`),
  `bin/vrg_epic_audit.py` (remove `_automated_sweep`), `bin/vrg_gh.py`,
  `tests/vergil_tooling/conftest.py` (`_hermetic_identity`)
- Test: `test_identity_mode.py`, `test_vrg_epic_audit.py`, `test_vrg_gh.py`

**Interfaces:**

- Produces:
  - **`IdentityMode.RUNNER = "runner"`.** With `on_ci_runner()` true and no
    explicit mode, `resolve()` returns RUNNER (`Signal.CI_RUNNER`).
  - **`VRG_IDENTITY_MODE=human` on a runner** also resolves to RUNNER, and
    `.disagreement` is set.
  - **RUNNER is refused everywhere that matters:**
    - `is_human()` and `is_authorised(...)` are False;
    - `vrg-gh` applies `_DENIED_AGENT` and allows only GET for `gh api`;
    - every human-only and automation-gated command refuses it, naming
      `runner`.
  - **`VRG_EPIC_SWEEP` is gone.** Remove it from `ops-epic-sweep.yml` in a
    `vergil-actions` follow-up PR filed by this task; it is harmless if left
    in place.
  - **`conftest._hermetic_identity`** also deletes `GITHUB_ACTIONS`, so the
    suite stays hermetic on CI.

- [ ] **Step 1: Write the tests:**
  - `test_undeclared_runner_resolves_runner`
  - `test_human_declared_on_runner_is_runner_with_disagreement`
  - `test_runner_refused_by_release_and_update_deps`
  - `test_runner_vrg_gh_agent_deny_list`
  - `test_epic_sweep_env_no_longer_authorises`
- [ ] **Step 2: Run** the tests. Expected: FAIL.
- [ ] **Step 3: Implement it.** Update T1's
  `test_undeclared_runner_still_resolves_human`, which now expects RUNNER.
- [ ] **Step 4: Run** `vrg-container-run -- vrg-validate`. Expected: PASS.
- [ ] **Step 5: Commit.** `vrg-commit --type fix --scope identity --message "fail closed on an undeclared CI runner instead of resolving to human"`

### Task D7: Deployment: release `vergil-tooling` with the cutover [`vergil-tooling`, `--kind deployment`]

**Blocked by:** T11.

**Precondition (human-attested):** a `vergil-tooling` release containing T11
has been cut and promoted to `v2.1`.

**Procedure:**

1. The next scheduled `ops.yml` runs in `vergil-tooling`, `vergil-containers`
   and `vergil-python` are all green. `epic-sweep`, `github-config` and
   `lifecycle` each report `automation` with their grant.
2. A scratch workflow on a branch, with no declaration, runs `vrg-whoami
   --explain`, which reports `runner`, and `vrg-release`, which is refused.
   Delete the branch afterwards.

**Acceptance:**

- Both observations are recorded.

## Bootstrap order

Each arrow is a `Blocked-by`.

```text
T1 ─┬─▶ T7 ─▶ T8 ─▶ T9 ─▶ T10 ─▶ D4 ─┬─▶ A3 ─▶ D5 ─▶ C3 ─┐
    │                                ├─▶ C1 ─────────────┼─▶ D6 ─▶ V1 ─▶ T11 ─▶ D7
T2 ─▶ T3 ─┬─▶ T4                     └─▶ P2 ─────────────┘
          ├─▶ T5 ─▶ T6
          │
T1..T6 ─▶ D1 ─┬─▶ A1 ─┐
              └─▶ A2 ─┴─▶ D2 ─▶ P1 ─▶ D3      (D2 also gates T11)
C2: independent ─▶ D6
```

Why this order is deadlock-free and safe:

1. **Multi-package support ships before anyone needs it.** Single-package
   repos are unchanged at D1 and D2, so nothing in the fleet moves until
   vergil-python's P1 opts in.
2. **Automation is opt-in until every ops workflow declares it.** The
   fail-closed cutover (T11) comes after D2 (the workflows have migrated) and
   V1 (the chain is proven), so the cutover can't be confused with chain
   failures.
3. **The driver is built and released (D4) before any workflow calls it.**
   It runs in production only after D6's human attestations: the signing
   environment, the App's scope, and a dry run.
4. **The first production run is a real lifecycle change.** The backfill
   generates real `runtime_published` lifecycle PRs, so V1 exercises
   merge-lifecycle, develop-CD, both unattended releases and the
   update-runtime path end to end. A GA add or EOL drop on the vergil-containers manifest goes
   through the same merge-lifecycle stage. Its manifest edits are already
   automated and tested from epic #407.

## Task filing map (epic-create step 9)

| Task | Repo | Kind | Blocked by |
|---|---|---|---|
| T1 Automation identity, grants and scopes | vergil-project/vergil-tooling | task | — |
| T2 `[[package]]` configuration | vergil-project/vergil-tooling | task | — |
| T3 Package dimension in matrix and builds | vergil-project/vergil-tooling | task | T2 |
| T4 Name-aware index and per-package retention | vergil-project/vergil-tooling | task | T3 |
| T5 Build/skip classification | vergil-project/vergil-tooling | task | T3 |
| T6 Per-package install-test, runtime check and index wait | vergil-project/vergil-tooling | task | T5 |
| D1 Release vergil-tooling (automation opt-in, multi-package) | vergil-project/vergil-tooling | deployment | T1–T6 |
| A1 Package dimension in CI and CD | vergil-project/vergil-actions | task | D1 |
| A2 Ops workflows declare automation | vergil-project/vergil-actions | task | D1 |
| D2 Release vergil-actions | vergil-project/vergil-actions | deployment | A1, A2 |
| P1 Multi-series vergil-python | vergil-project/vergil-python | task | D2 |
| D3 Release vergil-python with two series | vergil-project/vergil-python | deployment | P1 |
| T7 Unattended merge rule | vergil-project/vergil-tooling | task | T1 |
| T8 Update-deps window and rule | vergil-project/vergil-tooling | task | T1, T7 |
| T9 vrg-release on a runner | vergil-project/vergil-tooling | task | T1, T7, T8 |
| T10 vrg-lifecycle driver | vergil-project/vergil-tooling | task | T5, T7, T8, T9 |
| D4 Release vergil-tooling (merge rule, release, driver) | vergil-project/vergil-tooling | deployment | T7–T10 |
| P2 Hook reconciles series against the window | vergil-project/vergil-python | task | P1, D4 |
| C1 Per-family merge switch and App commits | vergil-project/vergil-containers | task | D4 |
| C2 Docker-publish cancel race | vergil-project/vergil-containers | task | — |
| A3 Reusable ops-lifecycle.yml | vergil-project/vergil-actions | task | D4 |
| D5 Release vergil-actions (ops-lifecycle) | vergil-project/vergil-actions | deployment | A3 |
| C3 Call ops-lifecycle from ops.yml | vergil-project/vergil-containers | task | C1, D5 |
| D6 Release vergil-containers and vergil-python, plus attestations | vergil-project/vergil-containers | deployment | C1, C2, C3, P2 |
| V1 First unattended chain run (backfill 3.12 and 3.13) | vergil-project/vergil-containers | validation | D6 |
| T11 Fail-closed runner cutover | vergil-project/vergil-tooling | task | D2, V1 |
| D7 Release vergil-tooling (cutover) | vergil-project/vergil-tooling | deployment | T11 |

**Bookends already filed:**

- `vergil-project/.github#444`: docs, closed by this PR.
- `vergil-project/vergil-tooling#3523`: documentation review, blocked by every
  row above.
- `vergil-project/.github#445`: follow-on brainstorm. It carries the
  must-file ruleset item.
- `vergil-project/.github#446`: retrospective (terminal).

**Related intake:** `vergil-project/vergil-tooling#3520`, `#3522` and `#3524`.

## Notes for alignment

- **The hook always gets the window (T8).** The spec (§5) kept the old
  hand-run behaviour "without a window". In the plan, `vrg-update-deps` always
  passes the released window, so a hand-run update reconciles the series list
  too. That's simpler (one behaviour), and it is the same "the series list
  follows the window" rule. Alignment should confirm or revert it.
- **No live GA or EOL during the epic.** Python 3.16 goes GA around October
  2027, and the next Python EOL (3.12) is around October 2028. V1 proves the
  chain with the real `runtime_published` lifecycle PRs that the backfill
  produces. That uses the same merge-lifecycle, develop-CD and release stages
  an add or drop would. The add and drop manifest edits are already covered
  by epic #407's tests. The first live GA run is recorded as a follow-on
  observation in the retrospective.
