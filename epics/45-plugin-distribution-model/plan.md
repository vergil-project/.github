# Plugin distribution model — implementation plan

Epic: vergil-project/.github#45 · Spec: `./spec.md`

Tasks are ordered. Task 1 is a **gate** — do not roll out settings changes until
it passes.

## Target settings shape

**Inside vergil-project** (dev channel) — every vergil-project repo's
`.claude/settings.json`:

```jsonc
"extraKnownMarketplaces": {
  "vergil-dev": {
    "source": { "source": "github", "repo": "vergil-project/vergil-claude-plugin", "ref": "develop" }
  }
},
"enabledPlugins": { "vergil@vergil-dev": true }
```

**Outside vergil-project** (released channel) — other orgs' repos:

```jsonc
"extraKnownMarketplaces": {
  "vergil": {
    "source": { "source": "github", "repo": "vergil-project/vergil-claude-plugin", "ref": "main" }
  }
},
"enabledPlugins": { "vergil@vergil": true }
```

The old single `vergil-marketplace` registration is removed.

## Task 1 (GATE): Validate the two-marketplace mechanic — closes #540

Confirm on the machine, before touching any repo en masse:

- Two registrations of the **same repo** under two local names (`vergil` @ `main`,
  `vergil-dev` @ `develop`) resolve to **two separate clones**, each on its own
  ref — i.e. the local marketplace name is free-form and does not have to match
  `marketplace.json`'s `name`.
- Each clone **re-fetches when its branch advances** (`/plugin marketplace
  update` then a fresh session): merge to `develop` → `vergil-dev` picks it up;
  cut a release to `main` → `vergil` picks it up.

If both hold → the recommended mechanism works; proceed. If either fails →
**stop and revisit** a `vrg`-side refresh tool (spec §5); do not roll out.

## Task 2: vergil-claude-plugin → dev channel

`.claude/settings.json`: replace the `vergil-marketplace` registration with the
`vergil-dev` block above; set `enabledPlugins` to `vergil@vergil-dev`.

## Task 3: Each vergil-project member repo → dev channel

For `vergil-tooling`, `vergil-vm`, `vergil-containers`, `vergil-actions`,
`.github`: same `.claude/settings.json` change as Task 2. One PR per repo (each a
task under this epic).

## Task 4: External-org consumption → released channel

Document the released-channel block (`vergil` @ `main`) and apply it where other
orgs (e.g. `logical-minds-foundry`) consume the plugin. Where those repos are
out of reach from here, deliver as documentation + the exact snippet.

## Task 5: Document the paradigm + SemVer deviation

- `vergil-claude-plugin/CLAUDE.md`: a "Plugin distribution" section — two
  channels, branch-tracked, **no SemVer**, with the §2 rationale in brief and a
  link to this spec.
- Any canonical distribution/refresh docs (e.g. the README "Update" section):
  update to the two-channel model; stop instructing tag-pinning.

## Task 6: Review & update site documentation

Review `vergil-claude-plugin`'s published site docs for the new model; update
anything describing the old marketplace/versioning approach. (Docs are broadly
stale; scope this task to the distribution-model change, flag the rest.)

## Definition of done

Every vergil-project repo consumes `vergil-dev@develop`; the released channel
(`vergil@main`) is documented for external orgs; the paradigm + SemVer-deviation
rationale is captured in CLAUDE.md, the README, and this spec; site docs reflect
it; #540 closed on Task 1 passing.
