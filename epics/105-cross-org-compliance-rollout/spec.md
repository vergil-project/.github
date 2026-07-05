# Complete the cross-org model-compliance rollout

- **Epic:** vergil-project/.github#105
- **Status:** Approved design (2026-07-05)
- **Docs task:** vergil-project/.github#106
- **Spawned by:** epic vergil-project/.github#85 (its ad-hoc migration was incomplete)

## 1. Summary

Bring the active orgs into full compliance with the epic/task model — all epics
in `.github`, no standing epics, a clean invariant audit — then retire the
`standing` alias. This is a **mostly-operational** epic: the design is a runbook
executed per org, with **near-zero new code**. The spec doubles as the plan.

## 2. Why (the gap)

The ad-hoc migration (epic #85, T10) relocated only **standing** epics in a
**single** org (`vergil-project`). It left behind:

- **Finite epics outside `.github`** (invariant 1 violations) — hand-rolled,
  pre-framework epics such as `vergil-project/vergil-tooling#2019`. Detected by
  `vrg-epic-audit`'s invariant check.
- **Other active orgs never migrated** — `logical-minds-foundry/.github#4`
  (a standing epic); the MQREST Admin Project org (slug to confirm).
- **Single-org tooling** — `vrg-adhoc-migrate` / `vrg-epic-audit` auto-detect the
  org from the current repo. Resolved by YAGNI (run from each org's checkout),
  **not** by adding `--org`.

## 3. Scope

**In scope (active orgs):** `vergil-project`, `logical-minds-foundry`, the MQREST
Admin Project org. **Out of scope:** Diogenes and the Nemesis Project — on
indefinite hold; their revival is the epic's follow-on (#111), not this work.

## 4. Approach — reuse, don't build (YAGNI)

- **No `--org` flag.** Run the existing tools from a local checkout of a repo in
  each org (a one-time `git clone` at most).
- **Standing epics** → the existing `vrg-adhoc-migrate` (kept deliberately; it
  will also serve the on-hold orgs when they are revived).
- **Finite epics** → **manual, case-by-case** relocation. Low-volume, each
  carries unique history. Mechanism per case: intra-org `gh issue transfer`
  (transfer is intra-org only, which is exactly our member-repo → `.github`
  case) *or* recreate-and-relink. **Only the parent (epic) moves; its children
  stay in their repos and are relinked** — the "parent and children move
  together" assumption is explicitly broken, and each relocation is verified.
- **The only code** is retiring the `standing` alias (T11 /
  `vergil-project/vergil-tooling#2129`), moved here from #85 as the final step.

## 5. Per-org rollout runbook

From a repo in each org, in order:

1. **Sync labels:** `vrg-ensure-label --repo <org>/.github --sync` — fix label
   drift (colours/descriptions), ensure `ad-hoc`/`epic`/intake labels are correct.
2. **Migrate standing epics:** `vrg-adhoc-migrate` (dry-run) → review →
   `vrg-adhoc-migrate --apply` (a human action).
3. **Relocate finite epics:** `vrg-epic-audit`; for each issue flagged under
   "Epics outside `.github`", relocate it manually (transfer or recreate-relink;
   relink children; verify).
4. **Confirm clean:** re-run `vrg-epic-audit` until the invariant-violations
   section is empty.

## 6. Tasks (under #105)

- **#106** — this spec/runbook (docs bookend).
- **#107** — sweep `vergil-project` (relocate `#2019`, verify clean; standing
  epics already migrated).
- **#108** — sweep `logical-minds-foundry` (full runbook; migrates `.github#4`).
- **#109** — sweep the MQREST Admin Project org (confirm slug; full runbook).
- **#110** — documentation review (closing bookend): the runbook + finite-epic
  procedure captured in the site docs (candidate: fold into the #2168 guide).
- **#111** — follow-on brainstorm (closing bookend): what comes next — likely
  reviving the on-hold orgs.
- **`vergil-project/vergil-tooling#2129`** — retire the `standing` alias (moved
  from #85); the **final** task, gated on all three orgs auditing clean.

## 7. Sequencing & dependencies

- Per org: labels → standing migration → finite relocation → audit clean.
- **T11 is last**, gated on all three orgs' audits being clean — retiring
  `standing` while any org still has a live standing epic would strand it. T11's
  partial implementation is preserved on branch `feature/2129-retire-standing`.
- Creating this epic **un-chained epic #85**: #85's #104 bookend closed on this
  epic's creation; T11 now rides here.

## 8. Out of scope

- An `--org` flag for the tooling (YAGNI; add later only if per-org sweeping
  becomes a routine, e.g. paired with the periodic-review epic #87).
- A finite-epic relocation tool (low-volume; manual).
- Reviving the on-hold orgs (Diogenes, Nemesis) — the follow-on (#111).
