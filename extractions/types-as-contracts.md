# Extraction: types-as-contracts

**Source:** Designing with Types — Scott Wlaschin (fsharpforfunandprofit.com)

> Operational rules distilled in our own words, plus the placement analysis that decided where this material landed.

---

# Designing with Types — Operational Rules

Extracted from Scott Wlaschin's "Designing with Types" series, restated language-neutrally. Each rule gives a trigger to recognize in a diff, the concrete bug the untyped version invites, the typed fix, and translations for TypeScript, Rust, Java/C#, and Python — with a fallback for languages that can't express the constraint at all.

The unifying mechanism underneath all seven rules is: **move a runtime check to construction time, and let the resulting type stand as proof the check happened.** Everything below is a variation on that move, or on knowing when not to make it.

---

## 1. Wrap primitives that carry domain meaning

**Trigger:** a function signature has two or more parameters of the same primitive type sitting next to each other — `transfer(fromAccount: int, toAccount: int, amount: decimal)`. Or a value flows through several layers of a call stack typed only as `string`/`int`, with its meaning carried entirely by a parameter name (`email`, `ssn`, `zipCode`) that the compiler cannot see.

**What goes wrong:** argument transposition compiles cleanly — `transfer(toAccount, fromAccount, amount)` moves money the wrong direction, and nothing objects, because both slots are `int`. Separately, an unvalidated value reaches a sink three layers from where it entered (an SMTP call, a file path, a SQL string), because by the time it gets there nobody can tell from its type whether the one function responsible for checking it ever ran. And because the type is a bare primitive, there's no single place to hang validation or behavior — every consumer re-decides whether to trust the value, so checks get duplicated in some places and skipped in others.

**What to do:** give each domain concept its own type, distinct from other types built on the same primitive, even when the runtime representation is identical.

```
EmailAddress = wraps(string)
ZipCode      = wraps(string)
// same underlying representation, incompatible types
```

- **TypeScript:** a branded/nominal type — `type EmailAddress = string & { readonly __brand: 'EmailAddress' }` — produced only by a parsing function, `parseEmailAddress(s: string): EmailAddress | undefined`. A plain `string` can't be passed where `EmailAddress` is expected without an explicit (and therefore reviewable) cast.
- **Rust:** a newtype — `struct EmailAddress(String)` — with a private field, constructed only via `TryFrom<&str>`.
- **Java/C#:** a small `final`/`sealed` class wrapping the primitive, private constructor, static factory returning the instance or an `Optional`/`null`.
- **Python:** a frozen dataclass with a validating `__post_init__`; `typing.NewType` alone is compile-time-only and adds no runtime guard, so pair it with a factory function if the guarantee needs to survive contact with untyped callers.
- **Fallback (no nominal typing at all):** a validating factory function as the sole documented entry point, with the raw constructor treated as private by convention. The language can't stop a bypass; code review has to catch it. Name this gap explicitly rather than assuming the convention holds.

**How to check:** scan a diff for a new function with two-or-more adjacent parameters of the same base type — that's a transposition risk independent of language. Look for a primitive-typed field whose name encodes a domain concept but whose type is `string`/`int`. Look for the *same* validation logic (a regex, a range check) appearing in more than one function against what is conceptually the same value — that's the "nowhere to hang it" symptom already manifesting.

---

## 2. Make illegal states unrepresentable

**Trigger:** a record has two or more fields whose validity depends on each other — most often a `status` field plus several optional/nullable fields, where which ones are meaningful changes depending on the status. Or a business rule of the shape "at least one of A or B," "A implies B," "at most one of A, B, C" implemented as independently-optional fields rather than as a structural constraint.

**What goes wrong:** the compiler happily accepts a value that violates the rule — a record with neither an email nor a postal address despite "must have at least one" being non-negotiable, or a `status: Delivered` with no delivery date. That value gets built somewhere (deserialization, a partial update, a test fixture) and now every downstream reader either re-derives the invalid combinations and guards against them, or doesn't — and the bug surfaces later, often past a service boundary where it's hard to trace back to the point of construction.

**What to do:** restructure the type so the invalid combination has no corresponding value at all — not a check that rejects it, a shape that cannot express it. Replace "one record, several optional fields" with a closed set of variants, each carrying only the fields meaningful to that case.

```
// before — both nullable, both-null is a valid value, and it shouldn't be
Contact { email: string?, postalAddress: string? }

// after — no case has neither
Contact = EmailOnly(email)
        | PostalOnly(postalAddress)
        | Both(email, postalAddress)
```

Now a `Contact` with neither field isn't a bug to catch at read time — it's a value that cannot be constructed. And every function consuming `Contact` is pushed toward handling all three cases explicitly.

- **TypeScript:** a discriminated union tagged by a literal field; `switch` over the tag with a `never`-typed default arm turns a missed case into a compile error.
- **Rust:** an `enum` with variant-carried data; `match` is exhaustive by default.
- **Java 17+/C#:** a `sealed`/`abstract` hierarchy, one subtype per case, with pattern matching over the sealed set.
- **Python:** a `Union` of frozen dataclasses; exhaustiveness is enforced only if `mypy --strict` runs in CI and an `assert_never` sits in the default arm — a convention backed by tooling, not a language guarantee.
- **No sum types available:** one class per variant behind a shared interface, obtained only through a factory — the same smart-constructor idea as Rule 1, applied to a whole state rather than one field.

**How to check:** flag any diff adding two or more nullable/optional fields to the same type together. Flag any new enum value added to a `status` field with no corresponding new shape for what only that value implies. For every nullable field, ask: is there a status value for which "present vs. absent" isn't actually a live choice? If yes, it belongs on a variant, not behind a null check.

---

## 3. Validate at the boundary; let the type be the proof

**Trigger:** a function's first few lines re-check something its argument's type nominally already promises — `function send(email: string) { if (!isValidEmail(email)) throw ... }` — or the same validation is written more than once against what is conceptually the same value as it passes between layers.

**What goes wrong:** the checks drift apart over time — two copies of "is this a valid email" disagree on an edge case, and which one is correct becomes a live argument. Worse, nothing stops a caller from reaching the inner function directly with an unchecked value, because the type of the argument doesn't distinguish "checked" from "unchecked" — there's no way for a reviewer, let alone the compiler, to notice that a validation step got silently dropped somewhere upstream.

**What to do:** give validated and unvalidated values different types. Make the raw constructor unreachable and expose a single factory that performs the check and returns either the validated value or an explicit failure. Holding a value of the validated type is then not a comment claiming it was checked — it's the only way such a value can exist. **This is Meyer's precondition-established-once claim, enforced by the type checker instead of by convention:** validate once at the boundary, and everything downstream is entitled to assume the condition holds without re-testing it.

```
ValidatedEmail:
  private new(raw)
  static parse(raw) -> ValidatedEmail | Failure:
    if not matches(raw, EMAIL_PATTERN): return Failure
    return new ValidatedEmail(raw)      // the only path to an instance
```

A function taking `ValidatedEmail` has no reason to check the format again — no code path can hand it one that skipped `parse`.

- **TypeScript:** private constructor + static factory returning a `Result<T, E>` (or `T | undefined`), combined with the branded type from Rule 1 so a bare `string` is rejected at the call site.
- **Rust:** `TryFrom<&str>` as the only constructor for a struct with a private field — close to free, since the compiler already enforces module-private fields.
- **Java/C#:** private constructor, static `of`/`create` returning `Optional<T>`/`T?`, class marked `final`/`sealed` so subclassing can't bypass the constructor.
- **Python:** leading-underscore constructor plus a `classmethod` factory is convention only; if the guarantee has to hold even against a caller who ignores the convention, put the check in `__post_init__` on a frozen dataclass so even direct construction fails loudly.
- **No visibility modifiers at all:** the factory is the documented sole entry point, the raw constructor is flagged in review whenever called directly, and the enforcement gap is stated rather than assumed away.

**How to check:** find a validation check duplicated across more than one function operating on "the same" value. Find a factory function that returns the raw primitive type instead of a validated wrapper — a factory not actually doing its job. In review, if `isValid(x)` is called more than one call-deep from where `x` entered the system, ask why `x`'s type doesn't already guarantee it.

---

## 4. Represent each state as its own type

**Trigger:** a specific, very common instance of Rule 2 — a `status`/`state` field paired with several fields that are populated only for some status values (`deliveryDate`, `signature` meaningful only once `status == Delivered`).

**What goes wrong:** every reader of a state-specific field has to remember to check the status first, and nothing enforces that they do — a report generator reads `signature` on a `Pending` record and gets an empty string, or a stale value left behind by a previous transition that was never cleared. Transition logic ("what happens if `markDelivered` is called on something already `Delivered`?") has no natural home; it becomes an `if/else` chain against the enum, repeated at every call site, and adding a new status means finding every such chain by hand rather than being told about the gap by the compiler.

**What to do:** give each state its own type, holding only the fields meaningful in that state, with the overall type a sum over those state-specific shapes rather than one shape with a sibling status field.

```
Pending   { orderId }
Shipped   { orderId, carrier, trackingNumber }
Delivered { orderId, carrier, trackingNumber, deliveredAt, signature }

Package = Pending(..) | Shipped(..) | Delivered(..)
```

A function that only makes sense once delivered (`printReceipt`) takes the `Delivered` shape as its parameter type directly — the compiler refuses to let it be called on a `Pending` value, full stop, not "only if you forgot the guard."

- **TypeScript/Rust:** discriminated union / enum as in Rule 2; a function specific to one state takes the narrowed variant's payload type, not the whole union.
- **Java/C#:** sealed hierarchy (or the State pattern, pre-sealed-classes), one concrete class per state, each exposing only the operations valid for it.
- **Python:** `Union` of per-state dataclasses; a state-specific function's parameter is annotated as that one dataclass, so mypy rejects a `Pending` argument where `Delivered` is expected.
- **Fallback:** one class per state behind a shared interface for cross-state operations only — the discipline is refusing to put a field on the shared base that not every state populates.

**How to check:** search a diff for a status/state enum sitting beside fields that aren't set for every value of that enum. Search for `if status == X` immediately guarding access to a field "supposed to" only be valid under `X` — every such guard is a hint the field belongs on a per-state type instead.

---

## 5. Constrain values at their edge, once

**Trigger:** a value has a rule narrower than its primitive type — a string with a max length, a required format (zip code, currency code), a number with a valid range (a quantity, a percentage) — enforced, if at all, at the point of use: a database column that truncates silently, an API handler that 400s, a form that validates — three different places, potentially three different rules.

**What goes wrong:** because the constraint isn't in the type, every consumer either re-derives it or skips it. A too-long string reaches a fixed-width column and gets silently truncated with no error anywhere. A quantity that should never go negative gets decremented past zero because the decrement operates on a bare `int`. Two independent checks on the same conceptual constraint (client-side vs. persistence-layer) disagree, producing the familiar bug where a value passes validation and fails, confusingly, three layers downstream.

**What to do:** define the constraint on its own type, checked once at construction — and critically, checked again on every *operation* that could push a value out of range, not just at the initial construction.

```
NonNegativeQty:
  private new(n)
  static of(n) -> NonNegativeQty | Failure:
    if n < 0: return Failure
    return new NonNegativeQty(n)

  decrement() -> NonNegativeQty | Failure:
    return NonNegativeQty.of(this.value - 1)   // re-validated on transition, not exempted
```

- **TypeScript:** branded numeric/string type plus factory, as Rule 1 — `type Percentage = number & {__brand:'Percentage'}` with `makePercentage(n): Percentage | undefined`.
- **Rust:** newtype with `TryFrom`/`new -> Result`; `String50` and `String100` are genuinely distinct types despite identical runtime representation, so a function expecting one rejects the other at compile time.
- **Java/C#/Python:** the same value-object-plus-validating-factory pattern as Rule 3; in Python, a frozen dataclass with `__post_init__` raising on violation, since there's no way to make an out-of-range instance simply unconstructable the way a sealed Java class or Rust newtype can.
- **When the storage layer genuinely can't express it** (the column really is `VARCHAR(50)` with no application type in front of it): decide once, explicitly, what happens on an over-length value — reject at the boundary or truncate — and make that decision in exactly one place, not as an emergent property of whichever layer happens to run first.

**How to check:** grep the diff for a length/range check (`.length >`, `< 0`, a format regex) appearing in more than one file against what is conceptually the same value. Look for a comment like `// should already be validated` near a use site — that "should" is unverified work the type isn't doing.

---

## 6. Let an awkward type shape interrogate the domain

**Trigger:** partway through modeling a type, the next required case doesn't fit cleanly — a two-case union grows a third requirement, then a fourth, and an N-way union starts looking obviously wrong, or several fields keep needing the exact same validation shape.

**What goes wrong (by omission):** nothing breaks visibly — this is a signal being ignored, not a bug. A team modeling the same requirement in loosely-typed terms (four optional fields, a bag of key-value pairs) never hits the moment where the shape visibly strains, so the missing concept — here, "a contact has a list of contact methods, each email-or-postal-or-phone" — never gets named, and the awkward four-field version ships instead of the one abstraction that would have made the "must have at least one" rule trivial (a non-empty list) instead of an ever-growing OR condition.

**What to do:** treat a combinatorially-expanding or repetitive type as a symptom that a concept is missing from the domain vocabulary, not a modeling problem to push through by adding another field or case. When one more variant "shouldn't be necessary," stop and ask what single entity would make the awkwardness disappear, and take the resulting question back to whoever owns the business rule rather than resolving it unilaterally — "does 'primary contact method' mean anything, and can two be equally primary?" is a requirements question the type shape surfaced, not an implementation detail.

This one isn't a per-language translation — it's a modeling habit that pays off regardless of enforcement strength. Even in a language with no sum types, going through the exercise surfaces the same missing concept; only what happens after differs.

**How to check:** not a grep — a review question. When a PR adds the Nth optional field or the Nth case to an already-large switch, ask whether a shared abstraction was considered and rejected for a stated reason, or simply not considered.

---

## 7. Know when the wrapper isn't worth it

**Trigger:** a primitive with no validation rule beyond its native type, used in exactly one place, whose meaning is fully carried by a nearby variable name and never at risk of transposition (a loop counter, a one-off local computation).

**What goes wrong if the earlier rules are applied unconditionally:** every value acquires a bespoke wrapper, function signatures fill with wrap/unwrap calls, and the boilerplate — construction, extraction, equality, serialization — grows faster than the bug count it prevents. The concrete tell: writing a small one-off script starts to feel like it needs the same ceremony as a production domain type, and the ceremony becomes a reason to avoid a change rather than a safety net for making one.

**What to do:** wrap a value when at least one is true — it carries a validation rule narrower than its primitive representation; it sits in a signature next to another value of the same primitive type (transposition risk); or it crosses a module/service/process boundary where the receiver needs a guarantee the sender already checked. A value with none of those stays a plain primitive. This is a per-value judgment call, not a blanket policy, and reasonably differs between a library's public surface (favor wrapping — callers are strangers) and its private internals (favor primitives — the whole file is visible at once).

**How to check:** a new wrapper type with a single call site and no validation logic beyond "is a string" is a candidate to inline back out. Conversely, a primitive parameter appearing unwrapped at a module or service boundary, especially beside a same-typed sibling, is a candidate to wrap.

---

## Placement

I'd split this, and lean against a third skill.

**The bulk — Rules 1, 2, 3, 5, 7 — extends `design-by-contract`, not `data-modeling`.** The reason is specific, not general "they're both about correctness": `design-by-contract` already states, in its own words, that a precondition established once at a boundary should never be re-checked downstream, that a class invariant should be written once at the type level, and that a filtering layer at the edge should convert messy input into something that satisfies internal preconditions. Rules 1, 3, and 5 here are that exact machinery with a compiler behind it instead of a documented convention — a validating factory *is* the filtering layer, a branded/newtype *is* the mechanism that makes "don't re-check your own precondition" enforceable rather than aspirational. Rule 2 is a direct strengthening of that skill's existing "Class invariants" section, which currently assumes a runtime-checked rule ("write it once, it holds whenever observable") — the type-level version goes one step further and makes the rule statically true by construction, so there's nothing left to check. Rule 7 is the cost caveat for all of it, and belongs wherever the enforcement machinery lives.

Concretely, I'd add a new section to `design-by-contract` — something like "Static preconditions: when the type can prove it" — placed right after "Preconditions" and before "Non-redundancy," covering the wrap/validate/constrain mechanism and cross-referencing the existing invariant and non-redundancy sections rather than restating them. That keeps the addition load-bearing instead of decorative: it answers "what do I do in a language that can enforce this?" for a skill that currently only covers the discipline for languages that can't.

**Rule 4 splits across both, and is the one place a single home would be wrong.** The *why* — a status/category implies that certain attributes are only meaningful at some of its values, and deciding that boundary is a modeling decision — is exactly what `data-modeling`'s `categories-and-change.md` reference already exists to cover (it's listed there for "status enums, type discriminators"); Rule 4 belongs as a code-level companion note in that reference, tying the schema-level "don't let a status imply meaning for columns that don't apply to it" back to the type-level sum-type mechanism. The *how* — sum types eliminating the null-check rather than a schema decision about column nullability — belongs in `design-by-contract`'s existing "Define errors out of existence" section, which already makes the closely related claim that "a nullable field every reader must guard" should be replaced by an unremarkable instance of the general case; Rule 4 is a second worked example of that same claim, not a new idea.

**Rule 6 is a small, standalone addition to `data-modeling`'s Purpose section**, not `design-by-contract`. That section already frames a schema as "a series of arbitrary, purpose-driven decisions... that get made implicitly" and the skill's whole premise is "surface those decisions while they're still cheap." Rule 6 generalizes that same framing from schemas to code-level types — the mechanism (an awkward shape is a symptom of a missing concept) is identical, just applied one layer up the stack. A two- or three-sentence addition there, not a section.

**No new skill.** Everything genuinely novel in this series is either (a) a concrete enforcement mechanism for a claim `design-by-contract` already makes in the abstract, or (b) a code-level instance of a surfacing habit `data-modeling` already teaches for schemas. Shipping a third skill would mean either re-explaining preconditions/invariants from scratch (redundant with `design-by-contract`) or re-explaining category boundaries from scratch (redundant with `data-modeling`), with the only new material — the specific TypeScript/Rust/Java/Python mechanics — better placed as reference material *inside* the skill whose section it operationalizes, where it'll actually get read at the point someone is deciding whether to write a runtime check or a type.
