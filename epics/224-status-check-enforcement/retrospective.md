# Retrospective: status-check-enforcement

**Epic:** vergil-project/.github#224
**Span:** opened 2026-08-06 → closed 2026-08-07 (~1 day)

## §0 At a glance

We set out to fix two orthogonal defects that surfaced from one incident
(`vrg-finalize-pr 934` hanging forever) and shared one theme — *how we manage the
status checks that gate a PR merge*. What shipped: `docs / docs` (and every
universal reusable-CI check) is now a **required** PR gate enforced by
`vrg-github-repo-config`; `vrg-finalize-pr` is **resilient to GitHub-orphaned
check-runs** (bounded watch → raise, never hang or merge past); the config was
**applied fleet-wide** (24 repos, 6 orgs); and the "no optional gate" principle
was extended into the standards docs (hard gates only, with a single named
advisory carve-out).

### Work delivered

| PR | Task | What it did |
|---|---|---|
| .github#229 | #225 | spec + plan |
| tooling#2614 | #2597 (A1) | require `docs / docs` + pin the universal reusable-CI check set (completeness test) |
| tooling#2615 | #2598 (B) | finalize: bounded check-wait; raise on orphaned check-run instead of hanging |
| — (no PR) | #2599 | deployment: `vrg-github-repo-config apply` across 24 managed repos |
| tooling#2624 | #2596 | doc-review: finalize + ci-architecture docs (required-gate invariant, orphan handling) |
| tooling#2631 | #2625 | source-control-guidelines → hard gates only (eliminate soft gates) |
| tooling#2632 | #2630 | reframe preview/eol advisory as the unsupported-version carve-out + fix stale cross-refs |
| .github (this PR) | #226 | this retrospective |

- **Repos touched:** `vergil-tooling` (code: `github_config.py`, `github.py`,
  `pr_merge.py` + tests; docs: 8 site pages), `vergil-project/.github` (epic
  spec/plan/retrospective), plus **24 managed repos across 6 orgs**
  (branch-protection config via the deployment — no code changes there).
- **Counts:** 9 sub-issues; 6 code/doc PRs + this retrospective PR (the
  deployment task closed via an `Outcome:` comment, no PR).
- **Releases:** `vergil-tooling` v2.1.174 → v2.1.175 (2026-08-07), human-run,
  installed fleet-wide (host + all Vergil VMs); A1+B shipped in this release
  train before the fleet apply.

## §1 How the plan evolved

The plan held on its core (A1 contract completeness + B orphan resilience +
deployment), but the epic grew and shifted in three notable ways:

- **A2 was deferred before implementation.** Alignment surfaced that an empirical
  "produced-vs-required" reconciler cannot run at config-apply time (no PR exists
  yet on a new/quiet repo), and its only consumer — the daily ops job — was out
  of scope. So A2 (and the daily job) moved to follow-on #227, leaving this
  epic's enforcement as the contract layer, which already covers every universal
  check. A planned *reduction*, made deliberately.
- **The epic grew a soft-gate arm mid-flight.** The doc-review (#2596) surfaced
  that, although no soft gates were *implemented*, several standards docs still
  *prescribed* them — contradicting the new "no optional gate" invariant. Rather
  than backlog it, soft-gate elimination was pulled into the epic: #2625
  (hard-gates-only standard) and #2630 (reframe preview/eol as a named
  advisory-on-unsupported-versions carve-out, not a generic soft gate).
- **The deployment changed hands.** The fleet apply was planned as an agent-run
  operational task, but the agent's `user` identity hit GitHub **403 "Resource
  not accessible by integration"** — it lacks admin/rulesets/actions permissions.
  It became a **human-run** step (handed off as `build/apply-fleet.sh`), then
  verified green across all 24 repos.

## §2 Lessons learned

- **Two symptoms, one theme, still two fixes.** Making `docs / docs` required
  would *not* have fixed the hang (it would have turned `CLEAN` into `BLOCKED`);
  the orphan-resilience fix was needed independently. Resisting the urge to
  collapse them into one change was correct.
- **Enforce at the layer that already has the truth.** Contract-based enforcement
  reused the *existing* `audit` machinery (which already hard-fails on
  required-set drift and gates `vrg-release`); the fix was largely *completeness
  of the desired set*, not new gating code.
- **Agents cannot do fleet config writes.** `vrg-github-repo-config` apply/audit
  is effectively human/admin-privileged (403 under the agent identity). Any
  future automation of it (the #227 daily job) must run under an admin-scoped
  token.
- **"No optional" generalizes.** The same reasoning that made an optional PR
  check unacceptable applies to soft gates — a check you will not fail on
  silently rots.

## §3 Compromises & tradeoffs

- **A2 deferred** (contract layer ships now; empirical repo-local drift detection
  waits for #227). Accepted: the contract layer covers every universal check,
  which is where the incident lived.
- **Orphan-wait edge case shipped, not blocked.** The restructured
  `wait_for_checks` returns immediately when *no* checks have registered yet
  (`all([]) is True`), losing the old #1490 registration-wait. Backstopped by the
  strict required-checks policy (GitHub refuses to merge un-run required checks),
  so a follow-up (#2620) was filed rather than reopening the frozen branch.
- **Soft-gate carve-out, not a purge.** `preview-`/`eol-` advisory checks were
  kept as one explicit named exception (advisory *because* the version is outside
  the supported set) rather than force-converting them to hard gates.

## §4 New problems & opportunities

- **`wait_for_checks` #1490 regression → #2620** (ad-hoc, logged).
- **The root cause is a GitHub bug.** Orphaned check-runs (non-terminal after the
  run completes) are GitHub-side; we made the tooling resilient, not the platform
  correct. Worth watching for recurrence.
- **Agent identity cannot audit/apply repo config (403).** Beyond this epic: it
  gates the #227 daily job and any agent-run config enforcement — needs an
  admin-scoped identity decision.
- *(Session-adjacent, outside the epic's scope but surfaced alongside it:*
  per-branch dev-image disk accumulation → triage #228; `vrg-validate`
  markdownlint file-count nit → #2601.)*

## §5 What's next

- **Follow-on #227 — daily drift-reporting ops job for `vrg-github-repo-config`**,
  which also carries the deferred **empirical repo-local reconciler** (A2). It is
  the intended production enforcement of the "no optional gate" invariant, and it
  needs the admin-scoped identity noted in §2/§4. Captured (closed as
  captured-for-later), not yet developed — pick up via its own `epic-create` when
  prioritized.

## Appendix A — Operational notes (deployment)

The fleet apply (`vrg-github-repo-config apply --repo <each>`) ran across **24
managed repos in 6 orgs** (vergil-project, logical-minds-foundry,
mq-rest-admin-project, diogenes-project, vergils-nemesis, wphillipmoore). Because
the agent `user` identity is 403 for config writes, it was **human-run** from
`build/apply-fleet.sh` with admin credentials, then verified: all 24 repos'
`vrg-github-repo-config audit` → exit 0 (compliant); `docs / docs` now required
everywhere. The apply is idempotent (re-runs stay green). Recorded on #2599 as
`Outcome: SUCCESS`.
