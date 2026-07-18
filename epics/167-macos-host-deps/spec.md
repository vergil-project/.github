# macOS host dependency management (`vrg-host`)

- **Epic:** vergil-project/.github#167
- **Status:** Design drafted (2026-07-18) via superpowers brainstorming. Pending
  paad pushback and human review.
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
    warning" — the motivating incident; absorbed as a `custom` manifest entry
    (§7.3, §12).
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
from a tooling-owned manifest, (b) **reports** installed-vs-latest gaps with
explicit severity, (c) **upgrades** them selectively or wholesale, and (d)
**pins** individual components when a release breaks our tooling — inheriting
#155's pin-as-technical-debt lifecycle. Proactive scheduling/reminders are
deliberately out of scope and handed to a follow-on epic (§11).

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
over Virgil-critical macOS host dependencies: enumerate, report the gap, upgrade,
and pin-as-debt.

**Non-goals (this epic).**

- **Proactive scheduling / reminders** — deferred to the follow-on epic (§11).
- **Cross-platform host management** — macOS only; Linux/other hosts are #909's
  concern, not this epic's.
- **Managing personal, non-Virgil tools** (e.g. Superwhisper) — out of scope by
  definition (§4).
- **Report-only upstream-security alerting (Dependabot-equivalent)** — as in
  #155, this is a plausible later concern, not this epic.

## 4. Scope

- **In:** Virgil-critical host tooling installed on top of a base macOS install.
  The enumeration task produces the authoritative list; the working set includes
  at least: gcloud, Docker (Desktop/engine), Lima/virtualization, uv, gh,
  shellcheck, hadolint, yamllint, actionlint, and the `vrg-*` tools themselves.
- **Out:** anything not required for the Virgil ecosystem — personal productivity
  tools, editors, voice-to-text (Superwhisper), etc. A dependency earns a manifest
  entry only if Virgil development needs it on the host.

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
host (not in a container), alongside the other host `vrg-*` tools.

- **`vrg-host status`** *(a.k.a. the #896 audit)* — the read-only report. For each
  manifest dependency: install method, installed version, latest version, the gap,
  severity, and pin state. Degrades explicitly (§8). No side effects. Default
  command when run bare.
- **`vrg-host upgrade [<dep>… | --all]`** — remediate. Upgrade the named
  dependencies, or everything, via each dependency's method handler. Respects pins
  (§9): will not move a pinned dependency past its constraint without `unpin` (or
  an explicit override flag).
- **`vrg-host pin <dep> --constraint <c> --inducing-release <v> --reason <text>
  [--tracking-issue <url>] [--deterministic]`** — record a pin. **`--reason` is
  required**; the command refuses a pin without one.
- **`vrg-host unpin <dep>`** — free a pin (state → freed).

## 7. Dependency manifest & descriptor model

**Descriptor model (decided): method-handlers + declarative manifest.**

### 7.1 The manifest

A tooling-owned, declarative data file shipped inside `vergil-tooling` (format —
YAML/TOML — settled at plan time; it is the single source of truth for *what the
host requires*). It is **not** per-repo config: host requirements are fleet-wide.
Each entry is a dependency descriptor.

### 7.2 Descriptor fields (per dependency)

- `name` — canonical tool name.
- `why` — one line: why Virgil needs it on the host (justifies inclusion; keeps
  the list honest per §4).
- `method` — the install-method handler: one of `brew-formula`, `brew-cask`,
  `uv-tool`, `custom`.
- `probe` fields for a `custom` entry only: explicit commands for
  `installed_version`, `latest_version`, and `upgrade`. Non-custom methods derive
  these from the handler.
- optional pin block (§9) when the tool is currently held back.

### 7.3 Method handlers

A small, closed set of handlers the tool ships; adding a common tool is a
one-line manifest entry, not code:

- **`brew-formula` / `brew-cask`** — installed via `brew list --versions`; latest
  via `brew info --json` / `brew outdated`; upgrade via `brew upgrade`.
- **`uv-tool`** — for tools installed as `uv tool install` (e.g. vergil-tooling
  itself); installed/latest/upgrade via uv.
- **`custom`** — the escape hatch for heterogeneous tools whose lifecycle no
  standard handler covers (gcloud and its self-updating components; standalone
  installers). The manifest entry names explicit shell commands for
  installed/latest/upgrade. gcloud — including the numpy-in-its-virtualenv
  requirement from #166 — is expressed here.

An unrecognized `method` is a manifest error and must fail loudly (no silent
fallthrough), consistent with the tooling-wide no-silent-failure rule.

## 8. Reporting: gap, severity, and no-silent-failure

`vrg-host status` classifies each dependency:

| State | Meaning | Severity |
|---|---|---|
| missing | required tool not installed / not on PATH | **ERROR** |
| current | installed == latest (or == pin constraint) | OK |
| behind | installed < latest, unpinned | WARN (drift) |
| pinned | held at a constraint below latest | INFO (tracked debt) |
| latest-unknown | latest could not be determined | WARN |

- **Explicit degradation.** When a latest-version probe cannot answer (offline,
  upstream API drift, rate-limit), `status` reports `latest: unknown (<reason>)`
  and classifies the dependency **latest-unknown** — it never silently treats
  installed as current. An undetermined gap is surfaced, not hidden.
- **Exit code = worst severity:** `0` all-OK, `1` warnings present (drift /
  unknown / pinned), `2` errors present (missing / broken). Matches #896's
  proposed model and makes `status` usable as a check.

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

## 10. Enumeration (the first substantive task)

Curating the authoritative dependency list is real work and its own task (it
absorbs the enumeration half of #896). For each candidate it must record: is it
Virgil-critical (§4 in/out test)? how is it installed on this host today (which
`method`)? how do we read its installed version, discover latest, and upgrade it?
The `custom` cases (notably gcloud) are where the effort concentrates. Output is
the seeded manifest (§7) with a descriptor per included tool.

## 11. Punted: scheduling / reminders → follow-on epic

Deliberately out of scope. The host has no `vrg-vm`-style chokepoint to hang a
nudge on, and macOS has no scheduling mechanism we have settled on — a strategic
problem larger than this audit (for code-resident recurring tasks a GitHub Action
would suffice; the host cannot use one). The closing follow-on-brainstorm bookend
(#169) reviews what shipped and brainstorms the macOS / human-environment
scheduling problem into the next epic. This epic ships the tool; you run it
manually until that epic delivers the reminder.

## 12. Disposition of existing issues

- **#896** — re-parented onto this epic (done). Its scope becomes the §10
  enumeration + §6 `status` work in the plan.
- **#166** — absorbed: gcloud's numpy requirement becomes a `custom` manifest
  entry whose `installed_version` probe detects the missing/ present numpy in
  gcloud's interpreter, so `status` would flag it and `upgrade` would remediate
  it. #166 is linked under this epic and closes as superseded once the entry
  exists. (The manual fix already applied on the primary host stands in the
  interim.)
- **#909** — referenced, not re-scoped; cross-platform host support is a separate
  concern.

## 13. Bookend tasks

Per the epic-create convention, this epic carries:

- **#168** (docs) — this spec + the plan; published into `.github`.
- **#169** (follow-on brainstorm) — the §11 scheduling problem → next epic.
- **vergil-tooling#2450** (documentation review) — final gate; a multi-repo sweep
  ensuring `vrg-host` and the host-pinning workflow are reflected in the human
  docs (especially the versioned site docs), spawning per-repo doc tasks as
  needed.

## 14. Testing considerations

- **Method handlers** are unit-testable against captured `brew`/`uv`/`custom`
  command output (installed/latest parsing, upgrade command construction).
- **`status` classification and exit codes** are testable with a fabricated
  manifest and stubbed probes, including the **latest-unknown** degradation path
  (a probe that fails must yield WARN + `unknown`, never a silent OK).
- **Pin semantics** — `upgrade` refusing to cross a pin, `pin` refusing an empty
  reason, and pin age/gap rendering in `status` — are testable without touching a
  real host.
- **`custom`/gcloud** probes are integration-tested where feasible and otherwise
  exercised against recorded fixtures; the numpy (#166) case is a concrete
  fixture.
