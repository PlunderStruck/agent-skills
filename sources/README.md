# Sources

Raw source texts the skills were distilled from. **Contents are gitignored** — this directory holds working copies for re-extraction, not redistributable material.

The manifest below records what belongs here and where it came from, so the provenance is version-controlled even when the text isn't.

## Freely licensed — safe to re-fetch, committed by URL not by copy

| File | Source | Licence | Feeds |
|---|---|---|---|
| `thinking-forth.pdf` | [thinking-forth.sourceforge.net](https://thinking-forth.sourceforge.net/) | CC BY-SA | `decomposition` |
| `how-complex-systems-fail.html` | [how.complexsystems.fail](https://how.complexsystems.fail/) | Free to distribute | `incident-review` |
| — | [fsharpforfunandprofit.com](https://fsharpforfunandprofit.com/series/designing-with-types/) | Author's free series | `design-by-contract`, `data-modeling` |
| — | [use-the-index-luke.com](https://use-the-index-luke.com/) | Author's free web edition | `sql-performance` |
| — | [sre.google/books](https://sre.google/books/) | Google, free | `slo-and-alerting`, `resilience`, `security` |
| — | [hpbn.co](https://hpbn.co/) | Author + O'Reilly, free | `network-performance` |
| — | [cheatsheetseries.owasp.org](https://cheatsheetseries.owasp.org/) | CC | `security` |
| — | [marabos.nl/atomics](https://marabos.nl/atomics/) | Author, free | `concurrency` |
| — | [pages.cs.wisc.edu/~remzi/OSTEP](https://pages.cs.wisc.edu/~remzi/OSTEP/) | Authors, free | `concurrency`, `durability` |

Web sources were read directly rather than archived — re-fetch from the URL when re-extracting.

## Copyrighted — supply your own copy

These are not fetched, not stored here by default, and never committed. If you own or otherwise obtain a copy, drop it in this directory under the filename below and the extraction tooling will find it.

| Expected filename | Work | Feeds |
|---|---|---|
| `ddia.*` | Designing Data-Intensive Applications — Kleppmann | `distributed-data`, `api-evolution` |
| `release-it.*` | Release It! — Nygard | `resilience` |
| `unit-testing.*` | Unit Testing: Principles, Practices, and Patterns — Khorikov | `unit-testing` |
| `wewlc.*` | Working Effectively with Legacy Code — Feathers | `legacy-code` |
| `data-and-reality.*` | Data and Reality — Kent | `data-modeling` |
| `debugging-9-rules.*` | Debugging: The 9 Indispensable Rules — Agans | `debugging` |
| `transaction-processing.*` | Principles of Transaction Processing — Bernstein & Newcomer | `distributed-data` |
| `oosc.*` | Object-Oriented Software Construction — Meyer | `design-by-contract` |
| `philosophy-of-software-design.*` | A Philosophy of Software Design — Ousterhout | `decomposition` |
| `the_design_of_everyday_things.pdf` | The Design of Everyday Things, Revised & Expanded — Norman | `interaction-and-error` |
| `normal-accidents.*` | Normal Accidents — Perrow | *(pending — every edition found is an image scan)* |

## Why the extractions matter more than these files

See [`../extractions/`](../extractions/). Those are original-prose distillations of each source — far denser than the published skills, and committed, because they're our own writing rather than the source text.

The three layers, from most to least detail:

1. **Source** — the book. Not committed. Re-obtainable.
2. **Extraction** — everything judged worth keeping, in our own words. Committed. This is the durable artifact.
3. **Skill** — what an agent should load mid-task. Committed. Deliberately compressed.

A skill is lossy on purpose. The extraction is where the detail survives.
