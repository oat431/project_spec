---
document_type: Operations Manual / Runbook
version: "0.1"
status: Draft
author: "DevOps Persona"
created: "2026-07-23"
last_updated: "2026-07-23"
project_name: "Panomete Platform"
project_id: "PAN-PLAT-001"
classification: "Internal"
tags: [runbook, operations, procedures, troubleshooting, panomete]
standard_ref:
  - SWEBOK v4 — Operations
  - SEBoK v2 — Operations
  - ITIL v4 — Service Operation
---

# Operations Manual / Runbook — Panomete Platform

> **Project:** Panomete Platform
> **Version:** 0.1 | **Status:** Draft — Awaiting PO Review
> **Last Updated:** 2026-07-23
> **Scope:** Phase 1 — Guard (Keycloak), Discover (Eureka), Gate (Spring Cloud Gateway)

---

## 1. Purpose

> Step-by-step operational procedures for managing the Panomete Platform foundation services. If it's not in the runbook, it doesn't exist as a procedure. Update after every incident.

---

## 2. System Overview

| Component | Technology | Container Name | Port | Domain |
|-----------|-----------|----------------|:---:|--------|
| **Flowero Guard** | Keycloak | `flowero-guard` | 8001 | `auth.panomete.com` |
| **Flowero Discover** | Spring Cloud Eureka | `flowero-discover` | 8999 / 3999 | `discovery.panomete.com` |
| **Flowero Gate** | Spring Cloud Gateway | `flowero-gate` | 8000 | `api.panomete.com` |
| PostgreSQL | PostgreSQL 18 | `local-postgres` | 5432 | — (internal) |
| Valkey | Valkey 9 | `local-valkey` | 6379 | — (internal) |
| Nginx | Reverse Proxy | _(host process)_ | 80 | — (edge) |
| Cloudflare | Tunnel | _(host process)_ | — | `*.panomete.com` |

### Access

| Resource | Method |
|----------|--------|
| Server SSH | `ssh flowero@remote.panomete.com` (key auth only, Tailscale) |
| Guard Admin Console | `https://auth.panomete.com/admin` |
| Discover Dashboard | `https://discovery.panomete.com` |
| PostgreSQL (CLI) | `docker exec -it local-postgres psql -U postgres` |
| Valkey (CLI) | `docker exec -it local-valkey valkey-cli -a <password>` |
| Portainer | `https://container.panomete.com` |
| Docker logs | `docker logs -f <container-name>` |

---

## 3. Service Startup Order

> Services must start in this order. The deployment plan uses `depends_on` + sleep, but manual restarts must follow the same sequence.

| Order | Service | Depends On | Health Check |
|:-----:|---------|-----------|-------------|
| 1 | PostgreSQL | _(already running)_ | `docker exec local-postgres pg_isready -U postgres` |
| 2 | Valkey | _(already running)_ | `docker exec local-valkey valkey-cli -a *** ping` → `PONG` |
| 3 | Flowero Guard | PostgreSQL healthy | `curl -sf http://localhost:8001/health/ready` |
| 4 | Flowero Discover | — | `curl -sf http://localhost:8999/actuator/health` |
| 5 | Flowero Gate | Guard + Discover + Valkey healthy | `curl -sf http://localhost:8000/actuator/health` |

---

## 4. Routine Operations

### 4.1 Daily Health Check

```bash
# SSH into server
ssh flowero@remote.panomete.com

# Quick health check script
echo "=== Guard ===" && curl -sf http://localhost:8001/health/ready && echo " ✅" || echo " ❌"
echo "=== Discover ===" && curl -sf http://localhost:8999/actuator/health && echo " ✅" || echo " ❌"
echo "=== Gate ===" && curl -sf http://localhost:8000/actuator/health && echo " ✅" || echo " ❌"
echo "=== PostgreSQL ===" && docker exec local-postgres pg_isready -U postgres && echo " || echo " ❌"
echo "=== Valkey ===" && docker exec local-valkey valkey-cli -a *** ping && echo " || echo " ❌"
echo "=== Disk ===" && df -h / | awk 'NR==2 {print $5}'
echo "=== Memory ===" && free -h | awk '/Mem/ {print $3 "/" $2}'
```

### 4.2 Weekly Maintenance

| # | Task | Command | Duration |
|---|------|---------|:---:|
| 1 | Backup all PostgreSQL databases | `docker exec local-postgres pg_dumpall -U postgres > ~/backups/pg_all_$(date +%Y%m%d).sql` | 5 min |
| 2 | Export Keycloak realm | Admin Console → Realm Settings → Export | 2 min |
| 3 | Check container disk usage | `docker system df` | 1 min |
| 4 | Review error logs | `docker logs --since 7d flowero-gate 2>&1 \| grep -i error` | 10 min |
| 5 | Docker image cleanup | `docker image prune -f` | 1 min |

### 4.3 Backup

| Data | Method | Frequency | Location |
|------|--------|-----------|----------|
| PostgreSQL (all DBs) | `pg_dumpall` | Daily | `~/backups/` → rclone → OneDrive |
| Keycloak realm | JSON export from Admin Console | On change | Version-controlled in repo |
| Docker volumes | `docker run --rm -v <vol>:/data ... tar` | Weekly | `~/backups/` |

---

## 5. Restart Procedures

### 5.1 Restart Single Service

```bash
cd ~/platform

# Restart Guard
docker compose -f docker-compose.platform.yml restart flowero-guard
# Wait for health
sleep 15 && curl -sf http://localhost:8001/health/ready

# Restart Discover
docker compose -f docker-compose.platform.yml restart flowero-discover
sleep 10 && curl -sf http://localhost:8999/actuator/health

# Restart Gate
docker compose -f docker-compose.platform.yml restart flowero-gate
sleep 10 && curl -sf http://localhost:8000/actuator/health
```

### 5.2 Full Platform Restart (All Services)

```bash
cd ~/platform

# Stop in reverse order: Gate → Discover → Guard
docker compose -f docker-compose.platform.yml stop flowero-gate
docker compose -f docker-compose.platform.yml stop flowero-discover
docker compose -f docker-compose.platform.yml stop flowero-guard

# Start in dependency order: Guard → Discover → Gate
docker compose -f docker-compose.platform.yml start flowero-guard
sleep 15
docker compose -f docker-compose.platform.yml start flowero-discover
sleep 10
docker compose -f docker-compose.platform.yml start flowero-gate
sleep 10

# Verify all
curl -sf http://localhost:8001/health/ready && echo "Guard ✅"
curl -sf http://localhost:8999/actuator/health && echo "Discover ✅"
curl -sf http://localhost:8000/actuator/health && echo "Gate ✅"
```

---

## 6. Troubleshooting Guide

### 6.1 General Diagnostic Commands

```bash
# See all container statuses
docker ps -a --format 'table {{.Names}}\t{{.Status}}\t{{.Ports}}' | grep flowero

# Check logs (last 100 lines)
docker logs --tail 100 flowero-guard
docker logs --tail 100 flowero-discover
docker logs --tail 100 flowero-gate

# Check resource usage
docker stats --no-stream flowero-guard flowero-discover flowero-gate

# Check network connectivity
docker exec flowero-gate ping -c1 local-valkey
docker exec flowero-gate ping -c1 local-postgres
docker exec flowero-guard ping -c1 local-postgres
```

### 6.2 Common Incidents

| # | Symptom | Possible Cause | Investigation | Resolution |
|---|---------|---------------|--------------|------------|
| 1 | **Gate returns 502 Bad Gateway** | Backend business service is down or unregistered in Eureka | Check Eureka dashboard for registered services. Check business service container. | Restart business service. Wait 30s for Eureka re-registration. |
| 2 | **Gate returns 401 for all requests** | Keycloak JWKS unreachable or Gate's JWKS cache is stale | `curl -sf https://auth.panomete.com/realms/panomete/protocol/openid-connect/certs` | Restart Gate (it re-fetches JWKS on startup). Check if Guard is healthy. |
| 3 | **Gate returns 429 Rate Limit** | Valkey rate limiter triggered or Valkey down | Check Gate health actuator for `redisReactiveHealthIndicator`. `docker exec local-valkey valkey-cli -a *** ping` | If Valkey down: restart Valkey then restart Gate. If legitimate: increase rate limit in `application.yml`. |
| 4 | **Guard (Keycloak) won't start** | PostgreSQL `keycloak` DB or role missing, or wrong password | `docker logs flowero-guard 2>&1 \| grep -i "database\|jdbc\|auth"` | Verify DB exists: `docker exec local-postgres psql -U postgres -c "\l keycloak"`. Verify role: `docker exec local-postgres psql -U postgres -c "\du keycloak"`. Check `.env` password. |
| 5 | **Guard (Keycloak) stuck on Liquibase** | Migration conflict or corrupted schema | `docker logs flowero-guard 2>&1 \| grep -i "liquibase\|migration"` | Last resort: `DROP DATABASE keycloak; CREATE DATABASE keycloak OWNER keycloak;` — Keycloak re-runs migrations. **Realm data lost** — re-import from JSON. |
| 6 | **Discover dashboard shows no services** | Services not registering, or Eureka self-preservation mode | Check Eureka dashboard. Check if Gate and Guard containers are running. | Restart the unregistered service. Check `spring.cloud.eureka` config in the service. |
| 7 | **Nginx returns 502 for `auth.panomete.com`** | Guard container down or port 8001 not bound | `docker ps \| grep flowero-guard`. `ss -tlnp \| grep 8001` | Restart Guard. Verify `127.0.0.1:8001:8080` port binding in compose. |
| 8 | **All services unreachable externally** | Cloudflare tunnel down or Nginx down | `sudo systemctl status cloudflared`. `sudo systemctl status nginx`. `curl -H 'Host: auth.panomete.com' http://localhost` | Restart the failed service: `sudo systemctl restart cloudflared` or `sudo systemctl reload nginx`. |
| 9 | **Existing services (AdGuard, Portainer) down** | Resource exhaustion or Docker daemon issue | `free -h`. `docker system df`. `docker ps -a` | Restart Docker: `sudo systemctl restart docker`. Check disk space. |
| 10 | **Slow API responses** | JVM heap too small, or PostgreSQL connection pool exhausted | `docker stats flowero-gate`. Check Gate health for DB pool. | Increase JVM heap in compose `-Xmx`. Increase Keycloak DB pool (`KC_DB_POOL_MAX_SIZE`). |

---

## 7. Keycloak-Specific Operations

### 7.1 Access Admin Console

1. Navigate to `https://auth.panomete.com/admin`
2. Login with admin credentials from `.env` (`KEYCLOAK_ADMIN` / `KEYCLOAK_ADMIN_PASSWORD`)
3. Select the `panomete` realm

### 7.2 Export Realm Configuration

> Do this after any realm change (new client, new role, user changes).

1. Admin Console → `panomete` realm → Realm Settings → Action → Partial Export
2. Select "Include groups and roles" + "Include clients"
3. Download JSON
4. Save to repo: `flowero-guard/panomete-realm.json`
5. Commit with message: `chore(guard): export realm config YYYY-MM-DD`

### 7.3 Register a New OAuth2 Client

1. Admin Console → `panomete` realm → Clients → Create client
2. Client ID: `<service-name>` (e.g., `cute-gufo`)
3. Client type: Confidential
4. Standard flow: Enabled (for browser login)
5. Service accounts: Enabled (for S2S calls)
6. Save → Credentials tab → Copy secret → Add to `.env`

---

## 8. Emergency Procedures

### 8.1 Complete Platform Outage

```bash
# 1. Check infrastructure first
sudo systemctl status nginx cloudflared
docker ps | grep -E 'local-postgres|local-valkey'

# 2. If infrastructure OK, restart platform
cd ~/platform
docker compose -f docker-compose.platform.yml down
docker compose -f docker-compose.platform.yml up -d

# 3. Wait and verify
sleep 30
curl -sf http://localhost:8001/health/ready
curl -sf http://localhost:8999/actuator/health
curl -sf http://localhost:8000/actuator/health
```

### 8.2 Database Recovery

```bash
# Restore PostgreSQL from backup
cat ~/backups/pg_all_YYYYMMDD.sql | docker exec -i local-postgres psql -U postgres

# Verify keycloak database
docker exec local-postgres psql -U postgres -c "\l keycloak"
docker exec local-postgres psql -U postgres -d keycloak -c "\dt" | head -20
```

### 8.3 Forgot Keycloak Admin Password

```bash
# 1. Stop Guard
docker compose -f docker-compose.platform.yml stop flowero-guard

# 2. Update .env with new password
nano ~/platform/.env
# Change KEYCLOAK_ADMIN_PASSWORD

# 3. Start Guard
docker compose -f docker-compose.platform.yml up -d flowero-guard

# 4. Verify login at https://auth.panomete.com/admin
```

---

## 9. Emergency Contacts

| Role | Name | Contact | When |
|------|------|---------|------|
| Server Owner | PO Persona | Tailscale / Chat | All issues |
| DevOps | DevOps Persona | Chat | Deployment, infrastructure |
| Developer | Dev Persona | Chat | Application bugs, code |

---

## 10. Useful Aliases (Add to `~/.bashrc`)

```bash
# Platform health
alias phealth='echo "Guard:" && curl -sf http://localhost:8001/health/ready && echo " ✅" || echo " ❌"; echo "Discover:" && curl -sf http://localhost:8999/actuator/health && echo " ✅" || echo " ❌"; echo "Gate:" && curl -sf http://localhost:8000/actuator/health && echo " ✅" || echo " ❌"'

# Platform logs
alias plogs='cd ~/platform && docker compose -f docker-compose.platform.yml logs -f --tail=50'

# Platform restart
alias prestart='cd ~/platform && docker compose -f docker-compose.platform.yml restart'

# Platform status
alias pstatus='docker ps --format "table {{.Names}}\t{{.Status}}" | grep -E "flowero|local-postgres|local-valkey"'
```

---

## Related Documents

| Document | Relationship |
|----------|-------------|
| [[052_deployment_plan]] | How services were deployed |
| [[051_CICD_pipeline_configuration]] | Pipeline that ships changes |
| `panomete_platform/02_design/025_software_architecture_document` | Architecture reference |
| [[MM04_devops-infra-audit_20260723]] | Infrastructure audit findings |
| Home Lab notes | `F:\obsidian_note\oralita_md\home-lab-installation\` |

---

> **Template Standard:** Based on SWEBOK v4, SEBoK v2, ITIL v4
> **Usage:** The runbook is the *operations bible*. If it's not in the runbook, it doesn't exist as a procedure. Keep it updated after every incident.
