# agentic-style: agent review against published coding standards — design

- **Epic:** vergil-project/.github#1374
- **Planning issue:** vergil-project/.github#1373 (review history is in its comments)
- **Status:** draft, in design review

## 1. Intent

Vergil's premise is that "is this code good?" can be answered mechanically:
linters, formatters, type checkers, spell and shell checks, CI gates. Those
are in place and stay the first line. This epic adds the one layer that is
not mechanical: an agent review that holds code to **published, respected
coding standards**, the way Damian Conway's *Perl Best Practices* shaped the
fleet's Python conventions.

The why: the goal is code nobody has to read, but code that, when a human
does read it, is plainly maintainable and comprehensible, never "something
only an AI could follow". Today the standards are implicit (the model knows
them, maybe) or written but unenforced (the Conway-derived naming docs in
vergil-tooling). This makes them explicit, cited and checked.

The first deliverable is a **report**, not a fix and not a gate. The human
wants to see what the reports look like and tune them before anything acts
on them.

### 1.1 The agentic family

`agentic-style` is designed as the third member of a family with paad's
`agentic-review` and `agentic-architecture`, each with its own lens and no
overlap:

- **`agentic-review` (paad), on a branch:** bugs: logic, error handling,
  concurrency, security, contracts, spec compliance.
- **`agentic-architecture` (paad), on a codebase:** structure: boundaries,
  coupling, abstraction (its 34 flaw types).
- **`agentic-style` (vergil), on a codebase:** the code a human reads line by
  line: naming, functions and control flow, language idioms, comments and
  docs, against cited published standards.

Verified against the human's fork (`wphillipmoore/paad`,
`plugins/paad/skills/`): `agentic-review` states "Not for codebase structure,
not for code style", and its logic lens says not to report "style, naming,
formatting, or readability issues". `agentic-architecture` covers
over-abstraction, leaky abstractions, god objects, low cohesion, magic
numbers, dead code and inconsistent error conventions; `agentic-review`'s
error-handling lens covers swallowed exceptions. `agentic-style` leaves all of
those to them.

It is the **lightweight** sibling, and **vergil-native**: inspired by paad
(hence the name), loosely coupled to it, and not intended for contribution to
paad. It borrows the reviewers → verify → report pattern, but none of paad's
branch scoping, blame routing, backlog machinery or `paad/` directory
conventions. It is built to look like a vergil skill and to fit vergil's
tooling and paths.

A key reason: epic-create carries awkward workarounds to exchange data with
paad's pushback and alignment skills through the file system. `agentic-style`
avoids that by design: its data hand-offs are explicit, versioned contracts
(section 5.3), so a caller or a wrapper can consume its output without
scraping. The human may later wrap paad's two skills in a vergil wrapper that
packages their output the same way; that is outside this epic.

The longer-term aim is that the three are run together, periodically, on code
that is meant to last (intake item vergil-project/.github#1372, routed to
planning).

## 2. Goals, non-goals, anti-goals

### 2.1 Goals

- A cross-language research survey of coding-standard sources for Python,
  Rust, TypeScript, Go, C++ and shell, so the human can compare what the
  standards look like across languages.
- A curated, cited **rulebook** for Python, Rust, TypeScript, shell and a
  cross-language set, drafted by the agent and cut hard by the human.
  Shell is built, not only researched, because nearly every repo carries
  scripts beside its primary language.
- An audit skill that applies the rulebook to a repo with themed reviewer
  subagents and a verify pass, and writes a report.
- A per-repo opt-in flag, so the audit runs only where a repo has chosen it.
- Calibration snippets per rule, so rule reliability can be measured.
- Tuning runs on the six fixture repos until the reports are worth acting on.

### 2.2 Non-goals (this epic)

- A fix skill. It comes after reports have been tuned (later epic).
- A merge-review check against the rulebook. Later, and only for rules that
  calibration shows are applied reliably.
- A cadence for running the three agentic skills periodically (a separate
  intake item).
- Seeding `issue-implement` with the rulebook. Later, once the rules have
  survived tuning (proposed, not agreed; see section 7).
- Building Go or C++ rulebooks. They are researched, not built; code in
  those languages is audited against `common` only and reported as a
  coverage gap.
- Auditing vergil-tooling. It is the most important target, but it comes
  after the fixtures have shown the reports are useful.
- Any rule a linter already enforces. If a mechanical check exists, it is
  used instead; the rulebook covers judgment only.

### 2.3 Anti-goals (a failure even if it works)

- Reports full of noise or taste, so findings get ignored.
- Rules with no cited source: "the agent's opinion" dressed as a standard.
- Copying copyrighted book text into prompts or the repo.
- Enforcing compliance on a repo that has not reached it.

## 3. Constraints

- **Licensing.** Books (*Perl Best Practices*, *Effective TypeScript*,
  *A Philosophy of Software Design*, ...) are copyrighted: cite them and write
  rules in our own words, never quote passages at length. Open guides can be
  adapted under their stated licenses; the survey records each license.
- **Sources are load-bearing.** Every rule cites a checkable source with a
  URL or book and section. The survey separates what a source says (data)
  from the agent's synthesis (judgment).
- **Where the skill runs.** The skill ships in `vergil-claude-plugin` and runs
  against other repos, so the rulebook must ship with the plugin. The existing
  naming docs in vergil-tooling's docs site link to the rulebook rather than
  hold a second copy.
- **`vergil.toml` is strictly validated.** `vergil-tooling`
  `src/vergil_tooling/lib/config.py` holds an allowed-key set per table, so a
  new `[standards]` table needs a vergil-tooling change and release before any
  repo can set it.
- **Fixtures exist to run Vergil's release cycle.** Planted violations do not
  go in fixture source; the calibration corpus lives with the rulebook.
- Workflow conventions apply as usual: worktrees, `vrg-git` / `vrg-gh`,
  `vrg-container-run -- vrg-validate`, report-ready handoff.

## 4. Approaches considered

1. **One reviewer per language** with the full rulebook. Cheapest and
   simplest, but with ~25 rules in one context attention spreads thin, rules
   get missed, and per-rule tuning data is unreliable.
2. **Themed reviewers plus a verify pass** (chosen). Rules are grouped by
   theme; each group gets its own reviewer subagent; a verify pass rechecks
   every finding against the rule text and drops those that do not hold.
   Mirrors paad's `agentic-review`, which has worked well in this fleet.
   More agents per run, but thorough, and per-rule data means something.
3. **One reviewer per rule.** Sharpest, but 25+ agents per language per run;
   does not scale and is premature before rules are proven.

**Chosen: 2.** The human's reason: the rules are many and varied, the goal is
thoroughness, and several focused contexts beat one large one, which misses
things.

## 5. The design, as decided

### 5.1 Phase 1: research survey

A report covering Python, Rust, TypeScript, Go, C++ and shell. Per language:

- candidate sources (style guides, API guidelines, books), with URL, author,
  license, and whether the source is open or copyrighted;
- what each covers (naming, structure, errors, API design, docs, ...);
- which parts are already enforced by the linters Vergil runs for that
  language (ruff, mypy/ty, clippy, rustfmt, eslint, tsc strict, ...), and so
  are out of scope for the rulebook;
- the residue: judgment rules worth considering.

Plus cross-language design sources (Conway's principles generalised,
Ousterhout's *A Philosophy of Software Design*, and others the survey finds).
Every claim about a source is checked against the source; anything unverified
is marked so.

### 5.2 Phase 2: the rulebook (agent drafts, human cuts)

- Markdown: one file per language (`python`, `rust`, `typescript`, `shell`)
  plus `common`. Target about 15 rules per language and 10 common. Conway's naming
  rules are always included (already adapted per language in vergil-tooling's
  `standards/development/<lang>/naming-conventions.md`; these become rules).
- Each rule: a stable ID (e.g. `naming.full-words`), a theme, a one-line
  statement, what a violation looks like, the source citation, why no linter
  covers it, and at least one bad and one good snippet (the calibration
  corpus).
- Themes (initial, the human may change them): **naming**, **functions and
  control flow** (length, nesting, early return, argument count), **language
  idioms**, **comments and documentation**. Every rule, language-specific or
  common, belongs to exactly one theme. Structure, coupling and abstraction
  belong to `agentic-architecture`; error-handling correctness belongs to
  `agentic-review`.
- Each rule notes any paad lens that covers adjacent ground, so the boundary
  is written down. A favourite rule that falls between the siblings stays in
  `agentic-style`.
- The human reviews brutally and iterates; the rulebook is approved before the
  skill is built.

### 5.3 Phase 3: the audit skill and agents

- A vergil plugin skill, `agentic-style` (invoked `/vergil:agentic-style`),
  written to vergil's skill conventions. Its description says what it is not
  for and names its two siblings.
- **Contracts, not scraping.** Following the `merge-reviewer` model (a
  versioned JSON request file in, a structured result back), each reviewer and
  verifier receives a versioned request and returns a versioned, structured
  result. The skill also writes a machine-readable findings file beside the
  Markdown report, under a versioned schema (e.g. `agentic-style-findings/1`),
  so callers, wrappers and later the fix skill and merge check consume data,
  not prose.
- It reads the repo's `vergil.toml` and refuses with a clear message if the
  `[standards]` level is `off` or absent.
- **Inventory.** It inventories every piece of code in the repo and audits all
  of it: the primary language (`project.primary-language`) first, then every
  other language found (shell scripts in a Python repo, for example). Test
  code is code and is audited under the same rules, with no exceptions; it
  asserts the correctness of the production code, so its readability matters
  at least as much. The inventory is auto-discovered and appears at the top
  of the report (language, file count, lines, rulebook used).
- It loads the rulebook for each language found, plus `common`. A language
  found with no rulebook is audited against `common` only, and the report
  says so plainly as a coverage gap, never silently.
- **Review pass.** For each language present and each theme, one read-only
  reviewer subagent gets that theme's rules (language plus common) and the
  repo's source for that language. It reports findings, each tied to a rule
  ID.
- **Verify pass.** Each finding is rechecked against the rule's text and the
  code by a separate subagent; findings that do not hold are dropped, and the
  drop is counted.
- **Report.** Dated, written to the output path (section 7). Each finding:
  rule ID and source, file and line, what is wrong, a suggested change, and
  its **reach**: `local`, `cross-module`, or `public-interface`. Reach is
  recorded because fixes that rename or restructure ripple to callers, docs,
  tests and sometimes other repos. A per-rule summary table: findings raised, findings
  surviving verification, so rules that never fire, always fire, or mostly
  fail verification are visible for tuning.
- **Calibration mode.** The skill can run against the rulebook's own snippet
  corpus and report, per rule, whether each bad snippet was flagged and each
  good one left alone. This is how a rule's reliability is measured. It is an
  LLM evaluation, not deterministic, and is not part of `vrg-validate`.

### 5.4 The repo flag

A `[standards]` table in `vergil.toml`:

- `level = "off"`: the default. The audit refuses to run.
- `level = "audit"`: the repo is working towards compliance. Reports only.
- `level = "enforced"`: the repo has reached compliance. Reserved: in this
  epic it behaves as `audit`; a later epic makes merge review check against
  it.

A repo moves to `enforced` only by an explicit edit to its `vergil.toml`.
Rule: never enforce compliance before a repo has reached it.

### 5.5 Phase 4: fixture tuning

- Set `level = "audit"` on all six fixtures (python, rust, ts; app and lib).
- Run the audit, the human reviews each report, and the rulebook and prompts
  are tuned; repeat. Fixing findings in the fixtures is done by hand or by
  ordinary issues during this phase (no fix skill yet).
- **Success:** each fixture's latest report is one the human judges mostly
  worth acting on; calibration shows which rules are reliable; the fixtures
  that reach a clean report are ready to be flipped to `enforced` when the
  merge check exists.

## 6. What is left to the builder

- The exact file path for the rulebook in the plugin.
- Whether review and verify are one agent definition with two modes or two
  definitions; their tools are read-only (Read, Glob, Grep), as with
  `merge-reviewer`.
- How the skill dispatches subagents and bounds cost on a large repo.
- The report's exact layout beyond the fields in 5.3.
- The issue breakdown across vergil-tooling (flag) and vergil-claude-plugin
  (rulebook, skill, agents), and the survey's home (proposal: this repo's
  epic folder).

## 7. Further decisions

- The default output location is ignored `build/` scratch, not a committed
  path; committing a report is a deliberate act by the caller.
- The rulebook ships with the plugin; site docs link to it.
- Calibration snippets live with the rulebook, not in fixture source.
- `enforced` is reserved and behaves as `audit` for now.
- Seeding `issue-implement` and the merge check are deferred to later epics.
- Shell gets a built rulebook in this epic (the human asked that all code in
  a repo be audited, and shell is in nearly every repo). Go and C++ stay
  research-only.
- "C" in the brainstorm is read as C++, matching the existing standards
  folder.
- **Mechanical first means "offered", not "turned on".** If a linter offers
  a check (e.g. ruff pep8-naming, clippy pedantic, eslint naming-convention)
  the rule stays out of the rulebook even if Vergil has it switched off. The
  survey lists each such check as "available, not enabled", and each becomes
  an intake issue to enable it in vergil-tooling, outside this epic.
- **Naming docs.** vergil-tooling's naming docs stay the authority, unchanged,
  until the rulebook is approved. After approval the Python, Rust and
  TypeScript docs link to their rulebook rules as the authority (a docs task
  in vergil-tooling); the Go, C++ and shell docs are untouched.
- **Book citations.** Rules may cite copyrighted books, but a citation the
  agent could not check against the text is marked `unverified` in the
  rulebook; the human verifies the ones that survive the cut. Where an open
  source says the same thing, the open source is cited first.
- **Tuning reports do not touch the fixtures.** The skill writes a file and
  nothing else: no issue, no commit, no handoff (as paad's audit skills do).
  The output path is an argument, so the caller decides where the report and
  findings file go; the default is the audited repo's ignored
  `build/agentic-style/<date>-<repo>.{md,json}` scratch, which vergil allows
  even in a read-only main worktree. The report's content and shape are
  standard wherever it lands. During tuning, reports are written into this
  epic's folder in `.github`, so tuning rounds never start a
  fixture's PR or release cycle. Fixing fixture findings goes through ordinary
  fixture issues.
- **Ordering.** The vergil-tooling `[standards]` flag change and release
  lands before any fixture sets the flag, because fixtures pin
  `vergil = "v2.1"`.
- **Success leans on calibration as well as fixture reports**, since the
  fixtures are small (about 6,500 lines over six repos) and many rules will
  not fire on them. A rule is judged by its calibration results first and its
  fixture findings second.
