# Container pinning & version management — Retrospective

## §0 At a glance

We set out to make container tool-version pinning **managed, not permanent**. Before this epic, a pin was a silent freeze: nobody was alerted, nothing recorded what was held back or why, and the drift had already leaked into the docs. The plan had three workstreams in vergil-containers:

- **WS2:** pins as data (`docker/pins/pins.yml`), a generated catalog, and a CI gate so no pin merges without a justification;
- **WS3:** immutable datestamp image aliases (`{prefix}-{lang}:{version}-YYYYMMDD`), a sliding-window reaper (prod 30 days, dev 7 days) and a repoint rollback;
- **WS4:** an internal-state exposure report that flags pins due for re-evaluation.

WS1 (nightly branch coupling) was factored out to vergil-containers#413 and closed on its own on 2026-08-12.

**What shipped:** all three workstreams, plus one piece the plan didn't have:

- **WS2:** the pin data model, extractor, CI gate and generated `CATALOG.md`. The audit freed 14 of the 24 seed pins: 13 package-manager tools, plus govulncheck once rollback existed.
- **WS2, added mid-flight:** the 9 binary-download tools couldn't float the way the plan assumed (§1). They became **auto-managed** pins instead: pinned in source and moved forward by a weekly bump-PR job.
- **WS3:** datestamp aliases on every image. The reaper is armed and validated, with real deletions confirmed on all 12 packages. A documented repoint procedure exists; turning it into tooling was split out to .github#205.
- **WS4:** the exposure report.
- **The closing bookends:** the report-only Dependabot brainstorm (#158) became its own epic, .github#420, which has shipped. The external-action SHA-pinning brainstorm (#165) moved to the intake queue at close (§5).

### Work delivered

- **Span:** epic opened 2026-07-16; spec and plan merged the same day (.github#163). Implementation ran 2026-07-16 → 2026-07-26. The epic then stayed open until 2026-10-24 for its two brainstorm bookends (§2).
- **Repos touched (3):** vergil-containers, vergil-tooling, .github.
- **Tasks:**
  - 19 sub-issues: 18 closed, plus this retrospective.
  - 3 were operational, with no PR: validation #436 and the brainstorms #158 and #165. #165 was moved to intake at close, not completed.
  - 7 tasks were added mid-flight: #165, #435, #436, #449, #451, #453 and vergil-tooling#2524.
- **Releases carrying epic work:**
  - vergil-containers **v2.1.10** (2026-07-16): all eight planned tasks;
  - vergil-containers **v2.1.12** (2026-07-28): the auto-bumper, the docs review and the three reaper fixes;
  - vergil-tooling **v2.1.165** (2026-07-24): the per-repo allowed-actions override.
- **PRs (15 merged), grouped by repo.**

| Repo | PR | Task | What it did |
| --- | --- | --- | --- |
| .github | #163 | #157 | Spec and plan for the epic |
| vergil-containers | #423 | #415 | `pins.yml` data model, extractor, and the `quality / pins` CI gate |
| vergil-containers | #425 | #417 | `CATALOG.md` generated from the Dockerfiles and `pins.yml`, with a staleness check |
| vergil-containers | #428 | #418 | Pin audit: 13 package-manager tools freed; 7 binary tools kept as mechanism pins |
| vergil-containers | #431 | #422 | govulncheck freed; trivy and scorecard kept as mechanism pins; semgrep confirmed floating |
| vergil-containers | #424 | #416 | Immutable datestamp aliases published at the rolling tag's digest |
| vergil-containers | #426 | #419 | Sliding-window reaper for datestamp aliases (prod 30d / dev 7d), dry-run by default |
| vergil-containers | #427 | #420 | Repoint rollback procedure documented |
| vergil-containers | #430 | #421 | Exposure report: due for re-evaluation, plus installed versions per image |
| vergil-containers | #443 | #435 | Weekly leading-edge bumper for the 9 binary tools; new `auto-managed` pin state |
| vergil-containers | #444 | #414 | Documentation review: containers docs reflect the pinning model |
| vergil-containers | #450 | #449 | `package-cleanup` startup failure from a newline inside a `${{ }}` expression |
| vergil-containers | #452 | #451 | `dataaxiom/ghcr-cleanup-action` allowed through the `[actions]` override |
| vergil-containers | #454 | #453 | Reaper's `exclude-tags` dropped; the action's ReDoS guard rejected it |
| vergil-tooling | #2525 | #2524 | Per-repo `[actions] extra-allowed-patterns` override in `vergil.toml` |

## §1 How the plan evolved

The plan had no "Evolution during execution" log. This narrative is drawn from the PR descriptions and the decision comments on the tasks.

**The planned work landed in a day.** All eight planned tasks merged on 2026-07-16, in the dependency order the plan gave (T1 → T2 → T3; T4 → T5, T6; T7 after T6 and T3; T8 after T2). They shipped together in vergil-containers v2.1.10. The plan's code-level steps carried most of the work. What changed was where the plan met real install mechanisms and real GitHub behaviour:

- **"Free the pin" depended on how the tool installs.** The plan assumed every seed pin could be freed by dropping its version. That held for the 13 tools installed by pip, uv, go, cargo or npm. It didn't hold for the 7 tools installed from release tarballs (shellcheck, shfmt, actionlint, git-cliff, hadolint, opentofu, nfpm): the version is part of the download URL and the checksum. The only floating form would have been an unauthenticated "latest" lookup on every image build. #418 kept them as documented **mechanism pins** and flagged the choice for the human. #422 added trivy and scorecard to the same class, which made 9.
- **The 9 mechanism pins got a new answer: automate the bump, don't float.** The human chose approach B for #435 on 2026-07-17. The version stays pinned in source, so builds stay hermetic and reproducible. A weekly job finds the leading edge through the `/releases/latest` redirect and opens a reviewed bump PR. This added an `auto-managed` state to `pins.yml`. It's a refinement of the doctrine: for these tools, "follow the leading edge" means a mechanized bump with a human merge gate, not a floating install.
- **The rollback demonstration moved to a validation task.** T6 planned to demonstrate a repoint end to end, but no datestamp aliases existed until the next build. The demonstration became validation task #436.
- **#436 found that the reaper had never run.** Its first run on 2026-07-24 failed: the `Package cleanup` workflow had hit `startup_failure` on every scheduled run since it was created. There were two causes, and a third problem behind them:
  1. the org's allowed-actions policy rejects the third-party `dataaxiom/ghcr-cleanup-action` when the run starts, before any job runs and without any logs. The policy is managed as code, so a change made in the UI would have been reverted. The fix was a new per-repo override in vergil-tooling (#2524) and its use in vergil-containers (#451);
  2. a newline inside a `${{ }}` expression (#449). Its PR (#450) also closed #449 without the allowlist change, and #451 was filed to catch that;
  3. once the workflow started, the action's ReDoS guard rejected the reaper's `exclude-tags` regex (#453). It was dropped because it was redundant with the precise delete rule.

  #436 passed on 2026-07-26 with real deletions confirmed.
- **The rollback was validated as a mechanism, not as an operation.** The live repoint reached the manifest `PUT` and stopped there, because the agent deliberately has no `write:packages`. Managed rollback and roll-forward tooling was split out to .github#205 rather than built on raw `docker` with blanket write access.
- **A new bookend appeared (#165).** On 2026-07-16, freeing pip, npm and go pins collided with hadolint's DL3013, DL3016 and DL3062 rules. On inspection, hadolint pins for *reproducibility*, which this epic deliberately handles a different way, while OpenSSF Scorecard's Pinned-Dependencies check pins external actions for *supply-chain security*, which the epic hadn't addressed. That split became a second closing brainstorm.

## §2 Lessons learned

- **Classify pins by install mechanism before the audit.** The free-or-justify rule was right, but "free" means different things for a package-manager install and for a release tarball. Had the audit started from the install path, the mechanism-pin class and the auto-bump answer would have been in the plan, not found mid-flight.
- **Scheduled automation isn't done until a scheduled run has been seen.** Three pieces of this epic depend on a schedule, and all three were broken at first in ways nothing reported:
  - the reaper never started for ten days (#449);
  - the exposure report, meant to anchor a daily morning review, wasn't called by any workflow until vergil-containers#577 (2026-08-23);
  - the bumper opened a new PR every week instead of refreshing the waiting one, until #576 (2026-08-23). Five PRs stacked, and trivy sat a month behind a release that fixed CVEs.

  Validation #436 caught the first; nothing caught the other two. Next time, the acceptance criteria for any scheduled job should say "a scheduled run was observed doing the right thing," not "the workflow merged."
- **Tests that CI doesn't run rot silently.** The pins test suite wasn't run by CI. A later golangci-lint re-pin left it red on develop, and nothing reported it until #577 wired the suite into the `pins` job.
- **Org policy fails before the workflow runs.** A disallowed action gives `startup_failure` with no logs. The agent can't read the run annotation (the `user` identity is denied `gh api`), so the root cause needed the human and the Actions UI. The first hypothesis, an `inputs` context problem, was wrong. When a workflow fails at startup, check the action allowlist first.
- **Brainstorm bookends shouldn't gate closure.** The implementation finished on 2026-07-26, but the epic stayed open for three more months because its closing bookends were brainstorms. #158 did its job (it produced epic #420), but it held a finished epic open while it did. The lifecycle has since made the retrospective the terminal gate and treats follow-on brainstorms as forward-looking items. This close-out applies that: #165 moved to intake.
- **Same next time:**
  - a plan detailed enough to execute in a day;
  - the dry-run-by-default reaper, armed only after a validated preview;
  - fixing policy as code (#2524) rather than in the UI, where the next `apply` would have reverted it.

## §3 Compromises & tradeoffs

- **9 tools are still pinned.** shellcheck, shfmt, actionlint, git-cliff, hadolint, opentofu, nfpm, trivy and scorecard are auto-managed, not floating. Builds stay reproducible and offline-capable, and each bump needs a human review. The cost: they lag the leading edge by up to a week plus review time, and longer when bump PRs go unreviewed (§2).
- **Rollback is a documented procedure, not tooling.** It needs raw `docker buildx imagetools` and `write:packages`. The mechanism is validated; making it a managed `vrg-*` tool waits on .github#205.
- **The cleanup action is allowed at any version.** The override is `dataaxiom/ghcr-cleanup-action@*`, a least-privilege pattern for one action, but not pinned to a commit SHA. The supply-chain pin is the open #165 question.
- **hadolint's pinning rules are ignored on purpose.** DL3013, DL3016 and DL3062 enforce reproducibility pins, which this epic replaced with floating plus mechanized management and datestamp aliases.
- **WS4 reports internal state only.** Upstream drift for floating tools was left to the Dependabot follow-on. Epic #420 covers security advisories, and lists a pin-drift view as a non-goal, so there is still no full upstream-drift view. #577's `auto_managed_behind` signal covers only the 9 auto-managed tools.
- **The rollback window is short.** Dev keeps aliases for 7 days and prod for 30. A regression found later than that can't be rolled back by repointing.

## §4 New problems & opportunities

| Surfaced | Where it went |
| --- | --- |
| Report-only Dependabot security audit | epic .github#420 (shipped and closed) |
| SHA-pinning external actions via build-time expansion (Scorecard Pinned-Dependencies) | .github#165, moved to intake at close |
| Managed rollback and roll-forward tooling | .github#205 (open) |
| Third-party actions blocked by the org allowlist | vergil-tooling#2524: a reusable per-repo override (shipped) |
| Binary tools can't float without a build-time network lookup | vergil-containers#435: weekly auto-bump (shipped) |
| Bump PRs stacking weekly; trivy a month behind | vergil-containers#573 → #576 (fixed 2026-08-23) |
| Exposure report never run; pins tests not in CI | vergil-containers#574 → #577 (fixed 2026-08-23) |
| ReDoS guard rejects `exclude-tags` | vergil-containers#453 → #454 (fixed) |
| A PR closed its task without the task's fix (#450 closing #449) | caught by #451; logged here, not otherwise acted on |
| Auto-bump to fold into the mechanized dependency update | epic .github#52 (open) |

## §5 What's next

- **.github#165** (intake): design build-time SHA expansion for external action refs in vergil-actions, and hand it off there as its own epic. It carries the doctrine split this epic found: a *reproducibility pin* is handled by floating plus mechanization, while a *security pin* means an immutable ref that is resolved and refreshed automatically, never hand-managed.
- **.github#205**: managed rollback and roll-forward tooling for image repointing.
- **.github#52**: mechanized dependency update. The vergil-containers bumper (#435) was built to fold into it.
- **.github#420**: the report-only Dependabot follow-on. It has shipped and closed, with its own retrospective.

## Appendix A — Operational notes

- **Arming the reaper:** it ships dry-run. Dispatch `Package cleanup` with *Preview only* checked, confirm the preview selects only `*-YYYYMMDD` aliases past the window, then set `PACKAGE_CLEANUP_DRY_RUN=false`. The armed run on 2026-07-26 left one dev alias (`…-20260720`) and all four prod aliases, with every rolling, version, `*-candidate` and `cache-*` tag intact.
- **Allowing a third-party action in one repo:** add `[actions] extra-allowed-patterns` to that repo's `vergil.toml` (vergil-tooling ≥ 2.1.165), then have a human run `vrg-github-repo-config apply --repo <repo>`. The agent identity gets a 403 on actions permissions.
- **Repoint rollback:** follow `docs/site/docs/operations/rollback.md` in vergil-containers. It needs `write:packages`, and works only within the retention window.
