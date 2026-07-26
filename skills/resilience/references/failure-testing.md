# Testing for Failure

Applies to integration tests and test doubles for any external dependency.

## Why passing integration tests prove so little

Integration tests exercise a dependency's **documented contract**. Even when they force an error, that error is an in-spec response: a 500, a validation rejection, a well-formed failure the client library knows how to parse.

Production failures happen *below* the contract, at the transport layer, where no interface documentation applies:

- The connection is accepted and nothing ever comes back.
- Bytes arrive one at a time, minutes apart, keeping the socket technically alive.
- The response is well-formed HTTP containing an ISP's captive-portal page instead of your API's JSON.
- The connection is refused, or accepted then reset mid-body.
- The TLS handshake stalls.

A mock cannot produce any of these, because a mock lives inside the interface. It answers the question "does my code call this correctly," not "does my code survive this dependency behaving pathologically."

That gap is why systems pass every test and then hang on their first real incident.

## The hostile harness

Build a deliberately antisocial test double per dependency *type*, operating at the socket level rather than the client-library level. It should be able to:

- Accept a connection and never respond.
- Respond after a long delay — longer than your configured timeout, to prove the timeout fires.
- Dribble the response a byte at a time.
- Send garbage, truncated, or wrong-content-type payloads.
- Refuse connections outright.
- Close the connection mid-response.

Then point your client at it and confirm the application **degrades** rather than hanging: the timeout fires, the breaker trips, the fallback path runs, the thread is released.

Two details that matter:

- **Run it under concurrent load.** Many of these defects only surface when many callers are blocked simultaneously — that is when pool exhaustion becomes visible. A single-threaded test against a hanging stub proves far less.
- **Reuse the harness across dependencies.** The failure modes are properties of the network, not of any particular API, so one well-built harness serves every HTTP dependency you have.

This is the only practical way to find hangs inside vendor client libraries before production does, since you cannot inspect or fix their internal blocking behavior.

## What good looks like

For each network dependency, you should be able to answer:

- What happens when it never responds? (Expected: timeout fires, thread released, breaker counts a failure.)
- What happens when it responds slowly but within timeout? (Expected: works, but check whether the caller's own timeout is now exceeded.)
- What happens when it returns something unparseable? (Expected: handled as a failure, not an unhandled exception escaping the request boundary.)
- What happens when it fails repeatedly? (Expected: breaker opens, calls stop reaching the network, degraded path runs.)
- What happens when it recovers? (Expected: half-open trial succeeds, breaker closes, traffic resumes.)

If any of those answers is "we have not tried," that is the test to write next.

## Verifying containment, not just handling

Beyond single-call behavior, the containment properties need testing too:

- **Pool exhaustion.** Saturate a dependency's pool and confirm callers of *other* dependencies still work. If they do not, the bulkhead is missing or misconfigured.
- **Load beyond peak.** Test at multiples of expected peak against the most expensive transaction, not an average click path. The expensive transaction is what a spike will find.
- **Traffic without cookies.** Generate requests that never send a session identifier and hammer a single URL. Realistic click-path load scripts will not do this, and commercial load-test tooling will not generate it by default — but bots do it constantly.
- **Fan-out at real instance counts.** Architectures with point-to-point communication or a shared coordinator behave completely differently at fifty nodes than at two. Testing at development scale proves nothing about production scale.

## What failure testing is also for

Two things this practice buys that are easy to miss, because neither shows up as a passing test.

**It calibrates people, not just systems.** An environment sanitised so thoroughly that engineers rarely see anything genuinely break produces operators who can follow a runbook and have no felt sense of how close the system runs to its edge — so their first real incident is also their first real judgement call. Gamedays and fault injection build that judgement before it's needed. Treat exposure to controlled failure as ongoing operating cost rather than onboarding.

**Every fix opens a new observation window.** Machinery adopted specifically to eliminate a known failure mode introduces different ones — typically rarer, larger, and slower to surface, because they need volume and time to appear. Risk gets declared reduced at cutover, which is exactly when the new failure surface is least understood. Hold heightened scrutiny for a defined period after any migration that "eliminates" a failure class, rather than closing the book when the old symptom stops.

Related, and worth weighing before you add another layer: **each protective mechanism is itself a component that can fail, be bypassed, or interact badly with something else.** A breaker, a bulkhead, an approval gate — each removes some risk and adds some coupling. That trade is usually worth it; it is not automatically worth it, and a system defended by a dozen interacting mechanisms has its own emergent behaviour to reason about.

## What to check in the code

- Is there a hostile double per dependency type, or only in-contract mocks?
- Do timeout tests actually verify the timeout fires, or just that a happy path works?
- Is breaker state transition covered — open, half-open, close — or only the closed path?
- Are failure tests run concurrently, or single-threaded?
- Has anyone tested what happens when a dependency returns valid HTTP with the wrong body?
