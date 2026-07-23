---
document_type: Meeting Minutes
version: "0.1"
status: Final
author: "DevOps Persona"
created: "2026-07-24"
last_updated: "2026-07-24"
project_name: "Panomete Platform"
meeting_type: "Phase 2 Planning — Foundation Hardening & Next Steps"
participants: ["DevOps Persona", "PO Persona"]
classification: "Internal"
tags: [meeting-minutes, phase-2, planning, ci-cd, observability, alerting, backup]
---

# Meeting Minutes — Phase 2 Planning

> **Date:** 2026-07-24
> **Type:** Phase 2 Planning Briefing
> **From:** DevOps Persona
> **To:** PO Persona (for review and prioritization)
> **Status:** ✅ Sprint 1 complete. Phase 2 scope proposed.

---

## 1. Purpose

> Review what Sprint 1 delivered, identify gaps in the foundation, and propose Phase 2 priorities for PO approval. This is a planning document — no execution yet.

---

## 2. Sprint 1 — Complete

| Service | Domain | Status |
|---------|--------|:------:|
| Flowero Guard (Keycloak) | `auth.panomete.com` | ✅ Deployed, verified |
| Flowero Discover (Eureka) | `discovery.panomete.com` | ✅ Deployed, verified |
| Flowero Gate (Gateway) | `api.panomete.com` | ✅ Deployed, verified |

**What works today:**
- Users can authenticate via Keycloak (OAuth2/OIDC)
- Services auto-register with Eureka
- Gate validates JWTs locally, rate-limits via Valkey, routes via `lb://`
- All 3 services accessible via HTTPS through Cloudflare → Nginx

**What doesn't work yet:**
- No business services deployed (Gate returns 502 — no backends registered)
- No CI/CD — deployments are manual (scp + docker compose build)
- No monitoring — no dashboards, no alerts
- No automated backups

---

## 3. The Problem — Manual Deployment

Current deployment workflow:

```
Dev finishes code
  → DevOps SCPs project to server (token-expensive, slow)
  → DevOps manually merges into docker-compose.platform.yml
  → DevOps runs docker compose up --build
  → Prone to YAML errors (happened twice during Sprint 1)
  → No tests run before deploy
  → No version tracking on images
  → No rollback capability
```

**This is the #1 bottleneck.** Every new service or update requires manual DevOps intervention. It doesn't scale.

---

## 4. Phase 2 Proposal — 6 Initiatives

### Initiative A: CI/CD Pipeline + Image Registry 🔴

> **Priority:** Must Have — unblocks everything else

**What:**
- GitHub Actions workflow for each service: lint → test → build → push to GHCR
- Docker images versioned in GitHub Container Registry (`ghcr.io/oat431/...`)
- Deploy via SSH: `docker compose pull && docker compose up -d`
- No more manual scp — deploying = `git push`

**Outcome:**
- New version deployed in <2 minutes after push
- Tests run automatically before deploy
- Rollback = pull previous image tag
- Works for all services (foundation + business + observability containers)

**Effort:** Medium — GitHub Actions YAML, SSH deploy key, GHCR setup
**Depends on:** Nothing
**Unblocks:** Every subsequent initiative (observability, business services, future phases)

---

### Initiative B: Observability Stack 🟡

> **Priority: Should Have — but strongly recommended before business services**

**What:**

| Tool | Purpose | Port | Domain |
|------|---------|:---:|--------|
| **Prometheus** | Scrape metrics from all services (`/actuator/prometheus`) | 9090 | `metrics.panomete.com` |
| **Grafana** | Dashboards — CPU, memory, request rate, error rate, latency, JVM | 3000 | `grafana.panomete.com` |
| **Loki** | Centralized log aggregation — all container logs searchable in one place | 3100 | — (internal) |

**What you get:**
- Visual dashboards showing real-time health of every service
- Search logs across all containers from one UI (no more `docker logs -f` per container)
- Historical data — "what happened at 3 AM?"
- All 3 foundation services already expose metrics and structured logs — just need collectors

**Effort:** Medium — 3 containers (Docker Compose), Prometheus scrape config, Grafana dashboard JSON
**Depends on:** Nothing (but benefits from CI/CD for easy deployment)
**Specs already written:** SAD §7 (Observability) — deferred from Phase 1

---

### Initiative C: Alerting 🟡

> **Priority: Should Have — comes after observability**

**What:** Grafana alert rules that notify when things break:

| Alert | Trigger | Severity |
|-------|---------|:---:|
| Service down | Health check fails for 1 min | Critical |
| High error rate | 5xx responses > 5% for 5 min | High |
| High memory | Container memory > 80% | Medium |
| Disk space low | Disk usage > 80% | High |
| Valkey down | Valkey ping fails | High |
| Eureka empty | Zero registered services | Critical |

**Notification channel:** Telegram/Discord webhook (PO preference needed)

**Effort:** Low — Grafana alert rules + webhook config (after Initiative B)
**Depends on:** Initiative B (Grafana must be running)

---

### Initiative D: Backup Automation 🟢

> **Priority: Nice to Have — but data loss risk without it**

**What:**

| Data | Method | Frequency | Destination |
|------|--------|-----------|-------------|
| PostgreSQL (all DBs) | `pg_dumpall` script | Daily 3 AM | `~/backups/` → rclone → OneDrive |
| Keycloak realm | JSON export (already version-controlled) | On change | Git |
| Docker volumes | `tar` script | Weekly | `~/backups/` → rclone |

> Notes say rclone is configured but "no cron yet". This automates it.

**Effort:** Low — bash script + cron entry
**Depends on:** Nothing

---

### Initiative E: Uptime Monitoring 🟢

> **Priority: Nice to Have — visual health dashboard**

**What:** A lightweight status page showing all `*.panomete.com` services as green/red.

| Tool | Purpose | Complexity |
|------|---------|:---:|
| **Uptime Kuma** | Pings all URLs every 30s, shows status page | Low |
| **Homarr** | Dashboard aggregating Portainer, Grafana, Uptime Kuma | Low |

**Effort:** Low — single Docker container
**Depends on:** Nothing

---

### Initiative F: First Business Service Onboarding 🟢

> **Priority: Nice to Have — proves the platform works end-to-end**

**What:** Deploy Cute Gufo (Blog) as the first business service through Gate.

- Registers with Eureka automatically
- Gate routes `/api/blog/**` → `lb://cute-gufo`
- JWT validated at Gate, claims forwarded as headers
- Blog backend trusts `X-User-Id` and `X-User-Roles`

**This proves the foundation works end-to-end.** Without a business service, the platform is just infrastructure with no traffic.

**Effort:** Medium — depends on Cute Gufo code being ready
**Depends on:** Ideally CI/CD (Initiative A) so deployment isn't manual

---

## 5. Recommended Execution Order

```
Phase 2a: Foundation Hardening (DevOps-driven)
├── Step 1: CI/CD + GHCR (A)          ← solves the manual deploy problem
├── Step 2: Prometheus + Grafana (B)   ← see what's happening
├── Step 3: Loki log aggregation (B)   ← stop docker logs per-container
├── Step 4: Alerting (C)               ← know when things break
├── Step 5: Backup cron (D)            ← protect the data
└── Step 6: Uptime Kuma (E)            ← visual health dashboard

Phase 2b: First Business Service (Dev-driven)
└── Onboard Cute Gufo (Blog) (F)       ← proves platform end-to-end

Phase 3: Scale (Future)
├── k3s migration
└── More business services (Todo, Short, Ledger, Recipe, Hora)
```

**The key insight:** Do CI/CD first. Every subsequent service — observability containers, business services, everything — deploys through the pipeline instead of manual scp. It's the one-time investment that pays off forever.

---

## 6. Dependency Graph

```
A (CI/CD) ──────────────────────────────────┐
                                            ▼
B (Observability) ──────► C (Alerting)    All future
                          deployments
D (Backup) ────────── independent           │
                                            ▼
E (Uptime Kuma) ────── independent        F (Business service)
                                            │
                                            ▼
                                         Phase 3 (k3s)
```

| Initiative | Depends On | Unblocks |
|-----------|------------|----------|
| A (CI/CD) | Nothing | B, C, D, E, F, Phase 3 |
| B (Observability) | Ideally A | C |
| C (Alerting) | B | Nothing (value add) |
| D (Backup) | Nothing | Nothing (risk mitigation) |
| E (Uptime) | Nothing | Nothing (convenience) |
| F (Business) | Ideally A | Phase 3 validation |

---

## 7. Effort Estimates

| Initiative | DevOps Work | Dev Work | Server Config | Total Effort |
|-----------|:---:|:---:|:---:|:---:|
| A — CI/CD + GHCR | High | Low | Low (SSH key) | Medium |
| B — Observability | Medium | None | Medium (3 containers) | Medium |
| C — Alerting | Low | None | Low (Grafana config) | Low |
| D — Backup | Low | None | Low (cron + script) | Low |
| E — Uptime | Low | None | Low (1 container) | Low |
| F — Business Service | Low (deploy) | High (code) | Low (Nginx) | Medium |

---

## 8. Decisions Needed from PO

| Decision ID | Question | Options |
|:-----------:|---------|---------|
| DEC-005 | Which initiatives to include in Phase 2? | All 6 / subset / different order |
| DEC-006 | Notification channel for alerts? | Telegram / Discord / Email / Slack |
| DEC-007 | Should CI/CD auto-deploy on push to `main`, or require manual trigger? | Auto / Manual approval |
| DEC-008 | Which business service to onboard first? | Cute Gufo (Blog) / Tiny Mchwa (Todo) / Fluffy Mouton (Short) |

---

## 9. Action Items

| Action ID | Action | Owner | Status |
|-----------|--------|-------|--------|
| ACT-018 | PO reviews Phase 2 proposal and makes priority decisions | PO | ⬜ Open |
| ACT-019 | PO decides notification channel (DEC-006) | PO | ⬜ Open |
| ACT-020 | PO decides CI/CD auto vs manual deploy (DEC-007) | PO | ⬜ Open |
| ACT-021 | PO decides first business service (DEC-008) | PO | ⬜ Open |
| ACT-022 | DevOps creates detailed execution plan once priorities set | DevOps | ⬜ Pending DEC-005 |

---

## 10. Documents for PO Review

| Document | Path | Why PO Needs It |
|----------|------|-----------------|
| SAD §7 (Observability) | `panomete_platform/02_design/025_software_architecture_document.md` | Observability was designed but deferred |
| SAD §9 (Open Issues) | Same | Risks identified during design |
| MM05 (Sprint 1 Priority) | `meeting-minute/MM05_sprint1-priority_20260723.md` | Sprint 1 execution context |
| MM06 (Gateway Handoff) | `meeting-minute/MM06_gateway-handoff_20260724.md` | How Dev/DevOps collaboration works |
| Platform README | `panomete_platform/README.md` | Overall architecture overview |
| Home Lab App status | `F:\obsidian_note\oralita_md\Quick Note\Home Lab App.md` | Current service inventory |

---

## Related Documents

| Document | Relationship |
|----------|-------------|
| [[MM03_next-steps_20260722]] | Original sprint backlog |
| [[MM04_devops-infra-audit_20260723]] | Infrastructure audit |
| [[MM05_sprint1-priority_20260723]] | Sprint 1 execution decision |
| [[MM06_gateway-handoff_20260724]] | Gateway adaptation handoff |

---

> **Status:** Phase 2 proposal ready for PO review. Sprint 1 foundation is live and stable. No execution begins until PO confirms priorities.
