# Isolation Levels and Write Races

Applies to any transaction, any `SELECT` that informs a subsequent write, and any counter or balance update.

## What each isolation level still permits

The level names are unreliable across vendors; the anomalies are not. Reason about the anomaly.

| Anomaly | Read committed | Snapshot isolation | Serializable |
|---|---|---|---|
| Dirty read (see uncommitted data) | prevented | prevented | prevented |
| Dirty write (overwrite uncommitted) | prevented | prevented | prevented |
| Read skew (query sees mixed points in time) | **allowed** | prevented | prevented |
| Lost update (concurrent read-modify-write) | **allowed** | vendor-dependent | prevented |
| Write skew (decide on a premise that then changes) | **allowed** | **allowed** | prevented |
| Phantom in a read-write transaction | **allowed** | **allowed** | prevented |

## Naming traps

Check these before assuming a level does what its name says:

- Postgres and MySQL call snapshot isolation **"repeatable read."**
- Oracle calls snapshot isolation **"serializable."** Oracle 11g does not implement real serializability.
- DB2 uses **"repeatable read"** to mean serializability.
- Default is **read committed** in Oracle, Postgres, SQL Server, and MemSQL.

The SQL standard's definitions predate snapshot isolation and are ambiguous enough that implementations diverge while all claiming compliance. "We use repeatable read" tells you almost nothing without naming the vendor.

## Lost update

```
value = SELECT counter FROM t WHERE id = 1;   -- reads 42
value = value + 1;
UPDATE t SET counter = 43 WHERE id = 1;       -- clobbers a concurrent +1
```

Two increments, one survives. This is the single most common concurrency bug in application code, and ORMs make it the path of least resistance — load object, mutate field, save.

The same shape, less obviously:

- Adding an element to a JSON array by parsing, appending, and writing the whole document back
- Two users editing a wiki page where save submits the full body
- Recomputing a denormalized total from its parts and storing it

Fixes, best first:

1. **Atomic write operation.** `UPDATE t SET counter = counter + 1 WHERE id = 1`. Concurrency-safe in every relational DB. Redis and MongoDB have equivalents. Use this whenever the change can be expressed as an operation rather than a value.
2. **Database-detected lost update.** Postgres repeatable read, Oracle serializable, and SQL Server snapshot isolation abort the offending transaction automatically. **MySQL/InnoDB repeatable read does not detect lost updates.** This is the most valuable option when available because it doesn't depend on the developer remembering.
3. **Explicit lock.** `SELECT ... FOR UPDATE` before the read-modify-write. Correct, but you have to remember it at every site, and forgetting is silent.
4. **Compare-and-set.** `UPDATE ... WHERE id = 1 AND content = 'old'`, then check the affected row count and retry. Caveat: if the DB permits the `WHERE` clause to read from an old snapshot, this does not prevent the lost update. Verify your engine's behavior before relying on it.

Under multi-leader or leaderless replication, locks and CAS do not apply — there is no single up-to-date copy. Use commutative operations (counters, set-adds) or merge siblings explicitly.

## Write skew

Two transactions read the same rows, then write to *different* rows, invalidating each other's premise.

```
-- Both transactions run this concurrently, both see count = 2, both proceed.
SELECT count(*) FROM doctors WHERE on_call = true AND shift_id = 1234;
-- if count >= 2:
UPDATE doctors SET on_call = false WHERE name = 'Alice' AND shift_id = 1234;
```

Result: zero doctors on call, invariant violated, no error.

Snapshot isolation does not prevent this and does not detect it. Lost-update detection does not catch it either, because the transactions touched different rows.

Recognize the shape:

1. A query checks whether some condition holds
2. Application code branches on the result
3. A write changes whether that condition still holds

Real instances: booking a room after checking for overlaps; claiming a username after checking availability; spending against a balance after summing it; moving a game piece to a square after checking it's empty; enforcing "at most N active X per account."

Fixes:

- **A unique constraint**, when the invariant is single-column uniqueness. Cheap, and it holds at weak isolation. Strongly prefer this.
- **Serializable isolation**, when the invariant spans rows. This is the only general fix.
- **Materializing conflicts** — pre-creating rows representing the lockable resource (every room × every 15-minute slot) so `SELECT ... FOR UPDATE` has something to lock. Ugly, leaks concurrency control into the data model, error-prone. Last resort.

Note that `SELECT ... FOR UPDATE` **cannot** fix the cases that check for the *absence* of rows. There is nothing to lock. This is the phantom problem, and it's why the booking and username cases are harder than the on-call case.

## Deadlocks are a retryable outcome, not a bug to eliminate

**Lock-conversion deadlock** is the guaranteed result whenever two concurrent transactions read a row and then upgrade to a write on it. Both take a shared lock, both ask to upgrade, neither can — because the other's read lock is in the way. This is not an edge case; it is what *always* happens when two transactions do read-then-maybe-write on the same row.

**Take the stronger lock up front** if you know you might write: `SELECT ... FOR UPDATE`, or a dedicated update-lock mode where the engine has one (SQL Server's `UPDLOCK` conflicts with writers but not plain readers, so it converts without the conversion risk).

**Treat "you were the deadlock victim" as its own retryable condition.** Catch the specific error code, not a generic exception. Back off with jitter and cap the attempts. If retries keep hitting the same pair, the fix is lock *ordering* — always acquire shared resources in the same order across every code path — not a higher retry limit.

Know which detection your engine uses: a timeout, which occasionally aborts an innocent transaction, or a wait-for graph, which only aborts real cycles at the cost of bookkeeping. Teams that don't know either over-trust "no deadlock detected" or panic at false positives.

## Composed transaction boundaries

A transactional method calling another transactional method resolves silently, and differently per framework.

**Requires-new** starts an independent transaction. If it commits and the *outer* one later aborts, the inner work is permanent — breaking the atomicity the caller assumed. **Required**, the usual default, folds the inner call into the caller's: an inner rollback-only can abort far more than its author intended, and an inner commit does nothing, which surprises anyone who tested that method standalone and watched it "work."

**Treat the propagation mode as part of the method's contract** and document it like a precondition. Reserve requires-new for work whose durability must genuinely survive the caller's failure — an audit or security-log entry. Use a savepoint for "undo part of what I did": widely supported, almost never used, and usually cheaper than the manual workaround people build instead.

**Check.** For every nested transactional call, state what happens to the inner work if the outer call throws after it returned. If that takes more than one sentence, the composition is undocumented.

## Choosing serializable

Three implementations, different failure characteristics:

- **Actual serial execution** (VoltDB, Redis, Datomic). Requires the working set in memory, transactions submitted as stored procedures rather than interactive statements, and write throughput that fits one core. Cross-partition transactions collapse throughput by orders of magnitude.
- **Two-phase locking** (MySQL InnoDB serializable, SQL Server serializable). Readers block writers and vice versa. Unstable tail latency; one slow transaction stalls everything behind it. Deadlocks are frequent and the application must retry.
- **Serializable snapshot isolation** (Postgres 9.1+, FoundationDB). Optimistic: no blocking, aborts at commit time when a serialization conflict is detected. Degrades under high contention because aborts multiply. Requires read-write transactions to be short. Usually the right default when available.

Under SSI and 2PL alike, **the application must handle aborts by retrying.** Rails ActiveRecord and Django do not retry aborted transactions by default — the exception propagates and the user's input is discarded. If you select an isolation level that aborts, wire up the retry, and make sure the retried work is idempotent.

## What to check in the code

- Every `SELECT` whose result feeds an `if` that guards a write — classify it as lost update or write skew and pick the matching fix.
- Every counter, balance, quota, or aggregate maintained by application code.
- Whether the isolation level is actually what the code assumes. Check the connection/pool configuration, not the docs.
- Whether aborts are retried, and whether the retry is safe to run twice.
- Whether an invariant enforced in application code could be moved into a constraint.
