# C++ language support — Implementation Plan

> **For agentic workers:** tasks are GitHub issues under epic
> vergil-project/.github#207, each 1:1 with a finalizing PR in the repo named.
> Steps use checkbox (`- [ ]`) syntax. Drive with `epic-implement`; run each code
> task via `issue-implement`, each operational task via `issue-deploy` /
> `issue-validate`. Task numbers below are plan-local labels (T1…T11); the GitHub
> issue numbers are assigned when the tasks are filed (see the epic).

**Goal:** add **C++** as a first-class supported language — two open-source
compilers (Clang primary, GCC secondary) run as independent CI diagnostic
engines, on the existing containerized multi-version model — built from the
ground up across five repos.

**Architecture:** container images in `vergil-containers` carry the toolchains
and the shared analysis toolset; `vergil-tooling` gains the registry entry, the
`[cpp]` schema, a new per-kind check-cardinality concept, and the container/
CodeQL/repo-init wiring; `vergil-actions` accepts `language: cpp` in the reusable
CI workflows; standards docs and an end-to-end cold-rebuild validation close it
out. Compiler×version is the image/CI-gate axis; C++ standard and stdlib are cheap
in-build flags.

**Tech stack:** Python (`vergil_tooling` CLIs + `pytest`); Docker (dev/prod
images); CMake + Conan 2; Clang/LLVM + GCC toolchains; `clang-format`,
`clang-tidy`, `cppcheck`, `gcovr`, CTest/GoogleTest, ASan/UBSan; GitHub Actions
reusable workflows; MkDocs site docs.

## Global constraints

- **Validation:** `vrg-container-run -- vrg-validate` is the only validation
  command (in `vergil-tooling` it expands to `uv run vrg-validate` via the
  `[validation]` override). Git via `vrg-git` / `vrg-commit`; PRs via
  `vrg-submit-pr` (human-gated — agents stop at `vrg-pr-workflow report-ready`).
- **Placement law:** each task lives in the repo where its PR lands; a PR only
  `Closes` a same-repo issue. Cross-repo links are `Ref`.
- **Dual compilers, both first-class:** Clang primary, GCC secondary. Warnings to
  11 (`-Wall -Wextra -Werror -Wpedantic` + curated extras), no standing
  suppression list.
- **Coverage:** 100% **per compiler** via `gcovr`; `// GCOVR_EXCL_*` markers
  reserved for genuinely-unreachable runtime branches (not routine `#ifdef`s).
- **Build/deps:** CMake + Conan 2. Test runner is **CTest** (framework-agnostic;
  GoogleTest is the documented default).
- **Prebuilt stable toolchains only** — never built from source. Two recent
  majors per family; v1 target `clang-19/20` + `gcc-14/15`, fallback
  `clang-18/19` + `gcc-13/14` if the newer majors are not cleanly prebuilt.
- **Per-kind cardinality:** TYPECHECK and TEST run **per compiler×version**;
  LINT and AUDIT run **once** (on the primary Clang image).
- **v1 pins:** standard `c++20`; stdlib `libstdc++` (single values). Everything
  else is on the spec §9 deferral ledger.

---

### T1 — Common C++ image base + Clang family (vergil-containers)

**Files:**
- Create: `docker/cpp/Dockerfile.base` (shared analysis-tool layer)
- Create: `docker/cpp/Dockerfile.clang`
- Modify: `docker/build.sh` (register the new image targets)

Foundational. Establishes the shared layer every C++ image inherits, then the
Clang family on top. **Includes the toolchain-availability discovery** that pins
the concrete majors.

- [ ] **Discovery (prebuilt-only, §3.5).** Determine which Clang majors are
  available as prebuilt stable binaries via the chosen base image / official LLVM
  apt channel; pin the two most recent that qualify (target `19`, `20`; fall back
  to `18`, `19`). Record the decision in the image dir (short `README`/comment).
- [ ] **Shared base layer.** Install the compiler-agnostic analysis toolset every
  image carries: `clang-format`, `clang-tidy`, `cppcheck`, `gcovr`, CMake,
  Conan 2, plus the sanitizer runtimes. Pin versions.
- [ ] **Clang images.** `prod-cpp-clang:<v>` and `dev-cpp-clang:<v>` for each
  pinned major, with `CC`/`CXX` defaulting to that clang and libstdc++ as the
  stdlib. Set the env so `vrg-container-run` and CMake pick the right compiler.
- [ ] **Smoke test.** A trivial CMake+Conan project compiles, links, and runs
  under each Clang image (build script check).
- [ ] **Validate + commit** (`feat(cpp): clang dev/prod images + shared C++ base`).

**Deliverable:** buildable Clang-family images carrying the full toolset.
**Blocks:** T2, T8, T10.

### T2 — GCC family (vergil-containers)

**Files:**
- Create: `docker/cpp/Dockerfile.gcc`
- Modify: `docker/build.sh`

- [ ] **Discovery.** Pin the two most recent prebuilt GCC majors (target `14`,
  `15`; fall back to `13`, `14`), same prebuilt-only rule.
- [ ] **GCC images.** `prod-cpp-gcc:<v>` / `dev-cpp-gcc:<v>` on the T1 shared
  base, `CC`/`CXX` defaulting to that gcc, libstdc++ stdlib.
- [ ] **Smoke test** under each GCC image; confirm the shared analysis tools
  (clang-tidy/clang-format/cppcheck/gcovr) are present and runnable here too.
- [ ] **Validate + commit** (`feat(cpp): gcc dev/prod images on shared C++ base`).

**Deliverable:** buildable GCC-family images.
**Depends on:** T1. **Blocks:** T8, T10.

### T3 — Per-kind check cardinality (vergil-tooling)

**Files:**
- Modify: `src/vergil_tooling/lib/languages.py` (CheckKind cardinality concept)
- Modify: `src/vergil_tooling/lib/github_config.py:224` (`desired_ci_gates_ruleset`)
- Test: `tests/lib/test_languages.py`, `tests/lib/test_github_config.py`

Language-agnostic infra change; lands first among the tooling edits because the
C++ registry entry (T5) expresses LINT/AUDIT as `once`.

- [ ] **Cardinality metadata.** Add a `per-version` / `once` attribute to the
  check-kind structure (default `per-version` so existing languages are
  unchanged). Unit-test the default and an explicit `once`.
- [ ] **Gate emission.** Teach `desired_ci_gates_ruleset` to emit `once` kinds a
  **single** required check (keyed to the primary version) and `per-version` kinds
  per version. Unit-test that a `once`-kind language yields exactly one gate for
  that kind and N for a per-version kind.
- [ ] **Regression.** Existing five languages' generated gates are byte-identical
  (snapshot test).
- [ ] **Validate + commit** (`feat(github-config): per-kind CI-gate cardinality`).

**Deliverable:** the registry can declare a check kind as run-once, and gate
generation honors it.
**Blocks:** T5, T8, T11.

### T4 — `[cpp]` config schema + language enum (vergil-tooling)

**Files:**
- Modify: `src/vergil_tooling/lib/config.py:30` (`_ENUMS["primary-language"]`) and
  the `[cpp]` block parse/validate
- Test: `tests/lib/test_config.py`

- [ ] **Enum.** Add `"cpp"` to `_ENUMS["primary-language"]`. Test that
  `primary-language = "cpp"` is accepted and an unknown value still warns.
- [ ] **`[cpp]` block.** Parse and validate `std` (e.g. `c++20`) and `stdlib`
  (e.g. `libstdc++`) as single string values, with defaults when the block is
  absent. Test valid values, defaults, and rejection of nonsense.
- [ ] **Validate + commit** (`feat(config): add cpp primary-language + [cpp] block`).

**Deliverable:** `vergil.toml` accepts `primary-language = "cpp"` and a `[cpp]`
block.
**Blocks:** T5, T6.

### T5 — C++ language registry entry (vergil-tooling)

**Files:**
- Modify: `src/vergil_tooling/lib/languages.py:115` (`_REGISTRY`) and license
  allowlist constants
- Create: `src/vergil_tooling/configs/cpp/` (packaged `clang-format`,
  `clang-tidy`, `cppcheck`, `gcovr` configs)
- Test: `tests/lib/test_languages.py`

The meaty task — the concrete tool commands. Depends on T3 (cardinality) and T4
(schema).

- [ ] **INSTALL.** `conan install . --build=missing` then a `cmake` configure with
  `CMAKE_EXPORT_COMPILE_COMMANDS=ON` and the `[cpp]` `std`/`stdlib` threaded in as
  cache vars.
- [ ] **LINT (`once`).** `clang-format --dry-run --Werror`, `clang-tidy` (over
  `compile_commands.json`), `cppcheck --enable=all --error-exitcode=1`, using the
  packaged `{configs}/cpp/*` configs.
- [ ] **TYPECHECK (`per-version`).** The warnings-to-11 build
  (`-Wall -Wextra -Werror -Wpedantic` + the curated extras finalized in T9) under
  the image's compiler.
- [ ] **TEST (`per-version`).** `ctest --output-on-failure`; coverage via
  `gcovr --fail-under-line 100` (using `gcov` on GCC, `llvm-cov gcov` on Clang); a
  second `-fsanitize=address,undefined` build+run.
- [ ] **AUDIT (`once`).** `conan audit` (with the OSV-Scanner-over-`conan.lock`
  contingency noted if the provider needs paid infra — spec §4) + best-effort
  Conan license-metadata check; add a C++ license allowlist constant.
- [ ] **EcosystemInfo + cardinality.** Fill `EcosystemInfo` (build/publish/env);
  mark LINT and AUDIT `once`, TYPECHECK and TEST `per-version`.
- [ ] **Tests.** `language_commands("cpp", kind)` returns the expected argv per
  kind, `{configs}` expands, and cardinality is as declared.
- [ ] **Validate + commit** (`feat(languages): C++ registry entry (clang/gcc)`).

**Deliverable:** `vrg-validate` knows every C++ stage command.
**Depends on:** T3, T4. **Blocks:** T6, T8, T9, T11.

### T6 — Container maps + language detection (vergil-tooling)

**Files:**
- Modify: `src/vergil_tooling/lib/container.py` (`_DEFAULT_VERSIONS`,
  `_DEFAULT_TEST_COMMANDS`, `detect_language`, image resolution)
- Test: `tests/lib/test_container.py`

- [ ] **Detection.** `detect_language` returns `"cpp"` on `CMakeLists.txt` +
  `conanfile.py`/`conanfile.txt`. Test each marker.
- [ ] **Image resolution.** Parse the compiler family from the `clang-`/`gcc-`
  version-tag prefix and build `prod-cpp-<family>:<v>` (matching T1/T2 image
  names). Test `clang-20 → prod-cpp-clang:20`, `gcc-15 → prod-cpp-gcc:15`.
- [ ] **Defaults.** `_DEFAULT_VERSIONS["cpp"]` (the primary Clang major) and
  `_DEFAULT_TEST_COMMANDS["cpp"]` (a `conan install && cmake && ctest` line for
  `vrg-container-test`).
- [ ] **Validate + commit** (`feat(container): C++ image resolution + detection`).

**Deliverable:** `vrg-container-run`/`vrg-container-test` select the right C++
image; detection works.
**Depends on:** T4, T5. **Blocks:** T11.

### T7 — CodeQL list fix + repo-init wiring (vergil-tooling)

**Files:**
- Modify: `src/vergil_tooling/lib/github_config.py:224`
  (`_CODEQL_SUPPORTED_LANGUAGES`)
- Modify: `src/vergil_tooling/lib/repo_init.py` (`_container_suffix`,
  `_container_tag`, and `_LANGUAGE_ACTION_PATTERNS` if needed)
- Test: `tests/lib/test_github_config.py`, `tests/lib/test_repo_init.py`

- [ ] **CodeQL.** Add `"cpp"` to `_CODEQL_SUPPORTED_LANGUAGES`, aligning it with
  `repo_init._CODEQL_LANGUAGES` (which already lists `cpp`) — fixes the confirmed
  discrepancy. Test that a `cpp` repo gets the CodeQL gate.
- [ ] **repo-init.** `_container_suffix`/`_container_tag` produce the
  compiler-family-aware suffix/tag for generated `vergil.toml`/CI; add any needed
  allowed-action patterns. Test the generated scaffolding for a `cpp` repo.
- [ ] **Validate + commit** (`fix(github-config): support cpp for CodeQL + repo-init`).

**Deliverable:** CodeQL runs for C++ repos; `repo-init` scaffolds C++ correctly.
**Depends on:** T4. **(Independent of T5/T6; can run in parallel.)**

### T8 — Reusable CI workflows accept `language: cpp` (vergil-actions)

**Files:**
- Modify: the reusable `.github/workflows/ci-*.yml` (quality/test/audit/security)
- Test: workflow-level (a dry-run / sample invocation)

- [ ] **Language input.** Accept `language: cpp` and the compiler-family container
  suffix so jobs pull `prod-cpp-clang`/`prod-cpp-gcc` images by version tag.
- [ ] **Cardinality alignment.** Ensure the generated job/gate names match what
  T3 emits (per-version typecheck/test; once lint/audit) so required checks line
  up with branch protection.
- [ ] **Semgrep/CodeQL.** Wire `p/c`/`p/cpp` Semgrep rules and CodeQL `cpp`.
- [ ] **Validate + commit** (`feat(ci): C++ support in reusable workflows`).

**Deliverable:** reusable workflows run the C++ pipeline per compiler×version.
**Depends on:** T1, T2, T3, T5. **Blocks:** T11.

### T9 — C++ standards docs (vergil-tooling · site docs)

**Files:**
- Create: `docs/site/docs/standards/development/cpp/overview.md`,
  `naming-conventions.md`, `testing-and-coverage.md`, `toolchain-and-warnings.md`

- [ ] **Overview.** The CMake + Conan 2 layout, the dual-compiler model, the
  prebuilt-only rule, GoogleTest-on-CTest default.
- [ ] **Warnings.** Finalize and document the concrete "curated extras" warning
  set per compiler (spec §3.2) and the no-suppression rule.
- [ ] **Testing & coverage.** The 100%-per-compiler gate and the **exclusion
  discipline** — what a legitimate `GCOVR_EXCL` looks like vs. abuse (whole-file
  exemptions), and that compiler-specific `#ifdef` blocks are a smell to minimize.
- [ ] **Validate + commit** (`docs(cpp): C++ development standards`).

**Deliverable:** per-language standards docs matching the `python/`/`go/` pattern.
**Depends on:** T5 (final tool commands). **(Doc-review sweep #2551 may spawn
siblings in `docs`.)**

### T10 — Publish C++ images to GHCR (vergil-containers) · **deployment**

Operational (not PR-workable). Run with `issue-deploy`.

- [ ] **Precondition self-check.** T1 + T2 images build cleanly (probe).
- [ ] **Publish.** Push `prod-cpp-clang`/`prod-cpp-gcc` (+ `dev-` variants) for
  each pinned version to GHCR so `vrg-container-run` and CI can pull them.
  **Any release/tag step is a human-gated precondition** (attested, not performed
  by the agent).
- [ ] **Record** `Outcome: SUCCESS` (image refs + digests) as a comment.

**Deliverable:** C++ images are pullable from GHCR.
**Blocked-by:** T1, T2. **Blocks:** T11.

### T11 — Cold-rebuild + full C++ pipeline validation (vergil-tooling) · **validation**

Operational (not PR-workable). Run with `issue-validate`. The end-to-end proof.

- [ ] **Precondition self-check.** Images deployed (T10), registry/actions merged
  (T3–T8).
- [ ] **Sample repo.** Stand up a minimal C++ repo (CMake + `conanfile` + a
  GoogleTest suite) declaring `primary-language = "cpp"` and the four-image
  `[ci].versions`.
- [ ] **Cold rebuild.** From clean, `vrg-container-run -- vrg-validate` passes
  every stage under **each** compiler×version: warnings-clean build, 100%
  per-compiler coverage, ASan/UBSan clean, `conan audit`, LINT/AUDIT once.
- [ ] **CI gates.** Confirm `desired_ci_gates_ruleset` emits the expected
  per-version and once gates and they match the vergil-actions job names.
- [ ] **Record** `Outcome: SUCCESS` (or FAILURE with detail; stays open on
  failure) as a comment.

**Deliverable:** demonstrated, reproducible C++ pipeline end-to-end.
**Blocked-by:** T3, T4, T5, T6, T7, T8, T10.

---

## Closing bookends (already seeded)

- **Documentation-review sweep** — vergil-tooling#2551 (this repo's site docs +
  spawns per-repo doc siblings, incl. `docs`); runs **before** the retrospective.
- **Follow-on brainstorm** — .github#209 (adjudicate the spec §9 deferral ledger).
- **Retrospective** — .github#210 (terminal; its docs PR closes the epic).

## Self-review

- **Spec coverage.** §3.1 dual compilers → T1/T2/T8; §3.2 warnings-to-11 → T5/T9;
  §3.3 matrix model + `[cpp]` → T4/T6; §3.4 CMake+Conan → T5; §3.5 prebuilt-only →
  T1/T2 discovery; §3.6 per-kind cardinality → T3; §4 tool matrix → T5 (+ T1/T2
  toolset, T8 SAST); §5 per-compiler coverage → T5/T9/T11; §6 touch points →
  T3–T7; §7 five-repo spread → all; §9 ledger → follow-on brainstorm #209; §10
  operational tasks → T10/T11.
- **Placement law.** Every code task is filed in the repo whose PR closes it
  (vergil-containers: T1/T2; vergil-tooling: T3–T7, T9; vergil-actions: T8);
  operational tasks T10 (vergil-containers) / T11 (vergil-tooling) home where they
  run; docs sweep #2551 spawns same-repo siblings rather than crossing a boundary.
- **Cardinality consistency.** T3 defines it, T5 declares it (LINT/AUDIT `once`;
  TYPECHECK/TEST `per-version`), T8 aligns CI job names, T11 verifies the gates.
- **No orphaned types.** Image names (`prod-cpp-<family>:<v>`) are produced by
  T1/T2 and consumed by T6/T8; the `[cpp]` `std`/`stdlib` keys are produced by T4
  and consumed by T5.

## Evolution during execution

_Frozen at execution start; dated entries record meaningful deltas — a task
added, dropped, or rescoped, a discovered dependency — with the reasoning._
