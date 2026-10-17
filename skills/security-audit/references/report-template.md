# Sign-off report template

Produce this only when a **full** verify run has passed the gate. Write it in plain language for a technical manager or a client's IT lead. Use the format from the house rules: a Markdown file in `.security-audit/report/` by default, or a Word or shared document if preferred and you can produce one in this environment. If you can't, write the Markdown file and tell the user.

Never include secret values (masked or not, beyond the type of secret), exploit steps, or personal data.

---

```markdown
# Security audit report: <project>

**Result:** Gate passed on <date> at commit <hash>
**Coverage:** Full / Reduced (<what was not covered by a tool>)
**Audit period:** <first run date> to <final run date> · <number> runs
**Approved by:** <name, role>, <date>

## 1. Summary
Two or three sentences: what was audited, what was found at the start, what was fixed, and what remains as accepted risk.

|                     | Critical | High | Medium | Low | Info |
|---------------------|----------|------|--------|-----|------|
| Found (all runs)    |          |      |        |     |      |
| Verified fixed      |          |      |        |     |      |
| Accepted risk       |          |      |        |     |      |
| False positive      |          |      |        |     |      |
| Open (non-blocking) |          |      |        |     |      |

## 2. Scope and method
- **In scope:** ...   **Out of scope:** ...
- **Application context:** exposure, data types, countries and sectors
- **Automated tools:** <tool and version, one per line>
- **AI agent and model:** <as recorded for each run, or "not recorded (version 1.0)">
- **Code review:** areas covered by AI-assisted review (access control, authentication, injection, SSRF, files, secrets and cryptography, configuration, supply chain and CI, logging and errors, business logic, AI features)
- **Gate applied:** <the gate, and the approver>

## 3. Findings and outcomes
| ID | Severity | Title | Category | Outcome | Run closed |
|----|----------|-------|----------|---------|------------|

## 4. Accepted risks
| ID | Severity | Reason | Compensating control | Approved by | Expires |
|----|----------|--------|----------------------|-------------|---------|

## 5. Exposed secrets
| ID | Secret type | Location | Rotated on | By |
|----|-------------|----------|------------|----|

## 6. Compliance flags
Points to confirm with the responsible person (data protection officer, compliance lead or legal adviser). These are not legal findings.

## 7. Recommendations
- Keep the CI security scan running on every pull request (in place: yes / no)
- Re-audit after significant changes, and at least every <n> months
- Re-check accepted risks before their expiry dates
- Penetration testing of the running application before go-live, where the data or exposure warrants it
- Any further hardening items from the Low and Info findings

## 8. Limitations
This audit combined automated scanning with AI-assisted review of the source code at commit <hash>. It did not include penetration testing of running systems, review of production infrastructure beyond the configuration files in the repository, or social-engineering testing. A passed gate means that no blocking findings remained against the stated scope and checks at that commit. It is not a guarantee that the application is free of vulnerabilities. Security should be re-checked after significant changes.

## Appendix: run history
| Run | Date | Commit | Type | Open C/H/M/L | Gate |
|-----|------|--------|------|--------------|------|
```
