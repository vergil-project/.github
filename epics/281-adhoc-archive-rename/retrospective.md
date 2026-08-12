# Reclassify per-quarter ad-hoc buckets as archives — Retrospective

- **Epic:** vergil-project/.github#281
- **Status:** Complete (2026-08-11)
- **Partners:** [`spec.md`](./spec.md), [`plan.md`](./plan.md)

## §0 At a glance

We set out to stop the terminal per-quarter ad-hoc buckets from masquerading as
epics. They were titled `Epic (ad hoc): <repo> — YYYY-Qn` and carried the `epic`
label, so every `label:epic` view was cluttered with historical quarter buckets.
What shipped: a new `archive` label, the buckets retitled to
`Archive (ad hoc): <repo> — YYYY-Qn` and relabelled `archive` + `ad-hoc`, a
self-healing creation path, an idempotent org-wide `vrg-adhoc-epic normalize`
sweep, one audit fix, updated standards docs, and a one-time migration of the
whole existing backlog. The live `Epic (ad hoc): <repo>` epic was left untouched.

### Work delivered

| PR / action | Repo | Task | What it did |
|---|---|---|---|
| [#288](https://github.com/vergil-project/.github/pull/288) | .github | #282 | Spec + plan (opening bookend) |
| [#2748](https://github.com/vergil-project/vergil-tooling/pull/2748) | vergil-tooling | #2739 | `archive` label, new/legacy recognizers, self-healing `ensure_adhoc_archive`, `normalize` sweep + subcommand, `stray_dotgithub_issue` fix, tests |
| deployment (no PR) | org-wide | #2740 | Provisioned `archive` into `.github` + 7 managed repos; migrated **18** legacy archives; `Outcome: SUCCESS` |
| [#2752](https://github.com/vergil-project/vergil-tooling/pull/2752) | vergil-tooling | #2737 | Standards docs updated to the archive form + `normalize` (docs-review bookend) |
| this PR | .github | #283 | Retrospective (terminal bookend) |

- **Repos touched (code/docs):** vergil-tooling (code + docs), vergil-project/.github (spec/plan/retrospective docs). Deployment additionally provisioned the `archive` label into all 7 managed repos and edited 18 issues in `.github`.
- **Counts:** 5 tasks; 3 code/docs PRs merged + this retrospective PR; 1 deployment (operational, no PR).
- **Releases cut:** none — the change landed on `develop`; the one-time migration ran directly from the merged dev-tree code. `normalize` reaches host installs on the next routine release.
- **Span:** opened 2026-08-11 14:36Z; all work merged/closed the same day (a single-session epic).

## §1 How the plan evolved

The plan was executed essentially verbatim — its eight tasks mapped 1:1 to the
commits, in order. There was **no structural deviation**: the design (repoint the
archive recognizer to the new form, add a legacy recognizer used only by the
self-heal and sweep, parameterize the finder's label set, self-heal on create,
add the idempotent sweep, fix the one audit) shipped as written.

The only delta was a **validation-reconciliation pass** the plan's illustrative
code did not anticipate: the snippets were written against a looser checker than
this repo enforces. Landing them green required (a) moving the annotation-only
`Sequence` import under `TYPE_CHECKING` (ruff `TC003`, since the module uses
`from __future__ import annotations`), (b) replacing a `tuple(home.split(...))`
with a concrete `(owner, repo)` unpack because `ty` does not honour the mypy
`# type: ignore[assignment]` the plan carried, (c) `ruff format`, and (d) three
extra tests to reach 100 % branch coverage (the `apply=True` path, the
new-form-archive rollup guard, and the defensive non-list/non-dict rows in
`plan_normalize_adhoc`). Small delta, contained to one commit — the planning
discipline held.

## §2 Lessons learned

- **Self-healing creation beats sequencing a backfill.** Making
  `ensure_adhoc_archive` resolve *new-form → heal legacy in place → create*
  removed the migration-order race entirely: the drain can never mint a duplicate
  new-form bucket next to a surviving legacy one, so the bulk sweep became a
  convenience and straggler-net rather than a correctness prerequisite. This is
  the pattern to reach for on any rename-with-backfill migration.
- **Preserve an existing invariant to shrink blast radius.** Keeping the `ad-hoc`
  label on archives meant `roadmap.py::_is_perpetual` already filtered them, so
  the roadmap needed *zero* code change. Choosing the migration shape around an
  existing filter is cheaper than adding a new exclusion.
- **A plan's inline code will meet the consuming repo's stricter gates.** Budget
  a reconciliation pass: annotation-only imports need `TYPE_CHECKING` under
  `from __future__ import annotations`; `ty` ignores mypy-style `# type: ignore`
  codes; and 100 % branch coverage demands tests for the defensive branches that
  illustrative snippets gloss over.

## §3 Compromises & tradeoffs

- **Duplicate archives were converted, not deduped.** `.github` still holds
  several same-repo/same-quarter buckets (e.g. multiple
  `Archive (ad hoc): .github — 2026-Q3`) — pre-existing artifacts of the #2698
  list-consistency race. The design deliberately *tolerates* duplicates
  (oldest-reused), so this migration faithfully renamed them; collapsing them is
  separate, deferred work (see §4).
- **Label sync used surgical single-label provisioning.** Rather than a full
  `vrg-ensure-label --sync` (which also deletes deprecated labels across every
  repo), the deployment provisioned just the `archive` label per repo — a
  conservative choice to keep the org-wide blast radius to exactly the new label.
- **Docs scope held to the genuinely-stale doc.** The cross-repo survey found
  only the canonical `standards/github-issues.md` made stale archive-form claims;
  optional org-`docs` niceties were left for later (§4) rather than widening the
  bookend.

## §4 New problems & opportunities

- **Duplicate-archive dedup.** The rename surfaced (but did not create) the
  standing duplicate buckets in `.github`. A follow-up could collapse
  same-repo/same-quarter archives into one. *Logged, not yet acted on.*
- **Optional docs completeness.** The org `docs` repo's `tools-and-skills.md`
  lists only `vrg-adhoc-epic ensure`, not `archive`/`normalize`. Not stale, just
  incomplete. *Logged, optional.*

## §5 What's next

No follow-on brainstorm is warranted — the archive reclassification is complete
and the model is stable. The only forward pointer is the optional, low-priority
dedup item in §4, which the human can promote to a task if and when it matters.

## Appendix A — Operational notes (deployment #2740)

The one-time migration ran from the merged `develop` code via the dev-tree
(`uv run`), so **no release was required** — the task was a backfill against
GitHub, not a tool install.

1. **Label sync** — provisioned `archive` (colour `6a737d`) into
   `vergil-project/.github` and all 7 managed repos (vergil-tooling,
   vergil-containers, .github, vergil-claude-plugin, vergil-vm, vergil-actions,
   docs) via single-label mode; confirmed present in `.github` before applying.
2. **Dry-run** — `vrg-adhoc-epic normalize --all-in vergil-project` reported 18
   legacy archives, all homed in `.github`; no live epic in the set.
3. **Apply** — `--apply` retitled + relabelled all 18 (`epic` → `archive`,
   `ad-hoc` kept).
4. **Spot-check** — `label:epic state:open` in `.github` showed 0 stamped
   buckets; `label:archive` showed the 18; the 6 live `Epic (ad hoc): <repo>`
   epics were unchanged; a second dry-run reported 0 (idempotent).
