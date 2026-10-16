# Opt-in formatter fix mode for `vrg-validate`

Epic: `vergil-project/.github#388`. Documentation task: `vergil-project/.github#389`.
Closing bookends: documentation review `vergil-project/vergil-tooling#3276`,
retrospective `vergil-project/.github#390`. Follow-on brainstorm:
`vergil-project/.github#392` (consolidate Markdown linting on rumdl).

## 1. Summary

Every language ecosystem in the fleet has a formatter, and `vrg-validate`
already runs each one in *check* mode. When a check fails, the agent fixes the
formatting by hand, re-runs validation, and repeats until the check passes. That
loop costs tokens, wall-clock time and full validation runs. It is avoidable:
the tool that detects the problem can usually also fix it.

This epic adds an **opt-in** `vrg-validate --fix-format` mode. It runs each
formatter in write mode before the normal checks, so a formatting failure costs
at most one validation run. Plain `vrg-validate` stays check-only. CI stays
check-only, and its gates are unchanged except for one equivalent split
(§5.3).

For Markdown, the most common bounce is line length (MD013), and
`markdownlint --fix` cannot fix it. The epic adds **rumdl**, a Markdown
formatter that reads our markdownlint config and can reflow long lines, to the
dev containers as a fix-only companion. markdownlint stays the gate.

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
- **markdownlint cannot reflow prose.** `markdownlint --fix` fixes only the
  rules that ship a fixer (blank lines, list and heading spacing, and similar).
  MD013 (line length, 100 characters in our strict config) has none, so long
  lines always need a manual fix.

## 3. Principles

1. **Validation does not mutate.** A validation run checks and reports. It
   must not change the tree unless the caller explicitly asks.
2. **Fixing is explicit and never commits.** Rewritten files are ordinary
   working-tree changes. The agent decides when to commit them, together with
   the rest of its work. `vrg-validate` runs many times while an agent works, so
   automatic commits would scatter "style" commits through the branch.
3. **The fix must match the gate.** A fix is only useful if the matching check
   then passes. Each fix and its check are declared together, once, and use the
   same tool, configuration and file set where possible. Where the fixer is a
   different tool from the gate (rumdl), agreement between them is proven by
   tests (§8).
4. **Hints must not mislead.** The failure output only promises that
   `--fix-format` will fix a failure when the failing command is a pure
   formatter check.
5. **No workflow change is required.** Agents already run `vrg-validate`. The
   failure output teaches the flag, so no skill or CLAUDE.md change is needed
   for the feature to work.
6. **The container boundary holds.** Formatters run only in the dev
   container, as they do today.

## 4. Scope

**In scope:** pure formatters, the format-fixable part of mixed tools, import
sorting, and Markdown reflow through rumdl.

A *pure* check fails only on formatting, so the fix always clears it. A
*mixed* check also reports findings no formatter can fix. A mixed tool gets a
fix-only companion: it is fixed by `--fix-format`, but its check is not
registered as a formatter check (see §5.1 for the hint consequence).

| Ecosystem | Gate check | Kind | Fix command |
|---|---|---|---|
| Python | `ruff format --check src/ tests/` | pure | `ruff format src/ tests/` |
| Python (imports) | `ruff check` (when the repo selects `I`) | mixed | `ruff check --select I --fix src/ tests/` |
| TypeScript | `prettier --check . --config <pkg>` | pure | `prettier --write . --config <pkg>` |
| Rust | `cargo fmt --all -- --check` | pure | `cargo fmt --all` |
| Java | `./mvnw spotless:check -B` (split out, §5.3) | pure | `./mvnw spotless:apply -B` |
| C++ | find … `\| xargs clang-format --dry-run --Werror` | pure | same find … `\| xargs clang-format -i` |
| Ruby | `bundle exec rubocop` | mixed | `bundle exec rubocop --fix-layout` † |
| Go | `golangci-lint run ./...` | mixed | `golangci-lint fmt` † |
| Markdown | `markdownlint` (strict and prose passes) | mixed | `rumdl fmt` (reflow) then `markdownlint --fix`, same file lists † |

† Not yet confirmed. See §7.

**Out of scope:**

- Lint autofixes other than import sorting, such as `ruff check --fix` in
  general or `rubocop -a`. Those are code changes that should stay visible to
  the agent as findings.
- yamllint, shellcheck, hadolint and actionlint. None of them has a formatter,
  so they stay check-only.
- Replacing markdownlint with rumdl as the Markdown gate. That is a fleet-wide
  gate change and is the subject of follow-on brainstorm
  `vergil-project/.github#392`.
- Per-save formatting through Claude Code `PostToolUse` hooks. This was
  considered and rejected: the formatters exist only in the container, and
  starting a container for every file save is out of proportion to the
  benefit. Installing seven languages' formatters on the host or VM to avoid
  that is exactly what the container architecture was built to prevent.
- Auto-commit of formatting changes. See Principle 2.
- Any change to CI gates, `vergil-actions`, evidence reports or branch
  protection, apart from the Java split in §5.3.

## 5. Design

### 5.1 CLI behavior

- **New flag:** `vrg-validate --fix-format`. It is named for formatting so it
  can't be read as "fix lint findings."
- **Plain `vrg-validate` is unchanged,** with one addition: a hint in the final
  summary.
  - When a failed command is a registered **pure** formatter check, the hint is
    firm:

    ```text
    formatting failures can be fixed with: vrg-container-run -- vrg-validate --fix-format
    ```

  - When a failed command is a **mixed** check with a fix-only companion
    (`ruff check`, `rubocop`, `golangci-lint run`, `markdownlint`), the hint is
    soft:

    ```text
    <tool> failed; some of its findings may be auto-fixable with: vrg-container-run -- vrg-validate --fix-format
    ```

  - No hint is printed for any other failure.
- **With `--fix-format`,** all applicable fix stages run before any check
  stage (§5.2). Then validation runs once, as normal.
- **Fix scope follows the selected stage.** Common fixes (Markdown) accompany
  the `common` stage, and language fixes accompany the `lint` stage. A full run
  fixes both. `--check common` fixes only Markdown, and `--check lint` fixes
  only the language formatters. Combining `--fix-format` with
  `--check typecheck`, `--check test` or `--check audit` is a usage error with
  exit code 2, because there is nothing to fix and silently doing nothing would
  mislead.
- **Change report:** after the fix stages, the run lists every file whose
  contents changed, grouped by formatter:

  ```text
  format: rewrote 4 files
    ruff format: src/foo.py, src/bar.py, tests/test_foo.py
    rumdl fmt: docs/site/docs/guide.md
  ```

  When nothing changed it prints `format: no changes`. It never stages or
  commits.
- **Refused in CI:** if `CI` or `GITHUB_ACTIONS` is set, `--fix-format` exits
  with code 2 and a clear message. CI stays check-only by construction.
- **Container guard:** the existing "must run inside a dev container" guard
  applies unchanged.

### 5.2 Stage ordering

Language formatters such as `ruff` (installed into `.venv`) and `prettier` only
become available after the `install` stage. Some fixers also touch files that
other stages check: TypeScript's `prettier --write .` rewrites `.md`, `.yml`
and `.json` files, which the `common` stage checks. So with `--fix-format`,
**every fix runs before any check**, and every check sees the final tree:

```text
install -> format-common -> format-lint -> common -> lint -> typecheck -> test -> audit -> custom
```

Within `format-common`, `rumdl fmt` runs before `markdownlint --fix`. In a
TypeScript repo prettier then runs last on Markdown, and markdownlint
(in `common`) validates the result. Any real conflict between the tools
therefore shows up in the same run instead of on the next one.

Without the flag the order is exactly today's:
`common -> install -> lint -> …`. Only with the flag can an `install` failure
(`fail_fast`) stop the run before the common checks.

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
    check: list[str] | None   # pure gate argv, or None for a fix-only companion
    fix: list[str]            # write-mode argv
    soft_hint_for: str | None = None  # mixed gate command this companion fixes
```

`Language` gains `formatters: list[Formatter]`, empty by default.

- **One source for LINT.** Each language's `CheckKind.LINT` list is built as
  the pure formatter checks followed by the remaining lint commands. Formatter
  checks run first, so they fail first and cheaply. Placeholders such as
  `{configs}` are expanded the same way for `check` and `fix` by the existing
  `language_commands` path.
- **Mixed tools get fix-only companions** (`check=None`), with `soft_hint_for`
  naming the mixed gate command whose failure triggers the soft hint: Python
  import sorting (`ruff check`), Ruby (`rubocop`), Go (`golangci-lint run`), and
  Markdown (`markdownlint`).
- **Python import sorting.** Import order is enforced by the existing
  `ruff check` when a repo selects rule `I`, as vergil-tooling does. A separate
  `ruff check --select I` gate would put a new failure mode on consuming repos
  that do not select `I`, so import sorting is a fix-only companion. In a repo
  that does not select `I`, `--fix-format` still sorts imports. That is
  harmless, and consistent with the formatter's intent.
- **Java split.** Today's single command
  `["./mvnw", "spotless:check", "checkstyle:check", "-B"]` becomes two:
  `["./mvnw", "spotless:check", "-B"]` (a pure formatter check) and
  `["./mvnw", "checkstyle:check", "-B"]`. The two commands check exactly what
  the one did. The cost is one extra Maven startup. This is the only change to
  any LINT command set, and the registry test (§8) names it explicitly.
- **Common Markdown.** The two markdownlint passes in
  `src/vergil_tooling/bin/validate_common.py` gain fix-only companions. Each
  reuses the existing file discovery (`_find_strict_markdown_files`,
  `_find_prose_markdown_files`, honouring `[markdownlint].ignore`) and the
  matching packaged config (`markdownlint.yaml`, `markdownlint-prose.yaml`).
  The fix for each pass is `rumdl fmt` and then `markdownlint --fix`, both on
  that pass's file list.
- **rumdl configuration.** rumdl reads markdownlint config files, but our
  packaged configs leave MD013 reflow off, which is rumdl's default. The
  tooling passes rumdl its own packaged config per pass, which mirrors the
  matching markdownlint config and adds `reflow: true` to MD013
  (`line-length: 100`, `tables: false`, `code-blocks: false` for the strict
  pass; the prose pass mirrors its relaxed MD013 settings). rumdl's reflow
  leaves code blocks, tables, headings and reference definitions alone. A long
  heading therefore still needs a manual fix.
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

### 5.5 Container work (vergil-containers)

rumdl is added to the shared layer that installs markdownlint today
(`node-markdownlint.dockerfile`), so it reaches every dev image. It is pinned
to an exact version, because rumdl is pre-1.0 and its output may change between
releases. The pin joins the tools that `bump-tools` keeps current.

The tooling depends on rumdl being in the images. A vergil-containers release
that publishes the new images is a **human-gated precondition** for the
tooling's Markdown fix and for the real-tool rumdl tests. It is never performed
by an agent.

## 6. Error handling

| Situation | Behavior |
|---|---|
| `--fix-format` with `CI`/`GITHUB_ACTIONS` set | exit 2, message: fixing is local-only |
| `--fix-format` with `--check typecheck/test/audit` | exit 2, usage error |
| outside the dev container | existing guard, exit 1 |
| a fixer exits non-zero | stage fails (`fail_defer`), output passes through, later stages run, overall non-zero |
| a fixer binary is missing (e.g. an older image without rumdl) | stage fails with a message naming the tool and the container release that provides it; never skipped silently |
| file listing or hashing fails | the format stage fails with the error, never silent |
| a language has no formatters | no fix stage for it; checks run as normal |

## 7. Verification of unconfirmed commands

Three fixes must be confirmed against the real container tool and a real
consuming-repo configuration before they ship:

- **Ruby:** `rubocop --fix-layout` (`-x`) corrects only Layout cops. Confirm
  that, in a repo whose only `rubocop` failures are Layout offences, this
  makes `bundle exec rubocop` pass.
- **Go:** `golangci-lint fmt` (golangci-lint v2) applies the formatters
  enabled in the repo's `formatters:` configuration. Confirm the container's
  version supports it and that a consuming repo enables formatters, so
  `golangci-lint run` fails on formatting and `fmt` fixes it.
- **Markdown (rumdl):** confirm that `rumdl fmt` with the packaged configs,
  followed by `markdownlint --fix`, leaves files that markdownlint passes. In
  particular, check that rumdl's reflow and markdownlint's rules do not loop
  (rumdl rewrites something markdownlint then flags). Run it on this
  repository's `docs/site/` and on the `.github` repository's `epics/`.

If a fix cannot be shown to clear what it targets, that language ships with no
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
  - the firm hint is printed only when a pure formatter check fails, the soft
    hint only when a mixed gate with a companion fails, and no hint otherwise;
  - a failing fixer gives a non-zero exit and later stages still run;
  - a missing fixer binary fails the stage with the named message;
  - the change report attributes files to the right fixer, and prints
    `no changes` correctly;
  - a hashing failure is an error.
- **Registry tests:**
  - for every language, every `Formatter` with a `check` appears in that
    language's LINT list;
  - every language's LINT command multiset equals today's, except Java, whose
    expected change (the split) is stated in the test;
  - every `Formatter` has a non-empty `fix`, and every fix-only companion names
    an existing gate command in `soft_hint_for`.
- **Real-tool tests (Python and Markdown, whose tools exist in this repo's
  container):** a badly formatted fixture is run through the fix command, and
  the matching check then exits 0. The Markdown fixture includes over-long
  paragraphs, so rumdl reflow is exercised. The rumdl tests need the
  vergil-containers release (§5.5).

## 9. Operational validation

A `validation` task, run after the implementation merges. It uses a dev ref of
vergil-tooling: the consuming repo's container install is pointed at `develop`
with `VRG_DOCKER_INSTALL_TAG=develop`, so the check does not wait for a tooling
release. It needs the vergil-containers release (§5.5). It runs in one
consuming repo per language that has a formatter: Python, TypeScript, Rust,
Java, C++, Ruby, Go. Markdown, including rumdl reflow, is covered by any repo.
For each:

1. Introduce a formatting-only defect.
2. Confirm plain `vrg-validate` fails and prints the right hint, firm or soft.
3. Run `vrg-validate --fix-format` and confirm the change report names the
   file.
4. Confirm a following plain `vrg-validate` passes the formatting checks.

Any pair that fails is removed in a follow-up task, and the gap is recorded.

## 10. Documentation

- vergil-tooling: the site docs `vrg-validate` reference page, and the
  Validation section of `CLAUDE.md`, gain a short description of
  `--fix-format`, including which Markdown issues stay manual (long headings,
  and anything rumdl leaves alone).
- vergil-containers: the README tool list gains rumdl.
- vergil-claude-plugin: one line in the validation guidance of the relevant
  skills ("on formatting failures, run with `--fix-format`"). This is optional;
  the failure hint already teaches the flag.
- The documentation-review bookend (`vergil-tooling#3276`) sweeps these and
  spawns per-repo tasks as needed.

## 11. Rollout

1. vergil-containers adds rumdl, and a human cuts a container release.
2. vergil-tooling ships `--fix-format`. The Markdown companion depends on step 1.
3. The cross-language validation (§9) runs against a tooling dev ref.
4. Agents get the feature through a normal vergil-tooling release, which is
   human-gated. No separate deployment task is seeded, because the container
   release and the tooling release are both human-gated preconditions rather
   than agent-run steps.
