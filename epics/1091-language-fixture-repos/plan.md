# Language fixture repos — implementation plan

- **Epic:** vergil-project/.github#1091
- **Planning issue:** vergil-project/.github#1022 (review history is in its comments)
- **Status:** draft

> **For agentic workers:** drive this plan with `vergil:epic-implement`. Each
> numbered task is one issue and one PR (or one operational task), run with
> the skill its kind names. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Four static, managed fixture repos (`vergil-fixture-python-lib`,
`vergil-fixture-python-app`, `vergil-fixture-rust-lib`,
`vergil-fixture-rust-app`), a `fixture-run` recipe that drives one through
change → release → validate, and one passing run on each.

**Architecture:** Two small vergil-tooling fixes make a Rust fixture possible
(a packaged Rust license policy and the Rust `.gitignore` fragment), and a
tooling release ships them. The human creates the four repos with
`vrg-github-repo-init`, and agents give each its program through ordinary PRs.
A vergil-claude-plugin skill, `fixture-run`, files a run's three tasks, and
`epic-create` gains the rule for when to seed one. The proof is one run per
fixture, filed with the released recipe.

**Tech Stack:** vergil-tooling (Python), vergil-claude-plugin (skills,
Markdown), the fixture repos (Python with uv; Rust with Cargo), cargo-deny.

**Spec:** [`spec.md`](spec.md) (approved).

## Global Constraints

- Repo names: exactly `vergil-fixture-python-lib`, `vergil-fixture-python-app`, `vergil-fixture-rust-lib`, `vergil-fixture-rust-app`.
- `repository-type`: `library` for `-lib`, `application` for `-app`.
- All four: `versioning-scheme = "semver"`, `branching-model = "library-release"`, `release-model = "tagged-release"` (vergil-tooling's own values); `[publish].release = true`, `[publish].docs = true`; `[ci].integration-tests = false`; no `[package]` table; no registry publishing.
- Coverage is 100% under the language's `vrg-validate` gate; no gate is lowered or suppressed in a fixture (spec §4).
- Each fixture has 2–3 real third-party runtime dependencies; the Python fixtures include `rich`.
- Nothing a run does reaches a package registry (crates.io, PyPI, the apt/dnf index, a container registry). Release attestations and the fixture's own Pages site are allowed.
- Agents never create repos, submit, merge or release; the human runs `vrg-github-repo-init`, `vrg-submit-pr`, `vrg-finalize-pr` and `vrg-release`.
- A fixture consumes only released Vergil, so every task that validates on a fixture is blocked by the release that ships what it tests.
- Each task lives in the repo where its closing PR lands (the placement law).
- Validation is `vrg-container-run -- vrg-validate`, nothing else.

## Review Focus

1. **A consumer repo with its own `deny.toml`.** The packaged policy is passed with `--config`, so a repo's own file is ignored. Expect the audit to use the fleet policy, not to fail or to merge the two. This is cargo-deny's own behaviour, so it is evidenced in Task 3 Step 1 from cargo-deny's `--config` documentation, not by a unit test.
2. **A crate whose license is an SPDX expression** (`MIT OR Apache-2.0`, `Apache-2.0 WITH LLVM-exception`, `Unicode-3.0` for `unicode-ident`). These appear in almost every Rust dependency graph. Expect them to pass. vergil-tooling's own gate cannot run cargo-deny, so this is evidenced in Task 3 Step 1 from cargo-deny's documentation, and proven live by the first Rust fixture audit in Task 6.
3. **An existing `.gitignore` on a Rust repo when the fragment changes.** `vrg-gitignore-sync` must add `target/` to the managed block without touching the repo's own lines. Pinned in Task 2's tests.
4. **The human's release lands before a proof run's preconditions can see it** (the tag exists but the release workflow is still running). Expect `issue-validate` to report blocked, not FAILURE. Written into Task 7's validation-task template.
5. **A first Rust fixture CI run hitting a parity gap not in spec §6.4's table.** Expect the gap to be judged with the human (fix it here if small or blocking, otherwise record it on #410), with any new task recorded in `## Evolution during execution`. Written into Task 6.

---

## Delivery table

| Task | Repo | Kind | What | Blocked by |
|---|---|---|---|---|
| 1 | .github | task (#1092) | Spec and plan (this PR) | — |
| 2 | vergil-tooling | task | Rust `.gitignore` fragment | — |
| 3 | vergil-tooling | task | Packaged Rust license policy (`deny.toml`) | — |
| 3b | vergil-tooling | task | Operational task bodies from a file (`vrg-issue-create`) | — |
| 4 | vergil-tooling | deployment | Release vergil-tooling with Tasks 2, 3 and 3b | 2, 3, 3b |
| 5 | .github | deployment | Human creates the four fixture repos | 4 |
| 6a–6d | each fixture repo | task | The fixture's program, tests, docs and README | 5 |
| 7 | vergil-claude-plugin | task | `fixture-run` recipe skill and the `epic-create` seeding rule | 3b |
| 8 | vergil-claude-plugin | deployment | Release the plugin and update the agent boxes | 7 |
| 9a–9d | each fixture repo | run trio | One fixture run per fixture, filed with `fixture-run` | 6x, 8 |
| — | vergil-tooling | bookend (#4317) | Documentation review, including the seeding rule in `standards/github-issues.md` | 9a–9d |
| — | .github | bookend (#1093) | Retrospective | all |

**Filing.** At epic-create's filing step, Tasks 2, 3, 3b, 4, 5, 7 and 8 are
filed. Tasks 6a–6d live in repos that do not exist yet: Task 5 files them
once it has created those repos. Tasks 9a–9d are filed by `fixture-run` once
Task 8 is closed. `Tasks filed: yes` covers what can be filed at planning
time.

A blocking gap found during Task 6 or Task 9 becomes a new lettered task in
the repo that owns it (for example `6e` in vergil-tooling), recorded with
its blockers as a dated entry in `## Evolution during execution` (spec
§6.4). The table itself is frozen once the first task ships.

---

### Task 1: Spec and plan

The documentation task, vergil-project/.github#1092: this PR publishes
`epics/1091-language-fixture-repos/spec.md` and `plan.md`.

---

### Task 2: Rust `.gitignore` fragment (vergil-tooling)

**Files:**

- Modify: `src/vergil_tooling/data/gitignore/rust` (empty today)
- Modify: `src/vergil_tooling/lib/gitignore.py:143` (the docstring naming rust as known-but-empty)
- Test: `tests/vergil_tooling/test_gitignore.py:242` and `:313-314`, and the frozen `_LEGACY_GITIGNORE_PATTERNS` set at `:41` with its comment block. The guards `test_split_is_lossless_against_baseline` (~`:111`) and `test_managed_vocabulary_equals_baseline_set` (~`:284`) assert that base plus fragments equals that set.

**Interfaces:**

- Produces: `gitignore.load_fragment("rust") == ["target/"]`; the managed block for a rust repo includes `target/`.

- [ ] **Step 1: Change the tests.** Add `"target/"` to `_LEGACY_GITIGNORE_PATTERNS`, with a comment-block entry naming epic vergil-project/.github#1091 as the source of the deliberate addition. `test_gitignore.py:242` becomes `assert gitignore.load_fragment("rust") == ["target/"]`. The render test at `:313` asserts `"target/" in block` for `render_block("rust")`. Pick another known language with an empty fragment for the base-only case, or drop that case if none is left. Add `test_sync_rust_adds_target_keeps_repo_lines`: a repo `.gitignore` with a user line `scratch/` outside the managed block, synced for rust, keeps `scratch/` and gains `target/` inside the block (Review Focus 3).
- [ ] **Step 2: Run `vrg-container-run -- vrg-validate`.** Expected: the changed tests fail.
- [ ] **Step 3: Write `target/` (one line) to `data/gitignore/rust`,** and update the `gitignore.py:143` docstring so it no longer names rust as empty.
- [ ] **Step 4: Run `vrg-container-run -- vrg-validate`.** Expected: pass.
- [ ] **Step 5: Refactor.** Tidy the frozen-set comment entry and the `gitignore.py:143` docstring so they read as one deliberate change, then re-run `vrg-container-run -- vrg-validate`. Expected: pass.
- [ ] **Step 6: Commit** with `vrg-commit --type fix --scope gitignore --message "ignore target/ in rust repos"`, then hand off with report-ready.

---

### Task 3: Packaged Rust license policy (vergil-tooling)

**Files:**

- Create: `src/vergil_tooling/configs/rust/__init__.py` (empty, as in `configs/cpp/`)
- Create: `src/vergil_tooling/configs/rust/deny.toml`
- Modify: `src/vergil_tooling/lib/languages.py:426` (the rust `AUDIT` entry), and the allowlist block at `:111-191`
- Modify: `pyproject.toml` `[tool.setuptools.package-data]`: add `"configs/rust/*.toml"`
- Test: `tests/vergil_tooling/test_languages.py:384` (`test_rust_audit_commands`)

**Interfaces:**

- Consumes: the `{configs}` placeholder that `languages.py:881` already resolves to the packaged `vergil_tooling.configs` directory.
- Produces: rust `AUDIT` = `[["cargo", "deny", "--config", "{configs}/rust/deny.toml", "check"]]`, and `_RUST_LICENSES_ALLOWLIST`, the reviewed list beside the other languages' lists.

- [ ] **Step 1: Confirm cargo-deny's behaviour with no config** from its documentation, for the cargo-deny version in the current `prod-rust` image (`cargo deny --version` in the container; the image installs it unpinned). Record in the PR notes that version, whether a missing `deny.toml` rejects every license, whether `--config` makes cargo-deny ignore a repo's own `deny.toml` (Review Focus 1), and how `licenses.allow` evaluates `OR` and `WITH` expressions (Review Focus 2), with a source for each. Either way this task proceeds: the spec puts the policy in the fleet, not in a fixture.
- [ ] **Step 2: Write the failing tests.**
  - `test_rust_audit_commands` asserts the argv above.
  - `test_rust_deny_toml_allowlist_matches_constant` parses the packaged `deny.toml` with `tomllib` and asserts `licenses.allow == _RUST_LICENSES_ALLOWLIST`.
- [ ] **Step 3: Run `vrg-container-run -- vrg-validate`.** Expected: the new tests fail.
- [ ] **Step 4: Add `_RUST_LICENSES_ALLOWLIST`** in `languages.py` beside the Go list, as a `list[str]` of SPDX IDs: `Apache-2.0`, `Apache-2.0 WITH LLVM-exception`, `BSD-2-Clause`, `BSD-3-Clause`, `BSL-1.0`, `ISC`, `MIT`, `MPL-2.0`, `Unicode-3.0`, `Zlib`. Give it a comment naming this epic. Write `configs/rust/deny.toml` (cargo-deny's version 2 schema):
  - `[licenses]`: `allow` equal to that list, `confidence-threshold = 0.93`;
  - `[advisories]`: cargo-deny's v2 defaults, which deny vulnerabilities and unsound advisories, with `unmaintained = "workspace"`, so only the repo's direct dependencies are flagged as unmaintained. That is close to Python's vulnerability-focused gate. Step 1 confirms the option names for the image's version;
  - `[bans]`: `multiple-versions = "warn"`;
  - `[sources]`: `unknown-registry = "deny"`, `unknown-git = "deny"`.

  Point the rust `AUDIT` entry at it, and add the package-data glob. SPDX expressions such as `MIT OR Apache-2.0` pass because both terms are allowed (Review Focus 2).
- [ ] **Step 5: Run `vrg-container-run -- vrg-validate`.** Expected: pass.
- [ ] **Step 6: Refactor.** Bring `_RUST_LICENSES_ALLOWLIST` into line with the existing allowlist block's comments and ordering. It stays a `list[str]`, unlike the other lists' joined strings, because it feeds a TOML array, and a comment says so. Re-run `vrg-container-run -- vrg-validate`. Expected: pass.
- [ ] **Step 7: Commit** with `vrg-commit --type feat --scope languages --message "packaged cargo-deny policy for rust"`, then hand off with report-ready.

---

### Task 3b: Operational task bodies from a file (vergil-tooling)

Today `vrg-issue-create` refuses `--body-file` with `--kind validation` or
`--kind deployment` (`src/vergil_tooling/bin/vrg_issue_create.py:96-98`). It
fills only the scaffold's `{intro}`, so the Preconditions, Procedure and
Acceptance sections stay placeholders (`data/validation_task_body.md:16`,
`data/deployment_task_body.md:17`). The `fixture-run` recipe needs real
content in them, through the sanctioned creation path.

This extends the operational tasks' *creation path* (what a body file may
fill). It does not extend the operational-task *model*: the kinds, labels and
run skills are unchanged, as spec §6.3 requires. It is a small gap that blocks
the recipe, so under spec §6.4 it is fixed here.

**Files:**

- Modify: `src/vergil_tooling/bin/vrg_issue_create.py` (the `--body-file` refusal at `:96-98`, and `_render_scaffold_body` at `:237`)
- Modify: `src/vergil_tooling/data/validation_task_body.md`, `src/vergil_tooling/data/deployment_task_body.md` (named slots in place of the placeholder lines)
- Test: `tests/vergil_tooling/test_vrg_issue_create.py`

**Interfaces:**

- Produces: `vrg-issue-create --kind validation|deployment --body-file <file>`, where `<file>` is Markdown with the level-2 sections `## Intro`, `## Preconditions`, `## Procedure` and `## Acceptance`, all required, plus `## Human-run steps`, optional and for `--kind deployment` only. Each section fills the matching scaffold slot:
  - `## Procedure` fills "Commands to run" for validation and "Deploy steps (agent-safe; idempotent)" for deployment;
  - `## Human-run steps` fills the deployment scaffold's "Human-run steps (need a human identity)" section, which is removed when the file has none. The tool keeps the frame (the self-check rule, the results template and the `Blocked-by:` lines). Without `--body-file`, today's behaviour is unchanged.

- [ ] **Step 1: Write the failing tests.**
  - `test_operational_body_file_fills_slots`: for each kind, the rendered body contains each section's text, and none of the placeholders Task 3b replaces remains: preconditions, commands or deploy steps, acceptance, and the human-run step. The frame's run-time placeholders (`<which>` in the blocked line, and `Evidence:` in the results template) stay, since the run skill fills them when it records a result. For deployment, check both with and without `## Human-run steps`; without it, the scaffold's human-run section is gone. A `## Human-run steps` section on a validation kind is refused.
  - `test_operational_body_file_keeps_frame`: the results template and the `Blocked-by:` lines are present.
  - `test_operational_body_file_rejects_missing_or_extra_section`: a file missing `## Preconditions`, or carrying an unknown `## Other`, exits non-zero with a message naming the section.
  - `test_operational_without_body_file_unchanged`: today's scaffold output, unchanged.
  - `test_retrospective_body_file_still_refused`: `--kind retrospective --body-file` still exits non-zero.
- [ ] **Step 2: Run `vrg-container-run -- vrg-validate`.** Expected: the new tests fail.
- [ ] **Step 3: Implement it.**
  - For `validation` and `deployment` only, replace the refusal with parsing the sections. The refusal stays for `retrospective`.
  - Give both templates `{preconditions}`, `{procedure}` and `{acceptance}` slots, plus `{human_steps}` in the deployment template, defaulting to today's placeholder text.
  - Make `_render_scaffold_body(*, kind: str, intro: str, deps: list[epics.IssueRef], sections: dict[str, str] | None = None) -> str` fill them.
  - Refuse the file loudly on a missing or unknown section.
- [ ] **Step 4: Run `vrg-container-run -- vrg-validate`.** Expected: pass.
- [ ] **Step 5: Refactor.** Define the section-name-to-slot mapping once and use it for both parsing and rendering. Re-run `vrg-container-run -- vrg-validate`. Expected: pass.
- [ ] **Step 6: Commit** with `vrg-commit --type feat --scope issue-create --message "operational task bodies from a sectioned file"`, then hand off with report-ready.

---

### Task 4: Release vergil-tooling (deployment)

**Precondition (human-attested):** the human has run `vrg-release` on
vergil-tooling, and the release contains Tasks 2, 3 and 3b.

`issue-deploy` confirms the release tag exists, that all three PRs are in it, and
that the rolling `v2.1` series tag points at this release or at a later one
that contains it, then records the version.
The agent never cuts the release.

---

### Task 5: Create the four fixture repos (deployment)

**Preconditions:**

- Task 4 is closed.
- Epic vergil-project/.github#1064's rollout has switched ops to its config-update mode (spec §6.2). If it has not, the human decides whether to go ahead, with the nightly ops check a known gap until it lands.

**Human steps.** Run from the parent directory the repos should be cloned
into, one invocation each:

```bash
vrg-github-repo-init vergil-project/vergil-fixture-python-lib --visibility public \
  --description "Vergil test fixture: a Python library. No users; integration test and example repo." \
  --repository-type library --language python \
  --branching-model library-release --versioning-scheme semver --release-model tagged-release \
  --publish-release --publish-docs --no-integration-tests --license MIT --non-interactive

vrg-github-repo-init vergil-project/vergil-fixture-python-app --visibility public \
  --description "Vergil test fixture: a Python application. No users; integration test and example repo." \
  --repository-type application --language python \
  --branching-model library-release --versioning-scheme semver --release-model tagged-release \
  --publish-release --publish-docs --no-integration-tests --license MIT --non-interactive

vrg-github-repo-init vergil-project/vergil-fixture-rust-lib --visibility public \
  --description "Vergil test fixture: a Rust library. No users; integration test and example repo." \
  --repository-type library --language rust \
  --branching-model library-release --versioning-scheme semver --release-model tagged-release \
  --publish-release --publish-docs --no-integration-tests --license MIT --non-interactive

vrg-github-repo-init vergil-project/vergil-fixture-rust-app --visibility public \
  --description "Vergil test fixture: a Rust application. No users; integration test and example repo." \
  --repository-type application --language rust \
  --branching-model library-release --versioning-scheme semver --release-model tagged-release \
  --publish-release --publish-docs --no-integration-tests --license MIT --non-interactive
```

Then, for each repo:

- add the `APP_CLIENT_ID` and `APP_PRIVATE_KEY` secrets, plus any other secret the generated workflows name, as for any onboarded repo (vergil-actions `docs/site/docs/configuration.md` "Secrets"). `CARGO_REGISTRY_TOKEN` is deliberately **not** set;
- create the repo's ad-hoc epic with `vrg-adhoc-epic`.

**Agent check** (`issue-deploy`), per repo:

- the repo exists and is public, with `develop` as default and `main` present;
- `vergil.toml` matches the Global Constraints;
- repo settings and rulesets: the verdict is the human's
  `vrg-github-repo-init` for that repo exiting 0, which applies them. If it
  did not, the human runs `vrg-github-repo-config apply` and that exit 0 is
  the verdict. The agent may run `vrg-github-repo-config audit` for
  information only: under the agent identity it exits 2 by design, because
  the admin-scoped settings cannot be read, and that exit code is never the
  verdict;
- `ops.yml`, `ci.yml`, `cd.yml` and `epic-rollup.yml` exist, and `.github/dependabot.yml` exists and is report-only;
- the ad-hoc epic exists.

It records each repo's first `ops.yml` run, which may be red until Task 6
lands code.

**Then it files Tasks 6a–6d**, one in each new repo, with
`vrg-issue-create --epic vergil-project/.github#1091 --repo vergil-project/<fixture> --blocked-by <this task's ref>`,
using Task 6's text as the body, and records their refs in this task's
result. This plan is the approval for filing them.

---

### Task 6a–6d: Each fixture's program (one task per fixture repo)

One task in each fixture repo, filed in that repo and linked under epic vergil-project/.github#1091,
implemented with `issue-implement`. Each brings the fixture to a
green first cycle: `vrg-container-run -- vrg-validate` passes locally, and
every required CI gate passes on the PR.

**Each fixture gets:**

- **A program.**
  - `-lib`: a small library with a public API of 3–4 functions.
  - `-app`: a small CLI with 2–3 subcommands.

  Either way, the shape takes one more function or subcommand per run without restructuring (spec §6.2).
- **2–3 real runtime dependencies,** chosen by the builder within spec §8.
  - Python: `rich` plus one or two.
  - Rust: crates that build in the dev image with no extra system packages and pass Task 3's policy, for example `clap` (app only), `serde` or `anyhow`.
- **Tests at 100% coverage** under the language gate.
- **A docs site:** what the fixture is for, its API or usage, and a link to the spec.
- **A README** that opens by saying this is a Vergil test fixture with no users, what it exercises, and a link to `epics/1091-language-fixture-repos/spec.md`.

**Steps:**

- [ ] **Step 1:** Write the tests for the public API or CLI, with the expected values (for example `greet("Ada") == "Hello, Ada"`).
- [ ] **Step 2:** Run `vrg-container-run -- vrg-validate`. Expected: the tests fail.
- [ ] **Step 3:** Implement the program, the docs pages and the README.
- [ ] **Step 4:** Run `vrg-container-run -- vrg-validate`. Expected: pass, with 100% coverage.
- [ ] **Step 5:** Refactor: check that the layout takes one more function or subcommand, with its tests, without restructuring (spec §6.2), and adjust it if not. Re-run `vrg-container-run -- vrg-validate`. Expected: pass.
- [ ] **Step 6:** Commit with `vrg-commit --type feat --scope fixture --message "<the fixture's program>"`, and hand off with report-ready.

**Gaps found here** (Review Focus 5): a gap not in spec §6.4's table that
stops `vrg-validate` or CI is stopped on and judged with the human. If it is
small or blocking, it becomes a new task in the repo that owns it, blocking
this one, and is recorded as a dated entry in `## Evolution during
execution`. Otherwise it is recorded on #410
(or filed in intake if it is outside Rust parity). A fixture never works
around a gap by lowering a gate.

---

### Task 7: The `fixture-run` recipe and the seeding rule (vergil-claude-plugin)

**Files:**

- Create: `skills/fixture-run/SKILL.md`
- Modify: `skills/epic-create/SKILL.md` (the "Operational tasks" section)
- Modify: `docs/site/docs/skills/index.md` and `README.md` (the skills table): list the new skill

**Interfaces:**

- Consumes: `vrg-issue-create --epic <ref> --repo <ref> [--kind deployment|validation] --title … --body-file … --blocked-by <ref>`, where an operational kind's body file uses Task 3b's four sections; `issue-implement`, `issue-deploy`, `issue-validate`.
- Produces: the skill `vergil:fixture-run`, invoked as `fixture-run <epic-ref> <fixture>[,<fixture>…] --validates <what> --after <owner/repo#N> [--after …]`.
  - `<what>` names the released change under validation, for example `vergil-tooling v2.1.N`.
  - `--after` is required and repeatable. Each ref is the release or deployment task that ships that change, and is stamped as the change task's `--blocked-by`.
  - The recipe refuses to file a run without `--after`.
  - It files one trio per fixture and prints the refs.

**`SKILL.md` must state:**

- **The four fixtures,** and their language and shape, as the skill's one list (spec §6.6).
- **The trio** (spec §6.3) with exact body templates:
  - **(a) Change task:**
    - "Ready to start", with its one defined rule: no other run's release or validation task on this fixture is open. Anything broken outside the fixture is filed, with the human's approval, in the repo that owns it;
    - the change, kept at 100% coverage;
    - the dependency rollback in the lockfile only, to an audit-clean version with the range left open, skipped on Rust until a Cargo updater exists;
    - the human's submit and finalize.
  - **(r) Release task** (`--kind deployment`, blocked by (a)): the human's `vrg-release`, with its exact command, under `## Human-run steps` (check: the command exits 0), and the closing check that the release contains (a)'s merge.
  - **(b) Validation task** (`--kind validation`, blocked by (r)):
    - the preconditions, including the deployed-version probe for `--validates`;
    - every check in spec §6.3 (b), in order;
    - the known-gaps rule and today's two known gaps on Rust;
    - SUCCESS/FAILURE recording. Each failure is filed in the repo that owns the broken behaviour, never in the fixture. A fix that belongs in the fixture itself is a separate, ordinary task in the fixture's ad-hoc epic.
- **A tag that exists before its release workflow has finished** means blocked, not FAILURE (Review Focus 4).
- **A learned-conditions section** for "Ready to start", empty at first, where each run records a new condition it found.
- **The recipe files tasks only.** It never runs them, and the task-creation approval comes from the epic plan that seeded the run.

**`epic-create` rule** (spec §6.5), added to "Operational tasks": when an
epic changes language support, container images, shared CI workflows or
release tooling, its plan seeds a fixture run, with `fixture-run`, for each
fixture whose language or shape the change affects. The run's change task is
blocked by the deployment or release task that ships the change (passed as
`--after`), never only by the code task. A change that affects every
language seeds all four.

The same rule goes into vergil-tooling's
`docs/site/docs/standards/github-issues.md`, the living conventions page,
through the documentation-review bookend (vergil-tooling#4317), which lands
in that repo. Convention #40 is a closed epic's spec and is not rewritten.

- [ ] **Step 1:** Write `skills/fixture-run/SKILL.md`, the `epic-create` rule and the index entry.
- [ ] **Step 2:** Run `vrg-container-run -- vrg-validate`, which includes the plugin's SKILL.md frontmatter check in `validate-custom`. Expected: pass.
- [ ] **Step 3:** Commit with `vrg-commit --type feat --scope skills --message "fixture-run recipe and the epic-create seeding rule"`, and hand off with report-ready.

---

### Task 8: Release the plugin and update the agent boxes (deployment)

**Precondition (human-attested):** the human has released
vergil-claude-plugin containing Task 7, and updated the host and VMs
(`vrg-host plugins`, `vrg-vm update --all`).

`issue-deploy` confirms that a new session lists `vergil:fixture-run`.

---

### Task 9a–9d: One proof run per fixture

Once Tasks 6x and 8 are closed, run
`fixture-run vergil-project/.github#1091 <fixture> --validates "vergil-tooling <Task 4 version>" --after <Task 4 ref> --after <Task 8 ref>`
for each of the four fixtures. Each run files a trio:

- the change task, in the fixture repo;
- the release task, in the fixture repo;
- the validation task, in the fixture repo.

All three are linked under #1091. Record their refs as a dated entry in
`## Evolution during execution`. Run
them one at a time per fixture, with their own skills.

Goal 5 is met when all four validation tasks record `Outcome: SUCCESS`.
Known gaps don't count against that; the two expected today, both on the
Rust fixtures, are the evidence bundle and the dependency update.

---

## Coverage

- **Spec coverage:**

  | Spec | Plan |
  |---|---|
  | §2 goals 1–2 | Tasks 5–6 |
  | §2 goal 3 | Task 7 |
  | §2 goal 4 | Task 7 |
  | §2 goal 5 | Task 9 |
  | §6.1 | Task 5 |
  | §6.2 | Tasks 5–6 |
  | §6.3 | Tasks 7, 9 |
  | §6.4 table | Tasks 2–3; known gaps in Tasks 7 and 9 |
  | §6.4 later gaps | Task 6 |
  | §6.5 | Task 7 |
  | §7 placement | the delivery table |

- **Releases gate the fixtures** (spec §6.5): Task 4 gates Task 5, and Task 8 gates Task 9.
