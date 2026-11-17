# TypeScript through the full cycle — implementation plan

- **Epic:** vergil-project/.github#1174
- **Planning issue:** vergil-project/.github#1161 (review history is in its comments)
- **Status:** approved

> **For agentic workers:** each task is one issue and one PR in the repo named
> in its heading, worked with `issue-implement`; operational tasks are worked
> with `issue-deploy` or `issue-validate`, and fixture runs are filed with
> `fixture-run`. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** a TypeScript library and a TypeScript app each run Vergil's full
cycle (change → PR → merge → release → validate) on Node, with no version
pin, tools sourced by the cross-language guideline, managed shared configs,
a release path, evidence files and a BUILD check.

**Architecture:** vergil-tooling carries almost all of it: the resolver's pin
rule, detection, the guideline page, a language-neutral BUILD check, the
TypeScript tool and entry-point checks, three managed files in the fleet
sync, `vrg-version` support, and the TypeScript commands and evidence.
vergil-actions adds the BUILD job, lets `cd-release` release TypeScript and
stages its reports. vergil-claude-plugin learns the TypeScript fixtures. Two
new fixture repos prove it, and only then does vergil-containers drop the
image's repo-side tools, proven by a second fixture run.

**Tech Stack:** Python 3 (vergil-tooling), pytest, GitHub Actions YAML and
bash (vergil-actions), Dockerfile (vergil-containers), Markdown skills
(vergil-claude-plugin), TypeScript 6, Node 24 and 22, npm, ESLint,
typescript-eslint, Prettier, Vitest.

**Spec:** [`spec.md`](spec.md) in this directory. Executors read both.

## Global Constraints

- TypeScript repos stay on TypeScript 6.x; TypeScript 7 is deferred to vergil-project/.github#1181 (spec §3).
- Every `npm ci` Vergil runs for TypeScript (INSTALL, BUILD, the release build) passes `--ignore-scripts` (spec §6.3).
- A gate never runs a global binary in place of a repo-declared tool, and never downloads one (spec §6.3, §7).
- Fail loudly with a message that names the fix: no fallback, no "not TypeScript" for a broken `package.json` (spec §7, §8).
- The BUILD check is language-neutral in code. This epic turns it on for TypeScript; Rust's is #1165 Task 13, and Python's is a planning item (spec §6.5, §6.10).
- Weakening a Vergil rule is a #375 exception and is not blocked here; omitting Vergil's base is (spec §6.4).
- No npm publishing: `publish_cmd` stays `None`, `registry-publish` stays `false` (spec §6.5).
- No agent cuts a release; releases and fixture repo creation are human steps (spec §8).
- The image keeps its repo-side TypeScript tools until the first fixture run passes (spec §6.9).
- Validation runs only through `vrg-container-run -- vrg-validate` in the task repo's worktree. Commit with `vrg-commit`; hand off with `vrg-pr-workflow report-ready`.

## Review Focus

1. **A `tsconfig.json` with comments, a trailing comma and `"extends": ["./tsconfig.vergil.json", "@tsconfig/node24/tsconfig.json"]`.** It passes the entry-point check. Pinned in Task 8b.
2. **A `package.json` declaring `"typescript": "latest"`, `"*"` or `">=5"`.** Each fails with the fix `npm install --save-dev typescript@^6`, as `^5.9.0` does. Pinned in Task 7.
3. **A `package.json` that carries a `"prettier"` key.** `vrg-validate` fails with the single-source error, even though `.prettierrc.json` is present. Pinned in Task 8b.
4. **A version bump on a `package.json` with 4-space indentation and no trailing newline, and a lockfile with `lockfileVersion: 3`.** Only the version values change; indentation, key order and the final newline are kept byte for byte. Pinned in Task 9.
5. **`npm ci --ignore-scripts` on a project that uses Vitest** (esbuild's platform binary arrives as an optional dependency, not through its install script). TEST still runs. Pinned in Task 7's command tests and proven by Run 1.

---

## Delivery table

| Task | Repo | Kind | What | Blocked by |
|---|---|---|---|---|
| 1 | .github | task (#1175) | Spec and plan (this PR) | — |
| 2 | vergil-tooling | task | A pin is required only for a multi-family window | — |
| 3 | vergil-tooling | task | A malformed `package.json` is an error | — |
| 4 | vergil-tooling | task | The tool-ownership guideline page | — |
| 5 | vergil-tooling | task | A language-neutral BUILD check kind | — |
| 6 | vergil-actions | task | The `build` job in `ci-quality.yml`, gated on vergil-tooling's BUILD flag | 5; #1165 Task 8 |
| 7 | vergil-tooling | task | TypeScript tools from the repo, with accepted ranges | 3 |
| 8a | vergil-tooling | task | Three managed TypeScript files in the fleet sync | — |
| 8b | vergil-tooling | task | Entry-point and single-Prettier checks; LINT reads the managed configs | 7, 8a |
| 9 | vergil-tooling | task | `vrg-version` supports `package.json`; a version mismatch fails on the PR | 7; #1165 Task 5 |
| 10 | vergil-tooling | task | TypeScript evidence files, ignored by the fragment | 7 |
| 11 | vergil-tooling | task | TypeScript build command, BUILD on for TypeScript, `build` script check | 5, 7 |
| 12a | vergil-actions | task | `cd-release` releases TypeScript | — |
| 12b | vergil-actions | task | The TypeScript row in the report contract | 10; #1165 Task 8 |
| 13 | vergil-claude-plugin | task | `fixture-run` learns the TypeScript fixtures; per-gap known-gap records | — |
| 14 | .github | deployment | Tooling, actions and plugin releases attested; plugin deployed | 2, 3, 4, 5, 6, 7, 8a, 8b, 9, 10, 11, 12a, 12b, 13 |
| 15 | .github | deployment | The two TypeScript fixture repos created | 14 |
| 16a | vergil-fixture-ts-lib | task | The library program | 15 |
| 16b | vergil-fixture-ts-app | task | The command-line app | 15 |
| Run 1 | fixtures | fixture run | Both fixtures through the full cycle, `v0.1.0` | 14; each fixture's change task also on its own program (16a or 16b) |
| 17 | vergil-containers | task | The image drops the repo-side TypeScript tools | Run 1 validations |
| 18 | .github | deployment | Containers release attested; ts-node images rebuilt | 17 |
| Run 2 | fixtures | fixture run | Both fixtures again, on images without the globals | 18 |
| — | vergil-tooling | bookend (#4451) | Documentation review | 2–13, 17 |
| — | .github | bookend (#1176) | Follow-on brainstorm: pick up epic 2 | Run 2 |
| — | .github | bookend (#1177) | Retrospective | all |

Tasks 2 and 3 are independent of everything else and land first, so the pin
rule and the detection fix ship without waiting on the fixtures.

**Filing.** At epic-create's filing step:

- Tasks 2–15, 17 and 18 are filed, each in its repo, linked under #1174, with
  its `Blocked-by:` refs. Task 17 is also blocked by Run 1's validation tasks,
  added once Run 1 is filed.
- Tasks 16a and 16b are filed by Task 15, once the repos exist.
- Run 1 is filed with `fixture-run` once Tasks 14, 16a and 16b exist, with
  `--after` Task 14 only, since `fixture-run` accepts only deployment tasks as
  `--after`. Then the ts-lib change task gets `Blocked-by:` Task 16a and the
  ts-app change task `Blocked-by:` Task 16b, so each run waits only on its own
  program. Run 2 is filed once Task 18 exists (see "Fixture runs").
- The cross-epic refs in "Coordination with #1165" are stamped: Tasks 6, 9 and
  12b get `Blocked-by:` their #1165 tasks, and #1165's Task 13 gets
  `Blocked-by:` this epic's Task 5. If a #1165 task is not filed yet, stop and
  ask for its ref.
- Four planning items are filed under the planning epic (spec §6.10): epic 2
  (TypeScript lifecycle and enforcement), epic 3 (TypeScript supply chain and
  the npm updater), the Python tool migration, and turning on the BUILD check
  for Python. Epic 4 is vergil-project/.github#959.
- One intake item is filed (spec §6.10): removing `bump()`'s and
  `bump_paths()`'s swallowed errors for every language, by making "no version
  file" an explicit case, reconciled with #1165's decision to keep them.
- Already recorded, so not filed again: the image-tool-versions comment on
  vergil-tooling#4416 and the effective-config comment on epic #375. Check that
  #4416 is still where the report contract's tool list is designed, and move
  that record if not.

### Coordination with #1165

Epic vergil-project/.github#1165 (Rust validation, supply-chain and
dependency-update parity) edits the same code. Its plan is merged, so its
overlapping tasks go first, except where it consumes this epic's work:

| Shared code | First | Then |
|---|---|---|
| `lib/version.py` (`_read_version`, `_write_version`) | #1165 Task 5 | Task 9 |
| `reports.sh` and the `evidence` job in `ci-quality.yml` | #1165 Task 8 | Tasks 6 and 12b |
| The BUILD check kind and its CI gating | Task 5 (and Task 6 for CI) | #1165 Task 13 (BUILD on for Rust) |

Because Task 6 reads the BUILD flag from vergil-tooling (Task 5), #1165's
Task 13 turns Rust's BUILD on in CI with no vergil-actions change.

---

### Task 2: A pin is required only for a multi-family window (vergil-tooling)

**Files:**

- Modify: `src/vergil_tooling/lib/versions/resolve.py` (`_is_family_keyed`, the unpinned error message)
- Modify: `docs/site/docs/guides/consuming-repo-setup.md`, `docs/site/docs/guides/ci-architecture.md`, `docs/site/docs/reference/lint/repo-profile.md`, `docs/site/docs/standards/development/runtime-version-support-policy.md`, `docs/site/docs/standards/development/typescript/overview.md` (where they say a TypeScript repo must pin)
- Test: `tests/vergil_tooling/test_versions_resolve.py`, `tests/vergil_tooling/test_repo_init.py`

**Interfaces:**

- Produces: `_is_family_keyed(families: tuple[WindowFamily, ...]) -> bool` is `len(families) > 1`. A token prefix alone no longer forces a pin.
- Produces: the unpinned multi-family error reads `{language} must pin [ci].versions: its window has more than one family ({keys}), so there is no single default. Fix: add [ci].versions = ["{newest}"] (window: {window_list})`, with `{keys}` the family keys comma-joined.

- [ ] **Step 1: Write the failing tests.** Replace `test_prefixed_single_family_unpinned_is_fatal` (`test_versions_resolve.py:139`) with `test_prefixed_single_family_unpinned_library_resolves_every_version` (versions `("node-24", "node-22")`, primary `node-24`) and `test_prefixed_single_family_unpinned_app_resolves_newest` (versions `("node-24",)`). `test_cpp_unpinned_is_fatal` asserts the message contains `more than one family (cpp-clang, cpp-gcc)`. `test_unpinned_family_language_stops_the_wizard` (`test_repo_init.py:1086`) keeps C++ and matches the new text.
- [ ] **Step 2: Run** `vrg-container-run -- vrg-validate`; expect the TypeScript tests to fail with the old "until its lifecycle turn" error.
- [ ] **Step 3: Implement** the predicate and message. Update the five docs pages: a pin is an exception, required only for a multi-family language (C++ today).
- [ ] **Step 4: Run** `vrg-container-run -- vrg-validate`; expect PASS.
- [ ] **Step 5: Commit** `vrg-commit --type fix --scope versions --message "require a CI version pin only for a multi-family window"`, then report-ready.

### Task 3: A malformed `package.json` is an error (vergil-tooling)

**Files:**

- Modify: `src/vergil_tooling/lib/container.py` (`_package_json_has_typescript_dep`, `detect_language`; a new `PackageJsonError`)
- Modify: the commands that reach `detect_language` and face a user: `src/vergil_tooling/bin/vrg_container_run.py` (around line 142), `src/vergil_tooling/bin/vrg_container_cache.py` (around lines 113, 127, 144), `src/vergil_tooling/bin/vrg_finalize_pr.py` (around lines 1645, 1738), and, through `build_container_args`, `src/vergil_tooling/bin/vrg_scorecard.py` (around line 86) and `src/vergil_tooling/bin/vrg_container_docs.py` (around line 135)
- Test: `tests/vergil_tooling/test_container.py`, `tests/vergil_tooling/test_vrg_container_run.py`, and the scorecard and container-docs command tests

**Interfaces:**

- Produces: `class PackageJsonError(ValueError)` in `lib/container.py`.
- Produces: `read_package_json(path: Path) -> dict[str, object]`, the one parser of `package.json` that Tasks 7 and 11 reuse. It raises `PackageJsonError("package.json at <path> cannot be parsed: <reason>. Fix: correct the JSON")` when the file cannot be read, is not JSON, or is not a JSON object.
- Produces: one shared helper in `lib/container.py`, `report_package_json_error(exc: PackageJsonError) -> NoReturn`, which prints `ERROR: <message>` to stderr and exits 1. Each command above catches `PackageJsonError` around its language resolution and calls it. No traceback reaches the user.
- Produces: `detect_language` reads `package.json`, whenever it exists, before the `tsconfig.json` shortcut.

- [ ] **Step 1: Write the failing tests:** `test_malformed_package_json_errors`; `test_malformed_package_json_with_tsconfig_errors` (a `tsconfig.json` does not hide it); `test_package_json_array_errors`; `test_package_json_without_typescript_is_not_typescript` (returns `""`); `test_declared_language_skips_detection` (`resolve_language` with `primary-language = "typescript"` and a broken `package.json` still returns `typescript`, as today); `test_container_run_reports_broken_package_json` (in `test_vrg_container_run.py`: exit 1, stderr starts `ERROR: package.json at`, no traceback), and the same test for `vrg-scorecard` and `vrg-container-docs` on a repo with no declared language.
- [ ] **Step 2: Run** `vrg-container-run -- vrg-validate`; expect the error cases to fail (they return `False`/`""` today).
- [ ] **Step 3: Implement** `read_package_json` and the new order; drop the "bounded, documented fallback" docstring.
- [ ] **Step 4: Run** `vrg-container-run -- vrg-validate`; expect PASS.
- [ ] **Step 5: Commit** `vrg-commit --type fix --scope container --message "fail on a package.json that cannot be parsed"`, then report-ready.

### Task 4: The tool-ownership guideline page (vergil-tooling)

**Files:**

- Create: `docs/site/docs/standards/development/tool-ownership.md`
- Modify: `docs/site/mkdocs.yml` (nav, beside `runtime-version-support-policy.md`)
- Modify: `docs/site/docs/standards/development/python/`, `rust/` and `typescript/` overview pages (a link and that language's split)

- [ ] **Step 1: Write the page** to spec §6.2: the default (follow the ecosystem), the exception (the image may own a tool when the reason behind the advice is met by the shared image, never a technical constraint), the test, the cost table with its acceptance, what the evidence records today (repo-declared versions through the lockfile; image-tool versions are vergil-project/vergil-tooling#4416's), the six sources with what each says, and the per-language split. Python's row says its migration is pending.
- [ ] **Step 2: Link it** from the three language overview pages, each listing its own split.
- [ ] **Step 3: Run** `vrg-container-run -- vrg-validate`; expect PASS (the docs check builds the site strictly).
- [ ] **Step 4: Commit** `vrg-commit --type docs --scope standards --message "add the cross-language tool-ownership guideline"`, then report-ready.

### Task 5: A language-neutral BUILD check kind (vergil-tooling)

**Files:**

- Modify: `src/vergil_tooling/lib/languages.py` (`CheckKind`, `EcosystemInfo`, `language_commands`, `check_cardinality`)
- Modify: `src/vergil_tooling/bin/vrg_validate.py` (`_CHECK_KINDS`, `_LANGUAGE_CHECK_ORDER`, the nothing-to-run message)
- Modify: `src/vergil_tooling/bin/vrg_ci_resolve.py` (a `build-enabled` field in its output)
- Modify: `docs/site/docs/guides/validation-matrix.md`, `docs/site/docs/guides/ci-architecture.md`
- Test: `tests/vergil_tooling/test_languages.py`, `tests/vergil_tooling/test_vrg_validate.py`, `tests/vergil_tooling/test_vrg_ci_resolve.py`

**Interfaces:**

- Produces: `CheckKind.BUILD = "build"`.
- Produces: `EcosystemInfo.validate_build: bool = False`. `language_commands(lang, CheckKind.BUILD)` returns `[ecosystem.build_cmd]` when `validate_build` is true and `build_cmd` is set, else `[]`. The BUILD command is therefore the same command the release job runs, with one source.
- Produces: BUILD's cardinality is `Cardinality.ONCE` for every language.
- Produces: `vrg-validate --check build`; `_LANGUAGE_CHECK_ORDER` is LINT, TYPECHECK, BUILD, TEST, AUDIT. With no BUILD command, `--check build` prints `build: no build check for <language> — nothing to run` and exits 0.
- Produces: `vrg-ci-resolve`'s output gains `build-enabled`, `"true"` when `language_commands(language, CheckKind.BUILD)` is non-empty, else `"false"`. This is the single source CI reads to decide whether the `build` job runs (Task 6), so turning BUILD on for a language in vergil-tooling turns it on in CI with no vergil-actions change.

- [ ] **Step 1: Write the failing tests:** `test_build_kind_off_for_every_language` (python, go, rust, cpp, typescript all give `[]`); `test_build_commands_come_from_build_cmd` (a spec with `validate_build=True` gives `[build_cmd]`); `test_build_cardinality_is_once`; `test_check_build_nothing_to_run` (`vrg-validate --check build` on a Python repo exits 0 with the message); `test_full_validate_runs_build_after_typecheck` (order, with a fake spec); `test_ci_resolve_build_enabled` (`"false"` for python, `"true"` for a fake spec with BUILD on).
- [ ] **Step 2: Run** `vrg-container-run -- vrg-validate`; expect them to fail (`CheckKind` has no `BUILD`).
- [ ] **Step 3: Implement** the kind, the flag and the wiring. Docs: the validation matrix lists BUILD, on for no language yet.
- [ ] **Step 4: Run** `vrg-container-run -- vrg-validate`; expect PASS.
- [ ] **Step 5: Commit** `vrg-commit --type feat --scope validate --message "add a language-neutral BUILD check kind"`, then report-ready.

### Task 6: The `build` job in `ci-quality.yml`, gated on vergil-tooling's BUILD flag (vergil-actions)

**Files:**

- Modify: `actions/shared/setup/vergil/action.yml` (a `build-enabled` output, passed through from `vrg-ci-resolve`)
- Modify: `.github/workflows/ci-quality.yml` (the `matrix` job's outputs; a `build` job; the `evidence` job's `needs` and its "Assert all required legs succeeded" step)
- Create: `actions/ci/evidence/assert/build-gate.sh` and `actions/ci/evidence/assert/tests/build-gate.test.sh` (the gate decision as a tested script, as `reports.sh` has `tests/reports.test.sh`)
- Modify: the CI workflow docs page that lists the quality jobs

**Interfaces:**

- Consumes: Task 5's `vrg-validate --check build` and `vrg-ci-resolve`'s `build-enabled`.
- Produces: the setup action's `build-enabled` output, and a `build-enabled` output on `ci-quality.yml`'s `matrix` job. An older vergil-tooling that emits no field gives an empty output, which counts as `false`: the job is skipped, so no repo calls `--check build` before a release supports it.
- Produces: job `build`, named `build / ${{ matrix.version }}`, on the collapsed `once` matrix (the primary version), with the same checkout, setup and `assert-toolchain` steps as `typecheck`, running `vrg-validate --check build`, with `if: needs.matrix.outputs.build-enabled == 'true'`. No language list lives in vergil-actions. `evidence` needs `build`. No ruleset change: `quality / evidence` already gates the quality workflow.
- Produces: `build-gate.sh <build-enabled> <build-result>`, which exits 0 for (`false`, `skipped`) and for (`true`, `success`), and exits 1 with a message for every other pair: (`true`, `skipped`), any `failure` or `cancelled`, and (`false`, anything but `skipped`). The `evidence` job's assert step calls it with `needs.matrix.outputs.build-enabled` and `needs.build.result`.

- [ ] **Step 1: Write the failing cases** in `build-gate.test.sh`: each pair above, with its exit status.
- [ ] **Step 2: Run** `vrg-container-run -- vrg-validate`; expect them to fail (no script).
- [ ] **Step 3: Implement** the script, the setup output, the `matrix` output, the job and the assert step's call. Run 1 is the live TypeScript proof that the job runs and gates.
- [ ] **Step 4: Run** `vrg-container-run -- vrg-validate`; expect PASS (the script tests and actionlint).
- [ ] **Step 5: Commit** `vrg-commit --type feat --scope ci --message "run the BUILD check in the quality workflow"`, then report-ready. The push is refused for `.github/workflows/**`; report-ready records it and the human's `vrg-submit-pr` pushes.

### Task 7: TypeScript tools from the repo, with accepted ranges (vergil-tooling)

**Files:**

- Create: `src/vergil_tooling/configs/typescript/dev-dependencies.json` (the accepted ranges, the one place they live)
- Create: `src/vergil_tooling/lib/typescript_tools.py`
- Modify: `src/vergil_tooling/lib/languages.py` (TypeScript INSTALL, LINT's ESLint step, TYPECHECK, TEST)
- Modify: `src/vergil_tooling/bin/vrg_validate.py` (`_build_stages`: a TypeScript pre-gate stage)
- Modify: `docs/site/docs/standards/development/typescript/overview.md`, `toolchain-and-strictness.md`, `testing-and-coverage.md`
- Test: `tests/vergil_tooling/test_typescript_tools.py`, `tests/vergil_tooling/test_languages.py`

**Interfaces:**

- Consumes: Task 3's `read_package_json`.
- Produces: `dev-dependencies.json`, mapping each required devDependency to its accepted major: `typescript` 6; `typescript-eslint` 8; `eslint` and `@eslint/js` one shared major that typescript-eslint 8 supports (`^8.57.0 || ^9.0.0 || ^10.0.0`); `vitest` and `@vitest/coverage-v8` one shared major. The implementer sets the eslint and vitest majors to the newest ones that satisfy every peer range, and names the sources in the commit body.
- Produces: `typescript_tools.load_required() -> dict[str, int]` and `typescript_tools.check_dev_dependencies(repo_root: Path) -> list[str]`, one problem per package, empty when all pass. A package fails when it is missing from `devDependencies`; when its declared range is not `^M…`, `~M…` or `M…` with `M` the accepted major; when it is not installed (`node_modules/<name>/package.json` is absent); or when the installed version (that file's `version`) has another major. The `vrg-validate` failure lists every problem and one fix line: `npm install --save-dev <name>@^<M> …`.
- Produces: the **TypeScript pre-gate stage** in `vrg_validate._build_stages`. It runs right after INSTALL, whenever any TypeScript language check kind runs: the full run and every single `--check lint|typecheck|build|test|audit`. CI runs one `--check` per job, so every TypeScript CI job enforces it. It takes the existing not-bootstrapped skip (`_bootstrap_state`), so a fresh fixture with no `package.json` yet skips it with the usual notice instead of failing. It runs a list of checks, `typescript_tools.PRE_GATE_CHECKS: tuple[Callable[[Path], list[str]], ...]`, and fails with every problem from all of them at once. This task adds `check_dev_dependencies`; Tasks 8b, 9 and 11 append theirs.
- Produces: INSTALL `npm ci --ignore-scripts`; ESLint, `tsc` and Vitest run by path, as `node_modules/.bin/<tool>`, so a tool missing from the repo fails instead of falling back to a global binary (`npx` falls back to the global `bin` even with `--no-install`). Prettier and license-checker stay bare image binaries. ESLint's config staging stays until Task 8b.

- [ ] **Step 1: Write the failing tests:** `test_all_required_present_passes`; `test_missing_package_named_in_problem`; `test_typescript_5_range_rejected` (`^5.9.0`); `test_latest_star_and_open_ranges_rejected` (`latest`, `*`, `>=5`); `test_installed_major_mismatch_rejected` (range `^6.0.0`, installed `7.0.2`); `test_declared_and_locked_but_not_installed_rejected` (in `package-lock.json`, absent from `node_modules`); `test_fix_line_lists_every_problem_package`; `test_typescript_install_ignores_scripts` (INSTALL is `["npm", "ci", "--ignore-scripts"]`); `test_typescript_gates_run_repo_binaries_by_path` (every TYPECHECK, TEST and ESLint argv starts with `node_modules/.bin/`); `test_check_lint_fails_on_missing_dev_dependency` (`vrg-validate --check lint` alone fails); `test_unbootstrapped_typescript_repo_skips_pre_gate` (no `package.json`: the stage is skipped with the not-bootstrapped notice).
- [ ] **Step 2: Run** `vrg-container-run -- vrg-validate`; expect them to fail.
- [ ] **Step 3: Implement** the module, the data file, the commands and the pre-gate stage. Docs: the required devDependencies and their majors, that tool versions float through the lockfile (drop "the same pinned tool versions"), and that install scripts never run, with a needed one declared as an exception (#375).
- [ ] **Step 4: Run** `vrg-container-run -- vrg-validate`; expect PASS.
- [ ] **Step 5: Commit** `vrg-commit --type feat --scope typescript --message "run TypeScript gates on the repo's own tools within accepted ranges"`, then report-ready.

### Task 8a: Three managed TypeScript files in the fleet sync (vergil-tooling)

**Files:**

- Modify: `src/vergil_tooling/lib/repo_config.py` (a per-language managed-file table and its drift function; `audit_local_config`)
- Modify: `src/vergil_tooling/lib/repo_config_fixes.py` (one fixer and its `FIXERS` entry)
- Modify: `src/vergil_tooling/lib/repo_init.py` (the config-files step, around lines 1455-1486, that writes every other managed file at creation)
- Modify: `src/vergil_tooling/configs/typescript/eslint.config.mjs` (it becomes the base a repo spreads: a default-exported array, no reference to its own file name)
- Modify: `docs/site/docs/reference/config-audit.md` (the managed-file list)
- Test: `tests/vergil_tooling/test_repo_config.py`, `tests/vergil_tooling/test_repo_config_fixes.py`, `tests/vergil_tooling/test_repo_init.py`

**Interfaces:**

- Produces: `LANGUAGE_MANAGED_FILES: dict[str, tuple[LanguageManagedFile, ...]]`, with `LanguageManagedFile(repo_path: str, master: str)`. TypeScript's entries: `tsconfig.vergil.json` from `configs/typescript/tsconfig.base.json`, `eslint.vergil.config.mjs` from `configs/typescript/eslint.config.mjs`, `.prettierrc.json` from `configs/typescript/prettier.config.json`. The master copies stay where the gates already read them, so there is one copy of each.
- Produces: `language_files_drift(root: Path) -> list[ManagedFileDrift]`, for the repo's `[project].primary-language`, on the existing `ManagedFileDrift` shape. Drift means missing or differs from the master. A `vergil.toml` that fails to load is drift with no fix, so the fixer raises `CannotFix`, as for `.gitignore`.
- Produces: `fix_language_files(root: Path) -> …`, in `FIXERS`, which writes each drifted file whole. Another language adds managed files with a table entry.
- Produces: `vrg-github-repo-init` writes the language's managed files from the same table in its config-files step, beside `.gitignore` and `dependabot.yml`. A new TypeScript repo is born without drift, with no wait for the nightly sync.

- [ ] **Step 1: Write the failing tests:** `test_typescript_repo_missing_files_is_drift`; `test_edited_managed_file_is_drift`; `test_matching_files_no_drift`; `test_non_typescript_repo_gets_none`; `test_same_named_file_in_python_repo_untouched`; `test_unloadable_vergil_toml_cannot_fix`; `test_fixer_writes_master_bytes`; `test_fresh_typescript_init_passes_audit` (in `test_repo_init.py`: the files repo-init writes give an empty `audit_local_config` for the managed files).
- [ ] **Step 2: Run** `vrg-container-run -- vrg-validate`; expect them to fail.
- [ ] **Step 3: Implement** the table, the drift function, the audit entry, the fixer and the repo-init write. Docs: the three files.
- [ ] **Step 4: Run** `vrg-container-run -- vrg-validate`; expect PASS.
- [ ] **Step 5: Commit** `vrg-commit --type feat --scope repo-config --message "manage the TypeScript base configs as per-language files"`, then report-ready.

### Task 8b: Entry-point and single-Prettier checks; LINT reads the managed configs (vergil-tooling)

**Files:**

- Modify: `src/vergil_tooling/lib/typescript_tools.py` (entry-point and Prettier checks, a JSONC loader)
- Modify: `src/vergil_tooling/lib/languages.py` (TypeScript LINT)
- Modify: `src/vergil_tooling/lib/typescript_tools.py` `PRE_GATE_CHECKS` (append both checks)
- Modify: `docs/site/docs/standards/development/typescript/overview.md` ("Extending the shareable base": the real `extends` path, `eslint.config.mjs`, Prettier, weakening as a #375 exception, the editor-version caveat)
- Test: `tests/vergil_tooling/test_typescript_tools.py`, `tests/vergil_tooling/test_languages.py`

**Interfaces:**

- Consumes: Task 7's module and `PRE_GATE_CHECKS`; Task 8a's file names.
- Produces: `load_jsonc(text: str) -> object`, which parses as `tsc` does: `//` and `/* */` comments outside strings, and trailing commas.
- Produces: `check_entry_points(repo_root: Path) -> list[str]`. `tsconfig.json` must exist and its `extends` must be `"./tsconfig.vergil.json"`, or an array containing it. `eslint.config.mjs` must exist, import a binding from `"./eslint.vergil.config.mjs"`, and include that binding in its default export in any of the common forms: spread into an array (`[...vergil, …]`), or passed as an argument to `defineConfig(…)` or `tseslint.config(…)`.
- Produces: `check_prettier_single_source(repo_root: Path) -> list[str]`, which fails on a `"prettier"` key in `package.json`, or on any `.prettierrc`, `.prettierrc.*` or `prettier.config.*` other than `.prettierrc.json`.
- Produces: LINT is `prettier --check . --config .prettierrc.json`, then `node_modules/.bin/eslint .`, with no staging and no `--config`.

- [ ] **Step 1: Write the failing tests:** `test_tsconfig_with_comments_and_trailing_comma_passes`; `test_extends_array_containing_base_passes` (Review Focus 1); `test_tsconfig_without_base_fails`; `test_eslint_config_without_spread_fails`; `test_eslint_config_spreading_base_with_ignores_passes`; `test_eslint_define_config_with_base_passes`; `test_eslint_tseslint_config_with_base_passes`; `test_prettier_key_in_package_json_fails` (Review Focus 3); `test_extra_prettierrc_fails`; `test_lint_has_no_staging` (no LINT argv mentions `.vergil-eslint.config.mjs`).
- [ ] **Step 2: Run** `vrg-container-run -- vrg-validate`; expect them to fail.
- [ ] **Step 3: Implement** the checks, the loader and the LINT commands. Do not detect weakened rules: that is epic #375's TypeScript plug-in.
- [ ] **Step 4: Run** `vrg-container-run -- vrg-validate`; expect PASS.
- [ ] **Step 5: Commit** `vrg-commit --type feat --scope typescript --message "check that TypeScript entry points build on the managed configs"`, then report-ready.

### Task 9: `vrg-version` supports `package.json` (vergil-tooling)

**Files:**

- Modify: `src/vergil_tooling/lib/version.py` (`_DEFAULT_VERSION_FILES`, `_LANGUAGES_WITH_SEPARATE_VERSION`, `_LOCKFILES`, `_read_version`, `_write_version`, `_run_lockfile_maintenance`, `bump_paths`, and `bump()`'s TypeScript path)
- Modify: `src/vergil_tooling/lib/typescript_tools.py` (`check_version_matches`, appended to `PRE_GATE_CHECKS`)
- Test: `tests/vergil_tooling/test_version.py`, `tests/vergil_tooling/test_typescript_tools.py`

**Interfaces:**

- Produces: TypeScript in `_DEFAULT_VERSION_FILES` (`package.json`), `_LANGUAGES_WITH_SEPARATE_VERSION` and `_LOCKFILES` (`package-lock.json`). It gets no `_LOCKFILE_COMMANDS` entry.
- Produces: a JSON `"version"` branch in `_read_version` and `_write_version` for `package.json`. The write locates the top-level `version` value's exact span by parsing (not a text replace, which silently does nothing on `"version":"0.1.0"`), rewrites only that span, and raises `VersionSyncError` naming the file when it cannot.
- Produces: `_write_package_lock_version(lockfile: Path, new: str) -> None`, called by `_run_lockfile_maintenance` for TypeScript. It sets the top-level `version` and `packages[""].version` in place, runs no subprocess, and leaves every other byte unchanged.
- Produces: a `package.json` with no `version` key raises `VersionSyncError("package.json has no \"version\". Fix: set it to the VERSION file's <v>")`; a mismatch raises the existing cross-check error, with the same fix.
- Produces: for TypeScript only, `bump()` raises `VersionSyncError` naming the file and the fix when `package.json` is missing, cannot be written, or has no `version` key, instead of reaching the `except (FileNotFoundError, ValueError): pass` handler. Every other language keeps that handler unchanged, as #1165 Task 5 decided; its fleet-wide removal is the intake item in Filing.
- Produces: `typescript_tools.check_version_matches(repo_root: Path) -> list[str]`, in `PRE_GATE_CHECKS`. It runs the same cross-check as `vrg-version show`, so a `package.json` version that differs from `VERSION` fails on the PR, not at `vrg-release` preflight.

- [ ] **Step 1: Write the failing tests:** `test_show_reads_package_json`; `test_bump_typescript_rewrites_only_version_fields` (every other key of `package.json` and every `node_modules/*` entry of the lockfile byte-identical); `test_bump_preserves_formatting` (Review Focus 4); `test_bump_paths_lists_package_json_and_lock`; `test_typescript_bump_runs_no_subprocess` (a `subprocess.run` that raises); `test_show_missing_version_key_names_fix`; `test_show_mismatch_names_fix` (`1.0.0` against `VERSION` `0.1.0`); `test_typescript_bump_write_failure_is_raised` (an unwritable `package.json` fails the bump); `test_compact_version_key_is_rewritten` (`"version":"0.1.0"` with no space); `test_go_bump_without_version_file_unchanged` (Go's handler is untouched); `test_pre_gate_fails_on_version_mismatch`.
- [ ] **Step 2: Run** `vrg-container-run -- vrg-validate`; expect them to fail.
- [ ] **Step 3: Implement** the branches, the lockfile writer and the map entries.
- [ ] **Step 4: Run** `vrg-container-run -- vrg-validate`; expect PASS.
- [ ] **Step 5: Commit** `vrg-commit --type feat --scope version --message "bump the version of a TypeScript repo's package.json and lockfile"`, then report-ready.

### Task 10: TypeScript evidence files, ignored by the fragment (vergil-tooling)

**Files:**

- Modify: `src/vergil_tooling/lib/languages.py` (TypeScript LINT, TYPECHECK, TEST and AUDIT; `NPM_AUDIT_REPORT`, the two quality report names, `TYPESCRIPT_REPORT_FILES`)
- Modify: `src/vergil_tooling/data/gitignore/typescript`
- Modify: `docs/site/docs/guides/ci-evidence-convention.md`
- Test: `tests/vergil_tooling/test_languages.py`, `tests/vergil_tooling/test_gitignore.py`

**Interfaces:**

- Consumes: Task 7's `node_modules/.bin/` argv for ESLint, `tsc` and Vitest. Task 8b also edits LINT; whichever lands second keeps both changes.
- Produces: `NPM_AUDIT_REPORT = "npm-audit.json"`; `ESLINT_REPORT = "quality-eslint.json"` (LINT, ESLint's JSON formatter, from the run that gates); `TSC_REPORT = "quality-tsc.txt"` (TYPECHECK, `tsc`'s diagnostics, from the run that gates); `TYPESCRIPT_REPORT_FILES = (COVERAGE_REPORT, JUNIT_REPORT, NPM_AUDIT_REPORT, ESLINT_REPORT, TSC_REPORT)`. The names follow the contract's `quality-*` pattern for Python and Rust; the implementer may pick the `tsc` report's format, as long as it is written on failure too.
- Produces: TEST writes `junit.xml` (Vitest's JUnit reporter) and Cobertura `coverage.xml` beside its console output, at 100% lines, both even when a test fails. Coverage on a failing run needs `--coverage.reportOnFailure`, since Vitest's default is false. AUDIT runs `npm audit --audit-level=high --json`, writes the JSON to `npm-audit.json` from the run that gates, and exits with its status. The threshold is unchanged. The fragment adds `npm-audit.json`, the two `quality-*` reports and `coverage/`. `base` already ignores `coverage.xml` and `junit.xml`.

- [ ] **Step 1: Write the failing tests:** `test_typescript_report_files`; `test_typescript_test_writes_junit_and_cobertura` (the argv or script names both outputs); `test_typescript_test_reports_coverage_on_failure` (`--coverage.reportOnFailure` is present); `test_typescript_audit_writes_json_and_gates` (the argv keeps `--audit-level=high`); `test_typescript_lint_writes_eslint_report`; `test_typescript_typecheck_writes_tsc_report`; `test_typescript_fragment_ignores_reports`.
- [ ] **Step 2: Run** `vrg-container-run -- vrg-validate`; expect them to fail.
- [ ] **Step 3: Implement** the commands, constants and fragment. Docs: the TypeScript report files.
- [ ] **Step 4: Run** `vrg-container-run -- vrg-validate`; expect PASS.
- [ ] **Step 5: Commit** `vrg-commit --type feat --scope typescript --message "write TypeScript quality, test and audit evidence"`, then report-ready.

### Task 11: TypeScript build command, BUILD on for TypeScript, `build` script check (vergil-tooling)

**Files:**

- Modify: `src/vergil_tooling/lib/languages.py` (TypeScript `EcosystemInfo`)
- Modify: `src/vergil_tooling/lib/typescript_tools.py` (`check_build_script`, appended to `PRE_GATE_CHECKS`)
- Modify: `docs/site/docs/guides/validation-matrix.md` (BUILD on for TypeScript), the TypeScript overview page (the `build` script requirement)
- Test: `tests/vergil_tooling/test_languages.py`, `tests/vergil_tooling/test_typescript_tools.py`

**Interfaces:**

- Consumes: Task 5's `validate_build` and `CheckKind.BUILD`; Task 3's `read_package_json`.
- Produces: TypeScript `build_cmd = ["sh", "-c", "npm ci --ignore-scripts && npm run build"]`, `validate_build = True`, `publish_cmd = None`. In `vrg-validate` and the CI `build` job this reinstalls `node_modules` after INSTALL already did. That cost is accepted, so validate and release run one command from one source. Run 1 shows how long it takes, and a slow result is a follow-on, not a change here.
- Produces: `check_build_script(repo_root: Path) -> list[str]`, which fails when `package.json` has no `scripts.build`, with the fix `add a "build" script to package.json (for example "tsc -p tsconfig.json")`.

- [ ] **Step 1: Write the failing tests:** `test_typescript_build_cmd_installs_first_without_scripts`; `test_build_on_for_typescript` (`language_commands(.., BUILD)` is non-empty for typescript and `[]` for python, go and cpp; it asserts nothing about rust, which #1165 Task 13 turns on); `test_missing_build_script_fails`; `test_publish_cmd_stays_none`.
- [ ] **Step 2: Run** `vrg-container-run -- vrg-validate`; expect them to fail.
- [ ] **Step 3: Implement** the ecosystem entry and the check. Docs as listed.
- [ ] **Step 4: Run** `vrg-container-run -- vrg-validate`; expect PASS.
- [ ] **Step 5: Commit** `vrg-commit --type feat --scope typescript --message "build TypeScript in validate and at release"`, then report-ready.

### Task 12a: `cd-release` releases TypeScript (vergil-actions)

**Files:**

- Modify: `.github/workflows/cd-release.yml` (the description at line 7; the language list at line 615)
- Modify: `actions/cd/release/registry-publish/action.yml` (the `language` input's description, which reads "Primary language (python, rust, go)")
- Modify: the `cd-release` docs page

**Interfaces:**

- Produces: "Build and publish" runs for `["python","rust","go","typescript"]`. `registry-publish` stays `false` by default, so TypeScript builds and is never published. The release job's checkout and push paths are unchanged.

- [ ] **Step 1: Implement** the list and both descriptions. No unit test fits a language list in a workflow expression; the PR's smoke run (Step 4) and Run 1's TypeScript release are the proof.
- [ ] **Step 2: Run** `vrg-container-run -- vrg-validate`; expect PASS.
- [ ] **Step 3: Commit** `vrg-commit --type feat --scope release --message "release TypeScript repos"`, then report-ready. The push is refused for `.github/workflows/**`; the human's `vrg-submit-pr` pushes.
- [ ] **Step 4: In PR watch,** confirm the PR's own `smoke-cd-release` run is green. It runs on any PR that touches `cd-release.yml`, but it does not exercise the language list: Run 1's release is the TypeScript proof.

### Task 12b: The TypeScript row in the report contract (vergil-actions)

**Files:**

- Modify: `actions/ci/evidence/reports/reports.sh` (the table, `has_contract`, `report_files`, `report_tools`; the TypeScript warning removed)
- Test: the reports action's test script, beside `reports.sh`

**Interfaces:**

- Consumes: Task 10's file names: test `coverage.xml`, `junit.xml`; audit `npm-audit.json`; lint `quality-eslint.json`; typecheck `quality-tsc.txt`. The per-language `lint` and `typecheck` contracts and `expect` mode that #1165 Task 8 added (vergil-actions 1f233c6).
- Produces: TypeScript rows for all four gates on #1165's pattern. `report_files typescript test` gives `coverage.xml junit.xml`, `audit` gives `npm-audit.json`, `lint` gives `quality-eslint.json` and `typecheck` gives `quality-tsc.txt`. TypeScript warns on no gate. Go and C++ still warn.
- Produces: `report_tools` gains `typescript/test` (`vitest`, `@vitest/coverage-v8`), `typescript/audit` (`npm audit`), `typescript/lint` (`prettier`, `eslint`) and `typescript/typecheck` (`tsc`) branches, and a default branch that fails as an internal error when a language with a contract has no tools entry, unless #1165 Task 8 already added one. A contract language can then never emit an empty `tools` silently. The "keep the two in step" note now names `TYPESCRIPT_REPORT_FILES`.

- [ ] **Step 1: Write the failing cases:** the four `report_files` results; the four `report_tools` results; no warning for typescript on any gate; `expect` mode requiring the TypeScript quality reports; and a contract language with no `report_tools` branch failing as an internal error.
- [ ] **Step 2: Run** `vrg-container-run -- vrg-validate`; expect them to fail.
- [ ] **Step 3: Implement** the row.
- [ ] **Step 4: Run** `vrg-container-run -- vrg-validate`; expect PASS.
- [ ] **Step 5: Commit** `vrg-commit --type feat --scope evidence --message "stage TypeScript test and audit reports"`, then report-ready.

### Task 13: `fixture-run` learns the TypeScript fixtures (vergil-claude-plugin)

**Files:**

- Modify: `skills/fixture-run/SKILL.md` (the fixture table, the known gaps and their rule, the rollback section, the result's known-gap lines)

**Interfaces:**

- Produces: rows `ts-lib` (`vergil-project/vergil-fixture-ts-lib`, typescript, `library`) and `ts-app` (`vergil-project/vergil-fixture-ts-app`, typescript, `application`).
- Produces: the Known gaps table gains a **Recorded on** column. Each failing known gap's evidence is posted on that row's issue. Rust's rows keep vergil-project/.github#410. TypeScript's one row: dependency update → epic 3's planning item (filed with this epic's tasks). TypeScript has no quality-gate known gap, because Tasks 10 and 12b give it lint and typecheck evidence. Every sentence that names #410 as the one record issue now says "the row's issue".
- Produces: a TypeScript rollback paragraph, skipped while TypeScript's dependency-update gap stands, as Rust's was.

- [ ] **Step 1: Implement** the rows, the column, the rule text and the paragraph. If epic 3's planning item is not filed yet when this task starts, stop and ask for its ref.
- [ ] **Step 2: Run** `vrg-container-run -- vrg-validate`; expect PASS.
- [ ] **Step 3: Commit** `vrg-commit --type feat --scope fixture-run --message "add the TypeScript fixtures and per-gap known-gap records"`, then report-ready.

### Task 14: Tooling, actions and plugin releases attested; plugin deployed (deployment, .github)

Blocked by 2–13.

- **Precondition (human-attested):** releases of vergil-tooling (Tasks 2, 3, 4, 5, 7, 8a, 8b, 9, 10, 11), vergil-actions (6, 12a, 12b) and vergil-claude-plugin (13) are published, and their versions are named in a comment. vergil-tooling is released first, because the actions release's `build` job calls `vrg-validate --check build`.
- **Procedure (agent-safe):**
  1. Confirm each release's notes list its tasks' PRs.
  2. Deploy the plugin release to the host and the VMs (`vrg-host plugins`, `vrg-vm update --all`), and confirm `fixture-run` lists `ts-lib` and `ts-app`.
  3. In a scratch directory, run `vrg-container-run -- vrg-validate --check build` on a Python repo and confirm the nothing-to-run message.
- **SUCCESS:** every check passes, recorded with the release versions.

### Task 15: The two TypeScript fixture repos created (deployment, .github)

Blocked by 14.

- **Human-run steps:** create each repo with `vrg-github-repo-init`, using epic #1091's Task 5 flags with `--language typescript`:
  - `vergil-project/vergil-fixture-ts-lib` with `--repository-type library`;
  - `vergil-project/vergil-fixture-ts-app` with `--repository-type application`.

  Add the `APP_CLIENT_ID` and `APP_PRIVATE_KEY` secrets (no npm token), and run `vrg-adhoc-epic` on each.
- **Procedure (agent-safe):** confirm on each repo's `develop` that `vergil.toml` has `primary-language = "typescript"` and no `[ci].versions`, and that `tsconfig.vergil.json`, `eslint.vergil.config.mjs` and `.prettierrc.json` are present and match their masters, as repo-init wrote them (Task 8a). Then file Tasks 16a and 16b in the new repos, linked under #1174.
- **SUCCESS:** both repos exist, carry the managed files, and have their program tasks filed.

### Tasks 16a and 16b: The fixture programs (vergil-fixture-ts-lib, vergil-fixture-ts-app)

Blocked by 15. Each follows epic #1091 spec §6.2:

- a small real program with at least one real runtime dependency;
- 100% line coverage, and a docs site;
- the required devDependencies within their accepted ranges (TypeScript 6);
- a `tsconfig.json` that extends `./tsconfig.vergil.json`, and an `eslint.config.mjs` that includes `./eslint.vergil.config.mjs`;
- a `build` script;
- a `package.json` `"version"` equal to the `VERSION` file repo-init wrote (`0.1.0`), never `npm init`'s `1.0.0` default.

16a is a library that emits `dist/` with declaration files. 16b is a command-line tool, the shape of the planned mnemosys CLI, built to `dist/`. Each PR passes `vrg-container-run -- vrg-validate`, BUILD included. Commit `vrg-commit --type feat --scope program --message "add the fixture program"`, then report-ready.

### Fixture runs

Filed with the `fixture-run` skill. The plan's seeding is the approval it files under.

- **Run 1:** `fixture-run vergil-project/.github#1174 ts-lib,ts-app --validates "vergil-tooling <v> (TypeScript through the full cycle), vergil-actions <v>" --after <Task 14>`.
  - The plugin is not in `--validates`: no fixture consumes it, and Task 14 confirms its deployment.
  - After filing, each fixture's change task gets `Blocked-by:` its own program task (see Filing).
  - Its release is each fixture's `v0.1.0`.
  - Known gap: the dependency update only (epic 3).
  - One check the recipe's templates do not carry is posted as a comment on each validation task after filing: the run's CI shows a `quality / build` job, and the library ran two legs (`node-24`, `node-22`) where the app ran one, with no `[ci].versions`.
- **Run 2:** `fixture-run vergil-project/.github#1174 ts-lib,ts-app --validates "vergil-containers <v> (ts-node images without the global TypeScript tools)" --after <Task 18>`. The recipe's own checks are the proof: every gate passes on images that no longer carry the tools, which Task 18 confirmed.

### Task 17: The image drops the repo-side TypeScript tools (vergil-containers)

Blocked by Run 1's validation tasks.

**Files:**

- Modify: `docker/common/typescript-analysis.dockerfile` (keep `prettier` and `license-checker`; remove `typescript@5`, `eslint`, `typescript-eslint`, `vitest`, `@vitest/coverage-v8`; update its header comment to cite the guideline)
- Modify: `docker/ts-node/Dockerfile` (the committed render of that fragment; regenerate it)
- Modify: `docker/ts-node/smoke/run-in-container.sh`, the smoke project's `package.json` (declares the repo-side tools at the accepted majors) and its `package-lock.json` (regenerated to match), a smoke `eslint.config.mjs` so ESLint has a config to run, and `docker/ts-node/checks.sh` (`typescript_tools_run` checks `prettier` and `license-checker` only)
- Modify: `docs/site/docs/languages/ts-node.md` and `docs/site/docs/images/index.md` (around lines 284-294, the list of global tools): what the image carries

- [ ] **Step 1: Change the smoke test first.** It runs `npm ci --ignore-scripts`, then `node_modules/.bin/` tsc, eslint and vitest from the smoke project, fails if any of them is missing there, and asserts that `tsc`, `eslint` and `vitest` are not on the image's `PATH` while `prettier` and `license-checker` are. Update the smoke `package.json`, lockfile and ESLint config to match.
- [ ] **Step 2: Run** `vrg-container-run -- vrg-validate` against the current fragment; expect the smoke test to fail on the globals' presence.
- [ ] **Step 3: Implement** the fragment change, the rendered `Dockerfile`, `checks.sh` and the docs.
- [ ] **Step 4: Run** `vrg-container-run -- vrg-validate`; expect PASS, the ts-node image build and smoke included.
- [ ] **Step 5: Commit** `vrg-commit --type feat --scope ts-node --message "leave repo-declared TypeScript tools to the repo"`, then report-ready.

### Task 18: Containers release attested; ts-node images rebuilt (deployment, .github)

Blocked by 17.

- **Precondition (human-attested):** a vergil-containers release carrying Task 17 is published, and its version is named in a comment.
- **Procedure (agent-safe):** for each window series, in the published prod `ts-node` image, `command -v tsc eslint vitest` finds nothing, and `prettier --version` and `license-checker --version` succeed.
- **SUCCESS:** both checks pass on every window image, recorded with the digests.

## Evolution during execution

### 2026-11-16: Task 10 runs after Task 8b

**What changed:**

- **Task 10 (vergil-project/vergil-tooling#4508) is dispatched only once Task 8b (vergil-project/vergil-tooling#4506) has merged.** #4508 carries `Blocked-by: vergil-project/vergil-tooling#4506`. Tasks 9 and 11 still run in parallel with Task 8b.
- The delivery table and the Task 8b and Task 10 sections above are unchanged. The plan froze when its first task merged.

**Why:**

- **Both tasks rewrite the TypeScript LINT command** in `src/vergil_tooling/lib/languages.py`. Task 8b drops the per-run ESLint config staging and the `--config` argument; Task 10 adds report output. Run in parallel, their PRs would conflict on the same lines, which a rebase cannot resolve mechanically. The plan listed Task 10 as blocked only by Task 7, and Task 10's interfaces left the overlap to whichever task landed second.
- The dependency was found during execution and approved at the epic's gate: [the ruling on the epic](https://github.com/vergil-project/.github/issues/1174#issuecomment-6928260800).

### 2026-11-17: Prettier's check skips Vergil-managed files, and a release task gates the fixture programs

**What changed:**

- **A new code task, vergil-project/vergil-tooling#4541:** TypeScript LINT's Prettier check leaves out the files Vergil manages, with a test that a newly created TypeScript repo passes LINT.
- **A new deployment task, vergil-project/.github#1223:** a vergil-tooling release carrying #4541 is attested and installed, and LINT passes on a fixture program branch. It carries `Blocked-by: vergil-project/vergil-tooling#4541`.
- **New blockers:** Tasks 16a and 16b (vergil-project/vergil-fixture-ts-lib#1 and vergil-project/vergil-fixture-ts-app#1) and Run 1's change tasks (vergil-project/vergil-fixture-ts-lib#2 and vergil-project/vergil-fixture-ts-app#2) now also carry `Blocked-by: vergil-project/.github#1223`.
- **Run 1's `--validates` is unchanged** (vergil-tooling v2.1.319, vergil-actions v2.1.78). Its deployed-version probe accepts a later release that contains it.
- The delivery table and every task section above are unchanged. The plan froze when its first task merged.

**Why:**

- **Both fixture programs passed every gate except LINT.** LINT's `prettier --check .` runs over the whole repo, and Vergil had no Prettier ignore list. Under the managed `.prettierrc.json`, Prettier flags three files that `vrg-github-repo-init` writes and the fleet sync manages: `.claude/settings.json`, `.github/workflows/ops.yml`, and the managed region of `CLAUDE.md`. So every newly created TypeScript repo failed LINT, and no repo could fix that itself. The fixture programs exist to catch exactly this; Tasks 8a and 8b's unit tests did not.
- **The fixture programs validate against installed vergil-tooling,** and v2.1.319 lacks the fix, so they wait on a release that carries it.
- The tasks were found during execution and approved at the epic's gate: [the N0.3 ruling](https://github.com/vergil-project/.github/issues/1174#issuecomment-6932148608) and [the N0.4 ruling](https://github.com/vergil-project/.github/issues/1174#issuecomment-6935022666).

### 2026-11-17: Run 1's fixes, and Run 2 goes before epic #1218's fixture run

**What changed:**

- **Three new vergil-tooling code tasks under the epic,** each fixing a defect Run 1 found, all released in vergil-tooling v2.1.321:
  - **vergil-project/vergil-tooling#4550 (PR #4551):** a container cache counts as stale when the pinned vergil-tooling line has a newer release. The cache key now includes the resolved release, and a failed lookup fails hard and names `VRG_DOCKER_INSTALL_TAG`. It supersedes vergil-project/vergil-tooling#496, closed as not planned.
  - **vergil-project/vergil-tooling#4552 (PR #4554):** repo-init writes a minimal `package.json` for a TypeScript repo, because Task 9's version check made a new repo's first CD run on `main` fail.
  - **vergil-project/vergil-tooling#4553 (PR #4555):** release notes are written Prettier-clean, and LINT's Prettier check leaves out `CHANGELOG.md` and `releases/*.md`, because every TypeScript release failed LINT on its own generated note. Both Run 1 release tasks (vergil-project/vergil-fixture-ts-lib#3 and vergil-project/vergil-fixture-ts-app#3) gained `Blocked-by: vergil-project/vergil-tooling#4553`.
- **Run 1's releases resumed with no change to the fixtures.** Both first failed at `merge-release` on vergil-tooling v2.1.320; they were re-checked on v2.1.321 and resumed with `vrg-release --resume`.
- **Run 2 ran before epic vergil-project/.github#1218's fixture run on the same fixtures.** #1218 had filed its trio on both fixtures (#10 to #12 in each) before Run 2 was filed. By the human's decision, Run 2 went first, and #1218's trio waited until Run 2's validations (#15 in each) closed. The decision is recorded on both runs' change tasks: vergil-project/vergil-fixture-ts-lib#13 and #10, and vergil-project/vergil-fixture-ts-app#13 and #10. The human coordinates the two epics.
- **Items routed out of the epic:**
  - vergil-project/.github#1250: the BUILD check leaves no report in the evidence bundle, for any language. Captured in intake.
  - `vrg-validate --check` passes when no language is detected. Captured in intake as vergil-project/.github#1231, since moved to vergil-project/vergil-tooling#4594.
  - merge-reviewer replies are wrapped in a code fence. Captured in intake as vergil-project/.github#1238, since moved to vergil-project/vergil-claude-plugin#1130.
  - vergil-project/vergil-actions#1268: the App secrets are org secrets. A doc task on vergil-actions' ad-hoc epic.
- The delivery table and every task section above are unchanged. The plan froze when its first task merged.

**Why:**

- **The fixtures exist to catch what unit tests miss, and they did.** The cache defect made both fixture programs run old tooling after the #4541 release. The missing `package.json` and the unformatted release note each break every new TypeScript repo, and no repo can fix them itself.
- **The sequencing was needed** because the fixture-run recipe's serialization rule cannot tell two filed but not yet started runs apart. The human chose to coordinate the two runs by hand for now rather than file a task.
- The tasks and routings were found during execution and approved by the human: for #4550, [the N0.2 ruling on vergil-project/vergil-fixture-ts-app#1](https://github.com/vergil-project/vergil-fixture-ts-app/issues/1#issuecomment-6947907559) and [the N0.1 ruling on #4550](https://github.com/vergil-project/vergil-tooling/issues/4550#issuecomment-6948580369), with [the human's scope change](https://github.com/vergil-project/vergil-tooling/issues/4550#issuecomment-6948563277); for #4552, [the N0.6 ruling](https://github.com/vergil-project/.github/issues/1174#issuecomment-6948980292); for #4553, [the N0.7 ruling](https://github.com/vergil-project/.github/issues/1174#issuecomment-6949304751); for #1250, [the N0.8 ruling](https://github.com/vergil-project/.github/issues/1174#issuecomment-6951452188); and for vergil-actions#1268, [the N0.2 ruling on the epic](https://github.com/vergil-project/.github/issues/1174#issuecomment-6931858423). The N0.3 and N0.4 rulings are linked in the entry above; the N0.4 comment also carries N0.5.
