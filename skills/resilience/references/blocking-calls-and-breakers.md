# Blocking Calls, Timeouts, and Circuit Breakers

Applies to every call that leaves the process: HTTP, RPC, database, cache, queue, third-party SDK, and any pool checkout or lock acquisition.

## Why the default is dangerous

Client libraries overwhelmingly default to waiting indefinitely. A call that looks like an ordinary function invocation can block a thread forever, and the code gives no visual hint that it might. Remote-object protocols are the worst offenders — they were designed to make a network call look local, which is exactly the illusion that gets people killed here.

The failure that matters is not "the dependency returned an error." It is "the dependency accepted the connection and went quiet." Errors free your thread. Silence does not.

## Timeouts

Set them explicitly, everywhere, rather than trusting defaults:

- **Connect timeout** and **read timeout** on every HTTP/socket client — these are separate settings and libraries frequently default only one of them.
- **Query timeout** at the database driver level, not just a statement-level hint.
- **Bounded pool checkout.** A pool that blocks indefinitely when exhausted converts a slow dependency into a stalled application.
- **Bounded lock acquisition and queue polling.** Use the timed variants of wait, poll, and lock acquisition; never the no-argument forms.

Choosing the value: the timeout must be shorter than the caller's own timeout, or the caller gives up first and your work is wasted. Timeouts should tighten as you go down the stack, not loosen.

On expiry, decide deliberately what happens next. A timeout with no defined fallback is only half a fix — the thread is freed, but the request still fails in whatever way the surrounding code happens to handle.

## Circuit breakers

A breaker converts a slow failure into a fast one, which is the whole point. It also gives the failing dependency room to recover instead of being hammered while it is down.

Three states:

- **Closed** — calls pass through; failures are counted.
- **Open** — the failure threshold was crossed. Calls fail immediately without touching the network. This is what stops the cascade.
- **Half-open** — after a cooldown, one trial call is allowed. Success closes the breaker; failure reopens it and restarts the cooldown.

Implementation notes that matter:

- **Count the right things.** Timeouts and connection failures should trip the breaker. Business-level errors (a 404, a validation rejection) usually should not — the dependency is healthy, it just said no.
- **Scope the breaker per dependency**, not per application. One breaker for everything means a failing report service stops your checkout path.
- **Log every state transition and count it separately from ordinary failures.** A breaker that trips silently is invisible; you lose both the incident signal and the near-miss data.
- **Decide the open-state behavior.** Failing fast is the default, but degrading — cached data, a reduced feature, a queued write — is often better and is where most of the user-visible value lives.

## Fail fast

The complement to a breaker on the caller's side: before starting expensive work, check whether it can plausibly succeed. Is the breaker open? Is the pool healthy? Are the required parameters valid?

Rejecting immediately beats getting halfway through a transaction and then discovering the dependency is gone. Partial work is more expensive to unwind than work never started, and it holds resources while it does.

## Prefer decoupling to defending

Every technique above is damage control on a synchronous call. Where the business logic tolerates it, an asynchronous or message-based integration removes the problem class entirely: the caller's thread lifetime stops being coupled to the callee's response time, and a slow consumer becomes a growing queue rather than a stalled caller.

This is a design-time choice, not something you can retrofit under incident pressure. When the interaction genuinely needs a synchronous answer, defend it. When it does not, decouple it.

## SLA arithmetic

Availability multiplies down a synchronous call chain. Depending on N dependencies at 99.9% each puts your ceiling near 0.999^N — three such dependencies caps you at roughly 99.7% before you write a single bug.

Two consequences:

- **Enumerate every outbound dependency**, including the infrastructure ones nobody lists: DNS, SMTP, the message broker, the auth provider, the metrics endpoint. An uncovered dependency silently caps everything above it.
- **Write availability targets per feature, not per system.** A breaker plus graceful degradation means a low-SLA dependency removes one feature rather than the whole application — but only if the feature boundary is real in the code.

## What to check in the code

- Client construction: are connect and read timeouts set explicitly, or inherited from defaults?
- Pool configuration: is checkout bounded, and is there a defined fallback on expiry?
- Timed variants: any no-argument `wait()`, `poll()`, `lock()`, or `join()` in a request path.
- Breaker coverage: does every network dependency have one, scoped individually?
- Breaker accounting: do transitions get logged and counted apart from ordinary errors?
- Retry logic: is it immediate (wrong) or backed off and bounded (right), and is the operation idempotent?
- Timeout ordering: does each layer's timeout fit inside its caller's?
