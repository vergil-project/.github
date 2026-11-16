# TypeScript through the full cycle — design

- **Epic:** vergil-project/.github#1174
- **Planning issue:** vergil-project/.github#1161 (review history is in its comments)
- **Status:** draft

## 1. Intent

TypeScript on Node becomes a language Vergil can carry through its whole
cycle. A TypeScript library and a TypeScript application each go change → PR
→ merge → release → validate, the way the Python and Rust fixtures do, with
no hand-set version pin, with tools sourced by one stated rule, and with
evidence a release can attach.

The epic also writes down a cross-language rule for where a tool lives, in
the repo or in the image. TypeScript is the first language built to it.

## 2. Why

- **A real consumer is coming.** `mnemosys-project/melete`, a Python repo,
  builds a small TypeScript app that generates Guitar Pro 8 files. A later
  epic extracts it into a standalone mnemosys CLI, and that CLI needs
  TypeScript to work end to end first.
- **The TypeScript pipeline has never run in a real repo.** The only
  end-to-end exercise is the ts-node image's own smoke project, which runs
  `tsc` and `vitest` directly rather than through `vrg-validate`. The gap
  inventory on the planning issue lists what is missing.
- **Python and Rust are not the standard to copy.** They are the two
  languages that work today, and they source their tools differently: Python
  declares every tool in the repo, Rust takes every tool from the image, and
  TypeScript mixes the two in a way that breaks (§4). This is the moment to
  decide the rule once, for every language.

This is epic 1 of the TypeScript-parity wave:

1. TypeScript through the full cycle (this epic).
2. Lifecycle and enforcement: `automation: "pr"`, the catch-up to Node 26,
   and toolchain enforcement.
3. Supply chain and updates: enforced license and audit gating, and an npm
   updater in `vrg-update-deps`.
4. The Node runtime architecture (vergil-project/.github#959): core and family
   Node, deb and rpm packages for Node, and Node on the agent VMs.
5. Extracting melete's TypeScript tool into a mnemosys CLI.

## 3. Goals, non-goals, anti-goals

### Goals

1. A TypeScript repo with no `[ci].versions` follows the version window. A
   library tests every window version, and an app builds the newest. A pin is
   required only for a language whose window has more than one family.
2. A cross-language standards page states where a tool lives (§6.2). The
   page holds the trade-off analysis and its sources.
3. TypeScript's tools follow that rule. Repos declare the tools their code
   depends on. The image owns the rest. A repo missing a required tool fails
   `vrg-validate` with a message that names the fix.
4. Every TypeScript repo gets Vergil's strict base tsconfig and ESLint config
   as managed files, kept current by the nightly fleet sync.
5. A TypeScript repo can release. `vrg-version` bumps `package.json` and
   refreshes `package-lock.json`, and `cd-release` accepts TypeScript.
6. The TypeScript gates write evidence files, and the CI report contract
   stages them.
7. A `package.json` Vergil cannot parse is an error, never "not TypeScript".
8. `vergil-fixture-ts-lib` and `vergil-fixture-ts-app` exist, release, and
   pass a fixture run.

### Non-goals

- Plain JavaScript. The wave is TypeScript on Node.
- Publishing to npm. A TypeScript release is tags, a GitHub Release and its
  evidence, as for the Rust fixtures.
- Release automation. It is separate, fleet-wide work.
- TypeScript 7, the native compiler. The fixtures and the shared configs stay
  on TypeScript 5.
- Package managers other than npm.
- Epics 2 to 5 of the wave (§2).
- Moving Python's standalone tools into the image under the new rule. The
  rule implies it, and it gets its own planning item (§6.10).

### Anti-goals (the epic fails if any of these happen)

- A TypeScript gate passes with a tool the repo did not lock, or with a tool
  missing.
- A repo turns red overnight because the image removed a tool the repo still
  calls from `PATH`, with no warning beforehand.
- The fleet sync overwrites a file a repo owns.
- A fixture passes only because a check was skipped without being recorded
  as a known gap.

## 4. Current state (facts the design builds on)

### vergil-tooling

- **Version resolution.** `lib/versions/resolve.py:98-99`: a language is
  "family-keyed" if its window has more than one family *or* any family has a
  token prefix. A family-keyed language with no `[ci].versions` fails with
  "must pin [ci].versions until its lifecycle turn" (`:139-146`). C++ has two
  families (`clang-`, `gcc-`). TypeScript has one, `ts-node`, whose tokens
  carry the prefix `node-`. Python, Go and Rust have one family each, with no
  prefix. Tests: `test_versions_resolve.py:130` (C++) and `:139` (TypeScript),
  `test_repo_init.py:1086` (C++).
- **Unpinned behaviour.** Without a pin a library resolves every window
  version and an app the newest (`resolve.py:190`). The window is read at the
  repo's `[dependencies].vergil` ref (`v2.1`), so it moves with Vergil
  releases. No Python or Rust repo pins `[ci].versions`.
- **TypeScript checks** (`lib/languages.py:844-948`):
  - INSTALL: `npm ci`.
  - LINT: `prettier --check . --config {configs}/typescript/prettier.config.json`,
    then the ESLint config is copied into the repo root as
    `.vergil-eslint.config.mjs`, run, and deleted. The copy exists because the
    config's ESM imports (`@eslint/js`, `typescript-eslint`) resolve from the
    config file's own directory, and ESM ignores `NODE_PATH` (#2771).
  - TYPECHECK: `tsc --noEmit`.
  - TEST: `vitest run --coverage` with the V8 provider at 100% lines. It writes
    no report file.
  - AUDIT: `npm audit --audit-level=high` and
    `license-checker --production --summary`. The second does not gate.
  - `build_cmd`, `publish_cmd`: `None`.
  - `{configs}` is the installed package directory (`languages.py:1011`).
- **Shared configs.** `configs/typescript/` holds `eslint.config.mjs`,
  `prettier.config.json` and `tsconfig.base.json`. No command reads the base
  tsconfig. The standards page tells consumers to
  `"extends": "<vergil>/typescript/tsconfig.base.json"`
  (`docs/site/docs/standards/development/typescript/overview.md:113`), a path
  that resolves nowhere.
- **Versioning.** `lib/version.py:28-56` maps the version file, lockfile and
  lockfile refresh for Python (`pyproject.toml`, `uv lock`) and Rust
  (`Cargo.toml`, `cargo update --workspace`, which updates only the
  workspace's own packages). TypeScript has no entry.
- **Evidence.** `RUST_REPORT_FILES` (`languages.py:112-116`) names
  `coverage.xml`, `junit.xml` and `cargo-deny.json`. The consumer side is a
  hand-kept copy in vergil-actions (below), and making vergil-tooling the one
  source is vergil-tooling#4416 (open).
- **Detection.** `lib/container.py:80-122`: a `package.json` that fails to
  parse returns `False` ("not TypeScript"), with no error.
- **Fleet sync** (epic #1064). Each managed file has one `<name>_drift(root)`
  in `lib/repo_config.py`. The audit (`audit_local_config`, `:326-339`) and
  the fixer (`FIXERS`, `lib/repo_config_fixes.py:84-91`) share it, and the
  nightly `vrg-github-repo-config update` applies the fixers through one
  automation PR. Language scoping already exists: `.gitignore` adds a
  per-language fragment and `dependabot.yml` a per-language ecosystem, both
  keyed on `[project].primary-language`. No managed file is yet a whole file
  copied per language from a master.
- **Repo init.** `vrg-github-repo-init` accepts `--language typescript`. There
  is no TypeScript skeleton, and no lock step (`lang_scaffold.py:157-161`).

### vergil-containers

- `docker/common/typescript-analysis.dockerfile` installs, globally and
  unpinned, `typescript@5 eslint typescript-eslint prettier vitest
  @vitest/coverage-v8 license-checker`.
- The ts-node smoke test (`docker/ts-node/smoke/run-in-container.sh`) requires
  all of them on `PATH` or in `npm ls -g`, then runs `npm ci`,
  `npx tsc --noEmit` and `npx vitest run` against a project that declares
  typescript, vitest and `@vitest/coverage-v8`. `docker/ts-node/checks.sh:143-151`
  runs `--version` on `tsc`, `eslint`, `prettier` and `vitest`.
- Node series: 24 and 22, `automation: "report"`. Node 26 is missing
  (vergil-containers#730), which is epic 2's work.

### vergil-actions

- `cd-release.yml:615` runs Build and publish only for
  `["python","rust","go"]`. With `registry-publish` false (the default), a
  release still builds, tags, publishes the GitHub Release with its evidence,
  back-merges and moves the `vX.Y` tag. Publishing is skipped.
- `actions/ci/evidence/reports/reports.sh:10-34` is the hand-kept report
  table. Go, C++ and TypeScript emit a warning and stage nothing.

### vergil-claude-plugin

- `skills/fixture-run/SKILL.md:31-53` holds the fixture table and the known
  gaps. Adding fixtures is a table row, plus a rollback paragraph per language
  (`:198-250`).

## 5. Approaches considered

### 5.1 Where a tool lives

1. **Every tool declared in the repo** (how Python works today). Each repo's
   lockfile records exactly what it was tested with, and a tool bump carries
   its own fallout. But every repo carries the tool list and its update churn,
   repos drift from each other, and each language needs an updater just to
   move its tools. Rejected as the default.
2. **Every tool owned by the image** (how Rust works today). Repos declare
   nothing, and tools float with the nightly rebuild. But some tools cannot
   work that way: ESLint plugins must be installed locally even with a
   global ESLint, test files import their test framework, and a library's
   compiler builds its shipped artifact. Forcing those into the image takes
   injection workarounds (for npm, a `--no-save` install behind the lockfile's
   back). Rejected.
3. **Follow the ecosystem, and let the image own a tool when the reason
   behind the ecosystem's advice is met as well by the image (chosen).** §6.2
   gives the rule. It gives every language the same test, and the outcomes
   differ only where the language's own design forces it.

### 5.2 How a repo gets Vergil's shared TypeScript configs

1. **Managed files written by the fleet sync (chosen).** They resolve in any
   editor and any build, inside the container or out, and the existing nightly
   sync keeps them current. The repo carries files it does not edit.
2. **An npm package.** This is the ecosystem's way, but Vergil does not
   publish to npm, and starting is an outward-facing step outside this epic.
3. **A fixed path in the image.** A repo's `tsconfig.json` would then only
   work inside the container. Rejected.

### 5.3 How to cut the epic

1. **Fixes and fixtures together (chosen).** The fixtures prove each fix on a
   real repo, and the blockers are known, so they are planned up front.
2. **Fixes first, fixtures in a later epic.** The first epic would have
   nothing to prove itself on. Rejected.
3. **Fixtures first, fixing blockers as they appear** (epic #1091's way). The
   blockers are already known. Leaving them unplanned is how that epic grew
   from 26 tasks to 73. Rejected.

## 6. Design

### 6.1 A pin is required only for a multi-family window (vergil-tooling)

- `_is_family_keyed` becomes "the window has more than one family". A token
  prefix alone no longer forces a pin.
- A TypeScript repo with no `[ci].versions` resolves like Python and Rust. A
  library gets every window version (`node-24`, `node-22` today), and an app
  gets the newest (`node-24`).
- C++ still requires a pin. Its error message drops "until its lifecycle
  turn" and says why: its window has more than one family (compilers), so
  there is no single default.
- Tests: the TypeScript case (`test_versions_resolve.py:139`) flips to
  resolving, for both a library and an app. The C++ tests stay. A new test
  covers one prefixed family resolving with no pin.
- Docs: the version-window pages say a pin is an exception, needed only for a
  multi-family language.

### 6.2 The tool-ownership guideline (vergil-tooling docs)

A new cross-language page under `docs/site/docs/standards/development/`
states the rule and how to apply it to a new language:

- **Default: follow the ecosystem.** A tool lives wherever the language
  community recommends.
- **Exception: the image may own a tool** when the *reason* behind the
  ecosystem's advice is met as well by Vergil's shared image. "Everyone on the
  team runs the same version" is such a reason: everyone runs the same image.
  A technical constraint is not.
- **The test:** a repo declares a tool when its code or tests import it, when
  it builds the shipped artifact, or when a tool the repo declares requires
  it. Every other tool may move to the image.
- **What the image costs, and why it is accepted.** The page includes the
  trade-off table from the planning issue: one fleet-wide version and no
  per-repo update churn, against a repo that can turn red with no commit of
  its own, and an old release that rebuilds with newer tools. Vergil floats
  on purpose, so a break is how the leading edge is discovered. The evidence
  records the tool versions a release ran.
- **The sources**, each with what it actually says:
  - [Prettier](https://prettier.io/docs/install): pin exactly, because even a
    patch release can change formatting. That is about team consistency.
  - [Ruff](https://docs.astral.sh/ruff/versioning/): minor releases may change
    stable rules and formatting. Same reason.
  - [ESLint](https://eslint.org/docs/latest/use/getting-started): plugins and
    shareable configs must be installed locally even with a global ESLint.
    That is a technical constraint.
  - [mypy](https://mypy.readthedocs.io/en/stable/installed_packages.html):
    it must run in, or be pointed at (`--python-executable`), the environment
    holding the packages. The environment is needed, not the declaration.
  - [Vitest](https://vitest.dev/config/globals): its APIs are not global by
    default, so test files import them.
  - [pytest](https://docs.pytest.org/en/stable/explanation/goodpractices.html):
    install pytest into the project's environment.
- **Applied, per language:**

  - **TypeScript.** The repo declares typescript, eslint, typescript-eslint,
    `@eslint/js`, vitest and `@vitest/coverage-v8`. The image owns prettier
    and license-checker.
  - **Python.** The repo declares pytest, pytest-cov and any pytest plugins.
    The image owns ruff, mypy, pip-audit and pip-licenses (migration pending,
    §6.10).
  - **Rust.** The repo declares nothing. rustfmt and clippy ship with the
    toolchain, and cargo-deny, cargo-llvm-cov and cargo-nextest come from the
    image.

  The TypeScript row has eslint on the repo side because typescript-eslint
  declares eslint as a peer dependency. `npm ci` installs a matching eslint in
  the repo anyway, and a global eslint beside it could be a major
  typescript-eslint does not support.
- **Each language's standards page** links the guideline and lists that
  language's split.

### 6.3 TypeScript tools follow the rule (vergil-tooling, vergil-containers)

- **Gates run the repo's tools.** LINT's ESLint step, TYPECHECK and TEST call
  the repo-local binaries, with no implicit download (for example
  `npx --no-install`). Prettier and license-checker stay image tools.
- **A required devDependency that is missing fails.** Before the TypeScript
  gates run, `vrg-validate` checks that `package.json` declares each required
  devDependency. It fails with one message naming every missing package and
  the `npm install --save-dev …` that fixes it. A declared tool that `npm ci`
  did not install fails the same way. Nothing falls back to a global binary.
- **The image drops the repo-side tools.** `typescript-analysis.dockerfile`
  keeps prettier and license-checker and removes the rest. The smoke test and
  `checks.sh` change to match: the smoke project declares the repo-side tools
  and runs them through `npx`, and the image check covers the image tools
  only.
- **Order.** The vergil-tooling release whose gates call the repo's tools
  ships, and the fixtures and the smoke project pass with it, before the
  containers release that removes the globals (§6.9). No repo is left calling
  a tool the image no longer has.
- **Docs.** The TypeScript standards pages list the required devDependencies.
  They drop the claim that every repository runs "the same pinned tool
  versions", and say instead that tool versions float through the lockfile.

### 6.4 Shared TypeScript configs as managed files (vergil-tooling)

- **Two new managed files**, both written whole from master copies in
  vergil-tooling's package data, for repos whose `[project].primary-language`
  is `typescript`:
  - `tsconfig.vergil.json`: the strict base, currently
    `configs/typescript/tsconfig.base.json`. The repo's own `tsconfig.json`
    holds only its paths and `"extends": "./tsconfig.vergil.json"`.
  - `eslint.config.mjs`: the flat config, found by ESLint with no
    `--config`. Its imports resolve from the repo's `node_modules`, which
    hold the declared plugins, so the copy-in, delete-after staging goes away.
- **One definition.** Each file has a `<name>_drift` in `repo_config.py`,
  a `_check_*`, a fixer in `FIXERS`, and an entry in `audit_local_config`,
  following the existing pattern. Drift is "differs from the master copy". The
  fixer rewrites the whole file.
- **The repo owns `tsconfig.json`.** The fleet sync never writes it.
  `vrg-validate` checks that it extends `./tsconfig.vergil.json`, and fails
  with the fix otherwise.
- **Scoping.** A non-TypeScript repo gets neither file. A file of the same
  name in a non-TypeScript repo is left alone. A repo that changes its
  language away from TypeScript is out of scope.
- **Prettier's config** stays packaged and is passed with `--config`,
  because Prettier is an image tool.
- **This is a new kind of managed file** (whole file, per language, from a
  master copy). It is built so that another language can add one with a
  table entry, not a new mechanism.

### 6.5 The TypeScript release path (vergil-tooling, vergil-actions)

- **`vrg-version`** gains TypeScript in each map in `lib/version.py`:
  - version file `package.json`;
  - lockfile `package-lock.json`;
  - lockfile refresh `npm install --package-lock-only --ignore-scripts`, run
    in the dev container like Cargo's. It rewrites the lockfile's own version
    fields and upgrades no dependency. A test proves no dependency moves.
- **The build command** is `npm run build`, and a TypeScript repo must define
  a `build` script. A missing script fails with the fix message.
- **`cd-release`** accepts `typescript` at line 615, and the description at
  line 7 says so. The publish step stays off: `publish_cmd` stays `None`, and
  `registry-publish` stays `false`. A TypeScript release produces what a Rust
  fixture's does: release PR, tags, GitHub Release with evidence, back-merge,
  and the moved `vX.Y` tag.

### 6.6 TypeScript evidence (vergil-tooling, vergil-actions)

- **TEST** adds Vitest's JUnit reporter (`junit.xml`) and the Cobertura
  coverage reporter, written to `coverage.xml`, beside its console output. The
  100% line gate is unchanged. Both files are written even when a test fails,
  as for Python and Rust.
- **AUDIT** writes `npm audit`'s JSON as `npm-audit.json`, from the same run
  that gates.
- **`TYPESCRIPT_REPORT_FILES`** in `languages.py` names the three files.
  `reports.sh`'s table gets the matching TypeScript row and stops warning for
  TypeScript, with the existing "keep the two in step" note. If
  vergil-tooling#4416 lands first, the row comes from there instead.
- **Known gap, shared with Rust:** quality-gate evidence (lint and typecheck
  reports) stays in vergil-tooling#4416's scope.

### 6.7 A malformed `package.json` is an error (vergil-tooling)

- Detection raises an error naming the file and the parse failure when
  `package.json` exists but cannot be read or parsed, or is not a JSON object.
  It no longer returns "not TypeScript".
- A `package.json` that parses but has no `typescript` devDependency and no
  `tsconfig.json` is still "not TypeScript". That is a real answer, not a
  failure.
- Tests cover each case.

### 6.8 The TypeScript fixtures

- **Repos.** `vergil-fixture-ts-lib` (repository type `library`) and
  `vergil-fixture-ts-app` (`application`). The human creates them with
  `vrg-github-repo-init --language typescript`, using the same flags as the
  Rust fixtures (epic #1091 plan, Task 5), after the vergil-tooling release
  that carries §6.1 and §6.4. No npm token is set.
- **Content.** Each follows epic #1091 §6.2: a small real program with at
  least one real runtime dependency, 100% line coverage, a docs site, and no
  `[ci].versions`. The app is a command-line tool, the shape of the planned
  mnemosys CLI. Both declare the required devDependencies, extend
  `./tsconfig.vergil.json`, and define a `build` script.
- **`fixture-run`** (vergil-claude-plugin) gains `ts-lib` and `ts-app` rows,
  a TypeScript rollback paragraph, and the TypeScript known gaps:
  - no npm dependency updater (epic 3), so the lockfile-rollback check proves
    nothing yet, as it was for Rust before its updater;
  - quality-gate evidence (vergil-tooling#4416).
- **Each fixture releases `v0.1.0`** and records `Outcome: SUCCESS` on a
  fixture-run validation.

### 6.9 Rollout and validation

1. **vergil-tooling:** §6.1–§6.7's tooling parts and the guideline page merge.
2. **vergil-actions:** `cd-release` and `reports.sh` merge.
3. **Release and deployment** (operational task, human-gated releases):
   vergil-tooling and vergil-actions releases carrying steps 1 and 2.
4. **Fixtures created** (human), then their programs land by agent PRs, then
   each releases `v0.1.0`.
5. **Fixture runs** on both fixtures, filed with `fixture-run`, `--after` the
   step 3 deployment task.
6. **vergil-containers:** the image drops the repo-side tools, and the smoke
   test and `checks.sh` follow (§6.3). This merges only after step 5 passes.
   It is released and deployed in its own operational task.
7. **A second fixture run** on both fixtures, `--after` step 6's deployment.
   It proves the gates no longer need the image's globals.

### 6.10 Follow-on items filed with this epic's tasks

- **Python tool migration:** a planning item to move ruff, mypy, pip-audit
  and pip-licenses into the image under §6.2, across every Python repo
  (vergil-tooling included).
- **Epic 2 and epic 3 planning items**, as §2 lists them.
- Epic 4 is vergil-project/.github#959, already in planning.

## 7. Failure model

- A missing required devDependency, a `tsconfig.json` that does not extend the
  managed base, a missing `build` script and an unparseable `package.json` all
  fail with a message that names the fix. None falls back.
- A gate never runs a global binary in place of the repo's tool, and never
  downloads one.
- A drifted managed file is reported by the audit and fixed by the nightly
  sync PR. A fixer that cannot apply raises `CannotFix`, as today.
- The lockfile refresh in a version bump fails the bump if it would change any
  dependency.

## 8. Constraints

- No agent cuts a release. Releases of vergil-tooling, vergil-actions,
  vergil-containers and the fixtures are human-gated.
- Fixtures consume only released Vergil, so every fixture run is blocked by
  the release that ships its change.
- Fail loudly (§7).
- The image's tool removal lands only after the gates no longer need it
  (§6.9).

## 9. Related work

- Epic vergil-project/.github#1091: the fixture framework and `fixture-run`.
- Epic vergil-project/.github#1095: the Rust epic this wave mirrors. Its
  generic lifecycle pin rendering already covers Node.
- Epic vergil-project/.github#1064: the fleet sync the managed files ride on.
- vergil-tooling#4416: one source of truth for the report contract.
- vergil-project/.github#959: epic 4. vergil-project/.github#384: the language
  updaters (TypeScript's goes to epic 3).
- vergil-containers#730: Node 26, epic 2.

## 10. Left to the builder

- The guideline page's file name and title.
- The exact repo-local invocation (`npx --no-install`, or
  `node_modules/.bin/…`), as long as nothing is downloaded and a missing tool
  fails per §6.3.
- The exact wording of the fix messages, following the shape of the existing
  resolver messages.
- The Vitest reporter flags, and how the Cobertura file lands at
  `coverage.xml`.
- The fixtures' programs and their runtime dependencies.
- Whether the smoke project keeps its own devDependency list or shares one
  with the fixtures.
