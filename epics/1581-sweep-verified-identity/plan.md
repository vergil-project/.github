# Verified-identity authorization for the epic sweep — implementation plan

- **Epic:** vergil-project/.github#1581
- **Planning issue:** vergil-project/vergil-tooling#3796 (review history is in
  its comments)
- **Status:** draft, in design review

> **For agentic workers:** each implementation task is one PR, run with
> `issue-implement`; operational tasks are run with `issue-deploy` or
> `issue-validate`. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** `vrg-epic-audit --close` admits the scheduled sweep only as the
verified automation identity, and a shared fail-closed agent check exists for
vergil-project/.github#593 to adopt.

**Architecture:** vergil-actions first passes `VRG_AUTOMATION_APP` to the
sweep's reconcile step and is released; vergil-tooling then gains
`identity_mode.agent_indicated()`, a `--close` gate of "verified automation, or
a human outside GitHub Actions with no agent indicated", and loses
`VRG_EPIC_SWEEP`; once that release is validated in every calling org,
vergil-actions drops the flag.

**Tech Stack:** Python 3 (vergil-tooling, pytest), GitHub Actions YAML and
Zensical/Markdown docs (vergil-actions).

**Spec:** [`spec.md`](spec.md) in this directory.

## Global Constraints

- The predicate is `identity_mode.agent_indicated() -> bool`, defined as
  `is_agent() or any_agent_signal()` (spec §4.1).
- `--close` is allowed only when `identity_mode.is_automation()`, or
  `identity_mode.is_human()` and not `identity_mode.agent_indicated()` and
  `os.environ.get("GITHUB_ACTIONS") != "true"` (spec §4.2).
- An `AutomationIdentityError` (including `ContradictoryIdentityError`) on the
  `--close` path exits 1 with the error's message on stderr, never a refusal,
  a fall-through or a traceback (spec §3, §4.2).
- `vrg-security-record list --details` refuses when `GITHUB_ACTIONS` is
  `true` (spec §4.4).
- `VRG_EPIC_SWEEP` is read by no code and set by no workflow once the epic
  ends (spec Goal 3); past release notes are not edited.
- Rollout order is fixed (spec §5): vergil-actions change → its release and
  per-org check → vergil-tooling change → its release → validation →
  vergil-actions flag removal.
- Releases and org-variable changes are human steps; agents never perform them.
- Public text stays at the mechanism level (security run).

## Review Focus

1. **The test suite under CI.** CI exports `GITHUB_ACTIONS=true`; every
   `--close` test that patches `is_human` to `True` would then be refused.
   Expected: the suite passes with `GITHUB_ACTIONS=true` in the environment
   (Task 4, Step 1 and its `conftest.py` change).
2. **An empty `VRG_AUTOMATION_APP`.** Expected: `--close` exits 1 with the
   "set but empty" message, no traceback (Task 4,
   `test_close_empty_automation_app_exits_1`).
3. **A token of a different App.** Expected: `--close` exits 1 naming the
   expected and actual login (Task 4,
   `test_close_wrong_app_token_exits_1`).
4. **The human on the host.** No signal, `GITHUB_ACTIONS` unset. Expected:
   `--close` still runs (Task 4, `test_close_allowed_for_human_off_actions`).
5. **The read-only report under Actions.** `vrg-epic-audit` without
   `--close` under `GITHUB_ACTIONS=true`. Expected: unchanged, exit 0 (Task 4,
   `test_report_without_close_unaffected_under_actions`).

## Delivery

| Task | Repo | Kind | What | Blocked by |
|---|---|---|---|---|
| 1 | `.github` | task | Spec and plan (vergil-project/.github#1582) | — |
| 2 | `vergil-actions` | task | Sweep step passes `VRG_AUTOMATION_APP`; workflow page lists it | — |
| 3 | `vergil-actions` | deployment | vergil-actions release carries Task 2; every calling org checked | 2 |
| 4 | `vergil-tooling` | task | Shared check, `--close` gate, flag retired, `--details` re-anchored, docs, tests | 3 |
| 5 | `vergil-tooling` | validation | Released tooling's sweep closes as automation in every calling org | 4 |
| 6 | `vergil-actions` | task | Drop `VRG_EPIC_SWEEP`; rewrite the workflow page's authorization text | 5 |
| 7 | `vergil-tooling` | task | Documentation review (vergil-project/vergil-tooling#5100) | 6 |
| 8 | `.github` | retrospective | Retrospective (vergil-project/.github#1583) | all others |

Task 3 is a deployment task: its closure is the signal that the workflow change
is live in `v2.1`, which Task 4 needs before it merges (spec §5 step 3). No
deployment task is needed for vergil-tooling: the sweep installs the newest
`v2.1.Z` tag at run time, so the release itself is the deploy, a human
precondition of Task 5.

When the tasks are filed, post the hand-off comment on vergil-project/.github#593
(spec §4.6) and hand the planning issue's advisories to Task 4, the task that
fixes them.

---

### Task 2: The sweep's reconcile step passes the automation identity (vergil-actions)

**Files:**

- Modify: `.github/workflows/ops-epic-sweep.yml` (the
  "Reconcile epic/task drift…" step's `env:`)
- Modify: `docs/site/docs/workflows/ops-epic-sweep.md` (Behavior, step 3)

**Interfaces:**

- Produces: the reconcile step's environment holds
  `VRG_AUTOMATION_APP: ${{ vars.VRG_AUTOMATION_APP }}` alongside the existing
  `GH_TOKEN`, `VRG_TRUSTED_AUTHORS` and `VRG_EPIC_SWEEP: "1"`.

- [ ] **Step 1: Add the variable to the step's `env:`**, after
  `VRG_TRUSTED_AUTHORS`, with a comment naming `vrg-epic-audit`'s verified
  automation identity (vergil-tooling `lib/automation_identity`). Keep
  `VRG_EPIC_SWEEP: "1"`; Task 6 removes it.
- [ ] **Step 2: Update the page.** Behavior step 3 lists `VRG_AUTOMATION_APP`
  in the step's environment. The authorization paragraph is unchanged here
  (Task 6 rewrites it).
- [ ] **Step 3: Validate.** Run `vrg-container-run -- vrg-validate`. Expected:
  pass, including `actions/ops/github-config/tests/automation-env.test.sh` and
  `actions/shared/setup/vergil/tests/trusted-authors-env.test.sh`.
- [ ] **Step 4: Commit** with `vrg-commit --type ci --scope ops-epic-sweep`.
  The push touches `.github/workflows/**`, so GitHub refuses the agent push;
  report-ready records the state and the human's `vrg-submit-pr` pushes it.

### Task 3: Release vergil-actions and confirm every calling org (deployment)

Filed with `vrg-issue-create --kind deployment --blocked-by <Task 2>`.

- **Preconditions:**
  - `vrg-release-record check vergil-project/vergil-actions#<Task 2>` exits 0.
    For exit 4, the cross-check is the whole check:
    `vrg-git merge-base --is-ancestor <Task 2 merge sha> <v2.1 tag sha>` exits 0.
  - Human-attested: the `v2.1` tag of vergil-actions includes Task 2. A
    release is a human step.
- **Human-run steps (need a human identity):**
  - The vergil-actions release that moves `v2.1`.
  - Per calling org, confirming that its org-level `VRG_AUTOMATION_APP`
    variable is set and non-empty. An org that does not have it is set up
    before the release, or named as excluded with its reason.
- **Procedure:**
  1. List the orgs whose `.github` calls `ops-epic-sweep.yml`. These are the
     orgs in the trusted-authors configuration with an `epic-sweep.yml`
     workflow. At planning time they were `vergil-project`,
     `logical-minds-foundry` and `mq-rest-admin-project`, the last with no run
     since 2026-09-01; `mnemosys-project` has no sweep.
  2. For each org that is not excluded, take the first `epic-sweep` run after
     the release (scheduled, or `workflow_dispatch` by the human). Confirm
     that its `epic-sweep` and `report` jobs both succeeded.
- **Acceptance:** every org that is not excluded has a green `epic-sweep` run
  on the released `v2.1`. Each exclusion is listed in the result with its
  reason.

### Task 4: The shared agent check and the verified-identity `--close` gate (vergil-tooling)

**Files:**

- Modify: `src/vergil_tooling/lib/identity_mode.py` (add `agent_indicated`)
- Modify: `src/vergil_tooling/bin/vrg_epic_audit.py` (gate at the `if
  args.close …` block; remove `_SWEEP_ENV` and `_automated_sweep`)
- Modify: `src/vergil_tooling/bin/vrg_security_record.py` (`_list`)
- Modify: `tests/vergil_tooling/conftest.py` (`_hermetic_identity`)
- Test: `tests/vergil_tooling/test_identity_mode.py`,
  `tests/vergil_tooling/test_vrg_epic_audit.py`,
  `tests/vergil_tooling/test_vrg_security_record.py`
- Docs: `docs/site/docs/reference/cli-tools-overview.md`,
  `docs/site/docs/reference/dev/security-record.md`,
  `docs/site/docs/standards/github-issues.md`

**Interfaces:**

- Consumes: `identity_mode.is_agent`, `any_agent_signal`, `is_human`,
  `is_automation`; `automation_identity.AutomationIdentityError`.
- Produces: `identity_mode.agent_indicated() -> bool`, for #593.

- [ ] **Step 1: Make the suite hermetic for `GITHUB_ACTIONS`.** In
  `_hermetic_identity`, add `monkeypatch.delenv("GITHUB_ACTIONS",
  raising=False)` beside the other identity variables, with a comment that the
  `--close` gate and the `--details` refusal read it. A test that needs it sets
  it itself. Run `GITHUB_ACTIONS=true vrg-container-run -- pytest
  tests/vergil_tooling -q`. Expected: the same result as without the variable.
- [ ] **Step 2: Write the failing predicate tests** in `test_identity_mode.py`,
  class `TestAgentIndicated`. The module manages its own signals; use its
  existing helpers and `_GRAPHQL` / `_viewer`:
  - `test_no_signal_is_false`: `agent_indicated() is False`
  - `test_agent_env_mode_is_true` (parametrized `user`, `audit`)
  - `test_human_env_with_mode_file_agent_is_true`
  - `test_human_env_with_app_pem_is_true`
  - `test_human_env_with_app_id_is_true`
  - `test_verified_automation_without_agent_signal_is_false`: the token is
    `<slug>[bot]`, so the result is `False`
  - `test_automation_failed_verification_raises`: a different login raises
    `AutomationIdentityError`
  - `test_automation_with_agent_signal_raises`: `app.pem` present and a
    verified token raise `ContradictoryIdentityError`
- [ ] **Step 3: Run and see them fail.** `vrg-container-run -- pytest
  tests/vergil_tooling/test_identity_mode.py -k AgentIndicated -q`. Expected:
  fail with `agent_indicated` undefined.
- [ ] **Step 4: Implement `agent_indicated() -> bool`** in
  `lib/identity_mode.py`. Its docstring states spec §4.1's contract: it raises
  when `VRG_AUTOMATION_APP` is set and verification fails or contradicts, and
  `is_agent()` is kept only for parity with the inline copies. Run Step 3
  again. Expected: pass.
- [ ] **Step 5: Write the failing gate tests** in `test_vrg_epic_audit.py`.
  Each test patches the network passes exactly as
  `test_close_allowed_for_scheduled_sweep` does today:
  - `test_close_allowed_for_verified_automation`: `is_automation` patched
    `True`, so rc is 0 and `close_drift` is called once
  - `test_close_allowed_for_human_off_actions`: no signal, `GITHUB_ACTIONS`
    unset, so rc is 0
  - `test_close_refused_for_human_under_actions`: `GITHUB_ACTIONS=true`, no
    signal, so rc is 1, stderr names both allowed callers, and
    `github.detect_org` is not called
  - `test_close_refused_when_agent_indicated`: `VRG_IDENTITY_MODE=human` with
    `VRG_APP_ID` set, so rc is 1
  - `test_close_ignores_retired_sweep_flag`: `VRG_EPIC_SWEEP=1`,
    `GITHUB_ACTIONS=true`, no automation, so rc is 1
  - `test_close_empty_automation_app_exits_1`: `VRG_AUTOMATION_APP=""`, so rc
    is 1, stderr has "is set but empty" and no traceback
  - `test_close_wrong_app_token_exits_1`: `VRG_AUTOMATION_APP` set and the
    GraphQL viewer is a different `[bot]` login, so rc is 1 and stderr names
    both logins
  - `test_report_without_close_unaffected_under_actions`:
    `GITHUB_ACTIONS=true` and no `--close`, so rc is 0 as before
- [ ] **Step 6: Rewrite the sweep-path tests (spec §4.3).** Every test that sets
  `VRG_EPIC_SWEEP` (on develop at planning time: `test_close_refused_for_agent`,
  `test_close_allowed_for_scheduled_sweep`, `test_close_sweep_also_drains_adhoc`,
  `test_close_runs_trusted_checks` and the tests near lines 1262, 1301, 1324,
  1403, 1426 and 1497) instead runs with
  `patch(f"{_MOD}.identity_mode.is_automation", return_value=True)`. Its
  `is_human` patch and the flag go. None is deleted, and none falls back to
  the human path. Then `vrg-git grep -n VRG_EPIC_SWEEP -- src tests` prints
  nothing.
- [ ] **Step 7: Run and see the gate tests fail.** `vrg-container-run --
  pytest tests/vergil_tooling/test_vrg_epic_audit.py -q`. Expected: the new
  refusal and automation tests fail.
- [ ] **Step 8: Implement the gate** in `bin/vrg_epic_audit.py`: a
  `_close_allowed() -> bool` built from the Global Constraints formula,
  checking `is_automation()` first. It is called where
  `identity_mode.is_human() or _automated_sweep()` is today. Wrap it so that
  `AutomationIdentityError` prints `vrg-epic-audit: <message>` to stderr and
  returns 1, following `bin/vrg_ops_report.py`'s handling. The refusal message
  names the verified automation identity and a human outside GitHub Actions,
  and no flag. Delete `_SWEEP_ENV`, `_automated_sweep` and their comments.
  Run Step 7 again. Expected: pass.
- [ ] **Step 9: Re-anchor `--details`.** Update
  `test_list_details_refused_under_epic_sweep` in `test_vrg_security_record.py` (it
  sets `VRG_EPIC_SWEEP=1` today): set `GITHUB_ACTIONS=true` instead, assert
  that stderr has `--details is refused under GitHub Actions`, and keep its
  "plain `list` still works" assertion. In `_list`, the condition becomes
  `os.environ.get("GITHUB_ACTIONS") == "true"` and the message reads
  `--details is refused under GitHub Actions (GITHUB_ACTIONS is true):
  advisory text must never reach a log`. Run `vrg-container-run -- pytest
  tests/vergil_tooling/test_vrg_security_record.py -q`. Expected: pass.
- [ ] **Step 10: Docs.** Rewrite each mention of `VRG_EPIC_SWEEP` in the three
  doc pages to the new rules: `--close` takes the verified automation identity
  or a human outside GitHub Actions, and `--details` is refused under GitHub
  Actions. `vrg-git grep -n VRG_EPIC_SWEEP -- docs/site` prints nothing.
- [ ] **Step 11: Validate and commit.** Run `vrg-container-run --
  vrg-validate`, then `GITHUB_ACTIONS=true vrg-container-run -- pytest
  tests/vergil_tooling -q` (Review Focus 1). Expected: both pass. Commit with
  `vrg-commit --type feat --scope identity`.

### Task 5: Validate the released gate in the real sweep (validation)

Filed with `vrg-issue-create --kind validation --blocked-by <Task 4>`.

- **Preconditions:**
  - `vrg-release-record check vergil-project/vergil-tooling#<Task 4>` exits 0.
    For exit 4, the cross-check is the whole check:
    `vrg-git merge-base --is-ancestor <Task 4 merge sha> <release tag sha>`
    exits 0.
  - Task 3 is closed.
- **Procedure:** for each calling org that Task 3 did not exclude, take the
  first `epic-sweep` run that installed a vergil-tooling release containing
  Task 4. Its log names the installed version. Confirm that the reconcile step
  passed the `--close` gate and that both jobs succeeded.
- **Acceptance:** a green run on the released tooling in every org that is
  not excluded.

### Task 6: Retire the sweep flag from the workflow (vergil-actions)

**Files:**

- Modify: `.github/workflows/ops-epic-sweep.yml` (delete `VRG_EPIC_SWEEP: "1"`
  and its comment)
- Modify: `docs/site/docs/workflows/ops-epic-sweep.md` (Behavior step 3; the
  paragraph beginning "`vrg-epic-audit --close` is otherwise a human-only
  action")

- [ ] **Step 1: Remove the flag** and its two comment lines from the reconcile
  step.
- [ ] **Step 2: Rewrite the page.** Behavior step 3 no longer lists the flag.
  The authorization paragraph says that `--close` admits the verified
  automation identity (the App token, checked against `VRG_AUTOMATION_APP`) or
  a human outside GitHub Actions, and that no environment flag authorizes it.
- [ ] **Step 3: Validate and commit.** Run `vrg-container-run --
  vrg-validate`. Expected: pass, and `vrg-git grep -n VRG_EPIC_SWEEP` prints
  nothing outside past release notes. Commit with `vrg-commit --type ci
  --scope ops-epic-sweep`. The push touches `.github/workflows/**`, as in
  Task 2.

### Task 7 and Task 8: Bookends

- **Task 7** (vergil-project/vergil-tooling#5100): the documentation-review
  sweep, per its issue. It spawns a same-repo doc task for any page outside
  vergil-tooling that still describes the old gate.
- **Task 8** (vergil-project/.github#1583): the retrospective, with
  `epic-retrospective`, once every other task is closed.
