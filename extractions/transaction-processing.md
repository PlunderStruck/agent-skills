# Extraction: transaction-processing

**Source:** Principles of Transaction Processing — Bernstein & Newcomer, 2nd ed.

> **Partial source.** The available text was a scraped preview covering chapters 1-6 only, cut off mid-paragraph in chapter 6. Chapters 7 (recovery and logging), 8 (two-phase commit in depth) and 9 (replication protocols) are absent from the source, not merely unextracted. Treat this as the application-facing half of the book.

---

# What Bernstein & Newcomer adds beyond `distributed-data`

## Coverage note (read first)

The extracted text file stops partway through Chapter 6, Section 6.9 ("B-Tree Locking"), mid-paragraph — it is a scraped preview, not the full 378-page book. It contains Chapters 1–6 (Introduction; TP Abstractions; Application Architecture; Queued TP; Business Process Management; Locking), missing only the back half of Locking (multigranularity locking detail, nested-transaction locking) plus everything after it: Chapter 7 (Recovery and Logging — WAL, checkpointing, redo/undo, ARIES), Chapter 8 (Two-Phase Commit in depth — presumed abort/commit, in-doubt windows, heuristic decisions), Chapter 9 (Replication protocols), Chapter 10 (Middleware products/standards), Chapter 11 (Future trends). Those chapters are named only in the front-matter table of contents, and I have not fabricated content for them.

So: three of your seven priority areas (2PC mechanics in depth, recovery/logging discipline, transactional replication protocols) are **not answerable from this file** — the source doesn't contain them, not that they're absent from the book. What follows is the genuine new material actually present: queued/transactional messaging as the alternative to distributed transactions, lock-manager behavior, and — a bonus not on your list but squarely in scope — durable multi-step business processes (sagas), which is where 2PC-avoidance and compensation actually get worked out at the application level.

If the 2PC/recovery/replication chapters matter to you, they need a re-extraction of pages roughly 200–330 of the same book; I'd treat this pass as "half the book, the more application-facing half."

---

## PART 1 — New material

### 1. Composed transaction boundaries (nested calls, savepoints)

**Trigger:** A transactional method calls another method that also brackets a transaction — `@Transactional` calling `@Transactional` in Spring, nested `Start`/`Commit` in any framework, or a stored procedure calling another. Also: any code using `SAVEPOINT`.

**What goes wrong:** Frameworks resolve this silently and differently. "Requires new" starts a second, independent transaction — if it commits and the *outer* transaction later aborts, the inner work is now permanent and unrecoverable, breaking the atomicity the caller assumed. "Required" (the common default) folds the inner call into the caller's transaction — an inner rollback-only can abort far more work than the inner method's author intended, and an inner *commit* does nothing (only the outermost commit is real), which surprises developers who test the inner method standalone and see it "work." Nobody reads the propagation attribute; everyone assumes the intuitive-sounding default.

**What to do:** Treat the propagation attribute (`REQUIRED`, `REQUIRES_NEW`, `MANDATORY`, `NESTED`/savepoint, `NOT_SUPPORTED`) as part of the method's contract, not an implementation detail — document it next to the method, the same way you'd document a precondition. Use `REQUIRES_NEW` only when the inner unit's durability must genuinely survive the outer transaction's failure (e.g., an audit/security log entry — see the book's own example of logging a security violation independently of the caller's abort). Use `SAVEPOINT` for "undo part of what I did, not all of it" inside one transaction — most engines support it and almost nobody uses it, so it's usually cheaper than the workaround people build by hand.

**How to verify:** For every nested transactional call in a diff, state the propagation mode out loud and trace what happens to the inner work if the outer call throws after the inner call returns successfully. If you can't answer that in one sentence, the composition is undocumented, not just unverified.

---

### 2. Deadlocks are an expected outcome to retry, not a bug to eliminate

**Trigger:** Any transaction that reads a row and *might* later write it in the same transaction ("check the row, decide, then maybe update it"), and any code path that can hold two locks acquired in different orders across concurrent transactions.

**What goes wrong:** Read-then-maybe-write is the classic **lock-conversion deadlock**: both transactions take a shared/read lock first, then both ask to upgrade to a write lock — neither can, because the other's read lock is in the way, and neither can back off without breaking the transaction. This isn't a rare edge case; it's the *guaranteed* outcome whenever two concurrent transactions do this to the same row. Separately, engines detect deadlocks one of two ways — a simple timeout (aborts whoever's been blocked too long, occasionally punishing an innocent transaction) or an explicit wait-for graph (only aborts real cycles, but costs bookkeeping). Teams that don't know which their database uses either over-trust "no deadlock detected" or panic at false positives. And naive retry-the-same-way logic can produce **cyclic restart**: if you always abort/retry the older transaction, two competing transactions can trade the "oldest" title back and forth forever without either completing.

**What to do:** If you know you'll conditionally write, take the stronger lock upfront — `SELECT ... FOR UPDATE` (Postgres/MySQL/Oracle), or the DB's dedicated "update lock" mode if it has one (SQL Server's `UPDLOCK`, which conflicts with other writers/updaters but not with plain readers, so it converts to a write lock without the risk of conversion deadlock). Treat "you were chosen as the deadlock victim" as a retryable condition, distinct from other errors — catch the specific error code, not a generic exception handler, back off with jitter, and cap the retry count. If retries keep hitting the same pair of transactions, that's a signal to fix lock *ordering* (always acquire locks on shared resources in the same order across all code paths), not to raise the retry limit.

**How to verify:** Grep for `SELECT` statements inside a transaction whose result feeds a conditional `UPDATE`/`INSERT` later in the same transaction — confirm they lock defensively. Confirm the retry path specifically matches the deadlock/serialization-failure error code and is bounded.

---

### 3. Techniques for a hot spot that finer partitioning can't fix

**Trigger:** A single row or counter that nearly every transaction must touch — a running balance, an end-of-table sequence number, "next order number," a global inventory count — where the row is already the finest possible granularity, so re-sharding or re-keying the data model (the usual "hot partition" fix) doesn't apply. This is a different failure mode than a hot *shard key* in a distributed store: it's row-level lock contention inside one database, and throughput is capped at roughly one transaction per average lock-hold-time on that row, regardless of how many cores or connections you add.

**What goes wrong:** Teams hit this ceiling, assume it's a scaling/sharding problem, and reach for partitioning tools that don't apply because there's nothing left to partition — it's already one row.

**What to do, in roughly this order of preference:** (a) Do the update as a single atomic statement (`UPDATE t SET n = n + 1`) so the lock is held only for the statement, not the surrounding transaction. (b) If the transaction must do other work too, touch the hot row *last*, immediately before commit, so the lock's hold time is minimized — this sometimes means restructuring the method to defer the write. (c) For a hot value that's *checked* but whose exact value doesn't need to be locked immediately (e.g., "is there still inventory?"), use an optimistic pattern: read and act without locking, then re-check the same condition at commit time and abort if it changed — this trades a rare late abort for avoiding the lock during the transaction's whole lifetime. (d) If many transactions append to the same hot counter/log and the exact moment of the update doesn't matter, batch updates in memory per-worker and flush periodically instead of touching the shared value every request. (e) If a row has both frequently-written and rarely-written columns, split it into two physical rows/tables so a reader of the cold columns isn't blocked behind writers of the hot ones.

**How to verify:** For any row identified via lock-wait telemetry as a throughput ceiling, confirm which of these five is in use, or that a deliberate decision was made to accept the ceiling.

---

### 4. Build a job queue on a plain table with `SKIP LOCKED`, not with polling

**Trigger:** Multiple worker processes each poll a table for the next unclaimed row (`WHERE status = 'pending' ORDER BY id LIMIT 1`), a very common way to build a lightweight queue without standing up a broker.

**What goes wrong:** This predates the book's own framing — it discusses `READPAST` only as an isolation-level nicety for read committed queries. The concrete modern failure is: naive claim logic either serializes every worker behind whoever holds the lock on "the next row" (defeating the whole point of having multiple workers), or does a plain `SELECT` then a separate `UPDATE`, which races two workers onto the same row.

**What to do:** Use an atomic locking read that skips rows already claimed by someone else — `SELECT ... FOR UPDATE SKIP LOCKED` (Postgres, MySQL 8+) or `READPAST` (SQL Server) — inside a single statement that both selects and marks the row claimed. This is the modern, load-bearing version of the pattern; it needs no separate lock table and no polling loop that backs off and retries on contention.

**How to verify:** Confirm the claim query is one locking statement with skip-locked semantics, not a read followed by a conditional write.

---

### 5. Don't hold a lock across user think-time — split the transaction, add a time-boxed hold

**Trigger:** An interactive flow that shows a user something from the database (available seats, remaining stock) and only writes after the user decides — with real elapsed time, sometimes minutes, between the read and the write.

**What goes wrong:** Doing this as one transaction holds a lock on the shared row for as long as the user takes to decide, and every other request touching that row queues up behind it. Doing it as two fully independent transactions with no coordination lets the value go stale — someone else can take the last item between your read and your write, so you overbook.

**What to do:** Split into two transactions (query, then separately write), and if the item must not be given away while the user is deciding, add an explicit tentative-hold row or flag with its own expiry, reclaimed by a cleanup job if the user never confirms — not a database lock held across the gap. This is exactly the pattern behind "your seat is held for 10 minutes" on ticketing and booking sites.

**How to verify:** Check that no transaction spans a network round-trip to a client or a wait on user input. Check that any "hold" concept has an expiry and a reclaim path, not just a creation path.

---

### 6. Poison messages need a dead-letter threshold, not infinite retry

**Trigger:** A transactional queue consumer where processing a message can throw due to bad data in the message itself (not a transient fault) — the transaction aborts, the message goes back on the queue, and it will be redelivered forever unless something intervenes.

**What goes wrong:** Without a bound, a single malformed message can be retried indefinitely, burning consumer capacity and hiding the fact that it's permanently stuck. It's also invisible: nothing marks the message as "this will never succeed," so nobody investigates until throughput visibly degrades.

**What to do:** Configure a maximum delivery/retry count after which the message is moved to a dead-letter or exception queue rather than redelivered, with a minimum interval between retries so a transient blip doesn't get treated as poison prematurely. Require the parked message to be handled explicitly (manual reconciliation, alerting, or an automated corrective job) rather than silently dropped.

**How to verify:** Confirm the consumer/broker config has an explicit max-receive-count or equivalent, and that there's a defined destination and owner for what lands there.

---

### 7. Reliable request/reply over a queue needs an explicit recovery state machine

**Trigger:** A client submits a request onto a queue and later reads a reply from another queue — used as a deliberate alternative to a synchronous call or a distributed transaction across a client/server boundary (or across two services that can't share one transaction manager).

**What goes wrong:** After a crash or reconnect, naïve clients guess at what happened — "no reply yet, so let me resubmit" — and either duplicate the request or spin forever waiting for a reply that already arrived and was silently missed. The set of possible states is small and enumerable, but almost nobody enumerates it.

**What to do:** There are exactly four states after a request is built: (a) never submitted, (b) submitted, no reply yet, (c) reply arrived, not yet processed, (d) reply already processed. Make this decidable by durably recording, alongside the request itself, the ID of the last request actually enqueued and the ID of the last reply actually consumed (in a DB row, a dedicated queue, or as broker-managed session state). On reconnect, compare the request's ID against those two markers to land in exactly one of the four states — never guess from a timeout alone.

**How to verify:** Kill the client process at each of the four points in a test and confirm recovery reaches the correct state without duplicating the request or losing the reply.

---

### 8. Sagas: durable compensation, and the isolation gap they open

**Trigger:** A user-facing operation that must span more than one transaction because of cross-service calls, long delays (human approval, external processing), or contention concerns — the shape underlying Temporal workflows, AWS Step Functions, and any hand-rolled orchestrator.

**What goes wrong (two distinct failures):**
- **Missing compensation.** Teams build the forward chain of steps and skip the reverse path. A mid-sequence failure leaves partial, uncompensated state (money debited from one account, never credited to the other) with nothing tracking which steps actually committed, so nothing can automatically clean it up.
- **Invisible in-flight state.** Even with compensation logic ready, the isolation guarantee is gone the moment you split one operation into several transactions. Anything that reads the data between steps — an audit, a report, a second user — can observe a state that could never occur in any serial execution of the whole business operation (money debited from A, not yet credited to B, looks like money vanishing). Holding locks across the whole saga to prevent this reintroduces exactly the contention and availability cost that made you split the transaction in the first place.

**What to do:** For every forward step, write its compensating step, and persist the saga's progress — which steps committed, and enough data to invoke each one's compensation — as part of each step's own transaction (a DB row or a queue element), not as an afterthought once the step is "done." Run a watchdog that finds sagas stuck past a timeout and executes compensations for whatever committed. For the isolation gap: don't attempt to hide it with more locking — design the read paths that can observe in-flight state to understand and represent it explicitly (a visible "pending transfer" status, or a suspense/in-transit account that a reconciliation report adds back), rather than presenting an in-flight view as if it were final.

**How to verify:** For every saga/orchestration, ask "if this stops after step *k*, what runs to clean up steps 1..*k*, and where is the durable record of which steps ran?" — no answer means the saga is missing its compensation half. Separately, ask what any concurrent reader sees during the gap between steps, and whether that reader's code accounts for it.

---

## PART 2 — Placement

| # | Cluster | Placement | Why |
|---|---|---|---|
| 1 | Composed transaction boundaries / savepoints | **Extend** `distributed-data` — new reference (e.g. `transaction-composition.md`), or a section in `isolation-and-races.md` | It's squarely inside the skill's stated scope ("SQL/ORM queries and transactions") and is a genuine gap: the existing files cover races *between* transactions, not the semantics of *composing* transactions within one call stack. Self-contained enough to be its own short reference; thin enough that it could also live as a section. Not a new skill — it's one failure mode, not a discipline. |
| 2 | Deadlocks as expected/retryable; lock-conversion cause | **Extend** `isolation-and-races.md` | That file already discusses 2PL and says "deadlocks are frequent and the application must retry" in one sentence — this expands exactly that sentence with the mechanism (why conversion causes them), the detection mechanism (timeout vs. graph — matters for how you interpret "no deadlock detected"), and the retry contract (catch the specific error, cap it, watch for cyclic restart). Tightly coupled to content already there. |
| 3 | Single-row hot-spot techniques (not partitioning) | **Extend** `partitioning-and-hotspots.md` | That file's "Hot keys that hashing cannot fix" section is the distributed-store version of this exact problem (celebrity key, viral post). This adds the single-database version — delay-to-commit, optimistic verify-at-commit, batching, hot/cold column split — as a sibling section, explicitly flagged as "this is the same shape, one database, no sharding available." |
| 4 | `SKIP LOCKED` job queue on a table | **Extend** `distributed-data` — a short addition to `isolation-and-races.md` (it's fundamentally a locking-read pattern, not a broker pattern) | Modern, extremely common (Postgres/MySQL-backed job queues), and not covered anywhere in the existing skill. Small enough to be a subsection rather than its own file. Explicitly call out that the book's own framing (`READPAST` as an isolation nicety) predates this being the standard way people build queues without a broker. |
| 5 | Reservation-hold / don't-lock-across-think-time | **Extend** `isolation-and-races.md` (or fold into the same "reducing lock hold time" section as #2/#3) | Same underlying concern as the hot-spot cluster — lock duration management — but the trigger is user interaction rather than throughput, so worth its own paragraph even if it lives in the same file. |
| 6 | Poison messages / dead-letter threshold | **Extend** `streams-and-queues.md` | Direct, clean gap — that file covers delivery semantics, ordering, and windowing in detail but says nothing about what happens when a message can never succeed. One clean subsection closes it. |
| 7 | Reliable request/reply over a queue (4-state recovery) | **Extend** `idempotency-and-retries.md` | That file already has a "multi-step operations without distributed transactions" pattern (request ID → log → downstream dedup), but it's framed around fan-out to multiple partitions. This is the sibling pattern for a client-server request/reply round trip specifically, and the "enumerate the four states, don't guess from a timeout" framing is a good concrete worked example to sit next to the existing rules. |
| 8 | Sagas (compensation + isolation gap) | **Extend** `distributed-data` — new reference (e.g. `sagas-and-compensation.md`) | Substantial enough (two distinct failure modes, each with real depth) to earn its own file rather than a subsection, and it's the direct sibling of `keeping-stores-in-sync.md`'s "why not 2PC" argument: that file tells you to avoid distributed transactions across stores; this tells you what replaces the atomicity you gave up when you follow that advice across *services*. Not a standalone skill — same audience, same "what breaks / what fixes it" register as the rest of `distributed-data`, and a saga-only skill would be thin. |

### Explicitly dropped (too infrastructure-specific, or superseded by a modern equivalent already well understood)

- **TPC-A/B/C/E benchmark mechanics.** Useful only if you're benchmarking a database vendor's product; irrelevant to writing application code.
- **RPC internals** (proxy/stub generation, marshaling, interface compilers, little/big-endian translation, binding/registry mechanics). This is what gRPC/Thrift/REST frameworks do for you now; nobody hand-rolls this today, and the concepts that do matter (idempotence of retries, at-most-once vs. exactly-once) are already covered thoroughly in `idempotency-and-retries.md`.
- **Thick/thin client and forms/menus front-end architecture.** Entirely superseded by the modern web/mobile client stack; nothing here changes a decision anyone makes today.
- **Sessions and cookies as shared-state mechanisms.** Standard, well-understood web development knowledge; nothing in the book's treatment is sharper than what's already common practice.
- **Broker-based vs. bus-based integration (EAI vs. ESB).** Directly predates, and is now expressed as, API gateways, service meshes, and managed integration platforms (MuleSoft, Kafka Connect, etc.) — the architectural tradeoff (central intermediary vs. direct peer calls with shared protocol) still exists but nothing in the book's treatment adds to how that tradeoff is reasoned about today.
- **WS-BPEL / BPMN / SQL Server Service Broker specifics.** Product/standard trivia; the durable ideas (persist saga state, compensate, don't assume one big transaction) are extracted into cluster 8 above — the XML syntax and vendor API details are not something an application developer needs.
- **TP monitor / application server product taxonomy** (CICS, Tuxedo, ACMS, Java EE vs. .NET Enterprise Services). Historical vendor landscape; today's equivalent is "a web framework plus a managed database," and nothing here changes how you'd build one.
- **Database internal layering** (page-oriented files, access methods, query executor/optimizer as internal architecture). Useful only if you're building a database engine, which is explicitly out of scope per your brief.
- **Multiversion/MVCC implementation mechanics** (version chains, commit lists). The *consequences* for an application developer — what anomalies snapshot isolation still allows, that Postgres/MySQL call it "repeatable read," the vendor-naming traps — are already thoroughly covered in `isolation-and-races.md`. The internal chain-and-commit-list bookkeeping doesn't change any decision you'd make as a caller.
- **Data warehousing as a query-isolation escape hatch.** The underlying idea (don't run analytical queries against the OLTP database; use a derived, eventually-consistent copy) is already exactly what `keeping-stores-in-sync.md` teaches via CDC/outbox; the book's ETL-batch framing is the pre-streaming version of the same advice.
