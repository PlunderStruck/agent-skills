# Integration Tests and Out-of-Process Dependencies

## What each layer is for

Unit tests carry the combinatorial weight. Business-logic edge cases are cheap to cover there, so cover all of them.

Integration tests exist to prove your code and a real dependency actually cooperate. That means **one full happy path per business scenario**, touching every out-of-process dependency involved, plus only those edge cases a unit test genuinely cannot reach.

Skip integration coverage for conditions that would crash the application immediately and visibly. If a misconfiguration prevents startup, the failure is already loud; a test adds little.

**Check.** "Would a unit test on the domain logic alone have caught this, without a real dependency?" If yes, it belongs at the unit layer.

## Managed versus unmanaged

This distinction decides everything about how an out-of-process dependency is treated.

**Managed** — only your application reaches it. Your own database, typically. Nobody else observes its state directly.

→ **Use a real instance.** Mocking a managed dependency throws away the only reason to integration-test at all. A mock proves you invoked an interface correctly, which the unit tests already established more cheaply. It cannot prove the schema matches, the query is valid, the transaction commits, or the mapping round-trips.

**Unmanaged** — other systems observe it or depend on it. An email gateway, a message bus another team consumes, a third-party API, tables a sibling service reads.

→ **Substitute a double**, because its state is part of your externally observable contract and you cannot freely reset or corrupt it. Double it only in integration tests, and only at the last adapter before the boundary.

**Split dependencies exist.** A database where most tables are yours but a few are read by another team is managed in part and unmanaged in part. Treat the shared tables as unmanaged and the rest as managed.

**Check.** For each out-of-process dependency: does any other process or team observe its state directly? Yes → unmanaged, double it. No → managed, use the real thing.

## Don't substitute a different technology

Running tests against a lightweight in-memory stand-in instead of the production engine reintroduces exactly the false confidence you were trying to eliminate. Engine behavior differences — transaction semantics, type coercion, constraint enforcement, ordering — are precisely what slips through.

Use the same technology as production. If that is genuinely impossible in your environment, the honest response is to unit-test the logic and *accept the coverage gap*, rather than manufacture confidence from a different engine.

## Verifying state

Read the result back independently of the values used to set the test up. A test that writes through one path and verifies through the same path can pass while both are wrong.

## Repositories and thin data-access classes

Do not write standalone tests for a repository whose only job is mapping objects to storage calls. It is low-logic, single-collaborator code, and testing it directly pays the full integration cost — real dependency setup and teardown — for coverage the scenario-level integration test already provides by exercising the same path.

If a repository contains genuine logic — nontrivial mapping, validation, conditional query construction — that logic does not belong there. Extract it into a plain class and unit-test it.

**Check.** If a proposed test's arrange/act/assert only calls one repository method and confirms storage reflects it, look for an existing scenario test covering that path. If one exists, the new test is redundant.

## Where to place the double

When a chain of your own classes sits between the caller and the real boundary, double the **last** one before the boundary — not an intermediate wrapper.

Doubling an intermediate leaves everything between it and the boundary untested, and couples the test to which wrapper happens to exist today. Assert on the literal payload that crosses the boundary: the actual message, the actual request body. That is the thing other systems depend on, and therefore the thing worth pinning.

## Pyramid shape

Let the domain complexity of the actual system determine the ratio rather than targeting a fixed percentage.

A system with substantial business rules should be unit-test heavy, with integration tests covering happy paths through real dependencies. A mostly-CRUD system with little logic outside orchestration legitimately inverts this — forcing a large unit-test layer there produces trivial tests written to hit a number.

Integration tests outnumbering unit tests in a genuinely complex domain is a warning sign: it usually means logic is tangled with I/O and nobody could test it any other way. That is a design problem showing up as a testing ratio, and the fix is the humble object split rather than more integration tests.
