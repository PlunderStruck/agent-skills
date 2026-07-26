# Extraction: sre-overload

**Source:** Site Reliability Engineering — Google

> Operational rules distilled from the source, written in our own words. This is the intermediate layer between the source text and the published skill — denser than the skill, and the place detail survives for re-distillation.

---

# Google SRE Book vs. `resilience` (Nygard) — Differentiation Report

Scope: eight chapters (handling-overload, addressing-cascading-failures, load-balancing-frontend, load-balancing-datacenter, managing-critical-state, distributed-periodic-scheduling, service-level-objectives, monitoring-distributed-systems). Organizational/process material (on-call, postmortems, hiring) excluded. Everything below binds to code or service configuration.

---

## PART 1 — New material `resilience` does not cover

### 1. Client-side adaptive throttling (not a circuit breaker)

**Trigger:** A client library calling a service that can reject requests under load, where the client currently either retries blindly or relies solely on a server-side circuit breaker.

**What breaks:** A binary circuit breaker (open/closed) reacts late — it trips only after enough failures accumulate, and while closed it sends 100% of traffic even as the backend is degrading. There's no gradient between "send everything" and "send nothing," so recovery is jerky (breaker closes, full traffic floods back in, breaker trips again).

**What to do:** Each client tracks two rolling counters over a short window (e.g., two minutes): requests attempted and requests accepted by the backend. Once attempted requests exceed accepted requests by a configurable multiplier K (Google's default is 2x), the client starts probabilistically rejecting its own requests locally — before they ever hit the network — with a rejection probability that rises smoothly as the ratio worsens. Lowering K makes self-throttling kick in earlier and harder; K near 1 means almost no slack. This is continuous and self-correcting rather than a discrete trip/reset state machine, and it works even when the backend has no explicit "overloaded" signal, because it's driven purely by the client's own accept ratio.

**How to verify:** Load-test a dependency past its capacity and confirm client-side rejection rate rises smoothly with backend saturation rather than the client hammering the wire until a breaker trips; confirm recovery is gradual (no full-traffic slam once the backend recovers).

### 2. Criticality levels and per-request cost accounting

**Trigger:** A shared backend serves multiple call sites or multiple customers/tenants from one capacity pool, and overload currently causes an undifferentiated mix of important and unimportant requests to fail together.

**What breaks:** Without a priority signal attached to the request, load shedding is capacity-blind — a batch job's retryable request and a user's synchronous request compete equally for the last available thread, and both may fail. Provisioning capacity by request-count or QPS also mis-forecasts cost, because requests on the same endpoint can vary in resource cost by 100–1000x.

**What to do:** Tag every request with a criticality (four tiers work well: must-not-fail, default production traffic, degradable batch traffic, freely sheddable traffic) set at the entry point and propagated by default to any downstream RPCs it triggers. Provision and grant quota per criticality, not just per caller — a tenant that exhausts its quota only starts losing its lowest tier first, and a backend under local overload sheds lowest-criticality traffic first as its own utilization crosses configured thresholds (higher thresholds allowed for higher tiers). Measure cost in actual consumed resources (CPU-seconds, memory) per caller rather than request count, and aggregate that usage globally in near-real-time to push effective per-caller limits back down to individual tasks.

**How to verify:** Inject synthetic overload and confirm shedding order matches criticality (lowest tier sheds first, `CRITICAL_PLUS`-equivalent survives longest); confirm a single noisy tenant's low-criticality burst doesn't consume quota that starves another tenant's high-criticality traffic.

### 3. Graceful degradation as a designed, load-tested mode

**Trigger:** A service currently has exactly two states under load — full-fidelity response or error — with nothing in between.

**What breaks:** Binary fail/succeed wastes headroom: a search service that could return partial, cheaper results instead returns nothing once it can't do the full-cost computation, even though a degraded answer would satisfy most callers.

**What to do:** Build an explicit cheaper code path per hot endpoint — search a smaller candidate set instead of the whole corpus, serve a slightly stale cached copy instead of hitting canonical storage, drop optional page elements (images, secondary widgets) — and make the switch to that path an intentional, centrally-controlled config flag rather than an emergent side effect of resource exhaustion. Because this path only executes during real overload, it will rot if untested; deliberately run a small slice of production capacity in degraded mode continuously (not just during incidents) so the code path stays exercised.

**How to verify:** Confirm the degraded path is covered by regular (not just incident-time) canary traffic, and that a load test past capacity shows the service settling into degraded-but-serving rather than oscillating into full failure.

### 4. Cascading-failure feedback loops, and why restarting servers makes it worse

**Trigger:** A fleet is running near its ceiling, and something removes capacity — a few crashed tasks, a bad push, a maintenance drain, or organic growth outrunning provisioning.

**What breaks — the self-sustaining part:** Load lost from the removed capacity lands on survivors, which slows their request handling. Slower requests mean more requests are in flight simultaneously, which consumes more memory, threads, and file descriptors per survivor. That resource pressure (e.g., GC pressure in a managed runtime) further slows request handling, which increases in-flight concurrency again — a positive feedback loop entirely internal to a single surviving task, independent of any new external trigger. In parallel, retries compound it: if a backend at 10,000 QPS capacity gets 10,100 QPS, the 100 QPS of failures retry and land as new load, pushing total offered load past 10,200; the retry volume itself grows every cycle. Retries fanning out across independent layers (frontend → backend → database, each retrying 3x) multiply combinatorially — three independently-retrying layers can turn one user action into up to 4³ (64) downstream attempts. A deadline mismatch compounds this further: if a small fraction of requests hang until a long timeout while most complete fast, that small fraction can consume a disproportionate share of a bounded thread pool (a 5% hang rate against a 100x-longer-than-typical deadline can push effective error rate above 80%, not 5%), because the pool exhausts on duration-weighted occupancy, not request count.

**Why naive restarts fail:** A crash-looping fleet is not healed by restarting the crashing processes, because a freshly restarted task typically starts with a cold cache and briefly needs more resources to reach steady state than a warm task — exactly what it doesn't have room for. Recovering from a cascade generally requires dropping load far below the level that originally triggered it (an order of magnitude, not a modest pullback), because only a small fraction of the fleet is healthy enough at first to absorb any load at all; feeding it the previous "acceptable" load immediately re-triggers the same collapse.

**What to do:** To stop an active cascade: add capacity if any is idle; distinguish process-alive health checks from can-serve-this-traffic health checks (don't let a saturated-but-alive task get killed and replaced with an equally cold one); shed traffic hard (as low as 1% of normal) and ramp back up gradually, letting caches rewarm before increasing further; only restart if genuinely wedged (GC death spiral, deadlock), and canary that restart rather than fleet-wide. To prevent cascades: give every outbound call a deadline that's a fraction of the remaining budget on the inbound request (deadline propagation — decrementing a shared clock through a call chain, not resetting a fresh fixed timeout at every hop, which lets deep calls keep working on a request the original caller already gave up on); propagate cancellation when the caller stops waiting; cap retries per request (e.g., 3 attempts) and per process (e.g., a fixed retries/minute budget) with randomized exponential backoff so a transient blip doesn't synchronize a retry storm; return a distinct "overloaded, do not retry" status instead of a generic error so clients back off instead of amplifying; and avoid peer-to-peer (intra-layer) request cycles that can deadlock on shared thread pools.

**How to verify:** Load test to failure and record what recovery load is required to stabilize (should be documented, not assumed); test that a permanently unresponsive noncritical dependency doesn't consume the calling service's thread pool or blow past its own deadline budget; verify retry counters/histograms exist so a backend can detect "everyone's retrying" and switch to non-retryable rejection.

### 5. Load-balancing algorithm choice, and why round robin misbehaves

**Trigger:** Multiple backend replicas behind a client-side or L7 balancer, currently balanced by naive round robin or unweighted connection count.

**What breaks:** Plain round robin assumes every request costs the same and every backend has identical capacity — neither holds in practice (request cost can vary 100-1000x on the same endpoint; machine performance varies with hardware generation and noisy neighbors by up to ~20%). The result is a persistent 2x spread between the least- and most-loaded backend, wasting reserved capacity. A naive fix — routing to whichever backend has the fewest active in-flight requests — has its own trap: a broken backend that fails instantly (rather than hanging) looks "available" because its active-request count stays near zero, so the balancer sinks disproportionate traffic into it (must be counteracted by counting recent errors as if they were still-active requests).

**What to do:** Move to a weighted scheme where each client periodically updates a per-backend capability score from that backend's self-reported utilization (CPU, error rate, recent throughput), and skews request share toward higher-capability backends. At the connection level (L4/VIP), don't route by a naive `hash(connection) mod N` — losing one backend changes N and re-maps nearly every existing connection to a different backend simultaneously; use a consistent-hashing scheme so backend-count changes only remap the minimal necessary slice of connections.

**How to verify:** Under synthetic load with deliberately uneven backend cost/capacity, measure CPU spread between least- and most-loaded backend before/after switching algorithms; inject a backend that fails fast (not slow) and confirm it doesn't attract a disproportionate share of traffic.

### 6. Subsetting: bounding client-backend connection fan-out without losing balance

**Trigger:** A large fleet of clients each maintaining connections/health-checks against a large fleet of backends (all-to-all), where connection/health-check overhead itself becomes a resource cost.

**What breaks:** Full mesh connectivity between M clients and N backends costs O(M×N) connections; that overhead (memory, periodic health-check CPU) can exceed the cost of actually serving requests. The obvious fix — each client randomly picks a fixed small subset of backends once — spreads load unevenly in practice (e.g., a 10% random subset can leave some backends at 50% of average load and others at 150%), because random-without-structure doesn't guarantee even coverage, especially as backends fail and reduce the effective pool further within already-uneven subsets.

**What to do:** Use deterministic subsetting: divide clients into rounds (a round is a batch of clients whose subsets exactly partition the backend set), shuffle backends with a round-specific seed, and assign each client in a round a distinct slice. This guarantees every backend gets almost exactly the same number of clients pointed at it, and — critically — uses a different shuffle per round so that if a backend fails, the load it was carrying spreads across the whole backend fleet rather than concentrating onto the few backends left in its own subset (which would otherwise compound into a mini cascade).

**How to verify:** Simulate backend failures within the subsetting scheme and confirm load redistributes fleet-wide, not just within the failed backend's original subset; measure connection-count reduction versus full mesh.

### 7. Distributed consensus for critical shared state (leader election, locks, config)

**Trigger:** Any place where the code currently elects a leader, acquires a distributed lock, or agrees on group membership using heartbeats/timeouts/gossip instead of a consensus protocol (Paxos/Raft/Zab or a service built on one, e.g. Zookeeper/etcd/Consul).

**What breaks:** Heartbeat-based leader election is not a reformulation of a solvable problem — it's the distributed consensus problem, and simple timeouts cannot solve it safely. Under degraded network conditions (not full partition, just packet loss or slow links) two peers can each independently decide the other is dead and both assume leadership — split brain — corrupting data or serving stale/conflicting state. Gossip-based membership has the analogous failure during a network partition: both halves independently elect a leader and both accept writes.
Indefinite locks compound this: a lock held by a process that then crashes blocks the resource forever.

**What to do:** For leader election, distributed locks, or any state that must be viewed consistently by multiple processes, use an actual consensus-backed primitive rather than hand-rolled heartbeats — either embed a consensus library or depend on a consensus-backed service. Locks and leadership should be leases (time-bounded, renewable) rather than indefinite locks, so a crashed holder's claim expires automatically instead of requiring manual intervention. Size replica counts for the failure tolerance you need: 3 replicas tolerate 1 failure but leave zero margin during planned maintenance; 5 tolerate 2 and allow maintenance without dropping below safe quorum. Don't over-extend the geographic failure domain beyond where your clients actually run — spreading a consensus cluster across a wider area than its clients only adds latency without adding real availability, since the clients go down with the same event anyway.

**How to verify:** Grep for any hand-rolled leader-election-by-heartbeat or "last writer wins" locking against shared state; confirm locks used for mutual exclusion carry an expiry/lease, not an indefinite hold; monitor leader-term change frequency (rapid flapping indicates network instability, not just failover) and confirm the system alerts on a sustained leaderless state.

### 8. Distributed periodic scheduling and thundering herds

**Trigger:** Any cron-like job scheduled at a round time (midnight, top of hour) across a fleet, or a single-machine cron whose failure domain is that one machine.

**What breaks:** Single-machine cron has zero failover — if the host is down, nothing fires or reschedules. At fleet scale, humans naturally converge job schedules on round numbers, so many independent teams' jobs (each of which may itself fan out into a large batch job) all fire in the same second, producing a synchronized spike in scheduling-system and downstream resource load — a thundering herd caused purely by human scheduling convention, not by any actual dependency between the jobs. Naively making cron itself distributed (multiple replicas racing to launch jobs) introduces a new risk: duplicate, non-idempotent job launches (e.g., a job that sends a customer notification) if two replicas both believe they're responsible.

**What to do:** Where the exact minute doesn't matter, let the scheduler pick the actual dispatch time within a window (e.g., "some minute this hour") deterministically from a hash of the job's identity, so jobs that could all fire at :00 get spread across the hour instead. Make the distributed cron service itself leader-based with a consensus-backed replicated log recording the start/end of each scheduled launch, so a fired-but-uncertain launch (e.g., because the leader died mid-dispatch) can be resolved unambiguously on failover instead of risking a duplicate; construct job identifiers up front that embed the scheduled time so a partially-completed launch can be recognized and not repeated.

**How to verify:** Chart the timing distribution of scheduled job starts across the fleet and confirm no unintentional synchronized spike at round times; test leader failover mid-launch and confirm the job neither double-fires nor silently drops.

### 9. Error budgets as a release/operational decision mechanism

**Trigger:** A service has an availability or latency target expressed informally ("should be fast," "should be reliable") with no quantitative gate on how much unreliability is acceptable, and no defined link between that target and release decisions.

**What breaks:** Without a numeric target and a numeric measure of current standing against it, "should we ship this risky change now" has no objective answer — teams either ship recklessly or freeze unnecessarily, and there's no shared vocabulary between whoever writes the feature and whoever operates it.

**What to do:** Define an SLI (a precisely specified measured quantity — e.g., proportion of requests under 200ms measured server-side over 1-minute windows), an SLO (a target range on that SLI, e.g., 99.9% under 200ms), and treat the SLO's complement as an error budget (0.1% of requests, tracked daily/weekly and rolled up monthly). Use that budget as a literal gate: while the error budget for the tracking period isn't exhausted, release velocity/risk-taking is allowed to continue at normal pace; once it's exhausted, deployment risk should be constrained until standing improves. Prefer few, simple, defensible SLOs over many precise ones — an SLO you can't win a prioritization argument with is a sign you don't need it.

**How to verify:** Confirm an automatic, continuously-updated dashboard or check compares live SLI against SLO and exposes remaining error budget for the period, and confirm there is an actual (even informal) process step where release approval consults that number rather than everyone just assuming things are fine.

### 10. The four golden signals, and alerting on symptoms instead of causes

**Trigger:** An alerting/monitoring setup that fires on internal conditions (a specific database's connection pool, a specific dependency being slow) rather than on user-visible outcomes, or that has no minimal, agreed-upon signal set for a service.

**What breaks:** Cause-based alerting doesn't scale with dependency count or refactor rate — every internal component gets its own alert, most of which are either redundant with an upstream symptom alert or fire for conditions that never actually hurt a user (traffic already drained from that instance, a test deployment, a transient blip a retry absorbed). This produces alert fatigue: engineers start ignoring or triaging without urgency, which then buries a genuinely urgent page in noise. Mean-based latency alerting hides the actual problem too — a 100ms average can coexist with a tail where 1% of requests take 5 seconds, and that tail is often the leading indicator of saturation before it shows up in the average at all.

**What to do:** For any user-facing service, instrument at minimum: latency (success and failure paths tracked separately — a fast error is not the same signal as a slow one), traffic (a domain-appropriate demand metric), errors (explicit failure codes, but also "wrong answer with a 200 status," and policy-violation cases), and saturation (headroom on whichever resource is the actual constraint, since many systems degrade well before hitting 100% utilization — this makes it a leading indicator, not a lagging one). Page only on symptoms that are urgent, actively user-visible, and actionable by a human right now; run a short gate before adding any paging rule — will this ever legitimately need to be ignored, does it definitely mean users are affected right now, is there a real action a human can take, is someone else already getting paged for the same underlying issue. Route everything else (internal-cause telemetry, known/rote-response conditions) to dashboards or tickets, not pages. Track latency as a histogram/percentile distribution (roughly exponential bucket boundaries), not an average, and treat a high percentile (99th) over a short window as an early saturation signal.

**How to verify:** Audit the current alert set against the five-question gate above and count how many currently-paging rules fail it; confirm latency dashboards show percentile buckets, not just a mean; confirm each of the four signals has at least one instrumented metric per user-facing service.

---

## PART 2 — Overlap with `resilience` (brief)

- **Timeouts on every blocking call** — overlaps with SRE's deadline handling, but SRE's *deadline propagation* (decrementing a shared budget through a call chain rather than each hop getting its own fresh fixed timeout) is materially deeper and quietly corrects a common naive implementation of "just add a timeout everywhere."
- **Circuit breaker state machine** — overlaps conceptually with load protection, but SRE's client-side adaptive throttling is a genuinely different mechanism (continuous probabilistic self-rejection driven by an accept/attempt ratio) rather than a binary trip/half-open/closed state machine — worth treating as a complement, not a replacement.
- **Fail fast** — overlaps with SRE's load-shedding guidance, and SRE adds a specific correction to a naive reading of "fail fast": an overloaded backend should *keep* accepting and serving up to its actual provisioned capacity and reject only the excess, not stop serving entirely. SRE also adds concrete queue-discipline guidance (keep queue length at or below ~50% of thread-pool size for steady traffic; consider CoDel-style dropping of requests unlikely to still be useful) that goes beyond a bare fail-fast rule.
- **SLA multiplication down a dependency chain** — related to but distinct from SRE's retry-amplification math (100 QPS of failures triggering a growing retry loop; three independently-retrying layers multiplying one user action into up to 64 downstream attempts). SLA multiplication is about compounding *availability probability*; retry amplification is about compounding *request volume* — both matter and are worth keeping separate.
- **Bulkheads and pool isolation** — adjacent to but distinct from SRE's subsetting (subsetting bounds connection/health-check overhead and balances load across a bounded backend set; bulkheads isolate failure domains within one process). Not a duplicate.
- **Chain reactions in homogeneous fleets** — names the same top-level pattern as SRE's cascading-failure chapter, but SRE's treatment is substantially deeper: the specific resource-exhaustion feedback loop (CPU→memory→GC→CPU), the retry-storm growth curve, the bimodal-latency thread-pool-exhaustion math, and — most importantly — the explicit, non-obvious reasoning for why naive restarts fail during an active cascade (cold-cache cost, need to drop load an order of magnitude, not just back off modestly). Treat the resilience skill's coverage as "recognize the pattern" and this material as "know the actual intervention."
- **O(n²) fan-out and shared coordinators** — names the same problem subsetting solves; SRE supplies the concrete algorithm (deterministic round-based subsetting) rather than just the anti-pattern name.
- **Unbalanced front-end/back-end capacity** — adjacent to but not the same as SRE's load-balancing-algorithm material (round robin's 2x spread, weighted round robin, the "wasted capacity" model of summed CPU differences from the least-loaded task). The resilience item is about tier-to-tier capacity mismatch; SRE's is about intra-tier balancing algorithm choice.
- **Handshaking / startup fail-fast and graceful drain** — overlaps with SRE's lame-duck state machine (healthy / refusing-connections / lame-duck, propagated via fast UDP health checks in 1-2 RTT, with a 10-150 second drain window before forced kill). SRE's version is more operationally specific (exact state names, propagation latency, drain-window sizing guidance) — treat as a deepening, not new material.
- **Self-inflicted load from promotions / session-creation floods** — the general pattern overlaps with SRE's connection-burst and thundering-herd material, but the periodic-scheduling instance (synchronized cron at round times) and its jitter-by-hash remedy is specific enough to stand as new (see Part 1 §8).
- **Unbounded result sets / steady-state violations** — no meaningful overlap with SRE material found beyond general "bound your resource usage" philosophy; not worth merging.
- **Pool contention** — overlaps loosely with SRE's lease mechanics (leases prevent a crashed holder from blocking a resource forever), but SRE's lease discussion lives in a distributed-consensus context (cluster-wide locks/leadership), not local connection-pool contention — different scale of problem.
- **Hostile failure testing** — overlaps with SRE's "load test until it breaks" and "test that a blackholed noncritical dependency can't consume the caller's resources" guidance; SRE's treatment is more prescriptive about test shape (gradual ramp vs. impulse, explicitly testing the recovery point, not just the breaking point) but the philosophy is the same as Nygard's.
- **Correlation IDs and operator-focused logging / what to instrument** — overlaps loosely with SRE's four golden signals, but SRE's contribution is a decision framework layered on top (what deserves a page vs. a ticket vs. a log line) rather than a logging-format concern — treat Part 1 §10 as the deepening, not a duplicate of existing logging guidance.

**Where Google's treatment mildly contradicts a naive resilience instinct:** an engineer reading "fail fast" and "circuit breaker" alone might implement "reject everything once overloaded" or "restart the failing pod" as the default recovery move. SRE explicitly warns against both defaults in the overload and cascading-failure chapters — keep serving at capacity rather than going fully dark, and don't restart during an active cascade without first identifying and fixing the trigger, because a cold restart can extend the outage rather than end it.
