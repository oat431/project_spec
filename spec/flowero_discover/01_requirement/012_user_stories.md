---
document_type: User Stories
version: "0.2"
status: Draft
author: "PO (Product Owner)"
created: "2026-07-22"
last_updated: "2026-07-22"
project_name: "Flowero Discover"
project_id: "PAN-DISCOVER-001"
parent_platform: "Panomete Platform"
ba_owner: "PO (Product Owner)"
po_owner: "PO (Product Owner)"
classification: "Internal"
tags: [user-stories, eureka, service-discovery, microservices, panomete]
standard_ref:
  - SWEBOK v4 — Requirements
  - ISO/IEC/IEEE 29148 — Requirements Engineering
---

# User Stories — Flowero Discover

> **Service:** Flowero Discover (Spring Cloud Netflix Eureka)
> **Platform:** Panomete Platform
> **Version:** 0.1 | **Status:** Draft
> **Last Updated:** 2026-07-22

---

## Document Control

| Field | Value |
|-------|-------|
| Document Owner | PO (Product Owner) |
| Business Analyst | PO (Product Owner) |
| Product Owner | PO (Product Owner) |

### Revision History

| Version | Date | Author | Change Description |
|---------|------|--------|--------------------|
| 0.1 | 2026-07-22 | PO | Initial draft — extracted from platform-level stories |
| 0.2 | 2026-07-22 | PO | Design review update — ports 8999/3999, discovery.panomete.com via Nginx, US-104 rewritten (dashboard via Nginx, not Gate) |

---

## 1. Purpose

> Flowero Discover is the service registry for the Panomete Platform — a Spring Cloud Netflix Eureka server. Every microservice registers itself on startup. Services discover peers dynamically by name instead of hardcoded URLs. Flowero Gate resolves routes through Discover.

## 2. Service Context

| Aspect | Detail |
|--------|--------|
| **Service Type** | Foundation — Service Registry |
| **Technology** | Spring Cloud Netflix Eureka Server |
| **Language / Stack** | Java 25 / Spring Boot 4.1.x |
| **Ports** | 8999 (BE — registration API), 3999 (FE — dashboard) |
| **Domain** | `discovery.panomete.com` (via Nginx) |
| **Dependencies** | None (standalone, no database) |
| **Consumed By** | All platform services (register + discover), Gate (route resolution for business services) |
| **Related Services** | Nginx (routes `discovery.panomete.com` → :3999/:8999) |

## 3. Personas

| Persona | Role |
|---------|------|
| **Platform Admin** | Deploys Eureka, monitors service health via dashboard |
| **Service Developer** | Adds Eureka client dependency — service auto-registers, discovers peers by name |

---

## 4. User Stories

### US-101: Deploy Eureka Server

**As a** Platform Admin
**I want** to deploy a Spring Cloud Eureka server via Docker Compose
**So that** all microservices can register and discover each other dynamically from day one

**Acceptance Criteria:**
- **AC-1:** Given `docker compose up`, When the Eureka container starts, Then the Eureka dashboard is accessible at `discovery.panomete.com` (via Nginx) within 30 seconds
- **AC-2:** Given Eureka is running, When I check the dashboard, Then it shows "No instances available" with zero registered services (clean initial state)
- **AC-3:** Given the Eureka server, When it restarts, Then it begins accepting registrations within 15 seconds
- **AC-4:** Given Eureka is deployed, When I inspect the configuration, Then it runs in standalone mode (single node) with `eureka.client.register-with-eureka: false` and `fetch-registry: false`

**Story Points:** 3
**Priority:** 🔴 Must Have
**Sprint:** M1 — Core Infrastructure
**Status:** Draft
**Mapped Objective:** OBJ-02 (Service Discovery)

---

### US-102: Service Auto-Registration on Startup

**As a** Service Developer
**I want** my Spring Boot microservice to automatically register with Eureka when it starts
**So that** other services and the Gateway can discover it without any manual configuration

**Acceptance Criteria:**
- **AC-1:** Given a Spring Boot service with `spring-cloud-starter-netflix-eureka-client` on the classpath, When the service starts, Then it appears in the Eureka dashboard within 30 seconds with status UP
- **AC-2:** Given the service registration, When I query Eureka's REST API (`GET /eureka/apps/{service-name}`), Then the response includes: service name, instance ID, host, port, health check URL, and metadata map
- **AC-3:** Given the service shuts down gracefully (SIGTERM), When it sends a deregistration request to Eureka, Then Eureka removes the instance within 30 seconds
- **AC-4:** Given the service crashes or loses network (no graceful shutdown), When heartbeats fail for 90 seconds, Then Eureka evicts the stale registration automatically (self-preservation mode handles transient network issues)

**Story Points:** 3
**Priority:** 🔴 Must Have
**Sprint:** M2 — End-to-End Auth
**Status:** Draft
**Mapped Objective:** OBJ-02

---

### US-103: Service Discovery by Name

**As a** Service Developer
**I want** my service to look up peer services by their registered Eureka name using a load-balanced client
**So that** inter-service communication is dynamic and resilient — no hardcoded URLs anywhere

**Acceptance Criteria:**
- **AC-1:** Given Service A needs to call Service B, When Service A resolves `http://b-service` via a load-balanced `RestTemplate` or `WebClient`, Then the actual host:port is resolved from Eureka and the request succeeds
- **AC-2:** Given 3 instances of Service B are registered, When Service A calls Service B 30 times, Then requests are distributed roughly evenly across all instances (client-side round-robin load balancing)
- **AC-3:** Given Service B is NOT registered, When Service A tries to resolve it, Then the call fails with a clear error — not a cryptic `UnknownHostException`
- **AC-4:** Given a service name, When I call Eureka's REST API, Then I get a list of all healthy instances (XML or JSON) within 100ms

**Story Points:** 3
**Priority:** 🔴 Must Have
**Sprint:** M2 — End-to-End Auth
**Status:** Draft
**Mapped Objective:** OBJ-02

---

### US-104: Eureka Dashboard Accessible via Nginx

**As a** Platform Admin
**I want** to view the Eureka dashboard at `discovery.panomete.com` through Nginx
**So that** I can monitor service health from a dedicated subdomain without exposing Eureka's port directly

**Acceptance Criteria:**
- **AC-1:** Given I navigate to `discovery.panomete.com`, When Nginx proxies the request to Eureka on port 3999, Then the Eureka dashboard renders correctly with all CSS, JavaScript, and images
- **AC-2:** Given the dashboard is loaded, When I view the "Instances currently registered with Eureka" section, Then I see every registered service with: application name, status (UP/DOWN), and instance count
- **AC-3:** Given I click on a service link in the dashboard, When the link resolves, Then I am taken to that service's info page (host, port, health URL, lease info)

**Story Points:** 2
**Priority:** 🟡 Should Have
**Sprint:** M3 — Production Hardening
**Status:** Draft
**Mapped Objective:** OBJ-02

---

## 5. Story Estimation Summary

| Story | Name | Points | Priority | Sprint |
|-------|------|--------|----------|--------|
| US-101 | Deploy Eureka Server | 3 | 🔴 | M1 |
| US-102 | Service Auto-Registration | 3 | 🔴 | M2 |
| US-103 | Service Discovery by Name | 3 | 🔴 | M2 |
| US-104 | Dashboard via Nginx | 2 | 🟡 | M3 |
| **Total** | | **11** | | |

---

## Related Documents

| Document | Relationship |
|----------|-------------|
| [[panomete_platform/011_business_objective]] | Platform-level objectives (OBJ-02) |
| [[flowero_discover/011_business_objective]] | Service-level objectives |
| [[flowero_discover/013_acceptance_criteria]] | BDD acceptance criteria for all stories |
| [[flowero_gate/012_user_stories]] | Gate resolves routes through Discover |
| [[flowero_guard/012_user_stories]] | Guard secures the Discover dashboard |

---

> **Template Standard:** Based on SWEBOK v4, ISO/IEC/IEEE 29148
> **Usage:** These stories define what Flowero Discover must deliver. Stories are refined with the Dev persona before sprint start.
