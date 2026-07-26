# Extraction: concurrency-memory-model

**Source:** Rust Atomics and Locks — Mara Bos

> Operational rules distilled from the source, written in our own words. This is the intermediate layer between the source text and the published skill — denser than the skill, and the place detail survives for re-distillation.

---

This document is complete and ready. It covers all nine chapters of Mara Bos's book, extracted and rewritten in language-neutral terms (~3540 words), organized into 9 topic clusters with the Trigger/What goes wrong/What to do/How to verify structure, explicit Rust-vs-other-languages callouts, a hard decision procedure defaulting to boring-correct options, and repeated emphasis on why ordinary testing misses these bugs.

# Concurrency Inside One Process: Rules for Coding Agents

Scope: correctness of shared mutable state between threads in one process. The reasoning comes from the C++/Rust memory model, which Java, Go, C#, and Swift all converge on. Where Rust's compiler statically forbids a mistake, that's called out explicitly — in Java, Go, C++, C#, or Swift the same mistake compiles and fails later, under load, on different hardware.

## 1. A data race is undefined behavior, not a wrong answer

**Trigger**: two threads touch the same memory location with no synchronization, and at least one writes.

```
shared int counter = 0
thread A: counter = counter + 1
thread B: counter = counter + 1
```

**What goes wrong**: a compiler assumes, by default, that a location is touched by only one thread at a time, and uses that assumption to cache values in registers, reorder reads/writes around other code, split a read-modify-write into separate steps, or drop "redundant" reads. If a second thread is actually mutating that memory concurrently, every one of those transformations becomes unsound — not just at that line, but potentially earlier, because the compiler reasoned backward from a false assumption. That is why a data race's failure mode is not "off by one," it is "anything, on a build that looks fine." The CPU compounds this: store buffers and out-of-order execution can make a write visible to other cores later than program order suggests, or as a torn (half-written) value.

**What to do**: every mutated location must be reached through exactly one of: a lock, an atomic, or a mechanism that hands exclusive ownership to one thread at a time (a channel, or an immutable value published before any reader starts). Two threads reaching the same mutable field outside one of those is a bug regardless of whether a test ever shows it.

**Language note**: Rust's borrow checker enforces "either many readers or one writer" for ordinary types at compile time — an unsynchronized data race on plain data is a compile error. Java, Go, C++, C#, and Swift have no such check: any thread holding a reference can read or write a field with zero compiler pushback. Garbage collection stops the memory from being freed under you; it does nothing about torn reads, reordering, or stale caches.

**How to verify**: thread sanitizers (TSan for C/C++, Go's `-race` is TSan-based; JCStress for JVM memory-model litmus tests) instrument every access and flag an unsynchronized concurrent access *even on a run that produced the right answer* — that's the key property, since a race might only flip the outcome one run in a million under a normal scheduler. A passing test suite is not evidence of a race-free data structure; it only means the particular interleavings that occurred were survivable.

## 2. Atomics: the operations, compare-and-exchange, and ABA

- **Fused read-modify-write, not load-then-store.** `v = counter.load(); counter.store(v+1)` is not atomic even though each half is — another thread can interleave between them and its update gets lost. Use `counter.fetchAdd(1, ordering)`. Fetch-add/sub/and/or/xor/max/min and swap execute as one indivisible operation and return the *previous* value.
- **Compare-and-exchange** is the general "check then act" primitive: `compareExchange(expected, new, successOrdering, failOrdering)` atomically replaces the value with `new` only if it currently equals `expected`, otherwise returns the actual value so the caller can retry. Every other read-modify-write op can be built from a compare-and-exchange retry loop. Many platforms also expose a "weak" version allowed to fail spuriously even when the value matched — on load-linked/store-conditional hardware (ARM, RISC-V) a stray cache eviction between the linked load and conditional store forces a false failure, and detecting that costs more than retrying. Use weak inside a retry loop; use strong when you only get one shot.
- **The ABA problem**: compare-and-exchange only compares the current value to what you expect — it has no memory of history. If a value goes A → B → A between your read and your compare-and-exchange, the operation succeeds even though the world changed. This is dangerous with pointers specifically: thread 1 reads a pointer to node A off a lock-free stack and stalls; thread 2 pops A, frees it, and a new allocation lands at the same address and gets pushed; thread 1 resumes, its compare-and-exchange matches the bit pattern, and it operates on a semantically different (or freed) object. A plain counter doesn't fix this if it can wrap; the real mitigations are a tagged/versioned pointer (pack a generation counter alongside the pointer in one atomic word), hazard pointers (a reader publishes what it's dereferencing so a reclaimer defers freeing it), or epoch-based reclamation (defer frees until every thread has left the epoch of the removal). These are the actual cost of a lock-free structure, not an afterthought.

**How to verify**: CAS loops with subtly wrong logic often pass every test because contention is rare in a small harness. Stress with many threads and minimal work per iteration to maximize interleavings per second, and prefer a model checker (Loom, JCStress, Relacy/CDSChecker) that exercises every reordering the memory model *allows*, not just the ones your CPU happens to produce today.

## 3. Memory ordering — the core of this

The assumption to unlearn: **"the order I wrote the statements" is not the order they execute in, or the order another thread observes.** Compilers reorder independent-looking operations, and CPUs execute out of order and buffer writes locally before other cores see them. Within one thread none of this is observable; across threads, nothing is guaranteed unless a memory ordering says so.

**Relaxed** guarantees only that operations on the *same* atomic are seen in one consistent order by every thread, and that each operation is atomic. It gives no ordering relative to any other memory operation and no happens-before relationship with another thread. The surprise: a relaxed store to X and a relaxed store to Y from the same thread can be observed by another thread in either order, even though the writer executed them in a fixed sequence. Use relaxed only for values with no dependency on other data — independent counters, IDs, statistics.

**Release/acquire** is what actually moves data safely between threads. A *release* store makes every ordinary write before it (in program order) visible to any thread that later does an *acquire* load of that atomic and observes the released value. That's happens-before: not "eventually visible" but a hard guarantee, enforced by the compiler and by hardware barrier instructions on weakly-ordered CPUs, that nothing before the release reorders past it and nothing after the acquire reorders before it. The publish pattern:

```
thread A:
    payload = compute()          // ordinary write, not atomic
    ready.store(true, release)

thread B:
    while not ready.load(acquire): wait()
    use(payload)                  // guaranteed to see A's write
```

If `ready` used relaxed on both sides, B could observe `ready == true` while still seeing a stale `payload` — the write is allowed to "arrive late." Release/acquire is the mechanism underneath locks and channels, not a separate thing from them. A read-modify-write op can also carry acquire on its read half and release on its write half at once (needed when the same operation must both see prior writers' data and publish its own).

**Sequentially consistent (SeqCst)** adds one more thing on top of acquire/release: a single global total order that every SeqCst operation in the program agrees on, across *all* variables, not just per-variable. This matters only in the specific case where two threads each publish a flag and then check the *other's* flag to decide whether it's safe to touch shared state — with plain release/acquire, both threads' checks can independently see "not set yet" because release/acquire only orders the pair that actually synchronized, so both threads can enter a supposedly-protected section simultaneously. **SeqCst is not a safe default.** It costs extra barrier instructions on weakly-ordered hardware, it does not make an already-broken algorithm correct, and the overwhelming majority of correct concurrent code needs nothing stronger than release/acquire. Seeing SeqCst reached for "to be safe" is a signal the ordering reasoning was never actually done — treat it as a flag for extra review, not as evidence of care.

**How to verify**: these bugs are the least likely of all to show up in testing, because strongly-ordered hardware (x86-64) supplies most acquire/release guarantees "for free" even for code that only asked for relaxed — so wrong ordering can pass every test on x86 and reorder visibly on ARM/RISC-V (mobile, Apple silicon, ARM servers), which actually reorders when not told not to. Do not treat "passes CI on x86" as proof of correct ordering; run the suite on a weakly-ordered target too, or model-check it.

## 4. Building a correct lock

**Trigger**: a naive spinlock retries a compare-and-exchange in a tight loop:

```
while lock.compareExchange(unlocked, locked, acquire, relaxed) fails:
    retry immediately
```

**What goes wrong**: every failed attempt is a *write* attempt, and a write requires exclusive ownership of the cache line — so every spinner constantly yanks the line away from the holder (who needs to write it to unlock) and from every other spinner. This can make the lock slower under contention than doing nothing, and it becomes catastrophic if the scheduler preempts the holder mid-critical-section: every other core burns its whole time slice spinning on a lock that provably cannot be released until the preempted thread runs again.

**What to do**: spin on a plain read first, and only attempt the compare-and-exchange once the read suggests it might succeed — reads of a line can be shared across cores without eviction, so idle spinners stop fighting the holder over ownership. Use a CPU pause/yield hint during the spin so the core deprioritizes the busy-wait. Above all, only spin when the wait is expected to be short (a handful of instructions), because spinning bets that blocking overhead exceeds wait time, and that bet loses badly once the holder can be preempted or migrated. For anything unbounded, block via an OS primitive keyed on the atomic's address (a futex on Linux, `WaitOnAddress` on Windows) so the scheduler stops giving the waiter CPU time, and have the unlocker wake exactly one waiter. The property a hand-rolled "check flag, then sleep" cannot get right: futex-style waits are atomic with respect to the wake — the kernel checks the expected value as part of going to sleep, so a racing wake can never be silently missed. A production-grade lock is hybrid: fast uncontended CAS, brief spin with a pause hint, then fall back to blocking — and it must guarantee the unlock happens even if the critical section throws.

**Language note**: Rust's guard types unlock via `Drop`, which runs even during a panic unwind, so "forgot to unlock on the error path" is structurally impossible with the standard API. Java, Go, C++, C#, and Swift have no automatic equivalent — you must use `try/finally` (Java, C#), `defer Unlock()` (Go), an RAII scope guard (C++), or `defer` (Swift) explicitly, every time, including on rarely-hit error paths.

**How to verify**: mutual-exclusion correctness (two threads observably inside the critical section at once) is catchable by a race detector. Bad spin strategy is usually invisible to unit tests entirely — it shows up as latency cliffs under production load or many-core hardware. "Works on a 4-core laptop" is no evidence about a 64-core server.

## 5. Condition variables

**What goes wrong**:
- *Spurious wakeup*: a wait can return with no corresponding notify. This is allowed by essentially every real implementation and is not a bug to route around — it's part of the contract.
- *Lost wakeup*: if "check the predicate" and "start waiting" aren't atomic relative to the notifier's "change state, then notify," a race window opens — the checker sees the predicate false, the notifier changes state and notifies in that instant, and the checker only then starts waiting, after the notification already fired into the void. The waiter sleeps forever.
- *Thundering herd*: `notifyAll()`/`broadcast()` wakes every waiter; all re-acquire the lock to recheck, one succeeds, the rest immediately re-block, having burned a scheduling slot and a lock acquisition for nothing. This scales linearly with waiter count.

**What to do**: always wait in a loop that rechecks the real predicate, never in an `if`:

```
lock.lock()
while not predicate(state):
    condvar.wait(lock)    // atomically unlocks, sleeps, relocks on return
lock.unlock()
```

The loop absorbs spurious wakeups for free. The lost-wakeup race closes because `wait` performs "unlock and begin waiting" as one atomic step relative to a notifier that mutates state and notifies *while still holding the same lock* — which is exactly why the mutation and the notify must happen before releasing that lock, not after. For thundering herd, prefer `notifyOne()` when only one waiter can possibly make progress; reserve `notifyAll()` for when every waiter's predicate might genuinely now be true.

**Language note**: the predicate-loop discipline is identical in Java (`wait()`/`notify()`), C++ (`condition_variable::wait` with a predicate overload — use it), Go (`sync.Cond`, no predicate overload, loop by hand), C# (`Monitor.Wait`/`Pulse`), and Swift (`NSCondition`). No language's compiler enforces the loop — it's a discipline everywhere, including Rust.

**How to verify**: lost wakeups and thundering herd are timing bugs by definition — a test that happens to schedule notify after wait every time will never see the race. Stress-run with many iterations under injected scheduling noise (priority churn, `yield()`/sleep jitter between check and wait) to widen the window, or use a deterministic-scheduling tool that can force the "notify lands in the gap" interleaving on demand.

## 6. Lock-free structures are usually a trap

**What goes wrong**: the hard part of a lock-free structure is almost never the compare-and-exchange loop — it's memory reclamation, exactly where a memory-ordering mistake becomes a memory-safety mistake instead of a logic mistake. Canonical example, manual atomic reference counting: incrementing on clone can safely use relaxed ordering (the thread already had valid access before and after — no new visibility requirement). Decrementing the *last* reference, the one that frees memory, needs release ordering on the decrement itself, plus an acquire fence executed by whichever decrement actually observes the count hitting zero, before that thread frees the memory. Skip the acquire fence and another thread's writes made just before its own decrement might not yet be visible to the freeing thread — meaning the free can race with work another thread believed finished, or two threads can each conclude they're last and double-free. Get this wrong and the bug is use-after-free or double-free, not "count is off by one."

**What to do**: default to a mutex. Boring and correct beats fast and occasionally corrupts memory, every time performance isn't measured and proven to require the alternative. If a lock-free structure is genuinely needed, treat reclamation as the actual design problem — pick a real strategy (hazard pointers, epoch-based reclamation, or the relaxed-increment/release-decrement/acquire-fence-on-zero pattern above), not "I'll be careful." Watch for ABA (section 2) anywhere nodes get recycled or addresses get reused.

**How to verify**: this category is where sanitizers and model checkers earn their keep most, because the bugs are rare-interleaving memory corruption — close to the worst case for "will a normal test catch it" and close to the best case for "will a formal tool catch it." A lock-free structure that has never run under TSan/Loom-equivalent tooling with heavy concurrent stress should be treated as unverified regardless of production uptime — absence of observed corruption is not evidence of absence, only evidence the triggering interleaving hasn't happened yet.

## 7. Cache coherence and false sharing

**Trigger**: two or more independently-used variables (often unrelated atomics — a per-thread counter, a stats field, a flag) happen to sit close enough in memory to land in the same cache line (typically 64 bytes).

**What goes wrong**: coherence protocols keep memory consistent at the granularity of a whole cache line, not a whole variable — this is also what makes atomics physically implementable. When core A writes "its" variable, the hardware invalidates every other core's cached copy of the *entire line*, including "unrelated" variable B on it, even though A and B never logically touch each other. Two threads on two completely independent, uncontended variables can ping-pong a cache line exactly as if they were contending on the same variable, tanking throughput by an order of magnitude. This is purely a performance bug: every read returns a correct, fully-synchronized value — no race, no ordering violation — which is what makes it so disorienting: the symptom (severe, contention-shaped slowdown scaling with thread count) looks exactly like a correctness-level synchronization bug, but a race detector finds nothing wrong, because nothing is wrong at the logical level, only at the memory-layout level.

**What to do**: pad or align independently-hot shared variables so each lands on its own cache line — align to 64 bytes, or insert explicit padding between hot fields owned by different threads. Especially relevant for per-thread counters packed into an array, and for lock state words sitting next to other frequently-written fields.

**How to verify**: invisible to correctness testing and to race detectors by design (there's no race to detect). Verification means profiling — hardware coherence-miss counters (`perf c2c` on Linux or the CPU vendor's equivalent), or a micro-benchmark comparing padded vs. unpadded layouts under load. If throughput doesn't scale with threads on independent-looking work and correctness tests are green, suspect layout before logic.

## 8. The decision procedure

Given shared mutable state, choose in this order and stop at the first fit — don't skip ahead toward the faster-looking option without a measured reason:

1. **Can it be immutable after construction and handed off before any thread reads it?** If yes, no primitive is needed beyond whatever safely performed the hand-off (thread spawn/join carries a happens-before edge; so does a channel send). No sharing, no race, nothing to reason about — this is the actual default, above even a mutex.
2. **Is it a single primitive value — a count, a flag, an ID generator, a pointer swap — with no invariant spanning it and other fields?** Use an atomic, with the weakest ordering provably sufficient: relaxed for values with no cross-thread data dependency, release/acquire for anything publishing other data alongside it, SeqCst only for the rare mutual-visibility pattern in section 3. Write down which happens-before edge you're relying on — "I used an atomic" is not itself an ordering argument.
3. **Does correctness depend on an invariant across multiple fields, or a critical section doing more than one memory operation atomically?** Use a mutex (or a reader-writer lock if reads dominate and are provably safe concurrently). This is the default for anything that isn't clearly case 1 or 2 — reach for it first, replace it with something cleverer only after profiling proves it's a bottleneck.
4. **Is the shape "one thread produces data for another to consume, with no further shared mutation after that"?** Use a channel — it's the hand-off pattern from case 1 packaged as a reusable primitive, and a consuming API (below) can make "used twice" or "read before sent" a compile error instead of a race.
5. **Only after 1–4 are ruled out, and only with a measured problem a mutex demonstrably causes**, consider a hand-rolled lock-free structure — as its own project with a reclamation-strategy design and a model-checking verification budget (section 6), not a drop-in swap.

**Language note**: Rust's channel/one-shot APIs typically consume `self` on send, so calling `send` twice is a compile-time "use of moved value" error, and the type system can strip cross-thread capability from a handle entirely. Java, Go, C++, C#, and Swift have no by-value-consumption enforcement for ordinary references — sending twice, or receiving before checking readiness, compiles fine and is caught only by a runtime check the library author remembered to add, or not at all. When implementing or reviewing a hand-off primitive in those languages, add the runtime guard explicitly; don't assume the type system has your back.

## 9. Why testing doesn't find these bugs

Every bug class above shares a property: the wrong interleaving, reordering, or cache state has to actually occur during a run for an assertion to fail, and on typical dev hardware (strongly-ordered x86, low core counts, low contention) the buggy interleaving can be astronomically rare or hardware-unreachable while remaining fully permitted by the memory model. A green CI run proves the tested interleavings were fine and says nothing about the ones not exercised. What actually addresses this: (1) dynamic race/ordering detectors (TSan and its per-language equivalents) instrument every real run and catch the unsynchronized-access pattern directly, independent of whether that run's timing happened to produce a wrong answer; (2) model-checking schedulers (Loom-style tools, JCStress for the JVM memory model, Relacy/CDSChecker for C++) don't rely on luck — they systematically enumerate the legal interleavings and orderings a correct implementation must survive, for a bounded but exhaustive slice of the state space, and find bugs no amount of random stress-testing would hit; (3) cross-architecture testing (the real suite on ARM, not just x86) surfaces ordering bugs strongly-ordered hardware hides for free; (4) for false sharing specifically, profiling with hardware coherence counters — not correctness testing — is the only lever that shows anything at all. "We have tests and they pass" is necessary and nowhere near sufficient for any code in this scope.
