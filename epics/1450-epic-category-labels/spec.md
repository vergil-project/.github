# epic-category-labels: a category label on every epic — design

- **Epic:** vergil-project/.github#1450
- **Planning issue:** vergil-project/.github#1447 (review history is in its comments)
- **Status:** draft, in design review

## 1. Intent

Every live epic carries the `epic` label, and the tools sort epics into four
categories:

1. **Management.** The epics that run the planning process itself: the
   intake, planning and deferred queue epics, and the hoshin epic, which
   manages the review cadence.
2. **Ad-hoc.** Each repository's single ad-hoc epic.
3. **Proxy.** The same-org stand-in for another org's development epic
   (epic vergil-project/.github#1417). There will be many, and they normally
   need not be seen.
4. **Development.** Every other epic: planned with `epic-create`, driven
   with `epic-implement`, where most feature work happens.

Two of the four categories can be filtered in the GitHub UI with one label
today. Ad-hoc epics carry `ad-hoc` and proxy epics carry `proxy`. The other
two cannot. Each management kind has its own label (`intake`, `planning`,
`deferred`, `hoshin`), but no label covers the four of them. A development
epic has no label of its own, because the tools treat "development" as
whatever is left over (`standing_kind_of`, `is_proxy` and `is_archive` in
`lib/epics.py`). On 2026-11-20, `vergil-project/.github` held 7 open
development epics among 25 open epics.

**The why:** one label click in the GitHub UI should show each category,
and above all should separate the development epics, the ones worth
attention, from the many ad-hoc, proxy and management epics.

## 2. Goals, non-goals, anti-goals

### 2.1 Goals

- **G1.** Every live epic carries exactly one category label, from the set
  `epic-management`, `ad-hoc`, `proxy` and `epic-development`.
- **G2.** An epic gets its category label when it is created, from the
  tool that creates it.
- **G3.** An epic whose category label is missing or wrong is corrected
  without a human, by the daily epic sweep. The first sweep after release
  backfills every existing epic.

### 2.2 Non-goals

- Renaming `ad-hoc`, `proxy` or the per-kind management labels (`intake`,
  `planning`, `deferred`, `hoshin`). They stay, with their current meaning.
- Changing how any tool decides an epic's category.
- Category labels on archives or on tasks.

### 2.3 Anti-goals

- **A second source of truth.** If a tool read the new labels to decide
  behaviour, a missing or stray label could change how an epic is rolled
  up, swept or queued. The new labels must never be read for that.
- **A label repair that hides a problem.** A repair the sweep cannot make
  (a failed `gh` call) fails the run. It is never skipped silently.

## 3. Constraints

- Every label a tool applies must be in the canonical registry
  (`vergil-tooling` `src/vergil_tooling/data/labels.json`); `ensure_labels`
  refuses any other (`UnknownLabelError`) and creates a registered label in
  a repo on first use.
- The daily sweep (`.github` `epic-sweep.yml`, calling vergil-actions
  `ops-epic-sweep.yml`) runs `vrg-epic-audit --close` with an app token that
  already has org-wide `issues: write`. It installs vergil-tooling at the
  `.github` repo's `vergil.toml` pin (`v2.1`, a line pin), so a vergil-tooling
  release on that line reaches it with no workflow change.
- Epics are homed by visibility (`resolve_epic_home`): a public repo's epics
  in `<owner>/.github`, a private repo's in itself. The labels apply in every
  home.

## 4. Approaches

- **A. Add the two missing labels, keep the rest (chosen).** Add
  `epic-development` and `epic-management`. Ad-hoc and proxy keep `ad-hoc`
  and `proxy`. Each category is one click; nothing existing changes. The
  names are uneven (two carry an `epic-` prefix, two do not), which the
  human accepted.
- **B. A uniform four-label `epic-*` scheme.** Add `epic-ad-hoc` and
  `epic-proxy` too. Rejected: they would sit beside `ad-hoc` and `proxy`
  doing the same job, or replace them, which means renaming the labels the
  code keys on throughout `lib/epics.py`, `lib/proxies.py` and
  `lib/epic_audit.py`, plus every existing epic and archive.
- **Repair: report only.** The audit could flag drift for a human instead of
  fixing it. Rejected: the backfill would then need its own migration, and
  the reports would need someone to keep up with them. The sweep already
  repairs drift of the same shape (it strips the retired
  `triage`/`idea`/`research` labels).

## 5. Design

### 5.1 The labels

Add two entries to the label registry:

- **`epic-development`**, with the description "Development epic: planned
  with epic-create, closes when its tasks close".
- **`epic-management`**, with the description "Management epic: the intake,
  planning, deferred or hoshin epic; never auto-closes".

The colours are the builder's call; they should differ from each other and
from `ad-hoc` and `proxy`, so the four categories read apart in the UI.

### 5.2 The category of an epic

An issue's category is derived from its labels and title, exactly as the
tools derive it today; no new rule is introduced:

| Category | Derived when | Category label |
| --- | --- | --- |
| not an epic | no `epic` label, or an archive (`is_archive`) | none |
| management | `standing_kind_of` is a management kind | `epic-management` |
| ad-hoc | `standing_kind_of` is ad-hoc | `ad-hoc` |
| proxy | `is_proxy` | `proxy` |
| development | an `epic` that is none of the above | `epic-development` |

One function, in `lib/epics.py`, returns an issue's category and its
expected category label. Creation (§5.3) and the audit (§5.4) both use it,
so the two can never disagree.

`ad-hoc` and `proxy` are both a category label and the marker the
derivation reads. The derivation reads them first; the category label
follows from the result.

An issue that carries the markers of two categories at once (for example
`ad-hoc` and `proxy`) has no single derived category. The audit reports it
and does not repair it (§5.4).

### 5.3 Applied at creation

- **`vrg-epic-create`** adds `epic-development` to the epic it creates when
  its derived category is development: no `--label` that makes it a standing
  kind, and not `--proxy-for`.
- **`ensure_standing_epic`** adds `epic-management` when it creates an
  intake, planning, deferred or hoshin epic. Ad-hoc creation is unchanged.
- **`--proxy-for`** is unchanged: a proxy keeps `epic` + `proxy`.

Each path calls `ensure_labels` before applying the label, as every
label-applying command does.

### 5.4 Checked and repaired by the audit

A new pass in `vrg-epic-audit` reads every open epic in each home the audit
already covers and compares its category labels with its derived category:

- **Missing:** the expected category label is absent.
- **Wrong:** a category label of another category is present
  (`epic-development` or `epic-management` on an epic of another category).
- **Ambiguous:** the epic carries the markers of two categories (§5.2).

Without `--close` the pass reports each finding, like the audit's other
passes. With `--close` (the sweep) it adds each missing label and removes
each wrong one, then reports what it changed. It never repairs an ambiguous
epic: that is reported for a human, because which marker is wrong is a
judgement.

The pass only ever adds or removes `epic-development` and
`epic-management`. It never touches `ad-hoc`, `proxy`, `epic` or a
per-kind management label, since those are the markers the derivation
reads. A `gh` failure while repairing fails the run with its error.

Closed epics are not checked: the goal is filtering the live set, and the
backfill does not need to rewrite history. (Whether the GitHub UI filter for
closed epics benefits enough to include them is left to the plan; see §7.)

### 5.5 Backfill

There is no separate migration. Once a vergil-tooling release on the `v2.1`
line carries §5.1 to §5.4, the next daily sweep adds the category label to
every existing open epic. A validation task confirms that run.

## 6. Error handling

- An unregistered label: `ensure_labels` raises `UnknownLabelError`, and
  the creating command fails. Nothing is created half-labelled silently.
- A failed label write at creation: the command fails with the `gh` error,
  and the audit's next run repairs the label if the epic itself was
  created.
- A failed repair in the sweep: the run fails with the `gh` error.
- An ambiguous epic: reported, never repaired.

## 7. Left to the plan

- How the work splits into tasks, and the release and validation tasks
  around them.
- Whether the audit pass also covers closed epics.
- The label colours.
- Which docs describe the categories: the epic convention, the site docs
  in `vergil-tooling/docs/site`, and the `epic-create` skill's notes. The
  documentation-review task (vergil-project/vergil-tooling#4843) sweeps them.
