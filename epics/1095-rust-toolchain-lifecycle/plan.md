# Rust toolchain on the leading edge — implementation plan

- **Epic:** vergil-project/.github#1095
- **Planning issue:** vergil-project/.github#410 (review history is in its comments)
- **Status:** draft, in design review

> **For agentic workers:** each task is one issue and one PR in the repo named
> in its heading, worked with `issue-implement`; operational tasks are worked
> with `issue-deploy` or `issue-validate`. Steps use checkbox (`- [ ]`)
> syntax for tracking.

**Goal:** Rust series arrive and leave through lifecycle PRs that carry their
toolchain pins, and inside every Vergil Rust image the image's own release is
the only toolchain a repo's configuration can reach.

**Architecture:** vergil-containers renders lifecycle PRs from in-memory pins
and pins rustup in the Rust image; vergil-tooling adds Rust's floor and
toolchain-file checks, keeps rustup's variables out of the container
passthrough and narrows the Rust allowed actions; vergil-actions asserts the
Rust leg's `x.y`. Rust then moves to `automation: "pr"` and catches up through
its own PRs.

**Tech Stack:** Python 3 (vergil-containers `docker/` scripts, vergil-tooling),
pytest, bash (`assert.sh`, `checks.sh`), Dockerfile, rustup.

**Spec:** [`spec.md`](spec.md) in this directory. Executors read both.

## Global Constraints

- Consumers deal only in `x.y`; no consumer file, test or doc names a Rust patch release (spec §1).
- The shared lifecycle code stays language-neutral: no family key appears as a branch in `apply.py` or `publish.py` (spec §8).
- Fail loudly: an unresolved pin, an unparseable `Cargo.toml` or `rust-version`, or a failed `rustc` call is an error, never skipped (spec §7, §8).
- Every version comparison is numeric: each component touched is tested with a `1.100` series beside `1.99` (spec §6.6).
- Lifecycle PRs stay human-merged; releases are human-gated; no agent cuts a release (spec §8).
- Validation runs only through `vrg-container-run -- vrg-validate` in the task repo's worktree.
- Commit with `vrg-commit`; hand off with `vrg-pr-workflow report-ready`.

## Review Focus

1. **Two adds and a drop in one publish run.** Each PR carries exactly its own pin change, and the shared checkout is untouched afterwards. Pinned in Task 2b.
2. **A drop of a series whose pin another manifest entry still needs** (`node: "22"` for `core.node`). The pin stays. Pinned in Task 2b.
3. **A workspace root that inherits its floor** (`[package] rust-version.workspace = true`). It reads `[workspace.package].rust-version` and never raises. Pinned in Task 4.
4. **A host with `[container].env-prefixes = ["RUST"]` and `RUSTUP_TOOLCHAIN` set.** The variable does not reach the container. Pinned in Task 5.
5. **`rustc 1.100.0` on a leg that expects `1.100`.** The assertion passes; `1.10` against `1.100` fails. Pinned in Task 7.

---

## Delivery table

| Task | Repo | Kind | What | Blocked by |
|---|---|---|---|---|
| 1 | .github | task (#1096) | Spec and plan (this PR) | — |
| 2a | vergil-containers | task | Renderers take the toolchain pins as a value | — |
| 2b | vergil-containers | task | Lifecycle add and drop PRs write and remove their pins | 2a |
| 3 | vergil-containers | task | The Rust image pins rustup to its own release | — |
| 4 | vergil-tooling | task | Rust floor and toolchain-file checks, and the Rust standards page | — |
| 5 | vergil-tooling | task | rustup variables never forwarded into a container | — |
| 6 | vergil-tooling | task | Rust allowed actions drop `actions-rust-lang/*` | — |
| 7 | vergil-actions | task | Rust branch of the toolchain assertion | — |
| 8 | .github | deployment | Tooling, actions and containers releases attested; Rust images rebuilt with the pin | 2b, 3, 4, 5, 6, 7 |
| 9 | vergil-containers | task | Rust moves to `automation: "pr"`, with the policy page's automation text | 8 |
| 10 | .github | deployment | Rust catch-up: lifecycle PRs merged in order | 9 |
| 11 | vergil-containers | task | Rust policy page: out-of-policy section rewritten after the catch-up | 10 |
| 12 | .github | validation | Containers release after the catch-up attested; end-to-end check on the Rust fixtures | 10, 11 |
| — | vergil-tooling | bookend (#4319) | Documentation review | 2b, 3, 4, 5, 6, 7, 9, 11 |
| — | .github | bookend (#1097) | Follow-on brainstorm: pick up Epic B (#1087) | 12 |
| — | .github | bookend (#1098) | Retrospective | all |

**Filing.** Every task above is filed at epic-create's filing step, each in
its repo and linked under #1095, with its `Blocked-by:` refs. Task 12's
dependency on the fixture repos (epic vergil-project/.github#1091) is a
precondition in its body, because the fixture tasks that create them are not
yet filed.

---

### Task 2a: Renderers take the toolchain pins as a value (vergil-containers)

**Files:**
- Modify: `docker/versions/generate.py` (`render_all`, `main`; `load_pins` stays)
- Modify: `docker/pins/generate_catalog.py` (`render`, `main`)
- Modify: `docker/versions/frozen.py` (`make_repo`'s two `render_all` calls pass the frozen `PINS`)
- Test: `docker/versions/test_generate.py`, `docker/pins/test_toolchains.py` (the catalog's toolchain-table tests)

**Interfaces:**
- Produces: `generate.render_all(m: mf.Manifest, root: Path, pins: rt.Toolchains) -> dict[Path, str]` — renders from `pins`, never reads `toolchains.yml`.
- Produces: `generate_catalog.render(root: Path, toolchains: rt.Toolchains) -> str` — `root` is the `docker/` dir as today; the toolchain table comes from `toolchains`.
- Both `main`s load the pins from disk (`generate.load_pins(root)`, `rt.load_toolchains(...)`) and pass them in, so the CLI output is byte-for-byte unchanged.

- [ ] **Step 1: Write the failing tests.** `test_render_all_uses_the_pins_passed_in`: a fixture root whose `toolchains.yml` pins rust 1.99 to `1.99.0`, rendered with a `Toolchains` value pinning `1.99.1`; `matrix.json` carries `TOOLCHAIN_VERSION: 1.99.1`. `test_catalog_render_uses_the_pins_passed_in`: the same for the catalog's "Toolchain pins" table. `test_cli_output_unchanged`: `main(["--check"])` on the real tree still passes.
- [ ] **Step 2: Run** `vrg-container-run -- vrg-validate`; expect the two new tests to fail with a `TypeError` on the extra argument.
- [ ] **Step 3: Implement** the signatures above and update every caller (`generate.main`, `generate_catalog.main`, both calls in `frozen.make_repo`, which pass `PINS`, and `apply.render_change`, which passes `generate.load_pins(root)` for now).
- [ ] **Step 4: Run** `vrg-container-run -- vrg-validate`; expect PASS.
- [ ] **Step 5: Commit** `vrg-commit --type refactor --scope versions --message "render the matrix and pin catalog from pins passed in"`, then report-ready.

### Task 2b: Lifecycle add and drop PRs write and remove their pins (vergil-containers)

**Files:**
- Modify: `docker/versions/lifecycle/apply.py` (`render_change`; new `edit_pins`)
- Modify: `docker/versions/lifecycle/publish.py` (a `fetch` keyword through `sync_prs` and `_publish_change`; `main` passes `rt.fetch_url`)
- Modify: `docker/versions/frozen.py` (`make_repo` also copies `docker/pins/pins.yml` and `docker/common/*.dockerfile`, and writes `docker/pins/CATALOG.md` rendered with `generate_catalog.render(dest / "docker", PINS)`, so `render_change` has every catalog input in the frozen tree)
- Modify: the lifecycle and toolchain-pin docs in `docs/site/docs/` that say lifecycle PRs leave `toolchains.yml` alone
- Test: `docker/versions/test_apply.py`, `docker/versions/test_publish.py`, `docker/pins/test_toolchains.py`

**Interfaces:**
- Consumes: Task 2a's `render_all(m, root, pins)` and `generate_catalog.render(root, toolchains)`; `rt.required_series`, `rt.required_tools`, `rt.RESOLVERS`, `rt.TOOL_RESOLVERS`, `rt.consistency_errors`, `rt.dump_toolchains`, `rt.load_toolchains`.
- Produces: `apply.edit_pins(after: Manifest, base: rt.Toolchains, fetch: rt.Fetch) -> rt.Toolchains` — the base pins, where every series in `required_series(after)` that `base` does not pin is resolved through `RESOLVERS[key]([series], fetch)`, every series `base` pins that `required_series(after)` no longer lists is removed, and each tool in `required_tools(after)` that `base` lacks is resolved through `TOOL_RESOLVERS`. Nothing else re-resolved. A `ResolutionError` or a non-empty `consistency_errors(after, result)` raises `ApplyError` naming the manifest family (`rt.KINDS[key].family`, so `ts-node`, never the toolchain key `node`) and the series.
- Produces: `apply.render_change(m, change, target, root, *, fetch: rt.Fetch) -> dict[Path, str]`, with `fetch` required (no default), so a test that forgets a fake fails with a `TypeError` and never reaches the network. It computes `edit_pins(after, generate.load_pins(root), fetch)`, renders from it, and adds `docker/pins/toolchains.yml` and `docker/pins/CATALOG.md` when their content differs from the files in `root`. Writes nothing to `root`.

**Test data.** The frozen snapshot (`testdata/manifest.yml`, `testdata/toolchains.yml`) has rust under `automation: "report"` at 1.93/1.92 and stays unchanged. Every rust test below builds its own manifest and pins from `frozen.SEED` and `frozen.PINS` with `dataclasses.replace`: rust under `automation: "pr"`, with series 1.99/1.98 pinned to `1.99.0`/`1.98.0`, and a rendered tree from `frozen.make_repo`'s helpers. A shared helper in `test_apply.py`, `rust_pr_fixture(tmp_path) -> tuple[Manifest, rt.Toolchains, Path]`, builds it once.

- [ ] **Step 1: Write the failing tests** in `test_apply.py`, with a fake fetch, on `rust_pr_fixture`:
  - `test_add_writes_the_new_series_pin`: a rust add of `1.100` yields `toolchains.yml` with a `"1.100"` pin, listed before `"1.99"`.
  - `test_add_does_not_rebump_other_series`: the `1.99` and `1.98` pins are byte-identical before and after.
  - `test_add_writes_a_missing_tool_pin_only`: `rustup-init` is resolved only when the base lacks it.
  - `test_drop_removes_an_orphaned_pin`: a go drop removes its pin.
  - `test_drop_keeps_a_pin_core_needs`: a ts-node `22` drop with `core.node: "22"` keeps `node: "22"`.
  - `test_unresolvable_pin_is_apply_error`: a fetch that raises gives `ApplyError` naming `rust` and the series; the same for a ts-node add names `ts-node`, not `node`.
  - `test_render_change_leaves_root_untouched`: the files under `root` are unchanged after a call.
  - `test_catalog_in_pr_files`: an add's file list includes `docker/pins/CATALOG.md`.
  - Rewrite `test_render_add_without_a_toolchain_pin_fails_as_an_apply_error` (`test_apply.py:657`), which asserts the behaviour this task removes: an add of python 3.16 with no pin now resolves it through the injected fetch, and the `ApplyError` case moves to `test_unresolvable_pin_is_apply_error`.
  - Every existing `render_change` and `sync_prs` test passes a fake fetch.
- [ ] **Step 2: Write the failing test** `test_one_run_two_adds_and_a_drop` in `test_publish.py`, on `rust_pr_fixture`: a run with rust add `1.100` and rust drop `1.98`, plus a go add from the frozen data (go is `pr` there). Each published branch's `toolchains.yml` differs from the base by exactly its own series.
- [ ] **Step 3: Write the three-digit-minor tests** (spec §6.6): `test_parse_branch_accepts_three_digit_minor` (`lifecycle/rust-add-1.100` parses) in `test_apply.py`; `test_dump_orders_1_100_before_1_99` and `test_parse_toolchains_accepts_1_100` (a `"1.100": 1.100.0` rust pin parses and passes the series check) in `test_toolchains.py`; `test_rust_window_with_1_100` (`RustAdapter.target` over endoflife data listing 1.100, 1.99 and 1.98 gives 1.100 and 1.99) in `test_adapters_report_only.py` (where the Rust adapter is tested today); `test_classify_orders_1_100_newest` (a rust family under `pr` at 1.99/1.98, against a window of 1.100/1.99, gives `add 1.100`, then `drop 1.98`) in `test_lifecycle_core.py`; and `test_rust_1_100_build_args` (the rendered matrix gives `RUST_VERSION=1.100` with `TOOLCHAIN_VERSION=1.100.0`) in `test_generate.py`.
- [ ] **Step 4: Run** `vrg-container-run -- vrg-validate`; expect the new tests to fail.
- [ ] **Step 5: Implement** `edit_pins` and the new `render_change`, and thread `fetch` through `sync_prs` and `_publish_change` from `publish.main`. Update the docs that describe lifecycle PRs.
- [ ] **Step 6: Run** `vrg-container-run -- vrg-validate`; expect PASS.
- [ ] **Step 7: Commit** `vrg-commit --type fix --scope lifecycle --message "write and remove toolchain pins in lifecycle add and drop PRs"`, then report-ready.

### Task 3: The Rust image pins rustup to its own release (vergil-containers)

**Files:**
- Modify: `docker/rust/Dockerfile.template` (after `ENV RUST_VERSION=${TOOLCHAIN_VERSION}`)
- Modify: `docker/rust/checks.sh`
- Modify: the Rust image docs under `docs/site/docs/images/` (or the toolchain-pin page) to describe the image-level pin
- Test: the generated Rust Dockerfiles via `generate.py --check`, and `checks.sh` at image build

**Interfaces:**
- Produces: in every Rust image, `RUSTUP_TOOLCHAIN` equals `RUST_VERSION` (the exact installed release) and rustup's auto-install is off.

- [ ] **Step 1: Verify the switch.** Read the rustup changelog and docs for the `rustup-init` version pinned in `docker/pins/toolchains.yml` (1.29.3 at planning time). Confirm: (a) `RUSTUP_TOOLCHAIN` outranks directory overrides and `rust-toolchain.toml`; (b) the name and value that turn auto-install off (`RUSTUP_AUTO_INSTALL=0` expected). Record both URLs in the PR body. If (b) differs, use the documented mechanism and correct spec §6.3 in a separate `.github` PR.
- [ ] **Step 2: Write the failing checks** in `checks.sh`, in its `ok`/`FAIL` style:
  - `RUSTUP_TOOLCHAIN` equals `RUST_VERSION`;
  - in `mktemp -d` holding `rust-toolchain.toml` with `channel = "1.0.0"`, `rustc --version` reports `RUST_VERSION`;
  - `(RUSTUP_TOOLCHAIN=1.0.0 rustc --version)` exits non-zero, and the output of `rustup toolchain list` plus `ls "$RUSTUP_HOME/toolchains"` is identical before and after.
- [ ] **Step 2b: Write the failing series-check test** `test_rust_series_check_three_digit_minor` in `docker/versions/test_ci_images.py` (or a new `docker/versions/test_rust_image.py` beside `test_python_image.py`). It extracts the `case "${TOOLCHAIN_VERSION}" in` block from `docker/rust/Dockerfile.template` (read from the file, not copied) and runs it with bash: `TOOLCHAIN_VERSION=1.100.0` is rejected for `RUST_VERSION=1.10` and accepted for `RUST_VERSION=1.100`, and `1.99.1` is accepted for `1.99`.
- [ ] **Step 3: Build** the Rust image locally through the repo's image build; expect the new checks to FAIL.
- [ ] **Step 4: Implement**: `ENV RUSTUP_TOOLCHAIN=${TOOLCHAIN_VERSION}` and the auto-install switch from Step 1, and regenerate with `generate.py`.
- [ ] **Step 5: Build** again; expect every check `ok`. Run `vrg-container-run -- vrg-validate`; expect PASS.
- [ ] **Step 6: Commit** `vrg-commit --type feat --scope rust --message "pin rustup to the image's own toolchain"`, then report-ready.

### Task 4: Rust floor and toolchain-file checks, and the Rust standards page (vergil-tooling)

**Files:**
- Create: `src/vergil_tooling/lib/versions/extensions/rust.py`
- Modify: `src/vergil_tooling/lib/versions/extensions/__init__.py` (register `rust` in `FLOOR_CHECKS` and `TOOLCHAIN_CHECKS`; not in `CONTAINER_ENVS`)
- Modify: `docs/site/docs/standards/development/rust/overview.md`
- Test: `tests/vergil_tooling/test_versions_rust_ext.py` (new), `tests/vergil_tooling/test_versions_floors.py` (`test_registries`)

**Interfaces:**
- Produces: `rust.check_floor(repo_root: Path, oldest: str, _newest: str) -> list[str]`, the `FloorCheck` type, following spec §6.4's lookup order. Warning text: `Cargo.toml rust-version floor <x.y> is below the oldest supported version <oldest>. Fix: raise it to <oldest>`.
- Produces: `rust.check_toolchain(repo_root: Path) -> list[str]`, the `ToolchainCheck` type. One error per present file among `rust-toolchain.toml` and `rust-toolchain`: `<file> selects the Rust toolchain, which belongs to Vergil (the Vergil Rust image pins it). Fix: delete <file>`.

- [ ] **Step 1: Write the failing tests** in `test_versions_rust_ext.py`:
  - floor: `[package] rust-version = "1.85"` with oldest `1.99` warns; `"1.85.0"` warns with `1.85`; `"1.99"` and `"1.100"` with oldest `1.99` are silent; `"1.99"` with oldest `1.100` warns.
  - `[package] rust-version.workspace = true` with `[workspace.package] rust-version = "1.85"` warns; a virtual workspace (no `[package]`) reads `[workspace.package]`; no `rust-version` anywhere is silent; no `Cargo.toml` is silent.
  - `VersionsError` for: invalid TOML; `rust-version = 185`; `rust-version = "one"`; `workspace = true` with no `[workspace.package].rust-version`.
  - toolchain: each file alone gives one error; both give two; neither gives none.
  - `test_registries` becomes `{"python", "go", "rust"}` and `{"python", "rust"}`.
- [ ] **Step 2: Run** `vrg-container-run -- vrg-validate`; expect the new tests to fail.
- [ ] **Step 3: Implement** `rust.py` (module docstring in the style of `go.py`, citing epic #407 spec §6.4–§6.5 and this epic's spec §6.4). Compare with `_version_sort_key`. Register both hooks.
- [ ] **Step 4: Update `rust/overview.md`.** Remove the `rust-toolchain.toml` recommendations: formatting config goes in `rustfmt.toml`; toolchain pinning says Vergil selects the toolchain through the image and a repo declares only an `x.y` floor in `rust-version`. Add the rule from spec §6.7: a component or target the image lacks is added to the image through vergil-containers, never through a toolchain file.
- [ ] **Step 5: Run** `vrg-container-run -- vrg-validate`; expect PASS.
- [ ] **Step 6: Commit** `vrg-commit --type feat --scope versions --message "add the Rust floor and toolchain-file checks"`, then report-ready.

### Task 5: rustup variables never forwarded into a container (vergil-tooling)

**Files:**
- Modify: `src/vergil_tooling/lib/container.py` (`build_container_args`, the env-prefixes passthrough)
- Test: `tests/vergil_tooling/test_container.py`

**Interfaces:**
- Produces: a module constant `NEVER_FORWARDED: frozenset[str] = frozenset({"VRG_VERSION_WINDOW", "RUSTUP_TOOLCHAIN", "RUSTUP_AUTO_INSTALL"})`, written with literal names so `container.py` keeps importing `versions.window` lazily, replacing the `name != WINDOW_ENV` test in the passthrough. `env_names` and `extra_env` are unaffected.

- [ ] **Step 1: Write the failing tests.** `test_never_forwarded_names_the_window_env`: `"VRG_VERSION_WINDOW" in NEVER_FORWARDED` and it equals `versions.window.WINDOW_ENV`. `test_env_prefixes_never_forward_rustup_selection`: with `env_prefixes=["RUST"]` and host `RUST_LOG`, `RUSTUP_TOOLCHAIN`, `RUSTUP_AUTO_INSTALL` set, the args contain `-e RUST_LOG` and neither rustup name. Keep the existing `VRG_VERSION_WINDOW` test passing.
- [ ] **Step 2: Run** `vrg-container-run -- vrg-validate`; expect FAIL.
- [ ] **Step 3: Implement** the constant and the check, and update the docstring that names `VRG_VERSION_WINDOW` as the only exclusion.
- [ ] **Step 4: Run** `vrg-container-run -- vrg-validate`; expect PASS.
- [ ] **Step 5: Commit** `vrg-commit --type fix --scope container --message "never forward rustup's toolchain selection into a container"`, then report-ready.

### Task 6: Rust allowed actions drop `actions-rust-lang/*` (vergil-tooling)

**Files:**
- Modify: `src/vergil_tooling/lib/github_config.py` (`_LANGUAGE_ACTION_PATTERNS["rust"]`)
- Test: `tests/vergil_tooling/test_github_config_lib.py` (`test_desired_actions_permissions_with_language_patterns`)

- [ ] **Step 1: Change the test** to expect `["actions/*", "docker/*", "github/*", "swatinem/*", "vergil-project/*"]` for `"rust"`.
- [ ] **Step 2: Run** `vrg-container-run -- vrg-validate`; expect FAIL.
- [ ] **Step 3: Implement**: `"rust": ["swatinem/*"]`.
- [ ] **Step 4: Run** `vrg-container-run -- vrg-validate`; expect PASS. Grep `src/`, `tests/` and `docs/site/` for `actions-rust-lang` and remove any remaining mention. Completed plans (`docs/plans/completed/`) and dated specs (`docs/specs/`) are records of earlier changes and stay as they are.
- [ ] **Step 5: Commit** `vrg-commit --type fix --scope github-config --message "allow only non-toolchain actions for Rust repos"`, then report-ready.

### Task 7: Rust branch of the toolchain assertion (vergil-actions)

**Files:**
- Modify: `actions/ci/toolchain/assert/assert.sh` (new `rust)` case; header comment; the notice for other languages)
- Modify: `docs/site/docs/actions/ci-toolchain-assert.md`
- Test: `actions/ci/toolchain/assert/tests/assert.test.sh`

**Interfaces:**
- Produces: for `LANGUAGE=rust`, `assert.sh` runs `rustc --version` in the working directory, takes the `x.y` with `x_y`, and compares it with `EXPECTED`. A failed `rustc` call is `::error::assert-toolchain: cannot ask rustc for its version: <output>`. No workflow file changes.

- [ ] **Step 1: Write the failing cases** in `assert.test.sh`, faking `rustc` the way `go` is faked: `rustc 1.99.1 (abc 2026-10-01)` vs `1.99` passes; vs `1.98` fails with `CI resolved 1.98 but the job runs 1.99`; `rustc 1.100.0 (...)` vs `1.100` passes; `rustc 1.100.0` vs `1.10` fails; `rustc` exiting 1 fails with the "cannot ask rustc" error; unparseable output fails.
- [ ] **Step 2: Run** `vrg-container-run -- vrg-validate`; expect the rust cases to fail (the notice path exits 0).
- [ ] **Step 3: Implement** the `rust)` case and update the header and the notice text (`only python, go and rust are checked`).
- [ ] **Step 4: Run** `vrg-container-run -- vrg-validate`; expect PASS. Update the action's docs page.
- [ ] **Step 5: Commit** `vrg-commit --type feat --scope toolchain --message "assert the Rust leg's toolchain version"`, then report-ready.

### Task 8: Releases attested; Rust images rebuilt with the pin (deployment, .github)

Blocked by 2b, 3, 4, 5, 6, 7.

- **Precondition (human-attested):** three releases are published, and their versions are named in a comment: vergil-tooling (Tasks 4, 5 and 6), vergil-actions (Task 7) and vergil-containers (Tasks 2b and 3). Prod images build only from vergil-containers `main` (`cd.yml`), and consumers read the version window at their `[dependencies].vergil` release ref (`window.py`), so a vergil-containers release is what puts the pin in front of consumers.
- **Procedure (agent-safe):**
  1. Confirm the release notes of each named release list the tasks' PRs.
  2. Confirm the Rust images are published for each window series in the manifest: dev images built from `develop` after Tasks 2b and 3 merged, and prod images built from the named vergil-containers release.
  3. In each published Rust image, dev and prod: `RUSTUP_TOOLCHAIN` equals `RUST_VERSION`, and `(RUSTUP_TOOLCHAIN=1.0.0 rustc --version)` fails as not installed.
- **SUCCESS:** every check above passes, recorded with the release versions and image digests.

### Task 9: Rust moves to `automation: "pr"` (vergil-containers)

Blocked by 8.

**Files:**
- Modify: `docker/versions/manifest.yml` (`rust.automation`)
- Modify: `docs/site/docs/languages/rust.md` (the "Automation" section, and the "Adapter quirks" line on `rust-toolchain.toml`)
- Test: none new. How a `pr` family classifies is tested in Task 2b; this task is proven by `test_generate.test_real_manifest_is_valid` (the repo's one live-manifest check) passing with the generated files regenerated.

- [ ] **Step 1: Implement** `automation: "pr"`; regenerate with `generate.py`. In `rust.md`: "Automation" says **`pr`** (lifecycle add and drop PRs, human-merged, each carrying its toolchain pin); the quirk line says a `rust-toolchain.toml` is an error in `vrg-validate` and the image pins rustup. Leave "Current out-of-policy versions" for Task 11.
- [ ] **Step 2: Run** `vrg-container-run -- vrg-validate`; expect PASS.
- [ ] **Step 3: Commit** `vrg-commit --type feat --scope versions --message "move Rust to pr lifecycle automation"`, then report-ready.

### Task 10: Rust catch-up — lifecycle PRs merged in order (deployment, .github)

Blocked by 9.

- **Precondition (machine probe):** Task 9 is merged, and a lifecycle run since then has opened Rust add and drop PRs (list them with `vrg-gh pr list --repo vergil-project/vergil-containers --search "head:lifecycle/rust-"`).
- **Procedure:** the human merges, the agent checks. Merging is a human action.
  1. Agent: list the open Rust lifecycle PRs and give the human the order by kind: adds newest first, then drops. Never a fixed version list.
  2. Human: merge the first PR in the order.
  3. Agent: after the next lifecycle run, confirm the remaining branches were rebuilt against the new base and their CI is green; report the next PR to merge. Repeat 2–3 until none is open.
- **SUCCESS:** the manifest's rust series equal the adapter's current window; `toolchains.yml` pins exactly those series; vergil-containers#640–643 are closed; images for every window series are published with the Task 3 pin.

### Task 11: Rust policy page — out-of-policy section after the catch-up (vergil-containers)

Blocked by 10.

**Files:**
- Modify: `docs/site/docs/languages/rust.md` ("Current out-of-policy versions")

- [ ] **Step 1:** Replace the dated 1.93/1.92 text and the "manifest is behind" paragraph with the current state: under `pr` automation, versions outside the window are dropped through lifecycle PRs, so there is no out-of-policy version; the open lifecycle PRs show any pending change.
- [ ] **Step 2: Run** `vrg-container-run -- vrg-validate`; expect PASS.
- [ ] **Step 3: Commit** `vrg-commit --type docs --scope rust --message "describe Rust's versions under pr automation"`, then report-ready.

### Task 12: End-to-end check on the Rust fixtures (validation, .github)

Blocked by 10 and 11.

- **Precondition (human-attested):** a vergil-containers release cut after Task 10's catch-up and Task 11 merged is published, and its version is named in a comment. It carries the caught-up manifest's `window.json` and the prod images for the new series.
- **Precondition (machine probe):** `vergil-project/vergil-fixture-rust-lib` and `vergil-project/vergil-fixture-rust-app` exist (epic vergil-project/.github#1091) and their latest `develop` CI is green. If not, comment "blocked: preconditions not met" and stop.
- **Procedure,** on each fixture (where epic #1091's `fixture-run` recipe is available, run the checks through it):
  1. The fixture's resolved Rust window equals the `window.json` of the named release, and its CI legs run prod images whose digests match that release's published images.
  2. CI passes on both window series, and each leg's `assert-toolchain` step logs `rust <x.y> matches the resolved version <x.y>`.
  3. On a throwaway branch, add `rust-toolchain.toml` (`channel = "1.0.0"`): `vrg-container-run -- vrg-validate` fails with the Task 4 toolchain error. Delete the branch.
  4. In `vrg-container-run`, `rustc --version` reports the image's release, and `(RUSTUP_TOOLCHAIN=1.0.0 rustc --version)` fails as not installed with nothing downloaded.
  5. On a throwaway branch that adds `RUST` to the fixture's `[container].env-prefixes`, with `RUSTUP_TOOLCHAIN=1.0.0` exported on the host, `vrg-container-run -- rustc --version` still reports the image's release. Delete the branch.
- **SUCCESS:** every step passes on both fixtures, recorded per fixture.
