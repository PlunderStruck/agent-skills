---
name: api-evolution
description: Apply compatibility rules when changing anything another process will read — API request/response shapes, event and message payloads, database columns and schemas, serialized records, RPC signatures, config formats, or stored JSON. Catches breaking changes that pass tests because old and new code never run together locally, silent field loss during rolling upgrades, and RPC written as if it were a local call.
---

# API and Schema Evolution

## Purpose

Most breaking-change bugs are invisible in development because development runs one version of the code. Production runs two — during every rolling deploy, and indefinitely for any client you don't control. This skill is about the changes that are safe under those conditions and the ones that aren't.

The governing fact: **data outlives code.** You can replace every running process in minutes. You cannot replace five-year-old rows, in-flight messages, or a mobile app version someone declines to update.

## When this applies

- Adding, removing, renaming, or retyping a field in any API response or request
- Changing an event or message payload
- Any database migration touching column names, types, or nullability
- Changing a function signature crossing a service boundary or an RPC interface
- Changing anything serialized into a queue, a cache, a cookie, or a file
- Introducing or changing a version negotiation scheme

## The two directions

Compatibility is a relationship between the writer and the reader. Name which direction you need — they have different rules and people routinely conflate them.

- **Backward compatibility** — new code can read data written by old code. Usually easy: you know the old format.
- **Forward compatibility** — old code can read data written by new code. Harder: the old code must ignore what it doesn't recognize, and it had to have been written that way already.

Which you need depends on how the data flows:

| Flow | Requirement |
|---|---|
| Database (write now, read later) | Backward — and forward too, because during a rolling upgrade old instances read new writes |
| Service request (client → server) | Backward on the server: servers upgrade first, so a new server must accept old requests |
| Service response (server → client) | Forward on the client: old clients must tolerate new response fields |
| Message queue | Both, in both directions — producers and consumers deploy independently and in any order |
| Public API / mobile client | Both, effectively forever. You cannot force an upgrade |

## Safe and unsafe changes

**Safe:**
- Adding an optional field / optional request parameter
- Adding a new field to a response
- Adding a new enum value *only if* every consumer already has a defined fallback for unknown values — otherwise this breaks them
- Adding a new column with a default (most relational databases do this without rewriting existing rows; MySQL often rewrites the table anyway)
- Adding a new endpoint, event type, or message type

**Unsafe:**
- Removing or renaming a field
- Changing a field's type, or narrowing it (`string` → `enum`, `int32` → `int16`)
- Making an optional field required
- Changing the meaning of an existing field while keeping its name — the worst of these, because nothing detects it
- Changing default values that consumers rely on
- Tightening validation on an existing input

For a rename or a type change, do it as expand/contract: add the new field, write both, migrate readers, stop writing the old one, then remove it — with a deploy boundary between each step.

## The silent field-loss bug

Worth calling out because it's easy to write and hard to see.

Old code reads a record, updates one field, writes the whole record back. The record contained a field added by newer code that the old code doesn't know about. If the old code deserialized into a typed model object and re-serialized it, **that unknown field is now gone** — silently, with no error, and the newer code's data is destroyed.

Encoding formats like Protocol Buffers, Thrift, and Avro preserve unknown fields at the wire level. That does not save you if your application decodes into a model object in between. Check:

- Any read-modify-write of a serialized document (JSON column, message payload, cached object)
- Any consumer that republishes a message to another topic after deserializing it
- ORM models that don't map every column

The fix is either to preserve the raw unrecognized fields explicitly, or to avoid whole-record rewrites in favor of targeted field updates.

## Versioning

There is no industry agreement on how a client indicates its API version. The common options:

- Version in the URL path (`/v2/orders`) — most visible, easiest to route, most churn
- Version in an `Accept` header — cleaner URLs, easier to miss
- Version stored per API key, changed through an admin interface — good for enterprise integrations

Pick one and be consistent. What matters more than the mechanism: once you break compatibility on a public API, **you are running both versions side by side for as long as clients exist.** Budget for that before choosing to break rather than extend.

## RPC is not a local call

Frameworks that make a remote call look like a method call (gRPC, Thrift, older CORBA/RMI/DCOM) hide differences that the calling code must handle anyway:

- A local call succeeds or throws. A remote call has a third outcome: **timeout, meaning unknown**. The request may have executed. Code that treats a timeout as "didn't happen" is wrong.
- Retrying a remote call duplicates the effect if the response — not the request — was what got lost. Retries require idempotency; see the `distributed-data` skill.
- Latency is variable by orders of magnitude, not roughly constant.
- Arguments must be serialized; you cannot pass references, and large objects are a real cost.
- Types don't map cleanly across languages. JavaScript numbers above 2^53 are the classic example.

Practical implications for code you write: set explicit timeouts (never rely on the default), make every mutating remote call carry an idempotency key, and represent "unknown" as a distinct outcome from "failed" wherever the distinction changes behavior.

Where you have a choice: REST for public and cross-organization APIs (debuggable with curl, universal tooling, no code generation); binary RPC for internal service-to-service calls where performance matters and you control both ends.

## Schema formats

If you're choosing one:

- **JSON without a schema** — universal, debuggable, no compatibility checking whatsoever. Every compatibility guarantee is manual and enforced by review.
- **JSON Schema / OpenAPI** — documents the contract and enables validation; compatibility checking depends on tooling you have to run.
- **Protocol Buffers / Thrift** — field tags decouple names from wire format, so renames are free and removals are safe if tag numbers are never reused. Compatibility rules are checkable.
- **Avro** — no tag numbers; reader and writer schemas are reconciled by name, which supports dynamically generated schemas well (a database dump whose schema follows the table).

The value of a schema here isn't type safety at runtime — it's that compatibility becomes something a tool can check in CI rather than something a reviewer has to notice.

## What to check in a diff

- Every changed field: classify as safe or unsafe against the table above; if unsafe, is there an expand/contract plan?
- Every migration: can the currently-deployed code still run against the new schema? Can the new code run against the old one?
- Whether old and new versions will overlap during deploy — if the answer is "only for a few minutes," that's still an overlap.
- Any read-modify-write of a serialized record, for field loss.
- New enum values, against every consumer's unknown-value handling.
- Consumers you don't control, and whether the change forces them to upgrade.
