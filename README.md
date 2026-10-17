# Security Audit Loop by ATeam

**Audit. Fix. Verify. Repeat until the gate passes.**

Security Audit Loop is a plugin for AI coding agents. It runs a full security audit of a software project and keeps the team on it until every blocking issue is fixed and confirmed. It's built for **Claude Code**, the **Claude app** (with a project folder), **Cursor** and **OpenAI Codex**.

Most security checks give developers one long list and stop. This plugin keeps going. Every finding goes into a tracked register with a permanent ID and owner. Developers fix and explain each issue. A rescan confirms the fix before it's closed, and a gate decides when the project is ready for sign-off.

It was built by [ATeam Soft Solutions](https://www.ateamsoftsolutions.com), is used by ATeam's delivery teams, and is shared as a contribution to the community.

## Install

### Claude Code and the Claude app

Search for "Security Audit Loop by ATeam" in Claude's plugin directory, or in Claude Code run:

```
/plugin install ateam-security-audit --marketplace ateamsoftsolutio/ateam-security-audit
```

On older versions of Claude Code, add the marketplace first:

```
/plugin marketplace add ateamsoftsolutio/ateam-security-audit
/plugin install ateam-security-audit@ateam-security
```

### Cursor

1. Open a terminal in Cursor's local plugin folder, `.cursor/plugins/local` inside your home folder. Create the folder if it doesn't exist.
2. Run:

   ```
   git clone https://github.com/ateamsoftsolutio/ateam-security-audit
   ```

3. In Cursor, run **Developer: Reload Window**.

To update later, run `git pull` in the `ateam-security-audit` folder and reload again. On Cursor Enterprise, local plugin imports are off by default, so ask your Cursor admin.

### OpenAI Codex

```
codex plugin marketplace add ateamsoftsolutio/ateam-security-audit
codex plugin add ateam-security-audit@ateam-security
```

Start a new Codex session after installing. To update later, run `codex plugin marketplace upgrade ateam-security`.

### Other agents

The repository also has a root `plugin.json` in the open [Agent Plugins](https://agent-plugins.org) format, so other agents that support that format may load it. We support Claude, Cursor and Codex.

## The three skills

| Skill | Say | What it does |
|---|---|---|
| `security-audit` | "Run Security Audit" | Scopes the project, maps the attack surface, runs open-source scanners, adds the agent's own code review, and writes the findings register |
| `security-fix` | "Help me fix SEC-004" | Explains the risk, gets the developer's root cause, guides or drafts the fix, looks for the same flaw elsewhere, and handles secret rotation, false positives and accepted risks |
| `security-verify` | "Verify" | Rescans, confirms each fix, reopens anything that came back, finds new issues, applies the gate, and produces the sign-off report and optional CI scan |

Asking in plain words works in every tool. You can also call a skill directly: type `/security-audit` in Cursor's Agent chat, or type `$` in Codex and choose the skill.

## What it checks

- **Secrets:** committed `.env` files, keys in code and in **git history**, `.env` files in public build folders, secrets in front-end bundles (`NEXT_PUBLIC_`, `VITE_`, `REACT_APP_`)
- **Code:** injection (SQL, NoSQL, command, code), XSS, SSRF, path traversal, unsafe deserialisation, weak cryptography
- **Access control and authentication:** missing ownership checks (IDOR), unprotected admin functions, session and JWT flaws, brute-force protection. Scanners rarely find these; the agent's own review does most of the work here.
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
- **One register format in every tool.** Each run records which agent and model did the review, where the tool states them.

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

Without scanners, the agent reviews the code alone and labels the audit **"Reduced coverage"**.

## Connectors (optional)

Nothing is required. If your tool has them connected (Claude connectors, or MCP servers in Cursor and Codex), and you approve each one when the audit asks:

- **GitHub:** reads existing code-scanning, secret-scanning and Dependabot alerts, creates issues for findings, and keeps them in step with the register
- **Jira, Linear or Asana:** findings as tickets
- **SonarQube or Snyk:** existing results merged into the register instead of scanning twice

The skills always ask before creating or changing anything in another system. Tickets never contain secrets or exploit steps.

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

- **Instructions only.** It contains only instructions for the agent (skills) and reference notes. It has no MCP servers, hooks or scripts of its own.
- **The review is only as good as the model running it.** The skills run on whichever model your tool uses. Each run records the agent and model in the register, where the tool states them.
- **Scanners run on your machine,** with your approval. Before running any scanner that contacts the internet, the agent says what it sends. For example, OSV-Scanner sends package names and versions, not code.
- **It doesn't attack running systems.** The audit is static by default. Live testing is only for local or staging environments, with your written confirmation that you're authorised.
- **The skills tell the agent never to print or store full secret values,** and to check the register for leaked secrets before showing it.
- **A passed gate isn't a guarantee.** It means no blocking findings remained against the stated scope and checks at a given commit. It doesn't mean the application is free of vulnerabilities. For high-risk applications, also commission a penetration test of the running system.
- **In Claude Code, it works alongside Anthropic's security-guidance plugin.** That plugin checks code as it's written; this one audits the whole project and tracks fixes to sign-off.

## Benchmark

Version 1.0.0 of the plugin, running with Claude, found 28 of 29 known issues in a deliberately vulnerable Node.js test application. Through the fix loop, it verified real fixes, rejected a fake fix and caught a new flaw introduced during fixing. Results with other models will differ. See [BENCHMARK.md](BENCHMARK.md) for the method and the caveats.

## Use only on code you are authorised to audit

The plugin asks you to confirm authorisation before it starts.

## Privacy

See the [Privacy policy](PRIVACY.md). In short: ATeam collects nothing, and the plugin has no servers of its own. Your code is processed by the AI tool you run it in, under that tool's terms.

## Files in this repository

| File | Used by |
|---|---|
| `.claude-plugin/plugin.json`, `.claude-plugin/marketplace.json` | Claude Code and Claude's plugin directory. Codex also reads the marketplace file. |
| `.cursor-plugin/plugin.json` | Cursor |
| `plugin.json` | Codex and other agents that support the Agent Plugins format |
| `skills/` | All of them |

## Licence

MIT. See [LICENSE](LICENSE).

## Feedback

Please raise issues and suggestions on the plugin's GitHub repository.
