# TypeScript lifecycle PRs and toolchain enforcement — implementation plan

- **Epic:** vergil-project/.github#1281
- **Planning issue:** vergil-project/.github#1208 (review history is in its comments)
- **Status:** in design review

> **For agentic workers:** each task is one issue and one PR in the repo named
> in its heading, worked with `issue-implement`; operational tasks are worked
> with `issue-deploy` or `issue-validate`. Steps use checkbox (`- [ ]`)
> syntax for tracking.

**Goal:** Node majors arrive through lifecycle PRs, the window catches up to
26/24/22, and a TypeScript repo can neither declare a Node or package-manager
version of its own nor run a CI leg on a Node major CI did not resolve.

**Architecture:** two independent tracks. Track A flips `ts-node` to
`automation: "pr"` in vergil-containers and merges the Node 26 add PR the
lifecycle job opens. Track B adds a TypeScript resolver extension to
vergil-tooling (toolchain-setting errors and a two-sided `engines.node`
warning), withholds Node and npm host variables from `vrg-container-run` with
a notice, and adds a TypeScript branch to vergil-actions' toolchain
assertion. Fixture runs on both TypeScript fixtures prove the two together.

**Tech Stack:** Python 3 (vergil-tooling, vergil-containers `docker/`
scripts), pytest, bash (`assert.sh`), npm semver range syntax.

**Spec:** [`spec.md`](spec.md) in this directory. Executors read both.

## Global Constraints

- Consumers deal only in the Node major; no consumer file, test or doc names a Node release (spec §1).
- The shared resolver has no language branches: every TypeScript rule lives in `src/vergil_tooling/lib/versions/extensions/typescript.py` (spec §8).
- The shared lifecycle code stays language-neutral: the `ts-node` switch is a manifest change, with no Node branch in `apply.py` or `publish.py` (spec §6.1, §8).
- Fail loudly: an unreadable `package.json`, `engines`, `engines.node` or `devEngines`, a range that admits nothing, and a failed `node` call are errors, never skipped (spec §7, §8).
- Lifecycle PRs stay human-merged; releases are human-gated; no agent cuts a release (spec §8).
- **Release lock.** Before the first PR merges whose change must not be released until other tasks are done, the agent stops, tells the human loudly, and waits for an explicit go-ahead. In this plan that PR is the first of Tasks 6 and 8 to be handed off for vergil-tooling's `develop`. From that merge until Task 10's fleet re-check has posted, the human holds every vergil-tooling release, including those of other epics running in parallel, which pause their own releases until this plan's conditions are met. The human is the lock; no tooling enforces it.
- Validation runs only through `vrg-container-run -- vrg-validate` in the task repo's worktree.
- Commit with `vrg-commit`; hand off with `vrg-pr-workflow report-ready`.

## Review Focus

1. **`vergil-fixture-ts-app`'s real `engines.node: ">=24"` against the window 26/24/22.** No warning: the floor is above the oldest and the range admits 26. Pinned in Task 7.
2. **`engines.node: "^22 || ^24"`.** One warning only, the upper bound (it excludes 26); no floor warning. Pinned in Task 7.
3. **`devEngines` as an array whose only entry is `{"os": …}`, or an object with only `cpu`.** No error. Pinned in Task 6.
4. **A `package.json` that is a JSON array, or not JSON, run through `vrg-validate`.** An `ERROR:` line naming the file, a failing exit, and no traceback. Pinned in Task 6.
5. **`env-prefixes = ["NODE"]` with `NODE_EXTRA_CA_CERTS` and `NODE_OPTIONS` set on the host.** The first is forwarded; the second is withheld with its notice. Pinned in Task 8.

---

## Delivery table

| Task | Repo | Kind | What | Blocked by |
|---|---|---|---|---|
| 1 | .github | task (#1282) | Spec and plan (this PR) | — |
| 2 | vergil-containers | task | `ts-node` moves to `automation: "pr"`, with its automation docs | — |
| 3 | .github | deployment | Node 26 catch-up: the lifecycle add PR merged | 2 |
| 4 | vergil-containers | task | ts-node policy page: dated window facts after the catch-up | 3 |
| 5 | .github | deployment | vergil-containers release with the 26/24/22 window attested | 4 |
| 6 | vergil-tooling | task | TypeScript toolchain-setting errors, and the standards page | — |
| 7 | vergil-tooling | task | Two-sided `engines.node` floor warning | 6 |
| 8 | vergil-tooling | task | Node and npm host variables withheld from containers, with notices | — |
| 9 | vergil-actions | task | TypeScript branch of the toolchain assertion | — |
| 10 | .github | deployment | Fleet re-check, then vergil-tooling and vergil-actions releases attested | 6, 7, 8, 9 |
| FR | the fixtures | fixture run | `ts-lib`, `ts-app` against the released enforcement and window | 5, 10 |
| — | vergil-tooling | bookend (#4638) | Documentation review | 2, 4, 6, 7, 8, 9 |
| — | .github | bookend (#1283) | Follow-on brainstorm: pick up epic 3 (#1209) | FR |
| — | .github | bookend (#1284) | Retrospective | all |

Tracks A (Tasks 2–5) and B (Tasks 6–10) share nothing until the fixture run.

**Filing.** Tasks 2–10 are filed at epic-create's filing step, each in its
repo and linked under #1281, with its `Blocked-by:` refs. The fixture run is
filed later, once Tasks 5 and 10 have closed with their release versions
recorded (below).

---

### Task 2: `ts-node` moves to `automation: "pr"` (vergil-containers)

**Files:**

- Modify: `docker/versions/manifest.yml` (`ts-node.automation`)
- Modify: the files `uv run docker/versions/generate.py` regenerates
- Modify: `docs/site/docs/languages/ts-node.md` ("Automation" section only)
- Modify: `docs/site/docs/languages/lifecycle.md` (the per-family automation table, around line 184; the "`pr` families" sentence, around line 45; and the sentence saying pull-request CI does not build images)
- Test: `docker/versions/test_generate.py` (`test_real_manifest_is_valid`)

This is a configuration task, exempt from test-first: it changes one
manifest value and docs, and the repo's existing checks prove it.

How a `pr` family classifies, and how an add PR writes a `ts-node` pin, is
already tested by the shared lifecycle suite (epic #1095 Task 2b's
`test_unresolvable_pin_is_apply_error` covers a ts-node add). This task adds
no lifecycle code.

- [ ] **Step 1: Implement** `automation: "pr"` for `ts-node`; regenerate with `uv run docker/versions/generate.py`. In `ts-node.md`, "Automation" says **`pr`**: the lifecycle job opens add and drop PRs, human-merged, each carrying its toolchain pin. Leave "Current out-of-policy versions" for Task 4. In `lifecycle.md`, the table row for `ts-node` says `pr`, the "`pr` families" sentence names TypeScript beside Python, Go and Rust, and the sentence on pull-request CI says what `ci-images.yml`'s `images` job does today: PR CI builds, scans and smoke-tests the images a change affects.
- [ ] **Step 2: Run** `vrg-container-run -- vrg-validate`; expect PASS.
- [ ] **Step 3: Commit** `vrg-commit --type feat --scope versions --message "move ts-node to pr lifecycle automation"`, then report-ready.

### Task 3: Node 26 catch-up — the lifecycle add PR merged (deployment, .github)

Blocked by 2.

- **Precondition (machine probe):** Task 2 is merged, and a lifecycle run since then has opened the Node 26 add PR: `vrg-gh pr list --repo vergil-project/vergil-containers --search "head:lifecycle/ts-node-"` lists it.
- **Procedure:** the human merges; the agent checks. Merging is a human action.
  1. Agent: confirm the PR adds only `ts-node` 26: the manifest entry, its `toolchains.yml` pin, and the regenerated files. Confirm its CI is green, including the ts-node 26 image build, `checks.sh` and the smoke test. Report it ready to merge.
     - **If its CI is red:** report the failing job and its cause. This is the first run of the ts-node smoke project (repo-declared TypeScript 6, ESLint 10 and Vitest 5, installed with `npm ci`) on Node 26 and npm 11. Propose the fix to the human and, on their ruling, file it as a vergil-containers task linked under #1281, and record it here as `Blocked-by:`. This task stays open until the fix has merged and the lifecycle job's next run has rebuilt the PR green.
  2. Human: merge it.
  3. Agent: confirm `dev-ts-node` images for 26 are published from `develop`, and that vergil-project/vergil-containers#730 is closed (the job stops reporting the add).
  4. Agent: post the add PR's vergil-vm follow-up line as a comment on epic 4's planning item, vergil-project/.github#959, where lining vergil-vm up with the window belongs (spec §6.1).
- **SUCCESS:** the manifest's `ts-node` majors are 26, 24, 22; `toolchains.yml` pins exactly those plus the core Node; dev images for 26 exist.

### Task 4: ts-node policy page — dated window facts after the catch-up (vergil-containers)

Blocked by 3.

**Files:**

- Modify: `docs/site/docs/languages/ts-node.md` (the header's "checked on 2026-10-21" date, the "Support rule" paragraph that says "On 2026-10-21 that is 24 and 22", the "Adapter quirks" line on vergil-vm's Node pin, and "Current out-of-policy versions")

- [ ] **Step 1:** The support-rule paragraph gives the window as 26, 24 and 22, dated the day Task 3 closed. "Current out-of-policy versions" says that under `pr` automation a major outside the window is dropped through a lifecycle PR, so there is none, and that the open lifecycle PRs show any pending change. Remove the "manifest is behind until 26 is built" paragraph. The header's "checked on" date becomes the day the upstream facts are re-checked for this edit. The vergil-vm line says lining vergil-vm's Node up with the window is epic 4's work (vergil-project/.github#959).
- [ ] **Step 2: Run** `vrg-container-run -- vrg-validate`; expect PASS.
- [ ] **Step 3: Commit** `vrg-commit --type docs --scope ts-node --message "describe the ts-node window under pr automation"`, then report-ready.

### Task 5: vergil-containers release with the 26/24/22 window (deployment, .github)

Blocked by 4.

- **Precondition (human-attested):** a vergil-containers release cut after Tasks 3 and 4 merged is published, and its version is named in a comment. Prod images build only from `main`, and consumers read the window at their `[dependencies].vergil` release ref, so this release is what puts Node 26 in front of consumers.
- **Procedure (agent-safe):**
  1. Confirm the release notes list Task 2's PR, the Node 26 add PR and Task 4's PR.
  2. Confirm the release's `docker/versions/window.json` lists `ts-node` 26, 24 and 22.
  3. Confirm `prod-ts-node` images for 26, 24 and 22 are published from that release. The images carry no Vergil release label yet (vergil-project/.github#1287), so tie them to the release this way: find the prod publish run the release triggered, check that its commit is the release tag's, and record the digests it pushed for 26, 24 and 22. Prod tags are rebuilt nightly from the release commit, so a current tag digest rebuilt from that commit also counts as a match (the ruling recorded in epic #1095's plan).
- **SUCCESS:** every check passes, recorded with the release version and image digests.

### Task 6: TypeScript toolchain-setting errors, and the standards page (vergil-tooling)

**Files:**

- Create: `src/vergil_tooling/lib/versions/extensions/typescript.py`
- Modify: `src/vergil_tooling/lib/versions/extensions/__init__.py` (register `typescript` in `TOOLCHAIN_CHECKS`; not in `CONTAINER_ENVS`)
- Modify: `src/vergil_tooling/lib/versions/floors.py` (the module docstring's list of toolchain-selection files gains TypeScript's)
- Modify: `docs/site/docs/standards/development/typescript/toolchain-and-strictness.md`
- Modify: `docs/site/docs/guides/validation-matrix.md` (the toolchain-selection checks, beside Python's and Rust's)
- Test: `tests/vergil_tooling/test_versions_typescript_ext.py` (new), `tests/vergil_tooling/test_versions_floors.py` (`test_registries`), `tests/vergil_tooling/test_vrg_validate.py`

**Interfaces:**

- Consumes: `vergil_tooling.lib.container.read_package_json(path: Path) -> dict[str, object]` and `PackageJsonError`. Import them inside `_package_json` if a module-level import would cycle (`container.py` already imports `versions` lazily).
- Produces: `typescript._package_json(repo_root: Path) -> dict[str, object] | None`: the root `package.json`, or `None` when absent. A `PackageJsonError` is re-raised as `VersionsError` with the same message. Task 7 reads the file through it.
- Produces: `typescript.check_toolchain(repo_root: Path) -> list[str]`, the `ToolchainCheck` type: one error per setting in spec §6.2's list, in this order: `.nvmrc`, `.node-version`, `packageManager`, `volta`, `engines.npm`, `engines.pnpm`, `engines.yarn`, `devEngines.runtime`, `devEngines.packageManager`. Each names the setting, says the Node runtime (or the package manager) belongs to Vergil, and ends with the fix: `Fix: delete <file>` for the two files, `Fix: remove the "<field>" field` for `packageManager` and `volta`, `Fix: remove engines.<key>` and `Fix: remove devEngines.<entry>` for the keys. The shape follows `python.check_toolchain`'s message.
- `devEngines` accepts npm's two forms for each entry: an object, or an array of objects. `devEngines` or `engines` not an object, or a `devEngines` array entry not an object, is a `VersionsError` naming the field.

- [ ] **Step 0: Confirm `devEngines`' behaviour** (spec §6.2) against npm's documentation for the npm bundled with each window major (npm 10 for Node 22, npm 11 for 24 and 26): that `npm install` checks `devEngines.runtime` and `devEngines.packageManager`, the forms each takes, and the default `onFail`. Record the URLs in the PR body. If the behaviour differs from the spec's paragraph, the error stays, and the spec is corrected in a separate `.github` PR.
- [ ] **Step 1: Write the failing tests** in `test_versions_typescript_ext.py`:
  - `test_each_setting_is_one_error`: parametrized over the nine settings above, each alone in a fixture repo, gives exactly one error ending in its fix text.
  - `test_package_manager_naming_npm_is_an_error`: `"packageManager": "npm@11.6.0"` gives the error.
  - `test_dev_engines_allowed_entries`: `devEngines` with only `os`, only `cpu` and only `libc`, each as an object and as an array, gives no error (Review Focus 3).
  - `test_dev_engines_runtime_array_form`: `devEngines.runtime` as `[{"name": "node", "version": "24"}]` gives the error.
  - `test_engines_node_alone_is_not_an_error`: `engines: {"node": ">=24"}` gives no error.
  - `test_malformed_fields_are_versions_errors`: `engines: "x"`, `devEngines: 1`, `devEngines: {"runtime": [1]}` each raise `VersionsError` naming the field.
  - `test_malformed_package_json_is_versions_error`: a `package.json` holding `[]`, and one holding `{`, each raise `VersionsError` whose message is `PackageJsonError`'s.
  - `test_no_package_json_no_files`: an empty repo gives no error.
  - `test_registries` in `test_versions_floors.py`: `TOOLCHAIN_CHECKS` keys become `{"python", "rust", "typescript"}`.
  - In `test_vrg_validate.py`, `test_typescript_malformed_package_json_reports_error`: a TypeScript repo with `package.json` holding `[]` makes the version step print `ERROR:` with the file's path, fail, and print no `Traceback` (Review Focus 4). The test asserts that the error comes from the version step: no later stage has run when it is printed. A later stage that reads `package.json` may already report the file today, so a test that only looked for `ERROR:` could pass before this task's code exists.
- [ ] **Step 2: Run** `vrg-container-run -- vrg-validate`; expect the new tests to fail.
- [ ] **Step 3: Implement** `typescript.py`: a module docstring in `python.py`'s style stating the rule (the Node runtime belongs to Vergil, as Python's does; epic #407 spec §6.5, this epic's spec §6.2), `_package_json` and `check_toolchain`. Register the hook.
- [ ] **Step 4: Update `toolchain-and-strictness.md`:** Vergil selects the Node runtime through the image; a repo declares at most an `engines.node` range, which should admit every window major; and every setting `check_toolchain` reports is an error, listed as spec §6.2 lists it, with its fix.
- [ ] **Step 4b: Update `validation-matrix.md`:** its toolchain-selection paragraph (beside Python's `.python-version` and Rust's toolchain files) names TypeScript's settings and links to `toolchain-and-strictness.md`.
- [ ] **Step 5: Run** `vrg-container-run -- vrg-validate`; expect PASS.
- [ ] **Step 6: Commit** `vrg-commit --type feat --scope versions --message "report TypeScript settings that select Node or the package manager"`.
- [ ] **Step 7: Release lock** (Global Constraints). If Task 8 has not yet been handed off, stop before report-ready: tell the human loudly that merging this PR starts the vergil-tooling release hold, and wait for an explicit go-ahead. Then report-ready, with the hold stated at the top of the notes.

### Task 7: Two-sided `engines.node` floor warning (vergil-tooling)

Blocked by 6.

**Files:**

- Modify: `src/vergil_tooling/lib/versions/extensions/typescript.py` (add `check_floor` and the range reader)
- Modify: `src/vergil_tooling/lib/versions/extensions/__init__.py` (register `typescript` in `FLOOR_CHECKS`)
- Modify: `pyproject.toml` and `uv.lock`, only if Step 1 picks a library
- Modify: `docs/site/docs/standards/development/typescript/toolchain-and-strictness.md` (the floor and upper-bound rule)
- Modify: `docs/site/docs/guides/validation-matrix.md` (the floor sentence that lists `requires-python`, the `go` directive and `rust-version`)
- Modify: `src/vergil_tooling/lib/versions/floors.py` (the module docstring's list of floors gains `engines.node`)
- Test: `tests/vergil_tooling/test_versions_typescript_ext.py`, `tests/vergil_tooling/test_versions_floors.py` (`test_registries`)

**Interfaces:**

- Consumes: Task 6's `_package_json(repo_root)`.
- Produces: `typescript.check_floor(repo_root: Path, oldest: str, newest: str) -> list[str]`, the `FloorCheck` type. `oldest` and `newest` are bare majors (`"22"`, `"26"`), as `floors.check_floors` passes them. Warning texts, exactly as spec §6.2 gives them:
  - `engines.node <range> admits Node <major>, below the oldest supported major <oldest>. Fix: raise it to >=<oldest>`
  - `engines.node <range> excludes Node <newest>. Fix: drop the upper bound, or widen it to admit <newest>`
- Produces (private): `_NodeRange` with `lowest() -> tuple[int, int, int] | None` (the lowest non-prerelease version admitted, `None` when none) and `admits_major(major: int) -> bool`; and `_parse_range(text: str) -> _NodeRange`, raising `VersionsError` on anything it cannot read.

- [ ] **Step 1: Choose the parser** (spec §10). Check PyPI for a maintained Python implementation of npm's range grammar (<https://github.com/npm/node-semver#ranges>): a release within the last two years, a licence vergil-tooling's licence check accepts, support for Python 3.15 (vergil-tooling's floor; its `requires-python` does not exclude it), and a pure-Python wheel. Confirm it with `uv lock` before choosing it. If one exists, wrap it behind `_parse_range`; if not, implement the reader in `typescript.py`. Record the choice and why in the PR body.
- [ ] **Step 2: Write the failing tests** in `test_versions_typescript_ext.py`, window 26/24/22 (`oldest="22"`, `newest="26"`):
  - `test_floor_and_upper_bound`, parametrized on `(range, expected warnings)`: `">=24"` none (Review Focus 1); `">=22"` none; `">=20"` floor 20; `"^22 || ^24"` upper bound only (Review Focus 2); `"^20 || ^24"` floor 20 and upper bound; `"~24.1"` upper bound; `"22.x"` upper bound; `"20 - 26"` floor 20; `">=22.12.0 <27"` none; `"*"` floor 0; `"<27"` floor 0; `">=24 <26"` upper bound.
  - `test_no_engines_node_is_silent`: no `engines`, and `engines` with only `os`, give nothing.
  - `test_range_admitting_nothing_is_an_error`: `">=26 <24"` raises `VersionsError` with `admits no Node release`.
  - `test_invalid_range_is_an_error`: `"latest"`, `">=x"`, `"24 ||| 26"` and `engines.node: 24` (a number) each raise `VersionsError` naming the value.
  - `test_prerelease_ignored`: `">=26.0.0-rc.1"` gives no floor warning and admits 26.
  - `test_registries`: `FLOOR_CHECKS` keys become `{"python", "go", "rust", "typescript"}`.
- [ ] **Step 3: Run** `vrg-container-run -- vrg-validate`; expect the new tests to fail.
- [ ] **Step 4: Implement** `_parse_range` and `check_floor`. If hand-written, the reader turns each `||` alternative into one interval (desugaring `x`-ranges, `~`, `^` and hyphen ranges as node-semver's README defines them), drops an alternative whose interval is empty, and raises when every one is empty. `lowest()` is the smallest lower bound; `admits_major(n)` is whether any interval meets `[n.0.0, (n+1).0.0)`. The floor test compares `lowest()`'s major with `int(oldest)`; the upper-bound test is `admits_major(int(newest))`. Register the hook.
- [ ] **Step 5: Update `toolchain-and-strictness.md`** with the two warnings and the two errors (an unreadable range, a range that admits nothing). Add `engines.node` to `validation-matrix.md`'s floor sentence and to `floors.py`'s docstring.
- [ ] **Step 6: Run** `vrg-container-run -- vrg-validate`; expect PASS.
- [ ] **Step 7: Commit** `vrg-commit --type feat --scope versions --message "warn on an engines.node range outside the Node window"`, then report-ready.

### Task 8: Node and npm host variables withheld from containers, with notices (vergil-tooling)

**Files:**

- Modify: `src/vergil_tooling/lib/container.py` (`NEVER_FORWARDED`, new `NEVER_FORWARDED_PREFIXES` and `withheld_reason`, the passthrough loop in `build_container_args`, its docstring and the module comment)
- Modify: `docs/site/docs/reference/container-config.md` (the `env-prefixes` entry)
- Test: `tests/vergil_tooling/test_container.py`

**Interfaces:**

- Produces: `NEVER_FORWARDED` gains `"NODE_OPTIONS"`.
- Produces: `NEVER_FORWARDED_PREFIXES: tuple[str, ...] = ("npm_config_", "COREPACK_", "VOLTA_")`, each matched without regard to case (npm reads its environment configuration case-insensitively).
- Produces: `withheld_reason(name: str) -> str | None`: `None` for a name the passthrough may forward, otherwise the reason. Reasons: `RUSTUP_TOOLCHAIN` and `RUSTUP_AUTO_INSTALL`, "it would override the image's Rust toolchain"; `VRG_VERSION_WINDOW`, "it names a host path the container cannot see"; `NODE_OPTIONS` and the prefixes, "it would change how Node or npm behaves inside the container".
- The passthrough loop forwards a name that matches the repo's `env-prefixes` only when `withheld_reason(name)` is `None`; otherwise it prints `vrg-container-run: not forwarding <NAME> from the host: <reason>` on stderr, once per name. `NODE_PATH` handling, `env_names` and `extra_env` are unchanged.

- [ ] **Step 1: Write the failing tests** in `test_container.py`:
  - `test_env_prefixes_withhold_node_and_npm`: `env_prefixes=["NODE", "npm", "NPM", "COREPACK", "VOLTA"]`, host `NODE_ENV`, `NODE_OPTIONS`, `npm_config_registry`, `NPM_CONFIG_REGISTRY`, `COREPACK_HOME`, `VOLTA_HOME` set: the args carry `-e NODE_ENV` and none of the others.
  - `test_withheld_variable_prints_a_notice`: the same run prints one `not forwarding <NAME>` line per withheld name on stderr (capsys), with its reason.
  - `test_other_node_variables_still_forwarded`: `env_prefixes=["NODE"]`, host `NODE_EXTRA_CA_CERTS` and `NODE_OPTIONS`: the first forwarded, the second withheld with its notice (Review Focus 5).
  - `test_rustup_withheld_with_notice`: the existing rustup case also prints its notice.
  - `test_node_path_unchanged`: with a build command declared and host `NODE_PATH` set, `-e NODE_PATH=<host value>` is still in the args.
- [ ] **Step 2: Run** `vrg-container-run -- vrg-validate`; expect FAIL.
- [ ] **Step 3: Implement** the constants, `withheld_reason` and the loop change.
- [ ] **Step 4: Update `container-config.md`:** the `env-prefixes` entry lists the names and prefixes never forwarded, why, and the notice.
- [ ] **Step 5: Run** `vrg-container-run -- vrg-validate`; expect PASS.
- [ ] **Step 6: Commit** `vrg-commit --type fix --scope container --message "withhold Node and npm host settings from containers, with a notice"`.
- [ ] **Step 7: Release lock** (Global Constraints). If Task 6 has not yet been handed off, stop before report-ready: tell the human loudly that merging this PR starts the vergil-tooling release hold, and wait for an explicit go-ahead. Then report-ready, with the hold stated at the top of the notes.

### Task 9: TypeScript branch of the toolchain assertion (vergil-actions)

**Files:**

- Modify: `actions/ci/toolchain/assert/assert.sh` (new `typescript)` case; the header comment; the notice for other languages)
- Modify: `actions/ci/toolchain/assert/action.yml` (`description` and the `expected` input's text)
- Modify: `docs/site/docs/actions/ci-toolchain-assert.md`, `docs/site/docs/actions/index.md`
- Test: `actions/ci/toolchain/assert/tests/assert.test.sh`

**Interfaces:**

- Produces: for `LANGUAGE=typescript`, `assert.sh` strips a leading `node-` from `EXPECTED` and requires `^[0-9]+$` of the rest (else `::error::assert-toolchain: expected must be a Node major for typescript (got '<value>')`); asks Node once, in the working directory, with `node -p 'process.versions.node + "\n" + process.execPath'`, as the Python branch asks its interpreter once for two lines; takes the major from line 1 (`26.1.0` → `26`) and logs line 2 as `assert-toolchain: the job's Node is <path>`. A failed `node` call is `::error::assert-toolchain: cannot ask node for its version: <output>`; an answer whose line 1 has no major, or with no line 2, is an error naming it. The comparison and its mismatch message are the shared ones. No workflow file changes.

- [ ] **Step 1: Write the failing cases** in `assert.test.sh`, faking `node` the way `go` is faked, with the one answer `26.1.0` and `/opt/node/26/bin/node` on two lines: `node-26` passes and logs the path; `26` passes; `node-24` fails with `CI resolved 24 but the job runs 26`; `node-x`, `24.1` and `v26` fail with the expected-format error; `node` exiting 1 fails with the "cannot ask node" error; an answer of `garbage` fails; an answer of `26.1.0` alone (no path line) fails.
- [ ] **Step 2: Run** `vrg-container-run -- vrg-validate`; expect the typescript cases to fail (the notice path exits 0).
- [ ] **Step 3: Implement** the `typescript)` case, the header comment's line, and the notice text (`only python, go, rust and typescript are checked`). Update `action.yml`.
- [ ] **Step 4: Update** `ci-toolchain-assert.md` and the actions index for TypeScript.
- [ ] **Step 5: Run** `vrg-container-run -- vrg-validate`; expect PASS.
- [ ] **Step 6: Commit** `vrg-commit --type feat --scope toolchain --message "assert the TypeScript leg's Node major"`, then report-ready.

### Task 10: Fleet re-check, then vergil-tooling and vergil-actions releases (deployment, .github)

Blocked by 6, 7, 8, 9.

- **Precondition (machine probe):** Tasks 6–9 are merged.
- **Procedure:**
  1. Agent, before any release: list the org's repos whose `vergil.toml` has `primary-language = "typescript"` (`vrg-gh search code` across the org, plus the fixtures by name). In each, check for the settings Task 6 reports and run Task 7's floor test against the window by reading `package.json`. Then list every repo whose `[container].env-prefixes` matches a name or prefix Task 8 withholds. Post the findings as a comment. If any repo would newly fail, stop and ask the human before the release: the fix for that repo is a ruling, not part of this task.
  2. Human-run step: publish releases of vergil-tooling (Tasks 6, 7, 8) and vergil-actions (Task 9), and name their versions in a comment.
  3. Agent: confirm each release's notes list its tasks' PRs.
- **SUCCESS:** the re-check is posted with no unresolved repo, and both releases are confirmed, recorded with their versions.

### Fixture run (FR)

Filed with the `fixture-run` skill once Tasks 5 and 10 have closed with their
release versions recorded, since `--validates` names those versions and
drives the validation's deployed-version probe. The plan's seeding is the
approval it files under.

- `fixture-run vergil-project/.github#1281 ts-lib,ts-app --validates "vergil-tooling <v> and vergil-actions <v> (TypeScript toolchain enforcement), vergil-containers <v> (ts-node window 26/24/22)" --after <Task 5> --after <Task 10>`. The versions are those Tasks 5 and 10 record.
- After filing, post on each validation task the checks the recipe's templates do not carry (spec §6.6):
  - CI passes: the library on 26, 24 and 22, the app on 26 alone;
  - each leg's `assert-toolchain` log reports the leg's major and `/opt/node/<major>/bin/node`;
  - on a scratch branch, adding `.nvmrc` fails `vrg-validate` with Task 6's error, and `engines.node: "^24"` gives Task 7's upper-bound warning;
  - on a scratch branch whose `[container].env-prefixes` includes `npm`, with `npm_config_registry` set on the host, `vrg-container-run -- env` does not show it and prints Task 8's notice.

  Scratch branches are deleted, never merged.
- Epic vergil-project/.github#1218's runs share both fixtures. The human sequences the two epics' runs by hand.
