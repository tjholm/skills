---
name: walkthrough
description: Trace a change or module in execution order with the user, stopping at each decision to ask what they expect before revealing it, and finishing with the user explaining it back. Use when the user wants to understand code they own but did not write, when working through a comprehension-debt target, and when the user says "walk me through", "help me understand this", or "I don't actually know how this works".
---

# Walkthrough

Code that passes its tests and that no human understands is a liability with a green badge. A walkthrough converts one unit of that code into code someone can reason about.

## 1. Scope

Agree the unit: a function, a module, a change set, an entry point and what it reaches. Under about three hundred lines per sitting. Larger than that, split by entry point and do several.

Identify the entry points and the exit points: where control comes in, where it leaves, what it returns or mutates.

## 2. Trace

Walk from the first entry point in execution order. Show the actual code for each step, a few lines at a time, not a paraphrase of it.

At every branch, loop boundary, error path, or call into something non-trivial, stop and ask what the user expects to happen next before showing it. A right answer moves on. A wrong answer is the point of the exercise: show what actually happens and why, then continue.

At every decision that is not obvious from the code, say why it was made if the reason exists: a comment, a commit message via `git log -S`, a linked issue. If no reason is recorded anywhere, say that. An unrecorded decision is a finding.

Do not skip the boring parts silently. Say "these twelve lines copy fields into the DTO, nothing conditional" and move on.

## 3. Explain back

When the trace is complete, the user explains the unit in their own words: what it does, what it assumes, where it can fail. Two to five sentences.

Compare against the trace and name the gaps concretely:

- a path they did not mention
- an assumption or invariant they missed
- something they stated that the code does not actually do

No praise for what they got right. The gaps are the output.

## 4. Leave the understanding in the code

Nothing is recorded outside the repository. What the walkthrough found goes where the next reader will meet it:

- a decision with no recorded reason gets a one-line why-comment at the site, or a note in the next commit that touches it, following the repo's comment rules
- code the walkthrough revealed as dead, wrong, or hedging is named as a candidate for `simplify-code` or an issue, not fixed now
- what the user missed in the explain-back is theirs to note; do not write it into the code

Then report those three lists, briefly.

## Do not

- Summarise the code instead of showing it. The user is here to read it.
- Reveal what happens at a branch before asking.
- Turn the walkthrough into a review. Findings are noted, not fixed, unless the user stops to fix one.
- Walk more than one unit in a sitting without finishing the explain-back for the first.
