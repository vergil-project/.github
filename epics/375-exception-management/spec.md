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
- **Hard gates, not warnings** (#763) — with one deliberate, reasoned exception:
  fleet-default staleness (§6.3) is reported, never failed, because failing it
  turns every central change into a fleet-wide flag day.
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

[[rule]]                         # a rule-level entry adopting a fleet policy
id      = "EXC-002"
tool    = "ruff"
rules   = ["D400"]
scope   = ["**"]
fleet   = "VRG-004"              # reason inherited from the fleet register
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
| `id` | required | required | `EXC-NNN`, unique in the file |
| `tool` | required | required | a supported plug-in name (§7) |
| `rules` | required | required | non-empty list of the tool's rule IDs |
| `scope` | required | — | globs; repo-wide must be written `["**"]` explicitly |
| `reason` | required unless `fleet` | required | non-empty prose |
| `fleet` | optional | — | a `VRG-NNN` ID; supplies the reason |
| `added` | required | required | date |
| `expires` | optional | optional | date; required for `trivy` entries |
| `issue` | required iff `expires` | required iff `expires` | `owner/repo#N` |

- There is **no list of locations** in the register. The audit finds sites by
  scanning, so a stored list could only go stale.
- Many sites may reference one `[[inline]]` entry (many-to-one), but every new
  site still needs human approval (§8).

### 5.2 Inline linkage

Each inline suppression names its register ID in the same comment:

```python
subprocess.run(cmd, check=True)  # noqa: S603  vrg-exc: EXC-014
```

```bash
# shellcheck disable=SC2086  vrg-exc: EXC-020
```

For a tool that rejects trailing text in its suppression comment, the plug-in
reads the ID from a `vrg-exc: EXC-NNN` comment on the line immediately above.
Each plug-in's accepted form is verified during planning. For file-based
mechanisms such as `.trivyignore`, the ID sits in a comment on the line above the
entry.

### 5.3 Fleet register

A TOML file shipped as vergil-tooling package data (e.g.
`src/vergil_tooling/configs/exceptions.toml`), with `VRG-NNN` IDs and the same
`[[rule]]` shape. It documents every rule disabled by the shared configs Vergil
ships, plus fleet-wide policies that repos adopt in their own configs (via the
`fleet` field). It carries a monotonically increasing `revision`, and records the
SHA-256 of each managed-block body (§6.3) for every revision it has ever shipped,
so older blocks remain verifiable.

## 6. The audit — `vrg-exceptions audit`

### 6.1 Failures (exit non-zero)

1. An inline suppression with no `vrg-exc:` ID.
2. An ID not present in the register (or a `fleet` reference not in the fleet
   register).
3. A site whose suppressed rule is not in its entry's `rules`.
4. A site whose entry is `[[rule]]` rather than `[[inline]]`, or vice versa.
5. A **dead** suppression — one that suppresses a rule the tool does not have
   enabled, or (where the tool can report it) suppresses nothing. Where the tool
   offers this natively (ruff `RUF100`, mypy `warn_unused_ignores`), the plug-in
   uses it.
6. A rule disabled in the repo's effective tool config that is neither a fleet
   default nor covered by a `[[rule]]` entry whose `scope` matches.
7. A `[[rule]]` entry that matches no disabled rule (stale register entry), and
   an `[[inline]]` entry referenced by no site.
8. A managed block whose content hash does not match the revision it claims
   (hand-edited; §6.3).
9. A register that fails schema validation (§5.1 field rules).
10. An entry past its `expires` date.

### 6.2 Reported, not failed

- Managed blocks behind the current fleet revision (§6.3).
- `[[inline]]` entries with several sites — promotion candidates (§4).
- Entries expiring within 30 days.
- Exception mechanisms found in the repo that no plug-in covers yet.

The report is written to stdout and, in CI, to the job summary.

### 6.3 Managed blocks and fleet revisions

How a fleet default reaches a repo depends on the tool:

- **Tool supports include/extend** (markdownlint, yamllint, ESLint, ruff
  `extend`): the repo config references the shipped file; changes arrive with the
  vergil-tooling version.
- **Tool has no include mechanism:** the setting is copied into the repo config as
  a managed block, the way `vrg-gitignore-sync` maintains `.gitignore`:

  ```text
  # >>> vergil-managed: fleet-exceptions rev 7 sha256:… (do not edit) >>>
  …
  # <<< vergil-managed <<<
  ```

The two kinds of drift are treated differently:

- **Edited by hand → fail.** The body's hash does not match the hash the fleet
  register recorded for the revision the block claims. This catches an agent
  editing a fleet exception in place, and never fails because of a change made
  elsewhere.
- **Behind the current revision → report.** Safe, because a stale block never
  produces an undocumented exception: a newly disabled fleet rule is still
  enforced (stricter), and a re-enabled one is still documented by the old
  revision. `vrg-exceptions sync` brings the block up to date.

Repos follow a vergil-tooling release *line* (`vergil = "v2.1"`), so a check
against the current revision would break every repo on every central change.
The revision stamp is what decouples them.

### 6.4 Activation

The audit runs as a common check in `vrg-validate` (locally and in PR CI, with no
workflow change). It is active in any repo that has a `vergil-exceptions.toml`;
without one it reports "not adopted" and passes. Adoption (§9) creates the file.
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
| coverage | `# pragma: no cover` | `exclude_lines` / `omit` |
| markdownlint | `<!-- markdownlint-disable ... -->` | disabled rules |
| shellcheck | `# shellcheck disable=` | `.shellcheckrc` `disable=` |
| yamllint | `# yamllint disable` | disabled rules |
| hadolint | `# hadolint ignore=` | `.hadolint.yaml` `ignored` |
| actionlint | — | config ignores |
| semgrep | `# nosemgrep:` | `--exclude-rule` |
| trivy | — | `.trivyignore` entries (CVE IDs; `expires` required) |

Notes:

- A bare `# pragma: no cover` has no rule ID; its entry uses `rules = ["no-cover"]`.
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

- **PR preview.** `vrg-exceptions diff --base <branch>` lists exceptions added,
  removed or changed (entries, scopes, sites). `vrg-submit-pr` includes it as an
  "Exceptions added or changed in this PR" section in the preview, omitted when
  empty.
- **Docs page.** `vrg-exceptions report --format markdown` renders the repo's
  rule-level exceptions with scopes, line-level entries with site counts,
  inherited fleet defaults and the repo's fleet revision, expiring entries (soonest
  first), and uncovered mechanisms. It is generated at docs-build time, never
  committed. This is secondary and may wait for the docs-build rework.

## 10. Adoption — cleanup, not grandfathering

There is no legacy bucket. Each repo's adoption is a joint human + agent task,
roughly half a day per repo:

1. **Inventory** every exception, grouped by tool × rule × directory.
2. **Mechanical cleanup**: remove dead suppressions, apply auto-fixers
   (`ruff check --fix`, `ruff format`, equivalents), fix trivial cases.
3. **Judgment calls**, group by group, with the human: fix the code; keep and
   document; or disable the rule at the narrowest workable scope (itself a
   `[[rule]]` entry, or a fleet default if it applies everywhere).
4. **Document** the remainder, tag inline sites with IDs, create
   `vergil-exceptions.toml`, and enable the gate.

This epic adopts in vergil-tooling only. Other repos follow as tasks under later
work, each adding the plug-ins its languages need. vergil-containers (trivy) is
the natural second.

## 11. Component boundaries

- `lib/exceptions/register.py` — parse and validate both registers (pure).
- `lib/exceptions/plugins/` — one module per tool, behind the §7 interface.
- `lib/exceptions/audit.py` — combine sites, config disables and registers into
  findings (pure, unit-testable with fixture repos).
- `lib/exceptions/managed.py` — managed-block render, parse, hash, sync.
- `bin/vrg_exceptions.py` — the `audit` / `sync` / `report` / `diff` CLI.
- `vrg-validate` — runs `audit` as a common check.
- `vrg-submit-pr` — calls `diff` for the preview section.
- vergil-claude-plugin — the skill, the hook, and skill cross-references.

## 12. Testing

- Register schema: every field rule in §5.1, including the conditional
  `issue`/`expires` and `reason`/`fleet` requirements.
- Each plug-in against fixture files: discovery, ID extraction (same-line and
  line-above forms), config-level disables, enabled-rule detection.
- Audit: one test per failure in §6.1 and per report item in §6.2.
- Managed blocks: hand-edit fails; older valid revision reports and passes; sync
  round-trips.
- `diff`: added, removed, widened-scope and new-site cases.
- Coverage stays at 100% under `vrg-validate`.

## 13. Out of scope

- Defending against a malicious agent (Mimir).
- The recurring quarterly exception review — belongs to a scheduled-operations
  epic alongside periodic refactoring reviews; this epic makes the register easy
  to review (`added`, `expires`, site counts, promotion candidates).
- Rethinking fleet syncs in general (they currently create flag days).
- A shared, centrally shipped ruff baseline config.
- Adoption beyond vergil-tooling.
- Making the gate mandatory in repos without a register.
