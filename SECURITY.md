# Security Policy

## Reporting a Vulnerability

If you discover a security vulnerability in any VERGIL component,
please report it through
[GitHub's private vulnerability reporting](https://github.com/vergil-project/.github/security/advisories/new).
This ensures your report is handled confidentially.

If private vulnerability reporting is unavailable, email
**w.phillip.moore@gmail.com** with the subject line
"VERGIL Security Report". Do not open a public issue for security
vulnerabilities.

## How Reports Are Tracked

Security findings, including those found by maintainers, are tracked
internally as private draft security advisories in this repository.
Each finding is paired with a neutral `security`-labelled issue that
drives the fix. That issue holds no details of the problem, only a
pointer to the advisory. Maintainers never put finding details in
public issues, pull requests, or commit messages.

A report you send through private vulnerability reporting lands in
the same advisory list. Maintainers pair it with a neutral issue and
handle it from there, alongside internal findings. See the
[Security Records guide](https://vergil-project.github.io/vergil-tooling/latest/guides/security-records/)
for how records are kept.

## Scope

The following components are in scope for security reports:

- **vergil-tooling** — Python CLI tools (`vrg-commit`, `vrg-validate`,
  `vrg-submit-pr`, `vrg-release`, `vrg-container-run`, and others),
  git hooks, and shared libraries
- **vergil-containers** — Dev container images, Dockerfiles, and build
  scripts
- **vergil-actions** — Reusable GitHub Actions workflows and composite
  actions
- **vergil-claude-plugin** — Claude Code plugin hooks, skills, and
  configuration

## Out of Scope

- Vulnerabilities in upstream dependencies (report these to the
  upstream maintainer)
- Vulnerabilities in GitHub, Docker, or other third-party platforms
- Social engineering attacks against project contributors

## Response Commitment

- **Acknowledgment**: within 7 days of receiving a report
- **Assessment**: initial severity assessment within 14 days
- **Resolution**: target fix or mitigation plan within 30 days of
  acknowledgment, depending on severity and complexity

These timelines reflect the project's current scale as a small
community project. Response times may vary, but every report will be
acknowledged and investigated.

## Disclosure Policy

We follow coordinated disclosure. Once a fix is available, we will:

1. Release the fix across affected components
2. Publish a security advisory on GitHub
3. Credit the reporter (unless they request anonymity)

We ask that reporters allow reasonable time for a fix before public
disclosure.
