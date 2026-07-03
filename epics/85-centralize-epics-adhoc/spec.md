# Centralize epics in `.github` — ad-hoc rename, input queues, and epic bookends

- **Epic:** vergil-project/.github#85
- **Status:** Draft design (2026-07-03) — pre-pushback
- **Docs task:** vergil-project/.github#86
- **Brainstorm source:** superpowers brainstorming session, 2026-07-03

## 1. Summary

The epic/task model works, but two asymmetries make the org-wide roadmap harder
to see than it should be:

1. **Finite epics are centralized in `.github` and visible; standing epics are
   scattered one-per-repo and hidden.** You cannot get the whole epic roster in
   one view.
2. **`.github` mixes two kinds of issue** — epics plus loose org-level triage
   dropped in as plain issues — so the list needs mental filtering.

This epic makes `.github` the single, high-signal home for the org's strategic
view: **all epics in one place, plus one org-wide intake queue**, with every
concrete unit of work living where its pull request lands. It also formalizes a
convention that keeps the roadmap alive: **an epic is never closed until you
have decided what comes next.**

## 2. The two invariants

Everything below follows from two rules:

1. **All epics live in `<org>/.github`** (finite and ad-hoc alike), labelled
   `epic`.
2. **Every repo other than `.github` holds nothing but single-PR tasks** — each
   task closed by exactly one PR in that same repo. `.github` additionally holds
   the three input-queue labels and the rare task whose closing PR modifies
   `.github` itself.

The load-bearing principle behind invariant 2 is **task↔PR locality**: a task
lives in the repo whose PR closes it (tasks and PRs are 1:1). This principle
*won* over the earlier, more elegant-sounding "zero tasks in `.github`" idea —
cohesion between a task and the PR that closes it matters more than a pristine
`.github` issue list. In practice `.github` reads as "epics only" because you
filter by the `epic` label; the exceptions are self-justifying (a task is in
`.github` iff its PR is in `.github`).

## 3. Ad-hoc epics (rename of "standing")

### 3.1 Terminology

Rename **"standing" → "ad hoc"** everywhere: the label, the epic title, the CLI,
the code, and the docs. The word does deliberate psychological work — "standing"
sounds permanent and legitimate, whereas "ad hoc" names the smell a serious
engineer wants to drive toward zero. The long-term goal is for ad-hoc epics to
become the rare exception as real work is increasingly shaped into finite,
tidy, auto-closing epics.

### 3.2 Model

- **Exactly one ad-hoc epic per repo**, including `.github` itself, **all
  physically located in `<org>/.github`**.
- **Title:** `Epic (ad hoc): <repo>` where `<repo>` is the **bare** repo name
  (e.g. `Epic (ad hoc): vergil-tooling`, `Epic (ad hoc): .github`). Bare because
  they all live in one org's `.github`; owner-qualifying would be noise.
- **Labels:** `epic` + `ad-hoc` (hyphenated).
- **Perpetual:** created once, never auto-closed, kept open even when empty.
  This perpetual-ness is the *only* thing distinguishing an ad-hoc epic from a
  finite epic.
- **Lookup (Option A — title-based):** the ad-hoc epic for repo `X` is found by
  searching `.github` for `epic` + `ad-hoc` with the title `Epic (ad hoc): X`.
- **Tasks live in the target repo**, cross-repo linked up to the ad-hoc epic in
  `.github` via native sub-issues (reflink fallback where sub-issues are
  unavailable).

### 3.3 Tooling changes

- `lib/epics.py`:
  - Rename `ensure_standing_epic(repo)` → `ensure_adhoc_epic(target_repo)`. It
    resolves `<org>` from `target_repo`'s owner and locates/creates the epic in
    `<org>/.github` titled `Epic (ad hoc): <bare target_repo>`, labelled
    `epic` + `ad-hoc`. "Exactly one" is now scoped by title within `.github`
    (many ad-hoc epics coexist there, one per repo).
  - `resolve_epic_ref`: accept the `"adhoc"` sentinel (retire `"standing"`).
  - `rollup`: the perpetual guard changes from `"standing" in labels` to
    `"ad-hoc" in labels`.
  - Rename the `_STANDING_EPIC_*` constants and update the body text.
- CLI: rename `vrg-standing-epic` → `vrg-adhoc-epic`; its `ensure` subcommand
  takes the target repo and ensures that repo's ad-hoc epic in `.github`. Update
  the `pyproject.toml` console-script entry.
- `--epic standing` sentinel → `--epic adhoc` across `vrg-issue-create` and any
  callers.

## 4. Input queues — triage / idea / research

Replace the single "triage" intake with **three labelled input shapes, all
routed to `.github`**, so the entire org-wide intake is one filtered view beside
the epic roster:

| Label | Captures | Graduates into |
|---|---|---|
| `triage` | A problem/bug not yet understood — needs diagnosis | an epic (or a task) |
| `idea` | A spark — "what if we did this crazy thing" | a feature/epic |
| `research` | An investigation that produces a **reproducible** result | an epic with tooling PRs + a report |

Key judgment baked in: **research is not ad-hoc work.** A result worth having is
a result worth reproducing, so research spawns automated tooling and PRs — it
becomes a proper finite epic, never something done by hand.

### 4.1 Tooling changes

- Extend the triage creator into **one tool with a `--kind triage|idea|research`
  flag** (default `triage`), emitting the corresponding label. The target repo
  **defaults to `<org>/.github`** (org auto-detected), replacing the old
  "current repo / `.github` for project-level seeds" default. One tool with a
  kind flag keeps this trivially extensible if a fourth intake shape ever
  appears, while three labels remain enough to avoid category sprawl.
- Create the `idea` and `research` labels (via `vrg-ensure-label`).
- **Open (for the plan):** whether to keep the tool named `vrg-triage-create`
  (minimal churn; slight misnomer now that triage is one of three kinds) or
  rename it to a neutral intake name. Recommend keeping the name to limit churn
  unless pushback disagrees.

*What happens to an intake item after capture* (become a task under an epic?
graduate to its own epic?) is deliberately **not formalized here** — that is
follow-on epic #87 (periodic review process). This epic only guarantees a single
visible place to capture and see them.

## 5. `docs` is demoted to an ordinary repo

The earlier idea of using `docs` as the org-wide "none-of-the-above" catch-all
is **dropped** — killed by the task↔PR-locality invariant, which makes a
doc-task's home obvious (it lands in `docs`, so it lives in `docs`). `docs`
becomes a normal member repo whose issue list holds only its own single-PR
tasks. The only thing still notable about `docs` is that, like `.github`, it is
a generic repo every org tends to have. Action: audit tooling/skills for any
`docs`-as-catch-all assumptions and remove them.

## 6. The epic-bookend convention

**An epic is never closed until you have formally decided what comes next.**
Almost no real-world problem is 100% closed by a single epic; you deliver a
tangible, completed subset and acknowledge the follow-on. So every epic carries
a minimum of two **bookend** tasks:

- **First task = documentation** — the spec/plan for the epic, born from
  planning.
- **Last task = review + brainstorm the follow-on** — review what shipped
  (successes, failures, mid-flight changes, newly-found problems and
  opportunities, both in the tooling and in the epic's target), then brainstorm
  and create the follow-on epic(s). If the answer is the rare "nothing," you are
  done — but you always ask.

This rides the **existing auto-close rollup**: an epic rolls up only when all
its tasks are closed, so the review task *naturally gates* closure. No new
closing mechanism is required — the forcing function is free.

Mechanization lives in **prose in the `epic-create` plugin skill**, not in rigid
tooling, because "what is the right follow-on?" is an inherently agentic,
abstract judgment. The skill describes the convention and seeds the two bookend
tasks; the agent fills them in per epic. (This epic itself models the
convention — see §8.)

## 7. Formalize `epic-create` as the orchestrating outer workflow

A discovery from the brainstorm: the concrete step sequence we followed to
*create this epic* is itself the formalization of how `epic-create` should work.
Today the nesting is backwards — a session starts in `superpowers:brainstorming`
and reaches for `epic-create` awkwardly at the end. **Invert it:** `epic-create`
becomes the *outer* workflow that owns the end-to-end sequence and interleaves
the Vergil epic/task tooling with the superpowers + paad skills at defined
handoff points:

1. `epic-create` (outer) begins by running `superpowers:brainstorming`.
2. On an approved design, initialize the epic in `.github` and seed the bookend
   tasks (docs first task + follow-on last task[s]).
3. On the docs task's worktree, write `spec.md`.
4. `paad:pushback` on the spec → commit to the same worktree.
5. Human review.
6. `superpowers:writing-plans` → `plan.md` into the same worktree.
7. `paad:alignment` → reconcile plan (and maybe spec).
8. Single docs PR (spec + plan) against `.github` → closes the docs task
   (human-run submit per the PR-handoff policy).
9. File the implementation tasks from the plan and link them under the epic.

This workstream rewrites the `epic-create` plugin skill accordingly (PR lands in
`vergil-claude-plugin`). It is the point where the superpowers/paad workflow and
the Vergil epic tooling become one integrated pipeline.

## 8. This epic embodies the convention

Already created under epic #85:

- **#86** — docs task (spec + plan), the first bookend (this document closes it).
- **#87** — follow-on: periodic review process (grooming intake + strategic
  cadence).
- **#88** — follow-on: inter-issue dependency metadata.
- **#89** — follow-on: deployment-state metadata.
- **#90** — follow-on: automated PR-review agent.

Per the convention, epic #85 cannot roll up until #87–#90 are dispositioned.

## 9. Migration

Existing per-repo standing epics must move into `.github`:

- For each repo that has an `Epic (standing): Ad-hoc maintenance`, create
  `Epic (ad hoc): <repo>` in `.github`, **re-link its child tasks** (which live
  in the member repo) to the new `.github` epic, then **close** the old per-repo
  standing epic.
- Also ensure the `.github` and `docs` ad-hoc epics exist.
- Recreate-and-relink is preferred over `gh issue transfer` because native
  sub-issue links may not survive a transfer; the plan confirms the mechanism.
- Retire the `standing` label (rename to `ad-hoc` or create `ad-hoc` and
  relabel).

The `migrate-repo` skill and any docs referencing standing epics are updated to
the new model.

## 10. Component checklist (to be detailed by the plan)

- `lib/epics.py` — ad-hoc rename, `.github`-scoped lookup, sentinel, rollup guard.
- `vrg-adhoc-epic` (rename of `vrg-standing-epic`) + `pyproject.toml`.
- Triage creator — `--kind`, default-to-`.github`.
- `vrg-ensure-label` calls / label set — `ad-hoc`, `idea`, `research`; retire
  `standing`.
- `lib/epic_audit.py` + roadmap — new invariants; ad-hoc epics never flagged
  stale; optional invariant-enforcement checks (non-`.github` repo carrying an
  `epic` label; `.github` carrying a non-epic/non-input/non-self-referential
  issue).
- `epic-create` plugin skill — orchestrator rewrite + bookend convention
  (`vergil-claude-plugin`).
- Migration of existing standing epics; `migrate-repo` skill update.
- Docs / CLAUDE.md — "standing" → "ad-hoc"; `docs` demotion.

## 11. Follow-on epics (captured, not designed here)

Filed as tasks #87–#90; each becomes its own brainstorm → epic:

1. **Periodic review process** — weekly/monthly grooming of the
   triage/idea/research queues into implementable epics, and formalizing the
   strategic review cadence.
2. **Inter-issue dependency metadata** — know when a task is implementable;
   enable agent queues that pick up dependency-free work.
3. **Deployment-state metadata** — distinguish "closed on merge to develop" from
   "deployed/available in main"; the lifecycle layer dependency checks need.
4. **Automated PR-review agent** — the human merge-gate judgment (rules beyond
   validation); revives the parked audit role. Speed gained on top of safety,
   never at its expense.

## 12. Out of scope (this epic)

- The four follow-on epics above (captured, not designed).
- The *process* for reviewing/dispositioning intake items (follow-on #87).
- Any implementation of dependency or deployment-state metadata
   (follow-ons #88/#89).
