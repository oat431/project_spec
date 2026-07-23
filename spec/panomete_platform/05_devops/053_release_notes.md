---
document_type: Release Notes
version: "0.1"
status: Placeholder
author: "DevOps Persona"
created: "2026-07-23"
last_updated: "2026-07-23"
project_name: "Panomete Platform"
project_id: "PAN-PLAT-001"
classification: "Internal"
tags: [release-notes, changelog, panomete]
standard_ref:
  - SWEBOK v4 — Operations
---

# Release Notes — Panomete Platform

> **Project:** Panomete Platform
> **Version:** 0.1 | **Status:** Placeholder — No releases yet. Will be populated when Sprint 1 ships.
> **Last Updated:** 2026-07-23

---

## 1. Purpose

> Documents what shipped in each release — features, fixes, breaking changes, and migration notes. Auto-generated from conventional commits when CI/CD is active.

---

## 2. Release Template

> Copy this template for each new release. Populate from conventional commit messages.

### vX.Y.Z — [YYYY-MM-DD]

**Release Name:** [name]
**Release Type:** [Major / Minor / Patch]
**Sprint:** [Sprint #]

#### 🚀 New Features

| # | Feature | Description | Service | Related Story |
|---|---------|-------------|---------|:---:|
| 1 | [feature] | [description] | [service] | US-XXX |

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

#### 🔄 Migration Notes

| # | Action Required | Who | When |
|---|----------------|-----|------|

---

## 3. Expected Releases (Phase 1 Roadmap)

> These are the planned releases. Actual content depends on sprint outcomes.

### v0.1.0 — Sprint 1: Core Infrastructure (Expected: 2026-08-XX)

| Story | Service | Description |
|-------|---------|-------------|
| US-001 | Guard | Keycloak container with Panomete realm, shared PostgreSQL, realm-as-code JSON |
| US-101 | Discover | Eureka server on :8999/:3999, standalone mode, dashboard via Nginx |

### v0.2.0 — Sprint 2: End-to-End Auth (Expected: 2026-09-XX)

| Story | Service | Description |
|-------|---------|-------------|
| US-002 | Guard | Register OAuth2 clients for platform services |
| US-201 | Gate | Spring Cloud Gateway on :8000, Valkey rate limiting, JSON logging |
| US-202 | Gate | Route business APIs (blog, short, todo paths) |
| US-203 | Gate | JWT local validation via Keycloak JWKS, claim header forwarding |
| US-102 | Discover | Service auto-registration on startup |
| US-103 | Discover | Service discovery by name (`lb://` resolution) |
| US-003 | Guard | Login flow + SSO across services |

### v0.3.0 — Sprint 3: Production Hardening (Expected: 2026-10-XX)

| Story | Service | Description |
|-------|---------|-------------|
| US-204 | Gate | Route resolution via Eureka (`lb://` caching, failover) |
| US-205 | Gate | Valkey-backed rate limiting (per-route, per-IP) |
| US-104 | Discover | Dashboard accessible via Nginx |
| US-004 | Guard | RBAC (realm roles → `@PreAuthorize`) |

---

## 4. Release History

| Version | Date | Type | Sprint | Features | Fixes |
|---------|------|------|--------|:---:|:---:|
| _(none yet)_ | — | — | — | 0 | 0 |

---

## Related Documents

| Document | Relationship |
|----------|-------------|
| [[052_deployment_plan]] | Deployment procedures |
| [[051_CICD_pipeline_configuration]] | Pipeline that produces releases |
| `03_construction/034_commit_messages_changelog` | Commit-level changes (when created) |
| [[MM03_next-steps_20260722]] | Sprint backlog and story inventory |

---

> **Template Standard:** Based on SWEBOK v4
> **Usage:** Release notes are for *everyone* — developers, PMs, users. Write them for humans, not machines.
