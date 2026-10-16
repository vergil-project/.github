# macOS host dependency management (`vrg-host`) — Retrospective

## §0 At a glance

We set out to bring order to the unmanaged macOS development host: one tool,
`vrg-host`, that lists the Virgil-critical host dependencies from a
tooling-owned manifest, reports installed-vs-latest drift per dependency,
upgrades what can be upgraded safely, and holds components back with pins
tracked as technical debt (epic #155's doctrine). **What shipped:** all of it.
`vrg-host status` / `upgrade` / `pin` / `unpin` are released in vergil-tooling,
with a curated manifest and a gcloud NumPy auxiliary check (absorbing #166).
The probe parsers were verified against real output captured from the
maintainer's Mac, and macOS updates are reported at every version level. The
first real `vrg-host status` run reported 7 of 9 host dependencies behind.

### Work delivered

- **Span:** epic opened 2026-07-18 (spec + plan merged the same day);
  implementation ran 2026-10-15 → 2026-10-16.
- **Repos touched:** `vergil-tooling` (engine, manifest, site docs),
  `vergil-vm` (host-setup docs), `.github` (spec, plan, retrospective).
- **PRs (12, all merged):**

| PR | Task | What it did |
| --- | --- | --- |
| .github#170 | #168 | Spec + plan for the epic. |
| vergil-tooling#3243 | #2451 | Manifest schema + strict TOML loader (`host_deps`). |
| vergil-tooling#3244 | #2452 | Install-method handler registry (`host_methods`); single `_run` shell-out with a privilege-escalation guard. |
| vergil-tooling#3246 | #2453 | `vrg-host status`: two-axis classification, severity roll-up, exit codes; pin-constraint grammar. |
| vergil-tooling#3247 | #2456 | `pin` / `unpin`; host-local pins file overlaid on the manifest; re-evaluation surfacing. |
| vergil-tooling#3248 | #2454 | Auxiliary-check remediation (`status --fix`) + gcloud NumPy check. |
| vergil-tooling#3249 | #2455 | `vrg-host upgrade`: auto/manual, pin-respecting, no escalation. |
| vergil-tooling#3250 | #2457 | Curated manifest; one version comparator (`host_versions`). |
| vergil-tooling#3262 | #2450 | Docs sweep: `vrg-host` reference page and site-wide pointers. |
| vergil-vm#319 | vergil-vm#318 | VM host-setup docs point at `vrg-host` for Lima, gcloud and OpenTofu upkeep. |
| vergil-tooling#3271 | #3261 | Live macOS captures folded into the fixtures; uv and Docker install methods corrected. |
| vergil-tooling#3277 | #3269 | macOS updates reported at every level (patch, minor, major). |

- **Tasks:** 14 children closed plus this retrospective. Two pre-existing
  issues were closed as superseded: vergil-tooling#896 (the original
  `vrg-audit-host` request) and .github#166 (the gcloud/NumPy incident). Three
  tasks were added mid-flight (#3261, #3269, vergil-vm#318). The follow-on
  brainstorm (#169) was moved to the ad-hoc epic rather than closed (§5).
- **Releases:** vergil-tooling **v2.1.242** and **v2.1.243** (attributed by
  merge and release timestamps: #3277 merged after v2.1.242 and before
  v2.1.243).

## §1 How the plan evolved

The plan had no "Evolution during execution" log; this narrative is
reconstructed from the execution record (issue comments, PR notes, and the
session that drove the epic).

The seven-task structure held. Every planned task shipped in the planned
order, and the T1 → T2 → T3 chain ran strictly in sequence. What moved was
mostly **design detail the plan's sample code had left open**, and **contact
with the real host**.

- **Pins moved out of the package (decided mid-flight).** The plan had
  `vrg-host pin` writing into the packaged manifest. Before T6 started, we saw
  that this would mutate the installed package, and a reinstall would silently
  drop every pin. The maintainer decided pins are user configuration, stored in
  `~/.config/vergil/host-pins.toml` alongside `identities.toml` and overlaid on
  the manifest at load time. The decision was recorded on #2456 before
  implementation.
- **The pin-constraint grammar was defined, not inherited.** T3 shipped a
  stand-in that compared an active pin to its constraint *string*, so nearly
  every pinned dependency would have shown WARN. Rather than defer the fix to
  T7, the maintainer had the frozen branch reopened (`unfreeze`) and fixed
  before submission. The grammar became a dotted prefix with an optional
  trailing `x` (`1.x`, `2.40.x`, `2.40.0`), and anything else is refused. The
  plan's own sample (`<=1.x`) became invalid as a result.
- **"Differs means behind" was removed rather than kept as a fallback.** T7
  replaced the stand-in with a single comparator. Two versions that cannot be
  ordered are now a probe error, not a guess.
- **Strictness above the plan throughout.** Every task tightened the plan's
  sketches under the no-silent-failures rule:
  - unknown manifest keys are refused;
  - unparseable output raises `ProbeError`;
  - failed upgrades raise errors;
  - `sudo`/`doas`/`su`/`pkexec` are refused;
  - self-updating casks refuse a brew upgrade.
- **Parallelism cost a correctness bug and three rebases.** T4, T5 and T6
  depended only on merged work, so they ran in parallel, but all three edited
  `vrg_host.py`. Beyond the merge conflicts, T5 (upgrade) and T6 (the
  pins-file overlay) were built blind to each other. As built, T5 would have
  upgraded straight past any pin set with `vrg-host pin`. This was caught when
  T6 reported, before either merged. The three were then submitted one at a
  time, each rebased, with T5 switched to the overlay loader. T7 later failed
  CI on a T5 test that enumerated the manifest, and was fixed on the open PR.
- **The real host overturned three assumptions.** T7 was built on a Linux VM,
  so its fixtures were constructed from documented formats. The session was
  temporarily moved onto the maintainer's Mac (running with human identity),
  restricted to read-only capture. That capture found:
  - **uv was Homebrew-installed**, so `uv self update` would have failed;
  - **Docker was being tracked as the CLI**, not Docker Desktop, so the app's
    drift (4.71 → 4.94) was invisible;
  - **macOS offered a minor and a major update at once**, which a single
    "latest" could not express.

  The maintainer also used the moment to make install methods deliberate
  rather than accidental: Homebrew by default, with self-updating tools owned
  by their own updater. These produced #3261 and #3269.

## §2 Lessons learned

- **Tell implementers to build on merged code, not the plan's sketches.** Plan
  code samples rot as soon as the first task lands with stricter or renamed
  interfaces (`UpgradeUnsupportedError`, the 7-field `HostPin`). Each
  dispatched task was told "read the merged modules first; they deviate", and
  the T2–T7 chain integrated cleanly. Keep doing this.
- **"Consumes only merged interfaces" is not the same as "safe to
  parallelise".** The plan's dependency analysis looked only at interfaces
  consumed. Tasks that *edit the same file* or *change a shared loading path*
  are coupled even when their interfaces are not. Next time:
  - check the files each task touches, not only the interfaces it consumes;
  - when two parallel tasks share a seam (here, the manifest loader), write
    the integration contract into both prompts.
- **Constructed fixtures validate the parser, not the assumptions.** All 17
  constructed fixtures parsed correctly, and three manifest assumptions were
  still wrong. Fixtures built from documentation cannot reveal *how a tool was
  installed* or *what the host actually offers*. A live capture belongs in the
  plan for any epic that models a real environment.
- **The capture-only out-of-sandbox protocol worked.** Moving the session out
  of the VM grants full human access. Agreeing the boundary up front kept the
  stint short and left nothing on GitHub under the human identity. The rules
  were: read-only commands only, files written only into the worktree, no
  commits or pushes or GitHub calls, and all writes deferred until back inside
  the VM as `user`. Reuse it.
- **Process correction.** The driving agent wrongly told the human that a
  submitted PR's branch was still frozen, and handed back a `pr-watch` command
  instead of fixing a one-line test. Submitting lifts the freeze, so fixing a
  red submitted PR is the agent's job.

## §3 Compromises & tradeoffs

- **Three fixtures are still constructed:** `softwareupdate` with nothing
  pending, `softwareupdate` with only non-OS updates, and a
  `uv tool list --outdated` that actually lists a tool. The host could not
  produce these on capture day. They are labelled constructed in the fixtures
  README.
- **lima, opentofu and git-cliff are not in the end-to-end capture test**
  (they were not captured). Their probes are the shared `brew-formula` path,
  which is covered.
- **uv's "latest" is Homebrew's stable**, which can trail upstream by a patch
  (0.13.0 vs 0.13.1 on capture day). We accepted that in exchange for one
  install method.
- **Docker Desktop and macOS are report-only.** Both update through their own
  updaters, which may need privileged helpers, and `vrg-host` never escalates.
- **Re-evaluation-due pins rate WARN, not OK.** This deliberately departs from
  the spec's "pinned = OK, tracked debt", so that `vrg-host status` exits
  non-zero once a newer release has passed the pin's inducing release. A
  pinned macOS with an update its pin allows is still listed but does not
  raise severity. That was flagged as a possible later decision.
- **`under-evaluation` pins can only be set by hand-editing the pins file.**
  `vrg-host pin` creates only active pins.
- **Nothing runs `vrg-host` on a schedule.** Drift is visible only when someone
  runs it. That gap is exactly the parked follow-on (§5).
- **Azure CLI (`az`) is deliberately excluded.** Its provider is a placeholder;
  gcloud is the cloud path in practice.

## §4 New problems & opportunities

- **Ecosystem-wide scheduling of recurring work.** This is much larger than
  host reminders: per-repo architecture and refactor reviews, tracking major
  releases in each language ecosystem, quarterly capacity and strategic
  reviews, at ecosystem, org and repo scope, wired into the epic framework.
  → **Logged, not yet acted on:** captured on .github#169, now under the
  ad-hoc epic (#99).
- **Overlap with epic #375's deferred review cadence.** The same scheduling
  need. → Linked from .github#169 so that one design serves both.
- **Docker Desktop → a headless open-source runtime** (Colima, Podman, or
  Docker Engine over Lima). If adopted, `vrg-host` could upgrade Docker
  automatically. → **Logged, not yet acted on:** carried in .github#169, to
  become its own epic.
- **The first real `vrg-host status` showed 7 of 9 dependencies behind**,
  including a 23-minor-version Docker Desktop gap. The tool's value was
  demonstrated immediately. The maintainer is pushing the host to the leading
  edge with `vrg-host upgrade --all`. The outcome of that first upgrade run is
  **unknown at time of writing**.
- **Plan sample-code defects** (the 6-vs-7-argument `HostPin` in T6's sample
  test, and the `<=1.x` constraint) were caught by implementers, not by
  pushback or alignment. → Not acted on. This is a hint that pushback could
  check sample code against the plan's own declared interfaces.

## §5 What's next

- **.github#169**: the parked brainstorm. Ecosystem-wide planning and
  scheduling of recurring work, and the Docker-runtime migration. Both are
  intended to become their own epics. The next step when it is picked up is
  `epic-create`, starting from how the maintainer plans day to day.

## Appendix A — Operational notes

**Moving the session out of the sandbox to capture live host output**
(2026-10-16):

1. Before the move, still inside the VM as `user`:
   - surface every pending GitHub write (here, a verification note on #3261);
   - create the task's worktree, so that the host session only writes files.
2. On the host, first confirm identity and platform: `vrg-whoami --explain`
   resolved `human` on `physical-host`. Then run only read-only probes, plus
   `vrg-host status` from the worktree code via
   `uv run --isolated --no-project --with <worktree> vrg-host status`, which
   writes only to uv's cache.
3. Stage the captures in a new subfolder of the worktree, with a notes file
   mapping each file to its exact command. Strip personal paths. No commits.
4. Back inside the VM, as `user`: fold the captures in, fix code, validate,
   commit, push, `report-ready`.
