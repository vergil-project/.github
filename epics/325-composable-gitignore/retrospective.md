# Retrospective — Composable `.gitignore` (base + language fragments) + fleet sync

Epic: vergil-project/.github#325 · partners `spec.md` and `plan.md`.

## §0 At a glance

We set out to kill two structural problems with the monolithic baseline
`.gitignore`: every repo was forced to carry every language's ignore rules (a
Python repo and `.github` both hauled the C++/Conan lines), and propagating any
baseline change meant a hand-made PR to every repo — a pain proven by the 12-PR
manual sweep of epic #322. What shipped: a **composable model** — a repo's
`.gitignore` is a marker-fenced managed block containing `base` + its one
`primary_language` fragment (base-only when there is no fragment), with repo-local
entries preserved outside the fence — enforced by a **fenced-only** standards
audit, applied by a one-repo tool (`vrg-gitignore-sync`) and propagated by a
generic-seam fleet driver (`vrg-fleet-sync`). The monolith is deleted; all 13
managed repos across both orgs are converted.

**Core deliverables (vergil-tooling unless noted):**

| Merged | PR | What it did |
|---|---|---|
| 08-27 | .github#329 | spec + plan (this epic's design docs) |
| 08-27 | #2929 (T1) | `lib/gitignore.py` compose + fragment data files + lossless-split invariant |
| 08-27 | #2930 (T2) | managed-block `render_block` / `parse` / `check` |
| 08-27 | #2931 (T3) | `sync()` — bootstrap (filter loose lines) vs update |
| 08-27 | #2933 (T4) | `vrg-gitignore-sync` applicator CLI (`--check`/`--write`) |
| 08-27 | #2932 (T5) | `_check_gitignore` accepts fenced form (transitional flag) |
| 08-27 | #2934 (T6) | repo-init composes via `render_block`; flagship `.gitignore` fenced |
| 08-27 | #2935 (T7) | generic fleet-sweep driver + `vrg-fleet-sync` |
| 08-27 | #2940 | strip stale baseline comment scaffolding on bootstrap (found in the sweep preview) |
| 08-28 | #2945 (T10) | tighten audit to fenced-only; **delete the monolith**; freeze 62 legacy patterns into the invariant |
| 08-28 | #2946 | fleet driver stages before commit + real-git regression test (driver bug) |
| 08-28 | #2950 | docs-review: site docs on the composable/fenced-only model |

**Migration sweep** (task #2927): one per-repo PR each across **12 repos** —
vergil-project `docs`/`vergil-actions`/`vergil-containers`/`vergil-claude-plugin`/
`vergil-vm`/`.github`, logical-minds-foundry `docs`/`mq-resiliency-lab-for-linux`/
`mq-resiliency-observability`/`mq-gateway-replay-lab`/`mq-protocol-gateway`/`.github`
(vergil-tooling was fenced earlier in T6). Plus **follow-up fixes the sweep
surfaced**: containers `__pycache__/` repo-local (#585); LMF docs missing CI docs
job (#10); LMF observability missing CI docs job + pip PYSEC-2026-3721 bump (#34).

- **Repos touched:** 13 (vergil-tooling + 12 fleet repos).
- **Operational tasks:** 1 deployment gate (#2926) + 1 sweep (#2927).
- **Releases cut:** 3 — **v2.1.208** (T1–T7), **v2.1.209** (comment-strip),
  **v2.1.210** (fenced-only + driver staging fix).
- **Span:** opened 2026-08-24 → terminal 2026-08-28 (~4 days).
- **Every code task landed green at 100% coverage**; ~27 PRs total.

## §1 How the plan evolved

The plan's spine held — the 10-task decomposition (compose lib → block ops → sync
→ CLI → audit swap → repo-init → fleet driver → release gate → sweep → tighten)
survived intact, and the release-first-always rollout with an explicit deployment
gate is exactly what ran. The deltas were all **discovered depth**, not
misdirection:

- **Dependency graph correction (mid-flight).** The task issues were filed with
  two wrong `Blocked-by` links: T5 (audit swap) was blocked on T3 though it needs
  only T1–T2, and T6 (repo-init) was blocked on T2 though it *converts the
  flagship `.gitignore`* and therefore needs T5's transitional audit merged first
  (else the flagship's fenced form fails its own CI). Corrected before dispatch,
  which also unlocked real parallelism (T3∥T5, then T4∥T6).
- **Comment-stripping was unplanned (#2939).** The plan's migration removed loose
  *pattern* lines but kept comment lines. The sweep **preview** revealed that
  repos carrying the fully-commented monolith would ship a fence followed by a
  now-false "single source of truth … must be a SUPERSET" preamble and a stack of
  orphaned section headers. That forced a new task + a second release before the
  sweep — and taught the nuance that most repos had *hand-written* `.gitignore`s
  whose own paraphrased headers must **not** be auto-stripped (that would delete
  genuine repo-local comments).
- **The baseline grew during the epic.** The "55 patterns" the spec enumerated
  had become 62 by implementation time (Conan `build-sanitize/` + CMakeDeps globs
  landed in parallel). The lossless-split invariant absorbed this cleanly — it
  targets the *current* baseline, so the cpp fragment simply absorbed the extra
  lines, and T10 froze the real 62.
- **The sweep became a fleet-wide canary.** Running the first PR against
  long-untouched repos surfaced pre-existing debt the migration had nothing to do
  with: a markdownlint MD036 violation (LMF `.github`), two repos missing the CI
  `docs` job their branch protection required (LMF `docs`, observability), a pip
  PYSEC CVE (observability), and the `build/` vs `!build/.gitkeep` semantic
  conflict (LMF lab). Each became its own small fix.

## §2 Lessons learned

- **Mocking `subprocess` globally hides integration bugs.** The fleet driver
  shipped (T7, green, 100% covered) with a fatal gap — it never `git add`ed
  before `vrg-commit`, so *every* real run failed "no staged changes." The tests
  patched the shared `subprocess` module, so all git went through mocks and the
  missing stage was invisible. The fix (#2946) deliberately runs against a **real
  throwaway git repo**, faking only the `vrg-*` tools — and that test fails
  without the stage. Lesson: a driver whose whole job is orchestrating real
  side-effecting commands needs at least one test that lets the real commands run.
- **A migration sweep is a free fleet audit.** The first CI run against a
  rarely-touched repo flushes out latent lint/dep/config debt. Budget for it:
  ~4 of the 12 sweep repos needed an unrelated fix before their `.gitignore` PR
  could merge.
- **Compose + a lossless-split invariant makes deletion safe.** Because
  `base ∪ all-fragments` provably reconstitutes the monolith, we could delete the
  monolith outright and filter foreign-language lines against the *complete*
  vocabulary — no frozen legacy constant needed at runtime, only in the test.
- **The two-layer seam paid off immediately.** Keeping the file-change
  (applicator) separate from the git/PR work-chain (driver) meant the driver bug
  was one localized fix and the applicator was independently, exhaustively
  testable.
- **Release-first-always is non-negotiable for fleet tooling.** The deployment
  gate (#2926) verifying the *released* tag before the sweep is what guaranteed
  the fence the sweep wrote equalled the fence the released audit enforced.

## §3 Compromises & tradeoffs

- **The fleet driver's first real use was a manual recovery.** Rather than clean
  up 12 half-created issues/branches and re-run after fixing the staging bug, we
  completed the 12 conversions by hand (they were correct in the worktrees) and
  fixed the driver separately. Pragmatic, but it means the driver's staging fix
  (#2946) has not yet been exercised by a real fleet run — the next sweep is its
  first true test.
- **Orphaned section-header comments accepted.** In repos with hand-written
  `.gitignore`s, the migration leaves the repo's own (now-empty) category
  comments below the fence. We chose *not* to strip these — the safe-detection
  boundary can't distinguish them from genuine repo-local comments without risking
  data loss — so a few PRs carried cosmetic residue a reviewer could tidy.
- **`build/` in base forced a repo decision.** base's `build/` is stricter than
  LMF lab's `build/*` + `!build/.gitkeep`. We resolved it by **dropping the
  `.gitkeep`** (base ignores the whole dir; consumers `mkdir` as needed) rather
  than complicating base — correct for the fleet, but it did change one repo's
  behavior.
- **The generic driver was deferred.** #328 (extract the work-chain into a
  general standards-sweep engine) is stashed under the ad-hoc epic, not built.

## §4 New problems & opportunities

- **Branch-protection ↔ CI drift is unchecked.** Two repos required a `docs / docs`
  status check their `ci.yml` never produced (missing `docs` job), permanently
  blocking every PR — invisible until a PR tried to merge. **Opportunity:** a
  standards check that a repo's required branch-protection checks are actually
  produced by its workflows. *(Logged here; not yet filed.)*
- **`vrg-finalize-pr` stale-check-while-BEHIND race** — surfaced during the #322
  sweep and filed as **vergil-tooling#2914**: the failed-checks gate hard-aborts
  on a stale red check when the branch is BEHIND, before the update-branch path
  runs. *(Filed, unscheduled.)*
- **Agents cannot push `.github/workflows/**`** — a correct safety guard, but it
  means any CI-workflow fix must be human-submitted; an agent can't quietly amend
  one onto an already-submitted branch. Worth documenting as a workflow
  expectation. *(Logged.)*
- **Generic fleet-sweep driver** — the reusable engine for fleet-wide standards
  enforcement. *(Captured as idea #328 under the ad-hoc epic.)*

## §5 What's next

- **#328 — generic fleet-sweep driver** (ad-hoc epic #99): promote to its own
  epic when a second concrete consumer exists to pull the abstraction against.
- **vergil-tooling#2914** — fix the finalize stale-check/BEHIND race.
- A branch-protection-vs-CI parity check (§4) — file when prioritized.
- First real `vrg-fleet-sync` run on the fixed driver (§3) will validate #2946.

## Appendix A — Operational notes (migration sequence)

The rollout followed release-first strictly:

1. **T1–T7 merged → cut v2.1.208 → deployment gate #2926** verified the released
   tag (transitional audit accepts both forms; `vrg-gitignore-sync` composes the
   intended fence) before any repo was touched.
2. **Sweep preview** (non-destructive `vrg-gitignore-sync --write` on copies of
   each repo's `.gitignore`) caught the orphaned-comment problem → **#2939 →
   v2.1.209** before the live sweep.
3. **Live sweep** via `vrg-fleet-sync` hit the driver staging bug on all 12 →
   **recovered manually** (conversions were correct in the worktrees) → 12 per-repo
   PRs, each human-submitted and merged.
4. **Per-repo blockers resolved as they surfaced:** containers `__pycache__/`
   (#585); LMF docs CI docs-job (#10, then the sweep PR auto-rebased via
   finalize's BEHIND handling); observability CI docs-job + pip bump (bundled,
   #34) — the CI-workflow change required a human `vrg-submit-pr` (agent push
   guard). Superseded the unfinished #322 monolith-append tail (5 LMF issues
   closed).
5. **T10 merged → cut v2.1.210** — audit tightened to fenced-only and the monolith
   deleted only after all 13 repos were confirmed fenced.
