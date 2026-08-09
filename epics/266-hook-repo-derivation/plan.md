# Event-hook repo derivation — Implementation Plan

> REQUIRED SUB-SKILL: `superpowers:subagent-driven-development` or
> `superpowers:executing-plans`.

**Goal:** Make the ad-hoc archiving event hook archive into the correct per-repo
bucket and ignore non-repo special epics, then recover Vergil's already-misfiled
children.

## Global Constraints

- Line length 100 (ruff); 100% branch coverage; validation is the only gate
  (`vrg-container-run -- vrg-validate`). All git via `vrg-git`, commits via
  `vrg-commit`; work in the assigned worktree.

---

## Task 1 — Code: derive archive repo from title + canonical guard

**Kind:** code (PR). **Repo:** vergil-project/vergil-tooling.
**Files:** `src/vergil_tooling/lib/epics.py`, `tests/vergil_tooling/test_epics.py`.

**Behavior:** In `rollup()`'s ad-hoc branch, replace the `parent.repo`-based target
with the epic's own bare name, and drain only for canonical per-repo epics.

- [ ] **Step 1 — failing tests** (`tests/vergil_tooling/test_epics.py`):
  - `test_rollup_public_repo_epic_archives_into_own_bucket` — parent title
    `Epic (ad hoc): some-repo` living in `.github`; `some-repo` is a real repo →
    `ensure_adhoc_archive` called with `"<owner>/some-repo"` (not `.github`).
  - `test_rollup_dotgithub_own_epic_unchanged` — parent title
    `Epic (ad hoc): .github` → target `"<owner>/.github"` (regression guard for the
    one case the old test covered).
  - `test_rollup_skips_non_repo_adhoc_epic` — parent title
    `Epic (ad hoc): Lab lifecycle reliability — cold rebuild …` (bare is not a
    repo) → `ensure_adhoc_archive` / `reparent_child` **not called**.

- [ ] **Step 2 — run, expect fail.**

- [ ] **Step 3 — implement.** Parse `<bare>` from the parent title with the
  existing `_ADHOC_EPIC_TITLE_PREFIX` (strip prefix; for the non-archive live
  title the whole remainder is the bare). Resolve the target as `<owner>/<bare>`.
  Guard: drain only if `<bare>` is a real repo — reject fast if it contains a space
  or `/`, then confirm via `github.list_org_repos(owner)` (or a repo lookup). If not
  a real repo, `return` (skip). Keep the stamped-archive early-return and the
  `if archive != parent` guard.
  - Note: prefer resolving `<bare>` from the **title**, not `parent.repo`. The
    `.github`-own epic still yields bare `.github` → target `<owner>/.github`, so
    that path is unchanged.

- [ ] **Step 4 — run, expect pass;** confirm existing rollup/drain tests still pass
  (the old `.github`-own hook test should continue to pass unchanged).

- [ ] **Step 5 — commit** `vrg-commit --type fix --scope epics --message "derive
  ad-hoc archive repo from epic title; skip non-repo epics (#<TASK>)"`.

---

## Task 2 — Deploy: release + install

**Kind:** `deployment`. **Repo:** vergil-project/vergil-tooling. **Blocked-by:** T1.

- Precondition (human-gated): T1 merged + a release cut. Agent-safe: install the
  released tag; confirm the hook code is present in the installed binary. The event
  hook runs the released tooling, so releasing stops further misfiling. Record
  `Outcome: SUCCESS`.

---

## Task 3 — Recover: re-file Vergil's misfiled children

**Kind:** `validation`. **Repo:** vergil-project/vergil-tooling. **Blocked-by:** T2.

**Procedure:**
1. For each `Epic (ad hoc): .github — <quarter>` archive in `vergil-project/.github`
   holding a **non-`.github`** child, atomically `reparent_child` that child into
   the correct `Epic (ad hoc): <child-repo> — <quarter>` archive (create-if-missing
   via the fixed tooling / `ensure_adhoc_archive`). Currently: `#1356`, `#1371`
   (both `vergil-tooling`, 2026-Q3) → `Epic (ad hoc): vergil-tooling — 2026-Q3`
   (`#249`). Close the emptied `.github` archives (`#263`, `#264`).
2. **Verify (acceptance):** no `Epic (ad hoc): .github — <quarter>` archive holds a
   non-`.github` child; the re-filed children sit in their per-repo archive; a
   subsequent close of an ad-hoc child lands in the correct per-repo bucket.

Record `Outcome`; on FAILURE file follow-on fix task(s) and leave open.

---

## Bookends (already seeded)

- #267 — this spec + plan.
- vergil-tooling#2706 — documentation-review sweep.
- #268 — retrospective (terminal).

## Cross-org follow-on (NOT part of this epic)

Tracked separately in `logical-minds-foundry/.github` after this ships: deploy the
fix; recover `#180/#181/#182`; reclassify `#104`; confirm private-repo token
visibility.
