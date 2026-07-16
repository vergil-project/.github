# Container version pinning & floating-dependency management

- **Epic:** vergil-project/.github#155
- **Status:** Approved design (2026-07-16) — superpowers brainstorming. Pending
  paad pushback + human review.
- **Repo:** vergil-project/vergil-containers (member); epic homed in `.github`.
- **Related work:**
  - vergil-project/.github#52 "Mechanize dependency-update" (vergil-tooling) —
    sibling epic; its Docker extension (#1597) is a plausible consumer of this
    epic's pin catalog, its warn-only task (#1599) is adjacent to the report-only
    Dependabot idea.
  - vergil-containers#60 "versioned tagging for dev-* images" (closed, deferred)
    — prior art for workstream 3.
  - vergil-containers#413 — workstream 1, already filed under ad-hoc epic #100.
- **Brainstorm source:** superpowers brainstorming session, 2026-07-16.

## 1. Summary

We pinned tool versions in the container images for stability but never built the
*update management* around those pins. A pin with no management process is a
permanent, silently-drifting freeze: the moment the next upstream release ships,
the pin is out of date, and the only question is how far behind it has fallen —
with no alerting and no catalog of what we hold back or why.

This epic (a) sets a **pinning philosophy** (default unpinned; pin only as a
justified reaction; least-specific pin; free-by-default when the reason is
unknown), (b) establishes a **two-axis management doctrine** separating
source-controlled container *structure* from build-time floating *dependency
versions*, and (c) builds the infrastructure that makes both enforceable:
correct nightly branch coupling, a pin audit + catch-up, immutable image-artifact
tagging with repoint rollback, and pin/version drift observability. The
report-only Dependabot security audit is explicitly deferred to a follow-on epic.

## 2. Problem / motivation

- **No update management for pins.** ~30 tools are pinned to exact versions and
  freeze until a human manually bumps them. Nothing watches them.
- **No automation.** There is no Dependabot/Renovate anywhere in the repo.
- **Drift is already leaking.** `CLAUDE.md` documents `uv 0.11.13`; the code is
  on `0.11.14`. Documentation of pins goes stale the moment code moves.
- **Trailing-edge security exposure.** A static pin slides from the leading edge
  toward the trailing edge over time; without alerting, a pinned tool can carry a
  known CVE indefinitely with no signal.
- **No rollback.** Image tags are rolling and mutable; there is no immutable
  per-build alias to repoint to, so recovering from a bad floated dependency
  means a source-pinned rebuild — slow and manual.

## 3. Pinning philosophy

1. **Default is unpinned.** Float on the leading edge of each product's release
   cycle.
2. **A pin is a reaction, never a default** — specifically a reaction to being
   *unable* to stay on the leading edge (a new release breaks us). That is the
   entire justification for a pin's existence.
3. **Every pin carries a written justification** — "held at X because Y." No
   unexplained pins survive the audit.
4. **Least-specific pin that solves the problem.** If v1 → v2 breaks, pin to
   `1.x`, not the exact patch — keep floating within the major, advance the pin
   manually only when ready to move to v2.
5. **When in doubt, set it free.** Unknown-reason pins get unpinned; if something
   breaks, root-cause the specific culprit, pin *that* one to the least-specific
   working constraint, and let the rest float.

## 4. Two-axis management doctrine

Two independent axes must be managed separately. Conflating them is the root of
the confusion this epic resolves.

### 4.1 Axis A — container structure (source-controlled)

Dockerfiles, tooling inventory, structural changes. Moves as a **step function**
through GitFlow: `develop → main` = `dev → prod`. **The structural canary lives
here** — structural changes are validated by pointing an affected repo at `dev-`
images before they release to prod. Rollback = revert / re-release, aided by
immutable image tags.

### 4.2 Axis B — dependency versions (build-time floating)

Whatever versions resolve at each nightly build, within pin constraints. A
**continuous** moving target; it cannot be turned into a clean step function.
Managed by:

1. **Aggressive nightly builds on both dev and prod, each from its correct
   branch** (dev ← develop, prod ← main). Leading-edge by default.
2. **Accept leading-edge breakage.** Detect via build failure + downstream
   testing; fix reactively — root-cause the culprit, apply the least-specific pin
   that works.
3. **Fast recovery via immutable image-artifact aliases.** Each nightly build is
   an immutable artifact (digest) containing that night's exact resolved deps.
   Publishing a per-build alias lets us **repoint** the rolling tag at an older
   digest — instant rollback of the floating layer, no rebuild. This is the
   container analog of `vrg-promote` repointing a `vX.Y` actions tag.
4. **Trailing-edge security exposure** covered by a **report-only Dependabot**
   nightly audit (no PRs) — deferred to the follow-on epic (§9).
5. **Observability of pin/version exposure** — the high-level view of what we run
   and how far each pin has drifted from the leading edge.

**Key clarification:** the canary is an Axis-A concept (structure), not Axis-B
(versions). We float versions aggressively on both dev and prod; we stage
*structural* changes through dev first. The two axes have different rollback
stories: Axis A rolls back via source (revert/re-release); Axis B rolls back via
artifact-digest repoint (§7) — a source rebuild would merely re-float and is
useless for Axis-B recovery.

## 5. Findings (repo investigation, 2026-07-16)

- **Nightly builds prod from the wrong branch (workstream 1, filed as #413).**
  Default branch is `develop`; scheduled `ops.yml` runs from `develop` and
  neither nightly job passes a `ref:`, so **nightly prod images are built from
  `develop`, not `main`.** dev-from-develop is accidentally correct.
- **No image rollback exists.** Tags are rolling only
  (`{prefix}-{lang}:{version}`), overwritten each build. The candidate digest is
  captured for attestation but never published as an immutable alias.
- **Rollback insight.** Immutable *artifact* aliases DO solve Axis-B rollback
  (repoint, not rebuild) — reviving the design direction of closed #60.
- **Inconsistent pins.** semgrep, Ruby bundler, and license_finder are unpinned
  while everything around them is pinned — resolve intent during the audit.
- **Language tooling is already language-scoped.** Go tools live only in `go/`,
  cargo tools only in `rust/` — freeing them cannot ripple past their own
  language image. `go-test-coverage` is already conditionally pinned by Go
  version (`v2.18.3` for Go 1.25, else `v2.18.8`) — a live example of tenet #4.

## 6. Workstreams

### WS1 — Fix nightly branch coupling *(filed: vergil-containers#413)*
Prod builds from `main`, dev from `develop`; explicit `ref:` per prefix. Factored
out as a standalone quick fix under ad-hoc epic #100. Referenced, not re-scoped.

### WS2 — Pin audit + catch-up
Enumerate every pin (seed catalog in Appendix A), classify each as: *justified*
(record the reason), *unknown* (free it, per tenet 5), or *language-coupled*
(decide float vs pin). Free the safe set; for anything that breaks, root-cause and
apply the least-specific working pin with a written justification. Deliverable: a
committed **pin catalog** with a justification per surviving pin, and the freed
pins merged.

### WS3 — Immutable image-artifact tagging + repoint rollback
Revive the design direction of #60. Publish, alongside the existing rolling tag,
an **immutable per-build alias** (date- or digest-stamped) for every nightly and
release build, and provide a **repoint mechanism** (a `vrg-*` command or a
documented `imagetools create` procedure) to roll a rolling tag back to a prior
immutable artifact. This delivers fast rollback on both axes (§7).

### WS4 — Pin/version observability
A high-level **exposure view**: for each image, the currently-installed versions,
how far each pinned tool has drifted from its upstream leading edge, and which
pins exist and why (sourced from WS2's catalog). Surface it where the solo
maintainer can scan it as a **daily morning ritual**. Form TBD in the plan
(generated report vs SBOM diff vs dashboard).

## 7. Immutable tagging + rollback design (WS3 detail)

- **Two tag classes per image:**
  - *Rolling* (today's behavior, unchanged): `{prefix}-{lang}:{version}` — always
    the latest build. Consumers default here.
  - *Immutable* (new): a per-build alias pinned to the build's digest, e.g.
    `{prefix}-{lang}:{version}-<datestamp>` → `sha256:…`. Never overwritten.
- **Rollback = repoint, not rebuild.** To recover from a bad float, repoint the
  rolling tag at the last-good immutable digest via `docker buildx imagetools
  create`. Consumers pulling the rolling tag immediately get the older image with
  its older frozen dependency closure. No source change, no rebuild.
- **Why source-tag rollback is insufficient for Axis B:** a source-version tag
  rebuilds and re-floats dependencies, reproducing the break. Only an artifact
  alias preserves the prior night's resolved versions.
- **Open design decisions (resolve in plan):** immutable suffix scheme
  (datestamp vs semver vs rolling-minor); retention (how many nights of immutable
  aliases to keep, coordinated with GHCR package-cleanup #384/#394); whether
  rollback is wrapped in a `vrg-*` command; backfill vs prospective-only.

## 8. Observability design (WS4 detail)

- **Inputs:** the per-image installed-version set (extractable from the image or
  its SBOM), the WS2 pin catalog + justifications, and upstream latest-version
  lookups for drift.
- **Output:** an exposure view answering "what are we running, how stale is each
  pin, and why is each pin held." Anchors the morning review.
- **Non-goal:** this is *reporting*, not an updater. Applying bumps is WS2 (manual
  catch-up) and, longer term, #52's Docker extension (#1597).

## 9. Closing bookend — report-only Dependabot (follow-on epic)

Brainstormed at epic close (task #158) to spawn a **separate follow-on epic**: a
**report-only Dependabot** (no PRs) nightly security audit for trailing-edge
exposure on static pins. Dependabot was previously disabled because it opened
out-of-band PRs; the follow-on constrains it to reporting/alerting only. It is a
**fleet-wide sweep** touching every repo's configuration, not just
vergil-containers — that scope is why it is its own epic, not a task here. Note
the conceptual kinship with #52's warn-only support (#1599); they are different
mechanisms (GitHub-native async scanning vs an in-house updater).

## 10. Relationship to #52 (overlap, not duplication)

`#52 "Mechanize dependency-update"` builds a deterministic `vrg-update-deps` tool
in vergil-tooling, primarily for *language* dependencies. This epic is container
pinning **infrastructure + doctrine**. Touch-points, not overlaps:

- #52's **Docker extension (#1597)** could *apply* the bumps this epic's audit
  identifies — the mechanized execution path for our catalog.
- #52's **warn-only support (#1599)** is conceptually adjacent to the report-only
  Dependabot follow-on.

Sibling epics, cross-referenced; neither subsumes the other.

## 11. Out of scope

- Implementing Dependabot (its own follow-on epic).
- Building the mechanized bump tool (that is #52 / #1597).
- Imposing a release cadence or pre-release ceremony on vergil-containers beyond
  what WS3's tagging requires; the repo ships on merge today and that remains the
  floor.
- Cross-org changes.

## 12. Acceptance / success criteria

- [ ] WS1 (#413) merged: nightly builds dev from develop, prod from main.
- [ ] WS2: a committed pin catalog exists; every surviving pin has a written
      justification; unknown-reason pins are freed; the images still build and
      pass validation after freeing.
- [ ] WS3: every build publishes an immutable artifact alias alongside the
      rolling tag; a documented (ideally `vrg-*`-wrapped) repoint rollback exists
      and is demonstrated end-to-end at least once.
- [ ] WS4: an exposure view exists showing installed versions + pin drift +
      justifications, usable as a daily scan.
- [ ] Closing bookend (#158): report-only Dependabot brainstormed and its
      follow-on epic created.
- [ ] Documentation-review gate (#414): docs (incl. site docs + CLAUDE.md pin
      inventory) reflect all of the above; per-repo doc tasks spawned as needed.

## 13. Open questions (for pushback / plan)

- Immutable-tag suffix scheme and retention window (§7).
- Observability surface form (§8).
- WS ordering: is WS3 (rollback) a prerequisite for aggressively freeing pins in
  WS2, or can WS2 proceed on the strength of "revert the PR if a freed build
  breaks"? (Working assumption: WS2 can proceed independently since source-level
  revert covers build-time breaks; WS3 covers the *floating* breaks that occur
  between source changes.)
- Whether WS4 should consume image SBOMs already produced by the publish
  pipeline, or compute the version set independently.

## Appendix A — current pin inventory (WS2 seed)

Common (all images unless noted):
- markdownlint-cli 0.48.0, ShellCheck 0.11.0, shfmt 3.13.1, actionlint 1.7.12,
  git-cliff 2.13.1, hadolint 2.14.0, scorecard 5.5.0, trivy 0.70.0,
  yamllint 1.38.0, ansible-lint 26.4.0, uv 0.11.14, OpenTofu 1.12.3, nfpm 2.47.0
- Node major 22 (patch floats via NodeSource); gh via apt (floats)
- pip floor `>=26.1.2` (CVE-2026-3219; CVE-2026-8643 / PYSEC-2026-196)
- base only: mkdocs-material 9.7.6, mike 2.2.0, pyyaml 6.0.3 (why pinned?),
  semgrep (UNPINNED)

Language-scoped:
- Go (`go/`): golangci-lint 2.12.2, govulncheck 1.3.0, go-licenses 2.0.1,
  gocyclo 0.6.0, goimports 0.45.0, go-test-coverage 2.18.8 (2.18.3 on Go 1.25)
- Rust (`rust/`): cargo-deny 0.19.6, cargo-llvm-cov 0.8.6 (clippy/rustfmt/
  llvm-tools float with toolchain)
- Ruby (`ruby/`): bundler, license_finder (UNPINNED)
- Java: none

Version matrix (major.minor, intentional): Ruby 3.2–3.4, Python 3.12–3.14,
Java 17/21, Go 1.25/1.26, Rust 1.92/1.93.
