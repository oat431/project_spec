---
document_type: Operations Manual / Runbook
version: "0.1"
status: Draft
author: "DevOps Persona"
created: "2026-07-23"
last_updated: "2026-07-23"
project_name: "Flowero Guard"
project_id: "PAN-GUARD-001"
parent_platform: "Panomete Platform"
classification: "Internal"
tags: [runbook, operations, troubleshooting, keycloak, panomete]
standard_ref:
  - SWEBOK v4 — Operations
  - SEBoK v2 — Operations
  - ITIL v4 — Service Operation
---

# Operations Manual / Runbook — Flowero Guard

> **Service:** Flowero Guard (Keycloak IAM)
> **Platform:** Panomete Platform
> **Version:** 0.1 | **Status:** Draft — Awaiting PO Review
> **Last Updated:** 2026-07-23

---

## 1. Purpose

> Step-by-step operational procedures for Flowero Guard (Keycloak). Covers health monitoring, admin operations, troubleshooting, backup/restore, and emergency procedures.

---

## 2. System Overview

| Component | Technology | Container | Port | Domain |
|-----------|-----------|-----------|:---:|--------|
| **Guard** | Keycloak 25+ | `flowero-guard` | `127.0.0.1:8001` | `auth.panomete.com` |
| **Database** | PostgreSQL 18 (shared) | `local-postgres` | `local-postgres:5432` | — (internal) |
| **Edge Proxy** | Nginx | _(host process)_ | `:80` | Routes `auth.*` |
| **TLS** | Cloudflare Tunnel | _(host process)_ | — | `*.panomete.com` |

### Key URLs

| Resource | URL |
|----------|-----|
| Admin Console | `https://auth.panomete.com/admin` |
| Account Console | `https://auth.panomete.com/realms/panomete/account` |
| OIDC Discovery | `https://auth.panomete.com/realms/panomete/.well-known/openid-configuration` |
| JWKS | `https://auth.panomete.com/realms/panomete/protocol/openid-connect/certs` |
| Token Endpoint | `https://auth.panomete.com/realms/panomete/protocol/openid-connect/token` |

### Access

| Resource | Method |
|----------|--------|
| Admin Console | Username + password from `.env` (`KEYCLOAK_ADMIN` / `KEYCLOAK_ADMIN_PASSWORD`) |
| PostgreSQL (CLI) | `docker exec -it local-postgres psql -U postgres -d keycloak` |
| Guard container logs | `docker logs -f flowero-guard` |
| Server SSH | `ssh flowero@remote.panomete.com` |

---

## 3. Health Monitoring

### 3.1 Daily Health Check

```bash
# Health endpoint
curl -sf http://localhost:8001/health/ready && echo "Guard ✅" || echo "Guard ❌"

# OIDC discovery (verifies realm is loaded)
curl -sf https://auth.panomete.com/realms/panomete/.well-known/openid-configuration | jq .issuer

# JWKS (critical for Gate's JWT validation)
curl -sf https://auth.panomete.com/realms/panomete/protocol/openid-connect/certs | jq '.keys | length'

# Container status
docker ps --format '{{.Names}}\t{{.Status}}' | grep flowero-guard
```

### 3.2 Key Metrics to Watch

| Metric | Healthy | Warning | Critical | Where to Check |
|--------|:---:|:---:|:---:|----------------|
| Container status | Up | Restarting | Exited | `docker ps` |
| Health endpoint | 200 | — | Non-200 / timeout | `curl :8001/health/ready` |
| JVM Heap | < 60% | 60-80% | > 80% | `docker stats flowero-guard` |
| PostgreSQL connections | < 10 | 10-15 | > 20 (pool max) | Guard logs, PG `pg_stat_activity` |
| Login success rate | > 95% | 90-95% | < 90% | Keycloak events |
| Response time (token endpoint) | < 200ms | 200-500ms | > 500ms | Manual curl timing |

---

## 4. Routine Operations

### 4.1 Restart Guard

```bash
cd ~/platform
docker compose -f docker-compose.platform.yml restart flowero-guard

# Wait for Keycloak to fully start (Liquibase check + realm import)
echo "Waiting for Guard..."
for i in $(seq 1 30); do
    if curl -sf http://localhost:8001/health/ready > /dev/null 2>&1; then
        echo "Guard healthy after ${i}0 seconds"
        break
    fi
    sleep 10
done
```

### 4.2 Export Realm Configuration

> **Do this after ANY realm change** (new client, new role, user changes, flow changes).

```bash
# Option A: Via Admin Console
# 1. Navigate to https://auth.panomete.com/admin
# 2. Select "panomete" realm
# 3. Realm Settings → Action → Partial Export
# 4. Check "Include groups and roles" + "Include clients"
# 5. Download JSON
# 6. Save to repo: flowero-guard/panomete-realm.json
# 7. Commit: git commit -m "chore(guard): export realm config YYYY-MM-DD"

# Option B: Via Keycloak CLI (kcadm)
docker exec -it flowero-guard /opt/keycloak/bin/kcadm.sh config credentials \
    --server http://localhost:8080 \
    --realm master \
    --user $KEYCLOAK_ADMIN \
    --password $KEYCLOAK_ADMIN_PASSWORD

docker exec flowero-guard /opt/keycloak/bin/kcadm.sh get \
    realms/panomete > ~/panomete-realm-export.json
```

### 4.3 Register a New OAuth2 Client

> When onboarding a new business service.

1. **Admin Console** → `panomete` realm → Clients → Create client
2. **Client ID:** `<service-name>` (e.g., `cute-gufo`)
3. **Client type:** Confidential (requires secret)
4. **Standard flow:** Enabled (for browser login redirect)
5. **Service accounts:** Enabled (for S2S client credentials flow)
6. **Valid redirect URIs:** `https://<service>.panomete.com/*`
7. **Web origins:** `https://<service>.panomete.com`
8. Save → **Credentials** tab → Copy client secret
9. Add secret to `~/platform/.env`: `<SERVICE>_CLIENT_SECRET=<secret>`
10. Export realm JSON (see 4.2)

### 4.4 Create a New User

1. **Admin Console** → `panomete` realm → Users → Add user
2. Fill in username, email, first/last name
3. Save → **Credentials** tab → Set password (temporary = user must change on first login)
4. **Role Mappings** tab → Assign realm roles (`admin`, `user`, or `viewer`)
5. Export realm JSON (see 4.2)

### 4.5 Backup

| Data | Method | Frequency | Location |
|------|--------|-----------|----------|
| keycloak DB (full) | `docker exec local-postgres pg_dump -U postgres keycloak > ~/backups/keycloak_$(date +%Y%m%d).sql` | Daily | `~/backups/` → rclone → OneDrive |
| Realm config | JSON export from Admin Console | On change | Version-controlled in git |
| Guard container config | `~/platform/docker-compose.platform.yml` + `.env` | On change | Version-controlled |

---

## 5. Troubleshooting Guide

### Incident 1: Guard won't start — JDBC connection failed

```
Symptom:    Container starts, then exits. Logs show "Connection refused" or
            "FATAL: password authentication failed for user 'keycloak'"
Cause:      Wrong KC_DB_URL, wrong password, or PostgreSQL not running.
Diagnosis:  docker logs flowero-guard 2>&1 | grep -iE "jdbc|database|connection|auth"
Resolution:
            1. Verify PostgreSQL: docker exec local-postgres pg_isready -U postgres
            2. Verify DB exists: docker exec local-postgres psql -U postgres -c "\l keycloak"
            3. Verify role: docker exec local-postgres psql -U postgres -c "\du keycloak"
            4. Test connection:
               docker exec local-postgres psql -U keycloak -d keycloak -h local-postgres -c "SELECT 1"
            5. Check .env: grep KC_DB ~/platform/.env
            6. Fix credentials → restart: docker compose restart flowero-guard
```

### Incident 2: Guard won't start — Liquibase migration failure

```
Symptom:    Container starts, Keycloak begins Liquibase, then fails.
            Logs show "Liquibase: Update has been unsuccessful" or
            "Migration failed for change set ..."
Cause:      Corrupted schema, partial migration, or version downgrade.
Diagnosis:  docker logs flowero-guard 2>&1 | grep -i "liquibase\|migration\|changelog"
Resolution:
            1. Check if this is a first deploy or an upgrade
            2. If first deploy: DROP and recreate the keycloak DB (see deployment plan rollback)
            3. If upgrade: check Keycloak version compatibility
            4. Last resort: DROP DATABASE keycloak → recreate → redeploy
               ⚠️ Realm data lost — re-import from panomete-realm.json
```

### Incident 3: Realm "panomete" not imported

```
Symptom:    Guard starts, admin console accessible, but "panomete" realm is missing.
            Only "master" realm is present.
Cause:      Realm JSON file not mounted, or --import-realm flag missing.
Diagnosis:
            docker exec flowero-guard ls /opt/keycloak/data/import/
            # Should show: panomete-realm.json
            docker inspect flowero-guard --format '{{.Config.Cmd}}'
            # Should include: --import-realm
Resolution:
            1. Check compose volume mount for panomete-realm.json
            2. Ensure command includes "start --import-realm"
            3. Rebuild image with updated realm JSON
            4. Force recreate: docker compose up -d --force-recreate flowero-guard
```

### Incident 4: Admin console returns 502 Bad Gateway

```
Symptom:    https://auth.panomete.com/admin returns 502
Cause:      Guard container down, port 8001 not bound, or Nginx misconfigured.
Diagnosis:
            docker ps | grep flowero-guard          # Is it running?
            ss -tlnp | grep 8001                    # Is port bound?
            curl -sf http://localhost:8001/health   # Does it respond?
            curl -sf -H 'Host: auth.panomete.com' http://localhost  # Nginx routing?
Resolution:
            1. If container down: docker compose restart flowero-guard
            2. If port not bound: check compose ports: "127.0.0.1:8001:8080"
            3. If Nginx: check /etc/nginx/sites-enabled/auth.panomete.com
            4. Test Nginx: sudo nginx -t && sudo systemctl reload nginx
```

### Incident 5: JWKS endpoint unreachable — Gate can't validate JWTs

```
Symptom:    Gate returns 401 for ALL requests, even valid tokens.
            Gate logs show "Couldn't retrieve remote JWK set" or
            "Error fetching JWK set".
Cause:      Guard down, JWKS endpoint changed, or network issue between Gate and Guard.
Diagnosis:
            curl -sf https://auth.panomete.com/realms/panomete/protocol/openid-connect/certs
            # Should return JSON with "keys" array
Resolution:
            1. If Guard down: restart Guard
            2. If Guard up but JWKS 404: realm "panomete" not loaded (Incident 3)
            3. Restart Gate to force JWKS cache refresh:
               docker compose restart flowero-gate
```

### Incident 6: Login fails — "Invalid username or password"

```
Symptom:    Users cannot log in. "Invalid username or password" error.
Cause:      Wrong credentials, user not created, user disabled, or
            brute-force protection locked the account.
Diagnosis:
            1. Admin Console → panomete realm → Users → search for user
            2. Check "Enabled" flag
            3. Check Events tab for failed login attempts
Resolution:
            1. Reset user password (Credentials tab)
            2. If brute-force locked: Authentication → Brute force detection →
               temporarily disable, or wait for lockout to expire
            3. Re-enable user if disabled
```

### Incident 7: High memory usage / OOM killed

```
Symbol:     docker stats shows Guard near 1GB limit.
            Container may be OOM-killed (docker ps shows "Exited (137)").
Cause:      JVM heap too small, too many sessions, or memory leak.
Diagnosis:
            docker stats flowero-guard
            docker inspect flowero-guard --format '{{.State.OOMKilled}}'
Resolution:
            1. Increase memory limit in compose: deploy.resources.limits.memory
            2. Set explicit JVM heap: add JAVA_OPTS_KC_HEAP_RATIO or -Xmx768m
            3. Reduce session timeout to free memory: realm settings → sessions
            4. Check for orphaned sessions: Admin Console → Sessions
```

### Incident 8: PostgreSQL connection pool exhausted

```
Symptom:    Guard logs show "FATAL: sorry, too many clients already" or
            "Connection pool exhausted".
Cause:      Too many concurrent requests, or connection leak.
Diagnosis:
            docker exec local-postgres psql -U postgres -c \
                "SELECT count(*) FROM pg_stat_activity WHERE datname='keycloak'"
            # Compare against KC_DB_POOL_MAX_SIZE (default 20)
Resolution:
            1. Increase pool size: KC_DB_POOL_MAX_SIZE=30 in .env
            2. Restart Guard
            3. Check for long-running queries:
               SELECT pid, state, query, query_start FROM pg_stat_activity
               WHERE datname='keycloak' AND state != 'idle';
```

---

## 6. Emergency Procedures

### 6.1 Complete Guard Failure

```bash
# 1. Check infrastructure
docker exec local-postgres pg_isready -U postgres
sudo systemctl status nginx cloudflared

# 2. Restart Guard
cd ~/platform
docker compose -f docker-compose.platform.yml restart flowero-guard

# 3. Wait and verify (up to 5 minutes for first boot with Liquibase)
for i in $(seq 1 30); do
    if curl -sf http://localhost:8001/health/ready > /dev/null 2>&1; then
        echo "Guard recovered"
        break
    fi
    sleep 10
done

# 4. If still down, check if DB is corrupted
docker logs flowero-guard 2>&1 | grep -i "liquibase\|fatal\|error"
```

### 6.2 Database Recovery

```bash
# Restore keycloak DB from backup
docker exec -i local-postgres psql -U postgres -c "DROP DATABASE IF EXISTS keycloak;"
docker exec -i local-postgres psql -U postgres -c "CREATE DATABASE keycloak OWNER keycloak;"
cat ~/backups/keycloak_YYYYMMDD.sql | docker exec -i local-postgres psql -U postgres -d keycloak

# Restart Guard
cd ~/platform
docker compose -f docker-compose.platform.yml restart flowero-guard
```

### 6.3 Forgot Admin Password

```bash
# 1. Stop Guard
docker compose -f docker-compose.platform.yml stop flowero-guard

# 2. Update .env
nano ~/platform/.env
# Change KEYCLOAK_ADMIN_PASSWORD=<new-password>

# 3. Start Guard
docker compose -f docker-compose.platform.yml up -d flowero-guard

# 4. Verify login at https://auth.panomete.com/admin
```

---

## 7. Useful Aliases

```bash
# Add to ~/.bashrc on homelab server

# Guard health
alias ghealth='curl -sf http://localhost:8001/health/ready && echo "Guard ✅" || echo "Guard ❌"'

# Guard logs
alias glogs='docker logs -f --tail 50 flowero-guard'

# Guard restart
alias grestart='cd ~/platform && docker compose -f docker-compose.platform.yml restart flowero-guard'

# Guard OIDC check
alias goidc='curl -sf https://auth.panomete.com/realms/panomete/.well-known/openid-configuration | jq .issuer'

# Guard JWKS check
alias gjwks='curl -sf https://auth.panomete.com/realms/panomete/protocol/openid-connect/certs | jq ".keys | length"'
```

---

## 8. Emergency Contacts

| Role | Name | Contact | When |
|------|------|---------|------|
| Server Owner | PO Persona | Tailscale / Chat | All issues |
| DevOps | DevOps Persona | Chat | Infrastructure, deployment |
| Developer | Dev Persona | Chat | Keycloak configuration, realm JSON |

---

## Related Documents

| Document | Relationship |
|----------|-------------|
| [[052_deployment_plan]] | How Guard was deployed |
| [[051_CICD_pipeline_configuration]] | Guard's CI/CD pipeline |
| `flowero_guard/02_design/022_API_specification` | Guard endpoints reference |
| `flowero_guard/02_design/023_database_schema_DDL` | Keycloak DB provisioning |
| `panomete_platform/05_devops/054_operations_manual_runbook` | Platform-wide runbook |

---

> **Template Standard:** Based on SWEBOK v4, SEBoK exportrealm JSON after every change — undocumented realm config is technical debt.
