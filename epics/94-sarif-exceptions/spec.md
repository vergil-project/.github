# Documented security-scan exceptions — `vrg-sarif-evaluate` allowlist

- **Epic:** vergil-project/.github#94
- **Status:** Reviewed design (2026-07-04) — brainstorming session
- **Brainstorm source:** superpowers brainstorming session, 2026-07-04
- **Implemented in:** vergil-tooling (tasks filed there at implementation time)
- **Motivating case:** logical-minds-foundry/mq-resiliency-lab-for-linux#491 / PR #492

## 1. Summary

`vrg-sarif-evaluate` is the CI security gate: it reads a scanner's SARIF output
and fails the build on any finding at a gating severity. It has **no way to
record a reviewed, justified exception** — no allowlist, and it ignores GitHub
code scanning's own dismiss-with-reason model. So a *legitimate, intentional*
finding hard-blocks every merge, and the only escapes are silent evasion
(rule-dodging synonyms or computed modes) or code contortions (e.g. POSIX ACLs)
to appease a context-free heuristic.

This epic builds the missing half of the review-and-acknowledge model: a
**version-controlled, justified exception allowlist** that the gate honors, so an
accepted finding is *documented* — `rule` + `path` + `reason` + `issue` — rather
than suppressed silently or worked around in code. The documentation **is** the
mechanism: an exception cannot exist without a stated reason and a tracking issue.

### Motivating case

Enabling code scanning on a newly-public repo surfaced CodeQL
`py/overly-permissive-file` on an **intentional** group-readable, non-sensitive
metrics file (the node-exporter textfile-collector pattern — the `.prom` must be
readable by a separate service user, so `0o640` via a shared group is
least-privilege). The query flags any `os.chmod`/`os.open` literal that sets a
group or world bit, read or write, with **zero context** — a false positive here,
but with no sanctioned way to express "reviewed, accepted" through our gate.

## 2. The problem in detail

Three facts combine into a hard block:

1. **The gate fails on the scan's own SARIF.** `vrg-sarif-evaluate` keeps every
   result whose `level ∈ {warning, error}` and fails if any remain. There is no
   allowlist and no severity nuance beyond the `--severity` set.
2. **It ignores SARIF `suppressions` and GitHub UI dismissals.** A fresh scan
   re-emits the finding into the SARIF the gate reads, so a Security-tab dismissal
   never reaches it, and an inline suppression (which CodeQL wouldn't honor
   anyway) would be dropped too.
3. **The scanners can't tell sensitive from non-sensitive.** CodeQL's rule is a
   blanket least-privilege heuristic. It is *right* to flag broad file
   permissions in general and *wrong* for this specific file — exactly the case
   the platform expects a human to review and accept.

GitHub's model already includes the "this is fine" half (dismiss-with-reason); our
pipeline dropped it. This restores it, in a form that lives with the code.

## 3. Design

### 3.1 Config surface — `vergil.toml`

Exceptions are repeated tables in the repo's existing `vergil.toml`, loaded via
the existing repo-config path:

```toml
[[security.sarif-exception]]
rule   = "py/overly-permissive-file"
path   = "clients/app_requester.py"          # repo-relative, glob-capable
reason = "Non-sensitive metrics .prom must be group-readable for node-exporter (a separate service user); 0o640 via the node_exporter group is least-privilege."
issue  = "logical-minds-foundry/mq-resiliency-lab-for-linux#491"
```

- `rule`, `path`, `reason`, `issue` are **all required**. A table missing `reason`
  or `issue` is a **config error** (the gate fails loud) — an undocumented
  exception is not an exception.
- `path` is repo-relative and **glob-capable** (e.g. `clients/*.py`).

### 3.2 Matching and per-scanner scoping

A finding is excepted **iff** `rule` equals the finding's rule id **and** `path`
matches the finding's file (glob).

**Applying** an exception needs only a matching result: any finding whose
`rule`/`path` an exception matches is skipped, in whatever SARIF it appears.

**Per-scanner scoping (the load-bearing decision) is about the *stale* check.**
`vrg-sarif-evaluate` runs once per scanner (separately over CodeQL's SARIF,
Semgrep's, Trivy's), so a naive "this exception matched nothing in *this* SARIF →
stale" would false-fail — a CodeQL-rule exception matches nothing when the Semgrep
gate runs. So an exception is only judged **stale** by a run whose scanner
*owns* its rule, determined from the SARIF's `tool.driver.rules` (the ruleset the
scanner ran): in-scope (rule ∈ this scanner's ruleset) + zero matches ⇒ stale;
rule not in this scanner's ruleset ⇒ out of scope, never stale here. A CodeQL
exception is thus only ever called stale by the CodeQL run. (Plan concern: confirm
each scanner populates `tool.driver.rules`; where it is absent, degrade
stale-detection to best-effort rather than false-fail.)

### 3.3 Gate behavior

Per run, over the in-scope exceptions:

- **Skip** any finding matched by an exception (match is `rule` + `path` on the
  result — independent of the scoping used for the stale check).
- **Report** every applied exception and how many findings it matched — never
  silent. An allowlist must not be able to hide a regression.
- **Fail on stale.** An in-scope exception (its `rule` *is* in this scanner's
  ruleset) that matched **zero** findings is stale — the code was fixed and the
  exception must be removed. This mirrors ruff `RUF100` (unused `noqa`) and mypy
  `--warn-unused-ignores`: an obsolete suppression is itself an error, so hygiene
  auto-corrects. Out-of-scope exceptions are ignored, never stale.
- Findings with no matching exception fail the gate as before.

### 3.4 Component boundaries

- `lib/sarif.py` — gains the exception dataclass, glob matching, per-scanner
  scoping (read `tool.driver.rules`), and stale detection. Pure and
  unit-testable; `evaluate_findings` grows an `exceptions` argument.
- `lib/repo_config.py` (or a small dedicated loader) — parse and validate the
  `[[security.sarif-exception]]` tables from `vergil.toml`, enforcing required
  fields.
- `bin/vrg_sarif_evaluate.py` — load the exceptions, pass them through, and emit
  the applied/stale report alongside the existing findings output.

## 4. Testing

Pure-function unit tests: exact and glob path matching; per-scanner scoping
(in-scope vs another tool's rule); required-field validation (missing
`reason`/`issue` errors); stale detection (in-scope zero-match fails; an
other-tool zero-match does **not**); and the applied-exception report content.

## 5. Non-goals

- No changes to CodeQL/Semgrep/Trivy rules or query suites.
- No inline source-comment suppression.
- No honoring of GitHub-UI dismissals or SARIF `suppressions` (a fresh scan
  doesn't carry them, and we want exceptions reviewable in-repo).
- No expiry/auto-review of exceptions in v1 (fail-on-stale is the hygiene lever).

## 6. Known tradeoffs

- **`vergil.toml` growth.** Chosen deliberately for one audited config surface
  reviewers already read, and because fail-on-stale actively fights accumulation.
  The risk is real — ruff/mypy ignore-lists have bloated before. **Escape hatch,
  if it bloats:** move exceptions to a dedicated **committed** file (e.g. a
  repo-root `sarif-exceptions.toml`). Explicitly **not** `.vergil/…` — `.vergil`
  is gitignored scratch and cannot hold a version-controlled file.
- **`rule` + `path` can over-match** a *new* instance of the same rule in the same
  file. Accepted for v1: the applied-exception report surfaces the match count, so
  an unexpected over-match is visible; line/fingerprint precision was rejected as
  brittle (line drift) or non-human-authorable (fingerprints).

## 7. First consumer (separate, cross-org)

`logical-minds-foundry/mq-resiliency-lab-for-linux` will allowlist the
`app_requester.py` `py/overly-permissive-file` finding to unblock PR #492. That is
a logical-minds-foundry task, blocked on this epic — tracked in that repo, not
linked here (cross-org linkage is out of scope for the epic model).
