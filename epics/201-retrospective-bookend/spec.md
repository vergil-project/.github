# Retrospective bookend for the epic lifecycle

- **Epic:** vergil-project/.github#201
- **Status:** Design approved 2026-07-24 (brainstormed directly; full epic
  convention).
- **Repo:** vergil-project/vergil-claude-plugin (skills + `CLAUDE.md`); epic
  homed in `.github`.
- **Promoted from:** direct brainstorm (no triage origin).

## 1. Summary

Give every epic a **mandatory, single, terminal retrospective** — the
backward-looking counterpart to the opening spec+plan docs PR. It partners
`spec.md`/`plan.md` at the documentation tier, so anyone revisiting an epic
later reads **spec → plan → retrospective**: what we set out to do, how we
planned it, and honestly how it went. Publishing the retrospective's docs PR is
what **closes the epic**.

## 2. Motivation

The opening bookend (spec + plan) is where the fleet's front-loaded judgment
already lands, and it works well. The *closing* side is thinner. Today's closing
bookends are a documentation-review sweep and a **follow-on brainstorm** — but
that brainstorm is purely **forward-looking** ("we solved X; what's now
possible?") and, in practice, those follow-ons are increasingly spun off *during*
implementation as specific tasks. Nothing mandates a **backward-looking** record
of how the epic actually went: what shipped vs. what was planned, lessons,
compromises, and the new problems it surfaced. Complex epics deviate from their
plans (the `plan.md` Evolution log captures the deltas, but not the synthesized
story), and that reflection is currently lost. The retrospective fills that gap
and becomes the durable "start here" summary for a later reader.

Its deeper purpose is **honesty about the delta between what we planned and what
we actually did** — the check a Gantt chart never circles back to make. A small
delta means the spec/planning discipline is working; a large one means the epic
taught us something. Either way it is captured, not lost.

This completes a **three-skill epic lifecycle**, each with a distinct invocation
cardinality:

- **`epic-create`** — run **once**: make the epic exist (spec + plan + bookends).
- **`epic-implement`** — run **N times**: drive the runnable frontier, pause on
  blockers (sometimes for weeks, sometimes spinning off a new epic to unblock),
  resume later when the need is real. Many epics live parked between runs.
- **`epic-retrospective`** — run **once**: the finishing gate every epic passes
  through.

## 3. The artifact — `epics/<N>-<slug>/retrospective.md`

A backward-looking record, **short and scannable up top, detail below**:

- **§0 At a glance** *(preamble, ≤ ½ page)* — one paragraph "what we set out to
  do → what shipped", then a **work-delivered visual**: a PR table (each with a
  one-line "what it did"), repos touched, task/PR counts, releases cut,
  opened→closed span. The landing view: *here is the bulk of what was done.*
- **§1 How the plan evolved** — a **synthesized narrative** from `plan.md`'s
  "Evolution during execution" log (the *why* behind deviations, not a copy).
- **§2 Lessons learned** — transferable insight; same/different next time.
- **§3 Compromises & tradeoffs** — corners cut, debt knowingly incurred, and the
  reasoning.
- **§4 New problems & opportunities** — what the epic surfaced, and for each,
  where it went (spun-off epic/task ref, or "logged, not yet acted on").
- **§5 What's next** — pointers to any follow-on brainstorms/epics (referenced,
  not duplicated).
- **Appendix A — Operational notes** *(optional; migration/deploy/validation-heavy
  epics only)* — the mechanical sequence (repo-by-repo, publish/deploy order,
  gotchas). A `summarize` operations-mode log may drop in here as a subset.
- **Appendix B — Extended metrics** *(optional)* — deeper stats worth keeping.

The retrospective is **honest, not a victory lap** — §3/§4 are load-bearing.

## 4. The `epic-retrospective` skill — authoring model

A new skill (sibling to `epic-create` / `epic-implement`) that mirrors
`writing-plans`, **not** the interactive `alignment` skill. It is the **single
entry point for *finishing* an epic**, and enforcement of terminality lives here
and nowhere else:

0. **Preflight — definition-of-done gate.** Resolve the epic, enumerate every
   sub-issue, and **abort** if *any* child other than the retrospective task is
   still open ("not the terminal task; N still open: #…"). Running the skill *is*
   the assertion "we're done"; the gate mechanically verifies it, the same
   refuse-and-stop shape as operational tasks' precondition self-check. The
   retrospective is thus **implicitly blocked by every other child, resolved
   dynamically at run time** — there are **no static `Blocked-by` reflinks** to
   maintain (they would not scale against a dynamic epic). This subsumes ordering:
   the documentation-review sweep and any per-repo siblings it spawned are just
   other open children the gate catches.
1. **Agent drafts the complete first draft.** The agent did the work, so it
   assembles §0 factually from **`vrg-epic-audit`** (the epic's task/PR graph)
   plus `vrg-gh` for PR titles / merge dates / releases — marking anything it
   cannot determine as *unknown* rather than inventing it — synthesizes §1 from
   the plan's Evolution log, and writes its honest read of §2–§5 from execution
   context.
2. **Draft → review loop with the human.** The agent presents the draft and asks
   for approval; the human either approves as-is or requests tweaks ("also
   capture X"), and the agent iterates. The human's heavy input was up front on
   spec+plan; here they add what the agent *missed*, not co-write from scratch.
3. **Closing docs PR.** On approval, the retrospective is published by the docs
   PR (agent `vrg-pr-workflow report-ready`; human `vrg-submit-pr`). Its merge
   closes the retrospective task and rolls up the epic.

## 5. Reframed closing bookends (`epic-create` / `epic-implement`)

The closing side becomes an ordered structure:

- **Documentation-review sweep** — unchanged, still mandatory; runs **before**
  the retrospective so the retro can honestly record "docs updated". Still a
  multi-repo sweep that can spawn per-repo doc tasks.
- **Follow-on brainstorm — demoted to semi-optional.** It is a **forward-axis**
  concern that no longer defines the end of an epic. Seed it as a bookend **only
  when a known enabling chain exists at creation time** ("we're building this
  specifically to unlock X/Y/Z"); otherwise it is accrued as specific follow-on
  tasks during implementation, or simply not needed. Its outcomes are *recorded*
  in the retrospective's §5.
- **Retrospective — new mandatory terminal gate.** Seeded as a bookend task at
  epic-creation time (docs PR against the epic's home repo). Terminality is
  **enforced by the `epic-retrospective` skill's preflight gate** (§4 step 0) —
  *not* by static `Blocked-by` reflinks — so it works correctly under a dynamic
  epic. When the skill passes its gate, publishing the retrospective's docs PR
  closes the last open child, and the existing `vrg-finalize-pr` rollup closes
  the epic.

Two existing statements are explicitly **superseded**: `epic-create`'s
"documentation review **is the final gate**" (it now runs *before* the
retrospective), and `epic-implement`'s terminal-handoff step. `epic-implement`'s
final act becomes **draft the retrospective → human review → publish**, not
"prepare the follow-on brainstorm."

## 6. Relationship to existing pieces

- **`plan.md` Evolution log** — the append-only deltas are the **raw feedstock**
  for retrospective §1; the retrospective synthesizes them into narrative, it
  does not replace the log.
- **`summarize` skill** — orthogonal. Its operations-mode output may appear as a
  **subset** inside Appendix A for operationally-heavy epics (e.g. a migration),
  but the retrospective does not route through `summarize`; it is purpose-built
  and epic-focused.
- **`documentation-review` bookend** — distinct job (verify the wider human-facing
  docs reflect reality); the retrospective is the epic's own historical record,
  not a docs-consistency sweep.

## 7. Bookends & prevention (this epic)

- **Documentation** (#202): this spec + plan.
- **Documentation review** (vergil-claude-plugin#658, closing sweep) — verify the
  skills (`epic-create`, `epic-implement`, new `epic-retrospective`) and
  `CLAUDE.md` reflect the change. Expected **plugin-contained**; spawns an outside
  sibling (e.g. `.github#40` convention) only if the sweep finds a real gap.
- **Retrospective** (#203, terminal gate) — **dogfooded**: authored with the new
  `epic-retrospective` skill this epic introduces. Its merge closes the epic.
- **No follow-on-brainstorm bookend** — no known enabling chain at creation; any
  follow-on is accrued during implementation.
- **No cold-rebuild validation** — behavioral/docs change, not infra.

## 8. Out of scope

- Retroactively authoring retrospectives for already-closed epics.
- Any change to the `summarize` skill (it is referenced, not modified).
- Automated metrics/dashboards beyond what §0 and Appendix B capture by hand.
- Tooling for a dedicated `--kind retrospective` scaffold — the terminal bookend
  is a plain docs task for now; a sanctioned `vrg-issue-create` scaffold is a
  candidate follow-on, not part of this epic.
