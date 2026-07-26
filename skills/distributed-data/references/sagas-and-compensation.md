# Sagas and Compensation

The sibling of [keeping-stores-in-sync](keeping-stores-in-sync.md). That reference argues against distributed transactions; this one covers what you owe once you've split an operation into several transactions instead.

Applies to anything spanning more than one transaction because of cross-service calls, human approval, long external processing, or contention — the shape underneath Temporal workflows, Step Functions, and every hand-rolled orchestrator.

## Two distinct failures, and the second is the one people miss

**Missing compensation.** Teams build the forward chain and skip the reverse. A mid-sequence failure leaves partial committed state — money debited from one account, never credited to the other — with nothing recording which steps actually ran, so nothing can clean it up automatically.

**The isolation gap.** This is the subtle one. The moment you split one operation into several transactions, **the isolation guarantee is gone**, and compensation logic doesn't restore it. Anything reading between steps — an audit, a report, another user — can observe a state that could never occur in any serial execution of the whole business operation. Money debited from A and not yet credited to B looks exactly like money vanishing.

Holding locks across the whole saga to prevent that reintroduces the contention and availability cost that made you split the transaction in the first place. So don't.

## What to do

**Write the compensating step alongside each forward step**, not after the forward path works. A compensation written later is written under time pressure by someone reconstructing what the forward step did.

**Persist progress inside each step's own transaction** — which steps committed, and enough data to invoke each compensation — as a row or queue element written *as part of* that step, not as a follow-up. If the progress record is a separate write, you have a dual write, and [keeping-stores-in-sync](keeping-stores-in-sync.md) applies to it.

**Run a watchdog** that finds sagas stuck past a timeout and executes compensations for whatever committed. Without one, a saga that dies between steps stays half-applied indefinitely and nothing notices.

**For the isolation gap, make the in-flight state visible rather than hiding it.** Design the read paths that can observe it to represent it honestly — a `pending transfer` status, or a suspense account that a reconciliation report adds back. Presenting an in-flight view as though it were final is what turns an expected intermediate state into a support ticket.

**Verify.** For every saga: *if this stops after step k, what runs to clean up steps 1..k, and where is the durable record of which ones ran?* No answer means the compensation half doesn't exist. Then separately: *what does a concurrent reader see during the gap, and does that reader's code account for it?*

## Reliable request/reply over a queue

The client-side companion. A client submits a request onto a queue and later reads a reply from another — a deliberate alternative to a synchronous call across a boundary where no shared transaction manager exists.

**What goes wrong.** After a crash or reconnect, naive clients guess: *no reply yet, so resubmit.* That either duplicates the request or waits forever for a reply that already arrived and was missed.

**The state space is small and nobody enumerates it.** After a request is built there are exactly four states:

1. Never submitted
2. Submitted, no reply yet
3. Reply arrived, not yet processed
4. Reply already processed

**Make it decidable rather than inferred.** Durably record, alongside the request, the ID of the last request actually enqueued and the ID of the last reply actually consumed. On reconnect, compare the request's ID against those two markers to land in exactly one state. **Never infer the state from a timeout** — a timeout cannot distinguish state 2 from state 3.

**Verify** by killing the client at each of the four points and confirming recovery reaches the right state without duplicating the request or losing the reply.

## Where this sits

- **[keeping-stores-in-sync](keeping-stores-in-sync.md)** — why not to use a distributed transaction.
- **This file** — what you owe once you've split the operation instead.
- **[idempotency-and-retries](idempotency-and-retries.md)** — the end-to-end request ID that makes each step safely re-runnable, which a saga depends on since any step may be retried by the watchdog.
