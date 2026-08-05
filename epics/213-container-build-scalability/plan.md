# Manifest-driven monorepo for scalable container builds — Implementation Plan

> **For epic workers:** These tasks are filed as **GitHub issues under epic
> vergil-project/.github#213** and implemented one PR each via
> `vergil:issue-implement` (which runs its own TDD cycle). Steps use checkbox
> (`- [ ]`) syntax. All code lands in **vergil-project/vergil-containers** unless
> noted.

**Goal:** Make container builds selective and cheap by deriving the build from a
committed `docker/languages.yml` manifest, so a change rebuilds only the images
whose bytes change while the nightly keeps everything CVE-fresh.

**Architecture:** A small Python package under `docker/matrix/` (mirroring the
existing `docker/pins/` idiom) becomes the single brain: it loads/validates the
manifest, derives each image's dependency graph by parsing `@include` directives,
and emits a build matrix — full (nightly/release/parity) or selective (develop-push
diff). `build.sh` and the CD workflow consume that matrix instead of hand-written
lists. Native arm64 runners and an optional shared `common-tools` layer are
orthogonal build-efficiency changes layered on top.

**Tech Stack:** Python 3 (stdlib + `pyyaml`, already in the dev image), Bash,
GitHub Actions, Docker Buildx. Tests: `pytest` colocated in `docker/matrix/`, run
in the dev container; an enforced `python3 docker/matrix/check_manifest.py` CI gate
mirrors `docker/pins/check_pins.py`.

## Global Constraints

- **No GHCR image name/tag changes.** Consumers resolve
  `ghcr.io/vergil-project/{prefix}-{lang}:{version}`; the C++ suffix-on-name
  convention (`prod-cpp-clang:20`, `prod-cpp-gcc:14`) is preserved verbatim.
- **Nightly rebuilds everything** (`no-cache`) for both `dev` and `prod` — never
  gate a language out of the nightly.
- **Release (`main`) rebuilds the full `prod` matrix**; selectivity is
  `develop`-push only.
- **A file is a rebuild input iff it changes image bytes** (spec §4.2 table);
  check-config files (`.trivyignore`, `.hadolint.yaml`, `docker/pins/**`,
  smoke-test) never trigger a rebuild.
- **No silent failures.** Every gate exits non-zero and prints a `::error::`
  annotation on drift (house style; matches `check_pins.py`).
- **Portable Bash, shellcheck-clean; Dockerfiles hadolint-clean.**
- **Validation is `vrg-container-run -- vrg-validate`** — the only validation entry
  point.
- **Git via `vrg-git`, PRs via the agent handoff** (`report-ready`; humans submit).

**Task dependency graph:**

```text
T0 (baseline) ─────────────────────────────────────────────┐ (gates T8)
T1 (manifest+gate) → T2 (includes) → T3 (matrix full+build.sh) → T4 (CD full parity)
                                                    └→ T5 (matrix selective) → T6 (CD selective+gate)
T6 → T7 (native arm64)
T0,(templates) → T8 (common-tools D1, gated on T0)
Validation (cold-rebuild) ── Blocked-by ── T4,T6,T7[,T8]
```

---

### Task 0 (Stage 0): Build-timing baseline harness

**Files:**
- Create: `docker/measure-build.sh`
- Create: `docs/build-timing-baseline.md` (committed baseline snapshot)

**Interfaces:**
- Produces: `docker/measure-build.sh [lang…]` — cold-builds the given images (all
  by default) via `generate.sh` + the resolved runtime, printing per-image
  wall-clock and writing a machine-readable `timings.tsv` (columns:
  `image	seconds`). Consumed by no code; its output seeds the baseline doc and
  the Stage D1 go/no-go.

- [ ] **Step 1: Write `docker/measure-build.sh`.** Reuse `build.sh`'s runtime
  detection (nerdctl/docker, `VRG_CONTAINER_RUNTIME`). For each requested lang +
  `base`: run `generate.sh <lang>`, then time a `--no-cache` single-arch build
  (`time` around the runtime `build`), appending `printf '%s\t%s\n' "$image"
  "$secs"` to `timings.tsv`. `set -euo pipefail`; shellcheck-clean.

- [ ] **Step 2: Verify it runs.** `vrg-container-run -- bash docker/measure-build.sh python`
  → prints a timing line and appends to `timings.tsv`. (Full run is expensive; one
  language proves the harness.)

- [ ] **Step 3: Capture the baseline.** Run the full harness once (locally or via a
  manual CD dispatch), paste the per-image table into
  `docs/build-timing-baseline.md` with the run's date, runtime, and host arch, plus
  a one-paragraph read of where time actually goes (esp. `node`/`gh`/`pandoc`/
  `semgrep` vs. the 9 COPY-able binaries) — this is the evidence that gates T8.

- [ ] **Step 4: Validate + commit.** `vrg-container-run -- vrg-validate`;
  `vrg-commit --type feat --scope build --message "add cold-build timing harness + baseline"`.

---

### Task 1 (Stage A): `languages.yml` manifest + loader/validator + CI gate

**Files:**
- Create: `docker/languages.yml`
- Create: `docker/matrix/manifest.py`
- Create: `docker/matrix/check_manifest.py`
- Create: `docker/matrix/test_manifest.py`
- Modify: `.github/workflows/ci.yml` (add a `manifest` gate job)

**Interfaces:**
- Produces:
  - `manifest.py`: `@dataclass(frozen=True) class Language(name: str, build_arg:
    str | None, versions: tuple[str, ...], context: str, smoke: str | None)` and
    `load(root: Path) -> list[Language]` (root = repo `docker/` dir; `base` is
    included as a `Language` with `build_arg=None`, `versions=()`).
  - `check_manifest.py`: `main(root: Path) -> int` — exits 1 with `::error::` if any
    `context`/`smoke` path is missing, any non-`base` language lacks `build_arg` or
    `versions`, or a manifest language has no matching `docker/<context>` dir.

- [ ] **Step 1: Write `docker/languages.yml`** enumerating today's exact set (spec
  §2.1): python 3.12/3.13/3.14, ruby 3.2/3.3/3.4, java 17/21, go 1.25/1.26, rust
  1.92/1.93, cpp-clang 19/20, cpp-gcc 13/14, each with its `build-arg`
  (`PYTHON_VERSION`, `RUBY_VERSION`, `JDK_VERSION`, `GO_VERSION`, `RUST_VERSION`,
  `CLANG_VERSION`, `GCC_VERSION`) and `context: docker/<lang>`; cpp entries add
  `smoke: docker/cpp/smoke-test.sh`. Add a top-level `base:` with `context:
  docker/base`.

- [ ] **Step 2: Write the failing test** `test_manifest.py`:

```python
from pathlib import Path
import manifest

DOCKER = Path(__file__).resolve().parent.parent

def test_loads_all_six_languages_plus_base():
    langs = {l.name: l for l in manifest.load(DOCKER)}
    assert set(langs) == {"python","ruby","java","go","rust","cpp-clang","cpp-gcc","base"}
    assert langs["python"].versions == ("3.12","3.13","3.14")
    assert langs["cpp-clang"].build_arg == "CLANG_VERSION"
    assert langs["cpp-clang"].smoke == "docker/cpp/smoke-test.sh"
    assert langs["base"].build_arg is None and langs["base"].versions == ()
```

- [ ] **Step 3: Run it, expect ImportError/FAIL.**
  `vrg-container-run -- python -m pytest docker/matrix/test_manifest.py -v`

- [ ] **Step 4: Implement `manifest.py`** (`from __future__ import annotations`,
  dataclass as above, `yaml.safe_load`, read `root / "languages.yml"`, map the
  `languages:` mapping to `Language` objects and append the `base:` entry). Follow
  the `docker/pins/` module style (top-level import, `Path`-based root).

- [ ] **Step 5: Run tests, expect PASS.**

- [ ] **Step 6: Write `check_manifest.py`** (`main(root)`; mirror
  `check_pins.py`: `::error::` + `return 1` on any missing path / missing
  `build_arg`/`versions` for non-base; `if __name__ == "__main__":
  sys.exit(main(Path(__file__).resolve().parent.parent))`). Add a consistency test
  asserting `main(DOCKER) == 0` on the real tree, and a negative test using a
  `tmp_path` manifest pointing at a nonexistent context → returns 1.

- [ ] **Step 7: Wire the CI gate** into `ci.yml` — add to the existing `pins` job
  (or a sibling `manifest` job on `prod-base`): `python3 docker/matrix/check_manifest.py`.

- [ ] **Step 8: Validate + commit.** `vrg-container-run -- vrg-validate`;
  `vrg-commit --type feat --scope build --message "add languages.yml manifest + consistency gate"`.

---

### Task 2 (Stage A): `@include` dependency-graph extractor

**Files:**
- Create: `docker/matrix/includes.py`
- Create: `docker/matrix/test_includes.py`

**Interfaces:**
- Consumes: nothing (reads templates from disk).
- Produces: `includes.py`:
  - `fragments_of(template: Path) -> set[str]` — parse `# @include common/<frag>`
    lines (same regex `generate.sh` honors: `^#\s*@include\s+(.+)$`) → set of
    fragment paths relative to `docker/` (e.g. `"common/validation-tools.dockerfile"`).
  - `dependency_map(root: Path) -> dict[str, set[str]]` — for every language in the
    manifest, map `name → {its context glob} ∪ {fragments its template includes}`
    as `docker/`-relative path globs (context dir as `docker/<lang>/**`).

- [ ] **Step 1: Write the failing test** `test_includes.py`:

```python
from pathlib import Path
import includes

DOCKER = Path(__file__).resolve().parent.parent

def test_python_includes_validation_tools_not_security_tools():
    frags = includes.fragments_of(DOCKER / "python" / "Dockerfile.template")
    assert "common/validation-tools.dockerfile" in frags
    assert "common/security-tools.dockerfile" not in frags  # python omits it

def test_dependency_map_scopes_security_tools_to_base_only():
    dm = includes.dependency_map(DOCKER)
    have_sec = {n for n, deps in dm.items()
                if "common/security-tools.dockerfile" in deps}
    assert "base" in have_sec and "python" not in have_sec
```

- [ ] **Step 2: Run it, expect FAIL.**
  `vrg-container-run -- python -m pytest docker/matrix/test_includes.py -v`

- [ ] **Step 3: Implement `includes.py`** — regex-parse the template lines
  (reuse the exact directive shape from `generate.sh`), and build `dependency_map`
  over `manifest.load(root)` (import `manifest`). Context glob = `f"{lang.context}/**"`
  relative to `root` → store as `docker/`-relative (`f"{lang.name}/**"` under
  `docker/`, i.e. the path the CD diff sees, e.g. `docker/python/**`).

- [ ] **Step 4: Run tests, expect PASS.**

- [ ] **Step 5: Validate + commit.** `vrg-commit --type feat --scope build
  --message "derive per-image @include dependency graph"`.

---

### Task 3 (Stage A): full-mode matrix generator + `build.sh` rewire

**Files:**
- Create: `docker/matrix/build_matrix.py`
- Create: `docker/matrix/test_build_matrix.py`
- Modify: `docker/build.sh`

**Interfaces:**
- Consumes: `manifest.load`, `includes.dependency_map`.
- Produces: `build_matrix.py`:
  - `full(root: Path) -> list[dict]` — one entry per (language, version):
    `{"language": name, "version": v, "build-arg": arg}`; `base` yields a single
    `{"language":"base","version":"latest","build-arg":None}`. Order stable
    (manifest order, then version order) so parity diffs are clean.
  - `as_json(entries) -> str` — `json.dumps({"include": entries})` for GitHub
    `fromJSON`.
  - CLI: `python3 build_matrix.py --mode full` prints the JSON (selective mode
    added in T5).

- [ ] **Step 1: Write the failing test** asserting `full()` reproduces today's 16
  variants + base, in order:

```python
from pathlib import Path
import build_matrix

DOCKER = Path(__file__).resolve().parent.parent

def test_full_matrix_matches_current_16_plus_base():
    entries = build_matrix.full(DOCKER)
    pairs = [(e["language"], e["version"]) for e in entries]
    assert pairs[:3] == [("python","3.12"),("python","3.13"),("python","3.14")]
    assert ("cpp-gcc","13") in pairs and ("cpp-gcc","14") in pairs
    assert ("base","latest") in pairs
    assert len([p for p in pairs if p[0] != "base"]) == 16
```

- [ ] **Step 2: Run it, expect FAIL.**

- [ ] **Step 3: Implement `build_matrix.py`** full mode + `as_json` + argparse CLI
  (`--mode {full}` for now; `--changed-file` added in T5).

- [ ] **Step 4: Run tests, expect PASS.**

- [ ] **Step 5: Rewire `docker/build.sh`** to iterate the manifest instead of the
  hand-written `build … ` list: read `build_matrix.py --mode full` (or import via a
  small `python3 -c`), and for each entry run `generate.sh <lang>` + the runtime
  build with `--build-arg <arg>=<version>` and tag `dev-<lang>:<version>`
  (`base` → `dev-base:latest`). Preserve the existing C++ smoke-test invocation by
  reading each entry's `smoke` (run it after that image builds). Keep the prune
  tail unchanged.

- [ ] **Step 6: Verify parity.** `bash docker/build.sh` builds exactly the same tag
  set as before (spot-check `docker images 'dev-*'` lists python 3.12/3.13/3.14 …
  cpp-gcc 13/14 + base). shellcheck-clean.

- [ ] **Step 7: Validate + commit.** `vrg-commit --type feat --scope build
  --message "generate full build matrix from manifest; build.sh consumes it"`.

---

### Task 4 (Stage A): CD + nightly consume the full manifest matrix (parity)

**Files:**
- Modify: `.github/workflows/cd-docker-publish.yml`
- Modify: `.github/workflows/ops.yml` (nightly stays full, now manifest-driven)

**Interfaces:**
- Consumes: `build_matrix.py --mode full`.
- Produces: a `setup` job output `matrix` (JSON) that `build-scan-push` consumes via
  `strategy.matrix: ${{ fromJSON(needs.setup.outputs.matrix) }}`.

- [ ] **Step 1: Add a `setup` job** to `cd-docker-publish.yml` (runs on
  `prod-base`, `needs: [hadolint]`): `run: python3 docker/matrix/build_matrix.py
  --mode full >> matrix.json` then `echo "matrix=$(cat matrix.json)" >>
  "$GITHUB_OUTPUT"`; declare `outputs.matrix`.

- [ ] **Step 2: Replace the hand-written `matrix.include`** in `build-scan-push`
  with `needs: [hadolint, setup]` and `strategy.matrix: ${{
  fromJSON(needs.setup.outputs.matrix) }}`, folding the standalone `publish-base` job
  into the matrix (deletes duplication). **`base` diverges from the language images
  and must be special-cased explicitly — a naive fold silently breaks its tag or
  cache:**
  - tag: `{prefix}-base:latest` + datestamp (the literal `latest` tag, *not* a
    coincidence of `version:"latest"`) — languages tag `{prefix}-{lang}:{version}`;
  - build-arg: **none** — guard the arg step so a null `build-arg` skips
    `--build-arg` entirely;
  - cache tag: `cache-latest` for base vs `cache-{version}` for languages — the
    cache-tag interpolation must resolve to `cache-latest` when `version == latest`.

- [ ] **Step 3: Parity check — languages *and* base.** In the PR, confirm the
  emitted matrix + the resulting **tag and cache-tag set** equals the old hand-written
  `build-scan-push` include list **and** the deleted `publish-base` job (attach
  `build_matrix.py --mode full` plus the derived tags to the PR; a reviewer diffs
  against both deleted blocks). Explicitly assert base still publishes
  `{prefix}-base:latest` with cache `cache-latest`. This is Stage A's
  behavior-preserving guarantee — **no image name/tag/cache/arch changes.**

- [ ] **Step 4: Point nightly at the manifest.** `ops.yml` already calls
  `cd-docker-publish.yml`; since that now derives the full matrix, nightly is
  automatically manifest-driven — confirm no `matrix.include` remains anywhere and
  both `dev`/`prod` nightly legs still pass `no-cache: true`.

- [ ] **Step 5: Add a `workflow_dispatch` matrix dry-run.** Because the `setup`
  wiring (`fromJSON`, and later the push-range diff) only fires on a real `develop`
  push, add a `workflow_dispatch` input `print-matrix-only: boolean` to
  `cd-docker-publish.yml`: when true, `setup` computes and echoes the matrix to the
  job summary and **no build runs**. This makes the plumbing testable pre-merge and
  on demand, instead of relying on watching the first develop-push. Verify by
  dispatching it and confirming the printed matrix equals `build_matrix.py --mode
  full`.

- [ ] **Step 6: Validate + report-ready.** `vrg-container-run -- vrg-validate`; hand
  off. (CD proper runs on develop, so note in the PR that the human should still
  watch the first post-merge develop-push; the dry-run from Step 5 de-risks it.)

---

### Task 5 (Stage B): selective-mode matrix (diff → images, via the graph)

**Files:**
- Modify: `docker/matrix/build_matrix.py`
- Modify: `docker/matrix/test_build_matrix.py`

**Interfaces:**
- Consumes: `includes.dependency_map`, plus a spec-§4.2 classification table.
- Produces:
  - `REBUILD_ALL_GLOBS = ("docker/generate.sh","docker/languages.yml",
    ".github/workflows/cd-docker-publish.yml")` and `CHECK_CONFIG_GLOBS`
    (`.trivyignore`, `trivy.yaml`, `.trivy/**`, `.hadolint.yaml`, `docker/pins/**`,
    `docker/cpp/smoke-test.sh`, `cliff.toml`, `vergil.toml`).
  - `selective(root: Path, changed: list[str]) -> list[dict]` — returns `full()` if
    any changed path matches `REBUILD_ALL_GLOBS` or `docker/base/**`; otherwise the
    union of languages whose `dependency_map` globs match a changed path (expanded to
    that language's version entries). Paths matching only `CHECK_CONFIG_GLOBS` (and
    nothing else) contribute **no** images. Returns `[]` when nothing matches.
  - CLI gains `--mode selective --changed-file <f>` (repeatable) / reads a newline
    list on stdin.

- [ ] **Step 1: Write failing tests** covering the classification matrix:

```python
def test_single_language_change_builds_only_that_language():
    m = build_matrix.selective(DOCKER, ["docker/cpp-clang/Dockerfile.template"])
    assert {e["language"] for e in m} == {"cpp-clang"}

def test_shared_fragment_fans_out_by_include_graph():
    m = build_matrix.selective(DOCKER, ["docker/common/validation-tools.dockerfile"])
    langs = {e["language"] for e in m}
    assert "python" in langs and "base" in langs   # both include it

def test_generate_sh_forces_full():
    assert build_matrix.selective(DOCKER, ["docker/generate.sh"]) == build_matrix.full(DOCKER)

def test_trivyignore_builds_nothing():
    assert build_matrix.selective(DOCKER, [".trivyignore"]) == []

def test_pins_yml_builds_nothing():
    assert build_matrix.selective(DOCKER, ["docker/pins/pins.yml"]) == []

def test_docs_only_change_builds_nothing():
    assert build_matrix.selective(DOCKER, ["README.md"]) == []
```

- [ ] **Step 2: Run, expect FAIL.**

- [ ] **Step 3: Implement `selective()`** + the glob constants + CLI extension.
  Use `fnmatch`/`pathlib.PurePath.match` for glob matching; a `common/` fragment
  maps to languages via the reverse of `dependency_map`.

- [ ] **Step 4: Run tests, expect PASS** (including the T3 full-mode tests, still
  green).

- [ ] **Step 5: Validate + commit.** `vrg-commit --type feat --scope build
  --message "selective build matrix from changed paths via include graph"`.

---

### Task 6 (Stage B): develop-push selectivity + aggregating build-gate

**Files:**
- Modify: `.github/workflows/cd-docker-publish.yml`
- Modify: `.github/workflows/cd.yml` (pass event context / ref into the reusable
  workflow so `setup` knows full-vs-selective)

**Interfaces:**
- Produces: a `build-gate` job (`needs: build-scan-push`, `if: always()`) that fails
  unless every scheduled build succeeded (passing when zero were scheduled) — the
  single required status check for branch protection.

- [ ] **Step 1: Teach `setup` the mode.** Add a workflow input
  `selective: boolean` (default false). `cd.yml` sets `selective: ${{ github.ref ==
  'refs/heads/develop' }}` (develop-push only; `main`/release and nightly stay
  full). When `selective`, `setup` computes changed files over the push range
  (`git diff --name-only ${{ github.event.before }} ${{ github.sha }}`) and runs
  `build_matrix.py --mode selective`; else `--mode full`.

- [ ] **Step 2: Handle the empty matrix.** If the matrix `include` is `[]`, the
  `build-scan-push` job is skipped by GitHub automatically; ensure `setup` emits
  valid empty JSON (`{"include":[]}`) and that downstream `if:` guards don't error.

- [ ] **Step 3: Add the `build-gate` job:**

```yaml
build-gate:
  needs: [build-scan-push]
  if: always()
  runs-on: ubuntu-latest
  steps:
    - name: Require all scheduled builds to have passed
      run: |
        r='${{ needs.build-scan-push.result }}'
        # success (some built) or skipped (none scheduled) are both green;
        # failure/cancelled are red.
        [ "$r" = "success" ] || [ "$r" = "skipped" ] || { echo "::error::build failed ($r)"; exit 1; }
```

- [ ] **Step 4: Repoint branch protection** (human step, note in PR): the required
  check becomes `build-gate`, not any per-language job, so a skipped language never
  blocks merge.

- [ ] **Step 5: Verify selectivity end-to-end** (PR notes for the human to watch the
  first two develop pushes after merge): a single-language change publishes only
  that language's dev images; a `common/` fragment change fans out per the graph; a
  `.trivyignore` change publishes nothing. Nightly/release still full.

- [ ] **Step 6: Validate + report-ready.**

---

### Task 7 (Stage C): native arm64 runners + manifest stitch

**Files:**
- Modify: `.github/workflows/cd-docker-publish.yml`

**Interfaces:** unchanged image names/tags; internal job graph reworked.

- [ ] **Step 1: Base-first proof.** Convert the `base` build to two per-arch jobs —
  `build-amd64` on `ubuntu-latest`, `build-arm64` on `ubuntu-24.04-arm` — each
  building `linux/<arch>` and pushing **by digest** to a temp tag, emitting its
  digest as an output. Drop QEMU (`setup-qemu-action`) from these legs. **Give each
  arch its own registry cache ref** — `cache-{version}-amd64` / `cache-{version}-arm64`
  (base: `cache-latest-{arch}`) — so the two native builds don't clobber a shared
  cache tag; preserve the existing `cache-from`/`cache-to` behavior per-arch.

- [ ] **Step 2: Stitch + scan + promote.** Add a job that
  `docker buildx imagetools create` the candidate multi-arch tag from the two
  digests, then runs the existing Trivy amd64/arm64 scans (arm64 now native on the
  arm runner, replacing the bespoke emulated `docker run`), then the existing
  promote-to-final + datestamp + **attest** — with the attestation `subject-digest`
  now the stitched manifest digest. Preserve the digest-preservation verify step.

- [ ] **Step 3: Roll to the language matrix.** Generalize the per-arch jobs across
  `fromJSON(needs.setup.outputs.matrix)` (matrix × 2 arches), keyed so each
  (language, version) stitches its own manifest. Keep `fail-fast: false`.

- [ ] **Step 4: Verify.** A CD run produces the same multi-arch tags (amd64+arm64
  manifest) as before, attestation present, Trivy still gating; confirm arm64 build
  time dropped vs. the Stage 0 baseline (record the delta in the PR).

- [ ] **Step 5: Validate + report-ready.** Note in PR: independently revertible to
  QEMU if arm runners misbehave.

---

### Task 8 (Stage D1, GATED on Task 0): shared `common-tools` layer

> **Gate:** implement only if `docs/build-timing-baseline.md` shows the 9 COPY-able
> binaries are a meaningful share of build time. If not, close this task as
> won't-do and record the number in the epic (the ledger's D2 remains the escalation
> for the runtime-heavy tools).

**Files:**
- Create: `docker/common-tools/Dockerfile.template`
- Modify: `docker/validation-tools.dockerfile` consumers → `COPY --from`
- Modify: `docker/build.sh`, `.github/workflows/cd-docker-publish.yml`
- Modify: `docker/languages.yml` (add `common-tools` pseudo-entry)

- [ ] **Step 1: Build `common-tools`** — a small `FROM debian:stable-slim` image
  that installs the 9 self-contained binaries (`shellcheck shfmt actionlint
  git-cliff hadolint scorecard trivy tofu nfpm`) into `/usr/local/bin`, multi-arch,
  published as `ghcr.io/vergil-project/{prefix}-common-tools:latest`. Reuse the
  existing per-arch download+checksum fragments (move them here).

- [ ] **Step 2: Replace the install fragments** in each consuming image's template
  with `FROM ghcr.io/vergil-project/{prefix}-common-tools:latest AS tools` +
  `COPY --from=tools /usr/local/bin/<tool> … /usr/local/bin/`. Keep the
  runtime-coupled fragments (`node`, `gh`, `pandoc`, `python-support`, semgrep,
  mkdocs) exactly as-is.

- [ ] **Step 3: Ordering.** In CD, `common-tools` becomes a `setup`-adjacent
  prerequisite the language jobs `needs`; in `build.sh`, build/pull `common-tools`
  before the language images (local parity).

- [ ] **Step 4: Verify** hadolint-clean, images still contain the tools
  (`vrg-container-run` a tool `--version`), no tag changes, arm+amd both stitched.
  Record the per-image build-time delta vs. the Stage 0 baseline.

- [ ] **Step 5: Validate + report-ready.**

---

### Operational task (filed at step 9): Cold-rebuild validation

`--kind validation`, `Blocked-by` T4, T6, T7 (and T8 if it lands). Runs the full
`no-cache` manifest-driven build of all variants + base for both arches; confirms
every image builds and each C++ smoke passes; confirms the produced GHCR name/tag
set is unchanged (spec §4.6); confirms selective scheduling on a representative
single-language change and full on a `generate.sh`/`languages.yml` change and
no-rebuild on a `.trivyignore`/`pins.yml` change. Records `Outcome: SUCCESS`.

## Self-Review

**Spec coverage:** Stage 0 → T0; §4.1 manifest + gate → T1; §4.2 include-graph → T2,
full matrix → T3/T4, selective + classification + build-gate → T5/T6; §4.4 native
arm → T7; §4.3 D1 → T8 (gated); §4.6 invariants → global constraints + Validation;
§8 validation → operational task; §9 D2/Option B → ledger (out of scope, correctly
unbuilt). No spec requirement is unmapped.

**Placeholder scan:** none — every code step carries real signatures/tests; YAML
tasks carry concrete snippets and explicit verification (parity diff, first
develop-push watch, timing deltas) appropriate to CI changes that aren't unit-TDD'd.

**Type consistency:** `manifest.Language(name, build_arg, versions, context,
smoke)` is used identically in T1/T2/T3; `build_matrix.full/selective/as_json` and
the glob constants are consistent across T3/T5/T6; `includes.fragments_of` /
`dependency_map` names match between T2 and T5.
