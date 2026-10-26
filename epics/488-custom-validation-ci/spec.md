# Run repos' custom validation in PR CI

- **Epic:** vergil-project/.github#488
- **Planning issue:** vergil-project/.github#476
- **Status:** Draft design (2026-10-26)
- **Brainstorm source:** superpowers brainstorming session, 2026-10-26
- **Implemented in:** vergil-tooling, vergil-actions, then fixes in each
  consumer repo that has a custom validator
- **Found while implementing:** vergil-project/packages#22

## 1. Summary

A repo can supply its own validation through a **custom validator**, which is
either `scripts/bin/validate-custom` or `vrg-validate-custom` on `PATH`.
`vrg-validate` runs it as the final `custom` stage, but **only when it is called
with no `--check`**. PR CI never calls it that way. The vergil-actions reusable
workflows run `vrg-validate --check common|lint|typecheck|test|audit`, and
`--check` has no `custom` choice. So no repo's custom validator has ever run in
PR CI.

Five vergil-project repos have one today: packages, vergil-claude-plugin,
vergil-containers, vergil-python and vergil-vm. Whatever their custom
validators catch can be merged by a PR that skipped local validation. The
documented claim that local `vrg-validate` matches PR CI is false for this
stage. This is a long-standing gap in the validation model, not a corner case.

This epic makes the custom stage **one more component of CI validation**,
treated exactly like common, lint and typecheck. It runs in PR CI, its
transcript goes into `quality / evidence`, and that existing gate fails when it
fails. The epic closes only when every repo with a custom validator passes the
stage in real PR CI.

## 2. Goals, non-goals, anti-goals

**Goals**

- `vrg-validate --check custom` runs the custom stage on its own.
- The vergil-actions quality workflow runs it on every PR for every consumer.
- Its transcript is a quality evidence report, and its result gates
  `quality / evidence`.
- Every vergil-project repo with a custom validator is green under it.
- The docs stop claiming parity that doesn't exist, and describe the custom
  stage as part of CI.

**Non-goals**

- No new required status check, and no change to branch protection or
  rulesets. The stage folds into the existing `quality / evidence` gate.
- No change to how a repo defines its custom validator, or to discovery order
  (the PATH entry point, then `scripts/bin/validate-custom`).
- No per-version matrix for the custom stage. It runs once.
- No redesign of the five validators beyond what each needs to pass in CI.

**Anti-goals** (failures even if the mechanism technically works)

- **A silent pass.** A validator that is present but not executable, crashes,
  or can't find a tool it needs must fail CI. It must never be reported green
  or skipped as if the repo had none. This is the existing #3349 rule, and it
  carries into CI unchanged.
- **Green-by-absence.** If the custom job fails or is skipped, the
  `quality / evidence` gate must fail. A missing job must never let the gate go
  green.
- **Different environments.** The custom stage in CI must not run somewhere
  that differs from where it runs locally, so that a repo is never green
  locally and red in CI, or the reverse, for environmental reasons alone.

## 3. Design

### 3.1 vergil-tooling — `vrg-validate --check custom`

- Add `custom` to the `--check` choices in `bin/vrg_validate.py`.
- `--check custom` builds two stages: the language **install** stage, when the
  repo's primary language defines install commands, and then the existing
  `custom` stage. This is the same environment a full run gives the custom
  stage locally, since that run executes install first. It runs no lint,
  typecheck, test, audit or common stage.
- **No custom validator:** `--check custom` prints one clear line, such as
  `custom: no custom validator (scripts/bin/validate-custom or vrg-validate-custom) — nothing to run`,
  and exits 0. A repo without a validator is the normal case, and CI must pass
  for it.
- **A validator that is present but unusable** (not a regular file, or not
  executable) fails loudly, through the existing `_check_custom_validator`
  path. A non-zero exit fails the stage with `custom validator exited <rc>`.
- A no-argument run behaves exactly as before: custom is still last, and is
  still omitted when there's no validator.
- `--all-versions` stays restricted to `test`. `--check custom --all-versions`
  remains a usage error.

### 3.2 vergil-actions — a `custom` job in `ci-quality.yml`

The job is modelled on the existing **`common`** job, not the matrix legs:

- `name: custom`, with `needs: setup`. It runs once, in the **primary**
  container
  (`<prefix>-<primary-container-suffix>:<primary-container-tag>`), which is
  the same image `common` uses.
- Steps: checkout, then the shared `setup/vergil` action. That action installs
  vergil-tooling, so `vrg-package` and the other `vrg-*` tools are on PATH as
  they are in the dev container. Next come the Python interpreter pin and
  `uv sync --group dev --frozen` when `language == 'python'`, mirroring lint,
  so a Python repo's validator sees its dev dependencies. Then:

  ```bash
  set -euo pipefail
  mkdir -p ci-reports
  vrg-validate --check custom 2>&1 | tee ci-reports/quality-custom.log
  ```

- Upload `ci-reports/` as the partial `ci-evidence-quality-reports-custom`,
  with `if: always()`.
- `evidence` job: add `custom` to `needs:` and to the
  `Assert all required legs succeeded` results list. Add
  `evidence-reports/quality-custom.log` to the emitted files, and
  `{"name": "vrg-validate-custom"}` to `tools`.
- Check whether `ci-quality.yml` and its callers declare the job list or the
  evidence contract anywhere else, such as the README, a smoke test, or a
  harvester's expected-file list. Any that do are updated in the same change.

### 3.3 Release and rollout — fail forward

All consumers call the reusable workflows through the floating `@v2.1` tag. A
vergil-actions release therefore makes the custom stage **required in every
consumer at once**. We accept that deliberately. This is a scale-of-one
environment with no concurrent contributors to block. A short window in which
some of the five repos are red, followed by an afternoon of fix PRs, costs less
than proving each repo green against an unreleased workflow first.

Sequence:

1. vergil-tooling implementation merges.
2. **Release vergil-tooling.** This comes first, because the CI job calls
   `--check custom` and the installed tooling must support it.
3. vergil-actions implementation merges.
4. **Release vergil-actions** under `v2.1`. From here the stage is live and
   required.
5. Validate each consumer. Fix the ones that fail.

Releases are **human-gated**. The deployment task that carries steps 2 and 4
records them as attested preconditions. The agent never cuts a release.

### 3.4 Per-repo validation

Each repo with a custom validator gets its own **validation** task, filed in
that repo and blocked by the deployment task. It passes when a real PR CI run
in that repo, after the release, shows all of the following:

- the `custom` job ran the repo's validator (named in the log, not the
  no-validator line) and succeeded;
- `quality-custom.log` is present in the `ci-evidence-quality` artifact;
- `quality / evidence` is green.

A failing repo gets a fix task, filed in that repo under the epic. The
validation task stays open until the fix lands and a later CI run passes.

Known risks to check during validation (these are guesses, not findings):

- **vergil-vm** prints a `SKIP` and still passes when `tofu` isn't installed.
  Validation must record whether the CI image has `tofu`. If it doesn't, the
  HCL checks are still not enforced in CI. That gap is tracked by
  vergil-docker#352 and must be reported, not hidden.
- **vergil-python** and **packages** need `vrg-package`, which the setup
  action should provide.
- **vergil-containers** already runs the same checks in its own
  `quality / pins` job. Running them in both places is redundant but not
  broken.

## 4. Done when

- `vrg-validate --check custom` is released in vergil-tooling.
- The `custom` job is released in vergil-actions under `v2.1` and folded into
  `quality / evidence`.
- All five validation tasks have closed with `Outcome: SUCCESS`, and any fix
  tasks they spawned have merged.
- The docs review has landed. It covers `docs/site/docs/guides/ci-architecture.md`,
  `docs/site/docs/guides/validation-matrix.md`,
  `docs/site/docs/guides/ci-evidence-convention.md`, the vergil-actions
  workflow docs, and the "local matches PR CI" claims.

## 5. Open questions for review

1. **Consumers outside vergil-project.** Repos in other orgs also call
   `vergil-actions@v2.1`. Any of them with a custom validator becomes required
   at the same release. Cross-org linking is out of scope for this epic. Should
   validation still check those repos, outside the epic, or is that their
   owners' concern?
2. **vergil-containers `quality / pins`.** Once the custom stage runs in CI,
   should the bespoke pins job be removed under this epic, or left to a
   follow-on?
