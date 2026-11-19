# Retire the develop-vX.Y.Z tags — plan

- **Epic:** vergil-project/.github#1406
- **Spec:** [spec.md](spec.md)
- **Planning issue:** vergil-project/.github#923 (review history is in its
  comments)
- **Status:** approved

## Delivery

| # | Task | Kind | Repo | Blocked by |
| --- | --- | --- | --- | --- |
| 0 | vergil-project/.github#1407: spec and plan | docs | .github | — |
| 1 | vergil-project/vergil-tooling#4770: `vrg-release` stops requiring the tag | task | vergil-tooling | — |
| 2 | vergil-project/vergil-tooling#4771: release vergil-tooling, install on host and VMs | deployment | vergil-tooling | 1 |
| 3 | vergil-project/vergil-actions#1337: tag-and-release stops creating the tag; delete the cliff configs | task | vergil-actions | 2 |
| 4 | vergil-project/vergil-actions#1338: release vergil-actions, promote `v2.1` | deployment | vergil-actions | 3 |
| 5 | vergil-project/.github#1409: delete the existing `develop-v*` tags across the fleet | deployment | .github | 4 |
| 6 | vergil-project/vergil-claude-plugin#1199: delete the unused cliff configs | task | vergil-claude-plugin | — |
| 7 | vergil-project/vergil-containers#972: delete the unused cliff configs | task | vergil-containers | — |
| 8 | vergil-project/vergil-tooling#4769: documentation review | task | vergil-tooling | 1, 3, 5, 6, 7 |
| 9 | vergil-project/.github#1408: retrospective | retrospective | .github | every other task |

## Tasks

### 1. vergil-tooling: stop requiring the tag

Remove the `develop-v` check and the `develop_tag` field from
`lib/release/confirm.py` and `lib/release/context.py`, and the summary line
from `lib/release/finalize.py`. In `lib/changelog.py`, pass
`--tag v<version>` and keep `--ignore-tags develop-v*`. In the bundled
`cliff.toml` and `cliff-release-notes.toml`, strip `v` instead of `develop-v`
and drop `develop-v` from `tag_pattern`. Check that a sample changelog
heading and release-notes title render exactly as before. Update the tests
and `docs/site` (`guides/releasing.md`, `reference/cli-tools-overview.md`).

### 2. Deploy vergil-tooling

The human cuts and promotes the release and updates the host and the VMs.
The agent confirms the version and that it has no `develop-v` check. This
gates step 3: see spec §3.

### 3. vergil-actions: stop creating the tag

Remove the "Tag develop for changelog boundaries" step from
`tag-and-release/action.yml`, along with its mention in the action's
description and in `push-tag.sh`'s header. Update the harness cases and the
two docs pages, and delete `cliff.toml` and `cliff-release-notes.toml`.

### 4. Deploy vergil-actions

The human cuts the release and promotes `v2.1`. The agent confirms that
`v2.1` has no develop-tag step, and that the next release of any repo
completed without creating a `develop-v` tag.

### 5. Delete the existing tags

The agent lists the `develop-v*` tags per repo in every org the App is
installed on, notes any with a GitHub Release attached, and writes the
delete commands. The human runs them with Admin rights. The agent then
confirms that none remain.

### 6 and 7. Delete the leftover cliff configs

In vergil-claude-plugin and vergil-containers, delete `cliff.toml` and
`cliff-release-notes.toml` after checking that nothing reads them.

### 8. Documentation review

Sweep the current docs for `develop-v*` as part of the release flow,
starting with vergil-tooling's and vergil-actions' docs sites and the `docs`
repo. File a same-repo doc task for any repo other than vergil-tooling that
needs a change.

### 9. Retrospective

Authored with `epic-retrospective`; its merge closes the epic.
