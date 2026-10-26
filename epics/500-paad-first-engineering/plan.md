# paad-first AI-assisted engineering (umbrella) — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Stand up the umbrella epic's own deliverables and prepare its
children:
- **B:** paad 2.0 hygiene and policy.
- **G:** minimal support for nesting an epic under an epic.
- **A, C, D, E and F:** filed as planning items with their dependencies, each
  guarded by a gate task so the umbrella cannot close before its children
  finish.

**Architecture:** The umbrella owns only B and G. A, C, D, E and F are
designed in their own `epic-create` runs, and their internals are out of scope
here. B and G are split into one task per repo, because a task is one PR in
the repo where it lands. Children are tracked ad hoc until .github#88 provides
a real mechanism:
- the spec's dependency table, and each child's `Umbrella:` / `Depends on:`
  lines, hold the order;
- one `validation` gate task per child holds the umbrella open.

**Tech Stack:** Python 3.12+ (argparse, pytest) in vergil-tooling; Markdown
skills and docs in vergil-claude-plugin; GitHub issues and sub-issues.

**Spec:** `epics/500-paad-first-engineering/spec.md`
(vergil-project/.github#500; planning issue vergil-project/.github#499)

**Shape:** this is an epic-level plan. Each numbered task is **one issue and
one PR in the repo named in its heading**, worked in a worktree of that repo,
with paths relative to that repo's root. Validation tasks are operational:
they have no code PR, run through `issue-validate`, and close on
`Outcome: SUCCESS`. The child planning items are not tasks. They are filed
during this epic's `epic-create` step 9 and worked through their own
`epic-create` runs.

## Global Constraints

- **The order follows dependencies only:**
  - A, B and G have no dependencies and may run in parallel; A is the
    priority.
  - C and D depend on A, B and G.
  - E depends on B.
  - F depends on C and D.
- **G stays small** (its scope tripwire). It consists of the
  `vrg-epic-link --child-epic` option, `epic-implement`'s stop rule, and the
  rollup and audit checks. Stop and return to the human if G needs any of:
  - rollup semantics changed;
  - `vrg-epic-audit` invariants reworked;
  - recursion in `epic-implement`;
  - anything resembling .github#88's general mechanism.
- **`epic-implement` never recurses into a child epic.**
- **Section 3 of the spec binds C and D.** Its rules are not reopened in this
  epic.
- **Gate tasks carry no `Blocked-by:` links** and run last.
- **Public-repo rule:** security details never appear in issues, comments, PR
  bodies, or report-ready notes, only as counts.
- **Tooling:**
  - validation gate: `vrg-container-run -- vrg-validate`;
  - commits: `vrg-commit`;
  - handoff: `vrg-pr-workflow report-ready`;
  - releases are human-gated, and the agent never cuts one.

## Review Focus

1. **`--child-epic` given a standing epic** (ad-hoc, intake or planning) is
   refused. Nesting a perpetual epic under a finite one would stop
   the finite epic from ever rolling up. Pinned by
   `test_child_epic_refuses_standing_epic` (Task 1).
2. **`--child-epic` given a plain task or an archive** is refused, since the
   flag is for epics only and `--task` remains the path for tasks. Pinned
   by `test_child_epic_refuses_non_epic` (Task 1).
3. **A child epic closing** rolls up its finite parent epic once all of the
   parent's other children are closed, and leaves the parent open while any
   remain. Pinned by `test_rollup_cascades_from_child_epic` and
   `test_rollup_child_epic_leaves_parent_open_while_siblings_open` (Task 1).
4. **A public child epic under a less visible parent** is refused, keeping
   the visibility guard. Pinned by `test_child_epic_keeps_visibility_guard`
   (Task 1).
5. **`epic-implement` meeting a child epic** in its runnable frontier stops
   and names the child, and does not dispatch it to `issue-implement` or
   recurse. Pinned by the scenario check in Task 2, Step 3.

---

### Task 1: `vrg-epic-link --child-epic`, plus rollup and audit checks for nested epics (vergil-tooling, implementation, G)

**Files:**

- Modify: `src/vergil_tooling/bin/vrg_epic_link.py` (`parse_args`, `main`,
  `_link`, module docstring)
- Test: `tests/vergil_tooling/test_vrg_epic_link.py`,
  `tests/vergil_tooling/test_epics.py` (rollup), and
  `tests/vergil_tooling/test_epic_audit.py`

**Interfaces:**

- Produces the CLI contract
  `vrg-epic-link --epic <ref> --child-epic <ref>`. It is mutually exclusive
  with `--task`, and exactly one of the two is required. Each child's
  `epic-create` run uses it to link its epic under #500.

- [ ] **Step 1: Write the failing tests** (follow the file's patterns:
  `patch(f"{_MOD}...")` and the autouse `_task_is_a_task` fixture)

```python
def test_child_epic_links_epic_under_epic() -> None:
    # non_task_kind -> "epic"; standing_kind_of(labels) -> None; is_public -> True
    rc = main(["--epic", "org/.github#500", "--child-epic", "org/.github#510"])
    assert rc == 0
    mock_add.assert_called_once_with(
        epics.IssueRef("org", ".github", 500), epics.IssueRef("org", ".github", 510)
    )

@pytest.mark.parametrize("labels", [{"epic", "ad-hoc"}, {"epic", "intake"},
                                    {"epic", "planning"}])
def test_child_epic_refuses_standing_epic(labels) -> None:
    # mocked label read -> labels; rc == 1; "is a standing epic" in stderr;
    # add_child not called

@pytest.mark.parametrize("labels", [set(), {"archive"}])
def test_child_epic_refuses_non_epic(labels) -> None:
    # a plain task or an archive (which carries no `epic` label);
    # rc == 1; "--child-epic (org/.github#510) is not an epic" in stderr

def test_child_epic_keeps_visibility_guard() -> None:
    # child public, parent home private -> rc == 1; add_child not called

def test_task_and_child_epic_are_mutually_exclusive() -> None:
    with pytest.raises(SystemExit):
        parse_args(["--epic", "o/.github#1", "--task", "#2", "--child-epic", "#3"])
```

In `test_epics.py`:

```python
def test_rollup_cascades_from_child_epic() -> None:
    # parent_of(child_epic) -> umbrella (labels {"epic"}); all_children_closed -> True
    rollup(child_epic)
    # asserts github.run("issue", "close", "500", "--repo", "org/.github")

def test_rollup_child_epic_leaves_parent_open_while_siblings_open() -> None:
    # all_children_closed -> False; github.run not called
```

In `test_epic_audit.py`: `test_epic_with_finite_epic_parent_is_not_drift`. An
open epic whose native parent is a finite epic yields no drift or orphan
finding.

- [ ] **Step 2: Run them**

  Run `vrg-container-run -- vrg-validate`. Expected: the new link tests
  FAIL. The rollup and audit tests may already PASS. If so, they are the G
  verification checks, so keep them as regression pins. If the rollup or
  audit tests FAIL, stop: fixing them means changing rollup or audit
  semantics, which trips G's tripwire, so report to the human.

- [ ] **Step 3: Implement `--child-epic` in `vrg_epic_link.py`**

  - **Arguments:** `parse_args` uses a required mutually exclusive group,
    `--task` / `--child-epic`.
  - **In `_link`, for `--child-epic`:** read the child's labels once.
    Expose a public `epics.labels_of(ref)` wrapper rather than calling the
    private `_labels` from a bin module. Then:
    - require `epics.non_task_kind_of_labels(labels) == "epic"` (an archive
      carries no `epic` label, so it is refused here);
    - refuse when `epics.standing_kind_of(labels)` is not None;
    - keep the existing visibility guard unchanged;
    - then call `epics.add_child`.
  - **Docstring:** cite #3591/#2842 and state that `--child-epic` is the one
    sanctioned exception.

- [ ] **Step 4: Run `vrg-container-run -- vrg-validate`.** Expected: PASS,
  with 100% coverage.

- [ ] **Step 5: Commit and hand off**
  (`vrg-commit --type feat --scope epic-link`, then `report-ready`). The
  report-ready notes record two facts already established, which are why the
  cascade fires end to end:
  - the reusable `vergil-actions/.github/workflows/ops-epic-rollup.yml`
    closes epics with an App token, not `GITHUB_TOKEN` (lines 35-55), so the
    close of a child epic triggers `.github`'s `epic-rollup.yml` caller
    (`on: issues: closed`) for the parent;
  - `vrg-epic-rollup` accepts an epic as its `--task`
    (`bin/vrg_epic_rollup.py:25-43`).

  The notes also list any child already promoted to an epic, so the human can
  link it with `--child-epic` once this merges.

### Task 2: `epic-implement` stops at a child epic (vergil-claude-plugin, implementation, G)

**Files:**

- Modify: `skills/epic-implement/SKILL.md`. In §3 "Work the runnable
  frontier", add a row and a rule; in §4 "Batch at the gate", add the gate
  item.

- [ ] **Step 1: Add the routing row.**
  `| epic (a child epic) | **none** — stop; see below |`
- [ ] **Step 2: Add the rule** below the routing table:

  > If a runnable sub-issue carries the `epic` label, do not dispatch it and
  > do not recurse. List it at the gate as "child epic `<ref>` needs its own
  > `epic-implement` run", and continue with the other runnable tasks. A
  > child epic is the human's to drive.

  Also add "child epics awaiting their own run" to §4's list of items
  needing the human.
- [ ] **Step 3: Scenario check (Review Focus 5).** Read the edited skill as
  if driving umbrella #500 with frontier {B task, child epic A}. Confirm the
  text directs dispatching the B task and listing A at the gate, and
  contains no path that sends A to `issue-implement` or recursively to
  `epic-implement`. Record the walkthrough in the PR body.
- [ ] **Step 4: Run `vrg-container-run -- vrg-validate`, then commit**
  (`vrg-commit --type feat --scope epic-implement`) and hand off.

### Task 3: Managed `.gitignore` ignores `paad/security/` (vergil-tooling, implementation, B)

**Files:**

- Modify: `src/vergil_tooling/data/gitignore/base`: add `paad/security/`
  after `.superpowers/`.
- Test: `tests/vergil_tooling/test_gitignore.py`. Also update any golden or
  fixture expectations it pins.

The managed `claude_settings.json` is **not** changed. Enabling paad there
would put every repo's checked-in `.claude/settings.json` out of
compliance with the config audit (`repo_config.py:386-390`; nightly ops,
and `vrg-release` preflight). It would also not reach agent VMs, which
install plugins from the user-level `~/.claude/settings.json`
(`vm_guest.py:378-395`).

- [ ] **Step 1: Write the failing test.** `test_base_ignores_paad_security`:
  the line `paad/security/` is in `load_base()`.
- [ ] **Step 2: Run `vrg-container-run -- vrg-validate`.** Expected: the test
  FAILS.
- [ ] **Step 3: Make the data-file edit,** then resync this repo's own fence
  with `vrg-gitignore-sync`.
- [ ] **Step 4: Run `vrg-container-run -- vrg-validate`.** Expected: PASS.
- [ ] **Step 5: Commit** (`vrg-commit --type feat --scope gitignore`) **and
  hand off.**

### Task 3a: Fleet sweep of the managed `.gitignore` (vergil-tooling, deployment, B)

`deployment` kind. It is blocked by Task 3, and it needs the human-gated
vergil-tooling release that ships Task 3.

- **Precondition:** a vergil-tooling release containing Task 3 is installed.
- **Procedure:** run `vrg-fleet-sync --target gitignore` across the active
  fleet (vergil-project, logical-minds-foundry, mq-rest-admin-project). This
  opens one sync PR per repo, as the managed-block sweeps have done before.
- **Acceptance:** every active repo's managed fence includes
  `paad/security/`, and `vrg-github-repo-config audit` passes fleet-wide.

### Task 4: paad usage policy and deviations register (vergil-claude-plugin, implementation, B)

**Files:**

- Create: `docs/paad-policy.md`. This becomes the living home of the
  deviations register, seeded from spec §6. C's doctrine reference document
  will sit beside it.
- Modify: `skills/handoff/SKILL.md` (a one-paragraph scope note) and
  `CLAUDE.md` (a pointer to `docs/paad-policy.md`).

- [ ] **Step 1: Write `docs/paad-policy.md`** with these sections, each
  stating the decision and its reason:
  1. **paad version in use and where it is configured:** 2.0.2.
     - paad is enabled in the human's user-level `~/.claude/settings.json`,
       which agent VMs copy from the host (`vm_guest.py:378-395`).
     - `vrg-vm update` moves every enabled plugin to its latest version
       (`vm_guest.py:463-500`). Record either "unpinned, follows latest" or
       a user-settings marketplace `ref`, whichever the settings-scope
       review settles.
  2. **Report files:**
     - design-review reports (pushback, alignment) are not written, because
       the planning-issue comments are the record (spec §3.2);
     - agentic-review output is routed by D (spec §3.3);
     - all other paad reports (architecture, test-roadmap and so on) are
       committed under `paad/`, as the authors recommend.
  3. **`paad/security/`:** never committed, and ignored by the managed
     `.gitignore` (Task 3). Details are handled by child A's mechanism.
  4. **`paad/config/`:** adopted only where a child epic needs it (D for
     agentic-review, C as the fallback for review-only mode). Each config
     file names the paad version it was written for.
  5. **`paad:handoff` and `vergil:handoff`:**
     - `vergil:handoff` remains the workflow's session-continuity skill,
       because it writes into memory under our policy;
     - `paad:handoff` is not used in vergil workflows;
     - always use the prefixed name.

     Record this as a deviation.
  6. **The deviations register:** the table from spec §6, plus the entries
     from items 2 and 5.
- [ ] **Step 2: Add the handoff scope note.** Add one paragraph to
  `skills/handoff/SKILL.md` saying that `paad:handoff` is a separate skill,
  that it is not used in vergil workflows, and pointing to
  `docs/paad-policy.md`.
- [ ] **Step 3: Run `vrg-container-run -- vrg-validate`, then commit**
  (`vrg-commit --type docs --scope paad`) and hand off.

### Task 5: Validation: paad is available where agents run (vergil-tooling, validation, B)

`validation` kind. It is blocked by Task 4, because it checks the version
policy that Task 4 records. It does not depend on any release.

- **Precondition:** Task 4 is merged.
- **Procedure:**
  1. Confirm that the host's user-level `~/.claude/settings.json` declares the
     `paad` marketplace (`Ovid/paad`) and `"paad@paad": true`.
  2. Run `vrg-vm update` on a Lima VM and a cloud VM.
  3. In a session on each, confirm that `claude plugin list` shows
     `paad@paad` enabled, at the version `docs/paad-policy.md` names.
  4. Confirm that `paad:agentic-review` and `paad:pushback` load.
- **Acceptance:** both VM types pass. On failure, record `Outcome: FAILURE`
  with the evidence. D cannot start until this passes.

### Children (filed at `epic-create` step 9, not implemented here)

Each child is a planning item in the `vergil-project/.github` planning queue
(.github#439). Each body contains:
- the lines `Umbrella: vergil-project/.github#500` and `Depends on: <ids>`;
- the relevant section of spec §4;
- this instruction to whoever promotes it: **"When `epic-create` creates
  this child's epic, link it under the umbrella with
  `vrg-epic-link --epic vergil-project/.github#500 --child-epic <new epic>`
  before closing this planning item."**

Children are filed **before** the gate tasks, because each gate names its
child's ref.

| Child | How it is filed | Depends on |
|---|---|---|
| A: Security-finding handling | new item: `vrg-issue-create --epic planning --repo vergil-project/.github` | none |
| C: Iterating design review | existing .github#469: add the two lines to its body | A, B, G |
| D: Agentic review in issue-implement | vergil-claude-plugin#677: `vrg-intake-route … planning`, then add the two lines | A, B, G |
| E: Steering-file and Claude settings review | new item | B |
| F: Wider paad adoption | new item | C, D |

Here "B" means Tasks 3, 3a, 4 and 5 and "G" means Tasks 1–2. When `epic-create`
promotes a child, its planning item's instruction links the new epic under
#500 with `vrg-epic-link --child-epic` (Task 1) and closes the planning item.
A child promoted before Task 1 merges is listed in Task 1's report-ready
notes and linked by the human once Task 1 lands.

### Gate tasks (filed at `epic-create` step 9; they run last)

There are five `validation` tasks in `vergil-project/.github` under #500,
one per child, titled `Child <X> complete: <name>`. They carry **no**
`Blocked-by:` links. Each body carries `Child: <planning item ref>`.
- **Precondition (a machine probe):**
  1. The child's planning item is closed.
  2. If its closing comment says `Planned as <epic ref>`, that epic is
     closed.
  3. If the planning item was closed without becoming an epic, the closing
     comment records the outcome.
- **Acceptance:** `Outcome: SUCCESS`, naming the closed epic or the resolved
  item.

`epic-implement` runs a gate task only after its child has closed. Before
that, the gate counts as waiting, not runnable.

### Bookends (already filed)

- Documentation: .github#501, which publishes this spec and plan.
- Documentation review: vergil-tooling#3658. It runs after every task and
  gate except the retrospective.
- Retrospective: .github#502. It is terminal and runs after all five gate
  tasks close.

## Self-Review

- **Spec coverage:**

  | Spec item | Covered by |
  |---|---|
  | B, gitignore | Tasks 3 and 3a |
  | B, report-file policy, handoff overlap and `paad/config` | Task 4 |
  | B, availability on VMs | Task 5 |
  | G, link option and audit and rollup checks | Task 1 |
  | G, stop rule | Task 2 |
  | Child filing | the Children section |
  | Gate tasks | the Gate tasks section |
  | Bookends | the Bookends section |
  | Order | Global Constraints and the children table |

  Spec §3 is implemented by C and D, not here.
- **Calls made in this plan:**
  - the deviations register's living home is
    `vergil-claude-plugin/docs/paad-policy.md`;
  - `vergil:handoff` is kept and `paad:handoff` is not used;
  - paad is enabled through user-level settings, not the managed repo
    settings;
  - non-review paad reports are committed under `paad/`, following the
    authors.
