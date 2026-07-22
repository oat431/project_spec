---
document_type: ADR (Architecture Decision Records)
version: "0.1"
status: Active
author: "Dev / SA Persona"
created: "2026-07-22"
last_updated: "2026-07-22"
project_name: "Flowero Guard"
project_id: "PAN-GUARD-001"
parent_platform: "Panomete Platform"
architect: "Dev / SA Persona"
classification: "Internal"
tags: [adr, keycloak, oauth2, iam, panomete]
standard_ref:
  - SWEBOK v4 — Architecture
  - ISO/IEC/IEEE 42010 — Architecture Description
---

# Architecture Decision Records — Flowero Guard

> **Service:** Flowero Guard (Keycloak IAM)
> **Platform:** Panomete Platform
> **Version:** 0.1 | **Status:** Active
> **Last Updated:** 2026-07-22

---

## ADR Index

| ADR | Title | Status | Date | Decision |
|-----|-------|--------|------|---------|
| ADR-G001 | Realm-as-Code (JSON Export) | ✅ Accepted | 2026-07-22 | Realm configuration version-controlled as JSON |
| ADR-G002 | Single Realm (panomete) | ✅ Accepted | 2026-07-22 | One realm for all platform services |
| ADR-G003 | Confidential Clients Only | ✅ Accepted | 2026-07-22 | All OAuth2 clients are confidential (require secret) |
| ADR-G004 | Admin-Created Users (No Self-Registration) | ✅ Accepted | 2026-07-22 | MVP: admin manually creates users |
| ADR-G005 | Standard OAuth2 Flows Only | ✅ Accepted | 2026-07-22 | Authorization Code + Client Credentials; no Device/Implicit |

---

## ADR-G001: Realm-as-Code (JSON Export)

| Field | Detail |
|-------|--------|
| **Status** | ✅ Accepted |
| **Date** | 2026-07-22 |

### Context

> Keycloak configuration (realms, clients, roles, identity providers, etc.) can be configured via the Admin Console UI or imported from a JSON file. UI-based config is not reproducible, not version-controlled, and cannot be audited.

### Decision

> **Export the `panomete` realm to a JSON file** (`keycloak/panomete-realm.json`) and import it on Keycloak startup using the `--import-realm` flag. The JSON file is committed to version control. All realm changes go through the JSON file, not the Admin Console.

### Consequences

**Positive:**
- Entire IAM configuration is versioned and auditable
- Reproducible — `docker compose up` on a fresh machine recreates the exact realm
- Infrastructure-as-Code — matches the platform's documentation-first philosophy
- Disaster recovery — JSON file is the backup

**Negative:**
- JSON export is verbose (a full realm export can be thousands of lines)
- Must manage client secrets outside the JSON (use env vars or vault)
- Keycloak's export format can change between versions

### Alternatives Considered

| Alternative | Why Not |
|-------------|---------|
| Admin Console only | Not reproducible. Manual steps not documented. Fails the "Future Self" test. |
| Terraform provider | Overkill for a single realm on a homelab. Adds a tool to the stack. |
| Keycloak Config CLI | Good alternative; JSON export is simpler and built-in. |

---

## ADR-G002: Single Realm (panomete)

| Field | Detail |
|-------|--------|
| **Status** | ✅ Accepted |
| **Date** | 2026-07-22 |

### Context

> Keycloak supports multiple realms for tenant isolation. With 6+ business services planned, we need to decide: one realm for everything, or one realm per service?

### Decision

> **Use a single `panomete` realm** for all platform services. All services register as clients in the same realm. Users, roles, and SSO sessions are shared. This is a personal homelab — there is one "tenant" (Self).

### Consequences

**Positive:**
- Single sign-on across all services — login once, access everything
- Simpler administration — one realm to configure, one set of users and roles
- Shared roles (`admin`, `user`, `viewer`) apply across services

**Negative:**
- No tenant isolation (irrelevant — single user)
- All clients share the same token issuer — a compromised client secret affects the same realm

### Alternatives Considered

| Alternative | Why Not |
|-------------|---------|
| One realm per service | No SSO across services. Admin overhead. Only makes sense for multi-tenant SaaS. |

---

## ADR-G003: Confidential Clients Only

| Field | Detail |
|-------|--------|
| **Status** | ✅ Accepted |
| **Date** | 2026-07-22 |

### Context

> Keycloak clients can be "public" (no secret — used for SPAs) or "confidential" (require a client secret — used for server-side apps). The platform's business services are backend APIs — they should be confidential clients.

### Decision

> **All OAuth2 clients in the `panomete` realm are confidential** with service accounts enabled. This means every client has a client ID + secret, can use the Client Credentials grant for service-to-service calls, and has its own service account user in Keycloak.

### Consequences

**Positive:**
- Client secrets provide an additional authentication factor for S2S calls
- Service accounts enable role assignment to services (e.g., "blog-service" has role "content-reader")
- Consistent configuration for all services

**Negative:**
- Must securely manage and rotate client secrets (env vars, never in git)
- SPAs cannot directly use confidential clients — frontend goes through the Gateway

---

## ADR-G004: Admin-Created Users (No Self-Registration)

| Field | Detail |
|-------|--------|
| **Status** | ✅ Accepted |
| **Date** | 2026-07-22 |

### Context

> Should users be able to self-register for the platform, or should the admin create all user accounts?

### Decision

> **MVP: Admin creates users manually** through the Keycloak Admin Console. Self-registration is disabled. The platform is a personal homelab — there is one primary user (Self) and possibly a few family members.

### Consequences

**Positive:**
- No spam accounts
- No need for email verification or CAPTCHA
- Simpler security model

**Negative:**
- Admin burden (minimal — single-digit user count)
- No self-service for future users

---

## ADR-G005: Standard OAuth2 Flows Only

| Field | Detail |
|-------|--------|
| **Status** | ✅ Accepted |
| **Date** | 2026-07-22 |

### Context

> OAuth2 defines multiple grant types. We need to support browser-based login (Authorization Code flow with PKCE) and machine-to-machine calls (Client Credentials flow). Other flows (Implicit, Device, Resource Owner Password) are deprecated or irrelevant.

### Decision

> **Enable only two flows:**
> 1. **Authorization Code flow (with PKCE)** — for browser-based login (SSO)
> 2. **Client Credentials flow** — for service-to-service authentication
>
> Disable: Implicit grant, Resource Owner Password grant, Device Authorization grant.

### Consequences

**Positive:**
- Security best practice — only modern, recommended OAuth2 flows
- PKCE prevents authorization code interception attacks
- Reduced attack surface

**Negative:**
- No native mobile app support (Device flow disabled) — irrelevant for homelab
- No legacy auth support (Resource Owner Password) — irrelevant

---

## Related Documents

| Document | Relationship |
|----------|-------------|
| [[flowero_guard/022_API_specification]] | Keycloak OAuth2/OIDC endpoints |
| [[flowero_guard/023_database_schema_DDL]] | PostgreSQL schema for Keycloak |
| [[flowero_guard/024_ERD]] | Logical realm data model |
| [[panomete_platform/021_architecture_decision_records]] | Platform-level ADRs |
| [[flowero_guard/012_user_stories]] | Stories driving these decisions |
