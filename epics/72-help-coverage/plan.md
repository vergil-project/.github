# Implementation plan — `vrg-*` help coverage (Epic #72)

Each task below is a single PR in `vergil-tooling`, filed at implementation time
and linked to the epic with
`vrg-epic-link --epic vergil-project/.github#72 --task vergil-project/vergil-tooling#<TASK>`.

## Task 1 — convention doc + validation gate

**Lands first, deliberately failing**, so the remaining gaps are visible in CI.

- Add a short convention note under `docs/` (this repo's convention home):
  every human-facing `vrg-*` tool responds to `-h/--help` (exit 0, non-empty),
  via `argparse`, describing purpose, scope, and read-only vs. change-making.
- Add a pytest test that reads `[project.scripts]` from `pyproject.toml`, runs
  each script with `--help` in a subprocess, and asserts exit 0 + non-empty
  stdout.
- Add a commented `EXEMPT` set containing `vrg-hook-guard` (hook, not a human
  CLI) with the rationale inline.

**Done when:** the test exists and fails only on the currently-uncovered tools.

## Task 2 — `vrg-epic-audit`

- Promote `github._detect_org()` to a public `detect_org()`.
- In `vrg_epic_audit.py`: add an `argparse` parser (`--window-days`, default 30;
  descriptive `--help` covering scope + read-only); resolve the org via
  `detect_org()`, erroring clearly when it is `None`.
- In `lib/epic_audit.py`: drop `_ORG` / `_WINDOW_DAYS` constants; the org and
  window flow in from the entrypoint; add a read-only banner to `render()`
  output.
- Update existing tests (`test_epic_audit.py`, `test_vrg_epic_audit.py`) for the
  new signatures and banner; add coverage for the undetectable-org error and
  `--window-days`.

**Done when:** `vrg-epic-audit --help` explains scope and read-only status;
running it in a repo audits that repo's org; running it outside a repo errors
clearly; the gate is green for this tool.

## Task 3 — remaining standalone tools

Convert to descriptive `argparse` parsers (most take no flags — the parser
exists for its `--help` description): `vrg-activity-log`, `vrg-container-test`,
`vrg-pr-issue-linkage`, `vrg-repo-profile`, `vrg-roadmap`.

**Done when:** the gate is green for all five.

## Task 4 — wrappers + `vrg-container-docs`

- `vrg-git`/`vrg-gh`: manual `--help` intercept describing the allowlist,
  deny-list, and identity-mode behavior before forwarding to git/gh.
- `vrg-container-docs`: recognize `-h/--help` explicitly (exit 0), reusing its
  existing `_usage()`.

**Done when:** the gate is green for the whole suite (only `vrg-hook-guard`
exempt).

## Verification (every task)

`vrg-container-run -- vrg-validate` from inside the task's worktree.
