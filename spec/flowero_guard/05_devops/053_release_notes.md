---
document_type: Release Notes
version: "0.1"
status: Placeholder
author: "DevOps Persona"
created: "2026-07-23"
last_updated: "2026-07-23"
project_name: "Flowero Guard"
project_id: "PAN-GUARD-001"
parent_platform: "Panomete Platform"
classification: "Internal"
tags: [release-notes, changelog, keycloak, panomete]
standard_ref:
  - SWEBOK v4 — Operations
---

# Release Notes — Flowero Guard

> **Service:** Flowero Guard (Keycloak IAM)
> **Platform:** Panomete Platform
> **Version:** 0.1 | **Status:** Placeholder — No releases yet. Will be populated when Sprint 1 ships.
> **Last Updated:** 2026-07-23

---

## 1. Purpose

> Documents what shipped in each Guard release — realm changes, client registrations, role updates, and configuration changes.

---

## 2. Release Template

### vX.Y.Z — [YYYY-MM-DD]

**Release Type:** [Major / Minor / Patch]
**Sprint:** [Sprint #]
**Image Tag:** `ghcr.io/panomete/flowero-guard:<tag>`

#### 🔐 Realm Changes

| # | Change | Description | Related Story |
|---|--------|-------------|:---:|
| 1 | [realm config] | [description] | US-XXX |

#### 🆕 New OAuth2 Clients

| # | Client ID | Type | Flows | Used By |
|---|-----------|------|-------|---------|
| 1 | [client-id] | [confidential/public] | [flows] | [service] |

#### 👥 Role Changes

| # | Role | Change | Impact |
|---|------|--------|--------|

#### ⚠️ Known Issues

| # | Issue | Workaround | Fix Planned |
|---|-------|-----------|------------|

#### 🔄 Migration Notes

| # | Action Required | Who | When |
|---|----------------|-----|------|

---

## 3. Expected Releases (Guard Roadmap)

### v0.1.0 — Sprint 1: Keycloak Container + Realm

| Story | Description |
|-------|-------------|
| US-001 | Keycloak container with Panomete realm, shared PostgreSQL, realm-as-code JSON |

**Expected deliverables:**
- Keycloak deployed at `auth.panomete.com`
- `panomete` realm with roles: `admin`, `user`, `viewer`
- Realm JSON version-controlled
- Shared PostgreSQL 18 backend
- Admin console accessible
- OIDC discovery + JWKS endpoints live

### v0.2.0 — Sprint 2: OAuth2 Clients + Login Flow

| Story | Description |
|-------|-------------|
| US-002 | Register OAuth2 clients for platform services |
| US-003 | Login flow + SSO across services |

**Expected deliverables:**
- OAuth2 clients registered for Gate and all business services
- Authorization Code flow working
- SSO across services
- Token endpoint issuing JWTs

### v0.3.0 — Sprint 3: RBAC + Service-to-Service Auth

| Story | Description |
|-------|-------------|
| US-004 | RBAC (realm roles → `@PreAuthorize`) |
| US-005 | Service-to-service auth (client credentials) — Should Have |
| US-006 | Token introspection for non-Java services — Should Have |

---

## 4. Release History

| Version | Date | Type | Sprint | Image Tag | Realm Changes |
|---------|------|------|--------|-----------|:---:|
| _(none yet)_ | — | — | — | — | 0 |

---

## Related Documents

| Document | Relationship |
|----------|-------------|
| [[052_deployment_plan]] | How Guard was deployed |
| [[051_CICD_pipeline_configuration]] | Guard-specific pipeline |
| `flowero_guard/02_design/022_API_specification` | Guard endpoints |
| `panomete_platform/05_devops/053_release_notes` | Platform release notes |

---

> **Template Standard:** Based on SWEBOK v4
> **Usage:** Realm changes are release-worthy. Every realm JSON commit is a release.
