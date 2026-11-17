# Size the planning queue — retrospective

- **Epic:** vergil-project/.github#1191
- **Planning issue:** vergil-project/.github#1185
- **Spec:** [`spec.md`](spec.md) · **Plan:** [`plan.md`](plan.md)

## 0. At a glance

We set out to give every planning-queue item an estimated epic size
(`epic-size:S|M|L|XL`) and an estimated task count, so the queue can be
sorted and filtered by size as well as age, and the small end worked on a
regular cadence. That shipped as planned. `vrg-intake-route` records a size
(a `size` route, and sizing on the planning route), a one-off clears it, and
`vrg-queue planning` shows, sorts and filters by it, with exit 2 for an item
whose sizing is inconsistent. `epic-review` and `epic-create` use it. The
backfill sized 55 planning items. The epic ran in one day: the planning
issue opened at 16:15Z on 2026-11-16 and the backfill finished at 22:58Z.

| Task | PR | Merged (UTC) | What it did |
| --- | --- | --- | --- |
| .github#1192 | .github#1200 | 11-16 18:19 | Spec and plan |
| vergil-tooling#4486 | vergil-tooling#4492 | 11-16 18:43 | `lib/epic_size.py` (scale, comment format, size record) and the four labels |
| vergil-tooling#4487 | vergil-tooling#4510 | 11-16 19:09 | `vrg-intake-route <ref> size`; optional sizing on the planning route |
| vergil-tooling#4488 | vergil-tooling#4511 | 11-16 19:14 | `vrg-queue` size column, `--sort`, `--size`, `--json` fields, exit 2 |
| vergil-tooling#4489 | vergil-tooling#4519 | 11-16 20:00 | A one-off clears the size and posts the `Size cleared …` marker |
| vergil-claude-plugin#1105 | vergil-claude-plugin#1109 | 11-16 20:11 | `epic-review` sizes items; `epic-review` and `epic-create` read exit 2 |
| vergil-tooling#4477 | vergil-tooling#4523 | 11-16 20:31 | Documentation review: issue standards, CLI reference, guides |
| vergil-claude-plugin#1112 | vergil-claude-plugin#1114 | 11-16 20:31 | Sweep task: one-off clearing in `epic-review` and the skills page |
| vergil-project/docs#120 | vergil-project/docs#121 | 11-16 20:32 | Sweep task: sizing on the org docs management pages |
| vergil-tooling#4536 | vergil-tooling#4537 | 11-16 22:51 | Follow-up: unknown size label; bare `vrg-queue` needs trusted authors |
| vergil-project/docs#122 | vergil-project/docs#123 | 11-16 22:51 | Follow-up: sizing is optional; `--size` keeps items in error |
| .github#1196 | — (deployment) | closed 11-16 | Releases attested and installed; route identity confirmed trusted |
| .github#1197 | — (deployment) | closed 11-16 | Backfill: 55 items sized, 0 failed, 0 mismatches |
| .github#1193 | this PR | — | Retrospective |

- **Repos touched:** 4 (.github, vergil-tooling, vergil-claude-plugin,
  docs).
- **Tasks:** 14: 1 spec and plan, 5 code, 2 deployment, the documentation
  review and its 2 per-repo sweep tasks, 2 follow-ups, and this
  retrospective.
- **PRs:** 11 merged, plus this one.
- **Releases:** vergil-tooling v2.1.319, whose notes list #4492, #4510,
  #4511 and #4519, and vergil-claude-plugin v2.1.74, whose notes list #1109
  (checked in #1196). vergil-claude-plugin#1114 and vergil-tooling#4537
  merged after those releases; they are documentation only and ship in the
  next ones.
- **Code size:** the five code PRs come to +2,515/−114 lines across 22 files.

## 1. How the plan evolved

The plan's task list held: every planned task shipped in the planned order,
and no task was dropped or reshaped. The changes were all at the edges, and
they are logged in the plan's `Evolution during execution` section, which
this PR appends.

- **The documentation sweep spread past its home repo.** The review found
  gaps in two repos it could not edit: the plugin never said a one-off clears
  the size, and the org docs site's management pages knew nothing of sizing.
  It filed vergil-claude-plugin#1112 and docs#120, as the bookend convention
  says.
- **Reviewers' below-the-bar notes became two follow-ups.** The merge
  reviews of #4477 and docs#120 each noted two small inaccuracies, below
  their bar for a finding. The branches were frozen, so the human ruled to
  file them as vergil-tooling#4536 and docs#122, one per repo. Both landed
  before the retrospective.
- **Tasks 3 and 5 ran in parallel, as planned,** and so did Tasks 4 and 6.
  The plan's dependency table made every frontier clear. Tasks 3 and 5 both
  edited the same two docs pages, in different sections, and merged without
  conflict.
- **The backfill surfaced a queue-placement error.** One sizing reason said
  vergil-tooling#4108 had been moved to ad hoc on 2026-11-08, though it still
  sat under the planning epic. The human ruled to route it as a one-off,
  which also exercised #4489's size clearing live.

The plan was strong going in. Three design-review rounds on #1185 raised 18
findings and accepted all 18. Their fixes were the work that later needed
no rework: the deferred view asking for `labels` (P1.1), a test fake that
returns only the fields asked for, and the backfill passing each field as
its own argument (P1.6).

## 2. Lessons learned

- **A narrow, well-reviewed plan runs in an afternoon.** From planning issue
  to finished backfill took under seven hours, which is the cadence the epic
  exists to enable for small items. The plan named every function, test and
  check, and the five code tasks each passed their merge review with no
  findings. A small delta between plan and outcome is the signal that the
  review rounds were worth their time.
- **The doc sweep should expect to cross repos.** The plan's doc review was a
  vergil-tooling bookend, but the epic changed user-facing behaviour
  described in three repos. Next time, the plan can name the doc pages in
  each repo up front, so the sweep confirms them instead of discovering them.
- **A below-the-bar note on a frozen branch costs a whole task.** Four small
  wording gaps each needed a ruling, an issue, a run and a PR. That is
  cheap, but it suggests asking reviewers to state such omissions as
  findings in documentation-only changes, where a fix is a one-line edit
  before handoff.
- **Keep the Evolution log as you go.** The log for this epic was written at
  retrospective time, from the session record. On a longer epic it would
  have been lost.
- **A human-attested precondition needs the comment, not just the word.**
  Task 7's precondition asked for the release versions in a comment on the
  issue. The human reported the releases in session, and the agent posted
  the attestation for them once they confirmed the versions. The deployment
  run then recorded the host-side install as human-reported: from inside the
  VM, only the VM's install could be checked directly.

## 3. Compromises and tradeoffs

- **Two rules stricter than the spec.** Task 2 added
  two rules the spec did not ask for: an unknown size label such as
  `epic-size:XXL` is an error, and a sizing comment repeating a field is
  malformed. Both are stricter and hide nothing, so they shipped. The spec
  was not amended; the docs now describe the first.
- **`vrg-queue` usage errors moved from exit 2 to exit 1.** Exit 2 now means
  "every queue read, but an item's sizing is inconsistent." Any outside
  script reading 2 as bad arguments would misread it. The plan's review found
  no such caller, and the two skills that call `vrg-queue planning` were
  updated in Task 6.
- **Fifteen planning items are still unsized.** The backfill applied only the
  2026-11-16 estimates: items in progress, ready to close or newer than the
  sizing pass were not estimated. They show as `?` and are left for the next
  `epic-review planning` pass, which sizes waiting items.
- **The actual task count is not recorded against the estimate.** Out of
  scope by design: it can be derived later from each planning issue's
  `Epic:` marker and that epic's sub-issues.

## 4. New problems and opportunities

**Unaddressed-findings sweep.** 13 tasks swept (every child but this one),
plus the epic issue, the planning issue's three design-review rounds and the
11 merged PRs (none had a review or a PR comment). All comments were by
trusted authors. Every `Awaiting a ruling` item has a closing ruling: N0.1–N0.4
on the epic were filed as vergil-tooling#4536 and docs#122, and N0.1–N0.3 on
.github#1197 were routed or declined. Every design-review finding was accepted, and
every discarded point carries its reason. **None found untracked.**

Surfaced during execution:

- **vergil-tooling#4108 was in the wrong queue.** Routed to vergil-tooling's
  ad-hoc epic on the human's ruling (see #1197).
- **vergil-claude-plugin#1037 and #1040** are single small changes held in
  planning only because the paad fork has no implementation path yet. Left
  in planning on the human's ruling.
- **Standalone `markdownlint` disagrees with `vrg-validate`.** Two runs
  (vergil-claude-plugin#1105 and docs#120) reported that
  `vrg-container-run -- markdownlint <file>` flagged line-length errors,
  including on untouched lines, that the full `vrg-validate` passes. It looks
  as if the standalone run does not get the per-path config `vrg-validate`
  gives each file, and files such as the plugin's `skills/*.md` are not
  linted by `vrg-validate` at all (vergil-project/.github#480). Filed to
  intake on the human's ruling as vergil-project/.github#1225.
- **The 15 unsized planning items** are for the next `epic-review planning`
  pass (listed in #1197's result comment).

## 5. What's next

- Start the small-end cadence: `vrg-queue planning --sort size --size S`
  lists the small items oldest first. The goal is to take the oldest and run
  `epic-create` and `epic-implement` on it within a few hours.
- Run `/vergil:epic-review planning` to size the 15 remaining `?` items.
- No follow-on brainstorm was raised.
