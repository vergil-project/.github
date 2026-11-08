# merge-review: retrospective

- **Epic:** vergil-project/.github#881
- **Spec:** [spec.md](spec.md)
- **Plan:** [plan.md](plan.md)
- **Planning issue:** vergil-project/.github#878
- **Span:** 2026-11-03 (epic opened) to 2026-11-08 (validation closed)

## 0. At a glance

We set out to replace `paad:agentic-review`, which asked "what is wrong with this code?" and never converged, with `merge-review`, which asks "is this change good enough to merge?". The new loop has three parts:

- one review by a read-only reviewer, at most one fix and one check;
- every stop goes to the human through `vrg-ruling`;
- every run ends with an eligibility line, saying whether the change could merge without a human.

The design was the prerequisite for agents submitting and merging their own PRs.

All of it shipped. The protocol spec, the result check, the change classifier and the request builder are in vergil-tooling. The reviewer agent and the skill are in vergil-claude-plugin, wired into `issue-implement` and `epic-implement`. Doc pages are in vergil-tooling's site and in the `docs` repo.

It then ran on the fleet's real work rather than on seeded scenarios. In the 41 live runs audited, it:

- passed clean changes in one step;
- fixed and checked defects;
- stopped for the human three times.

Live use found nine gaps (N0.1–N0.9), and each was filed and fixed inside the epic.

| | |
|---|---|
| Tasks | 30, plus this retrospective: 25 shipped by PR, 2 operational (deploy and validate), 3 closed without a PR (duplicate or superseded) |
| PRs merged | 25 |
| Repos touched | vergil-tooling, vergil-claude-plugin, docs, .github |
| Releases cut during the epic | vergil-tooling 2.1.285–2.1.289, vergil-claude-plugin 2.1.58–2.1.62 (not all of their content is this epic's) |
| Gaps found in live use and fixed | 9 (N0.1–N0.9, recorded on #881) |

| Task | PR | Merged | What it did |
|---|---|---|---|
| .github#882 | .github#905 | 11-05 | Spec and plan |
| .github#906 | .github#907 | 11-06 | Aligned the plan with `vrg-ruling`'s merged interface (`read --json`, respond's fatal exit) |
| vergil-tooling#3939 | #3975 | 11-07 | The merge-review protocol spec |
| vergil-tooling#3940 | #3981 | 11-07 | `vrg-merge-review-check`, the result check |
| vergil-tooling#3982 | #3999 | 11-07 | Named the marker-prefix refusal in check rules 6 and 7 |
| vergil-tooling#3941 | #3993 | 11-07 | `vrg-change-class` and the `docs-paths` key |
| vergil-tooling#3942 | #4005 | 11-07 | The validate record and `vrg-merge-review-request` |
| vergil-tooling#4006 | #4007 | 11-07 | Refuse a `--fix-base` that is not the previous step's head |
| vergil-claude-plugin#928 | #933 | 11-07 | The read-only `merge-reviewer` agent |
| vergil-claude-plugin#929 | #942 | 11-07 | The `merge-review` skill |
| vergil-claude-plugin#930 | #958 | 11-07 | Ran merge-review from `issue-implement`; resume through `epic-implement` |
| vergil-claude-plugin#957 | #963 | 11-07 | Left deleting the private directory to `issue-implement` |
| vergil-tooling#3943 | — | 11-07 | Deploy: released and installed tooling 2.1.287 and plugin 2.1.60 |
| vergil-tooling#3916 | #4019 | 11-07 | Documentation review: the Merge Review guide and site links |
| docs#89 | docs#90 | 11-07 | Merge review in the management docs |
| vergil-claude-plugin#969 | #973 | 11-07 | N0.3: made the non-review gate hard to skip |
| vergil-tooling#4015 | #4023 | 11-07 | N0.2: vergil-tooling's `docs-paths` |
| vergil-tooling#4014 | #4024 | 11-07 | N0.1: carried issue comments in the request |
| vergil-tooling#4020 | #4031 | 11-07 | N0.5: `vrg-scope-change`, a human-only scope-change command |
| vergil-tooling#4028 | #4032 | 11-07 | N0.6: the builder fetches the human's unedited comments itself |
| vergil-claude-plugin#970 | #976 | 11-07 | N0.1/N0.5: the human's comments reach the reviewer as scope amendments |
| vergil-claude-plugin#974 | #977 | 11-07 | N0.5: agents draft scope changes; the human posts them |
| vergil-tooling#4030 | #4034 | 11-07 | N0.7: refuse a clone that is not the issue's repository |
| vergil-claude-plugin#982 | #984 | 11-08 | N0.8: the skill stops writing the body and task files |
| vergil-claude-plugin#983 | #985 | 11-08 | N0.9: deferred-review drafts scope changes |
| vergil-tooling#4041 | #4047 | 11-08 | N0.8: the builder fetches the issue body and epic tasks itself |
| vergil-tooling#3944 | — | 11-08 | Validate: audit of 41 live runs, `Outcome: SUCCESS` |

Closed without a PR: vergil-claude-plugin#968 (covered by #845), vergil-claude-plugin#971 (duplicate of #970), and vergil-tooling#4018 (superseded by #4028, ruling N0.6).

## 1. How the plan evolved

The plan's seven implementation tasks landed close to as written. They merged in dependency order within one day, 2026-11-07, Their gates filed four follow-ups (vergil-tooling#3982, #4006 and #4009, and vergil-claude-plugin#957) and declined a handful of other items. The plan carries no "Evolution during execution" log. The evolution is recorded as rulings on #881 and on each task.

Two changes reshaped the epic after planning:

- **The interface amendment (#906).** `vrg-ruling` merged with a slightly different interface than the plan assumed: the reply ID comes from `read --json`, and `respond` can exit fatally. The plan was amended before any implementation started, so no task built against the stale interface.
- **Validation in production instead of a scratch repo (#3944).** The plan called for a dedicated scratch repo with sixteen seeded scenarios. Once the deploy task had installed the tools, the human replaced that with live use: every task any agent implemented ran merge-review for real, and the check became an after-the-fact audit. This was the most consequential decision of the epic. Seeded scenarios would have proven that the designed paths work. Live use instead found what the design had missed, and turned the second half of the epic into a find-and-fix loop.

That loop produced N0.1–N0.9, each a ruling recorded on #881. They fall into three themes:

- **What the reviewer is told the task is.**
  - The reviewer saw only the issue body, not the comments that had narrowed its scope (N0.1, from #3870, where the human had to rule `dismiss` on a defect the reviewer raised against scope that a comment had removed).
  - Only the human's own unedited comments may amend scope (N0.5, N0.6).
  - The builder must fetch the comments, the body and the epic's tasks itself rather than read files the agent wrote (N0.6, N0.8; on #974 the agent supplied the epic's body in place of the task's).
  - The clone must match the issue (N0.7).

  The principle that settled all of these: **what the reviewer is told should never be chosen by the agent whose work is under review.**
- **Who may change scope.** Scope changes had been improvised as agent comments labelled "maintainer". They are now posted by the human through `vrg-scope-change`, from a draft the agent prepares (N0.5, N0.9).
- **The non-review gate and configuration.** A run went to review with non-review items unsettled (N0.3), and vergil-tooling's doc pages were classed as code (N0.2).

## 2. Lessons learned

- **Validate in production as soon as the tool is safe to run there.** Sixteen seeded scenarios were planned. Live use exercised fewer of the designed paths but found nine real gaps that no seeded scenario would have, because each came from how agents and the human actually used the tool. The cost was that rare paths stayed untested (§3). Next time, plan validation as "deploy, then audit live use", with seeded scenarios only for paths that matter and are unlikely to occur naturally.
- **One review, one fix, one check converges.** In 41 audited runs, nothing came close to agentic-review's endless rounds:
  - 32 ended `eligible` with no fix;
  - 6 fixed once and passed the check;
  - 3 stopped for the human, who settled each in one reply.

  The spec's bet, that a merge-quality question has a bottom where a "what is wrong" question does not, held.
- **Trust boundaries belong in tooling, not in skill prose.** Most of N0.1–N0.9 were cases where the skill told the agent to do the right thing and an agent did something else, or could have. Each fix moved the decision into the builder (fetch it yourself, refuse a mismatched clone, accept only the human's login). Prose describes; tooling enforces.
- **A tooling change that removes an interface needs a release plan.** #4041 removed two builder flags that the installed skill still passed. Between the vergil-tooling and vergil-claude-plugin releases, every merge review on the fleet would have failed. It was bridged by having five runs follow develop's skill text. Either a deprecation window (accept the old flags, warning, for one release) or a strict back-to-back release would have avoided the gap.
- **The human reply loop works; the friction is copy-and-paste.** The agent drafts the `vrg-ruling-reply` command, and the human reads it, corrects it through the agent when it is wrong, and runs it. That makes the human's answer authoritative even when it differs from the recommendation. Moving the command text between the agent and the shell is the main remaining friction.

## 3. Compromises and tradeoffs

- **Most designed paths were never exercised live.** These scenarios rest on unit tests and the skill's text only:
  - design question with a ruling round;
  - `check-failed`;
  - `too-many-blockers`;
  - any security finding inside merge review (defer, fix not fixed);
  - out-of-scope revert and split;
  - pre-existing ruled `file`;
  - unclear reply;
  - B1 rebase;
  - D1 fetch failure.

  The human ruled the live evidence enough to close the validation. The first real occurrence of each is its live test.
- **The override path was accepted on process, not observed as a distinct case.** No audited run posted a reply differing from the request's recommendation. The human judged the read-and-correct reply loop sufficient evidence that a differing answer is carried out.
- **The deploy precondition was attested, not probed.** The deploy task could not reach the host or the other VMs from inside its VM. Its `Outcome: SUCCESS` rests on the human's attestation that both were installed.
- **Five early runs handed off with non-review items pending.** They ran on plugin 2.1.60, before the N0.3 gate shipped in 2.1.61. Their items were ruled afterwards, outside the gate. One (#3839) went unruled until the validation audit found it.
- **Fixed security findings left open.** The security findings this epic recorded privately are fixed in code, but their advisories were still open when the epic closed. Closing them is a human action outside this epic.

## 4. New problems and opportunities

Filed during the epic:

- **.github#922:** `epic-implement` should route the container epics (ad-hoc, intake, planning, deferred) to their skills, so queue work runs under a skill's rules rather than an ad-hoc prompt.
- **.github#925:** try a tightened fleet-wide check against every repo's develop before releasing it. vergil-tooling#4042 widened shellcheck and broke CI in four repos, which needed emergency fixes: vergil-claude-plugin#988, vergil-actions#1102, vergil-vm#391 and logical-minds-foundry/mq-resiliency-lab-for-linux#1708.
- **.github#926:** GitHub App authentication fails intermittently with a 401.
- **.github#927:** `vrg-release --abandon`, a supported way to abandon an untagged release. Recovering the failed plugin 2.1.62 release needed raw `gh` and `git` and a manual prune.
- **vergil-tooling#4009 (open):** a `vrg` command to fetch merge-review design sources into the private directory. It was filed at the merge-review skill's gate (vergil-claude-plugin#929) and is the last skill-side input still assembled by the agent.
- **.github#929 (open):** merge review settles non-review items on agent-written `Human ruling:` lines. Every ruling line in this epic was posted by the agent login, recording the human's terminal decisions.

**Unaddressed-findings sweep.** 32 issues were swept: the 30 children other than this one, the epic and the planning issue. Every `Awaiting a ruling` item has a closing ruling, and every `file` ruling's issue exists. The one security candidate was checked: its counts and its record are consistent. The 25 merged PRs carry no reviews or conversation comments.

Untracked problems, with the human's rulings:

- **C1:** port the paad pushback and alignment rounds to the new protocol, with the reviewer as a pure reader, and remove the doctrine's paad workarounds. The planning issue deferred this to a later epic. **Filed** as vergil-project/.github#930.
- **C2:** review inside the PR process, as defence in depth. The planning issue named it out of scope. **Filed** as vergil-project/.github#931.
- **C3:** a breaking tooling interface change needs a compatible transition (§2). **Filed** as vergil-project/.github#932.
- **C4:** a deploy task inside a VM cannot probe the host or the other VMs, so deploy preconditions there can only be attested. **Found:** recorded as a comment on the related vergil-project/.github#897 rather than filed.

## 5. What's next

- **Agent submit and merge.** This was the prerequisite the spec named. With `merge-review` in place and its eligibility line on every handoff, letting agents submit and merge `eligible` PRs is the next step. It needs its own planning issue.
- The intake items in §4 wait for the next intake review.
