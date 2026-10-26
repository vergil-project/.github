# Multi-series Python Runtime — Retrospective

Epic: [vergil-project/.github#472](https://github.com/vergil-project/.github/issues/472) ·
Spec: [`spec.md`](spec.md) · Plan: [`plan.md`](plan.md)

## 0. At a glance

**What we set out to do.** Keep every listed Python runtime series patched. vergil-python had
been building only `vergil-python3.15`, so `vergil-python3.14` was frozen at 3.14.8. The epic
set out to make vergil-python build one `vergil-python<x.y>` package per series in an explicit
list, refresh each series automatically and add a new leading-edge series when the version
window has one, and publish every series on every release. That needed multi-package support
in vergil-tooling first.

**What shipped.** All of it. vergil-python v1.0.4 publishes `vergil-python3.14
3.14.9+20261021.1.0.4` and `vergil-python3.15 3.15.1+20261021.1.0.4` side by side. A nightly
`vrg-upstream-refresh` run keeps them patched, and vergil-containers' tool bumps now run through
the same command. The acceptance target, 3.14 at 3.14.9 or later, is met. Adding or removing a
series is one reviewed change to `runtime.toml` and `vergil.toml`.

| Repo | PR | Task | What it did |
|---|---|---|---|
| .github | [#479](https://github.com/vergil-project/.github/pull/479) | #473 | Spec and plan |
| vergil-tooling | [#3636](https://github.com/vergil-project/vergil-tooling/pull/3636) | #3629 (T1) | Accept `[[package]]` entries: several packages per repo |
| vergil-tooling | [#3637](https://github.com/vergil-project/vergil-tooling/pull/3637) | #3630 (T6) | `vrg-upstream-refresh` command and `[upstream-refresh]` config |
| vergil-tooling | [#3638](https://github.com/vergil-project/vergil-tooling/pull/3638) | #3631 (T2) | One `package_version` helper and the `upstream-version` key |
| vergil-tooling | [#3639](https://github.com/vergil-project/vergil-tooling/pull/3639) | #3632 (T5) | Runtime pins read from the packages list, one pin per repo |
| vergil-tooling | [#3640](https://github.com/vergil-project/vergil-tooling/pull/3640) | #3633 (T3) | Package dimension in matrix, build, install-test and `check-runtime` |
| vergil-tooling | [#3642](https://github.com/vergil-project/vergil-tooling/pull/3642) | #3634 (T4) | Index matching by exact name; release waits on every package |
| vergil-actions | [#1063](https://github.com/vergil-project/vergil-actions/pull/1063) | #1057 (A1) | Reusable `ops-upstream-refresh.yml` |
| packages | [#29](https://github.com/vergil-project/packages/pull/29) | #28 (K1) | Index retention `keep = 1`, with a size projection |
| vergil-python | [#56](https://github.com/vergil-project/vergil-python/pull/56) | #51 (P1) | Series list, per-series build and per-series refresh hook |
| vergil-python | [#57](https://github.com/vergil-project/vergil-python/pull/57) | #52 (P2) | Add a leading-edge series from the version window |
| vergil-python | [#58](https://github.com/vergil-project/vergil-python/pull/58) | #53 (P3) | Nightly `upstream-refresh` job in `ops.yml` |
| vergil-containers | [#712](https://github.com/vergil-project/vergil-containers/pull/712) | #693 (C1) | `bump-tools` on `vrg-upstream-refresh` |
| vergil-containers | [#714](https://github.com/vergil-project/vergil-containers/pull/714) | #713 *(added)* | Refresh moved from `bump-tools.yml` into `ops.yml` |
| vergil-tooling | [#3652](https://github.com/vergil-project/vergil-tooling/pull/3652) | #3623 | Documentation review: site docs, `CLAUDE.md` and README |
| vergil-containers | [#718](https://github.com/vergil-project/vergil-containers/pull/718) | #717 *(spawned)* | Python language page describes multi-series |
| packages | [#32](https://github.com/vergil-project/packages/pull/32) | #31 *(spawned)* | README lists the per-series runtimes and `keep = 1` |
| docs | [#56](https://github.com/vergil-project/docs/pull/56) | #55 *(spawned)* | Runtime version format with the release suffix |
| vergil-actions | [#1070](https://github.com/vergil-project/vergil-actions/pull/1070) | #1069 *(spawned)* | Release-packages docs: one runtime per package |

- **Repos touched (7):** .github, vergil-tooling, vergil-actions, packages, vergil-python,
  vergil-containers, docs.
- **Children (24):**
  - 19 PR-closed tasks: the 18 in the table plus this retrospective.
  - 3 deployments: vergil-tooling#3635, vergil-actions#1059 and vergil-python#54.
  - 1 validation: vergil-python#55.
  - 1 spec/plan docs task: .github#473.
  - Of these, 16 were planned as rows (T1–T6, A1, K1, D1–D3, P1–P3, C1, V1). The other 5 were
    added during execution: #713 and the four doc tasks the review spawned.
- **PRs merged:** 19, including the spec/plan PR and excluding this one.
- **Releases cut (by the human):**
  - vergil-actions **v2.1.57**: `ops-upstream-refresh.yml` at `v2.1`.
  - vergil-tooling **v2.1.267**: T1–T6.
  - vergil-python **v1.0.4**: the first two-series release.
  - The package index was rebuilt to about 305 MB; the warning threshold is 750 MB.
- **Span:** epic opened 2026-10-25 12:16Z; last task closed 2026-10-26 12:22Z, about 24 hours.

## 1. How the plan evolved

The task map held up. Every planned row shipped in roughly the planned order and none was
dropped. The changes were inside tasks, plus one consolidation task the human added at the end.

- **vergil-tooling ran as a serial chain.** T1 (config) was the root. T2 and T5 followed together
  once it merged, then T3, then T4, and T6 ran alongside T1 from the start. Each link needed the
  human to merge it before the next could start, so the chain took five merge round trips over
  about 3.5 hours (19:20Z–22:48Z). That was correct, since each step built on the last, but it is
  where most of the day-one wall-clock time went.
- **Plan conflict: manifest `name`.** The plan's interface section gave every manifest artifact a
  `name`. Its own Global Constraint said a single-package manifest must not change. T3 followed
  the constraint: only multi-package manifests carry `name`. T4 treats a missing `name` as the
  single-package case.
- **Plan wording that couldn't hold: `runtime_depends`.** T2's plan asked for identical
  `runtime_depends` output across the old and new version formats. That is impossible, because
  the dependency floor contains the version string itself. The test checks the intent instead:
  the floor is the served version, and the CPython patch is read correctly from both formats.
- **Small design changes in T6.** It builds the refresh commit with `write-tree`/`commit-tree` on
  `origin/<base>` rather than on a detached HEAD, so no local branch moves and no commit hook
  fires. It also treats `upstream-refresh-*` branches as refresh PRs.
- **A1 and T6 coordinated mid-flight.** A1 surfaced that the reusable workflow's checkout is
  depth 1. That constraint was passed to T6 while it was still running, and it shipped with
  explicit shallow-clone handling and tests.
- **The reduced CI tier became per package.** One install-test cell per package and format, not
  per format, so a multi-package repo still tests every package.
- **P2 settled two cases the plan left open.** A window version that PBS ships for only one
  architecture is held back with a note rather than failing the run. When several newer window
  versions exist, all are held until the lowest ships, so adding a higher one never turns a lower
  one into a back-fill that can never land.
- **vergil-containers: C1, then a consolidation.** C1 followed the plan and kept the refresh in a
  standalone `bump-tools.yml`. After seeing vergil-python run it from `ops.yml`, the human asked
  for one layout. vergil-containers#713 was added to the epic, folded the refresh into `ops.yml`
  and deleted `bump-tools.yml`. The cadence changed from weekly to daily as a result.
- **V1 check 5 was unsatisfiable as written.** It asked for old- and new-format index entries to
  coexist, but K1's `keep = 1`, a deliberate epic decision, prunes the old entry. With the human's
  approval it was judged on its intent instead: an installed old version upgrades to the new
  format. apt actually performed that upgrade on the agent VM.
- **The documentation review spawned four cross-repo tasks.** It fixed vergil-tooling's own docs
  and filed same-repo doc tasks for vergil-containers, packages, docs and vergil-actions. All
  four were done in parallel and merged in one batch.

## 2. Lessons learned

- **Check the plan against itself.** Two of the deviations above (manifest `name`, V1 check 5)
  were plan-internal conflicts: an interface or validation step that contradicted a Global
  Constraint or another task's decision. A pass over the plan that checks each validation check
  against the constraints and the other tasks' outputs would have caught both before execution.
- **Deploy and validation steps must be runnable by the identity that runs them.**
  - The D2 deploy step used `vrg-gh api`, which is denied for the user identity. It was checked
    with `vrg-git ls-tree` on the tag instead.
  - The D1 step used `uv tool install --force`, which would have shadowed the packaged
    vergil-tooling on the agent VM. It was run with `uvx --from …@v2.1` instead.
  - V1 checks 2–3 needed `gh workflow run`, which the user identity can't do, so the human
    dispatched both runs.
  - Next time, plan each operational step for the identity that will run it, and label the
    human-only steps up front.
- **Workflow-touching PRs need a different submission path.** Five branches changed
  `.github/workflows/**`: A1, C1, P3, #713 and #1069. GitHub refused the agent push for each. The
  worktree-free `vrg-submit-pr <branch>` form then fails because `origin/<branch>` doesn't exist,
  so each had to be submitted from inside its worktree. Handoffs now state this every time. It is
  a recurring cost worth smoothing in tooling.
- **Parallel same-file work in one repo went fine.** T1 and T6, and then T2 and T5, each pair
  touching `config.py` or `vergil_index.py`, merged without a conflict. Keeping the edits local
  and appended worked.
- **Same next time:** sub-agents per task with the worktree prompt contract, per-repo batching,
  and handing off a repo's PRs as soon as its ready set was complete. Doing the doc-review
  spawns in parallel turned a five-repo sweep into one merge batch.

## 3. Compromises & tradeoffs

- **`keep = 1` is a stopgap.** It keeps the index well under `SIZE_WARN`: about 305 MB, close to
  the ~338 MB projected. A third Python series would push the worst-case projection to about
  900 MB, right at `SIZE_FAIL`. A third series is blocked on a retention or hosting redesign,
  .github#466. The cost of `keep = 1` is that only one release per line can be installed from the
  index, so there are no rollback candidates.
- **The refresh write path is unproven.** Both V1 refresh runs exited quietly because nothing had
  changed. A real push of `upstream-refresh` and a PR opened through the App token from a
  `contents: read` job have not happened yet. The human accepted this. .github#491 tracks
  confirming it on the first real upstream change.
- **The first real `vrg-upstream-refresh` run was in CI.** The command refuses agent identities,
  so no agent could dry-run it against the real repos. Its tests use a local bare origin with
  `gh` faked.
- **No trailing-edge automation.** Removing a series is a hand-edited PR by design (spec). There
  is no EOL reporting yet.
- **Small rough edges, accepted:**
  - After a manual run, `vrg-upstream-refresh` leaves the hook's edits uncommitted, so the next
    manual run's clean-tree check refuses until they are discarded.
  - `vrg-validate`'s shellcheck covers only `scripts/` in vergil-containers, so the new
    `docker/pins/upstream-refresh.sh` hook isn't linted.
  - The P1 merge hit an orphaned check-run: GitHub posted a result after `vrg-finalize-pr`'s
    5-minute grace period. A re-run merged cleanly.
- **One process deviation.** The T3 agent ran `ruff format` directly to clean up mechanical test
  edits, outside the "only `vrg-validate`" rule. Final validation still ran only through
  `vrg-validate`, and later agents were told explicitly not to do this.

## 4. New problems & opportunities

| Item | Where it went |
|---|---|
| Confirm the first real refresh PR is opened through the App token | [.github#491](https://github.com/vergil-project/.github/issues/491) (intake) |
| A replacement for GitHub Pages package hosting (1 GB cap), a prerequisite for a third series | [.github#466](https://github.com/vergil-project/.github/issues/466), existing |
| Fleet-wide: move other scheduled workflows (for example vergil-containers' `package-cleanup.yml`) under `ops.yml` | [.github#493](https://github.com/vergil-project/.github/issues/493) (intake) |
| Deploy/validation steps use commands the user identity can't run (`vrg-gh api`, `gh workflow run`) | Logged, not yet acted on |
| Worktree-free `vrg-submit-pr <branch>` can't submit a branch whose agent push was refused | Logged, not yet acted on |
| `vrg-whoami --mode` printed `User` where skills expect `user`; the identity mode should be case-insensitive | [.github#494](https://github.com/vergil-project/.github/issues/494) (intake) |
| `vrg-validate` shellcheck scope excludes `docker/pins/` in vergil-containers | [.github#362](https://github.com/vergil-project/.github/issues/362), existing |
| `vrg-upstream-refresh` leaves hook edits in the tree after a manual run | [.github#495](https://github.com/vergil-project/.github/issues/495) (intake) |

## 5. What's next

- **Out of scope by design, and still open:**
  - unattended merge or release of refresh PRs, [.github#470](https://github.com/vergil-project/.github/issues/470);
  - the notification gap, [.github#471](https://github.com/vergil-project/.github/issues/471);
  - a replacement for GitHub Pages package hosting (1 GB cap), [.github#466](https://github.com/vergil-project/.github/issues/466).
- **Watch:** the first real upstream change, to confirm the write path (.github#491). After that,
  the first automatic leading-edge add when the window gains 3.16.
- No follow-on brainstorm was spun out of this epic.

## Appendix A — Operational notes

The deploy order that worked:

1. **K1:** merge to `packages` develop. No release is needed; `publish-index.yml` reads
   `packages.toml` from develop.
2. **A1:** release vergil-actions (v2.1.57). Confirm the App grants Contents: write and Pull
   requests: write on each consuming repo.
3. **T1–T6:** release vergil-tooling (v2.1.267). Single-package regression check: run
   `vrg-package matrix` on vergil-tooling and packages, and confirm the cell ids are unchanged
   with no `--` qualifier.
4. **P1–P3:** release vergil-python (v1.0.4). Confirm both series are in the deb and rpm indexes
   at `<cpython>+<pbs_tag>.<release>`, and that publish-index printed no `SIZE_WARN`.
5. **C1 / #713:** merge only. Scheduled workflows run from develop and call
   `ops-upstream-refresh.yml@v2.1`.

**Gotcha:** run D1's check with `uvx --from 'vergil-tooling @ git+…@v2.1'`, not
`uv tool install --force`, so the agent VM's packaged vergil-tooling isn't shadowed.
