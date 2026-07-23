# Panomete Platform

> Production-grade microservice platform for a personal homelab. Built to showcase platform engineering and software architecture skills.

## Platform Overview

```mermaid
flowchart TB
    subgraph External["☁️ Cloudflare"]
        Client["Browser / API Client"]
        CF["Cloudflare Tunnel<br>TLS Termination"]
    end

    subgraph Edge["🛡️ Nginx Reverse Proxy"]
        Nginx["Nginx<br>Subdomain Router"]
    end

    subgraph Foundation["🔒 Foundation Services — ✅ Deployed"]
        Guard["Flowero Guard<br>Keycloak IAM :8001<br>auth.panomete.com"]
        Discover["Flowero Discover<br>Eureka Registry :8999/:3999<br>discovery.panomete.com"]
    end

    subgraph Gateway["Flowero Gate — ✅ Deployed"]
        Gate["Spring Cloud Gateway :8000<br>api.panomete.com<br>JWT Auth · Rate Limit · Routing"]
    end

    subgraph Observability["📊 Observability — Phase 2 🛠️"]
        Prom["Prometheus :9090<br>metrics.panomete.com"]
        Grafana["Grafana :3000<br>grafana.panomete.com"]
        Loki["Loki :3100<br>(internal)"]
        Kuma["Uptime Kuma :3001<br>status.panomete.com"]
    end

    subgraph Business["📦 Business Services — Phase 3+"]
        Blog["Cute Gufo 🦉<br>Blog :8005 / :3005"]
        URL["Fluffy Mouton 🐑<br>URL :8002 / :3002"]
        Todo["Tiny Mchwa 🐜<br>Todo :8003 / :3003"]
        Ledger["Big Schwein 🐷<br>Ledger :8004 / :3004"]
        Cook["Shy Ardilla 🐿️<br>Cook :8006 / :3006"]
        Hora["White Jelen 🦌<br>Hora :8007 / :3007"]
    end

    Client -->|"HTTPS"| CF
    CF -->|"cloudflared"| Nginx

    Nginx -->|"auth.panomete.com"| Guard
    Nginx -->|"discovery.panomete.com"| Discover
    Nginx -->|"api.panomete.com"| Gate
    Nginx -->|"metrics.panomete.com"| Prom
    Nginx -->|"grafana.panomete.com"| Grafana
    Nginx -->|"status.panomete.com"| Kuma

    Gate -->|"/api/blog/**"| Blog
    Gate -->|"/api/short/**"| URL
    Gate -->|"/api/todo/**"| Todo
    Gate -->|"/api/ledger/**"| Ledger
    Gate -->|"/api/recipe/**"| Cook
    Gate -->|"/api/hora/**"| Hora

    Guard -.->|"JWT validation (JWKS)"| Gate
    Gate -.->|"resolves routes via"| Discover

    Guard -->|"registers"| Discover
    Blog -->|"registers"| Discover
    URL -->|"registers"| Discover
    Todo -->|"registers"| Discover
    Ledger -->|"registers"| Discover
    Cook -->|"registers"| Discover
    Hora -->|"registers"| Discover

    Prom -.->|"scrapes /actuator"| Guard
    Prom -.->|"scrapes /actuator"| Discover
    Prom -.->|"scrapes /actuator"| Gate
    Loki -.->|"collects logs"| Foundation
    Loki -.->|"collects logs"| Gateway
    Grafana -->|"queries"| Prom
    Grafana -->|"queries"| Loki
    Grafana -.->|"alerts →"| Discord["Discord Webhook"]

    style External fill:#f6821f,stroke:#e06a10,color:#fff
    style Edge fill:#009639,stroke:#007a2e,color:#fff
    style Foundation fill:#2c3e50,stroke:#34495e,color:#ecf0f1
    style Gateway fill:#e74c3c,stroke:#c0392b,color:#fff
    style Observability fill:#6c3483,stroke:#5b2c6f,color:#ecf0f1
    style Business fill:#1a1a2e,stroke:#333,color:#aaa
    style Guard fill:#3498db,stroke:#2980b9,color:#fff
    style Discover fill:#27ae60,stroke:#1e8449,color:#fff
    style Gate fill:#e74c3c,stroke:#c0392b,color:#fff
    style CF fill:#f6821f,stroke:#e06a10,color:#fff
    style Nginx fill:#009639,stroke:#007a2e,color:#fff
    style Prom fill:#e67e22,stroke:#d35400,color:#fff
    style Grafana fill:#f39c12,stroke:#d68910,color:#fff
    style Loki fill:#8e44ad,stroke:#7d3c98,color:#fff
    style Kuma fill:#1abc9c,stroke:#16a085,color:#fff
    style Discord fill:#5865F2,stroke:#4752C4,color:#fff
```

## Foundation Services (Phase 1 MVP)

| Service | Code Name | Technology | Port | Domain | Role | Status |
|---------|-----------|-----------|------|--------|------|--------|
| **Flowero Guard** | — | Keycloak (uses shared PostgreSQL 18) | 8001 | `auth.panomete.com` | Identity & Access Management — issues OAuth2/OIDC tokens | ✅ Deployed |
| **Flowero Discover** | — | Spring Cloud Eureka | 8999 (BE) / 3999 (FE) | `discovery.panomete.com` | Service Registry & Discovery | ✅ Deployed |
| **Flowero Gate** | — | Spring Cloud Gateway | 8000 | `api.panomete.com` | API Gateway — JWT validation, Valkey-backed rate limiting, route to business services | ✅ Deployed |

## Business Services (Future Phases)

| Focus | Code Name | Animal | Language | Subdomain | Port BE | Port FE | Status |
|-------|-----------|--------|----------|-----------|---------|---------|--------|
| Blog | Cute Gufo | Owl | Go | `blog.panomete.com` | 8005 | 3005 | 📋 Spec Ready |
| URL Shortener | Fluffy Mouton | Sheep | TypeScript | `short.panomete.com` | 8002 | 3002 | 📋 Spec Ready |
| Todo List | Tiny Mchwa | Ant | TypeScript | `todo.panomete.com` | 8003 | 3003 | ✅ Spec Complete |
| Ledger | Big Schwein | Pig | TypeScript | `ledger.panomete.com` | 8004 | 3004 | ❌ Not Started |
| Cook Book | Shy Ardilla | Squirrel | TypeScript | `recipe.panomete.com` | 8006 | 3006 | ❌ Not Started |
| Hora | White Jelen | Deer | TypeScript | `hora.panomete.com` | 8007 | 3007 | ❌ Not Started |

## Tech Stack

| Layer | Choice | Rationale |
|-------|--------|-----------|
| **Edge / TLS** | Cloudflare Tunnel + Nginx | Cloudflare handles TLS termination + DDoS protection. Nginx does subdomain-based routing to internal services. Both already running in production. |
| **Foundation Language** | Java 25 / Spring Boot 4.1.x | Latest LTS Java. Spring ecosystem provides native Gateway (Spring Cloud Gateway), Security (OAuth2 Resource Server), and Discovery (Eureka) integrations |
| **Identity** | Keycloak | Production-grade OSS IAM with full OAuth2/OIDC support. Uses shared PostgreSQL 18 for persistence. |
| **API Gateway** | Spring Cloud Gateway | Reactive, non-blocking. Routes business APIs only (`api.panomete.com`). Validates JWT against Keycloak JWKS. |
| **Service Discovery** | Spring Cloud Netflix Eureka | Simplest path with Spring Boot; embeddable, no external dependency |
| **Rate Limiting** | Valkey 9 (shared instance) | Replaces in-memory rate limiting. Limits persist across Gate restarts. Already running in production. |
| **Deployment** | Docker Compose → Kubernetes (k3s) | Compose for homelab simplicity; K8s for portfolio growth |
| **CI/CD** | GitHub Actions + GHCR | Per-service pipelines: lint → test → build → push to GHCR. Manual deploy approval via `workflow_dispatch`. Phase 2. |
| **Observability** | Phase 2: Actuator + Prometheus + Grafana + Loki | In progress. Prometheus scrapes `/actuator/prometheus`, Grafana visualizes, Loki aggregates logs. Discord webhook for alerts. |

## Existing Infrastructure (Already Live)

| Component | Status | Notes |
|-----------|--------|-------|
| Docker + Compose | ✅ | All services run as containers |
| Cloudflare Tunnel | ✅ | TLS termination + external ingress |
| Nginx Reverse Proxy | ✅ | Subdomain-based routing |
| PostgreSQL 18 | ✅ | Shared database for Guard + business services |
| Valkey 9 | ✅ | Shared cache for Gate rate limiting |
| MongoDB 8 | ✅ | Available for document-oriented services |
| SeaweedFS S3 | ✅ | Object storage |
| Tailscale | ✅ | Secure remote access |
| UFW + Fail2ban | ✅ | Host-level security |

## Project Structure

```mermaid
---
title: Panomete Platform — Project Structure
---
treeView-beta
  spec/
  ├── 📋 panomete_platform/  → Platform-Level Docs
  │   ├── README.md  → Architecture Overview (this file)
  │   ├── 01_requirement/
  │   │   ├── 011_business_objective.md  → 5 SMART Objectives
  │   │   ├── 012_user_stories.md  → Umbrella Story Map
  │   │   └── 014_stakeholder_analysis.md  → 8 Stakeholders
  │   └── 03_construction/
  │       └── 031_README_developer_guide.md  → Platform Dev Guide
  ├── 🔒 flowero_guard/  → Keycloak IAM · :8001 · auth.panomete.com · ✅ Deployed
  │   ├── 01_requirement/  → 3 Stories · 30 BDD ACs
  │   ├── 02_design/  → ADR + API + DDL + ERD
  │   ├── 03_construction/  → 6 docs (031-036)
  │   └── 05_devops/  → CI/CD + Deploy + Runbook
  ├── 🟢 flowero_discover/  → Eureka Registry · :8999/:3999 · discovery.panomete.com · ✅ Deployed
  │   ├── 01_requirement/  → 4 Stories · 16 BDD ACs
  │   ├── 02_design/  → ADR + API
  │   ├── 03_construction/  → 6 docs (031-036)
  │   └── 05_devops/  → CI/CD + Deploy + Runbook
  ├── 🔴 flowero_gate/  → API Gateway · :8000 · api.panomete.com · ✅ Deployed
  │   ├── 01_requirement/  → 5 Stories · 24 BDD ACs
  │   ├── 02_design/  → ADR + API
  │   ├── 03_construction/  → 6 docs (031-036)
  │   └── 05_devops/  → CI/CD + Deploy + Runbook
  ├── 🐜 tiny_mchwa/  → Todo List · ✅ Spec Complete
  │   ├── 01_requirement/
  │   ├── 02_design/
  │   ├── 03_construction/
  │   ├── 04_testing/
  │   ├── 05_devops/
  │   ├── 06_security/
  │   └── 07_pm/
  ├── 🐑 fluffy_mouton/  → URL Shortener
  │   └── migration_note/
  ├── 🦉 cute_gufo/  → Blog · TBD
  └── 🐷 big_schwein/  → Ledger · TBD

  plan/
  └── phase2-foundation-hardening.md  → 5 Initiatives · 14 Action Items
```

## Development Status

| Icon | Meaning |
|------|---------|
| ❌ | Not started / not deployed |
| 📋 | Spec document ready |
| 🛠️ | In progress |
| ✅ | Completed / deployed |

## Quick Links

**Platform:**
- [[panomete_platform/011_business_objective | Platform Business Objectives]]
- [[panomete_platform/014_stakeholder_analysis | Stakeholder Analysis]]
- [[panomete_platform/03_construction/031_README_developer_guide | Platform Developer Guide]]

**Foundation Services (✅ Deployed):**
- [[flowero_guard/03_construction/031_README_developer_guide | Flowero Guard — Dev Guide]]
- [[flowero_discover/03_construction/031_README_developer_guide | Flowero Discover — Dev Guide]]
- [[flowero_gate/03_construction/031_README_developer_guide | Flowero Gate — Dev Guide]]

**Phase 2 Planning:**
- [[../../plan/phase2-foundation-hardening | Phase 2 — Foundation Hardening Plan]]

**Meeting Minutes:**
- [[../meeting-minute/MM07_phase2-planning_20260724 | Phase 2 Proposal (DevOps → PO)]]
- [[../meeting-minute/po-update-2026-07-22 | PO Requirements Update]]

---

> **Phase 1:** ✅ Foundation deployed (Guard, Discover, Gate) | **Phase 2:** 🛠️ Foundation Hardening (CI/CD, Observability, Alerting, Backup, Uptime) | **Phase 3:** 📋 Business Service Onboarding | **Architecture:** Cloudflare → Nginx → centralized auth, discovery, and API gateway | **Domain:** `*.panomete.com`
