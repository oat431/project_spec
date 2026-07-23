---
document_type: Dependency Manifest
version: "0.1"
status: Active
author: "Dev Persona"
created: "2026-07-23"
last_updated: "2026-07-23"
project_name: "Flowero Discover"
project_id: "PAN-DISCOVER-001"
parent_platform: "Panomete Platform"
classification: "Internal"
tags: [dependencies, gradle, spring-cloud, eureka, panomete]
standard_ref:
  - SWEBOK v4 — Construction
  - OWASP Top 10 (A06:2021 — Vulnerable Dependencies)
---

# Dependency Manifest — Flowero Discover

> **Service:** Flowero Discover (Spring Cloud Netflix Eureka)
> **Platform:** Panomete Platform
> **Version:** 0.1 | **Status:** Active
> **Last Updated:** 2026-07-23

---

## 1. Purpose

> Complete inventory of all Flowero Discover dependencies — direct, transitive, and platform-level. Know what you depend on. Audit regularly.

## 2. Dependency Overview

| Metric | Count |
|--------|-------|
| Direct Dependencies (implementation) | 2 |
| Test Dependencies | 2 |
| Transitive Dependencies (approx.) | ~40+ |
| Known Vulnerabilities | 0 |
| License Issues | 0 |

## 3. Production Dependencies

### Direct (declared in `build.gradle`)

| Dependency | Group:Artifact | Version | License | Purpose |
|-----------|---------------|---------|---------|---------|
| Eureka Server | `org.springframework.cloud:spring-cloud-starter-netflix-eureka-server` | 4.2.0 | Apache-2.0 | Service registry + dashboard |
| Actuator | `org.springframework.boot:spring-boot-starter-actuator` | 4.1.0 | Apache-2.0 | Health checks + management endpoints |

### Test Dependencies

| Dependency | Group:Artifact | Version | License | Purpose |
|-----------|---------------|---------|---------|---------|
| Spring Boot Test | `org.springframework.boot:spring-boot-starter-test` | 4.1.0 | Apache-2.0 | Integration testing with JUnit 5 + AssertJ |
| JUnit Launcher | `org.junit.platform:junit-platform-launcher` | 1.11.x | EPL-2.0 | JUnit 5 test engine |

### Key Transitive Dependencies (managed by Spring Cloud BOM `2025.1.2`)

| Dependency | Purpose | Managed By |
|-----------|---------|-----------|
| `spring-boot-starter-web` (Tomcat) | Embedded servlet container | Boot starter |
| `spring-cloud-netflix-eureka-server` | Eureka server core | Spring Cloud BOM |
| `eureka-core` / `eureka-client` | Netflix Eureka libraries | Spring Cloud BOM |
| `jersey-common` / `jersey-client` | JAX-RS client (Eureka uses Jersey 3) | Spring Cloud BOM |
| `jackson-databind` | JSON serialization | Spring Boot BOM |
| `xstream` | XML serialization (Eureka legacy format) | Spring Cloud BOM |
| `spring-boot-starter-actuator` | Health, info, metrics endpoints | Boot starter |

## 4. Dependency Management Strategy

### Spring Cloud BOM Pattern

The BOM (`spring-cloud-dependencies:2025.1.2`) manages ALL transitive Spring Cloud versions. We only declare top-level starters — the BOM ensures compatibility:

```groovy
ext {
    set('springCloudVersion', "2025.1.2")
}

dependencyManagement {
    imports {
        mavenBom "org.springframework.cloud:spring-cloud-dependencies:${springCloudVersion}"
    }
}
```

**Rule:** Never pin individual Spring Cloud library versions. Let the BOM handle version alignment across Eureka, Spring Web, Jackson, and Jersey.

### Why No Additional Libraries

Flowero Discover is deliberately **minimal**:
- **No database driver** — Eureka is fully in-memory
- **No Valkey/Redis client** — rate limiting is Gate's concern
- **No security library** — internal Docker network is trusted (ADR-D003)
- **No Lombok** — explicit code over magic (coding standard)
- **No web client** — Eureka Server doesn't make outbound HTTP calls in standalone mode

## 5. Runtime Dependencies

| Runtime | Version | Purpose |
|---------|---------|---------|
| Java (JDK/JRE) | 25 LTS (Eclipse Temurin) | JVM |
| Gradle | 9.5.1 (wrapper) | Build system |
| Docker Base Image | `eclipse-temurin:25-jre-noble` | Container runtime |

## 6. Platform-Level Shared Dependencies

Flowero Discover is a **standalone foundation service** — it has no runtime dependencies on other services. It IS the dependency that other services consume.

| Service | How It Depends on Discover |
|---------|---------------------------|
| Flowero Gate | Resolves `lb://` service routes through Eureka |
| Flowero Guard | Registers itself for health visibility |
| All business services | `spring-cloud-starter-netflix-eureka-client` → auto-register |

## 7. Vulnerability Status

| Scan Date | Tool | Critical | High | Medium | Low |
|-----------|------|:---:|:---:|:---:|:---:|
| 2026-07-23 | — (not yet scanned) | — | — | — | — |

> ⚠️ OWASP Dependency-Check not yet configured. Recommended: add `org.owasp.dependencycheck` Gradle plugin to CI pipeline.

## 8. Dependency Update Cadence

| Event | Action |
|-------|--------|
| Spring Boot minor/patch release | Test within 1 week, auto-merge if tests pass |
| Spring Cloud BOM update | Test within 1 week, may change Jersey/Eureka internals |
| Major version (Boot 5.x, Spring Cloud 2026.x) | Full regression test, manual review |
| Security vulnerability (Critical/High) | Patch immediately |
| Monthly audit | Scheduled CI dependency check |

---

## Related Documents

| Document | Relationship |
|----------|-------------|
| [[032_build_scripts]] | Build pipeline that resolves these deps |
| [[031_README_developer_guide]] | Dev setup instructions |
| [[021_architecture_decision_records]] | Why Eureka, why standalone, why no DB |
| [[062_coding_standards_security]] | Security scanning standards |

---

> **Template Standard:** Based on SWEBOK v4, OWASP Top 10 (A06:2021)
> **Usage:** Flowero Discover has exactly 2 production dependencies — deliberately minimal. The Spring Cloud BOM manages ~40 transitive dependencies. No database drivers, no cache clients, no security libraries.
