# Blast Radius and Containment

Applies to shared resources, multi-tenant paths, fleet-wide behavior, and anything a traffic spike will hit.

## Chain reactions

A fleet of identical instances shares every latent defect — a memory leak, a load-triggered race, a resource exhaustion bug. When one instance dies, its traffic redistributes to the survivors, which now hit the defect *faster* because they are carrying more load. Failures accelerate: minutes apart, then seconds.

The worst configuration is a pair. Losing one instance exactly doubles the survivor's load, which is usually enough to trigger whatever killed the first one.

Containment does not fix this — the defect is homogeneous, so isolation only buys time. The real fix is finding the leak or race. Bulkheads limit how fast it spreads while you do.

## Bulkheads

Partition a shared resource so that one consumer's overload cannot starve the others. The partition can be at whatever granularity matches the failure you care about: separate connection pools, separate thread pools, separate instances, separate clusters.

The canonical case: one misbehaving caller consumes every connection in a shared pool, and every other caller — doing nothing wrong — starts failing. With per-caller pools, the misbehaving one exhausts its own allocation and stops there.

Bulkheads cost capacity by design. Reserved-but-idle resources are the price of isolation. That trade is usually worth it for the paths that earn money.

## Scaling effects

Some architectures work at development scale and collapse at production scale. These cannot be caught by scaled-down load testing, because the defect *is* the scale.

- **Point-to-point communication between instances.** Connections grow as the square of the fleet size. Two instances means one connection; fifty means over a thousand. Fine in QA with two nodes, fatal at fifty. Replace with pub/sub or a broker before you need to.
- **A shared coordinator** — a lock manager, a cache invalidation service, a sequence generator — called by every instance. It looks harmless when three nodes call it and becomes the bottleneck and single point of failure at thirty. Either make it independently scalable or give callers a degraded fallback, such as optimistic locking when the lock service is unreachable.

Test these at production-scale instance counts, or design them out ahead of time. There is no third option.

## Unbalanced capacities

A large front end calling a much smaller back end, where the back end was sized for its *normal* share of traffic.

The front end can always generate more demand than the back end can absorb. Average-case capacity planning hides this: the numbers work until a promotion, a retry storm, or a batch job shifts the ratio, and then the small side is overwhelmed instantly.

Defenses:

- **Circuit breaker on the calling side**, so the front end stops hammering a back end that is already failing.
- **Handshaking on the serving side** — the callee tells callers to back off. HTTP and RPC have no native mechanism for this, so it usually means a health or capacity endpoint that callers poll before committing to expensive work. The cost is a doubled connection count.
- Where handshaking is not feasible, the breaker is the fallback.

## Self-inflicted load

Promotions, mass emails, and push notifications produce impulse traffic shaped nothing like organic traffic. Two properties make it dangerous:

- **Deep links bypass caching.** A campaign URL pointing straight at dynamic logic skips every CDN and cache layer you rely on.
- **"Limited" offers are never limited.** Aggregator sites and social sharing redistribute them within minutes, so the audience is not the list you emailed.

Route campaign traffic into a dedicated bulkhead, and serve a static landing page rather than routing directly into dynamic request handling.

## Session load

On any stateful tier, a request without a valid session identifier creates a *new* server-side session. Bots, scrapers, misconfigured proxies, and clients that ignore cookies therefore mint sessions at whatever rate they can issue requests.

This is the most common hidden capacity ceiling. One client hammering a single URL without cookies can create thousands of sessions per second until memory is exhausted — a self-inflicted denial of service requiring no malice.

Mitigations:

- Sessions hold identifiers and keys, not whole object graphs.
- Session timeout derives from measured think-time, not a copied default.
- Treat sessions as a discardable cache, never the source of truth — so losing them degrades experience rather than corrupting state.
- Monitor session *creation rate*, not just active count. The rate is the leading indicator; the count is the lagging one.

## What to check in the code

- Does every critical consumer of a shared pool have its own partition, or do they all draw from one?
- Are there point-to-point connections between instances that grow with fleet size?
- Is there a shared coordinator on the hot path, and what happens when it is unreachable?
- Does the front end have a breaker on calls to any smaller back end?
- Where does campaign or notification traffic land — a dedicated path, or straight into the main request pool?
- Can a request without a session cookie create a session? At what rate?
- Are load tests run against the *most expensive* transaction at multiples of expected peak, or against an average click path at expected peak?
