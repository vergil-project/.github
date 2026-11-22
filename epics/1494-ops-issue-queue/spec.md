# ops-issue-queue: an operations queue fed by every unattended job — design

- **Epic:** vergil-project/.github#1494
- **Planning issue:** vergil-project/.github#1470 (review history is in its comments)
- **Status:** approved

## 1. Intent

The fleet runs 17 jobs with no human watching them (scheduled ops jobs,
sweeps, refreshes, rebuilds, releases and publishes); 15 of them get a
reporting step here, one is deprecated and one is out of scope (§4.1).
When one of them fails, the only signal today is a GitHub notification
email; only the language lifecycle check files an issue on its own
failure, and only partly. Email is noise, so failures go unseen until
they hurt something, and several jobs are red right now with nobody
tracking them.

**The why:** the goal for the rest of 2026 is to keep the automation
running green, all day, every day. That needs every failure recorded as
work, in one queue per org that the human and the agents review every
morning. This epic scales an enterprise discipline (an operations queue
fed by the automation itself) down to a single operator.

## 2. Goals, non-goals, anti-goals

### 2.1 Goals

- **G1. An operations epic per org.** A new perpetual standing epic kind,
  `operations`: `Epic (operations): <org>`, labelled `epic` +
  `operations`, with the derived category label `epic-management`. It is
  the org's queue for operational issues: those filed by unattended
  automation and any filed by hand.
- **G2. Every unattended job reports itself.** Each of the jobs in the
  inventory (§4) ends with a reporting step that files an **ops issue**
  under its org's operations epic when the job fails, and records the
  first green run after a failure on an open issue.
- **G3. Every unattended job runs as its org's automation bot**, the
  GitHub App behind `APP_CLIENT_ID`/`APP_PRIVATE_KEY`, except where §5.4
  records an exception.
- **G4. Existing issue-filers follow the same rules.** The automation that
  files operational issues files them under the operations epic and
  closes none of them automatically. For the one such job today, the
  language lifecycle check, this binds its redesign (§5.5).
- **G5. The morning review reads the queue.** hoshin's daily review
  (vergil-claude-plugin `skills/hoshin/references/daily.md`) takes its
  failures from each org's operations epic: issues opened, failure
  comments and recoveries since the last review. Actions run history
  stays only as a backstop for runs that never reported (follow-on F1).

### 2.2 Non-goals (follow-on epics, recorded in the follow-on brainstorm)

- **F1. An org-level watcher** that reads each org's Actions run history
  nightly and reports runs that crashed before reporting or never ran (a
  scheduled workflow GitHub disabled after 60 days of repo inactivity).
- **F2. Splitting the monolithic `ops.yml`** files into one workflow per
  job, each with its own schedule. A workflow has one schedule, so today
  every bundled job starts at the same time.
- **F3. Filling the coverage gaps:** mq-rest-admin has no `ops.yml`,
  mnemosys has no epic sweep, diogenes runs an older release workflow, and
  vergils-nemesis has only CI. This epic covers the jobs that exist today.
- Tooling to work the operations queue (triage, bulk close, reports).
  The queue's shape will be learned by using it first.
- Removing GitHub email notifications. They may stay; they are no longer
  the signal.

### 2.3 Anti-goals

- **Auto-closing an ops issue.** A failure that recovers on its own is
  still a failure nobody explained; closing it on a green run would drop it
  from the queue and mask a problem likely to return. Nothing in this epic
  closes an ops issue. Closing is always a decision made by the human, or
  an agent working the queue, including "close it as unreproducible and
  see whether it comes back".
- **One issue per run.** A job failing night after night produces one
  issue with a growing history, never a pile of duplicates.
- **A silent reporter.** A reporting step that cannot file or update its
  issue fails red with the error; it never swallows it. (Its own failure
  is then email only; F1 is the backstop.)
- **Leaking secrets into public issues.** Ops issue bodies and comments are
  public. They carry the run link, the job, the step name and the
  conclusion, never log excerpts or environment values.
- **Publishing a security signal.** A job whose failure means a security
  finding exists (today `dependabot-audit` and `github-config`, the
  configuration audit, and any security scan added later) files no
  public ops issue. Its failure is recorded through
  `vrg-security-record`, and anything public about it names only the
  neutral record, never the repo's alerts or the job's output. This
  follows the fleet's existing rule for security-related failures
  (hoshin's daily review names the dependency audit and the GitHub-config
  audit among them). Every other job reports publicly as above.

## 3. Terms

- **Ops issue:** an issue filed (by automation or by hand) under an
  operations epic. Named so to avoid the existing term **operational
  task**, which means a `validation` or `deployment` task
  (`lib/epics.py`) and keeps that meaning.
- **Report key:** what identifies a job's ops issue: org, repo and a
  stable **job name** the reporting step is given explicitly (for example
  `nightly-rebuild-dev`). It never includes the workflow file name, so F2
  can move a job to another file without changing its key.

## 4. Current state (inventory, 2026-11-22)

Verified in code and, where marked, in recent run history.

### 4.1 The jobs

Caller sets: **OPS** is the `ops.yml` caller of `ops-github-config.yml`
in 24 repos (15 in vergil-project, 6 in logical-minds-foundry, 3 in
mnemosys-project; none in mq-rest-admin-project, diogenes-project or
vergils-nemesis). **ROLLUP** is `epic-rollup.yml` in the OPS repos plus
the active mq-rest-admin code repos. **SWEEP** is `epic-sweep.yml` in the
`.github` of vergil-project, logical-minds-foundry and
mq-rest-admin-project. **CD** is every repo's `cd.yml`.

Each entry gives the job's stable name, where it is defined, its trigger
and callers, and the identity it runs as today. "App" is the org's
automation App token; "verified" means the job checks the token against
`VRG_AUTOMATION_APP`.

1. **`github-config`**: vergil-actions `ops-github-config.yml`; daily;
   OPS. App, verified.
2. **`branch-sweep`**: same file; daily; OPS. App, unverified.
3. **`dependabot-audit`**: same file; daily; OPS. App, unverified.
4. **`epic-sweep`**: vergil-actions `ops-epic-sweep.yml`; daily; SWEEP.
   App (org-wide), unverified.
5. **`epic-rollup`**: vergil-actions `ops-epic-rollup.yml`; on issue
   close; ROLLUP. App, unverified.
6. **`upstream-refresh`**: vergil-actions `ops-upstream-refresh.yml`;
   daily; vergil-containers and vergil-python. `prepare` uses a read-only
   `GITHUB_TOKEN` by design; `publish` uses the App, verified.
7. **`pin-exposure`**: vergil-containers `ops.yml`; daily. Read-only
   `GITHUB_TOKEN`. **Deprecated, not reported:** it only prints how far
   pins lag upstream, to a run summary nobody reads; `upstream-refresh`
   (job 6) is the job that fixes lag, with a self-merging bump PR (§5.5).
8. **`language-lifecycle`**: vergil-containers `ops.yml`; daily. App,
   unverified.
9. **`nightly-rebuild-dev`**: vergil-containers `ops.yml` →
   `cd-docker-publish.yml`; daily. `GITHUB_TOKEN`.
10. **`nightly-rebuild-prod`**: vergil-containers `ops.yml` →
    `cd-docker-publish.yml@main`; daily. `GITHUB_TOKEN`.
11. **`package-cleanup`**: vergil-containers `package-cleanup.yml`;
    weekly. `GHCR_CLEANUP_TOKEN` (likely a personal token) or
    `GITHUB_TOKEN`.
12. **`package-cache-warm`**: vergil-containers `package-cache-warm.yml`;
    weekly. `GITHUB_TOKEN`.
13. **`publish-index`**: vergil-actions `publish-index.yml`; on dispatch
    and weekly; packages. `GITHUB_TOKEN`.
14. **`docs-refresh`**: vergil-actions `cd-docs-refresh.yml`; daily;
    vergil-project/docs and logical-minds-foundry/docs. `GITHUB_TOKEN`.
15. **`docs-deploy-develop`**: vergil-actions `cd-docs.yml`; push to
    develop; CD. `GITHUB_TOKEN`.
16. **`release`**: vergil-actions `cd-release.yml` (diogenes on
    `@v2.0`); push to main; CD, including vergil-actions' own. Mixed: tags
    and the index dispatch use the App; the GitHub Release uses
    `GITHUB_TOKEN`; registries use OIDC or `CARGO_REGISTRY_TOKEN`.
    **Out of scope:** a push to main comes only from a release a person
    runs and watches with `vrg-release`, and epic
    vergil-project/.github#1440's release records already track and
    recover each failed CD publish job, so an ops issue would track the
    same failure twice. Its package-index dispatch steps are covered the
    same way.
17. **`image-publish`**: vergil-containers `cd.yml` →
    `cd-docker-publish.yml`; push. `GITHUB_TOKEN`.

Jobs 15 and 17 also run on a release, but they run on every push to
develop as well, including the merges of self-merging automation PRs
(github-config sync, upstream refresh) that nobody watches, so they stay
in scope, reported only on pushes outside a release. A push to `main`
and a release's develop back-merge (recognised by its merge commit's
release branch) report nothing: the release records of epic
vergil-project/.github#1440 already track and recover their failed CD
jobs, as for job 16.

The stable names are this spec's proposal; the plan may refine them
before any reporting step ships, never after.

Red with nobody tracking it on 2026-11-22 (run history): vergil-containers
`upstream-refresh` publish and both nightly rebuilds, three nights running;
`github-config` in logical-minds-foundry `mq-resiliency-lab-for-linux` and
`mq-resiliency-observability` and in mnemosys-project `melete`, six nights
running; `dependabot-audit` in `mq-resiliency-lab-for-linux`.

Not unattended, and so out of scope: `vrg-update-deps` (human identity
only), `vrg-upstream-refresh` run locally, pull-request CI, smoke and
dispatch-only workflows. No host scheduler (cron, systemd timer) runs fleet
automation; the macOS host's launchd was not inspected.

### 4.2 By category

- **Email only on failure, files nothing (14):** jobs 1–3, 6, 7 and
  9–17.
- **Files issues as its purpose; its own failure is email only (2):** the
  epic sweep and the epic rollup (jobs 4, 5).
- **Files an issue on its own failure, partly (1):** the language
  lifecycle check (job 8).

- **Not running as the bot:** nightly-rebuild-dev/prod, the image publish
  after release, package-cache-warm, publish-index, cd-docs-refresh and the
  develop docs deploy use `GITHUB_TOKEN`; pin-exposure has no write token;
  package-cleanup uses `GHCR_CLEANUP_TOKEN || GITHUB_TOKEN` (likely a
  personal token); the main release creates its GitHub Release with
  `GITHUB_TOKEN`; upstream-refresh `prepare` uses a read-only
  `GITHUB_TOKEN` by design, keeping the App key off the runner that runs
  the hook. Only github-config and upstream-refresh `publish` verify that
  their token is the automation App's.
- **Silent today:** pin-exposure exits 0 when it finds lagging pins; the
  release job's package-index dispatch steps continue on error (job 16,
  out of scope).
- **Existing issue-filers:** the language lifecycle check files
  `language-lifecycle` issues under `Epic (ad hoc): vergil-containers` and
  closes a failure issue when its family recovers (`recovered in run …`)
  and a change issue when it is no longer reported. `vrg-update-deps`
  files a fix task under the repo's ad-hoc epic, but it is not unattended:
  it refuses every identity but a human's. The epic sweep and rollup create
  standing and archive epics and move issues; those are epic housekeeping,
  not ops issues, and stay as they are.

## 5. Design

### 5.1 The operations standing kind (vergil-tooling)

- A sixth `StandingKind` in `lib/epics.py`, `OPERATIONS`: key and
  sentinel `operations`, title prefix `Epic (operations):`, labels
  `epic` + `operations`, archive `Archive (operations):` labelled
  `archive` + `operations`. It is a management kind, so
  `category_labels` derives `epic-management` with no new code.
- Homed like hoshin (`hoshin_home`): always `Epic (operations): <org>` in
  `<org>/.github`, one per org, whatever repo the job ran in. No repo
  self-homes one, and nothing branches on a repo's visibility (epic
  vergil-project/.github#1476 removes private-repo support).
- `--epic operations` resolves it in `vrg-issue-create` and
  `vrg-epic-move`, created on first use by `ensure_standing_epic`.
- It is drained into quarterly archives by the sweep like the other
  non-ad-hoc kinds.
- The `operations` label (on the epic) and the `ops-report` label (on an
  ops issue the reporter files, §5.2) are added to the label registry
  (`data/labels.json`).
- **Ops issues are not PR-workable.** An ops issue records that something
  broke; it is never the task a PR closes. `vrg-pr-workflow report-ready`
  and `vrg-submit-pr` refuse an issue whose parent is an operations epic,
  as they refuse a queue item. A code fix found while working an ops issue
  is filed as a follow-up task in its own repo, never linked to the ops
  issue as its PR's task. The human and the agent working the queue close
  an ops issue once the automation is fixed or the problem is otherwise
  dealt with. The nightly sweep's drift close skips ops issues, and
  `vrg-intake-route` does not route them; they are not queue items.

### 5.2 The reporter: `vrg-ops-report` (vergil-tooling)

One command, run once at the end of every unattended job's run:

```text
vrg-ops-report --repo <owner>/<repo> --job <job-name> \
  --result <success|failure|cancelled> --run-url <url> \
  [--step <failing step>]
```

- **Every read goes through the trust layer** of epic
  vergil-project/.github#1467, a prerequisite of this epic: the issue
  lookup, the previous-occurrence lookup and any read of an issue's body or
  comments. Only an issue, and only a report comment, written by the
  org's automation App counts; a title or label alone never identifies an
  ops issue. The `ops-report` label is the reporter's alone: an ops issue
  filed by hand is an ordinary issue under the operations epic, and the
  reporter never matches it.
- **Find the issue** by the report key: an **open** issue in `--repo`
  labelled `ops-report` whose title is exactly
  `ops: <repo> / <job-name> failing`, written by the automation App. A
  closed issue is never matched.
- **failure / cancelled**: no open issue → create one with
  `vrg-issue-create --epic operations`, labelled `ops-report`, its body
  naming the job, the run, the step, the date, and the most recent closed
  issue with the same title ("previous occurrence: #N") if one exists. An
  open issue → add a comment with the same facts.
- **success**: comments only on a change of state. An open issue whose
  latest trusted report is a failure or cancelled → add a
  "recovered on DATE, RUN-LINK" comment. An open issue already marked
  recovered, or no open issue → do nothing. Every failure is recorded,
  and only the first green run after one is: the history reads fail,
  fail, recovered, fail, and so on.
- **Never closes an issue.**
- Two open issues with the same key (a race) → comment on the oldest,
  name the other in that comment, close neither.
- It verifies that its token belongs to the automation App
  (`VRG_AUTOMATION_APP`) and refuses otherwise.
- Any failed GitHub call exits non-zero with the error.

### 5.3 The reporting step in workflows (vergil-actions)

- The action mints the App token only on the default branch's ref
  (`develop`), the rule `ops-github-config.yml` and
  `ops-upstream-refresh.yml` already apply; on any other ref it reports
  nothing.
- A composite action `actions/ops-report` mints the App token, passes
  the trust layer's inputs (the delivered trusted-author list and
  `VRG_AUTOMATION_APP`, as epic vergil-project/.github#1467 §5.1a
  defines them) into the reporter's environment, and runs
  `vrg-ops-report`.
- Each workflow containing an unattended job adds a final job, one per
  reported job or one reporting all of them, that runs with `if:
  always()`, reads `needs.<job>.result`, and calls the action with that
  job's name. The job name is passed explicitly and is stable (§3).
- **One result per named job.** A named job may span several GitHub jobs
  (upstream-refresh is `prepare` and `publish`; package-cleanup is
  `cleanup` and `reap-aliases`; publish-index is `build-index` and
  `deploy`). Its result combines all of theirs, in this order: any
  `failure` → failure; else any `cancelled` → cancelled; else at least
  one `success` (the rest `skipped`) → success; all `skipped` → no
  report at all. So a failed `prepare` that skips `publish` reports as an
  upstream-refresh failure, and a run where every job was skipped by
  design reports nothing.
- A job reports only whether it worked. Reporting a problem that a job
  finds while succeeding is not this epic's model: a job that finds
  drift fixes it with an automated PR, and a fix it cannot make fails the
  job. The report-only `pin-exposure` job is therefore deprecated (§5.5).
- A security-signal job (§2.3) reports through the security-record path,
  never `vrg-ops-report`'s public issue; how the reporter or the
  workflow routes it is the plan's to decide.
- The reporter creates its labels if a repo lacks them (`ensure_labels`),
  so a repo outside github-config's label sync still works.

### 5.4 Bot identity

- Every unattended job's write steps use the App token.
- **Exception:** a step may use a weaker token only where the job's design
  needs it (upstream-refresh `prepare`'s read-only token, which keeps the
  App key off the runner that runs an untrusted hook) or where the App
  cannot perform the operation. Each exception is recorded in the spec's
  inventory with its reason. Whether the App can push to and delete from
  GHCR is **unverified** and decides the rebuild, publish and cleanup
  jobs: the plan's first rollout task proves it on one job before the
  others are changed.

### 5.5 Existing issue-filers

- The language lifecycle check's publisher is not changed here. Epic
  vergil-project/.github#1467 (§6.2) disables it until
  vergil-project/.github#1128 re-architects its PR and issue handling.
  That redesign follows this epic's rules: every issue it files, failure
  and change issues alike, goes under the operations epic, and it closes
  none of them automatically (a recovery, or a change no longer reported,
  is a comment). Once the job is re-enabled, it gets the generic
  `vrg-ops-report` step for its own crash, like any other job (§5.3).
- **`pin-exposure` is deprecated.** Its job is removed from
  vergil-containers' `ops.yml`, with a note that `upstream-refresh` is the
  path that fixes pin lag. A lag `upstream-refresh` cannot fix (a bump it
  cannot open or merge) fails that job, and the failure reaches the
  operations queue like any other.
- `vrg-update-deps` is not unattended and is out of scope.
- The epic sweep and rollup keep their housekeeping; their own failures
  are reported like any other job's.

### 5.6 Rollout

- **Prerequisite:** the trust layer of epic vergil-project/.github#1467
  has shipped in vergil-tooling before `vrg-ops-report` is built on it.
- **Reporter first, identity second.** The reporting rollout (G1, G2, G4, G5)
  ships first and depends on no identity change: the `ops-report` action
  mints its own App token whatever identity the job's other steps use.
  The identity changes (G3) follow, starting with the GHCR proof (§5.4),
  so an App that cannot do the GHCR work reshapes only the identity
  tasks, never the queue.
- vergil-tooling: the standing kind, labels and `vrg-ops-report`;
  released.
- **Callers pass the App secrets first.** Callers pin the moving `@v2.1`
  tag, so they pick up a vergil-actions release at once. A reusable
  workflow that gains reporting but declares no secrets today
  (`cd-docs.yml`, `cd-docs-refresh.yml`, `publish-index.yml`) gets
  `APP_CLIENT_ID` and `APP_PRIVATE_KEY` declared `required: false`, and
  every caller of it passes them, before the release that adds its
  reporting step. Otherwise that release turns every caller's run red.
- vergil-actions: the `ops-report` action, and the reporting step in each
  shared reusable workflow; released and tagged once the caller pass is
  complete. The identity fixes in those workflows ship in a later
  release.
- Each caller repo picks up the tag and adds reporting to the jobs it
  defines itself (vergil-containers' rebuilds, cleanup, cache warm; docs'
  refresh; packages' publish-index).
- vergil-claude-plugin: hoshin's daily review reads the operations epics
  (G5), once the first of them exists.
- A one-off deployment first checks that the automation App
  (`APP_CLIENT_ID`, `APP_PRIVATE_KEY`, `VRG_AUTOMATION_APP`) and the trust
  layer's delivered trusted-author list are configured in every org with
  unattended jobs, then creates each org's operations
  epic.
- Scope is every unattended job that exists today, in every org, including
  the sweep in mq-rest-admin and the rollup in mq-rest-admin and
  mnemosys, but not the release (job 16).
  Adding jobs an org lacks is F3.
- Validation: each repo's first scheduled run after rollout shows the
  reporter ran (an ops issue for a job already red, such as
  vergil-containers' nightly rebuilds, and nothing for a green job).

## 6. Approaches considered

- **A. Each job reports itself, in the workflow (chosen).** Immediate,
  precise (it knows the job and step), and it shares its edits with the
  identity fix. It cannot report a run that never happened.
- **B. One watcher per org reading run history.** No per-job edits, and it
  catches missed runs, but it is up to a day late, knows only that a run
  failed, does not fix identity, and needs its own backstop. Deferred as
  follow-on F1.
- **C. Both.** A now, B later if missed runs prove a real problem.

## 7. Left to the builder

- The exact `vrg-ops-report` flag names and body layout.
- Whether a workflow uses one reporting job per reported job or one for
  all of them.
- The order of the rollout across repos.
