# Interface Depth

Where [choosing-boundaries](choosing-boundaries.md) asks *should a seam exist here*, this asks *how much should cross it*. Same job, different measurement.

## Name which complexity you mean

"This is too complex" is not a usable complaint until it resolves to one of three separable costs. Fixes aimed at the wrong one make things worse — fewer lines can increase cognitive load by hiding logic behind indirection, and a shorter diff can still touch six files.

| Symptom | Question | Fix direction |
|---|---|---|
| **Change amplification** | Does a small conceptual change require edits in many places? | Consolidate the decision to one place |
| **Cognitive load** | How much must someone hold in mind to make one safe change? | Hide irrelevant detail — *not* the same as writing less code |
| **Unknown unknowns** | Is it even possible to know what else needs changing? | The worst of the three: nothing tells you it exists until a change breaks something distant |

Underneath all three sit two causes: **dependencies**, which force coordination across places, and **obscurity**, which hides that a dependency exists at all.

## Deep versus shallow modules

**Judge a module by the ratio of functionality it provides to the size of interface required to reach it.**

Unix's handful of file I/O calls cover disk layout, permissions, scheduling, and caching. That's depth: a small interface concealing a large amount of work.

A shallow module is the inverse — a wide interface with little behind it. A getter. A one-line delegator. A class whose entire body sets one field. Each costs a name, a signature, and a slot in every caller's memory, and returns almost nothing.

**Trigger.** Deciding whether a class, function, or service earns its boundary — at design time, or reviewing something that looks suspiciously thin.

**What goes wrong.** "Smaller is better" gets applied to *interfaces* the way it's applied to function length. The result is many classes each with a tidy-looking interface, and a system that's harder to use because a caller must now learn N small interfaces instead of one deep one. This is **classitis**: the pieces each look simple, and the total interface complexity is higher than one deep module would have been.

**The check.** Would fully documenting this module's interface take about as many words as documenting its implementation? Then it's shallow, however organised it looks.

**Watch for "split anything over N lines" as a house rule.** Applied mechanically, it manufactures shallow pieces at scale.

This is the measurable version of the rule that a boundary needs a reason. Depth gives that judgement a ratio instead of a feeling.

## Different layer, different abstraction

**Trigger.** A method body is almost entirely "call the same-named method on the thing I wrap." Or a variable threads through four signatures where only the innermost reads it.

**What goes wrong.** Adjacent layers end up presenting the *same* abstraction rather than different ones. A pass-through method adds a name and a dependency for zero new functionality — and when the wrapped signature changes, the wrapper changes too, having bought nothing. Pass-through variables do this to arguments: a credential threaded through three methods that never inspect it so a fourth can.

**What to do.**

- **Pass-through method:** decide which class actually owns the responsibility. Expose the lower layer directly, redistribute the functionality, or merge the classes — don't leave a stub on both sides.
- **Pass-through variable:** resist the two tempting non-fixes. A "shared object" that is itself a pass-through target one level up solves nothing, and a global breaks multiple instances and testability. A context object — one struct carrying an instance's shared state, passed explicitly into constructors — keeps the variable out of every intermediate signature.

A context is not clean. Without discipline it decays into an unlabelled grab-bag, so keep its fields immutable and documented.

**Check.** Could the suspected pass-through be deleted, with the caller invoking the wrapped method directly, losing nothing? Then it isn't pulling weight.

## Write the interface comment first

**Trigger.** About to write a new class or function, and the shape doesn't feel settled enough to document yet.

**What goes wrong.** A comment written after the code describes what the code *happens to do*, not what a caller needs to know — and by then the designer has moved on, so it gets minimum effort. Worse, a design flaw hides successfully behind code that already compiles and passes tests.

**What to do.** Write the interface comment before the body: behaviour, each argument, return value, side effects, exceptions, preconditions. This forces the abstraction into a form a caller can rely on **without reading the implementation**, which is a different and harder exercise than making the code work.

**The signal.** If the comment can't stay short without omitting something a caller needs, or can't be complete without describing internals a caller shouldn't need — that's information about the design, delivered before anything is built on top of it.

This is the paragraph-length version of the rule that a piece you can't name in one non-compound word has a wrong boundary. Same test, more resolution.

## Information leakage without a shared interface

`choosing-boundaries` covers the main case — the same fact understood by two modules, and the sequence-shaped splits that cause it. (Ousterhout calls that *temporal decomposition*, and diagnoses it independently: splitting by *when* something happens rather than by *what it needs to know*.)

The refinement worth adding: **leakage doesn't require a shared interface to be real.** Two classes can each embed knowledge of a file format with neither exposing it publicly. Nothing in either file looks wrong. The coupling surfaces only when the format changes and both need editing.

**Check.** Pick a design fact and ask how many modules would need to change if it changed — including ones that never call each other.
