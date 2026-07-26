# Memory Ordering

## The assumption to unlearn

**The order you wrote the statements is not the order they execute, and not the order another thread observes.**

Compilers reorder operations that look independent. CPUs execute out of order and buffer writes locally before other cores see them. Within a single thread none of this is observable — the hardware and compiler conspire to preserve the illusion. Across threads, nothing is guaranteed unless a memory ordering explicitly says so.

## Why a data race is undefined behavior

A compiler assumes by default that a memory location is touched by one thread at a time. On that assumption it will cache the value in a register, reorder reads and writes around other code, split a read-modify-write into separate steps, or delete a "redundant" reload.

If another thread is concurrently mutating that location, every one of those transformations becomes unsound — and not only at that line, because the compiler reasoned *forward* from a false premise and may have transformed code around it.

That is why a data race's failure mode is not "off by one." It is "anything at all, on a build that looks fine." The CPU compounds it: store buffers can make a write visible to other cores later than program order implies, or expose a torn, half-written value.

**Language note.** Rust's borrow checker enforces many-readers-or-one-writer for ordinary types at compile time, so an unsynchronized data race on plain data is a compile error. Java, Go, C++, C#, and Swift have no such check — any thread holding a reference can read or write with zero pushback. Garbage collection prevents the memory being *freed* underneath you; it does nothing about torn reads, reordering, or stale caches.

## The orderings

### Relaxed

Guarantees only that operations on **the same** atomic are seen in one consistent order by all threads, and that each operation is individually atomic. No ordering relative to any other memory operation, and no happens-before relationship with another thread.

The surprise: a relaxed store to X followed by a relaxed store to Y can be observed by another thread **in either order**, even though the writing thread executed them in sequence.

Use it only for values with no dependency on other data — independent counters, statistics, ID generators.

### Release / acquire

This is what actually moves data between threads safely.

A **release** store makes every ordinary write before it, in program order, visible to any thread that subsequently performs an **acquire** load of that same atomic and observes the released value. That's a happens-before edge: not "eventually visible" but a hard guarantee, enforced by the compiler and by barrier instructions on weakly-ordered hardware.

The publish pattern:

```
thread A:
    payload = compute()              # ordinary write, not atomic
    ready.store(true, release)

thread B:
    while not ready.load(acquire):
        wait()
    use(payload)                     # guaranteed to see A's write
```

**If both used relaxed**, thread B could observe `ready == true` while still reading a stale `payload` — the write is permitted to arrive late. That's the whole reason the ordering exists.

Release/acquire is the mechanism *underneath* locks and channels, not an alternative to them.

A read-modify-write operation can carry acquire on its read half and release on its write half simultaneously, which is what you need when the same operation must both observe prior writers' data and publish its own.

### Sequentially consistent

Adds one thing on top of acquire/release: a single global total order that every sequentially-consistent operation agrees on, across **all** variables rather than per-variable.

That matters in exactly one common situation: two threads each publish a flag and then check the *other's* flag to decide whether it's safe to proceed. Under plain release/acquire both checks can independently observe "not set yet," because release/acquire only orders the pair that actually synchronized — so both threads enter a section that was supposed to be exclusive.

**Sequential consistency is not a safe default.** It costs extra barriers on weakly-ordered hardware, and it does not make an already-broken algorithm correct. The overwhelming majority of correct concurrent code needs nothing stronger than release/acquire.

Reaching for it "to be safe" is usually a sign the ordering argument was never constructed. Treat its appearance as a prompt for review rather than evidence of care.

## Choosing an ordering

Write down the happens-before edge you are relying on. If you can't state it — "thread B's acquire load of `ready` synchronizes with thread A's release store, so A's writes to `payload` are visible" — you haven't chosen an ordering, you've guessed.

- No cross-thread data dependency → relaxed.
- Publishing or consuming other data alongside the atomic → release/acquire.
- Mutual visibility between two independently-publishing threads → sequentially consistent, and only then.

## Why testing does not find these bugs

This is the category least likely to surface in tests, for a structural reason: **x86-64 supplies most acquire/release guarantees for free**, even to code that only requested relaxed. Incorrect ordering therefore passes every test on typical development and CI hardware, and reorders visibly on ARM and RISC-V — which now means mobile devices, Apple silicon, and a growing share of servers.

"Passes CI on x86" is not evidence of correct ordering.

What actually helps:

- **Run the suite on a weakly-ordered target** as well as x86.
- **Model checkers** (Loom, JCStress, CDSChecker) enumerate the reorderings the memory model *permits*, rather than the ones your CPU happens to produce today.
- **Thread sanitizers** catch the unsynchronized access itself, independent of whether that particular run produced a wrong answer — which is the property that makes them worth more than stress testing.
