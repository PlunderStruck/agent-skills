---
name: decomposition
description: Decide where a boundary goes — and whether it earns its keep at all. Grouping by what changes together rather than by execution order, the signals that a chunk is secretly two ideas, the signals you have factored too far, using a name as a correctness check, and why premature generality costs more than duplication. Use when splitting something up, extracting a function or module, designing structure for new code, or judging whether an abstraction is worth building.
---

# Decomposition

## Purpose

Most advice about breaking code apart pushes in one direction: extract more, abstract more, generalise more. That advice is half a method, and the missing half is why codebases end up with layers nobody can justify.

This skill covers both: where a boundary should go, and when there shouldn't be one.

## A boundary needs a reason

Exactly two reasons justify a seam: **something will reuse this piece**, or **this piece will change independently of its neighbours**.

If you can't name which one applies — a specific second caller, or a specific requirement likely to shift on its own — it isn't a boundary yet. It's a line drawn because the file felt long.

The default wrong axis is **sequence**. "Input, then process, then output." "Step one, step two, step three." Those splits survive code review because they look organised on a diagram, and they fail the first real requirement change, because order is almost never the axis along which requirements actually move.

**Group by what changes together, not by what happens in sequence.**

The illustration worth internalising: an editor split into an edit stage and a separate refresh-the-display stage looks clean — refresh is decoupled from any particular edit. Then the requirement changes from redrawing the whole screen to redrawing incrementally, and refresh suddenly needs to know exactly which edit happened. The boundary was specifically built to hide that. It has to be torn open and its logic scattered back into every edit operation. A design that instead kept "change the data" and "update what that affects" together inside each operation absorbs the same requirement as one added line per operation.

## The test that actually works

Don't inspect a proposed decomposition and ask whether it looks tidy. **Take one plausible future requirement and trace it on paper.** Does the change stay inside one piece, or does it force edits across several?

A decomposition that has only been looked at has not been tested.

## Rules that apply without loading anything

**1. Simplify before you split.** Decomposing a tangle distributes the tangle. If you can't restate the requirement more simply than it was handed to you, you don't understand it well enough to draw boundaries in it yet.

**2. Settle error handling before dividing anything.** Whether failures surface, get swallowed, or halt processing changes the signature of nearly every unit. Deciding it after the pieces exist means reworking all of them.

**3. Wait for the third instance before generalising.** With one case you can't tell whether a general solution is warranted. With two it's marginal. With three the shared shape is visible enough to generalise *from* — rather than toward an imagined example. And expect real instances to diverge enough that a single universal structure is wrong anyway.

**4. If you can't name it with one non-compound word, the boundary is wrong.** Struggling to name an extracted piece is not a vocabulary problem to push through. It's feedback. A name like `enable-left-motor` is a cross-product of two independent concepts glued together; a name with a number baked in is data smuggled into an identifier.

**5. Name by effect, never by mechanism.** Test: could the internals be rewritten completely without the name becoming a lie? If not, you named the implementation.

**6. Fewer names can be progress.** Collapsing several near-duplicate named things into one parameterised by what actually varies is evidence the factoring was right. A rising name count is not automatically improvement.

**7. A good name doesn't prove the boundary is sound — check the calling protocol too.** If every call site must invoke things in a required order, the internal decision leaks through the convention even though the names look clean.

**8. Design it twice before committing.** Sketch two or three genuinely different alternatives — not variations on one idea — and list concrete pros and cons. A first idea is rarely challenged because nothing exists to contrast it against, so its weaknesses surface mid-implementation instead of on paper. If none of them is attractive, the shared weakness usually points at the design that's actually missing. Hours against days.

**9. Treat design quality as a running expense.** Roughly a tenth to a fifth of time on work that doesn't ship today's feature — picking a cleaner shape before typing it, fixing a design problem when noticed rather than filing it. Each shortcut is locally cheap; the accumulation is what costs, which is why the damage stays invisible until cleanup is a multi-month ask nobody approves. For any shortcut taken under pressure, ask whether there's a concrete point at which it gets paid back. "Never, we'll live with it" is the answer to worry about.

**10. Defer the restructuring you know is coming.** Noticing that two concerns are tangled, and choosing not to split them yet, is often correct — the right shape isn't knowable until more of the system exists. Prepare by naming things honestly, not by pre-building the hooks and generic layers the future change might need.

## Triage

| What you're doing | Reference |
|---|---|
| Deciding where the seam goes; shared state between components | [choosing-boundaries](references/choosing-boundaries.md) |
| Suspecting a chunk is doing two things; conditionals as a design signal | [when-to-split](references/when-to-split.md) |
| About to generalise, configure, or add a layer | [when-not-to-split](references/when-not-to-split.md) |
| Naming an extracted piece; deciding where state lives | [naming-and-state](references/naming-and-state.md) |
| Judging whether a boundary is deep enough; pass-through layers; interface comments | [interface-depth](references/interface-depth.md) |

## Working iteratively

Neither extreme survives contact with a real project. Planning everything before writing code fails because requirements aren't knowable until someone reacts to something running. Skipping design entirely fails differently and later: the system becomes unmaintainable within a couple of years because no record exists of *why* boundaries are where they are — often not even to the people who drew them.

The workable middle is enough of a conceptual model to know the problem's shape, then a tight loop of implement a slice, test it against the requirement, revise.

Two disciplines that make the loop work:

- **Build throwaway prototypes under an explicit rule that the code doesn't survive.** This removes the pull to freeze a boundary just because it already works. Teams that let a prototype quietly become the product reliably end up with worse boundaries.
- **Change as little as possible between test cycles**, and get the primary behaviour working before adding the variation that modifies it. When something breaks you want to know unambiguously which change broke it.

## Boundary with neighbouring skills

- **`complexity-cleanup`** and the maintainability-review family fire on code that *already exists and is measurably complex* — complexity reports, high-complexity symbols, architecture smells. They're review-shaped.
- **`decomposition`** fires at design time: you're about to draw a line, or judging whether an abstraction you're considering earns its keep.

They meet on one point — both warn against mechanical helper extraction. If you're reducing a complexity score on existing code, start there. If you're deciding how to structure something, start here.
