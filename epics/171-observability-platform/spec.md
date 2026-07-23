# Vergil Observability Platform — Design

**Epic:** vergil-project/.github#171
**Date:** 2026-07-19
**Status:** Draft — Design
**Docs task:** vergil-project/.github#172
**Related prior art (all in `vergil-tooling/docs/specs/`):**
`2026-06-14-vergil-forge-observability-design.md` (#1661 — proves the
instrument; explicitly defers the fleet-wide roll-up this program builds),
`2026-06-08-vergil-gui-vision-feasibility-design.md` (#1532),
`2026-06-13-vergil-forge-local-host-design.md` (#1653),
`2026-06-08-forge-abstraction-strategy-design.md` (#1521).
**Reference mined (not foundational):** `github.com/majiayu000/harness`.

## Purpose

Give a single solo developer running massive parallelism a way to **zoom out
and see all work in flight across the whole fleet** — every codebase, org, VM,
and Claude Code session — and lay the foundation for **metrics over time**.

The pain is concrete. The epic framework already moved the workload from
juggling dozens of issues to juggling a smaller number of epics. But the
developer no longer single-threads epics: one epic routinely spawns others
mid-flight, work spans repositories and organizations, and a growing share runs
on VMs (local Lima and cloud). After a macOS reboot the window layout is gone —
macOS restores apps but not the spatial arrangement of iTerm/Safari windows
across monitors — and there is **no way to see what epics, worktrees, and
sessions are active, or where.** Reconstructing the working environment is
guesswork against memory and GitHub.

This program builds the observability layer that makes the whole in-flight
workload visible at a glance, and — over later bricks — watchable over time.

## Foundational decisions

These were settled during the brainstorm and frame everything below.

1. **Two planes, one pane of glass.** "Observability" here is really two
   subsystems with different data models, and conflating them is the central
   mistake to avoid:

   | | **State-map plane** | **Metrics plane** |
   |---|---|---|
   | Question | *What is in flight, and where?* | *How busy/idle are the VMs? How does it trend?* |
   | Shape | Point-in-time **relational/graph** | **Time-series** |
   | Backend | A **correlator + canonical JSON contract** | **Prometheus/Grafana** (the #1661 pattern) |
   | Wrong backend | ❌ Prometheus (not a metrics problem) | ❌ a CLI snapshot (can't show trends) |

   The map of epics/worktrees/sessions is **not** a metrics problem; forcing it
   into a TSDB would be wrong. VM utilization over time genuinely **is** a
   metrics problem and a CLI snapshot cannot answer it. The platform hosts both
   behind one surface, but they stay distinct subsystems.

2. **Contract-first.** A **canonical, versioned JSON contract** is the
   load-bearing seam that every later surface consumes — the `vrg-fleet` CLI
   first, then Grafana, then a future GUI, then archival. Both prior strategy
   docs (#1661, #1532) independently converged on `--json` / a structured status
   contract as *the* integration point. Building the contract *is* starting the
   platform, from the data model up.

3. **Grouping by dependency, not a new management layer.** This is a **program
   of chained epics** that reference/depend on one another via `Blocked-by` /
   `Ref`. There is no new grouping layer above the epic model. The umbrella
   lives in this spec's architecture plus the dependency chain between epics.

4. **Chained rollup threads the program.** Per the bookend convention, each
   epic closes only after its follow-on-brainstorm task has stood up the next
   brick's epic. The program advances itself; nothing needs a separate roadmap
   artifact to stay coherent.

5. **This epic delivers vision + brick 1.** `spec.md` establishes the whole
   program; the implementation deliverable is **brick 1** — the state-map
   foundation. Standing infrastructure (metrics VMs) is deliberately *later*: a
   proven contract earns the infrastructure, not the reverse.

## The brick decomposition (the program map)

Value-first sequence; each brick after this epic is its own follow-on epic.

1. **Brick 1 — state-map foundation (this epic).** The fleet-state JSON
   contract + a collector that correlates sessions ↔ worktrees ↔ GitHub epics +
   the `vrg-fleet` CLI as its first renderer. **MVP hosts are the Mac and local
   Lima VMs** (both readable without a network transport); **cloud-VM
   collection is a droppable later phase that rides the relay pattern** (see
   §"The collector" and the phased path), not a hard dependency of the brick.
2. **Brick 2 — per-VM metrics layer.** Generalize the forge #1661
   Prometheus/Grafana pattern from "the forge" to *any* lab/service VM
   (node/cadvisor + domain metrics). Answers "how busy/idle is each VM."
3. **Brick 3 — the standing platform.** A small number of **boot-with-macOS
   observability VMs**: central Prometheus (federation / remote-write) + Grafana
   rendering *both* planes. Long-horizon: offload this moving-parts
   infrastructure to a home Mac-mini cluster or cloud, with the laptop as the UI.
4. **Brick 4 — GitHub/forge activity ingestion.** Builds, PRs, CI as
   events/metrics fleet-wide — the #1661 "pipeline layer" seam generalized.
5. **Later (own tracks).** The Tauri GUI (#1532) *links to / embeds* this
   surface — answering that doc's open question: yes, the multi-agent/fleet view
   is a **separate observability surface** the app merely links to, not
   something the GUI re-homes. Forgejo migration (#1653/#1521) proceeds
   independently; when it lands, the forge becomes one more monitored host.

Everything below this line specifies **brick 1**.

## Brick 1 — the state-map foundation

### The entity model

Three empirical facts about the environment ground the model; each was verified
during the brainstorm:

- **Claude sessions** live at `~/.claude/projects/<cwd-slug>/<uuid>.jsonl`. Each
  record may carry `cwd` and `gitBranch`; a `custom-title` record carries the
  session's title.
- **Worktrees** live at `<repo>/.worktrees/issue-<N>-<slug>/` on branch
  `feature/<N>-<slug>`. The issue number in the name is the **most reliable
  epic linkage available today**: `worktree → issue → parent epic`.
- **GitHub** holds the epics and their child tasks/PRs/CI in each org's
  `.github` (or a private repo that self-homes).

The correlated entity graph:

```text
host ──┬── repo ──┬── worktree (branch, issue?, dirty?, ahead/behind?)
       │          └── session  (title, lineage, cwd, branch, last-active, age)
       │
GitHub ┴── epic ──── task/issue ──── PR ──── CI status
```

The join that turns raw facts into the map:

- `session.cwd` → repo; `session.branch` and `worktree.branch` →
  `feature/<N>-…` → **issue N** → its **parent epic**.
- `session.title` of the form `epic-<N>-<slug>` → **epic N** directly (a
  secondary, weaker key — titles are unevenly applied, so the branch/worktree
  path is primary and the title corroborates).
- GitHub `epic → tasks → branches` closes the loop from the other side, so an
  epic with no local worktree/session is still shown (visible-but-idle), and a
  local branch with no epic is flagged (orphan/ad-hoc).

### Correctness constraints (proven during brainstorm, non-negotiable)

1. **Authoritative session title = the *last* `custom-title` record, not the
   first.** Claude Code appends a new `custom-title` record on essentially every
   message; a session reused across a rename carries the *old* title in early
   records and the *current* title only near EOF. A naive first-N-lines read
   reports **stale** titles. In the prototype, switching to a tail/reverse-scan
   read **tripled** epic-session visibility (4 → 11) and corrected the live
   board. The collector MUST read the authoritative (last) title, efficiently
   (tail/reverse-scan — these JSONL files reach multiple MB).

2. **Session lineage is free provenance.** The in-file transition
   (`vergil-user:NN → epic-<N>`) records the exact point a generic window became
   an epic session. The contract captures this lineage (ordered title history
   with first-seen positions); it is the seed for the future archival /
   "Mem-Palace" layer and must not be discarded.

3. **Host attribution for shared-home sessions is resolved positively or marked
   ambiguous — never guessed.** A local Lima VM can share `~/.claude` *and*
   identical `cwd` paths (`/Users/pmoore/…`) with the Mac, so a session read
   from the Mac's `~/.claude` may actually have run *inside the VM*. `uuid`
   de-duplication stops double-counting, but "which host" is undecidable from
   the shared file alone. The collector resolves `host` from a **positive VM
   signal** (a marker the VM writes into its session/environment, or the
   platform axis recorded in the session) and, absent one, sets
   `host: "ambiguous"` rather
   than attributing to the Mac by default. This is a named correctness
   constraint on par with the tail-read.

4. **Never lie; never silently swallow.** Per repo policy, a host that cannot be
   reached, a file that cannot be parsed, or a GitHub query that fails is
   surfaced as an **explicit error/unknown state in the contract and the
   render** — never omitted, never rendered as "nothing here." A silently
   dropped host reads as "no work there," which is the exact failure mode
   (observability that lies) the platform exists to prevent.

### The fleet-state JSON contract

The canonical, **versioned** artifact. Shape (illustrative; field-level schema
finalized in the plan):

```json
{
  "contract_version": "1.0",
  "generated_at": "<ISO-8601>",
  "generator": "vrg-fleet/<version>",
  "hosts": [
    {
      "id": "mac",
      "kind": "physical-host | local-vm | cloud-vm",
      "name": "…",
      "reachable": true,
      "claude_projects_root": "/…/.claude/projects",
      "collected_at": "<ISO-8601>",
      "error": null
    }
  ],
  "sessions": [
    {
      "host": "mac",
      "uuid": "…",
      "project_slug": "…",
      "cwd": "/…/vergil-tooling",
      "repo": "vergil-project/vergil-tooling",
      "branch": "develop",
      "title": "epic-79-observability-extraction",
      "title_lineage": [
        { "title": "vergil-user:01:…", "first_seen": 1 },
        { "title": "epic-79-observability-extraction", "first_seen": 689 }
      ],
      "last_active": "<ISO-8601>",
      "age_seconds": 3200,
      "epic": 79
    }
  ],
  "worktrees": [
    {
      "host": "mac",
      "repo": "vergil-project/vergil-tooling",
      "path": "…/.worktrees/issue-687-…",
      "branch": "feature/687-…",
      "issue": 687,
      "epic": 79,
      "dirty": false,
      "ahead": 0,
      "behind": 0
    }
  ],
  "github": {
    "epics": [
      {
        "org": "vergil-project",
        "number": 79,
        "title": "…",
        "state": "open",
        "tasks": [ { "number": 687, "repo": "…", "state": "open", "pr": null } ],
        "open_prs": [ { "number": 2458, "state": "open", "ci": "passing" } ]
      }
    ]
  }
}
```

Design commitments for the contract:

- **Versioned** (`contract_version`) so consumers can evolve independently.
- **Raw facts + confidently-derived joins.** `sessions`/`worktrees` carry an
  `epic` field only when it resolves unambiguously; otherwise `null`, never a
  guess. Ambiguity is data, not something to paper over.
- **Host-tagged throughout**, so the same contract describes the Mac and every
  VM uniformly — the mechanism that lets a VM (local or cloud) slot in without a
  schema change. A session's `host` is a host `id` **or the sentinel
  `"ambiguous"`** (constraint 3) — never a defaulted guess.
- **Additive-extensible** for later planes: bricks 2–4 add top-level keys
  (e.g. `metrics`, `activity`) rather than reshaping these.

### The collector

A **host is a data source** that yields `(sessions, worktrees, git-state)`. This
single abstraction is the future-proofing: the correlation and rendering layers
never know or care whether a host is the Mac or a VM.

- **Mac (local).** Read `~/.claude/projects` directly; enumerate `.worktrees`
  under the known `dev/projects/<org>/<repo>` tree; read git state locally.
  Enumeration is bounded to `.worktrees/` and does **not** descend into `.git/`,
  build, or validation-output directories.
- **Local Lima VMs (MVP).** Enumerate via `limactl list`; read each VM's data
  over `limactl shell`, running the *same* collector logic. A local VM whose
  home is shared with the Mac already surfaces via the Mac's `~/.claude`; the
  collector **de-duplicates by session `uuid`** and applies the host-attribution
  rule (constraint 3) so a shared session is counted once and attributed
  correctly or marked `ambiguous`.
- **Cloud VMs (droppable later phase — rides the relay, not live SSH).** A cloud
  VM is frequently **stopped** exactly when the map is needed (e.g. post-reboot,
  to save cost), so live SSH is the wrong primary transport. Instead, cloud
  fleet-state is delivered **cloud→Mac over a relay ref**, reusing the
  **GitHubTransport pattern epic #148 already shipped** — the state rides GitHub
  and is readable with the VM powered off. Live SSH collection, if added, is a
  secondary path for a running VM. This phase is **cut-able from brick 1**
  without losing the Mac+Lima value.
- **GitHub — and the fallback source of truth.** Pull open epics and their
  tasks/PRs/CI per org `.github` (and self-homed private repos) via `vrg-gh`.
  This shows epics *active in GitHub but idle locally*, flags *local branches
  with no epic*, and — critically — is the **fallback truth for any unreachable
  host**: an off VM's *pushed* branches/PRs still appear via GitHub even when
  its local session/worktree data cannot be read. Epic resolution
  (`feature/<N>-slug → issue N → parent epic`) uses a **single batched GraphQL
  query** resolving many issues→parents at once (not one round-trip per branch)
  plus a short-TTL cache; with no GitHub reachable, `epic` degrades to `null`
  shown as *unresolved*, never dropped.
- Any host that is genuinely unreachable with no relay/GitHub coverage is a
  **`reachable:false` host with an explicit error**, never a silent omission.

The collector is **read-only** with respect to the environment: it observes,
never mutates sessions, worktrees, or GitHub.

### The `vrg-fleet` CLI — the contract's first renderer

- **Default (human) render:** a recency-sorted tree,
  `org/project → repo → epic/issue → in-flight work (worktree + branch +
  session, with host + age)`. Idle-but-open epics and orphan branches are shown
  and labelled. Unreachable hosts render as explicit fault rows.
- **`--json`:** emits the raw contract to stdout — the load-bearing seam. Every
  later consumer (Grafana feeder, GUI, archival) reads this, not the tree.
- **Scoping flags** (exact set finalized in the plan): filter by org/project, by
  host, by recency window; a "live only" view for the post-reboot reconstruction
  use-case.
- **Fail-loud:** any host/collector error is visible in both renders; `vrg-fleet`
  exits non-zero if it could not collect a requested host, so scripts and CI
  cannot mistake a partial snapshot for a complete one.

## Prior-art reconciliation

- **#1661 (forge observability)** proves the metrics *instrument*
  (Prometheus/Grafana, provisioned-as-code, fail-loud, layered dashboard) and
  **explicitly defers the fleet-wide roll-up** as "named, not designed." This
  program builds that deferred roll-up. Brick 2 reuses #1661's pattern; brick 3
  is the roll-up it named. Nothing here contradicts #1661 — it is its sequel.
- **#1532 (GUI vision)** settled thin orchestration over `vrg-*` and asked
  whether the multi-agent view belongs *in* the app or is a *separate surface
  it links to*. This program answers: **separate surface.** The GUI becomes a
  consumer of the fleet-state contract, not its owner.
- **#1653/#1521 (forge host / abstraction)** run their own track; when the forge
  lands it is simply one more monitored host in the contract.

## Reference mined — `majiayu000/harness`

A solo-dev parallel-agent orchestrator (Rust + Postgres + OTel). We steal
*ideas*, not the engine — the way the VM architecture borrowed from Morrell's
Corral:

- **Domain model** `Thread(session; fork/resume/compact) → Turn → Task(GitHub
  issue/PR) → ExecPlan` is isomorphic to our `session → epic → task → spec`.
  Independent convergence validates our entity model.
- **OTLP / OpenTelemetry emission**, modeling agent work as **traces/spans**
  (a session is a trace; turns are spans) — a better fit for "what are the
  agents doing over time" than gauges. This opens a genuine design fork for the
  **metrics plane** (brick 2/3): Prometheus-scrape vs OTLP-push vs an OTel
  Collector doing both. *Flagged as an open question for the brick-2 epic; not
  decided here.*
- **`/api/dashboard` aggregation endpoint** validates the contract-first
  posture.
- **Signal-driven GC** (detecting chronic patterns — stalled epic, idle VM,
  recurring warning) names a future **analysis layer** above raw observability,
  tying into the archival/Mem-Palace idea.

## Cross-cutting concerns

- **No silent failures / no lying** (repo policy, restated because for an
  observability tool it is the *product*): unreachable host, unparseable file,
  failed query → explicit fault in contract and render, non-zero exit. Never an
  empty section standing in for "couldn't look."
- **Portability.** Host-side logic runs on macOS; remote collection runs against
  Linux VMs. Scripts must be macOS+Linux clean and shellcheck-clean per repo
  constraints.
- **Secrets.** GitHub access uses the existing `vrg-gh` credential path; VM
  reach uses existing `limactl`/SSH config. The collector introduces no new
  secret material and commits none.
- **Read-only.** The collector never mutates the environment it observes.
- **Testing/validation.** `vrg-container-run -- vrg-validate` remains the only
  validation command. The tail-read correctness property, the join logic, and
  the fail-loud behavior get unit tests to the repo's coverage bar, with
  fixture session JSONL (including a renamed/lineage case and a multi-MB
  tail case).

## Phased path (brick 1)

- **Phase 0 — contract + Mac collector + tree.** Define the versioned contract;
  collect the Mac's sessions/worktrees/git-state with the correct tail-read;
  render the tree and `--json`. Goal: `vrg-fleet` reproduces (correctly) the
  live board the corrected prototype produced.
- **Phase 1 — GitHub correlation.** Join local facts to GitHub epics/tasks/PRs;
  show idle-but-open epics and orphan branches; resolve the `epic` field.
- **Phase 2 — local Lima hosts (completes the MVP).** Enumerate via `limactl
  list`; collect each local VM over `limactl shell`; de-duplicate shared-home
  sessions by `uuid` and apply the host-attribution rule (`ambiguous` fallback);
  render stopped VMs as faults. **This is the MVP ceiling — brick 1 is shippable
  and valuable here.**
- **Phase 3 — cloud hosts via relay (droppable).** Deliver cloud fleet-state
  cloud→Mac over a relay ref (reusing #148's GitHubTransport) so an off VM still
  appears. Cut this phase if the relay/registry isn't ready without blocking the
  brick.
- **Phase 4 — polish.** Scoping flags, the "live only" reconstruction view,
  the active-recency threshold + default, non-zero-exit-on-partial, docs.

## Out of scope (this epic)

Bricks 2–4 (per-VM metrics, standing platform, activity ingestion), the GUI
(#1532), Forgejo migration (#1653/#1521), and the archival/Mem-Palace analysis
layer. Each is a follow-on epic or its own track. **If brick 1's cloud phase
(Phase 3) is cut**, cloud-VM collection becomes a small follow-on task/epic
riding the same relay — the Mac+Lima MVP still ships. Brick 1 is designed so
each later piece slots in additively (host abstraction, versioned additive
contract, captured lineage).

## Open questions (settle in plan or defer to the named brick)

- **Positive VM signal for host attribution:** what exact marker distinguishes
  a shared-home session that ran *inside* a local VM from one that ran on the
  Mac — a file the VM writes, the platform axis in the session records, or
  something else? Needed to keep constraint 3 from over-marking `ambiguous`.
  *Settle in the plan.*
- **Local collector presence in Lima VMs:** running the collector over `limactl
  shell` assumes the tool is present in the VM image; confirm, or fall back to
  reading raw files over the transport. *Confirm in the plan.*
- **Cloud relay producer:** what writes the cloud-VM fleet-state relay ref, and
  is the cloud-VM registry enumerable on the Mac side? (Rides #148's transport;
  cloud phase is droppable if not ready.) *Confirm in the plan.*
- **Metrics-plane transport (Prometheus-scrape vs OTLP-push vs OTel Collector):**
  *deferred to the brick-2 epic; recorded here so the fork is not forgotten.*

## Success criteria

1. From the Mac, `vrg-fleet` renders a recency-sorted tree of all in-flight
   work — the post-reboot "what was I doing, and where" view — with **correct
   session titles** (last-record semantics), including epic-named sessions the
   naive reader missed.
2. `vrg-fleet --json` emits the versioned contract; a second tool could consume
   it without re-deriving anything.
3. Each session/worktree shows its resolved epic where unambiguous, `null`
   (never a guess) otherwise; idle-but-open epics and orphan branches are both
   visible and labelled.
4. **Local Lima VMs appear as first-class hosts (MVP ceiling).** A stopped VM
   renders as an explicit fault (or, where a relay/GitHub covers it, as
   idle-with-pushed-work), and `vrg-fleet` exits non-zero on a partial snapshot
   — it never silently omits a host. Cloud-host collection is a droppable phase.
5. Sessions shared between a local VM and the Mac are counted once (`uuid`
   de-dup) and attributed by the host-attribution rule — resolved to a host or
   marked `ambiguous`, never defaulted to the Mac.
6. Session lineage (the `vergil-user:NN → epic-N` rename point) is captured in
   the contract.
7. The whole surface is documented in the site docs and passes `vrg-validate`.
