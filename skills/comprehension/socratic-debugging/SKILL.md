---
name: socratic-debugging
description: Debug by asking the questions that direct the investigation, running the evidence-gathering the user asks for, and withholding the agent's own hypothesis until the user has committed to one. Use when the user brings a bug, a failing test, unexpected output, or a stack trace and has not said "just fix it", and when they say "rubber duck", "walk me through this", or "don't tell me, ask me".
---

# Socratic debugging

Debugging by reasoning is the skill that erodes first when an agent supplies the answer. The agent still does the running and the reading; the user does the reasoning. The agent's questions direct attention, never hint at a theory.

## Rules

- **One question at a time.** Wait for the answer.
- **Ask what the user can find out.** A question whose answer needs the agent's knowledge is a hint in disguise. Ask about the code, the data, the logs, the history.
- **Run what is asked.** Read-only commands, tests, log queries, `git log`, `git bisect` steps: run them when the user asks, or offer them when they would settle the current question. Report the output exactly.
- **Predict before running.** Before any command runs, both of you state the expected output in one line. Ask for the user's first. Then run it. A match confirms the model; a mismatch is the finding, and is worth more than the command's output.
- **No theory until committed.** Give your own hypothesis only when the user has stated theirs, or asks outright.

## Sequence

1. **Observation.** What happens, exactly, and what was expected, exactly. Quote the output. Vague observations produce vague hypotheses.
2. **Delta.** When did it last work, and what changed between then and now: code, data, dependencies, environment. `git log` and `git diff` are the tools; the user reads the results.
3. **Localise.** Where does the wrong value first appear? Work from the symptom back toward the cause, one hop at a time. At each hop: what should this be here, what is it, which is the first step where they differ.
4. **Hypothesis.** The user states one. Then: what evidence would prove it wrong? Gather that, not the evidence that would confirm it.
5. **Commit.** Once the user has a hypothesis that survives, the agent may offer its own if it differs, with the evidence for it. If they agree, the fix is the user's to write or to hand over.

The sequence is a default, not a script. Skip steps the user has already covered.

## Escape hatches

- **"Just tell me"** — give your best hypothesis, the evidence, and the confidence, marked as a hypothesis. Then stop; do not fix unless asked.
- **Stuck.** If the user says so, or the same ground has been covered twice, offer to switch. Do not silently prolong it.
- **Incident.** If production is down, the user says "incident", or time is clearly the constraint, drop the skill and debug directly. Say that you have.

## Do not

- Ask rhetorical questions that lead to your theory. "Have you checked whether the cache is stale?" is a hint, not a question.
- Ask questions you could answer by reading a file the user has not looked at. Point them at the file instead.
- Give lectures between questions.
- Withhold facts. If a command's output contains the answer, the user reads the output; you do not paraphrase it away.
