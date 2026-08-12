# Retrospective — Continuous archiving of ad-hoc epics (by close-quarter)

- **Epic:** vergil-project/.github#238
- **Partners:** [`spec.md`](./spec.md) · [`plan.md`](./plan.md)
- **Authored:** 2026-08-08

## §0 At a glance

We set out to stop ad-hoc epics (`Epic (ad hoc): <repo>`) from growing without
bound. They are perpetual umbrellas that are never closed, so over time they
become huge lists of mostly-closed issues in which current work is lost. The epic
delivers **continuous draining**: closed children of a live ad-hoc epic are
re-parented into **per-quarter archive epics** (`Epic (ad hoc): <repo> —
<YYYY>-Q<n>`), bucketed by each child's **close-quarter** — so the live epic
always holds only currently-open work, and finished work flows out into archives
without ever renaming, replacing, or closing the live epic. That is exactly what
shipped, driven by three call sites (a manual CLI, the `issues.closed` event
hook, and the daily audit sweep) and **validated live against real GitHub**.

### Work delivered

| PR | Task | What it did |
|---|---|---|
| [.github#241](https://github.com/vergil-project/.github/pull/241) | #239 | spec + plan for the epic (design bookend) |
| [#2683](https://github.com/vergil-project/vergil-tooling/pull/2683) | #2678 | foundations — `closedAt` threading, `quarter_of`/`current_quarter`, live/archive epic finders |
| [#2684](https://github.com/vergil-project/vergil-tooling/pull/2684) | #2679 | drain engine — per-repo `plan/apply` + visibility-aware org-wide drain |
| [#2685](https://github.com/vergil-project/vergil-tooling/pull/2685) | #2680 | `rollup()` event-path hook — archive a just-closed child (steady state) |
| [#2686](https://github.com/vergil-project/vergil-tooling/pull/2686) | #2681 | `vrg-adhoc-epic archive` subcommand (`--repo`/`--all-in`/`--apply`) |
| [#2687](https://github.com/vergil-project/vergil-tooling/pull/2687) | #2682 | daily `vrg-epic-audit --close` sweep — backstop + backlog + past-archive close |
| [#2692](https://github.com/vergil-project/vergil-tooling/pull/2692) | #2691 | **fix** — atomic re-parent via `addSubIssue(replaceParent: true)` |
| [#2696](https://github.com/vergil-project/vergil-tooling/pull/2696) | #2676 | docs — document the feature in `standards/github-issues.md` |
| *(this PR)* | #240 | retrospective |

Plus **#2677 — live validation** (an operational task; no PR, closed on a
recorded `Outcome: SUCCESS`).

- **Repos touched:** `vergil-tooling` (all code + docs), `vergil-project/.github`
  (spec/plan/retrospective docs; also where the archive epics live).
- **Counts:** 10 child tasks (incl. 3 bookends: spec/plan, docs-review,
  retrospective); 8 merged PRs (7 in `vergil-tooling`, 1 in `.github`) + this
  retrospective PR; 1 validation task with no PR.
- **Releases:** shipped across the 2026-08-08 releases **v2.1.181** and
  **v2.1.182**; the #2691 re-parent fix was released **and installed** before the
  #2677 re-validation. (Exact per-PR→release mapping was not separately tracked.)
- **Span:** opened and closed **2026-08-08** — a same-day epic.

## §1 How the plan evolved

**Before code — a design pivot (recorded in `spec.md`).** The first design was a
*quarterly-batch* roll-up: rename the live epic, create a fresh one, migrate open
children. Pushback replaced it with the *continuous-drain* design — re-parent each
closed child into its close-quarter archive, leaving the live epic's identity
untouched — **explicitly to eliminate the rename/replace dance and its
crash-recovery complexity.** Hold that reason; §2 returns to it.

**The plan's 9 tasks mapped onto 5 code issues.** Tasks 1–3 → #2678, Tasks 4–5 → #2679,
Task 6 → #2680, Task 7 → #2681, Task 8 → #2682; the plan's Task 9 (full
validation) folded into each task's own green-gate rather than a separate issue.
All five went green on first implementation and merged the same day.

**The central deviation — a bug the plan's own tests could not see.** The plan
specified re-parenting as **add-before-remove**: `add_child(archive, child)` then
`remove_child(live, child)`, with unit tests asserting exactly that ordering
("never orphan-under-neither"). Those tests passed because they mock the GitHub
boundary. But the live-validation bookend (#2677), running the real
`vrg-adhoc-epic archive --apply` against real GitHub, **failed**: GitHub's native
sub-issues enforce a **single parent**, so `addSubIssue` is rejected while the
child is still linked to the live epic —

```text
gh: Failed to add sub-issue #243 to parent #242. Sub issue may only have one parent
```

add-before-remove can therefore *never* progress; it fails safe (the child stays
under the live epic) but is deadlocked. This forced a **new fix task not in the
plan** (#2691): replace the two-step with an **atomic** `addSubIssue(input: {…,
replaceParent: true})` — proven live to move a child between parents in one call —
and drop the separate remove entirely. Applied to **both** the batch drain and the
`rollup()` hook.

**Minor deltas.** The `_find_epic_by_title` refactor kept the "multiple ad-hoc
epics" error wording to keep a pre-existing test green; ruff auto-rewrote
`timezone.utc` → `UTC` (UP017) across tasks; #2682 needed a `github.target_org`
wrapper around its mutation (confirmed empirically — matched the plan's implementer
note); the docs task deliberately spawned **no** per-repo doc tasks (the canonical
convention text lives in this repo's `github-issues.md`, and the other repos'
epic-ops workflows have no consumer-facing doc); and the #2676 issue's stale
"rollup subcommand" wording was corrected to the shipped `archive`.

## §2 Lessons learned

- **Mocked unit tests give false confidence on real-API invariants.** The plan's
  Task 4/6 tests mocked `add_child`/`remove_child` and "proved" add-before-remove
  correct. The constraint that broke it — single-parent — exists only in real
  GitHub. **The live-validation bookend is not ceremony.** Without #2677 this
  would have shipped and silently broken *every* ad-hoc child close in production
  (the `rollup()` hook is the steady-state path). Keep the live bookend for any
  feature whose correctness depends on an external system's rules.
- **Prefer an atomic primitive over a compensating sequence.** add-before-remove
  was a hand-rolled two-step invented to avoid an orphan window. The platform's
  atomic `replaceParent` both *fixed the bug* and *removed the orphan concern
  entirely* — strictly simpler and more correct. When the platform offers an
  atomic operation, reach for it before inventing a safe ordering.
- **The design's instinct was right; the first implementation lost it.** The
  continuous-drain design was chosen precisely to avoid orphan/crash-recovery
  complexity (§1). The naive add-before-remove quietly reintroduced exactly that
  concern — and got it wrong. `replaceParent` restored the design's promise. When
  an implementation detail starts reasoning about orphan windows, that is a signal
  the chosen primitive is fighting the design.

## §3 Compromises & tradeoffs

- **Multi-quarter historical distribution is not live-tested.** GitHub sets
  `closedAt` to *now*, so you cannot stage a past-quarter-closed issue. The live
  validation proves real re-parenting, current-archive-open, past-archive-close,
  and idempotency; the pure "bucket each child by its own close-quarter across
  many quarters" math is covered only by the mocked unit tests. Accepted as an
  inherent limit, not a gap.
- **Validation staging left closeable residue.** The gated tooling can close but
  not delete issues, so the throwaway staging left a handful of closed issues
  (.github#242–#247), each commented "safe to delete." Flagged for optional manual
  deletion.
- **Past-archive close is home-scoped, not target-scoped.** `plan_adhoc_drain`
  computes the archives-to-close from `list_open_adhoc_archives(home)` — every
  ad-hoc archive in the home repo, not just the target bare repo's. Benign today
  (all `.github` ad-hoc archives are archives, and "past ⇒ closeable" holds
  globally), but a subtlety worth remembering if archive homing ever changes.

## §4 New problems & opportunities

- **Single-parent / add-before-remove bug** — surfaced by #2677, fixed by #2691.
  Resolved within the epic.
- **Orphaned-issue handling remains a known gap.** `replaceParent` removes the
  orphan window *here*, but the broader invariant "every non-epic issue belongs to
  an epic" is still unenforced (a separate effort). Logged, not acted on in this
  epic.
- **Brief list-consistency lag.** A freshly-created archive was not immediately
  visible to the next `gh issue list` (the dry-run omitted it; it reappeared
  seconds later). Benign for an idempotent drain that converges on re-run; noted.
- **Live-validation teardown friction.** No sanctioned way to hard-delete test
  fixtures (only close). A test-fixture teardown path or a dedicated sandbox repo
  would make future live validations cleaner. Logged, not acted on.

## §5 What's next

- **Automation is live.** The fix is released and installed, so the `issues.closed`
  rollup hook and the daily `vrg-epic-audit --close` sweep now drive continuous
  archiving in production. The one-time backlog distribution (~55+ current-quarter
  closed children across `vergil-tooling`/`vergil-containers`/`vergil-actions`)
  will flow into 2026-Q3 archives via the deployed sweep — or sooner via a manual
  `vrg-adhoc-epic archive --all-in vergil-project --apply`.
- **Candidate follow-ons** (referenced, not scoped here): enforce "every non-epic
  issue belongs to an epic" + orphan recovery; a sanctioned test-fixture/sandbox
  path for live-validation staging so throwaways can be torn down cleanly.

## Appendix A — Operational notes

- **Live validation method (#2677).** Staged an isolated throwaway scenario in
  `vergil-project/.github` (target repo `.github`, live epic #101 which had zero
  real closed children): one throwaway closed child under #101 plus a pre-staged
  open past-quarter archive, then ran the dev tree scoped to `--repo
  vergil-project/.github` only — the real backlog (#99/#100/#102, all
  current-quarter) was never touched. Cleaned up afterward; `.github` restored to
  its original state.
- **Deploy status.** Merged to `develop` → released (v2.1.181–v2.1.182) →
  installed, all on 2026-08-08. The #2691 fix was confirmed released+installed
  before the #2677 re-validation returned SUCCESS.
