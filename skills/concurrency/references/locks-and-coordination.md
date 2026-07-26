# Locks, Condition Variables, and Deadlock

## The two bugs that dominate

Of real-world non-deadlock concurrency bugs, roughly 97% are one of these two shapes. Review for them first.

### Atomicity violation

```
if resource.field != null:       # thread A checks
    use(resource.field)          # thread B set it null in the gap
```

The code assumed a group of accesses would be indivisible without enforcing it. A passes the check, loses the CPU, B invalidates the thing A just confirmed, A resumes and uses it.

**Fix:** hold **one** lock across the entire check-and-use region, in every thread that touches it. Locking separately around the check and around the use leaves exactly the gap that matters.

**Find it:** look for check-then-use on shared state where the use isn't adjacent to the check. Insert an artificial delay between them under load — a real bug then reproduces reliably instead of rarely.

### Order violation

```
child = spawn(worker)            # worker reads state
                                 # spawner hasn't finished writing it
```

Nothing enforces that setup happens before use; the code merely assumes it.

**Fix:** replace the assumption with an explicit condition variable or semaphore. **A bare flag is not sufficient** — see lost wakeups below.

**Find it:** force the consumer to run first by delaying the producer. This class is invisible under ordinary scheduling and appears only once the less-common ordering is forced.

## Condition variables

Two rules, both non-negotiable.

### Always loop on the predicate

```
lock(m)
while not predicate(state):      # while, never if
    condvar.wait(m)              # atomically unlocks, sleeps, relocks
unlock(m)
```

Signalling means "something changed, recheck." It is **not** a promise the predicate still holds when the woken thread is finally scheduled — another thread can consume the item in between. Spurious wakeups are also permitted by essentially every real implementation, and are part of the contract rather than a bug to route around.

The loop handles both for free. No language's compiler enforces it, including Rust.

**Find it:** flag every `wait()` not inside a `while`. Stress with more waiters than signallers — a one-to-one test is far less likely to expose the `if` version.

### Hold the same lock across check-and-wait and update-and-signal

```
# BROKEN — no shared lock
waiter:    if not done: wait(cond)
signaller: done = true; signal(cond)
```

The signaller can run entirely between the waiter's check and its `wait()` call. The signal fires with nobody asleep, then the waiter sleeps forever. This is a **lost wakeup**.

It closes because `wait()` performs "release the lock and begin waiting" as one atomic step relative to a signaller who mutates state and signals **while still holding that same lock** — which is why the mutation and the signal must both happen before releasing it.

Keep an explicit state variable so a waiter whose condition is already true never needs the signal at all.

### Signal versus broadcast

`signal()` wakes one arbitrarily chosen waiter. If waiters are waiting for *different things* — producers and consumers sharing one condition variable, or differently-sized allocation requests — the runtime can wake one whose predicate is still false. It rechecks, sleeps again, and the wakeup that belonged elsewhere is gone. In the worst case everyone ends up asleep.

**Fix:** one condition variable per waiter class where they're distinguishable. Where they aren't, use `broadcast()` — wake everyone, let each recheck, let the unsatisfied go back to sleep. This costs spurious wakeups but never silently loses one.

The cost of broadcast is a **thundering herd**: every waiter wakes, contends for the lock, one proceeds, the rest re-block having burned a scheduling slot each. That scales linearly with waiter count.

If switching `signal()` to `broadcast()` is the only way to fix a failing test, work out *why the wrong thread was targeted* before accepting it as the answer.

## Deadlock

Four conditions must all hold: **mutual exclusion**, **hold-and-wait**, **no preemption**, **circular wait**. Break any one and deadlock is impossible — but they are not equally practical to attack.

| Condition | Why it's usually impractical |
|---|---|
| Mutual exclusion | It's why the lock exists; removing it means lock-free structures |
| Hold-and-wait | Requires acquiring every lock atomically up front, collapsing concurrency |
| No preemption | Try-lock with backoff works but introduces livelock and partial-work unwinding |
| **Circular wait** | **The practical target** |

### Lock ordering

Enforce a single global order on lock acquisition so a cycle cannot form.

The classic failure:

```
transfer(a, b):  lock(a); lock(b); ...
```

called concurrently as `transfer(X, Y)` and `transfer(Y, X)`. Each locks its first argument and blocks on its second. The cycle is invisible in `transfer`'s own source — it exists only across two call sites that never appear together in a diff.

**Never order by argument position when arguments are interchangeable.** Order by an intrinsic property: lock whichever has the lower address or ID first, regardless of parameter position.

For many distinct lock types, maintain a documented order (row before index before page) and audit every multi-lock site against it.

### Why ordering erodes

It's convention, not compiler-enforced:

- One careless addition anywhere violates it.
- **Encapsulation actively fights it.** A call into another module gives you no visibility into what it locks or in what order, so an innocuous-looking call chain can close a cycle.
- A newly introduced lock must be verified against **every** existing lock, not only the ones it appears to interact with.

**Find it:** build a lock-order graph checker that records which lock was taken first whenever two are held together, and flags any pair seen in both orders. This catches what review misses, because the two conflicting call sites are rarely read together. Test under heavier contention than production expects.

## Semaphores versus mutexes

A semaphore is a counter with `wait` (decrement, block if negative) and `post` (increment, wake one). A mutex is the binary case initialized to 1.

**Rule of thumb: initialize to the number of resources you're willing to hand out immediately.**

- **Mutual exclusion** — initialize to 1; acquire and release belong to the same logical owner.
- **Ordering or signalling** — initialize to 0, so a waiter blocks until a different thread posts. This is where semaphores subsume condition variables for pure signalling, with no separate mutex needed.
- **Throttling** — initialize to N to cap concurrent workers.

**The mixing hazard:**

```
sem_wait(mutex)
sem_wait(full)        # blocks HERE while still holding mutex
get_item()
sem_post(mutex)
```

If `full` isn't ready this thread sleeps holding `mutex`, and the producer who would post to `full` needs `mutex` to proceed. Deadlock.

**Fix:** shrink the mutex to guard only the state mutation. Keep ordering waits outside it entirely.

**Find it:** flag any blocking wait issued while another lock is held, then check whether the thing being waited on can only be released by someone who needs the held lock.

## Lock implementation

If you're writing a lock rather than using one:

**Naive spinning is actively harmful.** A failed compare-and-exchange is still a *write* attempt, requiring exclusive ownership of the cache line — so every spinner repeatedly yanks the line away from the holder, who needs it to unlock. Under contention this can be slower than no lock at all, and catastrophic if the holder is preempted mid-section: every other core burns its full slice spinning on a lock that provably cannot be released.

**Spin on a plain read first**, attempting the compare-and-exchange only once the read suggests it might succeed. Reads share the line across cores without eviction, so idle spinners stop fighting the holder. Use a CPU pause hint during the spin.

**Only spin for genuinely short waits.** Spinning bets that blocking overhead exceeds wait time, and that bet loses badly once preemption is possible. For anything unbounded, block via an OS primitive keyed on the address — a futex, or `WaitOnAddress`.

The property a hand-rolled "check the flag then sleep" cannot achieve: futex-style waits are **atomic with respect to the wake**, because the kernel validates the expected value as part of going to sleep. A racing wake can never be silently missed.

**Language note.** Rust guards unlock via `Drop`, which runs during panic unwinding, so "forgot to unlock on the error path" is structurally impossible. Java, Go, C++, C#, and Swift require `try/finally`, `defer`, or an RAII scope guard explicitly — every time, including on rarely-hit error paths.
