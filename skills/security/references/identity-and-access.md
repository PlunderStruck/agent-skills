# Identity and Access

## Authentication

**Trigger.** Login and reset endpoints that respond differently — in message *or* timing — for "no such user" versus "wrong password." Password comparison with `==`. No rate limiting on attempts. Sensitive changes (email, password, payment details) permitted without re-authentication.

**Why it fails.** Differing responses leak which usernames are valid, which turns a broad credential-stuffing attack into a targeted one. Non-constant-time comparison leaks the correct value progressively through timing.

**The fix.** Return one identical message *and* comparable response time for every failure mode — "if that account exists, we've sent an email." Use library-provided constant-time comparison for any security-sensitive value. Rate-limit by account and by source. Require current credentials or MFA before changing password, email, or payment details.

## Session management

**Trigger.** Session IDs from a non-cryptographic RNG or derived from timestamps. Cookies missing `Secure`, `HttpOnly`, or `SameSite`. Session ID unchanged across the login boundary. Logout that only clears the client cookie.

**Why it fails.** Predictable identifiers can be guessed outright. A session ID accepted from the client and *not regenerated at login* enables session fixation: the attacker plants a known ID, waits for the victim to authenticate under it, and their copy is now privileged. Missing `HttpOnly` means any XSS bug anywhere exfiltrates the session.

**The fix.**

- Generate session IDs with a CSPRNG, at least 64 bits of entropy.
- **Regenerate the session ID on every authentication and privilege change.** This is the fixation defense and it's frequently missed.
- Set `Secure`, `HttpOnly`, and `SameSite=Lax` or `Strict`.
- Keep the cookie value an opaque reference — no session data, no PII.
- Enforce both idle and absolute timeouts, server-side.
- On logout, invalidate server-side. Clearing the client cookie is not invalidation.

## Password storage

**Never** plaintext, never reversible encryption, and never a general-purpose fast hash. MD5, SHA-1, and SHA-256 are engineered to be *fast*, which is exactly wrong here — it lets anyone with a stolen hash database try billions of guesses per second on commodity GPUs.

Use a deliberately slow, memory-hard password hash. Current guidance, in preference order:

| Algorithm | Minimum parameters | When |
|---|---|---|
| **Argon2id** | ≥19 MiB memory, 2 iterations, parallelism 1 | Default choice |
| **scrypt** | N=2^17 (128 MiB), r=8, p=1 | Where Argon2id is unavailable |
| **bcrypt** | cost ≥ 10 | Legacy stacks only |
| **PBKDF2-HMAC-SHA256** | 600,000 iterations | Only when FIPS-140 compliance forces it |

PBKDF2 is last because it isn't memory-hard, which makes it the weakest of the four against GPU attack.

Salting is per-user and handled automatically by all of these — don't hand-roll it. A shared application-wide pepper is optional defense in depth; if used, store it outside the database, in a secrets manager or HSM.

**Rehash on login when you strengthen parameters.** Keep the old verifier working, and replace it with a hash under the new parameters the next time that user successfully authenticates.

**Never encrypt a password as a substitute for hashing.** Encryption is reversible by definition, which defeats the entire point. The rule: encrypt what you need to read back, hash what you only need to verify.

## Authorization and IDOR

**Trigger.** An endpoint that checks the caller is *authenticated* but not that they may act on *this specific object*:

```
Project.find(params[:id])                  # any authenticated user, any project
current_user.projects.find(params[:id])    # scoped
```

Or authorization enforced only in the UI, or only in routing middleware.

**Why it fails.** Authentication establishes *who*. It says nothing about *what they may touch*. Without a per-object check, changing an identifier reaches someone else's data, because nothing in the request path validates the relationship between caller and object.

Randomizing IDs reduces guessing but is not a substitute — anyone holding one legitimate ID can try others.

**The fix.**

- **Default deny.** Every route decides explicitly, rather than falling through to permitted.
- **Enforce at the data access layer,** not only in middleware. Middleware answers "can this role hit this route." It cannot answer "does this user own this row."
- **Scope every fetch to the caller** at the query level, so a wrong ID returns not-found rather than another user's data.
- Treat horizontal escalation (a peer's data) and vertical escalation (an elevated role's functions) as two separate checks. Both are required.

**Verify.** For every data-returning or state-changing endpoint, write a test as a *different* user attempting to reach the first user's object by ID. Expect rejection, not data. Grep for `.find(` and `.findById(` unscoped by a current-user or tenant filter.

## Cross-site request forgery

**Trigger.** A state change reachable by `GET`, or a write handler that acts on nothing more than a valid session cookie.

**Why it fails.** Browsers attach cookies automatically to *any* request to your origin, including one triggered by a hostile page the victim happens to have open. The cookie proves identity; it does not prove intent.

**The fix.** The synchronizer token pattern: a unique unpredictable per-session token, embedded in forms and AJAX calls, validated server-side on **every** state-changing request.

`SameSite` is valuable defense in depth but **not sufficient alone**. `Lax` still permits the cookie on top-level GET navigations, so any state change reachable by GET bypasses it entirely — and it constrains cross-*site* requests only, not same-site subdomain abuse.

For stateless APIs, an HMAC-bound double-submit cookie (not a naive raw match) or a required custom header works, because a simple cross-site form submission cannot set custom headers.

## Mass assignment

**Trigger.** Binding a whole request body onto a model:

```
User.update(req.body)
User.create({ ...req.body })
```

**Why it fails.** Auto-binding maps every key to a matching property by name. If the model has `isAdmin` or `role` for internal use, the attacker adds `"isAdmin": true` to the body and the binder does the rest. No code anywhere asked for that field to be client-settable.

**The fix.** Never bind external input directly onto a persistence entity. Bind onto a purpose-built DTO containing only the fields the endpoint intends to accept, or use framework allowlisting that names bindable fields explicitly.

## API-specific concerns

**REST.** Allowlist accepted HTTP methods and reject the rest. Enforce request size and rate limits per authenticated identity. Validate `Content-Type` against an allowlist rather than echoing `Accept`. Use `403` rather than `404` for authorization failures so behavior doesn't leak resource existence — and never return stack traces.

**GraphQL.** Two failure modes are specific to the shape of the technology:

- **Unbounded query cost.** The schema graph makes deep, high-fan-out queries reachable without any single endpoint having been exposed deliberately. Enforce a maximum depth and a complexity budget *before* execution, plus a server-side timeout. Disable introspection and any GraphiQL UI in production.
- **Authorization checked only at the query root.** A top-level check misses nested objects reached through relationships — reading another user's private data via a nested `author` field on a post the caller legitimately owns. Re-run the object-level check inside **every resolver that returns a distinct object.**
