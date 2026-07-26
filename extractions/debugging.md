# Extraction: debugging

**Source:** Debugging: The 9 Indispensable Rules — David Agans

> Operational rules distilled from the source, written in our own words. Denser than the published skill; the place detail survives for re-distillation.

---

This reads well and covers all nine rules with the requested structure, weighted toward the failure modes specified (Rules 2, 3, 5, and 9 get the deepest treatment), covers sub-techniques (binary search, one-variable-at-a-time, audit trail detail, stimulate-vs-simulate), the "when to stop looking vs. start reading" guidance (in Rule 3), and the psychology of getting stuck (Rule 8, plus fixation threads through 3, 4, and 7). It's written entirely in my own words, no passages reproduced from the source. Returning the final document now.

# Debugging Discipline: Operational Rules for Agents

## 1. Understand the System — build the model before you touch anything

**Trigger:** Unfamiliar code, an unfamiliar failure, or a library you've "used before." The tempting move is to start reading the failing code immediately, or pattern-match against a similar bug you've fixed before, and start editing.

**Why the wrong move fails:** Without a working model of what the system is supposed to do, you have no baseline to compare against — you can't tell whether a value is corrupt or intentional, a path dead or load-bearing, an error expected or catastrophic. Partial understanding is more dangerous than none, because it manufactures false confidence: once you feel you "know" the system, you stop looking things up, and the one paragraph of the spec you skipped is exactly where the bug lives.

**What to do:** Read the actual interface contract — doc, type signature, comment, the commit that introduced the behavior — before forming a theory, not after getting stuck. Do this even for code you wrote yourself; authors leave "bug here?" comments they never resolve. Establish what "normal" looks like at each boundary so abnormal actually stands out instead of blending into everything else you don't understand. Map which module owns which responsibility, so once the failure is localized, you already know what that implicates.

**How to know it worked:** You can state, before looking at anything, what correct behavior should look like at a given checkpoint — specific enough to be falsified by a single observation, not vague enough to fit any outcome.

## 2. Make It Fail — reproduce on demand before you touch a fix

**Trigger:** A bug report describes something that happened once, or intermittently. The tempting move is to read the code, form a theory, and patch based on it without ever seeing the failure yourself.

**Why the wrong move fails:** If you can't make the system fail on command, you can't look at the failure directly, can't tell which conditions actually matter versus which are coincidence, and — critically — can't prove afterward that anything you changed made a difference. A fix applied against a failure you never reproduced is a guess wearing a lab coat. A system failing 1-in-10 times, "fixed," then run 20 times clean, looks solved — but a system now failing 1-in-30 would pass that same test. Without a reliable trigger, "it stopped failing," "I fixed it," and "I got lucky" are indistinguishable.

**What to do:** Before touching a fix, drive the system into the failure state deliberately and repeatably, from a known starting state, with the steps written down. If intermittent, hunt for the uncontrolled variable (timing, load, input shape, environment) instead of accepting randomness; amplify suspected conditions to raise the failure rate, but stimulate the *conditions* — don't simulate the *failure mechanism* with a synthetic repro that never touches the real code path, since a fabricated repro can "pass" while the real bug sits untouched. If still stuck, log enough state on every run to identify failing runs after the fact and treat them as reproduced.

**How to know it worked:** You have a sequence that fails 100% of the time it's run — not "usually." Anything short of that is a statistic, and statistics are exactly what let you fool yourself later, both about the cause and about whether you fixed it.

## 3. Quit Thinking and Look — observe the failure directly, don't infer it

The rule that most directly attacks theorizing about the cause instead of observing what the system is doing, and the one most worth over-weighting.

**Trigger:** You have a plausible mechanism in mind — "it's probably a race," "the cache must be stale," "that library is flaky." Thinking is fast and satisfying; instrumenting is slow and tedious. The tempting move is to act on the theory directly: rewrite the suspect section, add a retry, "harden" the timing.

**Why the wrong move fails:** A plausible theory is not evidence. The space of things that could produce a given symptom is always larger than what one person can enumerate by reasoning alone, so a guess — however well-formed — is disproportionately likely to be wrong. Wrong fixes cost time twice: they don't fix the bug, and they can mask it by shifting timing or state just enough to hide the failure temporarily, or stack a second bug on the first. Engineers who spend weeks rebuilding a board purely from inference off a marginal timing spec can finish with the original bug still present, because the actual mechanism was never once observed before the rebuild started — a later pass that simply probes the real signal finds it in an evening.

**What to do:** Before changing anything, look at the lowest level that shows the actual mechanism of failure, not just its downstream symptom — "the output is wrong" is a symptom, "the write fired twice" is the failure. Add instrumentation (logs, breakpoints, traces, a debug build) instead of reasoning about what the code "should" be doing. Feed the system a known, easy-to-recognize pattern instead of noisy production data when the real data is too irregular to read at a glance, so corruption is visually obvious instead of requiring careful diffing. Guessing is fine, but only to decide *where to look next* — never as a substitute for looking. If instrumentation doesn't confirm a guess, drop the guess immediately instead of defending it.

**When to stop looking and start reasoning again:** Keep looking until the observed failure narrows the space of possible causes to something small enough to reason about directly — not before (you'll misattribute the cause), and not much past that point (you'll waste time tracing code already proven fine). The rule isn't "look forever" — it's: look until the symptom pins the failure to one small, specific piece of the system, then read *that* code closely.

**How to know it worked:** You saw the actual mechanism — the specific byte, call, pointer, or condition — not an inference sitting two or three layers removed from it. You can describe what happened, not just what must have happened.

## 4. Divide and Conquer — narrow with successive approximation, not with a hunch

**Trigger:** A large surface area (a whole service, a whole pipeline) and a bug somewhere in it. The tempting move is to jump straight to the piece you suspect — often the newest code, or someone else's component — and start reading there.

**Why the wrong move fails:** A single jump to a suspect is a bet, not a search strategy, and it fails silently — if you're wrong, you just wander the wrong region without knowing it. Binary search over a failure path is dramatically cheaper than linear inspection or hunches: narrowing a hundred-way search to the answer takes about seven good cuts, not fifty guesses. It also structurally corrects "blame the other component": successive approximation doesn't care whose code it is, only which side of each probe point is good and which is bad, so it can't be derailed by territorial assumptions.

**What to do:** Treat the system as a pipeline with a good (upstream) end and a bad (downstream) end. Probe roughly halfway between them, determine which side is now bad, and repeat inside that half, driven by observed status at each point, not intuition. Start from where the badness is visible and work backward, rather than starting at the beginning and confirming everything is fine until you stumble on the break — there are always more correct things to verify than broken ones. Once you find one bug, fix it immediately before searching for others: multiple simultaneous bugs mask each other, and a known bug left in place corrupts every later probe.

**Where the "first plausible explanation" trap gets caught:** "which side of the cut am I on" forces continued probing even after a theory shows up — if the guess is wrong, the next probe reveals it, because you aren't free to stop just because a story you like appeared.

**How to know it worked:** Each probe genuinely halved the remaining suspect region. You can point to the specific good reading on one side and bad reading on the other that bracket the actual fault — not to a component you decided to blame and then confirmed by reading its code sympathetically.

## 5. Change One Thing at a Time — isolate causation, don't bundle changes

**Trigger:** Deep in a session, you notice two or three things that look off at once. The tempting move: fix the timing *and* add the missing null check *and* bump the retry count, then rerun.

**Why the wrong move fails:** If the bundle fixes the symptom, you know only that *something* in it mattered — not which part, and not whether the change you ship is the one doing the work or just riding along. A speculative change that "didn't seem to help" is not neutral by default; it already had *some* effect you don't know about, and it stays live, corrupting your next test, until explicitly removed. This is how credit gets misattributed: an engineer adds a speculative fix that doesn't visibly help and leaves it in — later, someone else finds and fixes the real bug, but the system is still broken, because the first, abandoned change is still corrupting things a second, different way.

**What to do:** Hold every variable constant except the one under test — same build, same input, same machine, same day if possible; even superficial-seeming differences between a working and failing run can be the actual signal, so don't manufacture new ones on top of the real one. When a change doesn't produce the expected effect, revert it immediately rather than leaving it "just in case," and don't stack a second speculative change on an unconfirmed first one. When comparing a working case to a failing one, diff them directly rather than from memory.

**How to know it worked:** You can name the single variable that changed between the failing run and the passing run, with every other variable provably identical. If you can't name that variable, you don't know what fixed it — only that the failure stopped.

## 6. Keep an Audit Trail — write down what you tried, in order, with results

**Trigger:** Mid-session, a step feels too trivial to record ("I just restarted it") — logging it feels like overhead when you're trying to move fast.

**Why the wrong move fails:** Memory silently discards exactly the details that later turn out to matter, because at the time you can't yet tell which details are load-bearing. Without a trail, you can't reliably compare a good run to a bad run (rule 5 depends on this), can't tell a collaborator or your future self what's already been ruled out (so the same dead end gets walked twice), and can't later prove what sequence actually produced or fixed the failure.

**What to do:** Log every attempt, its exact order, and its exact outcome, including attempts that seemed to do nothing — "no observed effect" is itself data. Correlate symptoms with timestamps and conditions precisely enough to match against system logs later ("failed for four seconds starting at 14:05:23," not "made a bad noise"). Keep the trail somewhere searchable, not only in memory.

**How to know it worked:** Someone else — or you, a week later — can reconstruct exactly what was tried, in what order, and why each attempt was ruled in or out, without asking you to remember it.

## 7. Check the Plug — question the assumption you're not questioning

**Trigger:** A failure looks bizarre ("but that can't happen"), or you're stuck after exhausting the obvious suspects in your own code. The tempting move is to escalate to something expensive — a rewrite, a redesign, blaming a colleague's module — rather than re-examine the boundary conditions you assumed without checking: power, initialization, configuration, whether you're even running the build you think you're running.

**Why the wrong move fails:** Foundational assumptions get skipped precisely because they're too basic to seem worth checking, which is exactly why bugs hide there longest — an underpowered source can starve a downstream system for months while investigation focuses elsewhere, because "the source is fine" was never on anyone's checklist. Instrument problems compound this: a stale build or a misdocumented library function produces confidently wrong readings that look exactly like real evidence, sending you chasing a bug that exists only in your tooling. This is also where "blame the other component" lives: refusing to check your own foundational assumptions (Is my code even running? Did the rebuild happen?) while pointing at someone else's module is the same failure, just aimed outward.

**What to do:** When something seems inexplicable, explicitly re-verify what you assumed was fine without checking — power, connection, initialization, configuration, whether the artifact under test is the one you think it is. Test your instruments before trusting their readings. Treat "that can't happen" as a signal to keep looking, not a reason to stop — the sentence is true about the theory in your head, not about the event that was actually reported.

**How to know it worked:** The explanation you land on accounts for why the "impossible" thing looked impossible in the first place — not just a new guess substituted for the old one.

## 8. Get a Fresh View — counter your own fixation

The direct countermeasure to stopping at the first plausible explanation, and to getting stuck generally.

**Trigger:** You've been on the same failure a while, you're attached to a theory (often because you already tried to fix it that way), and new evidence keeps not quite fitting — but you reinterpret it to fit anyway instead of questioning the theory.

**Why the wrong move fails:** Everyone develops blind spots specific to their own mental model, and once a theory is in place, new data tends to get bent to fit it rather than used to test it. Working alone inside that rut, you lose the ability to notice you're in it — the bias distorting your reading of the evidence is invisible from inside. Pride compounds this: asking for a second look can feel like admitting you should have found it yourself, so the ask gets delayed well past the point where it would have been cheap.

**What to do:** Bring in another perspective — for a different viewpoint, expertise you lack, or relevant experience — before sinking excessive time defending a theory that keeps almost-but-not-quite fitting. Report symptoms and observations, not your theory: leading with your theory drags the other party into your rut and buries the details they'd need for an independent read. It's fine to mention things you noticed but can't explain — you don't have to be sure something is relevant to report it. Even alone, writing the problem out from scratch as if for another person forces the same reorganization that breaks fixation.

**How to know it worked:** The fresh input either changes your theory or gives you a specific new thing to check — not just reassurance you're on the right track. If a second opinion just restates your own theory back at you, check whether you actually led with symptoms or quietly led with your conclusion.

## 9. If You Didn't Fix It, It Ain't Fixed — confirm the fix, confirm the cause, fix root not symptom

The rule that directly attacks declaring victory without confirming the change is what caused the improvement.

**Trigger:** You made a change, reran the test, and the failure didn't recur. The tempting move: call it fixed and move on.

**Why the wrong move fails:** "It didn't fail this time" and "it's fixed" are not the same claim unless you can rule out that the failure was intermittent, that some unrelated factor happened to differ this run, or that you're testing under conditions the bug never depended on in the first place — a trap that produces false confidence right up until the original conditions recur in production. Fixing a symptom without finding the cause manufactures the belief the problem is closed: a part replaced without understanding why it failed just buys time until an equivalent part fails the same way, and a bug "fixed" by leaving an unrelated speculative change in place (rule 5) resurfaces the moment that change is later removed.

**What to do:** Reproduce the original failure using the trigger from rule 2. Apply the fix, confirm the failure is gone. Then — the step almost everyone skips — remove the fix and confirm the failure comes back. Reapply it and confirm it goes away again. Only that full cycle (broken → fixed → broken → fixed, with nothing changing but the fix itself) proves the fix is doing the work, rather than some other variable that shifted alongside it. Once the immediate fix works, also ask what allowed the failure to occur at all, and whether that underlying condition will produce a related failure later under a different name — wiping up a leak without tightening the fitting, or replacing a part without addressing why it was stressed, both guarantee a recurrence that looks like a "new" bug.

**How to know it worked:** You've watched the specific failure cycle on command — broken with the fix removed, fixed with it restored — using the same reproducible trigger, not a single clean rerun. And you can state the mechanism by which your change closes the gap that caused the original failure, not just that the numbers came back clean.

---

## The common thread: fixation is the failure mode underneath all of these

Across every rule, the actual enemy is the same: settling on an explanation before the evidence forces you to, then unconsciously protecting that explanation instead of testing it. It shows up as skipping documentation because you already "know" the system, jumping to a suspect component because it's convenient to blame, bundling a speculative fix with other changes so a partial improvement feels like validation, and treating a single non-recurrence as proof. The countermeasure is procedural, not a matter of trying harder to be objective: reproduce before touching anything, observe the mechanism before theorizing about it, change and test exactly one variable, write down what happened instead of trusting memory, and prove causation by making the bug come back on demand before claiming to have banished it. None of this depends on being smart enough to guess right the first time — that is the point. The discipline exists because guesses, however well-reasoned, are still guesses, and the rules are what catch you when one is wrong.
