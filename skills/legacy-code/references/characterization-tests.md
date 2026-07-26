# Characterization Tests

A characterization test records what code does **right now**. It carries no opinion about whether that behavior is correct. Its only job is to fail later if someone changes the behavior without meaning to.

This is what makes it safe to touch code nobody fully understands. You are not required to reason your way to the right answer first.

## The procedure

It is deliberately mechanical, because the whole point is that it works without understanding the code.

1. Get the code running in a harness — whatever minimal construction it takes.
2. Write an assertion you believe is wrong, or none at all.
3. Run it and let it fail.
4. Read the actual value out of the failure output and paste it in as the expected result.
5. Repeat for other inputs, especially boundaries.

Cover each branch of a conditional with its own case. A single test that happens to exercise the common path proves only that the common path ran.

Where it's ambiguous whether a branch actually executed — or whether an implicit conversion happened — add a throwaway instrumentation flag to confirm, then delete it once you know.

## Scoping: what else needs covering

Don't test only the method you're editing. Trace forward along the three channels an effect can travel:

- **The return value**, read by a caller.
- **A mutated parameter**, read later by whoever holds that object.
- **Shared or global state**, read somewhere else entirely.

The third is the least visible and the most likely to surprise you.

Sketch it as a graph: a node for everything that could change, an arrow to everything downstream that could observe the change. Stop at a boundary the language genuinely enforces. Where a boundary is only conventional — a naming pattern, a comment, a team agreement — keep tracing, because nothing prevents a violation.

## Pinch points

When a change ripples across several classes, you don't necessarily have to break dependencies in each one.

Look for a **pinch point** — one method, or a small handful, that all the scattered effects funnel through. Covering the pinch point buys temporary safety for the whole cluster. You can then refactor underneath it freely as long as those tests stay green, and replace them with narrower per-class tests as real coverage grows.

This is often the difference between a change that takes an afternoon and one that takes a week.

## When to stop

There is no natural stopping point — the input space is infinite. Use two passes:

1. Stop once your genuine uncertainty about the behavior is resolved.
2. Then ask specifically: **would these tests catch a mistake at the change I'm about to make?** If not, add more.

If you still can't reach confidence, that's a signal to shrink the scope of the change rather than proceed unprotected. A smaller change needs a smaller net.

## When you find a bug while characterizing

You will. The code does something visibly wrong, and now you have a decision.

- **If nothing depends on the current behavior yet** — fix it outright.
- **If it's already in use** — flag it and resolve the question before touching it. Something downstream may have adapted to the bug, and "fixing" it breaks the adaptation.

Either way, **record what the code actually does today as the test.** The characterization test documents reality; the bug decision is a separate conversation. Conflating them is how you end up with a test asserting behavior the system never had.

## What these tests are not

They aren't specifications, and they shouldn't be treated as permanent. They're scaffolding: they hold the shape while you work. Once an area is genuinely covered by tests written against intended behavior, the characterization tests around it can be retired.

Leaving them in place indefinitely creates a subtle trap — a future reader assumes a pinned value was a deliberate requirement, when it was only ever a snapshot of whatever the code happened to do the day someone needed to change it.
