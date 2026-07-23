---
document_type: ADR (Architecture Decision Records)
version: "0.2"
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
  - ISO/IEC/IEEE 42010 — Architecture Description
---

# Architecture Decision Records — Panomete Platform

> **Project:** Panomete Platform
> **Version:** 0.2 | **Status:** Active — Updated per Design Review 2026-07-22
> **Last Updated:** 2026-07-22

---

## ADR Index

| ADR | Title | Status | Date | Decision |
|-----|-------|--------|------|---------|
| ADR-001 | Keycloak for Centralized Identity | ✅ Accepted | 2026-07-22 | Keycloak, not custom auth service. Guard IS Keycloak. |
| ADR-002 | Spring Cloud Gateway as Internal API Gateway | ✅ Accepted | 2026-07-22 | Gate routes business APIs only. Behind existing Nginx. |
| ADR-003 | Eureka for Service Discovery | ✅ Accepted | 2026-07-22 | Spring Cloud Netflix Eureka, standalone mode |
| ADR-004 | Java 25 / Spring Boot 4.1 for Foundation | ✅ Accepted | 2026-07-23 | All foundation services in Java |
| ADR-005 | Shared PostgreSQL 18 (Existing) | ✅ Accepted | 2026-07-22 | Use existing PostgreSQL. No dedicated container. |
| ADR-006 | Nginx Edge + Cloudflare TLS (Existing) | ✅ Accepted | 2026-07-22 | Keep existing edge infrastructure. Gate is internal. |
| ADR-007 | JWT Local Validation at Gate | ✅ Accepted | 2026-07-22 | Validate JWT signature locally via cached JWKS |
| ADR-008 | Valkey-Backed Rate Limiting | ✅ Accepted | 2026-07-22 | Shared Valkey 9. Survives Gate restarts. |
| ADR-009 | Subdomain (Nginx) + Path (Gate) Routing | ✅ Accepted | 2026-07-22 | Foundation = subdomains. Business APIs = paths through Gate. |

---

## ADR-001: Keycloak for Centralized Identity

| Field | Detail |
|-------|--------|
| **Status** | ✅ Accepted |
| **Date** | 2026-07-22 |

### Context

> Every service needs authentication. Options: custom auth service, off-the-shelf IAM, or SaaS. This is a personal homelab — no SaaS budget. Custom auth is a massive security risk.

### Decision

> **Use Keycloak.** Flowero Guard IS Keycloak — deployed as a Docker container, no wrapper service. Gate validates JWT signatures locally against Keycloak's JWKS (cached on startup) — no per-request call to Guard. Keycloak handles token issuance, user management, SSO, and RBAC.

### Consequences

**Positive:**
- Zero custom auth code. OAuth2/OIDC standard — any language can integrate.
- SSO across all services. Realm-as-code (JSON export) for version-controlled IAM.
- Portfolio value — demonstrates enterprise IAM experience.

**Negative:**
- Keycloak is resource-heavy (~500 MB baseline). Learning curve for realm config.
- PostgreSQL dependency — but PG is already running.

### Alternatives Considered

| Alternative | Why Not |
|-------------|---------|
| Custom Auth Service | Massive effort. Security risk. Reinventing the wheel. |
| Auth0 / Okta (SaaS) | Not free. Vendor lock-in. Defeats self-hosted ethos. |
| Zitadel / Authentik | Less mature. Keycloak has 10+ years of production use. |

---

## ADR-002: Spring Cloud Gateway as Internal API Gateway

| Field | Detail |
|-------|--------|
| **Status** | ✅ Accepted |
| **Date** | 2026-07-22 |

### Context

> Nginx already serves as the edge reverse proxy (subdomain routing, existing self-hosted apps). Should Gate replace Nginx, sit behind it, or not exist at all?

### Decision

> **Keep Nginx as the edge. Add Spring Cloud Gateway as an internal API gateway behind Nginx** on port 8000 at `api.panomete.com`. Gate routes **business APIs only** — `/api/{service}/**`. Foundation services (Guard, Discover) are routed directly by Nginx at their own subdomains. Gate handles JWT validation, Valkey-backed rate limiting, Eureka route resolution, and structured JSON logging.

### Consequences

**Positive:**
- Nginx continues to serve existing apps without disruption
- Gate inherits Spring ecosystem benefits: Spring Security OAuth2, Eureka `lb://` resolution, Actuator health/metrics
- Clear separation: Nginx = subdomain edge routing, Gate = API platform concerns
- Foundation services accessible directly via subdomains (simpler debugging)

**Negative:**
- Two proxies (Nginx → Gate) — marginal added latency
- Gate restarts affect all business APIs but not Guard/Discover

---

## ADR-003: Eureka for Service Discovery

*(Unchanged from v0.1 — standalone Eureka, auto-registration, dashboard via Nginx at `discovery.panomete.com`)*

---

## ADR-004: Java 25 / Spring Boot 4.1 for Foundation

| Field | Detail |
|-------|--------|
| **Status** | ✅ Accepted |
| **Date** | 2026-07-23 (updated from v0.2 — version upgraded) |

### Context

> Foundation services need a JVM runtime and web framework. Java 25 is the latest LTS release. Spring Boot 4.1.x is the latest GA version with Spring Cloud 2025.1.x (Oakwood) — which provides native Gateway, Security, and Discovery integrations.

### Decision

> **Use Java 25 / Spring Boot 4.1.x / Spring Cloud 2025.1.x for all foundation services.** This is the latest stable stack. Java 25 is LTS. Spring Boot 4.1 brings reactive-first improvements. Spring Cloud 2025.1 (Oakwood) provides the gateway, security, and Eureka integrations needed.

### Consequences

**Positive:**
- Latest LTS Java — long support window
- Boot 4.1 reactive-first design aligns with WebFlux/Netty gateway
- Records, pattern matching, and virtual threads available
- Spring Cloud 2025.1 renames gateway artifact to `spring-cloud-gateway-server-webflux`

**Negative:**
- Bleeding edge — some third-party libraries may lag behind Java 25 / Boot 4.1 compatibility
- Boot 4.x has breaking changes from 3.x (jakarta namespace, Security 7 API changes)

---

## ADR-005: Shared PostgreSQL 18 (Existing)

| Field | Detail |
|-------|--------|
| **Status** | ✅ Accepted |
| **Date** | 2026-07-22 |

### Context

> Keycloak needs a database. Options: spin up a dedicated PostgreSQL container, or use the existing shared PostgreSQL 18 that's already running and hosting other services.

### Decision

> **Use the existing shared PostgreSQL 18.** Create a `keycloak` database on it. No dedicated guard-db container. This reduces resource usage and follows the DRY infrastructure pattern already established (AdGuard, Portainer, etc. all share resources).

### Consequences

**Positive:**
- No additional container — reduces total RAM usage
- One PostgreSQL instance to manage, back up, and monitor
- Consistent — matches how Tiny Mchwa and future services use the same PG

**Negative:**
- Keycloak's schema cohabitates with other databases on the same instance
- Keycloak version upgrades run Liquibase migrations against the shared instance

---

## ADR-006: Nginx Edge + Cloudflare TLS (Existing)

| Field | Detail |
|-------|--------|
| **Status** | ✅ Accepted |
| **Date** | 2026-07-22 |

### Context

> The homelab already has Cloudflare Tunnel → Nginx handling external traffic with TLS termination. Should we replace this with Gate, or layer Gate behind it?

### Decision

> **Keep the existing infrastructure.** Cloudflare terminates TLS. Nginx routes by subdomain. Gate is an internal service at `api.panomete.com:8000` behind Nginx. Internal traffic is plain HTTP on the trusted Docker network.

### Consequences

**Positive:**
- No disruption to existing services (AdGuard, Portainer)
- Cloudflare provides DDoS protection, TLS, and DNS management already
- Nginx already configured for subdomain routing
- Gate doesn't need to worry about TLS

**Negative:**
- Gate depends on Nginx being correctly configured for `api.panomete.com`
- Two proxies = two places to debug routing issues

---

## ADR-007: JWT Local Validation at Gate

| Field | Detail |
|-------|--------|
| **Status** | ✅ Accepted |
| **Date** | 2026-07-22 |

### Context

> Gate must authenticate requests. Options: call Keycloak's introspection endpoint per request, or validate JWT signatures locally.

### Decision

> **Validate JWT signatures locally at Gate.** Gate downloads Keycloak's public key from the JWKS endpoint (`auth.panomete.com/realms/panomete/protocol/openid-connect/certs`) at startup and caches it. Per-request validation is local — zero network call, zero latency.

### Consequences

**Positive:**
- No network call per request — fast, resilient
- Gate works even if Keycloak is temporarily slow
- Standard Spring Security `oauth2ResourceServer` config

**Negative:**
- Token revocation is not immediate (max 5 min window until token expires)

---

## ADR-008: Valkey-Backed Rate Limiting

| Field | Detail |
|-------|--------|
| **Status** | ✅ Accepted |
| **Date** | 2026-07-22 |

### Context

> Gate needs rate limiting. In-memory limits reset on Gate restart. Valkey 9 is already running and used by other services.

### Decision

> **Use the shared Valkey 9 instance** for rate limiting. Spring Cloud Gateway's `RequestRateLimiter` is backed by Valkey. Limits persist across Gate restarts. No new infrastructure needed.

### Consequences

**Positive:**
- Rate limits survive Gate restarts
- Uses existing infrastructure (DRY)
- Can share rate limit state across multiple Gate instances in the future

**Negative:**
- Adds Valkey as a runtime dependency for Gate
- Slightly more complex than in-memory

---

## ADR-009: Subdomain (Nginx) + Path (Gate) Routing

| Field | Detail |
|-------|--------|
| **Status** | ✅ Accepted |
| **Date** | 2026-07-22 |

### Context

> Foundation services are accessed differently from business services. Should we route everything through Gate (path-based), or use subdomains for foundation services?

### Decision

> **Hybrid routing:**
> - **Foundation services** → Nginx subdomains: `auth.panomete.com`, `discovery.panomete.com`
> - **Business APIs** → Path-based through Gate: `api.panomete.com/api/{service}/**`
> - **Business FEs** → Nginx subdomains: `blog.panomete.com`, `todo.panomete.com`

> Foundation services don't need Gate's middleware (Guard IS the auth provider, Discover IS the registry). Business APIs benefit from centralized auth, rate limiting, and routing.

### Consequences

**Positive:**
- Foundation services are simpler to debug (direct access)
- Gate's scope is clear — business API platform only
- FE apps have clean URLs (`blog.panomete.com`)

**Negative:**
- Foundation services don't get rate limiting from Gate
- Nginx configuration grows with each service's subdomain

---

## Related Documents

| Document | Relationship |
|----------|-------------|
| [[025_software_architecture_document]] | Architecture that these ADRs define |
| [[029_architecture_overview]] | One-page architecture diagram |
| [[011_business_objective]] | Objectives driving these decisions |

---

> **Template Standard:** Based on SWEBOK v4, ISO/IEC/IEEE 42010
> **Usage:** ADRs capture *why*. These were validated against actual production infrastructure during the 2026-07-22 design review.
