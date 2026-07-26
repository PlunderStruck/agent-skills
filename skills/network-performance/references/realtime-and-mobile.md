# Realtime Transports and Mobile

## Choosing a realtime transport

Decide on two axes first: **direction** (does the client need to send on the same channel?) and **payload type** (text or binary?).

| Transport | Direction | Overhead | Use when |
|---|---|---|---|
| **Short-interval polling** | Request/response | Full protocol overhead per poll (~800 bytes) even when nothing changed | Updates are rare and latency-tolerant |
| **Long polling** | Request/response, server holds open | Lower than short polling at low message rates; converges with it at high rates | Fallback where SSE and WebSocket aren't viable |
| **SSE** | Server → client only, UTF-8 only | ~5 bytes per message | **Default for server push** |
| **WebSocket** | Full duplex, binary capable | ~6–14 bytes framing; client→server frames masked (+4) | Client must send on the same channel, or binary payloads |

**Why short polling is usually wrong for live data:** every poll pays full overhead whether or not anything changed. Ten thousand clients polling every 60 seconds at ~850 bytes each is over a megabit per second of pure overhead delivering nothing. Shortening the interval multiplies the waste; lengthening it raises average latency to half the interval. You lose either way.

**SSE deserves to be the default** for pure server push. It runs over one ordinary HTTP connection, and the browser's `EventSource` handles reconnection and resume via `Last-Event-ID` automatically — reconnection logic you would otherwise write and get subtly wrong.

**WebSocket has its own head-of-line blocking.** Frames from different messages can't interleave on one socket — there's no per-message stream ID — so a large message blocks everything behind it. Split large payloads at the application level.

**Use WSS in production, always.** Plain `ws://` is frequently mishandled by intermediary proxies that don't understand the protocol — buffered, blindly upgraded, or misclassified. TLS makes them pass it through.

## The failure that gets everyone

**Proxy and load-balancer idle timeouts default far below what long-lived connections need.** This is the single most common way SSE and WebSocket break in production while working perfectly in development, where there's no proxy in between.

Raise read, send, and idle-tunnel timeouts explicitly — on the order of an hour, not the default seconds or low minutes. Verify at every hop: CDN, load balancer, reverse proxy, application server.

## Mobile radio economics

**Waking the radio is the expensive part, not sending the bytes.**

A cellular radio is a state machine. Promoting from idle to a connected high-power state is a control-plane cost paid *before any bytes move*: under ~100 ms on LTE, 150–500 ms on HSPA, and up to two seconds on older 3G. After your last packet, the radio holds high power for a carrier-configured inactivity tail — seconds — before demoting.

So scattered small requests each pay promotion *and* restart the tail, instead of sharing one wake cycle. The illustrative figure: analytics beacons accounting for 0.2% of bytes transferred but nearly half of total power consumption, entirely from repeated radio tails.

Total one-shot request budget before your request is even sent — radio promotion, DNS, TCP, TLS — runs roughly 200–3500 ms on 3G and 100–600 ms on 4G.

**What follows:**

- **Batch and coalesce.** Do everything in one burst while the radio is already awake, then let it idle.
- **Defer non-critical requests** until the radio is up for another reason.
- **Treat background polling as expensive by default.** Push generally beats polling, but a high-frequency push stream is just as costly — aggregate notifications too.
- **Carrier NAT timeouts run 5–30 minutes.** A long-lived connection needs an application-level keepalive around every 5 minutes or the carrier silently drops it. Silently is the operative word: the client often doesn't learn until its next send fails.

**Confirm:** profile radio and network energy on a real device around a suspected chatty feature. Look for many small transfers separated by more than the demotion timer. Verify a supposedly persistent connection survives an idle period longer than the carrier timeout.

## What to check

- Is short-interval polling being used where SSE would do?
- Does the transport choice match the actual direction requirement, or was WebSocket picked by default?
- Are proxy and load-balancer idle timeouts raised at **every** hop?
- Is `wss://` used rather than `ws://`?
- Are large WebSocket messages split, or do they block the socket?
- Do background requests batch, or fire on independent timers?
- Is there an application-level keepalive under five minutes on long-lived mobile connections?
