---
name: distributed-data
description: Apply data-correctness rules when writing or reviewing code that touches shared state — SQL/ORM queries and transactions, read replicas, caches, message queues and consumers, background jobs, webhooks, retries, schedulers, or any write that must stay in sync with a second store. Catches stale-replica reads, isolation-level races (lost update, write skew, phantoms), non-idempotent retries, dual-write divergence, wall-clock and lease misuse, and hot partitions.
---

# Distributed Data Correctness

## Purpose

Concurrency and partial-failure bugs do not announce themselves. They pass code review, pass tests, and pass staging — because tests run on one node with one client and no faults. They surface in production as silently lost writes, double charges, drifted caches, and data that two systems disagree about forever.

This skill is a set of failure modes bound to the code that causes them. Use it while writing the code, not after the incident.

## When this applies

Any of these in the code you are writing or reviewing:

- A read that follows a write, against anything replicated or cached
- A `SELECT` whose result decides whether to `INSERT`/`UPDATE`
- A read-modify-write in application code
- A retry, a queue consumer, a webhook handler, a cron job
- A write to two stores (DB + cache, DB + search index, DB + third-party API)
- `now()`, `Date.now()`, timestamps used for ordering, TTLs, leases, locks
- A partition/shard key, or a key derived from a timestamp

## Triage

| What you're writing | Failure to check | Reference |
|---|---|---|
| Read after write; read replica; cache | Stale read, non-monotonic read | [replication-and-staleness](references/replication-and-staleness.md) |
| Transaction; `SELECT` then write; counter update | Lost update, write skew, phantom | [isolation-and-races](references/isolation-and-races.md) |
| Retry, queue consumer, webhook, job, payment | Duplicate execution | [idempotency-and-retries](references/idempotency-and-retries.md) |
| Write to DB *and* cache/index/other service | Permanent divergence | [keeping-stores-in-sync](references/keeping-stores-in-sync.md) |
| Timestamps, TTLs, leases, distributed locks | Silent write loss, split-brain | [clocks-leases-fencing](references/clocks-leases-fencing.md) |
| Shard keys, secondary indexes, pagination | Hot partition, scatter/gather | [partitioning-and-hotspots](references/partitioning-and-hotspots.md) |
| Stream/queue consumers, windowed aggregation | Replay, event-time skew | [streams-and-queues](references/streams-and-queues.md) |
| Multi-step operations spanning transactions; orchestrators | Missing compensation, invisible in-flight state | [sagas-and-compensation](references/sagas-and-compensation.md) |

## Rules that apply without loading anything

**1. A timeout tells you nothing.** If a request times out, the write may have succeeded. Code that treats timeout as failure and retries will duplicate the effect. Every retryable operation needs a client-generated ID that survives the retry.

**2. Check-then-act across a transaction boundary is a race.** `SELECT count(...)` → decide → `INSERT` is not safe under the default isolation level of Postgres, MySQL, Oracle, or SQL Server. Push the invariant into a DB constraint, or use serializable isolation. Do not rely on the check.

**3. Read-modify-write in application code loses updates.** `value = read(); write(value + 1)` drops a concurrent increment. Use an atomic DB operation (`SET n = n + 1`), or `SELECT ... FOR UPDATE`, or compare-and-set. ORMs make the unsafe version the natural one to write.

**4. Never order events across machines by wall clock.** Clocks drift, jump backward on NTP correction, and stall inside VMs. Last-write-wins by timestamp silently discards writes. Order by a sequence from a single log, or by version vectors.

**5. Writing to two stores in one request will diverge.** Two clients interleaving produces permanent inconsistency with no error raised. Make one store authoritative and derive the other from its change log or an outbox table.

**6. `w + r > n` is not a guarantee.** Quorum reads do not give you read-your-writes or monotonic reads. Sloppy quorums, concurrent writes, and partial write failures all break the overlap argument.

**7. A lease you hold can expire mid-function.** GC pauses, VM suspension, and page faults stop a thread for seconds to minutes. `if (lock.valid()) { write(); }` is unsafe. The *resource* must reject stale writes via a fencing token; a client-side check is worthless.

**8. Constraints beat application checks.** A `UNIQUE` constraint holds under weak isolation. An application-level "does this exist yet?" check does not.

## How to use this

1. Read the diff for the triggers in *When this applies*.
2. For each hit, open the matching reference and check the code against its failure modes.
3. Name the guarantee the code actually needs — "this read must reflect the user's own write" — rather than reaching for the strongest available setting.
4. State residual risk explicitly. "This is safe at read-committed because the invariant is enforced by a unique constraint" is a real answer. "We use transactions" is not.

## What this skill will not tell you

Whether the tradeoff is worth it. Serializable isolation, synchronous replication, and consensus all cost latency and availability. This skill identifies what breaks and what fixes it; the call on whether the failure matters for a given feature belongs to whoever owns the product.
