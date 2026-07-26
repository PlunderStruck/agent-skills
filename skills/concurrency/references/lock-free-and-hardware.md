# Atomics, Lock-Free Structures, and Hardware Effects

## Atomic operations

**A load followed by a store is not atomic**, even though each half is:

```
v = counter.load()
counter.store(v + 1)      # another thread's update is lost in the gap
```

Use a fused read-modify-write — `fetchAdd`, `fetchSub`, `fetchAnd`, `swap` — which executes indivisibly and returns the *previous* value.

## Compare-and-exchange

The general check-then-act primitive: atomically replace the value with `new` only if it currently equals `expected`, otherwise return what it actually holds so the caller can retry. Every other read-modify-write can be built from a retry loop around it.

**Weak versus strong.** Many platforms expose a "weak" variant permitted to fail spuriously even when the value matched. On load-linked/store-conditional hardware — ARM, RISC-V — a stray cache eviction between the linked load and the conditional store forces a false failure, and detecting that costs more than simply retrying.

Use weak inside a retry loop. Use strong when you get one attempt.

## The ABA problem

Compare-and-exchange compares the current value against what you expect. It has no memory of history.

If a value goes A → B → A between your read and your exchange, the operation **succeeds** — even though the world changed underneath it.

This is dangerous with pointers specifically:

1. Thread 1 reads a pointer to node A off a lock-free stack, then stalls.
2. Thread 2 pops A, frees it, and a new allocation lands at the same address and gets pushed.
3. Thread 1 resumes. Its compare-and-exchange matches the bit pattern and succeeds — operating on a semantically different, or already-freed, object.

A plain counter doesn't fix this if it can wrap. The real mitigations:

- **Tagged or versioned pointers** — pack a generation counter alongside the pointer in one atomic word.
- **Hazard pointers** — a reader publishes what it's currently dereferencing, so a reclaimer defers freeing it.
- **Epoch-based reclamation** — defer frees until every thread has left the epoch in which the removal happened.

These are the actual cost of a lock-free structure, not an afterthought.

## Why lock-free is usually a trap

The hard part is almost never the compare-and-exchange loop. It's **memory reclamation** — and that's precisely where a memory-ordering mistake stops being a logic bug and becomes a memory-safety bug.

The canonical example is manual atomic reference counting:

- **Incrementing** on clone can use relaxed ordering. The thread already had valid access before and after, so no new visibility requirement is created.
- **Decrementing** needs release ordering, **plus an acquire fence** executed by whichever decrement observes the count reaching zero, before that thread frees the memory.

Skip the acquire fence and another thread's writes made just before its own decrement may not be visible to the freeing thread. The free then races with work another thread believed complete — or two threads each conclude they were last and double-free.

Get this wrong and the symptom is use-after-free or double-free, not "the count is off by one."

**Default to a mutex.** Boring and correct beats fast and occasionally memory-corrupting, every time performance hasn't been measured and proven to require the alternative.

If a lock-free structure is genuinely warranted, treat reclamation as *the* design problem and pick a real strategy — not "I'll be careful."

**Verification.** This is the category where sanitizers and model checkers earn the most, because the bugs are rare-interleaving memory corruption: close to the worst case for ordinary testing, close to the best case for formal tooling. A lock-free structure that has never run under a thread sanitizer or a model checker with heavy concurrent stress should be treated as **unverified**, regardless of production uptime. Absence of observed corruption is evidence that the triggering interleaving hasn't happened yet, not that it can't.

## False sharing

**Trigger.** Two or more independently-used variables — per-thread counters, a stats field, a flag — happen to sit close enough in memory to land in the same cache line, typically 64 bytes.

**What happens.** Coherence protocols maintain consistency at the granularity of a whole cache line, not a variable. When one core writes *its* variable, the hardware invalidates every other core's copy of the **entire line**, including the logically unrelated variable sharing it.

Two threads working on completely independent, uncontended variables therefore ping-pong a cache line exactly as if they were contending — throughput can drop by an order of magnitude.

**Why it's disorienting.** This is purely a performance bug. Every read returns a correct, fully synchronized value. There is no race and no ordering violation. But the symptom — severe contention-shaped slowdown that worsens with thread count — looks exactly like a synchronization *correctness* bug, and a race detector finds nothing, because nothing is wrong at the logical level. The problem is memory layout.

**The fix.** Pad or align independently-hot shared variables so each occupies its own cache line. Most relevant for per-thread counters packed into an array, and for a lock's state word sitting beside other frequently written fields.

**Verification.** Invisible to correctness testing and to race detectors by construction. You need profiling — hardware coherence-miss counters (`perf c2c` or the vendor equivalent), or a micro-benchmark comparing padded against unpadded layouts under load.

**The diagnostic:** if throughput fails to scale with thread count on work that looks independent, and correctness tests are green, suspect layout before logic.
