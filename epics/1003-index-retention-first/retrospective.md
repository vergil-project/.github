# Package index: retention-first build — retrospective

- **Epic:** vergil-project/.github#1003
- **Spec:** [spec.md](spec.md) · **Plan:** [plan.md](plan.md)

## 0. At a glance

We set out to make `vrg-package index` do work in proportion to what it
publishes. Every `publish-index` run was spending 20 to 26 minutes in
`build-index`, which blocked vergil-tooling releases
(vergil-project/packages#46). The cause was the order of work. The index
downloaded and attestation-verified every packaged release of every product,
and only then applied retention, which keeps one release per line. What
shipped selects the releases to publish from one names-only listing per
product, then downloads and verifies only those. It also deletes the dependency
closure, the duplicate-bytes check and the retire checks, which only made sense
when the index held every release.

**Result:** on the first run with the fix (packages run 42890241068),
`build-index` took **1m 52s**, down from 26m 13s on the run just before it. The
index published the same products: vergil-tooling 2.1.299-1, both
vergil-python runtimes and the archive keyring.

| PR | Repo | What it did | Merged |
|---|---|---|---|
| [#1011](https://github.com/vergil-project/.github/pull/1011) | .github | Spec and plan | 2026-11-10 |
| [#4174](https://github.com/vergil-project/vergil-tooling/pull/4174) | vergil-tooling | Select releases before downloading and verifying; delete the closure, duplicate and retire checks; update the reference docs | 2026-11-10 |
| [#1140](https://github.com/vergil-project/vergil-actions/pull/1140) | vergil-actions | `publish-index.yml` header comment describes the new order | 2026-11-11 |
| [#50](https://github.com/vergil-project/packages/pull/50) | packages | `packages.toml` comment describes the new order | 2026-11-11 |
| [#1030](https://github.com/vergil-project/.github/pull/1030) | .github | Correction notes on epic #356's spec | 2026-11-11 |

- **Repos touched:** 4 (.github, vergil-tooling, vergil-actions, packages).
- **Tasks:** 9, plus this retrospective. That is 6 PRs counting this one, one
  deployment and one validation.
- **Releases cut:** vergil-tooling v2.1.299 (2026-11-10, by the human).
- **Span:** epic opened 2026-11-10T15:37Z; the last task closed
  2026-11-11T17:08Z.

## 1. How the plan evolved

The plan did not log its changes in an `## Evolution during execution`
section, although it changed during execution. The record is in the task
issues:

- **The deployment task's acceptance was too weak.** Task 2 accepted "a
  `publish-index` run started after the release", and that run existed. But it
  started 11 seconds after v2.1.299 was published, before the floating `v2.1`
  tag moved, so it installed v2.1.298 and ran the old full scan. The task
  closed SUCCESS on a run that did not contain the fix. Its correction comment
  records this. The validation was blocked rather than passed or failed, and
  the human started a fresh run once `v2.1` had moved. That run is the
  evidence in §0.
- **The docs-review sweep spawned three per-repo tasks** (vergil-actions#1135,
  packages#48, .github#1029). The design review had named those stale texts in
  the spec's bookend. vergil-tooling's own docs needed nothing more, because
  #4174 had rewritten them.
- **A late edit to Task 1's branch.** The merge-review fix added a test
  assertion that the removed wording never reappears. The human had it
  removed: the branch was unfrozen and the one line deleted before submission.

## 2. Lessons learned

- **Look for work that grows with history but not with output.** The full
  scan cost nothing when it was written (2026-10-05, almost no packaged
  releases), and its cost grew with every release. The spec it came from
  listed the steps in the wrong order, and the implementation followed it
  literally.
- **Guard against the world, not against ourselves.** The first spec carried
  checks that only fire if the fleet breaks itself: comparing the new index
  with the old algorithm's output, a dependency closure searching all of
  history for a runtime that is always in the selected release, and retire
  typo checks. The human ruled them out, and the design shrank to selection,
  download and verification. The same pattern came back during
  implementation as a never-again test assertion. This is now tracked as a
  fleet principle (.github#1006).
- **An operational task must check what actually ran, not only that
  something ran.** Task 2 checked that a run existed. Checking the installed
  version in that run's log would have caught the stale tag at once.
- **One design-review round per stage was enough for a small, well-understood
  change.** Spec pushback found the real pagination bug. Plan pushback found
  the tests and the `version_key` import the plan would have broken. Alignment
  removed a redundant per-release API call.

## 3. Compromises and tradeoffs

- **The selection no longer guards installability across products.** With the
  dependency closure and the retired-dependency check gone, a vergil-tooling
  release that depends on a vergil-python runtime missing from the selected
  vergil-python release would be indexed without it. The human accepted this.
  It only happens if we release a vergil-python that drops a runtime a current
  vergil-tooling still needs.
- **Duplicates in releases the index does not publish no longer fail the
  index.** Only the `vrg-release` preflight (`published.py`) refuses a
  re-published version, before release.
- **Epic #356's spec is annotated, not rewritten.** Its §7.2 "Duplicates"
  bullet still describes the removed check. The §7.2 correction note says
  that check is gone.

## 4. New problems and opportunities

| Problem | Where it went |
|---|---|
| Pages deployment reports success but is not served | vergil-project/packages#46 (existing; the human is investigating; a `live-check` step has since landed in `publish-index.yml`) |
| Engineering principle: no self-guarding checks or never-again tests | Filed: .github#1006 (intake) |
| A release dispatches the index rebuild before moving the floating minor tag, so the rebuild runs the previous tooling | Filed: .github#1015 (intake) |
| Old releases, CI evidence and other release side-effect artifacts are never pruned (vergil-tooling has 417 releases) | Found: .github#235, which tracks accumulating side-effect data. On the human's ruling, the pruning research from this epic's start was added to it as a comment. |
| The `vrg-release` preflight (`published.py`) still views every release on every release, about 417 calls, and `stable_tags` hard-fails at 1,000 releases | Filed on the human's ruling: .github#1032 (intake) |

**Unaddressed-findings sweep:** swept 8 tasks, the epic issue and planning
issue .github#1002. No `Awaiting a ruling` item is open. Every design-review
finding not accepted has a recorded reason. The five merged PRs have no
reviews that could hold inline comments. Two problems were untracked, the last
two rows above. One was found in .github#235 and the other filed as
.github#1032.

## 5. What's next

No follow-on epic. The open threads are the intake items .github#1006,
.github#1015 and .github#1032, and the pruning scope added to planning item
.github#235.
