# Remove private-repo support: implementation plan

- **Epic:** vergil-project/.github#1476
- **Planning issue:** vergil-project/.github#1474 (review history is in its
  comments)
- **Status:** draft, in design review

> **For agentic workers:** each task below is filed as its own GitHub issue
> in the repo its PR lands in, and run with `issue-implement`, or with
> `issue-deploy` for a deployment task. Steps use checkbox (`- [ ]`)
> syntax for tracking.

**Goal:** remove the last two private repos from the fleet, then remove
every private-repo special case from vergil-tooling, vergil-actions and
vergil-claude-plugin, so that every epic and queue homes in `<org>/.github`
and nothing branches on repo visibility.

**Architecture:** deletion only, written to one invariant that the
operational tasks establish: every non-archived repo is public. No runtime
guard or audit is added. The tasks run in this order:

1. Operational tasks remove the private repos.
2. The callers are cleaned up and vergil-actions is changed, in an order
   that never breaks CI.
3. vergil-tooling and the skills are trimmed, one subsystem per PR, so that
   each removal can be backed out and deployed on its own.

**Tech stack:** Python (vergil-tooling), reusable workflows and composite
actions (vergil-actions), skill Markdown (vergil-claude-plugin), MkDocs
sites.

**Spec:** `epics/1476-remove-private-repos/spec.md`.

## Global constraints

- The six organizations: `vergil-project`, `logical-minds-foundry`,
  `mnemosys-project`, `diogenes-project`, `mq-rest-admin-project` and
  `vergils-nemesis`. Archived repos are never touched.
- No runtime guard, audit check, deprecation period or shim. The one
  rejection added is the `[project].ghas` retirement (Task 11).
- The retirement reason, verbatim: `[project].ghas was removed in epic
  vergil-project/.github#1476: every repo is public`.
- History is never edited: `docs/specs/`, `docs/plans/`, `docs/superpowers/`,
  `docs/reports/`, `paad/`, `releases/`, `CHANGELOG.md` and `epics/`.
- Untouched uses of "private": App private keys, private security
  advisories and vulnerability reporting, the merge-review state directory,
  `"private": true` in `package.json`, and CMake `PRIVATE`.
- The leftover search for a repo is the spec's §5 grep, run from its root:

  ```bash
  P='\bself[-_]home(d)?\b|\bprivate repos?\b|\bprivate home\b'
  P="$P|upload-sarif|ghas_available|\[project\]\.ghas|visibility"
  grep -rniE "$P" . \
    --exclude-dir=.git --exclude-dir=.worktrees --exclude-dir=build \
    --exclude-dir=specs --exclude-dir=plans --exclude-dir=superpowers \
    --exclude-dir=reports --exclude-dir=paad --exclude-dir=releases \
    --exclude=CHANGELOG.md
  ```

  Every hit is either removed or read and confirmed as one of the allowed
  uses in spec §5. These include:
  - the `codeql-action/upload-sarif` action;
  - `vrg-gh`'s allow-listed `--visibility` flag;
  - `visibility` in a sense other than repository visibility;
  - the `ghas` retirement.

  Each task runs it over the files it touched. Task 14 runs it over the
  whole of vergil-tooling.
- Validation is `vrg-container-run -- vrg-validate` only. vergil-tooling
  keeps 100% branch coverage.
- **Pushes for workflow changes.** The agent cannot push a task that touches
  `.github/workflows/**`: Tasks 5, 7a–7z, and 11 if its template tests fall
  under that path. report-ready records the task, and the human's
  `vrg-submit-pr` pushes it.
- **Repo-level settings follow visibility, through the nightly config
  update.** No task changes a repo's settings by hand. Every repo that is
  public today keeps the settings it has. The lab repo is the exception:
  the first nightly `vrg-github-repo-config update` after its flip brings it
  up to the public-repo settings, which means secret scanning on and the
  CodeQL and GHAS checks required. Task 3 closes the window in which its
  CI still skips CodeQL.

## Review focus

1. **CI broken by the wrong removal order.**
   - Dropping an input that the reusable workflow still declares is
     harmless. Passing an input it no longer declares fails.
   - Dropping a permission grant that it still requests fails at startup.

   Expected: Tasks 3 and 4 merge before Task 5, and Task 7 runs after Task
   6's release. Task 5 step 4 checks every non-archived caller before it
   asks for the release.
2. **The CodeQL job wrongly skipped or wrongly required.** Expected: with
   `upload-sarif` gone, the CodeQL job runs whenever `run-security` and
   `run-codeql` are true. The evidence gate still accepts a `skipped` CodeQL
   result when `run-codeql` is false, and only then. Task 5 step 2 pins
   both.
3. **A `vergil.toml` that still sets `ghas` loads without error.** Expected:
   it fails with the retirement reason. Task 11 step 1 pins it.
4. **The Dependabot audit printing per-alert detail.** Expected: the single
   report is counts by severity plus a Security-tab link, for every repo.
   Task 13 step 1 pins it with an alert carrying a package name that must
   not appear in the output.
5. **An org with no `.github`.** Expected: resolving a home still fails with
   the existing #3643 message, which names the repo to create. It must not
   fail with a visibility error or a silent self-home. Task 8 step 1 pins
   it.

---

## Delivery table

Every task is linked under epic vergil-project/.github#1476. The exception
is the tasks in `logical-minds-foundry`, which go under that org's proxy
(`vrg-epic-create --proxy-for vergil-project/.github#1476 --repo
logical-minds-foundry/.github`) and are named by full ref.

Repos are abbreviated: `lmf/` is `logical-minds-foundry/`, `vp/` is
`vergil-project/`.

| Task | Repo | Kind | Blocked by |
| --- | --- | --- | --- |
| 1 Publish the lab repo | `lmf/.github` | deployment | 2 |
| 2 Archive the gateway | `lmf/.github` | deployment | none |
| 3 Lab caller cleanup, pass 1 | `lmf/mq-gateway-replay-lab` | impl | the flip |
| 4 Pass 1, resiliency lab | `lmf/mq-resiliency-lab-for-linux` | impl | none |
| 5 Drop private-repo SARIF paths | `vp/vergil-actions` | impl | 2, 3, 4 |
| 6 Release vergil-actions | `vp/vergil-actions` | deployment | 5 |
| FR Fixture runs, all six | the fixture repos | fixture-run | 6 |
| 7a–7z Caller cleanup, pass 2 | each caller repo | impl | 6 |
| 8 Epic home, queue placement | `vp/vergil-tooling` | impl | 1, 2 |
| 9 Epic-audit visibility passes | `vp/vergil-tooling` | impl | 8 |
| 10 Token-coverage guard | `vp/vergil-tooling` | impl | 9 |
| 11 GHAS inference, scaffolding | `vp/vergil-tooling` | impl | 1–4, 6 |
| 12 Routing, linking, proxies | `vp/vergil-tooling` | impl | 10 |
| 13 Dependabot reporting | `vp/vergil-tooling` | impl | 1, 2 |
| 14 Visibility primitives | `vp/vergil-tooling` | impl | 8–13 |
| 15 Release vergil-tooling | `vp/vergil-tooling` | deployment | 14 |
| 16 hoshin skill | `vp/vergil-claude-plugin` | impl | 1, 2 |
| 17 Epic skills | `vp/vergil-claude-plugin` | impl | 15 |
| Doc review | `vp/vergil-tooling#4893` | bookend | all above |
| Retrospective | `vp/.github#1478` | bookend | doc review |

Task 7's rows are filed one per repo from the list in Task 7. Once filed, a
task in `logical-minds-foundry` is named by its full ref in every
`Blocked-by:`. Security advisory GHSA-2v43-fjxg-fv3c, which tracks the
planning issue, is retargeted to Task 13 when the tasks are filed.

Two rows need a note:

- **Task 3 waits on the flip, not on a closed issue.** It has no
  `Blocked-by:`. It is filed, under the proxy, as soon as Task 1 step 1 has
  confirmed the flip, and run straight away (see the Global constraints).
- **Tasks 8, 9 and 10 are chained** because they edit the same files:
  `lib/epic_audit.py`, `bin/vrg_epic_audit.py` and `bin/vrg_adhoc_epic.py`.
  PRs validated separately could still break develop once both merge.
- **Task 12 is chained after Task 10.** Deleting `PrivateHomeError` also
  edits its handlers in `bin/vrg_epic_audit.py` and `bin/vrg_queue.py`.
  Tasks 11 and 13 touch none of those files, and stay parallel.

---

### Task 1: Publish logical-minds-foundry/mq-gateway-replay-lab

Repo: `logical-minds-foundry/.github`, kind `deployment`, under the proxy.
Run with `issue-deploy`.

**Precondition (human-attested):** the human has made the repo public. The
human's declaration that its content is public (spec §1.1) stands, and no
scan or review precedes the flip.

- [ ] **Step 1: Confirm the flip.** Run `vrg-gh repo view
  logical-minds-foundry/mq-gateway-replay-lab --json visibility,isArchived`.
  Expected: `"visibility":"PUBLIC"`, `"isArchived":false`. If it prints
  anything else, comment "blocked: preconditions not met" and stop.
- [ ] **Step 2: Move the queue items.** The org's nightly epic sweep
  (`logical-minds-foundry/.github`'s `epic-sweep.yml`) moves a now-public
  repo's queue items on its own, so check first.
  - If #322 and #329 are already closed with the sweep's `Repo is now
    public; queue items moved to …` comment, record the sweep's moves and
    go on.
  - Otherwise, move each open child of #322 (intake) and #329 (deferred)
    under the `logical-minds-foundry/.github` epic of the same kind with
    `vrg-epic-move`. Then close #322 and #329 with `Repo is now public;
    queue items moved to <target ref>.`

  The agent does not run `vrg-epic-audit --close` itself.
- [ ] **Step 3: Move the ad-hoc work.** Run `vrg-adhoc-epic ensure --repo
  logical-minds-foundry/mq-gateway-replay-lab` and confirm it reports
  `Epic (ad hoc): mq-gateway-replay-lab` in `logical-minds-foundry/.github`.
  Move each open child of #71 under that epic with `vrg-epic-move`, then
  close #71 and #319, each with a pointer to the new epic.
- [ ] **Step 4: Close #307** (`secret-scan history + make public`) as
  completed, with `Superseded by <this task ref>; the repo is public.`
- [ ] **Step 5: Verify.** `vrg-gh issue list --repo
  logical-minds-foundry/mq-gateway-replay-lab --state open --label epic`
  prints nothing. Each moved item appears under its new parent in
  `logical-minds-foundry/.github`.
- [ ] **Step 6: Record** the outcome, with the moved and closed refs. No
  fleet-wide visibility check follows. With the gateway archived (Task 2)
  and the lab public, the human's statement that no other private repo
  exists establishes the invariant.

### Task 2: Archive logical-minds-foundry/mq-protocol-gateway

Repo: `logical-minds-foundry/.github`, kind `deployment`, under the proxy.
Run with `issue-deploy`.

- [ ] **Step 1: Close the open issues.** Close every open issue in
  `logical-minds-foundry/mq-protocol-gateway` as not planned, with
  `Repo archived by vergil-project/.github#1476.` As of 2026-11-22 these are
  #36, #35 (the intake epic), #29 (the ad-hoc archive), #25 and #8 (the
  ad-hoc epic). Close children before their epics. Re-list the open issues
  first: the set may have changed.
- [ ] **Step 2: Hand off to the human.** Comment that the repo is ready to
  archive and stop.
- [ ] **Step 3: Precondition (human-attested): the human has archived it.**
  Run `vrg-gh repo view logical-minds-foundry/mq-protocol-gateway --json
  isArchived`. Expected: `"isArchived":true`. The repo stays private.
- [ ] **Step 4: Record** the outcome.

### Task 3: mq-gateway-replay-lab: caller cleanup, pass 1

Repo: `logical-minds-foundry/mq-gateway-replay-lab`. It is filed under the
proxy as soon as Task 1 step 1 confirms the flip, since a proxy refuses a
private target, and it is run straight away. Until it merges, the nightly
config update requires a CodeQL check that the lab's CI skips.

**Files:** modify `.github/workflows/ci.yml` (the `upload-sarif: false`
line in the `security:` job's `with:`), and `vergil.toml` if it sets
`ghas`.

- [ ] **Step 1:** Delete `upload-sarif: false`. The input defaults to
  `true`, so CodeQL now runs and SARIF uploads, which is correct for a
  public repo.
- [ ] **Step 2:** `vrg-container-run -- vrg-validate` passes. After the PR's
  CI runs, its CodeQL, Trivy and Semgrep jobs all report `success`.
- [ ] **Step 3:** Commit (`vrg-commit --type ci --scope security`). This
  touches `.github/workflows/**`, so the human pushes it.
- [ ] **Step 4: After the merge,** run `vrg-github-repo-config audit --repo
  logical-minds-foundry/mq-gateway-replay-lab`. Expected: no drift. The
  ruleset requires the CodeQL and GHAS checks, secret scanning is on, and
  a new PR's CodeQL check reports `success`. If it shows drift, the human
  runs `vrg-github-repo-config update` for the repo. Then re-run the audit.

### Task 4: mq-resiliency-lab-for-linux: caller cleanup, pass 1

Repo: `logical-minds-foundry/mq-resiliency-lab-for-linux`, under the proxy.

**Files:** modify `.github/workflows/ci.yml` (the `upload-sarif: true`
line in the `security:` job's `with:`, and the comment block above it) and
`vergil.toml` (the `ghas = true` line).

- [ ] **Step 1:** Delete both lines and the comment that explains them.
- [ ] **Step 2:** `vrg-container-run -- vrg-validate` passes, and the PR's
  security jobs report `success`.
- [ ] **Step 3:** Commit. The human pushes it.

### Task 5: vergil-actions: drop the private-repo SARIF paths

Repo: `vergil-project/vergil-actions`.

**Files:**

- Modify: `.github/workflows/ci-security.yml`:
  - the `upload-sarif` input;
  - the workflow-level permission comment and `actions: read`;
  - each job's `actions: read`;
  - the CodeQL job's `inputs.upload-sarif` condition;
  - the `upload-sarif:` values passed to the Trivy and Semgrep actions;
  - the evidence gate's "no GHAS" comment.
- Modify: `actions/ci/security/semgrep/action.yml` and
  `actions/shared/security/trivy/action.yml`: the `upload-sarif` input, and
  the steps that attach SARIF as an artifact when it is not `true`.
- Modify: the tests under `actions/ci/evidence/tests/` and any action
  harness test that passes `upload-sarif`.
- Modify: the docs pages `workflows/ci-security.md`,
  `ci-gates/security-scanning.md`, `actions/shared-security-trivy.md`,
  `ci-security-semgrep.md`, `ci-security-codeql.md` and `ops-epic-sweep.md`.
- Modify: this repo's own `.github/workflows/ci.yml`, at the `security:`
  job's `actions: read` grant and its comment, which is this repo's Task 7
  row done in place.

**Interfaces:** produces `ci-security.yml@v2.1` with no `upload-sarif`
input, and no `actions: read` request at the workflow level or in any job.
Tasks 7 and 11 rely on both.

- [ ] **Step 1: Remove every `actions: read`** from `ci-security.yml`: the
  workflow-level grant, its comment, and each job's grant.
  - `codeql-action/upload-sarif` needs it only on private repos.
  - The evidence job downloads only same-run artifacts with
    `actions/download-artifact`. Its docs say `actions: read` is needed
    only for downloads from another run or repo.
  - If the PR's own CI shows that a job fails without the grant, stop and
    report it. Keeping a grant is a spec change, not the implementer's
    call.
- [ ] **Step 2: Pin the gate.** In the evidence tests, assert two cases:
  - with `run-security: true` and `run-codeql: true`, the gate requires
    `needs.codeql.result == 'success'`;
  - with `run-codeql: false`, the gate accepts `skipped`.

  Then edit the gate's `if:` to match. Run the repo's action harness tests
  through `vrg-container-run`. Expected: pass.
- [ ] **Step 3: Delete the inputs, fallbacks and docs** listed above. Run
  the leftover search over the repo. Expected: the only hits are
  `codeql-action/upload-sarif` uses and historical paths.
- [ ] **Step 4: Check the callers.** For each of the six organizations,
  search every non-archived repo's `.github/workflows/` for `upload-sarif:`.
  Expected: no hits, once Tasks 3 and 4 have merged. If there is a hit, stop
  and report it.
- [ ] **Step 5:** `vrg-container-run -- vrg-validate` passes. Commit. The
  human pushes it.

### Task 6: Release vergil-actions

Repo: `vergil-project/vergil-actions`, kind `deployment`.

**Precondition (human-attested):** the human has cut the vergil-actions
release that moves the `v2.1` tag. The agent never cuts it.

- [ ] **Step 1:** Confirm that the release tag contains Task 5's merge:
  `vrg-git merge-base --is-ancestor <Task 5 merge sha> <v2.1 tag sha>`
  exits 0.
- [ ] **Step 2:** Confirm that `ci-security.yml` at `v2.1` has no
  `upload-sarif` input.
- [ ] **Step 3:** Record the outcome, with the release version.

### Fixture runs

File with `fixture-run vergil-project/.github#1476
python-lib,python-app,rust-lib,rust-app,ts-lib,ts-app --validates
"vergil-actions <Task 6 version>: private-repo SARIF paths removed; CodeQL
always runs" --after vergil-project/vergil-actions#<Task 6>`, once Task 6
exists.

### Task 7a–7z: caller cleanup, pass 2 (one task per repo)

One `impl` task per repo, filed in that repo, blocked by Task 6. The
`logical-minds-foundry` rows go under the proxy, and the rest go under the
epic. Each is the same one-line change.

**What changes.** In `.github/workflows/ci.yml`, in the `security:` job,
delete the `actions: read` line and the two-line comment above it
(`Requested by ci-security.yml@…: codeql-action/upload-sarif needs it on
private repos…`). Task 5 has removed every `actions: read` request from
`ci-security.yml`, so the grant is no longer needed. Never touch an
`actions: read` that has another reason: the workflow level, or a `cd.yml`
evidence harvest.

These are the rows, from a fleet-wide search of `origin/develop` on
2026-11-22. Re-run the search when filing.

- **vergil-project** (14): `docs`, `.github`, `packages`,
  `vergil-claude-plugin`, `vergil-containers`, `vergil-python`,
  `vergil-tooling`, `vergil-vm`, and the six `vergil-fixture-*` repos.
- **logical-minds-foundry** (5): `docs`, `.github`, `mq-gateway-replay-lab`,
  `mq-resiliency-lab-for-linux` and `mq-resiliency-observability`.
- **mnemosys-project** (3): `docs`, `.github` and `melete`.
- **mq-rest-admin-project** (6): `.github`, `mq-rest-admin-common`,
  `mq-rest-admin-dev-environment`, `mq-rest-admin-go`,
  `mq-rest-admin-python` and `mq-rest-admin-rust`.

vergil-actions is done in Task 5. diogenes-project and vergils-nemesis have
no grant.

- [ ] **Step 1:** Make the change.
- [ ] **Step 2:** `vrg-container-run -- vrg-validate` passes, and the PR's
  `security` job starts and passes.
- [ ] **Step 3:** Commit. The human pushes it.

### Task 8: vergil-tooling: epic home and queue placement

**Files:**

- Modify `src/vergil_tooling/lib/epics.py`:
  - `resolve_epic_home` always returns `<org>/.github`, with no visibility
    probe. It keeps the #3643 missing-`.github` refusal, which now checks
    existence directly.
  - `queue_home` returns `<org>/.github` and the org-level queue.
  - `_queue_homes` goes.
  - The self-home branches go from the ad-hoc drain and the archive-home
    paths, and from bare-`#N` resolution.
  - `require_org_home` keeps its existence check, but its message loses
    the `--repo` hint.
- Modify `src/vergil_tooling/lib/queue_view.py`: the repo-scoped planning
  view, and the `.github`-is-public branch for advisories, go.
- Modify the CLIs:
  - In `vrg_epic_docs.py`, `vrg_roadmap.py` and `vrg_epic_audit.py`, remove
    `--repo` where it exists only to reach a self-homed queue or home.
    Before removing each flag, confirm from its help text and its callers
    that self-homing is its only use. If it has another use, keep it as
    for `vrg-queue`, below.
  - `vrg_queue.py`'s `--repo` stays. It also narrows `adhoc` and `hoshin`
    to one repo and picks another org, and hoshin and epic-implement call
    it that way. Only its self-home help clause, and the self-home branch
    behind it, go.
  - In `vrg_epic_create.py`, `vrg_issue_create.py`, `vrg_intake_create.py`,
    `vrg_adhoc_epic.py` and `vrg_epic_move.py`, `--repo` names a target
    repo, so it stays. Only its self-home help text, and the home echo's
    `[visibility]` tag, go.
- Tests: `tests/vergil_tooling/test_epics.py`, `test_queue_view.py`,
  `test_vrg_queue.py`, `test_vrg_intake_create.py`,
  `test_vrg_issue_create.py` and the CLI tests of each touched command.
- Docs:
  - delete `docs/site/docs/guides/epic-home-visibility-flip.md` and its nav
    entry, plus inbound links;
  - edit `standards/github-issues.md` and the self-home entries in
    `reference/cli-tools-overview.md`.

**Interfaces:** `resolve_epic_home(org: str, target_repo: str) -> str` and
`queue_home(target_repo: str) -> tuple[str, str]` keep their signatures.
Task 14 relies on this task having removed their calls to `is_public`.

- [ ] **Step 1: Write the failing tests.**
  - `test_resolve_epic_home_is_org_dotgithub_without_probe`: with
    `github.is_public` patched to raise, `resolve_epic_home("o", "r") ==
    "o/.github"`.
  - `test_resolve_epic_home_missing_dotgithub_keeps_3643_message`: when
    `o/.github` does not exist, it raises `EpicHomeError` naming
    `o/.github`.
  - `test_queue_home_is_org_dotgithub`.
- [ ] **Step 2:** Run them. Expected: FAIL.
- [ ] **Step 3:** Make the edits above. Delete the tests of the removed
  branches, and the tests of each removed `--repo` flag.
- [ ] **Step 4:** Run the leftover search over the touched files. `vrg-
  container-run -- vrg-validate` passes, with 100% branch coverage and a
  strict MkDocs build.
- [ ] **Step 5:** Close vergil-project/vergil-tooling#4636 as not planned,
  with `Superseded by vergil-project/.github#1476: there are no private or
  self-homed queue homes.` Commit.

### Task 9: vergil-tooling: epic-audit visibility passes

**Files:**

- Modify `src/vergil_tooling/lib/epic_audit.py`:
  - invariant 1's private-repo skip;
  - `OpenIssue.public`;
  - `_repo_open_issues(public=)`;
  - `public_self_homed_queues`, `move_public_queues` and their renderers;
  - `private_under_public_epic` and `_render_private_under_public`.
- Modify `src/vergil_tooling/bin/vrg_epic_audit.py`: the "Queues of
  now-public repos" and "private under public" sections, and their
  `--close` hook in `_apply_orphan_pass`.
- Tests: `test_epic_audit.py` and `test_vrg_epic_audit.py`.

- [ ] **Step 1: Write the failing test**
  `test_invariant1_reports_every_non_dotgithub_epic`: an epic outside
  `.github` is reported regardless of the repo's visibility, with no
  visibility field read.
- [ ] **Step 2:** Run it. Expected: FAIL.
- [ ] **Step 3:** Make the edits. Delete the removed passes' tests.
- [ ] **Step 4:** Run the leftover search over the touched files. `vrg-
  container-run -- vrg-validate` passes.
- [ ] **Step 5:** Commit.

### Task 10: vergil-tooling: token-coverage guard

**Files:**

- Modify `src/vergil_tooling/lib/epic_audit.py`: `coverage_gap` and
  `_login_coverage_gap`.
- Modify `src/vergil_tooling/bin/vrg_adhoc_epic.py`: `_refuse_partial_org`
  and its `--all-in` call.
- Tests: their tests.
- Review: `vrg_gh.py`'s multi-owner refusal, the `live_branches` ls-remote
  fallback, and the #3463 cross-org token comments.

- [ ] **Step 1:** Delete the guard, its call sites and its tests.
- [ ] **Step 2:** For each reviewed path, remove it where its comment and
  history show that visibility is its only reason. Where it also guards
  writes, keep it and correct its comment. List each decision in the PR
  body.
- [ ] **Step 3:** Run the leftover search. `vrg-container-run --
  vrg-validate` passes.
- [ ] **Step 4:** Commit.

### Task 11: vergil-tooling: GHAS inference and scaffolding

**Files:**

- Modify `src/vergil_tooling/lib/github_config.py`:
  - `ghas_available` goes, along with the `ghas` flag threaded through its
    callers;
  - `desired_repo_settings` loses its `visibility` parameter and the
    private-only `allow_forking`;
  - `_is_private_forking_error` and `_PRIVATE_FORKING_HELP` go;
  - the missing-visibility default in `fetch_actual_state` goes.
- Modify `src/vergil_tooling/lib/ci_evidence.py`: drop the GHAS call. The
  code-scanning evidence is always expected.
- Modify `src/vergil_tooling/bin/vrg_github_repo_config.py`: drop the
  "requires GHAS for private repos" skip.
- Modify `src/vergil_tooling/lib/config.py`: retire `ghas`. Remove it from
  `_PROJECT_FIELDS` and the `ghas` attribute, and reject it with the reason
  in Global constraints. Removing it from `_PROJECT_FIELDS` alone would
  only warn, because unknown keys warn. So the project parser rejects
  `[project].ghas` explicitly. The error is modelled on the
  `_RETIRED_PRIMARY_LANGUAGES` one, not on `_DEPRECATED_INSTALL_KEY`, which
  warns and keeps reading the old key.
- Modify `src/vergil_tooling/lib/repo_init.py` and
  `bin/vrg_github_repo_init.py`:
  - the `--visibility` flag and its prompt go, so repos are always created
    public;
  - in the generated `ci.yml`, the private-only `upload-sarif: false` block
    goes;
  - the template's `actions: read` grant on the security job goes.
- Tests: `test_github_config_lib.py`, `test_ci_evidence_harvest.py`,
  `test_repo_init.py`, the config tests, and `test_vrg_github_repo_*`.
- Docs: the GHAS and visibility passages in `docs/site/`.

**Interfaces:** `desired_repo_settings(*, is_org: bool) ->
DesiredRepoSettings`.

- [ ] **Step 0: Precondition: re-run the `ghas` search.** For each of the
  six organizations, search every non-archived repo's `vergil.toml` on its
  default branch for a `ghas` key. Expected: no hits. A hit means a repo
  would fail to load once this change deploys. If there is one, stop and
  report it, so that it gets a cleanup PR first.
- [ ] **Step 1: Write the failing tests.**
  - `test_project_ghas_is_retired`: loading a `vergil.toml` with
    `[project] ghas = true` raises with the exact retirement reason.
  - `test_repo_init_ci_has_no_upload_sarif`: the generated `ci.yml` has no
    `upload-sarif`, and its security job has no `actions: read`.
  - `test_desired_settings_have_no_visibility`.
- [ ] **Step 2:** Run them. Expected: FAIL.
- [ ] **Step 3:** Make the edits. Collapse the 117 `visibility=` fixtures in
  `test_github_config_lib.py` to the public case.
- [ ] **Step 4:** Run the leftover search. `vrg-container-run --
  vrg-validate` passes.
- [ ] **Step 5:** Commit.

### Task 12: vergil-tooling: routing, linking, proxies, security-records home

**Files:**

- Modify `src/vergil_tooling/lib/intake_route.py`: `_guard_visibility` and
  its five call sites go.
- Modify `src/vergil_tooling/bin/vrg_epic_link.py`: the less-visible-epic
  guard goes.
- Modify `src/vergil_tooling/lib/proxies.py`: `_refuse_private` and its
  calls go.
- Modify `src/vergil_tooling/lib/security_records.py`: the
  private-`.github` branch of `_resolve_home`, and `PrivateHomeError`, go.
- Modify the handlers that catch `PrivateHomeError`:
  `src/vergil_tooling/bin/vrg_epic_audit.py`, `bin/vrg_queue.py` (its
  `_VIEW_ERRORS`) and `bin/vrg_security_record.py`.
- Tests: `test_intake_route.py`, `test_vrg_epic_link.py`,
  `test_proxies.py`, `test_security_records.py`, `test_vrg_epic_audit.py`,
  `test_vrg_queue.py` and `test_vrg_security_record.py`.
- Docs: the private passages in `guides/security-records.md`.

- [ ] **Step 1: Write the failing test**
  `test_proxy_ensure_does_not_probe_visibility`: with `github.is_public`
  patched to raise, ensuring a proxy for a public-org pair succeeds.
- [ ] **Step 2:** Run it. Expected: FAIL.
- [ ] **Step 3:** Make the edits. Delete the removed guards' tests.
- [ ] **Step 4:** Run the leftover search. `vrg-container-run --
  vrg-validate` passes.
- [ ] **Step 5:** Commit.

### Task 13: vergil-tooling: Dependabot reporting

Tracks security advisory GHSA-2v43-fjxg-fv3c.

**Files:**

- Modify `src/vergil_tooling/bin/vrg_dependabot_audit.py`:
  - `_counts_only` and the visibility branch go;
  - every repo gets the counts-by-severity report with the Security-tab
    link;
  - the module docstring's private-repo text goes.
- Tests: `test_vrg_dependabot_audit.py`.
- Docs: `docs/site/docs/reference/dependabot-audit.md`.

- [ ] **Step 1: Write the failing test**
  `test_report_never_prints_alert_detail`: given alerts for package
  `leftpad-canary` with advisory `GHSA-xxxx-canary`, the output contains
  the severity counts and the Security-tab URL, and neither string. It
  must not read the repo's visibility (patch `repo_visibility` to raise).
- [ ] **Step 2:** Run it. Expected: FAIL.
- [ ] **Step 3:** Make the edits. Delete the itemized-report tests.
- [ ] **Step 4:** Run the leftover search. `vrg-container-run --
  vrg-validate` passes.
- [ ] **Step 5:** Commit.

### Task 14: vergil-tooling: visibility primitives

**Files:**

- Modify `src/vergil_tooling/lib/github.py`: delete `repo_visibility` and
  `is_public`. `repo_exists` gets an existence probe of its own, and its
  docstring loses its self-homing framing.
- Tests: `test_github.py`.

**Interfaces:** `repo_exists(owner: str, name: str) -> bool` keeps its
signature.

- [ ] **Step 1: Write the failing test**
  `test_repo_exists_does_not_read_visibility`: the probe's `gh` call does
  not request a `visibility` field. A missing repo returns `False`, and any
  other API error raises.
- [ ] **Step 2:** Run it. Expected: FAIL.
- [ ] **Step 3:** Delete the two functions. Search `src/` for each name.
  Expected: no hits. A hit means an earlier task missed a caller: remove
  that caller here.
- [ ] **Step 4:** Run the leftover search over all of vergil-tooling.
  Expected: allowed uses only. `vrg-container-run -- vrg-validate` passes.
- [ ] **Step 5:** Commit.

### Task 15: Release vergil-tooling

Repo: `vergil-project/vergil-tooling`, kind `deployment`.

**Precondition (human-attested):** the human has cut a vergil-tooling
release.

- [ ] **Step 1:** Confirm that the release tag contains each merge of
  Tasks 8–14: `vrg-git merge-base --is-ancestor <merge sha> <tag sha>`
  exits 0 for each.
- [ ] **Step 2:** Sync the release onto the host and VMs with the existing
  deploy steps.
  - For each of `vrg-epic-audit`, `vrg-epic-docs` and `vrg-roadmap` whose
    `--repo` Task 8 removed, confirm its `--help` no longer lists `--repo`.
  - Confirm that `vrg-queue --help` still lists `--repo`, without the
    self-home clause.
- [ ] **Step 3:** Record the outcome, with the version.

### Task 16: vergil-claude-plugin: hoshin

**Files:**

- `skills/hoshin/references/security.md`: the counts-only rule for private
  repos, the "Checking visibility" section, Example A, and the private-name
  screen lists all go. The screen's parts for advisory text and secrets
  stay.
- `skills/hoshin/references/daily.md`, `weekly.md`, `monthly.md`,
  `yearly.md`, `logbook.md` and `skills/hoshin/SKILL.md`: the per-org count
  clauses and the private-repo epic notes go.

- [ ] **Step 1:** Make the edits. Every repo is reported by name, the same
  way public repos are today.
- [ ] **Step 2:** Run the leftover search over `skills/hoshin/`. `vrg-
  container-run -- vrg-validate` passes.
- [ ] **Step 3:** Commit.

### Task 17: vergil-claude-plugin: epic skills

**Files:**

- `skills/epic-create/SKILL.md`:
  - the self-homing text in the overview, Preflight and the planning-issue
    section;
  - `vrg-queue planning --repo <private-repo>`;
  - the `--repo` for a private home;
  - the private-refusal clause of `--proxy-for`.
- `skills/epic-review/SKILL.md`: the self-homed queue branches, and the
  "Private work stays private" rule.
- `skills/epic-implement/SKILL.md` and `skills/intake-capture/SKILL.md`:
  the private-home mentions.
- `docs/site/docs/skills/index.md`.

- [ ] **Step 1:** Make the edits. Every flag a skill names must exist in the
  Task 15 release: check each with `--help`.
- [ ] **Step 2:** Run the leftover search over `skills/` and `docs/`. `vrg-
  container-run -- vrg-validate` passes.
- [ ] **Step 3:** Close vergil-project/vergil-claude-plugin#1139 as not
  planned, superseded by vergil-project/.github#1476. Commit.

### Closing bookends

- **Documentation review** (vergil-project/vergil-tooling#4893). Sweep the
  site docs for leftovers, and the `docs` repo's
  `management/epics-and-tasks.md`, `intake.md` and `index.md`. File a
  same-repo task in each repo that needs a change.
- **Retrospective** (vergil-project/.github#1478), with
  `epic-retrospective`.
