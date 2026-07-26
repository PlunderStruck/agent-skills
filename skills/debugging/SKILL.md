---
name: debugging
description: Find the actual cause instead of a plausible one — reproduce on demand before touching a fix, observe the failure mechanism rather than inferring it, binary-search the failure path, change one variable at a time, and prove a fix by making the bug come back. Use when something is broken, a test fails intermittently, a bug resists a first fix, or you are about to declare something fixed.
---

# Debugging

## Purpose

The failure mode this corrects is not ignorance. It's **fixation**: settling on an explanation before the evidence forces you to, then protecting it instead of testing it.

It shows up as skipping the docs because you already "know" the system, jumping to blame the newest code or someone else's component, bundling a speculative fix with other changes so a partial improvement feels like validation, and treating one clean rerun as proof.

Every rule below is procedural rather than a call to be more objective, because guesses — however well-reasoned — are still guesses, and the discipline is what catches you when one is wrong.

## Rules that apply without loading anything

**1. Reproduce it on demand before you touch a fix.** If you can't make it fail reliably, you cannot tell afterward whether you fixed it or got lucky. A bug that failed 1-in-10, "fixed," then passing 20 runs, looks solved — but a bug now failing 1-in-30 passes that same test.

**2. Quit thinking and look.** A plausible mechanism is not evidence. Instrument until you see the actual failure — "the write fired twice," not "the output is wrong." Guessing is fine for deciding *where to look next*; it is never a substitute for looking.

**3. Binary-search the path, don't jump to a suspect.** Probe halfway between a known-good point and the known-bad symptom, determine which half is now bad, repeat. Narrowing a hundred-way search takes about seven good cuts. This also structurally defeats "it's probably their component" — the probe doesn't care whose code it is.

**4. Change one thing at a time.** If a bundle fixes the symptom you know only that *something* mattered. And a speculative change that "didn't seem to help" is not neutral — it had some effect you don't know about, and it stays live corrupting your next test until you remove it.

**5. Work backward from the failure, not forward from the start.** There are always more correct things to verify than broken ones.

**6. Fix one bug before hunting the next.** Multiple simultaneous bugs mask each other, and a known bug left in place corrupts every subsequent probe.

**7. Check the plug.** When something seems impossible, re-verify what you assumed without checking: is the build you're testing the build you think it is, did the rebuild happen, is the config the one being loaded. "That can't happen" is a statement about your model, not about the event that was reported.

**8. Prove the fix by making the bug come back.** Reproduce, apply the fix, confirm it's gone — then **remove the fix and confirm the failure returns**, then reapply. Only that cycle proves your change is doing the work rather than some other variable that shifted alongside it.

## The reproduction step people skip

Rule 1 is where most debugging goes wrong, so it's worth spelling out.

**Drive the system into failure deliberately and repeatably**, from a known starting state, with the steps written down. If it's intermittent, hunt the uncontrolled variable — timing, load, input shape, environment — rather than accepting randomness.

**Amplify the conditions, don't simulate the mechanism.** Raising load or tightening timing to make the failure more frequent is legitimate. Writing a synthetic reproduction that never touches the real code path is not — it can pass while the actual bug sits untouched.

**The standard is 100%**, not "usually." Anything less is a statistic, and statistics are exactly what let you fool yourself later about both the cause and the fix.

## Observing rather than inferring

The gap between symptom and mechanism is where most wasted time lives. "The output is wrong" is a symptom. "The value was written twice" is a mechanism.

- **Instrument at the lowest level that shows the mechanism** — logs, breakpoints, traces, a debug build.
- **Feed the system a recognisable pattern** rather than production data when real data is too noisy to read at a glance. Corruption becomes visually obvious instead of requiring careful diffing.
- **Drop a guess the moment instrumentation doesn't confirm it.** Defending it is the fixation this skill exists to interrupt.

**When to stop looking and start reading code:** when the observed failure has narrowed the space to something small enough to reason about — not before, or you'll misattribute the cause; not much after, or you're tracing code already proven fine.

## Keeping a trail

Log every attempt, in order, with its outcome — including attempts that appeared to do nothing, because "no observed effect" is data.

Memory discards exactly the details that later turn out to matter, because at the time you can't tell which ones are load-bearing. Without a trail you can't reliably compare a good run to a bad one, can't tell a collaborator what's already been ruled out, and will walk the same dead end twice.

Record conditions precisely enough to match against system logs later — "failed for four seconds starting at 14:05:23," not "it broke around lunchtime."

## Getting unstuck

When new evidence keeps almost-but-not-quite fitting your theory, and you keep reinterpreting it to fit rather than questioning the theory — that's the rut, and it's invisible from inside.

**Report symptoms and observations, not your theory.** Leading with your conclusion drags the other person into your rut and buries the details they'd need for an independent read. Mention things you noticed but can't explain; you don't have to be sure something is relevant to report it.

Writing the problem out from scratch, as if for someone else, forces the same reorganisation even with nobody to send it to.

## Symptom versus cause

A fix that stops the symptom without explaining the mechanism manufactures the belief the problem is closed. A part replaced without understanding why it failed buys time until an equivalent part fails the same way.

Once the immediate fix works, ask **what allowed this to happen at all**, and whether that condition will produce a related failure later under a different name.

**The completion standard:** you can state the mechanism by which your change closes the gap that caused the failure — not just that the tests came back clean.

## Boundary with neighbouring skills

- **`scip-root-cause`** — why a *family* of related bugs keeps recurring, which is a design-flaw diagnosis across many incidents.
- **`debugging`** — finding the cause of the one in front of you.
- **`incident-review`** — what to conclude, and how to judge decisions, after an outage is over.
