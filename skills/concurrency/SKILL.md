---
name: concurrency
description: Get shared mutable state right inside one process — choosing between immutability, an atomic, a mutex and a channel; memory ordering and why relaxed surprises people; condition variable predicates; deadlock and lock ordering; false sharing. Use when writing or reviewing multithreaded code, adding locking, using atomics, debugging a race or a hang, or reasoning about anything two threads touch.
---

# Concurrency

## Purpose

Concurrency bugs are the ones your tests tell you don't exist.

A green suite proves the interleavings that happened were survivable. It says nothing about the ones that didn't happen — and on a strongly-ordered dev machine with four cores and no contention, the dangerous ones may be astronomically rare or literally unreachable while remaining entirely permitted by the memory model. Then production runs on 64 cores under load, or a user runs it on ARM, and the bug arrives.

This skill covers correctness *inside* one process. For correctness across processes, see `distributed-data`.

## Where the bugs actually are

Worth knowing before you spend attention: a study of 105 real concurrency bugs fixed in MySQL, Apache, Mozilla and OpenOffice found **74 were non-deadlock, 31 were deadlock** — and of the non-deadlock bugs, **97% were atomicity violations or order violations.**

So the dominant real-world concurrency bug is not an exotic multi-lock cycle. It's:

**Atomicity violation** — a check-then-act sequence with no lock spanning the gap:

```
if resource.field != null:      # thread A checks
    use(resource.field)         # thread B nulled it in between
```

**Order violation** — assuming one thread finished before another starts, with nothing enforcing it:

```
child = spawn(worker)           # worker uses state
                                # the spawner hasn't finished setting up
```

Both are missing synchronization around something obvious in hindsight. Review for these two shapes first — they're where the returns are.

## The decision procedure

Given shared mutable state, take the **first** option that fits. Don't skip toward the faster-looking one without a measured reason.

1. **Can it be immutable after construction, published before any reader starts?** Then nothing is shared mutably and there's no race to reason about. Thread spawn/join and channel sends both carry the necessary ordering. This is the real default — above even a mutex.
2. **Is it a single primitive value** — a counter, a flag, an ID generator, a pointer swap — with no invariant spanning it and other fields? Use an atomic, with the weakest ordering you can actually justify. Write down which happens-before edge you're relying on; "I used an atomic" is not an ordering argument.
3. **Does correctness depend on an invariant across multiple fields**, or on a critical section doing more than one operation indivisibly? Use a mutex. This is the default for anything not clearly case 1 or 2.
4. **Is it one thread producing for another, with no shared mutation afterward?** Use a channel.
5. **Only after 1–4 are ruled out, with a measured problem a mutex demonstrably causes**, consider lock-free — as its own project with a memory-reclamation design and a model-checking budget, not a drop-in swap.

## Rules that apply without loading anything

**1. A data race is undefined behavior, not a wrong answer.** The compiler assumes single-threaded access and optimizes on that basis — caching in registers, reordering, splitting read-modify-writes. When the assumption is false those transformations become unsound, so the failure mode is "anything," not "off by one."

**2. Read-modify-write must be one operation.** `v = counter.load(); counter.store(v+1)` is not atomic even though both halves are. Use a fused `fetchAdd`.

**3. Always loop on the predicate when waiting.** `if (!ready) wait()` is wrong. Signal means "recheck," not "the predicate now holds" — another thread can invalidate it before you're scheduled, and spurious wakeups are permitted. Use `while`.

**4. The lock must span check *and* act.** Locking around the check and separately around the action leaves the gap open. That gap is the single most common concurrency bug there is.

**5. Sequential consistency is not a safe default.** It costs barriers on weakly-ordered hardware and does not rescue a broken algorithm. Reaching for it "to be safe" usually means the ordering reasoning was never done — treat it as a review flag, not evidence of care.

**6. Order locks by an intrinsic property, never by argument position.** `transfer(a, b)` and `transfer(b, a)` running concurrently deadlock. Order by address or ID so the cycle can't form.

**7. Passing on x86 proves little about ordering.** Strongly-ordered hardware supplies most acquire/release guarantees for free, so incorrect ordering passes CI and reorders visibly on ARM — which is now mobile, Apple silicon, and a growing share of servers.

**8. Pad independently-hot shared variables.** Coherence works on whole cache lines, so two unrelated variables sharing one ping-pong between cores as if contended. Correct results, an order of magnitude slower, and no race detector will say a word.

## Triage

| What you're doing | Reference |
|---|---|
| Atomics, memory ordering, publishing data between threads | [memory-ordering](references/memory-ordering.md) |
| Mutexes, condition variables, semaphores, deadlock, lock ordering | [locks-and-coordination](references/locks-and-coordination.md) |
| Compare-and-exchange loops, lock-free structures, reclamation, false sharing | [lock-free-and-hardware](references/lock-free-and-hardware.md) |

## How these bugs are actually found

Not by testing. Specifically:

- **Race detectors** (TSan, Go's `-race`, JCStress) instrument every access and flag unsynchronized concurrent access **even on a run that produced the right answer**. That property is what makes them worth more than any amount of stress testing.
- **Model checkers** (Loom, JCStress, CDSChecker) enumerate the interleavings and reorderings the memory model *permits*, rather than the ones your CPU happened to produce today.
- **Cross-architecture testing** — run the suite on ARM, not only x86.
- **Deliberate perturbation** — inject a delay at exactly the gap an atomicity or order violation identifies, and a one-in-a-million bug becomes reliable.
- **For false sharing**, hardware coherence counters. Correctness tooling finds nothing because nothing is logically wrong.

## A note on language

The reasoning here comes from the C++/Rust memory model, which Java, Go, C#, and Swift converge on. Where a language's type system *prevents* a mistake, the references say so — because in the languages that don't prevent it, the same mistake compiles cleanly and fails later, on different hardware, under load.
