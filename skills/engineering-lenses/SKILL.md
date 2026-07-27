---
name: engineering-lenses
description: Index of engineering failure-mode skills — routes to the right one for data correctness, resilience, security, concurrency, durability, SQL and network performance, testing, legacy code, decomposition, contracts, data modelling, debugging, incidents, API compatibility, alerting, interaction/user error, and autonomous agent design. Use when unsure which discipline applies, when auditing a codebase from multiple angles, or as a checklist before shipping something non-trivial.
---

# Engineering Lenses

Eighteen skills, each a catalogue of failure modes bound to code shapes. This routes to the right one.

Each entry below is its own router over reference files, so loading one costs a page, not a book. Read only what the task needs.

## Route by what you're writing

| You are working on | Load |
|---|---|
| Queries, transactions, replicas, caches, queues; writing to two stores | `distributed-data` |
| An HTTP/RPC client, pool, retry, health check, background worker | `resilience` |
| A file write that must survive a crash; a save path; a write-ahead log | `durability` |
| Anything two threads touch; locks, atomics, shared state | `concurrency` |
| Untrusted input, auth, sessions, crypto, secrets, an admin path | `security` |
| SQL, an index, a schema, pagination | `sql-performance` |
| A schema, a primary key, a status enum, versioning, integrating two systems | `data-modeling` |
| An API shape, event payload, DB column, RPC signature others consume | `api-evolution` |
| A public method or interface; an override; an error path | `design-by-contract` |
| Page load, an HTTP client/server, a realtime transport, mobile clients | `network-performance` |
| Tests, or reviewing someone else's | `unit-testing` |
| Code with no tests that won't go into a harness | `legacy-code` |
| Splitting something up; extracting a module; designing structure | `decomposition` |
| Monitoring, alerting rules, reliability targets | `slo-and-alerting` |
| A destructive action, a mode or toggle, a confirm/undo path, an error message | `interaction-and-error` |
| A tool-calling loop, an eval or scorer, an approval gate, an autonomous workflow | `agent-design` |

## Route by what went wrong

| Symptom | Load |
|---|---|
| Something is broken and a first fix didn't hold | `debugging` |
| An outage is over and you're writing it up, or judging a decision | `incident-review` |
| Data is right in one place and wrong in another | `distributed-data` |
| Fine in dev, falls over under load | `resilience`, then `sql-performance` |
| Data was acknowledged then lost | `durability` if single-node, `distributed-data` if replicated |
| Passes on your machine, fails on ARM or under load | `concurrency` |
| Slow despite adequate bandwidth | `network-performance` |
| Tests break on refactors that changed no behaviour | `unit-testing` |
| The same bug keeps coming back in different forms | `debugging` first, then `incident-review` for the pattern |
| Nobody can change one thing without touching six files | `decomposition` |
| Users keep destroying things "by accident", or tickets keep closing as user error | `interaction-and-error` |
| An agent loops, repeats itself, games its eval, or nobody can actually review its output | `agent-design` |

## Auditing a codebase

Running several lenses independently finds more than any one does, because each is blind to what the others see. A workable order:

1. **`security`** — highest cost of being wrong.
2. **`distributed-data`** and **`durability`** — silent corruption beats loud failure for damage.
3. **`resilience`** — availability under dependency failure.
4. **`sql-performance`** and **`network-performance`** — where the time actually goes.
5. **`decomposition`** and **`design-by-contract`** — structural debt, which shapes everything above.
6. **`unit-testing`** and **`legacy-code`** — whether any of it can be changed safely.

**Verify each finding before reporting it.** An audit that emits fifty unverified findings is worse than none, because someone has to triage it and will stop reading. Try to refute each one; keep what survives.

## Pairs that touch

Knowing which of two adjacent skills owns a question:

- **`resilience`** vs **`distributed-data`** — *will the system stay up* vs *will the data be correct*. A retry needs both: backoff so it doesn't amplify an outage, idempotency so it doesn't double-charge.
- **`durability`** vs **`distributed-data`** — bytes on this disk vs agreement across machines. Replication protects against losing a machine, not against every node having buffered the same write.
- **`decomposition`** vs **`design-by-contract`** — where the boundary goes vs what it promises.
- **`debugging`** vs **`incident-review`** — find this cause vs judge the decisions afterward. The second explicitly rejects single-root-cause thinking; the first does not.
- **`unit-testing`** vs **`legacy-code`** — what a test should assert vs how to make untestable code testable at all.
- **`sql-performance`** vs **`data-modeling`** — fast against the schema you have vs whether that schema represents the domain honestly.
- **`interaction-and-error`** vs **`incident-review`** — both use the Swiss cheese model and both reject single-root-cause thinking. The first applies it *before* the failure, designing so the error is hard to make and easy to undo; the second applies it *afterward*, judging decisions without hindsight.
- **`agent-design`** vs **`resilience`** — an agent calling tools *is* a distributed system, so timeouts, retries, and idempotency belong to `resilience` unchanged. `agent-design` covers what the agent should be and who can meaningfully supervise it.
- **`interaction-and-error`** vs **`security`** — whether an action was *intended* vs whether it was *authorized*. Least privilege bounds the blast radius; this bounds the chance a legitimate user triggers it without meaning to.

## What isn't here

Language- and framework-specific guidance, product decisions, and anything requiring knowledge of your particular system. These are lenses for spotting classes of failure — they narrow where to look, and they don't know your codebase.
