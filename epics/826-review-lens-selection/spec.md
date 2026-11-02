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
  DISPATCHED`.
- **The doctrine has one path.** Since vergil-project/vergil-claude-plugin#885
  (closing #879), every review round dispatches `paad:paad-delegate`, and a
  session without it runs no review round (doctrine §1.8, "The delegate
  requirement"). There is no fallback path.
- **`vergil.toml` already extends built-in lists.**
  `[actions].extra-allowed-patterns` adds a repo's own entries to a list the
  tooling carries. `config.py` registers sections in `_KNOWN_SECTIONS` and keys
  in `_KNOWN_KEYS`, and only warns on an unrecognised key; it does not reject
  it.
- **paad 2.1.0 is released upstream.** The fork's `unattended-mode` branch is
  based on 2.0.2 and must be rebased onto 2.1.0 before any further significant
  paad change (4.6).
- **vergil checks paad's announced version, and gates each move.** paad-policy
  §10 has two separate mechanisms. The version check fails any delegate round
  whose announced version is not the one §10 records as validated, today
  `2.0.2-preview`; upstream 2.1.0 announces `2.1.0-preview`. The check sits in
  doctrine §1.9 (the first-line check, which also applies to blocked
  results), in the opening lines of §2.4 and §3.4, and in paad-policy §1 and
  §10. The gate reruns
  the #733 validation sets after every move of the `fleet` branch, with
  reviews held until they pass. Only the gate shows that a moved paad still
  routes security findings privately, ignores planted instructions, and finds
  seeded defects (4.7).

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
- **Without the line**, all six are dispatched, exactly as today.
- **With the line**, exactly the named lenses are dispatched in Phase 2, and
  no others. The Verifier always runs. The large-diff rule applies to the
  named lenses: two instances of each.
- **Invalid input blocks the run.** The result is `Status: blocked: <reason>`
  when the line names a lens that does not exist, names no lens, or appears
  more than once. A lens named twice in one line is dispatched once.
- **The result names lenses by identifier, in every unattended run.**
  `Agents dispatched:` and `Lenses requested:` name each lens by its 4.1
  identifier, never by its display name, with or without a `Lenses:` line, so
  the field has one format and the gate compares it with the tool's output
  exactly. This changes the unattended result for every caller. vergil is
  the only one today, and the fork carries the mode, so it is a small result
  change under the guideline below, raised in the upstream conversation. The
  fork's `check-unattended` is extended to check it if it parses the field.
  Doctrine §3.4 accepts either form from rollout step 1 (4.6), so the fork's
  switch at step 3 breaks no round.
- **Each `Agents dispatched:` entry begins with the lens's identifier.** What
  follows it in the entry (a `BAIL:` reason, the large-diff partitioning)
  stays as paad writes it today. The gate reads only the leading identifier
  of each entry.
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
- **The interactive text is reconciled, not changed.** Wherever the skill says
  all six are dispatched every run, the unattended section says that under the
  delegate a `Lenses:` input takes precedence: the skill body, the Common
  Mistakes row ("the count is fixed at six") and
  `references/report-template.md`'s `Agents dispatched:` line ("all six
  lenses, each listed by name … every lens appears every run"). The text
  stays true for every interactive run.
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
in the task's worktree. It reads the changed paths, with their file modes, from
`vrg-git diff --raw -z <base>...HEAD`: the same diff the review covers.
Both paths of a rename or copy count as changed, and a deleted file counts by
its old path.

**Rules.** An ordered rule table in code; the first rule that matches decides.

| Order | Rule | Matches when | Lenses |
|---|---|---|---|
| 1 | `docs-only` | every changed path is on the docs allow-list (4.4) | the floor |
| 2 | `full` | always | all six |

Adding a rule later means one table entry, placed by order, with its tests.
Every rule's lens set must include the floor, and the last rule must be
`full`; the tool checks both when it starts, so a rule that drops the floor is
a failure, never a lighter review.

**Output.** The gate reads the `--json` form, which carries exactly one
`lenses` list, one `rule` and every non-docs path, with paths escaped. Plain
lines are the form for people:

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
docs-only rule, and how many more there are.

**Errors.** The tool exits non-zero, with the reason on stderr, when the base
does not resolve, the diff is empty, `vergil.toml` is missing or its stanza is
invalid (4.4), or a changed path contains a control character. It never
guesses a lens set on an error.

### 4.4 The docs allow-list

A change is docs-only only if **every** changed path is on the allow-list.

**Base list.** Fleet-standard documentation paths, carried in the tool's code.
No repo can remove them. It matches Markdown files only (`*.md`); any other file
type, even inside a listed directory (a site's `mkdocs.yml`, stylesheets,
images), is not docs. Only regular files can be docs: a changed path whose
mode at the base or at HEAD is anything but a regular file is not docs.

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

Even the listed directories are not clean. vergil-tooling's
`docs/specs/worktree-convention.md` is the worktree convention that every
consumer `CLAUDE.md` and `epic-implement`'s agent prompt contract link to as
binding. vergil-claude-plugin's `docs/site/docs/skills/index.md` holds the
execution doctrine that `issue-implement`, `issue-deploy` and `issue-validate`
bind to. vergil-tooling's `docs/site/docs/guides/claude-config-ownership.md` is
the ownership model the agent tools enforce. Each repo excludes such files
itself (repo exclusions, below). Moving rule documents out of `docs/`, so that
these exclusions are no longer needed, is vergil-project/.github#840.

**Built-in exclusions.** Paths that are never docs, whatever a repo adds:

- `**/SKILL.md`, `skills/**`, `agents/**`, `commands/**`, `hooks/**`;
- `**/CLAUDE.md`, `**/AGENTS.md`, `.claude/**`;
- `.github/**` (workflows, issue and PR templates);
- `paad/**` (paad's config, backlogs and review outputs);
- `vergil.toml`.

An exclusion wins over every allow-list entry, base or repo-added. Excluding
`vergil.toml` means a PR that widens a repo's allow-list is itself reviewed with
all six lenses; only later PRs benefit from the new paths.

**Repo additions and exclusions.** A new `vergil.toml` stanza adds a repo's
own patterns to either list:

```toml
[review]
extra-docs-paths = ["docs/runbooks/**/*.md", "notes/**/*.md"]
extra-non-docs-paths = ["docs/specs/worktree-convention.md"]
```

- Both keys follow `[actions].extra-allowed-patterns`: a list of strings,
  added to the built-in list. `review` is registered in `_KNOWN_SECTIONS` and
  both keys in `_KNOWN_KEYS`. A value that is not a list of strings, or a
  pattern that is empty or absolute, is a config error. Every
  `extra-docs-paths` pattern must also end in `.md`, as the base list matches
  Markdown only; any other pattern, such as `docs/runbooks/**`, is a config
  error, never a silent widening to non-Markdown files. The tool also treats
  any other key under `[review]` as a config error, rather than the warning
  `config.py` gives elsewhere, so a misspelt key cannot silently drop an
  exclusion.
- `extra-non-docs-paths` can only add exclusions; no key removes a built-in
  one. A repo exclusion wins over every allow-list entry, as a built-in one
  does.
- This epic adds the known rule documents to the stanza in vergil-tooling
  (`docs/specs/worktree-convention.md`,
  `docs/site/docs/guides/claude-config-ownership.md`) and vergil-claude-plugin
  (`docs/site/docs/skills/index.md`). First, a sweep covers every base-list
  pattern in every repo that carries `vergil.toml` in the vergil-project,
  logical-minds-foundry and mnemosys-project orgs, for any other file that
  skills, agents or a `CLAUDE.md` bind to, and each repo's stanza lists what
  it finds.
- **Accepted gap: repos in other orgs.** Repos in other orgs that carry
  `vergil.toml` are on hold and are not swept. Each needs the sweep before
  work resumes there, which vergil-project/.github#830 records.
- **Accepted gap until #840.** A rule document created later in an allowed
  path is docs-only, and so is every edit to it, until someone adds a repo
  exclusion for it. Until vergil-project/.github#840 moves rule documents out
  of `docs/`, such a file is covered only by the floor lenses and the
  documentation-review sweeps. This is accepted; no per-file detection is
  built.
- A repo addition is for documentation only, never for a path holding skills,
  agent instructions, templates or other files that carry logic. The built-in
  exclusions enforce the clearest cases; the rest relies on the full review
  that every change to `vergil.toml` gets.
- The tool reads `vergil.toml` from the working tree being reviewed.

**Pattern syntax.** Paths are repo-relative, with `/` separators. `*` matches
within one path segment, and `**` matches zero or more whole segments.
Allow-list patterns match case-sensitively; exclusions, built-in and
repo-added, match case-insensitively, so every case mismatch fails toward the
full review. vergil-tooling supports Python 3.12, which lacks
`PurePath.full_match`, so the plan chooses the implementation (a small matcher
in the tool, or a library), tested against these rules.

Which paths each repo should add is a follow-up intake item,
vergil-project/.github#830.

### 4.5 The gate

Changes to `issue-implement` and doctrine §3, in vergil-claude-plugin.

- **Each round runs the tool.** In §3.4's ordered pre-dispatch list, the tool
  runs in the task's worktree with the round's base, after the fetch and
  before the state-directory check. (The round's requirements file is written
  earlier, in §3.1, and does not depend on the lens set.) The set
  is recomputed every round, so a fix that touches a code file moves the
  review to all six from that round on.
- **A tool failure pauses the run** with the tool's reason, and is added to
  §3.8's list of pre-dispatch pause causes. The gate never substitutes a lens
  set of its own.
- **Every round sends `Lenses:`.** The delegate prompt (§3.2) gains a
  `Lenses: {{LENSES}}` line after its `Intent:` line, filled with the `--json`
  form's `lenses` list, joined with `, `. The prompt is otherwise unchanged.
- **The output checks** (§3.4) require an exact match:
  - `Lenses requested:` equals the value sent;
  - the leading identifiers of `Agents dispatched:`'s entries are exactly the
    tool's set, none `NOT DISPATCHED`, which includes both floor lenses;
  - any mismatch, including lenses dispatched beyond the set, fails the check
    and pauses the run (§3.8). A delegate that ignores or misreads the line
    has broken its input contract, and would otherwise lose the saving
    unseen and file its cost under the wrong rule;
  - the existing Spec Compliance and intent-source checks stay.

  A `fleet` rolled back to a paad without the input therefore pauses every
  agentic-review round until `fleet` moves forward again, through the gate.
- **The records** (§3.5): each round's findings comment carries the lens
  choice: `Lenses: <identifiers> (rule: <rule>)`, and, when the rule is not
  `docs-only`, the first few non-docs paths in a fenced code block. This lets
  #802's cost metrics be grouped by rule.
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

The fleet runs the maintainer's development paad, from the fork, until further
notice. vergil depends on no specific paad version. The work runs in this
order:

1. **Remove vergil's paad version check** (4.7). It ships as a plugin release,
   reviewed on today's paad; vergil-tooling has no paad version check. The same release makes §3.4 accept lens identifiers as
   well as display names in `Agents dispatched:` (4.2). It comes first because upstream 2.1.0 announces
   `2.1.0-preview`, which today's check fails: rebasing first would fail every
   delegate round in the fleet, including the gate's own runs, until the
   removal shipped.
2. **Rebase the fork onto paad 2.1.0.** The fork's `unattended-mode` branch is
   rebased onto upstream 2.1.0, and the human moves `fleet` to it and runs
   the gate (4.7) before reviews resume. This comes before any further
   significant paad change. The fork's git writes are the human's.
3. **The `Lenses:` input** is built on a `lenses-input` branch on top of the
   rebased `unattended-mode` and merged into `fleet`. The human moves `fleet`
   and runs the gate again, with the `Lenses:` scenarios added (4.7).
4. **The tool and the gate.** The vergil-tooling release with the tool, then
   the plugin release with the doctrine and `issue-implement` changes.

Another order matters for correctness: the fork's `Lenses:` change is
installed before the plugin release whose gate sends `Lenses:`, so that no
delegate gets a line it does not know. The tool and plugin releases are
otherwise safe:
the gate does not run the tool until the plugin release that carries the
doctrine change, and that release requires a vergil-tooling release with the
tool.

The lighter review takes effect only once all three are deployed: the updated
paad fork on the hosts and agent VMs, the vergil-tooling release, and the
plugin release. The plan carries a deployment task for that, in the order
above, and a validation task that runs the gate once on a docs-only task and
once on a code task and checks the lens sets, the dispatches and the records.

### 4.7 paad validation policy

The fleet runs the maintainer's development paad, from the fork, until
further notice, and the maintainer will keep changing it, offering the
changes upstream. Each such change runs in every fleet review before upstream
sees it. The policy follows from that.

- **No version check.** vergil depends on no specific paad version. The
  version check is removed wherever it sits: doctrine §1.9's first-line check
  (for complete and blocked results, in both modes), the version lines that
  open §2.4 and §3.4, the validated version in paad-policy §1 and §10, and any
  other paad version check the task's sweep finds in vergil-claude-plugin
  (vergil-tooling has none). The first line is still checked for the skill's name. The
  check only compared an announced string; it never showed that a moved paad
  still reviews correctly. When, if ever, the fleet goes back to a released
  paad, and whether versions are checked then, is the human's decision;
  nothing tracks it.
- **The gate stays, and is a standard step.** After every move of `fleet`
  (the maintainer's own paad changes, upstream syncs, and this epic's two
  moves), the full automated scenario set of vergil-project/.github#733 runs
  on the new commit, in newly started sessions, before reviews resume. That
  is every delegate run: the plain, seeded, planted-config,
  planted-dispatch, unsupported-skill and bad-`Intent:` sets. The set is not
  trimmed. It is the only check that a moved paad still routes security
  findings to the private section, ignores planted instructions (checked
  against the session transcripts), and finds seeded defects; each round's
  own output checks trust the delegate's report on those. Anything offered
  upstream has passed it.
- **Interactive runs leave the gate.** The human's interactive run of each
  changed skill is no longer part of it.
- **Every change that moves `fleet` carries a validation task** that runs the
  gate and records the outcome and the fleet commit. A paad change's plan
  files one beside its deployment task. For this epic, the gate's second run
  adds `Lenses:` scenarios: a floor-only request, an invalid lens, and a
  seeded security bug under a floor-only request.
- **Held reviews stay the human's process.** Until the gate passes on a moved
  commit, the human holds every review round that loads the fleet checkout,
  as §10 says today.
- **Accepted risk, unchanged.** An uncommitted edit in the fleet checkout is
  not covered by the gate.

paad-policy §10 is rewritten to match, and the doctrine's rules that mention
the validated version follow. The scenario set is defined today only in epic
vergil-project/.github#699's spec §5 and the closed #733. A standing gate needs
a maintained list, so it moves into vergil-claude-plugin, linked from §10, with
the `Lenses:` scenarios added; the plan settles the file.

## 5. Changes

| Repo | What |
|---|---|
| paad fork | rebase onto 2.1.0; `Lenses:` input in agentic-review's unattended mode; `Lenses requested:` result field; lens identifiers leading each `Agents dispatched:` entry; paad-help's agentic-review entry; fork checks extended as needed |
| vergil-tooling | the lens-selection command, its rule table, base allow-list and exclusions; the `[review]` stanza (`extra-docs-paths`, `extra-non-docs-paths`) in `config.py`; site docs for both; its own `[review].extra-non-docs-paths`; removal of any paad version check |
| vergil-claude-plugin | doctrine §1.9, §2.4, §3.1, §3.2, §3.4, §3.5 and §3.8; `issue-implement`'s gate steps; paad-policy §1 and §10 (no version check, the gate as a standard step) and the register entry for `Lenses:`; the gate's maintained scenario list; its own `[review].extra-non-docs-paths` |
| fleet repos | `[review].extra-non-docs-paths` entries where the sweep finds rule documents |
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
- **Moving rule documents out of `docs/`**: vergil-project/.github#840. Until
  then, repo exclusions name them.
- **Markdown linting changes.**
