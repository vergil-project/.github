# Explicit, purpose-named sessions (deprecate slots + archive)

- **Epic:** vergil-project/.github#230
- **Status:** Design approved 2026-08-06 (brainstormed directly via `epic-create`).
- **Repos:** vergil-project/vergil-tooling (`vrg-vm` implementation — the bulk of the
  work), vergil-project/vergil-vm (docs/site + the two session design specs),
  vergil-project/.github (epic docs). Epic homed in `.github`.
- **Supersedes:** vergil-tooling#2602 (archived sessions not filtered out of
  reconnect selection) — closed as superseded, its requirement folded in (§4.7).
- **Promoted from:** direct brainstorm (no triage origin).

## 1. Summary

`vrg-vm session` today names sessions `identity:slot:workspace` and, by default,
resumes the **most-recently-used** session for a repo. The slot is a meaningless
auto-number and the default *guesses* which session you meant; an entire
**staleness + archive** apparatus (fresh/warn/stale bands, auto-archive at 14 days,
an `archived@` name-marker) exists only to stop that guess from dragging stale
context into new work.

This epic makes reconnect **fully explicit by name** and binds names to **purpose**
(an epic or ad-hoc problem) instead of a slot. Because the system then never
guesses which session you meant, it can never guess *wrong* — so the entire
staleness/archive apparatus is **removed**, not merely bug-fixed. The name becomes
`label:workspace`; identity is dropped (vestigial). Creation is explicit and named
(`--label`); reconnect is exact-name (`--resume`, which already exists).

The deeper payoff, and a **first-class goal**: epic-associated names turn the pile
of persisted transcripts into a **navigable implementation-history archive, keyed
by epic**. "How did we actually build epic #213?" becomes answerable — exactly the
structured, epic-linked corpus a future Mem Palace wants. "Never delete" stops being
hoarding and becomes the point.

## 2. Motivation

### 2.1 How it works today (facts, from the code + specs)

- **Name** = `identity:slot:workspace` (colon-delimited), set via Claude Code's
  `-n/--name` flag on creation. Confirmed mechanism (from
  `docs/specs/2026-05-30-stale-session-lifecycle-design.md`): the name lives inside
  the transcript as append-only `{"type":"agent-name","agentName":…}` entries,
  **last one wins**; a `/rename` is just one appended line; the session id never
  changes. Transcript: `~/.claude/projects/<slug>/<sessionId>.jsonl`; live roster:
  `~/.claude/sessions/<pid>.json`.
- **Default resume** picks the most-recent (originally lowest-idle) slot for the
  repo. `--slot N` targets a slot; **`--resume <NAME>`** already resumes by *exact
  display name* and its help already cites `epic-85-centralize-epics-adhoc`.
- **`--name`** is the **VM instance** name (e.g. `local`/`cloud` in the resiliency
  lab), declared under `[vm.<identity>.instances.<name>]` — **not** a session label.
- **Archive** = a transcript whose current name carries an `archived@` marker,
  excluded from active name resolution. Driven by `--fresh` and the staleness
  bands: fresh (<`session_stale_days`, default 7 — silent resume), warn (7–14 —
  prompt), stale (≥`session_archive_days`, default 14 — auto-archived).
- Implementation: `src/vergil_tooling/bin/vrg_vm.py`, `lib/session.py`,
  `vrg_vm_resolve.py` (vergil-tooling). Docs: `vergil-vm/docs/site/docs/sessions.md`.

### 2.2 What breaks down

1. **Reconnecting to the wrong session.** After a reconnect the title bar showed a
   stale/archived session name — old context silently pulled into new work
   (vergil-tooling#2602). The archive marker excludes from active *resolution*, but
   a second code path (selection/title render) didn't honor it.
2. **Names carry no purpose.** `who:number:where` encodes identity and repo but not
   *what the session is for*. The slot is noise; identity is vestigial (the
   virtual-user/virtual-agent split it encoded did not pan out).

### 2.3 The reframe

The whole fresh/warn/stale/auto-archive machinery exists for **one reason** — to
protect the **auto-resume-most-recent** default from stale context. Make reconnect
**explicit by exact name** and the system never guesses, so it can never guess
wrong: the machinery that catches wrong guesses becomes unnecessary. That is why
"deprecate the archive concept" is coherent rather than reckless — it is a
*consequence* of explicit naming, not an independent gamble.

## 3. Goals / non-goals

**Goals**

1. **Explicit, purpose-named sessions** — create with a required human label;
   reconnect by exact name. No slot game, no most-recent guess.
2. **Delete the archive concept** — remove the `archived@` marker, the staleness
   bands, auto-archive, and the thresholds; replace with a recency display-filter.
3. **Preserve every transcript** and keep it reconnectable by exact name — the
   implementation-history archive.
4. **Fix the #2602 symptom** — the title bar/prompt always renders the session's
   current functional name.

**Non-goals**

- **No epic-state coupling** — a session's visibility tracking live GitHub epic
  open/closed state is deferred (§9).
- **No transcript pruning/deletion** — never-delete stands; Mem Palace indexing is
  separate.
- **No change to VM-instance semantics** — `--name` stays the VM instance.

## 4. Design decisions

### 4.1 Name = `label:workspace`

Colon-delimited, two fields. `label` is the human functional name; `workspace` is
the starting repo (relative to `projects_dir`) that anchors the CLAUDE.md/memory
bootstrap and remains a **required** positional argument. **Identity is removed**
from the name. The colon delimiter is retained (colons don't appear in labels or
workspace paths), so parsing stays unambiguous.

`label` is **soft-validated**: it must be a clean slug (so `label:workspace` always
parses and renders — a `:` or whitespace is rejected), and the tool **warns but
does not reject** if it does not start with `epic-` or `adhoc-`. This preserves the
real workflow: start with a bare slug, then rename to `epic-<N>-<slug>` once the
epic number exists.

### 4.2 Create (`--label`) and attach (`--resume`) are distinct verbs

- **`--label <name>`** (new flag; `--name` is taken by the VM instance) creates a
  new named session `label:workspace`. It **must not already exist** (§4.3 names are
  unique), so it can never clobber. Soft-warns on a non-`epic-`/`adhoc-` label.
- **`--resume <name>`** (already exists) attaches an existing session by exact name;
  it **errors if absent**.

Keeping create (must-not-exist) and attach (must-exist) as separate verbs is
deliberately **typo-safe**: a mistyped name on `--resume` errors instead of silently
creating a new empty session, and a mistyped `--label` that collides errors instead
of resuming the wrong thing. This is the concrete form of the "full explicit"
choice — you always state whether you are starting or resuming.

### 4.3 Names are unique; `--slot` and auto-resume-most-recent are removed

A label identifies exactly one visible session, so exact-name reconnect is never
ambiguous. The default "resume most-recent slot" behavior and the `--slot` flag are
**removed** — there is no slot concept. Running `vrg-vm session <workspace>` with no
`--label`/`--resume` no longer silently resumes; it must direct the user to create
(`--label`) or attach (`--resume`). (Exact no-arg behavior — error with guidance vs.
list-and-choose — is settled in the plan; default recommendation: print the recency
list and the two verbs.)

### 4.4 Archive deleted → recency display-filter

Remove the `archived@` marker, the fresh/warn/stale bands, auto-archiving, and the
`session_stale_days` / `session_archive_days` thresholds. Replace them with a pure
**display filter** on `list --sessions`: default to recently-active sessions;
`--all` reveals the rest. This is display-only — no name mutation, nothing moved —
so there is no archive-vs-active name state to mis-filter, which removes #2602's
entire bug class. Every transcript stays on disk and reconnectable by exact name.

`list` flags change accordingly: `--active`/`--idle` stay; **`--archived` is
removed**; `--all` changes meaning from "include archived" to "include
older-than-recent". A `session_recent_days` display threshold (cascading like the
old ones) governs the default window.

### 4.5 Rename, `--fresh`, `--fork`, and the fork guardrail

- **Rename** stays trivial (append one `agent-name` line; id stable) — the slug →
  `epic-<N>-<slug>` path. Exposed via Claude's native `/rename` and, for scripting,
  a `vrg-vm` rename verb.
- **`--fresh`** restarts a name cleanly: it renames the prior same-named session
  with a **retired suffix** (e.g. `epic-213-x~<datestamp>` — still reconnectable,
  never deleted) and hands the clean `label` to a new session. Same append-a-line
  mechanism; **not** a resurrected archive state — just a disambiguating rename that
  preserves name-uniqueness among visible sessions.
- **`--fork`** (branch a busy session into a new name) and the **no-double-attach
  guardrail** (refuse a second live client on one session) are **kept** unchanged.

### 4.6 Migration is minimal

No forced rename, no purge (transcripts are the archive). Legacy
`identity:slot:workspace` sessions and old `archived@`-marked sessions persist as
**opaque names**: still reconnectable by their full exact name, and hidden from the
default `list` by the recency filter once idle long enough. Attach logic treats a
name as an opaque string, so legacy names keep working without a parser for the old
scheme. **Optional** one-time cosmetic pass: strip `archived@` markers so the list
renders uniformly (purely cosmetic — it changes only how old names display).

### 4.7 Title-bar / prompt correctness (folds in #2602)

The terminal title bar and in-session prompt must always render the session's
**current** functional name (`label:workspace`), never a stale or archived label.
This is the concrete #2602 requirement; with the archive state gone, "current name"
is simply the last `agent-name` entry, and both the title render and the selection
path must read it from the same source of truth.

## 5. Command surface (before → after)

| | Before | After |
|---|---|---|
| Create | implicit (auto-slot) then `/rename` | **`--label <name>`** (required, must-not-exist) |
| Reconnect | default most-recent, or `--resume <name>` | **`--resume <name>`** (must-exist) — primary |
| Slots | `--slot N`, auto most-recent | **removed** |
| Fresh start | `--fresh` (archives old) | `--fresh` (retire-suffix rename of old) |
| Fork | `--fork` | `--fork` (unchanged) |
| VM instance | `--name` | `--name` (unchanged) |
| List | `--active/--idle/--archived/--all` | `--active/--idle/--all` (recency); `--archived` removed |
| Staleness config | `session_stale_days`, `session_archive_days` | removed; `session_recent_days` (display window) |

## 6. Rollout — independently shippable stages

- **Stage A — Named create + exact-name primary reconnect.** Add `--label` (create,
  unique, soft-warn); make `--resume` the primary reconnect; remove `--slot` and the
  auto-resume-most-recent default; drop identity from the name (`label:workspace`).
  Tests for create/attach/uniqueness/soft-warn.
- **Stage B — Delete archive, add recency filter.** Remove the `archived@` marker,
  staleness bands, auto-archive, thresholds; add the `list` recency filter +
  `session_recent_days`; retire `--archived`; re-point `--all`. Reconceive `--fresh`
  as retire-suffix rename. Tests.
- **Stage C — Title-bar/prompt correctness (#2602).** Ensure the rendered name reads
  the current `agent-name` from one source of truth; regression test the stale-name
  case.
- **Stage D — Docs + optional cosmetic migration.** Update
  `vergil-vm/docs/site/docs/sessions.md` and mark the two design specs superseded;
  optional one-time `archived@` cosmetic strip.

`writing-plans` refines these into tasks and `Blocked-by` ordering. Stage A is the
foundation; B depends on A; C is largely independent; D follows.

## 7. Risks and mitigations

- **Muscle-memory break** (people type `vrg-vm session <repo>` and expect a resume).
  → No-arg path prints the recency list + the two verbs with guidance, rather than
  erroring blankly; documented prominently.
- **Legacy names stop resolving.** → Attach treats names as opaque strings; legacy
  `identity:slot:workspace` names keep working by exact match. No old-scheme parser
  needed.
- **Typo spawns an empty session.** → Prevented by the create/attach split (§4.2):
  `--resume` errors on absent, `--label` errors on collision.
- **Name collisions / `--fresh` reuse.** → Uniqueness among visible sessions +
  retire-suffix rename keeps exact-name reconnect unambiguous while never deleting.
- **Hidden ≠ gone confusion.** → `--all` always reveals older sessions;
  `list` states the active recency window.

## 8. Validation (live-lab)

A `validation`-kind task (seeded at plan time, `Blocked-by` the impl tasks and the
release/deploy of the new `vrg-vm`) will, on the actual VM:

1. Create a named session (`--label epic-999-probe vergil-tooling`), confirm the
   name is `epic-999-probe:vergil-project/vergil-tooling` and the title bar renders
   it.
2. Detach and **reconnect by `--resume epic-999-probe`**; confirm same session,
   correct title.
3. Confirm a mistyped `--resume` **errors** (no empty session spawned) and a
   colliding `--label` errors.
4. Confirm no auto-archive occurs over the old thresholds; confirm `list` defaults
   to recent and `--all` reveals older sessions; confirm no `archived@` marker is
   produced.
5. Confirm a legacy `identity:slot:workspace` session still reconnects by exact name.
6. Record `Outcome: SUCCESS`.

## 9. Deferred / open questions (ledger)

- **Epic-state–bound visibility** — a session's default visibility could track its
  epic's open/closed state (auto-drop from the list when the epic closes). A display
  refinement on top of §4.4's recency filter; deferred to avoid coupling `vrg-vm` to
  live GitHub epic state until the recency filter proves insufficient.
- **Mem Palace indexing** — the epic-keyed transcript archive is exactly the corpus
  Mem Palace wants; indexing/search is separate future work.
- **No-arg `vrg-vm session <workspace>` behavior** — list-and-guide (recommended)
  vs. hard error; finalized in the plan.
- **`vrg-vm` rename verb vs. `/rename` only** — whether a scripting-friendly rename
  subcommand is worth adding beyond Claude's native `/rename`; decide in Stage A.
