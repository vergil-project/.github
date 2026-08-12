# Retrospective bookend — Implementation Plan

> **For agentic workers:** tasks are GitHub issues under epic
> vergil-project/.github#201, each 1:1 with a finalizing PR in the repo named.
> Steps use checkbox (`- [ ]`) syntax. Drive with `epic-implement`; run each code
> task via `issue-implement`.

**Goal:** add a mandatory, single, terminal **retrospective** to the epic
lifecycle — a new `epic-retrospective` skill plus the reframing of `epic-create`
and `epic-implement` around it — so every epic ends with an honest,
backward-looking record that closes it.

**Architecture:** one new skill defines the concept and owns enforcement (a
preflight definition-of-done gate); its two sibling skills and the `CLAUDE.md`
on-ramp are edited to route the closing bookends through it. All changes are
**plugin-contained** (`vergil-claude-plugin`). The change is dogfooded on this
very epic: task `#203` is authored with the new skill.

**Tech stack:** Claude-plugin skills (`skills/<name>/SKILL.md`, YAML frontmatter +
Markdown); `vrg-epic-audit` / `vrg-gh` for epic-state queries; `vrg-validate` for
validation.

## Global constraints

- Validation: `vrg-container-run -- vrg-validate` (the only validation command).
  Git via `vrg-git` / `vrg-commit`; PRs via `vrg-submit-pr` (human-gated).
- **Plugin-contained.** No changes outside `vergil-claude-plugin` are expected;
  the `.github#40` convention is edited only if the doc-review sweep (#658) finds
  a real gap.
- **Terminality is enforced only in the `epic-retrospective` skill preflight** —
  never via static `Blocked-by` reflinks (they do not scale against a dynamic
  epic).
- **Dogfooding order:** the `epic-retrospective` skill must be merged to
  `develop` **and the plugin released + reloaded** before task `#203` can invoke
  it. Until release, `/vergil:epic-retrospective` is not on the Skill list (the
  reload dance). This gates `#203`, not the earlier tasks.

---

### Task 1 — the `epic-retrospective` skill (vergil-claude-plugin)

**Files:**

- Create: `skills/epic-retrospective/SKILL.md`

The foundational deliverable. Authoring a single self-contained skill.

- [ ] **Frontmatter.** `name: epic-retrospective`; `description:` covering
  triggers — "finish an epic", "run the retrospective", "close out epic `<ref>`",
  "we're done with this epic" — and stating it is the terminal, run-once finishing
  gate of the three-skill lifecycle (`epic-create` → `epic-implement` →
  `epic-retrospective`).
- [ ] **Overview + Preflight (definition-of-done gate).** Confirm USER agent
  (`vrg-whoami --mode` = `user`). Resolve the epic ref; enumerate every
  sub-issue via `vrg-epic-audit`. **Abort** (refuse-and-stop, never fabricate) if
  any child *other than the retrospective task* is still open, printing the open
  refs: "not the terminal task; N still open: #…". This is the sole terminality
  enforcement; the retrospective is implicitly blocked by all other children,
  resolved dynamically at run time.
- [ ] **The artifact.** Specify `retrospective.md` at
  `epics/<N>-<slug>/retrospective.md` with the section set from spec §3: §0 At a
  glance (preamble, ≤ ½ page — PR table with one-liners, repos touched,
  task/PR counts, releases, opened→closed span), §1 How the plan evolved
  (synthesized from `plan.md` Evolution log), §2 Lessons learned, §3 Compromises
  & tradeoffs, §4 New problems & opportunities (+ where each went), §5 What's
  next, Appendix A Operational notes (optional — note that a `summarize`
  operations-mode log may be embedded here as a subset for operationally-heavy
  epics), Appendix B Extended metrics (optional).
- [ ] **§0 data sourcing.** §0 facts come from `vrg-epic-audit` (task/PR graph)
  plus `vrg-gh` (PR titles, merge dates, releases). Anything not determinable is
  marked *unknown* — never invented (no-fabrication doctrine).
- [ ] **Authoring model.** Mirror `writing-plans`, not `alignment`: the agent
  produces the complete first draft, then a **draft → review loop** with the
  human (approve as-is or request tweaks; iterate). Not an interrogation.
- [ ] **Close-out.** On approval: validate, publish via the closing docs PR
  (agent `vrg-pr-workflow report-ready`; human `vrg-submit-pr`). The PR's merge
  closes the retrospective task — the last open child — and `vrg-finalize-pr`
  rolls up and closes the epic.
- [ ] **Validate:** `vrg-container-run -- vrg-validate` green. Optionally
  cross-check with the `writing-skills` skill (frontmatter/structure sanity).
- [ ] **Commit** (`docs(epic-retrospective): new terminal finishing skill`).

**Deliverable:** an invocable-once-released skill that gates on epic completeness
and drives the retrospective to a closing PR.

### Task 2 — reframe the epic lifecycle docs (vergil-claude-plugin)

**Files:**

- Modify: `skills/epic-create/SKILL.md`
- Modify: `skills/epic-implement/SKILL.md`
- Modify: `CLAUDE.md`

Depends on Task 1 (these reference the new skill by name). One cohesive PR — a
reviewer accepts or rejects the reframe as a unit.

- [ ] **`epic-create`:** in "The epic architecture — bookend tasks", add the
  **retrospective** as the mandatory terminal closing bookend seeded at creation;
  reorder the closing bookends so **documentation-review runs before** the
  retrospective; **demote the follow-on brainstorm to semi-optional** (seed only
  when a known enabling chain exists at creation, else accrued during
  implementation). Supersede the line stating documentation-review "is the final
  gate" (`epic-create/SKILL.md:67`) — the **retrospective** is now the final gate.
  Add seeding the retrospective bookend to the step-2 workflow.
- [ ] **`epic-implement`:** rewrite the "Terminal handoff" section — the final act
  is **draft the retrospective → human review → publish** (via
  `epic-retrospective`), not "prepare the follow-on brainstorm." Keep the
  follow-on brainstorm as an optional, forward-axis note.
- [ ] **`CLAUDE.md`:** in the epic-convention on-ramp, document the **three-skill
  lifecycle** (create ×1 → implement ×N → retrospective ×1) and the retrospective
  terminal bookend; in "Plans evolve append-only", note that the **Evolution
  during execution log is the raw feedstock for the retrospective's §1**.
- [ ] **Consistency check:** every cross-reference among the three skills +
  `CLAUDE.md` resolves; no surviving text calls documentation-review the final
  gate.
- [ ] **Validate:** `vrg-container-run -- vrg-validate` green.
- [ ] **Commit** (`docs(epic-skills): route closing bookends through the retrospective`).

**Deliverable:** the lifecycle docs consistently name the retrospective as the
terminal gate and the brainstorm as semi-optional.

### Task 3 — documentation-review sweep (#658, vergil-claude-plugin) · closing

Closing bookend; **runs before** the retrospective. Verify the change is
reflected across human-facing docs — primarily the three skills + `CLAUDE.md`
(expected plugin-contained). Spawn a same-repo doc task in another repo (e.g.
`.github#40`) only if the sweep finds a real gap. Closed by its own PR (or
"nothing to change" close if the impl PRs already covered it).

### Task 4 — retrospective (#203, .github) · terminal, dogfooded

Run `epic-retrospective` for epic `#201` **after** Tasks 1–3 (and any accrued
tasks) are closed and the plugin has been released + reloaded so the skill is
invocable. Its preflight gate confirms `#201` has no other open child, then
drafts `epics/201-retrospective-bookend/retrospective.md`; human review; closing
docs PR closes `#203` and rolls up the epic.

## Self-review

- Spec §3 artifact → Task 1 (artifact section) · Spec §4 skill/preflight/§0-data
  → Task 1 (preflight, §0 sourcing, authoring, close-out) · Spec §5 reframed
  bookends + supersessions → Task 2 · Spec §6 relationships (Evolution-log
  feedstock) → Task 2 (`CLAUDE.md`) · Spec §7 this-epic bookends → Tasks 3
  (#658) and 4 (#203) · Spec §8 out-of-scope (no `--kind retrospective`
  scaffold, no `summarize` change) respected — no task touches them.
- No follow-on-brainstorm bookend (spec §7); no cold-rebuild validation
  (behavioral/docs change).
- Terminality enforced only in the Task 1 preflight (no `Blocked-by`
  bookkeeping), consistent with spec §4/§5.

## Evolution during execution

*Frozen at execution start; dated entries record meaningful deltas — a task
added, dropped, or rescoped, a discovered dependency — with the reasoning.*

### 2026-07-25 — epic was not "plugin-contained" (task added)

Filed on the belief that the change was contained in `vergil-claude-plugin`, but
the documentation-review sweep (#658) found the **authoritative** epic-lifecycle
standard in `vergil-tooling/docs/site/docs/standards/github-issues.md` (and a
stale skills-site catalogue in the plugin). Per the placement law, spawned
**vergil-tooling#2531** to correct the standard in its own repo. Task count 5 → 6.

### 2026-07-26 — three tooling improvements absorbed at the retrospective gate

Drafting the retrospective surfaced three small tooling gaps: a
`--kind retrospective` scaffold (**#2537**), brittle epic sub-issue enumeration in
the preflight/audit path (**#2538**), and a missing `vrg-gh release` subcommand
(**#2539**). Rather than fork them to triage — where small items tend to
accumulate unfinished — we **absorbed them into this epic** (if it is small
enough to finish now, finish it now). This reopened the runnable frontier and,
correctly, **held the terminal retrospective**: the `epic-retrospective` preflight
will not pass until these close. Task count 6 → 9.
