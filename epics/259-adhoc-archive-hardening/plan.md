# Ad-hoc archiving hardening — Implementation Plan

> REQUIRED SUB-SKILL: `superpowers:subagent-driven-development` or
> `superpowers:executing-plans` to implement task-by-task.

**Goal:** Kill the duplicate-archive race in the ad-hoc drain, then recover the
duplicates already created in production and finish the migration.

**Architecture:** One code change in `vergil-tooling/src/vergil_tooling/lib/epics.py`
(+ tests), followed by two operational tasks (deploy, recover). Sequenced
`code → deploy → recover`.

## Global Constraints

- **Line length 100** (ruff); **100% branch coverage** (`--cov-branch
  --cov-fail-under=100`); validation is the only gate
  (`vrg-container-run -- vrg-validate`).
- All git via `vrg-git`, commits via `vrg-commit`; work in the assigned worktree.

---

## Task 1 — Code: ensure-once-per-quarter + duplicate-tolerant resolver

**Kind:** code (PR). **Repo:** vergil-project/vergil-tooling.
**Files:** `src/vergil_tooling/lib/epics.py`, `tests/vergil_tooling/test_epics.py`.

**Interfaces:**
- `_find_epic_by_title(home, title, *, prefer_oldest: bool = False)` — when
  `prefer_oldest` and >1 open match, return the lowest-numbered instead of raising.
  Default `False` keeps the current raise (live-epic finders unchanged).
- `ensure_adhoc_archive(target_repo, quarter)` — calls `_find_epic_by_title(...,
  prefer_oldest=True)`, so a pre-existing duplicate archive is reused (oldest), not
  raised on.
- `apply_adhoc_drain(target_repo, plan)` — ensure each distinct quarter's archive
  **once**, cache by quarter, then reparent every child into the cached archive.

- [ ] **Step 1 — failing tests**
  - `test_find_epic_by_title_prefer_oldest_returns_lowest` — two open same-title
    rows → returns the lower number; without the flag it still raises.
  - `test_apply_drain_ensures_archive_once_per_quarter` — a plan with **N moves in
    the same quarter** calls `ensure_adhoc_archive` **once** (patch it, assert
    call count == 1) and `reparent_child` N times.
  - `test_apply_drain_two_quarters_ensures_twice` — moves spanning two quarters →
    `ensure_adhoc_archive` called once per distinct quarter.
  - `test_ensure_adhoc_archive_reuses_oldest_duplicate` — `read_json` returns two
    open archives of the target title → returns the lower number, does not create.

- [ ] **Step 2 — run, expect fail.**

- [ ] **Step 3 — implement**
  - Add the `prefer_oldest` param to `_find_epic_by_title`: on `len(rows) > 1`, if
    `prefer_oldest` return `min` by number, else raise as today.
  - `ensure_adhoc_archive` passes `prefer_oldest=True`.
  - In `apply_adhoc_drain`, replace the per-child `ensure_adhoc_archive` call with a
    per-quarter cache:
    ```python
    archives: dict[str, IssueRef] = {}
    for child, quarter in plan.moves:
        archive = archives.get(quarter)
        if archive is None:
            archive = ensure_adhoc_archive(target_repo, quarter)
            archives[quarter] = archive
        if archive == plan.live:
            continue
        reparent_child(archive, child)
    # close past archives unchanged
    ```

- [ ] **Step 4 — run, expect pass;** confirm existing drain/rollup tests still pass.

- [ ] **Step 5 — commit**
  `vrg-commit --type fix --scope epics --message "ensure ad-hoc archive once per
  quarter; tolerate duplicate archives (#<TASK>)"`.

---

## Task 2 — Deploy: release + install the fix

**Kind:** `deployment` (operational). **Repo:** vergil-project/vergil-tooling.
**Blocked-by:** Task 1.

- **Precondition (human-gated):** Task 1 merged to develop, **and** a release cut
  (bump/tag/publish) — the release is a human action, attested here, never
  performed by the agent.
- **Agent-safe step:** install the released tooling on the host
  (`uv tool install … @<tag>`), confirm `vrg-adhoc-epic archive --help` runs from
  the installed binary.
- **Why:** the daily `vrg-epic-audit --close` sweep runs the *released* tooling;
  releasing stops it re-creating duplicates. Record `Outcome: SUCCESS` on install.

---

## Task 3 — Recover: consolidate duplicates + re-drain + verify

**Kind:** `validation` (operational). **Repo:** vergil-project/vergil-tooling.
**Blocked-by:** Task 2.

**Procedure:**
1. **Consolidate.** For each `(repo, quarter)` with >1 open archive in
   `.github` (currently `#249`–`#258`): keeper = lowest-numbered; atomically
   `reparent_child(keeper, straggler)` for every child of the other archives; then
   close the emptied duplicates.
2. **Re-drain.** `vrg-adhoc-epic archive --all-in vergil-project --apply` (dev tree
   or installed — both now fixed) to migrate the ~46 remaining un-migrated closed
   children (vergil-tooling ≈40, vergil-containers ≈6).
3. **Verify (acceptance):** exactly **one** open archive per `(repo, quarter)`; no
   straggler children remain in any duplicate; each live ad-hoc epic holds only
   **open** children; a second `--apply` is a **no-op**.

Record `Outcome: SUCCESS`/`FAILURE` with evidence. On FAILURE, file follow-on
fix task(s) and leave open.

---

## Bookends (already seeded)

- #260 — this spec + plan (docs PR into `.github`).
- vergil-tooling#2697 — documentation-review sweep (runs before the retrospective).
- #261 — retrospective (terminal). Its §-notes should correct #238 §4's "benign"
  characterization of the list-consistency lag.
