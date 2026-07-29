# springboot-cicd-template

> **Enterprise reusable CI/CD pipeline for Spring Boot applications.**  
> This is the **template repository** in a two-repo CI/CD architecture.  
> The companion app repo (`springboot-api`) triggers this pipeline; all 12 stages run here.

---

## Two-Repo Architecture

```
┌─────────────────────────────┐        ┌──────────────────────────────────────────┐
│   springboot-api (app repo) │        │  springboot-cicd-template (this repo)    │
│                             │        │                                          │
│  Source code, IaC, Ansible  │        │  enterprise-pipeline.yml                 │
│                             │  calls │                                          │
│  deploy.yml ──────────────────────►  │  Stage 1:  Code Quality                 │
│  (gh api workflow dispatch) │        │  Stage 2:  Unit Tests                    │
│                             │        │  Stage 3:  Integration Tests             │
└─────────────────────────────┘        │  Stage 4:  Security Scan (OWASP)         │
                                       │  Stage 5:  Build JAR                     │
                                       │  Stage 6:  Docker Build & Push (GHCR)    │
                                       │  Stage 7:  Terraform Provision (EC2)     │
                                       │  Stage 8:  Ansible Configure             │
                                       │  Stage 9:  Deployment Verification       │
                                       │  Stage 10: Smoke Tests                   │
                                       │  Stage 11: Performance Tests (k6)        │
                                       │  Stage 12: Notify + Auto-Rollback        │
                                       └──────────────────────────────────────────┘
```

The template pipeline **checks out the app repo** at the dispatched SHA, runs all quality gates, builds and pushes the Docker image, provisions EC2 with Terraform, configures it with Ansible, and verifies the live deployment.

---

## How to Use This Template

### Method 1: workflow_call (GitHub-native reusable workflow)

Add this to your app repo's `.github/workflows/deploy.yml`:

```yaml
name: Deploy
on:
  push:
    branches: [main]
  workflow_dispatch: {}

jobs:
  enterprise-pipeline:
    uses: YOUR_ORG/springboot-cicd-template/.github/workflows/enterprise-pipeline.yml@main
    with:
      java_version: "17"
      image_name: "YOUR_ORG/YOUR_APP"
      aws_region: "us-east-1"
      instance_type: "t3.small"
      health_endpoint: "/actuator/health"
      run_perf_tests: true
      owasp_cvss_threshold: "9"
    secrets: inherit
```

### Method 2: workflow_dispatch via GitHub API (used by springboot-api)

```bash
gh api \
  --method POST \
  -H "Accept: application/vnd.github+json" \
  "/repos/YOUR_ORG/springboot-cicd-template/actions/workflows/enterprise-pipeline.yml/dispatches" \
  -f ref=main \
  -f "inputs[app_repo]=YOUR_ORG/YOUR_APP" \
  -f "inputs[app_ref]=<commit-sha>" \
  -f "inputs[image_name]=YOUR_ORG/YOUR_APP" \
  -f "inputs[aws_region]=us-east-1" \
  -f "inputs[instance_type]=t3.small" \
  -f "inputs[health_endpoint]=/actuator/health" \
  -f "inputs[run_perf_tests]=true" \
  -f "inputs[owasp_cvss_threshold]=9"
```

---

## Pipeline Stages

| # | Stage | Tool | Gate |
|---|-------|------|------|
| 1 | Code Quality | Checkstyle + SpotBugs | Blocks on violations |
| 2 | Unit Tests | Maven Surefire + JaCoCo | Blocks on failure |
| 3 | Integration Tests | Maven Failsafe | Blocks on failure |
| 4 | Security Scan | OWASP Dependency Check | Blocks on CVSS ≥ threshold |
| 5 | Build JAR | Maven package | Blocks on compile error |
| 6 | Docker Build & Push | Docker Buildx → GHCR | Blocks on build error |
| 7 | Terraform Provision | Terraform (AWS EC2) | Blocks on plan/apply error |
| 8 | Ansible Configure | Ansible (Docker + nginx) | Blocks on task failure |
| 9 | Deployment Verify | curl health check | Blocks if app not UP |
| 10 | Smoke Tests | curl API assertions | Blocks on endpoint failure |
| 11 | Performance Tests | k6 (50 VUs, 180s) | Blocks on p95 > 500ms |
| 12 | Notify + Rollback | GitHub Step Summary | Auto-rollback on failure |

---

## Required Secrets

These secrets must be available in the calling repository (passed via `secrets: inherit` or explicitly):

| Secret | Description |
|--------|-------------|
| `AWS_ACCESS_KEY_ID` | AWS IAM access key |
| `AWS_SECRET_ACCESS_KEY` | AWS IAM secret key |
| `TF_STATE_BUCKET` | S3 bucket for Terraform state |
| `PROJECT_NAME` | Platform project name (used as TF resource prefix) |
| `SSH_PRIVATE_KEY` | EC2 SSH private key (RSA PEM) |
| `SSH_PUBLIC_KEY` | EC2 SSH public key |
| `SSH_USER` | EC2 SSH user (e.g. `ubuntu`) |

---

## Inputs Reference

| Input | Type | Default | Description |
|-------|------|---------|-------------|
| `app_repo` | string | `""` | Source repo to check out (`owner/repo`) |
| `app_ref` | string | `""` | Git SHA or branch to deploy |
| `java_version` | string | `"17"` | Java version for build |
| `maven_args` | string | `"--no-transfer-progress"` | Extra Maven arguments |
| `image_name` | string | **required** | Docker image name (`owner/app`) |
| `aws_region` | string | `"us-east-1"` | AWS region |
| `instance_type` | string | `"t3.small"` | EC2 instance type |
| `health_endpoint` | string | `"/actuator/health"` | App health check path |
| `run_perf_tests` | boolean | `true` | Toggle k6 performance stage |
| `owasp_cvss_threshold` | string | `"9"` | OWASP fail-build CVSS threshold |

---

## Repository Structure

```
springboot-cicd-template/
└── .github/
    └── workflows/
        └── enterprise-pipeline.yml   ← The reusable workflow (all 12 stages)
README.md                             ← This file
```

The template repository is intentionally minimal — it contains only the reusable workflow definition. Application code, IaC, Ansible playbooks, and tests live in the app repository and are checked out by this pipeline at the specified `app_ref`.
