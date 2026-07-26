# Overload and Cascading Failure

Deeper treatment of what happens when a service is *already* past capacity, and how a cascade actually sustains itself. Complements [blocking-calls-and-breakers](blocking-calls-and-breakers.md), which covers preventing the first hang.

## Two corrections to naive instincts

Before anything else, because both are common and both make outages worse:

**"Fail fast" does not mean go dark.** An overloaded backend should keep serving up to its actual provisioned capacity and reject only the excess. A service that stops serving entirely when it crosses a threshold throws away capacity it still has.

**Do not restart during an active cascade.** A freshly restarted process starts with a cold cache and needs *more* resources to reach steady state than a warm one — exactly what is unavailable. Restarting a crash-looping fleet extends the outage. Restart only when a process is genuinely wedged (GC death spiral, deadlock), and canary it rather than doing the whole fleet.

## The feedback loop inside a single process

Cascades are usually described as load moving between machines. The more dangerous loop is internal to one surviving process, and needs no external trigger to continue:

1. Capacity is removed — crashed tasks, a bad push, a drain, or growth outrunning provisioning.
2. Survivors absorb the load, so each request takes longer.
3. Longer requests mean more requests **in flight simultaneously**.
4. More in-flight requests consume more memory, threads, and file descriptors.
5. That pressure — garbage collection pressure especially — slows request handling further, which returns to step 3.

Nothing external needs to keep pushing. The loop sustains itself.

## Retry amplification

Retries compound overload multiplicatively, not additively.

A backend with 10,000 QPS capacity receiving 10,100 QPS fails 100 QPS. Those retry and arrive as *new* load, so offered load becomes 10,200, which fails more, which retries. The retry volume grows every cycle.

Worse across layers: if three layers each retry three times independently, one user action can become up to **64 downstream attempts**. Each layer's retry budget looks reasonable in isolation.

Defenses:

- **Cap retries per request** (three attempts is a reasonable ceiling) *and* per process — a fixed retries-per-minute budget, so a widespread blip cannot synchronize a storm.
- **Randomized exponential backoff**, never fixed-interval retry.
- **Return a distinct "overloaded, do not retry" status** rather than a generic error, so clients back off instead of amplifying. This is the single cheapest fix and it requires cooperation from both sides.
- **Retry at one layer only** where you can arrange it. Layered independent retries are what produces the combinatorial blowup.

## Deadline propagation

A timeout at every hop is not the same as a deadline budget, and the difference matters.

If each hop starts a fresh fixed timeout, a deep call chain keeps working on a request the original caller abandoned long ago — burning capacity on output nobody will read.

Instead, propagate a **remaining budget**: each hop passes downstream the time left, decremented by what it has already spent. When the budget is exhausted, work stops everywhere. Propagate cancellation too, so an abandoned request releases resources rather than running to completion.

## Bimodal latency and thread-pool math

A small fraction of very slow requests can dominate a bounded pool, because occupancy is duration-weighted, not count-weighted.

If most requests are fast but a few percent hang until a long timeout, those few can consume most of the pool. A 5% hang rate against a deadline 100× typical can push the effective error rate past 80% — not the 5% you would predict from the hang rate alone.

This is why a mismatched deadline on a rarely-slow dependency is so much more dangerous than its failure rate suggests.

## Client-side adaptive throttling

A circuit breaker is binary: it sends 100% of traffic until it trips, then 0%. There is no gradient, so recovery is jerky — the breaker closes, full traffic floods back, the breaker trips again.

Adaptive throttling is continuous. Each client tracks, over a short rolling window, **requests attempted** versus **requests accepted** by the backend. Once attempted exceeds accepted by a configurable multiplier (2× is a reasonable default), the client begins probabilistically rejecting its *own* requests locally, before they reach the network, with the rejection probability rising smoothly as the ratio worsens.

Two properties make this valuable: it degrades gradually rather than in steps, and it needs no explicit "I am overloaded" signal from the backend — the accept ratio is sufficient.

Treat it as a complement to a breaker, not a replacement.

## Criticality and cost accounting

When one backend serves many callers from one capacity pool, undifferentiated shedding fails a user's synchronous request and a retryable batch job with equal probability.

- **Tag every request with a criticality level.** Four tiers work: must-not-fail, normal production, degradable batch, freely sheddable. Set it at the entry point and propagate it to every downstream call the request triggers.
- **Shed lowest criticality first** as local utilization crosses configured thresholds, with higher thresholds permitted for higher tiers.
- **Provision quota per criticality**, not just per caller, so a tenant exhausting its budget loses its lowest tier first.
- **Measure cost in consumed resources, not request count.** Requests on the same endpoint can differ in cost by three orders of magnitude, which makes QPS-based capacity planning badly wrong.

## Graceful degradation as a designed mode

Most services have exactly two states: full-fidelity response, or error. That wastes headroom — a degraded answer would satisfy most callers.

Build an explicit cheaper path per hot endpoint: search a smaller candidate set, serve a slightly stale cached copy, drop optional page elements. Make entering it a **deliberate config-controlled switch**, not an emergent consequence of resource exhaustion.

The critical operational point: **this path only executes during incidents, so it will rot.** Run a small slice of production traffic through the degraded path continuously so the code stays exercised and the behavior stays known.

## Load balancing

**Round robin assumes uniform request cost and uniform backend capacity.** Neither holds — request cost varies by orders of magnitude on the same endpoint, and machine performance varies by roughly 20% from hardware generation and noisy neighbors. The result is a persistent 2× spread between least- and most-loaded backend, which is reserved capacity you paid for and cannot use.

**Least-connections has a specific trap.** A backend that fails *instantly* looks idle, because its in-flight count stays near zero. The balancer then sinks disproportionate traffic into the broken one. Counteract by counting recent errors as though they were still-active requests.

**Prefer weighted distribution** driven by each backend's self-reported utilization and recent error rate.

**At the connection layer, never `hash(connection) mod N`.** Losing one backend changes N and remaps nearly every existing connection simultaneously. Consistent hashing remaps only the minimal necessary slice.

## Subsetting

Full mesh connectivity between M clients and N backends costs O(M×N) connections. Health-check and connection overhead can exceed the cost of serving actual requests.

The obvious fix — each client randomly picks a fixed subset — distributes load unevenly in practice, and worse, when a backend fails its load concentrates onto the few others in the same subsets, which is a cascade in miniature.

Use **deterministic subsetting**: divide clients into rounds whose subsets exactly partition the backend set, shuffle backends with a round-specific seed, and give each client in a round a distinct slice. Every backend then receives nearly the same number of clients, and — because the shuffle differs per round — a failed backend's load redistributes fleet-wide rather than concentrating.

## Scheduled jobs and thundering herds

Humans converge on round numbers. Independent teams all schedule at midnight or the top of the hour, so many jobs — each possibly fanning out into a large batch — fire in the same second. The synchronized spike has no underlying dependency between the jobs; it is purely a scheduling convention artifact.

- **Jitter deterministically.** Where the exact minute does not matter, derive the dispatch time within a window from a hash of the job's identity. Deterministic rather than random, so it is reproducible.
- **Single-machine cron has no failover.** If the host is down, nothing fires and nothing reschedules.
- **Naive distributed cron risks duplicate launches.** Two replicas both believing they are responsible will double-fire a non-idempotent job. Use a leader with a replicated log recording launch start and end, and construct job identifiers that embed the scheduled time so a partially completed launch is recognizable rather than repeated.

## Recovering from an active cascade

Ordered, because the order matters:

1. **Add capacity** if any is idle.
2. **Separate process-alive from can-serve health checks.** A saturated-but-alive task should not be killed and replaced with an equally cold one.
3. **Shed traffic hard** — as low as 1% of normal. Recovery generally requires dropping load *an order of magnitude* below what triggered the cascade, because only a small fraction of the fleet is healthy enough to absorb anything at first.
4. **Ramp back gradually**, letting caches rewarm before each increase.
5. **Restart only if genuinely wedged**, and canary it.

The load level required to stabilize should be measured in a load test and written down, not guessed at during the incident.

## What to check in the code

- Are retries capped per request *and* per process, with randomized backoff?
- Is there a distinct non-retryable overload status, and do clients honor it?
- Do downstream calls receive a decremented remaining budget, or a fresh fixed timeout?
- Is cancellation propagated when a caller gives up?
- Is there a degraded path, and does anything exercise it outside incidents?
- Does the load balancer weight by backend capability, or assume uniformity?
- Do scheduled jobs cluster at round times?
- Has anyone measured the load level required to recover, as opposed to the level that breaks it?
