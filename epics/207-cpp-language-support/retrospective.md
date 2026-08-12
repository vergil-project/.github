# C++ language support — Retrospective

- **Epic:** vergil-project/.github#207
- **Span:** 2026-08-04 18:35Z → 2026-08-05 (~24 hours, two calendar days).
- **Partners:** `spec.md` → `plan.md` → this. Read them in that order.

## §0 At a glance

We set out to add **C++ as a first-class supported language** on the fleet's
containerized, multi-version model — and shipped a **dual-compiler (Clang primary -
GCC secondary) four-image pipeline** (`clang-20/19`, `gcc-14/13`), validated
green end-to-end: warnings-to-11 `-Werror` builds, **100% per-compiler coverage**,
ASan/UBSan, and a **working `conan audit` dependency gate that demonstrably
detects CVEs**. The one thing we *didn't* ship — the generic fleet-secrets
mechanism the epic surfaced — is deliberately parked as its own future epic. v1 is
lean, real, and honest about its edges.

**Work delivered — 20 PRs across 4 repos, plus 5 operational/decision closes:**

| Task | Repo | PR | What it did | Merged |
|---|---|---|---|---|
| #208 | .github | #212 | Spec + plan published | 08-04 19:29 |
| #2552 | tooling | #2559 | Per-kind CI-gate cardinality (`once` vs `per-version`) | 08-04 19:47 |
| #2553 | tooling | #2560 | `cpp` enum + `[cpp]` config block | 08-04 19:50 |
| #467 | containers | #470 | Clang images + shared C++ analysis base | 08-04 19:54 |
| #2554 | tooling | #2565 | C++ language registry entry (the tool matrix) | 08-04 20:21 |
| #2556 | tooling | #2566 | CodeQL `cpp` + repo-init wiring | 08-04 20:23 |
| #468 | containers | #475 | GCC images on the shared base | 08-04 20:54 |
| #2555 | tooling | #2567 | C++ image resolution + project detection | 08-04 20:54 |
| #2557 | tooling | #2568 | C++ standards docs | 08-04 20:57 |
| #792 | actions | #793 | Reusable CI → compiler-family images (cardinality-aligned) | 08-04 21:34 |
| #480 | containers | #482 | **Publish C++ images to GHCR** (discovered gap) | 08-04 21:42 |
| #2572 | tooling | #2573 | AUDIT → OSV-Scanner; Conan build_type + gcovr fixes | 08-05 11:00 |
| #487 | containers | #488 | Close T11 image gaps (profile, clang-tidy, llvm-cov, osv) | 08-05 11:07 |
| #802 | actions | #803 | Wire the Conan token into ci-audit | 08-05 14:38 |
| #2578 | tooling | #2580 | Conan token VM injector (bespoke secret delivery) | 08-05 14:38 |
| #2579 | tooling | #2581 | **AUDIT reverted to `conan audit`** + cppcheck googletest | 08-05 14:41 |
| #2585 | tooling | #2586 | cppcheck curated `--enable` + build-tree exclusion | 08-05 16:53 |
| #2551 | tooling | #2590 | Docs sweep — corrected to final shipped state | 08-05 18:54 |
| #217 | .github | #218 | C++ added to the main project README | 08-05 18:57 |
| #503 | containers | #504 | Remove now-unused osv-scanner from images | 08-05 18:59 |

**Operational / decision closes (no PR):** T10 deploy #469 (images published) ·
T11 validation #2558 (cold-rebuild + audit-detection proof) · ledger #209 ·
secrets brainstorm #216 (parked → idea #222) · Conan-access setup #2577.

**Rollup:** 26 children (25 closed + this). **Repos:** `.github`, `vergil-tooling`
(13), `vergil-containers` (6), `vergil-actions` (2). **Releases in-window:**
tooling ×5 (v2.1.168→172), containers ×7 (v2.1.14→20), actions ×3
(v2.1.19→21) — 15 total; the high count is itself a finding (see §1/§2).

## §1 How the plan evolved

The plan's *shape* held — every task in `plan.md` shipped roughly as scoped — but
execution added six tasks the plan didn't foresee, almost all from **reality
pushing back at the boundaries the plan drew on paper.**

- **Prebuilt-only did its job by forcing a fallback.** `gcc-15` isn't a prebuilt
  package on Debian trixie, so the §3.5 "never build from source" rule triggered
  the documented fallback to `gcc-14/13`. The matrix bent exactly where the spec
  said it would.
- **Repo idioms beat plan filenames.** T1/T2 ignored the plan's literal
  `Dockerfile.base`/`.clang` names and built to the repo's `@include`-fragment
  structure — the right call ("match existing patterns"), and a reminder that a
  plan written without reading the target repo's conventions will name things
  that shouldn't exist.
- **"Built" ≠ "published" (task #480).** The plan implicitly assumed the images
  existed once their Dockerfiles merged. The deploy step (T10) found they were
  wired only into the *local* builder, never the GHCR publish pipeline — a whole
  task the plan missed, caught precisely because deployment was a first-class
  operational task rather than an afterthought.
- **T11 earned the entire "operational validation" bookend.** The cold rebuild
  caught a *cluster* of integration gaps that 4550 green unit tests never
  could — no default Conan profile, a Conan-Release-vs-CMake-Debug mismatch,
  config-relative `gcovr` paths, Clang using GNU `gcov` instead of `llvm-cov`,
  and `run-clang-tidy` off PATH — spawning the #487/#2572 fix cycle. This is the
  single strongest argument in the epic for validating cold, end-to-end, against
  published artifacts.
- **The audit saga — one requirement, one reversal, one correction.** AUDIT went
  `conan audit` → OSV-Scanner (to dodge the token/rate-limit) → **back to `conan
  audit`** when T11 proved OSV.dev carries *zero* ConanCenter advisories — a
  silent false all-clear. The "prove the audit **detects** a known CVE"
  acceptance criterion, added during `paad:alignment`, is the only reason this
  was caught instead of shipped. It fired twice: killing OSV, then confirming
  `conan audit` actually flags `zlib/1.2.11`.
- **The token dragged a whole subsystem into view.** Needing a ConanCenter token
  exposed that the fleet has **no generic runtime-secret mechanism** — every
  secret is bespoke. We shipped a bespoke injector (option A) mirroring the
  Anthropic token and, in doing so, became the **first real consumer of the
  dormant `[container].env-prefixes` passthrough.** The generic mechanism (option
  B) was deliberately deferred (§4/§5).

## §2 Lessons learned

- **Green unit tests are not a working system.** The gap between "4550 tests pass"
  and the cold-rebuild reality was large and entirely in wiring/packaging. For any
  infra/language epic, the operational validation bookend is load-bearing, not
  ceremonial — and it must run against *published* artifacts (the stale-local-cache
  trap bit us mid-validation and would have produced a false pass).
- **Verify data coverage, not just tool capability.** The OSV-Scanner
  recommendation correctly verified "does it parse `conan.lock`?" (yes) but missed
  "does OSV.dev *have* ConanCenter data?" (no). A sourced recommendation can be
  confidently, precisely wrong when it checks the wrong axis. Checking *efficacy*
  (does it detect?) is what a capability check can't substitute for.
- **Make security gates prove themselves.** "The audit runs green" is worthless if
  the audit can't see anything. Requiring a demonstrated vulnerable→flagged /
  clean→passed cycle turned a would-be no-op into a caught defect.
- **Fail closed, loudly.** The decision to *not* add a token-absent skip — a
  missing audit token hard-fails rather than silently skipping — is the doctrine
  working: a security gate that can't run should fail, prompting per-org setup.
- **Runtime-installed tooling is a feature.** Because containers install
  `vergil-tooling` at run time (not baked), registry/config fixes shipped via a
  plain release with *no image rebuild* — which repeatedly shortened the fix loop.
- **Batch PR handoffs by repository.** Mid-epic we corrected "one global batch" to
  "one batch per repo" — `vrg-submit-pr` is per-repo, and holding a ready repo's
  PRs hostage to another repo's slower task is pure latency.

## §3 Compromises & tradeoffs

- **Portable warning set, not per-compiler.** The registry has no "compiler
  dimension," and `-Werror` makes a flag only one compiler knows a hard error, so
  v1 uses one portable GCC/Clang set. Per-compiler tuning is deferred; the
  two-diagnostics win still comes from two compilers reading the shared set.
- **Bespoke secret delivery (A) over the generic mechanism (B).** A conscious "ship
  now, do it right later" call. The debt is real and named: there is *no* general
  runtime-secret path, secrets are undocumented as a whole, and bootstrap/DR would
  be painful. Captured as its own future epic (idea #222), not swept under a rug.
- **Free conancenter token, rate limits accepted.** The paid path is enterprise
  JFrog (~$950–3,800+/mo), not a cheap token — out of proportion for one gate. We
  took the free tier and hold the two-tier (release-time) scanning idea as the
  escape hatch if rate limits ever bite.
- **`gcc-14/13`, not `14/15`.** Forced by prebuilt availability; still two recent
  majors per family.
- **Best-effort license surfacing** (`conan graph info`) rather than hardened
  allowlist enforcement — deferred to v2.

## §4 New problems & opportunities

- **No generic fleet-secrets mechanism** *(the big one)* — surfaced here, not
  created here. Every secret is bespoke; there's no inventory doc; bootstrap/DR is
  undocumented. → parked as **idea .github#222** for a dedicated future epic
  (generic mechanism + inventory + bootstrap/DR).
- **Two-tier scanning** (cheap checks per-PR, expensive/higher-precision checks at
  release time) — a general CI-architecture idea, and the standing mitigation if
  the free `conan audit` rate limit becomes a bottleneck. → recorded on #209.
- **Per-compiler warning sets** — considered and **declined** (marginal value vs. a
  registry compiler-dimension). Recorded so it isn't re-litigated.
- **osv-scanner cruft** — the OSV→conan-audit reversal left an unused tool in the
  images; **cleaned up in-epic** (#503) rather than left as debt.
- **Release-cadence churn** — 15 releases in ~24h, driven by the fix iterations.
  Not wrong, but a signal that a tighter local cold-validation loop *before*
  first release could have collapsed several of them.

## §5 What's next

- **C++ improvements (v2)** — idea **.github#220**: c++23 sweep, libc++ axis,
  TSan/MSan, dual-ABI sweep, hardened license gating, cross-arch test matrix.
- **Swift** — idea **.github#221**: its own future epic, enabled by this C++/LLVM
  toolchain work.
- **Fleet secrets management** — idea **.github#222**: the generic mechanism +
  inventory docs + bootstrap/DR (§3/§4).

Adjudicated on the deferral ledger (#209); nothing was pulled into v1.

## Appendix A — Operational notes

- **Release order matters:** `vergil-tooling` releases *before* `vergil-containers`,
  because the images install the tooling at build time — a tooling fix must be
  released first for an image rebuild to pick it up.
- **Images are multi-arch** (amd64 + arm64); validation ran on arm64.
- **Token delivery chain (bespoke):** host token file → `identities.toml`
  `conan_audit_token_path` → `inject_credentials` writes `~/.config/vergil/conan.env`
  → sourced into the agent env → `[container].env-prefixes` passthrough → inside the
  container → `conan audit` reads `CONAN_AUDIT_PROVIDER_TOKEN_CONANCENTER`. CI uses
  the explicit-secret chain (no `secrets: inherit`).
- **Publish:** `dev-cpp-*` publish on the develop push; `prod-cpp-*` require a
  release to `main`; each new GHCR package needs repo Write granted once.
