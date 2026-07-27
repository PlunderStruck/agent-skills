# Specifying the Environment

Before writing a system prompt, four things need stating and seven need classifying. Most agent projects skip both and discover the answers in production.

## PEAS: write down the four before wiring anything

**Trigger.** Starting any agent — a new tool-calling loop, a subagent, an autonomous workflow.

**What goes wrong.** Each of the four gets assumed rather than stated, and the gaps only surface under load.

- **Performance measure** — the actual criterion the agent's behaviour is judged by. Left implicit, it becomes "whatever the agent got corrected for," which is a measure assembled retroactively out of complaints.
- **Environment** — the system it's embedded in: a codebase, a filesystem, a production API, other agents and humans acting on the same substrate concurrently.
- **Actuators** — the literal channels through which it changes that environment. For an LLM agent, the tool list. The rule of thumb transfers exactly: if the program is going to recommend *Walk*, the architecture had better have legs. A policy that will reach for actions no wired-up tool can execute produces confident nonsense, not an error.
- **Sensors** — what percepts actually flow back: tool outputs, file contents, test results. Critically distinct from what the model already believes from training. A belief is not a percept.

**What to do.** Write all four out explicitly. The common defects it surfaces: one tool quietly serving as both sensor and actuator so the agent can't distinguish observing from changing; a performance measure that exists nowhere; a tool list narrower than the policy's action space.

**Check.** Can you state, in one line each, what this agent is scored on, what it can observe, and what it can change? If the scoring line is hard to write, that's the finding.

## Why behaviour can't be a table of cases

Worth knowing because it justifies the whole approach. The agent function — the mapping from percept sequences to actions — cannot generally be enumerated. Chess alone has upward of 10^150 entries; an hour of camera input needs more than 10^250,000,000,000. Too large to store, write, or learn, and even at feasible size the designer has no principled way to fill it in.

That's the argument, decades early, for behaviour coming from a compact generalizing policy rather than an enumerated dispatch table keyed on literal cases. It's also the reason "just add a rule for that" stops scaling: you are attempting the table.

## The environment taxonomy

**Trigger.** Any agent design. Treat as a checklist — these are assumptions you're already making silently.

| Dimension | The question | What it decides |
|---|---|---|
| Fully vs. **partially observable** | Do the sensors give complete access to the relevant state? | Fully observable needs no internal state. Partial requires it |
| Single vs. **multiagent** | Does anything else act on this substrate concurrently? | Whether communication and locking are rational |
| Deterministic vs. **stochastic** | Is the next state fixed by current state and action? | Whether to plan or to re-observe |
| Episodic vs. **sequential** | Does this decision affect later ones? | Whether lookahead is needed at all |
| Static vs. **dynamic** | Can it change while the agent is still deciding? | Whether observations go stale mid-deliberation |
| Discrete vs. continuous | Fixed action vocabulary or continuous signal? | Usually discrete for tool-calling; least friction |
| **Known** vs. unknown | Do we know the outcomes of our actions? | Whether exploration is required before committing |

Three warrant expansion.

**Partial observability is the normal case.** A context window is a partial, curated view of a system, and the rest isn't simultaneously present. This is the most consequential dimension and the one whose default gets assumed wrong most often — designs correct under full observability get shipped into environments that aren't. It's the direct cause of the loop failure in [architectures-and-state](architectures-and-state.md).

**Dynamic means your reads go stale.** A codebase or production system another process can modify between the agent's read and its write is dynamic *for that agent*. Long deliberation, or a cached read, exposes it to acting on information no longer true — a race condition described at the level of agent design. Not deciding is itself a decision, since the world moves on regardless.

**Known vs. unknown is about knowledge, not visibility.** These are independent. Poker is known but partially observable — you know the rules, not the other hands. An unfamiliar video game is fully observable but unknown — you see the whole screen and don't know what the buttons do. An undocumented internal endpoint, or an unfamiliar codebase's real runtime behaviour, is *unknown* however much is visible in context. This is what licenses acting purely to learn how something behaves.

**A caution on stochastic.** This dimension describes how the *environment* responds, not whether the *agent's policy* is deterministic. Sampling temperature is a separate source of nondeterminism and says nothing about the environment. Conflating them is a mistake, not a mapping. What's genuinely stochastic for a tool-calling agent: flaky tests, races, transient network failures, another process mutating shared state between your read and your write.

**Check.** Classify your environment on all seven. The worst case — partially observable, multiagent, stochastic, sequential, dynamic, and unknown at once — is closer to most production agent deployments than teams initially assume. If you classified it as fully observable and static, check that again against what another process could do while the agent is thinking.
