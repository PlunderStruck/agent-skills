# Partitioning and Hot Spots

Applies to shard/partition key selection, primary key design, secondary indexes on partitioned data, and pagination.

## The goal and the failure

Partitioning spreads data and load across nodes. It works only if the load actually spreads. A partition with disproportionate load is a **hot spot**, and one hot partition caps the throughput of the entire cluster — nine idle nodes and one saturated one.

## Key choice determines everything

### Key-range partitioning

Keys sorted, each partition owns a contiguous range. Range scans are efficient; you can treat the key as a concatenated index and fetch related records in one query.

The trap: **anything monotonically increasing concentrates all writes on one partition.** A timestamp primary key means every write today goes to today's partition. Same for autoincrement IDs and sequential ULIDs at high write rates.

Fix: put something with cardinality first. `(sensor_id, timestamp)` spreads writes across sensors while keeping each sensor's readings ordered and range-scannable. The cost is that a cross-sensor time-range query now needs one query per sensor.

### Hash partitioning

Hash the key, assign hash ranges to partitions. Distributes evenly, and destroys sort order — range queries must hit every partition.

Two things to verify:

- **Use a stable hash.** Java's `Object.hashCode()` and Ruby's `Object#hash` can return different values in different processes. That is fatal for partitioning and the bug is intermittent and awful.
- **Compound keys give you both.** Cassandra hashes only the first component of a compound primary key and uses the rest as a sort key within the partition. `(user_id, post_timestamp)` spreads users across the cluster while keeping one user's posts ordered and range-scannable on a single partition. This is usually the shape you want.

## Hot keys that hashing cannot fix

If all reads and writes target *the same key* — a celebrity account, a viral post, a global counter — hashing does nothing. Identical IDs hash identically.

Mitigation is manual and has real costs: append a random suffix to spread one key across N keys. Then

- every read must fan out to all N and recombine,
- you need bookkeeping to know which keys are split (you don't want this overhead on the other 99.99% of keys),
- and the split has to be applied and unapplied as keys heat up and cool down.

No mainstream data system does this automatically. If your design has a naturally hot key, decide deliberately whether to split it, cache in front of it, or make the operation commutative so it can be aggregated.

## Single-row hot spots, where partitioning has nothing left to split

Distinct from a hot *shard key*. A single row every transaction must touch — a running balance, a sequence number, a global inventory count — is already the finest granularity, so re-sharding does not apply. Throughput caps at roughly one transaction per average lock-hold-time on that row, no matter how many cores or connections you add.

Teams hit that ceiling, read it as a scaling problem, and reach for partitioning tools that cannot help.

In order of preference:

1. **Single atomic statement** — `UPDATE t SET n = n + 1` — so the lock is held for the statement, not the surrounding transaction.
2. **Touch the hot row last**, immediately before commit, minimising hold time. Sometimes means restructuring the method to defer the write.
3. **Optimistic check** where the exact value need not be locked — act without locking, re-check the condition at commit, abort if it moved. Trades a rare late abort for not holding the lock across the transaction's life.
4. **Batch per worker** and flush periodically, where the precise moment of update does not matter.
5. **Split hot and cold columns** into separate rows so readers of the cold ones aren't queued behind writers of the hot ones.

**Check.** For any row identified by lock-wait telemetry as a ceiling, confirm which of these is in use — or that accepting the ceiling was a decision someone actually made.

## Secondary indexes

Two designs, opposite trade-offs. Know which one your store uses.

**Document-partitioned (local) index** — each partition indexes only its own documents. Used by MongoDB, Riak, Cassandra, Elasticsearch, SolrCloud, VoltDB.

- Writes are cheap: only one partition is touched.
- Reads are **scatter/gather** — query every partition and merge. Even in parallel, this is subject to tail-latency amplification: the query is as slow as the slowest partition, so p99 of the query approaches p99.9 of a single partition. Combining several secondary-index filters in one query makes this worse.

**Term-partitioned (global) index** — the index is partitioned by the indexed value, independently of the primary key.

- Reads hit one partition. Efficient.
- Writes touch multiple index partitions, so keeping the index synchronously consistent would require a distributed transaction. In practice **updates are asynchronous**: DynamoDB's global secondary indexes are typically current within a fraction of a second but may lag under fault conditions.

The practical consequence for application code: **a global secondary index is a replica with lag.** A read-your-writes flow that writes a row and then finds it via a global secondary index can miss it. Query by primary key when you need your own write back.

## Rebalancing

- **Never `hash mod N`** where N is the node count. Changing the node count reshuffles nearly every key. Assign hash *ranges* to a fixed set of partitions instead, and move partitions between nodes.
- **Fixed partition count** (Riak, Elasticsearch, Couchbase, Voldemort): create many more partitions than nodes up front; a new node steals whole partitions. The count is usually fixed forever, so it caps your maximum node count — pick it with growth in mind, but not so high that per-partition overhead dominates.
- **Dynamic partitioning** (HBase, RethinkDB, MongoDB 2.4+): partitions split when they exceed a size threshold. Adapts to data volume, but an empty database starts with one partition, so early writes are single-node until the first split. Pre-splitting fixes this if you can predict the key distribution.
- **Automatic rebalancing plus automatic failure detection is dangerous.** An overloaded node responds slowly, gets declared dead, and its load is moved — adding load to an already-struggling cluster and potentially cascading. A human in the loop for rebalancing is a defensible choice.

## Pagination

Offset-based pagination (`LIMIT 20 OFFSET 40`) over data that changes between requests skips and repeats rows. On partitioned stores it's also expensive, since the offset must be resolved across partitions.

Use keyset/cursor pagination — `WHERE (created_at, id) < (?, ?) ORDER BY created_at DESC, id DESC LIMIT 20` — with a tiebreaker column to make the sort total. This is stable under concurrent writes and maps onto a single partition when the sort key aligns with the partition key.

## What to check in the code

- Primary keys and partition keys that lead with a timestamp, autoincrement, or any monotonic value.
- Hash functions whose stability across processes and versions hasn't been verified.
- Queries filtering on a non-partition-key column — identify whether that's a scatter/gather.
- Reads that expect to find a just-written row via a secondary index.
- `OFFSET`-based pagination on mutable data.
- Any known celebrity/tenant/global-counter key, and whether the design acknowledges it.
