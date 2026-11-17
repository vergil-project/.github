# Rust validation, supply-chain and dependency-update parity — retrospective

- **Epic:** vergil-project/.github#1165
- **Spec:** [`spec.md`](spec.md) · **Plan:** [`plan.md`](plan.md)
- **Planning issue:** vergil-project/.github#1087
- **Span:** opened 2026-11-16 12:38 UTC; last task closed 2026-11-17 12:12 UTC (about 24 hours)

## 0. At a glance

We set out to bring Rust's validation, supply chain and dependency updates up to Python's level. That meant an enforced lockfile, pinned and digest-verified validation tools, a 100% lines-and-regions coverage gate, CI evidence naming Rust's own tools, a Cargo updater in `vrg-update-deps`, and the release profile built in validation, all proven by a fixture run on each Rust fixture.

All of it shipped. Both fixture runs passed every check on their own, with no known gaps. The two Rust known gaps the `fixture-run` skill used to excuse (the dependency rollback and the `quality` gate) are now ordinary checks that pass.

**Counts:**

- 27 tasks under the epic, 20 of them closed by a merged PR. The other 7: one deployment, two fixture releases, two fixture validations, the retrospective, and one coverage task closed with no change.
- 7 repos touched: `.github`, vergil-tooling, vergil-actions, vergil-containers, vergil-claude-plugin, vergil-fixture-rust-lib and vergil-fixture-rust-app.
- 6 releases cut.

**PRs:**

| Repo | PR | Task | What it did |
| --- | --- | --- | --- |
| .github | [#1194](https://github.com/vergil-project/.github/pull/1194) | #1166 (1) | Spec and plan. |
| vergil-tooling | [#4483](https://github.com/vergil-project/vergil-tooling/pull/4483) | #4470 (2) | `--locked` on every Rust cargo command. A missing `Cargo.lock` fails. The warmup never rewrites the lockfile and skips loudly. |
| vergil-tooling | [#4485](https://github.com/vergil-project/vergil-tooling/pull/4485) | #4474 (5) | `vrg-version` reads and bumps `[workspace.package].version`, through the new `cargo_semver` matcher. |
| vergil-tooling | [#4491](https://github.com/vergil-project/vergil-tooling/pull/4491) | #4471 (4) | `--workspace` on every Rust gate command, `cargo llvm-cov report` included. |
| vergil-tooling | [#4493](https://github.com/vergil-project/vergil-tooling/pull/4493) | #4475 (7) | The Cargo updater and its held-back report. |
| vergil-tooling | [#4499](https://github.com/vergil-project/vergil-tooling/pull/4499) | #4472 (3) | 100% lines and regions, with uncovered regions listed by llvm-cov's own count. |
| vergil-tooling | [#4518](https://github.com/vergil-project/vergil-tooling/pull/4518) | #4473 (6) | The clippy and cargo-check reports. |
| vergil-tooling | [#4521](https://github.com/vergil-project/vergil-tooling/pull/4521) | #4476 (13) | `cargo build --release --locked --workspace` through epic #1174's BUILD check. |
| vergil-tooling | [#4538](https://github.com/vergil-project/vergil-tooling/pull/4538) | #4443 (docs review) | CI, evidence and Rust standards pages brought up to date. |
| vergil-actions | [#1251](https://github.com/vergil-project/vergil-actions/pull/1251) | #1249 (8) | Per-leg lint and typecheck evidence. A passing leg with no report fails, Python included. |
| vergil-containers | [#920](https://github.com/vergil-project/vergil-containers/pull/920) | #917 (9) | The refresh pins prebuilt digests in source. cargo-nextest is verified against them. |
| vergil-containers | [#922](https://github.com/vergil-project/vergil-containers/pull/922) | #918 (10) | cargo-deny (prebuilt, digest-pinned) and cargo-llvm-cov (`cargo install --locked`) move to refreshed fragments. |
| vergil-containers | [#930](https://github.com/vergil-project/vergil-containers/pull/930) | #929 (docs) | Comments and docs describing the pinned Rust tools. |
| vergil-claude-plugin | [#1108](https://github.com/vergil-project/vergil-claude-plugin/pull/1108) | #1104 | `fixture-run` drops the Rust known gaps and turns on the Cargo rollback. |
| fixture-rust-app | [#19](https://github.com/vergil-project/vergil-fixture-rust-app/pull/19) | #18 (11b) | Region coverage from 98.64% to 100%, tests only. |
| fixture-rust-app | [#24](https://github.com/vergil-project/vergil-fixture-rust-app/pull/24) | #20 (run) | The `min` subcommand. `anyhow` rolled back in `Cargo.lock`. |
| fixture-rust-app | [#30](https://github.com/vergil-project/vergil-fixture-rust-app/pull/30) | #23 (docs) | Docs describe the lines-and-regions gate and the release build. |
| fixture-rust-lib | [#25](https://github.com/vergil-project/vergil-fixture-rust-lib/pull/25) | #21 (run) | `longest_word`. `memchr` rolled back in `Cargo.lock`. |
| fixture-rust-lib | [#31](https://github.com/vergil-project/vergil-fixture-rust-lib/pull/31) | #24 (docs) | The same docs fix. |
| .github | [#1201](https://github.com/vergil-project/.github/pull/1201) | #1198 | The Task 3 entry in the plan's evolution log. |

Task 11a (vergil-fixture-rust-lib#20) measured 100% regions already (55 of 55) and closed with no PR.

**Releases:**

- [vergil-tooling v2.1.319](https://github.com/vergil-project/vergil-tooling/releases/tag/v2.1.319)
- [vergil-actions v2.1.78](https://github.com/vergil-project/vergil-actions/releases/tag/v2.1.78)
- [vergil-containers v2.1.43](https://github.com/vergil-project/vergil-containers/releases/tag/v2.1.43)
- [vergil-claude-plugin v2.1.74](https://github.com/vergil-project/vergil-claude-plugin/releases/tag/v2.1.74)
- [vergil-fixture-rust-lib v0.1.1](https://github.com/vergil-project/vergil-fixture-rust-lib/releases/tag/v0.1.1)
- [vergil-fixture-rust-app v0.1.1](https://github.com/vergil-project/vergil-fixture-rust-app/releases/tag/v0.1.1)

The deployment (#1190) verified the Rust images (`prod-rust:1.99` and `1.98`): each pinned tool reports its pinned version, and each prebuilt tarball passed `sha256sum -c` against its pinned digest in every build log.

## 1. How the plan evolved

The plan held up. Every planned task shipped in the planned order, and nothing was dropped or split. The changes came from three places.

**A fixture measurement corrected the design before it shipped.** The plan put Task 3's region gate behind both fixtures' coverage tasks ("measure before switching on"). That ordering paid for itself. Bringing vergil-fixture-rust-app to 100% showed that llvm-cov counts a generic function by its best-covered instantiation alone, while the plan's region locator merged counts across instantiations. As planned, the gate could fail with an empty listing. The human ruled to follow llvm-cov's rule, posted a scope change on #4472, and #1198 logged it before Task 3 was implemented. The live check on Task 3 then showed the difference directly: a probe module with two missed regions was listed as two, where the planned merge would have listed one.

**Work arrived from outside the plan's task list:**

- vergil-claude-plugin#1104 was filed at epic creation (the plan's filing note implies it), but the delivery table never named it.
- The fixture runs were filed at implementation time, once the updated plugin was installed, rather than at planning.
- The documentation review spawned three per-repo doc tasks: vergil-containers#929, vergil-fixture-rust-lib#24 and vergil-fixture-rust-app#23.
- The plan amendment itself needed a task (#1198), because the tooling refuses a branch named for an epic.

Only the Task 3 change reached the evolution log during execution. The others were added to it in this retrospective's PR. That gap is a driver lapse, covered in §2.

**A dependency outside the epic landed on time.** Task 13 depended on epic #1174's BUILD check kind, which was not yet filed when the plan was approved. It was filed (vergil-tooling#4503) and merged mid-epic, so Task 13 never waited on it.

## 2. Lessons learned

- **Measuring before switching a gate on works; keep doing it.** Tasks 11a and 11b ran ahead of Task 3 for exactly this reason, and 11b found a design error that review had not.
- **Live checks find what string tests do not.** Each tooling task ran its change against a real fixture in the real image. That caught:
  - cargo 1.99 wording the stale-lockfile error differently from the plan's quote (#4470);
  - a grep pattern starting with `--` (#4470);
  - `cargo llvm-cov report` needing `--workspace`, or member crates escape the gate (#4471);
  - clippy's JSON printing each diagnostic twice under `--all-targets` (#4473).

  None of these would have failed a unit test.
- **A vergil-tooling release that changes managed files needs those files synced before the next fixture work.** Plan review round 2 discarded "the fixtures' `.gitignore` won't list the new reports until a sync" because no step was found that breaks. One did:
  - a dirty worktree blocks merge review;
  - an untracked report blocks `vrg-release`'s preflight.

  It is now a proposed learned condition for `fixture-run` (vergil-claude-plugin#1116).
- **A release order is a statement about who consumes what.** The plan said vergil-tooling first, because vergil-actions requires reports that tooling writes. The human released vergil-actions first, deliberately, after checking that no Rust repo's CI would run in the gap. The rule protects consumers in the gap. When there are none, the order can change, and the attestation on #1190 records why.
- **Log plan changes when they happen.** The driver logged the change it had to argue for (Task 3) and not the routine additions. Next time, every new task under the epic gets its evolution entry when it is filed, in the same task or batch.
- **A frozen branch can still need a rebase.** vergil-containers#917 conflicted with an unrelated change (#919) that merged while it waited. `vrg-pr-workflow unfreeze`, then a conflict resolution and a re-run of `report-ready`, worked cleanly. The merge review was not re-run (§3).

## 3. Compromises and tradeoffs

- **#917's conflict resolution was not re-reviewed.** The driver resolved it in the frozen branch and re-ran `vrg-validate`, but the merge review was not run again. The resolution merged two reviewed changes and adjusted test fixtures for both. In the result, #919's workflow pins are written after the all-or-nothing version-and-digest write, as #919's own code already did. So a failed workflow-pin write would leave the fragments updated.
- **Image builds were never run locally.** This host has no docker or podman. Each container task ran its fragment's install step inside the dev container instead. CI's build was the first full build of the new fragments. The deployment then verified the published images.
- **Two pins moved from the plan:**
  - cargo-deny is pinned at 0.21.0, not the plan's 0.20.2: 0.21.0 was released the day Task 10 ran.
  - cargo-llvm-cov has no published checksums, so it is trusted through `cargo install --locked` at a pinned version, as the spec accepts.
- **The Rust lint and typecheck legs now require `jq`.** Every vergil-containers image ships it. A repo validated outside those images would need it installed.
- **#4475's merge review reads `not-eligible (check-failed)`** with every finding fixed. The run used its single fix round on R1, and its R1 fix introduced C1, which needed the human's ruling. The record is accurate; it just reads worse than the code is.
- **Non-fleet languages skip the new lint and typecheck evidence.** The plan's literal rule would have failed every `ci-quality` leg for `shell`, `claude-plugin` and empty languages. #1249 gives them a notice instead. That also corrected their evidence, which had wrongly listed ruff and mypy.

## 4. New problems and opportunities

**Filed during the epic, each on the human's ruling:**

- vergil-project/vergil-tooling#4484: `vrg-version` prints a traceback for expected errors (F1).
- vergil-project/vergil-claude-plugin#1115: `issue-deploy` and `issue-validate` close a task before its result comment is confirmed (F4, seen on #1190).
- vergil-project/vergil-claude-plugin#1116: the managed-files-in-sync learned condition for `fixture-run`.
- vergil-project/.github#1228: rename `vrg-release`'s anachronistic `consumer-refresh` stage and its warning.
- vergil-project/.github#1229: review git-cliff's unparsed-commit warning, then fix it or suppress it.
- vergil-project/.github#1230: update `Cargo.lock` with the version bump, so the release warmup logs no false `error:` line.

**Recorded on existing issues:**

- Workspace `exclude` entries are matched exactly where Cargo treats them as path prefixes. Recorded on vergil-project/.github#1179 (F2).

**Declined:**

- Reading other registries' indexes in the Cargo updater (F3). Failing loud is the spec's choice.

**Unaddressed-findings sweep.** 26 tasks were swept, plus the epic issue and the planning issue (#1087).

- Every `Awaiting a ruling` item is closed, and every filed issue exists.
- No PR has a `COMMENTED` or `CHANGES_REQUESTED` review.
- No security-class item came up.
- Every planning-round finding not taken has its reason or ref.

Untracked problems it found, and the human's ruling on each:

1. **The plan's evolution log missed #1104 and the implementation-time fixture-run filing.** The planning issue said they would be logged ([comment](https://github.com/vergil-project/.github/issues/1087#issuecomment-6923249677)). **Fixed** in this PR, along with entries for #1198, the fixture runs and the three doc tasks.
2. **Deleting `Cargo.toml` but keeping `Cargo.lock` now fails on cargo's message,** not `vrg-validate`'s "not bootstrapped" message (#4470, below the review's fix bar). **Dropped:** it still fails loud.
3. **The Skill tool kept serving the cached plugin v2.1.73 after v2.1.74 was installed mid-session.** The driver filed the fixture runs from the installed v2.1.74 text by hand. **Filed** as vergil-project/.github#1236.
4. **The planning issue #1087 was still open,** though its docs PR (#1194) had merged. **Closed.**
5. **A TypeScript sentence in vergil-tooling's `ci-evidence-convention.md`** (around line 284) is stale, and belongs to epic #1174's documentation review. **Found:** noted on vergil-project/vergil-tooling#4451.

## 5. What's next

This is Epic B of the Rust-parity wave, on the way to rebuilding Vergil in Rust in 2027. The follow-ons already exist:

- **Full workspace support,** and whether the two Rust fixtures become one workspace fixture: vergil-project/.github#1179.
- **Branch coverage** once it works on stable Rust: vergil-project/.github#1171 (deferred).
- **The Rust lifecycle window** from Rust's release channel: vergil-project/vergil-containers#910.
- **Merging lifecycle PRs automatically:** vergil-project/.github#1128.
- **Turning BUILD on for Python:** vergil-project/.github#1211.

## Appendix A — Operational notes

**Release order, as run.**

1. vergil-claude-plugin v2.1.74, with #1104.
2. vergil-actions v2.1.78 and vergil-containers v2.1.43.
3. vergil-tooling v2.1.319, about 30 minutes after vergil-actions.

The human attested the reversed order on #1190. In the gap, a Rust repo's CI would fail its lint and typecheck legs on missing reports. Nothing ran there.

**Fixture-run sequence, per fixture:**

1. The change task, then `vrg-submit-pr`.
2. The managed-files sync: the human's `vrg-github-repo-config update`, then its `chore(repo-config)` PR.
3. A `git pull` in the clone the release runs from.
4. `vrg-release`, run by the human from their own terminal.
5. The release task, given the human's output.
6. The validation task.
7. The fixture's doc task.

**Gotchas:**

- **The managed-files sync:** before step 2, `vrg-release`'s preflight refuses ("Working tree is not clean") because of the untracked `quality-*.json` reports.
- **Bump and lockfile:** the back-merge bump changes `Cargo.toml` before `Cargo.lock`, so the warmup logs an `error:` line and skips. It is harmless; #1230 tracks it.
- **Rollback floors:** the advisory floor can set the rollback version. In rust-app, `anyhow 1.0.102` fails cargo-deny (RUSTSEC-2026-0190), so the rollback stopped at 1.0.103.
- **Image digests during a fixture run:** a nightly prod rebuild can change the digest between the change PR and the release. Both images were built from v2.1.43.
