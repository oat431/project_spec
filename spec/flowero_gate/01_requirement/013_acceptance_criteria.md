---
document_type: Acceptance Criteria (ATDD/BDD)
version: "0.2"
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
> **Version:** 0.2 | **Status:** Draft — Updated per Design Review 2026-07-22
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
| 0.1 | 2026-07-22 | PO | Initial draft |
| 0.2 | 2026-07-22 | PO | Design review — removed auth/discover routing ACs, removed TLS ACs, port 8000, api.panomete.com, Valkey rate limiting |

---

## 1. Purpose

> This document defines acceptance criteria for every Flowero Gate user story. Gate routes business APIs only at `api.panomete.com`. Foundation services are handled by Nginx.

## 2. Acceptance Criteria

### US-201: Deploy API Gateway

| AC ID | Scenario | Given | When | Then | Priority |
|-------|---------|-------|------|------|----------|
| AC-W201a | Healthy startup | Docker Compose includes Gate service with port 8000 | `docker compose up` is executed | Gate is reachable at `api.panomete.com` (via Nginx) within 30 seconds | 🔴 |
| AC-W201b | Default response | Gate starts with no business routes configured | Admin accesses `api.panomete.com` | Response is 200 with a default landing page OR 404 for unmatched route — not a connection refused | 🔴 |
| AC-W201c | Declarative route config | Routes are defined in `application.yml` under `spring.cloud.gateway.routes` | Gate starts | All routes are loaded and active; no runtime route configuration needed | 🔴 |
| AC-W201d | Structured request logging | Gate is processing requests | Admin checks Gate container logs | Each log line is valid JSON: `{"timestamp":"...", "method":"GET", "path":"/api/blog/posts", "status":200, "latency_ms":45, "route_id":"blog-route", "client_ip":"..."}` | 🔴 |
| AC-W201e | Health endpoint | Gate is running | Admin calls `GET /actuator/health` | Response 200 with `{"status": "UP"}` including Valkey and Eureka composite health | 🟡 |

### US-202: Route Business API Traffic

| AC ID | Scenario | Given | When | Then | Priority |
|-------|---------|-------|------|------|----------|
| AC-W202a | Route to blog service | Gate has route: `id=blog, path=/api/blog/**, uri=lb://cute-gufo` | Client sends `GET api.panomete.com/api/blog/posts` with valid auth | Request forwarded to Cute Gufo; response 200 with blog posts | 🔴 |
| AC-W202b | Route to URL shortener | Gate has route: `id=short, path=/api/short/**, uri=lb://fluffy-mouton` | Client sends `GET api.panomete.com/api/short/abc123` | Request forwarded to Fluffy Mouton; short URL resolved | 🔴 |
| AC-W202c | Unmatched path — 404 | No route matches the path `/nonexistent` | Client sends `GET api.panomete.com/nonexistent` | Response 404 with JSON `{"error":"Not Found","path":"/nonexistent","timestamp":"..."}` | 🔴 |
| AC-W202d | Path rewriting | Route rewrites `/api/blog/**` to `/**` on the backend | Client sends `GET /api/blog/posts` | Backend receives `GET /posts` (prefix stripped) | 🟡 |
| AC-W202e | Route restart without downtime | Gate is running with routes A, B, C | Admin adds route D to `application.yml` and restarts Gate | All routes are active within 5 seconds; no in-flight requests dropped | 🟡 |

### US-203: Enforce JWT Authentication

| AC ID | Scenario | Given | When | Then | Priority |
|-------|---------|-------|------|------|----------|
| AC-W203a | No token — 401 | Gate has JWT validation enabled on `api.panomete.com/api/**` | Client sends request without Authorization header | Response 401 with `{"error":"Authentication required"}` | 🔴 |
| AC-W203b | Valid token — forwarded with claims | Client has valid JWT from Keycloak (`sub: alice`, `realm_access.roles: [user]`) | Client sends request with `Authorization: Bearer <jwt>` | Request forwarded with headers: `X-User-Id: alice`, `X-User-Roles: user` | 🔴 |
| AC-W203c | Expired token — 401 | Client has JWT where `exp` is in the past | Client sends the expired token | Response 401 with `{"error":"Token expired"}` | 🔴 |
| AC-W203d | Token from unknown issuer | Client has valid JWT but signed by a different Keycloak instance | Client sends the token | Response 401 with `{"error":"Invalid token issuer"}` | 🔴 |
| AC-W203e | Tampered token — 401 | Client has JWT where the payload was modified (signature invalid) | Client sends the tampered token | Response 401 with `{"error":"Invalid token"}` — signature validation fails | 🔴 |
| AC-W203f | JWKS caching | Gate has successfully fetched Keycloak JWKS on startup | Gate validates 1000 tokens | No network calls to Keycloak — all validation is local against cached JWKS | 🟡 |

### US-204: Route Resolution via Service Discovery

| AC ID | Scenario | Given | When | Then | Priority |
|-------|---------|-------|------|------|----------|
| AC-W204a | Resolve `lb://` URI from Eureka | Route `uri: lb://cute-gufo`; Cute Gufo registered in Eureka | Client sends a matching request | Gate resolves actual host:port from Eureka and forwards the request | 🔴 |
| AC-W204b | Load balance across instances | 3 instances of Cute Gufo registered in Eureka | Client sends 30 requests | Requests distributed across all healthy instances (round-robin) | 🔴 |
| AC-W204c | Dead instance excluded | Instance #2 goes DOWN and is evicted from Eureka | Client sends next request | Gate routes only to instances #1 and #3; instance #2 receives zero traffic | 🔴 |
| AC-W204d | Eureka temporarily unavailable | Eureka is briefly unreachable | Gate needs to resolve a `lb://` URI | Gate uses cached instance list from last successful Eureka query; routing continues | 🟡 |

### US-205: Valkey-Backed Rate Limiting

| AC ID | Scenario | Given | When | Then | Priority |
|-------|---------|-------|------|------|----------|
| AC-W205a | Exceed rate limit | Rate limit: 100 req/min per IP for `api.panomete.com/api/**`, backed by Valkey | Client sends 101 requests within 60 seconds | 101st request returns 429 with `Retry-After` header | 🟡 |
| AC-W205b | Window resets | Client was rate-limited at second 30 of the window | Client waits until next minute and sends request | Request succeeds (200) — rate limit counter resets | 🟡 |
| AC-W205c | Limits survive Gate restart | Client is rate-limited; rate counters in Valkey | `docker compose restart gate` | After restart, the same client is still rate-limited — counters persisted in Valkey | 🟡 |
| AC-W205d | Per-route limits | Route `/api/blog/**` = 1000/min; `/api/short/**` = 100/min | Rate-limited client sends requests to each route | Each route enforces its own limit independently | 🟡 |

---

## 3. Acceptance Criteria Summary

| Story | AC Count | 🔴 Must Have | 🟡 Should Have |
|-------|---------|-------------|---------------|
| US-201: Deploy Gateway | 5 | 4 | 1 |
| US-202: Route Business APIs | 5 | 3 | 2 |
| US-203: JWT Authentication | 6 | 5 | 1 |
| US-204: Discover Resolution | 4 | 3 | 1 |
| US-205: Valkey Rate Limiting | 4 | 0 | 4 |
| **Total** | **24** | **15** | **9** |

---

## Related Documents

| Document | Relationship |
|----------|-------------|
| [[flowero_gate/012_user_stories]] | Stories these criteria verify |
| [[flowero_gate/011_business_objective]] | Objectives these criteria support |
| [[flowero_guard/012_user_stories]] | Token validation relies on Guard JWKS |
| [[flowero_discover/012_user_stories]] | Route resolution relies on Discover |

---

> **Template Standard:** Based on SWEBOK v4, ISO/IEC/IEEE 29119, ISO/IEC/IEEE 29148
