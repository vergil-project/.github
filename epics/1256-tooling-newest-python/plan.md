# Narrow vergil-tooling to the newest Python series — implementation plan

- **Epic:** vergil-project/.github#1256
- **Planning issue:** vergil-project/.github#958 (review history is in its
  comments)
- **Status:** draft, in design review

> **For agentic workers:** each task is one issue and one PR in the repo named
> in its heading, worked with `issue-implement`. Operational tasks are worked
> with `issue-deploy` (deployment) or `issue-validate` (validation). Steps use
> checkbox (`- [ ]`) syntax for tracking.

**Goal:** vergil-tooling declares `requires-python = ">=3.15"`, so its CI
tests one series today, and every install path that could otherwise land it on
an untested interpreter names that interpreter explicitly.

**Architecture:** the install paths are fixed first and released: vergil-actions
`install-tooling.sh` (host self-repo, host dev pin, index rebuild, container
`neither`), vergil-tooling `container_cache.py` and `vm_packages.dev_install`,
and vergil-vm's VM initialisation. Where no core Python is present, they install
the newest `vergil-python<x.y>` the vergil apt repository serves. A live check
then runs the rewritten host paths. Only after that does one vergil-tooling PR
raise the floor, add the drift guard and document the floor. The release that
carries it is followed by fixture runs on the Python fixtures.

**Tech Stack:** bash and its framework-free test harness (vergil-actions),
Python 3 and pytest (vergil-tooling), bash and the VM test scripts (vergil-vm),
apt, uv.

**Spec:** [`spec.md`](spec.md) in this directory. Executors read both.

## Global Constraints

- The floor is `requires-python = ">=3.15"`, with ruff `target-version = "py315"` and mypy `python_version = "3.15"` (spec §6.1).
- The tested series is every window series at or above the floor; the resolver's rule is unchanged (spec §3).
- No new definition of the core series. The package's `derived_runtime` and the images' `manifest.yml` `core.python` stay as they are (spec §2).
- Where no core Python is present, vergil-tooling runs on the newest packaged series: the highest `vergil-python<x.y>` the vergil apt repository serves, passed as `/opt/vergil/python/<x.y>/bin/python3` (spec §6.2).
- `self_repo_install` keeps `UV_PYTHON_PREFERENCE=only-system` (issue #1002) (spec §6.2).
- The index rebuild (`install-source: git`) is the one uv-managed interpreter: the floor series at the installed tag, read by a `git` fetch of `pyproject.toml`; only a bare `>=X.Y` is accepted (spec §6.2).
- Every install that names an interpreter checks afterwards that the installed tool's, or the synced venv's, interpreter is that series (spec §6.2).
- The drift guard compares documented `--python` literals with the floor, never with a derived core series (spec §6.3).
- No silent failures: no swallowed errors, no fallback to uv's implicit discovery (spec §2; repo engineering principles).
- Validation runs only through `vrg-container-run -- vrg-validate` in the task repo's worktree. Commit with `vrg-commit`; hand off with `vrg-pr-workflow report-ready`. Releases are human-gated; no agent cuts one.

## Review Focus

1. **The apt repository serves a `vergil-python` name that is not a plain series** (for example a `-dbg` or `-dev` package, or `vergil-python3.9` next to `3.15`). The newest-series selection must parse only `vergil-python<digits>.<digits>`, compare numerically (3.15 > 3.9), and fail when none matches. Pinned in Task 3a (`case_newest_packaged_python_numeric`, `case_newest_packaged_python_none`) and Task 5b (`test_newest_runtime_numeric_and_filtered`).
2. **A caller workflow exports `UV_PYTHON` for an older family series on the host self-repo job.** The explicit `--python` must win, and the venv check must catch it if it does not. Pinned in Task 3a (`case_host_self_repo_ignores_uv_python`).
3. **The index rebuild runs at an old tag whose `requires-python` is `>=3.12`, or one with a compound specifier.** The first installs on a uv-managed 3.12; the second fails loudly naming the specifier. Pinned in Task 3b.
4. **A VM dev install on a VM that already has the newest `vergil-python`.** It reinstalls nothing it does not need and still passes the explicit path. A later packaged install's `_purge_obsolete_runtimes` must not purge the runtime the packaged tool depends on. Pinned in Task 5b (`test_dev_install_runtime_already_present`).
5. **A container image with neither the vergil apt source nor a core Python** (a non-Vergil image under `vrg-container-run`, or a CI container job), if Task 2 finds no live caller. Both copies refuse with the same actionable message and install nothing. Pinned in Tasks 3c and 5a.

---

## Delivery table

| Task | Repo | Kind | What | Blocked by |
| --- | --- | --- | --- | --- |
| 1 | .github | task (#1258) | Spec and plan (this PR) | — |
| 2 | .github | task | Audit: confirm spec §5 and record the two open answers in this plan | 1 |
| 3a | vergil-actions | task | Host self-repo and host dev-pin installs on the newest packaged `vergil-python` | 2 |
| 3b | vergil-actions | task | Index rebuild on an explicit uv-managed floor interpreter | 2 |
| 3c | vergil-actions | task | Container `neither` path: refusal or packaged-Python rule, per Task 2 | 2 |
| 4 | vergil-actions | deployment | vergil-actions release with 3a–3c attested (`v2.1` moved) | 3a, 3b, 3c |
| 5a | vergil-tooling | task | `container_cache.py` `neither` path, matching 3c | 2 |
| 5b | vergil-tooling | task | `vm_packages.dev_install` on the newest packaged `vergil-python` | 2 |
| 6 | vergil-vm | task | VM initialisation stops writing its own install; tests and docs follow | 2 |
| 7 | vergil-tooling | deployment | vergil-tooling release with 5a and 5b attested and installed; 6 merged | 5a, 5b, 6 |
| 8 | vergil-actions | validation | Live run of the host dev-pin and index-rebuild paths | 4 |
| 9 | vergil-tooling | task | Floor raise, drift guard and floor documentation | 4, 7, 8 |
| 10 | vergil-tooling | deployment | vergil-tooling release with the floor raise attested and installed | 9 |
| 11 | fixtures | fixture runs | `python-lib`, `python-app` against the Task 10 release | 10 |
| — | vergil-tooling | bookend (#4578) | Documentation review | 3a–9 |
| — | .github | bookend (#1259) | Retrospective | all |

3a, 3b and 3c all change `install-tooling.sh` and its test harness, so they
land in that order to avoid conflicts, though none needs another's code. 5a and
3c must make the same choice (Task 2's answer) and use the same refusal text.

---

### Task 2: Audit — confirm spec §5 and record the open answers (.github)

**Files:**

- Modify: `epics/1256-tooling-newest-python/plan.md` (a new `## Audit results` section after the delivery table)

**Interfaces:**

- Produces, for Tasks 3c, 5a and 6, two recorded answers:
  - **A1, live container `neither` callers:** `none`, or the list of callers (repo, workflow or command, image).
  - **A2, standalone use of `vrg-vm-init.sh`:** `none` (only `vrg-vm create` reaches its install), or the callers.

- [ ] **Step 1: Re-run the spec §5 audit** against `origin/develop` of vergil-tooling, vergil-actions, vergil-vm and vergil-containers. Search for every `uv tool install`, `uv sync`, `--python` and `requires-python` that concerns vergil-tooling. Confirm each of rows 1–15 and list any place they miss.
- [ ] **Step 2: Answer A1.** List every container job and every `vrg-container-run` image that reaches `install-tooling.sh` `container_install`/`dev_install` or `container_cache._probe_lines` without both the vergil apt source and a core Python. Sources: the images vergil-containers publishes (all carry both since #649), the reusable workflows' `container:` lines, and any image a consumer can name in `vergil.toml`.
- [ ] **Step 3: Answer A2.** List every caller of `scripts/vrg-vm-init.sh` (vergil-vm scripts and docs, vergil-tooling `src/`, and the `vrg-vm` commands' help text).
- [ ] **Step 4: Write `## Audit results`:** one line per §5 row (`confirmed`, or the correction), any new row, then A1 and A2 with their evidence (paths and line numbers). If a new row needs a fix the plan does not have, add a task for it and a delivery-table line.
- [ ] **Step 5: Run** `vrg-container-run -- vrg-validate`; expect PASS.
- [ ] **Step 6: Commit** `vrg-commit --type docs --scope epic-1256 --message "record the install-path audit for narrowing vergil-tooling's Python"`, then report-ready.

### Task 3a: Host self-repo and host dev-pin installs on the newest packaged `vergil-python` (vergil-actions)

**Files:**

- Modify: `actions/shared/setup/vergil/install-tooling.sh` (`self_repo_install`, `dev_install`, new functions below, header comment)
- Modify: `actions/shared/setup/vergil/action.yml` (the description's install-path paragraph)
- Test: `actions/shared/setup/vergil/tests/install-tooling.test.sh`

**Interfaces:**

- Produces, in `install-tooling.sh`:
  - `newest_packaged_python`: after `apt_get update`, reads `apt-cache pkgnames vergil-python`, keeps names matching `^vergil-python[0-9]+\.[0-9]+$`, sorts by version (`sort -V`) and prints the highest `x.y`. Fails with `die` when nothing matches.
  - `host_packaged_python`: `bootstrap_vergil_repository`, `apt_get update`, `newest_packaged_python`, then `apt_get install -y vergil-python<x.y>`. Prints `/opt/vergil/python/<x.y>/bin/python3` and fails when that file is not executable afterwards.
  - `check_series <python3-path> <x.y>`: runs the interpreter's version probe and fails with `die` naming both when they differ.
- `self_repo_install`: when `is_host`, gets the path from `host_packaged_python` and runs `UV_PYTHON_PREFERENCE=only-system uv sync --frozen --group dev --python "<path>"`, then `check_series "${GITHUB_WORKSPACE}/.venv/bin/python" <x.y>`. In a container it is unchanged.
- `dev_install`: when `is_host` (checked before `environment_state`), runs `uv tool install --python "<path>"` from `host_packaged_python`, then `check_series` on the tool's interpreter (`$(uv tool dir)/vergil-tooling/bin/python`). In a container it is unchanged.
- The bootstrap source is removed afterwards, as `host_install` already does.

- [ ] **Step 1: Write the failing cases** in `install-tooling.test.sh`, using its `new_case`/`write_stub`/`run_script` seams with `apt-cache`, `apt-get` and the interpreter stubbed:
  - `case_newest_packaged_python_numeric`: the stub serves `vergil-python3.9`, `vergil-python3.14`, `vergil-python3.15`, `vergil-python3.15-dbg`. `3.15` is chosen and installed.
  - `case_newest_packaged_python_none`: no matching name → non-zero exit; stderr names `vergil-python`; nothing installed.
  - `case_host_self_repo_packaged_python`: a host self-repo run bootstraps the repository and installs `vergil-python3.15`. `uv sync` is called with `--python /opt/vergil/python/3.15/bin/python3` and `UV_PYTHON_PREFERENCE=only-system`, and no bootstrap file is left.
  - `case_host_self_repo_ignores_uv_python`: `UV_PYTHON=3.12` exported; the `uv sync` call still carries the explicit `--python`. A venv stub reporting 3.12 fails the job with `check_series`'s message.
  - `case_host_dev_pin_packaged_python`: a dev tag on a host → `uv tool install --python /opt/vergil/python/3.15/bin/python3 "vergil-tooling @ git+…@<tag>"`; `environment_state` is never consulted.
  - `case_container_self_repo_unchanged` and `case_container_dev_unchanged`: the container calls are byte-for-byte today's.
- [ ] **Step 2: Run** `vrg-container-run -- vrg-validate`; expect the new cases to fail.
- [ ] **Step 3: Implement** the three functions and the two host branches. Update the script's header comment and `action.yml`'s description, which are the paths' reference text.
- [ ] **Step 4: Run** `vrg-container-run -- vrg-validate`; expect PASS.
- [ ] **Step 5: Commit** `vrg-commit --type feat --scope setup-vergil --message "install vergil-tooling on the newest packaged vergil-python on host runners"`, then report-ready.

### Task 3b: Index rebuild on an explicit uv-managed floor interpreter (vergil-actions)

**Files:**

- Modify: `actions/shared/setup/vergil/install-tooling.sh` (`git_source_install`, new `read_floor`)
- Test: `actions/shared/setup/vergil/tests/install-tooling.test.sh`

**Interfaces:**

- Consumes: Task 3a's `check_series`.
- Produces: `read_floor <ref>`: fetches the ref into a temporary repository (`git init -q`, `git fetch --depth 1 "$GIT_URL" "refs/tags/<ref>"`) and prints `X.Y` from `git show FETCH_HEAD:pyproject.toml`'s `requires-python`. It accepts only a line matching `^requires-python = ">=([0-9]+\.[0-9]+)"$`, and fails with `die` naming the value on anything else or on a failed fetch. The temporary repository is removed on every path.
- `git_source_install`: after resolving `ref`, runs `uv python install "<floor>"` and `uv tool install --python "<floor>" --python-preference only-managed "vergil-tooling @ git+…@<ref>"`, then `check_series` on the tool's interpreter.

- [ ] **Step 1: Write the failing cases:**
  - `case_git_source_floor_315`: the stubbed fetch serves `requires-python = ">=3.15"` → the install call carries `--python 3.15 --python-preference only-managed`.
  - `case_git_source_old_tag_312`: `>=3.12` → `--python 3.12`.
  - `case_git_source_compound_specifier`: `>=3.15,<3.17` → non-zero exit, stderr names the value, nothing installed.
  - `case_git_source_fetch_failure`: the fetch fails → non-zero exit naming the ref; nothing installed; no temporary directory left.
  - `case_git_source_line_pin_reads_resolved_tag`: a line pin reads the floor at the `newest_release` tag, not at `vX.Y`.
- [ ] **Step 2: Run** `vrg-container-run -- vrg-validate`; expect the new cases to fail.
- [ ] **Step 3: Implement** `read_floor` and the `git_source_install` change; update the header comment's `install-source: git` paragraph.
- [ ] **Step 4: Run** `vrg-container-run -- vrg-validate`; expect PASS.
- [ ] **Step 5: Commit** `vrg-commit --type feat --scope setup-vergil --message "install the index-rebuild tooling on an explicit uv-managed floor interpreter"`, then report-ready.

### Task 3c: Container `neither` path, per Task 2's A1 (vergil-actions)

**Files:**

- Modify: `actions/shared/setup/vergil/install-tooling.sh` (`dev_install`, `container_install`)
- Test: `actions/shared/setup/vergil/tests/install-tooling.test.sh` (`case_container_uv` and its dev counterpart)

**Interfaces:**

- Consumes: Task 2's A1; Task 3a's `host_packaged_python` path logic, if A1 lists callers.
- Produces: the refusal text, shared verbatim with Task 5a: `vergil-tooling: the image has neither the vergil apt source (/etc/apt/sources.list.d/vergil.sources) nor a core Python (/opt/vergil/python/*/bin/python3); build it on a Vergil image`.

- [ ] **Step 1: Write the failing cases.** If A1 is `none`: `case_container_neither_refused` and `case_container_dev_neither_refused` expect a non-zero exit, the refusal text, and no `uv` call; `case_container_uv` is removed. If A1 lists callers: the same two cases expect the image to get its interpreter by the packaged-Python rule instead (the image has no source, so the bootstrap runs in the container), and `check_series`.
- [ ] **Step 2: Run** `vrg-container-run -- vrg-validate`; expect the new cases to fail.
- [ ] **Step 3: Implement** the chosen branch in both functions; `uv_git_install` is removed if nothing calls it any more.
- [ ] **Step 4: Run** `vrg-container-run -- vrg-validate`; expect PASS.
- [ ] **Step 5: Commit** `vrg-commit --type feat --scope setup-vergil --message "stop installing vergil-tooling on an image's implicit Python"`, then report-ready.

### Task 4: vergil-actions release with 3a–3c (deployment, vergil-actions)

Blocked by 3a, 3b, 3c.

- **Precondition (human-attested):** a vergil-actions release containing the PRs of 3a, 3b and 3c is published and the floating `v2.1` tag points at it. The version is named in a comment.
- **Procedure (agent-safe):**
  1. Confirm the release notes list the three PRs.
  2. `vrg-git ls-remote` vergil-actions: `refs/tags/v2.1` points at the same commit as the release's exact tag.
  3. The released `actions/shared/setup/vergil/install-tooling.sh` at `v2.1` defines `newest_packaged_python` and `read_floor`.
  4. The most recent vergil-tooling `pull_request` CI run after the move has a green `matrix` job whose log shows the host self-repo install on `/opt/vergil/python/3.15/bin/python3`.
- **SUCCESS:** every check passes, recorded with the version and the CI run URL.

### Task 5a: `container_cache.py` `neither` path, matching 3c (vergil-tooling)

**Files:**

- Modify: `src/vergil_tooling/lib/container_cache.py` (`_probe_lines`, `_UV_FROM_GIT`, the module docstring's install-path list around line 865)
- Test: `tests/vergil_tooling/test_container_cache.py`

**Interfaces:**

- Consumes: Task 2's A1; Task 3c's refusal text, verbatim.

- [ ] **Step 1: Write the failing tests** through the module's `$VRG_CACHE_ROOT` seam. If A1 is `none`: `test_probe_neither_refused` (exit 1, the 3c refusal text, no `uv` call); the test that expects `_TODAYS_INSTALL` is removed. If A1 lists callers: `test_probe_neither_packaged_python`, which expects the packaged-Python rule.
- [ ] **Step 2: Run** `vrg-container-run -- vrg-validate`; expect the new tests to fail.
- [ ] **Step 3: Implement** the branch in `_probe_lines`; remove `_UV_FROM_GIT` if unused; update the docstring.
- [ ] **Step 4: Run** `vrg-container-run -- vrg-validate`; expect PASS.
- [ ] **Step 5: Commit** `vrg-commit --type feat --scope container-cache --message "refuse an image with neither the vergil apt source nor a core Python"` (or the packaged-Python wording), then report-ready.

### Task 5b: `vm_packages.dev_install` on the newest packaged `vergil-python` (vergil-tooling)

**Files:**

- Modify: `src/vergil_tooling/lib/vm_packages.py` (`dev_install`, `_uv_tool_install`, new helpers, module docstring)
- Test: `tests/vergil_tooling/test_vm_packages.py`

**Interfaces:**

- Produces, in `vergil_tooling.lib.vm_packages`:
  - `_newest_runtime(transport: Transport) -> str`: runs `apt-cache pkgnames vergil-python` and returns the highest `x.y` among names matching `^vergil-python(\d+)\.(\d+)$`, compared as integers. Calls `_die` when none match.
  - `_install_runtime(transport: Transport, series: str) -> str`: `sudo apt-get install -y vergil-python<series>` (through `repo_setup.apt_get`), returning `/opt/vergil/python/<series>/bin/python3`. Calls `_die` if the path does not run afterwards.
  - `_uv_tool_install(transport, install_spec, *, reinstall: bool, python: str)`: both the install and the retry commands carry `--python <python>`.
  - `dev_install(transport, ref)`: `repo_setup.bootstrap(transport.run, org, "deb", codename, sudo=True)` as `packaged_install` does, then `apt-get update`, `_newest_runtime`, `_install_runtime`, then `_uv_tool_install(..., python=path)`, then a series check of `~/.local/share/uv/tools/vergil-tooling/bin/python`. The dev-ref file is written last, as today.

- [ ] **Step 1: Write the failing tests** with the module's fake transport:
  - `test_newest_runtime_numeric_and_filtered`: names `vergil-python3.9`, `vergil-python3.15`, `vergil-python3.15-dbg`, `vergil-python3.14` → `"3.15"`; none → `SystemExit` naming `vergil-python`.
  - `test_dev_install_fresh_vm`: no runtime installed → bootstrap, update, `install -y vergil-python3.15`, then `uv tool install --reinstall --python /opt/vergil/python/3.15/bin/python3 "vergil-tooling @ git+…@feature-x"`, in that order; the dev-ref file is written last.
  - `test_dev_install_retry_keeps_python`: the first install fails → the `--force --reinstall` retry also carries `--python`.
  - `test_dev_install_runtime_already_present`: the runtime is installed already → the install is a no-op for apt (`install -y` of an installed package) and the path is still passed.
  - `test_dev_install_series_mismatch`: the tool's interpreter reports 3.14 → `SystemExit` naming both series; the dev-ref file is not written.
  - `test_dev_install_bootstrap_failure`: `repo_setup.bootstrap` raises `PackageError` → `SystemExit` naming the repository URL, nothing installed.
- [ ] **Step 2: Run** `vrg-container-run -- vrg-validate`; expect the new tests to fail.
- [ ] **Step 3: Implement** the helpers and `dev_install`. Update the module docstring's dev-install paragraph.
- [ ] **Step 4: Run** `vrg-container-run -- vrg-validate`; expect PASS.
- [ ] **Step 5: Commit** `vrg-commit --type feat --scope vm-packages --message "install VM dev refs on the newest packaged vergil-python"`, then report-ready.

### Task 6: VM initialisation stops writing its own install (vergil-vm)

**Files:**

- Modify: `scripts/vrg-vm-init.sh` (the `# --- Install vergil-tooling ---` block, lines 119–125)
- Modify: `tests/test_vergil.sh`, `tests/check-vm-init-series.sh`
- Modify: `docs/site/docs/operations/build-and-test.md` (the `test_vergil.sh` row), `docs/site/docs/architecture/index.md` (the install step's description), `CLAUDE.md` if it describes the step

**Interfaces:**

- Consumes: Task 2's A2. Task 5b's `dev_install` is not consumed; the VM's install is `vrg-vm`'s.

- [ ] **Step 1: Write the failing checks.**
  - `check-vm-init-series.sh`: if A2 is `none`, assert that `vrg-vm-init.sh` contains no `uv tool install` and no vergil-tooling install step. Otherwise, assert that it calls `vrg-vm update` with the series it reads, and contains no `uv tool install`.
  - `test_vergil.sh`: assert the packaged install. `command -v vrg-commit` is `/usr/bin/vrg-commit`, and the interpreter in `/opt/vergil/vergil-tooling/venv/pyvenv.cfg`'s `home` lies under `/opt/vergil/python/`. Cleanup purges `vergil-tooling` and the `vergil-python<x.y>` it pulled in, and removes the apt pin and the vergil source it added; there is no `uv tool uninstall`.
- [ ] **Step 2: Run** `vrg-container-run -- vrg-validate`; expect the changed checks to fail.
- [ ] **Step 3: Implement** the chosen change in `vrg-vm-init.sh`, and the matching `test_vergil.sh` install (the packaged install as `vm_packages.packaged_install` performs it: bootstrap, pin, update, candidate check, install). Update the two docs pages.
- [ ] **Step 4: Run** `vrg-container-run -- vrg-validate`; expect PASS.
- [ ] **Step 5: Commit** `vrg-commit --type feat --scope vm-init --message "leave the vergil-tooling install to vrg-vm"`, then report-ready.

### Task 7: vergil-tooling release with 5a and 5b; 6 merged (deployment, vergil-tooling)

Blocked by 5a, 5b, 6.

- **Precondition (human-attested):** a vergil-tooling release containing the PRs of 5a and 5b is published, and Task 6's PR is merged (and released, if vergil-vm releases it). The versions are named in a comment.
- **Human-run steps:** on the host, run the release's `consumer-refresh` (`uv tool install --python 3.15 …@v<VERSION>`, then `vrg-vm update --all`).
- **Procedure (agent-safe), in the agent's VM:**
  1. Confirm the release notes list 5a's and 5b's PRs.
  2. `vrg-whoami --mode` runs from `/usr/bin` on a venv whose interpreter lies under `/opt/vergil/python/`.
  3. The installed `vergil_tooling/lib/vm_packages.py` defines `_newest_runtime`; the installed `container_cache.py` carries the 3c refusal text (or the packaged-Python branch, per A1).
- **SUCCESS:** every check passes, recorded with the versions.

### Task 8: Live run of the host dev-pin and index-rebuild paths (validation, vergil-actions)

Blocked by 4.

- **Precondition (machine probe):** Task 4 is closed with `Outcome: SUCCESS`, and `v2.1`'s `install-tooling.sh` defines `read_floor`.
- **Human-run steps:**
  1. Open a draft PR on `vergil-fixture-python-lib` whose only change sets `[dependencies].vergil` to a vergil-tooling dev ref (`develop`). Name the PR in a comment; it is closed unmerged after the check.
  2. Dispatch `smoke-publish-index.yml` on vergil-actions at `v2.1`.
- **Procedure (agent-safe):**
  1. On the draft PR's CI run, every host job that uses the setup action (the `matrix` jobs among them) is green, and each log shows the dev-pin install on `/opt/vergil/python/<x.y>/bin/python3`, with `<x.y>` the newest packaged series.
  2. The dispatched `smoke-publish-index` run is green, and its `build-index` log shows `uv tool install --python <floor> --python-preference only-managed` and a passing series check.
- **SUCCESS:** both runs green with the logged interpreters, recorded with the run URLs. **FAILURE:** either run red or an unexpected interpreter; the task stays open and Task 9 stays blocked.

### Task 9: Floor raise, drift guard and floor documentation (vergil-tooling)

Blocked by 4, 7, 8.

**Files:**

- Modify: `pyproject.toml` (`requires-python`, `[tool.ruff] target-version`, `[tool.mypy] python_version`), `uv.lock`
- Modify: every `src/` and `tests/` file that ruff's `UP` rules rewrite for `py315`
- Create: `tests/vergil_tooling/test_python_floor_docs.py`
- Modify: `docs/site/docs/guides/consuming-repo-setup.md` (the dev-tree override's `uv sync --group dev`), `docs/site/docs/guides/releasing.md` (the "Developer host" consumer-steps row)
- Modify: `docs/site/docs/guides/validation-matrix.md` (new section, spec §6.4), `docs/site/mkdocs.yml` if the section becomes its own page

**Interfaces:**

- Produces: `test_python_floor_docs.py`, reading the floor from `pyproject.toml` with `tomllib` and `packaging.specifiers`. A floor that is not a bare `>=X.Y` fails the test.

- [ ] **Step 1: Write the failing tests** in `test_python_floor_docs.py`:
  - `test_documented_python_matches_floor`: every `--python <x.y>` in `vergil.toml`'s `[publish].consumer-refresh` and in `getting-started.md`, `consuming-repo-setup.md` and `releasing.md` equals the floor. The test fails if a file is missing or yields zero matches.
  - `test_every_documented_install_names_python`: every line in those files containing `uv tool install` with `vergil-tooling`, or a documented `uv sync`, carries `--python <floor>`.
  - `test_floor_is_bare_lower_bound`: the floor parses as `>=X.Y` alone.
- [ ] **Step 2: Run** `vrg-container-run -- vrg-validate`. Expect `test_every_documented_install_names_python` to fail on the two commands with no interpreter, and the floor test to fail on `3.12`.
- [ ] **Step 3: Raise the floor:** `requires-python = ">=3.15"`, `target-version = "py315"`, `python_version = "3.15"`. Re-lock with `vrg-container-run -- uv lock`. Apply `vrg-container-run -- ruff check --fix --select UP .`, then review the rewrites.
- [ ] **Step 4: Fix the two documented commands** with `--python 3.15`.
- [ ] **Step 5: Write the §6.4 section:** floor, tested series and core series (both definitions named); the overlap period; the five-step floor-raise procedure exactly as spec §6.4 orders it.
- [ ] **Step 6: Run** `vrg-container-run -- vrg-validate`; expect PASS, with one test leg (3.15) in `--all-versions`.
- [ ] **Step 7: Commit** `vrg-commit --type feat --scope python --message "narrow vergil-tooling to Python 3.15 and guard the documented floor"`, then report-ready. The PR's CI shows one `unit / 3.15` leg.

If the `UP` rewrites make the PR too big to review, split it per the task-sizing rule: 9a the floor and its rewrites, then 9b the drift guard and the docs (blocked by 9a).

### Task 10: vergil-tooling release with the floor raise (deployment, vergil-tooling)

Blocked by 9.

- **Precondition (human-attested):** a vergil-tooling release containing Task 9's PR is published, its version named in a comment.
- **Human-run steps:** the release's `consumer-refresh` on the host, then `vrg-vm update --all`.
- **Procedure (agent-safe), in the agent's VM:**
  1. The installed package's metadata says `Requires-Python: >=3.15`.
  2. `vrg-whoami --mode` runs on `/opt/vergil/python/3.15/`.
  3. The release's own CI run had exactly one test leg.
- **SUCCESS:** every check passes, recorded with the version.

### Task 11: Fixture runs on the Python fixtures

Filed with the `fixture-run` skill once Task 10 exists:

```text
fixture-run vergil-project/.github#1256 python-lib,python-app \
  --validates "vergil-tooling v<VERSION> (requires-python >=3.15)" \
  --after vergil-project/vergil-tooling#<Task 10>
```

No other fixture is seeded unless Task 2 finds that it reaches a changed path.
