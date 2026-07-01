# Every `vrg-*` tool answers `--help`, enforced

- **Epic:** vergil-project/.github#72
- **Originating feedback:** `vrg-epic-audit` in `vergil-tooling` ran its audit
  when invoked with `--help` and had no usage output at all.
- **Status:** Approved design (2026-07-01)

## 1. Problem

The `vrg-*` CLI suite is inconsistent about `--help`. Running a tool with
`--help` is the canonical way for both humans and agents to learn what a tool
does and how to invoke it — but several tools ignore their arguments entirely
and just run, giving no usage output.

`vrg-epic-audit` is the trigger case: invoking `vrg-epic-audit --help` silently
performs the audit instead of explaining itself, and its output reads like a
menu of pending actions when it is in fact a **read-only** report. A user could
not tell whether the tool had *done* something or merely *shown* something, and
had no way to learn its scope (it audits the whole org, over a 30-day window,
and changes nothing).

## 2. Goal

Every human-facing `vrg-*` console script responds to `-h/--help` with exit 0
and a useful description, and a validation gate prevents regressions — a new
tool shipped without working help fails the suite.

## 3. Audit (current state, in `vergil-tooling`)

- ~40 tools already use `argparse` and get a working `-h/--help` for free
  (none disable it).
- 6 tools ignore `argv` entirely (no help): `vrg-epic-audit`,
  `vrg-activity-log`, `vrg-container-test`, `vrg-pr-issue-linkage`,
  `vrg-repo-profile`, `vrg-roadmap`.
- 1 partial: `vrg-container-docs` prints usage on bad/no args but does not
  recognize `-h/--help` explicitly (exits 1).
- 2 wrappers: `vrg-git`, `vrg-gh` forward `--help` to git/gh but have no
  wrapper-specific help explaining the allowlist / deny-list / identity-mode
  behavior that is the whole reason the wrappers exist.
- 1 hook: `vrg-hook-guard` is invoked by the Claude Code hook system with JSON
  on stdin, not a human CLI — exempt.

## 4. Design decisions (approved)

### 4.1 Convention — `argparse` everywhere

Every human-facing tool uses `argparse` with a descriptive parser. Tools with no
flags still get a parser whose `description`/`epilog` states purpose,
scope/inputs, and whether the tool is **read-only** or **makes changes**.

Rejected alternatives: a shared help decorator/helper (unnecessary machinery
that fights the `argparse` the majority already use) and per-tool hand-rolled
help strings (drift-prone, inconsistent). Change-making tools should document a
`--dry-run`; *adding* `--dry-run` where it is missing is out of scope for this
epic (see §6) — the convention only names it as expected.

### 4.2 Validation gate — runtime invoke `--help`

A test enumerates every console script from `pyproject.toml`
`[project.scripts]`, runs each with `--help` in a subprocess, and asserts exit 0
+ non-empty stdout. An explicit, commented `EXEMPT` set holds `vrg-hook-guard`.
`--help` short-circuits before any side effects, so the gate is hermetic. It
runs inside the existing test suite that `vrg-validate`/CI already executes.

### 4.3 Wrapper coverage

`vrg-git`/`vrg-gh` get a wrapper-specific `--help` that explains the allowlist,
deny-list, and identity-mode behavior *before* forwarding anything to the
underlying git/gh. Implemented as a small manual `--help` intercept, since
`argparse` cannot cleanly own their passthrough arguments.

### 4.4 `vrg-epic-audit` specifically

- `--help` documents: audits the **org of the current repo** (auto-detected from
  `origin`), PRs merged in the last **N days**, **read-only**.
- Auto-detect the org via `github._detect_org()`, promoted to a public
  `detect_org()`. **Error clearly** when it returns `None` (not in a repo,
  non-GitHub remote) — no hardcoded fallback. Removes the
  `_ORG = "vergil-project"` constant.
- Add `--window-days N` (default 30), replacing the `_WINDOW_DAYS` constant.
  There is intentionally **no `--org` flag** — the org follows the repo you run
  from.
- Add a header banner to the output stating the run is **read-only and changed
  nothing** — resolving the specific confusion that started this epic.

## 5. Sequencing (one epic, multiple task-PRs)

Each task is one PR in `vergil-tooling`, filed and linked (`vrg-epic-link`) at
implementation time. The gate lands first so each later task turns more of it
green.

1. **Convention doc + gate + exemption.** Land the gate (initially failing) plus
   the short convention note, making the gaps visible.
2. **`vrg-epic-audit`.** Help + org autodetect + `--window-days` + read-only
   banner + clear error on undetectable org.
3. **Remaining standalone tools.** Convert `vrg-activity-log`,
   `vrg-container-test`, `vrg-pr-issue-linkage`, `vrg-repo-profile`,
   `vrg-roadmap` to descriptive `argparse` parsers.
4. **Wrappers + container-docs.** Wrapper-specific `--help` for
   `vrg-git`/`vrg-gh`; explicit `-h/--help` recognition in `vrg-container-docs`.

## 6. Out of scope

- Adding `--dry-run` to change-making tools that lack it (follow-on tasks).
- Reworking any tool's actual behavior beyond help/usage and the specific
  `vrg-epic-audit` scope changes in §4.4.
