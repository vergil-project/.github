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
a new contributor doesn't know the process exists.

AI coding agents made this worse. They don't read wikis. They bypass
sandboxes — during a fleet-wide rollout, we found that Claude Code
subagents circumvented sandbox path restrictions
[53% of the time](https://github.com/vergil-project/vergil-claude-plugin/issues/241)
by creatively working around denials. They auto-merge PRs they
shouldn't touch. They hallucinate compliance with standards they've
never seen.

VERGIL replaces "please follow the process" with "the process won't
let you skip it." Every standard is enforced at the tooling layer —
pre-commit hooks, CI gates, container-based validation, and agent
guardrails that intercept non-compliant actions before they land.

---

## How it works

**Standards as code.** Every engineering standard is mechanically
enforced. Commit format, branch naming, PR linkage, pre-merge
validation, security scanning, release process — none depend on
someone reading a wiki and choosing to comply. The tools make
non-compliance harder than compliance.

**One system, many languages.** The same validation pipeline, CI
workflows, and container-based execution model — whether the
repository is Python, Go, Ruby, Java, or Rust. Adding a language
means a new container image and a new validation command; the
framework does the rest.

**Container-based validation.** What passes locally passes in CI.
Same containers, same tools, same results. No more "works on my
machine."

**AI agent governance.** A plugin layer intercepts agent actions at
the tool-call boundary and enforces the same rules that apply to
human developers — mechanically, not by asking nicely.

| Repository | Purpose |
|---|---|
| [vergil-tooling](https://github.com/vergil-project/vergil-tooling) | Python CLI tools, git hooks, validation engine |
| [vergil-actions](https://github.com/vergil-project/vergil-actions) | Reusable GitHub Actions and CI/CD workflows |
| [vergil-docker](https://github.com/vergil-project/vergil-docker) | Multi-language dev container images |
| [vergil-claude-plugin](https://github.com/vergil-project/vergil-claude-plugin) | Claude Code plugin — AI agent governance |

Full architecture details at the
**[VERGIL docs site](https://vergil-project.github.io/vergil-tooling/)**.

---

## The adversary

Every guardrail needs an adversary. VERGIL builds the walls;
[Mimir](https://github.com/vergils-nemesis) tries to break them.

Mimir is VERGIL's adversarial testing identity — a hostile outsider
that exercises every denied path in the permission model. It attempts
commits that violate branch protection, submits PRs that skip required
checks, pushes directly to protected branches, and generally does
everything an AI agent shouldn't. If the tooling catches it, the
guardrail works. If it doesn't, we found a bug before someone else
found an exploit.

Confidence without adversarial testing is just optimism with better
marketing. More on the Vergil/Mimir duality at
[The Infrastructure Mindset](https://the-infrastructure-mindset.ghost.io).

---

## Documentation

Full documentation is available at the VERGIL docs site:
**[vergil-tooling docs](https://vergil-project.github.io/vergil-tooling/)**

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
