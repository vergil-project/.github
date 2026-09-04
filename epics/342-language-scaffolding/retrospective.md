# Retrospective — Born-green language-skeleton scaffolding (cpp first) + workaround paydown

> Epic: [`vergil-project/.github#342`](https://github.com/vergil-project/.github/issues/342) ·
> Partners [`spec.md`](./spec.md) and [`plan.md`](./plan.md) at the documentation tier.
> Read the three in order: **spec → plan → retrospective**.

## §0 At a glance

We set out to give `vrg-github-repo-init` a **language-skeleton phase** so a new
repo of a supported language is *born green* — the tools run and pass immediately,
with no hand-assembled skeleton — starting with **cpp**, and to pay down the
defensive workarounds that a half-bootstrapped cpp tree had forced. What shipped
is exactly that: a repo-init scaffolding phase (render templates → containerized
`conan lock create` → full `vrg-validate`), the Conan-output relocation and
gitignore/warmup paydown that let the skeleton ship clean, the retrofit of the one
existing cpp repo onto the clean path, and — proven by a **live** end-to-end run —
a cpp repo that is green from birth. The born-green claim was validated for real,
and getting there surfaced (and fixed) three latent defects that only live cpp
execution could expose.

### Work delivered

| PR | Repo | What it did |
|---|---|---|
| [.github#345](https://github.com/vergil-project/.github/pull/345) | .github | Spec + plan (epic opening bookend) |
| [tooling#3037](https://github.com/vergil-project/vergil-tooling/pull/3037) | vergil-tooling | **T1** — Conan output → `build/`; cmake via `-DCMAKE_TOOLCHAIN_FILE` (retires the source-root prefix-path hack) |
| [tooling#3038](https://github.com/vergil-project/vergil-tooling/pull/3038) | vergil-tooling | **T2** — born-green scaffold: repo-init language phase + cpp skeleton templates + registry lock command |
| [tooling#3039](https://github.com/vergil-project/vergil-tooling/pull/3039) | vergil-tooling | **T4** — drop dead Conan generator globs from the cpp gitignore fragment |
| [tooling#3040](https://github.com/vergil-project/vergil-tooling/pull/3040) | vergil-tooling | **T5** — drop the dead cpp `_WARMUP_REQUIRES` entry |
| [mq-gateway#22](https://github.com/logical-minds-foundry/mq-protocol-gateway/pull/22) | mq-protocol-gateway | **Prereq** — commit `conan.lock` (mandatory for cpp validation since tooling#3021) |
| [mq-gateway#23](https://github.com/logical-minds-foundry/mq-protocol-gateway/pull/23) | mq-protocol-gateway | **T3** — retrofit onto the clean path (drop prefix-path hack + Conan gitignore blocks) |
| [tooling#3045](https://github.com/vergil-project/vergil-tooling/pull/3045) | vergil-tooling | **Fix** — re-ignore `CMakeUserPresets.json` (Task-4 over-prune, surfaced live) |
| [tooling#3050](https://github.com/vergil-project/vergil-tooling/pull/3050) | vergil-tooling | **Fix** — restore the `conan.lock` warmup prerequisite (born-green ordering, surfaced live) |
| [.github#350](https://github.com/vergil-project/.github/pull/350) | .github | Reconcile plan to the post-#325 fragment world + record the cross-repo release gate |
| [tooling#3054](https://github.com/vergil-project/vergil-tooling/pull/3054) | vergil-tooling | **Docs** — reflect born-green scaffolding + the clean Conan-output path in `docs/site` |
| _(this PR)_ | .github | Retrospective (closing bookend — closes the epic) |

- **Repos touched:** 3 — `vergil-tooling` (mechanism + paydown + docs), `.github` (spec/plan/reconcile/retrospective), `logical-minds-foundry/mq-protocol-gateway` (lock prereq + retrofit).
- **Tasks:** 11 epic children (10 closed + this retrospective); 5 planned code tasks (T1–T5), 2 unplanned live-surfaced fixes, 1 unplanned plan-reconcile, 1 cross-repo prereq (#20), 2 bookends (docs review, retrospective).
- **PRs merged:** ~11 substantive (excluding the 3 release-chore pairs).
- **Releases cut:** 3 — v2.1.220 (T1–T5), v2.1.221 (CMakeUserPresets fix), v2.1.222 (warmup fix).
- **Span:** opened 2026-09-02, closed 2026-09-03 (~1 day).
- **Live acceptance:** [#3036](https://github.com/vergil-project/vergil-tooling/issues/3036) — a fresh `vrg-github-repo-init --language cpp` reached a green repo end-to-end on v2.1.222.

## §1 How the plan evolved

The plan's five-task spine (T1 Conan→`build/` → T2 scaffold → T3 retrofit →
T4 gitignore paydown → T5 warmup paydown) held as the *logical* structure, but
execution corrected three of its assumptions:

- **Task 4 was written against a codebase that no longer existed.** The plan
  targeted a monolithic `gitignore.baseline` and a flagship-subset drift guard;
  epic #325 (composable-gitignore) had since replaced the monolith with
  per-language fragments (`data/gitignore/<lang>`) composed by `lib/gitignore.py`,
  guarded by a frozen lossless-split invariant. T4 was adapted to the fragment
  world (prune the cpp fragment; update the frozen `_LEGACY_GITIGNORE_PATTERNS` as
  an intentional, reviewed diff), and the durable plan was reconciled in a
  dedicated follow-up (#347 → .github#350).

- **"Landed" meant "released," not "merged."** The plan's ordering guard
  ("Tasks 4–5 after 1–3") was untenable once the cross-repo reality was faced:
  `mq-protocol-gateway` pins `vergil = "v2.1"` (a moving *release* tag), so the
  T3 retrofit and the live validation needed a vergil-tooling **release** carrying
  T1+, not merely a develop merge. The real spine became: land all same-repo tasks
  (1, 2, 4, 5) → **cut a release** → T3 retrofit + live validation → docs → this
  retrospective. Three releases (v2.1.220–222) punctuated the run for exactly this
  reason.

- **The born-green mechanism shipped two latent defects that only a live cpp run
  could catch** (see §2). Each spun off an unplanned fix task (#3044, #3049) and
  its own release before the epic could close.

A cross-repo prerequisite also appeared: `mq-protocol-gateway#20` (commit
`conan.lock`, mandatory since #3021) had to merge before T3 could validate green.

## §2 Lessons learned

1. **A mechanism that only runs in one environment must be *tested* in that
   environment.** The born-green cpp scaffold was unit-tested inside
   vergil-tooling's Python dev container, which has no `conan` — so its
   integration test is gated-skip, and two real defects (#3044, #3049) were
   structurally invisible until a live cpp run. Gated-skip coverage reads as
   "covered" while proving nothing. **The durable guard is a real cpp-container
   integration test in CI**, not a skipped unit test plus a manual smoke run.

2. **"Landed" is ambiguous across a repo boundary.** For a consumer pinned to a
   release tag, a dependency change isn't usable until it's *released*. Plans that
   say "with Task X landed" must state merge-vs-release explicitly whenever a
   downstream repo is involved.

3. **A plan authored against a moving codebase needs a reality-check at execution
   time.** Task 4's staleness (pre-#325) was caught only because the implementing
   agent verified the plan against the live tree before editing. Cross-epic
   concurrency silently invalidates plan detail.

4. **A bootstrap mechanism must tolerate its own transient half-built state.**
   Removing the cpp warmup-skip (T5) assumed "born-green ⇒ never half-bootstrapped."
   The opposite is true: born-green scaffolding *creates* a no-lock window (skeleton
   rendered, `conan.lock` not yet resolved) — precisely when the warmup must skip.
   The fix mirrored the existing python `uv.lock` pattern, which had the shape right
   all along.

## §3 Compromises & tradeoffs

- **The born-green integration test stays gated-skip in CI.** This epic fixed the
  defects but did *not* add the cpp-container CI test that would catch the next
  one pre-merge; the live #3036 validation remains a manual gate. Debt knowingly
  carried — logged as the top item in §4.

- **Three releases for one epic.** Each cross-repo/live-visible fix needed its own
  release so the consumer and the live validation could pick it up. Correct, but it
  made the tail of the epic release-bound rather than merge-bound.

- **Two throwaway smoke repos** (`born-green-smoke`, `born-green-smoke-2`) remain on
  GitHub, to be deleted via the web UI (the `gh` token lacks `delete_repo` scope —
  not worth widening for a one-off).

## §4 New problems & opportunities

- **cpp-container integration test in CI** *(not yet filed — should be)*. The
  single highest-value follow-up: a real `vrg-github-repo-init --language cpp`
  (or `scaffold_language`) run in a conan-capable container, gating merges. Would
  have caught both #3044 and #3049 before release.
- **`vrg-finalize-pr` robustness — two seams observed** *(logged, not yet acted on)*:
  (a) a false-red on a stale `.pyc` (`marshal data too short`) during post-merge
  validation; (b) a cleanup failure ("develop working tree is not clean" /
  "Cache build failed") from leftover pre-T1 Conan cruft plus a missing develop-keyed
  cpp cache image. Both are recoverable by hand but read as scary failures.
- **Born-green for the other languages.** The mechanism is per-language by design;
  python/typescript/… can follow the same shape (spec: "cpp first"). Future epic
  material.

## §5 What's next

- File the **cpp-container CI integration test** issue (§4, item 1) — the
  regression guard this epic proved is needed.
- Optionally capture the two **`vrg-finalize-pr` seams** as tooling issues.
- A follow-on **born-green-for-other-languages** epic when a second language needs
  the treatment.

## Appendix A — Operational notes

Release-bound tail (the mechanical sequence that mattered):

1. Land T1, T2, T4, T5 in `vergil-tooling` (batched per-repo) → **release v2.1.220**.
2. Merge `mq-protocol-gateway#20` (commit `conan.lock`) — prerequisite for T3.
3. T3 retrofit (`mq-protocol-gateway#23`) validates against v2.1.220.
4. Live validation #3036 fails → **CMakeUserPresets over-prune** (#3044) →
   **release v2.1.221**.
5. Re-run surfaces the **warmup/lock ordering** defect (#3049) → **release v2.1.222**.
6. Live #3036 passes on v2.1.222 (born-green proven) → docs review (#3032) →
   this retrospective.

Two finalize seams hit along the way (see §4) were resolved by cleaning stale
untracked Conan output and letting a fresh `vrg-container-run -- vrg-validate`
rebuild the develop-keyed cpp cache image.
