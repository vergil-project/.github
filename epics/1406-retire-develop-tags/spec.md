# Retire the develop-vX.Y.Z tags — design

- **Epic:** vergil-project/.github#1406
- **Planning issue:** vergil-project/.github#923 (review history is in its
  comments)
- **Status:** approved

## 1. Intent

Every release creates a `develop-vX.Y.Z` tag on the tip of `develop`. It was
the changelog boundary until vergil-project/vergil-tooling#3805 moved that
boundary to the merge base of `HEAD` and the previous `vX.Y.Z` tag. Since
then, nothing uses the tag. This epic stops creating it, removes the code that
checks for it, and deletes the tags that already exist.

This is a deliberately small epic. It wraps what would otherwise be
independent ad-hoc tasks so that their order is coordinated, and it ran
without design review rounds.

## 2. What still touches the tag

Every reference on `develop`, as checked on 2026-11-19:

| Where | What it does |
| --- | --- |
| vergil-actions `actions/cd/release/tag-and-release/action.yml` | Creates `develop-v<version>` on `origin/develop` after a release and pushes it with the App token. |
| vergil-tooling `lib/release/confirm.py` | Fails `vrg-release` when the tag does not exist. |
| vergil-tooling `lib/release/finalize.py` | Prints the tag in the release summary. |
| vergil-tooling `lib/changelog.py` | Passes `--tag develop-v<version>` to git-cliff only as the release's label, and ignores `develop-v*` tags. |
| vergil-tooling bundled `cliff.toml`, `cliff-release-notes.toml` | Strip `develop-v` from the label; `tag_pattern` names `develop-v`. |
| `cliff.toml`, `cliff-release-notes.toml` in vergil-actions, vergil-claude-plugin, vergil-containers | Leftovers. `vrg-release` reads only the bundled configs, and nothing in those repos reads them. |

No other repo in vergil-project references the tag outside historical records
(changelogs, release notes and dated specs and plans), which stay unchanged.

## 3. Design

1. **vergil-tooling stops depending on the tag.** Remove the confirm check
   and the summary line. git-cliff labels the release `v<version>`, and the
   bundled templates strip `v`, so the rendered changelog and release notes
   are unchanged. Keep `--ignore-tags develop-v*` until the old tags are
   gone, so that their presence never shifts a range.
2. **Release it and install it everywhere `vrg-release` runs**: the host and
   every agent VM. This must come before step 3. Every org's releases use
   vergil-actions `@v2.1`, so once it stops creating the tag, an older
   `vrg-release` fails at its confirm stage.
3. **vergil-actions stops creating the tag.** Remove the step from
   `tag-and-release`, update its docs and tests, and delete its unused cliff
   configs.
4. **Release vergil-actions and promote `v2.1`.** From then on, no release
   creates the tag. The first release afterwards is the live check.
5. **Delete the existing tags** in every org that releases through
   cd-release, as a matter of hygiene. The tag-protection rulesets allow only
   Admin or the automation App to delete a release tag, so the agent lists
   the tags and writes the delete commands, and the human runs them.

Independent of that order, vergil-claude-plugin and vergil-containers delete
their unused cliff configs.

## 4. Out of scope

- Any other change to the release flow.
- Fixture runs. The change removes a tag that no fixture reads, and the
  first release after step 4 exercises it live.
- Dropping `--ignore-tags develop-v*` from `changelog.py` after the cleanup.
  It is harmless once no such tags remain; a later change may remove it.
