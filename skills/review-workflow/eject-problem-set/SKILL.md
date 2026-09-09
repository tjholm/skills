---
name: eject-problem-set
description: Convert a list of review findings into self-contained task files under .review/, each fixable by a fresh agent with no memory of the review that produced it. Use when a code review, audit, or bug backlog returns five or more findings or spans three or more files, and whenever the user asks to split, eject, or park a problem set.
---

# Eject a problem set

A findings list is useless to a fresh agent because it leans on context the reviewing session had. Ejecting turns each finding into a standalone brief.

**Clean-context test.** The receiving agent gets one task file and the repo, nothing else. If it has to ask a question, guess an invariant, or read the review to start work, the file is incomplete. Apply this test to every section below.

## 0. Setup

Record the commit under review with `git rev-parse --short HEAD`. Every task file carries it.

Decide where `.review/` lives:

- Agents share one checkout: add `.review/` to `.gitignore`.
- Agents run in separate worktrees or clones: commit it, otherwise they cannot see their tasks.

## 1. Group findings into tasks

Findings map to tasks n:m.

- **Merge** findings that share one fix, or that sit in the same function. Separate agents editing the same lines will conflict.
- **Split** a finding spanning independent files into one task per file.
- **File as `wontfix`** anything you cannot state a concrete failure for, with the reason in the body. Do not drop it silently and do not hand an agent an unverified finding.

Group so that no two `todo` tasks list the same file. Where that is impossible, set `depends_on`.

Aim for one to four findings per task. If an agent could not finish it in one session, split it.

## 2. Write one file per task

Path: `.review/<run-slug>/NN-<task-slug>.md`, numbered in run order.

### Frontmatter

Fixed schema. Other skills scan these keys, so do not rename or extend them here without updating those skills.

```yaml
---
status: todo
severity: high
commit: a1b2c3d
files:
  - path/to/file.ext
depends_on: []
---
```

| key | values |
| --- | --- |
| `status` | `todo`, `in-progress`, `done`, `blocked`, `wontfix` |
| `severity` | `critical`, `high`, `medium`, `low`. For a merged task, use the highest of its findings. |
| `commit` | short SHA the finding was observed at |
| `files` | every file the fix may touch. Must include every path named in the body. |
| `depends_on` | task numbers that must reach `done` first, `[]` if none |

`status` is the only record of progress. `blocked` and `wontfix` both require a reason in the body.

### Body

In this order:

**Title.** One line.

**Location.** `path/to/file.ext:120-134`, with the current code quoted inline. Line numbers go stale as soon as an earlier task edits the file, so the quoted snippet is authoritative: the agent finds the site by matching the snippet.

**Defect.** A concrete failure. Given these inputs or this state, the result is wrong. Not "this is fragile".

**Required outcome.** What must be true when the task is done. Prescribe a patch only when one specific fix is acceptable, otherwise state the goal and let the agent choose.

**Context it cannot infer.** The invariant being violated, the callers that depend on current behaviour, the related path that must stay consistent, the convention this repo follows. This section decides whether the handoff works.

**Verification.** The exact command and what its passing output looks like. Derive it from the repo: `nix flake check` or `nixos-rebuild build --flake .#<host>` for a Nix config, `go test ./...` for Go, and so on. For a behavioural finding a passing build is not verification, so give the runtime check as well.

**Out of scope.** The adjacent problems that belong to other tasks, named by symptom rather than by task number. Without this, agents expand into each other's work.

Prose rules: imperative and self-contained. No "as noted above", no reference to the review or the session. Every pronoun resolves inside the file. Quote code instead of describing it.

## 3. Write the index

`.review/<run-slug>/INDEX.md` holds:

- one line of origin, being what was reviewed and at which commit
- a table of task file, one-line summary, severity

No status column and no ordering column. Both live in task frontmatter and would only drift here. The index is a map, and nothing may exist only in it.

## 4. Stop

Report the directory and the task count. Do not fix anything, do not dispatch agents, do not summarise the findings again. The files are the deliverable.

## Self-check

Run these before reporting:

1. Every path mentioned in a body appears in that file's `files`.
2. No path appears in two tasks unless one lists the other in `depends_on`.
3. Every location has a quoted snippet.
4. Every file has exactly one verification command with expected output.
5. `grep -riE 'as (noted|mentioned) above|the review|task [0-9]' .review/<run-slug>/[0-9]*.md` returns nothing.
6. Every `blocked` and `wontfix` file states why.