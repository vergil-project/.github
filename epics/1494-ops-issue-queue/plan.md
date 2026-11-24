# ops-issue-queue — implementation plan

- **Epic:** vergil-project/.github#1494
- **Planning issue:** vergil-project/.github#1470 (review history is in its comments)
- **Status:** approved

> **For agentic workers:** each task is one issue and one PR in the repo named
> in its heading, worked with `issue-implement`. Operational tasks
> (deployment, validation) are worked with `issue-deploy` and
> `issue-validate`. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Every unattended job reports its own failures, as the org's
automation App, into one perpetual operations epic per org that the
morning review reads; then every unattended job runs as that App.

**Architecture:**

- **vergil-tooling** adds the `operations` standing kind (homed like
  hoshin, one per org in `<org>/.github`) and `vrg-ops-report`, which
  finds a job's open ops issue through epic #1467's trust layer and opens
  it, comments on it or records a recovery. It never closes one.
- **vergil-actions** adds the `ops-report` composite action, declares
  optional App secrets where they are missing, and, once every caller
  passes them, adds a reporting job to each shared workflow.
- **Caller repos** pass the App secrets to the docs and publish-index
  workflows. **vergil-containers** reports its own jobs and drops
  `pin-exposure`. **vergil-claude-plugin** points hoshin's daily review at
  the operations epics.
- **Identity second:** after the reporter is live, a GHCR proof on one
  job decides how the remaining `GITHUB_TOKEN` jobs move to the App.

**Tech Stack:** Python 3 and pytest (vergil-tooling); GitHub Actions YAML
with the yq-based shell tests of `actions/ops/*/tests/` (vergil-actions,
vergil-containers); Markdown (vergil-claude-plugin, .github).

**Spec:** [`spec.md`](spec.md) in this directory. Executors read both.

## Global Constraints

- **Nothing closes an ops issue automatically** (spec §2.3): no code path
  in this epic closes an issue labelled `ops-report`.
- **One open issue per report key** (spec §3, §5.2): org, repo and a
  stable job name, never the workflow file. Title exactly
  `ops: <repo> / <job-name> failing`; label `ops-report`.
- **Every read goes through `lib/trust.py`** (epic #1467, task
  vergil-project/vergil-tooling#4912): only issues and report comments
  whose author is the org's automation App (`Role.AUTOMATION`) count.
- **Results** are `success`, `failure` and `cancelled` only; a named
  job's GitHub job results combine as any `failure` → failure, else any
  `cancelled` → cancelled, else at least one `success` → success, all
  `skipped` → no report (spec §5.3).
- **Comments mark state:** every failure or cancel adds a comment; only
  the first green run after one adds `recovered` (spec §5.2).
- **Public text:** run link, job name, step name, conclusion, date. Never
  log excerpts or environment values (spec §2.3).
- **Security-signal jobs** (`dependabot-audit`, `github-config`) file no
  public ops issue and get no reporting job (spec §2.3).
- **Every reporting job starts with `actions/checkout@v6`**
  (`persist-credentials: false`), before the `ops-report` action: setting
  up Vergil reads the workspace's `vergil.toml`, and local `./actions/`
  refs resolve against the workspace.
- **Each reporting job's `uses:` step of the `ops-report` action sets
  `VRG_TRUSTED_AUTHORS` and `VRG_AUTOMATION_APP` from `vars` in its own
  `env`** (the vergil-project/vergil-actions#1359 convention).
  `trusted-authors-env.test.sh` checks it in vergil-actions (Task 7);
  `test_ops_reporting.py` checks it in vergil-containers (Task 8,
  `test_reporting_step_sets_trusted_env`).
- **No concurrency group on a reporting job.** GitHub keeps one pending
  run per group and cancels the older one, which would drop a failure
  report; concurrent reports are handled by the post-create listing wait
  and the "comment on the oldest" rule (Task 2b).
- **The App token is minted only on `refs/heads/develop`**; on any other
  ref the reporting step reports nothing (spec §5.3).
- **Release pushes report nothing** for `docs-deploy-develop` and
  `image-publish`: a push to `main`, or a develop push whose head commit
  is the merge of a `release/*` branch (spec §4.1).
- **Stable job names** (spec §4.1) of the reported jobs: `branch-sweep`,
  `epic-sweep`, `epic-rollup`, `upstream-refresh`, `nightly-rebuild-dev`,
  `nightly-rebuild-prod`, `package-cleanup`, `package-cache-warm`,
  `publish-index`, `docs-refresh`, `docs-deploy-develop`,
  `image-publish`.
- **No silent failures:** a reporter that cannot read or write fails its
  job red with the error (spec §2.3).
- Each tooling task updates the CLI reference
  (`docs/site/docs/reference/cli-tools-overview.md`).
- Every code task ends with a refactor step before its commit: tests
  green, remove duplication the task introduced, re-run.

## Review Focus

1. **An outsider opens an issue with an ops title and label** (the label
   needs triage rights, but a title alone does not). The reporter ignores
   it and files its own. Pinned in Task 2b
   (`test_find_open_report_ignores_non_automation_author`).
2. **An outsider comment reading `ops-report: recovered`** on a failing
   issue. The state stays failing, so the next green run still records the
   recovery. Pinned in Task 2b
   (`test_latest_state_ignores_untrusted_comment`).
3. **Two open App-authored issues with the same title** (a race). The
   reporter comments on the oldest, names the other, closes neither.
   Pinned in Task 2b (`test_report_two_open_comments_oldest`).
4. **Every GitHub job of a named job skipped** (the disabled lifecycle job,
   or a condition false by design). No issue, no comment, exit 0. Pinned
   in Task 2b (`test_combine_all_skipped_is_none`) and Task 7
   (`ops-report-skipped.test.sh`).
5. **A feature-branch `workflow_dispatch` of a reporting workflow.** No
   token is minted and nothing is filed. Pinned in Task 4
   (`ops-report-ref-guard.test.sh`).

## Delivery table

Each line gives the task, its repo and kind, what it delivers, and what it
is blocked by. Tasks in another org are filed under that org's proxy epic
and named by full ref once filed.

### Reporter: foundation

- **1** (.github, task #1495): spec and plan, this PR.
- **2a** (vergil-tooling): the `operations` standing kind, its labels and
  the `--epic operations` sentinel.
- **2b** (vergil-tooling): `vrg-ops-report`. Blocked by 2a and
  vergil-project/vergil-tooling#4912.
- **2c** (vergil-tooling): ops issues are not PR-workable or routable,
  and the sweep
  skips them. Blocked by 2a.
- **3** (.github, deployment): the vergil-tooling release carrying 2a, 2b, 2c
  and vergil-project/vergil-tooling#4912, installed. Blocked by 2a, 2b, 2c.
- **4** (vergil-actions): the `ops-report` action, and optional App
  secrets declared on `cd-docs.yml`, `cd-docs-refresh.yml` and
  `publish-index.yml`. Blocked by 2b and vergil-project/vergil-actions#1351.
- **5** (.github, deployment): the vergil-actions release carrying 4,
  with `v2.1` moved. Blocked by 3, 4.

### Reporter: caller pass (one task per caller repo)

Each passes `APP_CLIENT_ID` and `APP_PRIVATE_KEY` to every call of
`cd-docs.yml`, `cd-docs-refresh.yml` and `publish-index.yml` in its repo.
All are blocked by 5.

- **6a–6k** (vergil-project): `docs`, `vergil-claude-plugin`,
  `vergil-containers`, `vergil-tooling`, `vergil-vm`,
  `vergil-fixture-python-lib`, `vergil-fixture-python-app`,
  `vergil-fixture-rust-lib`, `vergil-fixture-rust-app`,
  `vergil-fixture-ts-lib`, `vergil-fixture-ts-app`. (`packages` needs no
  task: its one call uses `secrets: inherit`. vergil-actions' own local
  call changes in Task 7.)
- **6l–6n** (logical-minds-foundry, under its proxy): `docs`,
  `mq-resiliency-observability`, `mq-resiliency-lab-for-linux`.
- **6o** (mnemosys-project, under its proxy): `docs`.
- **6p–6t** (mq-rest-admin-project, under its proxy):
  `mq-rest-admin-common`, `mq-rest-admin-dev-environment`,
  `mq-rest-admin-python`, `mq-rest-admin-go`, `mq-rest-admin-rust`.
  (`mq-rest-admin-java` is archived on GitHub, never runs, and has no
  task.)

### Reporter: reporting and rollout

- **7** (vergil-actions): reporting jobs in `ops-github-config.yml`,
  `ops-epic-sweep.yml`, `ops-epic-rollup.yml`, `ops-upstream-refresh.yml`,
  `cd-docs.yml`, `cd-docs-refresh.yml`, `publish-index.yml`, and its own
  `cd.yml` local call. Blocked by 3, 4, 11
  and vergil-project/vergil-actions#1351.
- **8** (vergil-containers): reporting jobs for `nightly-rebuild-dev`,
  `nightly-rebuild-prod`, `image-publish`, `package-cleanup`,
  `package-cache-warm`. Blocked by 5, 11.
- **9** (vergil-containers): deprecate `pin-exposure`. No blocker.
- **10** (vergil-claude-plugin): hoshin's daily review reads the
  operations epics, through `vrg-issue-read`. Blocked by 2a, 11 and
  vergil-project/vergil-tooling#4929.
- **11** (.github, deployment): the one-off: App and trusted-author checks
  in each org, then each org's operations epic created. Blocked by 3.
- **12** (.github, deployment): the vergil-actions release carrying 7,
  with `v2.1` moved. Blocked by 7, 6a–6t, 11.
- **10b** (.github, deployment): the plugin release carrying 10, and
  `vrg-issue-read`, installed. Blocked by 10.
- **13** (.github, validation): the reporter is live. Blocked by 8, 10b, 12.
- **14** fixture runs: `fixture-run vergil-project/.github#1494
  python-lib,python-app,rust-lib,rust-app,ts-lib,ts-app --validates
  "vergil-actions reporting release (Task 12)" --after <Task 12 ref>`.
  Filed once 12 exists. Blocked by 12.

### Identity (after the reporter)

- **15** (vergil-containers): `nightly-rebuild-dev` logs in to GHCR with
  the App token. Blocked by 13.
- **16** (.github, validation): the GHCR proof: the next nightly
  `nightly-rebuild-dev` pushes with the App token. Blocked by 15.
- **16b** (.github, deployment): a vergil-containers release puts Task
  15's `cd-docker-publish.yml` on `main`. Blocked by 15.
- **17** (vergil-containers): `nightly-rebuild-prod`, `image-publish`,
  `package-cleanup` and `package-cache-warm` on the App token, or a
  recorded exception each. Blocked by 16, 16b.
- **18** (vergil-actions): `publish-index`, `cd-docs`, `cd-docs-refresh`
  on the App token, and the `VRG_AUTOMATION_APP` check added to
  `branch-sweep`, `dependabot-audit`, `epic-sweep` and `epic-rollup`.
  Blocked by 13.
- **19** (.github, deployment): the vergil-actions release carrying 18,
  with `v2.1` moved. Blocked by 18.
- **19b** (.github): record every identity exception from 17 and 18 in
  `spec.md` §4.1 and §5.4. Blocked by 17, 18.
- **20** (.github, validation): every in-scope job runs as the App or has
  a recorded exception. Blocked by 17, 19, 19b.

### Bookends

- **Documentation review** (vergil-tooling, bookend
  vergil-project/vergil-tooling#4938). Blocked by 13, not by 20, so a
  stalled GHCR proof never holds the reporter docs; the identity docs are
  covered when the sweep runs. Its sweep covers the docs repo's
  `docs/site/docs/management/epics-and-tasks.md`
  (the epic kinds) and vergil-actions' workflow READMEs.
- **Follow-on brainstorm** (.github, bookend #1497). Blocked by 13.
- **Retrospective** (.github, bookend #1498). Blocked by everything else.

Notes:

- **Prerequisites from epic #1467:** the trust layer
  (vergil-project/vergil-tooling#4912) before 2b, and the delivery of
  `VRG_TRUSTED_AUTHORS` to jobs (vergil-project/vergil-actions#1351)
  before 7.
- **Order of the two vergil-actions releases** (spec §5.6): 5 declares
  the secrets as optional, the caller pass (6a–6t) passes them, and only
  then does 12 release the reporting jobs. Passing a secret a reusable
  workflow does not declare is a workflow error, and a reporting job
  without the secrets fails red, so neither order can be swapped.
- **`dependabot-audit` and `github-config` get no reporting job** (spec
  §2.3). Each failure stays a red run that hoshin's daily review reads
  from run history and records through `vrg-security-record` as "a
  security-related failure", as it does today; Task 10 keeps that path
  for these two jobs.
- **The language lifecycle job** is disabled by epic #1467 and gets no
  reporting job here; #1128's redesign adds one (spec §5.5). G4 binds
  that redesign: at epic-create's filing step, a comment on
  vergil-project/.github#1128 states the two rules (every issue it files
  goes under the operations epic; it closes none of them automatically)
  and links spec §5.5, so whoever plans #1128 inherits them.
- **If the GHCR proof (16) fails,** Task 17 records each GHCR job as an
  exception (Task 19b writes it into `spec.md` §4.1) instead of
  switching it; the reporter is unaffected.
- **Filing.** Every task above is filed at epic-create's filing step, in
  its repo, under #1494 or its org's proxy, with its `Blocked-by:` refs.

### Filed tasks

Each task as filed, linked under the epic or, for another org's repo,
under that org's proxy (logical-minds-foundry/.github#420,
mnemosys-project/.github#120, mq-rest-admin-project/.github#53).

- **2a:** vergil-project/vergil-tooling#4962
- **2b:** vergil-project/vergil-tooling#4963
- **2c:** vergil-project/vergil-tooling#4964
- **3:** vergil-project/.github#1510
- **4:** vergil-project/vergil-actions#1361
- **5:** vergil-project/.github#1511
- **6a:** vergil-project/docs#160
- **6b:** vergil-project/vergil-claude-plugin#1255
- **6c:** vergil-project/vergil-containers#986
- **6d:** vergil-project/vergil-tooling#4965
- **6e:** vergil-project/vergil-vm#444
- **6f:** vergil-project/vergil-fixture-python-lib#62
- **6g:** vergil-project/vergil-fixture-python-app#55
- **6h:** vergil-project/vergil-fixture-rust-lib#76
- **6i:** vergil-project/vergil-fixture-rust-app#70
- **6j:** vergil-project/vergil-fixture-ts-lib#71
- **6k:** vergil-project/vergil-fixture-ts-app#72
- **6l:** logical-minds-foundry/docs#38
- **6m:** logical-minds-foundry/mq-resiliency-observability#84
- **6n:** logical-minds-foundry/mq-resiliency-lab-for-linux#1754
- **6o:** mnemosys-project/docs#33
- **6p:** mq-rest-admin-project/mq-rest-admin-common#236
- **6q:** mq-rest-admin-project/mq-rest-admin-dev-environment#180
- **6r:** mq-rest-admin-project/mq-rest-admin-python#536
- **6s:** mq-rest-admin-project/mq-rest-admin-go#332
- **6t:** mq-rest-admin-project/mq-rest-admin-rust#131
- **7:** vergil-project/vergil-actions#1362
- **8:** vergil-project/vergil-containers#987
- **9:** vergil-project/vergil-containers#988
- **10:** vergil-project/vergil-claude-plugin#1256
- **10b:** vergil-project/.github#1513
- **11:** vergil-project/.github#1512
- **12:** vergil-project/.github#1514
- **13:** vergil-project/.github#1515
- **15:** vergil-project/vergil-containers#989
- **16:** vergil-project/.github#1516
- **16b:** vergil-project/.github#1517
- **17:** vergil-project/vergil-containers#990
- **18:** vergil-project/vergil-actions#1363
- **19:** vergil-project/.github#1518
- **19b:** vergil-project/.github#1519
- **20:** vergil-project/.github#1520
- **14** (fixture runs, change → release → validation):
  - vergil-fixture-python-lib: #63 → #64 → #65
  - vergil-fixture-python-app: #56 → #57 → #58
  - vergil-fixture-rust-lib: #77 → #78 → #79
  - vergil-fixture-rust-app: #71 → #72 → #73
  - vergil-fixture-ts-lib: #72 → #73 → #74
  - vergil-fixture-ts-app: #73 → #74 → #75
- **Bookends:** documentation review vergil-project/vergil-tooling#4938,
  follow-on brainstorm vergil-project/.github#1497 (both blocked by 13),
  retrospective vergil-project/.github#1498.

---

### Task 2a: the operations standing kind (vergil-tooling)

Spec §5.1.

**Files:**

- Modify: `src/vergil_tooling/lib/epics.py` (beside `HOSHIN`,
  `hoshin_home`, `ensure_hoshin_epic`; `resolve_epic_ref`'s sentinel map)
- Modify: `src/vergil_tooling/data/labels.json`
- Modify: `src/vergil_tooling/bin/vrg_issue_create.py`,
  `src/vergil_tooling/bin/vrg_epic_move.py` (help text for `--epic`)
- Modify: `docs/site/docs/reference/cli-tools-overview.md`
- Test: `tests/vergil_tooling/test_epics.py`,
  `tests/vergil_tooling/test_data_labels.py`,
  `tests/vergil_tooling/test_vrg_issue_create.py`

**Interfaces:**

- Produces: `OPERATIONS: StandingKind` with `key="operations"`,
  `sentinel="operations"`, `title_prefix="Epic (operations): "`,
  `labels=("epic", "operations")`,
  `archive_prefix="Archive (operations): "`,
  `archive_labels=("archive", "operations")`; it is in `STANDING_KINDS`
  (after `HOSHIN`), not in `QUEUE_KINDS`.
- Produces: `operations_home(target_repo: str) -> tuple[str, str]`,
  returning `(<owner>/.github, <owner>)` through `require_org_home`, as
  `hoshin_home` does;
  `ensure_operations_epic(target_repo: str) -> IssueRef`.
- Produces: an ensure-only command for a standing epic that creates or
  reuses it and prints its ref, creating no task (for example
  `vrg-epic-create --standing operations --org <org>`; the exact form is
  the implementer's, named in the CLI reference). Task 11 uses it.
- Produces: registry labels `operations` ("Perpetual epic of an org's
  operational issues; never auto-closes") and `ops-report` ("Ops issue
  filed by vrg-ops-report for an unattended job; never auto-closes");
  `epic-management`'s description becomes "Management epic: the intake,
  planning, deferred, hoshin or operations epic; never auto-closes".

- [ ] **Step 1: Write the failing tests** in `test_epics.py`:
  `test_operations_kind_is_standing_not_queue` (`OPERATIONS in
  STANDING_KINDS`, `OPERATIONS not in QUEUE_KINDS`,
  `standing_kind_of({"epic", "operations"}) is OPERATIONS`);
  `test_operations_home_is_org_github` (`operations_home("acme/widget")
  == ("acme/.github", "acme")`);
  `test_resolve_operations_sentinel_calls_ensure` (the `operations`
  sentinel resolves through `ensure_operations_epic`);
  `test_operations_epic_gets_epic_management` (`category_labels` of the
  kind's labels includes `epic-management`);
  `test_drain_standing_org_includes_operations` (`OPERATIONS in
  DRAINED_STANDING_KINDS`; draining and `epic-management` both derive
  from `STANDING_KINDS`, so no drain code changes). In
  `test_data_labels.py`:
  `test_operations_labels_registered` with the exact descriptions above.
  In `test_vrg_issue_create.py` and `test_vrg_epic_move.py`:
  `test_epic_operations_sentinel_accepted`.
  For the ensure-only command: `test_ensure_standing_operations_creates_no_task`
  and `test_ensure_standing_operations_reuses_existing`.
- [ ] **Step 2: Run them.** `vrg-container-run -- pytest
  tests/vergil_tooling/test_epics.py tests/vergil_tooling/test_data_labels.py
  tests/vergil_tooling/test_vrg_issue_create.py -q`. Expect FAIL.
- [ ] **Step 3: Implement** `OPERATIONS`, `operations_home`,
  `ensure_operations_epic`, the sentinel entry and the ensure-only
  command. Add the two
  labels and the new `epic-management` description to `labels.json`;
  list `operations` in the `--epic` help of both CLIs and in the CLI
  reference.
- [ ] **Step 4: Run the tests.** Expect PASS. Then the refactor step.
- [ ] **Step 5: Commit** `vrg-commit --type feat --scope epics --message
  "add the operations standing epic kind"`, then `vrg-container-run --
  vrg-validate` and report-ready.

### Task 2c: ops issues are not PR-workable (vergil-tooling)

Spec §5.1. Blocked by 2a.

**Files:**

- Modify: `src/vergil_tooling/lib/epics.py` (`queue_item_kind`'s
  neighbour; `_QUEUE_REFUSAL_TAILS`)
- Modify: `src/vergil_tooling/bin/vrg_pr_workflow.py`,
  `src/vergil_tooling/bin/vrg_submit_pr.py`,
  `src/vergil_tooling/lib/pr_workflow/batch_rules.py` (call the new
  function instead of `queue_item_kind`)
- Modify: `src/vergil_tooling/lib/epic_audit.py` (`_open_task_drift`)
- Modify: `src/vergil_tooling/lib/intake_route.py` (beside the hoshin
  item refusal, `HOSHIN_ITEM_REFUSAL`)
- Test: `tests/vergil_tooling/test_epics.py`,
  `tests/vergil_tooling/test_vrg_pr_workflow.py`,
  `tests/vergil_tooling/test_vrg_submit_pr.py`,
  `tests/vergil_tooling/test_epic_audit.py`,
  `tests/vergil_tooling/test_intake_route.py`

**Intake routing.** An item already under an operations epic is never
routed anywhere (an ops issue stays until it is closed by hand). An
intake item may be routed *into* the operations epic as a one-off
(`--epic operations`), so a hand-captured operational problem reaches
the queue through intake review.

**Interfaces:**

- Consumes: Task 2a's `OPERATIONS`.
- Produces: `unbuildable_parent_kind(ref: str, *, default_repo: str) ->
  str | None`: the queue kind of the native parent, as
  `queue_item_kind` returns it, or `"operations"` when the parent is an
  operations epic, else `None`. A failed lookup raises, as
  `queue_item_kind`'s does.
- Produces: `_QUEUE_REFUSAL_TAILS["operations"]` = "it is an ops issue,
  which no PR closes: file the fix as a follow-up task in its own repo,
  and close the ops issue by hand once the problem is dealt with."

- [ ] **Step 1: Write the failing tests:**
  `test_unbuildable_parent_kind_operations` and
  `test_unbuildable_parent_kind_queue_unchanged` (`test_epics.py`);
  `test_report_ready_refuses_ops_issue` (`test_vrg_pr_workflow.py`);
  `test_submit_refuses_ops_issue` (`test_vrg_submit_pr.py`);
  `test_task_drift_skips_ops_issue` (a merged PR whose linkage names an
  issue under an operations epic yields no drift; `test_epic_audit.py`);
  `test_route_refuses_item_under_operations` and
  `test_route_one_off_into_operations_allowed` (`test_intake_route.py`).
  Run them. Expect FAIL.
- [ ] **Step 2: Implement** `unbuildable_parent_kind`, switch the three
  callers to it, add the refusal tail, have `_open_task_drift` return
  `None` for an issue whose native parent is an operations epic, and add
  `OPERATIONS_ITEM_REFUSAL` to `intake_route.py` ("{item} is an ops issue
  under {epic}; it is closed by hand once dealt with, never routed").
- [ ] **Step 3: Run the tests.** Expect PASS. Then the refactor step.
- [ ] **Step 4: Commit** `vrg-commit --type feat --scope epics --message
  "refuse PRs for ops issues and keep the sweep off them"`, then
  `vrg-container-run -- vrg-validate` and report-ready.

### Task 2b: `vrg-ops-report` (vergil-tooling)

Spec §5.2, §5.3 (combined results).

**Files:**

- Create: `src/vergil_tooling/lib/ops_report.py`
- Create: `src/vergil_tooling/bin/vrg_ops_report.py`
- Modify: `pyproject.toml` (`vrg-ops-report =
  "vergil_tooling.bin.vrg_ops_report:main"`)
- Modify: `docs/site/docs/reference/cli-tools-overview.md`
- Test: `tests/vergil_tooling/test_ops_report.py`,
  `tests/vergil_tooling/test_vrg_ops_report.py`

**Interfaces:**

- Consumes: Task 2a's `ensure_operations_epic`; #4912's
  `trust.read_issue`, `trust.Role`, `trust.IssueContent`,
  `trust.UntrustedContentError`, `trust.TrustReadError`;
  `labels.ensure_labels`; `identity_mode` (refuse unless the mode is
  `IdentityMode.AUTOMATION`).
- Produces: `Result = Literal["success", "failure", "cancelled"]`;
  `combine_results(results: Sequence[str]) -> Result | None`;
  `report_title(repo: str, job: str) -> str` (`repo` is the bare repo
  name); `MARKER_FAILURE = "ops-report: failure"`,
  `MARKER_CANCELLED = "ops-report: cancelled"`,
  `MARKER_RECOVERED = "ops-report: recovered"` (the first line of every
  report the reporter writes, body or comment);
  `find_open_reports(repo: str, job: str) -> list[int]` (App-authored
  open issues with the exact title and label, oldest first);
  `latest_state(content: IssueContent) -> Literal["failing",
  "recovered"]` (from the body and `Role.AUTOMATION` comments only);
  `previous_occurrence(repo: str, job: str) -> int | None` (the most
  recent closed `ops-report` issue with the title whose `author_role` is
  `Role.AUTOMATION`, read through the trust layer);
  `Outcome = Literal["created", "commented", "recovered", "nothing"]`;
  `report(repo: str, job: str, result: Result | None, *, run_url: str,
  step: str | None, today: date) -> Outcome`.
- CLI: `vrg-ops-report --repo <owner>/<repo> --job <name> --results
  <r>[,<r>…] --run-url <url> [--step <name>]`. Each `<r>` is a GitHub job
  result (`success`, `failure`, `cancelled`, `skipped`); exit 0 on every
  outcome, 1 on any read, write or identity error.

- [ ] **Step 1: Write the failing tests** in `test_ops_report.py`, with
  `gh` and the trust layer faked:
  - `test_combine_any_failure_wins` (`["success", "failure",
    "cancelled"]` → `"failure"`); `test_combine_cancelled_over_success`;
    `test_combine_success_with_skipped` (`["success", "skipped"]` →
    `"success"`); `test_combine_all_skipped_is_none`;
    `test_combine_unknown_value_raises` (`ValueError`).
  - `test_report_title` (`report_title("docs", "docs-refresh") ==
    "ops: docs / docs-refresh failing"`).
  - `test_find_open_report_ignores_non_automation_author`;
    `test_find_open_report_ignores_closed`.
  - `test_latest_state_ignores_untrusted_comment` (an outsider comment
    starting `ops-report: recovered` after an App failure comment →
    `"failing"`).
  - `test_report_failure_no_open_creates` (issue created via
    `vrg-issue-create --epic operations`, label `ops-report`, body first
    line `MARKER_FAILURE`, naming the run URL, step and date →
    `"created"`); `test_report_failure_names_previous_occurrence`
    (`previous occurrence: #41` in the body);
    `test_previous_occurrence_ignores_non_automation_author` (a
    later-closed same-titled issue by an outsider is skipped).
  - `test_report_failure_open_comments` → `"commented"`.
  - `test_report_success_after_failure_records_recovery` →
    `"recovered"`; `test_report_success_after_recovery_does_nothing`;
    `test_report_success_no_open_does_nothing`;
    `test_report_none_does_nothing` (no read, no write).
  - `test_report_two_open_comments_oldest` (comments on the lower number,
    naming the other; no close call).
  - `test_report_relinks_orphan` (an open ops issue with no native
    parent, or a parent other than the org's operations epic, is linked
    under the operations epic on the next report of any result; a
    failed link exits 1).
  - `test_report_waits_until_created_issue_listed` (after a create,
    `report()` polls the title lookup until it lists the new issue, as
    `epics._await_listed` does, and raises if it never does).
  - `test_report_never_closes` (no `gh issue close` across every case).
  - `test_report_body_has_no_log_text` (the body holds only the marker,
    run URL, job, step, date and previous occurrence).
  In `test_vrg_ops_report.py`: `test_refuses_non_automation_mode`
  (exit 1, nothing written); `test_trust_read_error_exits_1`;
  `test_results_combined_from_cli`.
- [ ] **Step 2: Run them.** `vrg-container-run -- pytest
  tests/vergil_tooling/test_ops_report.py
  tests/vergil_tooling/test_vrg_ops_report.py -q`. Expect FAIL.
- [ ] **Step 3: Implement** `lib/ops_report.py` and the CLI. Ensure the
  `ops-report` label in the repo with `ensure_labels` before creating.
  Read issues only through `trust.read_issue(..., roles={Role.AUTOMATION})`;
  list candidates by title and label, then keep those whose
  `author_role` is `Role.AUTOMATION`. After a create, wait until the
  lookup lists the new issue (reuse `epics._await_listed`'s pattern and
  budget), so a run right behind it finds it rather than creating a
  second. Whenever `report()` finds or creates an issue, it checks the
  native parent and links it under `ensure_operations_epic` when it is
  missing or different (`vrg-issue-create` creates, then links, so a
  failed link leaves an orphan that the sweep would otherwise attach to
  intake).
- [ ] **Step 4: Run the tests.** Expect PASS. Then the refactor step.
- [ ] **Step 5: Commit** `vrg-commit --type feat --scope ops --message
  "add vrg-ops-report for unattended job failures"`, then
  `vrg-container-run -- vrg-validate` and report-ready.

### Task 3: vergil-tooling release for the reporter (deployment, .github)

Spec §5.6. Blocked by 2a, 2b, 2c.

- **Precondition (human-attested):** a vergil-tooling release containing
  Tasks 2a, 2b, 2c and vergil-project/vergil-tooling#4912 is published and
  installed on the host and every VM, its version and tag named.
- **Procedure:**
  1. For the merge commit of each of those four PRs, `vrg-git
     merge-base --is-ancestor <merge-sha> <tag-sha>` in vergil-tooling
     exits 0.
  2. The `v2.1` line resolves to this release or later, so Actions jobs
     using `setup-vergil` install it.
  3. `vrg-ops-report --help` and `vrg-issue-create --help` (lists
     `operations`) run on this host.
- **SUCCESS:** every check passes, recorded with the version and tag.

### Task 4: the `ops-report` action and optional secrets (vergil-actions)

Spec §5.3, §5.6. Blocked by 2b and vergil-project/vergil-actions#1351 (its
delivery of `VRG_TRUSTED_AUTHORS` is the one this action follows).

**Files:**

- Create: `actions/ops/report/action.yml`
- Create: `actions/ops/report/tests/ops-report-ref-guard.test.sh`,
  `actions/ops/report/tests/ops-report-inputs.test.sh`
- Modify: `.github/workflows/cd-docs.yml`,
  `.github/workflows/cd-docs-refresh.yml`,
  `.github/workflows/publish-index.yml` (`on.workflow_call.secrets`
  gains `APP_CLIENT_ID` and `APP_PRIVATE_KEY`, each `required: false`;
  no job uses them yet). `cd-docs.yml` also declares a boolean input
  `report` (default `true`, unused until Task 7), and
  `cd-docs-refresh.yml`'s nested `cd-docs.yml@develop` call passes
  `report: false` and forwards both secrets. The nested call resolves to
  develop's `cd-docs.yml`, so this must be in the released refresh before
  Task 7 adds the reporting job there.
- Modify: `actions/shared/setup/vergil/tests/trusted-authors-env.test.sh`
  (`vrg-ops-report` joins its list of GitHub-touching commands, with a
  fixture of a reporting step that omits `VRG_TRUSTED_AUTHORS`, which the
  test must catch)
- Test: `actions/ops/report/tests/optional-secrets.test.sh`

**Interfaces:**

- Produces: the action `vergil-project/vergil-actions/actions/ops/report`
  with inputs `job` (stable name, required), `results` (comma list of
  `needs.*.result` values, required), `step` (optional), `app-client-id`
  and `app-private-key` (required). A composite action cannot read
  `vars`, so, by the convention vergil-project/vergil-actions#1359 set,
  each reporting job's `uses:` step sets `VRG_TRUSTED_AUTHORS:
  ${{ vars.VRG_TRUSTED_AUTHORS }}` and `VRG_AUTOMATION_APP:
  ${{ vars.VRG_AUTOMATION_APP }}` in its own `env`, which reaches the
  action's steps. The action sets up Vergil, mints an issues-only App
  token for the repo and its org `.github`, and runs `vrg-ops-report --repo
  ${{ github.repository }} --job … --results … --run-url <run URL>`.
- Produces: a first step that exits 0 without minting when
  `github.ref != 'refs/heads/develop'`.

- [ ] **Step 1: Write the failing shell tests** (the yq pattern of
  `actions/ops/github-config/tests/automation-env.test.sh`):
  `ops-report-ref-guard.test.sh` asserts the ref check precedes the token
  step and every later step is conditioned on it;
  `ops-report-inputs.test.sh` asserts the token requests only
  `permission-issues: write` and the action's metadata contains no
  `vars.` or `secrets.` expression; `trusted-authors-env.test.sh` lists
  `vrg-ops-report` and catches its new fixture;
  `optional-secrets.test.sh` asserts the three workflows declare both
  secrets with `required: false`, `cd-docs.yml` declares `report`, and
  the refresh's nested call passes `report: false` and both secrets.
  Run them. Expect FAIL.
- [ ] **Step 2: Implement** the action and the three declarations.
- [ ] **Step 3: Run** the tests and `vrg-container-run -- vrg-validate`
  (actionlint). Expect PASS. Then the refactor step.
- [ ] **Step 4: Commit** `vrg-commit --type feat --scope ops --message
  "add the ops-report action and optional App secrets"`, then
  report-ready.

### Task 5: vergil-actions release with the action (deployment, .github)

Blocked by 3, 4.

- **Precondition (human-attested):** a vergil-actions release containing
  Task 4 is tagged and `v2.1` moved to it.
- **Procedure:** `vrg-git merge-base --is-ancestor <Task 4 merge-sha>
  <v2.1-tag-sha>` in vergil-actions exits 0; one caller's next CD run
  (any repo) still succeeds with no secrets passed.
- **SUCCESS:** both checks pass.

### Tasks 6a–6t: pass the App secrets (each caller repo)

Spec §5.6. Blocked by 5. One task per repo in the delivery table, each
titled `Pass the App secrets to the shared docs and publish workflows`.

- [ ] **Step 0:** Write the failing check: a yq assertion, run with
  `vrg-container-run`, that every call of `cd-docs.yml`,
  `cd-docs-refresh.yml` or `publish-index.yml` in `.github/workflows/`
  passes both `APP_CLIENT_ID` and `APP_PRIVATE_KEY` or uses `secrets:
  inherit`. Run it. Expect FAIL. (A one-off check in the PR notes; no
  test file is added to the repo.)
- [ ] **Step 1:** In every job of `.github/workflows/*.yml` that calls
  `cd-docs.yml`, `cd-docs-refresh.yml` or `publish-index.yml`, add
  `secrets: { APP_CLIENT_ID: ${{ secrets.APP_CLIENT_ID }},
  APP_PRIVATE_KEY: ${{ secrets.APP_PRIVATE_KEY }} }` (block form). Keep
  any secrets already passed. A call that uses `secrets: inherit` is left
  unchanged: it already passes both, and an explicit map cannot be
  combined with `inherit` (`packages`' `publish-index.yml` needs
  `inherit` for its signing environment, vergil-project/packages#6, so
  `packages` has no task here). Workflows that never run on `develop`
  (vergil-actions' `smoke-cd-docs.yml` and `smoke-publish-index.yml`)
  need no secrets: the reporter mints nothing off `develop`.
- [ ] **Step 2:** `vrg-container-run -- vrg-validate`. Expect PASS.
- [ ] **Step 3:** Commit `vrg-commit --type ci --scope cd --message "pass
  the App secrets to the shared docs workflows"`, then report-ready. The
  push is refused for a `.github/workflows/**` change; the human's
  `vrg-submit-pr` pushes it.

### Task 7: reporting jobs in the shared workflows (vergil-actions)

Spec §5.3, §4.1. Blocked by 3, 4, 11 and vergil-project/vergil-actions#1351:
vergil-actions' own local `cd-docs.yml` call runs develop's copy, so its
reporting job goes live when this task merges, and needs the released
`vrg-ops-report` and the org checks first.

**Files:**

- Modify: `.github/workflows/ops-github-config.yml` (`report-branch-sweep`;
  none for `github-config` or `dependabot-audit`),
  `ops-epic-sweep.yml` (`report`), `ops-epic-rollup.yml` (`report`),
  `ops-upstream-refresh.yml` (`report`, `needs: [prepare, publish]`),
  `cd-docs.yml` (`report`, with the release-push skip, gated on the
  `report` input Task 4 declared),
  `cd-docs-refresh.yml` (`report`; its nested call already passes (Task 4)
  `report: false` and forwards `APP_CLIENT_ID` and `APP_PRIVATE_KEY`),
  `publish-index.yml` (`report`, `needs` every job), and vergil-actions'
  own `.github/workflows/cd.yml` (its local `./.github/workflows/cd-docs.yml`
  call passes `APP_CLIENT_ID` and `APP_PRIVATE_KEY`: a local call runs
  develop's copy at once, so it must change in the same PR)
- Create: `actions/ops/report/tests/reporting-jobs.test.sh`,
  `actions/ops/report/tests/ops-report-skipped.test.sh`,
  `actions/ops/report/tests/release-push-skip.test.sh`,
  `actions/ops/report/tests/docs-refresh-reports-once.test.sh`
- Modify: `actions/ops/upstream-refresh/tests/key-boundary.test.sh` (the
  boundary's new written rule, below)

**Interfaces:**

- Consumes: Task 4's action; the stable job names in Global Constraints.

**The upstream-refresh key boundary.** The guard keeps the App key off
the runner that runs the repo's untrusted refresh hook. It gains one rule
and loses none: a third job, `report`, is allowed; it never runs the
hook, downloads no artifact, and is the only place other than
`publish`'s step where `APP_CLIENT_ID` and `APP_PRIVATE_KEY` appear (as
inputs to the `ops-report` action). Its `if: always()` is the one `if:`
the guard permits. A fixture that adds a secret to `prepare` still fails.

- [ ] **Step 1: Write the failing shell tests:** `reporting-jobs.test.sh`
  asserts each workflow above has one reporting job per stable name, with
  `if: always()`, `needs` naming every GitHub job of that name, `results`
  built from those `needs.*.result` values, and `job` set to the stable
  name, its first step `actions/checkout@v6` with `persist-credentials:
  false`; and that `ops-github-config.yml` has no reporting job for
  `github-config` or `dependabot-audit`. `ops-report-skipped.test.sh`
  asserts the `results` passed for upstream-refresh include both
  `prepare` and `publish`.
  `release-push-skip.test.sh` asserts the `cd-docs.yml` reporting job is
  conditioned off for `refs/heads/main` and for a head commit that merges
  a `release/*` branch. `docs-refresh-reports-once.test.sh` asserts the
  `cd-docs.yml` reporting job is conditioned on `inputs.report`, and the
  refresh's nested call passes `report: false` and both secrets.
  `key-boundary.test.sh` gains the `report` job rule above and a fixture
  with a secret on `prepare` that must fail. Run them. Expect FAIL.
- [ ] **Step 2: Implement** the reporting jobs, each calling
  `./actions/ops/report` with the job's App secrets.
- [ ] **Step 3: Run** the tests and `vrg-container-run -- vrg-validate`.
  Expect PASS. Then the refactor step.
- [ ] **Step 4: Commit** `vrg-commit --type feat --scope ops --message
  "report every shared unattended job to the operations queue"`, then
  report-ready.

### Task 8: reporting jobs in vergil-containers (vergil-containers)

Spec §5.3, §4.1. Blocked by 5, 11 (the org checks pass before any
reporter here goes live).

**Files:**

- Modify: `.github/workflows/ops.yml` (`report-nightly-rebuild-dev`,
  `report-nightly-rebuild-prod`), `.github/workflows/cd.yml`
  (`report-image-publish`, with the release-push skip),
  `.github/workflows/package-cleanup.yml` (`report`, `needs: [cleanup,
  reap-aliases]`), `.github/workflows/package-cache-warm.yml` (`report`)
- Test: `docker/versions/test_ops_reporting.py`

- [ ] **Step 1: Write the failing test** `test_ops_reporting.py`
  (pytest over the parsed YAML, as `test_ops_pin_exposure.py` does):
  `test_each_job_has_a_reporting_job` (the five stable names, `if:
  always()`, `needs` every GitHub job of the name);
  `test_image_publish_report_skips_release_pushes`;
  `test_reporting_uses_ops_report_action_at_v2_1`;
  `test_reporting_job_checks_out_first` (`actions/checkout@v6`, with
  `persist-credentials: false`, before the action). Run it. Expect FAIL.
- [ ] **Step 2: Implement** the reporting jobs, calling
  `vergil-project/vergil-actions/actions/ops/report@v2.1` with the App
  secrets the repo already holds.
- [ ] **Step 3: Run** the test and `vrg-container-run -- vrg-validate`.
  Expect PASS. Then the refactor step.
- [ ] **Step 4: Commit** `vrg-commit --type feat --scope ops --message
  "report the container jobs to the operations queue"`, then
  report-ready.

### Task 9: deprecate pin-exposure (vergil-containers)

Spec §5.5.

**Files:**

- Modify: `.github/workflows/ops.yml` (remove the `pin-exposure` job and
  its comment block; add a comment naming `upstream-refresh` as the path
  that fixes pin lag)
- Delete: `docker/versions/test_ops_pin_exposure.py`
- Keep: `docker/pins/report_exposure.py` (still runnable by hand)

- [ ] **Step 1:** Change `test_ops_reporting.py` (or, if Task 8 has not
  landed, a new `test_ops_jobs.py`) to assert `pin-exposure` is not a
  job of `ops.yml`. Run it. Expect FAIL.
- [ ] **Step 2:** Remove the job and the old test; add the comment.
- [ ] **Step 3:** Run the test and `vrg-container-run -- vrg-validate`.
  Expect PASS. Then the refactor step.
- [ ] **Step 4:** Commit `vrg-commit --type ci --scope ops --message
  "deprecate the report-only pin-exposure job"`, then report-ready.

### Task 10: hoshin's daily review reads the queue (vergil-claude-plugin)

Spec §2.1 G5. Blocked by 2a, 11 (the operations epics exist) and
vergil-project/vergil-tooling#4929.

**Files:**

- Modify: `skills/hoshin/references/daily.md` ("Check: Failed and
  cancelled runs")

- [ ] **Step 1:** Rewrite the check, which both the start (chōrei) and
  end (shūrei) checklists run, each over its own window, so its source is
  each org's
  `Epic (operations): <org>`: ops issues opened, failure comments and
  recoveries since the last review. Every read of an ops issue is
  `vrg-issue-read --roles automation` (epic #1467's trusted read
  command, vergil-project/vergil-tooling#4929); the check never reads an
  ops issue's body or comments through `vrg-gh`. Run history stays
  as a backstop for runs that never reported, named as such, and stays
  the source for the two security-signal jobs (`dependabot-audit`,
  `github-config`), whose failures it keeps recording only as "a
  security-related failure" with a neutral security record. Leave
  "Security and audit signals" unchanged.
- [ ] **Step 2:** `vrg-container-run -- vrg-validate`. Expect PASS.
- [ ] **Step 3:** Commit `vrg-commit --type docs --scope hoshin --message
  "read failures from the operations queue in the daily review"`, then
  report-ready.

### Task 10b: the morning review is installed (deployment, .github)

Blocked by 10.

- **Precondition (human-attested):** a vergil-claude-plugin release
  containing Task 10 is published and installed on the host and every VM,
  and a vergil-tooling release containing
  vergil-project/vergil-tooling#4929 (`vrg-issue-read`) is installed on
  the host.
- **Procedure:** `vrg-git merge-base --is-ancestor <Task 10 merge-sha>
  <plugin tag-sha>` in vergil-claude-plugin exits 0; the installed
  plugin's `skills/hoshin/references/daily.md` reads the operations
  epics; `vrg-issue-read --help` runs on the host.
- **SUCCESS:** all three hold.

### Task 11: the one-off: checks and operations epics (deployment, .github)

Spec §5.6. Blocked by 3.

- **Procedure:**
  1. For each of `vergil-project`, `logical-minds-foundry`,
     `mq-rest-admin-project` and `mnemosys-project`: the org variables
     `VRG_AUTOMATION_APP` and `VRG_TRUSTED_AUTHORS` exist
     (`vrg-trusted-authors check`, from epic #1467), and the App secrets
     exist in each repo that runs an in-scope job (listed in the result).
     A missing one is a FAILURE naming it.
  2. Create each org's operations epic with Task 2a's ensure-only
     command, and confirm it
     carries `epic`, `operations` and `epic-management`.
- **SUCCESS:** four operations epics exist, each check passed.

### Task 12: vergil-actions reporting release (deployment, .github)

Blocked by 7, 6a–6t, 11.

- **Precondition (human-attested):** a vergil-actions release containing
  Task 7 is tagged and `v2.1` moved to it.
- **Procedure:** `vrg-git merge-base --is-ancestor <Task 7 merge-sha>
  <v2.1-tag-sha>` exits 0; every caller task 6a–6t is closed.
- **SUCCESS:** both hold.

### Task 13: the reporter is live (validation, .github)

Blocked by 8, 10b, 12.

- **Procedure:** after the first nightly runs following 12 and 8:
  1. A job red at the time (for example vergil-containers
     `nightly-rebuild-dev`) has one open ops issue under
     `Epic (operations): vergil-project`, App-authored, titled per the
     Global Constraints, linking the run.
  2. A green job has no ops issue.
  3. A second red night adds one comment to the same issue.
  4. A recovery adds exactly one `recovered` comment and the issue stays
     open.
  5. hoshin's daily review (Task 10) lists the issue.
  6. **Every org and repo:** a table, one row per repo with a reporting
     job in `vergil-project`, `logical-minds-foundry`,
     `mq-rest-admin-project` and `mnemosys-project`, naming the repo, the
     job, its first run after rollout, and the reporter's outcome: an
     App-authored ops issue under that org's operations epic for a red
     job, nothing for a green one, and the reporting job itself green.
     A reporting job that failed is a FAILURE naming it.
- **SUCCESS:** steps 1–5 observed and linked, and every row of step 6
  as expected.

### Task 14: fixture runs (fixture-run)

Filed with the line in the delivery table once Task 12 exists. The
fixture runs exercise the released shared workflows, with the reporting
jobs, on each fixture.

### Task 15: GHCR login with the App token (vergil-containers)

Spec §5.4. Blocked by 13.

**Files:**

- Modify: `.github/workflows/cd-docker-publish.yml` (declares
  `APP_CLIENT_ID` and `APP_PRIVATE_KEY` as optional secrets and a boolean
  input `app-registry-login`, default `false`; when it is `true`, each
  push job mints an App token with `permission-packages: write` and logs
  in to GHCR with it, and a missing secret or a failed mint fails the job.
  There is no fallback to `GITHUB_TOKEN` once the input is `true`.) and
  `.github/workflows/ops.yml` (`nightly-rebuild-dev` passes
  `app-registry-login: true` and both secrets), and
  `.github/workflows/package-cleanup.yml` (a `workflow_dispatch` boolean
  input `app-token-dry-run`, default `false`: when `true`, the run is a
  dry run only and every GHCR call uses an App token minted with
  `permission-packages: write`, with no fallback; scheduled runs are
  unchanged)
- Test: `docker/versions/test_ops_reporting.py`
  (`test_nightly_rebuild_dev_uses_app_token`,
  `test_app_registry_login_has_no_github_token_fallback`,
  `test_cleanup_app_token_dry_run_is_dry_and_app_only`)

A reusable-workflow call job has no steps, and a minted token cannot
cross jobs as an output (GitHub blanks outputs holding a masked secret),
so the token is minted inside `cd-docker-publish.yml`'s push jobs.

- [ ] **Step 1:** Write the failing tests. Run them. Expect FAIL:
  `nightly-rebuild-dev` passes no `app-registry-login`, and
  `package-cleanup.yml` has no `app-token-dry-run` input.
- [ ] **Step 2:** Implement. Only `nightly-rebuild-dev` sets
  `app-registry-login: true`.
- [ ] **Step 3:** Run the test and `vrg-container-run -- vrg-validate`.
  Expect PASS. Then the refactor step.
- [ ] **Step 4:** Commit `vrg-commit --type ci --scope ops --message
  "log in to GHCR as the App for the dev rebuild"`, then report-ready.

### Task 16: the GHCR proof (validation, .github)

Blocked by 15.

- **Procedure:** the next nightly `nightly-rebuild-dev` run after 15
  merges logs in with the App token and pushes every `dev-*` image; a
  `package-cleanup` dispatch with `app-token-dry-run: true` (Task 15)
  lists the versions it could delete, using only the App token.
- **SUCCESS:** both succeed. **FAILURE:** record the error; Task 17 then
  records exceptions instead of switching.

### Task 16b: vergil-containers release for the App login (deployment, .github)

Blocked by 15. `nightly-rebuild-prod` calls `cd-docker-publish.yml@main`,
and passing an input or secret that `main`'s copy does not declare is a
workflow error that fails every job in `ops.yml`.

- **Precondition (human-attested):** a vergil-containers release
  containing Task 15 is merged to `main`.
- **Procedure:** `vrg-git merge-base --is-ancestor <Task 15 merge-sha>
  origin/main` in vergil-containers exits 0, and `main`'s
  `cd-docker-publish.yml` declares `app-registry-login`, `APP_CLIENT_ID`
  and `APP_PRIVATE_KEY`.
- **SUCCESS:** both hold.

### Task 17: the remaining container jobs on the App (vergil-containers)

Spec §5.4. Blocked by 16, 16b.

- [ ] **Step 1:** Extend `test_ops_reporting.py` with
  `test_ghcr_jobs_use_app_token` for `nightly-rebuild-prod`,
  `image-publish`, `package-cleanup` and `package-cache-warm`;
  `test_no_ghcr_cleanup_token` (the `GHCR_CLEANUP_TOKEN` fallback is
  gone); and `test_prod_rebuild_passes_only_what_main_declares` (every
  input and secret `nightly-rebuild-prod` passes is declared by
  `origin/main`'s `cd-docker-publish.yml`). Run it. Expect FAIL.
- [ ] **Step 2:** Switch each job. If Task 16 failed, instead leave the
  jobs and record each as an exception with its reason in `spec.md`
  §4.1 by Task 19b, and drop this task's code steps.
- [ ] **Step 3:** Run the test and `vrg-container-run -- vrg-validate`.
  Expect PASS. Then the refactor step.
- [ ] **Step 4:** Commit `vrg-commit --type ci --scope ops --message "run
  the container jobs as the App"`, then report-ready.

### Task 18: shared workflows on the App (vergil-actions)

Spec §5.4. Blocked by 13. `cd-docs-refresh.yml` nests `cd-docs.yml@develop`,
so this task's `cd-docs.yml` changes reach the docs refresh at merge, not
at Task 19; that is safe only because every caller passes the App
secrets by then (Task 12).

**Files:**

- Modify: `.github/workflows/publish-index.yml`, `cd-docs.yml`,
  `cd-docs-refresh.yml` (their write steps use an App token instead of
  `GITHUB_TOKEN`); `ops-github-config.yml` (`branch-sweep`,
  `dependabot-audit`), `ops-epic-sweep.yml`, `ops-epic-rollup.yml` (each
  checks its token against `VRG_AUTOMATION_APP`, as `github-config`
  does)
- Modify: `actions/cd/docs/deploy/action.yml` (its git identity becomes
  the App's `<slug>[bot]`, from an input, instead of
  `github-actions[bot]`)
- Create: `actions/ops/report/tests/app-identity.test.sh`

- [ ] **Step 1:** Write the failing shell test: no write step in the six
  workflows uses `secrets.GITHUB_TOKEN` or `github.token`; every
  write-path `actions/checkout` passes an App `token:`; the docs deploy
  action sets no `github-actions[bot]` identity; and the four jobs run
  the `VRG_AUTOMATION_APP` check before their first write. Confirm it
  fails on today's `cd-docs.yml` (it must not pass before the change).
  Run it. Expect FAIL.
- [ ] **Step 2:** Implement. A step whose operation the App cannot
  perform (found while implementing) keeps `GITHUB_TOKEN` with a comment
  giving the reason, and is listed in the PR notes for the exceptions
  record.
- [ ] **Step 3:** Run the test and `vrg-container-run -- vrg-validate`.
  Expect PASS. Then the refactor step.
- [ ] **Step 4:** Commit `vrg-commit --type ci --scope ops --message "run
  the shared unattended jobs as the App"`, then report-ready.

### Task 19: vergil-actions identity release (deployment, .github)

Blocked by 18. As Task 12, for Task 18's merge commit.

### Task 19b: record the identity exceptions (.github)

Spec §5.4. Blocked by 17, 18.

- [ ] **Step 1:** Collect every exception from Tasks 17 and 18 (their PR
  notes, and Task 16's outcome): the job, the step, the token it keeps
  and the reason.
- [ ] **Step 2:** Record each in `spec.md` §4.1 beside its job, and add a
  §5.4 line listing them. If there are none, say so in §5.4.
- [ ] **Step 3:** `vrg-container-run -- vrg-validate`. Expect PASS.
- [ ] **Step 4:** Commit `vrg-commit --type docs --scope epic-1494
  --message "record the identity exceptions"`, then report-ready.

### Task 20: every job runs as the App (validation, .github)

Blocked by 17, 19, 19b.

- **Procedure:** for each in-scope job in spec §4.1, the latest run's
  write steps authenticate as the App (its `[bot]` actor on pushes,
  issues and releases), or the job is in the recorded exceptions with
  its reason. The reporter still reports (no ops issue opened by the
  switch itself, or one that has recovered).
- **SUCCESS:** every job accounted for.

## Evolution during execution

### 2026-11-23: Task 7's identity test, Task 11's checks and Task 10's reads

**What changed:**

- **Task 7 also modifies
  `actions/ops/github-config/tests/automation-env.test.sh`**
  (vergil-actions). It extends the test to accept a step that sets
  `VRG_AUTOMATION_APP` from `vars` with no earlier token step and no App
  credentials from secrets, on `ops-report` `uses:` steps only. This adds
  the test to Task 7's **Files** list; the task text above stays as
  written.
- **Task 11's step-1 checks were run by the human,** as themselves. The
  deploy run (vergil-project/.github#1512) recorded the result from their
  output. Follow-ups filed: vergil-project/vergil-claude-plugin#1288 and
  vergil-project/vergil-tooling#5042.
- **Task 10 locates each operations epic with the default trusted
  roles.** Only the ops issues are read with `--roles automation`.
  `vrg-ops-report` is unaffected.
- **Follow-ups filed during execution, outside the plan,** all ad hoc:
  vergil-project/vergil-tooling#5021, vergil-project/vergil-tooling#5022,
  vergil-project/vergil-tooling#5042,
  vergil-project/vergil-claude-plugin#1288 and
  vergil-project/.github#1543.

**Why:**

- The test rejects any step that sets `VRG_AUTOMATION_APP` in its `env`
  without a `GH_TOKEN` from an earlier token step or the App credentials
  from secrets. The plan's convention, a reporting job's
  `uses: ./actions/ops/report` step setting `VRG_AUTOMATION_APP` from
  `vars`, has neither, because the `ops-report` action mints its own
  token. The run of Task 4 (vergil-project/vergil-actions#1361) found
  this.
- `vrg-trusted-authors check` refuses the user agent mode, and the agent
  cannot read Actions secrets.
- The four operations epics were created by the user agent in Task 11, so
  an automation-only read of an epic's body is refused. `vrg-ops-report`
  finds the epic by its title and labels, not by reading its body.

### 2026-11-23: Six caller tasks leave the epic, Task 12 stops waiting for them, Task 6d grows

**What changed:**

- **Six caller tasks left the epic for their repos' ad-hoc epics.** In
  each, the caller change is committed on its branch. By the human's
  ruling, the tasks no longer block the epic; each one's issue and branch
  stay in place, queued as ad-hoc work.
  - 6p mq-rest-admin-project/mq-rest-admin-common#236,
    6q mq-rest-admin-project/mq-rest-admin-dev-environment#180 and
    6r mq-rest-admin-project/mq-rest-admin-python#536.
  - 6s mq-rest-admin-project/mq-rest-admin-go#332 and
    6t mq-rest-admin-project/mq-rest-admin-rust#131.
  - 6m logical-minds-foundry/mq-resiliency-observability#84, whose branch
    had reached report-ready.
- **Task 12 (vergil-project/.github#1514) no longer waits for those
  six.** The human posted two scope changes on #1514. Task 12 still waits
  for Task 7 and the other 14 caller tasks.
- **Task 6d also changed the `vrg-repo-init` generator.**
  vergil-project/vergil-tooling#4965 changed `render_cd_workflow`, so new
  repos pass the App secrets to `cd-docs.yml`. This was the human's `fix`
  ruling, posted as a scope change on #4965.
- **Follow-ups filed during execution, outside the plan,** after those in
  the first entry, all ad hoc: vergil-project/vergil-containers#1010,
  vergil-project/vergil-actions#1381 and
  vergil-project/vergil-actions#1382.

**Why:**

- None of the six repos could be taken through validation, or, for 6m,
  the repo's CI does not work:
  - 6p, 6q and 6r each fail validation on problems already on develop: an
    unmerged `.gitignore` sync, a semgrep `secrets: inherit` finding on
    the `release` job and other pre-existing failures.
  - In 6s and 6t, the `[ci].versions` pins sit outside the published
    container window, so `vrg-container-run` will not start.
  - 6m's repo has no working CI, and the repo may be retired.
- Until each of the six lands, its repo's docs runs have a red reporting
  job, because no App secrets reach it. The human accepted that as a
  visible failure.
- vergil-tooling's parity test requires its own `cd.yml` to match the
  generated one.

### 2026-11-24: Task 13 closes on live evidence, Task 15 splits, the secret model follows epic 1384

**What changed:**

- **Task 13 (vergil-project/.github#1515) changed scope twice and closed
  SUCCESS.**
  - Step 6 counts a red unattended job as expected, so long as it has an
    ops issue, and leaves out the six repos that left the epic.
  - The task closes on the live evidence of steps 2 and 6. Steps 1, 3, 4
    and 5 rest on Task 2b's unit tests and the reporter's fail-red rule,
    not on failures caused on purpose.
  - vergil-project/vergil-containers#1013, a real ops issue that was
    opened, recovered and then closed at the morning review, was recorded
    as live evidence for steps 1 and 4 and part of step 5.
- **A dependency on epic vergil-project/.github#1384 was raised, then
  withdrawn.** The red `docs / report` runs in logical-minds-foundry and
  mnemosys-project had run on epic 1384's interim `secrets: inherit`
  callers. Epic 1384's by-name caller fix (logical-minds-foundry/docs#44)
  made them pass.
- **Task 15 was split.**
  - vergil-project/vergil-containers#989 kept the `package-cleanup.yml`
    App-token dry run.
  - The `nightly-rebuild-dev` GHCR login moved to
    vergil-project/vergil-containers#1029. It waited for epic 1384's
    ruling N0.16, which was B: pass the App keys by name to the repo's own
    local workflow, whose receiving jobs declare `environment: automation`.
    `INHERIT_ALLOWED` is unchanged. vergil-project/vergil-containers#1030
    corrected the test.
  - Tasks 16 and 16b wait for #1029. Task 17 uses the same by-name route.
- **Tasks 7, 8, 15 and 18 follow epic 1384's secret model.** The App keys
  live only in each repo's `automation` environment, and every job that
  reads them declares it. Same-org callers of vergil-actions workflows
  pass `secrets: inherit`; cross-org and local callers pass the keys by
  name.
- **Task 18 (vergil-project/vergil-actions#1363): N0.1 accepted.** The
  App-token docs deploy has no `GITHUB_TOKEN` fallback. From Task 19's
  release (vergil-actions v2.1.95), the six repos that left the epic stop
  deploying docs until each one's caller task lands. `publish-index`'s
  deploy keeps `GITHUB_TOKEN`, as a recorded exception for Task 19b.
- **The documentation-review sweep (vergil-project/vergil-tooling#4938)
  spawned four per-repo doc tasks** under the epic:
  vergil-project/docs#183, vergil-project/vergil-actions#1415,
  vergil-project/vergil-containers#1026 and
  vergil-project/vergil-claude-plugin#1334. Scope changes widened #1415
  and the ad-hoc follow-up vergil-project/vergil-actions#1418.
- **The follow-on brainstorm (vergil-project/.github#1497) filed four
  planning issues,** in order: vergil-project/.github#1603 (an org-level
  watcher), vergil-project/.github#1604 (missing jobs),
  vergil-project/.github#1605 (automated security records) and
  vergil-project/.github#1606 (splitting `ops.yml`).
- **Follow-ups filed during execution, outside the plan,** after those in
  the second entry, all ad hoc: vergil-project/vergil-claude-plugin#1302,
  vergil-project/vergil-claude-plugin#1303,
  vergil-project/vergil-claude-plugin#1304,
  vergil-project/vergil-claude-plugin#1340 and
  vergil-project/vergil-actions#1418. The `app-id` deprecation the fixture
  runs saw was triaged to the existing vergil-project/vergil-actions#1243.

**Why:**

- Task 13's second scope change said no red job had happened since the
  rollout. That was wrong: the search behind it covered only the four
  orgs' `.github` repos. A search of each whole org by label found #1013.
- `secrets: inherit` does not cross organizations, so the interim callers
  passed no App keys to the vergil-actions workflows. The first
  diagnosis at the epic's gate, that a by-name map cannot reach environment secrets,
  was wrong: keys passed by name reach a called workflow's jobs that
  declare the environment.
- The GHCR login could not land until epic 1384 ruled on how a repo
  reaches the App keys from its own local reusable workflow. The
  `package-cleanup.yml` dry run did not need that ruling.
- The plan's no-fallback rule holds for the docs deploy. The human
  accepted the six repos' stopped docs deploys as the cost of the caller
  tasks that left the epic.
