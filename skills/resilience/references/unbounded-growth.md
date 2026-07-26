# Unbounded Growth and Capacity Cliffs

Applies to queries returning collections, caches, sessions, logs, connection pools, and anything that accumulates over time.

## Unbounded result sets

A query with no explicit row cap, or an ORM association traversal that loads a whole collection, on the assumption the data stays small.

That assumption holds in development, where the table has forty rows. In production the same table reaches millions, and the code that loops over all of it exhausts memory. The severe version takes down an entire fleet at once: every instance restarts, every instance queries the same bloated table, every instance dies.

Rules:

- Put an explicit limit on every query — `LIMIT`, `TOP`, `ROWNUM`, whatever your engine calls it. Paginate rather than fetching everything.
- No relationship is safe because the producer *intends* to keep it small. Child collections, audit trails, event logs, and comment threads are all inherently suspect.
- Watch for the N+1 pattern: one query to fetch a list, then one query per element. This scales linearly with a number nobody is bounding.

A timeout is a stopgap here, not a fix. It stops the query eventually; it does not stop you from asking for a million rows.

## Steady state

Anything that accumulates without a matching removal path eventually saturates something. Disk fills and I/O starts failing — occasionally producing a runaway loop where the logging system tries to log its failure to log. Heap fills and garbage collection degrades until the process thrashes.

The reason this ships in version 1.0 is that it accumulates slowly enough to look fine for months, and the purge routine is always something to add later.

Give every accumulating resource a removal path **at launch**:

- **Data purging** belongs in the application, not a database cron job. Blind deletion breaks referential integrity, and application code often assumes collections are non-empty. The purge needs to know the domain.
- **Log rotation** by size or age, configured at the appender.
- **Cache eviction** with a bounded maximum size and an explicit policy. Time-based expiry is usually sufficient.
- **Session expiry** derived from real think-time.

Where the platform supports it, hold cached payloads behind weak or soft references so the collector can reclaim them under memory pressure rather than letting the cache win against the application.

## Caching deliberately

A cache is a bet that generating once and reusing many times beats generating every time. The bet does not always pay:

- Items that are **cheap to produce** or **rarely reused** cost more in bookkeeping than they save.
- **Low hit rate is a signal to remove the cache**, not to grow it. Measure it.
- Every cache needs a **bounded size** and an **invalidation trigger**. Unbounded caches are memory leaks wearing a performance costume.
- Across a fleet, plan the **invalidation broadcast path**. Point-to-point notification does not scale past a handful of nodes, and simultaneous invalidation makes every instance reload the same item at the same moment — a stampede against the thing you were protecting.

For content that changes rarely but is read constantly, precomputing on the change event and serving a static artifact beats caching a dynamic render. Handle the small personalized fraction separately rather than making the whole page dynamic for it.

## Pool contention

A pool sized below concurrent demand does not fail — it queues. Throughput hits a knee where added concurrency is spent entirely waiting, and slow transactions holding connections longer compound it.

- Size from measured concurrent demand, not intuition.
- Account for the multiplier on the far side: many application instances with small pools each can still overwhelm a database's connection limit and memory.
- Bound the checkout wait, and expose pool metrics — in-use count, wait time, high-water mark.

To verify: plot percentage of time blocked on checkout against concurrency under load. It should stay near zero up to expected peak and only rise beyond it.

## Query cost that only appears at scale

- **Unindexed association targets.** ORM relationship traversal on an unindexed column table-scans. Invisible on fixture data, crippling at volume.
- **Hand-built SQL** assembled by string concatenation is hard for anyone to tune later and easy to make unpredictable.
- Run query plans against production-scale data — or a scrubbed copy — rather than test fixtures. The plan on forty rows tells you nothing about the plan on four million.

## Chatty interfaces

A remote API designed like a local one needs several round trips per logical result. Each round trip costs roughly a thousand times a local call. Multiplied across a collection, an interface that is fine in-process becomes minutes of latency across regions — while holding the calling thread and any locks it carries.

Collapse multi-call interactions into a single call returning exactly what the caller needs. Count round trips per logical operation: anything scaling with collection size rather than staying constant is a latency multiplier.

## Payload and cookie weight

Extra bytes multiply at every hop — CPU to generate, memory to buffer, bandwidth to ship, and longer-held connections, which are a finite shared resource.

Cookies deserve specific attention. Putting serialized objects in them creates three problems: they are client-controlled and therefore tamperable and replayable, they bloat every single request, and the serialized shape must remain compatible across every future release or old cookies break new code. Cookies carry identifiers; real state stays server-side; anything arriving from a client is validated rather than trusted.

## What to check in the code

- Any query or association traversal without an explicit limit.
- Any accumulating store — table, log, cache, session map — without a purge, rotation, or eviction policy that actually runs.
- Cache definitions without a maximum size, and caches whose hit rate nobody measures.
- Debug or trace logging that can reach production. Strip it in the build rather than relying on a runtime flag.
- Pools without bounded checkout or exposed metrics.
- ORM associations on unindexed columns; N+1 query patterns.
- Remote interfaces whose round-trip count scales with collection size.
- Anything serialized into a cookie.
