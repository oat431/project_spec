---
document_type: User Stories
version: "0.2"
status: Draft
author: "PO (Product Owner)"
created: "2026-07-22"
last_updated: "2026-07-22"
project_name: "Flowero Gate"
project_id: "PAN-GATE-001"
parent_platform: "Panomete Platform"
ba_owner: "PO (Product Owner)"
po_owner: "PO (Product Owner)"
classification: "Internal"
tags: [user-stories, api-gateway, spring-cloud-gateway, routing, panomete]
standard_ref:
  - SWEBOK v4 — Requirements
  - ISO/IEC/IEEE 29148 — Requirements Engineering
---

# User Stories — Flowero Gate

> **Service:** Flowero Gate (Spring Cloud Gateway)
> **Platform:** Panomete Platform
> **Version:** 0.2 | **Status:** Draft — Updated per Design Review 2026-07-22
> **Last Updated:** 2026-07-22

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
| 0.1 | 2026-07-22 | PO | Initial draft |
| 0.2 | 2026-07-22 | PO | Design review update — port 8000, api.panomete.com, remove auth/discover routing, remove TLS (Cloudflare), Valkey rate limiting |

---

## 1. Purpose

> Flowero Gate is the API gateway for **business services only**. It routes traffic at `api.panomete.com`, validates JWT locally against Keycloak JWKS, and enforces Valkey-backed rate limiting. Foundation services (Guard, Discover) are routed directly by Nginx — Gate does not see that traffic.

## 2. Service Context

| Aspect | Detail |
|--------|--------|
| **Service Type** | Foundation — API Gateway (business APIs only) |
| **Technology** | Spring Cloud Gateway (Reactive, Netty-based) |
| **Language / Stack** | Java 25 / Spring Boot 4.1.x / WebFlux |
| **Port** | 8000 (internal, behind Nginx) |
| **Domain** | `api.panomete.com` |
| **Dependencies** | Flowero Guard (JWKS endpoint for local JWT validation), Flowero Discover (route resolution for business services), Valkey 9 (rate limiting) |
| **Routes To** | Business services only: `/api/blog/**`, `/api/short/**`, `/api/todo/**`, `/api/ledger/**`, `/api/recipe/**`, `/api/hora/**` |
| **Does NOT Route** | Guard (`auth.panomete.com`) and Discover (`discovery.panomete.com`) — handled by Nginx |

## 3. Personas

| Persona | Role |
|---------|------|
| **Platform Admin** | Configures routes, rate limits. Does NOT configure Gate for auth/discover — Nginx handles those. |
| **Service Developer** | Adds a route for their new business service in gateway config. Inherits JWT auth automatically. |
| **End User** | Accesses business APIs through `api.panomete.com`. Logs in at `auth.panomete.com`. |

---

## 4. User Stories

### US-201: Deploy API Gateway

**As a** Platform Admin
**I want** to deploy a Spring Cloud Gateway on internal port 8000, exposed at `api.panomete.com` through Nginx
**So that** all business API traffic is routed, secured, and rate-limited through a single internal service

**Acceptance Criteria:**
- **AC-1:** Given `docker compose up`, When the Gateway container starts, Then it listens on port 8000 and is accessible at `api.panomete.com` (via Nginx) within 30 seconds
- **AC-2:** Given the Gateway is running with no business routes configured, When I access `api.panomete.com`, Then it returns a 200 with a default landing page or 404 for unmatched routes
- **AC-3:** Given the Gateway configuration, When I inspect the `application.yml`, Then routes are defined declaratively with: `id`, `uri` (using `lb://` prefix for Discover-resolved routes), and `predicates` (path patterns)
- **AC-4:** Given a request passes through Gate, When I check the logs, Then each request is logged in JSON format with: `timestamp`, `method`, `path`, `status`, `latency_ms`, `route_id`, and `client_ip`

**Story Points:** 5
**Priority:** 🔴 Must Have
**Sprint:** M1 — Core Infrastructure
**Status:** Draft
**Mapped Objective:** OBJ-03 (API Gateway)

---

### US-202: Route Business API Traffic

**As an** End User
**I want** to access all business services through `api.panomete.com` with clean URL paths
**So that** I don't need to know which port each service runs on

**Acceptance Criteria:**
- **AC-1:** Given Gateway has route `id: blog`, path `/api/blog/**` → `lb://cute-gufo`, When I access `api.panomete.com/api/blog/posts`, Then the request is forwarded to the Cute Gufo service
- **AC-2:** Given Gateway has route `id: short`, path `/api/short/**` → `lb://fluffy-mouton`, When I access `api.panomete.com/api/short/abc123`, Then the request is forwarded to the Fluffy Mouton service
- **AC-3:** Given a request to an unmatched path (e.g., `api.panomete.com/nonexistent`), When Gateway processes it, Then it returns 404 with a JSON body: `{"error": "Not Found", "path": "/nonexistent"}`
- **AC-4:** Given route configuration changes, When I restart Gateway, Then new routes take effect without data loss or downtime (<5 second restart)

**Story Points:** 3
**Priority:** 🔴 Must Have
**Sprint:** M2 — End-to-End Auth
**Status:** Draft
**Mapped Objective:** OBJ-03

---

### US-203: Enforce JWT Authentication

**As a** Platform Admin
**I want** the Gateway to validate JWT tokens locally against Keycloak's JWKS on every request to protected routes
**So that** unauthenticated traffic is rejected at the API perimeter before reaching any business service

**Acceptance Criteria:**
- **AC-1:** Given a request to `api.panomete.com/api/**`, When the request has no `Authorization: Bearer <token>` header, Then Gateway returns 401 with `{"error": "Authentication required"}`
- **AC-2:** Given a request with a valid JWT issued by Flowero Guard, When Gateway validates the token signature against the JWKS endpoint (cached on startup), Then the request is forwarded with headers: `X-User-Id`, `X-User-Name`, `X-User-Roles`
- **AC-3:** Given a request with an expired JWT, When Gateway validates the `exp` claim, Then it returns 401 with `{"error": "Token expired"}`
- **AC-4:** Given a request with a JWT signed by an unknown issuer, When Gateway validates, Then it returns 401 with `{"error": "Invalid token issuer"}`
- **AC-5:** Given a request with a tampered JWT (modified payload, invalid signature), When Gateway validates, Then it returns 401 with `{"error": "Invalid token"}`

**Story Points:** 3
**Priority:** 🔴 Must Have
**Sprint:** M2 — End-to-End Auth
**Status:** Draft
**Mapped Objective:** OBJ-01, OBJ-03

---

### US-204: Route Resolution via Service Discovery

**As a** Platform Admin
**I want** the Gateway to resolve business service addresses from Eureka instead of hardcoded host:port
**So that** routes automatically adapt when services scale, restart, or move

**Acceptance Criteria:**
- **AC-1:** Given a route defined with `uri: lb://service-name`, When Gateway forwards a request, Then it resolves the actual host and port from Eureka's registry
- **AC-2:** Given 3 instances of a service are registered in Eureka, When Gateway routes traffic, Then requests are load-balanced across all healthy instances (round-robin)
- **AC-3:** Given a service instance goes DOWN and is evicted from Eureka (after 90s), When Gateway routes the next request, Then traffic is no longer sent to the dead instance
- **AC-4:** Given Eureka is temporarily unavailable, When Gateway needs to resolve a route, Then it uses the last known instance list (cached) and continues routing — no cascading failure

**Story Points:** 3
**Priority:** 🔴 Must Have
**Sprint:** M3 — Production Hardening
**Status:** Draft
**Mapped Objective:** OBJ-02, OBJ-03

---

### US-205: Valkey-Backed Rate Limiting

**As a** Platform Admin
**I want** to configure rate limits backed by the shared Valkey 9 instance
**So that** rate limits persist across Gate restarts and a single client cannot overwhelm the platform

**Acceptance Criteria:**
- **AC-1:** Given a rate limit of 100 requests per minute per client IP (stored in Valkey), When a client sends 101 requests within a minute, Then the 101st request receives 429 Too Many Requests
- **AC-2:** Given a 429 response, When the client receives it, Then the response includes a `Retry-After` header indicating when they can retry
- **AC-3:** Given the client waits for the rate limit window to reset, When they send the next request, Then it is processed normally (200)
- **AC-4:** Given Gate restarts, When it comes back up, Then rate limit counters from Valkey are preserved — a previously rate-limited client remains limited
- **AC-5:** Given different routes need different limits (e.g., `/api/blog/**` = 1000/min, `/api/short/**` = 100/min), When I configure per-route limits, Then each route enforces its own limit independently

**Story Points:** 3
**Priority:** 🟡 Should Have
**Sprint:** M3 — Production Hardening
**Status:** Draft
**Mapped Objective:** OBJ-03

---

## 5. Story Estimation Summary

| Story | Name | Points | Priority | Sprint |
|-------|------|--------|----------|--------|
| US-201 | Deploy API Gateway | 5 | 🔴 | M1 |
| US-202 | Route Business APIs | 3 | 🔴 | M2 |
| US-203 | JWT Authentication | 3 | 🔴 | M2 |
| US-204 | Route Resolution via Discover | 3 | 🔴 | M3 |
| US-205 | Valkey Rate Limiting | 3 | 🟡 | M3 |
| **Total** | | **17** | | |

---

## Related Documents

| Document | Relationship |
|----------|-------------|
| [[panomete_platform/011_business_objective]] | Platform-level objectives (OBJ-01, OBJ-02, OBJ-03) |
| [[flowero_gate/011_business_objective]] | Service-level objectives |
| [[flowero_gate/013_acceptance_criteria]] | BDD acceptance criteria for all stories |
| [[flowero_guard/012_user_stories]] | Gate validates JWT against Guard's JWKS |
| [[flowero_discover/012_user_stories]] | Gate resolves business service routes through Discover |

---

> **Template Standard:** Based on SWEBOK v4, ISO/IEC/IEEE 29148
> **Usage:** These stories define what Flowero Gate must deliver. Gate routes business APIs only. Foundation services are routed by Nginx.
