# checkpointed-epic-create — retrospective

- **Epic:** vergil-project/.github#1546
- **Spec:** [`spec.md`](spec.md)
- **Plan:** [`plan.md`](plan.md), with its `## Evolution during execution` log
- **Planning issue:** vergil-project/.github#1493

**Span** (how long the epic took):

| Window | Start | End | Elapsed |
| --- | --- | --- | --- |
| Whole epic | Mon 23 Nov 2026, 18:28 UTC | Tue 24 Nov 2026, 20:12 UTC | 1 d 1 h 43 min |
| Planning | Sun 22 Nov 2026, 16:52 UTC | Mon 23 Nov 2026, 22:59 UTC | 1 d 6 h 6 min |
| Implementation | Mon 23 Nov 2026, 22:59 UTC | Tue 24 Nov 2026, 20:12 UTC | 21 h 12 min |

The planning window opens when the planning item was filed. The
`epic-create` run itself started on Mon 23 Nov: the item was sized at
14:20 UTC and the brainstorm was approved at 18:28 UTC. Implementation
starts with the spec-and-plan PR (vergil-project/.github#1574), the
first task PR merged. The first code PR, vergil-project/vergil-tooling#5089,
merged at 23:27 UTC.

## 0. At a glance

We set out to make `epic-create` a sequence of checkpointed stages. At
each checkpoint a session can be cleared and the run resumed from the
planning issue and the docs worktree alone, told by a tool which stage
to enter. That shipped. `vrg-checkpoint` (vergil-tooling) keeps a strictly
parsed stage checklist in the planning issue's body, and `epic-create`
(vergil-claude-plugin) runs as nine stages with a tick at each
checkpoint. Task filing is keyed, so a run can resume mid-filing, and
skew is reconciled with the human rather than refused. A bootstrapped
skeleton can be abandoned cleanly. A live planning run with seven
resumes validated the main path. Using the skill on further planning
runs then turned up five more fixes, all folded into the epic and shipped.

| PR | Task | What it did |
| --- | --- | --- |
| vergil-project/.github#1574 | #1547 | Published the spec and plan |
| vergil-project/vergil-tooling#5089 | #5085 (2a) | Added `vrg-checkpoint` with `init`, `status`, `tick` and `reset` |
| vergil-project/vergil-tooling#5090 | #5086 (2b) | Added `vrg-checkpoint reconcile` |
| vergil-project/.github#1585 | #1584 | Recorded the first plan change: 3a–3c as three PRs, not one batch |
| vergil-project/vergil-claude-plugin#1319 | #1307 (3a) | Restructured `epic-create` into nine checkpointed stages |
| vergil-project/vergil-claude-plugin#1320 | #1308 (3b) | Keyed task filing, skew reconciliation, adoption of runs in flight |
| vergil-project/vergil-claude-plugin#1321 | #1309 (3c) | `epic-review planning` abandons a bootstrapped skeleton before routing |
| vergil-project/vergil-claude-plugin#1325 | #1293 (doc review) | Brought the plugin's README, site docs and skill text in line |
| vergil-project/docs#176 | docs#175 | Described the checkpointed stages in the planning workflow pages |
| vergil-project/vergil-tooling#5116 | #5115 | Described the nine stages in the GitHub issue standards |
| vergil-project/docs#178 | docs#177 | Fixed the bookend-convention and intake pages |
| vergil-project/vergil-claude-plugin#1347 | #1341–#1344 | Four fixes from live use, one commit each |
| vergil-project/vergil-claude-plugin#1348 | #1345 | Gave the documentation-review bookend its convention title |
| vergil-project/.github#1620 | #1619 | Recorded the remaining plan changes |

- **Repos touched:** 4 (`.github`, `vergil-tooling`,
  `vergil-claude-plugin`, `docs`).
- **Tasks:** 21 children: 20 closed before this retrospective, plus this
  one. 17 were PR tasks and 3 operational (deployments
  vergil-project/.github#1571 and #1572, validation #1573).
- **PRs:** 14 merged (#1347 closed four tasks).
- **Releases:** vergil-tooling v2.1.340 (`vrg-checkpoint`) and v2.1.342
  (issue-standards docs); vergil-claude-plugin v2.1.89 (the restructure),
  v2.1.90 (doc review) and v2.1.91 (live-use fixes).
- **Planned against delivered:** the plan had 8 tasks, 2 bookends and
  the spec task, 11 in all. 21 were filed. The extra 10 are two plan
  evolution records, three per-repo doc tasks from the sweep, and five
  live-use fixes. The plan's one plugin batch PR became three PRs.

## 1. How the plan evolved

The design held. Every change recorded in the plan's Evolution log is
about how the work was delivered or how far the epic reached. None
changed the mechanism.

**The batch broke on a rule the plan did not check.** The plan had 3a–3c
land as one batch PR, so that no plugin release could carry part of the
change. But 3a picked up the `security` label from a design-review
finding, and a security-labelled issue is never batched. The human kept
the batch's goal without the batch: 3a landed alone, 3b and 3c followed,
and the plugin release was held until all three had merged. It worked,
but the guarantee now rested on the human holding the release, not on
the shape of the PR.

**The documentation sweep fanned out as planned.** It filed a task per
repo (vergil-tooling#5115, docs#175), and the docs#175 run found two more
stale passages, filed as docs#177. This is the sweep doing its job, not
a surprise, although the plan named only the files and not the tasks.

**The validation was narrowed, then accepted with one gap.** The plan's
Task 5 asked for five resume scenarios. The live run on planning issue
vergil-project/.github#1544 exercised every stage with seven resumes,
and a throwaway item, #1587, proved the skeleton abandon. The human
dropped the three scenarios that fire only on an interrupted `file-tasks`,
an adopted pre-checklist run or a stage-list change, and deferred them
until they happen in real use. The run then failed one criterion: a
"not a security run" answer did not survive a `/clear`. The human
accepted the validation and had the fix filed instead of a re-run.

**The epic stayed open to absorb live use.** Rather than closing after
validation, the human kept the epic open while `epic-create` planned
other epics, and folded the fixes in (#1341–#1345). The last entry draws
the boundary: later findings go to the ad-hoc epics. One of these
fixes, #1343, narrows a rule the spec states (§4.8: the human confirms
every match of an unkeyed task). The spec text was left as written, and the
change is recorded in the Evolution log.

## 2. Lessons learned

- **The sizing was far off.** The planning item was sized S, about 5
  tasks, at low confidence. The plan had 11 and the epic filed 21. Most
  of the growth came after the plan (sweep tasks, live-use fixes, plan
  records), but the plan itself was twice the estimate. The elapsed time,
  about a day, was in line with S. Task count measured the overhead of the
  workflow more than the size of the work.
- **A human answer with no durable record is lost at a clear.** The only
  validation failure came from this: an ordinary run posted no marker,
  so a "no" could not be written down. The design reviews checked every
  stage tick and round comment, but not answers given in passing. Next
  time, list every question the skill asks the human, and give each
  answer, a "no" included, a durable record.
- **Every defect merge review caught was a re-entry defect.** Four
  defects were found and fixed before merge:
  - `docs-pr` re-entry skipped the writes after report-ready (#1307).
  - A multi-fixture plan entry shared one key (#1308).
  - A deferred `Blocked-by:` ref lived only in session memory (#1342).
  - A wrong spec section was cited (#1619).
  The first three all asked the same question: what if the session ends
  between these two writes? For a checkpointed skill, that question
  belongs in the plan's Review Focus, per write, not only per stage.
- **Composing with upstream skills is a dependency.** superpowers 7.0.x
  changed how its brainstorming skill ends, and a run followed the new
  ending instead of taking the checkpoint (#1344). The fix names which
  parts of the ending apply, and adds a note to review that step whenever
  the upstream skill changes.
- **A structural change in round 1 drives the later rounds.** Spec
  pushback round 1's `bootstrap` stage (S1.1) was right. But 8 of round
  2's 10 findings were repairs to it, and `reset`, which plan pushback
  added to support it, produced more repairs in plan pushback round 2
  and alignment round 1. Six design-review rounds raised 44 findings.
  Expect a cascade after a structural acceptance, and budget a round
  for it.
- **A real planning run is a good validation, and it informed a
  decision.** Validating on #1544, the next real item, tested the skill
  on real work and not a staged script. It also settled #1544's own
  question: its epic, #1596, concluded that `epic-implement`, which runs
  unattended, does not need human checkpoints, and was closed without
  being built.
- **Check batching rules while planning.** A batch can be refused (here,
  a security-labelled task cannot be batched), so the plan should check
  that before it depends on one.

## 3. Compromises & tradeoffs

- **Three resume paths are unproven live.** Partial filing, adoption
  after `reset`, and skew reconciled by `vrg-checkpoint reconcile` have
  never been exercised end to end. `reconcile` has unit tests. Merge
  review could not check the adoption section's claims about how the
  pre-checklist skill ordered its steps (#1308). The human accepted
  this: each failure (a duplicate or unkeyed task on one epic) is
  visible and can be fixed by hand.
- **The validation was accepted with a failed criterion.** The fix,
  #1341, shipped in plugin v2.1.91 and passed merge review, but no
  live run has tested it. The failure it fixes went the conservative
  way (a run treated as a security run), and nothing leaked.
- **The release hold was a human discipline.** Once the batch was split,
  nothing mechanical stopped a plugin release between 3a and 3c. During
  that window, the restructured skill stopped for the human on any skew
  or malformed block.
- **A body edit and a tick can race.** `tick` reads the body and then
  PATCHes it, so a concurrent web-UI edit can lose an update. This was
  accepted in plan pushback, with the same window `vrg-epic-docs` already
  has. The worst outcome is redoing a stage that creates nothing.
- **Deployment rested on attestation.** For 4b, the agent checked the
  install only on its own VM. The host and the other VMs were recorded
  on the human's word.
- **The spec text lags #1343.** The spec and plan still say the human
  confirms every match. The narrowing is recorded only in the Evolution
  log.
- **Below-bar review notes were left as they were.** Merge reviews of
  #5085 and #5086 noted small edge cases that did not meet the bar. The
  §4 sweep rules on each of them.
- **Security.** Spec pushback round 2 raised 2 security findings: 1
  accepted (GHSA-5wq3-h45q-gcjc, carried by vergil-claude-plugin#1307)
  and 1 rejected (GHSA-jqq5-mx78-fpx6).

## 4. New problems & opportunities

What the epic surfaced, and where each went:

- **Five fixes from live use:** filed under this epic and shipped in
  vergil-claude-plugin v2.1.91:
  - vergil-project/vergil-claude-plugin#1341: record the security-run
    answer.
  - #1342: add `Blocked-by:` refs when keying an existing task.
  - #1343: key a bookend with a convention title without asking.
  - #1344: say which parts of brainstorming's ending apply.
  - #1345: state the documentation-review bookend's title.
- **Stale docs outside the sweep's first pass:** vergil-project/docs#177,
  filed and fixed.
- **`vrg-release`'s skew refusal:** vergil-project/.github#1545,
  captured during the brainstorm. It is in the intake queue and has not
  been acted on.
- **Checkpoints for `epic-implement`:** vergil-project/.github#1544 was
  planned as epic #1596, an experiment. That epic decided against human
  checkpoints for `epic-implement` and was closed without being built
  (#1615 removed its spec and plan).

**The unaddressed-findings sweep.** It read 20 tasks (every child except
this one), their 14 merged PRs, the epic's own comments and the planning
issue's six design-review rounds. Every item was read through the trust
layer, and no untrusted content was skipped.

- Every `Awaiting a ruling` item has a closing ruling: five on the epic,
  one on #1573. Each `file` ruling's issue exists.
- #1573's one `Filed without a ruling` entry was `not filed`, but its fix
  has since been filed as #1341.
- Every design-review finding was accepted, or dropped with its reason
  in the round comment. The one rejected finding is the security finding
  above, which has its record.
- Two noted items were already dropped by the human on #1573: the three
  deferred resume scenarios (in the scope change), and an agent that
  went past a checkpoint pause (attributed to the human's own
  instruction to accept defaults).

Untracked problems found: 3, each ruled on by the human.

- **U1: filed** as vergil-project/.github#1621. The reference doc and
  the code disagree on a failed write. On a failed body write,
  `vrg-checkpoint` says the edit "may not have been applied", while
  `cli-tools-overview.md` says it "may have been applied". It is still so
  on vergil-tooling `develop` (869ab9a). Noted below the bar in #5085's
  merge review.
- **U2: dropped.** The marker scan in `lib/checkpoint.py` does not skip
  fenced code blocks, unlike `epic_docs` (#5085 review). The human's
  reason: it needs a trusted author to paste a block inside a fence, and
  a pasted block is then a duplicate, which is malformed and stops
  loudly, so it never fails silently.
- **U3: dropped.** Two cosmetic `reconcile` edge cases from #5086's
  review. The change lines go to stderr, not stdout, when the write
  fails. A removed stage recorded twice gets two "dropped" lines. The
  human's reason: the first still prints every line, and the second
  needs a malformed block written by hand.

## 5. What's next

- vergil-project/.github#1545 (reconsider `vrg-release`'s resume skew
  refusal) waits in the intake queue.
- The three deferred resume scenarios are handled when they first occur.
  A failure there is a fix task on the ad-hoc epic.
- Later findings in `epic-create` go to vergil-claude-plugin's ad-hoc
  epic, as the plan's last Evolution entry says.
- No follow-on epic: the `epic-implement` follow-on (#1544, then #1596)
  was settled as not needed.

## Appendix A — Operational notes

The release and deploy order the plan's tooling-before-plugin rule
required:

1. vergil-tooling #5089 and #5090 merged (23 Nov, 23:27 and 23:42 UTC).
   v2.1.340 was released and its install stage ran on the host and every
   VM. Deployment #1571 recorded it (24 Nov, 00:21 UTC).
2. The plugin restructure, now unblocked, landed as three PRs: #1319
   alone (01:18 UTC), then #1320 and #1321 (01:36 UTC).
3. vergil-claude-plugin v2.1.89 was published (01:43 UTC) and installed by
   the human (`vrg-host plugins`, `vrg-vm update --all`). Deployment #1572
   recorded it (11:15 UTC).
4. The documentation sweep merged (#1325, docs#176, #5116, docs#178), and
   was released as plugin v2.1.90 and tooling v2.1.342. The validation's
   planning sessions loaded v2.1.90.
5. The validation, #1573, ran on #1544 (13:29 to 15:10 UTC) and on the
   throwaway #1587, and was accepted at 15:35 UTC.
6. The live-use fixes (#1347, #1348) were released as plugin v2.1.91.

## Appendix B — Extended metrics

Design review on planning issue vergil-project/.github#1493:

| Round | Raised | Accepted | New / repair | Recommendation |
| --- | --- | --- | --- | --- |
| Spec pushback 1 | 6 | 6 | 6 / 0 | continue |
| Spec pushback 2 | 10 | 9 | 2 / 8 | stop |
| Plan pushback 1 | 6 | 6 | 6 / 0 | continue |
| Plan pushback 2 | 6 | 6 | 3 / 3 | continue |
| Alignment 1 | 11 | 11 | 5 / 6 | continue |
| Alignment 2 | 5 | 5 | 2 / 3 | continue |
| **Total** | **44** | **43** | **24 / 20** | |

The human stopped plan pushback and alignment after round 2 against the
"continue" recommendation. The spec review approved the spec with no
edits.

Merge review: 16 PR tasks have a merge-review record. The spec-and-plan
task, #1547, has none on its issue. 12 passed on the first review. Four
had one defect each, which was fixed and passed its check. There were no
security findings and no accepted risk.
