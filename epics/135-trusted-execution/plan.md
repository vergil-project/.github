# Front-Loaded Judgment, Trusted Execution — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Reverse the "Continuous Oversight" behavioral invariant across the vergil skills, and ship `epic-implement` — a thin, resumable driver that shepherds an epic's tasks to their human gates.

**Architecture:** This epic delivers **Markdown skill/doc authoring**, not code. Each task creates or edits a `SKILL.md` (or doc) in `vergil-claude-plugin`, lands one PR, and is verified by `vrg-container-run -- vrg-validate` (the *only* validation command) plus concrete content-acceptance checks. One new skill (`skills/epic-implement/SKILL.md`) plus four doctrine edits (`issue-implement`, `issue-localize`, `issue-validate`+`issue-deploy`, and a canonical statement in `CLAUDE.md`/`README`/`docs/site`). The four doctrine edits are mutually independent — a deliberate first exercise of the frontier-batching `epic-implement` enables.

**Tech Stack:** Markdown + YAML frontmatter (`name`, `description`); `vrg-git`/`vrg-gh`/`vrg-commit`/`vrg-submit-pr` wrappers; `vrg-container-run -- vrg-validate` for validation. No Python, no unit tests — skills are prose contracts, validated by lint + review against acceptance criteria.

## Global Constraints

- **Validation is exactly `vrg-container-run -- vrg-validate`.** Run it from inside the task's worktree. Do not run individual linters.
- **Wrappers only.** All git via `vrg-git`, all GitHub via `vrg-gh`, commits via `vrg-commit --type … --scope … --message …`. Raw `git`/`gh` are denied.
- **PRs are human-gated.** Agents record readiness via `vrg-pr-workflow report-ready`; the **human** runs `vrg-submit-pr`. Never open a PR yourself.
- **No heredocs** for multi-line CLI args — write a temp file and pass `--body-file`/`--file`.
- **Worktree per task.** `vrg-git worktree add -b feature/<N>-<slug> .worktrees/issue-<N>-<slug> origin/develop`, then `cd` in. The main worktree is read-only.
- **Preserve these invariants verbatim in intent during cleanups:** the **never-fabricate / block-not-fabricate** rule (operational skills) and the **never-suppress-a-gate** rule (`issue-implement`). Only the *foreground-for-oversight* framing is removed.
- **Skills are invoked as `/vergil:<name>`.** A skill is a directory `skills/<name>/SKILL.md` with `name` + `description` frontmatter; no `commands/` entry is needed.
- **Repo:** all PRs land in `vergil-project/vergil-claude-plugin`, each task linked under epic `vergil-project/.github#135`.

---

## File Structure

- `skills/epic-implement/SKILL.md` — **Create.** The new driver skill. *(Task A)*
- `skills/issue-implement/SKILL.md` — **Modify** (replace the foreground block, lines ~13–17). *(Task B)*
- `skills/issue-localize/SKILL.md` — **Modify** (replace the foreground block, lines ~22–27). *(Task C)*
- `skills/issue-validate/SKILL.md` — **Modify** (rewrite the foreground block, lines ~28–30, preserving never-fabricate). *(Task D)*
- `skills/issue-deploy/SKILL.md` — **Modify** (same as issue-validate). *(Task D)*
- `CLAUDE.md` — **Modify** (add the canonical doctrine section). *(Task E)*
- `README.md`, `docs/site/docs/agents/index.md`, `docs/site/docs/index.md` — **Modify** (audit + align to the new invariant). *(Task E)*

---

## Dependency / execution order

- **Task A** (`epic-implement` skill) — independent; the substantial task. Can proceed in parallel with the doctrine edits.
- **Tasks B, C, D, E** — mutually independent doctrine edits. Together they form the **first parallel frontier**; a human batches their PRs once.
- No task blocks another. `Blocked-by` links are not used (plain tasks); the plan itself is the ordering authority (per the spec's resume model).
- **Bookends (already created):** docs `#136` (this spec+plan PR), closing brainstorm `#137`, doc-review `vergil-claude-plugin#625`. The closing brainstorm is the **final human gate** — `epic-implement` never runs or closes it.

---

### Task A: The `epic-implement` skill

**Files:**
- Create: `skills/epic-implement/SKILL.md`

**Interfaces:**
- Produces: a `/vergil:epic-implement <epic-ref>` skill. Consumes (routes to) the existing `issue-implement`, `issue-validate`, `issue-deploy` skills and the `CLAUDE.md` "Agent prompt contract."
- Acceptance: `vrg-validate` green; the SKILL.md contains, verbatim in intent — resume-from-epic-issue+plan (not `vrg-epic-audit` as engine); the paste-ready `/rename` line; label→skill routing table; sub-agent dispatch via the CLAUDE.md Agent prompt contract; batch-once-then-stop gate; explicit "driver does NOT drive `pr-watch`"; escalate-on-problem; hybrid terminal handoff (drive doc-review, stop-and-prepare closing brainstorm; never close it).

- [ ] **Step 1: Create the worktree**

```bash
cd /Users/pmoore/dev/projects/vergil-project/vergil-claude-plugin
vrg-git worktree add -b feature/<TASK_A_N>-epic-implement .worktrees/issue-<TASK_A_N>-epic-implement origin/develop
cd .worktrees/issue-<TASK_A_N>-epic-implement
```

- [ ] **Step 2: Write `skills/epic-implement/SKILL.md`**

Write this file exactly (adjust only if `vrg-validate` flags a lint rule):

````markdown
---
name: epic-implement
description: Drive a GitHub epic's tasks to their human gates as the USER agent. Use whenever the human wants to start or resume work on an epic as a whole — "implement epic #135", "pick up epic X where we left off", "drive this epic", "keep working the epic" — with or without the slash command. A thin layer above the issue-* skills: it resumes from the epic issue and its referenced plan, works every currently-runnable task (sub-agents encouraged), batches everything needing the human once, and stops. It never opens PRs, never runs pr-watch, and never runs or closes the closing brainstorm.
---

# Epic implement

Shepherd an epic from wherever it stands to its next human gate. `epic-implement`
is the **epic-level driver** above the `issue-*` skills. It is *stateless by
design*: everything authoritative lives in GitHub — the epic issue, its
sub-issues' open/closed status, and the **aligned plan** the epic references. Any
session context you accumulate is a bonus, never a requirement; if this session
is lost or compacted, re-invoking `/vergil:epic-implement <epic-ref>`
re-derives position and continues.

## Preflight

1. Confirm you are the **USER** agent: `vrg-whoami --mode` must print `user`. If
   not, stop.
2. Resolve the epic ref (`<org>/.github#N`, `#N`, or a URL). Read the epic issue.

## 1. Emit the rename line

Read the epic title and print a paste-ready session rename for the human — the
agent cannot rename its own session:

```text
Suggested session name → run:  /rename epic-<N>-<slug>
```

## 2. Reconstruct state from GitHub, starting at the epic issue

Do **not** rely on any single tool as a state engine. Reconstruct position by:

1. Reading the epic issue and its **referenced plan** (`epics/<N>-<slug>/plan.md`
   in the epic's home repo) — the plan is the **authoritative execution driver**:
   the intended task sequence and the terminal bookends live there.
2. Reading each sub-issue's **open/closed** status.
3. Reconciling the two: the **runnable frontier** is every task the plan says is
   ready whose dependencies (per the plan) are closed, and which is itself open.

`vrg-epic-audit` may be consulted as a **consistency check** (linking, drift) but
is never the source of position. Where the plan's ordering is informal (no
`Blocked-by` links), infer it from the plan text plus issue state.

## 3. Work the runnable frontier

Drive **every** currently-runnable task to its gate. **Use sub-agents wherever
they make this efficient** — independent tasks should run in parallel. Route each
task by its kind label:

| Task kind | Run via |
|---|---|
| code (default `task`) | `issue-implement` |
| `validation` | `issue-validate` |
| `deployment` | `issue-deploy` |

**Dispatch parallel efforts as sub-agents using the "Agent prompt contract" in
`CLAUDE.md`** (the worktree convention). Each sub-agent gets one issue, its
worktree instruction, and runs the routed skill; it reports its
`report-ready` outcome back to you for the batch. Per-issue worktrees
(`.worktrees/issue-<N>-<slug>`) and branches (`feature/<N>-<slug>`) are naturally
unique, so parallel efforts never collide.

## 4. Batch at the gate, then stop

The **gate** is the general boundary where you can no longer proceed without the
human — not merely "a PR is ready." When the runnable frontier is worked,
present a **single consolidated batch** of everything needing the human:

- PRs ready to submit (each recorded via `vrg-pr-workflow report-ready`);
- operational tasks needing a human-gated release;
- any problems you got stuck on (see step 5).

Batch **once** and stop. Your responsibility **ends here.** The human takes the
batch through `vrg-submit-pr` → merge/finalize. **Do not run `pr-watch`** — it is
a rare, human-triggered exception the human invokes only if a gate goes red. When
the human returns and says continue (or re-invokes this skill), re-run steps 2–4:
re-derive state and advance to the next frontier.

## 5. Escalate on problems — don't thrash

The only reason to pull the human in mid-flight is a **problem you cannot
resolve.** Stop and ask, with what you tried and where you're stuck. Never thrash,
never fabricate, never suppress a validation gate.

## 6. Terminal handoff (hybrid)

When only the closing bookends remain:

- **Documentation-review task** — drive it to done like any other task (it is
  mechanical: verify the epic's changes are reflected in the docs, especially
  `docs/site`).
- **Follow-on brainstorm task** — **stop and prepare, never run it.** Assemble
  what you accumulated (what shipped, what went sideways, new problems and
  opportunities) into a seed and hand the human into the closing brainstorm. This
  task is the **final human gate**: it is human-attested and closes only via the
  human-gated docs PR it produces (or a manual human close), which then rolls up
  the epic. You never close it.

## Notes

- You never open a PR, never merge, never cut a release, and never close the
  closing-brainstorm task. Those are human gates.
- `/vergil:handoff` remains the recovery net, but is not required — the epic
  issue and its plan are the durable state.
````

- [ ] **Step 3: Validate**

Run: `vrg-container-run -- vrg-validate`
Expected: PASS (no lint/frontmatter errors on the new skill).

- [ ] **Step 4: Acceptance check**

Confirm the file contains: the `/rename` line; the routing table; "Agent prompt contract"; "Do not run `pr-watch`"; "human-attested"; "never close it". Fix any omission.

- [ ] **Step 5: Commit + report-ready**

```bash
vrg-commit --type feat --scope epic-implement --message "add epic-implement driver skill"
vrg-pr-workflow report-ready --issue <TASK_A_N> \
  --title "feat(epic-implement): add the epic-implement driver skill" \
  --summary "New epic-level driver that resumes from the epic issue + plan, works the runnable frontier with sub-agents, batches human gates, and hands off the closing brainstorm." \
  --notes "First exemplar of the Front-Loaded Judgment, Trusted Execution invariant."
```

Then tell the human: *"Ready — run `vrg-submit-pr`."*

---

### Task B: `issue-implement` cleanup

**Files:**
- Modify: `skills/issue-implement/SKILL.md` (replace the "Run it in the foreground" block)

**Acceptance:** `vrg-validate` green; the "never spawn a sub-agent / visible progress is oversight" text is gone; sub-agents encouraged + escalate-on-problem present; `report-ready` → human-submits and never-suppress-a-gate text elsewhere in the file is untouched.

- [ ] **Step 1: Worktree**

```bash
cd /Users/pmoore/dev/projects/vergil-project/vergil-claude-plugin
vrg-git worktree add -b feature/<TASK_B_N>-issue-implement-doctrine .worktrees/issue-<TASK_B_N>-issue-implement-doctrine origin/develop
cd .worktrees/issue-<TASK_B_N>-issue-implement-doctrine
```

- [ ] **Step 2: Replace the block**

Replace exactly this (lines ~13–17):

```markdown
## Run it in the foreground — be transparent

Do all of this **inline, in the foreground**, narrating as you go: what you are
implementing and why. Never spawn a sub-agent or run the work silently — the
visible progress is the human's oversight.
```

with:

```markdown
## Execute efficiently — trust and escalate

Implement by the most efficient means at your disposal. **Sub-agents are
encouraged** wherever they make the work faster or keep your context clean —
research fan-out, parallel sub-tasks, isolated exploration. You need not narrate
every step for the human to watch: oversight lives in the front-loaded plan and
at the hard gates (PR submit, merge, release), not in a live transcript. This is
the **Front-Loaded Judgment, Trusted Execution** doctrine (see `CLAUDE.md`).

The one thing that pulls the human back mid-flight is a **problem you cannot
resolve**: stop and ask rather than thrash or guess. Never fabricate a result and
never suppress a validation gate.
```

- [ ] **Step 3: Validate** — `vrg-container-run -- vrg-validate` → PASS.
- [ ] **Step 4: Acceptance check** — grep the file: no "never spawn a sub-agent"; "sub-agents are encouraged" present; the later "never suppress a gate" and `report-ready` text still present.
- [ ] **Step 5: Commit + report-ready**

```bash
vrg-commit --type docs --scope issue-implement --message "reverse oversight doctrine — encourage sub-agents, trust and escalate"
vrg-pr-workflow report-ready --issue <TASK_B_N> \
  --title "docs(issue-implement): reverse oversight doctrine" \
  --summary "Replace the foreground/no-sub-agent mandate with Front-Loaded Judgment, Trusted Execution; preserve never-suppress-a-gate." \
  --notes "Part of epic #135."
```

---

### Task C: `issue-localize` cleanup

**Files:**
- Modify: `skills/issue-localize/SKILL.md` (replace the "Run it in the foreground" block)

**Acceptance:** `vrg-validate` green; "never spawn a sub-agent" gone; sub-agents encouraged + escalate present; the never-fabricate-the-reconstructed-metadata intent preserved.

- [ ] **Step 1: Worktree**

```bash
cd /Users/pmoore/dev/projects/vergil-project/vergil-claude-plugin
vrg-git worktree add -b feature/<TASK_C_N>-issue-localize-doctrine .worktrees/issue-<TASK_C_N>-issue-localize-doctrine origin/develop
cd .worktrees/issue-<TASK_C_N>-issue-localize-doctrine
```

- [ ] **Step 2: Replace the block**

Replace exactly this (lines ~22–27):

```markdown
## Run it in the foreground — be transparent

Do all of this **inline, in the foreground**, narrating as you go: which branch
you localized, what validation showed, and the PR metadata you reconstructed.
Never spawn a sub-agent or run it silently — the visible progress is the human's
oversight.
```

with:

```markdown
## Execute efficiently — trust and escalate

Work by the most efficient means available; **sub-agents are encouraged** where
they help. You need not narrate every step — oversight is front-loaded and at the
hard gates, per the **Front-Loaded Judgment, Trusted Execution** doctrine (see
`CLAUDE.md`). If you hit a problem you cannot resolve, stop and ask. Never
fabricate the reconstructed metadata: if you cannot actually localize the branch
or run validation, say so and stop.
```

- [ ] **Step 3: Validate** — `vrg-container-run -- vrg-validate` → PASS.
- [ ] **Step 4: Acceptance check** — no "never spawn a sub-agent"; "sub-agents are encouraged" present; never-fabricate intent preserved.
- [ ] **Step 5: Commit + report-ready**

```bash
vrg-commit --type docs --scope issue-localize --message "reverse oversight doctrine — encourage sub-agents, trust and escalate"
vrg-pr-workflow report-ready --issue <TASK_C_N> \
  --title "docs(issue-localize): reverse oversight doctrine" \
  --summary "Drop the foreground/no-sub-agent mandate; keep never-fabricate on reconstructed metadata." \
  --notes "Part of epic #135."
```

---

### Task D: `issue-validate` + `issue-deploy` cleanup

**Files:**
- Modify: `skills/issue-validate/SKILL.md` (rewrite the foreground block, lines ~28–30)
- Modify: `skills/issue-deploy/SKILL.md` (rewrite the foreground block, lines ~28–30)

**Acceptance:** `vrg-validate` green; foreground-for-oversight framing gone from both; **never-fabricate rule preserved and prominent** in both; sub-agents-allowed + escalate noted.

- [ ] **Step 1: Worktree**

```bash
cd /Users/pmoore/dev/projects/vergil-project/vergil-claude-plugin
vrg-git worktree add -b feature/<TASK_D_N>-operational-doctrine .worktrees/issue-<TASK_D_N>-operational-doctrine origin/develop
cd .worktrees/issue-<TASK_D_N>-operational-doctrine
```

- [ ] **Step 2: Edit `issue-validate`**

Replace exactly this (lines ~28–30):

```markdown
## Run it in the foreground — be transparent

Do everything **inline, in the foreground**, narrating as you go. Never fabricate
or partially fake a result — if you cannot actually run a check, say so and stop.
```

with:

```markdown
## Never fabricate a result

If you cannot actually run a check, say so and stop — **never fabricate or
partially fake a result.** Use whatever means are efficient to run it (sub-agents
encouraged); oversight is front-loaded and at the hard gates, not in a live
transcript (the **Front-Loaded Judgment, Trusted Execution** doctrine; see
`CLAUDE.md`). If you hit a problem you cannot resolve, stop and ask.
```

- [ ] **Step 3: Edit `issue-deploy`**

Replace exactly this (lines ~28–30):

```markdown
## Run it in the foreground — be transparent

Do everything **inline, in the foreground**, narrating as you go. Never fabricate
or partially fake a result — if you cannot actually run a step, say so and stop.
```

with:

```markdown
## Never fabricate a result

If you cannot actually run a step, say so and stop — **never fabricate or
partially fake a result.** Use whatever means are efficient to run it (sub-agents
encouraged); oversight is front-loaded and at the hard gates, not in a live
transcript (the **Front-Loaded Judgment, Trusted Execution** doctrine; see
`CLAUDE.md`). If you hit a problem you cannot resolve, stop and ask.
```

- [ ] **Step 4: Validate** — `vrg-container-run -- vrg-validate` → PASS.
- [ ] **Step 5: Acceptance check** — both files: no "Run it in the foreground"; both retain "never fabricate or partially fake a result".
- [ ] **Step 6: Commit + report-ready**

```bash
vrg-commit --type docs --scope operational-skills --message "reverse oversight doctrine in issue-validate/issue-deploy; keep never-fabricate"
vrg-pr-workflow report-ready --issue <TASK_D_N> \
  --title "docs(operational): reverse oversight doctrine, keep never-fabricate" \
  --summary "Drop foreground-for-oversight framing in issue-validate/issue-deploy; preserve the never-fabricate rule prominently." \
  --notes "Part of epic #135."
```

---

### Task E: Canonical invariant statement

**Files:**
- Modify: `CLAUDE.md` (add a doctrine section)
- Modify: `README.md`, `docs/site/docs/agents/index.md`, `docs/site/docs/index.md` (audit + align any oversight-model prose)

**Acceptance:** `vrg-validate` green; `CLAUDE.md` carries one canonical positive statement of the new invariant; no doc still states the old "tune for oversight / never spawn sub-agents" model as a principle.

- [ ] **Step 1: Worktree**

```bash
cd /Users/pmoore/dev/projects/vergil-project/vergil-claude-plugin
vrg-git worktree add -b feature/<TASK_E_N>-invariant-statement .worktrees/issue-<TASK_E_N>-invariant-statement origin/develop
cd .worktrees/issue-<TASK_E_N>-invariant-statement
```

- [ ] **Step 2: Add the canonical section to `CLAUDE.md`**

Add this section (placement: near the top-level process guidance, e.g. after "Project Overview" or before "Architecture"):

```markdown
## Execution doctrine — Front-Loaded Judgment, Trusted Execution

Human judgment is spent **up front** (brainstorm → pushback → alignment) and at
the **hard gates** (PR submit, merge, release). Between those, agents execute by
the **most efficient means available — sub-agents encouraged** (research
fan-out, parallel implementation). The only mid-flight interrupt is a **problem
the agent cannot resolve**: it stops and asks. We do **not** tune agent behavior
for continuous human observability; that older "Continuous Oversight" invariant
(inline/foreground, narrate everything, never spawn sub-agents) is **retired**.
The never-fabricate and never-suppress-a-gate rules are unaffected.
```

- [ ] **Step 3: Audit `README.md` and the site docs**

Grep each for oversight/foreground/sub-agent language stated as a *principle*:

```bash
grep -nE "foreground|sub-?agent|oversight|transparen|visible progress|never spawn" \
  README.md docs/site/docs/agents/index.md docs/site/docs/index.md
```

For each hit that states the *old invariant as a principle*, rewrite it to the new doctrine (link to the `CLAUDE.md` section). Leave incidental/mechanical uses (e.g. a blocking-call "don't poll" note) untouched. Record what you changed in the PR notes.

- [ ] **Step 4: Validate** — `vrg-container-run -- vrg-validate` → PASS.
- [ ] **Step 5: Acceptance check** — `CLAUDE.md` has the canonical section; the grep in Step 3 shows no remaining *principle-level* statement of the old model.
- [ ] **Step 6: Commit + report-ready**

```bash
vrg-commit --type docs --scope doctrine --message "state Front-Loaded Judgment, Trusted Execution as canonical invariant"
vrg-pr-workflow report-ready --issue <TASK_E_N> \
  --title "docs(doctrine): canonical Front-Loaded Judgment, Trusted Execution statement" \
  --summary "Add the canonical invariant statement to CLAUDE.md and align README/site docs; retire the Continuous Oversight framing." \
  --notes "Part of epic #135."
```

---

## Self-Review (author checklist — completed)

**Spec coverage:** Deliverable 1 → Task A; 2 → Task B; 3 → Task C; 4 → Task D; 5 → Task E. Bookends (#136 docs, #137 closing brainstorm, #625 doc-review) exist. All spec sections mapped.

**Placeholder scan:** The only placeholders are `<TASK_?_N>` issue numbers — filled at task-filing time (epic-create step 9). All content (skill body, replacement prose, canonical statement) is complete and literal.

**Consistency:** The invariant name "Front-Loaded Judgment, Trusted Execution" is used identically across Tasks A–E. Each cleanup preserves its paired load-bearing rule (B: never-suppress-a-gate; C/D: never-fabricate). The `epic-implement` skill's behavior matches the spec (resume from epic issue+plan; no `vrg-epic-audit` engine; batch-once gate; no `pr-watch`; hybrid terminal handoff).

---

## Evolution during execution

*(Frozen at execution start. Append dated entries here for meaningful deviations — a task added, dropped, or rescoped, or a discovered dependency — with the reasoning. The GitHub sub-issues remain the authoritative live task list.)*
