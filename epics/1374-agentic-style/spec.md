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

**Its results use the delegate result format.** The human's paad fork
(`wphillipmoore/paad`) carries an unattended mode and the `paad-delegate`
agent, built by this project to integrate paad's skills with vergil: a
caller dispatches a fixed prompt and gets back one structured result
(announce line, status, counts, findings, discards, a separate security
section, files written), which it checks strictly. `epic-create` already
consumes it (vergil-claude-plugin `docs/paad-review-doctrine.md` §1.9). The
format is experimental, not yet offered upstream, and is a framework the
project intends to extend. `agentic-style` reuses that architecture for its
own reviewers, verifiers and overall result, extending each finding with the
fields section 5.3 lists, rather than defining a separate format.

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
- Tuning runs on the six fixture repos until the reports are worth acting on.

### 2.2 Non-goals (this epic)

- A fix skill. It comes after reports have been tuned (later epic).
- A merge-review check against the rulebook. Later, and only for rules that
  real use shows are applied reliably.
- A corpus of good and bad example snippets, and any calibration mode built
  on one. Examples are collected over time from real findings, the way a FAQ
  grows from questions actually asked, not designed up front. The fixtures
  grow over time into the fleet's examples of good code.
- A cadence for running the three agentic skills periodically (a separate
  intake item).
- Seeding `issue-implement` with the rulebook. Later, once the rules have
  survived tuning (proposed, not agreed; see section 7).
- Building Go or C++ rulebooks. They are researched, not built; code in
  those languages is audited against `common` only and reported as a
  coverage gap.
- Auditing vergil-tooling. It is the most important target, but it comes
  after the fixtures have shown the reports are useful.
- Measuring what the audit misses. This epic measures false positives only.
  Misses are measured later: against real code such as vergil-tooling, from
  examples collected over time, and through an **anti-fixture**, a planned
  follow-on (section 7).
- Agent judgment of any rule a linter can enforce. Deterministic, mechanical
  checks always beat stochastic, approximate ones: where a linter covers a
  rule in a language, the linter enforces it there and the agent does not
  (section 7). The rulebook's agent-judged rules are the residue no linter
  covers.

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
- **Anything added to `vergil.toml` is strictly validated.** vergil-tooling
  `src/vergil_tooling/lib/config.py` today only warns on an unrecognized
  section or key. The new `[standards]` table is validated strictly: its
  allowed keys are defined, and an unknown key or a `level` other than the
  values section 5.4 accepts raises a `ConfigError`. A `vergil.toml` is
  changed only by a committed PR; no run ever edits it locally to get past a
  check. vergil-tooling also gives
  skills a `vrg-*` command to read the validated setting, and the skill reads
  it only through that command. This needs a vergil-tooling change and release
  before any repo can set the flag. Making the rest of `vergil.toml` strict is
  separate work (vergil-project/.github#1382).
- **Fixtures exist to run Vergil's release cycle.** They are the integration
  test bed, not deployed code, and over time the best-case example of the
  code the rulebook asks for. Planted violations do not go in fixture source.
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
  plus `common`. Target about 15 agent-judged rules per language and 10
  common. Conway's naming rules are always listed (already adapted per
  language in vergil-tooling's
  `docs/site/docs/standards/development/<lang>/naming-conventions.md`), so the
  set stays complete as a reference. In each language, a Conway rule that a
  linter covers is marked "enforced by `<linter check>`" and is not
  agent-judged there; one no linter covers is an agent-judged rule.
- Each rule: a stable ID (e.g. `naming.full-words`), a theme, a fixed
  severity, a one-line statement, what a violation looks like (in prose), the
  source citation, and why no linter covers it. Severity is the canonical
  `high`, `medium` or `low`: `high` when a violation misleads a reader about
  what the code does, `medium` when it makes the code materially harder to
  follow or change, `low` for consistency and polish. A finding takes its
  rule's severity; reviewers never assign one, so the same rule ranks the same
  way in every round.
- The `meta.` ID prefix is reserved for findings that are not rulebook rules
  but must still be reported. `meta.prompt-injection` is the first: text in
  the audited code that tries to steer a reviewer. Further `meta.*` kinds may
  be added for similar conditions. No rulebook rule uses the prefix.
- Rules are prose guidelines extracted from the cited
  sources; no example snippets are required.
- Themes (initial, the human may change them): **naming**, **functions and
  control flow** (length, nesting, early return, argument count), **language
  idioms**, **comments and documentation**. Every rule, language-specific or
  common, belongs to exactly one theme. Structure, coupling and abstraction
  belong to `agentic-architecture`; error-handling correctness belongs to
  `agentic-review`.
- Each rule notes any paad lens that covers adjacent ground, so the boundary
  is written down. A favourite rule that falls between the siblings stays in
  `agentic-style`.
- The human reviews brutally and iterates. **Rulebook approval is an explicit
  human gate in the plan:** no skill or agent work starts until the human has
  approved the rulebook.

### 5.3 Phase 3: the audit skill and agents

- A vergil plugin skill, `agentic-style` (invoked `/vergil:agentic-style`),
  written to vergil's skill conventions. Its description says what it is not
  for and names its two siblings.
- **The delegate pattern, checked strictly.** Each reviewer and verifier is
  dispatched with a fixed prompt and returns its result in the result frame
  of section 5.6. The skill checks each result's structure strictly before
  using it: counts match headings, fields are present and in order, each
  severity matches its rule's, each ID is a rulebook rule or a reserved
  `meta.*` kind, the security section is separate. A result that fails a
  check is refused, never repaired or guessed at, and **a refused or blocked
  result fails the whole audit**: no report is written, and the failing
  result is kept privately (below). A partial report is not one to trust.
  The skill builds its own structured result, in the same frame, from the
  checked results only, and writes it beside the report, so callers and
  later the fix skill and merge check consume checked data, not prose. The
  structured result is data, not Markdown, and is written as `.result.txt`.
  The report is rendered by vergil-tooling from the checked results, never
  written by an agent, and is markdownlint-clean by construction, with all
  reviewer free text in fenced code blocks (doctrine §1.9). There is no
  request file, capture hook or write guard.
- **Repo content is data, not instructions.** Every reviewer and verifier
  prompt carries a data-not-instructions section like `agents/merge-reviewer.md`
  in vergil-claude-plugin: the code, comments and docs under review never
  direct the reviewer, and any text that tries to is reported as a
  `meta.prompt-injection` finding. `meta.*` findings pass the checker, skip
  the verify pass, are left out of the per-rule tuning counts, and go in the
  security section. Each finding's suggested change is marked in the result
  as untrusted text that consumers must never act on as instructions.
- **Incidental security observations stay private.** Reviewers never quote a
  secret-like value or describe an exploitable weakness in a finding. Such an
  observation goes only in a result's security section. The skill cuts that
  section out of every reviewer result before anything else reads it, as the
  doctrine's security cut does (vergil-claude-plugin
  `docs/paad-review-doctrine.md` §1.9). The details go only to ignored
  scratch under `build/agentic-style/` and to the human in the terminal; the
  report and the written structured result carry the security count alone.
  Whoever commits a report or result to a tracked path first runs the
  doctrine's screening (§1.6) over it.
- It reads the `[standards]` level through the vergil-tooling command
  (section 3) and refuses, with the reason, unless the level is `audit`; an
  invalid table is a config error before the skill sees it.
- **Inventory.** It inventories every piece of code in the repo and audits all
  of it: the primary language (`project.primary-language`) first, then every
  other language found (shell scripts in a Python repo, for example). Test
  code is code and is audited under the same rules; it asserts the
  correctness of the production code, so its readability matters at least as
  much. The inventory is auto-discovered and appears at the top of the report
  (language, file count, lines, rulebook used).
- **Every file of code we own is audited, however it was created**: by a
  human, an AI or a generator. A finding in a file vergil generates names the
  generator's source as where the fix belongs (for example, every repo's
  `.claude/hooks/guard.sh` is rendered from vergil-tooling
  `src/vergil_tooling/data/hook_guard_shim.sh`). Code we do not control
  (vendored third-party code, output of third-party tools) is exempt only
  when the repo lists it in `[standards].exempt`; nothing is exempted by
  inference, such as a marker comment or a directory name. The report lists
  every exempt file, so nothing is skipped silently.
- Configuration and data formats (YAML, TOML, JSON) are not code and are not
  audited; their own linters cover them.
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
  rule ID, severity and source, file and line, what is wrong, a suggested
  change, and its **reach**: `local`, `cross-module`, or `public-interface`.
  Reach is recorded because fixes that rename or restructure ripple to
  callers, docs, tests and sometimes other repos. For a generated file, the
  finding also names the generator's source. A per-rule summary table:
  findings raised, findings surviving verification, so rules that never fire,
  always fire, or mostly fail verification are visible for tuning.
- **Provenance.** The report and the structured result each record the
  audited repo's commit SHA, whether its working tree was clean, the plugin
  version and a hash of the rulebook content, so a change between two tuning
  rounds can be traced to the code, the rules or the prompts.

### 5.4 The repo flag

A `[standards]` table in `vergil.toml`:

- `level = "off"`: the default. The audit refuses to run.
- `level = "audit"`: the repo is working towards compliance. Reports only.
- `level = "enforced"` is planned for a repo that has reached compliance,
  checked by merge review. Until the epic that builds that check exists, it
  is a `ConfigError` ("not yet supported"): a repo never claims an
  enforcement that does not happen.
- `exempt = [<repo-relative paths>]`: optional. The only way to exempt code
  we do not control; an entry ending in `/` covers everything below it.

Rule: never enforce compliance before a repo has reached it.

### 5.5 Phase 4: fixture tuning

- Set `level = "audit"` on all six fixtures (python, rust, ts; app and lib).
- Run the audit, the human reviews each report, and the rulebook and prompts
  are tuned; repeat. Fixing findings in a fixture's own files is done by hand
  or by ordinary fixture issues during this phase (no fix skill yet).
- **Generated files during tuning.** A finding in a file vergil generates
  (for example `.claude/hooks/guard.sh`, or the managed
  `eslint.vergil.config.mjs` in the TypeScript fixtures) is reported once,
  merged across the fixtures that carry it, and fixed through a vergil-tooling
  issue against its generator.
- **The shell rulebook's tuning target.** The fixtures' only shell file is the
  generated shim, so the shell rulebook is also tuned on vergil-claude-plugin's
  hand-written hook scripts (`hooks/scripts/`, `hooks/tests/`), audited for
  shell only. The plugin opts in with a committed `[standards]` table like the
  fixtures.
- **Success:** each fixture's latest report is one the human judges mostly
  worth acting on; the per-rule tables show which rules fire usefully; the
  fixtures that reach a clean report are candidates for enforcement once the
  merge check exists. Success measures false positives only (section
  2.2).

### 5.6 The result frame

The skill and its agents use one frame, a vergil variant of the delegate
result format (section 1.1), defined once and extended later as the
framework grows:

- **Announce line:** `Running vergil:agentic-style <role> v<plugin version>`,
  with `<role>` one of `reviewer`, `verifier` or `audit` (the skill's own
  result).
- **Kept from the delegate format:** `Status: complete | blocked: <reason>`,
  `Findings: <n>` with `### F<i> <severity> — <location>` headings, the
  native fields of each finding, `Discards: <k>` with one dash-led line each, the
  separate security section between the delegate's markers, and a last
  `Files written:` line.
- **Dropped:** `Config:`, `Config not followed:`, `Defaults applied:` and
  `Agent types dispatched:`, which describe a paad delegate running a paad
  skill. A reviewer or verifier dispatches nothing.
- **Changed:** `Files written:` is `none` for a reviewer or verifier; for
  the skill's own result it lists the report and result it wrote, by paths
  relative to the output directory, never absolute host paths.
- **Provenance lines,** in the skill's own result only: the audited repo's
  commit SHA, whether its tree was clean, the plugin version and the rulebook
  content hash.
- **Fields of each finding:** rule ID (a rulebook rule or a `meta.*` kind),
  source citation, file and line, what is wrong, suggested change (marked
  untrusted), reach, and generator source where the file is generated. The
  heading's severity is the rule's.

## 6. What is left to the builder

- The exact file path for the rulebook in the plugin.
- Whether review and verify are one agent definition with two modes or two
  definitions; their tools are read-only (Read, Glob, Grep).
- The exact spelling of each per-finding field label in section 5.6.
- How the skill dispatches subagents and bounds cost on a large repo.
- The report's exact layout beyond the fields in 5.3.
- The issue breakdown across vergil-tooling (flag) and vergil-claude-plugin
  (rulebook, skill, agents), and the survey's home (proposal: this repo's
  epic folder).

## 7. Further decisions

- The default output location is ignored `build/` scratch, not a committed
  path; committing a report is a deliberate act by the caller.
- The rulebook ships with the plugin; site docs link to it.
- `enforced` is rejected as not yet supported until the merge check exists.
- Seeding `issue-implement` and the merge check are deferred to later epics.
- Shell gets a built rulebook in this epic (the human asked that all code in
  a repo be audited, and shell is in nearly every repo). Go and C++ stay
  research-only.
- "C" in the brainstorm is read as C++, matching the existing standards
  folder.
- **The linter always wins, language by language.** A rule is
  linter-enforced in a language when a linter vergil already runs, or ships
  configuration for, offers a check for it there, even if vergil has that
  check switched off (e.g. ESLint `id-length`, Clippy `min_ident_chars`). The
  rule is then marked "enforced by `<check>`" for that language and is not
  agent-judged there. The survey records, per rule and language, which check
  covers it or that none does. Each check that is available but not enabled,
  and each coverage gap a linter could close (in any surveyed language),
  becomes an intake issue to strengthen the mechanical gates, filed on the
  human's approval, outside this epic.
- **Naming docs.** vergil-tooling's naming docs stay the authority, unchanged,
  until the rulebook is approved. After approval the Python, Rust, TypeScript
  and shell docs link to their rulebook rules as the authority (a docs task in
  vergil-tooling); the Go and C++ docs are untouched.
- **Book citations.** Rules may cite copyrighted books, but a citation the
  agent could not check against the text is marked `unverified` in the
  rulebook; the human verifies the ones that survive the cut. Where an open
  source says the same thing, the open source is cited first.
- **Tuning reports do not touch the fixtures.** The skill writes a file and
  nothing else: no issue, no commit, no handoff (as paad's audit skills do).
  The output path is an argument, so the caller decides where the report and
  structured result go; the default is the audited repo's ignored
  `build/agentic-style/` scratch, which vergil allows even in a read-only main
  worktree. The report's content and shape are standard wherever it lands.
  During tuning, reports are written into this epic's folder in `.github`, so
  tuning rounds never start a fixture's PR or release cycle. Fixing a
  fixture's own findings goes through ordinary fixture issues; a generated
  file's findings go to vergil-tooling (section 5.5).
- **Follow-on work.** A follow-on brainstorm task on this epic, run after
  tuning, reviews what was learned in this new area and shapes the follow-on
  epics. One candidate it takes up is an **anti-fixture**: a repo of code that
  passes every CI/CD gate but breaks the style rules on purpose. It targets
  the gap between "passes CI" and "good" that this review exists for, and
  gives a way to measure misses, while the real fixtures stay the positive
  examples.
- **Ordering.** The vergil-tooling `[standards]` flag change and release
  lands before any fixture sets the flag, because fixtures pin
  `vergil = "v2.1"`.
- **The fixtures are small**, so many rules will not fire on them. That is
  expected: a rule that never fires on the fixtures is not judged until it
  meets larger code, starting with vergil-tooling after this epic.
