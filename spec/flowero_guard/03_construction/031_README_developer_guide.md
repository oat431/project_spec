---
document_type: README / Developer Guide
version: "0.1"
status: Draft
author: "DevOps Persona"
created: "2026-07-23"
last_updated: "2026-07-23"
project_name: "Flowero Guard"
project_id: "PAN-GUARD-001"
parent_platform: "Panomete Platform"
classification: "Internal"
tags: [readme, developer-guide, keycloak, iam, panomete]
standard_ref:
  - SWEBOK v4 — Construction
  - 12-Factor App Methodology
---

# README / Developer Guide — Flowero Guard

> **Service:** Flowero Guard (Keycloak IAM)
> **Platform:** Panomete Platform
> **Version:** 0.1 | **Status:** Draft
> **Last Updated:** 2026-07-23

---

## 1. Purpose

> The README is the **front door** for developers working on or consuming Flowero Guard. Guard is the identity backbone of the platform — every service depends on it for JWT issuance and validation.

## 2. Project Header

```markdown
# Flowero Guard 🔐

> **Identity Provider (IAM)** for the Panomete Platform.
>
> Keycloak 26.7 · OAuth2 / OIDC · Realm-as-Code
```

| Aspect | Detail |
|--------|--------|
| **Service Type** | Foundation — Identity Provider |
| **Technology** | Keycloak (Quarkus-based) |
| **Version** | Keycloak 26.7.0 |
| **Port** | 8001 (internal, Keycloak listens on 8080, mapped to 8001) |
| **Domain** | `auth.panomete.com` (via Nginx → Cloudflare) |
| **Database** | Shared PostgreSQL 18 — `keycloak` database |
| **Protocol** | OAuth2 / OIDC (Authorization Code + Client Credentials) |
| **Config** | Realm-as-Code (`panomete-realm.json`) |

## 3. Architecture

```
                    ┌─ Admin Console ─── /admin ─── realm management
                    │
  flowero-guard ────┼─ OIDC Discovery ─── /.well-known/openid-configuration
  (Keycloak)        │
                    ├─ JWKS Endpoint ─── /protocol/openid-connect/certs
                    │                     ↓
                    │                   Flowero Gate caches this
                    │
                    └─ Token Endpoint ─── /protocol/openid-connect/token
                                          ↑
                                        Services exchange auth codes / credentials for JWTs
```

### How Services Use It

```
1. User visits blog.panomete.com → SPA detects no token → redirects to auth.panomete.com
2. Keycloak shows login page → user authenticates → Keycloak issues JWT
3. SPA stores JWT → sends Authorization: Bearer <jwt> to api.panomete.com
4. Gate validates JWT locally using cached JWKS (zero calls to Guard per request)
5. Gate extracts claims → X-User-Id, X-User-Roles → forwards to business service
```

## 4. Quick Start

### Prerequisites

- **Docker** + **Docker Compose**
- **PostgreSQL 18** running with `keycloak` database created
- **Nginx** configured for `auth.panomete.com`

### Deploy

```bash
# Clone
git clone <repo-url>
cd flowero-guard

# The realm JSON is the "source code"
cat panomete-realm.json | jq .realm
# "panomete"

# Build and run via platform compose
cd ~/platform
docker compose -f docker-compose.platform.yml up -d flowero-guard
```

### Verify

```bash
# Health check
curl -sf http://localhost:8001/health/ready
# Expected: 200 OK

# OIDC Discovery
curl -sf https://auth.panomete.com/realms/panomete/.well-known/openid-configuration | jq .issuer
# "https://auth.panomete.com/realms/panomete"

# JWKS
curl -sf https://auth.panomete.com/realms/panomete/protocol/openid-connect/certs | jq '.keys | length'
# 2

# Admin Console
open https://auth.panomete.com/admin
```

## 5. Configuration Reference

### Compose Environment Variables

| Variable | Value | Why |
|----------|-------|-----|
| `KC_DB` | `postgres` | Database vendor |
| `KC_DB_URL` | `jdbc:postgresql://local-postgres:5432/keycloak` | Container name on `db-network` |
| `KC_DB_USERNAME` | `keycloak` | Dedicated DB user |
| `KC_DB_PASSWORD` | *(from .env)* | Never committed |
| `KC_BOOTSTRAP_ADMIN_USERNAME` | `admin` | Temporary admin (see Post Install) |
| `KC_BOOTSTRAP_ADMIN_PASSWORD` | *(from .env)* | Temporary admin password |
| `KC_HOSTNAME` | `auth.panomete.com` | Canonical hostname for issuer URL |
| `KC_HTTP_ENABLED` | `true` | Accept HTTP internally (Cloudflare handles TLS) |
| `KC_PROXY_HEADERS` | `xforwarded` | Trust X-Forwarded-* from Nginx |
| `KC_CACHE` | `local` | No distributed caching — single node |

### Realm Configuration (`panomete-realm.json`)

| Item | Value |
|------|-------|
| Realm name | `panomete` |
| Roles | `admin`, `user`, `viewer` |
| Access token lifespan | 5 minutes (300s) |
| SSO session idle timeout | 30 minutes (1800s) |
| SSL required | `none` (Cloudflare handles TLS externally) |
| Registration allowed | `false` (admin creates users) |
| Brute force protection | `true` |

## 6. API Reference

See the full [[022_API_specification|API Specification]].

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `/.well-known/openid-configuration` | `GET` | OIDC discovery (all endpoint URLs) |
| `/protocol/openid-connect/certs` | `GET` | JWKS — public keys for JWT validation |
| `/protocol/openid-connect/token` | `POST` | Token exchange (auth code, refresh, client credentials) |
| `/protocol/openid-connect/auth` | `GET` | Authorization endpoint (browser redirect) |
| `/protocol/openid-connect/userinfo` | `GET` | User claims (requires Bearer JWT) |
| `/protocol/openid-connect/logout` | `GET` | End SSO session |
| `/protocol/openid-connect/token/introspect` | `POST` | Token introspection (for non-Java services) |
| `/admin/` | `GET` | Admin console (HTML) |
| `/health/ready` | `GET` | Health check |

## 7. Related Services

| Service | How It Uses Guard |
|---------|-------------------|
| **Flowero Gate** | Caches JWKS for local JWT validation. Zero per-request calls to Guard. |
| **All business services** | Trust `X-User-Id` and `X-User-Roles` headers forwarded by Gate |
| **Frontend SPAs** | Redirect to `auth.panomete.com` for login |

## 8. Testing Guard

Guard is config-only — there's no Java code to unit test. Verification is done via:

```bash
# OIDC discovery returns valid config
curl -sf https://auth.panomete.com/realms/panomete/.well-known/openid-configuration | jq .

# JWKS returns RSA keys
curl -sf https://auth.panomete.com/realms/panomete/protocol/openid-connect/certs | jq .keys[0].kty
# "RSA"

# Admin console loads
curl -sf -o /dev/null -w '%{http_code}' https://auth.panomete.com/admin/
# 302 (redirect to login)

# Realm JSON is valid
jq . panomete-realm.json > /dev/null && echo "Valid JSON"
```

## 9. Design Decisions

| ADR | Decision |
|-----|----------|
| ADR-G001 | Realm-as-Code (JSON export in version control) |
| ADR-G002 | Single Realm (`panomete`) for all platform services |
| ADR-G003 | Confidential Clients Only (all OAuth2 clients require secret) |
| ADR-G004 | Admin-Created Users (no self-registration in MVP) |
| ADR-G005 | Standard OAuth2 Flows Only (Authorization Code + Client Credentials) |
| ADR-G006 | Shared PostgreSQL 18 (no dedicated container) |
| ADR-G007 | Guard IS Keycloak (no wrapper service) |

Full ADRs: [[021_architecture_decision_records]]

## 10. Project Structure

```
flowero-guard/
├── panomete-realm.json          # Realm configuration (version-controlled)
├── Dockerfile                   # Keycloak image + realm overlay
├── docker-compose.fragment.yml  # Compose entry for platform
└── README.md
```

> **Note:** Unlike Discover and Gate, Guard has no source code. The `panomete-realm.json` IS the codebase. Changes to the realm = changes to the service.

---

## Related Documents

| Document | Relationship |
|----------|-------------|
| [[011_business_objective]] | Service objectives (OBJ-GUARD-01 through 05) |
| [[012_user_stories]] | 6 user stories (US-001 through US-006) |
| [[013_acceptance_criteria]] | 20 BDD acceptance criteria |
| [[021_architecture_decision_records]] | 7 service-level ADRs |
| [[022_API_specification]] | Keycloak OAuth2/OIDC endpoints |
| [[023_database_schema_DDL]] | PostgreSQL provisioning |
| [[032_build_scripts]] | Docker build + realm import |

---

> **Template Standard:** Based on SWEBOK v4, 12-Factor App
> **Usage:** Guard is config-only. The realm JSON is the source of truth. Always export after Admin Console changes.
