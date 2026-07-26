---
name: security
description: Write code that resists attack — parameterizing rather than escaping, encoding for the right output context, checking authorization per object rather than per route, storing passwords and secrets correctly, and designing for least privilege and recovery. Use when handling untrusted input, building auth or sessions, touching crypto or secrets, exposing an API, reviewing code for vulnerabilities, or designing access control and admin paths.
---

# Security

## Purpose

Security bugs in application code cluster into a small number of shapes that recur across every language and framework. Most of them come from one of two mistakes.

**A value crossed from data into code**, because it got concatenated into something another parser would later interpret — SQL, a shell, HTML, a filesystem path. That's injection and XSS and traversal, and the fix is structural, not vigilance.

**Authority was assumed rather than checked.** The caller was authenticated, so the code acted; the route was permitted, so the object was returned. That's most access-control failure.

Above both sits a design question: **can this mistake even be expressed?** Secure-by-construction beats secure-by-review, because review doesn't scale and doesn't survive turnover.

## Rules that apply without loading anything

**1. Parameterize; don't escape.** Prepared statements compile the query structure before the value arrives. Escaping is database- and encoding-specific and routinely bypassed. Same for commands: pass an argument array, never a shell string.

**2. Encoding depends on the output context.** HTML entity encoding does not protect a value placed inside a `<script>` block. There is no single "escape" function — pick the one matching the sink, or use a safe sink.

**3. Authentication is not authorization.** Prove *who* the caller is, then separately check they may act on *this specific object*. Scope the query to the caller so a wrong ID returns not-found rather than someone else's data.

**4. Encrypt what you must read back; hash what you only verify.** Encrypting a password is reversible by definition, which defeats the point. Use a slow memory-hard password hash — Argon2id by preference — never a fast general-purpose one.

**5. Allowlist, don't denylist.** A denylist enumerates the attacks you already thought of. Anchor every validation regex at both ends.

**6. A rotated secret is remediated; a deleted one is not.** A secret committed to git persists in history, forks, and caches. Rotate it — history cleanup is tidying, not a fix.

**7. Fail closed on security-enforcing controls.** The reliability instinct is to serve anyway when a check is unavailable. For a control that enforces security, that lets an attacker force a bypass by taking its dependency offline. Decide which failure mode each control needs *before* it fails.

**8. Your recovery path is attack surface.** Revocation, rollback, breakglass, and backups are levers an attacker can pull. Hold them to the same rigor as what they protect.

## Triage

| What you're doing | Reference |
|---|---|
| Handling untrusted input — queries, HTML, files, URLs, XML, uploads | [injection-and-untrusted-input](references/injection-and-untrusted-input.md) |
| Login, sessions, passwords, authorization, CSRF, API surface | [identity-and-access](references/identity-and-access.md) |
| Encryption, keys, secrets, TLS, logging, dependencies | [crypto-and-secrets](references/crypto-and-secrets.md) |
| Access-control architecture, admin paths, deploy integrity, recovery design | [secure-design](references/secure-design.md) |

The first three are code-level: the vulnerable pattern and the correct one. The fourth is design-level: what to build so the vulnerable pattern becomes hard to write.

## Where the leverage is

Ordered by how much they reduce whole classes of bug rather than individual instances:

1. **Make the unsafe thing inexpressible.** A type whose only constructors enforce the invariant converts "prove no caller violates this" into "prove this one constructor is correct." The type system then does the review permanently, instead of a human doing it once per call site and forgetting.
2. **Centralize enforcement.** A check every request passes through by construction — a framework interceptor, a template system that escapes — means proving the property holds requires reading one implementation, not every handler.
3. **Default deny.** Every route, every object access, decides explicitly to permit. Falling through to allowed is how things get exposed nobody meant to expose.
4. **Then the per-site rules** in the references.

A useful diagnostic for any security property: **how many files would you have to read to prove it holds system-wide?** If the answer isn't close to one, that's the finding.

## What this skill will not do

Threat modeling for your specific system, or compliance. It covers the recurring code and design mistakes; it does not tell you who your adversary is, what they want, or which regulation applies. Those change the priorities, and they're yours to determine.
