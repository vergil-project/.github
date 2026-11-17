# Narrow vergil-tooling to Python 3.15 — implementation plan

- **Epic:** vergil-project/.github#1256
- **Planning issue:** vergil-project/.github#958 (review history is in its
  comments)
- **Status:** approved

> **For agentic workers:** each task is one issue and one PR in the repo named
> in its heading, worked with `issue-implement`; the release task is worked
> with `issue-deploy`.

**Goal:** vergil-tooling requires Python 3.15, and every place it is installed
or run uses 3.15.

**Spec:** [`spec.md`](spec.md) in this directory.

## Constraints

- The interpreter is the literal `3.15`, passed as `--python 3.15`.
- No install path changes mechanism; each one only names its interpreter.
- Validation runs through `vrg-container-run -- vrg-validate` in the task
  repo's worktree. Releases are human-gated.

## Tasks

| Task | Repo | Kind | What | Blocked by |
| --- | --- | --- | --- | --- |
| 1 | .github | task (#1258) | Spec and plan (this PR) | — |
| 2 | vergil-actions | task | `install-tooling.sh` names 3.15 | 1 |
| 3 | vergil-actions | deployment | vergil-actions release, `v2.1` moved | 2 |
| 4 | vergil-tooling | task | `container_cache.py`, `vm_packages.dev_install` and docs name 3.15 | 1 |
| 5 | vergil-vm | task | `vrg-vm-init.sh` and `test_vergil.sh` name 3.15 | 1 |
| 6 | vergil-tooling | task | Raise the floor to 3.15 | 3, 4, 5 |
| — | vergil-tooling | bookend (#4578) | Documentation review | 2–6 |
| — | .github | bookend (#1259) | Retrospective | all |

### Task 2: `install-tooling.sh` names 3.15 (vergil-actions)

- In `actions/shared/setup/vergil/install-tooling.sh`:
  - `self_repo_install`: on a host (`is_host`), `uv sync --frozen --group dev
    --python 3.15` without `UV_PYTHON_PREFERENCE=only-system`; in a container,
    unchanged.
  - `uv_git_install` and `git_source_install`: `uv tool install --python 3.15`.
- Update the script's header comment and `action.yml`'s description.
- Test: in `tests/install-tooling.test.sh`, the host self-repo case expects
  `--python 3.15` and no `only-system`. The container self-repo case is
  unchanged. The `neither`, dev-pin and `install-source: git` cases expect
  `--python 3.15`.

### Task 3: vergil-actions release (deployment)

- **Precondition (human-attested):** a vergil-actions release containing
  Task 2 is published and `v2.1` points at it.
- **Check:** `v2.1`'s `install-tooling.sh` contains `--python 3.15`, and a
  vergil-tooling PR's CI run after the move is green.

### Task 4: `container_cache.py`, `vm_packages.dev_install` and docs name 3.15 (vergil-tooling)

- `src/vergil_tooling/lib/container_cache.py`: the `neither` install in
  `_probe_lines` becomes `uv tool install --quiet --python 3.15 …`.
- `src/vergil_tooling/lib/vm_packages.py`: `_uv_tool_install`'s install and
  retry commands carry `--python 3.15`.
- Docs: add `--python 3.15` to the dev-tree `uv sync --group dev` in
  `docs/site/docs/guides/consuming-repo-setup.md`, `CLAUDE.md` and
  `README.md`, and to the developer-host install in
  `docs/site/docs/guides/releasing.md`.
- Tests: `tests/vergil_tooling/test_container_cache.py` (the `neither`
  install line) and `tests/vergil_tooling/test_vm_packages.py` (both
  commands) expect `--python 3.15`.

### Task 5: `vrg-vm-init.sh` and `test_vergil.sh` name 3.15 (vergil-vm)

- `scripts/vrg-vm-init.sh` and `tests/test_vergil.sh`: `uv tool install
  --python 3.15 …`.
- `tests/check-vm-init-series.sh`: the asserted install command includes
  `--python 3.15`.

### Task 6: Raise the floor to 3.15 (vergil-tooling)

- `pyproject.toml`: `requires-python = ">=3.15"`, ruff `target-version =
  "py315"`, mypy `python_version = "3.15"`. Re-lock with `vrg-container-run
  -- uv lock`.
- Apply `vrg-container-run -- ruff check --fix --select UP .` and review the
  rewrites.
- Check: `vrg-container-run -- vrg-validate` passes with one series under
  `--all-versions`, and the PR's CI shows a single `unit / 3.15` leg.
- Release is the human's normal vergil-tooling release; its consumer-refresh
  already installs with `--python 3.15`.
