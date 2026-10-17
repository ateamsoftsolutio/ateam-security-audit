# Privacy policy: Security Audit Loop by ATeam

Effective date: 17 October 2026

This policy covers the "Security Audit Loop by ATeam" plugin (`ateam-security-audit`), published by ATeam Soft Solutions.

## What the plugin is

The plugin contains only written instructions for AI agents (skills) and reference notes. It's built for Claude, Cursor and OpenAI Codex, and may load in other tools that support the same format. It has no MCP servers, hooks, scripts or other code that runs on its own. ATeam doesn't run any server or service for it.

## Data ATeam collects

None. The plugin has no analytics, no telemetry and no accounts. ATeam doesn't receive anything you or your AI agent do with it.

## Data the skills handle in your AI tool

When you use the skills, the agent reads the source code, configuration and git history of the project you choose to audit. This can include personal data in code or test fixtures, and secrets such as passwords or API keys. The agent:

- writes the findings register, run summaries and reports to the `.security-audit/` folder in your project
- is instructed never to store or display full secret values, and to mask them
- processes this content under the terms of the AI tool and model provider you use (for example Anthropic, Cursor or OpenAI) and your account settings. ATeam has no access to it.

## Outside services

- **Open-source scanners:** the agent may run scanners on your machine, with your approval. Some contact the internet:
  - Semgrep downloads rule packs. Code isn't sent when it's run as the plugin instructs.
  - OSV-Scanner, npm audit and pip-audit send package names and versions to public vulnerability databases. No code is sent.
  - Other scanners download vulnerability databases.

  The agent tells you what each scanner sends before running it.
- **Connectors:** if you connect services such as GitHub, Jira or Snyk (as connectors or MCP servers) and approve their use during an audit, the agent sends them only what you approve, such as ticket text. That data is governed by your agreement with those services.

## Retention

ATeam keeps no data from the plugin's use, because it receives none. The register and reports stay in your project until you delete them.

## Children

The plugin is meant for professional use and isn't intended for anyone under 18.

## Your responsibilities

Audit only code you own or are authorised to audit. Get the owner's consent before sending their code or findings to any outside service.

## Changes

Any change to this policy will be published in this file in the plugin's GitHub repository, with a new effective date.

## Contact

ATeam Soft Solutions, director@ateamsoftsolutions.com, https://www.ateamsoftsolutions.com
