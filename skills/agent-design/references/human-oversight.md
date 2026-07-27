# Human Oversight

Bainbridge's 1983 argument, which describes supervised autonomy more precisely than most current writing on agents. The claim is not that automation is bad. It is that **automating the routine part of a task systematically degrades the human's ability to handle the remainder** — and the remainder is exactly what gets handed back when things go wrong.

## The impossible monitoring task

**Trigger.** Any design whose safety story is "a human reviews the output."

**What goes wrong.** The system was deployed *because it does the job better than the person now asked to check it*. Two consequences follow, and they compound.

**The monitor needs to know what correct looks like.** For anything with a complex trajectory that requires either specific training or a purpose-built display. Without it, "does this look right?" has no referent.

**And where the decision can be fully specified, the machine makes it faster, across more dimensions, against more precisely stated criteria than a person can.** So there is no way to verify in real time that it followed its rules correctly. What remains is meta-level supervision: judging whether outputs seem *acceptable*.

Then the closing move:

> If the computer is being used to make the decisions because human judgement and intuitive reasoning are not adequate in this context, then which of the decisions is to be accepted? The human monitor has been given an impossible task.

For LLM agents this is worse than in Bainbridge's setting, not better. Plausibility assessment is the one remaining channel, and fluent generated output is specifically hard to distinguish from correct output at the surface. The reviewer is left with the exact discrimination the medium defeats.

**What to do.** Stop treating human review as a safety property and start treating it as one weak signal. Where real assurance is needed it has to come from something that doesn't require out-reasoning the agent in real time — effect-checks, invariants, tests the agent can't edit, staged rollout, reversibility.

**Check.** If the agent were wrong in a plausible-looking way, what in this system would catch it? If the honest answer is "the reviewer would notice," you're relying on the thing this argument says isn't available.

## Sustained attention is not available

**Trigger.** Approval prompts, permission gates, a stream of agent actions presented for confirmation.

**What goes wrong.** Vigilance research is unambiguous: a person cannot maintain effective attention to a source where almost nothing happens for more than about half an hour, however motivated. Monitoring for rare abnormalities is therefore not something a human can do — it has to be automated, which raises the question of who watches the watcher.

The classic countermeasure of requiring a log to force engagement doesn't work either: people write down numbers without noticing what they are.

**What to do.** Reserve interruption for the rare and consequential, and let everything routine pass without a prompt. A gate that fires constantly trains the reflex that defeats it — the same mechanism that makes a routine confirmation dialog inert (see `interaction-and-error`).

**Check.** How many approvals does this produce per hour, and what fraction are ever refused? A high-volume, near-zero-refusal gate is measuring compliance, not providing oversight.

## Skill erosion and the takeover paradox

**Trigger.** Any escalation path — the agent hands off when it gets stuck, hits a limit, or encounters something unusual.

**What goes wrong.** Skills decay without use. An engineer who spent a year reviewing generated work rather than doing it is, on the metrics that matter for takeover, no longer practised. And two kinds of knowledge degrade separately:

**Long-term knowledge** — retrieval depends on frequency of use, and this knowledge only develops through use *plus feedback about whether it worked*. Taught without practice it doesn't take. Bainbridge's warning names the succession problem exactly: current automated systems are riding on the skills of people trained before automation, and later generations cannot be assumed to have them.

**Working storage** — decisions are made against a model of the *current* state, and what an operator holds isn't raw data but accumulated predictions about where things are heading. It takes time to build. Manual operators would arrive half an hour before a shift to acquire the feel of the process. Someone forced to act immediately can act only on minimal information.

**The paradox:** handoff happens precisely when something is wrong, so unusual action is required. The person therefore needs to be *more* skilled and *less* loaded than average at the exact moment automation has left them less practised and more loaded.

**What to do.** Treat handoff as requiring context transfer, not just control transfer — what was attempted, what was observed, what state things are in. An escalation that says "I got stuck, over to you" hands someone the hardest version of the task with none of the accumulated model.

**Check.** At the moment of handoff, does the human receive the agent's working state, or just the fact that it stopped?

## Failures must be loud

**Trigger.** Retry logic, fallback paths, degraded modes, error recovery inside an agent.

**What goes wrong.** Catastrophic breaks are easy to spot. The dangerous case is **automation that masks a developing failure by compensating for it** — the controller absorbs the drift, so nothing looks wrong until the deviation exceeds what compensation can cover, at which point it's beyond recovery.

Bainbridge states the inversion plainly: graceful degradation, listed elsewhere as a virtue, is **not** something to aim for in automation, because it creates exactly this monitoring problem. Automatic systems should fail obviously.

The agent equivalents: silent retries, silently falling back to a weaker tool or model, silently truncating context, quietly working around a failing dependency, catching an exception and continuing with partial results.

**What to do.** Make every degradation visible as an event, not just a log line. The resilience instinct — absorb the error, keep going — is in genuine tension with supervisability, and the tradeoff should be made explicitly rather than assumed away.

**Check.** If this agent silently degraded right now, how would anyone find out? If the answer is a log line nobody reads, that's camouflage.

## Legibility is a real constraint, with a real cost

**Trigger.** Choosing between an efficient internal method and one a human can follow.

**What goes wrong.** Bainbridge's requirement is stronger than "keep logs": if a human must supervise the details, the system has to make decisions **using methods and criteria, and at a rate, that the operator can follow — even where that is not the technically most efficient approach.** Without shared intermediate steps, an operator who disagrees cannot trace back to find where they diverge. They can reject the output, but they cannot locate the error.

**What to do.** Where supervision genuinely matters, accept a less efficient but followable method as the cost of it. Where it doesn't, don't pay that cost — but make the choice deliberately.

**Check.** When a reviewer disagrees with the agent's conclusion, can they find the specific step they'd have taken differently? If the only artifact is a final answer, disagreement has nowhere to go.

## Check effects, not strategy

**Trigger.** Constraining an agent — allowed actions, required sequences, forbidden operations.

**What goes wrong.** Constraining *method* is brittle and fights the reason you deployed the agent, which was to find routes you didn't specify. Bainbridge's rule: except where a specific sequence genuinely must be followed, place checks on the **effects** of actions rather than the actions themselves, because effect-checks make no assumptions about the strategy used.

She pairs this with a mechanism worth carrying: manual operators self-correct within seconds because they get feedback on their actions, while people *setting up and monitoring* automation make the same errors without that feedback — so it has to be designed in deliberately.

**What to do.** Verify outcomes — tests green, no writes outside scope, invariants hold, diff within expected bounds — rather than prescribing routes. And surface the consequences of actions back to the agent, since an agent acting without seeing effects produces errors it would otherwise have caught itself.

**Check.** Are your guardrails written against what the agent *did* or what it *changed*? The first is evadable and brittle; the second isn't.

## Distrust doubles the load

**Trigger.** "We'll just review everything carefully" as the mitigation.

**What goes wrong.** Operators who distrust automation end up monitoring both the process *and* the automation — the load compounds rather than reverting to the un-automated baseline. (Sarter & Woods, 1995, via the 2020 retrospective.)

Bainbridge reports the matching empirical result directly: system performance was **worse** with computer aiding, because the operator made the decisions anyway and checking the machine's work was pure added load.

So the two failure directions aren't symmetric alternatives on one dial. Complacency and distrust are distinct failures with distinct costs, and "be more careful" moves you from one to the other rather than to safety.

Two related traps: artificially injecting failures to keep the monitor engaged destroys trust and is self-defeating. And observed trust tends to be *load-dependent* rather than accuracy-dependent — under light load people let the system carry the work; under heavy load they step in and override. That's backwards from optimal, since heavy load is when help is worth most.

**What to do.** If an agent's output requires full re-derivation to trust, it is net-negative and should either be made verifiable more cheaply or not used for that task. "Review it carefully" is not a design.

**Check.** Does using this agent take less total human effort than not using it — counting verification? If nobody has measured that, it's an assumption.

## Undocumented autonomy with no override

**Trigger.** Any automated behaviour the human isn't told about.

**What goes wrong.** The 737 MAX pattern reduces to three findings, independent of any statistical argument: operators were unaware the automated function existed, it wasn't documented, and there was no override path. The workaround that saved one aircraft the day before its fatal flight was an undocumented procedure known only to experienced crews.

**What to do.** Every autonomous behaviour needs to be discoverable, documented, and overridable. And the human must know *which tasks the system is handling and how* — otherwise the situation degenerates the way a human team does when responsibility is unassigned.

**Check.** Can the user name what this agent does without being asked? Can they turn a specific behaviour off without turning the whole thing off?

## The variety bound, and the final irony

**Two closing claims worth holding together.**

**Autonomy holds where variety is low and frequency is high** — a narrow, repetitive task space — and degrades as variety rises. It does not generalize to open-ended domains by adding capability. This predicts where agent autonomy is actually safe to grant, and it predicts that a "transitional" mixed-autonomy phase never ends, because variety keeps growing.

**And the irony that inverts most people's intuition:**

> It is the most successful automated systems, with rare need for manual intervention, which may need the greatest investment in human operator training.

Better agent → rarer handoff → more decayed the skill of whoever takes over → *more* investment the escalation path requires. Reliability and supervisability move in opposite directions. Budgeting for oversight in inverse proportion to how well the agent works is the mistake this predicts, and it's the default.
