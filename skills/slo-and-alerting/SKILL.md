---
name: slo-and-alerting
description: Define what "working" means and decide what deserves to wake someone up — service level indicators and objectives, error budgets as a release gate, the four golden signals, and why alerting on internal causes produces fatigue. Use when setting up monitoring or alerting, writing an SLO, choosing what to instrument, tuning noisy pages, or deciding whether a condition warrants a page, a ticket, or a dashboard.
---

# SLOs and Alerting

## Purpose

Two failures show up constantly and are related.

The first: a service has an informal reliability target — "should be fast," "should be reliable" — so "is it acceptable to ship this risky change right now" has no objective answer. Teams either ship recklessly or freeze unnecessarily, and there's no shared vocabulary between whoever writes the feature and whoever gets paged for it.

The second: alerts fire on internal conditions rather than user-visible outcomes. Every component gets its own rule, most of which are redundant with an upstream symptom or fire for things that never hurt anyone. Engineers learn to triage without urgency, and the genuinely urgent page gets buried in the noise it trained them to ignore.

## The measurement stack

Three terms, precisely distinguished, because conflating them is what makes the whole thing vague:

- **SLI** — a *measured quantity*, specified precisely enough to be unambiguous. Not "latency," but "proportion of requests completing under 200 ms, measured server-side, over one-minute windows."
- **SLO** — a *target* on that indicator. "99.9% of requests under 200 ms."
- **Error budget** — the SLO's complement, treated as a spendable quantity. A 99.9% target is a budget of 0.1% of requests, tracked over a defined period.

The precision in the SLI is the load-bearing part. Server-side or client-side, which percentile, what window, what counts as a request — each choice changes the number, and an unspecified SLI produces arguments rather than decisions.

## Error budgets as a decision mechanism

The budget's purpose is to convert reliability from an argument into arithmetic.

While budget remains for the period, release velocity and risk-taking proceed normally. Once it's exhausted, deployment risk gets constrained until standing recovers. That's the whole mechanism, and its value is that both sides of the usual tension — ship faster versus break less — are consulting the same number.

Two practical notes:

- **Prefer few, simple, defensible objectives over many precise ones.** An SLO you can't win a prioritization argument with is one you didn't need.
- **100% is the wrong target.** It forecloses shipping anything, and users typically can't perceive the difference between very high reliability numbers anyway — their ISP, device, and network contribute more unreliability than the last fraction of a percent you'd be buying.

## The four golden signals

For any user-facing service, instrument at minimum:

| Signal | What to capture | Trap to avoid |
|---|---|---|
| **Latency** | Time to serve a request | Track success and failure paths **separately** — a fast error and a slow success are different signals, and averaging them hides both |
| **Traffic** | A domain-appropriate demand measure | — |
| **Errors** | Explicit failure codes, *plus* wrong answers returned with a success status, plus policy violations | The 200-with-garbage case is invisible to status-code monitoring |
| **Saturation** | Headroom on whichever resource is actually the constraint | Most systems degrade well before 100% utilization, which makes this a *leading* indicator if you set the threshold below the cliff |

**Track latency as a distribution, never a mean.** A 100 ms average happily coexists with a tail where 1% of requests take five seconds. Use histogram buckets with roughly exponential boundaries, and treat a high percentile over a short window as an early saturation signal — the tail moves before the average does.

## Two states is not enough

Most monitoring encodes exactly two conditions: working and broken. That misses where systems actually spend their time.

**Name the degraded band explicitly.** A complex system's resting state is *functioning with a shifting population of individually-survivable defects* — not a clean baseline that occasionally deviates. If your only categories are up and down, early degradation gets normalised ("it's always a bit like that") until it compounds into something a postmortem later calls obvious in hindsight.

Give the degraded-but-tolerable band its own name, its own threshold, and — importantly — its own **accepted duration**. A system in that band for an hour and one that has been there for three weeks are different situations, and only the second is a decision waiting to be made.

**Keep a census of known-but-unfixed weaknesses.** The count of closed incidents measures what already hurt you. The population of tolerated defects is a leading indicator of the next conjunction, and it's usually tracked nowhere — scattered across issue backlogs, comments, and people's heads. Making it a single visible artifact is cheap and turns "we're fine" into a claim someone has to look at.

Neither of these is an alerting rule. They're dashboards and reviews — which is exactly why they get skipped.

## What deserves a page

Run every proposed paging rule through this gate. Any "no" means it shouldn't page:

1. Will this ever legitimately need to be ignored? (If yes, it will be — and the habit spreads.)
2. Does it definitely mean users are affected **right now**?
3. Is there a real action a human can take immediately?
4. Is someone already being paged for the same underlying cause?
5. Is it a symptom, or a cause?

**Page on symptoms, not causes.** Cause-based alerting doesn't scale with dependency count or refactor rate — every internal component accretes a rule, and most fire for conditions a retry already absorbed, or for instances already drained of traffic.

Everything that fails the gate still has a home: internal telemetry to dashboards, known conditions with rote responses to tickets or automation. The point isn't to monitor less; it's to page less.

## Boundary with `resilience`

- **`resilience`** — what the code should *do* when things go wrong: timeouts, breakers, load shedding, degradation.
- **`slo-and-alerting`** — how to know whether it's working, and who to tell.

They meet at instrumentation. `resilience` says a circuit breaker's state transitions must be emitted and counted; this skill says how to decide whether that counter should page, ticket, or sit on a dashboard.

## What this will not settle

The right target number. Whether your service should be 99.9% or 99.99% is a product and cost decision — each additional nine typically costs an order of magnitude more, and the correct answer depends on what users actually notice and what the business can spend. This skill tells you how to express and use a target, not which one to pick.
