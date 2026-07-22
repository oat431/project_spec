---
document_type: API Specification
version: "0.1"
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
  - SWEBOK v4 — Design
  - OpenAPI Specification 3.0
  - OAuth 2.0 (RFC 6749)
  - OpenID Connect Core 1.0
---

# API Specification — Flowero Guard

> **Service:** Flowero Guard (Keycloak IAM)
> **Platform:** Panomete Platform
> **Version:** 0.1 | **Status:** Draft
> **Last Updated:** 2026-07-22

---

## 1. Purpose

> Flowero Guard is a **Keycloak instance** — its APIs are the standard OAuth2/OIDC endpoints defined by RFC 6749 and OpenID Connect. This document catalogs the relevant endpoints the Panomete Platform consumes. This is NOT a custom API — it's Keycloak's built-in REST API.

---

## 2. API Overview

| Field | Detail |
|-------|--------|
| **Base URL (internal)** | `http://flowero-guard:8080` |
| **Base URL (external)** | `https://panomete.local/auth` (via Flowero Gate) |
| **Realm** | `panomete` |
| **Protocol** | HTTPS (external), HTTP (internal Docker network) |
| **Format** | JSON |
| **Authentication** | Client credentials (for introspection/admin endpoints) or public (for login/token endpoints) |
| **Discovery** | `GET /auth/realms/panomete/.well-known/openid-configuration` |

---

## 3. OAuth2 / OIDC Endpoints

> All paths are relative to the realm base: `/auth/realms/panomete/protocol/openid-connect/`

### 3.1 OpenID Connect Discovery

#### GET /.well-known/openid-configuration

| Field | Detail |
|-------|--------|
| **Description** | Returns the OIDC discovery document — all endpoint URLs, supported grant types, scopes, and JWKS URI |
| **Auth** | None (public) |

**Response (200):**
```json
{
  "issuer": "https://panomete.local/auth/realms/panomete",
  "authorization_endpoint": "https://panomete.local/auth/realms/panomete/protocol/openid-connect/auth",
  "token_endpoint": "https://panomete.local/auth/realms/panomete/protocol/openid-connect/token",
  "introspection_endpoint": "https://panomete.local/auth/realms/panomete/protocol/openid-connect/token/introspect",
  "userinfo_endpoint": "https://panomete.local/auth/realms/panomete/protocol/openid-connect/userinfo",
  "end_session_endpoint": "https://panomete.local/auth/realms/panomete/protocol/openid-connect/logout",
  "jwks_uri": "https://panomete.local/auth/realms/panomete/protocol/openid-connect/certs",
  "grant_types_supported": ["authorization_code", "client_credentials", "refresh_token"]
}
```

---

### 3.2 Authorization Endpoint

#### GET /auth

> Initiates the Authorization Code flow. Redirects user to Keycloak login page.

| Field | Detail |
|-------|--------|
| **Description** | Start the OAuth2 Authorization Code flow |
| **Auth** | None (redirects to login) |

**Query Parameters:**

| Param | Required | Description |
|-------|:---:|-------------|
| `response_type` | Yes | `code` |
| `client_id` | Yes | OAuth2 client ID (e.g., `fluffy-mouton`) |
| `redirect_uri` | Yes | Post-login redirect URL |
| `scope` | No | Space-separated scopes (default: `openid profile email`) |
| `state` | No | Opaque value returned in redirect (CSRF protection) |

**Response:** 302 redirect to Keycloak login page.

---

### 3.3 Token Endpoint

#### POST /token

> Exchanges an authorization code (or refresh token, or client credentials) for tokens.

| Field | Detail |
|-------|--------|
| **Description** | Obtain access, refresh, and ID tokens |
| **Content-Type** | `application/x-www-form-urlencoded` |

**Grant Types Supported:**

| Grant Type | `grant_type` value | Auth Required | Use Case |
|-----------|-------------------|:---:|---|
| Authorization Code | `authorization_code` | No (code is the credential) | Browser-based login |
| Refresh Token | `refresh_token` | No (refresh token is the credential) | Silent token renewal |
| Client Credentials | `client_credentials` | Yes (client ID + secret) | Service-to-service calls |

**Request — Authorization Code:**
```
POST /auth/realms/panomete/protocol/openid-connect/token
Content-Type: application/x-www-form-urlencoded

grant_type=authorization_code
&code={authorization_code}
&redirect_uri={same_as_auth_request}
&client_id={client_id}
&client_secret={client_secret}
```

**Request — Client Credentials:**
```
POST /auth/realms/panomete/protocol/openid-connect/token
Content-Type: application/x-www-form-urlencoded

grant_type=client_credentials
&client_id={client_id}
&client_secret={client_secret}
```

**Response (200):**
```json
{
  "access_token": "eyJhbG...",
  "expires_in": 300,
  "refresh_expires_in": 1800,
  "refresh_token": "eyJhbG...",
  "token_type": "Bearer",
  "id_token": "eyJhbG...",
  "session_state": "...",
  "scope": "openid profile email"
}
```

**Error (401):**
```json
{
  "error": "invalid_client",
  "error_description": "Invalid client credentials"
}
```

---

### 3.4 Token Introspection Endpoint

#### POST /token/introspect

> Validates a token and returns its claims. Used by non-Java services that cannot validate JWTs locally.

| Field | Detail |
|-------|--------|
| **Description** | Introspect an access or refresh token |
| **Auth** | Client credentials (Basic Auth — client ID:secret) |

**Request:**
```
POST /auth/realms/panomete/protocol/openid-connect/token/introspect
Content-Type: application/x-www-form-urlencoded
Authorization: Basic base64(client_id:client_secret)

token={access_token_or_refresh_token}
```

**Response — Active Token (200):**
```json
{
  "active": true,
  "sub": "user-uuid",
  "username": "alice",
  "email": "alice@panomete.local",
  "realm_access": {
    "roles": ["admin", "user"]
  },
  "exp": 1750000000,
  "iat": 1749999700,
  "iss": "https://panomete.local/auth/realms/panomete"
}
```

**Response — Inactive Token (200):**
```json
{
  "active": false
}
```

---

### 3.5 UserInfo Endpoint

#### GET /userinfo

> Returns claims about the authenticated user.

| Field | Detail |
|-------|--------|
| **Description** | Get current user's profile claims |
| **Auth** | Bearer JWT (access token) |

**Request:**
```
GET /auth/realms/panomete/protocol/openid-connect/userinfo
Authorization: Bearer {access_token}
```

**Response (200):**
```json
{
  "sub": "user-uuid",
  "email_verified": true,
  "name": "Alice",
  "preferred_username": "alice",
  "given_name": "Alice",
  "email": "alice@panomete.local"
}
```

---

### 3.6 JWKS (JSON Web Key Set) Endpoint

#### GET /certs

> Returns the public keys used to sign JWTs. **This is the most critical endpoint for the platform** — Flowero Gate downloads this at startup to validate JWT signatures locally.

| Field | Detail |
|-------|--------|
| **Description** | Get public keys for JWT signature verification |
| **Auth** | None (public) |

**Response (200):**
```json
{
  "keys": [
    {
      "kid": "abc123",
      "kty": "RSA",
      "alg": "RS256",
      "use": "sig",
      "n": "...",
      "e": "AQAB"
    }
  ]
}
```

---

### 3.7 Logout Endpoint

#### GET /logout

> Terminates the Keycloak session. After logout, all SSO sessions across services are invalidated.

| Field | Detail |
|-------|--------|
| **Description** | End user session (SSO logout) |
| **Auth** | None (requires session cookie or `id_token_hint`) |

**Query Parameters:**

| Param | Required | Description |
|-------|:---:|-------------|
| `id_token_hint` | No | ID token — identifies the session to terminate |
| `post_logout_redirect_uri` | No | Where to redirect after logout |

---

## 4. Error Codes

| HTTP Status | Error Code | Description |
|:---:|------|-------------|
| 400 | `invalid_grant` | Invalid authorization code or refresh token |
| 400 | `invalid_request` | Missing required parameters |
| 401 | `invalid_client` | Invalid client ID or secret |
| 401 | `invalid_token` | Token is expired, tampered, or from wrong issuer |
| 403 | `unauthorized_client` | Client not authorized for this grant type |
| 500 | `server_error` | Keycloak internal error |

---

## 5. Token Claims (JWT Payload)

> The JWT access token issued by Keycloak contains these claims:

| Claim | Type | Description |
|-------|------|-------------|
| `sub` | UUID | User's unique ID (Keycloak internal UUID) |
| `preferred_username` | string | Username |
| `email` | string | Email address |
| `email_verified` | boolean | Whether email is verified |
| `realm_access.roles` | string[] | Realm-level roles (`admin`, `user`, `viewer`) |
| `resource_access.{client_id}.roles` | string[] | Client-level roles |
| `iss` | string | Issuer URL |
| `exp` | number | Expiration timestamp (Unix seconds) |
| `iat` | number | Issued-at timestamp |
| `aud` | string | Audience (client ID) |

**Example decoded JWT payload:**
```json
{
  "sub": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "preferred_username": "alice",
  "email": "alice@panomete.local",
  "email_verified": true,
  "realm_access": {
    "roles": ["admin", "user"]
  },
  "iss": "https://panomete.local/auth/realms/panomete",
  "exp": 1750000000,
  "iat": 1749999700,
  "aud": "account"
}
```

---

## 6. Flowero Gate — How It Consumes Guard

> The Gateway does NOT call Guard's introspect endpoint on every request. Instead:

1. **At startup:** Gate downloads `GET /certs` (JWKS) and caches the public key
2. **At startup (periodic refresh):** Gate re-fetches JWKS on a schedule (default: every 5 minutes)
3. **Per request:** Gate validates the JWT signature **locally** using the cached public key — no network call
4. **Per request:** Gate extracts `sub` → `X-User-Id` header, `realm_access.roles` → `X-User-Roles` header
5. **On Keycloak restart / key rotation:** Spring Security's `NimbusJwtDecoder` automatically handles cache invalidation and new key retrieval

---

## 7. Admin REST API (Out of Scope for MVP)

> Keycloak provides a full Admin REST API at `/auth/admin/realms/panomete/`. This is used programmatically in future phases for:
> - Creating clients via API
> - Managing users
> - Resetting credentials

> For MVP, admin operations are done through the Keycloak Admin Console UI (accessible at `/auth/admin/`).

---

## Related Documents

| Document | Relationship |
|----------|-------------|
| [[flowero_guard/021_architecture_decision_records]] | Guard-specific ADRs |
| [[flowero_guard/024_ERD]] | Logical realm data model |
| [[flowero_guard/023_database_schema_DDL]] | Keycloak's database backend |
| [[flowero_gate/022_API_specification]] | Gate consumes these endpoints for auth |
| [[flowero_guard/012_user_stories]] | Stories this API supports |

---

> **Template Standard:** Based on SWEBOK v4, OAuth 2.0 (RFC 6749), OpenID Connect Core 1.0
> **Usage:** This document is the reference for any developer integrating a service with Flowero Guard. For the full Keycloak API reference, see [Keycloak Server Administration Guide](https://www.keycloak.org/docs/latest/server_admin/).
