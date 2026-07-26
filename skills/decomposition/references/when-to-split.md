# Signals A Chunk Is Two Ideas

Concrete tells that a piece of code is holding more than one concept. Each is a prompt to look, not an instruction to extract — check against [when-not-to-split](when-not-to-split.md) before acting.

## A flag or mode parameter that selects behaviour

**Trigger.** A function takes a boolean or mode and branches internally on it to decide which of two things to do.

**What's wrong.** You have asked the running program to make a decision you already knew the answer to at the call site. And the flag's meaning rarely matches how anyone thinks about the problem — call sites read as `doThing(true)` and `doThing(false)`, which communicates nothing.

**What to do.** Split into two named operations, one per behaviour, and factor out only what is genuinely common between them.

**Check.** Do the call sites read as sentences, or as `true, x` / `false, x`?

## Nested prerequisite checks

**Trigger.** Validation nested several levels deep, so a reader has to count indentation to work out when the actual work happens.

**What to do.** Give each check its own named unit with its own local failure handling, so the top level reads as a flat sequence of named steps rather than a funnel.

## A chain of if/else-if on one discriminant

**Trigger.** `if button A … else if button B … else if button C …`

**What's wrong.** Two costs. The program pays a sequence of comparisons on every call even though the input already determines the outcome. And every new case means editing existing code rather than adding something alongside it.

**What to do.** Replace sequential testing with routing keyed directly on the discriminant — a lookup rather than a chain.

**Check.** Does adding a case mean *inserting a branch into existing code*, or *adding one independent entry*? The second means dispatch is factored correctly.

## You need a comment to say what a variable still means

**Trigger.** Partway through a function you feel the need to remind yourself — in a comment — what a value currently represents.

**What's wrong.** That's complexity outrunning what a reader can hold at once. Roughly seven items is the practical ceiling for one unit, and the comment is you noticing you've passed it.

**What to do.** Factor at exactly that point. The extracted piece's name replaces the comment.

**Check.** Could you now delete the explanatory comment, because what it explained is a named call rather than inline reasoning?

## A loop inside a loop

Where the inner loop does something independently meaningful, that inner thing usually wants a name. Same working-memory argument.

## Names that are concatenations or carry numbers

`enable-left-motor` and `disable-right-solenoid` are a cross-product of two independent concepts — an action and a target — glued into one identifier per combination. `channel1`, `channel2`, `channel3` is data smuggled into an identifier instead of passed as an argument.

Both look like naming problems. Both are decomposition problems: **the rate at which you need new names as you add variants is evidence about the boundary.** A factoring that lets combinations compose beats one that enumerates them.

## Conditionals as a design diagnostic

Treat every conditional as a question rather than a neutral tool. The default assumption should be that most apparent decisions are not genuine branches but **dispatch** — the input already determines the outcome, and the honest implementation routes to it rather than testing for it.

**Check when writing an `if`.** Does this encode two genuinely different processes, or does it select between two values, or two names for the same process? The latter rarely needs a branch.

### Branches that differ only in which value comes out

If both arms produce a different value rather than doing different work, derive the result from the condition instead of branching — a boolean is already a 0/1, and a small lookup or direct calculation replaces the branch.

**Two cautions, and they matter as much as the technique.** This is legitimate only when it leans on an assumption that is genuinely stable and either well known or documented at the point of use; applied to something likely to change, or left unexplained, it becomes a trap for the next reader. And when the underlying decision is genuinely trivial, the clever arithmetic version sometimes isn't worth it even though it's shorter — a plain conditional that matches how a reader thinks about the problem can beat a denser one that doesn't.

### Hand-tuned special cases around one model

Three regimes with empirically adjusted transition thresholds, each with its own conditional and its own accumulated edge-case fixes, is a symptom rather than a set of individual bugs. Every patch treats a symptom.

The higher-leverage question: does a single reformulation remove the need for discrete regimes at all?

### Guards against states the caller already prevents

If you can point to the exact upstream place that already makes a condition impossible, a downstream re-check of the same condition is defensive weight rather than robustness — and it dilutes the signal of the checks that do matter.

Push the guarantee to where it's actually established and trust it below that point. More aggressively: look for a representation in which the special case becomes an ordinary case needing no check at all.
