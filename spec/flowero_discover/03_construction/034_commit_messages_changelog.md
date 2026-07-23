---
document_type: Commit Messages / Changelog
version: "0.1"
status: Active
author: "Dev Persona"
created: "2026-07-23"
last_updated: "2026-07-23"
project_name: "Flowero Discover"
project_id: "PAN-DISCOVER-001"
parent_platform: "Panomete Platform"
classification: "Internal"
tags: [commit-messages, changelog, conventional-commits, eureka, panomete]
standard_ref:
  - SWEBOK v4 — Construction
  - Conventional Commits v1.0
---

# Commit Messages / Changelog — Flowero Discover

> **Service:** Flowero Discover (Spring Cloud Netflix Eureka)
> **Platform:** Panomete Platform
> **Version:** 0.1 | **Status:** Active
> **Last Updated:** 2026-07-23

---

## 1. Purpose

> All Flowero Discover commits follow [Conventional Commits](https://www.conventionalcommits.org/). The changelog is the human-readable history derived from these commits.

## 2. Commit Message Format

```
<type>(discover): <description>

[optional body — explains WHY]

[optional footer(s)]
```

### Scopes

| Scope | When to use |
|-------|-----------|
| `discover` | Changes to Flowero Discover (default) |
| `docker` | Dockerfile or compose changes |
| `deps` | Dependency updates |

## 3. Commit Guidelines

| Rule | Rationale |
|------|----------|
| Type + scope in every commit | `feat(discover): ...` not `add feature` |
| Imperative mood | "add" not "added" |
| First line ≤ 72 chars | Readable in `git log --oneline` |
| Body explains WHY | Code shows what; commit explains why |
| One logical change per commit | Atomic — easy to revert, bisect |
| No WIP on main | Squash before merge |

## 4. Changelog

### [0.1.0] — 2026-07-23 (Initial Implementation)

#### Features
- **discover:** add `@EnableEurekaServer` to main application class
- **discover:** configure standalone mode (`register-with-eureka: false`, `fetch-registry: false`)
- **discover:** set dual ports — 8999 (REST API) + 3999 (dashboard via Docker mapping)
- **discover:** enable self-preservation with 85% renewal threshold
- **discover:** configure eviction timer at 5s interval
- **discover:** add Actuator health endpoint (`/actuator/health`)
- **discover:** expose `health` and `info` management endpoints
- **docker:** add multi-stage Dockerfile (JDK 25 build → JRE 25 runtime)
- **docker:** configure non-root user and ZGC flags
- **docker:** add docker-compose fragment with healthcheck

#### Tests
- **test:** verify context loads with Eureka server enabled
- **test:** verify dashboard reachable at `/`
- **test:** verify Actuator health returns `{"status":"UP"}`
- **test:** verify `/eureka/apps` returns valid response (XML)
- **test:** verify `/eureka/apps` returns JSON with Accept header
- **test:** verify no self-registration in standalone mode

#### Documentation
- **docs:** rewrite README with architecture, quick start, and API reference
- **docs:** add configuration reference table with rationale
- **docs:** add design decisions section with ADR links

#### Chores
- **chore:** add `.dockerignore` for clean Docker builds
- **chore:** add Actuator dependency to `build.gradle`

### [0.0.1] — 2026-07-22 (Project Skeleton)

#### Chores
- **chore:** initialize Spring Boot 4.1.0 project via Spring Initializr
- **chore:** configure Gradle 9.5.1 with Spring Cloud 2025.1.2 BOM
- **chore:** add `spring-cloud-starter-netflix-eureka-server` dependency
- **chore:** add `spring-boot-starter-test` with JUnit Platform

## 5. Commit Example

```bash
# Feature commit
git commit -m "feat(discover): configure standalone mode with self-preservation

Standalone mode prevents Eureka from self-registering or
fetching its own registry. Self-preservation keeps the registry
intact during transient network hiccups in the homelab environment.

- register-with-eureka: false
- fetch-registry: false
- enable-self-preservation: true
- renewal-percent-threshold: 0.85
- eviction-interval-timer-in-ms: 5000"

# Breaking change example (hypothetical)
git commit -m "feat(discover)!: switch from standalone to peer-aware mode

BREAKING CHANGE: Eureka now requires at least 2 peer nodes.
Update docker-compose to include a second discover instance."
```

## 6. Changelog Generation

Generated via [git-cliff](https://git-cliff.org/) from Conventional Commit history:

```bash
git cliff -o CHANGELOG.md --tag-pattern 'v*'
```

---

## Related Documents

| Document | Relationship |
|----------|-------------|
| [[035_coding_standards_development]] | Code standards these commits follow |
| [[031_README_developer_guide]] | Developer setup guide |
| [[032_build_scripts]] | Build process triggered by commits |

---

> **Template Standard:** Based on SWEBOK v4, Conventional Commits v1.0
> **Usage:** Every commit uses `type(scope): description` format. The changelog is auto-generated from commit history. Never write changelogs by hand.
