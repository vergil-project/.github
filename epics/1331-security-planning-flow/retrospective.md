# Harden the security-planning flow — retrospective

- **Epic:** vergil-project/.github#1331
- **Planning issue:** vergil-project/vergil-claude-plugin#828
- **Spec:** [`spec.md`](spec.md) · **Plan:** [`plan.md`](plan.md)
- **Retrospective task:** vergil-project/.github#1333
- **Span:** epic opened 2026-11-18T16:23Z; the last task before this
  retrospective closed 2026-11-19T13:44Z, about 21.4 hours later.

This is a security epic. Following `epic-create`'s public-wording rule, it
describes mechanisms only, and refers to findings by issue ref and GHSA ID
alone.

## 0. At a glance

**What we set out to do.** Make the flow that plans a `security` planning
issue safe from end to end. Where a check can be mechanical, the tools
refuse the unsafe move; the skill text says what to do when they do:

- `vrg-security-record retarget` moves an advisory only from an issue it
  tracks, and never silently stops tracking an open issue;
- the security-records sync never closes an advisory because a queue item,
  or an issue closed other than as completed, is all that ended it;
- the planning hold is one rule that fails closed, in `vrg-queue planning`
  and in the docs-PR close;
- `epic-create` detects a security run at step 0, states the public-wording
  rule there, gates the first publication on the human, and hands every
  advisory on with the new retarget contract;
- advisories closed before the sync guard existed are reviewed once, and
  the human decides each one.

**What shipped.** All of it:

- `retarget` requires `--from` and refuses, before any write, to drop an
  open tracked issue that `--drop` does not name. `create` makes the first
  pairing on its own path, and `vrg-intake-route` keeps every other tracked
  issue.
- The sync's close guard keeps such an advisory `draft` and reports it on a
  `held:` line, without failing the pass.
- One pure `advisory_hold` function decides the hold for the planning view
  and the docs-PR close, and holds when an advisory cannot be read.
- `epic-create`, `epic-review`, `epic-retrospective` and the review
  doctrine carry the step 0 check, the wording rule, the gate, the
  not-planned stop, the hold text and the `--from` hand-offs.
- The one-time review of closed advisories is done.

**Work delivered** (PR — what it did — merged, UTC):

- **Spec and plan:** .github #1363 — the spec and plan — 2026-11-18 21:31.
- **Planned tasks:**
  - Task 6, vergil-claude-plugin #1181 — step 0 security-run check and
    marker, wording rule, human gate, not-planned stop and hold text —
    2026-11-18 21:58.
  - Task 2, vergil-tooling #4724 — `retarget --from`, the open-issue drop
    check, the first-pairing path and the intake-route callers —
    2026-11-18 22:05.
  - Task 3, vergil-tooling #4725 — the sync close guard and its `held:`
    report — 2026-11-18 22:27.
  - Task 4, vergil-tooling #4726 — one fail-closed planning hold rule
    (vergil-tooling#3996, routed in from the deferred queue) —
    2026-11-18 22:49.
  - Task 8, docs #142 — the intake page names retarget's `--from` —
    2026-11-19 12:18.
  - Task 7, vergil-claude-plugin #1189 — the `--from` hand-offs and the
    cannot-be-read hold note in the skills — 2026-11-19 12:32.
- **Documentation review:**
  - vergil-tooling #4749 — the security-records guide and the GitHub Issues
    standard's planning hold — 2026-11-19 13:44.
  - docs #144 (spawned task docs#143) — three management pages aligned with
    the sync pass and the planning hold — 2026-11-19 13:44.
- **Deployment tasks** (no PR): .github#1360 (vergil-tooling release,
  closed 2026-11-19 12:02) and .github#1361 (plugin release, closed
  2026-11-19 13:27).

**Releases** (published, UTC):

- vergil-tooling v2.1.327 (#4724, #4725, #4726) — 2026-11-18 23:17. It
  also carried epic vergil-project/.github#1339's #4727–#4729.
- vergil-claude-plugin v2.1.80 (#1181) — 2026-11-19 01:03. Cut for epic
  #1339 (#1182); Task 6 rode along.
- vergil-claude-plugin v2.1.81 (#1189) — 2026-11-19 12:45. Task 9 attested
  this release; it contains both #1181 and #1189.
- vergil-tooling #4749 is merged but in no release as of 2026-11-19 (the
  latest is v2.1.327). The `docs` repo has no GitHub releases; when its
  site picked up #142 and #144 is unknown.

**Counts:**

- **Repos touched:** 4: .github, vergil-tooling, vergil-claude-plugin and
  docs.
- **Tasks:** 12 children, all closed once this retrospective merges:
  - 3 vergil-tooling code tasks;
  - 2 skill-text tasks;
  - 1 docs task;
  - 2 deployment tasks;
  - the documentation review and the docs task it spawned;
  - the spec and plan task;
  - this retrospective.
- **PRs:** 9. **Releases:** 3 (one of them shared with epic #1339).
- **Estimate against actual:** planning sized it "S, about 8 tasks"
  (confidence medium). The plan filed 8 tasks besides the bookends, and the
  documentation review added one.
- **Design review:** 5 rounds on #828 (spec pushback 2, plan pushback 2,
  alignment 1), each stopped by the human. 43 findings: 41 accepted, 2
  rejected with a reason. Of those, 15 were security-class: 14 accepted, 1
  rejected. With the 4 advisories the planning issue already carried, 18
  advisories were handed on to their fix tasks at filing.

## 1. How the plan evolved

The task list held. Every task in the Delivery table ran, in the order it
set: Tasks 2, 3 and 6 started at once; Task 4 followed Task 2; Tasks 7
and 8 followed the tooling deployment; Task 9 followed Task 7; the
documentation review came last. Four entries went into the plan's
evolution log: three from execution, below, and a fourth from this
retrospective, recording its sweep's follow-ups (§4).

- **The deployment tasks lost their install steps.** Tasks 5 and 9 each
  had a step to update the host's install and re-check it. The human ruled
  them out: a human-attested release already installs everywhere, so the
  planning had scheduled redundant work. Each task's result records step 2
  as covered by the attestation. The planning-side fix is
  vergil-project/.github#1370.
- **Spec §6.3's audit report is narrower than written.** Task 4 found that
  `vrg-epic-audit` reports an unlabelled `Security advisory:` line only
  when some advisory tracks the issue; the queue note covers the untracked
  case. The hold is unaffected. The human ruled to ship it as built and
  file the gap (N0.1 on the epic): vergil-project/.github#1369.
- **The documentation review spawned a `docs` task.** #4697 found three
  management pages in `docs` out of date, one of them saying the sweep runs
  the security-records pass. It filed vergil-project/docs#143 under the
  epic, as the plan's per-repo rule asks.

Inside the tasks, the deviations were small and each is recorded on its
issue:

- **Task 6** fixed one merge-review defect before handoff: step 10's
  fallback said the queue shows `held:` at handoff, when it shows it only
  once the docs task has closed.
- **Task 3** fixed one more at merge: the GitHub Issues standards page still
  gave the old sync close rule. pr-watch's merge review caught it after the
  rebase.
- **Task 7** stopped at merge review on one scope item (R1): the skills
  index paragraph it edited also described Task 6's behaviour. The human
  ruled keep. That closed one documentation-review item early.

## 2. Lessons learned

- **"Independent" tasks that share files still conflict.** Tasks 2, 3 and 4
  were independent in behaviour, and plan pushback round 2 discarded the
  shared-guide concern as "ordinary merge sequencing". That held: nothing
  shipped broken. But Tasks 2 and 3 were built in parallel on the same
  library, guide and reference files. #4725 conflicted once #4724 merged,
  and #4726 once #4725 merged, and pr-watch rebased each. The real
  conflicts were one CLI-reference row and one guide paragraph. Next time,
  when parallel tasks edit the same lib, guide or reference file, plan for
  the rebases: give the shared docs edit to one task, or serialize the
  merges.
- **A discard resting on a claim about code needs its case named.**
  Alignment round 1 discarded the §6.3 audit-report concern because "the
  existing `classify` already reports the resulting broken pair". That is
  true only when an advisory tracks the issue. Building Task 4 exposed the
  other case (#1369). A discard that cites existing behaviour should say
  which cases it checked.
- **A deployment task should take the release as given.** Epic #1339, run
  in parallel, reached the same lesson. A human-attested release of
  vergil-tooling or the plugin already installs on the host and every VM.
  A deployment step that reinstalls and re-checks it adds nothing, and
  re-checking release notes needs a tag-ancestry check, since the GitHub
  release bodies are only a docs link. #1370 asks planning to know what a
  release covers.
- **Attestation comments need to be easy to post.** Each deployment's
  precondition is a comment from the human's own login, naming the
  release. That caused friction. The human had to post it themselves and
  asked for paste-ready commands. A deployment task should give the human
  the exact command to paste.
- **The `--from` window opened at the release, not at Task 5.** The spec
  timed the window from the tooling deployment to the plugin deployment.
  The release installs everywhere, so the window began when v2.1.327 was
  published (2026-11-18 23:17), and closed when v2.1.81 was (2026-11-19
  12:45), about 13.5 hours later. Whether any retarget was refused in that
  window is not recorded (unknown).
- **Mechanical checks first, skill text second, worked.** Shipping the
  skill text that needed no new tool behaviour (Task 6) in parallel with
  the tooling took it off the critical path. The second skill part could
  then quote the released tool's exact wording, which its merge review
  checked against the source.
- **A plugin release does not reach sessions already running.** Sub-agents
  started after v2.1.81 still loaded 2.1.80 until the main session
  reloaded. The stale-plugin stop blocked the #1361 deployment twice, and
  the main session finished it itself.

## 3. Compromises and tradeoffs

- **`vrg-epic-audit` does not report every broken pair.** An unlabelled
  `Security advisory:` line on an issue no advisory tracks is reported only
  by the planning-queue note (#1369, open). The hold is correct either way.
  Only the audit's report is missing.
- **The deployment tasks ran without their install checks.** Steps 2 of
  #1360 and #1361 rely on the human's release attestation instead of a
  recorded check on the host. This is by ruling, and the release mechanism
  does the install, but the result comments carry no install evidence.
- **The skill text is still the guide for what the tools cannot check.**
  Public wording is judgment (a non-goal of the spec), so the step 0 rule
  and the human gate cover it, not a machine check.
- **Task 7's skills-index paragraph went beyond its scope.** It also
  describes Task 6's behaviour, kept by ruling because the paragraph needed
  it anyway and the text matched the skill.
- **The parallel start cost two rebases** (§2), accepted as cheaper than
  serializing three tooling tasks.

## 4. New problems and opportunities

**Unaddressed-findings sweep:**

- **Scope:** all 11 children other than this task, the epic issue, their 9
  merged PRs and the planning issue's round comments.
- **Records:**
  - The epic's one `Awaiting a ruling` item (N0.1) has a trusted
    `Human ruling: N0.1 file`, and its issue exists
    (vergil-project/.github#1369).
  - The merge-review items on the tasks are all closed: two fixed (#1178
    R1, #4719 R1) and one kept by ruling (#1179 R1).
  - Every documentation item recorded on #4697 is resolved, fixed there or
    handed to docs#143. #4749's notes list each one.
  - Every round finding on #828 was accepted, or rejected with a reason.
  - None of the 9 PRs has a review or a PR conversation comment.
- **Security findings:** none new from execution. The design review's are
  counted in §0.
- **Untracked problems found:** five, from the execution record and the
  task comments, each ruled on by the human. One was added to an existing
  issue, two were filed and two were dropped, as listed below.

**What the epic surfaced, and where each item went:**

- **`vrg-epic-audit` misses an unlabelled advisory line on an untracked
  issue.** Ruled "file" (N0.1): vergil-project/.github#1369 (intake, open).
- **Deployment tasks schedule installs a release already does.** Filed:
  vergil-project/.github#1370 (intake, open).
- **Sub-agents run a stale plugin after a release.** Found:
  vergil-project/.github#1381 (intake, open) makes the stale-plugin stop a
  warning, which is what blocked #1361 twice.
- **Attestation friction.** The human must post each deployment
  precondition from their own login, and asked for paste-ready commands.
  Found: added to vergil-project/.github#1370, which already asks to check
  `issue-deploy` and the deployment-task template, so that one change also
  gives the human the exact attestation command.
- **A deployment closed before its result posted.** The #1360 run posted
  its "Deployed — see the SUCCESS result above" close before the result
  comment. The skill rule against this already existed
  (vergil-claude-plugin#1115, closed 2026-11-17, before this epic), so the
  rule alone did not hold. Cosmetic here. Filed:
  vergil-project/.github#1387 (intake), to make the close refuse until the
  result comment exists.
- **`vrg-finalize-pr` reported a conflict from a stale mergeable status.**
  A few seconds after pr-watch pushed the #4725 rebase, finalize read
  GitHub's not-yet-recomputed status and reported "conflict". Filed:
  vergil-project/.github#1388 (intake), to wait for the status to settle.
- **A sub-agent's background wait loop never ran,** because it matched its
  own command line. Dropped by the human: it was an ad hoc shell loop in
  one run, not a fleet tool, and pr-watch already waits with
  `vrg-pr-await`.
- **The epic body still says "the scheduled sync".** Spec pushback round 2
  (S2.5) corrected the spec and plan: no schedule runs the sync. The epic
  body was not updated. Dropped by the human: this retrospective records
  the correction, and the body closes with the epic.
- **A laptop crash interrupted the main session.** No action. The issue
  comments were enough to resume.

## 5. What's next

- **The audit's broken-pair report:** vergil-project/.github#1369, in
  intake.
- **What a release covers, for planning and `issue-deploy`:**
  vergil-project/.github#1370, in intake, now also covering the attestation
  command (§4).
- **An operational task's close after its result:**
  vergil-project/.github#1387, in intake.
- **`vrg-finalize-pr` and a settling mergeable status:**
  vergil-project/.github#1388, in intake.
- **The stale-plugin stop as a warning:** vergil-project/.github#1381, in
  intake.
- **Planning item vergil-project/.github#593** (open, "Review
  identity-mode resolution for the identity-gated tools") was waiting on
  #828, per #828's 2026-11-07 note. With this epic closed, it can be
  planned.

## Appendix A — Operational notes

- **Order:** the spec and plan merged, then Tasks 2, 3 and 6 ran in
  parallel. Task 6 merged first (#1181, 21:58). Then #4724, #4725 (rebased
  by pr-watch) and #4726 (rebased by pr-watch) merged within 45 minutes of
  each other.
- **The tooling release:** v2.1.327 at 23:17. The human attested it on
  #1360 at 00:29 on 2026-11-19. #1360 closed at 12:02 that day; the
  session was interrupted in between, including by a laptop crash.
- **The one-time review** in #1360 ran after the release and is done.
- **Tasks 8 and 7** merged at 12:18 and 12:32. The plugin release v2.1.81
  followed at 12:45. The human attested it on #1361 at 12:56. The main
  session closed #1361 at 13:27, after sub-agents on the stale plugin were
  blocked twice.
- **The documentation review** filed docs#143 at 13:30. Both #4749 and
  #144 merged at 13:44.
- **Release notes:** both the vergil-tooling and the plugin GitHub release
  bodies are only a docs link. The deployments checked the tag's ancestry
  and the `CHANGELOG.md` at the tag instead.
