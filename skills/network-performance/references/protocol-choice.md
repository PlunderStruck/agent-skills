# HTTP Version and Delivery Strategy

## What each version constrains

**HTTP/1.1** has two hard limits: no request multiplexing on a connection, and a browser cap of six connections per origin. A slow response blocks everything queued behind it on that connection. Headers are re-sent in full plaintext every request — commonly 500–800 bytes before cookies — with no cross-request compression.

**HTTP/2** fixes both. Binary framing interleaves independent request and response frames on one connection, so a slow response no longer blocks unrelated ones. HPACK gives headers a shared dynamic index, so repeated fields shrink toward a byte each after the first request.

**HTTP/3** fixes the thing HTTP/2 could not.

## The limitation HTTP/2 inherits

HTTP/2 multiplexes logical streams over a single TCP connection — and TCP has no concept of those streams. TCP guarantees in-order delivery, so a single lost packet makes the receiver withhold every correctly-received-but-later segment until the retransmission arrives. Every multiplexed stream stalls, not just the one that lost a packet. Loss also halves the congestion window, throttling everything on the connection.

Nothing at the application layer fixes this. It is structural to TCP.

**HTTP/3 over QUIC solves it directly** by giving each stream independent loss recovery over UDP, so a lost packet stalls only its own stream. If your workload is multiplexed and runs over lossy links — mobile especially — this is the structural answer, and it did not exist when most HTTP/2 migration advice was written.

Under normal low-loss conditions, HTTP/2's compression and prioritization wins outweigh the head-of-line risk. Under loss, they don't.

## Optimizations that invert

These four exist purely to work around HTTP/1.1's limits. Under HTTP/2 and HTTP/3 each ranges from pointless to actively harmful. **Auditing for these is usually the highest-value thing in this reference**, because they're baked into build pipelines and nobody revisits them.

### Domain sharding

*Was:* serve assets from multiple subdomains to exceed the six-connection cap.

*Now:* every extra hostname costs its own DNS lookup, TCP handshake, TLS handshake, and independent cold congestion window. Under HTTP/2 it also fragments what should be one warm, well-compressed connection into several cold ones with separate HPACK contexts — directly undoing the benefit you migrated for. Worst on high-latency mobile links.

**Do:** stop sharding entirely under HTTP/2+. Under HTTP/1.1, shard sparingly and only after measuring.

### Concatenation and bundling

*Was:* merge all JS and CSS into one file to amortize per-request overhead.

*Now:* any one-line change invalidates the whole bundle and forces a full re-download. Parsing and execution can't start until everything arrives. Code unused on this page still ships.

**Do:** under HTTP/2+, prefer many small, independently cacheable files — multiplexing removed the per-request tax that justified bundling, and granular files improve cache hit rates. Under HTTP/1.1, roughly 30–50 KB compressed per bundle balances overhead against incremental execution.

### Spriting

*Was:* combine small images into one sheet to cut request count.

*Now:* the browser decodes the *entire* sheet into an RGBA bitmap regardless of the visible region. An 800×600 sprite becomes roughly 1.8 MB in memory (width × height × 4 bytes), which is disproportionately expensive on memory-constrained mobile devices. Updating one icon invalidates the whole sheet.

**Do:** individually requested, individually cacheable images — or an icon font or SVG set.

**Confirm:** decoded (not encoded) image memory in a memory profiler, on mobile.

### Inlining as data URIs

*Was:* embed small assets in HTML or CSS to skip a request.

*Now:* base64 adds roughly 33% byte overhead, and the inlined resource cannot be cached independently of its parent — it re-transfers on every page that inlines it, and invalidates whenever the parent document changes.

**Do:** reserve for genuinely tiny resources (1–2 KB) where request overhead would exceed the resource. Use far less aggressively under HTTP/2+.

**Confirm:** search shipped CSS and HTML for `data:` URIs above a few KB, and for the same payload duplicated across documents.

## Server push

Genuinely replaces the inlining tradeoff rather than recreating it: pushed resources stay individually cacheable, reusable across pages, and prioritized like any other stream — and the client can decline or cancel a push it doesn't want, which inlining never allowed.

It's easy to misuse. Pushing something already in the client's cache wastes bandwidth. Treat it as available, not default-on.

**Confirm:** check `PUSH_PROMISE` frames against whether pushed resources were consumed or cancelled.

## Migration checklist

When moving to HTTP/2 or HTTP/3, the protocol change is the easy half. The audit is:

- Remove domain sharding; consolidate onto one origin.
- Break bundles into granular, independently cacheable files.
- Replace sprites with individual assets, icon fonts, or SVG.
- Strip non-trivial data URIs back out into real requests.
- Verify ALPN negotiates the version you think it does.
- Verify requests actually multiplex onto one connection rather than spreading.

Migrating the protocol while keeping the workarounds is common, and gives up most of the benefit while paying the migration cost.
