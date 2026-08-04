# C++ language support (GCC + Clang, containerized multi-version)

- **Epic:** vergil-project/.github#207
- **Status:** Design approved 2026-08-04 (brainstormed directly via `epic-create`;
  full epic convention).
- **Repos:** vergil-project/vergil-tooling (registry + schema + standards docs),
  vergil-project/vergil-containers (images), vergil-project/vergil-actions
  (reusable CI), vergil-project/docs (summary docs); epic homed in `.github`.
- **Promoted from:** direct brainstorm (no triage origin).

## 1. Summary

Add **C++** as a first-class supported language in the Vergil tooling, built from
the ground up on the same containerized, multi-version model already used for the
existing five languages (Python, Ruby, Java, Rust, Go). The defining choice is
**two co-equal open-source compilers — Clang/LLVM primary, GCC secondary — both
run as first-class CI gates**, so every change is checked by two independent
diagnostic engines. Warnings are turned to 11 (`-Wall -Wextra -Werror` and up,
no standing suppression list). Builds are CMake; dependencies are Conan 2 (chosen
for its `conan audit` CVE story). Coverage holds the house 100% line via `gcovr`,
with explicit exclusion markers as the acknowledged-gap valve.

v1 is a deliberately lean-but-real bite; everything larger is captured in a
**deferral ledger** (§9) that nothing is allowed to fall out of.

## 2. Motivation

The tooling supports five languages but only Python is used in practice; the
other four are proven-out extension points. C++ is the first *new* language added
since the extension model matured, and it is wanted as a genuine target (native
components, and a stepping stone to Swift — an LLVM-family language whose
toolchain investment overlaps C++). Adding it exercises the full "add a language"
surface end-to-end and produces a reusable template for future compiled
languages.

C++ also reshapes the tooling model in one important way. In interpreted
languages the external tools exist to **recover guarantees the runtime does not
give** — `mypy`/`ty` exist because Python does not enforce types; `rubocop`/`steep`
backfill Ruby. In C++ the **compiler already enforces types, and with warnings
cranked up it enforces much of what a separate linter would**. So the `TYPECHECK`
stage *is the compiler*, the `LINT` stage shrinks to what the compiler still does
not do (formatting, higher-level patterns), and C++ *gains* a capability the
interpreted languages have no analog for: **runtime analysis via sanitizers**.

## 3. Design decisions

### 3.1 Dual compilers, both first-class

**Clang/LLVM primary, GCC secondary.** "GCC only" was the original framing but was
reconsidered during brainstorming: the intent behind it was *open-source compilers
only* (no proprietary vendor compilers), and by that rule Clang/LLVM — the
co-equal other half of the open-source C++ world, and the default compiler on
macOS/FreeBSD/Android — clearly belongs. The **payoff is two independent
diagnostic engines**: compiling the same code under both GCC and Clang catches
undefined behavior, non-portable constructs, and standard-conformance mistakes
that either alone would miss. This is the modern, consolidated form of the
"build clean across every vendor compiler" portability discipline — except both
compilers are now free, open-source, and containerized. Clang-first also sets up
Swift cleanly later.

### 3.2 Warnings to 11

`-Wall -Wextra -Werror -Wpedantic` as the floor, tuned upward from there, with a
**no-standing-suppression-list rule** — the house style applied to compiled code.
Each compiler×version image enforces its own warning set, so the two-diagnostics
win lands in the `TYPECHECK` stage.

The **exact "curated extras" set is a standards-doc deliverable** (§7), settled
during implementation — "warnings to 11" must resolve to a concrete, reviewable
list, not a vibe. Candidate baseline beyond the floor: `-Wshadow -Wconversion
-Wsign-conversion -Wcast-qual -Wold-style-cast -Wnon-virtual-dtor` (finalized in
the standards docs, since the two compilers spell some warnings differently).

### 3.3 Matrix model — profiles + a small `[cpp]` block

Not every axis needs its own container image. Sorting axes by cost is what keeps
the matrix from exploding:

| Axis | What it is | Distinct image? | v1 |
|---|---|---|---|
| **Compiler family × version** | `clang-19/20`, `gcc-14/15` | **Yes** (image + CI gate each) | 4 images |
| **C++ standard** | `-std=c++17/20/23` | No — a build flag | pinned `c++20` |
| **Standard library** | `libstdc++` (GNU) vs `libc++` (LLVM) | Partly (libc++ install) | `libstdc++` only |
| **Dual libstdc++ ABI** | `_GLIBCXX_USE_CXX11_ABI=0/1` | No — a build flag | default `=1` |

Modeled as **approach (A) plus a small `[cpp]` block**: compiler×version rides the
existing `[ci].versions` image-tag list (the `clang-`/`gcc-` prefix selects the
family), and the cheap axes are `[cpp]` keys swept inside a job. This reuses
today's per-version CI-gate generation (`github_config.desired_ci_gates_ruleset`)
with minimal schema surgery rather than building a full Cartesian product engine.

```toml
[project]
primary-language = "cpp"

[ci]
versions = ["clang-20", "clang-19", "gcc-15", "gcc-14"]  # image/gate axis

[cpp]
std = "c++20"          # in-build flag sweep (single value in v1)
stdlib = "libstdc++"   # libc++ deferred (ledger #2)
```

Container images follow the existing `prod-<lang>:<tag>` pattern extended for the
compiler family: `prod-cpp-clang:<v>` / `prod-cpp-gcc:<v>` (plus `dev-` variants),
built in `vergil-containers`.

### 3.4 Build system + dependency manager

**CMake + Conan 2.** CMake is the de-facto C++ build system and the integration
point for both package managers. Conan 2 is chosen over vcpkg specifically for its
**built-in `conan audit`** (dependency CVE scan) and per-package license metadata —
the closest analog C++ has to `pip-audit` / `cargo-deny`. `CMAKE_EXPORT_COMPILE_COMMANDS=ON`
produces the `compile_commands.json` that feeds `clang-tidy`.

### 3.5 Prebuilt stable toolchains only

**Compiler toolchains come exclusively from prebuilt stable binary packages —
never built from source.** Building GCC/Clang from source is non-trivial, slow,
and brittle in an image build, and there is no reason to own that cost. The
durable requirement is **two recent majors per family**; the *exact* majors are
whatever is cleanly available prebuilt (upstream images / official apt channels).
The matrix advances to a newer major **only once that major is available as a
prebuilt stable binary**. This is a hard boundary condition on `vergil-containers`,
not a preference: if `gcc-15` / `clang-20` are not cleanly prebuilt, the fallback
is `gcc-13/14` + `clang-18/19` — still two majors each, still the full
two-compiler win.

### 3.6 Per-kind check cardinality

The stages do not all have the same cardinality across the matrix, so the
registry gains a **per-kind cardinality** concept (`per-version` vs `once`):

| Stage | Cardinality |
|---|---|
| TYPECHECK (build + warnings-to-11) | **per compiler×version** — the two-diagnostics engine |
| TEST (test + coverage + sanitizers) | **per compiler×version** |
| LINT (clang-format / clang-tidy / cppcheck) | **once** (runs on the primary Clang image) |
| AUDIT (conan audit + license) | **once** |

The CI-gate generator (`github_config.desired_ci_gates_ruleset`) is taught to emit
**run-once kinds a single time** and per-version kinds per version — so LINT/AUDIT
become one required check each rather than N redundant, merge-wedging checks in
branch protection. This is a small, contained change in `languages.py` (kind
metadata) + `github_config.py` (emission), and it generalizes to any future
compiled language with the same once-vs-matrix shape (e.g. Swift).

## 4. Tool matrix

Legend: **[per-image]** = compiler-specific, runs in every image (the
two-diagnostics engine); **[once]** = compiler-agnostic, logically runs once.
`CheckKind` is the existing coarse taxonomy — there is no separate FORMAT /
COVERAGE / SECURITY kind (format folds into `LINT`, coverage into `TEST`,
dep-audit/license into `AUDIT`).

| CheckKind | Tools | Interpreted-world analog |
|---|---|---|
| **INSTALL** | `conan install --build=missing` (deps + lockfile) → `cmake` configure with `CMAKE_EXPORT_COMPILE_COMMANDS=ON` | `uv sync` / `bundle install` / `cargo fetch` |
| **LINT** *[once]* | `clang-format --dry-run --Werror` (format) · `clang-tidy` (static-analysis lint via `compile_commands.json`) · `cppcheck --enable=all --error-exitcode=1` | `ruff format --check` + `ruff check` / `clippy` |
| **TYPECHECK** *[per-image]* | **The compiler** — warnings-to-11 build (`-Wall -Wextra -Werror -Wpedantic` + curated extras) under each compiler×version | `mypy`/`ty`, but native |
| **TEST** *[per-image]* | `ctest --output-on-failure` (framework-agnostic runner) · coverage via `gcovr --fail-under-line 100` (uses `gcov` on GCC, `llvm-cov gcov` on Clang) · **sanitizer run**: second build with `-fsanitize=address,undefined` (ASan+UBSan) | `pytest --cov` / `cargo llvm-cov` — **plus** sanitizers (no interpreted analog) |
| **AUDIT** *[once]* | `conan audit` (dependency CVE scan) · Conan license-metadata allowlist (best-effort in v1) | `pip-audit`/`govulncheck`/`cargo-deny` + `pip-licenses`/`go-licenses` |
| **CI-level SAST** *(vergil-actions, language-agnostic)* | **CodeQL `cpp`** · Trivy · Semgrep C/C++ rules | Same jobs the other languages get |

**Runner vs framework.** The tooling mandates **CTest** as the *runner* (registry
command is `ctest`), so a repo may use GoogleTest, Catch2, or doctest and still
satisfy the gate. Standards docs recommend **GoogleTest** as the documented
default.

**Image contents & where the `[once]` checks run.** Every `prod-cpp-*` image —
**both** families — bundles the full analysis toolset (`clang-format`,
`clang-tidy`, `cppcheck`, `gcovr`), so any image *can* run any check and there are
no per-image tool gaps. The `[once]` LINT and AUDIT stages execute on the
**primary (Clang) image**; `clang-tidy` reads a GCC-generated
`compile_commands.json` fine (it parses source, it does not need GCC's binary).
Cardinality is enforced by §3.6.

**CI cost (accepted).** Per image, TEST implies **two distinct compiles** — a
coverage-instrumented build and a separate `-fsanitize=address,undefined` build
(the instrumentations do not share a build) — on top of TYPECHECK's warnings
build. Across 4 images that is up to ~12 compiles per run, plus LINT/AUDIT once.
This is correct and accepted, not a surprise; **build caching (`ccache`) and the
Conan package cache** are the implementation levers to keep wall-clock sane.

**Honest caveats (verify during implementation, do not assume turnkey):**

1. **`conan audit` provider/token.** Conan 2's `conan audit` (added in the 2.x
   line, ~early 2025) scans the dependency graph against a CVE database, but the
   provider setup **may require a free API token**. Confidence ~80% on the
   capability; the auth/provider details are verified in implementation, not
   asserted here. **Contingency:** if it needs paid infra, fall back to
   **OSV-Scanner** over `conan.lock`.
2. **License gating is weaker in C++** than pip/go — Conan carries per-package
   license metadata but enforcement is not as turnkey as `go-licenses`. v1 does a
   **best-effort** check; hardened license gating is on the ledger (§9 #7).

## 5. Coverage philosophy — hold 100% per compiler, acknowledge gaps explicitly

Coverage holds the house **100%** line via `gcovr`, using `// GCOVR_EXCL_LINE` /
`_START` / `_STOP` markers as the acknowledged-gap valve. The gate is a
**forcing function for explicitness**, not a claim that every line was exercised:

> **100% means "you tested everything you could, and positively acknowledged the
> things you couldn't."** Setting the bar at 90% moves the ambiguity into the
> untracked 10% — nobody can tell whether that gap is legitimately untestable or
> merely skipped. Holding 100% + explicit markers forces every gap to be a
> deliberate, reviewable decision instead of silent forgotten code.

**The gate binds per compiler (per image), not on a single merged report.** The
concern is untested code slipping through, and a per-compiler gate is the
strictest guard against it. This is affordable because C++ support targets
**greenfield Vergil components with a limited set of use cases** — not a
grandfathered legacy import that carries a pile of compiler-specific baggage.

Two consequences worth stating precisely:

- **Clean preprocessor splits mostly self-handle.** For a pure
  `#if defined(__GNUC__) && !defined(__clang__) … #else … #endif` split, the
  compiled-out branch is **not instrumented** under the other compiler — it never
  appears as an uncovered line — so each compiler naturally measures only its own
  branch and **no exclusion marker is needed**. *(Confidence ~90% on the gcov /
  `llvm-cov gcov` behavior; confirmed in implementation.)*
- **Exclusion markers are reserved for genuinely-unreachable runtime branches** —
  defensive `else` on impossible conditions, exhaustive-`switch` `default:`,
  code after `abort()`/`std::unreachable()`. These are not compiler-specific, so
  they are marked once and excluded in every compiler's report.

Compiler-specific `#ifdef` blocks are therefore treated as a **code smell to
minimize**, not a routine pattern — if one is truly justified (performance,
behavior), doubling its cost (the block *plus* any needed markers) is the
intended friction. The C++ standards docs (§7) document this exclusion discipline —
what a legitimate exclusion looks like versus abuse (e.g. exempting a whole
file) — as the written norm for human and agent authors.

## 6. The plug-in surface (in `vergil-tooling`)

From the architecture map, a language plugs in at ~12 points; C++ touches:

1. `lib/languages.py` `_REGISTRY` — new `Language` entry (INSTALL/LINT/TYPECHECK/
   TEST/AUDIT commands + `EcosystemInfo`), **plus a new per-kind cardinality
   concept** (`per-version` / `once`) on the check-kind structure (§3.6). This is
   the one genuinely new registry capability, not just data entry.
2. `lib/languages.py` license-allowlist constants — a C++ allowlist if license
   checking is wired.
3. `lib/config.py` `_ENUMS["primary-language"]` — add `"cpp"` (hard allowlist;
   blocks the whole feature otherwise).
4. `lib/config.py` — parse/validate the new `[cpp]` block (`std`, `stdlib`).
5. `lib/container.py` `_DEFAULT_VERSIONS` / `_DEFAULT_TEST_COMMANDS` — C++ entries.
6. `lib/container.py` `detect_language` — C++ marker files (`CMakeLists.txt`,
   `conanfile.py`/`conanfile.txt`).
7. `lib/container.py` image resolution — parse compiler family from the
   `clang-`/`gcc-` version prefix into `prod-cpp-<family>:<v>`.
8. `lib/github_config.py` `_CODEQL_SUPPORTED_LANGUAGES` — add `"cpp"` (this also
   **fixes the existing discrepancy**: `repo_init.py` `_CODEQL_LANGUAGES` already
   lists `cpp` but `github_config.py` does not).
8b. `lib/github_config.py` `desired_ci_gates_ruleset` — emit **run-once** check
    kinds a single time and per-version kinds per version (§3.6), so LINT/AUDIT do
    not mint N redundant required status checks.
9. `lib/github_config.py` `_LANGUAGE_ACTION_PATTERNS` — allowed action patterns
   (if needed).
10. `lib/repo_init.py` `_container_suffix` / `_container_tag` — C++ suffix/tag
    derivation (compiler-family aware).
11. Bundled configs (`src/vergil_tooling/configs/`) — packaged `clang-format`,
    `clang-tidy`, `cppcheck`, `gcovr` configs as needed.
12. Standards docs — `docs/site/docs/standards/development/cpp/`.

**External to `vergil-tooling` but required for a working feature:**

- **`vergil-containers`** — `prod-cpp-clang` / `prod-cpp-gcc` (and `dev-` variants)
  images carrying the toolchains (compiler, `clang-format`, `clang-tidy`,
  `cppcheck`, `gcovr`, CMake, Conan 2, sanitizer runtimes).
- **`vergil-actions`** — reusable `ci-*.yml` workflows accepting `language: cpp`
  and the compiler-family container suffix.

## 7. Five-repo spread & task placement

Each task lives in the repo where its closing PR lands (placement law):

- **`.github`** — this epic; `spec.md` / `plan.md` (this task, #208) and
  `retrospective.md` (#210).
- **`vergil-tooling`** — the §6 registry/schema/config/CodeQL wiring; the new
  `cpp/` standards docs; documentation-review sweep (#2551).
- **`vergil-containers`** — the `prod-cpp-*` / `dev-cpp-*` images.
- **`vergil-actions`** — the reusable `ci-*.yml` C++ support.
- **`docs`** — higher-level summary docs (per the documentation-review sweep).

Implementation tasks and the **operational** tasks (image **deployment** to GHCR;
**cold-rebuild validation** of the full pipeline) are filed at plan time (§10)
so their `Blocked-by` refs point at real impl-task numbers. Expected shape:
**impl (images/registry/actions) → deploy (publish images) → validate (cold
rebuild + full C++ pipeline against a sample repo).**

## 8. Non-goals for v1

- Any compiler other than GCC and Clang (no proprietary/vendor compilers — that
  was the original intent — and no non-LLVM/non-GCC open-source compilers).
- The `libc++` standard library, `c++23`, and the dual-ABI sweep (ledger).
- A full Cartesian product-matrix config engine (profiles suffice for v1).
- Cross-architecture builds (single-arch containers in v1).

## 9. Deferral ledger (nothing lost)

Adjudicated at the **follow-on-brainstorm bookend** (#209): each item is either
pulled into this epic before close or pushed to a future epic — never dropped.

1. `c++23` standard sweep (v1 pins `c++20`).
2. `libc++` standard-library axis (v1 is `libstdc++`-only).
3. Dual libstdc++ ABI sweep `_GLIBCXX_USE_CXX11_ABI=0/1` (v1 defaults `=1`).
4. Additional compiler versions — GCC 12/13, Clang 17/18, and future majors.
5. Cross-architecture ABIs (aarch64, etc.).
6. **Swift** as a future LLVM-family language (separate epic; C++ toolchain work
   is the enabler).
7. Hardened license gating (v1 is best-effort).
8. TSan / MSan sanitizers (v1 ships ASan + UBSan).

## 10. Bookends (this epic)

- **Documentation** (#208): this spec + plan.
- **Documentation-review sweep** (vergil-tooling#2551): multi-repo sweep,
  especially the versioned site docs; spawns per-repo doc siblings; runs **before**
  the retrospective.
- **Follow-on brainstorm** (#209): adjudicate the §9 ledger (seeded because a
  known enabling chain — Swift, and the deferred axes — exists at creation).
- **Retrospective** (#210, terminal): its docs PR closes the epic.
- **Operational** (filed at plan time): image **deployment** to GHCR, and a
  **cold-rebuild validation** (this is an infra/provisioning-shaped epic).

## 11. Open questions (resolved in plan / implementation)

- Exact GCC/Clang majors available as **prebuilt stable binaries** in the base
  images (drives the concrete `clang-19/20`, `gcc-14/15` pins vs. the
  `clang-18/19` + `gcc-13/14` fallback), bounded by the §3.5 prebuilt-only rule.
- `conan audit` provider/auth (see §4 caveat) and the OSV-Scanner contingency.
- The exact "curated extras" warning set per compiler (§3.2), finalized in the
  standards docs.

**Resolved during brainstorming/pushback** (recorded here so they are not
reopened): coverage binds **per compiler** (§5); the `[once]` LINT/AUDIT checks
run **once on the primary Clang image**, with the full toolset bundled in every
image (§4, §3.6); the `[cpp]` `std`/`stdlib` keys are **held single** in v1 (one
`std`, one `stdlib`); toolchains are **prebuilt-only** (§3.5).
