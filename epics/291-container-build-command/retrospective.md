# Retrospective: `[container].build-command` hook

- **Epic:** vergil-project/.github#291
- **Status:** Retrospective (2026-08-12)
- **Partners:** [`spec.md`](./spec.md) · [`plan.md`](./plan.md) → this document.

## 0. At a glance

We set out to let a repo bake a **non-apt** dependency into its dev/CI container
the way `[container].system-packages` (#272) bakes apt ones — driven by melete's
need for the npm package `@coderline/alphatab`. What shipped is a general
`[container].build-command` hook plus a `build-cache-files` companion key,
threaded through the same single-speller discipline as #272, wired into local
cache builds and a new fail-closed CI step, released, **validated against the real
released tooling**, and documented across three repos. The epic delivered its core
promise — and its live validation caught a genuine resolvability gap that a
unit-test-only path would have shipped, which cost one unplanned fix + release +
CI-parity cycle.

**Work delivered**

| PR | Repo | Merged | What it did |
|---|---|---|---|
| [#294](https://github.com/vergil-project/.github/pull/294) | .github | 08-11 | Spec + plan (documentation bookend) |
| [#2770](https://github.com/vergil-project/vergil-tooling/pull/2770) | vergil-tooling | 08-11 | Config keys: `build-command` + `build-cache-files` |
| [#2772](https://github.com/vergil-project/vergil-tooling/pull/2772) | vergil-tooling | 08-12 | Bake the command into the cached image + fold cache-key inputs |
| [#2773](https://github.com/vergil-project/vergil-tooling/pull/2773) | vergil-tooling | 08-12 | `vrg-container-build-command` speller (single source for CI) |
| [#2784](https://github.com/vergil-project/vergil-tooling/pull/2784) | vergil-tooling | 08-12 | **Follow-on fix:** expose `NODE_PATH` so baked npm libs resolve |
| [#2789](https://github.com/vergil-project/vergil-tooling/pull/2789) | vergil-tooling | 08-12 | Site-docs sweep (trust model, CLI, CI, TS xref) |
| [#820](https://github.com/vergil-project/vergil-actions/pull/820) | vergil-actions | 08-12 | New `shared/setup/build-command` CI step (fail-closed, no retry) |
| [#825](https://github.com/vergil-project/vergil-actions/pull/825) | vergil-actions | 08-12 | **Follow-on:** CI `NODE_PATH` parity |
| [#830](https://github.com/vergil-project/vergil-actions/pull/830) | vergil-actions | 08-12 | Docs parity (index / ci-test / configuration) |
| [#301](https://github.com/vergil-project/.github/pull/301) | .github | 08-12 | **Erratum:** correct the spec/plan "default resolution path" claim |

**Operational tasks (no PR):** #2765 *deployment* (the human-gated release gate →
vergil-tooling v2.1.189); #2766 *validation* (cold rebuild — **FAILURE on first
run, SUCCESS on re-run** after #2784).

- **Repos touched:** 3 (vergil-tooling, vergil-actions, .github). `docs` was swept
  and deliberately left unchanged.
- **Children:** 13 (12 closed + this retrospective). **PRs merged:** 10. **Operational
  tasks:** 2.
- **Releases cut during the epic:** vergil-tooling **v2.1.189** (base feature) and
  **v2.1.191** (NODE_PATH fix); vergil-actions **v2.1.24** and **v2.1.25** (CI step
  + parity).
- **Span:** opened 2026-08-11 → all work merged 2026-08-12 (~2 days).
- **Cross-org:** one adoption heads-up comment on `mnemosys-project/melete#85` (no
  linked task — cross-org linking is disallowed).

## 1. How the plan evolved

The plan named six tasks (config → bake → speller → release → CI step →
validation). Execution followed that spine exactly for the first five — the
`Blocked-by` chain drove them cleanly, including the **modeled release gate**
(#2765), which the alignment pass had added on purpose so "merged vs deployed"
was an explicit node rather than a human silently filling it.

The delta is entirely downstream of **one event: the validation failed the first
time.** The plan's Task 6 asserted `require.resolve("@coderline/alphatab")` would
resolve from a global `npm install -g`. It did not — Node's default resolution
path does not include the npm global dir. That single finding spawned four
issues that were **not** in the original plan:

- **#2781** (vergil-tooling) — inject `NODE_PATH` on the run path so a baked npm
  *library* resolves; then a second release (v2.1.191) and a re-run of #2766.
- **#824** (vergil-actions) — the same `NODE_PATH` in CI, because CI runs in the
  base image, not the cached one; without it, local and CI would diverge — the
  exact drift the epic's single-speller discipline exists to kill.
- **#829** (vergil-actions) and **#300** (.github) — spawned by the
  documentation-review sweep, which (as designed) fanned out per-repo rather than
  forcing one cross-repo docs PR.

So the plan-vs-actual delta is **+4 issues, +2 releases, one validation re-run** —
all traceable to a single technical discovery, not to planning drift. The first
five tasks landing exactly as planned is the signal the front-loaded spec+plan
discipline worked; the validation delta is the signal the epic taught us
something real about Node resolution that no one knew when the spec was written.

## 2. Lessons learned

- **The live validation earned its place.** Unit tests proved the config, the
  bake, and the speller; every one was green. They could not have caught that a
  global install is not `require`-resolvable — that needed a real cold rebuild +
  a real `require.resolve` on a clean tree. Infra/mechanism epics should keep
  paying for an operational validation against the *released* artifact, not a
  dev-tree stand-in.
- **The bind-mount is the whole design constraint.** `/workspace` is bind-mounted
  at both build and run time, so anything written there at build is masked at
  runtime. This is why "just `npm ci` in the vendored dir" cannot work and why the
  contract is *install outside the workspace*. Any future "bake something into the
  image" feature must reason about the mount first.
- **Runtime mechanism beats build-time cleverness when the runtime differs.** The
  NODE_PATH fix wanted to be a baked image `ENV`, but `nerdctl commit --change`
  rejects `ENV` (only CMD/ENTRYPOINT) — and nerdctl is the primary Mac/Lima path.
  A run-path `-e NODE_PATH` works identically on docker and nerdctl. Verify the
  actual runtime supports a mechanism before committing to it.
- **Modeling the release as a task paid off immediately.** The deployment gate
  (#2765) meant the CI step (#819) was never runnable before the tool was
  installable — and when the *fix* also needed a release, the pattern was already
  there to repeat. Determinism in the `Blocked-by` graph is what will eventually
  let an epic run end-to-end unattended.

## 3. Compromises & tradeoffs

- **ESM is not solved — knowingly.** `NODE_PATH` is honored by CommonJS `require`
  only; ESM `import` ignores it. The fix makes `require("@coderline/alphatab")`
  resolve, which is what the acceptance tested, but a consumer using
  `import … from` will still fail to resolve. We shipped the require path and
  documented the ESM limitation rather than solving it, because the driver's
  resolution style is the driver's (cross-org) call and a general ESM answer (a
  resolvable install location, a package `imports`/`exports` map, or a symlink) is
  a larger design question. **Debt, explicitly logged** (§4).
- **`build-cache-files` is a manual footgun.** A repo must remember to list its
  lockfile or a dependency bump silently reuses the stale image. We chose the
  explicit companion key over auto-detection (which can't statically know a shell
  command's inputs) and documented the sharp edge — correctness over convenience,
  but the burden is on the author.
- **Spec/plan corrected in place, not rewritten.** The "default resolution path"
  claim was factually wrong, but the Model B *design* was right — so #300 added
  bracketed errata rather than rewriting history. The design record stays honest
  about what it got wrong.

## 4. New problems & opportunities

- **ESM resolution for baked npm libraries** — the open technical question above.
  Flagged on `mnemosys-project/melete#85` for the consumer; **not yet a
  vergil-tooling task** — file one if a consumer genuinely needs `import`.
- **Three org-wide hygiene problems surfaced *during* this epic's execution** and
  were captured as evidence rather than dropped:
  - `.github#235` (dev-container image / disk accumulation) — a live incident
    (VM hit 100%, 24G reclaimed) added as prioritization evidence.
  - `.github#196` (fleet `.gitignore` management) — an un-ignored mkdocs
    `docs/site/site/` build artifact stranded a merged worktree; added as a
    concrete instance, incl. the "un-ignored output blocks finalize cleanup"
    consequence.
  - `.github#295` (worktree "ghosts") — `vrg-worktree-status` is single-repo and
    blind to non-canonical (`release-*`) worktrees; filed fresh from a stale
    release worktree found in vergil-actions.
  None are this epic's to fix; all are now tracked with real evidence.

## 5. What's next

- **melete adoption** (`mnemosys-project/melete#85`) — the driver consumes the
  feature; the heads-up comment there lists the traps (out-of-workspace install,
  the ESM caveat, the version/CI-parity gates). Cross-org, so it proceeds in that
  org.
- **No follow-on epic is chained** — this was a self-contained capability, not the
  first slice of a known enabling chain, so no forward brainstorm was seeded. The
  three surfaced hygiene issues (§4) are independent and already tracked.

## Appendix A — Operational notes

Release/deploy order that made the feature usable end-to-end:

1. vergil-tooling config + bake + speller merged → **v2.1.189** (deployment #2765)
   — makes `vrg-container-build-command` and the bake installable in CI.
2. vergil-actions CI step (#820) merged → **v2.1.24** — CI runs the build-command.
3. Validation #2766 run against v2.1.189 → **FAILURE** (require-resolve gap).
4. NODE_PATH fix (#2784) merged → **v2.1.191**; CI parity (#825) merged →
   vergil-actions **v2.1.25**.
5. Validation #2766 re-run against v2.1.191 → **SUCCESS**: on a clean tree,
   `vrg-container-run -- node -e 'require.resolve("@coderline/alphatab")'` →
   `/usr/lib/node_modules/@coderline/alphatab/dist/alphaTab.js`; fail-closed and
   rebuild-on-change confirmed.
6. Docs sweep (#2789, #830, #301) across the three repos.

Gotcha to remember: a bare `npm install -g <lib>` is baked and on `PATH` for
executables but **not** `require`-resolvable without `NODE_PATH`; the tooling now
sets it when a `build-command` is declared (require only — not ESM).
