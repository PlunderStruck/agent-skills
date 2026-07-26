# When Not To Split

The half of decomposition advice that usually goes unwritten. Factoring is a cost, not a virtue, and several of the most expensive structures in a codebase were added by someone following good advice too eagerly.

## Wait for the third instance

With **one** instance of something, you cannot tell whether a general solution is warranted at all. With **two**, it's marginal. With **three**, the shared shape becomes visible enough to generalise *from* — rather than toward an example you imagined.

And even then: expect real instances to diverge enough that a single universal structure ends up wrong. The more complex the underlying need, the less likely any one abstraction covers all of it.

**Trigger.** You're about to build a generic or configurable mechanism for a need you have exactly one concrete case of.

**What to do.** Write the concrete case. Build the general shape when you have two or three real, divergent examples in front of you.

**Check.** Can you point at actual second and third callers, or are they hypothetical?

## Defer the restructuring you already know is coming

It is normal — and frequently correct — to notice mid-implementation that two concerns are tangled in a way that should eventually be split, and to choose not to split them yet.

This isn't laziness. The right shape of the split usually isn't knowable until more of the system exists, and restructuring from a partial picture guesses wrong and gets redone.

**What to do instead of pre-building.** Prepare for the anticipated change by organising *information* cleanly — honest names, boundaries that accurately describe what exists today — rather than by pre-building *mechanism*: hooks, flags, strategy layers, configuration points the future change might use.

Add complexity when the iteration in front of you requires it, not when a future one might.

## Don't factor for its own sake

A short phrase that recurs is a tempting extraction target. Resist when:

- The raw phrase is already more transparent than any name you can invent for it.
- It turns out to be used once anyway.

**Good factoring sometimes reduces the number of names in a system.** Collapsing several near-duplicate named things into one thing parameterised by what actually varies is itself evidence the factoring was right. A rising name count is not automatically progress — it's frequently the opposite.

## Distinguish essential duplication from coincidental duplication

Two passages that look alike are a standard extraction trigger, and the trap is that resemblance isn't intent.

A helper extracted from three similar sites gets a name. A fourth, superficially similar case later needs a different final step. The tempting move — add a parameter, push the fourth case through the existing name — is usually wrong. It's the signal that the fragment always served two purposes wearing one name, and wants splitting at a finer grain instead.

**Check.** Does this piece serve one purpose, or does it currently look the same across sites that don't share intent? **If a later caller needs to bolt a condition onto the shared piece to handle its case, the earlier extraction was coincidental.**

## Interrogate whether the complexity is real

Given two solutions that both meet the requirement, prefer the simpler — not as a stylistic tiebreaker, but because it's easier to verify, debug and change. Added complexity has to be justified by something; it is never free.

Before building machinery to match a requirement that looks combinatorially complex, ask a specific question: **does this complexity describe the actual problem, or was it inherited from someone's prior solution being redescribed as the requirement?**

A specification built around a legacy implementation's shape — separate hand-tuned terms mirroring components of an old design — can collapse dramatically once you re-derive from what's actually needed rather than encoding what was asked for literally.

## Don't build a private interpreter

The urge to make something "configurable" or "data-driven" by inventing a small command language, rule engine, or plugin layer rarely reflects something genuinely unique about the problem. It reflects the mechanism being enjoyable to build.

Once it exists, the interpreter becomes the largest, least-tested, least-used part of the system — and you have quietly changed jobs from solving the problem to maintaining a runtime.

**Check.** Is there something about this problem a plain function or module genuinely cannot express, or are you enjoying building the indirection?

## Don't bury your tools

The opposite failure, and it comes from the same instinct toward tidiness.

An abstraction that exposes only a fixed menu of finished, "safe" operations — deliberately hiding the primitives it's assembled from — looks clean until a caller needs a composition its designer didn't anticipate. A hardware layer offering sixteen canned commands but no path to fast bulk output forces everyone who needs that to bypass the abstraction entirely.

**Expose the composable primitives alongside the convenience operations.** An abstraction usable only in the ways its author imagined gets worked around rather than respected — and the workarounds are worse than the primitives would have been.

## The unifying question

Every rule here reduces to one thing: **what does this structure buy, and what does it cost?**

The structures that hurt most are rarely the ones nobody thought about. They're the ones added deliberately, early, by someone applying a real principle to a case that hadn't earned it yet.
