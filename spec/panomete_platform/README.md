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

    subgraph Foundation["🔒 Foundation Services"]
        Guard["Flowero Guard<br>Keycloak IAM :8001<br>auth.panomete.com"]
        Discover["Flowero Discover<br>Eureka Registry :8999/:3999<br>discovery.panomete.com"]
    end

    subgraph Gateway["Flowero Gate — API Gateway"]
        Gate["Spring Cloud Gateway :8000<br>api.panomete.com<br>JWT Auth · Rate Limit · Routing"]
    end

    subgraph Business["📦 Business Services"]
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

    style External fill:#f6821f,stroke:#e06a10,color:#fff
    style Edge fill:#009639,stroke:#007a2e,color:#fff
    style Foundation fill:#2c3e50,stroke:#34495e,color:#ecf0f1
    style Gateway fill:#e74c3c,stroke:#c0392b,color:#fff
    style Business fill:#1a1a2e,stroke:#333,color:#aaa
    style Guard fill:#3498db,stroke:#2980b9,color:#fff
    style Discover fill:#27ae60,stroke:#1e8449,color:#fff
    style Gate fill:#e74c3c,stroke:#c0392b,color:#fff
    style CF fill:#f6821f,stroke:#e06a10,color:#fff
    style Nginx fill:#009639,stroke:#007a2e,color:#fff
```

## Foundation Services (Phase 1 MVP)

| Service | Code Name | Technology | Port | Domain | Role | Status |
|---------|-----------|-----------|------|--------|------|--------|
| **Flowero Guard** | — | Keycloak (uses shared PostgreSQL 18) | 8001 | `auth.panomete.com` | Identity & Access Management — issues OAuth2/OIDC tokens | 📋 Spec Ready |
| **Flowero Discover** | — | Spring Cloud Eureka | 8999 (BE) / 3999 (FE) | `discovery.panomete.com` | Service Registry & Discovery | 📋 Spec Ready |
| **Flowero Gate** | — | Spring Cloud Gateway | 8000 | `api.panomete.com` | API Gateway — JWT validation, Valkey-backed rate limiting, route to business services | 📋 Spec Ready |

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
| **Foundation Language** | Java 21 / Spring Boot 3.x | Spring ecosystem provides native Gateway (Spring Cloud Gateway), Security (OAuth2 Resource Server), and Discovery (Eureka) integrations |
| **Identity** | Keycloak | Production-grade OSS IAM with full OAuth2/OIDC support. Uses shared PostgreSQL 18 for persistence. |
| **API Gateway** | Spring Cloud Gateway | Reactive, non-blocking. Routes business APIs only (`api.panomete.com`). Validates JWT against Keycloak JWKS. |
| **Service Discovery** | Spring Cloud Netflix Eureka | Simplest path with Spring Boot; embeddable, no external dependency |
| **Rate Limiting** | Valkey 9 (shared instance) | Replaces in-memory rate limiting. Limits persist across Gate restarts. Already running in production. |
| **Deployment** | Docker Compose → Kubernetes (k3s) | Compose for homelab simplicity; K8s for portfolio growth |
| **Observability** | Phase 2: Actuator + Loki + Prometheus + Grafana | Deferred until foundation is stable. Uptime Kuma handles infra-level uptime monitoring in the interim. |

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
  │   └── 01_requirement/
  │       ├── 011_business_objective.md  → 5 SMART Objectives
  │       ├── 012_user_stories.md  → Umbrella Story Map
  │       └── 014_stakeholder_analysis.md  → 8 Stakeholders
  ├── 🔒 flowero_guard/  → Keycloak IAM · :8001 · auth.panomete.com
  │   └── 01_requirement/
  │       ├── 011_business_objective.md
  │       ├── 012_user_stories.md  → 6 Stories · 21 pts
  │       └── 013_acceptance_criteria.md  → 30 BDD Criteria
  ├── 🟢 flowero_discover/  → Eureka Registry · :8999/:3999 · discovery.panomete.com
  │   └── 01_requirement/
  │       ├── 011_business_objective.md
  │       ├── 012_user_stories.md  → 4 Stories · 11 pts
  │       └── 013_acceptance_criteria.md  → 17 BDD Criteria
  ├── 🔴 flowero_gate/  → API Gateway · :8000 · api.panomete.com
  │   └── 01_requirement/
  │       ├── 011_business_objective.md
  │       ├── 012_user_stories.md  → 5 Stories · 18 pts
  │       └── 013_acceptance_criteria.md  → 22 BDD Criteria
  ├── 🐜 tiny_mchwa/  → Todo List · ✅ Complete
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
```

## Development Status

| Icon | Meaning |
|------|---------|
| ❌ | Not started / not deployed |
| 📋 | Spec document ready |
| 🛠️ | In progress |
| ✅ | Completed / deployed |

## Quick Links

- [[panomete_platform/011_business_objective | Platform Business Objectives]]
- [[panomete_platform/014_stakeholder_analysis | Stakeholder Analysis]]
- [[flowero_guard/012_user_stories | Flowero Guard Stories]]
- [[flowero_discover/012_user_stories | Flowero Discover Stories]]
- [[flowero_gate/012_user_stories | Flowero Gate Stories]]
- [[meeting-minute/affect-doc | Design Review — Meeting Minutes (2026-07-22)]]

---

> **Built for:** Portfolio demonstration | **Architecture:** Microservices with Cloudflare → Nginx → centralized auth, discovery, and API gateway | **Phase 1:** Foundation services only | **Approach:** Documentation-first, AI-persona-driven implementation | **Domain:** `*.panomete.com`
