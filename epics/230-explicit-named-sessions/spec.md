# Explicit, purpose-named sessions (deprecate slots + archive)

- **Epic:** vergil-project/.github#230
- **Status:** Design approved 2026-08-06 (brainstormed via `epic-create`; revised after
  `paad:pushback` + a Claude Code internals audit).
- **Repos:** vergil-project/vergil-tooling (`vrg-vm` implementation — the bulk of the
  work), vergil-project/vergil-vm (docs/site + the two session design specs),
  vergil-project/.github (epic docs). Epic homed in `.github`.
- **Supersedes:** vergil-tooling#2602 (archived sessions not filtered out of
  reconnect selection) — closed as superseded, its requirement folded in (§4.7).
- **Promoted from:** direct brainstorm (no triage origin).

## 1. Summary

`vrg-vm session` names sessions `identity:slot:workspace` and, by default, resumes
the **most-recently-used** session for a repo. The slot is a meaningless auto-number
and the default *guesses* which session you meant; an entire **staleness + archive**
apparatus exists only to stop that guess from dragging stale context into new work.

This epic makes reconnect **fully explicit by name** and binds names to **purpose**
(an epic or ad-hoc problem) instead of a slot. Because the system never guesses
which session you meant, it can never guess *wrong* — so the staleness/archive
apparatus is **removed**, not merely bug-fixed. The name becomes `label:workspace`
(identity dropped, vestigial); creation is explicit and named; reconnect is by exact
name.

Two things reshaped this design during review and must be first-class:

- **First-class goal — the implementation-history archive.** Epic-associated names
  turn the pile of persisted transcripts into a **navigable history keyed by epic**
  ("how did we actually build #213?"), exactly the corpus a future Mem Palace wants.
  "Never delete" becomes the point, not hoarding.
- **First-class constraint — build on supported interfaces (§3).** The current
  implementation reverse-engineered Claude Code internals with no documentation, and
  those internals move under us (the naming transcript event silently changed
  `agent-name` → `custom-title`). Official docs state the transcript/roster formats
  are **internal and can break on any release**. So this epic must prefer *supported*
  Claude Code interfaces and **isolate** any remaining unsupported dependency behind a
  single seam — gated by an **empirical SDK-verification spike** before any
  mechanism-dependent work.

## 2. Motivation and ground truth

### 2.1 How it works today (facts)

- **Name** = `identity:slot:workspace`, set via Claude's `-n/--name`. The name lives
  inside the transcript as append-only naming events, last-one-wins; the resolver
  (`vrg_vm_resolve.py`) reads **both** `agent-name` and `custom-title` because Claude
  changed the event type (~2.1.16x). Live roster: `~/.claude/sessions/<pid>.json`.
- **Default resume** picks the most-recent slot; `--slot N` targets one;
  **`--resume <NAME>`** (vergil-tooling#2285) resumes by exact *display name* — but
  see §2.2: that name→id resolution is **our** layer, not a native Claude capability.
- **`--name`** is the **VM instance** (`local`/`cloud`), not a session label.
- **Archive** = a transcript whose current name carries an `archived@` marker,
  excluded from active resolution; driven by `--fresh` and the staleness bands.
- Code: `src/vergil_tooling/bin/vrg_vm.py`, `lib/session.py`, `vrg_vm_resolve.py`.

### 2.2 Claude Code internals audit (verified on the installed v2.1.220)

**Confirmed supported (from `claude --help`):**

- `-n, --name <name>` — sets the display name (prompt box, `/resume` picker, terminal
  title). **This is the supported way to name a session.**
- `-r, --resume [value]` — "Resume a conversation **by session ID**, or open an
  **interactive picker with optional search term**." `-c, --continue` (most recent),
  `--fork-session`, `--from-pr` also present.
- `/rename <name>` (in-session) — supported per docs; the sanctioned rename path.

**Confirmed NOT supported:**

- **Deterministic, non-interactive resume by exact *name*.** `--resume` takes a
  session **id** or an **interactive** picker — *not* a scripted name→session lookup.
  So resolving a name to a session id (which the whole "reconnect by name" model, and
  today's `--resume NAME`, depend on) is **inherently our layer**.

**Confirmed unstable (official docs):** the transcript JSONL format and event types,
and the `~/.claude/sessions/<pid>.json` roster, are **internal — "scripts that parse
these files directly can break on any release."** Our name-resolution and listing sit
on exactly these. (This is what the `agent-name → custom-title` shift already proved.)

**Reported but UNVERIFIED (must be checked by the §6 spike, not trusted yet):** an
Agent SDK (`claude-agent-sdk`, Python) may expose supported session functions
(listing, info, rename, tag) that would resolve name→id and enumerate sessions
*without* scraping. The research that surfaced this also overstated `--resume <name>`
(disproven above), so **no design decision may depend on these SDK functions until the
spike confirms they exist, resolve names, and see interactively-launched sessions**
(the SDK's `query()` is headless; our sessions are interactive TTYs — that the SDK's
session store even includes them is an open question).

### 2.3 What breaks down, and the reframe

1. **Wrong-session reconnect (vergil-tooling#2602).** The *selection* grabbed an
   archived session and the title bar faithfully showed *that* session's name — a
   **selection** defect, not a render defect (§4.7).
2. **Names carry no purpose.** `who:number:where` omits *what the session is for*;
   the slot is noise and identity is vestigial.

The whole staleness/archive machinery exists to protect the **auto-resume-most-recent
guess**. Make reconnect **explicit by exact name** and the machinery becomes
unnecessary — deleting archive is a *consequence* of explicit naming, not a gamble.

## 3. Design principle — supported interfaces, isolated hacks

1. **Use supported interfaces for everything they cover:** naming via `-n` at launch
   and `/rename` in-session; resume/fork via `--resume <id>` / `--continue` /
   `--fork-session`. **Never hand-write transcript naming events**, and never depend
   on a specific event type (`agent-name`/`custom-title`) in our own code.
2. **Isolate the unsupported remainder behind one seam.** The only capabilities with
   no confirmed supported CLI surface are **name → session-id resolution** and
   **session enumeration/last-activity**. Confine both to a single module (a
   `SessionStore` interface) with two interchangeable backends: the current
   transcript/roster **scrape** (isolated, not spread through the code) and a future
   **Agent SDK** backend — selected once the §6 spike verifies the SDK. The rest of
   `vrg-vm` speaks only to the seam, so swapping backends touches one file.
3. **Fail loud, not silent** on any assumption the seam can't satisfy (e.g. a name
   that resolves to zero or multiple live sessions — §4.3).

## 4. Design decisions

### 4.1 Name = `label:workspace`

Colon-delimited; `label` is the human functional name, `workspace` is the starting
repo (relative to `projects_dir`) that anchors the CLAUDE.md/memory bootstrap.
**Identity is removed** from the name. `label` is **soft-validated**: clean slug
required (so `label:workspace` parses/renders); **warn, don't reject**, if it lacks an
`epic-`/`adhoc-` prefix (preserving "start with a slug, rename once the epic number
exists"). The composed `label:workspace` is what we pass to Claude's `-n`.

### 4.2 Create (`--label`) and attach (`--resume`) are distinct verbs

- **`--label <name>`** (new flag — `--name` is the VM instance) launches a new session
  named `label:workspace` via `claude -n <label:workspace>`. It **must not collide**
  with an existing *visible* session (§4.3). Soft-warns on a non-`epic-`/`adhoc-` label.
- **`--resume <name>`** attaches an existing session: the seam resolves the name → a
  session id, then we call `claude --resume <sessionId>`. It **errors if the name
  resolves to no visible session**.

Keeping create (must-not-exist) and attach (must-exist) distinct is **typo-safe**: a
mistyped `--resume` errors instead of spawning an empty session; a colliding `--label`
errors instead of resuming the wrong thing.

**Workspace is derived from the resolved session on `--resume`, not re-specified.**
The name already carries the workspace, and the resumed session's cwd determines the
memory/project slug (`_project_slug(cwd)`; the vm-memory parity work #2419/#2423). So
resume takes the label and derives the workspace/cwd from the resolved session — a
separately-typed workspace positional that could diverge (and project the wrong
`MEMORY.md` onto a foreign transcript) is disallowed. Workspace stays required only on
**create**.

### 4.3 Uniqueness is operational; `--slot` and auto-resume-most-recent are removed

"Unique names" cannot mean "one transcript per name ever" — a Claude `/clear`
rotation leaves an abandoned session id still carrying the old name (the reason
`session.py:_displaces` / `build_slots` exist). So uniqueness is defined over
**visible (live/recent) sessions**, and the seam resolves a name to a session id by
the existing rule: **liveness first, then most-recent activity**. `--label` "collides"
when a *visible* session already holds the name. If `--resume` resolves to **zero**
visible sessions it errors; if it somehow resolves to **multiple co-equal** live
sessions it fails loud (never silently picks one). The `--slot` flag and the
auto-resume-most-recent default are **removed**.

### 4.4 Archive deleted → recency display-filter

Remove the `archived@` marker, the fresh/warn/stale bands, auto-archiving, and the
`session_stale_days`/`session_archive_days` thresholds. `list --sessions` defaults to
**recently-active** (a `session_recent_days` display window) with `--all` revealing the
rest — a pure display filter over the seam's enumeration, no name mutation, nothing
moved. Every transcript stays on disk, reconnectable by exact name. `--archived` is
removed; `--all` changes from "include archived" to "include older-than-recent". This
removes #2602's entire bug class: there is no archive-vs-active name state to
mis-filter.

### 4.5 Rename, `--fresh`, `--fork`, guardrail — via supported interfaces

- **Rename** is Claude's `/rename` (in-session) — the slug → `epic-<N>-<slug>` path.
  We do **not** write transcript events ourselves. A scripting-friendly `vrg-vm rename`
  verb, if added, goes through the seam (SDK `rename` if verified, else a documented
  mechanism) — never a raw transcript append.
- **`--fresh`** restarts a name cleanly by **renaming the prior same-named session**
  (via the supported rename) with a retired suffix (still reconnectable, never
  deleted) and launching a new session with the clean `label` — preserving
  visible-name uniqueness, no resurrected archive state.
- **`--fork`** maps to Claude's supported `--fork-session`; the **no-double-attach
  guardrail** stays.

### 4.6 Migration is minimal

No forced rename, no purge (transcripts are the archive). Legacy
`identity:slot:workspace` and old `archived@`-marked sessions persist as **opaque
names**: reconnectable by exact name, recency-hidden once idle. The seam treats a name
as an opaque string, so legacy names keep resolving without an old-scheme parser.
Optional one-time **cosmetic** strip of `archived@` markers (via supported rename) so
the list renders uniformly.

### 4.7 Selection correctness (folds in #2602)

The requirement is **selection**, not rendering: reconnect must never *pick* a stale or
archived session — satisfied structurally by explicit exact-name attach (§4.2) plus the
deletion of the archive state (§4.4). The title bar already re-asserts the resolved
session's name via `-n` on resume, so once selection is correct the title is correct.
Acceptance: after reconnect, the rendered title equals the name of the session the user
explicitly asked for.

## 5. Command surface (before → after)

| | Before | After |
|---|---|---|
| Create | implicit slot, then `/rename` | **`--label <name>`** → `claude -n label:workspace` |
| Reconnect | most-recent, or `--resume <name>` | **`--resume <name>`** → seam resolves name→id → `claude --resume <id>` |
| Slots | `--slot N`, auto most-recent | **removed** |
| Fresh | `--fresh` (archive old) | `--fresh` (supported-rename old to retired suffix) |
| Fork | `--fork` | `--fork` → Claude `--fork-session` |
| VM instance | `--name` | `--name` (unchanged) |
| List | `--active/--idle/--archived/--all` | `--active/--idle/--all` (recency); `--archived` removed |
| Staleness cfg | `session_stale_days`, `session_archive_days` | removed; `session_recent_days` (display) |

## 6. Rollout — independently shippable stages

- **Stage 0 — Agent SDK verification spike (BLOCKING).** Empirically verify (against
  the actually-installed `claude-agent-sdk`) which session-management functions exist;
  whether they resolve a **name → session id**, enumerate sessions with
  last-activity/cwd, and **see interactively-launched sessions** (not just SDK
  `query()` ones). Deliverable: a written finding + a go/no-go on the SDK backend for
  the seam. **Gates Stages A–B.** (This is why the seam exists: A/B ship on the scrape
  backend if the SDK can't do it, on the SDK backend if it can — same interface.)
- **Stage A — The `SessionStore` seam + named create + exact-name resume.** Define the
  seam (resolve-name→id, enumerate); implement the chosen backend; add `--label`
  (create via `-n`, uniqueness check), make `--resume` resolve-then-`--resume <id>`
  with workspace derived from the resolved session; remove `--slot` and the
  most-recent default; drop identity from the name. Tests against the seam interface.
- **Stage B — Delete archive; recency filter.** Remove marker/bands/auto-archive/
  thresholds; add the `list` recency filter + `session_recent_days`; retire
  `--archived`; re-point `--all`; reconceive `--fresh` as supported-rename-to-retired.
  Tests.
- **Stage C — Selection-correctness regression (#2602).** Prove reconnect never selects
  a non-requested session; title equals the requested name.
- **Stage D — Docs + optional cosmetic migration.** Update
  `vergil-vm/docs/site/docs/sessions.md`; mark the two design specs superseded; note the
  supported-interface stance; optional `archived@` cosmetic strip.

`writing-plans` refines these into tasks + `Blocked-by` ordering; Stage 0 blocks A/B.

## 7. Risks and mitigations

- **SDK can't resolve names / can't see interactive sessions.** → Stage 0 gates the
  choice; the seam ships on the isolated scrape backend if so, and the scrape lives in
  one module we already know how to write.
- **Claude changes internals again.** → Only the seam's scrape backend can be affected;
  the rest of `vrg-vm` speaks supported flags. Swapping to the SDK backend later is a
  one-module change.
- **Muscle-memory break** (bare `vrg-vm session <repo>` expecting a resume). → No-arg
  path prints the recency list + the two verbs; documented.
- **Legacy names stop resolving.** → Seam treats names as opaque; legacy names resolve
  by exact match.
- **Name→id ambiguity after `/clear`.** → Liveness-then-recency resolution; fail loud on
  co-equal live matches (§4.3).
- **Wrong memory slug on resume.** → Workspace derived from the resolved session, not a
  divergent positional (§4.2).

## 8. Validation (live-lab)

A `validation`-kind task (seeded at plan time; `Blocked-by` the impl tasks and the
release/deploy of the new `vrg-vm`) will, on the actual VM:

1. Create `--label epic-999-probe vergil-tooling`; confirm the name and that the title
   renders `epic-999-probe:...`.
2. Detach and **`--resume epic-999-probe`**; confirm same session, correct title, and
   the cwd/memory slug matches the session's origin repo.
3. Confirm a mistyped `--resume` **errors** (no empty session) and a colliding
   `--label` errors.
4. Confirm no auto-archive over the old thresholds; `list` defaults to recent, `--all`
   reveals older; no `archived@` marker is produced; `--fresh` retires the old name via
   the supported rename.
5. Confirm a legacy `identity:slot:workspace` session still reconnects by exact name.
6. Confirm the seam backend in use matches Stage 0's decision (SDK if verified).
7. Record `Outcome: SUCCESS`.

## 9. Deferred / open questions (ledger)

- **Agent SDK backend adoption** — decided by Stage 0. If the SDK resolves names,
  enumerates, and sees interactive sessions, the seam's SDK backend replaces the scrape
  entirely (and `tag`-based organization becomes possible); otherwise the scrape backend
  ships and SDK adoption is revisited when the SDK matures.
- **Epic-state–bound visibility** — a session's default visibility could track its
  epic's open/closed state; a display refinement on §4.4's recency filter, deferred to
  avoid coupling `vrg-vm` to live GitHub epic state.
- **Mem Palace indexing** — the epic-keyed transcript archive is the corpus Mem Palace
  wants; indexing/search is separate future work.
- **No-arg `vrg-vm session <workspace>` behavior** — list-and-guide (recommended) vs.
  hard error; finalized in the plan.
- **`vrg-vm rename` subcommand** — whether a scripting rename verb (through the seam) is
  worth adding beyond `/rename`; decide in Stage A.
