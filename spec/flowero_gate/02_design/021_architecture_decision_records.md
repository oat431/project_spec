---
document_type: ADR (Architecture Decision Records)
version: "0.2"
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
> **Version:** 0.2 | **Status:** Active — Updated per Design Review 2026-07-22
> **Last Updated:** 2026-07-22

---

## ADR Index

| ADR | Title | Status | Date | Decision |
|-----|-------|--------|------|---------|
| ADR-W001 | Declarative Route Config (YAML) | ✅ Accepted | 2026-07-22 | Routes in `application.yml`, version-controlled |
| ADR-W002 | Local JWT Validation (Not Introspection) | ✅ Accepted | 2026-07-22 | Validate JWT locally via cached JWKS |
| ADR-W003 | Forward User Claims as Headers | ✅ Accepted | 2026-07-22 | `X-User-Id`, `X-User-Roles` to downstream services |
| ADR-W004 | Path-Based Routing for Business APIs | ✅ Accepted | 2026-07-22 | `/api/{service}/**` — foundation services excluded |
| ADR-W005 | Valkey-Backed Rate Limiting | ✅ Accepted | 2026-07-22 | Shared Valkey 9. Survives Gate restarts. |
| ADR-W006 | Internal-Only Gateway (Behind Nginx) | ✅ Accepted | 2026-07-22 | Gate on :8000, behind existing Nginx + Cloudflare |
| ADR-W007 | No TLS at Gate (Cloudflare Handles It) | ✅ Accepted | 2026-07-22 | Internal plain HTTP. Cloudflare terminates TLS. |

---

## ADR-W005: Valkey-Backed Rate Limiting

| Field | Detail |
|-------|--------|
| **Status** | ✅ Accepted (supersedes v0.1 in-memory decision) |
| **Date** | 2026-07-22 |

### Context

> v0.1 planned in-memory rate limiting. But Valkey 9 is already running in the homelab. In-memory limits reset on Gate restart.

### Decision

> **Use the shared Valkey 9 instance** for rate limiting. Spring Cloud Gateway's `RequestRateLimiter` is backed by Valkey. Config:
> ```yaml
> spring:
>   cloud:
>     gateway:
>       routes:
>         - id: blog
>           filters:
>             - name: RequestRateLimiter
>               args:
>                 redis-rate-limiter.replenishRate: 100
>                 redis-rate-limiter.burstCapacity: 200
> ```

### Consequences

**Positive:**
- Rate limits survive Gate restarts
- Uses existing infrastructure (DRY)
- Can share limits across multiple Gate instances later

**Negative:**
- Adds Valkey as a runtime dependency (already present)

---

## ADR-W006: Internal-Only Gateway (Behind Nginx)

| Field | Detail |
|-------|--------|
| **Status** | ✅ Accepted |
| **Date** | 2026-07-22 |

### Context

> v0.1 assumed Gate would be the edge proxy on :80/:443. Reality: Nginx + Cloudflare Tunnel already serve this role and host other apps (AdGuard, Portainer).

### Decision

> **Gate runs internally on port 8000**, behind Nginx at `api.panomete.com`. Gate is NOT exposed to the internet directly. Nginx handles subdomain routing. Cloudflare handles TLS. Gate handles API-specific concerns: JWT validation, rate limiting, route resolution, JSON logging.

### Consequences

**Positive:** No disruption to existing services. Gate is simpler — no TLS config, no edge hardening.

**Negative:** Two-proxy hop (Nginx → Gate). Nginx must be configured for `api.panomete.com`.

---

## ADR-W007: No TLS at Gate (Cloudflare Handles It)

| Field | Detail |
|-------|--------|
| **Status** | ✅ Accepted (supersedes v0.1 TLS decision) |
| **Date** | 2026-07-22 |

### Context

> v0.1 planned self-signed TLS at Gate. But Cloudflare terminates TLS at the edge. Traffic from Cloudflare to Nginx is encrypted (cloudflared tunnel). Traffic inside the Docker network is trusted.

### Decision

> **Gate does NOT handle TLS.** It listens on plain HTTP (:8000). TLS is Cloudflare's responsibility. US-206 (TLS Termination) is removed from Gate scope. Internal Docker network is trusted.

### Consequences

**Positive:**
- No TLS configuration, cert management, or self-signed cert warnings
- One less story to implement (US-206 removed — saved 2 points)

**Negative:**
- If Cloudflare is bypassed, traffic would be unencrypted (mitigated: UFW + Tailscale for admin access)

---

*ADR-W001 through ADR-W004 unchanged from v0.1.*

---

## Related Documents

| Document | Relationship |
|----------|-------------|
| [[flowero_gate/022_API_specification]] | Gateway route table |
| [[panomete_platform/021_architecture_decision_records]] | Platform-level ADRs |
| [[flowero_guard/022_API_specification]] | JWKS endpoint Gate depends on |
| [[flowero_discover/022_API_specification]] | Discovery endpoints Gate depends on |
