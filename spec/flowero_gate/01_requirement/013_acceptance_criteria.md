---
document_type: Acceptance Criteria (ATDD/BDD)
version: "0.1"
status: Draft
author: "PO (Product Owner)"
created: "2026-07-22"
last_updated: "2026-07-22"
project_name: "Flowero Gate"
project_id: "PAN-GATE-001"
ba_owner: "PO (Product Owner)"
qa_lead: "QA persona (TBD)"
classification: "Internal"
tags: [acceptance-criteria, bdd, given-when-then, api-gateway, spring-cloud-gateway, panomete]
standard_ref:
  - SWEBOK v4 — Requirements
  - ISO/IEC/IEEE 29119 — Software Testing
  - ISO/IEC/IEEE 29148 — Requirements Engineering
---

# Acceptance Criteria — Flowero Gate

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
| QA Lead | QA persona (TBD) |

### Revision History

| Version | Date | Author | Change Description |
|---------|------|--------|--------------------|
| 0.1 | 2026-07-22 | PO | Initial draft — BDD criteria for all 6 stories |

---

## 1. Purpose

> This document defines acceptance criteria for every Flowero Gate user story using the **Given/When/Then** (GWT) format.

## 2. Acceptance Criteria

### US-201: Deploy API Gateway

| AC ID | Scenario | Given | When | Then | Priority |
|-------|---------|-------|------|------|----------|
| AC-W201a | Healthy startup | Docker Compose includes Gate service with ports 80:80 and 443:443 | `docker compose up` is executed | Gate is reachable at `https://panomete.local` (after accepting self-signed cert) within 30 seconds | 🔴 |
| AC-W201b | Default landing page | Gate starts with no business routes configured | Admin accesses `https://panomete.local` | Response is 200 with a default landing page OR 404 for unmatched route — not a connection refused | 🔴 |
| AC-W201c | Declarative route config | Routes are defined in `application.yml` under `spring.cloud.gateway.routes` | Gate starts | All routes are loaded and active; no runtime route configuration needed | 🔴 |
| AC-W201d | Structured request logging | Gate is processing requests | Admin checks Gate container logs | Each log line is valid JSON: `{"timestamp":"...", "method":"GET", "path":"/api/blog/posts", "status":200, "latency_ms":45, "route_id":"blog-route", "client_ip":"..."}` | 🔴 |
| AC-W201e | Health endpoint | Gate is running | Admin calls `GET /actuator/health` | Response 200 with `{"status": "UP"}` including discovery composite health (Eureka reachable?) | 🟡 |

### US-202: Route Traffic to Platform Services

| AC ID | Scenario | Given | When | Then | Priority |
|-------|---------|-------|------|------|----------|
| AC-W202a | Route to Keycloak (`/auth/**`) | Gate has route: `id=auth, path=/auth/**, uri=lb://flowero-guard` | Browser accesses `https://panomete.local/auth/realms/panomete/account` | Keycloak account console loads | 🔴 |
| AC-W202b | Route to Eureka (`/eureka/**`) | Gate has route: `id=discover, path=/eureka/**, uri=lb://flowero-discover` | Browser accesses `https://panomete.local/eureka/` | Eureka dashboard loads with all CSS and JS intact | 🔴 |
| AC-W202c | Route to business service (`/api/blog/**`) | Gate has route: `id=blog, path=/api/blog/**, uri=lb://cute-gufo` | Client sends `GET https://panomete.local/api/blog/posts` with valid auth | Request is forwarded to Cute Gufo; response 200 with blog posts | 🔴 |
| AC-W202d | Unmatched path — 404 | No route matches the path `/nonexistent` | Client sends `GET https://panomete.local/nonexistent` | Response 404 with JSON `{"error":"Not Found","path":"/nonexistent","timestamp":"..."}` | 🔴 |
| AC-W202e | Path rewriting | Route rewrites `/api/blog/**` to `/**` on the backend | Client sends `GET /api/blog/posts` | Backend service receives `GET /posts` (prefix stripped) | 🟡 |
| AC-W202f | Route restart without downtime | Gate is running with routes A, B, C | Admin adds route D to `application.yml` and restarts Gate | All 4 routes are active within 5 seconds; no in-flight requests are dropped during restart | 🟡 |

### US-203: Enforce JWT Authentication at the Perimeter

| AC ID | Scenario | Given | When | Then | Priority |
|-------|---------|-------|------|------|----------|
| AC-W203a | No token — 401 | Gate has JWT validation enabled on route `/api/**` | Client sends `GET /api/blog/posts` without Authorization header | Response 401 with `{"error":"Authentication required"}` | 🔴 |
| AC-W203b | Valid token — forwarded with claims | Client has a valid JWT from Keycloak (`sub: alice`, `realm_access.roles: [user]`) | Client sends `GET /api/blog/posts` with `Authorization: Bearer <jwt>` | Request is forwarded to backend with headers: `X-User-Id: alice`, `X-User-Roles: user` | 🔴 |
| AC-W203c | Expired token — 401 | Client has a JWT where `exp` is in the past | Client sends the expired token | Response 401 with `{"error":"Token expired"}` | 🔴 |
| AC-W203d | Token from unknown issuer | Client has a valid JWT but signed by a different Keycloak instance | Client sends the token | Response 401 with `{"error":"Invalid token issuer"}` | 🔴 |
| AC-W203e | Permit-all routes bypass auth | Routes `/auth/**` and `/eureka/**` are configured with `security: permit-all` | Unauthenticated client accesses `/auth/` or `/eureka/` | Request is forwarded without auth challenge; not a 401 | 🔴 |
| AC-W203f | Tampered token — 401 | Client has a JWT where the payload was modified (signature invalid) | Client sends the tampered token | Response 401 with `{"error":"Invalid token"}` — signature validation fails | 🔴 |

### US-204: Route Resolution via Service Discovery

| AC ID | Scenario | Given | When | Then | Priority |
|-------|---------|-------|------|------|----------|
| AC-W204a | Resolve `lb://` URI from Eureka | Route `uri: lb://cute-gufo` is configured; Cute Gufo is registered in Eureka | Client sends a request matching the route path | Gate resolves the actual host:port from Eureka and forwards the request | 🔴 |
| AC-W204b | Load balance across instances | 3 instances of Cute Gufo registered in Eureka | Client sends 30 requests | Requests are distributed across all 3 healthy instances (round-robin) | 🔴 |
| AC-W204c | Dead instance excluded | Instance #2 of Cute Gufo goes DOWN and is evicted from Eureka | Client sends the next request | Gate routes only to instances #1 and #3; instance #2 receives zero traffic | 🔴 |
| AC-W204d | Eureka temporarily unavailable | Eureka is briefly unreachable (network blip) | Gate needs to resolve a `lb://` URI | Gate uses cached instance list from last successful Eureka query; routing continues without failure | 🟡 |

### US-205: Rate Limiting

| AC ID | Scenario | Given | When | Then | Priority |
|-------|---------|-------|------|------|----------|
| AC-W205a | Exceed rate limit | Rate limit configured: 100 requests per minute per IP for `/api/**` | Client sends 101 requests within 60 seconds | 101st request returns 429 with `Retry-After` header | 🟡 |
| AC-W205b | Windows reset | Client was rate-limited at second 30 of the window | Client waits until second 0 of the next minute and sends request | Request succeeds (200) — rate limit counter resets each minute | 🟡 |
| AC-W205c | Per-route limits | Route `/api/blog/**` = 1000/min; `/auth/**` = 20/min | Rate-limited client sends requests to each route | Each route enforces its own limit independently; hitting `/auth` limit does not block `/api/blog` | 🟡 |
| AC-W205d | Different IPs isolated | Client IP 10.0.0.1 exceeds the limit | Client IP 10.0.0.2 sends a request | Client 10.0.0.2's request succeeds — rate limits are per-IP | 🟡 |

### US-206: TLS Termination

| AC ID | Scenario | Given | When | Then | Priority |
|-------|---------|-------|------|------|----------|
| AC-W206a | HTTPS encryption | Gate is configured with a TLS certificate (self-signed for dev) | Client connects via `https://panomete.local` | Connection is encrypted with TLS 1.2 or 1.3 | 🟡 |
| AC-W206b | Internal traffic is HTTP | Gate terminates TLS externally | Gate forwards request to backend service | Internal request uses plain HTTP (trusted Docker network) | 🟡 |
| AC-W206c | HTTP redirects to HTTPS | Gate listens on port 80 and 443 | Client connects via `http://panomete.local` | Gate returns 301 redirect to `https://panomete.local` | 🟡 |

---

## 3. Acceptance Criteria Summary

| Story | AC Count | 🔴 Must Have | 🟡 Should Have |
|-------|---------|-------------|---------------|
| US-201: Deploy Gateway | 5 | 4 | 1 |
| US-202: Route Traffic | 6 | 4 | 2 |
| US-203: JWT Auth at Perimeter | 6 | 6 | 0 |
| US-204: Discover Resolution | 4 | 3 | 1 |
| US-205: Rate Limiting | 4 | 0 | 4 |
| US-206: TLS Termination | 3 | 0 | 3 |
| **Total** | **28** | **17** | **11** |

---

## Related Documents

| Document | Relationship |
|----------|-------------|
| [[flowero_gate/012_user_stories]] | Stories these criteria verify |
| [[flowero_gate/011_business_objective]] | Objectives these criteria support |
| [[flowero_guard/012_user_stories]] | Token validation relies on Guard |
| [[flowero_discover/012_user_stories]] | Route resolution relies on Discover |

---

> **Template Standard:** Based on SWEBOK v4, ISO/IEC/IEEE 29119, ISO/IEC/IEEE 29148
