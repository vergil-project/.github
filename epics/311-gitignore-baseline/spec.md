# Centralized baseline `.gitignore` + self-policing ops audit

**Epic status:** proposed
**Epic:** `vergil-project/.github#311`
**Seed:** `vergil-project/.github#196` (idea)
**Follow-on (B):** `vergil-project/.github#313` (general required-workflows registry)

## Problem

`.gitignore` files across the Vergil fleet are **drifted, per-repo, and
hand-maintained**. Nothing guarantees the universal entries — `.venv/`,
`.worktrees/`, `.vergil/`, `.superpowers/`, and the full set of build/validation
output — are present everywhere. Two concrete failure modes recur:

1. **New ignorable artifacts require a fleet-wide hand-edit.** When a tooling
   change starts emitting a new artifact — most recently the CI-gate evidence
   files `quality-ruff.json` and `quality-mypy.xml` written at the workspace
   root — every repo must be edited by hand to ignore it. Stale feature branches
   that cannot be cleaned up strand the untracked files indefinitely (the
   originating observation on #196: a `logical-minds-foundry` worktree showing
   untracked `quality-mypy.xml` / `quality-ruff.json`).

2. **Un-ignored build output defeats automatic worktree cleanup.** A merged
   worktree dirtied only by regenerable build output (e.g. mkdocs `docs/site/site/`)
   makes `vrg-finalize-pr` correctly refuse to delete it — it cannot distinguish
   regenerable output from real uncommitted work. The worktree is stranded and
   contributes to disk pressure (recorded on #196, tied to disk-exhaustion
   incidents).

The baseline that *should* be universal already exists — but only as a hardcoded
string in `repo_init.py::render_gitignore()`, applied once at repo creation and
never re-checked. It is not a shared asset and it drifts.

## Goals

- **One source of truth** for the baseline `.gitignore`, owned by
  `vergil-tooling` and consumed by both repo scaffolding and the audit.
- **A single integrated baseline** — the union of the languages Vergil manages
  plus universal categories — applied to every repo regardless of language. No
  per-language branching.
- **Automatic drift detection** that is **fatal** where it fires, so a
  non-conforming repo cannot silently persist. Drift surfaces through the
  existing nightly `ops.yml` config audit.
- **A repo that runs the audit validates its own `ops.yml` wiring**: where the
  nightly audit runs, it asserts the repo's `ops.yml` is present and correctly
  wired to the config-audit workflow (catching a broken/edited `ops.yml`). Note
  the structural limit: this local check cannot detect a repo that *lacks*
  `ops.yml` entirely, because a repo with no `ops.yml` has no nightly run — see
  the Non-goals and follow-on C.
- **Every new repo is born conforming** — `repo_init` scaffolds both the baseline
  `.gitignore` and `ops.yml`.
- **The whole fleet is reconciled** to the baseline and made self-policing,
  across all four Vergil orgs.

## Non-goals

- **No per-language `.gitignore` logic.** We do not ignore Python things only in
  Python repos, TypeScript things only in TS repos, etc. One integrated file
  goes everywhere. (Deliberate simplicity, per brainstorming.)
- **No exact-match enforcement of the whole file.** A repo `.gitignore` must be
  a **superset** of the baseline; local additions are explicitly allowed. We do
  not assert the repo file equals the baseline. (Individual baseline *lines* are
  matched verbatim — see §2 — but the repo may carry any number of extra lines.)
- **No pattern-equivalence normalization.** A baseline line must appear verbatim;
  we deliberately do **not** treat `.venv/` and `.venv` (or leading-slash
  variants) as equivalent. The baseline defines the one canonical spelling per
  pattern and the sweep standardizes every repo to it (§2). Supporting variant
  spellings was considered and rejected as over-engineering — standardization is
  cheaper than a normalizer and keeps "fatal" exact-by-design.
- **No `vrg-validate` involvement.** The drift check does **not** run in
  `vrg-validate`. Its home is the nightly config audit only, to avoid per-commit
  overhead. (The check is cheap and local, so this is a deliberate
  latency-vs-noise choice, not a technical limit.)
- **No fleet-level `ops.yml` presence check in this epic.** The local
  `_check_required_workflows` validates the *wiring* of an `ops.yml` that exists;
  it cannot catch a repo missing `ops.yml` entirely (that repo runs no audit). A
  from-outside check that enumerates all managed repos and asserts each carries
  `ops.yml` requires a **general fleet-wide audit tool** — its own complex
  design — deferred to **follow-on C** (#315). This epic relies on
  `repo_init` scaffolding + the one-time rollout to establish `ops.yml`
  everywhere.
- **No general required-workflows registry in this epic.** We enforce only
  `ops.yml` presence-and-wiring locally. The general "assert all
  minimally-required workflows per repo class" framework is deferred to
  **follow-on B** (#313).
- **No new sync/reconcile engine.** Propagation rides the fleet's rolling
  `vergil-tooling@vX.Y` pin (§5); there is no push-based template renderer.
- **No cross-org linking.** Each other org's rollout is its own epic in its own
  `.github`; none is a sub-issue of #311.

## Design

### 1. The baseline as a data asset

Add `src/vergil_tooling/data/gitignore.baseline` (a real `.gitignore`-syntax
file) to the packaged `vergil_tooling.data` package. It is the single source of
truth. It is loaded at runtime via
`importlib.resources.files("vergil_tooling.data").joinpath("gitignore.baseline").read_text(...)`
— the same idiom already used for `claude_settings.json` and
`claude_md_consumer.md`.

**Content = "the integral."** The baseline is the union of:

- **Universal / editors / OS:** `.DS_Store`, `Thumbs.db`, `.idea/`, `.vscode/`,
  `*.swp`, `*.swo`, `*~`, `.env`, `.env.*`, `*.log`.
- **Vergil internals:** `.venv/`, `.worktrees/`, `.vergil/`, `.superpowers/`,
  `.claude/scheduled_tasks.lock`.
- **Build / validation output (all of it):** `dist/`, `build/`, `*.egg-info/`,
  `__pycache__/`, `*.pyc`, `.coverage`, `coverage.xml`, `junit.xml`,
  `pip-audit.json`, `licenses.json`, `quality-ruff.json`, `quality-mypy.xml`,
  `.pytest_cache/`, `.mypy_cache/`, `.ruff_cache/`, and the mkdocs build output
  `site/` (and/or `docs/site/site/` — see open question O1).
- **Managed-language artifacts:** the union across the languages in
  `lib/languages.py` (Python, TypeScript/Node, Go, Ruby, C++). E.g.
  `node_modules/`, `*.tsbuildinfo`, Go test/coverage output, Ruby
  `vendor/bundle/`, C++ object/build artifacts.

The concrete line set is **built during implementation** by integrating the
existing `.gitignore` files across the fleet (so nothing currently ignored is
lost) and de-duplicating. The list above is the required floor, not the ceiling.

**`repo_init` reads the asset.** `render_gitignore()` stops returning a
hardcoded string and instead reads `gitignore.baseline`. Result: scaffolding and
audit share one definition and cannot diverge.

### 2. Superset matching semantics

The check treats the baseline as a **set of pattern lines**:

- Comment lines (`#…`) and blank lines in the baseline are **ignored** for
  matching (they are documentation, not requirements).
- Each remaining baseline line, after trailing-whitespace trimming, must appear
  **verbatim as a line** somewhere in the repo's `.gitignore`. Matching is
  **order-independent**.
- The repo `.gitignore` may contain **any number of additional lines** (local
  additions) — that is expected and allowed.
- A missing baseline pattern is reported as a `DiffItem(expected=<pattern>,
  actual="missing")`, mirroring `_check_settings_section`. A repo with no
  `.gitignore` at all fails with every baseline pattern reported missing.

This is the simplest rule that catches the real failure (a missing `.venv/` or
`quality-ruff.json` line) without getting into negation ordering or pattern
semantics. (See open question O2 on comment-block preservation.)

**Canonical spelling + standardize-on-sweep.** Matching is verbatim, so the
baseline defines the **one canonical spelling per pattern** (e.g. `.venv/`, not
`.venv` or `/.venv/`). We deliberately do not normalize equivalent spellings in
the check — instead, the reconciliation sweep **rewrites** each repo's
`.gitignore` to the canonical baseline lines (replacing variant spellings, not
appending duplicates). After the sweep every repo carries the exact baseline
lines, so "fatal" means an actual missing rule, and any future variant spelling
is a (correct) failure that pushes the repo back to the standard. The fleet
integration step (§1) is where variant spellings are found and folded into the
canonical set.

### 3. Two new local audit checks

Both live in `lib/repo_config.py::audit_local_config()`, which is **pure local
file I/O — no network**. They therefore run inside the existing nightly
`ops.yml` config-audit job (which already exercises `audit_local_config`) and
are **fatal** there: `vrg-github-repo-config audit` returns exit 1 on any
`DiffItem`, so drift turns the scheduled run red.

**`_check_gitignore(repo_root)`** — implements §2. Reads the bundled baseline
and the repo's `.gitignore`; emits a `DiffItem` per missing baseline pattern.
Reuses the `_check_settings_section` superset-comparator pattern.

**`_check_required_workflows(repo_root)`** — a **wiring validator**: asserts
`.github/workflows/ops.yml` exists and references the reusable
`ops-github-config.yml` config-audit workflow (so a repo cannot carry an
`ops.yml` that omits the audit). Emits a `DiffItem` when the workflow is absent
or does not wire the audit.

**Known structural limit (deliberate, deferred to follow-on C).** Because this
check runs *inside* the nightly `ops.yml` job, it cannot detect a repo that
lacks `ops.yml` **entirely** — such a repo has no nightly run, so nothing
executes the check there. The local check therefore catches a *broken* or
*mis-wired* `ops.yml`, not a *missing* one. Guaranteeing `ops.yml` presence
across the fleet requires a from-outside check that enumerates all managed repos
— a **general fleet-wide audit tool**, which is a substantial design in its own
right and is deferred to **follow-on C** (#315). In this epic,
presence is established by `repo_init` scaffolding (new repos) and the one-time
rollout (existing repos); the local wiring validator keeps a present `ops.yml`
honest thereafter.

Both are registered in `audit_local_config`'s helper list alongside the existing
`_check_vergil_toml`, `_check_hook_guard_shim`, `_check_claude_md`,
`_check_claude_settings`, `_check_workflow_refs`.

### 4. `repo_init` scaffolds `ops.yml`

`repo_init` today renders `ci.yml`, `cd.yml`, and the epic-rollup workflow but
**not** `ops.yml`. Add `render_ops_workflow()` (emitting the daily-cron
`ops.yml` that calls `ops-github-config.yml@<pinned-ref>`) and write it during
init. New repos are then born self-policing and pass `_check_required_workflows`
from day one.

### 5. Propagation model

By fleet convention, every repo pins `vergil-tooling` to the **rolling
major-minor tag** `vX.Y` — never to a specific patch release. The nightly
`ops.yml` job installs `vergil-tooling@vX.Y`, so it always resolves to the
latest patch under that line, including its `gitignore.baseline`. Therefore:

> change the baseline → release a `vergil-tooling` patch under `vX.Y` → **every
> repo picks it up on its next nightly run, automatically** — no pin bump, no
> per-repo edit, nothing to touch.

This is the whole point of the rolling-tag convention (forward/backward movement
across upgrades without touching consumers), and it is what makes "we change the
central baseline and every repo starts flagging it's out of date" true on a
one-day horizon rather than whenever a pin is next hand-advanced. No push-based
renderer, no new sync engine — the rolling pin the fleet already uses *is* the
propagation mechanism.

## Enforcement & fleet rollout

### vergil-project (in this epic)

`ops.yml` exists in only **`vergil-tooling`** and **`vergil-containers`** today.
The remaining vergil-project repos have **no config-audit job at all**, so the
self-catch mechanism cannot work there until `ops.yml` lands. This epic files
operational (deployment) tasks to roll `ops.yml` **and** reconcile `.gitignore`
to the baseline for:

- `vergil-actions`
- `vergil-claude-plugin`
- `vergil-vm`
- `vergil-project/.github` (the org repo itself)

Plus reconcile `vergil-tooling` and `vergil-containers` `.gitignore` to the
final baseline (they already carry `ops.yml`).

**Per-repo audit-readiness is a precondition of each rollout task.** Turning on
the nightly audit runs the *full* `audit_local_config` (which also checks
`vergil.toml`, `guard.sh`, `CLAUDE.md`, `.claude/settings.json`, workflow refs)
plus the GitHub-API half — so a repo that was never fully onboarded will go red
on *those* checks, not on `.gitignore`, the moment `ops.yml` lands. Therefore
the **plan enumerates each target repo with its current audit-readiness** and
each rollout task's precondition self-check verifies the repo already passes
`vrg-github-repo-config audit` locally (everything but the new checks) before
adding `ops.yml`. Where a repo is not meant to be a managed member (open
question O4: is `vergil-project/.github` itself a managed repo with
`vergil.toml`, or should it be scoped out of the audit rollout?), the plan
either onboards it first or explicitly scopes it out — never silently rolls
`ops.yml` onto a repo that will red for pre-existing reasons.

### Other orgs (per-org follow-on epics)

The whole-fleet sweep is the goal, but cross-org linking is banned, so each
other org gets its **own rollout epic** homed in that org's `.github`, tracked
from #311's retrospective §5 (not linked). Each is a collection of operational
(deployment) tasks: roll `ops.yml` + reconcile `.gitignore` per repo.

| Org | Disposition |
|---|---|
| `vergil-project` | Done in this epic. |
| `logical-minds-foundry` | Follow-on rollout epic, worked aggressively. |
| `mnemosys-project` | Follow-on rollout epic, worked aggressively. |
| `mq-rest-admin-project` | Follow-on rollout epic created + planned, then parked. |

Standing up these three epics is part of this epic's closing work (before the
retrospective).

## Rollout sequencing (avoid a fleet-wide red flag-day)

Because the check is fatal in the nightly audit, enabling it before repos are
clean turns every drifted repo's nightly job red at once. To keep the red-X
signal meaningful:

1. **Land the tooling first** — baseline asset, both checks, `repo_init`
   scaffolding, tests — and release the `vergil-tooling` patch under `vX.Y`.
   Because every repo tracks the rolling `vX.Y` tag (§5), the new checks go live
   fleet-wide on the **next nightly run** with no per-repo pin bump — so the
   reconciliation must be *ready to move* when the release lands.
2. **Reconcile each repo to clean.** A repo goes green once its `.gitignore` is
   the canonical superset and its `ops.yml` is present-and-wired. There is no pin
   to advance — the rolling tag already carries the new baseline — so "reconcile
   the repo" is the whole job. Sequence the sweep so repos are reconciled
   promptly after the release rather than sitting red.
3. **Backstop:** a transiently-red nightly run is acceptable and self-resolving
   as the sweep proceeds; it is the acceptance signal that a repo is not yet
   clean. Feeds the GitHub failure-email operational discipline being adopted.

## Data model / touched surfaces (vergil-tooling)

- **New:** `src/vergil_tooling/data/gitignore.baseline` (packaged data).
- **Modified:** `src/vergil_tooling/lib/repo_config.py` — add `_check_gitignore`,
  `_check_required_workflows`; register both in `audit_local_config`.
- **Modified:** `src/vergil_tooling/lib/repo_init.py` — `render_gitignore()`
  reads the asset; add `render_ops_workflow()` + write it.
- **Packaging:** ensure `gitignore.baseline` ships as package data (pyproject /
  MANIFEST as applicable).
- **Docs:** config-audit reference, baseline reference, `repo_init` docs, and
  the consuming-repo contract (documentation-review sweep, #2833).

## Testing

- Unit tests for `_check_gitignore`: exact-baseline (pass), superset with local
  additions (pass), missing pattern (fail with correct `DiffItem`), absent file
  (fail), comment/blank-line handling, trailing-whitespace normalization.
- Unit tests for `_check_required_workflows`: present-and-wired (pass), absent
  (fail), present-but-not-wired-to-audit (fail).
- `repo_init` tests: a freshly scaffolded repo passes both new checks
  (round-trip: init → `audit_local_config` clean).
- Coverage: the repo's `--cov-fail-under=100` gate applies; note the
  multi-version PR-CI coverage matrix (3.12/3.13/3.14).

## Open questions (resolve during spec review / implementation)

- **O1 — mkdocs output path.** Ignore `site/` generically, or the specific
  `docs/site/site/` path some repos emit, or both? (Leaning: include the generic
  `site/` and any concrete paths found in the fleet integration.)
- **O2 — baseline comment blocks.** The baseline file will carry explanatory
  comments; §2 ignores comments for matching. Confirm we do **not** also require
  repos to carry the baseline's comments (superset of *patterns*, not of
  comment text).
- **O3 — `ops.yml` cron uniformity.** Adopt the existing `15 6 * * *` schedule
  fleet-wide, or stagger per repo to avoid a thundering herd? (Leaning: keep the
  established `15 6 * * *`.)
- **O4 — is `vergil-project/.github` a managed repo?** Does it carry
  `vergil.toml` and pass `audit_local_config` today, or should it be scoped out
  of the audit rollout? Decides whether its rollout task is "add two lines" or
  "onboard the repo." (Established in the plan's per-repo readiness pass.)

## Follow-on brainstorm bookends — deferred

Both are seeded as brainstorm tasks under this epic and run via
`/vergil:epic-create` to produce successor epics; their outcomes are recorded in
#311's retrospective §5. They are closely related (a fleet-wide auditor is
plausibly the *engine*; the required-workflows registry is the *policy* it
enforces) — see the note in the plan on whether they merge into one brainstorm.

- **Follow-on B (#313) — general required-workflows registry.** Assert the full
  minimally-required workflow set per repo class (likely coupled to repo
  class/language). This epic enforces only `ops.yml` presence-and-wiring.
- **Follow-on C (#315) — general fleet-wide audit tool / `ops.yml` presence check.** A
  from-outside auditor that enumerates all managed repos and asserts each carries
  a correctly-wired `ops.yml` (and, generally, other invariants a repo cannot
  self-check because a missing workflow means no run fires). Closes the
  structural gap in `_check_required_workflows` (§3).
