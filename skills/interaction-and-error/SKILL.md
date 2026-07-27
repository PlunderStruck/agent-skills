---
name: interaction-and-error
description: Design interactions people can figure out and recover from — signifiers versus affordances, natural versus arbitrary mapping, feedback timing and quality, the slip-versus-mistake distinction and why a confirmation dialog catches one but not the other, mode errors, forcing functions, and undo versus confirmation. Use when building a destructive action, a confirm or undo path, a mode or state toggle, an icon-only control, an error message, a settings panel, or a multi-step flow, and when deciding what feedback an action needs. For visual hierarchy, typography, spacing, colour, and aesthetics, use a visual design skill instead.
---

# Interaction and Error

## Purpose

When a competent person fails at something simple — a door, a form, a confirmation dialog — the failure is usually legible in the design rather than in the person. This skill supplies the vocabulary to name why an interaction will fail *before* it ships, and a taxonomy of error that says which fix addresses which failure.

The taxonomy is the part that pays. Most interface defences are applied to the wrong error category, where they are not merely weak but **inert while looking like a fix** — which is worse, because it closes the question.

## When this applies

- A destructive, bulk, or irreversible action
- A confirmation dialog, an undo path, or a "are you sure" of any kind
- A mode, toggle, or state that changes what other controls do
- An icon-only button, a gesture, a drag target, a swipe
- A settings or configuration surface where options affect each other
- An error message, a validation failure, a rejected input
- Any async action where the user must learn what happened
- Two adjacent controls where one is safe and one is not

## Rules that apply without loading anything

**1. A confirmation dialog catches slips. It does nothing against mistakes.** If the action was mis-executed, the unexpected prompt is itself the alarm — surprise is the detection mechanism. If the *plan* was wrong, the prompt matches what the person already intends, and they confirm it without reading. Same dialog, same code, opposite outcome. "We added a confirmation" is an answer to *how do we catch slips*, never to *how do we prevent destructive mistakes*.

**2. Undo is the stronger default, because it doesn't require noticing anything at decision time.** It's the one defence that works against both slips and mistakes, since it acts after the fact rather than gating the moment of commitment. Reach for it before reaching for friction.

**3. What's missing is almost always the signifier, not the affordance.** The tap, drag, or swipe is already wired up; nothing on screen *says so*. For any interactive element ask: is there a perceivable signal, independent of prior knowledge, that this action is possible here? A hover tooltip or a changelog entry is not a signifier.

**4. Any state where the same control means different things is a mode, and it needs a continuously visible indicator.** Mode errors get worse when an interruption separates selecting the mode from using it, and worst when two modes look similar but produce very different effects. Eliminate the mode if you can; make it salient if you can't.

**5. Never distinguish a destructive action from a safe one by position alone.** Descriptions only need to be precise enough to separate the options *currently present* — so a layout that has been fine for years starts producing wrong-target errors the moment a similar-looking row joins the list.

**6. A forcing function fixes a slip and makes a mistake worse.** It guarantees each step executes; if the plan is wrong, it just carries out the wrong plan more reliably. Identify the error category before choosing the defence.

**7. If an action genuinely can't be reversed, make it harder to perform — don't compensate with a confirmation.** Confirmation is friction on the click, not on the reasoning that led to it.

**8. Feedback must be immediate and must name what happened to what.** Delay past roughly a tenth of a second reads as unresponsive and invites a repeat press — which may fire a second, unintended effect. And bad feedback is a distinct failure from absent feedback: too little (something happened, but not what), too much (routine noise that trains people to mute the channel, including the one alert that mattered), or a channel the user can't attribute to the right source.

**9. The visible structure of your controls *is* the conceptual model, as far as the user is concerned.** If the grouping and labels imply two things are independent when they're actually coupled, a correct implementation underneath does not rescue you — the user's model is wrong and their actions will be too.

**10. "User error" is where an investigation begins, not where it ends.** Ask why the system made the error easy or undetectable first. Only then is "the person was also at fault" a defensible residual.

## Triage

| What you're building | Reference |
|---|---|
| An icon-only control, gesture, drag target, or a control layout | [signifiers-and-mapping](references/signifiers-and-mapping.md) |
| A destructive action, a confirm/undo path, a mode or toggle | [slips-and-mistakes](references/slips-and-mistakes.md) |
| A toast, loading state, async result, or a settings panel with coupled options | [feedback-and-models](references/feedback-and-models.md) |
| Validation, error handling, recovery, guardrails on a dangerous operation | [designing-for-error](references/designing-for-error.md) |

## The question underneath all of them

**Which of the two gulfs is this failing on?**

An action cycle runs seven stages: form a goal, plan, specify a sequence, perform — then perceive, interpret, compare against the goal. The **Gulf of Execution** is the gap between what someone wants and the actions the system makes discoverable. The **Gulf of Evaluation** is the effort to work out what happened afterward.

Every technique here narrows one or the other. Signifiers, mapping, and constraints narrow execution. Feedback and conceptual models narrow evaluation.

**This is a live diagnostic, not a diagram.** Ask which stage the person actually gets stuck at. Better feedback does nothing for a Gulf-of-Execution failure, and a clearer signifier does nothing for a Gulf-of-Evaluation one. Interfaces routinely get the fix from the wrong side of the cycle.

## Two things worth knowing that produce no check

**Slips happen more to experts than to novices.** Expert performance is automated, which is exactly what makes attention divertible at the junction where a wrong branch is available. Don't design destructive flows on the assumption that familiarity protects people — it's the mechanism of capture slips, not a defence against them.

**Checklists work through a structure you probably don't have.** The evidence Norman cites is for *collaborative, two-person* execution — one reads, one performs. One person doing both, or a second person re-checking afterward, affords each less care, because each assumes the other will catch it. More checkers can mean less attention per checker. For a solo developer the real substitute is a tool-enforced check, not a resolution to be careful.

## Boundary with neighbouring skills

- **Visual design skills** (`impeccable`, `frontend-design`, and similar) own hierarchy, typography, spacing, colour, and aesthetic direction. This skill owns whether the interaction can be understood and recovered from. A beautiful interface can be full of mode errors.
- **`security`** — an action being *authorized* is separate from it being *intended*. Least privilege limits blast radius; this skill reduces the chance a legitimate user destroys something on purpose-by-accident.
- **`incident-review`** — shares the Swiss cheese model and the rejection of single-root-cause thinking, applied after an outage. This skill applies the same reasoning before one.
- **`api-evolution`** — the same "your consumers built a mental model you must not silently break" argument, for machine consumers rather than human ones.
