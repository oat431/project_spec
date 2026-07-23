---
document_type: Meeting Minutes
version: "1.0"
status: Final
author: "PO (Product Owner)"
created: "2026-07-24"
last_updated: "2026-07-24"
project_name: "Panomete Platform"
meeting_type: "Phase 2 Planning — PO Decisions & Handoff to DevOps"
participants: ["PO Persona", "User (representing PO)"]
classification: "Internal"
tags: [meeting-minutes, phase-2, po-decisions, devops-handoff, ci-cd, observability]
---

# Meeting Minutes — Phase 2 PO Decisions & DevOps Handoff

> **Date:** 2026-07-24
> **Type:** PO Decision Record + Handoff
> **From:** PO Persona
> **To:** DevOps Persona
> **Status:** ✅ Decisions made. Phase 2 plan approved. Ready for DevOps execution.

---

## 1. Purpose

> PO reviewed the Phase 2 proposal from DevOps ([[MM07_phase2-planning_20260724]]), made 4 decisions, produced a Phase 2 plan, and created a platform-level construction overview. This document hands those decisions to DevOps for execution.

---

## 2. Context

- **Phase 1:** ✅ Complete. Guard, Discover, Gate all deployed and verified.
- **Current state:** Manual deployments via SCP. No monitoring. No CI/CD. No automated backups.
- **DevOps proposal:** 6 initiatives (A–F) for Phase 2.
- **PO decision:** Approve initiatives A–E (infra hardening). Defer F (business service) to Phase 3.

---

## 3. PO Decisions

| Decision ID | Question | PO Choice | Rationale |
|:-----------:|---------|-----------|-----------|
| **DEC-005** | Which initiatives in Phase 2? | **A + B + C + D + E** (all infra hardening). Defer F (business service) to Phase 3. | Platform foundation is already a strong portfolio piece. Hardening with CI/CD + observability makes it even stronger. Do this right before adding business features. |
| **DEC-006** | Alert notification channel? | **Discord webhook** | User has existing Discord server. Richer formatting than Telegram. |
| **DEC-007** | CI/CD auto-deploy or manual? | **Manual approval** | Push to `main` triggers CI (build + test + push to GHCR). Deploy to homelab requires manual `workflow_dispatch` trigger. Safety-first for single-developer homelab. |
| **DEC-008** | First business service? | **Deferred to Phase 3** | Not deciding now. Phase 2 is pure infrastructure hardening. |

---

## 4. Phase 2 Scope — Approved

| Initiative | Name | Priority | Owner |
|:----------:|------|:--------:|:-----:|
| **A** | CI/CD Pipeline + GHCR | 🔴 Must Have | DevOps |
| **B** | Observability (Prometheus + Grafana + Loki) | 🔴 Must Have | DevOps |
| **C** | Alerting (Grafana → Discord) | 🟡 Should Have | DevOps |
| **D** | Backup Automation (pg_dumpall + rclone) | 🟡 Should Have | DevOps |
| **E** | Uptime Monitoring (Uptime Kuma) | 🟢 Nice to Have | DevOps |

**Explicitly OUT of scope:**
- Initiative F (Business Service Onboarding) → Phase 3
- k3s migration → Phase 3+
- Trivy security scanning → optional add-on during Phase 2

---

## 5. Execution Plan

### Sprint 2 (Initiative A + D)

| Step | Initiative | What Ships |
|------|-----------|------------|
| 1 | A — CI/CD | GitHub Actions workflows for Guard, Discover, Gate. CI on push (compile + test + lint + build + push to GHCR). Separate deploy workflow (`workflow_dispatch`). SSH deploy key. Auto-rollback on smoke test failure. |
| 2 | D — Backup | `pg_dumpall` cron at 3:00 AM daily. 7-day local retention. rclone → OneDrive. Weekly Docker volume backup (Sunday 4 AM, 4-week retention). |

### Sprint 3 (Initiative B + C + E)

| Step | Initiative | What Ships |
|------|-----------|------------|
| 3 | B — Observability | Prometheus (:9090, `metrics.panomete.com`). Grafana (:3000, `grafana.panomete.com`). Loki (:3100, internal). Scrape configs for all 3 foundation services. Dashboards: Platform Overview, JVM Health, Eureka Registry, Gate Traffic. |
| 4 | C — Alerting | Grafana alert rules → Discord webhook. 6 alerts: service down, high error rate, high memory, disk space, Valkey down, Eureka empty. |
| 5 | E — Uptime | Uptime Kuma (:3001, `status.panomete.com`). Pings all `*.panomete.com` URLs every 30s. |
| 6 | Infra | Nginx routes + Cloudflare Tunnel config for 4 new subdomains (`metrics`, `grafana`, `status`, + Loki internal). |

---

## 6. New Ports & Domains (Phase 2)

| Service | Port | Domain | Initiative |
|---------|:----:|--------|:----------:|
| Prometheus | 9090 | `metrics.panomete.com` | B |
| Grafana | 3000 | `grafana.panomete.com` | B |
| Loki | 3100 | — (internal) | B |
| Uptime Kuma | 3001 | `status.panomete.com` | E |

> **Note:** Ports 3000-3001 use the self-hosted app FE range. Prometheus 9090 is the standard Prometheus port.

---

## 7. Critical Change to Existing CI/CD Docs

> ⚠️ **The existing per-service CI/CD docs (`051_CICD_pipeline_configuration.md`) currently show auto-deploy on push to `main`. This must be changed.**

**What needs to change (action P2-001):**

| Before | After |
|--------|-------|
| Single workflow: push → build → deploy | **Two workflows:** CI (push → build + push to GHCR) and Deploy (`workflow_dispatch` → SSH + smoke test) |
| Deploy runs automatically after build | Deploy requires **manual trigger** (GitHub Actions `workflow_dispatch` or environment protection rule on `homelab`) |
| `on: push: branches: [main]` for both build and deploy | CI: `on: push: branches: [main]`. Deploy: `on: workflow_dispatch` |

**Files to update:**
- `spec/flowero_guard/05_devops/051_CICD_pipeline_configuration.md`
- `spec/flowero_discover/05_devops/051_CICD_pipeline_configuration.md`
- `spec/flowero_gate/05_devops/051_CICD_pipeline_configuration.md`

---

## 8. Action Items

| Action ID | Action | Owner | Priority | Depends On |
|-----------|--------|:-----:|:--------:|:----------:|
| **P2-001** | Update CI/CD docs: split deploy to `workflow_dispatch` (manual approval) | DevOps | 🔴 | — |
| **P2-002** | Create GitHub Actions CI workflows (guard, discover, gate) | DevOps | 🔴 | P2-001 |
| **P2-003** | Create GitHub Actions deploy workflows (`workflow_dispatch`) | DevOps | 🔴 | P2-002 |
| **P2-004** | Set up GHCR + SSH deploy key (`HOMELAB_SSH_KEY`) | DevOps | 🔴 | — |
| **P2-005** | Write backup script (`pg_dumpall` + rclone → OneDrive) | DevOps | 🟡 | — |
| **P2-006** | Configure cron (daily 3AM db, weekly volumes) | DevOps | 🟡 | P2-005 |
| **P2-007** | Deploy Prometheus + scrape config for 3 foundation services | DevOps | 🔴 | P2-004 |
| **P2-008** | Deploy Grafana + import dashboards (4 dashboards) | DevOps | 🔴 | P2-007 |
| **P2-009** | Deploy Loki + Promtail (log aggregation) | DevOps | 🔴 | P2-007 |
| **P2-010** | Configure Grafana Discord webhook contact point | DevOps | 🟡 | P2-008 |
| **P2-011** | Create 6 alert rules in Grafana | DevOps | 🟡 | P2-010 |
| **P2-012** | Deploy Uptime Kuma + configure 7 URL monitors | DevOps | 🟢 | — |
| **P2-013** | Configure Nginx routes for new subdomains | DevOps | 🔴 | P2-007, P2-008, P2-012 |
| **P2-014** | Configure Cloudflare Tunnel for new subdomains | DevOps | 🔴 | P2-013 |

---

## 9. Definition of Done — Phase 2

Phase 2 is complete when ALL of the following are true:

- [ ] CI/CD pipeline builds, tests, and pushes images to GHCR for all 3 foundation services
- [ ] Deploy requires manual approval (`workflow_dispatch`) — NOT auto-deploy
- [ ] Smoke tests run after deploy and auto-rollback on failure
- [ ] Prometheus scrapes metrics from all 3 foundation services
- [ ] Grafana dashboards show JVM health, request rate, error rate, latency
- [ ] Loki aggregates logs from all containers
- [ ] Grafana alerts send to Discord webhook when triggered
- [ ] Backup cron runs daily at 3 AM, pushes to OneDrive
- [ ] Uptime Kuma monitors all `*.panomete.com` URLs
- [ ] All new services deployed through CI/CD (not manual scp)

---

## 10. Phase 2 → Phase 3 Transition Criteria

Before starting Phase 3 (Business Service Onboarding):

1. CI/CD is the only deployment path (no manual scp)
2. Observability dashboards are green for 7 consecutive days
3. Alerting is tested (at least 1 alert fired and received in Discord)
4. Backup restore has been verified (restore from backup, confirm data integrity)
5. PO decides on first business service (DEC-008)

---

## 11. Documents Produced This Session

| Document | Path | Purpose |
|----------|------|---------|
| **Phase 2 Plan** | `plan/phase2-foundation-hardening.md` | Approved plan with 5 initiatives, 14 action items, DoD |
| **Platform Construction Overview** | `spec/panomete_platform/03_construction/031_README_developer_guide.md` | Platform-level dev guide — monorepo structure, build profiles, coding standards |
| **Platform README (updated)** | `spec/panomete_platform/README.md` | Added observability layer to diagram, updated statuses to ✅ Deployed |
| **Root README (updated)** | `README.md` | Full rewrite — service catalog, infrastructure table, architecture diagram, status badges |
| **This meeting minute** | `spec/meeting-minute/MM08_phase2-po-decisions_20260724.md` | PO decisions + DevOps handoff |

---

## 12. Sprint 2 Kickoff Checklist

> DevOps can start Sprint 2 immediately. No blockers.

- [x] Phase 2 plan approved by PO
- [x] All 4 decisions made (DEC-005 through DEC-008)
- [x] CI/CD manual deploy requirement documented (DEC-007)
- [x] Discord webhook chosen for alerts (DEC-006)
- [x] Backup retention policy set (7 days local → OneDrive)
- [x] Phase 2 plan accessible at `plan/phase2-foundation-hardening.md`
- [ ] P2-001: DevOps updates existing CI/CD docs to manual deploy model
- [ ] P2-004: DevOps sets up GHCR + SSH key

---

## Related Documents

| Document | Relationship |
|----------|-------------|
| [[MM07_phase2-planning_20260724]] | DevOps original Phase 2 proposal (input to this meeting) |
| [[../plan/phase2-foundation-hardening]] | Approved Phase 2 plan (output of this meeting) |
| [[../spec/panomete_platform/README]] | Updated platform architecture overview |
| [[../spec/panomete_platform/03_construction/031_README_developer_guide]] | Platform construction overview |
| [[MM02_po-update_20260722]] | Previous PO update (Phase 1 requirements alignment) |

---

> **Next Step:** DevOps persona reviews this document → begins Sprint 2 (P2-001 through P2-006). No further PO input needed until Phase 2 DoD review.
> **Cross-persona handoff:** This document is the contract between PO (decisions made) and DevOps (execution begins).
