# Every issue belongs to an epic

## Problem

The issue framework lets issues exist with **no epic parent**, from two sources:

1. **Intake kinds created unlinked by design.** `vrg-triage-create` mints
   `triage` / `idea` / `research` issues with no parent epic — a deliberate
   "capture now, groom later" intake queue.
2. **Issues born outside the tools.** Anything holding the app credential (a
   direct API call, the web UI, a script, CI) can open a bare issue the agent
   sandbox never sees. This is how `vergil-tooling#2461` and `#2468` appeared.

Orphaned issues do not roll up to any epic and, when legacy, link with `Ref`
rather than `Closes`, so they never auto-close — surfaced when `#2461`/`#2468`
stayed open after their PRs merged. That behaviour is correct-by-design for
legacy `Ref` issues, but it exposed the real gap: **the framework's core promise
— every unit of work lives under an epic that manages it — was only half
enforced.** The optionality was the bug.

## Goals

- **The invariant:** in any repo with a `vergil.toml`, every issue has an epic
  parent. Asserted, universal, no opt-out.
- Contain the intake kinds (`triage`/`idea`/`research`) in a standing epic,
  mirroring the existing ad-hoc epic, without making them PR-workable.
- Make the invariant hold regardless of *how* an issue is born — including the
  credential/web-UI paths the sandbox cannot police — by auto-healing, not
  rejecting.
- Retro-attach every existing orphan across all managed repos in every org.

## Non-goals

- **Perfect placement.** The migration and the safety net attach orphans to a
  catch-all standing epic (intake or ad-hoc); *re-homing* to the right finite
  epic happens later through normal grooming (`triage-review`), not here.
- **Rejecting or closing issues.** Enforcement heals (auto-attach); it never
  fights creation. A hard-reject mode is explicitly out of scope.
- **Reopening merged work.** `#2461`/`#2468` are already merged; attaching them to
  an epic is about the invariant, not their code.
- **Cross-org linkage.** Each org has its own `.github`; the invariant is
  enforced per-org, never across orgs.

## Invariant (the assertion)

**Every issue is either an epic (top-level) or has an epic parent.** Using Vergil
means using the issue tooling; the resolver that decides "which epic" is
data-driven and extensible in principle, but the invariant is non-negotiable and
carries no per-repo opt-out. Epics are the sole top-level issues — they have no
parent by design and are never swept.

## Design

### Standing intake epic (per-repo)

Add `ensure_intake_epic(repo)` in `lib/epics.py` alongside the existing
`ensure_adhoc_epic`, reusing the same machinery: a **perpetual, never-auto-close**
`Epic (intake): <bare repo>`, labelled `epic` + `intake`, homed via
`resolve_epic_home` (public repo → `<org>/.github`; private → itself), created and
reused **idempotently**. It holds the three intake kinds. The ad-hoc epic keeps
holding actual ad-hoc *work*; the two are distinct grooming-source buckets.

The intake epic reuses the ad-hoc epic's perpetual, never-rolls-up handling (the
`ad-hoc` branch of the closure logic generalizes to "a standing epic"), so it
never gates or triggers rollup.

### The shared orphan resolver

One function — **given an issue, which epic does it belong to?** — with a single
rule:

- the issue carries a `triage` / `idea` / `research` label → the repo's **intake**
  epic (`ensure_intake_epic`);
- otherwise → the repo's **ad-hoc** epic (`ensure_adhoc_epic`).

Built once and consumed by **both** the safety-net Action and the migration, so
the two can never diverge. The resolver is pure/deterministic given (issue,
labels, repo); the "attach" side effect is a separate step.

### Enforcement — two layers

1. **Client (removes the intentional orphan surface).** `vrg-triage-create` stops
   creating unlinked issues: it resolves and attaches each new intake issue to the
   repo's intake epic (born-linked, exactly as `vrg-issue-create` born-links under
   its `--epic`). After this, **no sanctioned tool can birth an orphan**
   (`vrg-issue-create` already requires `--epic`; `vrg-epic-create` makes epics;
   raw `gh` is hook-blocked; `vrg-gh issue create` is denied).

2. **Safety net — a scheduled self-sweep (makes the invariant hold for every
   other path).** A **scheduled GitHub Action** (`on: schedule`) in **each managed
   repo** periodically sweeps that repo's own orphans and attaches each via the
   shared resolver, leaving a comment that it was auto-homed and can be groomed.
   Auto-heal, never reject. Rationale for the shape:
   - **Per-repo, self-sweeping** — an Actions `on: issues` trigger only fires for
     issues in its *own* repo, so a single workflow in `.github` could never see
     member-repo issues; instead each repo heals itself, which also removes any
     need for a central "list of managed repos" registry.
   - **Scheduled, not event-driven** — the orphan hole is tiny (everything real
     goes through the tooling), so eventually-consistent healing on a schedule is
     proportionate; there is no external cron infrastructure, only a workflow
     managed as code.
   - **Authenticates as the app** — the sweep runs as the same GitHub App
     installation the tooling uses, because attaching a member-repo issue to its
     epic in `.github` is a cross-repo write the repo-scoped default
     `GITHUB_TOKEN` cannot perform.

**Sweep domain.** The resolver and the sweep act only on **open** issues that are
**not** pull requests, **not** labelled `epic`, and **not already** epic-parented.
Closed issues are left untouched — the invariant governs live work, not history.

### Intake issues stay non-PR-workable (closes `#175`)

Being epic-attached must **not** make a `triage`/`idea`/`research` issue
PR-workable — otherwise a raw intake seed could acquire a PR and close as though
groomed. So the PR path refuses an intake-labelled issue the same way it already
refuses an operational-labelled one: `vrg-pr-workflow report-ready` and
`vrg-submit-pr` (and, upstream, `issue-implement`) detect the intake label and
refuse with "groom this into a real task first." This directly answers and closes
`vergil-project/.github#175`.

Grooming is the sanctioned exit: `triage-review` converts an accepted intake
issue into a real task — dropping the intake label and re-homing it under a finite
epic — at which point the PR path accepts it. The refusal gates only *un-groomed*
intake kinds.

### Migration — the same sweep, run once

The migration is not a separate mechanism: it is the **same sweep command** run
once for immediacy, after which the scheduled workflow maintains the invariant.
Run against each managed repo, it enumerates that repo's open orphans (per the
sweep domain above), attaches each via the shared resolver, is **idempotent and
re-runnable** (already-parented issues are skipped), reports a per-repo tally, and
never closes or relabels — it only attaches.

Because it mutates live GitHub state across many repos, the one-shot run is proven
by *running* it and recording the outcome, not by a code PR — see the operational
task in the breakdown.

## Data model — what varies and what does not

- **Varies (data-driven / extensible):** the set of intake kinds and their labels;
  the resolver's label→epic mapping; the standing-epic titles/labels. These live
  as constants beside the existing `_ADHOC_EPIC_*` definitions.
- **Does not vary (the invariant):** every issue has an epic parent. No repo may
  opt out; there is no "unlinked" terminal state.

## Testing

- `ensure_intake_epic`: idempotent create/reuse; correct home per visibility;
  perpetual/never-auto-close classification.
- Resolver: intake-labelled → intake epic; everything else → ad-hoc; pure and
  deterministic; unknown/multiple labels resolve deterministically.
- `vrg-triage-create`: new intake issues are born-linked under the intake epic;
  no unlinked path remains.
- PR-workability gate: `report-ready`/`vrg-submit-pr` refuse an intake-labelled
  issue with the grooming message; a groomed (re-kinded) issue is accepted.
- Sweep: enumerates open orphans; excludes epics/PRs/closed/already-parented;
  attaches via the resolver; idempotent re-run is a no-op; per-repo tally; never
  closes/relabels.
- Scheduled workflow: unit-test the resolver and sweep-domain predicate; the
  workflow itself follows `vergil-actions` test conventions.

## Task breakdown (this epic)

Documentation and closing bookends are already seeded (`#177`, `#178`,
`vergil-tooling#2474`). Implementation tasks, each filed in the repo where its PR
lands and linked under this epic:

1. **`lib/epics.py`** — `ensure_intake_epic` + the shared resolver
   (`vergil-tooling`).
2. **`vrg-triage-create`** — born-link intake issues under the intake epic
   (`vergil-tooling`).
3. **PR-workability gate** — refuse intake-labelled issues in
   `report-ready`/`vrg-submit-pr`; closes `#175` (`vergil-tooling`).
4. **Sweep command** — a `vrg-*` command that sweeps a repo's open orphans and
   attaches each via the shared resolver; serves both the one-shot migration and
   the scheduled workflow (`vergil-tooling`).
5. **Reusable scheduled-sweep workflow** — an `on: schedule` reusable workflow
   that runs the sweep command authenticated as the app (`vergil-actions`), plus
   its rollout into each managed repo's `.github/workflows/` via the standard
   scaffold.
6. **Operational: run the one-shot sweep** — a run task that executes the sweep
   across all managed repos once (proven by an `Outcome: SUCCESS` comment with the
   per-repo tally), blocked-by tasks 1 and 4 and their deployment.

## Follow-ons

- Whether the intake epic should ever *split* per kind if a single bucket grows
  unwieldy (deferred; start with one).
- Grooming ergonomics: `triage-review` re-homing swept orphans out of the
  catch-all epics is the ongoing counterpart to this one-shot invariant.
