# Steering-file and Claude settings review

- **Epic:** vergil-project/.github#558
- **Planning issue:** vergil-project/.github#506 (review history is in its
  comments)
- **Umbrella:** vergil-project/.github#500 (child E). Depends on B's Tasks 1
  and 3 (vergil-project/vergil-tooling#3666,
  vergil-project/vergil-claude-plugin#727), both closed.
- **Status:** draft, in design review
- **Brainstorm source:** umbrella spec §4.E

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
- **AGENTS.md files are dead and harmful.** By default, Claude Code does not
  read an `AGENTS.md` when a `CLAUDE.md` exists (§3), so these files are seen only by
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
  vergil tooling). Much of it is not personal at all:
  - its "Memory management" section is the authoritative memory policy. The
    template region (`data/claude_md_consumer.md:3-7`) points to it rather
    than restating it;
  - "Correctness over cost" and "No silent failures" are engineering policy.

### 2.1a The managed fleet

The template and the nightly config audit reach more than the nine
vergil-project repos. Any repo that pins `vergil = "v2.1"` and runs the
nightly `ops-github-config` audit is managed. On 2026-10-27 that is 16 repos:

- the nine vergil-project repos above;
- five in logical-minds-foundry: docs, mq-gateway-replay-lab,
  mq-protocol-gateway, mq-resiliency-lab-for-linux and
  mq-resiliency-observability;
- two in mnemosys-project: docs and melete.

These are out of scope:

- repos that pin `v2.1` but run no nightly audit (the mq-rest-admin-project
  repos);
- repos on `v2.0` (for example diogenes and the-infrastructure-mindset).

Throughout this spec, **"the fleet"** means the managed repos, found by a
mechanical scan at each step that needs the list (§6), not a hand-kept
list.

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
3. **Hooks** merge across user, project, local and plugin sources. In an
   interactive session, Claude Code holds back hooks from every settings
   file until the workspace trust dialog is accepted (`hooks.md`, "Workspace
   trust").
4. **Folder trust:** project `permissions.allow`, `extraKnownMarketplaces` and
   most `env` values "apply only after each teammate trusts the folder".
5. **`enabledPlugins`** merges per plugin, and the project file beats the user
   file: a user-level `false` cannot disable a plugin the project enables,
   but a `false` in `.claude/settings.local.json` can.
   - A plugin whose only `true` is in the project file is not fetched; a
     fetch needs a user, untracked-local, `--settings` or managed `true`.
   - A project `true` for a plugin that isn't installed is reported only in
     the `/plugin` **Errors** tab, which an agent session never opens.
   - **The local settings file:** Claude Code writes
     `.claude/settings.local.json` itself, on a standing approval ("Yes, and
     don't ask again") and for some `/config` options, and adds it to the
     global git excludes. In a worktree it reads and writes the copy at the
     **main checkout's root**, not the worktree's (`settings.md`).
   - After session start, Claude Code may still fetch a declared
     marketplace and download enabled plugins, so what is present at
     session start can differ from what is present a minute later
     (`plugins/loading.md`).
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
   - by default, `AGENTS.md` is read only when no `CLAUDE.md` exists
     (v2.1.277+), or when a `CLAUDE.md` imports it.
10. **`/doctor prompt-audit`** (v2.1.283+) is a built-in CLI command, not a
    skill, so an agent cannot invoke it through the Skill tool. It audits
    instruction files for stale paths and commands, contradictions, and
    patterns written for older models.
11. **Settings schema errors:** a value the schema rejects in a user,
    project or local file triggers a "Settings Error" dialog at the start of
    every interactive session (`settings.md`). Committed settings files must
    therefore hold only valid keys.

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

- Delivering fleet policy through a plugin `SessionStart` hook instead of the
  template. It may be worth a later look; it is out of scope here.
- Versioning the global `~/.claude/CLAUDE.md` or the user `settings.json` in a
  repo. They stay host files owned by the human. Keeping the global file
  minimal is what reduces the need for change control on it.
- Redesigning the memory mechanism. Memory is a candidate for replacement
  (intake item vergil-project/.github#561). This epic only **removes** the
  deprecated `memory-init`, `memory-audit` and `handoff` skills from
  vergil-claude-plugin (§5.2), keeps a short "no memory writes without
  approval" rule in the template, and adds no dependency on memory.
  `vrg-vm`'s memory projection is left for #561.
- Repos vergil doesn't manage (§2.1a). Rules that move out of the global
  `~/.claude/CLAUDE.md` into the vergil template no longer reach them, and
  that is accepted. A repo that needs them is brought under vergil
  management.
- Managed settings on agent VMs. They are the only scope a repo-local file
  cannot override, but they are not needed yet, and they raise the bar
  rather than close it. Revisit if the §5.3 local-settings guard proves
  inadequate.
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

**Placement rule.** A setting lives in the change-controlled repo
file unless the platform forces it into user scope. The repo file is
reviewed, versioned and audited nightly; the user file is none of those, and
it should stay minimal.

- **Forced into user scope:** plugin *installation* and the marketplace
  definitions. A project-only enable is never installed (§3.5), and a second
  marketplace definition overrides the first whole (§3.6).
- **Fleet policy, so it stays in the repo:**
  - the guard hook;
  - `attribution` off;
  - `allow Bash(vrg-*)`;
  - the enable for the vergil plugin. That plugin's own hooks enforce
    workflow policy, and a project `true` cannot be turned off by a
    user-level `false` (§3.5).

  In the repo file these apply to every checkout on any machine, including
  host sessions and fresh clones without the VM's copy of the user file.
  Fleet policy must not depend on the personal file.
- **Repo-local overrides are not allowed.** `.claude/settings.local.json`
  outranks the repo file. It stays gitignored, and in a managed repo the copy
  Claude Code actually loads must be absent or exactly `{}` (§3.5, §5.3).

The model:

| File | Owns | Must not hold |
|---|---|---|
| `~/.claude/settings.json` (host, copied verbatim to VMs) | plugin enablement (which installs them) and marketplaces for **all** plugins; the agent-VM permission mode (§9.9); personal model and UI settings | anything that works on only one OS, because the same file runs on the macOS host and on Linux VMs |
| repo `.claude/settings.json` (checked in, generated from the template, audited) | the guard `PreToolUse` hook; `permissions.allow: Bash(vrg-*)`; `attribution` off; `enabledPlugins: {"vergil@vergil-marketplace": true}` | `extraKnownMarketplaces` (a second definition overrides the user's whole entry, §3.6); any repo-specific key |
| `.claude/settings.local.json` (gitignored; checked by the Python chokepoints) | nothing: absent or exactly `{}` | any key |
| managed template `data/claude_settings.json` | the only source for the repo file's content | — |
| managed settings | unused (documented as deliberate, §4) | — |
| `~/.claude/CLAUDE.md` (host, human-owned, minimal) | only preferences that are genuinely personal and have no change-controlled home | engineering or workflow policy; anything a repo template, the plugin or repo docs can hold |
| template region of repo `CLAUDE.md` (`data/claude_md_consumer.md`) | fleet-wide agent policy: worktrees, `vrg-git`/`vrg-gh`, validation command, the PR handoff and freeze, the memory rule (condensed), engineering principles moved from the global file, the "`settings.local.json` stays absent or `{}`" rule, "report-ready text is public: no secrets", the `paad-policy.md` pointer | reference material; anything a hook enforces mechanically, beyond a one-line pointer |
| rest of repo `CLAUDE.md` | what an agent needs to work in that repo: overview, commands, constraints | catalogues and architecture detail (these go in site docs, linked); restated fleet policy |
| `.claude/rules/*.md` (path-scoped) | instructions that matter only for part of a repo | — |
| `AGENTS.md` | nothing: removed (§5.2) | — |

### 5.2 Steering-file cleanup

- **Template region** (`data/claude_md_consumer.md`):
  - target about 60 lines;
  - adds the PR-handoff and freeze rules (moved from vergil-tooling's
    `CLAUDE.md`), including one line saying report-ready title, summary and
    notes are public (the relay ref) and must hold no secrets;
  - adds the `paad-policy.md` pointer, one line saying
    `.claude/settings.local.json` must stay absent or `{}`, and every section
    the global-file classification moves into the template (below);
  - replaces the memory pointer with a short rule (no memory writes without
    explicit human approval), and drops the lines advertising the memory
    skills;
  - shortens the worktree section to the rules plus a link to the canonical
    spec, and the shell-policy section to a line, since the hook guard
    enforces it.
- **Every repo `CLAUDE.md`** is brought under 200 lines. Reference material
  moves into that repo's docs (site docs where they exist) and is linked;
  restated fleet policy is deleted; part-scoped guidance moves into
  `.claude/rules/` where it helps. Each repo is reviewed with
  `steering-audit` (§5.5) before and after its rewrite.
- **Deprecated skills removed.** `memory-init`, `memory-audit` and
  `handoff` are deleted from vergil-claude-plugin, along with their README,
  skills-catalogue and CLAUDE.md entries. `docs/paad-policy.md`'s handoff
  entry is updated: `paad:handoff` is the one to use.
- **Template markers.** Every managed repo still in the legacy form adopts
  the markers (five of the vergil-project repos, plus any found by the
  §2.1a scan). The audit's legacy path is then removed.
- **AGENTS.md** is deleted wherever a managed repo has one (today
  vergil-tooling, vergil-actions and vergil-claude-plugin). Anything in them
  that is still true and not already in `CLAUDE.md` is moved there first.
- **Global `~/.claude/CLAUDE.md`** is the priority, and the goal is to make
  it minimal. Its review happens in **two steps** (§6):
  1. **Classify, before the template is written.** Every section is sorted
     into one of these:
     - **delete**, because a change-controlled home already states it in
       full;
     - **move** to a change-controlled home: the template region, the
       vergil plugin, or repo docs. Engineering and workflow policy, such as
       "Correctness over cost" and "No silent failures", goes here, and so
       does "Memory management", condensed, because the template and
       `MEMORY.md` headers point to it today;
     - **keep**, only when it is genuinely personal and has no other home.

     Moves into the template are part of the first template change, so no
     second template cycle is needed.
  2. **Apply, after every sweep has merged.** The moved and deleted sections
     come out of the host file only once every managed repo carries the new
     template region, so no repo is ever without a rule.

  Moves to homes other than the template land through ordinary PRs in the
  receiving repos. The global file has no change control, so it is reviewed
  **in the session only**: the agent writes no file of any kind, and the
  human decides and makes every change on the host. Its content never goes
  into an issue, PR body or report-ready note; the epic records only that it
  was reviewed and applied, and on what date. Change control for personal
  files is intake item vergil-project/.github#586.

### 5.3 Settings cleanup

- **Template and audit (vergil-tooling).** Drop `extraKnownMarketplaces`
  from the template; keep `enabledPlugins: {"vergil@vergil-marketplace":
  true}`. The audit then requires the repo file to equal the template
  exactly (after JSON parsing), which replaces the per-key checks.
  `render_claude_settings` stops seeding the ref. The audit change reaches
  the fleet only through a vergil-tooling release, which is a human action.
- **Local settings file.** `.claude/settings.local.json` stays in the
  managed `.gitignore` block, untracked. `vrg-commit`,
  `vrg-pr-workflow report-ready` and `vrg-container-run` (host-side) check
  **both** copies Claude Code may load:
  - the main checkout's root, found through `git rev-parse
    --git-common-dir` (§3.5);
  - the current worktree's own copy.

  For an **agent** identity they refuse to run if either copy exists with
  any content other than `{}`. A **human** identity gets the same message as
  a warning and the tool proceeds: the file changes agent behavior, not the
  human's own release and dependency-update runs. An absent file passes. The
  message says why the file must stay absent or `{}`, and that any rule it
  held belongs in the repo template or the user file.
  - `vrg-validate`, inside the container, checks the worktree copy. It
    cannot see the main checkout's working tree there, so it says so rather
    than passing silently.
  - These are the agent's chokepoints, and they don't depend on Claude Code
    hooks. The hook guard is not one: a local file that disables hooks
    would disable it too.
  - The nightly audit does not check this file: it can't see an ignored
    file in a CI checkout.
- **Release model.** Every consuming repo pins `vergil = "v2.1"`, the
  rolling minor tag that each patch release force-updates
  (`vergil-tooling/docs/site/docs/guides/releasing.md`). Releases are cut
  from `develop` often (several a day), so any release ships whatever has
  merged. Both audit changes therefore ship as **v2.1 patch releases**, and
  each reaches every managed repo on the next nightly run, with no per-repo
  pin bump. The gate is on **merging**, not on releasing: an audit-change PR
  must not merge to `develop` until its precondition holds (§6).
- **vergil-tooling is the exception.** Its own nightly audit runs from source
  (`uv sync`, not the `v2.1` tag), so it checks `develop` the moment a PR
  merges. Each audit-change PR therefore updates vergil-tooling's own
  `CLAUDE.md` region and settings files in the same PR.
- **Bridge.** The config audit is not part of `vrg-validate`. It runs in each
  repo's nightly `ops.yml` job. A release doesn't rewrite any repo's
  CLAUDE.md template region or settings files. Without a bridge, every
  managed repo's nightly run would go red the night after the release and
  stay red until its sweep PR. So, **until the bridge-removal PR merges**,
  the audit accepts **either the previous or the new** shape of the CLAUDE.md
  template region and `.claude/settings.json`, with a deprecation warning
  for the old shape (the #1974 pattern).
- **Bridge removal.** This PR must not merge until every managed repo's
  sweep has merged, because the next release takes it fleet-wide that
  night. It also removes the code the new shape makes dead:
  `normalize_claude_ref` in `vrg-update-deps` and the marketplace-ref
  constants, wherever nothing else uses them.
- **Repo files.** Every managed repo's settings files are regenerated from
  the new template. A standing mechanism for later template changes (an
  audit that always accepts the current and previous template) is out of
  scope; it is intake item vergil-project/.github#580.
- **User file.** Reviewed in the session only, as for the global
  `CLAUDE.md`; the human makes the changes on the host. The changes to
  consider:
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
- a commit made by an agent has no `Co-Authored-By` trailer, and the
  `report-ready` summary and notes carry no attribution line (agents never
  build a PR body themselves);
- each repo's `CLAUDE.md` loads and no `AGENTS.md` remains;
- putting any key into `.claude/settings.local.json` at the **main
  checkout's root** makes `vrg-commit` in a worktree refuse, and removing
  the file lets it proceed.

It also records whether user-enabled plugins are installed **when a session
starts, without `/reload-plugins`** (§3.5), and without `vm_guest`'s
reconcile step. A rebuild always runs that step, so this is a separate probe:
uninstall the plugins, start a fresh session, and record what came back.
That settles §3.8.

### 5.5 Recurring review

There are two layers, matching what can be checked by a machine and what
needs judgment.

1. **Mechanical (the nightly config audit, `vrg-github-repo-config audit`
   in each repo's scheduled `ops.yml` run):**
   - repo `CLAUDE.md` at most 200 lines (a hard failure, enabled once the
     cleanup has landed fleet-wide);
   - template markers required;
   - no `AGENTS.md`;
   - `.claude/settings.json` equal to the template;
   - the `paad-policy.md` pointer present (it is in the template region, so
     the template check covers it).
2. **Judgment (a new `steering-audit` skill in vergil-claude-plugin):** it
   reviews the steering and settings files. For each item it verifies,
   suggests a disposition, and waits for the human's decision.
   - **Steering files:** it runs `/paad:pushback` as a findings-only review
     (no edits, no report files), plus its own mechanical
     checks: every referenced path exists, every named command resolves,
     the line count, and the ownership rules.
   - **Settings files:** it checks them against the ownership model.
   - **Optional extra:** `/doctor prompt-audit` (§3.10) can't be run by an
     agent. The model page mentions it as an optional check the human can
     run by hand; the skill does not depend on it.
   - **Scopes.** It has two explicit scopes:
     - **repo** (the default): that repo's `CLAUDE.md`, `.claude/rules/` and
       `.claude/settings.json`;
     - **user** (`steering-audit user`): the global `~/.claude/CLAUDE.md` and
       `~/.claude/settings.json`, reviewed **once per cycle**, not once per
       repo. It checks every section against the "minimal; move it to a
       managed home" rule (§5.2). User scope is **read-only and
       interactive only**: these files have no change control (§5.2), so
       the skill writes no file of any kind, and every change is the
       human's, made on the host. **Its findings never go to intake**: the
       public queue would expose personal content. The public record shows
       only that the review ran, its date, and counts. This is the user
       scope's exception to umbrella §2.4.
   - **Memory is out of scope.** `steering-audit` neither reads nor depends
     on memory (vergil-project/.github#561).
   - **When to run it:** at least quarterly, and whenever a Claude Code
     release changes settings or instruction-file behavior. The model
     document says so.
   - **What it re-verifies:** the model's platform facts, against the docs
     at the time of the run.
   - **Findings (repo scope):** handled under umbrella §2.4. Each one is
     fixed, filed to intake, or dropped by the human with a reason.

## 6. Constraints

- Public repos: the repo files, issues, PR bodies, report-ready notes and
  the relay ref are public. The user-level files (`~/.claude/CLAUDE.md`,
  `~/.claude/settings.json`) are **personal**: their content never goes into
  any of those. Only the fact and date of their review are recorded.
- Agents use `vrg-git` and `vrg-gh` only, and stop at `report-ready`.
- Placement law: each task is filed in the repo whose PR closes it.
- **Ordering:**
  1. classify the global file's sections (§5.2 step 1);
  2. the model page (§5.1) merged;
  3. the template and audit change, carrying the moves into the template;
     and a human v2.1 patch release of it;
  4. `steering-audit` merged **and released** in vergil-claude-plugin, so
     it is on the `main` marketplace ref that agents load (a human-attested
     release);
  5. the per-repo sweeps across every managed repo (§2.1a), found by a
     fresh scan. Each sweep needs steps 2–4;
  6. apply the user-level changes on the host, after **every** sweep has
     merged, so no repo is ever without a rule that moved out of the global
     file;
  7. the bridge removal and the hard line-budget check. The PR must not
     **merge** until every sweep has merged; the next v2.1 patch release
     then ships it.

  Every managed repo's nightly config audit and `vrg-validate` stay green
  throughout.
- **Cross-org sweeps.** Tasks are never linked across orgs. A sweep in
  another org (logical-minds-foundry, mnemosys-project) is filed under that
  repo's own ad-hoc epic. This epic carries one validation gate task that
  checks every such sweep is closed, and the bridge removal is blocked by
  that gate.
- Host files (`~/.claude/CLAUDE.md`, `~/.claude/settings.json`) are changed
  only by the human.

## 7. What is left to the builder (plan)

- Exact task breakdown and grouping (per repo vs batched).
- Exact template-region wording, and where in each repo's docs moved
  material lands.
- The bridge mechanics and the deprecation-warning text.
- Whether the audit compares the settings file whole or key by key, as long
  as the result is "equal to the template".
- The exact wording of the "`settings.local.json` must stay absent or `{}`"
  refusal.

## 8. Bookends and operational tasks

- Docs task: vergil-project/.github#559 (this spec and the plan).
- Documentation review: vergil-project/vergil-tooling#3720.
- Retrospective: vergil-project/.github#560.
- **Two deployment tasks**, one per v2.1 patch release: the bridged audit
  with the new templates, and later the tightened audit. Each release is a
  human-attested precondition. The task checks the next nightly audit
  across the fleet.
- **A plugin deployment task:** the vergil-claude-plugin release that puts
  `steering-audit` on the `main` marketplace ref (§6 step 4). Every sweep is
  blocked by it.
- **Validation task (§5.4):** cold-rebuild check on an agent VM, run after
  the sweeps and the user-file update.
- **Validation gate task (§6):** checks that every cross-org sweep is
  closed. The bridge removal is blocked by it.

## 9. Design decisions

1. **Marketplaces move to user scope only; the vergil plugin's enable stays
   in the repo too.** `extraKnownMarketplaces` is removed from the repo files
   and the template, because a second definition overrides the first whole
   (§3.6). The user file still installs every plugin (§3.5). The repo keeps
   `"vergil@vergil-marketplace": true` because that plugin's hooks enforce
   fleet policy, a user-level `false` cannot override a project `true`, and
   the line costs nothing.
2. **AGENTS.md is deleted, not made canonical or turned into a pointer.**
   Reason: only Claude Code is used, and by default Claude Code ignores the
   file next to a `CLAUDE.md`.
3. **The recurring review is a new `steering-audit` skill.** Its engine is
   `/paad:pushback` plus its own mechanical checks. The deprecated memory
   and handoff skills are removed rather than extended; memory is expected
   to be replaced, so the review is not tied to it.
4. The line budget becomes a hard audit failure after the cleanup, not a
   warning.
5. The model document is a new vergil-tooling site page.
6. Managed settings stay unused (§4).
7. The `AskUserQuestion` deny hook in the user file is the human's to
   decide; no change to it is proposed.
8. **`.claude/settings.local.json` stays gitignored, and the Python
   chokepoints guard it.** They check the copy Claude Code actually loads
   (the main checkout's root) as well as the worktree's, and refuse any
   content other than `{}`. Committing the file was rejected: Claude Code
   writes the file itself, and the tracked copy is not the one that takes
   effect.
9. **The user file carries the agent-VM permission mode.** Because the host
   and the VMs share that file, **every** Claude Code session started on the
   host uses it. By design, the only such sessions are VM sessions the human
   moves onto the host for supervised admin or debugging work. That is a
   deliberate, rare, accepted exception, and the recurring review treats it
   as correct. Setting the mode at `vrg-vm` session launch instead is intake
   item vergil-project/.github#583.
