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

1. **`--child-epic` given a standing epic** (ad-hoc, intake or planning) or an
   archive is refused. Nesting a perpetual epic under a finite one would stop
   the finite epic from ever rolling up. Pinned by
   `test_child_epic_refuses_standing_epic_or_archive` (Task 1).
2. **`--child-epic` given a plain task** is refused, since the flag is for
   epics only and `--task` remains the path for tasks. Pinned by
   `test_child_epic_refuses_plain_task` (Task 1).
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
                                    {"epic", "planning"}, {"archive"}])
def test_child_epic_refuses_standing_epic_or_archive(labels) -> None:
    # rc == 1; "is a standing epic or archive" in stderr; add_child not called

def test_child_epic_refuses_plain_task() -> None:
    # non_task_kind -> None; rc == 1; "--child-epic (org/.github#510) is not an epic" in stderr

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
  - **In `_link`, for `--child-epic`:**
    - require `epics.non_task_kind(child) == "epic"`;
    - refuse when `epics.standing_kind_of(epics._labels(child))` is not None,
      or the label set contains `archive`;
    - keep the existing visibility guard unchanged;
    - then call `epics.add_child`.
  - **Docstring:** cite #3591/#2842 and state that `--child-epic` is the one
    sanctioned exception.

- [ ] **Step 4: Run `vrg-container-run -- vrg-validate`.** Expected: PASS,
  with 100% coverage.

- [ ] **Step 5: Confirm that a rollup-driven close fires `epic-rollup.yml`.**
  Read `vergil-actions/.github/workflows/ops-epic-rollup.yml` and confirm the
  close runs under the App token, not `GITHUB_TOKEN`; a close made with
  `GITHUB_TOKEN` would not trigger the workflow for the parent. Record the
  finding in the PR body. If it uses `GITHUB_TOKEN`, stop and report: that
  trips the tripwire.

- [ ] **Step 6: Commit and hand off**
  (`vrg-commit --type feat --scope epic-link`, then `report-ready`).

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

### Task 3: Managed defaults: ignore `paad/security/` and enable paad (vergil-tooling, implementation, B)

**Files:**

- Modify: `src/vergil_tooling/data/gitignore/base`: add `paad/security/`
  after `.superpowers/`.
- Modify: `src/vergil_tooling/data/claude_settings.json`:
  - add an `extraKnownMarketplaces.paad` entry with source
    `{"source": "github", "repo": "Ovid/paad"}`;
  - add `"paad@paad": true` to `enabledPlugins`.
- Test: `tests/vergil_tooling/test_gitignore.py` and
  `tests/vergil_tooling/test_repo_config.py`. Also update any golden or
  fixture expectations those tests pin.

- [ ] **Step 1: Write the failing tests.**
  - `test_base_ignores_paad_security`: the line `paad/security/` is in
    `load_base()`.
  - `test_claude_settings_template_enables_paad`: the template's
    `enabledPlugins["paad@paad"] is True`, and its
    `extraKnownMarketplaces["paad"]["source"]["repo"] == "Ovid/paad"`.
- [ ] **Step 2: Run `vrg-container-run -- vrg-validate`.** Expected: these
  tests FAIL.
- [ ] **Step 3: Make the two data-file edits.** Version pinning: if the
  marketplace source accepts a `ref`, pin it to the paad version Task 4
  chooses. Otherwise leave it unpinned and say so in the PR body.
- [ ] **Step 4: Run `vrg-container-run -- vrg-validate`.** Expected: PASS.
  The repo's own `.gitignore` and `.claude/settings.json` resync through the
  managed fence (run `vrg-gitignore-sync`, and the repo-config sync, as the
  tests require).
- [ ] **Step 5: Commit** (`vrg-commit --type feat --scope managed-defaults`)
  **and hand off.** The change reaches consumer repos through their normal
  managed-file sync after the next vergil-tooling release, which is
  human-gated.

### Task 4: paad usage policy and deviations register (vergil-claude-plugin, implementation, B)

**Files:**

- Create: `docs/paad-policy.md`. This becomes the living home of the
  deviations register, seeded from spec §6. C's doctrine reference document
  will sit beside it.
- Modify: `skills/handoff/SKILL.md` (a one-paragraph scope note) and
  `CLAUDE.md` (a pointer to `docs/paad-policy.md`).

- [ ] **Step 1: Write `docs/paad-policy.md`** with these sections, each
  stating the decision and its reason:
  1. **paad version in use and pinning:** 2.0.2, and whether the
     marketplace `ref` pins it (feeds Task 3, Step 3).
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

`validation` kind. It is blocked by Task 3 and needs the human-gated
vergil-tooling release that ships it.

- **Precondition:** a vergil-tooling release containing Task 3 is installed
  on an agent VM.
- **Procedure:**
  1. Run `vrg-vm update` on a Lima VM and a cloud VM.
  2. In a session on each, confirm that `claude plugin list` shows
     `paad@paad` enabled, at the version `docs/paad-policy.md` names.
  3. Confirm that `paad:agentic-review` and `paad:pushback` load.
- **Acceptance:** both VM types pass. On failure (for example, .github#69's
  third-party marketplace gap), record `Outcome: FAILURE` with the
  evidence. D cannot start until this passes.

### Children (filed at `epic-create` step 9, not implemented here)

Each child is a planning item in the `vergil-project/.github` planning queue
(.github#439). Each body contains the lines
`Umbrella: vergil-project/.github#500` and `Depends on: <ids>`, and the
relevant section of spec §4.

| Child | How it is filed | Depends on |
|---|---|---|
| A: Security-finding handling | new item: `vrg-issue-create --epic planning --repo vergil-project/.github` | none |
| C: Iterating design review | existing .github#469: add the two lines to its body | A, B, G |
| D: Agentic review in issue-implement | vergil-claude-plugin#677: `vrg-intake-route … planning`, then add the two lines | A, B, G |
| E: Steering-file review | new item | B |
| F: Wider paad adoption | new item | C, D |

Here "B" means Tasks 3–5 and "G" means Tasks 1–2. When `epic-create`
promotes a child, it links the new epic under #500 with
`vrg-epic-link --child-epic` (Task 1) and closes the planning item. A child
promoted before Task 1 merges is linked as soon as it does.

### Gate tasks (filed at `epic-create` step 9; they run last)

There are five `validation` tasks in `vergil-project/.github` under #500,
one per child, titled `Child <X> complete: <name>`. They carry **no**
`Blocked-by:` links.
- **Precondition:** the child's epic is closed. For a child that never
  becomes an epic, its planning item has been resolved (closed with a
  recorded outcome).
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
  | B, gitignore and enabling paad | Task 3 |
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
  - enabling paad goes into the managed `claude_settings.json`, so it
    reaches every consumer repo and VM;
  - non-review paad reports are committed under `paad/`, following the
    authors.
