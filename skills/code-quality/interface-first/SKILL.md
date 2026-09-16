---
name: interface-first
description: Design the calling code and the domain model before writing any implementation, so that the interface is shaped by how it will be used rather than by how it happens to be built. Use before implementing any new module, service, client, type, or public function, when reviewing an interface for ergonomics, and when code exposes options bags, boolean flags, stringly-typed values, nullable returns, or leaks its internals to callers.
---

# Interface first

Implementation-first code has an interface shaped like its insides: a parameter for every branch, an option for every case the author met, a return type that is whatever the last line produced. Agents do this by default because they write the body before anyone has called it. The fix is to write the call sites first, then the model, then the interface, and only then the body.

## 1. Write the usage

Before any type or signature exists, write the three most common call sites as they would appear in real code in this repo, in the caller's vocabulary, with the caller's data already in hand. Then write one hard case: the error path, the concurrent case, the one that needs the unusual option.

```ts
const invoice = await billing.invoiceFor(customer, period)
await billing.send(invoice)

const overdue = await billing.overdue({ olderThan: days(30) })

// hard case
const result = await billing.invoiceFor(customer, period)
if (result.kind === 'already-invoiced') { … }
```

Read them as the caller. Every awkwardness here is an awkwardness every caller will meet. Fix it here, where it costs a line, not later, where it costs a migration.

If you cannot write the usage, you do not yet know what is being built. Stop and find out.

## 2. Model the domain

From the usage, list the nouns and the states each can be in, then the transitions between states. Give each a type.

- **One type per concept.** A customer id is not a string; a period is not two dates; money is not a number. A caller who can pass the wrong one is a caller who will.
- **Make impossible states unrepresentable.** A result that is `{ ok: true, value } | { ok: false, error }` cannot be both. A `status: string` with a comment listing the values can be anything.
- **States over flags.** `draft | sent | paid | void` as one field, not `isSent`, `isPaid`, `isVoid` as three that can disagree.
- **Model what the caller reasons about, not what the storage holds.** The database row is the persistence layer's problem.

Check the usage against the model. If a call site needs a value the model does not have, one of them is wrong.

## 3. Derive the interface

Now, and only now, write the signatures. Each is dictated by a call site from step 1.

- **Minimal surface.** Export what the usage calls. Nothing else, however useful it might someday be.
- **Required things are parameters. Optional things are rare.** If every caller passes the same option, it is not an option. If one caller passes it, ask whether that caller wants a different function.
- **No boolean parameters.** `render(doc, true)` tells the reader nothing. A two-value enum or two functions.
- **No options bag unless there are four or more genuinely independent optional settings.** Under that, named parameters or a small config type with defaults.
- **Return what the caller needs next**, not what the implementation has. If every caller immediately unwraps, transforms, or checks, the return type is wrong.
- **Errors are part of the signature.** Decide which failures the caller must handle, and put them in the type, the documented throw list, or the result variant. Nothing else escapes.
- **Names from the caller's vocabulary**, matching the repo's existing verbs. `fetch`, `get`, `load`, and `retrieve` are four names for one thing; the repo has already picked one.
- **Sync unless it must wait.** `async` on a function that awaits nothing is a cost every caller pays.
- **Nothing leaks.** No parameter that exists to tune the implementation. No return type that is the ORM's, the HTTP client's, or the protobuf's.

The interface is a judgment point. Under `decision-handoff`, it goes to the user before step 4.

## 4. Implement

Write the body to satisfy the signatures. The signatures do not move to make the body easier. When the body wants to change the interface, that is information about the model, not permission to change the interface; go back to step 2 and check.

## 5. Read it as a stranger

Delete the usage from step 1 from your head and read the interface cold:

- Can it be called wrongly in a way that compiles? Then the type is too loose.
- Does the reader need the implementation to know what a parameter does? Then the name or type is wrong.
- Is there a parameter, option, or exported symbol that no call site in step 1 needs? Remove it.
- Would a second implementation, in-memory or fake, be small? If it would need most of the same parameters and branches, the interface is exposing the implementation.

## Do not

- Start with the type of the data source and work outward. Start with the caller and work inward.
- Add a parameter "in case". Add it when the second caller exists.
- Accept several shapes of input for convenience. Convenience for the caller who passes the odd shape is a cost to every reader of the function.
- Return `null`, `undefined`, `any`, or a bare `Error` where a variant or a narrower type says what actually happened.
- Let the implementation's first draft decide anything about the interface.
