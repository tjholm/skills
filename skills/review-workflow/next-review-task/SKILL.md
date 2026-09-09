---
name: next-review-task
description: Pick up the first unhandled task file under .review/ and action it end to end, claiming it, fixing it, verifying it and recording the result. Use when the user says to work the next review task, take the next finding, or continue a parked review, and when a session is started by being handed a .review/ task path.
---

# Work the next review task

The `.review/<run-slug>/` tasks written by `eject-problem-set` are self-contained by design. This skill actions exactly one of them per session, then stops.

## 1. Select

If the user named a task file, that is the task. Otherwise:

```
grep -l '^status: todo$' .review/*/[0-9]*.md | sort
```

Take the first. Numbering is the intended order, and across several run directories the older run comes first.

Then read the task's `depends_on`. For each entry, check that task's `status` in its frontmatter only, not its body. If any prerequisite is not `done`, move to the next candidate and say which task you skipped and why.

Skip `done` and `wontfix`. Skip `in-progress`, which means another session holds it, and list any you saw in your final report so the user can spot an abandoned claim. Take a `blocked` task only when the user names it.

If nothing is `todo`, say so and stop. Do not invent work and do not re-review the code.

## 2. Claim it

Set `status: in-progress` before making any other change. This is the only lock, and skipping it lets two sessions edit the same file.

## 3. Read only your task

Read the task file. Do not read the body of any sibling task or of `INDEX.md`. Their contents are another agent's problem, and pulling them in is how one session's scope swallows the whole review.

The task file is the specification:

- **Required outcome** is what you must achieve.
- **Out of scope** is binding. An adjacent flaw you notice goes in your report, not in your diff.
- **Location** line numbers are indicative only. Find the site by matching the quoted snippet. If `git rev-parse --short HEAD` differs from the task's `commit`, expect the line numbers to be wrong and rely on the snippet alone.

If the file is incomplete, for example it assumes an invariant it never states, or the quoted snippet no longer exists anywhere in the repo, do not guess. Go to step 5 and record it as `blocked`.

## 4. Fix

Change only the files in `files`. If the fix needs a path that is not listed:

- The path appears in another task's `files`: stop and record `blocked`, because editing it here will conflict.
- It appears nowhere else: add it to `files` and continue.

Achieve the required outcome. Where the task states a goal rather than a patch, choose the fix that matches the surrounding conventions.

## 5. Verify

Run the command in the task's **Verification** section and compare its output against what that section says passing looks like. For a behavioural finding, run the runtime check too, not just the build.

Never write `status: done` off an unrun command.

If verification fails and you cannot fix it, decide what to leave behind:

- The tree still builds: keep your changes and describe them.
- Your changes leave the repo worse than you found it: revert them.

Either way the status is `blocked`, not `done`.

## 6. Record and stop

Append one section to the task file.

On success, `## Result`: what changed, the paths touched, and the verification output that proves it. Set `status: done`.

On failure, `## Blocked`: what is missing or failing, whether your changes are still in the tree, and what would unblock it. Set `status: blocked`.

Leave the file in place. `clean-review-tasks` removes it later.

Then report in one or two lines: the task, the outcome, anything you deliberately left alone, and any `in-progress` tasks you skipped. One task per session. Do not roll on to the next, so it gets the clean context it was written for.