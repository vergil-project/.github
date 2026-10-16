# Release-integrated dependency updates

Epic: `vergil-project/.github#381`. Documentation task: `vergil-project/.github#382`.
Supersedes: `vergil-project/.github#52` (Mechanize dependency-update) and its
tracking issue `vergil-project/vergil-tooling#1590`.
Prior art: `vergil-tooling/docs/specs/2026-06-10-vrg-update-deps-design.md`
(the `vrg-update-deps` design; Phase 1 shipped in
`vergil-project/vergil-tooling#1379`). See §10 for what this spec replaces.

## 1. Summary

The fleet has a deterministic dependency updater, `vrg-update-deps`, but nothing
runs it. It is a human-only command that someone has to remember, so dependencies
drift. The one scheduled updater that does exist, vergil-containers'
`bump-tools`, shows the second half of the problem: its bump PR has sat unmerged
for three months.

This epic makes dependency updating **part of every release**:

- every `vrg-release`, including routine patch releases, ends by upgrading the
  repo's dependencies to the newest versions their **declared bounds** allow, as
  the first change on the new develop version;
- a failed update never fails the release, but it **cannot be ignored**: it
  leaves an open PR, and the next release refuses to start until that PR is
  dealt with;
- each repo can add its own update logic through a **hook**, so tooling stays
  free of repo-specific code;
- every run reports what is **held back** by the declared bounds, so staleness
  is visible even when nothing changes.

## 2. The problem in detail

Measured across the nine repos checked out on 2026-10-15:

- **Only one repo has a language manifest.** vergil-tooling is the only repo
  with `primary-language` set. For the other eight, everything that goes stale
  is something other than a language package: vergil ecosystem refs, image tags,
  binary tool versions, OpenTofu providers, the packaged CPython runtime.
- **`vrg-update-deps` covers two categories.** The `python` updater runs
  `uv lock --upgrade`; the `vergil` updater normalises `vergil-*` workflow refs
  and the Claude marketplace ref. Neither is invoked by any workflow, cron or
  release step.
- **Known gaps in the `vergil` updater.** It only scans
  `.github/workflows/*.yml`, missing `action.yml` files. `repo_init.py`
  hard-codes `@v2.1` in new-repo templates. `vergil-vm/scripts/vrg-vm-init.sh`
  hard-codes `vergil-tooling@v2.1`. vergil-tooling's
  `[package.python] runtime = "3.14.8"` is an exact pin nobody bumps.
- **Drift is already visible.** vergil-actions' trivy image default is `0.71.0`;
  the containers ship `0.72.0`, and `0.74.0` (a CVE-fix release) is current
  upstream.
- **An unmerged PR is the same as no automation.** vergil-containers'
  `bump-tools` (from `vergil-containers#435`) resolves the 9 binary tools every
  Monday and refreshes a single bump PR. That PR (`vergil-containers#445`) has
  been open since 2026-07-20. `vergil-containers#573` records an earlier stall
  that left trivy a month behind a CVE fix.

## 3. Principles

1. **Stay at the leading edge within declared bounds.** The upgrade target is
   the newest version the repo's own constraints allow, every release.
2. **Bounds live in the ecosystem's native manifest.** `pyproject.toml` ranges,
   `[tool.uv]` constraints, OpenTofu `~>`, vergil-containers' `pins.yml`. The
   tool never duplicates them in `vergil.toml`. To hold a dependency back,
   constrain it where its ecosystem expects.
3. **The release never fails because of dependencies.** The update runs after
   the release is complete, and its failure is deferred, not fatal.
4. **A failed or pending update is never silent.** A failure that leaves work
   behind (local validation or remote CI) leaves an open labelled PR and a fix
   task, and the next release is gated on the PR. A failure before validation
   has no diff to gate on, so it is recorded on the tracking issue of every
   release until it is fixed (§5.2).
5. **Tooling carries no repo-specific logic.** A repo with bespoke dependencies
   owns its own hook; tooling only knows how to run it.
6. **Remove a pin before automating it.** If a pin can read its source of truth
   at runtime, change it to do that rather than keeping a bumper for it.
7. **No silent failures.** A resolution or hook failure fails loud. No fallback
   may make a broken lookup look like "no change".

## 4. Dependency taxonomy

Every pinned or referenced dependency in the fleet falls into one of three
tiers, by how it can be mechanized.

| Tier | Meaning | In this epic |
|---|---|---|
| 1. Built-in updater | Generic: any repo with the surface gets it | `python`, `vergil`, `images` |
| 2. Repo hook | Repo-specific: the repo owns the command | vergil-vm, vergil-python |
| 3. Report only | A judgement call, not a mechanical bump | The held-back report (§7) |

Tier 3 covers new language or runtime majors (`[ci].versions`, `requires-python`,
the image matrix), LTS rollovers (Ubuntu 24.04 to 26.04), and newer image series.
These are reported, never bumped.

**Not managed (floating).** Unversioned installs such as `apt-get install`
without versions, `npm install -g x@latest`, or `go install ...@latest` are
refreshed by vergil-containers' nightly `no-cache` rebuilds. They need no updater.

**Out of scope.** See §11.

## 5. Release integration

### 5.1 The `update-deps` stage

`vrg-release` gains one stage, `update-deps`, placed after `consumer-refresh` and
before `publish-status`:

```text
audit → preflight → prepare → merge-release → confirm-main → back-merge-bump
  → teardown-worktree → confirm-develop → promote → close-finalize
  → package-index → consumer-refresh → update-deps → publish-status
```

- **Mode: `fail_defer`**, like every stage after `back-merge-bump`. The release
  is already tagged, promoted, published and refreshed, so nothing about the
  release depends on this stage.
- **What it runs:** the existing `vrg-update-deps` pipeline (preflight → update →
  validate → PR → merge → cleanup) in its own managed worktree off develop.
  Develop is on the new patch version at this point, so the update is the first
  change on X.Y.Z+1.
- **Root checkout precondition:** the `vrg-update-deps` preflight requires the
  root checkout to be on a clean develop that is in sync with `origin/develop`.
  The stage fetches and fast-forwards develop before invoking the pipeline. If
  that sync fails, the stage fails (deferred), and nothing is updated.
- **Branch name:** `deps/update-<new-develop-version>` (for example
  `deps/update-2.1.224`), replacing the current `chore/dep-update-<date>`.
  Repos release several times a day, so a date-based name collides. `deps/` is
  deliberately not an issue-branch prefix (`lib/branch_names.py`): only the
  driver commits to this branch, and it carries no issue number. Fixes happen
  on a separate issue branch (§5.2).
- **No-op:** if no updater changes anything, no branch is pushed and no PR is
  opened. The held-back report (§7) is still posted.
- **Tracking:** the stage takes part in the release tracking issue like any
  `_tracked` stage. Its completion comment carries the held-back report and,
  when a PR was opened, the PR link.
- **`--without-deps`** on `vrg-release` skips the stage. This is the
  "default-on, opt-out" end state that the 2026-06-10 spec planned to reach in
  two steps; this epic goes straight to it.
- **Identity:** `vrg-release` and `vrg-update-deps` are both human-run.
  Embedding one in the other changes nothing about who may run it.

`vrg-update-deps` remains available as a standalone command for off-cycle runs.

### 5.2 Failure handling

**Every dependency-update PR is labelled `dependency-update` when it is
created**, on every path. The label is created idempotently with
`vrg-ensure-label`, and is also added to the canonical label registry
(`data/labels.json`) so `vrg-ensure-label --sync` provisions it fleet-wide. The
gate (§5.3) therefore sees any open deps PR, whatever left it open.

A run can fail in a way that leaves work behind in two places:

- **local validation fails** (the single `vrg-validate` run), or
- **remote CI fails or the merge aborts** (`pr_merge.MergeAbortError`). This is
  expected to be the more common case: local validation runs one interpreter,
  while PR CI runs the full `[ci].versions` matrix.

Both are handled the same way. The 2026-06-10 spec abandoned the work locally;
this spec does not. The stage:

1. ensures the `deps/update-<version>` branch is pushed and its labelled PR is
   open (on a local validation failure it pushes and opens it now), and does not
   merge it;
2. files a **fix task** in the repo under its ad-hoc epic
   (`vrg-issue-create --epic adhoc`), titled
   `Fix failed dependency update for <version>`, linking the PR;
3. records the failure on the release tracking issue (`comment_phase_failed`)
   and in `publish-status`.

**Failure output.** The PR body and the fix task carry the **failing
`vrg-validate` stage's output** followed by `vrg-validate`'s closing summary,
plus the command that reproduces it on the branch
(`vrg-container-run -- vrg-validate`, or the failing CI job's link). The output
is bounded to the last 200 lines and at most 20,000 characters, with a marker
line saying what was dropped. It never carries the full output: GitHub rejects
bodies over 65,536 characters, and a rejected body would leave no PR for the
gate to see.

If no failing stage can be isolated from the output, the section falls back to
the raw tail of the combined output, under a heading that labels it as the raw
tail. A failed update that fails nothing else counts **once** in the release
summary: the update-deps stage fails, and publish-status repeats it as a
warning rather than a second failure.

> **Amended after implementation** (vergil-project/vergil-tooling#3283, PR
> vergil-project/vergil-tooling#3285). The original text said "the last ~200
> lines of the failing output". The raw tail of a whole validation run is
> mostly passing stages and container noise, and the cause can sit hundreds of
> lines earlier, so the shipped code isolates the failing stage instead. It
> also stopped a deps-only failure being counted twice.

**Fixing.** The fix task is ordinary framework work. Its branch,
`chore/<N>-dep-update-<version>`, is cut from the `deps/update-<version>` branch,
so it starts from the failed update. It is fixed and handed off like any other
task (`issue-implement`, `report-ready`, human submit and merge).

The fix PR is squash-merged, so GitHub never marks the deps PR merged; it stays
open with its label. `vrg-release` preflight clears it: an open
`dependency-update` PR whose body links a fix task (the `Fix task:` line) that
is **closed as completed** is closed by the preflight. Before closing it, the
preflight comments on the PR naming the fix task, and the release records the
closed PR in its prepare-stage tracking comment. A fix task closed any other
way (not planned, duplicate, or with no state reason) means the update did not
land, so the PR keeps blocking with a message that says so; what to do with it
is a human call.

Every fix task is looked up before any PR is closed. A lookup, comment or close
that cannot complete raises and stops the release, naming the state the PR was
left in, so nothing is half-cleared.

> **Amended after implementation** (vergil-project/vergil-tooling#3286, PR
> vergil-project/vergil-tooling#3287). The original text said the fix PR's merge
> marks the deps PR merged and clears the gate with no manual close. That is
> false under squash merges, which left every landed deps PR blocking the next
> release.

An updater or hook that fails *before* validation (for example a resolution
error or a non-zero hook exit) produces no diff to push. The stage records the
failure on the tracking issue and in `publish-status`, removes the worktree, and
opens no PR.

### 5.3 The release gate

`vrg-release` preflight refuses to start while the repo has **any open PR
labelled `dependency-update`**. That covers both a failed update and one still
waiting to merge. The refusal names each blocking PR and says how to clear it.

The gate runs on a **fresh start only**; `vrg-release --resume` skips it, since
the release it resumes already passed it. On a fresh start it runs in this
order:

1. **Auto-close.** Deps PRs whose fix task closed as completed are closed and
   recorded (§5.2, "Fixing"). They no longer block.
2. **Refuse on what remains.** Each remaining PR falls into one of three cases,
   and the refusal describes the one that applies:
   - **No fix task linked** (for example an update still waiting to merge):
     merge the PR, or close it deliberately.
   - **Fix task open:** land the fix task; preflight then closes the PR on the
     next release.
   - **Fix task closed but not completed:** the update did not land, so a human
     decides whether to merge the PR, close it, or reopen the work.

A lookup that cannot complete — listing the open deps PRs, or reading a fix
task's state — **fails closed**: preflight refuses with "refusing to release
blind" rather than treating an unknown answer as "nothing pending".

**`--allow-pending-deps`** overrides the gate, for an urgent release such as a
security hotfix. It bypasses only what still blocks after the auto-close step.
When it is used, preflight prints a warning, and the bypassed PRs are recorded
in the release's prepare-stage tracking comment. A stale dependency update can
therefore never block a fix, and bypassing the gate is never silent.

> **Amended after implementation** (vergil-project/vergil-tooling#3286, PR
> vergil-project/vergil-tooling#3287). Added the resume skip, the auto-close
> step that runs first, the fail-closed lookups, and the per-case advice. The
> bypass record moved into the prepare-stage tracking comment rather than a
> separate comment. The exact refusal wording is owned by
> vergil-project/vergil-tooling#3297, so this section describes the cases
> rather than quoting messages.

## 6. The per-repo hook

### 6.1 Configuration

A new optional table in `vergil.toml`, parsed and validated like the existing
`[validation]` and `[container]` tables (unknown keys are warned):

```toml
[dependency-update]
hook = "scripts/dev/update-deps.sh"   # repo-owned command
```

`hook` is the only key. It must be a non-empty string; an empty or non-string
value is a config error.

This table is distinct from `[dependencies]`, which declares *what* the repo
depends on. `[dependency-update]` configures *how updating behaves*.

### 6.2 Contract

- **When:** after every built-in updater, so the hook sees lockfiles the
  built-in updaters have already refreshed.
- **Where:**
  `vrg-container-run -- env VRG_DEPS_REPORT=/workspace/.vergil/deps-report.jsonl <hook>`,
  from the root of the managed worktree. The hook gets the same dev container
  and tools as validation. *Amended after implementation* (the hook
  runner as shipped in PR vergil-project/vergil-tooling#3263): the command is wrapped in `env` so `$VRG_DEPS_REPORT` reaches the hook inside
  the container, at the container path of a gitignored file in the worktree.
- **What it may do:** edit files in the worktree.
- **What it must not do:** commit, or move HEAD in any other way. The driver
  records HEAD before the hook and verifies it after; a moved HEAD fails the run
  loud.
- **Success:** exit 0. The driver detects changes with `git status`, and if any
  exist commits them as `chore(deps): repo hook`. They go through the same single
  `vrg-validate` run and PR as the built-in updaters' commits.
- **Failure:** a non-zero exit fails the run loud, with the hook's output
  (§5.2, pre-validation failure).
- **Reporting:** the driver sets `$VRG_DEPS_REPORT` to a file path. The hook may
  append held-back entries to it (§7.2).

Hooks are implemented as a hook-runner updater that is last in the registry.
It applies only when `[dependency-update].hook` is set.

### 6.3 Hooks delivered by this epic

**vergil-vm:**

- `tofu init -upgrade` per module, within the existing `~>` provider ranges,
  refreshing `.terraform.lock.hcl`;
- `NERDCTL_VERSION` in `cloud-init.yaml.skel`, bumped to the newest release
  **within its current major** (the same rule as exact image tags, §8.3), then
  regenerating `cloud-init.yaml` via `scripts/build-cloud-init.sh`. A newer
  nerdctl major is reported, not bumped;
- report entries for Lima and the NodeSource major. `minimumLimaVersion` is a
  floor, not a pin, so it is reported only when Lima's latest **major** is above
  the floor's major. A new Lima minor is not noise-worthy. NodeSource is reported
  when the newest Node LTS major is above the `setup_<N>.x` major.

The hook needs `tofu` in vergil-vm's dev container. vergil-containers already
installs it in every base image (`docker/common/opentofu.dockerfile`), so no
`[container]` change is expected. The implementation confirms this before
relying on it.

**vergil-python:** resolve the newest python-build-standalone release for the
current CPython minor, refetch its `SHA256SUMS`, and rewrite `runtime.toml`
(`cpython`, `pbs_tag`, per-arch sha256) and `vergil.toml [package]` together.
`packaging/build.sh` already fails if the two disagree. A new CPython minor is
reported, not bumped.

**vergil-containers: no hook in this epic.** Its 9 binary tools stay with the
existing `bump-tools` workflow. Reconciling the two is follow-on brainstorm
`.github#385` (§11).

## 7. The held-back report

### 7.1 What it is

Every run, including a no-op run, produces a report of dependencies where **a
newer version exists but the declared bounds did not allow it**. This is the
sanity check: it shows what the repo is behind on, and why.

It appears in the dependency-update PR body (when there is a PR) and in the
`update-deps` stage's completion comment on the release tracking issue
(always). A standalone `vrg-update-deps` run also prints it, a no-op run
included. An empty report says so explicitly.

### 7.2 Entry format

Each entry is one JSON object per line:

```json
{"source": "python", "name": "example-pkg", "current": "1.9.4", "latest": "2.1.0", "note": "pyproject: example-pkg>=1.0,<2"}
```

`source`, `name`, `current` and `latest` are required strings; `note` is an
optional string. Built-in updaters produce entries directly. A hook appends
lines to `$VRG_DEPS_REPORT`. A malformed line fails the run loud rather than
being dropped.

### 7.3 Entries produced in this epic

| Source | Held back when |
|---|---|
| `python` | A **directly declared** dependency's latest release is outside its `pyproject.toml` / `[tool.uv]` bounds |
| `vergil` | A newer vergil-python *minor* exists in the package index than `[package.python] runtime` |
| `images` | A newer image series exists than a series-tagged ref (`python:3.14-slim` with 3.15 out), or a newer major than an exact-tagged ref |
| vergil-vm hook | Lima's latest major is above the `minimumLimaVersion` major; the newest Node LTS major is above the NodeSource major; a newer nerdctl major exists |
| vergil-python hook | A newer CPython minor exists |

## 8. Built-in updater changes

### 8.1 `python`

Unchanged upgrade behaviour (`uv lock --upgrade` in the dev container). It gains
held-back report entries: after the lock, it compares each **directly declared**
dependency with its latest release and reports any held back by the repo's own
bounds. "Directly declared" means `[project] dependencies`, every dependency
group, `[build-system] requires`, and `[tool.uv] constraint-dependencies`.
Transitive packages are never reported: their holds come from other packages'
bounds, which the repo cannot act on. (CVEs in transitive packages are already
caught by `pip-audit` in validation.)

### 8.2 `vergil`

- **Normalise all workflow and action files.** Extend the ref normaliser from
  `.github/workflows/*.yml|*.yaml` to every `action.yml`/`action.yaml` in the
  repo (for example vergil-actions' `actions/**`).
- **Bump the vergil-python runtime patch.** When `[package.python] runtime` is
  set, bump it to the newest vergil-python patch in the same minor that the
  **live vergil package repository serves in every index the release reads it
  from**. That means the shared deb build suite for both architectures, where
  `lib/package/python_builder.py` installs it for non-native cells, plus every
  target the repo's `[package]` selects: a native cell installs it from its own
  suite, and install-test resolves `Depends: vergil-python<runtime>` from each
  target's deb or rpm index. A version that is tagged or released on GitHub, or
  indexed for only some of those targets, is never a candidate; picking one
  would merge green and then fail the next release's package build or
  install-test. A newer minor is reported (§7.3), not bumped.

### 8.3 `images` (new)

Scans Dockerfile `FROM` lines, `container:` and `image:` values in workflow
files, and image defaults in `action.yml` inputs.

| Ref form | Action |
|---|---|
| Exact `x.y.z` tag (`trivy:0.71.0`) | Bump to the newest `x.y.z` tag **with the same major** (major 0 included, so `0.71.0 → 0.74.0`). Report a newer major; never bump across it |
| Numeric series tag: one or two numeric parts plus an optional suffix (`python:3.14-slim`, `ubuntu:24.04`, `postgres:16`) | Leave it; it floats. Report a newer series with the same suffix |
| `:latest`, a digest, an expression (`${{ … }}`, `$ARG`), or `ghcr.io/vergil-project/*` | Skip |
| Any other tag: codenames (`debian:trixie-slim`), or a series in the image name (`ubi9/ubi`) | Skip. These float and are never bumped; their series rollovers are not detected in this epic |

Registry lookups are injected so tests never touch the network. A lookup failure
fails loud. Today this updater changes one ref (vergil-actions' trivy default);
it is built generically because further repos will carry more.

### 8.4 Pin removals (not updaters)

- **`repo_init.py` templates** derive the vergil series from tooling's own
  version at runtime instead of hard-coding `@v2.1`.
- **`vrg-vm-init.sh`** reads the vergil series from `vergil.toml` instead of
  hard-coding `vergil-tooling@v2.1`.

## 9. Component boundaries

| Component | Owns | Does not own |
|---|---|---|
| `vrg-release` `update-deps` stage | Placement, deferral, tracking-issue comment, `--without-deps` | Updating anything itself |
| `vrg-release` preflight gate | Refusing on an open `dependency-update` PR; `--allow-pending-deps` | Fixing the PR |
| `vrg-update-deps` driver | Worktree, commits, single validation, labelled PR, failure push, fix-task filing, report collection | Any category's upgrade logic |
| Built-in updaters | One category each: upgrade at the source of truth, produce report entries | Git, validation, PRs |
| Hook runner | Running the repo hook, the HEAD check, `$VRG_DEPS_REPORT` | What the hook does |
| Repo hooks | Their repo's bespoke dependencies | Git history |
| `bump-tools` (vergil-containers) | Unchanged in this epic | – |

## 10. What this spec replaces in the 2026-06-10 design

| 2026-06-10 design | This spec |
|---|---|
| Docker and VM "extensions" enabled by config opt-in, with code in tooling | Repo-owned hooks; tooling carries no repo-specific logic (§6) |
| `[dependency-update]` override/pin schema (TBD) | No override schema; bounds live in native manifests (§3.2) |
| Warn-only entries declared in `[dependency-update]` | The held-back report, from updaters and hooks (§7) |
| A red validation aborts and leaves the worktree locally | The branch is pushed, the labelled PR stays open, and a fix task is filed (§5.2) |
| `vrg-release --with-deps` opt-in, then default-on | Default-on now, with `--without-deps` (§5.1) |
| Branch `chore/dep-update-<date>` (fails `vrg-commit`'s issue-branch check) | `deps/update-<new-develop-version>`; fixes on `chore/<N>-…` (§5.1, §5.2) |

The rest of that design (the stage pipeline, the updater interface, a single
validation run, no-op as first-class) stands. The documentation-review task
marks the old spec as partly superseded.

## 11. Out of scope and follow-on

- **Third-party GitHub Actions.** No tag bumping. They wait for SHA pinning:
  `.github#165` and `.github#194` (the `#155` bookends).
  `vergil-tooling#1596` points there.
- **Language and toolchain majors.** Reported only. Scheduled monitoring and
  alerting is follow-on brainstorm `.github#383`, which folds in
  `vergil-actions#140`.
- **Language updaters beyond Python** (Ruby, Go, Java, JS/TS, C++, Rust).
  Follow-on brainstorm `.github#384` restarts them as a new epic, taking over
  `vergil-tooling#1592`–`#1595` and `#1599`.
- **vergil-containers' `bump-tools`.** This epic takes ownership of it but leaves
  it running unchanged. Follow-on brainstorm `.github#385` reconciles scheduled
  and release-driven updates, and how update reporting reaches the human through
  the notification noise. `vergil-containers#599` (review and merge bump PR
  `#445`) follows it.
- **A scheduled trigger for `vrg-update-deps`.** Release-driven only in this
  epic; a schedule is part of `.github#385`.

## 12. Testing

- **Updaters:** unit tests per updater with network and registry lookups
  injected, following the pattern of vergil-containers' `resolve_latest.py`.
  This covers the change, no-change, held-back and lookup-failure paths.
- **Hook runner:** a hook that edits, one that does nothing, one that exits
  non-zero, one that commits (must fail on the HEAD check), and one that writes
  a malformed report line.
- **Config:** valid, empty, non-string and unknown-key `[dependency-update]`
  tables.
- **Release stage and gate:** orchestrator tests for:
  - success and no-op;
  - local validation failure and merge abort, each leaving the labelled PR open
    and unmerged, filing the fix task, and recording the failure;
  - failure output trimmed to the tail plus the reproduce command, staying under
    the body limit;
  - pre-validation failure (no PR);
  - `--without-deps`;
  - the gate refusing on an open labelled PR;
  - `--allow-pending-deps` logging its bypass.
- **Updater rules:** `images` never crosses a major; the vergil-python runtime is
  never bumped to a version missing from any index the build or install-test reads; `python` reports
  only direct dependencies.
- **Coverage:** vergil-tooling's 100% coverage gate applies.
- **End to end:** a validation task runs a live release cycle in vergil-tooling,
  vergil-vm and vergil-python. It checks the PR or no-op, the report comment, and
  the gate (tripped once deliberately, then bypassed with
  `--allow-pending-deps`).
