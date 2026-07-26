---
name: incident-review
description: Conduct a postmortem and judge decisions fairly after the fact — writing up a conjunction of contributing factors instead of a single root cause, correcting for hindsight bias, evaluating a call by what was knowable at the time, and recognising that added process is added coupling. Use when writing a postmortem, reviewing an outage or a near miss, analysing why something failed, or assessing a decision someone made under uncertainty.
---

# Incident Review

## Purpose

`resilience` covers surviving failure. `slo-and-alerting` covers noticing it. This covers what happens after: the review, and specifically the ways a review reliably goes wrong.

The failure modes here are cognitive rather than technical, which makes them harder to see. A review that names one cause, fixes it, and closes has *felt* rigorous. It usually wasn't.

## The premise most reviews get wrong

**A system that appears healthy is not a system without defects.** Complex systems run continuously with a shifting population of individually-survivable flaws — that's the resting state, not an alarm state. Nothing is "on fire" is not evidence of margin.

Accidents happen when several of those latent weaknesses line up. Which means:

**There is no root cause. There is a conjunction.** An outage that required four contributing conditions has four contributing conditions. Picking the most legible one and calling it *the* cause is a narrative convenience that leaves the other three in place — and they'll be there for the next conjunction.

The test for a completed writeup: **can you show that no single contributing factor would have produced the outage alone?** If one would have, you haven't found the conjunction yet — you've found a single point of failure, which is a different and simpler finding.

## Rules that apply without loading anything

**1. Delete the "Root Cause" field from your postmortem template.** A single-value field forces a multi-factor event into one box, and whatever lands in that box absorbs all the corrective attention. Replace it with a list of necessary-but-individually-insufficient conditions.

**2. Reconstruct what was visible at decision time.** After you know the ending, ambiguous signals look diagnostic. That's not reviewer sloppiness, it's a predictable effect of knowing the outcome, and it has to be actively corrected for. Rebuild the dashboards, logs, and alerts *as they appeared then* — not as annotated afterward — before judging any decision.

**3. Grade the decision, not the outcome.** Every action under uncertainty is a bet. If you praise the engineer whose risky hotfix worked and blame the one whose equally-informed bet didn't, you've taught everyone that the lesson is to hide uncertainty. Apply the same rubric regardless of how it landed.

**4. Ask what unresolved tension the operator was arbitrating.** When an organisation never settles "ship fast" against "never break prod," someone at the sharp end resolves it under pressure, mid-incident. Reviewing that as though they invented the tradeoff misses the actual finding, which is upstream.

**5. Count the defenses that worked.** A review naming the one safeguard that failed implies a single control would have prevented everything — while ignoring the several that caught near-identical conditions silently, leaving no trace. Near-miss data is as diagnostic as outage data, and it's usually not collected.

**6. Treat added process as added coupling.** Action items that bolt on a new approval gate or sign-off are themselves new components that can fail, be bypassed, or interact badly with something else. A remedy can raise total system complexity and manufacture fresh latent conditions. Weigh what it removes against what it adds; prefer eliminating a hazard over fencing it.

**7. Migrations that eliminate a failure class open an observation window.** New machinery adopted to remove a known failure mode introduces different ones — typically rarer and larger, and slower to surface because they need volume and time. Don't close the book at cutover; hold heightened scrutiny for a defined period.

## Safety is produced, not banked

The reframe worth internalising: the people running a system are never purely *doing the work* or purely *guarding against failure*. They are doing both, continuously, and the safety of the system is a byproduct of that ongoing balancing act rather than a separate deliverable.

Three consequences that bite:

- **Informal adaptation is load-bearing.** Practitioners constantly reroute load, pre-position attention, and improvise workarounds that never reach a runbook. Automating away a step that looks redundant can silently remove adaptation that was preventing incidents. Ask what people actually do before simplifying a workflow.
- **A quiet quarter is not a stable system.** It may be a system being actively held inside its limits by attention nobody is crediting. Cutting that attention because nothing is happening removes the reason nothing is happening.
- **Expertise is a depreciating, rotating asset.** Skill decays and people rotate off, narrowing the safety margin with no code change at all. Exposure to real incidents is a continuous operating cost, not an onboarding step.

This also reframes fault injection and gamedays. `resilience` treats them as *system* validation. They are equally *operator* calibration — building a felt sense of how close the system runs to its edge, so the first real incident isn't also the first real judgement call.

## Boundary with `scip-root-cause`

These sound contradictory and aren't, but the difference is worth stating because an agent could hit both.

- **`scip-root-cause`** asks: *why does this family of bugs keep recurring?* That's a design-flaw diagnosis. Recurring bugs usually do share one structural cause, and finding it is the job.
- **`incident-review`** asks: *why did this outage happen, and how should we judge the decisions made during it?* Accidents in complex systems are conjunctions of latent conditions plus a triggering context. There is no single box.

Different questions about different objects. A bug family has a design flaw. An outage has a conjunction. Applying either frame to the other's object produces a bad answer — hunting for one cause in an outage, or accepting "several things lined up" for a bug that recurs every sprint.

## What this skill will not do

Tell you whether to blame anyone. It argues that outcome-based judgement produces worse information, and that most sharp-end decisions are resolving tensions set upstream — but consequences for individuals are an organisational and human question, not a methodological one.
