# Quarterly roll-up of ad-hoc epics — time-bound the perpetual umbrella

- **Epic:** vergil-project/.github#238
- **Status:** Reviewed design (2026-08-08) — post-pushback (4 findings, all resolved)
- **Brainstorm source:** superpowers brainstorming session, 2026-08-08
- **Implemented in:** vergil-tooling (CLI), vergil-actions (reusable workflow),
  vergil-project/.github (scheduled caller) — tasks filed there at
  implementation time

## 1. Summary

Ad-hoc epics (`Epic (ad hoc): <repo>`, labelled `epic`+`ad-hoc`, homed in
`<org>/.github`) are **perpetual umbrellas**: `vrg-issue-create --epic adhoc`
routes small, unrelated work to them, and they are reused idempotently and
**never closed** (`epics.rollup()` explicitly skips any `ad-hoc`-labelled epic).
Finite epics are time-bounded — they close when their work and retrospective are
done — but ad-hoc epics have no closing mechanism at all, so they accumulate
without bound and become large, unmanageable lists of mostly-closed issues.

This epic time-bounds them with a **quarterly roll-up**. Once a quarter, per org,
each ad-hoc epic that has completed work is rolled up: the accumulated epic
becomes a **closed, period-stamped archive** of that quarter's finished work, and
a **fresh** `Epic (ad hoc): <repo>` takes over as the live routing target. Still-
open children carry forward into the fresh epic. History is preserved (nothing is
deleted); the live umbrella is bounded to roughly one quarter of clutter.

The design deliberately builds on what already exists: re-parenting reuses the
native sub-issue `add_child`/`remove_child` primitives (as `vrg-epic-move` and
`vrg-adhoc-migrate` do), and the scheduled automation copies the proven
`epic-sweep` three-layer shape (thin caller in `.github` → reusable workflow in
`vergil-actions` → `vrg-*` CLI, run under the App identity).

## 2. The problem in detail

The ad-hoc epic is resolved by **search, not a stored ref**
(`epics.ensure_adhoc_epic`): list open issues in the home repo with labels
`epic`+`ad-hoc`, then keep the row whose title exactly equals
`Epic (ad hoc): <bare-repo-name>`. Zero rows → create; one → reuse; more than one
→ error. Because it is a perpetual umbrella, `rollup()` returns early for it and
it never auto-closes.

The consequence is unbounded growth. Over months a single ad-hoc epic collects
dozens to hundreds of small closed issues plus a handful of open ones. The
signal (what is *currently* in flight) drowns in the archive (what was finished
long ago), and the epic stops being a useful lightweight routing target.

There is no existing time-bounding, rename, archive, or roll-up mechanism for
ad-hoc epics — this behavior is genuinely new.

## 3. Design

### 3.1 The roll-up algorithm (idempotent, per ad-hoc epic)

The driver is **resume-first**: it completes any roll left half-finished by a
prior crash *before* it starts a new one. Both phases run per repo.

**Phase A — resume any in-progress roll.** Find **open**, period-stamped archives
(`Epic (ad hoc): <repo> — <YYYY>-Q<n>`, still open, `epic`+`ad-hoc`) — an open
period-stamped archive *is* the signal of an unfinished roll, since a completed
archive is closed. For each: ensure a fresh `Epic (ad hoc): <repo>` exists,
re-parent its still-open children into that fresh epic, comment the pointer, and
close it. This is what makes a crash between rename and close self-healing.

**Phase B — start a new roll (per live ad-hoc epic).**
1. **Resolve** the live ad-hoc epic (`Epic (ad hoc): <repo>`, `epic`+`ad-hoc`,
   open).
2. **Skip** if it has **0 closed children** (nothing to archive — see 3.2).
3. **Rename** it to the period-stamped archive title (see 3.5), where the period
   is the **quarter just completed** relative to the run date — **unless** an
   archive with that exact stamped title already exists, in which case that
   quarter was already rolled: do **not** create a duplicate, and defer to Phase A
   to verify/finish it (this also makes a second same-quarter run a no-op).
4. **Create** a fresh `Epic (ad hoc): <repo>` — **only if** the canonical title
   is now absent (the idempotency check; a fresh epic already present means a
   prior partial run created it).
5. **Re-parent** every still-**open** child from the archive into the fresh epic
   via `remove_child(archive, task)` + `add_child(fresh, task)` — **skipping** any
   child already parented under the fresh epic.
6. **Comment** a pointer on the archive (`rolled up to #<fresh>`) and **close**
   it.

**Invariant:** every step is **check-before-act**, and Phase A completes any
partial roll, so a run that dies at *any* intermediate state (renamed-but-no-
successor, successor-present-but-children-unmigrated, migrated-but-not-closed) is
repaired by simply re-running — the next quarterly tick or a manual
`workflow_dispatch` finishes it. No step assumes the previous one ran in the same
invocation.

### 3.2 Skip rule — never create an empty archive

Cadence is **purely quarterly**; the skip rule is a guard, not a count-based
trigger. An ad-hoc epic is rolled **only when it has ≥1 closed child to leave
behind**:

| Children | Action |
|---|---|
| 40 closed + 0 open | roll (archive 40; fresh starts empty) |
| 5 closed + 3 open | roll (archive 5; migrate 3 open) |
| 0 closed + 3 open | **skip** (nothing finished to archive) |
| empty | **skip** |

This keeps roll-up from *creating* the clutter it exists to remove.

### 3.3 CLI surface — `vrg-adhoc-epic rollup`

A new `rollup` subcommand on the existing `vrg-adhoc-epic` command (which already
has `ensure`):

```
vrg-adhoc-epic rollup [--repo <owner>/<repo> | --all-in <org>/.github] [--apply]
```

- **Dry-run by default:** prints the exact plan — for each epic, the archive
  title it will take, the count and refs of open children that will migrate, and
  what will be closed — and makes **no** mutations.
- `--apply` executes the algorithm in 3.1.
- `--repo` targets one repo's ad-hoc epic; `--all-in <org>` sweeps the whole org
  (the scheduled path): it enumerates the org's repos and resolves each one's
  ad-hoc-epic home via `resolve_epic_home`, so **public** repos (homed in
  `<org>/.github`) **and private** repos (self-homed in themselves) are both
  covered — a fixed `<org>/.github` target would silently miss private repos'
  ad-hoc epics.
- Reuses `lib/epics.py` primitives; `lib/adhoc_migrate.py` (find → plan → render →
  human-gated apply → re-parent → close-with-pointer) is the structural template,
  minus its human-only gate (see 3.4).

### 3.4 Scheduled automation (three-layer, mirroring `epic-sweep`)

- **Thin caller — `vergil-project/.github`:**
  `.github/workflows/adhoc-rollup.yml`, `on: schedule` with a quarterly cron
  (`0 7 1 1,4,7,10 *` — 07:00 UTC on the first of Jan/Apr/Jul/Oct) plus
  `workflow_dispatch`. Delegates to the reusable workflow, passing the org.
- **Reusable workflow — `vergil-actions`:**
  `.github/workflows/ops-adhoc-rollup.yml@v2.1`, runs
  `vrg-adhoc-epic rollup --all-in <org> --apply` under the App identity
  (`APP_CLIENT_ID`/`APP_PRIVATE_KEY`), exactly as `ops-epic-sweep` runs
  `vrg-epic-audit --close`.
- **CLI — `vergil-tooling`:** the subcommand above.

**Posture:** unattended `--apply`. Safety comes from **idempotency** (3.1), not a
human gate — a partial run is re-runnable and the quarterly cadence would make a
human approval step a frequent forgotten blocker. The dry-run mode gives humans
and CI a preview on demand (`workflow_dispatch` of the caller, or the CLI
locally). This intentionally follows the `epic-sweep` precedent (routine
unattended maintenance) rather than the `vrg-adhoc-migrate --apply` precedent (a
one-shot, human-only relocation).

### 3.5 Archive naming and labels

- **Archive title:** `Epic (ad hoc): <repo> — <YYYY>-Q<n>`, e.g.
  `Epic (ad hoc): vergil-tooling — 2026-Q3`. The retained `Epic (ad hoc): <repo>`
  root keeps the lineage obvious; the appended `YYYY-Qn` suffix sorts
  chronologically and is unambiguous.
- **Resolver safety (belt and suspenders):** the appended suffix means the
  archive no longer exact-matches the canonical title, so `ensure_adhoc_epic`
  skips it; and the archive is **closed**, so the `state:open` filter skips it
  too. Either alone is sufficient.
- **Labels:** the archive **keeps** its `epic`+`ad-hoc` labels, so it stays
  discoverable as ad-hoc history; the closed state plus the period-stamped title
  distinguish it from the live epic. No new label is provisioned.

### 3.6 Component boundaries — where each piece lands

| Component | Repo | Closing artifact |
|---|---|---|
| `rollup` subcommand + plan/apply logic + tests | vergil-tooling | same-repo PR(s) |
| `ops-adhoc-rollup.yml` reusable workflow | vergil-actions | same-repo PR |
| `adhoc-rollup.yml` scheduled thin caller | vergil-project/.github | same-repo PR |
| spec + plan (this epic's docs) | vergil-project/.github | doc task PR (#239) |

Each implementation task is filed in the repo where its PR lands (the placement
law); cross-repo references are `Ref`, never `Closes`.

## 4. Testing

- **Unit** (`vergil-tooling`, existing faked-GitHub test pattern): plan
  construction (skip rule, archive title stamping across quarter boundaries,
  open-vs-closed partition, visibility-aware home resolution for public vs
  private repos); apply idempotency (a fresh epic already present is not
  duplicated; an already-migrated child is not re-parented; a pre-existing
  same-quarter archive is not duplicated); and **resume from each intermediate
  crash state** — renamed-but-no-successor, successor-present-but-children-
  unmigrated, migrated-but-not-closed — each converging to the correct end state
  (Phase A).
- **Live validation** (operational task vergil-tooling#2677): from a
  `vergil-tooling` worktree, run the **dev tree** via
  `uv run vrg-adhoc-epic rollup --apply` against a real ad-hoc epic once and
  verify the end-to-end outcome — archive renamed+closed with a pointer comment,
  fresh canonical epic present, open children re-parented, closed children left
  behind. Running from the dev tree (not the installed binary) exercises the real
  GitHub sub-issue GraphQL API — the check the faked-GitHub unit tests cannot
  prove — **without waiting on a release/`uv tool install`** (merged ≠ deployed).
  Blocked-by the implementation tasks.

## 5. Non-goals

- **Count-based or size-based triggering.** Cadence is purely quarterly; the only
  count consulted is the ≥1-closed-child skip guard. A count-based safety valve
  (roll early if an epic blows up mid-quarter) is explicitly deferred (YAGNI).
- **Rolling up finite epics.** Finite epics already close via their
  retrospective; this touches only `ad-hoc`-labelled epics.
- **Deleting or compacting history.** Archives are closed and kept forever.
- **Cross-org behavior.** Each org has its own `.github`; the sweep runs once per
  org over that org's ad-hoc epics.

## 6. Known tradeoffs

- **Unattended mutation.** The job renames, creates, re-parents, and closes
  issues with no human in the loop. Mitigated by dry-run-first previewing and
  strict idempotency; accepted as the same trust posture already granted to
  `epic-sweep`.
- **Quarter alignment on first run.** The very first roll archives an epic that
  accumulated work across more than one quarter; it is stamped with the quarter
  just completed. This is a one-time cosmetic imprecision, not a correctness
  issue — the archive is simply "everything finished as of `YYYY-Qn`".
- **Quarterly, not monthly.** A quarter of small work can still be sizeable;
  monthly was rejected because it would produce 12 archives/repo/year and defeat
  the decluttering goal. Quarterly (4/repo/year) balances archive count against
  live-epic size.

## 7. Edge cases and safety

- **Pre-existing ">1 ad-hoc epic" state.** If resolution already finds multiple
  open canonical ad-hoc epics for a repo (a corrupted state), the job reports it
  and skips that repo rather than guessing — it never compounds the corruption.
- **Already rolled this quarter.** If a fresh canonical epic exists and the only
  candidate archive is already period-stamped/closed, the run is a no-op for that
  repo.
- **Token scope.** All mutations run under the org's App token, scoped to the
  home repo's owner, exactly as the existing epic Actions do.
