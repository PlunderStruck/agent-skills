---
name: unit-testing
description: Decide what to test and how when writing or reviewing tests — choosing between a real object, a stub, and a mock; scoping a test around behavior instead of a class; avoiding tests that break on harmless refactors; and splitting logic from I/O so it can be tested at all. Use when writing tests, reviewing a test file, deciding whether to mock a dependency, fixing brittle or flaky tests, or judging whether a test is worth keeping.
---

# Unit Testing

## Purpose

Most bad test suites are not under-tested — they are tested wrong. They break constantly on refactors that changed no behavior, they assert that a method was called rather than that anything correct happened, and after enough false alarms nobody reads the failures. The cost lands months after the tests are written, which is why the mistakes survive review.

This skill is about the decisions that determine whether a test is an asset or a liability.

## The one idea everything follows from

A test is valuable only if it targets **observable behavior** rather than **implementation details**.

Observable behavior is an operation or piece of state that helps a caller achieve a goal. Everything else is an implementation detail — including internal collaborators, call order, private helpers, and intermediate state.

The diagnostic: **if a caller must invoke more than one public member to accomplish one goal, the abstraction is leaking.** Tests then have nothing to assert on except the leaked steps, and become welded to the current implementation.

## Four axes, multiplied

Every test scores on protection against regressions, resistance to refactoring, fast feedback, and maintainability. The value is the **product**, not the sum — a zero anywhere makes the test worthless.

Speed and thoroughness genuinely trade off. Resistance to refactoring does not: it is effectively binary, and a test that lacks it is near-worthless no matter how fast it runs. **Treat it as non-negotiable and trade only between coverage and speed.**

False positives are worse than false negatives over a project's life. A missed bug costs once; a test that cries wolf costs every run until people stop reading failures entirely.

## The decision procedure: real, stub, or mock

Resolve every dependency in this order. Most bad tests come from skipping to step 4.

1. **Immutable value** — number, string, enum, value object → **use the real thing.** Nothing to fake.
2. **In-process and mutable** — another domain object, an in-memory collaborator → **use the real instance.** Never substitute your own business collaborators. This is the single biggest source of brittleness.
3. **Out-of-process** — database, filesystem, network, queue → classify it:
   - **Managed** (only your application touches it — your own database): **use a real instance in an integration test.** Mocking it proves you called an interface correctly, which is not the thing you needed to know.
   - **Unmanaged** (other systems observe it — email gateway, message bus, third-party API, tables another team reads): **substitute a double**, only in integration tests, and only at the last adapter before the boundary.
4. **Within an unmanaged dependency**, split by command/query:
   - **Command** (causes a side effect) → **mock**, and assert it happened exactly the expected number of times.
   - **Query** (returns data, no side effect) → **stub** a canned value, and *never* assert that it was called.

The load-bearing distinction: **communication inside your system is always an implementation detail; communication crossing your system boundary is part of your contract.** Assert only on the latter.

## Fast checks when reviewing a test

- **Count the mocks.** Any mock standing in for a class that never itself crosses a process boundary is over-mocking. Replace with the real object.
- **Look for asserted queries.** A stub with a "was this called" assertion is over-specification. Delete the interaction assertion; keep the result assertion.
- **Ask the deletion question.** If you deleted the production implementation, would the test still tell you what the right answer is? If the expected value is computed with the same formula as the code, the test can never disagree with the implementation.
- **Ask the refactor question.** If internal collaborators were restructured but the output stayed identical, would this still pass? If no, it is coupled to a detail.
- **Ask the one-sentence question.** Can you state what the test verifies in language a domain expert would recognize? If it requires naming internal classes or call order, it is scoped to code rather than behavior.

## References

| Topic | File |
|---|---|
| Anti-patterns: private methods, test-only accessors, leaked domain knowledge, code pollution, time as a hidden input | [antipatterns](references/antipatterns.md) |
| Making untestable code testable: humble object, splitting logic from I/O, what to do when inputs can't be gathered up front | [testable-design](references/testable-design.md) |
| Integration tests, the pyramid, database and out-of-process testing | [integration-testing](references/integration-testing.md) |

## What this skill will not settle

How much to test. Coverage targets are a management question, not a technical one, and a percentage goal reliably produces trivial tests written to hit the number. This skill tells you whether a given test earns its keep; it does not tell you how many you owe.
