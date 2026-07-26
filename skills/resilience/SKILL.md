---
name: resilience
description: Keep code alive when its dependencies misbehave — timeouts on every remote call, circuit breakers, bulkheads, bounded pools and queries, graceful startup and shutdown, and operator-usable logging. Use when writing or reviewing HTTP/RPC clients, database access, connection or thread pools, caches, retry logic, health checks, background workers, or any code that calls something it does not control.
---

# Resilience

## Purpose

Systems rarely fall over because a component broke. They fall over because a component got **slow**, and nothing in the calling code was prepared for that.

A dependency that returns an error is easy — you handle the error. A dependency that accepts your connection and then never replies is what takes down production, because the default behavior of almost every client library is to wait forever. This skill is about the code-level defenses against that, and against the resource exhaustion that follows.

Most of these defects are invisible in development, where dependencies are fast, data is small, and one instance is running.

## When this applies

- Any call across a process boundary: HTTP, RPC, database, cache, queue, third-party SDK
- Connection pools, thread pools, worker pools, semaphores
- Queries that return collections; anything that accumulates (logs, sessions, caches, audit tables)
- Retry logic, health checks, readiness probes
- Startup and shutdown paths
- Anything a promotion, campaign, or traffic spike will hit

## The chain that causes most outages

One mechanism wearing four names. Understanding it once explains most of this skill:

1. A remote call is issued **with no timeout**. The dependency hangs. The thread blocks.
2. More requests arrive. More threads block on the same dependency. The **pool exhausts**.
3. The process is now alive, low-CPU, and 100% unresponsive — which naive health checks report as healthy. It is functionally a crash, but harder to detect.
4. Callers of *this* service now block on it too. The failure **cascades upward**, jumping layers even though nothing past the first hop actually broke.

The entry point to that chain is a single missing timeout. That is why the rules below are ordered the way they are.

## Rules that apply without loading anything

**1. Every blocking call gets an explicit timeout.** Connect timeout, read timeout, query timeout, bounded pool checkout, bounded lock acquisition. Never the no-argument blocking form. Library defaults are usually "wait forever" — set them yourself rather than trusting them.

**2. A slow response is worse than an error.** An error frees your thread immediately; a slow response holds resources on both ends for the full timeout, and users pressing reload multiply load on a system that is already drowning. Prefer failing fast over waiting hopefully.

**3. Wrap remote calls in a circuit breaker.** Count failures, trip open, fail immediately without touching the network, retry on a half-open trial after a cooldown. This is the single highest-leverage defense against a downstream failure climbing the stack.

**4. Availability multiplies.** Depending synchronously on N services at 99.9% each caps you near 0.999^N, not 99.9%. Any uncovered dependency — including DNS, SMTP, and the message broker everyone forgets — sets the ceiling for everything above it.

**5. Never retry a timeout immediately.** The immediate retry usually just repeats the failure while doubling load. Queue for delayed retry with backoff, and make the operation idempotent first — see `distributed-data` for how.

**6. Every query gets an explicit row limit.** A table that "obviously stays small" reaches millions of rows in production. Loops over unbounded result sets exhaust memory, and can take an entire fleet down together when every instance queries the same bloated table on restart.

**7. Everything that accumulates needs a matching removal path** — at launch, not later. Logs rotate by size, caches have a bounded size and an eviction policy, data has a working purge routine. Anything that grows without bound eventually saturates disk or heap.

**8. Isolate blast radius with bulkheads.** Separate pools per critical caller, tenant, or feature, so one consumer's overload cannot starve the others sharing a resource.

**9. "Fail fast" does not mean go dark.** An overloaded service should keep serving up to its actual capacity and reject only the excess. Refusing everything past a threshold throws away capacity you still have.

**10. Do not restart during an active cascade.** A restarted process starts cold and needs *more* resources to reach steady state than a warm one — exactly what is unavailable. Restarting a crash-looping fleet extends the outage. Recovery means shedding load far below the level that triggered the cascade, then ramping back as caches rewarm.

**11. Pass a decrementing deadline, not a fresh timeout per hop.** A fixed timeout at every layer lets deep calls keep working on a request the original caller abandoned. Propagate the remaining budget, and propagate cancellation.

## Triage

| What you're writing | Failure to check | Reference |
|---|---|---|
| HTTP/RPC client, DB call, any remote call | Hang, pool exhaustion, cascade | [blocking-calls-and-breakers](references/blocking-calls-and-breakers.md) |
| Shared pools, multi-tenant paths, fleet-wide behavior | One failure taking everything | [blast-radius](references/blast-radius.md) |
| Load shedding, retry budgets, load balancing, scheduled jobs | Self-sustaining cascade, retry amplification | [overload-and-cascades](references/overload-and-cascades.md) |
| Queries, caches, sessions, logs, background accumulation | Unbounded growth, capacity cliffs | [unbounded-growth](references/unbounded-growth.md) |
| Integration tests against a dependency | Passing tests, production hangs | [failure-testing](references/failure-testing.md) |
| Startup, shutdown, config, logging, metrics | Unoperable under incident | [operability](references/operability.md) |

## Boundary with `distributed-data`

They overlap on retries and timeouts but ask different questions:

- **`distributed-data`** — will the data be *correct*? Duplicate execution, lost updates, stale reads, divergence between stores.
- **`resilience`** — will the system stay *up*? Hangs, exhaustion, cascading failure, capacity cliffs.

A retry needs both: idempotency so it doesn't double-charge, and backoff plus a breaker so it doesn't amplify an outage. When writing retry logic, check both.

## What this skill will not tell you

Whether the availability is worth the cost. Bulkheads waste capacity by design, breakers reject requests that might have succeeded, and timeouts fail work that was merely slow. This skill identifies what breaks and how to contain it; how much availability the feature actually warrants is a product decision.
