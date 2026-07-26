# Extraction: resilience

**Source:** Release It! — Michael Nygard

> Operational rules distilled from the source, written in our own words. This is the intermediate layer between the source text and the published skill — denser than the skill, and the place detail survives for re-distillation.

---

This looks solid — comprehensive, properly distilled into my own words, organized in causal clusters, and within the word target. Here is the final document.

# Release It! — Distilled Rules for Production-Survivable Code

Organized into causal clusters — most of these failure modes chain into each other rather than occurring in isolation.

## Cluster A: The Blocking-Call Chain (root cause of most outages)

One mechanism, four names. An unprotected remote call blocks a thread; enough blocked threads exhaust a pool; an exhausted pool in one service makes its callers block too; a system with no way to shed load just gets slower until every caller times out at once.

**Trigger:** Any I/O across a process boundary — HTTP call, RPC, socket read, DB query, third-party client library — issued with no timeout, or a resource-pool checkout that blocks the caller indefinitely when exhausted. Remote-object protocols (RMI, DCOM, CORBA) are worst because they have no default timeout at all, so a call that looks local can hang forever.

**What breaks:** A slow/hung dependency doesn't fail — it never returns. Threads pile up waiting; once every thread in a pool is blocked, the process is alive (low CPU, looks "healthy" to naive checks) but 100% unresponsive — functionally a crash, harder to detect. The blast radius isn't local: callers of the frozen service block too, so the outage jumps layer to layer even though nothing physically broke past the first hop. A slow response is worse than an error, since it ties up resources on both ends for the full timeout — and users hitting reload multiply load on an already-drowning system.

**What the code must do, in priority order:**
1. Timeout every blocking call and every pool checkout — connect timeout, read timeout, query timeout, bounded `wait()`/`poll()`/`tryLock()`, never the no-argument blocking form.
2. Wrap remote calls in a circuit breaker: track failures, trip "open" so later calls fail immediately without touching the network, try a "half-open" trial after a cooldown, reset to "closed" on success. This is the single highest-leverage defense against a downstream failure propagating upward.
3. Fail fast on the caller side: check whether a call can plausibly succeed (pool health, breaker state) before doing expensive setup, and reject immediately rather than getting partway through a transaction.
4. Prefer async/message-based integration over synchronous request-reply where the business logic tolerates it — a queue fully decouples the caller's thread lifetime from the callee's response time.
5. On timeout, don't retry immediately (it usually just repeats the failure); queue for delayed retry instead.

**How to verify:** Grep client construction for explicit connect/read timeouts, not library defaults. Confirm every pool checkout has a bounded wait and a defined fallback on expiry. Point dependencies at a hostile stub that never replies, replies one byte at a time, or returns garbage, and confirm the caller degrades instead of hanging. Confirm breaker state transitions are logged and counted separately from ordinary failures.

## Cluster B: SLA Math

**Trigger:** A service claims an availability target while calling any dependency — including DNS, SMTP, a message broker, a third-party API — with a lower or undefined SLA and no fallback.

**What breaks:** Availability multiplies, not averages: depending synchronously on N services each up 99.9% caps you near 0.999^N, not 99.9%. Any uncovered dependency silently caps the whole system's ceiling — a purely mathematical trap, not fixable by better code downstream.

**What the code must do:** Decouple from lower-tier dependencies (circuit breaker + graceful degradation) so their outage removes only the dependent feature, not the whole system. Write SLAs per-feature, not per-system.

**How to verify:** Enumerate every outbound dependency (including infra ones people forget) with its SLA, and confirm no feature-level claim exceeds the weakest uncovered link in its call path.

## Cluster C: Blast-Radius Containment

**Chain Reactions. Trigger:** A homogeneous fleet sharing one latent defect (leak or load-triggered race). **What breaks:** One instance dies, its traffic redistributes, survivors leak/race faster under the extra load, failures accelerate (minutes apart, then seconds). A two-node pair is worst case: losing one exactly doubles the survivor's load.

**Shared resources / point-to-point fan-out. Trigger:** A shared coordinator (lock/cache manager) called by every node, or peer-to-peer connections between every pair of nodes. **What breaks:** Invisible below production scale — dev/QA runs 1–2 instances, so O(n²) connections or a single shared coordinator look fine, then collapse once the fleet is large. Can't be caught by scaled-down load tests; must be designed out ahead of time.

**Unbalanced Capacities. Trigger:** A large front end calling a much smaller back end sized only for its "normal" share of traffic. **What breaks:** The front end can always generate more demand than a disproportionate back end absorbs; any spike in usage rate (a promotion) overwhelms it even though average-case planning looked fine.

**What the code must do:** Bulkheads — separate pools per critical caller/tenant/feature, at whatever granularity (pool, CPU binding, whole farm) isolates the failure you care about — so one caller's overload can't starve another sharing the resource. Circuit breaker on the calling side of a small back end. Handshaking on the serving side: since HTTP/RMI don't signal "I'm overloaded, back off" natively, use a health-check endpoint callers poll first (costs a doubled connection) or fall back to a circuit breaker where handshaking isn't possible. Replace point-to-point comms with pub/sub or a queue once instance counts grow; make any shared coordinator itself scalable or give it a degraded fallback (e.g., optimistic locking when a distributed lock manager is unreachable).

**How to verify:** Confirm each critical consumer of a shared resource has its own pool. Load-test at 2–10x expected peak against the *most expensive* transaction specifically, and confirm degradation is graceful, not a hang. Test fan-out patterns at production-scale instance counts, not dev/QA's 1–2.

## Cluster D: Self-Inflicted and User-Driven Load

**Attacks of self-denial. Trigger:** A promotion/mass email/deep link bypassing CDN caching, sent to a wide audience, hitting a system sized for organic traffic. **What breaks:** "Limited" offers are never actually limited — aggregator sites redistribute them within minutes, producing an impulse far larger than the nominal audience; if not isolated in a bulkhead, it takes down unrelated traffic too. **What the code must do:** Route campaign traffic to a dedicated bulkhead pool, serve static "landing zone" pages instead of routing straight into dynamic logic.

**Session/user load. Trigger:** Any inbound request without a valid session cookie (bots, scrapers, misconfigured proxies) creating a *new* server-side session on a stateful tier. **What breaks:** Session memory is the most common hidden capacity ceiling — one client ignoring cookies and hammering one URL can generate thousands of sessions/second, exhausting RAM until the tier hangs. This is a self-inflicted DDoS requiring no malice. **What the code must do:** Keep sessions to identifiers/keys, not whole objects; set timeout from measured think-time, not a generic 30 minutes; treat sessions as a discardable cache, never the source of truth.

**How to verify:** Load-test with "noise" traffic that never sends cookies and hammers one URL — different from realistic click-path scripts, and load-test vendors won't generate this by default. Monitor session-creation *rate*, not just active-session count.

## Cluster E: Unbounded Accumulation

**Unbounded result sets. Trigger:** A query or ORM association traversal with no explicit row cap, on the assumption the data stays small. **What breaks:** True in dev/QA, false once production data accumulates — a table expected to hold hundreds of rows can silently reach millions, and "loop over all of it" exhausts memory, sometimes taking a whole fleet down together as every instance queries the same bloated table on restart. **What the code must do:** Cap explicitly (`LIMIT`/`TOP`/`ROWNUM`); no relationship is safe just because the producer intends to keep it small — treat unbounded children and audit tables as inherently suspect.

**Steady-state violations. Trigger:** No working data-purge routine, log rotation, or cache eviction policy at launch, on the theory it gets added "in six months." **What breaks:** Anything accumulating without matching removal saturates disk (I/O errors, sometimes a runaway exception-logging loop trying to log its own failure to log) or heap (worsening GC, then thrashing) — slow enough to ship in 1.0 and only get fixed under production pressure. **What the code must do:** Give every accumulating resource an application-aware purge routine (not a bolted-on DBA cron job — referential integrity and code assumptions about non-empty collections make blind deletion dangerous). Rotate logs by size. Bound every cache's size with a flush/invalidation policy (time-based is usually enough); hold cached payloads behind soft/weak references so GC can reclaim them under pressure. Minimize routine manual production access — every human touch is a chance to fat-finger an outage.

**How to verify:** Confirm a scheduled, tested purge job per accumulating table. Confirm log appenders rotate by size/time and that debug/trace logging can't ship to production (strip in the build, don't rely on a flag). Confirm every cache has a documented max size and eviction policy; watch hit rate — low hit rate means the cache costs more than it saves.

## Cluster F: Adversarial Testing

**Trigger:** Integration tests exercise only in-spec dependency behavior — timeouts, malformed responses, dropped connections are never simulated, because even a forced error from a real dependency stays within its documented contract.

**What breaks:** Nothing, in testing — that's the problem. The system passes every test, then hangs on the first production failure mode integration tests never modeled.

**What the code must do:** Build a dedicated hostile test harness per external dependency that misbehaves below the interface contract, at the transport layer: refuse connections, accept-and-never-reply, dribble one byte at a time, send garbage instead of the expected format. This differs from mocking, which stays inside the interface contract — hostile-harness testing operates where real production failures actually happen.

**How to verify:** Confirm a standing hostile double exists per network dependency type and is exercised under concurrent load, since many of these bugs only surface with many simultaneous callers.

## Cluster G: Capacity Killers Hiding in Ordinary Code

**Resource pool contention.** *Trigger:* pool sized below concurrent demand, blocking checkout. *Breaks:* throughput hits a "knee" where added concurrency is spent entirely waiting, and slower transactions holding resources longer compounds the contention. *Fix:* size near expected concurrent demand (watch the DB-side RAM multiplier across many small per-instance pools), bound checkout wait, expose pool metrics. *Verify:* plot % time blocked vs. concurrency under load — should be flat near zero up to expected peak.

**Handcrafted SQL / unindexed associations.** *Trigger:* multi-table hand joins, string-built WHERE clauses, ORM relationship traversal on an unindexed column. *Breaks:* fine on tiny dev data, table-scans once row counts grow — invisible until real volume exists. Watch for the ORM "N+1" pattern (one query for a list, one per member). *Fix:* index anything used as an ORM association target; keep handwritten SQL predictable enough for a DBA to tune. *Verify:* run query plans against production-scale (or scrubbed production) data, not fixtures.

**Chatty remote calls.** *Trigger:* remote APIs designed like local ones, needing several round trips per logical result. *Breaks:* each round trip costs roughly 1000x a local call; multiplied by N, an interface fine locally becomes minutes of latency cross-region, and holds the calling thread (and any locks it carries) the whole time. *Fix:* collapse multi-call interactions into one call returning exactly what the caller needs. *Verify:* count round trips per logical operation — anything scaling with collection size instead of O(1) is a latency multiplier.

**Wasted payload bytes / cookie bloat.** *Trigger:* generated responses with unnecessary whitespace/markup, or whole serialized objects stuffed into cookies. *Breaks:* extra bytes multiply — CPU to generate, memory to buffer at every hop, bandwidth everywhere, and longer-held server connections (a finite, shared resource). Client-controlled cookies also add a trust hole (tamperable, replayable-stale) and a deploy-coupling hazard (serialized shape must survive every future release). *Fix:* cookies carry identifiers only; real state stays server-side and untrusted client data gets validated, never treated as authoritative. *Verify:* compare response size/connection hold-time before and after trimming under load; audit anything in a cookie for tamper impact on business logic.

**Reload-button non-idempotency.** *Trigger:* a slow response causes retry/resubmission while the original request is still in flight; server code assumes one request equals one attempt. *Breaks:* two in-flight copies can deadlock each other or double-apply a side effect. Serializing by source IP makes it worse (CDNs/proxies share IPs across real users). *Fix:* design transactional endpoints to be safely re-invocable — idempotency keys, upserts, request-ID dedupe. *Verify:* fire the same logical request concurrently and confirm no deadlock, no duplicated side effect.

**Session bloat / chatty AJAX** are variants of the same theme: don't poll on a timer for cosmetic behavior; keep async payloads to data only, not markup; make sure every async call still carries the session identifier or it silently mints a new session per call.

## Cluster H: Capacity Patterns to Apply Deliberately

**Pool connections** — establishing one costs hundreds of milliseconds, so reuse — but always with a bounded checkout (Cluster A) and sizing from real concurrency, not guesswork (Cluster G).

**Cache deliberately.** A cache is a bet that "generate once, reuse many" beats "generate every time" — skip caching items that are cheap to produce or rarely reused (bookkeeping can cost more than it saves), always bound size, always have an invalidation trigger. Across instances, plan the invalidation-broadcast path (point-to-point notification doesn't scale past a handful of nodes) and avoid every instance reloading the same invalidated item simultaneously.

**Precompute what changes rarely but is read constantly.** If content changes on the order of hours/days but is served millions of times, render once on the change event and serve the static artifact; use a "punch-out" for the small personalized fraction instead of forcing the whole page dynamic.

**Re-tune the runtime after every release.** Managed-runtime tuning (GC, allocators) is workload-dependent and must be validated under production-scale traffic; a release that shifts allocation or traffic patterns invalidates prior tuning, so this is recurring, not one-time.

## Cluster I: Operational Readiness the Code Itself Must Provide

**Fail fast at startup.** Refuse to finish starting — and say why — if a required dependency (DB, schema version, config) is unavailable at boot, rather than starting "successfully" and failing on the first real request. Don't accept traffic until initialization (including pool warm-up/validation) is fully complete.

**Drain on shutdown.** Stop accepting new work, finish in-flight transactions, then exit, bounded by a timeout so shutdown itself can't hang forever. Abrupt termination mid-transaction is a self-inflicted version of the same failure this whole document is about.

**Keep environment-specific config out of the deploy artifact.** Anything that varies by environment (hosts, ports, credentials) must live outside the code/install tree that gets overwritten on every deploy, and separate from wiring/DI config ops shouldn't touch at all — mixing the two turns a one-line password rotation into a giant risky file edit. Name properties for what they do, not their type.

**Make operational actions callable, not just clickable.** Anything an operator needs under incident pressure — reset a breaker, drain a pool, disable a feature, restart one failing component instead of the whole fleet — should be scriptable (CLI/API), since a GUI requiring manual clicks across N servers won't get used mid-incident, and restarting one broken subsystem in place beats a full-fleet restart by orders of magnitude.

**Least privilege and secrets.** Nothing runs as root/admin beyond the one moment that strictly requires it (e.g., binding a privileged port), after which privileges drop. Credentials live outside the install directory and source control, restricted to the owning process's user, and ideally can't leak through a core dump.

## Cluster J: What the Code Must Emit for Operators to Survive an Incident

Transparency is designed in, not bolted on — and serves four distinct audiences no single log stream or dashboard covers alone: **historical** trend (needs a queryable store, not just log files), **predictive** capacity projection (built from historical correlation, invalidated by every release), **present status** (a handful of health states — not just up/down, but also "too much of a good thing," since flooded-yet-technically-responding is its own failure mode), and **instantaneous behavior** during an active incident (thread dumps, live metrics).

**Log for the operator at 3 a.m., not the developer who wrote it.** Reserve error/critical severity for conditions needing operator action (tripped breaker, failed DB connection) — routine input-validation failures aren't errors. Never ship debug/trace logging to production by default; strip it in the build pipeline rather than trusting a flag. Use a stable message catalog with unique codes so operators can runbook-lookup problems and "what exactly did it say" stops being ambiguous mid-incident. Put a correlation/transaction ID on every line touching one logical request so a huge log can be grepped to one flow. Favor a single-line, columnar, severity-first format a human eye can scan — multi-line or unstructured formats defeat both human scanning and `grep`.

**Expose broad internal state, keep policy external.** You can't predict which internal metric will matter later (pool high-water marks, breaker transition counts, cache hit rate, per-integration-point error counts), so expose generously — but keep thresholds/alerting policy in configuration outside the app, since policy changes far faster than metric-emitting code should.

**Tune alert thresholds from real data, or they get ignored.** A threshold set too loose or noisy trains operators to dismiss real signals with the noise; base thresholds on observed historical norms, not guesses, and tighten as data accumulates.

**Transparency without a review cadence is theater.** A metric nobody reviews on a schedule provides no defense — pair every exposed metric with a recurring review loop (weekly error/ticket review, monthly volume trend), and retire metrics once they stop producing decisions.

## Cross-Reference: Antipattern → Defending Pattern(s)

| Antipattern | Primary defense | Secondary defense |
|---|---|---|
| Integration Points (unbounded remote call) | Timeouts | Circuit Breaker |
| Blocked Threads (pool exhaustion) | Timeouts | proven concurrency primitives, not hand-rolled pools |
| Cascading Failures | Circuit Breaker | Timeouts, async/message decoupling |
| Slow Responses | Fail Fast | Timeouts |
| SLA Inversion | decouple from low-SLA dependency | Circuit Breaker, per-feature SLAs |
| Chain Reactions | Bulkheads | fix the underlying leak/race |
| Scaling Effects (fan-out, shared coordinator) | pub/sub over point-to-point | Bulkheads |
| Unbalanced Capacities | Handshaking (callee throttles caller) | Circuit Breaker, Bulkheads |
| Attacks of Self-Denial | Bulkheads (isolate promo traffic) | Fail Fast on the dedicated pool |
| Session/user load | small, short-lived sessions | — |
| Unbounded Result Sets | explicit query limits | Timeouts (stopgap only) |
| Steady-state violations | purge, rotate, bound caches | — |
| Resource Pool Contention | sized pool + bounded wait | Bulkheads |
| Handcrafted SQL / unindexed queries | indexing + production-scale testing | — |
| Chatty integration calls | coarser interfaces / async decoupling | — |
| Cookie/payload bloat | identifiers-only client state | — |
| Reload-button retries | idempotent transaction design | — |
| Vendor library blocking | hostile Test Harness (find it pre-prod) | Timeouts |

Every pattern above also depends on Transparency to be worth anything: a breaker that doesn't log its transitions, a bulkhead whose pool metrics aren't exposed, or a fail-fast path that doesn't surface *why* it failed are all half-implemented — the pattern prevents the outage, but without transparency nobody can verify it's actually working or learn from the near-miss.
