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
  paad's output. That belongs to C, D and later epics.
- Repos owned by a user account rather than an org. An org is a requirement.
- Enforcing the "about a day" lifetime in tooling.
- Strong protection against an agent editing or closing advisories. The
  user's stated risk model is that these are notes, and agents may manage
  them.

**Anti-goals** (failures even if everything technically works)
- Security details in any issue title or body, comment, PR body, commit
  message, report-ready note, or relay ref, in any repo, public or private.
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
  Every org's `.github` repo is public, so advisories are available there.
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
  - severity mapped from paad: Critical → critical, Important → high,
    Suggestion → medium. A finding with no paad severity is given one by
    the caller;
  - never published, and no CVE requested.
- **Issue**, created in the intake queue the same way `vrg-intake-create`
  creates one (the `.github` intake for a public repo; the repo's own intake
  for a private one):
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

### 4.2 The tool: `vrg-security-record`

A new vergil-tooling CLI (working name). It calls the advisories API
directly with the identity's App token. It does **not** go through
`vrg-gh api`, which stays denied to the user agent. It resolves the org's
`.github` repo itself; the caller names only the affected repo.

- `create --repo <o>/<r> --title <neutral> --body-file <neutral> --details-file
  <private> --severity <s>` creates the pair and links both ways. The order
  is: create the advisory, create the issue naming the GHSA ID, then patch
  the advisory with the issue reference. A failure at any step is reported
  loudly with what was and wasn't created (the GHSA ID and/or issue number),
  and the command exits non-zero. A re-run can complete a half-made pair
  (the builder decides how).
- `create … --issue <ref>` (attach mode, *my call*) attaches a new advisory
  to an **existing** issue instead of creating one: it adds the
  `Security advisory:` line and the label to that issue. The past-issues
  audit (4.6) uses this, so one problem never ends up with two issues.
- `list [--repo <affected>] [--state …]` lists pairs and their status, for
  the human and the sweep.
- `show <GHSA-id | issue ref>` shows one pair. Details are printed only to
  the terminal, never anywhere else.

### 4.3 State sync (the sweep)

The existing **scheduled sweep** (`vrg-epic-audit --close` under the org ops
App) gains one pass over the org's `.github` advisories:
- **Two-way state sync** (*my call*): the advisory's state follows its
  issue's. A closed issue (fixed, or dropped as not planned) means the
  advisory is closed; an open issue means it is a draft. So a reopened issue
  reopens its advisory.
- **Consistency check** (*my call*): it **reports** broken pairs: a
  `security` issue with no or a wrong advisory reference, or a draft
  advisory with no or a bad `Tracking issue:` line. It reports and does not
  repair, so the failure is loud and a human decides what to do.

The ops App therefore needs the advisories write permission.

### 4.4 Intake routing keeps the link (*my call*)

When `vrg-intake-route` moves a `security` item by recreating it, or splits
it, it rewrites the advisory's `Tracking issue:` line to name the issue that
carries the work (for a split: the first resulting task, with the rest
listed in the advisory). A plain transfer needs no change, because GitHub
redirects a transferred issue. This runs as the user agent, which has the
permission.

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
- add the `security` label;
- if a secret was exposed, report it for rotation (a human action);
- if the problem is still unfixed, attach an advisory to the existing issue
  (`create --issue`);
- report the total count.

The default is **audit and label only**: the text is already public and
probably indexed, and GitHub keeps edit history, so scrubbing gains little.
If the count is small, the human may then decide to scrub as well (rewrite
the issue neutrally and move the details into the advisory). That decision
is taken after the count, not now. The audit's working list must not be
posted anywhere public beyond what the issues already say.

### 4.7 Label

Add `security` to `data/labels.json` ("Has a private security record in the
org's .github advisories"). Existing tooling then ensures it wherever it is
applied.

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
  - the pre-flip checklist (4.5).
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

- Exactly how a half-made pair is completed on re-run.
- The exact `list` output format.
- Where the sweep pass lives in code (alongside the orphan pass is the
  obvious place).
- How the audit in 4.6 finds candidates (a search, then an agent's
  judgment), and how its working list is kept private.

## 7. Open questions for the human

None. The audit App's permission is deferred (4.8).
