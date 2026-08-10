# Retrospective — repo-specific system dependencies in dev containers

> **Epic:** vergil-project/.github#272 · **Spec:** `epics/272-container-system-packages/spec.md` · **Plan:** `epics/272-container-system-packages/plan.md`
> Reading order: **spec → plan → retrospective**.

## §0 At a glance

We set out to let a repository declare Debian system packages in `vergil.toml`
(`[container].system-packages`) and have them installed identically into the
local cached dev image and CI test jobs — without polluting the shared base
images. That is exactly what shipped: a single config key, one reader and one
apt speller shared by both install paths, fail-closed on a missing candidate,
proven end-to-end on a live cold rebuild, and documented on both the
consuming-repo and CI-operator sides.

**Work delivered**

| PR | Repo | What it did |
|---|---|---|
| [.github#276](https://github.com/vergil-project/.github/pull/276) | .github | Publish the epic spec + plan (documentation bookend, task #273) |
| [vergil-tooling#2732](https://github.com/vergil-project/vergil-tooling/pull/2732) | vergil-tooling | The mechanism: `[container].system-packages` parsing, the `apt_install_command` speller + cached-image bake, and the `vrg-container-system-packages` accessor (task #2730) |
| [vergil-actions#808](https://github.com/vergil-project/vergil-actions/pull/808) | vergil-actions | `shared/setup/system-packages` composite action; installs declared packages on CI **test jobs only**, bounded retry, fail-closed (task #807) |
| [vergil-tooling#2736](https://github.com/vergil-project/vergil-tooling/pull/2736) | vergil-tooling | Consuming-repo docs: Container Config Reference + CI Architecture guide (documentation-review bookend, task #2724) |
| [vergil-actions#813](https://github.com/vergil-project/vergil-actions/pull/813) | vergil-actions | CI-operator docs: Action Reference page + ci-test note (task #812, spawned by the docs sweep) |

**By the numbers**

- **Repos touched (delivery):** 3 — `vergil-project/.github`, `vergil-tooling`, `vergil-actions`.
- **Children:** 9 total — 5 PR-workable tasks (all merged), 1 `validation` operational task (#2725, closed on `Outcome: SUCCESS`), 1 origin research issue (#2718, closed as promoted), 1 forward-axis follow-on bookend (#274), and this retrospective (#275).
- **Releases cut:** 2 — vergil-tooling **v2.1.186** (accessor + bake; the `v2.1` rolling tag consumers install) and vergil-actions **v2.1.22** (the CI step).
- **Span:** opened 2026-08-10 15:18Z; reached the retrospective the same day. Epic closes when this PR merges.

## §1 How the plan evolved

The plan held almost exactly. It sequenced the work as **Issue A** (tooling:
config + bake + accessor, one PR) → **Issue B** (vergil-actions CI step, one PR,
`blocked-by` A *released*) → **cold-rebuild validation** (blocked-by both) → the
documentation bookend. Execution followed that spine with no re-planning.

The delta was small and of the anticipated kind:

- **The adoption gate behaved as designed.** Issue B genuinely could not proceed
  until A was not merely merged but **released** — vergil-actions installs
  vergil-tooling from the rolling `v2.1` tag, so its CI could not see
  `vrg-container-system-packages` until v2.1.186 shipped. The human release gate
  sat exactly where the plan's §5 sequencing said it would; no surprise, no
  workaround.
- **The documentation-review ran as a sweep, not a single edit.** It found the
  vergil-actions operator side undocumented and **spawned a same-repo doc task
  (#812)** rather than reaching across a repo boundary — the placement law
  playing out as intended. That was one more child than the plan enumerated up
  front, which is the expected shape of the sweep, not drift.
- **The forward-axis follow-on collapsed to a task.** #274 was seeded as a
  "create the follow-on epic" bookend; on brainstorming, the melete adoption
  proved to be a single logical change, so it was filed as a **task** (melete#51)
  under melete's ad-hoc epic instead of a standalone epic.

The plan carried no "Evolution during execution" log because there was nothing to
record mid-flight — the tasks landed as written. A small delta is the intended
outcome, and this is a small delta.

## §2 Lessons learned

- **One reader, one speller made CI parity nearly free.** Routing every install
  path through `config.container_system_packages` and `apt_install_command` meant
  the local bake and the CI step could not diverge on *what* gets installed —
  they differ only on *when* (baked per-branch vs. per-run). This is the design
  choice most responsible for the epic being small.
- **Leaning on the existing cache-key surface avoided new machinery.** Because
  `vergil.toml` is already cache-sensitive, changing the package list already
  forces a rebuild — no cache code was needed, only a guard test locking the
  invariant. Reusing an existing invariant beat inventing a new one.
- **Release-before-consume is a first-class cross-repo constraint.** The
  `report-ready` → human-submit → merge → **release** → downstream-consume chain
  is the real critical path in a multi-repo mechanism epic, not the code. Naming
  it in the plan up front is what kept Issue B from being started prematurely.
- **Documentation-review-as-sweep earns its keep.** Treating docs as a
  multi-repo sweep (not one PR) is what surfaced the vergil-actions operator gap;
  a single-PR docs task would have silently left it undocumented.

## §3 Compromises & tradeoffs

- **The new vergil-actions action tests are developer-run, not CI-gated.**
  `shell`-language repos have no `test` command in the tooling's language
  registry, so `vrg-validate --check test` runs nothing in vergil-actions and the
  `install.sh` bash harness is not enforced in CI. Accepted for this epic and
  logged (see §4); closing it is cross-repo tooling work, not in scope.
- **Pipeline shellcheck is scoped to `scripts/`.** `install.sh` and its harness
  live under `actions/` and are therefore not linted by the pipeline (verified
  clean manually). Left as a small follow-up rather than widening the glob here.
- **Validation used a small stand-in package.** The cold-rebuild validation
  (#2725) proved the mechanism with `figlet` (small, fast, absent from the base
  image) rather than `lilypond`. It exercises the identical code path; the
  `lilypond`-specific proof is deferred to melete#51's own acceptance.
- **Package-names-only trust model.** No third-party sources, keys, or scripts,
  and no curated allowlist — the trust surface is "Debian-main names, reviewed in
  the PR diff." A deliberate scope floor, not an oversight; richer provisioning
  is explicitly out.

## §4 New problems & opportunities

- **`shell`-language repos run no CI-gated test check.** Surfaced while adding
  the vergil-actions action tests. *Where it went:* logged here, not yet acted
  on — candidate vergil-tooling follow-up (a `shell` `test` command that
  discovers and runs `*.test.sh`).
- **shellcheck glob excludes `actions/**`.** Surfaced in the same PR. *Where it
  went:* logged here, not yet acted on — a small vergil-actions config change.
- **melete's aarch64-wheels approach is obsolete.** Installing the Debian
  `lilypond` binary removes the need to fork the PyPI packaging. *Where it went:*
  melete#21 **closed as superseded.**

## §5 What's next

The mechanism was built specifically to unblock `mnemosys-project/melete`'s
LilyPond render path. That downstream consumer work is now captured as backlog in
melete's ad-hoc epic (`mnemosys-project/.github#12`) — cross-org, so intentionally
not linked to this epic:

- **melete#51** — adopt `[container].system-packages = ["lilypond"]` so melete's
  dev/CI containers carry the binary (the container capability; one PR).
- **melete#52** — implement the two LilyPond integration tests (spec §14) and
  enable `integration-tests`; **blocked-by** #51 (the tests need the binary
  first).
- **melete#21** — retired as superseded (see §4).

## Appendix A — Operational notes

This epic was multi-repo and release-ordered; the mechanical sequence was:

1. **vergil-tooling** — merge the mechanism (#2732), then cut release **v2.1.186**
   (moves the `v2.1` rolling tag consumers install).
2. **vergil-actions** — merge the CI step (#808) once the accessor is released,
   then cut release **v2.1.22**.
3. **Validation** — on an aarch64 Lima host, cold-rebuild a throwaway fixture
   declaring a real Debian-main package; confirm the binary is on `PATH` and that
   a bogus name fails the build loudly, naming the package and `linux/arm64`
   (#2725, `Outcome: SUCCESS`).
4. **Docs** — consuming-repo docs in vergil-tooling (#2736); CI-operator docs in
   vergil-actions (#813), spawned by the sweep.

Gotcha worth remembering: the release step between the two implementation PRs is
load-bearing — vergil-actions CI resolves `vrg-container-system-packages` only
from the released `v2.1` tag, so merging Issue B before the vergil-tooling release
would have failed CI with "command not found."
