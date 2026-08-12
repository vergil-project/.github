# CI Evidence Archival — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> superpowers:subagent-driven-development (recommended) or
> superpowers:executing-plans to implement each task. Steps use checkbox
> (`- [ ]`) syntax for tracking. Each **task** below is filed as its own
> member-repo issue (1:1 with a PR) and linked to epic
> vergil-project/.github#140; the fine-grained red/green steps inside a task are
> executed when that task is implemented.

**Goal:** Capture the complete output of every required CI gate at release time
into a durable, attested, machine-verifiable evidence bundle attached to the
GitHub Release, with completeness enforced as a publish precondition.

**Architecture:** A convention (`ci-evidence-<gate>` artifacts emitted by
`vergil-actions` gate workflows) decouples producers from a single
language-agnostic harvester (`vrg-ci-evidence` in `vergil-tooling`). At publish,
`cd-release` runs the harvester first: it resolves the release PR, selects the
successful CI run, downloads the evidence artifacts, validates completeness
against the same config that drives branch protection, assembles a signed
`tar.gz`, and attaches it to the Release.

**Tech Stack:** Python 3.14 (`vergil-tooling`), GitHub Actions reusable/composite
workflows (`vergil-actions`), `gh` CLI, `actions/attest-build-provenance`,
`actions/upload-artifact` / `actions/download-artifact`.

**Spec:** `epics/140-ci-evidence-archival/spec.md` (this epic's design).

## Global Constraints

- **Portability:** all code/scripts run on macOS and Linux (verbatim from repo
  Key Constraints).
- **Validation:** the only validation command is
  `vrg-container-run -- vrg-validate` (expands to `uv run vrg-validate` here).
  Never run individual linters.
- **Coverage:** 100% enforced. Every new line has a test.
- **Git wrappers:** `vrg-git` / `vrg-gh` only; raw `git`/`gh` are denied.
- **Agents do not submit PRs:** hand off via `vrg-pr-workflow report-ready`;
  the human runs `vrg-submit-pr`.
- **`bin/` command shape:** `def main(argv: list[str] | None = None) -> int`,
  argparse, errors via `vergil_tooling.lib.output.emit_error`.
- **GitHub access:** go through `vergil_tooling.lib.github` (its `_run_with_retry`
  already handles transient retries — reuse it, do not shell out to `gh`
  directly).
- **Determinism:** harvester logic is a pure function of its inputs; timestamps
  (`generated_at`) are injected by the caller, never read from the clock inside
  testable functions.

## Repos and dependency order

```text
vergil-tooling                         vergil-actions
──────────────                         ──────────────
T1  evidence-gate derivation           T8 producer uploads  ─┐ (parallel, early)
T2  manifest + bundle (pure)           T10 all-hard-gates doc ┘ (independent)
T3  harvest layer (gh)
T4  completeness validator  (needs T1,T3)
T5  vrg-ci-evidence CLI     (needs T2,T3,T4)
T6  docs-stage link
T7  convention doc
T12 real report files       (no deps; BLOCKS T9 — no empty bundles)
                                       T9 cd-release wiring, WARNING mode
                                          (needs T5 + T8 + T12)
                                       T11 promote to enforcing, human-gated
                                          (needs T9 + ~1-2 week bake)
```

Dogfood: first real bundle lands on the next `vergil-tooling` release after T9,
in warning mode. The gate goes fatal only at T11, after the bake (spec §9.2).

---

## Task 1: Derive the required evidence-gate set from the branch-protection config

**Repo:** vergil-tooling · **Depends on:** none

**Files:**

- Modify: `src/vergil_tooling/lib/github_config.py` (add `EvidenceGate`,
  `classify_evidence_gate`, `required_evidence_gates`)
- Test: `tests/vergil_tooling/test_github_config_lib.py`

**Interfaces:**

- Consumes: existing `desired_ci_gates_ruleset(project, ci, *, ghas)` and its
  `required_status_checks` list.
- Produces:

  ```python
  @dataclass(frozen=True)
  class EvidenceGate:
      name: str          # "security" | "test" | "audit" | "quality"
      checks: tuple[str, ...]   # required check names classified under it

  def classify_evidence_gate(check_name: str) -> str | None:
      """Map a required status-check name to its evidence gate, or None if the
      check is non-evidence-producing (e.g. version/version-bump)."""

  def required_evidence_gates(
      project: ProjectConfig, ci: CiConfig, *, ghas: bool
  ) -> tuple[EvidenceGate, ...]:
      """The evidence-producing gates this repo MUST emit, derived from the same
      required-status-check computation that drives branch protection."""
  ```

**Classification rule (evidence-producing → gate name):**

- prefix `security /` OR literal `Trivy` / `Semgrep OSS` / `CodeQL` → `security`
- prefix `test /` → `test`
- prefix `audit /` → `audit`
- prefix `quality /` → `quality`
- prefix `version /` → `None` (non-blocking)

**Steps:**

- [ ] Step 1: Write failing tests:

  ```python
  class TestClassifyEvidenceGate:
      def test_quality_prefix(self):
          assert classify_evidence_gate("quality / lint / 3.14") == "quality"
      def test_security_prefixed(self):
          assert classify_evidence_gate("security / codeql") == "security"
      def test_security_ghas_literal(self):
          assert classify_evidence_gate("CodeQL") == "security"
          assert classify_evidence_gate("Trivy") == "security"
      def test_version_is_non_evidence(self):
          assert classify_evidence_gate("version / version-bump") is None

  class TestRequiredEvidenceGates:
      def test_groups_checks_by_gate(self):
          gates = required_evidence_gates(PY_PROJECT, PY_CI, ghas=True)
          names = {g.name for g in gates}
          assert names == {"security", "test", "audit", "quality"}
      def test_no_ghas_drops_codeql_from_security(self):
          gates = {g.name: g for g in
                   required_evidence_gates(PY_PROJECT, PY_CI, ghas=False)}
          assert not any("CodeQL" in c for c in gates["security"].checks)
      def test_gate_absent_when_no_required_check(self):
          gates = {g.name for g in
                   required_evidence_gates(SHELL_PROJECT, SHELL_CI, ghas=False)}
          assert "quality" not in gates  # shell has no lint check registry entry
  ```

  (Reuse the `VergilConfig` fixtures already in `test_github_config_lib.py`.)
- [ ] Step 2: Run — expect FAIL (names not defined).
- [ ] Step 3 (GREEN): Implement `classify_evidence_gate` (prefix/literal table)
  and `required_evidence_gates` (extract check names from the ruleset, classify,
  group, drop `None`). Minimal — no extra options.
- [ ] Step 4: Run — expect PASS.
- [ ] Step 5 (REFACTOR): move the prefix/literal table to a module-level
  constant; ensure `required_evidence_gates` reuses the *existing* ruleset
  check-name extraction rather than re-deriving it (single source of truth).
- [ ] Step 6: `vrg-container-run -- vrg-validate`; commit
  `feat(evidence): derive required evidence-gate set from branch-protection config`.

**Deliverable:** a single source of truth mapping a repo's *enforced* gates to
its *evidence-required* gates. No hand-maintained list.

---

## Task 2: Manifest builder and bundle assembler (pure, no I/O to GitHub)

**Repo:** vergil-tooling · **Depends on:** none

**Files:**

- Create: `src/vergil_tooling/lib/ci_evidence.py`
- Test: `tests/vergil_tooling/test_ci_evidence.py`

**Interfaces:**

- Produces:

  ```python
  @dataclass(frozen=True)
  class GateEvidence:
      name: str
      conclusion: str            # "success"
      tools: tuple[dict, ...]    # from each gate's evidence.json fragment
      metrics: dict
      files: tuple[Path, ...]    # staged under <staging>/gates/<name>/

  @dataclass(frozen=True)
  class HarvestContext:
      repo: str
      version: str
      tag: str
      released_commit: str
      release_pr: int
      validated_head_sha: str
      ci_run_urls: tuple[str, ...]

  def sha256_file(path: Path) -> str: ...

  def build_manifest(
      ctx: HarvestContext,
      gates: Sequence[GateEvidence],
      *,
      generated_at: str,
      missing_gates: Sequence[str],
      staging_dir: Path,
  ) -> dict: ...   # schema_version 1.0 object (see spec §8)

  def write_checks_json(conclusions: Mapping[str, str], staging_dir: Path) -> Path:
      """Write the raw check-run snapshot to <staging>/evidence/checks.json."""

  def write_readme(staging_dir: Path) -> Path:
      """Write the fixed human-orientation README to <staging>/evidence/README.md."""

  def copy_sbom(sbom_file: Path, staging_dir: Path) -> Path:
      """Copy an already-built SBOM into <staging>/evidence/gates/sbom/."""

  def assemble_bundle(staging_dir: Path, out_tarball: Path) -> Path:
      """tar.gz the `evidence/` tree at staging_dir into out_tarball; return it."""
  ```

**Steps:**

- [ ] Step 1: Write failing tests (use `tmp_path`):

  ```python
  def test_sha256_file(tmp_path):
      p = tmp_path / "a.txt"; p.write_text("hello")
      assert sha256_file(p) == (
          "2cf24dba5fb0a30e26e83b2ac5b9e29e1b161e5c1fa7425e73043362938b9824")

  def test_build_manifest_shape(tmp_path):
      staging = _stage_one_gate(tmp_path)   # helper writes gates/test/coverage.xml
      ctx = HarvestContext(repo="o/r", version="2.1.129", tag="v2.1.129",
          released_commit="deadbeef", release_pr=2281,
          validated_head_sha="cafef00d", ci_run_urls=("https://.../runs/1",))
      m = build_manifest(ctx, [_gate_test()], generated_at="2026-07-12T00:00:00Z",
          missing_gates=[], staging_dir=staging)
      assert m["schema_version"] == "1.0"
      assert m["release"]["tag"] == "v2.1.129"
      assert m["gates"][0]["name"] == "test"
      assert m["gates"][0]["files"][0]["sha256"]  # hashed
      assert m["missing_gates"] == []

  def test_assemble_bundle_roundtrip(tmp_path):
      staging = _stage_one_gate(tmp_path)
      out = tmp_path / "bundle.tar.gz"
      assemble_bundle(staging, out)
      with tarfile.open(out) as tf:
          names = tf.getnames()
      assert "evidence/gates/test/coverage.xml" in names

  def test_write_checks_json(tmp_path):
      staging = tmp_path / "s"; (staging / "evidence").mkdir(parents=True)
      p = write_checks_json({"test / unit / 3.14": "success"}, staging)
      assert json.loads(p.read_text())["test / unit / 3.14"] == "success"

  def test_write_readme(tmp_path):
      staging = tmp_path / "s"; (staging / "evidence").mkdir(parents=True)
      assert "evidence" in write_readme(staging).read_text().lower()

  def test_copy_sbom(tmp_path):
      sbom = tmp_path / "sbom.cdx.json"; sbom.write_text("{}")
      staging = tmp_path / "s"; (staging / "evidence").mkdir(parents=True)
      out = copy_sbom(sbom, staging)
      assert out == staging / "evidence" / "gates" / "sbom" / "sbom.cdx.json"
  ```

- [ ] Step 2 (RED): Run — expect FAIL (names not defined).
- [ ] Step 3 (GREEN): Implement with `hashlib`, `json`, `tarfile`, `shutil`
  (deterministic: sort tar members, no mtime dependence in assertions). Minimal
  code to pass — no speculative options.
- [ ] Step 4: Run — expect PASS.
- [ ] Step 5 (REFACTOR): extract the shared "path under `evidence/`" join into one
  helper; ensure `build_manifest`'s per-file hashing reuses `sha256_file` (no
  duplicated hashing); confirm README text is a module constant, not inline.
- [ ] Step 6: validate; commit
  `feat(evidence): add manifest builder, bundle assembler, and metadata files`.

**Deliverable:** the entire bundle-shaping core — manifest, `checks.json`,
`README.md`, SBOM copy, and the tarball — fully unit-tested with zero network
dependency.

---

## Task 3: GitHub harvest layer — PR resolution, run selection, artifact download

**Repo:** vergil-tooling · **Depends on:** Task 2 (types)

**Files:**

- Modify: `src/vergil_tooling/lib/ci_evidence.py`
- Reuse: `src/vergil_tooling/lib/github.py` (`read_json`, `run`,
  `_run_with_retry`), `src/vergil_tooling/lib/linkage.py`
  (`extract_tracking_issue` pattern for merge→PR)
- Test: `tests/vergil_tooling/test_ci_evidence_harvest.py`

**Interfaces:**

- Produces:

  ```python
  class NoQualifyingRunError(Exception): ...   # substantive: no green CI run

  def resolve_release_pr(repo: str, merge_sha: str) -> int:
      """merge commit -> PR number (via /commits/{sha}/pulls, else linkage)."""

  def select_ci_run(repo: str, head_sha: str, *, workflow: str = "CI") -> dict:
      """Return the COMPLETED + SUCCESS CI run for head_sha; raise
      NoQualifyingRunError if none. Ignores cancelled/in-progress runs."""

  def download_evidence_artifacts(repo: str, run_id: int, dest: Path) -> list[Path]:
      """Download every `ci-evidence-*` artifact of the run into dest/<gate>/."""

  def read_gate_conclusions(repo: str, head_sha: str) -> dict[str, str]:
      """check-run name -> conclusion, for the manifest and validation."""
  ```

**Key selection logic (spec §5.2):**

```python
runs = github.read_json(
    "api", f"repos/{repo}/actions/runs",
    "-f", f"head_sha={head_sha}", "--jq", ".workflow_runs")
qualifying = [r for r in runs
              if r["name"] == workflow
              and r["status"] == "completed"
              and r["conclusion"] == "success"]
if not qualifying:
    raise NoQualifyingRunError(head_sha)
return max(qualifying, key=lambda r: r["run_started_at"])
```

**Steps:**

- [ ] Step 1: Failing tests, mocking `github.read_json`/`github.run` via
  `monkeypatch` (mirror `test_github.py` style):

  ```python
  def test_select_ci_run_ignores_cancelled(monkeypatch):
      monkeypatch.setattr(github, "read_json", lambda *a, **k: [
          {"name":"CI","status":"completed","conclusion":"cancelled",
           "run_started_at":"2026-07-12T00:00:00Z"},
          {"name":"CI","status":"completed","conclusion":"success",
           "run_started_at":"2026-07-12T00:05:00Z","id":42}])
      assert select_ci_run("o/r","abc")["id"] == 42
  def test_select_ci_run_none_raises(monkeypatch):
      monkeypatch.setattr(github,"read_json",lambda *a,**k:[
          {"name":"CI","status":"completed","conclusion":"cancelled",
           "run_started_at":"t"}])
      with pytest.raises(NoQualifyingRunError):
          select_ci_run("o/r","abc")
  def test_download_evidence_artifacts_filters_prefix(monkeypatch, tmp_path):
      ...  # only ci-evidence-* downloaded, staged under dest/<gate>/
  ```

- [ ] Step 2: Run — expect FAIL.
- [ ] Step 3 (GREEN): Implement using `github.read_json`/`github.run`
  (`gh run download --name ci-evidence-<gate>` or the artifacts API). PR
  resolution: `gh api repos/{repo}/commits/{sha}/pulls` first, fall back to
  `extract_tracking_issue`.
- [ ] Step 4: Run — expect PASS.
- [ ] Step 5 (REFACTOR): route every `gh` call through `lib.github` (its
  `_run_with_retry` gives transient-retry for free — no bespoke retry loops);
  extract the run-qualification predicate into a named helper for reuse/testing.
- [ ] Step 6: validate; commit
  `feat(evidence): add GitHub harvest layer (PR resolve, run select, download)`.

**Deliverable:** all GitHub I/O, isolated behind mockable functions;
`NoQualifyingRunError` marks the one substantive-failure boundary.

---

## Task 4: Completeness validator

**Repo:** vergil-tooling · **Depends on:** Tasks 1, 3

**Files:**

- Modify: `src/vergil_tooling/lib/ci_evidence.py`
- Test: `tests/vergil_tooling/test_ci_evidence.py`

**Interfaces:**

- Consumes: `EvidenceGate` (T1), harvested gate dir (T3).
- Produces:

  ```python
  class IncompleteEvidenceError(Exception):
      def __init__(self, missing: list[str]) -> None:
          self.missing = missing
          super().__init__(f"missing evidence for required gates: {missing}")

  def validate_completeness(
      required: Sequence[EvidenceGate],
      harvested: Mapping[str, GateEvidence],
  ) -> None:
      """Raise IncompleteEvidenceError listing every required gate with no
      harvested evidence. Substantive failure — terminal, blocks publish."""
  ```

**Steps:**

- [ ] Step 1: Failing tests:

  ```python
  def test_all_present_ok():
      validate_completeness([EvidenceGate("test",("test / unit / 3.14",))],
                            {"test": _gate_test()})  # no raise
  def test_missing_required_raises():
      with pytest.raises(IncompleteEvidenceError) as e:
          validate_completeness(
              [EvidenceGate("security",("Trivy",)),
               EvidenceGate("test",("test / unit / 3.14",))],
              {"test": _gate_test()})
      assert e.value.missing == ["security"]
  ```

- [ ] Step 2: Run — expect FAIL.
- [ ] Step 3 (GREEN): Implement (set difference of required names vs harvested
  keys). Minimal.
- [ ] Step 4: Run — expect PASS.
- [ ] Step 5 (REFACTOR): keep `IncompleteEvidenceError` a thin data-carrying
  exception (`.missing`); ensure the message format is defined once for reuse by
  the CLI's `emit_error`.
- [ ] Step 6: validate; commit
  `feat(evidence): add completeness validator with substantive-failure error`.

**Deliverable:** the publish-invariant gate, as a pure function.

---

## Task 5: `vrg-ci-evidence` CLI (`bundle` subcommand)

**Repo:** vergil-tooling · **Depends on:** Tasks 2, 3, 4

**Files:**

- Create: `src/vergil_tooling/bin/vrg_ci_evidence.py`
- Modify: `pyproject.toml` (add `vrg-ci-evidence = "...:main"` console script)
- Test: `tests/vergil_tooling/test_vrg_ci_evidence.py`

**Interfaces:**

- Consumes: T2 `build_manifest`/`assemble_bundle`, T3 harvest fns, T4
  `validate_completeness`, T1 `required_evidence_gates`.
- CLI:

  ```bash
  vrg-ci-evidence bundle \
    --repo owner/name --version 2.1.129 --merge-sha <sha> \
    --generated-at <iso8601> --out-dir <dir> [--sbom-file <path>]
  # writes <out-dir>/v<version>-ci-evidence.tar.gz and
  #        <out-dir>/v<version>-ci-evidence-manifest.json
  # exit 0 on complete evidence; non-zero on IncompleteEvidenceError /
  # NoQualifyingRunError, with the missing gates / run identity on stderr.
  ```

**Orchestration (in `main`):** resolve PR → head SHA → `select_ci_run` →
`download_evidence_artifacts` → parse each `evidence.json` into `GateEvidence`
→ `read_gate_conclusions` → `required_evidence_gates` → `validate_completeness`
→ stage metadata (`write_checks_json`, `write_readme`, and `copy_sbom` when
`--sbom-file` is given) → `build_manifest` → `assemble_bundle` → write standalone
manifest. Wrap transient GitHub errors in retry (via `lib.github`); let
`IncompleteEvidenceError`/`NoQualifyingRunError` propagate to a non-zero exit
with `emit_error`.

**Steps:**

- [ ] Step 1: Failing test — full `main()` run with every `lib.ci_evidence`
  GitHub fn monkeypatched to fixtures; assert tarball + manifest written and
  exit 0.
- [ ] Step 2: Failing test — missing required gate → `emit_error` called, exit
  non-zero, no tarball written.
- [ ] Step 3: Run — expect FAIL.
- [ ] Step 4 (GREEN): Implement `main(argv)`; register console script.
- [ ] Step 5: Run — expect PASS.
- [ ] Step 6 (REFACTOR): keep `main` a thin orchestrator — each stage is a call
  into `lib.ci_evidence`, no business logic in the CLI; consolidate the
  error→exit mapping (`IncompleteEvidenceError`, `NoQualifyingRunError`) into one
  block.
- [ ] Step 7: validate; commit
  `feat(evidence): add vrg-ci-evidence bundle command`.

**Deliverable:** the runnable harvester, dogfoodable from a shell before any CI
wiring.

---

## Task 6: Doc-site evidence link in `vrg-docs-stage`

**Repo:** vergil-tooling · **Depends on:** none (uses only the asset URL
convention)

**Files:**

- Modify: `src/vergil_tooling/lib/docs.py` (add `evidence_link_line`,
  integrate into staged release pages)
- Modify: `src/vergil_tooling/bin/vrg_docs_stage.py` (flag + call)
- Test: `tests/vergil_tooling/test_docs.py`

**Interfaces:**

- Produces:

  ```python
  def evidence_link_line(repo: str, tag: str, *, has_asset: bool) -> str | None:
      """Return the '**CI Evidence:** ... [Download →](...)' markdown for a
      release page, or None when the release has no evidence asset."""
  ```

  Asset URL: `https://github.com/{repo}/releases/download/{tag}/{tag}-ci-evidence.tar.gz`.
  `has_asset` is resolved by the caller via `gh release view {tag} --json assets`
  (one cheap call per release at docs-build time).

**Steps:**

- [ ] Step 1: Failing tests:

  ```python
  def test_evidence_link_present():
      line = evidence_link_line("o/r","v2.1.129", has_asset=True)
      assert "All gates passed" in line and "v2.1.129-ci-evidence.tar.gz" in line
  def test_evidence_link_absent():
      assert evidence_link_line("o/r","v1.0.0", has_asset=False) is None
  ```

- [ ] Step 2: Run — expect FAIL.
- [ ] Step 3 (GREEN): Implement; append the line to each staged `releases/v*.md`
  whose asset exists (asset lookup mocked in tests).
- [ ] Step 4: Run — expect PASS.
- [ ] Step 5 (REFACTOR): keep `evidence_link_line` pure (no `gh` call inside it —
  the asset lookup stays in `vrg_docs_stage` so the function is trivially
  testable); define the asset-name pattern once and share it with the harvester's
  output naming (T5) to prevent drift.
- [ ] Step 6: validate; commit
  `feat(docs): link CI evidence bundle from release pages`.

**Deliverable:** the thin, deterministic doc-site surface (spec §11).

---

## Task 7: Convention documentation

**Repo:** vergil-tooling · **Depends on:** Tasks 1–5 (stable interfaces)

**Files:**

- Create: `docs/specs/ci-evidence-convention.md` (or `docs/site/docs/...`
  following the repo's docs placement)
- Test: n/a (doc) — validated by `vrg-validate` markdown checks

**Content:** the `ci-evidence-<gate>` artifact name; the `evidence.json` fragment
schema; the evidence-producing gate set and its derivation from
`desired_ci_gates_ruleset`; the bundle tree + `manifest.json` schema (v1.0); the
publish invariant; how a downstream auditor verifies (`gh attestation verify` +
per-file `sha256`).

**Steps:**

- [ ] Step 1: Write the doc from spec §7/§8/§10.
- [ ] Step 2: `vrg-container-run -- vrg-validate` (markdown passes).
- [ ] Step 3: commit `docs(evidence): document the CI evidence convention`.

**Deliverable:** the contract any repo/future gate conforms to.

---

## Task 8: Producer uploads in `vergil-actions` gate workflows

**Repo:** vergil-actions · **Depends on:** none (parallel; harmless everywhere)

**Files:**

- Modify: `.github/workflows/ci-security.yml`, `ci-test.yml`, `ci-audit.yml`,
  `ci-quality.yml` (and the composite actions they call)
- Add: a shared `actions/ci/evidence/emit` composite that writes `evidence.json`
  and calls `actions/upload-artifact` with name `ci-evidence-<gate>`

**Per gate:** after the gate's checks pass, stage its report files + an
`evidence.json` fragment (`{gate, tools:[{name,version}], metrics, files}`) and
upload as `ci-evidence-<gate>`. Security uploads SARIF **unconditionally**
(independent of code-scanning upload — spec §7).

**Steps:**

- [ ] Step 1: Add the `emit` composite; wire `ci-test` first (coverage.xml +
  junit.xml + evidence.json).
- [ ] Step 2: Open a PR; confirm on its own CI run that a `ci-evidence-test`
  artifact appears with the expected contents.
- [ ] Step 3: Repeat for `audit`, `quality`, `security` (security: ensure
  Trivy/Semgrep/CodeQL SARIF is emitted as an artifact regardless of GHAS).
- [ ] Step 4: commit per gate;
  `feat(ci): emit ci-evidence-<gate> artifacts from gate workflows`.

**Deliverable:** every required gate emits durable, harvestable evidence.
Verified by artifact presence on a live PR (no unit test possible for workflow
YAML).

---

## Task 12: Emit real report files from the check registry

**Repo:** vergil-tooling · **Depends on:** none (runnable now) · **Blocks:** T9
(no bundle is attached until its reports carry real data — spec §7.2)

**Problem:** the check registry produces pass/fail only — `pytest --cov ...
--cov-fail-under=100` writes no `coverage.xml`/`junit.xml`, and
`pip-audit`/license checks write no JSON. So `test`/`audit`/`quality` evidence
would be an `evidence.json` envelope with **no actual report** inside. There is no
point publishing empty reports (spec §7.2).

**Files:**

- Modify: the check-command registry in `src/vergil_tooling/` (per T8's finding,
  `languages.py`) so the evidence-producing gates emit machine-readable reports at
  **workspace-root paths that match the T8 composite's globs**:
  - `test`: add `--cov-report=xml:coverage.xml` (keep `--cov-fail-under=100`) and
    `--junitxml=junit.xml` to the pytest command.
  - `audit`: `pip-audit ... --format=json --output=pip-audit.json`; license check
    → `licenses.json`.
  - `quality`: emit the linter/type-checker machine-readable output where the tool
    supports it (e.g. a JSON report); if a tool has none, its `evidence.json`
    (tool + version + zero findings) is a complete statement, not an empty report.
- Test: `tests/vergil_tooling/` — assert the constructed command strings include
  the report-output flags and the agreed output paths.

**Path contract (shared with T8):** `coverage.xml`, `junit.xml`, `pip-audit.json`,
`licenses.json` at the workspace root. Both the emitter (here) and the composite
glob (T8) reference this one list — keep them in sync.

**Steps:**

- [ ] Step 1 (RED): failing tests asserting each evidence-producing check command
  contains its report-output flag/path.
- [ ] Step 2: Run — expect FAIL.
- [ ] Step 3 (GREEN): add the flags/paths to the registry command definitions.
- [ ] Step 4: Run — expect PASS; `vrg-container-run -- vrg-validate` green and
  confirm the report files actually appear after a check run.
- [ ] Step 5 (REFACTOR): define the report-path constants once and reference them
  from every command; keep the list aligned with the T8 glob list.
- [ ] Step 6: commit `feat(checks): emit machine-readable report files for evidence`.

**Deliverable:** every evidence-producing gate writes a real, machine-readable
report — so bundles contain data, not empty envelopes. Prerequisite for T9.

---

## Task 9: `cd-release` wiring, attestation, and evidence-first reorder

**Repo:** vergil-actions · **Depends on:** Task 5 (command), Task 8 (artifacts),
Task 12 (real report files — no empty bundles, spec §7.2)

**Files:**

- Create: `actions/cd/release/ci-evidence/action.yml`
- Modify: `.github/workflows/cd-release.yml` (run evidence FIRST; gate publish
  on it), `.github/workflows/cd.yml` **in each consuming repo** to add
  `actions: read` permission (spec §5.2)

**Composite `ci-evidence/action.yml`** — ships in **warning mode** (spec §9.2):

- Input `enforce` (default `false`). Input `timeout-minutes` (bounded run).

1. `vrg-ci-evidence bundle --repo ${{ github.repository }} --version <v>
   --merge-sha ${{ github.sha }} --generated-at <now> --out-dir evidence-out
   [--sbom-file <path-to-already-built-SBOM>]`. The SBOM is already produced in
   the CD workspace for its own Release asset; pass its path so the bundle is
   self-contained.
2. `actions/attest-build-provenance` over the bundle digest.
3. `gh release upload <tag> evidence-out/*.tar.gz evidence-out/*-manifest.json
   --clobber` (idempotent).

- **Mode handling:** when `enforce == false` (warning), the whole step is wrapped
  so that **any** failure/timeout in steps 1–3 emits a loud `::warning::` and the
  job succeeds (release proceeds, partial bundle attached if any). When
  `enforce == true`, a non-zero `vrg-ci-evidence` exit (substantive
  incompleteness) fails the job — the publish precondition.

**Reorder:** evidence step runs before build-and-publish and tag-and-release. In
warning mode it cannot block; in enforcing mode nothing publishes unless evidence
is complete (spec §9). Position is identical in both modes so the promotion
(Task 11) is a pure flag flip, no restructuring.

**Steps:**

- [ ] Step 1: Add the composite action with the `enforce` (default `false`) and
  `timeout-minutes` inputs and the warning-mode wrapper; call it as the first step
  in `cd-release.yml`.
- [ ] Step 2: Add `actions: read` to `cd.yml` permissions in `vergil-tooling`
  (the dogfood repo) and confirm harvest can list the release PR's artifacts.
- [ ] Step 3: Trigger a real `vergil-tooling` release in **warning mode**; confirm
  the bundle + manifest attach and `gh attestation verify` passes, and that an
  induced evidence failure produces a warning **without** aborting the release.
- [ ] Step 4: commit
  `feat(cd): harvest, attest, and attach CI evidence (warning mode)`.

**Deliverable:** the evidence gate live in warning mode across every
release-publishing repo — non-fatal, timeout-bounded, attaching bundles on every
release while reliability data accrues (spec §9.2, §14.1). Promotion to enforcing
is Task 11.

---

## Task 10: Codify the all-hard-gates principle

**Repo:** vergil-actions · **Depends on:** none (independent)

**Files:**

- Create: a foundational-principles doc under `vergil-actions` docs

**Content (spec §14 phase 4):** every check that matters is a hard, asserting
gate; there are no *permanent* report-only/warning gates, because an unheeded
warning is meaningless (reviewers optimize for "can I merge?" and never revisit
warnings); deprecation warnings are early signal of a future outage and are
treated as errors; avoid the five-page warning-exception list — if it matters,
assert it and fail on it. **Also document the gate deployment lifecycle (spec
§9.2):** a new hard gate ships in warning mode (non-fatal, bounded bake-in) and is
promoted to enforcing — warning mode is a *temporary deployment state of a hard
gate*, never a permanent soft gate. Use the CI-evidence gate as the example.
Cross-reference the publish-safety property (spec §9.1).

**Steps:**

- [ ] Step 1: Write the doc.
- [ ] Step 2: `vrg-validate` markdown passes.
- [ ] Step 3: commit `docs(ci): codify the all-hard-gates principle`.

**Deliverable:** the principle that makes public evidence bundles safe, on the
record.

---

## Task 11: Promote the evidence gate to enforcing (human-gated)

**Repo:** vergil-actions · **Depends on:** Task 9 shipped + a ~1–2 week
warning-mode bake with reliability data reviewed. **Opened only when the human
judges the gate stable** (spec §9.2, §14.1).

**Files:**

- Modify: the `enforce` default (`false` → `true`) at its single source — the
  `cd-release` evidence step / composite input default.

**This is deliberately a pure flag flip** — the evidence step's position and code
are identical between modes (Task 9), so promotion changes only fatality.

**Steps:**

- [ ] Step 1: Review accumulated warning-mode data — bundle-success rate across
  releases, every warning emitted, any defects — and confirm the gate is reliable
  enough to make fatal. (Human judgment; this is the gate.)
- [ ] Step 2: Flip the `enforce` default to `true`; `vrg-validate`.
- [ ] Step 3: Trigger a `vergil-tooling` release and confirm the **negative path**
  now fails closed: a deliberately withheld required-gate artifact fails the
  release with a clear "missing evidence for required gates" message and publishes
  nothing.
- [ ] Step 4: commit `feat(cd): promote CI evidence gate to enforcing`.

**Deliverable:** the §9 publish invariant, live. Rollback if needed: `vrg-promote`
demotion of the `2.1` tag (spec §14.1).

---

## Self-Review

**Spec coverage:**

- §5 architecture / §5.1 flow → T3, T5, T9. ✔
- §5.2 preconditions (`actions: read`, run selection) → T3 (selection), T9
  (permission). ✔
- §6 components → T1–T9 map 1:1 to the five units + producers. ✔
- §7 convention + §7.1 derived set + §7.2 producer prerequisite → T1 (derivation),
  T8 (artifact emission), T12 (real report files — no empty reports), T7 (doc). ✔
- §8 bundle/manifest, incl. `checks.json`, `README.md`, and `gates/sbom/` →
  T2 (manifest, checks.json, README, SBOM copy) + T5 (staging wiring) + T9
  (SBOM path). ✔ *(alignment gap closed)*
- §9 publish invariant + §9.1 publish safety + §9.2 deployment lifecycle →
  T4 (validator), T9 (warning-mode wiring + `enforce` flag), T11 (promotion to
  enforcing), T10 (safety principle + lifecycle doc). ✔
- §10 attestation → T9. ✔
- §11 doc-site link → T6. ✔
- §12 error handling / §12.1 forward-compat → T3 (`NoQualifyingRunError`), T4/T5
  (substantive vs transient via retry in `lib.github`), T9 (warning-mode collapses
  both to non-fatal). ✔
- §14 rollout / §14.1 warning-then-enforcing + rollback → task ordering, T9
  (warning-mode deploy), T11 (human-gated flip), `vrg-promote` rollback. ✔
- §15 future work → intentionally excluded. ✔

**Placeholder scan:** no TBD/TODO; every logic task carries real test code and
signatures. Workflow tasks (T8, T9) verify via live-PR/live-release artifact
presence because workflow YAML has no unit-test surface — the verification step
is concrete (named artifact appears / attestation verifies / negative path
fails closed).

**Type consistency:** `EvidenceGate` (T1) consumed by T4/T5; `GateEvidence` /
`HarvestContext` (T2) consumed by T3/T5; `IncompleteEvidenceError` (T4) and
`NoQualifyingRunError` (T3) handled in T5/T9. Names consistent across tasks. ✔

**Note for alignment:** the per-task step lists are TDD-shaped but coarse; the
paad:alignment pass will formalize each into strict red/green/refactor and can
split any task a reviewer would gate independently.
