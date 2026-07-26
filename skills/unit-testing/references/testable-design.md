# Designing Code So It Can Be Tested

When testing something is painful, the pain is usually a design signal rather than a testing problem.

## The diagnosis: complexity times collaborators

Score any class on two independent axes:

- **Decision density** — how many branches, rules, and calculations it contains.
- **Collaborator count** — how many dependencies it needs, weighted heavily toward out-of-process ones.

Four quadrants, four verdicts:

| | Few collaborators | Many collaborators |
|---|---|---|
| **High decision density** | Ideal unit-test target. Test exhaustively. | The problem. Split it. |
| **Low decision density** | Trivial. Do not test. | Controller. Cover with a few integration tests only. |

The top-right quadrant is what makes people conclude unit testing "does not scale." Attempts to cover it produce either a wall of doubles or a shallow test that skips the real logic. Neither is worth having.

## The humble object split

Extract the decision-making into a class with **zero out-of-process collaborators**. Leave behind a deliberately dumb wrapper whose only job is gathering inputs and applying outputs — with no branching of its own.

The logic class then takes plain values and returns plain values or a description of what should happen. It is trivially testable with real objects and no doubles at all. The wrapper has so little left to get wrong that a couple of integration tests suffice.

This is the single highest-leverage move in this skill. Most "untestable" code becomes easy once the decisions and the I/O stop sharing a class.

## When inputs can't all be gathered up front

The common objection: the operation needs to consult an external dependency *partway through*, based on what it found earlier. So you cannot read everything first.

The tempting fix — pass the dependency into the logic class — undoes the whole split, because now every test of that logic needs a double again.

Two better options, in order of preference:

1. **Read more up front and decide in one pass.** Fetch what you might need, compute a decision — or a list of decisions and events — as pure data with no side effects, and let the outer layer translate that into actual writes and calls. This trades some efficiency for testability. Often that trade is obviously correct and people skip it only from habit.

2. **Where full separation is genuinely too costly**, keep the *decision about whether an action is permitted* inside the logic class as a precondition, even though the controller performs the I/O. The controller can then no longer bypass the rule, and the rule stays unit-testable in isolation.

**Check.** In the logic-holding class, look for any call to something out-of-process — I/O, network, the clock, a random source. Each one is a candidate to hoist into the caller and pass in as a value.

## Leaking abstractions make tests brittle

If a caller must invoke more than one public member to accomplish a single goal, the class is exposing its internals. Tests then have nothing to assert on but those internals.

The classic shape is a normalize-then-assign pair, where callers are expected to call both in the right order. Collapse it: one operation that enforces the invariant internally. The public surface shrinks, encapsulation improves, and tests gain something behavioral to target.

**Check.** For each public member, ask whether some caller's goal requires it, or whether it only exists to serve an internal step. If the latter, make it private.

## Scoping around behavior, not classes

A unit is a unit of *behavior*, not a unit of code. It may span one method, one class, or a small cluster of collaborating objects — the class count is irrelevant.

Tests scoped per class read like a recitation of internal steps. Tests scoped per behavior read like requirements, and survive refactoring because the requirement did not change.

**Check.** Can you state what the test verifies in one sentence a domain expert would recognize? If stating it requires naming internal classes or call ordering, the test is scoped to code rather than behavior.

## The schools, briefly

Two traditions disagree about what "isolated" means, and the disagreement matters because one of them is the source of most brittleness.

- **The London school** isolates the class under test from all mutable collaborators using doubles. A test is "isolated" when one class runs with everything else faked. This over-produces doubles and welds tests to intra-system call graphs.
- **The Classical school** isolates *tests from each other*, so they can run in any order or in parallel. It substitutes only dependencies shared between tests — nearly always stateful out-of-process ones.

Default to the Classical reading: isolate tests from each other, isolate nothing else unless the dependency-classification procedure calls for it. But note that Classical alone does not settle system boundaries — you still need the managed/unmanaged distinction to decide what gets doubled.

**Check.** Would this test's outcome change if another test ran just before it or concurrently and touched the same storage? If yes, fix isolation at the shared-dependency level — not by mocking unrelated in-process collaborators.
