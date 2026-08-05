# Manifest-driven monorepo for scalable container builds

- **Epic:** vergil-project/.github#213
- **Status:** Design approved 2026-08-05 (brainstormed directly via `epic-create`;
  revised after `paad:pushback`).
- **Repos:** vergil-project/vergil-containers (build system + CI — the bulk of the
  work), vergil-project/.github (epic docs), vergil-project/vergil-tooling
  (`docs/site` CI-architecture guide, doc-review sweep only — no code change; the
  consumer-side image resolver is untouched). Epic homed in `.github`.
- **Promoted from:** direct brainstorm (no triage origin).

## 1. Summary

`vergil-containers` builds every dev/prod image from a single, fully coupled
pipeline. Adding a sixth language (C++) made the scaling ceiling concrete: a
change merged to `develop` rebuilds and republishes **all** dev images regardless
of what changed, the same shared tooling is re-installed from scratch in every
image, and every image is built multi-arch with the arm64 half under QEMU
emulation.

This epic keeps the **single repository** but makes the build **data-driven from a
language manifest** (`docker/languages.yml`). Everything — which images a change
rebuilds, the nightly matrix, release — derives from the manifest, so adding a
language becomes a *data* change (one manifest row + one template directory), not a
structural one. Four cohesive workstreams deliver it, each independently shippable,
**preceded by a measurement stage** so the effort is prioritized by evidence:

0. **Instrument the current build** — capture per-image, per-stage cold-build
   timings so the optimization workstreams are sequenced by data, not assumption.
1. **Manifest as single source of truth** — replace the hand-maintained build
   matrices (in `build.sh` and the CD workflow) with one `languages.yml`.
2. **Selective builds on the `develop`-push publish** — a merge to `develop`
   rebuilds only the images whose *bytes* actually change; a change to shared build
   inputs correctly fans out.
3. **Shared tooling layer** — build the shareable common binaries once and
   `COPY --from` them into each language image, instead of re-downloading per image
   (scoped to the portable binaries; gated on Stage 0 data).
4. **Native arm64 runners** — build each architecture on its native runner and
   stitch a manifest list, retiring QEMU emulation.

The nightly full rebuild is **preserved deliberately** — it is the CVE-freshness
backstop, and it is precisely what *lets* on-change builds go selective without any
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
  `@include`s a subset of the `docker/common/*.dockerfile` fragments (via
  `generate.sh`, which expands `# @include common/<fragment>` directives). So the
  coupling between languages is **shared source**, not a layer chain.
- Every image is built `linux/amd64,linux/arm64` via `docker/build-push-action`
  with QEMU; the arm64 half runs under emulation.
- **Where builds actually fire** (important — PRs do *not* build images):
  - **PR** (`ci.yml`): hadolint (lint), pins-check, shell quality/security,
    version, docs. **No image build/publish.**
  - **push to `develop`** (`cd.yml` → `cd-docker-publish.yml`): build + publish the
    **full 16-variant + base matrix** with the `dev` prefix.
  - **push to `main`** (release): same, `prod` prefix.
  - **nightly** (`ops.yml`, 06:15 UTC): both prefixes, `no-cache: true`.

### 2.2 The four problems

1. **Non-local blast radius on merge.** A merge to `develop` rebuilds and
   republishes **all 16 dev images** — Python, Ruby, Go, Rust, Java, C++ — even
   when the change touched one language. The matrix runs `fail-fast: false` in
   parallel, so this is primarily a *throughput and cost* problem (GitHub schedules
   16+ multi-arch builds for a one-language change), doubled again by the `prod`
   rebuild on release.
2. **Redundant per-build work.** Shared tooling is installed from scratch in every
   image that includes it, repeated across images and doubled across `dev`/`prod`,
   nightly. **Unquantified today** — Stage 0 (§6) measures it before we optimize.
3. **Emulated arm64.** The arm64 half of every image is built under QEMU — a large,
   avoidable per-build time sink.
4. **Coupled failure.** One language's scanner finding or broken build creates
   noise across the run and couples release of all `prod` images.

### 2.3 The reframe that makes this tractable

The historical sin was *rebuild-everything-on-every-change*, adopted before the
nightly existed, as the freshness mechanism. The nightly now guarantees freshness,
so on-change builds (the `develop`-push publish) can safely go **selective**: build
only what changed, and let the nightly keep everything — including the languages
nobody touched today — current with CVEs. This is the load-bearing insight of the
whole epic.

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
build needs — **but not a hand-maintained list of trigger paths** (that would be
the error-prone thing selective builds must avoid; see §4.2). Illustrative shape
(final schema settled in implementation):

```yaml
languages:
  python:
    build-arg: PYTHON_VERSION
    versions: ["3.12", "3.13", "3.14"]
    context: docker/python
  cpp-clang:
    build-arg: CLANG_VERSION
    versions: ["19", "20"]
    context: docker/cpp-clang
    smoke: docker/cpp/smoke-test.sh   # optional per-language post-build check
  # … ruby, java, go, rust, cpp-gcc
base:
  context: docker/base                 # first-class pseudo-language (§4.2)
```

Both `docker/build.sh` (local) and the CD workflow derive their matrix from this
file. `build.sh` becomes a thin reader; the CD `matrix.include` block is replaced
by a **dynamic matrix** emitted by a setup job (§4.2). CI gains a check that the
manifest is internally consistent (every `context`/`smoke` path exists, every
language has a `build-arg`), analogous to the existing `pins` check.

**`base` is a first-class matrix entry.** It is not a language, but it is built by
the same publish workflow and consumed as the CI runner container (`ci.yml` uses
`container: ghcr.io/vergil-project/prod-base:latest`). The matrix generator treats
it as a pseudo-language so it is rebuilt on its own inputs and on any fragment it
includes.

**Deliberately no `active`/`inactive` flag.** An early sketch had a toggle to drop
dormant languages from the *default* matrix. It is rejected: it reintroduces the
neglect risk (a toggled-off language quietly stops getting rebuilt and falls behind
on CVEs). Selective on-change builds already give "leave untouched languages alone
until tonight"; the nightly then rebuilds **all** languages unconditionally.
Freshness for every language is non-negotiable and is never gated on a flag.

### 4.2 Selective builds on the `develop`-push publish

Selectivity targets the **`develop`-push CD publish** (and, identically, could
apply to any on-change publish) — *not* PRs, which build no images today (§2.1).
The nightly and release paths **ignore the diff and always build the full matrix**
(freshness / release-snapshot semantics; see §4.6).

A `setup` job computes the matrix from the manifest and the push's changed files
(`git diff` over the push range, `github.event.before..after`), and the build job
consumes it via `strategy.matrix: ${{ fromJSON(needs.setup.outputs.matrix) }}`.

**Determinism — the dependency graph is *derived from source*, not asserted.** An
image's real inputs are: its `context` directory, and the exact `common/` fragments
its `Dockerfile.template` `@include`s. `generate.sh` already parses those
`@include` directives, so the same parse yields a precise, self-maintaining
dependency graph: a change to `common/security-tools.dockerfile` rebuilds exactly
the images whose templates include it (Python does not, so Python does not rebuild).
The manifest-consistency check guarantees every include resolves, so the graph
cannot drift from what actually goes into each image.

**Input classification — rebuild inputs vs. check-config.** A file is a *rebuild
input* **iff changing it changes the image's bytes**. Everything else is
check/scan/report config and triggers its own re-check lane, never an image
rebuild:

| File(s) | Changes image bytes? | Trigger |
|---|---|---|
| `docker/<lang>/**` template | **Yes** | Rebuild that language |
| `docker/common/*.dockerfile` fragment | **Yes** | Rebuild only images whose template `@include`s it (derived graph) |
| `docker/base/**` | **Yes** | Rebuild `base` |
| `docker/generate.sh` | **Yes** (generates the Dockerfile) | Fan-out: rebuild all |
| `docker/languages.yml` | **Yes** (versions / build-args) | Rebuild per the diff |
| `.trivyignore`, `trivy.yaml`, `.trivy/` | No — scan gating | Re-scan lane |
| `.hadolint.yaml` | No — Dockerfile lint config | Re-lint (hadolint job) |
| `docker/pins/**` (`pins.yml` + scripts) | No — pin *justification* + CI check; the image-affecting `ARG …_VERSION` lines live in the templates/fragments | Pins-check lane |
| `docker/cpp/smoke-test.sh` | No — post-build validation | Re-smoke, not rebuild |
| `cliff.toml`, `vergil.toml`, release-notes config | No — tooling/release config | n/a to image builds |

So the "fan-out to all" set is small — `generate.sh`, `languages.yml`, and the
workflow itself — with `base/**` rebuilding `base`, and `common/` fragments scoped
precisely by the include-graph. This is the guarantee behind the determinism
concern: nothing that changes image bytes is missed, and nothing that only changes
gating triggers a wasteful rebuild.

**Edge cases handled explicitly:** an empty matrix (a push that touched no image
inputs — e.g. docs only) must make the build job **skip cleanly**, not error; and a
single aggregating **build-gate** job depends on the dynamic matrix and passes when
all *scheduled* builds pass (including the zero-scheduled case), so branch
protection has one stable required check regardless of what was built (§4.5).

### 4.3 Shared tooling layer (`common-tools`, COPY-from — scoped, and gated on data)

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

- **Runtime-coupled — NOT cleanly COPY-portable** (kept as `@include` installs):
  `node` + `markdownlint-cli`, `python3` + uv tools (`yamllint`, `ansible-lint`),
  `pandoc`, `gh`, and in `base` the `semgrep` native compile + mkdocs. These need
  their language runtime or apt integration present. **These are likely the actual
  slow, redundant installs** — so the shared-binary COPY optimizes the *cheap*
  part. Whether to de-duplicate the runtime-heavy tools is **"D2" in the ledger**
  (§9): it requires a shared *runtime* base image that language images `FROM`
  (inverting today's official-upstream-base model and installing language runtimes
  ourselves), a much bigger and riskier change. D2 is considered **only if Stage 0
  data shows those tools dominate** — not a blind commitment.

**Sequencing and gating.** This workstream (call it **D1**, the 9 binaries only) is
sequenced **last** and its go/no-go is **gated on Stage 0** (§6): if the binaries
are not a meaningful fraction of build time, D1's added build-ordering complexity
(a `common-tools` prerequisite job that language jobs `needs`) is not worth it.

**Local parity.** If D1 lands, `build.sh` builds or pulls `common-tools` before the
language images that `COPY --from` it, so a local `build.sh cpp-clang` keeps
working without manual setup.

### 4.4 Native arm64 runners (retire QEMU)

Replace the single QEMU multi-arch build with a per-architecture build on its
native runner, then stitch a manifest list:

- Build `linux/amd64` on `ubuntu-latest` and `linux/arm64` on `ubuntu-24.04-arm`
  (GitHub-hosted native arm64), each producing a per-arch image by digest, then
  `docker buildx imagetools create` the final multi-arch tag from the two digests.
- This is **orthogonal** to the manifest and benefits every image immediately, so
  it can land independently.
- **This genuinely reworks the promote/attest graph, not merely relocates it.** The
  current flow builds one multi-arch image, scans both arches, promotes the
  candidate preserving a single digest, and attests that subject. Splitting into two
  native builds means two per-arch digests, a manifest stitch, per-arch scans on the
  native runners, and a **provenance attestation subject that becomes the stitched
  manifest** (and/or per-arch subjects). Feasible and worthwhile, but the candidate
  → scan → promote → attest job graph is restructured, not preserved as-is. If
  native arm runners prove unavailable or undesirable, this stage is independently
  revertible to QEMU without touching the manifest/selective work.

### 4.5 Failure isolation

Two mechanisms, both falling out of the manifest:

- The nightly and on-change publishes become a **per-language matrix of independent
  jobs** (already `fail-fast: false`); a language's failure is reported against that
  language and does not stop the others.
- A single aggregating **build-gate** job (§4.2) gives branch protection one stable
  required check while individual language jobs remain independently
  green/red/skipped.

### 4.6 Consumer transparency and release semantics — invariants

- Every consuming repo resolves images by GHCR name/tag
  (`ghcr.io/vergil-project/{prefix}-{lang}:{version}`) via
  `vergil_tooling/lib/container.py`. **No image name or tag changes in this epic.**
  The C++ family's suffix-on-name convention (`prod-cpp-clang:20`,
  `prod-cpp-gcc:14`) is preserved. Hard acceptance invariant, checked by the
  cold-rebuild validation (§8).
- **Release (`main`) rebuilds the full `prod` matrix** — a release is a coherent
  point-in-time snapshot of all images. Selectivity is a `develop`-push
  optimization only; the nightly keeps every prod image fresh regardless.

## 5. Non-goals

- **No repo split**, and no per-language repo semver (repo semver stays vestigial
  fleet-sync).
- **No change to GHCR image names/tags** — see §4.6.
- **No dropping the nightly rebuild** — freshness is preserved by design.
- **No change to the consumer-side resolver** (`container.py`) or to `vrg-actions`
  reusable workflows (the publish workflow is repo-local).
- **No PR-time image build** as a hard requirement — an optional PR build-validation
  is captured in the ledger (§9, "Option B"), deferred because post-merge build
  failures are already caught.
- **No new language added** here — this epic makes *adding* one cheap; the first
  proof is a follow-on, not scope now.

## 6. Rollout — independently shippable stages

Ordered so each stage is verifiable in isolation and low-risk:

- **Stage 0 — Instrument the current build.** Capture per-image, per-stage cold
  (`no-cache`) build timings for one nightly-equivalent run. Cheap, and it turns
  "redundant work is a problem" into evidence that prioritizes Stages 2–4 and gates
  D1. Deliverable: a recorded baseline (committed report or issue comment).
- **Stage A — Manifest + matrix generator (behavior-preserving refactor).**
  Introduce `docker/languages.yml`; make `build.sh` and the CD matrix derive from
  it (including `base` as a first-class entry); add the manifest-consistency CI
  check. The emitted matrix reproduces **today's full matrix exactly** — verifiable
  by diffing generated vs. current hand-written matrix.
- **Stage B — Selective on-change builds.** Add the `setup` diff-aware matrix job
  (derived include-graph + input-classification), the aggregating build-gate, and
  the check-config re-check lanes (re-lint / re-scan / pins-check). Nightly and
  release stay full. The headline behavior change.
- **Stage C — Native arm64 runners.** Split the multi-arch build onto native
  runners + manifest stitch; rework promote/attest accordingly. Orthogonal; can be
  sequenced before or after B.
- **Stage D1 — Shared `common-tools` layer (gated on Stage 0).** Build the 9
  binaries once; `COPY --from` into language images; `common-tools` becomes a
  prerequisite job; `build.sh` local parity. Sequenced last; skipped if Stage 0
  shows insufficient benefit.
- **Cold-rebuild validation** (§8) runs after the implementation stages land.

`writing-plans` refines this into concrete tasks and their `Blocked-by` ordering.

## 7. Risks and mitigations

- **Selective build misses a real dependency and ships stale bytes.** → The
  dependency graph is *derived* from `@include` parsing (not hand-listed), the
  input-classification table defines exactly what rebuilds, and the nightly is the
  backstop. The manifest-consistency check guards the graph.
- **Branch protection defeated by skipped jobs.** → Single aggregating build-gate
  job is the required check (§4.2/§4.5).
- **Manifest drift from reality.** → CI consistency check, same pattern as the
  existing `pins` check.
- **Shared-layer ordering / staleness (D1).** → `common-tools` is rebuilt in the
  same round before the language jobs that `needs` it; `no-cache` nightly keeps it
  fresh.
- **Native-arm runner availability/cost (C).** → Independently revertible to QEMU
  without affecting the rest.
- **Optimizing unmeasured work.** → Stage 0 precedes and prioritizes Stages 2–4;
  D1 is explicitly gated on it.

## 8. Validation (cold-rebuild)

Acceptance for the build-architecture change is proven by a **cold rebuild**, not
only by unit checks — this is an infrastructure epic. A `validation`-kind task
(seeded at plan time, `Blocked-by` the implementation tasks) will:

1. Run the full, no-cache, manifest-driven build of **all** language variants +
   base for both arches (the nightly path) and confirm every image builds and its
   per-language smoke check (where defined, e.g. the C++ CMake+Conan smoke) passes.
2. Confirm the produced GHCR names/tags are the **same set** as before the epic
   (the §4.6 invariant), so consumers resolve unchanged.
3. Confirm a **selective on-change build** schedules only the expected images for a
   representative single-language change, the full matrix for a `generate.sh` /
   `languages.yml` change, and **no rebuild** for a `.trivyignore` / `.hadolint.yaml`
   / `pins.yml` change (only the corresponding re-check lane).
4. Record `Outcome: SUCCESS` as a comment; the task closes only on success.

## 9. Deferred / open questions (ledger — nothing falls out silently)

- **Option B — PR-time build-validation.** Build (don't push) only the changed
  language on a PR, catching image breakage pre-merge. Deferred, not core:
  post-merge build failures are already caught (the `develop`-push build runs, and
  `vrg-finalize-pr`/`vrg-submit-pr` sanity-check post-merge runs and warn). Revisit
  if pre-merge breakage becomes a real pain.
- **D2 — shared runtime base for the runtime-heavy tools** (`node`+markdownlint,
  `python3`+uv tools, `pandoc`, `gh`, `semgrep`, mkdocs). Requires language images
  to `FROM` a shared base instead of the official upstream images. Considered only
  if Stage 0 shows those installs dominate build time; weigh against losing the
  official images' convenience and security cadence.
- **`common-tools` publication surface** (D1): published GHCR image (like `base`)
  vs. local-only build stage. Leaning published for CD reuse; settle in Stage D1.
- **Manifest schema finalization**: exact keys, and whether per-version overrides
  (a version-specific base image or build-arg) are needed now or deferred until a
  language demands one (YAGNI).
- **Stage C runner strategy**: GitHub-hosted `ubuntu-24.04-arm` vs. any self-hosted
  option; whether amd64 also benefits from a larger runner class.
- **First "add a language" proof**: adding a new language (e.g. the long-mooted
  Perl, or Swift per the C++ epic's LLVM setup) is the natural validation that the
  manifest achieved its goal — a *follow-on*, not scoped here.
