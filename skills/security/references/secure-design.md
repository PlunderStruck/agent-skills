# Secure Design

Design-altitude decisions that make whole classes of code-level bug hard to write. Three ideas run underneath everything here.

**Fail-safe and fail-open are different designs**, and "reliable" is not automatically "secure" — a control permitted to fail open can be forced open by an attacker who simply takes its dependency offline.

**Your recovery path is attack surface.** Revocation, rollback, breakglass, and backups are levers an attacker can pull if they aren't held to the same standard as what they protect.

**Secure-by-construction beats secure-by-review**, because review doesn't scale and doesn't survive turnover.

## Least privilege as architecture

**Replace general-purpose admin access with named, single-purpose APIs.** An SSH shell or a generic RPC surface means one compromised credential equals everything reachable through it — and it's unauditable, because an interactive session can bypass command logging (shelling out from inside an editor). Narrow APIs — push this config by hash, restart this process — bound the damage and produce structured audit entries instead of free text.

Sign configuration independently of the pipeline that pushes it, so compromising the pusher isn't equivalent to compromising the target.

**Eliminate standing grants.** A permanent admin group or an always-valid API key is ambient authority: unused most of the time, the first thing an attacker checks after landing, and invisible until an incident forces an audit. Convert to temporary, justification-backed access tied to a ticket.

Treat repeated requests for the same access as a signal to build a narrower permanent API — not a reason to leave a standing exception.

**Gate high-blast-radius actions on multi-party authorization** — and gate the *live action*, not just the deploy. Code review saw the code that could someday trigger a bulk delete; it never saw this specific invocation. The approval UI must show the real target and parameters rather than an opaque ID, or the second person is rubber-stamping.

**Breakglass needs monitoring or it becomes a backdoor.** Restrict who can invoke it, log and review every use, and drill it on a schedule so it works the one time it matters. Routine use is a signal that a safe API is missing.

**Graduate denial messages by the caller's privilege.** Overly informative denials are a reconnaissance oracle telling an attacker exactly what unlocks the resource. Overly opaque ones drive users to file tickets until someone grants broad access to stop the noise — so the "safe" choice degrades access control through a social path. Unprivileged callers get a bare denial; trusted callers get a usable hint.

## Design for understandability

**Centralize any property enforced in many places.** An auth check copy-pasted per handler, or escaping reimplemented per team, means nobody can verify the property holds everywhere and every copy is a fresh chance to get it subtly wrong.

Move it to one enforcement point every request passes through by construction. The test: **how many files must you read to prove the property holds system-wide?** Close to one, or you have a finding.

**Wrap primitives that carry invariants.** A parameter typed as a bare string, where only some values are safe downstream, makes the invariant implicit — so proving it holds means re-checking every caller, which never happens consistently. A type whose only constructors enforce the invariant converts that into proving one constructor correct.

Then check: does the sensitive sink accept the bare primitive, or only the wrapped type? If only the wrapped type, the invariant is structural rather than remembered.

**Watch what shares a trust boundary.** Components sharing a process, database, or origin "for now" silently expand the trusted computing base for every sensitive property to everything on that boundary — so a bug in an unrelated feature becomes a path to data its code never touched. For each sensitive property, enumerate the components whose bug could break it. That set should be small and deliberate.

## Design for change

**Make the emergency path the normal path with the rate limit raised.** A separate emergency process is unfamiliar and undertested exactly when speed matters, which is how emergency rollouts cause their own outages. Same canarying, same monitoring, same rollback — just faster, with the limit behind an independently changeable policy.

**Prevent rollback past a security floor.** A naive version denylist lets an attacker unzip downward, rolling back one version at a time until landing on an uncovered hole. Track a monotonically increasing security version alongside a minimum-acceptable floor, and **store the floor outside the component** so a downgraded copy can't self-report a fake one.

## Fail-safe versus fail-open

For every control that can fail to load or respond — ACL evaluation, certificate validation, a revocation check — **classify it as security-enforcing or not before deciding its failure mode.**

Non-enforcing paths can fail open. Enforcing paths must fail closed, and the engineering effort then goes into making the closed-failing control highly available — redundancy, cached last-known-good state, a short dependency list — rather than relaxing the failure mode.

For every dependency of a security control, get a written answer to "what happens if this is unreachable," then chaos-test it and confirm the system degrades to deny.

**Rank services by security criticality before you need the ranking.** Shared infrastructure protecting backends of very different sensitivity will apply one degradation policy to a login endpoint and a static asset alike. Decide in advance whether an outage or degraded security — login without a second factor — is worse *for each service*. The answer genuinely differs, and it shouldn't be implicit in whichever code path happens to error first.

**Treat any successful crossing of a compartment boundary as a bug.** Run scheduled tests that attempt an impassable crossing and alert loudly if one succeeds. Here the *absence* of a failure is the signal something is wrong.

## Design for recovery

**Distribute revocation.** A central revocation lookup is a single point that, taken offline, either fails closed (denial of service) or fails open (silently un-revoking everything) — both attacker-triggerable. Cache revocation data locally so nodes act on their best available copy, and **reject any revocation update broad enough to revoke its own issuing infrastructure**, so partial compromise can't be escalated into total revocation.

**Audit which clock a security decision trusts.** Anything depending on wall-clock time — certificate expiry, a time-bounded credential, a replay window — can be manipulated by anyone who can influence perceived time. Where the real property is "has this been superseded" rather than "has calendar time passed," use monotonic counters or epochs, rate-limited so a compromised issuer can't fast-forward them.

**Protect backups as strongly as production data.** Recovery is the least-exercised code in the system and runs under the most pressure. If backups aren't integrity-protected, a poisoned backup is faithfully restored and the recovery defeats itself. Sign on write, verify on restore, and confirm restore tooling rejects a deliberately tampered backup in test.

Make restoration granular enough to restore only the affected subset, and use routine data migrations as regular exercise so the machinery doesn't atrophy.

**Emergency access must not depend on what it's a fallback for.** Emergency tooling commonly reuses the SSO, DNS, or chat infrastructure it's meant to survive the loss of. Map every dependency and confirm none fail together with the outage class it exists for. Keep it in continuous low-stakes use so it's muscle memory rather than learned during a crisis — while avoiding the opposite mistake of long-lived standing credentials.

## Denial of service

**Layer defenses cheapest-and-earliest to most-expensive-and-latest** — edge filtering, geographic dispersal, then application-layer shedding near the bottleneck. Decide per service what graceful degradation means (stale or read-only content) rather than having one global fail behavior.

**Make mitigation responses fail static.** Naive reactive blocking — engage and disengage as traffic shifts — lets an attacker A/B test their way around the detection boundary in real time. Once engaged, hold for a fixed window. Canary automated policy changes on a small traffic slice first, even briefly.

## Code, test, and deploy

**Block new usage of an unsafe API rather than attempting a big-bang migration.** A lint or compiler annotation refuses new callers while existing usage persists as a named, tracked exemption list that must shrink. The list becomes the security backlog, and it's actionable in a way "we should migrate someday" never is.

**Never combine refactoring with functional change in one commit**, especially near a security control. A behavior change hidden inside a large "just a refactor" diff — two error branches swapped — is exactly what reviewers pattern-match past. Raise coverage first so behavior changes get caught mechanically. Mutation testing validates whether that coverage would actually catch one.

**Fuzz every parser that consumes attacker-influenced bytes** — file formats, protocol decoders, config parsers, not only web input. Hand-written tests cover the cases the author imagined; the crashing input is the one nobody thought of. Run continuously, seeded with a real corpus, alongside targeted unit tests.

**Don't populate test environments with production data.** Realistic test data is real sensitive data in a less-guarded place, reachable by anyone with staging access — a second copy of exactly what access control exists to protect. "We need real data" is a signal to invest in synthetic generation, not to widen access.

**Hold static analysis to a false-positive budget.** A noisy tool trains developers to dismiss findings reflexively, including the true ones. Track the dismissal rate as a first-class measure of tool health, not just the count of bugs found.

**Require signed provenance at a single deploy choke point.** Trusting an artifact because of where it came from means anyone who can influence an upstream step gets an untrusted binary treated as trusted, without ever touching the deploy target. Attach signed provenance — source, build config, pipeline identity — and enforce policy at one gate every deploy must pass.

**Make builds hermetic and keep signing keys out of them.** Build-time code execution is remote code execution by design: a compromised dependency's install step can exfiltrate keys or inject code into the artifact, upstream of every other control. Pin every input, remove ad hoc network access, and sign in a separate, more privileged step.

Verify hermeticity by running the build with network access removed and confirming it still succeeds.
