# Install paad from the fork and remove the agentic-review gate

- **Epic:** vergil-project/.github#885
- **Planning issue:** vergil-project/.github#880 (review history is in its
  comments)
- **Status:** draft
- **Repos:** the paad fork (`wphillipmoore/paad`, changed in the human's
  deployment task D1), vergil-tooling
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
gate and Preflight steps 6 and 7** (the leftover `agentic-review/<N>/`
check and the delegate check): it implements, validates to green and goes
straight to `report-ready`. The epic exists to remove the gate, and the
installed plugin still carries it until the removal ships in a plugin release. `epic-implement`
passes this order in every dispatch prompt for the epic. The task bodies
repeat it.

## 5. Install paad from the fork

### 5.1 The fork's `develop` branch

The fork `wphillipmoore/paad` gains a `develop` branch, and it is the one ref
the fleet installs from:

- `develop` is upstream's latest release tag plus the fleet's patches, kept
  rebased on each new upstream release (§5.3). It is pushed to GitHub. It
  never carries upstream's unreleased work: the fleet takes in upstream's
  preview changes only when upstream releases them.
- On `develop`, the root `.claude-plugin/marketplace.json` entry for `paad`
  has `"source": "./preview/paad"`. The fork's unattended-mode work, the
  delegate included, lives only in `preview/paad`; `plugins/paad` is
  upstream's released tree.
- The local `fleet` branch and the `paad-fleet` worktree
  (`/Users/pmoore/dev/projects/wphillipmoore/paad-fleet`) are retired last,
  when the override is removed (§7 step 3).

### 5.2 Version scheme

The fork keeps upstream's own version form, unchanged: the catalog entry,
`package.json` and `plugins/paad` carry the upstream base (`2.1.0`), and
`preview/paad/.claude-plugin/plugin.json` carries `<base>-preview`
(`2.1.0-preview`), with every `SKILL.md` announcing it. This is what the
fork's `make check-versions` and `check-skill-versions` require, so the
fleet adds no delta to the Makefile or the announce lines. A rebase onto a
new upstream release takes upstream's new versions as they are.

**Open question, settled by V1 (§7).** Claude Code may decide whether to
replace an installed plugin by its version rather than by the commit at the
ref. If it does, a push to `develop` that keeps the same version would not
reach installed boxes. V1 tests this. If V1 shows the version decides it, a
version scheme that changes on every push is designed before the override
is removed (§7 step 3), and recorded in paad-policy.

### 5.3 The rebase process

Written into paad-policy §10 (§5.5) and run once in this epic:

1. Fetch upstream (`Ovid/paad`) with its tags.
2. Rebase `develop` onto upstream's release tag (`paad--v<X.Y.Z>`), never
   onto `upstream/main`. Drop any fleet patch the release already contains.
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

A push that changes no behaviour (a version-only or documentation-only
change, such as V1's update test in §7) is exempt from the scenario sets.

Pushing `develop` is now the deliberate step that puts a paad change in front
of the fleet: the next marketplace refresh on each box installs it. Pushing
it only after the gate passes is the control, as moving `fleet` was before.

### 5.5 The host settings and the paad policy

**Host settings (human, in D1, §7).** The `paad` entry in the host's
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

**paad-policy (vergil-claude-plugin)** is rewritten to the new arrangement,
wherever it describes the old one: §1 (paad's version and the installed
paad), the deviations table's rows about the installed release and the
fleet checkout, §9, and §10. §10 states:

- the fleet installs paad from `wphillipmoore/paad` at ref `develop`, through
  the marketplace, on every box;
- the version scheme (§5.2), the rebase process (§5.3) and the validation
  gate (§5.4);
- **only the human changes the fork:** every push, rebase and branch move
  in `wphillipmoore/paad` is run by the human. Agents never run git
  commands that change the fork (`vrg-git` does not stop them); they give
  the human the commands to run;
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
  overview, the description and the creating-issues paragraph; the "Run
  this skill in a Lima session" paragraph and the Preflight wording that a
  cloud VM never has the delegate. The skill goes back to: Preflight, implement, validate to green,
  `report-ready`.
- **The paad review doctrine:** §3 (mode 2) in full, and every mode 2 branch
  in §1: the `agentic-review` column of §1.3's severity table, mode 2's
  handling in §1.6, §1.7, §1.8, §1.9 and §1.10, and the introduction's
  description of two modes. The doctrine
  becomes the rules for mode 1 design review; its section numbers for §1 and
  §2 are kept, so existing links stay valid.
- **paad-policy:** the agentic-review output rule, the mode 2 row of the
  deviations table (the three-round cap), and §10's mentions of the gate.
- **`epic-implement`:** the list of tasks paused at the gate in its report,
  and any dispatch text about gate pauses and `fix` confirmations.
- **`epic-retrospective`:** the step that reads each task's
  `agentic-review summary` or round comments.
- **The write hook** (`hooks/scripts/block-worktree-bypass-write.sh`): the
  hook already allows any git-ignored path under `build/`, with no per-kind
  allowance, so its logic is unchanged. Only its header comment and its
  test's `agentic-review` fixture paths change, to the renamed
  `issue-implement` kind (§6.2).
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
- **`issue-implement`'s private directory, renamed.** `issue-implement`
  already handles a security follow-up it finds as §1.7 says: a count in
  public, the details private until the human rules, filed only through
  `vrg-security-record`. Its private directory changes name only: §1.5's
  `agentic-review` row becomes `issue-implement`,
  `build/.vergil-state/issue-implement/<N>/`, and the `epic-implement`
  run that carries out the human's rulings writes a `record` ruling's
  `security-<i>.md` there.
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

The fleet has one user, so the epic runs as a freeze: no other fleet work
runs from D1 until the override is removed. Anything that breaks in between
is fixed as it is found.

**The file lists in this spec are starting points, not the scope.** Each
removal task starts with a search of vergil-tooling, vergil-claude-plugin
and `.github` for every mention of what it removes (`plugin-dirs`,
`plugin_dirs`, `--plugin-dir`, `paad-fleet`, the fleet checkout,
`agentic-review`, mode 2), and handles every hit outside archived and
historical records (`epics/`, `CHANGELOG.md`, `releases/`, completed plans
and specs). The documentation review sweeps again at the end.

1. **D1, deployment: install paad from the fork.** A task the human carries
   out, with the agent guiding; it closes on a recorded `Outcome: SUCCESS`
   like any operational task. Its procedure: §5.1–§5.4 on
   `wphillipmoore/paad` (rebase onto 2.1.0, catalog points at
   `preview/paad`, versions per §5.2, validation gate passes,
   `develop` pushed), then §5.5's host settings entry. The fork is outside
   the vergil-project org, so the fork's own changes have no fleet PR; the
   task records what was pushed (the `develop` commit) in its outcome.
2. **V1, validation,** blocked by D1: on a freshly refreshed cloud VM (which
   never applies the override), `paad@paad` is installed from
   `wphillipmoore/paad` at `develop` and a session lists
   `paad:paad-delegate`. Then a change pushed to `develop` with the version
   unchanged is checked for whether it reaches the box on refresh, settling
   §5.2's open question. The result is recorded in paad-policy.
3. **Remove the override (vergil-tooling),** blocked by V1:
   `lib/plugin_dirs.py`, its use and Lima-only message in `vrg-vm session`,
   `test_plugin_dirs.py`, the override cases in `test_vrg_vm.py` and
   `conftest.py`, and the docs (`cli-tools-overview.md`'s "The
   `--plugin-dir` override", `releasing.md`, `github-issues.md`, and in
   vergil-claude-plugin the README's Update section and the site
   `index.md` paragraph on loading a tree with `--plugin-dir`). The human
   then deletes `~/.config/vergil/plugin-dirs.toml`, retires the `fleet`
   branch and the `paad-fleet` worktree, and confirms a Lima session lists
   `paad:paad-delegate`.
4. **paad policy and doctrine §1.8 (vergil-claude-plugin),** blocked by V1:
   §5.5's rewrite, with V1's version finding.
5. **Remove the gate (vergil-claude-plugin):** §6. Independent of the fork
   half; it can land in parallel.
6. **Gate mentions in vergil-tooling's docs:** `github-issues.md`'s gate
   section, `security-records.md`'s agentic-review paths and
   `account-setup.md`'s "used first by the agentic review gate".
7. **V2, validation,** blocked by the plugin release carrying 4 and 5:
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
- **Version keying.** If updates key on the version, a push that keeps the
  version leaves boxes silently on the old paad. V1 settles whether this
  applies; if it does, §5.2's follow-up scheme is designed before the
  override is removed.
- **No pre-PR review until #878 lands.** Branches go to the human with only
  `vrg-validate` and CI behind them, as before the gate existed. Accepted by
  decision.
