# Ad-hoc archiving — event-hook repo derivation + non-repo epic guard

- **Epic:** vergil-project/.github#266
- **Status:** Design (2026-08-09)
- **Line:** third in the ad-hoc archiving series — #238 (feature), #259 (dup-race
  fix), this (event-hook repo derivation).
- **Implemented in:** vergil-tooling (code) + a Vergil recovery of already-misfiled
  children.

## 1. Summary

The ad-hoc archiving **event-path rollup hook** (`epics.rollup`, #238 Task 6)
sends a just-closed child into the wrong archive because it derives the archive's
repo from **where the ad-hoc epic issue lives** (`parent.repo`) instead of **which
repo the epic is for** (the `<bare>` in its `Epic (ad hoc): <bare>` title). Fix the
derivation and guard it to canonical per-repo ad-hoc epics only.

## 2. Root cause & evidence

`rollup()` computes `ensure_adhoc_archive(f"{parent.owner}/{parent.repo}", …)`.
A **public** repo homes its ad-hoc epic centrally in `<org>/.github`, so
`parent.repo == ".github"` and the archive title becomes `Epic (ad hoc): .github —
<quarter>` regardless of the repo the epic is for. Two failures:

1. **Public-repo children misfiled** into a single `.github` bucket instead of
   their own `…<repo> — <quarter>` archive.
2. **Special non-repo `ad-hoc` epics drained.** An `ad-hoc`-labelled epic whose
   title bare-name is a description, not a repo (e.g.
   logical-minds-foundry/.github#104, a perpetual "Lab lifecycle reliability"
   collection), passes the hook's only guard (not-a-stamped-archive) and its
   children are drained out.

The **batch** drain is correct on both — it selects targets by repo name, so it
buckets per-repo and ignores non-repo epics. The defect is **event-hook only**,
missed because #2677's live validation exercised only the batch and the hook's one
unit test used the `.github` repo's *own* ad-hoc epic — the lone case where
`parent.repo` equals the correct bare name.

**Confirmed live in vergil-project:** `.github#263` and `#264` are
`Epic (ad hoc): .github — 2026-Q3` archives each holding a **vergil-tooling** child
(`#1356`, `#1371`) that the live hook misfiled after closing — they belong in
`Epic (ad hoc): vergil-tooling — 2026-Q3` (`.github#249`). (They are also duplicates
of each other — a cross-invocation list-lag the #259 tolerance narrows but does not
fully close.)

## 3. Design

### 3.1 Code — derive from title + guard (vergil-tooling)

- **Derive the target repo from the title's bare name.** In `rollup()`, parse
  `<bare>` from the parent's `Epic (ad hoc): <bare>` title and build the target as
  `<owner>/<bare>` — so a `.github`-homed public-repo epic archives into its own
  per-repo bucket, matching the batch.
- **Canonical-epic guard.** Drain only when `<bare>` is a **real repo** in the org;
  otherwise skip (special non-repo epics fall through, exactly as the batch already
  ignores them). Implementation options for the plan: validate `<bare>` against the
  org repo list (`github.list_org_repos`) / a repo-existence lookup, or require the
  parent to equal `find_adhoc_epic(<owner>/<bare>)`. A cheap first filter — a repo
  name cannot contain spaces or `/` — rejects #104-style descriptive titles up
  front.
- **Tests for the cases the original hook test missed:** public-repo epic homed in
  `.github` → child archived into its per-repo bucket; `.github`-own epic →
  unchanged; special non-repo epic → **skipped** (no archive, no reparent).

### 3.2 Deploy (vergil-tooling)

Release + install so all consuming orgs get the corrected hook.

### 3.3 Recover (vergil-project)

Move the misfiled children out of the wrong `.github` archives into their correct
per-repo archive and close the empty misfiled archives: `#1356`/`#1371` →
`Epic (ad hoc): vergil-tooling — 2026-Q3` (`#249`), then close `#263`/`#264`. Then
verify no `Epic (ad hoc): .github — <quarter>` archive holds a non-`.github` child.

## 4. Sequencing

`code fix → release + install → Vergil recovery`. The recovery must run against the
fixed, deployed hook, or newly-closed children keep misfiling.

## 5. Acceptance

- Hook archives a closed child into `Epic (ad hoc): <the-epic's-repo> — <quarter>`,
  never `.github` (unless the epic is the `.github` repo's own).
- A closed child of a non-repo `ad-hoc` epic is left in place (skipped).
- Post-recovery: no `.github — <quarter>` archive holds a non-`.github` child; the
  misfiled children sit in their correct per-repo archive.

## 6. Out of scope — cross-org (tracked in logical-minds-foundry)

Per one-org-per-epic, the logical-minds-foundry remediation is separate: deploy the
fix there; recover its pre-existing duplicate archives
(`#180/#181/#182`, `mq-resiliency-lab-for-linux — 2026-Q3`); reclassify #104 (drop
the `ad-hoc` label / fold into the real ad-hoc flow); and confirm the drain token's
visibility into that org's **private repo** (which `gh repo list` did not surface
under the current token). Tracked in `logical-minds-foundry/.github` once this
ships.
