---
document_type: Build Scripts
version: "0.1"
status: Active
author: "Dev Persona"
created: "2026-07-24"
last_updated: "2026-07-24"
project_name: "Flowero Gate"
project_id: "PAN-GATE-001"
parent_platform: "Panomete Platform"
classification: "Internal"
tags: [build-scripts, ci-cd, gradle, docker, spring-boot, panomete]
standard_ref:
  - SWEBOK v4 — Construction
  - 12-Factor App (Build, release, run)
---

# Build Scripts — Flowero Gate

> **Service:** Flowero Gate (Spring Cloud Gateway)
> **Platform:** Panomete Platform
> **Version:** 0.1 | **Status:** Active
> **Last Updated:** 2026-07-24

---

## 1. Purpose

> Documents every command that transforms source code into a deployable artifact. The build pipeline IS the quality gate. If it's not in the pipeline, it doesn't happen.

## 2. Build Pipeline

```mermaid
flowchart LR
    CODE[Source Code] --> COMPILE[Compile Java]
    COMPILE --> PROCESS[Process Resources]
    PROCESS --> TEST[Test — JUnit 5]
    TEST --> JAR[Package bootJar]
    JAR --> DOCKER[Docker Build]
    DOCKER --> PUSH[Push to Registry]

    style CODE fill:#2196F3,color:#fff
    style COMPILE fill:#FF9800,color:#fff
    style PROCESS fill:#FF9800,color:#fff
    style TEST fill:#4CAF50,color:#fff
    style JAR fill:#9C27B0,color:#fff
    style DOCKER fill:#607D8B,color:#fff
    style PUSH fill:#4CAF50,color:#fff
```

## 3. Gradle Commands

| Command | Purpose | Output |
|---------|---------|--------|
| `./gradlew build` | Compile + test + package (full pipeline) | `build/libs/flowerogate-0.0.1-SNAPSHOT.jar` |
| `./gradlew build -x test` | Quick assemble, skip tests | Same JAR, no test run |
| `./gradlew compileJava` | Syntax check only | Class files in `build/` |
| `./gradlew test` | Run all tests (JUnit 5) | Test report at `build/reports/tests/test/` |
| `./gradlew bootJar` | Create fat JAR (includes embedded Netty) | `build/libs/flowerogate-0.0.1-SNAPSHOT.jar` |
| `./gradlew bootRun` | Start with embedded Netty (dev) | Port 8080 |
| `./gradlew clean` | Remove all build artifacts | Wipes `build/` |
| `./gradlew dependencies` | Print full dependency tree | Console output |
| `./gradlew dependencyCheckAnalyze` | OWASP vulnerability scan | Report at `build/reports/dependency-check/` |

**Gradle Wrapper:** Always use `./gradlew` (Gradle 9.5.1, checked into VCS). Never rely on system Gradle.

## 4. Gradle Build Phases

From actual `./gradlew build` output:

```
> Task :compileJava          # Java 25 source → bytecode
> Task :processResources     # Copy YAML configs → build
> Task :classes              # Compile complete
> Task :resolveMainClassName # Detect @SpringBootApplication
> Task :bootJar              # Fat JAR (84 MB)
> Task :jar                  # Plain JAR (42 KB)
> Task :assemble             # All archives done
> Task :compileTestJava      # Test sources
> Task :processTestResources # Test config files
> Task :testClasses          # Test compile done
> Task :test                 # JUnit 5 — 9 tests run

BUILD SUCCESSFUL in 32s
9 actionable tasks: 9 executed
9 tests completed, 0 failed
```

### Build Artifacts

| Artifact | Size | Path |
|---------|------|------|
| Fat JAR (bootJar) | 84 MB | `build/libs/flowerogate-0.0.1-SNAPSHOT.jar` |
| Plain JAR (no deps) | 42 KB | `build/libs/flowerogate-0.0.1-SNAPSHOT-plain.jar` |
| Test Report | HTML | `build/reports/tests/test/index.html` |

## 5. Docker Multi-Stage Build

```dockerfile
# ─── Stage 1: Build ───
FROM eclipse-temurin:25-jdk-alpine AS builder
WORKDIR /app
# Cache dependencies first (layer reuse)
COPY gradlew gradlew.bat settings.gradle build.gradle ./
COPY gradle/ gradle/
RUN chmod +x gradlew && ./gradlew dependencies --no-daemon 2>/dev/null || true
# Build application
COPY src/ src/
RUN ./gradlew bootJar --no-daemon -x test

# ─── Stage 2: Run ───
FROM eclipse-temurin:25-jre-alpine
RUN addgroup -S flowero && adduser -S flowero -G flowero
USER flowero
WORKDIR /app
COPY --from=builder /app/build/libs/*.jar app.jar
EXPOSE 8000
ENTRYPOINT ["java", \
  "-XX:+UseZGC", \
  "-XX:MaxRAMPercentage=75.0", \
  "-Djava.security.egd=file:/dev/./urandom", \
  "-jar", "app.jar"]
```

### JVM Flags

| Flag | Purpose |
|------|---------|
| `-XX:+UseZGC` | Z Garbage Collector — low pause, suitable for reactive workloads |
| `-XX:MaxRAMPercentage=75.0` | Cap heap at 75% of container memory (256-512 MB) |
| `-Djava.security.egd=file:/dev/./urandom` | Faster startup entropy |

## 6. Docker Compose Fragment

```yaml
services:
  gateway:
    image: oat431/flowerogate:latest
    container_name: flowerogate
    restart: unless-stopped
    ports:
      - "127.0.0.1:8000:8000"
    environment:
      SERVER_PORT: "8000"
      SPRING_PROFILES_ACTIVE: prod
      JWT_ISSUER_URI: https://auth.panomete.com/realms/panomete
      JWT_JWK_SET_URI: https://auth.panomete.com/realms/panomete/protocol/openid-connect/certs
      REDIS_HOST: local-valkey
      REDIS_PORT: "6379"
      REDIS_PASSWORD: ${VALKEY_PASSWORD}
      EUREKA_CLIENT_ENABLED: "true"
      EUREKA_URI: http://flowero-discover:8999/eureka
    networks:
      - shared-network

networks:
  shared-network:
    external: true
    name: db-network
```

## 7. Test Results

Tests run on every `./gradlew build`. All 9 tests pass:

| Test Class | Test Method | What It Verifies |
|-----------|-------------|-----------------|
| `FlowerogateApplicationTests` | `contextLoads()` | Spring context starts with all beans |
| `RouteTests` | `fallbackNotFoundAccessible()` | Fallback 404 endpoint returns 404 |
| `RouteTests` | `fallbackServiceReturns503()` | Circuit breaker fallback returns 5xx |
| `SecurityTests` | `actuatorHealthAccessibleWithoutAuth()` | Health endpoint is public |
| `SecurityTests` | `fallbackNotFoundAccessibleWithoutAuth()` | Fallback endpoint is public |
| `SecurityTests` | `protectedEndpointReturns401WithoutJwt()` | No token → 401 Unauthorized |
| `SecurityTests` | `protectedEndpointReturns401WithInvalidToken()` | Invalid token → 401 |
| `SecurityTests` | `protectedEndpointReturns401WithExpiredToken()` | Expired token → 401 |
| `SecurityTests` | `validJwtAccessesProtectedEndpoint()` | Valid JWT → not 401/403 |

### Known Cosmetic Warnings (non-blocking)

- **OTLP shutdown noise:** `Failed to publish metrics to OTLP receiver` — OTLP registry tries to flush on shutdown when no collector is running. Harmless.
- **Gradle toolchain warning:** If `JAVA_HOME` points to GraalVM with a path prefix quirk, Gradle's toolchain auto-detection still works. The warning is cosmetic.

---

## Related Documents

| Document | Relationship |
|----------|-------------|
| [[031_README_developer_guide]] | Build instructions for developers |
| [[033_dependency_manifest]] | Dependencies being built |
| [[034_commit_messages_changelog]] | Commit standards |
| [[052_deployment_plan]] | What happens after build |
