---
document_type: Operations Manual / Runbook
version: "0.1"
status: Draft
author: "DevOps Persona"
created: "2026-07-23"
last_updated: "2026-07-23"
project_name: "Flowero Discover"
project_id: "PAN-DISCOVER-001"
parent_platform: "Panomete Platform"
classification: "Internal"
tags: [runbook, operations, troubleshooting, eureka, panomete]
standard_ref:
  - SWEBOK v4 — Operations
  - SEBoK v2 — Operations
  - ITIL v4 — Service Operation
---

# Operations Manual / Runbook — Flowero Discover

> **Service:** Flowero Discover (Spring Cloud Netflix Eureka)
> **Platform:** Panomete Platform
> **Version:** 0.1 | **Status:** Draft — Awaiting PO Review
> **Last Updated:** 2026-07-23

---

## 1. Purpose

> Step-by-step operational procedures for Flowero Discover. Covers health monitoring, service registration troubleshooting, self-preservation mode, and emergency procedures.

---

## 2. System Overview

| Component | Technology | Container | Port | Domain |
|-----------|-----------|-----------|:---:|--------|
| **Discover** | Spring Cloud Eureka | `flowero-discover` | `127.0.0.1:8999` / `127.0.0.1:3999` | `discovery.panomete.com` |
| **Database** | None (in-memory) | — | — | — |
| **Edge Proxy** | Nginx | _(host process)_ | `:80` | Routes `discovery.*` → `:3999` |
| **TLS** | Cloudflare Tunnel | _(host process)_ | — | `*.panomete.com` |

### Key URLs

| Resource | URL / Command |
|----------|---------------|
| Dashboard | `https://discovery.panomete.com` |
| REST API (internal) | `http://flowero-discover:8999/eureka/apps` |
| Health endpoint | `http://localhost:8999/actuator/health` |
| Container logs | `docker logs -f flowero-discover` |
| Server SSH | `ssh flowero@remote.panomete.com` |

---

## 3. Health Monitoring

### 3.1 Daily Health Check

```bash
# Health endpoint
curl -sf http://localhost:8999/actuator/health && echo "Discover ✅" || echo "Discover ❌"

# Registered services count
curl -sf -H 'Accept: application/json' http://localhost:8999/eureka/apps \
    | jq '.applications.application | length'

# List all registered services
curl -sf -H 'Accept: application/json' http://localhost:8999/eureka/apps \
    | jq '.applications.application[].name'

# Container status
docker ps --format '{{.Names}}\t{{.Status}}' | grep flowero-discover
```

### 3.2 Key Metrics to Watch

| Metric | Healthy | Warning | Critical | Where to Check |
|--------|:---:|:---:|:---:|----------------|
| Container status | Up | Restarting | Exited | `docker ps` |
| Health endpoint | `UP` | — | `DOWN` / timeout | `curl :8999/actuator/health` |
| Registered services | Expected count | Count dropping | 0 (all evicted) | Dashboard or REST API |
| JVM Heap | < 60% | 60-80% | > 80% | `docker stats flowero-discover` |
| Dashboard accessible | 200 | — | Non-200 | `curl discovery.panomete.com` |
| Heartbeat failures | 0/min | 1-5/min | > 10/min | Eureka server logs |

---

## 4. Routine Operations

### 4.1 Restart Discover

```bash
cd ~/platform
docker compose -f docker-compose.platform.yml restart flowero-discover

# Wait for startup
echo "Waiting for Discover..."
for i in $(seq 1 12); do
    if curl -sf http://localhost:8999/actuator/health > /dev/null 2>&1; then
        echo "Discover healthy after ${i}0 seconds"
        break
    fi
    sleep 10
done
```

> **⚠️ Warning:** Restarting Discover clears all in-memory registrations. All services (Guard, Gate, business services) must re-register. This happens automatically within 30 seconds, but there is a brief window where `lb://` resolution may fail at Gate. Expect brief 502 errors from Gate during this window.

### 4.2 Check Registered Services

```bash
# All registered applications (JSON)
curl -sf -H 'Accept: application/json' http://localhost:8999/eureka/apps | jq .

# Specific service
curl -sf -H 'Accept: application/json' http://localhost:8999/eureka/apps/FLOWERO-GUARD | jq .

# List instance IDs and status
curl -sf -H 'Accept: application/json' http://localhost:8999/eureka/apps \
    | jq '.applications.application[] | {name, instances: [.instance[] | {id: .instanceId, status: .status, port: .port["$"]}]}'
```

### 4.3 Force Deregister a Stale Service

> Use when a service container is gone but Eureka still shows it as UP.

```bash
# Deregister by app name + instance ID
curl -X DELETE http://localhost:8999/eureka/apps/CUTE-GUFO/cute-gufo:8005

# Verify removal
curl -sf -H 'Accept: application/json' http://localhost:8999/eureka/apps/CUTE-GUFO
# Should return 404 if no instances remain
```

### 4.4 Backup

> Discover has **no persistent data** — it's fully in-memory. No backup needed. Service registrations are ephemeral; services re-register on restart.

---

## 5. Troubleshooting Guide

### Incident 1: No services showing in Eureka dashboard

```
Symptom:    Dashboard loads but shows "No instances available".
            REST API returns empty applications list.
Cause:      Services not configured to register, or services not running.
Diagnosis:
            1. Check if Guard/Gate containers are running:
               docker ps | grep -E 'flowero-guard|flowero-gate'
            2. Check if services have Eureka client config:
               docker logs flowero-gate 2>&1 | grep -i "eureka\|discovery\|register"
            3. Check Eureka self-preservation status (dashboard header)
Resolution:
            1. If services down: restart them — they auto-register on startup
            2. If services up but not registered: check spring.cloud.eureka config
            3. Restart the service to force re-registration
```

### Incident 2: Service shown as UP but unreachable

```
Symptom:    Eureka shows service as UP, but Gate returns 502 Bad Gateway when
            routing to it.
Cause:      Stale registration — service container was replaced but Eureka
            hasn't evicted the old instance yet.
Diagnosis:
            1. Check if the registered IP/port matches the current container:
               docker inspect <container> --format '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}'
            2. Compare with Eureka registration:
               curl -sf -H 'Accept: application/json' http://localhost:8999/eureka/apps/<APP> | jq .
Resolution:
            1. Force deregister the stale instance (see 4.3)
            2. Or wait for heartbeat timeout (90s default) — Eureka auto-evicts
            3. Restart Gate to clear its local Eureka cache
```

### Incident 3: Self-preservation mode activated

```
Symptom:    Dashboard shows red warning: "EMERGENCY! EUREKA MAY BE
            INCORRECTLY CLAIMING INSTANCES ARE UP WHEN THEY'RE NOT."
            Stale services are NOT being evicted.
Cause:      Eureka enters self-preservation when it sees a sudden drop in
            heartbeats (e.g., network partition or mass restart). This is
            BY DESIGN (ADR-D004) — it prevents cascading failures.
Diagnosis:
            curl -sf http://localhost:8999/actuator/health
            # May show "UP" but with self-preservation flag
Resolution:
            1. This is expected behavior — do NOT panic
            2. Verify all expected services are actually running
            3. Heartbeats will resume and self-preservation will auto-disable
            4. If truly stale: force deregister specific instances (see 4.3)
            5. If persistent: temporarily disable, then re-enable:
               # This requires application.yml change + restart
```

### Incident 4: Dashboard not accessible via Nginx

```
Symptom:    https://discovery.panomete.com returns 502 or connection refused.
            Internal health endpoint works.
Cause:      Nginx config broken, or dashboard port (3999) not bound.
Diagnosis:
            curl -sf http://localhost:3999/ -o /dev/null -w '%{http_code}'
            # If 000 → port not bound or Discover not serving dashboard on 3999
            curl -sf -H 'Host: discovery.panomete.com' http://localhost -o /dev/null -w '%{http_code}'
            # If 502 → Nginx can't reach :3999
Resolution:
            1. Check Discover port binding: docker port flowero-discover
            2. If dashboard runs on 8999 not 3999: update Nginx proxy_pass to :8999
               (May happen if dual-port design ADR-D005 is revised to single-port)
            3. Check Nginx config: sudo nginx -t
            4. Reload: sudo systemctl reload nginx
```

### Incident 5: Discover container OOM killed

```
Symptom:    Container shows "Exited (137)" — killed by OOM killer.
Cause:      JVM heap too small, or too many registered services consuming memory.
Diagnosis:
            docker inspect flowero-discover --format '{{.State.OOMKilled}}'
            docker stats flowero-discover
Resolution:
            1. Increase memory limit in compose: deploy.resources.limits.memory
            2. Increase JVM heap: -Xmx256m (from -Xmx192m)
            3. Restart: docker compose restart flowero-discover
```

### Incident 6: Eureka REST API returns XML instead of JSON

```
Symptom:    curl returns XML. jq fails to parse.
Cause:      Eureka defaults to XML. Must send Accept: application/json header.
Diagnosis:
            curl http://localhost:8999/eureka/apps | head -5
            # If starts with "<?xml" → missing Accept header
Resolution:
            Always use: curl -H 'Accept: application/json' http://localhost:8999/eureka/apps
```

---

## 6. Emergency Procedures

### 6.1 Complete Discover Failure

```bash
# 1. Restart Discover
cd ~/platform
docker compose -f docker-compose.platform.yml restart flowero-discover

# 2. Wait for startup
for i in $(seq 1 12); do
    if curl -sf http://localhost:8999/actuator/health > /dev/null 2>&1; then
        echo "Discover recovered"
        break
    fi
    sleep 10
done

# 3. Verify services re-register
sleep 30
curl -sf -H 'Accept: application/json' http://localhost:8999/eureka/apps | jq '.applications.application[].name'
```

### 6.2 Gate Cannot Resolve Routes (Eureka Down Cascade)

> If Discover is down, Gate uses its cached registry. But if Gate also restarts while Discover is down, Gate has no registry and cannot route.

```bash
# 1. Fix Discover first (see 6.1)
# 2. Restart Gate to force fresh registry fetch
docker compose -f docker-compose.platform.yml restart flowero-gate

# 3. Wait for Gate to fetch registry
sleep 15
curl -sf http://localhost:8000/actuator/health | jq '.components.discoveryComposite'
```

---

## 7. Useful Aliases

```bash
# Add to ~/.bashrc on homelab server

# Discover health
alias dhealth='curl -sf http://localhost:8999/actuator/health && echo "Discover ✅" || echo "Discover ❌"'

# Discover logs
alias dlogs='docker logs -f --tail 50 flowero-discover'

# Discover restart
alias drestart='cd ~/platform && docker compose -f docker-compose.platform.yml restart flowero-discover'

# List registered services
alias dservices='curl -sf -H "Accept: application/json" http://localhost:8999/eureka/apps | jq ".applications.application[].name"'

# Eureka dashboard status
alias dstatus='curl -sf -H "Accept: application/json" http://localhost:8999/eureka/apps | jq ".applications.application | length"'
```

---

## 8. Emergency Contacts

| Role | Name | Contact | When |
|------|------|---------|------|
| Server Owner | PO Persona | Tailscale / Chat | All issues |
| DevOps | DevOps Persona | Chat | Infrastructure, deployment |
| Developer | Dev Persona | Chat | Application bugs, Eureka config |

---

## Related Documents

| Document | Relationship |
|----------|-------------|
| [[052_deployment_plan]] | How Discover was deployed |
| [[051_CICD_pipeline_configuration]] | Discover's pipeline |
| `flowero_discover/02_design/022_API_specification` | Eureka REST API reference |
| `flowero_discover/02_design/021_architecture_decision_records` | ADRs (ADR-D004 self-preservation, ADR-D005 dual-port) |
| `panomete_platform/05_devops/054_operations_manual_runbook` | Platform-wide runbook |

---

> **Template Standard:** Based on SWEBOK v4, SEBoK v2, ITIL v4
> **Usage:** Self-preservation mode is NOT a bug — it's a feature. Understand it before disabling.
