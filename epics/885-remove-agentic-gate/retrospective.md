# Retrospective: install paad from the fork and remove the agentic-review gate

**Epic:** [vergil-project/.github#885](https://github.com/vergil-project/.github/issues/885)
**Planning issue:** [vergil-project/.github#880](https://github.com/vergil-project/.github/issues/880)
**Spec:** [spec.md](spec.md) · **Plan:** [plan.md](plan.md)

## 0. At a glance

We set out to get `issue-implement` working on cloud VMs again, and to take the agentic-review gate out of it. That meant two things:

- **Install paad from the fork.** Every box installs paad from the vergil fork (`wphillipmoore/paad`, ref `develop`) through the ordinary marketplace refresh. The Lima-only `--plugin-dir` override that kept the delegate off cloud VMs goes away.
- **Remove the gate.** `issue-implement` goes back to implement, validate to green, `report-ready`.

All of it shipped. Every box (the Mac host, Lima VMs and cloud VMs) now runs paad from the fork's `develop`, keyed by commit with no version to bump. The override and the fleet checkout are retired. `issue-implement` ran end to end on a cloud VM with no gate. `epic-create`'s mode 1 design review is unchanged.

| PR | Repo | Task | What it did | Merged | Release |
|---|---|---|---|---|---|
| [#896](https://github.com/vergil-project/.github/pull/896) | .github | #886 | Spec and plan | 2026-11-03 | n/a |
| [#920](https://github.com/vergil-project/vergil-claude-plugin/pull/920) | vergil-claude-plugin | #917 | Removed the gate from `issue-implement`, deleted doctrine §3 after moving its kept rules into §1.7, and renamed the private state kind to `issue-implement/<N>/` | 2026-11-04 | 2.1.58 |
| [#3928](https://github.com/vergil-project/vergil-tooling/pull/3928) | vergil-tooling | #3921 | Removed the gate from vergil-tooling's docs and comments | 2026-11-04 | 2.1.285 |
| [#924](https://github.com/vergil-project/vergil-claude-plugin/pull/924) | vergil-claude-plugin | #918 | paad-policy §10 rewritten as "paad from the fork" (install source, rebase process, gate and push rule, accepted risks); doctrine §1.8 | 2026-11-05 | 2.1.59 |
| [#3929](https://github.com/vergil-project/vergil-tooling/pull/3929) | vergil-tooling | #3920 | Removed the `vrg-vm session` `--plugin-dir` override, after pinning that a user's own `--plugin-dir` still passes through | 2026-11-05 | 2.1.285 |
| [#83](https://github.com/vergil-project/docs/pull/83) | docs | #82 | Updated the management pages for the fork install and the gate removal | 2026-11-05 | n/a |
| [#3936](https://github.com/vergil-project/vergil-tooling/pull/3936) | vergil-tooling | #3935 | Re-pointed the renamed paad-policy §10 anchor | 2026-11-05 | not yet released |

**Fork pushes** (by the human; not PRs): `develop` at `6c034c9` (D1), then at `9c77356` (V1's empty commit). The `unattended-mode` branch was deleted from the fork.

**Operational tasks:** all five closed with `Outcome: SUCCESS`.

| Task | Issue | What it did |
|---|---|---|
| D1 | #893 | Installed paad from the fork |
| V1 | #894 | Checked the fork install on a cloud VM |
| D3 | vergil-claude-plugin#919 | Installed the plugin release without the gate |
| D2 | vergil-tooling#3922 | Installed tooling without the override and retired the fleet checkout |
| V2 | #895 | Ran `issue-implement` end to end on a cloud VM |

**Counts:** 14 tasks under the epic.

- 7 closed by a merged PR. That includes the two doc tasks the documentation review spawned (vergil-tooling#3935 and docs#82).
- 5 operational.
- The documentation review (vergil-claude-plugin#916), closed with no PR because its own repo needed no change.
- This retrospective.

**Repos touched:**

- vergil-project/.github, vergil-claude-plugin, vergil-tooling and docs;
- the fork `wphillipmoore/paad`;
- outside the epic, logical-minds-foundry/mq-resiliency-lab-for-linux#1706, which was filed as V2's target.

**Releases cut:** vergil-claude-plugin 2.1.58 and 2.1.59, and vergil-tooling 2.1.285.

**Span:** the planning issue opened 2026-11-03 13:00 UTC. The epic was created at 13:53. The last task closed 2026-11-05 13:03, about two days later.

## 1. How the plan evolved

The plan has no evolution log. This account comes from the execution record: the task outcome comments, the PRs and the epic-implement session.

**The order held.** Task 5 (gate removal) and Task 6 (tooling gate mentions) ran in parallel at once. D1 (install from the fork) went next, then V1 (check it on a cloud VM). V1 unblocked Task 3 (override removal) and Task 4 (policy and doctrine), which ran in parallel. Then came D2 (install tooling, retire the fleet checkout), D3 (install the plugin release) and V2 (`issue-implement` on a cloud VM). All seven code tasks went green and reached `report-ready` on their first implementation, and none needed rework after review.

**D1 was easier than planned.**

- *Fewer conflicts.* The plan expected rebase conflicts in six places. Only `CHANGELOG.md` and `alignment/SKILL.md` conflicted. `agentic-review` merged cleanly and was checked by hand.
- *The agent edited the files.* The plan had the agent run only read-only commands in the fork clone and hand the human every change. During the run the human asked the agent to edit the working-tree files (conflict resolutions and the version-delta commit) while the human ran every git command. The rule that matters held: no agent changed the fork's git state or its remote.
- *A smaller gate.* By the human's ruling, the validation gate ran #843's six unattended runs plus one interactive run each of pushback, alignment and agentic-review, not the full #733 set.
- *A brief fixed mid-run.* The gate brief wrongly asked a VM session to read the host-only `plugin-dirs.toml`. It was corrected to a human attestation, and the gate passed.

**D3 had in effect deployed already.** Plugin 2.1.58, which carries #917, was published before D1's refresh. So D1's `vrg-host plugins` and `vrg-vm update --all` had already installed it everywhere. D3 checked it on every box and recorded it.

**V1 needed no re-add.** The host's `vrg-vm update --all` switched every box's marketplace to the fork without needing `claude plugin marketplace remove`. A commit pushed to `develop` installed on the next refresh, keyed by its SHA.

**V2 changed target twice.**

- *The plan's candidate was gone.* The plan named vergil-tooling#3921, which had already merged.
- *The cloud VMs are per-repo.* They are set up for `mq-resiliency-lab-for-linux`, not vergil-tooling.
- *The first try stopped.* It ran on vergil-tooling#3871 and stopped at Preflight, correctly, because the issue was still in intake. Once routed, it found #3871 wasn't a bug (see §4).
- *A real defect served instead.* A real one-file defect in the lab repo was filed as #1706, and it ran end to end on the cloud VM.

**The documentation review found nothing to change in its own repo.** It spawned vergil-tooling#3935 (a stale anchor) and docs#82 (two management pages), and closed with a summary instead of a PR.

**Two outside glitches.**

- *The tooling release.* Its first attempt failed on a transient package-download timeout in the rhel-10 install test. The failed jobs were re-run and it passed.
- *An auto-close that never happened.* GitHub never auto-closed vergil-tooling#3921 after PR #3928 merged with a valid `Closes #3921`. It was closed by hand.

## 2. Lessons learned

- **Front-loaded review paid for itself.** Four design-review rounds raised 35 findings, and all were accepted. The one that mattered most was plan pushback F1: move doctrine §3's kept rules into §1.7 before deleting §3. Without it, #917 would have left §1.7 citing rules that no longer existed. Every code task landed green first time.
- **Deleting a gate is cheap when it lives in skills and docs.** The removal took two PRs (#920 and #3928) and no code paths. The expensive part was the install source, not the gate.
- **Briefs for operational work must say where each file lives.** The Mac host, Lima VMs and cloud VMs see different filesystems. The D1 gate brief's host-file precondition stopped a VM session for no reason.
- **Cloud VMs are per-repo.** A validation task that says "on a cloud VM" has to name a task in a repo that VM is set up for.
- **Don't read exact values off the tool display.** The agent's tool display capitalizes the first character of a command result, so `vrg-whoami --mode` can look like `User`. Three runs tripped on it. Comparing in the shell fixes it (#900).
- **Parallel sub-agents across repos work well, but they can't coordinate after the freeze.** A note about the renamed anchor reached the #3920 agent after it had reported ready. Leaving it to the documentation-review sweep was the right fallback.
- **GitHub's closing-keyword auto-close is not reliable.** Check for it at finalize (#901).

## 3. Compromises and tradeoffs

- **No review on any of this epic's code.** By standing order, no task ran the agentic-review gate. The replacement reviewer (vergil-project/.github#878) is out of scope. Until it lands, `issue-implement` has no automated review: correctness rests on validation and on the human's PR review.
- **A reduced gate for D1.** Six unattended runs and three interactive runs instead of the full #733 set. The rebase changed skill text beside the unattended paragraphs, but no delegate contract.
- **Two format quirks accepted.** In the gate runs: multi-line `Claim:` fields, and extra prose on the `Agents dispatched:` line. No listed check covers either, and the rebase did not touch the result format.
- **The fork's release machinery no longer runs.** With no `version` in `preview/paad`'s manifest, `make bump-version`, `bump-tree` and `promote.py` no longer work on the fork. That is by design: the fork never releases, it rebases onto upstream tags.
- **Three accepted risks** are recorded in paad-policy §10:
  - every push to `develop` reaches every box;
  - the development clone is mounted read-write on every Lima VM;
  - nothing technical stops a push to the fork.

  The three security advisories that led to them are still drafts, their fix tasks closed.
- **A window with a broken link.** The old §10 anchor in vergil-tooling broke when #918 merged (2026-11-05 00:47 UTC) and stayed broken until #3935 merged (13:03 the same day).
- **Old skill caches.** Some sessions still ran plugin 2.1.56, which has the gate, while the epic's tasks ran. They followed the standing order.

## 4. New problems and opportunities

| Item | Where it went |
|---|---|
| Skill preflights read `vrg-whoami --mode` off the tool display, which capitalizes the first character | [.github#900](https://github.com/vergil-project/.github/issues/900) (intake): compare in the shell |
| GitHub skipped the auto-close for vergil-tooling#3921 | [.github#901](https://github.com/vergil-project/.github/issues/901) (intake): `vrg-finalize-pr` closes any closing reference GitHub left open |
| vergil-tooling#3871 ("`vrg-whoami` prints `User`") | Closed as not planned: the tool writes lowercase and its tests pin it; the display is the cause |
| A broken spec link in the lab repo | logical-minds-foundry/mq-resiliency-lab-for-linux#1706, filed and fixed as V2's target |
| The three security advisories (GHSA-j6m6-g8jx-6mhh, GHSA-v567-75gm-f5gm, GHSA-jxf8-64jh-m5jf) | Still drafts; their fix tasks (#893, vergil-claude-plugin#918) are closed. Closing them is the human's call |
| Private gate material from D1 | Kept on the host under `.github`'s `build/.vergil-state/{design-review,agentic-review}/893-gate/` for the human |
| Empty `agentic-review/` state directories left in the docs, vergil-claude-plugin, vergil-tooling, vergil-vm and `.github` clones | Logged, not acted on: the human may delete them |

**The unaddressed-findings sweep.**

- *Tasks swept:* 13, every task but this one.
- *No gate records at all,* none expected:
  - the code tasks (#917, #918, #3920, #3921, #3935, docs#82, #886), because the epic's standing order skipped the gate;
  - the operational tasks (#893, #894, #895, vergil-claude-plugin#919, vergil-tooling#3922), which have no PR;
  - #916, which closed with no PR.
- *Ruling lists:* every `Awaiting a ruling` list is closed. #893 listed one security item, which the human rejected. Its counts line was posted during this sweep.
- *Filed without a ruling:* none.
- *PR reviews:* none of the seven merged PRs has a review or a conversation comment.
- *The planning issue:* all 35 findings across its four rounds were accepted, and its discards each carry a reason.
- *Security items:* 3 recorded as advisories during design review, and 1 rejected in operations.
- *Untracked problems:* none found.

## 5. What's next

- **The replacement reviewer:** [vergil-project/.github#878](https://github.com/vergil-project/.github/issues/878). `issue-implement` has no automated review until it lands.
- **[#900](https://github.com/vergil-project/.github/issues/900) and [#901](https://github.com/vergil-project/.github/issues/901):** in intake.
- **The first real rebase onto a newer upstream paad tag,** following paad-policy §10's process and gate. This epic ran the process once (onto 2.1.0). The next run tests whether it is written well enough to follow cold.
- **vergil-tooling#3936's anchor fix** rides the next vergil-tooling release.

## Appendix A. Operational notes

**D1 (the fork's `develop`):**

1. `git switch -c develop unattended-mode && git rebase paad--v2.1.0`. Five fleet patches, conflicts in `CHANGELOG.md` and `alignment/SKILL.md` only.
2. `vrg-git diff paad--v2.1.0:plugins/paad paad--v2.1.0:preview/paad` shows only the `-preview` suffix, `metadata: internal: true` and the manifest version.
3. One commit. The catalog `source` becomes `./preview/paad`. There is no `version` in the catalog entry or in `preview/paad`'s manifest. `check-versions` compares `package.json` with `plugins/paad` and fails on either version key. `check-skill-versions` expects `v<plugins/paad version>-preview` for a tree with no version. Each patched check was proven to fail on a bad tree, with the files restored from a copy, not by git.
4. `make test` passes.
5. The gate, in a new session started with `vrg-vm session -- claude --plugin-dir …/paad/preview/paad` and the host's `plugin_dirs = []`.
6. Push guarded by the push rule: `test -z "$(git status --porcelain)" && test "$(git rev-parse --short=7 HEAD)" = <validated> && git push -u origin develop`.
7. On the host: `extraKnownMarketplaces.paad` becomes `wphillipmoore/paad` at ref `develop`, then `vrg-host plugins` and `vrg-vm update --all`.

**Where things live:**

- `~/.config/vergil/` files are read on the Mac host. A VM session cannot see them.
- `vrg-vm` runs only on the host.
- A plugin installed with no version reports its version as the first 12 characters of the commit SHA.

**D2's retirement:**

- delete `plugin-dirs.toml`;
- `git worktree remove paad-fleet`;
- `git branch -D fleet`;
- check `gh pr list -R Ovid/paad --state open --head unattended-mode` comes back empty before deleting `unattended-mode` locally and on `origin`.
