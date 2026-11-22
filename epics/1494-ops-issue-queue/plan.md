# ops-issue-queue — implementation plan

- **Epic:** vergil-project/.github#1494
- **Planning issue:** vergil-project/.github#1470 (review history is in its comments)
- **Status:** draft, in design review

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
- **Security-signal jobs** (`dependabot-audit`) file no public ops issue
  (spec §2.3).
- **The App token is minted only on `refs/heads/develop`**; on any other
  ref the reporting step reports nothing (spec §5.3).
- **Release pushes report nothing** for `docs-deploy-develop` and
  `image-publish`: a push to `main`, or a develop push whose head commit
  is the merge of a `release/*` branch (spec §4.1).
- **Stable job names** (spec §4.1): `github-config`, `branch-sweep`,
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
- **3** (.github, deployment): the vergil-tooling release carrying 2a, 2b
  and vergil-project/vergil-tooling#4912, installed. Blocked by 2a, 2b.
- **4** (vergil-actions): the `ops-report` action, and optional App
  secrets declared on `cd-docs.yml`, `cd-docs-refresh.yml` and
  `publish-index.yml`. Blocked by 2b.
- **5** (.github, deployment): the vergil-actions release carrying 4,
  with `v2.1` moved. Blocked by 3, 4.

### Reporter: caller pass (one task per caller repo)

Each passes `APP_CLIENT_ID` and `APP_PRIVATE_KEY` to every call of
`cd-docs.yml`, `cd-docs-refresh.yml` and `publish-index.yml` in its repo.
All are blocked by 5.

- **6a–6m** (vergil-project): `docs`, `packages`, `vergil-actions` (its
  local `./.github/workflows/cd-docs.yml` call), `vergil-claude-plugin`,
  `vergil-containers`, `vergil-tooling`, `vergil-vm`,
  `vergil-fixture-python-lib`, `vergil-fixture-python-app`,
  `vergil-fixture-rust-lib`, `vergil-fixture-rust-app`,
  `vergil-fixture-ts-lib`, `vergil-fixture-ts-app`.
- **6n–6p** (logical-minds-foundry, under its proxy): `docs`,
  `mq-resiliency-observability`, `mq-resiliency-lab-for-linux`.
- **6q** (mnemosys-project, under its proxy): `docs`.
- **6r–6v** (mq-rest-admin-project, under its proxy):
  `mq-rest-admin-common`, `mq-rest-admin-dev-environment`,
  `mq-rest-admin-python`, `mq-rest-admin-go`, `mq-rest-admin-rust`.

### Reporter: reporting and rollout

- **7** (vergil-actions): reporting jobs in `ops-github-config.yml`,
  `ops-epic-sweep.yml`, `ops-epic-rollup.yml`, `ops-upstream-refresh.yml`,
  `cd-docs.yml`, `cd-docs-refresh.yml`, `publish-index.yml`. Blocked by 4
  and vergil-project/vergil-actions#1351.
- **8** (vergil-containers): reporting jobs for `nightly-rebuild-dev`,
  `nightly-rebuild-prod`, `image-publish`, `package-cleanup`,
  `package-cache-warm`. Blocked by 5.
- **9** (vergil-containers): deprecate `pin-exposure`. No blocker.
- **10** (vergil-claude-plugin): hoshin's daily review reads the
  operations epics. Blocked by 2a.
- **11** (.github, deployment): the one-off: App and trusted-author checks
  in each org, then each org's operations epic created. Blocked by 3.
- **12** (.github, deployment): the vergil-actions release carrying 7,
  with `v2.1` moved. Blocked by 7, 6a–6v, 11.
- **13** (.github, validation): the reporter is live. Blocked by 8, 12.
- **14** fixture runs: `fixture-run vergil-project/.github#1494
  python-lib,python-app,rust-lib,rust-app,ts-lib,ts-app --validates
  "vergil-actions reporting release (Task 12)" --after <Task 12 ref>`.
  Filed once 12 exists. Blocked by 12.

### Identity (after the reporter)

- **15** (vergil-containers): `nightly-rebuild-dev` logs in to GHCR with
  the App token. Blocked by 13.
- **16** (.github, validation): the GHCR proof: the next nightly
  `nightly-rebuild-dev` pushes with the App token. Blocked by 15.
- **17** (vergil-containers): `nightly-rebuild-prod`, `image-publish`,
  `package-cleanup` and `package-cache-warm` on the App token, or a
  recorded exception each. Blocked by 16.
- **18** (vergil-actions): `publish-index`, `cd-docs`, `cd-docs-refresh`
  on the App token, and the `VRG_AUTOMATION_APP` check added to
  `branch-sweep`, `dependabot-audit`, `epic-sweep` and `epic-rollup`.
  Blocked by 13.
- **19** (.github, deployment): the vergil-actions release carrying 18,
  with `v2.1` moved. Blocked by 18.
- **20** (.github, validation): every in-scope job runs as the App or has
  a recorded exception. Blocked by 17, 19.

### Bookends

- **Documentation review** (vergil-tooling, bookend
  vergil-project/vergil-tooling#4938). Blocked by 13 and 20. Its sweep
  covers the docs repo's `docs/site/docs/management/epics-and-tasks.md`
  (the epic kinds) and vergil-actions' workflow READMEs.
- **Follow-on brainstorm** (.github, bookend #1497). Blocked by 13.
- **Retrospective** (.github, bookend #1498). Blocked by everything else.

Notes:

- **Prerequisites from epic #1467:** the trust layer
  (vergil-project/vergil-tooling#4912) before 2b, and the delivery of
  `VRG_TRUSTED_AUTHORS` to jobs (vergil-project/vergil-actions#1351)
  before 7.
- **Order of the two vergil-actions releases** (spec §5.6): 5 declares
  the secrets as optional, the caller pass (6a–6v) passes them, and only
  then does 12 release the reporting jobs. Passing a secret a reusable
  workflow does not declare is a workflow error, and a reporting job
  without the secrets fails red, so neither order can be swapped.
- **`dependabot-audit` gets no reporting job** (spec §2.3). Its failure
  stays a red run, read by hoshin's existing "Security and audit signals"
  check, which records it through `vrg-security-record`; Task 10 keeps
  that check.
- **The language lifecycle job** is disabled by epic #1467 and gets no
  reporting job here; #1128's redesign adds one (spec §5.5).
- **If the GHCR proof (16) fails,** Task 17 records each GHCR job as an
  exception in `spec.md` §4.1 (a follow-up docs change in this
  directory) instead of switching it; the reporter is unaffected.
- **Filing.** Every task above is filed at epic-create's filing step, in
  its repo, under #1494 or its org's proxy, with its `Blocked-by:` refs.

---

### Task 2a: the operations standing kind (vergil-tooling)

Spec §5.1.

**Files:**

- Modify: `src/vergil_tooling/lib/epics.py` (beside `HOSHIN`,
  `hoshin_home`, `ensure_hoshin_epic`; `resolve_epic_ref`'s sentinel map;
  `drain_standing_org`)
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
  returning `(<owner>/.github, <owner>)` as `hoshin_home` does;
  `ensure_operations_epic(target_repo: str) -> IssueRef`.
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
  `test_drain_skips_operations_outside_org_home` (no operations drain
  for a self-homed queue home). In `test_data_labels.py`:
  `test_operations_labels_registered` with the exact descriptions above.
  In `test_vrg_issue_create.py`: `test_epic_operations_sentinel_accepted`.
- [ ] **Step 2: Run them.** `vrg-container-run -- pytest
  tests/vergil_tooling/test_epics.py tests/vergil_tooling/test_data_labels.py
  tests/vergil_tooling/test_vrg_issue_create.py -q`. Expect FAIL.
- [ ] **Step 3: Implement** `OPERATIONS`, `operations_home`,
  `ensure_operations_epic` and the sentinel entry. In
  `drain_standing_org`, replace the `kind is HOSHIN` home check with a
  module constant `ORG_HOMED_KINDS = (HOSHIN, OPERATIONS)`. Add the two
  labels and the new `epic-management` description to `labels.json`;
  list `operations` in the `--epic` help of both CLIs and in the CLI
  reference.
- [ ] **Step 4: Run the tests.** Expect PASS. Then the refactor step.
- [ ] **Step 5: Commit** `vrg-commit --type feat --scope epics --message
  "add the operations standing epic kind"`, then `vrg-container-run --
  vrg-validate` and report-ready.

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
  `previous_occurrence(repo: str, job: str) -> int | None`;
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
    (`previous occurrence: #41` in the body).
  - `test_report_failure_open_comments` → `"commented"`.
  - `test_report_success_after_failure_records_recovery` →
    `"recovered"`; `test_report_success_after_recovery_does_nothing`;
    `test_report_success_no_open_does_nothing`;
    `test_report_none_does_nothing` (no read, no write).
  - `test_report_two_open_comments_oldest` (comments on the lower number,
    naming the other; no close call).
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
  `author_role` is `Role.AUTOMATION`.
- [ ] **Step 4: Run the tests.** Expect PASS. Then the refactor step.
- [ ] **Step 5: Commit** `vrg-commit --type feat --scope ops --message
  "add vrg-ops-report for unattended job failures"`, then
  `vrg-container-run -- vrg-validate` and report-ready.

### Task 3: vergil-tooling release for the reporter (deployment, .github)

Spec §5.6. Blocked by 2a, 2b.

- **Precondition (human-attested):** a vergil-tooling release containing
  Tasks 2a, 2b and vergil-project/vergil-tooling#4912 is published and
  installed on the host and every VM, its version and tag named.
- **Procedure:**
  1. For the merge commit of each of those three PRs, `vrg-git
     merge-base --is-ancestor <merge-sha> <tag-sha>` in vergil-tooling
     exits 0.
  2. The `v2.1` line resolves to this release or later, so Actions jobs
     using `setup-vergil` install it.
  3. `vrg-ops-report --help` and `vrg-issue-create --help` (lists
     `operations`) run on this host.
- **SUCCESS:** every check passes, recorded with the version and tag.

### Task 4: the `ops-report` action and optional secrets (vergil-actions)

Spec §5.3, §5.6.

**Files:**

- Create: `actions/ops/report/action.yml`
- Create: `actions/ops/report/tests/ops-report-ref-guard.test.sh`,
  `actions/ops/report/tests/ops-report-inputs.test.sh`
- Modify: `.github/workflows/cd-docs.yml`,
  `.github/workflows/cd-docs-refresh.yml`,
  `.github/workflows/publish-index.yml` (`on.workflow_call.secrets`
  gains `APP_CLIENT_ID` and `APP_PRIVATE_KEY`, each `required: false`;
  no job uses them yet)
- Test: `actions/ops/report/tests/optional-secrets.test.sh`

**Interfaces:**

- Produces: the action `vergil-project/vergil-actions/actions/ops/report`
  with inputs `job` (stable name, required), `results` (comma list of
  `needs.*.result` values, required), `step` (optional), `app-client-id`
  and `app-private-key` (required). It sets up Vergil, passes
  `VRG_TRUSTED_AUTHORS` and `VRG_AUTOMATION_APP` from `vars` into the
  reporter's environment, mints an issues-only App token for the repo and
  its org `.github`, and runs `vrg-ops-report --repo
  ${{ github.repository }} --job … --results … --run-url <run URL>`.
- Produces: a first step that exits 0 without minting when
  `github.ref != 'refs/heads/develop'`.

- [ ] **Step 1: Write the failing shell tests** (the yq pattern of
  `actions/ops/github-config/tests/automation-env.test.sh`):
  `ops-report-ref-guard.test.sh` asserts the ref check precedes the token
  step and every later step is conditioned on it;
  `ops-report-inputs.test.sh` asserts the token requests only
  `permission-issues: write` and the reporter step receives
  `VRG_TRUSTED_AUTHORS` and `VRG_AUTOMATION_APP` from `vars`;
  `optional-secrets.test.sh` asserts the three workflows declare both
  secrets with `required: false`. Run them. Expect FAIL.
- [ ] **Step 2: Implement** the action and the three declarations.
- [ ] **Step 3: Run** the tests and `vrg-container-run -- vrg-validate`
  (actionlint). Expect PASS.
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

### Tasks 6a–6v: pass the App secrets (each caller repo)

Spec §5.6. Blocked by 5. One task per repo in the delivery table, each
titled `Pass the App secrets to the shared docs and publish workflows`.

- [ ] **Step 1:** In every job of `.github/workflows/*.yml` that calls
  `cd-docs.yml`, `cd-docs-refresh.yml` or `publish-index.yml`, add
  `secrets: { APP_CLIENT_ID: ${{ secrets.APP_CLIENT_ID }},
  APP_PRIVATE_KEY: ${{ secrets.APP_PRIVATE_KEY }} }` (block form). Keep
  any secrets already passed.
- [ ] **Step 2:** `vrg-container-run -- vrg-validate`. Expect PASS.
- [ ] **Step 3:** Commit `vrg-commit --type ci --scope cd --message "pass
  the App secrets to the shared docs workflows"`, then report-ready. The
  push is refused for a `.github/workflows/**` change; the human's
  `vrg-submit-pr` pushes it.

### Task 7: reporting jobs in the shared workflows (vergil-actions)

Spec §5.3, §4.1. Blocked by 4, vergil-project/vergil-actions#1351.

**Files:**

- Modify: `.github/workflows/ops-github-config.yml` (`report-github-config`,
  `report-branch-sweep`; none for `dependabot-audit`),
  `ops-epic-sweep.yml` (`report`), `ops-epic-rollup.yml` (`report`),
  `ops-upstream-refresh.yml` (`report`, `needs: [prepare, publish]`),
  `cd-docs.yml` (`report`, with the release-push skip),
  `cd-docs-refresh.yml` (`report`), `publish-index.yml` (`report`,
  `needs` every job)
- Create: `actions/ops/report/tests/reporting-jobs.test.sh`,
  `actions/ops/report/tests/ops-report-skipped.test.sh`,
  `actions/ops/report/tests/release-push-skip.test.sh`

**Interfaces:**

- Consumes: Task 4's action; the stable job names in Global Constraints.

- [ ] **Step 1: Write the failing shell tests:** `reporting-jobs.test.sh`
  asserts each workflow above has one reporting job per stable name, with
  `if: always()`, `needs` naming every GitHub job of that name, `results`
  built from those `needs.*.result` values, and `job` set to the stable
  name; and that `ops-github-config.yml` has no reporting job for
  `dependabot-audit`. `ops-report-skipped.test.sh` asserts the `results`
  passed for upstream-refresh include both `prepare` and `publish`.
  `release-push-skip.test.sh` asserts the `cd-docs.yml` reporting job is
  conditioned off for `refs/heads/main` and for a head commit that merges
  a `release/*` branch. Run them. Expect FAIL.
- [ ] **Step 2: Implement** the reporting jobs, each calling
  `./actions/ops/report` with the job's App secrets.
- [ ] **Step 3: Run** the tests and `vrg-container-run -- vrg-validate`.
  Expect PASS.
- [ ] **Step 4: Commit** `vrg-commit --type feat --scope ops --message
  "report every shared unattended job to the operations queue"`, then
  report-ready.

### Task 8: reporting jobs in vergil-containers (vergil-containers)

Spec §5.3, §4.1. Blocked by 5.

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
  `test_reporting_uses_ops_report_action_at_v2_1`. Run it. Expect FAIL.
- [ ] **Step 2: Implement** the reporting jobs, calling
  `vergil-project/vergil-actions/actions/ops/report@v2.1` with the App
  secrets the repo already holds.
- [ ] **Step 3: Run** the test and `vrg-container-run -- vrg-validate`.
  Expect PASS.
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
  Expect PASS.
- [ ] **Step 4:** Commit `vrg-commit --type ci --scope ops --message
  "deprecate the report-only pin-exposure job"`, then report-ready.

### Task 10: hoshin's daily review reads the queue (vergil-claude-plugin)

Spec §2.1 G5. Blocked by 2a.

**Files:**

- Modify: `skills/hoshin/references/daily.md` ("Check: Failed and
  cancelled runs")

- [ ] **Step 1:** Rewrite the check so its source is each org's
  `Epic (operations): <org>`: ops issues opened, failure comments and
  recoveries since the last review, read with `vrg-queue`-style commands
  through `vrg-gh`, counting only App-authored reports. Run history stays
  as a backstop for runs that never reported, named as such. Leave
  "Security and audit signals" unchanged: it still reads the
  `dependabot-audit` run and records security failures privately.
- [ ] **Step 2:** `vrg-container-run -- vrg-validate`. Expect PASS.
- [ ] **Step 3:** Commit `vrg-commit --type docs --scope hoshin --message
  "read failures from the operations queue in the daily review"`, then
  report-ready.

### Task 11: the one-off: checks and operations epics (deployment, .github)

Spec §5.6. Blocked by 3.

- **Procedure:**
  1. For each of `vergil-project`, `logical-minds-foundry`,
     `mq-rest-admin-project` and `mnemosys-project`: the org variables
     `VRG_AUTOMATION_APP` and `VRG_TRUSTED_AUTHORS` exist
     (`vrg-trusted-authors check`, from epic #1467), and the App secrets
     exist in each repo that runs an in-scope job (listed in the result).
     A missing one is a FAILURE naming it.
  2. Create each org's operations epic with `vrg-issue-create --epic
     operations` resolution (or `ensure_operations_epic`), and confirm it
     carries `epic`, `operations` and `epic-management`.
- **SUCCESS:** four operations epics exist, each check passed.

### Task 12: vergil-actions reporting release (deployment, .github)

Blocked by 7, 6a–6v, 11.

- **Precondition (human-attested):** a vergil-actions release containing
  Task 7 is tagged and `v2.1` moved to it.
- **Procedure:** `vrg-git merge-base --is-ancestor <Task 7 merge-sha>
  <v2.1-tag-sha>` exits 0; every caller task 6a–6v is closed.
- **SUCCESS:** both hold.

### Task 13: the reporter is live (validation, .github)

Blocked by 8, 12.

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
- **SUCCESS:** all five observed and linked.

### Task 14: fixture runs (fixture-run)

Filed with the line in the delivery table once Task 12 exists. The
fixture runs exercise the released shared workflows, with the reporting
jobs, on each fixture.

### Task 15: GHCR login with the App token (vergil-containers)

Spec §5.4. Blocked by 13.

**Files:**

- Modify: `.github/workflows/ops.yml` (`nightly-rebuild-dev` passes an
  App token minted with `permission-packages: write`) and
  `.github/workflows/cd-docker-publish.yml` (accepts a `registry-token`
  secret, used for the GHCR login when present, `GITHUB_TOKEN`
  otherwise)
- Test: `docker/versions/test_ops_reporting.py`
  (`test_nightly_rebuild_dev_uses_app_token`)

- [ ] **Step 1:** Write the failing test. Run it. Expect FAIL.
- [ ] **Step 2:** Implement. Only `nightly-rebuild-dev` passes the token.
- [ ] **Step 3:** Run the test and `vrg-container-run -- vrg-validate`.
  Expect PASS.
- [ ] **Step 4:** Commit `vrg-commit --type ci --scope ops --message
  "log in to GHCR as the App for the dev rebuild"`, then report-ready.

### Task 16: the GHCR proof (validation, .github)

Blocked by 15.

- **Procedure:** the next nightly `nightly-rebuild-dev` run after 15
  merges logs in with the App token and pushes every `dev-*` image; a
  manual `package-cleanup` dry run with the same token lists versions it
  could delete.
- **SUCCESS:** both succeed. **FAILURE:** record the error; Task 17 then
  records exceptions instead of switching.

### Task 17: the remaining container jobs on the App (vergil-containers)

Spec §5.4. Blocked by 16.

- [ ] **Step 1:** Extend `test_ops_reporting.py` with
  `test_ghcr_jobs_use_app_token` for `nightly-rebuild-prod`,
  `image-publish`, `package-cleanup` and `package-cache-warm`, and
  `test_no_ghcr_cleanup_token` (the `GHCR_CLEANUP_TOKEN` fallback is
  gone). Run it. Expect FAIL.
- [ ] **Step 2:** Switch each job. If Task 16 failed, instead leave the
  jobs and record each as an exception with its reason in `spec.md`
  §4.1 by a follow-up docs task, and drop this task's code steps.
- [ ] **Step 3:** Run the test and `vrg-container-run -- vrg-validate`.
  Expect PASS.
- [ ] **Step 4:** Commit `vrg-commit --type ci --scope ops --message "run
  the container jobs as the App"`, then report-ready.

### Task 18: shared workflows on the App (vergil-actions)

Spec §5.4. Blocked by 13.

**Files:**

- Modify: `.github/workflows/publish-index.yml`, `cd-docs.yml`,
  `cd-docs-refresh.yml` (their write steps use an App token instead of
  `GITHUB_TOKEN`); `ops-github-config.yml` (`branch-sweep`,
  `dependabot-audit`), `ops-epic-sweep.yml`, `ops-epic-rollup.yml` (each
  checks its token against `VRG_AUTOMATION_APP`, as `github-config`
  does)
- Create: `actions/ops/report/tests/app-identity.test.sh`

- [ ] **Step 1:** Write the failing shell test: no write step in the six
  workflows uses `secrets.GITHUB_TOKEN`, and the four jobs run the
  `VRG_AUTOMATION_APP` check before their first write. Run it. Expect
  FAIL.
- [ ] **Step 2:** Implement. A step whose operation the App cannot
  perform (found while implementing) keeps `GITHUB_TOKEN` with a comment
  giving the reason, and is listed in the PR notes for the exceptions
  record.
- [ ] **Step 3:** Run the test and `vrg-container-run -- vrg-validate`.
  Expect PASS.
- [ ] **Step 4:** Commit `vrg-commit --type ci --scope ops --message "run
  the shared unattended jobs as the App"`, then report-ready.

### Task 19: vergil-actions identity release (deployment, .github)

Blocked by 18. As Task 12, for Task 18's merge commit.

### Task 20: every job runs as the App (validation, .github)

Blocked by 17, 19.

- **Procedure:** for each in-scope job in spec §4.1, the latest run's
  write steps authenticate as the App (its `[bot]` actor on pushes,
  issues and releases), or the job is in the recorded exceptions with
  its reason. The reporter still reports (no ops issue opened by the
  switch itself, or one that has recovered).
- **SUCCESS:** every job accounted for.
