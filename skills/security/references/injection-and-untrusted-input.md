# Injection and Untrusted Input

Every vulnerability here has one root cause: **a value crossed from data into code**, because it got concatenated into something another parser would later interpret.

## Injection: SQL, OS command, LDAP

**Trigger.** Building an executable string by concatenation:

```
"SELECT * FROM users WHERE name = '" + name + "'"
subprocess.call("convert " + filename, shell=True)
"(&(uid=" + user + ")(objectClass=person))"
```

**Why it fails.** Once code and data are merged into one string, the downstream parser — SQL, shell, LDAP filter — has no way to tell which characters you intended as syntax. Quotes, semicolons, pipes, asterisks, and parentheses become structure.

**The fix is parameterization, not escaping.** Prepared statements compile the query structure *before* the value arrives, so the value can never be reparsed as SQL. Escaping is a distant fallback: the rules are database- and encoding-specific, and encoding tricks route around them routinely.

Two cases that bite:

- **Identifiers can't be parameterized.** Table names, column names, and sort direction are not values. Map user input through a hardcoded switch or allowlist to a known-safe identifier. Never append the raw string.
- **Stored procedures aren't automatically safe.** A procedure that builds dynamic SQL internally from its parameters has the same hole, just relocated.

**For commands, skip the shell entirely.** Use the array form so each element is one atomic argument:

```
subprocess.run(["convert", filename], shell=False)     # safe
ProcessBuilder("convert", filename)                     # safe
```

Note that blocking shell metacharacters is not sufficient on its own — an attacker can still inject additional *arguments* to the intended program, which is its own class of attack. Allowlist both the executable and the argument shape.

**Verify.** Grep for concatenation or interpolation feeding `execute`, `query`, `system`, `exec`. Grep for `shell=True`. Submit `' or '1'='1`, `; id;`, and `*)(uid=*` and confirm they're inert.

## Cross-site scripting

**Trigger.** Untrusted data written into a response without encoding *matched to the output context*, or flowing into a dangerous DOM sink: `innerHTML`, `document.write`, `eval`, `setTimeout` with a string, `dangerouslySetInnerHTML`, `v-html`, Angular's `bypassSecurityTrustHtml`.

**Why it fails — and why the common fix is wrong.** The browser parses HTML, JavaScript, CSS, and URLs with *different grammars*. Encoding correct for one context is a no-op in another. HTML-entity-encoding a value placed inside a `<script>` block does not stop execution, because script bodies aren't HTML-decoded the same way — the payload still closes the string and injects a statement.

So encoding is not one operation. It's a family, and picking the wrong member silently leaves the hole open.

Some contexts are unsafe **at any encoding level**: directly inside `<script>` or `<style>`, inside an HTML comment, as a tag or attribute *name*, inside an event-handler attribute, or inside CSS `url()`. Don't put untrusted data there at all.

**The fix, in preference order:**

1. **Use a safe sink.** `textContent` instead of `innerHTML`. `createElement` and `createTextNode`. `setAttribute` with a hardcoded attribute name.
2. **Let the framework escape.** React JSX, Vue `{{ }}`, and Angular interpolation are default-safe. The moment code reaches for an escape hatch, that boundary is manual again and needs the same scrutiny as raw concatenation.
3. **Match the encoder to the sink** when you must: HTML entity encoding for element content, attribute encoding for attribute values, `\uXXXX` JavaScript encoding only inside a quoted JS string literal, CSS hex encoding for property values, percent-encoding for URL parameters. When contexts stack — a URL inside an attribute — apply both, innermost first.
4. **Sanitize with an allowlist library** (DOMPurify) when raw HTML genuinely must render — at the point of insertion, not at request intake, because only the insertion point knows the render context.

**Verify.** Grep the dangerous sinks and trace each source. Test per context: `<img src=x onerror=alert(1)>` in body, `" onmouseover="alert(1)` in an attribute, `');alert(1);//` in script.

## Deserialization

**Trigger.** Untrusted bytes into a native object deserializer: `ObjectInputStream.readObject`, `pickle.loads`, PHP `unserialize`, `Marshal.load`, .NET `BinaryFormatter`, `yaml.load` (rather than `safe_load`), or JSON.Net with `TypeNameHandling` set to anything but `None`.

**Why it fails.** Native deserialization *reconstructs objects*, invoking constructors and magic methods as it rebuilds the graph. An attacker doesn't need a bug in your code — they chain classes already on your classpath whose side effects, triggered in sequence, execute arbitrary code. The vulnerability is structural to the format, not a missing check.

**The fix.** Don't deserialize untrusted data with a native format. Use a data-only format validated against a strict schema — JSON and XML have no code-execution hooks. If legacy interop forces it: allowlist permitted classes before reconstruction, and require a signature verified *before* deserialization begins. Verifying after is not verifying.

## Server-side request forgery

**Trigger.** The server fetches a user-supplied or user-influenced URL — webhooks, "import from URL," link previews, PDF and screenshot generators.

**Why it fails.** Your server has network reach the attacker doesn't: internal services, admin panels, localhost, and cloud metadata endpoints that hand out credentials with no further authentication. The application becomes a proxy into its own private network.

**The fix.**

- **Allowlist destinations.** Denylists lose to alternate IP encodings, DNS rebinding, and redirects.
- **Resolve the hostname and check the resolved IP** against private, loopback, and link-local ranges — not the string the user typed.
- **Disable automatic redirect following,** or re-validate the destination at every hop.
- **Validate the response** too — content type and size.
- Network egress restrictions are worthwhile defense in depth, but not the only control.

**Verify.** Test with a metadata address, `localhost` with various ports, decimal and octal IP encodings, and a redirect chain terminating internally.

## XML external entities

**Trigger.** Parsing attacker-supplied XML with default parser settings.

**Why it fails.** Most XML parser defaults predate the current threat model and resolve `SYSTEM` entities — fetching local files, or internal URLs (turning XXE into SSRF), or expanding recursively until memory is exhausted.

**The fix.** Disable DTD processing and external entity resolution explicitly. This is the recommended default even when entities aren't a business requirement — `disallow-doctype-decl` in Java, `defusedxml` in Python, `DtdProcessing.Prohibit` with a null resolver in .NET.

## Input validation

**Trigger.** Rejecting input that matches a list of known-bad patterns.

**Why it fails.** A denylist enumerates the attacks you already thought of. Encoding variants and unanticipated syntaxes pass untouched, while the same filter rejects legitimate input it wasn't designed for — the apostrophe in `O'Brian`.

**The fix.** Validate positively. Define the acceptable type, length or range, and either a character-class regex **anchored at both ends** (`^[A-Za-z0-9_-]{1,32}$`) or membership in a fixed enumeration. Unanchored patterns are a frequent silent failure.

Layer syntactic validation (is this shaped like a date) with semantic validation (is this date in a sane range, does start precede end). For free-form text, normalize Unicode to a canonical form first, then allowlist character *categories*.

Always server-side. Client-side validation is UX, bypassed by replaying the request directly.

## File upload and path traversal

**Trigger.** Trusting the client's filename, extension, or `Content-Type`; storing under the supplied name inside the webroot; building a path by concatenating user input onto a base directory.

**Why it fails.** Filename and content type are attacker-controlled strings, not facts. A `.php` stored under its original name in a served directory can be requested and executed. And string concatenation doesn't understand filesystem semantics — only path resolution does, which is why substring checks for `..` lose to encoding and symlinks.

**The fix.**

- Allowlist extensions against business need, then **independently verify content by magic bytes**. Extension and declared MIME type are claims.
- **Generate the stored filename yourself** (a UUID), discarding the client's entirely.
- Store outside the webroot, or somewhere execution is disabled.
- Enforce a size limit.
- For paths: **canonicalize the resolved path and assert it is still a descendant** of the intended base before touching it.

**Verify.** Test double extensions, null-byte suffixes, encoded traversal, and absolute paths.
