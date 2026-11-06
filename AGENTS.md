# Security scanning kit

Reusable CI security stage and agent remediation rules. Copy into any repo
(single app, multi-app, or monorepo). Repo shape does not matter: configure
paths and languages, or turn off container scanning when there is no image.

## What is in this package

| Path | Role |
| --- | --- |
| `.github/workflows/security.yml` | GitHub Actions security job (Gitleaks → CodeQL → Trivy) |
| `.github/instructions/aikido_rules.instructions.md` | Agent rule: scan/fix/rescan with Aikido MCP |
| `templates/.gitleaksignore` | Empty allowlist template for reviewed Gitleaks fingerprints |
| `templates/call-from-deploy.yml.snippet` | How to `needs: [security]` from a deploy workflow |

## What it provides

### 1. Pipeline gate (`security.yml`)

A reusable / copy-in workflow that:

1. **Gitleaks** — scans full git history for leaked secrets (runs on every PR and on full scans; ~1 minute).
2. **CodeQL** — static analysis; fails the job on error-level findings (skipped on PRs).
3. **Trivy** — builds a container image and fails on HIGH/CRITICAL vulns with fixes available (skipped on PRs and when `scan_container: false`).

Steps are cheapest-first so a secret finding never pays for CodeQL or an image build.

**Triggers**

- `pull_request` → Gitleaks only
- `workflow_call` / `workflow_dispatch` → full scan (used to gate deploys)

**Inputs** (`workflow_call`)

| Input | Default | Purpose |
| --- | --- | --- |
| `codeql_languages` | `javascript-typescript` | CodeQL packs (comma-separated) |
| `scan_container` | `true` | Set `false` when there is no Dockerfile / image |
| `docker_context` | `.` | Build context (monorepo package dir) |
| `dockerfile` | `Dockerfile` | Dockerfile path from repo root |
| `image_ref` | `app:security-scan` | Local tag for Trivy |
| `trivy_severity` | `HIGH,CRITICAL` | Severities that fail the job |
| `gitleaks_version` | `8.30.1` | Pinned Gitleaks release |

No product name, app brand, or vendor-specific build secrets are baked in. If a Dockerfile needs build-time secrets, add `--secret` / env under the marked `CUSTOMIZE` step in `security.yml`.

### 2. Agent remediation (Aikido instructions)

Cursor/Copilot-style instruction file that tells coding agents to:

- Run `aikido_full_scan` on new/changed first-party code
- Apply Aikido remediations
- Rescan until clean
- Point the user at https://help.aikido.dev/ide-plugins/aikido-mcp if the MCP server is missing

This is the remediation half of the stage; CI fails the merge/deploy, agents fix findings in the IDE.

### 3. Gitleaks allowlist template

Reviewed false positives only. Format: `<commit>:<file>:<rule-id>:<line>`. Never use it to paper over a real secret.

## Install into a project

```bash
# From the consuming repo root:
cp /path/to/security-scanning/.github/workflows/security.yml .github/workflows/
mkdir -p .github/instructions
cp /path/to/security-scanning/.github/instructions/aikido_rules.instructions.md .github/instructions/
cp /path/to/security-scanning/templates/.gitleaksignore .gitleaksignore
```

Wire the deploy/release workflow with the snippet in `templates/call-from-deploy.yml.snippet`.

Required job permissions when calling:

```yaml
permissions:
  contents: read
  actions: read
```

## Repo-shape recipes

**Single app with Dockerfile at root** — copy files; call with defaults.

**Monorepo / multi-app** — call once per image (or matrix), set `docker_context`, `dockerfile`, `image_ref`; widen `codeql_languages` if needed.

**No container** — `scan_container: false`; Gitleaks + CodeQL still gate the pipeline.

**Non-JS** — set `codeql_languages` (e.g. `python`, `go`, `java-kotlin`).

## What this kit does not include

- App quality gates (lint, unit tests, dependency audit) — keep those in the project's own quality job
- Deploy, migrate, or host-specific steps
- Project-specific secret fingerprints — start from the empty template
- GitHub Advanced Security upload — SARIF is gated locally so it works without GHAS

## Agent expectations when working in a consuming repo

1. Treat a failing `security` job as a merge/deploy blocker; fix findings, do not weaken gates without an explicit human decision.
2. For Gitleaks hits: remove or rotate secrets; only add `.gitleaksignore` fingerprints for confirmed public/placeholder values, scoped to one commit/file/line/rule.
3. For CodeQL error-level findings: fix the code; do not disable the fail step.
4. For Trivy HIGH/CRITICAL (fixed available): upgrade base image or dependencies; prefer fixing the image over raising severity thresholds.
5. When editing first-party code and Aikido MCP is available, follow `aikido_rules.instructions.md` (scan → remediate → rescan).
