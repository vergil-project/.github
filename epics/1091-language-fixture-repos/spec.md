# Language fixture repos — design

- **Epic:** vergil-project/.github#1091
- **Planning issue:** vergil-project/.github#1022 (review history is in its comments)
- **Status:** approved

## 1. Intent

Vergil supports several project languages, but it has a real integration test
for only one of them, and only by accident. Python support is exercised every
day because Vergil's own tooling (vergil-tooling) is a Python repo. Every
other language family ships changes to its tooling, containers and CI with no
repo exercising them end to end. Epic vergil-project/.github#649 moved every
language family onto a new base image, but its pre-release check could run
only against Python. The rust, go, cpp and ts-node images shipped without a
consumer.

The multi-language MQ REST Admin repos used to fill that role informally:
Vergil was factored out of their five language implementations. They are now
stale, on hold and unbuildable, so they no longer prove anything.

This epic replaces both the accident and the stale testbed with
purpose-built **fixture repos**. They are small, real, managed repos whose
only job is to be driven through Vergil's normal development cycle after a
major change: a code change, its PR, the merge, and a release. A language's
support is proven by running that cycle on its fixtures, not by waiting for a
real project in that language to exist, and not by relying on the language
Vergil itself happens to be written in.

The fixtures serve a second purpose: each is a working example of a
well-formed Vergil repo for its language and shape. That is a reason to make
each one work properly rather than around a gap, and they are expected to
grow more useful as examples over time.

It is the first epic in the wave that brings Rust support up to par with
Python (planning issue vergil-project/.github#410). Every change in that work
is validated on the Rust fixtures this epic creates. The longer-term goal
behind that wave is a 2027 rewrite of Vergil in Rust, using Vergil to do it.

## 2. Goals

1. Four fixture repos in `vergil-project`:

   | Repo | Language | `repository-type` |
   |---|---|---|
   | `vergil-fixture-python-lib` | python | `library` |
   | `vergil-fixture-python-app` | python | `application` |
   | `vergil-fixture-rust-lib` | rust | `library` |
   | `vergil-fixture-rust-app` | rust | `application` |

2. Each fixture is a managed fleet repo, created with `vrg-github-repo-init`,
   with real code, real dependencies, 100% coverage and a docs site, and with
   every piece of fleet automation enabled.
3. A repeatable **fixture run**: a trio of tasks that takes one fixture
   through the full cycle (an ordinary change task, a release task for the
   human's release, then a validation task that checks the result), filed
   by a documented `fixture-run` recipe.
4. A rule in the epic workflow guidance that an epic touching language
   support, containers, CI or release tooling seeds a fixture run for each
   fixture it affects.
5. Proof: one passing fixture run on each of the four fixtures.

## 3. Non-goals

- **Agents submitting, merging or releasing.** In this epic the human runs the
  gated commands, exactly as on any real repo. Fixture-scoped agent rights are
  planning issue vergil-project/.github#1088.
- **Registry publishing and packaging.** No fixture publishes to crates.io,
  PyPI or the production apt/dnf index. Local artifact repositories to test
  publishing against are planning issue vergil-project/.github#1090.
- **Integration tests.** Today's integration-test feature
  (`[ci].integration-tests`) is a prototype from the MQ REST Admin repos that
  has not yet become a proper feature. The fixtures leave it off.
- **Other languages.** Go, C++ and TypeScript fixtures are out of scope. The
  design must extend to them by adding repos, not by redesigning anything.
- **Scheduled fixture runs.** Runs happen when an epic's validation task
  calls for one.
- **Cross-fixture dependencies.** An application fixture that depends on its
  library fixture, which would exercise the release's consumer-refresh stage,
  needs the library to be published somewhere first (#1090).
- **Rust parity at large.** Rust parity gaps the Rust fixtures expose
  belong to #410's work. This epic fixes only the known gaps §6.4 assigns to
  it, and later gaps that are small or that block a fixture; the rest are
  recorded on #410.

## 4. Anti-goals

- **A fixture that is not really managed.** A fixture that opts out of fleet
  automation, or is configured differently from a real repo of its type, tests
  a Vergil nobody uses. Fixtures take the same defaults as a real repo.
- **A fixture run that mocks a step.** A run that skips the real PR, the real
  CI or the real release, or that checks a simulated outcome, is a unit test
  under another name. Every step in a run happens for real on GitHub.
- **Rot.** A fixture that silently goes red between runs, the way MQ REST
  Admin did, is worse than none, because it looks like coverage. The run's
  first step exposes accumulated breakage instead of hiding it.
- **A public artifact.** No run may leave anything in a public registry.
- **Weakening a gate.** A fixture that fails a gate is not fixed by lowering
  the gate in the fixture, for example coverage below 100% or a suppressed
  audit finding. Doing that would hide exactly the breakage the fixture
  exists to catch.

## 5. Approaches considered

**Who drives a run's gated steps** (submit, merge, release):

- **A. The agent drives; the human runs the gated commands (chosen).** The
  same agent-and-human cooperation as on any real repo. It needs no
  permission change and works now. A run takes a few minutes of the human's
  attention. It replaces the manual check an epic would otherwise have run on
  a real repo, so it costs nothing new.
- **B. A scheduled workflow does everything with the org automation App's
  token.** Fully hands-off, but it works only because an App token inside an
  Actions job resolves to human mode. Using that would bypass the identity
  model instead of extending it. Rejected.
- **C. A fixture-scoped agent grant.** A per-repo allowance in the
  submit/release/promote refusals and an App that can merge. It is the
  natural first step in moving the agent boundary, but it is a
  security-sensitive change and would tie this epic to agent-access work not
  yet designed. Deferred to #1088.

**How fixtures stay current:**

- **A. Full fleet members (chosen).** The fleet automation keeps the
  fixtures current on its own, as on any real repo: config-sync PRs merge
  themselves once CI passes (epic vergil-project/.github#1064), and the
  release runs the dependency update. A run starts from a ready state
  (§6.3), and each run therefore also tests that automation.
- **B. Automation off; each run updates explicitly.** Less noise, but the
  scheduled automation, half of what the Rust-parity work is about, would
  never be tested. Rejected.

**Fixture shape:**

- **One fixture per language.** One app-shaped repo would cover only the
  newest-version build. It would miss the multi-version matrix, which is
  critical. Rejected.
- **A library and an application fixture per language (chosen).** The two
  differ now (a library's CI runs the whole version window; an application's
  runs the newest version) and diverge further once publishing is enabled (a
  library publishes to a registry and is never packaged on its own; an
  application is packaged).
- **One repo holding both a library and an application** (for example a Cargo
  workspace). Vergil's release tooling assumes one version per repo, and
  `vrg-version` does not handle a Cargo workspace today. Rejected.

## 6. Design

### 6.1 The fixture repos

All four are public repos in `vergil-project`, created by the human with
`vrg-github-repo-init`. `vrg-github-repo-init` refuses to run under an agent
identity. The plan supplies a complete non-interactive invocation for each
repo, ready to paste. Each invocation sets:

- `--repository-type` as in §2;
- `--language` python or rust;
- the same branching model, versioning scheme and release model as
  vergil-tooling;
- `--publish-release` on and `--publish-docs` on;
- `--no-integration-tests`;
- the window's versions as the tool derives them, never hand-picked.

The App secrets that `vrg-github-repo-init` does not set (`APP_CLIENT_ID`,
`APP_PRIVATE_KEY`, and any other the generated workflows need) are added by
the human as the usual onboarding step. The plan lists them.

Because `vrg-github-repo-init` has no Python or Rust code skeleton, each
fixture is empty of code after init. Its program is added through ordinary
agent PRs, which are themselves the first cycle each fixture goes through.

**Release with nothing published.** `[publish].release = true` with no
`registry-publish` input already produces a real release that uploads
nothing:

- a release PR merged to `main`;
- the `vX.Y.Z` and `develop-vX.Y.Z` tags;
- a GitHub Release with the CI evidence bundle attached;
- the back-merge to `develop` with its version bump;
- the moved `vX.Y` tag.

The fixtures need no new release mechanism. The Rust fixtures' generated
`cd.yml` forwards `CARGO_REGISTRY_TOKEN`; the secret is not set on them, and
publishing is off.

**No packaging.** The fixtures have no `[package]` table. The release's
package-index stage would otherwise publish fixture packages into the
production apt/dnf index.

### 6.2 What each fixture contains

Each fixture holds a small, real program in its language's idiomatic layout:

- **Library fixtures:** a small library with a public API and a test suite.
- **Application fixtures:** a small command-line program with a test suite.

Requirements for all four:

- **2–3 real third-party runtime dependencies** each, so the dependency
  updater, Dependabot, the vulnerability audit and the license check have
  real work to do. The Python fixtures should include `rich`, the library
  Vergil's own CLIs use. Which others to use is left to the builder (§8).
- **100% coverage** under the language's `vrg-validate` gate. Today that is
  branch coverage for Python and line coverage for Rust; the Rust-parity work
  may tighten Rust's gate.
- **A docs site** published to the fixture's own GitHub Pages: a short
  description of what the fixture is for and the API or usage of its
  program.
- **A README** saying, near the top, that the repo is a test fixture with no
  users, what it is for, and a link to this epic's spec.
- **Every fleet automation enabled** that a real repo of its type gets,
  nothing opted out. As of epic vergil-project/.github#1064 that is:
  report-only Dependabot with its nightly audit; `ops.yml`; the automated
  config-sync PRs that merge themselves once CI passes; the epic rollup; and
  the dependency update that `vrg-release` runs. A fixture has no upstream
  refresh, which needs a repo-specific hook it has no use for. The fixture
  repos are created after #1064's rollout has switched ops to its
  config-update mode, so they are born under the automated sync. If #1064
  slips, the fixtures may still be created, with the ops check in §6.3 a
  known gap until it lands.
- **An ad-hoc epic**, so fixture-run changes and anything a run finds can be
  filed as ordinary tasks in the fixture repo.

The program grows by one small increment per fixture run (§6.3), so it should
be designed to take additions without restructuring. A small library of
functions, or a CLI with subcommands, suits this well.

### 6.3 The fixture run

A fixture run is one pass through the full cycle on one fixture. It is a
**trio of tasks**, all linked under the epic that needs the run, in the
epic workflow's usual change → deploy → validate shape. Each task runs under
the skill already built for its kind. The operational-task model (its
kinds, labels and run skills) is unchanged. Only its creation path gains
one thing: a body file can fill the scaffold's sections (§6.4).

- **(a) The change task:** an ordinary task in the fixture repo, implemented
  with `issue-implement` exactly like a task on any real repo. It closes when
  its PR is finalized.
- **(r) The release task:** a `--kind deployment` task, blocked by (a), run
  with `issue-deploy`. Its precondition is the human's `vrg-release` of the
  fixture, attested by the human and never performed by the agent (the
  deployment autonomy boundary). It closes once that release exists and
  contains (a)'s merge.
- **(b) The validation task:** a `--kind validation` task, blocked by (r),
  run with `issue-validate`. It only checks and records, keeping
  `issue-validate`'s rule of no worktree, no commits and no PR.

A new `fixture-run` skill in vergil-claude-plugin is the recipe for a run. It
files the trio, with the bodies below and the `Blocked-by:` links between
them, when an epic's plan calls for a run, and it says which skill runs each
task. It does not run any of them itself.

**(a) The change task:**

1. **Ready to start.** Before implementing anything, the agent confirms the
   fixture is in a state from which the full cycle can begin. If it is not
   (for example, something left open or red from earlier), the agent stops
   and works with the human to get it there, and anything broken outside the
   fixture is filed, with the human's approval, in the repo that owns it.
   This epic deliberately does not define "ready" as a checklist, with one
   exception: **runs on one fixture are serialized.** A run's change task
   starts only when no other run's release or validation task on the same
   fixture is open, so each run gets its own release and its own evidence.
   The other conditions are expected to emerge from the first runs, and the
   `fixture-run` recipe records them as they are learned.
2. **Change.** One small, real change through the normal worktree flow: a new
   function or subcommand with its tests, kept at 100% coverage. It is
   validated with `vrg-container-run -- vrg-validate`, goes through merge
   review, and is handed off with `vrg-pr-workflow report-ready`.
   **Dependency rollback.** The same change moves one third-party dependency
   back to an earlier version **in the lockfile only** (`uv.lock` for
   Python; `Cargo.lock` for Rust, for example with
   `cargo update -p <crate> --precise <version>`). The declared range in
   `pyproject.toml` or `Cargo.toml` stays open, so the release's dependency
   update has a guaranteed upgrade to make. A pinned manifest would make
   leaving it alone the correct behaviour, and the check would prove
   nothing. The earlier version must pass the vulnerability audit, so the
   rollback never turns the PR red for the wrong reason. The rollback is
   skipped on a language whose dependency update is a known gap (below),
   since nothing would restore it and the lockfile would only drift
   backwards. Today that is Rust. The #410 change that delivers a Cargo
   updater turns the rollback on for Rust.
3. **Submit and merge.** The human runs `vrg-submit-pr`, then
   `vrg-finalize-pr` once CI is green, which closes the change task.

**(r) The release task.** The human runs `vrg-release` on the fixture; the
task body gives the exact command. `issue-deploy` confirms the release
exists and contains (a)'s merge, records its version, and closes the task.

**(b) The validation task.**

*Precondition self-check,* a machine probe before anything else:

- the fixture repo exists and the session can clone it;
- `develop` and `main` exist;
- `vergil.toml` has the expected `repository-type` and language;
- the release task (r) is closed, and the release it recorded contains the
  change task's merge;
- the change under validation is in what the fixture will actually consume.
  The fixtures consume only released Vergil: their workflows call
  vergil-actions at the rolling series tag, `setup/vergil` installs the
  tooling version `vergil.toml` pins, and CI runs on the `prod-*` images.
  So the probe checks that the vergil-tooling release, the vergil-actions
  tag or the published image digest the fixture resolves to includes the
  change the validation task names. A run whose probe cannot show that is
  blocked, never run against the previous release.

A run whose preconditions fail reports `blocked: preconditions not met` and
stops (the validation scaffold's rule).

*Checks.* The agent checks, and records with evidence (links, run IDs, tag
names):

- every required CI gate passed on the change PR and on `main`, and the
  library fixtures' matrix ran every version in the window;
- the fixture's latest scheduled ops run is green, and any config-sync PR
  since the previous run merged on its own;
- the release produced the `vX.Y.Z` tag, the GitHub Release with its
  evidence bundle, the back-merge and bump on `develop`, and the moved
  `vX.Y` tag;
- the evidence bundle's contents match the fixture's language: the tools it
  names and the reports it holds come from that language's gates, not just
  any bundle that exists;
- the release's dependency-update stage brought the rolled-back dependency
  up to the newest version its declared range allows, through its usual
  PR, with the versions before and after recorded;
- the docs site was published;
- nothing was published to a package registry: no crates.io, PyPI, apt/dnf
  index or container-registry upload. The release's provenance attestation
  and the fixture's own Pages site are not artifacts in this sense; every
  real repo produces them too.

**Known gaps.** A check that fails because of a gap already recorded on
vergil-project/.github#410 is reported as a known gap: it is recorded with
its evidence on #410, and it does not count against this epic's goal 5.
Today that covers two checks on the Rust fixtures: the dependency-update
check, because `vrg-update-deps` has no Cargo updater (so the Rust run also
skips the rollback), and the evidence-bundle check, because the bundles
name only Python's tools (§6.4). Once #410 closes a gap, the same check is
its proof, and a failure is an ordinary failure again.

*Record.* The results template gets `Outcome: SUCCESS` only if every
precondition passed and every check passed or is a known gap (below), with
each known gap listed in the result. Otherwise it gets `Outcome: FAILURE`,
with each failure filed in the repo that owns the broken behaviour (for
example vergil-tooling or vergil-containers), not in the fixture, under
`issue-validate`'s rule for failure fix tasks.

A run never weakens a fixture to make itself pass (§4). If the right fix for
a failure is in the fixture itself (for example, a dependency the fixture
uses was withdrawn), that fix is a separate, ordinary task in the fixture's
ad-hoc epic.

### 6.4 Bootstrapping: the first cycles

Each fixture's first code PRs (§6.1) run before any fixture run exists. They
are also where the Rust fixtures meet the Rust parity gaps recorded on #410.

**Gaps already known.** The plan carries a task or a recorded decision for
each, not a hope that it is found:

| Gap | Class | In this epic |
|---|---|---|
| No packaged Rust license policy: `cargo deny check` runs with no fleet `deny.toml` or allowlist, unlike Go, C++ and TypeScript | blocks | A task in vergil-tooling delivers a packaged Rust `deny.toml` and allowlist in the fleet's existing style, so a fixture inherits fleet policy instead of setting its own (§7). The task first confirms how cargo-deny behaves with no config. |
| The Rust `.gitignore` fragment is empty, so `target/` is ignored nowhere | blocks | A task in vergil-tooling fills in the fragment. |
| CI evidence bundles name only Python's tools and reports | only weaker | Not fixed here. The bundle-contents check (§6.3) reports it as a known gap on the Rust fixtures, recorded on #410. |
| `vrg-update-deps` has no Cargo updater | only weaker | Not fixed here. A known gap (§6.3), recorded on #410. |
| `vrg-issue-create` refuses a body file for validation and deployment tasks and fills only their intro, so their preconditions, procedure and acceptance stay placeholders (not Rust-specific) | blocks | A task in vergil-tooling lets those kinds take the sections from a body file, so the `fixture-run` recipe can file real release and validation bodies through the sanctioned creation path. |

**Gaps found later.** A gap found while building or running a fixture is
judged when it is found, with the human:

- if it is small, or it blocks a fixture from being created, validated or
  released, it is fixed in this epic as a task in the repo that owns it,
  linked under the epic, and recorded as a dated entry in the plan's
  `## Evolution during execution` (the delivery table stays frozen once
  execution starts);
- otherwise it is recorded clearly on #410 (or, for a gap outside Rust
  parity, filed in intake) and left for that work.

### 6.5 Epic workflow guidance

The epic workflow documentation gains a rule, in two places: the
`epic-create` skill, and the living issue-conventions page
(vergil-tooling `docs/site/docs/standards/github-issues.md`). Convention #40,
which the skill cites, is a closed epic's published spec and is not
rewritten. When an epic changes language support, container
images, shared CI workflows or release tooling, its plan seeds a fixture run
(the §6.3 trio) for each fixture whose language or shape the change affects.
The run's change task is blocked by the deployment or release task that
ships the change to the fixtures (a vergil-tooling release, a vergil-actions
tag, a container image publish), never only by the code task that merges it:
a fixture runs released Vergil, so a run after the merge alone would test
the previous release. A change that affects every language seeds all four. The Rust-parity work (#410) is the first epic to follow the rule.

The rule says *which fixtures* and *when*. It does not replace the epic's
other validation.

### 6.6 Extending to other languages and features

A new language gets fixtures by adding `vergil-fixture-<lang>-lib` and
`vergil-fixture-<lang>-app` repos to the same pattern, with no change to the
skill other than its list of fixtures. Turning on publishing (#1090),
packaging, integration tests or agent rights (#1088) later adds steps to
§6.3's change task (a) or release task (r), and checks to the validation
task's (b) *Checks*. It does not change the repos' purpose or
names.

## 7. Constraints

- **Placement law.** Each task lives in the repo where its closing PR lands.
  Fixture code changes are tasks in the fixture repo, the skill is a task in
  vergil-claude-plugin, and guidance edits are tasks where that guidance
  lives.
- **Agent pushes cannot touch `.github/workflows/**`.** The fixtures'
  workflows come from `vrg-github-repo-init` (human-run) and later
  automated config-sync PRs. A fixture task that must change a workflow file is pushed by the
  human's `vrg-submit-pr`, as on any repo.
- **Agents never create repos.** Only the human runs `vrg-github-repo-init`.
- **No fixture changes the fleet's policy.** A fixture uses the release,
  validation and ruleset policy that `vrg-github-repo-init` and
  `vrg-github-repo-config` give any repo of its type.
- **Public repos.** The fixtures are public like the rest of `vergil-project`,
  and hold nothing that could not be.

## 8. Left to the builder

- The program each fixture holds, its exact API or subcommands, and its 2–3
  dependencies (beyond `rich` for Python). Choose well-maintained, widely
  used, permissively licensed packages that pass the fleet's license policy
  for the language (for Rust, the packaged policy §6.4 delivers). For Rust, the dependencies must not need anything the dev
  image lacks to build.
- The exact `vrg-github-repo-init` flag values, derived from vergil-tooling's
  own `vergil.toml` and the tool's choices.
- The shape of the `fixture-run` recipe skill, how it reads the list of
  fixtures, and how it fills the trio's task bodies.
- The wording and placement of the epic-guidance rule in §6.5.
- How each run's code change is chosen, within §6.3 step 2.
