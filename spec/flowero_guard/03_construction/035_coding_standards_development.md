---
document_type: Coding Standards / Style Guide
version: "0.1"
status: Draft
author: "DevOps Persona"
created: "2026-07-23"
last_updated: "2026-07-23"
project_name: "Flowero Guard"
project_id: "PAN-GUARD-001"
parent_platform: "Panomete Platform"
classification: "Internal"
tags: [coding-standards, style-guide, keycloak, realm-json, panomete]
standard_ref:
  - SWEBOK v4 — Construction
---

# Coding Standards — Flowero Guard

> **Service:** Flowero Guard (Keycloak IAM)
> **Platform:** Panomete Platform
> **Version:** 0.1 | **Status:** Draft
> **Last Updated:** 2026-07-23

---

## 1. Purpose

> Coding standards for Flowero Guard. Guard is config-only — the "code" is `panomete-realm.json`. These standards govern realm JSON structure, Docker configuration, and operational conventions.

## 2. Technology Stack

| Aspect | Standard |
|--------|---------|
| Runtime | Keycloak 26.7.0 (Quarkus-based) |
| Base Image | `quay.io/keycloak/keycloak:latest` |
| Configuration | Realm JSON (`panomete-realm.json`) |
| Database | PostgreSQL 18 (shared instance) |
| Protocol | OAuth2 / OIDC |

## 3. Realm JSON Standards

### 3.1 Structure

```json
{
    "id": "panomete",
    "realm": "panomete",
    "displayName": "Panomete Platform",
    "enabled": true,
    "sslRequired": "none",
    "registrationAllowed": false,
    "loginWithEmailAllowed": true,
    "bruteForceProtected": true,
    "roles": { ... },
    "clients": [ ... ],
    "clientScopes": [ ... ],
    "accessTokenLifespan": 300,
    "ssoSessionIdleTimeout": 1800
}
```

### 3.2 Rules

| Rule | Rationale |
|------|----------|
| Always include `"realm": "panomete"` | Must match `KC_HOSTNAME` config |
| `"enabled": true` | Realms are enabled by default |
| `"sslRequired": "none"` | Cloudflare handles TLS externally |
| `"registrationAllowed": false` | Admin creates users (ADR-G004) |
| `"bruteForceProtected": true` | Security baseline |
| All roles must have `description` | Self-documenting |
| All clients must have `name` | Human-readable in Admin Console |
| `optionalClientScopes` must reference existing scopes | Prevents import failures |
| No `default-roles-panomete` client role references | Keycloak creates default roles automatically |

### 3.3 Naming Conventions

| Entity | Convention | Example |
|--------|-----------|---------|
| Realm | lowercase, platform name | `panomete` |
| Roles | lowercase, hyphen-separated | `admin`, `user`, `viewer` |
| Client IDs | lowercase, service name | `cute-gufo`, `flowero-gateway` |
| Client Scopes | lowercase, hyphen-separated | `web-origins`, `profile` |
| Redirect URIs | HTTPS URLs with wildcards | `https://blog.panomete.com/*` |

### 3.4 Security Rules

| Rule | Rationale |
|------|----------|
| No plaintext passwords in realm JSON | Secrets in `.env` only |
| `confidential` clients only (ADR-G003) | All OAuth2 clients require secret |
| `directAccessGrantsEnabled: false` by default | Prefer Authorization Code flow |
| `publicClient: false` by default | No public clients in MVP |
| `bruteForceProtected: true` | Prevent credential stuffing |

## 4. Docker Configuration Standards

### 4.1 Environment Variables

| Variable | Source | Rule |
|----------|--------|------|
| `KC_DB_PASSWORD` | `.env` | Never in compose YAML |
| `KC_BOOTSTRAP_ADMIN_PASSWORD` | `.env` | Never in compose YAML |
| `KC_HOSTNAME` | compose | Must match Nginx `server_name` |
| `KC_PROXY_HEADERS` | compose | Always `xforwarded` behind Nginx |
| `KC_CACHE` | compose | Always `local` for single-node |

### 4.2 Port Mapping

```yaml
ports:
  - "127.0.0.1:8001:8080"   # Always bind to localhost
```

> **Rule:** Never expose ports to `0.0.0.0`. Nginx proxies externally.

### 4.3 Network

```yaml
networks:
  - shared-network   # Always join db-network
```

> **Rule:** Guard must be on `db-network` to reach PostgreSQL by container name.

## 5. Realm Change Workflow

```
1. Make change in Admin Console
2. Export realm: Realm Settings → Action → Partial Export
3. Validate JSON: jq . panomete-realm.json
4. Save to project: cp export.json panomete-realm.json
5. Commit: git commit -m "chore(guard): export realm config YYYY-MM-DD"
6. Redeploy (if needed): docker compose restart flowero-guard
```

> **Rule:** NEVER make realm changes without exporting. The JSON is the source of truth.

## 6. What NOT to Do

| ❌ Don't | ✅ Do Instead |
|----------|--------------|
| Edit realm JSON by hand (except merge conflicts) | Export from Admin Console |
| Put passwords in compose YAML | Use `.env` file |
| Use `KC_PROXY=edge` (deprecated) | Use `KC_PROXY_HEADERS=xforwarded` |
| Use `KEYCLOAK_ADMIN` (deprecated) | Use `KC_BOOTSTRAP_ADMIN_USERNAME` |
| Expose ports to `0.0.0.0` | Bind to `127.0.0.1` |
| Skip realm export after changes | Always export and commit |
| Use `default-roles-panomete` with client role references | Let Keycloak create defaults |

---

## Related Documents

| Document | Relationship |
|----------|-------------|
| [[031_README_developer_guide]] | Developer onboarding |
| [[034_commit_messages_changelog]] | Commit format enforced |
| [[036_code_review_records]] | Standards enforced during review |

---

> **Template Standard:** Based on SWEBOK v4
> **Usage:** Guard's "code" is the realm JSON. These standards ensure the JSON is valid, secure, and maintainable.
