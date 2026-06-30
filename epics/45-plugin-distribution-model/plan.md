# Plugin distribution model — implementation plan

Epic: vergil-project/.github#45 · Spec: `./spec.md`

The design collapsed to a single released channel (`main`), so implementation is
small: point every consumer at `main`, document the model, and confirm it works.

## Target settings shape (every repo)

`.claude/settings.json` — keep the existing `vergil-marketplace` registration,
change only the **ref** to `main`:

```jsonc
"extraKnownMarketplaces": {
  "vergil-marketplace": {
    "source": { "source": "github", "repo": "vergil-project/vergil-claude-plugin", "ref": "main" }
  }
},
"enabledPlugins": { "vergil@vergil-marketplace": true }
```

No new marketplace, no name change — only `ref: develop|v2.1 → main`.

## Task 1: Point every repo at `main`

`.claude/settings.json` ref → `main` in each repo (one PR per repo, each a task
under this epic):

- `vergil-claude-plugin` (currently `develop`)
- `vergil-tooling`, `vergil-vm`, `vergil-containers` (currently `v2.1`)
- `vergil-actions` (currently unset → defaults to develop)
- `.github` (align)

## Task 2: Document the model

- `vergil-claude-plugin/CLAUDE.md`: a "Plugin distribution" section — single
  released channel on `main`; **no SemVer/channels**; to run unreleased
  behavior, tell the agent to read the `develop` file and follow it; releasing
  to `main` is what makes a skill a real slash command. Link this spec.
- `README.md` "Update" section: update to the single-channel model; drop any
  tag-pinning / two-channel language.

## Task 3: Review & update site documentation

Review `vergil-claude-plugin`'s published site docs; update anything describing
the old marketplace/versioning/refresh approach to the single-channel model.
Scope to this change; flag unrelated staleness.

## Task 4: Validate + close #540

Confirm a clean `vergil-marketplace`@`main` setup re-fetches when a release
advances `main` (`/plugin marketplace update` + fresh session). With the
collision gone this is the simple documented case. If it ever fails, the
"agent reads the file" path (spec §2) is the fallback — do **not** build a
refresh tool now. Closes #540.

## Definition of done

Every repo's `settings.json` points the plugin at `main`; the single-channel
model + the SemVer/channel-rejection rationale is captured in CLAUDE.md, the
README, this spec; site docs reflect it; #540 closed on Task 4.
