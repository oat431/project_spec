---
document_type: Database Schema (DDL)
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
tags: [database-schema, ddl, postgresql, keycloak, panomete]
standard_ref:
  - SWEBOK v4 — Design
---

# Database Schema (DDL) — Flowero Guard

> **Service:** Flowero Guard (Keycloak IAM)
> **Platform:** Panomete Platform
> **Version:** 0.2 | **Status:** Draft — Updated per Design Review 2026-07-22
> **Last Updated:** 2026-07-22

---

## 1. Important

> ⚠️ **We do NOT write DDL for Keycloak.** Keycloak uses Liquibase to manage its own schema. This document defines the database provisioning and connection configuration.

---

## 2. Database Provisioning

> Run once on the existing shared PostgreSQL 18 instance before starting Keycloak.

```sql
-- Create the Keycloak database on the shared PostgreSQL 18 instance
CREATE DATABASE keycloak
    WITH ENCODING 'UTF8'
    LC_COLLATE = 'en_US.UTF-8'
    LC_CTYPE = 'en_US.UTF-8';

-- Grant to Keycloak user
GRANT ALL PRIVILEGES ON DATABASE keycloak TO keycloak;
GRANT ALL ON SCHEMA public TO keycloak;
```

---

## 3. Keycloak Datasource Configuration

> Keycloak connects to the shared PostgreSQL 18 via JDBC.

| Parameter | Value | Notes |
|-----------|-------|-------|
| `KC_DB` | `postgres` | Database vendor |
| `KC_DB_URL` | `jdbc:postgresql://host.docker.internal:5432/keycloak` | Shared PostgreSQL 18 |
| `KC_DB_USERNAME` | `keycloak` | Database user |
| `KC_DB_PASSWORD` | From `.env` file | Never commit to git |
| `KC_DB_POOL_INITIAL_SIZE` | `5` | Connection pool |
| `KC_DB_POOL_MAX_SIZE` | `20` | Max connections |

> **Note:** `host.docker.internal` is used if PostgreSQL runs on the host (not in a container). If PostgreSQL is containerized on the same Docker network, use the container name.

---

## 4. Keycloak Internal Schema (Reference Only)

> Keycloak creates these core tables on first boot. **Do NOT modify manually.**

| Table | Purpose |
|-------|---------|
| `REALM` | Realm configuration |
| `CLIENT` | OAuth2 clients |
| `USER_ENTITY` | Users |
| `USER_ROLE_MAPPING` | User-to-role assignments |
| `KEYCLOAK_ROLE` | Realm and client roles |
| `CREDENTIAL` | Hashed passwords/credentials |
| `USER_SESSION` | Active sessions |
| `EVENT_ENTITY` | Audit events |

---

## 5. Realm Configuration (JSON)

> Version-controlled at `keycloak/panomete-realm.json`. Key configuration:

```json
{
  "realm": "panomete",
  "enabled": true,
  "roles": { "realm": [
    {"name": "admin"}, {"name": "user"}, {"name": "viewer"}
  ]},
  "accessTokenLifespan": 300,
  "ssoSessionIdleTimeout": 1800
}
```

---

## 6. Backup Strategy

| Strategy | Frequency | Method |
|----------|-----------|--------|
| Full DB backup | Daily | `pg_dump -U postgres keycloak > backup.sql` |
| Realm export | On change | Version-controlled JSON |

---

## Related Documents

| Document | Relationship |
|----------|-------------|
| [[flowero_guard/024_ERD]] | Logical realm data model |
| [[flowero_guard/022_API_specification]] | Endpoints served by this database |
| [[panomete_platform/025_software_architecture_document]] | Platform deployment topology |
