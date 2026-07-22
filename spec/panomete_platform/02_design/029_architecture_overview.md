---
document_type: Architecture Overview (HLD)
version: "0.1"
status: Draft
author: "Dev / SA Persona"
created: "2026-07-22"
last_updated: "2026-07-22"
project_name: "Panomete Platform"
project_id: "PAN-PLAT-001"
architect: "Dev / SA Persona"
classification: "Internal"
tags: [architecture-overview, hld, high-level-design, panomete, microservices]
standard_ref:
  - SWEBOK v4 — Design
  - ISO/IEC/IEEE 42010 — Architecture Description
---

# Architecture Overview — Panomete Platform

> **Project:** Panomete Platform
> **Version:** 0.1 | **Status:** Draft
> **Last Updated:** 2026-07-22

---

## 1. Purpose

> A single-page overview of the Panomete Platform architecture. This is the "map" — point new developers and portfolio reviewers here first. For detailed decisions, see the [[025_software_architecture_document|SAD]] and [[021_architecture_decision_records|ADRs]].

---

## 2. System Context (C4 — Level 1)

```mermaid
flowchart TB
    User(["👤 User<br>Browser / API Client"])

    subgraph Platform["Panomete Platform"]
        Gate["Flowero Gate<br>API Gateway<br>:80 / :443"]
        Guard["Flowero Guard<br>Keycloak IAM"]
        Discover["Flowero Discover<br>Service Registry"]

        subgraph Future["Future Business Services"]
            Blog["Cute Gufo 🦉"]
            URL["Fluffy Mouton 🐑"]
            Todo["Tiny Mchwa 🐜"]
        end
    end

    User -->|"HTTPS"| Gate
    Gate --> Guard
    Gate --> Discover
    Gate --> Future
    Guard --> Discover
    Future --> Discover
```

---

## 3. Container View (C4 — Level 2)

```mermaid
flowchart TB
    subgraph Host["🖥️ Homelab Host — Docker Compose"]
        subgraph Net["panomete-net (bridge)"]
            GATE["🐘 flowero-gate<br>Spring Cloud Gateway<br>:80, :443<br>Java 21 / 512MB"]

            KC["🐘 flowero-guard<br>Keycloak IAM<br>:8080<br>Java / 1GB"]

            KC_DB["🗄️ guard-db<br>PostgreSQL 15<br>:5432<br>512MB shared_buffers"]

            EUR["🐘 flowero-discover<br>Eureka Server<br>:8761<br>Java 21 / 256MB"]
        end
    end

    GATE -->|"HTTP :8080"| KC
    GATE -->|"HTTP :8761"| EUR
    KC -->|"TCP :5432"| KC_DB
    KC -.->|"health → register"| EUR
    GATE -.->|"health → register"| EUR
    GATE -.->|"resolve lb:// URIs"| EUR
    GATE -.->|"validate JWT (JWKS)"| KC
```

---

## 4. Request Flow — Authenticated API Call

```mermaid
sequenceDiagram
    participant U as 👤 Browser
    participant G as Flowero Gate
    participant K as Flowero Guard
    participant E as Flowero Discover
    participant S as Business Service

    U->>G: GET /api/blog/posts<br>Authorization: Bearer {JWT}
    G->>G: 1. Rate limit check
    G->>G: 2. JWT signature validation<br>(local, via cached JWKS from Guard)
    G->>G: 3. Extract claims → X-User-Id, X-User-Roles
    G->>E: 4. Resolve lb://cute-gufo
    E-->>G: host:port
    G->>S: 5. Forward: GET /posts<br>+ X-User-Id, X-User-Roles
    S-->>G: 200 + JSON
    G->>G: 6. Log (structured JSON)
    G-->>U: 200 + JSON
```

---

## 5. Technology Stack (At a Glance)

| Layer | Technology | Why |
|-------|-----------|-----|
| **API Gateway** | Spring Cloud Gateway | Reactive, Netty-based. Native Spring Security + Eureka integration. |
| **Identity Provider** | Keycloak | Production-grade OSS IAM. OAuth2/OIDC. Realm-as-code (JSON). |
| **Service Discovery** | Spring Cloud Netflix Eureka | Simplest path with Spring Boot. Auto-registration. Dashboard. |
| **Foundation Runtime** | Java 21 / Spring Boot 3.x | Mature ecosystem. Spring Cloud provides GW + Security + Discovery out of the box. |
| **Database** | PostgreSQL 15 | Keycloak's recommended production DB. ACID. Already in homelab stack. |
| **Deployment** | Docker Compose | One command: `docker compose up`. Design for k3s portability. |
| **Observability** | Spring Boot Actuator + Loki + Prometheus + Grafana | Built-in health/metrics. Lightweight log aggregation. |

---

## 6. Key Architecture Decisions

| # | Decision | Why (one sentence) |
|---|---------|-------------------|
| 1 | Keycloak, not custom auth | Eliminates auth code from every service; industry standard OAuth2/OIDC |
| 2 | Spring Cloud Gateway | Native Spring ecosystem integration; reactive and efficient |
| 3 | Eureka discovery | Services auto-register on boot; Gateway resolves routes dynamically |
| 4 | JWT + local validation | Zero-latency auth — no network call per request |
| 5 | Gateway-side auth | Services receive validated claims as headers; zero auth code |
| 6 | Docker Compose → k3s | Compose for MVP speed; designed for K8s from day one |

> Full rationale: [[021_architecture_decision_records]]

---

## 7. Security Model

```
🌐 External: HTTPS + JWT Bearer token → Gate
🔒 Perimeter: Gate validates JWT (local JWKS), enforces rate limits
🔑 Internal: Trusted Docker network. Gate forwards claims as headers.
🗄️ Guard DB: PostgreSQL on private network, no external port.
```

| Concern | Where | How |
|---------|-------|-----|
| Authentication | Gate | JWT signature validation (cached JWKS from Guard) |
| Authorization | Gate + Service | Gate: route-level (hasRole). Service: `@PreAuthorize` on endpoints |
| Rate Limiting | Gate | Per-IP, per-route (100 req/min default) |
| TLS | Gate | Terminates TLS 1.2+ at :443; internal HTTP |
| Secrets | Docker Compose `.env` | Never committed to git |

---

## 8. Startup Sequence

```
1. guard-db      (PostgreSQL)         ─── health: pg_isready
2. flowero-guard  (Keycloak)           ─── depends: guard-db healthy
3. flowero-discover (Eureka)           ─── standalone, no dependencies
4. flowero-gate   (Spring Cloud GW)    ─── depends: Guard + Discover healthy
```

---

## 9. Port Map

| Service | Internal Port | External Access | Notes |
|---------|:---:|---|-------|
| flowero-gate | 80, 443 | ✅ `https://panomete.local` | Only ports exposed to host |
| flowero-guard | 8080 | Via Gate: `/auth/**` | Internal Docker network only |
| guard-db | 5432 | None | Internal only |
| flowero-discover | 8761 | Via Gate: `/eureka/**` | Internal only |

---

## 10. What's NOT Shown Here

- **Observability stack** (Loki + Prometheus + Grafana) — Phase 1.5, see SAD §2.2
- **Business services** (Cute Gufo, Fluffy Mouton, etc.) — Phase 2+
- **CI/CD pipeline** — See DevOps persona docs (05_devops)
- **Multi-node / K8s deployment** — Future phase, see ADR-006

---

## Related Documents

| Document | Relationship |
|----------|-------------|
| [[025_software_architecture_document]] | Full architecture detail — component design, quality attributes, deployment |
| [[021_architecture_decision_records]] | Why we made each decision |
| [[README]] | Platform overview and service catalog |
| [[flowero_gate/022_API_specification]] | Gateway routing table (the platform's "API") |

---

> **Template Standard:** Based on SWEBOK v4, ISO/IEC/IEEE 42010
> **Usage:** Print this. Tape it to the wall. It's the map. When lost, return here.
