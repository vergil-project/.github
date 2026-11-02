# Choose agentic-review specialists per change

- **Epic:** vergil-project/.github#826
- **Planning issue:** vergil-project/.github#812 (review history is in its
  comments)
- **Status:** draft
- **Repos:** the paad fork (`wphillipmoore/paad`, the `Lenses:` input),
  vergil-tooling (the lens-selection tool and its `vergil.toml` stanza),
  vergil-claude-plugin (`issue-implement` and the paad review doctrine),
  `.github` (this spec and plan)

## 1. Problem

`issue-implement`'s agentic-review gate (mode 2 of the paad review doctrine)
runs `paad:agentic-review` on every task branch, for up to three rounds before
the human is asked. Each round dispatches all six of the skill's specialists
and its Verifier, at full depth, whatever the change. That is right for any
change to code. For a two-line documentation fix it is far out of proportion:
it slows the pipeline and spends a large number of tokens applying the full
review to a change that cannot break anything the other specialists look
for.

This matters more as the fleet moves to smaller PRs that are meant to move
faster, and as agents take on merges and releases. Agentic review is what makes
that safe, because the content review happens before submission and CI becomes
a last backstop. A review that always runs at maximum weight becomes the
bottleneck that the smaller PRs were meant to remove.

## 2. Decision

**The caller of agentic review chooses which specialists run, and vergil
chooses by deterministic rules, never by the agent's judgement.**

- **paad** gains an optional, backward-compatible input that names the
  specialists to dispatch. Without it, all six run, as they do today.
- **A vergil-tooling command** computes the specialist set from the branch's
  changed files. The implementing agent runs it, passes its answer to the
  reviewer, and records it. It cannot ask for fewer specialists than the tool
  gives.
- **Security and Spec Compliance always run, on every change.** They are the
  floor. Every change, however small, can carry a security problem, and every
  change must do what its task asked.
- **The first rule:** a change whose every changed file is pure documentation,
  judged by an explicit allow-list, runs the floor alone. Every other change
  runs all six. Any doubt resolves to all six.
- **Rules are added later from evidence.** When the other four specialists can
  be skipped is not settled here. It is left to experience and to the cost
  metrics of vergil-project/.github#802. This epic's job is a mechanism where
  adding such a rule is one table entry and its tests.

Documentation drift that a lighter review might miss has other backstops: each
epic's documentation-review sweep, and periodic repo-wide documentation
audits.

## 3. What exists today

Read from paad 2.0.2, the paad fork's `unattended-mode` branch
(`preview/paad/skills/agentic-review/`), and the doctrine on
vergil-claude-plugin's `develop`.

- **Fixed dispatch.** The skill dispatches six specialists in parallel: Logic
  & Correctness, Error Handling & Edge Cases, Contract & Integration,
  Concurrency & State, Security, and Spec Compliance. A Verifier then checks
  their findings. A diff of 500 or more lines gets two instances of each
  specialist. The skill says the count "is fixed at six", and that diff size
  never changes how many are dispatched.
- **Small diffs read more.** For a diff under 50 lines, Phase 1 widens each
  specialist's input to the whole module or package around each changed file.
- **Bail-outs come after dispatch.** On a documentation-only diff, Error
  Handling, Concurrency & State and Contract & Integration each bail with a
  `BAIL: <lens> no-surface` token, and Security bails with `no-boundary` when no
  trust boundary is touched. Logic & Correctness never bails: it reports
  "clean". Spec Compliance always runs under the gate, which always supplies an
  intent source. Every bailing specialist has already been dispatched with the
  full diff and the widened module in its prompt.
- **No reliable selector.** The skill takes only a base and a path filter.
  paad's experimental `paad/config/` files are free prose, and paad's
  `CONFIG.md` warns against using them to skip or shorten a step. Mode 2 adopts
  no config file in any case.
- **The unattended mode has a precedent for inputs.** Under `paad-delegate`, the
  dispatch prompt may carry an `Intent: <path>[, <path>…]` line, which feeds
  Spec Compliance. The result's `Agents dispatched:` field lists the lenses
  dispatched, with each `BAIL:` reason.
- **The doctrine requires all six.** Doctrine §3.1 says the gate always runs,
  "including on docs-only and trivial changes". Its delegate-path output checks
  (§3.4) require `Agents dispatched:` to list all six lenses, none `NOT
  DISPATCHED`, and the fallback path checks the same in paad's report.
- **The doctrine has two paths.** The delegate path dispatches
  `paad:paad-delegate`. The fallback path, used when the delegate is not
  loaded, dispatches a general-purpose reviewer that runs the interactive
  skill.
- **`vergil.toml` already extends built-in lists.**
  `[actions].extra-allowed-patterns` adds a repo's own entries to a list the
  tooling carries, and `config.py`'s `_KNOWN_KEYS` registry rejects unknown keys.

## 4. Design

### 4.1 Lens names and the floor

The tool, the paad input and the records name lenses by paad's own lens
identifiers, the ones its `[ref-loaded:<lens>]` tokens use:

| Lens | Identifier |
|---|---|
| Logic & Correctness | `logic-correctness` |
| Error Handling & Edge Cases | `error-handling` |
| Contract & Integration | `contract-integration` |
| Concurrency & State | `concurrency-state` |
| Security | `security` |
| Spec Compliance | `spec-compliance` |

The **floor** is `security` and `spec-compliance`. It is vergil policy. It
lives in the vergil tool and the doctrine's output checks, never in paad.

### 4.2 paad: the `Lenses:` input

A change to agentic-review's `## Unattended mode` section in the paad fork, in
the same shape as the `Intent:` input.

- **The input.** The dispatch prompt may carry one line
  `Lenses: <lens>[, <lens>…]`, naming lenses by the identifiers in 4.1.
- **Without the line**, all six are dispatched, exactly as today. Every caller
  that does not use the input sees no change.
- **With the line**, exactly the named lenses are dispatched in Phase 2, and
  no others. The Verifier always runs. The large-diff rule applies to the
  named lenses: two instances of each.
- **Invalid input blocks the run.** The result is `Status: blocked: <reason>`
  when the line names a lens that does not exist, names no lens, or appears
  more than once. A lens named twice in one line is dispatched once.
- **The result says what happened.** `Agents dispatched:` lists the lenses
  dispatched, as it does today, and a new `Lenses requested:` skill-specific
  field gives the line's value, or `all (no Lenses: input)`. A lens left out on
  purpose appears in neither `Agents dispatched:` nor as `NOT DISPATCHED`: it was
  not requested.
- **Phase 1 is unchanged.** The touched-lines map, the manifest and the
  small-diff widening run as written. Whether to narrow them for a smaller set
  is a later question, settled by the metrics.
- **Unattended mode only.** The interactive skill's "fixed at six" rule and
  its arguments are unchanged. An interactive form is a conversation with the
  upstream author, not part of this epic.
- **The interactive text is reconciled, not changed.** Where the skill body
  says all six are dispatched every run, the unattended section says that
  under the delegate a `Lenses:` input takes precedence. The text stays true
  for every interactive run.
- **Nothing vergil-specific** goes into paad: no floor, no allow-list, no
  vergil names. The fork's own checks (`make` targets, including
  `check-unattended`) pass, and are extended if they parse the unattended
  section's inputs.

**Backward compatibility is a guideline, not a strict rule.** paad changes
should be as unobtrusive as possible for anyone not using the new features.
Small additions to the result, such as the `Lenses requested:` field, are
acceptable. Any change to a default is discussed with the upstream author
first.

### 4.3 The lens-selection tool

A new vergil-tooling command. The working name is `vrg-review-lenses`; the plan
may settle a different name.

**Input.** The base the review compares against (`--base origin/develop`), run
in the task's worktree. It reads the changed paths with the equivalent of
`vrg-git diff --name-status <base>...HEAD`: the same diff the review covers.
Both paths of a rename or copy count as changed, and a deleted file counts by
its old path.

**Rules.** An ordered rule table in code; the first rule that matches decides.

| Order | Rule | Matches when | Lenses |
|---|---|---|---|
| 1 | `docs-only` | every changed path is on the docs allow-list (4.4) | the floor |
| 2 | `full` | always | all six |

Adding a rule later means one table entry, placed by order, with its tests.
Every rule's lens set must include the floor; the tool checks this when it
starts, so a rule that drops the floor is a failure, never a lighter review.

**Output.** Plain lines, parsed by the gate:

```text
Lenses: security, spec-compliance
Rule: docs-only
```

```text
Lenses: logic-correctness, error-handling, contract-integration, concurrency-state, security, spec-compliance
Rule: full
Not docs: src/vergil_tooling/bin/vrg_review_lenses.py, vergil.toml (+3 more)
```

`Not docs:` names the first few changed paths that kept the change off the
docs-only rule, and how many more there are. A `--json` form gives the same
fields, with every path.

**Errors.** The tool exits non-zero, with the reason on stderr, when the base
does not resolve, the diff is empty, or `vergil.toml`'s stanza is invalid
(4.4). It never guesses a lens set on an error.

### 4.4 The docs allow-list

A change is docs-only only if **every** changed path is on the allow-list.

**Base list.** Fleet-standard documentation paths, carried in the tool's code.
No repo can remove them. It matches Markdown files only (`*.md`); any other file
type, even inside a listed directory (a site's `mkdocs.yml`, stylesheets,
images), is not docs.

| Pattern | What it is |
|---|---|
| `README.md` | the repo's root README only |
| `CHANGELOG.md` | the root changelog |
| `releases/*.md` | release notes |
| `docs/site/**/*.md` | the versioned site docs |
| `docs/specs/**/*.md` | specs |
| `docs/plans/**/*.md` | plans |
| `docs/superpowers/**/*.md` | brainstorm specs and plans |
| `epics/**/*.md` | epic specs, plans and retrospectives |

This list is narrow on purpose. The fleet survey behind it found that `docs/`
also holds files that carry logic. In vergil-claude-plugin, `docs/*.md`
includes the paad review doctrine, the paad policy and the repository
standards, which the skills bind to as rules. In vergil-tooling,
`docs/audit-criteria/` holds criteria that audits apply. A blanket `docs/**` or
`*.md` would let a change to the doctrine pass as documentation.

**Built-in exclusions.** Paths that are never docs, whatever a repo adds:

- `**/SKILL.md`, `skills/**`, `agents/**`, `commands/**`, `hooks/**`;
- `**/CLAUDE.md`, `**/AGENTS.md`, `.claude/**`;
- `.github/**` (workflows, issue and PR templates);
- `paad/**` (paad's config, backlogs and review outputs);
- `vergil.toml`.

An exclusion wins over every allow-list entry, base or repo-added. Excluding
`vergil.toml` means a PR that widens a repo's allow-list is itself reviewed with
all six lenses; only later PRs benefit from the new paths.

**Repo additions.** A new `vergil.toml` stanza adds a repo's own patterns:

```toml
[review]
extra-docs-paths = ["docs/runbooks/**/*.md", "notes/**/*.md"]
```

- It follows `[actions].extra-allowed-patterns`: a list of strings, added to
  the base list, registered in `_KNOWN_KEYS`. A value that is not a list of
  strings, or a pattern that is empty or absolute, is a config error.
- A repo addition is for documentation only, never for a path holding skills,
  agent instructions, templates or other files that carry logic. The built-in
  exclusions enforce the clearest cases; the rest relies on the full review
  that every change to `vergil.toml` gets.
- The tool reads `vergil.toml` from the working tree being reviewed.

**Pattern syntax.** Paths are repo-relative, with `/` separators. `*` matches
within one path segment, and `**` matches zero or more whole segments.
Matching is case-sensitive. vergil-tooling supports Python 3.12, which lacks
`PurePath.full_match`, so the plan chooses the implementation (a small matcher
in the tool, or a library), tested against these rules.

Which paths each repo should add is a follow-up intake item,
vergil-project/.github#830.

### 4.5 The gate

Changes to `issue-implement` and doctrine §3, in vergil-claude-plugin.

- **Each round runs the tool.** Before it writes the round's requirements
  file, the implementer runs the tool in the task's worktree with the round's
  base. The set is recomputed every round, so a fix that touches a code file
  moves the review to all six from that round on.
- **A tool failure pauses the run** (doctrine §3.8), with the tool's reason.
  The gate never substitutes a lens set of its own.
- **The delegate prompt** (§3.2) gains a `Lenses: {{LENSES}}` line after its
  `Intent:` line, filled with the tool's `Lenses:` value. The prompt is
  otherwise unchanged.
- **The fallback path runs all six.** Its interactive skill has no `Lenses:`
  input, so it is not given one. Its records say so (below).
- **The output checks** (§3.4) change on the delegate path:
  - `Agents dispatched:` lists every lens in the tool's set, none `NOT
    DISPATCHED`, and includes both floor lenses.
  - Lenses dispatched beyond the tool's set are allowed, because they only add
    scrutiny. A paad that predates the input runs all six, which passes. The
    record shows the difference.
  - The existing Spec Compliance and intent-source checks stay.
  - On the fallback path the existing all-six check stays.
- **The records** (§3.5): each round's findings comment carries the lens
  choice: `Lenses: <identifiers> (rule: <rule>)`, the tool's `Not docs:` line
  when it printed one, and `Lenses dispatched: <identifiers>` when that differs
  from the requested set. On the fallback path the line is
  `Lenses: all six (fallback path)`. This lets #802's cost metrics be grouped by
  rule.
- **No override.** The agent cannot request fewer lenses than the tool gives.
  Neither the agent nor the human gets a way to request more in this epic. A
  human who wants the review skipped altogether can still direct that ad hoc in
  a session, recorded on the task issue, as an extreme exception; the
  framework gets no skip mechanism
  (vergil-project/vergil-claude-plugin#856, closed as not planned).
- **§3.1's "always runs"** stays true: the gate always runs, with the tool's
  lens set and never below the floor. The sentence about docs-only changes is
  rewritten to say so.
- **Mode 1 is unchanged.** Pushback and alignment rounds do not use lenses.

The paad policy register (`docs/paad-policy.md`) gains an entry for the
`Lenses:` input: what vergil uses that the upstream skill does not yet have,
and where the floor lives.

### 4.6 Rollout and compatibility

The pieces can ship in any order without lowering any review:

- A paad without the input ignores the line and runs all six, which the
  output checks accept.
- The gate does not run the tool until the plugin release that carries the
  doctrine change, and that release requires a vergil-tooling release with the
  tool.

The lighter review takes effect only once all three are deployed: the updated
paad fork on the hosts and agent VMs, the vergil-tooling release, and the
plugin release. The plan carries a deployment task for that, and a validation
task that runs the gate once on a docs-only task and once on a code task and
checks the lens sets, the dispatches and the records.

## 5. Changes

| Repo | What |
|---|---|
| paad fork | `Lenses:` input in agentic-review's unattended mode; `Lenses requested:` result field; fork checks extended as needed |
| vergil-tooling | the lens-selection command, its rule table, base allow-list and exclusions; the `[review].extra-docs-paths` stanza in `config.py`; site docs for both |
| vergil-claude-plugin | doctrine §3.1, §3.2, §3.4 and §3.5; `issue-implement`'s gate steps; the paad policy register entry |
| `.github` | this spec and plan |

Offering the paad change upstream is a human decision, made after it has run
in the fleet, as in epic vergil-project/.github#699.

## 6. Out of scope

- **Cost metrics** for sub-agents and for each specialist: vergil-project/.github#802.
- **Tuning each specialist's depth**, and further rules that drop lenses
  beyond docs-only: the follow-on brainstorm, vergil-project/.github#829, once
  metrics and experience exist.
- **Narrowing Phase 1** (the manifest and the small-diff widening) for a
  smaller lens set.
- **An interactive `Lenses:` form** in paad, and any change to paad's defaults.
- **A skip-review mechanism**, for agent or human.
- **Lens selection in mode 1.**
