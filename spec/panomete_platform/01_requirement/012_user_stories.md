---
document_type: User Stories (Platform Umbrella)
version: "0.1"
status: Draft
author: "PO (Product Owner)"
created: "2026-07-22"
last_updated: "2026-07-22"
project_name: "Panomete Platform"
project_id: "PAN-PLAT-001"
ba_owner: "PO (Product Owner)"
po_owner: "PO (Product Owner)"
classification: "Internal"
tags: [user-stories, umbrella, platform, microservices, panomete]
standard_ref:
  - SWEBOK v4 — Requirements
  - ISO/IEC/IEEE 29148 — Requirements Engineering
---

# User Stories — Panomete Platform (Umbrella)

> **Platform:** Panomete Platform
> **Version:** 0.1 | **Status:** Draft
> **Last Updated:** 2026-07-22
>
> ⚠️ **This is an umbrella document.** Detailed user stories live in each service's own directory. This document provides the consolidated view and cross-service traceability.

---

## Document Control

| Field | Value |
|-------|-------|
| Document Owner | PO (Product Owner) |
| Business Analyst | PO (Product Owner) |
| Product Owner | PO (Product Owner) |

### Revision History

| Version | Date | Author | Change Description |
|---------|------|--------|--------------------|
| 0.1 | 2026-07-22 | PO | Initial umbrella — split into service-level docs |

---

## 1. Platform Story Map

> Phase 1 Foundation consists of 3 services across 3 milestones.

| | Milestone 1: Core Infra | Milestone 2: End-to-End Auth | Milestone 3: Production Hardening |
|--|------------------------|---------------------------|--------------------------------|
| **Guard** | US-001 Deploy Keycloak, US-002 Register Clients | US-003 Login & SSO | US-004 RBAC, US-005 S2S Auth, US-006 Introspection |
| **Discover** | US-101 Deploy Eureka | US-102 Auto-Register, US-103 Discovery | US-104 Dashboard via Gate |
| **Gate** | US-201 Deploy Gateway | US-202 Route Traffic, US-203 JWT Auth | US-204 Discover Resolution, US-205 Rate Limiting, US-206 TLS |

---

## 2. Consolidated Story Inventory

| Service | Stories | Points | 🔴 Must | 🟡 Should | Document |
|---------|---------|--------|---------|----------|----------|
| Flowero Guard | 6 | 21 | 4 | 2 | [[flowero_guard/012_user_stories]] |
| Flowero Discover | 4 | 11 | 3 | 1 | [[flowero_discover/012_user_stories]] |
| Flowero Gate | 6 | 20 | 4 | 2 | [[flowero_gate/012_user_stories]] |
| **Phase 1 Total** | **16** | **52** | **11** | **5** | |

---

## 3. Cross-Service Traceability

### 3.1 Stories → Platform Objectives

| Platform Objective | Guard Stories | Discover Stories | Gate Stories |
|-------------------|--------------|-----------------|--------------|
| OBJ-01: Centralized Auth | US-001,002,003,004,005,006 | — | US-203 |
| OBJ-02: Service Discovery | — | US-101,102,103,104 | US-204 |
| OBJ-03: API Gateway | — | — | US-201,202,203,204,205,206 |
| OBJ-04: Onboarding DX | — | — | — (see platform starter template) |
| OBJ-05: Observability | US-001 (health) | US-101 (health) | US-201 (request logging) |

### 3.2 Inter-Service Story Dependencies

| Dependent Story | Depends On | Reason |
|----------------|-----------|--------|
| Gate US-203 (JWT Auth) | Guard US-001 (Keycloak deployed) | Gate needs Keycloak's JWKS endpoint to validate JWT tokens locally |
| Gate US-204 (Discover Resolution) | Discover US-101 (Eureka deployed) + US-102 (services registered) | Gate resolves `lb://` URIs from Eureka registry for business services |
| Guard US-003 (SSO) | Guard US-001 (Keycloak deployed) | Login flow at `auth.panomete.com` via Nginx — not through Gate |

---

## 4. Service-Level Documents

| Service | User Stories | Acceptance Criteria | Business Objectives | Architecture |
|---------|-------------|--------------------|--------------------|--------------|
| Flowero Guard | [[flowero_guard/012_user_stories]] | [[flowero_guard/013_acceptance_criteria]] | [[flowero_guard/011_business_objective]] | TBD |
| Flowero Discover | [[flowero_discover/012_user_stories]] | [[flowero_discover/013_acceptance_criteria]] | [[flowero_discover/011_business_objective]] | TBD |
| Flowero Gate | [[flowero_gate/012_user_stories]] | [[flowero_gate/013_acceptance_criteria]] | [[flowero_gate/011_business_objective]] | TBD |

---

## Related Documents

| Document | Relationship |
|----------|-------------|
| [[011_business_objective]] | Platform-level objectives |
| [[014_stakeholder_analysis]] | Platform stakeholders |
| [[README]] | Platform architecture overview |

---

> **Template Standard:** Based on SWEBOK v4, ISO/IEC/IEEE 29148
> **Usage:** This document is the index. For detailed stories, open the service-level documents linked above.
