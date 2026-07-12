---
document_type: ADR (Architecture Decision Records)
version: "1.0"
status: Active
author: "OraMesLita (AI)"
created: "2026-07-01"
last_updated: "2026-07-04"
project_name: "tiny mchwa"
project_id: "tiny-mchwa"
architect: "Panomete"
classification: "Internal"
tags: [adr, architecture-decisions, homelab]
standard_ref:
  - SWEBOK v4 — Architecture
  - ISO/IEC/IEEE 42010 — Architecture Description
---

# Architecture Decision Records

> **Project:** tiny mchwa 🐜
> **Version:** 1.0 | **Status:** Active
> **Last Updated:** 2026-07-04

---

## ADR Index

| ADR | Title | Status | Date | Decision |
|-----|-------|--------|------|---------|
| ADR-001 | REST API Only | ✅ Accepted | 2026-07-01 | Use REST, drop gRPC |
| ADR-002 | Computed Todolist Status | ✅ Accepted | 2026-07-01 | Status derived from tasks, not stored |
| ADR-003 | Go + Fiber v3 | ✅ Accepted | 2026-07-01 | Backend tech stack |
| ADR-004 | PostgreSQL | ✅ Accepted | 2026-07-01 | Primary database |
| ADR-005 | Keycloak OAuth | ✅ Accepted | 2026-07-01 | Authentication via homelab Keycloak |
| ADR-006 | Per-User Data Isolation | ✅ Accepted | 2026-07-01 | Data locked to Keycloak UUID |

---

## ADR-001: REST API Only

| Field | Detail |
|-------|--------|
| **Status** | ✅ Accepted |
| **Date** | 2026-07-01 |

### Context

Originally planned dual API: REST for main platform, gRPC for inter-service communication. Todo lists are small data — gRPC overhead not justified.

### Decision

Use REST only for all API communication. One protocol, one thing to maintain.

### Consequences

**Positive:**
- Simpler implementation and debugging
- One set of tooling (curl, Postman, OpenAPI)
- Lower learning curve

**Negative:**
- Slightly more verbose than gRPC for inter-service calls
- No automatic code generation (unlike gRPC protobuf)

### Alternatives Considered

| Alternative | Why Not |
|-----------|---------|
| gRPC only | Frontend can't easily consume gRPC |
| REST + gRPC | Two protocols to maintain, test, debug |

---

## ADR-002: Computed Todolist Status

| Field | Detail |
|-------|--------|
| **Status** | ✅ Accepted |
| **Date** | 2026-07-01 |

### Context

Todolist status could be stored in DB and updated on every task change, or computed on read from task states.

### Decision

Todolist status is computed on read — not stored in database. Derived from child tasks.

### Consequences

**Positive:**
- No sync issues between todolist status and task states
- Simpler data model (no status column on todolists)
- Always consistent

**Negative:**
- Slightly more complex read query (JOIN + CASE)
- Can't index todolist status for filtering (must join tasks)

### Alternatives Considered

| Alternative | Why Not |
|-----------|---------|
| Stored status with triggers | Risk of sync issues if trigger fails |
| Stored status with app logic | Risk of sync issues if update misses |

---

## ADR-003: Go + Fiber v3

| Field | Detail |
|-------|--------|
| **Status** | ✅ Accepted |
| **Date** | 2026-07-01 |

### Context

Need a performant backend framework for REST API. Team has Go experience.

### Decision

Use Go 1.22+ with Fiber v3 framework.

### Consequences

**Positive:**
- Fast, low memory footprint
- Express-like API (familiar)
- Built-in middleware (CORS, helmet, logger)

**Negative:**
- Smaller ecosystem than Node.js

---

## ADR-004: PostgreSQL

| Field | Detail |
|-------|--------|
| **Status** | ✅ Accepted |
| **Date** | 2026-07-01 |

### Context

Need ACID-compliant database for task management. Homelab already runs PostgreSQL.

### Decision

Use PostgreSQL as primary database.

### Consequences

**Positive:**
- ACID compliance
- UUID support native
- Team familiarity

---

## ADR-005: Keycloak OAuth

| Field | Detail |
|-------|--------|
| **Status** | ✅ Accepted |
| **Date** | 2026-07-01 |

### Context

Homelab uses Keycloak for centralized auth. All services should use it.

### Decision

Use Keycloak JWT tokens for authentication. All requests validated against Keycloak.

### Consequences

**Positive:**
- Centralized user management
- Single sign-on across homelab
- Industry standard OAuth2/OIDC

---

## ADR-006: Per-User Data Isolation

| Field | Detail |
|-------|--------|
| **Status** | ✅ Accepted |
| **Date** | 2026-07-01 |

### Context

Multiple users may use the service. Data must be isolated.

### Decision

All data locked to Keycloak UUID (`owned_by`). No sharing between users.

### Consequences

**Positive:**
- Simple security model
- No complex permission system needed

**Negative:**
- No collaboration features (future consideration)

---

## Related Documents

| Document | Relationship |
|----------|-------------|
| [[025_software_architecture_document]] | Architecture these decisions define |
| [[011_business_objective]] | Objectives driving decisions |
