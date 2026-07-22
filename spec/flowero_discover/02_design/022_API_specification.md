---
document_type: API Specification
version: "0.1"
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
  - SWEBOK v4 — Design
  - Spring Cloud Netflix Eureka REST API
---

# API Specification — Flowero Discover

> **Service:** Flowero Discover (Spring Cloud Netflix Eureka)
> **Platform:** Panomete Platform
> **Version:** 0.1 | **Status:** Draft
> **Last Updated:** 2026-07-22

---

## 1. Purpose

> Flowero Discover exposes Eureka's REST API for service registration, discovery, and health tracking. This document catalogues the endpoints that services and Flowero Gate consume. The Eureka dashboard is accessed through Gate at `/eureka/`.

---

## 2. API Overview

| Field | Detail |
|-------|--------|
| **Base URL (internal)** | `http://flowero-discover:8761` |
| **Base URL (external)** | `https://panomete.local/eureka` (via Flowero Gate) |
| **Protocol** | HTTP (internal Docker network) / HTTPS (external via Gate) |
| **Format** | XML (default), JSON (via `Accept: application/json` header) |
| **Authentication** | None (internal Docker network is trusted; Gate enforces auth for dashboard) |
| **Note** | ⚠️ Eureka's REST API uses XML by default. **Always include `Accept: application/json`** to get JSON responses. |

---

## 3. Service Registration

### 3.1 Register a Service Instance

#### POST /eureka/apps/{application-name}

| Field | Detail |
|-------|--------|
| **Description** | Register a new service instance with Eureka |
| **Auth** | None (internal network) |

**Request (JSON):**
```json
{
  "instance": {
    "instanceId": "cute-gufo:8081",
    "hostName": "cute-gufo",
    "app": "CUTE-GUFO",
    "ipAddr": "172.18.0.5",
    "vipAddress": "cute-gufo",
    "status": "UP",
    "port": {
      "$": 8081,
      "@enabled": "true"
    },
    "healthCheckUrl": "http://cute-gufo:8081/actuator/health",
    "statusPageUrl": "http://cute-gufo:8081/actuator/info",
    "dataCenterInfo": {
      "@class": "com.netflix.appinfo.InstanceInfo$DefaultDataCenterInfo",
      "name": "MyOwn"
    },
    "metadata": {
      "startup": "2026-07-22T10:00:00Z"
    }
  }
}
```

**Response (204):** No content — registration accepted.

**Note:** Spring Boot services with `spring-cloud-starter-netflix-eureka-client` register automatically — they do NOT call this endpoint manually. Non-Spring services (Go, TS) use this endpoint.

---

### 3.2 Deregister a Service Instance

#### DELETE /eureka/apps/{application-name}/{instance-id}

| Field | Detail |
|-------|--------|
| **Description** | Remove a service instance from the registry (graceful shutdown) |
| **Auth** | None (internal network) |

**Response (200):** Instance removed.

---

### 3.3 Send Heartbeat

#### PUT /eureka/apps/{application-name}/{instance-id}

| Field | Detail |
|-------|--------|
| **Description** | Renew the service instance's lease (heartbeat). Must be sent every 30 seconds. |
| **Auth** | None (internal network) |

**Response (200):** Heartbeat accepted, lease renewed.

**Response (404):** Instance not found — must re-register.

---

## 4. Service Discovery

### 4.1 List All Registered Applications

#### GET /eureka/apps

| Field | Detail |
|-------|--------|
| **Description** | Get all registered applications and their instances |
| **Auth** | None (internal network) |

**Response (200) — JSON:**
```json
{
  "applications": {
    "versions__delta": "1",
    "apps__hashcode": "UPDATED",
    "application": [
      {
        "name": "FLOWER-GUARD",
        "instance": [
          {
            "instanceId": "flowero-guard:8080",
            "hostName": "flowero-guard",
            "app": "FLOWERO-GUARD",
            "ipAddr": "172.18.0.3",
            "status": "UP",
            "port": { "$": 8080, "@enabled": "true" },
            "healthCheckUrl": "http://flowero-guard:8080/health/ready"
          }
        ]
      },
      {
        "name": "FLOWERO-DISCOVER",
        "instance": [
          {
            "instanceId": "flowero-discover:8761",
            "hostName": "flowero-discover",
            "status": "UP",
            "port": { "$": 8761, "@enabled": "true" }
          }
        ]
      }
    ]
  }
}
```

---

### 4.2 Get a Specific Application

#### GET /eureka/apps/{application-name}

| Field | Detail |
|-------|--------|
| **Description** | Get all instances of a specific application |
| **Auth** | None (internal network) |

**Query Parameters:**

| Param | Description |
|-------|-------------|
| `{application-name}` | Application name in UPPERCASE (e.g., `CUTE-GUFO`) |

**Response (200):** Same structure as above, but for a single application.

**Response (404):** Application not found.

---

### 4.3 Get a Specific Instance

#### GET /eureka/apps/{application-name}/{instance-id}

| Field | Detail |
|-------|--------|
| **Description** | Get details of a specific instance |
| **Auth** | None (internal network) |

**Response (200):** Single instance object.

---

## 5. Delta Updates (Optimization)

### GET /eureka/apps/delta

| Field | Detail |
|-------|--------|
| **Description** | Get only the changes since the last fetch (optimized polling). Used by Eureka clients to reduce bandwidth. |
| **Auth** | None (internal network) |

**Response (200):** Delta of applications with `actionType` (`ADDED`, `MODIFIED`, `DELETED`).

> **Note:** Spring Cloud's `DiscoveryClient` uses delta updates automatically. Non-Spring clients can use this for efficient polling.

---

## 6. How Flowero Gate Consumes Discover

> The Gateway does NOT call Eureka's REST API directly for each request. Instead:

```yaml
# In Gate's application.yml, routes use lb:// prefix:
spring:
  cloud:
    gateway:
      routes:
        - id: blog
          uri: lb://cute-gufo      # ← "lb://" triggers Eureka resolution
          predicates:
            - Path=/api/blog/**
```

1. **At startup:** Gate's `ReactiveLoadBalancer` discovers all instances via Eureka client
2. **Per request:** Gate resolves `lb://cute-gufo` from its local, cached registry (refreshed periodically from Eureka)
3. **On instance change:** Eureka client receives delta updates and updates the local cache
4. **On Eureka failure:** Gate uses the last cached instance list — no cascading failure

---

## 7. Eureka Dashboard (HTML — Served Through Gate)

> The Eureka dashboard is an HTML/JS frontend served by Eureka at `/`. When accessed through Gate at `https://panomete.local/eureka/`, Gate proxies all requests including static assets.

| Endpoint | Description |
|----------|-------------|
| `GET /` | Eureka dashboard HTML |
| `GET /eureka/css/*` | Dashboard CSS |
| `GET /eureka/js/*` | Dashboard JavaScript |
| `GET /lastn` | Last N events (dashboard polling) |

---

## 8. Configuration Reference

> Key Eureka server configuration (`application.yml`):

```yaml
server:
  port: 8761

eureka:
  instance:
    hostname: flowero-discover
  client:
    register-with-eureka: false   # Standalone mode
    fetch-registry: false          # Standalone mode
    service-url:
      defaultZone: http://${eureka.instance.hostname}:${server.port}/eureka/
  server:
    enable-self-preservation: true
    renewal-percent-threshold: 0.85
    eviction-interval-timer-in-ms: 5000
```

---

## 9. Error Codes

| HTTP Status | Description |
|:---:|-------------|
| 204 | Registration accepted (POST /apps) |
| 200 | Heartbeat accepted (PUT /apps/{app}/{id}) |
| 404 | Instance not found — must re-register |

---

## Related Documents

| Document | Relationship |
|----------|-------------|
| [[flowero_discover/021_architecture_decision_records]] | Discover-specific ADRs |
| [[panomete_platform/021_architecture_decision_records]] | Platform-level ADRs (ADR-003: Eureka) |
| [[flowero_gate/022_API_specification]] | Gate resolves routes through these endpoints |
| [[flowero_discover/012_user_stories]] | Stories implementing these endpoints |

---

> **Template Standard:** Based on SWEBOK v4, Spring Cloud Netflix Eureka
> **Usage:** This document is the contract for service registration and discovery. Spring Boot services get auto-registration for free. Non-Spring services implement the registration endpoints documented here.
