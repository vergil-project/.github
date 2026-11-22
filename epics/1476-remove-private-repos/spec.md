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

After this epic:

- Vergil manages only **non-archived repos owned by one of the fleet's six
  GitHub organizations**: `vergil-project`, `logical-minds-foundry`,
  `mnemosys-project`, `diogenes-project`, `mq-rest-admin-project` and
  `vergils-nemesis`.
- Every such repo is **public**. Meeting a private one is a loud, named
  error, never a silent fallback.
- Every epic and queue homes in its org's `.github`.

### 1.1 Why now

The fleet leaves GitHub in 2027 for a local Forgejo on the human's own
hardware. That inverts the visibility model. Everything on the local forge
is private by default, and "public" becomes the exception, done by
publishing a GitHub fork of a repo. The public/private concept is rethought
from scratch for that port. Removing the GitHub-era special cases now
simplifies the rest of 2026's work. It also shrinks what the port has to
re-implement.

Vergil is a personal laboratory, not a community-collaborative system, so
there is no need to design for shared private work. The human has declared
the content of both private repos public.

## 2. Goals

1. No current Vergil code, workflow, skill or doc branches on repo
   visibility, except the single guard in §4.1 and the audit check in §4.2.
2. A repo that Vergil meets fails loudly with a clear reason when it is any
   of these:
   - private;
   - owned by a personal account, not an organization;
   - outside the six organizations.
3. Archived repos are invisible to Vergil. Every org-wide listing skips them,
   and no tool acts on one.
4. Each epic and queue homes in `<org>/.github`. Nothing self-homes.
5. `logical-minds-foundry/mq-gateway-replay-lab` is public, and its queues
   live in `logical-minds-foundry/.github`.
6. `logical-minds-foundry/mq-protocol-gateway` is archived. Its open issues
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
- **Anti-goal: a fallback.** No path may quietly treat a private repo as
  public. It also may not drop such a repo from a listing, or home its epics
  somewhere "safe". The guard refuses, and the refusal names the repo and
  the rule.
- **Anti-goal: one big removal.** Each subsystem is removed in its own PR. A
  removal can then be backed out and deployed independently while agents
  merge to develop. Review size is not the reason, since an agent reviews
  the code.

## 4. Design

### 4.1 The guard (one rule, one place)

One `vergil-tooling` function, called `require_supported_repo(owner, repo)`
below (the builder names it), is the only place that inspects visibility. It
raises a single error type, with one message per case:

- **The owner is not an organization:** "Vergil requires repositories owned
  by an organization; `<owner>` is a personal account."
- **The owner is not one of the six organizations:** "`<owner>` is not a
  Vergil organization."
- **The repo is archived:** "`<owner>/<repo>` is archived; Vergil does not
  act on archived repositories."
- **The repo is private:** "Private repositories are unsupported;
  `<owner>/<repo>` is private."

- **Probe failures stay loud.** Any probe failure raises a loud error, as
  `resolve_epic_home` does today.
- **Where it is called.** It is called wherever Vergil today resolves a
  repo's epic or queue home, or decides a repo's visibility. Every one of
  those paths therefore refuses instead of branching.
- **The six organizations.** The list lives in one checked place, and the
  builder chooses where. A fixed constant is acceptable, since the list
  changes rarely and a change to it is a reviewed PR.
- **Archived repos in listings.** Every org-wide listing skips archived
  repos before anything else: the epic audit's open-issue scan, the
  queue-home scan, the GitHub config audit and the Dependabot audit.
  Skipping an archived repo in a listing is not a refusal. A tool aimed at
  one directly gets the error.

### 4.2 The audit check

The GitHub config audit (`vrg-github-repo-config` and the daily
`ops.yml`) reports a failure for every non-archived repo of a Vergil
organization that is private. It is a fail-loud invariant. If a repo is
ever made private, the audit fails rather than Vergil quietly self-homing
it. That replaces the visibility-flip guide, which goes.

### 4.3 What is removed

The inventory was taken against each repo's `develop` on 2026-11-22. It is a
guide, not a contract: each removal task's acceptance (§5) is behavioural,
so anything the inventory missed is still in scope.

**vergil-tooling: epic home and queue placement.**

- `lib/epics.py`: these collapse to `<org>/.github` behind the guard:
  - `resolve_epic_home`, with its self-home branches, including the
    no-`.github` owner case (#3603), which the organization rule now
    refuses;
  - `queue_home`;
  - `_queue_homes`;
  - the ad-hoc drain and archive-home self-home paths;
  - bare-`#N` resolution against a self-home.
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

`move_public_queues` is used once, by §4.4, before it is removed.

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
  - `allow_forking` for private org repos;
  - `_is_private_forking_error` and `_PRIVATE_FORKING_HELP`;
  - the "missing visibility means private" default, which becomes a loud
    error.
- `lib/ci_evidence.py`'s GHAS call.
- The "requires GHAS for private repos" skip in `vrg-github-repo-config`.
- `[project].ghas` in `lib/config.py` is retired: a `vergil.toml` that still
  sets it is rejected with a retirement reason. Every repo that sets it
  drops it first (§4.5).
- `lib/repo_init.py` and `vrg-github-repo-init`:
  - the `--visibility` flag and its prompt go (repos are created public);
  - the private-only `upload-sarif: false` block in the generated `ci.yml`
    goes.

**vergil-tooling: routing, linking and proxies.**

- `intake_route._guard_visibility` and its call sites go.
- `vrg-epic-link`'s less-visible-epic guard goes.
- `lib/proxies.py`'s private refusal is replaced by the guard.

**vergil-tooling: reporting.** `vrg-dependabot-audit`'s itemized-for-private
versus counts-only split collapses to one report.

**vergil-tooling: visibility primitives.** `lib/github.py`'s
`repo_visibility` and `is_public` remain only as the guard's and audit's
probes. `repo_exists`'s docstring loses its self-homing framing.
`security_records._resolve_home` and `PrivateHomeError` either fold into the
guard or remain as a "`.github` is public" assertion. The builder chooses,
and the assertion must still fail loudly.

**vergil-tooling: docs.**

- `docs/site/docs/guides/epic-home-visibility-flip.md` and its nav entry are
  deleted.
- The private-repo passages go from `standards/github-issues.md`,
  `reference/cli-tools-overview.md`, `reference/dependabot-audit.md` and
  `guides/security-records.md`.
- Each removal PR carries its own doc edits.

**vergil-actions.**

- `ci-security.yml` loses the `upload-sarif` input. SARIF always uploads and
  CodeQL always runs.
- The gate's "skipped (no GHAS)" handling goes.
- The private-repo input docs go from the Semgrep and Trivy actions, and
  from their docs pages and the `ops-epic-sweep` page.
- Removing an input from a reusable workflow breaks every caller still
  passing it. So every caller in the six organizations drops `upload-sarif`
  first (§4.5).

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

### 4.4 The operational side

These run before any code is removed, because the removal makes self-homed
queues unreachable.

**Publish `logical-minds-foundry/mq-gateway-replay-lab`.**

1. Scan its full git history for secrets. Its own issue #307 asks for this.
   A human attests the result. A finding is fixed (rotated, history
   rewritten if needed) before the flip, never after.
2. Flip the repo to public. This is a human action.
3. Move its self-homed queues (intake, deferred) into
   `logical-minds-foundry/.github`, using `vrg-epic-audit`'s existing
   now-public queue move. Its ad-hoc epic and ad-hoc archive are moved or
   closed the same way.
4. Drop `upload-sarif: false` and any `ghas` key from it. That is an
   ordinary PR in the repo.

**Archive `logical-minds-foundry/mq-protocol-gateway`.**

1. Close its open issues, including its intake and ad-hoc epics and the
   ad-hoc archive, as not planned, with a pointer to this epic.
2. Archive it, leaving it private. This is a human action.

From then on the guard and listings ignore it. If the repo is ever
unarchived, the audit check (§4.2) fails until it is made public.

### 4.5 Fleet-wide caller cleanup

Before `[project].ghas` is retired and before the `upload-sarif` input is
removed, every repo in the six organizations stops setting them. For
example, `logical-minds-foundry/mq-resiliency-lab-for-linux` sets both
`ghas` and `upload-sarif: true` today. Each such repo gets an ordinary PR.
The plan lists them from a fleet-wide search at plan time, and the removal
tasks re-run that search as a precondition.

## 5. Acceptance

- **Grep.** In vergil-tooling, vergil-actions and vergil-claude-plugin, a
  case-insensitive grep over current files for these terms returns hits
  only in the guard, the audit check, their tests and the historical paths:
  - `self-home`, `self_home`, `self-homed`;
  - `private repo`, `private home`;
  - `upload-sarif`, `ghas_available`, `[project].ghas`;
  - `visibility`, outside the retained probes and the timing sense of the
    word.

  The historical paths are `docs/specs/`, `docs/plans/`, `docs/superpowers/`,
  `docs/reports/`, `paad/`, `releases/` and `CHANGELOG.md`.
- **The guard.** Each of the four guard cases has a test proving its message.
  `vrg-queue`, `vrg-epic-create` and `vrg-issue-create` refuse a private
  target loudly.
- **The audit check.** It fails on a private, non-archived repo and ignores
  an archived one, proven by tests.
- **The two repos.** `vrg-gh repo list` for each of the six organizations
  shows no non-archived private repo. `mq-gateway-replay-lab` is public with
  its queues in `logical-minds-foundry/.github`, and `mq-protocol-gateway`
  is archived.
- **The audits run clean.** The org-wide `vrg-epic-audit` and the GitHub
  config audit run clean across the six organizations against the released
  guard.
- **Validation passes.** `vrg-container-run -- vrg-validate` passes in each
  changed repo, with vergil-tooling at 100% branch coverage.

## 6. Delivery shape

Operational first, then the guard, then one removal per subsystem.

1. **Operational** (in `logical-minds-foundry`, under a proxy of this
   epic): publish the lab repo (§4.4) and archive the gateway (§4.4). The
   human-gated steps (secret-scan attestation, visibility flip, archive)
   are attested preconditions, never performed by the agent.
2. **Caller cleanup** (§4.5): one PR per repo that sets `ghas` or
   `upload-sarif`.
3. **Guard** (§4.1, §4.2): one vergil-tooling PR. Then a release (a
   human-gated precondition), a deployment, and a **validation** that runs
   the org-wide audits across the six organizations against the released
   guard. The validation proves that no private repo is reachable, so every
   private branch is now dead code. Every removal task is blocked by it.
4. **Removals**, one PR each, independent of each other once the guard is
   validated:
   - epic home and queue placement;
   - epic-audit visibility passes (also blocked by the lab repo's queue
     move);
   - token-coverage guard;
   - GHAS inference and scaffolding (also blocked by the caller cleanup);
   - routing, linking and proxies;
   - Dependabot reporting.
5. **vergil-actions**: drop `upload-sarif` (blocked by the caller cleanup).
   It ships in a vergil-actions release. Because it changes the shared
   security workflow every language uses, the plan seeds **fixture runs**
   for all four fixtures, blocked by that release.
6. **vergil-claude-plugin**: the hoshin change, and the epic-skill change,
   each its own PR. The epic-skill change is blocked by the epic-home
   removal's release, so the runbooks never describe a tool that does not
   exist yet.
7. The documentation-review (vergil-project/vergil-tooling#4893) and
   retrospective (vergil-project/.github#1478) bookends close the epic.

## 7. Left to the builder

- The guard's name, module and error type, and where the six-organization
  list lives (§4.1).
- Whether `security_records`' private-home check folds into the guard or
  stays as an assertion (§4.3).
- For each related token path in §4.3, whether visibility is its only
  reason.
- Wording of the doc and skill edits.
- Whether a test that mixes private and public cases is trimmed or
  rewritten.
