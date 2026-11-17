# TypeScript lifecycle PRs and toolchain enforcement — design

- **Epic:** vergil-project/.github#1281
- **Planning issue:** vergil-project/.github#1208 (review history is in its comments)
- **Status:** in design review

## 1. Intent

Vergil owns the Node runtime of a TypeScript repo, as it owns Python's. A new
Node major arrives, and an old one leaves, through a reviewed lifecycle pull
request opened by the vergil-containers lifecycle job, not by hand, and Node
catches up to 26 that way. A TypeScript repo cannot declare a Node or
package-manager version that disagrees with the one Vergil runs, and every CI
leg checks that it runs the Node major CI resolved for it.

The consumer-facing invariant is Python's and Rust's: **a consuming repo deals
only in the major.** It builds and tests on "Node 24", never on a release. The
release (`24.z.w`) is owned by vergil-containers and moves forward each time
the containers rebuild, with no change in any consumer.

## 2. Why

- **Node is a runtime Vergil will own, like Python.** Epic 4 of the wave
  (vergil-project/.github#959) packages Node as Vergil packages Python (core
  and family Node, deb and rpm packages, Node on the agent VMs). A predictable,
  reliable runtime is the point of that work, and it starts here: a repo must
  not be able to state a Node version of its own.
- **Python is the model, not Rust.** Rust is a compiled toolchain chosen by
  rustup, which acts on `rust-toolchain.toml`; Vergil had to pin rustup inside
  the image (epic vergil-project/.github#1095). Node, like Python, is an
  interpreter runtime that Vergil installs and puts on `PATH`. Python's
  `.python-version` is an error because the runtime belongs to Vergil, whether
  or not a tool inside the image acts on the file. Node's version files follow
  the same rule. Once Node is installed on hosts and VMs (epic 4), work outside
  the container becomes real, and those files would otherwise act there.
- **The window has moved without Node.** Node 26 became LTS on 2026-10-28.
  The lifecycle job reports it (vergil-project/vergil-containers#730), but under
  `automation: "report"` nothing opens the change.
- **TypeScript now runs the full cycle** (epic vergil-project/.github#1174):
  both TypeScript fixtures were created, released and validated twice, so there
  is a live pipeline to enforce against and validate on.

This is epic 2 of the TypeScript-parity wave (epic vergil-project/.github#1174
spec §2). Epic 3 is TypeScript supply chain and the npm updater
(vergil-project/.github#1209); epic 4 is the Node runtime architecture
(vergil-project/.github#959).

## 3. Goals, non-goals, anti-goals

### Goals

1. The `ts-node` family's manifest automation is `pr`, and the window reaches
   26/24/22 through the automation's own add PR.
2. `vrg-validate` reports, for a TypeScript repo, each setting that names a
   Node or package-manager version as an error: a `.nvmrc` or `.node-version`
   file, a `packageManager` field, and a `devEngines` entry for the runtime or
   the package manager.
3. `vrg-validate` warns when the `engines.node` range admits a major below the
   window's oldest, or excludes the window's newest, as it does for Python's
   `requires-python`.
4. `vrg-container-run` never forwards the host's Node and npm settings into the
   container.
5. Every TypeScript CI leg asserts that it runs the Node major CI resolved for
   it, and logs the executable that ran.
6. The docs describe the new behaviour.

### Non-goals

- Node packaging, core and family Node on hosts, and Node on the agent VMs:
  epic 4, vergil-project/.github#959. That includes lining up the separate
  Node install in vergil-vm with the window.
- Lifecycle automation for the core Node (`core.node`, today 22):
  vergil-project/.github#1094.
- Automatic, in-order lifecycle merging: vergil-project/.github#1128.
  Lifecycle PRs stay human-merged.
- Narrowing the fleet-wide `actions/*` allowance, which covers
  `actions/setup-node` as it covers `actions/setup-python` and
  `actions/setup-go`. It is a change for every language; the CI assertion is
  the backstop against a workflow that switches Node.
- Taking the Node window from a source other than endoflife.date (§5).
- Supply chain, audit gating and the npm updater: epic 3,
  vergil-project/.github#1209.
- Package managers other than npm. Vergil runs npm; pnpm and yarn are not
  supported, and this epic adds no support for them.

### Anti-goals (the epic fails if any of these happen)

- A consumer repo has to name, or is exposed to, a Node release below the
  major.
- A repo setting names a Node or package-manager version and `vrg-validate`
  passes without reporting it.
- A TypeScript CI leg runs a Node major other than the one CI resolved, and
  passes.
- An `engines.node` range the check cannot read is treated as "no floor".
- The lifecycle add PR goes red in CI for a reason the automation itself
  caused.

## 4. Current state (facts the design builds on)

### vergil-containers

- `docker/versions/manifest.yml`: the `ts-node` family (`language:
  "typescript"`, `token_prefix: "node-"`, `build_arg: "NODE_MAJOR"`) is
  `automation: "report"` with majors 24 and 22. `core.node` is `"22"`, a
  separate pin for the core Node that base's own tools run on.
- `docker/versions/lifecycle/adapters/node.py`: the window is the majors that
  are LTS on the day of the check and not EOL, from endoflife.date `nodejs`. A
  major is GA on its LTS start date. A major is installable when nodejs.org's
  release index has a release of it with both Linux tarballs. Its
  `extra_changes` adds the follow-up "line up the separate Node.js pin in
  vergil-project/vergil-vm with the new ts-node window" to every add, drop and
  out-of-policy change.
- The lifecycle job currently reports `add` for Node 26 (GA 2026-10-28, EOL
  2029-04-30) as vergil-project/vergil-containers#730. Node 22's EOL is
  2027-04-30, so after the add the window is three majors, 26/24/22, and no
  drop is due until then.
- Lifecycle add and drop PRs write and remove their toolchain pins through
  shared code that dispatches to each family's resolver, node included (epic
  vergil-project/.github#1095, spec §6.1, shipped).
- `docker/ts-node/Dockerfile.template` unpacks the pinned nodejs.org tarball
  at `/opt/node/<major>` and links only `node`, `npm` and `npx` into
  `/usr/local/bin`. The core Node is at `/opt/vergil/node/<core-major>`, beside
  it. `docker/ts-node/checks.sh` checks at build time that `node`, `npm` and
  `npx` on `PATH` are the family's, and that markdownlint stays on the core
  Node. The tarballs for 22 and 24 carry `corepack` in `bin/`, unlinked; from
  Node 25 the tarball no longer ships it.
- The image has no Node version manager (no nvm, fnm, Volta or asdf).
- The ts-node policy page is `docs/site/docs/languages/ts-node.md`.

### vergil-tooling

- `src/vergil_tooling/lib/versions/extensions/__init__.py` registers the
  per-language hooks. `FLOOR_CHECKS` has python, go and rust;
  `TOOLCHAIN_CHECKS` has python and rust; `CONTAINER_ENVS` has python.
  TypeScript has none.
- `lib/versions/floors.py` calls a floor check with the window's oldest and
  newest versions for the language. For TypeScript these are bare majors
  (`"22"`, `"24"`), as `window.json` stores them; the token prefix is separate.
- `extensions/python.py` is the model:
  - `check_toolchain` errors on `.python-version`, with the fix "delete it" and
    the reason that the interpreter belongs to Vergil.
  - `check_floor` parses `requires-python` with `packaging`'s `SpecifierSet`.
    It warns when the floor is below the window's oldest version and when an
    upper bound excludes the newest. An invalid specifier is a `VersionsError`.
  - `container_env` hands uv the resolved version (`UV_PYTHON`) and pins the
    interpreter's source (`UV_PYTHON_PREFERENCE=only-system`). It exists
    because uv chooses among the interpreters it can find.
- A malformed `package.json` is already an error, not "not TypeScript" (epic
  vergil-project/.github#1174, spec §6.7).
- `lib/container.py`: `build_container_args` forwards every host variable
  matching the repo's `[container].env-prefixes` as `-e NAME`, which beats the
  image's `ENV`. `NEVER_FORWARDED` is a set of exact names
  (`VRG_VERSION_WINDOW`, `RUSTUP_TOOLCHAIN`, `RUSTUP_AUTO_INSTALL`); there is
  no prefix exclusion. `env_names` and `extra_env` are not filtered.
- vergil-tooling's runtime dependencies are `rich`, `pyyaml`, `pyelftools`,
  `packaging` and `cryptography`. None parses npm's semver ranges.

### vergil-actions

- `actions/ci/toolchain/assert/assert.sh` asserts `x.y` for python, go and
  rust, and posts a notice for any other language. Its `check_expected`
  requires `x.y`; a Node leg's version is a bare major.
- The Python branch logs the interpreter's path as well as its version.
- The assert step already runs on every leg for every language
  (`ci-quality.yml`, `ci-test.yml`, `ci-audit.yml`), with `expected:
  ${{ matrix.version }}`. Adding a TypeScript branch changes only `assert.sh`,
  its action description and its tests, not a workflow file.

### Fleet

The only TypeScript-primary repos are the fixtures `vergil-fixture-ts-lib`
and `vergil-fixture-ts-app` (`mnemosys-project/melete` is a Python repo that
builds a TypeScript app; its primary language is Python, so these checks do
not run on it). Neither fixture has a `.nvmrc`, `.node-version`,
`packageManager` or `devEngines`. `vergil-fixture-ts-app` declares
`engines.node: ">=24"`, which is above the window's oldest major (allowed) and
admits its newest. So the vergil-tooling release that adds the errors breaks
no consumer. The plan re-checks the fleet before that release.

## 5. Approaches considered

1. **Checks in vergil-tooling, the CI assertion as the backstop, no image
   change (chosen).** Python's shape without a container-environment hook.
   `vrg-validate` reports the settings, `assert.sh` checks what actually ran,
   and the passthrough keeps the host's settings out. Python needs a
   container hook because uv chooses among interpreters and must be told
   which; Node has no chooser, `PATH` decides, and the image already fixes and
   checks `PATH`. If epic 4 brings in something that chooses a Node, that is
   where a hook goes.
2. **Approach 1 plus image hardening** (removing the unlinked `corepack` from
   the 22 and 24 tarballs, build-time checks that version files in a scratch
   project change nothing). Rejected: nothing in the image acts on those files,
   and an explicit command a repo runs (`corepack enable`, `npx node@20`) is
   visible in its source and out of scope, as an explicit `uv python install`
   is for Python. It would change two series that are leaving.
3. **A container-environment hook that overrides npm's behaviour** (for
   example forcing `engine-strict` off so `devEngines` cannot fail an install).
   Rejected: it silently overrides what the repo declared. Reporting the
   setting as an error says it out loud.

**Window source.** Rust's window moved to its own release channel because
endoflife.date lagged a new series by days (vergil-project/vergil-containers#910).
Node's GA rule is the LTS start date, which the Node release schedule
publishes in advance, and endoflife.date listed Node 26 on that date. Node
keeps endoflife.date.

## 6. Design

### 6.1 `ts-node` moves to `pr` (vergil-containers)

- `manifest.yml`: `ts-node.automation: "report"` → `"pr"`. From then on the
  lifecycle job opens an add PR for each window major the manifest lacks and a
  drop PR for each manifest major outside the window. Today that is one add,
  for Node 26, in place of vergil-project/vergil-containers#730, which closes
  as the job stops reporting it.
- The add PR carries Node 26's toolchain pin through the shared pin mechanism
  (§4), with no Node special case, and passes CI as opened. That includes the
  ts-node image's build-time checks and smoke test on Node 26.
- The add PR carries the adapter's vergil-vm follow-up line. Adding Node 26
  does not change what vergil-vm installs; lining vergil-vm up with the window
  is epic 4's work (§3), and the follow-up is recorded there.
- The ts-node policy page's automation text changes in the same PR as the
  switch. Its dated window facts are rewritten once the add has merged, in a
  task of its own, as for Rust.
- `core.node` stays `"22"` (§3).

### 6.2 TypeScript toolchain checks (vergil-tooling)

A new `extensions/typescript.py`, registered in `extensions/__init__.py` as
`TOOLCHAIN_CHECKS["typescript"]` and `FLOOR_CHECKS["typescript"]`. Its module
docstring states the rule as Python's does: the Node runtime belongs to
Vergil.

**`check_toolchain`** returns one error per setting that names a Node or
package-manager version, each with the fix and the reason (Vergil selects the
Node runtime through the image, and runs npm):

- a `.nvmrc` file at the repo root. Fix: delete it.
- a `.node-version` file at the repo root. Fix: delete it.
- a `packageManager` field in the root `package.json`, whatever it names,
  `npm` included. Fix: remove the field.
- a `devEngines` field in the root `package.json` with a `runtime` or a
  `packageManager` entry, in either form npm accepts (an object or an array of
  objects). Fix: remove that entry. `devEngines`' `os`, `cpu` and `libc`
  entries are allowed.

`devEngines` is reported because npm acts on it: npm checks it on install and,
by default, fails the install when the running Node or npm does not match. The
spec's builder confirms that behaviour, and the npm versions that have it,
against npm's documentation for the npm bundled with each window major, and
records the URLs in the PR. If the behaviour differs from this paragraph, the
error stays (the setting still names a version Vergil owns) and the spec is
corrected in the same PR.

A `devEngines` or `engines` that is not an object, or a `devEngines` array
entry that is not an object, is a `VersionsError` naming the field. A missing
`package.json` is not checked here; a malformed one is already an error (§4).

**`check_floor`** reads `engines.node` from the root `package.json`:

- No `engines`, or no `engines.node`: no warning.
- `engines.node` not a string, or a string that is not a valid npm semver
  range: a `VersionsError` naming the value. Never "no floor".
- **Floor:** the major of the lowest version the range admits. When it is
  below the window's oldest major, warn: "engines.node `<range>` admits Node
  `<major>`, below the oldest supported major `<oldest>`. Fix: raise it to
  `>=<oldest>`". A range whose lowest admitted major is above the oldest is a
  deliberate restriction and is allowed, as for Python, Go and Rust.
- **Upper bound:** when the range admits no release of the window's newest
  major, warn: "engines.node `<range>` excludes Node `<newest>`. Fix: drop the
  upper bound, or widen it to admit `<newest>`". For example `^24` excludes
  26.
- Ranges with `||` are read as a whole: `^20 || ^24` has a floor of 20.
- Prerelease versions play no part in either test.
- Only the root `package.json` is read. Workspaces are not checked.

The range grammar is npm's (node-semver's documented range syntax: comparators,
`x`-ranges, `~`, `^`, hyphen ranges and `||`). §10 leaves the parser to the
builder, under one constraint: it reads that grammar faithfully, and anything
it cannot read is an error.

Tests cover each error and the allowed `devEngines` entries; the floor and
upper-bound warnings with `>=`, `^`, `~`, `x`, hyphen and `||` ranges; a floor
above the oldest; no `engines.node`; and invalid values. The window used in the
tests is 26/24/22.

### 6.3 Node settings are never forwarded (vergil-tooling)

The env-prefixes passthrough in `build_container_args` never forwards:

- `NODE_OPTIONS` (exact name), which changes how Node runs, for example by
  preloading code;
- any name starting `npm_config_`, compared without case, since npm reads its
  environment configuration case-insensitively (`NPM_CONFIG_` too);
- any name starting `COREPACK_` or `VOLTA_`.

`NEVER_FORWARDED` keeps its exact names, and a new `NEVER_FORWARDED_PREFIXES`
holds the prefixes; both are checked in the passthrough loop. `env_names` and
`extra_env` stay unfiltered, as today. The docstring and the module comment
name the new entries and why. Tests use `env-prefixes = ["NODE", "npm",
"NPM", "COREPACK", "VOLTA"]` with each excluded variable and a harmless
variable (`NODE_ENV`) set on the host, and check that only the harmless one is
forwarded.

The exclusion applies to every repo, not only TypeScript ones, as the rustup
exclusion does.

### 6.4 CI asserts the leg's Node major (vergil-actions)

`actions/ci/toolchain/assert/assert.sh` gains a `typescript` branch:

- The expected version is the leg's version as CI passes it. A leading `node-`
  is stripped, and what remains must be a bare major (`^[0-9]+$`); anything
  else is an error.
- It runs `node --version` in the repo checkout and takes the major
  (`v26.1.0` → `26`). It also runs `node -p process.execPath` and logs the
  path, as the Python branch logs the interpreter's, so the log shows whether
  the family Node or another one ran.
- A failure to run `node`, or an answer with no major, is an error.
- On a mismatch it prints `CI resolved <expected> but the job runs <actual>`,
  as the other branches do.

The header comment, the action's description, and the notice for unhandled
languages are updated. Tests cover a match, a mismatch, a `node` failure, an
expected value with and without `node-`, a malformed expected value and
`v26`.

### 6.5 Docs

- vergil-tooling site docs: the TypeScript standards page says Vergil selects
  the Node runtime; a repo declares at most an `engines.node` range, which
  should admit every window major; and `.nvmrc`, `.node-version`,
  `packageManager` and `devEngines` runtime or package-manager entries are
  errors. The validation and container pages list the new check and the new
  passthrough exclusions.
- vergil-containers: the ts-node policy page and the lifecycle page describe
  `pr` automation (§6.1).
- vergil-actions: the assert action's page and the actions index cover
  TypeScript.
- The documentation review (vergil-project/vergil-tooling#4638) sweeps these
  and the `docs` repo's summary pages, and files a per-repo doc task where one
  is needed.

### 6.6 Rollout and validation

1. vergil-tooling (§6.2, §6.3) and vergil-actions (§6.4) merge. Before the
   vergil-tooling release, the fleet is re-checked for TypeScript-primary
   repos with a setting §6.2 reports.
2. **Releases** (deployment task): vergil-tooling and vergil-actions releases
   carrying step 1 are published (human-gated) and attested.
3. vergil-containers: §6.1's switch to `pr` merges with the policy page's
   automation text. The lifecycle job opens the Node 26 add PR; it is reviewed
   and merged by the human. The policy page's dated facts are rewritten.
4. **Containers release** (deployment task): a vergil-containers release
   (human-gated) carrying the caught-up window, so consumers read 26/24/22 at
   their `[dependencies].vergil` ref and prod images for Node 26 exist.
5. **Fixture runs**, one on each TypeScript fixture
   (`vergil-fixture-ts-lib`, `vergil-fixture-ts-app`), filed with the
   `fixture-run` skill and blocked by step 4. Each checks:
   - CI passes: the library on every window major (26, 24, 22), the app on the
     newest (26), since an app's CI runs only the newest by design;
   - the assertion's log reports the leg's major and the family Node's path
     (`/opt/node/<major>/bin/node`);
   - on a scratch branch, adding a `.nvmrc` fails `vrg-validate` with the
     toolchain error, and an `engines.node` of `^24` produces the upper-bound
     warning;
   - `vrg-container-run` on the fixture, with `npm_config_registry` set on the
     host and a matching `env-prefixes`, does not see the variable.

   The scratch-branch changes are never merged. Epic
   vergil-project/.github#1218 also has runs filed on both TypeScript fixtures;
   the human sequences the two epics' runs by hand, because the recipe's
   serialization rule deadlocks when two runs are filed before either starts.

## 7. Failure model

- An unreadable `engines.node`, `engines` or `devEngines` is a `VersionsError`
  that fails `vrg-validate`, naming the field.
- A version file or field §6.2 reports fails `vrg-validate`, naming the fix.
- A Node mismatch in CI fails the leg with `CI resolved <major> but the job
  runs <major>`; a failure to run `node` fails it too.
- An upstream failure (endoflife.date, nodejs.org) surfaces as a lifecycle
  failure issue or an `ApplyError`, never as a silent "no change", as today.

## 8. Constraints

- No agent cuts a release; releases of vergil-tooling, vergil-actions and the
  images are human-gated.
- Lifecycle PRs stay human-merged.
- Fail loudly: no fallback that hides an unparsed range, a malformed
  `package.json` field or a failed `node` call.
- The shared resolver has no language branches; every TypeScript rule lives in
  `extensions/typescript.py`. The shared lifecycle code stays
  language-neutral.
- Lifecycle PRs write no file in a consumer repo. (A file Vergil writes into a
  TypeScript repo must be Prettier-clean; nothing in this epic writes one.)

## 9. Related work

- Epic vergil-project/.github#407: the version lifecycle and resolver design
  (§6.4 floors, §6.5 toolchain selection).
- Epic vergil-project/.github#1095: the Rust counterpart, which built the
  generic lifecycle pin handling this epic relies on.
- Epic vergil-project/.github#1174: TypeScript through the full cycle, epic 1 of
  the wave, and its fixtures.
- vergil-project/.github#959 (epic 4), vergil-project/.github#1209 (epic 3),
  vergil-project/.github#1094 (core lifecycle), vergil-project/.github#1128
  (automatic lifecycle merging).
- Epic vergil-project/.github#1218: shares the TypeScript fixtures for its own
  runs.

## 10. Left to the builder

- **The npm range parser.** A maintained Python library that implements npm's
  range grammar, added as a vergil-tooling runtime dependency, or a parser in
  `extensions/typescript.py` tested against node-semver's documented range
  examples. The choice and its reason are recorded in the PR. Either way, an
  input it cannot read is an error.
- The exact wording of the errors and warnings, following the Python messages'
  shape.
- Whether `devEngines`' allowed entries are listed in the error message.
