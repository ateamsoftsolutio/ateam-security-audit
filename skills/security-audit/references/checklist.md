# Claude's review checklist

Scanners are good at patterns. This checklist is for what they miss. Start with the hotspots in the attack surface map, then cover the rest as far as the project's size allows. For each item, read the real code path. Don't report an issue the framework already prevents.

## 1. Access control (OWASP A01:2025)

The most common serious flaw, and scanners rarely find it.

- [ ] For every route that reads, changes or deletes a record by ID, is there a check that the record belongs to the caller or that the caller's role allows it? Look for IDs taken from the URL, body or query and passed straight to a database call (IDOR).
- [ ] Are admin and staff functions protected on the **server**, not just hidden in the UI?
- [ ] Is authorisation applied consistently: middleware on every router, not just some? Are any routes registered before the authentication middleware?
- [ ] Multi-tenant apps: is every query scoped by tenant or organisation ID?
- [ ] Can users change fields they shouldn't, such as `role`, `isAdmin`, `price` or `ownerId`, through mass assignment?
- [ ] Are file downloads and signed URLs restricted to the owner?
- [ ] CORS: is `Access-Control-Allow-Origin` reflected from the request or set to `*` while credentials are allowed?
- [ ] Is CSRF protection in place for cookie-based sessions, through a token or SameSite cookies?

## 2. Authentication and sessions (A07:2025)

- [ ] Are passwords hashed with bcrypt, scrypt or Argon2, never stored in plain text or with MD5 or SHA-1?
- [ ] Is there brute-force protection on login, OTP and password-reset endpoints: rate limits or lockout?
- [ ] Do login and reset messages reveal whether an account exists (user enumeration)?
- [ ] Is the session ID regenerated after login (session fixation)? Is it destroyed on logout?
- [ ] Cookies: `HttpOnly`, `Secure`, `SameSite`, a sensible expiry, not the framework's default name?
- [ ] JWT: is the signature verified, the algorithm fixed (no `none`, no algorithm confusion), `exp` checked, the secret strong and not hard-coded? Are tokens kept out of `localStorage` when they carry high privilege?
- [ ] Password reset: are tokens random, single-use and short-lived? Is the reset link built from a request header an attacker can set (header poisoning)?
- [ ] Is multi-factor authentication available for admin accounts where the data warrants it?
- [ ] OAuth/OIDC: are `state` and PKCE used and redirect URIs validated?

## 3. Injection (A05:2025)

Trace from source (request, file, header, queue message, model output) to sink.

- [ ] **SQL:** queries built with string concatenation or template literals, or raw queries in an ORM.
- [ ] **NoSQL:** user objects passed into query filters (`{ $gt: "" }`), `$where`, or `mapReduce` with user input.
- [ ] **OS commands:** user input reaching shell-command functions (`child_process`, `os.system`, `subprocess` with `shell=True`, backticks).
- [ ] **Code execution:** `eval`, `new Function`, `setTimeout` with a string argument, `vm`, Python `exec`, template engines rendering user-supplied templates (SSTI).
- [ ] **XSS:** auto-escaping turned off, `dangerouslySetInnerHTML`, `v-html`, `innerHTML`, `|safe`, user-controlled `href` or `src` (`javascript:`), Markdown rendered without sanitising.
- [ ] **Path traversal:** user input in file paths without normalising and checking it stays inside an allowed folder.
- [ ] **LDAP, XPath, header and log injection:** user input written to logs without neutralising new lines.
- [ ] **Regular expressions:** user input matched against catastrophic patterns such as `(a+)+` (ReDoS).

## 4. Server-side request forgery and outbound calls (A01:2025)

- [ ] Does the server make requests to a URL or path the user controls (webhooks, link previews, adding images or files by link, PDF rendering)?
- [ ] Are internal addresses blocked, including the cloud metadata address, loopback, private ranges and redirects to them?
- [ ] Are outbound calls to third parties made with timeouts, and is TLS verification left on?

## 5. Files and uploads

- [ ] Are file type and size validated on the server? Content type, not just the extension?
- [ ] Are uploads stored outside the web root, with generated names?
- [ ] Is archive extraction safe from zip-slip?
- [ ] Is XML parsed with external entities disabled (XXE)?
- [ ] Is unsafe deserialisation avoided: `pickle`, Java native serialisation, `yaml.load`, `node-serialize`?

## 6. Secrets and cryptography (A04:2025)

- [ ] Are there hard-coded secrets in code or config (session secrets, JWT keys, API keys, database URLs)? Example values in `.env.example` are fine only if they are clearly placeholders.
- [ ] Are secrets loaded from environment variables or a secrets manager? Is `.env` in `.gitignore`?
- [ ] Is sensitive data encrypted at rest where it should be, such as national IDs, health data and bank details?
- [ ] Are weak algorithms (MD5, SHA-1 for passwords, DES, ECB mode) or a non-cryptographic random generator (`Math.random`, `random`) used for tokens?
- [ ] Is TLS enforced (HTTPS redirect, HSTS)? Is certificate verification turned off anywhere (`rejectUnauthorized: false`, `verify=False`)?
- [ ] Are private keys or certificates committed to the repository?

## 7. Configuration (A02:2025)

- [ ] Is debug mode or a verbose error page on in production configuration?
- [ ] Are security headers set: CSP, HSTS, `X-Content-Type-Options`, frame protection? Is `X-Powered-By` removed?
- [ ] Are default credentials or sample admin accounts present in seed scripts that could run in production?
- [ ] Are API docs (Swagger), admin consoles, metrics and health endpoints exposed without authentication?
- [ ] Are cloud storage buckets, database ports or dashboards public? Check IaC and compose files.
- [ ] Containers: running as root, `latest` tags, secrets in image layers or build arguments, privileged mode?

## 8. Supply chain and integrity (A03:2025, A08:2025)

- [ ] Are there known-vulnerable dependencies? (From the scanners. Focus on direct dependencies and anything reachable at runtime.)
- [ ] Are lock files present and committed?
- [ ] Do any dependencies come from unusual registries or git URLs, or have names that look like typosquats?
- [ ] CI: are actions pinned to commit SHAs? Are `pull_request_target` workflows checking out untrusted code? Are secrets exposed to forks? Is `GITHUB_TOKEN` write permission broader than needed?
- [ ] Do pages include third-party CDN assets without Subresource Integrity?
- [ ] Do auto-update or plugin mechanisms verify signatures?

## 9. Logging, errors and exceptional conditions (A09:2025, A10:2025)

- [ ] Are logins, permission failures and admin actions logged with enough detail to investigate?
- [ ] Are passwords, tokens, full card numbers, health data and personal data kept out of logs and error messages?
- [ ] Are stack traces or internal errors returned to users?
- [ ] Do errors "fail open"? For example, an exception in the permission check that lets the request through, or a payment status treated as success when the provider times out.
- [ ] Are there resource limits: request size, pagination caps, timeouts, rate limits on expensive endpoints?

## 10. Business logic (A06:2025)

- [ ] Can steps be skipped? Paying, approving, verifying email, accepting terms.
- [ ] Can amounts, quantities or prices be negative, zero, huge or changed on the client?
- [ ] Race conditions: double spending, using a coupon twice, booking the same slot twice.
- [ ] Is the server trusting state sent by the client (hidden fields, prices in the request, client-side role checks)?
- [ ] Are webhooks from payment and other providers verified by signature?

## 11. Front end and mobile

- [ ] Are secrets in bundles or public environment variables (`NEXT_PUBLIC_`, `VITE_`, `REACT_APP_`, `EXPO_PUBLIC_`)?
- [ ] Do Next.js or Nuxt server actions and API routes repeat the authorisation check rather than relying on page-level checks?
- [ ] Are source maps published in production?
- [ ] Mobile: secrets in the app package, certificate pinning where needed, sensitive data in insecure local storage?

## 12. AI and LLM features (OWASP Top 10 for LLM Applications 2025)

Check these whenever the project calls a language model, uses embeddings or retrieval, or runs agents.

- [ ] **LLM01 Prompt injection:** can untrusted content (user messages, emails, web pages, documents, retrieved chunks, tool results) change the model's behaviour? Is untrusted content clearly separated from instructions? Are sensitive actions gated outside the model?
- [ ] **LLM02 Sensitive information disclosure:** can the model reveal other users' data, secrets or internal data? Is personal or health data sent to providers without a lawful basis or a data-processing agreement?
- [ ] **LLM03 Supply chain:** models or datasets from untrusted sources, or pickled model files opened without checks?
- [ ] **LLM04 Data and model poisoning:** can users write into the retrieval store or fine-tuning data that others will receive?
- [ ] **LLM05 Improper output handling:** is model output passed into HTML, SQL, shell commands, file paths, `eval` or redirects without validation?
- [ ] **LLM06 Excessive agency:** can the model call tools that send, delete, pay or change records without a human confirmation step, with credentials scoped more widely than needed?
- [ ] **LLM07 System prompt leakage:** are secrets, keys or access rules kept only in the system prompt?
- [ ] **LLM08 Vector and embedding weaknesses:** is retrieval filtered by the caller's permissions and tenant?
- [ ] **LLM09 Misinformation:** in health, legal and financial features, are outputs grounded, shown with sources, and reviewed where the stakes require it?
- [ ] **LLM10 Unbounded consumption:** are there limits on tokens, request rate, loop iterations and cost per user? Are AI provider keys kept server-side?

## 13. Sector and compliance flags

Raise these as **flags to check with the right person**, not as legal findings.

- **Health data:** HIPAA (US); health-data rules in the UAE (DHA, DoH) and Saudi Arabia; the Australian Privacy Act; encryption, audit logs, access control and data residency.
- **Payment cards:** PCI DSS. Card numbers should never touch the application if a provider's payment page or tokenisation is available.
- **Personal data:** GDPR or UK GDPR, the UAE and Saudi PDPLs, India's DPDP Act, the Australian Privacy Act, US state laws. Check consent, retention, deletion, cross-border transfer and data residency.
- **Children's data:** extra consent and minimisation duties in most regions.

Add the regions and sectors from the scope and the house rules.
