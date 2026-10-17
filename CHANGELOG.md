# Changelog

## 1.1.0 (October 2026)

- Adds support for Cursor and OpenAI Codex alongside Claude. New files: a Cursor manifest (`.cursor-plugin/plugin.json`), a root `plugin.json` in the Agent Plugins format with the listing details OpenAI's plugin directory uses, and a logo in `assets/`.
- The skills no longer assume Claude is the agent. Each run records the agent and model that did the review, where the tool states them.
- Fixed: a different secret found later under the same key name in the same file is now registered as a new finding. Before, it could match an older finding and inherit its status.
- Fingerprints are now defined exactly (encoding, whitespace and hex case), so every agent and shell computes the same value.
- Registers written by version 1.0 stay readable and are converted the first time 1.1.0 writes them. See "Registers from version 1.0" in `skills/security-audit/references/register-format.md`. Everyone who shares a register should use 1.1.0 or later.
- README: install steps for Claude, Cursor and Codex.

## 1.0.x (October 2026)

- First release, for Claude Code and the Claude app.
