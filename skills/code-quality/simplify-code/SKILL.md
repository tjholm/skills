---
name: simplify-code
description: Strip a file, module, or diff down to the code its actual problem requires. Targets the shape of generated code — guards for unreachable states, options nobody passes, abstractions with one user, layers that only forward, branches that hedge between input shapes, logic reimplemented from the standard library — and rewrites each unit as someone who knew the real inputs and callers would have written it. Use when asked to simplify, de-bloat, or tidy code, when reviewing AI-generated code, or when a function spends more lines preparing for what might happen than doing what does.
---

# Simplify code

Generated code is complicated for one reason: the author did not know the facts. It did not know what the callers pass, whether a value can be null, what already exists in the repo, or which of two shapes the input really has. So it hedged, and every hedge became a branch, an option, a wrapper, or a check. The result is code that reasons about a problem far larger than the one it solves.

Simplifying is therefore not trimming. It is recovering the facts the author lacked and then writing the code that a person who had them would have written. Usually that code is a fraction of the size.

**The test.** For every branch, parameter, layer, and abstraction: name the concrete caller, input, or requirement that needs it. If you can, it stays. If you cannot, it goes.

## 1. Establish the facts

Do this before editing. Read the unit under review, then answer in writing:

- **What must it do?** State the requirement in one or two sentences, from the callers and the tests rather than from the code's own comments. The code often solves a more general problem than anyone asked for.
- **What are the real inputs?** `grep` every call site. Record the actual types and shapes that arrive. Note which parameters are always the same value, always present, or never passed.
- **Where is the boundary?** Mark where untrusted data enters: request handlers, CLI parsing, file and network reads, deserialisation, environment, external APIs. Everything inside receives values the boundary already checked.
- **What already exists?** Look for repo utilities, the standard library, and established patterns for the same job. Generated code reinvents what it did not know about.
- **Why is the odd part odd?** For anything that looks unjustified but deliberate, run `git log -S'<snippet>'` or `git blame` before removing it. A workaround with a reason in its commit message is not bloat. If the history shows it arrived in the same generated commit as everything else, it has no reason.

If a fact cannot be established, say so and leave the code that depends on it alone.

## 2. Find the unearned complexity

Read each function top to bottom. At every branch, parameter, indirection, and abstraction, ask what fact would have let the author leave it out. The patterns below are grouped by the fact that was missing.

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

## 3. Rewrite from the facts

Piecewise deletion leaves the skeleton of the hedging behind. For each function, once the facts are in hand, ask: what is the shortest correct code that does what the callers need with the inputs they send? Write that. Compare it with the original to be sure every reachable behaviour is preserved, then replace.

Rules while rewriting:

- **Tighten types instead of checking values.** If a guard exists because the parameter is `any`, `unknown`, `interface{}`, or `dict`, the fix is the type, and then the guard is provably dead. If tightening is out of scope, leave the guard and report the type.
- **Write the specific, not the general.** Replace the strategy pattern with the `if`. Replace the generic with the concrete type. Replace the option with the value every caller passes.
- **Inline single-use abstractions** into their one call site, unless the name is doing real work for the reader.
- **Follow the consequences.** Removing a nullable return lets you remove the null checks in every caller. Removing an option removes its plumbing through every layer. One fact often collapses several sites; take them all.
- **Delete, do not soften.** No comment saying the case cannot happen. No `assert` in place of a guard unless the repo already uses assertions for invariants. No condition left with an empty body.
- **Match the surrounding code.** The result should look like the rest of the repo wrote it, not like a different style arrived.

## 4. Prove each removal

A removal justified only by intuition is how real behaviour disappears. For each thing you delete, hold one piece of evidence:

- **Type.** The declared type excludes the case, and the checker still passes without the guard.
- **Callers.** Every call site is in the repo and none produces the case or passes the option. If callers exist outside the repo, the function is a boundary; leave its checks.
- **Upstream.** An earlier check on the same path already rejects the case, and nothing between can reintroduce it.
- **Construction.** The value is built by code you can read, which always produces the state assumed.
- **Existence.** The library or utility you are replacing the code with does the same thing for the same inputs, checked against its documentation, not its name.

If none applies, the code stays.

## 5. Verify

Type checker and tests pass before and after.

A test that fails because it fed an impossible input directly to an internal function, or exercised an option nothing passes, was testing the hedge rather than the behaviour. Delete it with the code it tested. If a boundary case is now uncovered, write the test at the boundary.

A test that fails for any other reason means your fact was wrong. Restore the code and record why before continuing.

## 6. Report

The diff, then one line per removal naming the fact that justified it:

```
parseConfig: dropped null check on opts — non-optional, all 4 callers pass a literal
Exporter: replaced Strategy interface with if/else — two implementations, both in this file
fetchAll: removed `parallel` option — every caller passes true
```

Nothing else. Do not list what you considered and kept, except a typing problem the user should fix.

## What is not unearned

- Validation at a boundary, however excessive it looks.
- Error handling for I/O, network, filesystem, and process failures. These are always reachable.
- Checks required by a public contract that external code depends on.
- Anything whose history gives a reason. Read the reason before deciding it no longer applies.
- Complexity the requirement demands. A hard problem is allowed hard code; the target is code that is harder than its problem.
