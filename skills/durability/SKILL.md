---
name: durability
description: Make data actually survive a crash — what fsync guarantees and what it does not, why a successful write() means nothing, the directory entry everyone forgets, safe write-temp-then-rename, torn writes, and write-ahead log ordering. Use when writing files that must persist, implementing a save or export path, building a write-ahead log or crash-recovery routine, or reviewing code that assumes data is on disk.
---

# Durability

## Purpose

**A successful `write()` does not mean the data is on disk.** It means the operating system accepted the bytes into its page cache and will get to the device eventually — typically within seconds, but with no guarantee and no error returned if a crash intervenes.

That gap is where data loss lives, and it's invisible in testing because a process that exits normally flushes everything. You only find it by crashing.

This skill is short because the rules are few. They're just consistently skipped.

## When this applies

- Writing a file whose contents must survive a crash or power loss
- Any save, export, checkpoint, or "commit to disk" path
- Implementing a write-ahead log or a crash-recovery routine
- The write-temp-then-rename idiom
- Reviewing code that treats a completed write as persisted

## The rules

**1. `write()` returning success means nothing about durability.** For anything that must survive a crash, call `fsync()` and **check its return value** — a failed fsync means the data is not durable and must be handled as a write failure, not ignored.

**2. Creating a file needs a second fsync, on the directory.** `fsync(fd)` forces that file's data to disk. It says nothing about the **directory entry** linking the name to the inode, which is separate metadata, buffered separately. A crash right after can leave the data durable with no name pointing at it — after reboot the file may simply not exist.

So after creating, renaming, linking, or unlinking: open the containing directory and `fsync` that descriptor too.

**3. Write ordering is not preserved.** Nothing guarantees write A reaches stable storage before write B just because you issued A first. Devices reorder internally, and many report completion once data lands in the drive's own cache rather than on the medium.

**4. The naive write-temp-then-rename is broken.**

```
write(tmp, contents)
close(tmp)
rename(tmp, target)        # BROKEN
```

The temp file's data blocks and the directory entry are independent on-disk state with no enforced ordering. The rename can become durable *before* the data does — leaving `target` correctly resolving to an inode whose contents were never flushed. The new file is garbage or truncated, and the old good content is gone.

The correct sequence:

```
write(tmp, contents)
fsync(tmp)                 # <- the step that matters
close(tmp)
rename(tmp, target)
fsync(directory)           # if the rename itself must survive an immediate crash
```

**5. Rename is atomic for the pointer, and nothing else.** After any crash the destination name resolves entirely to the old target or entirely to the new one — never a partial state. That's genuinely useful, and it is the *only* thing rename guarantees.

It says nothing about whether the new contents were durable (rule 4), nothing about the rename surviving an immediate crash (rule 2), and **nothing across multiple files.** Two renames are two independent operations; a crash between them leaves one applied and one not. Rename cannot express "these change together."

**6. Writes larger than the device's atomic unit can tear.** Devices generally guarantee atomicity only at their native write granularity — historically a 512-byte sector, larger on modern drives. A power failure partway through a larger write can leave some units updated and others not, in an arbitrary order, producing a state that is neither the old version nor the new one.

Keep anything that must be atomic within one unit, or add a checksum over the protected region and treat a mismatch as proof of tearing.

## Write-ahead logging

When several pieces of state must change together, no single-file trick suffices — you need a log.

```
append(log, begin + payload);  fsync(log)
append(log, commit_marker);    fsync(log)     # separate write, separate fsync
apply_to_primary_storage()                    # checkpoint
mark_log_reclaimable()
```

**The ordering is the whole mechanism.** Two ways to get it wrong:

- **Checkpointing before the log entry is durably committed.** A crash mid-checkpoint leaves primary storage partially updated with no log record to redo from — exactly the inconsistency the log exists to prevent.
- **Batching the commit marker with the payload.** If one large write covers both, the device may complete pieces out of order: `begin`, most of the payload, and the commit marker land, but one payload block doesn't. The log now *looks* like a valid committed transaction and recovery replays garbage into real storage — actively corrupting data that a naive recovery would have left alone.

To avoid the cost of a second forced wait, compute a checksum over the whole transaction and store it in the begin and commit records. Recovery validates the checksum and discards anything that doesn't match, catching torn and reordered writes without the extra `fsync` barrier.

**Recovery must be idempotent**, because a crash during recovery means it runs again.

## Verifying any of this

Ordinary tests prove nothing here — a process that exits cleanly flushes everything. You need **crash injection**:

- Kill the process (or the VM) at various points relative to each fsync, restart, and check what survived.
- Write recognizable data and kill *without* fsync. This should **sometimes** lose data — if it never does, your test isn't reaching the buffering window and proves nothing.
- Repeat with fsync inserted. This should never lose data.
- For a write-ahead log, inject a crash at all four boundaries: mid-payload (discard), payload durable but commit not (discard), commit durable but not checkpointed (redo), mid-checkpoint (redo again, proving idempotence). A recovery routine untested at all four has not been tested.
- For tearing, truncate a simulated write at arbitrary byte offsets — not just block boundaries — and confirm the checksum path flags every one.

As a cheap first pass, `strace` the save path and confirm the `fsync` calls are where you think they are. A common regression is someone removing one as an "optimization," which no ordinary test will catch.

## Boundary with other skills

- **`resilience`** — surviving a dependency failing or hanging.
- **`distributed-data`** — correctness across replicas and stores, including why an acknowledged write can still be lost on failover.
- **`durability`** — whether the bytes are actually on this machine's disk.

A replicated system still needs this: replication protects against losing a *machine*, not against every node having buffered the same write and none having flushed it.
