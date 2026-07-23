---
document_type: README / Developer Guide
version: "0.1"
status: Draft
author: "Dev Persona"
created: "2026-07-23"
last_updated: "2026-07-23"
project_name: "Flowero Discover"
project_id: "PAN-DISCOVER-001"
parent_platform: "Panomete Platform"
classification: "Internal"
tags: [readme, developer-guide, eureka, service-discovery, panomete]
standard_ref:
  - SWEBOK v4 — Construction
  - 12-Factor App Methodology
---

# README / Developer Guide — Flowero Discover

> **Service:** Flowero Discover (Spring Cloud Netflix Eureka)
> **Platform:** Panomete Platform
> **Version:** 0.1 | **Status:** Draft
> **Last Updated:** 2026-07-23

---

## 1. Purpose

> The README is the **front door** for developers working on or consuming Flowero Discover. Clone → build → run → verify in under 5 minutes.

## 2. Project Header

```markdown
# Flowero Discover 🟢

> **Eureka Service Registry** for the Panomete Platform.
>
> Java 25 · Spring Boot 4.1 · Spring Cloud 2025.1 (Oakwood)
```

| Aspect | Detail |
|--------|--------|
| **Service Type** | Foundation — Service Registry |
| **Technology** | Spring Cloud Netflix Eureka Server |
| **Stack** | Java 25 / Spring Boot 4.1.0 / Spring Cloud 2025.1.2 |
| **Ports** | 8999 (REST API) · 3999 (Dashboard, via Docker port mapping) |
| **Domain** | `discovery.panomete.com` (via Nginx) |
| **Database** | None — fully in-memory |
| **Mode** | Standalone (single node) |
| **Dependencies** | None (no database, no external services) |

## 3. Architecture Diagram

```
                    ┌─ REST API ─── :8999 ─── service registration
                    │                          heartbeat (PUT /eureka/apps/...)
  flowero-discover ─┤                          discovery queries (GET /eureka/apps)
                    │
                    └─ Dashboard ─── :3999 ─── discovery.panomete.com
                                               (Nginx proxy → :3999 → :8999 dashboard)
```

### How Services Use It

```
1. Service boots → registers with Eureka (name, host, port, health URL)
2. Service sends heartbeat every 30s (PUT /eureka/apps/{app}/{instance})
3. Gate resolves lb://cute-gufo → queries Eureka → returns host:port
4. Dead instance: heartbeats fail → evicted after 90s
```

## 4. Quick Start

### Prerequisites

- **JDK 25** ([Eclipse Temurin](https://adoptium.net/) recommended)
- **Gradle** (wrapper included — `./gradlew`)

### Build & Run

```bash
# Clone
git clone <repo-url>
cd flowerodiscovery

# Build (compile + test + package bootJar)
./gradlew build

# Run locally
./gradlew bootRun
```

Eureka starts on **port 8999**:
- **Dashboard:** http://localhost:8999/
- **Health:** http://localhost:8999/actuator/health
- **Registry API:** http://localhost:8999/eureka/apps

### Verify

```bash
# Health check
curl http://localhost:8999/actuator/health
# Expected: {"status":"UP"}

# Dashboard reachable
curl -s http://localhost:8999/ | grep -o "Instances currently registered with Eureka"
# Expected: "Instances currently registered with Eureka"
```

### Docker

```bash
# Build image
docker build -t flowero-discover .

# Run container (both ports)
docker run -p 8999:8999 -p 3999:8999 flowero-discover
```

## 5. Configuration Reference

Key settings in `application.yaml`:

| Property | Value | Why |
|----------|-------|-----|
| `server.port` | `8999` | REST API port |
| `eureka.instance.hostname` | `flowero-discover` | Logical name on Docker network |
| `eureka.client.register-with-eureka` | `false` | Standalone — don't self-register |
| `eureka.client.fetch-registry` | `false` | Standalone — don't fetch from self |
| `eureka.server.enable-self-preservation` | `true` | Keeps registry intact during network hiccups |
| `eureka.server.eviction-interval-timer-in-ms` | `5000` | Evict dead instances every 5s |
| `eureka.server.renewal-percent-threshold` | `0.85` | Self-preservation triggers below 85% heartbeats |
| `eureka.server.wait-time-in-ms-when-sync-empty` | `0` | No delay for empty registry |
| `management.endpoints.web.exposure.include` | `health,info` | Actuator endpoints exposed |

## 6. API Reference

See the full [[022_API_specification|API Specification]].

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `/eureka/apps` | `GET` | List all registered applications |
| `/eureka/apps/{name}` | `GET` | Get instances for a specific service |
| `/eureka/apps/{name}` | `POST` | Register a new instance |
| `/eureka/apps/{name}/{instance}` | `PUT` | Heartbeat / renew lease |
| `/eureka/apps/{name}/{instance}` | `DELETE` | Deregister an instance |
| `/actuator/health` | `GET` | Health check (`{"status":"UP"}`) |
| `/` | `GET` | Eureka dashboard (HTML) |

**Format:** Use `Accept: application/json` — Eureka defaults to XML.

## 7. Related Services

| Service | How It Uses Discover |
|---------|---------------------|
| **Flowero Gate** | Resolves `lb://` routes to business services |
| **Flowero Guard** | Registers health for monitoring |
| **All business services** | Register on startup, discover peers by name |

## 8. Testing

```bash
# Run all tests
./gradlew test

# Run with detailed output
./gradlew test --info
```

Tests cover:
- ✅ Context loads with Eureka server enabled
- ✅ Dashboard is reachable at `/`
- ✅ Actuator health returns `{"status":"UP"}`
- ✅ `/eureka/apps` returns valid response (XML + JSON)
- ✅ No self-registration (standalone mode verified)

**Test results (2026-07-23):** 6 tests, 0 failures, 0 errors.

## 9. Design Decisions

| ADR | Decision |
|-----|----------|
| ADR-D001 | Standalone single-node Eureka |
| ADR-D003 | Dashboard via Nginx at `discovery.panomete.com` |
| ADR-D004 | Self-preservation enabled for homelab stability |
| ADR-D005 | Dual ports: 8999 (API) + 3999 (dashboard via Docker mapping) |

Full ADRs: [[021_architecture_decision_records]]

## 10. Project Structure

```
flowerodiscovery/
├── build.gradle                          # Dependencies + build config
├── settings.gradle                       # Project name
├── Dockerfile                            # Multi-stage build (JDK → JRE)
├── docker-compose.fragment.yml           # Compose entry for platform
├── .dockerignore
├── gradlew / gradlew.bat                 # Gradle wrapper (committed)
├── gradle/wrapper/
├── src/
│   ├── main/
│   │   ├── java/panomete/flowerodiscovery/
│   │   │   └── FlowerodiscoveryApplication.java   # @SpringBootApplication + @EnableEurekaServer
│   │   └── resources/
│   │       └── application.yaml                   # Eureka standalone config
│   └── test/
│       └── java/panomete/flowerodiscovery/
│           └── FlowerodiscoveryApplicationTests.java  # 6 integration tests
└── README.md
```

---

## Related Documents

| Document | Relationship |
|----------|-------------|
| [[011_business_objective]] | Service objectives (OBJ-DISCOVER-01 through 03) |
| [[012_user_stories]] | 4 user stories (US-101 through US-104) |
| [[013_acceptance_criteria]] | 16 BDD acceptance criteria |
| [[021_architecture_decision_records]] | 5 service-level ADRs |
| [[022_API_specification]] | Eureka REST API endpoints |
| [[032_build_scripts]] | Gradle + Docker build pipeline |
| [[033_dependency_manifest]] | Dependency inventory |

---

> **Template Standard:** Based on SWEBOK v4, 12-Factor App
> **Usage:** The README is the front door. If a developer can't set up in 5 minutes using these instructions, the README needs work.
