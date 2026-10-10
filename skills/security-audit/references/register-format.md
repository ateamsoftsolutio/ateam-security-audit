# The findings register

The register is the single source of truth for the audit loop. All three skills read and write it in exactly this format, so that every run can be compared with the last.

## Folder layout

```
.security-audit/
├── scope.md            Scope, answers, attack surface map, coverage
├── findings.json       The register (source of truth)
├── FINDINGS.md         Readable view, regenerated from findings.json after every change
├── history.jsonl       Append-only event log
├── runs/R1.md          One summary per scan run
├── report/             Sign-off reports
├── findings.sarif      Optional SARIF export for GitHub's Security tab
├── raw/R1/             Raw scanner output (never committed)
└── .gitignore          Contains: raw/
```

**Git rules:**

- `raw/` is never committed: scanner output can contain code snippets and partial secrets.
- In a **private** repository, commit the rest so the whole team shares one register, unless the house rules say otherwise.
- In a **public** repository, add `.security-audit/` to the root `.gitignore`. The register is a map of the application's weaknesses.

## scope.md

```markdown
# Security audit scope: <project>

- **Audited by:** <name, as stated> · **Date:** <date> · **Commit:** <hash>
- **Purpose and users:** ...
- **Stack:** ...
- **Exposure:** Internet-facing / internal / not deployed
- **Data handled:** ...
- **Countries and sectors:** ...
- **Authorisation confirmed by:** <name, as stated>, <date>
- **In scope:** ... · **Out of scope:** ...
- **Repository:** private / public. Register committed: yes / no.
- **Connectors and consent:** GitHub (tickets: yes; code: not sent) · Snyk (not approved) · ...
- **Gate:** <default or as agreed, with approver>
- **Coverage:** Full / Reduced (<categories relying on Claude review alone>)

## Attack surface map
Entry points · Authentication · Authorisation · Data stores and external calls · AI features · Infrastructure and CI · Size

## Hotspots
1. ...
```

## findings.json

```json
{
  "schema": "security-audit-register/1",
  "project": "Clinic booking portal",
  "approver": "Product Owner",
  "gate": { "blocking": ["Critical", "High"], "medium": "fix-or-accept", "maxAcceptDays": 90, "criticalAcceptable": false },
  "nextId": 5,
  "runs": [
    { "id": "R1", "date": "2026-10-06", "commit": "a1b2c3d", "type": "full", "coverage": "Full",
      "tools": [{ "name": "semgrep", "version": "1.179.0" }, { "name": "gitleaks", "version": "8.28.0" }],
      "gate": "failed" }
  ],
  "findings": [
    {
      "id": "SEC-004",
      "title": "SQL injection in patient search",
      "severity": "High",
      "confidence": "Confirmed",
      "owasp": "A05:2025",
      "cwe": "CWE-89",
      "llm": null,
      "locations": [
        { "file": "api/patients/search.js", "line": 42, "symbol": "searchPatients", "fingerprint": "3f9a1c0be47d2a18",
          "fpInput": "CWE-89|api/patients/search.js|searchPatients|const sql = \"SELECT * FROM patients WHERE name = '\" + req.query.name + \"'\";",
          "resolved": false }
      ],
      "sources": ["semgrep:javascript.express.security.injection.tainted-sql-string", "claude-review"],
      "evidence": "req.query.name is concatenated into the SQL string passed to db.query()",
      "impact": "Any logged-in user can read or change the whole patients table",
      "fix": "Use a parameterised query; validate name length and characters",
      "compliance": ["Health data"],
      "status": "Open",
      "owner": null,
      "firstSeen": { "run": "R1", "commit": "a1b2c3d", "date": "2026-10-06" },
      "lastSeen": { "run": "R1", "commit": "a1b2c3d" },
      "rootCause": null,
      "fixNote": null,
      "fixCommit": null,
      "verification": null,
      "secret": null,
      "falsePositive": null,
      "acceptedRisk": null,
      "ticket": null
    }
  ],
  "triageDismissals": [
    { "run": "R1", "source": "semgrep:python.django.security.django-no-csrf-token", "location": "app/views/login.html:107",
      "reason": "Django rule matched a Swig template in an Express app; CSRF is covered by SEC-009" }
  ]
}
```

**Fields that hold extra detail when used:**

| Field | Contents |
|---|---|
| `secret` | `{ "type": "PostgreSQL password", "keyName": "DATABASE_URL", "masked": "pg_p••••x9Q1", "inHistoryOnly": false, "removedFromCode": false, "rotated": false, "rotatedBy": null, "rotatedOn": null }` |
| `verification` | `{ "run": "R3", "commit": "e7f8a9b", "evidence": "Query now parameterised (search.js:40-44); Semgrep rule no longer matches", "test": "added: test/search.sql-injection.test.js, passes" }`. If there's no test, say why, for example `"none: project has no test suite"`. |
| `falsePositive` | `{ "reason": "...", "raisedBy": "Anu", "claudeView": "agrees / disagrees: ...", "reviewerRequested": "Rahul", "reviewer": null, "date": null }` |
| `acceptedRisk` | `{ "reason": "...", "compensatingControl": "...", "proposedBy": "Rahul", "expiresRequested": "2027-01-05", "approverRequired": "Product Owner", "approver": null, "date": null, "expires": null }`. `approver`, `date` and `expires` are filled in only on approval; `expires` is counted from the approval date. |
| `ticket` | `{ "system": "github", "ref": "#118", "url": "<link to the issue>" }` |

Each location also stores `fpInput`, the exact values used for its fingerprint (never a secret), so later runs can reproduce it.

**Never store a full secret value anywhere in the register.**

## Statuses and who can change them

| Status | Meaning |
|---|---|
| `Open` | Present and not yet fixed |
| `Fixed` | The developer says it's fixed; waiting for a rescan |
| `Verified` | A rescan confirmed the fix. Closed. |
| `Reopened` | It was Verified, and a later scan found it again. Treated as Open. |
| `False positive (pending review)` | A developer says it isn't a real issue; a second person must agree |
| `False positive` | The second person agreed. Closed. |
| `Accepted risk (pending approval)` | The team proposes not to fix it; the approver must agree |
| `Accepted risk` | Approved, with an expiry date |

| Change | Made by | Requires |
|---|---|---|
| Open or Reopened → Fixed | `security-fix` | A fix note. For Critical and High, the developer's one-line root cause. For secrets, rotation confirmed by the developer. |
| Fixed → Verified | `security-verify` only | The scanner no longer reports it, Claude has read the changed code and agrees it removes the root cause, and any test added for it passes. Every location must be resolved. |
| Fixed → Open | `security-verify` | Still present; note "Fix did not resolve" with the reason |
| Verified → Reopened | `security-verify` | Found again |
| Open → False positive (pending review) → False positive | `security-fix` | A reason, plus a reviewer who is a named person other than the one who raised it. Claude records its own view, but can't be the reviewer. |
| Open → Accepted risk (pending approval) → Accepted risk | `security-fix` | A reason, a compensating control if any, the approver named in the house rules, and an expiry date no more than `maxAcceptDays` after the approval date. A longer request is capped, and the user is told. Critical findings can't be accepted unless `criticalAcceptable` is true. |
| Accepted risk → Open | `security-verify` | The expiry date has passed |

Names are recorded as the user states them, followed by "(stated)" in `history.jsonl`. Claude can't verify identities; commit and ticket history provide stronger evidence.

## Fingerprints

A fingerprint identifies the same issue across runs, even when line numbers change. Compute one per location as the first 16 hex characters of SHA-256 over these values, joined with `|`:

| Kind | Values |
|---|---|
| Code issues | `CWE` · `file path` · `enclosing function, method or route` (or `-`) · `primary code line with whitespace collapsed` |
| Secrets | `CWE-798` · `file path where it was first found` (keep this path even after the file is deleted) · `scanner rule ID`, or `claude:<secret type>` if found by review · the **variable or key name** holding it, exactly as written (for example `DATABASE_URL` or `apiKey`), or `-`. Never include the secret or a code line containing it. |
| Dependencies | `dep` · `ecosystem` · `package name` (no version, so an upgrade that's still vulnerable matches) |
| Configuration and IaC | `CWE` · `file path` · `resource or key name` |

Example:

```bash
printf '%s|%s|%s|%s' "CWE-89" "api/patients/search.js" "searchPatients" \
  "$(sed -n '42p' api/patients/search.js | tr -s '[:space:]' ' ' | sed 's/^ //;s/ $//')" | sha256sum | cut -c1-16
```

**Matching on a rescan:**

1. Look for an exact fingerprint match.
2. If there isn't one, look for the same CWE in the same file and function. That means the code changed but the issue may still be there. Read it and decide. Record "matched by function" in the history note.
3. If nothing matches and the issue is gone, the location is resolved. Confirm by reading the code.

**Secrets are different.** Match them using the `fpInput` stored in the register. A committed secret is still present, even if its file has been deleted, until `secret.rotated` is true: it remains in git history. Removing it from the code alone sets `removedFromCode: true` and leaves the finding Open.

## FINDINGS.md layout

```markdown
# Security findings: <project>

Last run R3 · commit e7f8a9b · 2026-10-08 · Gate: FAILED · Coverage: Full

|            | Critical | High | Medium | Low | Info |
|------------|----------|------|--------|-----|------|
| Open       | 0        | 1    | 3      | 6   | 2    |
| Fixed, awaiting verify | 0 | 1 | 0 | 0 | 0 |

## Open
### SEC-011 · High · JWT accepted without expiry check
- **Where:** auth/verify.js:18 (verifyToken)
- **Category:** A07:2025 · CWE-287 · Confidence: Confirmed
- **Found by:** Claude review
- **What's wrong:** ...
- **Impact:** ...
- **How to fix:** ...
- **Owner:** Anu · **Ticket:** #121 · **First seen:** R1
- **Secret:** (secrets only) type, masked value, rotated yes/no
- **Compliance:** (if any) flags to check

## Fixed, awaiting verification
## Accepted risks (with expiry dates)
## False positives
## Verified (closed)
SEC-004 · High · SQL injection in patient search · verified R3

## Triage dismissals (for human review)
```

Sort by severity, then ID. Keep Verified entries to one line each.

## runs/R<n>.md

Each run summary records:

- run ID, date, commit, type (full or quick) and coverage
- tools and versions, and any tool failures
- any scope changes since the previous run
- counts by severity and status
- **changes since the previous run:** Verified · Still open · Fix did not resolve · Reopened · New · Accepted risks expired
- the gate result, with the specific findings that block it
- out-of-scope hits and coverage gaps

## history.jsonl

One JSON object per line. Append only; never edit or delete lines.

```json
{"ts":"2026-10-06T14:12:00Z","id":"SEC-004","event":"created","run":"R1","commit":"a1b2c3d","severity":"High"}
{"ts":"2026-10-07T09:30:00Z","id":"SEC-004","event":"owner","to":"Rahul","by":"Rahul (stated)"}
{"ts":"2026-10-07T11:02:00Z","id":"SEC-004","event":"status","from":"Open","to":"Fixed","by":"Rahul (stated)","commit":"e7f8a9b","note":"Root cause: query built by string concatenation"}
{"ts":"2026-10-08T10:15:00Z","id":"SEC-004","event":"status","from":"Fixed","to":"Verified","run":"R3","commit":"e7f8a9b"}
```

Event types: `created`, `status`, `owner`, `severity` (with a reason), `fix-rejected` (a fix claim not accepted, with the reason), `ticket`, `note`.

Log an event for every status, owner, severity or ticket change, and for every rejected claim. Routine `lastSeen` updates don't need events.

## findings.sarif (optional)

Write SARIF 2.1.0 with:

- `tool.driver.name`: `"security-audit"`
- one rule per finding ID
- `level`: Critical and High → `error`, Medium → `warning`, Low and Info → `note`
- `locations` from the finding's locations
- `partialFingerprints`: `{ "securityAudit/v1": "<fingerprint>" }`

Include only findings that aren't closed. Never include secret values.
