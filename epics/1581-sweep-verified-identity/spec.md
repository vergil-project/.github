# Verified-identity authorization for the epic sweep — design

- **Epic:** vergil-project/.github#1581
- **Planning issue:** vergil-project/vergil-tooling#3796 (review history is in
  its comments)
- **Status:** draft, in design review
- **Repos:** vergil-tooling, vergil-actions
- **Related:** planning item vergil-project/.github#593 (the identity-mode
  review for the identity-gated tools, which builds on §4.1); epic
  vergil-project/.github#1064 (the automation identity and its credential
  check); epic vergil-project/.github#1384 (the `automation` environment);
  advisory GHSA-r5m3-gp2j-fqmh (private)

## 1. Intent

`vrg-epic-audit --close` writes across a whole org: it closes drifted tasks
and epics, drains archives and links orphans. Its gate allows two callers: a
human, and the scheduled reconciliation sweep (`ops-epic-sweep`). Today both
are recognised from environment signals. The sweep is recognised by a
dedicated flag that the workflow sets by hand, `VRG_EPIC_SWEEP=1`, and the
human by the first-signal resolver, `identity_mode.is_human()`.

The sweep already runs with the org automation App's installation token, and
vergil-tooling can already prove that: `identity_mode.resolve()` returns
`automation` only after the token's GraphQL `viewer.login` matches
`<slug>[bot]` (`lib/automation_identity.verify_token_is_app`). This epic moves
the gate onto that proof. It also adds one shared, fail-closed check for "an
agent identity is indicated", used by the gate's human half, which the wider
identity-mode review (vergil-project/.github#593) then adopts across the other
identity-gated commands.

The work is low severity: the gate is a client-side soft gate, and GitHub
still bounds every call by the token actually held. It must not block the
unattended lifecycle work.

## 2. Goals

1. The sweep's `--close` is authorized by the verified automation identity,
   not by a flag.
2. The gate's human half refuses whenever any agent signal is present.
3. `VRG_EPIC_SWEEP` is retired: no code reads it and no workflow sets it.
4. The one other reader of the flag, the `vrg-security-record list --details`
   refusal, keeps refusing in CI after the flag is gone.
5. The rollout never leaves a scheduled sweep unable to close.
6. The shared check ships in a form #593 can adopt unchanged.

## 3. Non-goals and anti-goals

### Non-goals

- Moving the other identity-gated commands onto the shared check, the
  inventory of `vrg-*` commands by intended identity, and the regression guard
  that keeps new human-only commands on it. All are #593.
- Changing `identity_mode.resolve()` or its signal order. #593 decides that.
- Identity gating of the VM lifecycle commands (#593).
- Workflow-provenance verification (§4.5).

### Anti-goals

- A transition window where `--close` accepts both the flag and the verified
  identity. The rollout order (§5) makes one unnecessary.
- A gate that turns a failed automation verification into a plain refusal or a
  fall-through to the human check. A run that names the automation App and
  cannot prove it is a configuration error and exits 1 naming it (no silent
  failures).
- Replacing the flag with another hand-set environment signal.

## 4. Design

### 4.1 The shared agent check

Add one predicate to `vergil_tooling.lib.identity_mode`, true when any
resolution signal indicates an agent identity:

```python
def agent_indicated() -> bool:
    return is_agent() or any_agent_signal()
```

It is the fail-closed check the release and promote commands already write
out inline (`bin/vrg_release.py`, `bin/vrg_promote.py`, `bin/vrg_vm.py`,
`bin/vrg_host.py`, `lib/promote.py`), named once. A human-only command refuses
when it is true. Its answer is false for `automation` and for `human` with no
agent signal; true for `user` or `audit`, and for any mode while the mode file,
`app.pem` or `VRG_APP_ID` indicates an agent.

The predicate stays correct under any later change to `resolve()`: if #593
makes `resolve()` itself refuse to let a mode signal outrank a provisioned
agent signal, the `any_agent_signal()` half becomes redundant, never wrong.

This epic uses it in one place (§4.2). The five inline copies stay as they are;
moving them is #593's.

Tests: one per signal combination that matters, at least each agent signal
alone, each agent signal alongside a `human` mode signal, `automation`
verified with no agent signal, and no signal at all.

### 4.2 The `--close` gate

`--close` is allowed when either holds:

- **the verified automation identity:** `identity_mode.is_automation()`; or
- **a human with no agent indicated:** `identity_mode.is_human()` and not
  `identity_mode.agent_indicated()`.

Anything else is refused, before any network work beyond the identity check,
with a message that names both allowed callers.

`is_automation()` makes a network call only when `VRG_AUTOMATION_APP` is set.
With it set, a failed verification (the token is not that App's, or its
identity cannot be read) or contradictory signals raise
`AutomationIdentityError`; `--close` exits 1 with that error, never a refusal
or a fall-through (§3). With it unset, the gate resolves locally as before.

The read-only report and the `--epic` and `--untrusted` modes are unchanged.

### 4.3 Retiring `VRG_EPIC_SWEEP`

- `bin/vrg_epic_audit.py`: `_SWEEP_ENV` and `_automated_sweep()` go; the
  refusal message stops naming the flag.
- `.github/workflows/ops-epic-sweep.yml` (vergil-actions): the reconcile step
  sets `VRG_AUTOMATION_APP: ${{ vars.VRG_AUTOMATION_APP }}`, as the report job
  already does, and later drops `VRG_EPIC_SWEEP` (§5).
- The docs that name the flag: `docs/site/docs/reference/cli-tools-overview.md`,
  `docs/site/docs/reference/dev/security-record.md` and
  `docs/site/docs/standards/github-issues.md`. Past release notes stay as
  written.

### 4.4 The `--details` refusal

`vrg-security-record list --details` prints advisory text and refuses under
the sweep, so that text never reaches a CI log. It keys that refusal on
`VRG_EPIC_SWEEP`, which stops being set (§4.3); left alone, the refusal would
silently stop applying.

It is re-anchored to running under GitHub Actions (`GITHUB_ACTIONS` is
`true`): no workflow calls `--details`, and every Actions log is the exposure
the refusal exists to prevent, not only the sweep's. An ambient signal is
acceptable here because it can only add a refusal, never grant anything. The
message names the new condition.

### 4.5 Why not workflow provenance

A stronger alternative would also require a GitHub OIDC token and check its
`job_workflow_ref` and `repository` claims against an allowlist of workflows.
This design leaves that out:

- The automation App's keys live only in the `automation` environment, whose
  deployment branch policy admits only `develop` and `main`
  (`lib/github_config.desired_automation_environment`, kept converged by the
  daily config audit; epic #1384). Only code already merged to a protected
  branch can mint the token, so an allowlist would separate one reviewed
  workflow from another.
- The gate is client-side. A holder of the token can call GitHub directly
  without vergil-tooling, so a stricter check of the automation path in the
  tooling adds little over what GitHub already enforces on the token.

Revisit it if the automation App's keys ever become available outside the
`automation` environment, or if a server-side decision starts to depend on the
tooling's identity check.

### 4.6 The hand-off to #593

When the tasks are filed, a public comment on #593 records that the shared
agent check (§4.1) ships in this epic and that #593 adopts it. The comment
names the mechanism only. #593's advisories stay with #593: this epic fixes
none of them.

## 5. Rollout

The sweep installs vergil-tooling from its caller's `v2.1` line pin at run
time, and runs `ops-epic-sweep.yml` from the `v2.1` tag of vergil-actions. So
each change reaches the sweep through a release, and releases are human steps.

1. **vergil-actions:** the reconcile step also sets `VRG_AUTOMATION_APP`. With
   the released tooling this is harmless: it resolves `automation`, and the
   flag still authorizes `--close`.
2. **vergil-actions release** moves `v2.1`; a sweep run after it succeeds
   with `VRG_AUTOMATION_APP` set.
3. **vergil-tooling:** §4.1 to §4.4, merged only after step 2.
4. **vergil-tooling release** on the `v2.1` line.
5. **Validation:** a sweep run on the released tooling completes `--close` as
   the verified automation identity.
6. **vergil-actions:** drop `VRG_EPIC_SWEEP` from the reconcile step; the next
   vergil-actions release carries it.

The reverse order breaks the sweep: a tooling release without the flag, run by
a workflow that does not yet set `VRG_AUTOMATION_APP`, refuses `--close`.

## 6. Verification

- Unit tests for §4.1, §4.2 and §4.4, including each mixed-signal case and the
  failed-verification exit.
- Step 2's sweep run proves the workflow change against the released tooling.
- Step 5's validation proves the new gate in the real sweep, in every org whose
  `.github` calls `ops-epic-sweep.yml`.

## 7. Left to the builder

- The predicate's final name, if `agent_indicated` collides with anything.
- The exact refusal and error message text.
- Whether anything else on the `--close` path behaves differently under the
  `automation` mode than under the old flag (for example the trust layer's
  view of the caller); step 2 is where that shows, and any difference is fixed
  before step 3 merges.
