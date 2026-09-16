# comprehension

Keeping human judgment and understanding in a codebase that an agent writes
most of. Each skill rests on a specific finding rather than on a metaphor.

    intent-first/         acceptance criteria before code; the diff is audited against them
    decision-handoff/     agent does rote, user makes the calls, options framed at each one
    socratic-debugging/   questions direct the investigation; the agent's theory waits
    walkthrough/          trace a unit in execution order, then the user explains it back
    comprehension-debt/   inventory agent-authored code nobody has walked; rank it

`walkthrough` writes `.comprehension/ledger`; `comprehension-debt` reads it.
The others stand alone.

Grounding, briefly: Osmani and Willison on comprehension and cognitive debt;
Beck's augmented-coding red flags; the Anthropic learning output style's
rote/judgment handover; the Socratic rubber-duck projects; the CHI 2025
knowledge-worker study on where critical thinking survives GenAI use; the
Thoughtworks Radar entry on codebase cognitive debt.
