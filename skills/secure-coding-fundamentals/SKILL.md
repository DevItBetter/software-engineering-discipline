---
name: secure-coding-fundamentals
description: "Identify and prevent security vulnerabilities in application code. Use this skill whenever the task involves a security review, threat modeling, authentication / authorization design, handling user input, working with secrets, designing or evaluating cryptography choices, reviewing trust boundaries, evaluating supply-chain risk (dependencies, packages), reviewing changes to LLM/AI systems for prompt-injection and excessive-agency risks, or reviewing any code that touches money, PII, credentials, or production access. Built on OWASP Top 10:2025 and OWASP Top 10 for LLM Applications (2025), with concrete patterns and what to flag in review."

---

# Secure Coding Fundamentals

Security failures usually look the same in retrospect: an input that wasn't validated where it crossed a trust boundary, an authorization check that wasn't there, a secret that was logged, a dependency that was trusted because nobody checked. The defenses are well-known. The discipline is applying them every time, not just when somebody is paying attention.

This skill is about building the reflexes for that discipline. It does not make anyone a security professional — for high-stakes systems (payments, health, identity), a real security review by a real security team is non-negotiable.

## Trust boundaries — the most important concept

A **trust boundary** is a point in your system where data crosses from one trust level to another. Examples:

- Network boundary: bytes coming from the internet are untrusted.
- Process boundary: bytes from another process you don't control are untrusted.
- User boundary: the authenticated user is *not* fully trusted; they may be malicious.
- Tenant boundary: data from one tenant must not leak to another.
- Storage boundary: data read from your own database is *less* trusted than data you just generated, because something else may have written it.
- LLM input/output boundary: text that came from or will go to an LLM is untrusted in special ways (see LLM section).

At every trust boundary:

- **Validate** the input. Specifically — check the shape, type, range, and constraints.
- **Authorize** the action. The right user is allowed to do this specific thing on this specific resource.
- **Log** the cross-boundary event with enough context for forensics.

Most security bugs are a missing check at a boundary. The defense is not "be careful." It is structural: the boundary must be at a known place in the code, every interaction crossing it must go through a documented checkpoint, and the checkpoint must be hard to bypass (enforced by type, by middleware, or by review discipline).

## OWASP Top 10:2025 — the application security baseline

Every reviewer should know these by heart. They are the categories of vulnerabilities that account for most real-world incidents. The 2025 release reordered, renamed, and consolidated several categories from the 2021 list — notably it absorbed Server-Side Request Forgery into Broken Access Control, replaced "Vulnerable and Outdated Components" with the broader **Software Supply Chain Failures**, and added **Mishandling of Exceptional Conditions**. Verify the current edition at `https://owasp.org/Top10/`.

### A01:2025 Broken Access Control

The user is authenticated, but the system fails to check whether they're allowed to do *this specific operation on this specific resource*. The 2025 edition consolidates Server-Side Request Forgery (SSRF) into this category — see the SSRF subsection below.

Common shapes:
- An endpoint that takes a resource ID and returns the resource without checking whether the calling user owns it. (`GET /orders/123` returns order 123 to anyone authenticated.)
- A list endpoint that doesn't filter by tenant. (Lists all orders, not just the caller's.)
- A field-level authorization gap. (`PATCH /users/{id}` lets users update their own record, but the body can include `role: "admin"`.)
- Insecure direct object references (IDOR). Sequential numeric IDs that can be enumerated.
- Hidden admin endpoints exposed via direct URL.

Defense:
- Every endpoint with a resource argument: check ownership/access *explicitly* at the start of the handler. **This is the load-bearing defense.** UUIDs and the rest below are defense-in-depth, not substitutes.
- Filter by tenant/owner at the database query level (e.g., row-level security). Don't filter in application code only — easy to miss; one missed query is one breach.
- Use UUIDs (not sequential integers) for IDs that appear in URLs. UUIDs make enumeration impractical, so a missed authz check is less catastrophic — but a missed authz check on UUIDs is still a breach for anyone who has a valid id.
- Define a permissions model centrally; enforce it via middleware or decorators that are hard to bypass.
- Test: write tests that try to access another tenant's data with a valid auth token. They should 403/404.

#### Server-Side Request Forgery (SSRF) — consolidated into A01 in the 2025 edition

The server fetches a URL controlled by the attacker, who uses it to reach internal services they shouldn't have access to. SSRF was a standalone category (A10) in the 2021 list and has been folded into Broken Access Control because the underlying flaw is the same: the server acts on behalf of the attacker without checking whether the attacker is authorized to reach the destination.

Common shapes:
- An "import from URL" feature that fetches whatever URL the user provides.
- A "convert to PDF" service that fetches the URL passed in.
- An OAuth callback URL not strictly validated.

Defense:
- Parse URLs with a vetted parser; allow only required schemes and ports.
- Allow-list the destinations the server will fetch from, then resolve DNS server-side and verify every resolved IP before connecting.
- Block loopback, private, link-local, multicast, IPv6 local ranges, and cloud metadata services (`169.254.169.254` is the famous AWS metadata endpoint).
- Re-check policy after redirects and before the final connect; do not rely on string-matching hostnames.
- Run outbound fetches through an egress proxy or firewall that enforces the same policy.

### A02:2025 Security Misconfiguration

The defaults that should have been changed weren't, or the production config has development-mode settings.

Common shapes:
- Default admin credentials still in place.
- Debug mode enabled in production (stack traces returned to users).
- CORS configured as `Access-Control-Allow-Origin: *` for sensitive APIs.
- Verbose error messages exposing internals.
- Unnecessary services running (admin panels, debug endpoints).
- Cloud storage buckets accidentally public.
- Permissive IAM policies.

Defense:
- Configuration as code, version controlled, reviewed.
- Different configs per environment, with prod being the most restrictive.
- Automated config scanning (cloud security posture tools, kube-bench, etc.).
- Sensible defaults at the framework level.

### A03:2025 Software Supply Chain Failures

New top-level category in the 2025 edition, broader than the 2021 "Vulnerable and Outdated Components." Covers the full chain: dependencies you import, build tools you run, signing infrastructure, package registries, CI/CD pipelines, and the integrity of the artifacts you ship.

Common shapes:
- Known-vulnerable dependencies left unpatched.
- Typosquatted or namespace-confused packages (`requests` vs `requesst`, internal name shadowed by a public registry).
- A build pipeline that pulls dependencies without verifying lockfile hashes or signatures.
- Updates pulled from CDNs or registries without integrity checks (no Subresource Integrity, no signature verification).
- Compromised maintainer accounts; malicious updates pushed to widely-used packages.
- A build server that can be tampered with to inject artifacts post-compile.

Defense:
- Dependency scanning in CI (Dependabot, Snyk, Trivy, OSV-Scanner). Patch SLAs by severity.
- Lock files committed; verify hashes on install. Reproducible builds.
- Sign artifacts (Sigstore, in-toto, SLSA provenance). Verify signatures before deploy.
- Subresource Integrity (SRI) for CDN-loaded scripts.
- Pin and audit transitive dependencies for critical paths.
- Watch upstream advisories for direct dependencies you carry.
- Treat the build pipeline as production infrastructure with the same access controls.

### A04:2025 Cryptographic Failures

Sensitive data exposed because crypto is missing, weak, or misused.

Common shapes:
- Passwords stored in plaintext or with weak hashing (MD5, SHA-1 unsalted).
- Tokens or session IDs that are guessable or reused.
- HTTP instead of HTTPS for sensitive operations.
- Custom cryptography (homegrown algorithms, custom protocols).
- Hardcoded keys in source.
- Sensitive data in URLs (`?token=...` — ends up in logs).

Defense:
- Hash passwords with argon2id, bcrypt, or scrypt. Never SHA-1/MD5/SHA-256 directly.
- Use the language's vetted crypto library (libsodium, golang.org/x/crypto, etc.). Never implement crypto from scratch.
- TLS for every external connection. HSTS for web.
- Sensitive data goes in headers or bodies, not URLs.
- Keys come from a secret manager. Rotate periodically. Never in source, never in logs.

### A05:2025 Injection (SQL, OS command, LDAP, NoSQL, expression injection)

User input concatenated into a query/command/expression that's then executed.

Common shapes:
- `f"SELECT * FROM users WHERE name = '{name}'"` — SQL injection.
- `subprocess.run(f"convert {filename} out.png", shell=True)` — command injection.
- `eval(user_input)`, `Function(user_input)` — code injection.
- HTML rendering without escaping — XSS (cross-site scripting).
- Template engines that auto-escape disabled.

Defense:
- **Parameterized queries.** Always. Every database call. The query string contains placeholders; the values come separately.
- **Subprocess with `args` list** (not `shell=True`). The shell is the injection vector.
- **Never `eval` user input.** No exceptions.
- **Escape output by context.** HTML escaping for HTML; URL encoding for URLs; SQL parameterization for SQL. The right escaping depends on where the value lands.
- **Use frameworks that auto-escape.** Modern templating engines escape by default; never disable that.
- **Content Security Policy** for web apps to mitigate XSS impact.

### A06:2025 Insecure Design

The vulnerability is in the design, not the code. The code does what the design says; the design is wrong.

Examples:
- A "forgot password" flow that lets attackers enumerate accounts.
- A registration flow with no rate limiting that allows credential stuffing.
- A trust model where any authenticated user can perform any privileged action.
- A workflow that depends on hidden client-side validation only.

Defense:
- Threat model new features. Ask: who can do what? What if they're hostile? What if they have stolen credentials? What if they can intercept traffic?
- Build security in at design time, not bolted on later.
- Use the principle of least privilege at every layer.

### A07:2025 Authentication Failures

Authentication is wrong, weak, or missing.

Common shapes:
- Predictable session IDs.
- Password complexity rules without rate limiting (the attacker has all the time in the world).
- "Remember me" cookies that don't expire or can't be revoked.
- Account enumeration (different responses for "user not found" vs "wrong password" — attacker confirms which usernames exist).
- Missing MFA on sensitive operations.
- Session fixation.

Defense:
- Use a vetted authentication library/service (Auth0, AWS Cognito, Authelia, Ory, etc.). Don't roll your own.
- Generic error messages for login failures (don't reveal whether the user exists).
- Rate limiting on authentication endpoints.
- MFA for any privileged action.
- Session rotation on privilege change.
- Short token lifetimes; revocation mechanism.

### A08:2025 Software or Data Integrity Failures

Code or data altered in transit, or trust placed in a deserializer or webhook without verification. Distinct from A03 Software Supply Chain Failures: A03 covers the chain of producing and distributing the artifact; A08 covers integrity at the consumption boundary.

Common shapes:
- Deserializing untrusted data (`pickle.loads()`, unsafe YAML, Java native serialization, .NET BinaryFormatter, PHP unserialize).
- Webhooks accepted without signature verification.
- Updates fetched without integrity checks at runtime.
- Auto-updating clients that don't verify signatures.

Defense:
- Never deserialize untrusted data with formats that allow code execution. Use safe deserializers (JSON, MessagePack with explicit schemas).
- Verify webhook signatures (HMAC) before processing — every time, on every webhook.
- Sign and verify auto-updates; pin certificates where appropriate.

### A09:2025 Security Logging and Alerting Failures

A breach happened; nobody noticed; nobody can investigate.

Defense:
- Log all authentication events (success and failure).
- Log all authorization decisions for sensitive operations.
- Log all admin actions.
- Centralized log aggregation; alerts on suspicious patterns.
- Logs immutable and retained for an appropriate period.
- Don't log secrets, full PII, or session tokens.

### A10:2025 Mishandling of Exceptional Conditions

New category in the 2025 edition. Errors, exceptions, and edge cases handled incorrectly become security failures: a swallowed exception that bypasses an authorization check, a fail-open default when a downstream service is unreachable, an error path that leaks stack traces or returns the wrong privilege level.

Common shapes:
- A `try/except` that catches a broad exception and proceeds as if successful, silently bypassing a check that raised.
- An "if the auth service is down, allow the request" fail-open policy.
- A timeout on a permission check that defaults to allow rather than deny.
- An exception handler that returns the raw exception message to the user, leaking SQL, file paths, internal IDs, or stack traces.
- A retry loop that retries an operation that already partially succeeded, without idempotency, producing inconsistent state.
- Different error responses for "user not found" vs "wrong password" — see Authentication Failures.

Defense:
- **Fail closed by default.** When in doubt, deny. Authorization timeouts, unreachable auth services, ambiguous error states should all default to deny.
- **Catch narrowly.** Catch the specific exceptions you can handle; let the rest propagate. Broad `except:` clauses are how authorization checks get silently bypassed.
- **Sanitize error messages crossing trust boundaries.** Internal exception details belong in logs, not in user-facing responses.
- **Distinguish recoverable from unrecoverable** in code. Recoverable errors get a retry path with idempotency; unrecoverable errors fail loudly.
- See `error-handling-and-resilience` for the broader discipline; the security-relevant subset is what concerns this category.

## OWASP Top 10 for LLM Applications (2025)

If the system involves an LLM (chatbot, agent, RAG, tool-using assistant), these are the categories. The 2025 list:

1. **Prompt Injection** — attacker manipulates the LLM via input. Direct (in user prompt) or indirect (in retrieved content). Mitigations: separation of trust contexts, output filtering, never giving the LLM unrestricted ability to act on its own outputs.
2. **Sensitive Information Disclosure** — LLM leaks PII, secrets, or training data. Mitigations: data minimization, output filtering, careful RAG context.
3. **Supply Chain** — compromised models, datasets, or RAG sources. Mitigations: vetted sources, integrity checks, model card review.
4. **Data and Model Poisoning** — training/fine-tuning data is compromised. Mitigations: data provenance, validation.
5. **Improper Output Handling** — LLM output used unsafely (executed as code, rendered as HTML, used to construct queries). Mitigations: treat LLM output as untrusted user input.
6. **Excessive Agency** — agent has more functionality, permissions, or autonomy than its task requires. Mitigations: least-privilege tools, human-in-the-loop for high-impact actions, action allow-lists.
7. **System Prompt Leakage** — internal instructions exposed. Mitigations: assume system prompts can leak; don't put secrets there.
8. **Vector and Embedding Weaknesses** — RAG/vector DB attacks (injection via retrieved documents, embedding inversion).
9. **Misinformation** — LLM produces confident wrong answers. Mitigations: grounding, citations, human review for stakes.
10. **Unbounded Consumption** — runaway costs from prompt loops, recursive tool calls. Mitigations: rate limits, cost ceilings, loop detection.

For agentic systems specifically: **excessive agency is often the highest-blast-radius category**. An agent with tools to send email, call APIs, write files, and execute code requires extensive guardrails. The default permission model should be "read-only and ask before acting"; "act first" requires explicit justification and tight scoping.

What "ask before acting" looks like operationally:
- Tools partition into **safe** (read-only, idempotent, low-blast-radius) and **gated** (writes, sends, transfers, irreversible).
- Safe tools the agent calls freely; gated tools require an explicit human confirmation in the UI (or a hard-coded allow-list of pre-approved invocations).
- The confirmation prompt shows the *exact* action: not "send the email" but "send THIS email to THESE recipients with THIS subject." The user approves a specific call, not a category.
- Approvals are not granted by the agent itself reading the user's prior message — they require a fresh, explicit user action in a separate channel from the model's input. Otherwise the model can grant itself approval via prompt injection.
- A timeout / re-confirm requirement for long-running agent sessions, so a hijacked session doesn't keep approving things forever.

## Threat modeling, lite

For any nontrivial new feature, run a quick threat model:

1. **What are the assets?** What data, what functionality, what access does this feature touch?
2. **Who might want to attack it?** External users, authenticated users, malicious insiders, compromised accounts.
3. **What can they do?** Brainstorm misuse, not just use. STRIDE is a useful checklist: **S**poofing, **T**ampering, **R**epudiation, **I**nformation disclosure, **D**enial of service, **E**levation of privilege.
4. **What are the existing mitigations?** Authentication, authorization, validation, monitoring.
5. **What gaps remain?** What residual risk are we accepting? What follow-up is needed?

Document the result in the design doc. A feature that ships with no threat model is a feature you'll discover the threat model for the hard way.

## Secrets management

- **Never in source.** Repos are forever; even private repos leak.
- **Never in logs.** Sanitize before logging.
- **Never in error messages** returned to users.
- **Never in URLs** (they end up in logs).
- **Avoid long-lived secrets in environment variables** when crash dumps, debug endpoints, process listings, or support bundles may expose them. Environment injection can be acceptable for platform-managed short-lived secrets, but prefer a secret manager that fetches at runtime and supports audit/rotation.
- **Use a secret manager** (Vault, AWS Secrets Manager, GCP Secret Manager, 1Password Secrets Automation, etc.) for production secrets.
- **Rotate periodically.** Especially after personnel changes.
- **Audit access.** Who can read the secret, and when did they last?

## What to flag in review

- An endpoint with no visible authorization check.
- A query built by string concatenation.
- `subprocess` with `shell=True` and user input.
- `pickle.loads`, `eval`, `Function`, unsafe `yaml.load` of untrusted data.
- A new direct dependency on an unvetted package.
- Secrets in source / logs / error messages.
- A redirect that takes a user-controlled URL with no allowlist.
- A file path or filename derived from user input.
- A new CORS rule that's overly permissive.
- A new admin/debug endpoint without explicit auth gating.
- An LLM agent given a tool with broad scope (filesystem write, shell execution, money movement) without explicit justification and human-in-the-loop.
- A webhook handler with no signature verification.
- An authentication or session change without rate limiting / monitoring.
- A new error message that leaks internal information (paths, internal IDs, stack traces).

## When to escalate

A code review by an AI or a peer reviewer is not a substitute for security review on:

- Anything touching authentication, authorization, or session management.
- Anything touching cryptography (key generation, signing, encryption).
- Anything touching payment, KYC, or regulatory data.
- Anything that reduces an existing security control.
- Anything in the LLM/agent path that touches privileged tools.

For these, the right action is "request a security-team review" — not approve based on your own analysis.

## Reference library

- `references/llm-security-deep-dive.md` — prompt injection, excessive agency, agent permission models in depth.
