# Manifest-driven monorepo for scalable container builds — Implementation Plan

> **For epic workers:** These tasks are filed as **GitHub issues under epic
> vergil-project/.github#213** and implemented one PR each via
> `vergil:issue-implement` (which runs its own TDD cycle). Steps use checkbox
> (`- [ ]`) syntax. All code lands in **vergil-project/vergil-containers** unless
> noted.
>
> **Rebased 2026-10-26** (spec §0, issue vergil-project/.github#528). The
> original 2026-08-05 plan is in this file's git history. Tasks T0, T1, T3, T4
> and T8 are closed: T1, T3 and most of T4 were delivered by epic #407; T4's
> remainder is folded into T7; T0 and T8 were dropped (spec §0.3). T6 is
> retargeted to PR CI as T6′.

**Goal:** Remove QEMU emulation from the image publish by building each
architecture on its native runner. Then make PR CI build and scan (without
pushing) only the images a PR changes, so a broken image is caught before merge
rather than by post-merge CD.

**Architecture:** The manifest (`docker/versions/manifest.yml`) and its
generated publish matrix (`docker/versions/matrix.json`) already exist (from
epic #407). T7 reworks `cd-docker-publish.yml` into native per-architecture builds,
each pushed by digest, plus a stitch job that creates the multi-arch candidate.
The existing scan → gate → attest → promote steps then run unchanged on the
stitched candidate. `base` becomes a matrix entry. T2 and T5 add two small
modules to `docker/versions/`: an `@include` dependency graph and a selector
that maps a PR's changed paths and its `matrix.json` diff to matrix entries. T6′
wires the selector into PR CI on the same native per-arch runners, with one
aggregating gate.

**Tech Stack:** Python 3 (stdlib + `pyyaml`), Bash, GitHub Actions, Docker
Buildx. Tests are colocated `pytest` files in `docker/versions/`, run by
`scripts/bin/validate-custom` (PR CI's `quality / custom`).

## Global Constraints

- **No GHCR image name or tag changes.** Consumers resolve
  `ghcr.io/vergil-project/{prefix}-{family}:{version}` and
  `{prefix}-base:latest`, each with a `-YYYYMMDD` datestamp alias.
- **Post-merge CD stays full.** `develop` and `main` pushes publish the whole
  matrix; the nightly rebuilds everything with `no-cache` for both prefixes
  (human decision, spec §0.3.2). Selectivity is PR CI only, and PR CI never
  pushes images.
- **No emulation.** No `docker/setup-qemu-action`, no binfmt registration, and
  each build's `--platform` matches its runner's architecture. Runners:
  `ubuntu-24.04` for amd64 and `ubuntu-24.04-arm` for arm64, matching
  vergil-tooling's `lib/package/matrix.py` `RUNNERS`. (Follow-on guard:
  vergil-project/.github#527.)
- **No silent failures.** Every new gate or script exits non-zero with an
  `::error::` annotation, and an unclassifiable input fails closed by selecting
  the full matrix, never nothing.
- **Generated files stay generated.** Change `generate.py` and regenerate;
  never hand-edit a `BEGIN generated` block or `matrix.json`.
- **Portable Bash, shellcheck-clean; Dockerfiles hadolint-clean.**
- **Validation is `vrg-container-run -- vrg-validate`**, the only validation
  entry point.
- **Workflow pushes are human.** T7 and T6′ edit `.github/workflows/**`, which
  GitHub refuses from the agent. `report-ready` records the state anyway, and the
  human's `vrg-submit-pr` pushes the branch.

**Task dependency graph:**

```text
T7 (#515, native arm64 + base fold) ──────────────┐
T2 (#510, include graph) → T5 (#513, selector) ───┴→ T6′ (#514, PR-CI build+scan+gate)
Validation (#517) ── Blocked-by ── T7, T6′
Docs review (#497) → Retrospective (#215)
```

T7 and T2 can run in parallel from the start.

---

### Task T7 (#515, Stage C): native arm64 runners, manifest stitch, and `base` in the matrix

**Files:**

- Modify: `docker/versions/generate.py`, `docker/versions/test_generate.py`
  (emit `base` as a matrix entry)
- Regenerate: `docker/versions/matrix.json`
- Modify: `.github/workflows/cd-docker-publish.yml`

**Interfaces:**

- `matrix.json` gains a final entry
  `{"language": "base", "version": "latest", "build-arg": null}`. Its only
  consumer is `cd-docker-publish.yml`; `docker/build.sh` keeps its hand-written
  `base` lines.
- `cd-docker-publish.yml`'s `versions` job emits two outputs from `matrix.json`
  via `jq`: `matrix` (the entries, unchanged) and `arch-matrix` (each entry ×
  `{arch: amd64, runner: ubuntu-24.04}` and `{arch: arm64, runner:
  ubuntu-24.04-arm}`).

- [ ] **Step 1: Fold `base` into the generated matrix.** In `generate.py`,
  `render_matrix_json` appends the `base` entry (reuse the existing `BASE_KEY` /
  `BASE_VERSION` constants). Add a test that the entry is last, has a null
  `build-arg`, and is the only entry with one. Regenerate, then
  `python3 docker/versions/generate.py --check` passes.

- [ ] **Step 2: Per-arch build jobs.** Replace the QEMU build in
  `build-scan-push` with a `build` job over `fromJSON(needs.versions.outputs.arch-matrix)`
  (`fail-fast: false`, `runs-on: ${{ matrix.runner }}`):
  - delete `docker/setup-qemu-action`;
  - `docker/build-push-action` with `platforms: linux/${{ matrix.arch }}` and
    `outputs: type=image,name=ghcr.io/vergil-project/<prefix>-<family>,push-by-digest=true,name-canonical=true,push=true`;
  - pass `build-args` only when `matrix.build-arg` is non-null (`base` has none);
  - per-arch cache ref `cache-${version}-${arch}` (so base gets
    `cache-latest-${arch}`), with `cache-from`/`cache-to` and `no-cache` unchanged;
  - write the pushed digest to a file and upload it as an artifact named
    `digest-<prefix>-<family>-<version>-<arch>`.

- [ ] **Step 3: Stitch, then reuse the existing tail.** The publish job (renamed
  from `build-scan-push`, over `fromJSON(needs.versions.outputs.matrix)`,
  `needs: [build]`) downloads both digests and runs
  `docker buildx imagetools create -t "$CANDIDATE" <repo>@<amd64> <repo>@<arm64>`.
  The existing steps then run **unchanged** against the stitched candidate:
  Trivy amd64 and arm64 (both already scan the registry candidate and execute
  nothing, so they need no emulation), Trivy gate, candidate digest, provenance
  attestation (subject = the stitched index digest), datestamp, promote, verify
  digest. Delete the standalone `publish-base` job.

- [ ] **Step 4: Keep `base`'s observable names.** The job name
  `publish: <prefix>-base:latest` and the tags `<prefix>-base:latest` and
  `-YYYYMMDD` come out unchanged from `version: latest`. The SARIF categories do
  not: base's are `trivy-image-<prefix>-base-<arch>`, not
  `...-base-latest-<arch>`. Use an expression that drops the version for base,
  so the code-scanning alert history keeps its category.

- [ ] **Step 5: Check registry retention.** `package-cleanup.yml` deletes
  untagged versions. Confirm the per-arch digests the stitched index references
  are protected the same way today's multi-arch child manifests are, and that a
  digest pushed but not yet stitched cannot be reaped mid-run. If either is
  unclear, add the exclusion and a test. Note in the PR that the old shared
  `cache-<version>` tags become unused; the human may delete them.

- [ ] **Step 6: Verify.** On a manual `workflow_dispatch` of CD (or the first
  post-merge `develop` push): every image publishes the same tag set as before,
  each final tag is a two-platform index (`docker buildx imagetools inspect`),
  attestations are present, both Trivy scans gate, and no job logs a QEMU setup.
  Record the per-family build-step times against the spec §0.2 baseline in the PR
  (python 384s, go 1792s, rust 3013s), and again from the first nightly after
  merge.

- [ ] **Step 7: Validate and report-ready.** `vrg-container-run -- vrg-validate`.
  Note in the PR that the change can be reverted to the QEMU build independently
  of the other tasks.

---

### Task T2 (#510, Stage B): `@include` dependency graph

**Files:**

- Create: `docker/versions/includes.py`
- Create: `docker/versions/test_includes.py`

**Interfaces:**

- `fragments_of(template: Path) -> frozenset[str]` parses `# @include <path>`
  lines with the same directive shape `docker/generate.sh` honours, and returns
  repo-relative paths (`"docker/common/validation-tools.dockerfile"`). It raises
  `IncludeError` naming the template and line when an include does not resolve
  to an existing file.
- `dependency_map(root: Path) -> dict[str, frozenset[str]]` maps every manifest
  family key, plus `"base"`, to its inputs as repo-relative globs: its
  `docker/<key>/**` context, every fragment its template includes, and its smoke
  directory when one exists (`docker/<key>/` or `docker/<language>/` holding
  `smoke-test.sh`, the same rule `docker/build.sh` uses; for example
  `docker/cpp/**` for both C++ families). Families come from
  `manifest.load_manifest`.

- [ ] **Step 1: Write the failing tests** against the real tree:

```python
from pathlib import Path
import includes

ROOT = Path(__file__).resolve().parent.parent.parent

def test_java_includes_osv_scanner_python_does_not():
    assert "docker/common/osv-scanner.dockerfile" in includes.fragments_of(
        ROOT / "docker/java/Dockerfile.template")
    assert "docker/common/osv-scanner.dockerfile" not in includes.fragments_of(
        ROOT / "docker/python/Dockerfile.template")

def test_security_tools_scoped_to_base():
    dm = includes.dependency_map(ROOT)
    assert {k for k, v in dm.items()
            if "docker/common/security-tools.dockerfile" in v} == {"base"}

def test_cpp_families_depend_on_shared_smoke_dir():
    dm = includes.dependency_map(ROOT)
    assert "docker/cpp/**" in dm["cpp-clang"] and "docker/cpp/**" in dm["cpp-gcc"]

def test_every_family_and_base_present():
    assert set(includes.dependency_map(ROOT)) == {
        "python", "java", "go", "rust", "cpp-clang", "cpp-gcc", "ts-node", "base"}
```

  Add a `tmp_path` test: a template that includes a missing fragment raises
  `IncludeError`.

- [ ] **Step 2: Run them and expect failure** (`vrg-container-run -- vrg-validate`,
  or pytest from `docker/versions/` inside the container).

- [ ] **Step 3: Implement `includes.py`** in the `docker/versions/` module style
  (`from __future__ import annotations`, `import manifest as mf`, a
  `ValueError` subclass for errors, no language branches).

- [ ] **Step 4: Run tests and expect them to pass. Validate, commit, report-ready.**
  `vrg-commit --type feat --scope versions --message "derive each image's @include dependency graph"`.

---

### Task T5 (#513, Stage B): selector from changed paths and the matrix diff

**Files:**

- Create: `docker/versions/selection.py` (not `select.py`, which would shadow
  the stdlib module)
- Create: `docker/versions/test_selection.py`

**Interfaces:**

- `select_entries(root: Path, changed: Iterable[str], base_matrix: dict,
  head_matrix: dict) -> list[dict]` returns the `head_matrix` entries to build,
  in `head_matrix` order. It is the union of:
  1. **Full set:** any changed path matching `FULL_GLOBS`, which covers inputs
     that touch every image or the build-and-scan pipeline itself:
     `docker/generate.sh`, `.github/workflows/cd-docker-publish.yml`,
     `.github/workflows/ci.yml`, `scripts/bin/trivy-*`, `.trivyignore`,
     `trivy.yaml`. In PR CI a scan-config change is re-proved by scanning
     every image; with a warm cache that costs little.
  2. **Matrix diff:** every `head_matrix` entry that is absent from, or differs
     from, `base_matrix`. This covers lifecycle add PRs and build-arg changes; a
     dropped entry selects nothing.
  3. **Graph:** every family whose `includes.dependency_map` globs match a
     changed path (all of that family's entries).
- Paths matching `NO_BUILD_GLOBS` contribute nothing: `docs/**`, `*.md`,
  `docker/pins/**`, `docker/versions/*.py`, `docker/versions/test*/**`,
  `docker/versions/manifest.yml` (its image effects arrive through the matrix
  diff and the regenerated templates), `docker/versions/window.json`,
  `docker/build.sh`, `.hadolint.yaml`, `cliff*.toml`, `vergil.toml`, other
  `.github/**` and `scripts/**` files. Each of these is either not an image
  input or is re-checked by an existing CI job.
- **Fail closed:** a changed path under `docker/` that matches none of the
  above selects the full set and names the path in a `::warning::`.
- CLI: `python3 docker/versions/selection.py --base-matrix <file>
  --head-matrix <file> --changed-files <file>` prints `{"include": [...]}`; an
  empty selection prints `{"include": []}`.

- [ ] **Step 1: Write failing tests** covering each rule:

```python
def test_single_family_template_selects_that_family(): ...  # docker/rust/Dockerfile.template -> rust entries only
def test_java_only_fragment_selects_java(): ...             # docker/common/osv-scanner.dockerfile -> java
def test_shared_fragment_fans_out_by_graph(): ...           # validation-tools.dockerfile -> every family + base
def test_lifecycle_add_selects_new_entry(): ...             # head matrix gains python 3.16 -> it is selected
def test_dropped_entry_selects_nothing(): ...
def test_trivyignore_selects_full(): ...
def test_docs_and_pins_select_nothing(): ...                 # README.md, docker/pins/pins.yml -> []
def test_unknown_docker_path_fails_closed_to_full(): ...    # docker/new-thing/x -> full set
def test_cpp_smoke_change_selects_both_cpp_families(): ...  # docker/cpp/smoke-test.sh
```

- [ ] **Step 2: Run them and expect failure.**

- [ ] **Step 3: Implement `selection.py`.** Match globs with `fnmatch` on
  repo-relative POSIX paths.

- [ ] **Step 4: Run tests and expect them to pass (T2's too). Validate, commit,
  report-ready.** `vrg-commit --type feat --scope versions --message "select the
  images a change affects from its paths and matrix diff"`.

---

### Task T6′ (#514, Stage B): selective image build and scan in PR CI

Blocked-by T5 and T7 (reuses T7's per-arch runner matrix and build shape).

**Files:**

- Modify: `.github/workflows/ci.yml` (or a new repo-local reusable workflow it
  calls, if that keeps `ci.yml` readable)

**Interfaces:**

- `images / select`: checks out with full history, writes the changed-file list
  (`git diff --name-only <base-sha>...<head-sha>`) and the base
  `matrix.json` (`git show <base-sha>:docker/versions/matrix.json`), runs
  `selection.py`, and outputs `matrix` and `arch-matrix` (expanded × arch the
  same way as T7).
- `images / build`: over `arch-matrix` on native runners. It builds one
  platform with `load: true` and **no push**, with `cache-from` the published
  per-arch dev cache (read-only, no `cache-to`; public GHCR reads need no login,
  so fork PRs work), then:
  - scans the local image with `scripts/bin/trivy-image-scan` and gates it with
    `scripts/bin/trivy-gate` (same suppressions and thresholds as publish);
  - runs the family's `smoke-test.sh` when one exists (`docker/<family>/` or
    `docker/<language>/`). Smoke tests run in CI here for the first time; they
    only ran in local `build.sh` until now.
- `images / evidence`: `needs: images / build`, `if: always()`. It passes when
  the build result is `success` or `skipped` (nothing selected) and fails, with
  `::error::`, otherwise.

- [ ] **Step 1: Add the three jobs** and the empty-selection path: an empty
  `include` skips `images / build` cleanly and `images / evidence` passes.

- [ ] **Step 2: Required check (human step, noted in the PR).** Add
  `images / evidence` to the branch ruleset's required checks, wherever this
  repo's ruleset is managed.

- [ ] **Step 3: Verify on real PRs** (recorded in the PR body): a docs-only PR
  selects nothing and `images / evidence` is green; a one-family template change
  builds only that family, natively on both arches; a `.trivyignore` change
  builds and scans everything.

- [ ] **Step 4: Validate and report-ready.**

---

### Validation (#517): cold rebuild and selection check

`--kind validation`, Blocked-by T7 and T6′. Run the full `no-cache` publish of
every matrix entry (now including `base`) for both prefixes on native runners.
Confirm:

1. every image builds, scans and publishes; each final tag is a two-platform
   index with a provenance attestation, and the published GHCR name/tag set is
   unchanged (spec §4.6);
2. no workflow run in the repo sets up QEMU or registers binfmt;
3. nightly wall-clock and Go/Rust build-step times have dropped against the spec
   §0.2 baseline (record the numbers);
4. PR CI selects as specified for a representative one-family change, a
   lifecycle-style matrix addition, a shared `common/` fragment change, a
   `.trivyignore` change and a docs-only change, and `images / evidence` goes red
   when a selected image fails to build (a throwaway PR is fine).

Record `Outcome: SUCCESS` as a comment.

## Self-Review

**Spec coverage:** spec §0.3.1 (native arm64 first + `base` fold) → T7;
§0.3.2/§0.3.3 (PR-CI selectivity, matrix-diff selection, aggregating gate) → T2,
T5, T6′; §4.2's derived include graph → T2; §4.2's classification table, updated
for PR scanning and fail-closed → T5; §4.4 → T7; §4.6 invariants → Global
Constraints + Validation; §0.3.4/§0.3.5 (Stage 0 and D1 dropped) → closed
issues; the ledger's Option B → T6′. Dropped by human decision: develop-push
selectivity.

**Type consistency:** `includes.fragments_of` / `dependency_map` (T2) are what
`selection.select_entries` (T5) consumes; the `matrix.json` entry shape
(`language`, `version`, `build-arg`) is shared by T7, T5 and T6′; the per-arch
expansion (`arch`, `runner`) is the same in T7 and T6′.
