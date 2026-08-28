# Fleet-wide Python test-suite performance — Implementation Plan

> **For agentic workers:** This is an **epic plan**. Each task below becomes a
> GitHub implementation task (issue) filed under epic `vergil-project/.github#333`
> and implemented later via `vergil:issue-implement`, which runs its own
> task-by-task TDD. Steps use checkbox (`- [ ]`) syntax for tracking. Tasks are
> ordered by dependency; the **Depends-on** line on each task is authoritative.

**Goal:** Make the full `vrg-validate` test-stage wall-clock materially faster for
every Python repo — proven with measured numbers — without changing what the 100%
branch-coverage gate measures.

**Architecture:** Prove the levers (sysmon coverage backend, pytest-xdist
worksteal, import-mode, subprocess-hotspot removal) on vergil-tooling's suite,
then land the universal ones in the shared `languages.py` TEST command so all
Python repos inherit them. The shared command **computes** its flags from
(interpreter version, xdist availability, `[test].parallel` config); config
declares intent, `languages.py` is the policy engine. Parallelism is on-by-default
with a per-repo opt-out so no repo is ever left known-broken.

**Tech Stack:** Python 3.12+, pytest, pytest-cov, pytest-xdist (worksteal),
coverage.py `sys.monitoring` backend, uv, the `vergil_tooling.lib.fleet_sweep`
driver.

**Spec:** `epics/333-python-test-perf/spec.md` (in `vergil-project/.github`).

## Global Constraints

- **Python floor:** `>=3.12,<4.0`. sysmon requires 3.12+; guard every sysmon
  activation on `sys.version_info >= (3, 12)`.
- **Coverage gate is inviolate:** `--cov-branch --cov-fail-under=100` must keep
  measuring the identical missing-line/branch set. Every speed change is gated
  behind the Phase-0 coverage-equivalence proof (Task 2).
- **CI matrix:** 3.12, 3.13, 3.14. Branch coverage can differ across versions;
  100% must hold on all three.
- **No silent failures** (repo policy): a fleet-sweep step that cannot edit a
  repo must report loudly and file a follow-up — never a silent no-op.
- **Shell wrappers:** `vrg-git` / `vrg-gh`, never raw `git`/`gh`. Validate with
  `vrg-container-run -- vrg-validate` (expands to `uv run vrg-validate` here).
- **Commits:** `vrg-commit --type <t> --scope <s> --message <m>`; agents stop at
  `vrg-pr-workflow report-ready`, humans submit.
- **Files change in the task's own repo** (placement law). Cross-repo tasks
  (Task 11 → vergil-containers) land their PR there.

---

## Phase 0 — Baseline, profile & fleet survey (evidence; no behavior change)

### Task 1: Measurement harness + baseline + hotspot map

**Repo:** vergil-tooling · **Depends-on:** none

**Files:**
- Create: `scripts/perf/measure_test_stage.py` — warm-run driver.
- Create: `epics/333-python-test-perf/evidence/baseline.md` — recorded results
  (committed as the evidence record; path mirrors the epic slug).
- Test: `tests/vergil_tooling/test_measure_test_stage.py`

**Interfaces:**
- Produces: `measure_test_stage.run(config: MeasureConfig) -> MeasureResult` where
  `MeasureResult` has `.median_seconds: float`, `.per_run: list[float]`,
  `.label: str`. Later tasks re-run this harness to measure their deltas.

**What it does:** runs a given pytest invocation N times (default 5) after one
warm-up run (container up, deps synced), discards the warm-up, reports the median
and the raw list. It parametrizes the pytest argv/env so the same harness measures
each cumulative configuration: `baseline (serial, C-tracer)`, `+sysmon`,
`+xdist -n auto`, `+worksteal`, `+import-mode`. It also captures a hotspot map
by running once with `--durations=25` and once with `-X importtime`, parsing the
top offenders into the evidence file.

- [ ] **Step 1: Write the failing test** — median math is the only pure logic to unit-test.

```python
# tests/vergil_tooling/test_measure_test_stage.py
from scripts.perf.measure_test_stage import median_seconds

def test_median_discards_warmup_and_returns_middle():
    # warm-up already stripped by caller; median of the timed runs
    assert median_seconds([3.0, 1.0, 2.0]) == 2.0

def test_median_even_count_averages_middle_pair():
    assert median_seconds([1.0, 2.0, 3.0, 4.0]) == 2.5
```

- [ ] **Step 2: Run to verify it fails**

Run: `uv run pytest tests/vergil_tooling/test_measure_test_stage.py -v`
Expected: FAIL (`ModuleNotFoundError` / `median_seconds` undefined).

- [ ] **Step 3: Implement `median_seconds` + the harness driver**

```python
# scripts/perf/measure_test_stage.py
from statistics import median

def median_seconds(samples: list[float]) -> float:
    return median(samples)
```

Then the run loop (subprocess timing around `uv run pytest …`, warm-up discarded,
argv/env injected per configuration). Keep the driver thin; the unit test covers
only `median_seconds`.

- [ ] **Step 4: Run to verify pass**

Run: `uv run pytest tests/vergil_tooling/test_measure_test_stage.py -v` → PASS.

- [ ] **Step 5: Produce the baseline evidence**

Run the harness across all cumulative configurations; write medians + the hotspot
map to `epics/333-python-test-perf/evidence/baseline.md`. Record the ranked
subprocess hotspots (file:line, count, measured duration) — this is Task 12's
work-list and Task 10's stopping-target input.

- [ ] **Step 6: Commit**

```bash
vrg-commit --type test --scope perf --message "add test-stage measurement harness + baseline (#333)"
```

### Task 2: Coverage-equivalence proof (C-tracer vs sysmon, serial vs parallel)

**Repo:** vergil-tooling · **Depends-on:** Task 1

**Files:**
- Create: `scripts/perf/coverage_equivalence.py` — runs the suite under each
  backend, extracts the missing-line/branch set from `coverage.xml`, diffs them.
- Create: `epics/333-python-test-perf/evidence/coverage-equivalence.md`
- Test: `tests/vergil_tooling/test_coverage_equivalence.py`

**Interfaces:**
- Produces: `coverage_equivalence.diff_reports(a: Path, b: Path) -> list[str]` —
  returns the symmetric difference of (file, line, branch) misses between two
  `coverage.xml` reports; empty list means identical.

**Why it gates everything:** Phase 1 (sysmon) and Phase 2 (xdist) must not ship
until this returns empty for `C-tracer serial` vs `sysmon serial` **and**
`C-tracer serial` vs `sysmon + -n auto`.

- [ ] **Step 1: Write the failing test**

```python
# tests/vergil_tooling/test_coverage_equivalence.py
from pathlib import Path
from scripts.perf.coverage_equivalence import diff_reports

def test_identical_reports_diff_empty(tmp_path: Path):
    xml = '<coverage><packages><package><classes><class filename="a.py">' \
          '<lines><line number="1" hits="1"/></lines></class></classes>' \
          '</package></packages></coverage>'
    (tmp_path / "a.xml").write_text(xml)
    (tmp_path / "b.xml").write_text(xml)
    assert diff_reports(tmp_path / "a.xml", tmp_path / "b.xml") == []

def test_differing_miss_is_reported(tmp_path: Path):
    hit = '<coverage><packages><package><classes><class filename="a.py">' \
          '<lines><line number="1" hits="1"/></lines></class></classes></package></packages></coverage>'
    miss = hit.replace('hits="1"', 'hits="0"')
    (tmp_path / "a.xml").write_text(hit)
    (tmp_path / "b.xml").write_text(miss)
    assert diff_reports(tmp_path / "a.xml", tmp_path / "b.xml") != []
```

- [ ] **Step 2: Run to verify it fails** — `uv run pytest tests/vergil_tooling/test_coverage_equivalence.py -v` → FAIL.
- [ ] **Step 3: Implement `diff_reports`** — parse `coverage.xml` (stdlib `xml.etree`), collect `(filename, line, condition-coverage)` misses into a set per report, return sorted symmetric difference as human-readable strings.
- [ ] **Step 4: Run to verify pass** → PASS.
- [ ] **Step 5: Produce the proof** — run the suite under each backend/parallelism combo, diff, write result to the evidence file. **If any diff is non-empty, STOP and surface it — this blocks Phase 1.**
- [ ] **Step 6: Commit** — `vrg-commit --type test --scope perf --message "add coverage-equivalence proof (#333)"`

### Task 3: Fleet surveys (collection-safety, dev-dependency-shape, Python floor)

**Repo:** vergil-tooling · **Depends-on:** none (parallel with Tasks 1–2)

**Files:**
- Create: `scripts/perf/fleet_survey.py`
- Create: `epics/333-python-test-perf/evidence/fleet-survey.md` — three tables.
- Test: `tests/vergil_tooling/test_fleet_survey.py`

**Interfaces:**
- Produces:
  - `fleet_survey.classify_test_layout(repo: Path) -> LayoutVerdict` with
    `.packaged: bool`, `.duplicate_basenames: list[str]`, `.importlib_safe: bool`.
  - `fleet_survey.classify_dev_deps(repo: Path) -> DevDepShape` — one of
    `UV_GROUPS | PEP621_OPTIONAL | REQUIREMENTS_TXT | POETRY | UNKNOWN`.
  - `fleet_survey.python_floor(repo: Path) -> str | None` — parsed
    `requires-python`.

These three verdicts are consumed by Task 9 (import-mode gate: `importlib_safe`),
Task 12 (sweep applicator: `DevDepShape`), and Task 4 (sysmon guard sanity:
`python_floor`).

- [ ] **Step 1: Write failing tests** — one per classifier, using `tmp_path` fixtures that materialize each layout/shape.

```python
# tests/vergil_tooling/test_fleet_survey.py
from pathlib import Path
from scripts.perf.fleet_survey import classify_test_layout, classify_dev_deps, DevDepShape

def test_packaged_unique_basenames_is_importlib_safe(tmp_path: Path):
    t = tmp_path / "tests"; t.mkdir()
    (t / "__init__.py").touch()
    (t / "test_a.py").touch()
    v = classify_test_layout(tmp_path)
    assert v.packaged and v.importlib_safe and v.duplicate_basenames == []

def test_duplicate_basenames_not_importlib_safe(tmp_path: Path):
    for sub in ("x", "y"):
        d = tmp_path / "tests" / sub; d.mkdir(parents=True)
        (d / "test_dup.py").touch()
    v = classify_test_layout(tmp_path)
    assert v.importlib_safe is False and "test_dup.py" in v.duplicate_basenames

def test_uv_dependency_groups_shape(tmp_path: Path):
    (tmp_path / "pyproject.toml").write_text('[dependency-groups]\ndev = ["pytest"]\n')
    assert classify_dev_deps(tmp_path) is DevDepShape.UV_GROUPS
```

- [ ] **Step 2: Run to verify they fail** → FAIL.
- [ ] **Step 3: Implement the three classifiers** — filesystem + TOML parsing (stdlib `tomllib`), no network.
- [ ] **Step 4: Run to verify pass** → PASS.
- [ ] **Step 5: Run the survey across the fleet** — enumerate Python repos (sibling clones), write the three tables to the evidence file. Flag any repo that is `UNKNOWN` shape or `importlib_safe is False`.
- [ ] **Step 6: Commit** — `vrg-commit --type test --scope perf --message "add fleet surveys: collection-safety, dev-dep shape, python floor (#333)"`

---

## Phase 1 — sysmon (the one true zero-risk universal win)

### Task 4: sysmon coverage backend in the shared command, guarded on 3.12+

**Repo:** vergil-tooling · **Depends-on:** Task 2 (equivalence proof must be empty)

**Files:**
- Modify: `src/vergil_tooling/lib/languages.py` — add `test_env_overlay()`.
- Modify: `src/vergil_tooling/bin/vrg_validate.py` — apply the overlay to
  `os.environ` before running the TEST check (it already mutates `os.environ` for
  PATH at ~line 209).
- Test: `tests/vergil_tooling/test_languages.py` (or the existing languages test
  module), `tests/vergil_tooling/test_validate_common.py`.

**Interfaces:**
- Produces: `languages.test_env_overlay(language: str | None, *, python_supports_sysmon: bool) -> dict[str, str]`
  — returns `{"COVERAGE_CORE": "sysmon"}` for `language == "python"` and
  `python_supports_sysmon`, else `{}`. Pure function; the live caller passes
  `sys.version_info >= (3, 12)`.

- [ ] **Step 1: Write the failing test** (pure builder — all branches via injection)

```python
def test_python_sysmon_overlay_on_312():
    assert test_env_overlay("python", python_supports_sysmon=True) == {"COVERAGE_CORE": "sysmon"}

def test_python_no_overlay_below_312():
    assert test_env_overlay("python", python_supports_sysmon=True) or True  # see both arms
    assert test_env_overlay("python", python_supports_sysmon=False) == {}

def test_non_python_never_gets_overlay():
    assert test_env_overlay("go", python_supports_sysmon=True) == {}
    assert test_env_overlay(None, python_supports_sysmon=True) == {}
```

- [ ] **Step 2: Run to verify it fails** → FAIL (`test_env_overlay` undefined).
- [ ] **Step 3: Implement `test_env_overlay`**

```python
def test_env_overlay(language: str | None, *, python_supports_sysmon: bool) -> dict[str, str]:
    if language == "python" and python_supports_sysmon:
        return {"COVERAGE_CORE": "sysmon"}
    return {}
```

- [ ] **Step 4: Wire it into `vrg_validate.py`** — before the TEST-kind run, apply
  `test_env_overlay(language, python_supports_sysmon=sys.version_info >= (3, 12))`
  to `os.environ`. Add a validate-level test asserting `COVERAGE_CORE` is set for
  the python TEST path on 3.12+.
- [ ] **Step 5: Run the full suite + equivalence check** — `vrg-container-run -- vrg-validate`; confirm 100% coverage preserved and Task 2's diff still empty under sysmon.
- [ ] **Step 6: Commit** — `vrg-commit --type feat --scope languages --message "activate sys.monitoring coverage backend on 3.12+ (#333)"`

---

## Phase 2 — xdist promotion (ordered: image → command/config → sweep)

### Task 5: `[test].parallel` config knob (`TestConfig`)

**Repo:** vergil-tooling · **Depends-on:** none (can land alongside Phase 1)

**Files:**
- Modify: `src/vergil_tooling/lib/config.py` — add `TestConfig`, fold into
  `VergilConfig`, parse `[test]`.
- Test: `tests/vergil_tooling/test_config.py`

**Interfaces:**
- Produces: `config.TestConfig(parallel: bool = True)` and
  `VergilConfig.test: TestConfig`. Task 6 reads `cfg.test.parallel`.

- [ ] **Step 1: Write failing tests**

```python
def test_test_config_defaults_parallel_true_when_section_absent():
    cfg = parse_config_from_str(MINIMAL_VALID_TOML)   # no [test] section
    assert cfg.test.parallel is True

def test_test_config_parallel_false_opt_out():
    cfg = parse_config_from_str(MINIMAL_VALID_TOML + "\n[test]\nparallel = false\n")
    assert cfg.test.parallel is False

def test_test_config_non_bool_parallel_raises_config_error():
    import pytest
    with pytest.raises(ConfigError):
        parse_config_from_str(MINIMAL_VALID_TOML + '\n[test]\nparallel = "yes"\n')
```

- [ ] **Step 2: Run to verify they fail** → FAIL.
- [ ] **Step 3: Implement**

```python
@dataclass
class TestConfig:
    parallel: bool = True

# in VergilConfig:
test: TestConfig = field(default_factory=TestConfig)

# in the parser:
test_raw = raw.get("test", {})
_parallel = test_raw.get("parallel", True)
if not isinstance(_parallel, bool):
    raise ConfigError(f"[test].parallel must be a boolean, got {_parallel!r} ({source})")
# ... test=TestConfig(parallel=_parallel)
```

- [ ] **Step 4: Run to verify pass** → PASS.
- [ ] **Step 5: Commit** — `vrg-commit --type feat --scope config --message "add [test].parallel opt-out knob (#333)"`

### Task 6: Computed Python TEST command (`build_python_test_argv` + xdist wiring)

**Repo:** vergil-tooling · **Depends-on:** Task 5; Task 2 (equivalence under `-n auto`)

**Files:**
- Modify: `src/vergil_tooling/lib/languages.py` — add `build_python_test_argv`,
  make the Python `CheckKind.TEST` computed, thread `test_parallel` through
  `language_commands`.
- Modify: `src/vergil_tooling/bin/vrg_validate.py` — pass `cfg.test.parallel` into
  `language_commands`.
- Test: `tests/vergil_tooling/test_languages.py`

**Interfaces:**
- Consumes: `config.TestConfig.parallel` (Task 5).
- Produces:
  `build_python_test_argv(*, python_supports_sysmon: bool, xdist_available: bool, parallel: bool) -> tuple[list[str], dict[str, str]]`.
  Returns `(pytest-argv, env-overlay)`. The env overlay is the same
  `{"COVERAGE_CORE": "sysmon"}` produced by Task 4's helper (call it internally so
  there is one source of truth). `-n auto --dist worksteal` is appended **iff**
  `xdist_available and parallel`. **Note:** `--import-mode=importlib` is NOT added
  here — it lands in Task 10, gated.
- `language_commands(..., *, test_parallel: bool = True)` — new keyword, threaded
  like `cpp_std`/`cpp_stdlib`.

- [ ] **Step 1: Write failing tests** — the full truth table via injected params

```python
import pytest

@pytest.mark.parametrize("sysmon,xdist,parallel,expect_n,expect_cov", [
    (True,  True,  True,  True,  True),
    (True,  True,  False, False, True),   # opt-out honored
    (True,  False, True,  False, True),   # xdist missing → serial, no error
    (False, True,  True,  True,  False),  # <3.12 → no sysmon, still parallel
])
def test_build_python_test_argv_truth_table(sysmon, xdist, parallel, expect_n, expect_cov):
    argv, env = build_python_test_argv(
        python_supports_sysmon=sysmon, xdist_available=xdist, parallel=parallel
    )
    assert ("-n" in argv and "auto" in argv and "worksteal" in " ".join(argv)) is expect_n
    assert (env.get("COVERAGE_CORE") == "sysmon") is expect_cov
    assert "--cov-fail-under=100" in argv          # gate preserved every time
    assert "--import-mode=importlib" not in argv    # deferred to Task 10
```

- [ ] **Step 2: Run to verify it fails** → FAIL.
- [ ] **Step 3: Implement `build_python_test_argv`** — start from the current
  static TEST argv (`--cov=src --cov-branch --cov-fail-under=100 …`), append xdist
  flags conditionally, return the sysmon overlay from Task 4's helper.
- [ ] **Step 4: Wire `language_commands` + `vrg_validate`** — for `(python, TEST)`,
  delegate to `build_python_test_argv` with live probes: `xdist_available` via
  `importlib.util.find_spec("xdist") is not None`, `parallel` from config. Apply
  the returned env overlay (supersedes Task 4's direct call so there's one path).
- [ ] **Step 5: Full validate under `-n auto --dist worksteal`** — `vrg-container-run -- vrg-validate`; 100% coverage preserved, Task 2 diff empty.
- [ ] **Step 6: Commit** — `vrg-commit --type feat --scope languages --message "compute Python TEST command: xdist worksteal gated on config + availability (#333)"`

### Task 7: Remove the repo-local `-n auto` now that the shared command owns it

**Repo:** vergil-tooling · **Depends-on:** Task 6

**Files:**
- Modify: `pyproject.toml` — remove `addopts = ["-n", "auto"]` (the shared command
  now supplies parallelism); keep the `[tool.pytest.ini_options]` block otherwise.

**Why:** #2880 added `-n auto` to this repo's `addopts` as a stopgap. Once Task 6
lands, keeping it means double-specification (and a direct `pytest` run would
diverge from `vrg-validate`). Removing it makes the shared command the single
source of truth. `pytest-xdist` stays in the `dev` group.

- [ ] **Step 1:** Remove the line; run `vrg-container-run -- vrg-validate`; confirm the suite still runs in parallel (via the shared command) at 100% coverage.
- [ ] **Step 2: Commit** — `vrg-commit --type refactor --scope test --message "drop repo-local -n auto; shared command owns parallelism (#333)"`

### Task 8: (operational) Deploy — publish vergil-containers dev image with pytest-xdist

**Repo:** vergil-containers · **Issue:** #587 (`deployment`) · **Depends-on:** the vergil-containers image PR that adds `pytest-xdist`

This is an **operational deployment task**, run via `vergil:issue-deploy`, not a
code task in this repo. Its precondition is a merged vergil-containers PR adding
`pytest-xdist` to the Python dev image; the image publish (release) is
**human-gated**. Closure (`Outcome: SUCCESS`) is the "xdist is live fleet-wide"
signal that gates Task 9's sweep.

- [ ] Land the vergil-containers image change (own PR in that repo: add
  `pytest-xdist` to the Python dev image; verify `python -c "import xdist"`).
- [ ] Run `vergil:issue-deploy` for #587 once the image is published; record
  SUCCESS.

### Task 9: Fleet-sweep xdist consumer (shape-aware, loud report)

**Repo:** vergil-tooling · **Depends-on:** Task 3 (dev-dep-shape survey), Task 8
(image live)

**Files:**
- Create: `src/vergil_tooling/bin/vrg_xdist_sync.py` — new `fleet_sweep` consumer
  (the #328 general-change rail), mirroring `vrg_fleet_sync.py`.
- Create: `src/vergil_tooling/lib/xdist_applicator.py` — the shape-aware applicator.
- Modify: `pyproject.toml` `[project.scripts]` — register `vrg-xdist-sync`.
- Test: `tests/vergil_tooling/test_xdist_applicator.py`

**Interfaces:**
- Consumes: `fleet_sweep.SweepSpec`, `fleet_sweep.run_sweep`,
  `fleet_sweep.AppResult`, and `fleet_survey.DevDepShape` (Task 3).
- Produces: `xdist_applicator.add_xdist(worktree: Path) -> AppResult` — adds
  `pytest-xdist` to the repo's dev deps in the correct shape; returns an
  `AppResult` whose summary says what changed. For `DevDepShape.UNKNOWN`, it makes
  **no edit** and returns an `AppResult` flagged so the driver emits a loud report
  and files a follow-up (per "no silent failures").

- [ ] **Step 1: Write failing tests** — one per shape + the unknown-shape loud path

```python
from pathlib import Path
from vergil_tooling.lib.xdist_applicator import add_xdist

def test_adds_to_uv_dependency_groups(tmp_path: Path):
    (tmp_path / "pyproject.toml").write_text('[dependency-groups]\ndev = ["pytest"]\n')
    r = add_xdist(tmp_path)
    assert r.changed and 'pytest-xdist' in (tmp_path / "pyproject.toml").read_text()

def test_idempotent_when_already_present(tmp_path: Path):
    (tmp_path / "pyproject.toml").write_text('[dependency-groups]\ndev = ["pytest","pytest-xdist"]\n')
    r = add_xdist(tmp_path)
    assert r.changed is False

def test_unknown_shape_reports_loudly_no_edit(tmp_path: Path):
    (tmp_path / "setup.cfg").write_text("[metadata]\nname = weird\n")
    before = list(tmp_path.iterdir())
    r = add_xdist(tmp_path)
    assert r.changed is False and r.needs_followup is True   # loud, not silent
    assert list(tmp_path.iterdir()) == before                 # no edit
```

- [ ] **Step 2: Run to verify they fail** → FAIL.
- [ ] **Step 3: Implement the applicator** — dispatch on `classify_dev_deps`;
  edit uv groups / PEP-621 optional-deps / `requirements-dev.txt` / poetry;
  idempotent; UNKNOWN → `needs_followup`. Extend `AppResult` with `needs_followup`
  if not present (coordinate with `fleet_sweep.AppResult`).
- [ ] **Step 4: Implement the CLI** (`vrg_xdist_sync.py`) — build the `SweepSpec`
  (title/body), drive `run_sweep`; after the sweep, print a prominent summary of
  every `needs_followup` repo and file follow-up issues. `--dry-run` touches no
  git/GitHub state. Register `vrg-xdist-sync` in `[project.scripts]`.
- [ ] **Step 5: Run to verify pass** → PASS; `vrg-container-run -- vrg-validate` green at 100%.
- [ ] **Step 6: Commit** — `vrg-commit --type feat --scope fleet --message "add vrg-xdist-sync: shape-aware pytest-xdist fleet sweep (#333)"`
- [ ] **Step 7 (rollout, human-run):** run `vrg-xdist-sync` over the fleet to open one PR per repo; humans submit/merge. Repos flagged `needs_followup` get filed follow-ups.

---

## Phase 3 — import-mode (gated on survey + measured speedup)

### Task 10: Add `--import-mode=importlib` — only if safe and faster

**Repo:** vergil-tooling · **Depends-on:** Task 3 (collection-safety survey),
Task 1 (measured speedup)

**Files:**
- Modify: `src/vergil_tooling/lib/languages.py` — `build_python_test_argv` appends
  `--import-mode=importlib` under a new `import_mode_importlib: bool` param.
- Test: `tests/vergil_tooling/test_languages.py`

**Decision gate (do this first):** consult Task 3's collection-safety table and
Task 1's `+import-mode` delta.
- If **fleet-wide safe AND measurably faster** → enable unconditionally.
- If **safe only for some repos** → gate at command-build time on a per-repo
  `importlib_safe` probe (reuse `fleet_survey.classify_test_layout`).
- If **no measurable speedup** → **drop this task**; record the decision and the
  numbers in `evidence/baseline.md`. import-mode does not ship on hygiene alone.

- [ ] **Step 1:** Record the gate decision (with numbers) in the evidence file.
- [ ] **Step 2 (if shipping): Write failing test** — `--import-mode=importlib` present iff `import_mode_importlib=True`:

```python
def test_import_mode_appended_only_when_enabled():
    argv_on, _ = build_python_test_argv(python_supports_sysmon=True, xdist_available=True,
                                        parallel=True, import_mode_importlib=True)
    argv_off, _ = build_python_test_argv(python_supports_sysmon=True, xdist_available=True,
                                         parallel=True, import_mode_importlib=False)
    assert "--import-mode=importlib" in argv_on
    assert "--import-mode=importlib" not in argv_off
```

- [ ] **Step 3:** Add the param + append logic; wire the live probe.
- [ ] **Step 4:** `vrg-container-run -- vrg-validate` green; equivalence diff empty.
- [ ] **Step 5: Commit** — `vrg-commit --type feat --scope languages --message "add importlib import-mode, gated on collection-safety + measured speedup (#333)"`

---

## Phase 4 — Subprocess-hotspot removal (bound to a measured target)

### Task 11: Refactor vergil-tooling subprocess hotspots to the measured target

**Repo:** vergil-tooling · **Depends-on:** Task 1 (hotspot map + projection)

**Files:**
- Modify: the top-ranked test files from `evidence/baseline.md` (e.g.
  `tests/vergil_tooling/test_vrg_vm.py` and the other worst offenders).
- Test: the same files (behavior preserved; only the seam changes).

**Stopping rule (measured, not a count):** refactor hotspots — replacing real
`subprocess.run` with mocks where the test exercises *our* logic, not integration —
until the test-stage wall-clock reaches within the agreed margin of the
sysmon+xdist projection from Task 1, **or** the top-N by `--durations` are all
addressed, whichever comes first. Record the achieved wall-clock in the evidence
file.

**Per hotspot (repeat):**
- [ ] **Step 1:** Pick the next-worst hotspot from the map. Read the test — is the subprocess incidental (testing our arg-construction/parsing) or the integration itself?
- [ ] **Step 2:** If incidental, replace with a mock/fake that asserts the argv we build (behavior-preserving). Leave genuine integration tests alone.
- [ ] **Step 3:** Run that file — `uv run pytest tests/vergil_tooling/<file> -v` → PASS, unchanged assertions.
- [ ] **Step 4:** Re-measure with the Task 1 harness; append the new median.
- [ ] **Step 5:** Commit — `vrg-commit --type test --scope perf --message "mock incidental subprocess in <area> (#333)"`
- [ ] **Repeat** until the stopping rule is met; record the final number.

---

## Closing bookends (run via their own skills, not implemented here)

- **Documentation review** (`vergil-tooling#2951`) — sweep the human-facing docs,
  especially `docs/site`: document `[test].parallel`, the sysmon backend, and the
  fleet parallelism default. Spawns per-repo doc tasks (incl. a vergil-containers
  doc note). Run via `vergil:issue-implement` on the review issue.
- **Validation** (`vergil-tooling#2952`) — after the sweep, confirm a sample of
  repos stay green at 100% under parallel order and `import xdist` succeeds in
  each swept container. Run via `vergil:issue-validate`.
- **Retrospective** (`.github#335`) — terminal; authored via
  `vergil:epic-retrospective` once every other child is closed.

---

## Dependency summary

```
Task 1 (baseline) ─┬─▶ Task 4 (sysmon)  [+ Task 2 equivalence]
Task 2 (equiv) ────┘         │
Task 3 (surveys) ──┬─────────┼─▶ Task 9 (sweep) ◀── Task 8 (image live)
                   │         │         ▲
Task 5 (config) ───┼─▶ Task 6 (xdist cmd) ─▶ Task 7 (drop repo-local -n auto)
                   │
Task 3, Task 1 ────┴─▶ Task 10 (import-mode, gated)
Task 1 ───────────────▶ Task 11 (hotspots, measured target)
```
