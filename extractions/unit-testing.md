# Extraction: unit-testing

**Source:** Unit Testing: Principles, Practices, and Patterns — Vladimir Khorikov

> Operational rules distilled from the source, written in our own words. This is the intermediate layer between the source text and the published skill — denser than the skill, and the place detail survives for re-distillation.

---

# Unit Testing: Operational Rules for Coding Agents

Distilled from *Unit Testing: Principles, Practices, and Patterns* (Khorikov), generalized away from any specific language or framework. Organized so each rule is checkable against a diff.

## Foundations: what a test is actually worth

Every test scores on four independent axes: **protection against regressions** (does it catch real bugs — a function of how much important code it exercises), **resistance to refactoring** (does it stay green when behavior is preserved but implementation changes), **fast feedback**, and **maintainability**. A test's value is the *product* of these, not the sum — zero on any one axis makes the test worthless. A test that runs instantly but breaks on every harmless rename is not a cheap asset; it's a liability with a low up-front cost.

The first three axes can't be maximized together (CAP-theorem-like trade-off), but resistance to refactoring is different: it's close to binary — a test either survives refactoring or it doesn't. Because of the multiplication rule, near-zero resistance to refactoring makes a test near-worthless regardless of speed or theoretical bug-catching power. **Treat resistance to refactoring as non-negotiable; trade off only between protection against regressions and speed.** False positives (failures with no real bug) are worse than false negatives over a project's life, because they erode trust until people stop reading failures at all.

The lever that controls resistance to refactoring is whether a test targets **observable behavior** or **implementation details**. That distinction — not "one class per test" — drives every rule below.

## Observable behavior vs. implementation details

**Trigger**: Deciding what a test should assert on, or reviewing whether a class's public surface is bigger than what its callers actually use.

**What goes wrong**: Code is observable behavior only if it exposes an operation or state that helps some caller reach a goal; everything else is an implementation detail. When a caller needs to invoke *more than one* public member to accomplish one goal, the class is leaking implementation details — and tests, having nothing else to assert on, couple to those details. This also breaks encapsulation, since callers can now skip or reorder the leaked steps.

**What to do instead**: Collapse multi-step public sequences into one operation that enforces the invariant internally (e.g., a setter that normalizes, not a normalize-then-assign pair). If one caller goal needs more than one public call, an abstraction is leaking.

**How to check**: For any public member, ask "does a caller's goal require this, or does it only serve an internal step?" If the latter, make it private — tests then have nothing to couple to but behavior.

---

**Trigger**: Deciding, in an integration-style test, whether a given call — to another class, a database, a queue, another service — is something to verify.

**What goes wrong**: Communication *inside* your system is always an implementation detail, even when it's the mechanism behind a real outcome. Communication that crosses your system's boundary to a party you don't control is part of your observable contract, since others depend on it. Verifying intra-system calls the same way as inter-system ones is the largest single source of brittle tests.

**What to do instead**: Before asserting on any interaction, classify it: does it stay inside the process and get consumed only by your own code, or does it produce an externally visible effect? Only the latter belongs in an assertion.

**How to check**: "If I restructured the internal collaborators but kept the final output/side effect identical, would this assertion still pass?" If no, it's coupled to an implementation detail.

## Mocks vs. stubs: the decision procedure

A **mock** substitutes and lets you examine an *outgoing* interaction (a command — produces a side effect). A **stub** substitutes an *incoming* interaction (a query — supplies input data, no side effect). This maps to command-query separation: commands get mocks, queries get stubs.

Given a dependency, resolve it in order:

1. **Plain value / immutable object** (number, string, enum, value object) → use the real thing always; nothing to fake.
2. **In-process and mutable** (another domain object, an in-memory collaborator) → use the real instance. Never substitute domain/business collaborators with a test double — that's what causes London-style over-mocking.
3. **Out-of-process** (database, filesystem, network, queue) → classify further: **managed** (only your app talks to it — your own database) vs. **unmanaged** (other systems observe or depend on it — an email gateway, a message bus, a third-party API, tables shared with another team).
   - **Managed** → use a real instance in an integration test. Never mock it; a mock only proves you called an interface correctly, not that the integration works.
   - **Unmanaged** → replace with a test double, only in integration tests, only at the last adapter class before the boundary (see "verify at the edges" below).
4. **Within an unmanaged dependency**, is the call a command or query? Command → mock; assert it happened the exact expected number of times, no more, no less. Query → stub a canned value; never assert that the query was called.

**Trigger**: A stub's canned return value has a "was this called" assertion attached in the same test.

**What goes wrong**: Verifying a query happened is over-specification — the call is a means, not the end. It ties the test to *how* the SUT got a result rather than *what* the result was.

**What to do instead**: Delete the interaction assertion; assert only the actual result.

**How to check**: "Is this checking a value the SUT produced, or a step it took to produce it?" If a step, and it doesn't cross the system boundary, remove the assertion.

## Over-mocking and why it produces brittle tests

**Trigger**: A test creates a mock/fake for a class you own that lives entirely in-process (never itself talks to a database/network/filesystem).

**What goes wrong**: This is the London-school default — isolate every class from every mutable collaborator. It couples tests to the exact call graph the SUT currently uses; any internal refactor (extract a helper, inline a collaborator, reorder calls) breaks tests that never should have known those details existed. Over-mocked tests also tend to be shallow, verifying only that glue code called the right methods rather than that the logic — replaced by a canned mock response — is correct.

**What to do instead**: Use real, in-memory instances of in-process collaborators. Reserve mocking for the outermost, unmanaged, out-of-process dependency only.

**How to check**: Count the mocks in a test. If any stands in for a class that never itself crosses a process boundary, that's over-mocking — replace it with the real object.

---

**Trigger**: An integration test mocks an interface that is itself a thin pass-through to another interface (a multi-hop chain between the controller and the real external boundary).

**What goes wrong**: Mocking an intermediate wrapper under-tests the code between it and the real boundary (weaker protection against regressions), and still couples to an internal detail (which wrapper exists) rather than what actually crosses the boundary.

**What to do instead**: Find the *last* class before the boundary and mock (or hand-write a spy for) that one. Assert on the literal payload/message that crosses the boundary, not a call to your own intermediate method.

**How to check**: Trace from the mocked interface to the real boundary. If another of your own classes sits in between and isn't exercised, move the mock outward.

## What "a unit" actually means

**Trigger**: Deciding test granularity — one test file per class, or something else.

**What goes wrong**: Treating "a unit" as "a class" drives the isolate-every-collaborator instinct above, and produces tests that read like a list of internal steps ("leg moves, then leg moves, then head turns") instead of an outcome ("the dog comes when called"). Such tests can't be traced back to something a domain expert would recognize as a requirement — itself a symptom of coupling to implementation.

**What to do instead**: A unit is a unit of *behavior*, not code. It may span one method, one class, or a small cluster of collaborators — the class count is irrelevant. Size the test around a behavior meaningful to the problem domain.

**How to check**: Can you state what the test verifies in one sentence a domain expert would recognize? If it requires naming internal classes or call order, the test is scoped to code, not behavior.

## London vs. Classical schools — what each gets wrong

Both schools want isolation; they disagree on what to isolate from what. **London** isolates the class under test from all its mutable collaborators via mocks, calling a test "isolated" when one class runs with everything else faked. This over-produces mocks and couples tests to intra-system call graphs (see over-mocking above). **Classical** isolates *tests from each other* (so they run in parallel/any order), substituting only dependencies *shared across tests* — almost always out-of-process, stateful ones like a database. This document defaults to Classical, but Classical alone doesn't automatically get inter-system boundaries right either — you still need the managed/unmanaged distinction above.

**Trigger**: No explicit test-style guidance exists in a codebase.

**What goes wrong**: Defaulting to "mock everything but immutable values" (London) silently imports brittleness into every new test. Defaulting to "share one big fixture across everything" (a naive reading of Classical) reintroduces cross-test interference.

**What to do instead**: Isolate tests from each other (never share mutable out-of-process state between runs); isolate nothing else by default. Reach for a test double only when the dependency-classification procedure above says to.

**How to check**: "Would this test's outcome change if another test ran concurrently or just before it and touched the same storage?" If yes, fix isolation at the shared-dependency level, not by mocking unrelated in-process collaborators.

## Anti-patterns

**Trigger**: A private method is hard to exercise through the public API, and the fix under consideration is loosening its visibility, or invoking it via reflection.

**What goes wrong**: This exposes an implementation detail solely for a test, reintroducing the brittleness the visibility boundary was protecting against.

**What to do instead**: Test the private method only indirectly, through the public behavior using it. If coverage genuinely can't be reached that way, it's either dead code (delete it) or a missing abstraction (extract it into its own directly-testable class). The rare legitimate exception: a private member that is itself part of a real external contract (e.g., a constructor a persistence framework must call) — exposing that isn't "for testing," it's honoring an existing contract.

**How to check**: If you removed the "for testing" justification, is there any other reason this member is accessible? If not, revert and extract instead.

---

**Trigger**: A field/property is made public only so a test can inspect internal state after an operation.

**What goes wrong**: The test gains privileges real callers never have, asserting on state no production caller looks at — an unbounded surface for future false positives as that representation evolves.

**What to do instead**: Assert only on state or output a real caller already consumes. Verify the *effect* the field is meant to produce, not the field itself.

**How to check**: For each test-only accessor, confirm at least one piece of non-test code also reads it. If none does, delete the accessor and test the actual consumed effect.

---

**Trigger**: A test computes its expected value with the same formula as the code under test (e.g., `expected = value1 + value2` mirroring production), for anything beyond trivial arithmetic.

**What goes wrong**: This duplicates the algorithm instead of checking it — the test can never disagree with the implementation, and a "fix" pasted into both sides lets a real regression through.

**What to do instead**: Hard-code expected outputs computed independently of the SUT — by hand, from a domain expert, from a spec, or from a trusted implementation being replaced.

**How to check**: Could you delete the production implementation and still know, from the test alone, what the correct output should be? If the expected value is derived by re-running SUT-like logic, it fails.

---

**Trigger**: Production code has a conditional, flag, or constructor parameter whose only purpose is to change behavior when running under tests.

**What goes wrong**: This is code pollution — production code that exists only to serve tests, increasing the shipped code's surface area and creating a path production traffic never exercises but that can still silently activate.

**What to do instead**: Extract the varying behavior behind an interface with two implementations — real and test-only fake — and inject the one needed. The production class no longer needs to know it's under test.

**How to check**: Grep production classes for conditionals/parameters referencing "test," "mock," "fake," or "environment." Any hit is a candidate for extraction into a swappable implementation.

---

**Trigger**: Wanting to fake part of a concrete class's behavior while keeping the rest real (e.g., override just the method reaching an external dependency, on a class that also holds real logic).

**What goes wrong**: Needing a partial fake means the class combines two responsibilities — I/O and logic — that shouldn't cohabit. The resulting hybrid's test setup is a maintenance burden.

**What to do instead**: Split along the two responsibilities: a small class does the out-of-process I/O behind its own interface; a second class does only the logic, taking plain data (not the I/O type) as input. Test the logic class with real instances; test the boundary class, if at all, in an integration test.

**How to check**: Does the class under test both talk to an external dependency *and* branch on business rules? If both, it's a Single-Responsibility violation — split it rather than partially fake it.

---

**Trigger**: Business logic reads the current wall-clock time directly rather than receiving it as an argument.

**What goes wrong**: Ambient time is a hidden input — identical calls at different moments return different results with no signature change. Tests become flaky, or forced onto a shared swappable global clock, which is itself a shared mutable dependency that reintroduces cross-test interference.

**What to do instead**: Pass time in explicitly, as a plain value, at the point a business operation begins, and thread that value through the rest of the operation.

**How to check**: Grep domain/business-logic code (outside an operation's entry point) for direct calls to the platform's current-time API. Any hit is a hidden dependency.

## Integration testing strategy and the test pyramid

**Trigger**: Deciding whether a scenario needs a unit test, an integration test, both, or neither.

**What goes wrong**: Treating the two as interchangeable, or integration-testing every edge case, makes the expensive slow layer duplicate the cheap layer's coverage without adding protection.

**What to do instead**: Unit tests carry the combinatorial weight — cover every business-logic edge case there, since it's cheap. Integration tests cover exactly one full happy path per business scenario (touching every out-of-process dependency involved) plus only edge cases a unit test genuinely can't reach. Skip integration coverage for conditions that would crash the app immediately and visibly if broken (fail-fast conditions) — the crash is already loud, so the test adds little.

**How to check**: "Would a unit test on the domain logic alone have caught this, without a real dependency?" If yes, it belongs at the unit layer. Also check the ratio: integration tests outnumbering unit tests in a system with real business-rule complexity signals an inverted pyramid and wasted maintenance cost. (Simple, mostly-CRUD systems are a legitimate exception — there a roughly equal or integration-heavy split is correct.)

**How to check (pyramid shape)**: Let the domain complexity of the actual system dictate the unit/integration ratio rather than targeting a fixed percentage. Look at how much logic lives outside orchestration code; if it's minimal, don't force a large unit-test layer just to hit a shape — it produces trivial, low-value tests.

## The Humble Object pattern and functional architecture

**Trigger**: A class is both logic-heavy (real branching/business rules) *and* collaborator-heavy (especially out-of-process ones), so testing it well needs either a huge arrange section or a wall of mocks.

**What goes wrong**: This "overcomplicated" combination is what makes unit testing look like it doesn't scale — coverage attempts produce either brittle mock-laden tests or thin ones that skip real logic.

**What to do instead**: Split along the Humble Object line: extract all decision-making into a class with zero out-of-process collaborators (the algorithm/domain side); leave a thin, deliberately dumb wrapper responsible only for gathering inputs and applying outputs, with no branching of its own. Test the algorithm side exhaustively with plain values; cover the thin wrapper, if at all, with a few integration tests, since it now has almost no logic left to get wrong.

**How to check**: Score branching points and collaborator count separately. High on both → split. High complexity/low collaborators → good unit-test candidate as-is. Low complexity/high collaborators → acceptable as an untested-in-detail controller, covered only by integration tests. Low on both → trivial, don't test it.

---

**Trigger**: A business operation needs to decide something partway through based on data from an out-of-process dependency, so inputs can't all be gathered up front.

**What goes wrong**: The tempting fix — pass the out-of-process dependency directly into the logic-holding class — reintroduces collaborators into exactly the code being kept clean, so every test of that logic needs a fake again.

**What to do instead**: Where feasible, read everything up front, compute a decision (or list of decisions/events) as plain data with no side effects, and let a thin outer layer convert that decision into actual writes/calls — trading some performance for testable logic. Where full separation isn't affordable, at minimum keep the decision of *whether an action is allowed* inside the logic-holding class as a precondition, so the controller can't bypass it even though it still triggers the I/O.

**How to check**: In the logic-holding class, look for any call to something out-of-process (I/O, network, current time, random) that isn't a passed-in plain value. Each one is a candidate to hoist to the caller.

## Database and out-of-process dependency testing

**Trigger**: Writing an integration test that touches a real datastore, or deciding whether to fake the datastore instead.

**What goes wrong**: Faking a dependency you fully control (your own database, unread by other systems) throws away the reason to integration-test at all — proving code and dependency cooperate — leaving only proof that an interface method was called, which unit tests already cover more cheaply. Conversely, substituting a different storage engine than production uses (e.g., a lightweight in-memory stand-in) reintroduces false confidence, since engine-behavior differences are exactly what slips through.

**What to do instead**: For a dependency only your app can reach, run tests against a real instance of the same technology as production; verify final state by reading it back independently of the values used to set up the test. For a dependency other systems can also observe (or a partly-shared datastore), split it: treat the shared/observable part as unmanaged and mock it, and the rest as managed and hit it for real.

**How to check**: Two questions per out-of-process dependency: (1) "does any other process/team observe this dependency's state directly?" — yes → unmanaged (mock); no → managed (use the real thing). (2) If a real managed dependency truly can't be exercised anywhere, don't substitute a different technology — drop to unit-testing the logic only and accept the coverage gap rather than fake confidence.

---

**Trigger**: Considering a dedicated test for a thin data-access/repository class whose only job is mapping domain objects to and from storage calls.

**What goes wrong**: A repository is low-logic, one-collaborator code — testing it standalone pays full integration-test cost (real dependency setup/teardown) for protection already covered by the broader scenario-level integration test that exercises the same call path.

**What to do instead**: Don't write standalone repository tests; let scenario-level integration tests exercise them. If a repository contains real logic (nontrivial mapping/validation), extract that logic into its own plain class and unit-test it directly instead.

**How to check**: If a proposed test's arrange/act/assert only calls one repository method and checks storage reflects it, check whether an existing scenario-level integration test already covers that path. If so, it's redundant.
