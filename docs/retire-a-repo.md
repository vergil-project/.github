# Retire a repo

How to retire a repo in an org that Vergil configures: close its open
issues, let the rollups those closes start finish, and hand the repo to the
human to archive. The agent runs steps 1 to 3; archiving is the human's.

## Why the wait matters

Each issue close starts an `Epic Rollup` run in the repo (its
`epic-rollup.yml` caller of vergil-actions' `ops-epic-rollup.yml`, which runs
`vrg-epic-rollup --task <closed issue>`). A run takes about 45 seconds to
set up before the command starts. If the repo is archived while a run is
queued or in progress, the run fails with
`Repository was archived so is read-only`. Two of the five rollups in the
`logical-minds-foundry/mq-protocol-gateway` retirement failed this way
(vergil-project/vergil-tooling#5005).

## Steps

In the commands, `<org>/<repo>` is the repo being retired.

1. **Close the open issues, children before their epics.** List them:

   ```bash
   vrg-gh issue list --repo <org>/<repo> --state open --limit 500 \
     --json number,title,labels
   ```

   Close each task before the epic, queue epic or archive that holds it, as
   not planned, with a comment naming the work that retires the repo:

   ```bash
   vrg-gh issue close <N> --repo <org>/<repo> --reason "not planned" \
     --comment "Repo retired by <ref>."
   ```

   Note the time of the first close (UTC, ISO 8601), for example from
   `vrg-gh issue view <first N> --repo <org>/<repo> --json closedAt`.

2. **Wait for the rollups the closes started.** List the repo's
   `Epic Rollup` runs created at or after the first close:

   ```bash
   vrg-gh run list --repo <org>/<repo> --workflow "Epic Rollup" \
     --created ">=<first close time>" --limit 100 \
     --json databaseId,status,conclusion,createdAt,url
   ```

   A run can take a little while to appear after its close, so expect one
   run per close you made. Repeat the list until every run's `status` is
   `completed`: none may be `queued`, `waiting`, `pending`,
   `requested` or `in_progress`. Then check that each finished: its
   `conclusion` should be `success`. A run with any other conclusion is
   investigated (`vrg-gh run view <id> --repo <org>/<repo> --log-failed`)
   and, if needed, re-run with `vrg-gh run rerun <id> --repo <org>/<repo>`
   before going on. Never archive with a rollup unfinished or failed.

3. **List the open issues again** with the step 1 command, and close any
   opened in the meantime, children before their epics. If you closed any,
   go back to step 2 for the runs those closes started. Go on only when
   the list is empty and every rollup run has finished.

4. **Hand off to the human to archive the repo.** Comment on the task that
   the repo is ready to archive, and stop. The human archives it; the task
   then checks that it is archived:

   ```bash
   vrg-gh repo view <org>/<repo> --json isArchived
   ```

   Expected: `"isArchived":true`.

## Related

- [Epic #1476's retrospective](../epics/1476-remove-private-repos/retrospective.md),
  which asked for the re-list after the last close (step 3)
- vergil-project/.github#1686, which added the wait for the rollups (step 2)
