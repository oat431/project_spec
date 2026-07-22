---
document_type: Meeting Minutes
version: "0.1"
status: Final
author: "Dev / SA Persona"
created: "2026-07-22"
last_updated: "2026-07-22"
project_name: "Panomete Platform"
meeting_type: "Design Phase — Session Close & Next Steps"
participants: ["PO Persona", "Dev / SA Persona", "Infra (existing)"]
classification: "Internal"
tags: [meeting-minutes, design-phase-complete, handoff, next-steps]
---

# Meeting Minutes — Design Phase Complete

> **Date:** 2026-07-22
> **Type:** Session Close — Phase 2 (Design) Complete
> **Participants:** PO Persona, Dev / SA Persona
> **Status:** ✅ Design documents aligned with requirements and infrastructure. Ready for implementation.

---

## 1. What We Did Today

### Morning — Requirements Review (PO + Dev)

- Reviewed all 12 PO requirement documents (v0.1)
- Dev persona conducted design review (`/grill-me`)
- Identified 9 design decisions that conflicted with existing production infrastructure
- Created [[affect-doc]] — 13 PO documents requiring changes

### Midday — PO Updates (PO Solo)

- PO updated all 12 requirement documents to v0.2
- Applied all 9 design decisions:
  - Guard = Keycloak (no wrapper)
  - Nginx is the edge proxy (not Gate)
  - `*.panomete.com` domain scheme
  - Foundation services on subdomains
  - Gate routes business APIs only
  - Ports: 8000 / 8001 / 8999
  - Shared PostgreSQL 18 + Valkey 9
  - Cloudflare handles TLS
  - Observability deferred to Phase 2
- Created [[po-update-2026-07-22]] — meeting record
- Handed back to Dev persona

### Afternoon — Design Rewrite (Dev/SA Solo)

- Fixed 3 minor PO document issues (version tags, stale text, table labels)
- Rewrote all 11 design documents to v0.2
- All documents now consistent with:
  - Updated PO requirements (v0.2)
  - Actual production infrastructure (Cloudflare, Nginx, PG18, Valkey9, Docker)

---

## 2. Document Inventory (Current State)

### Requirements (PO-Owned) — All v0.2 ✅

| # | Document | Status |
|:--:|----------|:--:|
| 1 | `panomete_platform/README.md` | ✅ v0.2 |
| 2 | `panomete_platform/01_requirement/011_business_objective.md` | ✅ v0.2 |
| 3 | `panomete_platform/01_requirement/012_user_stories.md` | ✅ v0.2 |
| 4 | `panomete_platform/01_requirement/014_stakeholder_analysis.md` | ✅ v0.1 (no changes needed) |
| 5 | `flowero_guard/01_requirement/011_business_objective.md` | ✅ v0.2 |
| 6 | `flowero_guard/01_requirement/012_user_stories.md` | ✅ v0.2 |
| 7 | `flowero_guard/01_requirement/013_acceptance_criteria.md` | ✅ v0.2 |
| 8 | `flowero_discover/01_requirement/011_business_objective.md` | ✅ v0.2 |
| 9 | `flowero_discover/01_requirement/012_user_stories.md` | ✅ v0.2 |
| 10 | `flowero_discover/01_requirement/013_acceptance_criteria.md` | ✅ v0.2 |
| 11 | `flowero_gate/01_requirement/011_business_objective.md` | ✅ v0.2 |
| 12 | `flowero_gate/01_requirement/012_user_stories.md` | ✅ v0.2 |
| 13 | `flowero_gate/01_requirement/013_acceptance_criteria.md` | ✅ v0.2 |

### Design (Dev/SA-Owned) — All v0.2 ✅

| # | Document | Status |
|:--:|----------|:--:|
| 1 | `panomete_platform/02_design/021_architecture_decision_records.md` | ✅ v0.2 — 9 ADRs |
| 2 | `panomete_platform/02_design/025_software_architecture_document.md` | ✅ v0.2 — Full SAD |
| 3 | `panomete_platform/02_design/029_architecture_overview.md` | ✅ v0.2 — One-page map |
| 4 | `flowero_guard/02_design/021_architecture_decision_records.md` | ✅ v0.2 — 7 ADRs |
| 5 | `flowero_guard/02_design/022_API_specification.md` | ✅ v0.2 |
| 6 | `flowero_guard/02_design/023_database_schema_DDL.md` | ✅ v0.2 |
| 7 | `flowero_guard/02_design/024_ERD.md` | ✅ v0.2 |
| 8 | `flowero_discover/02_design/021_architecture_decision_records.md` | ✅ v0.2 — 5 ADRs |
| 9 | `flowero_discover/02_design/022_API_specification.md` | ✅ v0.2 |
| 10 | `flowero_gate/02_design/021_architecture_decision_records.md` | ✅ v0.2 — 7 ADRs |
| 11 | `flowero_gate/02_design/022_API_specification.md` | ✅ v0.2 |

### Meeting Records

| # | Document | Status |
|:--:|----------|:--:|
| 1 | `spec/meeting-minute/affect-doc.md` | ✅ Design review — original findings |
| 2 | `spec/meeting-minute/po-update-2026-07-22.md` | ✅ PO update record |
| 3 | `spec/meeting-minute/next-steps-2026-07-22.md` | ✅ This document |

---

## 3. Architecture at a Glance

```
                    ┌─ auth.panomete.com ───────→ Flowero Guard (Keycloak) :8001
                    │                                Uses shared PostgreSQL 18
Cloudflare → Nginx ─┼─ discovery.panomete.com ───→ Flowero Discover (Eureka) :3999 / :8999
 (TLS)      (edge)  │
                    └─ api.panomete.com ─────────→ Flowero Gate :8000
                                                       Uses shared Valkey 9
                                                       ├─ /api/blog/**  → Cute Gufo :8005
                                                       ├─ /api/todo/**  → Tiny Mchwa :8003
                                                       └─ /api/short/** → Fluffy Mouton :8002
```

**9 Architecture Rules:**
1. Nginx is the edge. Cloudflare handles TLS.
2. Foundation services = subdomains. Business APIs = paths through Gate.
3. Gate validates JWT locally (cached JWKS). Zero per-request calls to Guard.
4. Internal = plain HTTP on trusted Docker network.
5. Rate limiting = Valkey-backed. Survives Gate restarts.
6. PostgreSQL 18 = shared. No dedicated DB container.
7. Guard IS Keycloak. No wrapper service.
8. FE = self-contained containers. Each service owns its full stack.
9. Observability = Phase 2 (Loki + Prometheus + Grafana).

---

## 4. Next Steps — Ordered by Priority

### Priority 1: Infrastructure Prep (DevOps Persona / Self)

> These need to happen BEFORE any code is written.

- [ ] **Nginx — Add 3 server blocks:**
  - `auth.panomete.com` → `http://flowero-guard:8001`
  - `discovery.panomete.com` → `http://flowero-discover:3999`
  - `api.panomete.com` → `http://flowero-gate:8000`
  - Reference: [[08-Reverse-Proxy-Nginx]] and [[08.1-Add-Subdomain]]

- [ ] **Cloudflare — Add DNS records for new subdomains:**
  - `auth.panomete.com`, `discovery.panomete.com`, `api.panomete.com`
  - Reference: [[07-Cloudflare-Tunnel]]

- [ ] **PostgreSQL — Create `keycloak` database:**
  ```sql
  CREATE DATABASE keycloak WITH ENCODING 'UTF8';
  GRANT ALL PRIVILEGES ON DATABASE keycloak TO keycloak;
  ```
  - Reference: [[flowero_guard/023_database_schema_DDL]]

- [ ] **Verify Valkey 9 is reachable from Docker network:**
  - Gate will connect to `valkey:6379` (or `host.docker.internal:6379`)
  - Test: `docker run --rm redis:alpine redis-cli -h host.docker.internal ping`

### Priority 2: Foundation Service Implementation (Dev Persona)

> These are the Phase 1 MVP stories. Start in this order:

**Sprint 1 — Core Infrastructure (8 points):**

| Story | Service | Points | What to Build |
|-------|---------|:---:|---|
| US-001 | Guard | 5 | Keycloak container with Panomete realm, shared PostgreSQL, realm-as-code JSON |
| US-101 | Discover | 3 | Eureka server on :8999/:3999, standalone mode, dashboard via Nginx |

**Sprint 2 — End-to-End Auth (16 points):**

| Story | Service | Points | What to Build |
|-------|---------|:---:|---|
| US-002 | Guard | 3 | Register OAuth2 clients for platform services |
| US-201 | Gate | 5 | Spring Cloud Gateway on :8000, Valkey rate limiting, JSON logging |
| US-202 | Gate | 3 | Route business APIs (blog, short, todo paths) |
| US-203 | Gate | 3 | JWT local validation via Keycloak JWKS, claim header forwarding |
| US-102 | Discover | 3 | Service auto-registration on startup |
| US-103 | Discover | 3 | Service discovery by name (`lb://` resolution) |
| US-003 | Guard | 5 | Login flow + SSO across services |

**Sprint 3 — Production Hardening (11 points):**

| Story | Service | Points | What to Build |
|-------|---------|:---:|---|
| US-204 | Gate | 3 | Route resolution via Eureka (`lb://` caching, failover) |
| US-205 | Gate | 3 | Valkey-backed rate limiting (per-route, per-IP) |
| US-104 | Discover | 2 | Dashboard accessible via Nginx |
| US-004 | Guard | 3 | RBAC (realm roles → `@PreAuthorize`) |

**Should Have (Sprint 3 or later):**

| Story | Service | Points |
|-------|---------|:---:|
| US-005 | Guard | 3 — Service-to-service auth (client credentials) |
| US-006 | Guard | 2 — Token introspection for non-Java services |

### Priority 3: Construction Documents (Dev Persona)

> After code is written:

- [ ] `03_construction/031_README_developer_guide.md` — Setup instructions for each service
- [ ] `03_construction/032_build_scripts.md` — Docker Compose files, Dockerfiles
- [ ] `03_construction/033_dependency_manifest.md` — `pom.xml` / `build.gradle` with pinned versions
- [ ] `03_construction/034_commit_messages_changelog.md` — Conventional commits
- [ ] `03_construction/035_coding_standards_development.md` — Java/Spring conventions

### Priority 4: Future Phases

| Phase | What | When |
|-------|------|------|
| Phase 1.5 | Canary service onboarding (Fluffy Mouton) | After foundation stable |
| Phase 2 | Observability (Loki + Prometheus + Grafana) | After canary live |
| Phase 2 | Business services (Blog, Ledger, Cook Book, Hora) | Incremental |
| Phase 3 | k3s migration | After Docker Compose proven stable |

---

## 5. Key Documents to Read First

> For anyone starting work on this tomorrow:

| Role | Read This First | Why |
|------|----------------|-----|
| **Dev** | [[025_software_architecture_document]] | Full architecture — components, flows, ports, deployment |
| **Dev** | [[029_architecture_overview]] | One-page map — tape to the wall |
| **Dev** | [[flowero_gate/022_API_specification]] | Gateway route table — what paths go where |
| **DevOps** | [[flowero_guard/023_database_schema_DDL]] | PostgreSQL provisioning for Keycloak |
| **DevOps** | SAD §6 (Deployment Architecture) | Docker Compose topology, startup order, resource limits |

---

## 6. Quick Reference — Port Map

| Service | BE Port | FE Port | Domain |
|---------|:---:|:---:|--------|
| Flowero Gate | 8000 | — | `api.panomete.com` |
| Flowero Guard | 8001 | — | `auth.panomete.com` |
| Flowero Discover | 8999 | 3999 | `discovery.panomete.com` |
| Cute Gufo (Blog) | 8005 | 3005 | `blog.panomete.com` |
| Fluffy Mouton (URL) | 8002 | 3002 | `short.panomete.com` |
| Tiny Mchwa (Todo) | 8003 | 3003 | `todo.panomete.com` |

---

## 7. Quick Reference — Story Inventory

| Service | Stories | Points | 🔴 Must | 🟡 Should |
|---------|:---:|:---:|:---:|:---:|
| Flowero Guard | 6 | 21 | 4 | 2 |
| Flowero Discover | 4 | 11 | 3 | 1 |
| Flowero Gate | 5 | 17 | 4 | 1 |
| **Phase 1 Total** | **15** | **49** | **11** | **4** |

---

## Related Documents

| Document | Relationship |
|----------|-------------|
| [[affect-doc]] | Original design review that triggered all changes |
| [[po-update-2026-07-22]] | PO update record |
| [[panomete_platform/README]] | Architecture overview |
| [[panomete_platform/025_software_architecture_document]] | Full SAD |
| [[panomete_platform/021_architecture_decision_records]] | All 9 platform ADRs |

---

> **Status:** Design phase complete. Requirements + Design + Infrastructure all aligned at v0.2.
> **Next:** Start Sprint 1 — Deploy Guard (Keycloak) + Discover (Eureka). See Priority 1 (infrastructure prep) and Priority 2 (sprint backlog).
> **Start date:** 2026-07-23 (tomorrow)
