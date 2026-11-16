# Rust validation, supply-chain and dependency-update parity — design

- **Epic:** vergil-project/.github#1165
- **Planning issue:** vergil-project/.github#1087 (review history is in its comments)
- **Status:** draft

## 1. Intent

A Rust repo is validated, audited and kept up to date with the same rigour as
a Python repo:

- its lockfile is current and enforced;
- the tools that validate it are pinned and verified;
- its coverage gate is as close to Python's as stable Rust allows;
- its CI evidence names its own tools and reports;
- `vrg-update-deps` updates its dependencies and reports what their
  requirements hold back.

## 2. Why

- **Rust is the 2027 target.** Vergil is to be rebuilt in Rust in 2027, with
  Vergil itself as the tool that does it. Epic vergil-project/.github#1095
  (Epic A of the Rust-parity wave) put the Rust toolchain on the leading edge.
  This epic (Epic B) brings the rest of the Rust gate up to Python's level.
- **The fixtures show the gaps.** The first fixture runs on
  `vergil-fixture-rust-lib` and `vergil-fixture-rust-app` (epic
  vergil-project/.github#1091) recorded two known gaps instead of passes:
  - no Cargo updater, so the dependency-rollback check was skipped;
  - an evidence bundle whose `quality` gate names ruff and mypy.

  The same runs are this epic's proof once those gaps are fixed.
- **Unenforced lockfiles and unpinned tools are supply-chain holes.** A
  validation that silently re-resolves `Cargo.lock` checks a dependency set
  nobody committed. A validation tool installed at whatever version crates.io
  serves at build time is a moving part of every Rust gate.

## 3. Goals, non-goals, anti-goals

### Goals

1. Every cargo command `vrg-validate` runs, and the release build and publish,
   uses the committed `Cargo.lock` unchanged. A stale or missing lockfile fails
   loudly, naming `Cargo.lock`. Libraries commit `Cargo.lock` too.
2. The Rust test gate requires 100% line coverage **and** 100% region
   coverage.
3. cargo-deny and cargo-llvm-cov are installed at pinned versions that are
   verified at install time, and the pin automation keeps them current.
4. Rust validation covers every crate in a workspace, and `vrg-version` reads
   and bumps a workspace's version.
5. The CI evidence bundle's `quality` gate, for a Rust repo, holds Rust's lint
   and typecheck reports and names Rust's tools.
6. `vrg-update-deps` updates a Rust repo's `Cargo.lock` within the
   requirements in `Cargo.toml`, and reports each direct dependency those
   requirements hold back.
7. The Rust fixtures' dependency-rollback and evidence-bundle checks pass.

### Non-goals

- Branch coverage. It needs a nightly toolchain (§4, §6.2); it is deferred to
  vergil-project/.github#1171.
- Taking the Rust lifecycle window from Rust's release channel:
  vergil-project/vergil-containers#910.
- Merging lifecycle PRs automatically: vergil-project/.github#1128.
- Editing `Cargo.toml` requirements to newer major versions (the equivalent of
  `cargo upgrade`). Python's updater does not edit `pyproject.toml` either.
- C++, TypeScript and Go parity; a cargo builder for `vrg-package`; the Rust
  beta channel (vergil-project/.github#424); wider Rust docs; the
  Python-to-Rust library map.

### Anti-goals (the epic fails if any of these happen)

- A gate quietly re-resolves or rewrites `Cargo.lock` during validation.
- A Rust check is weakened to pass: a lower threshold, a skipped crate, or an
  exclusion that hides untested code.
- A Python gate changes behaviour. The per-language changes leave Python's
  commands, reports and evidence byte-for-byte as they are.
- A tool is installed unverified, or a verification failure is downgraded to
  a warning.
- The Cargo updater swallows a failure, or reports "no change" when `cargo
  update` failed.

## 4. Current state (facts the design builds on)

Checked on `origin/develop` on 2026-11-15: vergil-tooling `60ac9e5cce`,
vergil-actions `4f76bf326`, vergil-containers `2dde127`.

### Already delivered (not this epic)

- A packaged `deny.toml` and license allowlist, passed with `--config`
  (vergil-tooling `configs/rust/deny.toml`; vergil-tooling#4358, #4368, #4396).
- The Rust gitignore fragment (`target/`, `cargo-deny.json`; vergil-tooling#4357).
- One CodeQL language list, `CODEQL_LANGUAGES` (vergil-tooling#4381).
- cargo-nextest JUnit output (vergil-tooling#4396). cargo-nextest itself is
  pinned and verified (vergil-containers#890).
- Per-language evidence for the `test` and `audit` gates (vergil-actions#1211,
  #1216).
- clippy runs with `--all-targets` (vergil-tooling#4385).

### vergil-tooling

- The Rust gate commands are in `lib/languages.py`, `"rust": Language(…)`:
  - `cargo fetch` (install);
  - `cargo fmt --all -- --check` and `cargo clippy --all-targets -- -D
    warnings` (lint);
  - `cargo check` (typecheck);
  - `_RUST_TEST_SCRIPT` and `cargo test --doc` (test);
  - `_RUST_AUDIT_SCRIPT` (audit);
  - `cargo build --release` and `cargo publish` (ecosystem).

  None of these commands passes `--locked`, `--frozen` or `--workspace`. The
  only exception is `cargo llvm-cov clean --workspace`.
- `vrg_validate.py` reads `cargo metadata --no-deps --format-version 1`. It
  uses the result to drop `cargo test --doc` when no default-member package has
  a library target (`_skip_rust_doctests_without_lib`).
- `_RUST_TEST_SCRIPT` gates on `cargo llvm-cov report --fail-under-lines 100`.
  Python's gate is `--cov-branch --cov-fail-under=100`.
- Report-file constants:
  - Python: `RUFF_REPORT = "quality-ruff.json"` and `MYPY_REPORT =
    "quality-mypy.xml"`, collected in `PYTHON_REPORT_FILES`.
  - Rust: `RUST_REPORT_FILES` holds `coverage.xml`, `junit.xml` and
    `cargo-deny.json`. It has no quality report.
- `lib/version.py` reads `data["package"]["version"]`. A workspace root with
  only `[workspace.package]` raises `KeyError`. A bump runs `cargo update
  --workspace` to refresh the lockfile.
- `vrg-update-deps` (`lib/update_deps/`) has these parts:
  - **The registry.** `DEFAULT_REGISTRY` in `orchestrator.py` holds
    `PythonUvUpdater`, `VergilUpdater`, `ImagesUpdater` and `RepoHookUpdater`.
    It has no Cargo updater.
  - **The updater protocol.** `Updater` in `updater.py` has `name`,
    `applies(ctx)` and `apply(ctx) -> UpdateResult`.
  - **The result.** `UpdateResult` carries `changed`, `summary`,
    `commit_message`, `warnings` and `held_back: list[HeldBack]`.
  - **The held-back report.** It works for any language (`report.py`:
    `HeldBack`, `collect`, `render_markdown`), and `finalize_stage` prints it
    on every run.
  - **The Python updater.** `updaters/python_uv.py` runs `vrg-container-run --
    uv lock --upgrade`. It then compares each bounded direct requirement's
    locked version with the latest version on PyPI.

### vergil-actions

- `actions/ci/evidence/reports/reports.sh` is the per-language report
  contract. `has_contract`, `report_files` and `report_tools` cover
  `python/test`, `python/audit`, `rust/test` and `rust/audit` only.
- `.github/workflows/ci-quality.yml` handles the `quality` gate:
  - Its lint and typecheck legs copy `quality-ruff.json` and
    `quality-mypy.xml` into `ci-reports/` with the leg's version suffix.
  - Its "Emit ci-evidence-quality artifact" step hardcodes the globs
    `quality-ruff-*.json` and `quality-mypy-*.xml` (lines 413-417). It also
    hardcodes the tools `vrg-validate-common`, `vrg-validate-custom`, `ruff`
    and `mypy` for every language.
  - CI runs the Rust checks through `vrg-validate --check {lint,typecheck}`,
    so CI and local validation share the commands in vergil-tooling.

### vergil-containers

- `docker/rust/Dockerfile.template:79-80` runs `cargo install cargo-deny &&
  cargo install cargo-llvm-cov`, with no version and no `--locked`.
- `docker/common/cargo-nextest.dockerfile` is the pattern to follow:
  - a prebuilt release tarball, verified against the release's per-asset
    `.sha256` with `sha256sum -c`;
  - `ARG CARGO_NEXTEST_VERSION`, auto-managed in `docker/pins/pins.yml` and
    floated by the daily upstream refresh;
  - its latest version resolved in `resolve_latest.py`.
- `docker/pins/extract_pins.py` can read a `cargo install <crate>@X.Y.Z` pin.

### Upstream (checked 2026-11-15)

- **Branch coverage is nightly-only.** cargo-llvm-cov marks `--branch`
  unstable: "branch coverage is currently optional and requires nightly"
  (<https://github.com/taiki-e/cargo-llvm-cov>, issue #8). rustc lists `-Z
  coverage-options` as unstable
  (<https://doc.rust-lang.org/rustc/instrument-coverage.html>). Region coverage
  and `--fail-under-regions` work on stable.
- **cargo-deny 0.20.2** publishes `x86_64-` and `aarch64-unknown-linux-musl`
  tarballs, each with a `.sha256`. That matches the nextest pattern.
- **cargo-llvm-cov v0.9.1** publishes unversioned asset names
  (`cargo-llvm-cov-x86_64-unknown-linux-gnu.tar.gz`) with no checksum files,
  only GitHub artifact attestations. Its README's from-source install is
  `cargo +stable install cargo-llvm-cov --locked`.
- Stable Rust has no way to exclude code from coverage: `#[coverage(off)]` is
  unstable.

## 5. Approaches considered

- **A. One epic, tasks grouped by repo, then one release-and-fixture round
  (chosen).** All the changes land behind a single release of each repo, and
  one round of fixture runs proves all of it.
- **B. Two epics: validation, then the Cargo updater.** This is cleaner on
  paper, but it costs two release rounds and two rounds of fixture runs for
  one small wave of work.
- **Folding in the lifecycle inputs from vergil-project/.github#1097**
  (taking the window from Rust's channel, and #1128's automatic merging) was
  rejected. They are lifecycle automation, not validation. #1128 changes
  policy for every language, and it builds on the fleet-sync framework.

For coverage, the options considered were:

- 100% lines and regions (chosen);
- lines only, with the gap documented;
- lines, regions and branches on a nightly toolchain, rejected because it
  breaks the stable-only toolchain pin from #1095.

## 6. Design

### 6.1 The lockfile is enforced (vergil-tooling)

- **`--locked` everywhere `Cargo.lock` is only an input.** Every Rust gate
  command in `languages.py` takes `--locked`: fetch, clippy, check, the
  llvm-cov/nextest test run, `cargo test --doc` and `cargo deny`. So does
  `cargo metadata` in `vrg_validate.py`, and so do the ecosystem build and
  publish commands.
  - `cargo fmt` reads no lockfile and is left alone.
  - The two commands that exist to change the lockfile stay unlocked:
    `version.py`'s `cargo update --workspace` and the Cargo updater's `cargo
    update` (§6.6).
- **Missing `Cargo.lock` fails.** Validation of a Rust repo with no committed
  `Cargo.lock` fails with an error naming the file and how to create it
  (`cargo generate-lockfile`, then commit it). This holds for libraries and
  applications alike, following Cargo's guidance since 2023 to commit the
  lockfile for libraries too.
  - If `--locked` alone does not produce that failure and message, an explicit
    check before the install step does. The behaviour is the requirement; the
    mechanism is the builder's.
- **Stale `Cargo.lock` fails.** A `Cargo.toml` change that needs a lockfile
  update fails the install step. Cargo's own error, which names `--locked`,
  is surfaced unchanged.
- The Rust standards page in the vergil-tooling site docs says `Cargo.lock` is
  committed and enforced.

### 6.2 Coverage: 100% lines and regions (vergil-tooling)

- `_RUST_TEST_SCRIPT`'s report step gates on both `--fail-under-lines 100`
  and `--fail-under-regions 100`. Missing regions are shown as missing lines
  are (`--show-missing-lines`). The builder confirms whether cargo-llvm-cov
  can show anything finer.
- **Regions are the stable stand-in for branches.** Each arm of an `if`, a
  `match` or a `?` early return is its own region. So 100% regions requires
  the cases Python's branch gate requires, as far as stable Rust can measure
  them.
- **No exclusion escape hatch.** Stable Rust cannot mark code as excluded from
  coverage. The gate offers no `--ignore-filename-regex` or other exclusion
  beyond what exists today. An unreachable region is a design signal (remove
  it, or make it testable), not something to exclude.
- **Measure before switching on.** Before the gate changes, measure both
  fixtures' region coverage with the new flag. If either falls short, the
  fixture's tests are brought to 100% as part of the same change, in the
  fixture's repo, before the vergil-tooling release that ships the gate.
- The Rust standards page states the gate (lines and regions), and states that
  branch coverage is deferred to #1171.

### 6.3 Pinned, verified validation tools (vergil-containers)

- **cargo-deny** follows the cargo-nextest pattern:
  - a new `docker/common/cargo-deny.dockerfile`;
  - `ARG CARGO_DENY_VERSION`;
  - the release's `<arch>-unknown-linux-musl` tarball, verified with
    `sha256sum -c` against its `.sha256`;
  - an auto-managed `pins.yml` entry, floated by the daily upstream refresh,
    with its latest version resolved in `resolve_latest.py`.
- **cargo-llvm-cov** is installed with `cargo install --locked
  cargo-llvm-cov@<version>`. Its release assets have no checksum files, and
  their names carry no version.
  - Cargo verifies the crate checksum against the crates.io index, and
    `--locked` builds it from its own published lockfile.
  - The version is a pin that `extract_pins.py` reads, auto-managed and
    floated the same way.
  - The registry cache is removed afterwards, as it is today.
- A pin change rebuilds the Rust images like any other auto-managed pin. A
  checksum mismatch, a missing asset, or a failed install fails the image
  build.

### 6.4 Workspaces (vergil-tooling)

- **Every crate is validated.** clippy, check, the coverage test run and
  `cargo test --doc` run with `--workspace`. That way a workspace whose root
  is a package also validates its member crates, not just the root.
  - `_skip_rust_doctests_without_lib` checks every workspace member, not just
    the default members, to match.
  - A single-crate repo behaves as it does today.
- **`vrg-version` reads a workspace version.**
  - With no `[package]` table, it reads `[workspace.package].version`.
  - A bump writes the version back where it was read from. Member crates that
    use `version.workspace = true` follow it.
  - A `Cargo.toml` with neither table fails with a clear error naming both
    tables, not a `KeyError`.

### 6.5 The `quality` gate's evidence is per-language (vergil-tooling, vergil-actions)

- **vergil-tooling writes Rust's quality reports** at the workspace root:
  - clippy's machine-readable diagnostics, in `--message-format=json` form;
  - `cargo check`'s, in the same form.

  Their names are contracts beside `RUFF_REPORT` and `MYPY_REPORT` (proposed:
  `quality-clippy.json` and `quality-cargo-check.json`). They are added to
  `RUST_REPORT_FILES` and the Rust gitignore fragment. The human-readable
  output in the check's log stays as it is today, so nobody loses the
  readable diagnostics. A failing lint or typecheck still writes its report
  before it exits non-zero.
- **vergil-actions derives the quality gate from `reports.sh`.**
  - It adds `python/quality` and `rust/quality` entries to `has_contract`,
    `report_files` and `report_tools`.
  - Python's entry reproduces today's files and tools exactly: ruff and mypy,
    plus the common and custom logs.
  - Rust's entry names `vrg-validate-common`, `vrg-validate-custom`, `rustfmt`
    and `clippy`, plus `cargo` for the typecheck. Its files are the common and
    custom logs and the two reports above.
  - `ci-quality.yml`'s lint and typecheck legs, and its emit step, take the
    files and tools from `reports.sh` instead of hardcoding them.
  - A Rust leg's reports are staged with the leg's version suffix, as Python's
    are.
- Languages with no quality contract keep today's warning path in
  `reports.sh`, which warns and gives an empty payload. They are not given
  Python's tools.

### 6.6 The Cargo updater (vergil-tooling)

- **`CargoUpdater` in `lib/update_deps/updaters/cargo.py`.** It is registered
  in `DEFAULT_REGISTRY` and implements the `Updater` protocol.
  - `applies(ctx)` needs both `Cargo.toml` and `Cargo.lock`.
  - `apply(ctx)` runs `vrg-container-run -- cargo update`. That updates every
    locked dependency to the newest version its `Cargo.toml` requirement
    allows, and edits no `Cargo.toml`. It then checks whether `Cargo.lock`
    changed.
  - Its summary and commit message name the crates that moved, from and to,
    in the style of the uv updater.
- **The held-back report.** The updater reads each direct dependency of every
  workspace member, from the normal, dev and build dependency tables. A
  dependency is held back when the newest stable version on crates.io does
  not satisfy its requirement.
  - Each held-back dependency becomes a `HeldBack` entry with its locked and
    latest versions.
  - Path, git and workspace-inherited dependencies resolve to the requirement
    they actually carry, or are skipped with a reason that the builder records
    in the code.
  - crates.io is queried with a generic client `User-Agent`, as crates.io
    requires of automated clients. No personal or project detail goes in it.
- **The minimum supported Rust version.** Cargo's resolver may prefer older
  versions compatible with a package's `rust-version`. If a dependency is
  locked below what its requirement allows for that reason, the held-back
  entry says so, so the report never misattributes a hold.
- **Failures are loud.** These all fail the updater's result:
  - a failed `cargo update`;
  - an unreadable `Cargo.toml` or `Cargo.lock`;
  - a crates.io query that fails.

  None of them reads as "no change". A crates.io failure is an error, not a
  silently empty held-back list.

### 6.7 Docs

The documentation review (vergil-project/vergil-tooling#4443) sweeps the
human-facing docs for every change above. That covers at least:

- the Rust standards page and the validation reference: lockfile, coverage and
  workspaces;
- the `vrg-update-deps` page: the Cargo updater and its held-back report;
- vergil-actions' evidence docs: the per-language `quality` gate;
- vergil-containers' image docs: the tool pins.

### 6.8 Rollout and validation

1. Land the vergil-tooling, vergil-containers and vergil-actions changes, and
   any fixture test additions §6.2 needs.
2. **Release and deploy.** The human releases vergil-tooling, vergil-actions
   and vergil-containers. The containers release publishes Rust images with
   the pinned tools. A deployment task records that the releases are out and
   the images published. The releases themselves are a human-gated
   precondition, attested and never cut by an agent.
3. **Fixture runs** on `vergil-fixture-rust-lib` and `vergil-fixture-rust-app`,
   blocked by the deployment task, filed with `fixture-run`. Each run:
   - rolls one crate back in `Cargo.lock`, and checks that the release's
     `vrg-update-deps` restores it with a Cargo update commit;
   - checks the evidence bundle's `quality` gate names Rust's tools and holds
     its reports;
   - passes the new coverage gate and the locked install.

   The two known-gap checks from the first fixture runs become passing checks.

## 7. Failure model

- **`Cargo.lock` missing:** validation fails at install, naming `Cargo.lock`
  and how to create it.
- **`Cargo.lock` stale:** validation fails at install with cargo's `--locked`
  error.
- **Region coverage below 100%:** the test gate fails, listing what is
  missing.
- **Tool checksum mismatch or missing asset:** the image build fails.
- **`cargo update` fails:** the Cargo updater's result fails, never "no
  change".
- **crates.io unreachable during the held-back check:** the updater fails
  loudly. The held-back list is never silently empty.
- **A Rust quality report is missing in CI:** the emit step fails, as a
  missing Python report does today.
- **A `Cargo.toml` with neither `[package]` nor `[workspace.package]`:**
  `vrg-version` fails, naming both tables.

## 8. Constraints

- **Stable Rust only.** The Rust image pins its toolchain and turns rustup
  auto-install off (#1095). Nothing in this epic may need nightly.
- **Python stays unchanged.** Python's gate commands, report names, evidence
  files and tools stay byte-for-byte the same (an anti-goal, §3).
- **Validation runs through `vrg-validate`.** CI calls `vrg-validate --check
  …` and holds no cargo commands of its own. This epic keeps that split.
- **Releases are human actions.** Agents never cut a release.
- **Placement law.** Each task lives in the repo its PR lands in.

## 9. Related work

- vergil-project/.github#1095: Epic A, the Rust toolchain lifecycle.
- vergil-project/.github#1091: the language fixture repos, and the source of
  the known gaps.
- vergil-project/.github#384: restarting the language updaters. This epic
  delivers its Rust part.
- vergil-project/.github#1171: branch coverage, deferred.
- vergil-project/vergil-containers#910: the Rust window from Rust's own
  channel.
- vergil-project/.github#1128: automatic merging of lifecycle PRs.

## 10. Left to the builder

- The exact names of the Rust quality report files, kept consistent across
  vergil-tooling and vergil-actions.
- Whether a missing `Cargo.lock` needs an explicit check or is covered by
  `--locked` alone (§6.1).
- How the Cargo updater reads the locked and latest versions: by parsing
  `Cargo.lock` and `cargo metadata`, or by reading `cargo update`'s output.
- How `CargoUpdater` treats path, git and workspace-inherited dependencies in
  the held-back report.
- How the resolver for the latest cargo-deny version finds it (the GitHub
  latest release, or crates.io), following `resolve_latest.py`'s existing
  patterns.
