# Retrospective — Ad-hoc archiving hardening (duplicate-archive race fix + recovery)

- **Epic:** vergil-project/.github#259
- **Partners:** [`spec.md`](./spec.md) · [`plan.md`](./plan.md)
- **Part one:** vergil-project/.github#238 (closed) — the feature this hardens.
- **Authored:** 2026-08-09

## §0 At a glance

Part one (#238) shipped continuous ad-hoc archiving; its first production
`--all-in --apply` run then created **duplicate same-quarter archives** and
skipped two repos. This epic set out to fix the race and recover the damage — and
did both: a small code change (ensure each `(repo, quarter)` archive **once per
drain**, and reuse the oldest archive if duplicates already exist), released as
**v2.1.183**, followed by a one-time operational recovery that consolidated every
duplicate and finished the migration. The backlog now holds exactly one archive
per quarter and every live ad-hoc epic is drained.

### Work delivered

| PR / task | Task | What it did |
|---|---|---|
| [.github#262](https://github.com/vergil-project/.github/pull/262) | #260 | spec + plan (design bookend) |
| [#2701](https://github.com/vergil-project/vergil-tooling/pull/2701) | #2698 | **fix** — ensure archive once per quarter; duplicate-tolerant resolver (`prefer_oldest`) |
| #2699 *(operational, no PR)* | #2699 | **deploy** — released v2.1.183 + installed; verified the fix in the installed binary (`Outcome: SUCCESS`) |
| #2700 *(operational, no PR)* | #2700 | **recovery** — consolidated duplicates + re-drained the backlog + verified (`Outcome: SUCCESS`) |
| [#2705](https://github.com/vergil-project/vergil-tooling/pull/2705) | #2697 | docs — note the once-per-quarter guarantee + tolerance |
| *(this PR)* | #261 | retrospective |

- **Repos touched:** `vergil-tooling` (code + docs), `vergil-project/.github`
  (spec/plan/retrospective docs; where the archive epics live).
- **Counts:** 6 child tasks (3 bookends: spec/plan, docs-review, retrospective);
  3 merged PRs + this retrospective; 2 operational tasks (deploy, recovery) with
  no PR, both `SUCCESS`.
- **Release:** the fix shipped in **vergil-tooling v2.1.183** (2026-08-09).
- **Span:** opened 2026-08-08, closed 2026-08-09.

## §1 How the plan evolved

**Small delta — the plan held.** The three planned tasks (code fix → deploy →
recovery) ran as written, in order, with no re-scoping. The code change matched
the plan's snippet (per-quarter cache in `apply_adhoc_drain`; `prefer_oldest`
flag on `_find_epic_by_title`, scoped to archive resolution so the live-epic
finder still raises on real corruption). The recovery followed its procedure
exactly: consolidate each `(repo, quarter)` into the lowest-numbered keeper, close
the emptied duplicates, re-drain, verify. A small planning-to-execution delta is
the sign the front-loaded design was right.

**The one framing correction owed to part one.** This epic exists *because* #238's
retrospective §4 flagged the `gh issue list` consistency lag and then rated it
**"benign for an idempotent drain."** It was not: inside a single drain with many
same-quarter children, the lag turns a per-child find-or-create loop into
duplicate archives. The flag was correct; the *assessment* was wrong — and a wrong
"benign" is worse than an unflagged risk, because it launders false confidence
into the record. The first production `--all-in` run is what exposed it.

## §2 Lessons learned

- **A mis-rated risk is worse than an unlisted one.** #238 §4 saw the exact hazard
  and dismissed it. When a retrospective (or review) flags something, the
  *severity call* is load-bearing — "benign" closes the door on it. Prefer "not yet
  observed to bite; would bite if N same-quarter children" over a bare "benign."
- **Create-then-read against an eventually-consistent index is a footgun.**
  Find-or-create keyed on a listing that lags creation will double-create under a
  tight loop. The remedy is structural: do the create **once** and **cache** it
  for the rest of the operation — not "retry" or "sleep."
- **Make find-or-create tolerant so it self-heals.** Raising on "found two" turned
  a transient race into a hard, sticky skip. Reusing the oldest degrades a
  duplicate to a no-op and lets the next run recover — while still *raising* for
  the one case where duplicates mean genuine corruption (two live ad-hoc epics).
  Tolerance and strictness are per-context, not global.

## §3 Compromises & tradeoffs

- **Duplicate-tolerance is archive-only, by design.** `find_adhoc_epic` /
  `ensure_adhoc_epic` still raise on duplicate *live* epics — that is corruption,
  not a race, and must stay loud. Only archive resolution reuses the oldest.
- **Closed, not deleted.** The 6 consolidated duplicate archives (and part-one's
  validation throwaways) are **closed**, not hard-deleted — the gated tooling can
  close but not delete issues. Harmless (the resolver filters to open archives),
  but they remain as closed issues; hard-deletion is a manual `gh issue delete` if
  ever wanted.
- **Cross-process races not fully eliminated.** The per-quarter cache fixes the
  single-drain loop; two *concurrent* drains (sweep + a manual run) could still
  race. The duplicate-tolerant resolver degrades that to "reuse the oldest" rather
  than a hard failure, which was judged sufficient (the daily sweep is serialized).

## §4 New problems & opportunities

- **No new defects surfaced.** The recovery re-drain, run against the deployed fix,
  completed with no skips and an idempotent second pass.
- **Carried forward from #238 §4/§5** (still logged, not acted on): enforce "every
  non-epic issue belongs to an epic" + orphan recovery; a sanctioned
  test-fixture/sandbox teardown path so live-validation and recovery throwaways can
  be hard-deleted cleanly rather than left closed.

## §5 What's next

- **Watch the deployed automation.** The daily `vrg-epic-audit --close` sweep and
  the `issues.closed` hook now run the fixed v2.1.183 code; the backlog is clean
  and one-archive-per-quarter. No further work is chained — monitor that the next
  quarter rolls over cleanly (current-quarter archive stays open, past closes).
- No follow-on epic is warranted; the two carried items above remain candidate
  future work, not a committed chain.

## Appendix A — Operational notes (recovery, #2700)

Run against the installed, released fix (v2.1.183), scoped with
`github.target_org("vergil-project")`:

1. **Consolidation** — per `(repo, 2026-Q3)` group, keeper = lowest-numbered;
   atomically `replaceParent` every straggler into the keeper, then close the
   emptied duplicate:
   - vergil-tooling → keeper **#249**, closed **#250**
   - vergil-containers → keeper **#251**, closed **#252, #253**
   - vergil-actions → keeper **#254**, closed **#255, #256**
   - vergil-claude-plugin → keeper **#257**, closed **#258**
2. **Re-drain** — `vrg-adhoc-epic archive --all-in vergil-project --apply` migrated
   the remaining **40** (vergil-tooling) + **6** (vergil-containers) closed children
   into their Q3 keepers, with **no skips and no new duplicates**.
3. **Verification** — one open archive per `(repo, quarter)`: #249 (44), #251 (9),
   #254 (3), #257 (4); every live ad-hoc epic at **0** closed-unmigrated children;
   a second `--apply` a clean no-op.
