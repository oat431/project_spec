---
document_type: API Specification
version: "0.2"
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
> **Version:** 0.2 | **Status:** Draft — Updated per Design Review 2026-07-22
> **Last Updated:** 2026-07-22

---

## 1. Purpose

> Flowero Gate is the **internal API gateway** for business services. It sits behind Nginx at `api.panomete.com:8000`. It routes business API traffic, validates JWT locally, enforces Valkey-backed rate limiting, and emits structured JSON logs. Foundation services (Guard, Discover) are NOT routed through Gate — they have their own subdomains through Nginx.

---

## 2. API Overview

| Field | Detail |
|-------|--------|
| **Internal Address** | `http://flowero-gate:8000` |
| **External Address** | `https://api.panomete.com` (via Nginx → Cloudflare) |
| **Protocol** | HTTP (internal). HTTPS is handled by Cloudflare. |
| **Format** | JSON (errors, health), pass-through for proxied services |
| **Auth** | Bearer JWT — validated locally against Keycloak JWKS (cached) |

---

## 3. Route Table (Business APIs Only)

> Gate routes business API traffic only. Foundation services are routed directly by Nginx.

| Route ID | Path | Backend (`lb://`) | Auth | Rate Limit |
|----------|------|------------------|:---:|:---:|
| `blog` | `/api/blog/**` | `lb://cute-gufo` | Yes | 100/min |
| `short` | `/api/short/**` | `lb://fluffy-mouton` | Yes | 100/min |
| `todo` | `/api/todo/**` | `lb://tiny-mchwa` | Yes | 100/min |
| `ledger` | `/api/ledger/**` | `lb://big-schwein` | Yes | 100/min |
| `recipe` | `/api/recipe/**` | `lb://shy-ardilla` | Yes | 100/min |
| `hora` | `/api/hora/**` | `lb://white-jelen` | Yes | 100/min |

**Not routed by Gate** (Nginx handles these):
- `auth.panomete.com` → Flowero Guard :8001
- `discovery.panomete.com` → Flowero Discover :3999

---

## 4. Gateway Endpoints

### GET /actuator/health — Health Check

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
          "details": {"services": ["FLOWERO-GUARD", "CUTE-GUFO"]}
        }
      }
    },
    "redisReactiveHealthIndicator": {
      "status": "UP",
      "details": {"version": "9.0"}
    }
  }
}
```

---

## 5. Authentication Behavior

### Without Token → 401

```
HTTP/1.1 401 Unauthorized
{"error": "Authentication required", "path": "/api/blog/posts"}
```

### With Valid JWT → Forwarded with Claim Headers

| Header | Source (JWT claim) | Example |
|--------|-------------------|---------|
| `X-User-Id` | `sub` | `a1b2c3d4-...` |
| `X-User-Name` | `preferred_username` | `alice` |
| `X-User-Roles` | `realm_access.roles` | `admin,user` |

### Expired / Invalid Token → 401

```
{"error": "Token expired"}  or  {"error": "Invalid token"}
```

---

## 6. Rate Limiting (Valkey-Backed)

### Rate Limit Exceeded → 429

```
HTTP/1.1 429 Too Many Requests
Retry-After: 45
{"error": "Rate limit exceeded", "retry_after_seconds": 45}
```

### Response Headers (All Requests)

| Header | Description |
|--------|-------------|
| `X-RateLimit-Limit` | Max requests per window |
| `X-RateLimit-Remaining` | Remaining in current window |
| `X-RateLimit-Reset` | Unix timestamp of reset |

---

## 7. Error Responses (Gateway-Level)

| HTTP | Condition | Body |
|:---:|---|------|
| 401 | Missing/invalid/expired JWT | `{"error": "..."}` |
| 403 | Insufficient role | `{"error": "Access Denied", "required_role": "admin"}` |
| 404 | No route match | `{"error": "Not Found", "path": "/xyz"}` |
| 429 | Rate limited | `{"error": "Rate limit exceeded", "retry_after_seconds": N}` |
| 502 | Backend unreachable | `{"error": "Bad Gateway", "route": "..."}` |
| 503 | Backend DOWN (Eureka evicted) | `{"error": "Service Unavailable", "service": "..."}` |

---

## 8. Request Logging (Structured JSON)

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
  "user_id": "a1b2c3d4...",
  "user_roles": ["admin", "user"]
}
```

---

## 9. Path Rewriting

> Gate strips `/api/{service}/` prefix before forwarding:

| Client Request | Forwarded to Backend |
|---------------|---------------------|
| `GET /api/blog/posts` | `GET /posts` |
| `POST /api/short/links` | `POST /links` |
| `GET /api/todo/todolists` | `GET /todolists` |

> Configured via `StripPrefix=1` filter on all `/api/**` routes.

---

## 10. Configuration Reference

```yaml
server:
  port: 8000

spring:
  application:
    name: flowero-gate
  cloud:
    gateway:
      routes:
        - id: blog
          uri: lb://cute-gufo
          predicates: [Path=/api/blog/**]
          filters:
            - StripPrefix=1
            - name: RequestRateLimiter
              args:
                redis-rate-limiter.replenishRate: 100
                redis-rate-limiter.burstCapacity: 200
  security:
    oauth2:
      resourceserver:
        jwt:
          issuer-uri: https://auth.panomete.com/realms/panomete
          jwk-set-uri: https://auth.panomete.com/realms/panomete/protocol/openid-connect/certs

  data:
    redis:
      host: valkey  # or host.docker.internal
      port: 6379

logging:
  pattern:
    console: '{"timestamp":"%d{ISO8601}","level":"%p","message":"%m"}%n'
```

---

## 11. Adding a New Service

1. Register the service in Eureka (`spring.application.name`)
2. Add a route block to Gate's `application.yml`
3. Register an OAuth2 client in Keycloak
4. Restart Gate

> The new service inherits JWT validation, rate limiting, and structured logging automatically.

---

## Related Documents

| Document | Relationship |
|----------|-------------|
| [[flowero_gate/021_architecture_decision_records]] | Gate-specific ADRs |
| [[panomete_platform/021_architecture_decision_records]] | Platform-level ADRs |
| [[flowero_guard/022_API_specification]] | JWKS endpoint Gate uses for auth |
| [[flowero_discover/022_API_specification]] | Discovery endpoints Gate uses for routing |
