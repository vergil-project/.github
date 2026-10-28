# Several Advisories per Issue — Retrospective

Epic: [vergil-project/.github#573](https://github.com/vergil-project/.github/issues/573) ·
Spec: [`spec.md`](spec.md) · Plan: [`plan.md`](plan.md)

## 0. At a glance

**What we set out to do.** Epic #517 gave each security finding a private draft advisory paired
with a neutral `security` issue. The tooling assumed one advisory per issue: it read only the
first `Security advisory:` line. Umbrella #500's children C (design review in `epic-create`) and
D (agentic review in `issue-implement`) will raise several findings against one issue. So this
epic set out to let one issue name several advisories everywhere in the tooling:

- the library, the scheduled sweep and `show`;
- intake routing and the planning hold;
- the `epic-create` handoff;

so that C and D can use the existing commands with no new ones.

**What shipped.** All of it, plus five unplanned tooling and plugin fixes and two spawned doc
tasks:

- vergil-tooling **v2.1.274** added several advisories per issue (T1, T2), the docs (T4) and two
  wording fixes.
- vergil-claude-plugin **v2.1.44** added the `epic-create` per-advisory handoff (T3) and its
  hold-note quote.
- The release itself found two `vrg-vm update` defects (v2.1.275, v2.1.276) and a missing
  version in the `vrg-release` summary (v2.1.275).
- The deployment task failed once, on one of those defects, then passed.
- The end-to-end check passed first time: two advisories on one issue, moved one at a time, and
  closed by the sweep.

| Repo | PR | Task | What it did |
|---|---|---|---|
| .github | [#585](https://github.com/vergil-project/.github/pull/585) | #574 | Spec and plan |
| vergil-tooling | [#3734](https://github.com/vergil-project/vergil-tooling/pull/3734) | #3730 (T1) | `parse_advisory_refs`; the sweep and `show` handle every advisory line |
| vergil-tooling | [#3735](https://github.com/vergil-project/vergil-tooling/pull/3735) | #3731 (T2) | Transfer and recreate retarget every advisory, split refuses several, the planning hold covers each |
| vergil-claude-plugin | [#757](https://github.com/vergil-project/vergil-claude-plugin/pull/757) | #756 (T3) | `epic-create` step 9: one retarget per advisory, one at a time |
| vergil-tooling | [#3745](https://github.com/vergil-project/vergil-tooling/pull/3745) | #3732 (T4) | Security-records guide, CLI reference, the C and D recipes, the migration check |
| vergil-tooling | [#3739](https://github.com/vergil-project/vergil-tooling/pull/3739) | #3736 *(added)* | Count-aware plurals in the held note and the `show` refusal |
| vergil-tooling | [#3744](https://github.com/vergil-project/vergil-tooling/pull/3744) | #3738 *(added)* | Held note ends "the task or tasks that fix it" |
| vergil-claude-plugin | [#765](https://github.com/vergil-project/vergil-claude-plugin/pull/765) | #758 *(added)* | `epic-create` step 10 quotes the corrected held note |
| vergil-tooling | [#3754](https://github.com/vergil-project/vergil-tooling/pull/3754) | #3751 *(added)* | `vrg-vm update` keeps a marketplace's declared ref |
| vergil-tooling | [#3756](https://github.com/vergil-project/vergil-tooling/pull/3756) | #3753 *(added)* | `vrg-release` prints the released version in its summary |
| vergil-tooling | [#3768](https://github.com/vergil-project/vergil-tooling/pull/3768) | #3764 *(added)* | `vrg-vm update` updates every plugin install, not one per plugin id |
| vergil-tooling | [#3778](https://github.com/vergil-project/vergil-tooling/pull/3778) | #3725 | Documentation review: issue standards, `CLAUDE.md`, the agent VM guide |
| vergil-claude-plugin | [#775](https://github.com/vergil-project/vergil-claude-plugin/pull/775) | #774 *(spawned)* | Skills catalogue and step 9 say "the task or tasks that fix it" |
| docs | [#69](https://github.com/vergil-project/docs/pull/69) | docs#68 *(spawned)* | Management pages cover several advisories per item |

- **Repos touched (4):** .github, vergil-tooling, vergil-claude-plugin, docs.
- **Children (17):**
  - 9 planned: T1–T6 and three bookends (#574, #3725, #575).
  - 8 added during execution:
    - three wording fixes (#3736, #3738, #758);
    - three tooling defects or gaps found while releasing (#3751, #3753, #3764);
    - two doc tasks spawned by the docs review (#774, docs#68).
  - 2 operational: deployment vergil-tooling#3733 (T5) and the end-to-end check .github#584
    (T6).
- **PRs merged:** 14, including the spec/plan PR and excluding this one.
- **Releases cut (by the human):**
  - vergil-tooling v2.1.274, v2.1.275 and v2.1.276.
  - vergil-claude-plugin v2.1.44 and v2.1.46. v2.1.45, in between, held nothing from this epic.
  - The docs review (#3778) is on develop but not yet in a tooling release. The docs repo has no
    releases.
- **Span:** the planning issue (#571) opened 2026-10-27 14:16Z and the epic 14:24Z. The last
  task before this one (#3725) closed 2026-10-28 12:40Z, about 22 hours.

## 1. How the plan evolved

The plan has no "Evolution during execution" log, so this section is reconstructed from the issue
record and the session.

- **The design was cut back hard before the spec was approved.** Planning started from the idea
  of one dedicated record issue per finding. That grew into an intake hold, a release step,
  defer guards and placement rules. The human stopped it as runaway complexity and restated the
  requirement: one issue must be able to name several advisories. The one-issue-per-finding idea
  had been a proposed mechanism, not a requirement. Two review rounds cut the spec down to
  multi-advisory correctness plus the existing `create --issue`, `retarget` and `split`
  commands. C and D got their recipes as comments on #469 and plugin#677.
- **T1–T4 shipped as planned.** T1 needed no sweep code: `vrg-epic-audit` picked up the change
  through `classify`, and a test pins it. T2 removed the old single-advisory parser once its last
  callers moved.
- **The plan's own wording was rejected.** The plan fixed the held note as
  `held: <k> advisory(ies) still track …`, and T2 copied it exactly. The human rejected the
  parenthesised plural, so #3736 made the noun and verb agree with the count. The issue's
  acceptance criteria contradicted themselves on the note's "fix task(s)" ending. #3738 fixed
  that with count-free wording the human chose. T3's skill text quoted the old note verbatim, so
  #758 updated the quote. That was three extra PRs for one message.
- **The release found two `vrg-vm` defects.** The tooling release's consumer refresh runs
  `vrg-vm update`.
  - It failed first because `vrg-vm` dropped the `ref` of a declared marketplace, and Claude Code
    2.1.306 now refuses a `marketplace add` that does not match the declared source (#3751).
  - With that fixed, the deployment task's check of the installed skill text failed. The VM's
    user scope and two of its three project installs were still on plugin 2.1.43. `vrg-vm` keyed
    `claude plugin list` by plugin id and updated only one install per plugin (#3764).
  - The human also asked for the released version to be shown in the `vrg-release` summary,
    which had never named it (#3753).

  The human had all three fixed under this epic, and T5 passed on v2.1.276.
- **T6 passed first time.** The migration count found 14 records and 0 broken pairs. The
  two-advisory test passed every step, and the sweep log never showed the probe text.
- **The docs review spawned two tasks and caused a conflict elsewhere.** It fixed three
  vergil-tooling files and filed doc tasks in the plugin and docs repos. Its one-sentence
  `CLAUDE.md` edit then conflicted, at submit time, with epic 558's #3727, which removes that
  whole section. That conflict was left to epic 558's session.

## 2. Lessons learned

- **When a design grows, restate the requirement.** The planning spiral came from treating a
  proposed mechanism (one issue per finding) as a requirement. Each fix to that mechanism added
  another. Asking "what must be true when this is done?" cut the spec to a fraction of its size,
  with no loss for C or D.
- **Plans should not freeze user-facing wording nobody has read aloud.** The plan's
  `advisory(ies)` passed spec review, plan review and alignment, and implementation copied it
  exactly. Messages a person will read deserve one look for how they read, at plan time.
- **Quoting tool output in another repo's docs or skills couples the two.** Each wording change
  to the held note cost a plugin task and a plugin release. Quote a message only where the exact
  text matters. Elsewhere, describe it.
- **Acceptance criteria can contradict each other.** #3736 said both "the rest of the message is
  unchanged" and "no `(s)` plurals remain". The implementer had to pick one and flag it, which
  cost another task. Criteria are worth reading as a set before filing.
- **Checking the installed artifact catches real defects.** #517's retrospective recommended
  checking the installed skill text, not the release number. T5 did that and found #3764, a
  defect that had silently left most plugin installs stale on every `vrg-vm update`. A check
  that "the update ran" would have passed.
- **A failed consumer refresh is a real bug, not noise.** Both `vrg-vm` defects surfaced only
  because a release ran `vrg-vm update` and someone read the error.
- **Say "tests first" in the sub-agent prompt.** T2's agent wrote tests and code together. Every
  later prompt asked for the failing run explicitly, and every later agent did it.

## 3. Compromises & tradeoffs

- **T2's tests were not seen failing first.** They exist, cover the plan's cases plus four
  failure paths, and pass. The skipped red step was reported and accepted, since the branch was
  frozen.
- **`vrg-vm`'s git-URL ref form was not tried live.** #3751 passes `<url>#<ref>` for a git URL
  source, following the Claude Code docs. No settings use a URL source today, so it ran only in
  unit tests. The GitHub form (`owner/repo@ref`) was checked in the VM.
- **The migration check had nothing to migrate.** No existing issue named more than one
  advisory, so the check showed only that the new code reads today's records cleanly.
- **Two parenthesised plurals were left for later.** The sweep's summary line
  (`advisory state(s) synced, <m> broken pair(s)`, from #517) and an `impl task(s)` diagram
  label in `epic-create` were filed in the ad-hoc epics as vergil-tooling#3779 and
  vergil-claude-plugin#784. The human judged them too small to hold the epic.
- **Some sub-agents ran `pytest` directly.** To watch new tests fail first, two agents ran
  `uv run pytest` inside the container, outside the "only `vrg-validate`" rule. Final validation
  was always `vrg-validate`.
- **Two security findings from spec review were resolved by design changes, with no record.** The
  findings came from spec review rounds 2 and 3, and the cut-back design removed what they were
  about. Their notes stayed in a session scratchpad, and no `vrg-security-record` was created.

## 4. New problems & opportunities

| Item | Where it went |
|---|---|
| `vrg-vm update` dropped a declared marketplace `ref` | Fixed: vergil-tooling#3751 (v2.1.275) |
| `vrg-vm update` updated one plugin install per plugin id, leaving the rest stale | Fixed: vergil-tooling#3764 (v2.1.276) |
| `vrg-release` never printed the version it released | Fixed: vergil-tooling#3753 (v2.1.275) |
| Parenthesised plurals in the held note and the `show` refusal | Fixed: vergil-tooling#3736, #3738; plugin quote #758 |
| Parenthesised plurals in the sweep summary line | Ad-hoc: vergil-tooling#3779 |
| `impl task(s)` in the `epic-create` operational-task diagram | Ad-hoc: vergil-claude-plugin#784 |
| The docs review's `CLAUDE.md` edit conflicts with epic 558's `CLAUDE.md` trim (#3727) | Handed to epic 558's session; keeping #3727's version loses nothing |
| "fix task(s)" in the validation-task scaffold and `issue-validate` | Logged, not acted on |

## 5. What's next

- Umbrella vergil-project/.github#500's children C (planning #469, epic #553) and D
  (vergil-claude-plugin#677, epic #568) use the recipes in the security-records guide. C uses
  `create --issue` plus one retarget per advisory at step 9. D uses plain `create`, then `split`
  when the fix is on the branch.
- The two ad-hoc wording items, #3779 and plugin#784, go in the next ad-hoc batch.
- No follow-on brainstorm was opened from this epic.

## Appendix A — Operational notes

**Release and deploy order.**

1. T1, T2, T4, #3736 and #3738 merged, then tooling v2.1.274.
2. T3 and #758 merged, then plugin v2.1.44.
3. The release's `vrg-vm update` failed (#3751). #3751 and #3753 merged, then tooling v2.1.275.
4. T5 failed: three of four plugin installs were stale (#3764).
5. #3764 merged, then tooling v2.1.276. `vrg-vm update` moved all four installs to plugin 2.1.45.
6. T5 passed, then T6.
7. The docs review and its two spawned tasks; plugin v2.1.46 carries #774.

**Sweep runs used as evidence (vergil-project/.github).**

| Run | Purpose | Result |
|---|---|---|
| [40683148951](https://github.com/vergil-project/.github/actions/runs/40683148951) | T6 migration check, on v2.1.276 | `0 advisory state(s) synced, 0 broken pair(s)` |
| [40684168173](https://github.com/vergil-project/.github/actions/runs/40684168173) | T6 step 6, after X closed | `2 advisory state(s) synced, 0 broken pair(s)`; no `PROBE-A′` in the log |

**End-to-end test records.**

- Advisories A (GHSA-9c38-m4qm-642q) and B (GHSA-6h9w-8wm3-g2g7) were attached to test issue I
  (.github#613), then retargeted one at a time to X (.github#614).
- I was dropped and X closed, and both advisories are now closed. They stay as the record of the
  test.
