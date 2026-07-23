---
document_type: Business Objectives
version: "0.2"
status: Draft
author: "PO (Product Owner)"
created: "2026-07-22"
last_updated: "2026-07-22"
project_name: "Flowero Gate"
project_id: "PAN-GATE-001"
parent_platform: "Panomete Platform"
sponsor: "Self"
ba_owner: "PO (Product Owner)"
classification: "Internal"
tags: [business-objectives, api-gateway, spring-cloud-gateway, panomete]
---

# Business Objectives — Flowero Gate

> **Service:** Flowero Gate (Spring Cloud Gateway)
> **Platform:** Panomete Platform
> **Version:** 0.2 | **Status:** Draft — Updated per Design Review 2026-07-22
> **Last Updated:** 2026-07-22
>
> ⚠️ See [[panomete_platform/011_business_objective]] for platform-level objectives.

---

## 1. Service Mission

> Flowero Gate is the API gateway for **business services only**. Its job: **route, secure, and rate-limit** business API traffic at `api.panomete.com`. Foundation services (Guard, Discover) are routed directly by Nginx — Gate does not see that traffic.

## 2. Service Context

| Aspect | Detail |
|--------|--------|
| **Service Type** | Foundation — API Gateway (business APIs only) |
| **Technology** | Spring Cloud Gateway (Reactive, Netty-based) |
| **Language / Stack** | Java 25 / Spring Boot 4.1.x / WebFlux |
| **Port** | 8000 (internal, behind Nginx) |
| **Domain** | `api.panomete.com` |
| **Dependencies** | Flowero Guard (JWKS endpoint for JWT validation), Flowero Discover (route resolution), Valkey 9 (rate limiting) |
| **Routes To** | Business services only: `/api/blog/**`, `/api/short/**`, `/api/todo/**`, `/api/ledger/**`, `/api/recipe/**`, `/api/hora/**` |
| **Does NOT Route** | Guard (`auth.panomete.com`) and Discover (`discovery.panomete.com`) — handled by Nginx directly |

## 3. Service Objectives

### OBJ-GATE-01: Deploy Internal API Gateway

| Field | Detail |
|-------|--------|
| **Statement** | Deploy Spring Cloud Gateway on internal port 8000, exposed at `api.panomete.com` through Nginx. |
| **Measurable** | Gateway healthy within 30s. Routes defined in `application.yml`. All requests logged as JSON. |
| **Priority** | 🔴 Must Have |
| **Owner** | Dev persona |
| **Parent Objective** | OBJ-03 |

### OBJ-GATE-02: Route Business APIs Only

| Field | Detail |
|-------|--------|
| **Statement** | Route business API traffic using path-based rules. Guard and Discover are NOT routed through Gate. |
| **Measurable** | Business services reachable via `api.panomete.com/api/{service}/**`. Unmatched paths return 404. Routes resolved from Discover (`lb://`). |
| **Priority** | 🔴 Must Have |
| **Owner** | Dev persona |
| **Parent Objective** | OBJ-03 |

### OBJ-GATE-03: Perimeter JWT Validation

| Field | Detail |
|-------|--------|
| **Statement** | Validate JWT tokens locally against Keycloak JWKS on every request. Forward user claims as headers. No per-request call to Guard. |
| **Measurable** | No token → 401. Valid token → forwarded with `X-User-Id`, `X-User-Roles`. Expired/tampered → 401. |
| **Priority** | 🔴 Must Have |
| **Owner** | Dev persona |
| **Parent Objective** | OBJ-01, OBJ-03 |

### OBJ-GATE-04: Valkey-Backed Rate Limiting

| Field | Detail |
|-------|--------|
| **Statement** | Use shared Valkey 9 for rate limiting so limits persist across Gate restarts. |
| **Measurable** | Client exceeds limit → 429 with `Retry-After`. Limits survive Gate restart. Per-route configurable limits. |
| **Priority** | 🟡 Should Have |
| **Owner** | Dev persona |
| **Parent Objective** | OBJ-03 |

---

## 4. Out of Scope

- TLS termination (Cloudflare handles it)
- Routing auth/discovery traffic (Nginx handles it)
- WAF features
- Custom filter chains beyond JWT validation + rate limiting

---

## Related Documents

| Document | Relationship |
|----------|-------------|
| [[flowero_gate/012_user_stories]] | Stories that deliver these objectives |
| [[flowero_gate/013_acceptance_criteria]] | Testable criteria for each story |
| [[panomete_platform/011_business_objective]] | Platform-level objectives |
