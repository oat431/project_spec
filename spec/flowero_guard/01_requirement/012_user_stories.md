---
document_type: User Stories
version: "0.1"
status: Draft
author: "PO (Product Owner)"
created: "2026-07-22"
last_updated: "2026-07-22"
project_name: "Flowero Guard"
project_id: "PAN-GUARD-001"
parent_platform: "Panomete Platform"
ba_owner: "PO (Product Owner)"
po_owner: "PO (Product Owner)"
classification: "Internal"
tags: [user-stories, keycloak, oauth2, iam, panomete]
standard_ref:
  - SWEBOK v4 — Requirements
  - ISO/IEC/IEEE 29148 — Requirements Engineering
---

# User Stories — Flowero Guard

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
| Product Owner | PO (Product Owner) |

### Revision History

| Version | Date | Author | Change Description |
|---------|------|--------|--------------------|
| 0.1 | 2026-07-22 | PO | Initial draft — extracted from platform-level stories |

---

## 1. Purpose

> Flowero Guard is the identity and access management layer of the Panomete Platform — a Keycloak instance that provides centralized authentication (OAuth2/OIDC), single sign-on (SSO), role-based access control (RBAC), and service-to-service authentication for all platform microservices.

## 2. Service Context

| Aspect | Detail |
|--------|--------|
| **Service Type** | Foundation — Identity Provider |
| **Technology** | Keycloak (Docker) with PostgreSQL backend |
| **Language / Stack** | Java (Keycloak is a Java application) |
| **Ports** | 8080 (internal), exposed through Flowero Gate |
| **Dependencies** | PostgreSQL database |
| **Consumed By** | All platform services (via OAuth2/OIDC) |
| **Related Services** | Flowero Gate (routes auth traffic), Flowero Discover (registers Keycloak health) |

## 3. Personas

| Persona | Role |
|---------|------|
| **Platform Admin** | Deploys and configures the Keycloak instance, manages realms, clients, and users |
| **Service Developer** | Registers their microservice as an OAuth2 client; relies on Guard for auth without writing auth code |
| **End User** | Logs in once and accesses all platform services seamlessly (SSO) |

---

## 4. User Stories

### US-001: Deploy Keycloak Instance

**As a** Platform Admin
**I want** to deploy a Keycloak instance via Docker Compose with a pre-configured Panomete realm
**So that** all microservices have a single, production-ready identity provider from day one

**Acceptance Criteria:**
- **AC-1:** Given `docker compose up`, When all containers start, Then Keycloak is accessible at `https://panomete.local/auth` (via Gate) within 60 seconds
- **AC-2:** Given Keycloak is running, When I access the admin console, Then a `panomete` realm exists with pre-configured roles (`admin`, `user`, `viewer`) and client scopes
- **AC-3:** Given the Keycloak container, When I inspect configuration, Then realm settings are imported from a version-controlled JSON export file
- **AC-4:** Given Keycloak restarts, When it comes back up, Then all clients, users, roles, and sessions are preserved (persistent PostgreSQL backend)

**Story Points:** 5
**Priority:** 🔴 Must Have
**Sprint:** M1 — Core Infrastructure
**Status:** Draft
**Mapped Objective:** OBJ-01 (Centralized Authentication)

---

### US-002: Register OAuth2 Clients for Platform Services

**As a** Service Developer
**I want** to register my microservice as an OAuth2 client in Flowero Guard with a standard configuration template
**So that** my service can authenticate users and validate tokens without writing auth logic from scratch

**Acceptance Criteria:**
- **AC-1:** Given a new service (e.g., Fluffy Mouton), When the admin registers it as a client in Keycloak, Then a client ID and secret are generated and the service can request tokens
- **AC-2:** Given a registered client, When the service initiates the authorization code flow, Then Keycloak returns a valid JWT with the service's configured scopes
- **AC-3:** Given the client registration, When viewed in Keycloak admin, Then it has standard settings: confidential access type, service accounts enabled, standard flow enabled, valid redirect URIs
- **AC-4:** Given the admin wants to register multiple services with similar config, When they use the client registration template, Then setup takes <5 minutes per service

**Story Points:** 3
**Priority:** 🔴 Must Have
**Sprint:** M1 — Core Infrastructure
**Status:** Draft
**Mapped Objective:** OBJ-01

---

### US-003: Login and Single Sign-On

**As an** End User
**I want** to log in once via the Keycloak login page and access any platform service without re-authenticating
**So that** I have a seamless experience across all Panomete services

**Acceptance Criteria:**
- **AC-1:** Given an unauthenticated user, When they access any protected service through Gate, Then they are redirected to the Keycloak login page
- **AC-2:** Given valid credentials, When the user submits the login form, Then Keycloak returns a JWT access token (5 min expiry) and a refresh token (30 min expiry)
- **AC-3:** Given an expired access token, When the service receives it, Then the service returns 401 and the client uses the refresh token to obtain a new access token silently
- **AC-4:** Given the user logged into Service A, When they navigate to Service B, Then they are NOT prompted to log in again (SSO via Keycloak session cookie)
- **AC-5:** Given the user clicks "Logout" on any service, When the logout completes, Then the Keycloak session is terminated and all service sessions are invalidated

**Story Points:** 5
**Priority:** 🔴 Must Have
**Sprint:** M2 — End-to-End Auth
**Status:** Draft
**Mapped Objective:** OBJ-01

---

### US-004: Role-Based Access Control (RBAC)

**As a** Platform Admin
**I want** to define roles in Keycloak and have services enforce them declaratively
**So that** different users and services have appropriate access levels across the platform

**Acceptance Criteria:**
- **AC-1:** Given roles defined in the Panomete realm (`admin`, `user`, `viewer`), When a JWT is issued, Then the token's `realm_access.roles` claim includes the user's assigned roles
- **AC-2:** Given a user with role `admin`, When they access an admin-only endpoint (`/api/admin/**`), Then the service allows access (200)
- **AC-3:** Given a user with role `user`, When they access an admin-only endpoint, Then the service denies access (403)
- **AC-4:** Given a service developer, When they annotate an endpoint with `@PreAuthorize("hasRole('admin')")`, Then Spring Security enforces the role check using the JWT claims — no additional code needed

**Story Points:** 3
**Priority:** 🔴 Must Have
**Sprint:** M3 — Production Hardening
**Status:** Draft
**Mapped Objective:** OBJ-01

---

### US-005: Service-to-Service Authentication (Client Credentials)

**As a** Service Developer
**I want** my backend service to authenticate to other backend services using the OAuth2 Client Credentials grant
**So that** internal service-to-service calls are secure without user intervention

**Acceptance Criteria:**
- **AC-1:** Given Service A configured with a client ID and secret, When it requests a token from Keycloak using `grant_type=client_credentials`, Then a JWT is returned with the service's client roles in the claims
- **AC-2:** Given Service A calls Service B with a valid service JWT, When Service B validates the token, Then Service B accepts the request and can identify the caller from the `sub` claim
- **AC-3:** Given Service A calls Service B without a token, When Service B receives the request, Then Service B returns 401
- **AC-4:** Given a service JWT with insufficient roles, When Service B checks permissions, Then Service B returns 403

**Story Points:** 3
**Priority:** 🟡 Should Have
**Sprint:** M3 — Production Hardening
**Status:** Draft
**Mapped Objective:** OBJ-01

---

### US-006: Token Introspection for Non-Java Services

**As a** Service Developer (building in Go, Rust, TypeScript, etc.)
**I want** to validate tokens via Keycloak's introspection endpoint
**So that** non-Java services that cannot validate JWTs locally can still participate in the platform's auth system

**Acceptance Criteria:**
- **AC-1:** Given a service with an opaque or reference access token, When it calls `POST /realms/panomete/protocol/openid-connect/token/introspect` with client credentials, Then Keycloak returns `{"active": true, ...claims}`
- **AC-2:** Given an expired or revoked token, When the introspection endpoint is called, Then Keycloak returns `{"active": false}`
- **AC-3:** Given the introspection endpoint, When called without proper client authentication, Then Keycloak returns 401
- **AC-4:** Given a JWT token (not opaque), When introspection is requested, Then Keycloak still returns the parsed claims (works for both JWT and opaque tokens)

**Story Points:** 2
**Priority:** 🟡 Should Have
**Sprint:** M3 — Production Hardening
**Status:** Draft
**Mapped Objective:** OBJ-01

---

## 5. Story Estimation Summary

| Story | Name | Points | Priority | Sprint |
|-------|------|--------|----------|--------|
| US-001 | Deploy Keycloak Instance | 5 | 🔴 | M1 |
| US-002 | Register OAuth2 Clients | 3 | 🔴 | M1 |
| US-003 | Login and SSO | 5 | 🔴 | M2 |
| US-004 | Role-Based Access Control | 3 | 🔴 | M3 |
| US-005 | Service-to-Service Auth | 3 | 🟡 | M3 |
| US-006 | Token Introspection | 2 | 🟡 | M3 |
| **Total** | | **21** | | |

---

## Related Documents

| Document | Relationship |
|----------|-------------|
| [[panomete_platform/011_business_objective]] | Platform-level objectives (OBJ-01) |
| [[flowero_guard/011_business_objective]] | Service-level objectives |
| [[flowero_guard/013_acceptance_criteria]] | BDD acceptance criteria for all stories |
| [[flowero_gate/012_user_stories]] | Gate routes auth traffic through Guard |
| [[flowero_discover/012_user_stories]] | Guard registers with Discover for health monitoring |

---

> **Template Standard:** Based on SWEBOK v4, ISO/IEC/IEEE 29148
> **Usage:** These stories define what Flowero Guard must deliver. Stories are refined with the Dev persona before sprint start.
