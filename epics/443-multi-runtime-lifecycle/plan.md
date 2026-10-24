# Multi-series runtimes and a fully automated lifecycle: implementation plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** `vergil-python` publishes and patches a runtime for every Python
version in the window. A scheduled, unattended workflow carries every
Python lifecycle change from detection to a published runtime, and alerts a
human only when a gate fails.

**Architecture:**

- **`vergil-tooling`** gains these:
  - **One entry point each for `gh` and git**, enforced by guard tests (T15).
  - **Multi-package support.** `[[package]]`, a package dimension in the
    matrix and manifest, name-aware index matching, a `build`/`skip`
    classification, and per-package retention.
  - **An `automation` identity**, with an allowlist of shared workflows
    (keyed on their own `job.workflow_ref`), a scope per operation, identity
    forwarding into containers, and a later fail-closed cutover for undeclared
    runners.
  - **Unattended-merge rules**: one for automation PRs, and one for release
    PRs.
  - **A refreshing App token source**, so no credential outlives an hour.
  - **A lazy, budgeted package index**, and promote gated on publish success.
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

> **Amended after pushback round 2** (2026-10-24, `vergil-project/.github#456`).
> New tasks: T12 (lazy index collect), T13 (App token source), T14 (promote
> requires publish success) and K1 (`packages` retention budget). Reworked:
> T1, T2, T4–T11, A1–A3, P1, P2, C2, C3, D1–D6, V1 and V2. The full list is in
> "Decisions recorded in pushback round 2" at the end.
>
> **Amended again after alignment round 2 and pushback round 3.**
>
> - New tasks: T15 (one `gh`/git entry point), A4, T16 (the tracking issue
>   is the release's state) and T17 (human-only prune).
> - Reworked for the "retain only what is in use" principle: T4, T12 and K1.
> - Each pass's decisions are in its own "Decisions recorded in …" section.

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
  overlay path and index output stay exactly as they are. There are two
  deliberate exceptions:
  - an already-published `(name, version)` becomes a logged skip instead of a
    refusal (§4.3);
  - manifest entries gain `name` and `version` (T3). That is additive, and
    old readers ignore unknown keys.
- **Auto-merge applies to Python only** (§6.4 step 2). Go and every other
  family stay `merge: "human"`.
- **Never fake a precondition.** Releases are human-gated until V1. A
  deployment task attests the release; it never cuts it.
- **Identity gates allow listed modes** (§6.1). Write a gate as `is_human()`,
  an explicit set of allowed modes, or `is_authorised(scope)`. Never write it
  as `not is_agent()` or `!= USER`.
- **No credential outlives an hour** (§6.2). Code that runs unattended gets its
  GitHub token from T13's `app_token.current(purpose)` on every call, through
  `run_gh(..., purpose=…)`, which defaults to `chain`. It never captures a
  token once.
- **Automation env names never reuse `VRG_APP_ID`.** That variable makes
  `identity_mode` resolve to USER. The automation App uses
  `VRG_AUTOMATION_APP_CLIENT_ID`, `VRG_AUTOMATION_APP_KEY_FILE` and
  `VRG_AUTOMATION_APP_SLUG`.

## Review Focus

These are conditions the spec implies that no task's ordinary tests would
exercise, most likely first. Each has a pinned test in the task named.

1. **A package that is published for some targets but not others** (a
   hand-deleted asset, or a newly added arch). Only the missing identities are
   built, with a loud notice naming them. The published ones are never
   rebuilt (spec §4.3, pushback round 3). Test: T5
   `test_classify_partial_builds_only_missing_identities`.
2. **A human or agent pushes onto an automation branch after the App did**
   (`lifecycle/*` or `deps/update-*`). The merge must be refused, and a push
   that lands after the check must make the merge itself fail. Tests: T7
   `test_refuses_pr_with_foreign_commit`, `test_merge_passes_match_head_commit`.
3. **`automation` declared where it doesn't belong:**
   - from an agent VM or the Mac (an agent signal is present);
   - off a runner;
   - from a shared workflow that isn't on the allowlist, or that declares a
     different scope;
   - inside a container whose host was not authorised.

   Each must be refused, with the reason named. Tests: T1
   `test_automation_refused_with_agent_signal`,
   `test_automation_requires_ci_runner`,
   `test_scope_refused_for_unlisted_shared_workflow`,
   `test_container_env_forwards_identity_signals`.
4. **The driver is re-run after the vergil-containers release succeeded but
   the vergil-python step failed.** It must resume at the vergil-python step.
   It must not cut a second vergil-containers release, because no lifecycle
   merge is outside the newest `vX.Y.Z` tag's ancestry. It must not close the
   vergil-python failure issue until that stage succeeds. Tests: T10
   `test_rerun_after_containers_release_resumes_at_runtime`,
   `test_failure_issue_survives_a_deferred_run`.
5. **Releases published before this epic** (no `name` in their manifest
   entry, and no `packages-plan.json`) must still be collected and matched. A
   series that is still **in use** (named in the newest release's plan, even
   as `skip`) must be kept, even though its last build was several releases
   ago. A series dropped from the window must disappear after its grace
   period. Tests: T4 `test_old_manifest_shape_still_matches`,
   `test_in_use_skipped_series_is_kept`,
   `test_dropped_series_gone_after_grace`.
6. **A release interrupted on one machine is resumed on another** (a runner
   dies; the next run is a fresh clone). It must continue from the tracking
   issue's state block alone. Test: T16
   `test_resume_on_fresh_clone_after_merge_release`.

---

### Task T1: The `automation` identity, allowlist and scopes (opt-in) [`vergil-tooling`]

**Blocked by:** none.

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
  `bin/vrg_pr_fix_body.py:67`, `bin/vrg_audit_approve.py:38`,
  `bin/vrg_gh.py` (`_get_denied_pairs`, api rule), `bin/vrg_whoami.py`,
  `lib/platform_env.py` (`_correlation_disagrees`), `lib/container.py`
  (the env passthrough at :340-354)
- Test: `tests/vergil_tooling/test_identity_mode.py`,
  `test_automation.py` (new), `test_identity_gates.py` (new), `test_vrg_gh.py`,
  `test_vrg_release.py`, `test_vrg_update_deps.py`, `test_vrg_epic_audit.py`,
  `test_vrg_finalize_pr.py`, `test_vrg_submit_pr.py`, `test_vrg_whoami.py`,
  `test_platform_env.py`, `test_container.py`

**Interfaces:**

- Produces, in `identity_mode.py`:
  - `IdentityMode.AUTOMATION = "automation"`, and `Signal.CI_RUNNER = "ci-runner"`.
  - `def on_ci_runner() -> bool`, which is `os.environ.get("GITHUB_ACTIONS") == "true"`.
  - `resolve()` accepts `VRG_IDENTITY_MODE=automation` **only** when
    `on_ci_runner()` is true and no **agent signal** is present: mode file,
    `app.pem`, `VRG_APP_ID`, or the `/vergil` mount. Otherwise it raises
    `IdentityModeError` naming the conflicting signal. It never falls back to
    another mode. The platform (`cloud-vm` on a hosted runner) is **not** an
    agent signal (§6.1).
  - `is_human()` stays `mode is HUMAN`, and `is_agent()` stays `mode in
    {USER, AUDIT}`.
- Produces, in `automation.py`:

  ```python
  SCOPE_RELEASE = "release"; SCOPE_UPDATE_DEPS = "update-deps"; SCOPE_MERGE = "merge"
  SCOPE_LIFECYCLE = "lifecycle"; SCOPE_EPIC_SWEEP = "epic-sweep"; SCOPE_REPO_CONFIG = "repo-config"

  JOB_WORKFLOW_REF_ENV = "VRG_JOB_WORKFLOW_REF"   # exported by each shared workflow from ${{ job.workflow_ref }}

  @dataclass(frozen=True)
  class AutomationGrant:
      workflow: str            # fnmatch pattern over VRG_JOB_WORKFLOW_REF: the *shared* (called) workflow's own ref, never the caller's
      scopes: frozenset[str]

  class AutomationRefused(Exception): ...           # message names mode, job workflow ref and scope

  def load_grants() -> tuple[AutomationGrant, ...]  # packaged data file; malformed file -> ConfigError
  def is_authorised(scope: str) -> bool             # mode is AUTOMATION and a grant matches VRG_JOB_WORKFLOW_REF + scope
  def require(scope: str) -> None                   # raises AutomationRefused unless is_authorised
  ```

- **Why the shared workflow's ref** (§6.1, decided in pushback round 2).
  `GITHUB_WORKFLOW_REF` names the *calling* file. That differs per org and per
  filename, and one caller hosts many jobs. `job.workflow_ref` names the shared
  vergil-actions workflow, which is the same in every org and holds exactly
  one scope.
- `automation_workflows.toml` initial content. These are the shared
  workflows that run gated commands:

  ```toml
  [[grant]]
  workflow = "vergil-project/vergil-actions/.github/workflows/ops-epic-sweep.yml@refs/tags/v*"
  scopes = ["epic-sweep"]

  [[grant]]
  workflow = "vergil-project/vergil-actions/.github/workflows/ops-lifecycle.yml@refs/tags/v*"
  scopes = ["lifecycle", "release", "update-deps", "merge"]
  ```

  `ops-github-config.yml` gets no grant, because its jobs call no gated
  command (the Step 5 sweep confirms this).
- **Identity in containers:** `vrg-container-run` always forwards
  `GITHUB_ACTIONS` and `VRG_JOB_WORKFLOW_REF` when they are set. It forwards
  `VRG_IDENTITY_MODE=<the host's resolved mode>`, from `current_mode()` on the
  host, rather than the raw variable (spec §6.1, pushback round 3). On an
  agent VM the mode file decides and the variable is often unset. These go
  through the same passthrough as `VRG_VERSION_WINDOW`
  (`lib/container.py:340-354`), regardless of `[container].env-prefixes`.
  Test: `test_container_forwards_resolved_mode_when_env_unset` (a mode file
  says `user`, there's no env var, and the container gets
  `VRG_IDENTITY_MODE=user`).
- **`VRG_JOB_WORKFLOW_REF` is set in step-level `env:`.** `job.workflow_ref`
  isn't available in `jobs.<id>.env` (verified). A2 and A3 follow this.

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
    it that way (`lib/release/finalize.py` `_FINALIZE_CMD`). This gate is new:
    `vrg-finalize-pr` has none today.
  - `vrg-pr-fix-body` and `vrg-audit-approve` are rewritten as explicit
    allowed sets: today's allowed modes, written out. So AUTOMATION (and T11's
    RUNNER) are refused.
- **`vrg-gh` under AUTOMATION:**
  - `_get_denied_pairs` uses `_DENIED_AGENT`;
  - `gh api` is allowed for GET only, using the existing `_api_is_get`;
  - nothing is added back (§6.1, decided in pushback round 2). The wrapper
    lists aren't the automation boundary, because no runner path calls the
    wrappers.
- **`vrg-whoami --explain`** reports `automation` together with the matching
  grant (or "no grant").
- **`platform_env._correlation_disagrees`** treats `cloud-vm` with
  AUTOMATION on a CI runner as agreeing, so a legitimate run doesn't warn.

- [ ] **Step 1: Write the identity tests** in `test_identity_mode.py`:
  - `test_automation_resolves_on_ci_runner`: with `GITHUB_ACTIONS=true` and
    `VRG_IDENTITY_MODE=automation`, `current_mode() is IdentityMode.AUTOMATION`.
  - `test_automation_requires_ci_runner`: same env without `GITHUB_ACTIONS`
    raises `IdentityModeError`, and the message contains `"not a CI runner"`.
  - `test_automation_refused_with_agent_signal`: parametrised over every
    agent signal (the mode file, `app.pem`, `VRG_APP_ID` and the `/vergil`
    mount). Each one raises, naming the signal. `/vergil` is read from
    `platform_env` today, so `identity_mode` gains that check.
  - `test_whoami_explain_reports_refused_automation`: `--explain` prints the
    refusal reason, not a traceback.
  - `test_undeclared_runner_still_resolves_human`: with `GITHUB_ACTIONS=true`
    and no mode set, the mode is HUMAN (unchanged until T11).
- [ ] **Step 2: Write the grant tests** in `test_automation.py`:
  - `test_scope_granted_for_listed_shared_workflow`:
    `VRG_JOB_WORKFLOW_REF=vergil-project/vergil-actions/.github/workflows/ops-lifecycle.yml@refs/tags/v2.1`
    gives `is_authorised("release")`.
  - `test_grant_ignores_caller_workflow_ref`: the same
    `VRG_JOB_WORKFLOW_REF`, with
    `GITHUB_WORKFLOW_REF=logical-minds-foundry/.github/.github/workflows/epic-sweep.yml@refs/heads/main`,
    still authorises. A caller in another org with any filename works.
  - `test_scope_refused_for_unlisted_shared_workflow`:
    `VRG_JOB_WORKFLOW_REF=vergil-project/vergil-actions/.github/workflows/ci-quality.yml@refs/tags/v2.1`.
    `require("release")` raises `AutomationRefused`, and the message contains
    the ref and the scope.
  - `test_scope_refused_outside_grant_scopes`: the epic-sweep ref with
    `require("release")` raises.
  - `test_missing_job_workflow_ref_refuses`.
  - `test_not_authorised_unless_mode_is_automation`: as HUMAN with a matching
    ref, `is_authorised(...)` is False.
  - `test_malformed_grant_file_is_config_error`.
- [ ] **Step 2b: Write the platform and container tests:**
  - `test_platform_env.py::test_cloud_vm_with_automation_on_runner_agrees`.
  - `test_container.py::test_container_env_forwards_identity_signals`: with
    `GITHUB_ACTIONS`, `VRG_IDENTITY_MODE` and `VRG_JOB_WORKFLOW_REF` set on the
    host and no `env-prefixes`, the docker argv carries all three.
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
  - `test_identity_gates.py::test_every_gate_refuses_new_modes`: parametrised
    over every `identity_mode` caller the sweep records. Each one under
    AUTOMATION with no scope refuses, or is listed in the test as deliberately
    ungated, with a reason. T11 adds RUNNER to the parametrisation.
- [ ] **Step 6: Implement the gates.** Then run
  `vrg-container-run -- vrg-validate`. Expected: PASS at 100% coverage.
- [ ] **Step 7: Commit.** `vrg-commit --type feat --scope identity --message
  "add the declared automation identity with workflow grants and scopes"`

### Task T2: `[[package]]` configuration [`vergil-tooling`]

**Blocked by:** none.

**Files:**

- Modify: `src/vergil_tooling/lib/config.py` (`PackageConfig`, `VergilConfig`,
  `_parse_package_config`, `_check_package_overlay`,
  `_check_package_python_lock`, `read_config`)
- Modify: the `cfg.package` presence checks in `lib/ci_evidence.py:723` and
  `lib/github_config.py:727`, which become `bool(cfg.packages)`
- Modify: the other single-package readers (§2.3, found in pushback round 2):
  - `lib/update_deps/updaters/vergil_eco.py:102-106` (`_declared_runtime`,
    which reads raw TOML);
  - `lib/versions/extensions/python.py:84,109`;
  - `lib/update_deps/vergil_index.py:187`.
- Modify: `lib/update_deps/orchestrator.py:94-108` (catch-all for updater
  crashes)
- Test: `tests/vergil_tooling/test_config.py` (the package section from :1774),
  `test_update_deps_vergil_eco.py`, `test_update_deps_orchestrator.py`,
  `test_versions_extension_python.py`, `test_update_deps_vergil_index.py`

**Why this ships at D1:** D3's hand release of the two-series vergil-python
runs `vrg-release`'s closing `update-deps` stage. With the raw-TOML read
unchanged, that stage would crash with `AttributeError` on `[[package]]`.

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
  - `def packages_from_toml(data: Mapping[str, Any], source: str) ->
    tuple[PackageConfig, ...]`. It parses an already-loaded `vergil.toml`
    mapping, for readers that hold text rather than a repo (`vergil_eco`).
  - `vergil_eco._declared_runtime(text)` uses `packages_from_toml`, then
    collects `pkg.python.runtime` over python-builder packages:
    - none → `None`;
    - all equal → that value;
    - different → `UpdateDepsError` naming each entry and its pin.
  - **Updater crashes are never silent.** The orchestrator's updater loop
    gains a final `except Exception as exc`, which becomes
    `DepsFailure(kind="update", message=f"{updater.name}: unexpected
    {type(exc).__name__}: {exc}", output_tail=<traceback tail>)`. That gets a
    fix task like any other update failure.
  - Python-extension and `vergil_index` readers iterate `cfg.packages`. A
    python-builder lookup that finds more than one python-builder package
    raises `ConfigError` naming them; no current repo has that.
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
  - `test_update_deps_vergil_eco.py::test_declared_runtime_array_form_staged_only`:
    two staged `[[package]]` entries give `None`, with no exception.
  - `test_declared_runtime_conflicting_python_pins_errors`.
  - `test_vergil_updater_apply_on_array_form_repo`: the whole
    `VergilUpdater.apply` runs on a two-`[[package]]` fixture repo without
    error.
  - `test_update_deps_orchestrator.py::test_unexpected_updater_exception_becomes_fix_task_failure`:
    an updater raising `AttributeError` gives `ctx.failure.kind == "update"`,
    and the message names the updater and the exception type.
- [ ] **Step 2: Run** `vrg-container-run -- uv run pytest tests/vergil_tooling/test_config.py -v`. Expected: the new tests FAIL.
- [ ] **Step 3: Implement it.** Keep `_parse_package_config` returning a single
  `PackageConfig` for one table. Add `_parse_packages(raw, source) ->
  tuple[PackageConfig, ...]`, which dispatches on table versus list and
  raises `ConfigError` when both forms appear.
- [ ] **Step 4: Run** `vrg-container-run -- vrg-validate`. Expected: PASS. Every
  existing package test is unchanged.
- [ ] **Step 5: Commit.** `vrg-commit --type feat --scope package --message "accept a [[package]] array with per-package overlays"`

### Task T3: Package dimension in matrix, build, staged and nFPM [`vergil-tooling`]

**Blocked by:** T2.

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

**Blocked by:** T3.

**Files:**

- Modify: `src/vergil_tooling/lib/package/index/collect.py` (`ManifestEntry`,
  `_manifest_entries`, `_match`), `retention.py` (`_by_lines`, `select`),
  `config.py` (`IndexConfig`: `grace_hours`, `pin`, `list_limit`), `site.py`
  (`projected_site_bytes`)
- Test: `test_package_index_collect.py`, `test_package_index_retention.py`,
  `test_package_index_site.py`

**Interfaces:**

- Consumes T3's manifest entry shape: `{"fmt", "arch", "suite", "name",
  "version"}`.
- Produces:
  - The metadata-only types that retention runs on. T12 fills them from
    GitHub.

    ```python
    @dataclass(frozen=True)
    class RemoteArtifact:
        product: str; tag: str; asset: str     # release-asset file name
        identity: Identity                     # from published.parse_filename
        size: int; digest: str | None          # "sha256:" prefix stripped; None when GitHub has none

    @dataclass(frozen=True)
    class RemoteRelease:
        product: str; tag: str; created_at: datetime
        plan_names: frozenset[str] | None      # names in packages-plan.json (build or skip); None if no plan asset
        manifest_names: frozenset[str] | None  # names in packages-manifest.json; the pre-plan fallback
        artifacts: tuple[RemoteArtifact, ...]
    ```

  - `ManifestEntry.name: str | None = None` and `ManifestEntry.version: str |
    None = None`.
  - **`pattern()` when `name` and `version` are set** matches the **exact
    filename** (§4.2, decided in pushback round 2), not a name prefix:
    - deb: `re.escape(f"{name}_{version}-{rev}_{arch}.deb")`
    - rpm: `re.escape(f"{name}-{version}-{rev}.{rpmarch}.rpm")`

    When `name` is None the old pattern applies, unchanged.
  - `_match` keeps its rule of exactly one file per entry and no files left
    unclaimed.
  - **Retention: in use only** (spec §4.4, principle 10; reworked in pushback
    round 3):
    - `_by_lines` groups by `(product, name)` instead of `product`, keeping its
      line semantics (`keep` per line, `lines` lines). Each name's lines come
      from the tags that carry that name.
    - **The in-use set.** A product's in-use names are the names listed in its
      **newest stable release's `packages-plan.json`**, as `build` or `skip`
      (T5). For a release with no plan (pre-epic), they are the names in its
      manifest.
      - `def in_use_names(releases: Sequence[RemoteRelease]) -> frozenset[str]`
        is per product.
      - A name **not** in the in-use set is dropped, once its grace has passed.
      - A runtime series that was refreshed long ago but is still listed as
        `skip` stays in use.
    - `select` keeps, for each in-use name not in `retire`, the newest
      `keep` versions within the newest `lines` lines. It keeps **every
      artifact of each kept name-version**, never "its whole release".
    - **Pins** (spec §4.4, alignment round 3): `IndexConfig.pins:
      tuple[Pin, ...]` is parsed from an optional `[retention] pin = [...]` of
      `"name=version"` or bare `"name"`. Pinned names (and exact versions)
      count as in use, whatever the newest release's plan says. A malformed
      entry is a `ConfigError`.
    - **The grace copy, budget-aware** (spec §4.4, alignment round 3).
      `IndexConfig.grace_hours: int = 48` comes from an optional `[retention]
      grace_hours`. These are kept until `now - createdAt(newer release) >=
      grace_hours`:
      - the version each kept name had **before** its newest;
      - every artifact of a name that just left the in-use set.

      But a grace copy is kept only while the selected total (with
      `projected_site_bytes`) stays under `SIZE_WARN`. Otherwise grace copies
      are dropped, the largest name-version first, each printed as `grace
      dropped: <name> <version> (budget)`. `createdAt` comes from release
      metadata (T12).
    - **Config ownership:** the new `[retention]` keys (`grace_hours`, `pin`,
      and T12's `list_limit`) are parsed in `lib/package/index/config.py`
      `IndexConfig`, which this task owns.
  - `def projected_site_bytes(kept: Sequence[RemoteArtifact]) -> int`, in
    `site.py`: the sum of the kept artifacts' sizes, plus the
    metadata-overhead constant `SITE_OVERHEAD_BYTES = 5_000_000`.

- [ ] **Step 1: Write the tests:**
  - `test_two_packages_same_arch_match_by_name`: a manifest with
    `vergil-python3.14` and `vergil-python3.15` for deb/amd64 matches both
    files.
  - `test_old_manifest_shape_still_matches`: a fixture manifest with no
    `name` for a single-package release collects exactly as before.
  - `test_old_shape_two_packages_still_errors`: the error text
    `"has 2 packages for deb/amd64"` is kept.
  - `test_in_use_skipped_series_is_kept`: product `vergil-python` with tags
    v1.0.4 (3.14 build, 3.15 build) and v1.0.5–v1.0.9 (3.14 skip, 3.15
    build), `keep=1`. The 3.14 artifacts from v1.0.4 are still selected,
    because v1.0.9's plan lists 3.14 as `skip`.
  - `test_dropped_series_gone_after_grace`: v1.1.0's plan omits 3.12, so
    3.12 is kept for 48 h after v1.1.0's `createdAt`, then dropped.
  - `test_previous_version_kept_within_grace`: with `keep=1`, the version
    before the newest is kept until `grace_hours` after the newer release's
    `createdAt`.
  - `test_grace_copies_dropped_largest_first_over_size_warn`: four series of
    134 MB are refreshed at once. Grace copies are dropped until the total is
    under `SIZE_WARN`, and each one dropped is printed.
  - `test_pinned_old_version_survives_keep_one`:
    `pin = ["vergil-tooling=2.1.40"]`, with v2.1.45 newest.
  - `test_bare_name_pin_keeps_newest_of_name`: `pin = ["vergil-python3.14"]`
    while the newest plan lacks 3.14.
  - `test_malformed_pin_is_config_error`.
  - `test_kept_name_never_pins_whole_release`: keeping 3.14's artifacts from
    v1.0.4 doesn't keep v1.0.4's 3.15 artifacts.
  - `test_retire_still_drops_by_name`: the existing `[retention].retire`
    behaviour is unchanged.
  - `test_exact_filename_match_no_prefix_collision`: packages `foo` and
    `foo-bar` in one release each match only their own files.
  - `test_single_package_line_keep_one`: vergil-tooling-shaped tags across
    two lines with `keep=1, lines=1` select only the newest 2.1 release
    (plus its grace copy).
  - `test_grace_hours_bad_value_is_config_error`.
  - `test_projected_site_bytes_sums_kept_sizes`.
  - `test_site_index_contains_both_series`.
- [ ] **Step 2: Run** the three test files. Expected: FAIL.
- [ ] **Step 3: Implement it.**
- [ ] **Step 4: Run** `vrg-container-run -- vrg-validate`. Expected: PASS.
- [ ] **Step 5: Commit.** `vrg-commit --type feat --scope package-index --message "match manifest entries by package name and retain only in-use packages"`

### Task T5: The `build`/`skip` classification in preflight and CD [`vergil-tooling`]

**Blocked by:** T3.

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
      action: Literal["build", "skip", "complete"]
      identities: tuple[Identity, ...]        # every identity of the package
      missing: tuple[Identity, ...] = ()      # for "complete": only these are built

  def classify(repo_root: Path, cfg: VergilConfig, product: str, version: str,
               gh: Run = local_run) -> list[PackagePlan]
  def render_plan(plans: Sequence[PackagePlan]) -> str   # one line per package: "build  <name> <version>" / "skip   <name> <version> (published)"
  ```

- **The classification rule:**
  - **`skip`:** every identity for the package is already in
    `published(product, exclude_tag=f"v{version}")`.
  - **`build`:** none of them is.
  - The driver (T10) and preflight treat `complete` like `build` when they
    decide whether to release.
  - **Some but not all → `complete`** (spec §4.3, pushback round 3, Review
    Focus 1):
    - `missing` lists the unpublished identities;
    - `render_plan` prints `complete <name> <version>: <fmt/arch …> missing`;
    - `vrg-package matrix --plan` emits **only** the cells that produce a
      missing identity;
    - the published identities are never rebuilt.
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
  to `$GITHUB_OUTPUT`, and writes **no** manifest file (§4.3, pushback round
  2). An empty `artifacts` list would break every later index rebuild.
- **The plan file is a release asset.** `packages-plan.json` (the
  `--plan-out` JSON) is attached to every release of a repo that has
  packaging configured, including all-skip releases (A1). T12's collect reads
  it to tell "all packages skipped" from "pre-packaging release".

- [ ] **Step 1: Write the tests:**
  - `test_classify_unpublished_is_build`
  - `test_classify_published_is_skip`
  - `test_classify_partial_builds_only_missing_identities`: deb/arm64 and
    rpm/aarch64 missing → `complete`, and the matrix holds only the arm64
    cells
  - `test_complete_is_printed_loudly`
  - `test_classify_multi_mixed`: 3.14 build, 3.15 skip
  - `test_single_package_published_is_now_skip`, the documented behaviour
    change
  - `test_preflight_records_plan_and_prints_skips`
  - `test_preflight_all_skipped_warns_and_proceeds`
  - `test_check_unpublished_writes_plan_and_summary`
  - `test_matrix_plan_omits_skipped_cells`
  - `test_matrix_plan_all_skipped_disables`
  - `test_matrix_plan_all_skipped_writes_no_manifest`
- [ ] **Step 2: Run** the test files. Expected: FAIL.
- [ ] **Step 3: Implement it.** Update the `check_unpublished` docstring and
  the advice that names `[package].version`.
- [ ] **Step 4: Run** `vrg-container-run -- vrg-validate`. Expected: PASS.
- [ ] **Step 5: Commit.** `vrg-commit --type feat --scope package --message "classify packages as build or skip instead of refusing a published version"`

### Task T6: Per-package install-test, check-runtime and release index wait [`vergil-tooling`]

**Blocked by:** T5.

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
    - calls `runtime.check_consistent(runtime, packages, where)` once per name
      that has a recorded runtime, that is, python-builder packages only.
      Packages from other builders, including the runtime packages
      themselves, record none and are skipped (§4.2).
  - **The release index wait** loops over the `build` **and `complete`**
    entries in `ctx.package_plan`, with one deadline for all of them. For a
    `complete` entry it waits on its missing identities.
  - `package_index.visible(name: str, version: str) -> bool` is the `_wait`
    predicate, extracted so that it can be called once with no wait. T10
    uses it for its index-visibility proof.
  - `ReleaseContext.package_index_urls: list[str]`. `package_index_url` stays
    set for single-package repos, for the existing summary at
    `orchestrator.py:304`.

- [ ] **Step 1: Write the tests:**
  - `test_install_test_uses_cells_package_smoke`
  - `test_check_runtime_per_package_series`: two packages on different
    `vergil-python<x.y>` pass
  - `test_check_runtime_mismatch_within_one_package_fails`
  - `test_check_runtime_ignores_packages_without_runtime`
  - `test_index_wait_waits_for_each_built_package`
  - `test_index_wait_skips_skipped_packages`
  - `test_index_wait_single_package_sets_url`
- [ ] **Step 2: Run** the tests. Expected: FAIL.
- [ ] **Step 3: Implement it.**
- [ ] **Step 4: Run** `vrg-container-run -- vrg-validate`. Expected: PASS.
- [ ] **Step 5: Commit.** `vrg-commit --type feat --scope package --message "install-test, runtime-check and index-wait per package"`

### Task T15: One entry point each for `gh` and git, with guards [`vergil-tooling`]

**Blocked by:** none.

Added in alignment, round 2 (spec §6.2). Today `gh` is executed ad hoc from
`lib/github.py`, `lib/package/index/collect.py` (5 sites; `published.py` goes
through the same runner), `bin/vrg_finalize_pr.py` (2),
`lib/release/subprocess.py` (1), `lib/github_config.py` (1) and
`bin/vrg_gh.py` (1). A credential, or any other cross-cutting concern, can't
be injected in one place. The human rejected a PATH shim, which would mask
the inconsistency. This task consolidates instead.

**Files:**

- Modify: `src/vergil_tooling/lib/github.py` (a new `run_gh`), and every site
  above
- Modify: `lib/git.py`, plus every module that execs `git` outside it (the
  implementer lists them in the PR body)
- Create: `tests/vergil_tooling/test_exec_entry_points.py`
- Test: the existing tests of every migrated module

**Interfaces:**

- Produces, in `lib/github.py`:

  ```python
  def run_gh(args: Sequence[str], *, check: bool = True, capture: bool = True,
             timeout: float | None = None, cwd: Path | None = None,
             env: Mapping[str, str] | None = None) -> subprocess.CompletedProcess[str]
      # The only code that execs gh. Builds the env with _gh_env(env); T13 adds
      # credential injection inside _gh_env and nowhere else.
  def _gh_env(extra: Mapping[str, str] | None) -> dict[str, str]
  ```

- `bin/vrg_gh.py` applies its allow and deny policy first, then execs through
  `run_gh` (passthrough I/O, with `capture=False`). Policy and credentials stay
  separate.
- `lib/git.py` exposes `run_git(args, *, check, capture, timeout, cwd)` as the
  only code that execs git.
- **The guards** in `test_exec_entry_points.py`:
  - `test_gh_is_executed_only_through_run_gh`: an AST scan of `src/` for
    `subprocess.*`, `os.exec*` and `shutil.which("gh")` calls whose argv
    starts with `"gh"`, outside `lib/github.py`. A hit fails, naming the file
    and line.
  - `test_git_is_executed_only_through_run_git`: the same for `"git"`,
    outside `lib/git.py`.

- [ ] **Step 1: Write the two guard tests.**
- [ ] **Step 2: Run** them. Expected: FAIL, listing today's ad-hoc sites.
- [ ] **Step 3: Add `run_gh` and `run_git`, and migrate every listed site.**
  Behaviour is unchanged: same argv, same env, same error handling.
- [ ] **Step 4: Run** `vrg-container-run -- vrg-validate`. Expected: PASS at
  100% coverage, with every pre-existing test unchanged.
- [ ] **Step 5: Commit.** `vrg-commit --type refactor --scope exec --message "route every gh and git execution through one entry point each"`

### Task T12: Lazy, bounded index collect, with closure and budget [`vergil-tooling`]

**Blocked by:** T4.

Added in pushback round 2 (§4.4). Today `collect` downloads and verifies every
artifact of every stable release before retention runs. With all-series
releases of about 560 MB, that fills a hosted runner's disk of about 14 GB
within roughly 25 releases.

**Files:**

- Modify: `src/vergil_tooling/lib/package/index/collect.py` (`collect`, and the
  per-release download at :357-358), `site.py` (the collect → verify → select
  order at :155-159), `retention.py` (`_check_conflicts`)
- Modify: `lib/package/index/published.py` (reuse `parse_filename`)
- Test: `test_package_index_collect.py`, `test_package_index_site.py`,
  `test_package_index_retention.py`

**Interfaces:**

- Consumes T4's `ManifestEntry`, `RemoteRelease`, `RemoteArtifact`,
  `in_use_names`, `IndexConfig.grace_hours`/`pins`, and T5's
  `packages-plan.json` asset.
- Produces:

  ```python
  LIST_LIMIT_DEFAULT = 20   # first page of newest stable releases per product
  LIST_HARD_CAP = 200       # adaptive paging never goes past this (spec §4.4, alignment round 3)

  def list_remote(product: str, *, limit: int | None = LIST_LIMIT_DEFAULT,
                  needed: Callable[[Sequence[RemoteRelease]], frozenset[str]] = lambda _: frozenset(),
                  gh: Run = local_run) -> list[RemoteRelease]
      # metadata only: releases (name, size, digest, createdAt), plus each release's two small JSON
      # assets (packages-plan.json, packages-manifest.json). No package bytes.
      # Pages past `limit` while `needed(listed)` names something not yet listed; raises IndexError
      # naming the missing names past LIST_HARD_CAP. limit=None lists everything (T17 only).
  def fetch(kept: Sequence[RemoteArtifact], dest: Path, gh: Run = local_run) -> list[Artifact]
  def close_over(kept: Sequence[Artifact], listed: Sequence[RemoteRelease], *, vendor: str,
                 dest: Path, gh: Run = local_run) -> list[Artifact]
      # steps 4–6 below as one function; T17 reuses it
  ```

  - `needed` returns the in-use names, the pins and (in the closure loop)
    the org-owned dependency names without a listed provider.
  - `[retention] list_limit` (default 20) overrides the first page.
  - The existing `RELEASE_LIST_LIMIT = 1000` path is no longer used by the
    index.
- **New `site.py` order, with an iterative closure** (spec §4.4, pushback
  round 3):
  1. `list_remote` for every product;
  2. manifest matching on names (T4);
  3. retention `select` on identities (T4's in-use rule);
  4. `fetch` the kept set, and read each artifact's real `depends` (the
     existing `deb_metadata` / `rpm_metadata`);
  5. **close:** for any **org-owned** dependency name (one starting with
     `f"{vendor}-"`) that no kept artifact provides, add the newest provider
     among the listed identities. If none is listed, page further
     (`list_remote`'s `needed`). Fetch it, and repeat until stable. OS
     dependencies are ignored, as `_close` does today;
  6. `_check_retired` on the closed set. Then assert that every org-owned
     dependency of a kept artifact is provided by the closed set, else
     `IndexError` naming both;
  7. verify (attestation and signature) the closed set;
  8. build.

  Disk use is then bounded by the in-use set and its closure.
- **`retire` scoped to the listing** (spec §4.4, alignment round 3):
  `_check_retire_matches` checks only the listed set. A retired name that
  matches nothing listed prints `warning: [retention].retire lists <name>,
  which matches no listed release; remove it`, and doesn't fail the build.
- **`_check_conflicts`** compares identities across releases by `digest`.
  Only when a digest is absent does it download that asset to hash it, then
  delete the asset. It logs a count of digest-less assets.
- **A release with `packages-plan.json` but no `packages-manifest.json`** is
  logged as `"<product>@<tag>: all packages skipped"`. A release with neither
  keeps the existing `"pre-packaging release — skipped"` log.
- `projected_site_bytes` (T4) is computed from `RemoteArtifact.size` before
  any download, and printed as `"projected site: <N> bytes"`.
- **`vrg-package index --budget-only`** runs steps 1–3, prints the projected
  size per product and in total, and downloads nothing (spec §4.4, reworked in
  pushback round 3).
  - **Headroom is net growth:**
    1. simulate the next release of each product, as its newest release with
       every in-use name at a bumped version and the same sizes;
    2. re-run `select` including that simulated release, with T4's
       budget-aware grace rule applied (grace copies dropped once over
       `SIZE_WARN`), and take the new total minus the current total;
    3. add one allowance of the largest single name-version's bytes, for a
       possible series add.
  - It **exits 1 when total plus headroom reaches `SIZE_FAIL`**. It prints a
    warning line when it reaches `SIZE_WARN`.
  - A1 runs it as its own job, so a breach turns a job red without stopping a
    publish.

- [ ] **Step 1: Write the tests:**
  - `test_list_remote_downloads_nothing`: the fake gh records no package
    downloads, only the two small JSON assets.
  - `test_list_remote_reads_only_newest_limit_releases`: 30 fixture releases
    with `limit=20` and nothing needed list 20.
  - `test_list_remote_pages_past_limit_for_in_use_name`: the kept 3.13
    version is at depth 25. It is listed and kept.
  - `test_list_remote_pages_for_org_dependency_provider`.
  - `test_list_remote_fails_loud_past_hard_cap`.
  - `test_os_dependency_ignored_org_dependency_required`: `libc6` is
    ignored, and an unresolvable `vergil-python3.13` fails.
  - `test_retire_unmatched_in_listing_warns_not_fails`.
  - `test_only_kept_artifacts_are_fetched`: with 10 fixture releases and
    `keep=1`, only the kept artifacts (and their grace copies) are
    downloaded.
  - `test_closure_fetches_missing_provider_iteratively`: a kept
    vergil-tooling depends on `vergil-python3.15`. If the provider isn't in
    the kept set, it is fetched and added.
  - `test_unresolvable_dependency_fails_loud`.
  - `test_retired_dependency_still_refused_after_fetch`.
  - `test_conflict_check_uses_digest`: the same identity with different
    digests in two releases raises the existing conflict error.
  - `test_conflict_check_downloads_only_when_digest_missing`.
  - `test_all_skipped_release_logged_distinctly`.
  - `test_projected_size_printed_before_fetch`.
  - `test_budget_only_exits_1_when_total_plus_headroom_reaches_fail`,
    `test_budget_only_warns_at_size_warn_but_exits_0`,
    `test_budget_only_downloads_nothing`, and
    `test_headroom_is_net_growth_plus_one_allowance`. The last one uses the
    measured fixture: 4 series × 134 MB, plus 28 MB of vergil-tooling, with
    keep=1, which is about 565 MB. A full refresh nets about 0 MB, because its
    grace copies are dropped at `SIZE_WARN`. Adding one 134 MB allowance
    gives about 700 MB, so the command prints no warning and exits 0. The same
    fixture at 5 series (about 700 MB, plus 134 MB, about 834 MB) warns and
    exits 0. At 6 series (about 835 MB, plus 134 MB, which reaches
    `SIZE_FAIL`) it exits 1.
- [ ] **Step 2: Run** the tests. Expected: FAIL.
- [ ] **Step 3: Implement it.**
- [ ] **Step 4: Run** `vrg-container-run -- vrg-validate`. Expected: PASS.
- [ ] **Step 5: Commit.** `vrg-commit --type feat --scope package-index --message "select from release-asset metadata and download only the retained set"`

### Task T14: Promote requires publish success, and never moves backwards [`vergil-tooling`]

**Blocked by:** T15 (it reads CD runs through `run_gh`).

Added in pushback round 2 (§6.2). This applies to **every** `vrg-release` run,
human or automation. It is a deliberate change to human releases of every
repo.

**Files:**

- Modify: `src/vergil_tooling/lib/release/orchestrator.py` (`_promote_phase`
  :204-209, and the summary at :230-234, :295-300), `lib/release/confirm.py`
  (a new `main_publish_failures`), `lib/promote.py` (monotonic check;
  `run_git`), `lib/release/tracking.py:132-150` and
  `lib/release/finalize.py:53-59` (the recovery text)
- Test: `tests/vergil_tooling/test_release_orchestrator.py`,
  `test_release_confirm.py`, `test_release_tracking.py`, `test_promote.py`

**Interfaces:**

- Produces:
  - `confirm.main_publish_failures(repo: str, release_sha: str) -> list[str]`
    reads the **newest attempt** of the `cd.yml` run on `main` for
    `release_sha` through `run_gh` (T15). It returns exactly what the existing
    generic rule returns: `_collect_deferred_publish(jobs)` plus
    `_unsettled_jobs(jobs)` (`confirm.py:349-374`).
    - Every non-release job must be `success` or `skipped`.
    - A skip that follows a real failure always has a failed upstream job.
    - So repos without packages, whose package jobs are `skipped`, pass
      (spec §6.2, clarified in pushback round 3).
  - **`_promote_phase` never trusts memory** (spec §6.2, alignment round 2):
    - it calls `main_publish_failures`, and raises
      `ReleaseError(phase="promote", message=f"refusing to promote v{X.Y}:
      publish job(s) not successful: {jobs}; re-run the failed jobs, then
      vrg-release --resume")` when the list is non-empty;
    - this holds on a first run and on `--resume` alike;
    - develop-side docs failures aren't in that run, so they don't block.
  - **Monotonic promote** (spec §6.2, pushback round 3): before moving it,
    `promote` resolves origin `vX.Y` to its `vX.Y.Z` with `run_git ls-remote`.
    If that is **higher** than the release being promoted, it doesn't move the
    tag. It records `superseded by vX.Y.W` as a warning in the summary and on
    the tracking issue, and the stage counts as done.
  - `lib/promote.py`'s raw `subprocess` git calls move onto `run_git` (T15).
  - `--no-promote` still skips the stage, as today.
  - The recovery texts in `tracking.py` and `finalize.py` change from "do not
    `--resume`" to "re-run the failed publish jobs, then `vrg-release
    --resume`; promote re-verifies".
- The release summary lists the refused promote and the jobs.

- [ ] **Step 1: Write the tests:**
  - `test_promote_refused_when_publish_failed`: `vrg-promote` is never
    called, and the error names the jobs.
  - `test_promote_runs_when_every_publish_succeeded`.
  - `test_promote_runs_when_package_jobs_skipped`: a vergil-containers-shaped
    run with `package-build`/`package-sign` `skipped` promotes.
  - `test_promote_never_moves_rolling_tag_backwards`: origin `v2.1` is at
    `v2.1.40`, and promoting `v2.1.39` records "superseded" without pushing.
  - `test_promote_uses_newest_run_attempt`.
  - `test_no_promote_flag_still_skips`.
  - `test_resume_without_fix_still_refused`: an empty in-memory list, with
    the GitHub run still failed, is refused.
  - `test_resume_after_rerun_succeeded_promotes`.
  - `test_develop_docs_failure_does_not_block_promote`.
  - `test_tracking_recovery_text_says_rerun_then_resume`.
- [ ] **Step 2: Run** the tests. Expected: FAIL.
- [ ] **Step 3: Implement it.** Update the docs that describe deferred publish
  failures, in the `vrg-release` docstring and the site docs page for
  `vrg-release`.
- [ ] **Step 4: Run** `vrg-container-run -- vrg-validate`. Expected: PASS.
- [ ] **Step 5: Commit.** `vrg-commit --type fix --scope release --message "refuse to promote the rolling tag while a publish job failed"`

### Task T16: The tracking issue is the release's state; resume works from any fresh checkout [`vergil-tooling`]

**Blocked by:** T15, T14 (`publish_verdict` and the live `deferred_publish`
use T14's `main_publish_failures`).

Added in pushback round 3 (spec §6.2, the human's principle: `vrg-release`
depends on no host-local state). Today `--resume` works only on the host that
started the release. The local `release/<v>` branch, the worktree, in-memory
context and the original CLI flags are the real state. The tracking issue
holds human prose that is never parsed back. An inventory of every local
dependency is in the pushback-round-3 record on `vergil-project/.github#456`.

**Files:**

- Create: `src/vergil_tooling/lib/release/state.py`
- Modify: `lib/release/tracking.py` (create the issue before the deps gate;
  write the state block), `lib/release/resume.py` (parse the state block;
  direct lookup, no `--search`), `lib/release/preflight.py`
  (`_acquire_release_worktree`, the deps gate, one release in flight per
  repo), `lib/release/prepare.py` (`_adopt_merged_pr`, which rebuilds from
  `pull/N/head`; cutting at the gated SHA is T9's),
  `lib/release/bump.py` (adopt `origin/release/post-<v>`),
  `lib/release/confirm.py` (CD runs by `head_sha`), `lib/release/finalize.py`
  and `orchestrator.py` (close the issue after the last stage),
  `lib/release/package_index.py`, `handoff.py` and `tracking.py` (read config
  at the tag), `bin/vrg_release.py` (persisted flags; `--resume` with an
  override)
- Test: `tests/vergil_tooling/test_release_state.py` (new),
  `test_release_resume.py`, `test_release_preflight.py`,
  `test_release_prepare.py`, `test_release_bump.py`,
  `test_release_confirm.py`, `test_release_orchestrator.py`

**Interfaces:**

- Produces, in `state.py`:

  ```python
  STATE_MARKER = "vrg-release:state"   # one hidden block: <!-- vrg-release:state {json} --> in the issue body

  @dataclass
  class TrackedReleaseState:                 # named apart from orchestrator.ReleaseState (pipeline state)
      version: str
      override: Literal["minor", "major"] | None
      flags: dict[str, bool]                 # promote, without_deps, skip_audit
      gated_develop_sha: str | None
      prepare_sha: str | None; bump_sha: str | None
      release_pr: int | None; bump_pr: int | None
      cd_run_ids: dict[str, int]             # "main", "develop"
      deps_gate: dict[str, list[str]]        # bypassed, cleared, undeleted
      publish_verdict: list[str] | None      # T14's main_publish_failures at promote time
      update_deps: dict[str, str] | None     # post-release outcome: pr, fix_task, failure (alignment round 3)

  def read(issue_body: str) -> TrackedReleaseState          # missing or unparseable block -> ReleaseError (loud)
  def write(issue_body: str, state: TrackedReleaseState) -> str
  def update(repo: str, issue: int, **changes) -> TrackedReleaseState   # read-modify-write through run_gh
  ```

- **Rules** (spec §6.2):
  - **Order: lock, gate, create, record** (alignment round 3):
    1. the any-open-issue check (excluding nothing; it runs before this
       release has an issue);
    2. the deps gate;
    3. create the tracking issue;
    4. record the gate's outcome in the state block straight away.

    A refused gate leaves **no** issue behind, so `--resume` can never skip
    the gate.
  - **Write as you go.** Each stage writes its outputs to the state block as
    soon as they exist.
  - **`--resume` reads the block, and never recomputes and discards.** The
    original override and flags apply on resume. `--resume` given a
    *different* override or flag is refused, with both values named.
  - **The release branch is rebuilt from GitHub.** After `merge-release`,
    the local `release/<v>` is created from `pull/<release_pr>/head`, never
    from develop. Before it, from `origin/release/<v>`, or at
    `gated_develop_sha` (T9).
  - **Adopted worktrees are verified.** An adopted worktree's HEAD branch
    must match the stage, otherwise `ReleaseError`.
  - **The bump branch is adopted from origin.** `release/post-<v>` is taken
    from origin when it exists without a PR.
  - **CD runs are found by `head_sha`** through `gh api
    repos/{repo}/actions/runs?head_sha=…`, with no "newest 5" window.
  - **Config is read at the tag.** Stages after `confirm-main` read
    `vergil.toml` and `pyproject.toml` with `git show v<ver>:<path>`.
  - **The issue closes after the release's own last stage,** after
    `package-index` and `consumer-refresh`. A failure there leaves it open and
    resumable.
  - **`update-deps` is post-release** (alignment round 3). It runs after the
    issue closes. Its outcome (PR, fix task, failure) goes into the closed
    issue's state block, and a failure is tracked by its fix task, never by
    keeping the release lock.
  - **One release in flight per repo** (spec §6.3): a fresh-start preflight
    refuses while **any** open `release: *` tracking issue exists. The
    message names it and both remedies: `vrg-release --resume`, or close it if
    it is stale.
  - **`resume.py` uses the direct tracking-issue lookup** (`gh issue list
    --state open`, exact title), never `--search`. This task owns that change.
  - `--state-json` (T9) prints `TrackedReleaseState`. Its `deferred_publish`
    field is T14's **live** `main_publish_failures`, not the sticky comment.

- [ ] **Step 1: Write the tests:**
  - `test_state_round_trips_through_issue_body`
  - `test_unparseable_state_block_fails_loud`
  - `test_refused_deps_gate_leaves_no_issue`
  - `test_gate_bypass_recorded_in_state_after_creation`
  - `test_resume_keeps_override_and_flags`
  - `test_resume_refuses_conflicting_flag`
  - `test_resume_on_fresh_clone_after_merge_release`: there is no local
    branch or worktree, delete-on-merge has run and develop has moved. The
    branch is rebuilt from `pull/N/head` and the release continues.
  - `test_resume_on_fresh_clone_after_back_merge_bump`
  - `test_adopted_worktree_on_wrong_branch_refused`
  - `test_cd_run_found_by_head_sha_beyond_five_runs`
  - `test_post_tag_stages_read_config_at_tag`
  - `test_issue_stays_open_when_package_index_fails`
  - `test_update_deps_failure_does_not_hold_lock`: after a deps failure, the
    next fresh release proceeds, and the fix task is recorded in the closed
    issue's state
  - `test_fresh_start_refused_while_other_release_issue_open`: the message
    names the issue and both remedies
  - `test_resume_lookup_does_not_use_search`
- [ ] **Step 2: Run** the tests. Expected: FAIL.
- [ ] **Step 3: Implement it.** The checklist stays for humans, and the state
  block is the machine record.
- [ ] **Step 4: Run** `vrg-container-run -- vrg-validate`. Expected: PASS at
  100% coverage.
- [ ] **Step 5: Commit.** `vrg-commit --type feat --scope release --message "keep release state in the tracking issue so any checkout can resume it"`

### Task T17: Human-only release prune [`vergil-tooling`]

**Blocked by:** T12.

Added in pushback round 3 (spec §4.4, principle 10; the human chose to
include it in this epic).

**Files:**

- Create: `src/vergil_tooling/bin/vrg_release_prune.py`, registered as
  `vrg-release-prune`
- Create: `src/vergil_tooling/lib/package/index/prune.py`
- Test: `tests/vergil_tooling/test_release_prune.py` (new)

**Interfaces:**

- Consumes T12's `list_remote(limit=None)` (which pages through every
  release, this command only) and `close_over`, and T4's `select`,
  `in_use_names` and `IndexConfig.pins`.
- Produces:
  - `vrg-release-prune --product OWNER/REPO [--apply]`.
  - It computes the **protected set**:
    - every release holding a kept artifact, a pinned version, a grace copy
      or a closure dependency (via `close_over`, which fetches what the
      closure needs);
    - the newest release of the product;
    - every open tracking issue's release.
  - It lists every **other** stable release, with its tag, date and asset
    bytes.
  - **Dry run is the default:** it prints the list and the total bytes, and
    changes nothing.
  - **`--apply`** prompts `type the product name to delete N releases`, then
    deletes each listed **GitHub release** with `gh release delete <tag>
    --yes` through `run_gh`, **without** `--cleanup-tag`. Git tags are never
    deleted.
  - **Human-only:** it refuses unless `is_human()`, naming the mode.

- [ ] **Step 1: Write the tests:**
  - `test_dry_run_deletes_nothing`
  - `test_protected_set_includes_kept_grace_and_closure`
  - `test_newest_release_always_protected`
  - `test_apply_never_deletes_tags` (argv has no `--cleanup-tag`)
  - `test_apply_requires_typed_confirmation`
  - `test_refuses_non_human_modes`
- [ ] **Step 2: Run** the tests. Expected: FAIL.
- [ ] **Step 3: Implement it.** Add a page to the site docs reference.
- [ ] **Step 4: Run** `vrg-container-run -- vrg-validate`. Expected: PASS.
- [ ] **Step 5: Commit.** `vrg-commit --type feat --scope package-index --message "add a human-only prune for releases outside the in-use set"`

It ships with the next routine `vergil-tooling` release. It isn't needed by
any other task.

### Task D1: Deployment: release `vergil-tooling` with automation opt-in and multi-package [`vergil-tooling`, `--kind deployment`]

**Blocked by:** T1, T2, T3, T4, T5, T6, T12, T14, T15, T16.

**Precondition (human-attested):** a `vergil-tooling` release that contains
T1–T6, T12 and T14–T16 has been cut and promoted to `v2.1`.

**Procedure:**

1. Run `uv tool install` from the new tag.
2. `vrg-whoami --explain` on the Mac reports HUMAN.
3. Run `vrg-package matrix` inside a scratch checkout of `vergil-python` at
   `develop`. The cell ids are unchanged.
4. Run `vrg-update-deps --only vergil` by hand in a scratch checkout of
   `vergil-tooling`. It completes, or reports nothing to do, with no
   traceback.
5. **Audit open tracking issues** (spec §6.3, alignment round 3). **Before**
   this release is promoted, list every open `release: *` issue in every org
   that consumes vergil-tooling (`vergil-project`, `logical-minds-foundry`,
   `mnemosys-project`), one `vrg-gh issue list --state open --search
   "release: in:title"` per repo that has releases. Close each stale one with
   a comment, or `--resume` it. Otherwise T16's one-release-in-flight rule
   refuses that repo's next fresh release.

**Acceptance:**

- The `v2.1` tag points at the release.
- Step 3's output is identical to the output before this release.
- Step 4 is observed.
- Step 5's audit is recorded, with each issue listed and how it was handled,
  and no open `release: *` issue remains.

### Task K1: In-use retention in the package index [`packages`]

**Blocked by:** D1.

**Files:**

- Modify: `packages.toml` (`[retention]`)

**Interfaces:**

- Consumes T4's in-use retention and `grace_hours`, and T12's `list_limit`.
- Produces (spec §4.4, principle 10):

  ```toml
  [retention]
  keep = 1          # newest version of each in-use package name
  lines = 1         # each name's newest line only
  grace_hours = 48  # previous version kept briefly for in-flight rollouts (budget-aware)
  list_limit = 20   # first page of releases read per product
  # Explicit in-use list for what a release plan can't see (spec §4.4).
  # TEMPORARY: vergil-python3.14 is served today but absent from vergil-python's
  # newest release until D3 republishes it. Remove this pin in D3.
  pin = ["vergil-python3.14"]
  ```

  The existing `retire = ["vergil-python3.14.8"]` entry is **removed**. It is
  redundant under in-use retention, and would otherwise warn once its
  release leaves the listing (spec §4.4, alignment round 3).

- [ ] **Step 1: Edit `packages.toml`.**
- [ ] **Step 2: Run** `vrg-container-run -- vrg-validate`. Expected: PASS.
- [ ] **Step 3: Commit.** `vrg-commit --type feat --scope retention --message "keep only in-use package versions in the index"`
- [ ] **Step 4: After merge,** check the next `publish-index` run's log:
  - it prints a `projected site:` line;
  - the site keeps the newest `vergil-python3.15`, the pinned
    `vergil-python3.14`, and the newest 2.1 `vergil-tooling`, plus any grace
    copies that fit the budget;
  - no 2.0-line artifact remains.

### Task A1: Package dimension in `ci-package.yml` and `cd-release.yml` [`vergil-actions`]

**Blocked by:** D1.

**Files:**

- Modify: `.github/workflows/ci-package.yml` (build L52-66), and the
  `cd-release.yml` jobs `package-matrix` (L179-199), `package-build`
  (L205-248) and the release-asset staging (L578-581)
- Modify: `.github/workflows/publish-index.yml`, which gains a `site-budget`
  job
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
    `vrg-package check-unpublished --version "$version" --plan-out packages-plan.json`,
    then `vrg-package matrix --github-output --manifest packages-manifest.json --plan packages-plan.json`.
    It uploads `packages-plan.json` together with the manifest.
  - **Artifact names** are unchanged (`package-build-${{ matrix.id }}`,
    `package-unsigned-${{ matrix.id }}`). They stay unique because the cell
    ids carry the package (T3).
  - **`packages-plan.json` is always attached** to the GitHub Release when
    packaging is configured, including when `enabled=false` because every
    package was skipped. `packages-manifest.json` is attached only when
    something was built (T5).
  - **`publish-index.yml` `site-budget` job:**
    - runs `vrg-package index --budget-only` (T12) on every publish-index run;
    - has no `needs:` on the publish job, so the publish never waits on it;
    - takes its exit status from `--budget-only`: red only when total plus
      headroom reaches `SIZE_FAIL`, and a warning line at `SIZE_WARN`. It
      writes the projected size to `$GITHUB_STEP_SUMMARY`.

- [ ] **Step 1: Write the multi-package test** in `package-multi.test.sh`.
  It runs the real `vrg-package` against an offline two-package fixture repo,
  then asserts that:
  - the matrix JSON has prefixed ids and per-row versions;
  - `--plan` with a skipped package removes that package's cells;
  - the workflow YAML (read with `yq`) passes `${{ matrix.version }}` to
    `--version` in both workflows;
  - the release-asset step attaches `packages-plan.json` regardless of
    `enabled`;
  - `publish-index.yml` has a `site-budget` job running `--budget-only`, with
    no dependency from the publish job.
- [ ] **Step 2: Run** `bash actions/cd/release/tests/package-multi.test.sh`. Expected: FAIL.
- [ ] **Step 3: Edit the workflows.**
- [ ] **Step 4: Run** both harnesses, then `vrg-container-run -- vrg-validate`
  (actionlint). Expected: PASS.
- [ ] **Step 5: Commit.** `vrg-commit --type feat --scope package --message "carry the package dimension and per-package version through CI and CD"`

### Task A2: Ops workflows declare `automation` [`vergil-actions`]

**Blocked by:** D1.

**Files:**

- Modify: `.github/workflows/ops-epic-sweep.yml` (the step at L75-85)
- Create: `actions/ops/tests/automation-env.test.sh`, and wire it into a
  `smoke-ops.yml` (new)

**Interfaces:**

- Consumes T1's `VRG_IDENTITY_MODE=automation`, `VRG_JOB_WORKFLOW_REF` and the
  `ops-epic-sweep.yml` grant.
- Produces:
  - The epic-sweep step sets `VRG_IDENTITY_MODE: automation` and
    `VRG_JOB_WORKFLOW_REF: ${{ job.workflow_ref }}`.
  - **Fallback if `job.workflow_ref` is not available** (Step 0 checks): the
    shared workflow sets the value itself, as
    `vergil-project/vergil-actions/.github/workflows/ops-epic-sweep.yml@refs/tags/v2.1`
    plus the major.minor it is released as. Under the accidents-only threat
    model (§6.1) that is equivalent. The test then asserts the literal.
  - `ops-epic-sweep.yml` keeps `VRG_EPIC_SWEEP: "1"` until A4.
  - A step before the sweep runs `vrg-whoami --explain`, so every run's log
    shows the resolved identity and grant. D2 reads it.
  - Every org that calls `ops-epic-sweep.yml@v2.1` picks this up at D2, with
    no change to its own calling workflow.
  - `ops-github-config.yml` is unchanged. Its jobs call no gated command
    (T1's sweep), so the T11 cutover can't break them.

- [ ] **Step 0: Check the context.** Run a scratch reusable workflow on a
  branch that echoes `${{ job.workflow_ref }}`, and record the value in the PR
  body. If it's empty or rejected by actionlint, use the fallback above.
- [ ] **Step 1: Write the test.** `automation-env.test.sh` uses `yq` to assert
  that the epic-sweep step sets `VRG_IDENTITY_MODE=automation` and
  `VRG_JOB_WORKFLOW_REF`, and that a `vrg-whoami --explain` step comes before
  it.
- [ ] **Step 2: Run** the test. Expected: FAIL.
- [ ] **Step 3: Edit the workflows.**
- [ ] **Step 4: Run** the harness and `vrg-container-run -- vrg-validate`. Expected: PASS.
- [ ] **Step 5: Commit.** `vrg-commit --type ci --scope ops --message "declare the automation identity in the ops workflows"`

### Task D2: Deployment: release `vergil-actions` [`vergil-actions`, `--kind deployment`]

**Blocked by:** A1, A2.

**Precondition (human-attested):** a `vergil-actions` release that contains
A1 and A2 has been cut and promoted to `v2.1`.

**Procedure:**

1. Trigger the org epic sweep (`workflow_dispatch` on its caller in
   `vergil-project/.github`).
2. Open the `epic-sweep` job log and check that A2's `vrg-whoami --explain`
   step shows `automation` with the `ops-epic-sweep.yml` grant.
3. Run `vergil-tooling`'s CD once (its next release, or a `workflow_dispatch`
   dry run), and check that the package matrix is unchanged for a
   single-package repo.
4. **Other orgs** (`logical-minds-foundry`, `mnemosys-project`): the human
   attests that each org's next scheduled epic sweep is green, or dispatches
   it.

**Acceptance:**

- Every sweep run is green and declares `automation` with its grant.
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
    no such entry. Every existing guard (L53-106) applies to that entry. It
    also checks **parity both ways**: every `[[series]]` has a `[[package]]`
    and the reverse. So a series can never be silently left unbuilt.
  - **In `update_deps.py`:**
    - `def load_series(runtime_text: str) -> list[dict[str, Any]]`;
    - `def render_runtime(text: str, series: str, cpython: str, tag: str,
      sums: dict[str, str]) -> str`, which rewrites only the named
      `[[series]]` block;
    - `rewrite_package(text, series, ...)`, which rewrites only the
      `[[package]]` block whose `name` is `vergil-python{series}`;
    - `main` refreshes each series independently.
  - **New PBS builds** move every series to the newest stable PBS tag **that
    ships it**, even when the CPython patch hasn't changed (§5, the human's
    decision).
  - **Per-series resolution** (§5, decided in pushback round 2):
    - `def pbs_tags_newest_first() -> list[str]` lists stable
      PBS release tags newer than or equal to the oldest pinned tag, from the
      releases Atom feed (see below).
    - `def resolve_series(series: str, pin: tuple[str, str], tags:
      Sequence[str], sums_for: Callable[[str], dict[str, dict[str, str]]]) ->
      SeriesResolution` walks the tags newest first. It picks the first tag
      whose `SHA256SUMS` ships `<series>.x` for **every** arch.
    - `SeriesResolution(series, cpython, tag, sums, frozen: bool)`. `frozen` is
      True when the newest tag doesn't ship the series at all. The series then
      stays at the best tag found, or at its pin.
    - A frozen series writes a held-back report line `frozen: upstream PBS
      <latest-tag> no longer ships CPython <x.y>; frozen at <cpython>+<tag>`.
      It doesn't stop the other series.
    - **Any series resolved to a tag older than the newest** (for example,
      because the newest tag is missing one arch) gets a held-back line
      `held: CPython <x.y> resolved to PBS <tag>, not the newest <latest>
      (<reason>)`. A held-back patch is never silent.
    - **Tag listing avoids the rate-limited API.** `pbs_tags_newest_first`
      reads the PBS releases Atom feed
      (`https://github.com/astral-sh/python-build-standalone/releases.atom`)
      and the per-tag `SHA256SUMS` download URL. It doesn't use the
      unauthenticated REST API, which allows 60 requests per hour per IP on
      shared runners.
    - **The feed's limit** (verified in pushback round 3): it has the newest
      **10** entries, isn't paginated, and is ordered by update time, not by
      tag. So `pbs_tags_newest_first()` takes no `limit`, sorts by tag
      descending, and logs `truncated: PBS feed oldest <tag>; CPython <x.y>
      pinned at <tag>` whenever a series' pin is older than the feed's oldest
      entry.
    - Tests: `test_feed_sorted_by_tag_not_update_order` and
      `test_pin_older_than_feed_logs_truncation`.
    - A series not found in any listed tag newer than its pin stays at its
      pin and is reported frozen. Never `_die`.
    - `_die` stays for real faults only: an unparseable `SHA256SUMS`, a tag
      older than the pin, or a render failure. Each series is rendered before
      it is written, so a fault in one series writes nothing for it. Other
      series already resolved are still written.
  - **Mixed `pbs_tag`s** across series are normal. `build.sh` already reads the
    tag per series.

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
  - `test_series_resolves_to_newest_tag_that_ships_it`: the latest tag lacks
    3.12, an older tag has it, so 3.12 moves to the older tag while 3.15 moves
    to the latest.
  - `test_series_absent_everywhere_is_frozen_and_others_refresh`: the report
    has a `frozen:` line for that series, its blocks are byte-identical, and
    the other series are rewritten.
  - `test_missing_arch_for_one_series_counts_as_not_shipped`.
  - `test_mixed_pbs_tags_render_and_build`.
  - `test_render_fault_in_one_series_writes_nothing_for_it`.
  - `test_series_on_older_tag_gets_held_line`.
  - `test_tag_listing_uses_atom_feed_not_api`.
  - `test_build.py::test_series_and_package_lists_match_both_ways`: a
    `[[series]]` with no `[[package]]` fails, and so does the reverse.
- [ ] **Step 3: Run** `bash tests/check-build.sh && bash tests/check-update-deps.sh`. Expected: FAIL.
- [ ] **Step 4: Implement it.** Then run the hook locally
  (`VRG_DEPS_REPORT=$(mktemp) scripts/dev/update-deps.sh`) to fill in the
  3.14 series. Check the 3.14 checksums against PBS `SHA256SUMS`.
- [ ] **Step 5: Run** `vrg-container-run -- vrg-validate`. Expected: PASS.
- [ ] **Step 6: Commit.** `vrg-commit --type feat --scope runtime --message "publish every listed series; resume 3.14 patches"`

### Task D3: Deployment: release `vergil-python` with two series [`vergil-python`, `--kind deployment`]

**Blocked by:** P1, K1. K1 has to land first, because a second series under
the old line retention would push the index site toward `SIZE_WARN`.

**Follow-up in this task:** once this release is published, remove K1's
temporary `pin = ["vergil-python3.14"]` from `packages.toml` (a small
`packages` PR). Now that the release carries 3.14, its plan keeps 3.14 in
use.

**Precondition (human-attested):** a `vergil-python` release containing P1 has
been cut. It was released by hand, with its CD green.

**Procedure:**

1. The CD job summary's package plan lists `build` for `vergil-python3.14`,
   and `build` or `skip` for 3.15, depending on whether P1's refresh moved it.
2. On a scratch Ubuntu 24.04 VM: `apt-get install vergil-python3.14
   vergil-python3.15`. Both interpreters run their smoke commands.
3. `apt-cache policy vergil-python3.14` shows the newest 3.14 patch, not
   `3.14.8+20261013`.
4. The release's closing `update-deps` stage ran without a traceback (T2's
   fix), or reported nothing to do.
5. The next publish-index run's `site-budget` job is green, and its projected
   size is recorded.

**Acceptance:**

- Steps 1–5 are observed.
- Record the release URL, the plan output and the projected site size.

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
  class PrCommit:
      oid: str
      authors: tuple[str, ...]          # commits[].authors[].login

  @dataclass(frozen=True)
  class PrProvenance:
      url: str
      author: str                       # author.login
      cross_repository: bool            # isCrossRepository
      head_sha: str                     # headRefOid
      commits: tuple[PrCommit, ...]

  class AutomationMergeRefused(Exception): ...   # names the PR and the failed condition

  def app_slug() -> str                           # VRG_AUTOMATION_APP_SLUG (the per-org App slug from the token step); unset -> AutomationMergeRefused
  def is_app_login(login: str, slug: str) -> bool # True for "app/<slug>", "<slug>[bot]" and "<slug>"; normalised, case-insensitive
  def check(pr: PrProvenance, *, expected_head_sha: str, slug: str) -> None
      # PR author is the App, head is same-repo and equals expected_head_sha, every author of every commit is the App
  def check_release(pr: PrProvenance, *, expected_head_sha: str, gated_develop_sha: str,
                    added_oids: frozenset[str], slug: str) -> None
      # §6.4 release-PR rule: PR author App, same repo, head pinned; only the commits in added_oids
      # (the pipeline's own, on top of gated_develop_sha) must be App-authored
  def release_added_oids(repo_dir: Path, gated_develop_sha: str, head_sha: str) -> frozenset[str]
      # `git rev-list gated_develop_sha..head_sha`; refuses (AutomationMergeRefused) unless
      # gated_develop_sha is an ancestor of head_sha
  def merge_automation_pr(pr: str, *, expected_head_sha: str, strategy: str,
                          release: ReleaseGate | None = None) -> None
      # require(SCOPE_MERGE); check or check_release on github.pr_provenance_view(pr); then
      # pr_merge.wait_and_merge(pr, strategy=strategy, match_head_commit=expected_head_sha)

  @dataclass(frozen=True)
  class ReleaseGate:
      gated_develop_sha: str
      repo_dir: Path
  ```

- **Which rule applies** (§6.4, decided in pushback round 2):
  - lifecycle PRs, `deps/update-*` PRs and the post-release back-merge PR use
    `check`;
  - the release PR (`release/<v>` → `main`) uses `check_release`, because it
    carries human feature commits from `develop` by design.
- **The App bot identity.** `def bot_identity(slug: str, gh: Run =
  local_run) -> tuple[str, str]` returns `(f"{slug}[bot]",
  f"{user_id}+{slug}[bot]@users.noreply.github.com")`. `user_id` comes from
  `gh api users/{slug}[bot] --jq .id`: it is the bot **user** ID, not the App
  ID. A3 uses it to set the git author for every automation commit.
- `github.pr_provenance_view(pr: str) -> PrProvenance` reads `gh pr view
  --json url,author,isCrossRepository,headRefOid`. Commit authors come from
  **REST** `repos/{repo}/pulls/{n}/commits` (`author.login`, `author.type`,
  and `commit.author.email`), because GraphQL's commit-author user can be
  null for bots (spec §6.4, pushback round 3). `is_app_login` also accepts
  the bot noreply email `<id>+<slug>[bot]@users.noreply.github.com` as a
  match. Test: `test_commit_author_matched_by_bot_email_when_login_empty`.
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
  - `test_missing_slug_env_refuses`
  - `test_app_login_forms_normalised`: `app/vergil-project-automation`,
    `vergil-project-automation[bot]` and `Vergil-Project-Automation[bot]` all
    match the slug `vergil-project-automation`, and `vergil-project-automationx`
    does not
  - `test_release_rule_accepts_human_commits_below_gated_sha`
  - `test_release_rule_refuses_foreign_commit_above_gated_sha`
  - `test_release_rule_refuses_when_gated_sha_not_ancestor`
  - `test_back_merge_uses_full_rule`
  - `test_bot_identity_uses_user_id_not_app_id`: a fake `users/<slug>[bot]`
    returning `id: 41898282` gives the email
    `41898282+<slug>[bot]@users.noreply.github.com`
  - `test_merge_passes_match_head_commit`: the recorded gh argv contains
    `--match-head-commit <sha>`
  - `test_merge_requires_merge_scope`
  - `test_engine_rechecks_after_its_own_branch_update`
- [ ] **Step 2: Run** the tests. Expected: FAIL.
- [ ] **Step 3: Implement it.**
- [ ] **Step 4: Run** `vrg-container-run -- vrg-validate`. Expected: PASS.
- [ ] **Step 5: Commit.** `vrg-commit --type feat --scope merge --message "add the unattended merge rules for automation and release PRs"`

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
  - **Pending, frozen and held entries:** a held-back report entry whose
    `note` starts with `PENDING_PREFIX = "pending:"`, `FROZEN_PREFIX =
    "frozen:"` or `HELD_PREFIX = "held:"` (constants in
    `lib/update_deps/report.py`) is copied into `ctx.pending` as well as the
    report, with its prefix kept. The driver reads it (T10). `held:` was added
    in alignment round 3: a series resolved to an older PBS tag than the
    newest. Test: `test_held_entries_collected`.
  - **The `VRG_VERSION_WINDOW` override applies only in dry-run** (§5, decided
    in pushback round 2). With the override set, a run that can push or merge
    exits 1 before any updater runs, with `refusing: VRG_VERSION_WINDOW is set
    (<path>); it is honoured only with --dry-run`. A `--dry-run` uses it and
    prints `window: override <path>`.
  - **`vrg-update-deps --dry-run`** runs preflight and the updaters in the
    worktree. It prints `git diff --stat` and the held-back report, and
    doesn't commit, push or open a PR. T10's dry run uses it.

  Deliberately, the hook **always** gets the window, including when run by
  hand (spec §5, decided in alignment).

- [ ] **Step 1: Write the tests:**
  - `test_hook_receives_released_window_file`
  - `test_window_load_failure_aborts_before_hook`
  - `test_pushed_sha_recorded`
  - `test_automation_merge_uses_rule_with_pushed_sha`
  - `test_human_merge_unchanged`
  - `test_pending_entries_collected`
  - `test_frozen_entries_collected`
  - `test_dry_run_prints_diff_and_report_without_pushing`
  - `test_window_override_honoured_in_dry_run`: the fixture path is what the
    hook receives
  - `test_window_override_refused_when_not_dry_run`: exit 1, no updater runs,
    and stderr names the variable
- [ ] **Step 2: Run** the tests. Expected: FAIL.
- [ ] **Step 3: Implement it.**
- [ ] **Step 4: Run** `vrg-container-run -- vrg-validate`. Expected: PASS.
- [ ] **Step 5: Commit.** `vrg-commit --type feat --scope update-deps --message "pass the released window to the repo hook and merge under the automation rule"`

### Task T13: A refreshing App token source [`vergil-tooling`]

**Blocked by:** T1, T15.

Added in pushback round 2 (§6.2). App installation tokens expire after one
hour, and the chain runs for hours. Nothing unattended may hold a token
captured once. T15 makes `run_gh` the only `gh` exec, so injection happens in
exactly one place.

**Files:**

- Create: `src/vergil_tooling/lib/app_token.py`
- Create: `src/vergil_tooling/bin/vrg_app_credential.py` (a git credential
  helper), registered in `pyproject.toml` `[project.scripts]` as
  `vrg-app-credential`
- Modify: `lib/github.py` (`_gh_env` from T15, `credential_kind`,
  `is_app_mode`, `_generate_jwt`)
- Test: `tests/vergil_tooling/test_app_token.py` (new),
  `test_vrg_app_credential.py` (new), `test_github.py`,
  `test_exec_entry_points.py` (from T15), `test_vrg_github_repo_config.py`

**Interfaces:**

- Produces, in `app_token.py`:

  ```python
  CLIENT_ID_ENV = "VRG_AUTOMATION_APP_CLIENT_ID"
  KEY_FILE_ENV = "VRG_AUTOMATION_APP_KEY_FILE"   # PEM written by the workflow, mode 0600
  REFRESH_MARGIN_SECS = 600                       # re-mint when < 10 min of validity remain

  def enabled() -> bool                           # both env vars set

  @dataclass(frozen=True)
  class TokenPurpose:
      name: str                                   # "chain" | "issues"
      repos: tuple[str, ...]
      permissions: Mapping[str, str]              # sent as the mint request's "permissions" map

  def purposes() -> tuple[TokenPurpose, ...]      # from VRG_AUTOMATION_APP_OWNER / _REPOS (spec §6.2, §8)
  def purpose(name: str) -> TokenPurpose          # "chain" or "issues"; any other name -> AppTokenError
  def current(purpose: TokenPurpose) -> str
      # cached per purpose; re-mints through the GitHub App JWT flow (issuer = Client ID string)
      # when missing or within REFRESH_MARGIN_SECS of expiry; the mint body carries both
      # "repositories" and "permissions"; failure -> AppTokenError (loud, never a stale token)
  ```

  **The permission maps** (spec §8, pushback round 3):
  - `chain` = `{"contents": "write", "pull_requests": "write", "issues":
    "write", "workflows": "write", "actions": "read", "checks": "read",
    "statuses": "read", "administration": "read"}` on the released repos;
  - `issues` = `{"issues": "write"}` on `<owner>/.github` **plus** the
    released repos, because a fix task (in vergil-python) is linked under
    the `.github` ad-hoc epic by a GraphQL `addSubIssue`, which spans both
    (spec §6.2, alignment round 3).

  **Choosing a token is explicit, never inferred.** `run_gh(args, *, purpose:
  str = "chain", ...)`. `lib/epics.py` (sub-issue linking) and the
  issue-create paths of `vrg-issue-create` (fix tasks) pass
  `purpose="issues"`. Nothing infers the token from `--repo` or the working
  directory, which can't work for `gh api graphql`.

- **One credential-detection source** (spec §6.2, pushback round 3).
  `lib/github.py` gains `credential_kind() -> Literal["app-installation",
  "app-user-env", "ambient"]`. It returns `"app-installation"` when
  `app_token.enabled()`.
  - `is_app_mode()` becomes `credential_kind() != "ambient"`, and every
    similar check delegates to it. For example, `vrg-github-repo-config
    audit`'s `app_mode` now sees the injected App token.
  - Guard test: `test_exec_entry_points.py::test_credential_env_read_only_in_github_py`
    forbids reading `GH_TOKEN`, `GITHUB_TOKEN`, `VRG_APP_ID` or `app.env`
    outside `lib/github.py` and `lib/identity_mode.py`.

  The cache is a 0600 file under `$RUNNER_TEMP/vrg-app-token/`, so separate
  processes (the driver, `vrg-release`, `vrg-update-deps`) share one token
  without each re-minting it. Minting holds an exclusive `fcntl.flock` on
  `<cache>.lock`, and re-reads the cache after taking the lock, so concurrent
  processes mint once.
- `owner` and `repos` come from `VRG_AUTOMATION_APP_OWNER` and
  `VRG_AUTOMATION_APP_REPOS` (comma-separated), set by A3.
- **`_gh_env`** (T15) sets `GH_TOKEN=app_token.current(purpose(<the call's
  purpose>))` on every `run_gh` call when `enabled()` is true. It is unchanged
  otherwise. No other code sets `GH_TOKEN`.
- **`vrg-app-credential get`** always serves the `chain` token, because git
  only pushes to released repos. It prints `username=x-access-token` and
  `password=<current(purpose("chain"))>` in git credential-helper format. A3 configures it with
  `git config --global credential.https://github.com.helper '!vrg-app-credential'`,
  and checks out with `persist-credentials: false`.
- `VRG_APP_ID` is never read or set here (Global Constraints).

- [ ] **Step 1: Write the tests:**
  - `test_current_mints_when_cache_missing`
  - `test_current_reuses_fresh_cached_token`
  - `test_current_remints_inside_refresh_margin`: a fake clock 51 minutes
    after minting
  - `test_mint_failure_raises_never_returns_stale`
  - `test_run_gh_injects_fresh_token_per_call`: two calls straddling
    expiry carry different `GH_TOKEN` values
  - `test_run_gh_unchanged_when_disabled`
  - `test_concurrent_mint_happens_once`: two processes racing on an empty
    cache produce one mint call
  - `test_credential_helper_get_format`
  - `test_identity_unaffected_by_app_token_env`: with both env vars set and
    `GITHUB_ACTIONS=true`, `identity_mode` doesn't resolve to USER
  - `test_mint_body_carries_repositories_and_permissions`
  - `test_issues_purpose_token_spans_dotgithub_and_released_repos`
  - `test_sub_issue_link_uses_issues_token`
  - `test_unknown_purpose_refused`
  - `test_jwt_issuer_accepts_client_id_string`
  - `test_credential_kind_reports_app_installation_when_enabled`
  - `test_repo_config_audit_app_mode_true_under_injected_token`
  - `test_credential_env_read_only_in_github_py`
- [ ] **Step 2: Run** the tests. Expected: FAIL.
- [ ] **Step 3: Implement it.** Mint the JWT with the existing App JWT code in
  `lib/github.py` (`_generate_jwt`), changing its issuer from `int(app_id)`
  to a string so it accepts a Client ID. If it doesn't fit, use `PyJWT`,
  which must already be in `uv.lock`. Add no new dependency.
- [ ] **Step 4: Run** `vrg-container-run -- vrg-validate`. Expected: PASS.
- [ ] **Step 5: Commit.** `vrg-commit --type feat --scope automation --message "add a refreshing app installation token source and git credential helper"`

### Task T9: `vrg-release` runs on a runner [`vergil-tooling`]

**Blocked by:** T1, T7, T8, T13, T14, T16.

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

- Consumes T7's `merge_automation_pr` and `ReleaseGate`, T13's token source,
  and T5's `ctx.package_plan`.
- Produces:
  - **Bounded watch, with a fresh token each poll:** `watch_workflow(repo,
    run_id, *, check_status=True, timeout_secs: int)` no longer runs
    `gh run watch`, because one long-lived process holds one token. It polls
    `gh run view <id> --json status,conclusion` every 30 s through the
    token-refreshing runner (T13). At the deadline it raises
    `ReleaseError(phase=...)`. Callers pass `release_timeout_secs()`, which
    reads `VRG_RELEASE_WATCH_TIMEOUT` and defaults to `5400`.
  - **The release-PR gate:** `vrg-release --gated-develop-sha SHA`. It is
    required under AUTOMATION and refused as HUMAN.
    - Preflight asserts only that `SHA` is an ancestor of `origin/develop`,
      and records it in the state block (T16).
    - **`prepare` cuts `release/<v>` at `SHA`**, not at develop HEAD (spec
      §6.4, pushback round 3). A develop merge after the gate ships in the
      next release and never fails this one. `--resume` uses the recorded
      SHA, so it needs no flag.
    - `release/merge.py` merges the release PR with `merge_automation_pr(...,
      release=ReleaseGate(SHA, repo_dir))` (T7 `check_release`).
    - The back-merge PR uses the plain `check`.
  - **Release state for the driver:** `vrg-release --state-json` prints
    `{"in_flight": bool, "version": str | null, "created_by": str | null,
    "resumable": bool, "deferred_publish": [str], "state": <T16
    ReleaseState> | null}` and exits 0, mutating nothing.
    `deferred_publish` is T14's **live** `main_publish_failures`, never the
    sticky deferral comment.
    - `in_flight`: a `release/<v>` branch or an open tracking issue exists.
    - `created_by`: the tracking issue's author login.
    - `resumable`: what `--resume` would accept today.

    T10 uses it to recognise and resume its own interrupted release.
  - **No search in the lock:** `find_existing_tracking_issue` lists open
    issues (`gh issue list --state open --json number,title,url --limit
    1000`) and matches the title exactly.
  - **Release and bump PRs under AUTOMATION:**
    - `release/merge.py` `wait_and_merge` uses `merge_automation_pr` with the
      SHA that `prepare` and `bump` pushed. The release PR goes through the
      release gate above, and the post-release bump/back-merge PR through the
      full rule;
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
  - `app_token.enabled()` (T13). A bare `GH_TOKEN` alone is refused under
    AUTOMATION, because it would expire mid-release.

  Each check fails with an actionable message.

- [ ] **Step 1: Write the tests:**
  - `test_watch_workflow_times_out_loud`
  - `test_watch_workflow_polls_without_long_lived_gh_process`
  - `test_tracking_lookup_does_not_use_search`
  - `test_release_merge_under_automation_pins_pushed_sha`
  - `test_release_merge_under_automation_uses_release_gate`
  - `test_gated_develop_sha_required_under_automation`
  - `test_gated_sha_not_ancestor_of_origin_refused`
  - `test_release_cut_at_gated_sha_despite_later_develop_merge`
  - `test_state_json_reports_resumable_and_deferred_publish`
  - `test_release_merge_human_unchanged`
  - `test_job_summary_written_when_env_set`
  - `test_job_summary_noop_without_env`
  - `test_preflight_automation_rejects_shallow_clone`
  - `test_preflight_automation_rejects_detached_head`
  - `test_preflight_automation_rejects_static_gh_token`
  - `test_human_release_refused_while_automation_release_in_flight`: an
    open `release: <v>` tracking issue made by an AUTOMATION run blocks a
    HUMAN fresh-start preflight for the **next** version, through T16's
    any-open-issue rule (§6.3 "The reverse direction")
  - `test_no_interactive_prompt_under_automation`: the release stage set runs
    under AUTOMATION with stdin closed and never blocks on input (spec §6.2,
    "Prompts")
- [ ] **Step 2: Run** the tests. Expected: FAIL.
- [ ] **Step 3: Implement it.**
- [ ] **Step 4: Run** `vrg-container-run -- vrg-validate`. Expected: PASS.
- [ ] **Step 5: Commit.** `vrg-commit --type feat --scope release --message "let vrg-release run unattended on a runner with bounded waits and pinned merges"`

### Task T10: `vrg-lifecycle`, the driver [`vergil-tooling`]

**Blocked by:** T5, T6, T7, T8, T9, T13, T14, T16.

**Files:**

- Create: `src/vergil_tooling/bin/vrg_lifecycle.py`, and register it in
  `pyproject.toml` `[project.scripts]`
- Create: `src/vergil_tooling/lib/lifecycle/__init__.py`, `orchestrator.py`,
  `trigger.py`, `lock.py`, `issues.py`, `deadline.py`
- Create: `src/vergil_tooling/lib/update_deps/landed.py`, extracted from
  `lib/release/preflight.py:316-366` (the landed-fix clearing). Release
  preflight then calls it, unchanged in behaviour.
- Test: `tests/vergil_tooling/test_lifecycle_orchestrator.py`,
  `test_lifecycle_trigger.py`, `test_lifecycle_lock.py`,
  `test_lifecycle_issues.py`, `test_lifecycle_deadline.py`,
  `test_lifecycle_generic.py`, `test_vrg_lifecycle.py`

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
                    --manifest-gate {cd|checks} --runtime-gate {cd|checks}
                    --deadline-secs N --workdir DIR --pushed PATH [--dry-run]
  ```

  - `--manifest-gate` and `--runtime-gate` name each repo's gate kind
    explicitly (spec §6.4, decided in alignment, round 2). A3 passes `cd` for
    vergil-containers and `checks` for vergil-python. The driver never infers
    the kind from workflow files.
  - `--deadline-secs` is the global deadline (spec §6.2). A3 passes the job's
    `timeout-minutes` × 60 minus 900.
  - `vrg-lifecycle report-abort --run-url URL` is A3's backstop (spec §6.5,
    reworked in pushback round 3).
    - Before each stage, the orchestrator writes `{"repo", "stage"}` to
      `$RUNNER_TEMP/vrg-lifecycle-current.json`.
    - `report-abort` reads that file and opens or refreshes **that stage's
      normal failure issue**, `lifecycle: <repo> <stage> failed`, with detail
      `job died (run <url>)`. It therefore closes only on proof, like every
      other failure.
    - With no file (death before the first stage), it uses the stage
      `preflight`.
    - It needs the `lifecycle` scope.
    - Tests: `test_report_abort_opens_current_stage_failure_issue`, and
      `test_report_abort_issue_survives_run_that_skips_stage`.

  - `--pushed` is the `{branch: sha}` JSON that `publish.py --pushed-out`
    writes (C1).
  - `--dry-run` prints every decision and makes no merge, release or PR. In
    a dry run, `update-runtime` runs `vrg-update-deps --dry-run` (T8), so the
    planned series diff is shown. `release-runtime` reports the `classify`
    plan of the would-be versions.
  - Repos are parameters, so nothing in the code names Python.
- **Stop rule** (spec §6.4, decided in alignment, round 2): only `failed`
  stops the run.
  - A `pending` or `deferred` outcome ends only the stages that depend on it.
  - `release-runtime` still runs when `update-runtime` is `pending` but merged
    a refresh.
  - A `deferred` `release-manifest-repo` doesn't stop stages 4–5.
  - The dependency table lives in `orchestrator.py`:
    `{"wait-develop-cd": "merge-lifecycle", "release-manifest-repo":
    "wait-develop-cd", "release-runtime": "update-runtime"}`.
- **Deadline:** `lifecycle.deadline.Deadline(total_secs)` provides
  `remaining() -> int` and `clip(bound: int) -> int`.
  - Every wait the driver makes, and every `vrg-release` or `vrg-update-deps`
    subprocess (through `timeout=` and `VRG_RELEASE_WATCH_TIMEOUT`), gets
    `clip(bound)`.
  - At zero, the current stage is `failed` with detail `deadline`, and the
    remaining stages aren't run.
  - `issues.sync` runs in a `finally`, so a deadline failure is still
    recorded.
- **Refresh after merges:** after stage 1 and stage 4 merge anything, the
  clone runs `git fetch origin` and `git reset --hard origin/develop`, so later
  stages see develop HEAD.
- **Produces** `orchestrator.run(opts: RunOptions) -> RunResult`, with the
  stages in order:
  0. **`preflight`.** Runs `release.preflight._check_host_prerequisites()`
     (git-cliff), checks that `docker info` succeeds, and checks that
     `app_token.enabled()` (T13) and `VRG_AUTOMATION_APP_SLUG` are set. In
     live mode (no `--dry-run`) it also refuses a set `VRG_VERSION_WINDOW`
     (spec §5). Any miss is `failed` before anything mutates (spec §6.2).
  1. **`merge-lifecycle`.** Merges every open PR on a `lifecycle/` head whose
     family has `merge: "auto"` in the manifest repo's
     `docker/versions/manifest.yml`. It reads the family with the existing
     `parse_branch` convention `lifecycle/<family>-<kind>-<version>`. The SHA
     comes from `--pushed`. A PR with no recorded SHA is a `failed` stage, with
     detail `no recorded SHA for <branch>` (spec §6.4).
  2. **`wait-develop-cd`** (gate kind `cd`). Waits for `cd.yml` on `develop`
     for **develop HEAD after stage 1's refresh**, with every stage 1 merge
     as its ancestor. When stage 1 merged nothing but the trigger holds, it
     waits for develop HEAD. It needs conclusion `success`, clipped by the
     deadline.
     - A run for an older SHA that ended `cancelled` counts as superseded,
       never as failed, once a run on a descendant SHA succeeds. C2 removes
       cancellation, so this is defensive.
     - The SHA that passed is recorded as `gated_develop_sha` for stage 3.
     - With gate kind `checks`, `gated_develop_sha` is develop HEAD, and
       nothing is waited for.
  3. **`release-manifest-repo`.** Runs when
     `trigger.lifecycle_merged_since_last_release(repo_dir) -> bool`: some
     merged PR with a `lifecycle/` head for an auto-merge family has a merge
     commit that is **not an ancestor of the newest stable `vX.Y.Z` tag**
     (`git merge-base --is-ancestor`; spec §6.4, corrected in alignment, round
     2). The `develop-v*` tag isn't used, because main CD creates it from
     develop *after* the release branch is cut
     (`vergil-actions/actions/cd/release/tag-and-release/action.yml:88-90`).
     - **First**, it runs the same `clear_landed_fix` and own-deps-PR resume
       as stage 4, on the **manifest** repo (spec §6.4 step 4, pushback round
       3). A deps PR left by an interrupted vergil-containers release is then
       never stranded, and never blocks the release preflight's deps gate.
     - It then runs `vrg-release --gated-develop-sha <sha>` in the clone.
  4. **`update-runtime`.** First runs `update_deps.landed.clear_landed_fix`
     on the runtime repo (§6.4 step 5, decided in pushback round 2):
     - a `deps/update-*` PR whose fix task is closed as completed is closed,
       and its branch deleted;
     - a `deps/update-*` PR with failed checks and an open fix task makes the
       stage `failed` (detail: the fix task URL), and `vrg-update-deps` is not
       run;
     - a `deps/update-*` PR with pending checks that passes `check` is waited
       on and merged (the resume).

     Then it runs `vrg-update-deps` in the runtime repo clone, and collects
     `pending` (`pending:` and `frozen:` entries).
  5. **`release-runtime`.** Runs when `classify(...)` for the runtime repo has
     at least one `build` or `complete`, and develop is clean.
     - **"Nothing to build" needs proof** (spec §6.4 step 6, pushback round
       3). When nothing is to be built, the stage runs
       `package_index.visible(name, version) -> bool` once, with no wait,
       for every package of the runtime repo's newest release. `visible` is
       the `_wait` predicate, extracted.
       - All visible → `nothing-to-do`.
       - Any missing → `pending`, with item `pending:index:<name>-<version>`
         (bound: `pending`).
     - **The would-be version** passed to `classify` is `vrg-version show` on
       develop without its dev suffix, the same rule `prepare` uses.
     - **"Develop is clean"** means local `develop` equals `origin/develop`,
       with no uncommitted changes.
     - **`gated_develop_sha`** comes from `--runtime-gate`:
       - `checks` (vergil-python): develop HEAD after the stage 4 refresh;
       - `cd`: the same wait as stage 2, on that repo's develop CD.
     - It then runs `vrg-release --gated-develop-sha <sha>`.
- **Lock:** `lock.in_flight(repo: str, slug: str) -> LockState` returns one of
  `clear`, `own-release(resumable | deferred-publish)` or `foreign(reason)`.
  - **A `release/*` branch** counts only if its newest PR is open or it has
    none, using the `vrg-branch-sweep` rules. A merged leftover is ignored.
  - **An open `release: <v>` tracking issue** (T9's lookup) counts as in
    flight. If its author is the App (`is_app_login`), it is `own-release`,
    classified with `vrg-release --state-json` in this precedence:
    1. **Live** `deferred_publish` non-empty (T14's `main_publish_failures`,
       read now; pushback round 3) → the stage is **`failed`**, never
       `deferred`, even when `resumable` is also true. It stays `failed` only
       while that live check fails. Recovery is a human re-running the failed
       publish jobs. The next run then sees an empty live list.
    2. `resumable` → the stage runs `vrg-release --resume` from a fresh
       clone (T16 makes that safe). T14's promote re-verifies from GitHub, so
       a resume can never promote past an unfixed publish.
    3. Neither flag → `failed`, with the state JSON in the detail. That
       covers version skew, a checklist mismatch and the like.
  - **An open PR** on a `deps/update-*` or `lifecycle/*` head that fails
    `automation_merge.check` (spec §6.3, decided in alignment, round 2):
    - **authored by the App** (`is_app_login`) → `failed`, with the failed
      condition in the detail. Tampering is never green.
    - **authored by anyone else** → `foreign`, which means `deferred` for that
      repo.

    One that passes `check` is the driver's own, and is handled by the stages
    above.
- **Results:**

  ```python
  StageOutcome = Literal["done", "nothing-to-do", "deferred", "pending", "failed"]
  @dataclass(frozen=True)
  class StageResult: stage: str; repo: str; outcome: StageOutcome; detail: str
  @dataclass(frozen=True)
  class RunResult: stages: tuple[StageResult, ...]
  def exit_code(r: RunResult) -> int   # 1 if any "failed", else 0
  ```

- **Results carry items:** `StageResult` gains `items: tuple[str, ...]`, the
  per-item keys behind a `pending` or `deferred` outcome. Examples are
  `pending:3.16`, `frozen:3.12` and `deferred:release/2.1.40 in flight`.
- **Issues:** `issues.sync(results: RunResult, *, run_url: str, now: datetime,
  bounds: Bounds, eol: Mapping[str, date]) -> RunResult`. It returns the
  result with overdue items applied.
  - **Failures:** one issue per `(repo, stage)`, titled `lifecycle: <repo>
    <stage> failed`, with label `lifecycle-automation`, filed in that repo.
    `failed` opens or refreshes it. It closes **only** when a later run
    reports that same `(repo, stage)` as `done` or `nothing-to-do` (spec
    §6.5). A stage that wasn't reached leaves it open.
  - **The watch issue** (spec §6.5, decided in alignment, round 2):
    - There is one open issue per repo, titled `lifecycle: <repo> watch`, with
      labels `lifecycle-automation` and `watch`.
    - Its body holds one hidden marker per live item: `<!-- vrg-watch
      item="frozen:3.12" first_seen="2026-10-24" -->`.
    - Items no longer reported are removed from it. When it has no items, it
      is closed.
    - **It is robust to humans** (spec §6.5, pushback round 3):
      - it is looked up by label and title **in any state**, and reopened
        when there are items. A human closing it can't reset `first_seen`;
      - a marker that doesn't parse makes the stage `failed`, with detail
        `unparseable watch marker: <text>`. It is never dropped.
      - Tests: `test_closed_watch_issue_reopened_with_markers_kept`, and
        `test_unparseable_marker_fails_loud`.
  - **Overdue items:**
    - An item past its bound gets its own issue, titled `lifecycle: <repo>
      <item> overdue` with label `overdue`, and the run's result becomes
      `failed`.
    - That item's issue persists while the item lasts, and closes when the
      item disappears. Nothing churns.
    - The bounds (`Bounds`) are:
      - `pending` (`VRG_LIFECYCLE_PENDING_DAYS`, default `14`), from first
        sight;
      - `deferred` (`VRG_LIFECYCLE_DEFERRED_DAYS`, default `3`), from first
        sight;
      - `frozen` (`VRG_LIFECYCLE_FROZEN_DAYS`, default `30`), from the
        series' **EOL date**. `eol` comes from the window's per-version `eol`
        when present, and from the manifest otherwise. Before that date a
        frozen series is green, however long it has been frozen.
      - `held` (`VRG_LIFECYCLE_HELD_DAYS`, default `7`, about one PBS cycle),
        from first sight (spec §6.5, alignment round 3). Tests:
        `test_held_series_recorded_in_watch_issue` and
        `test_held_series_overdue_after_bound_fails`.
  - Every outcome and item is written to `$GITHUB_STEP_SUMMARY`.
- **Dry-run merge-rule verdicts:** in `--dry-run`, `merge-lifecycle` calls
  `pr_provenance_view` and `check` on every open auto-family lifecycle PR,
  then prints `would merge` or `would refuse: <condition>`. D6 uses this to
  confirm App authorship against a real PR before going live (§6.4).

- [ ] **Step 1: Write the trigger and lock tests:**
  - `test_trigger_true_after_lifecycle_merge`
  - `test_trigger_false_with_only_bump_and_deps_merges`
  - `test_trigger_true_for_merge_during_release`. The fixture models the real
    tag placement: the release branch is cut, a lifecycle PR merges into
    develop, then main CD tags `vX.Y.Z` on main's merge **and** `develop-vX.Y.Z`
    at the new develop HEAD. The lifecycle merge is not an ancestor of
    `vX.Y.Z`, so the trigger is true.
  - `test_trigger_false_after_release_including_the_merge`
  - `test_lock_defers_on_foreign_release_branch`
  - `test_lock_ignores_merged_leftover_release_branch`
  - `test_lock_own_resumable_release_resumes`
  - `test_lock_own_deferred_publish_release_fails`
  - `test_lock_deferred_publish_wins_over_resumable`
  - `test_lock_resumes_once_live_publish_check_passes`
  - `test_runtime_nothing_to_do_requires_index_visibility`
  - `test_runtime_index_missing_is_pending_item`
  - `test_manifest_repo_landed_fix_cleared_before_release`
  - `test_lock_own_release_neither_flag_fails`
  - `test_lock_resumes_own_deps_pr`
  - `test_lock_defers_on_human_deps_pr`
  - `test_app_pr_with_foreign_commit_fails`
- [ ] **Step 2: Write the orchestrator tests**, with fake subprocess and gh
  runners:
  - `test_full_chain_order`
  - `test_go_family_pr_never_merged`, for `merge: "human"`
  - `test_pr_without_recorded_sha_fails_with_reason`
  - `test_lifecycle_pr_ci_red_is_failed`
  - `test_pending_series_does_not_block_runtime_release`
  - `test_manifest_repo_deferred_still_updates_runtime`
  - `test_runtime_gate_checks_uses_develop_head`
  - `test_manifest_gate_cd_waits_for_develop_cd`
  - `test_deadline_clips_waits`
  - `test_deadline_hit_fails_stage_and_still_syncs_issues`
  - `test_live_run_refuses_window_override`
  - `test_human_merge_after_stage_one_does_not_fail_release`
  - `test_develop_cd_failure_stops_before_release`
  - `test_rerun_after_containers_release_resumes_at_runtime`
  - `test_runtime_all_skip_no_release`
  - `test_pending_series_is_green_and_recorded`
  - `test_dry_run_makes_no_mutation`
  - `test_requires_lifecycle_scope`
  - `test_preflight_missing_git_cliff_fails_before_any_stage`
  - `test_preflight_missing_docker_fails_before_any_stage`
  - `test_preflight_static_token_without_app_key_fails`
  - `test_develop_cd_cancelled_but_superseded_by_descendant_success_passes`
  - `test_develop_cd_cancelled_without_descendant_success_fails`
  - `test_landed_fix_clears_stale_deps_pr_then_updates`
  - `test_failed_deps_pr_with_open_fix_task_is_failed_not_resumed`
  - `test_release_passes_gated_develop_sha`
  - `test_dry_run_reports_merge_rule_verdicts`
- [ ] **Step 3: Write the issues tests:**
  - `test_failure_issue_opened_then_closed_on_recovery`
  - `test_failure_issue_survives_a_deferred_run`
  - `test_failure_issue_survives_when_stage_not_reached`
  - `test_pending_item_recorded_in_watch_issue_not_own_issue`
  - `test_pending_item_overdue_gets_own_issue_and_fails`
  - `test_overdue_item_issue_persists_without_churn`
  - `test_frozen_item_green_before_eol_regardless_of_age`
  - `test_frozen_item_overdue_after_eol_plus_bound`
  - `test_two_items_tracked_independently`: `pending:3.16` and `frozen:3.12`
    in the same run
  - `test_watch_issue_closed_when_no_items`
  - `test_summary_lists_every_outcome`
- [ ] **Step 3a: Write the generic-code guard** (spec §3 principle 8):
  `test_lifecycle_generic.py::test_no_python_literals_in_shared_machinery`.
  It scans `lib/lifecycle/`, `lib/automation*.py` and `lib/app_token.py`, and
  fails on any `python` string or identifier (case-insensitive), apart from
  an allowlist of import lines.
- [ ] **Step 3b: Write the extraction test.**
  `test_update_deps_landed.py::test_release_preflight_behaviour_unchanged`: the
  existing release-preflight deps-gate tests pass, now calling `landed.py`.
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
    - adds every window series that is missing and has a PBS build, resolved
      with P1's `resolve_series` (the newest tag that ships it on every arch);
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
  - `test_drop_then_rerun_is_noop`.
  - `test_add_proceeds_while_another_series_is_frozen`.
  - `test_unset_window_fails_loud`.
  - `test_window_schema_mismatch_fails_loud`.
- [ ] **Step 2: Run** the tests. Expected: FAIL.
- [ ] **Step 3: Implement it.**
- [ ] **Step 4: Run** `vrg-container-run -- vrg-validate`. Expected: PASS.
- [ ] **Step 5: Commit.** `vrg-commit --type feat --scope runtime --message "reconcile the series list against the released window"`

### Task T11: Fail-closed runner cutover [`vergil-tooling`]

**Blocked by:** D2. The ops workflows declare `automation` before this ships
(spec §9.1, decided in alignment).

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
  - **`VRG_EPIC_SWEEP` is gone** from the code. Removing it from
    `ops-epic-sweep.yml` is task A4. It is harmless if left in place until
    then.
  - **`conftest._hermetic_identity`** also deletes `GITHUB_ACTIONS` and
    `VRG_JOB_WORKFLOW_REF`, so the suite stays hermetic on CI. **So do the
    autouse fixtures in `test_identity_mode.py` and `test_vrg_whoami.py`**,
    which opt out of conftest (`conftest.py:35,42`; pushback round 3).
    Otherwise those tests go red only in CI.
  - `platform_env._correlation_disagrees` labels RUNNER as `runner`, never as
    "human" (spec §6.1). Test: `test_runner_correlation_label`.

- [ ] **Step 1: Write the tests:**
  - `test_undeclared_runner_resolves_runner`
  - `test_human_declared_on_runner_is_runner_with_disagreement`
  - `test_runner_refused_by_release_and_update_deps`
  - `test_runner_vrg_gh_agent_deny_list`
  - `test_epic_sweep_env_no_longer_authorises`
  - T1's `test_every_gate_refuses_new_modes` gains RUNNER in its
    parametrisation
  - `test_container_on_runner_resolves_runner_not_human`: the container
    receives what T1 forwards on an undeclared runner (`GITHUB_ACTIONS=true`
    and `VRG_IDENTITY_MODE=runner`, the host's resolved mode), and
    resolution inside it is RUNNER
- [ ] **Step 2: Run** the tests. Expected: FAIL.
- [ ] **Step 3: Implement it.** Update T1's
  `test_undeclared_runner_still_resolves_human`, which now expects RUNNER.
- [ ] **Step 4: Run** `vrg-container-run -- vrg-validate`. Expected: PASS.
- [ ] **Step 5: Commit.** `vrg-commit --type fix --scope identity --message "fail closed on an undeclared CI runner instead of resolving to human"`

### Task A4: Remove `VRG_EPIC_SWEEP` from `ops-epic-sweep.yml` [`vergil-actions`]

**Blocked by:** D4.

- [ ] **Step 1:** Update `automation-env.test.sh` (A2) to assert that
  `VRG_EPIC_SWEEP` is absent. Run it. Expected: FAIL.
- [ ] **Step 2:** Delete the variable from the workflow. Run the harness and
  `vrg-container-run -- vrg-validate`. Expected: PASS.
- [ ] **Step 3: Commit.** `vrg-commit --type ci --scope ops --message "drop the retired VRG_EPIC_SWEEP switch"`

It ships with the next routine `vergil-actions` release. No deployment task is
needed: the variable is already ignored after D4.

### Task D4: Deployment: release `vergil-tooling` with the cutover, merge rule, unattended release and driver [`vergil-tooling`, `--kind deployment`]

**Blocked by:** T7, T8, T9, T10, T11, T13.

**Precondition (human-attested):** a `vergil-tooling` release containing T7–T11
and T13 has been cut and promoted to `v2.1`.

**Procedure:**

1. Running `vrg-lifecycle run --dry-run ...` on the Mac is refused, because
   the mode is not AUTOMATION.
2. Running `vrg-update-deps` by hand in `vergil-tooling` still opens and
   merges a deps PR exactly as before, or reports that there is nothing to
   update.
3. The next scheduled `ops.yml` runs in `vergil-tooling`, `vergil-containers`
   and `vergil-python` are all green. The org epic sweep reports `automation`
   with the `ops-epic-sweep.yml` grant, and `github-config` (ungated) is green
   under RUNNER.
3b. **Other orgs:** the human attests that the next epic sweep in
   `logical-minds-foundry` and `mnemosys-project` is green after the cutover.
4. On a scratch workflow on a branch, with nothing declared:
   - `vrg-whoami --explain` reports `runner`;
   - `vrg-release` is refused.

   Delete the branch afterwards.

**Acceptance:**

- All observations (1–4, with 3b) are recorded.

### Task C1: Per-family `merge` switch, App-authored commits and pushed SHAs [`vergil-containers`]

**Blocked by:** D4.

**Files:**

- Modify: `docker/versions/manifest.yml` (every family), `docker/versions/manifest.py`
  (`_FAMILY_KEYS`, `Family`, `_parse_family`, `dump_manifest`),
  `docker/versions/lifecycle/publish.py` (`GitCli.publish_branch`, `main`
  argparse, the PR body text)
- Modify: `.github/workflows/ops.yml` `language-lifecycle`: a bot-user-ID
  lookup step, and `LIFECYCLE_GIT_AUTHOR_*` on step 8. This lands in **the
  same PR** that makes them required, so the daily check never goes red in
  between (alignment, round 2).
- Test: `docker/versions/test_manifest.py`, `test_apply.py`, `test_runtime.py`,
  `test_publish.py`, `test_workflows.py` (created by C2, or here if C2 hasn't
  landed yet)

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
  - with `--pushed-out PATH`, writes `{branch: sha}` for **every managed
    lifecycle branch whose head it verified equals the commit it built this
    run**. That covers pushed, refreshed and unchanged branches alike (spec
    §6.4, alignment round 2). An unchanged branch is still the automation's
    own content, so a PR whose earlier merge failed can be retried.
- **The author env in `ops.yml`:**
  - a step before step 8 runs `gh api "users/${SLUG}[bot]" --jq .id`, where
    `SLUG` is the App token step's `app-slug` output;
  - step 8 sets `LIFECYCLE_GIT_AUTHOR_NAME="${SLUG}[bot]"` and
    `LIFECYCLE_GIT_AUTHOR_EMAIL="${ID}+${SLUG}[bot]@users.noreply.github.com"`,
    with the bot **user** ID, never the App ID.
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
  - `test_pushed_out_includes_verified_unchanged_branch`
  - `test_workflows.py::test_language_lifecycle_sets_author_env_from_bot_user_id`
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

- Modify: `.github/workflows/cd-docker-publish.yml` (L28-30)
- Test: `docker/versions/test_workflows.py` (new; reads the YAML with PyYAML)

**Interfaces:**

- Produces (spec §6.3, decided in alignment, round 2):

  ```yaml
  concurrency:
    group: docker-publish-${{ inputs.image-prefix }}
    cancel-in-progress: false
    queue: max
  ```

  - This applies to every caller: `cd.yml` (release and develop),
    `nightly-rebuild-prod` and `nightly-rebuild-dev`. There is no `trigger`
    input.
  - Runs are strictly serialised per image prefix, so an older build can never
    overwrite newer tags. Nothing is ever cancelled or dropped.
  - `queue: max` is valid only with `cancel-in-progress` false or unset
    (GitHub changelog, 2026-05-07). That is exactly this configuration.

- [ ] **Step 0: Check the block,** with actionlint and one scratch dispatch of
  each caller kind (a release-style call and a nightly call). Record both
  results in the PR body.
- [ ] **Step 1: Write the tests:**
  - `test_one_group_per_image_prefix_for_every_caller`
  - `test_no_caller_cancels_in_progress`
  - `test_group_queues_max`
  - `test_no_trigger_input`
- [ ] **Step 2: Run** the tests. Expected: FAIL.
- [ ] **Step 3: Edit the workflows.**
- [ ] **Step 4: Run** `vrg-container-run -- vrg-validate`. Expected: PASS.
- [ ] **Step 5: Commit.** `vrg-commit --type fix --scope ci --message "serialise every image publish per prefix and never cancel one"`

### Task A3: The reusable `ops-lifecycle.yml` [`vergil-actions`]

**Blocked by:** D4.

**Files:**

- Create: `.github/workflows/ops-lifecycle.yml`
- Create: `actions/ops/tests/ops-lifecycle.test.sh`, wired into `smoke-ops.yml`

**Interfaces:**

- Consumes:
  - T10's `vrg-lifecycle run` flags;
  - T1's grant for
    `vergil-project/vergil-actions/.github/workflows/ops-lifecycle.yml@refs/tags/v*`
    (keyed on this workflow's own `job.workflow_ref`, A2's Step 0 fallback
    included);
  - T7's `VRG_AUTOMATION_APP_SLUG` and `bot_identity`;
  - T13's `VRG_AUTOMATION_APP_*` env and `vrg-app-credential`.
- **Produces** a `workflow_call` with:
  - inputs: `manifest-repo` and `runtime-repo` (strings), `manifest-gate` and
    `runtime-gate` (`cd` or `checks`, both required), `dry-run` (boolean,
    default false), and `window-fixture` (string, default `""`). A non-empty
    `window-fixture` without `dry-run: true` fails the job in its first step.
    It is a repo-relative path to a fixture `window.json`, exported as
    `VRG_VERSION_WINDOW` (V2);
  - secrets: `APP_CLIENT_ID` and `APP_PRIVATE_KEY`.
- **Job id `lifecycle`:**
  - `runs-on: ubuntu-latest`, with **no container**, because
    `vrg-update-deps` needs Docker (§6.2);
  - `timeout-minutes: 300`;
  - `concurrency: {group: ops-lifecycle, cancel-in-progress: false, queue:
    max}`, so a scheduled run and a dispatched run never cancel a pending
    one.
- **Steps:**
  0. Check out with `persist-credentials: false` (spec §6.2), so no token is
     written to `.git/config`.
  1. Mint a first App token, with `owner: ${{ github.repository_owner }}` and
     `repositories` listing the manifest repo, the runtime repo **and
     `.github`** (spec §6.4 and §8: fix tasks are filed under
     `<owner>/.github`'s ad-hoc epic). It is used only for setup steps that
     finish within minutes, and to read the `app-slug` output. The same three
     repos go into `VRG_AUTOMATION_APP_REPOS`.
  2. Install uv and vergil-tooling (`./actions/shared/setup/vergil`).
  3. **Install git-cliff** (spec §6.2): the version `vergil-containers` pins
     for its images, downloaded from its GitHub release and checked with
     `sha256sum -c` against the checksum pinned in this workflow. Finish
     with `git-cliff --version`.
  4. Download the `lifecycle-pushed` artifact.
  5. **Set up the refreshing credentials** (T13):
     - write `secrets.APP_PRIVATE_KEY` to `$RUNNER_TEMP/app-key.pem`, mode
       0600;
     - export `VRG_AUTOMATION_APP_CLIENT_ID`, `VRG_AUTOMATION_APP_KEY_FILE`,
       `VRG_AUTOMATION_APP_OWNER` and `VRG_AUTOMATION_APP_REPOS`;
     - set `git config --global credential.https://github.com.helper
       '!vrg-app-credential'`.

     `GH_TOKEN` is **not** exported for the long step. Every clone the driver
     makes uses the credential helper, never a token embedded in a URL or
     `.git/config`.
  6. Set the env:
     - `VRG_IDENTITY_MODE: automation`;
     - `VRG_JOB_WORKFLOW_REF: ${{ job.workflow_ref }}`, or A2's literal
       fallback;
     - `VRG_AUTOMATION_APP_SLUG: ${{ steps.app-token.outputs.app-slug }}`;
     - git author and committer name and email from T7's `bot_identity`
       (`<bot-user-id>+<slug>[bot]@users.noreply.github.com`, where the ID is
       the bot user's, looked up from `users/<slug>[bot]`). This is never the
       App ID.
  7. Run `vrg-lifecycle run --manifest-repo ... --runtime-repo ...
     --manifest-gate ${{ inputs.manifest-gate }} --runtime-gate ${{
     inputs.runtime-gate }} --deadline-secs $(( 300*60 - 900 )) --workdir
     "$RUNNER_TEMP/lifecycle" --pushed lifecycle-pushed.json`, plus
     `--dry-run` when that input is set. A non-empty `window-fixture` is
     exported as an **absolute** path:
     `VRG_VERSION_WINDOW="$GITHUB_WORKSPACE/${{ inputs.window-fixture }}"`.
  8. **The backstop** (spec §6.2): a step with `if: failure() || cancelled()`
     runs `vrg-lifecycle report-abort --run-url "$RUN_URL"`. That opens or
     refreshes **the current stage's** normal failure issue (`lifecycle:
     <repo> <stage> failed`, read from T10's current-stage file) when the job
     dies before the driver's own `issues.sync` runs, for example when
     `timeout-minutes` kills it. T10 adds the `report-abort` subcommand.

- [ ] **Step 1: Write the test.** `ops-lifecycle.test.sh` uses `yq` to assert:
  - the job id is `lifecycle`;
  - there is no `container:` key;
  - the concurrency is set with no cancellation and `queue: max`;
  - `timeout-minutes` is set;
  - `VRG_IDENTITY_MODE` is `automation`, and `VRG_JOB_WORKFLOW_REF` is set;
  - the token step lists exactly the two input repos plus `.github`;
  - checkout has `persist-credentials: false`;
  - `--deadline-secs` is less than `timeout-minutes` × 60;
  - a `failure() || cancelled()` step runs `vrg-lifecycle report-abort`;
  - `window-fixture` is exported under `$GITHUB_WORKSPACE`;
  - the `vrg-lifecycle` step has no `GH_TOKEN` in its env, and the credential
    helper step comes before it;
  - the git author email is built from the bot user-ID lookup, not
    `app-id`;
  - a git-cliff install step with a `sha256sum -c` check comes before the
    `vrg-lifecycle` step;
  - the `window-fixture` guard step refuses a fixture without `dry-run`.
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
  gains `--pushed-out`; and add a new `lifecycle` job
- Test: `docker/versions/test_workflows.py` (from C2)

**Interfaces:**

- Consumes C1's `--pushed-out`, and A3's `ops-lifecycle.yml@v2.1`.
- **Produces:**
  - In `language-lifecycle`, step 8 passes `--pushed-out
    "${OUT}/lifecycle-pushed.json"`. The author env is already set there by
    C1.
  - It uploads that file as the artifact `lifecycle-pushed`, writing `{}` when
    no branch was pushed. A3 downloads it and passes `--pushed
    lifecycle-pushed.json`, so the file name is the same at both ends. Test:
    `test_pushed_file_name_matches_ops_lifecycle`.
  - The new job `lifecycle` has `needs: [language-lifecycle,
    nightly-rebuild-dev, nightly-rebuild-prod]` with `if: always() &&
    needs.language-lifecycle.result == 'success'`, and `uses:
    vergil-project/vergil-actions/.github/workflows/ops-lifecycle.yml@v2.1`.
    Running after the nightlies means the chain's CD waits never queue behind
    a no-cache rebuild (spec §6.3, pushback round 3). Test:
    `test_lifecycle_job_runs_after_nightlies`.
  - It runs only on the default branch, with `manifest-repo:
    vergil-project/vergil-containers` (`manifest-gate: cd`) and `runtime-repo:
    vergil-project/vergil-python` (`runtime-gate: checks`).
  - **The live switch:** `dry-run: ${{ vars.LIFECYCLE_LIVE != 'true' }}`.
    Until a human sets the repo variable `LIFECYCLE_LIVE=true` (D6), every
    run is a dry run.

- [ ] **Step 1: Write the tests:**
  - `test_lifecycle_job_needs_language_lifecycle`
  - `test_lifecycle_job_runs_only_on_default_branch`
  - `test_pushed_artifact_always_uploaded`
  - `test_lifecycle_dry_run_unless_live_variable_true`
  - `test_lifecycle_job_passes_gate_kinds`
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
- **The same environment check for `vergil-containers`:** no required
  reviewers on any environment its release uses (§8: "every repo the driver
  releases").
- **The App's minimum permission set** (§8, the real `automation` boundary)
  is reviewed and attested:
  - on `vergil-containers` and `vergil-python`: contents (write), pull
    requests (write), issues (write), **workflows (write)**, actions (read),
    **checks (read)**, **statuses (read)** and administration (read);
  - on `vergil-project/.github`: issues (write) at least.

  The installation is attested as a **superset**. T13 mints each token with
  exactly its purpose's map. Anything beyond the set is listed with its
  reason.

**Procedure:**

1. With `LIFECYCLE_LIVE` unset, run `workflow_dispatch` on `vergil-containers`
   `ops.yml`. The `lifecycle` job runs as a dry run (C3).
2. Its job summary plans `update-runtime` (adding 3.12 and 3.13) and
   `release-runtime` for `vergil-python`. It makes no merge, release or PR.
3. **The App-authorship check against a real PR** (§6.4). Have the lifecycle
   check open or refresh one Python lifecycle PR (for example, re-run
   `language-lifecycle` after C3, or wait for a `runtime` PR). The dry run's
   `merge-lifecycle` line for it reads `would merge`. Record
   `gh pr view <N> --json author,commits` for that PR. If it reads `would
   refuse`, stop: fix the identity mapping (T7, C3) before going live.
3b. **The permissions probe** (spec §8, pushback round 3). The dry run mints
   each purpose token (T13) and checks it:
   - it reads `GET /installation/repositories`, plus the token's granted
     `permissions` from the mint response;
   - it runs `gh pr checks` on that real PR with the chain token;
   - with the issues token, it creates a throwaway issue in vergil-python,
     links it under the `.github` ad-hoc epic (the fix-task path), and then
     closes it as not planned;
   - it asserts every permission in T13's maps is present, and prints each
     one.

   If anything is missing, stop: fix the installation first.
4. The human sets the repo variable `LIFECYCLE_LIVE=true` on
   `vergil-containers`. That's the switch that starts V1.

**Acceptance:**

- The dry-run output shows the planned stages and makes no mutation.
- Step 3's `would merge` verdict and the raw `author`/`commits` output are
  recorded.
- Step 3b's permission list is recorded, with every required permission
  present.
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
6b. **The chain outlives one token.** At least one run's job log shows the
   chain running for more than 60 minutes with no 401, and T13's re-mint
   logged. If no run lasted that long, record it as not yet observed.
7. **Patches land on 3.14 and 3.15.** On the first PBS release after V1
   starts, a later run moves every series. This is observed if a PBS build
   lands during the validation window; otherwise it is recorded as not yet
   observed.

**Acceptance:**

- Steps 1–6 are observed without human action, with run URLs recorded.
- Steps 6b and 7 are observed, or explicitly recorded as pending, with their
  tracking noted.
- **If any step needed a human,** that step's gap is filed as a follow-on
  under `.github#443` with the exact fallback ("prepared, a human acts"),
  per §6.5 and principle 6. V1 is then still closed as SUCCESS only if every
  gap is filed and the remaining steps passed.

### Task V2: Validation: fixture-driven dry run of a GA add and an EOL drop [`vergil-containers`, `--kind validation`]

**Blocked by:** V1.

**Precondition:** V1 is closed. A3's `window-fixture` input is released at
`v2.1`.

**Procedure:**

1. Commit a fixture `window.json` on a scratch branch of `vergil-containers`
   at `docker/versions/fixtures/v2-window.json`. It is the production window,
   plus `3.16` (`runtime_published: false`) and minus `3.12`.
2. Dispatch `ops-lifecycle.yml` (through a scratch caller on that branch)
   with `dry-run: true` and `window-fixture:
   docker/versions/fixtures/v2-window.json`.
3. The job summary shows:
   - `update-runtime`: the would-be diff adds a `3.16` `[[series]]` and
     `[[package]]` **or** reports `pending: no PBS build for CPython 3.16 yet`
     (whichever PBS supports on the day);
   - the diff drops the `3.12` series from both files;
   - `release-runtime`: the plan, with `build` for any added series and no
     entry for `vergil-python3.12`;
   - no merge, release, PR or issue was made.
4. Delete the scratch branch.

(The index side of a drop, where 3.12 leaves the index after its grace
period, is covered by T4's `test_dropped_series_gone_after_grace`. A dry run
can't produce a newest-release plan to exercise it live.)

**Acceptance:**

- Step 3 is observed, with the run URL recorded.
- The first **live** GA (Python 3.16, around October 2027) is recorded in the
  results comment as a follow-on observation for the retrospective.
- **Known gap** (spec §9): this dry run never merges, so the unattended
  merge of an **add** or **drop** lifecycle PR is first proven by that live
  GA or EOL. The results comment says so explicitly.

## Bootstrap order

Each arrow is a `Blocked-by`.

```text
T15 ─┬─▶ T14 ─▶ T16, T9, T10
     ├─▶ T16 ──────────────────▶ T9, T10
     └─▶ T13 (also needs T1) ──▶ T9, T10
T12 ─▶ T17 (ships with any routine release)
T1 ─┬─▶ T7 ─▶ T8 ─▶ T9 ─▶ T10 ─┐            (T10 also needs T5, T6, T13, T14)
    │                          ├─▶ D4 ─┬─▶ A3 ─▶ D5 ─▶ C3 ─┐
    │        D2 ─▶ T11 ────────┘       ├─▶ C1 ─────────────┼─▶ D6 ─▶ V1 ─▶ V2
T2 ─▶ T3 ─┬─▶ T4 ─▶ T12                ├─▶ P2 ─────────────┘   (P2 also needs P1)
          ├─▶ T5 ─▶ T6                 └─▶ A4
          │
T1..T6, T12, T14–T16 ─▶ D1 ──┬─▶ A1 ─┐
                             ├─▶ A2 ─┴─▶ D2 ─▶ P1 ─┐
                             └─▶ K1 ───────────────┴─▶ D3
C2: independent ─▶ D6
```

Why this order is deadlock-free and safe:

1. **Multi-package support ships before anyone needs it.** Single-package
   repos are unchanged at D1 and D2. Nothing in the fleet moves until P1 in
   `vergil-python` opts in.
2. **The security fix is live before anything runs unattended.** The opt-in
   identity ships at D1. The ops workflows declare it at D2. The fail-closed
   cutover (T11) ships with the driver at D4, so the driver's very first run
   happens under the fail-closed rules (spec §9.1).
3. **Nothing runs unattended until D6.** The driver is released at D4, but it
   only runs in production after D6's human attestations (signing
   environment, App scope), a reviewed dry run, and `LIFECYCLE_LIVE=true`.
4. **The first production run is a real lifecycle change.** The backfill
   creates real `runtime_published` lifecycle PRs, so V1 runs every stage end
   to end. V2 then drives an add and a drop through the driver and the hook
   with a fixture window and no production change.

## Task filing map (epic-create step 9)

| Task | Repo | Kind | Blocked by |
|---|---|---|---|
| T1 Automation identity, grants, scopes | vergil-project/vergil-tooling | task | — |
| T2 `[[package]]` configuration | vergil-project/vergil-tooling | task | — |
| T3 Package dimension in matrix and builds | vergil-project/vergil-tooling | task | T2 |
| T4 Name-aware index, per-package retention | vergil-project/vergil-tooling | task | T3 |
| T5 Build/skip classification | vergil-project/vergil-tooling | task | T3 |
| T6 Per-package install-test, runtime check, index wait | vergil-project/vergil-tooling | task | T5 |
| T12 Lazy, bounded collect, closure, budget *(new, round 2; reworked round 3)* | vergil-project/vergil-tooling | task | T4 |
| T15 One gh/git entry point, guards *(new, alignment round 2)* | vergil-project/vergil-tooling | task | — |
| T14 Promote: publish success, monotonic *(new, round 2; reworked round 3)* | vergil-project/vergil-tooling | task | T15 |
| T16 Tracking issue is the release state; resume anywhere *(new, round 3)* | vergil-project/vergil-tooling | task | T14, T15 |
| T17 Human-only release prune *(new, round 3)* | vergil-project/vergil-tooling | task | T12 |
| D1 Release vergil-tooling (automation opt-in, multi-package) | vergil-project/vergil-tooling | deployment | T1–T6, T12, T14–T16 |
| K1 In-use retention in the index *(new, round 2; reworked round 3)* | vergil-project/packages | task | D1 |
| A1 Package dimension in CI and CD | vergil-project/vergil-actions | task | D1 |
| A2 Ops workflows declare automation | vergil-project/vergil-actions | task | D1 |
| D2 Release vergil-actions | vergil-project/vergil-actions | deployment | A1, A2 |
| P1 Multi-series vergil-python | vergil-project/vergil-python | task | D2 |
| D3 Release vergil-python with two series | vergil-project/vergil-python | deployment | P1, K1 |
| T7 Unattended merge rule | vergil-project/vergil-tooling | task | T1 |
| T8 Update-deps window, rule and dry run | vergil-project/vergil-tooling | task | T1, T7 |
| T13 Refreshing App token source *(new, round 2)* | vergil-project/vergil-tooling | task | T1, T15 |
| T9 vrg-release on a runner | vergil-project/vergil-tooling | task | T1, T7, T8, T13, T14, T16 |
| T10 vrg-lifecycle driver | vergil-project/vergil-tooling | task | T5, T6, T7, T8, T9, T13, T14, T16 |
| T11 Fail-closed runner cutover | vergil-project/vergil-tooling | task | D2 |
| D4 Release vergil-tooling (cutover, merge rule, release, driver) | vergil-project/vergil-tooling | deployment | T7–T11, T13 |
| P2 Hook reconciles series against the window | vergil-project/vergil-python | task | P1, D4 |
| C1 Per-family merge switch, App commits | vergil-project/vergil-containers | task | D4 |
| C2 Docker-publish cancel race | vergil-project/vergil-containers | task | — |
| A3 Reusable ops-lifecycle.yml | vergil-project/vergil-actions | task | D4 |
| D5 Release vergil-actions (ops-lifecycle) | vergil-project/vergil-actions | deployment | A3 |
| A4 Remove VRG_EPIC_SWEEP *(new, alignment round 2)* | vergil-project/vergil-actions | task | D4 |
| C3 Call ops-lifecycle from ops.yml, live switch | vergil-project/vergil-containers | task | C1, D5 |
| D6 Release vergil-containers and vergil-python, attestations, go live | vergil-project/vergil-containers | deployment | C1, C2, C3, P2 |
| V1 First unattended chain run (3.12/3.13 backfill) | vergil-project/vergil-containers | validation | D6 |
| V2 Fixture-driven add/drop dry run | vergil-project/vergil-containers | validation | V1 |

**Bookends already filed:**

- `vergil-project/.github#444`: docs, closed by PR #452.
- `vergil-project/.github#456`: docs revision after pushback round 2.
- `vergil-project/vergil-tooling#3523`: documentation review, blocked by every
  row above.
- `vergil-project/.github#445`: follow-on brainstorm. It carries the
  must-file ruleset item.
- `vergil-project/.github#446`: retrospective (terminal).

**Related intake:** `vergil-project/vergil-tooling#3520`, `#3522` and `#3524`,
and `vergil-project/.github#455` (package hosting beyond the Pages cap).

## Decisions recorded in alignment (2026-10-24)

1. **The cutover moves early.** T11 is blocked only by D2 and ships with the
   driver at D4 (spec §9.1).
2. **V2 is added.** It is a fixture-driven dry run of a GA add and an EOL
   drop. To support it, A3 takes a `window-fixture` input (valid only with
   `dry-run`), and T8 adds `vrg-update-deps --dry-run` (spec §9.6).
3. **The tools are checked before the chain acts.** A3 installs a pinned,
   checksum-verified git-cliff, and T10's `preflight` stage checks git-cliff,
   Docker and the env before any stage runs (spec §6.2).
4. **The hook always reconciles,** including when run by hand. The spec §5
   bullet is amended to match. D6's `--without-deps` precondition keeps V1's
   backfill for the driver.
5. **The `LIFECYCLE_LIVE` switch is in the spec now** (§6.4, "The live
   switch").

## Decisions recorded in pushback round 2 (2026-10-24)

From the second `paad:pushback` pass on the merged spec (`.github#456`):

| # | Spec | Decision | Plan tasks |
|---|---|---|---|
| 1 | §6.4 | Release PRs have their own merge rule. Only the pipeline's commits above the gated develop SHA must be App-authored. | T7, T9, T10 |
| 2 | §6.2 | No credential outlives an hour. The token is refreshed per call, and git uses a credential helper. | T13, T9, T10, A3 |
| 3 | §6.1 | Authorisation keys on the shared workflow's own `job.workflow_ref`. | T1, A2, A3, T11, D2, D4 |
| 4 | §6.3, §6.5 | A failure issue closes only on proof. The driver resumes its own interrupted release. | T9, T10 |
| 5 | §6.2 | Promote requires every publish job to succeed, for every release. | T14 |
| 6 | §4.4 | One version per runtime name, plus a size-projection check, as a stopgap. Hosting goes to `.github#455`. | T4, T12, K1, A1 |
| 7 | §4.4 | Lazy index collect. | T12 |
| 8 | §5 | Per-series PBS resolution, mixed tags, and frozen series. | P1, P2, T8, T10 |
| 9 | §6.4 | Clear a landed fix before update-runtime. A failed deps PR is never resumed. | T10 |
| 10 | §6.3 | The docker-publish groups queue, and the develop-CD wait uses the newest SHA. | C2, T10 |
| 11 | §6.1 | Identity is forwarded into containers. | T1, T11 |
| 12 | §6.4 | App authorship: slug, bot-user-ID email, normalised logins, dry-run verdict. | T7, A3, C3, D6 |
| 13 | §6.1, §8 | The wrapper lists aren't the boundary. The minimum App permissions are attested. | T1, D6 |

Smaller corrections:

- Every package-config reader migrates, and an updater crash becomes a fix
  task (T2).
- Index matching uses exact filenames (T4).
- `retire` overrides newest-kept (T4).
- Per-name line semantics (T4).
- An all-skip release attaches no manifest, and the plan file is always
  attached (T5, A1, T12).
- `check-runtime` checks python-builder packages only (T6).
- `VRG_VERSION_WINDOW` is honoured only in dry-run (T8).
- ~~The trigger uses `develop-v*` ancestry (T10).~~ *Superseded in alignment
  round 2: ancestry to the newest stable `vX.Y.Z` tag.*
- The lock ignores leftover release branches (T10).
- `vrg-finalize-pr --cleanup-only` gets a scoped exception (T1).
- Explicit allowed sets for `vrg-pr-fix-body` and `vrg-audit-approve` (T1).
- The `cloud-vm` ↔ automation correlation (T1).

## Decisions recorded in alignment, round 2 (2026-10-24)

| # | Decision | Plan tasks |
|---|---|---|
| 1 | Each repo's release gate kind is explicit (`cd` / `checks`), passed as a driver input | T10, A3, C3 |
| 2 | A global deadline clips every wait. A deadline hit is `failed` and still syncs issues. A workflow backstop files the issue | T10, A3 |
| 3 | Only `failed` stops the run. `pending` and `deferred` end only their dependent stages | T10 |
| 4 | The lifecycle trigger tests ancestry against the newest stable `vX.Y.Z` tag, not `develop-v*` (corrects pushback batch item m) | T10 |
| 5 | Promote re-verifies publish jobs from the main CD run on GitHub. `deferred_publish` wins over `resumable` | T14, T10 |
| 6 | One watch issue per repo, per-item markers, an issue only past the bound, frozen timed from EOL, overdue turns red without churn | T10 |
| 7 | An App PR failing the rule is `failed`. A human PR on the prefixes is `deferred` | T10 |
| 8 | One entry point each for `gh` and git, with guard tests. The PATH shim was rejected by the human | T15, T13, T14 |
| 9 | The token covers `<owner>/.github` too. Environment checks cover every released repo | A3, T13, D6 |
| 10 | One docker-publish group per prefix, queueing everything, cancelling nothing | C2 |
| 11 | Every verified lifecycle branch has a recorded SHA. No SHA is `failed` with a reason | C1, T10 |
| 12 | C1 sets the lifecycle author env itself, so the daily check never goes red | C1, C3 |

Minor items:

- ~~`[retention.names]` is config~~ (*superseded in pushback round 3 by
  in-use retention with `keep=1`, `lines=1` and pins*), and `packages` is
  named in scope (spec).
- The `publish.py` no-merge test stays (spec).
- The budget check includes next-release headroom (T12).
- The live driver refuses `VRG_VERSION_WINDOW` (T10).
- The gate uses develop HEAD after a refresh (T10).
- An own in-flight release with neither flag is `failed`, and `resume.py`
  uses the direct lookup (T10).
- The partial-publication remedy is stated (T5, spec).
- The would-be version and "develop is clean" are defined (T10).
- Held-back lines are reported, and the Atom feed is used for tags (P1).
- Missing tests are added (T1, T2, T10, P1, P2), along with the
  no-Python-literal guard (T10).
- The Global Constraints wording, the diagram and the blocked-by lines are
  fixed.
- A4 is filed, and the A2 echo is added.
- The A3 fixture path is absolute, with `persist-credentials: false`.

## Decisions recorded in pushback round 3 (2026-10-24)

| # | Spec | Decision | Plan tasks |
|---|---|---|---|
| 1 | §6.2 | **The human's principle:** `vrg-release` depends on no host-local state. The tracking issue carries a parsed state block, resume rebuilds from GitHub on any fresh checkout, config is read at the tag, and the issue closes last | **T16** (new), T9, T10 |
| 2 | §6.2 | One credential-detection source (`credential_kind`), with a guard test. Without it, the repo-config audit misjudges the injected token | T13 |
| 3 | §8 | `workflows: write`, `checks: read` and `statuses: read` added with reasons, and the set proven by a dry-run probe | T13, D6 |
| 4 | §6.2, §6.3 | Promote is monotonic. One release in flight per repo | T14, T16 |
| 5 | §4.4 | The budget uses net-growth headroom plus one allowance, against `SIZE_FAIL`. #455's dates corrected | T12 |
| 6 | §6.2, §8 | Per-purpose tokens with explicit `permissions`: the chain token, and a `.github` token limited to issues | T13 |
| 7 | §4.4 | Iterative closure over fetched artifacts, keeping the retired-dependency guard | T12 |
| 8 | §4.3 | A partial package completes only its missing identities (`complete`) | T5 |
| 9 | §4.4 | A 48 h grace copy of the previous version | T4, K1 |
| 10 | §3, §4.4 | **The human's principle:** one user, retain only what is in use. Principle 9 reversed, #3522 folded in, keep=1 and lines=1, a bounded collect, and a human-only prune | T4, T12, K1, **T17** (new), V2 |
| 11 | §6.4 | `nothing-to-do` needs index-visibility proof | T10 |
| 12 | §6.4 | The release is cut at the gated SHA. Develop HEAD needn't equal it | T9, T16 |

Smaller corrections:

- Promote uses the generic deferred-publish rule, so skipped jobs are fine
  (T14).
- The live `deferred_publish` is used in the lock (T10, T9).
- The lifecycle job runs after the nightlies (C3).
- Landed-fix clearing runs on the manifest repo too (T10).
- `report-abort` writes the current stage's failure issue (T10).
- The watch issue is robust to human edits (T10).
- The Atom feed's 10-entry cap is handled (P1).
- The plan file name is `packages-plan.json` (A1).
- Retention keeps name-version artifacts only (T4).
- The host-resolved mode is forwarded into containers (T1).
- Both resolver test modules are made hermetic (T11).
- Commit authors are read through REST, with an email fallback (T7).
- RUNNER gets its own correlation label (T11).
- The Client ID is the JWT issuer (T13).
- `VRG_JOB_WORKFLOW_REF` is set at step level (T1, A2, A3).

## Decisions recorded in alignment, round 3 (2026-10-24)

| # | Decision | Plan tasks |
|---|---|---|
| 1 | Grace copies are budget-aware: kept only under `SIZE_WARN`, otherwise dropped largest-first, loudly. The budget simulation applies the same rule | T4, T12 |
| 2 | `[retention] pin` is an explicit in-use list (exact or bare name), for VM pin-backs and the K1 transition (a temporary `vergil-python3.14` pin, removed in D3) | T4, K1, D3 |
| 3 | Adaptive listing past K up to a hard cap of 200. Org-owned dependencies are defined by vendor prefix and must resolve | T12 |
| 4 | D1 audits the open `release: *` issues fleet-wide. The refusal names the remedies | D1, T16 |
| 5 | `update-deps` is post-release: the issue closes after `consumer-refresh`, and a deps failure is tracked by its fix task | T16 |
| 6 | Order: lock, gate, create issue, record. A refused gate leaves no issue | T16 |
| 7 | An `issues` token purpose spanning `.github` and the released repos, chosen explicitly. The D6 probe links a real sub-issue | T13, D6 |
| 8 | The redundant `retire` entry is removed. An unmatched `retire` warns instead of failing | K1, T12 |
| 9 | `held:` is a tracked watch item with a 7-day bound | T8, T10 |

Minor items:

- A1 goes red only at `SIZE_FAIL`.
- `ops-lifecycle` uses `queue: max`.
- V2's would-be retention step is dropped.
- A no-prompt test is added (T9).
- "Each name's newest line" replaces "2.1" (spec).
- The duplicate `deferred_publish` line is removed (T9).
- Dependency edges are added: T14 → T16, T9 and T10; T6 → T10.
- Each shared piece of work has one owner: the gated-SHA cut is T9's,
  `resume.py`'s move off `--search` is T16's, and `promote` onto `run_git` is
  T14's.
- A3's backstop text is fixed.
- The `lifecycle-pushed.json` name matches at both ends.
- The Global Constraints wording is updated.
- The state type is renamed `TrackedReleaseState`.
- `IndexConfig` owns the new keys.
- `RemoteRelease.manifest_names` is added.
- T6 waits on `complete` entries.
- `close_over` is exported for T17.
- T9's lock test is fixed, and T11's fixture too.
- Superseded log lines are struck through.
