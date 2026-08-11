# TypeScript language support — Implementation Plan

> **For agentic workers:** tasks are GitHub issues under epic
> vergil-project/.github#284, each 1:1 with a finalizing PR in the repo named.
> Steps use checkbox (`- [ ]`) syntax. Drive with `epic-implement`; run each code
> task via `issue-implement`, each operational task via `issue-deploy` /
> `issue-validate`. Task numbers below are plan-local labels (T1…T10); the GitHub
> issue numbers are assigned when the tasks are filed (see the epic).

**Goal:** add **TypeScript** as a first-class supported language — Node.js runtime
with `tsc` as the single canonical typechecker, on the existing containerized
multi-version model — built across five repos, reusing the C++ epic (#207) as the
onboarding template.

**Architecture:** container images in `vergil-containers` carry Node + npm and the
shared analysis toolset; `vergil-tooling` gains the registry entry, the
`[typescript]` schema, container/CodeQL/repo-init wiring, and a shareable strict
base tsconfig; `vergil-actions` accepts `language: typescript` in the reusable CI
workflows; standards docs and an end-to-end cold-rebuild validation close it out.
The **Node major version** is the image/CI-gate axis; module system and compile
target are cheap in-config flags. Unlike C++, the **per-kind cardinality machinery
already exists** (C++ shipped it) — TypeScript only *declares* its cardinalities.

**Tech stack:** Python (`vergil_tooling` CLIs + `pytest`); Docker (dev/prod
images); Node.js LTS + npm; `typescript` (`tsc`), ESLint + typescript-eslint,
Prettier, Vitest + `@vitest/coverage-v8`, `npm audit`, license tooling; GitHub
Actions reusable workflows; MkDocs site docs.

## Global constraints

- **Validation:** `vrg-container-run -- vrg-validate` is the only validation
  command (in `vergil-tooling` it expands to `uv run vrg-validate` via the
  `[validation]` override). Git via `vrg-git` / `vrg-commit`; PRs via
  `vrg-submit-pr` (human-gated — agents stop at `vrg-pr-workflow report-ready`).
- **Placement law:** each task lives in the repo where its PR lands; a PR only
  `Closes` a same-repo issue. Cross-repo links are `Ref`.
- **Single runtime, single typechecker:** Node.js runtime, `tsc` typechecker. The
  two static analyses are `tsc` (structural typing) + typescript-eslint (type-aware
  lint), **both run once**. Deno/Bun and `tsc` release channels are deferred.
- **Warnings to 11:** `tsc --noEmit` with `strict` + curated extras
  (`noUncheckedIndexedAccess`, `exactOptionalPropertyTypes`, `noImplicitOverride`,
  `noImplicitReturns`, `noFallthroughCasesInSwitch`, `noPropertyAccessFromIndexSignature`,
  `noUnusedLocals`, `noUnusedParameters`); no standing-suppression rule (no bare
  `@ts-ignore` / unexplained `@ts-expect-error`). Delivered via a **shareable
  strict base tsconfig** consumers `extends`.
- **Package manager:** **npm** (`npm ci` / `npm audit`). pnpm/yarn deferred.
- **Coverage:** 100% **per Node version** via `@vitest/coverage-v8`;
  `/* v8 ignore */` markers reserved for genuinely-unreachable runtime branches.
- **Prebuilt stable toolchains only** — official Node binaries, never built from
  source. Two recent LTS majors; v1 pins **`node-24` (primary) + `node-22`**;
  `node-20` excluded (EOL Apr 2026), `node-26` (Oct 2026) deferred.
- **Per-kind cardinality (reused, not built):** TYPECHECK, LINT, AUDIT run
  **once** (on the primary image); only TEST runs **per Node version**.
- **`versions[0]` is primary; order newest-first:** `versions = ["node-24", "node-22"]`.
  The once-gates key to `ci.versions[0]`.
- **v1 pins:** `[typescript]` `module = "esm"`, `target = "es2022"` (single
  values). Everything else is on the spec §9 deferral ledger.

---

### T1 — Common TS image base + primary Node image (vergil-containers)

**Files:**
- Create: `docker/typescript/Dockerfile.base` (shared analysis-tool layer)
- Create: `docker/typescript/Dockerfile.node`
- Modify: `docker/build.sh` (register the new image targets)

Foundational. Establishes the shared layer every TS image inherits, then the
primary Node image on top. **Includes the toolchain-availability discovery** that
confirms the concrete majors.

- [ ] **Discovery (prebuilt-only).** Confirm `node-24` and `node-22` are available
  as prebuilt stable official Node binaries for the base OS; pin them. Record the
  decision in the image dir (short `README`/comment). If a target major is not
  cleanly prebuilt, fall back per the constraint and note it.
- [ ] **Shared base layer.** Install the runtime-agnostic toolset every TS image
  carries: `typescript` (`tsc`), `eslint` + `typescript-eslint`, `prettier`,
  `vitest` + `@vitest/coverage-v8`, and license tooling (e.g. `license-checker`).
  Pin versions. npm ships with Node.
- [ ] **Primary Node image.** `prod-ts-node:24` and `dev-ts-node:24`, with the
  Node major on `PATH` and env set so `vrg-container-run` and npm/`tsc` resolve
  the right Node.
- [ ] **Smoke test.** A trivial `package.json` + `tsconfig.json` project installs
  (`npm ci`), typechecks (`tsc --noEmit`), and runs one `vitest` test under the
  image (build-script check).
- [ ] **Validate + commit** (`feat(ts): node dev/prod image + shared TS base`).

**Deliverable:** buildable primary Node image carrying the full toolset.
**Blocks:** T2, T7, T9.

### T2 — Second Node image (vergil-containers)

**Files:**
- Create: `docker/typescript/Dockerfile.node` reuse for the second major (or a
  build-arg on T1's Dockerfile)
- Modify: `docker/build.sh`

- [ ] **Second image.** `prod-ts-node:22` / `dev-ts-node:22` on the T1 shared
  base, Node 22 on `PATH`.
- [ ] **Smoke test** under the Node-22 image; confirm the shared analysis tools
  (`tsc`, `eslint`, `prettier`, `vitest`) are present and runnable here too.
- [ ] **Validate + commit** (`feat(ts): node-22 dev/prod images on shared TS base`).

**Deliverable:** buildable second Node image.
**Depends on:** T1. **Blocks:** T7, T9.

### T3 — `[typescript]` config schema + language enum (vergil-tooling)

**Files:**
- Modify: `src/vergil_tooling/lib/config.py:30` (`_ENUMS["primary-language"]`),
  the `_LANG_BLOCK_FIELDS`-style mapping, and a `_parse_typescript_config` +
  `TypeScriptConfig` (mirror the existing `_parse_cpp_config` / `CppConfig`)
- Test: `tests/lib/test_config.py`

- [ ] **Enum.** Add `"typescript"` to `_ENUMS["primary-language"]`. Test that
  `primary-language = "typescript"` is accepted and an unknown value still warns.
- [ ] **`[typescript]` block.** Parse and validate `module` (allowed: `esm`;
  `cjs` deferred) and `target` (e.g. `es2022`) as single string values, with
  defaults when the block is absent — following the `[cpp]` `std`/`stdlib`
  precedent exactly. Test valid values, defaults, and rejection of nonsense.
- [ ] **Validate + commit** (`feat(config): add typescript primary-language + [typescript] block`).

**Deliverable:** `vergil.toml` accepts `primary-language = "typescript"` and a
`[typescript]` block.
**Blocks:** T4, T5.

### T4 — TypeScript language registry entry (vergil-tooling)

**Files:**
- Modify: `src/vergil_tooling/lib/languages.py:232` (`_REGISTRY`) and license
  allowlist constants
- Create: `src/vergil_tooling/configs/typescript/` (packaged **shareable strict
  base `tsconfig.base.json`**, ESLint flat config, Prettier config)
- Test: `tests/lib/test_languages.py`

The meaty task — the concrete tool commands. Depends on T3 (schema). Reuses the
existing `Cardinality` concept (no new machinery).

- [ ] **INSTALL.** `npm ci` (clean, lockfile-pinned).
- [ ] **LINT (`once`).** `prettier --check .` and `eslint .` (flat config with
  type-aware typescript-eslint rules), using the packaged `{configs}/typescript/*`
  configs; include `@typescript-eslint/ban-ts-comment` to enforce the
  no-suppression rule.
- [ ] **TYPECHECK (`once`).** `tsc --noEmit`. **This task authors and tests the
  concrete "curated extras" set** in the packaged `tsconfig.base.json` (`strict`
  + the extras listed in Global constraints) — the base config consumers `extends`.
  T8 later *documents* this already-decided set (breaking any T4↔T8 loop).
- [ ] **TEST (`per-version`).** `vitest run --coverage` with V8 provider and a
  100% line threshold (configured so the gate fails under 100%). Note the
  `@vitest/coverage-istanbul` contingency (spec §4 caveat 2) if V8 source-map
  mapping is imprecise at the 100% bar.
- [ ] **AUDIT (`once`).** `npm audit` with an explicit severity threshold and
  `--omit=dev` scope (spec §4 caveat 1); note the OSV-Scanner-over-`package-lock.json`
  contingency. Add a best-effort license-metadata check + a TypeScript license
  allowlist constant.
- [ ] **EcosystemInfo + cardinality.** Fill `EcosystemInfo` (build/publish/env);
  declare TYPECHECK, LINT, AUDIT `once`, TEST `per-version` via the existing
  `cardinality` field.
- [ ] **Tests.** `language_commands("typescript", kind)` returns the expected argv
  per kind, `{configs}` expands, and cardinality is as declared.
- [ ] **Validate + commit** (`feat(languages): TypeScript registry entry (node)`).

**Deliverable:** `vrg-validate` knows every TypeScript stage command; the shareable
base tsconfig ships.
**Depends on:** T3. **Blocks:** T5, T7, T8, T10.

### T5 — Container maps + language detection (vergil-tooling)

**Files:**
- Modify: `src/vergil_tooling/lib/container.py:18` (`_DEFAULT_VERSIONS`,
  `_DEFAULT_TEST_COMMANDS`, `detect_language`, image resolution)
- Test: `tests/lib/test_container.py`

- [ ] **Detection.** `detect_language` returns `"typescript"` on `tsconfig.json`
  (and/or a `package.json` carrying a `typescript` devDependency). Test each
  marker; test that a non-TS `package.json` alone does **not** misdetect.
- [ ] **Image resolution.** Parse the runtime family from the `node-` version-tag
  prefix and build `prod-ts-node:<major>` (matching T1/T2 image names). Test
  `node-24 → prod-ts-node:24`, `node-22 → prod-ts-node:22`; a malformed tag falls
  back like an unknown language (no `prod-ts-node:` with empty major).
- [ ] **Defaults.** `_DEFAULT_VERSIONS["typescript"] = "node-24"` (primary) and
  `_DEFAULT_TEST_COMMANDS["typescript"]` (an `npm ci && vitest run` line for
  `vrg-container-test`).
- [ ] **Validate + commit** (`feat(container): TypeScript image resolution + detection`).

**Deliverable:** `vrg-container-run`/`vrg-container-test` select the right TS image;
detection works.
**Depends on:** T3, T4. **Blocks:** T10.

### T6 — CodeQL support + primary→CodeQL mapping + repo-init wiring (vergil-tooling)

**Files:**
- Modify: `src/vergil_tooling/lib/github_config.py:225`
  (`_CODEQL_SUPPORTED_LANGUAGES`)
- Modify: `src/vergil_tooling/lib/repo_init.py` (`_container_suffix`,
  `_container_tag`, and `_LANGUAGE_ACTION_PATTERNS` if needed)
- Test: `tests/lib/test_github_config.py`, `tests/lib/test_repo_init.py`

The pushback-flagged task (spec §6 point 8/8a) — the one place C++ is **not** a
clean copy-paste, because TypeScript is the first language whose primary-language
name (`typescript`) differs from its CodeQL identifier (`javascript-typescript`).

- [ ] **CodeQL gate emission.** Add **`"typescript"`** (the *primary-language*
  key, **not** `javascript-typescript`) to `_CODEQL_SUPPORTED_LANGUAGES` — this set
  is tested against `project.primary_language` (github_config.py:308). Test that a
  `typescript` repo gets the CodeQL gate. (`repo_init._CODEQL_LANGUAGES` already
  lists `typescript`; assert the two agree.)
- [ ] **repo-init.** `_container_suffix`/`_container_tag` produce the `node-`-aware
  suffix/tag for generated `vergil.toml`/CI; add any needed allowed-action
  patterns. Test the generated scaffolding for a `typescript` repo (the emitted
  `ci.yml` carries `language: typescript` — the primary-language, used for
  container resolution).
- [ ] **Validate + commit** (`feat(github-config): support typescript for CodeQL + repo-init`).

**Deliverable:** the CodeQL gate is emitted for TS repos; `repo-init` scaffolds
TS correctly. **The Action-side `typescript → javascript-typescript` mapping lands
in T7**, because `ci.yml`'s `language:` must remain the primary-language for
container resolution.
**Depends on:** T3. **(Independent of T4/T5; can run in parallel.)** **Blocks:** T7.

### T7 — Reusable CI workflows accept `language: typescript` (vergil-actions)

**Files:**
- Modify: the reusable `.github/workflows/ci-*.yml` (quality/test/audit/security)
- Test: workflow-level (a dry-run / sample invocation)

- [ ] **Language input.** Accept `language: typescript` and the `node-` container
  suffix so jobs pull `prod-ts-node:<major>` images by version tag.
- [ ] **CodeQL mapping (pushback fix, spec §6 point 8a).** In the CodeQL step, map
  the `language` input `typescript → javascript-typescript` for the codeql-action
  `languages:` input (bare `typescript` is not a valid CodeQL analysis language).
  Confirm whether `cpp` needs the same map (`cpp → c-cpp`) and route it through the
  same mapping if so. Test the resolved CodeQL language for `typescript`.
- [ ] **Cardinality alignment.** Ensure the generated job/gate names match what
  `desired_ci_gates_ruleset` emits (per-version test; once typecheck/lint/audit)
  so required checks line up with branch protection.
- [ ] **Semgrep.** Wire the JS/TS Semgrep rules (`p/typescript`).
- [ ] **Validate + commit** (`feat(ci): TypeScript support in reusable workflows`).

**Deliverable:** reusable workflows run the TS pipeline (TEST per Node version;
once typecheck/lint/audit), with CodeQL analyzing `javascript-typescript`.
**Depends on:** T1, T2, T4, T6. **Blocks:** T10.

### T8 — TypeScript standards docs + base tsconfig docs (vergil-tooling · site docs)

**Files:**
- Create: `docs/site/docs/standards/development/typescript/overview.md`,
  `naming-conventions.md`, `testing-and-coverage.md`, `toolchain-and-strictness.md`

- [ ] **Overview.** The npm + ESM layout, single-`tsc`/single-runtime model, the
  prebuilt-only rule, Vitest-as-runner default, and how consumers `extends` the
  shareable base tsconfig shipped in T4.
- [ ] **Strictness.** Document the concrete "curated extras" set **decided and
  tested in T4** (spec §3.2) and the no-suppression rule — this task describes the
  set, it does not choose it (ownership lives in T4).
- [ ] **Testing & coverage.** The 100%-per-Node-version gate and the **exclusion
  discipline** — what a legitimate `/* v8 ignore */` looks like vs. abuse
  (whole-file exemptions).
- [ ] **Validate + commit** (`docs(ts): TypeScript development standards`).

**Deliverable:** per-language standards docs matching the `python/`/`cpp/` pattern.
**Depends on:** T4 (final tool commands + base tsconfig). **(Doc-review sweep
#2738 may spawn siblings in `docs`.)**

### T9 — Publish TypeScript images to GHCR (vergil-containers) · **deployment**

Operational (not PR-workable). Run with `issue-deploy`.

- [ ] **Precondition self-check.** T1 + T2 images build cleanly (probe).
- [ ] **Publish.** Push `prod-ts-node:24` / `prod-ts-node:22` (+ `dev-` variants)
  to GHCR so `vrg-container-run` and CI can pull them. **Any release/tag step is a
  human-gated precondition** (attested, not performed by the agent).
- [ ] **Record** `Outcome: SUCCESS` (image refs + digests) as a comment.

**Deliverable:** TypeScript images are pullable from GHCR.
**Blocked-by:** T1, T2. **Blocks:** T10.

### T10 — Cold-rebuild + full TS pipeline validation (vergil-tooling) · **validation**

Operational (not PR-workable). Run with `issue-validate`. The end-to-end proof.

- [ ] **Precondition self-check.** Images deployed (T9), registry/actions merged
  (T3–T7).
- [ ] **Sample repo.** Stand up a minimal TypeScript repo (`package.json` +
  `package-lock.json` + `tsconfig.json` extending the base + a Vitest suite)
  declaring `primary-language = "typescript"` and `[ci].versions = ["node-24",
  "node-22"]`. This is an **ephemeral validation fixture** (built, exercised,
  discarded) — not a persistent published deliverable.
- [ ] **Cold rebuild.** From clean, `vrg-container-run -- vrg-validate` passes
  every stage: `tsc` warnings-clean typecheck (once), ESLint+Prettier (once), 100%
  V8 coverage under **each** Node version, `npm audit` (once).
- [ ] **Prove the audit detects (spec §4 acceptance).** Pin a dependency with a
  **known CVE**, confirm the AUDIT stage **fails**; revert to the clean version,
  confirm it **passes**. If `npm audit` signal/noise is unworkable, that is the
  trigger to switch T4's AUDIT to the **OSV-Scanner** contingency (and file the
  follow-up). Prevents shipping a no-op audit stage.
- [ ] **Prove CodeQL runs (spec §6 acceptance).** Confirm the CodeQL job actually
  runs and reports for the TS sample repo (analyzing `javascript-typescript`) — not
  merely that the gate name exists.
- [ ] **CI gates.** Confirm `desired_ci_gates_ruleset` emits the expected
  per-version TEST gate and once typecheck/lint/audit gates, and they match the
  vergil-actions job names.
- [ ] **Record** `Outcome: SUCCESS` (or FAILURE with detail; stays open on
  failure) as a comment.

**Deliverable:** demonstrated, reproducible TypeScript pipeline end-to-end.
**Blocked-by:** T3, T4, T5, T6, T7, T9.

---

## Closing bookends (already seeded)

- **Documentation-review sweep** — vergil-tooling#2738 (this repo's site docs +
  spawns per-repo doc siblings); runs **before** the retrospective. **The `docs`
  repo's higher-level summary docs (spec §7) are intentionally *not* a seeded
  up-front task** — they are handled by a `docs`-repo sibling this sweep spawns, so
  its PR lands in `docs` and honors the placement law.
- **Follow-on brainstorm** — .github#286 (adjudicate the spec §9 deferral ledger).
- **Retrospective** — .github#287 (terminal; its docs PR closes the epic).

## Self-review

- **Spec coverage.** §3.1 single runtime/typechecker → T1/T2/T4/T7; §3.2
  warnings-to-11 → T4 (base tsconfig) / T8 (docs); §3.3 matrix + `[typescript]` →
  T3/T5; §3.4 npm + audit → T4; §3.5 prebuilt-only → T1/T2 discovery; §3.6 per-kind
  cardinality (reused) → T4 declares, T7 aligns, T10 verifies; §4 tool matrix → T4
  (+ T1/T2 toolset, T7 SAST); §5 per-version coverage → T4/T8/T10; §6 touch points
  → T3–T6 (+ T7 for the CodeQL mapping); §6 point 8/8a CodeQL split → T6 (gate key)
  + T7 (Action mapping); §7 five-repo spread → all; §9 ledger → follow-on
  brainstorm #286; §10 operational tasks → T9/T10.
- **Placement law.** Every code task is filed in the repo whose PR closes it
  (vergil-containers: T1/T2; vergil-tooling: T3–T6, T8; vergil-actions: T7);
  operational tasks T9 (vergil-containers) / T10 (vergil-tooling) home where they
  run; docs sweep #2738 spawns same-repo siblings rather than crossing a boundary.
- **Cardinality consistency.** The concept is reused from C++ (no build task); T4
  declares it (TYPECHECK/LINT/AUDIT `once`; TEST `per-version`), T7 aligns CI job
  names, T10 verifies the gates.
- **CodeQL split consistency.** T6 adds `typescript` (primary-language key) to the
  gate-emission set; T7 maps `typescript → javascript-typescript` for the Action.
  The `ci.yml` `language:` stays `typescript` for container resolution throughout.
- **No orphaned types.** Image names (`prod-ts-node:<major>`) are produced by
  T1/T2 and consumed by T5/T7; the packaged base tsconfig is produced by T4 and
  consumed by T8/T10; the `[typescript]` `module`/`target` keys are produced by T3
  and consumed by T4.

## Evolution during execution

_Frozen at execution start; dated entries record meaningful deltas — a task
added, dropped, or rescoped, a discovered dependency — with the reasoning._
