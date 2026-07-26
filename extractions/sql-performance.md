# Extraction: sql-performance

**Source:** SQL Performance Explained — Markus Winand

> Operational rules distilled from the source, written in our own words. This is the intermediate layer between the source text and the published skill — denser than the skill, and the place detail survives for re-distillation.

---

# SQL Performance Rules for Index-Safe Query Writing

*Condensed from Markus Winand's "SQL Performance Explained" (use-the-index-luke.com). Each rule states the exact shape that triggers it, what the engine does instead of using the index, the fix, and how to confirm it in an execution plan.*

## Foundations: how an index lookup actually costs

A B-tree lookup has three cost components, and only the first is cheap and bounded: (1) walking root-to-leaf — grows with the *log* of row count, effectively flat regardless of table size; (2) walking the leaf node's linked list to collect every matching entry — grows with the *number of matches*; (3) fetching the underlying row for every matching entry, one random I/O each (unless the index is covering — see below). Steps 2–3 are where "using an index" quietly becomes a scan in disguise: an index matching thousands of rows isn't meaningfully faster than reading the whole table.

This is the vocabulary every rule below depends on. An **access predicate** bounds which leaf entries the engine walks (cheap). A **filter predicate** is evaluated after rows are already pulled — it discards rows without reducing the work spent fetching them. Every rule here is really the same question: does this predicate become access or filter? Plan syntax differs by vendor (Oracle labels `access()`/`filter()` explicitly; DB2 uses START/STOP vs. SARGable; SQL Server splits Seek Predicates vs. Predicate; Postgres/MySQL require inferring it from row-count drop-off between steps), but the concept is universal.

## Composite index column order

**Querying a non-leading column.**
- Trigger: index on `(customer_id, status)`; query filters only `WHERE status = 'pending'`.
- What happens: a composite index is one structure sorted by column 1, then sub-sorted by column 2 within each column-1 value. Without a value for column 1, the engine can't binary-search into the tree for column 2 — full scan.
- Fix: lead the index with `status`, or add a second index `(status)`, if that lookup pattern is common. Confirm: plan flips from full scan to index range scan once the leading predicate is supplied.

**Range predicate ahead of equality/sort columns — the highest-value fix in this document.**
- Trigger: index `(created_at, customer_id)`; query is `WHERE created_at BETWEEN ? AND ? AND customer_id = ?`.
- What happens: the range on `created_at` bounds the walk, but `customer_id` can no longer narrow *where* inside that range to look — it becomes a filter applied to every row in the date span.
- Fix: put equality columns before range columns: `(customer_id, created_at)` — both conditions become access predicates. Confirm: predicate section shows both as access/seek predicates, not one access plus one filter.

**Bind parameters vs. literals.**
- Trigger: literals interpolated into SQL text on every call, or a bind used against a highly skewed column.
- What happens: plan-cache engines (Oracle, SQL Server, Postgres prepared statements) re-optimize a literal-text statement every call; binds let the plan be reused. But a bind gives the optimizer no concrete value at plan time, so it assumes uniform distribution — risky if one value matches 10 rows and another a million.
- Fix: default to binds; for a known-skewed column, use a literal or a per-value-plan feature (Oracle adaptive cursor sharing, SQL Server `OPTIMIZE FOR`). Confirm: compare plans for different literal values of the skewed column — divergent access strategies mean one cached bind plan risks being wrong for some values.

## Functions and expressions wrapping the indexed column

**The core sargability rule.**
- Trigger: `WHERE UPPER(last_name) = 'SMITH'`, `WHERE price * 1.1 > ?`, any function/arithmetic applied directly to the indexed column.
- What happens: the engine treats a function as an opaque black box. The index stores raw values, not the transformed result, so it can't bound the search — full scan, function evaluated per row as a filter.
- Fix: move the function to the *literal* side (`WHERE price > ? / 1.1`), or build a matching expression index — native syntax in Postgres/Oracle (`CREATE INDEX ON t (UPPER(col))`); MySQL 5.7+/SQL Server need a generated/computed column plus an index on it, referenced by matching queries. DB2 LUW disallows UDFs in indexes entirely.
- Confirm: the wrapped column sits under filter before the fix; a plain range/seek on the expression index after.

**Implicit type conversion.**
- Trigger: string column compared to a numeric literal, e.g. `WHERE numeric_string_col = 42`.
- What happens: the engine must cast one side. If it casts the *column*, that's a function wrap — same failure, plus a correctness risk (numeric casts conflate `'42'`, `'042'`). Comparing a numeric column to a string literal is safe; only the literal converts.
- Fix: always cast the literal, never the column: `WHERE numeric_string_col = '42'`.
- Vendor notes: Postgres and DB2 reject cross-type comparisons with a type error. Oracle, MySQL/MariaDB, SQL Server, and SQLite silently cast — invisible until you check the plan for a CAST/CONVERT wrapping the column.

**Date truncation and column concatenation — the same failure, two common shapes.**
- Trigger: `WHERE TRUNC(created_at) = DATE '2026-07-25'` (or `DATE_FORMAT`/`TO_CHAR`), or `WHERE first_name || ' ' || last_name = ?` / `WHERE date_col + time_col > ?`.
- What happens: both wrap or derive from the indexed column(s), so even a composite index on the underlying raw columns can't act as an access predicate — full scan, expression evaluated as a filter.
- Fix (dates): rewrite as a half-open range on the raw column — `created_at >= DATE '2026-07-25' AND created_at < DATE '2026-07-26'`; never use inclusive `BETWEEN` for a day boundary, it mishandles rows with a time component. Fix (concatenation): store the derived value as a real (possibly generated) indexed column, or keep the concatenation but add a redundant sargable predicate on the leading raw column — works only if the concatenation preserves that column's sort order (e.g. ISO-8601 formatting).
- Confirm: the raw column drives an index range scan; the wrapped/derived expression shows up under filter, not access.

**User-defined functions in expression indexes.** Trigger: an index built on a UDF, e.g. `CREATE INDEX ON accounts (calc_age(birth_date))`, declared deterministic (Oracle `DETERMINISTIC`, Postgres `IMMUTABLE`). What happens: the engine trusts that declaration without verifying it — a function that actually reads the clock or other mutable state silently freezes stale values into the index instead of erroring. Fix: only mark a function deterministic if its output depends solely on its arguments; DB2 sidesteps the risk by disallowing UDFs in indexes entirely. Confirm: no plan signal exists for this — it's a code-review item, not something EXPLAIN shows.

**Over-indexing with narrow expression indexes.** Trigger: separate indexes on `UPPER(last_name)` and `LOWER(last_name)` because different call sites (or an ORM's implicit case-folding) query it both ways — each adds write-side maintenance (see below) for a problem one consistent expression would solve. Fix: standardize on one case-folding function codebase-wide; watch for ORMs (Hibernate is a known offender) silently injecting a case-folding call that doesn't match your index.

## LIKE and leading wildcards

- Trigger: `WHERE last_name LIKE '%mann'` (leading wildcard) vs. `WHERE last_name LIKE 'Wina%'` (trailing only).
- What happens: only the literal characters before the *first* wildcard act as an access predicate — a trailing-wildcard pattern behaves like a range scan bounded by that prefix. A leading wildcard leaves no usable prefix, so the scan degrades to a full scan with the whole pattern evaluated as a filter.
- Fix: avoid leading wildcards for indexed lookups; use full-text search for substring/suffix needs — Postgres `@@`/`pg_trgm`, MySQL `MATCH() AGAINST()`, Oracle Text/SQL Server `CONTAINS()`.
- Confirm: the LIKE pattern sits under filter (or a full scan) rather than a range/seek bounded by the literal prefix.
- Vendor note (easy to miss): most engines assume "no leading wildcard" when the pattern arrives as a bind parameter and still attempt a prefix scan. **Postgres is the opposite** — a bind-parameter LIKE pattern makes it assume a leading wildcard might be present and skip the index even for anchored patterns. Use a literal if index usage matters for a parameterized LIKE in Postgres.

## OR conditions and combining indexes

- Trigger: two range/inequality conditions on different columns, or an OR across two separately indexed columns, e.g. `WHERE status = 'A' OR region = 'west'`.
- What happens: a B-tree is one-dimensional — only one condition can ever bound the walk as a true access predicate. A second independent condition on a different column stays a filter predicate unless the engine combines two separate index scans (an index-merge/bitmap-combine strategy) and unions or intersects the row sets in memory — two tree traversals plus a merge, usually costlier than one composite index absorbing the more selective condition as an access predicate and leaving the other as a cheap filter.
- Fix: prefer a single composite index with the more selective condition leading; where an OR across independently indexed columns is unavoidable and both branches are selective, rewrite as `UNION ALL` of two indexed single-column lookups so each branch gets a clean access predicate.
- Confirm: plan shows two index-scan branches feeding a merge/bitmap-OR/bitmap-AND node (MySQL `index_merge`, Postgres `BitmapOr`/`BitmapAnd`, Oracle bitmap conversion) — compare cost against a single composite-index plan.

## NULL handling

- Trigger: `WHERE col IS NULL` / `IS NOT NULL` against an indexed column, especially in Oracle.
- What happens (Oracle-specific, worth flagging loudly): Oracle omits a row from a B-tree index entirely if *every* indexed column in the key is NULL for that row; the row stays if at least one indexed column is non-NULL. A single-column index therefore can't support `IS NULL` at all. A composite index where a second column is usually populated *can*, since the row wasn't dropped.
- Fix (Oracle): add a second, generally-populated column to the index, or emulate a partial index (below).
- Vendor divergence: this NULL-omission is Oracle-specific — Postgres, MySQL, and SQL Server generally store NULL entries and can index `IS NULL`/`IS NOT NULL`. Verify per engine rather than assuming Oracle's rule applies elsewhere.
- Confirm: check whether `IS NULL`/`IS NOT NULL` produces an index range scan or a full scan for your engine and index shape.

**Partial / filtered indexes.**
- Trigger: a small "interesting" subset of values queried far more than the rest, e.g. `WHERE status = 'pending'` where most rows are `'done'`.
- What happens unaddressed: a full-column index wastes space and write-maintenance cost on rows nobody queries by that predicate.
- Fix: Postgres — `CREATE INDEX ON orders (customer_id) WHERE status = 'pending'`. SQL Server — identical syntax as a "filtered index," but disallows functions and OR in the predicate. DB2 LUW approximates one via `EXCLUDE NULL KEYS`. Oracle has no direct syntax — emulate with a function-based index whose function returns NULL for excluded rows, leveraging the NULL-omission behavior above. MySQL has no documented equivalent.
- Confirm: the query's WHERE clause must provably imply the index's WHERE clause; check the plan for the partial index being chosen over a full or full-column-index scan.

## "Smart logic" catch-all WHERE clauses for optional filters

- Trigger: one static statement meant to serve many optional filters via bind-parameter escape hatches, e.g. `WHERE (:status IS NULL OR status = :status) AND (:region IS NULL OR region = :region)`.
- What happens: this is one of the worst common anti-patterns for index usage, and the root cause is plan caching, not the logic itself. Bind values are opaque at parse time, so the engine compiles one plan that must stay correct for every combination of NULL/non-NULL parameters — and the only plan safe for "every filter disabled" is a full table scan, even when every filtered column has its own index. None of the `IS NULL OR col = :p` branches become access predicates. Swapping in literal values for the same logic instead produces a normal index range scan at a fraction of the cost, proving plan caching (not the OR/IS NULL logic itself) is the culprit.
- Fix: build the WHERE clause dynamically in application/generated SQL so only the filters actually supplied appear in the statement text — while still binding the remaining values rather than inlining them as literals. This gets a tight, cacheable plan per filter combination without losing injection safety.
- Confirm: compare the catch-all plan (full scan, high cost) against the dynamic/per-combination plan (index range scan, low cost); look for `IS NULL OR` branches that never resolve into access predicates.
- Vendor mechanics differ substantially. Oracle: shared SQL-area cache; "bind peeking" reflects only the first execution's values (nondeterministic later behavior); "adaptive cursor sharing" (11g+) caches multiple plans per statement keyed to selectivity. SQL Server: "parameter sniffing" causes the same first-call nondeterminism; `RECOMPILE`/`OPTIMIZE FOR` are the escape hatches. Postgres: only long-lived prepared statements are affected — a generic plan only kicks in after several executions. MySQL has no plan cache at all, so this failure mode doesn't occur there. DB2 LUW: `REOPT(ALWAYS)` re-peeks every execution; `REOPT(ONCE)` peeks only the first call.

## Joins

- **Nested loop (N+1).** Trigger: small outer row set looked up row-by-row against an inner table. Cost is `outer_rows × inner_lookup_cost`; without an index on the inner table's join column, each lookup becomes a full scan of it. Fix: index the inner table's join column(s). Confirm: an index range scan (not a full scan) feeds the inner side of the loop.
- **Hash join.** Trigger: no useful join-column index, or one input notably smaller. The engine hashes the smaller side into memory and probes the other against it — indexing the join columns does *not* help, since the hash table replaces the lookup, and its cost scales with selected *columns*, not just rows. Fix: trim SELECT lists to shrink the hash table; indexes help only via independent filters that shrink an input before hashing. MySQL only gained hash joins in 8.0.18 — earlier versions fall back to nested loops even when hashing would be cheaper.
- **Sort-merge join.** Trigger: both inputs already sorted on the join key, often for free via an index. Each side sorts (or skips sorting if index order matches) then merges in one pass; if neither is pre-sorted, the double sort usually loses to a hash join. Confirm: explicit sort steps feeding a merge mean order didn't come from an index. MySQL has no sort-merge join operator at all.

## Covering indexes and index-only scans

- Trigger: every column referenced by SELECT, WHERE, and JOIN is present in one index, so the engine never visits the table.
- What happens: the query is answered entirely from index leaf pages, skipping the random-I/O row-fetch step — the single biggest win available for queries that would otherwise touch many scattered rows. The benefit shrinks if the table's row order already matches the index.
- Fix: widen the index's key columns to include SELECT-list columns, or (cheaper to maintain) add them as non-key included columns — Postgres/SQL Server `INCLUDE (...)`. Included columns live at the leaf level only, don't affect sort order, and don't count against key-size limits. Trim SELECT lists (avoid `SELECT *`).
- Confirm, per vendor: Postgres — `Index Only Scan` with `Heap Fetches` near zero (nonzero, or a plain `Index Scan`, means it isn't covered — often a stale visibility map; check/run `VACUUM`). Oracle — no `TABLE ACCESS BY INDEX ROWID` follows the index scan. MySQL — `Using index` in EXPLAIN's Extra column. SQL Server — Index Seek/Scan with no Key/RID Lookup.
- Watch for regressions: any WHERE condition on a column absent from the covering index silently kills the index-only path — recheck the plan after edits near one.

## Clustered / index-organized tables

- Trigger: choosing between a heap-organized table (rows anywhere, indexes hold physical pointers) and a clustered/index-organized table (row data stored inside the primary-key B-tree).
- What happens: primary-key lookups on a clustered table are effectively free index-only access. But every *secondary* index must store the clustering-key value instead of a physical pointer (rows move as the key is maintained), so a secondary-index lookup costs two tree traversals — secondary index → clustering key → row — instead of one scan plus a direct fetch, and secondary indexes grow larger.
- Fix: favor clustering for tables dominated by primary-key access; tables needing several secondary indexes are usually better off heap-organized with covering (INCLUDE) indexes.
- Vendor notes: SQL Server defaults to clustered (opt out via `NONCLUSTERED` on the PK). MySQL/InnoDB is always clustered on the PK with no opt-out; secondary indexes implicitly store the PK. Oracle defaults to heap; opt in via `ORGANIZATION INDEX`. Postgres has no true index-organized table — `CLUSTER` is a one-time reorg that drifts back out of order after subsequent writes.

## Sorting and grouping that can (or can't) use an index

- Trigger (works): `WHERE customer_id = ? ORDER BY created_at` against index `(customer_id, created_at)` — the equality predicate narrows the walk to one contiguous slice, and the next column is already ordered within it, so no sort runs.
- Trigger (breaks): same index, but a *range* predicate on the leading column, e.g. `WHERE created_at >= ? ORDER BY customer_id` — the range spans multiple leading-column values, the sort column's order resets within each, and an explicit sort reappears.
- Fix: order composite index columns as equality-filter columns, then ORDER BY/GROUP BY columns; never let a ranged column sit ahead of one needing global sort.
- Mixed directions: `ORDER BY a ASC, b DESC` can't be satisfied by walking a single-direction index in one pass (a uniformly reversed `DESC, DESC` can, by walking backward). Build the index with matching per-column direction: `(a ASC, b DESC)`. MySQL before 8.0 silently stores all index columns ascending, so this fix needs 8.0+.
- NULLS FIRST/LAST: only Postgres bakes this into the index definition; other engines can't avoid a sort when explicit NULL placement disagrees with their default.
- GROUP BY follows the identical mechanism — index order lets aggregation stream instead of materializing and sorting first. Confirm: a dedicated sort/aggregate operator in the plan (Oracle's no-sort `SORT ORDER BY NOSORT`/`SORT GROUP BY NOSORT` variants signal it was avoided) means index order wasn't usable when present.

## Pagination: OFFSET vs. keyset (seek) pagination

- Trigger: `SELECT ... ORDER BY created_at LIMIT 20 OFFSET 4000` (or `ROW_NUMBER() OVER (ORDER BY ...)` filtered by a row-number range) used to page through results.
- What happens: even when an index fully satisfies the ORDER BY and no sort runs, the engine must still walk past all `OFFSET` rows before returning the requested slice — the index avoids sorting but not the traversal of skipped rows. Cost per page is roughly `offset + limit`, so later pages get progressively slower and paging through a full result set costs quadratic total work. `ROW_NUMBER()`-based paging restates the same problem: Oracle, SQL Server, and Postgres 15+ detect the bound and stop early, but MySQL, MariaDB, and DB2 compute row numbers for the entire ordered set regardless of the requested page.
- Fix — keyset/seek pagination: carry forward the sort column's value and a unique tiebreaker (e.g. the primary key) from the last row of the previous page, and filter for rows strictly beyond that point instead of skipping a count: `WHERE (created_at, id) > (:last_created_at, :last_id) ORDER BY created_at, id LIMIT 20`. This makes the boundary a genuine access predicate — the engine seeks directly to the anchor, so each page costs roughly `O(limit)` regardless of depth. A unique tiebreaker is mandatory; without one, rows sharing a sort value have no stable order and pages can duplicate or skip rows.
- Trade-offs: can't jump to an arbitrary page number, only forward/backward from a known anchor — fine for infinite scroll, not "jump to page 47." More stable than OFFSET under concurrent writes, since it anchors to real values instead of recomputing from row zero.
- Vendor support for the row-value comparison above: Postgres and DB2 support it natively with index-optimized access; MySQL parses it but can't use it as an access predicate (filter-only); Oracle before 12c and SQL Server need it manually expanded: `created_at > ? OR (created_at = ? AND id > ?)`.
- Confirm: a seek/range-scan bounded by the anchor tuple, not a count-then-skip step; compare page-N latency at a deep offset before and after the rewrite.

## Write-side cost of indexes

- **INSERT.** Trigger: any INSERT into a table with N indexes. Every index needs its own B-tree insert — walk to the leaf, insert the key, split (with possible cascading parent updates) if full — typically the dominant cost of an insert; going from zero indexes to one can be an order of magnitude slower, with real but smaller marginal cost per additional index.
- **UPDATE.** Trigger: an UPDATE changing M columns, K indexed. Only indexes on the K changed columns are touched (a delete+insert pair per affected index). Fix: targeted `SET` clauses touching only changed columns, not blanket updates of every mapped column — some ORMs historically updated every column regardless of what changed, silently maintaining every index on every write.
- **DELETE.** Structurally like insert — remove the row and the corresponding entry from every index. Postgres-specific: a DELETE marks the row dead via MVCC instead of touching indexes immediately, deferring physical removal to `VACUUM` — throughput is decoupled from index count at delete time, but dead tuples accumulate until vacuumed.
- **General guidance.** No universal threshold exists for "too many indexes" — index only what's actually filtered, joined, or sorted on, and prefer one well-ordered composite index over several overlapping single-column ones.

## Testing at production scale

- Trigger: validating performance against a small development dataset, or trusting a plan labeled "index scan/seek" as automatically fast.
- What happens: an index that only partially bounds a query (access predicate on one column, filter predicate on another) looks identical to a fully-bounded index at small scale — both say "uses an index" — but they scale completely differently as data grows: one stays roughly flat, the other degrades linearly, and the gap only shows at realistic row counts. The same query can also slow substantially under concurrent load alone, since production exercises contention an isolated dev test never does.
- Fix: test with production-scale row counts and realistic concurrency, not correctness-sized sample data; don't stop at "an index is used" — confirm which conditions are access versus filter predicates, since that split predicts scaling.
- Confirm: re-run at 10x/100x current data volume and compare growth; a plan step whose row count doesn't shrink between the index step and the final output is the filter-predicate tell.
