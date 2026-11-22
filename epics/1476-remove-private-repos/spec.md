# Remove private-repo support — design

- **Epic:** vergil-project/.github#1476
- **Planning issue:** vergil-project/.github#1474 (review history is in its comments)
- **Status:** draft, in design review

## 1. Intent

Remove private-repository support from Vergil. Vergil carries special cases
for private repos throughout the fleet. Private repos home their own epics
and queues, and several guards keep private work out of public homes. On top
of that come GitHub Advanced Security (GHAS) inference, counts-only reporting
in hoshin, and a token-coverage guard. Two repos in the fleet are private,
and the special cases cost far more than those two repos are worth.

This epic first removes the last private repos, by operational tasks:

- `logical-minds-foundry/mq-gateway-replay-lab` is made public;
- `logical-minds-foundry/mq-protocol-gateway` is archived.

From then on, **every repo Vergil manages is public**. That is an invariant
of the fleet, and the code is written to it. Vergil no longer branches on
visibility anywhere, and nothing checks the invariant at runtime. Every epic
and queue homes in its org's `.github`.

### 1.1 Why now

The fleet leaves GitHub in 2027 for a local Forgejo on the human's own
hardware. That inverts the visibility model. Everything on the local forge
is private by default, and "public" becomes the exception, done by
publishing a GitHub fork of a repo. The public/private concept is rethought
from scratch for that port. Removing the GitHub-era special cases now
simplifies the rest of 2026's work. It also shrinks what the port has to
re-implement.

Vergil is a personal laboratory, not a community-collaborative system, so
there is no need to design for shared private work. The two repos were
private only out of caution, while it was unclear whether an employer
contract allowed the code to be shared. It does. The human has declared the
content of both repos public, with nothing in them that matters.

## 2. Goals

1. No current Vergil code, workflow, skill or doc branches on repo
   visibility.
2. Each epic and queue homes in `<org>/.github`. Nothing self-homes.
3. `logical-minds-foundry/mq-gateway-replay-lab` is public. Its queues and
   ad-hoc work live in `logical-minds-foundry/.github`.
4. `logical-minds-foundry/mq-protocol-gateway` is archived. Its open issues
   are closed first.

## 3. Non-goals and anti-goals

- **Non-goal: the Forgejo migration** and its private-by-default model.
- **Non-goal: the lab repo's own roadmap.** Its rename to
  `mq-swift-settlement-lab` and its README rework are that repo's own
  issues.
- **Non-goal: personal-account repos.** Repos under the human's personal
  account are not Vergil repos and are not managed by Vergil.
  - Any that still carry a `vergil.toml` or Vergil-made epics are cleaned up
    by the human, outside this epic.
  - Their visibility is never inspected or changed.
- **Non-goal: a central list of the fleet's organizations.** A separate
  epic introduces one.
- **Non-goal: historical records.** Past specs, plans, superpowers docs, paad
  reports, changelogs and release notes keep their mentions. That includes
  epic #130's visibility-based epic home. They record what was true then.
- **Non-goal: other uses of "private".** These stay untouched:
  - App private keys;
  - private (draft) security advisories and private vulnerability
    reporting;
  - the merge-review private state directory;
  - `"private": true` in `package.json`;
  - CMake `PRIVATE`.
- **Anti-goal: a runtime guard.** No check is added that a repo is public,
  owned by an organization, in the fleet, or not archived, and no audit
  asserts that every repo is public. The invariant is established once, by
  the operational tasks. The human will not create a private repo, and the
  fleet's time on GitHub is three to six months. A check that runs all the
  time to stop the architecture regressing guards against ourselves. It is
  complexity the fleet does not need. If a private repo ever appears, things
  break where they break.
- **Anti-goal: one big removal.** Each subsystem is removed in its own PR. A
  removal can then be backed out and deployed independently while agents
  merge to develop. Review size is not the reason, since an agent reviews
  the code.

## 4. Design

### 4.1 What is removed

The inventory was taken against each repo's `develop` on 2026-11-22. It is a
guide, not a contract: each removal task's acceptance (§5) is behavioural,
so anything the inventory missed is still in scope.

**vergil-tooling: epic home and queue placement.**

- `lib/epics.py`: these collapse to `<org>/.github`, with no visibility
  probe:
  - `resolve_epic_home`, with its self-home branches, including the private
    no-`.github` owner case (#3603);
  - `queue_home`;
  - `_queue_homes`;
  - the ad-hoc drain and archive-home self-home paths;
  - bare-`#N` resolution against a self-home.

  The existing refusal when `<org>/.github` does not exist (#3643) is not a
  visibility branch, and stays.
- `require_org_home` keeps its existence check, but its message loses the
  `--repo` private hint.
- The `--repo` self-home paths and help text go from `vrg-queue`,
  `vrg-epic-docs`, `vrg-roadmap`, `vrg-epic-create`, `vrg-intake-create`,
  `vrg-issue-create` and `vrg-epic-move`. The home echo's
  `[visibility]` tag goes too.
- `lib/queue_view.py` loses its repo-scoped planning view, and its
  "advisories only if `.github` is public" branch.

**vergil-tooling: epic-audit visibility passes.** These go from
`lib/epic_audit.py`, along with their renderers and the `vrg-epic-audit`
output sections:

- invariant 1's private-repo skip;
- `OpenIssue.public`;
- `_repo_open_issues(public=)`;
- `public_self_homed_queues`, `move_public_queues` and the
  "Queues of now-public repos" section;
- `private_under_public_epic`.

**vergil-tooling: token-coverage guard.**

- `coverage_gap`, `_login_coverage_gap` and `vrg-adhoc-epic`'s
  `_refuse_partial_org` (#4579) exist only because a token might not see
  every private repo. With every repo public, they go.
- These related paths are each checked: `vrg-gh`'s multi-owner refusal, the
  `live_branches` ls-remote fallback, and the cross-org token comments
  (#3463). Each is removed only where its sole reason is visibility, and
  kept, with its comment corrected, where it also guards writes.

**vergil-tooling: GHAS inference and repo scaffolding.**

- `lib/github_config.py`:
  - `ghas_available` and the `ghas` flag threaded through its callers;
  - the `visibility` parameter of `desired_repo_settings`, and
    `allow_forking` for private org repos;
  - `_is_private_forking_error` and `_PRIVATE_FORKING_HELP`;
  - the "missing visibility means private" default.
- `lib/ci_evidence.py`'s GHAS call.
- The "requires GHAS for private repos" skip in `vrg-github-repo-config`.
- `[project].ghas` in `lib/config.py` is retired: a `vergil.toml` that still
  sets it is rejected with a retirement reason. Every repo that sets it
  drops it first (§4.3).
- `lib/repo_init.py` and `vrg-github-repo-init`:
  - the `--visibility` flag and its prompt go (repos are created public);
  - the private-only `upload-sarif: false` block in the generated `ci.yml`
    goes, and so does its `actions: read` grant, once vergil-actions no
    longer requests it (§4.3).

**vergil-tooling: routing, linking and proxies.**

- `intake_route._guard_visibility` and its call sites go.
- `vrg-epic-link`'s less-visible-epic guard goes.
- `lib/proxies.py`'s private refusal goes.
- `security_records._resolve_home`'s private-`.github` branch and
  `PrivateHomeError` go.

**vergil-tooling: reporting.** `vrg-dependabot-audit`'s itemized-for-private
versus counts-only split collapses to the counts-only report: counts by
severity plus a link to the repo's Security tab. It never prints per-alert
detail, because its workflow logs are world-readable.

**vergil-tooling: visibility primitives.** `lib/github.py`'s
`repo_visibility` and `is_public` go, once the other removals have taken
away their callers. `repo_exists`, which is built on `repo_visibility`, is
kept with an existence probe of its own, and its docstring loses its
self-homing framing.

**vergil-tooling: docs.**

- `docs/site/docs/guides/epic-home-visibility-flip.md` and its nav entry are
  deleted.
- The private-repo passages go from `standards/github-issues.md`,
  `reference/cli-tools-overview.md`, `reference/dependabot-audit.md` and
  `guides/security-records.md`.
- Each removal PR carries its own doc edits.

**vergil-actions.** SARIF always uploads and CodeQL always runs.

- `ci-security.yml` loses the `upload-sarif` input.
- The Semgrep and Trivy actions lose their own `upload-sarif` inputs. They
  also lose the fallback steps that attach SARIF as an artifact when the
  input is not `true`.
- The gate's "skipped (no GHAS)" handling goes.
- `ci-security.yml`'s jobs stop requesting the `actions: read` permission,
  which `codeql-action/upload-sarif` needs only on private repos.
- The private-repo input docs go from those actions' docs pages, the
  `ci-security` workflow page and the `ops-epic-sweep` page.
- Removing an input from a reusable workflow breaks every caller still
  passing it, so every caller drops `upload-sarif` first. Removing a
  permission is the reverse: a caller that stops granting one the reusable
  workflow still requests fails at startup, so vergil-actions stops
  requesting `actions: read` first (§4.3).

**vergil-claude-plugin (skills).**

- `hoshin`:
  - the counts-only rule for private repos;
  - the "Checking visibility" section and Example A in
    `references/security.md`;
  - the private-name screen lists;
  - the per-org count clauses in `daily.md`, `weekly.md`, `monthly.md`,
    `yearly.md`, `logbook.md` and `SKILL.md`.

  Any part of the security screen that also guards something other than
  private names (advisory text, secrets) stays.
- `epic-create`, `epic-review`, `epic-implement` and `intake-capture` lose
  their self-homed-queue and private-home branches. That includes
  `epic-review`'s "private work stays private" rule, and `--proxy-for`'s
  private refusal note.
- The skills index page loses its mentions.

**Superseded issues.** vergil-project/vergil-tooling#4636 (a checked listing
of private and queue homes) and vergil-project/vergil-claude-plugin#1139
(move runbooks to that listing) are closed as superseded by this epic.

### 4.2 The operational side

These run before any code is removed, because the removal makes self-homed
queues unreachable. Every operational task is filed in
`logical-minds-foundry/.github`, under that org's proxy of this epic: a proxy
refuses a private target repo, and both repos are private until these tasks
run.

**Publish `logical-minds-foundry/mq-gateway-replay-lab`.**

1. Flip the repo to public. This is a human action, on the human's
   declaration that its content is public (§1.1). No scan or review of its
   history, issues, logs or artifacts precedes it.
2. Move its self-homed queue items. Move each open child of its intake epic
   (#322) and deferred epic (#329) under the `logical-minds-foundry/.github`
   epic of the same kind, with `vrg-epic-move`. Then close #322 and #329,
   each with a pointer to where its items went. The agent does this
   directly; `vrg-epic-audit --close` and its org-wide close pass are not
   used.
3. Move its ad-hoc work:
   - Create `Epic (ad hoc): mq-gateway-replay-lab` in
     `logical-minds-foundry/.github` with `vrg-adhoc-epic`, as every other
     repo in the org has.
   - Move each open child of the repo's own ad-hoc epic (#71) under it with
     `vrg-epic-move`.
   - Close #71 and the ad-hoc archive (#319), each with a pointer to where
     its items went.
4. Close the repo's #307 ("secret-scan history + make public") as
   superseded by this task, with a pointer.
5. Drop `upload-sarif: false` and any `ghas` key from it. That is an
   ordinary PR in the repo.

**Archive `logical-minds-foundry/mq-protocol-gateway`.**

1. Close its open issues, including its intake and ad-hoc epics and the
   ad-hoc archive, as not planned, with a pointer to this epic.
2. Archive it, leaving it private. This is a human action.

Once both are done, the invariant of §1 holds. The human has enumerated
the only two private repos, and nothing checks for others.

### 4.3 Fleet-wide caller cleanup

Callers are cleaned up in two passes, in an order that never breaks CI.
Every caller uses `ci-security.yml` through the moving `@v2.1` tag, so a
vergil-actions release reaches them all at once.

1. **Before the vergil-actions change:** every non-archived repo stops
   setting `upload-sarif` and `[project].ghas`. For example,
   `logical-minds-foundry/mq-resiliency-lab-for-linux` sets both today. The
   input defaults to `true`, so dropping `upload-sarif: true` changes
   nothing.
2. **After the vergil-actions release that stops requesting `actions:
   read`:** every non-archived repo stops granting it, and drops its
   private-repo comment.

Archived repos are left alone: they are read-only and run no CI. Each repo
that needs it gets an ordinary PR in each pass. The plan lists them from a
fleet-wide search at plan time, and each task that depends on a pass re-runs
that search as a precondition.

## 5. Acceptance

- **Grep.** In vergil-tooling, vergil-actions and vergil-claude-plugin, run
  a case-insensitive, word-bounded grep over current files for these terms:
  - `self-home`, `self_home`, `self-homed`;
  - `private repo`, `private repos`, `private home`;
  - `upload-sarif`, `ghas_available`, `[project].ghas`;
  - `visibility`.

  Word-bounded means `\bprivate repos?\b`, so "private reports" is not a
  hit. It returns no hits outside these allowed uses:
  - the `github/codeql-action/upload-sarif` action, which does the upload;
  - `vrg-gh`'s allow-list entries for `gh`'s own `--visibility` flag;
  - `visibility` in a sense other than repository visibility, such as
    timing or a session's visibility;
  - the `[project].ghas` retirement and its test;
  - the historical paths `docs/specs/`, `docs/plans/`, `docs/superpowers/`,
    `docs/reports/`, `paad/`, `releases/` and `CHANGELOG.md`.
- **No inputs or grants left.** No reusable workflow or action in
  vergil-actions has an input named `upload-sarif`. Neither vergil-actions'
  `ci-security.yml` nor any non-archived caller, nor the `ci.yml` that
  `vrg-github-repo-init` generates, requests or grants `actions: read` for
  the security workflow.
- **Homes.** `vrg-queue`, `vrg-epic-create` and `vrg-issue-create` resolve
  every repo's home to `<org>/.github` without a visibility probe.
- **The two repos.** `mq-gateway-replay-lab` is public. Its queues and
  ad-hoc work are in `logical-minds-foundry/.github`. `mq-protocol-gateway`
  is archived.
- **Validation passes.** `vrg-container-run -- vrg-validate` passes in each
  changed repo, with vergil-tooling at 100% branch coverage.

## 6. Delivery shape

Operational first, then the caller cleanup and the vergil-actions change
in their safe order, then one removal per subsystem.

1. **Operational** (in `logical-minds-foundry/.github`, under a proxy of
   this epic): publish the lab repo and archive the gateway (§4.2). The
   visibility flip and the archive are human-attested preconditions, never
   performed by the agent.
2. **Caller cleanup, pass 1** (§4.3): one PR per repo that sets `ghas` or
   `upload-sarif`.
3. **vergil-actions**: drop the `upload-sarif` inputs, their artifact
   fallback and the `actions: read` request (blocked by pass 1). It ships in
   a vergil-actions release. Because it changes the shared security workflow
   every language uses, the plan seeds **fixture runs** for every fixture in
   the `fixture-run` table, blocked by that release.
4. **Caller cleanup, pass 2** (§4.3): one PR per repo that grants `actions:
   read`, blocked by that vergil-actions release.
5. **Removals** in vergil-tooling, one PR each. Each is blocked by both
   operational tasks:
   - epic home and queue placement;
   - epic-audit visibility passes;
   - token-coverage guard;
   - GHAS inference and scaffolding (also blocked by pass 1 and by the
     vergil-actions release, since its generated `ci.yml` drops the
     `actions: read` grant);
   - routing, linking and proxies;
   - Dependabot reporting;
   - visibility primitives, last, blocked by the six removals above, which
     take away their callers.
6. **vergil-claude-plugin**: the hoshin change, and the epic-skill change,
   each its own PR. The epic-skill change is blocked by the epic-home
   removal's release, so the runbooks never describe a tool that does not
   exist yet.
7. The documentation-review (vergil-project/vergil-tooling#4893) and
   retrospective (vergil-project/.github#1478) bookends close the epic.

## 7. Left to the builder

- For each related token path in §4.1, whether visibility is its only
  reason.
- Wording of the doc and skill edits.
- Whether a test that mixes private and public cases is trimmed or
  rewritten.
