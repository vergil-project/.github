# Automatic fleet sync — retrospective

- **Epic:** vergil-project/.github#1064
- **Planning issue:** vergil-project/.github#909
- **Spec:** [spec.md](spec.md) · **Plan:** [plan.md](plan.md)

## 0. At a glance

Master copies of managed files (CLAUDE.md, the hook-guard shim, Claude settings, `.gitignore`, `dependabot.yml`, workflow pins) reached the fleet only when a human ran one of seven ad-hoc `vrg-*-sync` tools. Repos drifted between runs, and the nightly ops job could only report it. We set out to make sync automatic:

- a new `vrg-github-repo-config update`;
- an `automation` identity mode, verified against the org automation App's token;
- a reusable building block for mechanized PRs.

Each repo's nightly ops job now runs `update` as the App. It fixes GitHub-side drift, then opens one sync PR on a protected `automation/` branch and waits until it merges. Release a master-copy change in vergil-tooling, and every repo's `develop` has it the next morning, with no human involved.

That shipped and passed its live validation (vergil-actions#1158). On its first run, vergil-project/docs closed a stale sync PR, opened a fresh one as `vergil-project-automation[bot]`, merged it within the run and exited 0. Across the 22-repo fleet, 17 repos met the acceptance outright and 5 exited 2 by design, each naming its PR. The seven `vrg-*-sync` tools are retired.

**Code and docs PRs** (35; each closes its task):

| PR | Repo | What it did | Merged |
|---|---|---|---|
| [.github#1082](https://github.com/vergil-project/.github/pull/1082) | .github | Spec and plan | 2026-11-13 |
| [vergil-actions#1160](https://github.com/vergil-project/vergil-actions/pull/1160) | vergil-actions | A1: scope every automation token, per-job concurrency, App permission docs (#1153) | 2026-11-13 |
| [vergil-containers#857](https://github.com/vergil-project/vergil-containers/pull/857) | vergil-containers | C1: scope the lifecycle App tokens (#856) | 2026-11-13 |
| [vergil-tooling#4310](https://github.com/vergil-project/vergil-tooling/pull/4310) | vergil-tooling | Task 2: audit managed files in a fresh clone of `develop` (#4301) | 2026-11-13 |
| [vergil-tooling#4312](https://github.com/vergil-project/vergil-tooling/pull/4312) | vergil-tooling | Task 8: `vrg-repo-status` shows automation PRs as their own type (#4307) | 2026-11-13 |
| [vergil-tooling#4314](https://github.com/vergil-project/vergil-tooling/pull/4314) | vergil-tooling | Task 3a: fixers for the shim, CLAUDE.md and Claude settings (#4302) | 2026-11-13 |
| [vergil-tooling#4315](https://github.com/vergil-project/vergil-tooling/pull/4315) | vergil-tooling | Task 4: the mechanized-PR library (#4304) | 2026-11-13 |
| [vergil-tooling#4316](https://github.com/vergil-project/vergil-tooling/pull/4316) | vergil-tooling | Narrow the catch-all exception in the cwd repo match (#4311) | 2026-11-13 |
| [vergil-tooling#4318](https://github.com/vergil-project/vergil-tooling/pull/4318) | vergil-tooling | Task 3b: fixers for `.gitignore`, `dependabot.yml` and workflow pins (#4303) | 2026-11-13 |
| [vergil-tooling#4320](https://github.com/vergil-project/vergil-tooling/pull/4320) | vergil-tooling | Move the advisory sync out of `vrg-epic-audit --close` into `vrg-security-record sync` (#4313) | 2026-11-13 |
| [vergil-actions#1165](https://github.com/vergil-project/vergil-actions/pull/1165) | vergil-actions | `github-config` token gets Contents write to read merge settings (#1164) | 2026-11-13 |
| [vergil-actions#1169](https://github.com/vergil-project/vergil-actions/pull/1169) | vergil-actions | Scope the `epic-sweep` token; drop the App's advisories permission (#1159) | 2026-11-13 |
| [vergil-tooling#4328](https://github.com/vergil-project/vergil-tooling/pull/4328) | vergil-tooling | Repo settings GitHub does not return are "not checked", never `False` (#4322) | 2026-11-13 |
| [vergil-tooling#4329](https://github.com/vergil-project/vergil-tooling/pull/4329) | vergil-tooling | The not-checked line names Contents read and write for merge settings (#4326) | 2026-11-13 |
| [vergil-tooling#4332](https://github.com/vergil-project/vergil-tooling/pull/4332) | vergil-tooling | The check for an apply is the human's apply exiting 0, not an agent's `diff` (#4327) | 2026-11-13 |
| [vergil-tooling#4339](https://github.com/vergil-project/vergil-tooling/pull/4339) | vergil-tooling | Add mq-rest-admin-project to `IN_SCOPE_ORGS` and the identity docs (#4336) | 2026-11-13 |
| [vergil-actions#1174](https://github.com/vergil-project/vergil-actions/pull/1174) | vergil-actions | A1b: the dispatch-only bypass probe (#1155) | 2026-11-13 |
| [vergil-actions#1175](https://github.com/vergil-project/vergil-actions/pull/1175) | vergil-actions | List mq-rest-admin-project in the App table (#1173) | 2026-11-13 |
| [vergil-tooling#4351](https://github.com/vergil-project/vergil-tooling/pull/4351) | vergil-tooling | Task 1: the `automation` identity mode, verified against the token (#4300) | 2026-11-13 |
| [vergil-tooling#4356](https://github.com/vergil-project/vergil-tooling/pull/4356) | vergil-tooling | Task 5: auto-merge, the automation-branch ruleset, fatal empty bypass reads (#4305) | 2026-11-13 |
| [vergil-tooling#4360](https://github.com/vergil-project/vergil-tooling/pull/4360) | vergil-tooling | Task 6: `update`; `apply` deprecated (#4306) | 2026-11-13 |
| [.github#1089](https://github.com/vergil-project/.github/pull/1089), [#1100](https://github.com/vergil-project/.github/pull/1100), [#1119](https://github.com/vergil-project/.github/pull/1119), [#1144](https://github.com/vergil-project/.github/pull/1144) | .github | The plan's four evolution entries, plus the spec's develop-only rule (#1086, #1099, #1118, #1143) | 11-13 to 11-15 |
| [vergil-tooling#4372](https://github.com/vergil-project/vergil-tooling/pull/4372) | vergil-tooling | Sync PRs are watched until merged, updating a branch that falls behind (#4370) | 2026-11-14 |
| [vergil-actions#1186](https://github.com/vergil-project/vergil-actions/pull/1186) | vergil-actions | A2: `github-config` runs `update` as the automation identity (#1156) | 2026-11-14 |
| [vergil-actions#1188](https://github.com/vergil-project/vergil-actions/pull/1188) | vergil-actions | Cap the `github-config` job at 60 minutes (#1187) | 2026-11-14 |
| [vergil-tooling#4419](https://github.com/vergil-project/vergil-tooling/pull/4419) | vergil-tooling | Task 9: retire the `vrg-*-sync` tools (#4309) | 2026-11-15 |
| [vergil-tooling#4421](https://github.com/vergil-project/vergil-tooling/pull/4421) | vergil-tooling | `vrg-release` audits vergil-tooling against its in-tree templates (#4414) | 2026-11-15 |
| [vergil-tooling#4428](https://github.com/vergil-project/vergil-tooling/pull/4428) | vergil-tooling | Documentation review (#4268) | 2026-11-15 |
| [vergil-actions#1239](https://github.com/vergil-project/vergil-actions/pull/1239), [vergil-containers#900](https://github.com/vergil-project/vergil-containers/pull/900), [vergil-claude-plugin#1086](https://github.com/vergil-project/vergil-claude-plugin/pull/1086), [#1087](https://github.com/vergil-project/vergil-claude-plugin/pull/1087) | 3 repos | Per-repo doc tasks spawned by the review (#1233, #899, #1083, #1084) | 2026-11-15 |

**Operational tasks** (6, all closed SUCCESS):

- **D1** (vergil-actions#1154): the token scoping is live. It failed first (§1), then passed on a re-run.
- **D2** (.github#1083): each in-scope org's App is set to the asserted permissions, and `VRG_AUTOMATION_APP` is set.
- **V1** (.github#1084): an Administration-write App token reads `bypass_actors`, in 4 repos across 3 orgs, including a private one.
- **D3** (vergil-tooling#4308): the human's same-day `update` sweep. 18 repos, all exit 0, 17 sync PRs.
- **D4** (vergil-actions#1157): the switch released.
- **V2** (vergil-actions#1158): the live validation.

**By the numbers:**

- **Repos with PRs:** 5 (.github, vergil-actions, vergil-containers, vergil-tooling, vergil-claude-plugin). The fleet itself is 22 repos in 3 orgs.
- **Tasks:** 41, plus this retrospective.
  - 21 were planned: 19 implementation tasks and 2 bookends.
  - 20 were added during execution: 4 plan-log entries, 13 follow-ups filed on rulings, and 3 doc tasks from the documentation review.
- **Releases cut** (by the human), which carried epic work:
  - vergil-tooling v2.1.307, v2.1.308, v2.1.309, v2.1.310 and v2.1.314;
  - vergil-actions v2.1.66, v2.1.67, v2.1.68, v2.1.69 and v2.1.71.

  The mapping of PRs to tags is from merge times, and is confirmed by a task comment where one exists. The two final doc PRs (vergil-tooling#4428, vergil-actions#1239) merged after the last releases and are not yet released.
- **Span:** the first child opened 2026-11-12T20:36Z, and the last closed 2026-11-15T14:24Z. That is about 2 days 18 hours.

## 1. How the plan evolved

The plan's shape held: scope the tokens, configure the Apps, prove the premise, build the tooling, sweep, switch, validate, retire. Every planned task ran. What changed was around it. The plan's 21 tasks grew to 41, and almost every addition came from something a live step found that review had not. The four entries in the plan's evolution log record each change. In order:

- **Scoping the tokens exposed a hidden dependency (A1, #1153).** The nightly `epic-sweep` needed Repository security advisories write, which `actions/create-github-app-token` cannot request.
  - The human split the token off A1. The fix moved the advisory sync out of the automated sweep into a command a human runs (`vrg-security-record sync`, #4313), and then scoped the token (#1159).
  - With no automation job needing it, the advisories permission was reversed out of D2: the App must not hold it.
- **D1 failed honestly.** The scoped `github-config` token got no merge-setting fields back, and the audit reported nine false drifts.
  - The fix was the documented permission: Contents write (#1164).
  - The failure also showed the tool read missing fields as `False`. #4322 made them "not checked", and blocked Task 6 on it, before `update` could act on a reading never made.
- **D2's org scope was decided at the door.** The run found six orgs with managed repos.
  - The human configured four and ruled two dormant.
  - mq-rest-admin-project has no ops job, so V1, D3 and the switch cover three orgs. Onboarding it went to intake (.github#1107).
- **D3 found auto-merge was not enough.** Two of 17 sync PRs stalled behind `develop` with green CI, because the branch must be up to date to merge.
  - The human made "wait until merged" a required feature (#4370). The job merges the PR itself when it is ready, updates a branch that falls behind, and exits non-zero naming the PR if it does not merge within 40 minutes.
  - The automation token's fixed one-hour life set that budget, and a 60-minute job cap followed (#1187).
- **One security finding turned out to be its own epic.** The advisory on A2 asked for the App's credentials to be usable only on `develop` (#1185).
  - Scoping showed roughly seven pieces of work across five repos plus a human rollout.
  - The human deferred it to planning, as hardening that does not change this epic's deliverable.
- **The human held the switch for one release.** D4 waited for the job cap, so A2 and #1187 shipped together.
- **That release broke the install step.** v2.1.71 set `VRG_AUTOMATION_APP` for the whole job, and the tooling installer's identity check died without a token.
  - The fix (#1192) belonged to epic #1091 and landed the same day.
  - The human ruled V2 blocked, not failed, and did not revert.
- **The release preflight could not pass after a template change.** vergil-tooling holds the master copies, so its own files run ahead of the installed release. A template change (#4401) landed 21 minutes after v2.1.312, and `vrg-release` reported false drift. #4414 makes the release audit use the templates being released.

Smaller rulings settled open questions inside tasks:

- `diff`'s exit codes stayed as they are, and only the guidance changed (#4327). The human pointed out that an agent never runs `apply`, so the question was academic.
- The automation identity's spoofing advisory was rejected. A wrong `VRG_AUTOMATION_APP` changes only the tool's label, never the token's rights.
- The commit-authorship check was dropped from the automation-branch advisory, because the ruleset and the expected-head merge already cover it.

## 2. Lessons learned

- **Live steps found what review could not.** Six design-review rounds raised 73 findings. They still missed:
  - the sweep's hidden advisories permission;
  - the merge-settings permission;
  - the silent `False` reads;
  - the behind-`develop` stall;
  - the job-wide variable breaking the installer.

  Each surfaced at a deployment or validation step, and each was fixed forward the same day. The staged rollout earned its keep. Next time, put a live check as early as the premise allows, as D1 and V1 did.
- **A failing check should fail loudly, and that rule pays for itself.** D1's false drift was traced in minutes, because the tool printed exact field values. #4322's "not checked" made the next such gap visible instead of acted on, and #4370 made a PR that does not merge an exit 2 naming it. Every time, the no-silent-failures rule turned a would-be mystery into a named PR or field.
- **Auto-merge is an optimisation, not a guarantee.** The design assumed that turning on auto-merge finished the job. On a busy `develop`, it routinely does not. Anything that must land should watch until merged, and `vrg-finalize-pr`'s logic was already the right shape.
- **GitHub's hard limits shape designs; find them before you lean on them.**
  - App installation tokens live one hour, and cannot be renewed inside a job.
  - Environments are the only branch-scoped secrets.
  - Scheduled runs can be dropped.
  - A strict up-to-date rule blocks auto-merge.

  Each one changed a design after the plan was approved.
- **The repo that holds the master copies is special.** It is always ahead of its own release, so any check that compares it with the installed tool will lie after a template change. #4414 fixes the release audit; the same reasoning applies to any future self-check.
- **The agent's own checks have blind spots.** The agent identity cannot read private repos' files, org secrets or installation permissions, and cannot start workflows. The human's terminal output was the evidence for D2, D3 and V2, and the human's local clones replaced scratch clones.

## 3. Compromises & tradeoffs

- **The credentials environment is deferred** (vergil-actions#1185, in planning). The App's key, which holds Administration write since D2, is still readable by any workflow in a repo. The human judged the exposure narrow: agents cannot push workflow files or start workflows. Its advisory stays open on #1185.
- **V2 was validated partly by inference.** `update` does not log the identity mode it resolved or the bypass actors it read back. V2 relied on:
  - the guard passing;
  - the PR author being the App;
  - `update`'s exit code;
  - one ruleset read by the human.

  The plan (P3.9) asked for the read-back to be logged. See §4.
- **The merge watch's branch-update path has not run live.** Every PR that merged during V2 opened fresh and never fell behind. Tests cover the path against a mock clock, and a failure there exits 2, naming the PR.
- **No token renewal.** A job that needs more than the token's hour cannot use the building block. Config sync takes minutes. `update-deps`, at about 80 minutes, could not move onto it as built (§5).
- **Advisory states now lag.** With the sync out of the nightly sweep, they update only when someone runs `vrg-security-record sync`. The human accepted that cost on #4313.
- **The nightly schedule is not reliable.** On the morning V2 was checked, 7 of 22 repos had no scheduled run. They were started by hand.
- **Known small debts let go below the reviewer's bar, still on `develop`:**
  - the "queued, never cancelled" concurrency wording (GitHub replaces an older pending run);
  - `apply`'s second App-ID lookup exiting 1, not 2;
  - the `IN_SCOPE_ORGS` comment crediting only epic #558;
  - a `_resolve_config` docstring point.
- **Two fleet repos ended V2 red by design:** the Rust fixtures, until their unfinished release (#1091) bumps `develop`. Three more ended red from CI failures that predate this epic and have their own fix tasks.

## 4. New problems & opportunities

Where each surfaced item went:

| Item | Where it went |
|---|---|
| Move the App credentials into a `develop`-only environment | Planning: vergil-actions#1185 |
| Move the other PR-opening jobs onto the building block and the `automation` mode | Planning: .github#1067 (§5) |
| Onboard mq-rest-admin-project to nightly ops | Intake: .github#1107 |
| `vrg-github-repo-init` derives the clone location from a projects root | Intake: .github#1120 |
| Every follow-up the work found (#4311, #4313, #4322, #4326, #4327, #4336, #4370, #4414, #1159, #1164, #1173, #1187, plugin#1084) | Filed and landed in this epic |

**Unaddressed-findings sweep.** It covered 41 tasks and the epic issue, all 35 merged PRs, and the six design-review rounds on #909. All their comments were from trusted authors.

- Every `Awaiting a ruling` item has a closing ruling, and every filed issue exists.
- On #909, every finding not accepted has its reason.
- No PR had a review or comment to check.

The untracked items, and their outcomes:

| Source | Item | Outcome |
|---|---|---|
| vergil-actions#1158; plan P3.9 | `update` does not log its resolved identity mode, or each ruleset's bypass actors read back | **Filed:** intake, vergil-tooling#4433 |
| vergil-actions#1158 | The nightly schedule dropped 7 repos' runs on 2026-11-15 | **Found:** added as evidence on .github#1131, which tracks the fleet's scheduling |
| vergil-tooling#4308 | Under the agent identity, `audit` cannot check a private repo's files (no git credential on the agent VM) | **Dropped** by the human: documented, and ops and the human have credentials |
| vergil-actions#1154 | The fleet-wide CLAUDE.md drift item never got a formal ruling line | **Dropped** by the human: D3 and the nightly `update` fixed it, by design |
| vergil-tooling#4370 | Token renewal for jobs longer than an hour | **Dropped** by the human in #4370's scope change; noted on .github#1067 |
| Below the bar | The four small debts in §3 | **Dropped** by the human: already let go; fix in passing |

Security-class items: 2. Both were ruled on by the human.

## 5. What's next

- **The follow-on brainstorm, .github#1067** (in planning), is next. It covers two moves, and its notes carry this epic's lessons:
  1. move `update-deps`, `upstream-refresh` and the containers lifecycle job onto the mechanized-PR building block;
  2. move every other job that mints the App token onto the `automation` mode, with `develop`-only guards.

  The notes:
  - the block now waits until merged;
  - the one-hour token limit;
  - `update-deps` runs inside the human-gated `vrg-release`;
  - the guard pattern;
  - the job-wide variable trap.
- **vergil-actions#1185** should be planned with it, since both concern how the App token is handled on `develop` and `main`.

## Appendix A — Operational notes

The rollout, as it actually ran:

1. **Scope the tokens.** A1 (vergil-actions) and C1 (vergil-containers) shipped, then the follow-ups #1159 (the `epic-sweep` token) and #1164 (Contents write for merge settings). D1 confirmed them on vergil-tooling's ops run and on `.github`'s epic-sweep. The consumer was vergil-tooling, because every other repo's nightly run was red from CLAUDE.md drift at the time.
2. **Configure each App (D2), by hand, per org.** The orgs were vergil-project, logical-minds-foundry, mnemosys-project and mq-rest-admin-project.
   - Set the permission table: Administration, Contents, Issues, Pull requests and Workflows write; Metadata and Dependabot alerts read; no advisories.
   - Accept the change on the installation.
   - `gh variable set VRG_AUTOMATION_APP --org <org> --visibility all --body <org>-automation`.
   - Check with `gh api orgs/<org>/installations`, which shows only accepted permissions.

   The code search used the human's local clones (`git grep` on `origin/develop`), not new clones.
3. **Probe (A1b, V1).** The dispatch-only `bypass-probe` printed the ruleset's bypass actors, the variable's presence and the token's login, in a public and a private repo per org.
4. **Release the tooling, and sweep the same day (D3).** The human ran `update` for every repo from their own host, in a loop printing each exit code. A `!` command in the agent VM would run as the agent identity, which `update` refuses.
5. **Switch (A2, with #1187), in one vergil-actions release (D4).** If the first runs fail before `update`, look at the install step first (the #1192 trap).
6. **Validate (V2).** Start one repo with a known stale sync PR, then the rest, or let the nightly run do it. Read each `github-config` log for:
   - `closed the earlier sync PR`;
   - the new `automation/repo-config-sync/<run-id>` PR;
   - `merged`;
   - `local: compliant`;
   - exit 0.

   Check for repos the schedule skipped.
7. **After a template change in vergil-tooling,** expect the release preflight to need #4414's in-tree audit, now in v2.1.314.
