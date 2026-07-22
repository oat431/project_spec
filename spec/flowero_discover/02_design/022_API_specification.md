---
document_type: API Specification
version: "0.2"
status: Draft
author: "Dev / SA Persona"
created: "2026-07-22"
last_updated: "2026-07-22"
project_name: "Flowero Discover"
project_id: "PAN-DISCOVER-001"
parent_platform: "Panomete Platform"
tech_lead: "Dev / SA Persona"
classification: "Internal"
tags: [api-specification, eureka, service-discovery, rest, panomete]
standard_ref:
  - Spring Cloud Netflix Eureka REST API
---

# API Specification — Flowero Discover

> **Service:** Flowero Discover (Spring Cloud Netflix Eureka)
> **Platform:** Panomete Platform
> **Version:** 0.2 | **Status:** Draft — Updated per Design Review 2026-07-22
> **Last Updated:** 2026-07-22

---

## 1. Purpose

> Flowero Discover exposes Eureka's REST API for service registration and discovery. Dashboard is accessible at `discovery.panomete.com` via Nginx. Services register on port 8999.

---

## 2. API Overview

| Field | Detail |
|-------|--------|
| **BE (REST API)** | `http://flowero-discover:8999` |
| **FE (Dashboard)** | `https://discovery.panomete.com` (via Nginx → :3999) |
| **Format** | JSON (use `Accept: application/json` — Eureka defaults to XML) |
| **Auth** | None (trusted Docker network) |

---

## 3. Service Registration

### POST /eureka/apps/{app-name} — Register Instance

**Request (JSON):**
```json
{
  "instance": {
    "instanceId": "cute-gufo:8005",
    "hostName": "cute-gufo",
    "app": "CUTE-GUFO",
    "ipAddr": "172.18.0.5",
    "status": "UP",
    "port": {"$": 8005, "@enabled": "true"},
    "healthCheckUrl": "http://cute-gufo:8005/actuator/health",
    "dataCenterInfo": {"@class": "com.netflix.appinfo.InstanceInfo$DefaultDataCenterInfo", "name": "MyOwn"}
  }
}
```

**Response (204):** Registration accepted.

> Spring Boot services auto-register via `spring-cloud-starter-netflix-eureka-client` — no manual API call needed.

### DELETE /eureka/apps/{app-name}/{instance-id} — Deregister

### PUT /eureka/apps/{app-name}/{instance-id} — Heartbeat

> Renew lease every 30s. **Response 200** = OK. **Response 404** = must re-register.

---

## 4. Service Discovery

### GET /eureka/apps — All Applications

**Response (200):**
```json
{
  "applications": {
    "application": [
      {
        "name": "FLOWERO-GUARD",
        "instance": [{
          "instanceId": "flowero-guard:8001",
          "hostName": "flowero-guard",
          "status": "UP",
          "port": {"$": 8001, "@enabled": "true"}
        }]
      },
      {
        "name": "CUTE-GUFO",
        "instance": [{
          "instanceId": "cute-gufo:8005",
          "status": "UP",
          "port": {"$": 8005, "@enabled": "true"}
        }]
      }
    ]
  }
}
```

### GET /eureka/apps/{app-name} — Specific Application

### GET /eureka/apps/{app-name}/{instance-id} — Specific Instance

---

## 5. How Gate Consumes Discover

> Gate uses `lb://` URIs. Eureka client resolves them locally — no REST API call per request.

```yaml
# Gate application.yml
routes:
  - id: blog
    uri: lb://cute-gufo   # ← Eureka resolves to actual host:port
    predicates:
      - Path=/api/blog/**
```

1. Gate's Eureka client fetches the full registry at startup + periodic delta updates
2. Per-request: `lb://` resolution from the local cached registry — zero network call
3. On Eureka failure: Gate uses cached registry — no cascading failure

---

## 6. Eureka Dashboard (HTML)

> Served at `https://discovery.panomete.com` via Nginx → Eureka :3999.

| Endpoint | Description |
|----------|-------------|
| `GET /` | Dashboard HTML |
| `GET /eureka/css/*` | Stylesheets |
| `GET /eureka/js/*` | JavaScript |

---

## 7. Configuration Reference

```yaml
server:
  port: 8999  # REST API

eureka:
  instance:
    hostname: flowero-discover
    nonSecurePort: 8999
  client:
    register-with-eureka: false
    fetch-registry: false
  server:
    enable-self-preservation: true
    eviction-interval-timer-in-ms: 5000

# Dashboard port (separate embedded server or Eureka config)
# Served on :3999 via Nginx proxy
```

---

## Related Documents

| Document | Relationship |
|----------|-------------|
| [[flowero_discover/021_architecture_decision_records]] | Discover-specific ADRs |
| [[flowero_gate/022_API_specification]] | Gate resolves routes via these endpoints |
