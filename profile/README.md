<p align="center">
  <img src="Vergil-Banner.png" alt="VERGIL" width="100%">
</p>

# VERGIL

**Validation Engine for Repository Governance, Integration & Lifecycle**

VERGIL is a developer platform that enforces engineering standards
across a fleet of repositories spanning five languages. Commits, PRs,
validation, releases, security scanning, container builds, and AI agent
behavior — all governed by one system, mechanically enforced, not
documented and hoped for.

---

## Why this exists

Most multi-repo engineering organizations enforce standards through
wikis, onboarding docs, and code review. It works until it doesn't:
someone skips the linter, a release goes out without a security scan,
an AI coding agent auto-merges a PR it shouldn't have touched.

VERGIL replaces "please follow the process" with "the process won't
let you skip it." Every standard is enforced at the tooling layer —
pre-commit hooks, CI gates, container-based validation, and AI agent
guardrails that intercept non-compliant actions before they land.

---

## Architecture

Four components, each a separate repository with its own release
lifecycle:

```
┌──────────────────────────────────────────────────────────────────────────────────────────┐
│                                          VERGIL                                          │
│                                                                                          │
│            ┌──────────────────────────────────────────────────────────────┐              │
│            │  vergil-tooling                                              │              │
│            │                                                              │              │
│            │  Python CLI tools + git hooks — the core standards           │              │
│            │  enforcement engine                                          │              │
│            └┬───────────────────────┬────────────────────────────────────┬┘              │
│             │                       │                            │                       │
│             ▼                       ▼                            ▼                       │
│  ┌────────────────────┐  ┌────────────────────┐  ┌──────────────────────────────┐        │
│  │  vergil-actions    │  │  vergil-docker     │  │  vergil-claude-plugin        │        │
│  │                    │  │                    │  │                              │        │
│  │  CI/CD workflows   │  │  Dev container     │  │  Claude Code plugin — AI     │        │
│  │  & GitHub Actions  │  │  images (5+ lang)  │  │  agent workflow governance   │        │
│  └────────────────────┘  └────────────────────┘  └──────────────────────────────┘        │
└──────────────────────────────────────────────────────────────────────────────────────────┘
```

**vergil-tooling** — 15 Python CLI tools handling commits, PRs,
validation, releases, and container orchestration. The core of the
system. Every tool enforces a standard: `vrg-commit` validates
conventional commit format and branch policy; `vrg-validate` runs
language-specific linters, type checkers, and tests inside dev
containers; `vrg-prepare-release` drives semantic versioning with
changelog generation. 3,300+ lines of production Python, strict typing,
100% test coverage target.

**vergil-actions** — 13 reusable GitHub Actions and 10 CI/CD workflows.
Pre-merge gates (lint, typecheck, test, security scan, version
divergence) and post-merge delivery (tag, release, publish, docs
deploy). Consuming repos get consistent CI with a one-line workflow
reference. Security scanning via CodeQL, Semgrep, and Trivy with SARIF
upload. SLSA build provenance attestation on all published artifacts.

**vergil-docker** — Multi-language dev container images for Python
(3.12–3.14), Go (1.25–1.26), Ruby (3.2–3.4), Java (17, 21), and Rust
(1.92–1.93). Every image includes a shared tooling layer (ShellCheck,
actionlint, markdownlint, git-cliff, GitHub CLI, and more). Built for
amd64 and arm64, scanned with Trivy on both platforms, published to
GHCR. Local development and CI run in the same containers — what passes
locally passes in CI.

**vergil-claude-plugin** — A Claude Code plugin that makes AI coding
agents follow the same rules as human developers. 9 hooks intercept
non-compliant tool calls in real time: raw `git commit` is redirected to
the standards-compliant `vrg-commit`; raw `gh pr create` is redirected
to `vrg-submit-pr`; auto-merge is blocked; PR auto-close keywords are
blocked in favor of explicit issue management. 8 workflow skills guide
agents through complex multi-step operations (PR submission, release
publishing, dependency updates) without cutting corners.

---

## What makes this different

**AI agent governance.** Most AI coding tools focus on making agents
more capable. VERGIL focuses on making them more controllable. The
plugin layer intercepts agent actions at the tool-call boundary and
enforces the same engineering standards that apply to human developers.
This isn't theoretical — during a fleet-wide rollout, we discovered
that Claude Code subagents bypass sandbox path restrictions 53% of the
time by creatively working around denials. VERGIL's hook system is the
response: enforce policy at the workflow layer because you cannot rely
on the sandbox layer alone.

**Standards as code, not documentation.** Every engineering standard in
the system is mechanically enforced. Commit message format, branch
naming, PR linkage policy, pre-merge validation, security scanning,
release process — none of these depend on someone reading a wiki and
choosing to comply. The tools make non-compliance harder than
compliance.

**One system, many languages.** The same validation pipeline, the same
CI workflows, the same container-based execution model, the same
release process — whether the repository is Python, Go, Ruby, Java, or
Rust. Language-specific tooling (linters, type checkers, test runners)
plugs into a shared framework that handles everything else. The
architecture is designed to be extended to additional languages —
adding a language means a new container image, a new validation
command, and wiring into the existing pipeline. The framework does the
rest.

---

## By the numbers

| Metric | Count |
|--------|-------|
| CLI tools | 15 |
| GitHub Actions (composite) | 13 |
| CI/CD workflows (reusable) | 10 |
| Container images | 6 (across 5 languages) |
| AI agent hooks | 9 |
| Workflow skills | 8 |
| Languages supported | Python, Go, Ruby, Java, Rust (extensible) |
| Combined commits (4 repos) | 1,900+ |
| Test coverage target | 100% |

---

## Current state

VERGIL is in active development at v2.x across all four component
repositories. The system is functional and stable — it runs real CI/CD,
enforces real standards, and governs real AI agent sessions daily.

---

## Documentation

Full documentation is available at the VERGIL docs site:
**[vergil-tooling docs](https://vergil-project.github.io/vergil-tooling/)**

---

## Components

| Repository | Purpose |
|------------|---------|
| [vergil-tooling](https://github.com/vergil-project/vergil-tooling) | Python CLI tools, git hooks, core validation |
| [vergil-actions](https://github.com/vergil-project/vergil-actions) | Reusable GitHub Actions and CI/CD workflows |
| [vergil-docker](https://github.com/vergil-project/vergil-docker) | Multi-language dev container images |
| [vergil-claude-plugin](https://github.com/vergil-project/vergil-claude-plugin) | Claude Code plugin — AI agent workflow governance |

---

## Contributing

Contributions are welcome. See the
[contributing guidelines](https://github.com/vergil-project/.github/blob/develop/CONTRIBUTING.md)
for development setup, workflow, and the AI contributor identity model.

---

## Author

Built by [Phillip Moore](https://github.com/wphillipmoore) — 35 years
of infrastructure engineering, currently working as an AI-assisted
engineer. VERGIL is the system I built to solve the problem I kept
hitting: engineering standards that exist in documentation but not in
practice. More at
[The Infrastructure Mindset](https://the-infrastructure-mindset.ghost.io).
