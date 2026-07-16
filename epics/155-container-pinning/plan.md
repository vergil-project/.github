# Container Pinning & Version Management — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Make container tool-version pinning managed, not permanent — a generated + justified pin catalog with a CI gate, immutable datestamp image aliases with sliding-window repoint rollback, and an internal-state observability report that flags pins due for re-evaluation.

**Architecture:** Three workstreams over vergil-containers. WS2 turns pins into data (`docker/pins/pins.yml`) with a generator and a CI gate so no pin merges undocumented. WS3 makes each build an addressable artifact (`{prefix}-{lang}:{version}-YYYYMMDD`) reaped on a sliding window (prod 30d / dev 7d) with a repoint rollback. WS4 reports installed-versions-vs-pins and surfaces the "due for re-evaluation" list. WS1 (nightly branch coupling) is out of scope here — tracked separately as vergil-containers#413.

**Tech Stack:** Bash (matching `docker/generate.sh`), Python 3 (available in `dev-base`), GitHub Actions YAML, `dataaxiom/ghcr-cleanup-action`, `docker buildx imagetools`, `yq`/PyYAML for `pins.yml`.

## Global Constraints

- All git ops via `vrg-git` / commits via `vrg-commit` (never raw `git`); all GitHub ops via `vrg-gh`. Work inside a worktree under `.worktrees/issue-<N>-<slug>/` on `feature/<N>-<slug>`.
- Only validation command: `vrg-container-run -- vrg-validate`. Do not invoke linters directly.
- Dockerfiles are generated: edit `*/Dockerfile.template` and `common/*.dockerfile`, never the generated `Dockerfile`. Run `docker/generate.sh` before hadolint.
- Pinning doctrine (spec §3): default unpinned; least-specific pin; every EXACT pin (`x.y.z`) carries a `pins.yml` justification with an `inducing_release`; major-only pins (language matrix, `NODE_MAJOR`) are the intentional matrix and are exempt from `pins.yml`.
- Image tag scheme is fixed: rolling `{prefix}-{lang}:{version}` (unchanged) + immutable `{prefix}-{lang}:{version}-YYYYMMDD`.
- Retention windows are fixed: **prod = 30 days, dev = 7 days**.
- Do NOT build upstream-distance drift checking (delegated to the Dependabot follow-on epic #158). WS4 is internal state only.

---

## File Structure

- `docker/pins/pins.yml` — keyed justifications (source of truth for *why*). WS2.
- `docker/pins/extract_pins.py` — scan templates+fragments, emit the list of EXACT pins found in the images. WS2.
- `docker/pins/check_pins.py` — CI gate: every extracted EXACT pin has a `pins.yml` entry; every `pins.yml` entry still exists in the images. WS2.
- `docker/pins/generate_catalog.py` — join extracted pins + `pins.yml` → `docker/pins/CATALOG.md`. WS2.
- `docker/pins/CATALOG.md` — generated human-readable catalog (committed). WS2.
- `docker/pins/report_exposure.py` — WS4 observability report (installed vs pinned, due-for-re-evaluation).
- `.github/workflows/cd-docker-publish.yml` — MODIFY: publish the datestamp alias. WS3.
- `.github/workflows/package-cleanup.yml` — MODIFY: add sliding-window reap of `*-YYYYMMDD` aliases. WS3.
- `.github/workflows/ci.yml` — MODIFY: add the `pins` gate job. WS2.
- `docs/` (site) — MODIFY under the docs-review gate (#414), not here.

## Task dependency graph (for epic-create step 9 task filing)

```
Task 1 (pins.yml + extract + check gate)
  └─ Task 2 (generator + CATALOG.md)
       ├─ Task 3 (audit + free LOUD-failure pins)
       └─ Task 8 (WS4 exposure report)
Task 4 (datestamp aliases)  ─┬─ Task 5 (sliding-window reaper)
                             └─ Task 6 (repoint rollback)
Task 7 (free SILENT/fleet-gating pins)  ── blocked-by Task 6 AND Task 3
```
Filed tasks: T1→T2→T3 sequential; T4→{T5,T6} sequential; T7 blocked-by T6+T3; T8 blocked-by T2. WS2-catalog (T1–T2) and WS3 (T4–T6) proceed in parallel.

---

### Task 1: Pin data model + extractor + CI gate

**Files:**
- Create: `docker/pins/pins.yml`
- Create: `docker/pins/extract_pins.py`
- Create: `docker/pins/check_pins.py`
- Create: `docker/pins/test_pins.py`
- Modify: `.github/workflows/ci.yml` (add `pins` job)

**Interfaces:**
- Produces: `extract_pins.extract(root: Path) -> list[Pin]` where `Pin` is a dataclass `{tool: str, version: str, source: str}` (EXACT `x.y.z` pins only). `check_pins.main(root) -> int` (0 ok, 1 on any undocumented pin or stale entry). `pins.yml` shape: `pins: {<tool>: {constraint, inducing_release, deterministic, reason, state, tracking_issue}}`.

- [ ] **Step 1: Write the failing test for the extractor**

```python
# docker/pins/test_pins.py
from pathlib import Path
import extract_pins, check_pins

DOCKER = Path(__file__).resolve().parent.parent

def test_extractor_finds_known_exact_pins():
    tools = {p.tool for p in extract_pins.extract(DOCKER)}
    # exact-version pins that exist today (spec Appendix A)
    assert "shellcheck" in tools
    assert "trivy" in tools
    assert "uv" in tools
    assert "golangci-lint" in tools

def test_extractor_excludes_major_only_matrix():
    tools = {p.tool for p in extract_pins.extract(DOCKER)}
    # NODE_MAJOR=22 and language matrix are least-specific, not exact pins
    assert "node" not in tools
```

- [ ] **Step 2: Run it, verify it fails**

Run: `cd docker/pins && python3 -m pytest test_pins.py -k extractor -v`
Expected: FAIL — `ModuleNotFoundError: extract_pins`.

- [ ] **Step 3: Implement the extractor**

```python
# docker/pins/extract_pins.py
"""Scan Dockerfile templates + fragments for EXACT (x.y.z) version pins.
Major-only pins (NODE_MAJOR, language matrix ARGs) are intentional matrix and
are deliberately NOT reported."""
from __future__ import annotations
import re
from dataclasses import dataclass
from pathlib import Path

@dataclass(frozen=True)
class Pin:
    tool: str
    version: str
    source: str

# (regex, tool-name-group, version-group). Each targets one install idiom.
_PATTERNS = [
    # ARG SHELLCHECK_VERSION=0.11.0  (only x.y.z — three numeric components)
    (re.compile(r"ARG\s+([A-Z0-9_]+)_VERSION=(\d+\.\d+\.\d+)\b"), 1, 2),
    # pip/uv:  yamllint==1.38.0 ,  uv tool install ansible-lint==26.4.0
    (re.compile(r"([a-z0-9][a-z0-9._-]+)==(\d+\.\d+\.\d+)\b"), 1, 2),
    # go install ...tool@v2.12.2  /  ...govulncheck@v1.3.0
    (re.compile(r"/([a-zA-Z0-9._-]+)(?:/v\d+)?/[a-zA-Z0-9._-]*@v(\d+\.\d+\.\d+)\b"), 1, 2),
    # cargo install cargo-deny@0.19.6
    (re.compile(r"cargo install\s+([a-z0-9-]+)@(\d+\.\d+\.\d+)\b"), 1, 2),
    # npm install -g markdownlint-cli@0.48.0
    (re.compile(r"npm install -g\s+([a-z0-9@/-]+)@(\d+\.\d+\.\d+)\b"), 1, 2),
]

def _normalize(tool: str) -> str:
    return tool.lower().replace("_", "-").removesuffix("-version")

def extract(root: Path) -> list[Pin]:
    files = list((root / "common").glob("*.dockerfile"))
    files += [p / "Dockerfile.template" for p in root.iterdir()
              if (p / "Dockerfile.template").exists()]
    seen: dict[tuple[str, str], Pin] = {}
    for f in files:
        text = f.read_text()
        for rx, tg, vg in _PATTERNS:
            for m in rx.finditer(text):
                tool = _normalize(m.group(tg))
                ver = m.group(vg)
                seen.setdefault((tool, ver), Pin(tool, ver, str(f.relative_to(root))))
    return sorted(seen.values(), key=lambda p: p.tool)
```

- [ ] **Step 4: Run the extractor tests, verify pass**

Run: `cd docker/pins && python3 -m pytest test_pins.py -k extractor -v`
Expected: PASS (both extractor tests).

- [ ] **Step 5: Write the failing test for the check gate**

```python
# append to docker/pins/test_pins.py
def test_check_passes_when_every_pin_documented(tmp_path, monkeypatch):
    assert check_pins.main(DOCKER) == 0  # after Step 7 pins.yml is complete

def test_check_fails_on_undocumented_pin(tmp_path):
    # a fixture image dir with a pin but empty pins.yml
    (tmp_path / "common").mkdir()
    (tmp_path / "common" / "x.dockerfile").write_text("ARG FOO_VERSION=1.2.3\n")
    (tmp_path / "pins.yml").write_text("pins: {}\n")
    assert check_pins.main(tmp_path) == 1
```

- [ ] **Step 6: Run it, verify the undocumented-pin test fails (no module yet)**

Run: `cd docker/pins && python3 -m pytest test_pins.py -k check -v`
Expected: FAIL — `ModuleNotFoundError: check_pins`.

- [ ] **Step 7: Implement the check gate and seed `pins.yml` with every current exact pin**

```python
# docker/pins/check_pins.py
"""CI gate: every EXACT image pin has a pins.yml justification, and every
pins.yml entry still corresponds to a real pin. Exit 1 (loud) on drift."""
from __future__ import annotations
import sys
from pathlib import Path
import yaml
import extract_pins

def load_pins(root: Path) -> dict:
    data = yaml.safe_load((root / "pins.yml").read_text()) or {}
    return data.get("pins", {})

def main(root: Path) -> int:
    pinned = {p.tool for p in extract_pins.extract(root)}
    documented = set(load_pins(root))
    undocumented = sorted(pinned - documented)
    stale = sorted(documented - pinned)
    for t in undocumented:
        print(f"::error::pin '{t}' has no justification in pins.yml", file=sys.stderr)
    for t in stale:
        print(f"::error::pins.yml entry '{t}' no longer pins anything; remove it", file=sys.stderr)
    return 1 if (undocumented or stale) else 0

if __name__ == "__main__":
    sys.exit(main(Path(__file__).resolve().parent))
```

```yaml
# docker/pins/pins.yml — justification for every EXACT (x.y.z) image pin.
# Floating tools need no entry. A pin here with no matching image pin, or an
# image pin with no entry here, fails CI (check_pins.py).
#
# Schema per tool:
#   constraint:       the enforced version expression
#   inducing_release: upstream release whose problem caused the pin (null if seed/unknown)
#   deterministic:    is the problem reproducible/testable? (bool)
#   reason:           prose justification
#   state:            active | under-evaluation | freed
#   tracking_issue:   issue ref for under-evaluation pins (else null)
pins:
  go-test-coverage:
    constraint: "==2.18.3 on Go 1.25, else ==2.18.8"
    inducing_release: "2.18.4"
    deterministic: true
    reason: "go-test-coverage >=2.18.4 requires Go >=1.26; fails on the Go 1.25 image."
    state: active
    tracking_issue: null
  # SEED ENTRIES — every other current exact pin, pending Task 3 audit.
  # Each starts inducing_release: null / reason: 'seed — unaudited' so the gate
  # passes; Task 3 replaces each with a real justification OR frees the pin.
  markdownlint-cli: {constraint: "==0.48.0", inducing_release: null, deterministic: false, reason: "seed — unaudited", state: active, tracking_issue: null}
  shellcheck:       {constraint: "==0.11.0", inducing_release: null, deterministic: false, reason: "seed — unaudited", state: active, tracking_issue: null}
  shfmt:            {constraint: "==3.13.1", inducing_release: null, deterministic: false, reason: "seed — unaudited", state: active, tracking_issue: null}
  actionlint:       {constraint: "==1.7.12", inducing_release: null, deterministic: false, reason: "seed — held pending upstream client-id support (vergil-containers#260)", state: active, tracking_issue: "vergil-project/vergil-containers#260"}
  git-cliff:        {constraint: "==2.13.1", inducing_release: null, deterministic: false, reason: "seed — unaudited", state: active, tracking_issue: null}
  hadolint:         {constraint: "==2.14.0", inducing_release: null, deterministic: false, reason: "seed — unaudited", state: active, tracking_issue: null}
  scorecard:        {constraint: "==5.5.0",  inducing_release: null, deterministic: false, reason: "seed — unaudited", state: active, tracking_issue: null}
  trivy:            {constraint: "==0.70.0", inducing_release: null, deterministic: false, reason: "seed — unaudited", state: active, tracking_issue: null}
  yamllint:         {constraint: "==1.38.0", inducing_release: null, deterministic: false, reason: "seed — unaudited", state: active, tracking_issue: null}
  ansible-lint:     {constraint: "==26.4.0", inducing_release: null, deterministic: false, reason: "seed — unaudited", state: active, tracking_issue: null}
  uv:               {constraint: "==0.11.14", inducing_release: null, deterministic: false, reason: "seed — unaudited", state: active, tracking_issue: null}
  opentofu:         {constraint: "==1.12.3", inducing_release: null, deterministic: false, reason: "seed — unaudited", state: active, tracking_issue: null}
  nfpm:             {constraint: "==2.47.0", inducing_release: null, deterministic: false, reason: "seed — unaudited", state: active, tracking_issue: null}
  mkdocs-material:  {constraint: "==9.7.6",  inducing_release: null, deterministic: false, reason: "seed — unaudited", state: active, tracking_issue: null}
  mike:             {constraint: "==2.2.0",  inducing_release: null, deterministic: false, reason: "seed — unaudited", state: active, tracking_issue: null}
  pyyaml:           {constraint: "==6.0.3",  inducing_release: null, deterministic: false, reason: "seed — suspected belt-and-suspenders; audit for freeing", state: active, tracking_issue: null}
  golangci-lint:    {constraint: "==2.12.2", inducing_release: null, deterministic: false, reason: "seed — unaudited", state: active, tracking_issue: null}
  govulncheck:      {constraint: "==1.3.0",  inducing_release: null, deterministic: false, reason: "seed — unaudited", state: active, tracking_issue: null}
  go-licenses:      {constraint: "==2.0.1",  inducing_release: null, deterministic: false, reason: "seed — unaudited", state: active, tracking_issue: null}
  gocyclo:          {constraint: "==0.6.0",  inducing_release: null, deterministic: false, reason: "seed — unaudited", state: active, tracking_issue: null}
  goimports:        {constraint: "==0.45.0", inducing_release: null, deterministic: false, reason: "seed — unaudited", state: active, tracking_issue: null}
  cargo-deny:       {constraint: "==0.19.6", inducing_release: null, deterministic: false, reason: "seed — unaudited", state: active, tracking_issue: null}
  cargo-llvm-cov:   {constraint: "==0.8.6",  inducing_release: null, deterministic: false, reason: "seed — unaudited", state: active, tracking_issue: null}
```

Note: the seed entries make the gate pass immediately (documented, if unaudited). Task 3 is what replaces `seed — unaudited` with real audit outcomes (justify with an `inducing_release`, or free the pin and delete both the Dockerfile pin and the entry).

- [ ] **Step 8: Run all pin tests, verify pass**

Run: `cd docker/pins && python3 -m pytest test_pins.py -v`
Expected: PASS (extractor + check tests). If `test_check_passes...` fails, a real pin is missing from the seed — add it.

- [ ] **Step 9: Add the CI gate job**

```yaml
# .github/workflows/ci.yml — add after the `hadolint` job
  pins:
    name: "quality / pins"
    runs-on: ubuntu-latest
    container:
      image: ghcr.io/vergil-project/prod-base:latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v6
      - name: Every exact pin is justified
        run: python3 docker/pins/check_pins.py
```

- [ ] **Step 10: Validate and commit**

Run: `vrg-container-run -- vrg-validate`
Then:
```bash
vrg-commit --type feat --scope pins --message "add pin data model, extractor, and CI justification gate" \
  --body "pins.yml carries a justification per exact pin; check_pins.py fails CI on any undocumented pin or stale entry. Seeds every current exact pin (Task 3 audits them)."
```

---

### Task 2: Catalog generator

**Files:**
- Create: `docker/pins/generate_catalog.py`
- Create: `docker/pins/CATALOG.md` (generated, committed)
- Modify: `docker/pins/test_pins.py`
- Modify: `.github/workflows/ci.yml` (assert CATALOG.md is up to date)

**Interfaces:**
- Consumes: `extract_pins.extract`, `pins.yml`.
- Produces: `generate_catalog.render(root) -> str` (markdown). CLI writes `CATALOG.md`; `--check` exits 1 if the committed file is stale.

- [ ] **Step 1: Write the failing test**

```python
# append to docker/pins/test_pins.py
import generate_catalog
def test_catalog_lists_every_documented_pin():
    md = generate_catalog.render(DOCKER)
    assert "| go-test-coverage |" in md
    assert "inducing" in md.lower()
def test_catalog_check_matches_committed_file():
    assert generate_catalog.render(DOCKER) == (DOCKER / "pins" / "CATALOG.md").read_text()
```

- [ ] **Step 2: Run it, verify fail**

Run: `cd docker/pins && python3 -m pytest test_pins.py -k catalog -v`
Expected: FAIL — `ModuleNotFoundError: generate_catalog`.

- [ ] **Step 3: Implement the generator**

```python
# docker/pins/generate_catalog.py
"""Join extracted image pins with pins.yml into a human-readable catalog.
Version facts come from the Dockerfiles (single source of truth); justifications
come from pins.yml. Run without args to write CATALOG.md; --check to verify it."""
from __future__ import annotations
import sys
from pathlib import Path
import check_pins, extract_pins

def render(root: Path) -> str:
    pins = check_pins.load_pins(root)
    rows = []
    for p in extract_pins.extract(root):
        meta = pins.get(p.tool, {})
        rows.append(
            f"| {p.tool} | {p.version} | {meta.get('state','?')} | "
            f"{meta.get('inducing_release') or '—'} | "
            f"{'yes' if meta.get('deterministic') else 'no'} | "
            f"{meta.get('reason','—')} |"
        )
    body = "\n".join(rows)
    return (
        "# Pin catalog (generated — do not edit)\n\n"
        "Generated by `docker/pins/generate_catalog.py` from the Dockerfiles + "
        "`pins.yml`. Run `docker/pins/generate_catalog.py` to refresh.\n\n"
        "| tool | version | state | inducing release | deterministic | reason |\n"
        "| --- | --- | --- | --- | --- | --- |\n"
        f"{body}\n"
    )

def main(argv: list[str]) -> int:
    root = Path(__file__).resolve().parent
    out = root / "CATALOG.md"
    md = render(root)
    if "--check" in argv:
        if out.read_text() != md:
            print("::error::CATALOG.md is stale; run docker/pins/generate_catalog.py", file=sys.stderr)
            return 1
        return 0
    out.write_text(md)
    return 0

if __name__ == "__main__":
    sys.exit(main(sys.argv[1:]))
```

- [ ] **Step 4: Generate the catalog and run tests**

Run: `cd docker/pins && python3 generate_catalog.py && python3 -m pytest test_pins.py -k catalog -v`
Expected: `CATALOG.md` created; both catalog tests PASS.

- [ ] **Step 5: Add the staleness check to CI**

```yaml
# .github/workflows/ci.yml — add a step to the `pins` job, after the check step
      - name: CATALOG.md is up to date
        run: python3 docker/pins/generate_catalog.py --check
```

- [ ] **Step 6: Validate and commit**

Run: `vrg-container-run -- vrg-validate`
```bash
vrg-commit --type feat --scope pins --message "generate the pin catalog from Dockerfiles + pins.yml" \
  --body "CATALOG.md is generated (version facts from images, justifications from pins.yml) and CI-verified for staleness, so it cannot drift like CLAUDE.md did."
```

---

### Task 3: Audit + free loud-failure pins

**Files:**
- Modify: `docker/common/*.dockerfile`, `docker/*/Dockerfile.template` (remove freed pins)
- Modify: `docker/pins/pins.yml` (justify survivors with real `inducing_release`, delete freed)
- Modify: `docker/pins/CATALOG.md` (regenerate)

**Interfaces:** Consumes Task 1–2. No new code — this is the audit applying spec §3/§4 and §7-WS2.

- [ ] **Step 1: Classify every seed pin (record the outcome in the PR description)**

For each `seed — unaudited` pin, decide via Tenet 6:
- **Free it** (default when no testable inducing release is known) — this is the expected outcome for the linters/formatters/build tools and pyyaml.
- **Justify it** — only if a specific `inducing_release` + reason exists (like `go-test-coverage`).
Restrict this task to **loud-failure** tools (a bad float breaks the build visibly): markdownlint-cli, shellcheck, shfmt, actionlint, git-cliff, hadolint, yamllint, ansible-lint, uv, opentofu, nfpm, mkdocs-material, mike, pyyaml, goimports, gocyclo, go-licenses, golangci-lint, cargo-deny, cargo-llvm-cov.
Do NOT touch the silent-failure / fleet-gating set here (trivy, scorecard, semgrep, govulncheck) — that is Task 7, blocked on WS3.

- [ ] **Step 2: Free a pin — remove the version constraint in the fragment**

Example (shellcheck in `docker/common/validation-tools.dockerfile`): change `ARG SHELLCHECK_VERSION=0.11.0` handling so the tool installs latest. For an `ARG`-driven download, replace the pinned download with the "latest release" URL pattern per that tool's convention, or bump the ARG to a floating fetch. Delete the tool's entry from `pins.yml`.

- [ ] **Step 3: Regenerate Dockerfiles + catalog**

Run: `docker/generate.sh && docker/pins/generate_catalog.py`

- [ ] **Step 4: Build the affected images locally to confirm the freed versions install**

Run: `docker/build.sh` (or targeted `docker/generate.sh <lang>` + single build per README). Expected: all images build; freed tools resolve to current releases.

- [ ] **Step 5: Validate**

Run: `vrg-container-run -- vrg-validate`
Expected: PASS, including the `pins` gate (freed pins gone from both Dockerfiles and `pins.yml`; survivors justified).

- [ ] **Step 6: Commit**

```bash
vrg-commit --type feat --scope pins --message "audit pins; free loud-failure tools, justify survivors" \
  --body "Freed <list>; justified <list> with inducing_release. Silent-failure/fleet-gating scanners deferred to the WS3-gated task."
```

---

### Task 4: Publish immutable datestamp aliases

**Files:**
- Modify: `.github/workflows/cd-docker-publish.yml` (language promote ~L204-207; base promote ~L338-341)

**Interfaces:** Produces immutable tags `{prefix}-{lang}:{version}-YYYYMMDD` and `{prefix}-base:latest-YYYYMMDD`, pointing at the same digest as the rolling tag.

- [ ] **Step 1: Add a datestamp env to the build-scan-push job**

```yaml
# in the language job's env: block (near IMAGE/CANDIDATE)
      STAMP: ""   # set at runtime below
```
Add a step before "Promote to final tag":
```yaml
      - name: Compute datestamp
        id: stamp
        run: echo "date=$(date -u +%Y%m%d)" >> "$GITHUB_OUTPUT"
```

- [ ] **Step 2: Promote to BOTH the rolling tag and the immutable alias**

Replace the language "Promote to final tag" step body:
```yaml
      - name: Promote to final + immutable tags
        run: |
          docker buildx imagetools create \
            --tag "$IMAGE" \
            --tag "${IMAGE}-${{ steps.stamp.outputs.date }}" \
            "$CANDIDATE"
```
Apply the identical change to the base job's promote step (using its `IMAGE`).

- [ ] **Step 3: Verify digest preservation still holds for the rolling tag**

The existing "Verify digest preservation" step is unchanged (it checks `$IMAGE`). Confirm it still passes — the immutable alias shares the candidate digest by construction.

- [ ] **Step 4: Dry-run the workflow logic**

Run `actionlint` via validation (below) to confirm YAML/expression validity. A full publish runs only in CI; note in the PR that the first `develop` merge will mint the first `-YYYYMMDD` aliases.

- [ ] **Step 5: Validate and commit**

Run: `vrg-container-run -- vrg-validate`
```bash
vrg-commit --type feat --scope cd --message "publish immutable datestamp image aliases alongside rolling tags" \
  --body "Each build now also publishes {prefix}-{lang}:{version}-YYYYMMDD at the same digest, enabling repoint rollback."
```

---

### Task 5: Sliding-window reaper

**Files:**
- Modify: `.github/workflows/package-cleanup.yml`

**Interfaces:** Reaps `*-YYYYMMDD` aliases older than the window; keeps rolling tags, `cache-*`, and in-window aliases. prod matrix W=30, dev matrix W=7.

- [ ] **Step 1: Read the existing action's tagged-deletion options**

Confirm `dataaxiom/ghcr-cleanup-action@v1.2.2` supports `older-than` + a tag regex (`delete-tags` / `exclude-tags`). Pin the exact option names from its README before writing.

- [ ] **Step 2: Add a second cleanup job for datestamp aliases with per-prefix windows**

```yaml
  reap-aliases:
    name: "reap: ${{ matrix.package }} (${{ matrix.older_than }})"
    runs-on: ubuntu-latest
    strategy:
      fail-fast: false
      matrix:
        include:
          - { package: prod-base,   older_than: "30 days" }
          - { package: prod-python, older_than: "30 days" }
          - { package: prod-go,     older_than: "30 days" }
          - { package: prod-rust,   older_than: "30 days" }
          - { package: prod-java,   older_than: "30 days" }
          - { package: prod-ruby,   older_than: "30 days" }
          - { package: dev-base,    older_than: "7 days" }
          - { package: dev-python,  older_than: "7 days" }
          - { package: dev-go,      older_than: "7 days" }
          - { package: dev-rust,    older_than: "7 days" }
          - { package: dev-java,    older_than: "7 days" }
          - { package: dev-ruby,    older_than: "7 days" }
    steps:
      - name: Reap datestamp aliases past the window
        uses: dataaxiom/ghcr-cleanup-action@v1.2.2
        with:
          owner: vergil-project
          packages: ${{ matrix.package }}
          token: ${{ secrets.GHCR_CLEANUP_TOKEN || secrets.GITHUB_TOKEN }}
          older-than: ${{ matrix.older_than }}
          delete-tags: "*-2*"        # datestamp aliases only (YYYYMMDD starts 2xxx)
          exclude-tags: "latest,cache-*,3.*,1.*,17,21"  # never rolling/version/cache tags
          validate: true
          dry-run: >-
            ${{ !(
                  (github.event_name == 'workflow_dispatch' && inputs.dry_run == false)
                  || (github.event_name == 'schedule' && vars.PACKAGE_CLEANUP_DRY_RUN == 'false')
                ) }}
```

- [ ] **Step 3: Sanity-check the tag globs against real published aliases**

After Task 4 has run once in CI, list tags (`vrg-gh` or the anonymous GHCR API used in the capacity analysis) and confirm `delete-tags`/`exclude-tags` select ONLY `*-YYYYMMDD` and never a rolling/version tag. Adjust globs if a language version (e.g. `1.26`) would be caught.

- [ ] **Step 4: Validate and commit**

Run: `vrg-container-run -- vrg-validate`
```bash
vrg-commit --type feat --scope ops --message "reap datestamp image aliases on a sliding window (prod 30d / dev 7d)" \
  --body "Adds an age-based reaper for {prefix}-{lang}:{version}-YYYYMMDD aliases; rolling/version/cache tags untouched. Ships DRY-RUN like the existing cleanup."
```

---

### Task 6: Repoint rollback procedure

**Files:**
- Create: `docs/rollback.md` (or the repo's docs location) — the documented procedure
- (Decision) optionally a thin wrapper; default is a documented `imagetools` procedure, no new binary.

**Interfaces:** A repeatable "oh shit" procedure: repoint a rolling tag at an in-window immutable digest.

- [ ] **Step 1: Document the procedure**

```markdown
# Image rollback (repoint)

To roll a broken image back to a known-good prior build (within the retention
window — prod 30d, dev 7d):

1. List in-window datestamp aliases:
   `vrg-gh api ... ` (or the anonymous GHCR manifest listing) for
   `prod-<lang>` and pick the last-good `:<version>-YYYYMMDD`.
2. Repoint the rolling tag at that alias's digest:
   `docker buildx imagetools create --tag ghcr.io/vergil-project/prod-<lang>:<version> ghcr.io/vergil-project/prod-<lang>:<version>-YYYYMMDD`
3. Consumers pulling `:<version>` now get the prior build immediately (no rebuild).
4. Open a tracking issue to fix forward and record which alias was rolled to.
```

- [ ] **Step 2: Demonstrate end-to-end once (record evidence in the PR)**

On a non-critical image, publish two datestamp aliases, repoint the rolling tag to the older, and confirm via `imagetools inspect` the rolling tag's digest changed to the older alias. Capture the commands + output.

- [ ] **Step 3: Validate and commit**

Run: `vrg-container-run -- vrg-validate`
```bash
vrg-commit --type docs --scope rollback --message "document image repoint rollback procedure" \
  --body "Repoint a rolling tag at an in-window datestamp alias for instant Axis-B rollback; demonstrated end-to-end."
```

---

### Task 7: Free silent-failure / fleet-gating pins *(blocked-by Task 6 + Task 3)*

**Files:** Modify security-tool fragments + `pins.yml` + regenerate.

**Interfaces:** Frees trivy, scorecard, semgrep(already unpinned — confirm), govulncheck now that repoint rollback exists.

- [ ] **Step 1: Confirm WS3 rollback is live** (Task 6 merged; a repoint demonstrated). If not, stop — this task is gated.
- [ ] **Step 2: Free trivy, scorecard, govulncheck** in `docker/common/security-tools.dockerfile` and `docker/go/Dockerfile.template`; delete their `pins.yml` entries. Confirm semgrep remains intentionally floating (add no pin).
- [ ] **Step 3: Regenerate** — `docker/generate.sh && docker/pins/generate_catalog.py`.
- [ ] **Step 4: Build + validate** — `docker/build.sh` then `vrg-container-run -- vrg-validate`. Confirm the security jobs still run and gate correctly with floated scanners.
- [ ] **Step 5: Commit**
```bash
vrg-commit --type feat --scope pins --message "free security scanners now that repoint rollback exists" \
  --body "trivy/scorecard/govulncheck float; silent-regression risk is bounded by WS3 rollback. semgrep confirmed floating."
```

---

### Task 8: Pin/version exposure report (WS4) *(blocked-by Task 2)*

**Files:**
- Create: `docker/pins/report_exposure.py`
- Create: `docker/pins/test_report.py`

**Interfaces:** Consumes `pins.yml` + a per-tool latest-version lookup. Produces a markdown report; headline section = pins whose `inducing_release` is no longer the leading edge (**due for re-evaluation**).

- [ ] **Step 1: Write the failing test (pure logic, injected latest-version map)**

```python
# docker/pins/test_report.py
from pathlib import Path
import report_exposure as r
DOCKER = Path(__file__).resolve().parent.parent
def test_flags_pin_when_leading_edge_moved_past_inducer():
    pins = {"foo": {"inducing_release": "2.0.0", "state": "active", "constraint": "<2.0.0", "reason": "x", "deterministic": True}}
    due = r.due_for_reevaluation(pins, latest={"foo": "2.1.0"})
    assert "foo" in due
def test_not_flagged_when_inducer_still_leading():
    pins = {"foo": {"inducing_release": "2.0.0", "state": "active", "constraint": "<2.0.0", "reason": "x", "deterministic": True}}
    assert r.due_for_reevaluation(pins, latest={"foo": "2.0.0"}) == []
```

- [ ] **Step 2: Run it, verify fail**

Run: `cd docker/pins && python3 -m pytest test_report.py -v`
Expected: FAIL — `ModuleNotFoundError: report_exposure`.

- [ ] **Step 3: Implement the report logic**

```python
# docker/pins/report_exposure.py
"""WS4 exposure report: what is pinned, in what state, and which pins are due
for re-evaluation (leading edge has moved past the inducing_release).
Upstream-latest lookup is intentionally minimal and pluggable — full drift
tracking is the Dependabot follow-on, NOT built here."""
from __future__ import annotations
import sys
from pathlib import Path
from packaging.version import Version
import check_pins

def due_for_reevaluation(pins: dict, latest: dict[str, str]) -> list[str]:
    out = []
    for tool, meta in pins.items():
        if meta.get("state") != "active":
            continue
        induced = meta.get("inducing_release")
        newest = latest.get(tool)
        if induced and newest and Version(newest) > Version(induced):
            out.append(tool)
    return sorted(out)

def render(root: Path, latest: dict[str, str]) -> str:
    pins = check_pins.load_pins(root)
    due = due_for_reevaluation(pins, latest)
    lines = ["# Pin exposure report\n", "## Due for re-evaluation\n"]
    lines += [f"- **{t}** — inducing release {pins[t]['inducing_release']} is no "
              f"longer leading edge ({latest.get(t)}); re-evaluate per lifecycle." for t in due] or ["- none\n"]
    lines.append("\n## All pins\n")
    for t, m in sorted(pins.items()):
        lines.append(f"- {t} [{m['state']}] {m['constraint']} — {m['reason']}")
    return "\n".join(lines) + "\n"
```

- [ ] **Step 4: Run tests, verify pass**

Run: `cd docker/pins && python3 -m pytest test_report.py -v`
Expected: PASS.

- [ ] **Step 5: Wire a CLI entry that reads latest versions (best-effort) and writes the report**

Add an `if __name__ == "__main__"` that builds `latest` from whatever cheap sources exist (GitHub releases API for the binary tools; leave unknowns absent — absent = not flagged). Keep network calls optional so the unit tests never hit the network.

- [ ] **Step 6: Validate and commit**

Run: `vrg-container-run -- vrg-validate`
```bash
vrg-commit --type feat --scope pins --message "add pin exposure report (due-for-re-evaluation headline)" \
  --body "Reports pins whose inducing_release is no longer leading edge. Internal-state only; upstream drift is the Dependabot follow-on."
```

---

## Self-Review

**Spec coverage:**
- §3 philosophy / §4 lifecycle → `pins.yml` schema (T1) + audit (T3) + report's due-for-re-evaluation (T8). ✓
- §7 WS2 (generated catalog + CI gate + failure-mode sequencing) → T1, T2, T3, T7. ✓
- §7 WS3 (datestamp aliases, sliding window, reaper, rollback) → T4, T5, T6. ✓
- §7 WS4 (internal-state observability) → T8. ✓
- §7 WS1 → out of scope (referenced only). ✓
- §9 capacity (windows 30/7) → encoded in T5 matrix. ✓
- §11 Dependabot / upstream-drift → explicitly NOT built (T8 note). ✓

**Placeholder scan:** No "TBD/handle appropriately" — the one judgment step (T3 per-pin free/justify decision) is inherent audit work, with the decision rule stated. Regexes in T1 flagged for implementer refinement against the real files (Step 3 tests catch misses).

**Type consistency:** `extract_pins.extract → list[Pin]`, `check_pins.load_pins → dict`, `generate_catalog.render(root)`, `report_exposure.due_for_reevaluation(pins, latest)` used consistently across T1/T2/T8. ✓

**Note for filer (epic-create step 9):** file T1–T8 in vergil-containers, linked under #155. Encode blocked-by: T2←T1, T3←T2, T5←T4, T6←T4, T7←(T6,T3), T8←T2.
