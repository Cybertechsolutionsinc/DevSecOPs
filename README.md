# GitHub Security Scanning Workflows

A drop-in set of GitHub Actions workflows that scan **IaC**, **application code**, **secrets**, and **dependencies** using only free / open source tools. Results are published to the GitHub Security tab, attached as build artifacts, summarised on the job page, and posted as a sticky PR comment.

## What's included

| File | Purpose |
|---|---|
| `.github/workflows/security-scan.yml` | Self-contained full scan — drop into a single repo with no central dependency |
| `.github/workflows/security-scan-reusable.yml` | Reusable FULL scan (IaC + SAST + secrets + deps) — publish in central platform repo |
| `.github/workflows/call-security-scan.yml` | Example consumer that calls the full reusable workflow |
| `.github/workflows/iac-scan-reusable.yml` | Reusable IaC-only scan — lighter-weight alternative for infra-only repos |
| `.github/workflows/call-iac-scan.yml` | Example consumer for the IaC-only reusable workflow |
| `.gitleaks.toml` | Gitleaks config with sensible allowlist for tests/fixtures |

## Scanners

All five are free and open source, with no paid tier required.

- **Checkov** (Bridgecrew/Prisma) — IaC policy scanning for ARM, Bicep, Terraform, Kubernetes, Dockerfiles.
- **Trivy** (Aqua Security) — IaC config scan plus dependency and filesystem CVE scanning.
- **Semgrep** (OSS rules) — SAST across most mainstream languages with curated rulesets (`p/security-audit`, `p/owasp-top-ten`, `p/cwe-top-25`).
- **Gitleaks** — Secret detection across working tree and full git history.
- **OSV-Scanner** (Google) — Dependency vulnerability scanning against the OSV database.

## Installation

Copy the `.github/workflows/` directory and `.gitleaks.toml` into the root of your repository and commit. That's it for a single-repo setup.

## Required permissions

The workflows already declare the permissions they need:

```yaml
permissions:
  contents: read
  security-events: write   # upload SARIF to Security tab
  pull-requests: write     # post PR comments
  actions: read
```

No additional PATs or secrets are required for public repos. For private repos, SARIF upload to the Security tab requires **GitHub Advanced Security** (free for public repos, paid for private). If GHAS is not available, the SARIF files are still published as build artifacts.

## Gating behaviour

The workflow is configured to **fail the build on HIGH or CRITICAL** findings from any scanner. Logic is implemented in a small inline Python step per job that reads the SARIF output and checks the `security-severity` property (CVSS 7.0+ = HIGH, 9.0+ = CRITICAL).

To soften gating during rollout, set `soft-fail: true` on the reusable workflow, or change the severity threshold by editing the gate step.

## Running the workflow

Triggers configured:

- Every **pull request** to `main`
- Every **push** to `main`
- **Manual** via `workflow_dispatch`
- **Weekly** scheduled scan (Monday 06:00 UTC) — catches newly-disclosed CVEs

## Using the reusable workflow across many repos

For an org with many product teams, publish `iac-scan-reusable.yml` in a central `platform-workflows` repo, tag it `v1`, and have every product repo call it:

```yaml
jobs:
  iac:
    uses: your-org/platform-workflows/.github/workflows/iac-scan-reusable.yml@v1
    with:
      scan-path: ./infra
      fail-severity: HIGH
```

When standards change, update the reusable workflow and bump the tag — every consuming repo picks up the new version on its next run. Combined with GitHub's **required workflows** feature at the org level, you can enforce that every repo runs this scan before merge.

## Viewing results

1. **GitHub Security tab** — Code scanning alerts grouped by category (`checkov`, `trivy-iac`, `semgrep`, `trivy-deps`, `osv`).
2. **PR comment** — Sticky comment with a findings table, updated on every push to the PR.
3. **Job summary** — Per-job markdown summary on the workflow run page.
4. **Artifacts** — SARIF files attached to the run for 30 days.

## Tuning & suppressions

- **Checkov** — add `.checkov.yaml` at repo root with `skip-check: [CKV_AZURE_1, ...]`.
- **Trivy** — add `.trivyignore` with CVE IDs to ignore, one per line.
- **Semgrep** — add `.semgrepignore` (glob syntax) to exclude files.
- **Gitleaks** — extend `.gitleaks.toml` allowlist for known false positives.
- **OSV-Scanner** — pass `--config=osv-scanner.toml` with ignored vulns.

Always add a comment explaining **why** a rule is suppressed.

## Rollout checklist

1. Start with `soft-fail: true` / warn-only for 1–2 sprints to gather a baseline.
2. Triage the backlog — fix highs, suppress (with comments) or accept the rest.
3. Flip to blocking mode on PRs (`fail-severity: HIGH`).
4. Add the workflow as a **required status check** on your branch protection rules.
5. Promote to an **org-level required workflow** once proven on 3–5 pilot repos.

## Cost

Zero. All scanners used are free for unlimited use, including commercial. The only variable is GitHub Actions minutes — a full scan on a typical repo completes in 3–6 minutes total wall-clock (jobs run in parallel).
