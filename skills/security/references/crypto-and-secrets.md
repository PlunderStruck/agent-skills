# Cryptography, Secrets, and Transport

## The rule that prevents most crypto mistakes

**Encrypt what you need to read back. Hash what you only need to verify.**

Most crypto errors in application code are a category error before they're an implementation error — encrypting a password, or hashing something that later needs recovering. Settle which one you need before choosing an algorithm.

## Symmetric encryption

**Trigger.** A homemade algorithm. ECB mode. DES or RC4. A hardcoded key or IV in source. An IV or nonce reused across encryptions under the same key.

**Why it fails.**

- **ECB** encrypts identical plaintext blocks to identical ciphertext blocks, so structure in the plaintext survives visibly into the ciphertext.
- **IV reuse** under stream and counter modes lets an attacker XOR two ciphertexts together and recover relationships between the plaintexts.
- **Unauthenticated modes** (plain CBC with no MAC) decrypt tampered ciphertext into garbage or, worse, attacker-influenced bytes, with no error.

**The fix.** Use authenticated encryption — **AES-GCM** or **ChaCha20-Poly1305**. AEAD gives confidentiality *and* tamper detection in one primitive, so modified ciphertext fails to decrypt rather than silently succeeding. Minimum 128-bit keys, 256 preferred. For asymmetric, RSA ≥ 2048 bits or a modern curve.

**Let the library generate IVs and nonces** from its CSPRNG. Don't construct them, and never reuse one under the same key.

**Verify.** Grep for `DES`, `RC4`, `ECB`, and `IvParameterSpec` constructed from a literal byte array. Confirm the mode named in code is GCM, CCM, or Poly1305.

## Constant-time comparison

**Trigger.** Comparing a MAC, token, signature, or any secret with `==`, `.equals()`, or `strcmp`.

**Why it fails.** Ordinary comparison short-circuits at the first differing byte, so how long it takes reveals how many leading bytes were correct. That's enough to recover the value byte by byte given enough attempts.

**The fix.** Use the platform's constant-time comparison — `hmac.compare_digest`, `crypto.timingSafeEqual`, `MessageDigest.isEqual`, `subtle.ConstantTimeCompare`.

## Key management

**Trigger.** The encryption key stored in the same table or file as the ciphertext it protects.

**Why it fails.** Anyone who can read the data store gets both halves. The encryption adds no real barrier — it protects against someone who obtained the ciphertext *and nothing else*, which is a narrow threat model.

**The fix.** Keys live in a dedicated KMS, HSM, or secrets manager, separate from the data they protect, with rotation and access auditing. Wrap exported keys with a key-encryption key.

## Secrets

**Trigger.** API keys, database passwords, or private keys in source control, in a Dockerfile `ENV`, or baked into a build artifact.

**Why it fails.** A secret committed to git **persists in history** after the offending line is deleted or the commit reverted. The blob remains retrievable by anyone with clone access, indefinitely — including from forks and mirrors you don't control.

**The fix.** Inject at runtime from a secrets manager or an orchestrator-provided environment, never baked into the image. Prefer short-lived dynamically issued credentials where the backing service supports them.

**If a secret has already been committed, rotation is mandatory.** Deleting the line doesn't help. Rewriting history doesn't reliably help either — assume it was already cloned, cached, or scraped. Rotate the credential; treat history cleanup as tidying, not remediation.

**Verify.** Run a secrets scanner in a pre-commit hook *and* against full history — `detect-secrets`, `truffleHog`, `git-secrets`.

## Secrets in logs

Logs travel further than the application does — aggregators, tickets, support screenshots, third-party monitoring. A secret written to a log leaks through a much wider surface than the original bug would have.

Maintain an explicit never-log list: credentials, tokens, keys, card data, PII. Redact before the log call, not after.

Separately, **strip or encode CR/LF from user-controlled values before they enter a log line.** Otherwise an attacker injects forged entries that look legitimate, undermining the audit trail you're keeping the logs for. Structured (JSON) logging avoids this by construction, since fields can't bleed into one another.

Do log security-relevant events themselves — authentication successes and failures, authorization failures, validation failures. That's the trail you actually want.

## Transport security

**Trigger.** TLS 1.0 or 1.1 still enabled. Certificate validation disabled for local convenience and never re-enabled: `verify=False`, `rejectUnauthorized: false`, `InsecureSkipVerify`, a trust manager that returns true unconditionally.

**Why it fails.** Deprecated protocol versions remaining enabled permit a downgrade attack. And disabling certificate validation doesn't weaken TLS — it *removes the authentication guarantee entirely*, reducing it to unauthenticated encryption that any machine-in-the-middle defeats. The channel is still encrypted; you just don't know who you're talking to.

**The fix.** Require TLS 1.2 minimum, prefer 1.3, and disable older versions explicitly rather than relying on defaults. Use AEAD cipher suites; disable null, anonymous, export-grade, and non-forward-secret suites. Add HSTS so browsers refuse to downgrade.

**Never ship a skip-verification flag that was added for local testing.** Gate it behind an environment check that cannot be true in production, or remove it and use a real certificate locally.

**Verify.** Grep for `verify=False`, `rejectUnauthorized`, `InsecureSkipVerify`, and custom hostname verifiers. Run an external scanner against the deployed endpoint rather than trusting configuration alone.

## Error handling

**Trigger.** A raw stack trace, SQL error, or framework version banner returned in an HTTP response.

**Why it fails.** It's free reconnaissance — framework, version, and often schema details handed to whoever asked.

**The fix.** Centralize exception handling so every unhandled error returns one generic message to the caller while the full exception, with context, goes only to server-side logs.

## Dependencies

**Trigger.** Unpinned versions, no lockfile, no vulnerability scanning.

**The fix.** Commit lockfiles and install from them in CI (`npm ci`, not `npm install`). Run an SCA scanner in the pipeline and fail the build on high or critical findings. Enable automated update tooling so patches land quickly rather than accumulating. Generate an SBOM for anything shipped externally.
