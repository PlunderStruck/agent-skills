# Clocks, Leases, and Fencing

Applies to timestamps, TTLs, expiry checks, distributed locks, leader election, scheduled work, and any ordering derived from time.

## Two clocks, two jobs

- **Monotonic clock** (`CLOCK_MONOTONIC`, `System.nanoTime()`, `performance.now()`) — measures elapsed time. Always moves forward. Its absolute value is meaningless, and **comparing it across machines is meaningless.** Use it for timeouts, durations, and latency measurement.
- **Time-of-day clock** (`CLOCK_REALTIME`, `System.currentTimeMillis()`, `Date.now()`) — returns wall-clock time, synchronized by NTP. Use it for timestamps that humans or other systems will interpret. **Do not use it to measure elapsed time**, because it can jump backward.

Using the wrong one is a common and silent bug: measuring a duration with wall-clock time produces negative durations when NTP steps the clock backward.

## Why wall-clock time can't be trusted

- Quartz drift, temperature-dependent. Google budgets 200 ppm — about 17 seconds/day if unsynchronized.
- NTP forcibly resets a clock that's drifted too far; applications observing the reset see time jump backward or forward.
- A node firewalled off from NTP drifts silently. Nothing breaks visibly; it just gets further from reality.
- NTP accuracy is bounded by network round-trip. ~35 ms is achievable on the public internet at best, with spikes to a second under congestion.
- Leap seconds produce 59- and 61-second minutes and have crashed large systems.
- In VMs the clock is virtualized; when the hypervisor pauses a VM the clock appears to jump forward on resume.
- On user-controlled devices, the clock may be deliberately wrong.

A clock reading is not a point — it's an interval with a confidence bound. Almost no API exposes that bound. Google's TrueTime does, returning `[earliest, latest]`, and Spanner uses non-overlapping intervals to order transactions, deliberately waiting out the uncertainty before committing. Outside that, `clock_gettime()` gives you nanosecond resolution with no indication of whether it's accurate to 5 ms or 5 years.

## Never order events by timestamp

Two nodes, well-synchronized to within 3 ms, can still stamp a causally-later write with an earlier timestamp. When a third node resolves the conflict by keeping the higher timestamp, the later write is silently discarded.

This is **last-write-wins**, and it's the default conflict resolution in Cassandra, an option in Riak, and common in hand-rolled multi-writer code. Its failure modes:

- Writes disappear with no error reported to the application. A node with a lagging clock cannot overwrite a node with a fast clock until the skew elapses.
- It cannot distinguish "sequential, in quick succession" from "genuinely concurrent" — the two need different handling.
- Two nodes can generate identical millisecond timestamps, requiring a tiebreaker that itself violates causality.

LWW is only safe when a key is **written once and thereafter immutable** — which is why the recommended Cassandra pattern is a UUID per write.

For correct ordering use logical clocks: a sequence from a single log, version vectors, or Lamport timestamps. These track relative order rather than physical time, which is what you actually needed.

For client-reported event times (mobile apps buffering offline), log three timestamps — event time per device clock, send time per device clock, receive time per server clock — and use the difference between the last two to estimate and correct the device's offset.

## Process pauses break "check then act"

```java
if (lease.expiryTime - System.currentTimeMillis() > 10_000) {
    process(request);   // <-- the pause can happen right here
}
```

A thread can stop for an arbitrary length of time between the check and the action:

- Stop-the-world GC pauses, sometimes minutes
- VM suspension and live migration
- OS context switches and hypervisor steal time
- Synchronous disk I/O, including surprises like lazy classloading
- Page faults and swap thrashing
- `SIGSTOP` — including by accident

During the pause the rest of the cluster proceeds, times the node out, and elects a new leader. The paused node resumes with no indication that anything happened and continues acting on its expired authority. There is no amount of margin in the timeout check that makes this safe, because the pause is unbounded.

## Fencing tokens

The fix is to stop trusting the client's belief and make the resource enforce ordering.

Every time the lock service grants a lock or lease, it also returns a **monotonically increasing token**. Every write to the protected resource carries the token. The resource records the highest token it has processed and **rejects any write bearing a lower one**.

```
client 1 acquires lease, token = 33
client 1 pauses; lease expires
client 2 acquires lease, token = 34
client 2 writes with token 34  -> accepted, resource records 34
client 1 resumes, writes with token 33 -> REJECTED
```

Two things matter:

- **The resource must do the checking.** A client verifying its own lock status is worthless, because the client is the thing that was paused. If the resource has no native support, encode the token where the resource will enforce it — in the filename, in a conditional write predicate, in a `WHERE token > ?` clause.
- **The token must be monotonic.** With ZooKeeper, the `zxid` or node `cversion` works. A timestamp does not.

Fencing is also required when failing over stream-processing or job-runner nodes, for exactly the same reason.

## Truth is defined by the majority

A node cannot trust its own judgment about whether it is alive, whether it is the leader, or whether it holds the lock. A semi-disconnected node — one that receives but cannot send — is functioning perfectly and will still be declared dead. A GC-paused node was, for practical purposes, dead.

Decisions like "who is the leader" and "which node is dead" have to come from a quorum, and the individual node must accept the quorum's verdict even when it disagrees. There can only be one majority, which is what makes the decision safe.

Corollary for application code: if your design has a step where one node decides it is special and acts on that alone, it needs either a quorum or a fencing token, and usually both.

## What to check in the code

- `Date.now()` / `System.currentTimeMillis()` used to compute a duration — should be monotonic.
- Timestamps compared across machines to decide which write wins.
- Any distributed lock (Redis `SETNX`, ZooKeeper, etcd, a DB row) where the protected write does not carry and verify a token.
- TTL or expiry logic that assumes the check and the action are adjacent in time.
- Timestamps supplied by clients and trusted for ordering, billing, or windowing.
- Scheduled jobs that assume they cannot run concurrently with themselves — a paused-then-resumed instance overlaps the next scheduled run.
