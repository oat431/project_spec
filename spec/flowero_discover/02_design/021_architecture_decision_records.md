---
document_type: ADR (Architecture Decision Records)
version: "0.2"
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
> **Version:** 0.2 | **Status:** Active — Updated per Design Review 2026-07-22
> **Last Updated:** 2026-07-22

---

## ADR Index

| ADR | Title | Status | Date | Decision |
|-----|-------|--------|------|---------|
| ADR-D001 | Standalone Eureka (Single Node) | ✅ Accepted | 2026-07-22 | Single node, standalone mode |
| ADR-D002 | Eureka Client Auto-Registration | ✅ Accepted | 2026-07-22 | Auto-register via Spring Cloud dependency |
| ADR-D003 | Dashboard via Nginx (Not Gate) | ✅ Accepted | 2026-07-22 | `discovery.panomete.com` → Nginx → Eureka :3999 |
| ADR-D004 | Self-Preservation Enabled | ✅ Accepted | 2026-07-22 | Keep self-preservation for homelab stability |
| ADR-D005 | Dual Ports: BE 8999 + FE 3999 | ✅ Accepted | 2026-07-22 | Separate ports for registration API and dashboard |

---

## ADR-D003: Dashboard via Nginx (Not Gate)

| Field | Detail |
|-------|--------|
| **Status** | ✅ Accepted |
| **Date** | 2026-07-22 |

### Context

> Originally planned to serve the Eureka dashboard through Gate at `/eureka/**`. Design review clarified: foundation services get their own subdomains. Eureka's dashboard is an internal admin tool — it doesn't need Gate's middleware.

### Decision

> **Serve the Eureka dashboard directly through Nginx** at `discovery.panomete.com`. Nginx proxies to Eureka's dashboard on port 3999. The registration API runs on port 8999 (used by services registering, not by browsers).

### Consequences

**Positive:**
- No Gate involvement — simpler routing, easier to debug
- Dedicated subdomain makes it clear this is an infrastructure tool
- Gate scope stays focused on business APIs

**Negative:**
- No JWT auth on the dashboard (can be added via Nginx basic auth or Keycloak integration if needed later)

---

## ADR-D005: Dual Ports — BE 8999, FE 3999

| Field | Detail |
|-------|--------|
| **Status** | ✅ Accepted |
| **Date** | 2026-07-22 |

### Context

> Eureka serves both a REST API (for service registration/discovery) and an HTML dashboard. Using separate ports keeps registration traffic (8999) separate from dashboard traffic (3999) — cleaner for Nginx routing.

### Decision

> **Run Eureka on two ports:**
> - **8999 (BE):** REST API for service registration, discovery queries, heartbeats
> - **3999 (FE):** HTML/JS dashboard, served to `discovery.panomete.com`

### Consequences

**Positive:**
- Nginx routes `discovery.panomete.com` → :3999 (dashboard only)
- Services and Gate call :8999 directly on the Docker network (no Nginx needed)
- Clear separation of concerns

**Negative:**
- Two ports to configure in Eureka
- Eureka's dashboard links to the REST API may need relative URL adjustments

---

*ADR-D001, D002, D004 unchanged from v0.1.*

---

## Related Documents

| Document | Relationship |
|----------|-------------|
| [[flowero_discover/022_API_specification]] | Eureka REST API |
| [[panomete_platform/021_architecture_decision_records]] | Platform-level ADRs |
