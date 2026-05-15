# `diogenes-project/.github` Profile Repository — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> superpowers:subagent-driven-development (recommended) or
> superpowers:executing-plans to implement this plan task-by-task. Steps
> use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Create the `diogenes-project/.github` repository with the org
profile README, default community health files, and VERGIL tooling
harness. Then consolidate duplicated templates out of the existing
diogenes repo.

**Architecture:** Two-phase approach — `.github` repo creation with all
content, then consolidation (remove per-repo duplicates after verifying
inheritance).

**Tech Stack:** GitHub API via `gh` CLI, markdown, YAML

**Reference:** This plan is adapted from the
`vergil-project/.github` implementation plan
(`vergil-tooling/docs/plans/2026-05-14-github-profile-repo.md`).

**Project context:** diogenes-project is a single-repo org containing
Diogenes, a deterministic AI research coordinator combining nine
intelligence and scientific frameworks into an evidence-based process.
Available as a Claude Code plugin or standalone prompt. Python library,
GPL-3 licensed, uses VERGIL tooling.

---

## Phase 1: Prerequisites

### Task 0: Enable GitHub Discussions

**Files:** None (GitHub API / web UI)

SUPPORT.md will reference GitHub Discussions as the channel for
questions and general conversation. Discussions must be enabled before
SUPPORT.md goes live.

- [ ] Enable Discussions on `diogenes-project/diogenes` via
      `gh repo edit diogenes-project/diogenes --enable-discussions`
- [ ] Verify: visit the repo's Discussions tab in browser to confirm
      it is active

---

## Phase 2: Create the `.github` Repository

All content is created in a single repo. Tasks are ordered by
dependency.

### Task 1: Create the Repository

**Files:** None (GitHub API)

- [ ] Create `diogenes-project/.github` as a **public** repository
      via `gh repo create diogenes-project/.github --public --description "Org-level configuration, community health files, and profile for the diogenes-project organization"`
- [ ] Clone locally to `/Users/pmoore/dev/projects/diogenes-project/.github`
- [ ] Initialize with `develop` as the default branch

### Task 2: VERGIL Tooling Harness

**Files:** `vergil.toml`, `.githooks/pre-commit`, `.claude/settings.json`, `CLAUDE.md`

Minimal tooling setup so the repo follows the same development
workflow as the diogenes repo.

- [ ] Create `vergil.toml`:

      ```toml
      [project]
      repository-type = "documentation"
      versioning-scheme = "semver"
      branching-model = "library-release"
      release-model = "tagged-release"
      primary-language = "none"

      [project.co-authors]
      agent = "Co-Authored-By: wphillipmoore-agent <284101533+wphillipmoore-agent@users.noreply.github.com>"
      ```

- [ ] Copy `.githooks/pre-commit` from vergil-project/.github
      (identical across all VERGIL-managed repos)
- [ ] Create `.claude/settings.json` with vergil plugin marketplace
      config (identical to diogenes repo)
- [ ] Create `CLAUDE.md` — minimal agent guidance: docs-only repo,
      no Python, use `vrg-commit`, validation via
      `vrg-docker-run -- uv run vrg-validate`
- [ ] Create `LICENSE` — GPL-3 (matching the diogenes repo)
- [ ] Verify: `git config core.hooksPath .githooks` works, raw
      `git commit` is rejected, `vrg-commit` works

### Task 3: Issue Templates and PR Template

**Files:** `ISSUE_TEMPLATE/issue.yml`, `ISSUE_TEMPLATE/config.yml`, `pull_request_template.md`

Migrated from the diogenes repo's per-repo copies, which are identical
to the VERGIL standard templates.

- [ ] Copy `ISSUE_TEMPLATE/issue.yml` from diogenes repo
- [ ] Copy `ISSUE_TEMPLATE/config.yml` from diogenes repo
- [ ] Copy `pull_request_template.md` from diogenes repo
- [ ] Verify: content is byte-identical to the per-repo versions

### Task 4: CODE_OF_CONDUCT.md

**Files:** `CODE_OF_CONDUCT.md`

- [ ] Copy from `vergil-project/.github/CODE_OF_CONDUCT.md`
      (Contributor Covenant v2.1, already written)
- [ ] Update enforcement contact if needed (should be the same
      maintainer)
- [ ] Review: ensure the text is unmodified except for any
      project-specific enforcement contact changes

### Task 5: SECURITY.md

**Files:** `SECURITY.md`

- [ ] Adapt from `vergil-project/.github/SECURITY.md`
- [ ] Update scope: the Diogenes Claude Code plugin, the standalone
      prompt system, the MCP server, the dio CLI, and the research
      methodology framework definitions
- [ ] Update out-of-scope: vulnerabilities in upstream dependencies
      (Claude API, anthropic SDK, etc.)
- [ ] Keep same response commitment timelines (7-day acknowledge,
      30-day fix/mitigation target)
- [ ] Enable private vulnerability reporting on the org if not
      already enabled

### Task 6: CONTRIBUTING.md

**Files:** `CONTRIBUTING.md`

Adapted from the vergil-project version but scoped to a single-repo
project.

- [ ] Write "How the project works" section — single repo containing
      the plugin, standalone prompt, MCP server, and CLI
- [ ] Write "Development setup" section — prerequisites (Docker, uv,
      Python 3.14), installing vergil-tooling, enabling git hooks
- [ ] Write "Workflow" section — branch from `develop`, use
      `vrg-commit`, use `vrg-submit-pr`, validate via
      `vrg-docker-run -- uv run vrg-validate`
- [ ] Write "For contributors using AI tools" section — identity
      model (`<username>-agent`), accountability principle
- [ ] Write "For contributors not using AI" section — fork/branch/PR,
      same quality bar
- [ ] Write "What to expect from review" section — human approval
      required, CI must pass
- [ ] Write "Template override convention" section — document the
      all-or-nothing inheritance rule for template directories
- [ ] Review: ensure consistency with the vergil-project
      CONTRIBUTING.md and current tooling behavior

### Task 7: SUPPORT.md

**Files:** `SUPPORT.md`

- [ ] Adapt from `vergil-project/.github/SUPPORT.md`
- [ ] Support channels: GitHub Issues for bugs and feature requests,
      GitHub Discussions for questions and conversation
- [ ] Link to Diogenes README for reference material (no separate
      docs site currently)
- [ ] State clearly: community project, no SLA-backed support
- [ ] Verify: Discussions links point to working URLs (depends on
      Task 0 being complete)

### Task 8: Root README.md

**Files:** `README.md`

Short description of what the `.github` repo is, not the org profile.

- [ ] Write one-paragraph description: this repo holds org-level
      default configuration for the diogenes-project organization
- [ ] Explain which files live here and how GitHub's default community
      health file inheritance works
- [ ] Document the template override convention (all-or-nothing per
      directory)
- [ ] Link to the profile README (`profile/README.md`) and
      CONTRIBUTING.md
- [ ] Note the `.github/.github/workflows/` nesting and why it exists

### Task 9: Org Profile README

**Files:** `profile/README.md`

This is the public-facing org profile page. Content should be written
fresh — Diogenes has its own identity distinct from VERGIL.

- [ ] Write a concise description of the Diogenes project — what it
      is, what problem it solves, who it's for
- [ ] Highlight the nine frameworks (ICD 203, GRADE, PRISMA,
      Cochrane, CONSORT, IPCC, NAS, ROBIS, Chamberlin/Platt)
- [ ] Three usage modes: Claude Code plugin, standalone prompt, CLI
- [ ] Link to the diogenes repo as the primary entry point
- [ ] Add link to `CONTRIBUTING.md` as a call to action
- [ ] Review: verify all URLs resolve, description is accurate

### Task 10: CI Workflow

**Files:** `.github/workflows/ci.yml`

Minimal CI — this is a docs-only repo.

- [ ] Write `ci.yml` using vergil-actions reusable workflows with
      `primary-language = "none"` (common checks only: markdownlint,
      yamllint, shellcheck, actionlint)
- [ ] Reference the pattern from `vergil-project/.github`'s own
      CI workflow
- [ ] Verify: push a test branch and confirm CI runs without errors

---

## Phase 3: Consolidation

After the `.github` repo is live and inheritance is verified, remove
the duplicated files from the diogenes repo.

### Task 11: Verify Inheritance

**Files:** None (manual testing)

- [ ] Navigate to diogenes-project/diogenes → Issues → New issue.
      Confirm the org-level issue template appears (per-repo copy
      still exists, so org default won't show yet — this is expected)
- [ ] Navigate to Insights → Community Standards. Confirm
      CONTRIBUTING.md, CODE_OF_CONDUCT.md, SECURITY.md, and
      SUPPORT.md appear as inherited
- [ ] Temporarily rename the diogenes repo's `ISSUE_TEMPLATE/`
      directory and confirm the org default takes over, then rename
      it back

### Task 12: Remove Per-Repo Duplicates

**Files:** Issue templates and PR template in the diogenes repo

Single PR on the diogenes repo.

- [ ] Remove `.github/ISSUE_TEMPLATE/issue.yml`,
      `.github/ISSUE_TEMPLATE/config.yml`,
      `.github/pull_request_template.md` from diogenes repo
- [ ] After merging, verify inheritance: create a test issue and
      confirm the org template appears

### Task 13: Post-Consolidation Verification

**Files:** None (manual testing)

- [ ] Create a real issue on the diogenes repo using the inherited
      template — confirm all fields work
- [ ] Confirm blank issues are still disabled (config.yml inheritance)
- [ ] Open a PR on the diogenes repo — confirm the PR template
      appears
- [ ] Visit the org profile page — confirm `profile/README.md`
      renders correctly
- [ ] Spot-check community health files: visit the repo's
      "Community Standards" page and confirm all files show as present
