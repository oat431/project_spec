---
document_type: Commit Messages / Changelog
version: "0.1"
status: Active
author: "DevOps Persona"
created: "2026-07-23"
last_updated: "2026-07-23"
project_name: "Flowero Guard"
project_id: "PAN-GUARD-001"
parent_platform: "Panomete Platform"
classification: "Internal"
tags: [commit-messages, changelog, conventional-commits, keycloak, panomete]
standard_ref:
  - SWEBOK v4 — Construction
  - Conventional Commits v1.0
---

# Commit Messages / Changelog — Flowero Guard

> **Service:** Flowero Guard (Keycloak IAM)
> **Platform:** Panomete Platform
> **Version:** 0.1 | **Status:** Active
> **Last Updated:** 2026-07-23

---

## 1. Purpose

> All Flowero Guard commits follow [Conventional Commits](https://www.conventionalcommits.org/). Since Guard is config-only, most commits are realm configuration changes.

## 2. Commit Message Format

```
<type>(guard): <description>

[optional body — explains WHY]

[optional footer(s)]
```

### Scopes

| Scope | When to use |
|-------|-------------|
| `guard` | Changes to Flowero Guard (default) |
| `realm` | Realm JSON changes (roles, clients, scopes, settings) |
| `docker` | Dockerfile or compose changes |

### Types

| Type | When to use | Example |
|------|-------------|---------|
| `feat` | New client, new role, new realm setting | `feat(guard): add cute-gufo OAuth2 client` |
| `fix` | Fix misconfiguration, fix broken client | `fix(guard): correct redirect URI for blog client` |
| `chore` | Realm export, dependency update, config cleanup | `chore(guard): export realm config 2026-07-23` |
| `docs` | Documentation changes | `docs(guard): update README with new client docs` |
| `security` | Security-related changes | `security(guard): enable brute force protection` |

## 3. Commit Guidelines

| Rule | Rationale |
|------|----------|
| Type + scope in every commit | `feat(guard): ...` not `add client` |
| Imperative mood | "add" not "added" |
| First line ≤ 72 chars | Readable in `git log --oneline` |
| Body explains WHY | Config shows what; commit explains why |
| One logical change per commit | Atomic — easy to revert |
| Export after every Admin Console change | Realm JSON must match production state |

## 4. Changelog

### [0.1.0] — 2026-07-23 (Initial Deployment)

#### Features
- **realm:** create `panomete` realm with `admin`, `user`, `viewer` roles
- **realm:** configure `default-roles-panomete` composite role
- **realm:** add `account` and `admin-cli` clients
- **realm:** add `web-origins`, `profile`, `email`, `roles` client scopes
- **realm:** configure `realm_access.roles` JWT mapper for Gate claim extraction
- **realm:** set access token lifespan to 5 minutes (300s)
- **realm:** set SSO session idle timeout to 30 minutes (1800s)
- **realm:** enable brute force protection
- **docker:** deploy Keycloak 26.7.0 with realm import on startup
- **docker:** configure `KC_CACHE=local` for single-node deployment
- **docker:** configure `KC_PROXY_HEADERS=xforwarded` for Nginx/Cloudflare

#### Infrastructure
- **guard:** create `keycloak` database + role on shared PostgreSQL 18
- **guard:** add Nginx server block for `auth.panomete.com`
- **guard:** hardcode `X-Forwarded-Proto: https` in Nginx for Cloudflare TLS

#### Post-Install
- **guard:** create permanent admin user via Admin Console
- **guard:** delete temporary bootstrap admin

### [0.0.1] — 2026-07-22 (Project Planning)

#### Documentation
- **docs:** define business objectives (OBJ-GUARD-01 through 05)
- **docs:** define user stories (US-001 through US-006)
- **docs:** define acceptance criteria (20 BDD scenarios)
- **docs:** write architecture decision records (7 ADRs)
- **docs:** write API specification (OIDC + JWKS + token endpoints)
- **docs:** write database schema DDL (PostgreSQL provisioning)

## 5. Realm Export Commits

> After ANY change in the Admin Console, export and commit:

```bash
# Export from Admin Console → Realm Settings → Action → Partial Export
# Save as panomete-realm.json

git add panomete-realm.json
git commit -m "chore(guard): export realm config 2026-07-23

Changes:
- Added cute-gufo OAuth2 client
- Assigned 'user' role to default-roles-panomete"
```

---

## Related Documents

| Document | Relationship |
|----------|-------------|
| [[035_coding_standards_development]] | Standards these commits follow |
| [[031_README_developer_guide]] | Developer setup guide |
| [[032_build_scripts]] | Build process triggered by commits |

---

> **Template Standard:** Based on SWEBOK v4, Conventional Commits v1.0
> **Usage:** Guard commits are mostly realm exports. Always include what changed in the body.
