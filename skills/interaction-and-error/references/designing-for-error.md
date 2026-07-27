# Designing for Error

Assume errors will occur. The question is never whether, only whether the system detects them, survives them, and lets someone recover.

Read [slips-and-mistakes](slips-and-mistakes.md) first if you haven't — the defences below are category-specific, and applying one to the wrong category produces a guard that looks protective and isn't.

## Constraints narrow the space before anything goes wrong

**Trigger.** Designing input, ordering, or any operation with invalid combinations.

**What goes wrong.** Four kinds of constraint restrict plausible actions, distinguished by where the restriction lives:

- **Physical** — the object permits only the right action (a connector that fits one way).
- **Cultural** — a learned, shared convention (red for stop). Conventions are local: the same gesture can mean opposite things in two places, and they collide when someone crosses that boundary.
- **Semantic** — meaning in the world independent of convention (a windscreen only makes sense in front of the rider). These can expire when technology changes what the signal needs to mean.
- **Logical** — derivable by elimination (a part left over tells you something's unassembled). Natural mapping is a special case: spatial correspondence is a logical relationship.

The caution that gets missed: **a physical constraint only helps if it's perceivable in advance.** A restriction that merely blocks the wrong action *after* it's attempted is worse than one that prevents the attempt — and a symmetric-looking connector wired asymmetrically doesn't block anything, it just fails or damages something once you commit.

**What to do.** Prefer constraints that are visible before the action over validation that fires after it. In software, that's disabling what can't apply, and structuring input so invalid combinations can't be expressed — rather than accepting anything and rejecting it downstream.

**Check.** Can the user tell an option is unavailable before choosing it, or only after? If only after, the constraint is a rejection, not a constraint.

## Forcing functions fix slips and make mistakes worse

**Trigger.** A flow where skipping or forgetting a step causes real damage.

**What goes wrong.** A forcing function engineers a situation so that failing one step blocks the next. Three variants:

- **Interlock** — forces correct sequencing; the next step is unreachable until the previous one completes.
- **Lock-in** — holds a state active until a condition is deliberately satisfied. The save-on-exit prompt is the familiar case, and it's worth noting it can invert: once its behaviour is fully internalised, people begin using it *as* the save mechanism. A friction device becomes a shortcut.
- **Lockout** — prevents entry into a dangerous state at all.

**The critical constraint on all three:** a forcing function is the right defence against a **slip**, where the plan was already correct and only execution or follow-through would have failed. It is the wrong defence against a **mistake** — it guarantees each step executes, so a wrong plan simply executes more reliably. Adding steps to a flow whose *intent* was wrong makes the damage more certain, not less.

**What to do.** Use forcing functions where the goal is reliably right and the failure is one of execution or memory. For the memory-lapse case specifically, reorder so the thing the user came for is withheld until the forgettable step is complete — people forget housekeeping, not goals.

**Check.** If the user's plan were wrong, would this forcing function stop them? If it would just make the wrong plan happen faster, it's aimed at the wrong category.

## Sensibility checks are the defence that works against mistakes

**Trigger.** Any input that becomes a consequential action — a quantity, an amount, a recipient, a scope.

**What goes wrong.** Systems execute exactly what was typed. A transfer a hundred times larger than any previous one, a dose a thousand times the norm, a bulk operation matching ten thousand records where the user expected ten — each is syntactically valid and semantically absurd, and nothing checks the second.

This is the defence that reaches mistakes precisely **because it doesn't depend on the user's belief about the action.** A confirmation asks the person who already has the wrong model to re-endorse it. A sensibility check evaluates the content against context and objects independently.

**What to do.** For consequential actions, check the value against what's plausible given history and context — not just against type and range. Surface the anomaly specifically ("this is 100× your largest previous transfer"), not as a generic confirmation.

**Check.** Is there any input value that would be accepted here that should obviously alarm someone? If the only validation is type and bounds, yes.

## Reversibility beats confirmation; irreversibility should cost

**Trigger.** Any destructive action.

**What goes wrong.** Confirmation is friction on the click. It does nothing about the reasoning that produced the click, and it decays into reflex on any routine path.

**What to do.** Make actions reversible wherever possible. Undo is the strongest general defence here because it works after the fact and therefore doesn't require the user to notice anything at decision time — which is exactly why it catches both slips and mistakes when nothing at the commit point does.

Where an action genuinely cannot be reversed, make it **harder to perform** rather than compensating with a louder confirmation: require typing the resource name, restrict it to a narrower surface, gate it behind an explicit elevation step. Difficulty proportional to irreversibility, rather than a uniform prompt on everything.

Also: treat unexpected input as an approximation of what the person meant rather than something to reject. Help complete the real intent — preserve what they entered, indicate what's wrong in place, let them correct it — instead of discarding the attempt and restarting them.

**Check.** For each destructive action: is it reversible? If not, is it harder to perform than the reversible ones nearby, or does it use the same prompt?

## There is no single root cause

**Trigger.** Reviewing an incident, a support escalation, or any "the user did X and everything broke."

**What goes wrong.** Real failures pass through multiple layers of defence, each with gaps, and require the gaps to align. There is essentially never one cause — only a chain where any single link holding would have prevented the outcome.

Investigations that go looking for blame stop at the first human action they find. That's the point where analysis should *begin*: keep asking why past the first plausible answer.

The technique isn't an algorithm, and it's worth knowing its limits — "why" is ambiguous, investigators stop at the edge of their own understanding, and the format still tempts a single-cause narrative when the truth is a conjunction.

**What to do.** Don't let an investigation terminate at "user error." Ask what made the error easy, and what made it undetectable, before concluding anything about the person.

The honest version of this claim is narrower than "never blame the user." Fatigue, intoxication, and deliberate rule-violation are real and individually attributable — which is why regulations govern them. But when a large share of failures in a system get filed as human error, that ratio is evidence the system reliably produces the failure regardless of who is operating it.

**Check.** Does the explanation end at a person's action? If so, at least one more "why" is available, and it's usually where the fixable cause is.

## Automation hands back control at the worst moment

**Trigger.** Anything that handles the routine case automatically and escalates the exceptional one — a retry layer, an autoscaler, a fallback path, an assisted workflow.

**What goes wrong.** Automation covers ordinary conditions and silently returns control to a human for the rare, hard case — the exact moment the human has been out of the loop longest and has least context on the current state.

Structurally this is a mode error at system scale: the system quietly changed which mode it was operating in, the indicator was easy to miss, and the operator continued acting on a model that had stopped being true. A vessel ran aground because navigation had silently fallen back from satellite positioning to dead reckoning days earlier, behind a small indicator nobody was watching.

**What to do.** Make silent degradation loud. Any fallback to a lesser mode should be as visible as an outage, because from the operator's model it is one. When automation escalates, hand over the state it accumulated, not just control.

**Check.** If your system silently degraded to a fallback path right now, how would anyone know? If the answer is a log line or a small badge, it's the same failure shape.
