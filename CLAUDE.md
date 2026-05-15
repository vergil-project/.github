# CLAUDE.md

This file provides guidance to Claude Code when working in this repository.

## Project Overview

This is the `vergil-project/.github` profile repository. It holds
org-level default configuration for the vergil-project GitHub
organization: the org profile README, default community health files
(issue templates, PR template, CONTRIBUTING.md, SECURITY.md,
CODE_OF_CONDUCT.md, SUPPORT.md), and the repo's own CI workflow.

This is a documentation-only repository. There is no application code,
no Python package, no build artifacts.

## Shell command policy

Use `vrg-git` instead of `git` for all git operations. Use `vrg-gh`
instead of `gh` for all GitHub CLI operations. These wrappers enforce
subcommand allowlists, flag deny lists, credential selection, and
audit logging.

Raw `git` and `gh` are denied by the permission model. If a command
is not available through the wrappers, explain the situation to the
human who can run it directly via `! <command>` in the prompt.

## Development Commands

### Validation

```bash
vrg-docker-run -- uv run vrg-validate
```

This runs common checks only (markdownlint, yamllint, shellcheck,
actionlint). There are no language-specific checks because
`primary-language = "none"` in `vergil.toml`.

### Commits

Use `vrg-commit` for all commits. Raw `git commit` is blocked by the
pre-commit hook.

### Pull Requests

Use `vrg-submit-pr` for all pull requests.

## Architecture

### File Layout

- `profile/README.md` — org profile page (public-facing, renders on
  the GitHub org landing page)
- `ISSUE_TEMPLATE/` — default issue templates inherited by all repos
- `pull_request_template.md` — default PR template inherited by all repos
- `CONTRIBUTING.md`, `SECURITY.md`, `CODE_OF_CONDUCT.md`, `SUPPORT.md` —
  community health files inherited by repos without their own copies
- `.github/workflows/ci.yml` — CI for this repo (the nested `.github/`
  is correct — this repo's own workflows live in its own `.github/`
  directory)
- `README.md` — describes what this repo is (not the org profile)

### Inheritance Model

GitHub's `.github` repo provides org-level defaults:
- Standalone files (CONTRIBUTING, SECURITY, etc.) inherit per-file
- Template directories (ISSUE_TEMPLATE/) inherit per-directory
  (all-or-nothing)
- LICENSE files do not inherit — each repo must have its own
