# Visibility-based epic home

**Epic status:** proposed
**Convention amended:** `vergil-project/.github#40` (epic/task convention, §3.2)
**Related:** `vergil-project/.github#85` (centralize epics/ad-hoc)

## Problem

The epic/task convention hard-codes every epic's home to `<org>/.github`.
That repo is **public**. When a member repo is private — work kept off the
public surface until it is ready to share — its epics, plans, and specs would
still land in the public `.github`, leaking the existence and content of the
private work (issue titles, spec text, roadmap entries).

A private repo therefore cannot participate in the epic/task convention
without disclosure. We want private repos to keep their epics **in their own
issues list**, fully self-contained, so that nothing about them appears on any
public or org-level surface (issues, roadmap, docs site).

## Goals

- A private member repo homes its epics in **itself**, not in `<org>/.github`.
- Zero routing configuration — the home is derived automatically from
  repository visibility, so there is nothing to set, remember, or drift.
- Fully backward-compatible: every public repo behaves exactly as it does
  today, and a fully-private org behaves exactly as it does today.
- Self-contained: a private repo's epics, tasks, roadmap, and audit stay
  scoped to that repo and never surface in an org-level artifact.

## Non-goals

- **No mixed private-`.github`/public-repo topology.** A private `.github`
  is read as "the whole org is private," and everything routes to `.github`.
  We deliberately do not support a private `.github` fronting public member
  repos.
- **No new relocation tool.** Visibility flips reuse the existing
  `vrg-epic-move`.
- **No config key.** Routing is automatic; `vergil.toml` gains no epic section.
- Tasks do not move. Only the *epic home* varies (see Data model).
- **No hard-linking a more-public task under a less-public epic.** A public
  task may not be a native sub-issue / `Parent:` child of a private epic — that
  would leak the private repo's name into a public issue and break cross-boundary
  roll-up. Such dependencies use a soft reference from the private side (see
  Cross-visibility linkage).

## Design principle: explicit target, derived home

Two repos are in play for any epic:

- the **target** — the repo the epic is *about* / whose work it tracks;
- the **home** — the repo where the epic *issue physically lives*.

Epic-creating and epic-reporting commands take the **target explicitly** —
`--repo <owner>/<repo>` — defaulting to the current repo but never merely
*induced* from the working directory. The context of the discussion, not where
you happen to be checked out, determines what an epic is for; you may be in the
lab and realize the work belongs to the private repo, and you file it for the
private repo without moving.

Given the explicit target, the **home is derived automatically** by the
resolver below. This keeps both properties the initiative wants: **explicit
about the target, automatic (leak-safe) about the home** — you never have to
remember "this one is private, route it elsewhere."

## Data model — what varies and what does not

- **Epic home varies** by the resolver, keyed on the explicit target repo.
- **Tasks always live in their member repo** (1:1 with their PR), unchanged.
  This is required: GitHub only auto-closes a same-repo issue on merge, so a
  task must be co-located with its PR for `Closes #N` and the `issues.closed`
  roll-up to fire. In a private org the member repos are private anyway, so
  there is no leak. "Everything goes into `.github`" therefore means **epics**,
  not tasks.

## The resolver

A single, config-less, visibility-driven function whose input is the
**explicit target repo**. Visibility is treated as **binary: `PUBLIC` vs.
not-`PUBLIC`** — a repo whose visibility is anything other than `PUBLIC`
(including `INTERNAL`) counts as private for routing, because it is not
publicly visible, which is the whole concern.

```text
resolve_epic_home(org, target_repo):
    if target_repo == ".github":          return f"{org}/.github"   # already in .github
    if is_public(f"{org}/{target_repo}"): return f"{org}/.github"   # public target -> central (today)
    # target is private from here down:
    if is_public(f"{org}/.github"):       return f"{org}/{target_repo}"  # THE new case: self-contained
    return f"{org}/.github"                                          # private org -> central (unchanged)
```

Case table (target → home):

| `.github` | target repo | epic home | new? |
|---|---|---|---|
| public | public | `<org>/.github` | no — today's behavior |
| public | **private** | **`<org>/<target>`** | **yes — the only new case** |
| private | (any) | `<org>/.github` | no — private org, unchanged |

**Key insight:** the private-*org* case needs zero code change — the code is
already visibility-blind and always writes `.github`. The only genuinely new
behavior is *public `.github` + private target -> self*.

### Robustness requirements

- **Fail loud, never guess.** If a visibility lookup fails or the target repo
  does not exist (network, auth, typo, missing repo), the resolver raises — it
  must **not** fall back to `.github`, because that would leak a private repo's
  epic into a public repo. A hard error is correct; a silent default is a
  disclosure bug.
- **Print the resolved home.** The creating command echoes the derived home
  and its visibility (e.g. `-> epic home: vergil-project/.github [PUBLIC]`) so
  the operator sees where the issue will land before it is created.
- **Cache per run.** Visibility for `<org>/.github` and each target repo is
  looked up once per process and memoized, so audit/roadmap sweeps do not
  refetch. `.github` visibility is only probed when the target is private
  (public targets short-circuit before that check).

### Placement

- `resolve_epic_home(org, target_repo)` lives in `lib/epics.py`, alongside the
  epic vocabulary it already owns.
- The visibility probe (`is_public` / a `repo_visibility` helper) lives in
  `lib/github.py` next to `detect_org`, with per-run memoization.

## Consumers

The literal `.github` lives in exactly four places today; each swaps the
literal for the resolver, and the two creating/reporting entry points gain an
explicit `--repo <owner>/<repo>` target (cwd as default):

| Consumer | Today | After |
|---|---|---|
| `bin/vrg_epic_create.py:55` | `repo = f"{org}/.github"` | `--repo` target → `resolve_epic_home(org, target)`; print home |
| `lib/epics.py:293-342` `ensure_adhoc_epic` | `f"{owner}/.github"` | resolver on the target repo (via `vrg-adhoc-epic --repo`) |
| `lib/epic_audit.py` (invariants) | `f"{org}/.github"` | resolved home + per-epic rule (see Audit); `--repo` on `vrg-epic-audit` |
| `lib/roadmap.py:31-77` | reads `f"{org}/.github"` | reads the resolved home for the target; `--repo` on `vrg-roadmap` |

`vrg-epic-link` / `vrg-epic-move` / `vrg-epic-rollup` are already ref-driven
(home-agnostic) and need no logic change; only their `.github`-centric
docstrings are refreshed.

## Audit and roadmap stay self-contained

Both tools take their source from `resolve_epic_home(org, target)`:

- Target a **public** repo (or default cwd in a public repo) → source is
  `.github` → org-level view, exactly as today.
- Target the **private** repo → source is that repo → audits/plots only its own
  epics. Nothing private ever lands in an org-level artifact.

### Audit invariant generalization

Today `epic_audit.epic_outside_dotgithub` (L200-226) flags *any* `epic`-labelled
issue not in `.github`. New per-epic rule:

> Flag an epic outside `.github` **only if its repo is public**. An epic in a
> private repo (with a public `.github`) is legitimately homed and must not be
> flagged. Determine the repo's visibility via the memoized probe; if the probe
> **errors, fail loud** — never silently skip, or a genuine leaked-out public
> epic would be masked.

**Org audit stays in the public world.** The org-level audit (target `.github`)
validates `.github` + public repos and merely *refrains* from flagging
private-repo epics via the rule above. It does **not** enumerate the org's
private repos to check their internal hygiene. A private repo's own hygiene is
checked by running `vrg-epic-audit --repo <private>` against it. This keeps
private repos out of org-level operations, consistent with self-containment.

Invariant 2 (`stray_dotgithub_issue`, L229-262) is unchanged in the normal
public-`.github` case; in the private-org case `.github` legitimately holds
everything, which is already how it behaves today.

### Roadmap scope

The org roadmap (source `.github`) **omits private-repo epics by design** —
that absence is intentional (self-containment / no leakage), not a defect. A
private repo's epics appear only in its own roadmap (`vrg-roadmap --repo
<private>`).

## Cross-visibility linkage (asymmetric rule)

Once a private repo self-hosts its epics, a private epic may need work in
another repo — including a change in a *public* repo of the same org. Hard
machine-linkage (native sub-issue / `Parent:` reflink) between a public task and
a private epic fails two ways: the public task's body would name the private
repo (reverse leak), and the public repo's `issues.closed` roll-up runs with a
repo-scoped token that cannot read the private epic (broken roll-up).

The failure is **asymmetric**, and a single rule captures it:

> **A task may hard-link to an epic only if the task's repo is no more publicly
> visible than the epic's home** (order: `PUBLIC` > `INTERNAL` > `PRIVATE`; the
> task must be ≤ the home). A less-visible child under a more-visible parent is
> always fine; the reverse never is.

| Epic home | Task repo | Hard link? |
|---|---|---|
| public | public | ✅ |
| public | private | ✅ (private child names a public parent — no leak; roll-up reads public `.github`) |
| private | private | ✅ |
| private | **public** | ❌ (leak + broken roll-up) |

This generalizes the existing cross-org non-goal ("don't link across a trust
boundary") from the org boundary to the visibility boundary.

**Sanctioned pattern for cross-visibility work.** The public change is filed as
its own standalone public task (under the public repo's normal epic or the
`.github` ad-hoc epic), and the **private epic references it from its own
(private) body** via a soft `Blocked-by: <org>/<public-repo>#M`. That reference
lives in the private repo, visible only to those with access, and points *at* a
public thing — so nothing leaks. The dependency is captured as a weak link by
reference, not a public-facing child.

**Enforcement.** `vrg-epic-link` gains a fail-loud visibility-boundary guard:
after ref parsing (and the existing `single_target_org` check), it refuses to
link a `PUBLIC` task under a non-`PUBLIC` epic home, with a message pointing at
the soft-reference pattern. Binary `is_public` suffices (refuse when the task is
`PUBLIC` and the epic home is not).

## Visibility flips

Flipping a repo public↔private changes its resolved home, so its existing
epics must relocate. This reuses `vrg-epic-move` (which moves an epic and
re-points its linked sub-issues) — **no new tool**. A flip is: for each of the
repo's epics, `vrg-epic-move` from the old home to the newly-resolved home.
The procedure is documented; a thin `--to-home` convenience wrapper is
explicitly out of scope unless it proves painful in practice.

## Out-of-repo work (tasks in this epic, not code in one PR)

- **Convention `vergil-project/.github#40`** (and its `spec.md`) amended from
  "epics live in `.github`" to "epics live in their *resolved* home," with the
  explicit-target/derived-home principle, the case table, and the non-goal
  about the unsupported topology.
- **Skills** (in the marketplace repo, not vergil-tooling): `epic-create` and
  `migrate-repo` prose amended to teach the resolved home and the `--repo`
  target. `epic-create`'s preflight "confirm the org's epic home
  (`<owner>/.github`)" becomes "confirm the resolved epic home for the target."

## Testing

- Update tests that pin `.github`: `test_vrg_epic_create.py:27`,
  `test_epics.py:475/489`, `test_epic_audit.py:165/178`.
- New cases for every routing branch (public target, private target,
  private-org), the `.github`-itself short-circuit, `--repo` targeting vs. cwd
  default, and the fail-loud paths (probe error, nonexistent target).
- Audit: a private-repo epic is not flagged; a public-repo epic outside
  `.github` is flagged; probe-error fails loud.
- Memoization: assert visibility is probed once per repo across a sweep.

## Task breakdown (this epic)

1. **Resolver + visibility probe** — `resolve_epic_home(org, target_repo)` in
   `lib/epics.py`; memoized `repo_visibility`/`is_public` in `lib/github.py`;
   fail-loud (probe error and nonexistent target); unit tests for every branch.
   Foundational; the rest depend on it.
2. **`vrg-epic-create --repo`** — explicit target (cwd default), wire to the
   resolver, print the resolved home + visibility.
3. **`vrg-adhoc-epic --repo` / `ensure_adhoc_epic`** — resolver on the target.
4. **Generalize the audit invariant** — per-epic public-only flagging with
   fail-loud probe; org audit stays public-world; `vrg-epic-audit --repo`;
   render text updated.
5. **Roadmap sourcing** — read the resolved home for the target;
   `vrg-roadmap --repo`; org roadmap omits private by design.
6. **Bootstrap the private repo's epic machinery** — a repo cannot self-host
   epics until it has the `epic-rollup.yml` workflow, the `epic` + operational
   labels (`vrg-ensure-label`/`labels.json`), and a `vergil.toml`. Named task,
   **precondition of migrating the private repo's epic**. Writing-plans
   confirms whether a standard "install standard workflows into a repo" path
   exists or the steps are manual.
7. **Amend convention `#40`** spec/issue.
8. **Amend skills** (`epic-create`, `migrate-repo`) in the marketplace repo.
9. **Document the visibility-flip procedure** using `vrg-epic-move`.
10. **`vrg-epic-link` visibility-boundary guard** — refuse to hard-link a
    `PUBLIC` task under a non-`PUBLIC` epic home (fail-loud, points at the
    soft-reference pattern). Convention #40 documents the asymmetric rule and
    the `Blocked-by:`-from-the-private-side pattern.

## Follow-ons (tail of epic / after)

- **Docs build** so the public site never renders a private repo's docs — the
  self-contained principle extended to the published documentation surface.
  Picked up at the tail of the epic.
- Migrating the new private repo's own epic out of `.github` into itself, once
  the repo exists, is an **early task in that private repo's own plan** (paired
  with task 6's bootstrap), not a task of this tooling epic.
