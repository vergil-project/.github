# TypeScript supply chain and the npm updater — design

- **Epic:** vergil-project/.github#1339
- **Planning issue:** vergil-project/.github#1209 (review history is in its comments)
- **Status:** draft, in design review

## 1. Intent

A TypeScript repo's supply chain is gated, evidenced and kept current with
the same rigour as a Python or Rust repo:

- its dependencies' licenses are checked against Vergil's allowlist, the
  check fails the gate, and it leaves a report the CI evidence stages;
- `vrg-update-deps` updates its `package-lock.json` within the ranges in
  `package.json`, and reports each direct dependency those ranges hold back;
- the TypeScript fixture runs prove both, and the dependency-update known gap
  they have recorded since epic vergil-project/.github#1174 is closed.

## 2. Why

- **This is epic 3 of the TypeScript-parity wave.** Epic
  vergil-project/.github#1174 carried TypeScript through the full cycle, and
  epic vergil-project/.github#1281 put Node on reviewed lifecycle PRs and
  enforced the toolchain. This epic is the TypeScript counterpart of the Rust
  epic vergil-project/.github#1165, which did the same for Cargo. Epic 4, the
  Node runtime architecture (vergil-project/.github#959), follows it.
- **The license check gates nothing.** The packaged TypeScript allowlist has
  existed since epic vergil-project/.github#284, but the AUDIT check runs
  `license-checker --production --summary`, which prints a summary, never
  fails, and leaves no report. Python's `pip-licenses --allow-only` and Rust's
  `cargo deny` both gate and leave evidence.
- **The fixture runs cannot prove the dependency update.** Every TypeScript
  fixture run so far (vergil-fixture-ts-lib #4, #12, #15, #25;
  vergil-fixture-ts-app #4, #12, #15, #26) recorded check 5, the dependency
  update, as a known gap: with no npm updater, the change task skips the
  rollback and the release's dependency update opens no PR. The evidence is
  in the comments on vergil-project/.github#1209.
- **TypeScript's part of vergil-project/.github#384,** which restarts the
  per-language updaters. Rust's part shipped in #1165.

## 3. Goals, non-goals, anti-goals

### Goals

1. The TypeScript AUDIT check fails when any package in `package-lock.json`,
   dev dependencies included, has a license the allowlist does not allow, or
   no license, unless a live entry in the repo's `.npm-license-exceptions`
   covers it.
2. The license check writes `licenses.json`, and the CI report contract's
   `typescript/audit` entry stages it and names its tool.
3. `vrg-update-deps` updates a TypeScript repo's `package-lock.json` within
   its `package.json` ranges, never editing `package.json`, and reports each
   direct dependency the ranges hold back, with its reason.
4. The `fixture-run` skill's TypeScript dependency-update known gap is
   removed, and the TypeScript change task rolls a package back in the
   lockfile, as Python and Rust do.
5. Fixture runs on `vergil-fixture-ts-lib` and `vergil-fixture-ts-app` pass
   every check, the dependency update and the license gate included, with no
   known gap.
6. Once those runs pass, the removal of license-checker from the ts-node
   images is filed as an ad-hoc task in vergil-containers, with a matching
   ad-hoc doc task in vergil-tooling (§6.5).

### Non-goals

- **Editing `package.json` ranges,** or moving a dependency to a new major
  (`npm update --save`, `npm-check-updates`). Python's and Rust's updaters do
  not edit their manifests either.
- **Package managers other than npm,** and npm workspaces. A TypeScript repo
  is one package, as #1174 defines it.
- **Removing license-checker from the images.** The ad-hoc task filed by §6.5
  does it, outside this epic, so the rollout waits on no image release.
- **The TypeScript semgrep ruleset.** It is already in vergil-tooling (§4).
- **Removing the fixtures' stale picomatch audit exceptions:** fixture tasks
  vergil-project/vergil-fixture-ts-lib#20 and
  vergil-project/vergil-fixture-ts-app#20.
- **The Node runtime architecture:** epic 4, vergil-project/.github#959, this
  epic's follow-on brainstorm.
- **TypeScript 7:** vergil-project/.github#1181.

### Anti-goals (the epic fails if any of these happen)

- A package with a disallowed, unparseable or missing license passes the
  license gate without a live exception naming it and the exact license value
  it records.
- The license gate reads anything but the committed `package-lock.json`, so
  its verdict depends on what happens to be installed.
- The npm updater edits `package.json`, runs an install script, swallows a
  failure, or reports "no change" when `npm update` failed.
- A held-back list is silently empty because a registry read failed.
- A TypeScript repo turns red because vergil-actions requires `licenses.json`
  before the vergil-tooling release that writes it is out (§6.6).
- license-checker leaves the images while the vergil-tooling version the
  fleet's `vergil = "v2.1"` pin resolves to still calls it.
- A Python or Rust gate, report or updater changes behaviour.

## 4. Current state (facts the design builds on)

Checked on `origin/develop` on 2026-11-18: vergil-tooling `0185c9dc7b`, and
vergil-actions, vergil-containers and vergil-claude-plugin at their develop
heads that day.

### vergil-tooling

- **The TypeScript AUDIT check** (`lib/languages.py`, the `"typescript"`
  `Language`) runs two commands:
  - `_TYPESCRIPT_AUDIT_SCRIPT`: `npm audit --audit-level=high --json
    >npm-audit.json`, then`vrg-npm-audit-gate npm-audit.json --npm-exit
    "$rc"`. The audit covers the whole locked tree, dev dependencies included
    (#3552).
  - `["license-checker", "--production", "--summary"]`: non-gating, runtime
    dependencies only, no report. Its comment calls hardened gating against
    `_TYPESCRIPT_LICENSES_ALLOWLIST` deferred.
- **The allowlist** `_TYPESCRIPT_LICENSES_ALLOWLIST` (`lib/languages.py`) is a
  `;`-joined list of SPDX IDs: `0BSD`, `Apache-2.0`, `BlueOak-1.0.0`,
  `BSD-2-Clause`, `BSD-3-Clause`, `CC0-1.0`, `ISC`, `MIT`, `MPL-2.0`,
  `Python-2.0`, `Unlicense`. It sits beside Python's and C++'s allowlists as
  their one source of truth.
- **The audit exceptions** (`lib/npm_audit.py`, `bin/vrg_npm_audit_gate.py`):
  `.npm-audit-exceptions` holds one entry per line, `<GHSA-ID>
  exp:<YYYY-MM-DD> <tracking issue> <reason>`. An entry may expire at most
  `MAX_EXCEPTION_DAYS` (90) days ahead. A malformed, expired, duplicate or
  too-long entry fails the gate by name, and every bad entry is reported in
  one run. An entry no finding matches is reported as a warning so it gets
  removed.
- **The lockfile records each package's license.** npm 7 and later write a
  `license` field into each `packages` entry of a v2 or v3 lockfile, from
  the package's own `package.json`. `vergil-fixture-ts-app`'s lockfile
  (v3) has 177 entries besides the root, 174 of them dev dependencies, and
  every one carries a `license`: MIT, Apache-2.0, MPL-2.0, BSD-2-Clause, ISC,
  BlueOak-1.0.0, BSD-3-Clause and 0BSD, all on the allowlist.
- **`vrg-update-deps`** (`lib/update_deps/`):
  - `DEFAULT_REGISTRY` (`orchestrator.py`) holds `PythonUvUpdater`,
    `CargoUpdater`, `VergilUpdater`, `ImagesUpdater` and `RepoHookUpdater`.
    It has no npm updater.
  - The `Updater` protocol (`updater.py`): `name`, `applies(ctx)`,
    `apply(ctx) -> UpdateResult`. `UpdateResult` carries `changed`,
    `summary`, `commit_message`, `warnings`, `held_back: list[HeldBack]` and
    `packages: list[PackageChange] | None`.
  - `HeldBack` (`report.py`): `source`, `name`, `current`, `latest`, `note`.
  - `CargoUpdater` (`updaters/cargo.py`) is the model: it runs `cargo update`
    through `vrg-container-run`, itemizes the moves by diffing the parsed
    lockfile before and after (`changes.diff_versions`), and builds the
    held-back report from Cargo's sparse index, failing loudly on every
    error, an index read included.
- **Vergil's accepted majors** for the repo's own TypeScript tools are in
  `configs/typescript/dev-dependencies.json` (`typescript` 6,
  `typescript-eslint` 8, `eslint` 10, `@eslint/js` 10, `vitest` 5,
  `@vitest/coverage-v8` 5). `vrg-validate`'s pre-gate stage checks the repo's
  declared ranges against them (#1174 §6.3).
- **Every `npm ci` Vergil runs passes `--ignore-scripts`** (#1174's global
  constraint).
- **semgrep is already done.** `lib/semgrep.py` maps `typescript` to
  `p/typescript`, and vergil-tooling#4478 made `vrg-semgrep-scan` the one
  source of that ruleset. CI's semgrep action
  (`vergil-actions/actions/ci/security/semgrep/action.yml`) calls
  `vrg-semgrep-scan`. #1209's semgrep item needs no task.
- **`licenses.json` is already ignored** by the base gitignore fragment
  (`data/gitignore/base`), for Python's report.

### vergil-actions

- `actions/ci/evidence/reports/reports.sh` is the per-language report
  contract. `typescript/audit` lists `npm-audit.json` as its file and `npm
  audit` as its tool. `python/audit` lists `pip-audit.json` and
  `licenses.json`.
- CI's audit job runs `vrg-validate --check audit` (`ci-audit.yml`), so CI
  and local validation share the commands in vergil-tooling.
- Since #1165, a passing leg whose contracted report is missing fails its
  staging step. A contract entry that names a file the released vergil-tooling
  does not write therefore turns every TypeScript repo red (§6.6).

### vergil-containers

- license-checker 25.0.1 is installed globally in the ts-node images
  (`docker/common/typescript-analysis.dockerfile`) and checked by the image's
  `checks.sh` and smoke tests. It stays there through this epic (§6.5).

### vergil-claude-plugin

- The `fixture-run` skill's known-gaps table has one row: TypeScript,
  Dependency update, recorded on vergil-project/.github#1209. Removing a
  dependency-update row turns that language's rollback on.
- Its change-task template has a rollback paragraph for Python and Rust, and
  a "Dependency rollback: skipped" paragraph for TypeScript.

### Docs that name license-checker as a gate tool

- vergil-tooling: `docs/site/docs/standards/development/tool-ownership.md`
  (the TypeScript row's image-owned tools) and
  `docs/site/docs/standards/development/typescript/overview.md`.
- vergil-actions: `docs/site/docs/actions/ci-evidence-reports.md` and
  `docs/site/docs/workflows/ci-audit.md` (the per-language audit evidence).

## 5. Approaches considered

### 5.1 What runs the license check

- **A. `license-checker --onlyAllow` (the image tool).** The smallest change.
  Rejected: license-checker 25.0.1 has had no release in years, reads
  `node_modules` rather than the lockfile, matches license strings loosely
  instead of evaluating SPDX expressions, exits 1 on `--version`, and shows a
  private root as `UNLICENSED`. Exceptions and evidence would still be Vergil
  code bolted around it.
- **B. A Vergil-owned gate over `package-lock.json` (chosen).** It reads
  exactly the committed tree the audit scans, needs no `node_modules`,
  evaluates SPDX expressions properly, and reuses the audit exceptions'
  rules. It follows `vrg-npm-audit-gate`, which Vergil already owns.
- **C. A maintained fork (`license-checker-rseidelsohn`) on the image.**
  Rejected: it keeps A's loose matching and bolted-on exceptions.

### 5.2 Which packages the license check covers

- **Runtime dependencies only** (today's `--production`): fewer packages and
  fewer metadata surprises, but narrower than the audit. Rejected.
- **The whole locked tree (chosen),** matching the audit and Rust's
  `cargo deny`. The large dev toolchain is why the repo-level exceptions file
  exists.

### 5.3 How to cut the epic

- **One epic, tasks grouped by repo, one release round, then the fixture
  runs (chosen).** #1165's shape: every change ships behind one release of
  vergil-tooling and vergil-actions, and one round of fixture runs proves it
  all.
- **Two epics, license gate then updater.** Rejected: two release and fixture
  rounds for a small wave.
- **Removing license-checker from the images inside the epic.** Rejected: a
  vergil-containers release adds an image publish to the rollout for no
  benefit to the proof. It is filed as an ad-hoc task once nothing calls the
  tool (§6.5).

## 6. Design

### 6.1 The license gate (vergil-tooling)

- **`vrg-npm-license-gate`, a new command,** with its logic in a new library
  module beside `lib/npm_audit.py`. It takes the lockfile path (default
  `package-lock.json`) and writes `licenses.json`.
- **The AUDIT check runs it** in place of `license-checker --production
  --summary`, after the npm audit script. Both commands always run, so one
  `vrg-validate` run shows the audit's and the license gate's findings.
- **What it reads.** Every entry in the lockfile's `packages` map except:
  - the root entry `""`, which is the repo itself;
  - entries with `"link": true`, which point at a directory listed
    separately (none occur without workspaces, a non-goal; the skip and its
    reason are recorded in the code).

  Dev and runtime dependencies are both checked (§5.2). The package name is
  taken from the entry's `name` field when present, otherwise from the last
  `node_modules/` segment of its key, and its version from `version`. The
  same package at two versions is two packages to the gate.
- **A lockfile older than v2** (no `packages` map), a missing or unreadable
  lockfile, and an entry with no `version` each fail, naming the file and the
  fix (`npm install --package-lock-only` with a current npm).
- **The verdict on each package's `license`:**
  - It is parsed as an SPDX license expression: license IDs, an optional `+`,
    `WITH` exceptions, `AND`, `OR` and parentheses. A legacy object or array
    form, and any string that is not a valid expression (`SEE LICENSE IN …`,
    `UNLICENSED`, `Custom`), is unparseable.
  - An `OR` expression is allowed when any branch is allowed. An `AND`
    expression is allowed only when every branch is. A `WITH` term is allowed
    only when the whole term, ID and exception, is on the allowlist, so a
    bare ID never admits an exception clause the allowlist has not named.
  - License and exception IDs are matched against the allowlist without
    regard to case, and the operators `AND`, `OR` and `WITH` only in upper
    case, as the SPDX license-expression syntax specifies
    ([SPDX 2.3, Annex D](https://spdx.github.io/spdx-spec/v2.3/SPDX-license-expressions/)).
    `licenses.json` records the allowlist's canonical spelling of each ID. An
    ID with `+` matches only an allowlist entry spelled with `+`.
  - A package passes when its expression is allowed. It fails when its
    expression is disallowed, unparseable or missing, unless a live exception
    covers it.
- **The allowlist stays `_TYPESCRIPT_LICENSES_ALLOWLIST`** in
  `lib/languages.py`, with its comment updated to say the gate enforces it.
  The gate imports it, so there is one source of truth, beside Python's and
  C++'s. Changing the allowlist is a vergil-tooling change, reviewed like any
  other.
- **`.npm-license-exceptions`,** the license counterpart of
  `.npm-audit-exceptions`, with the same rules:
  - one entry per line, `#` comments and blank lines ignored: a package
    name, the license value it excuses, then `exp:<YYYY-MM-DD> <tracking
    issue> <reason>`;
  - the license value is the lockfile's `license` string exactly as
    recorded, or a marker for a missing field. Values can contain spaces
    (`(MIT OR CC0-1.0)`), so the value is delimited; the exact syntax is left
    to the builder (§10);
  - an entry covers a locked package only when its name matches and its
    recorded `license` equals the entry's value exactly. It covers every
    locked version whose license is unchanged, so a version move alone needs
    no new review, and a version whose license differs is not covered and
    fails;
  - a package may have more than one entry, one per excused license value;
  - the tracking issue takes the same forms as the audit file's;
  - an entry may expire at most 365 days ahead, a named constant of its own.
    The audit's 90 days (`MAX_EXCEPTION_DAYS`) fit an advisory waiting on an
    upstream fix. A license quirk, such as a non-SPDX string on an otherwise
    allowed package, is usually permanent, so a yearly review is enough;
  - a malformed, expired, duplicate or too-long entry fails the gate by line,
    every bad entry in one run;
  - an entry that matches no package, or matches only packages that pass on
    their own, is a warning naming it, so a stale entry gets removed.

  The parsing rules both files share (expiry, tracking issue, the one-run
  report of every bad line) are factored out of `lib/npm_audit.py` rather
  than duplicated. Each file keeps its own key: a GHSA ID for the audit, a
  package name and license value for licenses. A duplicate is the same key
  twice.
- **`licenses.json`** lists every package the gate checked, with its name,
  version, license as recorded (or none), whether it is a dev dependency, and
  its verdict: allowed, excepted (with the entry's tracking issue and expiry),
  or failing. It is written whenever the lockfile could be read, including
  when the gate fails, so the evidence records the failure.
- **Output.** A failing run lists each failing package with its version, its
  license value and why it fails, and names `.npm-license-exceptions` and the
  allowlist as the two fixes. Every applied exception is listed, as the audit
  gate does.

### 6.2 The npm updater (vergil-tooling)

- **`NpmUpdater` in `lib/update_deps/updaters/npm.py`,** registered in
  `DEFAULT_REGISTRY` and implementing the `Updater` protocol.
  - `applies(ctx)` needs both `package.json` and `package-lock.json`.
  - `apply(ctx)` runs `vrg-container-run -- npm update --ignore-scripts`.
    Since npm 8.3.2, `npm update` moves each locked package to the newest
    version its range allows and leaves the ranges in `package.json`
    unchanged unless `--save` is given
    ([npm-update](https://docs.npmjs.com/cli/v11/commands/npm-update)).
    `--ignore-scripts` keeps Vergil's rule that no dependency's install
    script runs.
  - The updater checks that `package.json` is byte-for-byte unchanged after
    the run, and fails if it is not. That guards the non-goal against a
    `save` setting in a repo's `.npmrc`.
  - It itemizes the moves by diffing the lockfile's packages before and
    after with `changes.diff_versions`. Its summary and commit message name
    each package that moved, from and to, in the style of the uv and Cargo
    updaters.
- **The held-back report uses npm's own semver.** Vergil has no npm range
  evaluator and gets none: after the update, the updater runs
  `vrg-container-run -- npm ci --ignore-scripts` and then
  `vrg-container-run -- npm outdated --json`, so npm itself matches every
  range, dist-tag specs included. For each direct dependency in
  `package.json` (`dependencies`, `devDependencies` and
  `optionalDependencies`) that `npm outdated` reports:
  - `current` is its locked version, `wanted` the newest version its range
    allows, and `latest` the registry's `latest` dist-tag, the version npm
    itself treats as newest;
  - it is held back when `current` is below `latest`. Each `HeldBack` entry
    carries `current` and `latest` and a note with its reason:
    - "outside the package.json range" when `wanted` is below `latest`;
    - "held by the resolver: another requirement in the dependency graph"
      when `current` is below `wanted` after the update (a peer requirement
      or an `overrides` entry, for example);
    - when the package is in `dev-dependencies.json`, the note adds that
      Vergil accepts only its listed major there, so a reader sees the hold
      is Vergil's (TypeScript at 6, for example, until
      vergil-project/.github#1181).

    Every entry carries a reason.
  - What `npm outdated` reports for a dependency that is not a registry
    package (`file:`, `link:`, git and URL specs) is skipped, with the reason
    recorded in the code.
  - `npm outdated` exits non-zero whenever any dependency is outdated, so its
    exit status alone is neither success nor failure. The updater reads its
    JSON: a non-zero exit with a well-formed report of outdated packages is
    the normal case, and anything else (no JSON, malformed JSON, an `error`
    object, or a non-zero exit with nothing outdated) fails.
- **Failures are loud.** Each of these fails the updater's result, and none
  reads as "no change":
  - a failed `npm update`, or a failed `npm ci` before `npm outdated`;
  - an unreadable `package.json` or `package-lock.json`, or a lockfile older
    than v2;
  - a changed `package.json`;
  - an `npm outdated` run that fails as above, including a registry it cannot
    reach.

### 6.3 Evidence (vergil-actions)

- The report contract (`reports.sh`) for `typescript/audit` lists
  `npm-audit.json` and `licenses.json`, and its tools are `npm audit` and
  `vrg-npm-license-gate`. Its tests cover the new entry.
- The evidence docs (`ci-evidence-reports.md`, `ci-audit.md`) describe the
  TypeScript license report alongside Python's.
- This change is released only after the vergil-tooling release that writes
  `licenses.json` (§6.6).

### 6.4 The fixture runs (vergil-claude-plugin, the fixtures)

- **The `fixture-run` skill** loses its TypeScript "Dependency update"
  known-gap row. Its change-task template replaces TypeScript's "Dependency
  rollback: skipped" paragraph with a rollback paragraph in the shape of
  Python's and Rust's:
  - move one package back to an earlier version in `package-lock.json`
    only, leaving `package.json` unchanged, so `npm ci` still passes and the
    release's dependency update has a guaranteed upgrade to make;
  - that means a transitive package, or a direct dependency whose range's
    floor is below its locked version;
  - the earlier version must pass the npm audit and the license gate, so the
    PR never goes red for the wrong reason;
  - record it as `Rolled back: <package> <locked version> -> <older>
    (declared <range>)`, or `(transitive)` for a package `package.json` does
    not name.

  The exact npm command for the rollback is left to the task (§10).
- **This plugin change is released** before the fixture runs are filed, as
  in #1165, so each run's change task is built from the new template.
- **Fixture runs** on `vergil-fixture-ts-lib` and `vergil-fixture-ts-app`,
  filed with `fixture-run`, blocked by the deployment task that records the
  vergil-tooling and vergil-actions releases (§6.6) and by the plugin
  release. Each run:
  - rolls one package back and checks that the release's `vrg-update-deps`
    restores it with an npm update commit;
  - passes the license gate, and its evidence bundle's `audit` gate holds
    `licenses.json` and names `vrg-npm-license-gate`;
  - passes every other check, with no known gap.

### 6.5 license-checker leaves the images, after the proof

- Once both fixture runs pass, the vergil-tooling the fleet runs no longer
  calls license-checker: repos float on `vergil = "v2.1"`, which resolves to
  the newest release, its gate no longer calls the tool, and the tool was
  never a repo devDependency.
- A step of this epic then **files** two ad-hoc tasks, each in the repo its
  PR lands in, so the docs follow the image:
  - in vergil-containers (`vrg-issue-create --epic adhoc --repo
    vergil-project/vergil-containers`): take license-checker out of
    `typescript-analysis.dockerfile`, `checks.sh`, the smoke tests and the
    ts-node docs;
  - in vergil-tooling: once that removal is released, drop license-checker
    from the docs that say the image ships it (`guides/ci-architecture.md`,
    `standards/development/typescript/overview.md` and the TypeScript row of
    `standards/development/tool-ownership.md`, as of this spec). It is
    blocked by the vergil-containers task, and that task's body names it and
    lists those lines.

  Both are processed outside the epic, so the epic never waits on an image
  release.
- **Never before the proof.** Filed any earlier, the ad-hoc task could remove
  the tool while every TypeScript repo's AUDIT still calls it.
- The step's precondition is that both fixture validations closed on
  `SUCCESS` and that the TypeScript AUDIT commands of the vergil-tooling
  version `v2.1` resolves to no longer name license-checker. Its result names
  both filed tasks.

### 6.6 Rollout and validation

1. **vergil-tooling:** the license gate (§6.1) and the npm updater (§6.2),
   each with its own docs, as separate tasks.
2. **vergil-actions:** the contract change (§6.3).
3. **Release and deploy.** The human releases vergil-tooling, then
   vergil-actions. **Order matters:** vergil-actions' contract makes a
   missing `licenses.json` fail CI, so it must not reach repos before a
   vergil-tooling that writes it. A deployment task records both releases
   and that the released vergil-tooling writes `licenses.json` for a
   TypeScript repo. Releases are human-gated preconditions, attested and
   never cut by an agent.
4. **vergil-claude-plugin:** the `fixture-run` change (§6.4), and its release.
5. **Fixture runs** on both TypeScript fixtures (§6.4).
6. **File the license-checker removal** (§6.5).
7. **Bookends:** the documentation review
   (vergil-project/vergil-tooling#4707), the follow-on brainstorm
   (vergil-project/.github#959, epic 4) and the retrospective
   (vergil-project/.github#1341).

### 6.7 Docs

Each change updates its own repo's docs in the same PR (the placement law):

- vergil-tooling: the TypeScript standards page (the license gate,
  `.npm-license-exceptions`, the updater), the validation matrix, the
  `vrg-update-deps` page (the npm updater and its held-back notes) and the
  tool-ownership page (license-checker no longer a TypeScript gate tool);
- vergil-actions: the evidence docs (§6.3);
- vergil-claude-plugin: the `fixture-run` skill itself.

The documentation review (vergil-project/vergil-tooling#4707) then sweeps
for what those PRs missed, and files a per-repo doc task for anything outside
vergil-tooling.

## 7. Failure model

- **A disallowed, unparseable or missing license:** the AUDIT check fails,
  naming the package, its version and license value, and both fixes.
- **A bad `.npm-license-exceptions` entry** (malformed, expired, duplicate,
  more than 365 days ahead): the gate fails, naming every bad line.
- **A stale license exception:** a warning naming the entry. The gate still
  passes.
- **A missing, unreadable or pre-v2 lockfile, or an entry with no
  version:** the license gate fails, naming the file and the fix.
- **`npm update` fails:** the npm updater's result fails, never "no change".
- **`package.json` changed by the update:** the updater fails, naming it.
- **`npm outdated` fails during the held-back check** (the registry
  unreachable, no or malformed JSON, an `error` object): the updater fails.
  Its non-zero exit when packages are outdated is not a failure. The
  held-back list is never silently empty.
- **`licenses.json` missing after a passing TypeScript audit leg in CI:** the
  staging step fails, as for every contracted report since #1165.

## 8. Constraints

- **The lockfile is the truth.** The license gate reads only
  `package-lock.json`, and the updater changes only `package-lock.json`.
- **No install scripts.** Every npm command Vergil runs passes
  `--ignore-scripts`.
- **Python and Rust are untouched.** Their gate commands, reports, evidence
  and updaters stay byte-for-byte the same (an anti-goal, §3). Factoring the
  shared exception rules out of `lib/npm_audit.py` changes no behaviour of
  the audit gate.
- **Validation runs through `vrg-validate`.** CI calls `vrg-validate --check
  audit` and holds no license commands of its own.
- **Releases are human actions,** and fixtures consume only released Vergil.
- **Fail loudly** (§7). No fallback hides an error.
- **Placement law.** Each task lives in the repo its PR lands in.

## 9. Related work

- vergil-project/.github#1174: epic 1, TypeScript through the full cycle; the
  audit gate, the tool-ownership guideline and the fixtures.
- vergil-project/.github#1281: epic 2, TypeScript lifecycle PRs and toolchain
  enforcement.
- vergil-project/.github#1165: the Rust counterpart, and the model for the
  updater, the evidence and the rollout.
- vergil-project/.github#384: restarting the language updaters; this epic
  delivers TypeScript's part.
- vergil-project/.github#959: epic 4, the Node runtime architecture; this
  epic's follow-on brainstorm.
- vergil-project/.github#1181: TypeScript 7, deferred.
- vergil-tooling#4682: the audit exceptions this design's license exceptions
  follow.
- vergil-tooling#4478: the TypeScript semgrep ruleset, already delivered.

## 10. Left to the builder

- The license gate module's name and the `licenses.json` field names, kept
  consistent across vergil-tooling and vergil-actions.
- The syntax that delimits a license value in `.npm-license-exceptions`, and
  the marker for a missing `license` field, as long as the match against the
  recorded value is exact.
- Whether the SPDX expression parser is Vergil's own or a vetted library, as
  long as it is strict and covers the grammar in §6.1.
- The npm command a fixture run's change task uses to roll a package back in
  the lockfile without touching `package.json`.
- How the license-checker removal step is filed as an epic task (its kind and
  scaffold), within §6.5's precondition.
