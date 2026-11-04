# Authenticated human replies on task issues

- **Epic:** vergil-project/.github#889
- **Planning issue:** vergil-project/.github#879 (review history is in its
  comments)
- **Status:** draft
- **Repos:** vergil-tooling (the two tools, their docs and the `build/tmp/`
  convention), `.github` (this spec and plan)

## 1. Problem

When a code review of a task's change needs the human's decision, the
decision reaches the agent that acts on it as text. Under `epic-implement`,
the dispatching agent copies the human's rulings into the dispatched run's
prompt, and `issue-implement` writes them onto the task issue as `Human
ruling:` lines in a comment from its own agent login. Nothing proves the
human said any of it.

This epic builds the tools for an authenticated channel. The agent posts a
request on the task issue. The human posts the reply from their own GitHub
account, in their own shell. The agent reads the reply from the issue and
checks who wrote it. A caller never relays the decision: under
`epic-implement` it says only "human response available", and the run reads
the issue itself.

The task issue is the unit of record. A task is closed by one PR, and its
comments record how that PR's change was reviewed and what the human
decided.

## 2. Scope and sequence

Three epics run in this order:

1. Removing agentic-review from the workflow (planned separately).
2. This epic: the tools, their documentation, their release and an end-to-end
   validation.
3. The merge review that replaces agentic-review. It defines the review loop
   inside `issue-implement` and adopts these tools.

The epic also documents the `build/tmp/` scratch convention (§11). It came up
in this design and is carried here as a general convention, though these
tools do not use it.

This epic changes no skill. The merge-review epic makes every skill change,
including how `epic-implement` resumes a run and how `issue-implement` orders
its decisions (§4).

## 3. Identities and assumptions

- **Trusted authors.** The tools read identity only from the existing
  trusted-authors configuration (`vrg-trusted-authors`,
  `lib/trusted_authors.py`): its `human` logins and its `agents` logins. A
  leading `app/` is stripped before comparing, as that module says. Comments
  by any other login are ignored by every check in this spec.
- **One human.** The configuration's `human` entry is a list, but vergil is
  built for a single human. Both tools require it to hold exactly one login,
  **the human login**, and refuse with a clear message when it holds none or
  several. The configuration format does not change.
- **The agent login.** The login the agent tool expects to post as is derived
  by the trusted-authors convention: `<the human login>-vergil-<mode>`, where
  `<mode>` is the session's identity mode. The agent tool runs only in the
  `user` mode, so the login is `<the human login>-vergil-user`; it refuses in
  `audit` mode, whose App can only read issues. The derived login must be on
  the `agents` list, or the tool refuses before posting. No
  new configuration. The tool never needs to ask GitHub its own login:
  `viewer { login }` is not reachable through `vrg-gh` in the agent identity
  (probe vergil-project/.github#864).
- **Comment authors.** Probe #864 found that comment authors appear exactly as
  `vrg-trusted-authors` lists them.
- **Agent sessions have only App credentials.** Every agent session runs in a
  `vrg-vm` VM whose only GitHub credentials are the App's. `vrg-gh`'s fallback
  to ambient gh auth finds none there and runs unauthenticated, which cannot
  post. So an agent can never post as the human. This is a stated assumption,
  not something the tools check (GHSA-9q3r-h2xj-rgq3).
- **Identity is never asserted in text.** A comment's author is the only
  source of who wrote it. A hidden marker may let the tools recognise a
  request or a response (§7), but never says who wrote it.

## 4. One round on a task issue

1. The review's findings, posted by an agent login.
2. **The request:** one comment listing every finding that needs the human,
   each with the agent's recommendation. Posted by the agent tool.
3. **The reply:** one plain-text comment covering everything in the request.
   Posted by the human tool, as the human.
4. **The response:** what the agent did with each item, and how it read the
   reply. Posted by the agent tool.

One request per round, and one reply per request. A reply has no grammar:
the skill that reads it interprets it, and states its reading in the
response. An unclear reply is answered by a new request.

Decisions that are not about review (approving a follow-up issue or a new
operational task, preflight stops, problems the agent cannot resolve) stay
outside this channel. The skill handles them first, before posting the
request, so the review exchange sits in one block on the issue. That is the
merge-review epic's to build.

In practice the human and the agent talk the items through in the terminal
first. That conversation is not recorded. Once the human decides, the agent
drafts the reply and hands the human a command with it already filled in;
the human reads it and runs it.

## 5. The checks

### 5.1 Accepting a reply

The agent tool accepts a reply only when:

- the request it answers was posted by the derived agent login (§3);
- its author is the human login;
- it is the first trusted comment after the request;
- neither the request nor the reply has been edited.

### 5.2 Turn order

The issue is single-threaded. After a request, the next trusted comment must
be the reply. After the reply, the next trusted comment must be the response.
Any other trusted comment in either gap is a turn violation and fatal: an
agent comment between the request and the reply, or a second human comment
before the response. To correct an answer, the human waits for the response,
and the agent asks again in a new request.

### 5.3 Write-once

Agents never edit a comment; they post a new one. The human never edits one
either. The tools enforce this on the request and the reply: an edited
request or reply is fatal, whoever wrote it.

### 5.4 Deletions

The check covers only the exchange in hand, never earlier rounds: `read`
fails when the request it was given is gone, and `respond` fails when that
request or its reply is gone. The response records both comment IDs, as the
record of what it acted on; nothing re-checks earlier responses. Deleting an
untrusted comment (marketing noise, for example) is allowed and ignored.

### 5.5 The audit's ceiling

Other comments in the exchange (the findings, earlier responses) are not
chained or checked, and anyone with write access can delete evidence,
up to a whole issue. GitHub's docs say only a comment's author or someone
with write access can edit or delete it, and that an edit stays visible as
edited; there is no way to make a comment uneditable. The record is as
trustworthy as the people with write access. The tools guarantee only that
the decision the agent acts on is the human's, unaltered.

## 6. Security items

A security finding's details never appear in a public record. Its decision
still goes through the request and the reply, by ID:

- The details reach the human only in the terminal, and reach a dispatched
  run only privately, as they do today (from the task's state directory),
  never from the issue.
- **In a request,** a security item is marked by an explicit flag, not
  recognised by the shape of its ID, and carries only its ID and a fixed
  text: `security finding, details given privately`. The agent tool refuses a
  request whose flagged item carries anything else (GHSA-r5f4-5586-mqfp).
- **In a response,** security items appear by ID only, with no detail, and nothing from the review's result is copied into a response
  word for word. The agent tool refuses a flagged item in a response that
  carries anything but its ID and outcome (§7) (GHSA-2fjg-qxrj-fc2v,
  GHSA-8459-fr73-66rm).
- **The reply** is the human's text, which the tools cannot screen. The agent
  drafts it by the same rule, security items by ID only, and the human reads
  it before running the command.

## 7. The agent tool: `vrg-ruling` (vergil-tooling)

Run by an agent. Every subcommand refuses unless the derived agent login is
on the `agents` list (§3).

- **`request`** posts a request on an issue from an items file (each item: an
  ID, a security flag, and for non-security items the question and the
  recommendation). It prints the request's comment ID. It refuses while the
  issue has an open exchange (an earlier request without a reply, or a reply
  without a well-formed response after it), and names that request's ID so a
  resumed run can pick it up instead.
- **`draft`** runs once the human has decided. It builds the reply from an
  items file shaped like `respond`'s: each item an ID, a security flag, and
  for non-security items the decision in prose; a flagged item carries only
  its ID and a decision. The drafted text is printable ASCII plus newline and
  tab, and the tool refuses anything else. It pushes the reply to the
  request's draft ref (§9) and prints the paste-ready human command (§8) for
  that issue and request. It posts no comment.
- **`read`** reads the reply to a given request ID, running every check in
  §5.1 to §5.4. It has three outcomes, each with its own exit code: a valid
  reply (printed), no reply yet, and a fatal check (naming the check and the
  comment).
- **`respond`** posts the response to a request and its reply, from an items
  file shaped like `request`'s: each item an ID, a security flag, and for
  non-security items free prose saying what was done. A flagged item carries
  only its ID and its outcome (for example `fixed` or `filed`); the tool
  refuses a flagged item carrying anything else. It re-runs `read`'s checks
  first, refuses unless the reply is still the latest trusted comment, and
  records the request and reply comment IDs in the response. The one
  exception is a re-run that finds its own well-formed response to that
  request and reply already landed after the reply: it reports that
  response's ID, posts nothing and succeeds (§10).

The request and the response each carry a required hidden HTML marker that
gives the comment's kind (`request` or `response`) and, for a response, the
request and reply comment IDs. The tools recognise the comments only by it.
An agent comment after a reply that is not a well-formed response is a turn
violation (§5.2). The marker never carries identity, and the tools recognise
it only in comments by the derived agent login: a marker anywhere else,
including a comment by the human, is ignored.

## 8. The human tool: `vrg-ruling-reply` (vergil-tooling)

Run by the human in their own shell on the host, logged in as themselves,
the way `vrg-submit-pr` is run. Never through `!` in the agent session, whose
shell carries the agent's identity and bypasses the hooks.

- It refuses in an agent identity mode, as `vrg-submit-pr` does, and refuses
  unless the login its credentials post as is the human login. The agent
  tool's author check (§5.1) is the guarantee; this refusal catches the tool
  being run in the wrong shell.
- It takes the issue and the request's comment ID. By default the reply is the
  agent's draft, fetched from the request's draft ref (§9); with
  `--response-text "<text>"` it is the human's own text instead. A reply is
  printable ASCII plus newline and tab, whichever way it comes, because the
  agent may have written a suggested `--response-text` command too; only
  `--response-text` with `--allow-unicode` admits other printable text.
- Before posting, it reads the issue and refuses when the request is not the
  latest trusted comment (it is already answered, or the turn has moved on).
  It shares the agent tool's comment-reading code.
- **The human sees exactly what is posted.** The tool reads the reply once,
  prints exactly the bytes it will post, and asks for confirmation at an
  interactive prompt. It posts exactly those bytes, so a draft changed after
  it was read is never posted. It refuses to run without an interactive
  terminal.
- **Only plain text is posted.** The tool refuses a reply (a draft or
  `--response-text`) that contains a control character other than newline
  and tab, or a vergil marker (§7). It shows the reply with any non-printing
  character made visible, with its byte length and SHA-256 hash, so nothing on
  screen can hide part of what is posted.
- After posting any reply to a request, the draft or the human's own text, it
  deletes that request's draft ref. A reply declined at the prompt keeps the
  ref, so the command can be run again. `respond` also deletes any draft ref
  left for its request.

## 9. The draft relay

The draft reaches the human through GitHub, never a shared filesystem, so
the handoff works the same from a Lima VM and from a cloud VM, whose
projects live on its own volume. It reuses the relay mechanism of
`vrg-pr-workflow report-ready` (`GitHubTransport`, the
`refs/vergil/pr-workflow/<branch>` ref that `vrg-submit-pr` reads):

- `draft` writes the reply as a single-file commit built with git plumbing
  and force-pushes it to a reserved ref on the issue's repo,
  `refs/vergil/ruling-draft/<issue>/<request comment ID>`. It never touches a
  branch, the index or a working tree.
- The human tool fetches that ref, shows its content and posts it only on
  confirmation (§8). Because what is posted is exactly what the human saw,
  the agent pushing the ref again cannot change a posted reply.
- The ref is world-readable, like the report-ready relay. A draft holds only
  what the reply will make public anyway, with security items by ID only
  (§6).

## 10. Failures and recovery

**Every comment is posted exactly once.** `request`, `respond` and the human
tool each post with no retry (vergil-tooling's `post_json_once`, or an
equivalent), because a retried POST that had already landed would post a
duplicate, and a duplicate request or reply is a fatal turn violation. After
an uncertain failure (a timeout or a 5xx), the tool re-reads the issue: if
its comment landed, it reports the comment's ID; if not, it fails with an
error saying the post did not land, and posts nothing more. A POST is
bounded by a timeout, so a hung request takes the same path. A re-run of
`respond` after such a failure finds its landed response and succeeds
without posting (§7).

A fatal check stops the run with a non-zero exit and a message naming the
check and the comment. The human triages it. The recovery is to start over
on a new issue carrying the same context, patching the spec or plan if
needed. The broken issue is not repaired. Nothing is built for recovery
beyond detecting and stopping; it is expected to be rare.

## 11. Documentation

vergil-tooling's docs gain:

- the protocol: the round (§4), the checks (§5), security items (§6) and
  recovery (§10), written for the merge-review epic to cite;
- a reference page for each tool;
- the `build/tmp/` convention: within the existing exception for ignored
  scratch in a main clone's `build/` (the worktree convention's rule 3),
  everything under `build/tmp/` is scratch, and a file there can be deleted
  once nothing is using it. It is a general convention; these tools do not
  use it.

## 12. Deployment and validation

- **Deployment** (vergil-project/vergil-tooling#3918) releases the tools so
  they are usable before the merge-review epic starts. Cutting the release is
  the human's.
- **Validation** (vergil-project/vergil-tooling#3919) runs real rounds on a
  throwaway issue. A stub step posting a canned finding stands in for the
  review; it lives in the validation's procedure, never as a skill. It
  checks:
  - a full round: `request`, `draft`, the human running `vrg-ruling-reply`,
    `read` and `respond`, once from a Lima VM and once from a cloud VM;
  - that a draft ref pushed again after the human was shown it is not what
    gets posted;
  - that each of these is fatal: an edited request, an edited reply, a
    deleted request, a reply deleted after `read` (caught by `respond`), and
    a trusted comment out of turn. A reply deleted before `read` cannot be
    told from no reply, and gives the "no reply yet" outcome;
  - that deleting an untrusted comment is not;
  - that the logins GitHub reports for the App and the human on these
    comments match the checks, live rather than against stubs
    (GHSA-xffj-678w-ggjp).

## 13. Out of scope

- Every skill change, including `issue-implement`, `epic-implement` and the
  merge review itself (§2).
- Decisions not about review (§4).
- `issue-deploy` and `issue-validate`, whose comments stay a plain log of the
  work.
- `epic-create` and direct human-to-agent conversation, whose record is the
  spec and plan.
- Making every human ruling an authenticated comment
  (vergil-project/.github#892).
- A grammar for replies, and what any particular answer means.
- Chaining or checking comments other than the request and the reply (§5.5).
- Anything from `epics/816-agent-cost-metrics/` (epics
  vergil-project/.github#816 and #848, closed as not planned). This spec
  supersedes its ruling protocol (§7.2 there): where the two differ, this one
  governs. That epic is unlikely to be implemented; if it ever is, it is
  re-planned on top of this work, so nothing here accommodates it. In
  particular, its `Supersedes:` correction links do not exist here: a
  correction is a new request, and nothing invalidates an earlier comment
  (GHSA-w38c-m89h-5v9q).
