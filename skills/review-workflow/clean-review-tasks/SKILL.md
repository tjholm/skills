---
name: clean-review-tasks
description: Prune finished task files from .review/ so only live work remains, confirming each claimed fix before deleting it and removing a run directory once it is empty. Use when the user asks to clean up, prune, tidy, or close out review tasks, or to see what is left in a review run.
---

# Clean up finished review tasks

`.review/` is a worklist. Finished entries left in it get re-read, re-fixed and re-reported, so they are removed, but only once their claimed fix has been confirmed to exist.

Deletion is not recoverable when `.review/` is gitignored. Survey first, get the user's go-ahead, then prune.

## 1. Survey

```
grep -H '^status:' .review/*/[0-9]*.md
```

Report counts per run directory: `todo`, `in-progress`, `done`, `blocked`, `wontfix`.

If the user only asked what is left in the run, stop here.

Otherwise say which files you intend to delete and wait for the user to confirm before changing anything.

## 2. Confirm each closed task

A `done` status is a claim, not proof. For every `done` file, in this order:

1. It must have a `## Result` section naming what changed. A `done` with no result is unverified.
2. Re-run the command in its **Verification** section and compare against the passing output that section describes. This is the strongest check and the file already contains it.
3. Where verification cannot be re-run, for example it needs hardware or a deploy, inspect the site instead: the quoted snippet in **Location** should no longer match, and the required outcome should hold in the current code.

`git diff` alone does not confirm anything. It shows nothing once the fix is committed and further commits have landed.

If the fix is absent:

- Set `status: todo`.
- Update the **Location** snippet to the current code if it has moved or changed, otherwise `next-review-task` will block on a snippet it cannot find.
- Record what was missing under `## Reopened`.
- Keep the file and list it in your report.

`wontfix` files need a stated reason, not a fix.

## 3. Record before deleting

Where the record goes depends on how the repo holds `.review/`:

- **Committed:** no ledger needed. Git history holds the task files.
- **Gitignored:** the task files are the only copy, so write a ledger line per closed task to a tracked file outside `.review/`, defaulting to `docs/review-log.md` and creating it if absent. Confirm the path with the user the first time. One line each: date, run slug, task, outcome, and for `wontfix` the reason.

Do not put the ledger in the run's `INDEX.md`. That file is deleted in step 4, which would destroy the record in the same pass that wrote it.

## 4. Prune

Delete the confirmed `done` and `wontfix` files.

For each deletion, remove that task's number from any remaining task's `depends_on`. The dependency is satisfied, and a reference to a file that no longer exists leaves `next-review-task` unable to check it.

Leave `todo`, `blocked` and `in-progress` files in place. An `in-progress` task is a live claim by another session, so never delete or reopen one on your own judgement. Flag every one you find and let the user decide.

When a run directory has no numbered task files left, delete the directory including `INDEX.md`.

## 5. Report

Counts pruned and counts remaining, plus the three things that need the user:

- Confirmed fixes still uncommitted in the working tree. The task file describing them is gone now, so the commit message wants writing while it is fresh.
- Anything reopened, with what was missing.
- Every `in-progress` task, so a claim abandoned by a dead session gets noticed.