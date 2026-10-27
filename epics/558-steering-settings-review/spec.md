# Steering-file and Claude settings review

- **Epic:** vergil-project/.github#558
- **Planning issue:** vergil-project/.github#506
- **Umbrella:** vergil-project/.github#500 (child E). Depends on B's Tasks 1
  and 3 (vergil-project/vergil-tooling#3666,
  vergil-project/vergil-claude-plugin#727), both closed.
- **Status:** approved design (2026-10-27)
- **Review history:** the comments on the planning issue,
  vergil-project/.github#506
- **Design source:** umbrella spec §4.E. The human skipped the brainstorm
  questions, so the decisions that §4.E leaves open are the agent's calls.
  They are collected in §9 for review.

## 1. Intent

Every agent session starts by loading a stack of files nobody has ever reviewed
as a set:

- **Steering files:** each repo's `CLAUDE.md` (and in three repos an
  `AGENTS.md`) and the global `~/.claude/CLAUDE.md`.
- **Settings files:** the user-level `~/.claude/settings.json`, each repo's
  checked-in `.claude/settings.json`, and the managed template that generates
  and audits the repo files
  (`vergil-tooling/src/vergil_tooling/data/claude_settings.json`).

These files have grown for months by small additions. paad's authors say to
validate the steering files first, because everything else inherits them. The
umbrella's later children (C, D, F) will add more instructions to them, so they
must be clean before that.

This epic delivers three things:

1. **An ownership model:** a written "which file owns what" document,
   verified against the current Claude Code docs, with the date of
   verification.
2. **A one-time cleanup** of every file in scope to match the model, with the
   fleet config audit passing at every step.
3. **A recurring review** that keeps the files matching the model: mechanical
   rules in the config audit, plus a periodic judgment review in a skill.

## 2. Current state (surveyed 2026-10-27)

### 2.1 Steering files

| Repo | `CLAUDE.md` lines | Template markers | `AGENTS.md` |
|---|---|---|---|
| vergil-tooling | 554 | yes | 45 lines |
| vergil-claude-plugin | 375 | no (legacy match) | 24 lines |
| vergil-containers | 307 | no (legacy match) | — |
| vergil-actions | 234 | no (legacy match) | 38 lines |
| .github | 158 | no (legacy match) | — |
| docs | 128 | yes | — |
| vergil-vm | 124 | no (legacy match) | — |
| vergil-python | 120 | yes | — |
| packages | 115 | yes | — |

The global `~/.claude/CLAUDE.md` is 120 lines.

Known problems:

- **Over budget.** Four repos exceed the 200-line target, and vergil-tooling
  is nearly three times over it. The managed consumer template region
  (`data/claude_md_consumer.md`) alone is 101 lines, half the budget for
  every repo.
- **Fleet policy in one repo only.** The "Identity modes and PR submission"
  section (agents stop at `report-ready` and never run `vrg-submit-pr`; the
  branch freeze) and the cloud-session prompt contract exist only in
  vergil-tooling's `CLAUDE.md`. Agents in the other eight repos never load
  them. The plugin skills cover part of this.
- **AGENTS.md files are dead and harmful.** Claude Code does not read an
  `AGENTS.md` when a `CLAUDE.md` exists (§3), so these files are seen only by
  other tools. Their content is also wrong: vergil-tooling's tells agents to
  run raw `git checkout -b`, and vergil-actions's says to treat a missing
  `standards-and-conventions` checkout as fatal and to load skills from it.
- **Unfinished template migration.** Five repos pass the `CLAUDE.md` audit
  through its legacy path (the template text present verbatim, without
  markers), which was meant only as a transition.
- **Missing pointer.** Only vergil-claude-plugin points to
  `vergil-claude-plugin/docs/paad-policy.md`. The umbrella requires every
  repo's steering files to point to it.
- **Global file is the priority concern.** It has grown ad hoc, with no
  change control, yet it steers every session on every machine. It is stale
  in places (for example, it still says "standard-tooling suite" for the
  vergil tooling). Much of it is not personal at all: its "Memory management"
  section is restated in every repo's template region, and sections such as
  "Correctness over cost" and "No silent failures" are engineering policy.

### 2.2 Settings files

- **Repo files.** All nine repo `.claude/settings.json` files have the same
  content. Each is the template plus a seeded `ref: "main"` on the
  `vergil-marketplace` source (`repo_init.render_claude_settings`). They hold:
  - `permissions.allow: ["Bash(vrg-*)"]`;
  - a `PreToolUse` `Bash` hook running `.claude/hooks/guard.sh`;
  - `extraKnownMarketplaces.vergil-marketplace`;
  - `enabledPlugins: {"vergil@vergil-marketplace": true}`;
  - `attribution: {"commit": "", "pr": ""}`.
- **The config audit requires those entries.** `repo_config.py`'s
  `_check_claude_settings` requires the `vergil` `enabledPlugins` entry, the
  marketplace source, the `main` ref, and the exact `attribution` value. The
  "redundant" repo entries are there because the audit demands them, so the
  cleanup must change the audit, not only the files.
- **User file.** The user-level `settings.json` is copied verbatim from the
  macOS host into every agent VM on each session (`vm_guest.copy_claude_config`).
  It enables all plugins (`vergil`, `paad`, `superpowers`), declares their
  marketplaces including `vergil-marketplace` with `ref: "main"` (so the
  umbrella spec's claim that the user file has no ref is wrong), and carries
  personal settings. Two of its entries are suspect:
  - a `statusLine` hard-coded to `/opt/homebrew/bin/node`, which doesn't
    exist on the Linux VMs, for `claude-hud`, which is disabled;
  - a `PreToolUse` hook that denies `AskUserQuestion`. It belongs to no
    documented decision and may be intentional.
- **Unverified code comment.** `vm_guest.py` (lines 401 and 449) says that
  "since Claude Code v2.1.195 an enabled entry no longer auto-installs". The
  2.1.195 changelog entry says something narrower (§3).
- **Unused scopes.** No repo has a `settings.local.json`, and no managed
  settings are used.

## 3. Platform facts (verified 2026-10-27)

Fetched from <https://code.claude.com/docs/en/settings.md>,
`settings-reference.md`, `permissions.md`, `plugins/loading.md`,
`hooks.md`, `hooks-guide.md` and `memory.md` (all under
`https://code.claude.com/docs/en/`), and from the Claude Code `CHANGELOG.md`.
The model document (§5.1) re-verifies them when it is written.

1. **Precedence:** managed > command line > project local > shared project >
   user. A key set at a higher level overrides it lower down. Lists such as
   `permissions.allow` combine.
2. **Permissions:** "If a tool is denied at any level, no other level can
   allow it."
3. **Hooks** merge across user, project, local and plugin sources. They are
   not gated by folder trust.
4. **Folder trust:** project `permissions.allow`, `extraKnownMarketplaces` and
   most `env` values "apply only after each teammate trusts the folder".
5. **`enabledPlugins`** merges per plugin, and the project file beats the user
   file: a user-level `false` cannot disable a plugin the project enables.
   A plugin whose only `true` is in the project file is not fetched; a fetch
   needs a user, local, `--settings` or managed `true`.
6. **`extraKnownMarketplaces`:** "Claude Code uses the entry from the
   highest-precedence file whole … inherits none of its fields" (since
   v2.1.228). If the project and user files both define
   `vergil-marketplace`, the project entry wins as a whole. A project entry
   without `ref` would therefore silently remove the user's `ref`.
7. **`attribution`:** empty strings for `commit` and `pr` hide attribution
   ("Set it to an empty string to hide commit attribution"). The `false` form
   needs v2.1.281+, and older versions skip the whole file, so shared files
   keep the object form. `sessionUrl` affects only cloud and Remote Control
   commits.
8. **Changelog 2.1.195** reads: "Fixed external plugins enabled only by
   project `.claude/settings.json` not requiring explicit install consent on
   every loader path". It doesn't say that user-enabled plugins stopped
   auto-installing. Whether a fresh VM installs user-enabled plugins without
   help is still to be verified (§5.4).
9. **CLAUDE.md:**
   - target under 200 lines per file;
   - imports (`@path`, at most four hops) don't reduce context cost;
   - path-scoped `.claude/rules/` files load only when Claude works with
     matching files;
   - `AGENTS.md` is read only when no `CLAUDE.md` exists (v2.1.277+), or
     when a `CLAUDE.md` imports it.
10. **`/doctor prompt-audit`** (v2.1.283+) audits CLAUDE.md, CLAUDE.local.md,
    AGENTS.md, rules, skills, commands and agents under `.claude/` and
    `~/.claude/`. It looks for stale paths, stale commands, contradictions,
    and patterns written for older models. It reports proposed edits and
    changes nothing until asked.

## 4. Goals, non-goals, anti-goals

### Goals

- Every setting and every instruction has exactly one owning file, and the
  model says which.
- Every repo `CLAUDE.md` is under 200 lines, including the template region.
- Fleet-wide agent policy reaches every repo through the managed template.
- The global `~/.claude/CLAUDE.md` is **minimal**. It holds only preferences
  that are genuinely personal and have no change-controlled home. Everything
  else moves to the managed template region, the vergil plugin, or repo docs.
- The nightly config audit enforces the model's mechanical rules, so drift
  turns the scheduled run red rather than waiting for a review.
- A recurring review exists and is documented.

### Non-goals

- Managed settings (`managed-settings.json`). This is a single-operator
  fleet, and the template plus the audit already enforce the repo layer. The
  model records this as a deliberate choice.
- Delivering fleet policy through a plugin `SessionStart` hook instead of the
  template. It may be worth a later look; it is out of scope here.
- Versioning the global `~/.claude/CLAUDE.md` or the user `settings.json` in a
  repo. They stay host files owned by the human. Keeping the global file
  minimal is what reduces the need for change control on it.
- Redesigning the memory mechanism. Memory is considered experimental and is
  a candidate for replacement (intake item vergil-project/.github#561). This
  epic therefore builds its recurring review as a separate `steering-audit`
  skill, leaves `memory-audit` untouched, and adds no new dependency on
  memory (§5.5).
- Rewriting the content of the plugin skills. Skill text may be trimmed only
  where it restates a steering file.

### Anti-goals (failures even if everything technically works)

- A fleet audit that goes red at any step of the transition.
- Moving lines out of `CLAUDE.md` into `@imports` to meet the line budget.
  Imports don't reduce context cost.
- An agent losing an instruction it actually relies on, such as the PR
  handoff or the no-raw-git rule, because the instruction was cut to save
  lines.
- A model document that disagrees with the docs it cites.
- Breaking plugin installation on agent VMs.

## 5. Design

### 5.1 The ownership model

This is a new versioned site page in vergil-tooling,
`docs/site/docs/guides/claude-config-ownership.md`, linked from
`permission-model.md`, `config-audit.md` and `consuming-repo-setup.md`. It
records the docs URLs and the verification date, and it is the reference the
recurring review checks against.

**Placement rule (approved).** A setting lives in the change-controlled repo
file unless the platform forces it into user scope. The repo file is
reviewed, versioned and audited nightly; the user file is none of those, and
it should stay minimal. Plugin enablement and marketplaces are forced into
user scope: a project-only enable is never installed (§3.5), and a second
marketplace definition overrides the first whole (§3.6). The guard hook,
`attribution` off, and `allow Bash(vrg-*)` are fleet policy. In the repo file
they apply to every checkout on any machine, including host sessions and
fresh clones without the VM's copy of the user file, and the hook applies
even before folder trust (§3.3). Fleet policy must not depend on the personal
file, so they stay in the repo.

The model:

| File | Owns | Must not hold |
|---|---|---|
| `~/.claude/settings.json` (host, copied verbatim to VMs) | plugin enablement and marketplaces for **all** plugins; personal mode, model and UI settings | anything that works on only one OS, because the same file runs on the macOS host and on Linux VMs |
| repo `.claude/settings.json` (checked in, generated from the template, audited) | the guard `PreToolUse` hook; `permissions.allow: Bash(vrg-*)`; `attribution` off | `enabledPlugins`, `extraKnownMarketplaces` (a second definition overrides the user's whole entry, §3.6); any repo-specific key |
| managed template `data/claude_settings.json` | the only source for the repo file's content | — |
| `.claude/settings.local.json`, managed settings | unused (documented as deliberate) | — |
| `~/.claude/CLAUDE.md` (host, human-owned, minimal) | only preferences that are genuinely personal and have no change-controlled home | engineering or workflow policy; anything a repo template, the plugin or repo docs can hold |
| template region of repo `CLAUDE.md` (`data/claude_md_consumer.md`) | fleet-wide agent policy: worktrees, `vrg-git`/`vrg-gh`, validation command, the PR handoff and freeze, the memory pointer, the `paad-policy.md` pointer | reference material; anything a hook enforces mechanically, beyond a one-line pointer |
| rest of repo `CLAUDE.md` | what an agent needs to work in that repo: overview, commands, constraints | catalogues and architecture detail (these go in site docs, linked); restated fleet policy |
| `.claude/rules/*.md` (path-scoped) | instructions that matter only for part of a repo | — |
| `AGENTS.md` | nothing: removed (§5.2) | — |

### 5.2 Steering-file cleanup

- **Template region** (`data/claude_md_consumer.md`):
  - target about 60 lines;
  - adds the PR-handoff and freeze rules (moved from vergil-tooling's
    `CLAUDE.md`) and the `paad-policy.md` pointer;
  - shortens the worktree section to the rules plus a link to the canonical
    spec, and the shell-policy section to a line, since the hook guard
    enforces it.
- **Every repo `CLAUDE.md`** is brought under 200 lines. Reference material
  moves into that repo's docs (site docs where they exist) and is linked;
  restated fleet policy is deleted; part-scoped guidance moves into
  `.claude/rules/` where it helps. Each repo is reviewed with
  `/doctor prompt-audit` and `/paad:pushback` before and after its rewrite.
- **Template markers.** The five legacy repos adopt the markers. The audit's
  legacy path is then removed.
- **AGENTS.md** is deleted in vergil-tooling, vergil-actions and
  vergil-claude-plugin. Anything in them that is still true and not already
  in `CLAUDE.md` is moved there first.
- **Global `~/.claude/CLAUDE.md`** is the priority, and the goal is to make
  it minimal. The agent reviews it with prompt-audit and pushback, and sorts
  every section into one of these:
  - **delete**, because it is restated elsewhere (for example, "Memory
    management" is in the template);
  - **move** to a change-controlled home: the template region, the vergil
    plugin, or repo docs. Engineering and workflow policy, such as
    "Correctness over cost" and "No silent failures", goes here;
  - **keep**, only when it is genuinely personal and has no other home.

  The moves land through ordinary PRs in the receiving repos. The revised
  global file itself is drafted **off GitHub**: in the session, or in a local
  scratch file. The human reviews it and applies it on the host. Its content
  never goes into an issue, PR body or report-ready note; the epic records
  only that it was reviewed and applied, and on what date.

### 5.3 Settings cleanup

- **Template and audit (vergil-tooling).** Drop `enabledPlugins` and
  `extraKnownMarketplaces` from the template. The audit then requires the
  repo file to equal the template exactly (after JSON parsing), which
  replaces the per-key checks. `render_claude_settings` stops seeding the
  ref. The audit change reaches the fleet only through a vergil-tooling
  release and pin bump, which are human actions.
- **Bridge.** The config audit is not part of `vrg-validate`. It runs in each
  repo's nightly `ops.yml` job, against the vergil-tooling version pinned in
  that repo. `vrg-update-deps` bumps the pin but doesn't rewrite the CLAUDE.md
  template region or `settings.json`. Without a bridge, every repo's nightly
  run would go red between its pin bump and its sweep PR. So, for one
  release, the audit accepts **either the previous or the new** shape of
  **both** the CLAUDE.md template region and `.claude/settings.json`, with a
  deprecation warning for the old shape (the #1974 pattern). The bridge is
  removed once every repo is swept.
- **Repo files.** Each of the nine repos is regenerated from the new
  template.
- **User file.** The agent drafts the changes off GitHub, as for the global
  `CLAUDE.md`, and the human applies them on the host. The drafted changes:
  - keep the `vergil-marketplace` definition, now the only one;
  - fix or remove the macOS-only `statusLine`;
  - confirm or remove the `AskUserQuestion` deny hook. This is the human's
    decision.
- **`vm_guest.py`.** Correct the 2.1.195 comments to match the changelog. The
  install/reconcile code stays unless §5.4 shows it is redundant; if so,
  removing it is a follow-up intake item, not part of this epic.

### 5.4 Verification on a real agent VM

A validation task, run after the settings and steering cleanup is deployed,
checks on a cold-rebuilt agent VM that:

- every enabled plugin is installed;
- the guard hook blocks a raw `git` call;
- a commit and a PR body made by Claude carry no attribution;
- each repo's `CLAUDE.md` loads and no `AGENTS.md` remains.

It also records whether user-enabled plugins installed without
`vm_guest`'s reconcile step, which settles §3.8.

### 5.5 Recurring review

There are two layers, matching what can be checked by a machine and what
needs judgment.

1. **Mechanical (the nightly config audit, `vrg-github-repo-config audit`
   in each repo's scheduled `ops.yml` run):**
   - repo `CLAUDE.md` at most 200 lines (a hard failure, enabled once the
     cleanup has landed fleet-wide);
   - template markers required;
   - no `AGENTS.md` unless `CLAUDE.md` imports it;
   - `.claude/settings.json` equal to the template;
   - the `paad-policy.md` pointer present (it is in the template region, so
     the template check covers it).
2. **Judgment (a new `steering-audit` skill in vergil-claude-plugin):** it
   reviews the steering and settings files. It borrows memory-audit's flow:
   verify each item, suggest a disposition, and wait for the human's
   decision. For steering files it runs `/doctor prompt-audit` and
   `/paad:pushback`, and it checks settings against the ownership model.
   - **Scopes.** It has two explicit scopes:
     - **repo** (the default): that repo's `CLAUDE.md`, `.claude/rules/` and
       `.claude/settings.json`;
     - **user** (`steering-audit user`): the global `~/.claude/CLAUDE.md` and
       `~/.claude/settings.json`, reviewed **once per cycle**, not once per
       repo. It checks every section against the "minimal; move it to a
       managed home" rule (§5.2). Drafts stay off GitHub.
   - **Memory is out of scope.** `memory-audit` is not changed. The memory
     mechanism is expected to be replaced (vergil-project/.github#561), so
     `steering-audit` neither reads nor depends on memory.
   - **When to run it:** at least quarterly, and whenever a Claude Code
     release changes settings or instruction-file behavior. The model
     document says so.
   - **What it re-verifies:** the model's platform facts, against the docs
     at the time of the run.
   - **Findings:** handled under umbrella §2.4. Each one is fixed, filed to
     intake, or dropped by the human with a reason.

## 6. Constraints

- Public repos: the repo files, issues, PR bodies, report-ready notes and
  the relay ref are public. The user-level files (`~/.claude/CLAUDE.md`,
  `~/.claude/settings.json`) are **personal**: their content never goes into
  any of those. Only the fact and date of their review are recorded.
- Agents use `vrg-git` and `vrg-gh` only, and stop at `report-ready`.
- Placement law: each task is filed in the repo whose PR closes it.
- **Ordering:** the template and audit change, then a human release and pin
  bump, then the per-repo sweeps, then the bridge removal and the hard
  line-budget check (each again through a release and pin bump). Every
  repo's nightly config audit and `vrg-validate` stay green throughout.
- Host files (`~/.claude/CLAUDE.md`, `~/.claude/settings.json`) are changed
  only by the human.

## 7. What is left to the builder (plan)

- Exact task breakdown and grouping (per repo vs batched).
- Exact template-region wording, and where in each repo's docs moved
  material lands.
- The bridge mechanics and the deprecation-warning text.
- Whether the audit compares the settings file whole or key by key, as long
  as the result is "equal to the template".
- How `steering-audit` invokes prompt-audit. If it cannot be
  invoked from a skill, the skill tells the human to run it and then reads
  the report.

## 8. Bookends and operational tasks

- Docs task: vergil-project/.github#559 (this spec and the plan).
- Documentation review: vergil-project/vergil-tooling#3720.
- Retrospective: vergil-project/.github#560.
- **Deployment task:** the vergil-tooling release with the new audit and
  template, and the fleet pin bump. The release is a human-attested
  precondition.
- **Validation task (§5.4):** cold-rebuild check on an agent VM, run after
  the sweeps and the user-file update.

## 9. Design decisions

The human skipped the brainstorm questions, so the agent made these calls.
The human confirmed the first three in design review (2026-10-27); the rest
stand as the agent's calls in the approved spec.

1. **Plugins and marketplaces move to user scope only** (confirmed). They
   are removed from the repo files and the template (§5.1, §5.3). Policy
   settings stay in the repo under the §5.1 placement rule. Reason: §3.5–3.6
   show that a second definition is either useless (a project-only enable
   doesn't install) or harmful (the whole-entry override, and a project
   `true` that the user cannot turn off).
2. **AGENTS.md is deleted, not made canonical or turned into a pointer**
   (confirmed). Reason: only Claude Code is used, and Claude Code ignores the
   file next to a `CLAUDE.md`.
3. **The recurring review is a new `steering-audit` skill** (the human's
   decision, replacing the agent's proposal to extend `memory-audit`). It
   uses `/doctor prompt-audit` as one of its engines. Memory is expected to
   be replaced, so the review must not be tied to it.
4. The line budget becomes a hard audit failure after the cleanup, not a
   warning.
5. The model document is a new vergil-tooling site page.
6. Managed settings stay unused.
7. The `AskUserQuestion` deny hook in the user file is left for the human to
   decide; the agent proposes no change to it.
