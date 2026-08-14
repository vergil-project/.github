# Retrospective — Centralized baseline `.gitignore` + self-policing ops audit

**Epic:** `vergil-project/.github#311`
**Span:** opened 2026-08-13, closed 2026-08-14 (~1 day)
**Partners:** [`spec.md`](./spec.md) · [`plan.md`](./plan.md) · this retrospective (read in that order)

## §0 — At a glance

We set out to stop `.gitignore` files drifting across the fleet by making a single
centrally-owned baseline the source of truth and making every repo self-police
its compliance. What shipped: the baseline is now a packaged data asset consumed
by both `repo_init` scaffolding and a new nightly config-audit check; two
pure-local audit checks (`_check_gitignore` superset + `_check_required_workflows`
ops.yml presence/wiring/schedule) are fatal in the nightly `ops.yml` run;
`repo_init` scaffolds a staggered-cron `ops.yml`; and the whole vergil-project
fleet was reconciled and made self-policing in one day.

**Work delivered (9 merged PRs across 7 repos + this retrospective):**

| PR | Repo | Merged | What it did |
|---|---|---|---|
| #316 | .github | 08-13 | Publish `spec.md` + `plan.md` (documentation task #312) |
| #2837 | vergil-tooling | 08-13 | **The tooling**: baseline asset, `_check_gitignore`, `_check_required_workflows`, `render_ops_workflow` (staggered cron), `repo_init` scaffolds `ops.yml` |
| #850 | vergil-actions | 08-13 | Reconcile `.gitignore` + add `ops.yml` |
| #685 | vergil-claude-plugin | 08-13 | Reconcile `.gitignore` + add `ops.yml` |
| #305 | vergil-vm | 08-13 | Reconcile (kept Lima/OpenTofu extras; `/build/`→`build/`) + add `ops.yml` |
| #318 | .github | 08-13 | Reconcile `.gitignore` + add `ops.yml` |
| #26 | docs | 08-13 | Reconcile `.gitignore` + add `ops.yml` |
| #554 | vergil-containers | 08-13 | Reconcile + **surgical** cron restagger (kept custom nightly-rebuild jobs) |
| #2841 | vergil-tooling | 08-14 | Documentation-review sweep (config-audit reference, consuming-repo contract, baseline reference) |

- **Repos touched:** 7 — vergil-tooling, .github, vergil-actions, vergil-claude-plugin, vergil-vm, docs, vergil-containers.
- **Tasks:** 10 children — 1 impl, 6 rollout, 1 documentation, 1 documentation-review, 1 retrospective (this).
- **Release cut:** `vergil-tooling v2.1.196` (2026-08-13 17:54Z), carrying impl PR #2837 (merged 17:37Z); this is the installed version that put the baseline on the rolling `v2.1` tag every repo tracks.
- **Staggered ops crons assigned:** 19/29/30/36/37/45 past 06:00 UTC (no collisions).
- **Follow-ons parked:** brainstorms B (#313) and C (#315) re-parented to ad-hoc epic #99 as `idea`s; three cross-org rollout epics created (`logical-minds-foundry#204`, `mnemosys-project#65`, `mq-rest-admin-project#39`).

## §1 — How the plan evolved

The plan survived execution largely intact; the deltas were mechanical
adaptations and two judgment calls, not design reversals.

- **Baseline grew by two lines during implementation.** The plan authorized a
  fleet-integration pass (Task 1 Step 3); it surfaced `*.bak` and
  `.claude/settings.local.json` as genuinely universal, which were folded in with
  canonical spelling. Everything else in the fleet's `.gitignore`s was confirmed
  repo-specific. Go artifacts (`*.test`, `*.out`) had already been added during
  alignment after the plan under-listed them versus the spec.
- **A pre-existing test fixture had to be repaired.** Registering the two new
  checks in `audit_local_config` broke `_write_compliant_repo`, which scaffolded
  neither a `.gitignore` nor an `ops.yml`. Fixing it also required bumping that
  fixture's `vergil.toml` pin to `v2.1` so the `ops.yml @v2.1` ref matched the
  workflow-ref checker — an interaction the plan didn't anticipate.
- **The containers restagger deviated deliberately.** The plan said "overwrite
  `ops.yml` with the rendered template." Followed literally, that would have
  deleted `vergil-containers`' repo-specific `nightly-rebuild-dev/prod` jobs. The
  rollout agent recognized `_check_required_workflows` is a wiring validator (not
  an exact-match), and restaggered *surgically* — changing only the cron minute.
  The "preserve repo-specific extras" principle the plan applied to `.gitignore`
  turned out to apply to workflows too.
- **A labeling correction.** The plan described the per-repo rollout as
  `deployment`-kind operational tasks; they are actually code PRs, so they were
  filed as regular PR-workable `task`s. The genuine human-gated *deployment* was
  the `v2.1.196` release. (Noted on #2833 for the doc sweep.)

## §2 — Lessons learned

- **Agents cannot push `.github/workflows/**`.** Every rollout branch adds or
  edits `ops.yml`, and the GitHub App (agent identity) lacks the `workflows`
  permission, so `vrg-git push` was refused on all six. The `vrg-git` wrapper
  correctly flags this as expected and directs proceeding to `report-ready`; the
  human's `vrg-submit-pr` pushes the branch at submit time. **Takeaway:** any
  fleet rollout that ships workflow files must plan for human-push-at-submit —
  the branches will not be on origin at report-ready, so the relay/worktree-free
  submit path does not apply; submit from each worktree.
- **Verbatim-superset + standardize-on-sweep was the right simplicity.** Only one
  repo (`vergil-vm`, `/build/`) carried a variant spelling; the fleet was
  overwhelmingly already canonical, vindicating the decision to reject a
  normalizer in favor of standardizing on the sweep.
- **"Preserve repo-specific extras" generalizes.** It was written for
  `.gitignore`; the containers restagger proved it applies to workflow files.
  Future "regenerate a managed file" steps should be surgical by default when the
  target can carry repo-specific content.
- **Front-loaded pushback/alignment paid off.** The one structural gap
  (`_check_required_workflows` cannot see a *missing* `ops.yml`) and the schedule
  assertion were both caught before code existed, not after.

## §3 — Compromises & tradeoffs

- **Known hole, accepted knowingly:** the local `_check_required_workflows` is a
  wiring validator — it cannot detect a repo that lacks `ops.yml` entirely,
  because no workflow means no nightly run to fire the check. This epic relies on
  `repo_init` scaffolding + the one-time rollout to establish presence, and defers
  the from-outside fleet auditor to follow-on **C (#315)**. This was a deliberate
  scope cut to keep the epic to one integrated baseline.
- **Single `ops.yml` enforcement only.** We enforce presence of one workflow, not
  the full required-workflow set per repo class — deferred to follow-on **B
  (#313)**.
- **Docs fully deferred to the bookend.** The implementation PR shipped code +
  tests only; all human-facing docs landed in the documentation-review sweep
  (#2841). Correct per the epic architecture, but it means the reference docs
  trailed the code by a day.
- **Unrelated staleness left in place.** The doc sweep found
  `consuming-repo-setup.md` Step 7 shows an outdated CI snippet; out of scope,
  logged for a separate follow-up (see §4).

## §4 — New problems & opportunities

- **`mq-rest-admin-project/.github` is missing canonical labels.** Seeding its
  retrospective bookend failed until the `retrospective` label was created by
  hand — that org's `.github` labels aren't synced. *Where it went:* surfaced
  here; its rollout epic (#39) or a label-sync pass should address it. Logged, not
  yet acted on.
- **`consuming-repo-setup.md` Step 7 is stale** (old `standards-compliance`
  composite-action snippet vs. the reusable-workflow `ci.yml`/`ops.yml`
  `repo_init` now generates). *Where it went:* logged, not yet acted on — a
  candidate follow-up doc issue.
- **The workflow-push friction** (agents can't push workflow files) is a recurring
  cost for any future workflow-shipping rollout. *Opportunity:* a documented
  rollout pattern (or tooling affordance) for workflow-file changes.

## §5 — What's next

- **Follow-on B — `#313` (required-workflows-by-repo-class registry):** re-parented
  to ad-hoc epic **#99** as an `idea`; strategic, deferred until there's time.
- **Follow-on C — `#315` (fleet-wide audit tool / from-outside `ops.yml` presence
  check):** re-parented to ad-hoc epic **#99** as an `idea`. Closes §3's known
  hole. B and C are related — C is plausibly the *engine*, B the *policy*; whoever
  picks them up should consider sequencing C first.
- **Cross-org fleet sweep** — the whole-fleet goal continues via independent
  per-org rollout epics (cross-org linking is banned, so these are not children of
  #311):
  - `logical-minds-foundry#204` — worked aggressively (per-repo tasks seeded).
  - `mnemosys-project#65` — worked aggressively (per-repo task seeded).
  - `mq-rest-admin-project#39` — created + parked.

  Drive each with its own `/vergil:epic-implement <ref>`.
