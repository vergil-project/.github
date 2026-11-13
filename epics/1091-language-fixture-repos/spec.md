# Language fixture repos — design

- **Epic:** vergil-project/.github#1091
- **Planning issue:** vergil-project/.github#1022 (review history is in its comments)
- **Status:** draft

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
3. A repeatable **fixture run**: a documented, agent-led procedure that takes
   one fixture through the full cycle and checks the result, run as a
   validation task.
4. A rule in the epic workflow guidance that an epic touching language
   support, containers, CI or release tooling seeds a fixture-run validation
   task for each fixture it affects.
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
- **Fixing what the fixtures find.** Rust parity gaps the Rust fixtures expose
  belong to #410's work. This epic records them; it does not fix them, unless
  a gap prevents a fixture from being created or run at all (see §6.4).

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

- **A. Full fleet members; each run first clears what piled up (chosen).**
  Each run then tests the scheduled automation as well.
- **B. Automation off; each run updates explicitly.** Less noise, but the
  scheduled automation, half of what the Rust-parity work is about, would
  never be tested. Rejected.
- **C. Full members, merged as automation PRs arrive.** Always current, but
  weekly chores for repos nobody uses. Rejected.

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
- **Every fleet automation enabled** as `vrg-github-repo-init` generates it:
  Dependabot, `ops.yml` (config audit and branch sweep), upstream refresh, CI
  sync and the epic rollup. Nothing is opted out.
- **An ad-hoc epic**, so fixture-run changes and anything a run finds can be
  filed as ordinary tasks in the fixture repo.

The program grows by one small increment per fixture run (§6.3), so it should
be designed to take additions without restructuring. A small library of
functions, or a CLI with subcommands, suits this well.

### 6.3 The fixture run

A fixture run is one pass through the full cycle on one fixture. It runs as a
**validation task** (`vrg-issue-create --kind validation`) and follows a new
`fixture-run` skill in vergil-claude-plugin, which `issue-validate` invokes.

**Precondition self-check.** A machine probe before anything else:

- the fixture repo exists and the session can clone it;
- `develop` and `main` exist;
- `vergil.toml` has the expected `repository-type` and language.

A run whose preconditions fail reports `blocked: preconditions not met` and
stops (the validation scaffold's rule).

**Steps:**

1. **Catch up.** List the fixture's open PRs. Automation PRs
   (Dependabot, upstream refresh, CI sync, lifecycle or update-deps PRs) are
   reviewed by the agent, which reports their CI state, and are merged by the
   human. A red automation PR is a finding (step 6), not something the run
   quietly works around. Any other open PR stops the run for the human to
   decide.
2. **Change.** The agent files a task in the fixture's ad-hoc epic and makes
   one small, real change through the normal worktree flow: a new function or
   subcommand with its tests, kept at 100% coverage. It validates with
   `vrg-container-run -- vrg-validate` and hands off with
   `vrg-pr-workflow report-ready`.
3. **Submit and merge.** The human runs `vrg-submit-pr`, then
   `vrg-finalize-pr` once CI is green. The skill prints the exact commands.
4. **Release.** The human runs `vrg-release` on the fixture. The skill prints
   the exact command.
5. **Check the outcome.** The agent checks, and records with evidence (links,
   run IDs, tag names):
   - every required CI gate passed on the PR and on `main`, and the library
     fixtures' matrix ran every version in the window;
   - the release produced the `vX.Y.Z` tag, the GitHub Release with its
     evidence bundle, the back-merge and bump on `develop`, and the moved
     `vX.Y` tag;
   - the docs site was published;
   - no artifact was published anywhere outside the fixture repo.
6. **Record.** The validation task's results template gets `Outcome: SUCCESS`
   only if every step and every check passed. Otherwise it gets
   `Outcome: FAILURE`, with each failure filed as an issue in the repo that
   owns the broken behaviour (for example vergil-tooling or
   vergil-containers), not in the fixture.

A run never weakens a fixture to make itself pass (§4). If the right fix for
a failure is in the fixture itself (for example, a dependency the fixture
uses was withdrawn), that fix is a separate, ordinary task in the fixture's
ad-hoc epic.

### 6.4 Bootstrapping: the first cycles

Each fixture's first code PRs (§6.1) run before any fixture run exists. They
are also where the Rust fixtures are most likely to meet the Rust parity gaps
recorded on #410, for example `target/` not being in any `.gitignore`.

- A gap that only makes Rust weaker than Python (for example line-only
  coverage) is recorded on #410 and left for that work.
- A gap that **prevents** a fixture from being created, validated or released
  at all is fixed in this epic, as a task in the repo that owns it, because
  the fixtures cannot exist without it. The plan cannot list these ahead of
  time. Each is filed as it is found, linked under the epic, and the plan's
  delivery table is updated.

### 6.5 Epic workflow guidance

The epic workflow documentation (the `epic-create` skill and the convention
it cites) gains a rule. When an epic changes language support, container
images, shared CI workflows or release tooling, its plan seeds a fixture-run
validation task for each fixture whose language or shape the change affects,
blocked by the work it validates. A change that affects every language seeds
all four. The Rust-parity work (#410) is the first epic to follow the rule.

The rule says *which fixtures* and *when*. It does not replace the epic's
other validation.

### 6.6 Extending to other languages and features

A new language gets fixtures by adding `vergil-fixture-<lang>-lib` and
`vergil-fixture-<lang>-app` repos to the same pattern, with no change to the
skill other than its list of fixtures. Turning on publishing (#1090),
packaging, integration tests or agent rights (#1088) later adds steps to the
run and checks to §6.3 step 5. It does not change the repos' purpose or
names.

## 7. Constraints

- **Placement law.** Each task lives in the repo where its closing PR lands.
  Fixture code changes are tasks in the fixture repo, the skill is a task in
  vergil-claude-plugin, and guidance edits are tasks where that guidance
  lives.
- **Agent pushes cannot touch `.github/workflows/**`.** The fixtures'
  workflows come from `vrg-github-repo-init` (human-run) and later CI-sync
  PRs. A fixture task that must change a workflow file is pushed by the
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
  used, permissively licensed packages that pass the fleet's license
  allowlist. For Rust, the dependencies must not need anything the dev
  image lacks to build.
- The exact `vrg-github-repo-init` flag values, derived from vergil-tooling's
  own `vergil.toml` and the tool's choices.
- The shape of the `fixture-run` skill and how it reads the list of fixtures.
- The wording and placement of the epic-guidance rule in §6.5.
- How each run's code change is chosen, within §6.3 step 2.
