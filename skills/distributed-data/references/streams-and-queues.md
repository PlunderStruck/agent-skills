# Streams, Queues, and Consumers

Applies to message-broker producers and consumers, event-driven handlers, stream processors, windowed aggregation, and anything computing metrics over a stream of events.

## Delivery semantics

Brokers redeliver a message when the consumer doesn't acknowledge it — which happens on crash, on timeout, and on a slow consumer that the broker gives up on. That is **at-least-once** delivery, and it is what you actually have regardless of what the marketing says.

So: **deduplication is the consumer's job.** See [idempotency-and-retries](idempotency-and-retries.md). The short version is that every message needs a stable identifier the consumer can check against work it has already done, and "the broker's message ID" is often not stable across redelivery — prefer an ID carried in the payload from the origin.

"Exactly-once" from a stream framework means exactly-once *within the framework's boundary*. Microbatching (Spark Streaming) and checkpointing (Flink) let the framework discard the output of a failed task and redo it. The moment output leaves — a database write, an outbound HTTP call, an email — the framework can no longer retract it, and a restart produces the side effect twice. Frameworks that extend the guarantee further (Google Cloud Dataflow, Kafka transactions) do it by keeping both state and messaging inside their own system; they do not cover your external calls.

Effectively-once is achievable, but through idempotence and fencing, not by trusting a config flag.

## A job queue on a plain table

Multiple workers polling `WHERE status = 'pending' ORDER BY id LIMIT 1` is the common way to avoid standing up a broker, and the naive version fails two ways: claim logic that serialises every worker behind whoever holds the lock on the next row — defeating the point of having workers — or a `SELECT` followed by a separate `UPDATE`, which races two workers onto one row.

**Use an atomic locking read that skips already-claimed rows**: `SELECT ... FOR UPDATE SKIP LOCKED` (Postgres, MySQL 8+) or `READPAST` (SQL Server), in a single statement that selects and marks claimed. No lock table, no backoff-and-retry polling loop.

## Poison messages need a bound

A consumer that throws on bad data *in the message itself* — not a transient fault — aborts, returns the message to the queue, and gets it redelivered forever.

Unbounded, one malformed message burns consumer capacity indefinitely, and it is invisible: nothing marks it as permanently stuck, so nobody looks until throughput visibly degrades.

**Configure a maximum delivery count** after which the message moves to a dead-letter queue, with a minimum interval between retries so a transient blip isn't misread as poison. Then give the dead-letter queue a defined owner and a handling path — parked-and-forgotten is the same outcome as dropped, with extra steps.

## Ordering

Order is preserved **within a partition**, not across partitions and not across topics. Two consequences:

- If two events must be processed in order, they must be routed to the same partition — which usually means partitioning by the entity they concern.
- A join or correlation across two streams has no ordering guarantee between them. If the result depends on which arrives first, the job is non-deterministic and re-running it on the same input can produce a different answer.

## Event time vs processing time

The clock on the machine doing the processing has nothing to do with when the event happened.

Windowing by processing time is simple and wrong whenever there's lag. The classic artifact: redeploy a consumer, it's down for a minute, it comes back and burns through the backlog. A rate metric computed on processing time shows a huge spike that never happened. Alerts fire; someone investigates a phantom.

Use the timestamp embedded in the event. This also makes reprocessing deterministic — the same input produces the same output, which is what makes replay-based recovery and bug fixes viable.

Sources of lag that make this matter: queueing, network faults, broker contention, consumer restarts, and deliberate reprocessing of history.

### Knowing when a window is complete

You can never be certain no more events for a window will arrive. Choose explicitly:

1. **Drop stragglers.** Track the count as a metric and alert when it becomes significant. Fine for most analytics.
2. **Publish corrections.** Emit an updated value for the window, and retract the previous output if downstream consumers need that.

Some systems support a watermark message ("no more events earlier than t"). With multiple producers, the consumer must track a threshold per producer, and adding or removing producers gets tricky.

### Untrusted client clocks

Events from mobile or browser clients carry the client's clock, which may be wrong by hours, deliberately. Log three timestamps — event time (device), send time (device), receive time (server) — and correct the event time by the offset between the last two.

## Window types

- **Tumbling** — fixed length, non-overlapping, each event in exactly one window. Round the timestamp down.
- **Hopping** — fixed length, overlapping by a hop interval. Implement as an aggregation over adjacent tumbling windows.
- **Sliding** — all events within an interval of each other; boundaries not fixed. Needs a time-sorted buffer with expiry.
- **Session** — no fixed duration; groups activity by the same entity until an inactivity gap. The usual choice for user-behavior analysis.

## Joins

- **Stream-stream (window join).** Correlating two event types by a shared key within a time window — search and click, request and response. Requires the processor to hold state (recent events indexed by key) and to decide what to emit when the window expires without a match. That "no match" output is often the point: click-through rate needs the searches with no click.
- **Stream-table (enrichment).** Attaching stored data to each event. Querying a remote database per message is slow and can overload the database; the standard approach is a local copy of the table kept current by change data capture. That makes it a join between two streams — the events and the table's changelog.
- **Table-table (materialized view).** Both sides are changelogs; the output is a continuously maintained view. This is how timeline/feed caches are maintained.

**Time dependence:** if a user updates their profile, which events join against the old version and which against the new? If the ordering across the two inputs is undetermined, the join is non-deterministic. Where correctness demands it, version the joined record (the data-warehouse "slowly changing dimension" pattern) and reference the version explicitly — an invoice should join the tax rate *as of the sale*, not the current one. Note this blocks log compaction, since old versions must be retained.

## Recovering state

Any consumer holding state — counters, aggregates, join buffers, local table copies — needs that state recoverable after a crash.

- Local state replicated to a durable log (Kafka Streams, Samza) or snapshotted to durable storage (Flink).
- Or rebuilt by replaying input: viable if the window is short, or if the state is a local copy of a table backed by a log-compacted changelog.
- Remote state avoids the problem but adds a network round-trip per message.

When failing over a consumer to a new node, **fence it** — the old node may not be dead, just paused. See [clocks-leases-fencing](clocks-leases-fencing.md).

## What to check in the code

- Consumer handlers with no deduplication. Ask what a redelivery does.
- Side effects (email, charge, external POST) in a consumer without an idempotency key.
- Windowing or rate calculation using the processing machine's clock.
- Consumers whose in-memory state would be silently wrong after a restart.
- Two events that must be ordered but are published to different partitions or topics.
- Consumer failover paths with no fencing token.
- Metric spikes that coincide with deploys — usually processing-time windowing, not a real event.
