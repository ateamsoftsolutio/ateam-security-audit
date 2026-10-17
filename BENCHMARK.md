# Benchmark: version 1.0.0

Tested in October 2026 on a deliberately vulnerable application, before release. These results are from a single run on one test application. Read the caveats before quoting them.

## Test application

- [OWASP NodeGoat](https://github.com/OWASP/NodeGoat) (Node.js and Express), a public training app with documented vulnerabilities.
- To make the test fairer, we removed the comments that describe the flaws, the tutorial pages and most of the NodeGoat branding.
- We then planted seven extra issues the original app doesn't have:
  - a committed `.env` with a production database password and a live-format Stripe key
  - an AWS key left only in git history
  - an API key in a public browser script
  - an AI assistant with four LLM flaws
- The audit ran with all core scanners installed: Gitleaks, Semgrep, OSV-Scanner, Checkov, zizmor and npm audit.

## Results

**Audit (run R1): 28 of 29 known issues found.**

| Group | Found |
|---|---|
| NodeGoat's documented issues | 21 of 22 |
| Planted issues | 7 of 7 |

- **Missed:** log injection (user input written to the log without neutralising new lines).
- **Total findings:** 51 (5 Critical, 13 High, 19 Medium, 12 Low, 2 Info). Most of the rest were real issues outside our list, such as end-of-life Node images, `.env` copied into the Docker image, and secrets logged at start-up.
- **Triage dismissals:** 6 scanner hits were dismissed as false positives with reasons, for human review.
- **Secrets:** none of the four planted secrets appeared in the register or reports. All were masked.
- **Scanners vs Claude:** Semgrep found the `eval()` injection, the open redirect, the cookie flaws and the committed private key. It didn't find the access-control flaws (IDOR, missing admin check), the NoSQL `$where` injection, the SSRF, the plain-text passwords or any AI flaw. Claude's review found those.

**Fix-and-verify loop (runs R1 to R2).** We then made four "developer fixes" in one commit: two real, one fake, plus one that deleted a leaked key without rotating it. The same commit also added a new debug route.

| Change | Outcome |
|---|---|
| `eval()` replaced with `parseInt` | Verified |
| User ID taken from the session instead of the URL | Verified |
| `String(threshold).trim()` plus `// nosemgrep` on the NoSQL injection | Rejected: the root cause was vague, and the change doesn't stop the injection |
| Leaked API key file deleted, key not rotated | Kept Open until rotation is confirmed |
| New unauthenticated `/debug/config` route exposing environment variables | Found as a new Critical |

Also in that loop:

- A false-positive claim that was really "low risk" was redirected to the accepted-risk route.
- An accepted-risk request longer than 90 days was capped.

## Caveats

- **The model recognised the app.** NodeGoat is public and well known. Despite our changes, a leftover comment and package metadata let the model identify its origin, so results on unfamiliar, private code are likely to be lower.
- **One run, one stack, one model.** The run used Claude. Results vary between runs, and in Cursor, Codex or other tools the review is done by whichever model you choose.
- **No penetration test.** The benchmark measures static audit and fix verification, not testing of a running system.
- **Changes since the run.** We improved the skills based on this test (dependency grouping, masked viewing of secret files, secret matching across deletions, diff review in the fix step). Those changes were not re-benchmarked. Version 1.1.0 wasn't re-benchmarked either, and the plugin hasn't been benchmarked outside Claude.

We'll publish further benchmarks on other stacks as we run them.
