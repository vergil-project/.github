# Narrow vergil-tooling to Python 3.15 — retrospective

- **Epic:** vergil-project/.github#1256
- **Planning issue:** vergil-project/.github#958
- **Spec:** [`spec.md`](spec.md) · **Plan:** [`plan.md`](plan.md)

## 0. At a glance

We set out to stop testing vergil-tooling on four Python series (3.12–3.15),
since every Vergil image and VM already runs it on 3.15. The plan was to name
`--python 3.15` on every install that named no interpreter, release the
vergil-actions change, and then raise `requires-python` to `>=3.15`.

All of that shipped. The planned mechanism for the last step did not work:
`requires-python` does not drive the CI matrix. The single series came instead
from changing vergil-tooling's `repository-type` from `library` to `tooling`.
CI now builds one series, the newest in the window with a published runtime.

| PR | Repo | Task | What it did | Merged |
| --- | --- | --- | --- | --- |
| [#1272](https://github.com/vergil-project/.github/pull/1272) | .github | #1258 | Spec and plan | 2026-11-17 |
| [#1283](https://github.com/vergil-project/vergil-actions/pull/1283) | vergil-actions | #1278 | `install-tooling.sh` names `--python 3.15`; host self-repo sync on 3.15 | 2026-11-17 |
| [#421](https://github.com/vergil-project/vergil-vm/pull/421) | vergil-vm | #420 | VM init and its install test name `--python 3.15` | 2026-11-17 |
| [#4622](https://github.com/vergil-project/vergil-tooling/pull/4622) | vergil-tooling | #4607 | `container_cache`, `vm_packages` and the developer docs name 3.15 | 2026-11-17 |
| [#4647](https://github.com/vergil-project/vergil-tooling/pull/4647) | vergil-tooling | #4608 | Floor `>=3.15`, ruff/mypy targets, `repository-type = "tooling"` | 2026-11-17 |
| [#4653](https://github.com/vergil-project/vergil-tooling/pull/4653) | vergil-tooling | #4578 | Documentation review: remaining install docs, stale README CI example | 2026-11-17 |
| [#426](https://github.com/vergil-project/vergil-vm/pull/426) | vergil-vm | #425 | VM architecture doc names 3.15; test header rewrapped | 2026-11-17 |
| [#129](https://github.com/vergil-project/docs/pull/129) | docs | #128 | Install table names the interpreter | 2026-11-17 |
| [#1296](https://github.com/vergil-project/vergil-actions/pull/1296) | vergil-actions | #1295 | Smoke workflow comment points at the documented `only-system` exemptions | 2026-11-17 |
| [#1296](https://github.com/vergil-project/.github/pull/1296) | .github | #1295 | Spec amended: how vergil-tooling reaches one series | 2026-11-17 |

- **Repos touched:** 5 (.github, vergil-actions, vergil-tooling, vergil-vm, docs).
- **Tasks:** 12 children, including this retrospective. 10 PRs, plus one
  deployment task (vergil-actions#1279) closed on verification.
- **Releases:** vergil-actions v2.1.80 carried #1283, and `v2.1` was moved to
  v2.1.81 (#1279). vergil-tooling v2.1.322 carried #4622. vergil-vm v2.1.53
  carried #421. The floor raise (#4647) merged after v2.1.322 and is **not yet
  in a vergil-tooling release**.
- **Span:** planning issue opened 2026-11-09; epic opened 2026-11-17; every
  implementation PR merged on 2026-11-17.

## 1. How the plan evolved

The plan was approved after a scope reset, and the tasks ran in order. Its
assumptions broke in three places, all in the last code task.

**The floor did not narrow CI.** The spec said the CI matrix and
`vrg-validate --all-versions` follow `requires-python`. They don't. The
resolver ignores `requires-python` except to warn, and picks versions by
`repository-type`: a `library` builds the whole window, every other type builds
only the newest version. With the floor at 3.15, #4608's CI would have run four
`unit` legs, three of them failing at `uv sync`.

The first ruling was a `[ci].versions = ["3.15"]` pin, with a resolver change
filed to follow (#4639). Merge review asked about the pin again (R1), and on a
second look a better answer turned up: retype vergil-tooling. Epic #407 had made
it a `library` because consumers installed it with a bare `uv tool install` on
whatever window version they built. This epic removed that reason, since every
install path now names its own 3.15 interpreter. So R1 was ruled `change`: no
pin, `repository-type = "tooling"`. #4639 was closed as not planned, because
narrowing a library's matrix by `requires-python` would hide breaches of the
library policy. No ruleset change was needed: the required checks are the
version-agnostic `<kind> / evidence` gates from epic #338.

**The ruff rewrites went further than planned.** At the `py315` target,
`ruff format` rewrote 37 `except (A, B):` clauses to the 3.14+ form. Python
3.15's new `sentinel` builtin made seven test names shadow it, so they were
renamed. The plan's `vrg-container-run -- ruff …` has to be
`vrg-container-run -- uv run ruff …` in vergil-tooling, because `ruff` isn't on
the dev image's PATH.

**One task split off.** Task 6's `--select UP` run touched `scripts/perf/`,
which had lint errors of its own that nobody had seen, because `vrg-validate`
lints only `src/` and `tests/`. That clean-up went to its own ad-hoc task
(vergil-tooling#4640). Its lint-gate half needs a fleet-wide setting and went to
intake (vergil-tooling#4652).

**Smaller changes:**

- Task 2 needed an exemption the plan didn't foresee, in vergil-actions'
  `pin.test.sh`, so that the host self-repo sync could run without
  `only-system`. It covers that exact line and nothing else.
- The documentation review turned up two stale docs beyond its recorded list
  (vergil-vm `architecture/index.md`, docs `packages.md`).
- The human dropped #1279's check that the GitHub release notes list the PR.
  The task had been written with a requirement nobody had checked:
  vergil-actions release pages carry only a docs link, and the PR list is in
  `releases/vX.Y.Z.md` and `CHANGELOG.md`.

The plan's `## Evolution during execution` log, added in this PR, records
the same points as dated entries.

## 2. Lessons learned

- **Check a mechanism's premise in the code, not in its docs.** The key
  sentence in the spec ("CI follows `requires-python`") was never checked
  against `resolve.py`. Three spec pushback rounds and a plan round went over
  install paths in fine detail and never asked how the matrix is chosen. One
  `grep` would have found it. Next time: for every "X follows Y" claim in a
  spec, cite the line of code that makes it true.
- **Look again at old classifications when their reason goes away.** Epic #407
  made vergil-tooling a `library` for a reason that this epic deleted. The
  brainstorm noticed the question but put it off (#1260) because nobody knew
  what else `repository-type` controls. When it was finally checked, it turned
  out to control version resolution and almost nothing else. Next time:
  checking a field's consumers takes a few minutes, so do it before deferring
  a question that depends on it.
- **The scope reset was the right call.** Spec pushback grew the design for
  three rounds, adding a packaged-Python rule, floor-reading logic, a drift
  guard and live checks. In the first plan round, the human reset it to the
  original purpose. The short plan that resulted shipped in a day, and the one
  real gap it had (the resolver) wasn't one the dropped mechanisms would have
  caught.
- **Steps written from assumption create false work.** #1279's release-notes
  check described how releases ought to look, not how they do look.
  Verification steps should be written against the actual artifact.
- **Merge review's second question paid off.** R1 asked again about a choice
  the human had already ruled on. That looked redundant, but it is what led to
  the better answer.

## 3. Compromises and tradeoffs

- **The install paths hard-code 3.15, but CI follows the newest series.** When
  vergil-python publishes 3.16, vergil-tooling's CI moves to 3.16 while the
  installs still say `--python 3.15`. That is deliberate: fail forward. CI
  finds out first whether 3.16 works, and moving the installs and the floor is
  a separate, deliberate change, tracked by #1261. Until then, the series CI
  tests and the series that runs in production can differ.
- **No live check of the rewritten install paths.** The scope reset dropped
  the live checks of the host dev-pin and index-rebuild paths. Coverage relies
  on `install-tooling.test.sh`, the smoke workflow, and #1279's check that
  vergil-tooling's own CI host jobs synced on 3.15 through v2.1.81.
- **A pre-existing test fails locally.** `missing_pin_fails` in
  `install-tooling.test.sh` fails inside the validation container because
  `python3` is missing there. Local validation leaves that suite to CI, so it
  is checked only in CI.
- **The `only-system` rule has a third exemption.** It is narrow, tested to
  apply nowhere else, and documented in `ci-toolchain-pin-python.md`.
- **The floor raise is merged but not released.** Consumers keep receiving
  vergil-tooling releases that allow `>=3.12` until the next vergil-tooling
  release. The next release's consumer refresh already installs with
  `--python 3.15`.

## 4. New problems and opportunities

| Item | Where it went |
| --- | --- |
| Resolver honouring `requires-python` | vergil-tooling#4639, filed and closed as not planned (see §1) |
| `scripts/perf/` lint clean-up | vergil-tooling#4640 (ad hoc; waiting on a merge-review reply) |
| Per-repo extra lint paths in `vergil.toml` | vergil-tooling#4652 (intake) |
| vergil-tooling's branching and release model (`library-release`) | .github#1260 (deferred); a note records that the type question is settled |
| A new series: move the installs and raise the floor | .github#1261 |
| Two definitions of the core Python series | .github#1267 |
| `vrg-host` handling of vergil-tooling | vergil-tooling#4603 |
| Release-notes check in deployment steps | dropped by the human: an invented requirement |

**Unaddressed-findings sweep.**

- **What was swept:** all 11 children other than this retrospective, the epic
  issue's comments, the planning issue's round comments (#958), and the 10
  merged PRs.
- **Rulings:** every `Awaiting a ruling` item has a trusted ruling (#4608
  N0.1–N0.3, and R1 recorded as changed and fixed; the epic's release-notes
  item recorded as dropped), and every `file` ruling's issue exists.
- **Planning rounds:** each finding not accepted gives its reason or an issue
  (S2.5 → vergil-tooling#4603, S3.6 judged moot, P1.1–P1.6 dropped with the
  scope reset).
- **PR reviews:** none of the 10 PRs has a review or a conversation comment.

Untracked problems found, with the human's rulings:

1. **A dead reference.** The spec's out-of-scope list and the epic body cited
   `vergil-project/.github#1266` for "`vrg-host` handling of vergil-tooling".
   No such issue exists in .github (vergil-tooling#1266 is an unrelated bug);
   the intended issue is vergil-tooling#4603. **Found:** the reference is
   corrected to vergil-tooling#4603 in the spec (this PR) and in the epic body.
2. **`scripts/dev/` isn't linted**, and nobody has checked whether it's clean.
   **Found:** vergil-tooling#4652 tracks the mechanism; a comment there records
   `scripts/dev/` as a candidate path.
3. **`missing_pin_fails` fails in the validation container** (§3).
   **Dropped** by the human: local validation leaves that suite to CI by
   design.

## 5. What's next

- **Release:** the next vergil-tooling release ships the floor raise.
- **When a new series arrives:** #1261, detecting it and prompting the move of
  the installs and the floor.
- **Deferred:** #1260 comes back from the deferred queue when this epic
  closes, narrowed to the branching and release model.
- **Not started:** #1267 and vergil-tooling#4603 are open follow-ups; no work
  has started on them.
