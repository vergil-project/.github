# ops-issue-queue: an operations queue fed by every unattended job — design

- **Epic:** vergil-project/.github#1494
- **Planning issue:** vergil-project/.github#1470 (review history is in its comments)
- **Status:** draft, in design review

## 1. Intent

The fleet runs 17 unattended jobs (scheduled ops jobs, sweeps, refreshes,
rebuilds, releases and publishes). When one of them fails, the only signal
today is a GitHub notification email; only the language lifecycle check
files an issue on its own failure, and only partly. Email is noise, so
failures go unseen until they hurt something, and several jobs are red
right now with nobody tracking them.

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
  automation and any filed by hand. A private repo has its own (§5.1).
- **G2. The hoshin epic exists.** `Epic (hoshin): vergil-project` is
  created now, so the `epic-management` filter shows it. The code already
  labels it correctly; it simply has never been created.
- **G3. Every unattended job reports itself.** Each of the jobs in the
  inventory (§4) ends with a reporting step that files an **ops issue**
  under its org's operations epic when the job fails or finds one or more
  problems, and records a green run on an open issue.
- **G4. Every unattended job runs as its org's automation bot**, the
  GitHub App behind `APP_CLIENT_ID`/`APP_PRIVATE_KEY`, except where §5.4
  records an exception.
- **G5. Existing issue-filers follow the same rules.** The automation that
  files operational issues today files them under the operations epic and
  stops closing them automatically.

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
   `GITHUB_TOKEN`.
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
17. **`image-publish`**: vergil-containers `cd.yml` →
    `cd-docker-publish.yml`; push. `GITHUB_TOKEN`.

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
  release job's package-index dispatch steps continue on error.
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
- Homed like the queue kinds (`queue_home`), not like hoshin: a public
  repo's ops issues go under `Epic (operations): <org>` in `<org>/.github`;
  a private repo self-homes its own `Epic (operations): <repo>` in its own
  issue list, so nothing about private work, not even a sub-issue count,
  reaches the public `.github`. Today that is two repos:
  `logical-minds-foundry/mq-protocol-gateway` and
  `logical-minds-foundry/mq-gateway-replay-lab`, both running `ops.yml`.
  The sweep drains every home the kind has, as it does the queues.
- `--epic operations` resolves it in `vrg-issue-create` and
  `vrg-epic-move`, created on first use by `ensure_standing_epic`.
- It is drained into quarterly archives by the sweep like the other
  non-ad-hoc kinds.
- The `operations` label (on the epic) and the `ops-report` label (on an
  ops issue the reporter files, §5.2) are added to the label registry
  (`data/labels.json`).
- Ops issues are ordinary, PR-workable tasks: a fix lands as a PR in the
  repo, which closes the ops issue only when the human decides it is
  fixed. `vrg-intake-route` does not route them; they are not queue items.

### 5.2 The reporter: `vrg-ops-report` (vergil-tooling)

One command, run once at the end of every unattended job's run:

```text
vrg-ops-report --repo <owner>/<repo> --job <job-name> \
  --result <success|failure|cancelled|findings> --run-url <url> \
  [--step <failing step>] [--findings-file <md>]
```

- **Find the issue** by the report key: an **open** issue in `--repo`
  labelled `ops-report` whose title is exactly
  `ops: <repo> / <job-name> failing`. A closed issue is never matched.
- **failure / findings / cancelled**: no open issue → create one with
  `vrg-issue-create --epic operations`, labelled `ops-report`, its body
  naming the job, the run, the step, the date, and the most recent closed
  issue with the same title ("previous occurrence: #N") if one exists. An
  open issue → add a comment with the same facts. `--findings-file` adds
  the job's findings (already public-safe, written by the job) to the body
  or comment.
- **success**: an open issue → add a "succeeded on DATE, RUN-LINK"
  comment. No open issue → do nothing.
- **Never closes an issue.**
- Two open issues with the same key (a race) → comment on the oldest,
  name the other in that comment, close neither.
- It verifies that its token belongs to the automation App
  (`VRG_AUTOMATION_APP`) and refuses otherwise.
- Any failed GitHub call exits non-zero with the error.

### 5.3 The reporting step in workflows (vergil-actions)

- A composite action `actions/ops-report` mints the App token and runs
  `vrg-ops-report`.
- Each workflow containing an unattended job adds a final job, one per
  reported job or one reporting all of them, that runs with `if:
  always()`, reads `needs.<job>.result`, and calls the action with that
  job's name. The job name is passed explicitly and is stable (§3).
- Jobs that find problems while succeeding report `findings` instead of
  exiting 0 in silence:
  - **pin-exposure** reports when the set of lagging pins differs from what
    its open issue last recorded; an unchanged lag adds no nightly comment,
    and the issue stays open until the human decides it is dealt with.
  - **The release job's** package-index dispatch steps, which continue on
    error, report a finding when they fail. A caller turning evidence
    enforcement off is a choice, not a failure, and is not reported.
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

- The language lifecycle check files all its issues, failure and change
  issues alike, under the operations epic instead of ad-hoc, and stops
  closing any of them: on a recovery, or a change no longer reported, it
  comments ("recovered in run …", "no longer reported in run …") and
  leaves the issue open. It keeps its per-family issues, which are finer
  than per-job; its generic `vrg-ops-report` step reports only a crash of
  the job itself, and a family error does not also turn the job red, so
  one red night opens one issue.
- `vrg-update-deps` is not unattended and is out of scope.
- The epic sweep and rollup keep their housekeeping; their own failures
  are reported like any other job's.

### 5.6 Rollout

- vergil-tooling: the standing kind, labels and `vrg-ops-report`;
  released.
- vergil-actions: the `ops-report` action, and the reporting step and
  identity fix in each shared reusable workflow; released and tagged.
- Each caller repo picks up the tag and adds reporting to the jobs it
  defines itself (vergil-containers' rebuilds, pin-exposure, lifecycle,
  cleanup, cache warm; docs' refresh; packages' publish-index).
- A one-off deployment first checks that the automation App
  (`APP_CLIENT_ID`, `APP_PRIVATE_KEY`, `VRG_AUTOMATION_APP`) is configured
  in every org and private repo with unattended jobs, then creates each
  operations epic (one per org with unattended jobs, one per private repo
  running them) and `Epic (hoshin): vergil-project`.
- Scope is every unattended job that exists today, in every org, including
  the sweep, rollup and release in mq-rest-admin, mnemosys and diogenes.
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
- How `findings` are produced by each job that has them.
- The order of the rollout across repos.
