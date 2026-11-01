# Plugin distribution via the floating `v2.1` tag — Retrospective

Epic: [vergil-project/.github#760](https://github.com/vergil-project/.github/issues/760) ·
Spec: [`spec.md`](spec.md) · Plan: [`plan.md`](plan.md)

## 0. At a glance

**What we set out to do.** Every consumer registered the `vergil-marketplace`
at `"ref": "main"`, so the deployed plugin was whatever `main` held and a
rollback needed a PR to `main`. The epic set out to:

- move the marketplace source to the floating `v2.1` tag, so that a deploy or
  a rollback is a tag move;
- protect the rolling tag as the deploy ref, with a tag ruleset and a
  branch-name ruleset;
- harden `vrg-promote` as the rollback tool;
- switch the whole fleet with no hand edits, through a shared, verified
  plugin refresh;
- prove it with a real rollback round trip and an end-to-end release.

**What shipped.** All of it, as specified, with one validation step replaced
by evidence (§3):

- vergil-tooling **v2.1.280**: the rolling-tag rulesets and the hardened
  `vrg-promote`.
- vergil-tooling **v2.1.281**: the shared marketplace refresh, `vrg-host
  plugins`, and `vrg-vm update` copying the host settings first.
- vergil-tooling **v2.1.282**: the rollback runbook and switching procedure,
  and the fix that limits the refresh to declared marketplaces.
- vergil-claude-plugin **v2.1.52**: the distribution docs, released through
  the normal flow as V2's end-to-end test.

The fleet (host and three VMs) follows `v2.1`. V2 rolled it back to 2.1.50,
forward to 2.1.51, and then a release moved it to 2.1.52 on its own.

| Repo | PR | Task | What it did |
|---|---|---|---|
| .github | [#771](https://github.com/vergil-project/.github/pull/771) | #761 | Spec and plan; epic #45's spec points to this one |
| .github | [#780](https://github.com/vergil-project/.github/pull/780) | #772 *(added)* | Spec §6 and Task 3 reconciled with V1: declare the ref, then re-add |
| vergil-tooling | [#3862](https://github.com/vergil-project/vergil-tooling/pull/3862) | #3848 (T1) | Rolling-tag tag ruleset and the rolling-tag branch-name ruleset |
| vergil-tooling | [#3863](https://github.com/vergil-project/vergil-tooling/pull/3863) | #3849 (T2) | `vrg-promote` refuses agents, checks its target and asks before moving a tag |
| vergil-tooling | [#3872](https://github.com/vergil-project/vergil-tooling/pull/3872) | #3851 (T3) | Shared, verified marketplace refresh; `vrg-host plugins`; `vrg-vm` refuses agents |
| vergil-tooling | [#3881](https://github.com/vergil-project/vergil-tooling/pull/3881) | #3879 *(added)* | The refresh updates only declared marketplaces, by name, and shows a failed command's stdout |
| vergil-tooling | [#3880](https://github.com/vergil-project/vergil-tooling/pull/3880) | #3853 (T5) | Rollback runbook and switching procedure, with four follow-ups folded in |
| vergil-claude-plugin | [#863](https://github.com/vergil-project/vergil-claude-plugin/pull/863) | #851 (T4) | Distribution docs follow the floating `v2.1` tag |
| vergil-claude-plugin | [#868](https://github.com/vergil-project/vergil-claude-plugin/pull/868) | #866 *(added)* | The release install step runs `vrg-host plugins` and `vrg-vm update --all` |
| vergil-claude-plugin | [#871](https://github.com/vergil-project/vergil-claude-plugin/pull/871) | #861 *(added)* | Install docs warn that changing a registered `ref` needs the switching procedure |
| vergil-claude-plugin | [#872](https://github.com/vergil-project/vergil-claude-plugin/pull/872) | #860 *(added)* | README Update and Verify cover a `v2.1` rollback |
| vergil-tooling | [#3889](https://github.com/vergil-project/vergil-tooling/pull/3889) | #3886 *(added)* | Consumer guides stop deferring the in-session refresh to the plugin README |
| vergil-claude-plugin | [#873](https://github.com/vergil-project/vergil-claude-plugin/pull/873) | #850 (doc review) | Docs-site sweep for the tag, rollback and verified refresh; #869 folded in |
| vergil-tooling | [#3891](https://github.com/vergil-project/vergil-tooling/pull/3891) | #3890 *(added)* | Consumer setup describes the per-declared-marketplace refresh; #3888 folded in |

- **Repos touched (3):** .github, vergil-tooling, vergil-claude-plugin.
- **Children (23):**
  - 12 planned: the nine plan tasks (T1–T5, V1, V2, D1, D2) and the three
    bookends (#761, vergil-claude-plugin#850, this task #762).
  - 11 added during execution:
    - the spec and plan reconciliation after V1, .github#772;
    - the refresh fix found in D2, vergil-tooling#3879;
    - three removed additions from T5, vergil-tooling#3876, #3877 and #3878,
      later folded back into T5's branch;
    - two removed additions from T4, vergil-claude-plugin#860 and #861;
    - the release install step found in V2, vergil-claude-plugin#866;
    - the removed addition from #860, vergil-claude-plugin#869;
    - two consumer-guide fixes, vergil-tooling#3886 and #3890.
  - 4 operational, all SUCCESS: V1 vergil-tooling#3850, D1 #3852, D2 #3854,
    V2 #3855.
  - 4 closed by hand as fold-ins, with no PR of their own: vergil-tooling
    #3876, #3877 and #3878 (into #3880) and vergil-claude-plugin#869 (into
    #873). Two intake issues outside the epic closed the same way:
    .github#797 (into #3880) and vergil-tooling#3888 (into #3891).
- **PRs merged:** 14, counting the spec and plan PR and leaving out this one.
- **Releases cut (by the human):** vergil-tooling v2.1.280 (T1, T2), v2.1.281
  (T3) and v2.1.282 (T5, #3879); vergil-claude-plugin v2.1.52 (T4, released as
  V2's step 5). Not yet in a release: vergil-tooling #3889 and #3891, and
  vergil-claude-plugin #868, #871, #872 and #873.
- **Span:** the planning issue opened 2026-10-30 11:30Z and the epic 11:49Z.
  The spec and plan merged at 16:49Z the same day. The last child before this
  one closed 2026-11-01 12:38Z, about 49 hours after the epic opened.

## 1. How the plan evolved

The plan has no "Evolution during execution" log, so this section is rebuilt
from the issue record and the coordinating session.

- **Design review stopped at round 2 in every stage.** Spec pushback, plan
  pushback and alignment each ran two rounds, and the human stopped each one
  there. Together they raised 78 findings and accepted 69. Each finding not
  accepted has its reason or intake ref (.github#763, #765).
- **V1 overturned Task 3's premise, before Task 3 started.** The plan expected
  the VM-shaped case not to switch, so Task 3 was a reconcile that rewrote
  Claude Code's registration file, with an executor protocol and a remove
  rung. On Claude Code 2.1.312, once `settings.json` declares the new ref,
  `claude plugin marketplace add <repo>@<ref>` re-clones at that ref and the
  following `plugin update` moves the installed version. .github#772 (PR #780)
  rewrote spec §6 and Task 3 to "declare it, then re-add", with a check of
  each registration's repo and ref afterwards. The rest of Task 3 stayed.
- **T1 needed a fix outside its pinned interface.** One blocking
  security-class finding could only be fixed by going beyond the interface the
  plan pinned. The human chose the fix, and dismissed a second one for this
  task as later hardening (counts only, §4).
- **T2 kept several deviations from the plan, by ruling.** Fixing round 1's
  findings changed the push path. The human kept the changes, among them
  pushing the checked release tag straight to the rolling tag's ref under the
  lease, retrying the Release read only under `vrg-release`, and treating a
  failed `ls-remote` as an error rather than as "no tag". Round 4 ran by ruling
  past the default three.
- **T3 grew a whole-command agent refusal.** The human extended the agent
  refusal from `vrg-vm update` to every `vrg-vm` subcommand, since an agent has
  no reason to drive the VMs it runs in. This is an approved extension of spec
  §6, which named only `vrg-host plugins` and `vrg-vm update`. Ten flagged
  additions were kept as in scope.
- **D1 replaced one live test with evidence.** The App's forced tag move could
  not be attempted (§3, deviation D1).
- **D2 found a stale marketplace on the host.** The first `vrg-host plugins`
  run failed because `standard-tooling-marketplace`, an old marketplace no
  settings file declares, could not refresh, and the failed command's stdout
  was hidden. The human removed it by hand and the switch completed.
  vergil-tooling#3879 then made the refresh update only declared marketplaces,
  by name, and report both output streams. It shipped in v2.1.282 the same day.
  - **Spec §6 and plan Task 3 are now out of date on this point:** they
    describe a bare `claude plugin marketplace update`. The shipped refresh
    re-adds each declared marketplace, updates each of them by name (plus
    `claude-plugins-official` when an enabled plugin needs it), updates every
    install, then verifies. The human's decision is to leave the spec and plan
    as the as-planned record and record the drift here.
- **T5 split three additions out and folded them back in.** Round 1 removed a
  `vrg-promote` reference link, a `plugin-dirs.toml` caveat and a
  roll-forward note as out of scope, filed as vergil-tooling#3876, #3877 and
  #3878. Round 4's ruling folded all three back into the branch, along with
  .github#797, which round 2 had shown the branch itself made wrong. Rounds 4
  and 5 ran by ruling.
- **T4 went the other way.** Round 2 removed the README Update and Verify
  rewrite and the "changing a registered `ref`" note as out of scope. They
  became vergil-claude-plugin#860 and #861, and the human accepted the branch
  at round 2 with the removal commit unreviewed.
- **V2's end-to-end release used T4.** With the human's approval, step 5
  released the real merged change vergil-claude-plugin#851 as v2.1.52 instead
  of a throwaway test change. The run found that the plugin's own release
  install step (`[publish].consumer-refresh`) still ran the old host-only
  commands. That became vergil-claude-plugin#866, which now runs `vrg-host
  plugins` and `vrg-vm update --all`.
- **The follow-ups formed a chain across the two repos.** #860's review
  showed two vergil-tooling guides deferring to a README section that no
  longer explained them (vergil-tooling#3886). #3886's review found that a
  guide misdescribed which marketplaces the refresh updates (#3888). The
  documentation review then filed #3890 for the same page and folded #3888
  into it. The documentation review also folded in #869.
- **Order held otherwise:** T1, T2 and V1 in parallel; then T3; D1 after
  v2.1.280; D2 after v2.1.281; T5 and then T4; V2 after v2.1.282; the
  documentation review last.

## 2. Lessons learned

- **Agentic review and human judgment worked well together.** The human
  points to vergil-tooling#3849 (T2) as the clearest example, even with the
  process still being fixed along the way. Review rounds found real problems
  in the push path, and round 2 found problems in round 1's fixes. Once the
  changes went past the plan, the agent stopped and gave the human a short
  list of specific items to rule on. It did not decide them itself or keep
  changing code.
- **One standing ruling can carry a known false alarm across tasks.** The
  stray-write check failed in essentially every round of every task, because
  it picks up the harness's and other sessions' temporary files. A single
  standing ruling (treat stray-write failures as false positives until further
  notice) kept the work moving across tasks without a stop each time. The cost
  is that the check protected nothing for the whole epic
  (vergil-claude-plugin#852).
- **Validate the premise before building on it.** The plan put V1 before Task
  3 for this reason, and it paid off. Without it, Task 3 would have shipped a
  rewrite of Claude Code's internal registration file that the CLI no longer
  needed.
- **A real host carries history a clean test does not.** V1 ran in an
  isolated config directory, so it could not see the stale marketplace that
  failed D2. A refresh should touch only what it manages; #3879 made it so.
- **One procedure described in many places breeds follow-ups.** The fleet
  pick-up procedure appears in the plugin README, its CLAUDE.md and docs site,
  and three vergil-tooling guides. Keeping them consistent produced five of
  the eleven added tasks. Next time, give a procedure one canonical page and
  have every other page link it.
- **Strict scope on docs tasks costs tasks.** Removing out-of-scope additions
  kept each review focused, but produced six small follow-up tasks, each with
  its own branch, CI run and PR. Several were folded back in and closed by hand,
  because a PR links only one issue (.github#800).
- **The human runs every host step.** `vrg-host plugins`, `vrg-vm` and
  `vrg-promote` refuse agents by design, so for D1, D2 and V2 the human ran each
  host command and pasted its output. This is correct, and it is most of the
  human's time in the operational tasks (Appendix A).
- **The round cap is a real decision point.** Three tasks ran past three
  rounds by ruling (#3849, #3851, #3853), and four branches were accepted with
  commits no review had seen (#3849, #3851, #851, #860). Each time it was the
  human's informed call, recorded on the issue.

## 3. Compromises & tradeoffs

- **D1: the App's forced tag move was not tested live.** `vrg-gh` denies
  `gh api` for the user identity, `vrg-git` denies force-push, and minting a
  raw App token is not allowed. In its place: the human's audit, which
  compared bypass actors and found Admin the only one, and the bypass messages
  on the Admin's own move and delete of a test tag. Branch creation by the App
  was tested live through `vrg-git push` and refused (GH013). A sanctioned
  probe is intake .github#787.
- **Spec §6 and plan Task 3 describe the refresh as planned, not as shipped**
  (§1). The docs describe what shipped.
- **V2's end-to-end release was a real change, not a test change.** v2.1.52
  also carried unrelated agentic-review gate changes (vergil-claude-plugin #859
  and #862), so the release that proved the deploy path shipped more than the
  change under test.
- **Branches accepted with unreviewed commits:** #3849 (round 4's fixes),
  #3851 (two commits), #851 (the removal commit) and #860 (round 3's fixes).
  Three passes also carry one late fix commit after the last reviewed HEAD:
  #3853, #3886 and #3890. #850 was accepted at round 3 with no further
  commit.
- **Findings left unfixed on purpose, by ruling or plan:**
  - T2: a failed `ls-remote` raises, a deviation from the plan's table;
  - T3: a failed `marketplace update` is collected and the refresh goes on,
    per the plan's "never swallows a failure" contract.

  Every other unfixed finding has an intake issue (§4).
- **Additions kept beyond the plan, by ruling:** eight in T2, ten in T3 and
  the whole-command `vrg-vm` refusal, two in T5.
- **One out-of-scope finding was filed without a ruling in conversation.**
  #850's R2.2 became .github#803 under the coordinator's standing direction
  that out-of-scope suggestions become follow-ups. The human can re-route it at
  intake review.
- **Several follow-ups are merged but not released.** The plugin's new
  release install step (#868) takes effect from the next plugin release; the
  docs changes in vergil-tooling #3889 and #3891 and vergil-claude-plugin #871,
  #872 and #873 publish with the next releases.
- **Security:** one security-class finding in T1 was dismissed for that task
  as later hardening and stays open as a record (counts in §4).

## 4. New problems & opportunities

- **Intake and planning filed during the epic, all open:**
  - From the planning issue: .github#763 (let an issue block rolling-tag
    promotion) and #765 (pin management at minor-line moves).
  - From T1: .github#775 (the source-control standard's ruleset list for
    documentation repos does not match the code).
  - From T2: .github#777 (share the promote flow between the CLI and
    `promote()`), #778 (one GitHub remote URL parser), #783 (require stdout to
    be a terminal before prompting), #784 (report a concurrent move to the same
    release as already promoted). A shared identity predicate was noted on
    .github#766.
  - From D1: .github#787 (a sanctioned App probe for ruleset checks).
  - From T3: .github#789 (write the guest config atomically), #790 (`vrg-vm
    start` and `rebuild` with no host `settings.json`), #792 (keep earlier
    failures in the refresh summary), #793 (tell an unreachable VM from a
    missing guest file), #794 (remove the uncalled `_marketplace_sources`),
    #795 (null or repo-less marketplace declarations).
  - From T5: .github#799 (off-platform not-running hints name `vrg-vm start`,
    which refuses them).
  - From #3879: .github#801 (does a plugin from an undeclared marketplace go
    stale after the refresh?).
  - From the session: .github#802 (log token use for every sub-agent call, and
    split `issue-implement` so both sides of the review loop are measured).
  - From the documentation review: .github#803 (the docs site names the
    vergil-tooling CLIs `st-*`).
  - Planning: .github#800 (batch several issues into one branch and PR), from
    the fold-ins.
  - From T1: vergil-claude-plugin#852 (the stray-write check fails when tasks
    run in parallel).
- **Filed by this retrospective's sweep:** .github#807, #808, #809 and #810
  (below).
- **Security:** findings went through the private advisory process only. The
  public record gives counts:
  - design review: 22 security-class findings across the six rounds, each
    recorded privately and retargeted to the task that fixes it;
  - the agentic-review gate: 21 found across the tasks, 17 fixed on their
    branches and 4 recorded, each with a public pointer issue in .github.

**The unaddressed-findings sweep** covered the 22 children other than this
one, read against the trusted-author lists. Every comment on them is from a
trusted agent identity.

- **Gate summaries (12 tasks):** vergil-tooling #3848 (pass after 2), #3849
  (accepted after 4), #3851 (accepted after 4), #3853 (pass after 5), #3879
  (pass after 1), #3886 (pass after 1) and #3890 (pass after 3);
  vergil-claude-plugin #851 (accepted after 2), #860 (accepted after 3), #861
  (pass after 1), #866 (pass after 1) and #850 (accepted after 3). Every
  finding they list as not fixed names an issue that exists, carries a
  recorded reason, or names a removed addition's task.
- **Round comments but no summary:** none.
- **No gate records, operational (no PR):** V1 vergil-tooling#3850, D1 #3852,
  D2 #3854 and V2 #3855. Their result comments were swept.
- **No gate records, folded into another task's branch:** vergil-tooling
  #3876, #3877 and #3878 (reviewed in #3853's rounds 4 and 5) and
  vergil-claude-plugin#869 (reviewed in #850's rounds).
- **No gate records, design documents:** .github#761, the spec and plan,
  reviewed in mode 1 on the planning issue. .github#772, the spec and plan
  reconciliation (PR #780), has no review record of either kind. The human
  ruled this acceptable: it was a docs-only correction of the spec and plan,
  with low risk. No issue was filed.
- **PR reviews:** none of the 14 PRs has a review or a conversation comment.
- **The planning issue, .github#759:** every finding not accepted has its
  reason or intake ref.
- **Untracked candidates and their outcomes:**
  - The stray-write check failing in essentially every round. **Found:**
    vergil-claude-plugin#852 (and #858, #836).
  - paad reporting `2.0.2-preview` where the doctrine expects `2.0.2`, a
    warning in most rounds. **Found:** expected while sessions load paad's
    preview tree through `plugin-dirs.toml` (.github#698, S4.11).
  - Follow-ups that could only be folded into another branch and closed by
    hand. **Found:** .github#800.
  - The plan's missing "Evolution during execution" log. **Found:**
    vergil-claude-plugin#832.
  - Spec §6 and plan Task 3 out of date after #3879. **Not a new issue:** the
    human's decision is to record it here (§1).
  - Rulings on #3851 and #3853 recorded outside the paused response comment,
    because the session said it could not edit that comment. **Filed:**
    .github#807.
  - A reviewer specialist on #850 ran `vrg-validate` inside the task's
    worktree. **Filed:** .github#808.
  - Tasks run after plugin v2.1.52 did not all follow its stricter doctrine
    (#3886 and #3890 passed with a commit after the last reviewed HEAD), and the
    gate records do not say which doctrine governed. **Filed:** .github#809.
  - A command typed into a message's text is not run as a shell command.
    **Dropped (human):** Claude Code platform behaviour, not ours.
  - GitHub's REST API refused the Admin's branch creation with only `422
    Reference update failed`; a git push gave the rule text. **Filed:**
    .github#810.

## 5. What's next

- The next vergil-tooling and vergil-claude-plugin releases ship the merged
  follow-ups, including the plugin's new release install step (#868).
- .github#763: let an issue hold the rolling tag, so a standing rollback is
  not undone by hand.
- .github#800: batch several issues into one branch and PR.
- .github#802: measure token use per sub-agent call and both sides of the
  review loop.
- [vergil-project/.github#699](https://github.com/vergil-project/.github/issues/699)
  (paad unattended mode) and vergil-claude-plugin#852 and #858: the
  stray-write check's future.
- Moving the plugin to a `v2.2` line stays a documented, deliberate change of
  the host `ref` (spec §8), not an automated one.

## Appendix A — Operational notes

The mechanical sequence, D1 → D2 → V2. Every host step was run by the human,
who pasted the output into the task; the agent checked what it could in its
own VM.

1. **Release vergil-tooling v2.1.280** (T1, T2) and install it on the host.
   From this release on, the nightly config audit expects the new rulesets.
2. **D1 (vergil-tooling#3852).**
   - The human ran `vrg-github-repo-config apply` on all nine repos (.github,
     docs, packages, vergil-actions, vergil-claude-plugin, vergil-containers,
     vergil-python, vergil-tooling, vergil-vm). Each reported the widened tag
     include and the new branch-name ruleset as missing, then `applied`.
   - The audit with the human's credentials, so bypass actors are compared,
     reported vergil-claude-plugin, vergil-tooling and vergil-actions
     `compliant`. The live branch-name include is `refs/heads/v[0-9]*`, equal to
     Task 1's `BRANCH_PATTERN`.
   - A test tag `v0.0` was created, moved and deleted by the Admin, each
     accepted with a "Bypassed rule violations" message.
   - Creating a `v0.0` branch was refused for the App (`vrg-git push`) and for
     the Admin. For the Admin, the REST call gave only `422 Reference update
     failed`; a git push gave the rule text (GH013).
   - Final state: no `v0.0` tag or branch, `v2.1` unchanged, no `refs/heads/v*`.
3. **Release vergil-tooling v2.1.281** (T3) and install it on the host and in
   every VM (`vrg-vm update --all`).
4. **D2 (vergil-tooling#3854).**
   - The human changed the host `settings.json` marketplace `ref` from `main`
     to `v2.1` and ran `vrg-host plugins`. It failed on the stale
     `standard-tooling-marketplace`. The human removed it with `claude plugin
     marketplace remove`, and the re-run passed.
   - `vrg-vm update --all` updated all three VMs, each re-registered at
     `vergil-project/vergil-claude-plugin@v2.1` and verified.
   - Every machine showed 2.1.51, the release `v2.1` pointed at. No fleet
     repo's `develop` declares a project-level `extraKnownMarketplaces`.
5. **Release vergil-tooling v2.1.282** (the runbook, #3879's fix).
6. **V2 (vergil-tooling#3855).**
   - Rollback: in a vergil-claude-plugin checkout, `vrg-git fetch --tags
     --force origin`, `vrg-promote --dry-run 2.1.50`, `vrg-promote 2.1.50`.
     After `vrg-host plugins` and `vrg-vm update --all`, every machine showed
     2.1.50, installed and in a fresh session.
   - Forward: the same steps to 2.1.51.
   - End to end: `vrg-release` of #851 published v2.1.52, and its promote
     stage moved `v2.1` on its own. Every machine showed 2.1.52. The fleet stays
     there, by the human's decision.

**Gotchas.**

- **Fetch tags with `--force`.** The rolling tag moves every release, and a
  plain fetch refuses to update a local tag that has moved.
- **The ref check cannot see a rollback.** The registered ref is `v2.1` before
  and after a tag move, so the refresh's verification passes either way.
  Check the installed version with `claude plugin list --json` (every scope)
  and in a freshly started session.
- **`plugin-dirs.toml` overrides only what it lists.** The host's file lists a
  paad tree only, so the installed vergil plugin still loads. A vergil tree
  listed there would override it, in Lima sessions only.
- **Undeclared marketplaces.** Before v2.1.282, any stale registered
  marketplace failed the whole refresh. Now the refresh leaves it alone.
- **`--no-promote` is not remembered.** It must be passed to every `vrg-release
  --resume` of a held release, or the resume moves `v2.1` forward over a
  rollback.
- **`vrg-git` denies `remote`.** Read origin's URL with `vrg-git ls-remote
  --get-url origin`.
- **Host-only commands refuse agents.** `vrg-host plugins`, `vrg-vm` and
  `vrg-promote` must be run by the human.

## Appendix B — Extended metrics

- **Design review (planning issue .github#759):** six rounds; 78 findings, 69
  accepted.

  | Stage | Rounds | Findings | Accepted |
  |---|---|---|---|
  | Spec pushback | 2 | 15 | 13 |
  | Plan pushback | 2 | 30 | 28 |
  | Alignment | 2 | 33 | 28 |

- **Agentic-review gate:** 30 rounds over 12 tasks; 7 passes and 5 accepted
  branches; 3 tasks past the default three rounds by ruling.

  | Task | Rounds | Outcome |
  |---|---|---|
  | vergil-tooling#3848 (T1) | 2 | pass |
  | vergil-tooling#3849 (T2) | 4 | accepted |
  | vergil-tooling#3851 (T3) | 4 | accepted |
  | vergil-tooling#3853 (T5) | 5 | pass |
  | vergil-tooling#3879 | 1 | pass |
  | vergil-tooling#3886 | 1 | pass |
  | vergil-tooling#3890 | 3 | pass |
  | vergil-claude-plugin#851 (T4) | 2 | accepted |
  | vergil-claude-plugin#860 | 3 | accepted |
  | vergil-claude-plugin#861 | 1 | pass |
  | vergil-claude-plugin#866 | 1 | pass |
  | vergil-claude-plugin#850 | 3 | accepted |

- **Review tokens:** reported only for #850 (392,665 over three rounds). The
  other tasks' records predate the cost line or did not report it; .github#802
  asks for this everywhere.
