# Shared SARIF-gate exception allowlist — documented exceptions for `evaluate_findings`

- **Epic:** vergil-project/.github#94
- **Status:** Reviewed design (2026-07-04) — post-pushback
- **Brainstorm source:** superpowers brainstorming session, 2026-07-04
- **Review:** paad pushback (3 issues, all resolved) — reframed to the shared gate
  across all three scanner CLIs, scope-by-tool-identity, optional `path`
- **Implemented in:** vergil-tooling (tasks filed there at implementation time)
- **Motivating case:** logical-minds-foundry/mq-resiliency-lab-for-linux#491 / PR #492

## 1. Summary

`evaluate_findings` (in `vergil-tooling`'s `lib/sarif.py`) is the **shared** CI
security gate: it reads a scanner's SARIF, keeps findings at a gating severity,
and fails the build if any remain. It is consumed by **all three** scanner CLIs —
`vrg-sarif-evaluate` (CodeQL), `vrg-semgrep-scan` (Semgrep), and `vrg-trivy-scan`
(Trivy) — each running separately over its own tool's SARIF.

The gate has **no way to record a reviewed, justified exception**: no allowlist,
and it ignores GitHub code scanning's own dismiss-with-reason model. So a
*legitimate, intentional* finding hard-blocks every merge, and the only escapes
are silent evasion (rule-dodging synonyms or computed values) or code contortions
to appease a context-free heuristic.

This epic adds the missing half of the review-and-acknowledge model: a
**version-controlled, justified exception allowlist** that the shared gate honors,
so an accepted finding is *documented* — `tool` + `rule` + optional `path` +
`reason` + `issue` — rather than suppressed silently or worked around in code. The
documentation **is** the mechanism: an exception cannot exist without a stated
reason and a tracking issue. Because it lives in the shared `evaluate_findings`,
**all three gates** get it uniformly.

### Motivating case

Enabling code scanning on a newly-public repo surfaced CodeQL
`py/overly-permissive-file` on an **intentional** group-readable, non-sensitive
metrics file (the node-exporter textfile-collector pattern — the `.prom` must be
readable by a separate service user, so `0o640` via a shared group is
least-privilege). The query flags any `os.chmod`/`os.open` literal that sets a
group or world bit, read or write, with **zero context** — a false positive here,
with no sanctioned way to express "reviewed, accepted" through our gate. The need
is general, not bespoke to that finding or that scanner.

## 2. The problem in detail

Three facts combine into a hard block:

1. **The gate fails on the scan's own SARIF.** `evaluate_findings` keeps every
   result whose `level` is in the gating set (default `{warning, error}`) and
   fails if any remain. No allowlist, no nuance beyond the severity set.
2. **It ignores SARIF `suppressions` and GitHub-UI dismissals.** A fresh scan
   re-emits the finding into the SARIF the gate reads, so a Security-tab dismissal
   never reaches it, and an inline suppression (which CodeQL wouldn't honor
   anyway) would be dropped too.
3. **Scanners can't tell sensitive from non-sensitive / exploitable from not.**
   The rules are blanket heuristics — right in general, wrong for a specific
   reviewed case. That is exactly what the platform expects a human to accept.

GitHub's model already includes the "this is fine" half (dismiss-with-reason); our
pipeline dropped it. This restores it, in a form that lives with the code and
applies to every scanner behind the shared gate.

## 3. Design

### 3.1 Config surface — `vergil.toml`

Exceptions are repeated tables in the repo's existing `vergil.toml`, loaded via
the existing `read_config(repo_root)` path (`tomllib` parses `[[…]]`
array-of-tables into a list natively):

```toml
[[security.sarif-exception]]
tool   = "codeql"                             # which scanner this exception is for
rule   = "py/overly-permissive-file"          # the SARIF ruleId
path   = "clients/app_requester.py"           # OPTIONAL, repo-relative, glob-capable
reason = "Non-sensitive metrics .prom must be group-readable for node-exporter (a separate service user); 0o640 via the node_exporter group is least-privilege."
issue  = "logical-minds-foundry/mq-resiliency-lab-for-linux#491"
```

- `tool`, `rule`, `reason`, `issue` are **required**. A table missing `reason` or
  `issue` is a **config error** (the gate fails loud) — an undocumented exception
  is not an exception. `tool` must be one of `codeql` / `semgrep` / `trivy`
  (matched against the SARIF's `tool.driver.name`; the implementation plan
  confirms each scanner's actual `driver.name` string).
- `path` is **optional**: omitted ⇒ match by `tool` + `rule` only (every instance
  of that rule for that scanner — the natural posture for a Trivy CVE that appears
  in several manifests, or a finding with no file location); present ⇒ repo-
  relative, glob-capable, matched against the finding's file.

### 3.2 Matching and per-scanner scoping

A finding is excepted **iff** an exception's `rule` equals the finding's rule id
**and** (`path` is absent **or** its glob matches the finding's file).

**Per-scanner scoping is by tool identity, not rule membership.** Each of the
three CLIs runs separately over one tool's SARIF, so an exception must be
associated with the scanner it belongs to — otherwise fail-on-stale (§3.3) would
false-fail (a CodeQL exception matches nothing when the Semgrep CLI runs). Each
run reads the SARIF's `tool.driver.name` and **only applies and stale-checks
exceptions whose `tool` matches it.** Exceptions for other tools are out of scope
for that run — neither applied nor judged stale.

> Scoping is by `tool.driver.name`, **not** `tool.driver.rules`. Only some
> scanners (CodeQL) list their full ruleset in `driver.rules`; Semgrep and Trivy
> typically list only rules that produced results, so a fixed finding would leave
> no `driver.rules` entry and its exception would look "out of scope" rather than
> "stale" — silently breaking fail-on-stale for those tools. Tool identity avoids
> that entirely.

### 3.3 Gate behavior

Per run, over the exceptions in scope for this scanner (matched by `tool`),
applied **after** the severity filter (an exception only ever removes a finding
that would otherwise gate):

- **Skip** any severity-gated finding matched by an in-scope exception.
- **Report** every applied exception and how many findings it matched, via the
  existing output lib (`emit_warning` / `write_summary`) so it lands in the job
  summary — never silent. An allowlist must not be able to hide a regression.
- **Fail on stale.** An in-scope exception (its `tool` matches this run's scanner)
  that matched **zero** findings is stale — the finding was fixed and the
  exception must be removed. This mirrors ruff `RUF100` (unused `noqa`) and mypy
  `--warn-unused-ignores`: an obsolete suppression is itself an error, so hygiene
  auto-corrects. Out-of-scope exceptions (other tools) are never judged stale
  here.
- Findings with no matching in-scope exception fail the gate as before.

### 3.4 Component boundaries

- **`lib/sarif.py`** — `evaluate_findings` grows an `exceptions` argument (default
  none, so today's callers are unaffected until wired). Add the exception
  dataclass, `tool`+`rule`+optional-`path`(glob) matching, tool-identity scoping
  (read `tool.driver.name`), and stale detection. Pure and unit-testable.
- **Config loader** — parse and validate the `[[security.sarif-exception]]`
  tables from `vergil.toml` via `read_config(repo_root)`, enforcing required
  fields and the `tool` enum. Each CLI resolves `repo_root` (git root / cwd) to
  find `vergil.toml`.
- **All three CLIs** — `bin/vrg_sarif_evaluate.py`, `bin/vrg_semgrep_scan.py`,
  `bin/vrg_trivy_scan.py` each load the exceptions and pass them into
  `evaluate_findings`, and emit the applied/stale report. One mechanism, three
  gates.

## 4. Testing

Pure-function unit tests on `lib/sarif.py`: exact and glob `path` matching;
optional-`path` (rule-wide) matching; **tool-identity scoping** (an exception for
another tool is neither applied nor stale here); required-field + `tool`-enum
validation (missing `reason`/`issue` or bad `tool` errors); stale detection
(in-scope zero-match fails; other-tool zero-match does **not**); severity ordering
(exception only removes an already-gated finding); and the applied-exception
report content. Plus a thin per-CLI test that each of the three CLIs loads
exceptions and passes them through.

## 5. Non-goals

- No changes to CodeQL/Semgrep/Trivy rules or query suites.
- No inline source-comment suppression.
- No honoring of GitHub-UI dismissals or SARIF `suppressions` (a fresh scan
  doesn't carry them, and we want exceptions reviewable in-repo).
- **No exception *governance* in v1** — no severity caps, no expiry/auto-review.
  This builds the thing to manage; managing it (the exception-lifecycle paradigm)
  is deliberate future work. Fail-on-stale is the one hygiene lever in v1, and it
  covers the common case (a fixed finding / patched CVE forces removal).

## 6. Known tradeoffs

- **`vergil.toml` growth.** Chosen deliberately for one audited config surface
  reviewers already read, and because fail-on-stale actively fights accumulation.
  The risk is real — ruff/mypy ignore-lists have bloated before. **Escape hatch,
  if it bloats:** move exceptions to a dedicated **committed** file (e.g. a
  repo-root `sarif-exceptions.toml`). Explicitly **not** `.vergil/…` — `.vergil`
  is gitignored scratch and cannot hold a version-controlled file.
- **`tool`+`rule` (no `path`) can over-match** other instances of that rule for
  that scanner. Intended for the CVE-wide / pathless case; the applied-exception
  report surfaces the match count so an unexpected over-match is visible.
- **Excepting a Trivy CVE accepts a known vulnerability** — heavier than
  excepting a CodeQL false positive. v1 treats all tools uniformly; the required
  `reason`+`issue` document *why*, the report keeps it visible, and fail-on-stale
  forces removal once the CVE is patched out. Severity-aware governance is future
  work (§5).

## 7. First consumer (separate, cross-org)

`logical-minds-foundry/mq-resiliency-lab-for-linux` will allowlist the
`app_requester.py` `py/overly-permissive-file` finding to unblock PR #492. That is
a logical-minds-foundry task, blocked on this epic — tracked in that repo, not
linked here (cross-org linkage is out of scope for the epic model).
