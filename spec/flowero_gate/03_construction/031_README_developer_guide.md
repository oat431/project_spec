---
document_type: README / Developer Guide
version: "0.1"
status: Draft
author: "Dev Persona"
created: "2026-07-24"
last_updated: "2026-07-24"
project_name: "Flowero Gate"
project_id: "PAN-GATE-001"
parent_platform: "Panomete Platform"
classification: "Internal"
tags: [readme, developer-guide, api-gateway, spring-cloud-gateway, panomete]
standard_ref:
  - SWEBOK v4 — Construction
  - 12-Factor App Methodology
---

# README / Developer Guide — Flowero Gate

> **Service:** Flowero Gate (Spring Cloud Gateway)
> **Platform:** Panomete Platform
> **Version:** 0.1 | **Status:** Draft
> **Last Updated:** 2026-07-24

---

## 1. Project Header

```
# Flowero Gate ⚙️

> **API Gateway** for the Panomete Platform — JWT validation, Valkey rate limiting, lb:// route resolution.
>
> Java 25 · Spring Boot 4.1 · Spring Cloud 2025.1 (Oakwood) · Netty
```

| Aspect | Detail |
|--------|--------|
| **Service Type** | Foundation — API Gateway |
| **Technology** | Spring Cloud Gateway (Reactive / WebFlux) |
| **Stack** | Java 25 / Spring Boot 4.1.0 / Spring Cloud 2025.1.2 / Netty |
| **Ports** | 8000 (API proxy) · Management shares main port |
| **Domain** | `api.panomete.com` (via Nginx → Cloudflare) |
| **Database** | None — fully stateless |
| **Cache / Rate Limit** | Valkey 9 (shared, external) |
| **Service Discovery** | Eureka (`lb://` route resolution) |
| **Auth** | JWT validated locally via cached JWKS from Keycloak |
| **Mode** | Internal-only gateway behind Nginx |

## 2. Architecture Diagram

```
                        ┌─ JWT Validation ─── cached JWKS ← Keycloak
                        │
                        ├─ Route Resolution ── lb://cute-gufo       ← Eureka
  flowero-gate :8000 ───┤                     lb://fluffy-mouton
                        │                     lb://tiny-mchwa
                        │
                        ├─ Rate Limiting ──── Valkey 9 (fail-open)
                        │
                        └─ Structured Logging ── JSON → stdout (Phase 2 → Loki)
```

### How Services Use It

```
1. Client hits api.panomete.com/api/blog/posts (via Cloudflare → Nginx → Gate :8000)
2. Gate validates JWT locally (cached JWKS from Guard) → 401 if invalid
3. Gate resolves lb://cute-gufo via Eureka → host:port
4. Gate forwards /posts (strip /api prefix) with X-User-Id, X-User-Roles headers
5. Valkey rate limiting per IP — 100 req/min/route, fail-open if Valkey down
```

## 3. Quick Start

### Prerequisites

- **JDK 25** ([Eclipse Temurin](https://adoptium.net/) recommended)
- **Gradle** (wrapper included — `./gradlew`)
- **Valkey 9** (optional for local dev — rate limiting disabled without it)
- **Keycloak** (optional for local dev — JWT validation hits JWKS endpoint)

### Build & Run

```bash
# Clone
git clone <repo-url>
cd flowerogate

# Build (compile + test + package bootJar)
./gradlew build

# Run locally (dev profile)
./gradlew bootRun
```

Gate starts on **port 8080** (local dev) / **8000** (Docker):
- **Health:** http://localhost:8080/actuator/health
- **Gateway Routes:** http://localhost:8080/actuator/gateway/routes

### Docker

```bash
# Build image
docker build -t panomete/flowerogate:latest .

# Run with compose (joins db-network, connects to Valkey + Eureka + Keycloak)
docker compose up -d
```

### Verify

```bash
# Health check
curl http://localhost:8000/actuator/health
# Expected: {"status":"UP","components":{"discoveryComposite":{"status":"UP"},...}}

# Protected route without JWT → 401
curl -o /dev/null -w '%{http_code}' https://api.panomete.com/api/blog/posts
# Expected: 401

# Eureka dashboard shows FLOWERO-GATE registered
curl http://localhost:8999/eureka/apps | grep flowero-gate
```

## 4. Configuration Reference

Key settings in `application.yaml`:

| Property | Value | Why |
|----------|-------|-----|
| `server.port` | 8080 (dev) / 8000 (prod) | Docker compose overrides to 8000 |
| `spring.application.name` | `flowero-gate` | Eureka registration name |
| `spring.cloud.gateway.routes[].uri` | `lb://{service}` | Eureka-based load-balanced routing |
| `spring.security.oauth2.resourceserver.jwt.issuer-uri` | `https://auth.panomete.com/realms/panomete` | JWT issuer — must match Keycloak realm |
| `spring.data.redis.host` | `local-valkey` (prod) / `localhost` (dev) | Valkey for rate limiting |
| `eureka.client.service-url.defaultZone` | `http://flowero-discover:8999/eureka` | Service registry endpoint |
| `cors.allowed-origins` | `https://*.panomete.com` | Wildcard CORS for all Panomete frontends |
| `app.post-login-redirect-url` | (required, no default) | Where browser redirects after OAuth2 login |
| `management.endpoints.web.exposure.include` | `health,info,prometheus,gateway,metrics` | Actuator endpoints |

### Profiles

| Profile | File | Purpose |
|---------|------|---------|
| `default` | `application.yaml` | Base config, port 8080, dev-oriented |
| `dev` | `application-dev.yaml` | Local development — localhost Keycloak, CORS for localhost |
| `prod` | `application-prod.yaml` | Production — Docker env vars, Eureka enabled, JSON logging |

## 5. API Reference

See the full [[022_API_specification|API Specification]].

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `/actuator/health` | GET | Health check (liveness + readiness + Eureka + Valkey) |
| `/actuator/gateway/routes` | GET | List all configured routes |
| `/actuator/prometheus` | GET | Prometheus metrics scrape endpoint |
| `/api/blog/**` | * | Blog API → `lb://cute-gufo` (auth required) |
| `/api/short/**` | * | URL Shortener API → `lb://fluffy-mouton` (auth required) |
| `/api/todo/**` | * | Todo API → `lb://tiny-mchwa` (auth required) |
| `/fallback/*` | * | Circuit breaker fallback handler |
| `/login/**` | GET | OAuth2 browser login redirect (public) |

## 6. Related Services

| Service | How It Uses Gate | Relationship |
|---------|-----------------|-------------|
| **Flowero Guard (Keycloak)** | Gate validates JWT against Guard's cached JWKS | Depends on (runtime) |
| **Flowero Discover (Eureka)** | Gate resolves `lb://` URIs via Eureka | Depends on (runtime) |
| **Cute Gufo (Blog)** | Receives proxied requests at `/api/blog/**` | Serves |
| **Fluffy Mouton (URL Shortener)** | Receives proxied requests at `/api/short/**` | Serves |
| **Tiny Mchwa (Todo)** | Receives proxied requests at `/api/todo/**` | Serves |
| **Valkey 9** | Shared rate limiting backend | Depends on (runtime, fail-open) |
| **Nginx** | Proxies `api.panomete.com` → Gate :8000 | Sits in front |

## 7. Testing

```bash
# Run all tests
./gradlew test

# Run specific test class
./gradlew test --tests "panomete.flowerogate.gateway.SecurityTests"
```

Tests cover (9 tests total):

| Test Class | Tests | Covers |
|-----------|:---:|--------|
| `FlowerogateApplicationTests` | 1 | Context loads with all beans |
| `RouteTests` | 2 | Fallback 404 endpoint, fallback 503 |
| `SecurityTests` | 6 | Public endpoints (actuator, fallback), 401 without JWT, 401 with invalid/expired token, 200 with valid JWT |

## 8. Design Decisions

| ADR | Decision |
|-----|----------|
| ADR-W001 | Declarative Route Config (YAML) — routes in `application.yml`, version-controlled |
| ADR-W002 | Local JWT Validation — validate JWT locally via cached JWKS, not introspection |
| ADR-W003 | Forward User Claims as Headers — `X-User-Id`, `X-User-Roles` to downstream |
| ADR-W004 | Path-Based Routing — `/api/{service}/**` for business APIs only |
| ADR-W005 | Valkey-Backed Rate Limiting — shared Valkey 9, survives Gate restarts |
| ADR-W006 | Internal-Only Gateway — Gate on :8000 behind Nginx, no edge exposure |
| ADR-W007 | No TLS at Gate — Cloudflare handles TLS, internal is plain HTTP |

Full ADRs: [[021_architecture_decision_records|Architecture Decision Records]]

## 9. Project Structure

```
flowerogate/
├── build.gradle                          # Dependencies + plugins
├── settings.gradle                       # rootProject.name = 'flowerogate'
├── docker-compose.yml                    # Production compose (db-network, Valkey, Eureka)
├── Dockerfile                            # Multi-stage: JDK build → JRE runtime
├── gradlew / gradlew.bat                 # Gradle wrapper
└── src/
    ├── main/
    │   ├── java/panomete/flowerogate/
    │   │   ├── FlowerogateApplication.java       # Entry point
    │   │   ├── config/
    │   │   │   ├── SecurityConfig.java            # OAuth2 Resource Server + OAuth2 Client
    │   │   │   ├── CorsConfig.java                # CORS with wildcard origin patterns
    │   │   │   ├── GatewayConfig.java             # Route config (placeholder)
    │   │   │   ├── RateLimiterConfig.java          # Key resolvers (principal, IP, API key)
    │   │   │   ├── ResilientRedisRateLimiter.java  # Fail-open Valkey rate limiter
    │   │   │   ├── CircuitBreakerConfig.java       # Resilience4j config
    │   │   │   └── ObservabilityConfig.java        # Micrometer common tags
    │   │   ├── filter/
    │   │   │   ├── JwtClaimHeaderFilter.java      # Extract JWT claims → downstream headers
    │   │   │   ├── TraceIdFilter.java              # W3C traceparent propagation
    │   │   │   ├── RequestLoggingFilter.java       # Structured JSON request logging
    │   │   │   ├── RateLimitResponseFilter.java    # Standardized 429 JSON body
    │   │   │   ├── OAuth2RedirectParamFilter.java  # Save post-login redirect URL
    │   │   │   └── SensitiveDataMasker.java        # Mask secrets in error logs
    │   │   ├── controller/
    │   │   │   ├── FallbackController.java         # 404 + circuit breaker fallbacks
    │   │   │   └── RateLimitAdminController.java   # Rate limit management endpoints
    │   │   └── exception/
    │   │       └── GatewayExceptionHandler.java    # Global error handler
    │   └── resources/
    │       ├── application.yaml                    # Base config
    │       ├── application-dev.yaml                # Dev overrides
    │       └── application-prod.yaml               # Prod overrides
    └── test/
        ├── java/panomete/flowerogate/
        │   ├── FlowerogateApplicationTests.java
        │   ├── gateway/
        │   │   ├── RouteTests.java
        │   │   ├── SecurityTests.java
        │   │   └── TestSecurityConfig.java
        │   └── support/
        │       └── JwtTestHelper.java
        └── resources/
            └── application.yaml                    # Test config
```

## 10. Reference

- [Spring Cloud Gateway Reference](https://docs.spring.io/spring-cloud-gateway/reference/)
- [Spring Security OAuth2 Resource Server](https://docs.spring.io/spring-security/reference/reactive/oauth2/resource-server/)
- [[025_software_architecture_document|Platform SAD]]
- [[029_architecture_overview|Architecture Overview]]
- [[flowero_discover/031_README_developer_guide|Flowero Discover]]
- [[flowero_guard/031_README_developer_guide|Flowero Guard]]

---

## Related Documents

| Document | Relationship |
|----------|-------------|
| [[022_API_specification]] | Route table + auth behavior |
| [[021_architecture_decision_records]] | Gate-specific ADRs (7 decisions) |
| [[032_build_scripts]] | Build pipeline + Gradle commands |
| [[033_dependency_manifest]] | Full dependency inventory |
| [[035_coding_standards_development]] | Java coding standards |
| [[052_deployment_plan]] | Deployment procedure |
