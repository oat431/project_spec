---
document_type: Acceptance Criteria (ATDD/BDD)
version: "0.1"
status: Draft
author: "PO (Product Owner)"
created: "2026-07-22"
last_updated: "2026-07-22"
project_name: "Flowero Guard"
project_id: "PAN-GUARD-001"
ba_owner: "PO (Product Owner)"
qa_lead: "QA persona (TBD)"
classification: "Internal"
tags: [acceptance-criteria, bdd, given-when-then, keycloak, oauth2, panomete]
standard_ref:
  - SWEBOK v4 — Requirements
  - ISO/IEC/IEEE 29119 — Software Testing
  - ISO/IEC/IEEE 29148 — Requirements Engineering
---

# Acceptance Criteria — Flowero Guard

> **Service:** Flowero Guard (Keycloak IAM)
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
| 0.1 | 2026-07-22 | PO | Initial draft — BDD criteria for all 6 stories |

---

## 1. Purpose

> This document defines acceptance criteria for every Flowero Guard user story using the **Given/When/Then** (GWT) format. Every criterion is testable, unambiguous, and approved by PO before development begins.

## 2. Acceptance Criteria

### US-001: Deploy Keycloak Instance

| AC ID | Scenario | Given | When | Then | Priority |
|-------|---------|-------|------|------|----------|
| AC-G001a | Healthy startup | Docker Compose file includes Keycloak + PostgreSQL services | `docker compose up` is executed | Keycloak is reachable at `https://panomete.local/auth` within 60 seconds; health check returns 200 | 🔴 |
| AC-G001b | Realm pre-configuration | Keycloak container starts with a mounted realm export JSON | Admin logs into the master realm | A `panomete` realm exists with pre-configured roles: `admin`, `user`, `viewer` | 🔴 |
| AC-G001c | Realm import from file | The realm JSON file exists at `./keycloak/panomete-realm.json` | Keycloak starts with `--import-realm` flag | The `panomete` realm is imported; no manual UI configuration needed | 🔴 |
| AC-G001d | Persistent state across restarts | Keycloak + PostgreSQL have been running; clients and users exist | `docker compose down && docker compose up` | All previously created clients, users, and roles are intact; no data loss | 🔴 |
| AC-G001e | Failed startup — missing DB | PostgreSQL container is not running | Keycloak starts | Keycloak logs a clear error about database connectivity and retries; does not silently fail | 🟡 |
| AC-G001f | Admin console access | Keycloak is running; admin credentials are set via env vars | Admin navigates to `/auth/admin/master/console/` | Admin console loads; admin can log in with credentials from environment variables | 🔴 |

### US-002: Register OAuth2 Clients

| AC ID | Scenario | Given | When | Then | Priority |
|-------|---------|-------|------|------|----------|
| AC-G002a | Register new client | Admin is logged into Keycloak admin console; a new service needs a client | Admin creates a client with ID `fluffy-mouton`, confidential access type, service accounts enabled | A client ID and secret are generated; the client can request tokens using client credentials | 🔴 |
| AC-G002b | Client requests token | Client `fluffy-mouton` is registered with a secret | Client sends `POST /realms/panomete/protocol/openid-connect/token` with `grant_type=client_credentials` | A valid JWT is returned containing the client's service account roles | 🔴 |
| AC-G002c | Invalid client secret | Client `fluffy-mouton` is registered | Client sends token request with an incorrect secret | Keycloak returns 401 with `{"error": "invalid_client"}` | 🔴 |
| AC-G002d | Standard client template | Multiple services need similar OAuth2 config | Admin manually creates a second client with the same settings as the first | The process takes <5 minutes; admin can document a step-by-step template | 🟡 |

### US-003: Login and Single Sign-On

| AC ID | Scenario | Given | When | Then | Priority |
|-------|---------|-------|------|------|----------|
| AC-G003a | Redirect to login | User accesses `https://panomete.local/api/blog/posts` without authentication | The request hits Gate, which redirects to Keycloak | User sees the Keycloak login page at `https://panomete.local/auth/realms/panomete/protocol/openid-connect/auth` | 🔴 |
| AC-G003b | Successful login | User enters valid username and password on the Keycloak login page | User clicks Sign In | Keycloak redirects back with an authorization code; the client exchanges it for a JWT access token (5 min expiry) and refresh token (30 min expiry) | 🔴 |
| AC-G003c | Failed login — bad password | User enters valid username but wrong password | User clicks Sign In | Keycloak displays "Invalid username or password" on the login page; no token issued | 🔴 |
| AC-G003d | Single Sign-On | User logged into Service A (blog) and has a valid Keycloak session cookie | User navigates to Service B (URL shortener) | User is NOT prompted to log in again; Service B receives a valid token via SSO | 🔴 |
| AC-G003e | Token refresh | User's access token is expired (5 min elapsed) | Client sends the refresh token to Keycloak's token endpoint | Keycloak returns a new access token and refresh token; no re-login required | 🔴 |
| AC-G003f | Logout | User has active sessions across multiple services | User clicks Logout on any service | Keycloak terminates all sessions; subsequent requests to any service require re-authentication | 🔴 |
| AC-G003g | Session timeout | User has a Keycloak session but has been idle for the configured SSO session max (e.g., 30 min) | User accesses a protected service | User is redirected to the Keycloak login page (session expired) | 🟡 |

### US-004: Role-Based Access Control

| AC ID | Scenario | Given | When | Then | Priority |
|-------|---------|-------|------|------|----------|
| AC-G004a | Admin role access | User `alice` has realm role `admin`; endpoint requires `ROLE_admin` | Alice accesses an admin-only endpoint | Service returns 200; endpoint logic executes | 🔴 |
| AC-G004b | User role denied | User `bob` has realm role `user`; endpoint requires `ROLE_admin` | Bob accesses an admin-only endpoint | Service returns 403 Forbidden with `{"error": "Access Denied", "required_role": "admin"}` | 🔴 |
| AC-G004c | Roles in JWT claims | User `alice` with roles `admin` and `user` logs in | The JWT is decoded | The `realm_access.roles` claim contains `["admin", "user"]` | 🔴 |
| AC-G004d | Declarative enforcement | A Spring Boot service has `@PreAuthorize("hasRole('admin')")` on an endpoint | Any request to that endpoint is made | Spring Security automatically checks the JWT roles claim; no manual role-checking code | 🔴 |
| AC-G004e | No roles — default access | User has no realm roles assigned | User accesses a public endpoint (no `@PreAuthorize`) | Service returns 200 (no role required) | 🟡 |

### US-005: Service-to-Service Authentication

| AC ID | Scenario | Given | When | Then | Priority |
|-------|---------|-------|------|------|----------|
| AC-G005a | Client credentials grant | Service A has client ID `service-a` and a secret registered in Keycloak | Service A requests `POST /token` with `grant_type=client_credentials` | Keycloak returns a JWT with `sub: service-a`, `aud: account`, and service account roles | 🟡 |
| AC-G005b | Service calls service | Service A has a valid client credentials JWT | Service A calls Service B's API with `Authorization: Bearer <jwt>` | Service B validates the token; response 200 | 🟡 |
| AC-G005c | No token — service call | Service A attempts to call Service B | Service A sends the request without an Authorization header | Service B returns 401 | 🟡 |
| AC-G005d | Insufficient service roles | Service A has role `reader` but Service B endpoint requires role `writer` | Service A calls Service B with its JWT | Service B returns 403 | 🟡 |

### US-006: Token Introspection

| AC ID | Scenario | Given | When | Then | Priority |
|-------|---------|-------|------|------|----------|
| AC-G006a | Introspect valid opaque token | A non-Java service has an opaque access token | Service calls `POST /realms/panomete/protocol/openid-connect/token/introspect` with its client credentials | Response: `{"active": true, "sub": "...", "realm_access": {"roles": [...]}, ...}` | 🟡 |
| AC-G006b | Introspect expired token | The opaque token has expired (`exp` claim is in the past) | Service calls the introspection endpoint | Response: `{"active": false}` | 🟡 |
| AC-G006c | Introspect with no auth | Any caller calls the introspection endpoint | Caller sends the request without client credentials | Response: 401 Unauthorized | 🟡 |
| AC-G006d | Introspect JWT token | A service has a JWT (not opaque) and wants to use introspection instead of local validation | Service calls the introspection endpoint with the JWT as the token | Response: `{"active": true, ...claims}` — works identically for JWT and opaque tokens | 🟡 |

---

## 3. Acceptance Criteria Summary

| Story | AC Count | 🔴 Must Have | 🟡 Should Have |
|-------|---------|-------------|---------------|
| US-001: Deploy Keycloak | 6 | 4 | 2 |
| US-002: Register Clients | 4 | 3 | 1 |
| US-003: Login & SSO | 7 | 6 | 1 |
| US-004: RBAC | 5 | 4 | 1 |
| US-005: S2S Auth | 4 | 0 | 4 |
| US-006: Introspection | 4 | 0 | 4 |
| **Total** | **30** | **17** | **13** |

---

## Related Documents

| Document | Relationship |
|----------|-------------|
| [[flowero_guard/012_user_stories]] | Stories these criteria verify |
| [[flowero_guard/011_business_objective]] | Objectives these criteria support |
| [[panomete_platform/011_business_objective]] | Platform-level objectives |

---

> **Template Standard:** Based on SWEBOK v4, ISO/IEC/IEEE 29119, ISO/IEC/IEEE 29148
> **Usage:** Every story MUST have ≥1 AC. Review in "3 amigos" session (PO + Dev + QA) before sprint start. Mark AC as ✅ when test passes.
