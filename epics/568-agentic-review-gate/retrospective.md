# Agentic Review in `issue-implement` — Retrospective

Epic: [vergil-project/.github#568](https://github.com/vergil-project/.github/issues/568) ·
Spec: [`spec.md`](spec.md) · Plan: [`plan.md`](plan.md)

## 0. At a glance

**What we set out to do.** `issue-implement` ran `vrg-validate` and then handed
the PR to the human, with no review of its own. This epic, child D of the paad
umbrella, set out to put a `paad:agentic-review` loop between the two:

- fresh review-only subagents review the branch, for up to three rounds;
- the implementer fixes what they find;
- every round is recorded on the task issue;
- everything not fixed is filed;
- the run pauses for the human when the rules say so;
- `epic-retrospective` sweeps for anything left untracked.

**What shipped.** The gate, as specified, is live:

- vergil-tooling **v2.1.277** added the managed `paad/code-reviews/` ignore
  line, the read-only `vrg-git symbolic-ref` form and the trusted-authors
  configuration.
- vergil-claude-plugin **v2.1.50** added doctrine §3, the gate in
  `issue-implement` and the sweep in `epic-retrospective`.

The dogfood run did not pass. A real task ran 5 rounds without converging, and
the human dropped it. That result is why the follow-on epic,
[vergil-project/.github#699](https://github.com/vergil-project/.github/issues/699)
(paad unattended mode), moves the fragile parts into paad itself. The gate stays
in use until then: it is advisory, and the human can always accept a branch at
a pause.

| Repo | PR | Task | What it did |
|---|---|---|---|
| .github | [#592](https://github.com/vergil-project/.github/pull/592) | #569 | Spec and plan |
| vergil-tooling | [#3780](https://github.com/vergil-project/vergil-tooling/pull/3780) | #3740 (T2) | The managed `.gitignore` ignores `paad/code-reviews/` |
| vergil-tooling | [#3777](https://github.com/vergil-project/vergil-tooling/pull/3777) | #3722 (T3) | `vrg-git` allows the read-only `symbolic-ref` form |
| vergil-tooling | [#3782](https://github.com/vergil-project/vergil-tooling/pull/3782) | #3741 (T4) | `[trusted_authors]` configuration and `vrg-trusted-authors` |
| vergil-tooling | [#3787](https://github.com/vergil-project/vergil-tooling/pull/3787) | #3774 *(added)* | The self-fence drift guard says how to resync from source |
| vergil-tooling | [#3788](https://github.com/vergil-project/vergil-tooling/pull/3788) | #3775 *(added)* | The CLI Tools Overview lists every console script |
| vergil-claude-plugin | [#803](https://github.com/vergil-project/vergil-claude-plugin/pull/803) | #759 (T5) | Doctrine §3 (mode 2) and the paad policy |
| vergil-claude-plugin | [#805](https://github.com/vergil-project/vergil-claude-plugin/pull/805) | #761 (T6) | The gate in `issue-implement`; `epic-implement`'s waiting list |
| vergil-claude-plugin | [#804](https://github.com/vergil-project/vergil-claude-plugin/pull/804) | #760 (T7) | The unaddressed-findings sweep in `epic-retrospective` |
| vergil-tooling | [#3808](https://github.com/vergil-project/vergil-tooling/pull/3808) | #3721 | Documentation review: the issues standard, and the security-records guide's agentic-review recipe |
| docs | [#74](https://github.com/vergil-project/docs/pull/74) | docs#73 *(spawned)* | The workflow and tools pages mention the gate |

- **Repos touched (4):** .github, vergil-tooling, vergil-claude-plugin, docs.
- **Children (17):**
  - 13 planned: T1–T10, and the three bookends (#569, #3721, #570).
  - 4 added during execution:
    - two tooling fixes found while deploying (#3774, #3775);
    - the mode 2 doctrine fix, vergil-claude-plugin#813 (dropped, no PR);
    - one doc task spawned by the docs review (docs#73).
  - 5 operational:
    - the re-spike .github#590 (T1, SUCCESS);
    - the tooling deployment vergil-tooling#3742 (T8, SUCCESS);
    - the plugin deployment vergil-claude-plugin#762 (T9, SUCCESS);
    - the dogfood run .github#591 (T10, FAILURE, closed by the human's ruling).
- **PRs merged:** 11, counting the spec/plan PR and leaving out this one.
- **Releases cut (by the human):**
  - vergil-tooling v2.1.277 (T2–T4, per #3742);
  - vergil-claude-plugin v2.1.50 (T5–T7, per #762).
  - Several other releases of both repos came out during the epic; this
    retrospective does not map them to tasks (unknown). #3808 and docs#74 are
    on `develop` and not yet in a release.
- **Span:** the epic opened 2026-10-27 14:08Z. The last task before this one
  (#3721) closed 2026-10-29 12:55Z, about 47 hours later.

## 1. How the plan evolved

The plan has no "Evolution during execution" log, so this section is rebuilt
from the issue record and the session.

- **The re-spike (T1) took three rounds to fix the prompt.**
  - Round 1: told only to return paad's answer unchanged, every reviewer
    reworded it, and one replaced the counts line with a table. The fix was to
    spell out an exact return format.
  - Round 2: every reviewer wrote scratch files outside its worktree, and none
    reported them. The fix was mode 1's "write nothing, anywhere" directive.
  - Round 3: this round added a stray-write sweep, which the human then added
    to doctrine §3.4.
- **The sweep needed tuning before it could ship.** On a shared VM, an
  unscoped sweep caught container-runtime files, tool caches and other
  sessions' scratchpads. The first version of §3.4 would have failed every
  round: it had no `.claude.json` exclusion and named the wrong
  container-runtime path. T5 was unfrozen, fixed and reported ready again.
- **Two tooling fixes joined during deployment.** T8's fleet sync turned up a
  misleading "already in sync" message (#3774) and a CLI overview that listed
  only part of the commands (#3775). Both were small, so the human had them
  done inside the epic rather than filed for later.
- **A CodeQL alert held up T4's PR.** A false positive on clear-text logging
  was cleared by printing a literal table name.
- **Order held otherwise:** T2–T4 in parallel, then T1, T5, T7, T8, T6 and T9,
  as planned.
- **The dogfood run became a doctrine fix, and then a drop.**
  - Mode 1's dogfood (epic 553, on .github#468) hit an ambiguity: reviewers
    echoed the "hold" routing answer as if it were the failure signal. The fix
    was vergil-claude-plugin#812.
  - Mode 2 has the same wording, so vergil-claude-plugin#813 was filed under
    this epic, and it became T10's real task.
  - It ran 5 rounds. Each round's fixes exposed a new edge case at the
    boundary with paad: keyword matches in file paths, a planted file or
    symlink under `paad/`, pauses with no comment, and resume paths.
  - The human dropped it once the follow-on epic to change paad itself was
    under way.
- **The docs review grew by one correction.** Besides a sentence about the
  gate, the docs review found that the security-records guide described an
  older agentic-review recipe that contradicted doctrine §3.6, and corrected
  it. The `docs` repo half was split out as docs#73.

## 2. Lessons learned

- **A reviewer subagent can't be made to report reliably from outside paad.**
  paad runs inside the reviewer's own context. Anything the gate reads from
  the reviewer's answer, whether prose, a keyword or a token, is under the
  reviewer's control, and two reviewers echoing a directive was enough to
  break a check. The only independent evidence is the filesystem:
  - what is in `paad/` before dispatch;
  - the structured fields of paad's report files after the review.

  The lasting fix is for paad to return a machine-readable result for its
  security routing. That is #699's job.
- **The spec's non-goal "changing paad" was the wrong boundary.** Working
  around paad from the outside produced a stack of doctrine rules, each
  patching the last. #813's five rounds were the cost of that boundary. Next
  time, when a workaround starts to grow its own edge cases, stop and ask
  whether the dependency itself should change.
- **Exact formats beat good intentions.** "Return the answer unchanged" was
  reworded every time; an exact, numbered return format held in every later
  run. The same goes for "write nothing": it held only once it said
  "anywhere, including temporary files and copies".
- **The stray-write sweep works, but it can't tell the reviewer's writes from
  the harness's own.** It flagged the harness's background-task output in
  every round of #813, and the main session's own notes in one. Each case
  cost a human ruling (#695).
- **Advisory is the right posture for now.** Because the human can rule past
  any pause, the gate added value on #3721 and docs#73 without blocking
  progress on #813. The real catches were a stale guide that contradicted the
  doctrine, and wording that overstated the doctrine.
- **Agentic review turns up real problems in docs-only changes.** The #3721
  review found a contradiction between the guide and the doctrine that the
  docs review was meant to catch and had not.

## 3. Compromises & tradeoffs

- **#813 was dropped rather than finished.** Mode 2's "hold" wording can
  still fail a valid round on develop. That fails safe: it costs a ruling and
  loses nothing. The fix is left to #699, which will replace the mechanism
  instead of patching it.
- **The dogfood validation was closed on a FAILURE.** The human ruled that
  T10's failures are input for #699, not fix tasks under this epic, so the
  epic could close. The failures:
  - the summary left out one intake issue and one reverted addition (now
    corrected by an addendum);
  - one public intake issue carried security-class detail.

  Review Focus 5 (a re-dispatch without a ruling) was never exercised.
- **One security record was made at the drop.** It covers a weakness that is
  on develop too, which #813 would have fixed. It stays open until #699 or a
  task fixes it.
- **The sweep exclusions are a list, not a design.** Each new source of noise
  on the VM needs a doctrine change. #695 tracks the latest one.
- **A docs-only change in #3721 went beyond the requirements.** The guide
  correction was flagged as an out-of-scope addition. It was kept, because
  reverting it would have left the contradiction, and the human merged it.
  One commit after the last reviewed HEAD (6ee47b43) was not reviewed.

## 4. New problems & opportunities

- **The follow-on epic:** [vergil-project/.github#699](https://github.com/vergil-project/.github/issues/699)
  (paad unattended mode). It holds the hand-off from T10, with the failures
  and lessons, and a question from the docs review about recording
  weaknesses that a branch happens to fix.
- **Intake filed during the epic, still open:**
  - From the re-spike: .github#667 (`vrg-queue` advisory hold), #668
    (`epic-create` steps 9–10), #669 (vergil-tooling `AGENTS.md`).
  - From #813: .github#695 (the sweep and harness output), #696 (a failed
    output check has no response comment to carry its `Paused:` line) and
    #697 (mode 1 should fail on a decorated token line).
  - From T8: .github#630 (`vrg-pr-await` resolves runs in the current
    directory's repo).
- **Security:** findings went through `vrg-security-record` only. The public
  record gives counts: the re-spike's findings were screened with the human
  and recorded or matched to existing records, and one new record was made at
  #813's drop.

**The unaddressed-findings sweep** covered 16 children (every child but this
one), read against the trusted-author lists.

- **Gate summaries:** three tasks have one: plugin#813 (dropped after 5
  rounds, plus an addendum), vergil-tooling#3721 (pass after 2) and docs#73
  (pass after 1). Every finding they list as not fixed names an issue that
  exists, or carries the reason it was not filed.
- **No gate records, operational:** .github#590 and #591, vergil-tooling#3742,
  and vergil-claude-plugin#762.
- **No gate records, merged before the gate shipped** (plugin v2.1.50,
  2026-10-28 17:54Z): .github#569, vergil-tooling#3722, #3740, #3741, #3774
  and #3775, and vergil-claude-plugin#759, #760 and #761.
- **PR reviews:** the only non-approving review is a code-scanning bot's on
  vergil-tooling#3782, and the CodeQL fix resolved it.
- **The planning issue, vergil-claude-plugin#677:** every finding not
  accepted has its reason or ref.
- **Untracked candidates and their outcomes:**
  - .github#590: 4 ordinary-class findings held for the human to screen.
    **Found:** they were screened with the human in the security walkthrough
    and handled as security records (counts only).
  - .github#590: out-of-scope additions in already-merged code, deliberately
    not filed. **Dropped (human):** the re-spike reviewed old, merged code as
    test input, not as work to change.
  - .github#590: the security files left for the human. **Found:** reviewed
    with the human and recorded or matched (counts only). The worktrees are
    removed.
  - .github#591: the public intake issue with security-class detail.
    **Found:** the weakness existed only on the dropped branch, and the
    missing screen is part of the hand-off to #699.
  - vergil-claude-plugin#813: R5.6, an out-of-scope addition left pending the
    human. **Dropped (human):** moot, as the branch was dropped.
  - vergil-tooling#3721: R2.3, the guide correction, to be confirmed at the
    PR. **Found:** the human merged PR #3808.
  - Planning issue: one security finding deferred to the system-wide planning
    issue without a ref. **Found:** it matches an existing record (counts
    only).

## 5. What's next

- [vergil-project/.github#699](https://github.com/vergil-project/.github/issues/699):
  paad unattended mode. It extends paad so that its review skills report
  structured results to an unattended caller, which retires most of doctrine
  §3's workarounds. It is in planning now.
- Until then, the gate runs as documented, and the human rules past any false
  pause.
