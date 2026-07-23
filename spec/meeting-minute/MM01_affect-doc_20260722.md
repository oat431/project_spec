---
document_type: Meeting Minutes
version: "0.1"
status: Draft
author: "Dev / SA Persona"
created: "2026-07-22"
last_updated: "2026-07-22"
project_name: "Panomete Platform"
meeting_type: "Design Review — Grill Session"
participants: ["PO Persona (represented by User)", "Dev / SA Persona"]
classification: "Internal"
tags: [meeting-minutes, design-review, affected-documents, po-handoff]
---

# Meeting Minutes — Design Review (Grill Session)

> **Date:** 2026-07-22
> **Type:** Architecture / Design Review
> **Participants:** PO (represented), Dev / SA
> **Purpose:** Validate design decisions against requirements; identify changes needed in requirement documents

---

## Decisions Made (What Changed)

| # | Decision | Previous Assumption | New Direction |
|---|----------|-------------------|---------------|
| D1 | Flowero Guard IS Keycloak | Guard was described as "Spring Boot OAuth2 Resource Server" in some places | Guard = Keycloak container directly. No wrapper service. Gate does JWT validation locally. |
| D2 | Nginx is the edge proxy | Gate was assumed to be on :80/:443 facing the internet | Nginx (already running) is the edge. Gate is internal on port 8000. |
| D3 | Domain scheme | `panomete.local` with path-based routing | `*.panomete.com` with subdomains through Nginx |
| D4 | Foundation service subdomains | Foundation services routed through Gate by path | Foundation services get their own subdomains: `auth.panomete.com`, `discovery.panomete.com` |
| D5 | Business service routing | All services routed through Gate | Business services use path routing: `api.panomete.com/api/{service}/**` |
| D6 | Port assignments | Gate: 80/443, Guard: 8080, Discover: 8761 | Gate: 8000, Guard: 8001, Discover: 8999 (BE) / 3999 (FE dashboard) |
| D7 | Shared infrastructure | Dedicated guard-db container, PostgreSQL 15, in-memory rate limiting | Use existing shared PostgreSQL 18 + Valkey 9. No dedicated DB container. |
| D8 | TLS termination | Self-signed cert at Gate | Cloudflare handles external TLS. Internal is plain HTTP. |
| D9 | Observability timing | Phase 1.5 (alongside foundation) | Phase 2 (after foundation stable). Uptime Kuma can handle infra-level uptime. |

---

## Documents Requiring Changes (PO-Owned)

> These documents are owned by the **Product Owner persona** and need revision to align with the design decisions above.

---

### 1. `spec/panomete_platform/README.md`

| Severity | 🔴 Critical |
|----------|------------|

**Problems:**
- The Mermaid architecture diagram (lines 7-58) shows Gate on `:80 / :443` as the external-facing entry point, with no Nginx layer
- Diagram shows Guard, Discover, and business services all behind Gate with path-based routing (`/auth/**`, `/eureka/**`, `/api/**`)
- No mention of Nginx, Cloudflare Tunnel, or `*.panomete.com` subdomain scheme
- Ports shown are outdated

**Changes needed:**
1. Add Nginx as the edge layer (between External Traffic and Gateway)
2. Add Cloudflare Tunnel as the external ingress
3. Update Gate port from `:80 / :443` to `:8000` (internal)
4. Update Guard port from `8080` to `8001`
5. Update Discover port from `8761` to `8999`
6. Show `auth.panomete.com` → Guard and `discovery.panomete.com` → Discover as direct routes through Nginx (not through Gate)
7. Keep `api.panomete.com` → Gate → business services as the API path
8. Update the Tech Stack table to mention Nginx + Cloudflare Tunnel

**Affected lines:** 7-58 (Mermaid diagram), 82-89 (Tech Stack table)

---

### 2. `spec/panomete_platform/01_requirement/011_business_objective.md`

| Severity | 🔴 Critical |
|----------|------------|

**Problems:**
- OBJ-01 (line 129): Describes Guard as "Deploy a Spring Boot OAuth2 Resource Server that validates JWT tokens from Keycloak" — should be "Deploy Keycloak IAM"
- OBJ-01 "Specific" field (line 142): "Deploy Keycloak instance + Flowero Guard service" — no separate "service" needed
- OBJ-01 "Measurable" field (line 143): "All 3 foundation services + 1 canary business service authenticate via Guard" — JWT validation is done at Gate, not Guard. Guard issues tokens.
- OBJ-03 (line 131): Gate described as listening on ports 80/443 — should be internal port 8000 behind Nginx
- OBJ-03 "Measurable" (line 177): "P95 added latency <50ms" — still valid but now includes Nginx hop
- Appendix B (line 424-430): Ports are outdated

**Changes needed:**
1. OBJ-01: Rewrite to clarify Guard IS Keycloak. Gate does JWT validation locally. Guard issues tokens + manages users.
2. OBJ-02: Discover port 8761 → 8999. Remove "dashboard via Gate" references → dashboard accessed directly at `discovery.panomete.com`.
3. OBJ-03: Gate port 80/443 → 8000. Add context that Gate is behind Nginx. Remove TLS termination responsibility (Cloudflare does it).
4. OBJ-01 detailed card: Update "Specific" to remove "Deploy Keycloak instance + Flowero Guard service" — it's just "Deploy Keycloak."
5. Appendix B: Update all port assignments. Add Cloudflare Tunnel and Nginx to technology decisions.

**Affected lines:** 129-133 (objective register), 142-147, 154-163, 175-183, 424-430

---

### 3. `spec/panomete_platform/01_requirement/012_user_stories.md`

| Severity | 🟡 Moderate |
|----------|------------|

**Problems:**
- §3.2 "Inter-Service Story Dependencies" (lines 82-88): Gate US-202 says "Route `/auth/**` → Guard" and "Route `/eureka/**` → Discover" — these no longer go through Gate
- §4 "Service-Level Documents" table (lines 93-98): Links are fine but architecture column says "TBD"

**Changes needed:**
1. Remove dependency: Gate US-202 → Guard US-001 (auth is now direct subdomain, not via Gate path route)
2. Remove dependency: Gate US-202 → Discover (Eureka dashboard is direct subdomain)
3. Update story dependency matrix to reflect that Gate only routes business services
4. Update architecture column if desired (can link to new design docs)

**Affected lines:** 82-88 (dependency table), 54 (story map — remove Discover/Gate auth routes from Gate column)

---

### 4. `spec/flowero_guard/01_requirement/011_business_objective.md`

| Severity | 🔴 Critical |
|----------|------------|

**Problems:**
- §2 "Service Context" mentions port 8080 — should be 8001
- Service context mentions "exposed through Flowero Gate" — should be "exposed through Nginx at `auth.panomete.com`"

**Changes needed:**
1. Change port from `8080` → `8001`
2. Change "exposed through Flowero Gate" → "exposed through Nginx reverse proxy at `auth.panomete.com`"
3. Remove any references to Gate routing `/auth/**`

**Affected lines:** Port references throughout

---

### 5. `spec/flowero_guard/01_requirement/012_user_stories.md`

| Severity | 🔴 Critical |
|----------|------------|

**Problems:**
- US-001 AC-1 (line 80): "Keycloak is accessible at `https://panomete.local/auth` (via Gate)" — should be `auth.panomete.com` directly
- §2 "Service Context" (line 56): Port 8080 → 8001. "exposed through Flowero Gate" → "exposed through Nginx at auth.panomete.com"
- US-003 AC-1 (line 120): "redirected to the Keycloak login page" — should reference `auth.panomete.com` not Gate

**Changes needed:**
1. All references to `https://panomete.local/auth` → `auth.panomete.com`
2. Port 8080 → 8001
3. Remove "(via Gate)" qualifiers — Guard is accessed directly through Nginx
4. Update related services note: Gate does NOT route auth traffic through Guard anymore

**Affected lines:** 56-59 (service context), 80, 120

---

### 6. `spec/flowero_guard/01_requirement/013_acceptance_criteria.md`

| Severity | 🔴 Critical |
|----------|------------|

**Problems:**
- AC-G001a (line 55): "Keycloak is reachable at `https://panomete.local/auth` within 60 seconds" — should be `auth.panomete.com`
- AC-G001f (line 60): "Admin navigates to `/auth/admin/master/console/`" — should use `auth.panomete.com`
- AC-G003a (line 75): "User accesses `https://panomete.local/api/blog/posts` without authentication" — domain should be `api.panomete.com`
- AC-G003a: "The request hits Gate, which redirects to Keycloak" — Gate now returns 401, doesn't redirect to Keycloak. The frontend handles the redirect to `auth.panomete.com`.

**Changes needed:**
1. Global: `https://panomete.local` → `*.panomete.com` (appropriate subdomain per context)
2. AC-G001a: `auth.panomete.com`
3. AC-G003a: Rewrite redirect flow — Gate returns 401; client/frontend redirects to `auth.panomete.com` for login
4. AC-G003a: "request hits Gate" → Gate does NOT proxy auth traffic

**Affected lines:** 55, 60, 75, and all domain references

---

### 7. `spec/flowero_discover/01_requirement/011_business_objective.md`

| Severity | 🔴 Critical |
|----------|------------|

**Problems:**
- OBJ-DISCOVER-01 (line 37): "dashboard accessible through Flowero Gate at `/eureka`" — should be direct at `discovery.panomete.com`
- No port specified in this doc, but implicit port 8761 throughout

**Changes needed:**
1. Dashboard accessible at `discovery.panomete.com` (direct Nginx proxy, not through Gate)
2. Port 8761 → 8999 (backend) / 3999 (frontend dashboard)
3. Remove Gate dependency for dashboard access

**Affected lines:** 37-43

---

### 8. `spec/flowero_discover/01_requirement/012_user_stories.md`

| Severity | 🔴 Critical |
|----------|------------|

**Problems:**
- US-101 AC-1 (line 79): "Eureka dashboard is accessible at `https://panomete.local/eureka` (via Gate)" — should be `discovery.panomete.com` directly
- §2 "Service Context" (line 57): Port 8761 → 8999. "exposed through Flowero Gate" → "exposed through Nginx at discovery.panomete.com"
- US-104 (entire story, lines 132-149): "Eureka Dashboard Accessible via Gateway" — this story is OBSOLETE. Dashboard is accessed directly, not through Gate.

**Changes needed:**
1. US-101 AC-1: `discovery.panomete.com` (via Nginx, not Gate)
2. Service context: Port 8761 → 8999. Remove Gate dependency.
3. **US-104 should be removed or rewritten** — dashboard is no longer "via Gateway." Replace with: "Eureka Dashboard Accessible via Nginx at discovery.panomete.com"
4. US-104 AC-3: Remove "Gate returns 401" — Nginx handles this differently
5. Update story estimation summary (remove or repoint US-104)

**Affected lines:** 57-60 (service context), 79, 132-149 (US-104 entire story), 160 (summary table)

---

### 9. `spec/flowero_discover/01_requirement/013_acceptance_criteria.md`

| Severity | 🔴 Critical |
|----------|------------|

**Problems:**
- AC-D101a (line 55): "Eureka dashboard is reachable at `https://panomete.local/eureka` within 30 seconds" — should be `discovery.panomete.com`
- AC-D101e (line 59): "Gateway routes `/eureka/**` to Eureka" — Nginx does this, not Gate
- US-104 criteria (entire section, lines 80-86): All reference Gate — should reference Nginx

**Changes needed:**
1. Global: `https://panomete.local/eureka` → `discovery.panomete.com`
2. AC-D101e: Replace "Gateway routes" with "Nginx proxies"
3. US-104 section: Rewrite all 3 ACs to reference Nginx and `discovery.panomete.com`. Gate is not involved.

**Affected lines:** 55, 59, 80-86

---

### 10. `spec/flowero_gate/01_requirement/011_business_objective.md`

| Severity | 🔴 Critical |
|----------|------------|

**Problems:**
- OBJ-GATE-01 (line 38): "listens on ports 80/443" — should be port 8000 (internal, behind Nginx)
- OBJ-GATE-01 "Measurable" (line 39): "Gateway healthy within 30s" — still valid
- OBJ-GATE-02 (line 48): "ALL external requests — including Keycloak login (`/auth/**`), Eureka dashboard (`/eureka/**`)" — these NO LONGER go through Gate
- OBJ-GATE-02 "Measurable" (line 49): "Keycloak, Eureka, and business services all accessible via `https://panomete.local/{path}`" — outdated

**Changes needed:**
1. OBJ-GATE-01: Port 80/443 → 8000. Add context: "behind Nginx reverse proxy."
2. OBJ-GATE-02: Remove Keycloak and Eureka from routing scope. Gate only routes business APIs. Foundation services are routed directly by Nginx.
3. OBJ-GATE-02 "Measurable": Update to reflect `api.panomete.com/api/{service}/**` pattern only
4. OBJ-GATE-03 (perimeter security): Gate still does JWT validation at the perimeter of the API surface — but "perimeter" is now `api.panomete.com`, not the entire platform
5. §3 "Out of Scope" (line 77-81): TLS termination is no longer Gate's concern

**Affected lines:** 38-39, 48-52, 77-81

---

### 11. `spec/flowero_gate/01_requirement/012_user_stories.md`

| Severity | 🔴 Critical |
|----------|------------|

**Problems:**
- US-201 AC-1 (line 80): "listens on ports 80 and 443 and is accessible at `https://panomete.local`" — should be port 8000 at `api.panomete.com`
- US-201 AC-2 (line 81): "accessible at `https://panomete.local`" — should be `api.panomete.com`
- US-202 AC-1 (line 100): "route `id: auth`, path `/auth/**` → `lb://flowero-guard`" — auth is NO LONGER routed through Gate
- US-202 AC-2 (line 101): "route `id: discover`, path `/eureka/**` → `lb://flowero-discover`" — discover is NO LONGER routed through Gate
- US-202 AC-3 (line 102): Blog route is fine (`/api/blog/**`) — this stays
- US-202 AC-4 (line 103): "unmatched path returns 404" — fine
- US-206 (entire story, lines 175-191): TLS termination at Gate — Cloudflare handles this now. Story should be removed or rewritten as "Internal TLS is not needed (Cloudflare + trusted Docker network)."
- §2 "Service Context" (line 56): Ports 80/443 → 8000. Dependencies section: Remove "Flowero Guard (routing)" and "Flowero Discover (routing)" — Gate routes only business services now.

**Changes needed:**
1. Global: `https://panomete.local` → `api.panomete.com`. Port references 80/443 → 8000.
2. US-201 AC-1/2: Update port and domain
3. US-202: Remove AC-1 (auth route) and AC-2 (discover route) — these routes don't exist in Gate
4. US-202: Keep AC-3+ for business services only
5. US-206: Remove or rewrite — TLS is now Cloudflare's concern
6. Service context: Update dependencies to reflect Gate only depends on Guard for JWT validation (JWKS), Discover for `lb://` resolution of business services. Not for routing Guard/Discover themselves.
7. Update story summary table (lines 196-204): Remove US-206 points, adjust US-202 points (fewer ACs)

**Affected lines:** 56-60, 80-81, 100-105, 175-191, 196-204

---

### 12. `spec/flowero_gate/01_requirement/013_acceptance_criteria.md`

| Severity | 🔴 Critical |
|----------|------------|

**Problems:**
- AC-W201a (line 55): "Gate service with ports 80:80 and 443:443" → port 8000
- AC-W201b (line 56): "Admin accesses `https://panomete.local`" → `api.panomete.com`
- AC-W202a (line 65): "Route to Keycloak (`/auth/**`)" — this route no longer exists in Gate
- AC-W202b (line 66): "Route to Eureka (`/eureka/**`)" — this route no longer exists in Gate
- AC-W203a (line 76): "Gate has JWT validation enabled on route `/api/**`" — correct, stays
- AC-W203e (line 80): "Routes `/auth/**` and `/eureka/**` are configured with `security: permit-all`" — these routes don't exist in Gate
- AC-W206a/b/c (lines 105-107): TLS termination at Gate — Cloudflare handles this

**Changes needed:**
1. Global: `https://panomete.local` → `api.panomete.com`. Remove port 80:80/443:443 references.
2. AC-W202a: Remove (auth route no longer exists in Gate)
3. AC-W202b: Remove (discover route no longer exists in Gate)
4. AC-W203e: Remove (permit-all for `/auth/**` and `/eureka/**` — these are Nginx's concern now)
5. AC-W206a/b/c: Remove or rewrite — TLS is Cloudflare, not Gate
6. Update AC summary table to reflect removed/reduced AC counts

**Affected lines:** 55-56, 65-66, 80, 105-107, and summary table (113-121)

---

### 13. `README.md` (project root)

| Severity | 🟡 Moderate |
|----------|------------|

**Problems:**
- Service table (lines 4-14): References to Flowero Guard as "Oauth" with Keycloak — should clarify Guard IS Keycloak
- No mention of Nginx, Cloudflare Tunnel, or existing infrastructure

**Changes needed:**
1. Add Nginx + Cloudflare Tunnel to the infrastructure context
2. Clarify Flowero Guard = Keycloak (not a proxy/wrapper)
3. Add reference to `spec/meeting-minute/affect-doc.md` for cross-persona handoff

**Affected lines:** 4-14 (service table), end of file (links section)

---

## Documents NOT Affected

> These PO-owned documents remain valid as-is:

| Document | Reason |
|----------|--------|
| `panomete_platform/01_requirement/014_stakeholder_analysis.md` | Stakeholder roles and concerns unchanged |
| `flowero_guard/01_requirement/011_business_objective.md` §1-2 (mission) | Guard's mission unchanged — still the identity provider |
| `flowero_discover/01_requirement/011_business_objective.md` §1 (mission) | Discover's mission unchanged — still the service registry |
| `flowero_gate/01_requirement/011_business_objective.md` §1 (mission) | Gate's mission unchanged — still routes, secures, observes |

---

## Summary: Change Impact by Severity

| Severity | Count | Documents |
|----------|:----:|-----------|
| 🔴 Critical (blocks development) | 12 | All Guard, Discover, Gate requirement docs + platform README + business objectives |
| 🟡 Moderate (creates confusion) | 1 | Root README.md |
| 🟢 No change needed | 2 | Stakeholder analysis, infrastructure scripts |

---

## Handoff Checklist

> Before handing updated requirements back to Dev persona:

- [ ] PO updates all 🔴 documents with corrections listed above
- [ ] PO reviews changed stories with Dev persona (3 amigos)
- [ ] PO updates story point estimates if scope changed (e.g., Gate US-206 removed)
- [ ] PO confirms the new domain scheme (`*.panomete.com`) in all ACs
- [ ] PO confirms route ownership: Nginx = subdomain routing, Gate = path routing for business APIs
- [ ] Dev persona rewrites 02_design docs to match corrected requirements

---

## Related Documents

| Document | Relationship |
|----------|-------------|
| [[panomete_platform/021_architecture_decision_records]] | Design decisions that triggered these changes |
| [[panomete_platform/025_software_architecture_document]] | Updated architecture reflecting decisions D1-D9 |
| [[../README]] | Project-level overview (also needs update) |

---

> **Next Step:** PO reviews this document → updates requirement docs → signals Dev persona to rewrite 02_design docs.
> **Cross-persona handoff:** This document is the contract between Dev (current) and PO (next action) on what must change.
