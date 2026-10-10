# Scanners

Use free, open-source scanners where they're installed or can be installed with the user's approval. Claude reviews and triages their output; they don't replace Claude's review, and Claude doesn't replace them.

Save all output to `.security-audit/raw/R<n>/`. That folder is never committed.

## Check what's installed

```bash
for t in gitleaks semgrep osv-scanner trivy checkov zizmor bandit pip-audit npm; do
  printf "%-12s" "$t"; command -v "$t" >/dev/null 2>&1 && echo "installed" || echo "missing"; done
```

## Core set

| Category | Tool | Install (choose one) | What leaves the machine |
|---|---|---|---|
| Secrets in files and git history | **Gitleaks** | `brew install gitleaks` · `go install github.com/zricethezav/gitleaks/v8@latest` | Nothing |
| Code (SAST) | **Semgrep** (Community Edition) | `pipx install semgrep` · `brew install semgrep` | Downloads rule packs. No code is sent when run with `--metrics=off`. |
| Dependencies | **OSV-Scanner** | `brew install osv-scanner` · `go install github.com/google/osv-scanner/v2/cmd/osv-scanner@latest` | Package names and versions go to the osv.dev API. No code. Offline mode is available. |
| Infrastructure as code, Dockerfiles | **Checkov** | `pipx install checkov` | Nothing with `--skip-download` and no API key |
| CI workflows (GitHub Actions) | **zizmor** | `pipx install zizmor` · `brew install zizmor` | Nothing with `--offline` |

## Optional, by stack

| Tool | When | Install | What leaves the machine |
|---|---|---|---|
| **Trivy** | Containers, images, broad file-system scan | `brew install trivy` (see the warning below) | Downloads its vulnerability database. No code. |
| **npm audit** | Node projects with a lock file | Comes with npm | The dependency tree goes to the npm registry |
| **pip-audit** | Python projects | `pipx install pip-audit` | Package names and versions go to PyPI or OSV |
| **Bandit** | Python code | `pipx install bandit` | Nothing |
| gosec, Brakeman, SpotBugs with Find Security Bugs | Go, Ruby on Rails, Java | The tool's official instructions | Nothing |

Always:

- **Ask before installing**, and use official package managers or the tool's signed releases.
- **Never pipe a downloaded script into a shell.**
- **Pin versions in CI.**

**Supply-chain warning: Trivy.** In March 2026 attackers published malicious Trivy releases: binary v0.69.4, and Docker images v0.69.5 and v0.69.6. All tags of `aquasecurity/trivy-action` up to 0.34.2 and of `setup-trivy` were also briefly compromised. Check the installed version. Don't use those versions, and pin GitHub Actions to full commit SHAs. If one of them ever ran, tell the user to rotate the secrets that environment could reach.

**TruffleHog**, if the user prefers it to Gitleaks: run it with `--no-verification` unless the user agrees otherwise. Verification sends each secret it finds to the provider's API to test whether it's live.

## Commands

Run from the project root. Replace `R1` with the current run number.

```bash
OUT=.security-audit/raw/R1 && mkdir -p "$OUT"

# Secrets: git history, then the working tree (catches untracked .env files).
# The config keeps Gitleaks out of earlier audit output and node_modules.
cat > .security-audit/raw/gitleaks.toml <<'EOF'
[extend]
useDefault = true

[[allowlists]]
description = "Exclude audit output and dependencies"
paths = ['''(^|/)\.security-audit/''', '''(^|/)node_modules/''']
EOF
gitleaks git --redact -c .security-audit/raw/gitleaks.toml --report-format json --report-path "$OUT/gitleaks-history.json" .
gitleaks dir --redact -c .security-audit/raw/gitleaks.toml --report-format json --report-path "$OUT/gitleaks-files.json" .
#   Gitleaks older than 8.19: gitleaks detect --source . --redact ... (add --no-git for the working tree)
#   Gitleaks older than 8.25: use [allowlist] instead of [[allowlists]] in the config

# Code
semgrep scan --config p/default --config p/owasp-top-ten --metrics=off --json -o "$OUT/semgrep.json" --quiet .
#   Add a language pack if useful: --config p/javascript, p/typescript, p/python, p/java, p/php, p/golang
#   Don't use --config auto: it requires metrics to be sent.

# Dependencies
osv-scanner scan source -r --format json --output-file "$OUT/osv.json" .
#   OSV-Scanner v1: osv-scanner -r --format json --output "$OUT/osv.json" .
#   Offline: add --offline-vulnerabilities --download-offline-databases
npm audit --json > "$OUT/npm-audit.json"                    # Node, if a lock file exists
pip-audit -r requirements.txt -f json -o "$OUT/pip-audit.json"  # Python

# Infrastructure and CI
checkov -d . --quiet --compact -o json --skip-download > "$OUT/checkov.json"
zizmor --offline --format json .github/workflows > "$OUT/zizmor.json"   # if the folder exists

# Optional
bandit -r . -f json -o "$OUT/bandit.json" -x ./node_modules,./.venv,./venv
trivy fs --scanners vuln,secret,misconfig --format json --output "$OUT/trivy.json" .
```

Many scanners exit with a non-zero code when they find issues:

| Exit code | Meaning |
|---|---|
| 1 | Gitleaks, Semgrep with `--error`, OSV-Scanner, Checkov and npm audit found issues |
| 10 or higher | zizmor found issues |

Check the output file before treating any exit code as a failure.

**Recording versions:** if `gitleaks version` prints "version is set by build process" (built with `go install`), record the version as "unknown (built from source)".

**Known blind spots:**

- Gitleaks often misses passwords inside connection strings such as `DATABASE_URL`. Check 5 below catches them.
- Checkov skips `docker-compose.yml`. Review compose files yourself with section 7 of `checklist.md`.
- Semgrep rarely finds missing access checks, NoSQL `$where` injection, SSRF through string-built URLs, or AI-specific flaws. Claude's review must cover those.

## Viewing sensitive files safely

Never display the raw contents of an environment file, key file, keystore or credentials file, or any file a scanner flagged for secrets. Whenever you need to look at one (in the working tree or in git history), pipe it through a masking filter you write once per run and save in `.security-audit/raw/`. The filter must:

- hide the password part of any connection string (`scheme://user:••••@server`)
- hide the value of any key whose name contains password, passwd, pwd, secret, token, api key, private key or access key
- shorten any other long token-like value to its first 4 and last 4 characters (`sk-l••••a9F2`)

It's acceptable for the filter to over-mask harmless long values. Check it on a sample before relying on it.

## Leak check before presenting

The register must never contain a secret. Scan it before showing results:

Run `gitleaks dir --redact --no-banner --exit-code 1 <file>` on each file in `.security-audit/` except the `raw/` folder. Any file it flags must be fixed before you go on.

If Gitleaks isn't available, grep the register for the masked values' surrounding text and for long token-like strings.

## Checks that need no scanner

Always do these, even in reduced-coverage mode. Write the commands for the project's own layout and keep all output masked.

1. **Sensitive files committed to git:** list tracked files whose names look like environment files, private keys, keystores, SSH keys, credential or secrets files, database files or backups.
2. **Environment files in publishable folders:** look for environment files under folders a web server or build could publish (`public`, `static`, `www`, `dist`, `build`, `out`, `assets`).
3. **Secrets exposed to the browser:** look for public front-end variable prefixes (`NEXT_PUBLIC_`, `VITE_`, `REACT_APP_`, `EXPO_PUBLIC_`, `NUXT_PUBLIC_`) whose names suggest a secret, private key, password, token, service role or API key.
4. **Source maps in build output:** look for `.map` files under build or public folders.
5. **Passwords inside connection strings:** search for `user:password@` patterns in URLs, because Gitleaks often misses these. Mask the password in any output.
6. **Lock file in step with the manifest:** check that every dependency in the manifest (for example `package.json`) appears in the lock file. Anything missing wasn't scanned.
7. **`.gitignore` coverage:** check that environment and key file patterns are ignored.

Some keys are designed to be public, such as Stripe publishable keys, Firebase web config and Supabase anon keys. Don't report those as leaks. Do check that the access rules behind them (Firebase rules, Supabase row-level security) are in place, and report them as Needs check if you can't confirm.

## Reading the output

- **Gitleaks:** `File`, `StartLine`, `RuleID`, `Commit`. With `--redact` the secret shows as `REDACTED`. A leak found only in history means the file was removed, but the secret is still exposed and must be rotated.
- **Semgrep:** `results[].check_id`, `path`, `start.line`, `extra.severity` (ERROR, WARNING or INFO), `extra.metadata.cwe` and `extra.metadata.owasp`.
- **OSV-Scanner:** `results[].packages[].package` and `vulnerabilities[]`. Expect hundreds of entries in older projects. Group them by package.
- **Checkov:** `results.failed_checks[]` with `check_id`, `file_path` and `file_line_range`.
- **zizmor:** `ident` (for example `unpinned-uses` or `excessive-permissions`). The file path is at `locations[].symbolic.key.Local.verbatim_path`.

Scanner severity is a starting point only. Re-rate every finding with `severity.md`.
