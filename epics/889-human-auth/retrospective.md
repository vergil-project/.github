# Retrospective: authenticated human replies on task issues

**Epic:** [vergil-project/.github#889](https://github.com/vergil-project/.github/issues/889)
**Planning issue:** [vergil-project/.github#879](https://github.com/vergil-project/.github/issues/879)
**Spec:** [spec.md](spec.md) · **Plan:** [plan.md](plan.md)

## 0. At a glance

We set out to give a review decision a channel the agent can trust. When the human decides something about a task's change, the decision should reach the agent as an authenticated comment the human posted on the task issue, not as text another agent passes on. The epic built the tools for that channel in vergil-tooling and changed no skill; the merge-review epic ([#881](https://github.com/vergil-project/.github/issues/881)) adopts them.

All of it shipped:

- **`vrg-ruling`, the agent's tool.** `request` posts the request on the issue. `draft` pushes the agent's draft reply to the relay ref `refs/vergil/ruling-draft/<issue>/<request ID>`. `read [--json]` reads and checks the reply, and `respond` posts the response recording both comment IDs.
- **`vrg-ruling-reply`, the human's tool.** It runs in the human's own shell and only as a `human` login. It posts the draft from the relay ref, or the human's `--response-text`, after showing the exact reply and asking for confirmation.
- **Fatal checks.** An edited or deleted request or reply is fatal, and so is a trusted comment out of turn, a request by the wrong author, a bad marker or a reply that isn't plain text. Each gives exit 4 and names its check (`edited`, `deleted`, `turn`, `request-author`, `marker`, `reply-text`).
- **The `build/tmp/` convention.** Everything under `build/tmp/` is deletable scratch. It is documented, although these tools don't use it.

The tools were proven end to end on real issues from a Lima VM and from a cloud VM.

| PR | Repo | Task | What it did | Merged | Release |
|---|---|---|---|---|---|
| [#899](https://github.com/vergil-project/.github/pull/899) | .github | #890 | Spec and plan | 2026-11-04 | n/a |
| [#3937](https://github.com/vergil-project/vergil-tooling/pull/3937) | vergil-tooling | #3923 | The shared `lib/rulings` library: identity, comment reads, the marker, the turn checks, post-once | 2026-11-05 | 2.1.286 |
| [#3938](https://github.com/vergil-project/vergil-tooling/pull/3938) | vergil-tooling | #3924 | `vrg-ruling request`, `read` and `respond` | 2026-11-05 | 2.1.286 |
| [#3945](https://github.com/vergil-project/vergil-tooling/pull/3945) | vergil-tooling | #3925 | The draft relay ref, `vrg-ruling draft`, and `read --json` (folded in from #881) | 2026-11-06 | 2.1.286 |
| [#3946](https://github.com/vergil-project/vergil-tooling/pull/3946) | vergil-tooling | #3926 | `vrg-ruling-reply`, the human's tool | 2026-11-06 | 2.1.286 |
| [#3948](https://github.com/vergil-project/vergil-tooling/pull/3948) | vergil-tooling | #3927 | The Human Replies guide and the `build/tmp/` convention | 2026-11-06 | 2.1.286 |
| [#3967](https://github.com/vergil-project/vergil-tooling/pull/3967) | vergil-tooling | #3962 | Quoted the issue ref in the commands `vrg-ruling` prints, for zsh | 2026-11-06 | not yet released |
| [#3969](https://github.com/vergil-project/vergil-tooling/pull/3969) | vergil-tooling | #3917 | Documentation review: listed the two tools in the Script Reference index | 2026-11-07 | not yet released |
| [#87](https://github.com/vergil-project/docs/pull/87) | docs | #86 | Listed the two tools on the Tools & skills page | 2026-11-07 | n/a |
| [#3972](https://github.com/vergil-project/vergil-tooling/pull/3972) | vergil-tooling | #3968 | Completed the Script Reference index and the list of tools that bypass the wrapper | 2026-11-07 | not yet released |

**Operational tasks:** both closed with `Outcome: SUCCESS`.

| Task | Issue | What it did |
|---|---|---|
| Deployment | vergil-tooling#3918 | Installed 2.1.286 and checked both tools' `--help` in a Lima VM and on the host |
| Validation | vergil-tooling#3919 | Ran spec §12's cases on throwaway issues: a full round from a Lima VM and from a cloud VM, every fatal case, and a reply deleted before `read` |

**Counts:** 21 tasks under the epic.

- 10 closed by a merged PR. That includes #3962 (filed during validation), docs#86 (spawned by the documentation review) and #3968 (filed from it).
- 2 operational.
- 8 throwaway issues for the validation (vergil-tooling#3955 to #3961 and #3963).
- This retrospective.

**Repos touched:** vergil-project/.github, vergil-tooling and docs.

**Releases cut:** vergil-tooling 2.1.286. #3967, #3969 and #3972 are on `develop` and ride the next release.

**Span:** the planning issue opened 2026-11-03 13:00 UTC and the epic was created at 21:02. The last task closed 2026-11-07 12:19, about four days later.

## 1. How the plan evolved

The plan has no evolution log. This account comes from the execution record: the task comments, the PRs and the epic-implement session.

**The order held.** Tasks 1 to 5 ran in series, as the plan's interfaces required, then the deployment, the validation, the documentation review and this retrospective. All five planned code tasks reached `report-ready` on their first implementation. None needed rework after the human's review.

**The session restarted on the right skills.** The first Task 1 dispatch ran on a cached plugin (2.1.56) that still had the agentic-review gate. The human had it stopped, and the session restarted on 2.1.59. The re-run kept the three draft files the first run had left.

**A scope addition from #881.** While Task 3 was frozen at `report-ready`, the merge-review epic asked for `vrg-ruling read --json`, so a reviewer can take the reply ID without parsing stderr. By the human's ruling the branch was unfrozen and the flag folded into #3925, with its tests and docs and notes for #3927 and #3919.

**Task 3 failed CI's semgrep at merge.** `security / semgrep` flagged the SHA-1 that checks a blob's content against its git blob ID. That is an integrity check, not a security use, since git IDs are SHA-1 by definition. Local `vrg-validate` doesn't run semgrep, so the failure showed only at `vrg-submit-pr`. By the human's ruling it was suppressed in the source. Getting the comment right took three tries:

- *ERA001.* Ruff's commented-out-code rule flagged a bare `# nosemgrep: <rule>` line.
- *Placement.* The reworded comment sat two lines above the call, and semgrep only honours the flagged line or the line directly above it.
- *The fix.* The reason line goes first, then the `nosemgrep` line with the words "(git ID)" appended, directly above the call.

**The epic body had drifted from the spec.** Spec pushback round 1 removed `--response-file` from the human tool, but the epic body still described it. The Task 4 brief copied the stale clause from the body. The sub-agent rightly followed the plan and flagged the gap. The human checked that #881 depends only on `draft` and `vrg-ruling-reply`, and the epic body was corrected rather than the tool extended.

**#3927 did not auto-close.** PR #3948's `Closes #3927` linkage did not close the issue, so it was closed by hand. This is the same symptom #885 saw ([#901](https://github.com/vergil-project/.github/issues/901)).

**The deployment** waited on the human's 2.1.286 release. The agent checked the Lima VM, and the human attested the host's `--help` output.

**The validation ran across two sessions and nine issues.**

- *Round A, from a Lima VM, on #3955.* It covered a request, a draft and the prompt. While the prompt waited, the agent force-pushed a second, different draft. The posted reply held exactly the first draft's 48 bytes, and the draft ref was deleted. Then `read`, `read --json` and `respond`, with a re-run of `respond` posting nothing. The logins and comment IDs matched what the checks expect.
- *The fatal cases, one throwaway issue each (#3956 to #3961).* A fatal check leaves its request open for good, so each case needed its own issue. All six gave exit 4 with the right check.
- *A reply deleted before `read`* gave exit 3, as specified.
- *Round B, from a cloud VM, on #3963,* ran in a separate session.
- *A waived case.* "A deleted untrusted comment is not fatal" was not run. The human has no account outside the trusted-author list to post one with, so the case was waived by the human's ruling.
- *A zsh bug.* The human's shell is zsh with `EXTENDED_GLOB`, where `#` is a glob operator. The command `draft` printed failed when pasted, with `zsh: no matches found`. By the human's ruling it was fixed inside the epic as #3962. The same flaw in other tools' printed commands went to #3966, outside the epic.

**The documentation review** found one gap in its own repo: the Script Reference index didn't list the two tools. It spawned docs#86. Two older doc gaps it found were filed by the human's ruling as #3968 and fixed inside the epic, because they were small and docs-only.

## 2. Lessons learned

- **The plan, not the epic body, is what a brief should quote.** The epic body is written once at creation and design review doesn't revise it. Briefs built from it can carry decisions that review has reversed. The epic-implement skill already names the plan as authoritative. This epic showed that a brief that quotes the body still lets a stale clause through.
- **Local validation is not the whole gate.** `vrg-validate` runs neither semgrep nor the strict docs build, and CI runs both. A finding from either shows up only after `vrg-submit-pr`. In this epic that cost one extra round trip, on #3945.
- **Paste-ready commands must survive the human's real shell.** Unit tests checked the exact printed text, but not that zsh with `EXTENDED_GLOB` would accept it. Only the live validation found it. Single-quoting every ref is cheap and safe, because the ref grammar admits no quote.
- **One throwaway issue per fatal case was the right shape.** A fatal check is permanent by design. Isolating the cases kept every result clean and every comment as evidence.
- **Validating a trust boundary needs an identity on each side of it.** The untrusted-author case couldn't run live with only trusted accounts. A future epic with a similar check should plan for a test account, or say up front that the case is unit-tested only.
- **Front-loaded review held up again.** Seven design-review rounds (two spec, two plan, three alignment) settled the protocol before any code. The only post-plan design change came from outside (#881's `read --json`), and it fit the existing interfaces.
- **Check claims about coverage before posting them.** One result comment on #3919 stated that unit tests covered the deleted-untrusted-comment case. They didn't, and the comment was corrected. A cross-session result also wrongly said no fix task had been filed, and it was corrected with a pointer to #3962.

## 3. Compromises and tradeoffs

- **One spec case was not validated live.** A deleted untrusted comment is not fatal: by design it lies outside the turn checks. But neither a live run nor a unit test exercises the deleted case; the nearest test covers an untrusted comment that is present. The unit test is [vergil-tooling#3974](https://github.com/vergil-project/vergil-tooling/issues/3974).
- **A semgrep rule is suppressed in the source.** The SHA-1 in `draft_ref._blob_text` carries a `nosemgrep` comment that gives its reason. That was the human's choice over a project-wide exclusion. It keeps the rule live everywhere else.
- **`--response-file` was dropped.** Cloud VMs have no path shared with the host, so the draft travels as a git ref instead. A human who wants to write a long reply by hand has `--response-text` or the agent's draft. There is no file input.
- **The fix #3962 is unreleased.** Until the next vergil-tooling release, the deployed `vrg-ruling draft` still prints the issue ref unquoted, and a zsh user with `EXTENDED_GLOB` has to quote it by hand.
- **No skill uses the tools yet.** Until #881 lands, review decisions are still ruled in the terminal and written onto the issue by the agent.
- **No automated code review.** As with #885, no task ran a review gate; correctness rested on the tests, the validation and the human's PR review.
- **Twenty draft security advisories** were retargeted to this epic's tasks (Task 1: 5, Task 2: 7, Task 3: 1, Task 4: 4, Task 5: 2, validation: 1). The tasks that address them are closed. Closing the advisories is the human's call, because the agent cannot.

## 4. New problems and opportunities

| Item | Where it went |
|---|---|
| Other tools print paste-ready commands with an unquoted issue ref (`vrg-epic-move`, `vrg-security-record retarget`) | [vergil-tooling#3966](https://github.com/vergil-project/vergil-tooling/issues/3966), under vergil-tooling's ad-hoc epic |
| The Script Reference index left out tools that have reference pages, and the list of tools that bypass the wrapper was incomplete | [vergil-tooling#3968](https://github.com/vergil-project/vergil-tooling/issues/3968), fixed in this epic |
| GitHub skipped the auto-close for vergil-tooling#3927 | Found: [.github#901](https://github.com/vergil-project/.github/issues/901) (intake) already tracks it |
| Let the human work in a cloud VM as their own identity (from spec pushback S1.1) | [.github#897](https://github.com/vergil-project/.github/issues/897) (intake) |
| Post every human ruling, not only review decisions, as an authenticated comment from the human | [.github#892](https://github.com/vergil-project/.github/issues/892) (intake) |
| `github-advanced-security`'s COMMENTED review on #3945 | Found: it is the semgrep SHA-1 alert, suppressed in the source by the human's ruling (§1) |
| No test covers a deleted untrusted comment | Filed by the human's ruling: [vergil-tooling#3974](https://github.com/vergil-project/vergil-tooling/issues/3974), a unit test at least |
| `vrg-validate` doesn't run semgrep, so CI's `security / semgrep` failures show only after `vrg-submit-pr` | Filed by the human's ruling: [.github#914](https://github.com/vergil-project/.github/issues/914) (intake). It also reviews every other CI PR gate local validation skips |
| The epic body drifts from the spec when design review changes it | Filed by the human's ruling: [.github#915](https://github.com/vergil-project/.github/issues/915) (intake), a small `epic-create` change to check the body whenever the spec or plan is published or updated |
| Two "choices to review" PR #3937 flagged (a per-line `noqa N818`; which comment `read_reply` names in a turn failure) | Dropped by the human: they were raised in the PR the human reviewed and merged |
| The 20 draft advisories | Still drafts; their tasks are closed. Closing them is the human's call |

**The unaddressed-findings sweep.**

- *Tasks swept:* 20, every task but this one, plus the epic and the planning issue. Every comment is by a trusted author.
- *Ruling lists:* every `Awaiting a ruling` item on the tasks is closed: #3926's `--response-file` gap, #3919's unquoted ref (#3962), #3962's other tools (#3966), and #3917's two doc gaps (#3968). The validation's waived case is recorded three times on #3919.
- *Filed without a ruling:* none.
- *PR reviews:* none of the ten merged PRs has a conversation comment. The one review is `github-advanced-security`'s on #3945, covered above.
- *The planning issue:* seven rounds. Every rejected finding carries a reason, and its follow-ups (#897, #892) exist.
- *Security items:* 14 raised in design review, 13 accepted. Together with the 7 the epic started with, they make the 20 advisories above.
- *Untracked problems:* four found. Three were filed by the human's ruling (#3974, #914, #915) and one was dropped.

## 5. What's next

- **Merge review:** [vergil-project/.github#881](https://github.com/vergil-project/.github/issues/881) adopts `vrg-ruling` and `vrg-ruling-reply` as the channel for review decisions.
- **[#892](https://github.com/vergil-project/.github/issues/892):** extend the channel to every human ruling.
- **[#897](https://github.com/vergil-project/.github/issues/897):** the human working in a cloud VM as their own identity.
- **[vergil-tooling#3966](https://github.com/vergil-project/vergil-tooling/issues/3966):** quote refs in the other tools' printed commands.
- **[vergil-tooling#3974](https://github.com/vergil-project/vergil-tooling/issues/3974):** the missing unit test.
- **[#914](https://github.com/vergil-project/.github/issues/914) and [#915](https://github.com/vergil-project/.github/issues/915):** in intake.
- **The next vergil-tooling release** carries #3967, #3969 and #3972.

## Appendix A. Operational notes

**The human's side of a round, as validated:**

```text
vrg-ruling-reply --issue 'owner/repo#N' --request <request ID>
```

The command shows the draft fetched from `refs/vergil/ruling-draft/<N>/<request ID>`, with its byte count and sha256, and asks for confirmation. It posts exactly what it showed, even if the ref moved meanwhile, and then deletes the ref.

**Exit codes:** 0 ok, 1 refusal or error, 2 usage, 3 no reply yet, 4 fatal check.

**Validation gotchas:**

- Quote the issue ref in zsh (fixed in #3967, unreleased).
- A reply posted from the GitHub web page often picks up non-ASCII typography, such as an em dash. `reply-text` rejects it. That is what the #3961 case tests, and it is why replies go through `vrg-ruling-reply`.
- A fatal check leaves the request open forever, so use a fresh issue for each fatal case.
- `vrg-gh api` and `vrg-gh run download` are denied, and `vrg-security-record` has no close command. Closing advisories and reading CI artifacts are human actions.
