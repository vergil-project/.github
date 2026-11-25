# agent-submitted-prs — retrospective

- **Epic:** vergil-project/.github#1384
- **Spec:** [`spec.md`](spec.md) · **Plan:** [`plan.md`](plan.md), with its
  [Evolution during execution](plan.md#evolution-during-execution) log
- **Planning issue:** vergil-project/.github#1383
- **Retrospective task:** vergil-project/.github#1386

**Span:**

| Window | Start | End | Elapsed |
| --- | --- | --- | --- |
| Whole epic | Thu 19 Nov 2026, 13:46 UTC | Wed 25 Nov 2026, 12:43 UTC | 5 d 22 h 56 min |
| Planning | Thu 19 Nov 2026, 13:45 UTC | Thu 19 Nov 2026, 18:41 UTC | 4 h 55 min |
| Implementation | Mon 23 Nov 2026, 19:21 UTC | Wed 25 Nov 2026, 12:43 UTC | 1 d 17 h 22 min |

The whole-epic window ends when the last child other than this
retrospective closed (vergil-project/vergil-tooling#4751, the documentation
review). The implementation window runs from the first merged PR after the
spec-and-plan PR (vergil-project/vergil-fixture-python-app#75) to the last
one (vergil-project/vergil-tooling#5179). Between the spec merging on
19 Nov and the first implementation PR on 23 Nov, the epic waited in the
queue; the rebase onto the recent epics (Task 1b) opened the work.

## 0. At a glance

**What we set out to do:** move the agent permission boundary so the
working (user) agent submits and finalizes its own PRs into `develop`, in
the three orgs under active work. A task would go from implementation to
merged code with no human step, and an epic could chain from one task to
the next. Releases and promotion to `main` stay human. The safeguards were
to be a `main` ruleset that keeps agents off `main`, write-capable secrets
confined to an environment, a narrowed provenance check, merge review on
every PR (with a human-approved exemption as the only skip), gates in
`vrg-submit-pr` and `vrg-finalize-pr`, a protected core of CI-wiring paths
needing a human protected-change approval, and a daily audit of every
merge into `develop` and `main`.

**What shipped:** all of it except the protected core, which the human
removed mid-epic (ruling N0.12, §1). The agent now opens, arms auto-merge
on and merges its own PRs into `develop` through the gated path, and
`vrg-gh` refuses its `pr create` and `pr merge` outside it. The first
agent-submitted PR was vergil-project/vergil-fixture-python-lib PR #87, on
24 Nov. The six fixture-run change PRs that followed were implemented,
merge-reviewed, submitted and merged by the agent with no human step, and
all six release-and-validation trios passed (ts-lib on its second release,
`v0.1.11`). The `main` ruleset is live, and the App keys exist only in the
`automation` environment. The user App holds `workflows: write` and
`pull_requests: write`. The daily agent audit runs in all 23 in-scope repos
(vergil-project/.github audits `develop` only, since it has no `main`).
Along the way the human's own submit-time skips, `--human-only` and
`--override-merge-review`, were removed too, so merge-review coverage is
now a merge-review record or an approved exemption, and nothing else. The
consumer `CLAUDE.md` template and this repo's steering describe the new
boundary.

| | |
| --- | --- |
| **Tasks** | 124 closed under the epic and its two proxies, plus this retrospective; another 20 vergil-project tasks were filed under each repo's ad-hoc epic once the epic hit GitHub's 100-sub-issue limit (14 Task 13 callers, by ruling N0.15, and 6 documentation-sweep tasks, by ruling N0.35) |
| **PRs** | 114 merged, plus this retrospective's. One is a batch PR covering two tasks (vergil-project/vergil-tooling#5120). The human submitted 99; the agent submitted 15, every one from 24 Nov 19:58 UTC on |
| **Repos touched** | 23: in vergil-project, `.github`, `docs`, `packages`, vergil-tooling, vergil-actions, vergil-claude-plugin, vergil-containers, vergil-python, vergil-vm and the six language fixtures; in logical-minds-foundry, `.github`, `docs`, mq-gateway-replay-lab, mq-resiliency-lab-for-linux and mq-resiliency-observability; in mnemosys-project, `.github`, `docs` and melete |
| **Releases named by the epic's deployment tasks** | vergil-tooling v2.1.339 (Tasks 2d and 3f), v2.1.340 (7d), v2.1.342 (17d, the gates), v2.1.345 (the own-merge fix Task 18d waited on) and v2.1.346 (16a's template); vergil-actions v2.1.93 (3r) and v2.1.94 (13r); vergil-claude-plugin v2.1.88 (release A) and v2.1.93 (release B); fixtures python-app v0.1.7, python-lib v0.1.8, rust-app v0.1.8, rust-lib v0.1.9, ts-app v0.1.10, ts-lib v0.1.10 and v0.1.11. The ts-lib release fixes shipped in vergil-tooling v2.1.347 (vergil-project/vergil-tooling#5153) and vergil-actions v2.1.98 (vergil-project/vergil-actions#1439) |

### Work delivered

The **Task** column is the plan's task number, or the ruling that added the
task. "Ad hoc" marks a task filed under its repo's ad-hoc epic, outside
this epic's tree. **Submitted by** is the PR's author: before Task 18d's
grant every PR was the human's; after it, the agent's.

vergil-tooling (refs are in vergil-project/vergil-tooling):

| Task | Issue | PR | What it did | Merged | Submitted by |
| --- | --- | --- | --- | --- | --- |
| 2 | #4772 | #5061 | The `main` ruleset and the audit-marker rules in the repo config | 2026-11-23 | human |
| 3b | #4773 | #5062 | The repo config converges the `automation` environment | 2026-11-23 | human |
| 3e | #4774 | #5074 | `secrets: inherit` allowed only for the App-key workflows | 2026-11-23 | human |
| 4 | #4775 | #5063 | The protected core and its match (later removed by #5117) | 2026-11-23 | human |
| 5 | #4776 | #5064 | A rebase-stable change fingerprint | 2026-11-23 | human |
| 6a | #4777 | #5075 | Protected-change and merge-review exemption request kinds | 2026-11-23 | human |
| 6b | #4778 | #5084 | Verification of past approvals | 2026-11-23 | human |
| 7 | #4779 | #5076 | A machine-readable merge-review closing record | 2026-11-23 | human |
| 10 | #4781 | #5065 | Provenance lets the user agent act on its own PR | 2026-11-23 | human |
| 3c | #5060 | #5088 | This repo's callers read the App keys from the environment | 2026-11-23 | human |
| 8 | #4780 | #5097 | One evaluator for approvals and merge-review coverage | 2026-11-24 | human |
| 9 | #4785 | #5112 | Agents submit through the gates | 2026-11-24 | human |
| 11 | #4786 | #5113 | Finalize re-checks the gates on every new head | 2026-11-24 | human |
| 12 | #4787 | #5111 | `vrg-agent-audit`, the daily audit of agent merges | 2026-11-24 | human |
| 16a | #4789 | #5148 | Consumer `CLAUDE.md` template and permission-model spec on the new boundary | 2026-11-24 | human |
| 12b, fix (N0.10) | #5059, #5108 | #5120 (batch) | Repo init creates the audit markers and passes the App keys by name outside vergil-project | 2026-11-24 | human |
| fix (N0.4) | #5095 | #5099 | The release merges into `main` with the human's ruleset bypass | 2026-11-24 | human |
| removal (N0.12) | #5117 | #5119 | The protected-change approval and the human's merge-review skips removed | 2026-11-24 | human |
| fix (N0.18) | #5131 | #5132 | The audit skips an audited branch the repo does not have | 2026-11-24 | human |
| fix (N0.22) | #5140 | #5144 | Provenance allows the agent's own merge close; finalize's CD check gets the agent's token | 2026-11-24 | human |
| 13 caller (ad hoc) | #5126 | #5129 | The daily agent audit in this repo's `ops.yml` | 2026-11-24 | human |
| fix (N0.29) | #5153 | #5170 | Resume a release whose PR has already merged | 2026-11-25 | human |
| sweep fix (ad hoc, N0.35) | #5176 | #5177 | A refused workflow push is a failure that names the missing grant | 2026-11-25 | agent |
| doc review | #4751 | #5179 | vergil-tooling's docs swept for the new boundary | 2026-11-25 | agent |

vergil-actions (refs are in vergil-project/vergil-actions):

| Task | Issue | PR | What it did | Merged | Submitted by |
| --- | --- | --- | --- | --- | --- |
| 3a | #1339 | #1389 | Reusable workflows read write-capable secrets from the `automation` environment | 2026-11-23 | human |
| added | #1387 | #1392 | The config job's update token gets Actions read, to audit the environment | 2026-11-23 | human |
| 3c | #1388 | #1398 | This repo's callers read the App keys from the environment | 2026-11-23 | human |
| 13 (A) | #1341 | #1409 | The reusable daily agent-audit workflow | 2026-11-24 | human |
| added (3d) | #1403 | #1404 | The configuration docs place the App keys in the environment | 2026-11-24 | human |
| fix | #1428 | #1429 | `ops-upstream-refresh` accepts environment-only App keys (a Task 3d regression) | 2026-11-24 | human |
| fix | #1432 | #1434 | `ops/report` refuses an empty App-key input by name | 2026-11-24 | human |
| 13 caller (ad hoc) | #1414 | #1416 | The daily agent audit in this repo's `ops.yml` | 2026-11-24 | human |
| fix (N0.29) | #1439 | #1440 | A re-run of a published release attaches its missing evidence or fails | 2026-11-25 | human |
| doc sweep (ad hoc, N0.35) | #1445 | #1447 | The App-key secret routes described as built | 2026-11-25 | agent |

vergil-claude-plugin (refs are in vergil-project/vergil-claude-plugin):

| Task | Issue | PR | What it did | Merged | Submitted by |
| --- | --- | --- | --- | --- | --- |
| 14 | #1200 | #1298 | Hooks point agents at `vrg-submit-pr` | 2026-11-23 | human |
| 15a | #1201 | #1306 | `merge-review` writes records, approvals and exemptions | 2026-11-23 | human |
| 15f | #1295 | #1299 | `hoshin` reads the agent audit as a security signal | 2026-11-23 | human |
| 3c | #1297 | #1311 | This repo's callers read the App keys from the environment | 2026-11-23 | human |
| 15e | #1202 | #1312 | Docs, retrospective and logbook PRs run merge review | 2026-11-24 | human |
| 15b | #1204 | #1357 | `issue-implement` owns the PR to the merge; one shared watch procedure | 2026-11-24 | human |
| 15c | #1206 | #1358 | Epic and fixture skills submit their own PRs | 2026-11-24 | human |
| removal (N0.12) | #1326 | #1327 | The protected-change approval removed from `merge-review` | 2026-11-24 | human |
| added (N0.28) | #1363 | #1364 | `fixture-run`'s ready-to-start conditions from the six runs | 2026-11-24 | human |
| 13 caller (ad hoc) | #1333 | #1338 | The daily agent audit in this repo's `ops.yml` | 2026-11-24 | human |
| doc sweep (ad hoc, N0.35) | #1374 | #1375 | The plugin docs describe the agent-submitted PR boundary | 2026-11-25 | agent |

vergil-project/.github:

| Task | Issue | PR | What it did | Merged | Submitted by |
| --- | --- | --- | --- | --- | --- |
| 1 | #1385 | #1420 | Spec and plan | 2026-11-19 | human |
| 1b | #1551 | #1554 | Spec and plan rebased onto the recent epics | 2026-11-23 | human |
| 1c | #1556 | #1557 | Audit markers created by repo init | 2026-11-23 | human |
| plan record (N0.1) | #1558 | #1559 | Task 6a depends on Tasks 4 and 5 | 2026-11-23 | human |
| 3c | #1563 | #1575 | This repo's callers read the App keys from the environment | 2026-11-23 | human |
| plan record (N0.2) | #1564 | #1565 | First execution pass in the Evolution log | 2026-11-23 | human |
| 13 caller | #1411 | #1602 | The daily agent audit in this repo's `ops.yml` | 2026-11-24 | human |
| plan record (N0.3) | #1578 | #1580 | Second execution pass in the Evolution log | 2026-11-24 | human |
| amendment (N0.12) | #1588 | #1589 | The protected core's approval removed from the spec and plan | 2026-11-24 | human |
| plan record | #1607 | #1613 | Third execution pass in the Evolution log | 2026-11-24 | human |
| plan record (N0.34) | #1641 | #1643 | Fourth execution pass in the Evolution log | 2026-11-25 | agent |

Task 16b (vergil-project/.github#1414) has no PR of its own: by ruling
N0.24 it became the template sync, which the human's config update merged
as vergil-project/.github#1626.

The other vergil-project repos:

| Task | Issue | PR | What it did | Merged | Submitted by |
| --- | --- | --- | --- | --- | --- |
| 3c | vergil-containers#1014 | vergil-containers#1023 | Callers read the App keys from the environment | 2026-11-23 | human |
| added (N0.16) | vergil-containers#1030 | vergil-containers#1032 | Test allows the App keys by name to this repo's own workflow | 2026-11-24 | human |
| 13 caller (ad hoc) | vergil-containers#1025 | vergil-containers#1027 | The daily agent audit | 2026-11-24 | human |
| doc sweep (ad hoc) | vergil-containers#1046 | vergil-containers#1047 | `GHCR_CLEANUP_TOKEN` documented as environment-only | 2026-11-25 | agent |
| 3c | packages#63 | packages#65 | Callers read the App keys from the environment | 2026-11-23 | human |
| 13 caller (ad hoc) | packages#67 | packages#68 | The daily agent audit | 2026-11-24 | human |
| doc sweep (ad hoc) | packages#70 | packages#72 | The README describes the `secrets: inherit` route as built | 2026-11-25 | agent |
| 3c | docs#171 | docs#174 | Callers read the App keys from the environment | 2026-11-23 | human |
| 13 caller (ad hoc) | docs#182 | docs#184 | The daily agent audit | 2026-11-24 | human |
| doc sweep (ad hoc) | docs#189 | docs#190 | The CI docs describe the secret routes as built | 2026-11-25 | agent |
| 3c | vergil-python#98 | vergil-python#100 | Callers read the App keys from the environment | 2026-11-23 | human |
| 13 caller (ad hoc) | vergil-python#102 | vergil-python#103 | The daily agent audit | 2026-11-24 | human |
| 3c | vergil-vm#447 | vergil-vm#452 | Callers read the App keys from the environment | 2026-11-23 | human |
| 13 caller (ad hoc) | vergil-vm#454 | vergil-vm#455 | The daily agent audit | 2026-11-24 | human |

The six language fixtures (`vergil-fixture-<name>`), three PRs each, plus
Task 18d's probe:

| Fixture | 3c caller (issue → PR) | 13 caller, ad hoc (issue → PR) | 19 change (issue → PR, agent-submitted) |
| --- | --- | --- | --- |
| python-app | #68 → #75 | #77 → #78 | #30 → #80: the `local` subcommand |
| python-lib | #75 → #82 | #84 → #85 | #36 → #89: `epoch_version` |
| rust-app | #83 → #90 | #92 → #93 | #45 → #95: the `mode` subcommand |
| rust-lib | #89 → #96 | #98 → #99 | #46 → #101: `last_word` |
| ts-app | #86 → #93 | #94 → #95 | #47 → #97: the `capitalize` subcommand |
| ts-lib | #85 → #92 | #93 → #94 | #46 → #96: `repeatedWords` |

The 3c and 13 PRs merged on 23 and 24 Nov, submitted by the human; the
change PRs merged on 24 Nov, submitted by the agent. Task 18d's probe,
vergil-project/vergil-fixture-python-lib#86, became PR #87, the first
agent-submitted PR (24 Nov, 19:58 UTC).

logical-minds-foundry and mnemosys-project, under the epic's two proxies
(logical-minds-foundry/.github#422 and mnemosys-project/.github#122). Every
one of these 25 PRs was submitted by the human, on 23 Nov (3c) and 24 Nov
(the rest):

| Repo | 3c caller (issue → PR) | App keys by name, N0.7 (issue → PR) | 13 caller (issue → PR) |
| --- | --- | --- | --- |
| logical-minds-foundry/.github | #423 → #425 | #426 → #427 | #428 → #429 |
| logical-minds-foundry/docs | #40 → #42 | #43 → #44 | #45 → #46 |
| logical-minds-foundry/mq-gateway-replay-lab | #346 → #348 | #349 → #350 | #351 → #352 |
| logical-minds-foundry/mq-resiliency-lab-for-linux | #1757 → #1759 | #1760 → #1761 | #1762 → #1763 |
| logical-minds-foundry/mq-resiliency-observability | #87 → #90 | #93 → #94 | #95 → #96 |
| mnemosys-project/.github | #123 → #125 | #126 → #127 | #128 → #129 |
| mnemosys-project/docs | #35 → #37 | #38 → #39 | #40 → #41 |
| mnemosys-project/melete | #278 → #281 | #282 → #283 | #284 → #285 |

Plus logical-minds-foundry/mq-resiliency-observability#91 → #92, the
`docs` caller Task 3c missed (a Task 3d ruling).

**Operational tasks, with no PR:** the deployments 2d
(vergil-project/vergil-tooling#4782), 3r (vergil-project/vergil-actions#1340),
3f (vergil-project/vergil-tooling#4783), 3d (vergil-project/.github#1410),
3g (vergil-project/.github#1415), 7d (vergil-project/vergil-tooling#4784),
13r (vergil-project/vergil-actions#1342), 15r
(vergil-project/vergil-claude-plugin#1203), 17d
(vergil-project/vergil-tooling#4788), 12d (vergil-project/.github#1412),
18d (vergil-project/.github#1413) and 21d
(vergil-project/vergil-claude-plugin#1207); the validation 19v
(vergil-project/vergil-fixture-python-lib#39); and the twelve fixture-run
release and validate tasks. Every one closed with `Outcome: SUCCESS`.
Closed as not planned: Task 13 (docs) (vergil-project/docs#145) and Task
15d (vergil-project/vergil-claude-plugin#1205) by ruling N0.12, and
vergil-project/vergil-actions#1408 by ruling N0.8.

## 1. How the plan evolved

The plan's order held: the standalone hardening first (the `main` ruleset
and the secrets moved into an environment), then the records, then the
gates, then the permission grants, then the skills and steering, with the
fixture runs as the first live agent cycle. What changed was the content
of the middle, and the number of tasks: the plan's Evolution log records
five execution passes, and the epic grew from its filed tasks to 124
children and 20 more under the repos' ad-hoc epics.

**The protected core came out.** The spec's largest single mechanism, a
protected-path set whose changes needed a human protected-change approval
recording each file's blob hash, was built (Tasks 4, 6a, 8, 9, 15a) and
then removed by ruling N0.12 on 24 Nov. The reasoning was the spec's own:
§1.1 argues that an approval given without looking protects nothing, and
an approval on every workflow and `vergil.toml` change, in an epic whose
aim is to trust the agents more, would be exactly that. Workflow changes
are worked out in specs and plans and judged by merge review like any
other. Removing it took three tasks (vergil-project/vergil-tooling#5117,
vergil-project/vergil-claude-plugin#1326, vergil-project/.github#1588),
dropped two (Task 15d and Task 13's docs registry) and narrowed several
more (Tasks 6a, 8, 9, 11, 12, 13 and 19v). The same ruling's addition
removed the human's two submit-time skips, since agents now write every
PR. No vergil-tooling release shipped until the removal merged, and it
shipped in the same release (v2.1.342) as the submit, watch and audit
gates, so no released gate ever enforced the approval. The
deferred question of protected files in planning is
vergil-project/.github#1590.

**`secrets: inherit` was wrong twice before probes settled it.** The plan
moved every caller from explicit App-key maps to `secrets: inherit` (Task
3c, 23 tasks), on the premise from vergil-project/packages#6 that only
`inherit` carries environment secrets into a reusable workflow. After the
move, the cross-org callers broke: `inherit` does not cross orgs. The
first fix proposed (N0.6, making the keys optional) rested on a second
wrong premise and was declined (N0.8). The callers in the other two orgs
then passed the keys by name (N0.7, eight tasks), and a probe
(logical-minds-foundry/mq-gateway-replay-lab run 45390389192, green with
the org keys hidden) showed that keys passed by name do reach a called
workflow's environment-declaring jobs. That superseded the packages#6
premise and answered epic vergil-project/.github#1494's question about a
repo's own local workflow (N0.16). Task 3d still caused one regression
(vergil-project/vergil-actions#1428) and a follow-on hardening
(vergil-project/vergil-actions#1432).

**Acceptance checks the plan could not run were re-ruled.** Several
checks needed an identity, a PR or a risk the plan had not provided for:
Task 2d's live refusal test (dropped as overkill), Task 17d's "an agent
submit fails at GitHub" (N0.17, then narrowed by N0.19 to no before-grant
check at all), Task 18d's test PR (a probe task, N0.20) and its `main`
refusal (covered by existing evidence, N0.23), Task 21d's live cycle
(handed to Task 19, N0.25) and Task 19v's case 3 (rescoped twice, N0.26
and N0.32, to the wrapper's refusals). Each ruling recorded what still
showed the boundary worked.

**Tasks started early or waited, by ruling.** Task 6a waited for Tasks 4
and 5, whose interfaces it consumed (N0.1). Tasks 9, 11 and 12 started
before Task 15r closed, since 15r's acceptance needed the merge review
they would run (N0.5). Task 16b became the template sync of 16a (N0.24),
so the steering changed no earlier than the skills.

**The fleet's limits showed up as tasks.** GitHub's 100-sub-issue limit
put the 14 vergil-project Task 13 callers (N0.15) and the six
documentation-sweep tasks (N0.35) under the repos' ad-hoc epics, where the
audit, the rollup and this retrospective's gate do not see them. Releasing
under the `main` ruleset needed the human's bypass in `vrg-release` (N0.4).
A repo with no `main` needed the audit to skip the missing branch (N0.18).
The out-of-scope orgs' installations took both grants, and the human
accepted that risk until they are renovated (N0.13, N0.21).

**The first live cycles found real defects, and they were fixed in the
epic.** The agent's first own merge tripped the provenance check on the
`closed` event GitHub records with every merge, and finalize's CD check
had no credentials in agent mode (N0.22, vergil-project/vergil-tooling#5140,
fixed before Task 18d closed). The ts-lib release failed after tagging, and
its re-run reported success without evidence: two defects were fixed and
released (N0.29, N0.30), but the evidence of `v0.1.10` had already been
overwritten, so an agent repaired `develop` and the human cut `v0.1.11`
(N0.31).

## 2. Lessons learned

**The human was the release lock, and nobody had written it down.** With
only the human merging into `develop`, `develop` never moved during a
release, because the human was running the release. Once agents merge
their own PRs, `develop` can move while a release is in flight. The first
symptom came within a day: another session's cleanup removed the in-use
release worktree from under `vrg-release` (intake
vergil-project/vergil-tooling#5187). This is the epic's main finding. An
implicit guarantee held by a person's attention disappears when the person
steps out of the loop, so moving a human out of a path means listing what
the human was silently guaranteeing first. The follow-ups are planning
issues vergil-project/.github#1662 (a release lock) and
vergil-project/.github#1663 (release blocks and PR sets across epics).

**Probe a platform premise before building on it.** The `secrets: inherit`
premise came from an earlier probe in one org and was carried into a plan
for three. It was wrong across orgs, and the first correction was wrong
too. One cheap probe with the org copies hidden settled what reading the
docs had not. Next time, a plan that rests on GitHub's secret, permission
or event behaviour names the probe that shows it, and runs it before the
fan-out tasks are filed.

**Plan the acceptance evidence with the identities that can produce it.**
Six acceptance checks were re-ruled because no task could produce the
evidence: a deployment opens no PR, the agent cannot use an identity it
does not hold, and a refusal on `main` cannot be tested without risking a
real commit. Where a check needs a PR, a probe task belongs in the plan,
as N0.20 added. Where it needs a refusal, the cheapest honest check is
often the wrapper's own refusal (N0.32).

**Count the sub-issues before filing.** A fleet-wide epic with a per-repo
fan-out (23 callers for Task 3c, 23 for Task 13, plus their fixes) reaches
GitHub's 100-child limit well before it finishes. An interrupted filing of
the Task 13 callers also left a duplicate set, which had to be closed and
unlinked by hand. The capacity work is epic
vergil-project/.github#1609.

**Exact-time assumptions about GitHub's events break.** The own-merge
allowance first missed the `closed` event altogether, then required it in
the same second as the merge; GitHub sometimes records it a second later
(vergil-project/vergil-tooling#5178, fixed with a bounded window). A rule
about the order of GitHub's events needs a tolerance, and a fixture taken
from a real timeline.

**Ask of the largest mechanism whether it earns its cost.** The six
planning rounds raised 90 findings and accepted all of them. They narrowed
the protected core to CI wiring and refined its approval, but kept it; the
question of whether it protected anything was settled by the human on the
second day of execution, after the seven tasks that built it had merged. Next time, a
pushback round asks of the design's largest mechanism, by name, whether
the epic still needs it.

**Record every ruling at once.** N0.17 was given in the driver's session
and posted only when a later deploy run asked for it. The rulings on the
epic are the record every later task reads, so a ruling not posted is a
ruling a sub-agent cannot see.

## 3. Compromises and tradeoffs

- **The out-of-scope orgs carry accepted risk.** The user App's
  `workflows: write` and `pull_requests: write` were accepted on every
  installation, including three orgs outside the epic's scope, which still
  keep App-key secrets at org level and have no agent audit. The human
  accepted the risk until those orgs are renovated (N0.13, N0.21).
- **Live checks were traded for evidence.** The before-grant refusal check
  was dropped (N0.19). The `main` refusal rests on the ruleset, the config
  audit and the agent audit rather than a live push (N0.23). Task 21d's
  live cycle was delegated to Task 19 (N0.25). Task 19v's case 3 checks the
  wrapper's refusals rather than the audit reporting an uncovered merge by
  name (N0.32), so the audit's uncovered-merge report has not been shown
  on a live merge. A coverage follow-up in the audit is a private advisory,
  tracked by vergil-project/.github#1640 (N0.33).
- **Part of the epic lives outside its tree.** The 14 Task 13 callers and
  the six sweep tasks under the repos' ad-hoc epics are invisible to the
  epic's audit, rollup and this retrospective's gate; the plan's Evolution
  log is their only index.
- **A fixture release shipped without its evidence.** vergil-fixture-ts-lib
  `v0.1.10` stays published without an evidence bundle (N0.31); `v0.1.11`
  is the clean release.
- **The plan keeps superseded text.** The Goal, the Architecture paragraph
  and several task bodies still describe the protected-change approval or
  the old `inherit` premise. The Evolution log names each passage as
  superseded rather than rewriting the plan, so it stays a record of what
  was planned.
- **Some fixes went beyond the hostile-agent anti-goal, and one was
  declined under it.** Task 9's merge review led to the gates running on
  every submit, which defends against more than the spec's non-goal, and
  the human kept it. vergil-project/vergil-vm#453 was closed as not planned
  under the same non-goal; its record, vergil-project/.github#1586, stays
  as an accepted property of the VM design.
- **`hoshin` keeps the human's submit.** The logbook PR gets merge review
  but stays human-submitted, as its sign-off (spec §4.10, corrected on the
  epic by vergil-project/.github#1641). `hoshin` is new, and this is worth
  revisiting once its cadence has settled.

## 4. New problems and opportunities

**Follow-ups the epic filed or found** (state as of this retrospective):

| Problem or opportunity | Where it went |
| --- | --- |
| A release can run while agents merge into `develop` | vergil-tooling#5187 (intake); planning .github#1662 (release lock) and .github#1663 (release blocks and PR sets) |
| GitHub's 100-sub-issue limit | vergil-tooling#3840 (closed); epic .github#1609 |
| Protected files acknowledged in planning, merge review stopping only on unplanned changes | planning .github#1590 |
| Signed packages from repos outside vergil-project | .github#1592 (N0.10's follow-up) |
| The own-merge close recorded a second after the merge | vergil-tooling#5178 (fixed) |
| A watch error reported as a release failure | vergil-tooling#5174 (intake, N0.31) |
| A CI re-run overwrites the earlier attempt's evidence | vergil-actions#1444 (intake, N0.31) |
| "not-accepted" content reported as "skipped untrusted" | vergil-tooling#5154 (intake) |
| Auto-merge found off on a PR with no violation | .github#1598 (intake) |
| Interactive apt prompts hang builds | .github#1577 (intake) |
| GitHub rate limits across the fleet | .github#1568 (intake) |
| Code scanning's Semgrep OSS check duplicates our job | .github#1600 (intake) |
| `fixture-run` should name exact plugin and tooling versions | vergil-claude-plugin#1373 (intake, N0.34) |
| The release and ops workflow docs still show the old secret model | vergil-actions#1418 |
| The deprecated `app-id` input | vergil-actions#1446 (intake, N0.35) |
| A coverage follow-up in the agent audit | private advisory, tracked by .github#1640 |
| Fixture-scoped agent rights, now rethinkable on this boundary | planning .github#1088 |

All refs are in vergil-project.

**The unaddressed-findings sweep.** It read every child of the epic other
than this retrospective (124 tasks, the proxies' included), the 20 tasks
filed under the repos' ad-hoc epics, the epic issue, 114 merged PRs (their
conversations, reviews and inline review comments) and the planning
issue's six review rounds. Every read went through `vrg-issue-read`; none
reported skipped content. Every `Awaiting a ruling` item carries a closing
ruling, every `Filed without a ruling` list is empty, and the three
security items raised on task issues are closed by their counts rulings.
All 90 planning findings were accepted into the spec or plan. Four
problems were found untracked, three of them from the driver's session
rather than the task records; the human ruled `file` on each:

- **PRs opened by the agent's submit carry `Closes #N`, not the default
  `Ref`.** Seen on vergil-project/vergil-fixture-python-lib PRs #96 and #97
  (19v's result), with `--linkage` left at its default. Filed as intake
  vergil-project/vergil-tooling#5191.
- **Skills that dispatch sub-agents fail inside the Workflow tool's
  agents,** which lack the Agent tool, so merge review could not run there
  and the runs were resumed as plain agents. Filed as intake
  vergil-project/vergil-claude-plugin#1382.
- **`vrg-finalize-pr` refuses to run from a worktree without saying to run
  it from the main clone.** Filed as intake
  vergil-project/vergil-tooling#5192.
- **Human-run steps were handed over interleaved with the agent's text.**
  The human asked for them as plain copy-paste blocks for the human's own
  terminal, never with the `!` prefix. Filed as intake
  vergil-project/.github#1667.

## 5. What's next

- **The release lock.** vergil-project/.github#1662 and #1663 take up the
  main finding; vergil-project/.github#467 (release holds) is related.
- **Sub-issue capacity.** Epic vergil-project/.github#1609.
- **Protected files in planning.** vergil-project/.github#1590.
- **Fixture-scoped agent rights.** vergil-project/.github#1088, deferred
  until this epic landed, can now be rethought on top of it.

## Appendix A — Operational notes

The rollout order that made the boundary move safe, as it ran:

1. **Hardening (23 Nov).** The `main` ruleset (2, 2d), the `automation`
   environment (3a, 3b, 3e, 3r, 3f), the 23 caller changes (3c), then the
   secrets moved and their org- and repo-level copies deleted (3d, 24 Nov).
   One tooling release (v2.1.339) and one converge per repo covered both 2d
   and 3f. The cross-org callers needed the by-name keys (N0.7) before the
   deletes.
2. **Records (23–24 Nov).** Fingerprint, request kinds, verifier and
   merge-review record (5, 6a, 6b, 7), released as v2.1.340 (7d); plugin
   release A (v2.1.88, 15r), whose merge reviews write the records.
3. **Gates (24 Nov).** Evaluator, submit, provenance, watch, audit (8–12),
   with the approval removed first (#5117), released as v2.1.342 (17d). The
   audit workflow (13, 13r at vergil-actions v2.1.94), its callers, and the
   go-live: the human set the markers in 22 repos, and all 23 audits ran
   green on 24 Nov (12d).
4. **Grants (24 Nov).** `pull_requests: write` (18d), proven by the probe
   PR, after the own-merge fix (v2.1.345); `workflows: write` (3g).
5. **Skills and steering (24 Nov).** 15b, 15c and 16a, then plugin release
   B (v2.1.93) with tooling v2.1.346 and the template sync (21d, 16b).
6. **First live cycles (24–25 Nov).** The six fixture runs (19) and the
   exemption and refusal checks (19v), then the documentation sweep, whose
   tasks the agent submitted itself.
