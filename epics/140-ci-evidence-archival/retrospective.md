# CI Evidence Archival — Retrospective

> Backward-looking record partnering [`spec.md`](spec.md) and [`plan.md`](plan.md).
> Read the three in order: what we set out to do, how we planned it, and — here —
> honestly how it went.

## §0 At a glance

**Set out to do:** publish durable, machine-verifiable proof of every CI gate that
validated a release — a compressed, attested evidence bundle attached to each
GitHub Release, with evidence completeness enforced as a publish precondition
("if we cannot prove it, we do not publish").

**What shipped:** the full mechanism — a `ci-evidence-<gate>` producer convention
in `vergil-actions`, a language-agnostic `vrg-ci-evidence` harvester in
`vergil-tooling`, `cd-release` wiring with build-provenance attestation, and a
doc-site link. It baked in warning mode across ~54 releases, was promoted to
**enforcing**, and — after a late but decisive correction — now attaches bundles
that carry **real, per-version report payloads for every gate**, guarded so the
empty-payload class of defect cannot silently return.

**The story in one line:** the mechanism worked for a month while quietly
publishing *empty envelopes* for three of four gates; the closing docs-review
sweep caught it, and the epic did not close until it actually delivered its
promise.

### Work delivered

| Dimension | Value |
|---|---|
| Span | opened 2026-07-12 → retrospective 2026-08-13 (~1 month) |
| Repos touched | 6 (vergil-tooling, vergil-actions, vergil-containers, vergil-claude-plugin, vergil-vm, .github) |
| Tasks / PRs (non-retrospective children) | 32 |
| First real bundle | v2.1.138 (2026-07-14) |
| Enforcing + payloads verified | v2.1.194 / v2.1.195 (2026-08-12/13) |

**Task distribution:** vergil-tooling 16 · vergil-actions 9 · vergil-containers 3
· vergil-vm 1 · vergil-claude-plugin 1 · .github 2 (bookends). Full graph in
Appendix B.

**The closing arc** (the part worth naming individually — the delta this
retrospective exists to capture):

| PR / task | What it did |
|---|---|
| vergil-actions#831 | Promoted the evidence gate warning→enforcing (`evidence-enforce` false→true) — the planned T11 that had never been filed |
| vergil-tooling#2811 | Emit machine-readable quality report files (ruff JSON, mypy JUnit-XML) from the check registry |
| vergil-actions#836 | **Root-cause fix:** bridge per-matrix report files across the CI job boundary into the evidence artifacts |
| vergil-tooling#2812 | Harden completeness from artifact-presence to report-**payload**-presence |
| vergil-tooling#2829, vergil-actions#842, vergil-containers#545 | Reconcile the docs with shipped reality (per-version filenames, enforcing lifecycle, payload rule) |

## §1 How the plan evolved

The build phase (T1–T12) tracked the plan closely: the derived gate set, the pure
manifest/bundle core, the harvest layer, the completeness validator, the
`vrg-ci-evidence` CLI, the producer convention, and the warning-mode `cd-release`
wiring all landed roughly as written, with the expected crop of small operational
fixes (the `actions: read` grants across every consuming repo's `cd.yml`; the
gate/attach split in #776 when evidence-first ordering collided with when the
Release and SBOM actually exist; the harvest/assemble perf split; a SARIF
partial-artifact crash). None of these were surprises — they are the normal
texture of wiring a fleet-wide pipeline.

Two deviations were substantive, and both surfaced only at the very end:

1. **The enforcing promotion (T11) fell through the cracks.** The plan named it
   explicitly — "a single, human-gated flag flip" — but it was never filed as an
   issue, so warning mode, which the design intended as a *temporary deployment
   state*, quietly became the *de facto* steady state for a month. It was
   recovered and executed (#831) during close-out.

2. **Warning mode proved the pipeline, not the payload.** The bake accumulated 54
   consecutive releases that each attached a bundle — and every one of them was
   an empty envelope for `test`, `audit`, and `quality`. Only `security` carried
   real files. The root cause was a job-boundary data loss: the producer gates
   emit their report files in one CI job, but the evidence artifact is assembled
   in a *separate* job with a clean checkout, and — unlike `ci-security.yml`,
   which correctly uploads/downloads across that boundary — `ci-test/audit/quality`
   never bridged the files. The completeness validator, which checked only that a
   gate's artifact *existed*, waved the empty envelopes through green even under
   enforcing mode. Fixing it took the producer bridge (#836), quality report
   emission that never existed (#2811), and a completeness check strengthened to
   require an actual report payload (#2812).

## §2 Lessons learned

- **"Attached" is not "complete." Verify contents, not existence.** The entire
  failure rests on one gap: acceptance for the warning-mode deployment (T9)
  confirmed *a bundle attached*, never *the bundle contained real per-gate data*.
  A single assertion — download the tarball, confirm each gate's report files
  exist and are non-empty — would have caught this in week one instead of week
  four. This is the transferable rule: when a gate's job is to collect evidence,
  test the evidence, not the collection.
- **A warning-mode bake earns trust in the plumbing, not the cargo.** 54 green
  releases proved harvest/attest/attach were reliable. They said nothing about
  whether the harvested data was meaningful, because nothing checked it. Bake
  criteria must assert the thing the gate ultimately guarantees.
- **A working reference in the same repo is worth copying deliberately.**
  `ci-security.yml` had the correct upload→download bridge the whole time. The
  other three gates were structurally similar but skipped it. Divergence from an
  in-repo working pattern is a smell worth a checklist.
- **A planned terminal step with no issue is a step that won't happen.** T11 was
  in the plan prose but not in the tracker, so it evaporated. Terminal
  promotions/flips deserve a filed, owned issue like any other task.

## §3 Compromises & tradeoffs

- **Known residuals shipped, tracked not hidden.** The manifest `metrics` object
  is empty `{}` for all gates and the `security` gate's `evidence.json` `tools`
  is empty `[]`. The *data* an auditor needs now lives in the report files;
  `metrics` is a convenience summary not yet extracted. Documented as current
  limitations rather than papered over.
- **The payload guard keys on report-file presence, not on `metrics`/`tools`.**
  Requiring populated `metrics` would have false-failed every gate today, and
  requiring `tools` would false-fail `security`. So #2812 asserts "≥1 report file
  besides `evidence.json`" — the minimal check that actually catches the
  empty-envelope class without over-firing. A subtle detail carried the fix:
  `GateEvidence.files` always includes `evidence.json` (built via `rglob`), so the
  guard must exclude it.
- **Deliberate deploy sequencing under an already-enforcing gate.** #2812 was held
  until the producer fixes were released and a real bundle was verified — shipping
  the stricter check first would have hard-failed every release while payloads
  were still empty. `evidence-enforce=false` remains the per-repo escape hatch.

## §4 New problems & opportunities

- **Empty-payload defect class — fixed and guarded.** Root-caused, corrected
  across producer + validator, verified on two consecutive releases (v2.1.194/195),
  and locked by the completeness guard so it cannot silently recur.
- **`metrics` / `tools` enrichment — logged, not yet acted on.** Populating the
  manifest `metrics` (coverage %, test/finding counts) and the security `tools`
  array is real polish worth a follow-up task; not yet filed.
- **Adjacent release-doc drift — logged, not yet acted on.** The sweep flagged two
  unrelated staleness items outside this epic's scope: vergil-claude-plugin's
  README references releasing via `vrg-publish` (CD is driven by
  `cd-release.yml@v2.1`), and vergil-containers' README references
  `docker-publish.yml` (actual: `cd-docker-publish.yml`). Worth separate triage.
- **Opportunity — a content-assertion in the gate-deployment lifecycle.** The
  all-hard-gates lifecycle doc should arguably require, as bake-in exit criteria,
  a positive assertion of the gate's *output*, not just its *execution*. This epic
  is the case study.

## §5 What's next

The follow-on brainstorm bookend was folded here rather than run separately — the
forward-looking items are small and concrete, captured above in §4 and echoed on
the closed bookend (#144):

- Populate manifest `metrics` and the security `tools` array (follow-up task).
- Triage the two adjacent README release-doc drifts.
- Consider adding an output-content assertion to future hard-gate bake criteria.

None of these rise to a new epic; each is a task to file when prioritized.

## Appendix A — Operational notes

The correction had a deploy order that mattered, because `cd-release` runs the
*previously released* tooling version, so each change activates one release cycle
later:

1. **T11 flip** (vergil-actions#831) → release; the gate becomes enforcing at
   `@v2.1`.
2. **Producer fixes** (vergil-tooling#2811 quality reports, vergil-actions#836
   bridge) → release. First correct bundle appears one cycle later — v2.1.193
   (whose CI ran pre-fix) was still empty; **v2.1.194** was the first populated
   bundle.
3. **Verify a real tarball** — not just the manifest — carries per-version report
   files for every gate. (v2.1.194 confirmed: `coverage-3.{12,13,14}.xml`,
   `junit-*`, `pip-audit-*`, `licenses-*`, `quality-ruff-*`, `quality-mypy-*`,
   plus security SARIF; `missing_gates: []`.)
4. **Payload guard** (vergil-tooling#2812) → release **last**, after step 3, so it
   never hard-fails a release while payloads are still empty. Verified on v2.1.195.

Rollout watch-item: the guard tightens the enforcing gate fleet-wide; each
consuming repo's *next* release is the first to run it, so confirm each repo's
required gates emit payloads (or set `evidence-enforce=false`) before it bites.

## Appendix B — Full task graph

- **vergil-tooling (16):** #2289 derive gate set · #2290 manifest/bundle core · #2291 report files from registry · #2301 harvest layer · #2306 completeness validator · #2311 `vrg-ci-evidence` CLI · #2316 doc-site link · #2317 convention doc · #2319 perf: batched asset lookups · #2320 `actions: read` · #2330 perf: harvest/assemble split · #2335 triage: warning-mode bundling failure · #2340 fix: SARIF partial-artifact crash · #2811 quality report emission · #2812 payload completeness guard · #2829 convention-doc reconciliation.
- **vergil-actions (9):** #762 producer artifacts · #763 all-hard-gates doc · #769 cd-release wiring (warning mode) · #770 fix: GH_TOKEN for docs stage · #776 fix: gate/attach split · #781 perf: shared harvest · #831 promote to enforcing (T11) · #836 producer bridge (root-cause fix) · #842 all-hard-gates doc → enforcing.
- **vergil-containers (3):** #404 / #408 `actions: read` grants · #545 architecture docs.
- **vergil-vm (1):** #281 `actions: read`.
- **vergil-claude-plugin (1):** #643 `actions: read`.
- **.github (2):** #143 docs-review bookend · #144 follow-on brainstorm bookend.
