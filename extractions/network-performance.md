# Extraction: network-performance

**Source:** High Performance Browser Networking — Ilya Grigorik

> Operational rules distilled from the source, written in our own words. This is the intermediate layer between the source text and the published skill — denser than the skill, and the place detail survives for re-distillation.

---

# Network Performance Operational Rules
*(client–server path: extracted from Ilya Grigorik's "High Performance Browser Networking")*

## 1. Latency dominates bandwidth for typical page loads

**Trigger:** Any request/response whose payload fits well inside the bandwidth-delay product — true for nearly all HTML, CSS, JS, and API payloads, which is most of the web.

**What it costs:** Time-to-complete for small transfers is governed by *number of sequential round trips × RTT*, not by payload size ÷ bandwidth. In a controlled study cited in the book, doubling bandwidth from 5 to 10 Mbps improved page load time by only ~5%, while every 20ms shaved off RTT produced a roughly linear improvement. This is a physical floor, not an engineering gap: light in fiber propagates at only ~200,000 km/s (refractive index ~1.5), giving irreducible RTT minimums — ~42ms NY–SF, ~56ms NY–London, 200–300ms NY–Sydney in practice. No amount of added bandwidth removes this floor.

**The fix:** Budget round trips, not kilobytes, when triaging a slow flow. Attack sequential blocking hops (handshakes, redirects, chained lookups, dependent API calls) before reaching for compression or minification.

**How to confirm:** In a waterfall, separate RTT-bound stages (DNS, TCP, TLS, redirect chains) from transfer-bound stages. If the RTT-bound sum dominates, a bandwidth upgrade won't move the number; only removing round trips will.

## 2. TCP connection setup and slow start

**Trigger:** Any new (non-reused) TCP connection.

**What it costs:** The three-way handshake (SYN → SYN-ACK → ACK) burns one full RTT before the first application byte can move. Once open, the connection is throttled by slow start regardless of true link capacity: the initial congestion window (cwnd) was historically 4 segments (RFC 2581) and is now standardized at 10 segments / ~14.6KB (RFC 6928, "IW10" — already the modern OS default, not a pending upgrade). cwnd roughly doubles every RTT until loss or a slow-start threshold is hit. Delivering a 64KB response from a cold IW10 connection over a 56ms-RTT link took 3 RTTs (168ms) in the book's worked example, versus 96ms on an already-warm connection — a 275% latency penalty purely from cold-start throttling, independent of bandwidth. General formula: RTTs-to-target ≈ log₂(target_segments ÷ initial_cwnd).

**The fix:** Keep connections warm rather than opening fresh ones; disable slow-start-restart-after-idle so a connection that goes quiet mid-session doesn't reset cwnd (`net.ipv4.tcp_slow_start_after_idle=0` on Linux); ensure window scaling (RFC 1323) is on so the receive window can exceed the unscaled 64KB ceiling — without it, bandwidth-delay-product math caps a 16KB window at ~1.31 Mbps over a 100ms-RTT path no matter how fast the link actually is.

**How to confirm:** `sysctl net.ipv4.tcp_slow_start_after_idle` and `net.ipv4.tcp_window_scaling`; a packet capture showing segment counts per RTT ramping from 10 upward rather than resetting after idle gaps, and in-flight byte counts approaching the computed bandwidth-delay product rather than plateauing at 64KB.

## 3. TCP head-of-line blocking is a transport-layer property, not fixable above the socket

**Trigger:** Any packet loss on a TCP connection carrying multiplexed application traffic — this is the crux of why HTTP/2, which multiplexes many logical streams onto one TCP connection, can still stall under loss.

**What it costs:** TCP's in-order delivery guarantee means the receiver withholds every correctly-received-but-out-of-order segment until the missing one is retransmitted and arrives, even though the application never sees this buffering directly — it shows up only as unpredictable jitter. Loss also halves cwnd under classic additive-increase/multiplicative-decrease congestion control, throttling the rest of the connection along with the stall.

**The fix:** Nothing at the application layer removes this — it's structural to TCP. **This is where the book's advice is dated**: HTTP/2 over TCP inherits this limitation completely, because TCP has no concept of independent streams. QUIC (HTTP/3's transport, standardized after this book) fixes it directly by giving each multiplexed stream independent loss recovery over UDP, so one lost packet stalls only its own stream instead of the whole connection. If a workload is loss-sensitive and multiplexed, treat HTTP/3 as the structural answer the book didn't have available.

**How to confirm:** Correlate measured packet loss on the path with simultaneous stalls across otherwise-unrelated streams in an HTTP/2 waterfall; compare against an HTTP/3 capture of the same flow on the same lossy link.

## 4. Connection reuse and keepalive

**Trigger:** An HTTP client, proxy, or server config that closes connections between requests, or application code that constructs a new HTTP client/agent per call.

**What it costs:** Without reuse, N sequential requests to the same origin pay ≈ (N−1) × RTT in pure repeated-handshake latency, stacked on top of repeated cold-start slow-start penalties each time. With ~90 requests being a typical page's baseline, this is seconds of avoidable latency, not milliseconds.

**The fix:** Verify keepalive is actually active (default in HTTP/1.1 and structurally the norm in HTTP/2), and that application code reuses a persistent connection pool/agent rather than instantiating a fresh HTTP client per call — a common accidental regression in service code.

**How to confirm:** Count distinct TCP handshakes versus distinct HTTP requests to the same origin in a packet capture or waterfall; in code, check that the HTTP client is configured with a shared, persistent agent.

**Related constraint — the browser's per-origin socket cap:** Browsers cap concurrent connections per origin at 6. Beyond that, additional requests queue client-side even when the server has spare capacity — this was the entire justification for domain sharding pre-HTTP/2 (see §6). Under HTTP/2/3, this ceases to matter because one connection multiplexes effectively unlimited concurrent streams. Confirm by checking for requests sitting "stalled/queued" in devtools despite available server capacity.

## 5. TLS handshake economics

**Trigger:** First connection to an HTTPS origin with no session-cache/ticket hit.

**What it costs (TLS 1.2 and earlier — DATED, see fix):** A full handshake adds 2 RTTs on top of TCP's 1 RTT — 3 RTTs total, ~168ms on the book's NY–London example, before any encrypted request byte moves.

**The fix:** TLS 1.3 (standardized 2018, after this book) collapses the full handshake to 1 RTT by default and supports 0-RTT resumption ("early data") for repeat connections — at the cost of replay exposure for non-idempotent requests sent in that early-data window, so gate 0-RTT to idempotent/safe requests only. This makes the book's TLS 1.2-era "False Start" workaround (start sending encrypted data before the handshake fully completes) largely moot — TLS 1.3 already gets you to 1-RTT without it. If anything in the delivery path still negotiates TLS 1.2, prioritize the 1.3 upgrade over other TLS tuning.

**How to confirm:** Negotiated TLS version (`openssl s_client -connect host:443`, or the browser devtools security panel); round-trip count before the first Application Data record appears in a packet capture.

**Session resumption:** avoids the full asymmetric-crypto handshake on repeat connections — a stateful server-side session cache (must be shared/sticky across a load-balanced fleet) or stateless session tickets (client-held, requiring periodic key rotation across the fleet). Confirm via the server/CDN's session-resumption hit-rate metric, or an abbreviated handshake (no full certificate exchange) visible on reconnect in a capture.

**ALPN:** folds protocol selection (HTTP/1.1 vs h2 vs h3) into the TLS handshake itself, avoiding a separate post-handshake `Upgrade` round trip that's also unreliable through intermediaries. Confirm ALPN is enabled server-side and intermediary-side by checking the extension list in ClientHello/ServerHello.

**OCSP stapling:** skipping it forces the client to query the CA's OCSP responder live mid-handshake — measured up to ~350ms added when it succeeds, and the Firefox telemetry cited in the book shows those live checks time out as much as 15% of the time, plus they leak the destination site to the CA. Fix: staple a signed OCSP response into the handshake server-side; confirm with `openssl s_client -status`.

**Certificate chain size:** an oversized chain (redundant intermediates, large keys) can push the ServerHello+Certificate flight past the ~10-segment initial congestion window, forcing an extra RTT of slow start before the handshake even finishes; a chain missing a required intermediate forces the client to pause and fetch it out-of-band (extra DNS + TCP + HTTP round trips). Fix: send only the necessary intermediates (never the root), minimize chain length, and if stapling OCSP (400B–4KB typically), keep the combined flight under the IW budget. Confirm by measuring total handshake-flight bytes against the ~14.6KB IW10 budget.

## 6. HTTP/1.x concurrency workarounds — and why they invert under HTTP/2+

These three techniques exist solely to route around HTTP/1.1's two limits: no request multiplexing on a single connection, and a 6-connection-per-origin cap. Each becomes questionable-to-actively-harmful once HTTP/2 removes both constraints.

**Domain sharding — Trigger:** serving assets from multiple subdomains purely to exceed 6 connections. **Cost:** every new hostname adds its own DNS lookup, its own TCP handshake, its own TLS handshake, and its own independent cold congestion window — the book flags this as hitting high-latency (mobile 3G/4G) clients hardest. **Fix:** under HTTP/1.1, shard sparingly and only after measuring benefit (rarely worth more than a few shards); under HTTP/2/3, stop sharding entirely — splitting traffic across origins forces multiple cold connections and multiple separate HPACK compression contexts instead of one warm, well-compressed connection, actively undoing H2's benefit. **Confirm:** count distinct origins serving first-party static assets; under H2, verify requests actually multiplex onto one connection ID rather than spreading across several.

**Concatenation/bundling — Trigger:** merging all JS/CSS into one file to amortize per-request overhead. **Cost:** any single-file change invalidates and forces re-download of the entire bundle; parsing/execution can't start until the whole bundle arrives; unused-on-this-page code still ships. **Fix:** under HTTP/1.1, keep bundles around 30–50KB compressed as a balance between overhead and incremental execution; under HTTP/2/3, prefer many small, independently-cacheable, fine-grained files — multiplexing removes the per-request tax that motivated bundling, so granular files improve cache-hit rates without the old cost. **Confirm:** measure bytes re-downloaded per deploy for a one-line change; check for unused-on-page code still shipped in the bundle.

**Spriting — Trigger:** combining small images into one sheet to cut request count. **Cost:** the browser decodes the *entire* sprite into an RGBA bitmap in memory regardless of the visible clipped region — an 800×600 sprite decodes to ~1.83MB (w×h×4 bytes) — disproportionately costly on memory-constrained mobile devices, and any single-icon update invalidates the whole sheet's cache. **Fix:** under HTTP/2/3, prefer individually-requested, individually-cacheable images (or icon fonts/SVG sets) — the request-count problem sprites solved no longer exists. **Confirm:** check decoded (not encoded) image memory in a devtools memory profiler on sprite-heavy pages, especially on mobile.

**Inlining (data URIs) — Trigger:** embedding small assets as base64 in HTML/CSS to skip a request. **Cost:** ~33% base64 byte overhead; the inlined resource can't be cached independently of its parent document, so it re-transfers on every page that inlines it and invalidates whenever the parent changes; some browsers cap total data-URI size. **Fix:** reserve inlining for genuinely tiny (~1–2KB) resources where request overhead would exceed the resource itself, and use it far less aggressively under H2/H3 where an extra request is nearly free. **Confirm:** search shipped CSS/HTML for `data:` URIs above a few KB; check for the same payload duplicated across multiple cached documents.

## 7. What HTTP/2 multiplexing actually fixes

**Trigger:** Multiple logical requests in flight over one HTTP/2 connection.

**What it fixes:** Binary framing interleaves independent request/response frames on a single TCP connection, so one slow response no longer blocks unrelated responses queued behind it — this *is* the HTTP/1.1 head-of-line-blocking fix, and it's the reason the §6 workarounds should be undone. It's also why HPACK header compression pays off: HTTP/1.x resends full plaintext headers every request (500–800 bytes typical, before cookies) with zero cross-request compression, while HPACK's shared dynamic index lets repeat headers shrink to near a byte per field once indexed — measured at 45–1142ms shaved off page load on a DSL-class link from header compression alone.

**What it doesn't fix (see §3):** TCP-level ordering underneath is untouched — a lost packet still stalls every multiplexed stream on that connection until retransmission, because TCP has no notion of H2 streams. Under normal loss this is outweighed by the compression/prioritization wins; on lossy links it can bite (relevant on mobile — see §8).

**Server push:** genuinely replaces the inlining/bundling tradeoff rather than reintroducing it — pushed resources stay individually cacheable, reusable across pages, and multiplexed/prioritized like any other stream, and critically the client can decline or cancel a push it doesn't want, unlike forced inlining. It's easy to misuse (pushing already-cached resources wastes bandwidth), so treat it as available, not default-on.

**How to confirm:** Single connection ID serving many requests in devtools; compare header bytes transferred per request between an H1.1 and H2 capture of the same page; check `PUSH_PROMISE` frames against whether pushed resources are actually consumed rather than cancelled.

## 8. Mobile radio economics

**Trigger:** Any network activity from an idle cellular radio.

**What it costs:** Cellular radios are a state machine, and *waking* the radio — not sending bytes — is the expensive part. Promoting from idle to a connected high-power state is a one-time control-plane latency tax paid before any bytes move: roughly under 100ms on LTE, 150–500ms on HSPA/3.5G, and 500–2500ms on older 3G stacks (some measured up to ~2s from full idle). After the last packet, the radio holds its high-power state for a carrier-configured inactivity "tail" (order of seconds) before demoting — so scattered small requests each restart the expensive tail instead of sharing one. The book's concrete illustration: analytics beacons accounted for 0.2% of bytes transferred but 46% of total power consumption, purely from repeated radio-tail costs. Total one-shot request budget (radio promotion + DNS + TCP + TLS) runs roughly 200–3500ms on 3G and 100–600ms on 4G before the request itself is even sent.

**The fix:** Batch and coalesce outbound/inbound activity into one burst while the radio is already awake, then let it go idle; defer non-critical requests until the radio is active for another reason anyway; treat periodic background polling as expensive by default and minimize it (push is generally more efficient than polling, but a high-frequency push stream can be just as costly — batch/aggregate notifications too). If keeping a long-lived connection alive (WebSocket/SSE), note most mobile carriers apply a 5–30 minute NAT idle timeout, so an application-level keepalive roughly every 5 minutes is needed or the carrier silently drops the connection.

**How to confirm:** Profile radio/network energy on a real device (Android battery/network profiler or carrier RRC state logs) around a suspected chatty feature; look for many small transfers each separated by more than the idle-demotion timer; verify a supposedly-persistent connection survives an idle period longer than the carrier NAT timeout without a keepalive.

## 9. Choosing a transport: polling vs. long-polling vs. SSE vs. WebSocket

**Short-interval XHR polling — Trigger:** client checks an endpoint on a fixed short interval. **Cost:** every poll is a standalone request/response paying full protocol overhead (~800 bytes) even when nothing changed — the book's aggregate example: 10,000 clients polling every 60s at ~850 bytes/request sums to >1 Mbps of pure overhead delivering zero new messages; shortening the interval multiplies this waste, lengthening it raises average message latency (up to half the interval). **Fix:** don't use fixed-interval short polling for latency-sensitive data — move to long-polling, SSE, or WebSocket. **Confirm:** measure the ratio of empty-response polls to total polls; a high ratio indicts both interval and transport choice.

**Long-polling — Trigger:** same need, but the server holds the request open until there's something to say instead of returning empty. **Cost/benefit:** cuts empty-response overhead and improves latency over short polling, but at high message-arrival rates can issue requests just as often as short polling would. **Fix:** treat as a fallback for infra that can't do SSE/WebSocket, not a primary design once those are viable. **Confirm:** compare actual request rate under real traffic to the short-polling baseline — if they converge, long-polling isn't buying anything.

**SSE vs. WebSocket — Trigger:** choosing a live-update transport where server-to-client push is the requirement. SSE is server-to-client only, UTF-8/text only, ~5 bytes overhead per message, runs over one ordinary HTTP connection, and the browser's EventSource API handles reconnection and resume (`Last-Event-ID`) automatically — it's the cheapest option when the client doesn't need to talk back on the same channel. WebSocket is full duplex and binary-capable, at ~6–14 bytes of per-message framing overhead (client-to-server frames must be masked, +4 bytes) — the right tool once the client must also send data or payloads are binary, but note it has its *own* head-of-line blocking: frames from different messages can't interleave on one socket (no per-message stream ID), so large messages should be split at the application level. WebSocket also needs WSS (TLS) in production, since plain-WS traffic is frequently mishandled — buffered, blindly upgraded, or misclassified — by intermediary proxies that don't understand the protocol.

**Fix:** default to SSE for pure server push; reach for WebSocket only when the client must send on the same channel or binary framing matters; use (long-)polling when updates are infrequent and latency-tolerant. Whichever is chosen, confirm the infra is actually configured for long-lived connections — proxy read/send timeouts and idle-tunnel timeouts default far below what these transports need and must be raised (the book's own examples raise them to roughly an hour).

**Confirm:** check directionality and payload-type requirements first; then verify proxy/load-balancer timeout config explicitly, since a silently-too-short timeout is the most common way these transports fail in production despite working in dev.

## 10. Master rule: eliminate round trips before compressing bytes

**Trigger:** Any performance investigation that defaults straight to "compress/minify/shrink the payload."

**What it costs to get this backwards:** Per §1, most transfers already fit inside a few segments/RTTs, so shaving bytes only helps if it removes a round trip (e.g., pulling a response under one congestion window or one MTU) — otherwise it optimizes bandwidth, a dimension that usually isn't the bottleneck.

**The fix:** Before compressing anything, count and cut the sequential (not parallel) round trips in the critical path — handshakes, redirects, chained calls (auth-then-fetch, resolve-then-connect, sequential dependent requests that could be combined or parallelized).

**How to confirm:** In a waterfall, a fix that genuinely removes one round trip shows up as a latency drop of ~1×RTT regardless of network speed — that fixed-size, speed-independent drop is the diagnostic signature of having fixed the right thing, as opposed to a proportional, bandwidth-dependent improvement from byte-level changes.
