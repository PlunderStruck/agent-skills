# Operability

Applies to startup and shutdown paths, configuration, logging, metrics, and anything an operator needs to act on during an incident.

## Startup

**Fail fast at boot.** If a required dependency is unavailable — database unreachable, schema version wrong, mandatory config missing — refuse to finish starting and say precisely why. Starting "successfully" and then failing on the first real request is worse: it passes the deploy check, joins the load balancer, and fails in front of users instead of in front of the deploy.

**Do not accept traffic until initialization is complete.** Pool warm-up, cache priming, and connection validation must finish before the readiness probe reports healthy. A process that answers the health check while still initializing gets traffic it cannot serve.

Note the distinction between liveness and readiness. Liveness means "is this process alive"; readiness means "can it serve." Conflating them causes both premature traffic and unnecessary restarts.

## Shutdown

**Drain rather than terminate.** Stop accepting new work, finish in-flight transactions, then exit — bounded by a timeout so shutdown itself cannot hang forever.

Abrupt termination mid-transaction is a self-inflicted version of exactly the failure this skill exists to prevent. On a rolling deploy it happens to every instance in sequence.

## Configuration

**Keep environment-specific values out of the deployable artifact.** Hostnames, ports, credentials, and endpoints must live outside the code tree that gets overwritten on every deploy — otherwise every rotation becomes a rebuild.

**Separate operational config from wiring config.** Values an operator legitimately changes under pressure (a timeout, a pool size, a feature toggle) should not sit in the same file as dependency-injection wiring that operators should never touch. Mixing them turns a one-line change into a risky edit of a large file.

Name properties for what they do rather than what they are. A property named for its effect is one an operator can reason about at 3 a.m.

## Making operations scriptable

Anything an operator needs during an incident should be callable from a script or API, not only from a UI: reset a breaker, drain a pool, disable a feature, restart one failing subsystem.

A control that requires manual clicking across N servers will not get used mid-incident. And restarting one broken subsystem in place is faster than a full-fleet restart by orders of magnitude — but only if the code exposes that granularity.

## Least privilege

Nothing runs with elevated privileges beyond the single moment that requires it — binding a privileged port, for example — after which privileges drop. Credentials live outside the install directory and outside source control, readable only by the owning process's user.

## Logging for the operator, not the author

**Reserve error and critical severity for conditions requiring operator action.** A tripped breaker, an unreachable database, a full disk. Routine input-validation failures are not errors; logging them as errors trains people to ignore the error level.

**Do not ship debug or trace logging to production by default.** Strip it in the build pipeline rather than trusting a runtime flag to stay set correctly.

**Use a stable message catalog with unique codes.** Operators can then look up a code in a runbook, and "what exactly did it say" stops being ambiguous. Message text changes across releases; codes should not.

**Put a correlation ID on every line touching one logical request.** Without it, a large log is unsearchable for the one flow you care about. With it, a single grep reconstructs the request.

**Favor single-line, severity-first, columnar output.** Multi-line stack-trace-style logging defeats both human scanning and line-oriented tooling. A human eye scanning for the anomalous line needs the fields in consistent positions.

## Metrics and transparency

**Expose broadly; keep policy external.** You cannot predict which internal value will matter during a future incident, so emit generously: pool high-water marks, breaker transition counts, cache hit rates, per-dependency error counts, session creation rate, queue depth. But keep thresholds and alerting rules in configuration outside the application, because policy changes far more often than instrumentation should.

**Four audiences, none served by one stream.** Historical trend analysis needs a queryable store rather than log files. Capacity projection is built from that history and is invalidated by every release. Present status needs more than up/down — "flooded but technically responding" is its own state and the one that precedes an outage. Live incident debugging needs thread dumps and current metrics.

**Tune alert thresholds from observed data.** A threshold set by guess is either so loose it never fires or so noisy it trains operators to dismiss it — and the dismissal habit carries over to the real signal.

**A metric nobody reviews is theater.** Pair exposed metrics with a recurring review, and retire ones that have stopped producing decisions.

## The dependency between patterns and transparency

Every containment pattern depends on transparency to be worth anything. A breaker that does not log its transitions, a bulkhead whose pool metrics are not exposed, a fail-fast path that does not surface *why* it rejected — each prevents the outage but leaves nobody able to confirm it is working, or to learn from the near-miss.

When adding any pattern from this skill, add its instrumentation in the same change.

## What to check in the code

- Does startup validate required dependencies and refuse to proceed, or start optimistically?
- Does readiness report healthy before initialization completes?
- Is there a drain path on shutdown, and is it bounded?
- Do environment-specific values live outside the deployed artifact?
- Can an operator reset a breaker or disable a feature without a deploy?
- Is error severity reserved for actionable conditions?
- Does every log line in a request path carry a correlation ID?
- Are breaker transitions, pool high-water marks, and cache hit rates actually emitted?
