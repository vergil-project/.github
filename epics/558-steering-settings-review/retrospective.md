# Steering-file and Claude settings review — Retrospective

**Epic:** vergil-project/.github#558 · **Spec:** [`spec.md`](spec.md) ·
**Plan:** [`plan.md`](plan.md) · **Planning:** vergil-project/.github#506

## §0 At a glance

We set out to write down which file owns each Claude Code setting and
instruction, clean up every steering and settings file in the managed fleet to
match, and keep them clean: a nightly audit for what a machine can check, and a
`steering-audit` skill for the periodic review that needs judgment. All of it
shipped. Each of the 18 managed repos in three orgs now has a `CLAUDE.md` of at
most 200 lines with the managed template region, a `.claude/settings.json` that
equals the template, and no `AGENTS.md`. The nightly audit enforces all three.
The user-level files hold only personal preferences.

| | |
|---|---|
| Planning opened → epic closed | 2026-10-26 (#506) → 2026-10-28 |
| Epic opened → last task closed | 2026-10-27 12:35Z → 2026-10-28 19:33Z |
| Epic children | 30 (29 tasks + this retrospective) |
| Merged PRs | 23 epic-task PRs, 9 cross-org sweep PRs, 4 ad-hoc fixes |
| Operational tasks | 6 (3 deployment, 3 validation), all SUCCESS |
| Repos touched | 18 managed repos across vergil-project, logical-minds-foundry and mnemosys-project |
| Releases | vergil-tooling v2.1.275 (bridged audit), v2.1.278 (tightened audit); vergil-claude-plugin v2.1.45 (steering-audit, deprecated skills removed); vergil-containers v2.1.33 (publish fixes) |
| `CLAUDE.md` lines, fleet total | 3,236 → 1,819 (−44%) |

### Work delivered

| Task | PR | What it did |
|---|---|---|
| .github#559 | .github#597 | Spec and plan |
| vergil-tooling#3723 (Task 1) | #3750 | Ownership model page, platform facts re-verified |
| vergil-tooling#3743 (Task 2b) | #3752 | Local-settings guard on `vrg-commit`, `report-ready`, `vrg-container-run` |
| vergil-tooling#3724 (Task 2) | #3757 | New `CLAUDE.md` and settings templates; bridged audit |
| vergil-tooling#3755 | #3759 | Permission-model docs: the local-settings rule |
| vergil-claude-plugin#763 (Task 4b) | #769 | Removed `memory-init`, `memory-audit`, `vergil:handoff` |
| vergil-claude-plugin#754 (Task 4) | #770 | `steering-audit` skill (repo and user scope); skill frontmatter check |
| vergil-tooling#3772 | #3784 | `vrg-reword` refuses a frozen branch |
| vergil-tooling#3773 | #3785 | `vrg-hook-guard` and the `guard.sh` shim fail closed |
| vergil-claude-plugin#776 | #777 | Plugin hook scripts fail closed |
| vergil-tooling#3727 (5a) | #3781 | vergil-tooling sweep: `CLAUDE.md` 456 → 144, `AGENTS.md` removed, reference detail moved to site docs |
| 5b–5i (8 repos) | plugin#788, containers#735, actions#1091, .github#644, docs#70, vm#372, python#69, packages#38 | vergil-project sweeps |
| 5j–5p + 2 scan-added | lmf docs#25, gateway-replay-lab#331, protocol-gateway#37, resiliency-lab#1701, resiliency-observability#64, lmf .github#392; mnemosys docs#24, melete#260, .github#109 | Cross-org sweeps (filed under each repo's own ad-hoc epic) |
| vergil-tooling#3728 (Task 7) | #3797 | Bridge removed; 200-line and no-`AGENTS.md` rules enforced |
| vergil-tooling#3776 | #3798 | CLI notes: reword freeze, guard fail-closed |
| vergil-tooling#3720 (docs review) | #3804 | Site docs brought to the final model |
| vergil-claude-plugin#809 | #810 | Plugin install docs: marketplace in user settings |

Ad-hoc fixes made along the way (not epic children): vergil-tooling#3790
(`report-ready` lease push after unfreeze), vergil-claude-plugin#789 (skills file
follow-ups at once), vergil-containers#733 and #734 (prod nightly from `main`'s
workflow; new CVE ignores).

## §1 How the plan evolved

`plan.md` has no "Evolution during execution" log; the deviations below are
reconstructed from the task and issue records. The plan's task sequence held.
What changed was mostly scope found in flight and how gates were met.

- **Fact 10 was already stale at Task 1.** The live docs showed `/doctor` is a
  bundled skill and the prompt audit runs through `claude-api`. The human chose
  to restate it on the model page only (no spec change); whether `steering-audit`
  should call it went to intake (.github#604).
- **The fleet was 18 repos, not 16.** Task 3's fleet scan found
  `logical-minds-foundry/.github` and `mnemosys-project/.github`, which also pin
  v2.1 and run the audit. Both got sweep tasks under the plan's "plus any repo the
  scan adds".
- **The findings-review model loosened as trust grew.** 5a's 33 findings were
  walked one at a time. For 5b–5i the human accepted all recommendations and
  walked only new judgment calls. For the nine cross-org sweeps the agents
  decided by the ownership model and stopped only on real problems. Quality held:
  every sweep validated clean and the second review found few new items.
- **Nightly runs became dispatched runs.** Tasks 3, 5-gate and 8 waited for "the
  next nightly". After the first day the human dispatched `ops.yml` by hand in all
  18 repos. That is the same workflow on the same code, and it cut hours off each
  gate (.github#646).
- **Scope added mid-flight, filed under the epic or ad hoc:** the
  `vrg-reword` freeze gap and the hook guard's fail-open (both found by 5a's
  second review), the plugin's hooks failing open, the CLI notes follow-up, the
  permission-model docs conflict, and plugin#809 from the docs review.
- **Not corrected in the plan:** #3720 noted a wording conflict in `plan.md`
  (Decided content against Task 6's "take anything new to intake"); `plan.md`
  was never amended because it lives in another repo. The general rule — plans
  evolve append-only with a log — is intake .github#634.
- **Small plan inaccuracies** surfaced in validation: Task 9 said "both"
  marketplaces where there are three; Task 6's "6 sections: 3 move, 3 keep"
  became 7 findings across both user files (3 sections deleted because the
  template now covers them, 2 settings deletions, 2 keeps).

## §2 Lessons learned

- **A bridge plus a hard gate works.** Shipping the new template with an audit
  that warned on the previous shape let 18 repos be swept over two days without a
  red nightly, and Task 7's removal landed with the fleet already clean.
- **Parallel sub-agents per repo scale; the human's time is the constraint.**
  Up to nine sweeps ran at once. The bottleneck was deciding findings, which is
  why the review model moved toward accept-all and then delegated judgment.
- **File follow-ups the moment they surface.** Early in the run follow-ups were
  offered "for later". The human asked that every one be filed immediately, and
  that became a skill rule (vergil-claude-plugin#787). By the end, every gap
  found was an issue before the next step.
- **Read a task's comments, not just its body, before implementing.** Task 7
  merged against an ordering note another session had left on #3728 (see §3).
  The briefs given to sub-agents pointed at the plan and issue body only.
- **Validation tasks are worth running for real.** The cold rebuild (Task 9) and
  the dispatched audits found real gaps: `vrg-vm update` not re-copying
  `~/.claude/` (.github#670), plugins returning at project scope (.github#675),
  and a false failure in `vrg-finalize-pr` (.github#643).
- **First run of the agentic review gate (epic 568).** It ran on the last two
  tasks and worked, but its write check failed both rounds on other sessions'
  files (.github#679), so each needed a human ruling.

## §3 Compromises & tradeoffs

- **The bridge was removed before vergil-tooling#3795 merged**, against the
  ordering note on #3728. The template can no longer change without every repo
  failing until synced. #3795's plan is stale and needs a decision: a standing
  N-1 window (.github#580), a one-off bridge, or one red nightly.
- **Line counts are hard-coded in some `CLAUDE.md` files** (for example "ten"
  auto-managed tools in vergil-containers) and can go stale the way "nine" did.
- **Four sweep PRs carried an unrelated `fix(deps)` commit** (msgpack advisory)
  because `develop` failed validation. They were squash-merged under docs titles;
  the fleet-wide bump is .github#645.
- **Commits made after a review round are unreviewed** (#3804: 3, #810: 1), as
  the doctrine allows; the PR notes disclose them.
- **The user-level review relied on the decided classification** and the human's
  ruling, not a fresh judgment of every section; the public record holds counts
  only, by design.
- **The plugin probe in Task 9 is recorded, not resolved.** The `vm_guest`
  reconcile stays until .github#675 explains the project-scope result.

## §4 New problems & opportunities

**Unaddressed-findings sweep.** 29 tasks swept (this retrospective excluded).
2 had an `agentic-review summary:` (vergil-tooling#3720, vergil-claude-plugin#809);
0 were swept from round comments; 27 had no gate records — 6 operational tasks
(#577, #578, #600, plugin#764, tooling#3726, tooling#3729), the spec task #559
(its mode-1 record is on #506), and 20 code tasks closed before the gate shipped
(plugin v2.1.48, 2026-10-28 16:16Z). All issue comments were by trusted authors.
None of the 36 PRs checked had a review or PR comment. The planning issue's
design-review rounds had no unaccepted finding without a reason or intake ref.
Security-class findings appear as counts only: 9 across the design-review rounds,
1 in #3720's review (fixed on the branch), 0 in #809's.

Candidates found untracked by the sweep, and their outcome:

| Problem | Outcome |
|---|---|
| `plan.md` contradiction noted on #3720 never corrected | found .github#634 (plan-evolution rule); recorded in §1 |
| Bridge removed before #3795; its plan stale | found vergil-tooling#3795 (status comment added) and .github#580 |
| Stale project-scope plugin copy on the VM (#764) | found .github#675, .github#670 |
| Transient image-publish failures (#3729) | found vergil-containers#743 |
| Unresolved ref ".github#628" on #560 | it was transferred: now vergil-tooling#3783 |
| lmf PR #64 body says the memory skills remain | PR body only; that repo's `CLAUDE.md` is correct — **dropped by the human:** a merged PR body is history and the repo's steering file is right, so there is nothing to fix |
| Plan said "both" marketplaces; Task 6 counts differ from the plan | recorded in §1 |

**Other issues the epic surfaced** (all filed during execution):

| Area | Refs |
|---|---|
| Review gate | .github#679 |
| Operational tasks and the nightly | .github#646, #643 |
| PR handoff | vergil-tooling#3783 (done), vergil-claude-plugin#786 |
| Agent VMs and Claude config | .github#670, #675, #666 |
| Config audit and validation | .github#537, #637, #631, #645 |
| Docs and steering content | .github#604, #615, #616, #632–#642, #680, #681; lmf .github#390, #391; mq-protocol-gateway#36; mnemosys .github#108 |
| Settings template | vergil-tooling#3795 (idle compaction), .github#580 (N-1 window) |
| Containers | vergil-containers#742, #743 |

## §5 What's next

- **Process and tooling follow-on:** planning item vergil-project/.github#686
  gathers the gaps this first full `epic-implement` run surfaced (review gate,
  operational tasks, PR handoff, agent VMs, audit and validation). It becomes an
  epic through `epic-create` when it is next to be implemented.
- **Settings template change:** vergil-tooling#3795 needs its approach decided
  (see §3) before it can ship.
- **Recurring review:** run `steering-audit` at least quarterly and on any Claude
  Code release that changes settings or instruction-file behavior; the next
  user-scope pass reviews the three sections that stayed.

## Appendix A — Operational notes

- **Release order that kept the fleet green:** bridged audit (v2.1.275) → plugin
  v2.1.45 → sweeps → cross-org gate → tightened audit (v2.1.278) → fleet audit.
- **Dispatching the fleet audit** (human only; agents can't run `workflow run`):

  ```bash
  for r in vergil-project/{.github,vergil-tooling,vergil-claude-plugin,vergil-python,docs,packages,vergil-vm,vergil-actions,vergil-containers} logical-minds-foundry/{.github,docs,mq-gateway-replay-lab,mq-protocol-gateway,mq-resiliency-lab-for-linux,mq-resiliency-observability} mnemosys-project/{.github,docs,melete}; do gh workflow run ops.yml --repo "$r"; done
  ```

- **Gotchas:** a `github-config` job log is readable only after the whole run
  ends (vergil-containers' image rebuilds run long); `vrg-vm update` does not
  re-copy `~/.claude/` — start a `vrg-vm session` or rebuild; the `claude plugin`
  uninstall and marketplace-remove commands also edit `settings.json`; superseded
  CD runs (one concurrency group for dev publishes) are not failures.
- **vergil-containers prod nightly** was broken before this epic (workflow from
  `develop`, files from `main`); #733 runs prod from `main`'s copy of the
  workflow.
