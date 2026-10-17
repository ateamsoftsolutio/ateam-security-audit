# Security house rules

Copy this file into your project root (or into `.security-audit/`) as `security-house-rules.md` and change the values to suit your organisation. The security skills read it at the start of every run. Delete a line to use the plugin's default for that setting.

- **Approver for accepted risks and sign-off:** Product Owner
- **Blocking severities:** Critical, High
- **Medium findings:** must be fixed or formally accepted (fix-or-accept). Alternative: "listed, not blocking".
- **Maximum acceptance period:** 90 days
- **Can Critical findings be accepted?** No
- **Register in git:** commit `.security-audit/` (except `raw/`) in private repositories; never in public ones
- **Report format:** Markdown file in `.security-audit/report/`
- **Countries and sectors for compliance flags:** (list them, for example "UAE and Saudi Arabia, healthcare")
- **Connectors allowed:** (for example "GitHub for issues and alerts; no cloud code scanners")
- **May client or project code be sent to cloud scanning services?** Only with written consent recorded in the scope
- **Create tickets for:** Critical and High findings; Mediums grouped by area
- **Scanners approved:** Gitleaks, Semgrep Community Edition, OSV-Scanner, Checkov, zizmor, plus stack-specific tools as needed
- **CI platform:** (for example GitHub Actions)
- **Full re-audit interval:** every 6 months, or after significant changes
- **Anything else the agent must always follow:** (add your own rules here)
