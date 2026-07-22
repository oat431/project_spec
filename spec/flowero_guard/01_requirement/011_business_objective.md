---
document_type: Business Objectives
version: "0.2"
status: Draft
author: "PO (Product Owner)"
created: "2026-07-22"
last_updated: "2026-07-22"
project_name: "Flowero Guard"
project_id: "PAN-GUARD-001"
parent_platform: "Panomete Platform"
sponsor: "Self"
ba_owner: "PO (Product Owner)"
classification: "Internal"
tags: [business-objectives, keycloak, oauth2, iam, panomete]
---

# Business Objectives — Flowero Guard

> **Service:** Flowero Guard (Keycloak IAM)
> **Platform:** Panomete Platform
> **Version:** 0.2 | **Status:** Draft — Updated per Design Review 2026-07-22
> **Last Updated:** 2026-07-22
>
> ⚠️ See [[panomete_platform/011_business_objective]] for platform-level objectives.

---

## 1. Service Mission

> Flowero Guard IS Keycloak — the identity provider for the Panomete Platform. Its job: **issue OAuth2/OIDC tokens** so that Gate and business services can validate them locally. No separate wrapper service. No per-request network calls for token validation.

## 2. Service Context

| Aspect | Detail |
|--------|--------|
| **Service Type** | Foundation — Identity Provider |
| **Technology** | Keycloak (Docker) using shared PostgreSQL 18 |
| **Port** | 8001 (internal) |
| **Domain** | `auth.panomete.com` (via Nginx) |
| **Dependencies** | Shared PostgreSQL 18 |
| **Consumed By** | Gate (JWKS endpoint), business services (token issuance) |
| **Related Services** | Nginx (routes `auth.panomete.com` → :8001), Flowero Discover (registers health) |

## 3. Service Objectives

### OBJ-GUARD-01: Deploy Keycloak with Realm-as-Code

| Field | Detail |
|-------|--------|
| **Statement** | Deploy Keycloak using shared PostgreSQL 18, with a pre-configured `panomete` realm imported from a version-controlled JSON file. |
| **Measurable** | `docker compose up` → Keycloak healthy within 60s. Realm, roles, and clients imported from file. Data survives restart. Dashboard at `auth.panomete.com`. |
| **Priority** | 🔴 Must Have |
| **Owner** | Dev persona |
| **Parent Objective** | OBJ-01 |

### OBJ-GUARD-02: OAuth2/OIDC Compliance

| Field | Detail |
|-------|--------|
| **Statement** | Expose standard OAuth2 endpoints (authorize, token, introspection, userinfo, JWKS) so Gate and business services can validate tokens locally. |
| **Measurable** | JWKS endpoint publicly accessible. Gate validates tokens locally with zero per-request network calls to Guard. Authorization code + client credentials flows functional. |
| **Priority** | 🔴 Must Have |
| **Owner** | Dev persona |
| **Parent Objective** | OBJ-01 |

### OBJ-GUARD-03: SSO Across All Services

| Field | Detail |
|-------|--------|
| **Statement** | Login once at `auth.panomete.com`. All services share the session. Logout terminates all sessions. |
| **Measurable** | Login → navigate to any service → no re-prompt. Logout → all services require re-auth. |
| **Priority** | 🔴 Must Have |
| **Owner** | Dev persona |
| **Parent Objective** | OBJ-01 |

---

## 4. Out of Scope

- User self-registration (MVP: admin creates users manually)
- Social login (Google, GitHub)
- MFA (TOTP/SMS)
- LDAP federation

---

## Related Documents

| Document | Relationship |
|----------|-------------|
| [[flowero_guard/012_user_stories]] | Stories that deliver these objectives |
| [[flowero_guard/013_acceptance_criteria]] | Testable criteria for each story |
| [[panomete_platform/011_business_objective]] | Platform-level objectives |
