# Plugin distribution via the floating `v2.1` tag

- **Epic:** vergil-project/.github#760
- **Planning issue:** vergil-project/.github#759 (review history is in its
  comments)
- **Status:** draft
- **Repos:** vergil-claude-plugin (its distribution docs), vergil-tooling (the
  consumer setup guide and a rollback runbook), `.github` (this spec and plan)
- **Supersedes:** the `main`-only decision of epic vergil-project/.github#45
  ([spec](../45-plugin-distribution-model/spec.md))

## 1. Problem

Every consumer registers the `vergil-marketplace` with `"ref": "main"` in the
user-level `~/.claude/settings.json`. The deployed plugin is therefore whatever
`main` holds, and the only way to put the fleet back on an earlier release is a
PR that changes `main`. The integration of the paad skills brought the first
release that had to be reverted, and that PR is the wrong tool for it.

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
  `v2.1`. `develop` is still not consumed through a marketplace (epic #45 §2
  stands).
- `main` stays the release branch. The tag, not the branch, is what consumers
  follow, so what is deployed can differ from what `main` holds.
- **Deploying a release** is moving `v2.1` forward to `v2.1.Z`, which the release
  already does (§4).
- **Rolling back** is moving `v2.1` back to an earlier `v2.1.Z` with
  `vrg-promote` (§5). It is a human act, gated like any release.

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
- **VMs.** VM provisioning copies the host `~/.claude/settings.json` into the
  guest (`vrg-vm` `_st_copy_config`). Its plugin refresh registers each declared
  marketplace with its `ref` (`vm_guest._marketplace_sources`, `owner/repo@ref`).
  No code change is needed.

## 5. Rollback procedure

1. A human runs `vrg-promote <X.Y.Z>` against `vergil-claude-plugin`, naming the
   release to return to. This is a release act and stays human-gated.
2. Each machine picks the move up the next time its marketplace and plugin
   update: `claude plugin marketplace update vergil-marketplace`, then `claude
   plugin update vergil@vergil-marketplace` (background auto-update does the
   same where it is on), then a restart of Claude Code. VMs pick it up with
   `vrg-vm update`.
3. The rollback lasts until the next release, which moves `v2.1` forward again.
   The fix goes forward as an ordinary release; there is no hold flag.

The procedure is documented in vergil-tooling's site docs next to the release
guide, and the plugin README points to it.

## 6. Switching existing machines

Changing the `ref` string of a marketplace that is already registered is a
source change, not an update. In the isolated test, editing only the `ref` in
`settings.json` left `known_marketplaces.json` on the old source, and `claude
plugin marketplace update vergil-marketplace` reported `Marketplace
'vergil-marketplace' not found`. The switch therefore needs a procedure that
re-registers the marketplace at the new ref without uninstalling the plugin.
The deployment task proves that procedure in an isolated `CLAUDE_CONFIG_DIR`
first, then applies it to the host and to every VM, and the proven steps go
into the consumer setup guide. The procedure must not use `claude plugin
marketplace remove`, which also uninstalls the marketplace's plugins and
deletes their data.

## 7. Changes

- **vergil-claude-plugin:** `README.md` (the "Distribution model" and "Install"
  sections), `CLAUDE.md` (the distribution paragraph) and
  `docs/site/docs/index.md` (the settings example) describe `v2.1` and the
  rollback, and link here.
- **vergil-tooling:** `docs/site/docs/guides/consuming-repo-setup.md` (the
  marketplace definition and the switch procedure from §6) and a rollback
  section in the release guide (§5). Test fixtures that use `"ref": "main"` are
  sample data for a ref-agnostic function and are left alone.
- **`.github`:** this spec and plan. Epic #45's spec gets one line under its
  header pointing to this spec as superseding its tag decision. Its text is not
  otherwise rewritten.
- **Deployment:** the host `~/.claude/settings.json` and every VM move to
  `v2.1` (§6).
- **Validation:** the real rollback of the fleet to 2.1.50 via §5, observed on
  the host and on at least one VM. It is the proof the mechanism works and the
  rollback that motivated the epic.

## 8. Out of scope

- **Moving to a new minor line.** When the plugin moves to `v2.2`, every
  consumer's settings must change from `v2.1` to `v2.2`, as for vergil-tooling
  and vergil-actions consumers. That change is documented, not automated.
- **Multiple channels, version pinning per consumer, a development
  marketplace.** All still ruled out (#45).
- **Historical documents** (completed plans, earlier epic specs and
  retrospectives, changelogs) that mention `ref: main` stay as written.
