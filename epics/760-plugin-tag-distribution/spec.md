# Plugin distribution via the floating `v2.1` tag

- **Epic:** vergil-project/.github#760
- **Planning issue:** vergil-project/.github#759 (review history is in its
  comments)
- **Status:** in design review
- **Repos:** vergil-claude-plugin (its distribution docs), vergil-tooling (the
  consumer setup guide and a rollback runbook), `.github` (this spec and plan)
- **Supersedes:** the `main`-only decision of epic vergil-project/.github#45
  ([spec](../45-plugin-distribution-model/spec.md))

## 1. Problem

Every consumer registers the `vergil-marketplace` with `"ref": "main"` in the
user-level `~/.claude/settings.json`. The deployed plugin is therefore whatever
`main` holds, and the only way to put the fleet back on an earlier release is a
PR that changes `main`. In practice the plugin cannot be rolled back. The
integration of the paad skills was the first time a release needed reverting,
and the plugin is about to take a run of major changes in which instability
is expected. So the fleet needs a general rollback mechanism. Whether it is
used for the paad integration is a separate decision, outside this epic.

vergil-tooling and vergil-actions do not have this problem. Their consumers
reference the floating `v2.1` tag, which each release moves forward and which
a human can move back.

## 2. Decision

**The `vergil-marketplace` source moves from `"ref": "main"` to `"ref": "v2.1"`.**

```json
{
  "extraKnownMarketplaces": {
    "vergil-marketplace": {
      "source": {
        "source": "github",
        "repo": "vergil-project/vergil-claude-plugin",
        "ref": "v2.1"
      }
    }
  }
}
```

- There is still **one channel and one marketplace**: every consumer tracks
  `v2.1`. The consumers are one user's fleet, and the deployment switches all
  of it at once: the host, every agent VM, and every repo under the
  `vergil-project` working tree. There are no outside consumers to notify or
  migrate. `develop` is still not consumed through a marketplace (epic #45 §2
  stands).
- `main` stays the release branch. The tag, not the branch, is what consumers
  follow, so what is deployed can differ from what `main` holds.
- **Deploying a release** is moving `v2.1` forward to `v2.1.Z`, which the release
  already does (§4).
- **Rolling back** is moving `v2.1` back to an earlier `v2.1.Z` with
  `vrg-promote` (§5). It is a human act, gated like any release.

**The rolling tag is the deploy ref, so it is protected like one.**

- Every `vX.Y` rolling tag, in every repo whose consumers follow one, is
  covered by a tag ruleset that blocks update and deletion except by the
  repository Admin role, the human release identity. The org's tag-protection
  ruleset in vergil-tooling (`lib/github_config.py`) declares it, and the daily
  config audit checks it. The audit runs on a GitHub App token, which cannot
  read a ruleset's bypass list, so it checks the protected refs and rules but
  not the bypass actors. This holds for every existing ruleset too.
- No branch may carry a rolling-tag name. A branch ruleset, declared in the
  same module and checked by the same audit, blocks creating any branch
  matching `refs/heads/v*`, with no bypass or only the repository Admin role.
  Claude Code resolves a ref as a branch before a tag (§3), so the tag's
  protection holds only while no branch of that name can exist.
- `vrg-promote` moves a rolling tag only to an existing `vX.Y.Z` release tag
  whose commit is on `origin/main`, and refuses any other target.
- `vrg-promote` refuses an agent identity, with the same check `vrg-release`
  uses (`identity_mode.is_agent()`, failing closed on any agent signal).
  Moving a rolling tag is a human act.
- **Order:** both rulesets are live on vergil-claude-plugin, and the audit
  reports them clean, before any machine switches to `v2.1`.

This reverses the part of epic #45 that ruled out `vX.Y` tags. #45's arguments
were against multiple channels and SemVer pinning (consumers holding different
versions). A single floating tag does neither: it is an indirection that gives
the plugin the same deploy and rollback control as the rest of the fleet. #45's
other decisions (no development marketplace, no per-branch marketplace `name`,
`develop` read on demand) are unchanged.

## 3. Evidence that Claude Code follows a floating tag

Verified against Claude Code 2.1.312 in an isolated `CLAUDE_CONFIG_DIR`, so the
real install was untouched:

- **Docs.** `ref` on a `github` or `git` marketplace source is "Branch or tag.
  Defaults to the repository's default branch"
  (<https://code.claude.com/docs/en/plugins/marketplace-reference.md>, "Fields
  by type").
- **A tag installs.** `claude plugin marketplace add
  vergil-project/vergil-claude-plugin#v2.1.50` followed by `claude plugin
  install vergil@vergil-marketplace` installed version `2.1.50` at commit
  `fc17882`, the commit `v2.1.50` peels to.
- **A moved tag is detected in either direction.** On update, the CLI's
  marketplace checkout probe runs `git ls-remote` for `refs/heads/<ref>`,
  `refs/tags/<ref>^{}` and `refs/tags/<ref>`, compares the first match with the
  commit the local checkout's HEAD is on, and re-clones on any difference
  (debug text: `Marketplace checkout probe: local <sha> remote <sha> — re-cloning`).
  The comparison is equality, not ordering, so a backward move is detected like
  a forward one.
- **A downgrade installs.** With 2.1.51 installed, the ref was made to resolve to
  the `v2.1.50` commit. `claude plugin marketplace update vergil-marketplace`
  printed `Replacing the existing marketplace clone…`, and `claude plugin update
  vergil@vergil-marketplace` printed `Plugin "vergil" updated from 2.1.51 to
  2.1.50`. The plugin cache compares versions for difference, so the release
  version in `plugin.json` must change with every release, which it does.

## 4. What already exists

- **The tag.** `vergil-claude-plugin` already carries `v2.1`, at the same commit
  as `v2.1.51` and `main` at the time of writing.
- **Forward moves.** `vrg-release` runs `promote` (vergil-tooling
  `lib/promote.py`), which force-updates `vX.Y` to the new `vX.Y.Z` and pushes
  it. The plugin's `cd.yml` calls vergil-actions' `cd-release.yml@v2.1`.
- **Backward moves.** `vrg-promote <X.Y.Z>` has no direction check: given an
  earlier release it moves `vX.Y` back to it. No new tool is needed.
- **VMs.** `vrg-vm` copies the host `~/.claude/settings.json` into the guest on
  every start and every cloud session (`_st_copy_config`). Its plugin refresh
  (`vm_guest.update_plugins`) re-runs `claude plugin marketplace add
  owner/repo@ref` for each declared marketplace, which is a no-op when the
  marketplace is already registered. So once the host declares `v2.1`, an
  existing VM declares `v2.1` while still registered at `main`, which is the
  mismatch §6 describes. Whether the VM refresh needs a code change depends on
  the switch mechanism §6 proves.

## 5. Rollback procedure

1. A human moves `v2.1` back to the release to return to. This is a release
   act and stays human-gated. The steps are exact, because `vrg-promote` acts
   on the repo of the current directory and the fleet's repos share tag names
   (`v2.1.50` exists in vergil-tooling and vergil-actions too):
   1. `cd` into a vergil-claude-plugin checkout;
   2. `vrg-git fetch --tags --force origin` (the rolling tag moves every
      release, and without `--force` the fetch refuses to update a local tag
      that has moved, as the release code already allows for);
   3. `vrg-promote --dry-run <X.Y.Z>`, and check the tag move it prints;
   4. `vrg-promote <X.Y.Z>`.

   Before it force-pushes, `vrg-promote` run directly prints the origin repo,
   the rolling tag and the old and new commits, and asks for confirmation.
   The prompt lives only in the `vrg-promote` command (`bin/vrg_promote.py`);
   the promote stage of `vrg-release` calls the library and is not prompted.
   With no terminal to ask on, `vrg-promote` refuses rather than proceeds.
   `--dry-run` prints the same origin repo, rolling tag and commits, and
   applies the §2 target check, so step 3 shows exactly what step 4 would do.
2. Each machine picks the move up the next time its marketplace and plugin
   update: `claude plugin marketplace update vergil-marketplace`, then `claude
   plugin update vergil@vergil-marketplace` (background auto-update does the
   same where it is on), then a restart of Claude Code. VMs pick it up with
   `vrg-vm update`.
3. The rollback lasts until the next release, which moves `v2.1` forward again.
   Keeping it in place until the fix ships is managed by hand: the human who
   rolled back makes sure the next release carries the fix, or cuts it with
   `vrg-release --no-promote`. There is no mechanism that holds the tag. An
   issue that blocks promotion until it is closed is a separate idea
   (vergil-project/.github#763).

The procedure is documented in a rollback section of vergil-tooling's release
guide, `docs/site/docs/guides/releasing.md`, and the plugin README points to
it.

## 6. Switching existing machines

Changing the `ref` string of a marketplace that is already registered is a
source change, not an update. In the isolated test, editing only the `ref` in
`settings.json` left `known_marketplaces.json` on the old source, and `claude
plugin marketplace update vergil-marketplace` reported `Marketplace
'vergil-marketplace' not found`. Re-running `claude plugin marketplace add`
over the existing registration printed `already on disk` and changed nothing.

The switch is **fully mechanized**: no machine, host or VM, is switched by
hand. Re-adding over the existing registration is already shown not to work.
The mechanism is the lightest one that is proven to work, tried in this
order:

1. rewriting the registered source to match the declared one, then updating;
2. `claude plugin marketplace remove vergil-marketplace`, then `add` at
   `v2.1` and reinstalling `vergil@vergil-marketplace`. `remove` also
   uninstalls the marketplace's plugins and deletes their data, options and
   secrets, so this step is taken only if the lighter ones fail, and the data
   it deletes is listed in the setup guide.

The proof runs in an isolated `CLAUDE_CONFIG_DIR` and covers two cases: a host
(the user settings edited, then the mechanism run) and a VM (an existing
`main` registration, the new `settings.json` copied in as `vrg-vm` does, then
`vm_guest.update_plugins`). If the VM case does not switch the source, a
vergil-tooling task teaches `update_plugins` the proven mechanism, so that
`vrg-vm update` switches every VM with no manual step. The proven steps go into
the consumer setup guide.

## 7. Changes

- **vergil-claude-plugin:** `README.md` (the "Distribution model" and "Install"
  sections), `CLAUDE.md` (the distribution paragraph) and
  `docs/site/docs/index.md` (the settings example) describe `v2.1` and the
  rollback, and link here. The "external orgs" wording in `README.md` and
  `CLAUDE.md` goes, since there are none.
- **vergil-tooling:** `docs/site/docs/guides/consuming-repo-setup.md` (the
  marketplace definition and the switch procedure from §6) and a rollback
  section in the release guide, `docs/site/docs/guides/releasing.md` (§5). Test fixtures that use `"ref": "main"` are
  sample data for a ref-agnostic function and are left alone.
- **`.github`:** this spec and plan. Epic #45's spec gets one line under its
  header pointing to this spec as superseding its tag decision. Its text is not
  otherwise rewritten.
- **vergil-tooling:** the rolling-tag ruleset, the rolling-tag branch-name
  ruleset, and the `vrg-promote` target check and agent refusal (§2), and the `vrg-promote` confirmation (§5).
- **vergil-tooling, if the §6 proof needs it:** `vm_guest.update_plugins`
  learns the proven switch mechanism.
- **Deployment:** a sweep of the whole fleet, started only once the §2 order
  condition holds (the rulesets live and audited clean). The host `~/.claude/settings.json`
  and every VM move to `v2.1` by the mechanism §6 proves. Every repo in the
  local working tree is checked for a project-level `extraKnownMarketplaces`
  entry for `vergil-marketplace`. Such an entry replaces the user entry as a
  whole, so one left at `main` would undo the switch in that repo. Any that is
  found is removed in favour of the template, which defines no marketplace.
- **Validation:** an experimental round trip on the real fleet. `v2.1` is
  rolled back by §5 to an earlier release chosen when the validation runs, the
  rolled-back version is observed on the host and on every VM, then `v2.1` is
  moved forward again to the latest release and the forward version is
  observed in the same places. Where the fleet is left afterwards is a separate
  decision.

## 8. Out of scope

- **Moving to a new minor line.** When the plugin moves to `v2.2`, every
  consumer's settings must change from `v2.1` to `v2.2`, as for vergil-tooling
  and vergil-actions consumers. That change is documented, not automated.
- **Multiple channels, version pinning per consumer, a development
  marketplace.** All still ruled out (#45).
- **Historical documents** (completed plans, earlier epic specs and
  retrospectives, changelogs) that mention `ref: main` stay as written.
