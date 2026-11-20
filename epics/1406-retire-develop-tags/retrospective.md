# Retire the develop-vX.Y.Z tags — retrospective

- **Epic:** vergil-project/.github#1406
- **Planning issue:** vergil-project/.github#923
- **Spec:** [`spec.md`](spec.md) · **Plan:** [`plan.md`](plan.md)
- **Retrospective task:** vergil-project/.github#1408
- **Span:** epic opened 2026-11-19T18:00Z. Execution began when the spec
  and plan merged, 2026-11-19T18:12Z. The last task before this
  retrospective closed 2026-11-20T19:20Z: about 25.3 hours from opening.
  All delivery work ran on 2026-11-20, from 12:38Z to 19:20Z.

## 0. At a glance

**What we set out to do.** Stop creating the `develop-vX.Y.Z` tag that
every release put on the `develop` tip, and delete the ones already there.
Nothing used the tag any more: vergil-project/vergil-tooling#3805 had
moved the changelog boundary to the merge base with the previous `vX.Y.Z`
tag. Only a loop kept it alive: vergil-actions' `tag-and-release` created
it, and `vrg-release`'s confirm stage checked that it existed. The order
mattered: `vrg-release` had to stop requiring the tag, and be installed
everywhere, before any release stopped creating it.

**What shipped.** All of it, in the planned order:

- **vergil-tooling:** `vrg-release` no longer requires or reports the
  tag. git-cliff labels a release `v<version>`, and the bundled cliff
  configs match only full `vX.Y.Z` tags. Released as v2.1.331.
- **vergil-actions:** `tag-and-release` creates only the `v<version>` tag
  and the GitHub Release. Released as v2.1.88, which moved `v2.1`.
- **Unused cliff configs** deleted from vergil-actions,
  vergil-claude-plugin and vergil-containers.
- **The existing tags:** 928 deleted across 19 repos in three orgs. 939
  existed; the 11 left are in two permanently archived repos, ruled out of
  scope.
- **The live check:** vergil-claude-plugin v2.1.82, the first release on
  the promoted `v2.1`, pushed only `v2.1.82` and created no `develop-v`
  tag.

**Work delivered** (PR — what it did — merged, UTC):

- **Task 0, .github #1416:** spec and plan — 2026-11-19 18:12.
- **Task 1, vergil-tooling #4826:** `vrg-release` stops requiring the
  tag, and the cliff configs label by `v<version>` — 2026-11-20 12:44.
- **Task 3, vergil-actions #1345:** `tag-and-release` stops creating the
  tag, and its cliff configs are deleted — 2026-11-20 14:04.
- **Task 6, vergil-claude-plugin #1215:** unused cliff configs deleted —
  2026-11-20 12:38.
- **Task 7, vergil-containers #979:** unused cliff configs deleted, with
  their entry in the image-selection path list — 2026-11-20 12:38.
- **Task 8, vergil-tooling #4838:** the releasing guide says the tags are
  retired and deleted — 2026-11-20 19:20.

The three deployment tasks had no PR: vergil-tooling#4771 (release and
install v2.1.331), vergil-actions#1338 (release v2.1.88 and promote
`v2.1`) and .github#1409 (delete the tags).

- **Repos changed:** 5 (.github, vergil-tooling, vergil-actions,
  vergil-claude-plugin, vergil-containers). Tags were deleted in 19 repos
  across vergil-project, mq-rest-admin-project and diogenes-project.
- **Tasks:** 10, including the two bookends. **PRs:** 6, plus this
  retrospective's.
- **Releases cut for the epic:** vergil-tooling v2.1.331 and
  vergil-actions v2.1.88.

## 1. How the plan evolved

The plan held. Every task ran in the planned order, and none was added,
dropped or split. No `Evolution during execution` entry was written,
though two decisions during execution belong in one:

- **Archived repos are out of scope.** The tag-deletion task asked for no
  `develop-v*` tag in any repo. Two repos, mq-rest-admin-java and
  mq-rest-admin-ruby, are archived, and an archived repo's tags cannot be
  deleted. The human ruled them out of scope: they are archived for good,
  so their 11 tags stay.
- **The live check came from another epic's release.** Task 4 needed a
  repo to release on the promoted `v2.1`. vergil-actions' own v2.1.88
  release created no tag, but it ran the repo's local copy of the action,
  so it did not count. Rather than cut a release for the check, the
  epic waited a few hours for vergil-claude-plugin v2.1.82, released for
  another epic.

Two smaller changes stayed inside their tasks. The vergil-containers task
also removed the `cliff*.toml` entry from the image-selection path list
and its test; the image selection is unchanged. The vergil-tooling task
tightened the bundled `tag_pattern` to full `vX.Y.Z` tags, so rolling
minor tags are left out; a sample history rendered the changelog and
release notes byte-identically under the old and new configs.

## 2. Lessons learned

- **Ordering a breaking pair across repos works.** The consumer
  (`vrg-release`) stopped checking before the producer (`tag-and-release`)
  stopped creating, with a deploy between them. No release failed at any
  point.
- **A vergil-tooling release includes its install.** The human's position,
  recorded on vergil-tooling#4771: a release without the host and VM
  install is not a complete release. The deploy task's scaffold instead
  made "update the host and VMs" a separate human step, and the run asked
  for version output as proof. That cost friction for no information.
- **A session in a VM cannot run host-only or Admin-only steps.** Both
  deploys that needed the human (the host install check and the Admin tag
  deletes) had to go to a macOS host terminal. `issue-deploy`'s
  `! <command>` hand-off assumes the session runs where the human's
  identity and tools are.
- **A release does not test its own action change.** vergil-actions
  releases with its local action copies, so its own release can never be
  the live check of a change to them. Plan the live check as "the next
  release of a consumer repo".
- **Same next time:** an epic this size, M with about 10 tasks, went from
  spec to the retrospective in about a day, with no rework.

## 3. Compromises & tradeoffs

- **Host and VM installs rest on attestation.** vergil-tooling#4771
  recorded the host and VM installs from the human's word that the release
  always includes them. Only the Lima VM's version was checked directly.
- **Eleven tags remain** in the two archived mq-rest-admin repos, by
  ruling. Clearing them would mean unarchiving repos that are not coming
  back.
- **A short window of incomplete docs.** From #4826's merge (12:44Z) until
  `v2.1` moved (14:17Z), the releasing guide no longer said that CD
  creates the tag, while CD still did. Accepted as too short to matter,
  and the documentation review confirmed the final wording.
- **`--ignore-tags develop-v*` is kept on purpose.** git-cliff still
  ignores any stray `develop-v*` tag, such as the 11 in the archived
  repos, rather than trusting that none exist.

## 4. New problems & opportunities

**Unaddressed-findings sweep.** Swept the 9 children other than this
retrospective, the epic issue and the planning issue. The planning issue
had no design-review rounds (skipped at the human's request), and none of
the epic's PRs had a review. This is not a security epic. The epic's merge
reviews raised no findings.

Five untracked problems came up, all resolved:

1. **The deploy scaffold names `vrg-host` for the host update**, which
   never upgrades vergil-tooling. From the vergil-tooling#4771 run, listed
   on the epic. Declined by the human (not filed); the human may raise it
   separately.
2. **The deploy scaffold asks for a redundant host and VM update.** Same
   source and outcome.
3. **`! <command>` hand-offs cannot reach host-only or Admin-only steps
   from a VM session.** From the vergil-tooling#4771 and .github#1409
   runs. Same outcome.
4. **.github#1409's listing command fails for private repos**
   (`ls-remote` on a bare https URL). Dropped by the human: a one-off task
   body, worked around with local clones.
5. **vergil-actions#1338's scaffold calls issue #1337 the step-3 PR** (the
   PR was #1345). Dropped by the human: wording in a closed task.

## 5. What's next

Nothing follows from this epic. The three deploy-workflow follow-ups in
§4 are with the human, who may raise them separately.

## Appendix A — Operational notes

1. vergil-tooling #4826, vergil-claude-plugin #1215 and vergil-containers
   #979 merged (12:38–12:44Z).
2. vergil-tooling v2.1.331 published at 13:43:46Z, with its install on
   the host and VMs. `confirm.py` at the tag has no `develop-v` check
   (v2.1.330 still had it).
3. vergil-actions #1345 merged at 14:04Z, and v2.1.88 was published at
   14:17:08Z, moving `v2.1` to `cc40c244`.
4. vergil-claude-plugin v2.1.82 released at 18:36Z on `v2.1` (CD run
   44794139537). It pushed only `v2.1.82`, and no `develop-v2.1.82` tag
   exists.
5. The human ran the generated delete script from a macOS host terminal
   with Admin rights. It pushes deletes 100 tags at a time per repo, then
   re-lists each repo and exits non-zero if any tag is left. An agent
   re-listing then confirmed 0 tags in every non-archived repo. Before:
   vergil-project 882 (vergil-tooling 451, vergil-actions 156,
   vergil-claude-plugin 124, …), mq-rest-admin-project 56,
   diogenes-project 1. None had a GitHub Release attached.
6. vergil-tooling #4838 merged at 19:20Z, closing the documentation
   review.
