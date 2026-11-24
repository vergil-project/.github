# Agents submit and finalize their own PRs into develop — implementation plan

- **Epic:** vergil-project/.github#1384
- **Planning issue:** vergil-project/.github#1383 (review history is in its
  comments)
- **Status:** approved
- **Spec:** [`spec.md`](spec.md)

> **For agentic workers:** each task below is one issue and one PR, driven
> by `epic-implement` / `issue-implement`; operational tasks run with
> `issue-deploy` or `issue-validate`. Steps use checkbox (`- [ ]`) syntax.

**Goal:** the user agent takes a task from implementation to a merge into
`develop` through `vrg-submit-pr` and `vrg-finalize-pr`, with the human
involved only for rulings, protected-change approvals, merge-review
exemptions, failures and releases.

**Architecture:** four new vergil-tooling modules carry the mechanism. They
are the protected-path set (`lib/protected_paths.py`), the change
fingerprint (`lib/change_fingerprint.py`), two new authenticated request
kinds with a past-exchange verifier (`lib/rulings/approvals.py`), and the
merge-review record (`lib/merge_review/record.py`). One shared evaluator
(`lib/agent_gates.py`) applies them, and submit, finalize's watch and a new
daily audit (`vrg-agent-audit`) all call it. Server-side, a `main` ruleset
and environment-scoped secrets come first. The plugin's skills and hooks and
the steering then move to the new boundary, and the rollout grants the App
permissions last, fixtures first.

**Tech stack:** Python 3.15 (vergil-tooling, pytest), bash (plugin hooks
with their `*.test.sh` suites), Markdown skills (vergil-claude-plugin),
GitHub Actions YAML (vergil-actions, `.github`), GitHub rulesets and
environments, the GitHub App settings.

## Global constraints

Copied from the spec; every task's requirements include them.

- Releases and promotion stay human-only: `vrg-release`, `vrg-promote`, and
  `--release` on `vrg-submit-pr` and `vrg-finalize-pr` refuse every agent
  identity.
- `vrg-submit-pr` opens PRs against `develop` only.
- The user agent may create, turn auto-merge on for, and merge **its own**
  PR (author is its login). On any other PR every action forbidden today
  stays forbidden. It never approves any PR.
- There is no protected core and no protected-change approval (amended
  2026-11-24, see Evolution during execution). Every path gets merge review
  like any other.
- A merge-review exemption is a `vrg-ruling`
  exchange: the request by the agent login, the reply by the human login,
  neither edited or deleted, and the reply the first trusted comment after
  the request. The reply carries exactly one decision line, `approve` or
  `refuse`. Anything else is not approved.
- The change fingerprint is the SHA-256 of the normalised zero-context diff
  defined in Task 5 (whitespace kept; `@@` and `index` lines dropped;
  binaries by head blob), never `git patch-id`. For one commit of a batch,
  the commit's diff against its parent is used.
- Every check fails closed: an unreadable exemption, record or API
  response refuses. Nothing passes silently.
- Every comment read goes through the trust layer of epic
  vergil-project/.github#1467 (`lib/trust.py`): the readers are built on
  `trust.read_comment_stream`, through `rulings.comments.read_stream`, with
  the roles they accept. "Trusted" is the trust layer's judgement. The human
  and agent logins come from `load_trusted_authors()` (`TrustedAuthors`),
  never from a new variable or flag. The trust guard test
  (`tests/vergil_tooling/test_trust_guard.py`) stays green.
- The orgs in scope are vergil-project, logical-minds-foundry and
  mnemosys-project (spec §4.11). Archived repos are left alone.
  diogenes-project, mq-rest-admin-project and vergils-nemesis are out of
  scope, and no installation there accepts the user App's new permissions.
- Every code task is red, green, refactor: write the failing test and see
  it fail, make it pass, then refactor with the tests green, before the
  commit. A task's "Run → PASS" step includes the refactor.
- `workflows: write` is granted only after the environment-scoped secrets
  are in place and verified. `pull_requests: write` is granted only after
  the `main` ruleset is converged and the daily audit runs green fleet-wide
  with its marker set.

## Review focus

Inputs the spec implies that no single happy-path test covers. Each line's
test is added to the task named.

1. **An exemption reply that is edited after posting, posted by the wrong
   login, or followed by a second human reply** → the edited or wrong-login
   reply is not approved; the second reply is ignored (Task 6b:
   `test_past_approval_rejects_edited_and_wrong_login`).
2. **A rebase over a `develop` edit next to the branch's hunks** → the
   fingerprint is unchanged; a conflict resolution that changes the branch's
   own lines changes it (Task 5:
   `test_fingerprint_survives_nearby_develop_edit`).
3. **A GitHub read failure during the watch's re-check** → auto-merge is
   turned off and the PR is reported failed with the read error, never left
   armed (Task 11: `test_recheck_read_failure_disarms`).
4. **A PR whose task issue cannot be resolved from its body or ready-state**
   → the gates refuse, naming the PR, and never pass for lack of an issue
   (Task 8: `test_unresolvable_task_issue_refuses`).

## Delivery

Tasks are numbered in rollout order (spec §5). "Blocked by" is merge order
for PR tasks and run order for operational ones.

Repos: `T` vergil-tooling, `P` vergil-claude-plugin, `A` vergil-actions, `G`
.github, `F` the six `vergil-fixture-*` repos. Kinds: `task`, `deploy`,
`valid` (validation), `fixture`, `docs`, `retro`.

| # | Task | Repo | Kind | Blocked by |
| --- | --- | --- | --- | --- |
| 1 | Spec and plan (#1385) | G | docs | — |
| 1b | Rebase onto the recent epics (#1551) | G | docs | 1 |
| 1c | Audit markers created by repo init (#1556) | G | docs | 1b |
| 2 | `main` ruleset restricting updates | T | task | — |
| 2d | Release tooling; apply the `main` ruleset | T | deploy | 2 |
| 3a | Reusable workflows read App secrets from an environment | A | task | — |
| 3r | Release vergil-actions with 3a | A | deploy | 3a |
| 3b | Repo config converges the secrets environment | T | task | — |
| 3c | Callers read the App keys from the environment | each | task | 3r |
| 3e | `repo_init` and the `secrets: inherit` check | T | task | 3a |
| 3f | Release tooling; apply the environment | T | deploy | 3b, 3e |
| 3d | Move the secrets | — | deploy | 3f, 3r, 3c |
| 4 | Protected-path set and match (removed, #5117) | T | task | — |
| 5 | Change fingerprint | T | task | — |
| 6a | Exemption request kind and the structured reply | T | task | 4, 5 |
| 6b | Past-exchange verifier and lookup | T | task | 6a |
| 7 | Merge-review record | T | task | 5 |
| 7d | Release tooling: record commands | T | deploy | 5, 6a, 6b, 7 |
| 14 | Plugin hooks' wording | P | task | — |
| 15a | `merge-review`: record, approvals, exemption | P | task | 7, 6a |
| 15e | Docs, retrospective and logbook PRs run merge review | P | task | 15a |
| 15f | `hoshin` reads the agent audit as a security signal | P | task | — |
| 15r | Plugin release A: records | P | deploy | 7d, 14, 15a, 15e, 15f |
| 8 | Shared gate evaluator | T | task | 4, 6b, 7 |
| 9 | Submit: agents allowed, gates before push | T | task | 8, 15r |
| 10 | Provenance: own-PR rules | T | task | — |
| 11 | Watch re-check; finalize `--release` refusal | T | task | 8, 15r |
| 12 | Daily audit: `vrg-agent-audit` | T | task | 8, 15r |
| 12b | Repo init creates the audit markers | T | task | 12 |
| 13 | The audit job (A) | A | task | 4, 12 |
| 13r | Release vergil-actions with 13 (A) | A | deploy | 13 (A) |
| 13 | The audit job's callers (G and others) | G, each | task | 13r |
| 17d | Release tooling: the gates | T | deploy | 15r, 8–12, 12b, vergil-project/vergil-tooling#5117 |
| 12d | Audit live fleet-wide, marker set, green | G | deploy | 17d, 13r, 13 |
| 18d | Grant the user App `pull_requests: write` | — | deploy | 2d, 12d |
| 15b | The implementer owns the PR to merge | P | task | 15a, 18d |
| 15c | Epic and fixture skills submit | P | task | 15b, 15e |
| 15d | `hoshin` on the protected-change approval (dropped) | P | task | 15e, 18d |
| 16a | Consumer CLAUDE.md and permission spec | T | task | 9, 18d |
| 16b | `.github` CLAUDE.md on the new boundary | G | task | 18d |
| 21d | Plugin release B and the steering | T, P | deploy | 15b, 15c, 16a, vergil-project/vergil-claude-plugin#1326 |
| 13 | Logbook registry (docs) (dropped) | docs | task | 21d |
| 3g | Grant `workflows: write` | — | deploy | 3d, 17d |
| 19 | Fixture runs (first live agent cycle) | F | fixture | 21d, 3g |
| 19v | Exemption and audit on one fixture | F | valid | 19 |
| R1 | Documentation review (#4751) | T | docs | 19v, 16b |
| R2 | Retrospective (#1386) | G | retro | R1 |

Once 1b has merged, tasks 2, 3a, 3b, 4, 5, 10, 14 and 15f have no
blockers and can be worked side by side. Task 6a follows once 4 and 5
have merged, since its request commands compute their inputs with their
interfaces. Tasks 2, 2d, 3a–3f and 3r are
the standalone hardening the spec puts
first (§5 step 0). They land even if the rest stalls.

**At filing:**

- vergil-project/vergil-actions#1185 is absorbed. Close it as planned under
  this epic, and hand its advisory to Task 3d.
- The twenty-eight advisories that track the planning issue are handed on as
  follows:
  - GHSA-4j45-whfp-5cx7: Tasks 3a, 3b and 3d
  - GHSA-cm96-w5w4-j3c5: Task 3a
  - GHSA-hf27-hchj-j89r: Task 11
  - GHSA-f4g9-945g-7cm3: Task 6a
  - GHSA-c53g-m85p-prvh: Tasks 2 and 12
  - GHSA-q437-22r7-q48p: Tasks 7 and 8
  - GHSA-x2r2-5gr2-5qpm: Task 8
  - GHSA-gqq6-qwrw-2rvj and GHSA-v4wq-c77g-9454: Task 12
  - GHSA-26cx-rf37-v9m4: Tasks 12d and 18d
  - GHSA-4ch5-4j8f-46mv: Tasks 4 and 13
  - GHSA-3qpq-6264-6289: Tasks 13 and 15d
  - GHSA-qgfm-wvg6-jjvx: Tasks 13 (docs) and 15d
  - GHSA-gmhj-jph8-6mc6: Task 5
  - GHSA-xxfj-536h-45m4: Task 8
  - GHSA-g687-cf78-gq8r and GHSA-58c7-jh4x-wmwc: Task 9
  - GHSA-x2rv-m922-fpf6 and GHSA-3mww-c9j5-vh5m: Task 12
  - GHSA-x8m3-f3qj-69gm: Task 2
  - GHSA-4xx7-28w3-7w84: Tasks 13, 12d and 18d
  - GHSA-x7r5-wx2m-34gm: Tasks 2 and 12
  - GHSA-5pr5-4fr5-rmwv: Task 4
  - GHSA-jv6m-5f5m-gq22: Tasks 3a and 3d
  - GHSA-468p-pw79-g966: Tasks 3c and 3e
  - GHSA-54j2-vrxg-m5vg: Task 3b
  - GHSA-r8hj-wm79-h4wf: Task 3a
  - GHSA-5f99-wf94-2gxh: the documentation task (#1385); the finding is
    resolved in the design alone, by the spec's §4.8 wording

**Other orgs.** Tasks are never linked across orgs. The changes this plan
makes in logical-minds-foundry and mnemosys-project repos are listed per
org in Task 3a's inventory: the Task 3c callers, the Task 13 `ops.yml`
callers and the Task 12d marker setup. Each is made by the fleet sync where
it manages the file. Otherwise it is filed under that org's proxy of this
epic, created with `vrg-epic-create --proxy-for vergil-project/.github#1384
--repo <org>/<repo>` (one proxy per org), and named by its full ref
(`<owner>/<repo>#<N>`) in this table and in every `Blocked-by:` that names
it. It is never filed under an ad-hoc epic. Tasks 3d, 12d and 18d list them
in their acceptance.

**Ordering rule:** until Task 18d the user App has no `pull_requests:
write`, so every PR in this epic before then is submitted by the human, as
today. A PR touching `.github/workflows/**` is pushed by the human's
`vrg-submit-pr` until Task 3g.

---

## Task 2: `main` ruleset restricting updates

**Files:**

- Modify: `src/vergil_tooling/lib/github_config.py`
  (`desired_branch_protection_ruleset`, `compute_desired_state`)
- Test: `tests/vergil_tooling/test_github_config_lib.py`

**Interfaces:**

Both follow the existing ruleset convention (`desired_tag_protection_ruleset`,
`desired_automation_branch_ruleset`): `app_id: int | None`, where `None`
means the bypass list is left unmanaged (`bypass_actors is None` on
`DesiredRuleset`).

- Produces: `desired_main_update_ruleset(app_id: int | None, *,
  app_merges_releases: bool) -> DesiredRuleset`, named `"Main updates"`,
  `ref_include=["refs/heads/main"]`, one rule `{"type": "update"}`. Its
  bypass actors are the admin role, plus the App (`actor_type="Integration"`)
  only when Step 1 finds `app_merges_releases`.
- Produces: `desired_audit_marker_ruleset(app_id: int | None) ->
  DesiredRuleset`, named `"Agent audit markers"`,
  `target="branch"`, `ref_include=["refs/heads/vergil/agent-audit/*"]`, rules
  `update`, `creation` and
  `deletion`, bypass actors the admin role and the automation App, so the
  identity being audited cannot move the audit's markers (Task 12).

- [ ] **Step 1: Confirm the facts.** Record in the PR body:
  - which identity merges a release PR into `main` today (read the
    `merge-release` stage, `lib/release/merge.py`, which `--resume` and
    `--supersede` share, and a recent release PR's timeline);
  - that the `update` rule is available on the org's plan and refuses a PR
    merge into `main` by a non-bypass identity (GitHub docs, plus a dry run
    on a fixture).

  If the rule cannot do this, stop and ask the human. The fallback is the
  spec's §8 item.
- [ ] **Step 2: Write the failing tests.**

  ```python
  def _bypass_ids(rs):
      return {(a["actor_type"], a["actor_id"]) for a in rs.bypass_actors}

  def test_main_update_ruleset_restricts_updates():
      rs = desired_main_update_ruleset(123, app_merges_releases=False)
      assert rs.ref_include == ["refs/heads/main"]
      assert [r["type"] for r in rs.rules] == ["update"]

  def test_main_update_bypass_is_admin_and_optional_app():
      assert _bypass_ids(desired_main_update_ruleset(
          123, app_merges_releases=False)) == {("RepositoryRole", 5)}
      assert _bypass_ids(desired_main_update_ruleset(
          123, app_merges_releases=True)) == {
          ("RepositoryRole", 5), ("Integration", 123)}

  def test_audit_marker_ruleset_bypass_is_admin_and_app_only():
      rs = desired_audit_marker_ruleset(123)
      assert rs.ref_include == ["refs/heads/vergil/agent-audit/*"]
      assert _bypass_ids(rs) == {("RepositoryRole", 5), ("Integration", 123)}

  def test_new_rulesets_leave_bypass_unmanaged_without_app_id():
      assert desired_main_update_ruleset(
          None, app_merges_releases=False).bypass_actors is None
      assert desired_audit_marker_ruleset(None).bypass_actors is None
  ```

- [ ] **Step 3: Run them:** `vrg-container-run -- pytest
  tests/vergil_tooling/test_github_config_lib.py -k "main_update or
  audit_marker or new_rulesets"` → FAIL.
- [ ] **Step 4: Implement.** Add both rulesets to `compute_desired_state`.
  Pass `app_merges_releases=True` only if Step 1 found the release merging
  as the App, and thread `app_id` as `compute_desired_state` already does. Leave
  "Branch protection" unchanged.
- [ ] **Step 5:** Run the tests again → PASS. Then `vrg-container-run --
  vrg-validate`.
- [ ] **Step 6: Commit** `feat(github-config): restrict updates to main`.

## Task 2d (deployment): Release vergil-tooling; apply the `main` ruleset

- **Preconditions:** Task 2 released (the record check under "Releases"
  below). The release is human-gated.
- **Procedure:** the daily config job (`ops.yml`, `vrg-github-repo-config
  update`) converges every repo, creating the "Main updates" and "Agent
  audit markers" rulesets. Run it once by hand after the release.
- **Acceptance:**
  - the config audit reports no drift on either ruleset in any repo;
  - **before Task 18d**, a test PR into `main` on a fixture is refused by
    the ruleset when merged with the token of an identity that already holds
    `pull_requests: write` without bypass (the audit App, if provisioned;
    otherwise the human states which identity was used). A permission
    refusal does not count;
  - the next human release merges normally.

## Task 3a: Reusable workflows read App secrets from an environment

**Files (vergil-actions):**

- Modify: every reusable workflow job that reads `APP_CLIENT_ID` or
  `APP_PRIVATE_KEY`. Inventory them with `grep -rn
  "APP_CLIENT_ID\|APP_PRIVATE_KEY" .github/workflows actions`; at least
  `cd-release.yml`, the epic-sweep and rollup callers, and `cd-docs.yml`,
  `cd-docs-refresh.yml` and `publish-index.yml`, which gained optional App
  keys for the ops reporting of epic vergil-project/.github#1494 (its
  Task 4), and every job with an `ops-report` step that mints the App token.
- Modify: `.github/workflows/README.md` (the environment and why).

- [ ] **Step 1: Inventory.** List every job in every in-scope org that reads
  a credential that can write or act. That means vergil-project,
  logical-minds-foundry and mnemosys-project, from the local clones. For
  each read-only token PR CI needs, give its reason. The one known now is
  `CONAN_AUDIT_PROVIDER_TOKEN_CONANCENTER`: a read-only vulnerability
  database key that PR-time CI reads for the C/C++ audit
  (vergil-project/.github#1391 tracks its skip). The inventory confirms
  whether there are others, and names each with its reason. Also list, by file,
  every **caller** of a reusable workflow that reads the App keys (at least
  `cd-release.yml`, `ops-github-config.yml`, the epic sweep and rollup
  workflows, and the three docs and index workflows above): for example
  `.github`'s `ops.yml`, `epic-sweep.yml` and `epic-rollup.yml`, each
  consumer's `ops.yml`, each generated `cd.yml`, and the callers epic #1494
  gave explicit App-key maps (its Tasks 6a–6t). List only the in-scope orgs.
  Put the list in the PR body; it is the input for Tasks 3c, 3e and 3d.
- [ ] **Step 2: The check (static), written first.** A test asserts that
  every job reading a write-capable secret declares `environment:
  automation`. Run it → FAIL. The live checks need the converged environment
  and belong later: the branch-policy refusal to Task 3f, and the undeclared
  job seeing the secret unset to Task 3d.
- [ ] **Step 3:** Each such job declares `environment: automation`; its
  steps are unchanged. Run the test → PASS, then refactor with it green.
- [ ] **Step 4:** `vrg-container-run -- vrg-validate`. Commit
  `ci: read the automation App secrets from the automation environment`.

## Task 3b: Repo config converges the secrets environment

**Files:**

- Modify: `src/vergil_tooling/lib/github_config.py`
- Test: `tests/vergil_tooling/test_github_config_lib.py`

**Interfaces:**

- Produces: `desired_automation_environment() -> DesiredEnvironment`, with
  name `automation`, deployment branch policy custom branches `develop` and
  `main`, and no reviewers. It is fetched, diffed and applied alongside the
  rulesets.

- [ ] **Step 1: Write the failing tests:**
  - `test_automation_environment_allows_only_develop_and_main` asserts the
    branch set is exactly `{"develop", "main"}`;
  - `test_environment_drift_is_reported` asserts that an environment
    allowing `*` shows up in `compute_diff`.
- [ ] **Step 2:** Run → FAIL. Implement `DesiredEnvironment`, its fetch in
  `fetch_actual_state` and its diff and apply. Run → PASS.
- [ ] **Step 3:** `vrg-container-run -- vrg-validate`. Commit
  `feat(github-config): converge the automation environment`.

## Task 3c (one per repo): Callers that mint the token declare the environment

> **Amended 2026-11-24.** The rationale below, that environment secrets
> reach a reusable workflow only through `secrets: inherit`, is superseded:
> `inherit` works within one org, and the App keys passed by name reach a
> called workflow's environment-declaring jobs across orgs and within a
> repo (spec §4.2; see Evolution during execution).

- **The repos:** every repo in Task 3a's inventory, both those that mint an
  automation token in their own workflows and those that call a reusable
  workflow that reads the App keys.
- **Why `secrets: inherit` here.** Environment secrets reach a reusable
  workflow in another repo only when the caller passes `secrets: inherit`
  (the probe recorded at vergil-actions `cd-release.yml:74-78`,
  vergil-project/packages#6), and a calling job cannot declare an environment.
  This narrows the least-privilege rule of epic vergil-project/.github#189 on
  purpose, and only for these callers. It supersedes the explicit App-key
  maps added just before this epic: vergil-tooling 1a58cce1a2 (#4750, the
  generated `cd.yml`), vergil-claude-plugin 5d262961 and vergil-vm 6b51ead,
  and the maps epic vergil-project/.github#1494 added to the callers of the
  docs and index workflows.
  The over-sharing it permits is small
  once no org- or repo-level secret that can write or act remains (Task 3d),
  and the called workflows are in the protected core. Every other caller
  keeps its explicit map.
- **Each task:**
  - a job that mints the token itself declares `environment: automation`;
  - a job that calls an App-key reusable workflow passes `secrets: inherit`
    in place of its explicit App-key map;
  - `vrg-container-run -- vrg-validate`;
  - commit `ci: read the automation App secrets from the automation
    environment`.

  The PR is human-pushed until Task 3g.

## Task 3e: `repo_init` and the `secrets: inherit` allow-list check

**Files:**

- Modify: `src/vergil_tooling/lib/repo_init.py` (the generated `cd.yml` and
  ops callers pass `secrets: inherit` to the App-key workflows only,
  superseding the explicit App-key map that 1a58cce1a2 (#4750) added)
- Create: a `vrg-validate` audit check, `secrets_inherit`, that fails when a
  workflow passes `secrets: inherit` to any `uses:` outside the allow-list
  `INHERIT_ALLOWED = ("cd-release.yml", "ops-github-config.yml",
  "cd-docs.yml", "cd-docs-refresh.yml", "publish-index.yml", …)` taken
  from Task 3a's inventory. The CI already runs a semgrep `secrets-inherit`
  rule (with a `nosemgrep` on vergil-tooling's own `cd.yml`). Read it first:
  the new check replaces it or the rule is narrowed to the same allow-list,
  so there are never two definitions; the PR body says which.
- Test: `tests/vergil_tooling/test_repo_init.py`, a new
  `tests/vergil_tooling/test_check_secrets_inherit.py`

- [ ] **Step 1: Write the failing tests:**
  - `test_generated_cd_inherits_only_for_cd_release`;
  - `test_inherit_on_unlisted_workflow_fails`;
  - `test_explicit_map_passes`.
- [ ] **Step 2:** Run → FAIL. Implement. Run → PASS. Then `vrg-container-run
  -- vrg-validate`.
- [ ] **Step 3: Commit** `feat(ci-audit): allow secrets inherit only for the
  App-key workflows`.

## Task 3r (deployment): Release vergil-actions with 3a

- **Preconditions:** Task 3a released (the record check under "Releases"
  below). The release is human-gated and moves `v2.1`.
- **Acceptance:** the released reusable workflows declare `environment:
  automation` on every job that reads a write-capable secret. Callers (Task
  3c) merge only after this release, since they pin the released ref.

## Task 3f (deployment): Release vergil-tooling; apply the environment

- **Preconditions:** Tasks 3b and 3e released (the record check under
  "Releases" below). The release is human-gated.
- **Procedure:** the daily config job converges every repo, creating the
  `automation` environment. Run it once by hand after the release.
- **Acceptance:**
  - the config audit reports no drift on the `automation` environment in
    any repo;
  - a job declaring `environment: automation` on a non-`develop` branch is
    refused by the environment's branch policy before any step runs;
  - `vrg-validate`'s `secrets_inherit` check runs in a consumer repo.

## Task 3d (deployment): Move the secrets; verify

- **Preconditions:** Tasks 3r, 3c and 3f closed, so the workflows are
  released and the environment is converged everywhere. This task does not
  wait for the rest of the epic.
- **Procedure (the human):**
  1. Move **every write-capable secret in Task 3a's inventory** into each
     repo's `automation` environment: `APP_CLIENT_ID` and `APP_PRIVATE_KEY`,
     and every publishing token, such as `CARGO_REGISTRY_TOKEN` for Rust
     publishers. Every job that reads one declares the environment (Task 3a);
     callers already pass them through `secrets: inherit` (Task 3c). Dry run
     first on one repo: move them there alone, remove its repo-level copies,
     and re-run its release, its ops job and an epic sweep before moving on.
  2. Remove them all at org and repo level.
  3. Re-run a release, an epic sweep and a rollup.
  4. Check whether the org setting can cap the permissions a workflow may
     request for its own `GITHUB_TOKEN` below write, and if so cap it.
     Record the answer either way.
- **Acceptance:**
  - no org- or repo-level secret that can write or act remains;
  - #1185's acceptance criteria all hold: no workflow outside the
    environment can read the credentials, every minting job still works,
    and `configuration.md` describes the environment;
  - a workflow on a scratch branch that echoes whether the secret is set
    reports it unset.

## Task 3g (deployment): Grant the user App `workflows: write`

- **Preconditions:** Task 3d and Task 17d closed.
- **Procedure (the human):** grant the permission and accept it on every
  installation in the three in-scope orgs, and on none outside them (spec
  §4.11).
- **Acceptance:** an agent push of a workflow-touching branch is accepted,
  and the PR still needs the protected-change approval at submit.

## Task 4: Protected-path set and match

Removed. vergil-project/vergil-tooling#4775 shipped `lib/protected_paths.py`
(the fleet default, the per-repo registry and `changed_protected`).
By the human's ruling of 2026-11-24 there is no protected core, and
vergil-project/vergil-tooling#5117 removes the module and its tests (see
Evolution during execution).

## Task 5: Change fingerprint

**Files:**

- Create: `src/vergil_tooling/lib/change_fingerprint.py`
- Test: `tests/vergil_tooling/test_change_fingerprint.py`

**Interfaces:**

- Produces:
  - `change_fingerprint(repo: Path, base: str, head: str) -> str`: the
    SHA-256 of a normalised `git diff -U0 -M <base>...<head>`. The diff keeps
    the `diff --git`, `---` and `+++` file headers and every `+` and `-`
    line with its whitespace exactly. It drops the `@@` hunk headers (line
    numbers) and the `index`, `similarity index` and `dissimilarity index`
    lines, whose values change whenever `develop` edits the file elsewhere.
    For each binary file, the line is
    replaced by `binary <path> <head blob>`. An empty diff raises
    `FingerprintError`. (`git patch-id` is not used: it ignores whitespace,
    and plain `git diff` reports binaries without their contents.)
  - `commit_fingerprint(repo: Path, commit: str) -> str`, the same for
    `<commit>^..<commit>`.

- [ ] **Step 1: Write the failing tests** on a temp repo:
  - `test_fingerprint_survives_nearby_develop_edit`: edit a line two lines
    from the branch's hunk on base, rebase, and expect an equal fingerprint;
  - `test_fingerprint_survives_develop_edit_elsewhere_in_file`;
  - `test_conflict_resolution_changes_fingerprint`;
  - `test_fingerprint_ignores_line_shift`;
  - `test_whitespace_only_change_changes_fingerprint`;
  - `test_binary_change_changes_fingerprint`;
  - `test_rename_survives_develop_edit_to_renamed_file`;
  - `test_empty_diff_raises`.
- [ ] **Step 2:** Run → FAIL. Implement. Run → PASS. Then `vrg-container-run
  -- vrg-validate`.
- [ ] **Step 3: Commit** `feat(fingerprint): a rebase-stable change
  fingerprint`.

## Task 6a: Exemption request kind and the structured reply

> **Amended 2026-11-24.** Shipped by vergil-project/vergil-tooling#4777
> with a second kind, `protected-change`, which
> vergil-project/vergil-tooling#5117 removes by the human's ruling (see
> Evolution during execution). The text below keeps only the exemption.

**Files:**

- Create: `src/vergil_tooling/lib/rulings/approvals.py`
- Modify:
  - `src/vergil_tooling/lib/rulings/marker.py`: a new `type` attribute on
    the request marker, `merge-review-exemption`
    (`<!-- vrg-ruling kind=request type=merge-review-exemption -->`). The
    existing `kind=request|response` discriminator is unchanged, and existing
    requests carry no `type`. Planning item vergil-project/.github#892
    (authenticate every ruling) extends the same channel: read its current
    state first, and if it has a design, keep `type` from clashing with it
    and say so in the PR body;
  - `src/vergil_tooling/bin/vrg_ruling_reply.py`: refuse to post a reply
    to the exemption kind without exactly one line `Decision: approve` or
    `Decision: refuse`.
- Test: `tests/vergil_tooling/test_rulings_approvals.py`,
  `tests/vergil_tooling/test_vrg_ruling_reply.py`

**Interfaces:**

- Consumes: `change_fingerprint` (Task 5).
- Produces:
  - `RequestKind = Literal["merge-review-exemption"]`;
  - `render_exemption_request(reason: str, fingerprint: str) -> str`;
  - `parse_decision(body: str) -> Literal["approve", "refuse"]`, which raises
    `RulingError` on zero or several decision lines or on any other word;
  - CLI `vrg-ruling request --kind merge-review-exemption --issue <ref>
    --reason <text>`. It computes the change fingerprint itself from the
    worktree's `HEAD` and merge base and accepts no fingerprint argument.
    It posts the request as the agent and prints the exact
    `vrg-ruling-reply` command for the human;
  - **the turn is closed as for every ruling:** once the human's reply is
    verified, the agent runs `vrg-ruling respond` before any other comment,
    so the issue is free for its next ruling. The skill that raises the
    request (Task 15a) says so.

- [ ] **Step 1: Write the failing tests:**
  - `test_parse_decision_accepts_only_one_exact_line`, covering `approve`,
    `refuse`, `Approve`, two lines, and none;
  - `test_ruling_reply_refuses_missing_decision_for_new_kinds`;
  - `test_request_cli_computes_changes_and_fingerprint`: the kind takes no
    fingerprint argument;
  - `test_existing_requests_unchanged`: old-form markers still parse as
    requests;
  - `test_second_request_accepted_after_response`: an exemption exchange
    closed by `respond` leaves the issue open for a later ruling.
- [ ] **Step 2:** Run → FAIL. Implement. Run → PASS. Then `vrg-container-run
  -- vrg-validate`.
- [ ] **Step 3: Commit** `feat(rulings): protected-change and exemption
  requests`.

## Task 6b: Past-exchange verifier and lookup

**Files:**

- Modify: `src/vergil_tooling/lib/rulings/approvals.py`
- Test: `tests/vergil_tooling/test_rulings_approvals.py`

**Interfaces:**

- Consumes: the trust layer's comment stream, through
  `rulings.comments.read_stream` (`trust.read_comment_stream`, which pages
  every comment, checks the account type, and raises `TrustReadError` on a
  truncated read), and `identity.human_login(authors)`.
- Adds to `lib/rulings/identity.py`: `expected_agent_login(authors:
  TrustedAuthors) -> str`, mode-free, deriving `<human>-vergil-user` and
  checking it against the configured agents. The readers here, in Tasks 8,
  11 and 12, and in the human's submit, use it. `agent_login(authors, mode)`
  keeps its USER-mode check for posting.
- Produces:
  - `@dataclass(frozen=True) class Approval: kind: RequestKind;
    request_id: int; decision: str; blobs: dict[str, str]; fingerprint: str
    | None`;
  - `verified_approvals(issue_ref: str, *, authors: TrustedAuthors) ->
    list[Approval]`, built on that stream, which keeps a
    request only when the agent login posted it, the human login posted
    the reply, neither comment was edited or deleted, and the reply is the
    first trusted comment after the request. Comments after the reply are
    ignored. A comment the stream leaves out (the automation App's, an
    outsider's, or one edited by either) never counts as the reply. A read
    failure raises; it never returns `[]`.

- [ ] **Step 1: Write the failing tests:**
  - `test_past_approval_ignores_later_trusted_comments`;
  - `test_past_approval_rejects_edited_and_wrong_login`;
  - `test_untrusted_comment_between_request_and_reply_is_skipped`: an
    automation or outsider comment between them leaves the approval valid;
  - `test_refuse_decision_is_not_an_approval`;
  - `test_read_failure_raises`.
  - The trust guard test stays green: no comment is read outside the trust
    layer.
- [ ] **Step 2:** Run → FAIL. Implement. Run → PASS. Then `vrg-container-run
  -- vrg-validate`.
- [ ] **Step 3: Commit** `feat(rulings): verify past approvals`.

## Task 7: Merge-review record

**Files:**

- Create: `src/vergil_tooling/lib/merge_review/record.py`,
  `src/vergil_tooling/bin/vrg_merge_review_close.py` (entry point
  `vrg-merge-review-close`)
- Test: `tests/vergil_tooling/test_merge_review_record.py`

**Interfaces:**

- Consumes: `change_fingerprint`, `commit_fingerprint` (Task 5).
- Produces:
  - `RECORD_PREFIX = "Merge review record:"`;
  - `render_record(head: str, fingerprint: str, outcome: str) -> str`,
    giving `Merge review record: head=<sha> fingerprint=<hex>
    outcome=<eligible|not-eligible (<reason>)>`;
  - `parse_record(body: str) -> Record | None`;
  - `records(issue_ref: str, *, authors: TrustedAuthors) -> list[Record]`,
    built on `rulings.comments.read_stream` narrowed to the agent role, so
    from comments by the agent login only;
  - CLI `vrg-merge-review-close --issue <ref> --outcome <outcome>
    --body-file <file>`, which computes the head and fingerprint itself,
    from the worktree's `HEAD` and merge base or, for a batch member, that
    member's commit. It then posts the closing comment with the eligibility
    line first, the `Branch as reviewed:` line merge review now writes
    second, and the record line third. Read the current closing-comment
    shape in `skills/merge-review/SKILL.md` first and keep every line it
    has.

- [ ] **Step 1: Write the failing tests:**
  - `test_record_round_trip`;
  - `test_close_computes_fingerprint_not_caller` (the CLI takes no
    fingerprint argument);
  - `test_records_ignore_non_agent_authors`;
  - the trust guard test stays green;
  - `test_records_read_failure_raises`;
  - `test_incomplete_outcome_is_refused` (stopped, waiting, fatal and error
    are not outcomes).
- [ ] **Step 2:** Run → FAIL. Implement. Run → PASS. Then `vrg-container-run
  -- vrg-validate`.
- [ ] **Step 3: Commit** `feat(merge-review): a machine-readable closing
  record`.

## Task 8: Shared gate evaluator

> **Amended 2026-11-24.** The protected-files rule that
> vergil-project/vergil-tooling#4780 shipped is removed by
> vergil-project/vergil-tooling#5117 (see Evolution during execution);
> `evaluate` checks merge-review coverage only. The same task removes the
> human override below (`Override`, its comment format and `overrides`):
> coverage is a record or an approved exemption only.

**Files:**

- Create: `src/vergil_tooling/lib/agent_gates.py`
- Test: `tests/vergil_tooling/test_agent_gates.py`

**Interfaces:**

- Consumes: Tasks 5, 6b and 7.
- Produces:
  - `@dataclass(frozen=True) class GateResult: ok: bool; reasons:
    tuple[str, ...]`;
  - `evaluate(repo: Path, *, base: str, head: str, issue_refs: list[str],
    authors: TrustedAuthors)
    -> GateResult`. `authors` is the trusted-authors configuration from
    `load_trusted_authors()`: the local copy on the host and in VMs, the
    delivered `VRG_TRUSTED_AUTHORS` copy in Actions. Every shared reader
    takes it the same way. It applies one rule:
    - **merge review:** a single-issue PR needs a record, an approved
      exemption, or a human override, whose fingerprint equals
      `change_fingerprint(base, head)`.
  - **The human override** is defined here so Task 8 does not depend on
    Task 9. It is a comment by the human login on the task issue whose first
    line is `Merge review override: fingerprint=<hex> reason=<text>`. It is
    keyed by fingerprint, not by PR, since it is posted before the PR
    exists. `overrides(issue_ref, *, authors) -> list[Override]` is the one
    reader, used here and by the audit, built on
    `rulings.comments.read_stream` narrowed to the human role, so the human
    login is checked as in Task 6b.
      A batch needs, for each issue, a record matching that issue's commit
      by `commit_fingerprint`. A whole-PR record for `base..head` covers a
      fix pushed to an open batch.
  - Any lookup failure gives `ok=False` with the error as the reason.
  - `agent_involved(...) -> bool`, the one shared test of whether the
    rule applies. At submit, where no PR exists yet, it is true when the
    ready-state was recorded by an agent identity; nothing else is consulted
    there. For an
    open or merged PR (watch and audit), it is true when an agent login
    opened it or GitHub attributes any action on it to one. A PR that is
    not agent-involved passes the rule; the actor check (Task 12) still
    applies to it.

- [ ] **Step 1: Write the failing tests:**
  - `test_not_eligible_record_passes`;
  - `test_missing_or_stale_record_fails`;
  - `test_batch_needs_each_issues_own_commit_record`;
  - `test_exemption_stands_in_for_record`;
  - `test_overrides_read_failure_raises`;
  - `test_override_by_non_human_role_ignored`;
  - `test_unresolvable_task_issue_refuses`.
- [ ] **Step 2:** Run → FAIL. Implement. Run → PASS. Then `vrg-container-run
  -- vrg-validate`.
- [ ] **Step 3: Commit** `feat(gates): one evaluator for approvals and
  merge-review coverage`.

## Task 9: Submit: agents allowed, gates before push

> **Amended 2026-11-24.** Submit's approval gate, shipped by
> vergil-project/vergil-tooling#4785, is removed by
> vergil-project/vergil-tooling#5117 (see Evolution during execution); the
> merge-review gate stays, run on every submission. The same task removes
> `--human-only` and `--override-merge-review`, so the human-only and
> override tests below no longer apply.

**Files:**

- Modify: `src/vergil_tooling/bin/vrg_submit_pr.py`:
  - `main`'s identity gate (`is_agent()`): refuse AUDIT; refuse every agent
    with `--release`, in the CLI emergency mode (`--issue/--summary/--title`)
    and on the relay path (`--branches`); allow USER in the template and
    batch modes, which report-ready drives;
  - a new step in `_open_one`, before `_recheck_batch`, run for **every**
    submission, single-issue included: fetch `origin/develop`, and the
    branch only when `push` is false (a pushing submit must not move the
    `--force-with-lease` ref), then `agent_gates.evaluate`, all before any
    push or `gh pr create`, and after submit's existing trust-layer
    preflight;
  - the docstring and help text.
- Modify: `src/vergil_tooling/bin/vrg_gh.py` (`_DENIED_AGENT` messages
  point at `vrg-pr-workflow report-ready` and `vrg-submit-pr`; the denials
  stay) and `src/vergil_tooling/lib/github_config.py` (the auto-merge
  comment above the "Branch protection" ruleset).
- Test: `tests/vergil_tooling/test_vrg_submit_pr.py`,
  `tests/vergil_tooling/test_vrg_gh.py`

- [ ] **Step 1: Write the failing tests:**
  - `test_user_agent_submits`;
  - `test_agent_release_refused`;
  - `test_agent_cli_mode_refused`;
  - `test_agent_relay_refused`;
  - `test_audit_refused`;
  - `test_single_issue_submit_runs_gates`;
  - `test_relay_mode_gates_fetch_branch`, run as the human;
  - `test_gate_failure_leaves_no_pr_and_pushes_nothing`: the stubbed `gh
    pr create` and push are never called;
  - `test_human_submit_of_agent_pr_runs_gates`;
  - `test_human_only_pr_skips_both_gates`: a PR with no agent identity in
    its ready-state or its history needs no record;
  - `test_human_override_posts_comment_and_submits`: the human-only flag
    `--override-merge-review "<reason>"` posts, from the human login, the
    Task 8 override line for the change's fingerprint on the task issue. It
    then submits;
  - `test_agent_cannot_use_override`;
  - `test_vrg_gh_agent_denial_names_submit`.
- [ ] **Step 2:** Run → FAIL. Implement. Run → PASS. Then `vrg-container-run
  -- vrg-validate`.
- [ ] **Step 3: Commit** `feat(submit-pr): agents submit through the gates`.

## Task 10: Provenance: own-PR rules

**Files:**

- Modify: `src/vergil_tooling/lib/pr_provenance.py` (`_FORBIDDEN`,
  `evaluate`). Since epic vergil-project/.github#1467 the table is keyed by
  the roles `classify(identity, *, org, authors)` returns through the trust
  layer, `UNTRUSTED` among them. "Its own PR" compares the PR's author, as
  a `trust.Identity`, with the user agent's identity, never a bare login
  string.
- Test: `tests/vergil_tooling/test_pr_provenance.py`

- [ ] **Step 1: Write the failing tests:**
  - `test_user_creates_and_automerges_own_pr`;
  - `test_user_merge_of_own_pr_allowed` (the `merged_now` path);
  - `test_user_action_on_human_pr_forbidden`;
  - `test_user_approval_forbidden_even_on_own_pr`;
  - `test_audit_rules_unchanged`;
  - `test_untrusted_rules_unchanged`.
- [ ] **Step 2: Decide `edited`.** Read whether submit edits a PR after
  creating it, and allow `edited` on the agent's own PR only if it does.
  State the finding in the PR body.
- [ ] **Step 3:** Run → FAIL. Implement: USER's forbidden set applies to a
  PR whose author is not the user login, and `approved` stays forbidden
  everywhere. Run → PASS. Then `vrg-container-run -- vrg-validate`.
- [ ] **Step 4: Commit** `feat(provenance): the user agent may act on its own
  PR`.

## Task 11: Watch re-check; finalize refuses `--release` for agents

> **Amended 2026-11-24.** The watch's approval re-check, shipped by
> vergil-project/vergil-tooling#4786, is removed by
> vergil-project/vergil-tooling#5117 (see Evolution during execution); the
> watch re-checks provenance and merge-review coverage, and accepts no
> human override.

**Files:**

- Modify: `src/vergil_tooling/lib/finalize_watch.py`:
  `_check_provenance` becomes `_check_gates`, which runs provenance
  and `agent_gates.evaluate` at watch start and on each new head (the watch
  already re-checks provenance on every head change, in `_judge`); failures
  go through `_disable_auto_merge`. Before each evaluation it fetches the
  PR's head ref and the base, since a new head may have been made outside
  the clone. It refuses, without evaluating, when the clone's origin is not
  the PR's repository.
- Modify: `src/vergil_tooling/bin/vrg_finalize_pr.py` (an early refusal of
  `--release` for agents, at the existing `--release` gate, which runs
  before both `--cleanup-only` and the watch).
- Test: `tests/vergil_tooling/test_finalize_watch.py`,
  `tests/vergil_tooling/test_vrg_finalize_pr.py`

- [ ] **Step 1: Confirm re-arming.** Read how a PR's auto-merge behaves
  after a push today, and how a fixed PR is re-armed. Put the gates on that
  path, and record it in the PR body. Also confirm from `vergil-vm` that no
  agent VM shares a clone with the host over a mount, since the checkout
  lock is per clone. If one does, stop and ask the human.
- [ ] **Step 2: Write the failing tests:**
  - `test_clean_rebase_keeps_auto_merge`;
  - `test_recheck_read_failure_disarms`;
  - `test_recheck_fetches_unseen_head`;
  - `test_recheck_refuses_other_repo_clone`;
  - `test_agent_release_refused_before_watch`;
  - `test_agent_cleanup_only_release_refused`.
- [ ] **Step 3:** Run → FAIL. Implement. Run → PASS. Then `vrg-container-run
  -- vrg-validate`.
- [ ] **Step 4: Commit** `feat(finalize): re-check the gates on every new
  head`.

## Task 12: Daily audit: `vrg-agent-audit`

> **Amended 2026-11-24.** The approval check and the
> `merge-changed-approved-file` condition, shipped by
> vergil-project/vergil-tooling#4787, are removed by
> vergil-project/vergil-tooling#5117 (see Evolution during execution); the
> merge-review, actor and `main` checks stay. The audit accepts no human
> override: check 1 below is met by a record or an exemption only.

**Files:**

- Create: `src/vergil_tooling/lib/agent_audit.py`,
  `src/vergil_tooling/bin/vrg_agent_audit.py` (entry point
  `vrg-agent-audit`)
- Test: `tests/vergil_tooling/test_agent_audit.py`

**Interfaces:**

- Consumes: `agent_gates.evaluate`, `pr_provenance.classify` and
  `check_pr`, `load_trusted_authors()`.
- Produces: `vrg-agent-audit --repo <owner/name> [--init]`.
  - **The human and agent logins** come from the trusted-authors
    configuration, loaded with `load_trusted_authors()`: in Actions, the
    delivered `VRG_TRUSTED_AUTHORS` copy every content-reading workflow
    already receives. There is no audit-specific variable or flag. The
    audit refuses (exit 1) when the configuration is missing, or names no
    human or more than one, as `rulings.identity.human_login` already does.
  - **Two markers** in the audited repo, the branches
    `vergil/agent-audit/develop`
    and `vergil/agent-audit/main` (rulesets cover only branches and tags, so
    they are branches, protected by Task 2), each holding the last audited merge
    commit on its branch. `--init` sets both to the current tips and audits
    nothing; it is used only by repo init (Task 12b) and the one-time
    go-live loop (Task 12d). Each branch's merges are caught up from its own
    marker. A scheduled run that finds a marker missing reports it as a
    `missing-marker` finding (exit 1) and never re-creates it. The marker
    names and the create helper live here, as the one definition Task 12b
    reuses.
  - It exits 0 when clean, 1 on any finding or read failure, and 2 on a
    usage error.
- Checks, per merge since each branch's marker:
  1. a merge-review record, exemption or human override covering the merged
     head's change
     (agent-involved PRs only: opened by an agent login, or with any
     action attributed to one; the human's own and the automation App's
     own PRs are exempt);
  2. the merge or auto-merge actor is in the expected set, judged by
     `classify` with account types: the human, the user agent, or the
     automation App, in its own org, on PRs it authored. An `UNTRUSTED`
     actor is always reported;
  3. any merge into `main` by anyone but the human, or the automation App
     where Task 2 found it merges releases.

  The audit is a security-signal job (spec §4.8): it files no issue and
  posts nothing. Its findings are its exit status and its run log, which
  `hoshin` reads (Task 15f).

- [ ] **Step 1: Confirm the pusher record.** Find which GitHub API records
  name the identity that pushed a commit to a PR branch, and use one if
  any is reliable. State the answer in the PR body.
- **Reading a merged PR.** Merged branches are deleted
  (`delete_branch_on_merge`), so the audit fetches `refs/pull/<n>/head` for
  the head and uses the merge commit's first parent as `base`.
- [ ] **Step 2: Write the failing tests:**
  - `test_init_sets_marker_and_audits_nothing`;
  - `test_missing_marker_is_a_finding`: a scheduled run with no marker
    exits 1, names the repo and branch, and creates nothing;
  - `test_missed_run_is_caught_up_from_marker`;
  - `test_main_merges_caught_up_from_main_marker`;
  - `test_missing_trusted_authors_configuration_refuses`;
  - `test_untrusted_actor_reported`;
  - `test_automation_only_pr_skips_record_check`;
  - `test_unexpected_actor_reported`;
  - `test_main_merge_by_agent_reported`;
  - `test_read_failure_exits_1`;
  - `test_merged_squash_pr_read_from_pull_ref`;
  - `test_merged_rebase_batch_read_from_pull_ref`.
- [ ] **Step 3:** Run → FAIL. Implement. Run → PASS. Then `vrg-container-run
  -- vrg-validate`.
- [ ] **Step 4: Commit** `feat(audit): a daily audit of agent merges`.

## Task 12b: Repo init creates the audit markers

**Files:**

- Modify: `src/vergil_tooling/lib/repo_init.py`: a new resumable step,
  `step_audit_markers`, after `step_github_config`. It creates
  `vergil/agent-audit/develop` and `vergil/agent-audit/main` at the current
  tips of `develop` and `main`, with Task 12's marker names and create
  helper (one definition). It creates only a missing marker and leaves an
  existing one untouched. The `--adopt` path runs it too.
- Test: `tests/vergil_tooling/test_repo_init.py`

`vrg-github-repo-init` stays human-only. It runs as the admin role, which
the "Agent audit markers" ruleset (Task 2) allows to create these branches,
so a repo made or adopted after the gates release (Task 17d) needs no
further step.

- [ ] **Step 1: Write the failing tests:**
  - `test_init_creates_both_audit_markers_at_branch_tips`;
  - `test_adopt_creates_missing_markers_only`;
  - `test_existing_marker_left_untouched`;
  - `test_marker_creation_failure_stops_init`: a failed create stops init
    with the error; it is never skipped.
- [ ] **Step 2:** Run → FAIL. Implement. Run → PASS. Then `vrg-container-run
  -- vrg-validate`.
- [ ] **Step 3: Commit** `feat(repo-init): create the agent audit markers`.

## Task 13: The audit job

> **Amended 2026-11-24.** The protected-path registries are dropped by the
> human's ruling (see Evolution during execution):
> vergil-project/vergil-actions#1341 is narrowed to the audit workflow, and
> 13 (docs), vergil-project/docs#145, is closed. By ruling N0.14, the
> `agent-audit` job runs after the config job with
> `if: ${{ !cancelled() }}`, so a failing config job does not skip it. By
> ruling N0.15, the 14 vergil-project callers are filed under each repo's
> ad-hoc epic (see Evolution during execution).

- **vergil-actions, the audit workflow:** a reusable workflow,
  `ops-agent-audit.yml`, that runs `vrg-agent-audit` for its calling repo as
  the automation App, in the `automation` environment. It passes the org
  variable `VRG_TRUSTED_AUTHORS` through, as `ops-github-config.yml` already
  does. It is a security-signal job (spec §4.8): it files no issue and has
  no `ops-report` step. It is an App-key workflow, so it goes on
  Task 3e's `secrets: inherit` allow-list. It is part of 13 (A).
- **Every repo's `ops.yml`, in the three in-scope orgs:**
  calls `ops-agent-audit.yml` after the config job, as a job named
  `agent-audit`, the name `hoshin` reads it by (Task 15f). This includes `.github`'s
  own `ops.yml`, filed as 13 (G). The other repos' changes are made by the
  same caller update as Task 3c, or by the fleet sync where it manages
  `ops.yml`. Callers merge only after Task 13r has released
  `ops-agent-audit.yml`, since they pin the released ref.
- Each is a separate PR in its own repo: 13 (A) and 13 (G), plus one per
  further caller. Each runs `vrg-container-run -- vrg-validate` and commits
  `feat(ops): the daily agent audit`.
- **The released-version check (spec §4.4)**, done in 13 (G) and recorded in
  its PR body:
  - confirm that consumer repos, the agent VMs and the audit job run
    released vergil-tooling and plugin versions;
  - confirm whether vergil-tooling's own CI runs the PR's code. If it does,
    that repo's PRs check themselves. This is the accepted seam of spec §3,
    covered by merge review and the human-gated release, and needs no
    further mechanism; the PR body says so.

## Task 14: Plugin hooks' wording

**Files (vergil-claude-plugin):**

- Modify, each with its own message; the blocks stay:
  - `hooks/scripts/block-agent-merge.sh:54`: agents merge only through
    `vrg-submit-pr`'s auto-merge, and never approve;
  - `hooks/scripts/block-github-contents-api.sh:6,69`: file changes go
    through the worktree, `vrg-commit`, report-ready and the agent's own
    `vrg-submit-pr` (in place of "the human submits the PR").
- `hooks/scripts/block-raw-gh-pr-create.sh` already points at
  `vrg-submit-pr` and is left alone unless its text is found stale.
- Test: `hooks/tests/command-matchers.test.sh` (denial text assertions).

- [ ] **Step 1:** Update the expected strings in the tests → FAIL.
- [ ] **Step 2:** Change the messages → PASS. Then `vrg-container-run --
  vrg-validate`. Commit `fix(hooks): point agents at vrg-submit-pr`.

## Task 15a: `merge-review`: record, approvals, exemption

**Files:** `skills/merge-review/SKILL.md`, `agents/merge-reviewer.md` if it
names the handoff.

- The closing comment is posted with `vrg-merge-review-close`, from both
  entry points.
- **A protected change:** the skill runs `changed_protected` through `vrg-ruling
  request --kind protected-change`, gives the human the printed command,
  and stops until the approval is verified.
- **An exemption:** only on the human's instruction, through `vrg-ruling
  request --kind merge-review-exemption`.
- Merge review runs on every PR; its docs-only class is unchanged.
- **Check:** the plugin's skill lint and tests pass under
  `vrg-container-run -- vrg-validate`. Commit `feat(merge-review): records,
  approvals and exemptions`.

## Task 15b: `issue-implement` owns the PR; one shared watch procedure

**Files:** `skills/issue-implement/SKILL.md`, `skills/pr-watch/SKILL.md`.

- `issue-implement` runs, after report-ready: `vrg-submit-pr` (which
  always finalizes), then the watch procedure until a merge or a human stop.
- The watch procedure lives once, in `pr-watch`. `issue-implement` invokes
  it; `pr-watch` stays the standalone entry for an open PR.
- **`pr-watch` changes:**
  - it reruns `vrg-finalize-pr` itself after a fix;
  - it pushes a `not-eligible` fix once every ruling is answered. It
    already does so for a fix whose only stop was an answered ruling
    (`Branch as reviewed: yes`); extend that rule rather than adding a
    second one;
  - it never runs anything with `--release`.
- **Removed:** "Only the human opens the PR" and the workflow-push special
  case. The cloud relay section stays.
- **Added, one line:** if a push is refused for workflow files (before Task
  3g), stop and hand off: report-ready has recorded the state, and the
  human's `vrg-submit-pr` pushes it.
- **Check:** the plugin's tests pass under `vrg-container-run --
  vrg-validate`. Commit `feat(skills): the implementer owns the PR to
  merge`.

## Task 15e: Docs, retrospective and logbook PRs run merge review

- `epic-create`'s spec-and-plan PR, `epic-retrospective`'s PR and `hoshin`'s
  logbook PR run merge review before report-ready, closing with
  `vrg-merge-review-close`. Their `docs-only` class keeps it light.
- `pr-watch`'s skip of merge review for a fix to a `Logbook:` PR goes: a
  logbook fix is merge-reviewed like any other, so the watch's re-check
  (Task 11) finds a record covering it.
- This needs no new permission, so it ships in plugin release A (Task 15r),
  before the gates. The human's submits of these PRs then pass from the
  start. Who submits them is unchanged here (Tasks 15c and 15d).
- **Check:** plugin tests pass. Commit `feat(skills): docs and logbook PRs
  run merge review`.

## Task 15f: `hoshin` reads the agent audit as a security signal

**Files:** `skills/hoshin/references/daily.md`, and
`skills/hoshin/references/security.md` if it lists the jobs.

- Add the `agent-audit` job (Task 13) to the security-signal jobs the daily
  review reads from run history, beside `github-config` and
  `dependabot-audit`. A failed run is "a security-related failure" with its
  security record's neutral ref, created first if it has none, exactly as
  for those two.
- A repo with no `agent-audit` runs yet is not a finding: the job goes live
  only at Task 12d. From 12d on, a scheduled run missing in an in-scope repo
  is flagged as not read.
- No new permission is needed, so it ships in plugin release A (Task 15r),
  before the audit goes live.
- **Check:** plugin tests pass under `vrg-container-run -- vrg-validate`.
  Commit `feat(hoshin): read the agent audit as a security signal`.

## Task 15c: `epic-implement`, `epic-create`, `epic-retrospective`, `fixture-run`

- **`epic-implement`:** after a merge, re-read the epic and continue with
  the next runnable task. Stop for the human only on a ruling, an approval,
  a failure, or a task blocked on a release or a human-gated deployment
  precondition (spec §4.10). The next runnable task may be a proxy task in
  another org; continuing covers it, through the primary epic. Drop "Your
  responsibility ends here" and the retrospective-PR line.
- **`issue-deploy`:** its line that agents hand off `vrg-submit-pr`, by the
  same policy that keeps releases human-gated, moves to the new boundary:
  agents submit; releases stay human-gated.
- **The other three:** the agent submits its docs or fixture PRs, after the
  merge review Task 15e added. Fixture releases stay human-gated.
- **Check:** plugin tests pass under `vrg-container-run -- vrg-validate`.
  Commit `feat(skills): agents submit epic and fixture PRs`.

## Task 15d: `hoshin` on the protected-change approval

Dropped by the human's ruling of 2026-11-24: with no protected core there
is no approval for the logbook sign-off to become, and
vergil-project/vergil-claude-plugin#1205 is closed (see Evolution during
execution). vergil-project/vergil-claude-plugin#1326 removes the approval
wording from `hoshin` and the other skills.

## Task 16a: Consumer CLAUDE.md template and the permission-model spec

- **Files (vergil-tooling):**
  - `src/vergil_tooling/data/claude_md_consumer.md` (its "PR handoff"
    section);
  - `docs/specs/2026-05-29-agent-permission-model-design.md` (the "human
    triggers" principle and the user App table);
  - the PR-workflow text that tells the agent the human submits:
    `lib/pr_workflow/freeze.py` (the freeze refusal message),
    `lib/pr_workflow/engine.py`, `lib/pr_workflow/state.py` and
    `lib/branch_sweep.py`. Grep `src/` for "human submits" and "vrg-submit-pr"
    for any others. Each repo's own CLAUDE.md that the fleet sync does not
    manage from this template is listed in the PR body for the documentation
    review (R1).
- **The new rule text:**
  - agents submit and finalize through the tools, never with `--release`;
  - releases and promotion are human actions;
  - a protected-path change needs the human's approval;
  - the freeze runs from report-ready to the agent's own submit;
  - until `workflows: write` is granted, a refused workflow push is handed
    off as before.

  The workflow-push paragraph becomes that one line, and goes after Task
  3g.
- **Check:** `vrg-container-run -- vrg-validate`. Commit `docs: the agent
  boundary moves to releases`.

## Releases: the order that makes the rollout safe

There is no per-repo switch (spec §5). Each release is fleet-wide. The
order is what matters: no check ships before anything can pass it, and no
instruction ships before its permission. Every release is human-gated.
"Release" means the first ordinary release of that repo containing the
tasks named.

**Confirming a release carries a task** (epic vergil-project/.github#1440).
Each deployment task's first precondition step is, for every task it is
blocked by: `vrg-release-record check <owner>/<repo>#<N>` exits 0 (`pass`).
Exit 3 (`fail`) is an unmet precondition. Exit 4 (no record) falls back to
`vrg-git merge-base --is-ancestor <merge-sha> <tag-sha>` exiting 0 for the
merge commit of the PR that closed the task. Exit 1 is a read error and a
stop. Never a release-notes or changelog check.

**The install is part of the release.** A vergil-tooling or plugin release
runs its own install (`[publish].install`: the tooling install and
`vrg-vm update --all`, or `vrg-host plugins` and `vrg-vm update --all`), and
its `Released:` record states the outcome. A deployment step that needs the
release installed checks that the record's Install line reads `ran`; it
runs no install by hand. After a plugin release, the session still needs
`/reload-plugins` or a new session.

## Task 7d (deployment): Release vergil-tooling with the record and request commands

- **Preconditions:** Tasks 5, 6a, 6b and 7 released (the record check
  above).
- **Procedure:** the next vergil-tooling release; its record's Install line
  reads `ran`.
- **Acceptance:** `vrg-merge-review-close --help` and `vrg-ruling request
  --kind protected-change --help` run in a VM.

## Task 15r (deployment): Plugin release A: records and hook wording

- **Preconditions:** Task 7d done; Tasks 14, 15a, 15e and 15f released (the
  record check above).
- **Procedure:** release the plugin; its record's Install line reads `ran`.
- **Acceptance:** the next merge review on any task issue, a docs or
  logbook PR included, posts a closing comment carrying a `Merge review
  record:` line. No check reads it yet.

## Task 13r (deployment): Release vergil-actions with 13 (A)

- **Preconditions:** Task 13 (A) released (the record check above). The
  release is human-gated and moves `v2.1`.
- **Acceptance:** `ops-agent-audit.yml` is callable at the released ref.
  The `ops.yml` callers (Task 13 (G) and the others) merge only after this.

## Task 17d (deployment): Release vergil-tooling with the gates

> **Amended 2026-11-24.** The human waived the acceptance check that an
> agent submit still fails at GitHub, as covered by Task 18d (see
> Evolution during execution).

- **Preconditions:** Task 15r done, so merge review already writes records;
  Tasks 8, 9, 10, 11, 12 and 12b, and the approval removal
  vergil-project/vergil-tooling#5117, released (the record check above).
  Task 4 is removed (see Evolution during execution).
- **Procedure:** the next vergil-tooling release; its record's Install line
  reads `ran`.
- **Acceptance:**
  - in a VM, `vrg-submit-pr --help` no longer says "human-only";
  - the human's next ordinary `vrg-submit-pr` passes the new gates, using
    the record merge review wrote;
  - an agent submit still fails, at GitHub, for lack of `pull_requests:
    write` (granted in Task 18d).

## Task 21d (deployment): Plugin release B and the steering

- **Preconditions:** Task 18d done; Tasks 15b, 15c and 16a, and the
  approval removal vergil-project/vergil-claude-plugin#1326, released (the
  record check above). Task 15d is dropped (see Evolution during
  execution).
- **Procedure:** release the plugin and vergil-tooling (for 16a's
  template); each record's Install line reads `ran`.
- **Acceptance:** an `issue-implement` run on a fixture task submits its PR
  and carries it to merge without a human step.

## Task 12d (deployment): Audit live fleet-wide, marker set, green

- **Preconditions:** Task 17d released; Task 13r released; every Task 13
  caller PR merged (the docs registry excepted); Task 15r released, so
  `hoshin` reads the audit's runs.
- **Procedure:** run `vrg-agent-audit --init` for every repo of the three
  in-scope orgs (archived repos excepted), then let one scheduled run pass.
  This `--init` loop is a one-time human-run step of the rollout, for the
  repos that predate Task 12b: the "Agent audit markers" ruleset (Task 2)
  lets only the admin role and the automation App create the
  `vergil/agent-audit/*` branches, so the user agent cannot set them. It
  skips a repo that already has both markers, such as one made by repo init
  after Task 17d. It never recurs: later repos get their markers from repo
  init.
- **Acceptance:**
  - a list, by in-scope org, of every repo the user App is installed on,
    each with both markers;
  - the scheduled run exits 0 in every one;
  - a deliberate dry run against a known human-merged PR reports nothing;
  - the next `hoshin` daily review lists the `agent-audit` runs among its
    security-signal jobs.

## Task 18d (deployment): Grant the user App `pull_requests: write`

- **Preconditions:** Tasks 2d and 12d closed.
- **Procedure (the human):** change the App's permission and accept it on
  each installation whose repos all appear in Task 12d's list with their
  markers. An installation not on that list is not granted until it is,
  and no installation outside the three in-scope orgs is granted (spec
  §4.11).
- **Acceptance:**
  - a hand-run agent `vrg-submit-pr` on a test PR in a fixture opens it
    with auto-merge on, and it merges;
  - a merge into `main` with the user agent's own token is now refused by
    the ruleset, repeating Task 2d's check with this identity.

## Task 16b: `.github` CLAUDE.md on the new boundary

The same rule text as Task 16a, in the "PR handoff" section of
`.github/CLAUDE.md`. It is filed
after 18d so the steering never runs ahead of the permission. Commit `docs:
the agent boundary moves to releases`.

## Task 19: Fixture runs

File with `fixture-run`, after Tasks 21d and 3g exist:

```text
fixture-run vergil-project/.github#1384 \
  python-app,python-lib,rust-app,rust-lib,ts-app,ts-lib \
  --validates "agent-submitted PRs (the Task 21d releases)" \
  --after vergil-project/.github#<21d> --after vergil-project/.github#<3g>
```

These are the first live agent cycles (spec §5 step 5). The agent submits
each run's change PR and carries it to merge. The release stays the
human's.

## Task 19v (validation): Exemption and audit on one fixture

- **Preconditions:** Task 19's runs closed on `vergil-fixture-python-lib`.
- **Procedure:**
  1. An agent PR with the human's merge-review exemption merges.
  2. An **agent-opened** PR with no merge-review record or exemption is
     merged by the human through the API, deliberately bypassing submit.
     (A human-only PR would be exempt, so it would prove nothing.)
- **Acceptance:** case 1 merges; the next audit run reports case 2 by
  name, and nothing else.

## Bookends

- **Task 1 (#1385):** this spec and plan.
- **R1 (vergil-project/vergil-tooling#4751):** documentation review. It
  covers the site docs on the new boundary, the approval and exemption
  commands, the audit and the release order, and spawns per-repo doc tasks
  as needed. Known places already on the old boundary: the plugin's
  `docs/site/docs/hooks/index.md`, `docs/site/docs/skills/index.md`,
  `docs/development/skills-architecture.md` and `README.md`; vergil-tooling's
  guides `git-workflow.md`, `permission-model.md`,
  `identity-architecture.md`, `credential-management.md`, `account-setup.md`
  and `cli-tools-overview.md`; and any repo CLAUDE.md Task 16a lists.
- **R2 (#1386):** retrospective, with `epic-retrospective`.

## Evolution during execution

- **2026-11-23: Task 6a waits for Tasks 4 and 5.** The first run of Task 6a
  (vergil-project/vergil-tooling#4777) found that its request commands
  compute their inputs with `changed_protected` and `ProtectedChange` (Task
  4) and `change_fingerprint` (Task 5), which the plan did not list. By the
  human's ruling, 6a waits for both. The delivery table, the side-by-side
  note and the Task 6a section were edited in place by
  vergil-project/.github#1558, before any implementation task had shipped.
- **2026-11-23: the Task 3c caller tasks are filed.** The Task 3a run
  (vergil-project/vergil-actions#1339) filed them by the human's ruling,
  each blocked by Task 3r (vergil-project/vergil-actions#1340):
  - **vergil-project**, under this epic: vergil-project/.github#1563,
    vergil-project/docs#171, vergil-project/packages#63,
    vergil-project/vergil-actions#1388,
    vergil-project/vergil-claude-plugin#1297,
    vergil-project/vergil-containers#1014,
    vergil-project/vergil-fixture-python-app#68,
    vergil-project/vergil-fixture-python-lib#75,
    vergil-project/vergil-fixture-ts-app#86,
    vergil-project/vergil-fixture-ts-lib#85,
    vergil-project/vergil-fixture-rust-app#83,
    vergil-project/vergil-fixture-rust-lib#89,
    vergil-project/vergil-python#98, vergil-project/vergil-tooling#5060 and
    vergil-project/vergil-vm#447.
  - **logical-minds-foundry**, under its new proxy of this epic,
    logical-minds-foundry/.github#422: logical-minds-foundry/.github#423,
    logical-minds-foundry/docs#40,
    logical-minds-foundry/mq-gateway-replay-lab#346,
    logical-minds-foundry/mq-resiliency-lab-for-linux#1757 and
    logical-minds-foundry/mq-resiliency-observability#87.
  - **mnemosys-project**, under its new proxy of this epic,
    mnemosys-project/.github#122: mnemosys-project/.github#123,
    mnemosys-project/docs#35 and mnemosys-project/melete#278.

  Task 3d's precondition, every caller task closed, now means these 23.
- **2026-11-23: a new task gives the config job's token Actions read.**
  The Task 3b run (vergil-project/vergil-tooling#4773) found that GitHub
  lists the environment reads under the App's Actions read permission,
  which the token minted by `ops-github-config.yml`'s update job did not
  request. Without it, the nightly `update` would report the `automation`
  environment as not checked after Task 3f. By the human's ruling,
  vergil-project/vergil-actions#1387 adds `permission-actions: read` to that
  token. It lands before Task 3f (vergil-project/vergil-tooling#4783), and
  each in-scope org's automation App is granted Actions read by the human
  before it merges.
- **2026-11-23: Task 3d also moves `GHCR_CLEANUP_TOKEN`.** The Task 3a
  inventory found this write-capable token, read by vergil-containers'
  `package-cleanup.yml` cleanup and reap-aliases jobs, which the plan did not
  name. The human added it to Task 3d (vergil-project/.github#1410) by a
  scope change on that issue.
- **2026-11-23: Task 2d drops the live refusal test.** By the human's
  ruling, a scope change on vergil-project/vergil-tooling#4782 drops the
  test of GitHub's own enforcement of the `main` update rule as overkill.
  2d's acceptance is now the fleet-wide converge plus the next human
  tooling release merging into `main` normally, and the task stays open
  until that release.
- **2026-11-23: one converge covered Tasks 2d and 3f.** Both shipped in
  the same release (v2.1.339), so one `vrg-github-repo-config update` per
  repo converged both. The issue bodies of
  vergil-project/vergil-tooling#4782 and #4783 labelled human steps as
  agent-safe, such as "run the daily config job by hand", which an agent
  cannot do since `vrg-gh` refuses `workflow run`. #4783's body also
  lacked the Actions-read precondition from
  vergil-project/vergil-actions#1387.
- **2026-11-23: two repos needed unblocking before their Task 3c callers
  could land.**
  - mnemosys-project/melete#265: the dev image had no `npm`. Its first fix
    hung on an interactive `tzdata` prompt, a problem captured fleet-wide
    as vergil-project/.github#1577.
  - logical-minds-foundry/mq-resiliency-observability#72: `librt` 0.13.0
    had no CPython 3.15 wheel for the docs job.
- **2026-11-23: Task 3d's rulings.** By the human's rulings on
  vergil-project/.github#1410, the signing keys count as write-capable,
  and `PROJECT_TOKEN` is deleted everywhere and its PAT revoked. Two tasks
  were filed:
  - logical-minds-foundry/mq-resiliency-observability#91, under
    logical-minds-foundry/.github#422, fixes the `docs` caller in that repo
    that Task 3c missed.
  - vergil-project/vergil-actions#1403, under this epic, meets the docs
    criterion of vergil-project/vergil-actions#1185. Task 3d's SUCCESS
    waits for it.
- **2026-11-23: Task 9 gains `recorded_by`.** The Task 8 run
  (vergil-project/vergil-tooling#4780) found that the ready state records
  no identity, so submit could not tell an agent-prepared branch from a
  human one. By the human's ruling, a scope change on
  vergil-project/vergil-tooling#4785 makes `report-ready` record that
  identity as `recorded_by`, and a missing or unknown value counts as
  agent-involved.
- **2026-11-23: a note for Task 15d.** Once the docs logbook paths are
  protected, merge review (Task 15a) asks for the protected-change approval
  itself, so 15d's own `vrg-ruling request --kind protected-change` step
  must not ask the human a second time. The Task 15e run
  (vergil-project/vergil-claude-plugin#1202) found this.
- **2026-11-24: the protected core's approval is removed.** By the human's
  ruling N0.12 on the epic, there is no protected-path set, no
  protected-change approval and no registry. The epic moves the trust bar
  to the agents: workflow and `vergil.toml` changes are worked out in specs
  and plans, and merge review judges each one, so a human approval on every
  such change would be a rubber stamp, the very approval without looking
  that the spec (§1.1) argues protects nothing. Merge review on every PR,
  the human-approved merge-review exemption and the human's
  `--override-merge-review` stay. The spec is amended in place
  (vergil-project/.github#1588).
  - **Removal tasks.** vergil-project/vergil-tooling#5117 removes the
    approval from the gates, the audit and the rulings; no vergil-tooling
    release ships until it merges. vergil-project/vergil-claude-plugin#1326
    removes it from `merge-review` and the skills Tasks 15a and 15e touched,
    for the next plugin release.
  - **Tasks that change.** Task 4 is removed. Tasks 6a (its
    `protected-change` kind), 8, 9, 11 and 12 lose their approval parts,
    each pointing to vergil-project/vergil-tooling#5117. Task 13 keeps the
    audit workflow and its callers and drops the registries:
    vergil-project/vergil-actions#1341 is narrowed, and 13 (docs),
    vergil-project/docs#145, is closed. Task 15d,
    vergil-project/vergil-claude-plugin#1205, is dropped, so Task 21d no
    longer waits for it. Task 19v, vergil-project/vergil-fixture-python-lib#39,
    validates the exemption and the audit only. The global constraints and
    the review focus list drop the approval; those sections and the task
    text above were edited for it. Task 17d's preconditions now wait for
    vergil-project/vergil-tooling#5117 in place of Task 4, and Task 21d's
    for vergil-project/vergil-claude-plugin#1326 in place of Task 15d, in
    their sections and in the delivery table.
  - **Text left as written, superseded by this entry.** The Goal and
    Architecture paragraphs, Task 3g's acceptance, Task 6b's `blobs` field,
    Task 7d's `--kind protected-change` acceptance, Task 15a's protected
    change step, Task 15c's stop on an approval, and the protected-path
    line of the rule text in Task 16a (and so Task 16b) all still name the
    approval. None of it applies any more.
  - **`secrets: inherit` across orgs.** The spec's §4.2 is also corrected:
    `inherit` carries the App keys, environment secrets included, only
    within one org, and the callers in logical-minds-foundry and
    mnemosys-project pass them by name (rulings N0.6 to N0.8). Whether a
    by-name map delivers a secret that exists only in the environment
    waits on the probe recorded on vergil-project/vergil-tooling#5108
    (rulings N0.9 and N0.10). Task 3c's rationale above states the old
    premise and is superseded the same way.
  - **Two earlier findings under the hostile-agent anti-goal.** Task 9's
    merge-review finding R1 (vergil-project/vergil-tooling#4785) was fixed
    by running the gates on every submit. That fix goes beyond the spec's
    anti-goal of defending against a deliberately hostile agent, and the
    human kept it. vergil-project/vergil-vm#453 (ruling N0.11) was closed
    as not planned under the same anti-goal; its record, tracked by
    vergil-project/.github#1586, stays as an accepted property of the VM
    design.
- **2026-11-24: Tasks 9, 11 and 12 start before Task 15r closes.** The plan
  held them until Task 15r (vergil-project/vergil-claude-plugin#1203)
  closed, but 15r's acceptance needs the first merge review after plugin
  release 2.1.88, and those tasks are what would run one. By the human's
  ruling N0.5, they started at once; 15r closes on the first `Merge review
  record:` line their reviews post, and no PR of theirs is submitted
  before 15r closes.
- **2026-11-24: the human's submit-time skips are removed too.** By the
  human's addition to ruling N0.12, vergil-project/vergil-tooling#5117 also
  removes `vrg-submit-pr`'s `--human-only` and the human merge-review
  override (`--override-merge-review`, its comment format and its
  acceptance in the watch and the audit), since agents write every PR.
  Merge-review coverage is now a merge-review record or an approved
  exemption only, and submit runs the gate on every submission. The
  earlier entry's statement that `--override-merge-review` stays is wrong
  and superseded by this one. The spec's §3, §4.5, §4.6 and §8, the global
  constraints, and the amendment notes of Tasks 8, 9, 11 and 12 are
  corrected in place; the override and human-only text in those tasks'
  bodies is left as written, superseded.
- **2026-11-24: the spec's §3 now agrees with §1.1 on the gate logic.**
  §1.1 treats an honest agent's weakening of a check as an ordinary change
  that merge review judges, while §3 put any weakening of the gate logic in
  the hostile-agent seam, caught "if at all". The spec is corrected to the
  design as built, which is §1.1's: an honest agent may change the gate
  logic, with no approval on it, and the change is judged by merge review
  on every PR and held back by the human-gated release. Only a deliberate
  weakening is in the seam. §3 is amended in place.
- **2026-11-24: `secrets: inherit` and by-name keys, settled.** The
  earlier entry's open question on a by-name map is answered: keys passed
  by name reach a called workflow's environment-declaring jobs, across
  orgs and within a repo (logical-minds-foundry/mq-gateway-replay-lab run
  45390389192, green with the org keys hidden), and `inherit` works within
  one org (vergil-project/vergil-fixture-python-lib run 45306218449). The
  vergil-project/packages#6 premise, that environment secrets reach
  another repo's reusable workflow only through `inherit`, is superseded.
  Spec §4.2 is corrected in place to the routes as built: `inherit` within
  vergil-project to the allow-listed vergil-actions workflows, and the App
  keys by name across orgs and to a repo's own local workflow. Task 3c's
  rationale carries an amendment note.
- **2026-11-24: the user App's `workflows: write` outside the epic's
  orgs.** Task 3g's grant was accepted on six installations, not only the
  three in-scope orgs. The other three (diogenes-project,
  mq-rest-admin-project and vergils-nemesis) still keep App-key secrets at
  org level, readable from any branch. By the human's ruling N0.13, those
  orgs are inactive or work in progress, and the risk is accepted until
  they are renovated.
- **2026-11-24: the audit job does not wait on a green config job.** By
  the human's ruling N0.14, Task 13's `agent-audit` job runs after the
  config job with `if: ${{ !cancelled() }}`, so a failing config job does
  not skip the audit. Task 13 carries an amendment note.
- **2026-11-24: Task 13's vergil-project callers are filed ad hoc.** The
  epic reached GitHub's 100-sub-issue limit, so by the human's ruling
  N0.15 the 14 vergil-project Task 13 callers were filed under each repo's
  ad-hoc epic, each naming Task 13: vergil-project/docs#182,
  vergil-project/packages#67, vergil-project/vergil-actions#1414,
  vergil-project/vergil-claude-plugin#1333,
  vergil-project/vergil-containers#1025,
  vergil-project/vergil-fixture-python-app#77,
  vergil-project/vergil-fixture-python-lib#84,
  vergil-project/vergil-fixture-rust-app#92,
  vergil-project/vergil-fixture-rust-lib#98,
  vergil-project/vergil-fixture-ts-app#94,
  vergil-project/vergil-fixture-ts-lib#93, vergil-project/vergil-python#102,
  vergil-project/vergil-tooling#5126 and vergil-project/vergil-vm#454. The
  audit, the rollup and the retrospective do not see them as this epic's
  tasks; vergil-project/vergil-tooling#3840 tracks the limit. The
  cross-org callers are filed under the proxies as usual.
- **2026-11-24: a call job passes the App keys by name to its own repo's
  workflow.** Answering epic vergil-project/.github#1494's question, by the
  human's ruling N0.16 a call job passes `APP_CLIENT_ID` and
  `APP_PRIVATE_KEY` by name to its repo's own local reusable workflow,
  whose receiving jobs declare `environment: automation`.
  `INHERIT_ALLOWED` is not widened. vergil-project/vergil-containers#1030
  corrects vergil-containers' test to allow exactly that map.
- **2026-11-24: Task 17d's "agent submit fails at GitHub" check is
  waived.** The human waived that acceptance check of Task 17d
  (vergil-project/vergil-tooling#4788), as covered by Task 18d. Task 17d
  carries an amendment note.
