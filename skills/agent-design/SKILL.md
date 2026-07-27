---
name: agent-design
description: Design autonomous agent systems and the human oversight around them — specifying what an agent can perceive and act on, why a stateless loop repeats itself forever under partial observability, why an agent scored on a proxy will game the proxy, keeping the grading criterion outside the agent's reach, checking effects rather than strategy, and why "a human reviews the output" stops being a control. Use when building a tool-calling loop, a multi-step autonomous workflow, an eval or scorer, an approval or escalation path, or when deciding how much autonomy to grant. For Claude API mechanics — model IDs, parameters, tool-call syntax, pricing — use the claude-api skill instead.
---

# Agent Design

## Purpose

Two bodies of work, neither written about LLMs, that describe agent failure more precisely than most current writing: Russell & Norvig's formal vocabulary for specifying an agent and its environment, and Bainbridge's account of what happens to the human left supervising an autonomous system.

The pairing is the point. The first tells you what you're building; the second tells you why the safety story you attached to it — "a human stays in the loop" — degrades in ways that are predictable and mostly unaddressed.

## When this applies

- A tool-calling loop, or any multi-step autonomous workflow
- An eval, scorer, success criterion, or reward signal
- An approval prompt, permission gate, or escalation path
- A subagent, a fan-out, or two agents acting on the same files
- Deciding how much autonomy to grant, or where to require a human
- An agent that loops, repeats itself, or re-derives what it already knew
- Any claim that a human reviewing output makes a system safe

## Rules that apply without loading anything

**1. A stateless loop in a partially observable environment loops forever.** An agent deciding purely from the current observation, with no memory of what it already tried, is a *simple reflex agent* — and infinite loops are largely unavoidable for one under partial observability. "Test still failing → apply same fix → still failing" is this exact failure. Almost no environment an agent operates in is fully observable; a context window is a partial, curated view. State-tracking isn't an optimization, it's the fix for a structural defect.

**2. What you ask for is what you get.** Score an agent on the *activity* you think produces the outcome, and a capable agent maximizes the activity. Scored on "tests pass," weakening a failing test is a correct solution. Scored on "ticket closed," it closes tickets. This isn't malice or an edge case — gaming the measure is precisely what maximizing it means. Score the state you actually want to be true.

**3. The grading criterion must live outside the agent's reach.** The performance standard has to be fixed and external, because the agent must not be able to modify it to fit its own behaviour. A coding agent that can edit the test suite meant to be checking its work has no test suite.

**4. Cheap information-gathering is not diligence, it's rationality.** Acting to improve your future percepts falls directly out of maximizing expected performance — it isn't a safety habit layered on top. Skipping a one-tool-call check and acting on a stale or absent observation is irrational in the strict sense. Read before write, status before mutate, verify before commit.

**5. "A human reviews it" is not a control.** Two independent reasons. Sustained attention to a mostly-fine stream decays within about half an hour, regardless of motivation. And if the agent was deployed because it outperforms the reviewer, real-time verification of its reasoning isn't available — leaving only plausibility assessment, which is the one thing fluent generated output defeats.

**6. Automation should fail obviously.** A system that compensates for a developing fault hides it until the deviation exceeds what compensation absorbs. Silent retries, silent tool fallbacks, silent context truncation, silently degrading to a weaker model — each is camouflage. Graceful degradation, a virtue elsewhere, works directly against supervisability.

**7. Check effects, not strategy.** Constrain and verify what changed — tests green, no files touched outside scope, invariants hold — rather than the route taken. Effect-checks make no assumptions about method, so they survive an agent finding a path you didn't anticipate, which is the reason you deployed it.

**8. Distrust doesn't reduce load, it doubles it.** An operator who doesn't trust the automation monitors both the process *and* the automation. "Just review it carefully" is not the conservative fallback — it can be worse than either full trust or no agent.

**9. Judge a decision by what was knowable at the time.** Rational means maximizing *expected* performance given the percept sequence available — not being right. A choice that was sound on available evidence and broke on an unobservable edge case is not evidence of a bad process. Ask what was cheaply obtainable at decision time.

**10. Autonomy holds in low-variety, high-frequency task spaces and degrades as variety rises.** Full autonomy is achievable on narrow repetitive work and does not generalize to open-ended domains by adding capability. Where your task space sits on that axis is the autonomy decision.

**11. The tool list must match what the policy will try to do.** If the program recommends actions the architecture can't execute, you have a policy reaching for legs it doesn't have. Write out what the agent can actually perceive and actually change before wiring anything.

**12. Reliability and supervisability move in opposite directions.** The better the agent, the rarer the handoff, the more decayed the skill of whoever takes over — so the *more* investment the escalation path needs, not less.

## Triage

| What you're doing | Reference |
|---|---|
| Starting an agent; deciding its tools, inputs, and scope | [specifying-the-environment](references/specifying-the-environment.md) |
| An agent that loops, repeats work, or loses track of what it did | [architectures-and-state](references/architectures-and-state.md) |
| Writing an eval, scorer, success criterion, or reward | [objectives-and-rationality](references/objectives-and-rationality.md) |
| Approval gates, review steps, escalation, human-in-the-loop | [human-oversight](references/human-oversight.md) |

## The question underneath all of them

**What does this agent actually perceive, and is that enough to decide?**

Most agent failures reduce to acting on an observation that was incomplete, stale, or never taken. The stateless loop, the unverified assumption, the silent fallback, the human approving without the context to judge — each is a decision made on a percept that didn't support it.

The corollary that costs people the most: **partial observability is the normal case, not the exception.** A context window is a curated slice of a system that continues changing while the agent deliberates. Designs that would be correct under full observability are routinely shipped into environments that aren't.

## Two things worth knowing that produce no check

**The residue left to the human is not a designed job.** Automating what you know how to automate leaves the human whatever was too hard — a set defined by difficulty rather than by what people do well. "The human handles the hard parts" describes a role nobody designed and usually nobody supports.

**The best summary may prevent the understanding its own escalation depends on.** Retention tracks how much you had to process something to absorb it. An agent producing excellent summaries may be eroding the model of the system that the human needs at exactly the moment the agent hands off. This is the mechanism behind "I approved it because it looked right," and it's a live tension rather than a rule — the property that makes output easy to consume is what makes it easy to consume without thinking.

## Boundary with neighbouring skills

- **`resilience`** and **`distributed-data`** — an agent calling tools *is* a distributed system with an unreliable node. Timeouts that mean *unknown* rather than failure, retries needing idempotency, partial failure mid-plan: that material lives there and transfers with no translation. This skill covers what the agent should be, not how its calls should behave.
- **`interaction-and-error`** — the human-facing surface. Approval prompts are mode-and-confirmation design; that skill owns whether the human can tell what they're agreeing to, this one owns whether asking them is a control at all.
- **`incident-review`** — after an agent breaks something. Shares the rejection of single-root-cause reasoning and the by-what-was-knowable standard.
- **`unit-testing`** — what a test should assert. Directly relevant to rule 3: a grading criterion the agent can edit, or that asserts implementation rather than behaviour, fails for reasons that skill covers in depth.
