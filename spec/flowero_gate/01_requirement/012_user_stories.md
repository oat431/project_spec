---
document_type: User Stories
version: "0.1"
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
> **Version:** 0.1 | **Status:** Draft
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
| 0.1 | 2026-07-22 | PO | Initial draft — extracted from platform-level stories |

---

## 1. Purpose

> Flowero Gate is the single entry point for ALL external traffic to the Panomete Platform — a Spring Cloud Gateway that routes requests to internal services, enforces JWT authentication at the perimeter, resolves backend addresses dynamically via Flowero Discover, and provides rate limiting. Even Keycloak login pages and the Eureka dashboard are accessed through Gate.

## 2. Service Context

| Aspect | Detail |
|--------|--------|
| **Service Type** | Foundation — API Gateway |
| **Technology** | Spring Cloud Gateway (Reactive, Netty-based) |
| **Language / Stack** | Java 21 / Spring Boot 3.x / WebFlux |
| **Ports** | 80 (HTTP), 443 (HTTPS) — the only ports exposed externally |
| **Dependencies** | Flowero Guard (token validation), Flowero Discover (route resolution) |
| **Consumed By** | All external clients (browsers, mobile apps, API consumers) |
| **Routes To** | All platform services: Guard, Discover, and business services |

## 3. Personas

| Persona | Role |
|---------|------|
| **Platform Admin** | Configures routes, rate limits, and TLS. Monitors gateway metrics. |
| **Service Developer** | Adds a route for their new service in gateway config. Expects it to "just work" with auth inherited from Gate. |
| **End User** | Accesses all platform services through `https://panomete.local`. Never sees internal ports. |

---

## 4. User Stories

### US-201: Deploy API Gateway

**As a** Platform Admin
**I want** to deploy a Spring Cloud Gateway via Docker Compose
**So that** all external traffic enters the platform through a single, secure, observable entry point

**Acceptance Criteria:**
- **AC-1:** Given `docker compose up`, When the Gateway container starts, Then it listens on ports 80 and 443 and is accessible at `https://panomete.local` within 30 seconds
- **AC-2:** Given the Gateway is running with no routes configured, When I access `https://panomete.local`, Then it returns a 200 with a default landing page or 404 for unmatched routes
- **AC-3:** Given the Gateway configuration, When I inspect the `application.yml`, Then routes are defined declaratively with: `id`, `uri` (using `lb://` prefix for Discover-resolved routes), and `predicates` (path patterns)
- **AC-4:** Given a request passes through Gate, When I check the logs, Then each request is logged in JSON format with: `timestamp`, `method`, `path`, `status`, `latency_ms`, `route_id`, and `client_ip`

**Story Points:** 5
**Priority:** 🔴 Must Have
**Sprint:** M1 — Core Infrastructure
**Status:** Draft
**Mapped Objective:** OBJ-03 (API Gateway)

---

### US-202: Route Traffic to Platform Services

**As an** End User
**I want** to access all platform services through `https://panomete.local` with clean URL paths
**So that** I don't need to know or remember which port each service runs on

**Acceptance Criteria:**
- **AC-1:** Given Gateway is configured with route `id: auth`, path `/auth/**` → `lb://flowero-guard`, When I access `https://panomete.local/auth/`, Then I see the Keycloak login page
- **AC-2:** Given Gateway has route `id: discover`, path `/eureka/**` → `lb://flowero-discover`, When I access `https://panomete.local/eureka/`, Then I see the Eureka dashboard
- **AC-3:** Given Gateway has route `id: blog`, path `/api/blog/**` → `lb://cute-gufo`, When I access `https://panomete.local/api/blog/posts`, Then the request is forwarded to the Cute Gufo service
- **AC-4:** Given a request to an unmatched path (e.g., `https://panomete.local/nonexistent`), When Gateway processes it, Then it returns 404 with a JSON body: `{"error": "Not Found", "path": "/nonexistent"}`
- **AC-5:** Given route configuration changes, When I restart Gateway, Then new routes take effect without data loss or downtime (<5 second restart)

**Story Points:** 5
**Priority:** 🔴 Must Have
**Sprint:** M2 — End-to-End Auth
**Status:** Draft
**Mapped Objective:** OBJ-03

---

### US-203: Enforce JWT Authentication at the Perimeter

**As a** Platform Admin
**I want** the Gateway to validate JWT tokens on every request to protected routes
**So that** unauthenticated traffic is rejected at the network edge before reaching any internal service

**Acceptance Criteria:**
- **AC-1:** Given a request to a protected route (`/api/**`), When the request has no `Authorization: Bearer <token>` header, Then Gateway returns 401 with `{"error": "Authentication required"}`
- **AC-2:** Given a request with a valid JWT issued by Flowero Guard, When Gateway validates the token signature against Keycloak's JWKS endpoint, Then the request is forwarded with headers: `X-User-Id`, `X-User-Name`, `X-User-Roles` populated from JWT claims
- **AC-3:** Given a request with an expired JWT, When Gateway validates the `exp` claim, Then it returns 401 with `{"error": "Token expired"}`
- **AC-4:** Given a request with a JWT signed by an unknown issuer, When Gateway validates, Then it returns 401 with `{"error": "Invalid token issuer"}`
- **AC-5:** Given the `/auth/**` and `/eureka/**` routes, When configured as permit-all, Then unauthenticated requests to these paths are allowed through

**Story Points:** 3
**Priority:** 🔴 Must Have
**Sprint:** M2 — End-to-End Auth
**Status:** Draft
**Mapped Objective:** OBJ-01, OBJ-03

---

### US-204: Route Resolution via Service Discovery

**As a** Platform Admin
**I want** the Gateway to resolve backend service addresses from Eureka instead of hardcoded host:port
**So that** routes automatically adapt when services scale, restart, or move — no gateway config changes needed

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

### US-205: Rate Limiting per Client

**As a** Platform Admin
**I want** to configure rate limits on the Gateway to prevent any single client from overwhelming the platform
**So that** the platform remains available and responsive for all users under load

**Acceptance Criteria:**
- **AC-1:** Given a rate limit of 100 requests per minute per client IP, When a client sends 101 requests within a minute, Then the 101st request receives 429 Too Many Requests
- **AC-2:** Given a 429 response, When the client receives it, Then the response includes a `Retry-After` header indicating when they can retry (in seconds)
- **AC-3:** Given the client waits for the rate limit window to reset (60 seconds), When they send the next request, Then it is processed normally (200)
- **AC-4:** Given different routes need different limits (e.g., `/api/blog/**` = 1000/min, `/auth/**` = 20/min for login), When I configure per-route limits, Then each route enforces its own limit independently

**Story Points:** 2
**Priority:** 🟡 Should Have
**Sprint:** M3 — Production Hardening
**Status:** Draft
**Mapped Objective:** OBJ-03

---

### US-206: TLS Termination

**As a** Platform Admin
**I want** the Gateway to terminate TLS with a valid certificate
**So that** all traffic between clients and the platform is encrypted

**Acceptance Criteria:**
- **AC-1:** Given the Gateway is configured with a TLS certificate, When a client connects via HTTPS, Then the connection is encrypted with TLS 1.2 or higher
- **AC-2:** Given the Gateway terminates TLS, When traffic is forwarded to internal services, Then internal traffic can use plain HTTP (internal Docker network is trusted)
- **AC-3:** Given a self-signed certificate for local development, When a browser connects, Then the admin can accept the certificate warning and proceed (acceptable for homelab)

**Story Points:** 2
**Priority:** 🟡 Should Have
**Sprint:** M3 — Production Hardening
**Status:** Draft
**Mapped Objective:** OBJ-03

---

## 5. Story Estimation Summary

| Story | Name | Points | Priority | Sprint |
|-------|------|--------|----------|--------|
| US-201 | Deploy API Gateway | 5 | 🔴 | M1 |
| US-202 | Route Traffic to Services | 5 | 🔴 | M2 |
| US-203 | JWT Auth at Perimeter | 3 | 🔴 | M2 |
| US-204 | Route Resolution via Discover | 3 | 🔴 | M3 |
| US-205 | Rate Limiting | 2 | 🟡 | M3 |
| US-206 | TLS Termination | 2 | 🟡 | M3 |
| **Total** | | **20** | | |

---

## Related Documents

| Document | Relationship |
|----------|-------------|
| [[panomete_platform/011_business_objective]] | Platform-level objectives (OBJ-01, OBJ-02, OBJ-03) |
| [[flowero_gate/011_business_objective]] | Service-level objectives |
| [[flowero_gate/013_acceptance_criteria]] | BDD acceptance criteria for all stories |
| [[flowero_guard/012_user_stories]] | Gate validates tokens against Guard |
| [[flowero_discover/012_user_stories]] | Gate resolves routes through Discover |

---

> **Template Standard:** Based on SWEBOK v4, ISO/IEC/IEEE 29148
> **Usage:** These stories define what Flowero Gate must deliver. Gate is the most critical service — if it's down, the entire platform is unreachable.
