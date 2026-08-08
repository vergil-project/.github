# Continuous archiving of ad-hoc epics — drain closed work by close-quarter

- **Epic:** vergil-project/.github#238
- **Status:** Reviewed design (2026-08-08) — post-pushback, revised to
  continuous-drain (supersedes the quarterly-batch design)
- **Brainstorm source:** superpowers brainstorming session, 2026-08-08
- **Implemented in:** vergil-tooling (CLI + shared lib + hooks into the existing
  rollup/sweep) — tasks filed there at implementation time

## 1. Summary

Ad-hoc epics (`Epic (ad hoc): <repo>`, labelled `epic`+`ad-hoc`, homed by
visibility) are **perpetual umbrellas**: `vrg-issue-create --epic adhoc` routes
small, unrelated work to them, they are reused idempotently, and they are
**never closed** (`epics.rollup()` explicitly skips any `ad-hoc`-labelled epic —
`epics.py:470`). With no closing mechanism they accumulate without bound and
become large, unmanageable lists of mostly-closed issues.

This epic keeps them small by **continuously draining closed work out** of the
live ad-hoc epic into **per-quarter archive epics**, bucketed by the quarter in
which each issue was **closed**. The live ad-hoc epic therefore holds only
*currently-open* work at all times; finished work flows out incrementally into
`Epic (ad hoc): <repo> — <YYYY>-Q<n>`.

The live ad-hoc epic is **never renamed, replaced, or closed** — its identity is
untouched, so the `ensure_adhoc_epic` resolver keeps working with no changes. The
single operation is: *re-parent each closed child of a live ad-hoc epic into its
close-quarter archive.* This replaces the earlier quarterly-batch design (rename
the live epic, create a fresh one, migrate open children) — that design's
rename/replace dance and its crash-recovery complexity are gone entirely.

Because the work is incremental, it **reuses the epic system's existing
automation** — the `issues.closed` event path and the daily `epic-sweep` — rather
than adding a new quarterly cron.

## 2. The problem in detail

The ad-hoc epic is resolved by **search** (`epics.ensure_adhoc_epic`,
`epics.py:350`): list open issues in the home repo with labels `epic`+`ad-hoc`,
keep the row whose title exactly equals `Epic (ad hoc): <bare-repo-name>`. Because
it is a perpetual umbrella, `rollup()` returns early for it (`epics.py:470`) and
it never auto-closes.

The consequence is unbounded growth: over months a single ad-hoc epic collects
dozens to hundreds of small closed issues plus a handful of open ones, and the
signal (what is *currently* in flight) drowns in the archive (what was finished
long ago). There is no existing mechanism to bound it — this behavior is new.

## 3. Design

### 3.1 The archiving operation (continuous drain)

Given a scope (one repo, or an org — see 3.4), for each **live** ad-hoc epic
(canonical title, open):

1. **List** its children with each child's state and `closedAt` timestamp (both
   available from the sub-issue enumeration / `gh issue view --json`).
2. For each **closed** child:
   a. Compute its **close-quarter** `Q` deterministically from `closedAt`
      (`Q = (month-1)//3 + 1`, in UTC — see 3.2).
   b. **Ensure** the archive epic `Epic (ad hoc): <repo> — <YYYY>-Q<n>` exists in
      the same home (create-if-missing by exact stamped title, labelled
      `epic`+`ad-hoc`; same idempotent create pattern as `ensure_adhoc_epic`).
   c. **Re-parent** the child into that archive: `add_child(archive, child)` then
      `remove_child(live, child)` — **skipping** any child already parented under
      its archive (idempotent no-op).
3. After draining, **close** any archive epic whose quarter is strictly earlier
   than the current quarter and is still open (it is done receiving). The current
   quarter's archive stays open.

**Nothing mutates the live epic except the loss of a re-parented child** — it is
never renamed, recreated, or closed. There is no rename window, no fresh-epic
creation for the live umbrella, and no open-child migration, so the crash-recovery
complexity of the superseded batch design does not exist here.

**Idempotency / convergence.** Every step is check-before-act: archives are
create-if-missing by exact title; a child already under its archive is skipped; a
past archive already closed is skipped. A run that dies partway is repaired by the
next run. A **late straggler** (an issue whose close-quarter archive was already
closed) is handled by `add_child` reopening a closed target (`epics.py:279`); the
step-3 close rule re-closes it on the same or next run. Ordering step 2c as
`add_child` then `remove_child` means a crash between the two leaves the child
momentarily under *both* parents (harmless, visible) rather than orphaned under
*neither*; the next run's skip check completes the removal.

### 3.2 Close-quarter bucketing — history is correct up front

Each closed issue is filed by **its own `closedAt` quarter**, not "the current
quarter." This makes the semantics *"the `YYYY-Qn` archive holds the ad-hoc work
finished in that quarter,"* and it means the **existing backlog self-distributes
correctly on first run**: the months of already-closed issues sitting in today's
ad-hoc epics are spread into their true historical quarter archives (created and,
being past quarters, immediately closed in the same run) with **no special-case
first-run logic**. Steady state and backfill are the same code path.

### 3.3 Archive lifecycle

- **Created on demand** the first time an issue closes in (or is discovered for)
  that quarter — never pre-created for empty quarters, so no empty archives.
- **Open while current**, receiving that quarter's closures.
- **Closed once its quarter has passed** (step 3), becoming a static historical
  record. Re-closed automatically if transiently reopened to accept a straggler.

### 3.4 CLI surface — `vrg-adhoc-epic archive`

A new `archive` subcommand on the existing `vrg-adhoc-epic` command (which already
has `ensure`):

```
vrg-adhoc-epic archive [--repo <owner>/<repo> | --all-in <org>] [--apply]
```

- **Dry-run by default:** prints the plan — for each live ad-hoc epic, which
  closed children move to which quarter archive, which archives will be created,
  and which past archives will be closed — with no mutations.
- `--apply` executes 3.1.
- `--repo` targets one repo; `--all-in <org>` sweeps the whole org: it enumerates
  the org's repos and resolves each one's ad-hoc-epic home via
  `resolve_epic_home`, so **public** repos (homed in `<org>/.github`) **and
  private** repos (self-homed) are both covered — a fixed `<org>/.github` target
  would silently miss private repos' ad-hoc epics.
- The subcommand is a thin wrapper over a shared `lib/epics.py` function so the
  identical logic backs the CLI, the event path, and the sweep (3.5).

### 3.5 Automation — reuse existing wiring, no new cron

The operation is incremental, so it folds into the epic system's existing
event+sweep architecture instead of adding a quarterly batch job:

- **Event path (steady state):** `issues.closed` already fires
  `vrg-epic-rollup --task <closed>` (`epic-rollup.yml` → `ops-epic-rollup.yml`).
  Extend `epics.rollup()` so that when the just-closed task's parent is a **live**
  ad-hoc epic (canonical title), it archives that one task (3.1 for a single
  child) instead of returning early. Re-parenting does not change issue state, so
  it triggers no further `issues.closed` event — no loop.
- **Daily sweep (backstop + backfill + archive-close):** the existing daily
  `epic-sweep` (`vrg-epic-audit --close`) is the backstop for missed close events
  elsewhere; it gains the org-wide drain (3.1 with `--all-in`), which also
  performs the one-time backlog distribution and the past-archive closing that no
  single event can trigger.
- **Manual / validation:** the `vrg-adhoc-epic archive` CLI (3.4), for on-demand
  previews and the dev-tree live validation (§4).

The exact wiring (whether the sweep gains a flag or an adjacent step) is settled
in the plan; the binding constraint is **reuse the existing crons and add no new
quarterly batch**. This is the same event-plus-sweep redundancy the rest of the
epic model already relies on.

### 3.6 Archive naming and labels

- **Archive title:** `Epic (ad hoc): <repo> — <YYYY>-Q<n>`, e.g.
  `Epic (ad hoc): vergil-tooling — 2026-Q3`. The retained `Epic (ad hoc): <repo>`
  root keeps the lineage obvious; `YYYY-Qn` sorts chronologically.
- **Resolver safety (belt and suspenders):** the appended suffix means the archive
  never exact-matches the canonical title, so `ensure_adhoc_epic`'s title filter
  (`epics.py:383`) skips it; and once closed, the `state:open` filter skips it too.
  While a current-quarter archive is *open*, the exact-title filter alone keeps it
  from being mistaken for the live epic (no false `>1` error).
- **Labels:** archives carry `epic`+`ad-hoc`. The `ad-hoc` label is deliberate:
  it makes `rollup()` skip them (`epics.py:470`), so adding an already-closed child
  never trips the finite-epic auto-close — the archive's closure is controlled
  solely by the step-3 quarter rule. No new label is provisioned.

### 3.7 Component boundaries — where each piece lands

| Component | Repo | Closing artifact |
|---|---|---|
| Shared drain logic in `lib/epics.py` + `rollup()` hook + `archive` subcommand + sweep hook + tests | vergil-tooling | same-repo PR(s) |
| Workflow wiring changes, **if any** (event/sweep already call the CLIs) | vergil-actions / `.github` | same-repo PR(s) only if a call/flag changes |
| spec + plan (this epic's docs) | vergil-project/.github | doc task PR (#239) |

The bulk lands in `vergil-tooling`. Because the automation reuses existing event
and sweep wiring, workflow-repo changes are minimal or none — confirmed at plan
time. Each implementation task is filed where its PR lands (the placement law);
cross-repo references are `Ref`, never `Closes`.

## 4. Testing

- **Unit** (`vergil-tooling`, existing faked-GitHub test pattern):
  - close-quarter computation from `closedAt` across all four quarters and the
    year boundary (Dec → Q4, Jan → Q1), UTC;
  - drain plan: only closed children move; each goes to its `closedAt`-quarter
    archive; open children stay in the live epic;
  - archive create-if-missing (not duplicated when present) and past-archive
    close rule (current-quarter archive stays open);
  - idempotency/convergence: re-running a partially-applied drain is a no-op; a
    child already under its archive is not re-parented; a straggler into a closed
    archive reopens-and-recloses;
  - visibility-aware home resolution for public vs private repos;
  - the `rollup()` event hook archives a single just-closed child under a live
    ad-hoc epic and no-ops when the parent is an archive or a finite epic.
- **Live validation** (operational task vergil-tooling#2677): from a
  `vergil-tooling` worktree, run the **dev tree** via
  `uv run vrg-adhoc-epic archive --apply` against a real ad-hoc epic once and
  verify end-to-end — closed children moved into their correct `closedAt`-quarter
  archives, past archives closed, current-quarter archive open, open children left
  in the live epic. Running from the dev tree exercises the real GitHub sub-issue
  GraphQL API (the check the faked-GitHub unit tests cannot prove) **without
  waiting on a release/`uv tool install`** (merged ≠ deployed). Blocked-by the
  implementation tasks.

## 5. Non-goals

- **Quarterly batch roll-up.** Superseded by continuous drain; there is no
  rename of the live epic and no bulk quarterly job.
- **Rolling up finite epics.** Finite epics already close via their retrospective;
  this touches only `ad-hoc`-labelled epics.
- **Deleting or compacting history.** Archives are kept forever (closed).
- **Cross-org behavior.** Each org has its own `.github`; the sweep runs per org.
- **Count/size-based triggering.** There is no threshold; closed work drains
  continuously regardless of volume.

## 6. Known tradeoffs

- **Unattended mutation.** The drain re-parents and closes issues with no human in
  the loop. Mitigated by dry-run-first previewing and strict idempotency; accepted
  as the same trust posture already granted to `epic-sweep`/`epic-rollup`.
- **Archive granularity is fixed at quarter.** A very active repo's quarter
  archive can still be sizeable, but it is bounded to one quarter of *finished*
  work and never mixes with in-flight work. Finer buckets (monthly) were not
  chosen — quarter matches how ad-hoc work is reasoned about and keeps archive
  count low (≤4/repo/year).
- **Momentary dual-parent on crash.** Ordering `add_child` before `remove_child`
  means a crash between them leaves a child under both the live epic and its
  archive until the next run — chosen deliberately over the orphan-under-neither
  alternative; it is visible and self-heals.

## 7. Edge cases and safety

- **Pre-existing ">1 canonical ad-hoc epic" state.** If resolution finds multiple
  open canonical ad-hoc epics for a repo (a corrupted state), the job reports it
  and skips that repo rather than guessing — it never compounds corruption.
- **Duplicate stamped archive.** `ensure` for an archive is create-if-missing by
  exact stamped title; if two archives already share a quarter title (corruption),
  report and skip that quarter.
- **Live epic emptied.** If every child of a live ad-hoc epic is closed, they all
  drain out and the live epic is simply left open and empty — correct; it remains
  the routing target for the next ad-hoc issue.
- **Token scope.** All mutations run under the org's App token, scoped to the home
  repo's owner, exactly as the existing epic Actions do.
