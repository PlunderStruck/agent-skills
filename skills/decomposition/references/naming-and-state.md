# Naming and State

## Naming is a correctness check, not a labelling step

**If you can't give it one non-compound name, it isn't a well-formed concept yet.**

This is the sharpest rule here, and it runs in both directions. Struggling to name an extracted piece is not a vocabulary problem to push through — it's feedback that the boundary is wrong.

**What goes wrong if you push through.** Shipping the adequate-but-not-right name lets the fuzzy concept survive. The fuzziness resurfaces later as awkward call sites built around something nobody can quite describe.

**What to do.** Treat the difficulty as a prompt to re-examine what the chunk actually *is*. Often a workaround name is standing in for a real, distinct concept nobody has recognised yet — and finding it is worth more than the extraction was.

## Name by effect, not mechanism

A name should say what a piece accomplishes for its caller, not how it works inside.

**Check.** Could the internals be rewritten completely without the name becoming a lie? If a reimplementation would make the name inaccurate, you named the mechanism.

## Read the call site as a sentence

Choose names against the phrase they'll form where they're used, not in isolation. If a call site doesn't parse as a natural description of an action, the boundary underneath it is probably in the wrong place — not merely labelled badly.

This is the same test that catches concatenated names and numbered identifiers: **how fast the name count grows as you add variants is evidence about the boundary.** Factoring that collapses combinations into composable parts scales; factoring that enumerates them does not.

---

## Where state lives

**Default to the narrowest scope, and prefer explicit parameters over ambient state.**

A unit that reads or writes something outside its own parameters can't be tested or reused in isolation, and can't be verified without recreating whatever ambient context it silently depends on.

When introducing state, start local to one routine. Widen only when a second, genuinely independent component needs it.

**Check.** Can you call this twice in a row, in isolation, and get consistent results?

## Set state at the start rather than saving and restoring at the end

Save-then-restore looks tidy and breaks under any non-local exit. An early return, an error, an abort — each skips the restore and leaves corrupted ambient state for something else, somewhere else, to trip over later. The failure surfaces far from its cause.

**What to do.** Have an operation establish the state it needs explicitly, rather than mutating shared state and promising to put it back. Where restoration genuinely can't be avoided, centralise it in one routine invoked at every real exit path instead of duplicating restore logic per path.

## Invert, don't wrap

**Trigger.** You have a routine that operates on some shared value, and now need a version operating on an arbitrary value without disturbing the shared one. The tempting move is a save/restore wrapper around the existing routine.

**What to do instead.** Extract the core behaviour to take the value as an explicit parameter, then rebuild the original ambient-state version as a thin call into the now-parameterised core.

This removes the fragile save/restore step rather than adding a second layer of it.

## Two uses of one shared flag

Two independent uses of a single shared flag or resource are safe only under two conditions: **they never overlap in time**, and **each restores prior state on the way out**.

If either can't be guaranteed — if the uses can nest — a boolean toggle means one use's cleanup clobbers the other's still-active state.

**What to do.** Either give the second concern its own independent state, or replace the boolean with a nesting counter so overlapping activations compose instead of colliding. Guard against the counter going negative on an unpaired call, because that's how you find out the pairing was wrong.

## Values always used together are one concept

If you repeatedly save, restore, or swap the same cluster of values as a unit, they aren't several variables — they're one structure that hasn't been named.

Merge them so the group moves atomically. Saving five of six related variables, or restoring them in the wrong order, is a failure mode that a grouped structure eliminates by construction rather than by discipline.

## Estimating

Related, and easy to skip: **estimate cost by summing the smallest real pieces, not by judging the whole.**

A change that feels trivial as a single lump reliably costs several times a naive estimate once redesign, retesting, and documentation across every actually-touched piece are counted. Leaving a task undecomposed *in your head* produces bad judgement about its size — independently of whether the resulting code is any good.
