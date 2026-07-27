# Slips and Mistakes

The most load-bearing distinction in the book, and the one most often flattened into "users make errors." The two categories occur at different altitudes of the action cycle and take **different, sometimes opposite, fixes**.

- A **slip**: the goal and plan were right, the execution wasn't. You did something other than what you intended. Lives in the lower stages — perform, perceive, interpret — governed by automatic processing.
- A **mistake**: the goal or plan itself was wrong. Execution may have been flawless; it carried out a bad plan. Lives in the upper stages — goal, plan, evaluate — governed by conscious reasoning.

**Slips happen more to experts.** Expert action is automated, which is exactly what leaves attention free to wander at the junction where a wrong branch is available. Familiarity is the mechanism, not the protection.

---

## The confirmation dialog catches one and not the other

**Trigger.** Any "are you sure?" guarding a destructive action.

**What goes wrong.** Consider the same dialog in two situations.

*Against a slip:* you mean to type one shortcut and hit an adjacent one. A dialog appears that you were not expecting. **The surprise is the detection mechanism** — you didn't intend this, so the unexpected prompt tells you something diverged, and you cancel. It works.

*Against a mistake:* you deliberately close a window you believe is inactive. It isn't. The dialog appears and says exactly what you already believe you're doing. It matches your (wrong) plan, so you confirm without reading, and lose the work. It does nothing.

Identical element, identical code path, opposite outcome — because the two error categories are being conflated. This is why "we added a confirmation" is not an answer to *how do we prevent destructive mistakes*. It answers only *how do we catch slips at the moment of divergence*.

And a dialog on a routine path stops catching even slips, because it stops being a surprise. Confirmation shown on every delete is a click people learn to emit reflexively; the surprise it depends on has been trained away.

**What to do.** Name which error category you are defending against before choosing the defence.

- Against **slips** — confirmation works, but only where the action is rare enough that the prompt stays surprising. Otherwise use divergence (below) or undo.
- Against **mistakes** — nothing at the confirm click helps. Move upstream: make the current state visible so the person's diagnosis is less likely to be wrong, and run sensibility checks on the *content* of the action rather than adding friction to the act of confirming. See [designing-for-error](designing-for-error.md).
- Against **both** — undo, because it doesn't require noticing anything at decision time.

**Check.** For each confirmation in the codebase: is the action rare enough that the prompt is still a surprise? If it fires on a routine path, it's a reflex, and you have a defence that reads as protection while providing none.

---

## Capture: a frequent path swallows a rare one

**Trigger.** Two flows that share opening steps, where one is common and the other is rare and consequential.

**What goes wrong.** The more-practised sequence captures control at the point where the two are still identical, and carries on down the familiar branch. The person doesn't notice, because up to the divergence they were doing exactly the right thing.

**What to do.** Make the rare, consequential path diverge from the common one as early as possible — not at the point where they differ semantically, but at the first interaction. A destructive operation sharing four steps with a routine one, differing only at the last, is built for capture.

**Check.** Where does the dangerous flow first look different from the safe flow it resembles? If that's the final step, the divergence is too late.

---

## Description-similarity: right action, wrong object

**Trigger.** A list, grid, or menu where several targets are visually or verbally similar and at least one is destructive.

**What goes wrong.** The target gets selected using a mental description precise enough to distinguish among the options *currently present*. When a second object satisfies that same description equally well, the wrong one gets picked.

The important consequence is temporal: **a layout that has worked for years breaks the moment a new similar-looking item joins the set.** Nothing about the existing design changed. The discrimination that was sufficient stopped being sufficient.

**What to do.** Make controls for meaningfully different actions differ in more than position — wording, colour, weight, iconography, or placement outside the group. Treat adding an item to a list containing a destructive action as a change to that action's safety, not a neutral addition.

**Check.** If a new row or button joined this set tomorrow, would the destructive one still be distinguishable by something other than where it sits?

---

## Mode errors: the same control, two meanings

**Trigger.** Any state that changes what other controls do — an edit/preview toggle, a selected environment, a vim-style modal input, a "currently editing X" context.

**What goes wrong.** Modes become unavoidable once a system has more meanings than controls. They fail when the user isn't tracking which mode is active — most often because an interruption separated selecting the mode from using it, or because two modes look similar while producing very different effects.

The severe version: an input that is valid in both modes but means something entirely different in each. The value itself carries no signal that the interpretation differs, so nothing about the entry looks wrong.

**What to do.** Eliminate the mode where you can. Where you can't, make the active mode continuously visible in the place the user is looking — not in a corner, not in a status bar they've stopped seeing. Where the same input means different things per mode, make the *input* differ too, not just the indicator.

**Check.** After an interruption, can the user tell which mode they're in without performing an action to find out? And is there any input that would be silently accepted with a different meaning in the other mode?

---

## Memory-lapse slips: the housekeeping step

**Trigger.** Any flow with a step after the one the user actually came for — retrieving a card after the cash, closing a transaction after the result, saving after reviewing.

**What goes wrong.** People almost never forget the goal. They forget the trailing steps that don't serve it, particularly across an interruption.

**What to do.** Reorder so the thing they came for is withheld until the forgettable step is done. A forcing function is the right tool here specifically because the plan was already correct — see [designing-for-error](designing-for-error.md).

**Check.** Does any step come *after* the user has got what they wanted? If so, assume some proportion will not happen.

---

## Mistakes: three kinds, none fixed by friction

**Rule-based** — the situation is misclassified and the wrong rule applied; or the right rule is applied but doesn't cover this case (a policy written for routine operation with no exception path for an emergency); or the rule works and its outcome is misread as a malfunction.

**Knowledge-based** — the situation is genuinely novel, no rule applies, and reasoning from scratch goes wrong because the underlying knowledge is incomplete or false. Unit confusion is the classic: computing in one unit where another was required, correctly, all the way to the wrong answer.

**Memory-lapse mistakes** — the goal or plan is lost wholesale rather than one step within it, usually across an interruption, and a fresh and possibly wrong decision gets made from nothing. **Harder to detect than the slip equivalent**: a missing step in an intact plan violates an expectation, so something feels off. A lost plan leaves no expectation to be violated.

**What to do.** All three are upstream of the interface's last line of defence. What helps is keeping goal, plan, and current system state visible so the diagnosis is less likely to be wrong — and sensibility checks on the action's content, which catch a wrong plan precisely because they don't depend on the user's belief about it.

**Check.** If the user's understanding of the current state were wrong, would anything in this interface contradict them before the action completes? If the only gate is a prompt describing what they already think they're doing, the answer is no.
