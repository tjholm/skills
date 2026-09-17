---
name: whiteboard-defense
description: Pre-shipping check that the developer's understanding of a change matches what the agent actually wrote, by having them explain the change and defend its decisions, then grading each answer against the diff. Use before opening or merging a PR for work an agent wrote most of, when the user says "whiteboard me", "quiz me on this", "can I defend this", or "does this do what I think", and after a walkthrough to confirm the understanding stuck.
---

# Whiteboard defense

The benchmark for responsible use of an agent: anyone should be able to pull you aside and ask how the thing you shipped works and why, and you can answer. Not the function names or the line-level detail. The mechanism, the trade-offs, the trust boundaries, and where it breaks.

When an agent writes most of a change, the developer's picture of it comes from the conversation, not from the code. The two drift. This check finds the drift before a reviewer or an incident does.

Skip it for throwaway work the user says is throwaway. Otherwise, anything about to ship qualifies, whoever it is for.

## 1. Read the change

Read the diff against the base branch, plus enough surrounding code to know what it touches. Pull out the material:

- decisions: places another competent engineer would plausibly have done differently
- trust boundaries: points where input arrives from something that could lie
- data structures and storage shapes that carry the new state
- failure modes: timeout, partial failure, duplicate delivery, bad input, races
- anything the diff does that the conversation did not ask for

Questions come from this list, not from a checklist.

## 2. Expectation first

Ask the user to say, in a few sentences, what the change does and how. Before any question, compare that against the diff and hold the differences:

- behaviour in the diff they did not mention
- behaviour they mentioned that the diff does not have
- a mechanism they described that the code does differently

These become questions in the next step. Do not reveal them yet.

## 3. Ask

Five to eight questions, one at a time, each naming something in this change. Wait for each answer. Cover all four kinds, and aim the questions at the places their expectation and the diff diverged:

| kind | shape |
| --- | --- |
| **decision** | "Why X instead of Y?" where Y is a real alternative the code did not take |
| **adversary** | "What happens if this actor behaves maliciously?" for a specific actor at a specific boundary |
| **structure** | "What data structure did you use here and why?" where the shape matters to correctness or cost |
| **failure** | "Where does this fail?" for a specific path: dependency down, message twice, two requests race |

A good answer is a mechanism. "Retries find the existing row because the idempotency token is the key in the orders table" defends. "There's an idempotency check" does not; follow up once with "how?". If the second answer is still a label, record the gap and move on. Do not teach mid-check.

## 4. Grade against the code

Each answer is one of:

- **Matches.** The mechanism and the reason match the code. Line-level inaccuracy does not count against it.
- **Don't know.** A gap. Note where in the diff the answer lives.
- **Diverges.** What they expect is not what the code does. The central finding: this is the change they would have shipped believing something false about it. Note what they said and what the code does.
- **No reason exists.** The decision cannot be defended because nobody made it: the agent chose, and nothing records why. "The agent did it that way" is not a defense.

## 5. Report

```
Matches    5 of 7
Diverges
  failure    expected retries to stop at 3; code retries until the 30s deadline — jobs/sync.ts:112
  unasked    diff adds a fallback to cached prices when the rates API errors; not mentioned — pricing/quote.ts:58
Gaps
  adversary  handler trusts account_id from payload before the signature check — api/webhooks.ts:41
No reason on record
  decision   pending jobs in a list, not a map; cancel is O(n) — jobs/queue.ts:20
```

Then one line: ready to ship, or what to do first. The options are always the same three:

- **walk it through:** each gap or divergence is a `walkthrough` target at the line named
- **record the reason:** each undefended decision gets a why-comment at the site, or gets reconsidered
- **fix the code, not the understanding:** when the divergence is because the code does something the developer never wanted

The user decides. Nothing is written outside the code and its history.

## Do not

- Ask for function names, file paths, or lines. The test is the model, not the recall.
- Ask generic questions. Every question names something in this diff.
- Give the answer before the user has committed to one.
- Accept a label as a mechanism. "It's validated" is the prompt for the follow-up, not the answer.
- Turn it into a review. What the code should have done belongs to `code-critique` or `simplify-code`; here, only what it does and whether the user knows.
- Soften a divergence. Say what they expected, say what the code does.
