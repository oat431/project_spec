---
document_type: Meeting Minutes
version: "0.1"
status: Final
author: "DevOps Persona"
created: "2026-07-23"
last_updated: "2026-07-23"
project_name: "Panomete Platform"
meeting_type: "Sprint 1 Execution Priority — Sequencing Decision"
participants: ["PO Persona", "DevOps Persona"]
classification: "Internal"
tags: [meeting-minutes, sprint-1, execution-priority, sequencing, phase-1]
---

# Meeting Minutes — Sprint 1 Execution Priority

> **Date:** 2026-07-23
> **Type:** Sequencing Decision — Sprint 1 Execution Order
> **Participants:** PO Persona, DevOps Persona
> **Status:** ✅ Decision locked. Execution order agreed.

---

## 1. Purpose

> Decide the deployment/build order for Phase 1 Sprint 1 foundation services (Keycloak, Discovery, Gateway). Sequence must minimize rework and respect dependency chains.

---

## 2. Context

> Following the infrastructure audit (MM04), DevOps specs were upgraded to Java 25 / Spring Boot 4.1 / Gradle (from Java 21 / Boot 3.x / Maven). The existing flowerogate codebase was reviewed and found ~80% aligned with the new specs. PO is ready to begin Sprint 1 execution.

---

## 3. Dependency Analysis

| Service | Runtime Dependencies | Can Start Independently? |
|---------|---------------------|:---:|
| **Keycloak (Guard)** | PostgreSQL only (already running) | ✅ |
| **Discovery (Eureka)** | None — fully standalone | ✅ |
| **Gateway (Gate)** | Keycloak JWKS + Discovery registry + Valkey | ❌ Most dependent |

---

## 4. Discussion

### PO Proposed Order

```
Keycloak → Gateway → Discovery
```

### DevOps Concern

> Deploying Gateway before Discovery creates a rework trap. The gateway needs Eureka for `lb://` route resolution. Without Discovery, routes must be hardcoded (`http://host:port`) and later rewritten to `lb://service-name` when Eureka arrives — touching route config twice.

### Alternative Considered

> Parallel deployment of Keycloak + Discovery (both independent). Viable, but PO prefers to do server work manually and sequentially — one thing at a time.

---

## 5. Decisions Made

| Decision ID | Decision | Rationale | Authority | Impact |
|:-----------:|---------|-----------|:---------:|--------|
| DEC-004 | Sprint 1 execution order: Keycloak → Discovery → Gateway | Deploys in dependency order, least-dependent first. Gateway wires `lb://` routes correctly in one pass — zero route rework. | PO + DevOps | Schedule |

---

## 6. Agreed Execution Order

```
Step 1: Keycloak (Guard)
Step 2: Discovery (Eureka)
Step 3: Gateway (Gate)
```

### Step 1 — Keycloak (Guard)

| Task | Owner | Pre-req |
|------|:---:|---------|
| Create `keycloak` database + role on PostgreSQL | PO (manual) | PostgreSQL healthy ✅ |
| Add Nginx server block: `auth.panomete.com` | PO (manual) | Nginx running ✅ |
| Deploy Keycloak container | PO (manual) | DB + role created |
| Configure `panomete` realm (realm-as-code JSON) | PO (manual) | Container running |
| Verify: OIDC discovery + JWKS + admin console | PO + DevOps | Realm imported |

### Step 2 — Discovery (Eureka)

| Task | Owner | Pre-req |
|------|:---:|---------|
| Add Nginx server block: `discovery.panomete.com` | PO (manual) | Step 1 complete |
| Build Eureka server (Java 25 / Boot 4.1 / Gradle) | PO (manual) | Specs updated ✅ |
| Deploy container (ports 8999 + 3999) | PO (manual) | Image built |
| Verify: dashboard + REST API | PO + DevOps | Container healthy |

### Step 3 — Gateway (Gate)

| Task | Owner | Pre-req |
|------|:---:|---------|
| Adapt flowerogate codebase (7 fixes from review) | PO (manual) | Step 1 + 2 complete |
| Add Nginx server block: `api.panomete.com` | PO (manual) | — |
| Deploy container | PO (manual) | Image built |
| Verify: 401 rejection + JWKS + Eureka `lb://` | PO + DevOps | All deps healthy |

---

## 7. Flowerogate Adaptation Checklist (Step 3 Reference)

> From the code review — these 7 fixes are needed before the gateway can deploy against the new architecture.

| # | Fix | Detail |
|---|-----|--------|
| 1 | Realm name | `flowerogate` → `panomete` in all configs |
| 2 | Redis host | `host.docker.internal` → `local-valkey` + add password |
| 3 | Docker network | Add `db-network` to compose |
| 4 | Routes | Demo services → Panomete `lb://` routes |
| 5 | Eureka | Enable + point to `flowero-discover:8999` |
| 6 | CORS origins | Update to Panomete domains |
| 7 | Post-login redirect | Remove hardcoded short-link URL |

---

## 8. Action Items

| Action ID | Action | Owner | Due | Status |
|-----------|--------|-------|-----|--------|
| ACT-009 | Execute Step 1: Deploy Keycloak | PO | Sprint 1 | ⬜ Open |
| ACT-010 | Execute Step 2: Build + Deploy Discovery | PO | Sprint 1 | ⬜ Open |
| ACT-011 | Execute Step 3: Adapt + Deploy Gateway | PO | Sprint 1 | ⬜ Open |
| ACT-012 | DevOps available for verification at each step | DevOps | Sprint 1 | ⬜ Open |

> **Note:** PO will do all server-side work manually. DevOps provides documentation, verification commands, and support on request.

---

## 9. Success Criteria — Sprint 1 Complete When

- [ ] `https://auth.panomete.com/admin` — Keycloak admin console accessible
- [ ] `https://auth.panomete.com/realms/panomete/.well-known/openid-configuration` — OIDC discovery returns valid config
- [ ] `https://auth.panomete.com/realms/panomete/protocol/openid-connect/certs` — JWKS returns RSA keys
- [ ] `https://discovery.panomete.com` — Eureka dashboard loads
- [ ] `http://localhost:8999/actuator/health` — Eureka health UP
- [ ] `https://api.panomete.com/api/blog/posts` — Returns 401 (JWT validation active)
- [ ] `http://localhost:8000/actuator/health` — Gate health UP with Valkey + Eureka components

---

## Related Documents

| Document | Relationship |
|----------|-------------|
| [[MM03_next-steps_20260722]] | Sprint backlog and story inventory |
| [[MM04_devops-infra-audit_20260723]] | Infrastructure audit + discrepancy findings |
| `panomete_platform/05_devops/052_deployment_plan` | Step-by-step deployment procedures |
| `flowero_guard/05_devops/052_deployment_plan` | Keycloak-specific deployment |
| `flowero_discover/05_devops/052_deployment_plan` | Discovery-specific deployment |
| `flowero_gate/05_devops/052_deployment_plan` | Gateway-specific deployment |

---

> **Status:** Execution order locked. PO begins manual server work starting with Keycloak. DevOps on standby for verification.
