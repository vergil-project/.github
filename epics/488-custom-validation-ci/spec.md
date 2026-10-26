# Run repos' custom validation in PR CI

- **Epic:** vergil-project/.github#488
- **Planning issue:** vergil-project/.github#476
- **Status:** Reviewed design (2026-10-26), after pushback
- **Brainstorm source:** superpowers brainstorming session, 2026-10-26
- **Review:** paad pushback, 2 issues, both resolved. The epic is now
  fleet-wide across the active orgs, and vergil-containers' duplicate pins
  job is removed.
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
vergil-containers, vergil-python and vergil-vm. They are the only custom
validators in the active fleet (§3.4). Whatever their custom
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
- Every repo in the active fleet (§3.4) with a custom validator is green
  under it.
- The docs stop claiming parity that doesn't exist, and describe the custom
  stage as part of CI.

**Non-goals**

- No new required status check, and no change to branch protection or
  rulesets. The stage folds into the existing `quality / evidence` gate.
- No change to how a repo defines its custom validator, or to discovery order
  (the PATH entry point, then `scripts/bin/validate-custom`).
- No per-version matrix for the custom stage. It runs once.
- No redesign of the five validators beyond what each needs to pass in CI,
  apart from the two cleanups in §3.4 (vergil-vm's stale `tofu` skip, and
  vergil-containers' duplicate pins job).
- No work in on-hold orgs (diogenes-project, mnemosys-project,
  vergils-nemesis). They need a larger revival before any of this applies.
  diogenes-project/diogenes has a custom validator, but it is pinned to v2.0,
  so this change never reaches it.

**Anti-goals** (failures even if the mechanism technically works)

- **A silent pass.** A validator that is present but not executable, crashes,
  or can't find a tool it needs must fail CI. It must never be reported green
  or skipped as if the repo had none. This is the existing #3349 rule, and it
  carries into CI unchanged.
- **Green-by-absence.** If the custom job fails or is skipped, the
  `quality / evidence` gate must fail. A missing job must never let the gate go
  green.
- **A new environment difference.** CI quality jobs run on the prod image,
  while local `vrg-validate` runs in the dev container. That split already
  applies to every stage. The custom job must run on the same image as the
  `common` job, so it adds no environment difference of its own.

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
  they are in the dev container. Next come the Python interpreter pin
  (`pin-python`, with the primary version) and `uv sync --group dev --frozen`
  when `language == 'python'`, mirroring lint, so a Python repo's validator
  sees its dev dependencies. The `setup` job currently exports only
  `primary-container-tag` and `primary-container-suffix`, so it must also
  export `primary-version` for that pin. Then:

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

**Assumption: vergil-tooling has a single deployed version.** Every user of
the fleet is its one owner, and a vergil-tooling release replaces every active
install. That covers the host `uv tool install` at the end of the release, the
container install done on each container run, and the VM install after a
release. No older vergil-tooling stays in use, so the CI job needs no guard
against tooling that lacks `--check custom`.

Every active consumer calls the reusable workflows through the floating `@v2.1` tag. A
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

### 3.4 Fleet scope and per-repo validation

**The update covers the whole active fleet.** The active orgs are
vergil-project, logical-minds-foundry and mq-rest-admin-project. Every repo in
them, including each `.github`, is checked out under
`~/dev/projects/<org>/<repo>`, and that tree is the inventory source. GitHub
code search is not used, because it is unreliable here: it finds only 2 of
the 5 known vergil-project validators. A repo has a custom validator when it
has `scripts/bin/validate-custom`, or declares a `vrg-validate-custom` entry
point.

The inventory at design time (2026-10-26) found the five vergil-project repos
and nothing in logical-minds-foundry or mq-rest-admin-project. The deployment
task re-runs the inventory just before the vergil-actions release, so a
validator added in the meantime is not missed.

**Tracking across orgs.** A vergil-project repo's tasks link under #488
directly. A repo in another active org gets its validation task under that
org's ad-hoc epic, filed in that repo, and is weakly linked from #488 by a
comment. GitHub cannot link sub-issues across orgs. The epic is not done until
every such task, in any org, has closed with SUCCESS.

Each repo with a custom validator gets its own **validation** task, filed in
that repo and blocked by the deployment task. It passes when a real PR CI run
in that repo, after the release, shows all of the following:

- the `custom` job ran the repo's validator (named in the log, not the
  no-validator line) and succeeded;
- `quality-custom.log` is present in the `ci-evidence-quality` artifact;
- `quality / evidence` is green.

A failing repo gets a fix task, filed in that repo under the epic. The
validation task stays open until the fix lands and a later CI run passes.

Per-repo requirements beyond passing:

- **vergil-vm:** the base image includes OpenTofu
  (`docker/base/Dockerfile.template` includes `common/opentofu.dockerfile`),
  so the script's `SKIP: tofu not installed (vergil-docker#352)` branch is
  stale. A missing `tofu` must be a hard error, not a skip that still exits 0.
  That is the "no silent pass" anti-goal. Validation fails if the log shows the
  `SKIP` line, and the fix changes that branch to fail.
- **vergil-containers:** its `quality / pins` job in `ci.yml` runs exactly the
  steps its custom validator runs. It existed only because the custom stage
  never ran in CI. Once a PR shows the custom job running all five checks
  green, remove the `quality / pins` job and the validator's "keep this list
  in step with the pins job" comment, so there is one copy of the checks.

Risk to watch: **vergil-python** and **packages** need `vrg-package`. The
setup action's vergil-tooling install should provide it.

## 4. Done when

- `vrg-validate --check custom` is released in vergil-tooling.
- The `custom` job is released in vergil-actions under `v2.1` and folded into
  `quality / evidence`.
- Every validation task in the active fleet, the five known ones plus any the
  pre-release inventory adds, has closed with `Outcome: SUCCESS`, and any fix
  tasks they spawned have merged. That includes vergil-vm's hard failure on a
  missing `tofu` and the removal of vergil-containers' `quality / pins` job.
- The docs review has landed. It covers `docs/site/docs/guides/ci-architecture.md`,
  `docs/site/docs/guides/validation-matrix.md`,
  `docs/site/docs/guides/ci-evidence-convention.md`, the vergil-actions
  workflow docs, and the "local matches PR CI" claims.
