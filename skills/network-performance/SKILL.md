---
name: network-performance
description: Make the network between server and client fast — cutting round trips rather than bytes, connection reuse and TLS handshake cost, choosing between HTTP/1.1, HTTP/2 and HTTP/3, picking polling versus SSE versus WebSocket, and why mobile radios make small frequent requests disproportionately expensive. Use when optimizing page load or API latency, configuring an HTTP client or server, choosing a realtime transport, or diagnosing why something is slow despite adequate bandwidth.
---

# Network Performance

## Purpose

Most network optimization effort goes into making payloads smaller, and most network latency has nothing to do with payload size.

For anything that fits within a few packets — which is nearly all HTML, CSS, JS, and API responses — completion time is governed by **the number of sequential round trips multiplied by round-trip time**, not by bytes divided by bandwidth. Doubling bandwidth from 5 to 10 Mbps moves typical page load by only a few percent. Cutting round trips moves it linearly.

That single asymmetry drives everything below.

## The master rule

**Eliminate round trips before compressing bytes.**

Shaving bytes helps only when it *removes a round trip* — pulling a response inside one congestion window, or under one MTU. Otherwise it optimizes a dimension that wasn't the bottleneck.

Diagnostic signature: a fix that genuinely removed a round trip shows up as a latency drop of roughly one RTT **regardless of connection speed**. A fixed, speed-independent improvement means you fixed the right thing. A proportional, bandwidth-dependent one means you shaved bytes.

There's a physical floor here that no engineering removes: light in fiber travels around 200,000 km/s, which puts irreducible round trips at roughly 42 ms New York–San Francisco, 56 ms New York–London, and 200–300 ms New York–Sydney in practice.

## Rules that apply without loading anything

**1. Count sequential round trips in the critical path first.** Handshakes, redirects, chained dependent calls (authenticate, then fetch), DNS. Parallel requests don't add; sequential ones do.

**2. Reuse connections.** A new connection costs a TCP handshake round trip plus a cold congestion window. Application code that constructs a fresh HTTP client per call is a common and invisible regression — use a shared persistent pool.

**3. A new connection is slow even on a fast link.** Congestion control starts around 10 segments (~14.6 KB) and roughly doubles per round trip. A 64 KB response on a cold connection takes several round trips that a warm connection wouldn't need.

**4. Under HTTP/2 and HTTP/3, undo the HTTP/1.1 workarounds.** Domain sharding, aggressive bundling, spriting, and inlining were all workarounds for limits that no longer exist — and each is actively harmful now. See [protocol-choice](references/protocol-choice.md).

**5. TLS version determines handshake cost.** TLS 1.2 costs two round trips on top of TCP's one. TLS 1.3 costs one, with zero-RTT resumption available for repeat connections. If anything in the path still negotiates 1.2, that upgrade outranks other TLS tuning.

**6. On cellular, waking the radio is the expensive part, not sending the bytes.** Scattered small requests each pay a promotion cost and restart an inactivity tail. Batch them.

**7. Default to SSE for server-to-client push.** Reach for WebSocket only when the client must also send on the same channel or you need binary frames.

## Triage

| What you're doing | Reference |
|---|---|
| Configuring clients/servers, TLS, certificates, keepalive; diagnosing slow first requests | [connection-setup](references/connection-setup.md) |
| Choosing or migrating HTTP versions; auditing build-time asset optimizations | [protocol-choice](references/protocol-choice.md) |
| Picking a realtime transport; anything targeting mobile clients | [realtime-and-mobile](references/realtime-and-mobile.md) |

## A note on currency

The source material for this skill predates widespread HTTP/3 and TLS 1.3, and several of its round-trip calculations have since changed. Where that matters, the references state the current position rather than the historical one — most importantly that TCP head-of-line blocking, which HTTP/2 inherits and cannot fix, is structurally solved by HTTP/3 over QUIC.

Treat any advice you encounter elsewhere that predates 2018 with the same suspicion, particularly around asset bundling and TLS round-trip counts.
