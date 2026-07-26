---
name: sql-performance
description: Write SQL and design indexes that stay fast as data grows — composite index column ordering, keeping predicates sargable, avoiding function wrapping and implicit casts, covering indexes, keyset instead of OFFSET pagination, and reading an execution plan for access versus filter predicates. Use when writing or reviewing queries, adding an index, diagnosing a slow query, designing a schema, or building pagination.
---

# SQL Performance

## Purpose

"The query uses an index" is not the same as "the query is fast," and the gap between those two statements is where most database performance problems live.

An index lookup has three cost components, and only the first is cheap:

1. **Walking root to leaf** — grows with the *logarithm* of row count. Effectively flat no matter how large the table gets.
2. **Walking the leaf list** to collect every match — grows with the *number of matching entries*.
3. **Fetching each matching row** — one random I/O per match, unless the index covers the query.

Steps 2 and 3 are where an index quietly becomes a scan wearing a disguise. An index matching thousands of rows is not meaningfully better than reading the table.

## The one distinction everything reduces to

**An access predicate bounds which index entries get walked. A filter predicate is evaluated on rows already fetched.**

A filter predicate discards rows *after* paying to retrieve them. It reduces the result, not the work. Every rule in this skill is really the same question asked about a different syntax: **does this condition become access, or filter?**

A plan showing "index seek" or "index scan" tells you nothing on its own. You have to look at which conditions actually bounded the traversal.

Reading it per engine:

| Engine | How to tell |
|---|---|
| **Oracle** | Explicit `access()` versus `filter()` in the predicate section |
| **SQL Server** | Seek Predicates (access) versus Predicate (filter) |
| **DB2** | START/STOP keys versus SARGable predicates |
| **Postgres / MySQL** | Not labeled — infer from where the row count drops between plan steps |

The Postgres/MySQL tell: **a plan step whose row count doesn't shrink between the index operation and the output is a filter predicate.**

## Rules that apply without loading anything

**1. Equality columns before range columns.** In a composite index, a range predicate stops any later column from narrowing the search. Index `(created_at, customer_id)` with `WHERE created_at BETWEEN ? AND ? AND customer_id = ?` walks the whole date span and filters. `(customer_id, created_at)` makes both conditions access predicates. **This is the single highest-value fix in this skill.**

**2. Never wrap an indexed column in a function.** `WHERE UPPER(name) = ?` or `WHERE price * 1.1 > ?` makes the index unusable — it stores raw values, not transformed ones. Move the operation to the literal side, or build a matching expression index.

**3. Cast the literal, never the column.** Comparing a string column to a numeric literal makes the engine cast the *column*, which is a function wrap with a correctness bug attached. Postgres and DB2 reject the mismatch loudly; Oracle, MySQL, SQL Server, and SQLite cast silently.

**4. A leading wildcard defeats the index.** `LIKE 'Wina%'` uses the literal prefix as a range bound. `LIKE '%mann'` has no prefix and scans. Substring search needs full-text, not `LIKE`.

**5. Catch-all optional filters produce a full-scan plan.** `WHERE (:status IS NULL OR status = :status) AND ...` compiles one plan that must be correct when every filter is disabled — and the only safe such plan is a table scan. Build the WHERE clause dynamically, binding the values that are actually present.

**6. Use keyset pagination, not OFFSET.** `OFFSET 4000` still walks 4000 rows before returning anything, so cost grows with page depth. Anchor on the last row's sort value plus a unique tiebreaker instead.

**7. A composite index beats several single-column indexes.** A B-tree is one-dimensional — only one condition can bound the walk. Two single-column indexes require a merge of two traversals, usually costlier than one well-ordered composite.

**8. Test at production scale.** A partially-bounded index and a fully-bounded one look identical on development data and diverge completely as rows grow. Correctness-sized data proves nothing about scaling.

## Triage

| What you're doing | Reference |
|---|---|
| Adding or reordering an index; covering, partial, clustered choices; join indexing; write cost | [index-design](references/index-design.md) |
| Writing a WHERE clause; diagnosing why an existing index isn't used | [query-shapes](references/query-shapes.md) |
| ORDER BY, GROUP BY, or any pagination | [sorting-and-pagination](references/sorting-and-pagination.md) |

## Boundary with `distributed-data`

- **`sql-performance`** — making a query fast against one logical database.
- **`distributed-data`** — correctness across replicas and partitions, plus how partition key choice creates hot spots.

They meet at pagination: this skill covers why `OFFSET` is slow, `distributed-data` covers why it also skips and repeats rows under concurrent writes. Keyset pagination fixes both, which is why both recommend it.
