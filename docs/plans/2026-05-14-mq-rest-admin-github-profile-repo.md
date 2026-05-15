# `mq-rest-admin-project/.github` Profile Repository — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> superpowers:subagent-driven-development (recommended) or
> superpowers:executing-plans to implement this plan task-by-task. Steps
> use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Create the `mq-rest-admin-project/.github` repository with the
org profile README, default community health files, and VERGIL tooling
harness. Then consolidate duplicated templates out of the seven active
repos.

**Architecture:** Three-phase approach — prerequisites (license gap),
`.github` repo creation with all content, and consolidation (remove
per-repo duplicates after verifying inheritance).

**Tech Stack:** GitHub API via `gh` CLI, markdown, YAML

**Reference:** This plan is adapted from the
`vergil-project/.github` implementation plan
(`vergil-tooling/docs/plans/2026-05-14-github-profile-repo.md`).

**Project context:** mq-rest-admin-project is a multi-repo org providing
typed wrappers for the IBM MQ administrative REST API across five
languages (Python, Java, Go, Ruby, Rust), plus a shared documentation
repo (common) and a Dockerized test environment (dev-environment). There
is also an archived template repo. All repos are GPL-3 licensed and use
VERGIL tooling.

**Active repos (7):**

| Repo | Language | Description |
|---|---|---|
| mq-rest-admin-python | Python | pymqrest — Python wrapper |
| mq-rest-admin-java | Java | Java wrapper |
| mq-rest-admin-go | Go | Go wrapper |
| mq-rest-admin-ruby | Ruby | Ruby wrapper |
| mq-rest-admin-rust | Rust | Rust wrapper |
| mq-rest-admin-common | None | Shared documentation fragments |
| mq-rest-admin-dev-environment | None | Dockerized MQ test environment |

**Archived (skip consolidation):** mq-rest-admin-template

---

## Phase 1: Prerequisites

### Task 0: Enable GitHub Discussions on All Repos

**Files:** None (GitHub API / web UI)

SUPPORT.md will reference GitHub Discussions as the channel for
questions and general conversation. Discussions must be enabled before
SUPPORT.md goes live.

- [ ] Enable Discussions on `mq-rest-admin-project/mq-rest-admin-python`
      via `gh repo edit --enable-discussions`
- [ ] Enable Discussions on `mq-rest-admin-project/mq-rest-admin-java`
- [ ] Enable Discussions on `mq-rest-admin-project/mq-rest-admin-go`
- [ ] Enable Discussions on `mq-rest-admin-project/mq-rest-admin-ruby`
- [ ] Enable Discussions on `mq-rest-admin-project/mq-rest-admin-rust`
- [ ] Enable Discussions on `mq-rest-admin-project/mq-rest-admin-common`
- [ ] Enable Discussions on `mq-rest-admin-project/mq-rest-admin-dev-environment`
- [ ] Verify: visit each repo's Discussions tab in browser to confirm
      it is active

### Task 1: Add Missing LICENSE File

**Files:** `LICENSE` in mq-rest-admin-common

mq-rest-admin-common is the only active repo missing a LICENSE file.
All other repos already have GPL-3.

- [ ] Copy GPL-3 LICENSE to mq-rest-admin-common (PR)
- [ ] Verify: GitHub shows "GPL-3.0" badge on the repo

---

## Phase 2: Create the `.github` Repository

All content is created in a single repo. Tasks are ordered by
dependency.

### Task 2: Create the Repository

**Files:** None (GitHub API)

- [ ] Create `mq-rest-admin-project/.github` as a **public** repository
      via `gh repo create mq-rest-admin-project/.github --public --description "Org-level configuration, community health files, and profile for the mq-rest-admin-project organization"`
- [ ] Clone locally to `/Users/pmoore/dev/projects/mq-rest-admin/.github`
- [ ] Initialize with `develop` as the default branch

### Task 3: VERGIL Tooling Harness

**Files:** `vergil.toml`, `.githooks/pre-commit`, `.claude/settings.json`, `CLAUDE.md`

Minimal tooling setup so the repo follows the same development
workflow as the other repos in the org.

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
      config (identical to other repos)
- [ ] Create `CLAUDE.md` — minimal agent guidance: docs-only repo,
      no application code, use `vrg-commit`, validation via
      `vrg-docker-run -- uv run vrg-validate`
- [ ] Create `LICENSE` — GPL-3 (matching all other repos in the org)
- [ ] Verify: `git config core.hooksPath .githooks` works, raw
      `git commit` is rejected, `vrg-commit` works

### Task 4: Issue Templates and PR Template

**Files:** `ISSUE_TEMPLATE/issue.yml`, `ISSUE_TEMPLATE/config.yml`, `pull_request_template.md`

Migrated from the per-repo copies, which are identical across all
seven active repos and match the VERGIL standard templates.

- [ ] Copy `ISSUE_TEMPLATE/issue.yml` from any active repo
- [ ] Copy `ISSUE_TEMPLATE/config.yml` from any active repo
- [ ] Copy `pull_request_template.md` from any active repo
- [ ] Verify: content is byte-identical to the per-repo versions

### Task 5: CODE_OF_CONDUCT.md

**Files:** `CODE_OF_CONDUCT.md`

- [ ] Copy from `vergil-project/.github/CODE_OF_CONDUCT.md`
      (Contributor Covenant v2.1, already written)
- [ ] Update enforcement contact if needed (should be the same
      maintainer)
- [ ] Review: ensure the text is unmodified except for any
      project-specific enforcement contact changes

### Task 6: SECURITY.md

**Files:** `SECURITY.md`

- [ ] Adapt from `vergil-project/.github/SECURITY.md`
- [ ] Update scope: the language-specific REST API wrapper libraries
      (Python, Java, Go, Ruby, Rust), the shared documentation
      fragments (common), and the Dockerized test environment
      (dev-environment) including its composite GitHub Action
- [ ] Update out-of-scope: vulnerabilities in IBM MQ itself, the
      IBM MQ REST API, or upstream language dependencies — report
      those to IBM or the upstream maintainer respectively
- [ ] Keep same response commitment timelines (7-day acknowledge,
      30-day fix/mitigation target)
- [ ] Enable private vulnerability reporting on the org if not
      already enabled

### Task 7: CONTRIBUTING.md

**Files:** `CONTRIBUTING.md`

Adapted from the vergil-project version but scoped to a multi-repo,
multi-language project.

- [ ] Write "How the project works" section — seven-repo architecture
      overview: five language ports sharing a common API design, a
      shared documentation repo, and a test environment
- [ ] Write "Development setup" section — prerequisites (Docker, uv,
      language-specific toolchains), installing vergil-tooling,
      enabling git hooks. Note that each language port has its own
      development prerequisites
- [ ] Write "Workflow" section — branch from `develop`, use
      `vrg-commit`, use `vrg-submit-pr`, validate via
      `vrg-docker-run -- uv run vrg-validate`. Note language-specific
      validation runs automatically based on each repo's
      `primary-language` setting
- [ ] Write "For contributors using AI tools" section — identity
      model (`<username>-agent`), accountability principle
- [ ] Write "For contributors not using AI" section — fork/branch/PR,
      same quality bar
- [ ] Write "What to expect from review" section — human approval
      required, CI must pass
- [ ] Write "Template override convention" section — document the
      all-or-nothing inheritance rule for template directories
- [ ] Write "Integration testing" section — how to use the
      dev-environment repo for integration tests against a real
      MQ instance
- [ ] Review: ensure consistency with the vergil-project
      CONTRIBUTING.md and current tooling behavior

### Task 8: SUPPORT.md

**Files:** `SUPPORT.md`

- [ ] Adapt from `vergil-project/.github/SUPPORT.md`
- [ ] Support channels: GitHub Issues for bugs and feature requests,
      GitHub Discussions for questions and conversation
- [ ] Note that issues should be filed on the specific language repo,
      not on the `.github` repo
- [ ] State clearly: community project, no SLA-backed support.
      This is not an IBM product — it is an independent wrapper
      library
- [ ] Verify: Discussions links point to working URLs (depends on
      Task 0 being complete)

### Task 9: Root README.md

**Files:** `README.md`

Short description of what the `.github` repo is, not the org profile.

- [ ] Write one-paragraph description: this repo holds org-level
      default configuration for the mq-rest-admin-project organization
- [ ] Explain which files live here and how GitHub's default community
      health file inheritance works
- [ ] Document the template override convention (all-or-nothing per
      directory)
- [ ] Link to the profile README (`profile/README.md`) and
      CONTRIBUTING.md
- [ ] Note the `.github/.github/workflows/` nesting and why it exists

### Task 10: Org Profile README

**Files:** `profile/README.md`

This is the public-facing org profile page. Content should be written
fresh — mq-rest-admin has its own identity.

- [ ] Write a concise description: typed wrappers for the IBM MQ
      administrative REST API across five languages
- [ ] Components table listing all repos with language, package name,
      and brief description
- [ ] Highlight the shared API design: consistent method signatures
      and attribute naming across all ports
- [ ] Note the dev-environment repo for integration testing
- [ ] Note the common repo for shared documentation
- [ ] Add link to `CONTRIBUTING.md` as a call to action
- [ ] Review: verify all URLs resolve, repo names are correct

### Task 11: CI Workflow

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
the duplicated files from the seven active repos. The archived
template repo is left as-is.

### Task 12: Verify Inheritance

**Files:** None (manual testing)

- [ ] Navigate to mq-rest-admin-project/mq-rest-admin-python →
      Issues → New issue. Confirm the org-level issue template appears
      (per-repo copy still exists, so org default won't show yet —
      this is expected)
- [ ] Navigate to Insights → Community Standards. Confirm
      CONTRIBUTING.md, CODE_OF_CONDUCT.md, SECURITY.md, and
      SUPPORT.md appear as inherited
- [ ] Temporarily rename one repo's `ISSUE_TEMPLATE/` directory and
      confirm the org default takes over, then rename it back

### Task 13: Remove Per-Repo Duplicates

**Files:** Issue templates and PR templates in all seven active repos

One PR per repo. These can run in parallel since they're independent.

- [ ] mq-rest-admin-python: remove `.github/ISSUE_TEMPLATE/issue.yml`,
      `.github/ISSUE_TEMPLATE/config.yml`,
      `.github/pull_request_template.md`
- [ ] mq-rest-admin-java: remove the same three files
- [ ] mq-rest-admin-go: remove the same three files
- [ ] mq-rest-admin-ruby: remove the same three files
- [ ] mq-rest-admin-rust: remove the same three files
- [ ] mq-rest-admin-common: remove the same three files
- [ ] mq-rest-admin-dev-environment: remove the same three files
      (note: this repo also has `.github/actions/setup-mq/action.yml`
      — leave that in place, only remove templates)
- [ ] After merging all seven PRs, verify inheritance: create a test
      issue on each repo and confirm the org template appears

### Task 14: Post-Consolidation Verification

**Files:** None (manual testing)

- [ ] Create a real issue on one repo using the inherited template —
      confirm all fields work (type dropdown, problem/goal,
      acceptance criteria)
- [ ] Confirm blank issues are still disabled (config.yml inheritance)
- [ ] Open a PR on one repo — confirm the PR template appears
- [ ] Visit the org profile page — confirm `profile/README.md`
      renders correctly
- [ ] Spot-check community health files: visit a repo's
      "Community Standards" page and confirm all files show as present

### Note: `.DS_Store` Cleanup

Several mq-rest-admin repos have `.DS_Store` files committed under
`.github/`. These should be removed and added to `.gitignore` as part
of the template removal PRs in Task 13, or as a separate cleanup.
