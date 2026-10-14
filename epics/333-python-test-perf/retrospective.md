# Retrospective — Fleet-wide Python test-suite performance

> Epic: [`vergil-project/.github#333`](https://github.com/vergil-project/.github/issues/333) ·
> Partners [`spec.md`](./spec.md) and [`plan.md`](./plan.md) at the documentation tier.
> Read the three in order: **spec → plan → retrospective**.

## §0 At a glance

We set out to make the `vrg-validate` test stage materially faster for every
Python repo without changing what the 100% branch-coverage gate measures. The
spec named four levers: the `sys.monitoring` coverage backend, pytest-xdist
worksteal parallelism, `--import-mode=importlib`, and subprocess-hotspot
removal. Each lever had to pass a measurement gate before it could ship. **Only
parallelism shipped, and it delivered all of the measured gain:** vergil-tooling's
test stage went from **23.47s to 7.41s (3.17×)**, and serial and parallel runs
produced an identical coverage missing-set. The shared Python TEST command now
adds `-n auto --dist worksteal` whenever pytest-xdist is importable and the repo
has not set `[test].parallel = false`. pytest-xdist ships in the published
dev-python images and in the dev dependencies of every active Python repo. The
other three levers were **dropped on measured evidence**: sysmon does not run
under branch coverage, import-mode was neither faster nor safe, and no
subprocess hotspot was left to remove. The Phase-0 gates exist to catch exactly
these cases, and they did.

### Work delivered

| PR | Repo | What it did |
|---|---|---|
| [.github#336](https://github.com/vergil-project/.github/pull/336) | .github | Spec + plan (epic opening bookend) |
| [tooling#2963](https://github.com/vergil-project/vergil-tooling/pull/2963) | vergil-tooling | **T1**: test-stage measurement harness, baseline, and hotspot map |
| [tooling#2964](https://github.com/vergil-project/vergil-tooling/pull/2964) | vergil-tooling | **T3**: `lib/test_layout` + `lib/dev_deps` classifiers and the fleet survey |
| [tooling#2965](https://github.com/vergil-project/vergil-tooling/pull/2965) | vergil-tooling | **T5**: `[test].parallel` opt-out knob (`TestConfig`) |
| [tooling#2971](https://github.com/vergil-project/vergil-tooling/pull/2971) | vergil-tooling | **T2**: coverage-equivalence proof (serial vs parallel: empty diff) |
| [containers#589](https://github.com/vergil-project/vergil-containers/pull/589) | vergil-containers | Add pytest-xdist to the Python dev image (published and verified via deploy task #587) |
| [tooling#2973](https://github.com/vergil-project/vergil-tooling/pull/2973) | vergil-tooling | **T6**: computed Python TEST command (`build_python_test_argv`, xdist worksteal gated on config + availability) |
| [tooling#2974](https://github.com/vergil-project/vergil-tooling/pull/2974) | vergil-tooling | **T7**: drop the repo-local `-n auto`; the shared command owns gate parallelism |
| [tooling#2972](https://github.com/vergil-project/vergil-tooling/pull/2972) | vergil-tooling | **T9**: `vrg-xdist-sync`, the shape-aware pytest-xdist fleet sweep |
| [tooling#2978](https://github.com/vergil-project/vergil-tooling/pull/2978) | vergil-tooling | **Docs review**: document `[test].parallel` and the fleet-wide parallel default |
| [containers#594](https://github.com/vergil-project/vergil-containers/pull/594) | vergil-containers | **Docs**: note that the dev-python image ships pytest-xdist |
| [mq-gateway-replay-lab#295](https://github.com/logical-minds-foundry/mq-gateway-replay-lab/pull/295) | mq-gateway-replay-lab | **Rollout**: add pytest-xdist (sweep output) |
| [mq-resiliency-lab-for-linux#1053](https://github.com/logical-minds-foundry/mq-resiliency-lab-for-linux/pull/1053) | mq-resiliency-lab-for-linux | **Rollout**: add pytest-xdist (sweep output, lock refreshed by hand) |
| [tooling#3197](https://github.com/vergil-project/vergil-tooling/pull/3197) | vergil-tooling | **Fix** (#2979): resolve relative `--repos` paths (false `UNKNOWN`, surfaced live) |
| [tooling#3198](https://github.com/vergil-project/vergil-tooling/pull/3198) | vergil-tooling | **Fix** (#2980): refresh `uv.lock` after the xdist edit; refuse Poetry repos (surfaced live) |
| *(this PR)* | .github | Retrospective (closing bookend; closes the epic) |

- **Repos touched:** 5. Core work was in `vergil-tooling`, `vergil-containers`,
  and `.github`; rollout PRs landed in two `logical-minds-foundry` repos
  (`mq-gateway-replay-lab`, `mq-resiliency-lab-for-linux`).
- **Tasks:** 17 epic children: 11 planned tasks (T1–T11, including the
  deployment task #587), the container image task #588 and its docs note #593,
  and four bookends (spec/plan docs, docs review, validation, retrospective). Of the 11, **3 were closed `NOT_PLANNED` on
  evidence** (#2958 import-mode, #2959 hotspots, #2960 sysmon).
- **Releases:** vergil-tooling cut v2.1.210–v2.1.212 while the core work was
  merging (2026-08-28/29). This is a date match: which PR landed in which
  release was not checked. The two live-surfaced sweep fixes shipped in v2.1.239
  (2026-10-13).
- **Span:** opened 2026-08-28. Implementation finished 2026-08-29 (~2 days), the
  rollout fixes landed 2026-10-13, and the epic closed 2026-10-14.
- **Validation:** [tooling#2952](https://github.com/vergil-project/vergil-tooling/issues/2952)
  closed SUCCESS on 2026-10-14. vergil-tooling `develop` passed the full gate
  under `-n auto --dist worksteal` (6,559 passed, 100.00% branch coverage) and the
  3.12/3.13/3.14 CI legs. A follow-up comment corrects the validation's repo scope.

## §1 How the plan evolved

The plan had no "Evolution during execution" log. The record of how it changed
is in the closure comments on #2958/#2959/#2960 and in the Phase-0 evidence
(`vergil-tooling: epics/333-python-test-perf/evidence/`).

- **Phase 0 shrank the epic from four levers to one.** The plan ordered the work
  so the measurement harness (T1) and the equivalence proof (T2) came before any
  lever shipped. Measurement contradicted two of the spec's premises:
  - **sysmon (spec §4.1).** The spec assumed coverage.py had supported
    `sys.monitoring` branch measurement since about 7.7. In practice, coverage
    7.14 on CPython 3.12 printed `no-sysmon: sys.monitoring can't measure
    branches` and **silently fell back to the C tracer**. Measured speedup was
    1.02×, which is noise, so Phase 1 (#2960) was dropped. The C-tracer-vs-sysmon
    equivalence diff was empty only because both sides ran the C tracer, and the
    evidence records it as trivially empty rather than as a proof.
  - **Subprocess hotspots (spec §3).** The spec cited "~551 real
    `subprocess.run` sites". They were almost all `patch("subprocess.run")` mock
    targets: the suite had **12** real inline subprocess calls, and the slowest
    test took 0.33s. Parallelism alone already met Phase 4's stopping target, so
    #2959 was dropped.
  - **import-mode** (#2958) failed both conditions of its own gate. It gave no
    measurable speedup (7.42s vs 7.41s), and it broke collection (rc=1, because
    `scripts.perf` resolved only through prepend-mode `sys.path`). Dropped.
- **Parallelism became the whole deliverable, and it held.** T5→T6→T7 (knob →
  computed command → drop the repo-local flag) shipped as planned. The
  serial-vs-parallel coverage diff was empty, which is the proof that mattered.
- **The fleet sweep worked only after live runs.** T9's driver was unit-tested
  with mocked git/subprocess. The first real rollout over `logical-minds-foundry`
  hit two defects that the mocks hid: relative `--repos` paths sent the
  applicator to an empty directory, which classified real repos as `UNKNOWN` and
  filed 3 false follow-up issues (#2979); and edits never refreshed `uv.lock`, so
  every rollout PR failed the audit gate until someone ran `uv lock` by hand
  (#2980). Both were fixed six weeks later (v2.1.239).
- **"Every Python repo" became "every active Python repo."** The fleet survey
  (T3) covered only the vergil-project clones on disk, where vergil-tooling was
  the only Python repo, and its own text asked for a re-run against a fuller set
  of clones before the sweep. That re-run was not recorded. At close the fleet has
  seven Python repos. The three active ones carry pytest-xdist. The other four are
  out of scope by maintainer decision (see §3).

## §2 Lessons learned

1. **Evidence gates paid for themselves.** Phase 0 cost about one task's worth of
   effort, and it kept two levers that do nothing and one that breaks collection
   out of the shared command for every Python repo. Measure-before-shipping should
   stay the default for any performance epic. Plans should include
   "drop on evidence" as an explicit, cheap outcome, as this one did.
2. **Spec premises about third-party behaviour need a citation or a probe.** Both
   wrong premises (sysmon branch support, 551 real subprocess sites) were stated
   as fact in the spec. A one-line check would have caught each one before
   planning: the coverage.py `no-sysmon` docs, and a grep separating real calls
   from mocks. Next time, a spec's quantitative or third-party claims should carry
   a source or be listed as hypotheses for Phase 0 to test.
3. **Mocked tests can't prove a driver whose job is real side effects.** All three
   sweep defects found live (#2944, which came before this epic, plus #2979 and
   #2980) were invisible because git, subprocess, and `uv` were mocked. A
   fleet-sweep consumer needs at least one integration test against a real
   throwaway git repo. The dry-run also never calls the applicator, so it can't
   preview skip, follow-up, or change (filed as .github#371).
4. **"The fleet" needs a definition the epic writes down.** The survey's
   local-clones scope, the sweep's `--repos` list, and the validation's sample
   each used a different idea of the fleet. The validation result on #2952 first
   said vergil-tooling was "the fleet's only Python repo" and had to be corrected.
   A cross-org epic should list its target repos (and the excluded ones, with
   reasons) in the plan.
5. **Validation scaffolds should be filled in when the task is filed.** #2952 sat
   with the blank template (placeholder preconditions, commands, and criteria)
   until it was run. Its checklist had to be reconstructed from one line in the
   plan, so its acceptance bar was set at run time rather than up front.

## §3 Compromises & tradeoffs

- **Four Python repos run serially.** `diogenes-project/diogenes`,
  `mnemosys-project/melete`, and `mq-rest-admin-project/mq-rest-admin-python` are
  on hold. `logical-minds-foundry/mq-resiliency-observability` is about to be
  rebuilt from scratch, so its open sweep PR (#37) was not taken. The maintainer
  decided these repos will catch up on current vergil standards, including
  running `vrg-xdist-sync`, when they become active again. This is safe: the
  shared command adds `-n auto` only when xdist is importable, so these repos
  lose the speedup but don't break. It does depart from the spec's "every Python
  repo's dev deps" acceptance criterion.
- **Two spec acceptance criteria were knowingly not met.** "sysmon lives in the
  shared command" and "subprocess hotspots refactored to the measured target"
  were replaced by evidence-backed drops. The spec's §10 allows this explicitly
  for import-mode only. For sysmon and hotspots the drop was a driver decision
  recorded on the closed issues, not a pre-agreed outcome.
- **The validation sample is thin.** The full local gate was run only on
  vergil-tooling. mq-resiliency-lab-for-linux is covered by CI on its rollout
  PR and on current `develop`. mq-gateway-replay-lab's rollout PR (#295) has no
  recorded CI checks, so it was not independently re-validated.
- **The rollout fixes lagged the rollout by six weeks.** #2979 and #2980 were
  filed during the late-August rollout and fixed on 2026-10-13. In between, the
  sweep was known to be unsafe with relative paths and to produce PRs that fail
  the audit gate.
- **Measurement scripts stay in the tree.** `scripts/perf/` (the harness and the
  equivalence proof) is kept as a re-runnable evidence record, outside the 100%
  coverage gate apart from the unit-tested pure helpers.

## §4 New problems & opportunities

| Item | Where it went |
|---|---|
| `vrg-xdist-sync --dry-run` reports every repo `ready` without running the applicator, so it can't preview skip or follow-up, and a live run over non-Python repos files spurious follow-ups | [.github#371](https://github.com/vergil-project/.github/issues/371) (triage) |
| Relative `--repos` paths caused false `UNKNOWN` classification in every `fleet_sweep` consumer | Fixed: tooling#2979 → #3197 |
| Sweep left `uv.lock` stale; rollout PRs failed the audit gate | Fixed: tooling#2980 → #3198 |
| The manual lock refresh on mq-resiliency-lab-for-linux#1053 also moved the repo to Python 3.14. A dev box still on 3.12 then misreported PEP 758 `except A, B:` syntax as a regression (#1099, closed as a misdiagnosis). Rollout PRs should stay single-purpose. | Logged, not yet acted on |
| sysmon could become viable once a coverage.py/CPython combination measures branches under `sys.monitoring` (the gate's CI matrix now includes 3.14). Not verified here. | Logged, not yet acted on |
| Out-of-scope repos (above) still need pytest-xdist | Deferred to each repo's re-activation catch-up |

## §5 What's next

No follow-on brainstorm or epic was spawned. The two open threads are the
dry-run fidelity fix (.github#371, awaiting `triage-review`) and the xdist
catch-up for the four deferred repos when they return to active development.

## Appendix A — Operational notes

- **Order that worked:** image PR (containers#589) → publish dev-python images →
  deploy verification (#587: `import xdist` by digest on 3.12/3.13/3.14) → shared
  command (tooling#2973/#2974) → per-repo sweep. Repos without xdist degrade to
  serial, so the image and the command can ship before the sweep.
- **Running the sweep:** pass **absolute** `--repos` paths (required before
  v2.1.239), run from a host with `uv` on `PATH` (v2.1.239+ refreshes
  `uv.lock`), and pass only Python repos. Until #371 is fixed, `--dry-run` won't
  show which repos would be skipped or flagged.

## Appendix B — Extended metrics

Phase-0 baseline, from vergil-tooling `epics/333-python-test-perf/evidence/baseline.md`:
CPython 3.12.14 (aarch64), 5,192 tests, median of 5 warm runs after 1 warm-up.

| Configuration (cumulative) | Median (s) | Speedup |
| --- | ---: | ---: |
| baseline (serial, C tracer) | 23.47 | 1.00× |
| +sysmon (serial; fell back to C tracer) | 23.01 | 1.02× |
| +xdist `-n auto` | 7.71 | 3.04× |
| +worksteal | 7.41 | 3.17× |
| +import-mode (rc=1) | 7.42 | 3.16× |

At close (2026-10-14, `develop` @ `c352d3c86`): 6,559 tests, test stage 18.50s
under `-n auto --dist worksteal`, full `vrg-validate` 22s, 100.00% branch
coverage. The suite grew by ~26% and the host differs, so this is not
directly comparable to the baseline above.
