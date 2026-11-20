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

- **G1.** Every open epic in the six orgs of §3 carries exactly one
  category label, from the set `epic-management`, `ad-hoc`, `proxy` and
  `epic-development`.
- **G2.** An epic gets its category label when it is created, from the
  tool that creates it.
- **G3.** The epics that exist before this epic ships get their category
  label once, from a one-off deployment that also creates the two new
  labels in every repo of the six orgs.

### 2.2 Non-goals

- Renaming `ad-hoc`, `proxy` or the per-kind management labels (`intake`,
  `planning`, `deferred`, `hoshin`). They stay, with their current meaning.
- Changing how any tool decides an epic's category.
- Category labels on archives, on tasks or on closed epics.
- **Ongoing drift repair.** No audit pass or sweep checks the category
  labels after the one-off. Once the tools apply the label at creation and
  the existing epics are labelled once, a check would be more machinery
  than the problem needs. An epic created outside the tools, or whose
  label is removed by hand, stays wrong until someone notices; since the
  labels are for display only (§2.3), that changes no behaviour.

### 2.3 Anti-goals

- **A second source of truth.** If a tool read the new labels to decide
  behaviour, a missing or stray label could change how an epic is rolled
  up, swept or queued. The new labels must never be read for that, and
  never used to find an existing epic.
- **A silent half-deployment.** A label the one-off cannot create or apply
  (a failed `gh` call) is reported with its error, and the deployment does
  not record success until every repo and epic is done.

## 3. Constraints

- Every label a tool applies must be in the canonical registry
  (`vergil-tooling` `src/vergil_tooling/data/labels.json`); `ensure_labels`
  refuses any other (`UnknownLabelError`) and creates a registered label in
  a repo on first use.
- Epics are homed by visibility (`resolve_epic_home`): a public repo's
  epics in `<owner>/.github`, a private repo's in itself.
- The six orgs in scope are `vergil-project`, `logical-minds-foundry`,
  `mq-rest-admin-project`, `mnemosys-project`, `diogenes-project` and
  `vergils-nemesis`. Each holds open epics today. The last three are
  dormant; the one-off only manages labels there and runs no other code.

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
- **Existing epics: a one-off deployment (chosen), not a self-healing
  audit.** A `vrg-epic-audit` pass run by the daily sweep could add missing
  labels and remove wrong ones, and so backfill on its first run. Rejected
  as unnecessary complexity: once the tools label every new epic, the only
  epics without a label are the ones that exist today, and labelling them
  once is a mechanical task. It would also have covered only the orgs that
  run the daily sweep, which three of the six do not.

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

One function, in `lib/epics.py`, returns the category and its category
label for a set of labels and a title. Every creation path (§5.3) uses it,
and the one-off (§5.4) applies the same rules, so they never disagree.

`ad-hoc` and `proxy` are both a category label and the marker the
derivation reads. The derivation reads them first; the category label
follows from the result.

An issue that carries the markers of two categories at once (for example
`ad-hoc` and `proxy`) has no single derived category. The function
reports that as an error, never a guess.

### 5.3 Applied at creation

- **`vrg-epic-create`** applies the category label the §5.2 function
  derives from the epic's final label set, whatever the category: a plain
  epic gets `epic-development`, and one created with `--label intake` (or
  `planning`, `deferred`, `hoshin`) gets `epic-management`. `vrg-epic-create`
  does not call `ensure_labels` today; it gains the call, for every label it
  applies, before it creates the issue. An ambiguous label set (§5.2) makes
  it refuse, creating nothing.
- **`ensure_standing_epic`** adds `epic-management` when it creates an
  intake, planning, deferred or hoshin epic. The label is passed only to
  `ensure_labels` and to `create_issue`, as an extra label at creation.
  `StandingKind.labels` stays as it is, and so does every lookup that filters
  on it (`_find_epic_by_title`, `_await_listed`, `find_standing_epic`):
  `StandingKind.labels` is also the filter that finds the existing epic, so
  adding the label there would make every existing standing epic invisible
  until labelled, and create a duplicate. A test checks that an existing
  standing epic without `epic-management` is found and reused, not
  duplicated. Ad-hoc creation is unchanged.
- **`--proxy-for`** is unchanged: a proxy keeps `epic` + `proxy`.

### 5.4 The one-off deployment

One deployment task, run once after a vergil-tooling release carries §5.1
to §5.3, so that no epic is created unlabelled after the backfill:

1. **Create the labels.** In every repo of each of the six orgs, create
   `epic-development` and `epic-management` from their registry
   definitions, using the existing label code (`ensure_labels`, as
   `vrg-ensure-label` exposes it). Only these two labels: a full
   `vrg-ensure-label --sync` would also delete and recolour other labels,
   which this epic does not intend in the dormant orgs.
2. **Label the open epics.** In every epic home of the six orgs (each
   org's `.github`, and every private repo that homes its own epics), add
   to each open epic the category label §5.2 derives for it, when it is
   `epic-development` or `epic-management`. Nothing else is added or
   removed: `ad-hoc`, `proxy`, `epic` and the per-kind management labels
   already carry their categories.
3. **Report the rest.** An open epic whose labels are ambiguous (§5.2) is
   left unlabelled and listed in the result for a human.

The result comment records, per org, the repos the labels were created in
and the epics labelled, and then `Outcome: SUCCESS` only when both steps
finished for all six orgs. Re-running it is safe: creating a present label
and adding a present label are both no-ops.

## 6. Error handling

- An unregistered label: `ensure_labels` raises `UnknownLabelError`, and
  the creating command fails before it creates anything.
- An ambiguous label set at creation: `vrg-epic-create` refuses.
- A failed `gh` call in the one-off: reported with its error, and the
  deployment stays open (no `Outcome: SUCCESS`) until a re-run finishes it.
- An ambiguous existing epic: reported in the one-off's result, not
  labelled.

## 7. Left to the plan

- How the work splits into tasks, and the release task the deployment is
  blocked by.
- How the one-off is carried out: a procedure over `vrg-ensure-label` and
  `vrg-gh issue edit`, or a small reviewed script that uses the §5.2
  function. Either way it applies §5.2's rules exactly.
- How the one-off lists every repo and epic home of an org.
- The label colours.
- Which docs describe the categories: the epic convention, the site docs
  in `vergil-tooling/docs/site`, and the `epic-create` skill's notes. The
  documentation-review task (vergil-project/vergil-tooling#4843) sweeps them.
