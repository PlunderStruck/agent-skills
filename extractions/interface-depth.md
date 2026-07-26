# Extraction: interface-depth

**Source:** A Philosophy of Software Design — John Ousterhout

> Operational rules distilled from the source, written in our own words. Denser than the published skill; the place detail survives for re-distillation.

---

# Ousterhout Extraction: What's New Beyond `decomposition`

## Part 1 — New Material

### A. Complexity has three distinguishable symptoms, not one

**Trigger.** A design argument goes in circles because "this is too complex" means something different to each person in the room.

**What goes wrong.** Complexity gets treated as one undifferentiated bad feeling instead of three separable costs, so fixes get aimed at the wrong one. Fewer lines can make cognitive load worse by hiding logic behind indirection. A shorter diff can still touch six files.

**What to do.** Name which symptom is actually in play before proposing a fix. *Change amplification*: does a small conceptual change require edits in many places — fix by consolidating the decision to one place. *Cognitive load*: how much must a developer hold in mind to make one safe change — fix by hiding irrelevant detail, which isn't the same as writing less code. *Unknown unknowns*: is it even possible to know what else needs to change — the worst of the three, because nothing tells you it exists until a change breaks something far away. Underneath all three sit two causes: dependencies (which force multi-place coordination) and obscurity (which hides that a dependency exists at all).

**How to check.** When someone calls a design "complex," ask which of the three they mean. "It's complex" isn't a usable complaint until it resolves to one of these.

### B. Deep vs. shallow modules, and classitis

**Trigger.** Deciding whether a class, function, or service's boundary earns its keep — at design time, or when reviewing something that already looks suspiciously thin.

**What goes wrong.** "Smaller is better" gets applied to interfaces the way it gets applied to function length, producing many classes each with a small, tidy-looking interface — and the system as a whole gets *harder* to use, because a caller now has to learn N small interfaces instead of one deep one. A module with a wide interface and little behind it (a getter, a one-line delegator, a class whose whole body is `data.put(key, null)`) costs a name, a signature, and a slot in every caller's memory, and returns almost nothing for that cost.

**What to do.** Judge a module by the ratio of functionality it provides to the size of interface required to reach it. Unix's five I/O system calls cover disk layout, permissions, scheduling, and caching — that's depth. A linked-list class whose insert is three lines is shallow no matter how clean its method names are, because using it isn't meaningfully simpler than reimplementing it. When tempted to add a class, ask what it hides that its caller doesn't already know.

**How to check.** Would fully documenting this module's interface take about as many words as documenting its implementation? If so, it's shallow regardless of how organized it looks. Watch for "split anything over N lines" as a house rule — applied mechanically, it manufactures shallow pieces at scale (this is *classitis*: many small classes accumulate into more total interface complexity than one deep one, even though each piece looks simple alone).

*This overlaps with decomposition's existing point that a boundary needs a reason — deep-vs-shallow just gives that judgment a ratio to measure instead of a gut feeling.*

### C. Information leakage — mostly agreement, one refinement

**Trigger.** The same fact — a file format, a protocol detail, a unit convention — turns out to be understood by two modules that never call each other's interface to learn it.

This is the mechanism decomposition already routes around: "group by what changes together, not sequence" and Ousterhout's own diagnosis — *temporal decomposition*, splitting by *when* something happens rather than *what it needs to know* — describe the same failure from two directions (decomposition's editor edit/refresh split and Ousterhout's read-then-parse HTTP split are structurally identical mistakes). The one genuinely new piece worth carrying over: leakage doesn't require a shared interface to be real. Two classes can each embed knowledge of a format with neither exposing it publicly — leakage invisible in either file alone, surfaced only when the shared fact changes and both need editing. **The check**: pick a design fact and ask how many modules would need to change if it changed, even ones that never call each other.

### D. Different layer, different abstraction

**Trigger.** A method body is almost entirely "call the same-named method on the thing I wrap," or a variable threads through four signatures where only the innermost one reads it.

**What goes wrong.** Adjacent layers end up presenting the *same* abstraction instead of different ones. A pass-through method (a wrapper's `insert` that just calls the wrapped object's `insert`) adds a name and a dependency for zero new functionality — if the wrapped signature changes, the wrapper must change to match, and nothing was gained. Pass-through variables do the same thing to arguments: a certificate path threaded through three methods that never look at it, just so a fourth one can. Decorators are the systematic version — wrapping to add one feature at a time until opening a file needs three constructors.

**What to do.** For a pass-through method, decide which class actually owns the responsibility and either expose the lower layer directly, redistribute functionality, or merge the classes — don't leave a stub on both sides. For pass-through variables, resist the two tempting non-fixes: a "shared object" that's itself just a pass-through target one level up, and a global that breaks multiple instances and testability. Reach instead for a context object — one struct carrying an instance's global state, passed explicitly into constructors so the variable stops appearing in every intermediate signature. A context is not a clean solution; without discipline it decays into an unlabeled grab-bag, so keep its fields immutable and documented.

**How to check.** Could a suspected pass-through method be deleted, with the caller invoking the wrapped method directly, with no loss of meaning? If yes, it isn't pulling weight. For a context field, ask whether it's the same fact at every call site or several unrelated facts sharing a name.

### E. Define errors — and special cases — out of existence

**Trigger.** About to throw or return an error for a condition where it's unclear what the caller should even do about it, or writing a branch to guard a "doesn't exist yet" state.

**What goes wrong.** The reflex "detecting more is safer" makes operations grow an exception for every slightly-off input — deleting a variable that was never set, indexing past a string's end. Each exception joins the interface, and every caller now needs code to survive it whether or not it can do anything useful in response. The handling code is disproportionately hard to get right (you can't easily manufacture a disk failure in a test) and rarely runs, so it's rarely exercised before it matters.

**What to do.** Before writing a handler, ask whether the contract can be redefined so the condition simply isn't exceptional. "Delete this variable" can mean "ensure it's gone" instead of "fail if it wasn't there" — after the redefinition there's nothing to catch. An out-of-range substring request can clamp to the string's bounds instead of throwing. A selection that doesn't exist yet can be an empty range at a valid position instead of a nullable field guarded everywhere it's read. Where the condition genuinely can't be designed away, mask it at the lowest level that can actually act on it (retry inside the transport, not in every caller), or aggregate distinct exceptions into one handler where the response is the same regardless of which fired (one error-response generator, not one catch block per missing parameter).

**How to check.** For each exception a module can throw: would a slightly different definition of the operation make this case ordinary? For a defaulted or nullable field: can the special case be represented as an unremarkable instance of the general case, rather than a flag every caller must test?

### F. Tactical vs. strategic programming

**Trigger.** A deadline is close and the fix that ships today is visibly worse-shaped than the fix that ships in three more days.

**What goes wrong.** Each shortcut looks cheap alone, and the case for taking it is locally correct every time — it's the accumulation that costs, so the damage stays invisible until a project is well into it, at which point "clean it up" is a multi-month ask nobody approves. A team that consistently takes the faster-but-worse option grows a contributor whose entire output is exactly that: fast, working, unmaintained by the time anyone else touches it — and everyone else absorbs the bill later while it looks, from outside, like they're the slow ones.

**What to do.** Treat design quality as a running expense, not an occasional project: spend roughly a tenth to a fifth of total time on things that don't ship today's feature — picking a cleaner shape before typing it, fixing a design problem the moment it's noticed rather than filing it, writing down reasoning that isn't in the code. The payoff isn't distant — teams holding this discipline report being faster within months, because what they're extending stays cheap to extend.

**How to check.** For a shortcut taken under pressure: is there a concrete point at which it gets paid back — next sprint, next release — or is the honest answer "never, we'll live with it"? The second answer is the one to worry about.

### G. Comments as a design tool — write the interface comment first

**Trigger.** About to write a new class or function, and the shape doesn't feel settled enough yet to write the doc comment.

**What goes wrong.** A comment written after the code is "done" describes what the code happens to do, not what a caller needs to know — and by then the designer has mentally moved on, so it gets minimum-viable effort. Worse, a design flaw successfully hides behind code that already compiles and passes tests.

**What to do.** Write the interface comment — behavior, each argument, return value, side effects, exceptions, preconditions — *before* the body. This forces the abstraction into a form a caller can rely on without reading the implementation, which is a different and harder exercise than making the code work. If the comment can't stay short without omitting something a caller needs, or can't stay complete without describing internals a caller shouldn't need, that's a signal about the design, delivered before anything is built on top of it.

**How to check.** Read the finished comment back: would it need to mention how the thing works internally to be complete? If yes, the boundary is shallow — the comment is reporting the same problem a name that won't come would, just at paragraph length instead of word length.

### H. Designing it twice

**Trigger.** One idea for how to shape an interface or split a module, it seems reasonable, and there's pressure to just start building it.

**What goes wrong.** A first idea is rarely challenged, because nothing exists to contrast it against — its weaknesses only surface next to an alternative that handles the same case differently. Committing early means the weaknesses appear mid-implementation, when reversing is expensive, instead of on paper, where it's nearly free.

**What to do.** Before the real implementation, sketch two or three alternatives that are genuinely different from each other, not variations on one idea — even one you're confident will be bad. List concrete pros and cons: ease of use for the caller, how general-purpose it is, what it costs at runtime. If none is attractive, that's diagnostic too — the shared weakness across all of them usually points at the design that's actually missing. This costs hours against days or weeks of implementation.

**How to check.** Can a second design that was seriously considered and rejected be named, with a specific reason? "I only had one idea" isn't evidence the first idea was right — it's evidence it hasn't been tested against anything.

---

## Part 2 — Placement

**Headline recommendation: ship a new `interface-design` skill**, and extend `decomposition` with two of the eight clusters. The two skills answer different questions asked at different moments: `decomposition` asks *should a seam exist here, and where* — a question you can only ask while something is still one piece or several. `interface-design` asks *given a boundary that exists or is proposed, what should cross it* — a question that applies equally to code that's never been touched, to a five-year-old public API under review, and to a class you're two minutes from writing. Cramming the second question into `decomposition` would blur a skill whose current description is entirely about seam placement; cramming the first into a new skill would duplicate territory `decomposition` already owns cleanly.

**→ New skill, `interface-design`:** clusters B, D, E, and G, framed by A.
- **A (complexity defined)** becomes the opening "why," the same role `decomposition`'s "a boundary needs a reason" plays for that skill — depth is the direct embodiment of the change-amplification/cognitive-load triad, so it earns the frame rather than a standalone section.
- **B (deep vs. shallow, classitis)** is the load-bearing center: the depth ratio is the single lens that also explains D and E.
- **C's refinement (back-door leakage)** doesn't need its own section anywhere — a two-line cross-reference into `decomposition`'s existing information-leakage material, plus the one added check, is enough; duplicating the full concept would be exactly the redundant-restatement this exercise is trying to avoid.
- **D (different layer, different abstraction)** and **E (define errors out of existence)** are both about what an interface should hide once it exists — pure interface-shape questions decomposition currently has no vocabulary for.
- **G (comments-as-design-tool)** belongs here because "write the interface comment first" is literally a technique for measuring depth before you've built anything to measure — it operationalizes B. It's a close sibling of `decomposition`'s existing "name it in one word or the boundary is wrong," just one level up (a paragraph-length check instead of a word-length one); a one-line cross-reference from `decomposition`'s naming reference into `interface-design` covers that relationship without merging the two skills.

This is coherent enough to be a real skill rather than a thin one: it has one question, one governing metric (interface size vs. functionality hidden), and four concrete techniques that all reduce to applying that metric (avoid shallow decomposition, avoid layer duplication, avoid unnecessary error surface, use the comment as the depth gauge).

**→ Extend `decomposition`:** clusters F and H.
- **F (tactical vs. strategic)** is the weakest fit of the eight — it's a general engineering-discipline argument, not a boundary or interface question specifically. But `decomposition` already stakes out process-discipline territory in its "Working iteratively" section (throwaway prototypes, changing little between test cycles), and F supplies the economic argument for *why* that discipline pays off plus a concrete failure mode to name (the tactical tornado) — close enough to extend rather than orphan, and too small on its own to justify a new skill.
- **H (designing it twice)** is the direct generative counterpart to `decomposition`'s existing verification test ("trace one plausible future requirement" — you need something to trace before you can test it). Ousterhout applies it to interface choice, but he's explicit that it applies equally to choosing where to split a system into major modules, which keeps it inside `decomposition`'s stated job rather than `interface-design`'s narrower one. `interface-design` can cross-reference it as "when shaping an interface, generate alternatives per `decomposition`'s design-it-twice check" rather than duplicating it.

**Rejected alternative:** a single skill covering all eight. F and H are process/discipline; B, D, E, G are shape-of-interface. Bundling them under one description would be the two-jobs-one-skill problem this task explicitly warned against — a user asking "how should I shape this API" and a user asking "should I take this shortcut under deadline pressure" are doing different work, even though both trace back to the same book.
