# Report-only Dependabot alerting across the fleet

- **Epic:** vergil-project/.github#420
- **Status:** Draft design (2026-10-22): superpowers brainstorming, builder
  check run, and three human decisions folded in (accepted red window;
  exact-match `dependabot.yml`; App/account inventory before the permission
  change). Hardened by paad pushback the same day: the nightly file check moves
  to a second release, public-repo reports show counts only, the file check
  becomes a local audit, and `diff` runs before the fleet-wide `apply`. Pending
  human review.
- **Repos:** vergil-project/vergil-tooling and vergil-project/vergil-actions
  (implementation); every fleet repo (rollout). The epic is homed in `.github`.
- **Related work:**
  - vergil-project/.github#155 (container pinning): the parent epic. §11 of its
    spec deferred report-only Dependabot to this follow-on.
  - vergil-project/.github#158: the brainstorm bookend that spawned this epic.
  - vergil-project/.github#52 / vergil-tooling#1599 (warn-only dependency
    update): conceptually adjacent, different mechanism.
  - vergil-project/.github#325 (fleet sweep): `vrg-fleet-sync` / `fleet_sweep.py`,
    reused here for the `dependabot.yml` rollout.
- **Seeded tasks:** .github#421 (documentation), vergil-tooling#3423
  (documentation review), .github#422 (retrospective), vergil-actions#999
  (deployment: App permission walk-through).

## 1. Summary

Turn on **Dependabot alerts** for every repo in the fleet, with Dependabot never
opening a pull request. Add a nightly check to the shared ops workflow that
**fails on any open Dependabot alert**. The failed scheduled run, and the email
GitHub sends for it, is the alert. Commit a PR-suppressed `.github/dependabot.yml`
in every repo so external auditors (the OpenSSF Scorecard) see a dependency-update
tool.

## 2. Why

The fleet keeps dependencies current as a side effect of its workflow. Every
release runs a dependency update (`vrg-update-deps`), active development pulls
pins forward, and `vrg-validate` runs a language audit (`pip-audit`,
`govulncheck`, …) on every change. None of that happens in a **dormant** repo.
Its pins stay fixed while the world moves on and new advisories are published
against those exact versions. No development means no audit, so a known
vulnerability can sit there unnoticed indefinitely. This is the trailing-edge
exposure §2 of #155 named.

This epic closes that gap: every repo is watched every night, active or not.

Two drivers:

1. **Passive monitoring of dormant repos**, so the team can respond to a new
   advisory before the repo falls further behind.
2. **OpenSSF Scorecard.** The aim is the highest achievable score, and the
   *Dependency-Update-Tool* check needs a Dependabot (or Renovate) config.

This is **defense in depth, a nice-to-have, not a primary control.** The release
cycle and CI remain the primary dependency hygiene.

## 3. Goals, non-goals, anti-goals

**Goals**

- G1. Every managed repo has Dependabot alerts enabled, enforced by desired
  state, so new repos and drift are covered automatically.
- G2. Every night, every repo's ops workflow fails if the repo has **any** open
  Dependabot alert, at any severity, and the run output names each alert.
- G3. Every managed repo carries a `.github/dependabot.yml` that opens no PRs, so
  Scorecard's Dependency-Update-Tool check passes.
- G4. New repos get all of the above from scaffolding, with no manual step.

**Non-goals**

- **Pin-drift reporting** (how far pins lag latest, beyond known CVEs). #155
  delegated an "upstream-distance" view here; the human ruled it an **optional
  add-on**, to be built in our own tooling if ever wanted, not in this epic.
- **Container / Dockerfile pins.** The dependency graph does not cover Docker
  (§4). It is out of scope by human decision: the dev and prod images are rebuilt
  every night, and those builds fail on security issues, so the containers are
  already watched daily.
- **The response process.** Reviewing and acting on nightly/weekly batch-job
  failures is a **separate, planned epic**. This epic builds the *mechanism* that
  produces the failure; it does not define who triages it or how.
- **Auto-remediation.** No fix PRs, no auto-bumps.

**Anti-goals** (failure even if everything "works")

- A1. **Dependabot opens a PR anywhere.** PRs created outside `vrg-submit-pr` and
  the human-gated merge flow are incompatible with the fleet's workflow.
- A2. **Something slips through.** A low-severity alert that does not fail the
  job, or an audit that silently passes when it cannot read alerts (alerts
  disabled, missing permission, API error), defeats the purpose. Per the fleet's
  no-silent-failures rule, "could not check" is a failure, never a pass.

## 4. Verified facts (data) and how they shape the design

Each fact was checked against a primary source on 2026-10-22.

| # | Fact | Source | Design consequence |
|---|------|--------|--------------------|
| F1 | Scorecard *Dependency-Update-Tool* "can determine only whether the dependency update tool is enabled," detected by the tool's config file in its standard location. It does not check that PRs are opened or merged. | [Scorecard checks.md](https://github.com/ossf/scorecard/blob/main/docs/checks.md) | A config file that opens no PRs is enough (G3). |
| F2 | Scorecard *Vulnerabilities* uses OSV.dev and is independent of Dependabot. | same | This epic does not move that check; no work here. |
| F3 | `open-pull-requests-limit: 0` disables **version-update** PRs. **Security-update** PRs have a separate fixed limit that this key does not control. | [dependabot.yml options](https://docs.github.com/en/code-security/dependabot/dependabot-version-updates/configuration-options-for-the-dependabot.yml-file) | PR suppression needs **both** `limit: 0` in the file **and** the repo setting "Dependabot security updates" disabled (A1). |
| F4 | `GET /repos/{o}/{r}/dependabot/alerts` takes `state` (incl. `open`) and returns `security_advisory.ghsa_id`, `security_advisory.severity`, `dependency.package.{ecosystem,name}`, `dependency.manifest_path`, `html_url`. | [Dependabot alerts REST](https://docs.github.com/en/rest/dependabot/alerts) | The audit command's input and its report fields (§5.2). |
| F5 | That endpoint needs the GitHub App repository permission **"Dependabot alerts: Read"**. `PUT /repos/{o}/{r}/vulnerability-alerts` needs **"Administration: Write"**. | [App permissions](https://docs.github.com/en/rest/authentication/permissions-required-for-github-apps) | The ops App needs a new permission: deployment task vergil-actions#999. |
| F6 | The dependency graph does not support Docker. GitHub Actions workflows are supported. | [Graph ecosystems](https://docs.github.com/en/code-security/supply-chain-security/understanding-your-software-supply-chain/dependency-graph-supported-package-ecosystems) | Grounds the container non-goal (§3). Workflow `uses:` pins **are** covered. |
| F7 | The dependency graph **reads `uv.lock` natively.** GitHub's docs do not list uv, and community reports conflict, so this was checked empirically: alerts were enabled on vergil-tooling on 2026-10-22, and the human saw many `uv.lock` entries in its dependency graph. | Human observation (Insights → Dependency graph) | No dependency-submission workaround is needed for the uv-based Python fleet. |

Current state in code: `desired_security_settings()` in
`vergil-tooling/src/vergil_tooling/lib/github_config.py` hard-codes
`vulnerability_alerts=False` and `dependabot_security_updates="disabled"`. The
read side (`_fetch_vulnerability_alerts`) and the apply side
(`_apply_security_settings`, which PUTs/DELETEs `vulnerability-alerts`) already
exist. No repo has a `dependabot.yml` today.

## 5. Design

The design rides the existing **nightly ops** model. Every repo's `ops.yml`
calls the reusable `vergil-actions/.github/workflows/ops-github-config.yml@v2.1`,
which already runs `vrg-github-repo-config audit` and `branch-sweep` for the
calling repo. Anything added to that reusable workflow reaches the whole fleet
with **no consumer change**, the same way `branch-sweep` was rolled out (#952).
The human's original phrase "add it to our ops workflow" means exactly this.

### 5.1 Repo settings: alerts on, security-update PRs off (vergil-tooling)

- `desired_security_settings()` returns `vulnerability_alerts=True` for **every**
  repo, public and private (Dependabot alerts do not need GHAS).
  `dependabot_security_updates` stays `"disabled"`.
- No new mechanism: the existing nightly `vrg-github-repo-config audit` reports
  any repo whose alerts are off, or whose security-update PRs are on, as
  non-compliant (exit 1, failing the nightly run). The human's existing
  `vrg-github-repo-config apply --repo <r>` turns them on.
- Update the existing tests that assert `vulnerability_alerts is False`.

### 5.2 The alert audit command (vergil-tooling)

A new console script, `vrg-dependabot-audit` (the name is the builder's call
within fleet `vrg-*` conventions):

- `vrg-dependabot-audit --repo OWNER/REPO` lists **open** Dependabot alerts via
  `GET /repos/{repo}/dependabot/alerts?state=open`, paginating through all pages
  through the existing `github.py` wrappers.
- **Exit codes**, matching `vrg-github-repo-config` (exit 1 = non-compliant,
  exit 2 = could not complete):
  - `0`: no open alerts. It prints a one-line "clean" summary.
  - `1`: one or more open alerts, at **any severity** (no threshold, no
    severity filter). The human decided "fail on anything, no nits."
  - `2`: the audit could not complete: alerts disabled on the repo, permission
    denied (403), repo not found, or any other API/auth error. It prints a clean
    diagnostic that names the cause, e.g. "Dependabot alerts are disabled for
    <repo>" vs. "token lacks Dependabot alerts: Read". Never a pass (A2).
- **Report** on exit 1 depends on repo visibility, because Actions logs on a
  public repo are world-readable while Dependabot alerts are restricted to users
  with admin or security access:
  - **Private repo:** one line per alert, sorted by severity (critical → low),
    then package. Each line has severity, ecosystem/package, the vulnerable
    manifest path, GHSA ID, and the alert's `html_url`, plus a total count by
    severity. The run log tells a human *what to upgrade* without opening the
    Security tab.
  - **Public repo:** only the total count by severity, plus a link to the repo's
    Dependabot alerts page (`https://github.com/<repo>/security/dependabot`).
    Details stay behind GitHub's access control. The log must never carry
    package names, manifest paths, or advisory IDs for a public repo.
  - Visibility comes from the repo's API metadata at run time. If visibility
    cannot be determined, the command falls back to the public (counts-only)
    format.
- **Dismissal is the escape hatch.** Only `state=open` counts. A human who
  dismisses an alert on GitHub (false positive, not reachable, accepted risk)
  clears it from the audit. There is no in-repo suppression file.
- Help text and a reference page follow the existing conventions for
  `vrg-*` tools.

### 5.3 The nightly job (vergil-actions)

Add a `dependabot-audit` job to `.github/workflows/ops-github-config.yml`,
modelled on the existing `branch-sweep` / `github-config` jobs:

- Same container (`ghcr.io/vergil-project/<prefix>-base:latest`), same
  checkout + `./actions/shared/setup/vergil` install.
- Mint an App token **scoped to the calling repo only**, with the narrowest
  permission: `permission-vulnerability-alerts: read` on
  `actions/create-github-app-token`. Follow the `branch-sweep` precedent and its
  comment explaining why `GITHUB_TOKEN` cannot be used. (*Builder: confirm the
  exact input name against the action's docs.*)
- Run `vrg-dependabot-audit --repo "${{ github.repository }}"`. A non-zero exit
  fails the job and the scheduled run. GitHub's standard failed-workflow
  notification delivers the email.
- Consumer `ops.yml` files are **unchanged**. Their caller-level
  `permissions:` (`contents: read`, `issues: write`) are untouched, because the
  job uses the App token, not `GITHUB_TOKEN`.
- Following the repo's convention, the job may be a composite action under
  `actions/ops/github-config/` (like `branch-sweep`) or an inline step (like
  `github-config`): builder's call.

### 5.4 `.github/dependabot.yml` (vergil-tooling)

**Content policy.** `version: 2`, with one `updates` entry per ecosystem. Every
entry carries `open-pull-requests-limit: 0` and a `schedule.interval` (required
key; `weekly` is fine since it opens nothing). The ecosystems:

- `github-actions` (directory `/`): **always**, for every repo, including repos
  with no primary language (`.github`, `docs`, `packages`).
- The primary language's ecosystem, where Dependabot supports it. Derive it from
  `vergil.toml` `[project].primary-language`: python → `uv`, go → `gomod`,
  rust → `cargo`, java → `maven` (or `gradle`, by build file),
  typescript → `npm`, cpp → none (Conan is unsupported). *Builder: verify each
  ecosystem key against the current Dependabot docs; if `uv` is not accepted,
  use `pip`.*

With `limit: 0` these entries are inert. They exist for Scorecard (F1), not for
behaviour. Alerts come from the dependency graph either way.

**One renderer, three consumers.** A single function renders the canonical file
from the repo's `vergil.toml`, used by:

1. **Scaffolding.** `vrg-repo-init` (`lib/repo_init.py`, which already renders
   `ops.yml` and friends) writes `.github/dependabot.yml` for new repos (G4).
2. **Fleet rollout.** A second applicator on the fleet-sweep seam
   (`lib/fleet_sweep.py`). The rule is **exact match**: the canonical render is
   the only allowed content, and the applicator overwrites. Local additions are
   not supported. The entries are inert, so extra ones buy nothing, and merging
   is deferred until a real need appears (human decision, YAGNI). Each repo that
   lacks the file, or whose file differs from the canonical one, gets a tracking issue under this epic, a branch, a
   commit, and a `report-ready` handoff. The human submits each PR. The CLI shape
   (a new `vrg-*` entry like `vrg-fleet-sync`, or a mode of it) is the builder's
   call. Note that `fleet_sweep.py` names a second consumer as follow-on #328's
   territory; this epic adds exactly one hard-coded applicator, not
   config-driven discovery.
3. **Guards** (below).

**Guards** (A1: no one quietly re-enables PRs):

- **PR-time, `vrg-validate` common check.** If `.github/dependabot.yml` exists,
  every `updates` entry must have `open-pull-requests-limit: 0`. A missing key
  fails too, because Dependabot's default is 5. This runs locally and in CI for
  every repo. It does **not** require the file to exist, so releasing it cannot
  break validation in repos the sweep has not reached yet.
- **Nightly, the local repo-config audit.** A `_check_dependabot_yml` in
  `lib/repo_config.py`'s `audit_local_config()` reads the file from the nightly
  job's checkout. It reports the repo as non-compliant if
  `.github/dependabot.yml` is **missing or differs from the canonical render**
  (exact match), which covers the limit-0 rule and catches a file edited outside
  the PR flow. This mirrors `_check_gitignore`: `repo_init` and the audit check
  render through **one shared function**, so scaffolding and audit cannot
  disagree. No contents-API read. This check ships in **release B**, after the
  sweep has landed (§6), never before.

### 5.5 GitHub App permission (human-gated deployment)

The App behind each org's `APP_CLIENT_ID` / `APP_PRIVATE_KEY` must gain
**Dependabot alerts: Read** (F5), and each installation must **accept** the
changed permission. Org-owner actions; the agent guides, the human performs.
Tracked by **vergil-actions#999** (`deployment`, guided walk-through).

**Unverified assumption.** The human believes one GitHub App (their own
user-agent App) and one consistent configuration back `APP_CLIENT_ID` across the
whole fleet, but has not confirmed it. The vergil-actions release notes refer to
"all three orgs," each passing its own secrets. So #999 **starts with an
inventory**: list every account (org or personal) whose repos run `ops.yml`,
record which App each one's `APP_CLIENT_ID` belongs to, and only then add and
accept the permission on each distinct App and installation. A missed owner
would show up as exit 2 every night on its repos. The
existing apply path already needs **Administration: Write** to toggle alerts and
is assumed already granted, since `apply` manages that setting today.
*Builder: confirm that on the App during #999.*

## 6. Rollout order

`vergil-actions` is consumed at the floating `@v2.1` tag, so a vergil-actions
release reaches **every repo's next nightly run at once**. vergil-tooling is
installed fresh in the ops job. Sequencing therefore avoids a fleet-wide wall of
red that carries no information.

**Accepted red window (human decision).** The settings flip cannot be applied
ahead of its release, because today's code would actively turn alerts *off*. So
between the vergil-tooling release (step 1) and the fleet-wide `apply` (step 3),
every repo's nightly `vrg-github-repo-config audit` reports alerts-off drift.
This is accepted. It is kept short by running step 3 as a scripted, fleet-wide
apply on the **same day** as the release, before the next nightly window. There
is no staged release and no temporary dual-state acceptance.

1. **vergil-tooling release A:** settings flip (§5.1), `vrg-dependabot-audit`
   (§5.2), the renderer, scaffolding, the `vrg-validate` limit-0 guard, and the
   sweep applicator (§5.4). **Not** the nightly missing/non-canonical file
   check. Human-gated **release**.
2. **App permission:** vergil-actions#999 (human walk-through). It can run in
   parallel with step 1 and must finish **before** step 6.
3. **Apply settings fleet-wide, same day as release A:** first run
   `vrg-github-repo-config diff --repo <r>` across the fleet and review any drift
   **unrelated to alerts**. `apply` pushes *all* desired state, so unrelated
   pending changes must not land in bulk as a side effect. Then the human runs a
   scripted loop of `vrg-github-repo-config apply --repo <r>` (a `deployment`
   task; Administration-write is a human action) and confirms alerts are on.
4. **`dependabot.yml` sweep:** run the released applicator across the fleet; the
   human submits and merges every PR.
5. **vergil-tooling release B:** turn on the nightly missing/non-canonical
   `dependabot.yml` check (§5.4) **only after every sweep PR has merged**, so no
   repo goes red over a file it was never sent. Human-gated **release**.
6. **vergil-actions:** the `dependabot-audit` job (§5.3). Human-gated
   **release**. From the next nightly run, the fleet is live.
7. **Validation:** a `workflow_dispatch` of `ops.yml` on a sample of repos (one
   per org, plus one known to have alerts) confirms exit 0 / exit 1 behaviour
   and the absence of exit 2. Then confirm Scorecard's Dependency-Update-Tool
   result on one public repo.

**Expected first-night outcome.** Turning alerts on will surface **real
existing findings**. Repos with open alerts will fail every night until someone
upgrades or dismisses. That is the intended behaviour, not a rollout defect.
Triaging those findings is the separate response-process epic's business.
Already today: alerts were manually enabled on vergil-tooling on 2026-10-22, so
its nightly `vrg-github-repo-config audit` will report drift against the current
alerts-off desired state until step 1 ships.

## 7. Acceptance

- Every managed repo: alerts **on**, Dependabot security updates **off**, and a
  canonical `.github/dependabot.yml` with every entry at
  `open-pull-requests-limit: 0`, all confirmed by a clean nightly
  `vrg-github-repo-config audit`.
- The nightly ops run includes `dependabot-audit`. It fails with an itemized
  report on any open alert, fails loudly on any inability to check, and passes
  only when there are verifiably no open alerts. A public repo's run log
  carries only severity counts and the Security-tab link, never package names,
  manifest paths, or advisory IDs.
- No Dependabot-authored PR appears anywhere in the fleet.
- `vrg-repo-init` scaffolds the file for new repos.
- Scorecard's Dependency-Update-Tool check passes on a sampled public repo.

## 8. Builder's calls (left open on purpose)

- The command name and CLI shape for the audit and the sweep entry point.
- Composite action vs. inline step for the ops job.
- `maven` vs `gradle` detection for Java; final ecosystem keys per F-verified
  docs.
- Canonical ecosystem selection beyond `github-actions` + primary language. The
  §5.4 mapping is a reasonable default, not a human decision.
- The granularity of operational tasks: §6 steps 2, 3 and 7 are natural
  `deployment` / `validation` tasks, to be seeded at plan time.
