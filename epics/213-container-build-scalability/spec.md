# Manifest-driven monorepo for scalable container builds

- **Epic:** vergil-project/.github#213
- **Status:** Design approved 2026-08-05 (brainstormed directly via `epic-create`;
  full epic convention).
- **Repos:** vergil-project/vergil-containers (build system + CI — the bulk of the
  work), vergil-project/.github (epic docs), vergil-project/vergil-tooling
  (`docs/site` CI-architecture guide, doc-review sweep only — no code change; the
  consumer-side image resolver is untouched). Epic homed in `.github`.
- **Promoted from:** direct brainstorm (no triage origin).

## 1. Summary

`vergil-containers` builds every dev/prod image from a single, fully coupled
pipeline. Adding a sixth language (C++) made the scaling ceiling concrete: a
change to one language drags all the others through build + publish, the same
shared tooling is re-installed from scratch in every image, and every image is
built multi-arch with the arm64 half under QEMU emulation.

This epic keeps the **single repository** but makes the build **data-driven from a
language manifest** (`languages.yml`). Everything — which images a PR builds, the
nightly matrix, release — derives from the manifest, so adding a language becomes
a *data* change (one manifest row + one template directory), not a structural one.
Four cohesive workstreams deliver it, each independently shippable:

1. **Manifest as single source of truth** — replace the hand-maintained build
   matrices (in `build.sh` and the CD workflow) with one `languages.yml`.
2. **Selective PR builds** — a PR builds only the languages whose inputs changed;
   a change to *shared* inputs correctly fans out to all.
3. **Shared tooling layer** — build the shareable common binaries once and
   `COPY --from` them into each language image, instead of 16× from scratch.
4. **Native arm64 runners** — build each architecture on its native runner and
   stitch a manifest list, retiring QEMU emulation.

The nightly full rebuild is **preserved deliberately** — it is the CVE-freshness
backstop, and it is precisely what *lets* PR-time builds go selective without any
language falling behind.

## 2. Motivation

### 2.1 The current shape (facts)

- **16 language image variants** across 6 languages, plus a standalone `base`
  image:
  python (3.12/3.13/3.14), ruby (3.2/3.3/3.4), java (17/21), go (1.25/1.26),
  rust (1.92/1.93), cpp-clang (19/20), cpp-gcc (13/14).
- The build matrix is **hand-maintained in two places** that must be kept in sync:
  `docker/build.sh` (local) and the `matrix.include` list in
  `.github/workflows/cd-docker-publish.yml` (CI).
- Language images do **not** `FROM` `base`. Each is `FROM` its official upstream
  image (`python:3.14-slim`, `ruby:3.3`, `golang:1.26`, …) and independently
  `@include`s the same `docker/common/*.dockerfile` fragments. So the coupling
  between languages is **shared source**, not a layer chain: `docker/common/`,
  `docker/pins/`, `.trivyignore`, `.hadolint.yaml`, and the publish workflow.
- Every image is built `linux/amd64,linux/arm64` via `docker/build-push-action`
  with QEMU; the arm64 half runs under emulation.
- **Nightly** (`.github/workflows/ops.yml`, 06:15 UTC) rebuilds all 16 variants +
  base, for **both `dev` and `prod`**, with `no-cache: true` — 34 multi-arch
  from-scratch builds every night. **Release** (push to `main`) version-bumps and
  rebuilds all `prod` images.

### 2.2 The four problems

1. **Non-local blast radius.** A PR touching one language rebuilds the planet. The
   matrix runs `fail-fast: false` in parallel, so this is a *throughput and cost*
   problem more than a wall-clock one: GitHub schedules 16+ multi-arch builds for
   a change that affected one.
2. **Redundant per-build work.** The shareable common binaries (§4) are downloaded,
   checksum-verified, and installed from scratch in every image that includes them
   — repeated across images, doubled across `dev`/`prod`, nightly.
3. **Emulated arm64.** The arm64 half of every image is built under QEMU — a large,
   avoidable per-build time sink.
4. **Coupled failure.** One language's scanner finding or broken build creates
   noise across the run and couples release of all `prod` images.

### 2.3 The reframe that makes this tractable

The historical sin was *rebuild-everything-on-every-PR*, adopted before the
nightly existed, as the freshness mechanism. The nightly now guarantees
freshness, so PR-time builds can safely go **selective**: build only what changed,
and let the nightly keep everything — including the languages nobody touched today
— current with CVEs. This is the load-bearing insight of the whole epic.

## 3. Approach chosen and alternatives rejected

**Approach 3 — manifest-driven monorepo.** Selected after a full brainstorm over
three options.

- **Approach 1 (optimize in place, no manifest).** A subset of Approach 3. Its
  build-efficiency pieces (shared tooling layer, native arm runners) are folded in
  here as orthogonal workstreams; what it lacks is the manifest, which is what
  turns "add a language" into a data change and keeps a future split cheap.
- **Approach 2 (per-language repos).** Rejected. Its only durable benefit is
  independent governance / release trains per language. For solo-practitioner
  tooling that trigger will effectively never fire, and the repo semver is already
  vestigial (the **images**, tag-versioned, are the real deliverable — not the
  repo). Meanwhile a split multiplies per-repo machinery (nightly cron, pins,
  trivyignore, scanner config, reusable-workflow pins × N) and invites exactly the
  **neglect** this epic guards against: the config for languages you are *not*
  actively working on is the config that silently rots. A monorepo centralizes the
  freshness machinery so inactive languages ride the same train automatically.
  The manifest is the seam along which a split — should the governance trigger ever
  arrive — remains a cheap, later refactor rather than a foreclosed option.

### Design goals

1. **Locality** — a change to one language rebuilds only that language.
2. **Freshness preserved** — the nightly still rebuilds *everything* for CVEs.
3. **Cheap per build** — build the shareable common tooling once, not per image.
4. **Failure isolation** — one language's build/scan failure does not gate others.
5. **Near-zero marginal cost to add a language** — doubling languages doubles
   manifest rows, not machinery.

## 4. Design decisions

### 4.1 `languages.yml` — the single source of truth

A committed manifest at `docker/languages.yml` replaces the two hand-maintained
matrices. One entry per language, listing its build versions and the metadata the
build needs. Illustrative shape (final schema settled in implementation):

```yaml
languages:
  python:
    build-arg: PYTHON_VERSION
    versions: ["3.12", "3.13", "3.14"]
    context: docker/python
    # inputs whose change triggers a rebuild of THIS language (in addition to
    # the always-shared inputs listed once, globally, below)
    paths: ["docker/python/**"]
  cpp-clang:
    build-arg: CLANG_VERSION
    versions: ["19", "20"]
    context: docker/cpp-clang
    paths: ["docker/cpp-clang/**", "docker/cpp/**"]
    smoke: docker/cpp/smoke-test.sh   # optional per-language post-build check
  # … ruby, java, go, rust, cpp-gcc
shared-paths:   # a change here fans out to ALL languages + base
  - docker/common/**
  - docker/pins/**
  - docker/base/**
  - .trivyignore
  - .hadolint.yaml
```

Both `docker/build.sh` (local) and the CD workflow derive their matrix from this
file. `build.sh` becomes a thin reader; the CD `matrix.include` block is replaced
by a **dynamic matrix** emitted by a setup job (§4.2). CI gains a check that the
manifest is internally consistent (every `context`/`smoke` path exists, every
language has a `build-arg`), analogous to the existing `pins` check.

**Deliberately no `active`/`inactive` flag.** An early design sketch had a toggle
to drop dormant languages from the *default* matrix. It is rejected: it
reintroduces the neglect risk (a toggled-off language quietly stops getting
rebuilt and falls behind on CVEs). Selective *PR* builds already give "leave
untouched languages alone until tonight"; the nightly then rebuilds **all**
languages unconditionally. Freshness for every language is non-negotiable and is
never gated on a flag.

### 4.2 Selective PR builds (dynamic matrix from changed paths)

A `setup` job computes the build matrix and the rest of the pipeline consumes it:

1. `setup` reads `languages.yml` and the PR's changed files (`git diff` against the
   base). For each language, if any of its `paths` changed **or** any
   `shared-paths` changed, the language is included. It emits the matrix as JSON.
2. `build-scan-push` runs `strategy.matrix: ${{ fromJSON(needs.setup.outputs.matrix) }}`.
   A PR that touched only `docker/cpp-clang/**` schedules only the cpp-clang jobs —
   the other languages are **not scheduled** (not "run and pass fast"). The PR's CI
   time equals that language's build time, exactly as a dedicated repo would give.
3. A change to any `shared-path` yields the full matrix — correct, because those
   inputs genuinely affect every image, and a shared-tool bump *should* be proven
   against all languages before it merges.

The **nightly and release paths ignore the diff** and always build the full matrix
from the manifest (freshness). Selectivity is a PR-only optimization.

Edge cases to handle explicitly in implementation: an empty matrix (a PR that
touched no image inputs — e.g. docs only) must make the build job **skip
cleanly**, not error; and a required-status-check name must remain stable even
when a language is not built, so branch protection is not defeated by a job that
did not run (use a single aggregating "build gate" job that depends on the dynamic
matrix and passes when all scheduled builds pass, including the zero-scheduled
case).

### 4.3 Shared tooling layer (`common-tools`, COPY-from — scoped honestly)

The `docker/common/` fragments fall into two classes, and only one is cleanly
shareable:

- **COPY-portable — downloaded binaries into `/usr/local/bin`** (9 tools):
  `shellcheck`, `shfmt`, `actionlint`, `git-cliff`, `hadolint`
  (`validation-tools`); `scorecard`, `trivy` (`security-tools`); `tofu`
  (`opentofu`); `nfpm`. Each currently does a per-arch download + checksum verify
  in every image that includes it. These self-contained binaries can be built once
  into a `common-tools` builder image and pulled in with:

  ```dockerfile
  FROM ghcr.io/vergil-project/common-tools:latest AS tools
  # … language base …
  COPY --from=tools /usr/local/bin/shellcheck /usr/local/bin/shfmt … /usr/local/bin/
  ```

  buildx resolves `--from=tools` per target platform, so multi-arch is preserved
  (the `common-tools` image is itself built multi-arch). The download + checksum
  work is then paid **once per build round**, not once per image.

- **Runtime-coupled — NOT cleanly COPY-portable** (kept as `@include` installs for
  now): `node` + `markdownlint-cli`, `python3` + uv tools (`yamllint`,
  `ansible-lint`), `pandoc`, `gh`. These need their language runtime or apt
  integration present, so copying just a binary is insufficient. Whether to fold
  any of them into the shared layer (e.g. a multi-stage copy that brings the
  runtime too, or a shared base the runtime-heavy CI image uses) is an explicit
  **sub-decision deferred to implementation** (§9), scoped and measured rather than
  assumed. The clean, low-risk first cut is the 9 binaries above.

This workstream introduces a build ordering dependency **only when it changes**:
`common-tools` builds before the language images that copy from it. In the nightly
and CD, `common-tools` becomes a prerequisite job; language jobs `needs` it. This
is the point of the optimization — the shared binaries are built once and reused,
instead of 16×.

### 4.4 Native arm64 runners (retire QEMU)

Replace the single QEMU multi-arch build with a per-architecture build on its
native runner, then stitch a manifest list:

- Build `linux/amd64` on `ubuntu-latest` and `linux/arm64` on `ubuntu-24.04-arm`
  (GitHub-hosted native arm64), each pushing a per-arch image (or by-digest),
  then `docker buildx imagetools create` the final multi-arch tag from the two
  digests. The existing candidate → scan → promote → attest flow is preserved;
  only *where each arch is built* changes.
- This removes emulation from the hot path and is **orthogonal** to the manifest —
  it can land independently and benefits every image immediately.
- The existing arm64 Trivy scan (currently a bespoke emulated `docker run`) is
  simplified to a native scan on the arm runner.

### 4.5 Failure isolation

Two mechanisms, both falling out of the manifest:

- The nightly becomes a **per-language matrix of independent jobs** (already
  `fail-fast: false`); a language's failure is reported against that language and
  does not stop the others.
- A single aggregating **build-gate** job (§4.2) gives branch protection one stable
  required check while individual language jobs remain independently
  green/red/skipped.

### 4.6 Consumer transparency — an invariant, not a workstream

Every consuming repo resolves images by GHCR name/tag
(`ghcr.io/vergil-project/{prefix}-{lang}:{version}`) via
`vergil_tooling/lib/container.py`. **No image name or tag changes in this epic.**
The redesign is invisible to consumers, and the C++ family's suffix-on-name
convention (`prod-cpp-clang:20`, `prod-cpp-gcc:14`) is preserved. This is a hard
acceptance invariant, checked by the cold-rebuild validation (§8).

## 5. Non-goals

- **No repo split**, and no per-language repo semver (repo semver stays vestigial
  fleet-sync).
- **No change to GHCR image names/tags** — see §4.6.
- **No dropping the nightly rebuild** — freshness is preserved by design.
- **No change to the consumer-side resolver** (`container.py`) or to `vrg-actions`
  reusable workflows (the publish workflow is repo-local).
- **No new language added** here — this epic makes *adding* one cheap; the first
  proof that it worked is adding one in a follow-on, not in scope now.

## 6. Rollout — independently shippable stages

Ordered so each stage is verifiable in isolation and low-risk:

- **Stage A — Manifest + matrix generator (behavior-preserving refactor).**
  Introduce `docker/languages.yml`; make `build.sh` and the CD matrix derive from
  it; add the manifest-consistency CI check. The emitted matrix reproduces
  **today's full matrix exactly** — no behavior change — so it is verifiable by
  diffing the generated matrix against the current hand-written one.
- **Stage B — Selective PR builds.** Add the `setup` diff-aware matrix job and the
  aggregating build-gate; nightly/release stay full. The headline behavior change.
- **Stage C — Native arm64 runners.** Split the multi-arch build onto native
  runners + manifest stitch. Orthogonal; can be sequenced before or after B.
- **Stage D — Shared `common-tools` layer.** Build the 9 binaries once; `COPY
  --from` into language images; `common-tools` becomes a prerequisite job. The
  runtime-coupled sub-decision (§4.3) is scoped here.
- **Cold-rebuild validation** (see §8) runs after the implementation stages land.

Stages C and D are the build-*efficiency* wins and are independent of A/B; A is the
foundation for B. `writing-plans` refines this into concrete tasks and their
`Blocked-by` ordering.

## 7. Risks and mitigations

- **Branch protection defeated by skipped jobs.** A required check that names a
  per-language job would "pass" by never running. → Single aggregating build-gate
  job is the required check (§4.2).
- **Manifest drift from reality.** A manifest that disagrees with the tree (missing
  context dir, stale version) silently mis-builds. → CI consistency check, same
  pattern as the existing `pins` check.
- **Shared-layer ordering / staleness.** Language images copying from a stale
  `common-tools` could ship old tools. → `common-tools` is rebuilt in the same
  nightly/CD round before the language jobs that `needs` it; no-cache nightly keeps
  it fresh.
- **Native-arm runner availability/cost.** GitHub-hosted arm runners are a distinct
  runner class. → If unavailable/undesirable, Stage C is independently revertible to
  QEMU without affecting A/B/D.
- **Selective build hides a real breakage** (a shared change that only manifests in
  an unbuilt language). → `shared-paths` deliberately fans out to the full matrix;
  the nightly is the backstop that would catch anything slipping through on a PR.

## 8. Validation (cold-rebuild)

Acceptance for the build-architecture change is proven by a **cold rebuild**, not
only by unit checks — this is an infrastructure epic. A `validation`-kind task
(seeded at plan time, `Blocked-by` the implementation tasks) will:

1. Run the full, no-cache, manifest-driven build of **all** language variants +
   base for both arches (the nightly path) and confirm every image builds and its
   per-language smoke check (where defined, e.g. the C++ CMake+Conan smoke) passes.
2. Confirm the produced GHCR names/tags are **byte-for-byte the same set** as before
   the epic (the §4.6 invariant), so consumers resolve unchanged.
3. Confirm a **selective PR build** schedules only the expected languages for a
   representative single-language change, and the full matrix for a `shared-paths`
   change.
4. Record `Outcome: SUCCESS` as a comment; the task closes only on success.

## 9. Deferred / open questions (ledger — nothing falls out silently)

- **Runtime-coupled common tooling** (§4.3): whether/how to bring `node` +
  `markdownlint`, `python3` + uv tools, `pandoc`, `gh` into the shared layer, or
  leave them as per-image installs. First cut leaves them per-image; revisit with
  measured build-time data.
- **`common-tools` publication surface**: is it a published GHCR image (like
  `base`) or a local-only build stage? Leaning published for CD reuse; settle in
  Stage D.
- **Manifest schema finalization**: exact keys, and whether per-version overrides
  (e.g. a version-specific base image or build-arg) are needed now or deferred
  until a language demands one (YAGNI: add when first required).
- **Stage C runner strategy**: GitHub-hosted `ubuntu-24.04-arm` vs. any
  self-hosted option; and whether amd64 also benefits from a larger runner class.
- **First "add a language" proof**: adding a new language (e.g. the long-mooted
  Perl, or Swift per the C++ epic's LLVM setup) is the natural validation that the
  manifest achieved its goal — captured as a *follow-on*, not scoped here.
