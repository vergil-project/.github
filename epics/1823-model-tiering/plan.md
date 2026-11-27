# model-tiering: per-skill model selection — implementation plan

- **Epic:** vergil-project/.github#1823
- **Planning issue:** vergil-project/.github#1822 (review history is in its
  comments)
- **Status:** draft

> **For agentic workers:** each implementation task is one PR, run with
> `issue-implement`; operational tasks are run with `issue-deploy` or
> `issue-validate`. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** every subagent the fleet dispatches runs on a model chosen by
design: task runs drive on Sonnet, changes are written on Haiku, the merge
reviewer runs on Sonnet, design reviews stay on Opus, and guards keep the
pins from drifting.

**Architecture:** two new agents carry the pins, `vergil:task-runner`
(Sonnet, the driver `epic-implement` dispatches for every task run) and
`vergil:implementer` (Haiku, the writer the driver dispatches for every
change to the task's tracked files). The skills change only where they
dispatch: `issue-implement`, `merge-review`'s fix step and `pr-watch` hand
edits to the writer, and every other helper dispatch names its model. A CI
check fails on an agent with no `model:`, a PreToolUse hook on the Agent
tool refuses a task run dispatched as anything but an unpinned-at-call
`vergil:task-runner`, and one site page, the model map, records every
model and why.

**Tech Stack:** Markdown agent definitions and skills
(vergil-claude-plugin), bash hook scripts and their bash tests, the plugin's
`scripts/bin/validate-custom`, MkDocs site docs.

**Spec:** [`spec.md`](spec.md) in this directory. Section references (§5.3)
are to it.

## Global Constraints

- Every code task lands in `vergil-project/vergil-claude-plugin`. Paths
  below are relative to its root.
- Model values are the aliases `haiku`, `sonnet` and `opus`, exactly as
  written; no full model IDs in this epic.
- Agent names: `vergil:task-runner` (`agents/task-runner.md`, `model:
  sonnet`) and `vergil:implementer` (`agents/implementer.md`, `model:
  haiku`). `vergil:merge-reviewer` becomes `model: sonnet`;
  `vergil:bootstrap` stays `model: haiku`.
- No skill gains a frontmatter `model:` (it lasts one turn, §3). A pin
  lives in an agent definition, or in a dispatch's per-invocation `model`.
- A `vergil:task-runner` dispatch never carries a per-invocation `model`
  (§5.5).
- Helper model rule (§5.3 "Helpers"), used by every skill that dispatches
  a helper: a read-only search or exploration helper is dispatched with
  `model: haiku`; any other helper with `model: sonnet`; a design-review
  delegate with `model: opus`. The one exception is `issue-validate`'s
  skill-under-test subagent, which takes the skill's validation alias from
  the model map.
- The model map is `docs/site/docs/agents/models.md`, titled "Model map",
  under Agents in `docs/site/mkdocs.yml`'s nav. It is the single source of
  models: every later change to a pin updates it in the same PR (§6).
- Every task ends with `vrg-container-run -- vrg-validate` green and
  commits with `vrg-commit`. Hook changes are test-first.
- The plugin release is a human action; the deployment task never cuts
  one. Per the plugin's `skills/issue-deploy/references/release-coverage.md`,
  the release installs the plugin on the host that ran it and in every
  running VM; what it leaves over is `/reload-plugins` in each running
  session, a human-run step.

## Review Focus

1. **A near-miss task description.** `epic-implement` writes `Task 6b
   #4931` (a short ref) or adds a trailing word. Expected: the hook's
   anchored forms do not match it, but its first word is `Task` or
   `Batch`, followed by a space, so the hook refuses it whatever its `subagent_type`, naming the forms;
   the dispatch never runs on an inheriting agent type (Task 3,
   `test_short_ref_refused_as_malformed`,
   `test_trailing_word_refused_as_malformed`).
2. **A writer that claims `green` falsely.** The writer returns `green`
   with an uncommitted edit, or with `vrg-validate` red at `HEAD`.
   Expected: the driver's own check (clean tree, the commits on the branch,
   its own green validate) catches it before merge review, and the driver
   re-dispatches the writer once on Sonnet rather than fixing it itself; a
   second failure of either kind stops the run for the human (Task 4a, the
   "Check the writer's result" and "Escalation" steps).
3. **An interrupted writer on resume.** A re-dispatched run finds the
   worktree dirty with the writer's uncommitted work. Expected: the driver
   does not finish or discard it itself; it dispatches the writer to finish
   it, naming the dirty state (Task 4a).
4. **An interrupted rebase on resume.** A re-dispatched run finds a
   `vrg-git rebase` stopped mid-way (a conflict or a batch fold).
   Expected: the driver finishes or aborts it itself under its rebase
   rules and never hands it to the writer (Task 4a, "Resume").
5. **Batch commit shape.** In a batch, the writer's commits for the
   second issue are not fixups of that issue's first commit. Expected: the
   driver's check reads `vrg-git log` and re-dispatches the writer to
   correct the shape (Task 4a).
6. **A double `stuck` on a merge-review fix.** Expected: the run stops for
   the human; the driver never falls back to writing the fix itself (Task
   4b).

---

## Delivery

Every task is filed under the epic, vergil-project/.github#1823, in the repo
its PR lands in. The plan task ID is the `Plan task:` key `file-tasks`
writes.

| Task | Repo | Kind | What | Blocked by |
|---|---|---|---|---|
| 1 | `.github` | task | Spec and plan (vergil-project/.github#1824) | — |
| 2 | `vergil-claude-plugin` | task | The two new agents, the merge reviewer on Sonnet, the agent model guard and the model map | — |
| 3 | `vergil-claude-plugin` | task | The task-dispatch hook; `epic-implement` dispatches every task run as `vergil:task-runner` | 2 |
| 4a | `vergil-claude-plugin` | task | `issue-implement` hands every tracked-file change to the writer | 2 |
| 4b | `vergil-claude-plugin` | task | `merge-review`'s fix step takes the writer from its caller; `pr-watch` uses it under an `issue-implement` driver | 4a |
| 5 | `vergil-claude-plugin` | task | Every other helper dispatch names its model; design-review rounds pin the delegate to Opus | 3, 4b |
| 6 | `vergil-claude-plugin` | deployment | A plugin release carries Tasks 2–5; every running session reloads its plugins | 2, 3, 4a, 4b, 5 |
| 7 | `vergil-claude-plugin` | validation | The transcripts of a real task run and a design-review round show the pinned models | 6 |
| doc-review | `vergil-claude-plugin` | task | Documentation review: `Documentation review: model-tiering` | 7 |
| retrospective | `.github` | retrospective | `Retrospective: model-tiering` | all others |

No fixture run is seeded: the epic changes no language support, container
image, shared CI workflow or release tooling. No step needs fresh `dev-`
images.

### Notes for `file-tasks`

- **Advisories.** Several advisories track the planning issue
  (`vrg-security-record list` names them; take the set from there, never
  from a count). Before `Tasks filed: yes`,
  retarget each from vergil-project/.github#1822 to the task or tasks that
  fix it, chosen from the advisory's own notes, one retarget after another.
  Every candidate fix task is in `vergil-project`, so no advisory moves.
- **The wave-2 roadmap** (§7) is recorded on the model map (Task 2). Its
  items become follow-on planning items when the epic closes: the
  retrospective records them in its §5, and each is filed in the planning
  queue only as the human approves. They are not tasks of this epic.

---

## Task 2: Agents, the model guard and the model map (vergil-claude-plugin)

Spec §4, §5.2, §5.4, §5.5 (the CI guard), §6.

**Files:**

- Create: `agents/task-runner.md`, `agents/implementer.md`,
  `docs/site/docs/agents/models.md`
- Modify: `agents/merge-reviewer.md` (`model: opus` → `model: sonnet`),
  `scripts/bin/validate-custom`, `docs/site/docs/agents/index.md`,
  `docs/site/mkdocs.yml` (nav)
- Test: `hooks/tests/agent-models.test.sh` (new; `validate-custom` runs
  every `hooks/tests/*.test.sh`)

**Interfaces:**

- Produces: the agent types `vergil:task-runner` and `vergil:implementer`;
  the implementer's result block (below), which Tasks 4a and 4b parse; the
  model map page and its two tables, which Task 5 reads the validation
  alias from; `check_agent_models` in `validate-custom`.

**The agents.**

- `agents/task-runner.md`: frontmatter `name: task-runner`, a
  `description` saying it is dispatched only by `epic-implement` for one
  task run (or batch) of `issue-implement`, `issue-validate` or
  `issue-deploy`, `model: sonnet`, and **no `tools:` line**, so it inherits
  the full tool set as `general-purpose` does (it needs Agent and Skill).
  Body: it runs the routed skill under the Agent prompt contract named in
  its prompt; the skills carry the procedure.
- `agents/implementer.md`: `name: implementer`, `tools: Read, Edit, Write,
  Glob, Grep, Bash`, `model: haiku`. Body, from §5.2: in the given
  worktree, carry out the given change to the task's tracked files; commit
  in small `vrg-commit` commits in the convention the prompt gives;
  validate with `vrg-container-run -- vrg-validate` until green. It never
  suppresses a gate, never submits, pushes or posts on GitHub, never
  creates issues, never runs merge review, and never edits anything under
  `build/.vergil-state/`. It returns `stuck` once five `vrg-validate` runs
  in one dispatch have all exited non-zero, judged by exit code alone. Its
  last message is exactly this block:

  ```text
  Implementer result: green|stuck
  Commits: <sha> <subject>   (one line each, oldest first; "none" if none)
  Validate runs: <n>, last exit <code>
  Tried: <stuck only: what was tried and what still fails>
  Notes: <deviations from the issue and why; concerns noticed; or "none">
  ```

**The guard.** `check_agent_models` in `validate-custom`, called once over
every `agents/*.md`:

- each file's frontmatter has exactly one `model:` line (reuse the
  one-line count `check_agent_tools` does for `tools:`);
- its value is `haiku`, `sonnet`, `opus` or `inherit`;
- `inherit` passes only if `docs/site/docs/agents/models.md` has an agents
  table row whose first cell is `` `vergil:<name>` ``, whose model cell is
  `inherit` and whose reason cell is not empty;
- every `agents/*.md` has an agents-table row in the model map whose first
  cell is `` `vergil:<name>` `` and whose Model cell equals its `model:`
  value, and every `vergil:` row names an existing agent file.

Also add `check_agent_tools implementer "Read, Edit, Write, Glob, Grep,
Bash"`, so the writer can never gain Agent.

**The model map** (`docs/site/docs/agents/models.md`), from §3, §4, §6, §7:

- the pinning facts of §3, with the two doc links and their check date;
- an **agents table**: Agent | Model | Reason, one row per file in
  `agents/` plus `paad:paad-delegate` (`opus`, by the dispatch) and
  `paad:paad-analyst` (`inherit`: Opus from the delegate that dispatches
  it, with that reason);
- the **helper model rule** of the Global Constraints, with its one
  exception;
- a **skills table**: Skill | Where it runs | Model there | Validation
  alias. A skill that runs in more than one place has one row per place,
  its alias on each (§6): `issue-implement`, `merge-review` and `pr-watch`
  take `sonnet`; `issue-deploy`, `issue-validate` take `sonnet`; every
  skill that runs only on a session's model takes `opus`, `fixture-run`
  (on its caller's session) included;
- how to change or revert a pin (edit the agent's `model:`, or the
  dispatch's `model`, and this page in the same PR), and that the human
  may run `epic-implement` on Sonnet with `/model`;
- the escalation's per-invocation `model: sonnet` recorded as **not
  verified** (§8);
- the wave-2 roadmap of §7, one bullet per candidate.

`docs/site/docs/agents/index.md` drops its two `**Model**:` lines, gains
entries for the two new agents, and links to the model map.

- [ ] **Step 1: Write the failing tests** in
  `hooks/tests/agent-models.test.sh`, sourcing `validate-custom` (as the
  hook-modes tests do) against a scratch root:
  `test_agent_without_model_fails`, `test_two_model_lines_fails`,
  `test_unknown_model_value_fails` (`gpt-5`),
  `test_inherit_without_map_row_fails`,
  `test_inherit_with_map_row_and_reason_passes`,
  `test_inherit_with_empty_reason_fails`, `test_map_model_mismatch_fails`,
  `test_agent_missing_from_map_fails`, `test_map_row_without_agent_fails`,
  `test_pinned_agents_pass`.
- [ ] **Step 2:** run `vrg-container-run -- bash
  hooks/tests/agent-models.test.sh`; expected: FAIL (no
  `check_agent_models`).
- [ ] **Step 3:** implement `check_agent_models`, wire it and the
  implementer tools check into the main block; add the two agents, the
  reviewer pin, the model map, the nav entry and the agents-page change.
- [ ] **Step 4:** the test file passes; `vrg-container-run --
  vrg-validate` is green; commit.

## Task 3: The task-dispatch hook and `epic-implement`'s dispatch (vergil-claude-plugin)

Spec §5.5 (the hook), §6 (the three `general-purpose` places). The hook
and the dispatch change land together: the hook alone would refuse every
dispatch `epic-implement` makes today.

**Files:**

- Create: `hooks/scripts/guard-task-dispatch.sh` (mode 100755),
  `hooks/tests/guard-task-dispatch.test.sh`
- Modify: `hooks/hooks.json` (a PreToolUse entry, `"matcher": "Agent"`),
  `hooks/tests/hook-input.test.sh` (the new gate in its fail-closed suite),
  `skills/epic-implement/SKILL.md` (the frontmatter `description`, §3
  "Work the runnable frontier", §4 "Batch at the gate, then stop" (its
  gate paragraph and its frozen-branch re-entry paragraph), and the adhoc
  path's Steps 5 and 6), `hooks/register.tsx` (header comment only),
  `docs/site/docs/hooks/index.md` ("Agent row labels", and a section for
  the new hook)

**Interfaces:**

- Consumes: the `vergil:task-runner` agent type (Task 2).
- Produces: the hook's refusal messages, which Task 7 checks for.

**The hook.** Built like `block-agent-merge.sh`: `hook_input_read gated`,
the managed-repo check, then read `tool_input.description`,
`tool_input.subagent_type` and `tool_input.model` with `jq`. A missing or
non-string `description` is refused fail-closed, as the hook-input library
does for a missing required field. A description is a **task run** when it
matches, whole, one of these (POSIX ERE):

```text
^Task [^ ]+ [A-Za-z0-9_.-]+/[A-Za-z0-9_.-]+#[0-9]+$
^Task [A-Za-z0-9_.-]+/[A-Za-z0-9_.-]+#[0-9]+$
^Batch [^ #][^ ]*( #[0-9]+)+$
^Batch( #[0-9]+)+$
```

Then deny the call, as `block-agent-merge.sh` does (a
`permissionDecision: "deny"` JSON response on stdout, exit 0, its
`permissionDecisionReason` the one-line reason below; exit 2 stays for
input that cannot be parsed), when:

- it is a task run and `subagent_type` is not exactly
  `vergil:task-runner` (absent counts as not): `task run '<description>'
  must be dispatched as vergil:task-runner, not <type>`;
- `subagent_type` is `vergil:task-runner` and `model` is present and
  non-empty: `vergil:task-runner must not be dispatched with a model; its
  pin is in the agent definition`;
- it is not a task run but the description matches `^(Task|Batch)[ ]`
  (its first word is `Task` or `Batch`, followed by a space), whatever its
  `subagent_type` (absent included): `description
  '<description>' starts like a task run but matches none of the task-run
  forms; use one of: Task <id> <owner>/<repo>#<N>, Task
  <owner>/<repo>#<N>, Batch <ids> #<N>…, Batch #<N>…`.

Everything else exits 0 with no output.

**The skill.** In `epic-implement`, every task-run dispatch (a re-dispatch
included) passes `subagent_type: "vergil:task-runner"` and never `model`.
The main session no longer runs `pr-watch` on a task's PR itself: when the
human reports that a task's PR did not merge or asks for it to be fixed,
and when a reported-ready task with an open PR has no run watching it, it
dispatches a task run of `issue-implement` (in a task-run form, as
`vergil:task-runner`), whose Preflight resumes at "Submit and watch to the
merge" and runs `pr-watch` on that PR under its driver, so the writer and
the Sonnet pin apply (spec §5.3, "`epic-implement`'s re-entries"). Nor does it submit a task
itself: a reported-ready task whose session was lost before submit (no PR
yet) is re-dispatched the same way, as a task run of `issue-implement`,
which its Preflight resumes at "Submit and watch to the merge". Both
re-entries take this one path. The sentence that says the mod labels the row "instead of
`general-purpose`" says `vergil:task-runner`, and the table's intro says
the forms are matched exactly, since the dispatch hook keys on them.

- [ ] **Step 1: Write the failing tests** in
  `guard-task-dispatch.test.sh`, with a realistic PreToolUse payload in a
  scratch managed repo: `test_each_form_refused_as_general_purpose` (the
  four forms, e.g. `Task 6b vergil-project/vergil-tooling#4931`, `Batch
  6b/6c/9 #4917 #4918 #4926`, `Task vergil-project/vergil-tooling#5012`,
  `Batch #5012 #5013`), `test_each_form_allowed_as_task_runner`,
  `test_non_digit_id_is_a_task_run` (`Task D2
  vergil-project/vergil-tooling#5012`),
  `test_missing_subagent_type_refused_for_task_run`,
  `test_writer_dispatch_naming_task_allowed` (`Write
  vergil-project/vergil-tooling#4931 (Task 6b)` as `vergil:implementer`),
  `test_short_ref_refused_as_malformed` (`Task 6b #4931`, refused both as
  `general-purpose` and as `vergil:task-runner`),
  `test_trailing_word_refused_as_malformed`,
  `test_task_word_mid_description_allowed` (`Run Task 6b checks` as
  `general-purpose`, allowed: only a first word `Task` or `Batch`
  counts), `test_tasks_word_allowed` (`Tasks overview` as
  `general-purpose`, allowed),
  `test_task_runner_with_model_refused`,
  `test_task_runner_with_empty_model_allowed`,
  `test_unmanaged_repo_noop`, `test_pr_watch_reentry_form_allowed_as_task_runner`
  (a re-entry's task-run description as `vergil:task-runner`, for both the
  open-PR and the no-PR re-entry). Add the gate to `hook-input.test.sh`'s
  malformed-input suite.
- [ ] **Step 2:** run both test files; expected: FAIL (no script).
- [ ] **Step 3:** implement the script, register it, change the skill, the
  `register.tsx` header comment and the hooks page.
- [ ] **Step 4:** check: `grep -n 'pr-watch' skills/epic-implement/SKILL.md`,
  and settle every hit: none tells the main session to run `pr-watch` on a
  task's PR itself, and each that names a re-entry names the task run.
- [ ] **Step 5:** both test files pass; `vrg-container-run -- vrg-validate`
  green (its hook-mode check covers the new script); commit.

## Task 4a: `issue-implement` hands every change to the writer (vergil-claude-plugin)

Spec §5.3 (Implement, The writer rule, Security fixes, Escalation, Helpers,
The human-run path).

**Files:**

- Modify: `skills/issue-implement/SKILL.md`: "Execute efficiently — trust
  and escalate" (the sub-agents paragraph), the implementation step, Batch
  mode, and a new section `## The writer`

**Interfaces:**

- Consumes: `vergil:implementer` and its result block (Task 2).
- Produces: the anchor `issue-implement/SKILL.md#the-writer`, which Task 4b
  links from `merge-review` and `pr-watch` for the dispatch and the check.

**The writer section** states, in this order:

- **The rule.** Every change to the task's tracked files in its worktree
  (code, docs, config) is made by `vergil:implementer`, never by an ad-hoc
  helper and never by the driver. The two exceptions, stated here: the
  driver makes a security item's fix itself and never passes its details
  to the writer; and the driver resolves a stopped `vrg-git rebase`'s
  conflicts itself, logic included. Files under `build/.vergil-state/` and
  scratch tool inputs stay with the driver and never go to the writer.
- **The dispatch.** `subagent_type: "vergil:implementer"`, `description:
  "Write <owner>/<repo>#<N>"` (never a task-run form), with a prompt
  holding: the issue exactly as `vrg-issue-read` prints it, the rulings
  being carried out, the worktree's absolute path, the commit convention
  (a plain first commit, or in a batch the issue's first commit then
  `vrg-commit --fixup <that commit>`), and the instruction to end with the
  result block.
- **Check the writer's result** before anything else reads it: the block is
  well formed; `vrg-git -C <worktree> status --porcelain` is empty; the
  commits it names are on the branch beyond the base and, in a batch,
  follow the fixup convention; and the driver's own `vrg-container-run --
  vrg-validate` is green at `HEAD`. The driver never patches a failed
  check; it is a failure of the writer, handled by the escalation below. A
  fabricated `green` never reaches merge review.
- **Escalation.** A writer failure is a `stuck` result or a `green` that
  fails the check. On the first, re-dispatch once with `model: "sonnet"`,
  passing the first attempt's `Tried:` line or what the check found. One
  budget per dispatch (the implementation, each merge-review fix, each CI
  fix), shared by both kinds. A second failure of either kind stops the run
  for the human as an unresolvable problem does today.
- **Resume.** A dirty worktree on a re-dispatch is sorted before anything
  is dispatched. A `vrg-git rebase` in progress (a conflict resolution or a
  batch fold) stays with the driver, which finishes or aborts it as its
  rebase rules say. Uncommitted work from the driver's own exceptions, a
  security item's fix included, stays with the driver, which finishes it.
  Before it starts such work, the driver writes a marker file in the task
  worktree's own git directory, at the path `vrg-git -C <worktree>
  rev-parse --git-path vergil-driver-work` prints, naming the exception (a
  security fix or a rebase), and removes it once that work is committed.
  The marker is never tracked, sits outside every state directory (so
  doctrine §1.5's rules never see it), and goes with the worktree; the
  writer never touches it. On resume the marker alone decides that a dirty
  tree is the driver's, never a pending security details file or any
  other inference.
  Only the rest is the writer's
  interrupted work: dispatch the writer to finish it, naming the dirty
  state.
- **Notes.** The writer's `Notes:` are screened and handled as follow-ups,
  as a concern any sub-agent reports is today.
- **The human-run path.** `/issue-implement` in the human's session
  dispatches the writer the same way.

The sub-agents paragraph becomes the helper model rule of the Global
Constraints, with the writer rule linked; it no longer invites a helper to
make changes.

- [ ] **Step 1:** edit the skill as above.
- [ ] **Step 2:** check: `grep -n -i 'sub-agent\|subagent'
  skills/issue-implement/SKILL.md` shows no dispatch without a named model
  or the writer; `vrg-container-run -- vrg-validate --markdown --fix
  skills/issue-implement/SKILL.md` clean.
- [ ] **Step 3:** `vrg-container-run -- vrg-validate` green; commit.

## Task 4b: The writer in `merge-review`'s fix step and `pr-watch` (vergil-claude-plugin)

Spec §5.3 (Fixes, Who has a writer, merge-review's fix step, Rebase
conflicts).

**Files:**

- Modify: `skills/merge-review/SKILL.md` ("Fix and check", and wherever
  it says who makes a fix to an open PR), `skills/pr-watch/SKILL.md`
  (its fix steps, its conflict handling, and the "sub-agents remain fine
  elsewhere" line, ~73)

**Interfaces:**

- Consumes: `issue-implement/SKILL.md#the-writer` (Task 4a).

**merge-review.** "Fix and check" gains an input from the caller: whether
it has a writer. Only an `issue-implement` driver has one (itself, and
`pr-watch` when that driver runs it). With a writer, in this order: the
driver first makes every security-item fix itself and commits it (in a
batch, as a fixup of the issue's first commit), then runs
`vrg-container-run -- vrg-validate` itself to green at `HEAD`, fixing any
failure inside its own security change, so the writer never starts from
a tree its security fixes turned red; then step 1's file edits
for every non-security defect and every ruling that calls for a code
change, and step 2's validation over the whole tree, go to the writer as
`issue-implement#the-writer` says, with the reviewer's findings (by ID and
text, security items left out) and the rulings in its prompt; the driver
checks the result there as well, its green `vrg-validate` at `HEAD`
covering the security fixes too, then does every filing step 1 calls for,
then builds the check (step 3). A validate failure that the driver's
security fix causes is the driver's to fix, inside that security change,
never the writer's; the driver's own green run before the writer's
dispatch is what keeps it there. With no non-security edit to make, that
green run is step 2's validation, and the writer is not dispatched.
Without a writer,
the caller's session does steps 1 and 2, as today.

**pr-watch.** Under an `issue-implement` driver, the code of a fix for
failing CI or a review comment goes to the writer; the driver still
diagnoses, checks and pushes. A stopped rebase's conflicts are resolved by
the driver itself, logic included, and merge review gates the result.
`epic-implement` never runs `pr-watch` on a task's PR in its own session:
it dispatches a task run for it (Task 3). Run standalone or by
`epic-create`, `epic-retrospective` or `hoshin`, `pr-watch` has no writer
and fixes in the session. Line 73's helper
sentence takes the helper model rule.

- [ ] **Step 1:** edit both skills; each "who fixes" statement names the
  writer case and the no-writer case.
- [ ] **Step 2:** `vrg-container-run -- vrg-validate --markdown --fix`
  on both files, then a full `vrg-container-run -- vrg-validate`; commit.

## Task 5: Every other helper names its model (vergil-claude-plugin)

Spec §5.3 (Helpers), §5.4 (the design-review pin).

**Files:**

- Modify: `skills/epic-implement/SKILL.md` ("Use sub-agents wherever they
  make this efficient", §3), `skills/issue-deploy/SKILL.md` (~33),
  `skills/issue-validate/SKILL.md` (~31, and the skill-under-test subagent
  at ~91), `skills/epic-create/SKILL.md` (design-review round step 1, the
  `paad:paad-delegate` dispatch sentence),
  `docs/paad-review-doctrine.md` (§1.9 "Dispatch" and §2.2's dispatch
  step), and any other skill the sweep below finds

**Interfaces:**

- Consumes: the model map's skills table (Task 2) for the validation
  alias; Task 3's `epic-implement` text (the task-run dispatch is not a
  helper and stays unpinned at call).

- [ ] **Step 1:** in `epic-implement`, `issue-deploy` and
  `issue-validate`, apply the helper model rule. In `issue-validate`, the
  subagent that loads a skill to check it is dispatched with `model` set
  to that skill's validation alias from the model map, read per run from
  the installed plugin's own copy, the same tree as the loaded skill,
  through the skill's relative link
  (`../../docs/site/docs/agents/models.md`), never from the cwd or a
  plugin clone, so the alias and the skill come from one release; a skill
  with no row there stops the validation for the human.
- [ ] **Step 2:** in `epic-create`'s round step 1 and in the doctrine's
  §1.9 and §2.2, the `paad:paad-delegate` dispatch carries `model:
  "opus"`. The paad fork is not changed.
- [ ] **Step 3: Sweep.** `grep -rn -i 'sub-agent\|subagent\|Agent tool\|dispatch'
  skills/` and check each hit: every helper dispatch names a model (or is
  the writer, the reviewer, the delegate or a task run). Change any other
  skill found the same way, and list in the PR notes each hit judged not
  to be a dispatch.
- [ ] **Step 4:** `vrg-container-run -- vrg-validate` green; commit.

## Task 6: Release the plugin (deployment)

- Preconditions: `vrg-release-record check` exits 0 for each of Tasks 2,
  3, 4a, 4b and 5.
- Procedure: those checks, alone; nothing is installed (the release did
  it).
- Human-run steps: `/reload-plugins` (or a restart) in every running Claude
  Code session, so each loads the new agents, the dispatch hook and the
  changed skills together. A session left on the old plugin keeps
  dispatching task runs as `general-purpose` on the session's model.
- Acceptance: every check passes and the human confirms the reload; the
  result names the release version.

## Task 7: The pinned models in real transcripts (validation)

Spec §8.

- Preconditions: Task 6 closed. Human-attested, naming the refs: after the
  reload, (a) one real implementation task of any epic was driven to its
  merge through `epic-implement`, and (b) one `epic-create` design-review
  round ran, on any planning issue. The attestation names the task, the
  planning issue and the round, and the session directory under
  `~/.claude/projects/` that holds each run's transcripts.
- Procedure (read-only):
  1. In the `epic-implement` session's transcript (`<session>.jsonl`),
     find the Agent `tool_use` for the task run and confirm
     `input.subagent_type == "vergil:task-runner"` and no `input.model`.
  2. In the subagent transcripts (`<session>/subagents/agent-*.jsonl`), for
     the task runner, its implementer dispatches and its merge-reviewer
     dispatch, list the models of their turns:
     `jq -r 'select(.type == "assistant") | .message.model' <file> | sort -u`.
     The task runner's are all Sonnet, the implementer's all Haiku (Sonnet
     only on a dispatch the runner's transcript shows with `model:
     "sonnet"`), the merge reviewer's all Sonnet.
  3. In the design-review session's transcript, the Agent `tool_use`
     dispatching `paad:paad-delegate` has `input.model == "opus"`.
  4. If the attested task's PR needed a fix after submit, or its submit
     was resumed after a lost session, the `epic-implement` session's
     transcript shows that re-entry dispatched as a task run
     (`input.subagent_type == "vergil:task-runner"`), not run in the main
     session; otherwise record that no re-entry occurred.
  5. In the validation run's own session (a managed repo), an Agent call
     with description `Task vergil-project/vergil-tooling#1` as
     `general-purpose` is refused with Task 3's message; cite the refusal
     (dispatch nothing else).
  If the transcript layout differs from the paths above, find the same
  fields where it keeps them and record where; if a model cannot be read
  at all, the outcome is FAILURE, never a guess.
- Acceptance: each step's evidence cited (file and the `jq` output);
  `Outcome: SUCCESS`. The escalation pin stays recorded as not verified.

## Documentation review and retrospective

- **Documentation review** (`vergil-claude-plugin`, `Plan task:
  doc-review`, blocked by Task 7): sweep the plugin site docs: the skills
  page sections of `issue-implement`, `merge-review`, `pr-watch`,
  `epic-implement`, `issue-validate`, `issue-deploy` and the execution
  doctrine for the writer and helper rules; the agents page and the model
  map against the agents as shipped; the hooks page for the dispatch hook.
  Spawn a per-repo doc task in the `docs` repo if its fleet overview names
  the models agents run on, under the epic.
- **Retrospective** (`.github`, `--kind retrospective`, `Plan task:
  retrospective`): `epic-retrospective`, once every other task is closed.
  Its §5 records the wave-2 candidates of §7 as follow-on planning items,
  filed as the human approves.
