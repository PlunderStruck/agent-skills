# Inheritance and Error Paths

The two places a contract gets broken without anyone at the call site being able to see it.

## Substitutability: weaken preconditions, strengthen postconditions

**Trigger.** Overriding a method, implementing an interface, or subclassing anything with a documented contract.

**Why this is the dangerous case.** A caller holds a reference typed as the base. It calls a method. At runtime, polymorphism may dispatch to a subclass override the caller has never heard of and cannot inspect.

If that override **demands more** than the base contract did, a call that was legal under the base contract now fails — and the caller did nothing wrong. If it **promises less**, the caller proceeds on a guarantee that no longer holds.

Either way the caller kept its side of the agreement and got cheated. That's precisely what substitutability is supposed to prevent. (This is the property usually cited today as the Liskov Substitution Principle; it also falls straight out of the contract metaphor as subcontracting.)

**The rule:**

| | Override may be | Never |
|---|---|---|
| **Precondition** | equal or **weaker** — accept everything the base accepted, optionally more | stronger |
| **Postcondition** | equal or **stronger** — guarantee everything the base guaranteed, optionally more | weaker |
| **Invariant** | adds to the ancestor's | replaces it |

**Make it structural rather than remembered.** Implement an override's condition as *base precondition OR new condition*, and *base postcondition AND new condition*, rather than replacing either wholesale. Then narrowing what's accepted or weakening what's promised is impossible by construction rather than caught by review.

### When a subclass genuinely needs a narrower precondition

Sometimes reality demands it — a bounded stack's `push` really can fail when full; a general stack's cannot.

Don't hardcode the narrower rule as a literal strengthening. **Expose the constraining condition as an overridable query on the base type** — a `full` check the base defines as permanently false, which the bounded subclass redefines against real capacity.

The documented precondition then reads *not full* at every level of the hierarchy. Its text never changes; only its concrete meaning specialises. Callers coded against the abstract precondition stay correct for every subclass.

**How to check a diff.** For any override: is every case the parent accepted still accepted? Does it still guarantee everything the parent promised? And does it depend on state invisible from the base's public interface — meaning a caller holding the base type could be surprised?

## Error paths: retry, or fail cleanly

**Trigger.** Writing a `catch`, `rescue`, or `except` block, or any recovery path.

**The anti-pattern**, and it is everywhere: the handler logs something, then **returns as though the operation succeeded**. The caller proceeds believing the postcondition holds. It doesn't. The real failure surfaces much later, far from its cause, as corrupted state or a wrong answer nobody can trace back.

This is a lie told across a module boundary, and it is functionally indistinguishable from a `goto` that skips the part where the work happened.

**A handler has exactly two honest options:**

**1. Retry.** Change something about the conditions that caused the failure — state, input, strategy, or simply waiting out a transient external condition — then attempt the *original* operation again from the start. If it eventually succeeds, the caller never needs to know anything went wrong, because the contract was ultimately fulfilled.

**2. Fail cleanly.** Give up, **restore the object to a state satisfying its invariant** even though the postcondition cannot be met, and propagate the failure so the caller decides what happens next.

What a handler must never do is exit through the error path while presenting the result as a success.

**The invariant clause in option 2 is the part that gets skipped**, and it matters more than the propagation. An error path that leaves invariant-violating state behind is worse than one that fails loudly, because the corruption resurfaces later as a different and much harder bug.

### The narrow exception

Genuine false alarms — a spurious external signal known to be harmless — can legitimately resume where they left off. This applies to a small class of external-signal cases and **never to assertion violations**. Even then, preventing the spurious signal is usually better than catching and ignoring it in routine after routine.

**How to check a diff.** For every catch block, trace what happens after it. Does execution definitely end in either a fresh attempt at the original operation, or a propagated failure plus a state that still satisfies the invariant?

If it ends by returning a default, logging and continuing, or falling through as though nothing happened — while the postcondition was never achieved — that's the anti-pattern.

## Why these two belong together

Both are cases where **you are on the hook for a promise made somewhere else.** An override inherits a contract it didn't write. An error handler is responsible for a postcondition it may not be able to deliver.

In both, the caller cannot see what you decided. That's what makes them worth a separate check rather than trusting the code to read correctly.
