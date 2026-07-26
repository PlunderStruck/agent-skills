# Extraction: complex-systems-failure

**Source:** How Complex Systems Fail — Richard I. Cook

> Operational rules distilled from the source, written in our own words. Includes the placement analysis that decided this material became a new skill rather than extending existing ones.

---

# How Complex Systems Fail — Operational Distillation and Placement Recommendation

*Source: Richard I. Cook, "How Complex Systems Fail" (1998–2000). Eighteen numbered observations, distilled below into practice, followed by a placement judgment against the `resilience` and `slo-and-alerting` skills.*

## Part 1 — Operational content

### Cluster A: Hazard and defense are structural, not incidental [1, 2, 3]

The starting condition isn't a clean system that occasionally breaks — it's a system built specifically because the work it does is dangerous, wrapped in enough overlapping safeguards that most danger never surfaces.

- **[1] Risk doesn't retire.** *Trigger:* treating an outage as a deviation from an otherwise-safe baseline. *Failure:* teams budget resilience work as a backlog to clear, then stop investing once it's "done." *Do instead:* treat defense-in-depth spend as a standing operating cost tied to what the system does, not a project with an end date.
- **[2] Most defenses already work, silently.** *Trigger:* a postmortem names one missing safeguard as "the fix." *Failure:* this implies a single well-placed control would have stopped it, while ignoring the dozen controls that caught near-identical situations and left no trace. *Do instead:* credit and inventory the defenses that fired correctly in the same incident, not just the one that didn't — a near-miss log is as diagnostic as an outage report.
- **[3] Accidents are conjunctions, not single failures.** *Trigger:* the incident timeline stops at the first plausible culprit. *Failure:* a fix ships for one contributing weakness, and the class is declared closed, even though the real event needed several independent weaknesses to line up. *Do instead:* keep enumerating contributing factors until you can show that none of them alone would have produced the outage.

### Cluster B: "Normal" already means "degraded," and every action is a bet [4, 5, 6, 10]

This is the observation that most cuts against instinct: a healthy-looking system is not a flawless one, and the same uncertainty that makes a bad call look like negligence afterward also makes a good call look like foresight.

- **[4] Unfixed, tolerable defects are the permanent background state.** *Trigger:* "we're fine, nothing's on fire." *Failure:* absence of an active incident gets read as absence of risk, when a shifting set of individually-survivable defects is always present and occasionally combines. *Do instead:* maintain a live census of known-but-unfixed weaknesses as a tracked artifact, not just a count of closed incidents.
- **[5] Degraded-but-functioning is the resting state, not an alarm state.** *Trigger:* expecting a fully clean baseline against which any deviation reads as anomalous. *Failure:* early degradation gets normalized ("it's always a little like that") until it compounds into the thing the postmortem later calls obvious in retrospect. Proximity to catastrophe [6] isn't a condition you enter during an incident — it's the resting distance, so surviving a scare is not evidence of margin. *Do instead:* give the degraded-but-tolerable band its own name, its own accepted duration, and its own threshold, instead of only having "up" and "down."
- **[10] Every action is a gamble, judged unfairly by its outcome.** *Trigger:* praising the engineer whose risky hotfix worked and blaming the one whose equally-informed bet didn't. *Failure:* outcome becomes the standard instead of the quality of the decision given what was knowable at the time, which teaches people to hide uncertainty rather than surface it. *Do instead:* score decisions by the information available when they were made, applying the identical rubric whether the bet paid off or not.

### Cluster C: Post-incident explanation is structurally biased [7, 8, 11, 15]

Once you know the ending, the path there looks obvious — and that illusion is not a character flaw in reviewers, it's a predictable cognitive effect that has to be actively corrected for.

- **[7] There is no root cause, only a conjunction that a review has to hold onto in full.** *Trigger:* a postmortem template with a single "Root Cause" field. *Failure:* a multi-factor event gets forced into one named culprit, which then absorbs all the corrective attention while the rest of the actual conjunction goes untouched. *Do instead:* require the writeup to show the joint set of necessary-but-individually-insufficient factors, not a chain terminating in one box.
- **[8] Hindsight makes ambiguous signals look diagnostic.** *Trigger:* "how did nobody notice X was about to fail" in review. *Failure:* the reviewer already knows the ending, so cues that were genuinely ambiguous in the moment look obviously significant after the fact, and that gap gets scored as operator negligence. *Do instead:* reconstruct the decision using only what was visible at decision-time — dashboards and logs as they looked then, not as annotated later — before judging the choice.
- **[11] Sharp-end operators resolve tensions leadership left open.** *Trigger:* an organization leaves "ship fast" and "never break prod" unreconciled until someone has to choose between them mid-incident. *Failure:* the review then treats that choice as if the engineer invented the tradeoff, rather than being forced to arbitrate a conflict management never settled. *Do instead:* ask what unresolved organizational tension the operator was forced to resolve, and fix that tension — not just the local decision.
- **[15] Blame-shaped remedies add coupling instead of removing risk.** *Trigger:* action items that are new approval gates, sign-offs, or checks bolted onto whatever got blamed. *Failure:* each gate is itself a new component that can fail, be bypassed, or interact badly with something else — so the fix can raise total system complexity and manufacture fresh latent failures. *Do instead:* weigh a remedy's added coupling against the risk it removes, and prefer eliminating a hazard over fencing it with process.

### Cluster D: Operators are simultaneously producing and defending [9, 12, 13, 17, 18]

The most consequential reframe in the piece: the person running the system is never purely "doing the work" or purely "watching for danger" — they are doing both, continuously, and the system's safety is a byproduct of that ongoing balancing act, not a separate deliverable.

- **[9] Production and defense are one continuous role, not two alternating ones.** *Trigger:* treating on-call as "shipping work" outside incidents and "safety work" during them. *Failure:* a fast hotfix under pressure is itself a production-versus-safety bet; treating it as pure heroism or pure recklessness erases the tradeoff that was actually made. *Do instead:* name the production pressure explicitly as a factor in design decisions and reviews, rather than assuming it's too obvious to state.
- **[12, 17] People actively reshape the system to keep it inside tolerable limits, and that ongoing work *is* the safety.** *Trigger:* assuming the documented process is what's actually keeping the system running, or that a quiet quarter means the system reached a stable, safe state on its own. *Failure:* both miss that people are constantly rerouting load, pre-positioning attention, and improvising workarounds that never reach a runbook — automating away a "redundant-looking" manual step can silently remove load-bearing adaptation, and cutting the people who provide it because "nothing's happening" removes the thing that was preventing incidents. *Do instead:* interview practitioners about their informal adjustments before simplifying a workflow, and give visibility/credit to the routine judgment calls that prevent incidents, not only the response to the ones that occur.
- **[13] Expertise is a depreciating, rotating asset, not a fixed one.** *Trigger:* assuming current on-call competence is permanent. *Failure:* skill decays and turns over as people rotate off, quietly narrowing the safety margin without any code change. *Do instead:* treat shadowing and exposure to real incidents as continuous operating cost, not a one-time onboarding step.
- **[18] Calibrated judgment requires real contact with failure.** *Trigger:* sanitizing staging and on-call so thoroughly that engineers rarely see anything actually break. *Failure:* it produces operators who can follow a runbook but have no felt sense of how close the system is to its edge, so their first real incident is also their first real judgment call. *Do instead:* deliberately expose practitioners to controlled failure — gamedays, fault injection, shadowing live incidents — so judgment is built before it's needed for real.

### Cluster E: Fixes open new, unseen failure surfaces [14]

*Trigger:* adopting new tooling or infrastructure specifically to eliminate a known, well-understood failure mode. *Failure:* risk is declared reduced once the old failure mode disappears, without noticing that the new abstraction created a different — often rarer and larger — failure mode that hasn't shown up yet, because it takes time and volume to surface. *Do instead:* treat any migration that "eliminates" a failure class as opening an observation window for new ones; hold heightened scrutiny for a defined period afterward instead of closing the book at cutover.

### Cluster F: Safety is a property of the whole system, and it's made continuously [16]

*Trigger:* "we have a safety/reliability team, we added a scanner, therefore we're safe." *Failure:* this treats safety as owned by one component, team, or gate — satisfiable once, then ignorable — when it actually lives in how every part interacts under simultaneous pressure. *Do instead:* evaluate safety as emergent, whole-system behavior under combined stress (how the system degrades when several things go wrong at once), rather than certifying it through a checklist one team owns.

---

## Part 2 — Placement recommendation

### Where each observation actually lands

| # | Content | Verdict |
|---|---|---|
| 1 | Hazard is structural, defenses are permanent cost | Restates `resilience`'s premise; not new tactics |
| 2 | Defenses already work in layers | Restates `resilience`'s premise (timeouts/breakers/bulkheads *are* this) |
| 3 | Accidents need joint conditions | Adjacent to `resilience`'s cascading-failure model; worth a short extension |
| 4 | Latent flaws are the constant background | New; touches monitoring, not yet in `slo-and-alerting` |
| 5 | Degraded mode is the resting state | **New — emphasis item.** Not in either skill |
| 6 | Catastrophe is always proximate | Thin restatement of 1/2; skip as standalone |
| 7 | No root cause, only a conjunction | **New.** Neither skill discusses postmortem methodology |
| 8 | Hindsight bias corrupts review | **New.** Not covered anywhere |
| 9 | Dual role: producer and defender | **New — emphasis item.** Not covered |
| 10 | Actions are outcome-biased gambles | **New.** Not covered |
| 11 | Sharp end absorbs unresolved org ambiguity | **New.** Not covered |
| 12 | Operators actively reshape the system | **New.** Adjacent to `resilience`'s "graceful degradation," but that's automated behavior, not human adaptation |
| 13 | Expertise constantly turns over | New but thin; minor addition at most |
| 14 | Fixes create new failure surfaces | Adjacent to `resilience`'s hostile-failure-testing; worth a short extension |
| 15 | Blame-remedies add coupling | New, but literally about the cost side of `resilience`'s own patterns — good bridging note |
| 16 | Safety is a system property | **New — emphasis item.** Core organizing idea, absent from both |
| 17 | Safety is produced continuously, not banked | **New — emphasis item.** Absent from both |
| 18 | Judgment requires contact with real failure | Adjacent to `resilience`'s hostile-failure-testing, but from the *operator-training* angle rather than the *system-validation* angle |

### The honest read

Cut the noise (1, 2, 6, 13 are framing, not new tactics) and what's left splits cleanly into two groups.

**Group one — literal extensions of what you already have (3, 14, 15, 18 → `resilience`; 4, 5 → `slo-and-alerting`).** These observations sharpen practices those skills already teach, they don't introduce a new domain:

- `resilience` already teaches hostile-failure testing, cascading-failure dynamics, and defense layering. Cook adds *why*: fault injection also calibrates human judgment about the edge of the envelope, not just system behavior [18]; a cascade needs joint conditions so single-factor postmortems undershoot the fix [3]; new resilience machinery is itself a new failure surface requiring an observation window [14]; and every added layer of protection is itself a coupling cost that should be weighed, not assumed free [15]. This is three or four paragraphs, not a skill.
- `slo-and-alerting` already teaches symptom-based alerting and the four golden signals. Cook adds a leading-indicator argument: track the population of known-but-tolerable defects and name a "degraded but functioning" band explicitly, rather than only alerting on up/down [4, 5]. Also a few paragraphs, not a skill.

**Group two — a genuinely separate competency, currently covered by neither skill.** Observations 7, 8, 9, 10, 11, 12, 16, and 17 are not about how to build a resilient system or what to alert on. They're about how to run an incident review, how to judge a human decision fairly, and how to think about operators and organizational safety at all. That's four coherent, non-overlapping bodies of practice:

1. Postmortem structure (no root-cause field; conjunction of factors) [7]
2. Decision review methodology (hindsight correction, outcome-independent grading of gambles, reconstructing what was knowable at decision time) [8, 10]
3. Organizational framing (production-versus-defense duality, sharp-end ambiguity that management left unresolved) [9, 11]
4. What "safety" actually is and how it's made (emergent system property, produced continuously by operator adaptation rather than banked by a control) [12, 16, 17]

Neither `resilience` (system design under dependency failure) nor `slo-and-alerting` (defining "working" and what pages) has any natural hook for "how should we conduct the review after the page fires" or "how should we evaluate the human who made the call." Stretching either skill to cover this would blur its scope — `resilience` would start teaching incident-review methodology under a skill about circuit breakers, and `slo-and-alerting` would start teaching blame psychology under a skill about paging thresholds. That's the wrong kind of thin.

### Recommendation

**(d), but lopsided toward (a).** Ship a new skill — something like `incident-review` or `postmortem-practice` — scoped to conducting postmortems, reviewing sharp-end decisions without hindsight bias, and treating safety as an emergent, continuously-produced system property (observations 7, 8, 9, 10, 11, 12, 16, 17 as the core; 1, 2, 6, 13 folded in as opening framing rather than standalone rules). That's a full, coherent skill, not a thin wrapper around one essay — it's the methodological complement to `resilience` (which tells you how to survive failure) and `slo-and-alerting` (which tells you how to notice it), covering the part neither touches: what to do with a human decision after the fact.

Alongside that, add the two small bridging extensions identified above: three or four paragraphs to `resilience` (3, 14, 15, 18) and two paragraphs to `slo-and-alerting` (4, 5). Cross-link the new skill from both, since "why we test failure this way" and "why the postmortem doesn't award a root cause" are two views of the same underlying claim — but don't let that shared claim talk you into merging them. Design-time resilience, monitoring definitions, and post-incident human judgment are three different jobs, done at three different times, by people who may not even be the same person.
