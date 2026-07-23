---
document_type: Commit Messages / Changelog
version: "0.1"
status: Active
author: "Dev Persona"
created: "2026-07-24"
last_updated: "2026-07-24"
project_name: "Flowero Gate"
project_id: "PAN-GATE-001"
parent_platform: "Panomete Platform"
classification: "Internal"
tags: [commit-messages, changelog, conventional-commits, panomete]
standard_ref:
  - SWEBOK v4 — Construction
  - Conventional Commits v1.0
---

# Commit Messages / Changelog — Flowero Gate

> **Service:** Flowero Gate (Spring Cloud Gateway)
> **Platform:** Panomete Platform
> **Version:** 0.1 | **Status:** Active
> **Last Updated:** 2026-07-24

---

## 1. Purpose

> Standardized commit messages and human-readable changelog. Good commit messages are documentation — "fix bug" is useless. "fix(gate): resolve NPE on missing JWT claims" is useful.

## 2. Commit Message Format

Based on [Conventional Commits](https://www.conventionalcommits.org/):

```
<type>(<scope>): <description>

[optional body — explains WHY, not WHAT]

[optional footer(s)]
```

### Types Used

| Type | Description | Example from this repo |
|------|-----------|----------------------|
| `feat` | New feature | `feat: e2e oauth gateway` |
| `fix` | Bug fix | `fix: use direct http:// URL instead of lb:// for shortlink route` |
| `refactor` | Code restructuring | `refactor: docker-compose pulls from Docker Hub` |
| `test` | Add or update tests | `test: add security integration tests + test infrastructure` |
| `chore` | Build, tooling, deps | `chore: add Dockerfile, docker-compose.yml, .dockerignore` |
| `add` | New non-feature addition | `add: https redirect` |

### Scopes

| Scope | Applies To |
|-------|-----------|
| `gate` | Gateway routing, filters, predicates |
| `security` | Auth, CORS, JWT handling |
| `docker` | Container config, compose, health checks |
| `test` | Test infrastructure, test config |

## 3. Commit Guidelines

| Rule | Rationale |
|------|----------|
| Use imperative mood | "add" not "added" — matches git merge default |
| First line ≤ 72 characters | Readable in `git log --oneline` |
| Body explains WHY, not WHAT | Code shows *what* changed; commit explains *why* |
| One logical change per commit | Atomic commits → easy to revert, bisect |
| No WIP commits on main | Squash before merge if needed |

## 4. Changelog

### [0.1.0] — 2026-07-24 (Panomete Platform Adaptation)

#### Features
- **gate:** Replace demo routes with Panomete business routes — `lb://cute-gufo`, `lb://fluffy-mouton`, `lb://tiny-mchwa`
- **gate:** Enable Eureka registration pointing to `flowero-discover:8999`
- **gate:** Switch rate limiting backend to shared Valkey (`local-valkey`) with password auth
- **gate:** Join `db-network` external Docker network
- **gate:** Wildcard CORS support via `addAllowedOriginPattern` for `*.panomete.com`
- **gate:** Realm migration — `flowerogate` → `panomete` in all configs
- **gate:** Application name → `flowero-gate` for Eureka dashboard alignment

#### Fixes
- **gate:** Remove hardcoded post-login redirect URL — now configurable via `app.post-login-redirect-url`
- **gate:** Fix test context — add OAuth2 client registration config for SecurityTests
- **gate:** Fix test issuer references — `/realms/flowerogate` → `/realms/panomete`

#### Documentation
- **gate:** Full construction docs — README, build scripts, dependency manifest, coding standards, code review records

---

### [0.0.1-SNAPSHOT] — 2026-07-22 (Initial Implementation)

#### Features
- **gate:** Spring Cloud Gateway with reactive Netty runtime
- **gate:** OAuth2 Resource Server — local JWT validation via cached JWKS
- **gate:** OAuth2 Client — browser login redirect flow via Keycloak
- **gate:** `JwtClaimHeaderFilter` — extracts `sub`, `email`, `realm_access.roles`, `scope` → forwards as `X-User-*` headers
- **gate:** `ResilientRedisRateLimiter` — fail-open when Valkey is down
- **gate:** `RateLimiterConfig` — principal, IP, and API-key key resolvers
- **gate:** `TraceIdFilter` — W3C traceparent + X-Trace-Id propagation
- **gate:** `RequestLoggingFilter` — structured JSON logging (method, path, status, latency, route)
- **gate:** `RateLimitResponseFilter` — standardized 429 JSON body
- **gate:** `SecurityConfig` — OAuth2 Resource Server + OAuth2 Client
- **gate:** `FallbackController` — 404 + circuit breaker fallbacks
- **gate:** `GatewayExceptionHandler` — global error handler
- **gate:** Error body logging with sensitive data masking

#### Chores
- **gate:** Initial Spring Initializr skeleton — Boot 4.1.0, Cloud 2025.1.2, Java 25
- **docker:** Multi-stage Dockerfile — JDK build → JRE runtime, non-root user, ZGC
- **docker:** Docker Compose with healthchecks, resource limits

#### Tests
- **gate:** `RouteTests` — fallback 404, circuit breaker 503
- **gate:** `SecurityTests` — public endpoints, 401 without/invalid/expired JWT, valid JWT access
- **gate:** `JwtTestHelper` — RSA key generation, JWK set, signed JWT factory
- **gate:** `TestSecurityConfig` — permissive security for routing tests

---

## 5. Changelog Generation

### Tooling

| Tool | Purpose | Command |
|------|---------|---------|
| [git-cliff](https://git-cliff.org/) | Auto-generate CHANGELOG.md from Conventional Commits | `git cliff -o CHANGELOG.md` |
| Conventional Commits | Commit message standard | Followed manually (no commitlint hook yet) |

---

## Related Documents

| Document | Relationship |
|----------|-------------|
| [[035_coding_standards_development]] | Code standards these commits follow |
| [[031_README_developer_guide]] | Project overview and setup |
| [[032_build_scripts]] | Build process these commits trigger |
