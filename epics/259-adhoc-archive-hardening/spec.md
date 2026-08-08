# Ad-hoc archiving hardening — duplicate-archive race fix + backlog recovery

- **Epic:** vergil-project/.github#259
- **Status:** Design (2026-08-08)
- **Part one:** vergil-project/.github#238 (closed) — shipped continuous ad-hoc
  archiving; this epic hardens it.
- **Implemented in:** vergil-tooling (code) + a one-time operational recovery of
  production state.

## 1. Summary

Continuous ad-hoc archiving (#238) drains closed children of a live ad-hoc epic
into per-quarter archive epics. On the first production
`vrg-adhoc-epic archive --all-in vergil-project --apply` run it created
**duplicate same-quarter archives** and skipped two repos. This epic fixes the
race in code and recovers the duplicates already created.

## 2. Root cause

`apply_adhoc_drain` calls `ensure_adhoc_archive(repo, quarter)` **once per closed
child** (`src/vergil_tooling/lib/epics.py`). `ensure_adhoc_archive` is
find-or-create by title; the "find" (`gh issue list`) index **lags a few seconds
behind issue creation**. In a tight per-child loop, same-quarter children each
fail to see the archive a prior child just created and mint their own. Once ≥2
exist, `_find_epic_by_title` raises on the ambiguity and the whole repo is
skipped.

Evidence from the run: vergil-actions (3 closed children) → **3 archives, one
child each** (`.github#254/#255/#256`); vergil-tooling and vergil-containers were
skipped after making 2–3 duplicates each; ~46 closed children remain un-migrated.

This is precisely the list-consistency lag flagged in #238's retrospective §4 and
mis-rated there as "benign" — within a single drain, with many same-quarter
children, it is not.

## 3. Design

### 3.1 Code — kill the race (vergil-tooling)

- **Ensure once per `(repo, quarter)` per drain.** In `apply_adhoc_drain`, resolve
  the distinct quarters needed from `plan.moves` up front, call
  `ensure_adhoc_archive` **once per quarter**, cache the result, and reparent every
  child into the cached archive. The archive is created at most once per quarter,
  so there is no per-child create window for the lag to exploit.
- **Make `ensure_adhoc_archive` duplicate-tolerant (archive resolution only).**
  When resolving an **archive** title and more than one open match exists, **return
  the lowest-numbered (oldest)** rather than raising, so the drain self-heals: the
  skipped repos un-stick on the next run, and any future lag-induced duplicate
  degrades to "reuse the oldest" instead of a hard skip. Scope this to archive
  resolution via a `prefer_oldest` flag that `ensure_adhoc_archive` passes; the
  **live** ad-hoc epic finder (`find_adhoc_epic` / `ensure_adhoc_epic`) must keep
  **raising** on duplicate live epics — two `Epic (ad hoc): <repo>` is real
  corruption, not a benign race, and must not be silently masked.

### 3.2 Operational — recover current production state

- **Consolidate duplicates** (`.github#249`–`#258`): for each `(repo, quarter)`
  with more than one archive, pick the lowest-numbered as **keeper**, atomically
  `replaceParent` every straggler child into the keeper, then close the emptied
  duplicates.
- **Re-drain the remainder** against the **deployed** fix: run
  `vrg-adhoc-epic archive --all-in vergil-project --apply` to migrate the ~46
  still-un-migrated closed children (vergil-tooling ≈40, vergil-containers ≈6).
- **Verify:** exactly one open archive per `(repo, quarter)`; live epics hold only
  open children; a second `--apply` is a no-op.

## 4. Sequencing

`code fix (impl) → release + install (deploy, human-gated) → operational
recovery`. The recovery re-drain must run against the **fixed** code, or it will
manufacture fresh duplicates exactly as before. **Release matters independently:**
the daily `vrg-epic-audit --close` sweep runs the *released* tooling, so until the
fix ships that automation will keep re-creating duplicates on repos with many
same-quarter closed children. Releasing promptly stops the bleeding; the recovery
then cleans up and completes the migration.

## 5. Acceptance

- After the fix, a drain with N same-quarter children creates **one** archive.
- Duplicate-tolerant resolver reuses the oldest archive instead of raising.
- Post-recovery: one open archive per `(repo, quarter)`; no straggler children left
  in duplicate archives; live epics fully drained of closed children; idempotent
  re-run.

## 6. Out of scope

- The broader "every non-epic issue belongs to an epic" / orphan-recovery
  invariant (#238 §4).
- A sanctioned test-fixture/sandbox teardown path for live validations (#238
  §4/§5).
- Cross-drain / concurrent-process races beyond the single-drain fix (the daily
  sweep is serialized; the event hook processes one child) — the duplicate-tolerant
  resolver already degrades these gracefully.
