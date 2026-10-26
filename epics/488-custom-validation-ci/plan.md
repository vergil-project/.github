# Run repos' custom validation in PR CI — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Make each repo's custom validator a required part of PR CI. It runs
as `vrg-validate --check custom` in a new `custom` job, gated by
`quality / evidence`, and every custom validator in the active fleet passes it.

**Architecture:** vergil-tooling gains `--check custom`, which runs the
language install stage and then the existing custom stage, or exits 0 with
one line when the repo has no validator. vergil-actions' `ci-quality.yml`
gains a single-container `custom` job, modelled on `common`, whose transcript
becomes the quality evidence file `quality-custom.log`. vergil-tooling is
released, then vergil-actions under `v2.1`. The fleet then fails forward: each
repo with a validator is validated in real PR CI and fixed where needed.

**Tech Stack:** Python 3.12+ (`argparse`, pytest), GitHub Actions reusable
workflows, bash.

**Spec:** `epics/488-custom-validation-ci/spec.md`
(vergil-project/.github#488)

**Shape:** this is an epic-level plan. Each task below is **one issue and one
PR in the repo named in its heading**, worked in a worktree of that repo. Paths
are relative to that repo's root. Deployment and validation tasks are
operational: they have no code PR and close on `Outcome: SUCCESS`.

## Global Constraints

- The custom stage folds into the existing `quality / evidence` gate: **no new required check, and no ruleset change.**
- No custom validator: `--check custom` prints exactly
  `custom: no custom validator (scripts/bin/validate-custom or vrg-validate-custom) — nothing to run`
  and exits 0.
- A validator that is present but unusable (not a regular file, or not executable) **fails loudly**. This keeps the #3349 rule, through `_check_custom_validator`.
- A no-argument `vrg-validate` run is unchanged: custom runs last, and is omitted when there is no validator.
- `--all-versions` stays test-only: `--check custom --all-versions` is a usage error.
- The custom job runs **once**, on the same image as the `common` job: `<prefix>-<primary-container-suffix>:<primary-container-tag>`.
- Evidence contract: the partial is `ci-evidence-quality-reports-custom`, the file is `quality-custom.log`, and the tool is `{"name": "vrg-validate-custom"}`.
- Release order: vergil-tooling **before** vergil-actions. Releases are human-gated, and the agent never cuts one.
- vergil-tooling has a single deployed version, so there is no guard for tooling that predates `--check custom`.
- Active fleet: vergil-project, logical-minds-foundry and mq-rest-admin-project. The inventory source is `~/dev/projects/<org>/<repo>`, including `.github`, and never GitHub code search.
- Validation gate: `vrg-container-run -- vrg-validate`. Commit with `vrg-commit`. Hand off with `vrg-pr-workflow report-ready`.

## Review Focus

1. **A repo with no validator** running `--check custom` gets exit 0 and the exact line above, with no install stage run. Pinned by `test_main_check_custom_no_validator` (Task 1).
2. **A validator that is present but not executable** under `--check custom` fails with a non-zero exit and no silent skip. Pinned by `test_main_check_custom_non_executable_fails` (Task 1).
3. **A Python repo with a validator** runs install (fail_fast) before custom, so the validator sees dev dependencies. Pinned by `test_build_stages_check_custom_runs_install_then_custom` (Task 1).
4. **`--check custom --all-versions`** is a usage error, not a silent ignore. Pinned by `test_check_custom_rejects_all_versions` (Task 1).
5. **A failed or skipped custom job** blocks the merge: `quality / evidence`, the required gate, fails rather than going green by absence. Pinned by `actions/ci/evidence/tests/gate.test.sh` (Task 3), which fails if any `ci-quality` stage job is missing from `evidence.needs` or `RESULTS`. The custom job running on the same image as `common` is pinned by `image.test.sh` (Task 3).

---

### Task 1: `vrg-validate --check custom` (vergil-tooling, implementation)

**Files:**

- Modify: `src/vergil_tooling/bin/vrg_validate.py`: `_CHECK_KINDS`, the module docstring, `_build_stages` (custom block), and the empty-stages message in `main`.
- Test: `tests/vergil_tooling/test_vrg_validate.py`

**Interfaces:**

- Produces: the CLI contract `vrg-validate --check custom`. Tasks 2 and 3 rely on it, and on the exact no-validator line from Global Constraints.

- [ ] **Step 1: Write the failing tests** (follow the file's existing `_build_stages` and `main` test patterns, with `patch(_MOD + "...")`)

```python
def test_build_stages_check_custom_runs_install_then_custom() -> None:
    # language_commands -> [["echo", kind.value]]; _find_custom_validator -> "/path/to/custom"
    stages = _build_stages("custom", "python", Path("/tmp/r"))
    assert [s.name for s in stages] == ["install", "custom"]
    assert stages[0].mode == "fail_fast"
    assert stages[1].mode == "fail_defer"

def test_build_stages_check_custom_no_language() -> None:
    # language None, validator present
    assert [s.name for s in _build_stages("custom", None, Path("/tmp/r"))] == ["custom"]

def test_build_stages_check_custom_no_validator() -> None:
    # validator None: no install either
    assert _build_stages("custom", "python", Path("/tmp/r")) == []

def test_build_stages_check_lint_excludes_custom() -> None:
    # validator present; --check lint never adds custom
    assert "custom" not in [s.name for s in _build_stages("lint", "python", Path("/tmp/r"))]

def test_main_check_custom_no_validator(tmp_path, capsys) -> None:
    # _write_config(tmp_path); _find_custom_validator -> None
    assert main(["--check", "custom"]) == 0
    assert ("custom: no custom validator (scripts/bin/validate-custom or "
            "vrg-validate-custom) — nothing to run") in capsys.readouterr().out

def test_main_check_custom_runs_validator(tmp_path) -> None:
    # validator "/path/to/custom", _check_custom_validator patched, progress.run records argv -> 0
    assert main(["--check", "custom"]) == 0
    assert ran == ["/path/to/custom"]          # common and language stages not run

def test_main_check_custom_failure(tmp_path) -> None:
    # progress.run -> 1
    assert main(["--check", "custom"]) == 1

def test_main_check_custom_non_executable_fails(tmp_path) -> None:
    # real file tmp_path/scripts/bin/validate-custom, mode 0o644; shutil.which -> None
    assert main(["--check", "custom"]) == 1

def test_check_custom_rejects_all_versions() -> None:
    with pytest.raises(SystemExit) as exc:
        main(["--check", "custom", "--all-versions"])
    assert exc.value.code == 2
```

- [ ] **Step 2: Run them, and confirm they fail**

Run: `vrg-container-run -- uv run pytest tests/vergil_tooling/test_vrg_validate.py -k custom -v`
Expected: the new tests FAIL (`invalid choice: 'custom'`, or wrong stages). The existing custom tests PASS.

- [ ] **Step 3: Implement**

- Add `"custom": None` to `_CHECK_KINDS`. The language block already yields no kinds for it, because no `CheckKind` is named `custom`.
- `_build_stages`: change the custom block's guard from `check is None` to `check in (None, "custom")`. When `check == "custom"`, a validator is found, and `language` is not None, insert `_command_stage("install", install_cmds, mode="fail_fast")` before the custom stage if `language_commands(language, CheckKind.INSTALL, cpp_std=..., cpp_stdlib=...)` is non-empty.
- `main`: when `not stages` and `args.check == "custom"`, print the exact no-validator line and return 0. Other checks keep the existing `No <check> commands …` message.
- Docstring: add `custom` to the list of single checks.

- [ ] **Step 4: Run the tests, then full validation**

Run: `vrg-container-run -- uv run pytest tests/vergil_tooling/test_vrg_validate.py -v`. All PASS.
Run: `vrg-container-run -- vrg-validate`. It passes with 100% coverage.

- [ ] **Step 5: Commit and hand off**

`vrg-commit --type feat --scope validate --message "add --check custom to run the custom validator stage alone"`, then `vrg-pr-workflow report-ready`.

---

### Task 2: Release vergil-tooling with `--check custom` (vergil-tooling, deployment)

Blocked by Task 1. **Preconditions (human-attested):** a vergil-tooling
release containing Task 1 has been cut, and the `v2.1` line resolves to it.

- [ ] Probe: in a fresh container, `vrg-container-run -- vrg-validate --check custom` on any repo with no validator prints the exact no-validator line and exits 0.
- [ ] Record `Outcome: SUCCESS` with the released version.

---

### Task 3: `custom` job in `ci-quality.yml` (vergil-actions, implementation)

Blocked by Task 2, because this repo's own PR CI runs the job against the
released tooling. The PR touches `.github/workflows/**`, so the agent's push
is refused: `report-ready` records the state anyway, and the human's
`vrg-submit-pr` pushes the branch.

**Files:**

- Modify: `.github/workflows/ci-quality.yml`
- Modify: `actions/shared/setup/primary-suffix/tests/image.test.sh`, the single-container job registry.
- Create: `actions/ci/evidence/tests/gate.test.sh`, the evidence-gate wiring test.
- Modify: `.github/workflows/smoke-setup-vergil.yml`, to run `gate.test.sh` and update its header comment listing the single-container jobs.
- Modify: `.github/workflows/README.md`, `docs/site/docs/workflows/index.md`, `docs/site/docs/ci-gates/all-hard-gates.md`. These are same-repo docs that describe the quality jobs and evidence files.

**Interfaces:**

- Consumes: `vrg-validate --check custom` (Task 1).
- Produces: the job `custom` (check name `quality / custom`), the partial `ci-evidence-quality-reports-custom`, and `quality-custom.log` in `ci-evidence-quality`.

- [ ] **Step 0a (RED): register `custom` as a single-container job**

In `image.test.sh`, add `"ci-quality.yml:custom"` to `jobs=` and update the
header comment that lists the single-container jobs.
Run: `bash actions/shared/setup/primary-suffix/tests/image.test.sh`. Expected:
`not ok` for `ci-quality.yml:custom`, because the job doesn't exist yet.

- [ ] **Step 0b (RED): write the evidence-gate wiring test**

`actions/ci/evidence/tests/gate.test.sh` is framework-free, uses `yq` and `jq`,
and follows `image.test.sh`'s `ok`/`not ok` style. For `ci-quality.yml`: every
job except `matrix`, `setup` and `evidence` must be in `jobs.evidence.needs`,
and must have a line `<job>=${{ needs.<job>.result }}` in the `RESULTS` env of
the step named `Assert all required legs succeeded`. Exit non-zero on any
`not ok`.
Run: `bash actions/ci/evidence/tests/gate.test.sh`. Expected: PASS on today's
workflow (common, lint and typecheck are wired). It goes RED in Step 2 when
`custom` is added before Step 3 wires it.
Add it to `smoke-setup-vergil.yml` next to the other `*.test.sh` steps.

- [ ] **Step 1: `setup` job exports `primary-version`**

Add the output `primary-version: ${{ steps.derive.outputs.primary-version }}`.

- [ ] **Step 2: Add the `custom` job** after `common`

`needs: setup`, and a `container:` expression identical to `common`'s. Steps,
in order:

1. `actions/checkout@v6`
2. `./actions/shared/setup/vergil`
3. When `inputs.language == 'python'`: `./actions/ci/toolchain/pin-python` with `version: ${{ needs.setup.outputs.primary-version }}`, then `uv sync --group dev --frozen`.
4. A bash step with `set -euo pipefail; mkdir -p ci-reports; vrg-validate --check custom 2>&1 | tee ci-reports/quality-custom.log`.
5. `if: always()` upload of `ci-reports/` as `ci-evidence-quality-reports-custom`, with `if-no-files-found: warn`.

Add a comment block in the style of the file's existing comments, citing epic
vergil-project/.github#488.

- [ ] **Step 3: Wire the evidence job**

- `needs: [common, custom, lint, typecheck]`.
- Add `custom=${{ needs.custom.result }}` to `RESULTS`.
- Add `evidence-reports/quality-custom.log` to `files`.
- Add `{"name": "vrg-validate-custom"}` to `tools`.
- Update the job's header comment.

Between Steps 2 and 3, run `gate.test.sh`. Expected: RED (`custom` is not in
`evidence.needs` or `RESULTS`). After Step 3, it is GREEN.

- [ ] **Step 4: Verify (GREEN)**

Run: `bash actions/shared/setup/primary-suffix/tests/image.test.sh`. It passes,
including `ci-quality.yml:custom`, so the job runs on the same image as `common`.
Run: `bash actions/ci/evidence/tests/gate.test.sh`. It passes. This is Review Focus 5.
Run: `vrg-container-run -- vrg-validate`. actionlint, yamllint and shellcheck pass.

- [ ] **Step 5: Update the same-repo docs**

Wherever these docs list the quality jobs or `quality-common.log`, add the
`custom` job and `quality-custom.log`.

- [ ] **Step 6: Commit and hand off**

`vrg-commit --type feat --scope ci-quality --message "run the custom validation stage in PR CI under quality / evidence"`, then `report-ready`.

**Acceptance, checked on the PR's CI:** `quality / custom` is green and its
log shows the no-validator line, since vergil-actions has none.
`ci-evidence-quality` contains `quality-custom.log`, and `quality / evidence`
is green.

---

### Task 4: Release vergil-actions with the custom job (vergil-actions, deployment)

Blocked by Task 3. **Preconditions (human-attested):** a vergil-actions release
containing Task 3 has been cut, and `v2.1` points at it.

- [ ] **Re-run the fleet inventory** just before the release:
  `for d in ~/dev/projects/{vergil-project,logical-minds-foundry,mq-rest-admin-project}/*/ ~/dev/projects/{vergil-project,logical-minds-foundry,mq-rest-admin-project}/.github/`.
  A repo has a validator if `scripts/bin/validate-custom` exists, or its `pyproject.toml` declares `vrg-validate-custom`. Record the list. For each repo not in Tasks 6–10, file a validation task. Use `--epic vergil-project/.github#488` inside vergil-project. Elsewhere, use `--epic adhoc --repo <org>/<repo>` and post a `Ref` comment on #488.
- [ ] **Probe:** `ci-quality.yml` at the `v2.1` tag contains the `custom` job.
- [ ] Record `Outcome: SUCCESS` with the release version and the inventory.

---

### Task 5: vergil-vm — a missing `tofu` is a hard failure (vergil-vm, implementation)

This task is not blocked. Its scope is independent of the release.

**Files:** Modify: `scripts/bin/validate-custom`, in its section 2.

- [ ] Replace the `else` branch's `SKIP: … (vergil-docker#352)` message, which still exits 0, with `echo "ERROR: required tool not found on PATH: tofu" >&2; fail=1`. Update the header comment to say a missing `tofu` fails.
- [ ] Verify the pass path: `vrg-container-run -- vrg-validate` passes, and the custom-stage log shows `tofu fmt/validate/test` lines with no `SKIP`.
- [ ] Verify the fail path: `vrg-container-run -- bash -c 'mkdir -p /tmp/notofu && for t in $(ls "$(dirname "$(command -v tofu)")"); do [ "$t" = tofu ] || ln -sf "$(dirname "$(command -v tofu)")/$t" /tmp/notofu/; done; PATH=/tmp/notofu:/usr/bin:/bin scripts/bin/validate-custom'` exits non-zero with the ERROR line. Any equivalent that hides only `tofu` from PATH is fine.
- [ ] `vrg-commit --type fix --scope validate --message "fail the custom validator when tofu is missing instead of skipping"`, then `report-ready`.

---

### Tasks 6–10: per-repo validation (one `validation` task per repo, in that repo)

| Task | Repo | Blocked by |
| --- | --- | --- |
| 6 | vergil-project/packages | 4 |
| 7 | vergil-project/vergil-claude-plugin | 4 |
| 8 | vergil-project/vergil-containers | 4, 11 |
| 9 | vergil-project/vergil-python | 4 |
| 10 | vergil-project/vergil-vm | 4, 5 |

Each task:

- [ ] Use a PR opened after the Task 4 release. Any real PR will do; a no-op docs PR is fine.
- [ ] Check that `quality / custom` succeeded, that its log names `scripts/bin/validate-custom` (not the no-validator line), that `quality-custom.log` is in the `ci-evidence-quality` artifact, and that `quality / evidence` is green.
- [ ] For vergil-vm only, also check that the log has no `SKIP` line and does show `tofu fmt/validate/test`.
- [ ] On failure: file a fix task in that repo with `vrg-issue-create --epic vergil-project/.github#488 --repo <repo>` and a title naming the failing check, and comment on this task with `blocked: fix task <ref>`. Re-check after the fix merges.
- [ ] Record `Outcome: SUCCESS` with the PR and run URLs.

---

### Task 11: vergil-containers — remove the duplicate `quality / pins` job (vergil-containers, implementation)

Blocked by Task 4.

**Files:** Modify: `.github/workflows/ci.yml` to delete the `pins` job.
Modify: `scripts/bin/validate-custom` to drop the "It runs every step of the
CI pins job…" sentence and the "Keep this list in step with the pins job"
line, rewording the header so it describes the five checks directly.

- [ ] Precondition (human-attested, because `vrg-gh` does not allow `ruleset` for agents): `quality / pins` is not a required status check in the repo's ruleset. If it is, or nobody has confirmed it, stop and comment `blocked: confirm quality / pins is not a required check (human drops it from the ruleset if it is)`. Never edit rulesets as the agent.
- [ ] Make the edits. `vrg-container-run -- vrg-validate` passes.
- [ ] `vrg-commit --type ci --scope pins --message "drop the quality / pins job now that the custom stage runs in PR CI"`, then `report-ready`. The push is refused (`.github/workflows/**`), so the human's `vrg-submit-pr` pushes the branch.
- [ ] **Acceptance on this PR's CI:** `quality / custom` is green, and its log shows all five pins and versions checks running. That is the proof that removing the job loses nothing.

---

### Bookends (already filed)

- Docs: vergil-project/.github#489. This plan and the spec, in one PR.
- Documentation review: vergil-project/vergil-tooling#3649, blocked by Tasks 1–11. It sweeps `docs/site/docs/guides/ci-architecture.md`, `validation-matrix.md`, `ci-evidence-convention.md`, the `CLAUDE.md` "local matches PR CI" text, and the `vrg-validate` reference. It spawns same-repo doc tasks for any other repo that needs one.
- Retrospective: vergil-project/.github#490. Terminal.

## Self-Review

- **Spec coverage:** §3.1 is Task 1. §3.2 is Task 3, including the `primary-version` export. §3.3 (release order and fail forward) is Tasks 2 and 4. §3.4: the inventory is Task 4, validation is Tasks 6–10, cross-org tracking is in Task 4, the vergil-vm hard failure is Task 5, and the pins removal is Task 11. §4 is covered by the validation tasks plus the docs review.
- **Ordering:** Task 3's own CI needs the released tooling (Task 2). Task 11's PR CI needs the live job (Task 4) to prove nothing is lost.
- **Alignment (paad, 2026-10-26):** Task 3 gains two red/green tests: the `image.test.sh` registry entry, and the new `gate.test.sh` evidence-wiring test.
- **Names:** `--check custom`, `quality-custom.log`, `ci-evidence-quality-reports-custom` and `vrg-validate-custom` are used identically in Tasks 1, 3 and 6–11.
