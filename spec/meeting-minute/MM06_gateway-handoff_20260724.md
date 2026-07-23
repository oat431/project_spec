---
document_type: Meeting Minutes
version: "0.1"
status: Final
author: "DevOps Persona"
created: "2026-07-24"
last_updated: "2026-07-24"
project_name: "Panomete Platform"
meeting_type: "Gateway Handoff — Flowero Gate Adaptation Briefing"
participants: ["DevOps Persona", "PO Persona"]
classification: "Internal"
tags: [meeting-minutes, gateway, handoff, flowerogate, adaptation, sprint-1]
---

# Meeting Minutes — Gateway Handoff

> **Date:** 2026-07-24
> **Type:** Handoff Briefing — Flowero Gate Adaptation
> **From:** DevOps Persona
> **To:** Dev Persona (via PO)
> **Status:** ✅ Ready for Dev to pick up

---

## 1. Purpose

> Hand off the Flowero Gate (gateway) adaptation work to the Dev persona. The existing `flowerogate` project at `F:\projects\flowerogate` is ~80% aligned with the platform specs. This minute documents what needs to change, the target architecture, and deployment context so Dev can adapt the code and DevOps can deploy it.

---

## 2. Current State — What Exists

The `flowerogate` project is a **production-quality Spring Cloud Gateway** built on:

| Aspect | Current (flowerogate) | Target (spec) |
|--------|:---:|:---:|
| Java | 25 | 25 ✅ |
| Spring Boot | 4.1.0 | 4.1.x ✅ |
| Spring Cloud | 2025.1.2 (Oakwood) | 2025.1.x ✅ |
| Build tool | Gradle 9.5+ | Gradle ✅ |
| Gateway artifact | `spring-cloud-gateway-server-webflux` | ✅ |

**Already implemented (reuse these):**
- ✅ JWT validation via OAuth2 Resource Server
- ✅ `JwtClaimHeaderFilter` — extracts `sub`, `email`, `realm_access.roles`, `scope` → forwards as `X-User-Id`, `X-User-Email`, `X-User-Roles`, `X-User-Scope`
- ✅ `ResilientRedisRateLimiter` — fail-open when Valkey is down
- ✅ `RateLimiterConfig` — principal, IP, and API-key key resolvers
- ✅ `TraceIdFilter` — W3C traceparent + X-Trace-Id propagation
- ✅ `RequestLoggingFilter` — structured JSON logging (method, path, status, latency, route)
- ✅ `RateLimitResponseFilter` — standardized 429 JSON body
- ✅ `SecurityConfig` — OAuth2 Resource Server + OAuth2 Client (browser login)
- ✅ `FallbackController` — 404 + circuit breaker fallbacks
- ✅ `GatewayExceptionHandler` — global error handler
- ✅ Dockerfile — multi-stage, non-root user, ZGC
- ✅ Tests — RouteTests, SecurityTests, JwtTestHelper

---

## 3. What Needs to Change — 7 Fixes

### Fix 1: Realm name `flowerogate` → `panomete`

**Files affected:** `application.yaml`, `application-dev.yaml`, `application-prod.yaml`, `docker-compose.yml`

**Current:**
```yaml
issuer-uri: https://auth.panomete.com/realms/flowerogate
jwk-set-uri: https://auth.panomete.com/realms/flowerogate/protocol/openid-connect/certs
```

**Target:**
```yaml
issuer-uri: https://auth.panomete.com/realms/panomete
jwk-set-uri: https://auth.panomete.com/realms/panomete/protocol/openid-connect/certs
```

### Fix 2: Redis host `host.docker.internal` → `local-valkey` + add password

**Files affected:** `docker-compose.yml`, `application-prod.yaml`

**Current (docker-compose.yml):**
```yaml
REDIS_HOST: host.docker.internal
REDIS_PORT: "6379"
REDIS_PASSWORD: ***
REDIS_SSL: "false"
extra_hosts:
  - "host.docker.internal:host-gateway"
```

**Target:**
```yaml
REDIS_HOST: local-valkey
REDIS_PORT: "6379"
REDIS_PASSWORD: ${VALKEY_PASSWORD}
# Remove extra_hosts — not needed on db-network
```

> **Note:** Valkey requires authentication. The password is in `~/platform/.env` on the server.

### Fix 3: Add `db-network` to compose

**Files affected:** `docker-compose.yml`

**Current:** No network section.

**Target:**
```yaml
services:
  flowero-gate:
    # ... existing config ...
    networks:
      - shared-network

networks:
  shared-network:
    external: true
    name: db-network
```

### Fix 4: Routes — demo services → Panomete `lb://` routes

**Files affected:** `application.yaml`

**Current:**
```yaml
routes:
  - id: route-shortlink
    uri: http://shortlink-service:8080
    predicates: [Path=/api/v1/short/**]
    filters:
      - RemoveRequestHeader=Authorization
```

**Target (per Gate API spec 022):**
```yaml
routes:
  - id: blog
    uri: lb://cute-gufo
    predicates: [Path=/api/blog/**]
    filters:
      - StripPrefix=1
      - name: RequestRateLimiter
        args:
          key-resolver: "#{@ipKeyResolver}"
          redis-rate-limiter.replenishRate: 100
          redis-rate-limiter.burstCapacity: 200

  - id: short
    uri: lb://fluffy-mouton
    predicates: [Path=/api/short/**]
    filters:
      - StripPrefix=1
      - name: RequestRateLimiter
        args:
          key-resolver: "#{@ipKeyResolver}"
          redis-rate-limiter.replenishRate: 100
          redis-rate-limiter.burstCapacity: 200

  - id: todo
    uri: lb://tiny-mchwa
    predicates: [Path=/api/todo/**]
    filters:
      - StripPrefix=1
      - name: RequestRateLimiter
        args:
          key-resolver: "#{@ipKeyResolver}"
          redis-rate-limiter.replenishRate: 100
          redis-rate-limiter.burstCapacity: 200

  # Future routes (Phase 2+):
  # - id: ledger    uri: lb://big-schwein    predicates: [Path=/api/ledger/**]
  # - id: recipe    uri: lb://shy-ardilla    predicates: [Path=/api/recipe/**]
  # - id: hora      uri: lb://white-jelen    predicates: [Path=/api/hora/**]
```

> **Note:** Path prefix is `/api/{service}/**` (no `v1`). `StripPrefix=1` removes `/api` before forwarding.

### Fix 5: Enable Eureka registration

**Files affected:** `docker-compose.yml`, `application-prod.yaml`

**Current (docker-compose.yml):**
```yaml
EUREKA_CLIENT_ENABLED: "false"
```

**Target:**
```yaml
EUREKA_CLIENT_ENABLED: "true"
EUREKA_URI: http://flowero-discover:8999/eureka
```

**Current (application-prod.yaml):**
```yaml
eureka:
  client:
    enabled: true
    service-url:
      defaultZone: ${EUREKA_URI:http://eureka:8761/eureka}
```

**Target:**
```yaml
eureka:
  client:
    enabled: true
    service-url:
      defaultZone: ${EUREKA_URI:http://flowero-discover:8999/eureka}
```

### Fix 6: Update CORS origins

**Files affected:** `application-prod.yaml`, `application.yaml`

**Current:**
```yaml
cors:
  allowed-origins: ${CORS_ALLOWED_ORIGINS:https://short.panomete.com,https://gateway.panomete.com}
```

**Target:**
```yaml
cors:
  allowed-origins: ${CORS_ALLOWED_ORIGINS:https://*.panomete.com}
```

### Fix 7: Post-login redirect — make configurable

**Files affected:** `SecurityConfig.java`

**Current:**
```java
@Value("${app.post-login-redirect-url:https://short.panomete.com/short-link}")
private String postLoginRedirectUrl;
```

**Target:** Keep the `@Value` pattern but change the default to a generic URL or make it required:
```java
@Value("${app.post-login-redirect-url}")
private String postLoginRedirectUrl;
```

> This is low priority — can be deferred to when a frontend service exists.

---

## 4. Deployment Context — What Dev Needs to Know

### Server Details

| Item | Value |
|------|-------|
| Server | `remote.panomete.com` (SSH: `flowero@remote.panomete.com`) |
| Docker | 29.6.2 + Compose v5.3.1 |
| Network | `db-network` (external, all services join this) |
| PostgreSQL | `local-postgres:5432` on `db-network` |
| Valkey | `local-valkey:6379` on `db-network` (password-protected) |
| Nginx | Host-level process, proxies by `Host` header |
| Cloudflare | Tunnel, `*.panomete.com` wildcard |

### Port Conventions

| Service | Host Bind | Container Port |
|---------|-----------|:---:|
| Guard (Keycloak) | `127.0.0.1:8001` | 8080 |
| Discover (Eureka) | `127.0.0.1:8999` + `127.0.0.1:3999` | 8999 |
| **Gate (Gateway)** | **`127.0.0.1:8000`** | **8000** |

> All services bind to `127.0.0.1` — Nginx proxies externally.

### Compose Pattern

The platform compose (`~/platform/docker-compose.platform.yml`) currently has Guard and Discover. Gate will be added as a third service. The compose uses:
- `shared-network` (external, maps to `db-network`)
- `.env` file for secrets
- Healthchecks with `curl` (install `curl` in Dockerfile if using JRE base)

### Nginx

`api.panomete.com` Nginx block is already created on the server:
```nginx
server {
    server_name api.panomete.com;
    location / {
        proxy_pass http://127.0.0.1:8000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto https;
    }
}
```

---

## 5. Success Criteria

Gate adaptation is complete when:

- [ ] Realm name is `panomete` in all configs
- [ ] Redis connects to `local-valkey:6379` with password
- [ ] Container joins `db-network`
- [ ] Routes use `lb://` URIs for blog, short, todo
- [ ] Eureka registration is enabled, pointing to `flowero-discover:8999`
- [ ] CORS allows `*.panomete.com`
- [ ] `./gradlew build` passes (compile + test)
- [ ] `./gradlew test` passes (RouteTests, SecurityTests)
- [ ] Docker image builds successfully
- [ ] DevOps can deploy and verify:
  - `curl http://localhost:8000/actuator/health` → `{"status":"UP"}`
  - `curl -o /dev/null -w '%{http_code}' https://api.panomete.com/api/blog/posts` → `401`
  - Eureka dashboard shows `FLOWERO-GATE` registered

---

## 6. Reference Documents

| Document | Path | Why Dev Needs It |
|----------|------|------------------|
| Gate API Spec | `flowero_gate/02_design/022_API_specification.md` | Route table, auth behavior, error codes |
| Gate ADRs | `flowero_gate/02_design/021_architecture_decision_records.md` | Why Valkey, why no TLS, why internal-only |
| SAD §3.3 | `panomete_platform/02_design/025_software_architecture_document.md` | Gate component design |
| SAD §6 | `panomete_platform/02_design/025_software_architecture_document.md` | Deployment topology, startup order |
| DevOps — Gate | `flowero_gate/05_devops/051_CICD_pipeline_configuration.md` | CI/CD, Dockerfile, Gradle build |
| DevOps — Gate Deploy | `flowero_gate/05_devops/052_deployment_plan.md` | Step-by-step deployment |
| Platform Overview | `panomete_platform/02_design/029_architecture_overview.md` | One-page architecture map |
| flowerogate source | `F:\projects\flowerogate` | Existing codebase to adapt |

---

## 7. Action Items

| Action ID | Action | Owner | Status |
|-----------|--------|-------|--------|
| ACT-013 | Adapt flowerogate codebase (7 fixes above) | Dev Persona | ⬜ Open |
| ACT-014 | Ensure `./gradlew build` + `./gradlew test` pass | Dev Persona | ⬜ Open |
| ACT-015 | Hand adapted project back to DevOps for deployment | Dev Persona | ⬜ Open |
| ACT-016 | Deploy Gate + verify all health checks | DevOps Persona | ⬜ Open |
| ACT-017 | Create Gate construction docs (03_construction) | Dev Persona | ⬜ Open |

---

## 8. Sprint 1 Progress

| Step | Service | Status |
|:---:|---------|:---:|
| 1 | Keycloak (Guard) | ✅ Deployed + verified |
| 2 | Discovery (Eureka) | ✅ Deployed + verified |
| 3 | **Gateway (Gate)** | ⬜ **In progress — Dev adapting code** |

---

## Related Documents

| Document | Relationship |
|----------|-------------|
| [[MM03_next-steps_20260722]] | Sprint backlog |
| [[MM04_devops-infra-audit_20260723]] | Infrastructure audit findings |
| [[MM05_sprint1-priority_20260723]] | Execution priority decision |
| `flowero_gate/02_design/022_API_specification` | Target route table |
| `flowero_gate/05_devops/052_deployment_plan` | Deployment procedure |

---

> **Status:** Handoff complete. Dev persona has all context needed to adapt flowerogate. DevOps stands by for deployment.
