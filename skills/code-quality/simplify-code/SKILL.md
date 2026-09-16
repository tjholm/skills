---
name: simplify-code
description: Review a file, module, diff, or codebase for code its actual problem does not require, and report each instance with the fact that makes it unnecessary. Targets the shape of generated code — guards for unreachable states, options nobody passes, abstractions with one user, layers that only forward, branches that hedge between input shapes, logic reimplemented from the standard library. Use when asked to simplify, de-bloat, or tidy code, when reviewing AI-generated code, or when a function spends more lines preparing for what might happen than doing what does. Reports findings; fixes only when asked to.
---

# Simplify code

Review whatever code you are pointed at for complexity the problem does not demand. Report findings only. No praise, no restating what the code does, no rewriting unless asked to fix.

Generated code is complicated for one reason: the author did not know the facts. It did not know what the callers pass, whether a value can be null, what already exists in the repo, or which of two shapes the input really has. So it hedged, and every hedge became a branch, an option, a wrapper, or a check. A finding here is therefore not "this is too complex"; it is "this exists because of a fact the author lacked, and here is the fact".

**The test.** For every branch, parameter, layer, and abstraction: name the concrete caller, input, or requirement that needs it. If you can, it stays. If you cannot, and you can state the fact that rules it out, it is a finding.

## 1. Establish the facts

Before judging, for the unit under review:

- **What must it do?** The requirement in one or two sentences, from the callers and the tests rather than from the code's own comments.
- **What are the real inputs?** `grep` every call site. Record the actual types and shapes that arrive, which parameters are always the same value, always present, or never passed.
- **Where is the boundary?** Request handlers, CLI parsing, file and network reads, deserialisation, environment, external APIs. Everything inside receives values the boundary already checked.
- **What already exists?** Repo utilities, the standard library, established patterns for the same job.
- **Why is the odd part odd?** For anything that looks unjustified but deliberate, `git log -S'<snippet>'` before flagging it. A workaround with a reason in its commit message is not a finding.

A fact you cannot establish is not a finding either. Say what you could not determine and move on.

## 2. Find the unearned complexity

Read each function top to bottom. At every branch, parameter, indirection, and abstraction, ask what fact would have let the author leave it out. Patterns, grouped by the missing fact:

### Did not know the inputs

| pattern | unnecessary when |
| --- | --- |
| `if (x == null)`, `if x is None` | the type is non-nullable, or every caller passes a value |
| `typeof x === 'string'`, `Array.isArray`, `isinstance` on a typed parameter | callers are in-repo and typed |
| accepting `T \| T[]`, `string \| string[]`, and normalising | every caller passes one shape |
| `x ?? default`, `x \|\| {}` | the left side cannot be nullish |
| `if (!list.length) return []` before a loop | the loop already does nothing on empty |
| `a?.b?.c` on a fully required type | no link is optional |
| re-validating or re-parsing a value parsed upstream | the function should accept the parsed type |
| `.trim()`, `.toLowerCase()`, `Math.max(0, n)` on a normalised value | the boundary already normalised it |
| `default:` throwing "unreachable" on an exhaustive switch | the compiler enforces exhaustiveness |
| `if (!this.ready) throw` in every method | construction is the only way to get the object and it readies it |

### Did not know the requirement

- A parameter, option, or config field that every caller sets to the same value, or never sets.
- A generic with one instantiation. An interface with one implementation. An abstract class with one subclass.
- A registry, strategy, plugin, or factory for two cases that a conditional would serve.
- A feature flag, mode enum, or `options.legacy` branch that nothing toggles.
- A callback or event where the only subscriber is defined next to the emitter.
- Anything justified by a comment containing "future", "extensible", "in case", or "for flexibility".

### Did not know what already existed

- A hand-rolled `groupBy`, `chunk`, `debounce`, `deepClone`, `isEmpty`, path join, date format, or retry loop when the standard library or an existing repo utility does it.
- A manual index loop doing what `map`, `filter`, `find`, or a comprehension does.
- A custom error class hierarchy for one error site.
- A pattern that differs from how the rest of the repo does the same thing.

### Did not decide where things go

- A function whose body is a single call to another function with the same arguments.
- A helper used once, defined far from its only use.
- A class that holds no state and exists to namespace one method.
- A layer that receives a value and passes it on unchanged.
- A file split into `types.ts`, `utils.ts`, `constants.ts`, `helpers.ts` for forty lines of code.

### Did not reason about the flow

- Two branches of a conditional that do the same thing.
- A condition tested, then tested again a few lines later on the same path.
- A value computed, then recomputed from the same inputs.
- A conversion `A → B → A`.
- A flag set on one line and checked on the next.
- A state variable that is always derivable from another.
- `try { … } catch (e) { throw e }`, or a catch that logs and rethrows into a caller that logs again.
- `async` on a function that awaits nothing, `await` on a value that is not a promise.
- A defensive copy of a value nothing mutates.
- Intermediate variables that hold a value for one line and add no name worth reading.

### Narrated instead of wrote

- Comments restating the line below them, `// Step 1:` markers, section banners.
- Doc comments that repeat the signature parameter by parameter.
- A log line per step of a function that has no operational reason to log.

## 3. Prove each finding

A finding without evidence is an opinion, and a fresh agent acting on it later will delete real behaviour. Each finding carries one of:

- **Type.** The declared type excludes the case.
- **Callers.** Every call site is in the repo; list them; none produces the case or passes the option. If callers exist outside the repo, the function is a boundary and there is no finding.
- **Upstream.** An earlier check on the same path already rejects the case, named by file and line, and nothing between can reintroduce it.
- **Construction.** The value is built by code you can read, named, which always produces the assumed state.
- **Existence.** The library or utility that replaces the code does the same thing for the same inputs, checked against its documentation, not its name.

If the proof would be "the type says so" but the type is `any`, `unknown`, `interface{}`, or `dict`, the finding is the type, not the guard: flag the loose type, with the guard as its consequence.

Nothing with no proof is reported as a finding. It may be reported, separately, as a fact that could not be established.

## 4. Report

Every finding, regardless of how expensive the fix is or how many there are. A scoped request narrows what you are asked about, not what you may flag.

Each finding: location — what is unnecessary — the fact that makes it so, with the proof — the fix — what the fix unlocks, if anything. For example:

> `src/jobs/parseConfig.ts:41-44` — null check and early return on `opts`. `opts: ParseOptions` is non-optional and all four callers (`cli.ts:88`, `server.ts:120`, `worker.ts:33`, `parseConfig.test.ts:12`) pass a literal. Fix: delete lines 41-44. Unlocks: return type becomes `Config` not `Config | null`, which makes the null checks at `cli.ts:90` and `server.ts:122` dead too. *(callers)*

> `src/export/Exporter.ts:1-60` — `ExportStrategy` interface with two implementations, both in this file, selected by a string. Neither is used elsewhere; nothing registers a third. Fix: one function with an `if` on the format. *(callers)*

> `src/util/collections.ts:12-30` — hand-rolled `groupBy`. `Object.groupBy` is in the repo's target (`tsconfig` lib `es2024`) and does the same for these inputs. Fix: replace and delete. *(existence)*

Order by what each fix unlocks, most first. Group findings that share one fact: a tightened return type and the caller checks it makes dead are one finding with several locations.

After the findings, and separately:

- **Could not establish.** Facts you needed and could not get, each with what would settle it. These are not findings.
- **Type problems.** Loose types whose tightening would turn several hedges into provably dead code.

Then stop. Five or more findings, or three or more files, is `eject-problem-set`'s territory; the format above gives it what each task file needs. If asked to fix rather than review, apply the fixes, run the type checker and tests before and after, and delete any test that fails only because it fed an impossible input to an internal function.

## What is not a finding

- Validation at a boundary, however excessive it looks.
- Error handling for I/O, network, filesystem, and process failures. These are always reachable.
- Checks required by a public contract that external code depends on.
- Anything whose history gives a reason. Read the reason before deciding it no longer applies.
- Complexity the requirement demands. A hard problem is allowed hard code; the target is code that is harder than its problem.
