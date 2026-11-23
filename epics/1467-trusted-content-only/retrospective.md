# trusted-content-only — retrospective

- **Epic:** vergil-project/.github#1467
- **Spec:** [`spec.md`](spec.md) · **Plan:** [`plan.md`](plan.md), with its
  [Evolution during execution](plan.md#evolution-during-execution) log
- **Planning issue:** vergil-project/.github#579
- **Retrospective task:** vergil-project/.github#1469

**Span:**

| Window | Start | End | Elapsed |
| --- | --- | --- | --- |
| Whole epic | Sat 21 Nov 2026, 21:35 UTC | Mon 23 Nov 2026, 21:54 UTC | 2 d 0 h 18 min |
| Planning | Tue 27 Oct 2026, 14:32 UTC | Sun 22 Nov 2026, 16:22 UTC | 26 d 1 h 50 min |
| Implementation | Sun 22 Nov 2026, 16:44 UTC | Mon 23 Nov 2026, 21:54 UTC | 1 d 5 h 9 min |

The planning window runs from the security record that became the planning
issue (opened 2026-10-27) to the spec-and-plan PR's merge. Most of its 26 days
was queue time: planning started on 2026-11-21. The implementation window runs
from the first merged PR after the spec-and-plan PR (vergil-actions#1354) to
the last one (vergil-containers#1018).

## 0. At a glance

**What we set out to do:** make one rule hold everywhere in Vergil's
automation: act only on GitHub content written by a trusted identity (the
human's account, the human's agent Apps, or an org's own automation App, as
the human's Vergil configuration names them). Every behaviour-driving read
would go through one trust layer in vergil-tooling. Untrusted content would
stop the item it is load-bearing for and otherwise be reported, never dropped
silently. A guard test and an agent hook would keep new raw reads out, and the
morning review would show outsider activity.

**What shipped:** all of it, across vergil-tooling v2.1.336 to v2.1.338,
vergil-actions v2.1.89 and v2.1.90, and vergil-claude-plugin v2.1.86.
`lib/trust.py` is the one place a trust decision is made. It judges authors
and every editor of bodies, comments and reviews, close and reopen actors,
same-repo PRs for branch lookups, and account types for each role. The
configuration gained a per-org `[orgs.<org>]` stanza with the org's automation
App, and four orgs are listed. `vrg-trusted-authors publish`, `check` and
`render` carry it to Actions as org variables, and the daily config audit
checks them as the automation App. Every planned site moved onto the layer,
and so did 9 more reads that the guard test found. The guard test's
`[[pending]]` table is empty on `develop`. `vrg-issue-read` is the agents'
trusted read command, with a trusted search and list mode, and every skill
that reads issue or PR content uses it. The hook guard refuses agents' raw
content reads and names `vrg-issue-read`. `vrg-epic-audit --untrusted` is the
outsider report, and `hoshin start` runs it per org. The vergil-containers
lifecycle job is off until vergil-project/.github#1128. Task 22 validated the
layer on real data in all four orgs. The outsider report works, but it is slow,
and on vergil-project the search cap limits it to about three days
(vergil-project/vergil-tooling#5057).

| | |
| --- | --- |
| **Tasks** | 71 closed, plus this retrospective: 34 implementation tasks and 3 bookends filed at planning, 18 fixture-run tasks (the plan's Task 19), 13 added during execution (3 plan records, 10 follow-up tasks) and 4 per-repo documentation tasks from the review |
| **PRs** | 47 merged, plus this retrospective's. 4 of them are batch PRs covering 11 tasks |
| **Repos touched** | 12: `.github`, `docs`, vergil-tooling, vergil-actions, vergil-claude-plugin, vergil-containers and the six language fixtures. mq-rest-admin-project has a proxy epic with no tasks |
| **Releases** | 14: vergil-tooling v2.1.336, v2.1.337 and v2.1.338; vergil-actions v2.1.89, v2.1.90 and v2.1.93; vergil-claude-plugin v2.1.86; vergil-containers v2.1.48; fixtures python-app v0.1.5, python-lib v0.1.6, rust-app v0.1.6, rust-lib v0.1.7, ts-app v0.1.8, ts-lib v0.1.8 |
| **Size** | +30,286 / −5,116 lines, 25,812 of the additions in vergil-tooling |

### Work delivered

vergil-tooling (in plan order; the release that carries each PR is in the last
column):

| Task | Issue | PR | What it did | Release |
| --- | --- | --- | --- | --- |
| 3a | #4911 | #4951 | Vergil org list and per-org automation identities in the trusted-authors configuration | v2.1.336 |
| 3b | #4912 | #4955 | The trust layer for issue bodies and comments | v2.1.336 |
| 3c | #4913 | #4960 | Trusted events, branch PR lookups and reviews in the layer | v2.1.336 |
| 3d | #4914 | #4981 | The release identity rules moved onto the layer, with their own parameters | v2.1.336 |
| 4 | #4915 | #4956 | `vrg-trusted-authors publish`, `check` and `render` for the Actions copies | v2.1.336 |
| 6a | #4916 | #4982 | The release adopts only same-repo trusted PRs | v2.1.336 |
| 6b, 6c, 9, 10 | #4917, #4918, #4926, #4927 | #4972 (batch) | PR lookups, CI evidence, the release gates and PR provenance onto the layer | v2.1.336 |
| 7a, 7b, 8c | #4919, #4920, #4924 | #4961 (batch) | Queue markers, outcomes and security-record lines read only from trusted content | v2.1.336 |
| 7c | #4921 | #4985 | The remaining comment and body readers moved onto the layer; old login-only helpers removed | v2.1.336 |
| 8a, 11 | #4922, #4928 | #4973 (batch) | Parents, blockers, rollups, orphan attach and routing trust only trusted authors | v2.1.336 |
| 8b | #4923 | #4983 | The sweep nativises parents and closes drift only from trusted content | v2.1.336 |
| 8d | #4925 | #4979 | Trusted authors required for task checks, release markers, linkage and relay payloads | v2.1.336 |
| 13 | #4929 | #4968 | `vrg-issue-read`, with its trusted search and list mode | v2.1.336 |
| 14 | #4930 | #4980 | The outsider report, `vrg-epic-audit --untrusted` | v2.1.336 |
| 15 | #4931 | #5004 | The guard test: fails on content reads outside the layer | v2.1.336 |
| 16 | #4932 | #5041 | The hook refuses agents' raw issue and PR content reads | v2.1.338 |
| added | #4966 | #4969 | Queue views stop only the item whose comments cannot be read | v2.1.336 |
| added | #4967 | #4975 | `vrg-resolve-tracking-issue` reports an API fallback failure as exit 2 | v2.1.336 |
| added | #4970 | #4976 | The activity log leaves out outsider-opened and outsider-closed issues, with a batched trusted read | v2.1.336 |
| added | #4971 | #4977 | Task views and the epic audit stop only the child whose body is untrusted | v2.1.336 |
| added | #4978 | #4984 | The outsider report lists every untrusted edit of reviews and inline review comments | v2.1.336 |
| added | #4986 | #5013 | Release notes and rebuild read PRs through the layer | v2.1.337 |
| added | #4987 | #5014 | Finalize and intake routing read closing references through the layer | v2.1.337 |
| added | #4988 | #5015 | The batch check and in-flight marks judge PR bodies through the layer | v2.1.337 |
| added | #4989 | #5016 | Epic docs and the planning hold read bodies through the layer | v2.1.337 |
| added | #4990 | #5017 | Security records read issue bodies only through the layer | v2.1.337 |
| doc review | #4887 | #5066 | The docs brought in line with the epic | not yet released |

All refs in this table are in vergil-project/vergil-tooling.

The other repos:

| Task | Issue | PR | What it did | Release |
| --- | --- | --- | --- | --- |
| 1 | .github#1468 | .github#1491 | Spec and plan | — |
| plan record | .github#1505 | .github#1506 | First execution changes in the plan | — |
| plan record | .github#1522 | .github#1523 | Second execution changes in the plan | — |
| plan record | .github#1528 | .github#1529 | Third execution changes in the plan | — |
| 5a | vergil-actions#1351 | vergil-actions#1359 | The trusted authors passed into every job that reads content | v2.1.90 |
| 5b | vergil-actions#1352 | vergil-actions#1354 | Only same-repo release heads get the full CI matrix | v2.1.89 |
| 5c | vergil-actions#1353 | vergil-actions#1360 | The daily config audit checks the trusted-authors variables | v2.1.90 |
| 12 | vergil-containers#983 | vergil-containers#984 | The language lifecycle job disabled until its re-architecture | v2.1.48 |
| 17a | vergil-claude-plugin#1241 | vergil-claude-plugin#1281 | Work skills read task bodies and comments through `vrg-issue-read` | v2.1.86 |
| 17b, 17c | vergil-claude-plugin#1242, #1243 | vergil-claude-plugin#1282 (batch) | Review and planning skills read through `vrg-issue-read`; `hoshin start` shows the outsider report and the configuration audit | v2.1.86 |
| doc review | docs#170 | docs#172 | Management docs describe the trust rule, the outsider report and the trusted-authors check | — |
| doc review | vergil-actions#1386 | vergil-actions#1391 | Docs name the publish command, the setup order and the variable's consumers | v2.1.93 |
| doc review | vergil-claude-plugin#1296 | vergil-claude-plugin#1300 | The doctrine and skill docs point to `vrg-issue-read` | not yet released |
| doc review | vergil-containers#1012 | vergil-containers#1018 | Docs say the language-lifecycle job is off | not yet released |
| 19 | six change tasks | python-app#62, python-lib#69, rust-app#77, rust-lib#83, ts-app#80, ts-lib#79 | One small feature per fixture, plus a one-release lock rollback for the dependency update to redo | the six fixture releases |

Operational tasks, with no PR: Task 2's inventory (.github#1486), Task 18's
first tooling release (.github#1487), Task 20's plugin release (.github#1488),
Task 21's second tooling release (.github#1489), Task 22's real-data
validation (.github#1490), and the fixture runs' release and validate tasks.
Every one closed with `Outcome: SUCCESS`.

## 1. How the plan evolved

The plan held in outline: the configuration and the layer, then the sites by
group with the outsider report alongside, then the guard test, the skills and
the hook, then the releases and validation. The three plan records
(vergil-project/.github#1505, #1522 and #1528) log what moved, and Task 18's
result adds two changes the records never received.

**It started early, by ruling.** The spec made implementation wait for epic
vergil-project/.github#1440 to close. Once #1440's code was released and
validated live (vergil-project/.github#1455 and #1457), the human ruled that
this epic could start while #1440's bookends were still open. Task 3d alone
also waited for #1440's fixture-release validation
(vergil-project/.github#1496), since a fix found there could have changed the
rules 3d moved. 3d's merge review then raised nothing. The question of who
locks a release tracking issue came up at the same time. It was answered live
(the lock and the close were both made by the human's own `User` account), so
the identity handling did not change. The question stays a separate
follow-up, vergil-project/vergil-tooling#5006. The plan names it
vergil-project/.github#1503, which no longer resolves; it appears to have
been transferred.

**Task 2's inventory widened the read command.** The inventory found no
behaviour-driving read outside vergil-tooling but the lifecycle publisher
(Task 12) and the fork release-head choice (Task 5b, already merged), so it
filed no task. It did find skills that act on the results of a search or a
list, not only on one issue. By ruling, `vrg-issue-read` gained a trusted
search and list mode, Tasks 17a and 17b used it, and the hook also refuses
the raw search and list forms. Task 16's merge review later found the scope
did not say which forms count. The human ruled one rule for both: a list or
search made only of qualifiers, requesting no content field, is allowed;
anything that matches text or requests content is refused.

**Task 12 kept the plan's switch.** actionlint rejects a constant `if: false`.
The human ruled a targeted ignore of that one rule for `ops.yml`, naming
vergil-project/.github#1128 as the work that removes both, rather than
removing the job.

**Execution split, batched and filed as it went.** Task 11 needed Task 3c's
close-event reader, which the delivery table had missed. It then split: the
orphan-attach, routing and sizing stops stayed, and the activity log moved to
vergil-project/vergil-tooling#4970, because per-item trust reads across a
30-day window of closed issues would exhaust the docs deploy's token. Task
5c's check runs only in each org's `.github` repo, since one run covers the
whole org. Four batch PRs ran on the human's rulings: 7a/7b/8c, 6b/6c/9/10 and
8a/11 in vergil-tooling, and 17b/17c in the plugin, which the first plan
record had already allowed. Merge reviews and rulings filed five follow-up
tasks under the epic (vergil-project/vergil-tooling#4966, #4967, #4970, #4971
and #4978).

**The guard test found what the site list missed.** Task 15's test flagged 13
behaviour-driving reads outside the plan's sites. Four were fixed in Task 15,
and nine were split into five tasks
(vergil-project/vergil-tooling#4986 to #4990), each with its own advisory.
The test landed with a `[[pending]]` table naming each open site and the
task that removes it, and the human accepted that design (accepted risk 1). The five tasks blocked the
documentation review rather than the first release, because those reads were
already in released tooling. All five merged within about three hours and
shipped in v2.1.337. The `[[pending]]` table is now empty. This was the
point of the structural approach (spec §4, approach A): the gap list was
checked, not asserted.

**The release order deadlocked once.** The plan's Task 18 order was: tag
vergil-actions, cut and install vergil-tooling, then publish the org
variables. But the first trust-layer release's own CI evidence gate runs the
tooling being released, and that needs `VRG_TRUSTED_AUTHORS`. The human
published the variables first, with `uv run vrg-trusted-authors publish` from
the vergil-tooling `develop` clone, then resumed the release. The `Ops`
dispatch that Task 18 gave the agent turned out to be a human-run step, since
the agent identity may not dispatch workflows. Both changes are recorded on
vergil-project/.github#1487. The third plan record had merged before Task 18
found them, so they never reached the plan's log. This retrospective records
them (Appendix A).

**Scope moved out at the edges.** mq-rest-admin-project has no `Ops`
workflow, so its config audit never ran. The human ruled the gap out of this
epic, as mq-rest-admin-project/.github#55 in that org's ad-hoc epic. The proxy
mq-rest-admin-project/.github#54 was left with no tasks. In Task 22,
trusted PR reviews and agent PR content were ruled not applicable: the fleet
does not review PRs, and agent-written PR content arrives with epic
vergil-project/.github#1384. The scheduled sweep, refresh and config-audit
runs were replaced by dispatched runs of the same jobs, by ruling.

## 2. Lessons learned

Lessons marked *(session)* rest on the main session's account of the run,
not on the issue record. The rest are drawn from the record.

- **A structural check beats a site list.** The advisories' site inventory
  was written carefully, and the guard test still found 13 more reads. When
  the goal is "no gaps", plan the check before the site tasks, so that its
  findings arrive while the site tasks are still open.
- **Record procedure changes in the plan before the next task needs them.**
  Task 18's two procedure changes stayed on its issue. The documentation
  task was briefed from the plan's order. Its implementer wrote the stricter
  publish-first order, and merge review sent it back to the plan's
  install-then-publish order. In our judgment the docs are right for any org
  set up from now on, because installed tooling already carries `publish`.
  Still, an operational task's procedure change should go into the next plan record
  before any later task reads the plan.
- **A bootstrap release can depend on itself.** The first release of a
  command that its own CI needs cannot use the installed tooling. Plan the
  bootstrap explicitly: run the new command from a source checkout before
  the cut.
- **Check what the agent identity can do before filing its procedure.** The
  `Ops` dispatch and the live GraphQL checks needed the human, which the plan
  did not foresee. The human ran the live checks, recorded on
  vergil-project/vergil-tooling#4913, #4925, #4970, #4978 and #4987.
  *(session)* One early agent ran a live query through the tooling's own
  code rather than through
  the wrappers. Later briefs forbade that. Briefs should say up front which
  checks are human-run.
- **Tests must not inherit the runner's environment.** *(session)* Several
  tests passed locally and failed in CI because they did not clear
  `GITHUB_ACTIONS`, which switches the layer to counts-only output. Later briefs required it, and some
  modules gained autouse fixtures. A suite-wide fixture would remove the trap
  (§4).
- **A security word in a name can trip CodeQL.** *(session)* Function names containing
  "trusted" set off CodeQL's clear-text-logging rule. Renaming fixed each one.
  Expect it when naming security code.
- **Parallel PRs on one shared file cost rebases.** *(session)* The guard test's
  exemptions file and `trust.py` took repeated rebase conflicts as parallel
  PRs landed. Where many tasks add entries to one file, chaining them or
  giving each its own section would cut that cost.
- **The main session made four missteps.** *(session)* It batched security-labelled
  tasks without first checking the guideline against it. The human then set
  the experimental guideline aside, so the batches stood. It asked the human
  to clone a repo that was already cloned, because it missed a dot-directory.
  It over-prepared a trivial plugin release. And it passed along a "post a
  review" validation step that meant nothing in a fleet with no PR reviews.
  The lesson: check the guidelines and the filesystem before acting or asking,
  size the preparation to the task, and test each relayed step against how
  the fleet actually works.
- **Planning estimates for security work run low.** The planning issue sized
  the epic at about 22 tasks. Planning filed 34 implementation tasks and three
  bookends, and execution added 13 more and 18 fixture-run tasks.

## 3. Compromises & tradeoffs

- **The hook is a blocklist, not a boundary.** By the human's design-review
  ruling, the agent hook refuses the obvious raw read forms and accepts what
  it cannot see. Agents are not the adversary. An allowlist, if ever needed,
  is a separate follow-up.
- **The guard test shipped with known gaps listed.** The `[[pending]]` table
  let the test pass while nine reads were still open, and the first release
  went out with them, by ruling (accepted risk 1). The reads already existed
  in released tooling. All nine closed in v2.1.337, the next release.
- **The lifecycle job is off.** New toolchain versions are noticed later,
  until vergil-project/.github#1128 re-architects the job and turns it back
  on. The actionlint ignore stays until then.
- **The config-audit token warns on every run.** The `trusted-authors-check`
  job's token step passes Variables permission inputs that
  `actions/create-github-app-token` does not declare. The inputs take effect,
  and the token is scoped, but the runner warns. No supported way to ask for
  them without the warning exists upstream yet. The fix is deferred until
  upstream supports it, with the related `app-id` deprecation, as
  vergil-project/vergil-actions#1369.
- **The outsider report reaches only about three days on vergil-project.**
  GitHub's 1,000-result search cap and per-item reads limit it, and it is
  slow. The refusal is loud, by design. Making it fast enough for the daily
  review is vergil-project/vergil-tooling#5057, in intake.
- **One org is listed but not audited.** mq-rest-admin-project is in the
  configuration, but it has no `Ops` workflow, so its trusted-authors check
  never runs (mq-rest-admin-project/.github#55).
- **Parts of Task 22 were ruled rather than run.** Trusted reviews on real
  PRs and agent content on PRs were ruled not applicable. Reading trusted
  reviews rests on the unit tests, and agent PR content on epic
  vergil-project/.github#1384's fixture runs. Dispatched runs stood in for the
  scheduled ones.
- **Fixture ownership was granted by hand.** The human gave this epic the six
  fixtures over other epics' open fixture runs, and attested that each
  fixture's configuration was compliant. The general fix is
  vergil-project/.github#1502, which the human called urgent.
- **A known minor gap was left in a PR's notes.** A proxy with an untrusted
  body can be counted as a stop twice in one sweep run; the exit status is
  unaffected (vergil-project/vergil-tooling#4983's PR). See §4.
- **Task 18's procedure changes stayed off the plan** (§1). See §4.
- **The epic body was not updated after the early-start ruling.** Its
  "Order" section still says implementation waits for epic
  vergil-project/.github#1440 to close. The ruling is in the epic's comments,
  and #1440 is now closed.

## 4. New problems & opportunities

**Filed during the epic, by human ruling, and closed in it:**
vergil-project/vergil-tooling#4966, #4967, #4970, #4971, #4978, and #4986
to #4990.

**Tracked outside the epic:**

- vergil-project/vergil-tooling#5006 (open, ad hoc): whether a release should
  verify its tracking issue's lock, and whether lock events belong in the
  layer.
- vergil-project/vergil-tooling#5057 (open, intake): a faster outsider report.
- vergil-project/vergil-actions#1369 (deferred): the token permission-input
  warning and the `app-id` deprecation.
- vergil-project/.github#1128 (open, planning): the lifecycle job's
  re-architecture.
- mq-rest-admin-project/.github#55 (open, that org's ad-hoc epic): its `Ops`
  workflow.
- vergil-project/.github#1502 (open, planning): shared fixture ownership.
- vergil-project/.github#1538 (open, intake): release records report trusted
  comments they do not accept as "skipped untrusted content". The noise hides
  the skips that matter.

**Intake captures made during the epic:**

- vergil-project/.github#1524: always write issue references fully qualified.
- vergil-project/vergil-claude-plugin#1267: label each subagent's panel row
  with its task. It appears to have been filed as vergil-project/.github#1526
  and transferred.
- vergil-project/.github#1567: a stale passage in the docs site's `ci.md`
  (raised from epic 1064's work).

**Unaddressed-findings sweep.** 71 tasks swept: every child but this
retrospective (the epic has no proxy tasks). Also swept: the epic's 8 trusted
comments, the planning issue's 8 design-review round comments (two of them
broken rounds) and its other comments, and the 47 merged PRs' conversations,
reviews and inline review comments. None of the PRs carries a review or a
conversation comment. Every `Awaiting a ruling` item on a task or the epic
has a closing ruling, except two proposals on Task 18
(vergil-project/.github#1487) and items A and C on Task 22
(vergil-project/.github#1490). The human then tracked A as
vergil-project/vergil-tooling#5057, and C and Task 18's token-warning
proposal on vergil-project/vergil-actions#1369. Task 18's other proposal
(the dispatch step and the plan record) is items 2 and 6 below. Every
`Filed without a ruling` list says none. The security items are covered by
`Human ruling: security:` counts lines and by merge-review rulings. Every design-review finding not accepted
has its reason: one rejected security finding and one finding dropped by the
human with a scope change (to vergil-project/.github#1128). `vrg-issue-read`
reported no skipped untrusted item on any issue or PR it read.

Problems noted but not tracked by a task or a ruling, and the human's
ruling on each (2026-11-23):

1. **Found:** the plan-review delegate's broken security section, which broke
   spec round 1 and plan round 2, is vergil-project/vergil-claude-plugin#1037
   (open).
2. **Found:** Task 18's and Task 22's `Ops` dispatches being human-run steps.
   vergil-project/.github#497 (open) covers deployment and validation steps
   that must be runnable by the identity that runs them, and
   vergil-project/vergil-claude-plugin#835 (open) covers accepting a
   dispatched run in place of the nightly.
3. **Found:** the fixture-run claim problem is vergil-project/.github#1502.
   The `fixture-run` change-task scaffold's readiness command is now refused
   by the hook. The human reports that an existing issue tracks it; the sweep
   could not locate its ref.
4. **Filed:** a suite-wide test fixture that clears `GITHUB_ACTIONS` and the
   delivered-configuration variables, so a test cannot pass locally and fail
   on a runner: vergil-project/vergil-tooling#5087 (intake).
5. **Dropped:** the epic sweep can count a proxy with an untrusted body as a
   stop twice in one run (noted on vergil-project/vergil-tooling#4983's PR).
   The exit status is right either way.
6. **Dropped:** Task 18's publish-before-cut order was never put in the plan,
   and the vergil-actions docs give install-then-publish. The deadlock needed
   a release that both introduced `publish` and depended on it, which cannot
   recur. Appendix A records the bootstrap.
7. **Dropped:** the plan's stale ref for the lock question
   (vergil-project/.github#1503, now vergil-project/vergil-tooling#5006),
   since §1 records the current ref.
8. **Dropped:** the epic body's out-of-date "Order" section (§3), since
   #1440 is closed.
9. **Dropped:** CodeQL false positives on names containing "trusted", since
   renames fixed them and §2 records the workaround.
10. **Closed:** the empty proxy mq-rest-admin-project/.github#54, as not
    planned, by the human's ruling. Its only task moved to that repository's
    ad-hoc epic.
11. **Security-class:** 1 item, presented to the human in the terminal only.
    Human ruling: security: 1 findings: 0 fix, 1 record, 0 reject.

## 5. What's next

- **vergil-project/.github#892**, decisions posted by the human's own account,
  is the planned follow-on (spec §8). It is in the planning queue and builds
  on the layer's role parameters (spec §5.4).
- **vergil-project/.github#1384** brings agent-written PR content, which the
  layer's `agents` role must then pass on real PRs.
- **vergil-project/.github#1128** re-enables the lifecycle job through the
  layer.
- The open follow-ups in §4: vergil-project/vergil-tooling#5006 and #5057,
  vergil-project/vergil-actions#1369, vergil-project/.github#1502 and #1538,
  and mq-rest-admin-project/.github#55.
- The spec's proportion constraint still applies: the fleet's planned move to
  a self-hosted forge with no outside access will reshape this security
  architecture.

## Appendix A — Operational notes

**The release order that worked.**

1. The human added the four orgs' `[orgs.<org>]` stanzas to the host
   `identities.toml`, and granted each automation App read-only Variables
   permission at org and repository level.
2. vergil-actions v2.1.90 was tagged with Tasks 5a and 5c (5b was already in
   v2.1.89), and the floating `v2.1` tag moved.
3. The human ran `vrg-trusted-authors render`, then `publish --org <org>` for
   each org, from the vergil-tooling `develop` clone with `uv run`. This has
   to come before the first trust-layer cut, because that release's own CI
   evidence gate reads `VRG_TRUSTED_AUTHORS`.
4. vergil-tooling v2.1.336 was cut (after a re-run of the failed job and
   `vrg-release --resume 2.1.336`) and installed, then `vrg-vm update --all`.
5. The human dispatched `Ops` in each org's `.github`, and the agent checked
   the `trusted-authors-check` job and its values artifact. The agent identity
   cannot dispatch workflows.
6. The six fixture trios ran against v2.1.336 and v2.1.90.
7. vergil-claude-plugin v2.1.86 was released and installed before the hook.
   Then vergil-tooling v2.1.338 shipped the hook. In between, v2.1.337
   carried the five guard-test follow-ups, together with other epics' work.

**Gotchas met.**

- The first live GraphQL sighting of a `Bot` actor came only with
  vergil-project/vergil-tooling#4970's check. An agent App's login comes back
  bare, with no `[bot]` suffix.
- Editing an old issue comment moves the issue's `updatedAt`, so the outsider
  report's search finds it (vergil-project/vergil-tooling#4978).
- The reviews query's nested first pages exceeded GitHub's GraphQL node
  limit until merge review caught it (vergil-project/vergil-tooling#4913).
- Trust-layer errors at command boundaries surfaced as tracebacks in several
  commands until merge reviews caught them. Each command now maps them to a
  one-line error.

## Appendix B — Extended metrics

- **Design review (planning issue):** 6 completed rounds (2 spec pushback, 3
  plan pushback, 1 alignment), plus 2 broken rounds whose kept results were
  used by ruling. 65 findings raised: 63 accepted, 1 rejected and 1 dropped.
  22 of them were security findings (21 accepted), each recorded as an
  advisory.
- **Merge review (execution):** 74 review steps across the tasks, raising 21
  defects, 3 design questions, 1 pre-existing item and 3 security items. One
  security item was ruled `wont-fix` (accepted risk 1). Every other item was
  fixed, filed or ruled.
- **Security items in execution, outside merge review:** 15, from Task 2 (1),
  Task 14 (1) and Task 15 (13). Each was ruled with a counts line: 5 fixed,
  10 recorded and carried by the follow-up tasks above.
- **PRs by repo:** vergil-tooling 27, `.github` 4, vergil-actions 4,
  vergil-claude-plugin 3, vergil-containers 2, docs 1, each fixture 1.
