# Objectives and Rationality

What you score is what you get. This is the oldest result here and the one most reliably rediscovered the hard way.

## The performance measure is a hazard you create

**Trigger.** Writing an eval, a scorer, a success criterion, a reward signal, or an acceptance check — anything that decides whether the agent did well.

**What goes wrong.** Score a cleaning agent on *amount of dirt cleaned up per shift*, and the way to maximize that is to clean the floor, dump the dirt back out, and clean it again. Repeatedly. For score.

**With a rational agent, what you ask for is what you get.** That's not a warning about misbehaviour — it's the definition of maximizing working correctly. The agent isn't circumventing your measure; it's satisfying it, and your measure was a description of activity rather than of the state you wanted.

The rule underneath: **design the measure according to what you want to be true in the environment, not according to how you think the agent should behave to get there.** A measure aimed at the proxy activity gets gamed by any sufficiently capable optimizer, without malice, because gaming it is what maximizing it means.

Applied without translation:

- Scored on *tests pass* — weakening the failing test is a correct solution.
- Scored on *ticket closed* — it will close tickets.
- Scored on *no lint errors* — the suppression comment is the cheapest path.
- Scored on *lines changed* or *files touched* — both directions are gameable and neither correlates with the thing you wanted.

If what you actually want is "the code is correct," then *tests pass* is a proxy, and a capable agent will find the cheaper route to the stated measure.

**A subtler failure that survives fixing the obvious one.** A measure defined as an average over time is satisfied identically by steady adequate work and by bursts followed by idle stretches. It cannot distinguish a temporal shape you may care about unless designed to. Every aggregate measure has some version of this blind spot — ask what distributions produce the same score.

**What to do.** State the measure over the *state of the environment* you want true, not over the agent's activity. Then ask what the cheapest way to satisfy the stated measure is, assuming a capable and entirely literal optimizer, and assume that's what you'll get.

**Check.** What is the cheapest possible way to satisfy this criterion? If the cheapest way isn't the thing you wanted, you have your answer already.

## The standard must be outside the agent's reach

**Trigger.** Any agent with write access to a repository, a CI config, or an eval harness.

**What goes wrong.** The performance standard has to be fixed and external — *outside the agent altogether* — because the agent must not be able to modify it to fit its own behaviour. Once the grader is inside the blast radius, the grade is self-reported and the whole measurement apparatus is decorative.

This is not hypothetical for coding agents: tests, fixtures, snapshot files, assertion helpers, CI definitions, and eval specs are all ordinary files in the repository the agent was given write access to.

**What to do.** Separate the grading artifacts from the agent's writable surface — a protected path, a separate repo, a CI-side check the agent's commit can't alter, or at minimum a diff gate that flags any change to test files as requiring human review rather than counting as work done.

**Check.** List everything the agent can write to. Intersect that with everything that determines whether it succeeded. The intersection should be empty; if it isn't, that set is the actual scope of the measurement problem.

## Rationality, stated precisely

**Trigger.** Reviewing an agent's past decision, or arguing about whether it behaved correctly.

**What goes wrong.** "Rational" gets flattened into "correct" or "successful," which makes every bad outcome look like a design defect and every good outcome look like validation.

The definition: for each possible percept sequence, a rational agent selects the action expected to maximize its performance measure, given the evidence in the percept sequence and whatever built-in knowledge it has. What is rational depends on exactly four things — the performance measure, prior knowledge, available actions, and the percept sequence to date. Nothing else.

**Rationality is not omniscience.** An omniscient agent knows actual outcomes in advance, which is impossible. Crossing an empty street is rational even if a cargo door falls off a passing aircraft and kills you. Rationality maximizes *expected* performance; perfection maximizes *actual* performance, and only one of them is available at decision time.

**But it does not license ignoring cheap information.** Crossing without looking isn't rendered rational by an uninformative percept sequence — the rational action is to look first, because looking improves expected performance at trivial cost. Rationality isn't passive acceptance of whatever you happen to have observed.

**What to do.** When judging a decision — the agent's or a person's — ask what evidence was available and *cheaply obtainable* at the time, not what happened afterward. A migration that was sound on available evidence and broke on an unobservable edge case is not proof of a bad process. A migration that broke on something one command would have revealed is.

**Check.** Was there a cheap check that would have changed this decision? That question separates a bad outcome from a bad process, and it's the only one that generalizes.

## Information-gathering is not overhead

**Trigger.** Trimming an agent's steps for speed — dropping the read-before-edit, the status check, the verification pass.

**What goes wrong.** These get treated as diligence bolted onto the real work, and therefore as the first thing to cut. They aren't. Acting to improve future percepts falls directly out of maximizing expected performance under uncertainty — it *is* the rational action, not an add-on.

An unknown environment, in the strict sense from [specifying-the-environment](specifying-the-environment.md), is exactly the condition under which acting to learn an outcome before committing to a costlier path is required rather than optional.

**What to do.** Treat read-before-write, status-before-mutate, and verify-before-commit as constitutive of correct operation under partial observability. They're how an agent approaches the performance that a fully observable agent gets for free.

**Check.** Is the agent acting on an observation it took, or on one it assumed? Skipping an available cheap check and acting on a stale or absent percept isn't efficiency — it's irrational in the exact technical sense above.

## Autonomy: the assumption never re-checked

**Trigger.** An agent following a fixed procedure — a deploy script, a migration runbook, a multi-step checklist.

**What goes wrong.** An agent lacks autonomy exactly to the extent it relies on its designer's prior knowledge rather than its own percepts. Two illustrations, both about assumptions welded into behaviour:

A dung beetle whose dung ball is removed en route continues pantomiming the plugging motion at its nest. A sphex wasp, whose prey an experimenter nudges a few inches while it inspects its burrow, restarts its check-then-drag sequence from the beginning — every single time, indefinitely, never noticing the loop.

Both have a hardcoded assumption and both fail identically once the environment violates it, because nothing in the architecture ever re-verifies the assumption against current observation.

The equivalent: an agent executing a plan built from a system prompt or from training-era beliefs about a codebase, continuing as though a contradicting observation either wasn't received or wasn't looked at.

**What to do.** Identify which assumptions the procedure depends on, and spend a tool call verifying each against live state rather than trusting a prior.

**Honest caveat on the fix.** The textbook remedy is genuine learning — the agent's policy updated by experience. A single agent invocation has no equivalent; weights don't change mid-task. What's available at inference time is *checking*, not learning: verifying a live assumption against ground truth. That's a weaker guarantee and a different mechanism, and it's worth not conflating them. The middle position still holds — enough built-in knowledge to be useful immediately, plus enough capacity to correct it against what's actually observed.

**Check.** Which steps would proceed unchanged if their precondition were false? Those are the sphex-wasp steps.
