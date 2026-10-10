# CI security scan templates

Offer these at sign-off so scans keep running on every pull request. Adapt them to the project's CI platform and stack.

## Rules for generating CI files

- **Pin everything.**
  - GitHub Actions: pin to a full 40-character commit SHA, with the version as a comment.
  - Container images: pin by digest (`@sha256:...`), with the tag as a comment.
- **Never invent a SHA or digest.** Resolve them:
  - actions: `git ls-remote https://github.com/actions/checkout 'refs/tags/v4*'`
  - images: `docker buildx imagetools inspect <image>:<tag>`
  - or use the GitHub connector

  If you can't resolve one, leave `<PIN_ME>` and tell the user exactly which values to fill in.
- **Least privilege:** `permissions: contents: read`, unless SARIF upload is wanted, which also needs `security-events: write`.
- **Trigger on `pull_request` and pushes to the main branch.** Never use `pull_request_target` for scans.
- **Decide what blocks.** Start by failing the build on secrets and on high-severity code findings. Report dependency results without blocking until the existing dependency backlog is cleared, then switch them to blocking.
- **Test first.** Run the workflow once on a branch and fix any path or permission problems before merging.
- Don't use Trivy releases or actions from the March 2026 compromise. See `scanners.md`.

## GitHub Actions

`.github/workflows/security-scan.yml`

```yaml
name: Security scan
on:
  pull_request:
  push:
    branches: [main]
permissions:
  contents: read
jobs:
  scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@<PIN_ME> # v4.x
        with:
          fetch-depth: 0          # full history so secrets in old commits are found
          persist-credentials: false

      - name: Secrets (Gitleaks)
        run: |
          docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/repo" \
            zricethezav/gitleaks@sha256:<PIN_ME> \
            git --redact --exit-code 1 /repo

      - name: Code (Semgrep)
        run: |
          docker run --rm -v "$PWD:/src" -w /src semgrep/semgrep@sha256:<PIN_ME> \
            semgrep scan --config p/default --config p/owasp-top-ten --metrics=off \
            --severity ERROR --error

      - name: Dependencies (OSV-Scanner, report only)
        continue-on-error: true   # remove once the dependency backlog is cleared
        run: |
          docker run --rm -v "$PWD:/src" ghcr.io/google/osv-scanner@sha256:<PIN_ME> \
            scan source -r /src
```

Optional additions:

- `zizmor` for workflow files (`pipx run zizmor --offline .github/workflows`)
- `checkov -d . --skip-download` for IaC
- SARIF upload to GitHub code scanning with `github/codeql-action/upload-sarif`, pinned by SHA, which needs `security-events: write`

## GitLab CI

`.gitlab-ci.yml` (add the stage to existing pipelines)

```yaml
security_scan:
  stage: test
  image: docker:<PIN_ME>
  services: ["docker:<PIN_ME>-dind"]
  variables:
    GIT_DEPTH: 0
  script:
    - docker run --rm -v "$CI_PROJECT_DIR:/repo" zricethezav/gitleaks@sha256:<PIN_ME> git --redact --exit-code 1 /repo
    - docker run --rm -v "$CI_PROJECT_DIR:/src" -w /src semgrep/semgrep@sha256:<PIN_ME> semgrep scan --config p/default --metrics=off --severity ERROR --error
    - docker run --rm -v "$CI_PROJECT_DIR:/src" ghcr.io/google/osv-scanner@sha256:<PIN_ME> scan source -r /src || true
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH
```

Other platforms (Bitbucket, Azure DevOps, Jenkins) use the same three commands in their own pipeline syntax.
