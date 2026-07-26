# Breaking Dependencies

## Find the obstacle by construction, not by reading

Don't study the constructor looking for what will be a problem. Write a bare construction call with no assertions and let the compiler tell you what's missing. The error list *is* the obstacle list, and it's shorter than the one you'd assemble by reading.

Then match the obstacle to the fix:

| Obstacle | Move |
|---|---|
| Constructor argument is expensive or unavailable | Extract a narrow interface, pass a fake |
| Dependency created *inside* the constructor | Add an overload accepting it; keep the original signature as a default-supplying wrapper |
| Dependency buried deep in construction | Overridable factory method — only where the language dispatches virtual calls during construction; otherwise a post-construction swap |
| Pervasive singleton | A way to swap its backing instance under test |
| Sealed or external library type | Depend on the narrowest slice of its behavior you actually use |

Once the class instantiates but one method still won't run:

- **Private method** — test it through its public caller. Loosening visibility is a last resort, not an opening move.
- **Invisible effects** — peel the method into named pieces that separate computation from mutation, so the unobservable part can be isolated and overridden.
- **Touches no instance state** — pull it out as a static or free function.
- **Long and tangled** — move it wholesale into its own small class.

## Working safely before tests exist

Step 3 of the change procedure happens without the net it's building. That warrants stricter discipline in place of the missing tests:

**Single-goal edits.** Do exactly one thing per edit. When you notice a second necessary change, write it down rather than interleaving. Two changes made together can't be mentally unwound independently, and when something breaks you won't know which one did it.

**Preserve signatures.** When relocating code purely to open a seam, copy parameter lists and call sites by cut-and-paste rather than retyping. Retyping introduces exactly the class of error — a transposed argument, a subtly different type — that you have no tests to catch. Save genuine API cleanup for after coverage exists.

**Lean on the compiler.** Deliberately break a declaration and let the resulting errors enumerate every call site that must change. This beats hunting manually. But be clear about what it proves: structural consistency, not preserved behavior. It has blind spots — reflection, dynamic dispatch, string-keyed lookups, anything resolved at runtime.

**Get a second pair of eyes.** This is the one category of edit with no automated net, which makes review worth its cost here specifically.

Once characterization tests exist, all of this can relax.

## Monster methods

An oversized method resists testing in one of two ways:

- **Bulleted** — loosely related chunks in sequence, with temp variables quietly threading state between them.
- **Snarled** — deep nesting that's hard to trace at all.

**If you have a refactoring tool** that guarantees behavior-preserving extraction, use it exclusively for the session. Mixing in manual edits voids the guarantee, because the tool can no longer verify its assumptions about what you changed.

**Without one:**

- Extract the smallest, most confidently nameable chunk first.
- Prefer chunks with **no values crossing the boundary** — nothing to pass means nothing to get wrong.
- For chunks that do share state, add a throwaway sensing variable, write a quick test against it, extract, confirm the test still passes, then delete the instrumentation.
- Keep new extractions **on the current class**, even when they look like they belong elsewhere. Moving them is a second change; do it later.
- Expect to redo early extractions. The right seams only become visible once the shape starts emerging.

## Code with no structure

Loss of structure is usually loss of *shared understanding* as much as tangled code. Nobody holds the whole shape, so people patch wherever they last worked, and the drift compounds.

The fix isn't a rewrite or a grand redesign — it's incremental and folded into ordinary work.

The useful signal: **a concept everyone names fluently in conversation that has no corresponding thing in the code.** People talk about "the retry policy" or "the eligibility rules" as though they exist, and in the code they're loose counters and scattered conditionals across six files. That gap is a missing class, and it can be extracted one change at a time as you touch those files for other reasons.

## Choosing a technique

Before opening the catalogue, name the exact obstacle — never the general feeling that the code is bad. Then ask three questions in order:

1. **Construction problem or invocation problem?** Can't build it, or can't run the method?
2. **One call site or many?** A single site takes a local fix. A dependency reached from dozens of places needs one shared seam introduced once, with the compiler used to find every remaining caller.
3. **Testing existing behavior, or adding new behavior?** For new behavior, don't force the surrounding code under test first — isolate the new logic behind the smallest possible seam and test that instead.

The third question is the one most often skipped, and it's usually the cheapest path.
