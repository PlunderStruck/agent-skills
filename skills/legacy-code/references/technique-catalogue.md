# Technique Catalogue

Each entry is the cheapest move that removes one specific named obstacle. All of them trade a small, deliberate, temporary ugliness for a seam that makes surrounding behavior verifiable.

---

## Adding new behavior without testing the old code first

These four exist because forcing tangled code under test before you can add anything is often not worth it. Isolate the new logic instead.

### Sprout Method

**Use when** the change is a distinct chunk of new logic expressible as one call from inside an existing untested method.

**What you do.** Write the call at its insertion point, commented out, so you can see it in context. Pass whatever local state it needs as parameters. Develop the new method test-first, fully decoupled from its surroundings. Then enable the call. If the class can't be instantiated at all, make it a free function instead.

**Cost.** The old method and class stay exactly as untested as before, and the one-line wiring call remains unverified.

### Sprout Class

**Use when** Sprout Method isn't enough — the containing class's dependencies are too tangled to harness in reasonable time, or the new logic is a genuinely separate responsibility.

**What you do.** Same move, but the new code becomes a whole class, constructed with only the state it needs, built and tested independently, then wired in at one construction or call site.

**Cost.** Adds a class that can feel disconnected from the application's existing concepts and may never fully reconcile into an obvious home. Accepted because it's still safer than an invasive edit to unverifiable code.

### Wrap Method

**Use when** new behavior must run strictly before or after existing logic, without interleaving into it.

**What you do.** Either rename the old method and reintroduce its original name as a thin wrapper calling the new behavior plus the renamed original — transparent to callers — or leave the original alone and add a separately named method that sequences new-then-old for callers who opt in.

**Cost.** Forces you to invent a name for "the old code minus its old name," which is usually mediocre until further extraction is safe.

### Wrap Class

**Use when** behavior needs to wrap a method across many call sites, or you need to add behavior without growing a class that shouldn't get bigger.

**What you do.** Build a class implementing the same interface as the original, holding a reference to it, defaulting to plain delegation and overriding only the operations that need new behavior. Decorator-style, so wrappers compose. If the original has no interface yet, extract one first. For one or two sites, a narrower one-off wrapper is enough.

**Cost.** Chains of wrappers get hard to trace. A one-off wrapper that keeps accreting responsibilities is a deferred extraction wearing a disguise.

---

## Breaking dependencies in existing code

### Extract Interface

**Use when** a fake needs to stand in for a class dependency and faking the concrete class isn't practical.

**What you do.** Declare a new, initially empty abstraction. Make the real class satisfy it. Change the reference at the point of use to the abstraction. Then, driven *only* by what the compiler demands, add just the operations that call site actually uses — and write a fake behind it.

**Cost.** A new type that may correspond to no real domain concept at first. In languages where dispatch is non-virtual by default, moving a same-named method onto the abstraction can quietly change which implementation resolves elsewhere in a hierarchy.

### Parameterize Constructor / Parameterize Method

**Use when** a constructor or method builds a collaborator internally, and that inline construction blocks substitution.

**What you do.** Add an overload accepting the collaborator as a parameter. Keep the original signature as a thin wrapper that supplies the real object by default.

**Cost.** Widens the public surface with an extra overload, and gives the rest of the codebase a new path to couple to the collaborator's concrete type. Usually acceptable — this is one of the most frequently useful techniques here.

### Extract and Override Call

**Use when** a single call inside a method — to a static, a global, or another troublesome dependency — is the one thing blocking testing.

**What you do.** Wrap that call in a new method with the same signature, replace the call site with a call to the wrapper, then override the wrapper in a test-only subclass.

**Cost.** Cheap and safe, but doesn't scale past a handful of call sites for the same dependency. Beyond that, centralize instead.

### Subclass and Override Method

**Use when** one isolable piece of behavior — commonly object creation, or a call into an unreachable dependency — blocks testing.

**What you do.** Find the smallest set of methods whose override neutralizes the problem, loosen their visibility and make them overridable, then subclass for tests and override only those.

**Cost.** This is the foundational move underneath most of the others, so use restraint. Widening visibility on deliberately private methods is a real cost, and over-aggressive overriding means the test stops exercising the class's actual behavior.

### Introduce Instance Delegator

**Use when** the blocking dependency is a static or utility call with no object to substitute.

**What you do.** Add an instance method that forwards to the existing static call, then migrate call sites to go through an instance.

**Cost.** Leaves an interim mix of statics and thin pass-throughs until every caller migrates, and creates a second question: how does each site now obtain that instance?

### Break Out Method Object

**Use when** a method is long, entangled with instance state, and its class is too costly to instantiate for a test.

**What you do.** Create a class whose constructor takes a reference to the original object plus the method's original parameters as fields. Move the body verbatim into a run-style method. Expose whatever original-class members it still touches. Delegate the original method to an instance of the new class.

**Cost.** Loosens encapsulation on the source class and produces an odd interim shape. Accepted because former locals become fields, which can now be sensed freely.

### Expose Static Method

**Use when** a method uses no instance state but its class is very hard to instantiate.

**What you do.** Extract the body into a public static or free function; let the original delegate to it.

**Cost.** Makes something conceptually tied to an instance callable from anywhere. If the logic doesn't truly belong on this class, exposing it can entrench it there.

### Adapt Parameter

**Use when** a parameter's type is wide, external, or unmodifiable, and can't be narrowed with Extract Interface directly.

**What you do.** Define a small abstraction expressing only the subset of behavior actually needed. Write a production adapter wrapping the real parameter, and a fake behind the same abstraction. Change the method to accept it.

**Cost.** The one technique here that does **not** preserve the original signature, so every caller changes. And if the narrowed abstraction doesn't faithfully mirror the original's semantics, this introduces a genuine behavioral bug rather than just a testing convenience. Verify the mapping.

### Encapsulate Global References

**Use when** logic depends on several scattered globals or free functions and no single call-site fix covers all the uses.

**What you do.** Identify globals used together conceptually. Create a class to own them, keeping their original names to minimize churn. Let the compiler point out every now-broken reference and redirect each through the new owning object. Once centralized, apply a substitution technique to swap in a fake.

**Cost.** Deliberately ugly at first — one global object standing in for several globals is not a design improvement by itself, only a staging point that makes substitution possible.

### Pull Up Feature / Push Down Dependency

**Use when** a class mixes a few methods you need to test with an unrelated heavy dependency — a database call, a hardware handle.

**What you do.** Split along that line. Either move the testable methods up into a new abstract superclass and test through a lightweight subclass, or push the dependency-laden members down into a subclass and test the freed base directly.

**Cost.** Splits one class into two types purely for testability, which reads as arbitrary when the halves aren't conceptually related. Interim, pending a proper composition-based cleanup.

### Extract and Override Factory Method

**Use when** a constructor buries creation of a dependency you need to replace, in a language where a constructor's virtual calls dispatch to subclass overrides.

**What you do.** Move the creation into an overridable factory method; override it in a test subclass.

**Cost and caveat.** This doesn't work where base construction can't reach derived overrides — notably C++. There, either defer creation behind a lazily-initializing getter and override that, or, if the dependency is also used elsewhere in the constructor, add a distinctly named method that swaps the field after construction and call it only from test setup. That last variant weakens the invariant that a dependency is fixed for an object's lifetime, so confine it to tests.
