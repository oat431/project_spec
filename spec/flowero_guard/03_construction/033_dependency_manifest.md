---
document_type: Dependency Manifest
version: "0.1"
status: Active
author: "DevOps Persona"
created: "2026-07-23"
last_updated: "2026-07-23"
project_name: "Flowero Guard"
project_id: "PAN-GUARD-001"
parent_platform: "Panomete Platform"
classification: "Internal"
tags: [dependencies, docker, keycloak, panomete]
standard_ref:
  - SWEBOK v4 — Construction
  - OWASP Top 10 (A06:2021 — Vulnerable Dependencies)
---

# Dependency Manifest — Flowero Guard

> **Service:** Flowero Guard (Keycloak IAM)
> **Platform:** Panomete Platform
> **Version:** 0.1 | **Status:** Active
> **Last Updated:** 2026-07-23

---

## 1. Purpose

> Complete inventory of Flowero Guard dependencies. Guard is a config-only service — it runs the official Keycloak image with a realm JSON overlay. No compiled code, no build-time dependencies.

## 2. Dependency Overview

| Metric | Count |
|--------|-------|
| Docker Base Image | 1 (Keycloak) |
| Configuration Files | 1 (`panomete-realm.json`) |
| Runtime Dependencies | 1 (PostgreSQL 18) |
| Build-time Dependencies | 0 (no compilation) |
| Known Vulnerabilities | 0 (as of 2026-07-23) |

## 3. Docker Image

| Image | Version | Registry | License | Purpose |
|-------|---------|----------|---------|---------|
| `quay.io/keycloak/keycloak` | `latest` (26.7.0) | Quarkus Registry | Apache-2.0 | Keycloak IAM server |

> **Pin to a specific version in production.** `latest` is acceptable for homelab but should be replaced with a tagged version (e.g., `26.7.0`) for reproducibility.

### Image Contents

The Keycloak image includes:

| Component | Version | Purpose |
|-----------|---------|---------|
| Quarkus | 3.33.2.1 | Application framework |
| Java (OpenJDK) | 21+ | JVM runtime |
| PostgreSQL JDBC | Bundled | Database connectivity |
| Liquibase | Bundled | Schema migrations |
| Infinispan | Bundled | Distributed caching (disabled via `KC_CACHE=local`) |
| JGroups | Bundled | Cluster communication (disabled via `KC_CACHE=local`) |

## 4. Runtime Dependencies

### Required (must be running before Guard starts)

| Dependency | Version | Container | Network | Purpose |
|------------|---------|-----------|---------|---------|
| PostgreSQL | 18 | `local-postgres` | `db-network` | Keycloak schema + data storage |

### Optional (not required for Guard, but part of platform)

| Dependency | Version | Container | Purpose |
|------------|---------|-----------|---------|
| Valkey | 9 | `local-valkey` | Used by Gate, not Guard |
| Eureka | — | `flowero-discover` | Guard can register for health visibility |

## 5. Configuration Dependencies

| File | Purpose | Version Controlled |
|------|---------|:---:|
| `panomete-realm.json` | Realm configuration (roles, clients, scopes, settings) | ✅ |
| `docker-compose.platform.yml` | Container definition + environment | ✅ |
| `.env` | Secrets (DB password, admin password) | ❌ Never committed |

## 6. Keycloak Internal Schema (Managed by Liquibase)

> Keycloak manages its own database schema. These tables are created automatically on first boot. **Do NOT modify manually.**

| Table | Purpose |
|-------|---------|
| `REALM` | Realm configuration |
| `CLIENT` | OAuth2 clients |
| `USER_ENTITY` | Users |
| `USER_ROLE_MAPPING` | User-to-role assignments |
| `KEYCLOAK_ROLE` | Realm and client roles |
| `CREDENTIAL` | Hashed passwords/credentials |
| `USER_SESSION` | Active sessions |
| `EVENT_ENTITY` | Audit events |
| `MIGRATION` | Liquibase changelog |

## 7. Platform-Level Shared Dependencies

| Service | How It Depends on Guard |
|---------|-------------------------|
| **Flowero Gate** | Caches JWKS for local JWT validation |
| **Frontend SPAs** | Redirect to `auth.panomete.com` for login |
| **All business services** | Trust claim headers forwarded by Gate |

## 8. Vulnerability Status

| Scan Date | Image | Critical | High | Medium | Low |
|-----------|-------|:---:|:---:|:---:|:---:|
| 2026-07-23 | `keycloak:latest` | — | — | — | — |

> ⚠️ Image vulnerability scanning not yet configured. Recommended: add Trivy scan to CI pipeline.

## 9. Dependency Update Cadence

| Event | Action |
|-------|--------|
| Keycloak patch release | Test within 1 week, update image tag |
| Keycloak minor release | Test within 2 weeks, verify realm import compatibility |
| Keycloak major release | Full regression test, manual review |
| PostgreSQL patch release | Auto-update (shared instance) |
| Security vulnerability (Critical/High) | Patch immediately |

---

## Related Documents

| Document | Relationship |
|----------|-------------|
| [[032_build_scripts]] | Build pipeline that uses these dependencies |
| [[031_README_developer_guide]] | Dev setup instructions |
| [[023_database_schema_DDL]] | PostgreSQL provisioning for Keycloak |
| [[021_architecture_decision_records]] | Why Keycloak, why shared PostgreSQL |

---

> **Template Standard:** Based on SWEBOK v4, OWASP Top 10 (A06:2021)
> **Usage:** Guard has 1 Docker image and 1 config file. The Keycloak image bundles everything else. Pin the image version for production.
