# Security Audit Loop by ATeam

**Audit. Fix. Verify. Repeat until the gate passes.**

Security Audit Loop is a Claude plugin that runs a full security audit of a software project and keeps the team on it until every blocking issue is fixed and confirmed. It works in **Claude Code** and in **Claude Cowork** with a project folder.

Most security checks give developers one long list and stop. This plugin keeps going. Every finding goes into a tracked register with a permanent ID and owner. Developers fix and explain each issue. A rescan confirms the fix before it's closed, and a gate decides when the project is ready for sign-off.

It was built by [ATeam Soft Solutions](https://www.ateamsoftsolutions.com), is used by ATeam's delivery teams, and is shared as a contribution to the community.

## The three skills

| Skill | Say | What it does |
|---|---|---|
| `security-audit` | "Run Security Audit" | Scopes the project, maps the attack surface, runs open-source scanners, adds Claude's own code review, and writes the findings register |
| `security-fix` | "Help me fix SEC-004" | Explains the risk, gets the developer's root cause, guides or drafts the fix, looks for the same flaw elsewhere, and handles secret rotation, false positives and accepted risks |
| `security-verify` | "Verify" | Rescans, confirms each fix, reopens anything that came back, finds new issues, applies the gate, and produces the sign-off report and optional CI scan |

## What it checks

- **Secrets:** committed `.env` files, keys in code and in **git history**, `.env` files in public build folders, secrets in front-end bundles (`NEXT_PUBLIC_`, `VITE_`, `REACT_APP_`)
- **Code:** injection (SQL, NoSQL, command, code), XSS, SSRF, path traversal, unsafe deserialisation, weak cryptography
- **Access control and authentication:** missing ownership checks (IDOR), unprotected admin functions, session and JWT flaws, brute-force protection. Scanners rarely find these; Claude's review does most of the work here.
- **Dependencies:** known vulnerabilities, grouped by package, with the version that fixes them
- **Infrastructure and CI:** Dockerfiles, compose, Kubernetes and Terraform files, and GitHub Actions workflows (unpinned actions, excessive permissions, unsafe triggers)
- **Business logic:** skipped steps, client-side prices, race conditions, unverified webhooks
- **AI features:** the OWASP Top 10 for LLM Applications, including prompt injection, excessive agency, improper output handling and unbounded consumption
- **Compliance flags** for health, payment and personal data. These are prompts to check with the right person, not legal advice.

Findings are mapped to the **OWASP Top 10:2025** and **CWE**.

## How the loop works

```
security-audit  →  register (.security-audit/)  →  security-fix  →  security-verify
                                                       ↑                  │
                                                       └── gate failed ───┘
                                                gate passed → sign-off report + CI scan
```

- **One register:** `findings.json` with a readable `FINDINGS.md`, a run summary for every scan and an append-only `history.jsonl`. Optional SARIF export for GitHub's Security tab.
- **Fingerprints** match the same issue across scans even when the code moves, so rescans say exactly what was fixed, what is still open, what came back and what is new.
- **Only a rescan closes a finding.** Developers can mark a finding Fixed; only `security-verify` can mark it Verified, after reading the fix.
- **People decide.** False positives need a second person to agree. Accepted risks need the approver, a reason and an expiry date. Critical findings can't be accepted by default.
- **The default gate:** no open Critical or High findings, and every Medium fixed or formally accepted.

## Scanners

The plugin uses free, open-source scanners where they're installed, and offers to install missing ones with your approval:

| Purpose | Tool |
|---|---|
| Secrets | Gitleaks |
| Code | Semgrep Community Edition |
| Dependencies | OSV-Scanner (npm audit and pip-audit optional) |
| Infrastructure as code | Checkov |
| CI workflows | zizmor |
| Language-specific, optional | Bandit, Trivy and others |

Without scanners, Claude reviews the code alone and labels the audit **"Reduced coverage"**.

## Connectors (optional)

Nothing is required. If you have them connected, and approve each one when the audit asks:

- **GitHub:** reads existing code-scanning, secret-scanning and Dependabot alerts, creates issues for findings, and keeps them in step with the register
- **Jira, Linear or Asana:** findings as tickets
- **SonarQube or Snyk:** existing results merged into the register instead of scanning twice

Claude always asks before creating or changing anything in another system. Tickets never contain secrets or exploit steps.

## Make it yours

Copy `security-house-rules.md` into your project and edit it to set:

- approver
- blocking severities
- acceptance period
- report format
- compliance regions
- allowed connectors
- your own rules

## What this plugin does and does not do

- **Instructions only.** It contains only instructions for Claude (skills) and reference notes. It has no MCP servers, hooks or scripts of its own.
- **Scanners run on your machine,** with your approval. Before running any scanner that contacts the internet, Claude says what it sends. For example, OSV-Scanner sends package names and versions, not code.
- **It doesn't attack running systems.** The audit is static by default. Live testing is only for local or staging environments, with your written confirmation that you're authorised.
- **It never prints or stores full secret values.**
- **A passed gate isn't a guarantee.** It means no blocking findings remained against the stated scope and checks at a given commit. It doesn't mean the application is free of vulnerabilities. For high-risk applications, also commission a penetration test of the running system.
- **It works alongside Anthropic's security-guidance plugin for Claude Code.** That plugin checks code as it's written; this one audits the whole project and tracks fixes to sign-off.

## Benchmark

On a deliberately vulnerable Node.js test application, version 1.0.0 found 28 of 29 known issues. Through the fix loop, it verified real fixes, rejected a fake fix and caught a new flaw introduced during fixing. See [BENCHMARK.md](BENCHMARK.md) for the method and the caveats.

## Use only on code you are authorised to audit

The plugin asks you to confirm authorisation before it starts.

## Licence

MIT. See [LICENSE](LICENSE). Privacy: see [PRIVACY.md](PRIVACY.md).

## Feedback

Please raise issues and suggestions on the plugin's GitHub repository.
