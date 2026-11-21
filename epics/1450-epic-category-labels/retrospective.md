# Category labels for every epic — retrospective

- **Epic:** vergil-project/.github#1450
- **Planning issue:** vergil-project/.github#1447
- **Spec:** [`spec.md`](spec.md) · **Plan:** [`plan.md`](plan.md)
- **Retrospective task:** vergil-project/.github#1452
- **Span:** the epic opened at 2026-11-20T21:27Z. Execution began when the
  spec and plan merged, at 2026-11-20T22:59Z. The last task before this
  retrospective closed at 2026-11-21T11:54Z. That is about 14.5 hours from
  opening and about 13 hours of execution.

## 0. At a glance

**What we set out to do.** Give every epic exactly one category label, so
one click in the GitHub UI shows one category, and the development epics
stand apart from the many ad-hoc, proxy and management epics. Two labels
are new, `epic-development` and `epic-management`; `ad-hoc` and `proxy`
already served. The labels are for display only: no tool reads the new
ones to decide anything.

**What shipped.** All of it:

- **At creation.** `vrg-epic-create` derives the category label from the
  epic's other labels and applies it. It refuses a caller-given
  `epic-development` or `epic-management`, and labels that mark two
  categories. `ensure_standing_epic` adds `epic-management` when it creates
  an intake, planning, deferred or hoshin epic, and still finds and reuses
  one without it.
- **`vrg-ensure-label --label <name>`** creates one label from its registry
  definition, and refuses a name that is not registered.
- **The one-off**, across 8 epic homes in the six orgs:
  - 14 closed legacy `Epic (standing):` umbrellas lost `epic`, so they are
    no longer epics;
  - both labels were created in every home (16 creates);
  - 208 existing epics, open and closed, were labelled: 197
    `epic-development` and 11 `epic-management`.
  - No epic was ambiguous, and no issue changed state.
- **Docs** in vergil-tooling, the docs site and the `epic-create` skill.

**Work delivered.**

| PR | Task | What it did |
| --- | --- | --- |
| `.github#1461` | #1451 | Spec and plan |
| `vergil-tooling#4864` | #4861 | Category function, labels at creation |
| `vergil-tooling#4865` | #4862 | `vrg-ensure-label --label` from registry |
| `vergil-tooling#4871` | #4843 | Docs review: "Epic categories" |
| `vergil-tooling#4872` | #4870 | `--help` for the registry form |
| `docs#153` | #152 | Category labels in `epics-and-tasks.md` |
| `vergil-claude-plugin#1235` | #1234 | `epic-create` names the label |

All PRs are in `vergil-project`; each task is in its PR's repo.

- **Repos touched:** 4: `.github`, `vergil-tooling`, `docs`,
  `vergil-claude-plugin`.
- **Tasks:** 10 including this one: 7 planned, 3 added during execution.
- **PRs:** 7 merged, plus this retrospective's.
- **Deployment tasks:** 2 (#1459, #1460), both SUCCESS.
- **Release:** vergil-tooling `v2.1.332` carries #4864 and #4865. #4872
  merged after it and is not yet in a release.

## 1. How the plan evolved

The plan held. The two tooling tasks ran in parallel as planned and
merged together, and the deployment chain ran in order. Every deviation
was a correction at the edges, recorded in the plan's
[Evolution during execution](plan.md#evolution-during-execution) log:

- **The release check asked for something Task 2b never required.** Task
  3's step 4 expected `vrg-ensure-label --help` to describe the new form,
  but Task 2b's steps updated only the docstring and the reference page.
  The check failed on the help text alone; the behaviour was installed.
  The run filed vergil-tooling#4870 for the help text, and the human ruled
  the installed code sufficient for the check, so the one-off did not wait
  for another release.
- **The sweep spawned two doc tasks, not one.** The plan foresaw
  docs#152 (`epics-and-tasks.md`). The sweep also found that the
  `epic-create` skill said new epics get only `epic`, and filed
  vergil-claude-plugin#1234.
- **The legacy list was a snapshot.** The plan named 13 `Epic (standing):`
  umbrellas; the live listing found 14, the extra being
  mq-rest-admin-project/.github#22. The human ruled the rule governs, not
  the list.

## 2. Lessons learned

- **A check that tests an artifact must be backed by a step that produces
  it.** Task 3's `--help` check had no matching step in Task 2b, and the
  plan's alignment round did not catch the gap. It cost a FAILURE, a fix
  task and a ruling. Next time: when a deployment check reads an output
  (help text, a file, a label), trace it to the task step that writes it.
- **Dated lists in a plan are evidence, not the contract.** The alignment
  round had already corrected the umbrella list once (8 to 13) and it was
  still one short. The procedure's rule, applied live, was right both
  times. Write the rule as the contract and the list as "as of" a date.
- **Read-only first, then apply, worked well for a fleet-wide write.** The
  one-off ran as a read-only planning pass, then an apply run that re-listed
  everything and would stop on any difference from the approved list. The
  human saw all 238 changes (14 removals, 16 label creates, 208 adds)
  before any happened, and the apply run found no difference.
- **The batching guideline excluded two small CLI tasks.** 2a and 2b were
  independent and in one repo, and ran as two PRs only because each
  changed a CLI. The human judged the exclusion too broad; see §4.

## 3. Compromises and tradeoffs

- **The release check passed on installed-code evidence.** Check 4 was
  accepted under a ruling, not as written, so the `--help` gap was live in
  `v2.1.332`. The fix (#4872) is merged but not yet released; it ships with
  the next vergil-tooling release.
- **Two closed `[archived]`-titled issues got `epic-development`.**
  vergil-project/.github#683 and #443 carry `epic` but neither the
  `archive` label nor a stamped title, so the archive rule does not see
  them. The human ruled closed, old-convention issues out of scope, and the
  procedure applied as written.
- **No drift repair, by design.** Nothing checks the labels after the
  one-off (spec §2.2). An epic created by old tooling, or relabelled by
  hand, stays wrong until someone notices.

## 4. New problems and opportunities

- **Narrow the batching never-batch rules.** Filed as intake
  vergil-project/.github#1462 on the human's request, with 2a and 2b as the
  example: batching exists to share one CI run, and the exclusions should
  target size and risk, not the CLI category.
- **Reopening closed development epics.** Raised in planning (S2.2) and
  already filed as intake vergil-project/vergil-tooling#4844; out of this
  epic's scope.
- **The `--help` gap** was filed and fixed in the epic itself
  (vergil-tooling#4870, PR #4872).

**Unaddressed-findings sweep.** Swept 9 tasks (all but this one), the epic
issue and the planning issue #1447, with the trusted-author list.

- Every `Awaiting a ruling` item has a closing ruling: #1459 N0.1 accept;
  #1460 N0.1 accept, N0.2 dismiss, N0.3 accept.
- The one `Filed without a ruling` item, vergil-tooling#4870, exists and
  is closed.
- Every planning finding not accepted has its reason in its round comment,
  and the one routed to intake (vergil-tooling#4844) exists.
- All six merge reviews ended `eligible; accepted-risk 0`. No merged PR has
  a review or a conversation comment.
- One candidate checked and found tracked: the two `Epic (deferred):
  vergil-project` epics (#531 open, #532 closed). #532 was already closed
  as a duplicate of #531.
- **Result: no untracked problems found.**

## 5. What's next

- The next vergil-tooling release carries #4872's help text; no task waits
  on it.
- The batching discussion starts from intake vergil-project/.github#1462.
