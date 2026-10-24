# Every Issue Belongs to an Epic — Retrospective

**Epic:** vergil-project/.github#176 · **Spec:** [`spec.md`](spec.md) (revision 2) ·
**Plan:** [`plan.md`](plan.md) (revision 2) · **Retrospective task:** #432

## 0. At a glance

**What we set out to do.** Eliminate orphan issues. In any repo with a `vergil.toml`,
every issue is either a legitimate top-level kind (an epic, an archive, or a release
tracking issue) or the child of an epic. Spec revision 2 (#430) widened this into
the **intake → planning → ready** model:

- Unreviewed work lives under a standing intake epic.
- Epic creation is queued and tracked under a standing planning epic.
- A parent-based PR refusal keeps both queues un-buildable.
- A one-shot universal sweep makes the invariant true everywhere.

**What shipped.** All of it, in vergil-tooling `v2.1.260` → `v2.1.264` and
vergil-claude-plugin `v2.1.39`:

- Intake and planning standing epics, with per-quarter archives.
- Three new CLIs: `vrg-intake-create`, `vrg-intake-route` and `vrg-queue`.
- The queue PR refusal.
- An orphan pass in the daily `vrg-epic-audit --close` sweep.
- Rewritten skills: `intake-capture`, `intake-review`, and `epic-create` through planning items.
- Docs in four repos.

The migration ran in vergil-project and logical-minds-foundry: 91 retired labels
stripped, 88 orphans attached, and repeat runs changed nothing. The other three
in-use orgs carry a reminder task instead (§3).

| Task | Merged PR | Merged | What it did |
|---|---|---|---|
| .github#177 | .github#184 | 2026-07-23 | Spec + plan, revision 1 |
| .github#430 | .github#431 | 2026-10-23 | Spec + plan, revision 2: absorbed the intake/planning model from .github#429 |
| vergil-tooling#2481 (T1) | vergil-tooling#3476 | 2026-10-23 | `StandingKind` table, intake/planning epics, `queue_home`, the legitimate-top-level and managed-repo predicates, the `intake`/`planning` sentinels |
| vergil-tooling#3469 (T2) | vergil-tooling#3477 | 2026-10-23 | Generalized the ad-hoc archive drain to every standing kind |
| vergil-tooling#2482 (T3) | vergil-tooling#3503 | 2026-10-23 | `vrg-intake-create` (born-linked under intake); `vrg-triage-create` kept as a deprecated alias |
| vergil-tooling#2485 (T4) | vergil-tooling#3505 | 2026-10-23 | `report-ready` and `vrg-submit-pr` refuse intake and planning items, by parent |
| vergil-tooling#2483 (T5) | vergil-tooling#3504 | 2026-10-23 | Orphan pass in `vrg-epic-audit`: nativize, strip retired labels, move public queues, attach orphans; stops at the cap |
| vergil-tooling#3470 (T6) | vergil-tooling#3506 | 2026-10-23 | `vrg-intake-route`: drop, defer, one-off, planning and split, with a transfer → recreate fallback |
| vergil-tooling#3471 (T7) | vergil-tooling#3507 | 2026-10-23 | `vrg-queue intake\|planning` read-only views |
| vergil-claude-plugin#703 (T8) | vergil-claude-plugin#704 | 2026-10-23 | `intake-capture` / `intake-review` skills; `epic-create` works through planning items; every other skill renamed to match |
| vergil-tooling#3514 | vergil-tooling#3516 | 2026-10-24 | Fix: the `--epic intake\|planning` sentinels were rejected by a cross-org guard (§2, §3) |
| vergil-tooling#2474 (docs review) | vergil-tooling#3515 | 2026-10-24 | Site standards, CLI reference, visibility guide, CLAUDE.md |
| vergil-actions#1043 | vergil-actions#1044 | 2026-10-24 | `ops-epic-sweep` comments and site page cover the drain and orphan pass |
| docs#46 | docs#47 | 2026-10-24 | Management pages rewritten for intake → planning → ready |

**Non-PR closures:**

- **#3474 (deployment), SUCCESS 2026-10-24:** tooling and plugin installed on the VM and the Mac; the sweep runners verified on `v2.1` = `v2.1.263`.
- **#185 (validation), SUCCESS 2026-10-24:** the migration in both orgs, as above.
- **vergil-tooling#2484:** merged into #2482 by revision 2.
- **vergil-actions#786:** closed as not planned on 2026-10-14, with no closing comment. Spec revision 2 records that `ops-epic-sweep` "needs no change", because it already runs `vrg-epic-audit` with `VRG_EPIC_SWEEP=1`. The orphan pass (T5) rides that existing `--close`.

**By the numbers:**

| | |
|---|---|
| Repos touched | 5: .github, vergil-tooling, vergil-claude-plugin, vergil-actions, docs |
| Children | 19, including this retrospective: 14 PR-closed, 2 operational, 3 closed without a PR |
| Merged PRs | 14, plus this retrospective's |
| vergil-tooling releases | `v2.1.260` (T1 + T2), `v2.1.263` (T3–T7), `v2.1.264` (the fix and docs) |
| Plugin release | `v2.1.39` (T8) |
| Span | opened 2026-07-23 → closes 2026-10-24, about 3 months. Implementation itself took about 2 days, 2026-10-23 → 10-24. |

## 1. How the plan evolved

`plan.md` carries no "Evolution during execution" log, so this section comes from
the issue record and the execution session.

- **Revision 1 sat for three months, then was replaced before any work started.**
  The July plan (standing intake epic + resolver + client/server enforcement + a
  migration) never started. In October, the intake-review seed (.github#429)
  showed that intake also needed a *planning* stage and defined issue states.
  Revision 2 (#430) absorbed it on 2026-10-23:
  - surviving tasks kept their issue numbers;
  - #2484 merged into #2482;
  - the archive drain (T2), `vrg-intake-route` (T6) and `vrg-queue` (T7) were added;
  - the server-side layer is no longer the separate `issues.opened` safety-net
    Action that the epic issue's original design named. It is the orphan pass in
    the existing daily sweep, which needed no workflow change (#786 had already
    been closed as not planned).
- **Execution followed the plan's order, 1 → 2 → (3–7) → 8 → 9 → 10.**
  - T1 and T2 went one at a time.
  - T3–T7 ran as five parallel agents. Each was given the exact insertion points
    for the shared files (`pyproject.toml` scripts, `epics.py` near `queue_kind`).
    All five merged cleanly in any order, and the combined tree validated green
    (8318 tests) before handoff.
  - T8 followed the merge, and was written against the tools as actually built.
- **The deployment and validation steps happened in an unplanned order.** The plan
  assumed a human `workflow_dispatch` would run the migration as Task 10. Once
  `v2.1.263` was released, though, the scheduled sweep in logical-minds-foundry ran
  the migration unattended at 07:24 UTC, before #185 started. The vergil-project
  migration did follow the plan: preview first, then dispatch, after a deliberate
  decision to hold the release until the plugin shipped.
- **Scope added during execution.**
  - #3514, a defect in T1/T3's sentinel surface found during close-out. It was
    briefly filed under the ad-hoc epic, then moved back under this epic.
  - Two per-repo doc tasks spawned by the docs review: docs#46 and vergil-actions#1043.
- **Scope removed.**
  - #2842 (refuse epic- and archive-labelled issues where a task is expected) was
    moved to the vergil-tooling ad-hoc epic. It had been linked here on theme
    alone, and was never planned.
  - #178 (the follow-on label-set brainstorm) was moved into the new intake queue
    rather than kept as a closing gate.

## 2. Lessons learned

- **A plan that waits months should be re-aligned before execution, and here it was.**
  Revision 2 cost one docs PR and folded the server-side enforcement into the
  existing sweep instead of a separate `issues.opened` Action. Keep doing this: re-run alignment when a stale plan
  is picked up.
- **Parallel agents on one repo work when shared-file edits are pre-assigned.**
  Telling each agent where to insert in `pyproject.toml` and `epics.py`, and to
  never append at end-of-file, gave five frozen branches that merged in any order.
  A combined trial merge plus full validation before handoff caught nothing, which
  is what it is for. Do this every time parallel branches touch the same files.
- **Unit tests per module missed an end-to-end gap between CLI layers.**
  - T1 taught `resolve_epic_ref` the `intake`/`planning` sentinels. But
    `vrg-epic-move` and `vrg-issue-create` run a cross-org guard *before*
    resolving, and that guard exempted only `adhoc`.
  - Every unit test passed, and the T8 skill author wrote `--epic intake` into
    `migrate-repo` without running it.
  - It surfaced only when the sentinel was used for real (#3514). The fix derives
    the guard's list from the same `StandingKind` table.
  - Next time: when a skill names a CLI flag or value, run it once (read-only, or
    `--help`) before shipping the skill.
- **Releasing a sweep change is the same as running it.** Once a release contains
  new `--close` behaviour, every org's scheduled sweep executes it on its next
  cron tick. Plan the "preview first" gate *before* the release, not after it.
- **The plugin has two install scopes, and both need updating.** The deployment
  found the user-scope plugin stale at 2.1.39 (before #704). In-project `/plugin`
  refused to update it because the repo's `.claude/settings.json` declares the
  marketplace with `ref: main` while user settings do not. Updating from `~` worked.
- **A published release is not necessarily a promoted release.**
  - `vrg-release` publishes the tag in `confirm-main`, but moves the floating
    `v2.1` tag, the apt index and update-deps only *after* `back-merge-bump`.
  - When GitHub's code-scanning processing stalled on the back-merge PR (the
    `CodeQL` check sat QUEUED for 30+ minutes with nothing queued in Actions),
    `v2.1.264` existed but nothing that consumes `v2.1` or apt got it until
    `--resume`.
  - Worth knowing when "released" is the precondition of a deploy.

## 3. Compromises & tradeoffs

- **Not every org is migrated.** vergil-project and logical-minds-foundry are
  migrated and validated. mnemosys-project and diogenes-project have no sweep
  caller, and mq-rest-admin-project's sweep has not run since 2026-09-01. Each
  carries an ad-hoc reminder task to add or repair the sweep and run
  preview → dispatch → validate:
  - mnemosys-project/.github#101
  - diogenes-project/.github#21
  - mq-rest-admin-project/.github#44

  The spec said "universal". In practice it is universal for the orgs in use,
  with the rest deferred deliberately. vergils-nemesis is not yet in use and was
  excluded.
- **The PR refusal checks the native parent only (T4).** An intake item linked
  only by a body `Parent:` line would not be refused until the sweep converts it
  to a native link. The migration did that conversion (0 were found), so the gap
  is theoretical, but it exists.
- **`vrg-queue` exits 1 when the queue epic does not exist yet**, rather than
  reporting an empty queue. This is the strict reading of "no silent failures".
  The skills treat it as "no queue yet".
- **`vrg-intake-route` assumes `gh issue transfer` prints the new URL.** No test
  checks this against live GitHub. If the URL is missing, the tool stops and asks
  for a manual link rather than recreating the issue (which would duplicate it).
- **The cap of 95 is hardcoded in docs.** The vergil-actions and tooling site
  docs state "95", while the code reads `_ADHOC_ARCHIVE_SEGMENT_CAP`. The docs go
  stale if the constant changes.
- **`labels.json` descriptions lag the docs.** `archive` still says "closed ad-hoc
  work", and `research` keeps its old description. They were left for the
  label-set review (#178).
- **`vrg-triage-create` stays as a deprecated alias** for one release, so callers
  keep working while they move off it.
- **Process slips.**
  - One agent ran `ruff format` directly. Repo policy is to run tools only through
    `vrg-validate`, which was green afterwards.
  - The skill docs avoid the hyphenated old skill names in order to pass the
    plan's grep, rather than because the grep's intent required it.

## 4. New problems & opportunities

| Surfaced | Where it went |
|---|---|
| Agent-identity `vrg-epic-audit` preview silently misses private repos (found with logical-minds-foundry `mq-gateway-replay-lab`) | vergil-project/vergil-tooling#3513 |
| `--epic intake\|planning` sentinels rejected by the cross-org guards | vergil-project/vergil-tooling#3514, fixed in `v2.1.264` |
| `--epic` help text did not list the sentinels | #3512, absorbed into #3514 |
| Tools accept epic- or archive-labelled issues where a task is expected | vergil-project/vergil-tooling#2842 (ad-hoc epic) |
| Private-repo work under a public `.github` epic (`mq-gateway-replay-lab#305`–`#307`, under logical-minds-foundry/.github#245) | Report-only by design; resolve by hand. Logged, not yet acted on. |
| Validation tasks closed without a PASS comment: 15 in vergil-project, 8 in logical-minds-foundry (all pre-existing) | Logged, not yet acted on |
| `vrg-queue intake --json` returns `"epic": null` for intake items | Logged, not yet acted on |
| `.github`'s `epic-sweep.yml` header comment still describes only drift closing | Logged, not yet acted on (judged too small for a task) |
| Marketplace `ref: main` (project) vs no ref (user) blocks in-project `/plugin` updates | Logged, not yet acted on |

## 5. What's next

- **Follow-on brainstorm: review the wider label set.** vergil-project/.github#178,
  now in the intake queue. The `labels.json` description drift (§3) belongs there.
- **Bring the remaining orgs onto the model:** mnemosys-project/.github#101,
  diogenes-project/.github#21, mq-rest-admin-project/.github#44.
- **First real intake-review pass.** The vergil-project queue holds 51 items,
  oldest from 2026-07; logical-minds-foundry holds 34.

## Appendix A — Operational notes

1. **Releases.**
   - `v2.1.260` carried T1 + T2 together. T2 had to ship with T1 so closed
     intake/planning children drain.
   - `v2.1.263` carried T3–T7, held until the plugin (T8, `v2.1.39`) was ready,
     so the sweep would not migrate under the old triage skills.
   - `v2.1.264` carried #3514 and the docs. Its `back-merge-bump` stalled on
     GitHub code-scanning processing, and `vrg-release --resume` completed the
     promote of `v2.1`, the package index and update-deps.
2. **Deployment (#3474).**
   - The VM was already on package `2.1.263-1`; the human ran the three `--help`
     probes on the Mac.
   - The user-scope plugin was updated with
     `claude plugin update vergil@vergil-marketplace --scope user`, run from `~`.
   - The sweep runners were verified statically: `.github` pins `v2.1`, and
     `v2.1` == `v2.1.263`.
3. **Migration (#185).**
   - **logical-minds-foundry:** its scheduled sweep (07:24 UTC, run 40073983117)
     migrated first: 36 labels stripped, 38 orphans attached.
   - **vergil-project:** read-only preview (50 orphans, 41 labels) → dispatch
     (run 40079544994) matched exactly → post-preview all zeros.
   - **Repeat runs:** dispatches 40079587409 (logical-minds-foundry) and
     40080201607 (vergil-project) changed nothing.
   - **Checks:** no full-epic stop. All 13 open archives are still unparented, and
     no release tracking issue was open.
4. **Gotchas.**
   - The `user` identity cannot `gh api`, `workflow run` or `run rerun`. Human
     steps were the dispatches, the CodeQL rerun and `/plugin`.
   - `vrg-git` has no `merge-tree` or `-c`. Trial merges were done in a scratch
     clone with `GIT_AUTHOR_*` / `GIT_COMMITTER_*` env vars.
   - A branch touching `.github/workflows/**` cannot be pushed by the agent; run
     `vrg-submit-pr` from its worktree.
