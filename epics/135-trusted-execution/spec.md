# Front-Loaded Judgment, Trusted Execution

**Epic status:** proposed
**Epic:** `vergil-project/.github#135`
**Invariant reversed:** the oversight-tuned behavior mandate in the `issue-*` skills
**Related:** `vergil-project/.github#40` (epic/task convention),
`vergil-project/.github#115` (post-merge validation),
`vergil-project/.github#124` (operational tasks)

## Problem

The vergil skills encode a behavioral invariant tuned for **maximum human
observability**: implementation skills run **inline, in the foreground**,
narrate every step, and **never spawn sub-agents**, on the explicit rationale
that *the visible progress is the human's oversight* (`issue-implement`,
`issue-localize`, `issue-validate`, `issue-deploy`).

That invariant was set when the human watched implementation happen — scrolling
back to audit what the agent did, catching mistakes mid-flight. That working
style is no longer practiced. The judgment now lives in a heavily front-loaded
planning pipeline (brainstorm → pushback → alignment) and at the hard,
human-gated boundaries (PR submit, merge, release). Between those, the human is
not watching — yet the skills still pay a real economic cost, in tokens and
context, to enable monitoring that is not leveraged. Worst of all, the
"never spawn a sub-agent" clause **bans the single most powerful efficiency
mechanism available** — sub-agent fan-out for research and parallel work —
purely to preserve a linear, human-readable transcript nobody reads.

Separately, there is no skill that operates at the **epic** level. The human
drives an epic by hand: reading `vrg-epic-audit`, picking the next runnable
task, invoking `issue-implement`, remembering where things stand across
interruptions. When an epic is 89% done and the session is lost or sidetracked,
reconstructing "where were we" is manual and error-prone.

These two problems share one root and one fix. The oversight invariant is what
*prevents* an efficient epic-level driver from existing; reversing it is what
*enables* one. So we reverse the invariant and ship the driver together — the
driver is both the motivating capability and the first exemplar of the new rule.

## Goals

- **Reverse the invariant** across every skill and doc that states it, replacing
  "tune for oversight" with "front-loaded judgment, trusted execution," and
  record the reversal as a single canonical positive statement (not merely
  deleted text).
- **Encourage sub-agents** as a first-class efficiency mechanism wherever they
  help (research fan-out, parallel implementation), removing the categorical ban.
- **Ship `epic-implement`**, a thin driver above the `issue-*` skills that
  resumes an epic purely from GitHub state, works the runnable frontier
  (sub-agents encouraged), batches human-gated actions, and escalates on
  problems.
- **Preserve every hard gate.** PR submission, merge, and release stay
  human-gated. The reversal changes *mid-flight execution*, not the boundaries.
- **Preserve every non-oversight invariant** untouched — especially the
  never-fabricate / block-not-fabricate rule in the operational-task skills and
  the never-suppress-a-gate rule in `issue-implement`.

## Non-goals

- **No new enforcement machinery for autonomous-close prevention.** The closing
  brainstorm is human-attested by riding the existing human-gated merge; we add
  no new guard (YAGNI). See *Closing-brainstorm invariant*.
- **No agent self-rename.** The Claude Code session cannot be renamed by the
  agent (`/rename` is human-only — no tool, hook, or API). `epic-implement`
  emits a paste-ready rename line; it does not attempt to rename the session.
- **No intra-task parallelism mandate.** Sub-agents are *encouraged where
  efficient*, a judgment call — not required. Nothing forces fan-out.
- **No change to the front-loaded planning pipeline.** brainstorm → pushback →
  writing-plans → alignment is unchanged; this epic changes what happens *after*
  the plan, during execution.
- **No fleet/session-grouping mechanism.** Naming a session after its epic stays
  the human's manual act (paste the emitted rename line). No tooling for it.

## The invariant reversal (old → new)

**Old — "Continuous Oversight."** Tune agent behavior for maximum human
observability. Work inline / in the foreground, narrate everything, never spawn
sub-agents — because the visible progress *is* the oversight.

**New — "Front-Loaded Judgment, Trusted Execution."** Human judgment is spent
**up front** (brainstorm → pushback → alignment) and at the **hard gates** (PR
submit, merge, release). Between those, the agent executes by the most efficient
means available — **sub-agents encouraged**. The only mid-flight interrupt is a
**problem the agent cannot resolve**: it stops and asks. Observability for its
own sake is dropped when it costs tokens and context.

This is a suite-wide behavioral invariant, not a one-skill tweak. It is recorded
once as a canonical statement (destination decided at plan time — a candidate is
a short doctrine section in `CLAUDE.md` and the site docs) and every skill that
contradicted it is brought into line.

## The `epic-implement` driver

`/vergil:epic-implement <epic-ref>` — a thin layer above the `issue-*` skills.

### 1. Resume from GitHub, always

State is reconstructed from `vrg-epic-audit`: closed / runnable / blocked tasks
and the dependency graph. **No session state is load-bearing.** Re-invoking
after a lost or compacted session simply re-derives position and continues.
In-session accumulated context (richer judgment, spotted follow-ups, notes for
the closing brainstorm) is a *bonus*, never a requirement — its loss under
compaction is acceptable by design.

### 2. Emit a paste-ready rename line

On startup the driver reads the epic title and prints, e.g.
`/rename epic-135-trusted-execution`, for the human to paste. This is the
whole of the "name the session after its epic" feature — the agent cannot do it
itself.

### 3. Work the runnable frontier

Take every currently-runnable task (all blockers closed) and drive each to its
gate, **parallel sub-agents encouraged where efficient**, routing by label:

| Task kind | Run via |
|---|---|
| code (default `task`) | `issue-implement` |
| `validation` | `issue-validate` |
| `deployment` | `issue-deploy` |

### 4. Batch at the gate

When the frontier is worked, present a single consolidated human-gated set:
PRs ready to submit/merge, operational tasks needing a human-gated release, and
any problems the driver got stuck on. The human acts out of band, returns, and
says continue (or re-invokes) → the driver re-derives state and advances to the
next frontier.

### 5. Escalate on problems, don't thrash

Matching the new invariant: a problem the agent cannot resolve is the *only*
mid-flight reason to pull the human in. Stop and ask; never thrash or fabricate.

### 6. Terminal handoff (hybrid)

Drive the **mechanical** closing bookend — the documentation-review task — to
done like any other task. For the **follow-on brainstorm** bookend, **stop and
prepare**: assemble the accumulated observations (what shipped, what went
sideways, new problems and opportunities) into a seed, and hand the human into
the closing brainstorm. The driver never runs or closes that brainstorm itself.

## Closing-brainstorm invariant

The closing brainstorm is the **final human gate**. By construction almost no
epic is 100% done: it enables follow-on work and exposes new problems, and the
epic is not wrapped until the human and agent review what shipped and decide
what is next.

An epic auto-rolls-up when all its tasks close, but the **closing-brainstorm
task is human-attested** — it closes only via the human-gated docs PR that the
brainstorm produces (or a manual human close). `epic-implement` never closes it.
Because the closer is a human-gated merge, premature autonomous closure is
already impossible; **no new enforcement machinery is added** (YAGNI).

## Doctrine cleanup (concrete edits)

- **`issue-implement`** — delete the "Run it in the foreground — be transparent /
  never spawn a sub-agent / visible progress is the oversight" block; replace
  with new-invariant language (execute by the most efficient means, sub-agents
  encouraged, escalate problems). **Keep** never-suppress-a-gate and
  `report-ready` → human-submits.
- **`issue-localize`** — same removal/replacement (identical clause).
- **`issue-validate` / `issue-deploy`** — drop only the foreground-for-oversight
  framing. **Preserve the never-fabricate / block-not-fabricate rule** in those
  same paragraphs — a separate, load-bearing invariant.
- **`pr-watch`** — leave as-is; its "foreground, don't poll" concerns correct use
  of the blocking `vrg-pr-await` call, not oversight. Optional one-line clarifier
  so it does not read as the old doctrine.
- **`CLAUDE.md` / `README` / `docs/site`** — audit for any statement of the
  oversight model as a principle; rewrite to the new invariant and add the one
  canonical positive statement.

## Deliverables (plan will expand into tasks)

All implementation lands in `vergil-claude-plugin`:

1. **`epic-implement` skill** — the driver (the substantial task).
2. **`issue-implement` cleanup** — invariant reversal in that skill.
3. **`issue-localize` cleanup** — same reversal.
4. **`issue-validate` / `issue-deploy` cleanup** — drop oversight framing, keep
   never-fabricate.
5. **Canonical invariant statement** — `CLAUDE.md` / `README` / `docs/site`.

Tasks 2–5 are **mutually independent** — a deliberate first exercise of
`epic-implement`'s frontier-batching. Bookends already seeded: documentation
(`#136`), closing brainstorm (`#137`), doc review (`vergil-claude-plugin#625`).

## Risks and trade-offs

- **Loss of a debugging affordance.** Removing mandated foreground narration
  means a failed run is less linearly auditable. Mitigation: the escalate-on-
  problem contract, the hard gates, and the fact that authoritative state lives
  in GitHub, not the transcript. Accepted deliberately.
- **Sub-agent worktree contention.** Parallel implementation means multiple
  in-flight branches/worktrees at once. This is consistent with the existing
  parallel-agent worktree convention (one worktree per issue); the driver must
  honor it rather than stacking issues in one tree.
- **`vrg-epic-audit` is now load-bearing for resume.** The driver's correctness
  depends on the audit accurately reporting runnable-vs-blocked. Any gap there
  surfaces as the driver picking the wrong frontier — worth verifying against a
  real multi-task epic (this epic itself is a candidate).

## Open questions (for pushback / alignment)

- Where exactly the canonical invariant statement lives (CLAUDE.md vs a site-docs
  doctrine page vs both).
- Whether `epic-implement` should itself run parallel `issue-implement`
  sub-agents, or fan out lighter research sub-agents while implementing tasks
  sequentially — i.e. how aggressive the default parallelism should be.
- Whether the driver should proactively *file* newly-discovered follow-on issues
  mid-epic (via `triage-capture`) or only surface them in the batch report.
