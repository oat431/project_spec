---
document_type: ADR (Architecture Decision Records)
version: "0.1"
status: Active
author: "Dev / SA Persona"
created: "2026-07-22"
last_updated: "2026-07-22"
project_name: "Panomete Platform"
project_id: "PAN-PLAT-001"
architect: "Dev / SA Persona"
classification: "Internal"
tags: [adr, architecture-decisions, panomete, platform]
standard_ref:
  - SWEBOK v4 — Architecture
  - SEBoK v2 — System Architecture
  - ISO/IEC/IEEE 42010 — Architecture Description
---

# Architecture Decision Records — Panomete Platform

> **Project:** Panomete Platform
> **Version:** 0.1 | **Status:** Active
> **Last Updated:** 2026-07-22

---

## ADR Index

| ADR | Title | Status | Date | Decision |
|-----|-------|--------|------|---------|
| ADR-001 | Keycloak for Centralized Identity | ✅ Accepted | 2026-07-22 | Use Keycloak, not a custom auth service |
| ADR-002 | Spring Cloud Gateway as API Gateway | ✅ Accepted | 2026-07-22 | Use Spring Cloud Gateway, not Kong/Traefik/Nginx |
| ADR-003 | Eureka for Service Discovery | ✅ Accepted | 2026-07-22 | Use Spring Cloud Netflix Eureka, not Consul/k8s DNS |
| ADR-004 | Spring Boot / Java 21 for Foundation Services | ✅ Accepted | 2026-07-22 | All foundation services in Java, not Go/TS/Python |
| ADR-005 | PostgreSQL for Guard Persistence | ✅ Accepted | 2026-07-22 | Use PostgreSQL for Keycloak backend, not H2 |
| ADR-006 | Docker Compose for Deployment (MVP) | ✅ Accepted | 2026-07-22 | Start with Compose; design for k3s later |
| ADR-007 | JWT (Stateless) for Service Authentication | ✅ Accepted | 2026-07-22 | Use JWT with local signature validation, not opaque token introspection |
| ADR-008 | Gateway-Side Authentication | ✅ Accepted | 2026-07-22 | Enforce auth at Gateway; services trust forwarded headers |

---

## ADR-001: Keycloak for Centralized Identity

| Field | Detail |
|-------|--------|
| **Status** | ✅ Accepted |
| **Date** | 2026-07-22 |
| **Decision Makers** | Dev / SA Persona |

### Context

> Every service in the Panomete Platform needs authentication. The options are: (a) build a custom auth service, (b) use an off-the-shelf identity provider, (c) use a SaaS identity provider. The platform is a personal homelab — no budget for SaaS, and a custom auth service is a massive security risk and maintenance burden.

### Decision

> Use **Keycloak** as the centralized identity provider. Keycloak is:
> - Production-grade, open-source, OAuth2/OIDC compliant
> - Mature (Red Hat sponsored, part of the CNCF ecosystem)
> - Docker-deployable with PostgreSQL backend
> - Supports realm-as-code (JSON export/import)
> - Provides admin console, user management, SSO sessions, RBAC out of the box

### Consequences

**Positive:**
- Zero custom auth code — Keycloak handles login, logout, token issuance, token validation, user management, role management
- OAuth2/OIDC standard — any language (Java, Go, TypeScript) can integrate
- Single sign-on across all platform services
- Realm export as JSON → version-controlled, reproducible IAM configuration
- Portfolio value — demonstrates experience with enterprise IAM

**Negative:**
- Keycloak is a large Java application (~500 MB memory baseline)
- Learning curve for realm configuration, client setup, and token flows
- PostgreSQL dependency — adds a stateful service to the platform
- Keycloak upgrades can be complex (database migration scripts)

### Alternatives Considered

| Alternative | Why Not |
|-------------|---------|
| Custom Auth Service (Spring Security) | Massive effort — login, MFA, RBAC, session management, token lifecycle. Reinventing the wheel. Security risk. |
| Auth0 / Okta (SaaS) | Not free. Vendor lock-in. Doesn't fit the "self-hosted homelab" ethos. |
| Zitadel | Newer, less mature, smaller community. Keycloak has 10+ years of production use. |
| Authentik | Good alternative, but Keycloak is the industry standard for portfolio demonstration. |

---

## ADR-002: Spring Cloud Gateway as API Gateway

| Field | Detail |
|-------|--------|
| **Status** | ✅ Accepted |
| **Date** | 2026-07-22 |
| **Decision Makers** | Dev / SA Persona |

### Context

> The platform needs a single entry point for all external traffic — routing, authentication, rate limiting, TLS termination, and request logging. Options include: Spring Cloud Gateway, Kong, Traefik, Nginx, or Envoy.

### Decision

> Use **Spring Cloud Gateway**. Rationale:
> - Native Spring ecosystem — integrates seamlessly with Spring Security (for OAuth2 JWT validation) and Spring Cloud Discovery (for Eureka route resolution)
> - Reactive, non-blocking (Netty-based) — efficient for an API gateway
> - Declarative route configuration in `application.yml` — no external admin API needed for MVP
> - Portfolio value — demonstrates Spring ecosystem depth
> - Already in the chosen tech stack (Java 21 / Spring Boot 3.x)

### Consequences

**Positive:**
- Single codebase/ecosystem for all foundation services — no polyglot complexity at the infra layer
- Spring Security OAuth2 integration = JWT validation is a few lines of config, not custom code
- `lb://` URI scheme resolves backend addresses from Eureka — zero config when services move
- Actuator health endpoints, Micrometer metrics, structured logging — all built-in

**Negative:**
- JVM-based gateway has higher memory footprint than Nginx (~256-512 MB vs ~10 MB)
- Less battle-tested than Nginx at massive scale (irrelevant for homelab)
- No built-in dashboard/admin UI (Eureka dashboard covers service health)
- Route changes require restart (acceptable for MVP; Spring Cloud Gateway supports dynamic routes via Actuator if needed later)

### Alternatives Considered

| Alternative | Why Not |
|-------------|---------|
| Kong | Requires PostgreSQL + admin API. Overkill for a 3-service foundation. |
| Traefik | Excellent, but would add a Go component to an otherwise Java foundation stack. Discovery integration less native. |
| Nginx | Battle-tested but imperative config; no native Eureka integration; JWT validation requires OpenResty/Lua scripting. |
| Envoy | Extremely powerful but complex. Better suited for service mesh (Istio) than a simple API gateway. |

---

## ADR-003: Eureka for Service Discovery

| Field | Detail |
|-------|--------|
| **Status** | ✅ Accepted |
| **Date** | 2026-07-22 |
| **Decision Makers** | Dev / SA Persona |

### Context

> Microservices need to discover each other dynamically — no hardcoded host:port. Options: Spring Cloud Netflix Eureka, HashiCorp Consul, Kubernetes DNS, or manual configuration.

### Decision

> Use **Spring Cloud Netflix Eureka** in standalone mode. Rationale:
> - Simplest path with Spring Boot — embed the Eureka server, add the Eureka client dependency, and services auto-register
> - No external binary dependency (unlike Consul, which requires a separate agent)
> - Dashboard included — visual service health overview
> - Spring Cloud Gateway resolves `lb://` URIs from Eureka natively
> - Portfolio value — demonstrates Spring Cloud ecosystem

### Consequences

**Positive:**
- Services auto-register on startup with minimal configuration (`spring.application.name` + Eureka client on classpath)
- Gateway route resolution via `lb://service-name` — no gateway config changes when services scale or move
- Health-based eviction — dead instances are removed from registry within 90 seconds
- Dashboard at `/eureka` shows all services and their status

**Negative:**
- Single node in MVP — no high availability (acceptable for homelab)
- Self-preservation mode can be confusing (prevents eviction during network blips)
- Eureka is specific to Spring Cloud — non-Spring services need REST API calls for registration
- Less feature-rich than Consul (no KV store, no health checking beyond heartbeats)

### Alternatives Considered

| Alternative | Why Not |
|-------------|---------|
| HashiCorp Consul | Richer features (KV store, service mesh) but requires running a Consul agent on every host. Overkill for homelab. |
| Kubernetes DNS | The eventual target (k3s) provides this natively, but Compose doesn't. Eureka works now and can be replaced later. |
| Manual configuration | Not production-grade. Fails OBJ-02: Service Discovery. |

---

## ADR-004: Spring Boot / Java 21 for Foundation Services

| Field | Detail |
|-------|--------|
| **Status** | ✅ Accepted |
| **Date** | 2026-07-22 |
| **Decision Makers** | Dev / SA Persona |

### Context

> The foundation services (Guard, Discover, Gate) need a technology stack. Options span Java (Spring Boot), Go, TypeScript/Node.js, Python, or Rust. Business services may use different languages — the foundation should pick the best tool for infrastructure concerns.

### Decision

> Use **Java 21 + Spring Boot 3.x** for all three foundation services. Rationale:
> - Spring Cloud provides native Gateway, Security, and Discovery — the three concerns the foundation solves
> - One language for infrastructure = consistent tooling, build pipelines, Docker images, JVM tuning
> - Business services are free to use any language (the foundation is the platform, not the apps)
> - Portfolio value — demonstrates enterprise Java ecosystem competency

### Consequences

**Positive:**
- Spring ecosystem integration is seamless (Gateway ↔ Security ↔ Discovery)
- Mature tooling: Maven/Gradle, JUnit, Mockito, Actuator, Micrometer
- Large community — every problem has been solved
- Virtual threads in Java 21 for efficient I/O

**Negative:**
- JVM memory footprint (mitigated by setting explicit heap limits)
- Slower startup than Go/Rust (acceptable — services aren't auto-scaled in homelab)
- Verbose compared to Go/TS for simple services (irrelevant — foundation services are configuration-heavy, not logic-heavy)

### Alternatives Considered

| Alternative | Why Not |
|-------------|---------|
| Go | Excellent for business services, but Go lacks an ecosystem equivalent to Spring Cloud. Would need to assemble Gateway + Discovery from scratch or use external tools. |
| TypeScript/Node.js | Event-driven and lightweight, but no mature service discovery or API gateway framework comparable to Spring Cloud. |
| Python | Not suitable for high-concurrency API gateway. No mature microservice infrastructure framework. |

---

## ADR-005: PostgreSQL for Guard Persistence

| Field | Detail |
|-------|--------|
| **Status** | ✅ Accepted |
| **Date** | 2026-07-22 |
| **Decision Makers** | Dev / SA Persona |

### Context

> Keycloak needs a database backend for persistent storage of realms, clients, users, roles, and sessions. Options: PostgreSQL, MySQL, H2 (embedded), or external cloud DB.

### Decision

> Use **PostgreSQL 15+** as Keycloak's database backend. This is Keycloak's recommended production database. H2 (embedded) is explicitly not production-grade and has known issues with concurrent access and data durability.

### Consequences

**Positive:**
- ACID compliance — no data corruption on crash
- Persistent across restarts — `docker compose down && up` preserves all users, clients, roles
- Keycloak documentation and community assume PostgreSQL in production
- Already in the homelab stack (Tiny Mchwa uses it, future business services will too)

**Negative:**
- Adds a stateful service that must be backed up
- Startup ordering: Guard depends on guard-db being healthy
- Slightly higher resource usage than H2

### Alternatives Considered

| Alternative | Why Not |
|-------------|---------|
| H2 (embedded) | Keycloak docs explicitly state H2 is NOT for production. Data loss on restart. Concurrency issues. |
| MySQL | Keycloak supports it, but PostgreSQL is preferred. Homelab already uses PostgreSQL. |
| External cloud DB | Defeats the purpose of a self-hosted homelab. Adds cost. |

---

## ADR-006: Docker Compose for Deployment (MVP)

| Field | Detail |
|-------|--------|
| **Status** | ✅ Accepted |
| **Date** | 2026-07-22 |
| **Decision Makers** | Dev / SA Persona |

### Context

> The platform must run on a single homelab machine. Options: Docker Compose, Kubernetes (k3s/microk8s), or bare-metal process management. The long-term goal is k3s for portfolio value, but Phase 1 should optimize for development velocity.

### Decision

> Use **Docker Compose** for Phase 1 MVP. Design all services with Kubernetes portability in mind (health checks, env-var config, stateless where possible). Migrate to k3s in a future phase.

### Consequences

**Positive:**
- One command to start the entire platform: `docker compose up`
- Simple networking — all services on a shared bridge network, DNS resolution by service name
- No Kubernetes learning curve during development
- `depends_on` with health checks for startup ordering
- `.env` file for configuration — matches K8s ConfigMap/Secret pattern

**Negative:**
- No auto-scaling, self-healing, or rolling updates
- Single-host only (acceptable for homelab)
- Manual restart if a service crashes (mitigated by `restart: unless-stopped`)
- Migration to k3s will require refactoring (acceptable — designed for it from day one)

### Alternatives Considered

| Alternative | Why Not |
|-------------|---------|
| k3s from day one | Steep learning curve. Delays Phase 1 delivery. Overkill for 3-4 services. |
| Bare-metal | No isolation, dependency conflicts, harder to reproduce. Docker is the standard. |
| Nomad | Simpler than K8s but less portfolio value. Smaller community. |

---

## ADR-007: JWT (Stateless) for Service Authentication

| Field | Detail |
|-------|--------|
| **Status** | ✅ Accepted |
| **Date** | 2026-07-22 |
| **Decision Makers** | Dev / SA Persona |

### Context

> When a request reaches the Gateway, it must be authenticated. Two approaches: (a) validate the JWT signature locally using Keycloak's public key (JWKS), or (b) call Keycloak's introspection endpoint for every request. The trade-off is between latency (local validation is faster) and token revocation support (introspection can check if a token was revoked).

### Decision

> Use **JWT with local signature validation** at the Gateway. The Gateway downloads Keycloak's public key from the JWKS endpoint at startup (and refreshes periodically). It validates the JWT signature, expiration, issuer, and audience locally — no network call per request. Token revocation is handled by short token lifetimes (5 minutes).

### Consequences

**Positive:**
- Zero network latency for token validation — Gateway validates locally
- Resilient — Gateway works even if Keycloak is temporarily slow (it only needs the public key, cached in memory)
- Simpler architecture — no introspection call on every request
- Standard Spring Security OAuth2 Resource Server config — minimal code

**Negative:**
- Token revocation is NOT immediate — a revoked token is valid until it expires (5 min maximum window)
- Cannot support opaque tokens (requires introspection endpoint)
- Must handle Keycloak key rotation (Spring Security handles this automatically via JWKS caching)

### Alternatives Considered

| Alternative | Why Not |
|-------------|---------|
| Token introspection (opaque tokens) | Network call per request to Keycloak. Higher latency. Gateway depends on Keycloak being responsive for EVERY request. |
| Session-based auth (server-side sessions) | Stateful — doesn't scale. Gateway would need to store sessions. Breaks stateless Gateway model. |

---

## ADR-008: Gateway-Side Authentication

| Field | Detail |
|-------|--------|
| **Status** | ✅ Accepted |
| **Date** | 2026-07-22 |
| **Decision Makers** | Dev / SA Persona |

### Context

> Authentication can be enforced at the Gateway (perimeter security) or delegated to each individual service. Delegating means every service must implement JWT validation, extract claims, and handle auth errors — duplicating code and increasing the attack surface.

### Decision

> **Enforce authentication at the Gateway.** The Gateway validates the JWT, extracts user claims (ID, roles), and forwards them as trusted headers (`X-User-Id`, `X-User-Roles`) to downstream services. Services trust these headers because they are only reachable from the Gateway on the internal Docker network.

### Consequences

**Positive:**
- Services receive pre-validated user context — no auth code needed in business services
- Single choke point for security — audit, rate limiting, and auth in one place
- Business services can focus on business logic, not auth
- Consistent auth behavior — all services behave identically for 401/403 scenarios

**Negative:**
- Services must trust the Gateway (acceptable on a private Docker network)
- Service-to-service calls still need auth — handled via Client Credentials grant (ADR-007)
- Gateway becomes a critical single point of failure for security (acceptable — if Gateway is down, the platform is unreachable anyway)

### Alternatives Considered

| Alternative | Why Not |
|-------------|---------|
| Per-service auth | Every service duplicates JWT validation, claim extraction, error handling. Violates DRY. Increases attack surface. |
| Service mesh (Istio/Linkerd) | Overkill for 3 services. Adds significant complexity. Future option with k3s. |

---

## ADR Management

| Rule | Description |
|------|-------------|
| One decision per ADR | Each ADR covers exactly one architecture decision |
| Immutable once accepted | Don't edit an accepted ADR — create a new ADR to supersede (e.g., ADR-006 superseded by ADR-009) |
| Store in version control | ADRs live in the spec repo alongside code |
| Link to trade studies | Reference service-level ADRs for deeper dives |

---

## Related Documents

| Document | Relationship |
|----------|-------------|
| [[025_software_architecture_document]] | Architecture that these ADRs define |
| [[flowero_guard/021_architecture_decision_records]] | Guard-specific ADRs |
| [[flowero_discover/021_architecture_decision_records]] | Discover-specific ADRs |
| [[flowero_gate/021_architecture_decision_records]] | Gate-specific ADRs |
| [[011_business_objective]] | Objectives driving these decisions |

---

> **Template Standard:** Based on SWEBOK v4, SEBoK v2, ISO/IEC/IEEE 42010
> **Usage:** ADRs capture *why*. Code shows *what*, docs show *how*, ADRs show *why*. Future you (and portfolio reviewers) will thank you.
