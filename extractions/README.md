# Extractions

The intermediate layer between a source book and a published skill.

Each file is everything judged worth keeping from one source, written in our own words — roughly 3,000–4,500 words per book, several times denser than the skill built from it. **This is where detail survives.** A skill is deliberately compressed to what an agent can afford to load mid-task; anything cut on the way to the skill is still here.

## Why this layer exists

Skills are lossy on purpose. That compression is what makes them work — an agent mid-task can load a router and one reference, and a single description per concern means the right skill fires. The alternative people reach for — one skill per chapter — produces dozens of sibling descriptions competing for the same trigger, which makes invocation unpredictable while being *closer to the book and less useful*.

The fix for "don't lose the detail" isn't bigger skills. It's this layer plus a re-obtainable source.

| Layer | Committed | Purpose |
|---|---|---|
| `sources/` | No (gitignored) | The book. Re-obtainable; provenance recorded in the manifest |
| `extractions/` | **Yes** | Everything worth keeping, our own prose. The durable artifact |
| `skills/` | Yes | What fires mid-task. Compressed |

## Contents

| Extraction | Source | Skill it fed |
|---|---|---|
| [`resilience`](resilience.md) | Release It! — Nygard | `resilience` |
| [`sre-overload`](sre-overload.md) | Site Reliability Engineering — Google | `resilience`, `slo-and-alerting` |
| [`unit-testing`](unit-testing.md) | Unit Testing: P/P/P — Khorikov | `unit-testing` |
| [`legacy-code`](legacy-code.md) | Working Effectively with Legacy Code — Feathers | `legacy-code` |
| [`sql-performance`](sql-performance.md) | SQL Performance Explained — Winand | `sql-performance` |
| [`network-performance`](network-performance.md) | High Performance Browser Networking — Grigorik | `network-performance` |
| [`security-design`](security-design.md) | Building Secure and Reliable Systems — Google | `security` |
| [`security-code`](security-code.md) | OWASP Cheat Sheet Series | `security` |
| [`concurrency-memory-model`](concurrency-memory-model.md) | Rust Atomics and Locks — Bos | `concurrency` |
| [`concurrency-and-durability`](concurrency-and-durability.md) | OSTEP — Arpaci-Dusseau | `concurrency`, `durability` |
| [`complex-systems-failure`](complex-systems-failure.md) | How Complex Systems Fail — Cook | `incident-review` |
| [`decomposition`](decomposition.md) | Thinking Forth — Brodie | `decomposition` |

**Not present:** `distributed-data` and `api-evolution` were extracted from *Designing Data-Intensive Applications* by reading the text directly rather than via a sub-agent, so there's no separate intermediate file — those two skills are themselves the extraction, and are correspondingly more detailed than the others.

## Re-distilling

To sharpen a skill, or to build a different one from the same source, start here rather than re-reading the book. The extraction already discarded what a competent model knows by default, which is the expensive judgement call.
