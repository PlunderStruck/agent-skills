# Extraction: concurrency-and-durability

**Source:** Operating Systems: Three Easy Pieces — Arpaci-Dusseau

> Operational rules distilled from the source, written in our own words. This is the intermediate layer between the source text and the published skill — denser than the skill, and the place detail survives for re-distillation.

---

# OSTEP Extract: Concurrency Bugs and Crash Consistency

Source: Arpaci-Dusseau, *Operating Systems: Three Easy Pieces* — "Locks," "Condition Variables," "Semaphores," "Common Concurrency Problems," "Interlude: Files and Directories," "Crash Consistency: FSCK and Journaling."

---

## PART 1 — Concurrency Bugs

### The empirical split: deadlock is the minority problem

A study by Lu et al. examined every concurrency bug fixed in MySQL, Apache, Mozilla, and OpenOffice's histories — 105 bugs total. 74 were non-deadlock; only 31 were deadlocks. Of the non-deadlock bugs, 97% were **atomicity violations** or **order violations**. That ratio is the most useful fact here: the dominant real-world concurrency bug is a missing lock around a check-then-act sequence, or a missing wait for something to happen first — not an exotic multi-lock cycle.

**Trigger (atomicity violation):**
```
// Thread A
if resource.field != null:
    use(resource.field)
// Thread B
resource.field = null
```
No lock spans the gap between check and use.

**What goes wrong:** A passes the check, then loses the CPU before acting. B runs in that gap and invalidates what A just confirmed. A resumes and uses something no longer valid. The code assumed a group of accesses would be indivisible without ever enforcing it.

**What to do:** Hold one lock across the entire check-and-use region in every thread that touches it, not separately around the check and the use.

**How to verify:** Review for check-then-use on shared fields where the use isn't adjacent to the check. Insert an artificial delay between check and use under concurrent load — a real bug then reproduces reliably instead of rarely.

**Trigger (order violation):**
```
// Thread A (creator)
child = spawn(worker)
// Thread B (worker)
use(child.state)   // assumes A already finished setup
```
Nothing enforces "A's setup" happens-before "B's use" — B just assumes it.

**What goes wrong:** Under a different schedule, B runs before A finishes setup and touches state that doesn't exist yet.

**What to do:** Replace the assumption with an explicit condition variable or semaphore that B waits on and A signals once true. A bare flag is not sufficient (see lost wakeup, below).

**How to verify:** Force the consumer/worker thread to run first (delay the producer's setup) — this class of bug is invisible under "usual" scheduling and only appears once the less-common ordering is forced.

### Deadlock's four conditions, and the one worth attacking

Deadlock needs all four: **mutual exclusion**, **hold-and-wait**, **no preemption**, **circular wait**. Breaking any one prevents it, but they aren't equally practical:

- **Mutual exclusion** is usually why the lock exists; removing it means lock-free structures built on hardware atomics — a large undertaking for a few hot structures, not a general strategy.
- **Hold-and-wait** avoidance means acquiring every lock a routine needs atomically up front, which collapses concurrency and requires callers to know their full lock set in advance.
- **No preemption** can be approximated with try-lock-and-backoff, but this introduces livelock and requires unwinding partial work on failure.
- **Circular wait** is the practical target: enforce a single ordering on lock acquisition so a cycle can never form.

**What to do:** Default to lock ordering (below). Reserve try-lock/backoff for narrow spots where the full lock set is known locally and bounded retry is acceptable.

**How to verify:** Deadlock is a liveness bug — threads stop, they don't crash. Build a lock-order-graph checker that records which lock was acquired first whenever two are held simultaneously, and flags any pair seen in both orders. Test under heavier contention than production expects; low-contention testing easily misses the window.

### Lock ordering as the standard discipline — and why it erodes

**Trigger:**
```
function transfer(a: Account, b: Account):
    lock(a.mutex); lock(b.mutex); ...
```
Called as `transfer(X, Y)` by one thread and `transfer(Y, X)` by another, concurrently.

**What goes wrong:** Each call locks its first argument, then blocks on its second, which the other call already holds — a cycle invisible in `transfer`'s own source; it only exists across two call sites that never appear together.

**What to do:** Never order by argument position when arguments are interchangeable — order by an intrinsic property of the lock (e.g., lock whichever has the lower address/ID first, regardless of parameter position). For many distinct lock types, maintain an explicit documented order (row lock before index lock before page lock) and audit every multi-lock call site against it.

**Why it erodes at scale:** Ordering is convention, not compiler-enforced — one careless addition anywhere can violate it. Encapsulation actively fights it: a call into another module has no visibility into what that module locks or in what order, so "safe-looking" call chains can still close a cycle. And a new lock must be re-verified against every existing lock, not just the ones it appears to touch.

**How to verify:** Scan for functions acquiring more than one lock and check against the documented order. Runtime lock-order-graph checkers catch violations code review misses, since the two conflicting call sites are rarely read together.

### Condition variables: recheck in a loop, or lose the wakeup

**Trigger:**
```
lock(m)
if !predicate: wait(cond, m)
// proceeds assuming predicate now holds
unlock(m)
```

**What goes wrong:** Signaling means "something changed, recheck" — not a promise the predicate is still true once the woken thread actually runs. Between signal and resumption, another thread can slip in and consume/invalidate the condition (a second consumer grabs the one item a producer just made, before the woken consumer is scheduled). `if` proceeds on a now-false predicate. Some runtimes can also deliver spurious wakeups with no matching signal.

**What to do:** Always recheck in a `while`, never `if`:
```
lock(m)
while !predicate: wait(cond, m)
unlock(m)
```

**How to verify:** Flag any `wait()` not inside a `while`. Stress-test with more waiters than signalers (multiple consumers, one producer) under contention — a 1-to-1 test is far less likely to expose the `if` version's failure.

**Trigger (lost wakeup):**
```
// Waiter, no lock held
if !done: wait(cond)
// Signaler, no lock held
done = true; signal(cond)
```

**What goes wrong:** Without a shared lock serializing check-and-wait against update-and-signal, the signaler can run entirely between the waiter's check and its `wait()` call. The signal fires with nobody asleep yet; the waiter then sleeps forever.

**What to do:** Hold the same lock across check-and-wait and update-and-signal. Real `wait()` implementations require the lock as a parameter for this reason — it atomically releases the lock and sleeps. Keep an explicit state variable so a waiter's own check can observe an already-true state without needing the signal at all.

**How to verify:** Any wait/signal pair not guarded by the same lock is a defect on inspection. Force the signaling thread to complete before the waiter reaches `wait()` — correct code is invariant to this; broken code hangs.

### Signal vs. broadcast: knowing who to wake

**Trigger:** One condition variable shared by waiters waiting for logically different things — producers and consumers on one `cond`, or several differently-sized allocation requests waiting for "some memory freed."

**What goes wrong:** `signal()` wakes one waiter, arbitrarily chosen. If waiters want different things, the runtime can wake the wrong one, which rechecks, finds its own predicate still false, and sleeps again — the wakeup that should have gone elsewhere is now gone. In the worst case every thread ends up asleep.

**What to do:** When waiters fall into distinguishable classes, use one condition variable per class (separate `empty`/`full` for producers/consumers). When they can't be separated, use `broadcast()`: wake everyone, let each recheck its own `while` predicate, and let those still blocked go back to sleep. This "covering condition" costs extra spurious wakeups but never silently loses one.

**How to verify:** If switching `signal()` to `broadcast()` is the only way to fix a failing test, investigate why the wrong thread was targeted before accepting broadcast as the answer. Multi-class waiter tests expose this; single-class tests don't.

### Semaphores vs. mutexes: pick by what you're modeling

A semaphore is an integer plus `wait()` (decrement, block if negative) and `post()` (increment, wake one waiter). A mutex is the special case: a binary semaphore initialized to 1. Rule of thumb: **initialize to the number of "resources" you're willing to give away immediately.**

**Mutex use** — acquire and release belong to the same logical owner:
```
sem = init(1); sem.wait(); critical_section(); sem.post()
```

**Ordering/signaling use** — producer and consumer are different threads, initialize to 0 (or N for throttling N concurrent workers):
```
sem = init(0)
// waiter: sem.wait()     // consumer
// producer: produce(); sem.post()
```
This is where semaphores subsume condition variables — no separate mutex+CV pair needed for pure signaling.

**What goes wrong when mixed carelessly:** Guarding a bounded buffer with one mutex around the whole put/get *and* blocking on an ordering semaphore inside it:
```
sem_wait(mutex)
sem_wait(full)   // may block here, still holding mutex!
get_item(); sem_post(mutex)
```
If `full` isn't ready, this thread blocks while holding `mutex`. A producer that needs `mutex` to post to `full` is now stuck too — deadlock.

**What to do:** Shrink the mutex's scope to guard only the actual state mutation; keep ordering waits (`empty`/`full`) outside it entirely.

**How to verify:** Flag any blocking wait issued while another lock/semaphore is already held; trace whether the thing being waited on can only be released by someone who needs the held one. Lock-order-graph tooling catches this the same way it catches ordinary lock deadlock.

### Why concurrency bugs are hard to reproduce, and what finds them

**The mechanism:** A race's outcome depends on exactly where a context switch lands relative to a handful of unprotected accesses. The space of possible interleavings across even a few threads is enormous, and buggy ones are a small fraction of it. A thousand passing runs haven't proven correctness — they've sampled a thousand mostly-safe points. Production then samples differently (more cores, different load, different memory pressure), so bugs invisible in CI appear in the field, and field bugs resist reproduction on a laptop.

**What finds it:**
- **Increase contention deliberately** — more threads than cores, sustained load — to raise the odds of hitting the bad interleaving.
- **Inject scheduling perturbation** at the exact gaps the atomicity/order patterns identify — a forced delay between check and use, or before a producer's setup — turns a one-in-a-million bug into a reliable one.
- **Force deadlock outright**: given the lock orders two paths use, drive both concurrently and synchronize them at the exact cycle-forming point instead of waiting for luck.
- **Race/deadlock detectors** — instrumentation tracking accesses against a happens-before relation, and lock-order-graph tools flagging any pair acquired in both orders — reason over the space of orderings instead of needing to sample the bad one.
- **Targeted code review** for exactly the two patterns above is disproportionately effective, since they account for 97% of non-deadlock bugs.

---

## PART 2 — Crash Consistency and Durability

### What `fsync()` guarantees — and the directory gap

**Trigger:**
```
fd = open("data.txt", CREATE|WRITE)
write(fd, buffer, size)
fsync(fd)
close(fd)
```

**What goes wrong:** `write()` only hands data to the OS's in-memory page cache; the OS defers the actual device write for seconds. `fsync(fd)` forces every dirty block of that file to disk and blocks until complete — but if `data.txt` was newly *created*, the directory entry linking the name to the inode is a separate piece of metadata, buffered the same way, and this `fsync` says nothing about it. A crash right after can leave the file's data durable with no directory entry pointing at it — after reboot the file may not exist at all.

**What to do:** After creating, renaming, linking, or unlinking anything, open the containing directory and `fsync()` its descriptor too, separately from the file's own fsync.

**How to verify:** Crash/power-fail injection: kill the process at various points relative to each fsync call, restart, check whether the file is visible and correct. A single non-crashing run proves nothing — the bug only matters across an actual crash. `strace` review as a cheap first pass: confirm a directory-fd `fsync` exists alongside every file-creation path's own fsync.

### Write ordering is not preserved by default

**What goes wrong:** Disks with write buffering ("immediate reporting") report completion once data lands in the disk's own cache, not once it's physically on the medium. Even without that, nothing guarantees write A completes before write B on stable storage just because A was issued first — the device can reorder internally unless an explicit write barrier forbids it, and historically some firmware has silently ignored barriers for benchmark speed.

**Trigger — naive "write temp, then rename":**
```
fd = open("data.txt.tmp", CREATE|WRITE)
write(fd, new_contents, size)
close(fd)
rename("data.txt.tmp", "data.txt")
```

**What goes wrong:** `write()` and `rename()` touch independent on-disk state (temp file's data blocks vs. the directory entry) with no ordering enforced between them. The rename's directory update can become durable before the temp file's data does. A crash in that window leaves `data.txt` correctly resolving to the new inode (rename's own atomicity holds) — but that inode's data was never flushed, so the "new" file is garbage or truncated, and the previously-good old content is gone.

**What to do:** `write()`, then `fsync()`, then `close()`, then `rename()`. The fsync is what stands between "rename" and "rename over corrupt data."

**How to verify:** Power-fail injection with kill points between `close()` and `rename()`. Confirm via `strace` that `fsync()` actually precedes `rename()` — a common regression is dropping the fsync as an "optimization" that only crash-injection tests, not ordinary tests, will catch.

### The atomic-rename idiom: what it buys, and what it doesn't

**What it guarantees:** `rename()` is atomic with respect to crashes for the pointer swap itself — after any crash, the destination name resolves entirely to the old target or entirely to the new one, never partial or missing.

**What it doesn't guarantee:**
- Nothing about whether the new file's contents were durable at rename time — that's the fsync-before-rename step, above. Rename protects the pointer, not the data behind it.
- Nothing about the rename itself surviving a crash right after — its directory entry is buffered like any other metadata; if the rename must survive an immediate crash, the directory needs its own fsync afterward too.
- No atomicity across multiple files or renames. Two separate `rename()` calls are two independent operations, not one pair; a crash between them can leave one done and one not. Rename cannot express "these must change together" — that needs a real journal.

**What to do:** Treat rename as a single-pointer atomic swap, nothing more. Pair it with fsync-before (content durability) and fsync-after-on-the-directory (rename durability) when both matter; build an explicit WAL when multiple objects must change atomically together.

**How to verify:** Review any code performing more than one rename, or a rename alongside another independent write, for a silent assumption that they happen together — crash injection between the two operations will expose the gap.

### Journaling: why log write, commit, and checkpoint order matters

The problem: updating several on-disk structures for one logical change (an index node, an allocation bitmap, a data block) needs multiple writes, and a crash between any two leaves them mutually inconsistent. Write-ahead logging fixes this by writing the intended change to a separate log first, then applying ("checkpointing") it to its real location — so recovery can consult the log instead of inferring state from scratch.

**Trigger — an application-level WAL, not relying on the filesystem's own journal:**
```
append_to_log(begin, payload); fsync(log)
append_to_log(commit_marker); fsync(log)
apply_payload_to_primary_storage()   // checkpoint
mark_log_entry_reclaimable()
```

**What goes wrong if order is violated:**
- **Checkpoint before the log entry is durably committed:** a crash mid-checkpoint leaves primary storage partially updated with no log record to redo from — the exact inconsistency journaling exists to prevent.
- **Commit marker batched with the payload:** if a single big write covers payload plus commit marker, the device may complete pieces out of order — begin, most of the payload, and the commit marker land, but one payload block doesn't. The log now looks like a valid committed transaction (matching begin/commit) but contains garbage. Recovery, trusting the marker, replays that garbage into the real location — actively corrupting data a naive recovery would otherwise have left alone.

**What to do:** Keep phases strictly ordered, each fully durable before the next: (1) write payload, wait for completion; (2) write a *distinct* commit marker, wait for that separately — never batched with the payload; (3) only then checkpoint to the real location. To avoid the extra wait between (1) and (2), compute a checksum over the whole transaction and store it in the begin/commit blocks; recovery validates the checksum and discards anything that doesn't match, catching torn or reordered writes without a forced wait.

**How to verify:** Crash injection at each boundary: mid-payload (must discard entirely), payload-done-but-commit-not-durable (must discard), commit-durable-but-not-checkpointed (must redo from log), mid-checkpoint (redo must be idempotent). A recovery routine untested at all four points hasn't actually been tested.

### Torn writes and partial-sector writes

**Trigger:** Assuming a multi-sector write is applied atomically — that a crash mid-write leaves it either fully old or fully new.

**What goes wrong:** Devices generally guarantee atomicity only at their native write granularity (historically a 512-byte sector; larger on 4Kn/SSD devices), not for an arbitrary larger write spanning several units. A power failure partway through can leave some sectors updated and others not (or an arbitrary subset, since the device may reorder pieces internally) — a torn write that is neither the old nor the new version. This is exactly why journal commit markers are deliberately kept to a single atomically-writable block.

**What to do:** Never assume a write larger than the device's atomic unit is all-or-nothing. Keep anything that must be atomic within that unit, or add a checksum spanning a larger protected region and treat a mismatch as proof of tearing — discard and fall back to the last known-good version.

**How to verify:** Fault injection that truncates a simulated write at arbitrary byte offsets (not just block boundaries), confirming the checksum path flags every truncation point as corrupt, including ones mid-sector.

### Why a successful `write()` does not mean the data is on disk

**Trigger:** Code that treats a successful `write()` (or `close()`) as proof data will survive a crash, with no `fsync()` anywhere on that path.

**What goes wrong:** `write()`'s success only means the OS accepted the bytes into its page cache — nothing about when, or whether, they reach the device. The OS deliberately batches actual device writes for throughput, creating a real window (on the order of seconds) where a crash silently loses data the application already believes is saved, with no error ever returned, because the call that would have surfaced the problem already returned successfully.

**What to do:** For any write whose durability matters across a crash, call `fsync()` and check its return value — a failed fsync means the data is not durable and must be treated as a write failure. This compounds with the earlier points: a new file also needs its directory fsynced, and a rename needs fsync before (and, if it must itself survive a crash, after).

**How to verify:** Write recognizable data, kill the process (or VM) immediately without fsync, restart, check whether it survived — this should sometimes fail, proving the buffering window is real. Repeat with fsync inserted before the kill — this should never fail. Automate as a scripted power-fail test in CI; the loss window is timing-dependent and won't reproduce on every manual attempt.
