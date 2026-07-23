---
document_type: CI/CD Pipeline Configuration
version: "0.1"
status: Draft
author: "DevOps Persona"
created: "2026-07-23"
last_updated: "2026-07-23"
project_name: "Flowero Guard"
project_id: "PAN-GUARD-001"
parent_platform: "Panomete Platform"
classification: "Internal"
tags: [ci-cd, pipeline, github-actions, keycloak, devops, panomete]
standard_ref:
  - SWEBOK v4 — Operations
---

# CI/CD Pipeline Configuration — Flowero Guard

> **Service:** Flowero Guard (Keycloak IAM)
> **Platform:** Panomete Platform
> **Version:** 0.1 | **Status:** Draft — Awaiting PO Review
> **Last Updated:** 2026-07-23

---

## 1. Purpose

> Defines the CI/CD pipeline specific to Flowero Guard. Guard is unique among the platform services — it deploys **Keycloak directly** (no compiled application code). The pipeline focuses on realm JSON validation, Docker image build, and deployment to the homelab.

---

## 2. What Makes Guard Different

| Aspect | Guard (Keycloak) | Discover / Gate (Java) |
|--------|:----------------:|:---------------------:|
| Build artifact | Realm JSON + Docker image | Compiled JAR + Docker image |
| Source code | Config-only (realm JSON) | Java/Spring Boot |
| CI tests | JSON validation | JUnit + Checkstyle + SpotBugs |
| Deployment trigger | Realm JSON change | Code change |
| First-boot behavior | Liquibase migrations run automatically | Spring Boot starts |

> Guard is configured via realm JSON (`panomete-realm.json`). Keycloak imports it on startup via `--import-realm`. There is no Java code to compile or unit-test.

---

## 3. Repository Structure (Guard-Specific)

```
flowero-guard/
├── panomete-realm.json          # Realm configuration (version-controlled)
├── Dockerfile                   # Custom Keycloak image with realm pre-loaded
├── docker-compose.guard.yml     # Standalone compose for local testing
└── README.md
```

---

## 4. Pipeline Configuration

### 4.1 CI — Validate Realm JSON (runs on PRs and main)

```yaml
# .github/workflows/guard-ci.yml
name: Guard CI

on:
  push:
    branches: [main, develop]
    paths: ['flowero-guard/**']
  pull_request:
    branches: [main]
    paths: ['flowero-guard/**']

jobs:
  validate-realm:
    name: Validate Keycloak Realm
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Validate JSON syntax
        run: jq . flowero-guard/panomete-realm.json > /dev/null

      - name: Validate required realm fields
        run: |
          REALM=$(jq -r '.realm' flowero-guard/panomete-realm.json)
          if [ "$REALM" != "panomete" ]; then
            echo "❌ Realm name must be 'panomete', got '$REALM'"
            exit 1
          fi
          ENABLED=$(jq -r '.enabled' flowero-guard/panomete-realm.json)
          if [ "$ENABLED" != "true" ]; then
            echo "❌ Realm must be enabled"
            exit 1
          fi
          echo "✅ Realm JSON valid"

      - name: Check for required roles
        run: |
          ROLES=$(jq -r '.roles.realm[].name' flowero-guard/panomete-realm.json | sort | tr '\n' ',')
          echo "Roles found: $ROLES"
          for REQUIRED in admin user viewer; do
            if ! echo "$ROLES" | grep -q "$REQUIRED"; then
              echo "❌ Missing required role: $REQUIRED"
              exit 1
            fi
          done
          echo "✅ All required roles present"

      - name: Security check — no secrets in realm JSON
        run: |
          # Ensure no plaintext passwords are committed
          if grep -qiE '(password|secret|credential).{0,5}:.{0,5}["\x27][^"\x27]{6,}' flowero-guard/panomete-realm.json; then
            echo "❌ Potential plaintext secret found in realm JSON"
            exit 1
          fi
          echo "✅ No plaintext secrets detected"
```

### 4.2 CD — Build & Deploy Guard

```yaml
# .github/workflows/guard-deploy.yml
name: Guard Build & Deploy

on:
  push:
    branches: [main]
    paths: ['flowero-guard/**']

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: panomete/flowero-guard

jobs:
  build:
    name: Build Guard Image
    runs-on: ubuntu-latest
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
          context: flowero-guard
          file: flowero-guard/Dockerfile
          push: true
          tags: |
            ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:latest
            ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }}

  deploy:
    name: Deploy Guard to Homelab
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
            docker compose -f docker-compose.platform.yml pull flowero-guard
            docker compose -f docker-compose.platform.yml up -d flowero-guard
            sleep 15

      - name: Smoke Test
        run: |
          # Guard health check
          curl -sf http://remote.panomete.com:8001/health/ready || exit 1
          # OIDC discovery endpoint
          curl -sf https://auth.panomete.com/realms/panomete/.well-known/openid-configuration | jq .issuer || exit 1
          # JWKS endpoint
          curl -sf https://auth.panomete.com/realms/panomete/protocol/openid-connect/certs | jq .keys[0].kty || exit 1

      - name: Rollback on Failure
        if: failure()
        uses: appleboy/ssh-action@v1
        with:
          host: remote.panomete.com
          username: flowero
          key: ${{ secrets.HOMELAB_SSH_KEY }}
          script: |
            cd ~/platform
            docker compose -f docker-compose.platform.yml stop flowero-guard
            # Revert to previous image
            docker compose -f docker-compose.platform.yml pull flowero-guard
            docker compose -f docker-compose.platform.yml up -d flowero-guard
```

---

## 5. Pipeline Stages

| Stage | Purpose | Duration (est.) | Failure Action |
|-------|---------|:---:|----------------|
| JSON Syntax Check | Validate `panomete-realm.json` is valid JSON | < 10 sec | Block merge |
| Realm Field Validation | Verify realm name, enabled flag | < 10 sec | Block merge |
| Role Validation | Ensure `admin`, `user`, `viewer` roles exist | < 10 sec | Block merge |
| Secret Scan | No plaintext passwords in realm JSON | < 10 sec | Block merge |
| Docker Build | Build Keycloak image with realm pre-loaded | < 3 min | Block deploy |
| Deploy | SSH + docker compose pull/up | < 2 min | Alert |
| Smoke Test | Health + OIDC + JWKS endpoints | < 30 sec | Auto-rollback |

---

## 6. Guard Dockerfile

```dockerfile
# flowero-guard/Dockerfile
FROM quay.io/keycloak/keycloak:latest

# Copy realm configuration for import on startup
COPY panomete-realm.json /opt/keycloak/data/import/panomete-realm.json

# Keycloak will use --import-realm flag (set in compose command)
```

---

## 7. Secrets Management

| Secret | Location | Purpose | Rotation |
|--------|----------|---------|----------|
| `KEYCLOAK_ADMIN_PASSWORD` | Homelab `~/platform/.env` | Admin console access | Quarterly |
| `KC_DB_PASSWORD` | Homelab `~/platform/.env` | PostgreSQL `keycloak` DB access | Quarterly |
| `POSTGRES_PASSWORD` | Homelab `database/postgres/.env` | PostgreSQL superuser | On change |
| `HOMELAB_SSH_KEY` | GitHub Actions secrets | Deploy job SSH access | On team change |

> **Rule:** Client secrets for OAuth2 clients (e.g., Gate's client) are generated **after** deployment via the Admin Console. They are stored in `~/platform/.env` and never committed.

---

## 8. Quality Gates

A pull request changing `flowero-guard/**` cannot merge unless:

- [ ] Realm JSON is valid (syntax + required fields)
- [ ] Required roles (`admin`, `user`, `viewer`) are present
- [ ] No plaintext secrets in realm JSON
- [ ] Docker image builds successfully
- [ ] Code review approved (when team grows)

A deployment of Guard cannot proceed unless:

- [ ] CI pipeline passed on `main`
- [ ] `keycloak` database + role exist on PostgreSQL (pre-deployment)
- [ ] PostgreSQL is healthy
- [ ] Smoke test passes (health + OIDC + JWKS)
- [ ] Rollback plan is verified

---

## 9. Local Testing

> Developers can test realm changes locally before pushing.

```bash
# Build and run Guard locally
cd flowero-guard
docker build -t flowero-guard:local .
docker run -d \
  --name flowero-guard-local \
  -p 8080:8080 \
  -e KEYCLOAK_ADMIN=admin \
  -e KEYCLOAK_ADMIN_PASSWORD=admin \
  -e KC_DB=dev-file \
  flowero-guard:local \
  start --import-realm

# Verify realm was imported
curl http://localhost:8080/realms/panomete/.well-known/openid-configuration | jq .issuer
```

---

## Related Documents

| Document | Relationship |
|----------|-------------|
| `panomete_platform/05_devops/051_CICD_pipeline_configuration` | Platform-level pipeline (all services) |
| [[052_deployment_plan]] | Guard-specific deployment procedure |
| `flowero_guard/02_design/023_database_schema_DDL` | Keycloak DB provisioning |
| `flowero_guard/02_design/022_API_specification` | Endpoints served by Guard |
| `panomete_platform/05_devops/054_operations_manual_runbook` | Platform runbook (includes Guard incidents) |

---

> **Template Standard:** Based on SWEBOK v4
> **Usage:** Guard's pipeline validates realm config, not compiled code. The pipeline is *the single path to production*.
