# Choosing Boundaries

## Before you split anything

**Understand the problem well enough to restate it more simply.** Requirements arrive as tangles — rate tables, mode matrices, nested exceptions. Decomposing a tangle as given just distributes the tangle across more files.

Work on the requirement first:

- Separate the independent dimensions buried inside it.
- Check each dimension's cases are genuinely exhaustive and mutually exclusive.
- Look specifically for a dimension that is really a *parameter to one calculation* rather than a grid of special cases. This is the most common hidden simplification.

If you can't compress the problem statement, you're not ready to draw lines in it.

**Settle the outer interface and error policy first.** Error handling isn't a local decision. Whether failures surface, get swallowed, get retried, or halt processing changes the shape of nearly every unit's signature. Deciding it after the pieces exist means reworking all of them.

Test: can you describe what happens on bad input without opening any particular module?

## The two legitimate reasons for a seam

A boundary earns its keep if — and only if — one of these is true:

1. **Something will reuse this piece.** Not hypothetically; name the second caller.
2. **This piece will change independently of its neighbours.** Name the requirement that would move on its own.

If neither is nameable, you have a line, not a boundary.

## Group by change, not by sequence

The most common wrong axis is execution order, because it's the one the problem statement hands you: input → process → output, step one → step two.

Sequence-shaped boundaries pass review and fail requirements changes, because order is rarely the axis along which requirements move.

**The editor case.** Split an editor into an edit stage and a separate display-refresh stage. It looks correct — refresh is decoupled from any particular edit, which is exactly what a boundary is supposed to achieve. Then the requirement changes from full-screen redraw to incremental redraw. Refresh now needs to know precisely which edit occurred, and the boundary exists specifically to hide that. It gets torn open and its logic redistributed into every edit operation.

The rival design — keeping "mutate the data" and "update what that affects" together inside each operation — looks less tidy on a diagram and absorbs the same change as one added line per operation.

**Ask:** if this specific detail changed, what data and behaviour would have to move together? Draw the line around that.

## Shared state needs an owner

**Trigger.** Two components both reach directly into the same structure — a buffer, a record, a config blob.

**What goes wrong.** Each side develops its own private understanding of what the structure means. A change to one side's assumptions breaks the other, and nothing — not the compiler, not a reviewer reading either file alone — can see why.

**What to do.** Give the structure and the operations on it a home of their own that both sides call into. Express what crosses that boundary in terms meaningful to the *problem* rather than the representation — degrees rather than raw sensor counts — so a different implementation on either side doesn't force the other to change.

## Not everything read across a boundary is a stable interface value

A subtler case, and one that produces bugs rather than just friction.

A formatting component sets a margin width; an output component reads it. That looks like ordinary shared interface data. But the formatter is free to change that value between the moment output reads it and the moment output still depends on it — silently corrupting work that was already considered finished.

**The question:** can the producer legitimately change this value while the consumer's result is still in flight?

If yes, it isn't an interface value. It's the producer's private mutable state that a consumer happens to be reading. Hand the consumer a frozen snapshot rather than a live variable.

## The calling protocol leaks too

An allocator with cleanly named `allocate` and `mark` operations still leaked its internal direction-of-growth decision — because every one of twenty call sites was implicitly required to call them in a particular order. When the growth direction reversed, all twenty sites needed editing regardless of how good the names were.

**Test:** could the internal decision flip with zero change to the *order* callers must invoke things in?

If not, the abstraction leaks through its calling convention. Good names hide the decision from a reader; only a good protocol hides it from a caller.

## Verifying a decomposition

Don't look at it and judge tidiness. **Pick one plausible future requirement and trace it.**

- Does the change stay inside one piece?
- Or does it force edits across several — meaning none of them actually owns the thing that changed?

This is the only check that distinguishes a boundary that reflects the problem's structure from one that reflects the order somebody happened to write things in.
