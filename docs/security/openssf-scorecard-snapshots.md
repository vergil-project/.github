# OpenSSF Scorecard snapshots

Point-in-time [OpenSSF Scorecard](https://github.com/ossf/scorecard) captures for
this repository. A Scorecard result is a report of security posture at one
commit, not an open work item — so we capture it here as dated snapshots and
re-run it on a cadence, rather than accreting findings in the issue tracker.

**Remediation is tracked separately.** This document is the *record*, not the
work. The actual OpenSSF hardening work lives in epic
[vergil-project/.github#54](https://github.com/vergil-project/.github/issues/54),
tracking issue
[vergil-project/vergil-tooling#828](https://github.com/vergil-project/vergil-tooling/issues/828).

## How to add a snapshot

Re-run the Scorecard from a checkout of this repo and append a new dated section
at the top of [Snapshots](#snapshots) (newest first), then add a row to the
[trend table](#trend):

```bash
vrg-scorecard --repo=github.com/vergil-project/.github --show-details
```

Record, for each snapshot: the aggregate score, the evaluated commit SHA, the
Scorecard version, the per-check table, and the detailed findings. Cadence:
roughly quarterly, or after any deliberate hardening change worth measuring.

## Trend

| Date | Aggregate | Commit | Scorecard |
|------|-----------|--------|-----------|
| 2026-10-25 | 7.1 / 10 | `b5995462bc14` | v5.6.0 |
| 2026-05-19 | 5.2 / 10 | `091fd5015ce0` | v5.5.0 |

Score legend: 🟢 10/10 · 🟡 5–9/10 · 🔴 0–4/10 (needs work) · ⚪ -1/10 (not
applicable / not detected).

## Snapshots

### 2026-10-25

**Aggregate score:** 7.1 / 10
**Commit:** `b5995462bc14da7665c91519fb060fd86242e819`
**Scorecard version:** v5.6.0
**Tracking issue:**
[vergil-project/vergil-tooling#828](https://github.com/vergil-project/vergil-tooling/issues/828)

#### Changes since the 2026-05-19 baseline

The aggregate rose 5.2 → 7.1. The Scorecard version also moved from v5.5.0 to
v5.6.0, so part of any movement may reflect scoring changes upstream. Movement
by check:

- **Maintained** 0 → 10 — the repo is no longer "created within the last 90
  days"; 30 commits in the trailing 90 days now score it.
- **Dependency-Update-Tool** 0 → 10 — Dependabot is now configured
  (`.github/dependabot.yml`).
- **SAST** 6 → 10 — all 30 sampled commits are now checked with a SAST tool
  (was 20 of 30).
- **CI-Tests** 9 → 10 — 30 of 30 merged PRs checked by a CI test (was 9 of 10).
- **Code-Review** 0/30 (was 0/10) and **Pinned-Dependencies** 0 unpinned of 6
  (was 0 of 2) — same scores, larger sample / more workflows.

All other checks are unchanged from the baseline.

#### Scores by check

| Score | Check | Reason |
|-------|-------|--------|
| ⚪ -1/10 | Packaging | packaging workflow not detected |
| ⚪ -1/10 | Signed-Releases | no releases found |
| 🔴 0/10 | CII-Best-Practices | no effort to earn an OpenSSF best practices badge detected |
| 🔴 0/10 | Code-Review | Found 0/30 approved changesets |
| 🔴 0/10 | Contributors | project has 0 contributing companies or organizations |
| 🔴 0/10 | Fuzzing | project is not fuzzed |
| 🔴 0/10 | Pinned-Dependencies | dependency not pinned by hash detected |
| 🔴 4/10 | Branch-Protection | branch protection is not maximal on development and all release branches |
| 🟢 10/10 | Binary-Artifacts | no binaries found in the repo |
| 🟢 10/10 | CI-Tests | 30 out of 30 merged PRs checked by a CI test |
| 🟢 10/10 | Dangerous-Workflow | no dangerous workflow patterns detected |
| 🟢 10/10 | Dependency-Update-Tool | update tool detected |
| 🟢 10/10 | License | license file detected |
| 🟢 10/10 | Maintained | 30 commit(s) and 0 issue activity found in the last 90 days |
| 🟢 10/10 | SAST | SAST tool is run on all commits |
| 🟢 10/10 | Security-Policy | security policy file detected |
| 🟢 10/10 | Token-Permissions | GitHub workflow tokens follow principle of least privilege |
| 🟢 10/10 | Vulnerabilities | 0 existing vulnerabilities detected |

#### Detailed findings

##### Packaging (-1/10)

**Reason:** packaging workflow not detected

**Warnings:**

- `no GitHub/GitLab publishing workflow detected.`

**Documentation:** https://github.com/ossf/scorecard/blob/61e84743356275f6ffca23b1b2f2cb945e19eb96/docs/checks.md#packaging

##### Signed-Releases (-1/10)

**Reason:** no releases found

**Documentation:** https://github.com/ossf/scorecard/blob/61e84743356275f6ffca23b1b2f2cb945e19eb96/docs/checks.md#signed-releases

##### CII-Best-Practices (0/10)

**Reason:** no effort to earn an OpenSSF best practices badge detected

**Documentation:** https://github.com/ossf/scorecard/blob/61e84743356275f6ffca23b1b2f2cb945e19eb96/docs/checks.md#cii-best-practices

##### Code-Review (0/10)

**Reason:** Found 0/30 approved changesets -- score normalized to 0

**Documentation:** https://github.com/ossf/scorecard/blob/61e84743356275f6ffca23b1b2f2cb945e19eb96/docs/checks.md#code-review

##### Contributors (0/10)

**Reason:** project has 0 contributing companies or organizations -- score normalized to 0

**Documentation:** https://github.com/ossf/scorecard/blob/61e84743356275f6ffca23b1b2f2cb945e19eb96/docs/checks.md#contributors

##### Fuzzing (0/10)

**Reason:** project is not fuzzed

**Warnings:**

- `no fuzzer integrations found`

**Documentation:** https://github.com/ossf/scorecard/blob/61e84743356275f6ffca23b1b2f2cb945e19eb96/docs/checks.md#fuzzing

##### Pinned-Dependencies (0/10)

**Reason:** dependency not pinned by hash detected -- score normalized to 0

**Warnings:**

- `third-party GitHubAction not pinned by hash: .github/workflows/ci.yml:15`
- `third-party GitHubAction not pinned by hash: .github/workflows/ci.yml:21`
- `third-party GitHubAction not pinned by hash: .github/workflows/ci.yml:32`
- `third-party GitHubAction not pinned by hash: .github/workflows/epic-rollup.yml:24`
- `third-party GitHubAction not pinned by hash: .github/workflows/epic-sweep.yml:19`
- `third-party GitHubAction not pinned by hash: .github/workflows/ops.yml:14`

<details><summary>Info details</summary>

- `0 out of 6 third-party GitHubAction dependencies pinned`

Remediation helper (pin by hash): https://app.stepsecurity.io/securerepo

</details>

**Documentation:** https://github.com/ossf/scorecard/blob/61e84743356275f6ffca23b1b2f2cb945e19eb96/docs/checks.md#pinned-dependencies

##### Branch-Protection (4/10)

**Reason:** branch protection is not maximal on development and all release branches

**Warnings:**

- `branch 'develop' does not require approvers`
- `codeowners review is not required on branch 'develop'`
- `'last push approval' is disabled on branch 'develop'`

<details><summary>Info details (7 items)</summary>

- `'allow deletion' disabled on branch 'develop'`
- `'force pushes' disabled on branch 'develop'`
- `'branch protection settings apply to administrators' is required to merge on branch 'develop'`
- `'stale review dismissal' is required to merge on branch 'develop'`
- `'up-to-date branches' is required to merge on branch 'develop'`
- `status check found to merge onto on branch 'develop'`
- `PRs are required in order to make changes on branch 'develop'`

</details>

**Documentation:** https://github.com/ossf/scorecard/blob/61e84743356275f6ffca23b1b2f2cb945e19eb96/docs/checks.md#branch-protection

##### Token-Permissions (10/10)

**Reason:** GitHub workflow tokens follow principle of least privilege

**Warnings:**

- `jobLevel 'security-events' permission set to 'write': .github/workflows/ci.yml:24`

<details><summary>Info details (9 items)</summary>

- `topLevel 'contents' permission set to 'read'` in `ci.yml`, `epic-rollup.yml`,
  `epic-sweep.yml` and `ops.yml`
- `jobLevel 'contents' permission set to 'read'` in `ci.yml`, `epic-rollup.yml`,
  `epic-sweep.yml` and `ops.yml`
- `jobLevel 'actions' permission set to 'read': .github/workflows/ci.yml:25`

</details>

**Documentation:** https://github.com/ossf/scorecard/blob/61e84743356275f6ffca23b1b2f2cb945e19eb96/docs/checks.md#token-permissions

---

### 2026-05-19 (baseline)

**Aggregate score:** 5.2 / 10
**Commit:** `091fd5015ce0`
**Scorecard version:** v5.5.0
**Tracking issue:**
[vergil-project/vergil-tooling#828](https://github.com/vergil-project/vergil-tooling/issues/828)

> First captured snapshot, preserved verbatim from the tracking issue
> ([#20](https://github.com/vergil-project/.github/issues/20)); only heading
> levels were adjusted to fit this document.

#### Scores by check

| Score | Check | Reason |
|-------|-------|--------|
| ⚪ -1/10 | Packaging | packaging workflow not detected |
| ⚪ -1/10 | Signed-Releases | no releases found |
| 🔴 0/10 | CII-Best-Practices | no effort to earn an OpenSSF best practices badge detected |
| 🔴 0/10 | Code-Review | Found 0/10 approved changesets |
| 🔴 0/10 | Contributors | project has 0 contributing companies or organizations |
| 🔴 0/10 | Dependency-Update-Tool | no update tool detected |
| 🔴 0/10 | Fuzzing | project is not fuzzed |
| 🔴 0/10 | Maintained | project was created within the last 90 days. Please review its contents carefull |
| 🔴 0/10 | Pinned-Dependencies | dependency not pinned by hash detected |
| 🔴 4/10 | Branch-Protection | branch protection is not maximal on development and all release branches |
| 🟡 6/10 | SAST | SAST tool is not run on all commits |
| 🟡 9/10 | CI-Tests | 9 out of 10 merged PRs checked by a CI test |
| 🟢 10/10 | Binary-Artifacts | no binaries found in the repo |
| 🟢 10/10 | Dangerous-Workflow | no dangerous workflow patterns detected |
| 🟢 10/10 | License | license file detected |
| 🟢 10/10 | Security-Policy | security policy file detected |
| 🟢 10/10 | Token-Permissions | GitHub workflow tokens follow principle of least privilege |
| 🟢 10/10 | Vulnerabilities | 0 existing vulnerabilities detected |

#### Detailed findings

##### Packaging (-1/10)

**Reason:** packaging workflow not detected

**Warnings:**

- `no GitHub/GitLab publishing workflow detected.`

**Documentation:** https://github.com/ossf/scorecard/blob/c395761df6afe1a69e476bc60a013a94bcbc153f/docs/checks.md#packaging

##### Signed-Releases (-1/10)

**Reason:** no releases found

**Documentation:** https://github.com/ossf/scorecard/blob/c395761df6afe1a69e476bc60a013a94bcbc153f/docs/checks.md#signed-releases

##### CII-Best-Practices (0/10)

**Reason:** no effort to earn an OpenSSF best practices badge detected

**Documentation:** https://github.com/ossf/scorecard/blob/c395761df6afe1a69e476bc60a013a94bcbc153f/docs/checks.md#cii-best-practices

##### Code-Review (0/10)

**Reason:** Found 0/10 approved changesets -- score normalized to 0

**Documentation:** https://github.com/ossf/scorecard/blob/c395761df6afe1a69e476bc60a013a94bcbc153f/docs/checks.md#code-review

##### Contributors (0/10)

**Reason:** project has 0 contributing companies or organizations -- score normalized to 0

**Documentation:** https://github.com/ossf/scorecard/blob/c395761df6afe1a69e476bc60a013a94bcbc153f/docs/checks.md#contributors

##### Dependency-Update-Tool (0/10)

**Reason:** no update tool detected

**Warnings:**

- `no dependency update tool configurations found`

**Documentation:** https://github.com/ossf/scorecard/blob/c395761df6afe1a69e476bc60a013a94bcbc153f/docs/checks.md#dependency-update-tool

##### Fuzzing (0/10)

**Reason:** project is not fuzzed

**Warnings:**

- `no fuzzer integrations found`

**Documentation:** https://github.com/ossf/scorecard/blob/c395761df6afe1a69e476bc60a013a94bcbc153f/docs/checks.md#fuzzing

##### Maintained (0/10)

**Reason:** project was created within the last 90 days. Please review its contents carefully

**Warnings:**

- `Repository was created within the last 90 days.`

**Documentation:** https://github.com/ossf/scorecard/blob/c395761df6afe1a69e476bc60a013a94bcbc153f/docs/checks.md#maintained

##### Pinned-Dependencies (0/10)

**Reason:** dependency not pinned by hash detected -- score normalized to 0

**Warnings:**

- `third-party GitHubAction not pinned by hash: .github/workflows/ci.yml:22: update your workflow using https://app.stepsecurity.io/secureworkflow/vergil-project/.github/ci.yml/develop?enable=pin`
- `third-party GitHubAction not pinned by hash: .github/workflows/ci.yml:15: update your workflow using https://app.stepsecurity.io/secureworkflow/vergil-project/.github/ci.yml/develop?enable=pin`

<details><summary>Info details (1 items)</summary>

- `0 out of   2 third-party GitHubAction dependencies pinned`

</details>

**Documentation:** https://github.com/ossf/scorecard/blob/c395761df6afe1a69e476bc60a013a94bcbc153f/docs/checks.md#pinned-dependencies

##### Branch-Protection (4/10)

**Reason:** branch protection is not maximal on development and all release branches

**Warnings:**

- `branch 'develop' does not require approvers`
- `codeowners review is not required on branch 'develop'`
- `'last push approval' is disabled on branch 'develop'`

<details><summary>Info details (7 items)</summary>

- `'allow deletion' disabled on branch 'develop'`
- `'force pushes' disabled on branch 'develop'`
- `'branch protection settings apply to administrators' is required to merge on branch 'develop'`
- `'stale review dismissal' is required to merge on branch 'develop'`
- `'up-to-date branches' is required to merge on branch 'develop'`
- `status check found to merge onto on branch 'develop'`
- `PRs are required in order to make changes on branch 'develop'`

</details>

**Documentation:** https://github.com/ossf/scorecard/blob/c395761df6afe1a69e476bc60a013a94bcbc153f/docs/checks.md#branch-protection

##### SAST (6/10)

**Reason:** SAST tool is not run on all commits -- score normalized to 6

**Warnings:**

- `20 commits out of 30 are checked with a SAST tool`

**Documentation:** https://github.com/ossf/scorecard/blob/c395761df6afe1a69e476bc60a013a94bcbc153f/docs/checks.md#sast

##### CI-Tests (9/10)

**Reason:** 9 out of 10 merged PRs checked by a CI test -- score normalized to 9

**Documentation:** https://github.com/ossf/scorecard/blob/c395761df6afe1a69e476bc60a013a94bcbc153f/docs/checks.md#ci-tests
