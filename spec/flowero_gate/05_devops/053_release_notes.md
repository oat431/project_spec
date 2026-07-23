---
document_type: Release Notes
version: "0.1"
status: Placeholder
author: "DevOps Persona"
created: "2026-07-23"
last_updated: "2026-07-23"
project_name: "Flowero Gate"
project_id: "PAN-GATE-001"
parent_platform: "Panomete Platform"
classification: "Internal"
tags: [release-notes, changelog, spring-cloud-gateway, panomete]
standard_ref:
  - SWEBOK v4 — Operations
---

# Release Notes — Flowero Gate

> **Service:** Flowero Gate (Spring Cloud Gateway)
> **Platform:** Panomete Platform
> **Version:** 0.1 | **Status:** Placeholder — No releases yet. Will be populated when Sprint 2 ships.
> **Last Updated:** 2026-07-23

---

## 1. Purpose

> Documents what shipped in each Gate release — new routes, security changes, rate limiting updates, and fixes.

---

## 2. Release Template

### vX.Y.Z — [YYYY-MM-DD]

**Release Type:** [Major / Minor / Patch]
**Sprint:** [Sprint #]
**Image Tag:** `ghcr.io/panomete/flowero-gate:<tag>`

#### 🛣️ Route Changes

| # | Route | Backend | Change | Related Story |
|---|-------|---------|--------|:---:|
| 1 | `/api/{service}/**` | `lb://{service}` | [Added/Modified/Removed] | US-XXX |

#### 🚀 New Features

| # | Feature | Description | Related Story |
|---|---------|-------------|:---:|
| 1 | [feature] | [description] | US-XXX |

#### 🔒 Security Changes

| # | Change | Impact |
|---|--------|--------|

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

## 3. Expected Releases (Gate Roadmap)

### v0.1.0 — Sprint 2: Gateway Core + Auth

| Story | Description |
|-------|-------------|
| US-201 | Spring Cloud Gateway on :8000, Valkey rate limiting, JSON logging |
| US-202 | Route business APIs (blog, short, todo paths) |
| US-203 | JWT local validation via Keycloak JWKS, claim header forwarding |

**Expected deliverables:**
- Spring Cloud Gateway running on port 8000
- Valkey-backed rate limiting (100 req/min default)
- JWT validation using Keycloak JWKS (cached locally)
- Claim headers forwarded (`X-User-Id`, `X-User-Roles`)
- Structured JSON request logging
- Routes configured for blog, short, todo paths
- 401 for missing/invalid tokens
- 429 for rate limit exceeded

### v0.2.0 — Sprint 3: Eureka Integration + Hardening

| Story | Description |
|-------|-------------|
| US-204 | Route resolution via Eureka (`lb://` caching, failover) |
| US-205 | Valkey-backed rate limiting (per-route, per-IP) |

**Expected deliverables:**
- `lb://` route resolution via Eureka
- Cached registry for zero-per-request Eureka calls
- Per-route and per-IP rate limiting
- Failover to cached registry on Eureka failure

---

## 4. Release History

| Version | Date | Type | Sprint | Image Tag | Routes | Features |
|---------|------|------|--------|-----------|:---:|:---:|
| _(none yet)_ | — | — | — | — | 0 | 0 |

---

## Related Documents

| Document | Relationship |
|----------|-------------|
| [[052_deployment_plan]] | How Gate was deployed |
| [[051_CICD_pipeline_configuration]] | Gate's pipeline |
| `flowero_gate/02_design/022_API_specification` | Full route table and auth behavior |
| `panomete_platform/05_devops/053_release_notes` | Platform release notes |

---

> **Template Standard:** Based on SWEBOK v4
> **Usage:** Every route change is a release. Security changes must be highlighted.
