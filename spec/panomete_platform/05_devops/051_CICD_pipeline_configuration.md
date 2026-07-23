---
document_type: CI/CD Pipeline Configuration
version: "0.1"
status: Draft
author: "DevOps Persona"
created: "2026-07-23"
last_updated: "2026-07-23"
project_name: "Panomete Platform"
project_id: "PAN-PLAT-001"
classification: "Internal"
tags: [ci-cd, pipeline, github-actions, devops, panomete]
standard_ref:
  - SWEBOK v4 — Operations
  - ISO/IEC 20000 — IT Service Management
---

# CI/CD Pipeline Configuration — Panomete Platform

> **Project:** Panomete Platform
> **Version:** 0.1 | **Status:** Draft — Awaiting PO Review
> **Last Updated:** 2026-07-23
> **Scope:** Phase 1 — Guard (Keycloak), Discover (Eureka), Gate (Spring Cloud Gateway)

---

## 1. Purpose

> Defines the CI/CD pipeline for the Panomete Platform foundation services. Every push to `main` triggers the full pipeline. Pull requests trigger lint + test. **If it's not in the pipeline, it doesn't ship.**

---

## 2. Pipeline Overview

```mermaid
flowchart LR
    PUSH[Git Push] --> LINT[Lint & Compile]
    LINT --> TEST[Unit Test]
    TEST --> BUILD[Build JAR]
    BUILD --> IMAGE[Docker Image Build]
    IMAGE --> SCAN[Security Scan]
    SCAN --> PUSH_REG[Push to GHCR]
    PUSH_REG --> DEPLOY[Deploy to Homelab]
    DEPLOY --> SMOKE[Smoke Test]
    SMOKE --> VERIFY[Health Check]

    style PUSH fill:#2196F3,color:#fff
    style LINT fill:#FF9800,color:#fff
    style TEST fill:#4CAF50,color:#fff
    style BUILD fill:#9C27B0,color:#fff
    style SCAN fill:#f44336,color:#fff
    style DEPLOY fill:#4CAF50,color:#fff
    style VERIFY fill:#4CAF50,color:#fff
```

> **Note:** The homelab has a single environment (no separate staging). "Deploy to Homelab" IS the production deploy. Observability (Phase 2) will add a staging-like verification step via smoke tests.

---

## 3. Repository Structure

> Monorepo recommended for Phase 1. Each service gets a subdirectory.

```
panomete-platform/
├── .github/
│   └── workflows/
│       ├── ci.yml                    # Lint + test (runs on PRs + main)
│       └── deploy.yml                # Build + deploy (runs on main only)
├── flowero-guard/                    # Keycloak realm config + Dockerfile
├── flowero-discover/                 # Eureka server source
├── flowero-gate/                     # Gateway source
├── docker-compose.platform.yml       # Phase 1 compose (Guard + Discover + Gate)
├── .env.example                      # Template for secrets
└── README.md
```

---

## 4. Pipeline Configuration

### 4.1 CI — Lint + Test (runs on PRs and main)

```yaml
# .github/workflows/ci.yml
name: CI

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  lint-test:
    name: Lint & Test
    runs-on: ubuntu-latest
    strategy:
      matrix:
        service: [flowero-discover, flowero-gate]  # Guard is config-only (no Java to test)
    steps:
      - uses: actions/checkout@v4

      - name: Set up JDK 25
        uses: actions/setup-java@v4
        with:
          java-version: '25'
          distribution: 'temurin'
          cache: gradle

      - name: Compile
        run: cd ${{ matrix.service }} && ./gradlew compileJava -q

      - name: Lint (Checkstyle)
        run: cd ${{ matrix.service }} && ./gradlew checkstyleMain -q

      - name: Unit Tests
        run: cd ${{ matrix.service }} && ./gradlew test -q
```

### 4.2 CD — Build + Deploy (runs on main only)

```yaml
# .github/workflows/deploy.yml
name: Build & Deploy

on:
  push:
    branches: [main]

env:
  REGISTRY: ghcr.io
  IMAGE_PREFIX: panomete

jobs:
  build:
    name: Build Docker Images
    runs-on: ubuntu-latest
    strategy:
      matrix:
        include:
          - service: flowero-discover
            dockerfile: flowero-discover/Dockerfile
          - service: flowero-gate
            dockerfile: flowero-gate/Dockerfile
          - service: flowero-guard
            dockerfile: flowero-guard/Dockerfile
    steps:
      - uses: actions/checkout@v4

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Log in to GHCR
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Build & Push
        uses: docker/build-push-action@v5
        with:
          context: ${{ matrix.service }}
          file: ${{ matrix.dockerfile }}
          push: true
          tags: |
            ${{ env.REGISTRY }}/${{ env.IMAGE_PREFIX }}/${{ matrix.service }}:latest
            ${{ env.REGISTRY }}/${{ env.IMAGE_PREFIX }}/${{ matrix.service }}:${{ github.sha }}

  deploy:
    name: Deploy to Homelab
    needs: build
    runs-on: ubuntu-latest
    environment: homelab
    steps:
      - uses: actions/checkout@v4

      - name: Deploy via SSH
        uses: appleboy/ssh-action@v1
        with:
          host: remote.panomete.com
          username: flowero
          key: ${{ secrets.HOMELAB_SSH_KEY }}
          script: |
            cd ~/platform
            # Pull latest images
            docker compose -f docker-compose.platform.yml pull

            # Rolling restart (Guard → Discover → Gate)
            docker compose -f docker-compose.platform.yml up -d flowero-guard
            sleep 10
            docker compose -f docker-compose.platform.yml up -d flowero-discover
            sleep 10
            docker compose -f docker-compose.platform.yml up -d flowero-gate

      - name: Smoke Test
        run: |
          # Guard health
          curl -sf http://localhost:8001/health/ready || exit 1
          # Discover health
          curl -sf http://localhost:8999/actuator/health || exit 1
          # Gate health
          curl -sf http://localhost:8000/actuator/health || exit 1

      - name: Rollback on Failure
        if: failure()
        uses: appleboy/ssh-action@v1
        with:
          host: remote.panomete.com
          username: flowero
          key: ${{ secrets.HOMELAB_SSH_KEY }}
          script: |
            cd ~/platform
            docker compose -f docker-compose.platform.yml rollback
```

---

## 5. Pipeline Stages

| Stage | Purpose | Duration (est.) | Failure Action |
|-------|---------|:---:|----------------|
| Compile | Java 25 compilation | < 1 min | Block merge |
| Lint | Checkstyle + SpotBugs | < 1 min | Block merge |
| Unit Test | JUnit + Mockito | < 3 min | Block merge |
| Docker Build | Build multi-arch image | < 3 min | Block deploy |
| Security Scan | Trivy vulnerability scan | < 2 min | Block on critical |
| Deploy | SSH + docker compose pull/up | < 2 min | Alert team |
| Smoke Test | Health endpoint checks | < 1 min | Trigger rollback |
| Rollback | Revert to previous image | < 1 min | Notify PO |

---

## 6. Environment Configuration

| Environment | Branch | Auto-Deploy | Approval | URL |
|-------------|--------|:---:|:---:|-----|
| Development | `develop` | No (manual) | No | Local Docker |
| Homelab (Prod) | `main` | Yes (after CI passes) | No (solo dev) | `*.panomete.com` |

> **Note:** This is a personal homelab with a single developer. No formal approval gate is required for production deploys. When the team grows, add a manual approval job (`environment: homelab` with required reviewers).

---

## 7. Secrets Management

| Secret | Location | Used By | Rotation |
|--------|----------|---------|----------|
| `HOMELAB_SSH_KEY` | GitHub Actions secrets | Deploy job | On team change |
| `POSTGRES_PASSWORD` | Homelab `.env` file | Guard (JDBC) | Quarterly |
| `KEYCLOAK_ADMIN_PASSWORD` | Homelab `.env` file | Guard (admin) | Quarterly |
| `VALKEY_PASSWORD` | Homelab `.env` file | Gate (rate limiting) | On change |
| `KEYCLOAK_CLIENT_SECRET_*` | Homelab `.env` file | Gate, business services | Per client |

> **Rule:** Secrets never in git. The `.env` file lives at `~/platform/.env` on the homelab and is never committed. GitHub Actions uses repository secrets for CI/CD.

---

## 8. Docker Registry

| Field | Value |
|-------|-------|
| Registry | GitHub Container Registry (GHCR) |
| URL | `ghcr.io/panomete/<service>` |
| Auth | `GITHUB_TOKEN` (automatic in Actions) |
| Tags | `latest` (rolling) + `<git-sha>` (immutable) |
| Cleanup | Keep last 10 images per service |

---

## 9. Keycloak Realm Import (Special Case)

> Guard (Keycloak) is configured via realm JSON, not compiled code. CI does not test Guard's Java. Instead:

1. **Realm JSON** (`flowero-guard/panomete-realm.json`) is version-controlled
2. **Guard's Dockerfile** copies the realm JSON and uses `--import-realm`
3. **CI validates JSON** with `jq` (syntax check)
4. **Deploy** pulls the Guard image — Keycloak imports the realm on first boot
5. **Realm changes** = edit JSON → commit → pipeline rebuilds Guard image

```yaml
# In ci.yml — add realm validation
  validate-realm:
    name: Validate Keycloak Realm JSON
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Validate realm JSON syntax
        run: jq . flowero-guard/panomete-realm.json > /dev/null
```

---

## 10. Quality Gates

A pull request cannot merge to `main` unless:

- [ ] CI pipeline passes (compile + lint + test)
- [ ] No new SpotBugs warnings
- [ ] Keycloak realm JSON is valid (if changed)
- [ ] Code review approved (when team grows)

A deployment to homelab cannot proceed unless:

- [ ] CI pipeline passed on `main`
- [ ] Docker images built and pushed to GHCR
- [ ] Trivy scan has zero critical vulnerabilities
- [ ] Smoke test passes after deploy (auto-rollback if not)

---

## Related Documents

| Document | Relationship |
|----------|-------------|
| [[052_deployment_plan]] | Step-by-step deployment procedure |
| [[053_release_notes]] | What shipped in each release |
| [[054_operations_manual_runbook]] | How to operate and troubleshoot |
| `panomete_platform/02_design/025_software_architecture_document` | Deployment topology (SAD §6) |
| `flowero_guard/02_design/023_database_schema_DDL` | Keycloak DB provisioning |

---

> **Template Standard:** Based on SWEBOK v4, ISO/IEC 20000
> **Usage:** The pipeline is *the single path to production*. No hotfixes bypass the pipeline.
