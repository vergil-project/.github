# Retrospective — TypeScript language support (epic #284)

> Partners `spec.md` and `plan.md`. Read the three in order: what we set out to
> do → how we planned it → honestly how it went.

## §0 At a glance

We set out to add **TypeScript as a first-class Vergil language** on the existing
containerized, multi-version model — registry entry, config schema, container
images, reusable CI, and standards docs — using the C++ epic (#207) as the
template. That is what shipped: a strict-by-default TypeScript toolchain (Node
runtime, single canonical `tsc`, npm, Vitest + 100% V8 coverage) delivered across
five repos, with `prod-/dev-ts-node:{24,22}` live on GHCR and the reusable CI
gates wired — **executed in roughly one day** (epic opened 2026-08-11 14:47Z,
last task PR merged 2026-08-12 13:49Z).

**Work delivered**

| Repo | PR | What it did |
|---|---|---|
| .github | #289 | Docs: spec + plan (bookend) |
| .github | #297 | Add TypeScript to the profile README supported-languages list |
| docs | #22 | Higher-level TypeScript summary docs |
| vergil-actions | #815 | T7 — reusable CI accepts `language: typescript` (+ CodeQL mapping) |
| vergil-containers | #522 | T1 — common TS image base + primary Node image |
| vergil-containers | #532 | T2 — second Node image (node-22) |
| vergil-containers | #533 | Pin container TypeScript to the stable 5.x line |
| vergil-containers | #538 | Add ts-node (24, 22) to the CD publish matrix |
| vergil-containers | #543 | Docs: add dev-ts-node to the Available Images catalog |
| vergil-tooling | #2749 | T3 — `[typescript]` config schema + language enum |
| vergil-tooling | #2754 | T6 — CodeQL support + primary→CodeQL mapping + repo-init |
| vergil-tooling | #2755 | T4 — TypeScript language registry entry + strict base tsconfig |
| vergil-tooling | #2760 | T5 — container maps + language detection |
| vergil-tooling | #2761 | T8 — TypeScript standards docs |
| vergil-tooling | #2777 | Fix ESLint LINT: resolve packaged flat-config from the consumer repo |
| vergil-tooling | #2783 | Documentation-review sweep |

**Operational tasks (no PR — closed on a recorded `Outcome: SUCCESS`):** T9
(vergil-containers #521, publish images to GHCR) · T10 (vergil-tooling #2747,
cold-rebuild + full-pipeline validation).

**By the numbers:** 5 repos · 18 task children (16 merged PRs + 2 operational) +
this retrospective · ~1-day open→close span. **Releases attributed to the epic:**
vergil-containers **v2.1.24** (published the ts-node images), vergil-tooling
**v2.1.188** (registry entry) and **v2.1.190** (ESLint fix), plus vergil-actions
releases carrying T7. (Several other in-window releases exist; whether each
carried epic-specific vs. concurrent content is not individually verified here.)

**Plus one related-but-not-in-epic fix** born from this work: docs #24 (the docs
repo `ci.yml` gate fix — see §4).

---

## §1 How the plan evolved

`plan.md`'s "Evolution during execution" log was never kept live during the run
(itself a lesson — §2), so this narrative is reconstructed from the execution
record. The plan's T1–T10 skeleton held remarkably well — every original task
shipped roughly as scoped — but four unplanned deltas emerged, three of them real
code the plan never anticipated:

1. **The compiler line moved under us.** T1 installed `typescript` unpinned, and
   it resolved to **7.0.2 — the Go-based native port ("tsgo")**, not the classic
   5.x line the whole spec was written against. We stopped, pinned the container
   to **`typescript@5`** (a new task, #531 → tsc 5.9.3), and put native-port
   adoption on the ledger. typescript-eslint@8 actually deduped *better* against
   5.x, confirming the call.

2. **"Publish the images" hid a code prerequisite.** T9 was planned as a pure
   operational push, but the CD publish matrix (`cd-docker-publish.yml`) is
   **hand-maintained and had no `ts-node` rows** — nothing would ever build or
   publish the images. That became a new code task (#537), after which a release
   published them and T9 verified GHCR pullability.

3. **Validation caught a packaging bug that every green unit check missed.** T4
   packaged the ESLint flat config *inside the Python package*, whose ESM imports
   (`@eslint/js`, `typescript-eslint`) can't resolve from that path — so the LINT
   stage was broken for **every** consumer. T10's cold rebuild is exactly what
   surfaced it; the fix (#2771) stages the config into the consumer repo at lint
   time.

4. **A docs-repo CI gate wedged the doc PRs.** Not an epic task at all — when
   submitting the TS summary docs, the docs repo's required `docs / docs` check
   never posted on PRs because its `ci.yml` lacked the build-only `ci-docs.yml`
   job. A pre-existing, *documented-but-deferred* infra bug (epic #224 / #2726).
   One-line fix (docs #24) unblocked it; a systemic follow-up was filed (§4).

Smaller course-corrections: stale plan paths (`tests/lib/` → `tests/vergil_tooling/`;
a misattributed `_LANGUAGE_ACTION_PATTERNS`), and T7 routing `cpp → c-cpp`
alongside `typescript → javascript-typescript` through one CodeQL mapping table.

The delta is honest but small: the *language-onboarding* plan was accurate; the
misses were all at the **edges where TypeScript meets the fleet's existing
machinery** (image publishing, config packaging, docs-repo CI) — not in the
language work itself.

## §2 Lessons learned

- **Pin language compilers deliberately; they are not package-manager tools.** An
  unpinned `typescript` silently rode onto a preview native port. The fleet's
  float-the-toolchain doctrine is right for npm/formatters but wrong for the
  *compiler major*. Next language onboarding: pin the compiler major up front and
  ledger "adopt next major" explicitly.
- **Interrogate operational tasks for hidden code.** "Publish the images" read as
  free but depended on a hand-maintained matrix. When planning an operational
  task, verify its mechanism actually exists rather than assuming the push is
  mechanical.
- **Cold end-to-end validation is load-bearing — keep paying for it.** T10 caught
  the ESLint packaging break that all unit-level greens missed. "Prove it cold in
  a real consumer" is the check that earns its cost.
- **A required check must be producible on the event that gates the merge.** The
  `docs / docs` wedge is the generic failure mode of a required check that never
  posts on `pull_request`. The producibility guard existed but only at repo-init;
  the ongoing config-sync path needs it too (filed: #2788).
- **Keep the plan's Evolution log live.** We reconstructed §1 after the fact; had
  each delta been logged when it happened, this section would be evidence, not
  memory.
- **The C++-epic template paid off.** Reusing #207 (bookends, cardinality
  machinery, registry shape) made this epic mostly *declaration* — the lighter
  epic the spec predicted.

## §3 Compromises & tradeoffs

- **ESLint config via a copy-in shim (Option 1), not a real shareable npm
  package (Option 2).** Chosen because the fleet has **no npm publish/install
  infrastructure yet**. Debt: a per-lint file-staging step; the clean answer
  (`@vergil/eslint-config`) waits on that infra epic.
- **Pinned tsc 5.x; riding the classic line while tsgo matures.** Conscious —
  stable and ecosystem-aligned now; the native port is a future epic.
- **License gating is best-effort surfacing, not enforced** — matches the
  C++/fleet posture; hardening deferred.
- **T10's "CodeQL actually runs" acceptance was met by proxy, not a live run.**
  The mapping, gate, and job-name are verified; an actual CodeQL analysis on a
  live GHAS repo was not executable in-environment. A conscious v1 acceptance,
  tracked (#2782) for the first real TS-repo onboarding — an honest gap, recorded
  not hidden.
- **`cpp → c-cpp` shipped as a side effect of T7**, changing existing C++ repos'
  CodeQL SARIF category. Accepted because C++ isn't in active development; flagged
  to deal with if it surfaces.

## §4 New problems & opportunities

- **Docs-repo required-check wedge** — fixed (docs #24). Systemic follow-up filed:
  **vergil-tooling#2788** — run the CI-gates producibility guard on the
  `vrg-github-repo-config` *sync* path, not just repo-init, so a required-but-
  unproducible gate can't be silently re-applied.
- **npm publish/install infrastructure gap** — surfaced by the ESLint fix; the
  **keystone future epic** (unblocks `@vergil/eslint-config` and published-artifact
  validation). Recorded in the brainstorm (#286).
- **TypeScript 7.x / tsgo native port** — future epic (evaluate + adopt).
- **Pre-existing doc gaps the sweep surfaced** — the vergil-containers image
  catalog also omits the C++ images; the docs "Supported languages" table omits
  Ruby (no standards page to link). *Logged, not yet acted on* — candidate small
  captures.
- **Live CodeQL run for TypeScript** — tracked ad-hoc **#2782**, to be pulled into
  the first real TS-repo's epic.

## §5 What's next

Full dispositions live in the **follow-on brainstorm (#286)**; not duplicated
here. Headlines:

- **Future epics:** npm publish/install infrastructure (keystone) + `@vergil/eslint-config`;
  TypeScript 7.x / tsgo; bundling + published-artifact validation (rides on the
  npm-infra epic).
- **Demand-driven / near-term:** Node 26 when it LTS-lands; pnpm/yarn, CommonJS,
  monorepo/workspaces, plain-JS — on demand; hardened license gating (fleet-wide
  small task).
- **Tracked follow-ups:** vergil-tooling#2788 (sync-path guard), #2782 (live
  CodeQL).

---

## Appendix A — Operational notes

The cross-repo publish/release ordering that made the pipeline real (the parts a
code diff doesn't show):

1. **vergil-containers:** T1/T2 build the images → pin to `typescript@5` (#531) →
   add ts-node rows to the CD publish matrix (#537) → **release v2.1.24** builds &
   publishes `prod-/dev-ts-node:{24,22}` to GHCR. *Gotcha:* images cannot publish
   until they're in the hand-maintained matrix — the #537 gap.
2. **vergil-tooling:** T3 (config) → T4 (registry + base tsconfig) → T5/T6 → T8
   (docs); **v2.1.188** shipped the registry, **v2.1.190** the ESLint fix.
   *Gotcha:* T10's cold rebuild pulls the *released, installed* tooling — so the
   ESLint fix had to be released before re-validating.
3. **vergil-actions:** T7 (reusable CI + CodeQL mapping) → releases carrying it.
4. **Operational verification last:** T9 confirms GHCR pullability; T10 stands up
   an ephemeral TS repo and runs the full pipeline cold, including a known-CVE
   audit-detect proof.

## Appendix B — Extended metrics

- **Span:** ~1 day (2026-08-11 14:47Z → 2026-08-12 ~13:49Z last task PR).
- **Throughput:** 16 merged PRs + 2 operational tasks across 5 repos.
- **Discovered work:** 3 unplanned code fixes (5.x pin, publish matrix, ESLint
  resolution) + 1 out-of-epic infra fix (docs gate) + 2 tracked follow-ups.
- **Epic-attributed releases:** containers v2.1.24; tooling v2.1.188, v2.1.190;
  vergil-actions release(s) for T7.
