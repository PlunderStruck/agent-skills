---
name: legacy-code
description: Change code that has no tests, without breaking it — finding seams to substitute behavior, pinning current behavior with characterization tests, and breaking dependencies so untestable code can run in a harness. Use when you need to modify code you can't easily test, can't instantiate a class in a test, are tempted to rewrite something that works, or are adding behavior to a large untested method.
---

# Legacy Code

## Purpose

Legacy code means code without tests. Not old code, not ugly code — any code lacking a fast way to tell you that you just broke something.

This skill exists to correct one specific reflex: **seeing tangled untested code and reaching for a rewrite.** A rewrite throws away the single thing that code has going for it — it works, in ways nobody has fully enumerated, including edge cases discovered in production and never written down. The alternative is smaller and less satisfying: find a seam, break one dependency, pin the current behavior, then change it.

The goal is never "make this clean." It is "know immediately when I've broken something." Cleanliness becomes affordable only after that.

## When this applies

- You need to change code that has no tests
- A class can't be instantiated in a test — the constructor reaches for a database, a network call, or an object graph you can't build
- A method runs but you can't observe what it did
- You're adding behavior to a long method you don't fully understand
- You catch yourself thinking the code should just be rewritten

## The central bind

To get code under test you usually have to edit it. Editing untested code is the exact risk you're trying to avoid.

The way out is to make those edits small and mechanical enough that the risk of getting one wrong is smaller than the value of the seam it opens — and to apply stricter discipline while you're working without a net. See [breaking-dependencies](references/breaking-dependencies.md) for that discipline.

## Seams

A **seam** is a place where you can change behavior without editing the code at that place. Its **enabling point** is where the choice actually gets made.

A plain call, constructed and used in one scope with no way to swap the type, is not yet a seam. You have to create one — usually by moving construction outward or making the target overridable.

| Seam type | Where it lives | Enabling point | Use when |
|---|---|---|---|
| **Object** | A call whose target isn't pinned by the call's own text | Constructor argument, factory, which subclass is built | Default choice — visible in source, no build cooperation needed |
| **Link** | Resolution happens after the caller compiles | Build script, module path, search path | A dependency called from dozens of places |
| **Preprocessing** | A text substitution stage before compilation | Build flag, alternate definitions | Nothing else reaches it |

Prefer object seams. Link and preprocessing seams are invisible in a diff, so reach for them only when the obstruction is systemic — and keep the divergence conspicuous in the build configuration.

## Every obstacle is sensing or separation

- **Separation** — you can't get the code to *run* in a harness. Many causes, many fixes; that's why there's a technique catalogue.
- **Sensing** — the code runs but you can't *observe* its effect, because it disappeared into a collaborator. One recurring fix: a fake behind a seam that records what it was told, so the test can assert against the recording.

Naming which one you're facing narrows the options immediately.

## The change procedure

1. **Find the change points** — where the new behavior belongs.
2. **Find the test points** — where its effects can be observed.
3. **Break dependencies** — get the change point into a harness by exploiting a seam. This is the uncomfortable step, done without a net.
4. **Write characterization tests** — pin what the code does *now*, before touching it.
5. **Make the change**, then refactor now that the area is covered.

Treat it as a loop. Tested regions accumulate as islands that eventually grow together, so later changes nearby get progressively cheaper.

## Rules that apply without loading anything

**1. Don't rewrite. Find a seam.** The rewrite discards undocumented working behavior and replaces a known-imperfect system with an unknown one.

**2. Pin behavior before changing it.** A characterization test records what the code actually does, with no notion of whether that's correct. Let the code compute the expected value and paste it in — don't reason toward what it *should* return.

**3. Don't read the constructor hunting for the problem.** Write a bare construction call with no assertions and let the compiler enumerate what's missing.

**4. One goal per edit while working without tests.** Note the second change you notice; don't interleave it. Interleaved changes can't be unwound independently in your head.

**5. Preserve signatures by cut-and-paste, not retyping.** When relocating code purely to open a seam, copy parameter lists and call sites exactly. Real API cleanup waits until coverage exists.

**6. Trace effects along three channels** when scoping what to test: the return value, mutated parameters read later, and shared or global state. The third is least visible and most likely to bite.

**7. Accept temporary ugliness.** An extra overload, a wrapper class, a loosened access modifier — each is the cheapest move that removes one named obstacle. They're staging points, not designs.

## Triage

| Situation | Reference |
|---|---|
| Need to pin current behavior; unsure how much to test or when to stop | [characterization-tests](references/characterization-tests.md) |
| Can't instantiate; can't observe; editing safely before tests exist; monster methods | [breaking-dependencies](references/breaking-dependencies.md) |
| Need the specific named technique for a specific obstacle | [technique-catalogue](references/technique-catalogue.md) |

## Boundary with `unit-testing`

- **`unit-testing`** — how to design tests worth having for code you can already test. Mock versus stub, behavior versus implementation detail.
- **`legacy-code`** — how to make untestable code testable at all, and how to change it safely in the meantime.

Feathers gets you to a harness; Khorikov tells you what to write once you're there. When adding new behavior to legacy code, sprout it into a separately testable unit and apply `unit-testing` to that new unit — rather than forcing the surrounding mess under test first.
