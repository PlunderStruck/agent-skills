# Extraction: security-design

**Source:** Building Secure and Reliable Systems — Google

> Operational rules distilled from the source, written in our own words. This is the intermediate layer between the source text and the published skill — denser than the skill, and the place detail survives for re-distillation.

---

# Design-Altitude Security Rules (from *Building Secure and Reliable Systems*)

Three throughlines run under everything below. First, **fail-safe and fail-open are different designs, and "reliable" isn't automatically "secure"** — a control allowed to fail open can be forced open by an attacker who just DoSes it. Second, **your recovery path is itself attack surface** — revocation, rollback, breakglass, and backups are all levers an attacker can pull if they aren't held to the same rigor as what they protect. Third, **secure-by-construction beats secure-by-review** — make the unsafe thing impossible to express (a type that can't hold unvalidated data, an API with no unauthenticated path), because review doesn't scale and doesn't survive turnover.

## 1. Least Privilege as Architecture, Not Policy

**Trigger:** An admin/automation channel with interactive or POSIX-level access — SSH with a shell, "run as root," a generic RPC surface that lets the caller do anything the target process can do.
**What goes wrong:** One compromised credential or automation bug equals compromise of everything reachable through it. It's also unauditable: an attacker in an interactive session can bypass command logging (e.g., shelling out from inside an editor), so the audit trail lies.
**What to do:** Replace the general shell/POSIX surface with a small number of named, single-purpose admin APIs (push this config by hash, restart this process, delete this record). Sign configs independently of the automation that pushes them, so compromising the pipeline isn't equivalent to compromising the target.
**How to verify:** Name the worst single action each admin credential can take; confirm it's bounded. Audit entries should read as structured actions, not free text. Confirm a compromised pusher can't forge a signature the target trusts.

**Trigger:** Any standing credential, group membership, or role grant — a permanent "prod-admin" group, an always-valid API key, a role nobody remembers assigning.
**What goes wrong:** Standing access is ambient authority — unused most of the time, the first thing an attacker checks post-compromise, invisible until an incident forces an audit.
**What to do:** Default-deny; convert standing grants into temporary, justification-backed access tied to a ticket/case ID, issued on demand or via scheduled rotation. Treat repeated requests for the same access as a signal to promote it into a narrower permanent API, not a reason to leave it a manual exception.
**How to verify:** "Who has standing access to X" should return a small, explicable list, each entry traceable to a still-open justification. Alert on generic/boilerplate justification text — a sign the control is being routed around.

**Trigger:** A single human or single pipeline can unilaterally trigger a high-blast-radius action — bulk delete, policy change, rollback of unknown scope.
**What goes wrong:** One phished engineer or buggy pipeline executes it alone; ordinary code review saw the code that *could* trigger the action someday, never the specific live invocation.
**What to do:** Gate the live action, not just the deploy, behind multi-party authorization — a second person approves the actual request with real context. Where approvers might share a compromised device fleet, add an out-of-band channel (a separate hardened device class) so one compromised laptop population can't rubber-stamp itself.
**How to verify:** Approval UI should show the real target and parameters, not an opaque ID. Test whether an approver can say no without friction — if the social dynamics don't allow it, the control is theater.

**Trigger:** A breakglass/emergency-override path that bypasses normal authorization.
**What goes wrong:** Undermonitored breakglass becomes a permanent backdoor. Routine use signals a missing safe API; unreviewed use lets it decay back into ambient authority.
**What to do:** Restrict who can invoke it and, where relevant, from where. Log and mandatorily review every use. Drill it on a schedule so it works the one time it's needed.
**How to verify:** Usage trends near zero, with 100% of uses reviewed within days, and a recent successful drill.

**Trigger:** An authorization-denial code path (a 403, an access-check failure).
**What goes wrong:** Overly informative denials become a reconnaissance oracle (attacker learns exactly what unlocks the resource). Overly opaque ones drive users to file tickets until someone grants broad access "to stop the noise."
**What to do:** Graduate disclosure by the caller's own privilege — unprivileged callers get a bare denial; semi-privileged callers get an opaque token to self-serve escalation; only trusted callers get a human-readable hint.
**How to verify:** Probe denials at each privilege tier for leaks at the lowest one. Track support-ticket volume from denials as a proxy for whether self-service escalation works.

## 2. Design for Understandability

**Trigger:** A security- or availability-critical property enforced independently in many places — an auth check copy-pasted per handler, escaping reimplemented per team.
**What goes wrong:** No one can verify the property holds everywhere; each independent copy is a fresh chance to get it subtly wrong, and reviewers would have to re-audit every call site forever, which in practice they don't.
**What to do:** Move responsibility into one centralized enforcement point every request passes through by construction — authn/authz/logging as framework interceptors, escaping as a property of the template system — so proving the property means reading one implementation, not N.
**How to verify:** Ask how many files you'd need to read to prove the property holds system-wide. It should be close to one, not "every service."

**Trigger:** A function/API where the parameter is a bare primitive (string, int) but only some values are valid or safe downstream — URLs, identifiers, money, durations.
**What goes wrong:** The invariant is implicit, so proving it holds means re-checking every caller that could produce the value — work that never happens consistently — until an invalid value reaches the sink.
**What to do:** Wrap the value in a type whose only constructors already enforce the invariant (a validating parse function, a literal-only constructor, a sanitizing builder). This turns "prove no caller violates the invariant" into "prove this one constructor is correct" — the type system does the review permanently instead of a human doing it once per call site.
**How to verify:** Search for the bare primitive at sensitive sinks; if the sink only accepts the wrapped type, the invariant is structural. The count of places that manually re-validate should trend to zero.

**Trigger:** Multiple components sharing a process, database, or network origin "for now," where at least one handles sensitive data.
**What goes wrong:** The trusted computing base for any sensitive property silently expands to everything sharing that boundary. A bug in an unrelated feature becomes a path to data the "owning" code never touched.
**What to do:** Decompose along the property you're protecting — a separate backend/datastore for the sensitive function, a separate origin for pages touching sensitive data — and pass identity through explicitly instead of letting a shared frontend assert authority for any user.
**How to verify:** For each sensitive property, enumerate the actual set of components whose bug could break it. It should be small and intentional, not "the whole app."

## 3. Design for a Changing Landscape

**Trigger:** A separate "emergency patch process" that differs from the normal deploy pipeline.
**What goes wrong:** A rarely-exercised emergency path is unfamiliar and undertested exactly when speed matters most — which is how emergency rollouts cause their own outages.
**What to do:** Build the emergency path as the normal pipeline with its rate limiter turned up — same canarying, monitoring, and rollback, just faster. Put the rate limit behind an independently changeable policy so speed can be tuned without touching the push system.
**How to verify:** Confirm the "emergency" path shares code with the normal one and gets exercised outside real emergencies.

**Trigger:** Any self-updating component (agent, firmware, base image) that also supports rollback.
**What goes wrong:** A naive version denylist lets an attacker "unzip" downward — roll back one version at a time until landing on a hole the list doesn't cover. Rollback in general reintroduces every previously patched vulnerability unless something prevents it.
**What to do:** Track a monotonically increasing security version number separately from a minimum-acceptable floor, stored outside the component so a downgraded copy can't self-report a fake floor.
**How to verify:** Script a downgrade past the intended floor in test and confirm rejection.

**Trigger:** Rolling out a fix under vendor/researcher embargo.
**What goes wrong:** Teams either break embargo to get ahead of disclosure, or wait and rush an unvalidated patch under public pressure — both raise risk.
**What to do:** Negotiate a synchronized announcement date in advance; patch high-value systems quietly where the embargo allows; pre-stage accelerated (never skipped) validation so disclosure day runs a rehearsed plan.
**How to verify:** A written plan mapping disclosure-day actions to owners should exist before disclosure, not get assembled after.

## 4. Design for Resilience: Fail-Safe and Fail-Open Are Different Decisions

**Trigger:** Any control that can itself fail to load or respond — ACL evaluation, cert validation, a revocation check.
**What goes wrong:** Reliability instinct says "serve anyway" when the control is down. For a security-enforcing control, failing open means an attacker can force a bypass just by knocking its dependency offline — the reliability fix becomes the vulnerability.
**What to do:** Classify each control as security-enforcing or not before deciding its failure mode. Non-enforcing paths can fail open. Enforcing paths must fail closed — then the engineering effort shifts to making the closed-failing control highly available (redundant instances, cached last-known-good state, minimal dependencies) rather than relaxing its failure mode.
**How to verify:** For every dependency of a security control, get an explicit written answer to "what happens if this is unreachable." Chaos-test it and confirm the system degrades to deny, not allow.

**Trigger:** A system decomposed into compartments (cells, shards, tenants, regions) for reliability.
**What goes wrong:** Compartments sized wrong in either direction — too coarse and a breach or overload in one takes everything down; too fine and normal operation becomes complex enough to hide bugs.
**What to do:** Compartmentalize along role, location, and time, not just capacity — and treat any successful crossing of a compartment boundary as a bug by default.
**How to verify:** Run scheduled tests that attempt to cross a boundary that should be impassable; alert loudly if one ever succeeds — the absence of a failure is the signal something's wrong.

**Trigger:** Shared infrastructure (a rate limiter, load balancer, sandbox) protecting backends of very different sensitivity.
**What goes wrong:** One degradation policy treats a login endpoint the same as a static asset. Under real load or attack, there's no way to prioritize the security-relevant path.
**What to do:** Rank services by criticality on a security dimension, not just traffic value, before you need the ranking — and decide in advance whether outage or degraded security (e.g., login without a second factor) is worse for each service. The right answer genuinely differs by service; don't let it be implicit in whichever code path errors first.
**How to verify:** The degradation ladder per service should be a written, reviewed artifact, not something inferred after an incident.

## 5. Design for Recovery — Why the Recovery Path Is Itself Attack Surface

**Trigger:** A revocation mechanism (cert revocation, session kill, credential denylist) served from a central lookup.
**What goes wrong:** The revocation service is a single point that, if knocked offline, either takes the system down (fail closed) or silently un-revokes everything (fail open) — both attacker-triggerable. The recovery mechanism becomes a lever against you.
**What to do:** Distribute revocation data — cache a revocation list locally, let nodes act on their best available copy — and reject any revocation update broad enough to revoke its own issuing infrastructure, so a partial compromise can't be used to revoke everything.
**How to verify:** Take the revocation service down in a chaos test and confirm the intended degrade direction. Submit an overly broad revocation update in test and confirm rejection.

**Trigger:** A security decision that depends on wall-clock time — certificate expiry, a time-bounded credential, a replay window.
**What goes wrong:** Anyone who can influence perceived time (spoofed NTP, a frozen clock) can revive expired credentials or invalidate valid ones — invisible until exactly the wrong moment.
**What to do:** Where the real property is "has this been superseded" rather than "has calendar time passed," replace wall-clock dependence with monotonic counters or epoch numbers, rate-limited so a compromised issuer can't fast-forward them.
**How to verify:** Audit every expiry check for which clock source it trusts; test against a deliberately skewed clock in non-prod.

**Trigger:** Backup/restore tooling and the path used to recover from corruption or an incident.
**What goes wrong:** Recovery is the least-exercised code in the system, most likely to fail under the most pressure. If backups aren't integrity-protected as strongly as primary data, a poisoned backup gets faithfully restored, defeating the recovery.
**What to do:** Give backups the same integrity guarantees as production data — signed on write, verified on restore. Make restoration granular enough to isolate and restore only the affected subset. Treat routine data migrations as regular exercise of the same restore machinery so it doesn't atrophy.
**How to verify:** Run a scheduled restore drill and measure success and time-to-restore. Confirm restore tooling rejects a deliberately tampered backup in test.

**Trigger:** Emergency remote-access tooling meant to work when normal infrastructure (SSO, chat, DNS) is down.
**What goes wrong:** Emergency access commonly reuses the infrastructure it's meant to be a fallback for, so it fails during exactly the outage it exists for. The opposite mistake — long-lived credentials "just in case" — creates a standing liability the rest of the time.
**What to do:** Build a minimal, independently-provisioned credential and communication path with its own short dependency list, kept in continuous low-stakes use by responders so it's muscle memory, not learned during a crisis.
**How to verify:** Map every dependency of the emergency path; confirm none fail together with the outage class it's meant to survive. Check drill cadence and pass rate.

## 6. Mitigating Denial of Service

**Trigger:** Capacity planning for a public-facing or high-value internal service.
**What goes wrong:** A single-layer defense (a bigger fleet, a firewall) either can't scale to attacker resources or drops legitimate traffic indiscriminately with attack traffic.
**What to do:** Layer defenses from cheapest/earliest to most expensive/latest — edge filtering, geographic dispersal of load, then application-layer shedding close to the bottleneck — and decide per service what graceful degradation looks like (serve stale/read-only content) instead of one global fail behavior.
**How to verify:** Load-test the full layered pipeline, not just the backend. Confirm degraded-mode behavior is a designed, tested path, not an accident of whichever component errors first.

**Trigger:** An automated abuse-response system that blocks IPs or issues challenges based on live traffic signals.
**What goes wrong:** Naive reactive blocking — react and un-react as traffic shifts — lets an attacker A/B test their way around the detection boundary in real time.
**What to do:** Make mitigation responses "fail static": once engaged, hold for a fixed window instead of reacting to every sample. Canary automated policy changes on a small traffic slice before wide rollout, even for a few seconds.
**How to verify:** Confirm mitigation state is held, not flapping with each observation. Confirm the response pipeline itself has a canary step in its own deploy path.

## 7. Writing Code

**Trigger:** A hand-rolled admin/debug capability exposed "temporarily" through a broad primitive — dynamic eval, unrestricted shell exec, unconstrained reflection.
**What goes wrong:** Temporary broad capabilities outlive their justification and become permanent, unaudited power nobody remembers granting.
**What to do:** Prefer strongly typed, narrow wrapper types over bare primitives for anything with a real invariant — units, identifiers, validated values — so misuse is a compile error, not a runtime surprise. Same secure-by-construction move as Section 2, applied at the statement level.
**How to verify:** Grep for the bare primitive type at security-sensitive call sites; confirm new code can't bypass the typed wrapper without an explicit, reviewed escape hatch.

**Trigger:** Legacy code with an unsafe-but-flexible API living alongside a newer, safer one.
**What goes wrong:** A big-bang migration stalls forever; leaving both live indefinitely lets the unsafe path keep accumulating new callers.
**What to do:** Block *new* usage of the unsafe API at review time (a lint/compiler annotation), while existing usage persists as a tracked, named exemption list that must shrink — the list itself becomes the security backlog.
**How to verify:** Exemption count should trend down and require named approval to grow. CI should fail on new, unlisted unsafe usage.

**Trigger:** A refactoring commit that also changes behavior, especially near a security control.
**What goes wrong:** A refactor that quietly alters behavior (two error-handling branches swapped) hides inside a large diff; reviewers pattern-match on "just a refactor" and wave it through.
**What to do:** Never combine refactoring and functional changes in one commit. Raise test coverage before refactoring so behavior changes get caught mechanically.
**How to verify:** Scan history for diffs mixing structure and logic changes in security-sensitive files. Mutation testing — deliberately break a condition, confirm a test fails — validates whether coverage would actually catch a hidden change.

## 8. Testing Code

**Trigger:** Any parser, deserializer, or format handler consuming attacker-influenced bytes — file formats, protocol decoders, config parsers, not just web input.
**What goes wrong:** Hand-written unit tests only cover cases the author imagined; the actual crash-triggering input is something nobody thought of, and it ships.
**What to do:** Put a fuzz target on the parsing function's boundary, seeded with a small real-world corpus, run continuously rather than as a one-time gate — complementing targeted unit tests, not replacing them.
**How to verify:** Track the coverage jump from adding a seed corpus — a real increase signals the target is reaching interesting code. Confirm crashes get deduplicated and triaged, not just accumulated.

**Trigger:** An integration/staging environment provisioned with a copy of production data because it's "more realistic."
**What goes wrong:** Realistic test data is also real sensitive data, now reachable by anyone with test-environment access — a second, less-guarded copy of exactly what access control was built to protect.
**What to do:** Use synthetic or anonymized data for integration tests. Treat "we need real data" as a signal to invest in synthetic data generation, not a reason to widen access.
**How to verify:** Audit who can reach test/staging datastores; confirm the set isn't a superset that defeats production access controls.

**Trigger:** Static analysis or linting wired into code review.
**What goes wrong:** A high false-positive rate trains developers to reflexively dismiss warnings — including the true positives.
**What to do:** Hold static analysis to an explicit false-positive budget, and prefer tools that suggest or auto-apply the exact fix over ones that only flag.
**How to verify:** Track the dismissal rate on findings as a first-class tool-health metric, not just the raw count of bugs found.

## 9. Deploying Code

**Trigger:** A deploy pipeline that trusts an artifact based on where it came from, without cryptographic evidence of how it was built.
**What goes wrong:** Anyone who can influence an upstream step — source repo, build script, a dependency — can get an untrusted binary treated as trusted without touching the deploy target at all.
**What to do:** Attach signed binary provenance to every artifact (source, build config, pipeline identity) and enforce policy at a single deployment choke point every deploy must pass through — peer-reviewed source, official pipeline, clean recent scan — instead of trusting artifacts by convention.
**How to verify:** Attempt to deploy an artifact failing one policy condition in test and confirm rejection. Confirm no path bypasses the choke point, such as direct access to worker nodes.

**Trigger:** A build process that can reach the network or read local secrets mid-build — arbitrary build scripts, unpinned dependencies fetched at build time.
**What goes wrong:** Build-time code execution is remote code execution by design: a compromised dependency's install step can exfiltrate signing keys or inject code into the artifact, upstream of every other control.
**What to do:** Make builds hermetic — every input pinned and specified up front, no ad hoc network access mid-build — and keep signing keys entirely out of the build environment, applying signatures only in a separate, more privileged step.
**How to verify:** Run the build with network access removed and confirm it still succeeds, proving hermeticity. Confirm the signing key isn't readable from the build sandbox's credentials.

**Trigger:** A deploy-time breakglass override used for incident response.
**What goes wrong:** Same pattern as access breakglass — necessary during real outages, but decays into an unaudited bypass of every deploy control if usage isn't rare and reviewed.
**What to do:** Keep the override, but pair it with mandatory alerting on every use and fast, unavoidable audit review. Treat rising usage as a signal something else is broken.
**How to verify:** Audit-log coverage should be 100% of breakglass deploys reviewed within a fixed SLA; trend the usage count.
