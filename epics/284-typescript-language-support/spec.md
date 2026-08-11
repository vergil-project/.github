# TypeScript language support (Node LTS, containerized multi-version)

- **Epic:** vergil-project/.github#284
- **Status:** Design approved 2026-08-11 (brainstormed via `epic-create`, using the
  C++ epic (#207) as boilerplate; full epic convention).
- **Repos:** vergil-project/vergil-tooling (registry + schema + standards docs),
  vergil-project/vergil-containers (images), vergil-project/vergil-actions
  (reusable CI), vergil-project/docs (summary docs); epic homed in `.github`.
- **Promoted from:** direct brainstorm (no triage origin).

## 1. Summary

Add **TypeScript** as a first-class supported language in the Vergil tooling,
built on the same containerized, multi-version model already used for the
existing languages (Python, Ruby, Java, Rust, Go, C++). This epic reuses the C++
onboarding template (#207) adapted to the JS/TS toolchain. The defining choices:
**Node.js is the runtime and `tsc` is the single canonical typechecker**, so the
expensive matrix axis is the **Node major version** (v1: `node-22`, `node-24`).
TypeScript has no Clang-vs-GCC analog — there is one `tsc` — so the "two
independent diagnostic engines" become **`tsc` (structural typing) +
typescript-eslint (type-aware lint), both run once**. Warnings are turned to 11
(`strict` + curated extras, no standing-suppression rule), delivered via a
**shareable strict base tsconfig** consumers extend. Package management and the
audit are **npm** (`npm ci` / `npm audit`) — chosen for universality at zero
adoption barrier. Coverage holds the house **100%** line via **Vitest + V8
coverage**, with `/* v8 ignore */` markers as the acknowledged-gap valve.

v1 is a deliberately lean-but-real bite; everything larger is captured in a
**deferral ledger** (§9) that nothing is allowed to fall out of.

## 2. Motivation

TypeScript is the most widely used typed language on the web and in tooling, and
is the natural next language target after the extension model was proven out by
C++ — the first *new* language added since the model matured. Adding it exercises
the full "add a language" surface a second time and hardens the reusable template
(the first re-use is the real test of a template).

TypeScript is also an instructive contrast to C++ in how the stages map. In C++
the **compiler** enforces types, so `TYPECHECK` *was the compiler* and it ran
**per compiler×version** (Clang and GCC are two genuinely different engines). In
TypeScript there is exactly one canonical type engine — `tsc` — so **`TYPECHECK`
runs once**, and only `TEST` is per-Node-version (runtime behavior differs across
Node majors). This makes TypeScript *lighter* than C++ on the matrix: fewer
per-version merge gates. And because C++ already built the **per-kind cardinality
machinery** (its T3), TypeScript **inherits it for free** — it only *declares* its
cardinalities rather than building the concept. The bulk of this epic is therefore
a registry entry + config + containers + CI acceptance + docs.

## 3. Design decisions

### 3.1 Single runtime, single typechecker

**Node.js runtime, `tsc` typechecker; the expensive axis is the Node major
version.** The C++ headline was *dual compilers as two diagnostic engines*.
TypeScript has no clean analog: `tsc` is the one canonical structural type
checker, and the alternatives (swc, esbuild, babel) *strip* types rather than
check them — they are transpilers, not type engines. So the honest mapping is a
**single-compiler** model where the two independent static analyses are already
present as **`tsc`** (structural typing) and **typescript-eslint's type-aware
rules** (lint that consumes type information) — both run **once**, not as a
per-version duel. Alternative *runtimes* (Deno, Bun) are the closest philosophical
mirror of Clang+GCC, but they are runtime engines, not static type engines, and
they carry real image/gate cost for uncertain static-analysis payoff; they are
deferred to the ledger (§9 #2), exactly as C++ deferred `libc++` and extra
compilers.

### 3.2 Warnings to 11

`tsc --noEmit` with **`strict: true`** as the floor, tuned upward with curated
extras, and a **no-standing-suppression-list rule** — the house style applied to
typed code. `strict` already implies `noImplicitAny`, `strictNullChecks`,
`strictFunctionTypes`, `strictBindCallApply`, `strictPropertyInitialization`,
`noImplicitThis`, `useUnknownInCatchVariables`, and `alwaysStrict`. The candidate
curated set beyond the floor: `noUncheckedIndexedAccess`,
`exactOptionalPropertyTypes`, `noImplicitOverride`, `noImplicitReturns`,
`noFallthroughCasesInSwitch`, `noPropertyAccessFromIndexSignature`,
`noUnusedLocals`, `noUnusedParameters`. The **no-standing-suppression** rule bans
bare `// @ts-ignore` and unexplained `// @ts-expect-error` / `eslint-disable`
(mirroring C++'s no-suppression-list rule), enforced in the standards docs and by
an ESLint rule (`@typescript-eslint/ban-ts-comment`).

The **exact curated set is a standards-doc deliverable** (§6/§7), settled during
implementation — "warnings to 11" must resolve to a concrete, reviewable list.
The delivery mechanism is a **shareable strict base tsconfig** shipped by Vergil
that consumers `extends`, the direct analog to C++ enforcing a concrete
`-Wall -Wextra -Werror -Wpedantic`-plus-extras warning set.

### 3.3 Matrix model — Node versions + a small `[typescript]` block

Sorting axes by cost keeps the matrix from exploding:

| Axis | What it is | Distinct image? | v1 |
|---|---|---|---|
| **Node major version** | `node-22`, `node-24` | **Yes** (image + per-version `TEST` gate each) | 2 images |
| **Module system** | ESM vs CJS | No — `package.json` `"type"` / tsconfig | ESM only |
| **Compile target** | `target`/`lib` (`es2022`, …) | No — a tsconfig flag | pinned `es2022` |
| **Package manager** | npm / pnpm / yarn | Partly (lockfile + audit) | npm only |

Modeled as the existing `[ci].versions` image-tag list (the `node-` prefix selects
the runtime family) plus a small `[typescript]` block for the cheap axes. This
reuses today's per-version CI-gate generation
(`github_config.desired_ci_gates_ruleset`) with minimal schema surgery.

```toml
[project]
primary-language = "typescript"

[ci]
versions = ["node-24", "node-22"]   # image/gate axis (TEST per-version)

[typescript]
module = "esm"     # cjs deferred (ledger #4)
target = "es2022"  # single pinned value in v1
```

**`versions[0]` is the primary; order the list newest-first.** The CI-gate
generator keys every `once` gate (TYPECHECK/LINT/AUDIT — §3.6) to
`ci.versions[0]`, so the first entry is the image those checks run on. Listing
`node-24` first runs the once-checks on the newest Node (the "primary image" in
§4) and matches the C++ convention of ordering the primary compiler first
(`["clang-20", …]`). Whichever Node is primary must satisfy every bundled tool's
minimum version.

Container images follow the existing `prod-<lang>:<tag>` pattern extended for the
runtime family: **`prod-ts-node:<major>`** (plus `dev-` variants), built in
`vergil-containers`. The `-node` family segment leaves room for future
`prod-ts-deno` / `prod-ts-bun` families (§9 #2) without a naming change — the same
forward-compatible shape C++ used for `prod-cpp-clang` / `prod-cpp-gcc`.

### 3.4 Package manager + audit

**npm, mandated in v1.** `npm ci` drives `INSTALL` (clean, lockfile-pinned) and
`npm audit` drives `AUDIT`. Unlike C++ — where `conan audit` was genuinely better
than the alternatives and *drove* the build-tool choice — in the npm ecosystem
`npm audit` is already first-class and queries the **same** advisory data
(GitHub Advisory / npm registry) as `pnpm audit` / `yarn npm audit`. So npm
delivers the audit at **zero adoption barrier**: every TypeScript repo already has
it, no per-consumer migration. pnpm's unique benefit — strict dependency-declaration
hygiene (no phantom deps) — is real but is a *different* strictness from the
type/audit strictness this epic is about, and it carries a mandatory per-consumer
lockfile migration; it is deferred (§9 #1). No bundler/build step is mandated in
v1 — `tsc --noEmit` covers compile-correctness; emit/bundling is deferred (§9 #6).

### 3.5 Prebuilt stable toolchains only

**Node toolchains come exclusively from prebuilt stable binaries** (official Node
images / NodeSource), never built from source. The durable requirement is **two
recent LTS majors**; the exact majors are whatever is cleanly available prebuilt.
v1 pins **`node-22` (Active LTS) + `node-24` (LTS)**; **`node-20` is excluded**
(end-of-life April 2026) and **`node-26`** (GA October 2026) is a future pull-in
(§9 #5). The matrix advances to a newer major only once it is a prebuilt stable
release.

### 3.6 Per-kind check cardinality (inherited from C++)

C++ introduced the **per-kind cardinality** concept (`per-version` vs `once`) in
`languages.py` (#207 T3). TypeScript **reuses it as-is** — it declares
cardinalities, it does not build the machinery. TypeScript's shape differs from
C++ precisely because there is one `tsc`:

| Stage | Cardinality | Why |
|---|---|---|
| TYPECHECK (`tsc --noEmit`) | **once** | one canonical type engine (contrast C++, where it was per-compiler) |
| LINT (eslint + typescript-eslint + prettier) | **once** | runtime-independent static analysis |
| TEST (vitest + coverage) | **per Node version** | runtime behavior differs across Node majors |
| AUDIT (npm audit + license) | **once** | dependency graph is runtime-independent |

So only **one** stage (`TEST`) fans out per version — the CI-gate generator emits
`TYPECHECK`/`LINT`/`AUDIT` as one required check each and `TEST` per Node version.
This is data + declaration on top of the mechanism C++ already shipped.

## 4. Tool matrix

Legend: **[per-version]** = runs in every Node image; **[once]** =
runtime-agnostic, logically runs once. `CheckKind` is the existing coarse
taxonomy — format folds into `LINT`, coverage into `TEST`, dep-audit/license into
`AUDIT`.

| CheckKind | Tools | Interpreted-world analog |
|---|---|---|
| **INSTALL** | `npm ci` (deps from `package-lock.json`) | `uv sync` / `bundle install` / `cargo fetch` |
| **LINT** *[once]* | `prettier --check` (format) · `eslint` flat config with **typescript-eslint** type-aware rules | `ruff format --check` + `ruff check` / `clippy` |
| **TYPECHECK** *[once]* | **`tsc --noEmit`** with `strict` + curated extras (warnings-to-11) | `mypy` / `ty` — the direct analog, and here it is the *canonical* engine |
| **TEST** *[per-version]* | `vitest run` · coverage via **`@vitest/coverage-v8`** `--coverage` with a **100% line** threshold | `pytest --cov` / `cargo llvm-cov` (no sanitizer analog — see below) |
| **AUDIT** *[once]* | `npm audit` (dependency CVE scan, severity-thresholded) · license-metadata allowlist (best-effort in v1, e.g. `license-checker`) | `pip-audit` / `govulncheck` / `cargo-deny` + `pip-licenses` |
| **CI-level SAST** *(vergil-actions, language-agnostic)* | **CodeQL `javascript-typescript`** · Trivy · Semgrep JS/TS rules | Same jobs the other languages get |

**No sanitizer analog.** C++ *gained* runtime analysis via ASan/UBSan; TypeScript
has no memory-sanitizer equivalent. The type system + strict type-aware lint cover
statically much of what sanitizers recover for C++, so `TEST` is just
`vitest` + coverage — there is no second instrumented build.

**Runner vs framework.** The tooling mandates **Vitest** as the runner (registry
command `vitest run`), and V8 coverage ships with it. Standards docs recommend
Vitest's built-in `describe`/`it`/`expect` as the documented default.

**Image contents & where the `[once]` checks run.** Every `prod-ts-node:*` image
bundles the full toolset (`typescript`, `eslint`, `typescript-eslint`,
`prettier`, `vitest`, `@vitest/coverage-v8`, license tooling), so any image can
run any check. The `[once]` LINT/TYPECHECK/AUDIT stages execute on the **primary
image** (highest Node major); cardinality is enforced by §3.6.

**Honest caveats (verify during implementation, do not assume turnkey):**

1. **`npm audit` must be proven to *detect*, not just to run.** A dependency audit
   that passes because there is nothing to find is a silent no-op. Validation
   (§10, T10) must pin a dependency with a **known CVE**, confirm the audit
   **fails**, then confirm the clean version **passes**. Two npm-specific tuning
   points settle in implementation: the **severity threshold**
   (`npm audit --audit-level=<level>`) and the **prod-vs-dev scope**
   (`--omit=dev` to avoid gating on dev-only advisories). Confidence ~90% on the
   capability; the exact gate tuning is verified, not asserted. **Contingency:** if
   `npm audit`'s signal/noise is unworkable, fall back to **OSV-Scanner** over
   `package-lock.json`.
2. **V8 coverage precision under source maps.** The 100% line gate depends on
   `@vitest/coverage-v8` mapping V8's byte-range coverage back through TS source
   maps accurately. Confidence ~85%; if mapping is imprecise at the 100% bar, the
   contingency is the **istanbul provider** (`@vitest/coverage-istanbul`), which
   instruments source directly. Decided in implementation against the real gate.
3. **License gating is best-effort in v1** (npm license metadata is inconsistent);
   hardened license gating is on the ledger (§9 #7).
4. **ESM/CJS interop.** ESM-first (§3.3) can surface CJS-only dependencies; Node's
   ESM↔CJS interop covers the common cases, and a CJS sweep is deferred (§9 #4).

## 5. Coverage philosophy — hold 100% per Node version, acknowledge gaps explicitly

Coverage holds the house **100%** line via `@vitest/coverage-v8`, using
`/* v8 ignore next */` / `/* v8 ignore start */` / `/* v8 ignore stop */` markers
as the acknowledged-gap valve. The gate is a **forcing function for
explicitness**, not a claim that every line was exercised:

> **100% means "you tested everything you could, and positively acknowledged the
> things you couldn't."** Setting the bar at 90% moves the ambiguity into the
> untracked 10% — nobody can tell whether that gap is legitimately untestable or
> merely skipped. Holding 100% + explicit markers forces every gap to be a
> deliberate, reviewable decision instead of silent forgotten code.

**The gate binds per Node version.** The concern is untested code slipping
through; a per-version gate is the strictest guard. This is affordable because
TypeScript support targets **greenfield Vergil components with a limited set of
use cases** — not a grandfathered legacy import. Exclusion markers are reserved
for **genuinely-unreachable runtime branches** — defensive `default:` on
exhaustive `switch`, code after an `assertNever`, impossible-condition guards.
The TypeScript standards docs (§7) document this exclusion discipline — what a
legitimate exclusion looks like versus abuse (e.g. ignoring a whole file) — as the
written norm for human and agent authors.

## 6. The plug-in surface (in `vergil-tooling`)

A language plugs in at ~12 points; TypeScript touches (and, unlike C++, **builds
no new registry capability** — it reuses the cardinality concept C++ shipped):

1. `lib/languages.py` `_REGISTRY` — new `Language` entry (INSTALL/LINT/TYPECHECK/
   TEST/AUDIT commands + `EcosystemInfo`), **declaring per-kind cardinalities**
   (§3.6) with the existing `per-version`/`once` concept.
2. `lib/languages.py` license-allowlist constants — a TypeScript allowlist if
   license checking is wired.
3. `lib/config.py` `_ENUMS["primary-language"]` — add `"typescript"` (hard
   allowlist; blocks the whole feature otherwise).
4. `lib/config.py` — parse/validate the new `[typescript]` block (`module`,
   `target`).
5. `lib/container.py` `_DEFAULT_VERSIONS` / `_DEFAULT_TEST_COMMANDS` — TS entries
   (`node-22`, `node-24`).
6. `lib/container.py` `detect_language` — TS marker files (`tsconfig.json`,
   `package.json` with a `typescript` devDependency).
7. `lib/container.py` image resolution — parse the runtime family from the
   `node-` version prefix into `prod-ts-node:<major>`.
8. `lib/github_config.py` `_CODEQL_SUPPORTED_LANGUAGES` — add **`"typescript"`**
   (the *primary-language* key, **not** `"javascript-typescript"`). This set is
   keyed by `project.primary_language` and gates whether the CodeQL check is
   emitted (`if lang in _CODEQL_SUPPORTED_LANGUAGES`, github_config.py:308);
   putting the CodeQL identifier here would make the membership test fail and
   **silently drop CodeQL for every TS repo**. (`repo_init._CODEQL_LANGUAGES`
   already lists `typescript`, so the validation side needs no change.)
8a. **New: primary-language → CodeQL-language mapping (`typescript →
    javascript-typescript`).** For all six existing languages the primary-language
    name *is* the CodeQL identifier (even `cpp` passed straight through
    `render_ci_workflow`'s `language: {primary_language}`), so **no mapping layer
    exists today**. TypeScript is the **first** language where they diverge —
    CodeQL's analysis language is `javascript-typescript`, not bare `typescript`
    — so this epic must introduce the map wherever the CodeQL Action receives its
    `languages:` input (the reusable CodeQL workflow in `vergil-actions`, or the
    rendered `ci.yml`). This is genuinely new work, not a C++ copy-paste, and the
    task must also confirm whether `cpp` currently relies on a CodeQL alias that
    should route through the same map.
8b. `lib/github_config.py` `desired_ci_gates_ruleset` — reuse the run-once/
    per-version emission C++ added (§3.6) so TYPECHECK/LINT/AUDIT are one required
    check each and TEST is per version.
9. `lib/github_config.py` `_LANGUAGE_ACTION_PATTERNS` — allowed action patterns
   (if needed).
10. `lib/repo_init.py` `_container_suffix` / `_container_tag` — TS suffix/tag
    derivation (runtime-family aware).
11. Bundled configs (`src/vergil_tooling/configs/`) — packaged **shareable strict
    base tsconfig**, ESLint flat config, and Prettier config.
12. Standards docs — `docs/site/docs/standards/development/typescript/`.

**External to `vergil-tooling` but required for a working feature:**

- **`vergil-containers`** — `prod-ts-node:22` / `prod-ts-node:24` (and `dev-`
  variants) carrying Node + npm and the toolset (`typescript`, `eslint`,
  `typescript-eslint`, `prettier`, `vitest`, `@vitest/coverage-v8`, license tool).
- **`vergil-actions`** — reusable `ci-*.yml` workflows accepting
  `language: typescript` and the `node-` container suffix.

## 7. Five-repo spread & task placement

Each task lives in the repo where its closing PR lands (placement law):

- **`.github`** — this epic; `spec.md` / `plan.md` (this task, #285) and
  `retrospective.md` (#287).
- **`vergil-tooling`** — the §6 registry/schema/config/CodeQL wiring; the new
  `typescript/` standards docs + shareable base tsconfig; documentation-review
  sweep (#2738).
- **`vergil-containers`** — the `prod-ts-node:*` / `dev-ts-node:*` images.
- **`vergil-actions`** — the reusable `ci-*.yml` TypeScript support.
- **`docs`** — higher-level summary docs (per the documentation-review sweep).

Implementation tasks and the **operational** tasks (image **deployment** to GHCR;
**cold-rebuild validation** of the full pipeline) are filed at plan time (§10) so
their `Blocked-by` refs point at real impl-task numbers. Expected shape: **impl
(images/registry/actions) → deploy (publish images) → validate (cold rebuild +
full TS pipeline against a sample repo).**

## 8. Non-goals for v1

- Any runtime other than Node.js (Deno / Bun deferred — ledger #2).
- Any package manager other than npm (pnpm / yarn detection deferred — ledger #1).
- CommonJS module output and plain (untyped) JavaScript projects (ledger #4, #8).
- A bundler / emit / published-artifact step (ledger #6) — `tsc --noEmit` covers
  compile-correctness in v1.
- A full Cartesian product-matrix config engine (the `[ci].versions` list +
  `[typescript]` block suffice for v1).
- Cross-architecture builds and monorepo/workspaces (ledger #5, #9).

## 9. Deferral ledger (nothing lost)

Adjudicated at the **follow-on-brainstorm bookend** (#286): each item is either
pulled into this epic before close or pushed to a future epic — never dropped.

1. **pnpm / yarn** lockfile detection (package-manager axis; v1 is npm-only).
2. **Alternative runtimes — Deno, Bun** (the dual-engine mirror of Clang+GCC set
   aside; a future `prod-ts-deno` / `prod-ts-bun` family).
3. `tsc` **release-channel** matrix (stable + `next`/beta) to catch type-checker
   regressions early.
4. **CommonJS (CJS)** module sweep (v1 is ESM-only).
5. Additional Node versions (**`node-26`** GA Oct 2026, future majors).
6. **Bundling / emit** + published-artifact validation (tsup / esbuild / rollup).
7. Hardened **license gating** (v1 is best-effort).
8. **Plain-JavaScript** (untyped) project support.
9. **Monorepo / npm-workspaces** (multiple `package.json`).

## 10. Bookends (this epic)

- **Documentation** (#285): this spec + plan.
- **Documentation-review sweep** (vergil-tooling#2738): multi-repo sweep,
  especially the versioned site docs; spawns per-repo doc siblings; runs **before**
  the retrospective.
- **Follow-on brainstorm** (#286): adjudicate the §9 ledger (seeded because a known
  enabling chain — runtime diversity and the package-manager axis — exists at
  creation).
- **Retrospective** (#287, terminal): its docs PR closes the epic.
- **Operational** (filed at plan time): image **deployment** to GHCR, and a
  **cold-rebuild validation** (this is an infra/provisioning-shaped epic). The
  validation's acceptance explicitly includes **proving the dependency audit
  detects a known CVE** (§4 caveat 1), and **proving the CodeQL job actually runs
  and reports for a TS repo** (§6 point 8/8a) — not merely that the gate names
  exist or the pipeline runs green.

## 11. Open questions (resolved in plan / implementation)

- Exact Node LTS majors available as **prebuilt stable binaries** in the base
  images (confirms the `node-22` / `node-24` pins vs. any fallback), bounded by the
  §3.5 prebuilt-only rule.
- `npm audit` gate tuning — severity threshold and prod-vs-dev scope (§4 caveat 1)
  and the OSV-Scanner contingency.
- V8 vs istanbul coverage provider under TS source maps at the 100% bar (§4
  caveat 2).
- The exact "curated extras" `tsconfig` + ESLint rule set (§3.2), finalized in the
  standards docs and the shareable base tsconfig.

**Resolved during brainstorming/pushback** (recorded here so they are not
reopened): the matrix is **single-runtime / single-typechecker** with the Node
major as the only expensive axis (§3.1); **npm** is the mandated package manager
and audit at zero adoption barrier (§3.4); **TYPECHECK runs once** (one `tsc`),
only **TEST** is per-version (§3.6); the `[typescript]` `module`/`target` keys are
**held single** in v1 (ESM, `es2022`); toolchains are **prebuilt-only** (§3.5);
runtime diversity (Deno/Bun) is **deferred**, not in v1 (§3.1, §9 #2).
