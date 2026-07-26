# agent-skills

Skills that make coding agents better at the things they're confidently wrong about.

Each one is a catalogue of **failure modes bound to code shapes** — not a summary of a topic. The distinction matters: a model already knows what eventual consistency *is*. What it doesn't reliably do is remember it while writing the queue consumer.

## The skills

| Skill | Fires when you're writing | Catches |
|---|---|---|
| [`distributed-data`](skills/distributed-data) | Queries, transactions, replicas, caches, queues, anything writing to two stores | Stale reads, lost updates, write skew, non-idempotent retries, dual-write divergence, clock and lease misuse, hot partitions |
| [`resilience`](skills/resilience) | HTTP/RPC clients, pools, retries, health checks, startup and shutdown | Missing timeouts, cascading failure, pool exhaustion, unbounded queries, capacity cliffs, unoperable services |
| [`api-evolution`](skills/api-evolution) | API shapes, event payloads, DB columns, RPC signatures | Breaking changes that pass tests because old and new code never run together locally |
| [`unit-testing`](skills/unit-testing) | Tests, or reviewing someone else's | Over-mocking, tests coupled to implementation details, tests that can't disagree with the code |
| [`legacy-code`](skills/legacy-code) | Changes to code that has no tests and won't go in a harness | The rewrite reflex, changing behavior before pinning it, editing without a net |
| [`network-performance`](skills/network-performance) | HTTP clients and servers, page load, realtime transports, mobile clients | Optimizing bytes when round trips are the bottleneck, HTTP/1.1 workarounds kept after migrating, radio-expensive polling |
| [`slo-and-alerting`](skills/slo-and-alerting) | Monitoring, alerting rules, reliability targets | Alerting on causes instead of symptoms, mean latency hiding the tail, targets too vague to decide anything |
| [`sql-performance`](skills/sql-performance) | Queries, indexes, schemas, pagination | Wrong composite index order, functions defeating an index, implicit casts, catch-all optional filters, OFFSET pagination |
| [`security`](skills/security) | Untrusted input, auth, crypto, secrets, admin paths | Escaping instead of parameterizing, encoding for the wrong context, authorization assumed from authentication, secrets that were deleted but not rotated |
| [`concurrency`](skills/concurrency) | Anything two threads touch | Atomicity and order violations, `if` instead of `while` on a condition variable, argument-position lock ordering, ordering bugs that only appear off x86 |
| [`durability`](skills/durability) | File writes that must survive a crash | Treating `write()` as persisted, the missing directory fsync, rename before the data is flushed, torn writes |
| [`decomposition`](skills/decomposition) | Splitting something up, extracting a function or module, designing structure | Boundaries drawn on execution order, premature generality, abstractions that enumerate combinations instead of composing them, names that reveal a wrong seam |
| [`design-by-contract`](skills/design-by-contract) | Public methods, interfaces, overrides, error paths | Defensive re-checking of your own preconditions, overrides that quietly demand more than the base, handlers that return as if they succeeded |
| [`data-modeling`](skills/data-modeling) | Schemas, keys, enums, versioning, integrating two systems | Natural keys that change, rows whose "one thing" is ambiguous, relationship metadata smeared onto an endpoint, categories with no slot for the record that arrived |
| [`debugging`](skills/debugging) | Anything broken, especially if a first fix didn't hold | Theorising instead of observing, changing three things at once, declaring victory on one clean rerun |
| [`incident-review`](skills/incident-review) | Postmortems, outage analysis, judging a decision after the fact | Single-root-cause writeups, hindsight bias, grading the outcome instead of the decision, remedies that add more coupling than they remove |

Each is a short router `SKILL.md` plus reference files loaded on demand, so the depth doesn't cost context until it's needed.

**[`engineering-lenses`](skills/engineering-lenses)** indexes all sixteen — route by what you're writing, by what went wrong, or use it as an audit order. If you install one thing, install that.

## Design principles

These were extracted deliberately, against a few rules:

**Failure modes, not concepts.** Anything a competent model already applies by default was cut. What's left is the counterintuitive material — that MySQL/InnoDB's repeatable read doesn't detect lost updates, that Oracle's "serializable" is actually snapshot isolation, that a timeout is a third outcome meaning *unknown* rather than failure.

**Bound to a code shape.** Every rule names something a reviewer could point at in a diff. "Consider your consistency requirements" is useless. "Any `SELECT` whose result decides whether to `INSERT` is a race unless the invariant is a database constraint" is actionable.

**Trigger → what breaks → what to do → how to verify.** The consistent shape means a rule can fire mid-task without needing the surrounding theory.

**Non-overlapping triggers.** Skills whose descriptions compete get invoked unpredictably. `resilience` and `distributed-data` both touch retries, so each carries an explicit boundary section pointing at the other rather than duplicating: `distributed-data` asks *will the data be correct*, `resilience` asks *will the system stay up*.

## Sources

The material was distilled from these books. The text here is original — operational rules written from scratch, not excerpts, summaries, or condensations. None of it substitutes for reading the originals, which are considerably better and go far deeper than a skill file can:

- **Designing Data-Intensive Applications** — Martin Kleppmann → `distributed-data`, `api-evolution`
- **Release It!** — Michael Nygard → `resilience`
- **Unit Testing: Principles, Practices, and Patterns** — Vladimir Khorikov → `unit-testing`
- **Working Effectively with Legacy Code** — Michael Feathers → `legacy-code`
- **Site Reliability Engineering** — Google (free at [sre.google/books](https://sre.google/books/)) → `slo-and-alerting`, and the overload/cascade material in `resilience`
- **High Performance Browser Networking** — Ilya Grigorik (free at [hpbn.co](https://hpbn.co/)) → `network-performance`
- **SQL Performance Explained** — Markus Winand (free at [use-the-index-luke.com](https://use-the-index-luke.com/)) → `sql-performance`
- **Building Secure and Reliable Systems** — Google (free at [sre.google/books](https://sre.google/books/)) → `security` (design)
- **OWASP Cheat Sheet Series** — OWASP (CC-licensed, [cheatsheetseries.owasp.org](https://cheatsheetseries.owasp.org/)) → `security` (code)
- **Rust Atomics and Locks** — Mara Bos (free at [marabos.nl/atomics](https://marabos.nl/atomics/)) → `concurrency`
- **Operating Systems: Three Easy Pieces** — Arpaci-Dusseau (free at [pages.cs.wisc.edu/~remzi/OSTEP](https://pages.cs.wisc.edu/~remzi/OSTEP/)) → `concurrency`, `durability`
- **How Complex Systems Fail** — Richard I. Cook (free at [how.complexsystems.fail](https://how.complexsystems.fail/)) → `incident-review`, plus extensions to `resilience` and `slo-and-alerting`
- **Thinking Forth** — Leo Brodie (CC-licensed, free at [thinking-forth.sourceforge.net](https://thinking-forth.sourceforge.net/)) → `decomposition`
- **Object-Oriented Software Construction** — Bertrand Meyer → `design-by-contract`
- **A Philosophy of Software Design** — John Ousterhout → the interface-depth material in `decomposition`
- **Data and Reality** — William Kent → `data-modeling`
- **Debugging: The 9 Indispensable Rules** — David Agans → `debugging`

If a skill here is useful to you, the book it came from will be more so. Buy them.

## Install

Both Claude Code and Codex CLI use the same `SKILL.md` format, so one copy serves both.

```bash
git clone https://github.com/PlunderStruck/agent-skills.git
cd agent-skills

# Claude Code
ln -s "$PWD"/skills/* ~/.claude/skills/

# Codex CLI
ln -s "$PWD"/skills/* ~/.codex/skills/
```

Symlinking rather than copying means `git pull` updates both tools at once.

To install one skill:

```bash
ln -s "$PWD/skills/resilience" ~/.claude/skills/resilience
```

## Writing your own

The extraction process, if you want to do this with another book:

1. Get the text into a form you can read programmatically.
2. Identify the chapters carrying failure modes rather than concepts. In most books this is a minority of the pages.
3. For each failure mode, write down the code shape that triggers it, the mechanism that breaks, the named remedy, and the check that confirms it's fixed.
4. Delete anything a model already does by default. This is the step people skip, and it's what separates a skill from a wiki page.
5. Split into a router plus references once it exceeds roughly a thousand words.

Good candidates share a property: they're catalogues of things that go wrong, written by someone who watched them go wrong. Books that are mostly conceptual, inspirational, or reference-manual shaped tend not to yield much.

## License

MIT — see [LICENSE](LICENSE).
