---
name: comprehension-debt
description: Inventory a repository for code that was agent-authored and that no human has demonstrably understood, rank it by risk, and produce a paydown list of walkthrough targets. Use when the user asks where the comprehension debt is, what parts of the codebase nobody understands, what to walk through next, or how the debt has changed since last time.
---

# Comprehension debt

Comprehension debt is the gap between the code a team owns and the code it understands. Unlike technical debt it produces no friction until the day someone has to change the code, so it has to be measured deliberately.

## Signals

**Authorship.** Agent-authored commits carry trailers. Collect their SHAs:

```sh
git log --all --format='%H' -i --grep='Co-Authored-By:.*\(Claude\|Copilot\|Codex\|Cursor\|Gemini\|Aider\|Devin\)' > /tmp/agent-commits
```

Adjust the pattern to what this repo's agents actually write. If the repo has no trailers, say so and stop; do not infer authorship from style. The user may name the agent-heavy areas instead.

**Agent fraction per file.** Share of current lines whose last change was an agent commit:

```sh
git ls-files | while read -r f; do
  git blame --line-porcelain -- "$f" 2>/dev/null \
    | awk -v f="$f" '/^[0-9a-f]{40} /{n++; sha[$1]++}
        END{ while((getline s < "/tmp/agent-commits")>0) a+=sha[s]; if(n) printf "%d\t%d\t%s\n", a, n, f }'
done | awk '$2>0{printf "%.2f\t%d\t%s\n", $1/$2, $2, $3}' | sort -rn > /tmp/agent-fraction
```

Slow on large repos. Restrict `git ls-files` to source paths if it takes more than a minute.

**Churn.** Commits touching the file in the last ninety days. Frequently changed code that nobody understands is the highest risk.

**Understood.** `.comprehension/ledger`, written by `walkthrough`, lists files a human has traced, with the SHA at the time. A file counts as understood if it has a ledger entry and no agent commit has touched it since:

```sh
git log --format=%H <ledger-sha>..HEAD -- <path> | grep -qxf /tmp/agent-commits && echo stale || echo understood
```

## Ranking

Score each file as `agent_fraction × (1 + churn)`, zero if understood. Rank descending. Weight upward, by judgment rather than formula, anything that is an entry point, a boundary, handles money or auth, or is imported widely.

## Report

A table, top ten to twenty:

```
score  agent%  lines  churn90  last-walked  path
 4.20     84%    312       4   never        services/billing/reconcile.ts
 2.10     70%    145       2   2026-03-02*  api/handlers/webhook.ts
```

`*` marks a stale ledger entry. Below the table:

- total lines, agent lines, understood lines, as three numbers
- the change since the last report if `.comprehension/last-report` exists; then overwrite it
- the recommended next three `walkthrough` targets and why those

Nothing else. The user decides what to pay down.

## Caveats to state

- Trailers mark commits, not lines. A human who rewrote half of an agent commit's lines before committing still shows as agent.
- A human review at PR time is not in the signal. If the team wants review to count, add a trailer such as `Reviewed-by:` and treat those commits as understood.
- The score is for ordering, not for reporting upward as a metric. Once it is a target it will be gamed by walking through trivial files.

## Do not

- Guess authorship from code style.
- Pay down debt in the same session. Inventory ends at the table; `walkthrough` does the work.
- Re-walk a file whose ledger entry is current just because its agent fraction is high. Understood is understood.
