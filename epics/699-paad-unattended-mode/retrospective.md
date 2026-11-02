# Retrospective: paad unattended mode (epic #699)

## §0 At a glance

We set out to let vergil's review rounds run paad without a person present: a
`paad:paad-delegate` runner in paad that asks nothing, writes nothing and
returns one structured result, loaded into the fleet from our paad fork, with
vergil's review doctrine moved onto it. What shipped goes further than the
plan. The delegate and unattended mode for pushback, alignment and
agentic-review are on the fork (`unattended-mode`, five commits) and run
fleet-wide through `vrg-vm session`'s `--plugin-dir` override. The doctrine
runs every review round on the delegate. The fallback path the plan kept "until
a paad release carries the mode" is gone, because the human made the fork setup
permanent. Mode 2 finding handling was redesigned mid-epic, so the human
approves every security action. The upstream offer is held until paad is fully
integrated.

- **Span:** opened 2026-10-29, last task merged 2026-11-02; closes when this
  retrospective merges.
- **Tasks:** 46 (26 closed by PR, 19 by comment or as delivered by another
  task's PR, plus this one).
- **PRs:** 26 across 9 repos: .github, docs, packages, vergil-actions,
  vergil-claude-plugin, vergil-containers, vergil-python, vergil-tooling and
  vergil-vm. The paad fork (`wphillipmoore/paad`) took 5 commits and no PRs.
- **Releases:**
  - vergil-tooling 2.1.279 (#3843) and 2.1.280 (#3861); #3901 is unreleased.
  - vergil-claude-plugin 2.1.52–2.1.56.
  - vergil-vm 2.1.49 (#378); #384 and #386 are unreleased.
  - Other repos: not applicable.
- **Security (counts only):** 92 found by agentic review across 6 tasks, 75
  fixed on their branches. 1 was recorded and 1 rejected; the rest were declined,
  deferred or ended by an earlier ruling (details by task in Appendix B). Of
  the deferrals, 3 were recorded privately at the retrospective and 1 rejected. Task
  9's validation recorded 4 more, in 2 records. 1 was reported privately to the
  paad author.

| Task | PR | Merged | What it did |
|---|---|---|---|
| .github#700 | .github#736 | 10-29 | Spec and plan |
| vergil-tooling#3824 (Task 7) | vergil-tooling#3843 | 10-29 | `--plugin-dir` override read from `plugin-dirs.toml` |
| .github#764 | .github#779 | 10-30 | Amended spec and plan: mode 2 finding handling |
| vergil-tooling#3860 (Task 16) | vergil-tooling#3861 | 10-30 | Worktree convention allows ignored `build/` scratch |
| vergil-vm#376 (Task 14) | vergil-vm#378 | 10-30 | `make` on the agent VM |
| 8 template syncs | .github#788, docs#78, packages#42, vergil-actions#1098, vergil-claude-plugin#855, vergil-containers#748, vergil-python#75, vergil-vm#384 | 10-30 | Consumer `CLAUDE.md`: the `build/` exception line |
| vergil-claude-plugin#829 (13a) | vergil-claude-plugin#859 | 10-31 | State directories and their guards |
| vergil-claude-plugin#849 (13b) | vergil-claude-plugin#862 | 10-31 | Mode 2 pauses on findings that need the human; every fix is reviewed |
| vergil-claude-plugin#853 (13c) | vergil-claude-plugin#870 | 10-31 | Creating an issue needs the human's approval |
| vergil-claude-plugin#827 (Task 10) | vergil-claude-plugin#878 | 11-02 | Doctrine: the delegate path, with the fallback |
| vergil-claude-plugin#879 (Task 17) | vergil-claude-plugin#885 | 11-02 | Fallback path removed; the delegate required |
| .github#844 | .github#845 | 11-02 | Plan amended with Tasks 17–21 |
| vergil-claude-plugin#883 (Task 18) | vergil-claude-plugin#887 | 11-02 | Counts-line and `Scope:` checks; the notice's entities decoded |
| vergil-claude-plugin#886 (Task 21) | vergil-claude-plugin#888 | 11-02 | Security discard lines in the result template |
| vergil-claude-plugin#820 (docs review) | vergil-claude-plugin#892 | 11-02 | Docs sweep |
| vergil-claude-plugin#893 | vergil-claude-plugin#896 | 11-02 | Write and Edit allowed anywhere in the main clone's ignored `build/` |
| vergil-tooling#3900 | vergil-tooling#3901 | 11-02 | Docs: the `build/` exception, mode 2 security records, the delegate |
| docs#79 | docs#80 | 11-02 | Docs: review rounds need the delegate and a Lima session |
| vergil-vm#385 | vergil-vm#386 | 11-02 | Docs: `make`, `socat` and `chrony` in the tool tables |

Closed by comment:

- **Fork:**
  - .github#726 (Task 1): the fork was kept, not recreated, a recorded deviation.
  - .github#727: the spike.
  - .github#728–#731: fork commits 7f4eb0b, 8bd2bf2, 353f66c and abf879f.
  - .github#825: fork commit 58db477.
- **Deployment:** .github#732.
- **Validation:** .github#733 (20 of 20 unattended runs and 4 of 4 interactive)
  and .github#843 (a reduced set of 6).
- **Brainstorm:** .github#734 (other harnesses, not pursued).
- **Decision:** .github#735.
- **Private report:** .github#768.
- **Delivered by another task's PR:** vergil-claude-plugin#838–#841 and #848.
- **Declined:** vergil-claude-plugin#846.

## §1 How the plan evolved

The plan has no "Evolution during execution" log. It changed through two
amendments, #764 and #844, and through work added after the docs sweep. This is
what happened and why.

1. **Mode 2 was redesigned before deployment (#764, 10-29).** Tasks 1–7 went as
   planned, apart from Task 1 keeping the existing fork rather than recreating
   it. Before Task 8, the epic paused. The handling of mode 2 findings let an
   agent act on security findings on its own, and #755 had just shown how a
   review gate ratchets a design when nobody questions the threat model. The
   amendment added Task 13 (13a state directories, 13b the mode 2 loop with
   pauses and human rulings, 13c issue creation needs approval), Task 14
   (`make` on the agent VM), Task 15 (a private report to the paad author) and
   Task 16 (the `build/` exception in the worktree convention). It also folded
   in four vergil-claude-plugin issues (#838–#841), retargeted #848, and declined
   a fifth (#846). Task 16's template change then produced eight mechanical
   `CLAUDE.md` sync PRs.
2. **Validation needed one prompt amendment (#733).** The first planted runs
   failed "cleanly": a specialist ran the module under review and left a
   bytecode cache. The prompts gained a rule to read the code and never run it,
   and every run after that passed. The recorded-commit runs also found 4 real
   security weaknesses on develop, which were recorded privately.
3. **Task 11 was researched and dropped.** Kiro and Antigravity are closed
   source, which the human rules out whether paid or not. Pi is open source, but
   what it would add is largely covered already.
4. **Task 12 changed the direction (#735).** The human held the upstream offer:
   more fork work is coming, and the collected changes will go to the author
   together, probably as a paad 2.1. The human also made the fork setup
   permanent: the fleet runs the fork's tree, tracks upstream and stays a patch
   or two ahead. That removed the reason for the fallback path, so retiring it
   became Task 17 rather than a cleanup deferred to a paad release. Of the
   planned cleanup, the `plugin-dirs.toml` removal was dropped; the
   state-directory guards were deferred (.github#824); fork syncing became
   .github#823.
5. **The delegate's first fleet use drove Tasks 18–21 (#844).** Task 17's own
   review rounds were the first real delegate rounds. They showed format slips
   that 20 validation runs had not:
   - a counts line that disagreed with the findings;
   - a malformed `Defaults applied:`;
   - stray lines in the security section;
   - HTML entities in the completion notice.

   The doctrine gained checks (Task 18). The fork's contract gained rules (Task
   19, commit 58db477). The fleet moved under a §10 gate (Task 20, a reduced set
   by the human's ruling, because the change was to the contract's wording only).
   The doctrine's template caught up with the contract (Task 21).
6. **The docs sweep raised one design question (#893).** The sweep spawned doc
   tasks for vergil-tooling, docs and vergil-vm. It also found that the consumer
   `CLAUDE.md` allows ignored scratch anywhere in `build/`, while the write hook
   allowed only `build/.vergil-state/`. The human decided in favour of the
   template. `build/` is the scratch area the human can see, and agents' temp
   directories are invisible to them, so the hook was widened, with a
   `git check-ignore` gate.

## §2 Lessons learned

- **The fail-closed security procedure attracted a review ratchet, again.** On
  #827 the security findings ran 6, 8, then 13 across three rounds; 7 of the 13
  were gaps in the previous round's own fixes. #879 and #883 showed the same
  shape, and #893's `check-ignore` fix produced three new edge cases in the
  next round. This is #755's lesson in a new place. What stopped it was stating
  the threat model: the write hook guards against accidental edits, not a
  hostile agent, because Bash can write anywhere. Next time, state the threat
  model before the first round of any guard change, not in round 2.
- **Removing machinery beat hardening it.** Task 17 deleted the fallback path
  and, with it, the stray-write sweep. Every round of #827 had paused on the
  sweep's false positives (#858). Simplification should be the first question
  when a mechanism keeps generating findings.
- **One validation run per case shows a rule works, not that it holds.** The
  delegate's output varies from run to run: the counts error appeared in one
  round of six, and a duplicated label in one of three. The caller's
  fail-closed format checks (Task 18) are what make that tolerable: a slip
  pauses a round instead of passing silently.
- **Files the human must see go in `build/`, never `/tmp`.** An agent's temp
  directory is invisible to the human, which cost a round trip over a commit
  message. #893 made `build/` writable for exactly this.
- **Tasks that edit the same doctrine sections must run in sequence.** #883 was
  held until #879 merged rather than stacked on it, and that avoided rebasing a
  reviewed branch.
- **Check a sub-agent's proposed wording against the doctrine before relaying
  it.** The orchestrator passed on a docs-review wording ("accept the branch
  unreviewed on a cloud VM") that the doctrine does not allow. Repairing it
  took a round, and its own repair took a third.
- **The epic grew past its plan, and that was right.** About a third of the
  tasks were added during execution. The amendments kept the plan the
  authoritative driver, but #844 was written late. Amend the plan when the
  direction changes, not after the follow-on tasks are already running.

## §3 Compromises and tradeoffs

- **Fixes after the final round, listed as unreviewed, on five branches** (#827,
  #879, #883, #886, #893). The doctrine says `accept-branch` takes no fixes. The
  human directed it each time, because the round 3 fixes were narrow and leaving
  known gaps was worse. Five in a row is a pattern, not an exception, and it
  belongs in the agentic-review epic: either the doctrine allows it, or the
  round cap changes.
- **The reduced gate for 58db477.** It ran 6 runs, not 20, by the human's
  one-off ruling for a change to the contract's wording only. It is recorded in
  §10 and on #843, with no general rule.
- **The plan amendment #844 had no design review round.** The orchestrator
  handed it off as "plan text only". The doctrine would put a plan amendment
  through mode 1, or ask the human to waive it. It was a process slip.
- **Eight template-sync PRs have no gate records.** They are mechanical copies
  of one `CLAUDE.md` line, merged after the gate shipped.
- **The validated-version check cannot tell fork commits apart.** abf879f and
  58db477 both announce `v2.0.2-preview`. Keeping an unvalidated build out of
  reviews rests on the human holding reviews after each move (.github#846).
- **No reviews on cloud VMs.** A session without the delegate runs no review
  round, and Lima is the only supported environment.
- **Accepted risks, recorded privately as counts:** with the stray-write sweep
  gone, nothing checks for writes outside the worktree under review. One
  case-insensitive-filesystem edge case in #893 was rejected under the
  accidental-edit threat model.

## §4 New problems and opportunities

**Filed during the epic:**

| Ref | What | State |
|---|---|---|
| .github#813 | `vrg-security-record` cannot edit an advisory's text | intake |
| .github#814 | R1.11 from #827 | open |
| .github#822 | A paused round leaves its `mktemp` directory behind | intake |
| .github#823 | Routine for syncing the fork with upstream | intake |
| .github#824 | Shrink the state-directory guards as paad improves | deferred |
| .github#846 | Bump the preview version on every `fleet` move | intake |
| .github#850 | vergil-vm docs: `git` listed as apt-installed | intake |
| .github#851 | Fork: delegate contract slips (`Scope:`, out-of-tree fetch, `Files written:`) | intake |
| vergil-vm#377 | `vrg-validate` skips template and manifest checks | open |

**Problems surfaced:**

- **The delegate's contract adherence** varies from run to run (.github#851).
- **The doctrine's security procedure has grown intricate**, with an exit, a
  precedence or a kept file for every edge case. The agentic-review epic should
  ask what can be removed, not only what to add.
- **The routing of paad's findings by symbol** holds plain fixes behind the
  security hold-back. Noted for the paad author on #735.

**The unaddressed-findings sweep:** 45 tasks swept.

- **With summaries (15):** docs#79, vergil-claude-plugin#820, #827, #829, #849,
  #853, #879, #883, #886 and #893, vergil-tooling#3824, #3860 and #3900, and
  vergil-vm#376 and #385.
- **Swept from round comments only:** none.
- **No gate records (30):**
  - 13 operational or fork tasks;
  - 6 delivered by another task's PR, or declined;
  - .github#700 and #764, which have mode 1 records instead;
  - .github#844 (see §3);
  - the 8 template syncs (see §3).
- **PR reviews:** all 26 PRs have none.
- **Every `file` ruling's issue exists.**

Untracked items and their outcomes (ruled by the human):

1. **Deferrals that #858 left open when it closed:**
   - #829 R4.2: recording and deleting kept rounds across sessions;
   - #849 R2.13: crash-safe state for the act path;
   - #853's two items: where the security details of epic-implement's own items
     and of retrospective items go, and a kept-file record for a failed
     rulings-only details file.

   #858 closed on the sweep's removal, which none of these depend on. **Filed:**
   .github#862.
2. **#829 R4.9:** the "only your own files" rule in §1.5 was an out-of-scope
   addition with no recorded ruling. **Dropped:** it shipped through review and
   merge, so it is accepted.
3. **The held upstream offer:** no issue tracked making it at the integration
   milestone. **Filed:** .github#863. It carries the material collected on #735,
   the two design requests from spec §8, and the §4.9 and §4.10 material. It also
   flags that the hold cites "planning #468", an unrelated issue.
4. **"Review Focus 5 never exercised"** (a re-dispatch with no ruling reports
   the pause and stops). **Dropped:** epic-implement's own rule covers it, since
   a paused task re-dispatched with no ruling stops again.
5. **Housekeeping:**
   - vergil-claude-plugin#857 was still open, though #893 delivered it.
     **Closed** as delivered.
   - GitHub shows vergil-containers#735 as closing .github#726, through a wrong
     `Closes` line in an unrelated PR. Noted, no action.

Security-class (counts only):

- **The #843 observation** was about paad's severity grading of a seeded
  finding, not a weakness of ours. It went to the upstream material (.github#863).
- **The deferrals to #858** came to 4 distinct items: 5 from #849, one of them
  reported twice, plus a partial from #853. The #853 partial is the
  details-location item in .github#862. Of the four, 3 were recorded privately,
  tracked by .github#862, and 1 was rejected as an accepted risk.

## §5 What's next

- **The agentic-review epic**, in planning. It rebases the fork on upstream and
  rewrites paad-policy for the permanent fork. It inherits the open questions:
  - fixes after the final round, and the round cap;
  - simplifying the security procedure;
  - the delegate contract slips (.github#851);
  - a preview-version counter (.github#846).
- **The upstream offer to the paad author**, at the integration milestone
  (.github#863).
- **Fork syncing** (.github#823), first to pick up the author's patch for the
  issue vergil reported.
- **Shrinking the state-directory guards** (.github#824), as paad learns to
  confine its writes.

## Appendix B — Extended metrics

Security found by agentic review, by task (counts only):

| Task | Found | Fixed | Other outcomes |
|---|---|---|---|
| vergil-claude-plugin#827 | 27 | 27 | 3 held back, ended by approved fixes |
| vergil-claude-plugin#829 | 22 | 19 (+1 after a targeted review) | 1 recorded, 2 repeats of it |
| vergil-claude-plugin#849 | 21 | 12 | 2 declined, 5 deferred to #858, 2 ended by earlier rulings |
| vergil-claude-plugin#853 | 9 | 9 | 1 held back and ended; 1 fix partly deferred to #858 |
| vergil-claude-plugin#883 | 6 | 4 | 2 known under earlier rulings |
| vergil-claude-plugin#893 | 7 | 4 | 1 rejected, 2 ended by earlier rulings |

Design review: 29 found in planning (#698) and 38 in the amendment (#764); 37 of
those 38 were accepted.

Delegate rounds after Task 17: about 4–7 minutes and 90–125k tokens per
agentic-review round.
