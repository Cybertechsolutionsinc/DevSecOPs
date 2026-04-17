# GitHub Security Scanning Workflows

A drop-in set of GitHub Actions workflows that scan **IaC**, **application code**, **secrets**, and **dependencies** using only free / open source tools. Each scanner now has its own reusable workflow, plus an orchestrator that runs the whole suite.

## File layout

```
.github/workflows/
├── checkov-reusable.yml         # Checkov — IaC (ARM, Bicep, TF, K8s, Dockerfile)
├── trivy-reusable.yml           # Trivy — config (IaC) OR fs (deps/secrets/misconfig)
├── semgrep-reusable.yml         # Semgrep — SAST
├── gitleaks-reusable.yml        # Gitleaks — secret detection in code + git history
├── osv-scanner-reusable.yml     # OSV-Scanner — dependency CVEs
├── security-scan-reusable.yml   # Orchestrator — composes all of the above
│
├── security-scan.yml            # Self-contained full scan (single-repo, no central dep)
├── call-security-scan.yml       # Example consumer of the orchestrator
├── iac-scan-reusable.yml        # Legacy combined IaC (Checkov + Trivy) — still supported
└── call-iac-scan.yml            # Example consumer for IaC-only reusable
.gitleaks.toml                   # Gitleaks allowlist for tests/fixtures
```

## How to use

### Single repo, no central dependency
Copy `security-scan.yml` and `.gitleaks.toml` into the repo. One file, five scanners, works immediately.

### Org-wide (recommended)

Host the `*-reusable.yml` files and the orchestrator in a central `platform-workflows` repo, tag `v1`, and have every product repo call the orchestrator with three lines:

```yaml
jobs:
  scan:
    uses: your-org/platform-workflows/.github/workflows/security-scan-reusable.yml@v1
    permissions:
      contents: read
      security-events: write
      pull-requests: write
    with:
      fail-severity: HIGH
```

### Pick individual scanners
Teams that want finer control — for example, a Terraform-only repo that doesn't need SAST — can call the per-scanner workflows directly:

```yaml
jobs:
  checkov:
    uses: your-org/platform-workflows/.github/workflows/checkov-reusable.yml@v1
    permissions: { contents: read, security-events: write }
    with: { frameworks: terraform, fail-severity: HIGH }

  gitleaks:
    uses: your-org/platform-workflows/.github/workflows/gitleaks-reusable.yml@v1
    permissions: { contents: read }
```

## Scanners

| Workflow | Tool | What it finds |
|---|---|---|
| `checkov-reusable.yml` | Checkov | IaC misconfigs (ARM, Bicep, Terraform, K8s, Dockerfile) |
| `trivy-reusable.yml` | Trivy | IaC config (`scan-type: config`) or filesystem vulns, secrets, misconfigs (`scan-type: fs`) |
| `semgrep-reusable.yml` | Semgrep | SAST — OWASP Top 10, CWE Top 25, secrets, security-audit |
| `gitleaks-reusable.yml` | Gitleaks | Hardcoded credentials in working tree + full git history |
| `osv-scanner-reusable.yml` | OSV-Scanner (Google) | Dependency CVEs via OSV database |

All are free / open source with no paid tier.

## Common inputs

Most per-scanner workflows accept the same core inputs so callers don't have to relearn each one:

| Input | Default | Purpose |
|---|---|---|
| `scan-path` | `.` | Path to scan relative to repo root |
| `fail-severity` | `HIGH` | `LOW` / `MEDIUM` / `HIGH` / `CRITICAL` — gating threshold |
| `soft-fail` | `false` | Report but never fail the build — use during rollout |
| `upload-sarif` | `true` | Upload SARIF to GitHub code scanning |
| `artifact-retention-days` | `30` | How long to keep SARIF artifacts |

## Required permissions on the caller

```yaml
permissions:
  contents: read
  security-events: write   # SARIF upload to Security tab
  pull-requests: write     # PR comment (orchestrator only)
```

## Versioning

Publish the reusable workflows in your `platform-workflows` repo and tag:

```bash
git tag -a v1.0.0 -m "v1.0.0 initial release"
git tag -a v1 -m "v1 latest stable"
git push origin v1.0.0 v1
```

Consumers pin to `@v1` and get bug-fix updates; pin to `@v1.0.0` for fully immutable builds.

## Tuning

- **Checkov** — add `.checkov.yaml` at repo root with `skip-check: [CKV_AZURE_1, ...]`
- **Trivy** — add `.trivyignore` with CVE IDs to skip
- **Semgrep** — add `.semgrepignore` for file globs
- **Gitleaks** — extend `.gitleaks.toml` allowlist (included)
- **OSV-Scanner** — add `osv-scanner.toml` for vulnerability exclusions

Always document *why* a rule is suppressed in a nearby comment.

## Rollout checklist

1. Publish `platform-workflows` repo with all `*-reusable.yml` files, tag `v1`
2. Pilot on 3–5 volunteer repos in `soft-fail: true` mode for one sprint
3. Triage the baseline — fix critical, suppress acceptable, document decisions
4. Flip `soft-fail: false` + add the scan as a required status check on `main`
5. Bulk-add the caller workflow to remaining repos via PR bot script
6. (Optional, requires GHEC) register the orchestrator as an org-level **required workflow**
7. Track compliance in a central dashboard; review weekly

## Cost

Zero. All scanners are free and open source. The only variable is GitHub Actions minutes — on a typical repo, the parallelised orchestrator finishes in ~3–6 minutes wall-clock.
