# Epic issues link their spec, plan and retrospective — Retrospective

Epic: [vergil-project/.github#819](https://github.com/vergil-project/.github/issues/819) ·
Spec: [`spec.md`](spec.md) · Plan: [`plan.md`](plan.md)

## 0. At a glance

**What we set out to do.** Epic issues named their spec, plan and
retrospective in plain text, if they named them at all. The epic set out to:

- give every epic a standard, marked **Documents** section at the top of its
  body, linking each document or saying in a short note why there isn't one;
- add a `vrg-epic-docs` command to write it (`set`) and to backfill every
  existing epic (`backfill`);
- add a `vrg-epic-audit` check that flags a missing or stale section;
- have `epic-create` and `epic-retrospective` write the links at their PR
  handoffs;
- sweep the whole fleet before closing.

**What shipped.** All of it. There is one addition beyond the spec, and one
by-hand repair of data the backfill would have lost (§3).

- vergil-tooling **v2.1.284**: `vrg-epic-docs set` and `backfill`, the audit
  check, and the site docs.
- vergil-claude-plugin **v2.1.57**: `epic-create` steps 2 and 8 and the
  `epic-retrospective` close-out call `vrg-epic-docs set`.
- **156 epics backfilled across six orgs**, with every audit clean:

| Org | Epics |
|---|---|
| vergil-project | 80 |
| logical-minds-foundry | 61 |
| mnemosys-project | 7 |
| mq-rest-admin-project | 6 |
| diogenes-project | 1 |
| vergils-nemesis | 1 |

The `wphillipmoore` account has no epics.

**Work delivered**

| PR | Repo | What it did |
|---|---|---|
| [#841](https://github.com/vergil-project/.github/pull/841) | .github | Spec and plan |
| [#3899](https://github.com/vergil-project/vergil-tooling/pull/3899) | vergil-tooling | `epic_docs` library and `vrg-epic-docs set` (Task 1, #3896) |
| [#3902](https://github.com/vergil-project/vergil-tooling/pull/3902) | vergil-tooling | `vrg-epic-docs backfill` (Task 2, #3897) |
| [#3911](https://github.com/vergil-project/vergil-tooling/pull/3911) | vergil-tooling | `vrg-epic-audit` Epic documents check and site docs (Task 3, #3898) |
| [#912](https://github.com/vergil-project/vergil-claude-plugin/pull/912) | vergil-claude-plugin | Skills write the section (Task 4, #837) |

- **Repos touched:** `.github`, vergil-tooling and vergil-claude-plugin. The
  deployments also edited epic bodies in five other orgs.
- **Tasks:** 15 children:
  - 4 code tasks;
  - 8 deployments (D1–D8, vergil-project/.github#832–#839);
  - the docs task (#820);
  - the documentation review (vergil-tooling#3895), closed by hand with no
    changes needed;
  - this retrospective.
- **PRs:** 5, plus this one.
- **Releases:** vergil-tooling v2.1.284 and vergil-claude-plugin v2.1.57.
- **Span:** opened 2026-11-01. Implemented and deployed 2026-11-02 to
  2026-11-03.

## 1. How the plan evolved

The spec and plan held up. Every task landed in the order the plan set, and
no task was added or dropped. The changes during execution were small:

- **Scope came out during design.** Reviewing `migrate-repo` against this
  epic showed it is legacy: every repo is migrated, and the orphan check plus
  `intake-review` now do its job. It was taken out of scope and spun off,
  together with a review for other vestigial tools, as planning issue
  vergil-project/.github#831.
- **Docs moved a task early.** Task 1 had to add the `vrg-epic-docs` CLI
  reference entry, because an existing site-docs test fails for any
  registered command without one. Task 3 expanded it rather than adding a
  second.
- **Review findings moved between tasks.** Three of Task 1's round 4 findings
  (a wrapped `Documents:` bullet, fence indentation, the failed-write
  message) were carried into Task 2 rather than fixed with another round on
  Task 1.
- **Discovery grew one prefix.** Task 2 also reads `docs/superpowers/plans/`,
  which spec §5.2 did not list. That was almost certainly an omission from
  the spec, and it was kept by ruling.
- **Real data needed one repair.** The logical-minds-foundry dry run showed
  five legacy `Spec:` lines (#7, #8, #9, #12, #13) carrying notes the spec's
  removal rule would delete: related spec files, and deliverable and stage
  references. They were put back by hand as `Related:` lines below the
  section.
- **The final sweep earned its place.** D8, added during alignment, found one
  epic created after D1 without a section (#881). It also found two epics
  closed as not planned since D1 (#816, #848).

## 2. Lessons learned

- **Review effort was far out of proportion to the task.** The three
  vergil-tooling tasks took 9 agentic-review rounds and about 1.5 million
  review tokens, and paused for human rulings 10 times. The tool was a
  one-time sync of about 150 bodies, all fully known in advance.
  - Each round found a new layer of parser edge cases and "security" items.
    Exploiting any of those items needed edit rights on the epic body, the
    same trust boundary as editing the epic directly.
  - The human called it out mid-epic, and was right. The check that actually
    mattered was one person reading the dry run over the real data, which
    took minutes and found the one real data problem (the five `Related:`
    notes).
  - **Next time:** when a tool's input is a small, fixed and fully known
    data set, say so in the plan, and make reviewing the dry run against
    that data the gate. Recommend accept-branch once a round finds only
    polish or same-boundary security items, rather than passing each layer
    to the human.
- **Dry-run-first with a condensed table works.** One line per epic, plus
  the list of removed lines, let the human approve 155 body edits in two
  short reads. The `Related:` problem was visible only there.
- **Review-hygiene rules cost more than they caught here.**
  - The stray-write sweep failed rounds on files no reviewer wrote: other
    sessions' `/tmp` round directories, and this session's own task output.
    The human gave a standing dismiss ruling. This is tracked in
    vergil-project/vergil-claude-plugin#852.
  - The state-directory confirmation asked the human to vouch for files
    they cannot see. The driver had already checked the hashes, so the
    question added nothing.
- **Grandfathering beat retrofitting.** Choosing notes ("predates the spec
  and plan convention" and so on) over reconstructing documents for old
  epics kept the backfill mechanical. It meant no epic needed a human
  decision.

## 3. Compromises and tradeoffs

- **Security findings rejected, risk accepted.** Across the three tooling
  tasks the review summaries count 43 security-tagged findings, some of them
  repeats across rounds. Of these, 4 were fixed on the branches and 30 were
  rejected; the rest were ended by earlier rulings. Each rejected finding
  needs epic-body edit rights or repo-admin rights in the org to exploit,
  and the human accepted that risk.
- **Known gaps declined without follow-up:**
  - No `vrg-epic-docs` mode repairs a malformed section or clears an entry
    back to a note. The human's reasoning: the data set is known and has no
    malformed sections.
  - `vrg-epic-audit --close` takes its closed-epic snapshot before it closes
    drift. An epic whose last open child the same run closes could be
    wrongly reopened. This bug was already on `develop`, found by the review
    of #3898, and declined with no follow-up.
  - Minor parser edges declined on #3896–#3898: list-item fences,
    re-read-and-compare before a write, and a 1,000-entry directory
    listing.
- **By-hand edits outside the tool:** the five logical-minds-foundry
  `Related:` lines (§1).
- **The documentation review closed without a PR.** The sweep found the site
  docs already accurate. The plugin's skills index does not mention the
  section; that is an omission, not an error, and it was left as is.

## 4. New problems and opportunities

**Unaddressed-findings sweep.**

- **Tasks swept:** 14, every child except this retrospective.
- **With an agentic-review summary:** vergil-tooling#3896, #3897 and #3898,
  and vergil-claude-plugin#837. Every `Not fixed` entry ruled `file` names
  an issue that exists (#3896's R4.1, R4.2 and R4.4, carried to #3897).
  Every other entry is declined or rejected by a human ruling.
- **No gate records:**
  - the deployments D1–D8 (vergil-project/.github#832–#839), which are
    operational, with no PR;
  - the docs task #820, a docs PR reviewed through the planning issue's
    design rounds;
  - the documentation review vergil-tooling#3895, closed without a PR.
- **PR reviews:** none of the five PRs has a review or conversation comment.
- **Planning issue vergil-project/.github#815:** spec pushback round 1
  accepted all 6 findings, and alignment round 1 accepted all 13.
- **Result:** no untracked problems were found.

**Surfaced during the epic:**

- The stray-write sweep and state-directory confirmation friction (§2):
  vergil-project/vergil-claude-plugin#852, open. The
  verify-hashes-and-ask-only-on-mismatch idea is logged here and not yet
  acted on.
- `migrate-repo` and other vestigial tools: vergil-project/.github#831, in
  planning.
- `vrg-epic-audit` needs a repo working directory to pick the App identity,
  even with `--repo`; run from elsewhere, it refuses. This is logged, not
  acted on.
- CI on vergil-claude-plugin PR #912 failed once on a truncated GitHub
  releases API response in `vrg-docs-stage`. The agent identity cannot
  re-run CI ("Resource not accessible by integration"), so the human
  re-ran it. This is logged, not acted on.
- Task numbers collided: vergil-project/.github#837 (the vergils-nemesis
  backfill) and vergil-project/vergil-claude-plugin#837 (the skills task).
  It was harmless, because refs were always fully qualified.

## 5. What's next

- vergil-project/.github#831: deprecate `migrate-repo` and review the
  `vrg-*` tools and skills for other vestigial ones.
- vergil-project/vergil-claude-plugin#852: make the stray-write sweep
  workable when sessions run in parallel.
- From now on, new epics get the section from `epic-create`, and
  `vrg-epic-audit` flags any epic without one.

## Appendix A — Operational notes

The deployment order: D1 (vergil-project), then D2–D7, then D8 once the
plugin release was installed. Each followed the same steps:

1. **Private-repo check.** No org had a private repo, so only each
   `.github` was swept.
2. **Dry run**, condensed for review.
3. **`--apply`**, then a confirming dry run that reported "no change".
4. **Audit:** `vrg-epic-audit --repo <org>/.github`. It prints the "Epic
   documents" section only when there is a finding.

| Task | Org | Added | Changed | Notes |
|---|---|---|---|---|
| D1 #832 | vergil-project | 79 | 0 | 20 legacy lines removed |
| D2 #833 | logical-minds-foundry | 61 | 0 | 5 `Related:` lines restored by hand |
| D3 #834 | mq-rest-admin-project | 6 | 0 | |
| D4 #835 | mnemosys-project | 7 | 0 | |
| D5 #836 | diogenes-project | 1 | 0 | |
| D6 #837 | vergils-nemesis | 1 | 0 | |
| D7 #838 | wphillipmoore | — | — | 12 repos, no epics |
| D8 #839 | vergil-project | 1 | 2 | #881 new; #816 and #848 closed as not planned |

None of the 156 epics needed a human decision.
