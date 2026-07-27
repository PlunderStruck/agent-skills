# Architectures and State

Four agent shapes of increasing capability. Each exists to fix a specific failure of the previous one, which makes them a diagnostic ladder rather than a taxonomy: identify which failure you're seeing, and it names the architecture you actually need.

## Simple reflex: the loop that can't stop

**Trigger.** An agent that decides from the current observation alone — a stateless turn, a fresh invocation per step, no memory of what it already attempted.

**What goes wrong.** A simple reflex agent selects its action from the current percept via condition-action rules, ignoring percept history. **It works only if the environment is fully observable** — only if the current observation alone determines the right move.

The worked failure is exact. A vacuum agent with a dirt sensor but no location sensor has exactly two possible percepts: dirty, clean. On *clean* it must choose a fixed direction — and if it starts on the wrong side, it fails forever. **Infinite loops are largely unavoidable for simple reflex agents in partially observable environments.**

This is the least strained mapping in any of this material. An agent that sees "tests failing," applies a fix, sees "tests failing," and applies the same fix is this agent. It cannot escape, because nothing in its percept distinguishes *this is the first attempt* from *this is the fourth*. Adding a better model does not fix it; the defect is structural.

Randomizing the action on an ambiguous percept genuinely helps and is explicitly a last resort — it converts a guaranteed loop into a probabilistic escape, not into correct behaviour.

**What to do.** Give the agent state. Anything that lets it know it already tried something rather than re-deriving that from the current observation.

**Check.** If this agent ran the same step twice, could it tell? If the only input is the current observation and the observation is identical, the answer is no, and the loop is a matter of time.

## Model-based reflex: knowing what you already did

**Trigger.** The fix for the above.

**What goes wrong without it.** The agent re-discovers facts it established minutes ago, retries dead ends, and loses the thread across a long task.

**What to do.** Maintain internal state that depends on percept history, encoding two things: how the world evolves independently of the agent, and how the agent's own actions change it. Together those are a **model**.

A detail that clarifies what belongs in state: it need not describe the world. "Driving home" is a fact about the *agent*, not the world — the same location with a different destination is a different agent-state. Intent, attempt history, and current objective all belong in state even though no sensor reports them.

In practice this is what working agents already do under other names: a scratchpad, a running todo list, a memory file, a git-status check folded into the decision. The formalism just says why it's load-bearing rather than convenient.

**Check.** Does the agent carry forward what it tried and what resulted? If its state is only "the conversation so far" and that gets truncated, the model is being silently discarded at exactly the length where it starts mattering.

## Goal-based: when the right move depends on where you're going

**Trigger.** A decision that can't be made from the current observation because it depends on the target — at a junction, whether to turn depends on the destination, and nothing in the current percept contains it.

**What to do.** Represent the desired future state explicitly and consider action *sequences* that reach it, rather than choosing one action at a time.

The gain is that the knowledge becomes explicit and swappable: change the destination and behaviour changes automatically, where a reflex agent would need every rule rewritten. This is what plan-then-execute and todo-driven modes are doing.

**Check.** If the objective changed, how much would have to be rewritten? If the answer is "the rules," the goal is implicit and should be data.

## Utility-based: when goals conflict

**Trigger.** Two acceptable outcomes and no principled way to choose — faster-but-riskier against slower-but-safer; several candidate patches that all pass.

**What goes wrong.** Goals give only a binary satisfying/unsatisfying verdict, which is inadequate the moment goals conflict and only a tradeoff is available, or when several outcomes are each achievable with some probability and likelihood must be weighed against value. "Reaches the goal" is true of both options and gives you nothing.

**What to do.** Use a **utility function** — a real-valued preference ordering over states rather than a pass/fail label, and essentially an internalization of the performance measure. A rational utility-based agent maximizes **expected** utility: the probability-weighted average across an action's possible outcomes, not the utility of the one outcome it expects.

Utility isn't required for rationality — a reflex or goal-based agent can be perfectly rational without one. It's required when the system must make a genuine tradeoff rather than confirm a goal was met. An agent generating several candidate patches and selecting the highest-scored is doing utility-based selection whether or not anyone called it that; making the scoring explicit turns an arbitrary choice into a designed one.

**Check.** When two options both satisfy the goal, what picks between them? If the answer is "whichever the model happened to produce first," that's an undesigned decision in a place that needed one.

## Learning agents, and the part that transfers

The book's learning agent has a **performance element** (whichever architecture chooses actions), a **critic** judging outcomes against a standard, a **learning element** modifying the performance element, and a **problem generator** proposing deliberately exploratory actions to surface useful feedback.

**Most of this does not port.** Nothing in a single LLM-agent session learns in this sense — weights don't update mid-task. The nearest real mechanisms are on entirely different timescales (fine-tuning between deployments) or are different in kind (memory files that persist information without changing the policy). Calling a scratchpad "learning" obscures that difference.

**One part transfers exactly, and it's the important one.** The performance standard must be fixed and external — outside the agent altogether — because *the agent must not be able to modify it to fit its own behaviour.*

A coding agent that can edit the test suite meant to be checking its work has no test suite. This is not a learning-mechanism problem; it's a problem of letting the standard live inside the agent's reach, and it applies to any architecture. See [objectives-and-rationality](objectives-and-rationality.md).

**Check.** Can the agent write to anything that grades it — tests, fixtures, eval definitions, the CI config, the assertions? If yes, the grade is self-reported.
