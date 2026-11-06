# Remove vestigial CLIs and skills

- **Epic:** vergil-project/.github#910
- **Planning issue:** vergil-project/.github#831 (review history is in its
  comments)
- **Status:** draft
- **Repos:** vergil-tooling (the CLIs, their library code, tests and site
  docs), vergil-claude-plugin (the `migrate-repo` skill and the references to
  it, and the host tool list), `docs` (the fleet tools-and-skills page),
  `.github` (this spec and plan)

## 1. Problem

vergil-tooling installs 78 `vrg-*` commands. Most of them are the fleet's
working tools, but some were built for one-off jobs that are now finished, or
were replaced by later tools. Each one still carries its command module, its
unit tests, its reference docs and its mentions in skills and hooks. That code
is built, linted, type-checked and tested on every vergil-tooling change, and
every reader of the docs or the code has to work out that it is not used.

The test suite is about 122,000 lines. Removing tools that do nothing is the
cheapest way to make it smaller and faster.

## 2. How the candidates were found

Every command was checked against everything that could call it: other
vergil-tooling code, the workflows and composite actions in vergil-actions,
the vergil-claude-plugin skills and hooks, vergil-vm and vergil-containers.
History was left out (release notes, changelogs, epic and plan documents).
Commands that nothing calls were then read one by one and decided with the
human. Usage counts from session history were not available, so a command
that the human runs by hand was kept whenever the human said so.

## 3. Decisions

### 3.1 Remove

| Item | Repo | What it was | Why it goes |
| --- | --- | --- | --- |
| `migrate-repo` skill | vergil-claude-plugin | One-time pass that brought each repo's old issue backlog into the epic/task framework | Every repo has been migrated, and new repos start inside the framework. The `vrg-epic-audit` orphan check and `intake-review` handle any issue that ends up outside the model. |
| `vrg-adhoc-migrate` (and `lib/adhoc_migrate.py`) | vergil-tooling | One-shot move of per-repo standing epics into the org `.github` (epic #85) | The move is done. Nothing else uses the library. |
| `vrg-wait-until-green` | vergil-tooling | Blocks until a PR's checks pass | Left over from an older PR workflow. The merge and release code (`lib/pr_merge.py`, `lib/release/merge.py`) wait through `github.wait_for_checks`, which stays. |
| `vrg-changelog` | vergil-tooling | Hand-run wrapper around git-cliff | `vrg-release` calls the changelog library directly. The library stays. |
| `vrg-container-test` | vergil-tooling | Runs a repo's tests in a dev container, guessing the language | Replaced by `vrg-container-run -- vrg-validate`, the only sanctioned validation command. |
| `vrg-resolve-tooling-version` | vergil-tooling | Reads the vergil-tooling install tag from `vergil.toml` | No workflow or action calls it any more. |
| `vrg-pr-fix-body` | vergil-tooling | Rebuilds a PR body through the validated body builder | `pr-watch` does not use it. The body builder (`lib/pr_body.py`) and `lib/linkage.py` stay, because `vrg-submit-pr` uses them. |
| `vrg-epic-unlink` | vergil-tooling | Removes a task's sub-issue link to its epic | `vrg-epic-move` covers moving a task to another epic. Unlinking with no new parent is rare and `vrg-gh` can do it. |

### 3.2 Drop the standalone command only

`vrg-repo-profile` checks `vergil.toml` and the README structure. `vrg-validate`
runs it in process (`validate_common` imports the module and calls `main`), and
nothing runs it as a command. The `vrg-repo-profile` entry point goes away. The
module, the check and its tests stay.

Its docs change as follows:

- `docs/site/docs/reference/lint/repo-profile.md` stays, because it is the only
  per-key reference for `vergil.toml` and other pages link to it. It loses its
  "Installed as" line and its usage block, and describes the check as one step
  of `vrg-validate`.
- The guide pages that mention the command (`getting-started.md`, `index.md`,
  `guides/ci-architecture.md`, `guides/consuming-repo-setup.md`,
  `guides/validation-matrix.md`, `reference/index.md` and
  `standards/repository-structure.md`) are reworded the same way.
- Its `### vrg-repo-profile` entry leaves
  `docs/site/docs/reference/cli-tools-overview.md`, which a test requires to
  match `[project.scripts]` exactly.

### 3.3 Keep

`vrg-reword` (the only sanctioned way to fix a commit message on a branch),
`vrg-github-repo-init` (the human creates new repos with it),
`vrg-adhoc-epic` and `vrg-ensure-label`.

`vrg-audit-approve` also stays. It posts the `vergil-audit/approved`
check-run for the AUDIT identity, whose use is on hold until the fleet starts
building on the agentic APIs. It is kept as the placeholder for that work, and
the plugin's AUDIT write guard and its docs, which name the check, are left
unchanged.

### 3.4 Moved to other issues

- **The fleet sync tools** (`vrg-fleet-sync`, `vrg-ci-sync`, `vrg-xdist-sync`,
  `vrg-gitignore-sync` and `vrg-dependabot-sync`) are reviewed as a family in
  planning issue vergil-project/.github#909, which replaces them with automatic
  propagation. They are not touched here.
- **`vrg-generate-commands`** belongs in the mq-rest-admin project:
  vergil-project/.github#908 (deferred).

## 4. What a removal covers

Removing a command means removing all of the following:

1. its `[project.scripts]` entry in `pyproject.toml`;
2. its `src/vergil_tooling/bin/` module;
3. any library module, function or constant that no remaining source code
   calls, checked with a search after the removal. Tests are not callers:
   code that only tests still use is dead, and goes with those tests. For
   example `lib/adhoc_migrate.py`, and `_DEFAULT_TEST_COMMANDS` and
   `detect_language` in `lib/container.py`;
4. the tests of everything removed. A test that covers shared code which stays
   is kept, and only its references to the removed command are changed;
5. its entry in `docs/site/docs/reference/cli-tools-overview.md`, its own
   reference page if it has one, and its `mkdocs.yml` nav entry;
6. any mention in comments, docstrings and help text that names it as a live
   tool (for example the `migrate-repo` mentions in `vrg_epic_create.py` and
   `vrg_adhoc_epic.py`, and the comments in `lib/container.py`,
   `lib/pr_body.py`, `lib/linkage.py` and `data/host_dependencies.toml`);
7. its entry in a tool list outside vergil-tooling (the `HOST_TOOLS` list in
   vergil-claude-plugin's `hooks/scripts/lib/host-container-tools.sh` names
   `vrg-wait-until-green`).

Recorded test fixtures that happen to contain a command's name (for example
`tests/vergil_tooling/fixtures/host/uv-tool-list.txt`) are captured output, not
references. They change only if a test needs them to match the installed
commands.

History stays as it is: release notes, changelogs, and epic, spec and plan
documents are records of what was true at the time.

## 5. Order and releases

Each repo's change is independent. The plugin's tool list naming a command
that is no longer installed does nothing harmful, and no skill calls any
removed command. So the tasks can land in any order and ship in each repo's
normal release. No deployment or validation task is needed: the change takes
effect when the hosts and VMs pick up the next releases, the same as any
other change.

These are removals from the public command set, but every consumer is inside
the fleet and none calls them. Each repo bumps its version by its usual rules.

## 6. Success

- No live file in the fleet names a removed item as something to run (the
  search in §4 comes back with only history and recorded fixtures).
- `vrg-container-run -- vrg-validate` passes in each changed repo.
- The vergil-tooling task records the test count and the wall-clock time of
  the test stage before and after the change in its PR. The retrospective
  reports them.

## 7. Out of scope

- The fleet sync tools (#909) and `vrg-generate-commands` (#908).
- Any change to how the remaining commands behave.
- A standing process or audit for finding unused commands in future.
