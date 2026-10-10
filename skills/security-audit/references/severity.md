# Rating severity

Rate every finding on two things, then adjust for context.

- **Exploitability:** how easy it is to use. Who can reach it (anyone on the internet, any logged-in user, admins only, someone with local access) and how much skill or luck it needs.
- **Impact:** what happens if it's used. Data exposed or changed, accounts taken over, money moved, the service stopped, or the server taken over.

## Levels

| Level | Meaning | Typical examples |
|---|---|---|
| **Critical** | Easy to exploit and severe impact. Fix immediately; a live secret also needs rotating today. | Live production secret committed or published (database password, cloud key, payment secret key); remote code execution; authentication bypass; any user reading or changing all users' sensitive data; SQL injection on an internet-facing endpoint |
| **High** | Serious impact with some precondition, or moderate impact that's easy to exploit | IDOR on personal data that needs a login; stored XSS; SSRF to internal services; JWT signature not verified; plain-text passwords; missing access control on admin functions; a dependency with a known exploited vulnerability that is reachable |
| **Medium** | Limited impact, or needs unusual conditions | Reflected XSS needing user interaction; missing CSRF protection on non-critical actions; no rate limit on login; user enumeration; weak session settings; verbose errors; open redirect; missing security headers on a sensitive app |
| **Low** | Minor, or defence in depth | Missing headers on low-risk pages; `X-Powered-By` disclosed; old but non-exploitable dependency versions; development-only issues |
| **Info** | Good practice, no direct risk | Hardening suggestions, unpinned versions in non-production tooling |

## Context modifiers

Apply these after the base rating. A rating can move by at most one level up or down for context, unless you explain why.

- **Raise one level** if the app is internet-facing and the flaw is reachable without logging in, or if it involves health, payment, children's or national-ID data.
- **Lower one level** if the code is only in tests, local tooling or development configuration that never ships, or if a strong compensating control is confirmed (for example, a WAF rule plus a network restriction).
- **Secrets:**
  - A committed secret is **Critical** if it looks live and gives production access.
  - It's **High** if its environment is unclear, or it was only in git history and has since been removed. It's still exposed to anyone who has the history.
  - It's **Low** if it's an obvious placeholder or a test credential for a local-only service.
  - Keys designed to be public, such as publishable keys, aren't findings.
- **Dependencies:** start from the highest advisory severity in the group. Lower one level for development-only dependencies, and one level if the vulnerable function clearly isn't used. Raise one level if the advisory is on CISA's Known Exploited Vulnerabilities list or has a public exploit and the code path is reachable. Dependency ratings may move by up to **two** levels in total, with the reason stated in the evidence.

## Confidence

Record confidence separately from severity:

- **Confirmed:** you traced the full path in the code, or a scanner found it and you checked it.
- **Likely:** strong evidence, but one link can't be seen.
- **Needs check:** depends on runtime or deployment details.

A Critical or High finding rated "Needs check" still counts towards the gate until someone checks it and marks it Verified fixed or False positive.

## Standards mapping

**OWASP Top 10:2025**

| ID | Category |
|---|---|
| A01:2025 | Broken Access Control (includes SSRF) |
| A02:2025 | Security Misconfiguration |
| A03:2025 | Software Supply Chain Failures |
| A04:2025 | Cryptographic Failures |
| A05:2025 | Injection |
| A06:2025 | Insecure Design |
| A07:2025 | Authentication Failures |
| A08:2025 | Software or Data Integrity Failures |
| A09:2025 | Security Logging and Alerting Failures |
| A10:2025 | Mishandling of Exceptional Conditions |

**Common CWEs**

| CWE | Weakness |
|---|---|
| CWE-22 | Path traversal |
| CWE-78 | OS command injection |
| CWE-79 | Cross-site scripting |
| CWE-89 | SQL injection |
| CWE-94 | Code injection |
| CWE-200 | Information exposure |
| CWE-287 | Improper authentication |
| CWE-352 | CSRF |
| CWE-384 | Session fixation |
| CWE-502 | Unsafe deserialisation |
| CWE-601 | Open redirect |
| CWE-639 | Authorisation bypass through a user-controlled key (IDOR) |
| CWE-798 | Hard-coded credentials |
| CWE-862 | Missing authorisation |
| CWE-918 | SSRF |
| CWE-943 | NoSQL injection |
| CWE-1333 | ReDoS |
| CWE-1104 | Unmaintained third-party components |

For AI findings, also give the LLM Top 10 (2025) ID, from LLM01 to LLM10.

**Consistent mappings for common cases:**

| Finding | OWASP category |
|---|---|
| Hard-coded or exposed credentials (CWE-798) | A07:2025 |
| Weak or missing encryption of stored data | A04:2025 |
| Vulnerable dependencies | A03:2025 |
| SSRF | A01:2025 |
| Missing security headers or debug settings | A02:2025 |
