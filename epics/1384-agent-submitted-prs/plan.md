# Agents submit and finalize their own PRs into develop — implementation plan

- **Epic:** vergil-project/.github#1384
- **Planning issue:** vergil-project/.github#1383 (review history is in its
  comments)
- **Status:** draft
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
- Protected paths: `.github/workflows/**` and `vergil.toml` in every repo,
  the registry file itself, plus each repo's registry entries
  (vergil-actions `actions/**`, `.github` `docs/logbook/**`). Nothing else
  starts in the core.
- A protected-change approval or merge-review exemption is a `vrg-ruling`
  exchange: the request by the agent login, the reply by the human login,
  neither edited or deleted, and the reply the first trusted comment after
  the request. The reply carries exactly one decision line, `approve` or
  `refuse`. Anything else is not approved.
- An approval covers a protected file by its git blob hash; a deleted file
  is recorded as `deleted`.
- The change fingerprint is `git diff -U0 <base>...<head> | git patch-id
  --stable`. For one commit of a batch, the commit's diff against its parent
  is used.
- Every check fails closed: an unreadable registry, sign-off, record or API
  response refuses. Nothing passes silently.
- `workflows: write` is granted only after the environment-scoped secrets
  are in place and verified. `pull_requests: write` is granted only after
  the `main` ruleset is converged and the daily audit runs green fleet-wide
  with its marker set.

## Review focus

Inputs the spec implies that no single happy-path test covers. Each line's
test is added to the task named.

1. **A registry file that is malformed, or unreadable at the base or the
   head** → submit refuses, naming the file and the ref, and never treats it
   as an empty registry (Task 4:
   `test_malformed_registry_refuses_not_empty`).
2. **A protected file renamed or deleted by the PR** → both the old and the
   new path count as changed; a deletion is covered only by an approval that
   records `deleted` (Task 4:
   `test_rename_counts_both_paths_and_delete_needs_deleted_blob`).
3. **A reply that is edited after posting, posted by the wrong login, or
   followed by a second human reply** → the edited or wrong-login reply is
   not approved; the second reply is ignored (Task 6b:
   `test_past_approval_rejects_edited_and_wrong_login`).
4. **A rebase over a `develop` edit next to the branch's hunks** → the
   fingerprint is unchanged; a conflict resolution that changes the branch's
   own lines changes it (Task 5:
   `test_fingerprint_survives_nearby_develop_edit`).
5. **A GitHub read failure during the watch's re-check** → auto-merge is
   turned off and the PR is reported failed with the read error, never left
   armed (Task 11: `test_recheck_read_failure_disarms`).
6. **A PR whose task issue cannot be resolved from its body or ready-state**
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
| 2 | `main` ruleset restricting updates | T | task | — |
| 3a | Reusable workflows read App secrets from an environment | A | task | — |
| 3b | Repo config converges the secrets environment | T | task | — |
| 3c | Callers that mint the token declare the environment | each | task | 3a |
| 4 | Protected-path set and match | T | task | — |
| 5 | Change fingerprint | T | task | — |
| 6a | Two request kinds and the structured reply | T | task | — |
| 6b | Past-exchange verifier and lookup | T | task | 6a |
| 7 | Merge-review record | T | task | 5 |
| 8 | Shared gate evaluator | T | task | 4, 6b, 7 |
| 9 | Submit: agents allowed, gates before push | T | task | 8 |
| 10 | Provenance: own-PR rules | T | task | — |
| 11 | Watch re-check; finalize refuses `--release` for agents | T | task | 8 |
| 12 | Daily audit: `vrg-agent-audit` | T | task | 8 |
| 13 | Registries in vergil-actions and `.github` | A, G | task | 4 |
| 14 | Plugin hooks' wording | P | task | — |
| 15a | `merge-review`: record, approvals, exemption | P | task | 7, 6a |
| 15b | The implementer owns the PR to merge | P | task | 15a |
| 15c | Epic and fixture skills submit | P | task | 15b |
| 15d | `hoshin` on the protected-change approval | P | task | 15a, 13 |
| 16a | Consumer CLAUDE.md and permission spec | T | task | 9 |
| 2d | Release vergil-tooling; apply the `main` ruleset | T | deploy | 2 |
| 17d | Release tooling, plugin | T, P | deploy | 2d, 3b, 9–12, 14, 15c–d, 16a |
| 12d | Audit live fleet-wide, marker set, green | G | deploy | 17d, 13 |
| 18d | Grant the user App `pull_requests: write` | — | deploy | 2d, 12d |
| 16b | `.github` CLAUDE.md on the new boundary | G | task | 18d |
| 3d | Move secrets; grant `workflows: write` | — | deploy | 3a, 3c, 17d |
| 19 | Fixture runs (switch on, agent cycle) | F | fixture | 18d, 3d |
| 19v | Protected path, exemption and audit on one fixture | F | valid | 19 |
| 20 | Switch on for the rest of the fleet (one per repo) | each | task | 19v |
| R1 | Documentation review (#4751) | T | docs | 20 |
| R2 | Retrospective (#1386) | G | retro | R1 |

Tasks 2, 3a, 3b, 4, 5, 6a, 10 and 14 have no blockers and can be worked
side by side. Tasks 2 and 3a–3d are the standalone hardening the spec puts
first (§5 step 0). They land even if the rest stalls.

**At filing:**

- vergil-project/vergil-actions#1185 is absorbed. Close it as planned under
  this epic, and hand its advisory to Task 3d.
- The twelve advisories that track the planning issue are handed on as
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

**Ordering rule:** until Task 18d the user App has no `pull_requests:
write`, so every PR in this epic before then is submitted by the human, as
today. A PR touching `.github/workflows/**` is pushed by the human's
`vrg-submit-pr` until Task 3d.

---

## Task 2: `main` ruleset restricting updates

**Files:**

- Modify: `src/vergil_tooling/lib/github_config.py`
  (`desired_branch_protection_ruleset`, `compute_desired_state`)
- Test: `tests/vergil_tooling/test_github_config_lib.py`

**Interfaces:**

- Produces: `desired_main_update_ruleset(automation_app_id: int | None) ->
  DesiredRuleset`, named `"Main updates"`, `ref_include=["refs/heads/main"]`,
  one rule `{"type": "update"}`, bypass actors the admin role and, when
  `automation_app_id` is set, that App (`actor_type="Integration"`).

- [ ] **Step 1: Confirm the facts.** Record in the PR body:
  - which identity merges a release PR into `main` today (read
    `lib/release/` and a recent release PR's timeline);
  - that the `update` rule is available on the org's plan and refuses a PR
    merge into `main` by a non-bypass identity (GitHub docs, plus a dry run
    on a fixture).

  If the rule cannot do this, stop and ask the human. The fallback is the
  spec's §8 item.
- [ ] **Step 2: Write the failing tests.**

  ```python
  def test_main_update_ruleset_restricts_updates():
      rs = desired_main_update_ruleset(automation_app_id=None)
      assert rs.ref_include == ["refs/heads/main"]
      assert [r["type"] for r in rs.rules] == ["update"]

  def test_main_update_bypass_is_admin_and_optional_app():
      ids = lambda rs: {(a["actor_type"], a["actor_id"]) for a in rs.bypass_actors}
      assert ids(desired_main_update_ruleset(None)) == {("RepositoryRole", 5)}
      assert ids(desired_main_update_ruleset(123)) == {
          ("RepositoryRole", 5), ("Integration", 123)}
  ```

- [ ] **Step 3: Run them:** `vrg-container-run -- pytest
  tests/vergil_tooling/test_github_config_lib.py -k main_update` → FAIL.
- [ ] **Step 4: Implement.** Add the ruleset to `compute_desired_state`.
  Pass the App ID only if Step 1 found the release merging as the App. Leave
  "Branch protection" unchanged.
- [ ] **Step 5:** Run the tests again → PASS. Then `vrg-container-run --
  vrg-validate`.
- [ ] **Step 6: Commit** `feat(github-config): restrict updates to main`.

## Task 2d (deployment): Release vergil-tooling; apply the `main` ruleset

- **Preconditions:** Task 2 merged. The release is human-gated.
- **Procedure:** the daily config job (`ops.yml`, `vrg-github-repo-config
  update`) converges every repo. Run it once by hand after the release.
- **Acceptance:**
  - the config audit reports no drift on "Main updates" in any repo;
  - a test PR into `main` on a fixture, opened by the human and merged with
    the user App's token, is refused (run after Task 18d grants the
    permission; until then, record the refusal of the API call itself);
  - the next human release merges normally.

## Task 3a: Reusable workflows read App secrets from an environment

**Files (vergil-actions):**

- Modify: every reusable workflow job that reads `APP_CLIENT_ID` or
  `APP_PRIVATE_KEY`. Inventory them with `grep -rn
  "APP_CLIENT_ID\|APP_PRIVATE_KEY" .github/workflows actions`; at least
  `cd-release.yml` and the epic-sweep and rollup callers.
- Modify: `.github/workflows/README.md` (the environment and why).

- [ ] **Step 1: Inventory.** List every job in every in-scope org that reads
  a credential that can write or act. That means vergil-project,
  logical-minds-foundry and mnemosys-project, from the local clones. For
  each read-only token PR CI needs, give its reason. The Conan audit token
  is one; vergil-project/.github#1391 tracks its skip. Put the list in the
  PR body; it is the input for Task 3c and Task 3d.
- [ ] **Step 2:** Each such job declares `environment: automation`. Its
  steps are unchanged.
- [ ] **Step 3: The check.** A workflow test (or the smoke release) runs a
  job declaring the environment from a non-`develop` branch and asserts the
  App secret is empty there.
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

- **The repos:** those Task 3a's inventory finds minting an automation token
  in their own workflows (not through a vergil-actions reusable workflow).
- **Each task:**
  - the job declares `environment: automation`;
  - `vrg-container-run -- vrg-validate`;
  - commit `ci: read the automation App secrets from the automation
    environment`.

  The PR is human-pushed until Task 3d.

## Task 3d (deployment): Move the secrets; verify; grant `workflows: write`

- **Preconditions:** Tasks 3a and 3c closed, and vergil-actions released
  with 3a (human-gated). Task 17d is released, so the environment exists
  everywhere.
- **Procedure (the human):**
  1. Move `APP_CLIENT_ID` and `APP_PRIVATE_KEY` into each repo's
     `automation` environment.
  2. Remove them at org and repo level.
  3. Re-run a release, an epic sweep and a rollup.
  4. Check whether the org setting can cap the permissions a workflow may
     request for its own `GITHUB_TOKEN` below write, and if so cap it.
     Record the answer either way.
  5. Then grant the user App `workflows: write`.
- **Acceptance:**
  - #1185's acceptance criteria all hold: no workflow outside the
    environment can read the credentials, every minting job still works,
    and `configuration.md` describes the environment;
  - a workflow on a scratch branch that echoes whether the secret is set
    reports it unset;
  - an agent push of a workflow-touching branch is accepted.

## Task 4: Protected-path set and match

**Files:**

- Create: `src/vergil_tooling/lib/protected_paths.py`
- Test: `tests/vergil_tooling/test_protected_paths.py`

**Interfaces:**

- Produces:
  - `FLEET_DEFAULT: tuple[str, ...] = (".github/workflows/**",
    "vergil.toml", "protected-paths.toml")`;
  - `load_registry(repo: Path, ref: str) -> tuple[str, ...]`, which reads
    `protected-paths.toml` (`paths = [<glob>, …]`) at that ref; an absent
    file is `()`, and a malformed or unreadable one raises
    `ProtectedPathsError`;
  - `protected_set(repo: Path, ref: str) -> tuple[str, ...]`;
  - `@dataclass(frozen=True) class ProtectedChange: path: str; blob: str`
    (`"deleted"` for a deletion);
  - `changed_protected(repo: Path, base: str, head: str) ->
    list[ProtectedChange]`, which matches against the union of
    `protected_set` at the merge base and at the head, using `git diff
    --name-status -M <base>...<head>`.

- [ ] **Step 1: Write the failing tests:**
  - `test_workflow_and_vergil_toml_are_protected_by_default`;
  - `test_registry_removing_its_own_entry_is_judged_by_base`: the head's
    registry drops a path, the change to that path is still protected, and
    so is the registry;
  - `test_rename_counts_both_paths_and_delete_needs_deleted_blob`;
  - `test_malformed_registry_refuses_not_empty`;
  - `test_ordinary_file_is_not_protected`.
- [ ] **Step 2:** `vrg-container-run -- pytest
  tests/vergil_tooling/test_protected_paths.py` → FAIL.
- [ ] **Step 3: Implement** with `fnmatch`-style globs where `**` crosses
  directories. Blobs come from `git ls-tree <head> -- <path>`.
- [ ] **Step 4:** Run → PASS. Then `vrg-container-run -- vrg-validate`.
- [ ] **Step 5: Commit** `feat(protected-paths): the protected core and its
  match`.

## Task 5: Change fingerprint

**Files:**

- Create: `src/vergil_tooling/lib/change_fingerprint.py`
- Test: `tests/vergil_tooling/test_change_fingerprint.py`

**Interfaces:**

- Produces:
  - `change_fingerprint(repo: Path, base: str, head: str) -> str`, from `git
    diff -U0 <base>...<head> | git patch-id --stable`; an empty diff raises
    `FingerprintError`;
  - `commit_fingerprint(repo: Path, commit: str) -> str`, the same for
    `<commit>^..<commit>`.

- [ ] **Step 1: Write the failing tests** on a temp repo:
  - `test_fingerprint_survives_nearby_develop_edit`: edit a line two lines
    from the branch's hunk on base, rebase, and expect an equal fingerprint;
  - `test_conflict_resolution_changes_fingerprint`;
  - `test_fingerprint_ignores_line_shift`;
  - `test_empty_diff_raises`.
- [ ] **Step 2:** Run → FAIL. Implement. Run → PASS. Then `vrg-container-run
  -- vrg-validate`.
- [ ] **Step 3: Commit** `feat(fingerprint): a rebase-stable change
  fingerprint`.

## Task 6a: Two request kinds and the structured reply

**Files:**

- Create: `src/vergil_tooling/lib/rulings/approvals.py`
- Modify:
  - `src/vergil_tooling/lib/rulings/marker.py`: a `kind` value on the
    request marker, `protected-change` or `merge-review-exemption`; the
    existing requests carry none;
  - `src/vergil_tooling/bin/vrg_ruling_reply.py`: refuse to post a reply
    to either kind without exactly one line `Decision: approve` or
    `Decision: refuse`.
- Test: `tests/vergil_tooling/test_rulings_approvals.py`,
  `tests/vergil_tooling/test_vrg_ruling_reply.py`

**Interfaces:**

- Produces:
  - `RequestKind = Literal["protected-change", "merge-review-exemption"]`;
  - `render_protected_change_request(changes: list[ProtectedChange],
    diff_url: str) -> str`;
  - `render_exemption_request(reason: str, fingerprint: str) -> str`;
  - `parse_decision(body: str) -> Literal["approve", "refuse"]`, which raises
    `RulingError` on zero or several decision lines or on any other word;
  - CLI `vrg-ruling request --kind <kind> …`, which posts the request as the
    agent and prints the exact `vrg-ruling-reply` command for the human.

- [ ] **Step 1: Write the failing tests:**
  - `test_protected_change_request_lists_each_path_and_blob`;
  - `test_parse_decision_accepts_only_one_exact_line`, covering `approve`,
    `refuse`, `Approve`, two lines, and none;
  - `test_ruling_reply_refuses_missing_decision_for_new_kinds`;
  - `test_existing_requests_unchanged`.
- [ ] **Step 2:** Run → FAIL. Implement. Run → PASS. Then `vrg-container-run
  -- vrg-validate`.
- [ ] **Step 3: Commit** `feat(rulings): protected-change and exemption
  requests`.

## Task 6b: Past-exchange verifier and lookup

**Files:**

- Modify: `src/vergil_tooling/lib/rulings/approvals.py`
- Test: `tests/vergil_tooling/test_rulings_approvals.py`

**Interfaces:**

- Consumes: `exchange`'s comment stream and `identity.human_login()` /
  `agent_login()`.
- Produces:
  - `@dataclass(frozen=True) class Approval: kind: RequestKind;
    request_id: int; decision: str; blobs: dict[str, str]; fingerprint: str
    | None`;
  - `verified_approvals(issue_ref: str) -> list[Approval]`, which keeps a
    request only when the agent login posted it, the human login posted
    the reply, neither comment was edited or deleted, and the reply is the
    first trusted comment after the request. Comments after the reply are
    ignored. A read failure raises; it never returns `[]`.

- [ ] **Step 1: Write the failing tests:**
  - `test_past_approval_ignores_later_trusted_comments`;
  - `test_past_approval_rejects_edited_and_wrong_login`;
  - `test_refuse_decision_is_not_an_approval`;
  - `test_read_failure_raises`.
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
  - `records(issue_ref: str) -> list[Record]`, from comments by the agent
    login only;
  - CLI `vrg-merge-review-close --issue <ref> --outcome <outcome>
    --body-file <file>`, which computes the head and fingerprint itself,
    from the worktree's `HEAD` and merge base or, for a batch member, that
    member's commit. It then posts the closing comment with the eligibility
    line first and the record line second.

- [ ] **Step 1: Write the failing tests:**
  - `test_record_round_trip`;
  - `test_close_computes_fingerprint_not_caller` (the CLI takes no
    fingerprint argument);
  - `test_records_ignore_non_agent_authors`;
  - `test_incomplete_outcome_is_refused` (stopped, waiting, fatal and error
    are not outcomes).
- [ ] **Step 2:** Run → FAIL. Implement. Run → PASS. Then `vrg-container-run
  -- vrg-validate`.
- [ ] **Step 3: Commit** `feat(merge-review): a machine-readable closing
  record`.

## Task 8: Shared gate evaluator

**Files:**

- Create: `src/vergil_tooling/lib/agent_gates.py`
- Test: `tests/vergil_tooling/test_agent_gates.py`

**Interfaces:**

- Consumes: Tasks 4, 5, 6b and 7.
- Produces:
  - `@dataclass(frozen=True) class GateResult: ok: bool; reasons:
    tuple[str, ...]`;
  - `evaluate(repo: Path, *, base: str, head: str, issue_refs: list[str])
    -> GateResult`. It applies two rules:
    - **protected files:** each `ProtectedChange` blob must appear in an
      approved `protected-change` approval on any of `issue_refs`;
    - **merge review:** a single-issue PR needs a record or an approved
      exemption whose fingerprint equals `change_fingerprint(base, head)`.
      A batch needs, for each issue, a record matching that issue's commit
      by `commit_fingerprint`. A whole-PR record for `base..head` covers a
      fix pushed to an open batch.
  - Any lookup failure gives `ok=False` with the error as the reason.

- [ ] **Step 1: Write the failing tests:**
  - `test_not_eligible_record_passes`;
  - `test_missing_or_stale_record_fails`;
  - `test_batch_needs_each_issues_own_commit_record`;
  - `test_exemption_stands_in_for_record_but_not_for_approval`;
  - `test_unresolvable_task_issue_refuses`.
- [ ] **Step 2:** Run → FAIL. Implement. Run → PASS. Then `vrg-container-run
  -- vrg-validate`.
- [ ] **Step 3: Commit** `feat(gates): one evaluator for approvals and
  merge-review coverage`.

## Task 9: Submit: agents allowed, gates before push

**Files:**

- Modify: `src/vergil_tooling/bin/vrg_submit_pr.py`:
  - `main`'s identity gate at line 168: refuse AUDIT, and agents with
    `--release`; allow USER;
  - the per-PR checks beside `batch_rules.check_*` around line 1005:
    `agent_gates.evaluate` before any push or `gh pr create`;
  - the docstring and help text.
- Modify: `src/vergil_tooling/bin/vrg_gh.py` (`_DENIED_AGENT` messages
  point at `vrg-pr-workflow report-ready` and `vrg-submit-pr`; the denials
  stay) and `src/vergil_tooling/lib/github_config.py:158-163` (the comment).
- Modify: the agent-mode rollout switch, `[pr-workflow] agent-submit` in
  `vergil.toml`, default `false`. Submit refuses an agent in a repo where it
  is not `true`.
- Test: `tests/vergil_tooling/test_vrg_submit_pr.py`,
  `tests/vergil_tooling/test_vrg_gh.py`

- [ ] **Step 1: Write the failing tests:**
  - `test_user_agent_submits_when_switch_on`;
  - `test_user_agent_refused_when_switch_off`;
  - `test_agent_release_refused`;
  - `test_audit_refused`;
  - `test_gate_failure_leaves_no_pr_and_pushes_nothing`: the stubbed `gh
    pr create` and push are never called;
  - `test_rerun_after_approval_submits`;
  - `test_human_submit_runs_same_gates`: there is no human override. The
    human's way past merge review is the exemption (spec §4.6);
  - `test_vrg_gh_agent_denial_names_submit`.
- [ ] **Step 2:** Run → FAIL. Implement. Run → PASS. Then `vrg-container-run
  -- vrg-validate`.
- [ ] **Step 3: Commit** `feat(submit-pr): agents submit through the gates`.

## Task 10: Provenance: own-PR rules

**Files:**

- Modify: `src/vergil_tooling/lib/pr_provenance.py` (`_FORBIDDEN` at
  69-76, `evaluate`)
- Test: `tests/vergil_tooling/test_pr_provenance.py`

- [ ] **Step 1: Write the failing tests:**
  - `test_user_creates_and_automerges_own_pr`;
  - `test_user_merge_of_own_pr_allowed` (the `merged_now` path);
  - `test_user_action_on_human_pr_forbidden`;
  - `test_user_approval_forbidden_even_on_own_pr`;
  - `test_audit_rules_unchanged`.
- [ ] **Step 2: Decide `edited`.** Read whether submit edits a PR after
  creating it, and allow `edited` on the agent's own PR only if it does.
  State the finding in the PR body.
- [ ] **Step 3:** Run → FAIL. Implement: USER's forbidden set applies to a
  PR whose author is not the user login, and `approved` stays forbidden
  everywhere. Run → PASS. Then `vrg-container-run -- vrg-validate`.
- [ ] **Step 4: Commit** `feat(provenance): the user agent may act on its own
  PR`.

## Task 11: Watch re-check; finalize refuses `--release` for agents

**Files:**

- Modify: `src/vergil_tooling/lib/finalize_watch.py`:
  `_check_provenance` at 486 becomes `_check_gates`, which runs provenance
  and `agent_gates.evaluate` at watch start and on each new head; failures
  go through `_disable_auto_merge`.
- Modify: `src/vergil_tooling/bin/vrg_finalize_pr.py` (an early refusal of
  `--release` for agents, before any watch).
- Test: `tests/vergil_tooling/test_finalize_watch.py`,
  `tests/vergil_tooling/test_vrg_finalize_pr.py`

- [ ] **Step 1: Confirm re-arming.** Read how a PR's auto-merge behaves
  after a push today, and how a fixed PR is re-armed. Put the gates on that
  path, and record it in the PR body. Also confirm from `vergil-vm` that no
  agent VM shares a clone with the host over a mount, since the checkout
  lock is per clone. If one does, stop and ask the human.
- [ ] **Step 2: Write the failing tests:**
  - `test_new_head_touching_workflow_without_approval_disarms`;
  - `test_clean_rebase_keeps_auto_merge`;
  - `test_recheck_read_failure_disarms`;
  - `test_agent_release_refused_before_watch`.
- [ ] **Step 3:** Run → FAIL. Implement. Run → PASS. Then `vrg-container-run
  -- vrg-validate`.
- [ ] **Step 4: Commit** `feat(finalize): re-check the gates on every new
  head`.

## Task 12: Daily audit: `vrg-agent-audit`

**Files:**

- Create: `src/vergil_tooling/lib/agent_audit.py`,
  `src/vergil_tooling/bin/vrg_agent_audit.py` (entry point
  `vrg-agent-audit`)
- Test: `tests/vergil_tooling/test_agent_audit.py`

**Interfaces:**

- Consumes: `agent_gates.evaluate`, `pr_provenance.classify_login`, the
  trusted-authors configuration passed by `--trusted-authors <file>`.
- Produces: `vrg-agent-audit --repo <owner/name> [--init]`.
  - The marker is the ref `refs/vergil/agent-audit/last` in the audited
    repo, holding the last audited merge commit. `--init` sets it to the
    current `develop` tip and audits nothing.
  - It exits 0 when clean, 1 on any finding or read failure, and 2 on a
    usage error.
- Checks, per merge since the marker:
  1. protected files covered by approval (every PR);
  2. a merge-review record or exemption covering the merged head's change
     (agent-involved PRs only: opened by an agent login, or with any
     action attributed to one);
  3. the merge or auto-merge actor is in the expected set: the human, the
     user agent, or the automation App on PRs it authored;
  4. any merge into `main` by anyone but the human, or the automation App
     where Task 2 found it merges releases.

  A merged blob that differs from the head blob is reported as a separate
  `merge-changed-approved-file` condition.

- [ ] **Step 1: Confirm the pusher record.** Find which GitHub API records
  name the identity that pushed a commit to a PR branch, and use one if
  any is reliable. State the answer in the PR body.
- [ ] **Step 2: Write the failing tests:**
  - `test_init_sets_marker_and_audits_nothing`;
  - `test_missed_run_is_caught_up_from_marker`;
  - `test_automation_only_pr_skips_record_check`;
  - `test_unexpected_actor_reported`;
  - `test_main_merge_by_agent_reported`;
  - `test_squash_changed_approved_file_is_its_own_condition`;
  - `test_read_failure_exits_1`.
- [ ] **Step 3:** Run → FAIL. Implement. Run → PASS. Then `vrg-container-run
  -- vrg-validate`.
- [ ] **Step 4: Commit** `feat(audit): a daily audit of agent merges`.

## Task 13: Registries in vergil-actions and `.github`

- **vergil-actions:** `protected-paths.toml` with `paths = ["actions/**"]`.
- **.github:** `protected-paths.toml` with `paths = ["docs/logbook/**"]`,
  plus the audit job in `.github/workflows/ops.yml`. The job runs
  `vrg-agent-audit` for each repo in the org as the automation App, in the
  `automation` environment, after the config job, and opens or updates one
  issue on failure.
- Each is a separate PR in its own repo, filed as 13 (A) and 13 (G). Each
  runs `vrg-container-run -- vrg-validate` and commits `feat: protected-path
  registry` / `feat(ops): the daily agent audit`.
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

- Modify: `hooks/scripts/block-agent-merge.sh:54`,
  `hooks/scripts/block-github-contents-api.sh:6,69`,
  `hooks/scripts/block-raw-gh-pr-create.sh:39-42`. The blocks stay; the
  text says agents merge only through `vrg-submit-pr`'s auto-merge.
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
  chains finalize), then the watch procedure until a merge or a human stop.
- The watch procedure lives once, in `pr-watch`. `issue-implement` invokes
  it; `pr-watch` stays the standalone entry for an open PR.
- **`pr-watch` changes:**
  - it reruns `vrg-finalize-pr` itself after a fix;
  - it pushes a `not-eligible` fix once every ruling is answered;
  - it never runs anything with `--release`.
- **Removed:** "Only the human opens the PR" and the workflow-push special
  case. The cloud relay section stays.
- **Check:** the plugin's tests pass under `vrg-container-run --
  vrg-validate`. Commit `feat(skills): the implementer owns the PR to
  merge`.

## Task 15c: `epic-implement`, `epic-create`, `epic-retrospective`, `fixture-run`

- **`epic-implement`:** after a merge, re-read the epic and continue with
  the next runnable task. Drop "Your responsibility ends here" and the
  retrospective-PR line.
- **The other three:** the agent submits its docs or fixture PRs after
  merge review. Fixture releases stay human-gated.
- **Check:** plugin tests pass under `vrg-container-run -- vrg-validate`.
  Commit `feat(skills): agents submit epic and fixture PRs`.

## Task 15d: `hoshin` on the protected-change approval

- The logbook PR gets merge review.
- The human's in-session approval of the full text is posted as the
  protected-change approval for the `docs/logbook/**` files that Task 13
  protects.
- Drop "the human submitting the PR is the review's sign-off".
- **Check:** plugin tests pass. Commit `feat(hoshin): the logbook sign-off is
  a protected-change approval`.

## Task 16a: Consumer CLAUDE.md template and the permission-model spec

- **Files (vergil-tooling):**
  - `src/vergil_tooling/data/claude_md_consumer.md:29-42`;
  - `docs/specs/2026-05-29-agent-permission-model-design.md` (the "human
    triggers" principle and the user App table).
- **The new rule text:**
  - agents submit and finalize through the tools, never with `--release`;
  - releases and promotion are human actions;
  - a protected-path change needs the human's approval;
  - the freeze runs from report-ready to the agent's own submit.

  The workflow-push paragraph goes.
- **Check:** `vrg-container-run -- vrg-validate`. Commit `docs: the agent
  boundary moves to releases`.

## Task 17d (deployment): Release vergil-tooling and the plugin

- **Preconditions:** the tasks in its "Blocked by" are closed; the releases
  are human-gated.
- **Procedure:** release vergil-tooling, then the plugin; run `vrg-vm update
  --all` and `vrg-host plugins`.
- **Acceptance:**
  - in a VM, `vrg-submit-pr --help` no longer says "human-only";
  - in a repo with the switch off, an agent submit is refused with the
    switch named.

## Task 12d (deployment): Audit live fleet-wide, marker set, green

- **Preconditions:** Task 17d released; Task 13 (G) merged.
- **Procedure:** run `vrg-agent-audit --init` for every repo, then let one
  scheduled run pass.
- **Acceptance:**
  - every repo has a marker;
  - the scheduled run exits 0;
  - a deliberate dry run against a known human-merged PR reports nothing.

## Task 18d (deployment): Grant the user App `pull_requests: write`

- **Preconditions:** Tasks 2d and 12d closed.
- **Procedure (the human):** change the App's permission and accept it on
  every installation.
- **Acceptance:**
  - an agent `vrg-submit-pr` in a repo with the switch off is still
    refused;
  - Task 2d's deferred check (a merge into `main` with the agent's token is
    refused) passes.

## Task 16b: `.github` CLAUDE.md on the new boundary

The same rule text as Task 16a, in `.github/CLAUDE.md:34-47`. It is filed
after 18d so the steering never runs ahead of the permission. Commit `docs:
the agent boundary moves to releases`.

## Task 19: Fixture runs

File with `fixture-run`, after Tasks 18d and 3d exist:

```text
fixture-run vergil-project/.github#1384 \
  python-app,python-lib,rust-app,rust-lib,ts-app,ts-lib \
  --validates "agent-submitted PRs (the Task 17d releases)" \
  --after vergil-project/.github#<18d> --after vergil-project/.github#<3d>
```

Each run's change task turns `[pr-workflow] agent-submit = true` on. That is
a `vergil.toml` change, so it needs the human's protected-change approval,
which is its own first proof. The agent then submits the change. The
release stays the human's.

## Task 19v (validation): Protected path, exemption and audit on one fixture

- **Preconditions:** Task 19's runs closed on `vergil-fixture-python-lib`.
- **Procedure:**
  1. An agent PR editing a workflow is refused, then approved, then merges.
  2. An agent PR with the human's merge-review exemption merges.
  3. The human merges one PR with no approval, deliberately, through the
     API.
- **Acceptance:** cases 1 and 2 merge; the next audit run reports case 3 by
  name, and nothing else.

## Task 20 (one per repo): Switch on for the rest of the fleet

Each repo's PR sets `agent-submit = true`, with the human's protected-change
approval. That repo's registry, if it needs one, lands first in the same PR.
Each runs `vrg-container-run -- vrg-validate`. The switch stays after the
rollout, as a per-repo opt-out (spec §5 step 4).

## Bookends

- **Task 1 (#1385):** this spec and plan.
- **R1 (vergil-project/vergil-tooling#4751):** documentation review. It
  covers the site docs on the new boundary, the approval and exemption
  commands, the audit, and the rollout switch, and spawns per-repo doc tasks
  as needed.
- **R2 (#1386):** retrospective, with `epic-retrospective`.
