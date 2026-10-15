# Exception management — a documented, audited register for every gate suppression

Epic: `vergil-project/.github#375`. Documentation task: `vergil-project/.github#376`.
Prior art: `vergil-project/.github#94` (SARIF-gate exception allowlist — closed,
never implemented), `vergil-project/vergil-tooling#3107` (SARIF gate honours
in-source suppressions), `vergil-project/vergil-actions#763` (all-hard-gates
principle).

## 1. Summary

Every quality and security gate in the fleet has an escape hatch: `# noqa`,
`# type: ignore`, `# pragma: no cover`, `# nosemgrep`, `# shellcheck disable=`,
`.trivyignore`, ruff `per-file-ignores`, rules disabled in shared configs. Nothing
tracks their use. An agent that finds a rule inconvenient can suppress it on one
line and the gate goes green; nobody is asked and nothing records why.

This epic makes every exception **accounted for**:

- every exception in a repo is documented in one central, readable **register**;
- agents must obtain **explicit human approval** before adding or widening one;
- a deterministic **audit gate** fails any exception that is not documented;
- new exceptions are **visible** in the PR preview and on a published docs page.

The long-term target is zero exceptions. An exception is, by definition,
exceptional: a specific, justified departure from a rule.

## 2. The problem in detail

Measured in `vergil-tooling` (2026-10-14):

| Mechanism | Sites |
|---|---|
| `# noqa` | ~480 |
| `# pragma: no cover` | 61 |
| `# type: ignore` | 9 |
| `# nosemgrep` | 9 |

- The ~480 `noqa` sites collapse to ~54 (rule × top-level directory) groups;
  three rules (S603, S607, S108) are ~75% of them. Many were added unilaterally
  by agents to move past a finding.
- Some are **dead**: they suppress rules (`ANN001`, `PLR2004`, `PLC0415`,
  `BLE001`) that are not in the repo's ruff `select` set.
- The shared configs vergil-tooling ships (`src/vergil_tooling/configs/`) disable
  rules fleet-wide (e.g. markdownlint `MD060`, yamllint `document-start`) with no
  recorded rationale.
- `vergil-containers/.trivyignore` holds ~180 lines of CVE suppressions; its
  comments are good, but nothing forces review or removal.
- Existing written rules (coverage-exception justification, "AI tooling must not
  suppress warnings by default") are conventions only — nothing checks them.
- Epic #94 designed a justified SARIF allowlist in `vergil.toml`, but the code
  never shipped; #3107 then made the SARIF gate honour in-source `nosemgrep`
  comments, which closed a real blocker but means any agent can silence a
  semgrep finding with no review.

The PR merge is already a human action, but a suppression buried in a large diff
is easy to miss, and reviewers optimise for "can I merge?" (#763). Visibility
alone is not enough; an explicit, recorded justification is required.

## 3. Threat model and principles

- **Non-malicious agent.** The agent is assumed to be cooperative but fallible —
  it will sometimes forget the rules. Defending against a deliberately malicious
  agent (the "Mimir" concept) is **out of scope**; this design has no human-only
  trust anchor, and an agent that set out to forge a register entry could.
- **Two layers.** Guidance (rules and a skill) catches most cases up front; a
  deterministic gate catches what guidance misses.
- **Hard gates, not warnings** (#763) — with two deliberate, reasoned
  exceptions, both reported rather than failed because failing them blocks
  unrelated work on a trigger outside the PR: fleet-default staleness (§6.3),
  which would turn every central change into a fleet-wide flag day; and expired
  entries (§6.2), which would fail every PR in a repo on a date. Both are made
  actionable by a scheduled review (§13), not by blocking.
- **Exceptions are temporary by default.** An exception should exist only until
  it can be removed; a permanent exception should itself be the exception.
  `expires` (§5.1) and the scheduled review are how that is tracked.
- **Never silent.** Anything the audit cannot check is reported as uncovered, not
  passed quietly.
- **Minimum scope.** A rule-level exception covers the smallest set of
  directories that needs it. Directories are preferred over individual files.

## 4. Three levels of exception

| Level | What | Owned by | Documented in |
|---|---|---|---|
| 1. Fleet default | A rule Vergil disables for every repo | Vergil | Fleet register (package data) |
| 2. Rule-level | A rule disabled repo-wide or for paths, beyond fleet defaults | Repo | Repo's `vergil-exceptions.toml` |
| 3. Line-level | An inline suppression on one line or block | Repo | Repo's `vergil-exceptions.toml` |

- A rule-level exception **is** the pre-approval for that rule in its scope.
- A line-level exception is a one-off. When the same rule keeps needing
  line-level exceptions, that is a signal to promote it to a (narrowly scoped)
  rule-level exception — a decision for the human, prompted by the audit report
  (§6.2).

## 5. The registers

### 5.1 Repo register — `vergil-exceptions.toml`

A committed file at the repo root. The name is namespaced so it cannot collide
with another tool's conventional file name, and it establishes a
`vergil-<topic>.toml` convention for Vergil-specific configuration that should not
crowd `vergil.toml` (no other files are created under that convention by this
epic).

It holds **only** the repo's own exceptions (levels 2 and 3). In vergil-tooling it
documents vergil-tooling's own exceptions and never the fleet defaults.

```toml
[[rule]]                         # level 2
id      = "EXC-001"
tool    = "ruff"
rules   = ["S101"]
scope   = ["tests/**"]
reason  = "pytest's assertion model requires bare assert in tests."
added   = 2026-10-20

[[rule]]
id      = "EXC-002"
tool    = "mypy"
rules   = ["import-untyped"]
scope   = ["scripts/**"]
reason  = "Build scripts import a vendored helper that ships no type stubs."
added   = 2026-10-20

[[inline]]                       # level 3
id      = "EXC-014"
tool    = "ruff"
rules   = ["S603"]
reason  = "Argument list is built from validated, fixed git subcommands; no shell."
added   = 2026-10-20

[[inline]]
id      = "EXC-031"
tool    = "trivy"
rules   = ["CVE-2026-33818"]
reason  = "Go encoding/asn1 DoS in bundled Go CLIs; not reachable from container usage."
issue   = "vergil-project/vergil-containers#555"
expires = 2027-01-15
added   = 2026-10-20
```

Field rules:

| Field | `[[rule]]` | `[[inline]]` | Notes |
|---|---|---|---|
| `id` | required | required | `EXC-NNN` (at least three digits), unique in the file |
| `tool` | required | required | a supported plug-in name (§7) |
| `rules` | required | required | non-empty list of the tool's rule IDs |
| `scope` | required | — | globs; repo-wide must be written `["**"]` explicitly |
| `reason` | required | required | non-empty prose |
| `added` | required | required | date |
| `expires` | optional | optional | date; required for `trivy` entries |
| `issue` | required iff `expires` | required iff `expires` | `owner/repo#N` |

- There is **no list of locations** in the register. The audit finds sites by
  scanning, so a stored list could only go stale.
- Many sites may reference one `[[inline]]` entry (many-to-one), but every new
  site still needs human approval (§8).
- `EXC-NNN` is the **only** ID namespace. A repo entry never cites a fleet
  entry: fleet defaults reach the repo through the package or a managed block
  (§6.3) and are documented once, in the fleet register. A rule the repo
  disables in its own config is the repo's exception and needs its own entry.

### 5.2 Inline linkage

The canonical form is a **separate** `# vrg-exc: EXC-NNN` comment after the
tool's own suppression, on the same line:

```python
subprocess.run(cmd, check=True)  # noqa: S603  # vrg-exc: EXC-014
return x  # type: ignore[return-value]  # vrg-exc: EXC-002
```

```bash
# shellcheck disable=SC2086  # vrg-exc: EXC-020
```

The ID must be its own comment, not trailing text inside the suppression:
mypy 2.1.0 rejects `# type: ignore[return-value]  vrg-exc: EXC-002` as
`Invalid "type: ignore" comment [syntax]`, while both ruff 0.15.13 and mypy
accept the separate-comment form (verified 2026-10-15).

In Markdown the marker is an HTML comment: `<!-- vrg-exc: EXC-005 -->`.

**Line-above form.** The plug-in also reads the ID from a comment-only
`# vrg-exc: EXC-NNN` line immediately above the suppression. It is used:

- for any tool that cannot take a second comment on the same line;
- for `nosemgrep`, because semgrep reads the rest of a `nosemgrep:` comment as
  its rule-ID list;
- for `.trivyignore`, where one marker covers a **group**: every CVE line that
  follows it, up to the next blank line. This keeps the grouped-by-carrier
  layout. The entry's `rules` must still list every CVE in the group, so adding
  a CVE to a group still changes the register.

Inline markers take only the repo's `EXC-NNN` IDs. Line-level exceptions are
always per repo, because the line is in the repo: there are no fleet line-level
entries, including for lines that Vergil tooling generates (§10, repo-init).

### 5.3 Fleet register

A TOML file shipped as vergil-tooling package data
(`src/vergil_tooling/configs/exceptions.toml`). It holds `[[rule]]` entries only,
in the repo register's shape but with **no `id`**: a fleet default is identified
by its tool and rule (for example "markdownlint `MD060`"), and each tool-and-rule
pair appears in one entry at most. It has no `[[inline]]` entries.

It documents every rule disabled by the shared configs Vergil ships
(markdownlint, yamllint, ansible-lint, C++ and TypeScript configs, which
`vrg-validate` passes to the tools directly from the package), the semgrep
`DEFAULT_EXCLUDED_RULES`, and the fleet ruff ignores delivered by managed block
(§6.3). Only the human decides what becomes a fleet default. It carries a
monotonically increasing `revision`, and records the SHA-256 of each
managed-block body (§6.3) for every revision it has ever shipped, so older
blocks remain verifiable.

## 6. The audit — `vrg-exceptions audit`

### 6.1 Failures (exit non-zero)

1. An inline suppression with no `vrg-exc:` ID.
2. An ID not present in the repo register (including any ID that is not
   `EXC-NNN`).
3. A site whose suppressed rule is not in its entry's `rules`.
4. A site whose entry is `[[rule]]` rather than `[[inline]]`, or vice versa.
5. A **dead** suppression — one that suppresses a rule the tool does not have
   enabled, or (where the tool can report it) suppresses nothing. Ruff's is
   detected with `RUF100`. mypy and ty report unused ignores themselves in
   strict mode (`warn_unused_ignores`), so their own run fails on those and the
   audit does not repeat the check.
6. A rule disabled in the repo's own config that is not covered by a repo
   `[[rule]]` entry whose `scope` matches. Rules in a valid managed block are
   covered by the fleet register.
7. A `[[rule]]` entry that matches no disabled rule (stale register entry), and
   an `[[inline]]` entry referenced by no site.
8. A managed block whose content hash does not match the revision it claims, or
   which claims a revision the fleet register does not know (hand-edited; §6.3).
9. A register that fails schema validation (§5.1 field rules).
10. A **blanket** suppression — one that names no rule (a bare `# noqa`,
    `# type: ignore`, `# nosemgrep`, or `<!-- markdownlint-disable -->`),
    whatever its marker says. An approval covers specific rules, and a blanket
    suppression also silences rules added later.

### 6.2 Reported, not failed

- Managed blocks behind the current fleet revision (§6.3).
- `[[inline]]` entries with several sites — promotion candidates (§4).
- Entries past their `expires` date, and entries expiring within 30 days.
  Expiry never fails the gate: a date-triggered failure would block every PR in
  the repo, including unrelated ones. Acting on expired entries is the job of the
  scheduled review (§13).
- Exception mechanisms found in the repo that no plug-in covers yet.

The report is written to stdout and, in CI, to the job summary.

### 6.3 Managed blocks and fleet revisions

How a fleet default reaches a repo depends on the tool:

- **Passed from the package:** markdownlint, yamllint, ansible-lint and the C++
  and TypeScript configs are handed to the tools by `vrg-validate` directly from
  `vergil_tooling.configs` (`validate_common.py`, `languages.py` `{configs}`).
  Repos hold no copy, so there is nothing to drift; the fleet register only
  documents them.
- **Repo-local config:** the setting is copied into the repo config as a managed
  block, the way `vrg-gitignore-sync` maintains `.gitignore`:

  ```text
  # >>> vergil-managed: fleet-exceptions rev 7 sha256:… (do not edit) >>>
  …
  # <<< vergil-managed <<<
  ```

**v1 builds managed blocks for one tool: ruff, in `pyproject.toml`.** Ruff is
configured per repo, carries the richest set of exceptions, and is the only v1
tool whose fleet defaults must live in the repo. The block sits inside
`[tool.ruff.lint]` `extend-ignore`, delimited by comments, so a repo's own
ignores can sit beside it:

```toml
[tool.ruff.lint]
extend-ignore = [
    # >>> vergil-managed: fleet-exceptions rev 3 sha256:… (do not edit) >>>
    "D400",
    # <<< vergil-managed <<<
]
```

`sync` rewrites the block as text, preserving the rest of the file's formatting
and comments, and reuses the block handling in `lib/gitignore.py` where it fits.
The block machinery (render, parse, hash, revision check, sync) is written
tool-agnostically so other tools can adopt it; extending it to other tools — and
to fleet syncs generally — is the follow-on brainstorm in §13.

The two kinds of drift are treated differently:

- **Edited by hand → fail.** The body's hash does not match the hash the fleet
  register recorded for the revision the block claims. This catches an agent
  editing a fleet exception in place, and never fails because of a change made
  elsewhere.
- **Behind the current revision → report.** Safe, because a stale block never
  produces an undocumented exception: a newly disabled fleet rule is still
  enforced (stricter), and a re-enabled one is still documented by the old
  revision. `vrg-exceptions sync` brings the block up to date.
- **Missing → report, as behind.** A repo with no block is behind at revision 0:
  it still enforces every fleet rule. So adding the first fleet ruff default
  never fails the repos that have not synced yet.

Repos follow a vergil-tooling release *line* (`vergil = "v2.1"`), so a check
against the current revision would break every repo on every central change.
The revision stamp is what decouples them.

### 6.4 Activation

The audit runs as a common check in `vrg-validate` (locally and in PR CI, with no
workflow change). It is active in any repo that has a `vergil-exceptions.toml`;
without one it reports "not adopted" and passes. Adoption (§10) creates the file.
Once the fleet has adopted it, making it mandatory everywhere is a follow-on
decision.

## 7. Plug-ins

Each exception mechanism is a plug-in implementing one interface:

- **discover** — find inline sites (file, line, rules, `vrg-exc` ID) and
  config-level disables (rule, scope);
- **enabled** — report whether a rule is enabled for a path, to detect dead
  suppressions;
- **detect** — report whether the mechanism is present in the repo at all, so
  uncovered mechanisms can be reported (§6.2).

v1 plug-ins:

| Tool | Inline | Config-level |
|---|---|---|
| ruff | `# noqa:` | `ignore`, `extend-ignore`, `per-file-ignores` (rules never selected are not exceptions) |
| mypy | `# type: ignore[...]` | `disable_error_code`, per-module overrides |
| ty | `# ty: ignore[...]` | rules set to `ignore` in `[tool.ty.rules]` |
| coverage | `# pragma: no cover` | `exclude_lines` / `omit` |
| markdownlint | `<!-- markdownlint-disable ... -->` | disabled rules; `vergil.toml` `[markdownlint].ignore` path exclusions |
| shellcheck | `# shellcheck disable=` | `.shellcheckrc` `disable=` |
| yamllint | `# yamllint disable` | disabled rules |
| hadolint | `# hadolint ignore=` | `.hadolint.yaml` `ignored` |
| actionlint | — | config ignores |
| semgrep | `# nosemgrep:` | `--exclude-rule` flags in workflows and shell scripts (scope `["**"]`); `.semgrepignore` paths |
| trivy | — | `.trivyignore` entries (CVE IDs; `expires` required) |

Notes:

- A bare `# pragma: no cover` has no rule ID; its entry uses `rules = ["no-cover"]`.
  It is the one bare form that is not a blanket (§6.1 item 10), because coverage
  has nothing narrower to name.
- A per-repo semgrep `--exclude-rule` disables the rule repo-wide, so it is
  audited like any other config disable. The vergil-actions semgrep action has
  no exclusion input, so a direct flag is the only route a repo has. The fleet's
  `DEFAULT_EXCLUDED_RULES` are fleet defaults (§5.3).
- ansible-lint has no plug-in in v1; its `skip_list` and similar keys are
  reported as uncovered (§6.2).
- The exact config keys and syntaxes for each plug-in are confirmed at planning.
- Opaque policy files (e.g. trivy's Rego ignore policy) are covered by a
  `[[rule]]` entry naming the file in `scope`.
- Other languages' tools (TypeScript, C++, Rust, Go, Java) are added when a repo
  using them adopts the register. CodeQL has no in-source suppressions in use
  today; until a plug-in exists it is reported as uncovered if detected.
- Epic #94's `[[security.sarif-exception]]` design is superseded: SARIF-gate
  exceptions are expressed through the register and plug-ins above.

## 8. Agent workflow

**Rule** (in the consumer CLAUDE.md template, `claude_md_consumer.md`, and
referenced from `issue-implement` and `epic-implement`):

> Never add, widen, or reuse an exception — an inline suppression or a rule
> disable — without explicit human approval. Use `/vergil:exception-request`.

**Skill — `vergil:exception-request`:**

1. Try to fix the code; show the attempt.
2. If a fix is not practical, present to the human: tool, rule, location, why the
   fix is not practical, the proposed level (line or rule), the proposed scope
   (smallest directory set), and the proposed register entry (new, or an existing
   one it fits).
3. Wait for an explicit yes. Silence or ambiguity is a no.
4. On yes: write the register entry (or reuse the agreed one) and the inline ID
   with the wording both agreed.
5. On no: fix the code.

**Every new site needs approval**, including a site that reuses an existing
entry. Widening a `[[rule]]` scope is an exception change and also needs
approval. There is no "pre-approved" flag: a rule-level exception is the
pre-approval mechanism.

**Hook.** A PostToolUse hook on Edit/Write, modelled on
`detect-deprecation-warnings`, detects newly added suppression patterns and
reminds the agent to run the skill immediately.

## 9. Visibility

- **PR preview.** `vrg-exceptions diff --base <ref> --head <ref>` lists
  exceptions added, removed or changed (entries, scopes, sites). It reads both
  sides from git objects and never from the working tree, so it gives the same
  answer however the branch reaches submission — including worktree-free
  submission of a cloud-developed branch from the Mac's main worktree, where the
  branch exists only as `origin/<branch>`. `vrg-submit-pr` always passes both refs
  (e.g. `origin/develop` and `origin/<branch>`) and includes the result as an
  "Exceptions added or changed in this PR" section. The section always
  appears; when the diff ran and found nothing it says "(none)", so an empty
  result can never be mistaken for a check that did not run. A ref that cannot
  be resolved is a hard error, never an empty section.
- **Docs page.** `vrg-exceptions report` renders a Markdown page: the repo's
  rule-level exceptions with scopes, line-level entries with site counts,
  inherited fleet defaults (by tool and rule) with the repo's fleet revision,
  expiring entries (soonest first), and uncovered mechanisms. It is generated at
  docs-build time, never committed. This is secondary and may wait for the
  docs-build rework.

## 10. Adoption — cleanup, not grandfathering

There is no legacy bucket. Each repo's adoption is a **"kill the exceptions"**
series: a sequence of tasks whose only job is to purge exceptions while keeping
all functionality, ending in one task that documents what is left. It is run
with the human:

1. **Inventory** every exception, grouped by tool × rule × directory.
2. **Mechanical cleanup**: remove dead suppressions, apply auto-fixers
   (`ruff check --fix`, `ruff format`, equivalents), fix trivial cases.
3. **Judgment calls**, group by group, with the human: purge (fix the code — one
   purge task per group or small batch); keep and document; or disable the rule
   at the narrowest workable scope (a `[[rule]]` entry). Proposing a fleet
   default is also an option, but what becomes a fleet default is the human's
   decision alone. The series is also how the fleet register's contents get
   decided, repo by repo.
4. **Document** the remainder, tag inline sites with IDs, create
   `vergil-exceptions.toml`, and enable the gate. This includes the
   `secrets-inherit` `nosemgrep` that repo-init generated in a `[package]`
   repo's `cd.yml`: each repo that has it adds its own entry (§5.2).

New repos start adopted: repo-init writes `vergil-exceptions.toml` (empty, or
with the entry for the `nosemgrep` it generates) and tags that line, so the
audit is active from the first commit.

This epic runs the series for vergil-tooling only. Other repos run theirs later,
each under its own epic and adding the plug-ins its languages need.
vergil-containers (trivy) is the natural second.

## 11. Component boundaries

- `lib/exceptions/register.py` — parse and validate both registers (pure).
- `lib/exceptions/plugins/` — one module per tool, behind the §7 interface.
- `lib/exceptions/audit.py` — combine sites, config disables and registers into
  findings (pure, unit-testable with fixture repos).
- `lib/exceptions/managed.py` — managed-block render, parse, hash, sync.
- `lib/exceptions/report.py` — the docs page renderer.
- `bin/vrg_exceptions.py` — the `audit` / `sync` / `report` / `diff` CLI.
- `vrg-validate` — runs `audit` as a common check.
- `vrg-submit-pr` — calls `diff` for the preview section.
- `vrg-docs-stage` / `vrg-docs-patch-nav` — publish the docs page.
- `lib/repo_init.py` — seeds `vergil-exceptions.toml` and tags the generated
  `nosemgrep`.
- vergil-claude-plugin — the skill, the hook, and skill cross-references.
- A deployment task (release both, then verify the command, the hook and the
  skill in a consumer repo) gates closure; the releases themselves are
  human-gated.

## 12. Testing

- Register schema: every field rule in §5.1, including the conditional
  `issue`/`expires` requirement; the fleet register's no-`id`, `[[rule]]`-only
  and one-entry-per-tool-and-rule rules (§5.3).
- Each plug-in against fixture files: discovery, ID extraction (same-line,
  line-above and `.trivyignore` group forms), config-level disables (including
  semgrep `--exclude-rule`), enabled-rule detection.
- Audit: one test per failure in §6.1 (including blanket suppressions and a
  non-`EXC` marker ID) and per report item in §6.2.
- Managed blocks (ruff `extend-ignore` in `pyproject.toml`): hand-edit fails;
  older valid revision reports and passes; a missing block reports and passes;
  sync round-trips and leaves the rest of `pyproject.toml` byte-identical;
  repo-owned ignores beside the block are untouched.
- repo-init: a `[package]` release repo's generated `cd.yml` and seeded register
  pass the audit together; every other repo gets an empty register that passes.
- Expiry: an expired entry is reported and the audit still passes.
- `diff`: added, removed, widened-scope and new-site cases, run between two refs
  with no worktree for the head; an unresolvable ref is a hard error.
- Coverage stays at 100% under `vrg-validate`.

## 13. Follow-on and out of scope

This epic deliberately enables two follow-on brainstorms, seeded as tasks under
the epic:

- **Managed blocks beyond ruff, and fleet syncs generally.** Apply the §6.3
  mechanism to other tools with repo-local config, and rethink fleet syncs that
  today create flag days (CLAUDE.md delivery is the hardest case).
- **A scheduled review cadence.** How Vergil schedules and triggers recurring
  reviews and actions that need planning with the human — starting with the
  exception review (expired entries, promotion candidates, stale fleet blocks,
  the push toward zero exceptions) and extending to periodic refactoring
  reviews. This epic makes the register easy to review (`added`, `expires`, site
  counts, promotion candidates); the cadence itself is the follow-on.

It also files one follow-on task:

- **Revisit the generated `secrets: inherit`.** The `nosemgrep` repo-init writes
  works around a GitHub limitation (environment secrets reach a cross-repo
  reusable workflow only under `secrets: inherit`;
  `vergil-project/packages#6`). It is a workaround, not a fix. Investigate a
  design that needs no suppression; if one works, remove it from repo-init and
  from existing repos.

Out of scope:

- Defending against a malicious agent (Mimir).
- A shared, centrally shipped ruff baseline config beyond the managed fleet
  ignore block.
- Adoption beyond vergil-tooling.
- Making the gate mandatory in repos without a register.
