# Report-only Dependabot Alerting — Retrospective

Epic: [vergil-project/.github#420](https://github.com/vergil-project/.github/issues/420).
Spec: [`spec.md`](spec.md). Plan: [`plan.md`](plan.md).

## 0. At a glance

We set out to watch every fleet repo for vulnerable dependencies, including
dormant ones, without Dependabot ever opening a PR. Alerts are on, security-update
PRs are off, every repo carries a `dependabot.yml` with
`open-pull-requests-limit: 0`, and the nightly ops run fails on any open alert.
All of that shipped and is enforced across all 18 inventory repos in three
orgs. The first nightly run found 16 real alerts in two repos, and both were
fixed. The epic also exposed a gap in the primary control, the per-change
`vrg-validate` audit, which was closed for Python.

| | |
|---|---|
| Span | 2026-10-22 → 2026-10-24 (3 days) |
| Epic tasks | 27 (26 closed + this retrospective) |
| PRs merged | 45: 22 on epic tasks, plus 23 filed under the per-repo ad-hoc epics of the other two orgs (see below) |
| Repos touched | 18: vergil-project (9), logical-minds-foundry (6), mnemosys-project (3) |
| Releases carrying epic work | vergil-tooling v2.1.256 (release A), v2.1.257, v2.1.258, v2.1.261 (release B), v2.1.263; vergil-actions v2.1.50, v2.1.52 |

### Work delivered

#### Core (vergil-tooling and vergil-actions)

| PR | Task | What it did |
|---|---|---|
| .github#423 | #421 | Spec and plan |
| vergil-tooling#3435 | #3426 | Alerts on in the desired repo config; new `vrg-dependabot-audit` (exit 0/1/2, counts-only on public repos) |
| vergil-tooling#3436 | #3427 | Canonical `dependabot.yml` renderer, scaffolding, `vrg-validate` limit-0 guard, `vrg-dependabot-sync`, fleet-sweep target |
| vergil-actions#1004 | vergil-actions#1000 | `dependabot-audit` job in the shared `ops-github-config.yml` |
| vergil-tooling#3490 | #3430 | Release B: nightly `local.dependabot` audit against the canonical render |
| vergil-tooling#3450 | #3447 | Fleet sweep files its task under the repo's ad-hoc epic when the epic is in another org |
| vergil-tooling#3453 | #3451 | An unreadable `vergil.toml` is an error, not a silent "no language", in the managed-file renders |
| vergil-tooling#3457 | #3454 | `apply`'s CI-gates guard recognises local reusable-workflow calls (false refusal on vergil-actions) |
| vergil-tooling#3464 | #3463 | `current_repo(cwd=…)` mints its token for the clone's own org (private cross-org repos) |
| vergil-tooling#3508 | #3498 | The Python audit covers every `uv.lock` group, not just the synced `dev` environment |

#### Rollout

| What | PRs |
|---|---|
| `dependabot.yml` sweep, vergil-project (epic tasks) | .github#427, docs#42, packages#16, vergil-actions#1015, vergil-claude-plugin#702, vergil-containers#665, vergil-python#33, vergil-tooling#3449, vergil-vm#354 |
| `dependabot.yml` sweep, other orgs (ad-hoc epics) | lmf/.github#379, lmf/docs#19, mq-resiliency-lab-for-linux#1688, mq-resiliency-observability#53, mq-gateway-replay-lab#318, mq-protocol-gateway#28, mnemosys/.github#93, mnemosys/docs#14, melete#248 |
| App permission ("Dependabot alerts: Read") walk-through | vergil-actions#999 (deployment, no PR) |
| Fleet `diff` + `apply` | vergil-tooling#3428 (deployment, no PR) |
| End-to-end validation | vergil-actions#1001 (validation, no PR) |

#### Problems the rollout surfaced and fixed (ad-hoc epics)

| PR | What it did |
|---|---|
| mnemosys/docs#16, mnemosys/.github#97 | CI never started (`startup_failure`): `ci.yml` still passed inputs the v2.1 reusable workflows had removed |
| melete#250 | urllib3 / certifi past published advisories (CI audit was red) |
| melete#252, mnemosys/.github#100, mnemosys/docs#18, mq-resiliency-observability#55, mq-gateway-replay-lab#321 | Local repo-config drift (Claude attribution block, `.gitignore` fence) |
| vergil-containers#676 | 13 alerts in the ts-node smoke fixture (vitest 5) |
| mq-resiliency-lab-for-linux#1691 | 3 alerts in the `docs` group (mkdocs-material, pymdown-extensions) |
| mq-resiliency-observability#57, mq-gateway-replay-lab#324, melete#254, mq-resiliency-lab-for-linux#1693 | filelock / setuptools advisories published the evening of 2026-10-23, fixed before Dependabot raised alerts |

#### Documentation

| PR | Task | What it did |
|---|---|---|
| vergil-tooling#3521 | #3423 | Documentation sweep: security posture, alert audit, `dependabot.yml` guards; replaced the non-existent `vergil-release[bot]` with the per-org automation Apps |
| vergil-actions#1046 | vergil-actions#1045 | App secrets and permissions page rewritten for the org-owned automation Apps; new `ops-github-config.yml` workflow page |
| docs#49 | docs#48 | Nightly-governance summary: three jobs, report-only posture, correct "audits, doesn't reconcile" wording |

## 1. How the plan evolved

The plan held up structurally. Its four parts, three deployments and one
validation all ran in the order it set out, and no task was dropped. What
changed was the volume of unplanned work, and almost all of it came from the
rollout meeting the real fleet, not from the design.

- **The fleet is three orgs, and the tooling mostly assumed one.** The plan
  treated the sweep and `apply` as uniform loops. In practice
  logical-minds-foundry and mnemosys-project broke three single-org assumptions:
  - the sweep tried to file its tasks under an epic in another org (#3447);
  - private cross-org clones minted tokens for the wrong org (#3463);
  - `vrg-issue-create` refuses cross-org parenting, so every remediation task in
    those orgs lives under its own repo's ad-hoc epic.

  The epic's task graph therefore undercounts the work by about half (§0).
- **Dormant repos were broken, which was the epic's own thesis.** The mnemosys
  repos had gone about 60 days without a commit. Their CI had been failing to
  start since the v2.1 reusable workflows dropped two inputs, and nothing had
  noticed. Their Ops schedules had been disabled by GitHub's inactivity rule, so
  the nightly that this epic extends wasn't running there either.
- **The first nightly found real alerts.** vergil-containers had 13 and
  mq-resiliency-lab-for-linux had 3. Neither had been caught by `vrg-validate`.
  The lab's 3 sat in the `docs` dependency group, which pip-audit never saw
  because it audits only the synced `dev` environment. That turned the "defense
  in depth" framing into a concrete finding: the secondary control found a hole
  in the primary one. It was fixed in vergil-tooling (#3498, v2.1.263), and the
  same gap in the Java, TypeScript and C++ profiles was split off (#3499).
- **The App permissions were not what the plan said.** The plan expected the
  automation App to need Administration: Read & write for `apply`. The #999
  walk-through found that `apply` runs as the human, so the App rightly stays
  read-only. It also found that the docs described a single `vergil-release[bot]`
  App that doesn't exist; each org has its own `<org>-automation` App. Fixing
  that became most of the documentation sweep, which spawned two per-repo doc
  tasks.
- **Validation was wider than planned.** The plan sampled one repo per account.
  Validation checked all 18, because the extra runs cost little once the fleet
  could be dispatched in one loop.

## 2. Lessons learned

- **Ship the report-only alarm before you trust the per-change audit.** Within a
  day the nightly found alerts that every repo's green CI had missed. A gate is
  only as wide as its input set. The pip-audit scope gap was invisible until
  something with a different view of the same lockfile disagreed with it.
- **A nightly control is only as good as the schedule that runs it.** Five of 18
  repos' Ops schedules had silently stopped: two by GitHub's 60-day inactivity
  rule, three disabled by hand on about 2026-10-21 for an unknown reason. A
  stopped schedule looks exactly like a passing one. Nothing in the fleet watches
  for this yet (§4).
- **Exit 2 earned its keep on day one.** On the 2026-10-23 nightly,
  vergil-actions and melete hadn't been applied yet and exited 2 with
  `Dependabot alerts are disabled for …`. That was loud and correctly not a
  pass. Treating "could not check" as a failure meant the release-ordering slip
  in §3 was visible, not silent.
- **Advisory feeds are faster than Dependabot.** The filelock and setuptools
  advisories published on the evening of 2026-10-23 were found the next morning
  by querying OSV directly against every lockfile, hours before Dependabot raised
  alerts for them. Dependabot is a good backstop but not the fastest signal.
- **Cross-org is a first-class case.** Three bugs in one epic came from
  single-org assumptions. New fleet-wide tooling should be tested against a
  private repo in a second org before its first sweep.

## 3. Compromises & tradeoffs

- **The vergil-actions release went out ahead of its preconditions.** The plan
  made the Part 3 release wait for release A, #999 and the fleet `apply`.
  vergil-actions v2.1.50 was published at 19:56 UTC on 2026-10-22, about 25
  minutes before release A (v2.1.256) and a day before #999 and `apply`
  completed. The cost was the accepted red window in spec §6, on 2026-10-23:
  - Exit 2 on the two repos that weren't applied yet: vergil-actions and melete.
  - The rest of the fleet already had alerts on.
- **The exit-1 evidence predates the final release.** By the time validation
  ran, no repo had open alerts, so the "alerting repo exits 1" criterion uses the
  2026-10-23 scheduled runs on vergil-tooling 2.1.256 and 2.1.257. The audit
  command's source is unchanged since then, so it is the same code path, but it
  is not a fresh dispatch on an alerting repo.
- **Twelve of the eighteen "next nightly" results were manual dispatches.** #3428
  called for the next nightly. To close it the same morning, vergil-project and
  mnemosys-project were dispatched by hand. They run the same jobs against the
  same state, but they are not literally the scheduled run.
- **Scorecard was checked with the local CLI.** The public Scorecard API has no
  record for vergil-tooling (HTTP 404), so Dependency-Update-Tool was verified
  with `vrg-scorecard` (10/10), not the published viewer.
- **Some App facts are documented as unverified.** The docs make no claim about
  whether `APP_CLIENT_ID` / `APP_PRIVATE_KEY` are org or repo secrets, and they
  give the minimum permission set the workflows request rather than each App's
  actual settings. Agents can't read secrets or App settings, so those are left
  for the maintainer to confirm.
- **The Python audit gap is closed; the others are deferred.** #3498 fixed
  Python. The Java (no vulnerability scan at all), TypeScript (`--omit=dev`) and
  C++ gaps were split off to #3499 rather than held in this epic.
- **Remediation lives outside the epic graph.** Because of the cross-org
  parenting rule, 23 of the 45 PRs are under per-repo ad-hoc epics, so
  `vrg-epic-audit` on #420 shows only about half the work.

## 4. New problems & opportunities

| Surfaced | Where it went |
|---|---|
| Java, TypeScript and C++ audit-scope gaps | vergil-project/vergil-tooling#3499 (ad-hoc; brainstorm input) |
| MkDocs 2.0 breaks plugins; the fleet pins `mkdocs<2` with no exit plan | vergil-project/.github#436 (research) |
| Scheduled Ops workflows can silently stop (`disabled_inactivity`, `disabled_manually`), and nothing detects it | Logged, not yet acted on (brainstorm input) |
| Cause of the three logical-minds-foundry Ops workflows `disabled_manually` around 2026-10-21 | Unknown; re-enabled, not investigated |
| vergil-containers' nightly image rebuild fails its Trivy gate. On 2026-10-24 all 32 images were blocked by one finding: undici 6.29.0 bundled in npm 12.2.1 (CVE-2026-109869), which no npm release fixes yet. It is outside this epic, whose spec lists container pins as a non-goal | vergil-project/vergil-containers#677 |
| The CVE rate makes remediation a roughly weekly, sometimes daily, chore. Detection, diffing and drafting fixes are mechanical; deciding exploitability and suppression is not | Logged as a brainstorm input: semi-mechanized CVE triage |
| Runner annotations: `ubuntu-latest` moves to Ubuntu 26 from 2026-10-19; `create-github-app-token` deprecates `app-id` in favour of `client-id` | Logged, not yet acted on |

## 5. What's next

A follow-on brainstorm, not yet held, takes these inputs:

- vergil-project/vergil-tooling#3499: close the vulnerability-audit scope gaps in
  the remaining language profiles.
- vergil-project/.github#436: the fleet's path off MkDocs 1.x.
- Detecting a scheduled audit that has silently stopped running.
- Semi-mechanized CVE triage, for dependency alerts and container images alike:
  automate detection, diffing and the draft fix, and keep the exploitability and
  suppression call with a human.

## Appendix A — Operational notes

Rollout order as executed:

1. **2026-10-22.**
   - Released vergil-tooling v2.1.256 (release A) and vergil-actions v2.1.50
     (the ops job).
   - Swept the nine vergil-project repos.
   - Fixed #3447 and #3451.
2. **2026-10-23.**
   - Fixed #3454 and released v2.1.257 with all three fixes.
   - Ran the App permission walk-through (#999) on all three orgs' automation
     Apps.
   - Swept the logical-minds-foundry and mnemosys-project repos, after fixing
     #3463 (v2.1.258).
   - Fixed the mnemosys CI `startup_failure`s and melete's red audit so those
     sweep PRs could merge.
   - Released v2.1.261 (release B, `local.dependabot`).
   - Ran the fleet `diff` and `apply`, then fixed the local drift that `apply`
     reported.
   - Remediated vergil-containers and mq-resiliency-lab-for-linux.
   - Released v2.1.263 (all-groups Python audit).
   - Re-enabled the five stopped Ops schedules.
3. **2026-10-24.**
   - Ran the OSV sweep and the four filelock/setuptools fixes.
   - Ran the Ops audits on all 18 repos and closed #3428.
   - Closed the end-to-end validation (vergil-actions#1001).
   - Merged the documentation sweep and its two spawned doc tasks.

Gotchas:

- Agent pushes that touch `.github/workflows/**` are refused; `vrg-submit-pr`
  pushes those.
- `vrg-github-repo-config diff` needs Administration: Read, which the agent
  identity lacks, so the human ran `diff` and `apply`.
- A workflow GitHub disabled for inactivity must be re-enabled by hand.
  Dispatching it is not enough.
