---
name: intent-first
description: Get the intent of a change written down before any code is written, then judge the finished work against that intent rather than against taste. Use whenever asked to implement, change, or fix something that is more than a few lines, when the user says "before you start" or "here's what I want", and when the user has written something themselves and asks what they missed.
---

# Intent first

A change judged against the agent's own sense of good code passes when it is good code. A change judged against a stated intent passes when it is the right code. Writing the intent down before the work is where the human's thinking happens; everything after is verification.

## 1. Get the intent

For anything beyond a trivial edit, do not start from a one-line request. Ask for, or extract from what the user said:

- **What must be true afterwards.** Observable behaviour, in terms of inputs and outputs or user-visible effect.
- **What must not change.** Existing behaviour, public signatures, performance characteristics, files that are off limits.
- **Constraints.** Libraries to use or avoid, patterns this repo follows, deadlines that rule out the thorough approach.

Restate it back in at most five lines and wait for a yes. If the user's request already contains all three, restate without asking.

This is not a plan or a design document. It is the acceptance criteria, short enough to hold in the head while reviewing.

Skip this for edits where the intent is the request: a rename, a typo, a one-line fix the user has already described precisely.

## 2. Do the work

Hold the intent, not the request, as the specification. When the two diverge, the intent wins and the divergence is worth a sentence.

## 3. Report the decisions the intent did not make

Every non-trivial change involves choices the intent left open. Surface each one, in one line, with the alternative not taken:

```
Stored pending jobs in a map keyed by id, not a list — O(1) cancel; list would keep insertion order
Unknown status from the API raises, does not log-and-skip — intent said "must not silently lose jobs"
Added retry on 429 with backoff — intent said nothing about rate limits; remove if unwanted
```

Cover: data structures, error handling strategy, names of anything public, dependencies added, behaviour added that was not asked for, tests changed or removed, files touched outside those the intent named.

Then, separately and always, the red flags, even to say there are none:

- functionality the intent did not ask for
- tests deleted, disabled, or weakened
- retries, loops, or fallbacks added around something that failed during the work

Do not list decisions the intent settled. The report is for the reader to audit judgment calls, not to re-read the diff.

## 4. Check mode

When the user has written the change and asks what they missed, review against the intent only:

- a case the intent requires that the code does not handle
- something the intent forbids that the code does
- an invariant the intent states that a path breaks

Not style, not structure, not what you would have done. If the intent was never stated, ask for it before reviewing; a review without one is a review against taste.

## Do not

- Pad the restatement into a plan, a task list, or a design.
- Ask for intent on trivial edits.
- Hide a scope expansion inside the work and mention it only in the report. Stop and ask first.
- Offer to widen the scope. The report may note what else was seen; the user decides.
