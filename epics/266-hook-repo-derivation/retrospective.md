# Retrospective — Event-hook repo derivation + non-repo epic guard

- **Epic:** vergil-project/.github#266
- **Partners:** [`spec.md`](./spec.md) · [`plan.md`](./plan.md)
- **Line:** third in the ad-hoc archiving series — #238 (feature), #259 (dup-race
  fix), this (event-hook repo derivation).
- **Authored:** 2026-08-09

## §0 At a glance

Investigating whether the archiving logic was safe for another org
(logical-minds-foundry — which has the corner cases Vergil lacks: a private repo
and a special-purpose `ad-hoc`-labelled epic) surfaced a bug in the **event-path
rollup hook**: it derived the archive's repo from *where the epic lives*
(`parent.repo`, always `.github` for public repos) instead of *which repo it's
for* (the title's bare name). This epic fixed the derivation and added a
canonical-repo guard, shipped it as **v2.1.184**, and recovered the children the
hook had already misfiled in Vergil.

### Work delivered

| PR / task | Task | What it did |
|---|---|---|
| [.github#269](https://github.com/vergil-project/.github/pull/269) | #267 | spec + plan (design bookend) |
| [#2712](https://github.com/vergil-project/vergil-tooling/pull/2712) | #2709 | **fix** — derive archive repo from epic title bare name; skip non-repo epics; new `github.repo_exists` guard |
| #2710 *(operational, no PR)* | #2710 | **deploy** — v2.1.184 released + installed; fix verified in the binary (`Outcome: SUCCESS`) |
| #2711 *(operational, no PR)* | #2711 | **recovery** — re-filed Vergil's misfiled children (`Outcome: SUCCESS`) |
| [#2716](https://github.com/vergil-project/vergil-tooling/pull/2716) | #2706 | docs — archiving targets only canonical per-repo ad-hoc epics |
| *(this PR)* | #268 | retrospective |

- **Repos touched:** `vergil-tooling` (code + docs), `vergil-project/.github`
  (spec/plan/retrospective docs; archive epics).
- **Counts:** 6 child tasks (3 bookends); 3 merged PRs + this retrospective; 2
  operational tasks (deploy, recovery), both `SUCCESS`.
- **Release:** the fix shipped in **vergil-tooling v2.1.184** (2026-08-09).
- **Span:** opened and closed **2026-08-09** — a same-day epic.

## §1 How the plan evolved

**The plan held; the recovery grew.** The three tasks (fix → deploy → recover) ran
as written. The one delta was scope drift *inside* the recovery: because the hook
kept misfiling on every public-repo ad-hoc child close until the release landed,
by the time T3 ran the Vergil damage had grown from the 2 children the plan named
(#1356, #1371) to **4** (add #2269, #2213), spread across two `.github` archives.
The recovery simply re-filed whatever it found rather than the enumerated list —
which is the right shape for a "clean up live damage" task.

**The origin is the story.** This epic was not born from a plan; it was born from
a question — "will the drain break the other org's special epic?" The bug was not
in the special epic at all; asking the question forced a read of the event path,
which is where the real, general bug lived.

## §2 Lessons learned

- **Two code paths that do "the same thing" need the same validation.** The batch
  drain and the event hook both re-parent closed children, but only the batch was
  live-validated (#2677). The bug lived entirely in the un-live-tested path.
  Whenever behavior forks into two implementations, the weaker-covered fork is
  where the defect hides.
- **A mock that picks the input where right and wrong agree proves nothing.** The
  hook's one unit test used the `.github` repo's *own* ad-hoc epic — the single
  case where `parent.repo` equals the correct bare name — so it passed while every
  other repo was mis-derived. Test the input where the buggy and correct code
  **diverge**, not one where they coincide.
- **Corner cases you don't have are found by looking where they do exist.** Vergil
  has no private repos and no special epics, so its testing could never surface
  this. Deliberately checking the org that *does* have them (the "go look at the
  other configuration" instinct) is what found it. Cross-configuration review is a
  bug-finding tool, not just a safety check.

## §3 Compromises & tradeoffs

- **The canonical-repo guard costs a repo-existence probe per hook fire.** Cheap
  and memoized, and it must stay a real existence check (not repo-list membership)
  so a **private self-homed** repo — visible to the App token but not to a weaker
  one — still passes. Verified live against logical-minds-foundry's private repo.
- **The cross-process concurrent-drain duplicate is still unhandled** (a known
  #259 §3 tradeoff) and **fired for the first time in the wild** during the
  logical-minds-foundry cleanup (a manual drain racing the org's automation created
  a second private-repo archive). The `prefer_oldest` tolerance self-healed it on
  the next pass, but it is now an observed event, not a theoretical one.

## §4 New problems & opportunities

- **Concurrent-drain duplicate, observed live** — logged as a candidate hardening
  (a per-repo drain lock, or a post-drain dedup pass) if it recurs. Not yet acted
  on.
- **Cross-org remediation was done outside this epic** — per one-org-per-epic, the
  logical-minds-foundry cleanup (consolidate its duplicate archives, drain its
  backlog incl. the private repo, confirm #104 is safe) was executed as direct
  operational work, recorded in Appendix A, **not** a separate LMF epic (chosen for
  expedience over ceremony). If LMF accrues more managed work, an epic there is the
  right home.
- **#104 needed no reclassification.** The non-repo guard makes the hook skip it
  and the `ad-hoc` label keeps the finite-rollup from closing it — the special
  epic is safe as-is once the fix is deployed. A worry that dissolved into a
  no-op.

## §5 What's next

- **Confirm logical-minds-foundry's automation is on v2.1.184+** (moving `@v2.1`
  tag auto-picks it up; a frozen older tag would keep re-dirtying). The only
  open external dependency.
- **Concurrent-drain hardening** is the one candidate follow-on, held pending
  recurrence. No follow-on epic is chained.
- This closes the ad-hoc archiving trilogy (#238 → #259 → #266): a feature, the
  race that its first production run exposed, and the derivation bug that
  cross-org review exposed. The recurring thread across all three:
  **the real bugs were external-invariant bugs found by running against real
  GitHub / real configurations, never by the mocked unit tests.**

## Appendix A — Operational notes

**Vergil recovery (T3, #2711):** re-filed `vergil-tooling#1356/#2269/#2213` (from
`.github#263`) and `#1371` (from `.github#264`) into
`Epic (ad hoc): vergil-tooling — 2026-Q3` (`.github#249`); closed the emptied
`#263`/`#264`. Verified no `.github — <quarter>` archive holds a non-`.github`
child.

**logical-minds-foundry remediation (cross-org, outside this epic):** consolidated
`mq-resiliency-lab-for-linux — 2026-Q3` (#180/#181/#182 → keeper #180) and the
private repo's `mq-gateway-replay-lab — 2026-Q3` (#261/#262 → keeper #261);
re-drained #29's 21 stragglers and the private repo's straggler; left #104
untouched (33 closed children; safe under the fix). Final: one archive per
`(repo, quarter)`, all live epics 0 closed-unmigrated, idempotent re-run.
