# ops-issue-queue — retrospective

- **Epic:** vergil-project/.github#1494
- **Spec:** [`spec.md`](spec.md)
- **Plan:** [`plan.md`](plan.md), with its
  [`## Evolution during execution`](plan.md#evolution-during-execution) log
- **Planning issue:** vergil-project/.github#1470

**Span:**

| Window | Start | End | Elapsed |
| --- | --- | --- | --- |
| Whole epic | Sun 22 Nov 2026, 16:59 UTC | Wed 25 Nov 2026, 18:43 UTC | 3 d 1 h 43 min |
| Planning | Sun 22 Nov 2026, 12:29 UTC | Sun 22 Nov 2026, 20:19 UTC | 7 h 50 min |
| Implementation | Sun 22 Nov 2026, 20:19 UTC | Wed 25 Nov 2026, 18:25 UTC | 2 d 22 h 5 min |

## 0. At a glance

We set out to turn every failure of the fleet's unattended automation into
tracked work. Each failure would land in a per-org operations queue that the
morning review reads, instead of in a GitHub notification email. After that,
every unattended job would run as its org's automation App. **What shipped:**

- An `operations` standing epic exists in each of the four orgs.
- `vrg-ops-report` and the `ops-report` action are in use, and every
  in-scope shared and container job has a reporting job.
- hoshin's daily review reads the queue.
- The reporter was proven live: real ops issues were opened, recovered and
  closed (vergil-project/vergil-containers#1013, #1042, #1049).
- Every in-scope job runs as the App, except five recorded `GITHUB_TOKEN`
  exceptions, which GitHub's own limits forced (§3).

Six of the twenty caller repos left the epic as ad-hoc work.

**Work delivered** (PR, repo, what it did):

| PR | What it did |
| --- | --- |
| vergil-project/.github#1521 | Spec and plan (Task 1) |
| vergil-project/vergil-tooling#4974 | The `operations` standing epic kind, its labels, `--epic operations` (2a) |
| vergil-project/vergil-tooling#5024 | `vrg-ops-report`: open, comment, record recovery; trust-layer reads; never closes (2b) |
| vergil-project/vergil-tooling#5025 | PR tools refuse ops issues; the drift close and intake routing leave them alone (2c) |
| vergil-project/vergil-actions#1376 | The `ops-report` composite action; optional App secrets on the docs and index workflows (4) |
| vergil-project/vergil-actions#1380 | A reporting job in every shared unattended workflow (7) |
| vergil-project/vergil-containers#1009 | Reporting jobs for the five container jobs (8) |
| vergil-project/vergil-containers#1001 | Remove the report-only `pin-exposure` job (9) |
| vergil-project/vergil-claude-plugin#1289 | hoshin's daily review reads the operations queue (10) |
| vergil-project/vergil-containers#1031 | App-token dry run for package cleanup (15, split) |
| vergil-project/vergil-containers#1035 | Dev rebuild logs in to GHCR as the App (15, split); reverted by #1045 |
| vergil-project/vergil-containers#1045 | Revert the App GHCR login after the proof failed |
| vergil-project/vergil-containers#1048 | Container API jobs on the App; registry jobs kept as exceptions (17) |
| vergil-project/vergil-containers#1052 | Package cleanup back to a PAT, plus `verify-deletes` (#1050, #1051) |
| vergil-project/vergil-containers#1055 | Package cleanup deletes with `GITHUB_TOKEN` (#1054) |
| vergil-project/vergil-actions#1417 | Shared docs and index jobs on the App; the `VRG_AUTOMATION_APP` check in four jobs (18) |
| vergil-project/.github#1661 | Record the identity exceptions in the spec (19b) |
| vergil-project/.github#1552, #1555, #1618, #1676, #1679 | Five plan-evolution entries and spec corrections |
| vergil-project/vergil-tooling#5128, vergil-project/docs#185, vergil-project/vergil-actions#1427, vergil-project/vergil-containers#1028, vergil-project/vergil-claude-plugin#1339 | Documentation-review sweep and its four per-repo doc tasks |
| 14 caller PRs (6a–6o, less 6m) | Pass `APP_CLIENT_ID` / `APP_PRIVATE_KEY` to the shared docs workflows: vergil-project/docs#169, vergil-claude-plugin#1294, vergil-containers#1008, vergil-tooling#5058 (also the `vrg-repo-init` generator), vergil-vm#446, the six fixtures' #67, #74, #82, #88, #84, #85, logical-minds-foundry/docs#39, logical-minds-foundry/mq-resiliency-lab-for-linux#1756, mnemosys-project/docs#34 |
| 6 fixture change PRs (Task 14) | One change per fixture, released and validated against vergil-actions v2.1.92 |

- **Counts:** 71 tasks closed (the retrospective excluded) and 47 PRs merged
  (the tables above).
- **Repos touched:** 16 repos across 3 orgs (vergil-project, logical-minds-foundry,
  mnemosys-project). The fourth org, mq-rest-admin-project, got only its
  operations epic.
- **Releases the epic's tasks installed or validated:**
  - vergil-tooling v2.1.337 (Task 3)
  - vergil-actions v2.1.91 (Task 5), v2.1.92 (Task 12) and v2.1.95 (Task 19)
  - vergil-claude-plugin v2.1.87 (Task 10b)
  - vergil-containers v2.1.50 (Task 16b)
  - the six fixture releases: python-app v0.1.6, python-lib v0.1.7,
    rust-app v0.1.7, rust-lib v0.1.8, ts-app v0.1.9 and ts-lib v0.1.9

## 1. How the plan evolved

The reporter half of the plan ran close to its script. The identity half did
not. The plan's own ordering, reporter first and a GHCR proof deciding the
identity work, is what kept the identity half's problems from reaching the
reporter.

- **Day one (23 Nov): small corrections, then the fleet's state pushed
  back.**
  - Task 4 found that the existing identity test rejected the planned
    reporting-step convention, so Task 7 grew a test change.
  - Task 11's checks turned out to need a human identity.
  - The six caller repos in mq-rest-admin-project and
    `mq-resiliency-observability` could not pass `vrg-validate` because of
    problems already on their `develop` branches. The human moved them to
    ad-hoc work and accepted visible red reporting jobs there, rather than
    widening the epic into repo repair.
  - Task 6d grew to fix the `vrg-repo-init` generator, so new repos start
    correct.
- **Day two (24 Nov): validation on live evidence, and a dependency on epic
  1384.**
  - Task 13 closed on real ops traffic (#1013), not on failures caused on
    purpose.
  - A dependency on epic vergil-project/.github#1384 was raised and then
    withdrawn: the diagnosis had read the current workflow, not the one the
    failing runs used.
  - Task 15 split so its dry run could land while epic 1384 ruled on how a
    repo passes App keys to its own local workflow.
  - From then on the epic followed 1384's secret model (App keys only in
    the `automation` environment).
- **Day three (25 Nov): the identity track met GitHub's limits, twice.**
  - The GHCR proof failed: registry pushes refuse App installation tokens,
    and that held for 11 hours after the grant. The epic reverted the dev
    login and narrowed Task 17 to the API-only jobs.
  - The API-only `package-cleanup` move then failed live: every delete was
    answered "not found". It also showed that the action counts such
    answers as success, so `verify-deletes` was added.
  - The first fix returned cleanup to a PAT that turned out never to have
    existed. The second fix settled on `GITHUB_TOKEN`, which the last green
    run had in fact been using all along.
  - Each turn added tasks under the epic (#1044, #1050, #1051, #1054) and a
    plan-evolution task (five in all).

**The delta:**

- The reporter, the queue and the morning review match the spec.
- The identity goal shipped in a narrower form, with five exceptions
  instead of "every job on the App".
- Nine tasks were added during execution:
  - the revert and the three cleanup tasks (#1044, #1050, #1051, #1054);
  - #1029, split from Task 15;
  - the plan-evolution and spec-correction tasks #1549, #1553, #1617, #1671
    and #1675.
- Six caller tasks left the epic. None of the plan's Review Focus risks
  came up in production.

## 2. Lessons learned

- **A proof must exercise the operation the change depends on.** Task 16's
  dry run listed package versions and was read as proof of the API path.
  Listing is not deleting. A delete in the proof would have caught the
  failure before Task 17 shipped. (Recorded in the plan's fourth entry.)
- **Check a token against the secrets that exist and the run that last
  worked.** #1050 picked `GHCR_CLEANUP_TOKEN` without checking either. The
  last green run had resolved `GHCR_CLEANUP_TOKEN || GITHUB_TOKEN` to
  `GITHUB_TOKEN`. (Plan, fifth entry.)
- **A fallback hides which credential actually works.** The `||` fallback
  made a non-existent PAT look like the working credential for months. The
  no-fallback rule this epic applied to the docs deploy is the right
  default for credentials too.
- **Diagnose a failing run against the workflow that run used**, not the
  current file. The withdrawn dependency on epic 1384 cost a round trip.
- **Search where the thing lives.** An ops issue lives in its job's repo, so
  a search of only the `.github` repos missed #1013 and briefly misstated
  Task 13's evidence. Org-wide label searches are the right query.
- **Fleet health gates fleet-wide epics.** A plain caller pass across 20
  repos lost 6 to problems that were already on their `develop` branches.
  Next time, check each target repo's validation health when the plan is
  written, and plan the repair or exclusion up front.
- **Undocumented platform limits decide designs.** That GHCR refuses App
  tokens for registry pushes rests on the failures alone; GitHub documents
  it nowhere. The plan's choice to gate identity on a proof, and to keep it
  after the reporter, was right, and should be the pattern whenever a
  design rests on an unverified platform capability.
- **Validation that depends on a schedule needs a fresh run.** The six
  fixture validations judged `ops.yml` runs that predated the release, so
  the new reporting jobs never ran there until the human dispatched them.
  This is now tracked as vergil-project/vergil-claude-plugin#1302.
- **Same next time:**
  - The tiered order: reporter, then a proof, then the identity work.
  - Recording each change as it happened, with the plan-evolution log and
    a ruling for each follow-up.
  - Driving cross-org work through proxy epics.

## 3. Compromises & tradeoffs

- **Five `GITHUB_TOKEN` identity exceptions** (spec §4.1, §5.4):
  - the registry-push jobs `nightly-rebuild-dev`, `nightly-rebuild-prod`
    and `image-publish`;
  - `publish-index`'s Pages deploy;
  - `package-cleanup`.

  These are platform limits, recorded with their reasons, not chosen debt.
- **Six caller repos left the epic.** Until each one's caller task lands,
  its docs runs carry a red reporting job. Since vergil-actions v2.1.95, its
  docs deploy has also stopped, because the App-token deploy has no
  fallback. The human accepted this as a visible failure, not a hidden one.
- **Task 13 closed partly on unit tests.** Steps 1, 3, 4 and 5 rest on Task
  2b's tests and the fail-red rule, not on failures caused on purpose. Live
  traffic (#1013, #1042, #1049) has since exercised open, comment and
  recovery.
- **The App secrets stay optional** on the three shared docs and index
  workflows, so a caller that forgets them gets a red reporting job rather
  than an error at the call. Tracked: vergil-project/vergil-actions#1381.
- **Package cleanup's deleting jobs no longer declare the `automation`
  environment** (#1055), so they can be dispatched from any branch. Anyone
  who can dispatch from a branch can already edit that branch's workflow,
  so this adds no capability. It was a deliberate choice.
- **Security-signal jobs stay out of the public queue.** `dependabot-audit`
  and `github-config` get no reporting job. The morning review records their
  failures privately from run history. Recording them from Actions is
  vergil-project/.github#1605.
- **Blind spots stay out of scope.** A run that crashes before its reporting
  job, or never runs (a workflow GitHub disables, say), files nothing. Two
  examples came up during this epic: mq-rest-admin-project's `epic-sweep`,
  disabled since September, and `nightly-harnesses`, which had never run.
  The watcher that would catch these is vergil-project/.github#1603.
- **The docs sweep ran before the identity track finished.** The plan chose
  this, so that a stalled proof could not hold up the reporter docs. The
  cost is the identity-docs gap in §4.
- **Nothing closes an ops issue automatically.** Recovered issues stay open
  until someone working the queue closes them. vergil-project/vergil-containers#1042
  and #1049 are open, recovered, now.

## 4. New problems & opportunities

**Follow-ups filed during execution** (each ruled on the epic):

- **vergil-tooling:** #5021, #5022, #5042, #5175 and #5196.
- **vergil-claude-plugin:** #1288, #1302, #1303, #1304 and #1340.
- **vergil-actions:** #1381, #1382, #1418 and #1451. The `app-id`
  deprecation was triaged to the existing #1243.
- **vergil-containers:** #1010, and #1053 (superseded by #1054).
- **.github:** #1543.

All of these are vergil-project repos.

**Follow-on planning issues** from the brainstorm (vergil-project/.github#1497):

- vergil-project/.github#1603: an org-level watcher of run history.
- vergil-project/.github#1604: filling each org's missing jobs.
- vergil-project/.github#1605: automated security records.
- vergil-project/.github#1606: splitting `ops.yml` into one workflow per job.

**Unaddressed-findings sweep:**

- **What was swept:**
  - all 71 closed tasks, their 47 merged PRs (conversation, reviews and
    inline comments), the epic issue's comments and the planning issue's
    five design-review rounds;
  - 0 untrusted items skipped, 0 read failures.
- **Already closed:**
  - every `Awaiting a ruling` item on the epic and its tasks has a ruling;
  - all 49 design-review findings are settled: 46 accepted, 2 rejected and
    1 dropped, each with its reason;
  - the 7 security-class findings from design review were all handled
    through advisories (counts only).
- **Untracked problems found: 4.**
  - **U1. Bot identity and its exceptions are documented only in the epic
    spec.** The documentation sweep (vergil-project/vergil-tooling#4938, PR
    #5128) and vergil-project/docs#185 both deferred the identity track
    because it was still in flight. The five exceptions were settled after
    the sweep closed, and no human-facing page states the rule or lists
    them. vergil-actions#1415 and #1418 cover only the secret model.
    **Outcome: filed** on the human's ruling as intake
    vergil-project/.github#1683: one docs task, aimed at vergil-project/docs
    (the fleet CI page), for the rule and its exceptions.
  - **U2. The `package-cleanup` exception reason in vergil-containers
    overstates the limit.** The workflow comments and `test_ops_reporting.py`
    say "GitHub Packages does not accept App installation tokens". But the
    App reads packages (`package-cache-warm`, the Task 16 dry run). What
    fails is registry pushes, and deletes and untags. Spec §5.4 already says
    this correctly. **Outcome: filed** on the human's ruling as intake
    vergil-project/.github#1684, low severity, aimed at
    vergil-project/vergil-containers.
  - **U3. Package cleanup's deleting jobs can run from any branch** (since
    #1055). **Outcome: dropped** by the human: a deliberate, recorded
    trade-off that adds no capability (§3).
  - **U4. hoshin's check, as Task 10 shipped it, names two security-signal
    jobs and not the trusted-authors check.** **Outcome: dropped** by the
    human: the current skill (v2.1.96) has its own trusted-authors check.

## 5. What's next

- The four follow-on planning issues above (#1603–#1606), in the order the
  brainstorm proposed. The watcher (#1603) closes this epic's main blind
  spot.
- The six caller tasks that left the epic, now ad-hoc work in their repos:
  - mq-rest-admin-project/mq-rest-admin-common#236
  - mq-rest-admin-project/mq-rest-admin-dev-environment#180
  - mq-rest-admin-project/mq-rest-admin-python#536
  - mq-rest-admin-project/mq-rest-admin-go#332
  - mq-rest-admin-project/mq-rest-admin-rust#131
  - logical-minds-foundry/mq-resiliency-observability#84

  Until each lands, its repo's docs deploy stays stopped.
- The language lifecycle check's redesign (vergil-project/.github#1128)
  inherits this epic's two ops-issue rules: file under the operations epic,
  and never close automatically.
- The open, recovered ops issues (vergil-project/vergil-containers#1042,
  #1049) wait for the queue's next review to close them.
