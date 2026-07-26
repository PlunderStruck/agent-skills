# Keeping Two Stores in Sync

Applies whenever one logical change must land in more than one place: database plus cache, database plus search index, database plus data warehouse, service plus service, table plus denormalized copy.

## Why dual writes fail

```
db.update(x, value)
searchIndex.update(x, value)
```

This looks fine and is broken in two independent ways.

**Race.** Two clients write concurrently. Client 1 sets `x = A`, client 2 sets `x = B`. The database happens to process 1 then 2 and ends at `B`. The search index happens to process 2 then 1 and ends at `A`. The two stores now disagree **permanently**, and no error was raised anywhere. Nothing in the system will notice or repair it.

**Partial failure.** The first write succeeds, the second throws. Now they disagree, and the obvious fix — wrap both in a distributed transaction — is expensive and brings its own failure modes (see below).

The root cause is that there is no single leader. The database orders its own writes; the index orders its own writes; neither follows the other. It is multi-leader replication with no conflict resolution.

## The fix: one leader, everything else derived

Make one store the system of record. Every other copy becomes a follower that consumes an ordered log of changes from it. A single log means a single order, which is what makes the copies converge.

Two ways to produce that log:

### Change data capture (CDC)

Read the database's own replication log and republish it. Debezium and Maxwell parse the MySQL binlog; Bottled Water and Postgres logical decoding handle Postgres; Mongoriver reads the MongoDB oplog; GoldenGate covers Oracle. Kafka Connect wires these into a log-based broker that preserves order.

Trade-offs:
- Asynchronous by design — the source doesn't wait for consumers, so all replication-lag caveats apply to the derived stores.
- Database triggers can implement CDC but are fragile and impose real overhead. Prefer log parsing.
- Schema changes in the source are the hard part; check how your tool handles them.

### Transactional outbox

When you don't control the infrastructure enough for CDC: within the same transaction as the business write, insert a row into an `outbox` table. A separate process reads the outbox in order and publishes. The transaction gives you atomicity between the change and the intent to publish; the outbox reader gives you ordering and retry.

This is usually the right pattern in an ordinary application codebase, and it composes with the idempotency rules — put the request ID in the outbox row.

## Bootstrapping a new derived store

A change log alone is not enough to build a new index — it's missing everything not recently modified. You need either:

- **A consistent snapshot** taken at a known log position, then the log applied from that position forward. The position matters; without it you cannot tell where to resume.
- **A log-compacted topic**, where the broker retains only the most recent value per key. Then a new consumer starting at offset 0 sees the current state of every key, and no separate snapshot is needed. Kafka supports this; deletions must be represented as tombstones so they survive compaction.

If you find yourself planning "we'll re-run a full sync job nightly to fix drift," that's a signal the sync design is wrong — nightly repair means up to 24 hours of known-wrong data, and the repair job itself races with live writes.

## Why not distributed transactions

2PC/XA does solve the atomicity problem. It also brings:

- **In-doubt transactions hold locks.** Once a participant votes yes, it cannot unilaterally decide. If the coordinator crashes, those locks are held until it recovers — potentially forever if its log is lost or corrupted. Other transactions touching those rows block. Large parts of the application become unavailable.
- **The coordinator is a database.** Its log is durable state that must be recovered to resolve in-doubt transactions. If the coordinator library runs inside your application server, that server is no longer stateless and can no longer be freely added and removed.
- **Failure amplification.** Every participant must respond for the transaction to commit. Adding participants lowers total availability — the opposite of what fault-tolerance work is supposed to achieve.
- **Lowest common denominator.** XA cannot detect deadlocks across heterogeneous systems and cannot work with serializable snapshot isolation, because both would require cross-system protocols that don't exist.
- **Cost.** Distributed transactions in MySQL have been measured at over 10× slower than single-node, largely from extra `fsync`s and round-trips.

Escape hatches like XA "heuristic decisions" let a participant guess. That is a euphemism for breaking atomicity.

Database-*internal* distributed transactions (one vendor coordinating its own nodes — VoltDB, MySQL NDB, Spanner, FoundationDB) avoid most of this and are much more reasonable. The warning is specifically about heterogeneous 2PC spanning different technologies.

**Default recommendation:** log the change once, atomically, in one place; derive everything else; deduplicate by request ID. You get comparable correctness with better performance and far better operational behavior.

## What to check in the code

- Any request handler that writes to two clients/SDKs/connections. Ask which one is authoritative.
- Cache invalidation that happens next to a DB write rather than downstream of it.
- Search index or denormalized table updates scattered across many call sites — each is a place someone will forget.
- "Sync" cron jobs that reconcile two stores; they're evidence of a dual-write design.
- Any use of XA, `javax.transaction`, or a transaction manager spanning a DB and a broker.
