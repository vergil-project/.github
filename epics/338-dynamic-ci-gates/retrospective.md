# Dynamic, version-agnostic CI gates — Retrospective

## §0 At a glance

We set out to make CI versions **dynamic and version-agnostic** so a
`[ci].versions` change "just works" fleet-wide — no `--admin`, no hand-edited
`ci.yml` — killing the deadlock where a matrix-reducing change stranded a PR on
version-suffixed required checks that never ran. **What shipped:** the full
three-track design — stable `<kind>/evidence` aggregate gates as the required
checks, reusable workflows that read the matrix from `vergil.toml`, thin-caller
`ci.yml` fleet-wide, `[ci].primary-version`, `ops.yml` nightly governance, and
audit hardening — implemented, released, and rolled out across the active fleet.

### Work delivered

- **28 sub-issues** (27 closed + this retro). **3 tracks + bookends.**
- **Repos touched:** `vergil-tooling` (engine), `vergil-actions` (reusable
  workflows), and the CI-sync rollout across `docs`, `vergil-claude-plugin`,
  `vergil-containers`, `vergil-vm` (vergil-project) and `docs`,
  `mq-gateway-replay-lab`, `mq-protocol-gateway`, `mq-resiliency-lab-for-linux`,
  `mq-resiliency-observability` (logical-minds-foundry).
- **Releases:** vergil-tooling **v2.1.214–v2.1.219**, vergil-actions
  **v2.1.32–v2.1.36** (several forced by incidents, not planned tasks).
- **Track 1 (unblock):** #2982 evidence-gate ruleset · #2985 unproducible-context
  audit · #2986 classic-protection read + scoped-clean · #2988 *(deploy)* ruleset
  rolled fleet-wide.
- **Track 2 (dynamic matrix):** #2983 `[ci].primary-version` · #872 setup outputs
  · #874 workflows read `[ci].versions` · #873 evidence gate fails red · #875
  *(deploy)* actions release · #2987 thin-caller renderer · #2990 `vrg-ci-sync` +
  rollout · #876 deprecated inputs dropped. Follow-ons: #886 / #893 (family-routed
  `primary-container-tag`), #3009 (vergil-tooling thin-caller canary).
- **Track 3 (governance):** #2984 ops.yml audit signal · #2989 *(deploy)* ops.yml
  backfill.
- **Bookends:** #339 spec + plan · #2981 / #33 / #898 documentation · #2991
  validation *(deferred — see §3)* · #340 retrospective.

## §1 How the plan evolved

The plan's two-track structure held; the **delta was almost entirely in unplanned
work the rollout exposed**, not in the design.

- **The release toolchain broke mid-rollout.** Cutting the vergil-actions 2.1.32
  release, `vrg-release` failed `confirm-main` — GitHub never scheduled the CD run
  for the merge commit (a transient scheduling miss, not our code). Recovering it
  exposed a real bug: `vrg-release --resume` couldn't recover a failure at/after
  `merge-release` because `prepare` re-ran against an already-merged release PR.
  Fixed out-of-band (vergil-tooling #2998 — `prepare` adopts a merged PR) so the
  release could complete.
- **C++ was a cascade.** `mq-protocol-gateway` (first real cpp consumer) surfaced
  four separate tooling gaps once its `ci.yml` went thin: single-container jobs
  used the raw `primary-version` as a container tag (#893 — family-routed tag);
  `conan lock create` regenerated a lockfile that dirtied the tree and defeated
  reproducibility (#3021 — consume a committed lock); the gcovr coverage gate
  aborted on third-party dependency headers (#3026); and — discovered but *not*
  fixed here — the container warmup requires a committed `conan.lock` it cannot
  bootstrap. On learning cpp bootstrapping is owned by a separate agent, we
  stepped out and handed the remaining cpp work off.
- **The fleet is multi-org.** `vrg-ci-sync` hardcoded the epic
  (`vergil-project/.github#338`) for every swept repo's task issue; it cannot link
  a `logical-minds-foundry` issue to a vergil-project epic, so the sweep failed on
  all 5 LMF repos (bug #3015). We completed those by hand under each repo's own
  ad-hoc epic.
- **The acceptance test had no target.** #2991 (a matrix-*reduction* PR merging
  with no `--admin`, on Python **and** non-Python) was deferred: the sweep proved
  matrix *changes* merge cleanly fleet-wide, but no in-lane non-Python repo has a
  reducible matrix (shell repos are single-version; cpp is out of lane).

## §2 Lessons learned

- **Local-green ≠ PR-green.** Tier-1 `vrg-validate` does not run the Tier-2
  security scanners (semgrep/codeql). Sub-agents reported "clean" repos that then
  failed at merge on pre-existing scanner debt — the mq-rest-admin sweep died
  here. Gate a fleet sweep on the *PR-CI* surface, not just local validation,
  before declaring branches mergeable.
- **The first consumer of a framework is the real spec.** Every C++ gap was latent
  until `mq-protocol-gateway` exercised the thin-caller path. A framework is not
  "done" until a real repo of that language has driven it end-to-end.
- **Fleet tooling must be org-agnostic from day one.** The `vrg-ci-sync` cross-org
  failure (#3015) is a class of bug that only appears the moment a tool crosses an
  org boundary — which a happy-path single-org test never does.
- **Fail-closed guardrails earn their keep.** `confirm-main` correctly refused to
  bless a release GitHub never actually published. The pain was real, but the
  alternative — a silently half-published release — is worse.
- **Canary before fan-out.** Proving the dynamic matrix on vergil-tooling (#3009)
  before sweeping the fleet caught the shape early; the #893 cpp tag bug was then
  caught on one repo rather than ten.

## §3 Compromises & tradeoffs

- **#2991 acceptance deferred.** The epic's own definition of done — an explicit
  matrix-reduction PR merging with no `--admin` on Python + non-Python — was
  closed unrun. Justification: the fleet-wide sweep demonstrated the mechanism; a
  dedicated non-Python target does not exist in-lane. **This is real debt** — the
  headline capability is not proven by its own designed test.
- **mq-rest-admin sweep abandoned in place.** 7 ci-sync branches (all report-ready
  locally, 2 with security dep-bumps) left unmerged, blocked on each repo's
  pre-existing PR-CI debt (semgrep/codeql/audit). Left as their own reminders
  pending an org-level decision. Consequence: **#876 removed the deprecated inputs
  while those 7 (+ diogenes) still pass them** — their dormant CI will error until
  they are re-swept on reactivation. Accepted because they are on-hold and already
  red.
- **C++ handed off unfinished.** #3021 / #3026 / #893 / #886 shipped, but the
  warmup-bootstrap gap (`conan.lock` not in `_WARMUP_REQUIRES`) and the
  committed-`conan.lock` question were left to the cpp-owning agent rather than
  fixed here, to avoid stepping on planned work.

## §4 New problems & opportunities

- **vrg-ci-sync cross-org** (vergil-tooling #3015, filed) — needs per-repo-org
  epic resolution or a `--epic` override.
- **vrg-release `--resume` bug** (vergil-tooling #2998, fixed & shipped) —
  `prepare` now adopts an already-merged release PR.
- **cpp warmup bootstrap gap** (logged, handed to the cpp owner) — a lock-less cpp
  repo's container build aborts instead of skipping the warmup.
- **vrg-finalize-pr / confirm-main check-settle race** (noted, not yet filed) —
  the merge precheck reads the check rollup once; an aggregate gate's conclusion
  can lag, causing a spurious "checks failed" that a re-run clears. Worth
  hardening to poll-until-settled.
- **Transient GitHub CD-scheduling miss** (external) — a push to main did not
  spawn its CD run; recovered by manual dispatch. A monitoring opportunity if it
  recurs.

## §5 What's next

- **#2991 non-Python matrix-reduction validation** — revisit once a multi-version
  non-Python repo is in lane (likely via the cpp repo once its owning agent has it
  green).
- **mq-rest-admin-project org decision** — the operator is deciding the org's
  fate; the 7 parked ci-sync branches + tracked issues resume from there.
- **C++ framework hardening** — owned separately; the four gaps this epic surfaced
  are the starting backlog.
- **File the finalize-race hardening** as a tracked vergil-tooling bug.

## Appendix A — Operational notes

Rollout order that worked: Track-1 tooling (evidence ruleset) released →
`vrg-github-repo-config apply` fleet-wide (#2988) to clear the deadlock → Track-2
dynamic matrix released in vergil-actions → vergil-tooling thin-caller canary
(#3009) → fleet ci-sync sweep (active fleet merged; LMF by hand; mq-rest-admin
parked) → #876 deprecated-input removal. Incident recoveries: the 2.1.32 release
was completed via manual CD dispatch + `vrg-release --resume` after the #2998 fix;
the fleet ruleset apply was operator-run (the agent identity 403s on Actions
permissions).
