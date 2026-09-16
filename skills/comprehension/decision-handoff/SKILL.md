---
name: decision-handoff
description: Split implementation work into rote and judgment, do the rote, and stop at each judgment point to hand the decision to the user with options and consequences. Use during any implementation task of more than a few lines, when the user says "you do the boilerplate", "stop at decisions", "let me make the calls", or "I'll take rote too", and when a choice arises with more than one defensible answer.
---

# Decision handoff

Offloading mechanical work to an agent is cheap and safe. Offloading judgment is how a codebase fills with decisions nobody made. This skill keeps the two apart: the agent takes the rote, the human takes the calls.

## What is rote and what is judgment

| rote, the agent does it | judgment, the user decides |
| --- | --- |
| imports, wiring, registration, plumbing a value through layers | business rules and their edge cases |
| test fixtures, factories, mocks that mirror an existing type | error handling strategy: fail, retry, degrade, ignore |
| config and setup code with one sensible shape | algorithm or data structure where the trade-off matters |
| mechanical refactors: rename, move, extract with no behaviour change | public API shape: names, parameters, return types, error contract |
| CRUD with no rules beyond persistence | concurrency and ordering |
| the obvious implementation when only one is defensible | what is persisted, and in what shape |
| type declarations that restate a schema | what is logged, measured, or alerted |
| following a pattern the repo already uses for the same job | user-visible behaviour and copy |
| | anything security-relevant: auth, validation at a boundary, secrets |
| | anything with two defensible answers |

The test for judgment: would two competent engineers who know this codebase plausibly choose differently? If yes, hand it over.

## Handing over

At a judgment point:

1. **State the decision** in one line. What is being chosen, where in the code.
2. **Give the options**, two or three, each with its consequence in one line. Not pros-and-cons lists; the thing that would make someone pick it and the thing it costs.
3. **Give your lean** in one line only if one option is clearly better for this codebase, or the user asks.
4. **Prepare the site.** The file open, the signature or stub in place, the test that will exercise it named. If the user chooses to write it themselves, they start typing, not searching.

Then stop and wait.

```
Decision: how `syncAccounts` handles a partial failure (3 of 40 accounts error).
  a. fail the whole run — simplest, callers already handle a thrown error; 37 good syncs redone next run
  b. continue, collect errors, throw at end — nothing redone; callers must learn to read a partial result
  c. continue, log, return success — matches `syncContacts`; errors invisible unless someone reads logs
Lean: b — `syncContacts` losing errors is an open bug (#412).
Site: services/sync.ts:88, `syncAccounts` body; test stub in sync.test.ts `describes partial failure`.
```

## Pace

Do not stop every thirty seconds. Collect judgment points that do not depend on each other and present them together at a natural pause. Stop immediately only when the next rote work depends on the answer.

Do not manufacture a decision where one answer is obviously right. Make it, and report it under `intent-first`'s decision report.

## Switches

- **"I'll take rote too"** — the agent writes nothing. It still prepares sites and frames decisions.
- **"You decide"**, for one decision or the session — the agent chooses, states which and why in one line, and moves on. The choice still appears in the decision report.
- **"Just build it"** — the skill is off for this task. Say so, so it is not silently off next time.

## Do not

- Present more than three options. If there are more, the decision is really two decisions.
- Frame a rote choice as a decision to look consultative.
- Recommend by default. The lean is for when it earns its line.
- Write the judgment code after handing it over unless asked. Handing over means the user may write it.
