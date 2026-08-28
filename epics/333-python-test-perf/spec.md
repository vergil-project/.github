# Fleet-wide Python test-suite performance

- **Epic:** vergil-project/.github#333
- **Status:** Reviewed design (2026-08-28) — post-pushback
- **Brainstorm source:** superpowers brainstorming session, 2026-08-28
- **Review:** paad pushback (3 findings, all resolved) — import-mode split out of
  the zero-risk phase and gated on a collection-safety survey + measured speedup;
  fleet-sweep applicator made shape-aware with a loud-report contract (no silent
  no-op); Phase 3/4 hotspot work bound to a measured target
- **Testbed:** vergil-tooling (fleet's largest suite, ~5,100 tests)
- **Shared surface:** `vergil-tooling/src/vergil_tooling/lib/languages.py`
  (Python `CheckKind.TEST` command), consumed by every Python repo via
  `vrg-validate`
- **Prior art:** #2880 (xdist scoped to vergil-tooling only, fleet-wide
  promotion deliberately parked)

## 1. Summary

Every `vrg-validate` run on a Python repo executes its test suite under a
`--cov-branch --cov-fail-under=100` gate. For vergil-tooling — the fleet's
largest suite — this is the dominant cost of the validation gate we run on
nearly every commit. Several well-understood Python test-performance levers are
not yet applied. This epic proves them on vergil-tooling with **measured
numbers**, then lands the winning ones in the **shared** `languages.py` TEST
command so every Python repo benefits, **without changing what the 100%
branch-coverage gate measures**.

The primary optimization target is the **full validation-gate wall-clock** (the
suite always runs under coverage; the fleet almost always runs the full gate,
not a subset). The local inner-loop developer experience is explicitly out of
scope for this epic.

## 2. Goals and non-goals

### Goals

- Materially reduce the test-stage wall-clock of the full validation gate,
  measured as the median of warm runs, for vergil-tooling and fleet-wide.
- Preserve the exact meaning of the 100% branch-coverage gate: identical
  missing-line/branch set before and after every change.
- Land the universal levers in the shared `languages.py` command so all Python
  repos benefit with zero per-repo action.
- Promote parallelism fleet-wide **on-by-default** with a per-repo opt-out, so no
  repo is ever left in a known-broken state.

### Non-goals

- Local inner-loop tooling (pytest-testmon, incremental/affected-only selection,
  plugin-autoload trimming). Valuable, but per-developer ergonomics — a possible
  follow-on epic, not this one.
- Exposing worker-count or distribution-mode tuning knobs (YAGNI — see §5).
- Changing the coverage threshold, the multi-version CI matrix, or any
  non-Python language's TEST command.

## 3. The levers

| Lever | Effect | Correctness surface | Reach |
|---|---|---|---|
| `COVERAGE_CORE=sysmon` | CPython 3.12 `sys.monitoring` coverage backend instead of the C tracer; typically the largest single win under a 100%-coverage suite | None (cannot affect test outcomes or order) | Universal, zero new deps, guarded on interpreter >= 3.12 |
| `-n auto --dist worksteal` | pytest-xdist parallel execution with work-stealing load balancing (better than default `load` when durations are lopsided — one 6.3k-line test file dominates) | Non-deterministic order/process → surfaces hidden shared-state or order leaks | Requires `pytest-xdist` in every image + dev group |
| `--import-mode=importlib` | Cleaner import/collection semantics (import hygiene, **not** a proven speedup) | **Collection-correctness surface** — suites with duplicate test basenames, or unpackaged `tests/` relying on pytest's default `prepend` sys.path insertion, fail to collect under importlib | Universal *only after* a per-repo collection-safety survey (see Phase 0); ships in its own gated step, **not** Phase 1 |
| Subprocess-hotspot removal | Profile-driven refactor of the worst of the ~551 real `subprocess.run` sites in vergil-tooling's suite; mock the ones testing our own logic (not integration) | None (correctness-preserving) | vergil-tooling-specific |

Two levers carry a correctness surface: **xdist** (non-deterministic order,
fleet-wide) and **import-mode** (collection resolution, fleet-wide). xdist is
gated by the `[test].parallel` opt-out; import-mode is gated by a Phase-0
collection-safety survey and a measured speedup before it ships (§7). sysmon is
the only lever with no correctness surface at all.

## 4. Coverage correctness

The 100% branch-coverage gate must not silently change meaning while chasing
speed. Three interactions, each de-risked by a Phase-0 equivalence proof rather
than by trusting a version number:

1. **Branch coverage under `sysmon`.** coverage 7.14.0 is past the point where
   `sys.monitoring` gained branch-coverage support (statement-only support
   arrived in 7.4; branch support followed around 7.7). Phase 0 nonetheless runs
   the suite both ways (C-tracer vs `sysmon`) and **diffs the exact
   missing-line/branch report** — not merely "did it reach 100%." Any divergence
   blocks the sysmon rollout.
2. **`sysmon` + xdist + pytest-cov composition.** Each xdist worker is a separate
   process that reads `COVERAGE_CORE` and writes its own backend-agnostic data
   file; pytest-cov combines them. The Phase-0 diff is also run under `-n auto`,
   so "sysmon + parallel + combine" is proven equal to today's serial number
   before shipping.
3. **Interpreter guard.** `sys.monitoring` exists only on 3.12+; requesting it on
   an older interpreter errors rather than falling back. `vrg-validate` is the
   inner layer — it runs inside the container in the same interpreter that runs
   pytest — so the guard is a runtime `sys.version_info >= (3, 12)` check at the
   point the TEST check is materialized, correct by construction. Phase 0 surveys
   every repo's Python floor; if the fleet is uniformly 3.12+, the guard is
   belt-and-suspenders, but it stays for graceful degradation.

`worksteal` has no coverage interaction — it changes only which worker runs a
test, not what is measured.

## 5. Configuration — the `[test]` knob

A new `[test]` section in `vergil.toml`, parsed into a `TestConfig` dataclass
folded into `VergilConfig` with a default factory, so **every repo without a
`[test]` section gets the aggressive defaults** (on-by-default fleet-wide, zero
per-repo action):

```toml
# vergil.toml — all keys optional; defaults shown
[test]
parallel = true          # -n auto --dist worksteal ; set false to opt out
```

```python
@dataclass
class TestConfig:
    parallel: bool = True
```

- **`parallel` is the only key** — a pure escape hatch. A repo whose suite has a
  hidden order-dependency sets `parallel = false`, degrades to single-process,
  and files an isolation-leak follow-up. No worker count, no dist-mode selection
  (YAGNI; `auto` + `worksteal` is right for every real case, and knobs can be
  added later non-breakingly).
- **No `sysmon`/`import-mode` knob** — those are unconditional universal wins with
  no correctness surface; the only thing that gets an opt-out is the only thing
  with a fleet-wide risk.
- **Config declares intent; `languages.py` is the policy engine.** `parallel =
  true` is a *request*: the command still gates actual `-n auto` on
  `pytest-xdist` being importable, so a missing dependency degrades to serial
  rather than erroring.

## 6. Architecture

The shared TEST command becomes **computed**, not static. The intelligence
(interpreter-version guard, xdist-availability probe) lives in `languages.py`;
config stays a declaration of intent.

The command builder is a **pure, injectable function** so every guard branch is
covered on the 3.12/3.13/3.14 matrix without dead-branch gaps (a runtime
`sys.version_info` read would leave the `< 3.12` arm uncoverable on a 3.12+
runner):

```python
def build_python_test_argv(
    *, python_supports_sysmon: bool, xdist_available: bool, parallel: bool
) -> tuple[list[str], dict[str, str]]:
    """Return (pytest argv, environment overlay) for the Python TEST check."""
```

The real caller passes the live probes (`sys.version_info >= (3, 12)`, an import
check); unit tests drive all corners of the truth table directly.

**Surfaces touched:**

1. `lib/languages.py` — Python `CheckKind.TEST` becomes computed via
   `build_python_test_argv`; injects `--import-mode=importlib`, conditionally
   sets `COVERAGE_CORE=sysmon`, conditionally adds `-n auto --dist worksteal`.
2. `lib/config.py` + `vergil.toml` schema — `TestConfig` / `[test].parallel`.
3. vergil-containers — `pytest-xdist` in the Python dev image.
4. Every Python repo's `dev` dependency group — `pytest-xdist` added via a new
   `fleet_sweep` consumer (the #328 general-change rail).
5. vergil-tooling's own suite — Phase 3 subprocess-hotspot refactor (local).

## 7. Phased rollout (evidence-gated)

Each phase is independently shippable and gated on the prior phase's evidence.

- **Phase 0 — Baseline, profile & fleet survey.** A measurement harness +
  recorded baseline: test-stage wall-clock as the median of N warm runs (reported
  per-check); per-lever cumulative attribution (`baseline → +sysmon → +xdist →
  +worksteal → +import-mode → +hotspot fixes`); the coverage-equivalence proof
  (§4); and a hotspot map (`--durations=25` + `-X importtime`) ranking the worst
  subprocess sites. **Plus two fleet surveys that gate later phases:** (i) a
  **collection-safety survey** — every Python repo's test layout (packaged vs
  unpackaged, duplicate basenames) to determine which repos are importlib-safe;
  (ii) a **dev-dependency-shape survey** — how each Python repo declares dev deps
  (uv `[dependency-groups]`, PEP-621 `[project.optional-dependencies]`,
  `requirements-dev.txt`, poetry groups) so the Phase-2c applicator knows what it
  is editing. Output is numbers + the two survey tables, captured into this epic
  as the evidence record. No behavior change.
- **Phase 1 — sysmon (the one true zero-risk universal win).** `COVERAGE_CORE=sysmon`
  (guarded on interpreter >= 3.12) into `languages.py`. Ships to all repos on next
  run. Gated on Phase 0's coverage-equivalence proof. sysmon alone — because it is
  the only lever with no correctness surface.
- **Phase 2 — xdist promotion**, ordered so xdist is present before any repo is
  told to use it:
  - **(a)** vergil-containers: `pytest-xdist` into the Python dev image
    (**deployment task** #587 tracks the "image is live" signal).
  - **(b)** `languages.py`: `-n auto --dist worksteal` (guarded on
    xdist-importable + `[test].parallel`); `TestConfig` in `config.py`. The shared
    command becomes the **single source of truth for gate parallelism**, so
    vergil-tooling drops its repo-local `addopts = ["-n", "auto"]` (#2880). Accepted
    tradeoff: a bare `pytest` run (not via `vrg-validate`) then executes serially —
    acceptable because the epic optimizes the **validation gate**, and the inner
    loop is explicitly out of scope (§2).
  - **(c)** fleet-sweep consumer adds `pytest-xdist` to every repo's dev deps —
    one PR per repo. The applicator **handles each dev-dependency shape from the
    Phase-0 survey**, and for any repo whose shape it cannot safely edit it emits
    a **loud per-repo report and files a follow-up**, never a silent no-op (a
    silent skip would leave the repo carrying the on-by-default parallel config
    while degrading to serial with no signal — a silent failure). A repo that
    breaks under parallel order sets `[test].parallel = false` and files an
    isolation-leak follow-up; never left red.
- **Phase 3 — import-mode (gated).** Add `--import-mode=importlib` to the shared
  command **only after** the Phase-0 collection-safety survey confirms fleet-wide
  safety (or restricts it to the safe repos, detected at command-build time)
  **and** a measured speedup justifies it. If Phase 0 shows no measurable gain,
  this phase is dropped — import-mode is the weakest lever and does not ship on
  hygiene alone.
- **Phase 4 — Hotspot removal.** Refactor vergil-tooling's subprocess hotspots
  from the Phase-0 map, **bound to a measured target**: refactor until the
  test-stage wall-clock reaches within an agreed margin of the sysmon+xdist
  projection, or until the top-N sites by `--durations` are addressed, whichever
  comes first — done-ness ties to Phase-0 evidence, not to an open-ended count.
  Pure vergil-tooling, correctness-preserving, measured against the baseline.

## 8. Testing strategy

- **Command-builder unit tests** — table-driven over the truth table
  (sysmon-capable × xdist-available × parallel), asserting the presence/absence of
  `COVERAGE_CORE=sysmon`, `-n auto --dist worksteal`, and `--import-mode=importlib`.
  All four corners covered via injected parameters — no mocking of
  `sys.version_info` or the import machinery, only injection of their results.
- **`TestConfig` parsing tests** — missing `[test]` → `parallel=True`; explicit
  `false`/`true`; malformed value raises `ConfigError` in the existing style.
- **Fleet-sweep consumer tests** — idempotent (already-present → no PR), adds when
  absent, dry-run touches no git/GitHub state; **one case per dev-dependency shape**
  (uv `[dependency-groups]`, PEP-621 optional-dependencies, `requirements-dev.txt`,
  poetry groups); and an **un-editable-shape case asserting a loud report + filed
  follow-up, never a silent no-op**. Follows the `vrg-fleet-sync` test patterns.
- **Not permanent tests, by design:** the coverage-equivalence proof (a Phase-0
  gate producing evidence, run once per configuration, not a suite fixture that
  would double CI cost) and the vergil-containers image verification (owned by
  that repo's CI — image builds, `import xdist` succeeds).

## 9. Risks and mitigations

- **A repo's suite breaks under parallel order.** Mitigated by on-by-default +
  `[test].parallel = false` opt-out and the per-repo-PR sweep — one repo's break
  never blocks others and never lands known-broken. The validation task (#2952)
  samples repos green under parallel post-rollout.
- **A repo silently never receives xdist** (unhandled dev-dependency shape) and
  degrades to serial while carrying the parallel config. Mitigated by the Phase-0
  dev-dependency-shape survey and the applicator's **loud-report-and-follow-up**
  contract (never a silent no-op), reinforced by the validation task (#2952)
  asserting `import xdist` succeeds in each swept repo's container.
- **import-mode breaks collection in some repo.** Mitigated by gating import-mode
  (now Phase 3) behind the Phase-0 collection-safety survey and a measured
  speedup — it never ships in the no-opt-out sysmon phase.
- **sysmon changes the measured coverage set.** Mitigated by the Phase-0
  equivalence proof gating the Phase-1 rollout.
- **A fleet repo targets < 3.12.** Mitigated by the interpreter guard (sysmon set
  only on 3.12+) and the Phase-0 fleet Python-floor survey.
- **Container-image dependency ordering.** Mitigated by Phase 2's explicit
  ordering (image first, command+config next, sweep last) and the deployment task
  gating the sweep.

## 10. Acceptance criteria

- Phase 0 evidence record (baseline, per-lever deltas, equivalence proof, hotspot
  map, **collection-safety survey, dev-dependency-shape survey**) committed to
  this epic.
- `sysmon` lives in the shared `languages.py` command; coverage report proven
  identical to the C-tracer baseline.
- `pytest-xdist` present in the vergil-containers dev image (deployment #587
  closed SUCCESS) and in every Python repo's dev deps — or, for any repo whose
  dependency shape the applicator can't edit, a filed follow-up (never a silent
  skip).
- `-n auto --dist worksteal` on-by-default via the shared command, with a working
  `[test].parallel = false` opt-out.
- `--import-mode=importlib` either shipped (fleet-wide or restricted to
  survey-confirmed-safe repos) with a measured speedup, **or** explicitly dropped
  with the Phase-0 evidence recording why.
- vergil-tooling's subprocess hotspots refactored to the Phase-4 measured target
  (within the agreed margin of the sysmon+xdist projection, or top-N by duration).
- 100% branch coverage preserved across the 3.12/3.13/3.14 matrix throughout.
- Validation task #2952 records SUCCESS (sampled repos green under parallel;
  `import xdist` confirmed in each swept repo's container).
