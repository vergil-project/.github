# Container version pinning & floating-dependency management

- **Epic:** vergil-project/.github#155
- **Status:** Approved design (2026-07-16) — superpowers brainstorming, hardened
  via paad pushback the same day (6 findings resolved; Tenet 6 reframed and a pin
  lifecycle algorithm added during pushback). Pending human review.
- **Repo:** vergil-project/vergil-containers (member); epic homed in `.github`.
- **Related work:**
  - vergil-project/.github#52 "Mechanize dependency-update" (vergil-tooling) —
    sibling epic; its Docker extension (#1597) is a plausible consumer of this
    epic's pin catalog, its warn-only task (#1599) is adjacent to the report-only
    Dependabot idea.
  - vergil-containers#60 "versioned tagging for dev-* images" (closed, deferred)
    — prior art for workstream 3; its deferral rationale ("staging is
    hypothetical") is what this epic invalidates.
  - vergil-containers#413 — workstream 1, already filed under ad-hoc epic #100.
  - vergil-containers#384 / #394 / #125 — existing GHCR package-cleanup; WS3's
    reaper must coordinate with these.
- **Brainstorm source:** superpowers brainstorming session, 2026-07-16.

## 1. Summary

We pinned tool versions in the container images for stability but never built the
*update management* around those pins. A pin with no management process is a
permanent, silently-drifting freeze. This epic (a) sets a **pinning philosophy**,
(b) defines a **pin lifecycle** that makes every pin re-evaluable and prevents
permanence, (c) establishes a **two-axis management doctrine** separating
source-controlled *structure* from build-time floating *versions*, and (d) builds
the enabling infrastructure: correct nightly branch coupling, a generated pin
catalog + audit, immutable image-artifact tagging with sliding-window rollback,
and pin/version observability. Report-only Dependabot is deferred to a follow-on
epic.

## 2. Problem / motivation

- **No update management for pins.** ~30 tools are exact-pinned and freeze until a
  human manually bumps them. Nothing watches them.
- **No automation.** No Dependabot/Renovate anywhere.
- **Drift already leaking.** `CLAUDE.md` says `uv 0.11.13`; code is on `0.11.14`.
- **Trailing-edge security exposure.** A static pin slides from leading edge to
  trailing edge over time; with no alerting, a pin can carry a known CVE
  indefinitely.
- **No rollback.** Image tags are rolling/mutable; there is no immutable per-build
  alias to repoint to.

## 3. Pinning philosophy

1. **Default is unpinned.** Float on each product's leading edge.
2. **A pin is a reaction, never a default** — a reaction to being unable to stay
   on the leading edge (a release breaks us).
3. **Every pin carries a written justification** — "held because of release X,
   for reason Y."
4. **Least-specific pin that solves the problem** — pin `1.x`, not the exact
   patch.
5. **When in doubt, set it free** — unpin, see what breaks, then pin only the
   specific culprit at the least-specific working constraint.
6. **A pin is anchored to the release that induced it, and carries a
   re-evaluation trigger — never permanence.** Every pin records the specific
   upstream release whose problem caused it. The pin is valid *only while that
   release is the leading edge*; the moment the leading edge moves past the
   inducing release, the pin is automatically due for re-evaluation. The
   management trigger ("has the leading edge moved past the inducing release?") is
   always deterministic and observable, **even when the underlying problem is
   not** — this is what makes every pin releasable and kills permanent pins
   structurally. (Superseded finding: an earlier draft required the *problem* to
   be testable; that wrongly excluded legitimate non-deterministic/stability
   pins. Anchoring to the inducing release is the correct invariant.)

## 4. Pin lifecycle & re-evaluation

### 4.1 Pin states

A pin entry carries an explicit **state** (never a commented-out line — the
metadata must stay structured):

- **active** — enforced; the version constraint applies.
- **under-evaluation** — *not* enforced (floats), but retained with full metadata
  and a scheduled follow-up issue. The observe phase for a non-deterministic pin.
- **freed** — removed; the tool floats with no record needed.

### 4.2 Pin metadata (the `pins.yml` schema — see WS2)

Per pin: `tool`, `constraint` (e.g. `<=1.x`), `inducing_release` (the upstream
version whose problem caused the pin), `deterministic` (bool — is the problem
reproducible/testable?), `reason` (prose), `state`, `tracking_issue` (for
under-evaluation).

### 4.3 Re-evaluation algorithm

**Trigger:** the leading edge moves past a pin's `inducing_release` (detected by
observability, §10).

- **Deterministic pin** (problem is reproducible → testable):
  - Test the new leading edge against the known problem.
  - **Problem gone** → **delete the pin** (state → freed; float again).
  - **Problem persists** → **re-anchor**: set `inducing_release` to the new
    leading edge (keep holding; the trigger won't fire again until something
    *newer* appears). A persistent problem advances the anchor, it does not delete
    the pin.
- **Non-deterministic / stability pin** (cannot pre-test):
  - **Suppress and observe**: set state → under-evaluation (constraint no longer
    enforced; tool floats to the new leading edge) with a note, and open a
    **scheduled follow-up issue** to proactively check stability at a future date.
    WS3 rollback covers the interim if instability recurs.
  - Instability recurs → restore the pin (state → active, re-anchor past the bad
    release). Stable across the evaluation window → state → freed.

**Mechanization boundary:** the *trigger* is fully mechanizable (observability
flags it); the *deterministic re-test* often is (a CI test); the
*non-deterministic observation* is not — but its **process** is fully described
(under-evaluation state + tracking issue), so nothing silently rots.

## 5. Two-axis management doctrine

### 5.1 Axis A — container structure (source-controlled)

Dockerfiles, tooling inventory. Moves as a **step function** through GitFlow:
`develop → main` = `dev → prod`. **The structural canary lives here** (validate
`dev-` images before they release to prod). Rollback = revert / re-release.

### 5.2 Axis B — dependency versions (build-time floating)

Whatever versions resolve at each nightly build, within pin constraints. A
**continuous** target; cannot be a clean step function. Managed by: aggressive
nightly builds on both dev+prod **from their correct branches** (WS1); accept
leading-edge breakage and fix reactively with a least-specific pin; **fast
recovery via immutable image-artifact aliases** repointed within a sliding
retention window (WS3); report-only Dependabot for trailing-edge security
(follow-on); and observability of pin state and re-evaluation triggers (WS4).

The canary is an Axis-A concept (structure), not Axis-B (versions). Axis A rolls
back via source; Axis B rolls back via artifact-digest repoint — a source rebuild
would merely re-float and is useless for Axis-B recovery.

## 6. Findings (repo investigation, 2026-07-16)

- **Nightly builds prod from the wrong branch (WS1, filed #413).** Default branch
  is `develop`; the scheduled `ops.yml` passes no `ref:`, so nightly prod images
  are built from `develop`, not `main`.
- **No image rollback exists.** Tags are rolling only; the candidate digest is
  captured for attestation but never published as an immutable alias.
- **Immutable *artifact* aliases solve Axis-B rollback** (repoint, not rebuild) —
  reviving #60's direction.
- **A successful container build is not a good container.** Goodness is only
  proven when a downstream consumer exercises it, which can lag the build by days
  or weeks (stable images go static). This is why "last-good" cannot be marked at
  build time — and why rollback needs a *window* of history, not just N-1 (§8).
- **Inconsistent pins.** semgrep, Ruby bundler, license_finder are unpinned while
  peers are pinned — resolve during the audit.
- **Language tooling is already language-scoped** (Go in `go/`, cargo in `rust/`).
  `go-test-coverage` is already conditionally pinned by Go version — a live Tenet
  4/6 example (inducing condition: newer requires Go ≥1.26; release condition:
  when Go 1.25 leaves the matrix).
- **Security scanners float.** Pinning trivy/scorecard/semgrep/govulncheck to
  guard against an *untestable* silent regression fails Tenet 6 (no release
  condition) and freezes aging scan logic — worse for security. They float; the
  vuln DB updates at scan time regardless of binary version.

## 7. Workstreams

### WS1 — Fix nightly branch coupling *(filed: vergil-containers#413)*
Prod from `main`, dev from `develop`; explicit `ref:` per prefix. Under ad-hoc
epic #100. Referenced, not re-scoped.

### WS2 — Pin audit + catch-up (generated catalog)
- **Generated catalog, not hand-maintained** (the founding drift problem must not
  recur). Version *facts* are generated from the Dockerfile templates/fragments
  (single source of truth). *Justifications* live in a keyed `pins.yml` (§4.2). A
  generator joins them into the human-readable catalog. A **CI gate fails if any
  pinned tool lacks a `pins.yml` justification** — a new pin cannot merge
  undocumented.
- **Audit each pin through Tenet 6:** it survives only if it names an
  `inducing_release`, `deterministic?`, and a reason; otherwise it is freed.
- **Free by failure mode (sequencing, per Finding 1):** do catalog+justify
  immediately (zero risk). Free **loud-failure** tools now (linters/formatters — a
  bad float breaks the build visibly; source-revert suffices). **Gate freeing of
  silent-failure / fleet-gating tools** (security scanners, coverage gates) on WS3
  landing, so nothing whose failure is silent or fleet-wide floats free before we
  can repoint out of it. Plan encodes `WS2-free-risky` blocked-by `WS3`.

### WS3 — Immutable image-artifact tagging + sliding-window rollback
- Publish, alongside the rolling tag, an **immutable datestamp alias** per build
  (e.g. `{prefix}-{lang}:{version}-YYYYMMDD` → digest). Datestamp (not semver)
  because it enables age-based cleanup and chronological bisection.
- **Repoint rollback**: recover a bad float by repointing the rolling tag at a
  prior immutable digest via `docker buildx imagetools create`. No rebuild.
- **Sliding-window retention** (not N-1 — a successful build isn't a good build,
  §6): keep all aliases within a window; reap beyond via an age-based reaper.
  **prod W = 30 days** (consumer rollback + bisection range); **dev W = 7 days**
  (churning canary). The window is both the "oh shit" undo *and* the bisection
  range for downstream-detected problems.
- **Coordinate with existing cleanup first.** WS3's first task is to understand
  the existing GHCR package-cleanup (#384/#394/#125) and make it alias-aware: its
  keep-set becomes "aliases within W," everything else reaps. Optional guard: do
  not reap the active rollback target during an open incident.

### WS4 — Pin/version observability (internal state)
- **Narrowed to what is local and un-driftable** (upstream-distance is delegated
  to the report-only Dependabot follow-on, §11): per image, the installed version
  set (from the image/SBOM) joined with `pins.yml`.
- **Headline signal:** *pins whose `inducing_release` is no longer the leading
  edge* — i.e. **due for re-evaluation** (§4.3). Deterministic; local plus one
  upstream latest-version lookup per pinned tool.
- Surfaced for a daily morning-review ritual. Reads the same `pins.yml` as WS2.

## 8. Immutable tagging + rollback design (WS3 detail)

- **Two tag classes:** rolling `{prefix}-{lang}:{version}` (unchanged; consumers
  default here) and immutable `{prefix}-{lang}:{version}-YYYYMMDD` (new, never
  overwritten, reaped by age).
- **Why a window, not N-1:** downstream detection lag means you often cannot tell
  *which* of several nightly builds introduced a problem; recovery may require
  walking back several builds. N-1 also silently ages out the last-good the moment
  a second bad build lands. A sliding window supports both one-step undo and
  multi-step bisection.
- **Retention = rollback SLA:** you can only repoint to an alias still in-window.
  prod W=30, dev W=7.

## 9. Capacity & cost analysis

**Data (measured GHCR manifests, amd64+arm64 compressed):** base 891 MB,
python 469 MB, go 1,828 MB, rust 1,122 MB, java 775 MB, ruby 689 MB. Full matrix
(13 image-versions × 2 prefixes = 26 images) ≈ **23 GB per nightly build**
(naive). At prod W=30 / dev W=7 ≈ **~380 GB** retained at steady state (naive
upper bound).

**Data (sourced):** public GHCR packages are **currently free, unlimited storage
+ bandwidth**, ≥1-month notice before any change; our packages are confirmed
public. If the free policy ever ends, historical GitHub Packages storage is
**$0.25/GB/month**; data transfer is free when pulled inside Actions.
(<https://docs.github.com/en/billing/concepts/product-billing/github-packages>)

**Judgment:**
- **No capacity wall today** (public = free). The **age-based reaper is what
  prevents the wall** — footprint plateaus at ~W×daily and stays flat rather than
  accelerating.
- **Forward risk is policy, not capacity.** If billing ever lands: worst case
  ~380 GB × $0.25 ≈ **~$95/mo** at these windows (no dedup); realistically lower
  with base-layer dedup. Within tolerance, and W is a one-line knob to reduce.
  `no-cache` rebuilds limit dedup on our own layers, so I do not claim a precise
  dedup factor.
- Consumer pulls are via Actions → transfer cost negligible.

## 10. Observability design (WS4 detail)

- **Inputs:** installed-version set per image + `pins.yml`.
- **Output:** "what we run, what's pinned, why, and which pins are due for
  re-evaluation." The due-for-re-evaluation list (leading edge past
  `inducing_release`) is the actionable signal.
- **Non-goal:** an updater, or a full cross-registry drift checker (that is the
  Dependabot follow-on / #52's #1597).

## 11. Closing bookend — report-only Dependabot (follow-on epic)

Brainstormed at epic close (task #158) to spawn a **separate follow-on epic**: a
report-only Dependabot (no PRs) nightly security audit for trailing-edge exposure
on static pins, and the upstream-distance drift view WS4 delegates. It is a
**fleet-wide sweep** touching every repo's config — that scope is why it is its
own epic. Conceptual kin to #52's warn-only support (#1599); different mechanisms.

## 12. Relationship to #52 (overlap, not duplication)

`#52` builds a deterministic `vrg-update-deps` tool for *language* dependencies.
This epic is container pinning **infrastructure + doctrine**. Touch-points: #52's
Docker extension (#1597) could *apply* the bumps this epic's audit identifies;
its warn-only support (#1599) is adjacent to the Dependabot follow-on. Sibling
epics, cross-referenced.

## 13. Out of scope

- Implementing Dependabot (own follow-on epic).
- The mechanized bump tool (#52 / #1597).
- Imposing a release cadence beyond what WS3 tagging requires; the repo ships on
  merge today and that remains the floor.
- Cross-org changes.

## 14. Acceptance / success criteria

- [ ] **WS1 (#413)** merged: nightly builds dev from develop, prod from main.
- [ ] **WS2:** generated pin catalog exists; `pins.yml` carries the §4.2 schema
      per surviving pin; a CI gate blocks any undocumented pin; unknown-reason and
      Tenet-6-failing pins are freed; loud-failure tools freed now, silent/
      fleet-gating tool freeing gated on WS3; images still build + pass validation.
- [ ] **WS3:** every build publishes an immutable datestamp alias alongside the
      rolling tag; the cleanup is alias-aware with prod W=30 / dev W=7; a
      documented (ideally `vrg-*`-wrapped) repoint rollback is demonstrated
      end-to-end at least once.
- [ ] **WS4:** an exposure view shows installed versions + pin state + the
      due-for-re-evaluation list, usable as a daily scan.
- [ ] **Closing bookend (#158):** report-only Dependabot brainstormed; follow-on
      epic created.
- [ ] **Docs-review gate (#414):** docs (site docs + CLAUDE.md pin inventory)
      reflect all of the above, including the §4.3 pin re-evaluation lifecycle as
      an operator-facing page (linked from the WS4 report); per-repo doc tasks
      spawned as needed.

## 15. Open questions (for plan)

- Whether WS3's rollback repoint is wrapped in a new `vrg-*` command or a
  documented `imagetools` procedure.
- Whether the optional "don't reap the active rollback target during an incident"
  guard is worth the machinery, or fail-forward suffices.
- WS4 surface form (generated markdown report vs richer view) — MVP is a report.
- Exact `pins.yml` location and the generator's integration into `generate.sh` /
  validation.

## Appendix A — current pin inventory (WS2 seed)

Common (all images unless noted): markdownlint-cli 0.48.0, ShellCheck 0.11.0,
shfmt 3.13.1, actionlint 1.7.12 (see #260), git-cliff 2.13.1, hadolint 2.14.0,
scorecard 5.5.0, trivy 0.70.0, yamllint 1.38.0, ansible-lint 26.4.0, uv 0.11.14,
OpenTofu 1.12.3, nfpm 2.47.0. Node major 22 + gh via apt (float). pip floor
`>=26.1.2` (CVE-2026-3219; CVE-2026-8643 / PYSEC-2026-196). Base only:
mkdocs-material 9.7.6, mike 2.2.0, pyyaml 6.0.3 (why pinned?), semgrep (UNPINNED).

Language-scoped — Go (`go/`): golangci-lint 2.12.2, govulncheck 1.3.0,
go-licenses 2.0.1, gocyclo 0.6.0, goimports 0.45.0, go-test-coverage 2.18.8
(2.18.3 on Go 1.25). Rust (`rust/`): cargo-deny 0.19.6, cargo-llvm-cov 0.8.6.
Ruby (`ruby/`): bundler, license_finder (UNPINNED). Java: none.

Version matrix (intentional major.minor): Ruby 3.2–3.4, Python 3.12–3.14,
Java 17/21, Go 1.25/1.26, Rust 1.92/1.93.
