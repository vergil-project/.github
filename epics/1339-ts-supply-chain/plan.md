# TypeScript supply chain and the npm updater — implementation plan

- **Epic:** vergil-project/.github#1339
- **Planning issue:** vergil-project/.github#1209 (review history is in its comments)
- **Status:** draft, in design review

> **For agentic workers:** each task is one issue and one PR in the repo named
> in its heading, worked with `issue-implement`. Operational tasks are worked
> with `issue-deploy` or `issue-validate`, and fixture runs with the trio
> `fixture-run` files. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** A TypeScript repo's dependency licenses are gated against Vergil's
allowlist with evidence, `vrg-update-deps` updates its `package-lock.json`
and reports what is held back, and the TypeScript fixture runs prove both
with no known gap.

**Architecture:**

- **vergil-tooling** factors the time-boxed exception rules out of the npm
  audit gate, adds `vrg-npm-license-gate` over `package-lock.json` in the
  TypeScript AUDIT check, and adds `NpmUpdater` to `vrg-update-deps`.
- **vergil-actions** adds `licenses.json` to the `typescript/audit` report
  contract.
- **vergil-claude-plugin** drops the `fixture-run` TypeScript known gap and
  gives the change task a rollback.
- Releases, then a fixture run on each TypeScript fixture, prove the set.
  Once they pass, an operational step files license-checker's removal from
  the images as ad-hoc work.

**Tech Stack:** Python 3 and pytest (vergil-tooling), bash
(`reports.sh` and `tests/reports.test.sh`), GitHub Actions YAML, npm 11 in
the ts-node images, Markdown skills (vergil-claude-plugin).

**Spec:** [`spec.md`](spec.md) in this directory. Executors read both.

## Global Constraints

- The license gate reads only the committed `package-lock.json`; the
  updater changes only `package-lock.json` (spec §8).
- Every npm command Vergil runs passes `--ignore-scripts` (spec §8).
- Python's and Rust's gate commands, reports, evidence and updaters stay
  byte-for-byte the same. Factoring the exception rules out of
  `lib/npm_audit.py` changes no audit-gate behaviour (spec §3, §8).
- No silent failures: an unreadable or pre-v2 lockfile, a failed `npm
  update`, a changed `package.json`, a failed `npm outdated` and an entry
  without `license` after an update are all errors (spec §7).
- License exceptions run at most 365 days; audit exceptions keep 90 (spec
  §6.1).
- Release order: vergil-tooling before vergil-actions. Task 5 makes a
  missing `licenses.json` fail CI, which only Task 3's release writes (spec
  §6.6).
- Releases are human actions. Agents never cut one (spec §8).
- Validation runs only through `vrg-container-run -- vrg-validate` in the
  task repo's worktree. Commit with `vrg-commit`, and hand off with
  `vrg-pr-workflow report-ready`.
- **Live check for TypeScript command changes.** vergil-tooling's own
  validation runs no npm, so Tasks 3 and 4 each run the real thing before
  report-ready:
  - install the task branch's vergil-tooling into the current ts-node
    image inside `vrg-container-run`, for example `uv tool install` from
    the worktree;
  - in a scratch clone of `vergil-fixture-ts-app`, run `vrg-validate
    --check audit` (Task 3) or `vrg-update-deps` after the §6.4 rollback
    (Task 4) with it;
  - the stage the task changed must pass, and the output goes in the PR.

## Review Focus

1. **`(MIT OR GPL-3.0-only)` passes and `(MIT AND GPL-3.0-only)` fails.**
   Pinned in Task 3 (`test_or_any_branch`, `test_and_every_branch`).
2. **`"license": "mit"` passes, and `"mit or Apache-2.0"` is unparseable.**
   IDs match without regard to case; operators only in upper case. Pinned in
   Task 3 (`test_id_case_insensitive`, `test_lowercase_operator_unparseable`).
3. **One package locked at two versions with different licenses.** Each is
   judged on its own, and an exception for one license value does not cover
   the other. Pinned in Task 3 (`test_two_versions_two_verdicts`).
4. **A repo whose `.npmrc` sets `save=true`.** `npm update` rewrites
   `package.json`, and the updater fails, naming it. Pinned in Task 4
   (`test_changed_package_json_fails`).
5. **`npm outdated` exits 1 printing `{}`, or exits 0 printing nothing.**
   Both fail; exit 0 with `{}` is an empty held-back list. Pinned in Task 4
   (`test_outdated_exit_contract`).

---

## Delivery table

Each line gives the task, its repo and kind, what it delivers, and what it is
blocked by.

- **1** (.github, task #1340): spec and plan, this PR.
- **2** (vergil-tooling): the time-boxed exception rules factored out of
  `lib/npm_audit.py`, no behaviour change.
- **3** (vergil-tooling): `vrg-npm-license-gate` in the TypeScript AUDIT
  check. Blocked by 2.
- **4** (vergil-tooling): `NpmUpdater` and its held-back report.
- **5** (vergil-actions): `licenses.json` in the `typescript/audit`
  contract. Blocked by 3 (the file and tool names).
- **6** (.github, deployment): vergil-tooling and vergil-actions releases
  attested; the released gate writes `licenses.json`. Blocked by 3, 4 and 5.
- **7** (vergil-claude-plugin): `fixture-run` drops the TypeScript
  dependency-update known gap and gains the TypeScript rollback. Blocked by
  6.
- **8** (.github, deployment): the vergil-claude-plugin release with Task 7
  attested and installed. Blocked by 7.
- **Fixture runs** (both TypeScript fixtures): one `fixture-run` trio per
  fixture. Blocked by 6 and 8.
- **9** (.github, validation): nothing the fleet runs calls license-checker;
  its removal filed as ad-hoc work. Blocked by both fixture validations.
- **Documentation review** (vergil-tooling, bookend #4707). Blocked by 2–5
  and 7.
- **Follow-on brainstorm** (.github, bookend #959, epic 4). Not blocked; it
  is worked through planning once this epic is nearly done.
- **Retrospective** (.github, bookend #1341). Blocked by everything else.

Notes:

- **Tasks 3 and 4 are independent** and run in parallel. Both touch only
  their own modules, plus `lib/languages.py` (Task 3) and
  `lib/update_deps/orchestrator.py` (Task 4).
- **Task 7 waits for the release, not the merge.** Other epics file
  TypeScript fixture runs too. Removing the known-gap row before the
  released vergil-tooling carries the updater would fail their rollback
  check.
- **Filing.** Every task above is filed at epic-create's filing step, in its
  repo, linked under #1339, with its `Blocked-by:` refs. The fixture runs
  are filed with `fixture-run` once Tasks 6 and 8 exist; Task 9 gets their
  validation refs as `Blocked-by:` when they are filed:

  ```text
  fixture-run vergil-project/.github#1339 ts-lib,ts-app \
    --validates "TypeScript supply chain: the releases named in Task 6" \
    --after vergil-project/.github#<Task 6> \
    --after vergil-project/.github#<Task 8>
  ```

---

### Task 2: Time-boxed exception rules shared (vergil-tooling)

Spec §6.1, the shared parsing. A refactor with no behaviour change, so Task
3 builds on it rather than copying it.

**Files:**

- Create: `src/vergil_tooling/lib/exception_file.py`
- Modify: `src/vergil_tooling/lib/npm_audit.py` (uses it)
- Test: `tests/vergil_tooling/test_exception_file.py`;
  `tests/vergil_tooling/test_npm_audit.py` and
  `tests/vergil_tooling/test_vrg_npm_audit_gate.py` pass unchanged

**Interfaces:**

- Produces, in `lib/exception_file.py`:
  - `class ExceptionFileError(ValueError)` with `problems: list[str]`, moved
    from `npm_audit.py`; `npm_audit` re-exports it, so existing imports
    keep working.
  - `parse_expiry(token: str) -> datetime.date | None`: `exp:YYYY-MM-DD`, or
    `None` when malformed.
  - `is_tracking_ref(token: str) -> bool`: `#N`, `owner/repo#N` or a GitHub
    issue URL.
  - `window_problem(*, key: str, expires: datetime.date, tracking: str,
    today: datetime.date, max_days: int) -> str | None`: the expired and
    too-far-ahead messages `npm_audit.parse_exceptions` emits today, with
    the advisory ID generalized to `key`.
  - `content_lines(text: str) -> Iterator[tuple[int, str]]`: numbered,
    stripped lines, skipping blanks and `#` comments.

- [ ] **Step 1: Write the failing tests** in `test_exception_file.py`:
  `test_parse_expiry_valid_and_malformed`, `test_tracking_ref_forms`,
  `test_window_expired_message`, `test_window_too_far_message`,
  `test_content_lines_skips_comments_and_blanks`.
- [ ] **Step 2: Run** `vrg-container-run -- vrg-validate`. Expect the new
  tests to fail.
- [ ] **Step 3: Implement** the module and rewrite `npm_audit.parse_exceptions`
  on it. Every audit message stays byte-for-byte the same.
- [ ] **Step 4: Run** `vrg-container-run -- vrg-validate`. Expect PASS,
  with the audit tests untouched.
- [ ] **Step 5: Commit** `vrg-commit --type refactor --scope typescript
  --message "share the time-boxed exception rules"`, then report-ready.

### Task 3: `vrg-npm-license-gate` in the TypeScript AUDIT check (vergil-tooling)

Spec §6.1 and §6.7 (vergil-tooling docs). Blocked by 2.

**Files:**

- Create: `src/vergil_tooling/lib/npm_licenses.py`,
  `src/vergil_tooling/bin/vrg_npm_license_gate.py`
- Modify: `pyproject.toml` (`vrg-npm-license-gate =
  "vergil_tooling.bin.vrg_npm_license_gate:main"`)
- Modify: `src/vergil_tooling/lib/languages.py` (AUDIT's second command;
  `TYPESCRIPT_REPORT_FILES` gains `LICENSES_REPORT`; the allowlist and AUDIT
  comments say the gate enforces it)
- Modify: `docs/site/docs/standards/development/typescript/overview.md`
  (the license gate and `.npm-license-exceptions`),
  `docs/site/docs/guides/validation-matrix.md`,
  `docs/site/docs/standards/development/tool-ownership.md` (wording on
  gating only; license-checker stays in the image-owned column, spec §6.7)
- Test: `tests/vergil_tooling/test_npm_licenses.py`,
  `tests/vergil_tooling/test_vrg_npm_license_gate.py`,
  `tests/vergil_tooling/test_languages.py`

**Interfaces:**

- Consumes: Task 2's `lib/exception_file.py`;
  `languages._TYPESCRIPT_LICENSES_ALLOWLIST` (split on `;`).
- Produces, in `lib/npm_licenses.py`:
  - `EXCEPTIONS_FILE = ".npm-license-exceptions"`,
    `MAX_EXCEPTION_DAYS = 365`, `MISSING = "-"`.
  - `license_key(value: object) -> str`: a string as recorded; a non-string
    as `json.dumps(value, sort_keys=True, separators=(",", ":"))`; an absent
    field as `MISSING`.
  - `parse_expression(text: str) -> Expr`, raising `LicenseExpressionError`:
    IDs, `+`, `WITH`, `AND`, `OR`, parentheses; `AND` binds tighter than
    `OR`. Operators are upper case only.
  - `allowed(expr: Expr, allowlist: Iterable[str]) -> bool`: IDs compared
    case-insensitively; a `WITH` term only as a whole allowlist entry; `+`
    only against an entry spelled with `+`.
  - `@dataclass(frozen=True) class LockedPackage: name: str; version: str;
    license: object; dev: bool` (`license` is the raw value, or `None` when
    absent).
  - `read_lockfile(path: Path) -> list[LockedPackage]`, raising
    `LockfileError` naming the file and `npm install --package-lock-only`
    for a missing, unreadable or pre-v2 lockfile or an entry with no
    `version`. Skips the `""` root and `"link": true` entries, each with a
    comment giving the reason. The name is `name` when present, else the
    last `node_modules/` segment of the key.
  - `@dataclass(frozen=True) class LicenseException: package: str;
    license: str; expires: date; tracking: str; reason: str; line: int`.
  - `parse_exceptions(text: str, *, today: date, source: str) ->
    list[LicenseException]`. A line is `<package> <value> exp:<date>
    <tracking> <reason>`, where `<value>` is a JSON string literal (the
    recorded value, quoted and escaped) or the bare `-` for a missing field.
    Duplicate means the same `(package, license)` twice.
  - `judge(packages, *, allowlist, exceptions) -> LicenseVerdict` with
    `rows`, `applied`, `unmatched` and `failing`. A package is covered when
    an exception's `package` equals its name and `license` equals
    `license_key(package.license)`.
  - `report(verdict) -> list[dict]`, written as `licenses.json`: one object
    per package with `name`, `version`, `license` (canonical spelling when
    parsed, else the raw key), `dev`, `verdict` (`allowed`, `excepted` or
    `failing`), and `exception` (`tracking`, `expires`) when excepted.
- Produces: `vrg-npm-license-gate [--lockfile PATH] [--output PATH]`
  (defaults `package-lock.json` and `licenses.json`). It writes the report
  whenever the lockfile was read, then exits 1 on any failing package or bad
  exception line, and 0 otherwise. Failures list name, version, license
  value and reason, then name `.npm-license-exceptions` and the allowlist as
  the fixes. Applied and unmatched exceptions are listed, unmatched as
  warnings.
- Produces: the TypeScript AUDIT commands are `["sh", "-c",
  _TYPESCRIPT_AUDIT_SCRIPT]` then `["vrg-npm-license-gate"]`.

- [ ] **Step 1: Write the failing tests:**
  - `test_or_any_branch`, `test_and_every_branch`,
    `test_with_needs_whole_term`, `test_plus_needs_plus_entry`,
    `test_id_case_insensitive`, `test_lowercase_operator_unparseable`,
    `test_non_spdx_strings_unparseable` (`SEE LICENSE IN LICENSE`,
    `UNLICENSED`, `Custom`);
  - `test_license_key_forms`: `"MIT"`, `{"type": "MIT", "url": "u"}` as
    `{"type":"MIT","url":"u"}`, and absent as `-`;
  - `test_read_lockfile_skips_root_and_links`,
    `test_read_lockfile_name_from_key`, `test_pre_v2_lockfile_fails`,
    `test_entry_without_version_fails`;
  - `test_exception_covers_only_its_value`: an entry for `"SEE LICENSE IN
    LICENSE"` does not cover the same package recorded as `GPL-3.0-only`;
  - `test_two_versions_two_verdicts`;
  - `test_missing_license_needs_dash_entry`;
  - `test_exception_window_365`, `test_duplicate_key_fails`,
    `test_unmatched_entry_warns`, `test_entry_for_passing_package_warns`;
  - `test_gate_writes_report_on_failure` and `test_gate_exit_codes`, through
    `main()`;
  - `test_typescript_audit_runs_license_gate`, and
    `test_typescript_report_files_include_licenses`.
- [ ] **Step 2: Run** `vrg-container-run -- vrg-validate`. Expect the new
  tests to fail.
- [ ] **Step 3: Implement** the interfaces above, and update the docs.
- [ ] **Step 4: Run** `vrg-container-run -- vrg-validate`. Expect PASS.
- [ ] **Step 4b: Live check** (Global Constraints) against
  `vergil-fixture-ts-app`: AUDIT passes, and `licenses.json` lists 176
  packages, every one `allowed`. Record the output in the PR.
- [ ] **Step 5: Commit** `vrg-commit --type feat --scope typescript
  --message "gate TypeScript dependency licenses on the lockfile"`, then
  report-ready.

### Task 4: `NpmUpdater` and its held-back report (vergil-tooling)

Spec §6.2 and §6.7 (vergil-tooling docs).

**Files:**

- Create: `src/vergil_tooling/lib/update_deps/updaters/npm.py`
- Modify: `src/vergil_tooling/lib/update_deps/orchestrator.py`
  (`NpmUpdater()` after `CargoUpdater()` in `DEFAULT_REGISTRY`)
- Modify: `docs/site/docs/reference/dev/update-deps.md` (the npm updater and
  its held-back notes), `docs/site/docs/standards/development/typescript/overview.md`
  (dependency updates)
- Test: `tests/vergil_tooling/test_update_deps_npm.py`,
  `tests/vergil_tooling/test_update_deps_orchestrator.py`,
  `tests/vergil_tooling/test_update_deps_registry.py`

**Interfaces:**

- Consumes: `UpdateResult`, `HeldBack`, `changes.diff_versions`,
  `UpdateDepsError`, `progress.run`; `configs/typescript/dev-dependencies.json`.
- Produces, in `updaters/npm.py`:
  - `UPDATE_CMD = ("vrg-container-run", "--", "npm", "update",
    "--ignore-scripts")`, `CI_CMD = ("vrg-container-run", "--", "npm", "ci",
    "--ignore-scripts")`, `OUTDATED_CMD = ("vrg-container-run", "--", "npm",
    "outdated", "--json")`.
  - `COMMIT_SUBJECT = "chore(deps): npm update"`,
    `REFRESH_COMMIT_MESSAGE = "chore(deps): refresh package-lock.json"`.
  - `OUTSIDE_RANGE = "outside the package.json range"`,
    `HELD_BY_RESOLVER = "held by the resolver: another requirement in the
    dependency graph"`, and `vergil_major_note(name: str, major: int) -> str`
    returning `f"Vergil accepts {name} {major} only (dev-dependencies.json)"`;
    notes are joined with `"; "`.
  - `locked_package_versions(lock: dict) -> dict[str, list[str]]`, from the
    v2/v3 `packages` map, raising `UpdateDepsError` naming the lockfile when
    it is pre-v2.
  - `outdated(run=run_captured) -> dict[str, dict]`: runs `CI_CMD` then
    `OUTDATED_CMD`. The JSON document is stdout from the first line
    beginning `{`, after the `vrg-container-run` header. Exit 0 with `{}`, or
    exit 1 with a non-empty object, is normal; anything else (no JSON,
    malformed JSON, an `error` key, exit 1 with `{}`, exit 0 with a
    non-empty object, a failed `npm ci`) raises `UpdateDepsError`.
  - `held_back(ctx, run=run_captured) -> list[HeldBack]` with
    `source="npm"`, over `dependencies`, `devDependencies` and
    `optionalDependencies`. A missing `current` is taken from the lockfile's
    `node_modules/<name>` entry; a name missing there too raises. `file:`,
    `link:`, git and URL specs are skipped, with the reason in a comment.
  - `class NpmUpdater` with `name = "npm"`; `applies` needs `package.json`
    and `package-lock.json`; `apply` reads both, runs `UPDATE_CMD`, fails if
    `package.json` changed by a byte, fails naming any added or moved entry
    with no `license`, itemizes moves with `diff_versions`, and fills
    `held_back`. Summary and commit-message shapes follow `CargoUpdater`.

- [ ] **Step 1: Write the failing tests,** with a fake runner:
  - `test_npm_applies_needs_both_files`;
  - `test_update_runs_in_container_with_ignore_scripts`;
  - `test_update_reports_moved_packages`, and
    `test_lock_rewritten_without_moves` (refresh message, `packages=[]`);
  - `test_changed_package_json_fails`;
  - `test_moved_entry_without_license_fails`;
  - `test_update_failure_raises`, `test_pre_v2_lockfile_fails`;
  - `test_outdated_exit_contract`: the five exit/output cases above;
  - `test_outdated_after_header_lines`: header lines before the JSON are
    ignored;
  - `test_held_back_outside_range`: `@types/node` `^24`, current and wanted
    24.19.2, latest 26.6.6;
  - `test_held_back_vergil_major`: `typescript` `^6.0.3`, latest 7.0.2,
    note `outside the package.json range; Vergil accepts typescript 6 only
    (dev-dependencies.json)`;
  - `test_held_back_by_resolver`: current below wanted;
  - `test_missing_current_from_lockfile`, `test_missing_everywhere_fails`;
  - `test_non_registry_specs_skipped`;
  - `test_npm_in_default_registry`, and `test_orchestrator_records_npm_failures`.
- [ ] **Step 2: Run** `vrg-container-run -- vrg-validate`. Expect the new
  tests to fail.
- [ ] **Step 3: Implement** the interfaces above, and update the docs.
- [ ] **Step 4: Run** `vrg-container-run -- vrg-validate`. Expect PASS.
- [ ] **Step 4b: Live check** (Global Constraints): roll `@babel/parser`
  back in a scratch clone of `vergil-fixture-ts-app` with the spec §6.4
  sequence, run `vrg-update-deps`, and confirm it restores the package, leaves
  `package.json` unchanged, and reports `typescript` and `@types/node` held
  back with their notes. Record the output in the PR.
- [ ] **Step 5: Commit** `vrg-commit --type feat --scope update-deps
  --message "add the npm updater and its held-back report"`, then
  report-ready.

### Task 5: `licenses.json` in the `typescript/audit` contract (vergil-actions)

Spec §6.3. Blocked by 3. The PR touches `.github/workflows/**`, so GitHub
refuses the agent push; the human's `vrg-submit-pr` pushes it.

**Files:**

- Modify: `actions/ci/evidence/reports/reports.sh` (`report_files` and
  `report_tools` for `typescript/audit`, and the header table)
- Modify: `.github/workflows/ci-audit.yml` (the staging comment at line 129
  names TypeScript's `npm-audit.json` and `licenses.json`)
- Modify: `docs/site/docs/actions/ci-evidence-reports.md`,
  `docs/site/docs/workflows/ci-audit.md`
- Test: `actions/ci/evidence/reports/tests/reports.test.sh`

**Interfaces:**

- Consumes: Task 3's file name `licenses.json` and tool name
  `vrg-npm-license-gate`.
- Produces: `report_files typescript audit` prints `npm-audit.json` then
  `licenses.json`; `report_tools typescript audit` prints
  `[{"name":"npm audit"},{"name":"vrg-npm-license-gate"}]`.

- [ ] **Step 1: Write the failing tests** in `reports.test.sh`: the two
  outputs above.
- [ ] **Step 2: Run** `vrg-container-run -- vrg-validate`. Expect them to
  fail.
- [ ] **Step 3: Implement,** and update the docs and the staging comment.
- [ ] **Step 4: Run** `vrg-container-run -- vrg-validate`. Expect PASS.
- [ ] **Step 5: Commit** `vrg-commit --type feat --scope evidence
  --message "stage the TypeScript license report"`, then report-ready.

### Task 6: Releases attested, `licenses.json` written (deployment, .github)

Blocked by 3, 4 and 5.

- **Precondition (human-attested):** the releases are published and their
  versions named in a comment: vergil-tooling (Tasks 2–4) first, then
  vergil-actions (Task 5).
- **Procedure (agent-safe):**
  1. Confirm each release's notes list its tasks' PRs.
  2. Confirm `vergil = "v2.1"` resolves to the named vergil-tooling
     release.
  3. In a scratch clone of `vergil-fixture-ts-app`, run
     `vrg-container-run -- vrg-validate --check audit` with the released
     vergil-tooling: it runs `vrg-npm-license-gate` and writes
     `licenses.json`.
- **SUCCESS:** every check passes, recorded with the release versions.

### Task 7: `fixture-run` drops the TypeScript known gap (vergil-claude-plugin)

Spec §6.4. Blocked by 6.

**Files:**

- Modify: `skills/fixture-run/SKILL.md` (the known-gaps table loses its
  TypeScript "Dependency update" row; the change-task template's
  `<TypeScript:>` "Dependency rollback: skipped" paragraph becomes a
  rollback paragraph; the lead-in that says TypeScript skips the rollback
  is corrected)

**Interfaces:**

- Produces: the TypeScript rollback paragraph, in the shape of Python's and
  Rust's. It names spec §6.4's three-command sequence as the example, says
  `package.json` must be unchanged before commit, says the older version
  must pass the npm audit and the license gate, steers to a transitive
  package (both fixtures lock every direct dependency at its range floor),
  and records `Rolled back: <package> <locked version> -> <older>
  (transitive)` or `(declared <range>)`.

- [ ] **Step 1: Edit** the skill as above. With no known gap left, the
  table section says there are none today, keeping its rules for future
  rows.
- [ ] **Step 2: Run** `vrg-container-run -- vrg-validate`. Expect PASS.
- [ ] **Step 3: Commit** `vrg-commit --type feat --scope fixture-run
  --message "turn on the TypeScript dependency rollback"`, then
  report-ready.

### Task 8: Plugin release attested and installed (deployment, .github)

Blocked by 7.

- **Precondition (human-attested):** the vergil-claude-plugin release that
  carries Task 7 is published, its version named in a comment.
- **Procedure (agent-safe):** confirm the installed plugin's `fixture-run`
  skill has no TypeScript known-gap row and carries the TypeScript rollback
  paragraph.
- **SUCCESS:** recorded with the plugin version.

### Fixture runs (both TypeScript fixtures)

Filed with `fixture-run` (see Filing), blocked by Tasks 6 and 8. Beyond the
standard trio, each run checks:

- The change task rolls one transitive package back in `package-lock.json`
  with the spec §6.4 sequence, `package.json` unchanged, and the change PR's
  `npm ci` and gates pass. The release's `vrg-update-deps` restores it with
  an npm update commit naming the package. This replaces the
  dependency-update known gap.
- The AUDIT check runs `vrg-npm-license-gate`, which passes, and the
  evidence bundle's `audit` gate holds `npm-audit.json` and `licenses.json`
  and names `npm audit` and `vrg-npm-license-gate`.
- Every other check passes, with no known gap.

### Task 9: license-checker's removal filed (validation, .github)

Spec §6.5. Blocked by both fixture validations (their refs are added as
`Blocked-by:` when `fixture-run` files them).

- **Precondition (machine probe):** both fixture validations closed on
  `SUCCESS`, and the TypeScript AUDIT commands in the vergil-tooling
  release `vergil = "v2.1"` resolves to do not name `license-checker`
  (`lib/languages.py` at that tag).
- **Procedure:**
  1. File the containers task: `vrg-issue-create --epic adhoc --repo
     vergil-project/vergil-containers --title "Remove license-checker from
     the ts-node images"`, its body naming `typescript-analysis.dockerfile`,
     `checks.sh`, the smoke tests and the ts-node docs, and the vergil-tooling
     task below.
  2. File the docs task: `vrg-issue-create --epic adhoc --repo
     vergil-project/vergil-tooling --title "Drop license-checker from the
     docs once the ts-node images no longer ship it" --blocked-by
     <the containers task>`, its body listing
     `docs/site/docs/guides/ci-architecture.md`,
     `docs/site/docs/standards/development/typescript/overview.md` and the
     TypeScript row of `docs/site/docs/standards/development/tool-ownership.md`.
- **SUCCESS:** both tasks filed, their refs recorded in the result.
