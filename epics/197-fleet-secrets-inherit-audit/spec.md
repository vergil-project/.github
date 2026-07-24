# Fleet-wide `secrets: inherit` audit

- **Epic:** vergil-project/.github#197
- **Status:** Design approved 2026-07-24 (brainstormed; fast-tracked — simple,
  well-understood from epic #189). Full epic convention.
- **Repo:** vergil-project/vergil-tooling + all fleet orgs; epic homed in `.github`.
- **Promoted from:** triage idea vergil-project/.github#195.

## 1. Summary

`secrets: inherit` hands a called reusable workflow **every** repository secret
instead of only what it needs — a least-privilege violation, now flagged by
semgrep's `secrets-inherit` rule. Epic #189 fixed **vergil-project**. This epic
**audits the entire fleet — every repo in all six orgs — for `secrets: inherit`**
and drives each hit to an explicit least-privilege map (or removal).

## 2. Shape — cross-org coordination epic

The epic/task model **forbids linking tasks across orgs**, but a fleet audit is
inherently cross-org. So this epic (homed in `vergil-project/.github`) owns the
**audit**, not the other-org fixes:

- **Discovery** — one fleet sweep → a classified catalog of every `secrets:
  inherit` hit.
- **vergil-project residual** — impl tasks under this epic for anything #189 missed.
- **Per-org handoff** — for each *other* org, a triage/seed-epic filed in **that
  org's `.github`** (referenced here, never epic-linked) with that org's catalog
  subset. Each org runs its own remediation.

**Closes** when discovery is complete AND every affected org has a tracked
remediation initiative. It does not (cannot) own the other-org PRs — that is the
placement/linkage law (convention #40; the cross-org boundary that produced the
earlier illegal cross-repo close).

## 3. Discovery

**Blocked-by vergil-project/vergil-tooling#2505** (`vrg-gh search code`). Then
`vrg-gh search code 'secrets: inherit' --owner <org>` across all six orgs
(`diogenes-project`, `logical-minds-foundry`, `mq-rest-admin-project`,
`vergil-project`, `vergils-nemesis`, `wphillipmoore`) — indexed search reaches
every repo, including non-cloned. Cross-checked against the local fleet checkouts
under `~/dev/projects/<org>/<repo>` (no redundant clones). Coverage is logged (no
silent caps).

## 4. Classification per hit (reuse #189)

- uses our `cd-release.yml`: python/go → **remove** (OIDC/none); rust →
  `CARGO_REGISTRY_TOKEN`; ruby → `RUBYGEMS_API_KEY`; java → `CENTRAL_USERNAME`,
  `CENTRAL_TOKEN`, `GPG_PRIVATE_KEY`, `GPG_PASSPHRASE`.
- uses a **different/custom** publish workflow (likely in other orgs, e.g.
  mq-rest-admin's rust/java libs): inspect the `secrets.*` it reads → **explicit
  map**.
- non-publishing / `GITHUB_TOKEN`-only → **remove**.

## 5. Scope

All `secrets: inherit` in **any** workflow (not just `cd.yml`), across all six
orgs. **Only** `secrets: inherit` — `mutable-action-tag` and other semgrep
findings are out of scope (backlog #194 / exempted).

## 6. Bookends & prevention

- **Documentation** (#198): this spec + plan.
- **Documentation review** (#2519, closing gate) — **light**: the explicit-secrets
  model docs already shipped in #189 (#2507 / vergil-actions#787); confirm the
  catalog is published and every affected org has a handoff.
- **No follow-on-brainstorm bookend** — reintroduction is already prevented by the
  semgrep `secrets-inherit` rule (deliberately *not* exempted), which fails CI
  fleet-wide on any new `secrets: inherit`.
- **No cold-rebuild validation** — config audit, not infra.

## 7. Out of scope

- Other semgrep findings (`mutable-action-tag` → #194; the ruleset-determinism
  thread).
- Non-`secrets:-inherit` secret usage.
