# Retrospective — Retrospective bookend for the epic lifecycle

- **Epic:** vergil-project/.github#201
- **Authored:** 2026-07-26 (via `epic-retrospective`, dogfooded on its own epic)

## §0 At a glance

We set out to give every epic a **mandatory, single, terminal retrospective
bookend** — the backward-looking counterpart to the opening spec+plan. We shipped
exactly that, plus more than we expected: a new `epic-retrospective` skill (with a
preflight definition-of-done gate), the reframing of `epic-create` /
`epic-implement` / `CLAUDE.md` around it, the correction of the human-facing
*standards* that still described the old model, and — discovered while writing
this document — three small tooling improvements the epic surfaced and then
**fixed within itself**. #201 is the **first epic to carry a retrospective of its
own**, and this is it.

**Work delivered** (8 merged PRs across 3 repos; this retrospective is the 9th):

| PR | Repo | What it did | Release |
|----|------|-------------|---------|
| [#204](https://github.com/vergil-project/.github/pull/204) | `.github` | Spec + plan | — |
| [#661](https://github.com/vergil-project/vergil-claude-plugin/pull/661) | `vergil-claude-plugin` | New `epic-retrospective` skill (terminal finishing gate) | 2.1.36 |
| [#665](https://github.com/vergil-project/vergil-claude-plugin/pull/665) | `vergil-claude-plugin` | Reframed `epic-create`/`epic-implement`/`CLAUDE.md` — retrospective as final gate | 2.1.37 |
| [#669](https://github.com/vergil-project/vergil-claude-plugin/pull/669) | `vergil-claude-plugin` | Skills-site catalogue: added `epic-create`/`epic-retrospective` | 2.1.38 |
| [#2533](https://github.com/vergil-project/vergil-tooling/pull/2533) | `vergil-tooling` | Authoritative epic-lifecycle standard rewritten for the retrospective bookend | 2.1.166 |
| [#2541](https://github.com/vergil-project/vergil-tooling/pull/2541) | `vergil-tooling` | `vrg-issue-create --kind retrospective` (label + scaffold) | 2.1.167 |
| [#2542](https://github.com/vergil-project/vergil-tooling/pull/2542) | `vergil-tooling` | `vrg-epic-audit --epic <ref>` — machine-readable children + states | 2.1.167 |
| [#2543](https://github.com/vergil-project/vergil-tooling/pull/2543) | `vergil-tooling` | `vrg-gh release` (read-only) added to the allowlist | 2.1.167 |

- **Repos touched:** 3 — `vergil-claude-plugin`, `.github`, `vergil-tooling`.
- **Tasks:** 9 (5 planned at creation + 1 spawned by the doc-review sweep + 3
  absorbed at the retrospective gate). **PRs merged:** 8.
- **Releases cut:** plugin 2.1.36 → 2.1.38; tooling 2.1.166 → 2.1.167. *(Queried
  authoritatively via `vrg-gh release` — the subcommand this epic added in #2543.
  An earlier draft **inferred** these from version-bump commits and was off by
  one; the new command corrected it.)*
- **Span:** opened 2026-07-24 19:49Z; last task merged 2026-07-26 10:46Z;
  retrospective same day. ~2 calendar days, ~1 of active execution.

## §1 How the plan evolved

The plan's core sequence held exactly — **skill (#659) → reframe (#660) →
doc-review (#658)** — each green on first validation (one trivial `markdownlint`
fix). The design survived pushback and alignment intact; the two decisions that
mattered were made *before* execution (dropping static `Blocked-by` reflinks for
the skill-preflight gate, and naming `vrg-epic-audit` as the §0 data spine).

Two deviations are recorded in `plan.md`'s Evolution log — its **first real
entries**, and exactly the feedstock this section exists to synthesize:

1. **The epic was not "plugin-contained" (2026-07-25).** It was filed on that
   belief. The documentation-review sweep (#658) found the *authoritative*
   bookend standard in `vergil-tooling` and a stale catalogue in the plugin, and
   spawned `#2531`. Task count 5 → 6. The closing bookend did precisely the job
   it exists to do.
2. **Three tooling improvements absorbed at the gate (2026-07-26).** Drafting the
   retrospective surfaced three small gaps; rather than fork them to a backlog,
   we pulled them into the epic (#2537/#2538/#2539). Task count 6 → 9. This
   reopened the frontier and (correctly) held the terminal retrospective.

## §2 Lessons learned

- **The closing bookends earned their place on their first outing.** The
  doc-review sweep caught a wrong scope assumption; the retrospective gate caught
  an epic that wasn't actually done. Neither was theoretical — both fired on the
  very first epic to use them.
- **The dynamic preflight gate beat static bookkeeping — twice.** Because
  terminality is enforced by enumerating open children at run time, both the
  spawned `#2531` and the three absorbed tasks were caught automatically, with no
  dependency list to maintain. The decision to drop `Blocked-by` reflinks
  validated itself.
- **"Absorbed at the gate" turned finds into fixes.** Applying the
  do-it-now-if-it's-small discipline, the three discovered gaps ship *inside*
  this epic — so the retrospective reads "found and fixed," not "found and
  filed." Small items that get backlogged tend to rot; these didn't.
- **Dogfooding compounded.** The tools discovered as gaps were then used to
  *finish the epic that produced them*: `vrg-epic-audit --epic` (#2542)
  re-verified the terminal gate, and `vrg-gh release` (#2543) corrected this
  document's own §0 release figures. The epic improved the very machinery it ran
  on.

## §3 Compromises & tradeoffs

- **The Evolution log was reconstructed, then earned.** A ~1-day core meant the
  first draft's §1 leaned on session memory; but the two real deviations *were*
  logged as they happened, so the log is now genuine — not the empty formality it
  might have been.
- **`--kind retrospective` shipped (#2541), but the skill hasn't adopted it yet.**
  The `epic-retrospective` preflight still identifies the retrospective task by
  **title convention** (`Retrospective: <slug>`); switching it to the new label
  is a small plugin follow-up (§5). We fixed the tooling gap without yet closing
  the loop in the skill.
- **We were honestly wrong once, and caught it.** The pre-#2543 draft inferred
  release numbers from `bump version` commits and got them off by one. The fix we
  shipped exposed the error before this document merged. Owning it beats hiding
  it.

## §4 New problems & opportunities

The three opportunities the earlier draft would have *filed* were instead
**resolved within the epic** (#2541/#2542/#2543). What remains:

- **Adopt the `retrospective` label in the `epic-retrospective` preflight** —
  replace the title-convention match with the mechanical label now that #2541
  ships it. Small plugin follow-up. *Logged; not filed.*
- **An `epic-create` "where do the docs actually live?" prompt** — front-load the
  doc-surface question so a containment assumption is checked at creation, not at
  the closing sweep (this epic's `#2531` miss). *Logged; not filed.*

## §5 What's next

No follow-on epic is seeded — there is no known enabling chain, and the
three-skill lifecycle is now complete and documented end to end, on *better*
tooling than when the epic started. The two small candidates in §4 are recorded
here rather than spun into work; file them as `triage` if they earn priority. The
real "what's next" is that **every future epic now closes through this gate** —
and #201 proved it does, on itself.
