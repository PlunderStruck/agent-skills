---
name: design-by-contract
description: State and honour what a boundary promises — preconditions as the caller's obligation, postconditions as the supplier's guarantee, class invariants, why re-checking your own precondition makes a system less reliable rather than more, and the substitutability rule for overrides. Use when writing a public method or interface, deciding who validates what, overriding a method or implementing an interface, or writing an error-handling path.
---

# Design by Contract

## Purpose

Two questions cause more argument than they should: *should this function validate its arguments?* and *what is this function actually promising?*

Both are unanswerable without a rule for whose responsibility a condition is. Without one, teams oscillate between trusting blindly and checking everything — and checking everything feels safer while making the system worse.

A contract makes the division explicit, which turns both questions into design decisions rather than matters of taste.

## The metaphor, and the part people get backwards

Every function boundary has two sides with obligations and benefits:

|  | Caller | Supplier |
|---|---|---|
| **Precondition** | Obligation — must arrange it | **Benefit** — fewer cases to handle |
| **Postcondition** | **Benefit** — can build on it | Obligation — must achieve it |

The part that gets missed: **a precondition is a benefit to the function, not a burden on it.** It narrows what the function must cope with. A strong precondition makes the supplier's job easier; a strong postcondition makes the caller's job easier. Interface design is negotiating that trade, not maximising either side.

## Preconditions

**Assign every consistency condition to exactly one side.** Either the caller guarantees it (a precondition) or the function handles it as a normal case (an ordinary branch, not an error). Both is worse than either.

Two legitimate styles:

- **Demanding** — the caller is responsible. Prefer this for software-to-software boundaries, especially reusable code. A function trying to tolerate every possible abuse ends up doing three jobs: its actual work, guessing what abnormal input meant, and reporting that guess. It does none well.
- **Tolerant** — the function handles it. Legitimate, but then it isn't a precondition and shouldn't be documented as one.

**Two constraints on what may be a precondition:**

1. **It must be justifiable from the specification**, not from your implementation's convenience. A condition that exists only to protect one implementation choice should be restated in terms of what the function promises — or dropped.
2. **The caller must be able to observe it.** A precondition referencing private state the caller cannot inspect is not a contract; it's a trap.

## Non-redundancy: don't re-check your own precondition

**The rule:** once a condition is the caller's responsibility, the function must not re-test it.

This feels wrong. An extra check seems free. The argument that it isn't:

- Locally, one redundant check is harmless. **Systemically, every redundant check is new code** — with its own bugs, its own maintenance, and its own drift away from the condition it duplicates.
- Multiply by every routine and you get a system chasing reliability by adding code, which needs more checks, indefinitely — while the actual cause of unreliability, a fuzzy division of responsibility, is never addressed.
- The hardware intuition doesn't transfer. Software doesn't wear out or accumulate transmission noise, so sender-and-receiver-both-verify has no analogue in a software-to-software call.

If a precondition is violated at runtime, that is unambiguously a **bug in the caller**. The function's correct behaviour is undefined, and it owes nothing.

**If you don't trust the caller, that's a signal the condition belongs on the callee side** — or that this isn't a software-to-software boundary at all. It is not a signal to check twice.

Reliability comes from specifying more precisely and checking less redundantly.

## Postconditions

State what is true on return, relative to the inputs and the prior state — **not how the function computes it.**

The difference matters: "assign `count + 1` to `count`" is a mechanism. "On return, `count` equals its entry value plus one" is a promise. Only the second survives a refactor.

This produces the rule that makes postconditions worth writing: **anything true of the current implementation but not stated in the postcondition is an implementation detail, and callers depending on it are depending on something you never promised.** Without a stated postcondition, the implementation silently becomes the specification, and any behaviour-preserving refactor becomes a breaking change by accident.

## Class invariants

For a type whose fields must stay mutually consistent — a count and its backing array, a cached total and its source list, a connected flag and a live socket.

**Write the rule once, at the type level.** It holds whenever the object is observable from outside: after construction, and before and after every public method. It is explicitly **permitted to be violated mid-method** — otherwise you could never write a method touching two fields.

That permission is what makes invariants practical rather than theoretical.

Effectively the invariant is conjoined to every public method's pre- and postcondition — the implementer may assume it on entry and must restore it on exit. Keeping it stated once rather than copied into each method is what lets it document what the type *is*, and what makes it apply automatically to methods added later.

## Contracts are not input validation

A precondition describes an agreement between two pieces of software you control. It says nothing to a human at a prompt or a malformed payload off a socket.

A "precondition" of the form *input is well-formed* on a function reading raw external input is wishful thinking wearing a check's clothing.

**Put a filtering layer at the edge.** Its job is to accept messy external input and either reject it or convert it into something that satisfies the internal preconditions downstream — which can then stay simple and demanding. Contracts resume behind the filter.

Related: don't use assertions for ordinary case dispatch. Expected cases are conditionals. Preconditions describe what must hold under correct usage, and their violation is a bug rather than a branch.

## Define errors out of existence

Before writing a handler, ask whether the *contract* can be redefined so the condition isn't exceptional at all.

The reflex that detecting more is safer makes operations grow an exception per slightly-off input. Each one joins the interface, and every caller needs code to survive it whether or not it can do anything useful in response — handling code that is disproportionately hard to get right and rarely exercised before it matters.

Often a small change to what the operation *promises* removes the case entirely. "Delete this variable" can mean *ensure it is gone* rather than *fail if it was not there*. An out-of-range substring can clamp rather than throw. A selection that doesn't exist yet can be an empty range at a valid position rather than a nullable field every reader must guard.

Where the condition genuinely can't be designed away, **mask it at the lowest level that can actually act on it** — retry inside the transport, not in every caller — or aggregate distinct exceptions into one handler where the response is identical regardless of which fired.

**Check.** For each exception a module can throw: would a slightly different definition of the operation make this case ordinary? For each nullable or defaulted field: can the special case be represented as an unremarkable instance of the general one?

## Contracts do not replace tests

A contract states what should be true. A test is independent evidence the implementation achieves it under real conditions — including the adversarial and boundary cases a specification won't surface on its own.

What contracts do for testing is change the *failure mode*: a violated assertion converts a silent wrong-answer bug into an immediate, localised failure that also says which side broke the agreement.

## When checking in advance isn't possible

The discipline has honest limits, and recognising them is part of applying it rather than an exception to it. Three cases where an up-front precondition is the wrong tool:

- **Verification costs as much as the operation.** You cannot cheaply determine a matrix is invertible without doing most of the work of inverting it.
- **The property isn't expressible** as a simple assertion — global structural conditions like "this graph is acyclic."
- **The condition depends on the outside world** — a device, a network, a user — and can only be established by attempting the operation.

In all three, attempt the operation and check the *outcome* instead of gating on the input. That's not a violation of the discipline; it's the discipline correctly identifying which form of contract is available.

**When reviewing what looks like a redundant check, classify before deleting.** If it guards something genuinely expensive or impossible to verify in advance, it's legitimate. If it cheaply re-verifies what the caller already promised, it's the redundancy this skill is about.

## Reference

| Topic | File |
|---|---|
| Overriding a method, implementing an interface, writing an error path | [inheritance-and-errors](references/inheritance-and-errors.md) |
| Enforcing a contract in the type system rather than by convention | [types-as-contracts](references/types-as-contracts.md) |

## Boundary with neighbouring skills

- **`decomposition`** — *where* the boundary goes, and what it should hide. This skill covers what the boundary, once placed, promises.
- **`unit-testing`** — what a test should assert. A well-written postcondition is the observable-behaviour half of that decision, stated as a contract instead of an assertion; the test is then an executable instance of it.
- **`api-evolution`** — compatibility when a shape changes for external consumers. Related to the substitutability rule here, but about wire and schema compatibility rather than subtype behaviour.
