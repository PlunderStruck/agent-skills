# Idempotency and Retries

Applies to retry logic, queue consumers, webhook handlers, background jobs, payment and email side effects, and any request that can be resubmitted by a user.

## The core fact

**A timeout is not a failure. It is an unknown.** If a request times out, three things are possible: it never arrived, it arrived and failed, or it arrived and succeeded and the response was lost. The client cannot distinguish them.

Code that catches a timeout and retries is choosing "duplicate the effect" over "lose the effect." That is often the right choice, but only if the operation is idempotent. If it isn't, you have chosen "charge the customer twice."

## When retrying is wrong

- **Permanent errors.** Constraint violations, validation failures, auth errors. Retrying accomplishes nothing and burns capacity.
- **Overload.** Retrying into an overloaded system deepens the overload. Distinguish overload errors from transient ones, use exponential backoff, and bound the retry count.
- **Side effects outside the database.** A transaction abort rolls back the DB writes but does not un-send the email, un-charge the card, or un-call the third-party API. Move side effects after commit, or make them individually idempotent, or record intent in the DB and have a separate worker perform them.

Retrying *is* right for: deadlock aborts, serialization conflicts, transient network faults, and failover.

## Making an operation idempotent

Some operations already are: setting a key to a fixed value, marking a flag true, deleting by ID. Some are not: incrementing a counter, appending to a list, sending a message.

To make a non-idempotent operation idempotent, attach an identifier and enforce uniqueness on it:

```sql
ALTER TABLE requests ADD UNIQUE (request_id);

BEGIN;
INSERT INTO requests (request_id, from_account, to_account, amount)
  VALUES ('0286FDB8-D7E1-423F-B40B-792B3608036C', 4321, 1234, 11.00);
UPDATE accounts SET balance = balance + 11.00 WHERE account_id = 1234;
UPDATE accounts SET balance = balance - 11.00 WHERE account_id = 4321;
COMMIT;
```

The second attempt fails the unique constraint and aborts. This works at weak isolation levels, because relational databases maintain uniqueness constraints correctly even when they permit write skew — which is exactly why a constraint beats an application-level "have I seen this ID?" check.

The `requests` table doubles as an event log. The balance updates are derivable from it, which opens the door to applying them downstream instead of inline.

For stream consumers, the equivalent trick is to store the source offset alongside the value you write, then skip any message whose offset you've already recorded. Relying on this assumes replay delivers the same messages in the same order, that processing is deterministic, and that no other writer touches the same value concurrently.

## The end-to-end argument — why the ID must come from the client

This is the part that gets missed. Every layer offers you deduplication, and none of them is sufficient:

- **TCP** dedupes packets, but only within one connection. A reconnect starts fresh.
- **Database transactions** dedupe within a transaction, which is usually scoped to a connection. If the client loses the connection after sending `COMMIT` but before hearing back, it cannot tell whether the transaction committed.
- **2PC** lets a coordinator reconnect and resolve an in-doubt transaction, which fixes the DB layer and nothing above it.
- **Stream processors** offering "exactly-once" guarantee it within the framework's boundary. The moment output leaves — a DB write, an email, an external call — the guarantee stops.

None of these prevent a user on a flaky connection from submitting a form, seeing an error, and pressing submit again. From the server's view that is a second request; from the database's view, a second transaction.

The general principle (Saltzer, Reed, and Clark, 1984): a function like duplicate suppression can only be implemented correctly with the knowledge of the endpoints. Lower layers reduce the probability of the problem; they cannot eliminate it.

**Practically:** generate the operation ID at the true origin — the browser, the mobile app, the calling service — as a UUID or a hash of the meaningful request fields. Thread it through every hop. Enforce it at the store that performs the write.

The same argument applies to integrity checking (Ethernet, TCP, and TLS checksums do not catch corruption introduced by your own software or by the disk) and to encryption (TLS does not protect against a compromised server).

## Multi-step operations without distributed transactions

You often need an operation to be atomic across partitions or services without paying for 2PC. The pattern:

1. Client generates a request ID and submits the operation as **a single message** to one log/table, partitioned by request ID. Single-object writes are atomic almost everywhere, so this either happens or it doesn't.
2. A processor reads that log and emits the derived instructions — one per affected partition — carrying the original request ID.
3. Each downstream processor applies its instruction, deduplicating by request ID.

If step 2's processor crashes and resumes from a checkpoint, it re-emits the same instructions; because it is deterministic and step 3 deduplicates, the end state is unchanged. You get the correctness property (each request applied exactly once to each account) without an atomic commit protocol.

Validation that must reject a request — "this would overdraw the account" — goes in a processor partitioned by the account, before the request enters the log.

## Timeliness vs integrity

Worth separating, because they have different urgency:

- **Timeliness** — is the user seeing a current view? Violations are temporary and self-healing. Stale reads are annoying.
- **Integrity** — is the data corrupt, lost, or self-contradictory? Violations are permanent and require explicit repair.

Slogan: violations of timeliness are eventual consistency; violations of integrity are perpetual inconsistency.

In almost every application integrity matters more. A credit card statement missing today's transaction is normal. A statement whose balance doesn't equal the sum of its transactions is a serious bug. When you're deciding how much consistency machinery to buy, buy integrity first — idempotency, duplicate suppression, atomic single-message writes — and accept lag.

## What to check in the code

- Every `catch` around a network or DB call: does it retry, and is the retried work idempotent?
- Every retry: is the error classified as transient before retrying? Is there backoff and a bound?
- Every queue consumer: what happens when the same message arrives twice?
- Every side effect (email, charge, push, external write): can it fire twice? Does it fire even when the surrounding transaction aborts?
- Every user-submitted mutation: is there an idempotency key originating at the client, or does the server generate one (which doesn't help)?
