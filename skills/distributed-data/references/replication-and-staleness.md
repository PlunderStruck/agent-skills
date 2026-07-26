# Replication and Staleness

Applies when reads can be served by anything other than the node that accepted the write: a read replica, a cache, a CDN, a search index, a follower in another region.

## The setup that causes this

Read-heavy workloads scale by adding followers and spreading reads across them. That only works with **asynchronous** replication — synchronous replication to every follower means one slow node blocks all writes. So the default configuration of nearly every production database is: writes go to the leader, reads may go to a follower, and the follower may be behind.

Normal lag is sub-second. Lag under load, during a follower rebuild, or across a congested link is seconds to minutes. There is no upper bound. "Eventually consistent" is deliberately vague about *when*.

## Three anomalies, three different fixes

### Read-your-writes

The user submits something and immediately views it. The read hits a follower that doesn't have it yet. To the user, their data vanished.

This is the one that generates support tickets, and it's the one agents most reliably miss, because in development there's one database node and the bug cannot occur.

Fixes, in order of how much they cost:

- **Route by ownership.** Read the user's own editable data from the leader; read everyone else's from a follower. Works when you can tell statically what a user can modify (their own profile, their own orders).
- **Route by recency.** Track the timestamp of the user's last write; send all their reads to the leader for the next N seconds. Cruder, but works when most things are user-editable.
- **Gate on write position.** Client remembers the log position (LSN / binlog coordinate) of its last write; the read either waits for a replica to reach that position or falls back to the leader. Most precise, most plumbing.

Two complications worth checking for:

- **Cross-device.** Same user on phone and laptop. Device-local "last write timestamp" doesn't transfer. The metadata has to be centralized, and requests from both devices may need pinning to the same region.
- **Monitoring.** If replicas can exceed a lag threshold, reads should be pulled from them automatically. This requires the lag metric to actually be wired to routing, not just to a dashboard.

### Monotonic reads

The user refreshes and sees data go *backward* — a comment appears, then disappears. Caused by consecutive reads landing on replicas with different lag.

Fix: pin each user to one replica, chosen by hash of user ID rather than round-robin. Needs a fallback path when that replica dies.

Note this is strictly weaker than read-your-writes and strictly stronger than eventual consistency. You can need one without the other.

### Consistent prefix

A reader sees an effect before its cause — the reply before the question, the update before the insert. Happens when causally related writes live on different partitions that replicate independently, so there's no global write order.

Fix: route causally related writes to the same partition, or track causal dependencies explicitly. Note this is a *partitioning* problem wearing a replication costume.

## What to check in the code

- Does any request path write and then read the same entity? Trace whether the read can reach a follower.
- Does a test suite that runs against a single node give false confidence here? (Yes. Always.) Consider a test harness that deliberately serves reads from a lagged replica.
- Is a cache read being treated as equivalent to a database read? A cache is a replica with unbounded, unmonitored lag.
- Does the code assume a just-written row is immediately visible to a background job, a webhook, or another service? That's a read-your-writes assumption across a process boundary.

## Leaderless / quorum stores

If you're on Dynamo-style storage (Cassandra, Riak, Voldemort), quorum reads and writes with `w + r > n` sound like they guarantee fresh reads. They do not, in these cases:

- **Sloppy quorums**: writes accepted by nodes outside the key's home set, so the read set and write set may not overlap at all. On by default in Riak; off by default in Cassandra and Voldemort — check your config.
- **Concurrent writes**: no defined order, so "latest" is undefined. If the resolution is last-write-wins by timestamp, writes are lost to clock skew.
- **Partially failed writes**: a write that succeeded on fewer than `w` replicas is reported as failed but is *not* rolled back. Later reads may or may not see it.
- **Restored replicas**: a node restored from an old backup can drop the count of replicas holding the new value below `w`.

Critically: quorums give you none of read-your-writes, monotonic reads, or consistent prefix. If you need those, you need them at the application layer or you need a different store.

## Failover data loss

With asynchronous replication, promoting a follower discards any writes the old leader hadn't yet shipped. Two consequences that bite:

- A write acknowledged to the client can vanish. Durability was not what the acknowledgement implied.
- If another system keyed off that data — a cache, a search index, an external service holding the ID — it now references rows that no longer exist. GitHub had exactly this outage: a promoted stale replica reused autoincrement primary keys that Redis was already referencing, and served private data to the wrong users.

If your code stores IDs generated by one system inside another system, ask what happens to that reference after a failover.
