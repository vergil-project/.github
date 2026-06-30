# Plugin distribution model — released vs development marketplaces

- **Epic:** vergil-project/.github#45
- **Originating bug:** vergil-project/vergil-claude-plugin#540
- **Status:** Approved design (2026-06-30)

## 1. Problem

The `vergil` Claude Code plugin is delivered through a **marketplace** — which
is, underneath the CLI abstraction, a **git clone** kept once per marketplace
*name* in the user's home directory (`~/.claude/plugins/marketplaces/<name>/`),
machine-wide and shallow (single ref).

We attempted to consume it the way we consume everything else — pinned by a
moving `vX.Y` tag, per repo. That does not work here, for two compounding
reasons:

1. **One clone per name, machine-wide.** Because all repos registered the *same*
   marketplace name (`vergil-marketplace`) with *different* refs (`develop` for
   the plugin repo, `v2.1` for consumers), the single shallow clone could only
   be on one ref. It stuck on `develop`, and the `v2.1`-pinned consumers
   silently ran `develop` (#540).
2. **No per-repo version pinning.** Even with the collision resolved, a shallow
   clone tracks exactly one ref head. Unlike `pip install <pkg>==2.1` or a
   `vX.Y` image tag in a URL — where each consumer resolves its own version —
   the marketplace cannot give repo A plugin 2.1 while repo B is on 2.2 at the
   same time. The mechanism assumes a single current version.

## 2. Why SemVer does not apply to the plugin

SemVer manages a tight, **testable** contract: a consumer pins a version because
a breaking change to that contract would break *their* code. The plugin has no
such contract. It is **prose guidance for an AI** — skills, hooks, agent
instructions — full of judgment calls, special cases, and non-determinism **by
design**, because the agent makes human-like decisions. There is nothing to pin
against and nothing that mechanically "breaks" on update.

Versioning the plugin imposed a model built for testable APIs onto something
fundamentally interpretive — a **category error**. vergil-tooling (a Python
package with real APIs) and the container images (runtime environments) *must*
be SemVer-pinned because code links against them and a breaking change breaks
builds. The plugin has none of those properties.

Dropping SemVer for the plugin is therefore not a compromise forced by tooling
limits — it is the **correct** model for what the plugin *is*. That it also
happens to be the only model the Claude marketplace mechanism supports makes it
both realistic and permanent (until Claude changes the mechanism).

## 3. The model

The plugin has exactly **two states**, each served by its own marketplace —
**distinct local names → distinct clones → no collision**:

| Marketplace | Tracks | Consumed by |
| --- | --- | --- |
| `vergil` (released) | `main` | Everything outside `vergil-project` (other orgs) |
| `vergil-dev` (development) | `develop` | Everything **inside** `vergil-project` |

- **`main` is the released line.** Release PRs already merge there; it is "latest
  released," always. Consumers get a release on their next refresh — no staged
  rollout.
- **`develop` is the single staging area.** To preview unreleased behavior,
  point at the `vergil-dev` marketplace. There is no other pre-release channel.
- **Tags `vX.Y` survive only as human release markers** — changelog and
  **rollback targets**. Rollback = revert `main`; everyone refreshes. Tags are
  never a per-repo dependency pin.

### Org boundary as a validation domain

Scoping the dev channel to the whole `vergil-project` org (not just the plugin
repo) is deliberate: every vergil-project session dogfoods unreleased behavior,
making the org our **integration/validation domain**. A bad `develop` change
surfaces across our own work first, before any external consumer sees it.
Acceptable because the payload is behavioral guidance — the blast radius is
"agents behave differently," not broken builds.

## 4. Tracking a branch, not a tag

Both channels track a **branch** (`main` / `develop`), not a moving tag. This is
what makes the recommended mechanism viable: a shallow clone re-fetches a branch
tip far more reliably than it re-resolves a moved tag. It also sidesteps the
shallow-clone-has-no-tags failure that produced #540.

## 5. Validate before rollout

Before changing every repo, confirm on the machine that a clean, collision-free
two-marketplace setup actually works: two registrations of the same repo under
two local names at two refs, each its own clone, each re-fetching when its
branch advances. **Only if that provably fails** do we revisit a `vrg` refresh
tool that operates on the clone directly — explicitly deferred, not chosen.

## 6. Non-goals

- **No SemVer-managed, per-repo plugin version pinning.** Out of scope by design.
- **No staged multi-version migration** as a standing capability. If ever truly
  needed, the escape hatch is a *temporary* extra named marketplace
  (`vergil-next`) for the upgrading cohort, retired afterward — an exception,
  not the model.
- **No clone-manipulation tooling** unless §5 validation fails.
- vergil-tooling and container-image versioning are unaffected; they keep SemVer.
