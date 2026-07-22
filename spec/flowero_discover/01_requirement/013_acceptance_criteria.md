---
document_type: Acceptance Criteria (ATDD/BDD)
version: "0.1"
status: Draft
author: "PO (Product Owner)"
created: "2026-07-22"
last_updated: "2026-07-22"
project_name: "Flowero Discover"
project_id: "PAN-DISCOVER-001"
ba_owner: "PO (Product Owner)"
qa_lead: "QA persona (TBD)"
classification: "Internal"
tags: [acceptance-criteria, bdd, given-when-then, eureka, service-discovery, panomete]
standard_ref:
  - SWEBOK v4 — Requirements
  - ISO/IEC/IEEE 29119 — Software Testing
  - ISO/IEC/IEEE 29148 — Requirements Engineering
---

# Acceptance Criteria — Flowero Discover

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
| QA Lead | QA persona (TBD) |

### Revision History

| Version | Date | Author | Change Description |
|---------|------|--------|--------------------|
| 0.1 | 2026-07-22 | PO | Initial draft — BDD criteria for all 4 stories |

---

## 1. Purpose

> This document defines acceptance criteria for every Flowero Discover user story using the **Given/When/Then** (GWT) format.

## 2. Acceptance Criteria

### US-101: Deploy Eureka Server

| AC ID | Scenario | Given | When | Then | Priority |
|-------|---------|-------|------|------|----------|
| AC-D101a | Healthy startup | Docker Compose includes Eureka service | `docker compose up` is executed | Eureka dashboard is reachable at `discovery.panomete.com` within 30 seconds | 🔴 |
| AC-D101b | Clean initial state | Eureka starts for the first time | Admin opens the dashboard at `discovery.panomete.com` | Dashboard shows "No instances available" in the "Instances currently registered with Eureka" section | 🔴 |
| AC-D101c | Fast restart | Eureka has been running; services are registered | `docker compose restart eureka` | Eureka accepts new registrations within 15 seconds; previously registered services re-register | 🔴 |
| AC-D101d | Standalone mode | Eureka container is the only instance | Admin checks Eureka's `/eureka/apps` endpoint | `eureka.client.register-with-eureka: false` and `fetch-registry: false` — Eureka does not try to register with itself | 🔴 |
| AC-D101e | Dashboard CSS/JS loads | Nginx proxies `discovery.panomete.com` to Eureka on port 3999 | Admin opens `discovery.panomete.com` | Dashboard renders completely — all CSS, JavaScript, and images load correctly (no broken resources) | 🟡 |

### US-102: Service Auto-Registration

| AC ID | Scenario | Given | When | Then | Priority |
|-------|---------|-------|------|------|----------|
| AC-D102a | Spring Boot service registers | A Spring Boot service has `spring-cloud-starter-netflix-eureka-client` on classpath and `eureka.client.service-url.defaultZone` configured | Service starts | Service appears in Eureka dashboard with status UP within 30 seconds | 🔴 |
| AC-D102b | Registration metadata | Service `cute-gufo` registers with Eureka | Admin queries `GET /eureka/apps/CUTE-GUFO` | Response includes: `hostName`, `port`, `healthCheckUrl`, `statusPageUrl`, and `metadata` map | 🔴 |
| AC-D102c | Graceful deregistration | Service is registered and receives SIGTERM | Service shuts down, sending `DELETE /eureka/apps/{app}/{instance}` | Eureka removes the instance within 30 seconds; dashboard reflects the change | 🔴 |
| AC-D102d | Crash eviction | Service crashes (no deregistration signal) | Heartbeats fail for 90 seconds | Eureka evicts the stale registration; service no longer appears in the registry | 🔴 |
| AC-D102e | Non-Spring Boot service registers | A Go service manually sends a registration request to Eureka's REST API | Service sends `POST /eureka/apps/{app}` with JSON/XML body | Service appears in Eureka dashboard within 30 seconds; heartbeats maintain registration | 🟡 |

### US-103: Service Discovery by Name

| AC ID | Scenario | Given | When | Then | Priority |
|-------|---------|-------|------|------|----------|
| AC-D103a | Resolve by name | Service A has a `@LoadBalanced RestTemplate` bean | Service A calls `restTemplate.getForObject("http://cute-gufo/api/posts", ...)` | The host and port are resolved from Eureka; the call reaches a Cute Gufo instance | 🔴 |
| AC-D103b | Client-side load balancing | 3 instances of Cute Gufo are registered | Service A makes 30 requests to `http://cute-gufo/api/posts` | Requests are distributed roughly evenly: each instance receives 8–12 requests | 🔴 |
| AC-D103c | Service not found | Service A tries to resolve `http://nonexistent-service` | There is no service registered with that name | The call fails with a clear `IllegalStateException` or `ServiceUnavailableException` — not an `UnknownHostException` | 🔴 |
| AC-D103d | Discovery API latency | Eureka has 10 services registered | A client queries `GET /eureka/apps` | Response is returned within 100ms at p95 | 🟡 |

### US-104: Dashboard via Nginx

| AC ID | Scenario | Given | When | Then | Priority |
|-------|---------|-------|------|------|----------|
| AC-D104a | Dashboard access | Nginx is configured to proxy `discovery.panomete.com` → Eureka :3999 | Admin navigates to `discovery.panomete.com` | Dashboard loads; all registered services visible with UP/DOWN status | 🟡 |
| AC-D104b | Service detail drill-down | Dashboard is loaded; services are listed | Admin clicks on a service link | Service detail page loads showing: instance ID, host, port, health check URL, lease expiration | 🟡 |

---

## 3. Acceptance Criteria Summary

| Story | AC Count | 🔴 Must Have | 🟡 Should Have |
|-------|---------|-------------|---------------|
| US-101: Deploy Eureka | 5 | 4 | 1 |
| US-102: Auto-Registration | 5 | 4 | 1 |
| US-103: Discovery by Name | 4 | 3 | 1 |
| US-104: Dashboard via Nginx | 2 | 0 | 2 |
| **Total** | **16** | **11** | **5** |

---

## Related Documents

| Document | Relationship |
|----------|-------------|
| [[flowero_discover/012_user_stories]] | Stories these criteria verify |
| [[flowero_discover/011_business_objective]] | Objectives these criteria support |
| [[flowero_gate/012_user_stories]] | Gate resolves routes through Discover |

---

> **Template Standard:** Based on SWEBOK v4, ISO/IEC/IEEE 29119, ISO/IEC/IEEE 29148
