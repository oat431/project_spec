---
document_type: ADR (Architecture Decision Records)
version: "0.1"
status: Active
author: "Dev / SA Persona"
created: "2026-07-22"
last_updated: "2026-07-22"
project_name: "Flowero Gate"
project_id: "PAN-GATE-001"
parent_platform: "Panomete Platform"
architect: "Dev / SA Persona"
classification: "Internal"
tags: [adr, api-gateway, spring-cloud-gateway, panomete]
standard_ref:
  - SWEBOK v4 — Architecture
  - ISO/IEC/IEEE 42010 — Architecture Description
---

# Architecture Decision Records — Flowero Gate

> **Service:** Flowero Gate (Spring Cloud Gateway)
> **Platform:** Panomete Platform
> **Version:** 0.1 | **Status:** Active
> **Last Updated:** 2026-07-22

---

## ADR Index

| ADR | Title | Status | Date | Decision |
|-----|-------|--------|------|---------|
| ADR-W001 | Declarative Route Configuration (YAML) | ✅ Accepted | 2026-07-22 | Routes defined in `application.yml`, not programmatic API |
| ADR-W002 | JWT Validation at Gateway (Not Introspection) | ✅ Accepted | 2026-07-22 | Validate JWT signature locally, don't call Guard per request |
| ADR-W003 | Forward User Claims as Headers | ✅ Accepted | 2026-07-22 | Pass `X-User-Id` and `X-User-Roles` to downstream services |
| ADR-W004 | Path-Based Routing Only (MVP) | ✅ Accepted | 2026-07-22 | Route by path pattern; no header/query-param routing in MVP |
| ADR-W005 | In-Memory Rate Limiting (No Redis) | ✅ Accepted | 2026-07-22 | Use Spring Cloud Gateway's built-in RequestRateLimiter with in-memory backend |
| ADR-W006 | Self-Signed TLS for Development | ✅ Accepted | 2026-07-22 | Self-signed cert for homelab; Let's Encrypt later |

---

## ADR-W001: Declarative Route Configuration (YAML)

| Field | Detail |
|-------|--------|
| **Status** | ✅ Accepted |
| **Date** | 2026-07-22 |

### Context

> Gateway routes must be defined somewhere. Options: (a) `application.yml` — declarative, version-controlled, no runtime API; (b) Spring Cloud Gateway's `RouteDefinitionWriter` API — dynamic, programmable; (c) external config server. For MVP with 3-5 routes, dynamic configuration is overkill.

### Decision

> **Define all routes declaratively in `application.yml`** under `spring.cloud.gateway.routes`. Each route specifies: `id`, `uri` (using `lb://` for Eureka-resolved backends), `predicates` (path patterns), and `filters` (auth, rate limiting).

### Consequences

**Positive:**
- Routes are version-controlled alongside code
- Single source of truth — no runtime changes possible (security benefit)
- Simpler debugging — all routes in one file
- Route changes are code changes → go through PR/review

**Negative:**
- Route changes require a Gateway restart (~5 seconds downtime — acceptable for homelab)
- Cannot add routes dynamically without restart
- No admin UI for route management (Eureka dashboard shows registered services)

### Alternatives Considered

| Alternative | Why Not |
|-------------|---------|
| `RouteDefinitionWriter` API | Dynamic routes are overkill for 3-5 services. Adds complexity without benefit in MVP. |
| External config server (Spring Cloud Config) | Adds another service to manage. Overkill for homelab. Can adopt later. |

---

## ADR-W002: JWT Validation at Gateway (Not Introspection)

| Field | Detail |
|-------|--------|
| **Status** | ✅ Accepted |
| **Date** | 2026-07-22 |

### Context

> Every request to a protected route must be authenticated. Two approaches: (a) call Keycloak's introspection endpoint for every request — network call, latency, dependency on Guard's availability; (b) validate the JWT signature locally — download Keycloak's public key from the JWKS endpoint, validate locally.

### Decision

> **Validate JWT signatures locally** at the Gateway. Spring Security's OAuth2 Resource Server support downloads the JWKS at startup, caches it, and validates every JWT locally — no network call per request.

### Consequences

**Positive:**
- Zero added latency — no network call per request
- Resilient — Gateway works even when Keycloak is temporarily slow (only needs the cached public key)
- Standard Spring Security — minimal configuration: `spring.security.oauth2.resourceserver.jwt.issuer-uri`
- Automatic key rotation handling — NimbusJwtDecoder refreshes JWKS periodically

**Negative:**
- Token revocation is not immediate — a revoked token is valid until it expires (5 min max window)
- Must trust Keycloak's key rotation (Spring Security handles this)

> **See also:** Platform ADR-007 (JWT Stateless Auth) — this is the Gateway-specific implementation of that platform-level decision.

---

## ADR-W003: Forward User Claims as Headers

| Field | Detail |
|-------|--------|
| **Status** | ✅ Accepted |
| **Date** | 2026-07-22 |

### Context

> After validating the JWT, the Gateway must communicate the authenticated user's identity to downstream services. Options: (a) forward the original JWT — services re-validate; (b) extract claims and forward as trusted headers; (c) use a sidecar pattern.

### Decision

> **Extract claims from the validated JWT and forward them as HTTP headers.** The Gateway adds:
> - `X-User-Id` — the user's Keycloak UUID (from `sub` claim)
> - `X-User-Name` — the user's preferred username
> - `X-User-Roles` — comma-separated realm roles

> Services trust these headers because they are only reachable from the Gateway on the internal Docker network.

### Consequences

**Positive:**
- Services receive pre-digested user context — no JWT parsing needed
- Consistent headers across all services
- Business services don't need Spring Security or any JWT library
- Gateway is the single choke point for auth

**Negative:**
- Services must trust the Gateway (acceptable on private Docker network)
- Header spoofing risk if an attacker gains access to the internal network
- Services that need fine-grained claims must still call Guard's userinfo endpoint

**Implementation (Gateway filter):**
```java
@Bean
public GlobalFilter addUserHeadersFilter() {
    return (exchange, chain) -> {
        ServerHttpRequest request = exchange.getRequest();
        Authentication auth = SecurityContextHolder.getContext().getAuthentication();
        if (auth instanceof JwtAuthenticationToken jwtAuth) {
            Jwt jwt = jwtAuth.getToken();
            request.mutate()
                .header("X-User-Id", jwt.getSubject())
                .header("X-User-Name", jwt.getClaimAsString("preferred_username"))
                .header("X-User-Roles", String.join(",",
                    jwt.getClaimAsMap("realm_access") != null
                        ? (List<String>) jwt.getClaimAsMap("realm_access").get("roles")
                        : List.of()
                ));
        }
        return chain.filter(exchange);
    };
}
```

---

## ADR-W004: Path-Based Routing Only (MVP)

| Field | Detail |
|-------|--------|
| **Status** | ✅ Accepted |
| **Date** | 2026-07-22 |

### Context

> Spring Cloud Gateway supports routing by path, header, query parameter, host, and more. For a homelab with a single domain (`panomete.local`), path-based routing is sufficient.

### Decision

> **Use path-based routing only for MVP.** Each service gets a URL prefix:
> - `/auth/**` → Flowero Guard
> - `/eureka/**` → Flowero Discover
> - `/api/blog/**` → Cute Gufo
> - `/api/short/**` → Fluffy Mouton
> - etc.

### Consequences

**Positive:**
- Simplest routing logic — one predicate: `Path=`
- Clean, predictable URLs for users
- Easy to add new services — add one route block

**Negative:**
- Cannot route by subdomain (e.g., `blog.panomete.local`) — requires DNS/hosts file config
- No content-based routing (header, query) — not needed for MVP

---

## ADR-W005: In-Memory Rate Limiting (No Redis)

| Field | Detail |
|-------|--------|
| **Status** | ✅ Accepted |
| **Date** | 2026-07-22 |

### Context

> Spring Cloud Gateway's `RequestRateLimiter` filter needs a `RateLimiter` implementation. Options: (a) Redis-backed — rate limits survive Gateway restart, shared across instances; (b) in-memory — simpler, no external dependency.

### Decision

> **Use an in-memory `KeyResolver` + `RateLimiter`** for MVP. Rate limits reset when the Gateway restarts (acceptable for a single-user homelab). If the platform scales to multiple Gateway instances, migrate to Redis-backed rate limiting.

### Consequences

**Positive:**
- No Redis dependency — one less service to run
- Simpler configuration
- Sufficient for single Gateway instance

**Negative:**
- Rate limits reset on Gateway restart
- Cannot share rate limit state across multiple Gateway instances (future)
- Memory usage grows with unique client IPs (irrelevant for single-digit users)

---

## ADR-W006: Self-Signed TLS for Development

| Field | Detail |
|-------|--------|
| **Status** | ✅ Accepted |
| **Date** | 2026-07-22 |

### Context

> External traffic to the Gateway must be encrypted (HTTPS). Options for TLS certificates: (a) self-signed certificate; (b) Let's Encrypt with a real domain; (c) mkcert for local development; (d) no TLS (HTTP only).

### Decision

> **Use a self-signed certificate for development.** Generate with Java's `keytool`:
> ```bash
> keytool -genkeypair -alias panomete -keyalg RSA \
>   -keystore keystore.p12 -storetype PKCS12 \
>   -dname "CN=panomete.local" -validity 365
> ```
> Configure in Gateway's `application.yml`:
> ```yaml
> server:
>   ssl:
>     key-store: classpath:keystore.p12
>     key-store-password: ${KEYSTORE_PASSWORD}
>     key-alias: panomete
> ```
> Migrate to Let's Encrypt if/when a real domain is used.

### Consequences

**Positive:**
- Encrypted traffic even in development
- Works without internet access
- No domain registration or DNS setup needed

**Negative:**
- Browser shows "Your connection is not private" warning — must click "Advanced → Proceed"
- Not trusted by default on any device
- Must accept the cert on every new browser/device

---

## Related Documents

| Document | Relationship |
|----------|-------------|
| [[flowero_gate/022_API_specification]] | Gateway route table and contract |
| [[panomete_platform/021_architecture_decision_records]] | Platform-level ADRs |
| [[flowero_guard/022_API_specification]] | Auth endpoints Gate depends on |
| [[flowero_discover/022_API_specification]] | Discovery endpoints Gate depends on |
| [[flowero_gate/012_user_stories]] | Stories driving these decisions |
