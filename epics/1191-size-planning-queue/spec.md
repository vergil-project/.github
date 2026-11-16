# Size the planning queue — design

- **Epic:** vergil-project/.github#1191
- **Planning issue:** vergil-project/.github#1185 (review history is in its
  comments)
- **Status:** draft, in design review

## 1. Intent

Every planning-queue item carries an **estimated epic size** and an
**estimated task count**, so that people and agents can sort and filter the
planning queue by size as well as by age.

The size is a guess at how big the item's epic would be once `epic-create`
expands it, counted in tasks, bookends and operational tasks included. It is
recorded twice, each copy for its own reader:

- an `epic-size:<S|M|L|XL>` **label**, which GitHub's issue lists and the
  tooling's filters read;
- a dated **sizing comment** with the task count, a confidence and a one-line
  reason; the latest one is the current estimate and earlier ones are the
  history.

One command writes both, and the queue view reads both and refuses to guess
when they disagree.

## 2. Why

Work reaches two well-served paths today: small one-PR items go to ad-hoc
epics and move fast, and large architectural epics (double-digit task counts,
phased releases, days or weeks of work) get the maintainer's main focus. The
planning items between the two are too big for ad hoc and too small for
architectural attention, so they wait.

The aim is a regular cadence at the small end of the planning queue: list the
smallest items, oldest first, take the oldest, run `epic-create` and
`epic-implement` on it within a few hours, or defer it if it turns out
bigger. That session runs in parallel with `epic-implement adhoc` and the
large epics. Every planning item still goes through `epic-create`, attended;
nothing here is about unattended agents.

Sorting by age alone cannot support that. The queue needs a size.

## 3. Goals, non-goals, anti-goals

### Goals

- Every planning item can carry a size and a task count, written by one
  command, `vrg-intake-route <ref> size`.
- Routing into planning can size the item in the same step, and the
  `epic-review` skill always does.
- `vrg-queue planning` sorts by age (as now), by size then age, or by task
  count, filters by size, and shows the size and task count on every line
  and in `--json`.
- A deferred item keeps its size and comes back to planning sized.
- The items open on 2026-11-16 are sized from the recorded estimates
  (the backfill).

### Non-goals

- **Recording an epic's actual task count against its estimate.** The data
  already exists: each planning issue's `Epic:` marker names its epic, and the
  epic's sub-issues are the actual count. A later recalibration can join the
  two without any new marker.
- **Sorting the deferred view by size.** It shows the size; its order stays
  least recently deferred first.
- **Sizing intake items.** Intake is uncurated; a size is given when an item
  is routed to planning.
- **Any change to how `epic-create` chooses what to plan.** The human still
  chooses; the size is a reading aid.
- **Unattended planning or implementation.**

### Anti-goals (the epic fails if any of these happen)

- The tooling picks one of two disagreeing sizes, or treats a malformed
  sizing comment as absent, instead of reporting it.
- An unsized item is hidden from a listing that does not filter by size.
- A comment from an untrusted author changes an item's size or makes the
  queue view fail.
- A made-up size: a caller with no sizing judgment is forced to supply a
  number.

## 4. Current state (facts the design builds on)

### vergil-tooling

- **`vrg-intake-route`** (`bin/vrg_intake_route.py`, `lib/intake_route.py`)
  routes one container-epic item: `drop`, `defer`, `intake`, `one-off`,
  `planning`, `split`. `ROUTES` (`lib/intake_route.py`) says which routes each
  container (intake, planning, ad hoc, deferred, unparented) allows, and every
  route runs under the item owner's App token (`github.target_org`). Every
  existing route moves or closes the item.
- **The planning route** reparents an intake, deferred or unparented item, or
  a follow-on brainstorm under a development epic, onto the planning epic of
  the same queue home. `defer` and `planning` reparent in place, so an item's
  labels survive them. A `one-off` transfers the issue (comments move with
  it; labels survive only if they exist in the target repo) or, when GitHub
  refuses the transfer, recreates it (`_recreate` carries every registered
  label except the retired ones and `in-progress`, and no comments).
- **Labels.** `data/labels.json` is the canonical registry, and
  `labels.ensure_labels(repo, names)` is the point-of-use primitive every
  label-applying command calls: it creates a registered label in the repo the
  first time it is applied there and refuses an unregistered one
  (`UnknownLabelError`). `labels.sync_labels`, used by `vrg-ensure-label
  --sync` and `vrg-repo-init`, applies the whole registry. So a new label
  needs a registry entry; the fleet sync only tidies.
- **`vrg-queue`** (`bin/vrg_queue.py`, `lib/queue_view.py`) lists the
  intake, planning and deferred queues and the ad-hoc tasks, read-only. A
  planning `QueueItem` has `ref`, `title`, `created_at`, `state`, `note` and
  `epic`; the text line is `<slug>  <age>  <state>  <title>[  — <note>]`;
  planning is oldest first, deferred least recently deferred first. A queue
  that cannot be read is named on stderr and the run exits 1; a bare
  `--json` run then prints nothing on stdout.
- **The skills' error rule.** `epic-review` step 2 ("Collect the queue") and
  `epic-create`'s no-argument summary forgive exactly one exit 1 from
  `vrg-queue`, the missing-queue message; any other is a real failure that
  stops them.
- **Comment parsing** in `queue_view` (`parse_latest_deferral`,
  `parse_planning_markers`) reads every comment body, **whatever its author**.
- **`lib/trusted_authors.load_trusted_authors()`** returns the configured
  trusted authors (VM file, else the host `identities` stanza) and raises
  `TrustedAuthorsNotConfiguredError` rather than defaulting.
- **Planning items live in several repos**, not only `.github` (for example
  vergil-tooling#3840 and vergil-claude-plugin#828), so the labels must work in
  every repo that holds a planning or deferred item.

### vergil-claude-plugin

- **`epic-review`** (`skills/epic-review/SKILL.md`): intake mode routes items
  with `vrg-intake-route … planning`; planning mode reviews waiting items
  oldest first (leave, defer, drop, one-off) and ends with a sorted list from
  `vrg-queue planning`; its notes say "Do not invent fields, labels or
  priority scores."
- **`epic-create`** files a free-text idea's planning issue with
  `vrg-issue-create --epic planning`, not through the planning route, and
  marks it `in-progress` at once.

### The calibration data

Of 70 completed development epics in vergil-project/.github on 2026-11-16,
20 had up to 9 tasks, 29 had 10–19, 15 had 20–39 and 6 had 40 or more.
Calendar time barely tracked task count (most closed within 1–4 days), so
task count is the proxy, not elapsed days. The 2026-11-16 estimates for the
items then waiting in planning are recorded as a TSV in a trusted comment on
the planning issue.

## 5. Approaches considered

- **A `size` route in `vrg-intake-route` (chosen).** It reuses the route
  layer's container checks, the item-owner token handling and the error
  reporting, and sits beside the routes a queue review already uses.
- **A separate `vrg-size` command.** It would duplicate the container checks,
  the token handling and the planning-epic lookup that a route gets for free.
- **The size in the issue body.** Body sections are already owned by tools
  (`vrg-epic-docs`, `vrg-security-record`), a body keeps no history, and a
  body edit is invisible in an issue list.
- **GitHub Projects custom fields.** The one place GitHub has a number field,
  but nothing in the fleet uses Projects, the `vrg-gh issue` tooling cannot
  see it, and it would be a second source of truth.
- **A label alone, or a comment alone** (from the planning issue): a label
  alone has no task count and no reasoning; a comment alone is invisible in
  GitHub's issue lists.

## 6. Design

### 6.1 The scale

| Label | Estimated tasks | Rough effort |
| --- | --- | --- |
| `epic-size:S` | 1–9 | plan and implement in a sitting (hours) |
| `epic-size:M` | 10–19 | about a day |
| `epic-size:L` | 20–39 | several days, phased releases |
| `epic-size:XL` | 40+ | a wave of several epics |

The count includes the standard bookends (documentation, documentation
review, retrospective) and any deployment and validation tasks. **The task
count is the estimate; the size is derived from it** by this table, so every
count has exactly one size and no caller states both. The name
`epic-size` is deliberate: Kubernetes' `size/*` labels PRs by lines changed,
a different meaning, so a bare `size` is ruled out.

### 6.2 The labels (vergil-tooling)

The four labels go in `data/labels.json`, one colour family, each description
naming its band (for example "Planning estimate: epic of 10–19 tasks"). Every
command that applies one calls `ensure_labels` first, so a repo that has never
seen it gets it at point of use. The next repo-config sync adds them to the
rest of the fleet; nothing waits on that.

### 6.3 The sizing comment

One comment per sizing, posted by the tool, which derives the size from the
task count:

```text
Sized 2026-11-16: epic-size M, about 14 tasks.
Epic-size: M
Estimated-tasks: 14
Confidence: med
Reason: <one sentence>
```

- A **sizing comment** is a comment by a trusted author whose first line
  matches `Sized YYYY-MM-DD:`. Its `Key: value` lines start at the start of a
  line, like the deferral record's `Revisit when:`.
- `Epic-size` is `S`, `M`, `L` or `XL`; `Estimated-tasks` a bare positive
  integer in that size's band; `Confidence` `high`, `med` or `low`; `Reason`
  non-empty text on one line.
- The **latest** sizing comment is the current estimate; earlier ones are the
  history.
- **Only trusted authors count.** The parser reads each comment's author and
  counts only those `load_trusted_authors()` lists (after stripping a leading
  `app/`), the rule `epic-create` already applies when resuming. A sizing
  comment from anyone else is ignored, so it can neither change a size nor
  make the view fail. Trusted authors that are not configured, or are
  configured invalidly, fail the run loudly; there is no default.
- **A malformed latest sizing comment is an error**, never skipped in favour
  of an earlier one: a missing key, a value outside its set, a non-integer or
  an `Epic-size` that is not the one the task count derives (a comment
  written by hand).
- **The cleared marker.** A trusted comment whose first line matches
  `Size cleared YYYY-MM-DD:` (posted by a one-off, 6.7) ends the item's
  sizing history: only sizing comments after the latest such marker count.

### 6.4 The item's size and its consistency check (vergil-tooling)

`queue_view` derives one **size record** per item from its labels and its
latest trusted sizing comment after the latest cleared marker (6.3):

| Labels | Latest sizing comment | Result |
| --- | --- | --- |
| none | none | **unsized** |
| one `epic-size:X` | well-formed, `Epic-size: X` | **sized** |
| none | present | **error**: comment without label |
| one | none | **error**: label without comment |
| one `epic-size:X` | `Epic-size: Y`, Y ≠ X | **error**: disagreement |
| two or more | any | **error**: several labels |
| any | malformed | **error**: malformed comment |

An error never resolves to a size. The listings show it (6.8), each such
item and what is wrong is named on stderr, and the run exits **2**, a status
of its own (6.8): the listing is complete and usable, and some items in it
need fixing. Exit 1 keeps its meaning, a queue that could not be read.

### 6.5 `vrg-intake-route <ref> size` (vergil-tooling)

```text
vrg-intake-route <ref> size --tasks N [--confidence high|med|low] --reason TEXT
```

- **Allowed from planning and deferred**, in any planning state: re-sizing
  an in-progress item is a normal re-estimate. Refused, with the reason, from
  intake, ad hoc, a development epic or an unparented item, through the same
  `ROUTES` table as every other route. It is the one route that neither moves
  nor closes the item.
- The size is derived from `N` (6.1).
- **Refused before any change** when `N` is not a positive integer, or the
  reason is empty or spans several lines.
- `--confidence` defaults to `med`; the comment always carries the line.
- **Order of writes:** post the sizing comment first (it is the record), then
  set the label in one edit that adds `epic-size:X` and removes every other
  `epic-size:*` label on the item, after `ensure_labels`. If the label edit
  fails after the comment posted, the route raises with the item's state (a
  comment without its label, which `vrg-queue` reports as an error) and the
  exact command that finishes it: re-running the same `size` command is safe,
  since a second sizing comment with the same values only adds history.
- Prints `Sized <slug>: epic-size X, about N tasks.` on success.

### 6.6 Sizing on the way into planning (vergil-tooling)

```text
vrg-intake-route <ref> planning [--tasks N [--confidence …] --reason TEXT]
```

- `--tasks` and `--reason` are given **together or not at all**; either one
  alone is refused before any change. `--confidence` needs `--tasks`.
- **Optional in the tool.** Without them the route behaves as today and the
  item arrives unsized (`?`). Callers with no sizing judgment to hand — the
  follow-on-brainstorm handoff, a bare revival of a deferred item — never have
  to make up a number. The `epic-review` skill always passes a size (6.9).
- With them, the route reparents the item first, then applies the same sizing
  as 6.5, validated before the reparent so a bad size refuses before any
  change. A sizing that fails after the reparent raises naming the item (now
  in planning, unsized or half-sized) and the `size` command that finishes it.
- A deferred item that already carries a size keeps it; giving `--tasks` on
  its way back re-sizes it.

### 6.7 A one-off clears the size (vergil-tooling)

An item routed out as a one-off is no longer an epic candidate, so its
estimate ends there, on every path:

- **Recreate:** `_recreate` adds the `epic-size:*` family to the labels it
  does not carry, beside the retired labels and `in-progress`.
- **Transfer, or a one-off within the item's own repo:** once the item has
  settled under its target epic, the route removes every `epic-size:*` label
  from the final issue, the way it already removes `in-progress`.
- **Every path:** when the item carried a size (a label or a sizing comment),
  the route posts a trusted cleared marker on the final issue,
  `Size cleared YYYY-MM-DD: moved out of planning as a one-off.`, so the
  carried sizing comments no longer count (6.3). A recreated issue carries no
  comments, so it needs only the label rule.
- A failure here raises the same way as the route's other post-move steps,
  naming the item and the command that finishes it.

A later return to planning or deferred starts unsized, and is sized afresh.

### 6.8 `vrg-queue` (vergil-tooling)

**Planning view:**

- `--sort age|size|tasks`, default `age` (today's order, oldest first).
  `size` orders S, M, L, XL, then unsized, then items in error, oldest first
  within each group; `tasks` orders by estimated task count ascending, ties
  oldest first, then unsized, then items in error.
- `--size S,M` filters to the named sizes; the token `unsized` selects
  unsized items (for example `--size S,unsized`). An unknown token is refused.
  Items in error are **always listed**, whatever the filter, so a filter never
  hides a problem.
- Each text line shows the size and task count, as `M/14`, `?` for unsized
  and `!` for an item in error, whose note then names the error instead of
  the latest comment.
- `--json` adds `epic_size` (`"S"` … `"XL"` or `null`), `estimated_tasks`
  (integer or `null`), `confidence` (string or `null`) and `size_error`
  (string or `null`) to each item.
- `--sort` and `--size` apply to the planning view only, and are refused with
  any other view, including a bare `vrg-queue`.

**Deferred view:** each line and JSON item carries the same size fields, read
the same way; its order does not change.

**Intake and ad-hoc views:** unchanged.

**Exit status**, for every view and a bare `vrg-queue`:

- **0:** every queue was read and no item is in error; the output is
  complete.
- **2:** every queue was read and one or more items are in error (6.4); the
  output is complete and usable, `--json` included, and each item in error
  is named on stderr.
- **1:** a queue could not be read; as today, a bare text run lists what it
  read and a bare `--json` run prints nothing.

Exit 1 wins over 2 when both apply.

### 6.9 The skills (vergil-claude-plugin)

- **Exit 2 is not a failure.** `epic-review` step 2 (every mode) and
  `epic-create`'s no-argument summary treat a `vrg-queue` exit 2 as "show the
  list, flag the `!` items, carry on"; exit 1 other than the missing-queue
  message stays a real failure.
- **The notes' rule** "do not invent fields, labels or priority scores" gains
  one exception: the `epic-size:*` family, written only through
  `vrg-intake-route`.
- **Planning mode** sizes every waiting item it reviews that is unsized or in
  error, and re-sizes a sized item when the review shows the estimate is off.
  Each item's paragraph recommends a task count (with the size it derives),
  a confidence and a one-line reason for the human to confirm with the
  batch; the confirmed sizes are applied with
  `vrg-intake-route <ref> size --tasks …`. Sizing is not a
  fifth outcome; it rides alongside leave (and is skipped for an item being
  deferred, dropped or moved out).
- **Its end-of-pass list** can be shown by size (`vrg-queue planning --sort
  size`) when the human asks, oldest first being the default.
- **Intake mode** always passes `--tasks` and `--reason` when it
  routes an item to planning, recommending them in the item's paragraph.
- **Deferred mode** passes a size when it revives an unsized item to
  planning.
- **`epic-create`'s "Choosing what to plan next"** gains one line: `vrg-queue
  planning --sort size --size S` lists the small items oldest first, for the
  small-end cadence.

### 6.10 Rollout and backfill

1. The vergil-tooling changes land and ship in a vergil-tooling release (a
   human-gated precondition), and are deployed to the host and VMs.
2. The vergil-claude-plugin changes land and ship in a plugin release.
3. **Backfill**, an operational task blocked by the tooling deployment: for
   each line of the 2026-11-16 TSV on the planning issue whose item is still
   open in planning or deferred, run `vrg-intake-route <ref> size --tasks
   <n> --confidence <c> --reason "<reason>"`, and check the size it derives
   against the TSV's size column; a mismatch is reported, not applied. Items
   closed or moved out since are skipped and listed in the results comment.
   The TSV's own line for this planning issue is skipped too: it closes when
   this epic's docs land. Then `vrg-queue planning --sort size` must exit 0
   and list no item in error; the remaining unsized items are listed for the
   next `epic-review planning` pass.

## 7. Failure model

- **Disagreeing or malformed data** (6.4) is listed, flagged `!`, named on
  stderr, and the run exits 2. It is never resolved silently, and it never
  stops the review that fixes it.
- **A half-applied sizing** (comment posted, label edit failed) raises
  naming the state and the finishing command; until then the item shows as an
  error.
- **Trusted authors not configured** fail `vrg-queue planning` and
  `vrg-queue deferred` with the loader's message; the views do not fall back
  to counting every comment.
- **A label removed or added by hand** shows as "comment without label" or
  "label without comment" while the item is in planning or deferred;
  re-running `size` fixes it.
- **A one-off** clears the size on every path (6.7), so a moved item never
  carries a stale or half-carried estimate.

## 8. Constraints

- Both repos' changes go through the normal worktree and PR flow, and are
  validated with `vrg-container-run -- vrg-validate`.
- Tooling and plugin releases are human-gated; agents never cut one.
- Every public text this epic writes (comments, labels) is neutral: a
  security item's sizing reason must not describe its finding.
- The parsing follows the existing `queue_view` conventions: a line-anchored
  marker, the latest wins.

## 9. Related work

- Epic vergil-project/.github#176 and #508: the container epics and
  `vrg-intake-route`.
- vergil-project/.github#950: the planning-item lifecycle; it touches
  `vrg-queue` and `vrg-intake-route` too, so its plan should rebase onto this
  one.

## 10. Left to the builder

- The label colours and the exact descriptions.
- Column widths and alignment in the `vrg-queue` text line, within the
  `M/14`, `?`, `!` forms above.
- How the backfill reads the TSV (a small one-off script in the results, or
  the commands run one by one); the result must name every line's outcome.
- Where the size-record derivation lives inside `queue_view` and how
  `intake_route` shares the band and format constants with it.
