# Types as Contracts

Everything in the main skill describes a discipline enforced by convention. Where the language allows it, the same guarantees can be enforced by the compiler instead — which changes them from things a reviewer must notice into things that cannot be written.

**The single move underneath all of this:** shift a runtime check to construction time, and let the resulting type stand as proof the check happened.

## Wrap primitives that carry meaning

**Trigger.** A signature with two or more adjacent parameters of the same primitive type — `transfer(from: int, to: int, amount: decimal)`. Or a value flowing through several layers typed only as `string`, its meaning carried entirely by a parameter name the compiler cannot see.

**What goes wrong.** Transposition compiles cleanly: `transfer(to, from, amount)` moves money the wrong way and nothing objects, because both slots are `int`. Separately, an unvalidated value reaches a sink three layers from where it entered, and by then nobody can tell from its type whether the one function responsible for checking it ever ran. And with a bare primitive there is nowhere to hang the validation, so every consumer re-decides whether to trust it — duplicated in some places, skipped in others.

**What to do.** Give each domain concept its own type, distinct from other types over the same primitive, even when the runtime representation is identical.

| Language | Mechanism |
|---|---|
| TypeScript | Branded type + parsing function as the only producer |
| Rust | Newtype with a private field, constructed via `TryFrom` |
| Java / C# | `final`/`sealed` wrapper, private constructor, static factory |
| Python | Frozen dataclass with validating `__post_init__` — `NewType` alone adds no runtime guard |
| No nominal typing | Factory as the sole documented entry point; **state the enforcement gap** rather than assuming convention holds |

**Check.** A new function with two same-typed adjacent parameters is a transposition risk in any language. The same validation logic appearing in more than one place against conceptually one value is the "nowhere to hang it" symptom already showing.

## Make illegal states unrepresentable

**Trigger.** Two or more fields whose validity depends on each other — most often a status plus optionals that are only meaningful for some statuses. Or a rule of the form *at least one of A or B* implemented as two independent optionals.

**What goes wrong.** The compiler accepts a value violating the rule. Something builds it — deserialization, a partial update, a test fixture — and every downstream reader either re-derives the invalid combinations or doesn't. The bug surfaces past a service boundary, far from the construction site.

**What to do.** Restructure so the invalid combination has no corresponding value. Not a check that rejects it — a shape that cannot express it.

```
// before: both nullable, so "neither" is constructible and shouldn't be
Contact { email: string?, postal: string? }

// after: no case has neither
Contact = EmailOnly(email) | PostalOnly(postal) | Both(email, postal)
```

A `Contact` with neither is no longer a bug to catch. It is a value that cannot be built.

Sum types in TypeScript (discriminated union, `never` default arm), Rust (`enum`, exhaustive `match`), Java 17+/C# (sealed hierarchy), Python (`Union` of frozen dataclasses, exhaustiveness only if `mypy --strict` runs in CI with `assert_never`).

**Check.** Flag any diff adding two optional fields to one type together. For every nullable field ask: is there a status for which present-versus-absent isn't a live choice? If so it belongs on a variant, not behind a null check.

## Validate at the boundary; the type is the proof

**Trigger.** A function re-checks something its argument's type nominally already promises, or the same validation appears at more than one layer.

**What goes wrong.** The copies drift, and which is correct becomes a live argument. Worse, nothing stops a caller reaching the inner function directly with an unchecked value, because the type doesn't distinguish checked from unchecked — so a dropped validation step is invisible to compiler and reviewer alike.

**What to do.** Give validated and unvalidated values *different types*. Make the raw constructor unreachable; expose one factory that checks and returns either the validated value or an explicit failure.

```
ValidatedEmail:
  private new(raw)
  static parse(raw) -> ValidatedEmail | Failure:
      if not matches(raw, PATTERN): return Failure
      return new ValidatedEmail(raw)     // the only path to an instance
```

**This is the non-redundancy rule made enforceable.** The main skill argues a function must not re-check its own precondition. Here it *cannot* usefully do so — no code path can hand it a value that skipped `parse`.

**Check.** A factory returning the raw primitive rather than a wrapper isn't doing its job. If `isValid(x)` is called more than one call deep from where `x` entered, ask why `x`'s type doesn't already carry the guarantee.

## Represent each state as its own type

The common case of illegal states: a status field beside fields populated only for some values.

**What goes wrong.** Every reader of a state-specific field must remember to check the status, and nothing enforces it — a report reads a signature on a pending record and gets a stale value from a transition that never cleared it. Transition logic has no home and becomes an `if/else` chain repeated at each call site, so adding a status means finding them all by hand.

**What to do.** Each state gets its own type holding only the fields meaningful in it. A function that only makes sense once delivered takes the delivered shape as its parameter type — the compiler refuses a pending value outright, rather than "only if you forgot the guard."

This is the same claim as *define errors out of existence* in the main skill, reached from the type side: a nullable field every reader must guard becomes an unremarkable instance of the general case.

## Constrain values at their edge — and on every transition

**Trigger.** A rule narrower than the primitive — max length, format, valid range — enforced at the point of use, if at all.

**What goes wrong.** A too-long string reaches a fixed-width column and truncates silently. A quantity that should never go negative gets decremented past zero because the decrement operates on a bare integer.

**What to do.** Define the constraint on its own type, checked at construction — **and re-checked on every operation that could push it out of range.** That second half is what people skip:

```
NonNegativeQty:
  static of(n) -> NonNegativeQty | Failure   // n >= 0
  decrement()  -> NonNegativeQty | Failure   // re-validated, not exempted
```

Where the storage layer genuinely cannot express the constraint, decide *once, explicitly* whether an over-length value is rejected or truncated — rather than letting it emerge from whichever layer happens to run first.

## When the wrapper isn't worth it

Applied unconditionally, this becomes ceremony that grows faster than the bugs it prevents. The tell: a small script starts to feel like it needs production-domain ceremony, and that becomes a reason to avoid a change rather than a net for making one.

**Wrap when at least one holds:**

- It carries a validation rule narrower than its primitive.
- It sits beside another value of the same primitive type — transposition risk.
- It crosses a module, service, or process boundary where the receiver needs a guarantee the sender already established.

None of those, and it stays a primitive. This is a per-value judgement, and it reasonably differs between a library's public surface (favour wrapping — callers are strangers) and its internals (favour primitives — the whole file is visible at once).

**Check.** A new wrapper with one call site and no validation beyond "is a string" is a candidate to inline back out. A bare primitive at a service boundary beside a same-typed sibling is a candidate to wrap.
