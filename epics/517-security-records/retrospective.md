# Security Records — Retrospective

Epic: [vergil-project/.github#517](https://github.com/vergil-project/.github/issues/517) ·
Spec: [`spec.md`](spec.md) · Plan: [`plan.md`](plan.md)

## 0. At a glance

**What we set out to do.** paad 2.0 keeps security findings out of public places, in a
gitignored folder on one machine that is easily lost. We had been putting security problems in
ordinary public issues. The epic set out to give findings a private, durable home and keep the
public issue neutral:

- a private draft repository security advisory in the org's `.github`, paired with a neutral
  `security`-labelled intake issue, each pointing at the other;
- one command to create, attach, resume, list, show and retarget a pair (`vrg-security-record`);
- the scheduled sweep keeps each advisory's state in step with its issues and reports broken
  pairs;
- intake routing and the planning queue keep the link intact, and `epic-create` hands a planning
  issue's advisory to its fix tasks;
- a one-time audit of past public issues for security disclosures.

**What shipped.** All of it, plus one unplanned fix and five extra doc tasks:

- vergil-tooling **v2.1.270** added the core library and the labels (T2).
- **v2.1.271** added the CLI, the sweep pass, the intake-route link and the planning hold
  (T3–T6).
- **v2.1.273** added the docs (T8) and the retarget re-run fix (#3702).
- vergil-claude-plugin **v2.1.43** added the `epic-create` handoff (T7).
- The end-to-end check took a test pair through two retargets and two human-dispatched sweeps.
- The past-issues audit covered 3,566 issues in 9 repos, created 1 record and labelled 13 issues.

| Repo | PR | Task | What it did |
|---|---|---|---|
| .github | [#526](https://github.com/vergil-project/.github/pull/526) | #518 | Spec and plan |
| vergil-tooling | [#3685](https://github.com/vergil-project/vergil-tooling/pull/3685) | #3677 (T2) | `lib/security_records.py` and the `security` / `security-disclosed` labels |
| vergil-tooling | [#3693](https://github.com/vergil-project/vergil-tooling/pull/3693) | #3678 (T3) | `vrg-security-record`: create, attach, resume, list, show, retarget |
| vergil-tooling | [#3694](https://github.com/vergil-project/vergil-tooling/pull/3694) | #3679 (T4) | Security-records pass in `vrg-epic-audit --close` |
| vergil-tooling | [#3696](https://github.com/vergil-project/vergil-tooling/pull/3696) | #3680 (T5) | `vrg-intake-route` keeps the link on transfer, recreate and split |
| vergil-tooling | [#3697](https://github.com/vergil-project/vergil-tooling/pull/3697) | #3681 (T6) | `vrg-queue planning` holds an issue an advisory still tracks |
| vergil-claude-plugin | [#740](https://github.com/vergil-project/vergil-claude-plugin/pull/740) | #732 (T7) | `epic-create` hands the advisory to the fix tasks |
| vergil-tooling | [#3707](https://github.com/vergil-project/vergil-tooling/pull/3707) | #3682 (T8) | Security-records guide, CLI reference, App permissions |
| vergil-tooling | [#3708](https://github.com/vergil-project/vergil-tooling/pull/3708) | #3702 *(added)* | `retarget` finishes an interrupted run when re-run |
| vergil-tooling | [#3716](https://github.com/vergil-project/vergil-tooling/pull/3716) | #3671 | Documentation review: issue standards and `CLAUDE.md` |
| .github | [#545](https://github.com/vergil-project/.github/pull/545) | #544 *(spawned)* | `SECURITY.md` describes internal records; stale tool names fixed |
| vergil-claude-plugin | [#751](https://github.com/vergil-project/vergil-claude-plugin/pull/751) | #750 *(spawned)* | Skills catalogue notes the `epic-create` handoff |
| docs | [#65](https://github.com/vergil-project/docs/pull/65) | #64 *(spawned)* | Management pages cover security records |
| vergil-tooling | [#3717](https://github.com/vergil-project/vergil-tooling/pull/3717) | #3714 *(added)* | Security-boundaries register follows the records policy |
| vergil-tooling | [#3718](https://github.com/vergil-project/vergil-tooling/pull/3718) | #3715 *(added)* | `vrg-epic-audit` section in the CLI overview |

- **Repos touched by PRs (4):** .github, vergil-tooling, vergil-claude-plugin, docs. The audit
  also labelled issues in vergil-containers and vergil-actions.
- **Children (20):**
  - 14 planned: T1–T11 and three bookends (#518, #3671, #519).
  - 6 added during execution: the retarget fix (#3702), three doc tasks spawned by the docs
    review (#544, #750, docs#64), and two doc tasks the human chose to fix under the epic rather
    than backlog (#3714, #3715).
  - 4 operational: the probe .github#523 (T1, waived), deployment vergil-tooling#3683 (T9), the
    end-to-end check .github#524 (T10) and the audit .github#525 (T11).
- **PRs merged:** 15, including the spec/plan PR and excluding this one.
- **Releases cut (by the human):** vergil-tooling v2.1.270, v2.1.271 and v2.1.273 (v2.1.272, in
  between, held nothing from this epic); vergil-claude-plugin v2.1.43. The last three doc PRs
  (#3716–#3718) are on develop but not yet in a release.
- **Span:** epic opened 2026-10-26 18:56Z; the last task before this one (#3715) closed
  2026-10-27 12:19Z, about 17½ hours.

## 1. How the plan evolved

The plan has no "Evolution during execution" log, so this section is reconstructed from the issue
record and the session. T2–T8 shipped as planned. The deviations were the probe, one defect the
docs task found, the release order, and the size of the docs review.

- **The probe (T1) was waived.** T1 was meant to prove, before any code, that both App tokens
  could create, read, edit and close draft advisories. Running it needed hand-built tokens for
  two Apps, after first working out which of the two Apps the docs meant (the human had already
  changed the agent user App; the org automation App was changed during the walkthrough). The
  human judged that too costly, accepted the risk, and waived it. The waiver is recorded on #523
  as WAIVED, not PASS. That made the end-to-end check (T10) the first live test of the advisory
  API. It passed first time.
- **Writing the docs found a real defect in `retarget`.** Describing "what happens after a
  failure" showed that a re-run could not finish an interrupted retarget. The unmark step read
  its list from the tracking lines that the previous step had already rewritten, so a dropped
  issue kept its label and line, and the sweep would report it as broken every night. The human
  asked for it to be fixed under the epic rather than backlogged. #3702 now also finds dropped
  issues from the issues themselves (a REST listing, never search), only removes this advisory's
  line, and posts the move comment once. The docs PR was then updated to describe the fixed
  behaviour, and the two were submitted together.
- **Releases were interleaved with another epic.** The deferred-queue epic (#508) needed tooling
  releases at the same time. The human released merged security-records work early (v2.1.270,
  v2.1.271) alongside it, which was safe because nothing used the new code until the deploy task.
  The plugin release came last, and the deploy task (T9) was blocked once because v2.1.42 had
  been cut before T7 merged. The fix was simply another plugin release.
- **T9's sweep check needed a dispatched run.** The sweep precondition could not be shown from a
  scheduled run, because none had happened since v2.1.273. The human dispatched one, and its log
  showed v2.1.273 and the new summary line.
- **The docs review grew into six doc tasks.** The planned scope was the vergil-tooling pages.
  The review spawned one task each in .github, the plugin and the docs repo. The audit then
  flagged the public security-boundaries register, which described an open gap in prose. The
  review also found two older gaps unrelated to this epic. The human had both fixed under the
  epic (#3714, #3715). One of the two older gaps, the missing `deferred` label row, had already
  been fixed on develop by #3713 by the time #3715 ran.
- **Three rebase conflicts and one more at submit time.** T2, T5 and T6 conflicted with develop
  (the label count, a docstring, and the queue view next to the new deferred view). The docs
  review conflicted with #3713's queue restructure. Each was resolved by keeping both sides and
  re-validating; none needed a design change.

## 2. Lessons learned

- **Writing the reference docs is a design review.** The retarget defect survived the spec, the
  plan, the implementation and its tests, and was found only by having to explain the failure
  path in prose. Docs tasks that describe failure and recovery behaviour are worth scheduling
  before the release, not after.
- **A waived probe moves the risk, it does not remove it.** Waiving T1 was a reasonable call, but
  it meant every implementation task was built on an untested assumption about the advisory API.
  It held. Next time, a probe that is hard to run by hand is a sign it should be a small script
  run under each identity, not a checklist.
- **"Which App?" must be unambiguous in the docs.** The walkthrough of the App permission
  changes was confusing because the docs said "both Apps" where the human sees one user App and
  one org automation App. The guide now names each App by its pattern (`<username>-vergil-user`,
  `<org>-automation`) and says which tool needs which.
- **Name the plugin release in the deploy task's preconditions.** T9 named the change, not the
  version. A release cut minutes before the change merged looked like it satisfied the task.
  Checking that the installed skill text contains the change caught it.
- **Neutral wording has to reach existing docs, not just new issues.** The security-boundaries
  register was written before this policy and described an open gap in a public page. A
  policy like this needs a sweep of existing docs as well as existing issues.

## 3. Compromises & tradeoffs

- **The probe was waived (§1).** The human accepted that risk explicitly; T10 then exercised the
  API end to end.
- **One retarget case stays unreachable.** An issue whose `security` label was removed by hand
  while its advisory line stayed can be found again only through the advisory's old tracking
  lines. A re-run cannot reach it without search, which the design rules out because the index
  lags. The sweep reports it as a broken pair, so it is visible, not silent.
- **The one unfixed record will outlive the one-day target.** The spec sets a target of about a
  day to fix a recorded finding. The audit's one unfixed hit (vergil-tooling#3524, severity
  low) is blocked on epic .github#443, so it will stay open well beyond that. The target is
  policy, not enforced, and this record is a known exception.
- **No scrub of past issues.** The human chose audit-and-label only. The 13 labelled issues keep
  their original text; all were fixed or accepted, and the one unfixed issue now has a record.
- **The deploy step ran on an already-updated VM.** T9's `vrg-vm update` is a host command. The
  agent VM already had v2.1.273 and plugin v2.1.43, so the step was verified by the installed
  versions rather than run from the agent.
- **Labels were not provisioned ahead of use.** `security-disclosed` did not exist in any of the
  four repos the audit labelled, and was created from the registry at audit time. `security` is
  created on demand by `vrg-security-record`. Nothing broke, but the registry and the repos were
  out of step until something needed the label.

## 4. New problems & opportunities

| Item | Where it went |
|---|---|
| `retarget` could not finish an interrupted run | Fixed: vergil-tooling#3702 (v2.1.273) |
| Automation identity can be spoofed by ambient signals (found by the audit) | Private record GHSA-r5m3-gp2j-fqmh on vergil-tooling#3524; fix blocked on epic .github#443 |
| Public security-boundaries register described an open gap | Fixed: vergil-tooling#3714 |
| `SECURITY.md` named retired tools and a retired repo | Fixed: .github#544 |
| `cli-tools-overview.md` had no `vrg-epic-audit` section | Fixed: vergil-tooling#3715 |
| Umbrella #500's child B was hard to start: nested epics don't make it easy to see which tasks belong to which epic | Recorded on vergil-project/.github#88, with the suggestion of an epic of epics that has no implementation tasks of its own |
| Registry labels are not pushed to repos until first use | Logged here; not yet acted on |
| The sweep pass fails (exit 1) in an org whose automation App lacks the advisories permission or whose `.github` is private | Checked for logical-minds-foundry: its 2026-10-27 sweep ran v2.1.273 and the pass reported cleanly. mnemosys-project has no Epic Sweep workflow. Logged; nothing to act on now |
| SECURITY.md promises to publish an advisory after a fix, while internal records are never published by default | Reviewed: consistent, since publishing is a separate human decision. No action |

## 5. What's next

- Umbrella vergil-project/.github#500's children C and D build on this epic: wiring paad's
  output and the skills (`issue-implement`, `intake-capture`, and paad itself) to record findings
  with `vrg-security-record`.
- The fix for vergil-tooling#3524 under epic .github#443, which will close its record through
  the sweep.
- No follow-on brainstorm was opened from this epic.

## Appendix A — Operational notes

**Release and deploy order.** Tooling v2.1.270 (T2) and v2.1.271 (T3–T6), interleaved with the
deferred-queue epic's releases → plugin T7 merged → tooling v2.1.273 (T8 and #3702) → plugin
v2.1.43 → T9 verified in the agent VM and in a dispatched sweep → T10 → T11.

**Sweep runs used as evidence (vergil-project/.github).**

| Run | Purpose | Result |
|---|---|---|
| 40496334643 | T9: sweep installs the release | v2.1.273; `0 advisory state(s) synced, 0 broken pair(s)` |
| 40497118633 | T10: one test issue still open | Advisory stayed `draft`; no `PROBE-E2E` in the log |
| 40497553377 | T10: both test issues closed | `1 advisory state(s) synced`; advisory `closed`; no `PROBE-E2E` |

**End-to-end test pair.** GHSA-hgp4-f6fc-fjcg, tracked by .github#539 and #542, both closed. The
advisory is closed and stays as the record of the test.

**Past-issues audit (#525).** 9 public repos, 3,566 issues and 7,575 comments; 873 candidates
from the strong search terms, 75 read in full. 14 hits: 1 unfixed (record created) and 13
fixed or accepted (`security-disclosed`). No secrets to rotate. The search terms and
per-hit evidence stayed in the agent's private notes; the public outcome comment on #525 has
counts and refs only.
