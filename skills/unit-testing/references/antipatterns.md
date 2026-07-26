# Test Anti-Patterns

Each of these looks locally reasonable and costs later.

## Testing private methods

**Trigger.** A private method is hard to reach through the public API, and the proposed fix is widening its visibility or invoking it reflectively.

**What goes wrong.** You have exposed an implementation detail purely for a test, reintroducing exactly the coupling the visibility boundary was preventing. Every future change to that method now breaks a test that no caller would have noticed.

**What to do instead.** Exercise the private method indirectly, through the public behavior that uses it. If coverage genuinely cannot be reached that way, there are only two honest explanations: it is dead code, in which case delete it; or it is a missing abstraction, in which case extract it into its own class with its own public surface and test it directly.

**The one legitimate exception.** A private member that is part of a real external contract — a constructor a persistence framework must invoke, for instance — is not being exposed "for testing." It is honoring an obligation that already existed.

**Check.** Strip away the testing justification. Is there any other reason this member is accessible? If not, revert and extract.

## Test-only accessors on state

**Trigger.** A field or property is made public so a test can inspect internal state after acting.

**What goes wrong.** The test now has privileges no real caller has, asserting on state nothing in production reads. That is an unbounded surface for future false positives — every change to the internal representation breaks tests despite the behavior being intact.

**What to do instead.** Assert on state or output that a real caller already consumes. Verify the *effect* the field produces rather than the field.

**Check.** For each test-only accessor, confirm some non-test code also reads it. If none does, delete the accessor and find the observable consequence to assert on instead.

## Leaking domain knowledge into the test

**Trigger.** The test computes its expected value using the same formula as the code under test — anything beyond trivial arithmetic.

**What goes wrong.** The test duplicates the algorithm rather than checking it. It can never disagree with the implementation, so it verifies nothing. Worse, a wrong "fix" pasted into both sides passes cleanly while the bug ships.

**What to do instead.** Hard-code expected outputs derived independently of the implementation — worked by hand, taken from a specification, supplied by a domain expert, or captured from a trusted system being replaced.

**Check.** If you deleted the production implementation entirely, would the test still tell you what the correct answer is? If the expected value is produced by re-running the same logic, it fails this test.

## Code pollution

**Trigger.** Production code contains a conditional, flag, or constructor parameter whose only purpose is to behave differently under test.

**What goes wrong.** Production now carries code that exists solely to serve tests, and — worse — a branch that production traffic never exercises but that can still activate accidentally. The shipped surface area grew for no user-facing reason.

**What to do instead.** Extract the varying behavior behind an interface with two implementations, real and fake, and inject the appropriate one. The production class stops needing to know it is being tested.

**Check.** Search production classes for conditionals or parameters referencing test, mock, fake, or environment. Every hit is a candidate for extraction.

## Partially faking a concrete class

**Trigger.** You want to fake part of a class's behavior while keeping the rest real — typically overriding only the method that reaches an external dependency, on a class that also holds business logic.

**What goes wrong.** Needing a partial fake is a diagnosis, not a technique. It means the class combines two responsibilities that should not cohabit: talking to the outside world, and deciding things. The resulting hybrid needs elaborate setup in every test.

**What to do instead.** Split along the seam the partial fake revealed. One small class performs the out-of-process work behind its own interface; another contains only logic and accepts plain data as input. Test the logic class with real objects; cover the boundary class, if at all, with an integration test.

**Check.** Does the class both call something external *and* branch on business rules? If both, split rather than partially fake.

## Ambient time

**Trigger.** Business logic reads the current clock directly rather than receiving time as an argument.

**What goes wrong.** Time becomes a hidden input — identical calls return different results with no change in signature. Tests go flaky, or get forced onto a shared mutable global clock, which is itself a cross-test interference problem.

**What to do instead.** Capture the time once at the entry point of the operation and pass it through as a plain value.

**Check.** Search business logic outside operation entry points for direct calls to the platform clock. Each one is a hidden dependency. The same reasoning applies to randomness, environment lookups, and any other ambient source of nondeterminism.

## Asserting on queries

**Trigger.** A stub supplies a canned value, and the same test also asserts the stub was called.

**What goes wrong.** The call is a means, not an end. Asserting it over-specifies: the test now fails when the SUT obtains the same correct answer a different way.

**What to do instead.** Delete the interaction assertion. Keep the assertion on the result.

**Check.** Is this assertion checking a value the code produced, or a step it took to produce it? If a step, and it does not cross the system boundary, remove it.

## Mocking your own in-process collaborators

**Trigger.** A test creates a double for a class you own that never itself touches a database, network, or filesystem.

**What goes wrong.** The test becomes welded to the current call graph. Extracting a helper, inlining a collaborator, or reordering calls breaks tests despite behavior being unchanged. These tests also tend to be shallow — they confirm glue code called the right methods while the actual logic was replaced by a canned answer.

**What to do instead.** Use the real instance. Reserve doubles for the outermost unmanaged out-of-process dependency.

**Check.** Count the doubles in the test. Any standing in for a class that never crosses a process boundary is over-mocking.

## Mocking an intermediate wrapper

**Trigger.** An integration test doubles an interface that is itself a pass-through to something closer to the real boundary.

**What goes wrong.** Everything between the double and the actual boundary goes untested, and the test still couples to an internal structural choice — which wrapper happens to exist.

**What to do instead.** Find the last class before the boundary and double *that*. Assert on the literal payload crossing the boundary, not on a call to one of your own methods.

**Check.** Trace from the doubled interface to the real boundary. If another class of yours sits in between unexercised, move the double outward.
