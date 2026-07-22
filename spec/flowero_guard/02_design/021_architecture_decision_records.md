---
document_type: ADR (Architecture Decision Records)
version: "0.2"
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
> **Version:** 0.2 | **Status:** Active — Updated per Design Review 2026-07-22
> **Last Updated:** 2026-07-22

---

## ADR Index

| ADR | Title | Status | Date | Decision |
|-----|-------|--------|------|---------|
| ADR-G001 | Realm-as-Code (JSON Export) | ✅ Accepted | 2026-07-22 | Realm config version-controlled as JSON |
| ADR-G002 | Single Realm (panomete) | ✅ Accepted | 2026-07-22 | One realm for all platform services |
| ADR-G003 | Confidential Clients Only | ✅ Accepted | 2026-07-22 | All OAuth2 clients require a secret |
| ADR-G004 | Admin-Created Users | ✅ Accepted | 2026-07-22 | MVP: admin creates users manually, no self-registration |
| ADR-G005 | Standard OAuth2 Flows Only | ✅ Accepted | 2026-07-22 | Authorization Code + Client Credentials only |
| ADR-G006 | Shared PostgreSQL 18 (No Dedicated Container) | ✅ Accepted | 2026-07-22 | Use existing shared PostgreSQL instance |
| ADR-G007 | Guard IS Keycloak (No Wrapper) | ✅ Accepted | 2026-07-22 | Deploy Keycloak directly. Gate validates JWT locally. |

---

## ADR-G006: Shared PostgreSQL 18

| Field | Detail |
|-------|--------|
| **Status** | ✅ Accepted |
| **Date** | 2026-07-22 |

### Context

> Keycloak needs a database. A dedicated guard-db container was initially planned, but PostgreSQL 18 is already running in production hosting other databases.

### Decision

> **Use the existing shared PostgreSQL 18 instance.** Create a `keycloak` database on it. Connect Keycloak via JDBC: `jdbc:postgresql://host:5432/keycloak`. No new container. No additional resource usage.

### Consequences

**Positive:** No new container. One PG instance to manage. Consistent with the homelab's DRY infrastructure pattern.

**Negative:** Keycloak's Liquibase migrations run against a shared instance. Must ensure Keycloak's schema doesn't conflict with other databases.

---

## ADR-G007: Guard IS Keycloak (No Wrapper)

| Field | Detail |
|-------|--------|
| **Status** | ✅ Accepted |
| **Date** | 2026-07-22 |

### Context

> Initial requirements described Guard as a "Spring Boot OAuth2 Resource Server." During design review, it was clarified: Guard IS the Keycloak instance. There is no separate Spring Boot service wrapping or proxying Keycloak.

### Decision

> **Deploy Keycloak directly as Flowero Guard.** Gate handles JWT validation locally (caches Keycloak's JWKS on startup — no per-request call to Guard). Guard's sole job: issue tokens, manage users, handle OAuth2/OIDC flows.

### Consequences

**Positive:** One less service to build and maintain. Clean separation: Guard = identity provider, Gate = auth enforcement point.

**Negative:** No middleware layer for custom auth logic. All custom logic must be in Keycloak extensions (SPIs) or at Gate.

---

*ADR-G001 through ADR-G005 unchanged from v0.1.*

---

## Related Documents

| Document | Relationship |
|----------|-------------|
| [[flowero_guard/022_API_specification]] | Keycloak OAuth2/OIDC endpoints |
| [[flowero_guard/023_database_schema_DDL]] | PostgreSQL provisioning for Keycloak |
| [[flowero_guard/024_ERD]] | Logical realm data model |
| [[panomete_platform/021_architecture_decision_records]] | Platform-level ADRs |
