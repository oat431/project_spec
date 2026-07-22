---
document_type: ADR (Architecture Decision Records)
version: "0.1"
status: Active
author: "Dev / SA Persona"
created: "2026-07-22"
last_updated: "2026-07-22"
project_name: "Flowero Discover"
project_id: "PAN-DISCOVER-001"
parent_platform: "Panomete Platform"
architect: "Dev / SA Persona"
classification: "Internal"
tags: [adr, eureka, service-discovery, panomete]
standard_ref:
  - SWEBOK v4 — Architecture
  - ISO/IEC/IEEE 42010 — Architecture Description
---

# Architecture Decision Records — Flowero Discover

> **Service:** Flowero Discover (Spring Cloud Netflix Eureka)
> **Platform:** Panomete Platform
> **Version:** 0.1 | **Status:** Active
> **Last Updated:** 2026-07-22

---

## ADR Index

| ADR | Title | Status | Date | Decision |
|-----|-------|--------|------|---------|
| ADR-D001 | Standalone Eureka (Single Node) | ✅ Accepted | 2026-07-22 | Single Eureka node in standalone mode |
| ADR-D002 | Eureka Client Auto-Registration | ✅ Accepted | 2026-07-22 | Services register automatically via Spring Cloud dependency |
| ADR-D003 | Dashboard Accessed Through Gateway | ✅ Accepted | 2026-07-22 | Dashboard served through Gate at `/eureka`, not exposed directly |
| ADR-D004 | Graceful Eviction with Self-Preservation | ✅ Accepted | 2026-07-22 | Keep self-preservation mode enabled for homelab stability |

---

## ADR-D001: Standalone Eureka (Single Node)

| Field | Detail |
|-------|--------|
| **Status** | ✅ Accepted |
| **Date** | 2026-07-22 |

### Context

> Eureka supports both standalone (single node) and clustered (peer-aware) modes. A homelab with 3 foundation services and ~6 future business services does not need high availability at the registry level.

### Decision

> **Run Eureka in standalone mode** — a single Docker container. Configure with:
> ```yaml
> eureka:
>   client:
>     register-with-eureka: false
>     fetch-registry: false
> ```
> This tells Eureka: "You are the registry. Don't try to register with yourself or fetch from peers."

### Consequences

**Positive:**
- Simplest possible deployment — one container, no clustering configuration
- No split-brain scenarios
- Lower resource usage (single JVM ~256 MB)
- Services have built-in resilience — they cache the last known registry for when Eureka restarts

**Negative:**
- Single point of failure — if Eureka goes down, new registrations fail and new route resolutions fail
- No high availability (acceptable for homelab)

### Alternatives Considered

| Alternative | Why Not |
|-------------|---------|
| Eureka cluster (3 nodes) | Overkill for homelab. More resource usage, more config complexity. |
| Eureka in "self-preservation" peer mode | Adds complexity; self-preservation still works in standalone mode |

---

## ADR-D002: Eureka Client Auto-Registration

| Field | Detail |
|-------|--------|
| **Status** | ✅ Accepted |
| **Date** | 2026-07-22 |

### Context

> Services can register with Eureka via (a) Spring Cloud `@EnableDiscoveryClient` auto-registration, or (b) manual REST API calls to `/eureka/apps/{app}`. The platform foundation services are Spring Boot — auto-registration is simpler.

### Decision

> **Use Spring Cloud's auto-registration** for all Spring Boot services. Add `spring-cloud-starter-netflix-eureka-client` as a dependency and set `spring.application.name` — the service auto-registers on startup. Non-Spring services (Go, TypeScript) use Eureka's REST API for registration.

### Consequences

**Positive:**
- Zero code — a dependency + one property, and the service registers
- Health-based registration — service reports UP/DOWN, Eureka tracks via heartbeats
- Graceful deregistration on shutdown (SIGTERM → `/eureka/apps/{app}/{instance}` DELETE)
- Instance metadata (host, port, health URL) auto-populated

**Negative:**
- Tightly couples services to Spring Cloud (acceptable — foundation services are already Spring Boot)
- Non-Spring services need manual REST API integration

### Alternatives Considered

| Alternative | Why Not |
|-------------|---------|
| Manual REST API only | Every service (including Spring Boot ones) must call `/eureka/apps` on startup and heartbeat on a schedule. Reimplementing what Spring Cloud does for free. |

---

## ADR-D003: Dashboard Accessed Through Gateway

| Field | Detail |
|-------|--------|
| **Status** | ✅ Accepted |
| **Date** | 2026-07-22 |

### Context

> Eureka's dashboard runs on port 8761. Exposing this port to the host breaks the "single entry point" architecture — all traffic should go through Flowero Gate.

### Decision

> **Route the Eureka dashboard through Flowero Gate** at `/eureka/**`. Gate proxies requests to Eureka's internal port (8761). The dashboard is firewall-protected by Gate's auth filter — only authenticated admins can view it.

### Consequences

**Positive:**
- Single domain — `https://panomete.local/eureka` instead of `http://host:8761`
- Auth enforced by Gate — no separate authentication needed on Eureka
- Consistent architecture — every service is behind the Gateway
- CSS/JS/assets load correctly (Gateway path rewriting handles static resources)

**Negative:**
- Eureka's self-referential links may point to internal `host:8761` (cosmetic — navigation still works through Gateway)
- Additional Gateway route configuration

---

## ADR-D004: Graceful Eviction with Self-Preservation

| Field | Detail |
|-------|--------|
| **Status** | ✅ Accepted |
| **Date** | 2026-07-22 |

### Context

> Eureka's self-preservation mode prevents mass eviction of services during network blips. When Eureka doesn't receive heartbeats from a percentage of services, it assumes a network partition and stops evicting — preventing healthy services from being incorrectly removed. In a small homelab deployment (3-4 services), a single service going down could trigger self-preservation mode.

### Decision

> **Keep self-preservation mode enabled.** This is the default and is correct for a homelab. Configure:
> ```yaml
> eureka:
>   server:
>     enable-self-preservation: true
>     renewal-percent-threshold: 0.85
>     eviction-interval-timer-in-ms: 5000
> ```

### Consequences

**Positive:**
- Prevents cascading unregistration during network issues
- Standard behavior — matches production Eureka deployments
- Quick eviction: 5-second interval for checking stale registrations

**Negative:**
- Self-preservation warning in the dashboard may be confusing (red banner: "RENEWALS ARE LESSER THAN THRESHOLD")
- With only 1-2 services, any instance going down triggers the warning (cosmetic — services still work)
- Can mask real service failures if the threshold is met (unlikely with 3+ services)

---

## Related Documents

| Document | Relationship |
|----------|-------------|
| [[flowero_discover/022_API_specification]] | Eureka REST API |
| [[panomete_platform/021_architecture_decision_records]] | Platform-level ADRs (ADR-003: Eureka) |
| [[flowero_discover/012_user_stories]] | Stories driving these decisions |
