# Security records: private advisories paired with neutral issues

- **Epic:** vergil-project/.github#517
- **Planning issue:** vergil-project/.github#505
- **Umbrella:** vergil-project/.github#500 (child A). Children C and D depend
  on this epic.
- **Status:** approved design (2026-10-26), in design review
- **Review history:** the comments on the planning issue,
  vergil-project/.github#505
- **Brainstorm source:** superpowers brainstorming session, 2026-10-26

## 1. Intent

paad 2.0 never writes security findings anywhere public. It keeps them in a
gitignored `paad/security/` folder, and its authors call that folder
"scratch, not state": it lives on one machine, and `git clean -x` or
deleting the worktree loses it. Until now we have put security problems in
ordinary public issues and moved on. This epic adopts paad's standard
rather than working around it: security details get a private, durable home,
and the issue carries only a neutral pointer.

The private home is a GitHub **draft repository security advisory**, kept
centrally in the org's `.github` repo. We use advisories as secure
note-taking, not as a disclosure process; nothing is published by default.
Advisories **complement** issues. They don't replace them: issues still
drive the work.

Security is a top priority. A security record is meant to be short-lived:
once one is captured, fixing it comes first, with a target of about a day.
This is policy, stated in the docs; the tooling does not enforce it.

This epic builds the mechanism only. Wiring paad's output and the skills to
use it belongs to the follow-on children C and D (and any later epic) that
depend on this one.

## 2. Goals, non-goals, anti-goals

**Goals**
- Any security finding can be recorded with one command, which produces a
  private advisory holding the details and an intake issue holding only a
  neutral title and a pointer. Each references the other.
- The same rule for every repo, public or private: issues are always
  neutral, and details always live in the advisory.
- Advisories follow their issue's state, and broken pairs are reported.
- Past public issues that disclosed security problems are found and labelled,
  and anything still unfixed gets a proper record.

**Non-goals**
- Publishing advisories, requesting CVEs, or any coordinated-disclosure
  process. These are possible later, and nothing here prevents them.
- Wiring the tool into `issue-implement`, `epic-create`, `intake-capture` or
  paad's output, meaning *when findings are recorded*. That belongs to C, D
  and later epics. The one exception is the record's own lifecycle in
  planning (4.4b), which this epic delivers.
- Repos owned by a user account rather than an org. An org is a requirement.
- Orgs whose `.github` repo is private. Every org in use is public (private
  repos exist inside public orgs, for client work), and supporting fully
  private orgs would be an architecture extension. `create` refuses such
  an org loudly (4.2).
- Enforcing the "about a day" lifetime in tooling.
- Strong protection against an agent editing or closing advisories. The
  user's stated risk model is that these are notes, and agents may manage
  them.

**Anti-goals** (failures even if everything technically works)
- Security details in any issue title or body, comment, PR body, commit
  message, report-ready note, relay ref, or **Actions log** (the sweep runs
  in the public `.github` repo, so its logs are public), in any repo, public
  or private.
- An advisory without an issue, or an issue whose advisory reference is
  missing or wrong. Half-created pairs must never pass silently.
- Opening `vrg-gh api` to the user agent as a general write path.

## 3. Verified facts this design rests on

From docs.github.com (read 2026-10-26):
- REST reference:
  https://docs.github.com/en/rest/security-advisories/repository-advisories
- Concepts:
  https://docs.github.com/en/code-security/concepts/vulnerability-reporting-and-management/repository-security-advisories
- Who can see and do what:
  https://docs.github.com/en/code-security/reference/permissions/repository-security-advisory
- Token permissions:
  https://docs.github.com/en/rest/authentication/permissions-required-for-github-apps

What they say:
- Repository security advisories are documented for **public** repos only.
  This design requires the org's `.github` repo to be public (§2
  non-goals), so advisories are available there.
- A draft is visible only to repo admins, owners, security managers, and
  people or teams invited to that advisory. Nothing becomes public without an
  explicit publish. Drafts can be closed and reopened; there is no documented
  delete.
- REST: `POST/GET /repos/{o}/{r}/security-advisories` and `GET/PATCH
  …/{ghsa_id}`. Create requires `summary` (≤1024 chars), `description`
  (≤65535) and `vulnerabilities[]` with a `package.ecosystem`; `other` is an
  allowed ecosystem, and `package.name` is a free string or null. Severity is
  optional. A CVE is only ever requested by a separate call.
- Permission: "Repository security advisories". Read covers list and get;
  write covers create and update. The docs list installation tokens as
  supported. They also say the caller must be an admin or security manager,
  and how that applies to an App's bot identity is undocumented.

From the code:
- Neither the user nor the audit App has an advisories permission
  (`docs/site/docs/guides/account-setup.md` declares App permissions).
- `vrg-gh api` is denied for the user agent and is GET-only for audit
  (`bin/vrg_gh.py`).
- The scheduled sweep (`ops-epic-sweep`, called from `.github`'s
  `epic-sweep.yml`) runs as a separate **org ops App**
  (`APP_CLIENT_ID`/`APP_PRIVATE_KEY`).
- There is no `security` label. The fleet label set is
  `src/vergil_tooling/data/labels.json`, and `lib/labels.ensure_labels`
  ensures a label wherever it is applied.

## 4. Design

### 4.1 The pair

Every security finding produces **one pair**: one advisory and one issue.
paad's backlog is per finding too, so one pair per finding matches it.

- **Advisory**, always in **`<org>/.github`**, whichever repo the finding
  is about:
  - `summary`: a short description; this may be specific, because it is
    private;
  - `description`: the full finding (paad's entry, or the agent's account),
    followed by a line `Tracking issue: <owner>/<repo>#<N>`;
  - `vulnerabilities: [{package: {ecosystem: "other", name: "<affected
    repo>"}}]`, so the affected repo is a structured, queryable field
    (*my call*);
  - a severity in GitHub's terms (critical, high, medium, low). The docs
    carry a mapping from each paad scale:

    | paad scale (skills) | critical | high | medium | low |
    |---|---|---|---|---|
    | Critical / Important / Suggestion (agentic-review) | Critical | Important | Suggestion | n/a |
    | Critical / High / Medium / Low (agentic-owasp) | Critical | High | Medium | Low |
    | High / Medium / Low (pushback, agentic-architecture) | n/a | High | Medium | Low |

    A finding with no paad severity is given one by the caller;
  - never published, and no CVE requested.
- **Issue**, created in the intake queue the same way `vrg-intake-create`
  creates one. Its home is always `epics.queue_home(<affected repo>)`: the
  `.github` intake for a public repo, the repo's own intake for a private
  one. It is never created in the public code repo itself:
  - a **neutral** title and body, supplied by the caller, **even in a
    private repo**; the tool can't judge neutrality, so it is the caller's
    duty, and the docs say so;
  - the label `security`;
  - a line `Security advisory: <org>/.github <GHSA-id>` (the ID reveals
    only that a record exists).

Intake review then routes the issue like any other item. The issue drives
the work; the advisory holds the details.

**Trade-off, accepted:** advisories no longer sit next to the code they are
about, and every admin or security manager of `.github` can see every draft,
including those about a private (for example, client) repo. Today those are
the same people. The package-name field keeps the affected repo queryable.

**Trade-off, accepted:** for a public repo, the issue is public. Its
`security` label and advisory line, together with a title that names the
affected area so it can be routed, tell anyone that an unfixed security
problem exists in that repo. The issue never says what the problem is, and
the record is meant to live about a day (§1). This is a deliberate, small
disclosure, accepted in exchange for a visible, queryable label.

### 4.2 The tool: `vrg-security-record`

A new vergil-tooling CLI (working name). It calls the advisories API
directly with the identity's App token, or, in human mode (where there is
no App token), with the ambient `gh` credential; both are supported. It
does **not** go through `vrg-gh api`, which stays denied to the user agent.
It resolves the org's `.github` repo itself, and refuses loudly if that
repo is not public; `--repo` always names the **affected** repo, never where
anything is created.

**Output rule.** Advisory `summary` and `description` text is printed only by
`show`, and by `list --details`, both interactive commands. Everything else
(the sweep, the consistency report, `list` without `--details`, error
messages) names only GHSA IDs, issue refs, labels and states. `--details` is
refused when `VRG_EPIC_SWEEP` is set.

- `create --repo <o>/<r> --title <neutral> --body-file <neutral> --details-file
  <private> --severity <critical|high|medium|low>` creates the pair and
  links both ways. The order is: create the advisory, create the issue
  naming the GHSA ID, then patch the advisory with the issue reference. A
  failure at any step is reported loudly with what was and wasn't created
  (the GHSA ID and/or issue number), and the command exits non-zero.
- **Resume.** Drafts can't be deleted, so a re-run must never create a
  second advisory for the same finding.
  - When `create` fails after the advisory exists, its error prints the
    exact command to finish the pair: the same arguments plus
    `--resume <GHSA-id>`, and `--issue <ref>` too once the issue exists.
    With `--resume`, `create` skips creating the advisory and completes
    whatever steps are missing.
  - When resume must find the issue itself, it looks among the children of
    the queue home's intake epic for the `Security advisory:` line, never
    through search, whose index can lag.
  - **Duplicate guard.** Before creating an advisory, `create` looks for an
    existing draft with the same summary and affected repo, and refuses if
    it finds one, naming its GHSA ID. This covers a create whose response
    was lost, leaving no ID to resume from.
- `create … --issue <ref>` (attach mode, *my call*) attaches a new advisory
  to an **existing** issue instead of creating one: it adds the
  `Security advisory:` line and the label to that issue. The past-issues
  audit (4.6) uses this, so one problem never ends up with two issues.
- `retarget <GHSA-id> --to <ref> [--to <ref> …]` moves an advisory's
  tracking to new issues. Each target gets the `security` label and the
  `Security advisory:` line, then the advisory's `Tracking issue:` lines are
  replaced. Only after both succeed does the caller close or move the old
  issue; any failure stops loudly. It is the one way a link moves: intake
  routing (4.4) and planning (4.4b) both use it.
- `list [--repo <affected>] [--state …] [--details]` lists pairs and their
  status, for the human and the sweep (output rule above).
- `show <GHSA-id | issue ref>` shows one pair, with details, to the terminal
  only.

### 4.3 State sync (the sweep)

The existing **scheduled sweep** (`vrg-epic-audit --close` under the org ops
App) gains one pass over the org's `.github` advisories:
- **Issue-driven sync** (*my call*): the advisory's state follows its
  issues'. The advisory is closed when **every** issue it tracks is closed
  (fixed, or dropped as not planned), and is a draft while any of them is
  open. So a reopened issue reopens its advisory. The issue always wins: a
  manual change to an advisory's state is overwritten on the next run, and
  the docs say to manage a record through its issue.
- **Classifying advisories.** An advisory is **paired** if any `security`
  issue's `Security advisory:` line names it. An unpaired advisory that came
  from private vulnerability reporting (enabled on `.github`, see
  `SECURITY.md`; GitHub marks it with `triage` state or a `submission`
  field) is an **unpaired external report**: listed as such, never treated
  as broken, and never changed by the sweep. Any other unpaired advisory is
  a broken pair.
- **Consistency check** (*my call*): it **reports** broken pairs: a
  `security` issue with no or a wrong advisory reference; an advisory whose
  `Tracking issue:` lines are missing, or name an issue that doesn't exist
  or doesn't point back; or an unpaired advisory that is not an external
  report. It reports and does not repair, so the failure is loud and a human
  decides what to do. Its output follows the output rule (4.2): IDs, refs
  and states only.

The ops App therefore needs the advisories write permission.

### 4.4 Intake routing keeps the link (*my call*)

Whenever `vrg-intake-route` moves a `security` item to a new issue number
(by transfer or by recreating it), it retargets the advisory to the new
issue (`retarget`, 4.2). The checker never relies on GitHub's transfer
redirect.

When it **splits** a `security` item, it retargets the advisory to every
resulting task **before** closing the original, and stops loudly on any
failure, leaving the original open. The advisory then closes only when all
of the tasks are closed (4.3).

This runs as the user agent, which has the permission.

### 4.4b Security items in planning

A security item may go through planning like any other item. Planning
discusses the problem in general terms; the vulnerability's details stay in
the advisory. The planning issue tracks the advisory until the plan exists.

- **Planning hands the advisory to the fix.** When `epic-create` files the
  implementation tasks (its step 9), it retargets the advisory from the
  planning issue to the task or tasks that fix the problem. From then on,
  the PRs that close those tasks are what close the advisory (4.3). This is
  a new instruction in the `epic-create` skill, delivered by this epic.
- **Guard.** `vrg-queue planning` never shows a planning issue as
  `ready-to-close` while an advisory still tracks it, and says why. So the
  planning issue can't be closed (step 10) before the handoff, which would
  close the advisory with the fix unwritten.

### 4.5 A private repo going public

Because issues are neutral everywhere, a private repo holds no security
details in its issues, and nothing needs converting. What tooling can't
cover remains. The docs carry a short pre-flip checklist for the human:
- git history and commit messages;
- PR bodies and review comments;
- wikis and Actions logs;
- any issue written before this policy existed (the 4.6 audit, run on that
  repo).

One known private repo is a candidate for going public, so this checklist is
needed now.

### 4.6 Past public issues

A one-time audit of open and closed issues in every public vergil-project
repo, looking for ones that disclosed security problems. For each hit:
- if the problem is still unfixed, attach an advisory to the existing issue
  (`create --issue`), which adds the `security` label;
- otherwise (already fixed), add the `security-disclosed` label, which
  marks a past public disclosure with no record;
- if a secret was exposed, report it for rotation (a human action);
- report the total count.

The default is **audit and label only**: the text is already public and
probably indexed, and GitHub keeps edit history, so scrubbing gains little.
If the count is small, the human may then decide to scrub as well (rewrite
the issue neutrally and move the details into the advisory). That decision
is taken after the count, not now. The audit's working list must not be
posted anywhere public beyond what the issues already say.

### 4.7 Labels

Add two labels to `data/labels.json`. Existing tooling then ensures each
wherever it is applied.
- `security`: "Has a private security record in the org's .github
  advisories". The consistency check (4.3) applies only to this label.
- `security-disclosed`: "Past issue that disclosed a security problem
  publicly; fixed, no private record". The check ignores it.

### 4.8 Permissions (human actions)

Add "Repository security advisories: Read and write" to:
- every **user** App (`<user>-vergil-user`);
- the **org ops App** that runs the sweep;
- **not** the audit App. It is not in use today (agent work runs
  interactively, as the user identity), so the decision is deferred until
  the audit identity is revived. Reviving it should revisit this.

Update `account-setup.md` to match. Changing an App's permissions requires
each installation to accept the change, which is a human step.

### 4.9 Probe before building

Before the tool is built, a validation task checks the undocumented points
directly in `vergil-project/.github`, leaving one closed, clearly titled
test draft (drafts can't be deleted):
- a **user** App installation token can create, read, update and close a
  draft;
- `ecosystem: other` with a repo name as the package name is accepted;
- under the **ops App** token: list drafts, close one and reopen it, which
  is exactly what the sweep needs.

If either App can't do its part, stop and re-plan with the human (for
example, making `create` human-run). Don't build around it silently.

### 4.10 Documentation

- A security-records page in the vergil-tooling site docs:
  - the policy, and the about-a-day lifetime target;
  - what counts as a security finding (paad's definition: "if reading it
    would help an attacker…; on the edge, treat it as security");
  - neutral-wording rules;
  - the severity mapping table (4.1);
  - managing a record through its issue, never by editing the advisory's
    state (4.3);
  - the pre-flip checklist (4.5);
  - the planning handoff (4.4b).
- `vergil-claude-plugin` `epic-create` skill: the step 9 handoff and the
  step 10 guard (4.4b).
- `account-setup.md`: the new permission.
- `.github/SECURITY.md`: a mention of internal records, if it fits.

## 5. Constraints

- Public repos: issue comments, PR bodies, report-ready notes and the relay
  ref are public.
- Agents use `vrg-git`/`vrg-gh`. The new tool is a sanctioned path, like
  `vrg-intake-create`.
- No silent failures, especially for half-created pairs.
- Changes to App permissions and repo visibility are human actions.
- This epic's own records (the planning issue, PRs) must not contain any
  real security finding's details.

## 6. Left to the builder

- The exact `list` output format.
- Where the sweep pass lives in code (alongside the orphan pass is the
  obvious place).
- How the audit in 4.6 finds candidates (a search, then an agent's
  judgment), and how its working list is kept private.

## 7. Open questions for the human

None. The audit App's permission is deferred (4.8).
