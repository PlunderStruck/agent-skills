# Sorting, Grouping, and Pagination

## When an index can satisfy ORDER BY

An index already stores its columns in order. If the query's required order matches a contiguous stretch of that index, no sort operation runs at all — the engine just walks.

**Works:**

```sql
-- index (customer_id, created_at)
WHERE customer_id = ? ORDER BY created_at
```

The equality predicate narrows the walk to one contiguous slice, and within that slice `created_at` is already ordered.

**Breaks:**

```sql
-- same index (customer_id, created_at)
WHERE created_at >= ? ORDER BY customer_id
```

Here the leading column is ranged, so the walk spans many `customer_id` values, and ordering resets within each. An explicit sort reappears.

**The ordering rule:** equality-filter columns first, then the ORDER BY or GROUP BY columns. Never let a ranged column sit ahead of one that needs global ordering.

## Mixed sort directions

`ORDER BY a ASC, b DESC` cannot be satisfied by walking a single-direction index in one pass. A uniformly reversed `ORDER BY a DESC, b DESC` *can* — the engine walks the leaf list backward.

Fix: build the index with matching per-column direction, `(a ASC, b DESC)`. **MySQL before 8.0 silently stores all index columns ascending** and accepts the `DESC` keyword without honoring it, so this fix requires 8.0+.

## NULL placement

Only Postgres bakes `NULLS FIRST` / `NULLS LAST` into the index definition. On other engines, an explicit NULL placement that disagrees with the engine's default forces a sort no index can avoid.

## GROUP BY

Follows exactly the same mechanism. When index order matches the grouping columns, aggregation streams as it walks instead of materializing everything and sorting first.

**Confirm:** a dedicated sort or aggregate operator in the plan means index order wasn't usable. Oracle is explicit about the good case — `SORT ORDER BY NOSORT` and `SORT GROUP BY NOSORT` signal that the sort was avoided.

## Why OFFSET pagination degrades

```sql
SELECT ... ORDER BY created_at LIMIT 20 OFFSET 4000
```

Even when an index fully satisfies the ordering and no sort runs, **the engine must still walk past all 4000 skipped rows** before returning anything. The index avoided the sort; it did not avoid the traversal.

Cost per page is roughly `offset + limit`, so later pages get progressively slower, and paging through a full result set costs quadratic total work.

`ROW_NUMBER() OVER (ORDER BY ...)` filtered by a row-number range has the same problem. Oracle, SQL Server, and Postgres 15+ detect the bound and stop early; MySQL, MariaDB, and DB2 compute row numbers across the entire ordered set regardless of which page you asked for.

## Keyset pagination

Carry forward the sort value and a unique tiebreaker from the last row of the previous page, and ask for rows strictly beyond that point:

```sql
WHERE (created_at, id) > (:last_created_at, :last_id)
ORDER BY created_at, id
LIMIT 20
```

The boundary is now a genuine access predicate. The engine seeks directly to the anchor, so **each page costs roughly the page size regardless of depth.**

**The unique tiebreaker is mandatory.** Without it, rows sharing a sort value have no stable order, and pages will duplicate or skip rows — a correctness bug, not just a performance one.

**Tradeoffs.** You can only move forward or backward from a known anchor, not jump to page 47. That suits infinite scroll and API cursors; it doesn't suit numbered page links. In exchange it's far more stable under concurrent writes, because it anchors to real values rather than recomputing a position from row zero.

**Row-value comparison support varies:**

- **Postgres, DB2** — support the tuple syntax above natively, with index-optimized access.
- **MySQL** — parses it but cannot use it as an access predicate; it degrades to a filter.
- **Oracle before 12c, SQL Server** — need it expanded manually:

```sql
WHERE created_at > :last_created_at
   OR (created_at = :last_created_at AND id > :last_id)
```

**Confirm:** the plan shows a seek or range scan bounded by the anchor, not a count-then-skip step. Compare latency at a deep page before and after the rewrite — that's where the difference appears, and it's the reason this problem ships so often. Page one is always fast.
