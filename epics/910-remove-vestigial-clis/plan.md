# Remove vestigial CLIs and skills: implementation plan

- **Epic:** vergil-project/.github#910
- **Planning issue:** vergil-project/.github#831 (review history is in its
  comments)
- **Status:** draft

> **For agentic workers:** each task below is filed as its own GitHub issue
> in the repo its PR lands in, and run with `issue-implement`. Steps use
> checkbox (`- [ ]`) syntax for tracking.

**Goal:** remove seven unused `vrg-*` commands, the standalone
`vrg-repo-profile` command and the `migrate-repo` skill, with their tests,
docs and references.

**Architecture:** this is deletion only. Nothing is added, and no remaining
command changes behaviour. Each task deletes code and the references to it,
and is proven by the existing gates: `test_cli_tools_overview.py` (the CLI
overview matches `[project.scripts]`), `test_cli_entry_points.py`, the strict
mkdocs build, and a search for the removed names.

**Tech stack:** Python (vergil-tooling), Markdown skills and shell hooks
(vergil-claude-plugin), MkDocs sites.

**Spec:** `epics/910-remove-vestigial-clis/spec.md`.

## Global constraints

- Removed commands: `vrg-adhoc-migrate`, `vrg-wait-until-green`,
  `vrg-changelog`, `vrg-container-test`, `vrg-resolve-tooling-version`,
  `vrg-pr-fix-body`, `vrg-epic-unlink`. The `vrg-repo-profile` entry point is
  removed. Its check stays, with the module renamed to `bin/repo_profile.py`. The `migrate-repo` skill is removed.
- Kept: `vrg-reword`, `vrg-github-repo-init`, `vrg-adhoc-epic`,
  `vrg-ensure-label`, `vrg-audit-approve`, the five sync tools, and
  `vrg-generate-commands`.
- Shared code stays: `github.wait_for_checks`, `lib/changelog.py`,
  `lib/pr_body.py`, `lib/linkage.py`, and `detect_language` (called by
  `resolve_language`).
- Tests are not callers: library code that only tests use after a removal is
  deleted, together with those tests (spec §4 item 3).
- History is never edited: `releases/`, `CHANGELOG.md`, and `epics/`,
  `docs/specs/` and `docs/plans/` documents.
- Recorded fixtures that contain a command name (`tests/vergil_tooling/fixtures/host/uv-tool-list.txt`,
  `tests/vergil_tooling/fixtures/update_deps/vrg-validate-typecheck-failure.stdout.txt`)
  change only if a test needs them to.
- Validation is `vrg-container-run -- vrg-validate` only.

## Review focus

1. **A runtime import of a removed module that the tests do not reach.**
   Expected: no `src/` file imports a deleted module. Task 1 step 6 searches
   for each deleted module name across `src/`.
2. **A skill or hook still telling an agent to run a removed command.**
   Expected: agents are never told to run a command that is not installed.
   Task 3's final search covers `skills/`, `hooks/`, `commands/` and `agents/`.
3. **Broken links to deleted pages or anchors**, for example
   `cli-tools-overview.md#vrg-container-test` from `reference/index.md`.
   Expected: the strict mkdocs build in `vrg-validate` passes. Tasks 1 and 2
   remove the links along with the anchors.
4. **Stale shims on hosts and VMs after upgrade.** Expected: `uv tool install`
   of the new release removes the old console scripts. Nothing to change, and
   the PR notes say so.
5. **A remaining doc that still names `vrg-repo-profile` as a command to
   run.** Expected: the docs describe it as a `vrg-validate` check. Task 2's
   final search covers `docs/site/`.

---

### Task 1: vergil-tooling: remove seven unused commands

Repo: vergil-project/vergil-tooling.

**Files:**

- Modify: `pyproject.toml` (`[project.scripts]`: remove the seven entries)
- Delete: `src/vergil_tooling/bin/vrg_adhoc_migrate.py`,
  `vrg_wait_until_green.py`, `vrg_changelog.py`, `vrg_container_test.py`,
  `vrg_resolve_tooling_version.py`, `vrg_pr_fix_body.py`, `vrg_epic_unlink.py`
- Delete: `src/vergil_tooling/lib/adhoc_migrate.py`
- Delete: `tests/vergil_tooling/test_vrg_adhoc_migrate.py`,
  `test_vrg_wait_until_green.py`, `test_vrg_changelog.py`,
  `test_vrg_container_test.py`, `test_vrg_resolve_tooling_version.py`,
  `test_vrg_pr_fix_body.py`, `test_vrg_epic_unlink.py`, and any test file
  for `lib/adhoc_migrate.py`
- Modify: `src/vergil_tooling/lib/container.py` (remove
  `_DEFAULT_TEST_COMMANDS` at line 48 and the comment at line 61)
- Modify: `tests/vergil_tooling/test_container.py` (remove the tests of
  `_DEFAULT_TEST_COMMANDS`, around lines 12 and 368-385)
- Modify (comments and docstrings only):
  `src/vergil_tooling/lib/github.py:1567`, `lib/pr_body.py:4`,
  `lib/linkage.py:13`, `data/host_dependencies.toml:142`,
  `bin/vrg_epic_create.py:10`, `bin/vrg_adhoc_epic.py:6`,
  `tests/vergil_tooling/test_github_stale_checks.py:551`
- Modify: `docs/site/docs/reference/cli-tools-overview.md` (remove the seven
  `### vrg-…` entries, the `migrate-repo` mention at line 516, and the
  `vrg-container-test` mention at line 2003)
- Modify: `docs/site/docs/reference/index.md` (remove the rows for removed
  commands, for example line 31)
- Modify: `docs/site/docs/reference/dev/finalize-pr.md:118`
- Delete: `docs/site/docs/reference/dev/pr-fix-body.md`. Modify:
  `docs/site/mkdocs.yml:85` (its nav entry)

**Interfaces:**

- Consumes: nothing.
- Produces: `pyproject.toml` and `cli-tools-overview.md` without the seven
  commands. Task 2 edits both files after this task merges.

This task only deletes code, so it has no red/green/refactor cycle. The
existing gates are its tests: `test_cli_tools_overview.py`,
`test_cli_entry_points.py`, the strict mkdocs build, and the step 6 search.

- [ ] **Step 1: Record the baseline.** On the untouched branch, run
  `vrg-container-run -- vrg-validate` and note the pytest passed count from its
  log in `.vergil/`. Then time the test stage with the repo's harness,
  `vrg-container-run -- uv run python scripts/perf/measure_test_stage.py --emit-evidence --runs 5 --out build/perf/before.md`
  (`build/` is ignored, so nothing is committed), and note the median of the
  configuration that matches the current `vrg-validate` test gate. This only
  measures time; `vrg-validate` remains the validation.
- [ ] **Step 2: Remove the seven `[project.scripts]` entries** and their
  `cli-tools-overview.md` entries together, so `test_cli_tools_overview.py`
  stays green.
- [ ] **Step 3: Delete the seven `bin/` modules, `lib/adhoc_migrate.py` and
  their test files.**
- [ ] **Step 4: Remove `_DEFAULT_TEST_COMMANDS`** and its tests in
  `test_container.py`.
- [ ] **Step 5: Edit the comments, docstrings and docs pages** listed under
  Files, so none names a removed command as a live tool. Delete
  `pr-fix-body.md` and its nav entry.
- [ ] **Step 6: Search for leftovers.** Each of these must return no match
  in `src/`, `tests/` (except the two recorded fixtures), `docs/site/` and
  `pyproject.toml`:

  ```bash
  grep -rnE 'vrg-(adhoc-migrate|wait-until-green|changelog|container-test|resolve-tooling-version|pr-fix-body|epic-unlink)\b|vrg_(adhoc_migrate|wait_until_green|changelog|container_test|resolve_tooling_version|pr_fix_body|epic_unlink)\b|lib import adhoc_migrate|lib\.adhoc_migrate|_DEFAULT_TEST_COMMANDS|migrate-repo' src tests docs/site pyproject.toml
  ```

  Then, for each `lib/` function the deleted modules called (for example
  `github.edit_pr_body`, `github.update_branch` and `github.merge_status`),
  confirm that a remaining `src/` file still calls it. Delete any that has
  only test callers, together with those tests.
- [ ] **Step 7: Validate.** `vrg-container-run -- vrg-validate` passes. Note
  the new passed count, then rerun the step 1 harness command with
  `--out build/perf/after.md` and note the same configuration's new median.
- [ ] **Step 8: Commit** with `vrg-commit --type refactor --scope cli
  --message "remove unused vrg commands"`. The PR notes give the before and
  after numbers from steps 1 and 7.

### Task 2: vergil-tooling: drop the standalone `vrg-repo-profile` command

Repo: vergil-project/vergil-tooling. Blocked by Task 1, which edits the same
two files.

**Files:**

- Modify: `pyproject.toml` (remove `vrg-repo-profile`)
- Modify: `docs/site/docs/reference/cli-tools-overview.md` (remove
  `### vrg-repo-profile`, and reword the "Exit code contract" line that says
  exit 2 is "used by `vrg-repo-profile`" to name the `vrg-validate`
  repo-profile check)
- Modify: `docs/site/docs/reference/lint/repo-profile.md` (drop the
  "Installed as" line and the usage block; describe the check as a step of
  `vrg-validate`)
- Modify: `docs/site/docs/getting-started.md`, `index.md`,
  `guides/ci-architecture.md`, `guides/consuming-repo-setup.md`,
  `guides/validation-matrix.md`, `reference/index.md`,
  `standards/repository-structure.md`
- Rename: `src/vergil_tooling/bin/vrg_repo_profile.py` →
  `src/vergil_tooling/bin/repo_profile.py`. The standard
  (`docs/site/docs/standards/development/python/cli-entry-points.md`) says a
  `bin/` module that is not a console script carries no command prefix, as
  `validate_common.py` already does. Its argparse `prog` (line 48) becomes
  `"repo-profile"`, so it no longer names a removed command.
- Rename: `tests/vergil_tooling/test_vrg_repo_profile.py` →
  `tests/vergil_tooling/test_repo_profile.py`
- Modify: `src/vergil_tooling/bin/validate_common.py` (the import at line 32
  and the call at line 44 use `repo_profile`)
- Modify: `tests/vergil_tooling/test_validate_common.py` (every
  `vergil_tooling.bin.validate_common.vrg_repo_profile.main` patch target
  becomes `vergil_tooling.bin.validate_common.repo_profile.main`)

**Interfaces:**

- Consumes: Task 1's `pyproject.toml` and `cli-tools-overview.md`.
- Produces: nothing later tasks use.

- [ ] **Step 1 (red): Point the tests at the new name first.** Rename
  `test_vrg_repo_profile.py` to `test_repo_profile.py` and change its import
  to `vergil_tooling.bin.repo_profile`. Change every
  `vergil_tooling.bin.validate_common.vrg_repo_profile.main` patch target in
  `test_validate_common.py` to `vergil_tooling.bin.validate_common.repo_profile.main`.
  `vrg-container-run -- vrg-validate` fails at the test stage, because
  `vergil_tooling.bin.repo_profile` does not exist yet.
- [ ] **Step 1a (green): Rename the module** to `bin/repo_profile.py`, update
  the import and call in `validate_common.py`, set `prog` to `"repo-profile"`,
  and remove the `[project.scripts]` entry and its overview entry together.
  The check's behaviour and its tests' assertions are unchanged.
- [ ] **Step 2: Rewrite `reference/lint/repo-profile.md`** as the reference
  for a `vrg-validate` check. Keep its per-key `vergil.toml` rules and its
  path, so existing links still resolve.
- [ ] **Step 3: Reword the seven guide pages** so each one presents the check
  as part of `vrg-validate`, not as a command to run.
- [ ] **Step 4: Search.** `grep -rn 'vrg_repo_profile' src tests` returns
  nothing outside the recorded fixtures under `tests/vergil_tooling/fixtures/`,
  and `grep -rn 'vrg-repo-profile' docs/site pyproject.toml src`
  returns only text that names the check, never an instruction to run the
  command.
- [ ] **Step 5: Validate.** `vrg-container-run -- vrg-validate` passes,
  including `test_cli_entry_points.py`, `test_validate_common.py` and the
  strict mkdocs build.
- [ ] **Step 6: Commit** with `vrg-commit --type refactor --scope cli
  --message "drop the standalone vrg-repo-profile command; vrg-validate runs the check"`.

### Task 3: vergil-claude-plugin: remove `migrate-repo` and the stale host tool

Repo: vergil-project/vergil-claude-plugin. Independent of Tasks 1 and 2.

**Files:**

- Delete: `skills/migrate-repo/` (holds only `SKILL.md`)
- Modify: `skills/epic-create/SKILL.md:604-606` (the Notes bullet: drop the
  `migrate-repo` sentence, and keep the `intake-capture` and `intake-review`
  pointers)
- Modify: `README.md:264` (remove the skill's table row)
- Modify: `docs/site/docs/skills/index.md` (the table row at line 68 and the
  `## migrate-repo` section from line 302)
- Modify: `hooks/scripts/lib/host-container-tools.sh` (remove
  `vrg-wait-until-green` from `HOST_TOOLS`)

**Interfaces:** none.

- [ ] **Step 1: Delete the skill directory** and edit the four files.
- [ ] **Step 2: Search.** `grep -rnE 'migrate-repo|vrg-wait-until-green|vrg-(adhoc-migrate|changelog|container-test|resolve-tooling-version|pr-fix-body|epic-unlink)\b' skills hooks commands agents docs/site README.md`
  returns nothing.
- [ ] **Step 3: Validate.** `vrg-container-run -- vrg-validate` passes.
- [ ] **Step 4: Commit** with `vrg-commit --type refactor --scope skills
  --message "remove the migrate-repo skill and a stale host tool entry"`.

### Task 4: docs: update the tools-and-skills page

Repo: vergil-project/docs. Independent of the other tasks.

**Files:**

- Modify: `docs/site/docs/management/tools-and-skills.md`: line 44 (drop
  `vrg-epic-unlink` from the epic-link line), line 88 (remove the
  `vrg-adhoc-migrate` entry), line 147 (remove the `migrate-repo` entry)

**Interfaces:** none.

- [ ] **Step 1: Edit the three places.**
- [ ] **Step 2: Search.** `grep -rnE 'migrate-repo|vrg-(adhoc-migrate|wait-until-green|changelog|container-test|resolve-tooling-version|pr-fix-body|epic-unlink)\b|vrg-repo-profile' docs/site`
  returns nothing, apart from `vrg-repo-profile` text that describes the
  check rather than a command.
- [ ] **Step 3: Validate.** `vrg-container-run -- vrg-validate` passes.
- [ ] **Step 4: Commit** with `vrg-commit --type docs --scope management
  --message "drop removed tools and the migrate-repo skill"`.

## After the tasks

The documentation-review bookend (vergil-project/vergil-tooling#3947) runs
the spec §6 search across the fleet. The retrospective
(vergil-project/.github#912) reports Task 1's before and after test numbers.
