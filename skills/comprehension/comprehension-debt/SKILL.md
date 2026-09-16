---
name: comprehension-debt
description: Find the code in a repository that has the measurable shape of generated code, rank it by risk, and produce a list of walkthrough targets. Use when the user asks where the comprehension debt is, what parts of the codebase nobody understands, what looks AI-generated, what to walk through next, or how the debt has changed since last time.
---

# Comprehension debt

Comprehension debt is the gap between the code a team owns and the code it understands. It produces no friction until someone has to change the code, so it has to be measured deliberately.

Who wrote a file is not the question and is not reliably knowable. What is knowable is whether the code has the shape generated code tends to have, and how often it changes.

## Signals

**Shape.** Three measurable traits of generated code, each backed by large-scale analysis of AI-era commits rather than by intuition:

*Duplication.* The strongest signal. Assistants paste rather than reuse: across 2023–2026, duplicated blocks per million changed lines rose 81% and copy/pasted lines outnumber moved-and-refactored lines about five to one (GitClear). Per file, the share of lines that are part of a clone elsewhere in the repo:

```sh
npx jscpd --min-tokens 50 --reporters json --output /tmp/jscpd --silent . \
  && jq -r '.duplicates[] | [.firstFile.name, .lines] , [.secondFile.name, .lines] | @tsv' /tmp/jscpd/jscpd-report.json \
  | awk -F'\t' '{d[$1]+=$2} END{for(f in d) print d[f] "\t" f}' | sort -rn > /tmp/dup-lines
```

*Error masking.* Constructs that catch and continue rose 47% over the same period (GitClear). Per file, count the single-line forms of a catch that swallows or a handler that bails with nothing:

```sh
grep -cE '(catch\b[^{]*\{\s*\}|^\s*(pass|return( None| nil)?|continue)\s*$|\.catch\(\s*\(\)\s*=>\s*\{?\s*\}?\)|^\s*_ = err\b|^\s*rescue\b\s*$)' "$f"
```

*Comment density.* Machine-written code carries noticeably more comments than human code from the same tasks, and comment tokens are among the most discriminative stylometric features (Shi et al., 2024). This undercounts multi-line handlers and overcounts bare `return` in ordinary control flow; it is a ranking input, so that is acceptable.

Comment lines over total lines:

```sh
grep -cE '^\s*(//|#|/\*|\*|"""|\x27\x27\x27)' "$f"
```

Combine as `shape = dup_lines/lines + masking/lines + comment_lines/lines`, each term first divided by the repo median so no one term dominates.

This ranks; it does not prove. A careful human writing a boundary parser will score high on masking, and a clean agent will score low on everything. Treat the score as a reason to look, not a finding.

Redundant guards, options bags, and narrating comments are what practitioners report seeing in generated code, but the empirical studies do not confirm them as markers; one large review found AI code *omits* guards more often than humans do. If a codebase visibly has a house tell, add one `grep -cE` for it to the shape sum and label it as local.

**Churn.** Commits touching the file in the last ninety days: `git log --since=90.days --format=%H -- <path> | wc -l`. Frequently changed code that nobody understands is the highest risk.

If the repo happens to record authorship — `Co-Authored-By` trailers, a bot committer — use it as one more shape signal, not as truth.

## Ranking

`shape × (1 + churn) × lines`. Rank descending. Then, by judgment rather than formula, move up anything that is an entry point, a boundary, handles money or auth, or is imported widely.

## Report

Top ten to twenty:

```
score  dup%  mask  cmt%  lines  churn90  path
 41.2    18     6    22    312        4  services/billing/reconcile.ts
 18.7     9     2    31    145        2  api/handlers/webhook.ts
```

Below the table, the next three `walkthrough` targets and one line each on why. Run it again after paying some down; the output is a function of the tree and the log, and nothing else.

Nothing else. The user decides what to pay down.

## Do not

- Present the shape score as evidence of authorship. It measures duplication, masking, and commentary, which need a human's attention regardless of who wrote them.
- Add unevidenced patterns to the shape sum without labelling them local. The three defaults are the ones the data supports.
- Pay down debt in the same session. Inventory ends at the table; `walkthrough` does the work.
- Treat a high score on a file the team already understands as a reason to walk it again. That is `simplify-code`'s cue, not `walkthrough`'s.
- Report the score upward as a metric. Once it is a target it will be gamed by walking through trivial files.
