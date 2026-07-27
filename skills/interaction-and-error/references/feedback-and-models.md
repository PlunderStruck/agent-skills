# Feedback and Conceptual Models

Everything here narrows the **Gulf of Evaluation** — the effort required to work out what just happened and whether it matched the intent. A perfectly discoverable control with unreadable consequences still fails.

## Feedback has two axes, and most code checks one

**Trigger.** Any action whose result isn't instantaneous and self-evident — a save, an async request, a background job, a state change, a bulk operation.

**What goes wrong.** Teams check *whether* feedback exists. The two axes that actually matter are **timing** and **quality**, and failure on either is a distinct defect from absence.

*Timing.* Delay past roughly a tenth of a second reads as unresponsive. The user repeats the action — and if the first invocation did register, the repeat fires a second, unintended effect. A slow endpoint with no immediate acknowledgement doesn't just feel bad; it manufactures duplicate operations.

*Quality.* "Bad feedback" fails in three specific ways:

- **Too little** — something happened, but not what, or not to which object. A generic success toast after a bulk operation tells you the request returned, not what it did.
- **Too much** — a stream of correct, routine notifications trains people to mute the channel entirely, including the one alert that mattered. Every non-essential confirmation spends credibility that the important message later needs.
- **Wrong channel, or uninterpretable encoding** — a signal the user can't attribute to the right source, or many states crammed into one indicator via a private code nobody learns. Worse when several components on the same surface use incompatible encodings.

**What to do.** Acknowledge immediately, even before the result is known — the acknowledgement and the result are two separate messages, and conflating them is what produces the dead interval. Name what happened *and to what*. Rank feedback so routine confirmations don't compete with consequential ones.

Also worth noting: removing incidental feedback during a redesign silently strips confidence signals people relied on without articulating. A "cleaner" flow that removes intermediate progress cues often reads as broken rather than clean.

**Check.** Does something visibly change within ~100ms of the action, independent of the operation completing? Does the eventual message name the object affected? Would a user who received ten of these today still read the eleventh?

## Feedforward: what *can* happen, before committing

**Trigger.** An action whose consequences aren't apparent until it's done — especially a destructive or bulk one.

**What goes wrong.** Feedback answers "what happened." It arrives too late to prevent anything. The counterpart question — "what *will* happen if I do this?" — often goes unanswered entirely, so the user's only route to knowing is to try it.

**What to do.** Where an action's scope isn't obvious, state it before the commit point rather than after: what will be affected, how many, and whether it can be undone. This is not the same as a confirmation dialog — it's supplying the information the user needs to *form* a correct plan, which is the one thing that helps against mistakes.

**Check.** Before committing, can the user tell the scope of what they're about to do — count, targets, reversibility? If that information only appears in the result, it arrived too late to be useful.

## The visible structure of your controls is the conceptual model

**Trigger.** A settings panel, configuration surface, or any grouped set of controls where options affect one another.

**What goes wrong.** A **conceptual model** is the user's simplified explanation of how something works. It does not need to be accurate to be useful — it needs to predict the effect of actions correctly.

The **system image** is everything the user can actually perceive: visible structure, behaviour, labels, documentation, error messages. It is the *only* channel through which their model gets shaped, because you and they never communicate directly. **Whatever the system image fails to convey, the user invents** — and if what they invent is wrong, they cannot succeed however carefully they try.

The instructive failure: two labelled dials on a fridge, one marked *freezer* and one marked *refrigerator*, strongly implying two independent thermostats. The actual mechanism is one thermostat and one valve diverting cold air between compartments. The system image doesn't merely fail to explain the mechanism — it actively asserts a false one. Compounded by a 24-hour feedback delay, the correct model is effectively undiscoverable by experiment.

Settings panels do this constantly. Two toggles presented as siblings, where one silently overrides the other. Options grouped by which service owns them rather than by what they affect. A slider whose effect is clamped by a value on another tab.

**What to do.** Make the grouping and labels state the actual causal structure. If two controls are coupled, show the coupling — disable the dependent one, show the effective value, or group them together. Correct implementation underneath does not compensate for a layout that asserts the wrong model.

**Check.** From the visible structure alone, what would a user conclude about which settings affect which? Where that conclusion is wrong, that's a defect in the interface, not a documentation gap.

## Long feedback delays make the model undiscoverable

**Trigger.** Any setting whose effect isn't observable for minutes, hours, or a billing cycle.

**What goes wrong.** Users build models by acting and observing. When the delay between the two is long enough, the loop is broken — no amount of experimentation converges, and whatever model they started with persists uncorrected. This is what turns a merely-confusing control into a permanently misunderstood one.

**What to do.** Where the real effect is delayed, supply a proxy that isn't: a preview, a projected value, a statement of what will change and when. If you can't shorten the loop, close it with a prediction.

**Check.** How long until a user can tell whether their change did what they wanted? If the answer is longer than a session, they will never learn this control by using it.

## A reminder needs both halves

**Trigger.** Any deferred-action mechanism — a notification, flag, badge, "remind me later."

**What goes wrong.** A reminder has two separable components: the **signal** that something needs attention, and the **message** of what it is. Most mechanisms supply one. A badge with no content is a signal without a message; a note filed somewhere the person won't look again is a message without a signal.

**What to do.** Ensure both are present and that the signal appears where and when the action is actually possible.

**Check.** Does the reminder tell the user *that* something needs doing and *what*, at a moment when they can act on it?
