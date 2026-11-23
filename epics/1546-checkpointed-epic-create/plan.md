# checkpointed-epic-create — implementation plan

- **Epic:** vergil-project/.github#1546
- **Planning issue:** vergil-project/.github#1493 (review history is in its comments)
- **Status:** approved

> **For agentic workers:** each task is one issue and one PR in the repo named
> in its heading, worked with `issue-implement`. Operational tasks
> (deployment, validation) are worked with `issue-deploy` and
> `issue-validate`. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** A fresh session, after `/clear`, resumes `epic-create` at the
exact stage it stopped, told by a tool which stage that is, with nothing
redone that created something and nothing lost.

**Architecture:**

- **vergil-tooling** adds `vrg-checkpoint`, which owns one delimited,
  strictly parsed stage checklist per workflow in a tracking issue's body,
  read and written only through the trust layer. Each workflow's stage list
  is defined once in the tool.
- **vergil-claude-plugin** restructures `epic-create` into the nine stages
  of spec §4.3: `status` on entry, a `tick` as the last write of each
  stage, the checkpoint pause, keyed task filing, skew handling and the
  in-flight adoption. `epic-review` abandons a bootstrapped skeleton before
  it defers or drops a planning item.

**Tech Stack:** Python 3 and pytest (vergil-tooling); Markdown skills
(vergil-claude-plugin).

**Spec:** [`spec.md`](spec.md) in this directory. Executors read both.

## Global Constraints

- **The stage list of `epic-create`**, in this order, exactly: `bootstrap`,
  `brainstorm`, `spec-pushback`, `spec-review`, `plan`, `plan-pushback`,
  `alignment`, `file-tasks`, `docs-pr` (spec §4.2, §4.3).
- **Block markers:** `<!-- vrg-checkpoint:<workflow> -->` and
  `<!-- /vrg-checkpoint:<workflow> -->`; a stage line is exactly
  `- [ ] <stage>` or `- [x] <stage>` (spec §4.2).
- **Strict parse:** any other line inside the block, or any text after a
  stage name, is a malformed block, reported with its own exit code, and
  nothing is written (spec §4.2).
- **`vrg-release` is unchanged:** `vergil_tooling/lib/release/checklist.py`
  and its lenient parse are not modified (spec §4.2).
- **Every body read goes through `trust.read_issue(..., require_body=True)`**;
  an untrusted body is a stop with its own exit code, and nothing is written
  (spec §4.7).
- **A write rewrites only the block;** every other byte of the body is kept
  (spec §4.1).
- **A tick is the last write of a stage** (spec §4.3): work, progress
  comment, then `tick`.
- **No silent remapping:** every change to the cursor after skew is shown to
  the human before it is made (spec §4.4).
- **Stable task keys:** a `Plan task: <id>` line in each filed task's body;
  the closing bookends use the fixed IDs `doc-review` and `retrospective`
  (spec §4.3, §4.5).

## Review Focus

1. **A box ticked by hand in the web UI** by a trusted identity. `status`
   reads it as done and names the next stage. Pinned in Task 2a
   (`test_status_reads_hand_tick`).
2. **An outsider edits the body,** ticking a box or not. Every operation
   exits 3 and writes nothing. Pinned in Task 2a
   (`test_untrusted_body_exits_3_and_writes_nothing`).
3. **A re-entry repeats a tick** already made. It succeeds and writes
   nothing, so every re-entry stays safe to repeat. Pinned in Task 2a
   (`test_tick_already_ticked_is_noop`).
4. **The rest of the body** (the `Planning:` line, the security lines, the
   Documents section, the human's text) passes through `init`, `tick` and
   `reconcile` byte for byte. Pinned in Task 2a
   (`test_write_preserves_rest_of_body`) and Task 2b
   (`test_reconcile_preserves_rest_of_body`).
5. **Two blocks for the same workflow** in one body (a pasted copy). It is
   malformed, not silently the first. Pinned in Task 2a
   (`test_duplicate_block_is_malformed`).

## Delivery table

Each line gives the task, its repo and kind, what it delivers, and what it
is blocked by.

- **1** (.github, task #1547): spec and plan, this PR.
- **2a** (vergil-tooling): `vrg-checkpoint` with `init`, `status`, `tick` and
  `reset`, the `epic-create` stage list, skew and malformed detection, and
  its docs.
- **2b** (vergil-tooling): `vrg-checkpoint reconcile`. Blocked by 2a.
- **3a** (vergil-claude-plugin): `epic-create` restructured into the stages,
  with `status`, `tick`, the checkpoint pause, the dirty-worktree and
  closed-issue rules, `bootstrap`, the early exits and the epic body write.
  Blocked by 4a.
- **3b** (vergil-claude-plugin): `epic-create`'s `file-tasks` keys, skew
  handling and in-flight adoption. Blocked by 3a.
- **3c** (vergil-claude-plugin): `epic-review` abandons a bootstrapped
  skeleton before deferring or dropping a planning item. Blocked by 3a.
- **4a** (.github, deployment): the vergil-tooling release carrying 2a and
  2b, installed on the host and every VM. Blocked by 2a, 2b.
- **4b** (.github, deployment): the vergil-claude-plugin release carrying
  the 3a–3c batch, installed on the host and every VM. Blocked by 3a, 3b,
  3c.
- **5** (.github, validation): a live planning run cleared and resumed at
  several checkpoints. Blocked by 4b.

### Bookends

This epic was created before its own change, so its closing bookends were
filed at creation, not in `file-tasks`.

- **Documentation review** (vergil-claude-plugin, bookend
  vergil-project/vergil-claude-plugin#1293). Blocked by 4b. Its sweep covers
  vergil-tooling's `docs/site/docs/reference/cli-tools-overview.md` and
  the docs repo's description of the planning workflow.
- **Retrospective** (.github, bookend #1548). Blocked by everything else.

Notes:

- **No fixture runs** and no `dev-` image validation: the change touches no
  language support, container image, shared workflow or release tooling.
- **One plugin batch, after the tool is released.** 3a, 3b and 3c are
  three issues landed as one batch PR in vergil-claude-plugin
  (`.worktrees/issue-<3a>-batch-checkpointed-stages/`), so no plugin release
  carries part of the change. The batch starts only once 4a is done: the
  skill calls `vrg-checkpoint`, so a plugin release must never reach a host
  before the command does.
- **Filing.** Every task above is filed at epic-create's filing step, in
  its repo, under #1546, with its `Blocked-by:` refs.

---

### Task 2a: `vrg-checkpoint` init, status, tick and reset (vergil-tooling)

Spec §4.1, §4.2, §4.3 (stage list), §4.4 (detection), §4.7.

**Files:**

- Create: `src/vergil_tooling/lib/checkpoint.py`
- Create: `src/vergil_tooling/bin/vrg_checkpoint.py`
- Modify: `pyproject.toml` (`vrg-checkpoint =
  "vergil_tooling.bin.vrg_checkpoint:main"`, beside `vrg-epic-docs`)
- Modify: `docs/site/docs/reference/cli-tools-overview.md` (a
  `### vrg-checkpoint` section beside `### vrg-epic-docs`),
  `docs/site/docs/reference/index.md` (a row beside `vrg-epic-docs`'s) and
  `docs/site/docs/guides/permission-model.md` (its list of tools that
  bypass the wrapper)
- Test: `tests/vergil_tooling/test_checkpoint.py`,
  `tests/vergil_tooling/test_vrg_checkpoint.py`

A sibling of `lib/release/checklist.py`, not a generalization of it: the
strict parse must not change what `vrg-release --resume` accepts. Follow
`vrg_epic_docs.py` for the read, the write and the error handling:
`trust.read_issue` with `require_body=True` under
`github.target_org(owner)`, then a `PATCH` of the body.

**Interfaces:**

- Produces, in `lib/checkpoint.py`:
  - `WORKFLOWS: Mapping[str, tuple[str, ...]]`, with `"epic-create"` mapped
    to the Global Constraints' stage list.
  - `class Malformed(Exception)`, `class Missing(Exception)`.
  - `@dataclass(frozen=True) class Skew: recorded: tuple[str, ...];
    current: tuple[str, ...]; added: tuple[str, ...]; removed: tuple[str, ...];
    reordered: bool`.
  - `begin(workflow: str) -> str`, `end(workflow: str) -> str`.
  - `render(workflow: str, checked: Iterable[str] = ()) -> str`: the block
    for the workflow's current list.
  - `parse(body: str, workflow: str) -> list[tuple[str, bool]]`: raises
    `Missing` with no block, `Malformed` for a second block, an unclosed
    block or any non-stage line.
  - `skew(pairs: Sequence[tuple[str, bool]], workflow: str) -> Skew | None`.
  - `next_stage(pairs: Sequence[tuple[str, bool]]) -> str | None`: the
    first unticked stage.
  - `upsert(body: str, block: str, workflow: str) -> str`: replaces the
    block in place, or appends it after a blank line.
  - `tick(body: str, workflow: str, stage: str) -> str`: raises
    `ValueError` for a stage not in the block or with an earlier stage
    unticked; returns *body* unchanged when already ticked.
- Produces, the CLI: `vrg-checkpoint init <ref> --workflow <name> [--tick
  <stage>]…`, `status <ref> --workflow <name> [--json]`, `tick <ref>
  --workflow <name> <stage>`, `reset <ref> --workflow <name>`. `--workflow`
  is required on every subcommand, so a body can carry blocks for several
  workflows. `reset` removes the workflow's block, and the blank line before
  it, leaving the rest of the body as it was; with no block it succeeds and
  writes nothing. `reset` removes only closed blocks, each from its begin
  marker to its end marker: a block with bad lines inside, or two closed
  blocks, are all removed. A begin marker with no end marker is refused
  with exit 6 and nothing is written, since where such a block ends is
  unknown.
- Produces, in `lib/checkpoint.py`: `remove(body: str, workflow: str) ->
  str`, used by `reset`.
- Exit codes: 0 success; 1 failure or refusal (a read or write error, an
  existing well-formed block on `init`, a refused tick); 2 bad arguments;
  3 untrusted body; 4 block missing; 5 skew; 6 malformed, from `status` and
  every write (`init` on a malformed existing block included), with nothing
  written.
- `status --json`: `{"issue": "<owner>/<repo>#<N>", "workflow": str,
  "state": "ok" | "missing" | "skew" | "malformed", "stages": [{"name": str,
  "done": bool}], "next": str | null, "skew": {"recorded": [str],
  "current": [str], "added": [str], "removed": [str], "reordered": bool} |
  null}`. It prints and exits with the state's code; for `missing` and
  `malformed`, `stages` is `[]`.

- [ ] **Step 1: Write the failing library tests** in `test_checkpoint.py`

```python
EC = "epic-create"
STAGES = ("bootstrap", "brainstorm", "spec-pushback", "spec-review", "plan",
          "plan-pushback", "alignment", "file-tasks", "docs-pr")

def test_epic_create_stage_list():
    assert checkpoint.WORKFLOWS[EC] == STAGES

def test_render_then_parse_round_trips():
    block = checkpoint.render(EC, ["bootstrap"])
    assert checkpoint.parse(block, EC) == [(s, s == "bootstrap") for s in STAGES]

def test_status_reads_hand_tick():  # a trusted web-UI tick, uppercase X
    body = checkpoint.render(EC).replace("- [ ] bootstrap", "- [X] bootstrap")
    assert checkpoint.next_stage(checkpoint.parse(body, EC)) == "brainstorm"

def test_annotation_is_malformed():
    body = checkpoint.render(EC).replace("- [ ] plan\n", "- [ ] plan (done by hand)\n")
    with pytest.raises(checkpoint.Malformed):
        checkpoint.parse(body, EC)

def test_stray_line_is_malformed(): ...  # prose, or any blank line inside
#   the markers: between two stages or next to a marker
def test_duplicate_block_is_malformed(): ...  # the block twice in one body
def test_unclosed_block_is_malformed(): ...   # begin marker with no end marker
def test_missing_block_raises_missing(): ...  # body with no block at all
def test_crlf_body_parses_and_ticks(): ...  # CRLF body: parses, ticks, stays CRLF

def test_skew_reports_added_removed_reordered():
    pairs = [("brainstorm", True), ("bootstrap", True), ("old-stage", False)]
    s = checkpoint.skew(pairs, EC)
    assert s.removed == ("old-stage",)
    assert "spec-pushback" in s.added and s.reordered

def test_no_skew_for_current_list():
    assert checkpoint.skew(checkpoint.parse(checkpoint.render(EC), EC), EC) is None

def test_tick_refuses_with_earlier_unticked():
    with pytest.raises(ValueError):
        checkpoint.tick(checkpoint.render(EC), EC, "brainstorm")

def test_tick_already_ticked_is_noop():
    body = checkpoint.render(EC, ["bootstrap"])
    assert checkpoint.tick(body, EC, "bootstrap") == body

def test_write_preserves_rest_of_body():
    pre = "Planning: x/y#1\n\nSecurity advisory: GHSA-aaaa\n\n"
    post = "\n\nhuman text\n"
    body = pre + checkpoint.render(EC) + post
    out = checkpoint.tick(body, EC, "bootstrap")
    assert out.startswith(pre) and out.endswith(post)
```

- [ ] **Step 2: Run to verify they fail**

Run: `vrg-container-run -- pytest tests/vergil_tooling/test_checkpoint.py -v`
Expected: FAIL, `ModuleNotFoundError: vergil_tooling.lib.checkpoint`.

- [ ] **Step 3: Implement `lib/checkpoint.py`** to the Interfaces above.
  The item pattern is `^- \[([ xX])\] ([a-z][a-z0-9-]*)$`, matched against
  every line between the markers, and it skips none: a blank line inside
  the markers makes the block malformed, as `render` writes none. One
  `\r` before a `\n` is part of the line ending, as in `lib/epic_docs.py`
  (GitHub's web editor saves CRLF), and a write uses the line ending the
  body already uses.

- [ ] **Step 4: Run to verify they pass** (the Step 2 command): PASS.

- [ ] **Step 5: Write the failing CLI tests** in `test_vrg_checkpoint.py`,
  with a fake for the trust read and the body write as in
  `test_vrg_epic_docs.py`:
  - `test_init_writes_block_unticked` and `test_init_with_tick_prefix`;
    `test_init_refuses_existing_block` (exit 1, no write);
    `test_init_malformed_existing_block_exits_6` (no write);
    `test_init_tick_not_a_prefix_refused` (`--tick brainstorm` alone, exit
    1);
  - `test_status_json_ok` (the JSON shape above, `next` the first unticked);
    `test_status_missing_exits_4`; `test_status_skew_exits_5` (the `skew`
    object filled); `test_status_malformed_exits_6`;
    `test_status_all_ticked_next_null` (exit 0, `"next": null`);
  - `test_tick_writes_once`; `test_tick_already_ticked_writes_nothing`
    (exit 0, write not called); `test_tick_refused_exits_1`;
    `test_tick_malformed_exits_6_and_writes_nothing`;
  - `test_reset_removes_block_keeps_rest_of_body`;
    `test_reset_without_block_writes_nothing` (exit 0);
    `test_reset_removes_malformed_block`;
    `test_reset_removes_duplicate_blocks`;
    `test_reset_unclosed_block_exits_6_and_writes_nothing`;
    `test_reset_then_status_exits_4`;
  - `test_untrusted_body_exits_3_and_writes_nothing`, for each subcommand;
  - `test_unknown_workflow_exits_2`; `test_runs_under_owner_org` (the read
    and the write under `github.target_org("<owner>")`).

- [ ] **Step 6: Run to verify they fail**

Run: `vrg-container-run -- pytest tests/vergil_tooling/test_vrg_checkpoint.py -v`
Expected: FAIL, `ModuleNotFoundError: vergil_tooling.bin.vrg_checkpoint`.

- [ ] **Step 7: Implement `bin/vrg_checkpoint.py`** (`main(argv: list[str]
  | None = None) -> int` under `@cli_main`, `parse_args`), and add the
  script to `pyproject.toml`. A write is skipped when the new body equals
  the old one.

- [ ] **Step 8: Run to verify they pass** (the Step 6 command): PASS.

- [ ] **Step 8a: Refactor** with the tests green: consolidate the CRLF
  handling with `lib/epic_docs.py`, and the trust read and body write path
  with `bin/vrg_epic_docs.py`, where they duplicate it. Rerun the Step 2 and
  Step 6 commands: PASS.

- [ ] **Step 9: Document it** in `cli-tools-overview.md`: the subcommands,
  `--workflow`, the exit codes, the JSON shape, the strict line rule and the
  trust rule; and add `vrg-checkpoint` to the reference index and the
  permission model's list.

- [ ] **Step 10: Validate and commit**

Run: `vrg-container-run -- vrg-validate`. Expected: pass.
Commit: `vrg-commit --type feat --scope checkpoint --message "add
vrg-checkpoint with init, status and tick"`.

### Task 2b: `vrg-checkpoint reconcile` (vergil-tooling)

Spec §4.4. Blocked by 2a.

**Files:**

- Modify: `src/vergil_tooling/lib/checkpoint.py`,
  `src/vergil_tooling/bin/vrg_checkpoint.py`,
  `docs/site/docs/reference/cli-tools-overview.md`
- Test: `tests/vergil_tooling/test_checkpoint.py`,
  `tests/vergil_tooling/test_vrg_checkpoint.py`

**Interfaces:**

- Consumes: Task 2a's `parse`, `skew`, `render`, `upsert`, `Skew`.
- Produces: `reconcile(body: str, workflow: str, ticks: Iterable[str]) ->
  tuple[str, list[str]]`: the new body, and one line per change made (a
  kept tick, a dropped stage, a new stage ticked or left unticked). A stage
  named in *ticks* that is not new raises `ValueError`.
- Produces, the CLI: `vrg-checkpoint reconcile <ref> --workflow <name>
  [--tick <stage>]…`. It refuses a body with no skew (exit 1, nothing
  written). After the write it posts one comment on the issue whose first
  line is `vrg-checkpoint: reconciled <workflow>` followed by the change
  lines, so the log shows it. The change lines are always printed to
  stdout. If the comment fails after the write, it exits 1 with a message
  saying the block was written, followed by the comment text to post by
  hand.

- [ ] **Step 1: Write the failing tests**
  - `test_reconcile_keeps_matching_ticks`; `test_reconcile_drops_removed_stage`
    (reported); `test_reconcile_new_stage_unticked_by_default`;
    `test_reconcile_new_stage_ticked_when_named`;
    `test_reconcile_tick_of_existing_stage_refused`;
    `test_reconcile_preserves_rest_of_body`;
  - CLI: `test_reconcile_writes_then_comments` (the comment posted after the
    body write, never before); `test_reconcile_comment_failure_exits_1_with_text`;
    `test_reconcile_without_skew_exits_1`;
    `test_reconcile_untrusted_body_exits_3`; `test_reconcile_malformed_exits_6`.
- [ ] **Step 2: Run to verify they fail**

Run: `vrg-container-run -- pytest tests/vergil_tooling/test_checkpoint.py
tests/vergil_tooling/test_vrg_checkpoint.py -v`
Expected: FAIL on the missing `reconcile`.

- [ ] **Step 3: Implement** `reconcile` and the subcommand.
- [ ] **Step 4: Run to verify they pass** (the Step 2 command): PASS.
- [ ] **Step 4a: Refactor** with the tests green: share the change-line
  rendering between the stdout output and the comment body. Rerun the
  Step 2 command: PASS.
- [ ] **Step 5: Document** `reconcile` in `cli-tools-overview.md`.
- [ ] **Step 6: Validate and commit**: `vrg-container-run -- vrg-validate`
  passes; `vrg-commit --type feat --scope checkpoint --message "add
  vrg-checkpoint reconcile"`.

### Task 3a: `epic-create` in stages (vergil-claude-plugin)

Spec §4.3, §4.5 (review rounds), §4.6, §4.7. Blocked by 4a; the first issue
of the plugin batch.

**Files:**

- Modify: `skills/epic-create/SKILL.md`

The skill has no test harness. Its check is the review, and the live run
of Task 5.

- [ ] **Step 1: Add a `## Stages` section** after "Progress comments and
  markers", defining the nine stages with each one's end, evidence and
  re-entry exactly as spec §4.3 states them, the order at a checkpoint
  (work, progress comment, `vrg-checkpoint tick`), and the dirty-worktree
  rule.
- [ ] **Step 2: Rewrite step 0** so it always ends with `vrg-checkpoint
  status <M ref> --workflow epic-create --json`, enters the stage `next`
  names, and posts the "Resuming at `<stage>`" line on a resume. The
  closed-state check of the planning issue, and the `status` it needs, run
  before `vrg-gh issue edit … --add-label in-progress`, so a closed
  planning issue is never labelled. Exit 3
  stops for the human, as an untrusted body does today. Exit 4 with no
  evidence of any stage runs `init` without asking. Exits 4 with evidence,
  5 and 6 are left to Task 3b: until then, the skill stops and asks the
  human. Add the closed-issue rule (finish `docs-pr` only when the
  documentation task is closed as completed; otherwise stop) and the
  all-ticked rule: run `vrg-queue planning` (exit 2 is a complete listing);
  if it shows the item `ready-to-close`, close it as step 10 says;
  otherwise report its note ("handed off, awaiting merge", or the `held:`
  note) and exit, changing nothing.
  Add the abandoned-epic check: after a `status` of 0 (or, once Task 3b
  lands, 5), read the state of the epic the latest trusted `Epic:` marker
  names (`vrg-gh issue view <E> --repo <home> --json state,stateReason`).
  If it is closed as not planned, show the human; once confirmed, finish
  the **abandon the skeleton** step (each part skips what is already done,
  so it is safe to re-enter), which ends with `reset`, then start at
  `bootstrap`.
- [ ] **Step 3: Replace steps 1–3** with the `bootstrap` and `brainstorm`
  stages. `bootstrap` creates the epic (placeholder body and `Planning:`
  line), the documentation task, the `vrg-epic-docs set` section and the
  docs worktree, and nothing else. `brainstorm` ends on the spec commit and
  the one-time **write the epic body** step. Include the shared **abandon
  the skeleton** step (epic closed as not planned first, then the
  documentation task, then the worktree and branch removed, and last
  `vrg-checkpoint reset`, so a revived item starts afresh) and the three
  early exits that use it. The step never removes a branch with commits
  beyond its base: it checks with `vrg-git rev-list --count
  origin/develop..<branch>` before anything else, and on a non-zero count
  closes nothing and stops for the human. The check runs in the `.github`
  main clone (`dirname "$(vrg-git rev-parse --path-format=absolute
  --git-common-dir)"`), and has three outcomes: the branch exists
  locally, so its commits are counted as above; it is absent locally and
  on `origin` (`vrg-git ls-remote --exit-code --heads origin <branch>`
  exits 2), so nothing can be lost and removing the worktree and branch is
  skipped; any other failure stops for the human. Add the security-run rules (a
  neutral skeleton;
  a "No" at the gate keeps it and publishes nothing further).
- [ ] **Step 4: Move the bookends and the operational seeding.** In "The
  epic architecture — bookend tasks" and step 2's text, the
  documentation-review and retrospective tasks are filed in `file-tasks`
  under `Plan task: doc-review` and `Plan task: retrospective`. Operational
  and follow-on tasks are plan tasks. Remove "Seed operational tasks the
  epic will need" from epic creation; "Operational tasks" says that the
  plan lists them.
- [ ] **Step 4a: Align the sections that describe resuming and the step
  order** with `## Stages`, so the skill states one resume model:
  - "Resuming": the stage comes from `status` and the stage list, not from
    the latest round comment; keep its trust, security-run and kept
    scratch-copy rules, and point its dirty-worktree rule at the one in
    `## Stages`;
  - the Entry points table's planning row: resume at the stage `status`
    names, not "the recorded resume point";
  - "Progress comments and markers": the stage-boundary list in the new
    order, with the epic created at `bootstrap`, before the brainstorm;
  - "Epic body sync": its opening says the placeholder body is written at
    `bootstrap` and replaced by the one-time write at the end of
    `brainstorm`;
  - step 0's security check: a marked run resumes at the stage `status`
    names.
- [ ] **Step 5: Review-stage re-entry and the pause.** In "Design review
  rounds", re-entry after a round comment starts at step 7 and dispatches
  nothing unasked. Add the checkpoint pause (spec §4.6), with the two
  commands ready to paste, after each round's comment (step 6) and at each
  stage end. Step 7's question is asked after the pause, whether the human
  continues in the same session or resumes after a clear. Continuing in the
  same session re-enters through `status`.
- [ ] **Step 6: Tick each stage** as the last write of its existing step 4–9
  procedure, unchanged otherwise.
- [ ] **Step 7: Validate and commit**: `vrg-container-run -- vrg-validate`
  passes; `vrg-commit --type feat --scope epic-create --message
  "restructure epic-create into checkpointed stages"`.

### Task 3b: `epic-create` keyed filing, skew and in-flight runs (vergil-claude-plugin)

Spec §4.4, §4.5 (task filing), §4.8. Blocked by 3a; in the plugin batch.

**Files:**

- Modify: `skills/epic-create/SKILL.md`

- [ ] **Step 1: Keyed filing.** Step 8 writes a `Plan task: <id>` line in
  every filed task's body. On re-entry it lists the tasks under the epic
  and its proxy epics with `vrg-epic-audit --epic <epic ref>` (its
  `children`), reads only each one's key with `vrg-issue-read <ref> --json
  | jq -r '.issue.body | split("\n") | map(sub("\r$"; "")) |
  map(select(startswith("Plan task: "))) | first // ""'`, so one line per
  task reaches context, and files only the plan's IDs that are missing. The
  pipeline runs under `set -o pipefail`, so exit 3 or 1 from either command
  stops for the human; an empty line means the task has no key. A task
  created but not linked stops the stage before any pause, to be linked or
  recorded in a progress comment.
- [ ] **Step 2: Skew.** On `status` exit 5 the skill shows the human the
  `skew` object, asks how to proceed (for a new stage before the cursor:
  run it or tick it), and runs `vrg-checkpoint reconcile` with the human's
  `--tick`s. Exit 6 shows the block and stops for the human.
- [ ] **Step 3: In-flight adoption.** On exit 4, the skill infers the
  completed stages from the markers, round comments, commits and filed
  tasks, shows the human its proposal, and runs `init --tick` for each one
  confirmed. An epic closed as not planned, and its documentation task, are
  not evidence: they are an abandoned skeleton, and the revived run starts
  at `bootstrap`. `bootstrap` re-entry ignores any `Epic:` or `Docs task:`
  marker naming an issue closed as not planned, so the revived run creates
  a new epic and a new documentation task and posts both markers, which
  are then the latest. On every entry to `file-tasks`, first or re-entry,
  if Step 1's listing shows any child other than the documentation task
  with an empty key (bookends or operational tasks seeded at epic creation
  by the earlier skill, or tasks filed before an interruption), the skill
  first matches each one to a plan ID (or `doc-review`, `retrospective`),
  shows the matches, and adds the key line to each one confirmed, before
  filing anything.
- [ ] **Step 4: Validate and commit**: `vrg-container-run -- vrg-validate`
  passes; `vrg-commit --type feat --scope epic-create --message "key filed
  tasks, reconcile skew and adopt in-flight runs in epic-create"`.

### Task 3c: `epic-review` abandons a bootstrapped skeleton (vergil-claude-plugin)

Spec §4.3 (the early exits). Blocked by 3a; in the plugin batch.

**Files:**

- Modify: `skills/epic-review/SKILL.md` ("Planning mode", "Drop in every
  mode")

- [ ] **Step 1: Amend the rule "`in-progress` items belong to
  `epic-create`".** An `in-progress` item is still offered only **leave**,
  with one exception: a **skeleton**, an item for which `vrg-checkpoint
  status <ref> --workflow epic-create --json` exits 0 with `"next":
  "brainstorm"` (bootstrap done, no spec committed). For a skeleton,
  planning mode also offers defer and drop. Once the batch is confirmed, it
  runs `epic-create`'s **abandon the skeleton** step (linked by section),
  and only then `vrg-intake-route … defer|drop`. Any other `status`
  result, or any other stage, is not a skeleton: offer only leave.
- [ ] **Step 2: Validate and commit**: `vrg-container-run -- vrg-validate`
  passes; `vrg-commit --type feat --scope epic-review --message "abandon a
  bootstrapped skeleton before deferring or dropping a planning item"`.

### Task 4a: the vergil-tooling release, installed (deployment, .github)

Spec §5. Blocked by 2a, 2b.

- **Precondition (human-attested):** a vergil-tooling release carrying 2a
  and 2b is published, its version and tag named.
- **Procedure:**
  1. `vrg-release-record check <ref>` exits 0 for each of 2a and 2b. For a
     task with no record (exit 4), `vrg-git merge-base --is-ancestor
     <merge-sha> <tag-sha>` in vergil-tooling exits 0 instead.
  2. Confirm the release's `install` stage ran: it installs the release on
     the Mac that ran it and runs `vrg-vm update --all` on every VM.
     **Human-run steps (need a human identity):** on any other host,
     `vrg-tooling-install <version>`; `vrg-vm update --all` only if a VM
     was not reached by the install stage. (`vrg-host` never upgrades
     vergil-tooling.)
  3. `vrg-checkpoint --help` lists `init`, `status`, `tick`, `reset` and
     `reconcile` on the host.
- **SUCCESS:** every check passes, recorded with the version and tag.

### Task 4b: the vergil-claude-plugin release, installed (deployment, .github)

Spec §5. Blocked by 3a, 3b, 3c.

- **Precondition (human-attested):** a vergil-claude-plugin release carrying
  the 3a–3c batch is published, its version and tag named.
- **Procedure:**
  1. `vrg-release-record check <ref>` exits 0 for each of 3a, 3b and 3c,
     with the same fallback as 4a, in vergil-claude-plugin.
  2. **Human-run steps (need a human identity):** `vrg-host plugins` on the
     host; `vrg-vm update --all` for the VMs.
  3. The installed `epic-create` skill contains the `## Stages` section.
- **SUCCESS:** every check passes, recorded with the version and tag.

### Task 5: a checkpointed planning run (validation, .github)

Spec §2 (goals), §5. Blocked by 4b.

- **Precondition (machine probe):** `vrg-checkpoint --help` succeeds, and
  the installed `epic-create` has the `## Stages` section.
- **Procedure:** run the next planning item through `epic-create`,
  preferably vergil-project/.github#1544, clearing with `/clear` and
  resuming with `/vergil:epic-create <ref>` at least:
  1. right after `bootstrap`;
  2. inside `spec-pushback`, after a round comment: re-entry asks the
     round's step 7 question before dispatching anything;
  3. inside `file-tasks`, after some but not all tasks are filed. Before
     the first entry to `file-tasks`, file a documentation-review task and
     a retrospective task under the epic by hand with no `Plan task:` line,
     as the earlier skill seeded them: on entry the skill proposes keying
     them as `doc-review` and `retrospective`, and files neither again;
  4. **adoption:** later in `file-tasks`, after more tasks are filed,
     remove the `Plan task:` line from one or two filed plan tasks, then
     run `vrg-checkpoint reset` on the planning issue and resume: the skill
     proposes the stages and the missing task keys from the evidence, the
     human confirms, the keys are added, and only the missing tasks are
     filed, none twice;
  5. **skew:** before one resume, delete the `docs-pr` line from the block
     by hand: `status` exits 5, the skill shows the difference, and
     `reconcile` restores the line, with its comment posted.
- **The skeleton abandon:** create a throwaway planning item, take it
  through `bootstrap` only, then defer it with `epic-review planning`: the
  epic and its documentation task are closed as not planned (epic first),
  the worktree and branch are gone, the block is removed, and the item is
  in the deferred queue.
- **Checks:** after the run, the epic has exactly one of each task in the
  plan, plus the documentation-review and retrospective tasks, each with
  its `Plan task:` line; the planning issue has one block, with every stage
  ticked; no stage's work was done twice (one epic, one documentation task,
  one spec commit per stage change); every decision recorded before a
  clear is present after it.
- **SUCCESS:** all five resumes and the skeleton abandon observed and
  linked, and every check holds.
