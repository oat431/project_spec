---
document_type: Release Notes
version: "0.1"
status: Placeholder
author: "DevOps Persona"
created: "2026-07-23"
last_updated: "2026-07-23"
project_name: "Flowero Discover"
project_id: "PAN-DISCOVER-001"
parent_platform: "Panomete Platform"
classification: "Internal"
tags: [release-notes, changelog, eureka, panomete]
standard_ref:
  - SWEBOK v4 — Operations
---

# Release Notes — Flowero Discover

> **Service:** Flowero Discover (Spring Cloud Netflix Eureka)
> **Platform:** Panomete Platform
> **Version:** 0.1 | **Status:** Placeholder — No releases yet. Will be populated when Sprint 1 ships.
> **Last Updated:** 2026-07-23

---

## 1. Purpose

> Documents what shipped in each Discover release — new features, configuration changes, and fixes.

---

## 2. Release Template

### vX.Y.Z — [YYYY-MM-DD]

**Release Type:** [Major / Minor / Patch]
**Sprint:** [Sprint #]
**Image Tag:** `ghcr.io/panomete/flowero-discover:<tag>`

#### 🚀 New Features

| # | Feature | Description | Related Story |
|---|---------|-------------|:---:|
| 1 | [feature] | [description] | US-XXX |

#### ✨ Improvements

| # | Improvement | Before | After |
|---|-----------|--------|-------|

#### 🐛 Bug Fixes

| # | Fix | Issue | Impact |
|---|-----|-------|--------|

#### ⚠️ Known Issues

| # | Issue | Workaround | Fix Planned |
|---|-------|-----------|------------|

#### 📦 Dependencies

| Package | Old Version | New Version | Reason |
|---------|-----------|------------|--------|

---

## 3. Expected Releases (Discover Roadmap)

### v0.1.0 — Sprint 1: Eureka Server

| Story | Description |
|-------|-------------|
| US-101 | Eureka server on :8999/:3999, standalone mode, dashboard via Nginx |

**Expected deliverables:**
- Eureka server running on ports 8999 (API) and 3999 (dashboard)
- Standalone mode (no peer replication)
- Dashboard accessible at `discovery.panomete.com`
- Self-preservation mode enabled
- Health endpoint at `/actuator/health`

### v0.2.0 — Sprint 2: Auto-Registration + Discovery

| Story | Description |
|-------|-------------|
| US-102 | Service auto-registration on startup |
| US-103 | Service discovery by name (`lb://` resolution) |

**Expected deliverables:**
- Guard and Gate auto-register with Eureka on startup
- Gate can resolve `lb://` URIs via Eureka
- Heartbeat mechanism working (30s renewal)

### v0.3.0 — Sprint 3: Dashboard Polish

| Story | Description |
|-------|-------------|
| US-104 | Dashboard accessible via Nginx |

**Expected deliverables:**
- Dashboard properly proxied through Nginx
- Relative URLs work correctly
- (If dual-port design changed) consolidated to single port

---

## 4. Release History

| Version | Date | Type | Sprint | Image Tag | Features |
|---------|------|------|--------|-----------|:---:|
| _(none yet)_ | — | — | — | — | 0 |

---

## Related Documents

| Document | Relationship |
|----------|-------------|
| [[052_deployment_plan]] | How Discover was deployed |
| [[051_CICD_pipeline_configuration]] | Discover's pipeline |
| `flowero_discover/02_design/022_API_specification` | Eureka REST API |
| `panomete_platform/05_devops/053_release_notes` | Platform release notes |

---

> **Template Standard:** Based on SWEBOK v4
> **Usage:** Release notes are for humans. Write clearly.
