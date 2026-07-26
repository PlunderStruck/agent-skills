# Query Shapes That Defeat an Index

Every rule here is the same failure: the condition can't bound the index walk, so it degrades to a filter applied after rows are already fetched.

## Functions and expressions on the column

The engine treats a function as opaque. The index stores raw values, not transformed ones, so there's nothing to search against.

```sql
WHERE UPPER(last_name) = 'SMITH'     -- index on last_name unusable
WHERE price * 1.1 > ?                -- index on price unusable
```

Two fixes:

- **Move the operation to the literal side** — `WHERE price > ? / 1.1`. Always prefer this; it needs no new index.
- **Build a matching expression index** — Postgres and Oracle index expressions directly (`CREATE INDEX ON t (UPPER(col))`). MySQL 5.7+ and SQL Server need a generated or computed column with an index on it, and queries must reference it in the matching form. DB2 LUW disallows user-defined functions in indexes entirely.

**On user-defined functions in indexes:** Oracle's `DETERMINISTIC` and Postgres's `IMMUTABLE` declarations are *trusted, not verified*. Declaring a function that actually reads the clock or session state doesn't error — it silently freezes stale values into the index, producing wrong results rather than slow ones. No execution plan reveals this. It's a code-review item.

**On over-indexing expressions:** separate indexes on `UPPER(col)` and `LOWER(col)` do the same job twice and tax every write. Standardize on one case-folding convention codebase-wide, and watch for ORMs injecting a case-folding call that doesn't match your index.

## Implicit type conversion

Comparing a string column to a numeric literal forces a cast. If the engine casts the *column*, that's a function wrap — with a correctness bug attached, since numeric casting conflates `'42'` and `'042'`.

```sql
WHERE numeric_string_col = 42      -- casts the column: scan + wrong results
WHERE numeric_string_col = '42'    -- casts nothing: index usable
```

Comparing a numeric column to a string literal is safe — only the literal converts.

**Postgres and DB2 reject cross-type comparisons with an error.** Oracle, MySQL/MariaDB, SQL Server, and SQLite cast silently, so this is invisible until you look for a `CAST` or `CONVERT` wrapping the column in the plan.

## Date truncation and derived values

Same failure, two very common shapes.

```sql
WHERE TRUNC(created_at) = DATE '2026-07-25'          -- wraps the column
WHERE first_name || ' ' || last_name = ?             -- derives from columns
```

**For dates, rewrite as a half-open range on the raw column:**

```sql
WHERE created_at >= DATE '2026-07-25'
  AND created_at <  DATE '2026-07-26'
```

Half-open, not `BETWEEN` — an inclusive upper bound mishandles rows with a time component on the final day.

**For concatenation**, store the derived value as a real or generated indexed column. Alternatively, keep the concatenation but add a redundant sargable predicate on the leading raw column — which only works when the concatenation preserves that column's sort order, as ISO-8601 formatting does.

## LIKE and wildcards

Only the literal characters *before the first wildcard* can bound the search.

```sql
WHERE last_name LIKE 'Wina%'    -- prefix acts as a range bound
WHERE last_name LIKE '%mann'    -- no prefix, full scan
```

For substring or suffix matching, use full-text search rather than fighting `LIKE`: Postgres `pg_trgm` or `@@`, MySQL `MATCH() AGAINST()`, Oracle Text, SQL Server `CONTAINS()`.

**A Postgres-specific trap worth knowing:** most engines optimistically assume a bind-parameter `LIKE` pattern has no leading wildcard and attempt a prefix scan. Postgres assumes the opposite — it skips the index for a parameterized `LIKE` even when the pattern is anchored. If index usage matters there, use a literal.

## OR across columns

A B-tree is one-dimensional, so only one condition can bound the walk. An `OR` across two separately indexed columns forces the engine to either scan, or to run two index traversals and combine the row sets in memory — a bitmap or index merge, generally costlier than one composite index.

**Fixes, in order:**

1. A single composite index with the more selective condition leading, letting the other be a cheap filter.
2. Where both branches are genuinely selective and an `OR` is unavoidable, rewrite as `UNION ALL` of two indexed lookups so each branch gets a clean access predicate.

**Confirm:** look for a merge node — MySQL `index_merge`, Postgres `BitmapOr`/`BitmapAnd`, Oracle bitmap conversion — and compare its cost against a composite-index plan.

## NULL handling

**Oracle behaves differently from everyone else here, and it surprises people.** Oracle omits a row from a B-tree index entirely when *every* indexed column is NULL for that row. A single-column index therefore cannot serve `IS NULL` at all. A composite index where a second column is usually populated *can*, because the row was never dropped.

Fix on Oracle: add a generally-populated column to the index, or emulate a partial index using the same NULL-omission behavior deliberately.

Postgres, MySQL, and SQL Server generally store NULL entries and can index `IS NULL` and `IS NOT NULL` normally. Verify per engine rather than carrying Oracle's rule elsewhere.

## Catch-all optional filters

The worst common anti-pattern in this document, and the cause is plan caching rather than the logic itself.

```sql
WHERE (:status IS NULL OR status = :status)
  AND (:region IS NULL OR region = :region)
```

Bind values are opaque at parse time, so the engine must compile one plan that stays correct for *every* combination of supplied and omitted filters — including all-omitted. The only plan safe for that case is a full table scan. None of the branches become access predicates, no matter how well-indexed the columns are.

The proof: substituting literals for the same logic produces a normal index range scan at a fraction of the cost.

**Fix:** build the WHERE clause dynamically so only the filters actually supplied appear in the statement text — while still *binding* their values rather than inlining them. You get a tight, cacheable plan per filter combination without giving up injection safety.

## Bind parameters and skew

Binds let a plan be reused instead of re-optimized per call. The tradeoff: the optimizer sees no concrete value at plan time and assumes uniform distribution. On a badly skewed column — where one value matches ten rows and another matches a million — a single cached plan will be wrong for one of them.

Default to binds. For a known-skewed column, use a literal, or the engine's per-value plan feature.

**Engine mechanics differ substantially:**

- **Oracle** — bind peeking reflects only the first execution's values; adaptive cursor sharing (11g+) caches multiple plans keyed to selectivity.
- **SQL Server** — parameter sniffing causes the same first-call dependence; `RECOMPILE` and `OPTIMIZE FOR` are the escape hatches.
- **Postgres** — affects only long-lived prepared statements, and a generic plan is chosen only after several executions.
- **MySQL** — no plan cache, so this failure mode doesn't occur.
- **DB2 LUW** — `REOPT(ALWAYS)` re-peeks every execution; `REOPT(ONCE)` peeks only the first.

**Confirm:** compare plans for different literal values of the skewed column. Divergent strategies mean one cached bind plan is wrong for some inputs.
