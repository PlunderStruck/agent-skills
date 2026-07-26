# Index Design

## Composite column ordering

A composite index is one structure sorted by the first column, then sub-sorted by the second within each first-column value, and so on. Everything about column ordering follows from that.

**Querying a non-leading column doesn't work.** With an index on `(customer_id, status)`, a query filtering only on `status` cannot binary-search the tree — there's no value for the leading column to descend into. Full scan.

**Equality before range.** This is the highest-value ordering rule. A range predicate spans multiple values of its column, so any *later* column's ordering resets within each one and can no longer narrow the search.

```sql
-- index (created_at, customer_id): range leads, so customer_id only filters
WHERE created_at BETWEEN ? AND ? AND customer_id = ?

-- index (customer_id, created_at): both become access predicates
```

The general ordering: **equality columns first, then the range or sort column.** At most one range column can be usefully bounded, and it goes last.

**Confirm:** both conditions appear as access/seek predicates, not one access plus one filter.

## Covering indexes and index-only scans

When every column a query touches — SELECT list, WHERE, JOIN — lives in one index, the engine never visits the table at all. That skips the random-I/O row fetch entirely, which is the largest single win available for queries that would otherwise touch many scattered rows.

Two ways to get there:

- **Widen the key** — adds the column to the sort order and counts against key-size limits.
- **Include non-key columns** — Postgres and SQL Server support `INCLUDE (...)`. These live only at the leaf level, don't affect sort order, and don't count against key limits. Cheaper to maintain; prefer this when the column is only needed for output.

Trimming `SELECT *` down to actually-used columns is often what makes covering feasible in the first place.

**Confirming an index-only scan:**

| Engine | Signal |
|---|---|
| Postgres | `Index Only Scan` with `Heap Fetches` near zero |
| Oracle | No `TABLE ACCESS BY INDEX ROWID` after the index scan |
| MySQL | `Using index` in EXPLAIN's Extra column |
| SQL Server | Index Seek/Scan with no Key Lookup or RID Lookup |

Postgres caveat: nonzero `Heap Fetches`, or a plain `Index Scan` where you expected index-only, usually means a stale visibility map. Run `VACUUM` before concluding the index is wrong.

**Watch for silent regressions.** Adding a WHERE condition on a column absent from the covering index kills the index-only path without any error. Recheck the plan after edits near one.

## Partial and filtered indexes

When a small subset of values is queried far more than the rest — pending orders among mostly-completed ones — a full-column index spends space and write cost on rows nobody looks up that way.

```sql
CREATE INDEX ON orders (customer_id) WHERE status = 'pending';
```

Support varies more than usual:

- **Postgres** — full support, as above.
- **SQL Server** — same syntax, called a filtered index, but the predicate can't contain functions or `OR`.
- **DB2 LUW** — approximate with `EXCLUDE NULL KEYS`.
- **Oracle** — no direct syntax. Emulate with an expression index whose function returns NULL for excluded rows, exploiting Oracle's NULL-omission behavior (see [query-shapes](query-shapes.md)).
- **MySQL** — no equivalent.

The query's WHERE clause must *provably imply* the index's WHERE clause for the optimizer to use it. "Logically overlapping" isn't enough.

## Clustered versus heap tables

A **heap** table stores rows wherever there's room; indexes hold physical row pointers. A **clustered** or index-organized table stores the row data inside the primary key's B-tree itself.

The tradeoff is entirely about secondary indexes. On a clustered table, primary-key lookups are free — the row is already there. But secondary indexes can't store physical pointers, because rows move as the clustering key is maintained. They store the clustering key instead, so a secondary lookup costs **two tree traversals**: secondary index to clustering key, then clustering key to row. Secondary indexes also grow larger, since every entry carries the full key.

Guidance: cluster tables dominated by primary-key access. Tables needing several secondary indexes usually do better heap-organized with covering indexes.

Defaults differ:

- **SQL Server** — clustered by default; opt out with `NONCLUSTERED` on the PK.
- **MySQL/InnoDB** — always clustered on the PK, no opt-out. Secondary indexes implicitly carry the PK.
- **Oracle** — heap by default; opt in with `ORGANIZATION INDEX`.
- **Postgres** — no true index-organized table. `CLUSTER` is a one-time reorganization that drifts back out of order as writes continue.

## Indexing for joins

The join algorithm determines whether an index helps at all.

**Nested loop** — the outer row set is probed row by row against the inner table. Cost is outer rows × inner lookup cost, so **an index on the inner table's join column is essential**; without it each probe is a full scan. This is the N+1 pattern in database form.

**Hash join** — the smaller input is hashed into memory and the other probed against it. **Indexing the join columns does not help** — the hash table replaces the lookup. What helps is shrinking the inputs: trim the SELECT list (hash table size scales with selected columns, not just rows), and index columns used in independent filters that reduce an input before hashing. MySQL only gained hash joins in 8.0.18; earlier versions fall back to nested loops even when hashing would be cheaper.

**Sort-merge join** — both inputs sorted on the join key, then merged in one pass. Free when index order already provides the sort. If neither side is pre-sorted, the double sort usually loses to a hash join. Explicit sort steps feeding a merge mean the ordering didn't come from an index. MySQL has no sort-merge operator at all.

## The write-side cost

Every index is a tax on every write touching it.

**INSERT** — each index needs its own B-tree insert: walk to leaf, insert, possibly split with cascading parent updates. This is typically the *dominant* cost of an insert. Going from zero indexes to one can be an order of magnitude slower; each additional index adds real but smaller marginal cost.

**UPDATE** — only indexes on the columns actually changed are touched, as a delete-plus-insert pair each. This makes targeted `SET` clauses matter: some ORMs historically wrote every mapped column on every save, silently maintaining every index on every update. Check what your ORM emits.

**DELETE** — structurally like insert, removing the entry from every index. Postgres differs: MVCC marks the row dead and defers physical index cleanup to `VACUUM`, so delete throughput is decoupled from index count — at the cost of dead tuples accumulating until vacuumed.

There's no universal threshold for "too many indexes." Index what is actually filtered, joined, or sorted on, and prefer one well-ordered composite over several overlapping single-column indexes.

## Testing at scale

A partially-bounded index and a fully-bounded one produce identical-looking plans on development data — both say "index used." They diverge completely as rows grow: one stays roughly flat, the other degrades linearly.

Test at production-scale row counts with realistic concurrency. Don't stop at "an index is used" — confirm which conditions are access versus filter, because that split is what predicts scaling. Re-run at ten and a hundred times current volume and compare growth.
