# Rust validation, supply-chain and dependency-update parity — implementation plan

- **Epic:** vergil-project/.github#1165
- **Planning issue:** vergil-project/.github#1087 (review history is in its comments)
- **Status:** draft, in design review

> **For agentic workers:** each task is one issue and one PR in the repo named
> in its heading, worked with `issue-implement`. Operational tasks are worked
> with `issue-deploy`, and fixture runs with the trio `fixture-run` files.
> Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** A Rust repo's lockfile is enforced, its validation tools are pinned
to digests and versions held in source, its coverage gate requires 100% lines
and regions, its CI evidence names its own tools and reports, and
`vrg-update-deps` updates its `Cargo.lock` and reports what is held back.

**Architecture:**

- **vergil-tooling** changes the Rust gate commands and the dev-container
  warmup, writes the Rust quality reports, reads workspace versions, and adds
  a Cargo updater.
- **vergil-actions** derives each lint and typecheck leg's evidence from
  `reports.sh`, and requires the reports for both languages.
- **vergil-containers** pins prebuilt tool digests in source through the
  daily refresh, and moves cargo-deny and cargo-llvm-cov into refreshed
  `docker/common` fragments.
- Releases, then a fixture run on each Rust fixture, prove the whole set.

**Tech Stack:** Python 3 and pytest (vergil-tooling, vergil-containers
`docker/pins/`), bash (`reports.sh` and its `tests/reports.test.sh`), GitHub
Actions YAML, Dockerfile, cargo, cargo-llvm-cov, cargo-deny, cargo-nextest.

**Spec:** [`spec.md`](spec.md) in this directory. Executors read both.

## Global Constraints

- Stable Rust only. Nothing needs a nightly toolchain (spec §8).
- Python's commands, report names, evidence files and tools stay
  byte-for-byte the same, except the one named fix: a passing lint or
  typecheck leg with no report fails (spec §3, §6.5).
- Every change works for a workspace as well as a single crate. Nothing
  assumes one crate per repo or a library/application split between repos
  (spec §8).
- No silent failures: a missing lockfile, a failed tool install, a digest
  mismatch, a failed `cargo update` and a failed sparse-index read are all
  errors (spec §7).
- No coverage exclusions are added, and no threshold is lowered (spec §3,
  §6.2).
- Releases are human actions. Agents never cut one (spec §8).
- Validation runs only through `vrg-container-run -- vrg-validate` in the
  task repo's worktree. Commit with `vrg-commit`, and hand off with
  `vrg-pr-workflow report-ready`.

## Review Focus

1. **A repo with `Cargo.toml` and no `Cargo.lock`.** `vrg-container-run`
   still starts, and `vrg-validate` fails at install, naming `Cargo.lock` and
   `cargo generate-lockfile`. Pinned in Task 2.
2. **An uncovered `?` error path on a line that ran.** The test gate fails
   and prints that region's file, line and column. Pinned in Task 3.
3. **A passing Python lint leg whose `quality-ruff.json` is missing.** The
   leg's staging step fails. A failing leg with no report shows only its
   original failure. Pinned in Task 8.
4. **A crate whose only newer versions are yanked or pre-releases.** It is
   not reported as held back. Pinned in Task 7.
5. **An upstream release with no checksum for one architecture, during the
   refresh.** The refresh fails, naming the tool and the architecture, and
   writes no fragment. Pinned in Task 9.

---

## Delivery table

Each line gives the task, its repo and kind, what it delivers, and what it is
blocked by.

- **1** (.github, task #1166): spec and plan, this PR.
- **2** (vergil-tooling): lockfile enforced; the warmup never rewrites it.
- **3** (vergil-tooling): coverage gate, 100% lines and regions, misses
  located. Blocked by 11a and 11b.
- **4** (vergil-tooling): `--workspace` on every Rust gate command.
  Blocked by 2.
- **5** (vergil-tooling): `vrg-version` reads and bumps workspace versions.
- **6** (vergil-tooling): Rust lint and typecheck reports. Blocked by 4.
- **7** (vergil-tooling): Cargo updater with the held-back report.
- **8** (vergil-actions): per-leg quality evidence from `reports.sh`;
  reports required.
- **9** (vergil-containers): the refresh pins prebuilt digests;
  cargo-nextest uses them.
- **10** (vergil-containers): cargo-deny and cargo-llvm-cov move to
  refreshed fragments. Blocked by 9.
- **11a** (vergil-fixture-rust-lib): region coverage brought to 100%.
- **11b** (vergil-fixture-rust-app): region coverage brought to 100%.
- **12** (.github, deployment): releases attested; Rust images published
  with the pinned tools. Blocked by 2–10, 11a and 11b.
- **Fixture runs** (both Rust fixtures): one `fixture-run` trio per fixture.
  Blocked by 12.
- **Documentation review** (vergil-tooling, bookend #4443). Blocked by 2–10.
- **Retrospective** (.github, bookend #1167). Blocked by everything else.

- **Ordering in vergil-tooling.** Tasks 2, 4 and 6 change the same Rust entry
  in `lib/languages.py`, so they land in that order. Task 5 touches only
  `lib/version.py` and runs alongside them. It is ordered so that nothing
  waits on it.
- **Task 3 waits for the fixtures.** It is blocked by 11a and 11b, so the new
  gate never ships ahead of fixtures that would fail it (spec §6.2).
- **Release order.** vergil-tooling first, then vergil-actions and
  vergil-containers. Task 8 requires reports that Task 6 writes, so a
  vergil-actions release ahead of vergil-tooling's would fail every Rust
  lint and typecheck leg.
- **Filing.** Every task above is filed at epic-create's filing step, in its
  repo, linked under #1165, with its `Blocked-by:` refs. The fixture runs
  are filed with `fixture-run` once Task 12 exists:

  ```text
  fixture-run vergil-project/.github#1165 rust-lib,rust-app \
    --validates "Rust validation parity: the releases named in Task 12" \
    --after vergil-project/.github#<Task 12>
  ```

---

### Task 2: Lockfile enforced; the warmup never rewrites it (vergil-tooling)

Spec §6.1.

**Files:**

- Modify: `src/vergil_tooling/lib/languages.py` (the `"rust"` entry's
  commands, and `EcosystemInfo.build_cmd` and `publish_cmd`)
- Modify: `src/vergil_tooling/bin/vrg_validate.py` (the `cargo metadata`
  call; a new missing-lockfile check; `_RUST_DOCTEST_CMD` removed, so the
  doctest skip imports `RUST_DOCTEST_CMD`)
- Modify: `src/vergil_tooling/lib/container_cache.py`
  (`_WARMUP_REQUIRES["rust"]`, `_warmup_command`)
- Modify: `docs/site/docs/standards/development/rust/overview.md` (the
  lockfile is committed and enforced, for libraries too)
- Test: `tests/vergil_tooling/test_languages.py`,
  `tests/vergil_tooling/test_vrg_validate.py`,
  `tests/vergil_tooling/test_container_cache.py`

**Interfaces:**

- Produces: every Rust command in `languages.py` that reads `Cargo.lock`
  carries `--locked`: `cargo fetch`, `cargo clippy`, `cargo check`, the
  coverage script's `cargo llvm-cov nextest` and `cargo llvm-cov report`,
  `cargo test --doc`, the `cargo deny` audit, `cargo build --release` and
  `cargo publish`. `cargo fmt` does not.
- Produces: `languages.RUST_DOCTEST_CMD: list[str] == ["cargo", "test",
  "--doc", "--locked"]`. The Rust TEST list and
  `_skip_rust_doctests_without_lib` both use it, so adding a flag can never
  make the skip stop matching. Task 4 adds `--workspace`.
- Produces: `vrg_validate.check_rust_lockfile(repo_root: Path) -> None`.
  It raises `ValidationError` with the message `Cargo.lock is missing:
  run 'cargo generate-lockfile' in the container and commit it` when
  `Cargo.toml` exists and `Cargo.lock` does not. It runs before the Rust
  install stage.
- Produces: `_WARMUP_REQUIRES["rust"] == [["Cargo.toml"], ["Cargo.lock"]]`,
  two groups. A group is satisfied when any of its names is present, so a
  lockfile in the same group as `Cargo.toml` would not be required.
- Produces: `_warmup_command("rust", root)` emits `cargo fetch --locked`.
  When it fails, the warmup prints `warmup skipped: Cargo.lock does not
  match Cargo.toml` and the image build continues, like the existing
  missing-file skip (`container_cache.py`). The warmup never writes
  `Cargo.lock`.

- [ ] **Step 1: Write the failing tests.**
  - `test_rust_lockfile_commands_are_locked`: each command listed above
    contains `--locked`, and `cargo fmt` does not.
  - `test_rust_build_and_publish_locked`: the same, for the ecosystem's
    build and publish commands.
  - `test_missing_cargo_lock_fails_with_guidance`: a tmp repo with only
    `Cargo.toml` gives the error message above.
  - `test_rust_warmup_requires_lockfile`: a repo with only `Cargo.toml`
    skips the warmup.
  - `test_rust_warmup_is_locked`: with both files present, the warmup
    command is `cargo fetch --locked`.
  - `test_rust_warmup_stale_lock_skips_loudly`: a failing locked fetch
    gives the skip notice, the build continues, and `Cargo.lock` is
    byte-identical afterwards.
  - `test_cargo_metadata_locked`: the metadata call carries `--locked`.
  - The existing binary-only doctest-skip tests in `test_vrg_validate.py`
    pass unchanged against the locked command.
- [ ] **Step 2: Run** `vrg-container-run -- vrg-validate`. Expect the new
  tests to fail.
- [ ] **Step 3: Implement** the interfaces above, and update the Rust
  standards page.
- [ ] **Step 4: Run** `vrg-container-run -- vrg-validate`. Expect PASS.
- [ ] **Step 5: Commit** `vrg-commit --type feat --scope rust --message
  "enforce Cargo.lock and keep the warmup from rewriting it"`, then
  report-ready.

### Task 3: Coverage gate: 100% lines and regions, misses located (vergil-tooling)

Spec §6.2. Blocked by 11a and 11b.

**Files:**

- Modify: `src/vergil_tooling/lib/languages.py` (`_RUST_TEST_SCRIPT`)
- Create: `src/vergil_tooling/lib/rust_coverage.py` (reads the uncovered
  regions from cargo-llvm-cov's JSON export)
- Modify: `src/vergil_tooling/bin/vrg_validate.py` (after a failed Rust
  TEST coverage command, run the JSON export and print the regions)
- Modify: `docs/site/docs/standards/development/rust/overview.md` (the gate
  is lines and regions; branch coverage is deferred to
  vergil-project/.github#1171)
- Test: `tests/vergil_tooling/test_languages.py`,
  `tests/vergil_tooling/test_rust_coverage.py`,
  `tests/vergil_tooling/test_vrg_validate.py`

**Interfaces:**

- Produces: `rust_coverage.uncovered_regions(export: dict) -> list[Region]`,
  where `Region` is a frozen dataclass with `file: str`, `line: int` and
  `col: int`. It reads `data[].functions[].regions` from `cargo llvm-cov
  report --json`. These are the records the region percentage counts.
  Each region array is `[line_start, col_start, line_end, col_end,
  execution_count, file_id, expanded_file_id, kind]`, and the file is
  `filenames[file_id]`. It keeps regions with count 0 and kind 0 (code),
  deduplicates them by file, line and column, and sorts them.
- Produces: `rust_coverage.format_regions(regions: list[Region]) ->
  list[str]`, one `<file>:<line>:<col>: uncovered region` line each.
- `_RUST_TEST_SCRIPT`'s report step passes `--fail-under-lines 100
  --fail-under-regions 100` and is otherwise unchanged.
- Produces: in `vrg_validate.py`, when the Rust TEST stage's coverage
  command fails, `vrg-validate` runs `cargo llvm-cov report --json
  --output-path <tmp>` in the same environment and prints
  `format_regions(uncovered_regions(...))`. The stage still fails with the
  coverage command's status. The listing runs in vergil-tooling's own
  process, because the Rust image's `python3` cannot import vergil-tooling.
  If the export fails, that failure is reported too, and the stage still
  fails.

- [ ] **Step 1: Write the failing tests.**
  - `test_rust_gate_requires_regions`: the script contains
    `--fail-under-regions 100` and `--fail-under-lines 100`.
  - `test_uncovered_regions_on_a_covered_line`: a JSON fixture in which a
    function's region `[12, 21, 12, 30, 0, 0, 0, 0]` has count 0 while
    line 12's other regions ran gives `[Region("src/lib.rs", 12, 21)]`.
  - `test_uncovered_regions_ignores_non_code_kinds`: a count-0 region with
    a non-zero kind (an expansion or gap) is not listed.
  - `test_uncovered_regions_dedupes_instantiations`: the same region in two
    generic instantiations is listed once.
  - `test_uncovered_regions_none`: a fully covered export gives `[]`.
  - `test_failed_coverage_prints_regions`: in `test_vrg_validate.py`, with a
    fake runner whose coverage command fails, the output contains
    `src/lib.rs:12:21: uncovered region` and the stage fails.
- [ ] **Step 2: Run** `vrg-container-run -- vrg-validate`. Expect the new
  tests to fail.
- [ ] **Step 3: Implement** the interfaces above, and update the standards
  page.
- [ ] **Step 4: Run** `vrg-container-run -- vrg-validate`. Expect PASS.
- [ ] **Step 5: Commit** `vrg-commit --type feat --scope rust --message
  "require 100% region coverage and locate uncovered regions"`, then
  report-ready.

### Task 4: `--workspace` on every Rust gate command (vergil-tooling)

Spec §6.4, first half. Blocked by 2.

**Files:**

- Modify: `src/vergil_tooling/lib/languages.py` (clippy, check, the
  coverage script, `RUST_DOCTEST_CMD`, `_RUST_AUDIT_SCRIPT`)
- Modify: `src/vergil_tooling/bin/vrg_validate.py` (the skip checks every
  workspace member)
- Test: `tests/vergil_tooling/test_languages.py`,
  `tests/vergil_tooling/test_vrg_validate.py`

**Interfaces:**

- Consumes: Task 2's `--locked` commands and `languages.RUST_DOCTEST_CMD`.
- Produces: `RUST_DOCTEST_CMD == ["cargo", "test", "--doc", "--locked",
  "--workspace"]`.
- Produces: clippy, check, `cargo llvm-cov nextest` and `cargo deny … check`
  all carry `--workspace`.

- [ ] **Step 1: Write the failing tests.**
  - `test_rust_gate_commands_cover_workspace`: each command listed above
    contains `--workspace`, including the audit script.
  - `test_doctest_skip_uses_real_command`: the commands are built with
    `language_commands("rust", CheckKind.TEST)`. A tmp workspace whose
    members are all binary-only has the doctest command dropped.
  - `test_doctest_skip_keeps_lib_member`: a workspace with a binary-only
    root and one library member keeps the doctest command.
- [ ] **Step 2: Run** `vrg-container-run -- vrg-validate`. Expect the new
  tests to fail.
- [ ] **Step 3: Implement** the interfaces above.
- [ ] **Step 4: Run** `vrg-container-run -- vrg-validate`. Expect PASS.
- [ ] **Step 5: Commit** `vrg-commit --type feat --scope rust --message
  "validate every workspace member in the Rust gate"`, then report-ready.

### Task 5: `vrg-version` reads and bumps workspace versions (vergil-tooling)

Spec §6.4, second half.

**Files:**

- Modify: `src/vergil_tooling/lib/version.py` (`_read_version`,
  `_write_version`, `bump`, `_cross_check_language_file`, `bump_paths`)
- Test: `tests/vergil_tooling/test_version.py`

**Interfaces:**

- Produces: `version.cargo_version_table(data: dict) -> tuple[str, ...]`,
  which returns the TOML table path holding the version: `("package",)`
  when `[package].version` is a string, and `("workspace", "package")` when
  `[package]` is absent or has `version.workspace = true`. It raises
  `CargoVersionError` (a new exception, not a `ValueError` subclass) naming
  `[package]` and `[workspace.package]` when neither holds a version.
- Produces: `_write_version` for Rust rewrites the `version` key inside the
  table `cargo_version_table` names, never the first text match.
- `bump()` and `bump_paths()` keep their `except (FileNotFoundError,
  ValueError)` handlers, and `CargoVersionError` passes through them.
- `bump()` resolves `cargo_version_table` before it writes any file,
  `VERSION` included. A manifest with no version table therefore fails with
  nothing changed.

- [ ] **Step 1: Write the failing tests.**
  - `test_cargo_version_from_package`
  - `test_cargo_version_from_virtual_workspace`
  - `test_cargo_version_from_inheriting_root`: a root
    `[package] version.workspace = true` reads `[workspace.package]`.
  - `test_bump_writes_workspace_package_not_dependencies`: a root with
    `[workspace.dependencies] foo = { path = "foo", version = "1.2.3" }`
    and `[workspace.package] version = "1.2.3"`. A patch bump changes only
    `[workspace.package]`.
  - `test_bump_with_no_version_table_fails`: `bump` raises
    `CargoVersionError`, and `VERSION` is unchanged.
  - `test_show_with_no_version_table_fails`
- [ ] **Step 2: Run** `vrg-container-run -- vrg-validate`. Expect the new
  tests to fail.
- [ ] **Step 3: Implement** the interfaces above. Edit the table with a
  TOML-preserving write: `tomlkit` if vergil-tooling already depends on it,
  otherwise a line edit scoped to the table's header and the next header.
- [ ] **Step 4: Run** `vrg-container-run -- vrg-validate`. Expect PASS.
- [ ] **Step 5: Commit** `vrg-commit --type feat --scope version --message
  "read and bump Rust workspace versions"`, then report-ready.

### Task 6: Rust lint and typecheck reports (vergil-tooling)

Spec §6.5, vergil-tooling half. Blocked by 4.

**Files:**

- Modify: `src/vergil_tooling/lib/languages.py` (new `CLIPPY_REPORT` and
  `CARGO_CHECK_REPORT` constants; the clippy and check commands;
  `RUST_REPORT_FILES`)
- Modify: `src/vergil_tooling/data/gitignore/rust`
- Test: `tests/vergil_tooling/test_languages.py`,
  `tests/vergil_tooling/test_gitignore.py`

**Interfaces:**

- Produces: `CLIPPY_REPORT = "quality-clippy.json"` and `CARGO_CHECK_REPORT
  = "quality-cargo-check.json"`. These names are the contract with Task 8.
- Produces: the LINT clippy command and the TYPECHECK command each run
  through `sh -c`. They write `--message-format=json` output to their
  report, and print the human-readable diagnostics to the log, rendered
  from that JSON. The command exits with cargo's status, and the report is
  written even when cargo fails.
- Produces: `RUST_REPORT_FILES` gains both names, and the Rust gitignore
  fragment lists them.

- [ ] **Step 1: Write the failing tests.**
  - `test_rust_quality_report_names`: the two constants have the values
    above.
  - `test_rust_report_files_include_quality`
  - `test_rust_lint_writes_clippy_report`: the built command redirects JSON
    to `quality-clippy.json`, and still carries `--all-targets --locked
    --workspace -- -D warnings`.
  - `test_rust_typecheck_writes_report`
  - `test_rust_gitignore_lists_quality_reports`
- [ ] **Step 2: Run** `vrg-container-run -- vrg-validate`. Expect the new
  tests to fail.
- [ ] **Step 3: Implement** the interfaces above. Render the readable output
  from the JSON stream's `message.rendered` fields, so cargo runs only once.
- [ ] **Step 4: Run** `vrg-container-run -- vrg-validate`. Expect PASS.
- [ ] **Step 5: Commit** `vrg-commit --type feat --scope rust --message
  "write Rust lint and typecheck reports"`, then report-ready.

### Task 7: Cargo updater with the held-back report (vergil-tooling)

Spec §6.6.

**Files:**

- Create: `src/vergil_tooling/lib/update_deps/updaters/cargo.py`
- Create: `src/vergil_tooling/lib/update_deps/sparse_index.py`
- Create: `src/vergil_tooling/lib/update_deps/cargo_semver.py`
- Modify: `src/vergil_tooling/lib/update_deps/orchestrator.py`
  (`DEFAULT_REGISTRY`)
- Modify: `docs/site/docs/reference/dev/update-deps.md`
- Test: `tests/vergil_tooling/test_update_deps_cargo.py`,
  `tests/vergil_tooling/test_update_deps_sparse_index.py`,
  `tests/vergil_tooling/test_update_deps_orchestrator.py`

**Interfaces:**

- Produces: `sparse_index.versions(crate: str, fetch: Fetch) -> list[IndexVersion]`.
  `IndexVersion` has `vers: str`, `yanked: bool` and `rust_version: str |
  None`. It reads `https://index.crates.io/<prefix path>/<crate>` using
  Cargo's index path rule (`1`, `2`, `3/<first char>`, or `<ab>/<cd>`),
  applied to the lowercased crate name. A fetch or parse failure raises
  `SparseIndexError`.
- Produces: `cargo_semver.matches(req: str, version: str) -> bool`,
  following Cargo's documented requirement rules: caret (the default, so
  bare `1.2` means `^1.2`), tilde, `*` wildcards, comparison operators,
  comma-joined comparators, `=` exact, and pre-release matching. An
  unparseable requirement raises `UpdateDepsError`. `packaging` is not
  used, because it implements Python's version rules, not Cargo's.
- Produces: `sparse_index.newest_stable(versions) -> IndexVersion | None`,
  the highest non-yanked, non-pre-release version.
- Produces: `CargoUpdater` with `name = "cargo"`.
  - `applies(ctx)` requires `Cargo.toml` and `Cargo.lock`.
  - `apply(ctx)` runs `vrg-container-run -- cargo update`. It parses
    `Cargo.lock`'s `[[package]]` entries before and after into
    name-to-versions maps (a crate can be locked at several versions), and
    sets `packages=changes.diff_versions(before, after)`, the helper the uv,
    vergil and images updaters already use (vergil-tooling #4456). The
    `summary` and `commit_message` are derived from that list, as
    `<crate> <old> -> <new>`.
  - It fills `held_back` from `held_back(ctx, fetch)`.
- Produces: `cargo.held_back(ctx, fetch) -> list[HeldBack]` over every
  workspace member's normal, dev and build dependencies. These come from
  `vrg-container-run -- cargo metadata --format-version 1 --locked`, run
  in the container like `cargo update` (host tooling never needs a Rust
  toolchain), with stdout captured and parsed. A non-zero exit raises
  `UpdateDepsError`. Each entry has `source="cargo"`
  and a `note` that is one of:
  - `"outside the Cargo.toml requirement"`
  - `"needs a newer Rust than rust-version"`
  - both of those, joined with `"; "`
  - `"held by the resolver: another requirement in the dependency graph"`

  Path and git dependencies are skipped.
- A failed `cargo update`, an unreadable manifest or lockfile, or a
  `SparseIndexError` makes `apply` raise. It never returns "no change".
  Every one of these reaches the orchestrator as an `UpdateDepsError`, the
  type `run_updaters_stage` records as a failure (`orchestrator.py`):
  - `SparseIndexError` subclasses `UpdateDepsError`, with `phase="update"`
    and `command` set to the URL fetched;
  - `OSError` and `tomllib.TOMLDecodeError` from reading `Cargo.toml` or
    `Cargo.lock` are wrapped in `UpdateDepsError`, naming the file.

- [ ] **Step 1: Write the failing tests,** with a fake fetch and a fake
  container runner:
  - `test_index_path_rule`: `a`, `ab`, `abc`, `serde` and `Inflector` map
    to `1/a`, `2/ab`, `3/a/abc`, `se/rd/serde` and `in/fl/inflector`.
  - `test_cargo_semver_matches`: `^1.2`, bare `1.2`, `~1.2`, `1.*`,
    `>=1.2, <1.5` and `=1.2.3`, each against versions just inside and just
    outside the requirement, plus a pre-release case.
  - `test_newest_stable_skips_yanked_and_prerelease`
  - `test_index_failure_raises`
  - `test_cargo_applies_needs_lockfile`
  - `test_metadata_runs_in_container`: the fake runner sees
    `vrg-container-run -- cargo metadata --format-version 1 --locked`, and
    no bare `cargo` call. Its failure raises `UpdateDepsError`.
  - `test_cargo_update_reports_moved_crates`: `result.packages` lists the
    moved crate with its old and new versions, and the summary names it.
  - `test_cargo_update_multi_version_crate`: a crate locked at two versions,
    one of them moved, is itemized correctly.
  - `test_cargo_update_failure_raises`
  - `test_orchestrator_records_cargo_failures`: in
    `test_update_deps_orchestrator.py`, an index failure, an unreadable
    `Cargo.lock` and a corrupt `Cargo.toml` each produce a recorded failure,
    and the held-back report still prints.
  - `test_held_back_outside_requirement`: `serde = "1"` locked at
    `1.0.200`, newest `2.0.0`.
  - `test_held_back_rust_version`: the newest version's `rust_version` is
    above the package's `rust-version`.
  - `test_held_back_by_resolver`: a sibling's `=1.0.150` holds it.
  - `test_not_held_back_when_only_yanked_newer`
  - `test_cargo_in_default_registry`
- [ ] **Step 2: Run** `vrg-container-run -- vrg-validate`. Expect the new
  tests to fail.
- [ ] **Step 3: Implement** the interfaces above, and update the
  `update-deps` reference page.
- [ ] **Step 4: Run** `vrg-container-run -- vrg-validate`. Expect PASS.
- [ ] **Step 5: Commit** `vrg-commit --type feat --scope update-deps
  --message "add the Cargo updater and its held-back report"`, then
  report-ready.

### Task 8: Per-leg quality evidence from `reports.sh`; reports required (vergil-actions)

Spec §6.5, vergil-actions half. The PR touches `.github/workflows/**`, so
GitHub refuses the agent push. The human's `vrg-submit-pr` pushes it.

**Files:**

- Modify: `actions/ci/evidence/reports/reports.sh` (`check_inputs`,
  `has_contract`, `report_files`, `report_tools`)
- Modify: `.github/workflows/ci-quality.yml` (the lint and typecheck legs'
  staging, and the "Emit ci-evidence-quality artifact" step)
- Modify: `actions/ci/evidence/reports/action.yml` (the gate input's allowed
  values)
- Test: `actions/ci/evidence/reports/tests/reports.test.sh`

**Interfaces:**

- Consumes: Task 6's `quality-clippy.json` and `quality-cargo-check.json`,
  and the existing `quality-ruff.json` and `quality-mypy.xml`.
- Produces: gate names `lint` and `typecheck`, with these contracts:

  - `python/lint`: file `quality-ruff.json`, tool `ruff`;
  - `python/typecheck`: file `quality-mypy.xml`, tool `mypy`;
  - `rust/lint`: file `quality-clippy.json`, tools `rustfmt` and `clippy`;
  - `rust/typecheck`: file `quality-cargo-check.json`, tool `cargo`.

- Produces: each lint and typecheck leg stages through `reports.sh` stage
  mode with its version suffix. A passing leg with a missing or empty
  report fails.
- Produces: the emit step's tools are `vrg-validate-common`,
  `vrg-validate-custom`, then the lint and typecheck tools for the repo's
  language. The common and custom logs keep their unsuffixed paths.
- Python's staged file names and its tools list stay identical to today's.

- [ ] **Step 1: Write the failing tests** in `reports.test.sh`:
  - `lint_and_typecheck_are_gates`
  - `rust_lint_files_and_tools`
  - `rust_typecheck_files_and_tools`
  - `python_lint_matches_today`: the files and tools equal the strings
    `ci-quality.yml` hardcodes today.
  - `passing_leg_missing_report_fails`, for each of the four entries.
  - `failing_leg_missing_report_not_extra_error`
- [ ] **Step 2: Run** `vrg-container-run -- vrg-validate`. Expect the new
  tests to fail.
- [ ] **Step 3: Implement** the interfaces above in `reports.sh` and
  `ci-quality.yml`.
- [ ] **Step 4: Run** `vrg-container-run -- vrg-validate`. Expect PASS.
- [ ] **Step 5: Commit** `vrg-commit --type feat --scope evidence --message
  "derive lint and typecheck evidence per language and require the
  reports"`, then report-ready.

### Task 9: The refresh pins prebuilt digests; cargo-nextest uses them (vergil-containers)

Spec §6.3.

**Files:**

- Modify: `docker/pins/resolve_latest.py` (new `TOOL_DIGESTS`,
  `resolve_digests` and `rewrite_digests`; `main --write` writes digests
  with versions)
- Modify: `docker/common/cargo-nextest.dockerfile` (adds
  `ARG CARGO_NEXTEST_SHA256_AMD64` and `_ARM64`; verifies against them, not
  against the downloaded `.sha256`)
- Modify: `docker/pins/pins.yml` (the cargo-nextest entry's reason)
- Test: `docker/pins/test_resolve.py`

**Interfaces:**

- Produces: `TOOL_DIGESTS: dict[str, DigestSource]`, where `DigestSource`
  holds the per-architecture checksum URL templates. The entry for
  `cargo-nextest` names its release's per-asset `.sha256` for
  `x86_64-unknown-linux-gnu` and `aarch64-unknown-linux-gnu`.
- Produces: `resolve_digests(tool: str, version: str, fetch) ->
  dict[str, str]`, which returns `{"AMD64": <hex>, "ARM64": <hex>}`. A
  missing or malformed checksum raises `ResolutionError`, naming the tool
  and the architecture.
- Produces: `rewrite_digests(docker_root, tool, digests) -> bool`, which
  rewrites `ARG <TOOL_ARG_STEM>_SHA256_<ARCH>=` lines. It raises
  `ResolutionError` if a line is missing.
- `main --write` resolves the digests for every tool in `TOOL_DIGESTS` and
  writes them in the same run as the version, so a refresh PR shows the
  version and digest changes together. Nothing is written when any lookup
  fails.

- [ ] **Step 1: Write the failing tests:**
  - `test_resolve_digests_nextest`
  - `test_resolve_digests_missing_arch_raises`
  - `test_rewrite_digests_updates_both_arches`
  - `test_rewrite_digests_missing_arg_raises`
  - `test_write_is_all_or_nothing`: a digest failure after a version
    resolves leaves every fragment unchanged.
  - `test_nextest_fragment_verifies_pinned_digest`: the fragment's verify
    line uses `${CARGO_NEXTEST_SHA256_…}`, and downloads no `.sha256`.
- [ ] **Step 2: Run** `vrg-container-run -- vrg-validate`. Expect the new
  tests to fail.
- [ ] **Step 3: Implement** the interfaces above. Seed the fragment's two
  digests from the current pinned version's release.
- [ ] **Step 4: Run** `vrg-container-run -- vrg-validate`, and build the
  rust image locally for both architectures if the repo's checks do. Expect
  PASS.
- [ ] **Step 5: Commit** `vrg-commit --type feat --scope pins --message "pin
  prebuilt tool digests in source and verify cargo-nextest against them"`,
  then report-ready.

### Task 10: cargo-deny and cargo-llvm-cov move to refreshed fragments (vergil-containers)

Spec §6.3. Blocked by 9.

**Files:**

- Create: `docker/common/cargo-deny.dockerfile` (`ARG CARGO_DENY_VERSION`,
  `ARG CARGO_DENY_SHA256_AMD64` and `_ARM64`, and the
  `<arch>-unknown-linux-musl` tarball verified against the pinned digest)
- Create: `docker/common/cargo-llvm-cov.dockerfile` (`ARG
  CARGO_LLVM_COV_VERSION`, then `cargo install --locked
  cargo-llvm-cov@${CARGO_LLVM_COV_VERSION}`, then the registry cache
  removed)
- Modify: `docker/rust/Dockerfile.template` (the bare `cargo install` RUN
  replaced by `@include` of both fragments)
- Modify: `docker/pins/resolve_latest.py` (`TOOL_REPOS`, `TOOL_ARGS`, and
  `TOOL_DIGESTS` for cargo-deny)
- Modify: `docker/pins/pins.yml` (two auto-managed entries; the header
  note records the reversal of the WS2 decision, #418, and why)
- Test: `docker/pins/test_resolve.py`, `docker/pins/test_pins.py`

**Interfaces:**

- Consumes: Task 9's `TOOL_DIGESTS`, `resolve_digests` and
  `rewrite_digests`.
- Produces: `TOOL_REPOS["cargo-deny"] = "EmbarkStudios/cargo-deny"` and
  `TOOL_REPOS["cargo-llvm-cov"] = "taiki-e/cargo-llvm-cov"`. cargo-llvm-cov's
  tags are `vX.Y.Z`. cargo-deny's are bare `X.Y.Z`; confirm this against
  the live release.

- [ ] **Step 1: Write the failing tests:**
  - `test_cargo_deny_registered_with_digests`
  - `test_cargo_llvm_cov_registered`
  - `test_rewrite_args_finds_new_fragments`
  - `test_template_has_no_bare_cargo_install`: `Dockerfile.template`
    contains no `cargo install` line.
  - `test_pins_entries_for_rust_tools`
- [ ] **Step 2: Run** `vrg-container-run -- vrg-validate`. Expect the new
  tests to fail.
- [ ] **Step 3: Implement** the files above. Seed the versions and digests
  from the current upstream releases (cargo-deny 0.20.2 and cargo-llvm-cov
  0.9.1 at the time of writing).
- [ ] **Step 4: Run** `vrg-container-run -- vrg-validate`, plus the image
  build check. Expect PASS, with `cargo deny --version` and `cargo llvm-cov
  --version` in the build log.
- [ ] **Step 5: Commit** `vrg-commit --type feat --scope rust --message
  "pin cargo-deny and cargo-llvm-cov through refreshed fragments"`, then
  report-ready.

### Tasks 11a and 11b: Fixture region coverage brought to 100% (both fixtures)

Task 11a is in vergil-fixture-rust-lib and Task 11b in
vergil-fixture-rust-app.

Spec §6.2, "measure before switching on". One task per fixture repo.

- [ ] **Step 1: Measure.** In the fixture's worktree, run `vrg-container-run
  -- cargo llvm-cov nextest --locked --workspace` followed by `cargo
  llvm-cov report --fail-under-regions 100 --json`. Record the region
  percentage and any uncovered regions in the issue.
- [ ] **Step 2:** If coverage is 100%, comment the measurement and close the
  task as done with no PR. Otherwise, add tests that exercise each uncovered
  region. Never add an exclusion.
- [ ] **Step 3: Run** `vrg-container-run -- vrg-validate`, and repeat
  Step 1's measurement. Expect 100% lines and regions.
- [ ] **Step 4: Commit** `vrg-commit --type test --scope coverage --message
  "cover every region"`, then report-ready.

### Task 12: Releases attested; pinned-tool Rust images (deployment, .github)

Blocked by 2–10, 11a and 11b.

- **Precondition (human-attested):** the releases are published and their
  versions named in a comment:
  - vergil-tooling (Tasks 2–7), first;
  - then vergil-actions (Task 8) and vergil-containers (Tasks 9 and 10).
- **Procedure (agent-safe):**
  1. Confirm each named release's notes list its tasks' PRs.
  2. Confirm the prod Rust images for every window series are built from
     the named vergil-containers release.
  3. In each image, the checks below hold:
     - `cargo deny --version`, `cargo llvm-cov --version` and `cargo nextest
       --version` report the pinned versions;
     - the image's build log shows each prebuilt tarball verified against
       its pinned digest.
- **SUCCESS:** every check passes, recorded with the release versions and
  image digests.

### Fixture runs (both Rust fixtures)

Filed with `fixture-run` (see Filing), blocked by Task 12. Beyond the
standard trio, each run checks:

- The change task rolls one crate back in `Cargo.lock`, and the release's
  `vrg-update-deps` restores it with a Cargo update commit that names the
  crate. This replaces the earlier "dependency rollback: skipped" known gap.
- The evidence bundle's `quality` gate lists `rustfmt`, `clippy` and `cargo`
  beside the common and custom tools, and holds the version-suffixed
  clippy and cargo-check reports. This replaces the earlier evidence-bundle
  known gap.
- The test gate passes at 100% lines and regions, and the install step runs
  `--locked`.
