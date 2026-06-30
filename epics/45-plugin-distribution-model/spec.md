# Plugin distribution model — single released channel (`main`)

- **Epic:** vergil-project/.github#45
- **Originating bug:** vergil-project/vergil-claude-plugin#540
- **Status:** Approved design (2026-06-30)

## 1. Decision

**Every consumer of the `vergil` plugin — external orgs, all of `vergil-project`,
and the plugin repo itself — tracks `main`.** One marketplace, one ref, one
name. There is **no development marketplace, no SemVer, no version tags as pins,
and no per-channel mechanism of any kind.**

- `main` is the released line (release PRs already merge there). Releasing to
  `main` is the single act that makes a skill/hook/agent "just work" for
  everyone.
- `develop` remains the staging area — changes land there first — but it is
  **not consumed through a marketplace.**

## 2. How `develop` is consumed: the agent reads the file

To run an unreleased change, you tell the agent: *"read the `develop` version of
this skill/file and do what it says."* The agent reads it and follows it — the
AI-equivalent of running it manually. This is not a workaround grafted on; it is
the realization that **makes the whole channel mechanism unnecessary.**

This was demonstrated live: when the marketplace failed to ship the migrate-repo
skill, no release was needed — the agent read the `develop` SKILL.md and
executed it directly.

## 3. Why we abandon SemVer *and* the channel mechanism

SemVer and version-pinned release channels manage a tight, **testable**
contract: a consumer pins a version because a breaking change to that contract
breaks *their code*. Two assumptions hold there — consumption is deterministic,
and the artifact has a spec that can break.

**Neither holds for this plugin.** It is prose guidance for an AI making
human-like judgment calls — non-deterministic by design, with no testable
contract to break. And its consumer is an agent that can *read and follow a
file directly*, so "use the unreleased version" needs no version-resolution
machinery at all. Imposing channels/SemVer here was a category error: applying
a model built for deterministic, constrained software to something that is
neither.

vergil-tooling (real APIs) and the container images (runtime environments) keep
SemVer — code links against them and breaking changes break builds. The plugin
has none of those properties. The asymmetry is the point.

## 4. Why a single channel is also forced by the mechanism

Even had we wanted two channels, the Claude marketplace makes it costly: the
marketplace **name** comes from `.claude-plugin/marketplace.json`, one clone per
name machine-wide, and "same name replaces." Two channels from one repo would
require the `marketplace.json` `name` to **differ per branch** (`vergil-dev` on
develop, `vergil` on main) and the release tooling to maintain that divergence
across the `main → develop` back-merge. We reject that complexity in favor of §1.

## 5. The trade

We give up *automatic* org-wide `develop` dogfooding — everyone runs released
behavior, and previewing `develop` is on-demand (§2) rather than ambient.
Accepted: the "read the file" move covers the same need without standing
infrastructure, and a single released version is the right model for
behavioral guidance anyway.

## 6. Non-goals

- No development marketplace / second channel.
- No SemVer or `vX.Y` tag pinning of the plugin (tags, if kept, are human
  changelog markers only).
- No `vrg` clone-refresh tool, no per-branch `marketplace.json` `name`, no
  `vrg-release` changes for distribution.
- vergil-tooling and container-image versioning are unaffected.

## 7. Residual reliability note

The one remaining concern is whether `claude plugin marketplace update`
reliably re-fetches `main` when a release advances it (the shallow-clone
staleness behind #540). With the collision gone (one marketplace, one ref),
this is the simple, documented case and is expected to work; if it ever
doesn't, the §2 "read the file" path is the immediate fallback, and a refresh
tool can be reconsidered then — explicitly **not** now.
