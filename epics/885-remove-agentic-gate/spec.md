# Install paad from the fork and remove the agentic-review gate

- **Epic:** vergil-project/.github#885
- **Planning issue:** vergil-project/.github#880 (review history is in its
  comments)
- **Status:** draft
- **Repos:** the paad fork (`wphillipmoore/paad`, human-gated), vergil-tooling
  (`vrg-vm session`'s `--plugin-dir` override and its docs),
  vergil-claude-plugin (`issue-implement`, the paad review doctrine, the paad
  policy and the skills that read the gate's records), `.github` (this spec
  and plan)

## 1. Problem

Two decisions made together have left the fleet unable to implement on cloud
VMs, and with a review step it no longer wants.

**`issue-implement` is Lima-only.** Its agentic-review gate (mode 2 of the
paad review doctrine) dispatches every review round to `paad:paad-delegate`,
paad's unattended runner. Only the vergil fork of paad has the delegate, and
the fleet loads the fork through `vrg-vm session`'s `--plugin-dir` override,
which points at a checkout on the Mac's projects tree
(`~/.config/vergil/plugin-dirs.toml` →
`/Users/pmoore/dev/projects/wphillipmoore/paad-fleet/preview/paad`). A cloud
VM has no such mount, so the override is skipped there and the session runs
with upstream paad, which has no delegate. `issue-implement`'s Preflight
step 7 then refuses to start. The override was set up as a temporary bridge;
the fork is now permanent, so the bridge has become the thing that broke
cloud implementation.

**The gate is too heavy.** The gate runs `paad:agentic-review` on every task
branch for up to three rounds, with a large record-keeping protocol around
it (doctrine §3, about 800 lines). Epic vergil-project/.github#826, which
would have made it proportionate, was closed as not planned: the fleet will
build a lighter reviewer of its own instead, from scratch, in
vergil-project/.github#878. Until then the gate is an obstacle.

## 2. Goals

1. **Every box installs paad from the fork.** Lima and cloud VMs alike get
   the fork's paad, delegate included, through the ordinary marketplace
   refresh that `vrg-vm` already runs. No session-time override.
2. **The fork stays a small delta ahead of upstream,** by a written,
   repeatable rebase process, run once in this epic onto paad 2.1.0.
3. **The `--plugin-dir` override is gone** from `vrg-vm session`, its
   configuration file, its tests and every doc that describes it.
4. **`issue-implement` is back to implement, validate to green,
   `report-ready`,** with no agentic-review gate and no dependency on the
   delegate, so it runs on any VM.
5. **No skill calls agentic-review**, and code that existed only to serve
   the gate is removed.

## 3. Non-goals

- **The replacement reviewer.** vergil-project/.github#878 designs it from
  scratch. This epic leaves no placeholder or hook for it.
- **Mode 1 design review** (spec pushback, plan pushback and alignment in
  `epic-create`) is unchanged in behaviour. Only its text about where the
  delegate comes from changes (§5.5). A side effect of goal 1 is that it
  also works on cloud VMs.
- **Other uses of paad** (`steering-audit`'s pushback run, interactive use of
  any paad skill) are unchanged.
- **Standalone improvements that arrived with the gate stay** (§6.2).

## 4. Standing order

Every `issue-implement` run on this epic's tasks **skips the agentic-review
gate**: it implements, validates to green and goes straight to
`report-ready`. The epic exists to remove the gate, and the installed plugin
still carries it until the removal ships in a plugin release. `epic-implement`
passes this order in every dispatch prompt for the epic. The task bodies
repeat it.

## 5. Install paad from the fork

### 5.1 The fork's `develop` branch

The fork `wphillipmoore/paad` gains a `develop` branch, and it is the one ref
the fleet installs from:

- `develop` is upstream `main` plus the fleet's patches, kept rebased on
  upstream (§5.3). It is pushed to GitHub.
- On `develop`, the root `.claude-plugin/marketplace.json` entry for `paad`
  has `"source": "./preview/paad"`. The fork's unattended-mode work, the
  delegate included, lives only in `preview/paad`; `plugins/paad` is
  upstream's released tree. Pointing the catalog at `preview/paad` is a
  one-line change that keeps the fleet's delta out of the files upstream
  edits most.
- The local `fleet` branch and the `paad-fleet` worktree
  (`/Users/pmoore/dev/projects/wphillipmoore/paad-fleet`) are retired once
  the fleet installs from `develop`.

### 5.2 Version scheme

`preview/paad/.claude-plugin/plugin.json`'s `version` (and the catalog
entry's, kept equal) is **the upstream base version plus a preview counter**:
`<base>-preview.<n>`, for example `2.1.0-preview.1`.

- Every push to `develop` that changes the plugin bumps `<n>`.
- A rebase onto a new upstream release sets `<base>` to that release and
  resets `<n>` to 1.

**Why.** Claude Code's plugin update is believed to key on the version, so an
unchanged version would leave a pushed change uninstalled. The first
validation (§7, V1) tests this. If an unchanged version does update, the
scheme still stands, because it names the upstream base and the fleet's
distance from it, but it is no longer what makes updates arrive.

### 5.3 The rebase process

Written into paad-policy §10 (§5.5) and run once in this epic:

1. Fetch upstream (`Ovid/paad`).
2. Rebase `develop` onto `upstream/main`. Drop any fleet patch upstream has
   since merged.
3. Set the version per §5.2.
4. Run the fork's own checks (`make test`).
5. Run the validation gate (§5.4) on the result.
6. Force-push `develop` (`--force-with-lease`).

Contributions upstream go as PRs from topic branches in the fork, never from
`develop`. Once upstream merges one, the next rebase drops the fleet's copy.

This epic's run rebases onto paad 2.1.0. That release carries upstream's fix
for the security issue the fleet reported, for which the fleet already has a
workaround, so the rebase changes no fleet behaviour.

### 5.4 The validation gate

Before any change reaches `develop`, including a rebase, the #733 validation
scenario sets (vergil-project/.github#733, spec §5) pass against the
candidate tree, in newly started sessions. The candidate is loaded with
Claude Code's own `claude --plugin-dir <candidate>/preview/paad` flag, as a
**manual test** run through `issue-validate`, not as fleet plumbing:
nothing in vergil configures or reads it.

Pushing `develop` is now the deliberate step that puts a paad change in front
of the fleet: the next marketplace refresh on each box installs it. Pushing
it only after the gate passes is the control, as moving `fleet` was before.

### 5.5 The host settings and the paad policy

**Host settings (human).** The `paad` entry in the host's
`~/.claude/settings.json` `extraKnownMarketplaces` becomes:

```json
"paad": {
  "source": { "source": "github", "repo": "wphillipmoore/paad", "ref": "develop" }
}
```

`vrg-vm` copies the file into each VM, and
`lib/claude_marketplace.py` registers the marketplace with
`claude plugin marketplace add wphillipmoore/paad@develop`. The plugin id
stays `paad@paad`, so no skill or agent name changes anywhere. The host
itself picks it up through `vrg-host plugins`. If Claude Code refuses to
re-add a marketplace whose repo changed under the same name, the existing
registration is removed first; V1 (§7) finds out.

**paad-policy §10 (vergil-claude-plugin)** is rewritten to the new
arrangement:

- the fleet installs paad from `wphillipmoore/paad` at ref `develop`, through
  the marketplace, on every box;
- the version scheme (§5.2), the rebase process (§5.3) and the validation
  gate (§5.4);
- the `--plugin-dir` override, the fleet checkout, the `fleet` branch, the
  "Lima only" paragraph and the accepted risks that only existed because of
  them (the read-write mount, a session started without the override) are
  removed;
- the delegate stays required for mode 1 rounds; a session without it is
  now a misconfigured box, and the fix it names is the host settings entry
  above.

The doctrine's §1.8 (the delegate requirement) is updated to match: the
delegate comes from the installed paad, the message names the settings
entry, and the text about the override, the fleet checkout and cloud boxes
having no delegate goes.

## 6. Remove the agentic-review gate

### 6.1 The rule

Every call to `paad:agentic-review` in the plugin's skills is removed, and so
is code and text that existed only to serve the gate. Anything that stands on
its own is kept (§6.2).

What goes:

- **`issue-implement`:** "The agentic-review gate" section; Preflight step 6
  (leftover `agentic-review/<N>/` state) and step 7 (the delegate check);
  the resume-a-paused-task line in step 8; the gate's mentions in the
  overview, the description and the creating-issues paragraph (the
  `epic-implement` ruling-carrying run that writes `security-<i>.md` into
  `agentic-review/<N>/` goes with it); the "Run this skill in a Lima
  session" paragraph and the Preflight wording that a cloud VM never has the
  delegate. The skill goes back to: Preflight, implement, validate to green,
  `report-ready`.
- **The paad review doctrine:** §3 (mode 2) in full, and every mode 2 branch
  in §1: the `agentic-review` column of §1.3's severity table, its row in
  §1.5's state-directory table, mode 2's handling in §1.6, §1.7, §1.8, §1.9
  and §1.10, and the introduction's description of two modes. The doctrine
  becomes the rules for mode 1 design review; its section numbers for §1 and
  §2 are kept, so existing links stay valid.
- **paad-policy:** the agentic-review output rule, the mode 2 row of the
  deviations table (the three-round cap), and §10's mentions of the gate.
- **`epic-implement`:** the list of tasks paused at the gate in its report,
  and any dispatch text about gate pauses and `fix` confirmations.
- **`epic-retrospective`:** the step that reads each task's
  `agentic-review summary` or round comments.
- **The write hook** (`hooks/scripts/block-worktree-bypass-write.sh`): its
  allowance for `build/.vergil-state/agentic-review/`, with its test cases.
  The `design-review/` allowance stays.
- **Docs:** the plugin README's and site skills page's descriptions of the
  gate and of `issue-implement` as Lima-only, and the "what a session needs
  for paad review rounds" section's Lima-only limit and override
  instructions.

### 6.2 What stays

- **Preflight step 5, the trusted-author check.** It stops an issue body
  from an untrusted author steering the implementation, which matters with or
  without a review. Its link moves from doctrine §3.3 (deleted) to §1.2
  (classification), and its pause becomes a plain stop: tell the human and
  wait for their ruling.
- **Creating issues needs the human's approval** (doctrine §1.7) and the
  `Awaiting a ruling` list: mode 1 and `issue-implement` both use it. Only
  its "inside the agentic-review gate" branch goes.
- **vergil-tooling's generic paad hygiene:** `vrg-git`'s read-only
  `symbolic-ref` allowance and the managed `.gitignore`'s `paad/code-reviews/`
  pattern. Both serve any run of paad's skills, not only the gate; their
  comments are reworded so they no longer name the gate as the reason.
- **`vrg-trusted-authors`** and its configuration, used by mode 1 and
  step 5.

### 6.3 Leftover state and in-flight tasks

- A task currently paused at the gate (a trusted `agentic-review round <n>:
  response` comment with a live `Paused:` line) is resumed after this epic
  as an ordinary `issue-implement` run. The plan lists any such tasks found
  when the removal is implemented.
- Leftover `build/.vergil-state/agentic-review/` directories in main clones
  are deleted by the human; agents report them and do not delete them.

## 7. Delivery order and validation

The fork half comes first: the gate removal does nothing for cloud
implementation until boxes have the delegate-free path, and the override
must not be removed until the fork install is proven.

1. **Fork (human-gated precondition).** §5.1–§5.4 on `wphillipmoore/paad`:
   rebase onto 2.1.0, catalog points at `preview/paad`, version
   `2.1.0-preview.1`, validation gate passes, `develop` pushed. The fork is
   outside the vergil-project org, so this is attested by the human on the
   epic, not filed as a fleet task.
2. **Host settings (human).** §5.5's settings entry.
3. **V1, validation:** on a freshly refreshed cloud VM and a Lima VM, with
   no `plugin-dirs.toml`, `paad@paad` is installed from `wphillipmoore/paad`
   at `develop` and a session lists `paad:paad-delegate`. Then a version
   bump pushed to `develop` reaches both boxes on refresh, and an unchanged
   version is tested to settle §5.2's question. The result decides whether
   the version bump is required, and is recorded in paad-policy.
4. **Remove the override (vergil-tooling),** blocked by V1:
   `lib/plugin_dirs.py`, its use and Lima-only message in `vrg-vm session`,
   `test_plugin_dirs.py`, the override cases in `test_vrg_vm.py` and
   `conftest.py`, and the docs (`cli-tools-overview.md`'s "The
   `--plugin-dir` override", `releasing.md`, `github-issues.md`). The human
   deletes `~/.config/vergil/plugin-dirs.toml`.
5. **paad policy and doctrine §1.8 (vergil-claude-plugin),** blocked by V1:
   §5.5's rewrite, with V1's version finding.
6. **Remove the gate (vergil-claude-plugin):** §6. Independent of the fork
   half; it can land in parallel.
7. **Gate mentions in vergil-tooling's docs:** `github-issues.md`'s gate
   section, `security-records.md`'s agentic-review paths and
   `account-setup.md`'s "used first by the agentic review gate".
8. **V2, validation,** blocked by the plugin release carrying 5 and 6:
   `issue-implement` runs end to end on a cloud VM on a real task and
   reaches `report-ready` with no gate and no delegate check. Plugin
   releases are human-gated, so the release is a precondition of V2, not a
   step any agent performs.

Then the closing bookends: the documentation review
(vergil-project/vergil-claude-plugin#916) and the retrospective
(vergil-project/.github#887).

## 8. Risks

- **Every push to the fork's `develop` reaches every box.** A mistaken push
  runs in every session's paad on its next refresh. The validation gate
  before each push is the control (§5.4), and it rests on the human's
  process, as moving `fleet` did.
- **Version keying.** If updates key on the version and a push forgets the
  bump, boxes silently keep the old paad. V1 settles whether this applies;
  if it does, the rebase process and paad-policy make the bump a checked
  step.
- **No pre-PR review until #878 lands.** Branches go to the human with only
  `vrg-validate` and CI behind them, as before the gate existed. Accepted by
  decision.
