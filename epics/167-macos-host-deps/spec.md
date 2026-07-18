# macOS host dependency management (`vrg-host`)

- **Epic:** vergil-project/.github#167
- **Status:** Design drafted (2026-07-18) via superpowers brainstorming; hardened
  via paad pushback the same day (4 findings resolved: observe-only upgrade
  capability, auxiliary requirement checks, split latest states, trust/privilege
  invariants). Pending human review.
- **Repo:** vergil-project/vergil-tooling (member); epic homed in `.github`.
- **Related work:**
  - vergil-project/.github#155 "Container version pinning & floating-dependency
    management" — the **canonical pinning doctrine**; this epic inherits its
    philosophy, pin schema, and re-evaluation lifecycle wholesale and applies them
    to the macOS host (§5, §9).
  - vergil-project/vergil-tooling#896 "add `vrg-audit-host` command to check local
    macOS configuration" — the original request; brought forward under this epic
    (re-parented from ad-hoc epic #99). Seeds the enumeration + `status` work.
  - vergil-project/.github#166 "Durably silence gcloud IAP 'install NumPy'
    warning" — the motivating incident; absorbed as a gcloud **auxiliary
    requirement check** (§7.4, §13).
  - vergil-project/vergil-tooling#909 "Multi-platform host support: POSIX hook
    rewrite and platform documentation" — adjacent (host portability); not in
    scope here.
- **Brainstorm source:** superpowers brainstorming session, 2026-07-18.

## 1. Summary

We aggressively manage dependencies inside dev containers (#155) and inside VMs,
but the macOS development **host** is unmanaged. The host must carry the tooling
that cannot live in a container or VM — gcloud, Docker, Lima/virtualization, uv,
gh, the linters — yet we have no way to see what is installed, how far behind
leading edge it is, or to keep it current. This epic delivers `vrg-host`: a
command-line tool that (a) **enumerates** the Virgil-critical host dependencies
from a tooling-owned manifest, (b) **reports** installed-vs-latest gaps and
per-dependency health with explicit severity, (c) **upgrades** the ones that can
be upgraded automatically (selectively or wholesale), and (d) **pins** individual
components when a release breaks our tooling — inheriting #155's
pin-as-technical-debt lifecycle. Proactive scheduling/reminders are deliberately
out of scope and handed to a follow-on epic (§12).

## 2. Problem / motivation

- **The host is the one unmanaged tier.** Containers and VMs are rebuilt from
  declarative sources; the host accretes tools installed by hand over time with no
  inventory and no drift signal.
- **Drift is silent until it breaks something.** The gcloud/NumPy incident (#166)
  is the archetype: a locally-installed dependency (numpy in gcloud's managed
  virtualenv) was silently missing, degraded the IAP tunnel, printed a warning on
  every VM login, and was fixed by hand — a fix that is itself not durable and
  that no tool would have caught or would notice regressing.
- **No single entry point.** Unlike `vrg-vm` (which nudges on every run), the
  host has no chokepoint that could surface staleness, and the tools are too
  heterogeneous to wrap uniformly.
- **Pinning without management is a permanent freeze.** The same failure #155
  identified for containers applies to the host: when we hold a host tool back
  (a release breaks us), that hold must be tracked, justified, and released — not
  left to rot.

## 3. Goal & non-goals

**Goal.** A `vrg-host` command giving command-line observability and management
over Virgil-critical macOS host dependencies: enumerate, report the gap and
health, upgrade what can be upgraded, and pin-as-debt.

**Non-goals (this epic).**

- **Proactive scheduling / reminders** — deferred to the follow-on epic (§12).
- **Cross-platform host management** — macOS only; Linux/other hosts are #909's
  concern, not this epic's.
- **Migrating the container runtime off Docker Desktop** — a headless/OSS runtime
  (Colima, podman, docker-engine over Lima) would make Docker `auto`-upgradeable
  and drop the macOS GUI dependency (§4), but that is a separate dev-environment
  decision, not this epic's work. It is a designated topic for this epic's closing
  follow-on brainstorm (#169, §12).
- **Managing personal, non-Virgil tools** (e.g. Superwhisper) — out of scope by
  definition (§4).
- **Report-only upstream-security alerting (Dependabot-equivalent)** — as in
  #155, a plausible later concern, not this epic.

## 4. Scope

- **In:** Virgil-critical host tooling installed on top of a base macOS install.
  The enumeration task produces the authoritative list; the working set includes
  at least: gcloud, Docker (engine/Desktop), Lima/virtualization, uv, gh,
  shellcheck, hadolint, yamllint, actionlint, and the `vrg-*` tools themselves.
  **macOS itself** is in scope for *reporting* (surface a pending OS update) but is
  `manual`-upgrade only (§6) — `vrg-host` never applies a system update.
- **Out:** anything not required for the Virgil ecosystem — personal productivity
  tools, editors, voice-to-text (Superwhisper), etc. A dependency earns a manifest
  entry only if Virgil development needs it on the host.

**Docker note.** Docker Desktop is a GUI cask and therefore `manual` today. This
is an artifact of the Docker Desktop choice, not of Docker: migrating to a
headless/OSS runtime (Colima / podman / docker-engine over the Lima we already
run) would remove the GUI dependency and let Docker become `auto`. That migration
is out of scope here (§3) and is a designated topic for the closing follow-on
brainstorm (#169); the manifest records Docker's reality as it is today.

## 5. Pinning philosophy — inherited from #155

This epic does **not** define a new pinning model. It adopts vergil-project/
.github#155's doctrine as the single, org-wide standard for managing pins as
technical debt, applied to host tools. Restated for reference (canonical text in
#155 §3–§4):

1. **Default is unpinned** — float on each tool's leading edge.
2. **A pin is a reaction, never a default** — a reaction to a release breaking us.
3. **Every pin carries a written justification.**
4. **Least-specific pin that solves the problem** (pin `1.x`, not the exact patch).
5. **When in doubt, set it free** — unpin, see what breaks, pin only the culprit
   at the least-specific working constraint.
6. **A pin is anchored to the inducing release and carries a re-evaluation
   trigger — never permanence.** The pin is valid only while its `inducing_release`
   is the leading edge; when the leading edge moves past it, the pin is
   automatically due for re-evaluation.

Pin **states** (`active` / `under-evaluation` / `freed`) and the **re-evaluation
algorithm** (deterministic pin → re-test then free-or-re-anchor; non-deterministic
pin → suppress-and-observe under a tracking issue) are inherited verbatim from
#155 §4. Host-specific mechanics are in §9.

## 6. The tool: `vrg-host`

A single host-side console script (`vrg-host`) with subcommands. It runs on the
host (not in a container), alongside the other host `vrg-*` tools, and **runs
unprivileged** (§10).

- **`vrg-host status`** *(a.k.a. the #896 audit)* — the read-only report. For each
  manifest dependency: install method, installed version, latest version, the gap,
  the result of any auxiliary requirement checks (§7.4), severity, and pin state.
  Degrades explicitly (§8). No side effects. Default command when run bare.
- **`vrg-host upgrade [<dep>… | --all]`** — remediate. Upgrades only dependencies
  whose descriptor declares `upgrade: auto`, via each dependency's method handler,
  and respects pins (§9): it will not move a pinned dependency past its constraint
  without `unpin` (or an explicit override flag). A `manual` dependency (§7.2) is
  **never** silently skipped: `upgrade` refuses it with an actionable message
  ("macOS: run Software Update"), and `--all` prints the list of `manual`
  dependencies it deliberately did not touch. `vrg-host` never silently escalates
  privilege — if an `auto` upgrade turns out to require `sudo` or a protected
  write, it is reported as needing manual action rather than triggering a surprise
  password prompt (§10).
- **`vrg-host pin <dep> --constraint <c> --inducing-release <v> --reason <text>
  [--tracking-issue <url>] [--deterministic]`** — record a pin. **`--reason` is
  required**; the command refuses a pin without one.
- **`vrg-host unpin <dep>`** — free a pin (state → freed).

## 7. Dependency manifest & descriptor model

**Descriptor model (decided): method-handlers + declarative manifest.**

### 7.1 The manifest

A tooling-owned, declarative data file shipped inside `vergil-tooling` (format —
YAML/TOML — settled at plan time; it is the single source of truth for *what the
host requires*). It is **not** per-repo config and is **never repo-overridable**:
host requirements are fleet-wide, and the manifest (including any `custom` shell
in it) is trusted tooling-authored content — see the security invariant in §10.
Each entry is a dependency descriptor.

### 7.2 Descriptor fields (per dependency)

- `name` — canonical tool name.
- `why` — one line: why Virgil needs it on the host (justifies inclusion; keeps
  the list honest per §4).
- `method` — the install-method handler: one of `brew-formula`, `brew-cask`,
  `uv-tool`, `custom`.
- `upgrade` — capability: `auto` (the tool can be upgraded programmatically and
  safely) or `manual` (report the gap, but `vrg-host upgrade` must not attempt it;
  e.g. macOS, Docker Desktop). Default `auto`; `manual` is the explicit exception.
- `latest` — manageability of the latest-version signal: a probe (for `auto`/known
  feeds) or the explicit marker that latest is **not programmatically knowable**
  for this tool (→ `latest-unmanaged`, §8). Reserves the `latest-unknown` WARN for
  genuine probe *failures*, not for tools that structurally have no feed.
- `probe` fields for a `custom` entry only: explicit commands for
  `installed_version`, `latest_version` (unless `latest` is unmanaged), and
  `upgrade` (unless `manual`). Non-custom methods derive these from the handler.
- `checks` — zero or more **auxiliary requirement checks** (§7.4).
- optional pin block (§9) when the tool is currently held back.

### 7.3 Method handlers

A small, closed set of handlers the tool ships; adding a common tool is a
one-line manifest entry, not code:

- **`brew-formula` / `brew-cask`** — installed via `brew list --versions`; latest
  via `brew info --json` / `brew outdated`; upgrade via `brew upgrade` (a
  `brew-cask` for a GUI app is typically `upgrade: manual`).
- **`uv-tool`** — for tools installed as `uv tool install` (e.g. vergil-tooling
  itself); installed/latest/upgrade via uv.
- **`custom`** — the escape hatch for heterogeneous tools whose lifecycle no
  standard handler covers (gcloud and its self-updating components; standalone
  installers). The manifest entry names explicit commands for the probes it
  supports. An unrecognized `method` is a manifest error and must fail loudly (no
  silent fallthrough), consistent with the tooling-wide no-silent-failure rule.

### 7.4 Auxiliary requirement checks

A dependency's correctness is more than its version. A descriptor may declare zero
or more **auxiliary requirement checks**, each `{name, probe → satisfied /
unsatisfied, remediation}`, evaluated by `status` **alongside** the version gap. A
check is a presence/health assertion, not a version comparison.

This is the axis that genuinely absorbs #166: gcloud carries a check
`numpy-in-interpreter` whose probe imports numpy in gcloud's resolved Python
(`gcloud info --format='value(basic.python_location)'`) and whose remediation is
`<that python> -m pip install numpy`. `status` reports the check as satisfied or
not; an unsatisfied check is an **ERROR** (the dependency is present but
misconfigured). `vrg-host upgrade` (or a dedicated remediation path) can run a
check's remediation when its `upgrade` capability permits.

## 8. Reporting: gap, health, severity, and no-silent-failure

`vrg-host status` classifies each dependency on two axes — its **version state**
and its **auxiliary-check results** — and reports the worst.

Version state:

| State | Meaning | Severity |
|---|---|---|
| missing | required tool not installed / not on PATH | **ERROR** |
| current | installed == latest (or == pin constraint) | OK |
| behind | installed < latest, unpinned | WARN (drift) |
| pinned | held at a constraint below latest | INFO (tracked debt) |
| latest-unmanaged | tool has no programmatic latest feed (declared) | INFO |
| latest-unknown | a latest probe that should work failed this run | WARN |

Auxiliary-check result: each declared check (§7.4) is **satisfied** (OK) or
**unsatisfied** (**ERROR** — present but misconfigured).

- **Explicit degradation, honestly labeled.** A transient probe failure yields
  `latest-unknown` (WARN); a tool that structurally cannot report latest is
  `latest-unmanaged` (INFO) — never conflated, so WARN stays rare and actionable
  and chronic cases don't erode trust. Neither is ever silently treated as
  `current`.
- **Exit code = worst severity across both axes:** `0` all-OK, `1` warnings
  present (drift / latest-unknown), `2` errors present (missing / broken /
  unsatisfied check). Matches #896's proposed model and makes `status` usable as a
  check.

## 9. Pin mechanism (host application of #155 §4)

- **Storage.** A pinned dependency carries a pin block in the manifest with the
  #155 schema: `constraint`, `inducing_release`, `deterministic` (bool), `reason`,
  `state`, `tracking_issue`. The manifest is the host analog of #155's `pins.yml`;
  keeping pins beside their descriptor (rather than in a separate file) is a
  host-side simplification, but the **fields and their semantics are #155's**.
- **A pin is a real hold.** `vrg-host upgrade` treats an unpinned dependency's
  target as *latest* and a pinned dependency's target as *the constraint*; it will
  not cross the constraint without `unpin` or an explicit override.
- **Surfaced as debt.** `status` renders each pin with its constraint, the latest
  it is holding back from (so the size of the debt is visible), the reason, the
  tracking issue, and how long it has stood (age) — old debt reads loud.
- **Justification gate.** A pin with no `reason` is refused at creation
  (`vrg-host pin`), the host analog of #155's "a new pin cannot merge
  undocumented" CI gate.
- **Re-evaluation.** The #155 algorithm applies unchanged: when the leading edge
  moves past `inducing_release`, the pin is due; deterministic pins are re-tested
  (free or re-anchor), non-deterministic pins move to `under-evaluation` with a
  tracking issue. Mechanizing the *trigger* (observability of "leading edge moved
  past inducing release") is desirable but may land in a later task or the
  follow-on epic; the *process* is fully specified here so nothing rots silently.

## 10. Security & privilege invariants

- **Manifest trust.** The host-dependency manifest is tooling-owned and **never
  repo-overridable**. `custom` probe/upgrade/check commands are trusted,
  tooling-authored content — not user- or repo-supplied input. This is a
  load-bearing security invariant: allowing a consuming repo to override or extend
  the manifest (the way `vergil.toml` overrides other settings) would turn those
  shell strings into an arbitrary-code-execution surface on the host, so any
  future "let repos extend the host manifest" change must be rejected on this
  basis.
- **Unprivileged, no silent escalation.** `vrg-host` runs as the ordinary user and
  never silently invokes `sudo`. An `auto` upgrade (or a check remediation) that
  turns out to require elevation or a protected write is reported as needing manual
  action — effectively demoted to `manual` with an actionable message — rather than
  triggering a surprise password prompt mid-`upgrade --all`.

## 11. Enumeration (the first substantive task)

Curating the authoritative dependency list is real work and its own task (it
absorbs the enumeration half of #896). For each candidate it must record: is it
Virgil-critical (§4 in/out test)? how is it installed on this host today (which
`method`)? is it `auto` or `manual` to upgrade? is its latest programmatically
knowable, and how? what auxiliary requirement checks (§7.4) does it need? The
`custom` cases (notably gcloud) are where the effort concentrates. Output is the
seeded manifest (§7) with a descriptor per included tool.

## 12. Punted: scheduling / reminders → follow-on epic

Deliberately out of scope. The host has no `vrg-vm`-style chokepoint to hang a
nudge on, and macOS has no scheduling mechanism we have settled on — a strategic
problem larger than this audit (for code-resident recurring tasks a GitHub Action
would suffice; the host cannot use one). The closing follow-on-brainstorm bookend
(#169) reviews what shipped and brainstorms the macOS / human-environment
scheduling problem into the next epic. This epic ships the tool; you run it
manually until that epic delivers the reminder. That same closing brainstorm
(#169) also takes up the Docker Desktop → headless/OSS-runtime migration (§4) as a
candidate follow-on epic.

## 13. Disposition of existing issues

- **#896** — re-parented onto this epic (done). Its scope becomes the §11
  enumeration + §6 `status` work in the plan.
- **#166** — absorbed as gcloud's `numpy-in-interpreter` auxiliary requirement
  check (§7.4): `status` flags it when numpy is missing from gcloud's interpreter,
  and the check's remediation reinstalls it. #166 is linked under this epic and
  closes as superseded once the check exists. (The manual fix already applied on
  the primary host stands in the interim.)
- **#909** — referenced, not re-scoped; cross-platform host support is a separate
  concern.

## 14. Bookend tasks

Per the epic-create convention, this epic carries:

- **#168** (docs) — this spec + the plan; published into `.github`.
- **#169** (follow-on brainstorm) — the §12 scheduling problem **and** the Docker
  Desktop → headless/OSS-runtime migration (§4) → next epic(s).
- **vergil-tooling#2450** (documentation review) — final gate; a multi-repo sweep
  ensuring `vrg-host` and the host-pinning workflow are reflected in the human
  docs (especially the versioned site docs), spawning per-repo doc tasks as
  needed.

## 15. Testing considerations

- **Method handlers** are unit-testable against captured `brew`/`uv`/`custom`
  command output (installed/latest parsing, upgrade command construction).
- **`status` classification and exit codes** are testable with a fabricated
  manifest and stubbed probes, including: the **latest-unknown** (probe failure →
  WARN) vs **latest-unmanaged** (declared no-feed → INFO) distinction, and the
  two-axis worst-severity roll-up (version state × auxiliary checks).
- **Auxiliary requirement checks** — satisfied/unsatisfied classification and the
  ERROR roll-up for an unsatisfied check — are testable with stubbed probes; the
  numpy (#166) case is a concrete fixture.
- **Pin semantics** — `upgrade` refusing to cross a pin, `pin` refusing an empty
  reason, pin age/gap rendering — are testable without a real host.
- **Upgrade capability & privilege** — `upgrade` refusing a `manual` dep with an
  actionable message, `--all` listing skipped `manual` deps, and the
  no-silent-escalation path — are testable with stubbed handlers.
- **`custom`/gcloud** probes are integration-tested where feasible and otherwise
  exercised against recorded fixtures.
