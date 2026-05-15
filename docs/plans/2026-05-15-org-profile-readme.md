# Org Profile README Rewrite — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> superpowers:subagent-driven-development (recommended) or
> superpowers:executing-plans to implement this plan task-by-task.
> Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Rewrite the vergil-project org profile README from a
technical spec sheet into a problem-first narrative with the
Vergil/Mimir adversarial duality woven throughout.

**Architecture:** Single-file rewrite of `profile/README.md`.
No code changes, no new files. Content follows the design spec
at `docs/specs/2026-05-15-org-profile-readme-design.md`.

**Tech Stack:** Markdown

**Worktree:**
`/Users/pmoore/dev/projects/vergil-project/.github/.worktrees/issue-821-org-profile-readme/`

**Branch:** `feature/821-org-profile-readme`

---

### Task 1: Write Section 1 — Banner + Opening

**Files:**
- Modify: `profile/README.md`

- [ ] **Step 1: Read the current README**

Read `profile/README.md` in its entirety. Note the banner
markup and opening paragraph structure.

- [ ] **Step 2: Replace the opening paragraph**

Keep the banner image markup and title exactly as-is:

```markdown
<p align="center">
  <img src="Vergil-Banner.png" alt="VERGIL" width="100%">
</p>

# VERGIL

**Validation Engine for Repository Governance, Integration & Lifecycle**
```

Replace the current opening paragraph with one that frames
VERGIL broadly — a developer platform for enforcing engineering
standards across a fleet of repositories. AI agent governance
is a primary focus, but the tooling is generally useful for
repository governance. End with the "mechanically enforced, not
documented and hoped for" line.

Delete everything below the opening paragraph. The remaining
tasks will rebuild the content section by section.

- [ ] **Step 3: Verify the banner renders**

Confirm the image tag references `Vergil-Banner.png` and that
the file exists at `profile/Vergil-Banner.png`.

```bash
ls profile/Vergil-Banner.png
```

Expected: file exists.

---

### Task 2: Write Section 2 — Why This Exists

**Files:**
- Modify: `profile/README.md`

- [ ] **Step 1: Write the "Why this exists" section**

Append a horizontal rule and the section after the opening
paragraph. Three beats:

1. **Standards decay** — wikis, onboarding docs, code review.
   Works until someone skips the linter, a release ships
   without a security scan, a new contributor doesn't know the
   process exists.

2. **AI makes it urgent** — AI coding agents don't read wikis.
   They bypass sandboxes, auto-merge things they shouldn't,
   hallucinate compliance. Ground with the concrete stat:
   during fleet-wide rollout, Claude Code subagents bypassed
   sandbox path restrictions 53% of the time
   ([vergil-claude-plugin#241](https://github.com/vergil-project/vergil-claude-plugin/issues/241)).
   That finding is what triggered the hook-based enforcement
   system.

3. **VERGIL's thesis** — replace "please follow the process"
   with "the process won't let you skip it."

Frame as a problem for all engineering (humans skip linters
too), with AI as the accelerant. Do not position VERGIL as
exclusively for AI-assisted development.

---

### Task 3: Write Section 3 — How It Works

**Files:**
- Modify: `profile/README.md`

- [ ] **Step 1: Write architectural highlights**

Append a horizontal rule and the section. Four short
paragraphs, each with a bold lead:

- **Standards as code.** Every standard is mechanically
  enforced — commit format, branch naming, PR linkage,
  pre-merge validation, security scanning, release process.
  None depend on someone choosing to comply.

- **One system, many languages.** Same validation pipeline,
  CI workflows, container-based execution — Python, Go, Ruby,
  Java, Rust. Extensible to additional languages.

- **Container-based validation.** What passes locally passes
  in CI. Same containers, same tools, same results.

- **AI agent governance.** A plugin layer intercepts agent
  actions at the tool-call boundary. The same rules that apply
  to human developers apply to AI agents — mechanically, not
  by asking nicely.

- [ ] **Step 2: Write the component table**

After the highlights, add a compact component table:

```markdown
| Repository | Purpose |
|------------|---------|
| [vergil-tooling](...) | Python CLI tools, git hooks, validation engine |
| [vergil-actions](...) | Reusable GitHub Actions and CI/CD workflows |
| [vergil-docker](...) | Multi-language dev container images |
| [vergil-claude-plugin](...) | Claude Code plugin — AI agent governance |
```

Use full GitHub URLs for the repository links. Follow with a
single line linking to the docs site for full architecture
details.

---

### Task 4: Write Section 4 — The Adversary

**Files:**
- Modify: `profile/README.md`

- [ ] **Step 1: Write the adversary section**

Append a horizontal rule and the section. This is the
signature section — tone shifts slightly, with personality.

Core content:

- Every guardrail needs an adversary. VERGIL builds the
  walls; [Mimir](https://github.com/vergils-nemesis) tries
  to break them.
- The adversarial testing identity exercises every denied
  path in the permission model. If the tooling catches it,
  the guardrail works. If not, we found a bug before someone
  else found an exploit.
- Philosophy line: confidence without adversarial testing is
  just optimism with better marketing.
- Link to the
  [Infrastructure Mindset blog](https://the-infrastructure-mindset.ghost.io)
  (placeholder — the specific article introducing the
  Vergil/Mimir concept is being written; link to the blog
  landing page for now).
- Link to the
  [vergils-nemesis org](https://github.com/vergils-nemesis).

Keep to one or two paragraphs plus a closing line. Vergil
*wants* Mimir to try — that's the whole point.

---

### Task 5: Write Sections 5-7 — Documentation, Contributing, Author

**Files:**
- Modify: `profile/README.md`

- [ ] **Step 1: Write the Documentation section**

Append a horizontal rule and:

```markdown
## Documentation

Full documentation is available at the VERGIL docs site:
**[vergil-tooling docs](https://vergil-project.github.io/vergil-tooling/)**
```

- [ ] **Step 2: Write the Contributing section**

Append a horizontal rule and a brief contributing paragraph
linking to the
[contributing guidelines](https://github.com/vergil-project/.github/blob/develop/CONTRIBUTING.md).

- [ ] **Step 3: Write the Author section**

Append a horizontal rule and the author section. Keep the
current two-sentence structure:

```markdown
## Author

Built by [Phillip Moore](https://github.com/wphillipmoore) — 35 years
of infrastructure engineering, currently working as an AI-assisted
engineer. VERGIL is the system I built to solve the problem I kept
hitting: engineering standards that exist in documentation but not in
practice. More at
[The Infrastructure Mindset](https://the-infrastructure-mindset.ghost.io).
```

This establishes credibility and closes the page with the same
seriousness the 53% stat opens it with.

---

### Task 6: Review and validate

**Files:**
- Read: `profile/README.md`

- [ ] **Step 1: Read the complete README**

Read `profile/README.md` top to bottom. Verify:

- Banner + title + acronym preserved
- Opening paragraph is broad (not AI-only)
- "Why this exists" has all three beats with the 53% stat
- "How it works" has four highlights + component table +
  docs link
- "The adversary" introduces Mimir with personality, links
  to vergils-nemesis and Infrastructure Mindset
- Documentation, Contributing, Author sections present
- No ASCII diagram, no metrics table, no "by the numbers"
- Horizontal rules between sections

- [ ] **Step 2: Run validation**

```bash
vrg-docker-run -- uv run vrg-validate
```

Expected: all checks pass (markdownlint, yamllint,
shellcheck, actionlint).

- [ ] **Step 3: Fix any validation failures**

If markdownlint or other checks flag issues, fix them and
re-run until clean.

- [ ] **Step 4: Commit**

```bash
vrg-git add profile/README.md
vrg-commit --type docs --scope profile \
  --message "rewrite org profile README" \
  --body "Problem-first narrative with Vergil/Mimir duality. Replaces technical spec sheet with marketing-facing landing page." \
  --agent wphillipmoore-agent
```

---

### Task 7: Submit PR

- [ ] **Step 1: Submit the PR**

Use the `vergil:pr-workflow` skill. The PR addresses
vergil-project/vergil-tooling#821 (cross-repo issue).

```bash
vrg-submit-pr \
  --issue 821 \
  --summary "Rewrite org profile README from technical inventory to problem-first narrative with Vergil/Mimir duality" \
  --linkage Ref \
  --title "docs(profile): rewrite org profile README" \
  --notes "Replaces the spec-sheet landing page with a narrative: why VERGIL exists, how it works (highlights only), and the adversarial testing philosophy. Cuts the ASCII diagram, metrics table, and detailed component descriptions in favor of a compact table and links to the docs site."
```

- [ ] **Step 2: Wait for CI green**

```bash
vrg-wait-until-green <pr-url>
```

- [ ] **Step 3: Hand off to user for review**
