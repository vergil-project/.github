# resumable-release-records — retrospective

- **Epic:** vergil-project/.github#1440
- **Spec:** [`spec.md`](spec.md) · **Plan:** [`plan.md`](plan.md)
- **Planning issue:** vergil-project/.github#1437
- **Retrospective task:** vergil-project/.github#1442

## 0. At a glance

**What we set out to do:** make `vrg-release` one workflow that
`vrg-release --resume` always finishes, with its state kept on a locked
tracking issue that only the human can change, and have it post a `Released:`
record on every issue a release contains. Agents would then read those
records as release evidence, rather than asking the human to attest a
release by hand.

**What shipped:** all of it, in vergil-tooling v2.1.334, and in use from
v2.1.335. The tracking issue carries a state block and is locked. An identity
rule stops `vrg-release` when anyone but the human has changed the issue. A
resume runs only the stages not yet done, including failed CD publish jobs,
the package-index wait and the dependency update. A fresh release refuses
while another is open, and `--supersede` replaces a release that cannot
finish, including one whose tracking issue tripped the identity rule.
`--no-promote`, `--no-install` and `--no-finalize` are gone. `install`
follows the release line `vX.Y` and no longer waits on packaging. The
`release-records` stage posts the records, `vrg-release-record check`
verifies them, and `issue-deploy`, `epic-create` and the deployment scaffold
use that check. The first live release under the new tooling
(vergil-tooling v2.1.335), a vergil-claude-plugin release (v2.1.84) and six
fixture releases all posted their records, and every check on them passed.
One planned task, Task 20 (removing the deprecated `consumer-refresh` key),
left the epic. It is vergil-project/vergil-tooling#4860, waiting to become
an epic of its own.

| | |
| --- | --- |
| **Span** | 2026-11-20 19:08Z (epic opened) → 2026-11-22 18:56Z (last task before this one closed), about 48 hours |
| **Tasks** | 55 closed, plus this retrospective. 44 were planned (Task 20 then left the epic), and 13 were added during execution |
| **PRs** | 38 merged, plus this retrospective's |
| **Repos touched** | 10: `.github`, `docs`, `vergil-tooling`, `vergil-claude-plugin` and the six language fixtures |
| **Releases** | 10: vergil-tooling v2.1.332, v2.1.334 and v2.1.335; vergil-claude-plugin v2.1.84; fixtures python-app v0.1.3, python-lib v0.1.4, rust-app v0.1.4, rust-lib v0.1.5, ts-app v0.1.6, ts-lib v0.1.6 |

### Work delivered

vergil-tooling (implementation, in plan order):

| Task | Issue | PR | What it did |
| --- | --- | --- | --- |
| 2 | #4845 | #4863 | `vX.Y` resolves through its rolling tag; `vrg-tooling-install vX.Y` |
| 3a | #4846 | #4876 | Release state kept in a state block on the tracking issue |
| 3b | #4847 | #4879 | Tracking issue locked; only human state changes trusted |
| 4 | #4848 | #4884 | `--resume` runs from the first undone stage |
| 5 | #4849 | #4880 | Refuse a fresh release while any release is open |
| 6 | #4850 | #4881 | List the issues a release closes, from trusted closing references |
| 7a | #4851 | #4891 | Deliver failed CD publish jobs on resume |
| 7b | #4852 | #4890 | Resumable package-index wait and dependency update |
| 8 | #4853 | #4894 | Decouple promote and install from packaging; `consumer-refresh` renamed `install`; `close-tracking` stage |
| 9a | #4854 | #4895 | Remove `vrg-release --no-promote` and `--no-install` |
| 9b | #4855 | #4897 | Remove the submit and finalize opt-outs |
| 10 | #4856 | #4896 | Post a `Released:` record on every issue a release contains |
| 11 | #4857 | #4898 | `vrg-release --supersede` |
| 12 | #4858 | #4933 | `vrg-release-record check` and the deployment scaffold; `filteredCount` fix |
| 14a | #4859 | #4939 | vergil-tooling installs through `[publish].install` and `v<MINOR>` |

vergil-tooling (added during execution):

| Issue | PR | What it did |
| --- | --- | --- |
| #4877 | #4885 | Order open releases by semantic version, not as strings |
| #4878 | #4910 | Supersede a release whose tracking issue fails the identity rule |
| #4882 | #4883 | Restore the `cast` import that merging #4879 and #4880 together dropped |
| #4886 | #4888 | Refuse a resume whose release branch is missing |
| #4889 | #4892 | Let a resume past `merge-release` run without the deleted release branch |
| #4899 | #4909 | Record the target version so a resume never drops the bump |

vergil-claude-plugin:

| Task | Issue | PR | What it did |
| --- | --- | --- | --- |
| 14b | #1232 | #1244 | Install through `[publish].install` |
| 15 | #1233 | #1245 | Skills read release evidence with `vrg-release-record check` |
| added | #1236 | #1237 | `pr-watch` may push a fix whose only stop was a ruling that left the branch as reviewed |

Documentation (the review bookend and its per-repo siblings):

| Issue | PR | What it did |
| --- | --- | --- |
| vergil-tooling#4837 | vergil-tooling#4947 | Bring vergil-tooling's docs in line with resumable releases and records |
| docs#157 | docs#158 | Release and workflow pages in the docs site |
| vergil-claude-plugin#1250 | vergil-claude-plugin#1252 | Plugin skills: always-finalizing submit, the record check |
| vergil-claude-plugin#1251 | vergil-claude-plugin#1253 | Four more lines that still said submit finalizes "by default" |

`.github`: the spec and plan (#1441, PR #1458), and the three plan
evolution records (#1464 → #1465, #1471 → #1472, #1499 → #1500).

Operational tasks (no PR):

| Task | Issue | Outcome |
| --- | --- | --- |
| 2d | .github#1454 | vergil-tooling v2.1.332 carried the resolver fix alone; it was installed before the freeze opened |
| 13 | .github#1455 | vergil-tooling v2.1.334 carried Tasks 3a to 12 and the six follow-ups, which ended the freeze. One stale tracking issue (vergil-containers#953) was closed first |
| 16 | .github#1456 | vergil-claude-plugin v2.1.84, the first release whose records were checked with `vrg-release-record check` |
| 17 | .github#1457 | SUCCESS: the live v2.1.335 release left a locked, closed tracking issue and one human-authored record; install went through `v<MINOR>` |
| 18 | 18 fixture issues | Change, release and validate on all six fixtures, against v2.1.334 |
| 19 | .github#1496 | SUCCESS: all six fixture releases left a locked, closed tracking issue and one record on every issue in the release |

## 1. How the plan evolved

The plan held. Every planned design element shipped as specified, and the
task graph changed in four ways.

**Merge reviews kept surfacing the edges of resume.** Five of the six
vergil-tooling tasks added during execution came out of merge reviews or
task runs, and every one fixed behaviour this epic introduced. Most were
about what a resume finds when it comes back. It could find the release
branch missing (#4886), or deleted on purpose by `delete_branch_on_merge`
(#4889). It could find a version bump that `prepare` never committed
(#4899). Or it could find a tracking issue that can never pass the identity
rule again (#4878). The spec described resume as "run the undone stages".
Execution showed that each undone stage also has preconditions that an
interrupted run may have left half-met. All of these shipped inside the
freeze, in Task 13's release, on the human's rulings. Each extended Task 13's
`Blocked-by` list through a scope change.

**The breach recovery changed the design.** Task 3b's merge review (N0.1)
showed that the spec's recovery, "the human cleans up the issue, then
resumes", cannot work. GitHub keeps the edit history and the close and
reopen events, so a tracking issue that a foreign identity has touched fails
the rule for good. The human ruled `file`, and #4878 built option (b): an
explicit `--supersede` of a breached release never reads its state block.
It rebuilds the release from GitHub and git facts instead. This was the one
real design change of the epic.

**A latent defect sat in merged code until a later task tried it live.**
Task 6's closing-reference check read GitHub's unfiltered
`timelineItems.totalCount`, which counts every timeline item, not just the
filtered ones. Against real PRs it would have refused almost every release
at preflight. The unit tests passed because their fixtures reported a
filtered count. It surfaced only when Task 12 ran `vrg-release-record check`
against a real task. The human ruled `fix` on Task 12's branch and asked for
an audit. The audit found the same mistake in Task 3b's tracking-issue
guard, and the human ruled `keep` on fixing that in Task 12 too.

**Stage moves and one departure.** `close-tracking` moved from Task 10 to
Task 8, so that from Task 8 onward something closes the tracking issue and
any failed stage leaves it open and resumable. Task 20 left the epic
(ruling of 2026-11-22). The plan cut it into "the first minor release after
Task 13's", and minor releases are avoided as costly, so it would have held
the documentation review and this retrospective open indefinitely. It is
vergil-project/vergil-tooling#4860, in the planning queue (#439), and will be
planned as a small epic. That epic's rule is that every `vergil.toml` using
the old key moves to `[publish].install` before the key is removed. This
move came after the plan's third evolution entry, so the plan's log does not
record it. This retrospective is its record.

Also outside the log: the documentation review grew beyond vergil-tooling.
It spawned docs#157 and vergil-claude-plugin#1250, and #1250's merge review
found four more stale lines, which became #1251. #1236 (`pr-watch`) came from
friction in the epic's own workflow. It was worked inside the epic and
shipped in the plugin release.

### Spec corrections

On the human's ruling (2026-11-22, on #1442), the approved spec stays as
written. These corrections, found by the documentation review
(vergil-project/vergil-tooling#4837), are recorded here instead:

- **§4.2, the identity rule:** "the human inspects and cleans up the issue,
  then resumes" does not hold. A breached tracking issue can never pass the
  rule again. The release is superseded with `vrg-release --supersede X.Y.Z`,
  which rebuilds it from GitHub and git facts (#4878).
- **§4.3, the worktree:** "a resume that reaches a branch-work stage always
  finds its branch" does not hold after `merge-release`, because GitHub
  deletes `release/<version>` on merge. A resume past the merge runs without
  the branch, using the existing worktree, a detached worktree at `develop`
  or the root checkout (#4889).
- **§4.7, supersede:** the absorbed issue list is "read from its state block
  under the identity rule" only for an issue that passes the rule. A breached
  issue's list is rebuilt from facts instead (#4878).

## 2. Lessons learned

- **Fixtures that mirror the code's own assumption prove nothing about the
  API.** The `totalCount` defect passed review and tests because the test
  fixtures encoded the same misreading of GitHub's API as the code. The only
  thing that caught it was running against a live GitHub. Next time, for
  every new GraphQL field a task relies on, record one live response as the
  fixture (or check one by hand in the merge review), rather than writing
  the fixture from the docs.
- **Resume is a state machine, and its edges need their own review.** The
  spec specified the stage order well and the re-entry preconditions thinly:
  which branch, which worktree, which version, which issue. Most of the
  added tasks lived there. A future epic that adds resumability should
  enumerate, for each stage, what a resume may find missing and what it does
  then, before planning.
- **Ask whether a recovery path can work, not only what it does.** "Clean
  up, then resume" read well in review across five design rounds. It failed
  the moment someone asked what GitHub actually lets you delete. Recovery
  paths in a spec deserve a "can this physically be done?" check against the
  platform.
- **The freeze worked, and so did shipping the resolver fix first.** No
  half-landed release tooling was released. Task 2d's early release meant
  the `vX.Y` change was already proven when the big release came. The
  freeze's cost was that every follow-up had to land before Task 13, which
  turned each `file` ruling into a scope change on the release. That was the
  right trade here.
- **Parallel merging has a real, known cost.** #4879 and #4880 each passed
  alone and broke `develop` together (#4882). This is the risk epic
  vergil-project/.github#1218 accepted when it dropped the up-to-date rule.
  Finalize's post-merge validate caught it, and the fix landed before any
  other merge, which is the designed recovery.
- **Moving tasks out beats holding the epic open.** Task 20 was tied to a
  release cadence the epic did not control. Moving it to the planning queue
  let the epic close on what it delivered.
- **Fixture runs found agent-access gaps, not tooling bugs.** All six
  fixtures passed on the first run. Their friction was the agent identity
  being unable to read release assets (check 4 needed a human-run step on
  every fixture) and evidence bundles that omit tools which write no report.
  Both are tracked (§4).

## 3. Compromises and tradeoffs

- **Live validation covered the happy path only.** Task 17 and Task 19 ran
  releases where nothing failed. Every recovery path in spec §8 is
  verified by unit tests and code review, not by a live release: the CD
  publish failure and resume, the index miss, `--resume --without-deps`,
  `--supersede` (tagged, untagged and breached), the identity-rule stop, and
  `install` refusing a `vX.Y` not on the release. So is the demotion case for
  `vX.Y`: Task 2d checked the resolver's ancestry and help, and did not demote
  a tag live. The first real failure will be the first live test of these
  paths.
- **The breach-recovery premise is documented, not observed.** #4878's docs
  and breach message state that GitHub keeps the editor of a deleted body
  revision. The evidence is GitHub's documentation for comments, and the
  same behaviour for issue bodies is inferred. A live check was proposed and
  dropped by the human, who accepted the design with the caveat on record.
  One security finding from #4878's review was recorded as an advisory on
  the human's ruling, with no mitigation task.
- **Spec drift is recorded, not repaired.** The spec keeps three statements
  that execution disproved (§1, Spec corrections). A reader must follow
  spec → plan → retrospective to get the true design.
- **The deprecated key lives on.** `[publish].consumer-refresh` still works,
  with a warning, until #4860's epic removes it.
- **Agent-run validation needed the human.** Every fixture validation's
  check 4 (the evidence bundle) used a download that the human ran, because
  the agent identity cannot read release assets. The human ruled
  vergil-project/.github#1432 the real fix, with no stop-gap.
- **A finished tracking issue looks unfinished.** `publish-status` runs after
  `close-tracking` freezes the body, so every complete release shows one
  unticked box (#4946). The state block's `install_done` reads `false` on a
  release with no install configured, by design: it means "install ran
  commands". Readers of the fixture issues took both for faults.

## 4. New problems and opportunities

### Filed or found during execution

| Problem | Where it went |
| --- | --- |
| Removing the deprecated `consumer-refresh` key | vergil-tooling#4860, planning queue (#439), a small epic of its own |
| Install's reported version and commit not kept in the record | vergil-tooling#4945 (from Task 17) |
| `publish-status` left unticked on a finished tracking issue | vergil-tooling#4946 (from Task 17) |
| The `locked` event's actor shows as the org, not an identity | .github#1503 (from Task 17) |
| `vrg-release-record check` compares repo names in two different ways | vergil-tooling#4949 (from the documentation review) |
| The dependency-update branch is left on origin after its PR merges | vergil-tooling#4952 (from the fixture runs) |
| The evidence bundle omits tools that write no report (`ty`, `rustfmt`, `prettier`, the build gate) | vergil-tooling#4953 (from the fixture runs) |
| The agent identity cannot download release assets | .github#1432 (existing intake), ruled the real fix |
| The `app-id` input deprecation in the release job | vergil-actions#1243 (existing, open) |
| `paad-delegate` closing its security section with a second start marker (hit in three planning rounds) | vergil-claude-plugin#1037 (existing, open) |
| Mechanizing release freezes | .github#1453 (named in the plan) |
| Recording the PR workflow on its issues, as releases now are | .github#1444, the follow-on brainstorm, now in the deferred queue (#531) |

### The unaddressed-findings sweep

Swept: all 55 closed tasks, the epic issue and the planning issue, using
their comments (240 of them, all by trusted authors), the `Awaiting a ruling`
and `Filed without a ruling` lists, and the reviews and conversations on the
38 merged PRs.

- **Awaiting-a-ruling items:** every item is closed by a ruling, or by an
  issue filed on one, with these notes:
  - Task 17's three items (epic comment of 2026-11-22 18:08) have no ruling
    line on the epic. They were filed as #4945, #4946 and #1503, and each
    issue's body records the human's ruling.
  - Two items were dropped by the human: the live check of the deleted-revision
    premise (#4878), and Python tools missing from `PATH` in
    `vrg-container-run`.
  - The ts-lib fixture's README API list (vergil-fixture-ts-lib#58, R1) was
    declined by ruling.
- **Planning rounds:** every finding not accepted has its reason, except
  spec round 1's single rejection, which the round comment gives only as a
  count. The human confirmed that it was the security finding not accepted
  that round (GHSA-rvpf-68pw-9w74, closed).
- **Documentation-review items:** each one handed to #4837 is fixed, filed
  (#4949), or gone from the code (`comment_publish_deferred` no longer exists
  on `develop`).
- **PR reviews:** no human review comments. One code-scanning review in state
  `COMMENTED` on vergil-tooling#4881, with an empty body, was passed to the
  human to check its inline comments. CodeQL passed on that PR.
- **Untracked, ruled at this retrospective (2026-11-22):**
  - vergil-tooling#4590 (rename `consumer-refresh` and drop its warning) was
    still open, although #4853 did both. **Found:** commented and closed as
    completed.
  - `install_done: false` on a release with no install configured.
    **Dropped** by the human: it is by design (§3).
  - ts-lib's npm-audit exception is reported as unmatched. **Found:**
    vergil-fixture-ts-lib#20 tracks the entry, and a comment there records
    the observation.
- **Security:** 1 candidate, ruled `fix` in the terminal. Counts only here.

## 5. What's next

- **vergil-tooling#4860:** plan the small epic that audits every
  `vergil.toml` (in this fleet and in other orgs) and then removes the key in
  a minor release.
- **.github#1444:** the follow-on brainstorm, recording the PR workflow on
  its issues. Now that the release records have run on real releases, it can
  start once it is revived from the deferred queue.
- **The first failing release:** watch the first release that hits a
  recovery path (§3). It is the first live exercise of resume and supersede.
- **The filed follow-ups:** #4945, #4946, #4949, #4952, #4953 (vergil-tooling)
  and #1503 (.github) sit in the ad-hoc and planning queues.

## Appendix A — Operational notes

1. **Resolver first (Task 2d).** v2.1.332 carried Task 2 alone. It was
   installed on the host and every VM, and checked by ancestry and
   `vrg-tooling-install --help`, before Task 3a merged.
2. **The freeze.** It opened when Task 3a merged (PR #4876, 2026-11-21 12:16Z)
   and ended with Task 13's v2.1.334 cut. v2.1.333 (12:01Z) was cut before
   3a merged, and does not contain it. The tags go straight from v2.1.333 to
   v2.1.334.
3. **Clearing the fleet before the switch.** The new tooling refuses to
   resume an old-style release, so before the cut the open `release:` issues
   across the org were listed. One was found, vergil-containers#953 (2.1.45),
   whose deferred publish had already been delivered. It was closed on the
   human's instruction.
4. **The old tooling cuts the new.** v2.1.332 and v2.1.334 were cut by older
   tooling and posted no records. v2.1.335 (Task 14a) was the first
   vergil-tooling release cut by the new tooling, and the first with records.
5. **Config only after deploy.** Tasks 14a and 14b moved to
   `[publish].install` only after v2.1.334 was installed, since the release
   is run by the installed tooling.
6. **Plugin.** v2.1.84 carried Tasks 14b and 15 and #1236. It was checked
   with `vrg-release-record check` on all three.
7. **Fixtures.** All six ran change → release → validate against v2.1.334,
   with the epic taking fixture ownership on the human's ruling. Each
   validation's check 4 used a download the human ran.

## Appendix B — Extended metrics

- **Merge review:** 37 tasks went through it, with 41 closing runs (a few
  tasks were reviewed again after a rebase). Of the 43 review steps, 30
  passed, 9 asked for a fix (11 defects and 1 security finding, all fixed on
  their branches) and 4 stopped for a human ruling. No run ended with
  accepted risk.
- **Design review (planning):** five rounds (spec pushback × 2, plan
  pushback × 2, alignment × 1). Their round comments give 57 findings raised
  and 54 accepted. They also give security counts of 10 raised and 9
  accepted, which became advisories (GHSA-9rj7-jmfj-cjqq,
  GHSA-r67r-gh5h-7j92, GHSA-phxw-fpvx-rm39, GHSA-826g-5hcc-9p76,
  GHSA-m5jx-7295-476w, GHSA-j675-33vm-gh52, GHSA-4g9g-g7vg-c62v,
  GHSA-32p4-93jj-q8w5, GHSA-7r62-g8v4-h3wx) and were handed on to their fix
  tasks. The comments do not say whether the security counts are included
  in the totals.
- **Code volume (vergil-tooling PRs):** about 14,200 lines added and 3,800
  removed across 22 PRs. Task 8 (#4894) was the largest, at +1,565/−1,779.
- **Planned vs delivered:** 44 planned task issues (counting the 18 fixture
  issues and both bookends), 1 removed (Task 20), 13 added (6 tooling
  follow-ups, 1 plugin task, 3 plan records, 3 documentation siblings).
