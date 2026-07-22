---
document_type: API Specification
version: "0.1"
status: Draft
author: "Dev / SA Persona"
created: "2026-07-22"
last_updated: "2026-07-22"
project_name: "Flowero Gate"
project_id: "PAN-GATE-001"
parent_platform: "Panomete Platform"
tech_lead: "Dev / SA Persona"
classification: "Internal"
tags: [api-specification, api-gateway, spring-cloud-gateway, routing, panomete]
standard_ref:
  - SWEBOK v4 — Design
  - OpenAPI Specification 3.0
---

# API Specification — Flowero Gate

> **Service:** Flowero Gate (Spring Cloud Gateway)
> **Platform:** Panomete Platform
> **Version:** 0.1 | **Status:** Draft
> **Last Updated:** 2026-07-22

---

## 1. Purpose

> Flowero Gate is the **single entry point** for all external traffic to the Panomete Platform. This document defines the Gateway's "API" — the public-facing route table, authentication behavior, error responses, rate limiting, and headers. Business service APIs are documented in their own specs — this document shows how to reach them.

---

## 2. API Overview

| Field | Detail |
|-------|--------|
| **Base URL** | `https://panomete.local` |
| **Protocol** | HTTPS (TLS 1.2+), with HTTP→HTTPS redirect on :80 |
| **Format** | JSON (errors, health), pass-through for proxied services |
| **Authentication** | Bearer JWT token (issued by Flowero Guard) |
| **Rate Limiting** | 100 req/min per client IP (default); 20/min for `/auth/**` |

---

## 3. Route Table (Public API Surface)

> This is the complete public API of the Panomete Platform. Every URL a client can access.

### Phase 1 — Foundation Services

| Route ID | Method | Path | Backend | Auth Required | Rate Limit | Description |
|----------|:---:|------|---------|:---:|:---:|---|
| `auth` | ALL | `/auth/**` | `lb://flowero-guard` | No (permit-all) | 20/min | Keycloak — login, token, logout |
| `discover` | ALL | `/eureka/**` | `lb://flowero-discover` | Yes (`admin` role) | 100/min | Eureka dashboard |
| `health` | GET | `/actuator/health` | Gateway itself | No | 100/min | Gateway health check |

### Phase 2+ — Business Services

| Route ID | Method | Path | Backend | Auth Required | Rate Limit | Description |
|----------|:---:|------|---------|:---:|:---:|---|
| `blog` | ALL | `/api/blog/**` | `lb://cute-gufo` | Yes | 100/min | Blog service |
| `short` | ALL | `/api/short/**` | `lb://fluffy-mouton` | Yes | 100/min | URL shortener |
| `todo` | ALL | `/api/todo/**` | `lb://tiny-mchwa` | Yes | 100/min | Todo list |
| `ledger` | ALL | `/api/ledger/**` | `lb://big-schwein` | Yes | 100/min | Ledger |
| `recipe` | ALL | `/api/recipe/**` | `lb://shy-ardilla` | Yes | 100/min | Cook book |
| `hora` | ALL | `/api/hora/**` | `lb://white-jelen` | Yes | 100/min | Hora |

---

## 4. Gateway-Specific Endpoints

### 4.1 Health Check

#### GET /actuator/health

| Field | Detail |
|-------|--------|
| **Description** | Gateway health status — includes composite health for Eureka connectivity |
| **Auth** | None |

**Response (200):**
```json
{
  "status": "UP",
  "components": {
    "discoveryComposite": {
      "status": "UP",
      "components": {
        "discoveryClient": {
          "status": "UP",
          "details": {
            "services": ["FLOWERO-GUARD", "FLOWERO-DISCOVER"]
          }
        }
      }
    }
  }
}
```

---

### 4.2 Default Landing Page

#### GET /

| Field | Detail |
|-------|--------|
| **Description** | Default landing page when no route matches |

**Response (200):** Simple HTML or plain text indicating the Gateway is running.
```
Panomete Platform — API Gateway
━━━━━━━━━━━━━━━━━━━━━━━━━━━━
All services are accessed through this gateway.
See /actuator/health for service status.
```

---

## 5. Authentication Behavior

### 5.1 Protected Routes

> All routes except `/auth/**` and `/actuator/health` require authentication.

**Without token:**
```
HTTP/1.1 401 Unauthorized
Content-Type: application/json

{
  "error": "Authentication required",
  "path": "/api/blog/posts",
  "timestamp": "2026-07-22T10:00:00Z"
}
```

**With valid token:** Request is forwarded with user claim headers:

| Header | Source (JWT claim) | Example |
|--------|-------------------|---------|
| `X-User-Id` | `sub` | `a1b2c3d4-e5f6-7890-abcd-ef1234567890` |
| `X-User-Name` | `preferred_username` | `alice` |
| `X-User-Roles` | `realm_access.roles` | `admin,user` |

**With expired token:**
```
HTTP/1.1 401 Unauthorized
Content-Type: application/json

{
  "error": "Token expired",
  "path": "/api/blog/posts",
  "timestamp": "2026-07-22T10:00:00Z"
}
```

**With tampered/invalid token:**
```
HTTP/1.1 401 Unauthorized
Content-Type: application/json

{
  "error": "Invalid token",
  "path": "/api/blog/posts",
  "timestamp": "2026-07-22T10:00:00Z"
}
```

### 5.2 Permit-All Routes

> `/auth/**` and `/actuator/health` bypass authentication. The Gateway proxies `/auth/**` to Keycloak, which handles its own auth (login page, token endpoints).

---

## 6. Rate Limiting Behavior

### 6.1 Rate Limit Exceeded

```
HTTP/1.1 429 Too Many Requests
Content-Type: application/json
Retry-After: 45

{
  "error": "Rate limit exceeded",
  "path": "/api/blog/posts",
  "retry_after_seconds": 45,
  "timestamp": "2026-07-22T10:00:00Z"
}
```

### 6.2 Rate Limit Headers (All Responses)

| Header | Description |
|--------|-------------|
| `X-RateLimit-Limit` | Maximum requests per window (e.g., `100`) |
| `X-RateLimit-Remaining` | Requests remaining in current window |
| `X-RateLimit-Reset` | Unix timestamp when the window resets |

---

## 7. Error Responses (Gateway-Level)

> Errors that the Gateway itself generates (before reaching a backend service).

| HTTP Status | Condition | Response Body |
|:---:|---|------|
| 301 | HTTP request → redirect to HTTPS | `Location: https://panomete.local/...` |
| 400 | Malformed request | `{"error": "Bad Request", ...}` |
| 401 | Missing/invalid/expired JWT | `{"error": "Authentication required|Token expired|Invalid token", ...}` |
| 403 | Insufficient role for route (e.g., `/eureka/**` without `admin` role) | `{"error": "Access Denied", "required_role": "admin", ...}` |
| 404 | No route matches the path | `{"error": "Not Found", "path": "/nonexistent", ...}` |
| 429 | Rate limit exceeded | `{"error": "Rate limit exceeded", "retry_after_seconds": N, ...}` |
| 502 | Backend service unreachable | `{"error": "Bad Gateway", "route": "blog", ...}` |
| 503 | Backend service DOWN (from Eureka) | `{"error": "Service Unavailable", "service": "cute-gufo", ...}` |
| 504 | Backend service timeout | `{"error": "Gateway Timeout", "route": "blog", ...}` |

---

## 8. Request Logging (Structured JSON)

> Every request through the Gateway is logged to stdout as structured JSON for consumption by Loki/Promtail:

```json
{
  "timestamp": "2026-07-22T10:00:00.123Z",
  "method": "GET",
  "path": "/api/blog/posts",
  "status": 200,
  "latency_ms": 45,
  "route_id": "blog",
  "backend": "cute-gufo",
  "client_ip": "192.168.1.100",
  "user_agent": "Mozilla/5.0 ...",
  "user_id": "a1b2c3d4...",
  "user_roles": ["admin", "user"]
}
```

---

## 9. Path Rewriting

> The Gateway strips the `/api/{service}/` prefix before forwarding to backend services:

| Client Request | Forwarded to Backend |
|---------------|---------------------|
| `GET /api/blog/posts` | `GET /posts` |
| `GET /api/blog/posts/123` | `GET /posts/123` |
| `POST /api/short/links` | `POST /links` |
| `GET /api/todo/todolists` | `GET /todolists` |

> **Note:** This is configured via `StripPrefix=1` filter on `/api/**` routes. Business services define their API paths relative to their root, not including the `/api/{service}` prefix.

---

## 10. Configuration Reference

```yaml
# application.yml — Gateway configuration
server:
  port: 443
  ssl:
    key-store: classpath:keystore.p12
    key-store-password: ${KEYSTORE_PASSWORD}
    key-alias: panomete

spring:
  application:
    name: flowero-gate
  cloud:
    gateway:
      default-filters:
        - AddResponseHeader=X-Gateway, flowero-gate
      routes:
        # Foundation Services
        - id: auth
          uri: lb://flowero-guard
          predicates:
            - Path=/auth/**
        - id: discover
          uri: lb://flowero-discover
          predicates:
            - Path=/eureka/**
          filters:
            - name: RequestRateLimiter
              args:
                redis-rate-limiter.replenishRate: 100

        # Business Services (Phase 2+)
        - id: blog
          uri: lb://cute-gufo
          predicates:
            - Path=/api/blog/**
          filters:
            - StripPrefix=1
        - id: short
          uri: lb://fluffy-mouton
          predicates:
            - Path=/api/short/**
          filters:
            - StripPrefix=1

  security:
    oauth2:
      resourceserver:
        jwt:
          issuer-uri: http://flowero-guard:8080/auth/realms/panomete
          jwk-set-uri: http://flowero-guard:8080/auth/realms/panomete/protocol/openid-connect/certs

logging:
  pattern:
    console: '{"timestamp":"%d{ISO8601}","level":"%p","logger":"%c","message":"%m"}%n'
```

---

## 11. Adding a New Route

> To add a new business service (e.g., a future "Notes" service):

1. **Register the service in Eureka** (service auto-registers with `spring.application.name: notes-service`)
2. **Add a route block** to Gateway's `application.yml`:
   ```yaml
   - id: notes
     uri: lb://notes-service
     predicates:
       - Path=/api/notes/**
     filters:
       - StripPrefix=1
   ```
3. **Restart Gateway** — new route is active

> The service inherits authentication, rate limiting, and request logging automatically — no additional configuration.

---

## Related Documents

| Document | Relationship |
|----------|-------------|
| [[flowero_gate/021_architecture_decision_records]] | Gate-specific ADRs |
| [[panomete_platform/021_architecture_decision_records]] | Platform-level ADRs |
| [[flowero_guard/022_API_specification]] | Auth endpoints Gate proxies to |
| [[flowero_discover/022_API_specification]] | Discovery endpoints Gate resolves from |
| [[flowero_gate/012_user_stories]] | Stories implementing this routing table |

---

> **Template Standard:** Based on SWEBOK v4, OpenAPI Specification 3.0
> **Usage:** This is the platform's public API contract. All external consumers — browsers, mobile apps, API clients — access the platform exclusively through these routes.
