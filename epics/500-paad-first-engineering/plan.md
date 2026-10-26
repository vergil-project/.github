# paad-first AI-assisted engineering (umbrella) — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** deliver the umbrella's own work, **B** (paad 2.0 hygiene and policy),
and set up its children. **A, C, D, E and F** are filed as planning items with
their dependencies, and each is guarded by a gate task so that the umbrella
cannot close before its children finish.

**Architecture:** the umbrella owns only B. A, C, D, E and F are designed in
their own `epic-create` runs, and their internals are out of scope here. B is
split into one task per repo, because a task is one PR in the repo where it
lands. Children are tracked ad hoc until .github#88 provides a mechanism:
- children are linked to the umbrella **by text only** (`Umbrella:` and
  `Depends on:` lines), and child epics stay top-level;
- one `validation` gate task per child holds the umbrella open.

**Tech Stack:** Python 3.12+ data files and pytest in vergil-tooling; Markdown
docs in vergil-claude-plugin; GitHub issues and sub-issues.

**Spec:** `epics/500-paad-first-engineering/spec.md`
(vergil-project/.github#500; planning issue vergil-project/.github#499)

**Shape:** this is an epic-level plan. Each numbered task is **one issue and
one PR in the repo named in its heading**, worked in a worktree of that repo,
with paths relative to that repo's root. Deployment and validation tasks are
operational: they have no code PR and close on `Outcome: SUCCESS`. The child
planning items are not tasks. They are filed during this epic's
`epic-create` step 9 and worked through their own `epic-create` runs.

## Global Constraints

- **The order follows dependencies only.** Anything without a dependency runs
  whenever it is ready.
  - A and B have no dependencies; A is the priority.
  - C, D and E need Task 1 merged and Task 3 merged. D also needs Task 4.
  - F needs C and D.
  - The Task 2 fleet sweep is hygiene that runs in parallel and gates nothing.
    paad 2.0 already writes its own `paad/security/.gitignore` containing `*`,
    so the root-level line is defence in depth, not a prerequisite.
- **paad is unpinned and follows the latest release.** It is enabled through
  the user-level `~/.claude/settings.json`, which agent VMs copy from the
  host. The managed repo `claude_settings.json` is not changed.
- **Spec section 3 binds C and D.** Its rules are not reopened in this epic.
- **Gate tasks carry no `Blocked-by:` links** and run last.
- **Public-repo rule:** security details never appear in issues, comments, PR
  bodies, or report-ready notes, only as counts.
- **Tooling:**
  - validation gate: `vrg-container-run -- vrg-validate`;
  - commits: `vrg-commit`;
  - handoff: `vrg-pr-workflow report-ready`;
  - releases are human-gated, and the agent never cuts one.

## Review Focus

1. **Every language's composed `.gitignore`** includes `paad/security/`, not
   only the base. Pinned by `test_compose_every_language_ignores_paad_security`
   (Task 1).
2. **A repo between the vergil-tooling release and its sync PR merging**
   fails its `local.gitignore` audit, and `vrg-release` preflight blocks it.
   The sweep therefore runs straight after the release, and its acceptance
   checks that field in every repo (Task 2).
3. **A child that turns out to be a single task** (planning item closed with
   `Filed as task <ref>`) must not satisfy its gate while that task is open.
   Pinned by the gate precondition text (Gate tasks section).
4. **A VM whose guest settings predate the host's paad entry** must not
   produce a false failure. Task 4 refreshes the guest settings before
   checking.
5. **The deviations register** must not contradict spec §3.2 (review history
   lives in planning-issue comments). Pinned by Task 3, Step 2.

---

### Task 1: Managed `.gitignore` ignores `paad/security/` (vergil-tooling, implementation, B)

**Files:**

- Modify: `src/vergil_tooling/data/gitignore/base`: add `paad/security/`
  after `.superpowers/`.
- Test: `tests/vergil_tooling/test_gitignore.py`. Also update any golden or
  fixture expectations it pins.

The managed `claude_settings.json` is **not** changed. Enabling paad there
would put every repo's checked-in `.claude/settings.json` out of compliance
with the config audit (`repo_config.py:383-389`: nightly ops, and the
`vrg-release` preflight). It would also not reach agent VMs, which install
plugins from the user-level settings copied from the host
(`vm_guest.py:528-548`).

- [ ] **Step 1: Write the failing tests.**
  - `test_base_ignores_paad_security`: `"paad/security/" in load_base()`.
  - `test_compose_every_language_ignores_paad_security`: for every `lang` in
    `FRAGMENT_LANGS`, and for `None`, `"paad/security/" in compose(lang)`.
- [ ] **Step 2: Run `vrg-container-run -- vrg-validate`.** Expected: both
  tests FAIL.
- [ ] **Step 3: Make the data-file edit,** then resync this repo's own fence
  with `vrg-gitignore-sync`.
- [ ] **Step 4: Run `vrg-container-run -- vrg-validate`.** Expected: PASS.
- [ ] **Step 5: Commit** (`vrg-commit --type feat --scope gitignore`) **and
  hand off.**

### Task 2: Fleet sweep of the managed `.gitignore` (vergil-tooling, deployment, B)

`deployment` kind. It is blocked by Task 1, and it needs the human-gated
vergil-tooling release that ships Task 1. Run it **straight after that
release**: until each repo's sync PR merges, that repo fails its
`local.gitignore` audit, and its `vrg-release` preflight is blocked.

- **Precondition:** a vergil-tooling release containing Task 1 is installed
  on the host.
- **Procedure:**
  1. Agent: run
     `vrg-fleet-sync --target gitignore --dry-run --repos <clones>`, then
     run it again without `--dry-run`.
     - `<clones>` is every active repo's local clone, at
       `~/dev/projects/<org>/<repo>` (including `.github`), for the orgs
       vergil-project, logical-minds-foundry and mq-rest-admin-project.
     - The sweep opens one sync PR per drifted repo. Each is tracked under
       that repo's ad-hoc epic, which is the tool's default.
  2. **Human:** submit and merge each sync PR (the sweep stops at
     report-ready).
- **Acceptance:** for every active repo, the `local.gitignore` field of
  `vrg-github-repo-config audit` passes. Unrelated audit fields are out of
  scope.

### Task 3: paad usage policy and deviations register (vergil-claude-plugin, implementation, B)

**Files:**

- Create: `docs/paad-policy.md`. This becomes the living home of the
  deviations register, seeded from spec §6. C's doctrine reference document
  will sit beside it.
- Modify: `skills/handoff/SKILL.md` (a one-paragraph scope note) and
  `CLAUDE.md` (a pointer to `docs/paad-policy.md`).

- [ ] **Step 1: Write `docs/paad-policy.md`** with these sections, each
  stating the decision and its reason:
  1. **paad version:**
     - unpinned, following the latest release;
     - enabled in the user-level `~/.claude/settings.json`, which agent VMs
       copy from the host;
     - `vrg-vm update` moves every enabled plugin to its latest version
       (`vm_guest.py:463-500`), so a pin would not hold where agents run;
     - every `paad/config/` file names the paad version it was written for,
       and D detects drift.
  2. **Report files:**
     - design-review reports (pushback, alignment) are not written, because
       the planning-issue comments are the record (spec §3.2);
     - agentic-review output is routed by D (spec §3.3);
     - all other paad reports (architecture, test-roadmap and so on) are
       committed under `paad/`, as the authors recommend.
  3. **`paad/security/`:** never committed, and ignored by both paad's own
     `.gitignore` and the managed one (Task 1). Details are handled by child
     A's mechanism.
  4. **`paad/config/`:** adopted only where a child epic needs it (D for
     agentic-review, C as the fallback for review-only mode).
  5. **`paad:handoff` and `vergil:handoff`:**
     - `vergil:handoff` remains the workflow's session-continuity skill,
       because it writes into memory under our policy;
     - `paad:handoff` is not used in vergil workflows;
     - always use the prefixed name.
  6. **The deviations register:** the table from spec §6, plus the entries
     from items 2 and 5.
- [ ] **Step 2: Check the register against spec §3.2.** No row may claim that
  review history lives inside a spec or plan.
- [ ] **Step 3: Add the handoff scope note.** Add one paragraph to
  `skills/handoff/SKILL.md` saying that `paad:handoff` is a separate skill,
  that it is not used in vergil workflows, and pointing to
  `docs/paad-policy.md`.
- [ ] **Step 4: Run `vrg-container-run -- vrg-validate`, then commit**
  (`vrg-commit --type docs --scope paad`) and hand off.

### Task 4: Validation: paad is available where agents run (vergil-tooling, validation, B)

`validation` kind. It is blocked by Task 3 and does not depend on any
release.

- **Precondition:** Task 3 is merged.
- **Procedure:**
  1. Confirm that the host's `~/.claude/settings.json` declares the `paad`
     marketplace (`Ovid/paad`) and `"paad@paad": true`.
  2. For a Lima VM and a cloud VM:
     1. Run `vrg-vm restart`, which copies the host's settings into the guest.
     2. Run `vrg-vm update`, which reconciles plugins from the guest settings.
     3. In a session, check that `claude plugin list` shows `paad@paad`
        enabled, and record its version.
     4. Check that `paad:agentic-review` and `paad:pushback` load.
- **Acceptance:** both VM types have paad enabled and loadable, and their
  versions are recorded in the outcome comment. On failure, record
  `Outcome: FAILURE` with the evidence. D cannot start until this passes.

### Children (filed at `epic-create` step 9, not implemented here)

Each child is a planning item in the `vergil-project/.github` planning queue
(.github#439). Each body contains:
- the lines `Umbrella: vergil-project/.github#500` and `Depends on: <ids>`;
- the relevant section of spec §4;
- this instruction to whoever promotes it: **"When `epic-create` creates this
  child's epic, put `Umbrella: vergil-project/.github#500` in the epic's body,
  and close this planning item with `Planned as <epic ref>`."**

Children are filed **before** the gate tasks, because each gate names its
child's ref.

| Child | How it is filed | Depends on |
|---|---|---|
| A: Security-finding handling | new item: `vrg-issue-create --epic planning --repo vergil-project/.github` | none |
| C: Iterating design review | existing .github#469: add the lines to its body | A; Tasks 1 and 3 |
| D: Agentic review in issue-implement | vergil-claude-plugin#677: `vrg-intake-route vergil-project/vergil-claude-plugin#677 planning` (reparents it in place), then add the lines | A; Tasks 1, 3 and 4 |
| E: Steering-file and Claude settings review | new item | Tasks 1 and 3 |
| F: Wider paad adoption | new item | C and D |

### Gate tasks (filed at `epic-create` step 9; they run last)

There are five `validation` tasks in `vergil-project/.github` under #500, one
per child, titled `Child <X> complete: <name>`. They carry **no**
`Blocked-by:` links, and each body carries `Child: <planning item ref>`.
- **Precondition (a machine probe):**
  1. The child's planning item is closed.
  2. Every issue that its closing comment names, whether
     `Planned as <epic ref>` or `Filed as task <ref>`, is also closed.
  3. If the closing comment names no issue, it records the outcome (for
     example, "not planned" with a reason).
- **Acceptance:** `Outcome: SUCCESS`, naming what was checked.

`epic-implement` runs a gate task only after its child's work has closed.
Before that, the gate counts as waiting, not runnable.

### Bookends (already filed)

- **Documentation:** .github#501, which publishes this spec and plan.
- **Documentation review:** vergil-tooling#3658. It covers the umbrella's
  own changes (Tasks 1–4) and runs once they close; it does not wait for the
  children, because each child epic has its own documentation review.
- **Retrospective:** .github#502. It is terminal and runs after all five gate
  tasks close.

## Self-Review

- **Spec coverage:**

  | Spec item | Covered by |
  |---|---|
  | B, gitignore | Tasks 1 and 2 |
  | B, report-file policy, handoff overlap, `paad/config` and version | Task 3 |
  | B, availability on VMs | Task 4 |
  | Child filing | the Children section |
  | Gate tasks | the Gate tasks section |
  | Bookends | the Bookends section |
  | Order | Global Constraints and the children table |

  Spec §3 is implemented by C and D, not here.
- **Calls made in this plan:**
  - the deviations register's living home is
    `vergil-claude-plugin/docs/paad-policy.md`;
  - `vergil:handoff` is kept and `paad:handoff` is not used;
  - paad reports other than design reviews are committed under `paad/`,
    following the authors;
  - the documentation review covers only the umbrella's own tasks.
