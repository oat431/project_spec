---
document_type: Operations Manual / Runbook
version: "0.1"
status: Draft
author: "DevOps Persona"
created: "2026-07-23"
last_updated: "2026-07-23"
project_name: "Flowero Gate"
project_id: "PAN-GATE-001"
parent_platform: "Panomete Platform"
classification: "Internal"
tags: [runbook, operations, troubleshooting, spring-cloud-gateway, panomete]
standard_ref:
  - SWEBOK v4 — Operations
  - SEBoK v2 — Operations
  - ITIL v4 — Service Operation
---

# Operations Manual / Runbook — Flowero Gate

> **Service:** Flowero Gate (Spring Cloud Gateway)
> **Platform:** Panomete Platform
> **Version:** 0.1 | **Status:** Draft — Awaiting PO Review
> **Last Updated:** 2026-07-23

---

## 1. Purpose

> Step-by-step operational procedures for Flowero Gate. Gate is the API gateway — when it fails, all business API traffic fails. Covers health monitoring, JWT validation troubleshooting, rate limiting, route debugging, and emergency procedures.

---

## 2. System Overview

| Component | Technology | Container | Port | Domain |
|-----------|-----------|-----------|:---:|--------|
| **Gate** | Spring Cloud Gateway (WebFlux/Netty) | `flowero-gate` | `127.0.0.1:8000` | `api.panomete.com` |
| **JWKS source** | Guard (Keycloak) | `flowero-guard` | `127.0.0.1:8001` | `auth.panomete.com/.../certs` |
| **Registry** | Discover (Eureka) | `flowero-discover` | `127.0.0.1:8999` | — (internal) |
| **Rate limiting** | Valkey 9 | `local-valkey` | `127.0.0.1:6379` | — (internal) |
| **Edge Proxy** | Nginx | _(host process)_ | `:80` | Routes `api.*` → `:8000` |
| **TLS** | Cloudflare Tunnel | _(host process)_ | — | `*.panomete.com` |

### Key URLs

| Resource | URL / Command |
|----------|---------------|
| API base | `https://api.panomete.com` |
| Health endpoint | `http://localhost:8000/actuator/health` |
| Container logs | `docker logs -f flowero-gate` |
| Server SSH | `ssh flowero@remote.panomete.com` |

---

## 3. Health Monitoring

### 3.1 Daily Health Check

```bash
# Full health check (includes all dependency indicators)
curl -sf http://localhost:8000/actuator/health | jq .

# Quick check
curl -sf http://localhost:8000/actuator/health > /dev/null && echo "Gate ✅" || echo "Gate ❌"

# Check specific components
curl -sf http://localhost:8000/actuator/health | jq '.components | keys'
# Expected: discoveryComposite, redisReactiveHealthIndicator, etc.

# Verify 401 rejection (no token = 401, proving JWT validation is active)
curl -sf -o /dev/null -w '%{http_code}' https://api.panomete.com/api/blog/posts
# Expected: 401

# Container status + resource usage
docker stats --no-stream flowero-gate
```

### 3.2 Key Metrics to Watch

| Metric | Healthy | Warning | Critical | Where to Check |
|--------|:---:|:---:|:---:|----------------|
| Container status | Up | Restarting | Exited | `docker ps` |
| Overall health | `UP` | — | `DOWN` | `curl :8000/actuator/health` |
| Valkey connection | `UP` | — | `DOWN` | Health `redisReactiveHealthIndicator` |
| Eureka connection | `UP` | — | `DOWN` | Health `discoveryComposite` |
| 401 rejection rate | Works | — | Bypassed (200 without token) | Manual curl test |
| JVM Heap | < 60% | 60-80% | > 80% | `docker stats` |
| Response time p95 | < 200ms | 200-500ms | > 500ms | Manual curl timing |
| 502 error rate | 0/min | 1-5/min | > 10/min | Gate logs |
| 429 rate limit hits | 0/min | Occasional | Constant | Gate logs |

---

## 4. Routine Operations

### 4.1 Restart Gate

```bash
cd ~/platform
docker compose -f docker-compose.platform.yml restart flowero-gate

# Wait for startup (Gate fetches JWKS + Eureka registry on boot)
echo "Waiting for Gate..."
for i in $(seq 1 12); do
    if curl -sf http://localhost:8000/actuator/health > /dev/null 2>&1; then
        echo "Gate healthy after ${i}0 seconds"
        break
    fi
    sleep 10
done

# Verify JWKS cache is loaded
curl -sf -o /dev/null -w '%{http_code}' https://api.panomete.com/api/blog/posts
# Should be 401 (JWT validation active), not 500 (JWKS not loaded)
```

> **Note:** Restarting Gate briefly drops all API traffic (5-10 seconds). Existing services (AdGuard, Portainer) are unaffected. Gate is stateless — no data loss.

### 4.2 Add a New Route

> When onboarding a new business service.

1. **Edit Gate's `application.yml`:**
```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: ledger                         # New route
          uri: lb://big-schwein
          predicates: [Path=/api/ledger/**]
          filters:
            - StripPrefix=1
            - name: RequestRateLimiter
              args:
                redis-rate-limiter.replenishRate: 100
                redis-rate-limiter.burstCapacity: 200
```

2. **Ensure the service is registered in Eureka** (auto-registration via Spring Cloud)
3. **Register an OAuth2 client in Keycloak** for the new service
4. **Restart Gate:** `docker compose restart flowero-gate`
5. **Verify:** `curl -sf -o /dev/null -w '%{http_code}' https://api.panomete.com/api/ledger/test` → should be 401

### 4.3 Check Rate Limiting

```bash
# Check Valkey for rate limit keys
docker exec local-valkey valkey-cli -a "$VALKEY_PASSWORD" keys "request_rate_limiter*"

# Check a specific key's TTL (how long until rate limit resets)
docker exec local-valkey valkey-cli -a "$VALKEY_PASSWORD" ttl "request_rate_limiter.{192.168.1.100}.token"

# Flush rate limits for testing (clears ALL rate limit data)
docker exec local-valkey valkey-cli -a "$VALKEY_PASSWORD" del $(docker exec local-valkey valkey-cli -a "$VALKEY_PASSWORD" keys "request_rate_limiter*" | tr '\n' ' ')
```

### 4.4 Inspect Eureka Registry from Gate's Perspective

```bash
# Check what services Gate knows about
curl -sf http://localhost:8000/actuator/health | jq '.components.discoveryComposite'

# Detailed registry info
curl -sf -H 'Accept: application/json' http://localhost:8999/eureka/apps | jq .
```

### 4.5 Backup

> Gate is **stateless** — no backup needed. All configuration lives in:
> - `application.yml` (version-controlled in Git)
> - `~/platform/.env` (Valkey password — should be backed up with the `.env` file)
> - `~/platform/docker-compose.platform.yml` (version-controlled)

---

## 5. Troubleshooting Guide

### Incident 1: All API requests return 502 Bad Gateway

```
Symptom:    All /api/** requests return 502. Health endpoint is UP.
Cause:      Backend business service is down or not registered in Eureka.
Diagnosis:
            1. Check which services are registered:
               curl -sf -H 'Accept: application/json' http://localhost:8999/eureka/apps | jq '.applications.application[].name'
            2. Check if the target business service container is running:
               docker ps | grep -E 'cute-gufo|fluffy-mouton|tiny-mchwa'
            3. Check Gate logs for which backend it tried:
               docker logs flowero-gate 2>&1 | tail -20
Resolution:
            1. Restart the down backend service
            2. Wait 30s for Eureka re-registration
            3. If Eureka registry is stale, restart Gate to force fresh fetch
```

### Incident 2: All API requests return 401 (even with valid token)

```
Symptom:    Even valid JWTs are rejected with 401.
Cause:      JWKS cache is stale, Guard is down, or key rotation happened.
Diagnosis:
            1. Check Guard health:
               curl -sf http://localhost:8001/health/ready
            2. Check JWKS endpoint directly:
               curl -sf https://auth.panomete.com/realms/panomete/protocol/openid-connect/certs | jq '.keys | length'
            3. Check Gate logs:
               docker logs flowero-gate 2>&1 | grep -i "jwk\|jwt\|invalid\|expired"
Resolution:
            1. If Guard is down: restart Guard, wait for health, then restart Gate
            2. If JWKS changed (key rotation): restart Gate to re-fetch JWKS
            3. Verify issuer URI in Gate config matches Guard's actual issuer:
               curl -sf https://auth.panomete.com/realms/panomete/.well-known/openid-configuration | jq .issuer
               # Must match: https://auth.panomete.com/realms/panomete
```

### Incident 3: API requests return 429 Too Many Requests

```
Symptom:    Legitimate requests are rate-limited (429).
Cause:      Rate limit too aggressive, or a single IP is hitting the limit.
Diagnosis:
            1. Check Valkey for rate limit keys:
               docker exec local-valkey valkey-cli -a "$VALKEY_PASSWORD" keys "request_rate_limiter*"
            2. Check the Retry-After header:
               curl -sf -D- https://api.panomete.com/api/blog/posts 2>&1 | grep -i retry
            3. Check Gate logs for 429 entries:
               docker logs flowero-gate 2>&1 | grep 429
Resolution:
            1. If limit too low: increase in application.yml:
               redis-rate-limiter.replenishRate: 200  # from 100
               redis-rate-limiter.burstCapacity: 400  # from 200
            2. Restart Gate after config change
            3. For testing: flush rate limit keys in Valkey (see 4.3)
```

### Incident 4: Valkey connection lost — rate limiting fails

```
Symptom:    Gate health shows redisReactiveHealthIndicator: DOWN.
            All requests may return 500 (rate limiter can't function).
Cause:      Valkey container down, wrong password, or network issue.
Diagnosis:
            docker exec local-valkey valkey-cli -a "$VALKEY_PASSWORD" ping
            # Should return PONG
            docker logs flowero-gate 2>&1 | grep -i "redis\|valkey\|connection"
Resolution:
            1. If Valkey down: docker restart local-valkey
            2. If wrong password: check ~/platform/.env VALKEY_PASSWORD
            3. Restart Gate after fixing: docker compose restart flowero-gate
            4. Verify: curl http://localhost:8000/actuator/health | jq '.components.redisReactiveHealthIndicator'
```

### Incident 5: Gate returns 503 Service Unavailable

```
Symptom:    Requests return 503 with {"error":"Service Unavailable","service":"..."}.
Cause:      Eureka has evicted the target service (heartbeat timeout).
Diagnosis:
            1. Check Eureka for the service:
               curl -sf -H 'Accept: application/json' http://localhost:8999/eureka/apps | jq '.applications.application[].name'
            2. Check if target service is running:
               docker ps | grep <service-name>
            3. Check Eureka self-preservation mode (see Discover runbook Incident 3)
Resolution:
            1. Restart the evicted service — it auto-registers
            2. Wait 30s for registration
            3. If self-preservation is showing stale entries: force deregister (Discover runbook 4.3)
```

### Incident 6: Gate returns 500 Internal Server Error

```
Symptom:    500 errors from Gate.
Cause:      Unhandled exception — usually a misconfiguration or dependency failure.
Diagnosis:
            docker logs flowero-gate 2>&1 | grep -iE "error|exception" | tail -20
Resolution:
            1. Read the stack trace in logs
            2. Common causes:
               - JWKS fetch failed → see Incident 2
               - Valkey connection failed → see Incident 4
               - Route config error → check application.yml
            3. If can't diagnose: restart Gate
               docker compose restart flowero-gate
```

### Incident 7: Nginx returns 502 for api.panomete.com

```
Symptom:    https://api.panomete.com returns 502.
            Internal health endpoint may work.
Cause:      Gate container down, port 8000 not bound, or Nginx misconfigured.
Diagnosis:
            docker ps | grep flowero-gate
            ss -tlnp | grep 8000
            curl -sf http://localhost:8000/actuator/health
Resolution:
            1. If container down: docker compose restart flowero-gate
            2. If port not bound: check compose ports "127.0.0.1:8000:8000"
            3. If Nginx: check /etc/nginx/sites-enabled/api.panomete.com
            4. Test: sudo nginx -t && sudo systemctl reload nginx
```

### Incident 8: Slow API responses

```
Symptom:    Response times > 500ms.
Cause:      JVM heap pressure, backend service slow, or Eureka resolution overhead.
Diagnosis:
            1. Check JVM: docker stats flowero-gate
            2. Check if backend is slow:
               docker logs flowero-gate 2>&1 | grep latency | tail -10
            3. Check Eureka resolution:
               curl -sf -H 'Accept: application/json' http://localhost:8999/eureka/apps | jq .
Resolution:
            1. Increase JVM heap: -Xmx512m (from -Xmx384m)
            2. If backend slow: investigate backend service, not Gate
            3. Eureka resolution should be cached — if not, check Spring Cloud LoadBalancer config
```

---

## 6. Emergency Procedures

### 6.1 Complete Gate Failure

```bash
# 1. Check dependencies first
curl -sf http://localhost:8001/health/ready > /dev/null && echo "Guard ✅" || echo "Guard ❌"
curl -sf http://localhost:8999/actuator/health > /dev/null && echo "Discover ✅" || echo "Discover ❌"
docker exec local-valkey valkey-cli -a "$VALKEY_PASSWORD" ping > /dev/null && echo "Valkey ✅" || echo "Valkey ❌"

# 2. If all deps OK, restart Gate
cd ~/platform
docker compose -f docker-compose.platform.yml restart flowero-gate

# 3. Wait and verify
for i in $(seq 1 12); do
    if curl -sf http://localhost:8000/actuator/health > /dev/null 2>&1; then
        echo "Gate recovered"
        break
    fi
    sleep 10
done

# 4. Verify auth is working
curl -sf -o /dev/null -w '%{http_code}' https://api.panomete.com/api/blog/posts
# Should be 401 (proving JWT validation is active)
```

### 6.2 Gate Down + Can't Restart (Dependency Failure)

```bash
# If Gate won't start because a dependency is down:
# 1. Fix Guard first (if JWKS is the problem)
# 2. Fix Discover (if Eureka is the problem)
# 3. Fix Valkey (if rate limiting is the problem)
# 4. Then restart Gate

# Order of recovery: Guard → Discover → Valkey → Gate
```

### 6.3 Bypass Gate Temporarily (Emergency Only)

> **⚠️ WARNING:** This bypasses JWT validation and rate limiting. Only for debugging.

```bash
# Temporarily route api.panomete.com directly to a backend via Nginx
# This is a LAST RESORT and removes all security controls

# Edit /etc/nginx/sites-available/api.panomete.com
# Change proxy_pass to the backend directly (e.g., http://127.0.0.1:8005 for blog)
# Reload Nginx
sudo nginx -t && sudo systemctl reload nginx
```

---

## 7. Useful Aliases

```bash
# Add to ~/.bashrc on homelab server

# Gate health
alias gwhealth='curl -sf http://localhost:8000/actuator/health && echo "Gate ✅" || echo "Gate ❌"'

# Gate detailed health (all components)
alias gwdetail='curl -sf http://localhost:8000/actuator/health | jq .'

# Gate logs
alias gwlogs='docker logs -f --tail 50 flowero-gate'

# Gate restart
alias gwrestart='cd ~/platform && docker compose -f docker-compose.platform.yml restart flowero-gate'

# Gate auth check (should return 401)
alias gwauth='curl -sf -o /dev/null -w "%{http_code}" https://api.panomete.com/api/blog/posts && echo " (expect 401)"'

# Gate resource stats
alias gwstats='docker stats --no-stream flowero-gate'
```

---

## 8. Emergency Contacts

| Role | Name | Contact | When |
|------|------|---------|------|
| Server Owner | PO Persona | Tailscale / Chat | All issues |
| DevOps | DevOps Persona | Chat | Infrastructure, deployment |
| Developer | Dev Persona | Chat | Application bugs, route config, JWT validation |

---

## Related Documents

| Document | Relationship |
|----------|-------------|
| [[052_deployment_plan]] | How Gate was deployed |
| [[051_CICD_pipeline_configuration]] | Gate's pipeline |
| `flowero_gate/02_design/022_API_specification` | Route table, auth behavior, error codes |
| `flowero_gate/02_design/021_architecture_decision_records` | Gate ADRs |
| `flowero_guard/05_devops/054_operations_manual_runbook` | Guard runbook (JWKS source) |
| `flowero_discover/05_devops/054_operations_manual_runbook` | Discover runbook (Eureka source) |
| `panomete_platform/05_devops/054_operations_manual_runbook` | Platform-wide runbook |

---

> **Template Standard:** Based on SWEBOK v4, SEBoK v2, ITIL v4
> **Usage:** Gate is the API perimeter. If JWT validation is bypassed, all business services are exposed. Never run Gate without auth for extended periods.
