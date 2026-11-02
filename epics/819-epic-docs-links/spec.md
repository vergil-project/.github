# Epic issues link their spec, plan and retrospective

- **Epic:** vergil-project/.github#819
- **Planning issue:** vergil-project/.github#815 (review history is in its
  comments)
- **Status:** draft, in design review
- **Repos:** vergil-tooling (the `vrg-epic-docs` command and the
  `vrg-epic-audit` check), vergil-claude-plugin (`epic-create` and
  `epic-retrospective`), `.github` (this spec and plan, and the fleet
  backfill)
- **Supersedes:** vergil-project/vergil-claude-plugin#837

## 1. Problem

An epic issue names its spec, plan and retrospective in plain text, if at
all: for example "Spec: `epics/558-steering-settings-review/spec.md`
(published by the documentation task)". There is no link, so a reader who
opens an epic cannot click through to the documents that explain it. Older
epics, from before the spec, plan and retrospective conventions existed,
give no sign of whether documents exist at all.

A closed epic should document itself from its issue: open it, and the top of
the body links every document the epic has, or says why one does not exist.

## 2. The Documents section

Every epic body starts with one standard section, wrapped in hidden markers
so tools can find and replace it:

```markdown
<!-- vrg-epic-docs -->
**Documents**

- Spec: [`epics/819-epic-docs-links/spec.md`](https://github.com/vergil-project/.github/blob/develop/epics/819-epic-docs-links/spec.md)
- Plan: [`epics/819-epic-docs-links/plan.md`](https://github.com/vergil-project/.github/blob/develop/epics/819-epic-docs-links/plan.md)
- Retrospective: pending
<!-- /vrg-epic-docs -->
```

- **Position:** the section is the first thing in the body. The rest of the
  body (description, `Planning:` line and so on) follows it unchanged.
- **Always three entries,** in the order Spec, Plan, Retrospective. Each is
  either a link or a note (§3).
- **Link form:** the link text is the document's repo-relative path; the
  target is a `blob` URL on the default branch of the repo that holds it,
  queried from GitHub rather than assumed. A document in another repo of the
  same org (§5.2) shows `<repo>/<path>` as its text.
- **One writer.** The section is only ever written by `vrg-epic-docs` (§5),
  never by hand, so its format lives in one place.

## 3. Entry rules

The entry for a document is a link when the document exists; otherwise the
note is chosen from the epic's state. A note is chosen because a document is
missing; dates only decide its wording. The rows are read top to bottom and
the first that matches decides the entry.

| Epic state | Document missing | Entry |
| --- | --- | --- |
| any | — (document exists) | link |
| open | any | `pending` |
| closed as not planned | any | `not written (closed as not planned)` |
| closed, created before 2026-07-12 | spec or plan | `not written (predates the spec and plan convention)` |
| closed, created before 2026-07-27 | retrospective | `not written (predates the retrospective convention)` |
| closed, otherwise | any | `not written` |

- **2026-07-12** is when epic vergil-project/.github#85 closed, after which
  `epic-create` published a spec and plan for every epic. **2026-07-27** is
  when epic vergil-project/.github#201 closed, introducing the retrospective
  bookend. The dates are fleet-wide constants, because every org runs the
  same tooling and skills.
- Both cutoffs compare the epic's **creation** date, so an old epic that
  stayed open past a cutoff still predates it: #40, created 2026-06-29 and
  closed 2026-08-15, gets `not written (predates the retrospective
  convention)`, as does #156. #171, closed as not planned, gets `not
  written (closed as not planned)` by the first-match rule.
- An open epic says `pending` even for a document it may never get; the
  entry is settled when the epic closes (§4).
- **Nothing is written after the fact.** No missing spec, plan or
  retrospective is authored to fill a gap; the note is the record.

## 4. When links are written

Links are written at **PR handoff**, not after merge.

| Moment | Who | Section change |
| --- | --- | --- |
| Epic created (`epic-create` step 2) | `epic-create` | Section added, all three `pending` |
| Docs PR handed off (`epic-create` step 8) | `epic-create` | Spec and plan become links |
| Retrospective PR handed off | `epic-retrospective` | Retrospective becomes a link |

- **Why at handoff:** the retrospective PR closes the epic, so no open task
  remains to edit the body after it merges; the docs PR has the same problem
  with sequencing. Writing at handoff needs no post-merge hook and runs in the
  session that already knows the epic and the `epics/<N>-<slug>/` path.
- **Accepted window:** until the PR merges, a new link returns 404. That is
  accepted; agents are close to finalizing their own PRs, which shrinks the
  window to minutes. A PR that never merges leaves a broken link, which the
  audit reports once the epic closes (§6).
- **Closing an epic without a retrospective** is not a normal path, so no
  skill step settles a `pending` entry at close. The audit reports a closed
  epic with a `pending` entry (§6), and `vrg-epic-docs backfill` settles it
  by §3.

## 5. `vrg-epic-docs` (vergil-tooling)

A new command, with a library module shared with the audit, that owns the
section's format.

### 5.1 `vrg-epic-docs set`

```text
vrg-epic-docs set --epic <owner>/<repo>#<N> [--spec PATH] [--plan PATH] [--retrospective PATH]
```

- Adds the section at the top of the body if it is missing, otherwise
  replaces the region between the markers. An entry not named on the command
  line keeps its current value; a new section starts all `pending`.
- `PATH` is relative to the epic's home repo, or `<repo>:<path>` for a
  document in another repo of the same org.
- It does **not** check that the file exists: at handoff it exists only on
  the PR branch.
- Re-running with the same arguments changes nothing (no edit is made when
  the body is already correct).

### 5.2 `vrg-epic-docs backfill`

```text
vrg-epic-docs backfill (--org <org> | --repo <owner>/<repo>) [--apply]
```

`--org` works on the org's `.github`; `--repo` works on a private repo that
keeps its own epics.

- **Which issues:** every issue labelled `epic` in the org's epic home, open
  and closed, found by label (some epics lack an `Epic:` title, for example
  vergil-project/.github#76, #80, #224 and #322). It skips container epics
  and epics whose title starts `[archived]`. A container epic is one that
  `epics.is_standing` recognises (ad hoc, intake, planning, deferred), one
  carrying the retired `standing` label, or one whose title starts
  `Epic (standing):` (for example vergil-project/.github#44).
- **Finding the documents,** for each epic `N`:
  1. the directory `epics/<N>-*/` in the home repo's default branch;
  2. otherwise, an `epics/<dir>/` path the body mentions that exists;
  3. otherwise, `docs/specs/…` or `docs/plans/…` paths the body mentions,
     in the home repo or another repo of the same org (written
     `<repo>/docs/specs/…`), that exist. A file under `specs/` is the spec,
     under `plans/` the plan. This covers epics such as
     vergil-project/.github#51–53, whose specs live in member repos.

  In a found directory, `spec.md`, `plan.md` and `retrospective.md` are the
  three documents. Other files in it (`audit-findings.md`, …) are ignored.
- **Ambiguity is not guessed.** Two candidate directories, or two candidate
  specs, for one epic: the epic is reported as needing a decision and left
  unchanged. The human settles it with `vrg-epic-docs set`.
- **Legacy document lines.** Hand-written document references outside the
  markers are removed when the section is written:
  - a `Documents:` line followed by Spec, Plan and Retrospective bullets (as
    in vergil-project/.github#558 and #819);
  - a standalone line starting `Spec:`, `Plan:` or `Retrospective:`, with
    or without a trailing "(published by …)" note (as in
    vergil-project/.github#51, #500 and #550).

  The dry run shows every line it would remove. No other body text changes.
- **Dry run by default.** Without `--apply` it prints, per epic, the section
  it would write and whether that is an add, a change or no change, then a
  summary count. With `--apply` it makes the edits.
- **Exit status:** 0 when every epic is correct or was corrected; 1 when any
  epic needs a decision or an edit failed. Each failure is named; a failed
  edit never stops the run silently.
- **Identity:** it edits issue bodies through the existing `lib/github.py`
  write path, inside `github.target_org(<org>)`, so the agent can run it for
  any org the App is installed in.

## 6. `vrg-epic-audit` check (vergil-tooling)

A new **Epic documents** finding section in `vrg-epic-audit`, over every
`epic`-labelled issue in the home, open and closed, with the same skips as
§5.2:

- **Every epic:** the section is present, has its markers, and has exactly
  the three entries, each a well-formed link or a note from §3.
- **Closed epics only:** every link resolves on its repo's default branch
  (contents API; a 404 is a finding), and no entry is `pending`.
- **Open epics:** links are not checked for resolution.

The audit has no per-epic body check today. It already lists closed epics
for `closed_epic_open_child`; the new check shares that listing, with the
§5.2 skips, rather than adding a second one. Findings are reported like the existing
sections and do not change the exit status. Pointing it at another org uses
the existing `--repo <org>/.github`.

## 7. Plugin skills (vergil-claude-plugin)

- **`epic-create`:**
  - step 2, after `vrg-epic-create`: run `vrg-epic-docs set --epic <ref>` to
    seed the section;
  - step 8, at the docs-PR handoff: run `vrg-epic-docs set --epic <ref>
    --spec epics/<N>-<slug>/spec.md --plan epics/<N>-<slug>/plan.md`.
- **`epic-retrospective`:** at the close-out handoff, run `vrg-epic-docs set
  --epic <ref> --retrospective epics/<N>-<slug>/retrospective.md`. The skill
  finds `<slug>` from the epic's existing `epics/<N>-*/` directory.
- **`migrate-repo`:** an epic it reconstructs gets its section from
  `vrg-epic-docs set` like any other.

These steps run as the user agent; editing an issue body is an ordinary
agent action, unlike PR submission.

## 8. Fleet backfill

The epic closes only once every org is clean. The backfill runs as
**deployment tasks** under this epic, filed in vergil-project/.github and
not PR-workable. Each:

1. checks its precondition: the released `vrg-epic-docs` is installed;
2. lists the org's private repos and records which, if any, keep their own
   epics (a private repo self-homes its epics, §5.2 covers only the org's
   `.github`). Each one found gets steps 3–6 as well, run with `--repo
   <owner>/<repo>` in place of `--org <org>`;
3. runs `vrg-epic-docs backfill --org <org>` as a dry run and records the
   summary;
4. settles every epic reported as needing a decision with
   `vrg-epic-docs set`, after the human decides it;
5. runs it with `--apply`;
6. runs `vrg-epic-audit --repo <org>/.github` and records a clean Epic
   documents section as its `Outcome: SUCCESS`.

Order: **vergil-project first**, where the format is proven on its 77
epics.
Then one task each, blocked on the vergil-project task, for
**logical-minds-foundry, mq-rest-admin-project, mnemosys-project,
diogenes-project and vergils-nemesis**. The personal `wphillipmoore`
account has no `.github` and therefore no epic home; its task checks that
none of its repos carries an `epic`-labelled issue and records the result.
Nothing in another org is linked to this epic; the sweeps only edit issue
bodies there.

The expected vergil-project result, computed on 2026-11-02 under §3 and the
§5.2 skips, is 77 epics. The vergil-project task checks its dry run against
this, allowing for epics created or closed since:

| Spec | Plan | Retrospective | Epics |
| --- | --- | --- | --- |
| link | link | link | 34 |
| link | link | predates the retrospective convention | 16 |
| predates the spec and plan convention | predates the spec and plan convention | predates the retrospective convention | 8 |
| link | link | pending | 5 |
| link | predates the spec and plan convention | predates the retrospective convention | 5 |
| pending | pending | pending | 4 |
| closed as not planned | closed as not planned | closed as not planned | 2 |
| link | link | closed as not planned | 1 |
| not written | not written | predates the retrospective convention | 1 |
| not written | not written | not written | 1 |

## 9. Documentation

- vergil-tooling `docs/site/docs/standards/github-issues.md`: the section's
  format and entry rules in §Epics and §Creating an epic, and the new
  invariant in §Compliance invariants.
- vergil-tooling `docs/site/docs/reference/cli-tools-overview.md`:
  `vrg-epic-docs`, and the new `vrg-epic-audit` section.
- vergil-claude-plugin `skills/epic-create/SKILL.md`,
  `skills/epic-retrospective/SKILL.md` and `skills/migrate-repo/SKILL.md`.

The documentation-review task (vergil-project/vergil-tooling#3895) sweeps
for anything else.

## 10. Out of scope

- Writing a missing spec, plan or retrospective for any epic.
- Closing the 404 window before a PR merges.
- Linking anything across orgs.
- Changing where documents live (`epics/<N>-<slug>/`) or what they contain.
- Any section for task issues; this is for epics only.
