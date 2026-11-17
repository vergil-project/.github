# Rust validation, supply-chain and dependency-update parity — implementation plan

- **Epic:** vergil-project/.github#1165
- **Planning issue:** vergil-project/.github#1087 (review history is in its comments)
- **Status:** approved

> **For agentic workers:** each task is one issue and one PR in the repo named
> in its heading, worked with `issue-implement`. Operational tasks are worked
> with `issue-deploy`, and fixture runs with the trio `fixture-run` files.
> Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** A Rust repo's lockfile is enforced, its validation tools are pinned
to digests and versions held in source, its coverage gate requires 100% lines
and regions, its CI evidence names its own tools and reports, and
`vrg-update-deps` updates its `Cargo.lock` and reports what is held back,
and `vrg-validate` builds its release profile before release.

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
- **Live check for Rust command changes.** String tests cannot show that
  cargo accepts a command, and vergil-tooling's own validation never runs
  cargo. So every task that changes a Rust gate command (Tasks 2, 3, 4, 6
  and 13) runs the real commands before report-ready:
  - install the task branch's vergil-tooling into the current Rust image,
    for example from the worktree with `uv tool install`, inside
    `vrg-container-run`;
  - in a clone of `vergil-fixture-rust-lib`, run `vrg-validate` with it;
  - every stage the task changed must pass, and the output goes in the PR.
  A command that only a live run can prove (a flag's position, a `sh -c`
  wrapper, a JSON export) is proved this way, not by a string test alone.

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
  located. Blocked by 4, 11a and 11b.
- **4** (vergil-tooling): `--workspace` on every Rust gate command.
  Blocked by 2.
- **5** (vergil-tooling): `vrg-version` reads and bumps workspace versions.
- **6** (vergil-tooling): Rust lint and typecheck reports. Blocked by 3.
- **7** (vergil-tooling): Cargo updater with the held-back report. Blocked
  by 5.
- **8** (vergil-actions): per-leg quality evidence from `reports.sh`;
  reports required.
- **9** (vergil-containers): the refresh pins prebuilt digests;
  cargo-nextest uses them.
- **10** (vergil-containers): cargo-deny and cargo-llvm-cov move to
  refreshed fragments. Blocked by 9.
- **11a** (vergil-fixture-rust-lib): region coverage brought to 100%.
- **11b** (vergil-fixture-rust-app): region coverage brought to 100%.
- **12** (.github, deployment): releases attested; Rust images published
  with the pinned tools. Blocked by 2–10, 11a, 11b and 13.
- **13** (vergil-tooling): Rust release build in `vrg-validate`, through
  #1174's BUILD check. Blocked by 6 and by the #1174 task that adds the
  BUILD check kind.
- **Fixture runs** (both Rust fixtures): one `fixture-run` trio per fixture.
  Blocked by 12.
- **Documentation review** (vergil-tooling, bookend #4443). Blocked by 2–10
  and 13.
- **Retrospective** (.github, bookend #1167). Blocked by everything else.

- **Ordering in vergil-tooling.** Tasks 2, 4, 3, 6 and 13 change the same Rust
  entry in `lib/languages.py` (Task 3 rewrites `_RUST_TEST_SCRIPT`'s report
  line, which Tasks 2 and 4 also change) and all touch `vrg_validate.py`.
  So they land in that order. Task 5 touches `lib/version.py` and creates
  `lib/cargo_semver.py`. Task 7 touches only `lib/update_deps/`, uses
  `cargo_semver`, and follows Task 5. Both run alongside the chain, and the
  chain never waits on them.
- **Task 3 also waits for the fixtures.** It is blocked by 11a and 11b, so
  the new gate never ships ahead of fixtures that would fail it (spec
  §6.2).
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
  Only when it fails because the lockfile is stale does the warmup print
  `warmup skipped: Cargo.lock does not match Cargo.toml` and let the image
  build continue, like the existing missing-file skip
  (`container_cache.py`). That case is recognised by cargo's own error on
  stderr, the "needs to be updated but --locked was passed" message. Any
  other failure (a registry or network outage, or a git dependency that
  cannot be fetched) fails the build loudly with cargo's output, as a
  warmup failure does today. The warmup never writes `Cargo.lock`.
- Produces: the skip is visible even when the warmup's output is captured
  (`quiet_warmup=True`, as `vrg-finalize-pr` provisions). The warmup shell
  writes a fixed marker line with the notice, and the host side
  (`_build_cached_image`) finds it in the captured output and prints the
  notice itself.

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
  - `test_rust_warmup_stale_lock_skips_loudly`: a locked fetch that fails
    with cargo's stale-lockfile message gives the skip notice, the build
    continues, and `Cargo.lock` is byte-identical afterwards.
  - `test_rust_warmup_other_failure_fails_build`: a locked fetch that fails
    with a network error fails the image build and shows cargo's output.
  - `test_rust_warmup_skip_visible_when_quiet`: with `quiet_warmup=True`,
    the stale-lockfile skip still prints its notice.
  - `test_cargo_metadata_locked`: the metadata call carries `--locked`.
  - The existing binary-only doctest-skip tests in `test_vrg_validate.py`
    pass unchanged against the locked command.
- [ ] **Step 2: Run** `vrg-container-run -- vrg-validate`. Expect the new
  tests to fail.
- [ ] **Step 3: Implement** the interfaces above, and update the Rust
  standards page.
- [ ] **Step 4: Run** `vrg-container-run -- vrg-validate`. Expect PASS.
- [ ] **Step 4b: Live check** (Global Constraints) against
  `vergil-fixture-rust-lib`. Expect every changed stage to pass, and record
  the output in the PR.
- [ ] **Step 5: Commit** `vrg-commit --type feat --scope rust --message
  "enforce Cargo.lock and keep the warmup from rewriting it"`, then
  report-ready.

### Task 3: Coverage gate: 100% lines and regions, misses located (vergil-tooling)

Spec §6.2. Blocked by 4, 11a and 11b.

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
  `filenames[file_id]`. It keeps only kind 0 (code) regions, and merges each
  region's count across a function's instantiations by file, line and
  column, taking the maximum. A region that ran in any instantiation
  therefore counts as covered, as in llvm-cov's own summary. It returns the
  regions whose merged count is 0, sorted.
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
  - `test_uncovered_regions_dedupes_instantiations`: the same region at
    count 0 in two generic instantiations is listed once.
  - `test_uncovered_regions_covered_in_one_instantiation`: a region at count
    0 in one instantiation and 3 in another is not listed.
  - `test_uncovered_regions_none`: a fully covered export gives `[]`.
  - `test_failed_coverage_prints_regions`: in `test_vrg_validate.py`, with a
    fake runner whose coverage command fails, the output contains
    `src/lib.rs:12:21: uncovered region` and the stage fails.
- [ ] **Step 2: Run** `vrg-container-run -- vrg-validate`. Expect the new
  tests to fail.
- [ ] **Step 3: Implement** the interfaces above, and update the standards
  page.
- [ ] **Step 4: Run** `vrg-container-run -- vrg-validate`. Expect PASS.
- [ ] **Step 4b: Live check** (Global Constraints) against
  `vergil-fixture-rust-lib`. Expect every changed stage to pass, and record
  the output in the PR.
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
- Produces: clippy, check and `cargo llvm-cov nextest` carry `--workspace`.
  The audit becomes `cargo deny --format json --config <policy> --locked
  --workspace check --show-stats`. cargo-deny's `--format`, `--config`,
  `--locked` and `--workspace` are root options, so they come before
  `check`.

- [ ] **Step 1: Write the failing tests.**
  - `test_rust_gate_commands_cover_workspace`: each command listed above
    contains `--workspace`, including the audit script.
  - `test_cargo_deny_root_flags_before_check`: in the audit script,
    `--locked` and `--workspace` appear before `check`.
  - `test_doctest_skip_uses_real_command`: the commands are built with
    `language_commands("rust", CheckKind.TEST)`. A tmp workspace whose
    members are all binary-only has the doctest command dropped.
  - `test_doctest_skip_keeps_lib_member`: a workspace with a binary-only
    root and one library member keeps the doctest command.
- [ ] **Step 2: Run** `vrg-container-run -- vrg-validate`. Expect the new
  tests to fail.
- [ ] **Step 3: Implement** the interfaces above.
- [ ] **Step 4: Run** `vrg-container-run -- vrg-validate`. Expect PASS.
- [ ] **Step 4b: Live check** (Global Constraints) against
  `vergil-fixture-rust-lib`. Expect every changed stage to pass, and record
  the output in the PR.
- [ ] **Step 5: Commit** `vrg-commit --type feat --scope rust --message
  "validate every workspace member in the Rust gate"`, then report-ready.

### Task 5: `vrg-version` reads and bumps workspace versions (vergil-tooling)

Spec §6.4, second half.

**Files:**

- Create: `src/vergil_tooling/lib/cargo_semver.py`
- Modify: `src/vergil_tooling/lib/version.py` (`_read_version`,
  `_write_version`, `bump`, `_cross_check_language_file`, `bump_paths`)
- Test: `tests/vergil_tooling/test_version.py`,
  `tests/vergil_tooling/test_cargo_semver.py`

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
- Before writing anything, `bump()` also checks every versioned path
  dependency on a workspace member: the `[workspace.dependencies]` entries
  whose `path` is a member, and path dependencies in member crates' own
  manifests that carry a `version` (spec §6.4). If
  the new version does not satisfy an entry's requirement (for example
  `0.1.3` to `0.2.0` under `^0.1.3`), it raises `CargoVersionError` naming
  each such entry and writes nothing. Cargo would reject the bumped
  workspace, and the later `cargo update --workspace` would fail with
  `VERSION` already rewritten. Rewriting those entries belongs to the
  workspace follow-on, vergil-project/.github#1179. The check uses
  `cargo_semver.matches`, which this task creates.
- Produces: `vergil_tooling.lib.cargo_semver.matches(req: str, version: str)
  -> bool`, following Cargo's documented requirement rules: caret (the
  default, so bare `1.2` means `^1.2`), tilde, `*` wildcards, comparison
  operators, comma-joined comparators, `=` exact, and pre-release matching.
  An unparseable requirement raises `ValueError`. `packaging` is not used,
  because it implements Python's version rules, not Cargo's. Task 7 uses it
  too.

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
  - `test_cargo_semver_matches`: `^1.2`, bare `1.2`, `~1.2`, `1.*`,
    `>=1.2, <1.5` and `=1.2.3`, each against versions just inside and just
    outside the requirement, plus a pre-release case and an unparseable
    requirement.
  - `test_bump_breaking_member_requirement_fails_first`: with
    `foo = { path = "foo", version = "0.1.3" }` and a workspace at `0.1.3`,
    a minor bump raises `CargoVersionError` naming `foo`, and `VERSION` and
    `Cargo.toml` are unchanged. A patch bump succeeds.
  - `test_bump_breaking_member_manifest_requirement_fails_first`: the same
    check for a member crate's own `bar = { path = "../foo", version =
    "0.1.3" }`.
- [ ] **Step 2: Run** `vrg-container-run -- vrg-validate`. Expect the new
  tests to fail.
- [ ] **Step 3: Implement** the interfaces above. Edit the table with a
  TOML-preserving write: `tomlkit` if vergil-tooling already depends on it,
  otherwise a line edit scoped to the table's header and the next header.
- [ ] **Step 4: Run** `vrg-container-run -- vrg-validate`. Expect PASS.
- [ ] **Step 5: Commit** `vrg-commit --type feat --scope version --message
  "read and bump Rust workspace versions"`, then report-ready.

### Task 6: Rust lint and typecheck reports (vergil-tooling)

Spec §6.5, vergil-tooling half. Blocked by 3.

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
  - `test_failing_lint_still_writes_report`: a failing clippy still writes
    `quality-clippy.json` and exits with cargo's status. The same for a
    failing `cargo check` and `quality-cargo-check.json`.
  - `test_rust_gitignore_lists_quality_reports`
- [ ] **Step 2: Run** `vrg-container-run -- vrg-validate`. Expect the new
  tests to fail.
- [ ] **Step 3: Implement** the interfaces above. Render the readable output
  from the JSON stream's `message.rendered` fields, so cargo runs only once.
- [ ] **Step 4: Run** `vrg-container-run -- vrg-validate`. Expect PASS.
- [ ] **Step 4b: Live check** (Global Constraints) against
  `vergil-fixture-rust-lib`. Expect every changed stage to pass, and record
  the output in the PR.
- [ ] **Step 5: Commit** `vrg-commit --type feat --scope rust --message
  "write Rust lint and typecheck reports"`, then report-ready.

### Task 7: Cargo updater with the held-back report (vergil-tooling)

Spec §6.6. Blocked by 5.

**Files:**

- Create: `src/vergil_tooling/lib/update_deps/updaters/cargo.py`
- Create: `src/vergil_tooling/lib/update_deps/sparse_index.py`
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
- Consumes: Task 5's `cargo_semver.matches`. A `ValueError` from it is
  wrapped in `UpdateDepsError`, naming the crate and the requirement.
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
  - `changed` comes from `git status --porcelain Cargo.lock`, as in the uv
    updater. When the lockfile changed but no package moved (cargo dropped
    unused entries or rewrote the format), the commit message is
    `chore(deps): refresh Cargo.lock` and `packages` is `[]`.
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

  A workspace-inherited dependency (`foo.workspace = true`) uses the
  requirement `cargo metadata` reports for it. Path and git dependencies
  are skipped, with a comment in the code giving the reason: they have no
  index entry to compare against.
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
  - `test_cargo_lock_rewritten_without_moves`: the lockfile changes with no
    version moved, giving `changed=True`, `packages=[]` and the refresh
    commit message.
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
  - `test_inherited_dependency_uses_metadata_requirement`: `serde.workspace
    = true` is checked against the requirement `cargo metadata` reports.
  - `test_path_and_git_dependencies_skipped`
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
- Modify: `docs/site/docs/actions/ci-evidence-reports.md`,
  `docs/site/docs/actions/ci-evidence-emit.md` and
  `docs/site/docs/workflows/ci-quality.md` (the lint and typecheck
  contracts, the required reports, and the `quality-*` emit glob)
- Modify: `docs/site/docs/actions/cd-release-registry-publish.md` (Rust's
  build is `cargo build --release --locked --workspace` and its publish
  `cargo publish --locked`, from Tasks 2 and 13)
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
- Produces: the emit step's files are `evidence-reports/quality-*`,
  replacing the hardcoded ruff and mypy globs, as `ci-test.yml`'s emit uses
  a broad glob. The emit action skips patterns that match nothing, so
  before it runs, the step checks that every lint and typecheck contract
  file for the repo's language is present at least once in
  `evidence-reports/` (version-suffixed), and fails naming any missing
  file. The check is a new `reports.sh` mode,
  `expect <language> <dir>`.
- Python's staged file names and its tools list stay identical to today's.

- [ ] **Step 1: Write the failing tests** in `reports.test.sh`:
  - `lint_and_typecheck_are_gates`
  - `rust_lint_files_and_tools`
  - `rust_typecheck_files_and_tools`
  - `python_lint_matches_today`: the files and tools equal the strings
    `ci-quality.yml` hardcodes today.
  - `passing_leg_missing_report_fails`, for each of the four entries.
  - `failing_leg_missing_report_not_extra_error`
  - `expect_rust_finds_both_reports`: a directory holding
    `quality-clippy-1.99.json` and `quality-cargo-check-1.99.json` passes,
    and one with the clippy report removed fails, naming it.
  - `expect_python_matches_today`: the ruff and mypy reports, as staged
    today, pass.
  - `expect_no_contract_language_warns`: for `go`, `expect` warns through
    `warn_no_contract` and passes.
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
- Modify: `docker/pins/CATALOG.md` (regenerated) and
  `docs/site/docs/languages/rust.md` (prebuilt tools carry pinned digests)
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
- Modify: `docker/pins/CATALOG.md` (regenerated) and
  `docs/site/docs/languages/rust.md` (cargo-deny and cargo-llvm-cov are
  pinned and refreshed)
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
  - `test_exposure_report_resolves_rust_tools`: the pin-exposure report,
    which builds its tool map from `TOOL_REPOS` (vergil-containers #916),
    resolves cargo-deny and cargo-llvm-cov.
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

- [ ] **Step 1: Measure.** In the fixture's worktree, run both commands in
  the container: `vrg-container-run -- cargo llvm-cov nextest --locked
  --workspace --no-report`, then `vrg-container-run -- cargo llvm-cov report
  --fail-under-regions 100 --text --show-missing-lines`. Record the region
  percentage, and each uncovered region from the annotated text, in the
  issue.
- [ ] **Step 2:** If coverage is 100%, comment the measurement and close the
  task as done with no PR. Otherwise, add tests that exercise each uncovered
  region. Never add an exclusion.
- [ ] **Step 3: Run** `vrg-container-run -- vrg-validate`, and repeat
  Step 1's measurement. Expect 100% lines and regions.
- [ ] **Step 4: Commit** `vrg-commit --type test --scope coverage --message
  "cover every region"`, then report-ready.

### Task 13: Rust release build in `vrg-validate` (vergil-tooling)

Spec §6.9. Blocked by 6 (the same `languages.py` entry) and by the epic
vergil-project/.github#1174 task that adds the generic BUILD check kind.
That task is not yet filed. Its ref is added to this task's `Blocked-by:`
when it is, and this task's body names it as a precondition until then.

**Files:**

- Modify: `src/vergil_tooling/lib/languages.py` (the Rust entry enables
  BUILD; `EcosystemInfo.build_cmd`)
- Modify: `docs/site/docs/standards/development/rust/overview.md` (the
  release profile is built in validation)
- Test: `tests/vergil_tooling/test_languages.py`,
  `tests/vergil_tooling/test_vrg_validate.py`

**Interfaces:**

- Consumes: #1174's BUILD check kind and the way it enables a language, as
  that epic's plan defines them. This task does not change the mechanism.
- Produces: Rust's `build_cmd == ["cargo", "build", "--release", "--locked",
  "--workspace"]`, and BUILD enabled for Rust. The release job's build runs
  the same command, so validation and release build the same profile.

- [ ] **Step 1: Write the failing tests.**
  - `test_rust_build_cmd_is_locked_release_workspace`
  - `test_rust_build_check_enabled`: the Rust language runs BUILD with that
    command.
  - `test_rust_build_failure_fails_validate`: a failing build command fails
    the BUILD stage.
- [ ] **Step 2: Run** `vrg-container-run -- vrg-validate`. Expect the new
  tests to fail.
- [ ] **Step 3: Implement** the interfaces above, and update the standards
  page.
- [ ] **Step 4: Run** `vrg-container-run -- vrg-validate`. Expect PASS.
- [ ] **Step 4b: Live check** (Global Constraints) against
  `vergil-fixture-rust-lib`. Expect the BUILD stage to run the release
  build and pass, and record the output in the PR.
- [ ] **Step 5: Commit** `vrg-commit --type feat --scope rust --message
  "build the Rust release profile in vrg-validate"`, then report-ready.

### Task 12: Releases attested; pinned-tool Rust images (deployment, .github)

Blocked by 2–10, 11a, 11b and 13.

- **Precondition (human-attested):** the releases are published and their
  versions named in a comment:
  - vergil-tooling (Tasks 2–7 and 13), first;
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
  The rolled-back crate stays within its `Cargo.toml` requirement: a
  transitive crate, or a direct dependency whose floor is below its locked
  version. The change PR's locked install must pass on the rolled-back
  lockfile. In `vergil-fixture-rust-lib`, every direct dependency is locked
  at its floor, so the rollback there uses a transitive crate.
- The evidence bundle's `quality` gate lists `rustfmt`, `clippy` and `cargo`
  beside the common and custom tools, and holds the version-suffixed
  clippy and cargo-check reports. This replaces the earlier evidence-bundle
  known gap.
- The test gate passes at 100% lines and regions, and the install step runs
  `--locked`.
- `vrg-validate` runs the BUILD stage with `cargo build --release --locked
  --workspace`, and it passes before the release PR merges.

## Evolution during execution

### 2026-11-16: Task 3's region locator follows llvm-cov's own count

**What changed:**

- **Task 3's `rust_coverage.uncovered_regions` no longer merges a
  region's count across a generic function's instantiations.** It
  reproduces `cargo llvm-cov`'s own count. It groups a function's
  instantiations by source function (file and start position), takes the
  instantiation with the most covered code regions (ties go to the first
  in export order), and reports that instantiation's kind 0 regions with
  count 0. A function with one instantiation is handled as before. The
  authoritative scope is the human's
  [scope change on vergil-project/vergil-tooling#4472](https://github.com/vergil-project/vergil-tooling/issues/4472#issuecomment-6924781131).
- **Task 3's tests change to match.**
  `test_uncovered_regions_covered_in_one_instantiation` is replaced by a
  case in which a region at count 0 in the best-covered instantiation is
  listed even though another instantiation ran it. A new test checks the
  listing against the export's region totals: for an export below 100%
  regions, the listing is non-empty and its length equals the number of
  regions the totals count as missed.
- Task 3's own text above is unchanged. The plan froze when its first
  task to merge, Task 8 (vergil-project/vergil-actions#1249), merged.

**Why:**

- **The planned merge disagrees with the gate.** Measured on
  vergil-project/vergil-fixture-rust-app#18
  ([item N0.1](https://github.com/vergil-project/vergil-fixture-rust-app/issues/18#issuecomment-6923613164),
  [the human's ruling](https://github.com/vergil-project/vergil-fixture-rust-app/issues/18#issuecomment-6924742223)):
  for a generic function, llvm-cov's summary counts only the regions of
  its best-covered instantiation. The plan said the merge matched
  llvm-cov's summary, but it does not. With the planned merge, where a
  region counts as covered if any instantiation ran it,
  `--fail-under-regions 100` could fail while the listing was empty.

The entries below were written at the retrospective (vergil-project/.github#1167), because the epic's driver did not log them when the work was added. Each is dated to when the change happened.

### 2026-11-16: the fixture-run skill change is its own task

**What changed:** vergil-project/vergil-claude-plugin#1104 retires the `fixture-run` skill's two Rust known gaps and turns on the Cargo rollback. It was filed under #1165 at the filing step, blocked by Tasks 6 and 7 and by Task 8, but the delivery table above does not name it. It merged before the plugin release that the fixture runs depend on (v2.1.74).

**Why:** the fixture runs could prove the two fixes only once the skill stopped excusing them. The planning issue said this addition would be logged ([comment](https://github.com/vergil-project/.github/issues/1087#issuecomment-6923249677)).

### 2026-11-16: the plan amendment needed a task

**What changed:** vergil-project/.github#1198 was filed under #1165 to carry the Task 3 entry above.

**Why:** `vrg-git` refuses a branch named for an epic, so an edit to this plan needs a task of its own.

### 2026-11-16: the fixture runs were filed at implementation time

**What changed:** the two trios were filed with `fixture-run` once Task 12 (vergil-project/.github#1190) existed and plugin v2.1.74 was installed, blocked by Task 12. They are vergil-project/vergil-fixture-rust-lib#21–#23 and vergil-project/vergil-fixture-rust-app#20–#22, as the Filing note above says. Each validation task also carries this plan's four extra checks from "Fixture runs", as checks 8–11. The rust-lib change task carries the transitive-crate note.

**Why:** the trio's text comes from the installed `fixture-run` skill, which had to include #1104 first.

### 2026-11-16: the documentation review spawned three per-repo tasks

**What changed:** the review (vergil-project/vergil-tooling#4443) fixed vergil-tooling's own pages, and spawned:

- vergil-project/vergil-containers#929: comments and docs describing cargo-deny and cargo-llvm-cov as pinned Rust image tools;
- vergil-project/vergil-fixture-rust-lib#24 and vergil-project/vergil-fixture-rust-app#23: the fixtures' docs describe the lines-and-regions gate and the release build. Each was blocked by its fixture run's validation task, so neither landed inside a run.

**Why:** a review never edits across a repo boundary. Each other repo's fix is its own task, closed by a same-repo PR.
