# Remove the cutover vergil-tooling install switch — retrospective

- **Epic:** vergil-project/.github#1292
- **Planning issue:** vergil-project/.github#983
- **Spec:** [`spec.md`](spec.md) · **Plan:** [`plan.md`](plan.md)

## 0. At a glance

Epic #649 gave vergil-tooling's cache build and vergil-actions' setup action a
switch that probed each image (the vergil apt source and a core vergil-python)
and picked the packaged install or a `uv tool install` from git, refusing an
image with only one of the two. It existed only to survive the Debian-to-Ubuntu
cutover in either direction. The cutover held, so we set out to delete the
switch in both repos with nothing in its place: in a container, a released ref
does the packaged install and a dev ref installs from git on the core Python.

That shipped as planned. Both deletions landed, the vergil-actions half went
out as the only significant change in v2.1.82, and the documentation review
found one stale page in the `docs` repo, which a spawned task fixed.

| PR | Repo | Task | What it did | Merged |
| --- | --- | --- | --- | --- |
| [#1299](https://github.com/vergil-project/.github/pull/1299) | .github | #1293 | Spec and plan | 2026-11-17 |
| [#4664](https://github.com/vergil-project/vergil-tooling/pull/4664) | vergil-tooling | #4657 | Cache build drops the switch; tests and `ci-architecture.md` follow (+60/−154) | 2026-11-17 |
| [#1300](https://github.com/vergil-project/vergil-actions/pull/1300) | vergil-actions | #1297 | `install-tooling.sh` drops the container switch; tests, action and docs follow (+53/−153) | 2026-11-17 |
| [#131](https://github.com/vergil-project/docs/pull/131) | docs | #130 | `packages.md` "Dev containers" row names only the two remaining paths | 2026-11-18 |

- **Repos touched:** 4 (.github, vergil-tooling, vergil-actions, docs).
- **Tasks:** 7 children, including this retrospective. 4 PRs before this one;
  one deployment task (vergil-actions#1298) and one documentation review
  (vergil-tooling#4649) closed without a PR.
- **Releases:** vergil-actions v2.1.82 (published 2026-11-18) carried #1300,
  and `v2.1` was moved to it. #4664 merged after vergil-tooling v2.1.322 and
  is **not yet in a vergil-tooling release**; it rides the next normal one, as
  planned.
- **Span:** planning issue opened 2026-11-09; epic opened 2026-11-17 22:18Z;
  last task closed 2026-11-18 12:00Z.

## 1. How the plan evolved

Very little. Tasks 2 and 3 ran in parallel, as the plan allowed, and each
followed its steps as written, including the expected reds: 17 failures in
vergil-tooling before the code change, and the "refusing to guess" failures
in vergil-actions once the apt source left the kept container cases. Each
merge review passed with no findings, and the release check passed on the
first run.

Two entries went into the plan's evolution log:

- **One more doc page.** The spec and plan listed every doc in the three
  member repos that described the switch, and plan-review alignment confirmed
  the list. But the `docs` repo also described it, in the "Dev containers"
  row of `packages.md`. The documentation review caught it and spawned
  docs#130, which landed the same morning.
- **Where the release version was recorded.** Task 4 expected the human to
  name the release in a comment on the task; the human named it in the
  driving session instead, and the deployment run's result comment recorded
  it. No effect beyond where the record lives.

## 2. Lessons learned

- **A deletion still repays design review.** Four review rounds raised 17
  findings, all accepted. Two were high: with the count check gone,
  `find_core_pythons` would have exited silently under `set -e` on an image
  with no core Python (A1.1), and dropping the test fixture's source
  parameter needed every call site updated (A1.2). Neither is the kind of
  thing a "just delete it" PR gets right by default. The payoff showed at
  merge review, which found nothing in any of the three code PRs.
- **A doc search has to cover the `docs` repo, not just the member repos.**
  The sweep is the backstop and it worked, but the plan's doc list would have
  been complete if the planning search had included `docs`. Next time, search
  `docs` during planning for any change to how Vergil installs or runs.
- **"Only significant change" is a judgment, not a mechanical check.**
  v2.1.82 also carried two comment-only docs PRs (vergil-actions #1292 and
  #1296). The deployment run read both diffs and judged them non-significant;
  the revert path to v2.1.81 stayed clean. The rule worked because someone
  read the diffs.
- **The CI-only suite needs naming in the plan.** `vrg-validate` skips
  `install-tooling.test.sh`, so the local gate never ran the changed paths.
  Plan review caught this (P1.1), the run ran the suite by hand in the
  container (31 pass, the one known-failing case), and the PR's CI was the
  real gate. Without P1.1 the run would have reported a green validation that
  proved nothing about the change.
- **Same again next time:** parallel independent tasks across repos, per-repo
  submission, and an isolated release with a human hold on the repo.

## 3. Compromises and tradeoffs

- **No check replaces the switch, by design.** Every image carries exactly one
  core Python and the vergil apt source, so the probes are stale. The cost:
  - an image with no core Python fails at an unbound array element under
    `set -u`, loudly but without a tailored message;
  - an image with two core Pythons would install on the first match (S2.3);
  - the candidate-check advice no longer fits an image without the vergil apt
    source (S1.4). No Vergil image or workflow reaches it, so it was left.
- **The way back to Debian images is gone.** That was the point, after the
  human judged the cutover had held; reverting vergil-actions to v2.1.81
  remains a clean undo for the setup action only.
- **vergil-tooling's half is merged but not released.** Until the next
  vergil-tooling release, cache builds from a released vergil-tooling still
  run the old switch. That is harmless on Ubuntu images, where the switch
  already took the packaged path.
- **One slip during Task 3, caught in-run.** The run's deletion script dropped
  the test file's executable bit and left extra blank lines. Validation caught
  the mode and the first review the blank lines; both were fixed in the same
  commit before a fresh review.

## 4. New problems and opportunities

**Unaddressed-findings sweep.** Swept the 6 children other than this
retrospective, the epic issue and the planning issue (#983), with trusted
authors checked. No `Awaiting a ruling` or `Filed without a ruling` list
exists on any of them. The four merged PRs have no reviews or conversation
comments. Every design-review finding (17 across four rounds) was accepted
into the spec or plan; S1.4 was accepted as a known limitation (§3).
Results:

- **docs `packages.md` described the switch** — filed by the documentation
  review as vergil-project/docs#130 and fixed by docs#131.
- **Planning issue #983 is still open** after its docs PR merged. Found:
  vergil-project/vergil-tooling#4670 tracks the general fix (the docs PR
  should close the planning issue). #983 itself still needs closing.

No other untracked problem found.

## 5. What's next

No follow-on brainstorm or epic. This closes step 12 of epic #649's spec, as
widened here to the whole switch. The remaining open item is vergil-tooling's
next normal release, which ships #4664.
