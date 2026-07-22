---
document_type: API Specification
version: "0.2"
status: Draft
author: "Dev / SA Persona"
created: "2026-07-22"
last_updated: "2026-07-22"
project_name: "Flowero Guard"
project_id: "PAN-GUARD-001"
parent_platform: "Panomete Platform"
tech_lead: "Dev / SA Persona"
classification: "Internal"
tags: [api-specification, keycloak, oauth2, oidc, panomete]
standard_ref:
  - OAuth 2.0 (RFC 6749)
  - OpenID Connect Core 1.0
---

# API Specification — Flowero Guard

> **Service:** Flowero Guard (Keycloak IAM)
> **Platform:** Panomete Platform
> **Version:** 0.2 | **Status:** Draft — Updated per Design Review 2026-07-22
> **Last Updated:** 2026-07-22

---

## 1. Purpose

> Flowero Guard is a **Keycloak instance** — its APIs are the standard OAuth2/OIDC endpoints. This document catalogues the endpoints consumed by the Panomete Platform. Guard is accessed directly at `auth.panomete.com` through Nginx — NOT proxied through Gate.

---

## 2. API Overview

| Field | Detail |
|-------|--------|
| **Internal Address** | `http://flowero-guard:8001` |
| **External Address** | `https://auth.panomete.com` (via Nginx → Cloudflare) |
| **Realm** | `panomete` |
| **Protocol** | HTTPS (external via Cloudflare), HTTP (internal Docker network) |
| **Format** | JSON |
| **Discovery** | `GET /.well-known/openid-configuration` |

---

## 3. Key Endpoints

### 3.1 OpenID Connect Discovery

#### GET /.well-known/openid-configuration

> Public. Returns all OIDC endpoint URLs.

**Response (200):**
```json
{
  "issuer": "https://auth.panomete.com/realms/panomete",
  "authorization_endpoint": "https://auth.panomete.com/realms/panomete/protocol/openid-connect/auth",
  "token_endpoint": "https://auth.panomete.com/realms/panomete/protocol/openid-connect/token",
  "jwks_uri": "https://auth.panomete.com/realms/panomete/protocol/openid-connect/certs",
  "introspection_endpoint": "https://auth.panomete.com/realms/panomete/protocol/openid-connect/token/introspect",
  "userinfo_endpoint": "https://auth.panomete.com/realms/panomete/protocol/openid-connect/userinfo",
  "end_session_endpoint": "https://auth.panomete.com/realms/panomete/protocol/openid-connect/logout"
}
```

---

### 3.2 Authorization Endpoint

#### GET /auth

> Initiates Authorization Code flow. Redirects user to Keycloak login.

| Param | Required | Description |
|-------|:---:|-------------|
| `response_type` | Yes | `code` |
| `client_id` | Yes | OAuth2 client ID |
| `redirect_uri` | Yes | Post-login redirect |
| `scope` | No | `openid profile email` |
| `state` | No | CSRF protection |

---

### 3.3 Token Endpoint

#### POST /token

> Exchanges authorization codes, refresh tokens, or client credentials for tokens.

**Grant Types:**

| Grant | `grant_type` | Auth Required | Use Case |
|-------|-------------|:---:|---|
| Authorization Code | `authorization_code` | No | Browser login |
| Refresh Token | `refresh_token` | No | Silent renewal |
| Client Credentials | `client_credentials` | Yes (client ID+secret) | S2S calls |

**Response (200):**
```json
{
  "access_token": "eyJhbG...",
  "expires_in": 300,
  "refresh_expires_in": 1800,
  "refresh_token": "eyJhbG...",
  "token_type": "Bearer",
  "scope": "openid profile email"
}
```

---

### 3.4 JWKS Endpoint (Critical)

#### GET /certs

> **Most critical endpoint for the platform.** Gate downloads this at startup to cache Keycloak's public key for local JWT validation. Zero per-request calls to Guard for auth.

| Field | Detail |
|-------|--------|
| **Auth** | None (public) |

**Response (200):**
```json
{
  "keys": [{
    "kid": "abc123", "kty": "RSA", "alg": "RS256",
    "use": "sig", "n": "...", "e": "AQAB"
  }]
}
```

---

### 3.5 Token Introspection

#### POST /token/introspect

> Validates a token and returns claims. Used by non-Java services.

**Request:** `POST` with `Authorization: Basic base64(client_id:client_secret)` and `token={token}`

**Response — Active:** `{"active": true, "sub": "...", "realm_access": {"roles": ["admin"]}, ...}`
**Response — Inactive:** `{"active": false}`

---

### 3.6 UserInfo

#### GET /userinfo

> Returns claims about the authenticated user. Auth: Bearer JWT.

### 3.7 Logout

#### GET /logout

> Terminates SSO session. `?id_token_hint={id_token}&post_logout_redirect_uri={url}`

---

## 4. Token Claims (JWT Payload)

| Claim | Type | Description |
|-------|------|-------------|
| `sub` | UUID | User's Keycloak UUID |
| `preferred_username` | string | Username |
| `email` | string | Email |
| `realm_access.roles` | string[] | Roles: `admin`, `user`, `viewer` |
| `iss` | string | `https://auth.panomete.com/realms/panomete` |
| `exp` | number | Expiration (Unix seconds) |
| `iat` | number | Issued-at |

---

## 5. How Gate Consumes Guard

> Gate does NOT call Guard per request. Instead:

1. **At startup:** Gate downloads `GET /certs` (JWKS) → caches public key
2. **Periodic refresh:** Gate re-fetches JWKS every 5 minutes (Spring Security default)
3. **Per request:** Gate validates JWT signature **locally** — zero network call
4. **Claim extraction:** Gate extracts `sub` → `X-User-Id`, `realm_access.roles` → `X-User-Roles`

---

## 6. Error Codes

| HTTP | Error | Description |
|:---:|-------|-------------|
| 400 | `invalid_grant` | Invalid auth code or refresh token |
| 401 | `invalid_client` | Invalid client credentials |
| 401 | `invalid_token` | Expired/tampered token |
| 403 | `unauthorized_client` | Client not authorized for grant type |

---

## Related Documents

| Document | Relationship |
|----------|-------------|
| [[flowero_guard/021_architecture_decision_records]] | Guard-specific ADRs |
| [[flowero_guard/024_ERD]] | Logical realm data model |
| [[flowero_guard/023_database_schema_DDL]] | Database provisioning |
| [[flowero_gate/022_API_specification]] | Gate consumes JWKS + token endpoints |
