# Reclassify per-quarter ad-hoc buckets as archives

- **Epic:** vergil-project/.github#281
- **Status:** Design (2026-08-11)
- **Implemented in:** vergil-tooling (label, drain/creation paths, normalize
  sweep, tests). Docs across vergil-tooling (`docs/site`) and this repo where the
  ad-hoc/epic model is described.

## 1. Summary

When closed children drain out of a repo's **live** ad-hoc epic, they are
re-parented into **per-quarter buckets**. Today those buckets are themselves
titled `Epic (ad hoc): <repo> — YYYY-Qn` and labelled `epic` + `ad-hoc`. Because
they carry the `epic` label, every `label:epic` filter — in `.github` and in
member repos — is cluttered with quarter-specific historical buckets that exist
only for the record and that nobody needs to look at day to day.

This epic reclassifies those terminal buckets as **archives**:

- Title `Archive (ad hoc): <repo> — YYYY-Qn` (identical to today except the
  leading word `Epic` becomes `Archive`).
- Labels `archive` + `ad-hoc` — a new `archive` label replaces `epic`; `ad-hoc`
  is retained.

The **live** ad-hoc epic (`Epic (ad hoc): <repo>`, labels `epic` + `ad-hoc`) is
**unchanged**. All drain / rollup / re-parenting / quarter-bucketing mechanics
are **unchanged**. This is a naming + labeling change, a new `archive` label, a
self-healing creation path, and a one-time migration of the existing archive
backlog.

## 2. Motivation & constraints

The archive buckets are a bookkeeping device: they hold the closed ad-hoc work
for a finished quarter so the live issue lists stay uncluttered and the
historical record is preserved. They are terminal (past quarters are closed) and
carry no forward work. Labelling them `epic` conflates them with the active
epics a reader actually cares about, so `label:epic state:open` — the natural
"what epics are in flight?" view in the GitHub UI — is polluted with per-repo,
per-quarter buckets.

Constraints that shape the design:

- **The `ad-hoc` label is retained.** `roadmap.py::_is_perpetual` already filters
  out anything `ad-hoc`-labelled, so keeping `ad-hoc` means archives stay off the
  strategic roadmap with no roadmap-code change, and the archive remains
  self-documenting as ad-hoc-derived.
- **The live epic must not be touched.** It keeps `epic` + `ad-hoc` and its
  unstamped `Epic (ad hoc): <repo>` title. Only the `— YYYY-Qn`-stamped buckets
  change.
- **No duplicate archives during migration.** The new-form title
  (`Archive (ad hoc): …`) differs from the old-form title (`Epic (ad hoc): …`),
  so a naive creation path would mint a second bucket for a quarter whose
  old-form archive still exists. The creation path must recognize and heal a
  legacy archive in place (§3.4), so migration order can never cause a
  duplicate.

## 3. Design

All logic lives in `src/vergil_tooling/lib/epics.py` and its CLI driver
`src/vergil_tooling/bin/vrg_adhoc_epic.py`, plus the label set in
`src/vergil_tooling/data/labels.json`.

### 3.1 The new `archive` label

Add to `src/vergil_tooling/data/labels.json`:

```json
{"name": "archive", "color": "<hex>", "description": "Terminal per-quarter archive of closed ad-hoc work; historical record"}
```

- `vrg-ensure-label` provisions the label across all managed repos on the next
  label sync, and — critically — into `vergil-project/.github`, where archives
  live, so the label exists **before** any archive is created or relabeled.
- Pick a color visually distinct from `epic` (`6f42c1`) and `ad-hoc` (`5319e7`)
  so the three read apart in the UI; the plan resolves the exact hex.

### 3.2 Constants and the archive recognizer

In `epics.py`:

- Add `_ADHOC_ARCHIVE_TITLE_PREFIX = "Archive (ad hoc): "`.
- Add `_ADHOC_ARCHIVE_LABELS = ("archive", "ad-hoc")`.
- Repoint `_ADHOC_ARCHIVE_RE` to the new form:
  `^Archive \(ad hoc\): (?P<bare>.+) — (?P<quarter>\d{4}-Q[1-4])$`. This regex is
  how `rollup` distinguishes a terminal archive from the live ad-hoc epic, so it
  must match the new prefix.
- Add a **legacy recognizer** — `_LEGACY_ADHOC_ARCHIVE_RE`
  (`^Epic \(ad hoc\): … — YYYY-Qn$`) plus the old `("epic", "ad-hoc")` label
  pair — used **only** by the self-healing creation path (§3.4) and the normalize
  sweep (§3.5) to locate archives that still need converting. Steady-state code
  keys off the new recognizer only.

### 3.3 Creation and discovery paths

- `ensure_adhoc_archive` builds the title from `_ADHOC_ARCHIVE_TITLE_PREFIX` and
  applies `_ADHOC_ARCHIVE_LABELS`.
- `list_open_adhoc_archives` and every other archive-discovery query switch from
  `--label epic --label ad-hoc` to `--label archive --label ad-hoc`.
- Where `_find_epic_by_title` is reused for archive lookup with a hardcoded
  `epic`+`ad-hoc` pair, **parameterize the label set** (or add an archive-specific
  variant) so archive lookups use `archive`+`ad-hoc` while **live-epic** lookups
  (`ensure_adhoc_epic`, `resolve_epic_ref`, `is_epic`, the audits) keep
  `epic`+`ad-hoc`. There must be exactly one place that spells the archive label
  set, mirrored from `_ADHOC_ARCHIVE_LABELS`, so the discovery query and the
  creation labels cannot drift.
- `ensure_adhoc_epic` (the live epic) and its discovery are **untouched**.

### 3.4 Self-healing creation — no migration-order race

`ensure_adhoc_archive(target_repo, quarter)` resolves in this order:

1. **New-form lookup.** Find an existing archive by its new title
   (`Archive (ad hoc): <bare> — <quarter>`) under `--label archive --label
   ad-hoc`. If found, return it.
2. **Legacy heal.** Else, look for a **legacy-form** archive for the same
   repo+quarter (old title `Epic (ad hoc): <bare> — <quarter>`, `epic`+`ad-hoc`).
   If found, **normalize it in place** — retitle to the new form, add `archive`,
   remove `epic`, keep `ad-hoc` — and return it.
3. **Create.** Else, create a fresh archive in new form.

This makes the creation path idempotent and race-free regardless of whether the
bulk sweep (§3.5) has run: the drain/rollup path can never produce a duplicate
bucket for a quarter that already has a legacy archive, because it heals the
legacy one instead of minting a new-form sibling. The standalone sweep is then a
**bulk convenience and safety net**, not a correctness prerequisite.

### 3.5 One-time idempotent normalize sweep

- New `normalize_adhoc_archives(org, apply=…)` in `epics.py`, surfaced as a
  `vrg-adhoc-epic normalize` subcommand (dry-run by default, `--apply` to
  execute, mirroring the existing `archive` subcommand's preview/apply shape).
- It **resolves each repo's own epic home the way the drain engine does** —
  iterating `list_org_repos` and resolving `resolve_epic_home` per repo, deduping
  homes so `.github` is swept once rather than once per public repo — so it covers
  **both** public archives (in `<org>/.github`) and **private** repos' archives
  (self-homed in the repo itself, `epics.py:381`). Scanning only `<org>/.github`
  would silently leave every private repo's archives in legacy form; mirroring
  `drain_adhoc_org` (`epics.py:631-650`) is the traversal already trusted to find
  archives wherever they live.
- Within each home it finds every **legacy-form** archive (old title +
  `epic` label), **open or closed**, and for each performs the same in-place
  conversion as §3.4 step 2: retitle to `Archive (ad hoc): …`, add `archive`,
  remove `epic`, keep `ad-hoc`.
- **Idempotent:** an already-converted (new-form) archive is skipped, so the
  sweep is safe to re-run and doubles as a permanent straggler net. The dry-run
  prints exactly what it would convert.
- Run once at rollout with `--apply` to migrate the whole existing backlog (the
  decision is to migrate **all** existing archives, open and closed).

### 3.6 Roadmap & audit safety

- **Roadmap:** no change. `roadmap.py::_is_perpetual` filters `ad-hoc`, which
  archives retain, so they stay off the strategic roadmap exactly as today.
- **Audit invariant 2 — `stray_dotgithub_issue` MUST be changed (required
  work, not a "verify").** This is the one audit that definitely breaks. Today an
  archive is skipped by that check **only** because it carries the `epic` label
  (`epic_audit.py:332`, `"epic" in labels`). An archive is a **top-level** issue
  with **no parent**, so once the `epic` label is gone it is neither epic-labelled,
  nor intake, nor parented-under-an-epic — and every open (current-quarter)
  archive across the org is reported as a stray `.github` violation (fail-loud).
  Fix: **add `archive` to the skip set in `stray_dotgithub_issue`**, treating an
  `archive`-labelled home issue as legitimately non-stray alongside `epic` and
  intake, with a regression test asserting an open archive is not flagged.
- **Other epic audits — checked, no change needed.** They query `--label epic`,
  so a non-epic archive simply drops out:
  - `epic_outside_dotgithub` (invariant 1, `gh search issues --label epic`) —
    archive no longer matched; safe.
  - `closed_epic_open_child` (`--label epic --state closed`, `epic_audit.py:342`)
    — archive no longer returned at all; safe. (It was previously returned but
    skipped via the `ad-hoc` guard at `:375`; the outcome is unchanged.)
  The re-parenting linkage itself uses the GitHub sub-issue relationship, which is
  independent of labels, so a closed child under an `archive` parent is unaffected.
  The plan restates this invariant-by-invariant disposition.

## 4. Scope boundary — what this does not touch

- **The live ad-hoc epic** — unchanged in title and labels.
- **Drain / rollup / re-parenting / quarter bucketing** — unchanged; only the
  title and label of the bucket an item lands in change.
- **Finite/strategic epics and `vrg-epic-create`** — unchanged; they keep the
  `epic` label and their audits.
- **The `epic` label definition** — unchanged; it stays for real epics and the
  live ad-hoc epic. Only its application to quarter buckets is removed.
- **Standing→ad-hoc migration (`adhoc_migrate.py`)** — unrelated; it produces the
  live `Epic (ad hoc): <repo>` epic, which keeps the `epic` label.

## 5. Sequencing

`label (labels.json) → code (epics.py + vrg_adhoc_epic.py + tests) → label sync → normalize --apply`.

- The `archive` label must exist in `.github` before any archive is created or
  relabeled, so a **label sync** (`vrg-ensure-label`) precedes the first run of
  the normalize sweep. Landing the label in `labels.json` in the same PR as the
  code keeps them together; the sync and the sweep are the operational
  (post-merge) steps.
- Because the creation path self-heals (§3.4), there is **no ordering hazard**
  between "code is live" and "sweep has run": any archive the drain touches after
  the code lands is healed in place; the sweep merely converts the ones the drain
  will not touch again (closed past archives) in bulk.

## 6. Acceptance

- A newly drained quarter bucket is titled `Archive (ad hoc): <repo> — YYYY-Qn`
  and labelled `archive` + `ad-hoc`, not `epic`.
- The live ad-hoc epic is untouched: still `Epic (ad hoc): <repo>`, still
  `epic` + `ad-hoc`.
- `label:epic state:open` in `.github` no longer returns per-quarter archive
  buckets; `label:archive` returns them.
- Running the drain against a quarter whose archive still exists in **legacy
  form** heals that archive in place and does **not** create a duplicate bucket.
- `vrg-adhoc-epic normalize` dry-run lists every legacy-form archive (open and
  closed); `--apply` converts them; a second `--apply` is a no-op.
- After the rollout sweep, no archive across the org remains in legacy form —
  **including private repos' self-homed archives** — and the historical record
  (including closed archives) reads as `Archive (ad hoc): …` under `label:archive`.
- The strategic roadmap is unchanged (archives never appeared and still do not).
- `stray_dotgithub_issue` does **not** flag an open archive as a stray, and no
  other epic audit flags a closed child that sits under a converted archive.
- The `archive` label exists in `.github` and every managed repo after the label
  sync.
- Docs (the ad-hoc/epic model in `docs/site` and any tooling docs describing the
  archive buckets) describe the archive title, the `archive` label, and the
  normalize sweep.

## 7. Testing

- **`epics.py` constants/regex** — the new `_ADHOC_ARCHIVE_RE` matches new-form
  titles and rejects the live epic and legacy-form titles; the legacy recognizer
  matches old-form and rejects new-form.
- **`ensure_adhoc_archive` self-heal** — three branches: (a) new-form archive
  exists ⇒ returned unchanged; (b) only a legacy archive exists ⇒ healed in place
  (retitled, `archive` added, `epic` removed, `ad-hoc` kept) and returned, no new
  issue created; (c) neither exists ⇒ new-form archive created with
  `archive`+`ad-hoc`.
- **Drain/rollup end-to-end** — a closed child drains into a new-form archive;
  re-running against an existing legacy archive heals rather than duplicates;
  the live epic is never relabeled.
- **`normalize_adhoc_archives`** — legacy (open) and legacy (closed) both
  convert; a new-form archive is skipped; a second run is a no-op; the live
  ad-hoc epic is never touched; **a private repo's self-homed legacy archive is
  converted** (the per-repo home traversal, not `.github`-only).
- **Discovery queries** — archive lookups use `--label archive --label ad-hoc`;
  live-epic lookups still use `--label epic --label ad-hoc`.
- **Audit — `stray_dotgithub_issue`** — an open `archive`+`ad-hoc` issue with no
  parent is **not** flagged as a stray (regression guard for §3.6); a closed
  child under an `archive`-labelled parent is not flagged by
  `closed_epic_open_child`.
- **Label definition** — `labels.json` carries a well-formed `archive` entry.
