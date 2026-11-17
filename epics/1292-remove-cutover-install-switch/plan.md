# Remove the cutover vergil-tooling install switch — implementation plan

- **Epic:** vergil-project/.github#1292
- **Planning issue:** vergil-project/.github#983 (review history is in its
  comments)
- **Status:** draft, in design review

> **For agentic workers:** each code task is one issue and one PR in the repo
> named in its heading, worked with `issue-implement`; the deployment task is
> worked with `issue-deploy`.

**Goal:** In a container, vergil-tooling's cache build and vergil-actions'
setup action install vergil-tooling with no environment switch: the packaged
install for a released ref, a git install on the core Python for a dev ref.

**Architecture:** Pure deletion in two independent places. Each repo's tests
are changed first (deleted or rewritten to the remaining path), then the code
that only the deleted tests reached is removed.

**Tech Stack:** Python and pytest (vergil-tooling), bash and its shell test
harness (vergil-actions), Markdown site docs.

**Spec:** [`spec.md`](spec.md) in this directory.

## Global Constraints

- Nothing replaces the switch: no new check, guard or test. Tests change only
  by deletion or by rewriting an existing test to the remaining path.
- Out of scope, unchanged: host-runner installs, the self-repo `uv sync`,
  `install-source: git`, `actions/package/setup`, the post-install check that
  `vrg-*` runs from `/usr/bin` on vergil-python, and `DOCKER_DEV_IMAGE`.
- The vergil-actions change ships as the only significant change in its
  release. The human holds every other vergil-actions change until that
  release is out.
- Validation runs through `vrg-container-run -- vrg-validate` in the task
  repo's worktree. Releases are human-gated.
- No fixture runs.

## Review Focus

Each of these is already pinned by an existing test that the change must keep
green; none needs a new test.

- A python-family image exports `UV_PYTHON`; a dev ref must still install on
  the core Python, not on `UV_PYTHON`
  (`test_setup_dev_ref_binds_core_python` passes `UV_PYTHON`).
- A dev ref on a host runner still installs with `--python 3.15` after
  `dev_install` stops calling the environment check
  (`case_dev_tag_installs_pinned_uv`).
- A released ref's packaged install still refuses a `vrg-whoami` that is
  not the package's or not on vergil-python
  (`test_setup_packaged_check_refuses_*`,
  `case_packaged_check_refuses_foreign_interpreter`).
- A released ref whose candidate does not match still fails, with no
  fallback (`test_setup_line_candidate_mismatch_refuses`,
  `case_container_candidate_mismatch`).
- vergil-tooling's own repo and `install-source: git` are unaffected
  (`test_setup_self_repo_unchanged`, `test_build_cached_image_self_repo_skips_uv_install`,
  `case_self_repo_*`, `case_install_source_git_*`).

## Tasks

| Task | Repo | Kind | What | Blocked by |
| --- | --- | --- | --- | --- |
| 1 | .github | task (#1293) | Spec and plan (this PR) | — |
| 2 | vergil-tooling | task | Cache build drops the switch | 1 |
| 3 | vergil-actions | task | Setup action drops the switch | 1 |
| 4 | vergil-actions | deployment | Isolated release, `v2.1` moved | 3 |
| — | vergil-tooling | bookend (#4649) | Documentation review | 2, 4 |
| — | .github | bookend (#1294) | Retrospective | all |

Tasks 2 and 3 are independent and land in either order. Task 2 ships in
vergil-tooling's next normal release and has no deployment task.

### Task 2: Remove the switch from `container_cache.py` (vergil-tooling)

**Files:**

- Modify: `src/vergil_tooling/lib/container_cache.py` (`_UV_FROM_GIT`,
  `_probe_lines`, `_dev_core_lines`, `vergil_tooling_setup_command` and its
  docstring)
- Test: `tests/vergil_tooling/test_container_cache.py`
- Modify: `docs/site/docs/guides/ci-architecture.md` (the paragraph starting
  "The image chooses how vergil-tooling is installed")

**Interfaces:**

- Produces: `vergil_tooling_setup_command(tag: str) -> str`, unchanged in
  signature and in its `bash -euo pipefail -c` contract.

- [ ] **Step 1: Delete the tests of removed paths**

  Delete `test_setup_dev_ref_without_core_is_todays_install`,
  `test_setup_dev_ref_refuses_more_than_one_core_python`,
  `test_setup_dev_ref_half_environment_refuses`,
  `test_setup_half_environment_refuses` and the `_TODAYS_INSTALL` constant.
  Drop `_Env`'s `source` parameter and the `_SOURCE_REL` constant: once the
  probe is gone nothing reads that file. `_Env` keeps its `core` parameter.

- [ ] **Step 2: Rewrite the tests that assert removed content for a released tag**

  The rule: a test that asserts the neither-branch's `uv tool install`, the
  probes or `_UV_FROM_GIT` for a released tag asserts the packaged install
  instead. A test that uses `uv tool install` as an order marker uses
  `vergil-tooling: packaged install (<tag>)` instead. On origin/develop
  today these are:

  - `test_build_cached_image_uses_uv_tool_install` and
    `test_build_cached_image_python_includes_uv_install`: assert
    `"install -y vergil-tooling" in setup_cmd` and
    `"uv tool install" not in setup_cmd`. The first keeps its `pip install`
    assertion and is renamed `test_build_cached_image_uses_packaged_install`;
    the second keeps its `uv sync --frozen --group dev` assertion.
  - `test_setup_line_ref_branches_on_environment`: rename to
    `test_setup_line_ref_packaged_fragment`; assert the fragment starts with
    `bash -euo pipefail -c` and a space, contains `vergil-tooling: packaged install
    (v2.1)`, `install -y vergil-tooling` and `apt-get -o`, and contains
    neither `/etc/apt/sources.list.d/vergil.sources` nor `uv tool install`.
  - `test_build_cached_image_prepends_apt_install_to_setup`,
    `test_compose_setup_slots_build_after_install_before_warmup`,
    `test_build_cached_image_bakes_build_command_into_setup`,
    `test_compose_setup_omits_warmup_on_unbootstrapped_repo` and
    `test_compose_setup_exports_build_env_before_every_step`: replace each
    `"uv tool install"` marker or membership check with
    `"vergil-tooling: packaged install (v2.0.7)"`.

  Then grep the file for `uv tool install`, `vergil.sources`, `refusing to
  guess` and `image without the vergil apt repository`. Every remaining hit
  must be a dev-ref test of the core-Python install, a `not in` assertion,
  or a comment updated to the new order marker.

- [ ] **Step 3: Run the tests to see the rewritten ones fail**

  Run `vrg-container-run -- uv run pytest -q` on
  `tests/vergil_tooling/test_container_cache.py`.
  Expected: exactly three tests FAIL, because the fragment still holds the
  probes and the neither branch:
  `test_build_cached_image_uses_packaged_install`,
  `test_build_cached_image_python_includes_uv_install` and
  `test_setup_line_ref_packaged_fragment`. The five marker rewrites pass
  both before and after the change: today's fragment already carries the
  packaged install's log line. Every other test passes.

- [ ] **Step 4: Remove the switch**

  In `container_cache.py`:

  - delete `_UV_FROM_GIT`, the `has_source` and `has_core` probes, both
    half-image refusals and the neither branch;
  - in `_dev_core_lines`, delete the "more than one core Python" refusal, so
    it is the log line and `exec uv tool install --quiet --python
    "${cores[0]}" {spec}`;
  - keep the preamble both paths need: `root="${VRG_CACHE_ROOT:-}"`,
    `shopt -s nullglob` and `cores=("$root"/opt/vergil/python/*/bin/python3)`.
    Whether it stays a function or is inlined is the builder's call;
  - rewrite the docstring of `vergil_tooling_setup_command`: a released tag
    is the packaged install, a dev tag the git install on the core Python;
    no environment probes.

  Drop `_CORE_PYTHON_DIR`, `_VERGIL_SOURCE` or other constants only if no
  remaining line uses them (`_packaged_lines` uses `_CORE_PYTHON_DIR`).

  Then grep `container_cache.py` and `test_container_cache.py` for
  "probe", "environment" and "refusing", and update every comment that
  describes the removed switch.

- [ ] **Step 5: Run the tests to see them pass**

  Run the same file again. Expected: PASS.

- [ ] **Step 6: Update the site docs**

  In `ci-architecture.md`, replace the paragraph describing the two checks
  with: a release version installs the vergil-tooling package at that
  release (keep the `vX.Y` line sentence and its link), the build checks that
  `vrg-*` runs from `/usr/bin` on vergil-python, and a branch is `uv tool
  install` from git on the core Python. Keep "The build log names the path
  it took." Remove "With neither …" and "An image with only one of the two
  fails the build."

- [ ] **Step 7: Validate and commit**

  Run: `vrg-container-run -- vrg-validate`
  Expected: PASS.
  Commit with `vrg-commit --type refactor --scope container`, message
  "remove the cutover install switch from the cache build".

### Task 3: Remove the container switch from `install-tooling.sh` (vergil-actions)

**Files:**

- Modify: `actions/shared/setup/vergil/install-tooling.sh` (header comment,
  `environment_state`, `dev_install`, `container_install`)
- Test: `actions/shared/setup/vergil/tests/install-tooling.test.sh`
- Modify: `actions/shared/setup/vergil/action.yml` (description)
- Modify: `docs/site/docs/actions/shared-setup-vergil.md`
- Modify: `.github/workflows/smoke-setup-vergil.yml` (header comment only)

**Interfaces:**

- Produces: the action's inputs, outputs and log lines for the remaining
  paths, unchanged.

- [ ] **Step 1: Delete the tests of removed paths**

  Delete `case_container_uv`, `case_container_half_source_only`,
  `case_container_half_runtime_only`, `case_dev_tag_container_two_cores_refused`
  and `case_dev_tag_container_half_refused`, and their lines in the run list
  at the bottom of the file. Keep `case_dev_tag_installs_pinned_uv` (host)
  and `case_dev_tag_container_binds_core_python`.

- [ ] **Step 2: Run the tests**

  Run `vrg-container-run -- bash` on
  `actions/shared/setup/vergil/tests/install-tooling.test.sh`.
  Expected: every case passes except `missing_pin_fails`, which always
  fails in the container: it runs `read-pin.sh` on `PATH=/usr/bin:/bin`,
  and the image has `python3` only in `/usr/local/bin`. That is why
  `scripts/bin/validate-custom` lists the suite in `CI_ONLY_ACTION_SUITES`.
  Leave that case alone.

- [ ] **Step 3: Remove the container switch**

  In `install-tooling.sh`:

  - delete `environment_state`;
  - `container_install` does, with no state check: log `packaged install
    (<tag>)`, `write_pin`, `apt_get update`, `check_candidate`,
    `packaged_install`, `ensure_uv`;
  - `dev_install` branches on `is_host`. On a host: the existing log line
    for a dev ref, `ensure_uv`, `uv_git_install "$tag"`. In a container:
    `find_core_pythons`, log `uv tool install from git on the core Python
    (<tag>)`, `ensure_uv`, `uv tool install --python "${core_pythons[0]}"
    …`, with no count check;
  - keep `find_core_pythons`, which the container dev path still uses;
  - rewrite the header comment's bullets for a dev tag and for a released
    tag in a container to the paths above;
  - grep `install-tooling.sh` and `tests/install-tooling.test.sh` for
    "probe", "environment" and "refusing", and update every comment that
    describes the removed switch (the `# The environment check` banner and
    the suite's header among them).

- [ ] **Step 4: Run the tests to see them pass**

  Run the suite again as in Step 2.
  Expected: every case passes except `missing_pin_fails`, including
  `case_container_packaged`, `case_dev_tag_container_binds_core_python`
  and `case_dev_tag_installs_pinned_uv`.

- [ ] **Step 5: Update the docs**

  `action.yml`'s description, `shared-setup-vergil.md` and the header
  comment of `smoke-setup-vergil.yml` describe the remaining paths: in a
  container, the packaged install for a released tag and a git install on
  the core Python for a dev tag; on a host runner, unchanged. Remove every
  mention of an image with neither or with only one of the two.

- [ ] **Step 6: Validate and commit**

  Run: `vrg-container-run -- vrg-validate`
  Expected: PASS. It skips `install-tooling.test.sh` as CI-only, so it does
  not run the changed paths.
  Commit with `vrg-commit --type refactor --scope setup-vergil`, message
  "remove the cutover install switch from container jobs".

  The PR touches `.github/workflows/**`, so GitHub refuses the agent's push;
  `vrg-pr-workflow report-ready` records the state and the human's
  `vrg-submit-pr` pushes the branch. The real gate for the changed paths is
  the PR's `smoke-setup-vergil` workflow, whose harness step runs the suite
  on a host runner; the report-ready notes say so.

### Task 4: vergil-actions release (deployment)

- **Precondition (human-attested):** a vergil-actions release containing
  Task 3's PR, as its only significant change, is published, and the
  floating `v2.1` tag points at it. The version is named in a comment on the
  task.
- **Checks:**
  1. The release notes list Task 3's PR and no other significant change.
  2. `refs/tags/v2.1` points at the release's exact tag (`vrg-git
     ls-remote`).
  3. `actions/shared/setup/vergil/install-tooling.sh` at `v2.1` has no
     `environment_state`.
  4. A CI run of a consumer repo other than vergil-tooling, started after
     the tag moved, is green, and a container job's log shows
     `vergil-tooling: packaged install`.
- **On failure:** revert `v2.1` to the previous release (a human action),
  then fix forward with a new task.
- **Afterwards:** the human's hold on other vergil-actions changes ends.
