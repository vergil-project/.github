# Retrospective: iterating design review in epic-create

**Epic:** vergil-project/.github#553 · **Umbrella:** vergil-project/.github#500
(child C) · **Planning issue:** vergil-project/.github#469 ·
**Spec:** [spec.md](spec.md) · **Plan:** [plan.md](plan.md)

## 0. At a glance

We set out to replace `epic-create`'s single `paad:pushback` and single
`paad:alignment` pass with the paad authors' full design loop. That loop is
mode 1 of the umbrella's review doctrine. It runs spec pushback rounds, a
single plan-writing pass, plan pushback rounds and alignment rounds. Each
round uses a fresh review-only subagent, classifies its findings, has the
human decide each one, and ends with a checkpoint that recommends whether to
continue. What shipped is the doctrine (common core plus mode 1) and the
rewritten `epic-create` skill, released in vergil-claude-plugin v2.1.49. A
seeded spike validated them before release, and a real `epic-create` run
dogfooded them afterwards. Two follow-up fixes from the dogfood shipped, and
the docs in four repos were updated.

| PR | Merged | Task | What it did |
|---|---|---|---|
| .github#624 | 2026-10-28 | .github#554 | Published the spec and plan |
| vergil-claude-plugin#795 | 2026-10-28 | plugin#781 | Added the paad review doctrine (common core and mode 1) and updated the paad policy |
| vergil-claude-plugin#799 | 2026-10-28 | plugin#782 | Rewrote `epic-create` for human-gated design-review rounds |
| vergil-claude-plugin#815 | 2026-10-29 | plugin#812 | Gave the reviewer's security-check failure a distinct signal |
| vergil-claude-plugin#816 | 2026-10-29 | plugin#814 | Briefed the brainstorm from the planning issue and never skipped it implicitly |
| vergil-tooling#3811 | 2026-10-29 | vergil-tooling#3719 | Updated the site docs: the `epic-create` pipeline and the security-records guide |
| docs#76 | 2026-10-29 | docs#75 | Updated the management docs: workflow, tools and skills, index |
| vergil-claude-plugin#822 | 2026-10-29 | plugin#821 | Updated the `epic-create` row in the plugin README |

- **Repos touched:** 4 (.github, vergil-claude-plugin, vergil-tooling, docs).
- **Children:** 13. That is 8 code or docs tasks with a PR and 3 operational
  tasks: the spike .github#625, the dogfood .github#626 and the deploy
  plugin#783. It also includes 1 review task closed without a PR
  (plugin#811) and this retrospective.
- **Releases:** vergil-claude-plugin v2.1.49 (2026-10-28) carries #781 and
  #782, as attested in plugin#783. Whether a release yet carries #815, #816
  and #822 is **unknown**: the release notes don't list PRs.
- **Span:** planning issue opened 2026-10-25; epic created 2026-10-27; last
  task closed 2026-10-29.

## 1. How the plan evolved

`plan.md` has no evolution log and was not edited after it merged. The
deviations below are reconstructed from the issue record.

- **Planning ran twice, then paused.** Two `epic-create` sessions ran on #469
  at once. One set (epic #550 and its bookends) was abandoned, and its single
  useful idea, one canonical severity scale, was carried into this spec. The
  human then paused at spec pushback round 2 while sibling children A′ (#573),
  D (#568) and E (#558) settled overlapping questions. On resume the spec was
  reconciled with them:
  - it adopted A′'s security-records contract;
  - it adopted D's rule that paad writes inside the docs worktree, which
    reversed an earlier pushback finding;
  - it adopted E's recommendation tuning, where only new findings argue for
    another round and a repair-heavy round recommends stop;
  - it adopted D's `vrg-trusted-authors` for classification.
- **The spike failed first, and the fallback was not needed.** Task 1
  (.github#625) failed one criterion: in the seeded alignment run the reviewer
  reported a security seed as an ordinary finding. The human chose to tighten
  the prompt rather than take the spec §3.11 fallback. A self-test and a fixed
  security-file format were added, and the re-run passed. The contingent
  fallback task was never filed. The spike's four gaps went straight into the
  doctrine:
  - name the skill's own severity scale;
  - "write nothing" includes temporary files;
  - use one security-file format;
  - screening is load-bearing.

  The structural stray-write check came out of the second gap.
- **The dogfood added two tasks.** Task 5 (.github#626) ran mode 1 on a real
  planning issue (#468, epic #683). It found that the prompt used "hold" for
  both a routing answer and a failure signal, which broke two alignment
  rounds by the letter of the doctrine. That led to plugin#812 here and its
  mode 2 twin, plugin#813. It also found two brainstorm problems: re-asking a
  need the issue already stated, and treating "skip the questions" as "skip
  the brainstorm". Those went to intake #694 and became plugin#814.
- **A human read-through added a gating review.** plugin#811 asked whether the
  doctrine's complexity was design or workaround. It concluded that the
  complexity is mostly interface work adapting paad, which is written for a
  person, to an unattended subagent. It also found that paad's config
  mechanism can't express "skip a step". The fix moved out of this epic to
  #698: an unattended mode built as an upstream paad patch.
- **The documentation review spawned two per-repo tasks:** docs#75 and
  plugin#821.

## 2. Lessons learned

- **The seeded spike paid for itself.** It caught a real routing miss and a
  real stray write before anything shipped. Every one of its gaps became
  doctrine text. Keep seeded validation for prompt-driven mechanisms.
- **Dogfooding on real work finds what spikes don't.** One real run surfaced
  three problems the spike's synthetic setting never exercised: the "hold"
  wording and the two brainstorm problems.
- **Steering paad from the outside is fragile.** The stray-write check, the
  "hold" wording fix and the review-only directives are all compensations for
  a skill built for an interactive user. Fixing that at the source (#698) is
  better than layering more guards.
- **Sibling children that share a mechanism should settle it once.** C, D, A′
  and E each answered "where does paad write" and "how are security findings
  recorded". Pausing C to reconcile avoided a conflict, but cost a restart.
  Next time, settle a shared contract first, or sequence the siblings that
  share it.
- **The checkpoint recommendation leans toward continue.** *(judgment from
  the records)*
  - On #469 the recommendation said continue at every stage end (spec round
    3, plan round 1, alignment round 1), and the human stopped each time.
  - On the dogfood, it said continue at plan round 2 and alignment round 1,
    and the human stopped both times.
  - Across the six stage ends, the recommendation said stop only once (the
    dogfood's spec round 2), and the human agreed. At the other five it said
    continue, and the human stopped anyway. The rule may be too eager.

## 3. Compromises and tradeoffs

- **The stray-write check shipped knowing it was a workaround.** It is a
  backdated reviewer `TMPDIR` plus a sweep for new files. Under
  `epic-implement`'s parallel runs it gave false positives on all three
  documentation-review branches. The hits were harness task outputs and
  other sessions' scratch copies. The human has ruled it overboard and is
  removing it (#698; the specific gap is #707).
- **The doctrine stays as it is until #698 lands,** even though #811 judged
  much of it to be workaround.
- **vergil-tooling#3719 was accepted without a clean review round.** Its only
  round failed the stray-write check, so its 1 suggestion was never read, and
  the human accepted the branch as it stood.
- **docs#75 merged with 2 suggestion fixes committed after the reviewed
  head.** The handoff notes marked them as unreviewed.
- **The dogfood left two paths unexercised:** resuming a session, and
  classification against an untrusted comment. No timings were recorded.

## 4. New problems and opportunities

| Item | Where it went |
|---|---|
| Doctrine complexity is interface work; paad config can't skip steps | Planning issue .github#698 (open): paad unattended mode, an upstream patch |
| The stray-write sweep fails on harness task outputs and parallel scratch copies | .github#707 (open) |
| The short `epic-create` pipeline chains leave out the human spec review | .github#713 (open) |
| "hold" was ambiguous in mode 1 and mode 2 | Fixed: plugin#812, plugin#813 |
| The brainstorm re-asked a stated need, or skipped the stage on "skip the questions" | Fixed: plugin#814 (from intake #694) |
| The intake epic .github#438 reached GitHub's 100-sub-issue cap, so #707 could not be linked | Intake .github#714. The human drained the queue, and #707 is now linked. The tooling's handling of a full parent is still open |
| Two `epic-create` sessions can run on one planning issue at once | Intake .github#715 |
| The recommendation rule leans toward continue (§2) | Intake .github#716 |
| The dogfood did not exercise resume or untrusted-comment classification | Intake .github#717 |

### Unaddressed-findings sweep

- **Scope:** 12 children were swept (all except this retrospective), along
  with their 8 merged PRs and the planning issue #469. The trusted authors
  are from `vrg-trusted-authors`. No merged PR had a review or a
  conversation comment.
- **With an agentic-review summary:**
  - vergil-tooling#3719: accepted after 1 round. 1 suggestion was never used
    because the round failed its output checks.
  - docs#75: pass after 1 round. Its R1.1 merge-ordering concern is
    resolved, because vergil-tooling#3811 merged before docs#76. Its R1.3 is
    tracked as #713.
  - plugin#821: pass after 1 round, with nothing unfixed.
- **Without gate records:**
  - Operational tasks: the spike .github#625, the dogfood .github#626 and
    the deploy plugin#783. The docs task .github#554 is covered by the
    design-review rounds on #469 instead. The review task plugin#811 closed
    with a comment and no PR.
  - Code and docs tasks: plugin#781, #782, #812 and #814, implemented
    2026-10-28 to 29 morning. Whether mode 2 was live in their sessions is
    not established from the record.
- **Planning issue #469:**
  - Every design-review finding was accepted: spec pushback 4 + 13 + 5,
    plan pushback 9, alignment 17. None needs a reason or an intake ref.
  - Its reconciliation and input comments were all absorbed into the spec.
- **Untracked problems:** 4, all filed as intake during the sweep
  (#714–#717). Everything else surfaced in comments was found already
  tracked.

## 5. What's next

- **.github#698:** paad unattended mode, being planned now. When it lands,
  the doctrine is simplified and the stray-write check removed.
- **Umbrella #500:** with C done, its gate .github#510 can run. Child F
  (planning item .github#507, wider paad adoption) is unblocked once C and D
  are both done, and both now are.
