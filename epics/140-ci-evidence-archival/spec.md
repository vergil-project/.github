# CI evidence archival — durable, attested gate evidence per release

- **Epic:** vergil-project/.github#140
- **Status:** Approved design (2026-07-12) — superpowers brainstorming, hardened
  via paad pushback (7 issues resolved) the same day. Amended 2026-07-13: the
  gate deploys in **warning mode** first and is promoted to enforcing via a
  human-gated flip after a bake-in (§9.2, §14.1) — a staged deployment of a hard
  gate, not a change to the all-hard-gates requirement.
- **Related work:** `vrg-release` idempotency/retriability epic (forthcoming) —
  this epic is engineered to be forward-compatible with it (§12.1)
- **Origin issue:** vergil-project/vergil-tooling#120 (was filed under the
  ad-hoc epic vergil-project/.github#99; promoted to a finite epic here)
- **Brainstorm source:** superpowers brainstorming session, 2026-07-12

## 1. Summary

Every release runs an extensive set of CI gates — CodeQL, Trivy, and Semgrep
security scans; 100%-enforced coverage; dependency and license audits; the full
test suite. That work proves a release is secure and stable, but the proof is
**ephemeral**: GitHub Actions logs and artifacts expire after roughly 90 days,
SARIF lands in the Security tab with no stable per-release URL, and coverage and
audit results are pass/fail only with no persisted report. A consumer who pulls
a release six months later has no way to verify it was as trustworthy as claimed.

This epic makes that evidence **durable, complete, and machine-verifiable**. At
publish time, the release harvests every gate's full output, bundles it into a
compressed, self-describing archive, cryptographically attests the archive to the
pipeline and the exact released commit, and attaches it to the GitHub Release —
where it lives permanently, independent of Actions retention. The doc-site
release page links to it. Evidence completeness becomes a **publish
precondition**: if the pipeline cannot assemble complete, attested evidence, the
release does not publish.

The design is **convention-based and applies across every release-publishing
repo by construction**: gates emit evidence under a naming convention, and a
single language-agnostic harvester bundles whatever was emitted, so every repo
that cuts releases benefits without per-language code. Scope is bounded to repos
that publish releases through `cd-release` (§6.1); repos that do not cut package
releases — e.g. `.github`, docs-only repos — are out of scope by construction,
because the evidence step only ever runs inside `cd-release`.

## 2. Goals and non-goals

### Goals

- Capture the **complete output** of every CI gate that validated a release —
  not summaries, the actual reports (SARIF, coverage, JUnit, audit JSON, SBOM).
- Store that evidence **durably** — surviving Actions log/artifact expiry — as an
  asset on the GitHub Release.
- Make the bundle **self-describing and machine-verifiable** — a `manifest.json`
  index with per-file `sha256`, so an automated auditor can confirm what ran,
  with what result, against which commit, and that the archive is intact.
- **Cryptographically bind** the bundle to the repo's pipeline and the released
  commit (build-provenance attestation) — a verifiable chain of custody.
- Enforce a **publish invariant**: a published release always has complete,
  attested evidence. Missing evidence hard-fails the release.
- Surface a simple **"full CI evidence available" link** on each doc-site
  release page.
- Work across **every release-publishing repo** and all language ecosystems with
  no per-language harvesting code.

### Non-goals

- **No live, browsable evidence UI.** The realistic consumer is an automated
  auditor (an AI or a script), not a human clicking through logs. A durable,
  downloadable, machine-readable bundle is the deliverable; an indexed browse
  experience is explicitly out of scope (revisitable later — see §15).
- **No archiving of raw multi-megabyte logs into git history.** Evidence lives on
  Release assets, never committed to the repo (see §4.3).
- **No retroactive backfill** of evidence for releases published before this
  epic ships.
- **No new gates.** This captures the output of the gates that already run; it
  does not add or change what is scanned.

## 3. Background — how releases work today

- **Release notes.** git-cliff generates `releases/v{version}.md` in each repo.
- **CI gates.** `ci.yml` calls reusable workflows in `vergil-actions`
  (`ci-audit`, `ci-quality`, `ci-security`, `ci-test`, `ci-version`, `ci-docs`)
  on `pull_request` and `workflow_call`. Security scanners upload SARIF to the
  Security tab; coverage is enforced but not uploaded; audits are pass/fail only.
- **Publish.** `cd.yml` runs on push to `main` and calls
  `vergil-actions/.github/workflows/cd-release.yml`, which builds/publishes the
  package, then runs the `tag-and-release` composite action to create the git tag
  and the GitHub Release. SBOMs (CycloneDX) are already attached as Release
  assets for some languages.
- **Doc site.** `vrg-docs-stage` (`vergil_tooling/lib/docs.py`) copies
  `releases/v*.md` into the mkdocs site, builds the release index, and
  `vrg-docs-patch-nav` patches the nav. Deployed with `mike` (versioned docs).

### 3.1 The gaps this epic closes

- No per-release, durable record of which gates ran or their results.
- No stable URL for security scan results per release.
- Coverage, audit, and test outputs are not persisted at all.
- CI run URLs are not captured at publish time.
- The GitHub Release body does not reference any of the above.

## 4. Determinism — the core feasibility question

The central risk raised during brainstorming: *can we deterministically extract
the gates that validated the released commit?* The answer is yes, but the anchor
is subtle.

### 4.1 The released commit is not where the gates ran

`develop → main` promotion uses a **merge commit** (`strategy="merge"` in the
release orchestrator). The security/audit/test gates run on **pull requests**,
not on pushes to `main`. Consequently the final `main` merge-commit SHA has **no
gate check-runs anchored to it** — querying `commits/{main_sha}/check-runs`
returns nothing useful. Evidence cannot be reconstructed from the released SHA
after the fact.

### 4.2 The anchor: the release PR, harvested while fresh

The gates *do* run on the **release PR**, and the release orchestrator already
waits on those checks before merging. The tree that PR validated is identical to
the tree that lands on `main` (no commits interleave), so evidence harvested from
the release PR's runs is bound to exactly what ships.

Therefore evidence is **harvested at publish time**, immediately after merge,
while the release PR's workflow runs and artifacts are still within retention.
Walking the merge commit back to its PR reuses the existing
`vrg-resolve-tracking-issue` linkage pattern.

### 4.3 Storage — the one irreversible choice

Full raw reports run ~2–12 MB per release (SARIF dominates, carrying large rule
metadata even with zero findings), compressing ~5–10× to ~0.5–2 MB. This data is
stored as a **compressed GitHub Release asset**, which GitHub hosts free, without
expiry, and — critically — **without ever touching git clone size**. Committing
raw reports into the repo was rejected: it grows history permanently, is paid by
every clone and CI checkout forever, and is hard to undo (history rewrite). The
repo commits nothing but the (unchanged) git-cliff release notes; the doc-site
link is derived from the deterministic asset URL (§11).

## 5. Architecture

Two sides coupled only by a **naming convention** — no direct calls — which is
what keeps the harvester language-agnostic and the whole mechanism fleet-wide.

- **Producer side (`vergil-actions` CI gates).** Each gate reusable workflow
  gains one additive step: upload its full report(s) as a workflow-run artifact
  named `ci-evidence-<gate>`.
- **Consumer side (publish-time harvest).** A new composite action in
  `vergil-actions` (`actions/cd/release/ci-evidence`), invoked from
  `cd-release.yml`, calls a new `vergil-tooling` command `vrg-ci-evidence`, which
  harvests, validates, bundles, and attests the evidence and attaches it to the
  Release.

### 5.1 Data flow (one release)

```
push to main
  └─ cd-release.yml
       ├─ actions/cd/release/ci-evidence  (runs FIRST — see §9)
       │    └─ vrg-ci-evidence bundle
       │         1. resolve the release PR from the merge commit
       │         2. select the COMPLETED, SUCCESSFUL CI run by PR head SHA
       │            + workflow + green conclusion (never a cancelled run)
       │         3. download every  ci-evidence-*  artifact (while fresh)
       │         4. read check-run conclusions via API (evidence FILES come
       │            only from artifacts — no code-scanning API dependency)
       │         5. VALIDATE completeness: every required evidence-producing
       │            gate (§7, derived from branch protection) passed AND
       │            emitted its artifact — else hard fail (§9)
       │         6. assemble tree + manifest.json + per-file sha256
       │         7. tar.gz  →  v{version}-ci-evidence.tar.gz
       ├─ build & publish package to registry   (only if evidence complete)
       ├─ tag-and-release  → create GitHub Release
       ├─ attest bundle (build-provenance over its digest) + gh release upload
       └─ (later) docs deploy → vrg-docs-stage adds the evidence link
```

Ordering evidence **first** is what makes the publish invariant real (§9): no
package, tag, or Release is created unless complete evidence has already been
assembled.

### 5.2 Preconditions (explicit, validated — not assumed)

Harvesting one workflow run's artifacts from within a *different* (post-merge) CD
run has hard preconditions the implementation must establish and the harvester
must assert:

- **Permission.** `cd.yml` / `cd-release` must grant **`actions: read`** — listing
  and downloading another run's artifacts requires it (the codebase already relies
  on this scope for SARIF upload; see `lib/github_config.py`). Without it, harvest
  fails outright.
- **Run selection.** `ci.yml` sets `concurrency: cancel-in-progress: true`, so a
  release PR may have a **cancelled/superseded** CI run alongside the successful
  one. The harvester binds only to the run that (a) matches the PR **head SHA**,
  (b) is the CI **workflow**, (c) is **completed** with a **success** conclusion,
  and (d) carries the required checks branch protection demanded. A cancelled or
  partial run is never trusted; if no qualifying run exists, that is a *substantive*
  failure (§9), surfaced with the run identity.

## 6. Components (units)

Five well-bounded, independently testable units.

1. **Evidence-emission convention** — *`vergil-actions`, per gate.* A gate
   uploads artifact `ci-evidence-<gate>` containing its raw report files, plus a
   small `evidence.json` fragment (§7). Purely additive to existing gate
   workflows.
2. **`vrg-ci-evidence` command** — *`vergil-tooling`, new `bin/` +
   `lib/ci_evidence.py`.* The harvester/bundler, internally decomposed into: a
   PR resolver (reusing `vrg-resolve-tracking-issue` logic), a run/artifact
   enumerator + downloader (over `lib/github.py`), a check-run metadata fetcher,
   a completeness validator, and a bundle assembler (tree, manifest, `sha256`,
   `.tar.gz`).
3. **`cd-release` integration** — *`vergil-actions`, new composite action
   `actions/cd/release/ci-evidence`.* Invokes `vrg-ci-evidence bundle`, performs
   the attestation, and uploads assets. Reorders `cd-release.yml` so evidence
   runs before publish.
4. **Doc-site link emission** — *`vergil-tooling`, `lib/docs.py` /
   `vrg-docs-stage`.* Appends the evidence link to each release page when the
   release has an evidence asset.
5. **Convention doc** — *`vergil-tooling/docs`.* Specifies the artifact naming,
   the `evidence.json` schema, the evidence-producing gate set, and the
   bundle/manifest format, so any repo or future gate can conform.

The only cross-repo coupling is the `ci-evidence-*` name and the `evidence.json`
shape.

### 6.1 Scope boundary

The evidence step lives inside `cd-release`, so it applies to — and only to —
**repos that publish releases through `cd-release`**. Repos that do not cut
package releases (`vergil-project/.github`, docs-only repos) never invoke it;
there is nothing to enforce and nothing to fail for them. "Fleet-wide" throughout
this spec means "every release-publishing repo," not literally every repository.

## 7. Evidence-emission convention

- **Artifact name:** `ci-evidence-<gate>` (e.g. `ci-evidence-security`,
  `ci-evidence-test`, `ci-evidence-audit`).
- **Contents:** the gate's full report files (SARIF, coverage XML/HTML, JUnit
  XML, audit/license JSON, SBOM, …), plus an `evidence.json` fragment at the
  artifact root:

  ```json
  {
    "gate": "security",
    "tools": [{ "name": "codeql", "version": "..." }],
    "metrics": { "findings_by_severity": { "critical": 0, "high": 0 } },
    "files": ["codeql.sarif", "trivy.sarif", "semgrep.sarif"]
  }
  ```

  If the fragment is absent, the harvester still bundles the raw files and records
  the gate's conclusion from the check-runs API — but see completeness (§9).
- **Single uniform evidence source — always artifacts.** Every gate, including the
  security scanners, uploads its report(s) as a `ci-evidence-<gate>` artifact
  **unconditionally**, decoupled from any code-scanning upload. Today the no-GHAS
  path already "preserves Trivy/Semgrep SARIF as build artifacts"; this convention
  extends that to *always* — CodeQL under GHAS also drops its SARIF as an artifact
  alongside its code-scanning upload. Consequently the harvester reads **artifacts
  only** and never touches the code-scanning API — one code path, no GHAS
  branching, one fewer external dependency to fail (which matters given release
  fragility). Severity metrics come from each gate's `evidence.json` fragment,
  computed from its own SARIF at gate time, not from a server API.

### 7.1 Evidence-producing gate set — derived from the common config

The set of gates that MUST emit evidence is **not a hand-maintained list**. It is
**derived from the same source of truth that drives branch protection**:
`lib/github_config.py:desired_ci_gates_ruleset()` computes a repo's required
status checks from its `VergilConfig` (language, `[ci]` versions, GHAS
availability). The evidence layer consumes that *same* computation, so the gates
that are **enforced to merge** and the gates that are **required to have
evidence** are provably the same set, with no drift — exactly the "management of
the required gates and the collection of their auditing must come from common
configuration code" constraint.

- **Classification by check-name prefix.** Required checks are grouped by prefix:
  `security/` (plus the GHAS `Trivy`/`Semgrep OSS`/`CodeQL` checks), `test/`,
  `audit/`, and `quality/` are **evidence-producing** — each must emit its
  artifact. `version/` is **non-blocking** (a sanity check on version state, not
  substantive evidence); `docs` is likewise low-signal. Their absence does not
  fail the release.
- **Per-repo correctness for free.** Because the set is derived, a repo with a
  different real profile — no GHAS (CodeQL not required), a non-Python stack, a
  gate legitimately disabled — demands evidence for exactly the gates it actually
  gates on. It cannot spuriously hard-fail for a gate it never ran. This is what
  makes enforce-everywhere (§14) safe across a heterogeneous fleet.

The guiding principle: **any gate that can block the build is evidence worth
keeping** — quality (lint/typecheck) sits alongside security, test, and audit as
first-class evidence. Adding a future required gate automatically pulls it into
the evidence set via the shared config; the harvester never changes.

## 8. Bundle and manifest format

The bundle is designed to be consumed by a machine auditor: self-describing and
verifiable.

```
v{version}-ci-evidence.tar.gz
└─ evidence/
   ├─ manifest.json          # top-level machine index (below)
   ├─ checks.json            # raw check-runs snapshot (name, conclusion, timing, log URL)
   ├─ gates/
   │   ├─ security/          # codeql.sarif, trivy.sarif, semgrep.sarif, evidence.json
   │   ├─ test/              # coverage.xml, htmlcov/, junit.xml, evidence.json
   │   ├─ audit/             # pip-audit.json, licenses.json, evidence.json
   │   └─ sbom/              # sbom.cdx.json
   └─ README.md              # human orientation for the archive
```

**`manifest.json`:**

```json
{
  "schema_version": "1.0",
  "repo": "vergil-project/vergil-tooling",
  "release": { "version": "2.1.129", "tag": "v2.1.129",
               "released_commit": "<main merge SHA>" },
  "provenance": { "release_pr": 2281, "validated_head_sha": "<PR head SHA>",
                  "ci_run_urls": ["https://github.com/.../actions/runs/123"] },
  "generated_at": "<ISO-8601, injected by CD>",
  "gates": [
    { "name": "security", "conclusion": "success",
      "tools": [{ "name": "codeql", "version": "..." }],
      "metrics": { "findings_by_severity": { "critical": 0, "high": 0 } },
      "files": [{ "path": "gates/security/codeql.sarif", "sha256": "..." }] },
    { "name": "test", "conclusion": "success",
      "metrics": { "coverage_pct": 100, "tests": 1423 }, "files": [ ... ] }
  ],
  "missing_gates": []
}
```

Two trust properties: every file carries a **`sha256`** (an auditor can prove the
archive is intact), and **`missing_gates`** is explicit — a gate that produced no
evidence is recorded as data, never silently dropped (and, per §9, blocks the
release). A standalone copy of `manifest.json` is also attached as a small,
separate Release asset so a tool can read the summary without downloading the
full tarball.

The `gates/sbom/` entry is populated by **copying the SBOM already built during
publish** (it is also a standalone Release asset) into the bundle at harvest
time — the archive stays fully self-contained without a separate SBOM upload
path. `checks.json` is the raw check-run snapshot (distinct from the manifest's
curated per-gate view), and `README.md` is a fixed human orientation for the
archive; both are written by the bundle assembler.

> Timestamps are injected by the CD environment; the harvester itself is
> deterministic given its inputs, which keeps it unit-testable.

## 9. Publish invariant and `cd-release` restructuring

**Invariant (the enforcing end state): a published release always has complete,
attested evidence.** If the pipeline cannot prove it, it does not publish. This is
the gate's permanent target state — but it is *reached through a staged
deployment*, not switched on the day the code lands (§9.2). Sections 9–12 describe
**enforcing mode**; §9.2 defines the warning-mode deployment that precedes it.

To make this true by construction rather than after the fact, `cd-release.yml` is
restructured so **evidence harvest + completeness validation runs first**, before
the package registry push, the tag, and the GitHub Release. The validator requires
that **every gate in the derived evidence-producing set (§7.1) both passed and
emitted its evidence artifact**. A required gate that ran but produced no evidence
is a **substantive failure**: hard, loud, terminal — nothing is published.

**Two failure classes, treated differently (§12):** a *substantive*
incompleteness (a required gate genuinely emitted nothing) is a terminal block; a
*transient* fetch failure (an API blip while retrieving evidence that does exist)
is retried, and — because the whole step is idempotent and re-runnable — a failed
CD run is simply replayed rather than permanently stranded. This preserves both
the publish invariant and the established "never strand `develop`" philosophy
(commit #1859).

Net effect: there is no window in which a release exists without its proof, and
the mechanism's robustness is self-enforcing — if evidence capture is genuinely
broken, releases stop until it is fixed. This is a deliberate, requested trade-off:
correctness of the evidence trail over release throughput.

### 9.1 Publish safety — why a public bundle is safe

Bundling *everything* onto a public Release asset is safe **precisely because
every gate is a hard gate**. Evidence generation only runs after all gates are
green, and there are no report-only/warning gates — so a bundle by definition
reflects a *passing* scan with no unremediated findings to expose. There is no
dedicated secret-scanning gate emitting secret locations (only CodeQL/Trivy/
Semgrep), and on a public repo the code and dependency inventory are already
public. **This safety property is load-bearing on the all-hard-gates model:**
introducing any *permanent* soft/report-only gate in the future would let a
passing release carry non-fatal findings into a public bundle, and would require
revisiting this boundary. That model is itself a foundational principle worth
codifying (§14).

Note that warning-mode deployment (§9.2) does **not** weaken this: warning mode
governs only the *evidence step's own* fatality, not the underlying gates. The
bundle is still generated only from releases whose hard gates all passed, so its
contents are as safe in warning mode as in enforcing mode.

### 9.2 Deployment lifecycle — warning mode first, then enforcing

**The requirement that all gates are hard is unchanged. What is staged is the
*deployment* of this new hard gate.** A new hard gate is introduced in **warning
mode** and promoted to **enforcing mode** only once it is proven reliable in
production:

- **Warning mode (initial).** The evidence step runs the full path — harvest →
  bundle → attest → attach — but on **any** failure (transient error, substantive
  incompleteness, upload/attest failure, or timeout) it emits a **loud warning and
  the release proceeds**, attaching whatever evidence it did gather. It never
  aborts a release. The step is **timeout-bounded** so it can never hang the
  pipeline.
- **Bake-in.** Warning mode runs across normal release cadence (a couple releases
  per day) for ~1–2 weeks, accumulating reliability data; defects are worked out
  as they surface.
- **Enforcing mode (end state).** Once stable, a **single flag flip** promotes the
  gate to the §9 invariant: evidence-first, substantive incompleteness is
  terminal, nothing publishes without complete attested evidence.

**Warning mode is a temporary deployment state of a hard gate — not a permanent
soft gate.** A permanent soft/report-only gate remains rejected (§9.1, §14). The
end state of this gate is always enforcing; warning mode is the on-ramp, bounded
in time and closed by a human-gated flip (§14.1).

**Mechanism.** An `enforce` flag on the `cd-release` evidence step (composite
input), default `false` (warning) at introduction. The flip to `true` is a
single, global, human-gated change (§14.1) made after the bake, on the strength of
the collected reliability data. In warning mode the step's failure handling
collapses the §12 transient/substantive distinction — *all* failures are
non-fatal; that distinction only takes effect once enforcing.

## 10. Attestation — chain of custody

`cd.yml` already grants `attestations: write` and `id-token: write`. After the
bundle is assembled, the pipeline produces a **build-provenance attestation over
the bundle's digest** (`actions/attest-build-provenance`), binding the archive to
this repository's workflow and the released commit. Six months later an auditor
verifies it with `gh attestation verify` — turning "here is a tarball" into "here
is a tarball cryptographically proven to be the genuine output of this pipeline
for this commit." This is the chain-of-custody core of the epic, not an add-on.

## 11. Doc-site integration (deliberately thin)

`vrg-docs-stage` gains one behavior: for each rendered release page, if the
release has a `*-ci-evidence.tar.gz` asset (one cheap `gh release view` check at
docs-build time), append:

> **CI Evidence:** All gates passed — full audit bundle available.
> [Download →](.../releases/download/v2.1.129/v2.1.129-ci-evidence.tar.gz)

No gate count (if it did not pass, it did not publish, so "all gates passed" is
the invariant statement). Releases without a bundle get no line. The asset URL is
deterministic, so this needs no new committed files and no manifest parsing at
build time — the presence of the asset is the source of truth.

## 12. Error handling and edge cases

Consistent with the fleet's no-silent-failure rule. **The fatality of each case
below is mode-dependent (§9.2): in warning mode every failure is loud but
non-fatal; the terminal behavior described here applies once enforcing.**

- **Missing/partial evidence** → recorded in `manifest.json → missing_gates` and
  printed loudly in every mode. In **enforcing mode** this (per §9) **fails the
  release**; in **warning mode** it is reported and the release proceeds with a
  partial bundle. Never hidden.
- **Release PR unresolvable** → **hard failure, no fallback.** Every legitimate
  path to `main` goes through a PR: standard releases merge a release PR, and the
  hotfix policy requires the `hotfix/*` branch be merged into `main` (i.e. via
  PR, per the no-direct-commits rule). A commit that reached `main` with no
  resolvable PR is therefore a violation of the mandatory-PR rule, and there is
  no gate evidence to stand behind — so the evidence layer refuses to publish
  rather than papering over it with weaker-provenance guesses. This never
  triggers in normal operation; if it does, failing loudly is the correct
  outcome.
- **Transient fetch failure** (API blip, rate-limit, momentary 5xx while
  retrieving evidence that *does* exist) → **retried with backoff**; because the
  step is idempotent (below) a run that still can't complete is *replayed*, not
  permanently stranded. This is distinct from substantive incompleteness (a
  required gate emitted nothing), which is terminal (§9).
- **Idempotency / re-runs** → the evidence step is a pure function of
  (release PR → its qualifying run → its artifacts): re-harvest, re-bundle, and
  re-upload (`gh release upload --clobber`) are safe to repeat, re-attestation is
  additive, and the step relies on no partial local state. `tag-and-release` is
  already idempotent, so a failed CD run can be fixed and replayed end-to-end.

### 12.1 Forward-compatibility with `vrg-release` idempotency

This epic adds critical functionality to the release pipeline — the fleet's most
fragile surface — so it is explicitly engineered to compose with the forthcoming
**`vrg-release` idempotency/retriability epic** without rework. Concretely: all
GitHub calls go through a retry/backoff wrapper that can later adopt whatever
shared retry primitive that epic introduces; every mutation is idempotent; and no
step depends on state that would not survive a fresh CD re-invocation. That epic
is **not implemented here** — this section only guarantees we do not paint it into
a corner.

## 13. Testing strategy

- **Unit (`vergil-tooling`):** manifest builder (synthetic runs/artifacts/checks
  → correct manifest + hashes), bundle tree layout, `sha256` integrity,
  completeness validator (present/missing gates), PR resolution + failure paths,
  docs-stage link emission (asset present/absent). GitHub API mocked at the
  `lib/github.py` boundary; timestamps injected so the builder is deterministic.
- **Integration:** a fixture directory of downloaded artifacts → full `.tar.gz`
  → assert tar contents and validate `manifest.json` against its schema.
- **`vergil-actions`:** gate artifact-upload steps verified on a real PR; one
  end-to-end smoke release confirming harvest → validate → attest → attach →
  link, including a deliberately-missing-gate run that must fail closed.

## 14. Rollout, rollback, and task decomposition

Built in sequence; each phase is independently shippable and maps to one or more
tasks (filed in their member repos and linked to this epic at implementation
time).

1. **Producer uploads** (`vergil-actions`): every required gate uploads its
   `ci-evidence-<gate>` artifact unconditionally (§7). Harmless everywhere
   immediately; unlocks harvesting.
2. **Harvester** (`vergil-tooling`): build `vrg-ci-evidence` to a high robustness
   bar — comprehensive retry/backoff, idempotency, clear failure diagnostics —
   with full unit/integration coverage. No release wiring yet.
3. **Convention doc** (`vergil-tooling`): publish the evidence convention and
   bundle/manifest spec.
4. **Codify the all-hard-gates principle** (`vergil-actions` docs): write up, as a
   foundational principle, *why every check that matters is a hard, asserting gate
   and there are no report-only/warning gates* — a warning only has value if a
   human acts on it, and at modern code-generation rates reviewers optimize for
   "can I merge?" and never revisit warnings, so an unheeded warning is
   meaningless; deprecation warnings especially are early signal of a future
   outage and are treated as errors. This principle is what makes the public
   evidence bundle safe (§9.1); it deserves its own home in the docs.
5. **Wire + attest, in warning mode** (`vergil-actions`): add
   `actions/cd/release/ci-evidence`, grant `actions: read` (§5.2), restructure
   `cd-release.yml` (evidence-first), add attestation, and expose the `enforce`
   flag **defaulting to `false` (warning, §9.2)** — non-fatal, timeout-bounded.
   First real bundle on the next `vergil-tooling` release.
6. **Doc-site link** (`vergil-tooling`): `vrg-docs-stage` evidence-link emission.
7. **Promote to enforcing** (human-gated, after the bake): flip the `enforce`
   default to `true`. Gated on the reliability data collected in warning mode; the
   human decides when. This is the closing step of the deployment lifecycle (§9.2).

### 14.1 Enforcement posture — warning mode first, then a human-gated flip

The gate is deployed in **warning mode** (§9.2) — non-fatal, timeout-bounded — for
a ~1–2 week bake across normal release cadence, then promoted to **enforcing** by a
single global flag flip once the reliability data justifies it. This replaces the
earlier "enforce on day one" posture: the release pipeline is the fleet's most
fragile surface and this epic adds to it, so a bake-in window that cannot break or
delay a release is the responsible way to introduce a new hard gate. It does **not**
change the requirement that the gate's end state is hard/enforcing.

Two independent safety nets back the rollout:

- **Warning mode** means the evidence step cannot abort a release during the bake,
  regardless of harvester defects or GitHub flakiness.
- **Rollback** remains available for the enforcing phase: the change ships as a
  single isolated **`vergil-tooling` patch release**, and consumers pin the rolling
  **`2.1` major-minor tag**, so a bad enforcing release is backed out by demoting
  that tag to the prior patch with **`vrg-promote`** — trivial and near-instant.

The flip to enforcing (phase 7) is **global and human-gated**, not per-repo: the
human promotes once, on the strength of fleet-wide warning-mode data, having worked
out the defects that surface during the bake (e.g. during the MQ-Rest admin work).
The robustness mandate (phase 2) and immediate dogfooding still apply — warning
mode is the on-ramp to a robust hard gate, not a substitute for building it well.

Task linkage template:

```
vrg-epic-link --epic vergil-project/.github#140 \
              --task vergil-project/<repo>#<TASK>
```

## 15. Future work (explicitly deferred)

- **Live, indexed evidence browsing.** A rendered, navigable view layered on top
  of the durable bundles. Nice-to-have, not required for the audit-trail goal,
  and deferrable without rework because the bundle + manifest are already the
  source of truth.
- **Richer per-gate metrics** in the manifest (trend data across releases).
- **Attestation verification in downstream consumers** (e.g. a `vrg-*` verifier
  that fetches a release's bundle, checks `gh attestation verify`, and validates
  every `sha256`).
