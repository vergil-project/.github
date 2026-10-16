# Opt-in formatter fix mode for `vrg-validate`

Epic: `vergil-project/.github#388`. Documentation task: `vergil-project/.github#389`.
Closing bookends: documentation review `vergil-project/vergil-tooling#3276`,
retrospective `vergil-project/.github#390`.

## 1. Summary

Every language ecosystem in the fleet has a formatter, and `vrg-validate`
already runs each one in *check* mode. When a check fails, the agent fixes the
formatting by hand, re-runs validation, and repeats until the check passes. That
loop costs tokens, wall-clock time and full validation runs. It is avoidable:
the tool that detects the problem can also fix it.

This epic adds an **opt-in** `vrg-validate --fix-format` mode. It runs each
formatter in write mode before the normal checks, so a formatting failure costs
at most one validation run. Plain `vrg-validate` stays check-only. CI stays
check-only, and its gates are unchanged.

Priority: medium-low. The gain is efficiency, not correctness.

## 2. The problem

- **Formatters are audited but never applied.** Today's formatter checks are
  `ruff format --check` (Python), `prettier --check` (TypeScript),
  `cargo fmt --check` (Rust), `spotless:check` (Java),
  `clang-format --dry-run --Werror` (C++), plus formatting cops inside
  `rubocop` (Ruby), formatters inside `golangci-lint run` (Go), and
  `markdownlint` (common to every repo). None of these is ever run in write
  mode by the tooling.
- **Agents fix formatting by hand.** An agent that hits a formatting failure
  edits whitespace, line wrapping or import order itself, then re-validates.
  It is slow and often takes several attempts.
- **Formatters exist only in the dev container.** None of these tools is
  installed on the macOS host or the agent VM. That is deliberate: the
  container architecture exists so that seven languages of toolchains never
  have to be managed on a workstation. The plugin's
  `enforce-host-container-split` hook enforces it.

## 3. Principles

1. **Validation does not mutate.** A validation run checks and reports. It
   must not change the tree unless the caller explicitly asks.
2. **Fixing is explicit and never commits.** Rewritten files are ordinary
   working-tree changes. The agent decides when to commit them, together with
   the rest of its work. `vrg-validate` runs many times while an agent works, so
   automatic commits would scatter "style" commits through the branch.
3. **The fix must match the gate.** A fix is only useful if the matching check
   then passes. Each fix and its check are declared together, once, and use the
   same tool, configuration and file set.
4. **No workflow change is required.** Agents already run `vrg-validate`. The
   failure output teaches the flag, so no skill or CLAUDE.md change is needed
   for the feature to work.
5. **The container boundary holds.** Formatters run only in the dev
   container, as they do today.

## 4. Scope

**In scope:** pure formatters, plus import sorting.

| Ecosystem | Gate check (unchanged) | Fix command |
|---|---|---|
| Python | `ruff format --check src/ tests/` | `ruff format src/ tests/` |
| Python (imports) | `ruff check` (when the repo selects `I`) | `ruff check --select I --fix src/ tests/` |
| TypeScript | `prettier --check . --config <pkg>` | `prettier --write . --config <pkg>` |
| Rust | `cargo fmt --all -- --check` | `cargo fmt --all` |
| Java | `./mvnw spotless:check …` | `./mvnw spotless:apply -B` |
| C++ | find … `\| xargs clang-format --dry-run --Werror` | same find … `\| xargs clang-format -i` |
| Ruby | `bundle exec rubocop` | `bundle exec rubocop --fix-layout` † |
| Go | `golangci-lint run ./...` | `golangci-lint fmt` † |
| Markdown (strict) | `markdownlint --config markdownlint.yaml <files>` | same with `--fix` |
| Markdown (prose) | `markdownlint --config markdownlint-prose.yaml <files>` | same with `--fix` |

† Inferred, not yet confirmed. See §7.

**Out of scope:**

- Lint autofixes other than import sorting, such as `ruff check --fix` in
  general or `rubocop -a`. Those are code changes that should stay visible to
  the agent as findings.
- yamllint, shellcheck, hadolint and actionlint. None of them has a formatter,
  so they stay check-only.
- Per-save formatting through Claude Code `PostToolUse` hooks. This was
  considered and rejected: the formatters exist only in the container, and
  starting a container for every file save is out of proportion to the
  benefit. Installing seven languages' formatters on the host or VM to avoid
  that is exactly what the container architecture was built to prevent.
- Auto-commit of formatting changes. See Principle 2.
- Any change to CI gates, `vergil-actions`, evidence reports or branch
  protection.

## 5. Design

### 5.1 CLI behavior

- **New flag:** `vrg-validate --fix-format`. It is named for formatting so it
  can't be read as "fix lint findings."
- **Plain `vrg-validate` is unchanged,** with one addition. When any failed
  command is a registered formatter check, the final summary prints:

  ```text
  formatting failures can be fixed with: vrg-container-run -- vrg-validate --fix-format
  ```

  The hint is never printed for other failures.
- **With `--fix-format`,** fix stages run before the check stages they pair
  with. Then validation runs once, as normal.
- **Fix scope follows the selected stage.** Common fixes (markdownlint)
  accompany the `common` stage, and language fixes accompany the `lint` stage.
  A full run fixes both. `--check common` fixes only Markdown, and
  `--check lint` fixes only the language formatters. Combining `--fix-format`
  with `--check typecheck`, `--check test` or `--check audit` is a usage error
  with exit code 2, because there is nothing to fix and silently doing nothing
  would mislead.
- **Change report:** after the fix stages, the run lists every file whose
  contents changed, grouped by formatter:

  ```text
  format: rewrote 4 files
    ruff format: src/foo.py, src/bar.py, tests/test_foo.py
    markdownlint --fix: docs/site/docs/guide.md
  ```

  When nothing changed it prints `format: no changes`. It never stages or
  commits.
- **Refused in CI:** if `CI` or `GITHUB_ACTIONS` is set, `--fix-format` exits
  with code 2 and a clear message. CI stays check-only by construction.
- **Container guard:** the existing "must run inside a dev container" guard
  applies unchanged.

### 5.2 Stage ordering

Language formatters such as `ruff` (installed into `.venv`) and `prettier` only
become available after the `install` stage. Markdown fixing needs no install.
With `--fix-format`, a full run is ordered:

```text
format-common -> common -> install -> format-lint -> lint -> typecheck -> test -> audit -> custom
```

Without the flag the order is exactly today's:
`common -> install -> lint -> …`.

Fix stages use `fail_defer` mode. A failing fixer, such as `ruff format` on a
file with a syntax error, is recorded and its output passes through unchanged.
The remaining stages still run, and the overall exit is non-zero. Nothing is
hidden.

### 5.3 The registry change

A new record in `src/vergil_tooling/lib/languages.py`:

```python
@dataclass(frozen=True)
class Formatter:
    name: str                 # label for reports, e.g. "ruff format"
    check: list[str] | None   # gate argv, or None for a fix-only companion
    fix: list[str]            # write-mode argv
```

`Language` gains `formatters: list[Formatter]`, empty by default.

- **One source for LINT.** Each language's `CheckKind.LINT` list is built as
  the formatter checks followed by the remaining lint commands. The set of
  commands CI runs is unchanged; only the order changes, so formatter checks
  fail first and cheaply. Placeholders such as `{configs}` are expanded the
  same way for `check` and `fix` by the existing `language_commands` path.
- **Fix-only companion: Python import sorting.** Import order is enforced by
  the existing `ruff check` when a repo selects rule `I`, as vergil-tooling
  does. Adding a separate `ruff check --select I` gate would put a new failure
  mode on consuming repos that do not select `I`, which breaks the "CI gates
  unchanged" scope. So import sorting is a `Formatter` with `check=None`, and
  its fix runs `ruff check --select I --fix src/ tests/`. Known limitation: an
  `I001` failure appears inside the general `ruff check` output, so the
  `--fix-format` hint does not fire for it. In a repo that does not select `I`,
  `--fix-format` still sorts imports. That is harmless, and consistent with
  the formatter's intent.
- **Common Markdown.** The two markdownlint passes in
  `src/vergil_tooling/bin/validate_common.py` become `Formatter` pairs. They
  reuse the existing file discovery (`_find_strict_markdown_files`,
  `_find_prose_markdown_files`, honouring `[markdownlint].ignore`) and the same
  packaged configs. The fix is the same invocation plus `--fix`.
- **C++.** One helper builds the `find … -print0 | xargs -0 -r clang-format …`
  pipeline from a single find expression, including the `build`,
  `build-sanitize` and `.worktrees` prunes. Check mode passes
  `--dry-run --Werror` and fix mode passes `-i`, so both walk the same files.

### 5.4 Detecting changed files

Before the fix stages run, the driver records a content hash of every file in
`git ls-files --cached --others --exclude-standard`. After each fixer it
re-hashes, and attributes changed files to that fixer. Failures in listing or
hashing are errors and fail the stage. They are never reported as "no
changes".

## 6. Error handling

| Situation | Behavior |
|---|---|
| `--fix-format` with `CI`/`GITHUB_ACTIONS` set | exit 2, message: fixing is local-only |
| `--fix-format` with `--check typecheck/test/audit` | exit 2, usage error |
| outside the dev container | existing guard, exit 1 |
| a fixer exits non-zero | stage fails (`fail_defer`), output passes through, later stages run, overall non-zero |
| file listing or hashing fails | the format stage fails with the error, never silent |
| a language has no formatters | no fix stage for it; checks run as normal |

## 7. Verification of inferred commands

Two fix commands were inferred and must be confirmed against the real
container tool and a real consuming-repo configuration before they ship:

- **Ruby:** `rubocop --fix-layout` (`-x`) corrects only Layout cops. Confirm
  that, in a repo whose only `rubocop` failures are Layout offences, this
  makes `bundle exec rubocop` pass.
- **Go:** `golangci-lint fmt` (golangci-lint v2) applies the formatters
  enabled in the repo's `formatters:` configuration. Confirm the container's
  version supports it and that a consuming repo enables formatters, so
  `golangci-lint run` fails on formatting and `fmt` fixes it.

If a pair cannot be shown to make its check pass, that language ships with no
`Formatter` entry for it, and the gap is recorded in the retrospective. A
guessed fix command never ships.

The same rule applies to every other pair during the cross-language validation
task (§9).

## 8. Testing

In vergil-tooling, run in the dev container through `vrg-validate`:

- **Driver unit tests:**
  - stage order with and without `--fix-format`;
  - fix scope by `--check`;
  - exit 2 for CI and for `--check typecheck/test/audit`;
  - the hint is printed only when a formatter check fails;
  - a failing fixer gives a non-zero exit and later stages still run;
  - the change report attributes files to the right fixer, and prints
    `no changes` correctly;
  - a hashing failure is an error.
- **Registry tests:**
  - for every language, every `Formatter` with a `check` appears in that
    language's LINT list;
  - every language's LINT command multiset equals today's, guarding the CI
    gates;
  - every `Formatter` has a non-empty `fix`.
- **Real-tool tests (Python and Markdown, whose tools exist in this repo's
  container):** a badly formatted fixture is run through the fix command, and
  the paired check then exits 0.

## 9. Operational validation

A `validation` task, run after the implementation merges, using a dev ref of
vergil-tooling (the consuming repo's container install pointed at `develop`
rather than a release tag; the exact mechanism is settled in the plan), in one
consuming repo per language that has a formatter: Python, TypeScript, Rust,
Java, C++, Ruby, Go. Markdown is covered by any repo. For each:

1. Introduce a formatting-only defect.
2. Confirm plain `vrg-validate` fails and prints the `--fix-format` hint.
3. Run `vrg-validate --fix-format` and confirm the change report names the
   file.
4. Confirm a following plain `vrg-validate` passes the formatting checks.

Any pair that fails is removed in a follow-up task, and the gap is recorded.

## 10. Documentation

- vergil-tooling: the site docs `vrg-validate` reference page, and the
  Validation section of `CLAUDE.md`, gain a short description of
  `--fix-format`.
- vergil-claude-plugin: one line in the validation guidance of the relevant
  skills ("on formatting failures, run with `--fix-format`"). This is optional;
  the failure hint already teaches the flag.
- The documentation-review bookend (`vergil-tooling#3276`) sweeps these and
  spawns per-repo tasks as needed.

## 11. Rollout

Agents get the feature through a normal vergil-tooling release. Release is a
human-gated step, so no deployment task is seeded. The operational validation
(§9) runs against a dev ref and does not wait for the release.
