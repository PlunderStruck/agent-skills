# Signifiers and Mapping

Everything here narrows the **Gulf of Execution** — the gap between what someone wants to do and the actions your interface makes discoverable. If the user can't work out what's possible or which control does what, no amount of feedback afterward helps.

## The affordance exists; the signifier is what's missing

**Trigger.** An icon-only button, a swipeable card, a draggable handle, a long-press action, a gesture, or any interaction a user is expected to discover.

**What goes wrong.** These two get used interchangeably and they are not the same thing, which matters because it sends you to fix the wrong half.

An **affordance** is a relationship between what an object is and what a particular person can do with it — not a property of the object alone. A chair affords sitting for an adult and not for a child too weak to move it; the chair hasn't changed. Affordances can be entirely real and entirely invisible, which is why birds fly into glass.

A **signifier** is a perceivable indicator of where and how to act. It can be deliberate (a label, an arrow) or accidental (a worn path across grass, a pile of cups saying "put things here").

In software the affordance is nearly always already wired up — the whole screen affords tapping, the element accepts the drag. What's absent is any signal that this particular action is available *here*. Calling that missing signal a "missing affordance" points the fix at the event handler, which already works.

Signifiers can also lie. A control styled to look draggable that isn't, or a decorative element that reads as a handle, is a signifier making a false promise — worse than none, because it produces confident wrong actions rather than hesitation.

**What to do.** For each interactive element, ask what a first-time user perceives that tells them the action exists. If the honest answer is "they'd have to try it," "there's a tooltip on hover," or "it's in the release notes," there is no signifier.

**Check.** Could someone who has never used this, and cannot hover, discover this action from what is on screen? Hover doesn't count — it doesn't exist on touch, and it requires already suspecting something is there.

## A mapping is arbitrary unless the arrangement itself tells you

**Trigger.** Any layout where a set of controls corresponds to a set of things controlled — a settings panel, a toolbar, a grid of actions, a form whose fields map to entities.

**What goes wrong.** A mapping is **natural** when spatial or perceptual analogy carries the relationship without learning: a control mounted on the thing it affects, or arranged in the same configuration as the things it controls. Ranked best to worst — on the object, adjacent to the object, arranged in the same spatial pattern as the objects.

It's **arbitrary** when none of those hold and the only route is memorization or a label. The canonical case is a rectangle of stove burners driven by a row of knobs: several different pairings all ship commercially, none derivable from the layout, so every unfamiliar stove is re-learned from scratch.

The software equivalents are everywhere — a settings list grouped by internal architecture rather than by which options actually interact, a drag target far from what it affects, tab order that follows DOM order rather than reading order. Each costs a lookup on every use, however logical it looked to whoever built it.

**What to do.** Arrange controls so the arrangement itself encodes the relationship. Where that's impossible, accept that you now owe a label and a learning cost, and count it rather than assuming the grouping is self-evident.

**Check.** Strip the labels mentally. Does the layout still tell you which control affects what? If not, the mapping is arbitrary — which is sometimes unavoidable, but should be a decision rather than an accident.

## Put basic operation in the world, without taxing the expert

**Trigger.** Deciding how much to show by default — a toolbar, a command surface, a form with optional advanced fields.

**What goes wrong.** Behaviour draws on knowledge split between the head (learned convention, practised skill) and the world (visible labels, structure, constraints). Each has a real cost. Knowledge in the world needs no prior learning but must be searched and interpreted at the moment of use, and it clutters. Knowledge in the head is effortless once acquired but requires the acquisition, and stays unreliable unless something visible triggers recall.

The failure is picking one globally. An interface that puts everything in the world is unusable at speed; one that assumes everything is in the head is unlearnable.

**What to do.** Put what's needed for *basic* operation into the world, so an infrequent user needs to have memorized nothing. Then provide a path that doesn't force an expert back through that same friction — a shortcut, a command palette, a collapsed advanced section.

**Check.** Can someone returning after three months complete the core task with nothing memorized? And can a daily user complete it without walking the same discovery path every time? Both, or you've optimised for one population.

## Standardization is a last resort and a first resort

**Trigger.** Inventing an interaction pattern, or choosing between an established platform convention and something better-fitting.

**What goes wrong.** Standardization is explicitly the fallback for when no natural mapping is achievable — a single-lever mixer tap has no natural spatial mapping for two variables, so a learned convention is all that's left. That framing gets read as "standardization is weak," which inverts badly in practice.

The direction that matters: it's a **last** resort when you're designing a genuinely new interaction, because a convention everyone must learn once is worse than a mapping nobody has to learn at all. It's a **first** resort when a real platform convention already exists, because inventing a novel gesture where an established one exists optimises the wrong axis entirely — you've spent the user's learning budget to avoid a solved problem.

**What to do.** Ask whether a convention exists before designing. If it does, use it. If it doesn't, try for a natural mapping first, and only fall back to inventing-and-teaching a convention when that fails.

**Check.** Is there an established platform or ecosystem pattern for this? If yes and you've diverged, what does the divergence buy that's worth the relearning?
