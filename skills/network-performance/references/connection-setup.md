# Connection Setup

Where the round trips actually go before your first byte of application data moves.

## TCP handshake and slow start

Opening a connection costs one round trip for the three-way handshake before any application byte can travel.

Once open, throughput is governed by congestion control rather than link capacity. The initial congestion window is standardized at 10 segments — roughly 14.6 KB — and roughly doubles each round trip until loss or a threshold. This is already the modern default, not a pending upgrade.

Practical consequence: a 64 KB response on a cold connection needs several round trips purely from window growth. On a 56 ms path that's the difference between roughly 170 ms and roughly 96 ms for identical bytes over an identical link. The penalty is cold-start throttling, and no bandwidth buys it back.

Rough arithmetic for how many round trips a transfer needs: log₂(target segments ÷ initial window).

**Settings worth verifying:**

- **Disable slow-start-restart after idle.** A connection that goes quiet mid-session otherwise resets its window and pays the ramp again. On Linux: `net.ipv4.tcp_slow_start_after_idle=0`.
- **Confirm window scaling is enabled.** Without it the receive window caps at 64 KB, which throttles a 100 ms path to roughly 1.3 Mbps regardless of actual link speed. This is bandwidth-delay-product arithmetic, not a tuning preference.

## Connection reuse

Without reuse, N sequential requests to one origin pay roughly (N−1) extra round trips in handshakes alone, plus a cold window each time. For a page making dozens of requests this is seconds, not milliseconds.

Keepalive is the default in HTTP/1.1 and structural in HTTP/2, so the usual failure is in application code: constructing a new HTTP client, agent, or session per call rather than sharing a pooled one. This is easy to introduce accidentally and invisible in code review.

**Confirm:** count distinct TCP handshakes against distinct HTTP requests to the same origin in a capture. They should diverge sharply.

## The per-origin connection cap

Browsers cap concurrent connections per origin at six. Past that, requests queue client-side even when the server has capacity to spare.

This limit is the entire justification for domain sharding — and it disappears under HTTP/2 and HTTP/3, where one connection multiplexes effectively unlimited concurrent streams. See [protocol-choice](protocol-choice.md).

**Confirm:** requests sitting in a stalled or queued state in devtools while the server is idle.

## TLS handshake cost

**TLS 1.2 and earlier:** two round trips on top of TCP's one — three total before an encrypted request byte moves.

**TLS 1.3:** one round trip by default, and zero-RTT resumption for repeat connections.

Zero-RTT carries a caveat: early data is replayable, so restrict it to idempotent requests. Sending a non-idempotent operation in the zero-RTT window invites duplicate execution.

TLS 1.3 also makes the older False Start workaround largely irrelevant — it reaches one round trip without it. **If any part of the path still negotiates TLS 1.2, that upgrade outranks every other TLS optimization here.**

**Confirm:** `openssl s_client -connect host:443` for the negotiated version; count round trips before the first Application Data record in a capture.

## Session resumption

Avoids the full asymmetric handshake on reconnect. Two mechanisms:

- **Server-side session cache** — stateful, so it must be shared or sticky across a load-balanced fleet. A cache that only works when you hit the same instance is a cache that mostly misses.
- **Session tickets** — stateless, client-held, requiring periodic key rotation across the fleet.

**Confirm:** the resumption hit-rate metric, or an abbreviated handshake with no full certificate exchange on reconnect.

## ALPN

Folds protocol negotiation — HTTP/1.1 versus h2 versus h3 — into the TLS handshake itself, avoiding a separate `Upgrade` round trip that intermediaries handle unreliably anyway.

**Confirm:** the extension list in ClientHello and ServerHello.

## OCSP stapling

Without it, the client queries the CA's responder live, mid-handshake. That adds up to several hundred milliseconds when it works, times out a meaningful fraction of the time, and leaks the destination to the CA.

Staple a signed response server-side.

**Confirm:** `openssl s_client -status`.

## Certificate chain size

An oversized chain — redundant intermediates, large keys — can push the certificate flight past the initial congestion window, forcing an extra round trip of slow start *before the handshake even completes*.

A chain *missing* a required intermediate is worse: the client pauses to fetch it out of band, paying DNS, TCP, and HTTP round trips mid-handshake.

Send exactly the necessary intermediates, never the root. If stapling OCSP (typically 400 B–4 KB), keep the combined flight inside the initial window budget.

**Confirm:** measure total handshake flight bytes against roughly 14.6 KB.

## What to check

- Is a persistent connection pool actually shared, or is a client constructed per call?
- Negotiated TLS version — 1.3 or still 1.2?
- Is zero-RTT restricted to idempotent requests?
- Is session resumption shared across the fleet, or per-instance?
- Is OCSP stapled?
- Does the certificate flight fit inside the initial window?
- Is slow-start-after-idle disabled on long-lived connections?
