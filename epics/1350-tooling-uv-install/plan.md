# Install vergil-tooling with uv everywhere — implementation plan

- **Epic:** vergil-project/.github#1350
- **Planning issue:** vergil-project/.github#466 (review history is in its
  comments)
- **Status:** draft, in design review
- **Spec:** [`spec.md`](spec.md)

> **For agentic workers:** each task below is one issue and one PR, driven
> by `epic-implement` / `issue-implement`; operational tasks run with
> `issue-deploy` or `issue-validate`. Steps use checkbox (`- [ ]`) syntax.

**Goal:** every vergil-tooling install is `uv tool install` of one resolved
release commit on a uv-managed Python 3.15, and nothing on the fleet's
development path reads the vergil package repository.

**Architecture:** one Python module in vergil-tooling,
`lib/tooling_install.py`, resolves a ref to a tag and SHA once and renders
the install as shell lines; the container cache build, `vrg-vm` and a new
`vrg-tooling-install` command (the Mac's consumer refresh) all use it. The
setup action keeps an equivalent shell implementation. `base` moves its core
Python to a managed 3.15 whose path it publishes in `VERGIL_CORE_PYTHON_DIR`.

**Tech stack:** Python 3.15 (vergil-tooling, pytest), bash (vergil-actions,
vergil-vm, with their `*.test.sh` suites), Dockerfile templates
(vergil-containers, `docker/generate.sh`), uv, GitHub rulesets.

## Global constraints

Copied from the spec; every task's requirements include them.

- Every vergil-tooling install: `uv tool install --python 3.15
  --managed-python --no-config`, with `UV_PYTHON`, `UV_PYTHON_PREFERENCE` and
  `UV_NO_MANAGED_PYTHON` unset.
- No install path names `/opt/vergil/python/…` or any interpreter path.
- A released ref resolves to a commit SHA exactly once (`git ls-remote`);
  code, lock, log and (for a line pin) cache key use the SHA. An exact pin's
  cache key is the tag.
- Released installs are constrained by the resolved commit's `uv.lock`,
  read over git, never `raw.githubusercontent.com`. Dev refs install
  unconstrained.
- No fetch of mutable content through a CDN-cached URL on an install path.
- No install switch and no package fallback; a failed uv install fails
  loudly. A line pin never installs an older release.
- `vrg-release`'s `package-index` stage is unchanged.
- Package publishing, the `pkg-*` images and their install tests are
  unchanged.

Two stated exceptions, and no others:

- **The installer bootstrap.** The Mac's consumer refresh, its documented
  first install and `vrg-vm-init.sh` run `uv tool run --no-config --python
  3.15 --managed-python --from 'vergil-tooling @ git+<URL>@v<VERSION>'
  vrg-tooling-install v<VERSION>`: by tag and unconstrained, in an
  ephemeral environment that exists only to run the installer, which then
  installs the tool by SHA with its lock.
- **The package setup action's driver.** It keeps its `python` input
  (`--python "$VRG_DRIVER_PYTHON"`) for the driver interpreter, and still
  takes `--managed-python`, `--no-config` and the cleared `UV_*`
  variables.

## Review focus

Inputs the spec implies that no single happy-path test covers; each line's
test is added to the task named.

1. **`git ls-remote` fails or returns no tag on the line** → a loud error
   naming the URL and the line, never an older or already-installed version
   (Task 3: `test_resolve_line_ls_remote_failure_is_loud`).
2. **The lock export names vergil-tooling itself or carries an editable
   line** → the constraints file holds only third-party pins, or `uv tool
   install` rejects it (Task 3: `test_constraints_exclude_the_project`).
3. **Reinstalling over an existing install** (same SHA, newer SHA, or a dev
   ref) → the result is exactly the requested commit; a newer SHA replaces
   the old one (Task 3: `test_install_lines_replace_existing_install`).
4. **A VM mid-transition with both the package and a uv copy** → after
   `vrg-vm update`, one copy, from uv, and the shadow check passes; a second
   update removes nothing and succeeds (Task 5a:
   `test_update_after_partial_cleanup_is_idempotent`).
5. **A container on a non-Vergil image** (no `VERGIL_CORE_PYTHON_DIR`) →
   the install downloads 3.15 and logs it; on a Vergil image with a wrong
   path it fails instead of downloading (Task 4 and Task 8, one test each).

## Delivery

Tasks are numbered in rollout order (spec §5). "Blocked by" is merge order
for PR tasks and run order for operational ones.

Repos: `T` vergil-tooling, `A` vergil-actions, `V` vergil-vm, `C`
vergil-containers, `G` .github. Kinds: `task`, `deploy`, `valid`
(validation), `fixture`, `docs`, `retro`.

| # | Task | Repo | Kind | Blocked by |
| --- | --- | --- | --- | --- |
| 1 | Spec and plan (#1351) | G | docs | — |
| 2a | Release tags pushed with the App token | A | task | — |
| 2b | Each repo passes the App secrets | each | task | — |
| 2ad | Release vergil-actions with 2a | A | deploy | 2a |
| 2 | Tag protection: add creation | T | task | 2a |
| 2d | Apply the tag ruleset fleet-wide | T | deploy | 2, 2ad, 2b |
| 3 | `tooling_install` and `vrg-tooling-install` | T | task | — |
| 4 | Cache build and key on `tooling_install` | T | task | 3 |
| 5a | VM install and cleanup | T | task | 3 |
| 5b | VM non-login `PATH` guarantee | T | task | 5a |
| 6 | `vrg-host` reports only (#4603) | T | task | — |
| 7d | Release; the Mac and every VM | T | deploy | 2d, 4, 5b, 6 |
| 8 | Setup action installs from git | A | task | — |
| 9 | Package setup action follows §4.2 | A | task | — |
| 10d | Release, moving `v2.1` | A | deploy | 7d, 8, 9 |
| 11 | vergil-vm scripts on the new rules | V | task | 3 |
| 13 | `base` on a managed 3.15 | C | task | 10d |
| 13v | The develop CD build's `dev-` images | C | valid | 13 |
| 14d | Release vergil-containers | C | deploy | 13v |
| 15 | Fixture runs | fixtures | fixture | 14d |
| 16v | Cold rebuild and headline check | V | valid | 11, 14d |
| R1 | Documentation review (#4710) | T | docs | 15, 16v |
| R2 | Retrospective (#1352) | G | retro | R1 |

Tasks 3, 6, 8, 9 and 11 can be worked side by side; their merges are
ordered only by the table. The two security advisories recorded on the
planning issue are handed on at filing: the tag ones to Tasks 2 and 2a,
the resolution one to Tasks 3, 4, 8 and 9. Tasks 2a, 2b, 2ad, 2 and 2d
touch no install path, so they can land before the freeze starts, and so
can Tasks 3, 6, 8, 9 and 11 (merged, not released). **The freeze starts at
Task 7d**, the first release that changes a live install path, and lasts
until Task 14d.

---

## Task 2: Tag protection: add creation

**Files:**

- Modify: `src/vergil_tooling/lib/github_config.py`
  (`desired_tag_protection_ruleset`)
- Test: `tests/vergil_tooling/test_github_config_lib.py`

**Interfaces:**

- Produces: `desired_tag_protection_ruleset(app_id: int | None) ->
  DesiredRuleset`, rules `creation`, `update`, `deletion`,
  `non_fast_forward` on `refs/tags/v*.*.*` and `refs/tags/v*.*`.

- [ ] **Step 1: Confirm the identities.** After Task 2a, `vX.Y.Z` tags are
  created with the automation App's installation token; `vX.Y` is moved by
  the Admin-run `vrg-promote`. Record both in the PR body.
- [ ] **Step 2: Write the failing tests.**

  ```python
  def test_tag_protection_blocks_creation():
      rules = {r["type"] for r in desired_tag_protection_ruleset(app_id=123).rules}
      assert rules == {"creation", "update", "deletion", "non_fast_forward"}

  def test_tag_protection_bypass_is_admin_and_release_identity_only():
      actors = desired_tag_protection_ruleset(app_id=123).bypass_actors
      assert {(a["actor_type"], a["actor_id"]) for a in actors} == {
          ("RepositoryRole", 5), ("Integration", 123)}
  ```

  The bypass never includes the GitHub Actions integration, which would
  let every workflow create a release tag.
- [ ] **Step 3: Run them:** `vrg-container-run -- pytest
  tests/vergil_tooling/test_github_config_lib.py -k tag_protection` → FAIL.
- [ ] **Step 4: Implement.** Thread `app_id` the way
  `desired_automation_branch_ruleset` does, and update
  `compute_desired_state`'s call.
- [ ] **Step 5: Run them again** → PASS; then `vrg-container-run --
  vrg-validate`.
- [ ] **Step 6: Commit** `feat(github-config): protect release tag creation`.

## Task 2a: Release tags pushed with the automation App's token

**Files (vergil-actions):**

- Modify: `actions/cd/release/tag-and-release/action.yml` (tag pushes use
  an App installation token, from `actions/create-github-app-token` with
  `APP_CLIENT_ID`/`APP_PRIVATE_KEY`, never the checkout's `GITHUB_TOKEN`),
  `.github/workflows/cd-release.yml` (the release job requires the two App
  secrets and fails before tagging when they are missing), and
  vergil-actions' own `.github/workflows/cd.yml` where it tags itself
- Test: `smoke-cd-release.yml` (a tag pushed by the App identity), the
  action's tests if any

- [ ] **Step 1: Write the failing check:** the smoke release asserts the
  pushed tag's creator is the App, and that a release job without the App
  secrets fails with a message naming them.
- [ ] **Step 2: Implement**; the `develop` changelog tag in the same action
  uses the same token.
- [ ] **Step 3:** `vrg-container-run -- vrg-validate`; the smoke workflow
  passes; commit `feat(release): push release tags with the automation App
  token`. (A delta under `.github/workflows/` is pushed by the human's
  `vrg-submit-pr`.)

## Task 2b (one per repo): Pass the App secrets to `cd-release.yml`

- **Step 1, before filing:** count every repo, in every org that
  `vrg-github-repo-config` manages, whose `cd.yml` calls `cd-release.yml`
  without `secrets: inherit` or the two App secrets. File one task per repo
  under this epic (today at least vergil-containers, vergil-vm and the
  language fixtures).
- **Each task:** that repo's `cd.yml` passes `APP_CLIENT_ID` and
  `APP_PRIVATE_KEY` (or `secrets: inherit`, matching the repo's existing
  style); `vrg-container-run -- vrg-validate`; commit `ci(cd): pass the
  automation App secrets to the release`.

## Task 2ad (deployment): Release vergil-actions with Task 2a

- **Preconditions:** Task 2a closed; the release is human-gated.
- **Procedure:** after the human's release moves `v2.1`, the next release
  of any one repo (the human's choice) shows its `vX.Y.Z` tag created by
  the App.
- **Acceptance:** that tag's creator is the App identity.

## Task 2d (deployment): Apply the tag ruleset fleet-wide

- **Preconditions:** Task 2 merged and released in vergil-tooling (a
  human-gated release); Task 2ad and every Task 2b closed.
- **Procedure:** run `vrg-github-repo-config` in apply mode for every repo
  of every managed org, then the config audit (`ops.yml`, or the audit
  command it runs).
- **Acceptance:** the audit reports no drift on "Tag protection" in any
  repo; a test tag push `v0.0.0-ruleset-check` by an agent identity to
  vergil-tooling is refused; and the next real release in a repo other
  than vergil-tooling creates its tag.

## Task 3: `tooling_install` module and `vrg-tooling-install`

**Files:**

- Create: `src/vergil_tooling/lib/tooling_install.py`
- Create: `src/vergil_tooling/bin/vrg_tooling_install.py` (console script
  `vrg-tooling-install` in `pyproject.toml`)
- Modify: `vergil.toml` (`[publish].consumer-refresh`)
- Modify: `docs/site/docs/guides/consuming-repo-setup.md`,
  `docs/site/docs/guides/releasing.md` (the Mac's first install and refresh)
- Test: `tests/vergil_tooling/test_tooling_install.py`,
  `tests/vergil_tooling/test_vrg_tooling_install.py`

**Interfaces:**

- Produces:
  - `GIT_URL = "https://github.com/vergil-project/vergil-tooling"`
  - `@dataclass(frozen=True) class ResolvedRef: ref: str; kind:
    Literal["line", "exact", "dev"]; tag: str | None; sha: str | None`
  - `resolve(ref: str, *, ls_remote: Callable[[str, str], str]) ->
    ResolvedRef`: a line → newest `vX.Y.Z` and its SHA; exact → its SHA; dev
    → `tag=None, sha=None`. Raises `ToolingInstallError` naming the URL and
    the ref.
  - `unresolved(ref: str) -> ResolvedRef`: an exact pin with `tag=ref`,
    `sha=None`, built with no network call. Raises `ToolingInstallError`
    for a line or dev ref.
  - `install_lines(resolved: ResolvedRef, *, python_dir: str | None,
    tool_dir: str | None, bin_dir: str | None) -> list[str]`: the shell
    lines that clear the three `UV_*` variables, export the commit's lock as
    constraints, run the install and log tag and SHA; with `python_dir`
    set they also set `UV_PYTHON_INSTALL_DIR` and
    `UV_PYTHON_DOWNLOADS=never`. For an exact ref with `sha=None` the lines
    begin with the one `git ls-remote` of the tag, fail loudly if it finds
    no tag, and use that SHA for the lock and the install.
  - `check_lines(bin_dir: str) -> list[str]`: `command -v vrg-whoami` must
    equal `<bin_dir>/vrg-whoami`, and `vrg-whoami --mode` must run.
  - `class ToolingInstallError(RuntimeError)`.
  - CLI: `vrg-tooling-install <ref>` resolves and installs on this host
    into uv's default tool directory, then runs the check.

- [ ] **Step 1: Write the failing tests.**

  ```python
  def test_resolve_line_picks_newest_patch_and_its_sha():
      r = resolve("v2.1", ls_remote=fake({"v2.1.9": "a"*40, "v2.1.10": "b"*40}))
      assert (r.tag, r.sha) == ("v2.1.10", "b"*40)

  def test_resolve_line_ls_remote_failure_is_loud(): ...  # names URL, ref
  def test_resolve_dev_ref_has_no_sha(): ...
  def test_unresolved_exact_needs_no_network(): ...  # no ls_remote call
  def test_install_lines_resolve_unresolved_exact_once_in_shell():
      # exactly one "git ls-remote", and its SHA feeds both the fetch
      # and the install spec
  def test_install_lines_use_sha_never_tag():
      lines = "\n".join(install_lines(resolved_exact, python_dir=None,
                                       tool_dir=None, bin_dir=None))
      after = lines.split("uv tool install", 1)[1]
      assert f"@{SHA}" in after and "@v2.1.10" not in after
  def test_install_lines_clear_uv_python_vars_and_ignore_config(): ...
      # "unset UV_PYTHON UV_PYTHON_PREFERENCE UV_NO_MANAGED_PYTHON", "--no-config",
      # "--python 3.15", "--managed-python" all present
  def test_install_lines_with_python_dir_forbid_downloads(): ...
  def test_constraints_exclude_the_project(): ...
      # the export uses --no-emit-project --no-dev --no-hashes --frozen
  def test_install_lines_replace_existing_install(): ...  # --force present
  def test_dev_ref_installs_unconstrained(): ...
  def test_cli_fails_loudly_on_failed_install(): ...
  def test_check_lines_fail_on_shadowing_usr_bin_shim(): ...
      # command -v resolves /usr/bin/vrg-whoami first → the check fails
  def test_consumer_refresh_runs_the_released_installer():
      # vergil.toml's consumer-refresh runs
      # uv tool run --no-config --python 3.15 --managed-python \
      #   --from 'vergil-tooling @ git+<GIT_URL>@v<VERSION>' \
      #   vrg-tooling-install v<VERSION>
  ```

- [ ] **Step 2: Run them** (`vrg-container-run -- pytest
  tests/vergil_tooling/test_tooling_install.py
  tests/vergil_tooling/test_vrg_tooling_install.py`) → FAIL.
- [ ] **Step 3: Implement `tooling_install.py`.** The lock export fetches
  the commit alone (`git init`, `git fetch --depth 1 <GIT_URL> <sha>`,
  `git checkout FETCH_HEAD`) into a temp directory and runs `uv export
  --frozen --no-dev --no-emit-project --no-hashes --format requirements-txt
  -o <tmp>/constraints.txt` there. The install is `uv tool install --force
  --no-config --python 3.15 --managed-python -c <tmp>/constraints.txt
  'vergil-tooling @ git+<GIT_URL>@<sha>'`. Reuse `vm_packages.classify_ref`
  for the ref kinds.
- [ ] **Step 4: Implement the CLI** and wire `vergil.toml`'s
  consumer-refresh to the `uv tool run … vrg-tooling-install v<VERSION>`
  form above, so the release's own installer runs on the Mac. Its second
  line, `vrg-vm update --all`, stays, so the release moves every VM too
  (Task 7d checks them). Update the
  two guides' Mac install text to the same command.
- [ ] **Step 5: Confirm the pinned uv's flags.** `vrg-container-run -- uv
  tool install --help` lists `--managed-python`, `--constraints`,
  `--force`; `uv --help` lists `--no-config`.
- [ ] **Step 6: Run the tests** → PASS; `vrg-container-run -- vrg-validate`.
- [ ] **Step 7: Commit** `feat(tooling-install): resolve once and install a
  release commit with uv`.

## Task 4: Cache build and cache key on `tooling_install`

**Files:**

- Modify: `src/vergil_tooling/lib/container_cache.py`
  (`vergil_tooling_setup_command`, delete `_packaged_lines`,
  `_dev_core_lines`, `_TOOLING_PIN_FILE`, `_CORE_PYTHON_*`;
  `_resolve_tooling`)
- Modify: `src/vergil_tooling/lib/tooling_release.py` (index reading goes;
  `resolve_tooling_tag` becomes a thin call to `tooling_install.resolve`)
- Test: `tests/vergil_tooling/test_container_cache.py`,
  `tests/vergil_tooling/test_tooling_release.py`

**Interfaces:**

- Consumes: `tooling_install.resolve`, `install_lines`, `check_lines`.
- Produces: `vergil_tooling_setup_command(resolved: ResolvedRef) -> str`;
  `_resolve_tooling(...) -> tuple[str, ResolvedRef]`, the cache key part
  being `"<tag>@<sha>"` for a line pin and `"<tag>"` for an exact pin.

- [ ] **Step 1: Write the failing tests.**

  ```python
  def test_setup_command_has_no_apt(): assert "apt" not in vergil_tooling_setup_command(r)
  def test_setup_command_wrong_python_dir_fails_not_downloads(): ...
      # VERGIL_CORE_PYTHON_DIR set to an empty dir → install fails, no download
  def test_setup_command_reads_image_python_dir():
      # the fragment sets UV_PYTHON_INSTALL_DIR="$VERGIL_CORE_PYTHON_DIR" and
      # UV_PYTHON_DOWNLOADS=never only when VERGIL_CORE_PYTHON_DIR is set,
      # and otherwise logs "vergil-tooling: no VERGIL_CORE_PYTHON_DIR;
      # uv may download 3.15"
  def test_setup_command_installs_into_usr_local_bin_and_checks_it(): ...
      # UV_TOOL_DIR=/opt/vergil/uv/tools, UV_TOOL_BIN_DIR=/usr/local/bin,
      # check_lines("/usr/local/bin")
  def test_line_pin_cache_key_records_sha(): ...
  def test_exact_pin_key_needs_no_network():
      # an exact pin with an ls_remote that raises still returns the tag
  def test_exact_pin_resolves_sha_inside_the_build(): ...
  def test_resolve_error_message_keeps_exact_pin_remedy(): ...  # VRG_DOCKER_INSTALL_TAG
  ```

- [ ] **Step 2: Run them** → FAIL.
- [ ] **Step 3: Implement.** For an exact pin, `_resolve_tooling` uses
  `tooling_install.unresolved` and the fragment's own `ls-remote` resolves
  the SHA inside the build; for a line pin the host calls `resolve` once
  and the fragment receives the SHA.
- [ ] **Step 4: Run the tests** → PASS, then a real cache build on today's
  `base` (`vrg-container-run -- vrg-whoami --mode` in a scratch repo
  pinned `v2.1`), which downloads 3.15 and logs it, as Review focus 5
  expects before Task 13.
- [ ] **Step 5:** `vrg-container-run -- vrg-validate`; commit
  `feat(container-cache): install vergil-tooling from one release commit with uv`.

## Task 5a: VM install and cleanup

**Files:**

- Modify: `src/vergil_tooling/lib/vm_packages.py` (`packaged_install`
  replaced by `released_install`; `_remove_uv_copy`,
  `_purge_obsolete_runtimes` go; `DEV_BANNER`; new `cleanup_packages`)
- Modify: `src/vergil_tooling/lib/vm_guest.py` (`_install`,
  `get_tooling_version`)
- Modify: `src/vergil_tooling/lib/lima.py`, `vm_cloud.py` (create stops
  calling `repo_setup.bootstrap`)
- Test: `tests/vergil_tooling/test_vm_packages.py`,
  `tests/vergil_tooling/test_vm_guest.py`

**Interfaces:**

- Consumes: `tooling_install.resolve`, `install_lines`, `check_lines`.
- Produces: `released_install(transport: Transport, ref: str) -> None`;
  `cleanup_packages(transport: Transport) -> list[str]` (names what it
  removed); `DEV_BANNER = "DEV tooling (ref {ref}) — dev ref, not a
  release"`.

- [ ] **Step 1: Write the failing tests.**

  ```python
  def test_released_install_runs_tooling_install_lines(): ...
  def test_released_install_clears_dev_ref_marker(): ...  # rm ~/.config/vergil/tooling-dev-ref
  def test_cleanup_removes_named_packages_source_keyring_and_pin():
      # apt-get remove -y vergil-tooling + each installed vergil-python*,
      # vergil-archive-keyring; rm /etc/apt/sources.list.d/vergil.sources and
      # /etc/apt/preferences.d/vergil-tooling; never "autoremove"
  def test_cleanup_runs_only_after_the_check_passes(): ...
  def test_cleanup_confirms_nothing_left(): ...  # raises if anything is left
  def test_update_after_partial_cleanup_is_idempotent(): ...
  def test_create_does_not_bootstrap_the_vergil_repository(): ...
  def test_dev_banner_text(): assert "not a release" in DEV_BANNER
  ```

- [ ] **Step 2: Run them** → FAIL.
- [ ] **Step 3: Implement**; keep `_uv_tool_install`'s cache self-heal
  around the install lines.
- [ ] **Step 4: Run them** → PASS; `vrg-container-run -- vrg-validate`.
- [ ] **Step 5: Commit** `feat(vm): install vergil-tooling with uv and
  remove the vergil packages`.

## Task 5b: VM non-login `PATH` guarantee

**Files:**

- Modify: `src/vergil_tooling/lib/vm_guest.py` (`update_tooling` applies
  the `/etc/environment` `PATH` line, the same text as vergil-vm's
  `templates/provision/15-environment.sh`)
- Modify: every bare in-guest `vrg-*` call found by Step 1 (at least
  `bin/vrg_vm.py`'s `transport.run("vrg-vm-resolve-session", …)`)
- Test: `tests/vergil_tooling/test_vm_guest.py`,
  `tests/vergil_tooling/test_vrg_vm.py`

**Interfaces:**

- Consumes: `vm_packages.released_install`.
- Produces: `ensure_login_path(transport: Transport) -> bool` (True when it
  changed `/etc/environment`); when it returns True on a Lima transport,
  `vrg-vm update` closes Lima's SSH ControlMaster connection before the
  check, so the check opens a fresh session that reads the new `PATH`
  (pam_env reads `/etc/environment` once per session; vergil-vm issue
  #110). The post-install check runs as `transport.run("vrg-whoami",
  "--mode")` with no `PATH` prepend.

- [ ] **Step 1: Inventory.** List every in-guest `vrg-*` call on each
  transport (Lima, gcloud ssh, ssh) that runs without a `PATH` prepend; put
  the list in the PR body.
- [ ] **Step 2: Write the failing tests:**
  `test_update_writes_local_bin_into_etc_environment`,
  `test_lima_reconnects_after_environment_change`,
  `test_no_reconnect_when_line_already_present`,
  `test_post_install_check_is_bare_non_login`,
  `test_every_bare_in_guest_call_is_listed` (a static check over the
  inventory).
- [ ] **Step 3: Run** → FAIL; **implement**; **run** → PASS.
- [ ] **Step 4: Prove it on a real VM.** On one Lima VM and one cloud VM,
  `vrg-vm update` to a dev ref of this branch, then a bare
  `vrg-vm-resolve-session --list-json` over each transport succeeds. Record
  the output in the PR body.
- [ ] **Step 5:** `vrg-container-run -- vrg-validate`; commit
  `fix(vm): guarantee ~/.local/bin on PATH for non-login remote commands`.

## Task 6: `vrg-host` reports vergil-tooling only (#4603)

**Files:**

- Modify: `src/vergil_tooling/data/host_dependencies.toml` (the
  vergil-tooling entry), `src/vergil_tooling/bin/vrg_host.py`
- Test: `tests/vergil_tooling/test_vrg_host.py`,
  `tests/vergil_tooling/test_host_methods.py`

- [ ] **Step 1: Write the failing tests:**
  `test_upgrade_never_runs_uv_tool_upgrade_for_vergil_tooling` (no
  `uv tool upgrade vergil-tooling` in the recorded commands, `--all`
  included) and `test_status_reports_tooling_owned_by_release` (output
  names the installed version and "installed by the vergil-tooling
  release", and does not claim it is current).
- [ ] **Step 2: Run** → FAIL; **implement** per #4603's body; **run** →
  PASS; `vrg-container-run -- vrg-validate`.
- [ ] **Step 3: Commit** `fix(vrg-host): report vergil-tooling and leave its
  install to the release`.

## Task 7d (deployment): Release vergil-tooling; the Mac and every VM

- **Preconditions:** Tasks 2d, 4, 5b and 6 closed; the release itself is
  human-gated.
- **Procedure:** the human runs `vrg-release`. Its consumer refresh runs
  the new `vrg-tooling-install` on the Mac, then `vrg-vm update --all`,
  which moves every VM. Then check the Mac and each VM, and close
  vergil-project/vergil-tooling#4706 as superseded by this epic.
- **Acceptance, the Mac:** `uv tool list --show-version-specifiers` shows
  `@<sha>` of the release; `command -v vrg-whoami` is
  `~/.local/bin/vrg-whoami`; `vrg-host status` reports vergil-tooling as
  owned by the release.
- **Acceptance, per VM:** `dpkg-query -W 'vergil-*'` lists nothing; no
  `/etc/apt/sources.list.d/vergil.sources`; a bare non-login `vrg-whoami
  --mode` over the VM's transport resolves `~/.local/bin/vrg-whoami`; the
  dev banner is absent. A VM that fails stays as it is, for the human.
- **Acceptance:** #4706 closed.

## Task 8: Setup action installs from git

**Files:**

- Modify: `actions/shared/setup/vergil/install-tooling.sh` (delete
  `container_install`, `host_install`, `bootstrap_vergil_repository`,
  `write_pin`, `check_candidate`, `verify_packaged`, `packaged_install`,
  `find_core_pythons`; add `resolve_sha`, `export_constraints`,
  `released_install`)
- Modify: `actions/shared/setup/vergil/action.yml`,
  `docs/site/docs/actions/shared-setup-vergil.md`
- Test: `actions/shared/setup/vergil/tests/install-tooling.test.sh`

**Interfaces:**

- Consumes: the same rules as `tooling_install` (kept equivalent by hand).
- Produces: in every job, `vrg-*` in `$(uv tool dir --bin)`, appended to
  `GITHUB_PATH`; in a container job with `VERGIL_CORE_PYTHON_DIR` set, no
  interpreter download.

- [ ] **Step 1: Write the failing tests** (bash, in the suite's style):
  `test_released_ref_installs_sha_with_constraints`,
  `test_line_ref_resolves_newest_patch`,
  `test_container_job_uses_image_python_dir_and_forbids_download`,
  `test_container_job_without_dir_logs_download`,
  `test_container_job_wrong_dir_fails_not_downloads`,
  `test_no_apt_calls_on_any_path`,
  `test_check_expects_uv_bin_dir_not_usr_bin`,
  `test_install_runs_with_no_config_and_cleared_uv_vars`.
- [ ] **Step 2: Run** `vrg-container-run -- bash
  actions/shared/setup/vergil/tests/install-tooling.test.sh` → FAIL.
- [ ] **Step 3: Implement**, with `ensure_uv`'s pinned uv; confirm it
  accepts `--managed-python` and `--no-config`, else bump its pin and
  hashes in the same PR.
- [ ] **Step 4: Run** → PASS; `vrg-container-run -- vrg-validate`; the
  `smoke-setup-vergil.yml` workflow passes on the PR.
- [ ] **Step 5: Commit** `feat(setup-vergil): install vergil-tooling from
  one release commit with uv`.

## Task 9: Package setup action follows §4.2

**Files:**

- Modify: `actions/package/setup/detect.sh`, `actions/package/setup/action.yml`
- Test: `actions/package/setup/tests/detect.test.sh`

- [ ] **Step 1: Write the failing tests:**
  `test_line_pin_prints_sha_spec` (a `vX.Y` pin yields
  `vergil-tooling @ git+…@<sha>` of the newest `vX.Y.Z`),
  `test_install_is_constrained_by_the_commit_lock`,
  `test_self_repo_still_installs_checkout`.
- [ ] **Step 2: Run** → FAIL; **implement** (keep `--python
  "$VRG_DRIVER_PYTHON"` and `--no-config`); **run** → PASS;
  `vrg-container-run -- vrg-validate`.
- [ ] **Step 3: Commit** `feat(package-setup): install the driver
  vergil-tooling from one release commit`.

## Task 10d (deployment): Release vergil-actions, moving `v2.1`

- **Preconditions:** Tasks 7d, 8 and 9 closed; the release is human-gated.
- **Procedure:** after the human's release moves `v2.1`, run one consumer
  repo's CI (a host job and a container job) and one packaged repo's
  package job.
- **Acceptance:** each log shows the resolved tag and SHA and no apt call;
  the container job logs the download note (the new `base` is not out
  yet).

## Task 11: vergil-vm scripts on the new rules

**Files:**

- Modify: `scripts/vrg-vm-init.sh`, `tests/test_vergil.sh`
- Check: `templates/provision/15-environment.sh` and the generated
  `cloud-init.yaml` files (`scripts/build-cloud-init.sh`,
  `tests/check-cloud-init-generation.sh`) keep the `~/.local/bin` `PATH`
  line

- [ ] **Step 1:** where `vrg-vm create` already installs vergil-tooling,
  delete the script's own install; otherwise call `vrg-tooling-install`
  via `uv tool run` as Task 3's consumer refresh does.
- [ ] **Step 2:** `test_vergil.sh` asserts `command -v vrg-whoami` is under
  `~/.local/bin` and that no `vergil-tooling` package is installed.
- [ ] **Step 3:** `vrg-container-run -- vrg-validate`; commit
  `feat(vm): use the vergil-tooling uv install rules`.

## Task 13: `base` on a managed 3.15

**Files:**

- Modify: `docker/common/core-python.dockerfile`,
  `docker/common/core-tools.dockerfile`, `docker/base/Dockerfile.template`
  (drop the `vergil-apt.dockerfile` include from `base`), then regenerate
  with `docker/generate.sh`
- Delete: `docker/common/vergil-apt.dockerfile` (`base` is its only
  includer)
- Modify: `docker/base/checks-invariants.sh`,
  `docker/base/checks-defaults.sh`
- Modify: `docker/versions/test_generate.py` (remove `vergil-apt` from
  `_CORE_FRAGMENTS`, which `test_base_template_shape` and
  `test_core_fragments_are_base_only` read)
- Test: the image build checks themselves, and `test_generate.py`

**Interfaces:**

- Produces: `ENV VERGIL_CORE_PYTHON_DIR=/opt/vergil/uv/python` in `base`;
  `/usr/local/bin/python3` → the managed 3.15 under it; core tools in
  `/opt/vergil/uv/tools`.

- [ ] **Step 1: Inventory** every reader of the vergil apt source in the
  image tree (including any `[container].system-packages` user); list them
  in the PR body. Any reader other than `base`'s own fragments stops the
  task for the human.
- [ ] **Step 2: Change the checks first**, so they fail on today's image:
  core tools and `python3` resolve under `$VERGIL_CORE_PYTHON_DIR`; a
  build fails if `/etc/apt/sources.list.d/vergil.sources` or the vergil
  keyring exists. Run them against today's `base` → FAIL.
- [ ] **Step 3: Change the fragments.** `UV_PYTHON_INSTALL_DIR=$VERGIL_CORE_PYTHON_DIR
  uv python install 3.15`, passed only to that command and to the core
  tools' creation (never an image-wide `UV_*`); link `python3`; record the
  resolved patch in the build log.
- [ ] **Step 4: Build** `base` and one image per family; every image's
  checks pass, and `docker/python/checks.sh`'s no-managed-interpreter check
  is unchanged and passes.
- [ ] **Step 5:** `vrg-container-run -- vrg-validate`; commit
  `feat(base): move the core Python to a managed 3.15 and drop the vergil
  apt source`.

## Task 13v (validation): The develop CD build's `dev-` images

- **Preconditions:** Task 13 merged.
- **Procedure:** find the develop CD run for a commit containing Task 13;
  confirm it succeeded; in the published `dev-base` and one `dev-python`
  image, `printenv VERGIL_CORE_PYTHON_DIR`, `python3 --version` (3.15), and
  `test ! -e /etc/apt/sources.list.d/vergil.sources`.
- **Acceptance:** all pass; a cache build with `container-prefix: dev`
  installs vergil-tooling with no interpreter download.

## Task 14d (deployment): Release vergil-containers

- **Preconditions:** Task 13v closed; the release is human-gated; a recount
  of every consuming repo's vergil-tooling pin shows no consumer whose
  cache build still runs pre-Task-7d tooling.
- **Acceptance:** the `prod-` images carry what Task 13v checked.

## Task 15: Fixture runs

File with the `fixture-run` skill once Task 14d exists, for every language
fixture it lists:

```text
fixture-run vergil-project/.github#1350 <all fixtures> \
  --validates "vergil-tooling <7d>, vergil-actions v2.1 <10d>, \
               vergil-containers <14d>" \
  --after <7d> --after <10d> --after <14d>
```

## Task 16v (validation): Cold rebuild and headline check

- **Preconditions:** Tasks 11 and 14d closed.
- **Procedure:**
  1. Create a new Lima VM from scratch: vergil-tooling from uv, no vergil
     package, no vergil apt source, bare non-login `vrg-whoami` works.
  2. A cache build on the new `prod-base` installs vergil-tooling with no
     interpreter download (`UV_PYTHON_DOWNLOADS=never` in its log).
  3. Headline: right after the next vergil-tooling release (human-gated),
     with no wait, a fresh cache build and a `vrg-vm update` both install
     it.
- **Acceptance:** all three pass; record timings for step 3.

## Bookends

- **R1, documentation review** (vergil-project/vergil-tooling#4710): the
  sweep the spec names, spawning per-repo doc tasks in vergil-actions,
  vergil-containers, vergil-vm and docs.
- **R2, retrospective** (vergil-project/.github#1352).
- **The package-hosting follow-on** (spec §7) is filed into the deferred
  queue when the tasks are filed, carrying vergil-project/.github#466's
  survey list and vergil-project/.github#365, with "revisit when the home
  hosting platform exists (February 2027)".
