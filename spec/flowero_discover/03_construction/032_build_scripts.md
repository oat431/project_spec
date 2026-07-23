---
document_type: Build Scripts
version: "0.1"
status: Active
author: "Dev Persona"
created: "2026-07-23"
last_updated: "2026-07-23"
project_name: "Flowero Discover"
project_id: "PAN-DISCOVER-001"
parent_platform: "Panomete Platform"
classification: "Internal"
tags: [build-scripts, gradle, docker, ci-cd, eureka, panomete]
standard_ref:
  - SWEBOK v4 — Construction
  - 12-Factor App (Build, Release, Run)
---

# Build Scripts — Flowero Discover

> **Service:** Flowero Discover (Spring Cloud Netflix Eureka)
> **Platform:** Panomete Platform
> **Version:** 0.1 | **Status:** Active
> **Last Updated:** 2026-07-23

---

## 1. Purpose

> Documents the build pipeline for Flowero Discover — Gradle commands, Docker multi-stage build, and artifact outputs. The build pipeline **IS the quality gate**.

## 2. Build Pipeline

```mermaid
flowchart LR
    SRC["src/main/java + resources"] --> COMPILE["compileJava"]
    COMPILE --> PROCESS["processResources"]
    PROCESS --> TEST["test (6 tests)"]
    TEST --> PACKAGE["bootJar"]
    PACKAGE --> IMAGE["Docker Image"]
    IMAGE --> DEPLOY["docker compose up"]

    style SRC fill:#2196F3,color:#fff
    style COMPILE fill:#FF9800,color:#fff
    style TEST fill:#4CAF50,color:#fff
    style PACKAGE fill:#9C27B0,color:#fff
    style IMAGE fill:#607D8B,color:#fff
    style DEPLOY fill:#4CAF50,color:#fff
```

## 3. Gradle Commands

| Command | Purpose | Output |
|---------|---------|--------|
| `./gradlew build` | Full pipeline: compile → test → bootJar | `build/libs/flowerodiscovery-0.0.1-SNAPSHOT.jar` |
| `./gradlew build -x test` | Quick assemble, skip tests | JAR only, no test run |
| `./gradlew compileJava` | Syntax check only | `build/classes/` |
| `./gradlew test` | Run all tests (JUnit 5) | `build/reports/tests/test/index.html` |
| `./gradlew bootJar` | Package fat JAR | `build/libs/flowerodiscovery-0.0.1-SNAPSHOT.jar` |
| `./gradlew bootRun` | Start embedded server locally | Eureka on `:8999` |
| `./gradlew clean` | Remove build artifacts | Deletes `build/` |
| `./gradlew dependencies` | Print dependency tree | stdout |
| `./gradlew dependencyCheckAnalyze` | OWASP vulnerability scan | `build/reports/dependency-check/` |

### Gradle Wrapper

The wrapper (`gradlew`, `gradlew.bat`, `gradle/wrapper/`) is **committed to VCS**. All developers use `./gradlew` — never system Gradle.

```bash
# Wrapper version
./gradlew --version
# Gradle 9.5.1, JVM 25
```

## 4. Gradle Build Configuration

**`build.gradle` key settings:**

| Setting | Value | Purpose |
|---------|-------|---------|
| Spring Boot plugin | `4.1.0` | Boot 4.1.x |
| Dependency management | `1.1.7` | BOM version management |
| Java toolchain | `25` | JVM version pin |
| Spring Cloud BOM | `2025.1.2` | Compatible Spring Cloud versions |
| JUnit Platform | `useJUnitPlatform()` | JUnit 5 test engine |

**Build phases (from `./gradlew build`):**

```
1. compileJava         — Compile main sources
2. processResources    — Copy application.yaml
3. classes             — Done compiling
4. compileTestJava     — Compile test sources
5. testClasses         — Done compiling tests
6. test                — Run 6 JUnit 5 tests
7. bootJar             — Package fat JAR with embedded Tomcat
8. jar                 — Package plain JAR (no Tomcat)
9. check               — All verification done
10. build              — BUILD SUCCESSFUL
```

## 5. Docker Multi-Stage Build

**Dockerfile structure:**

```dockerfile
# Stage 1: Build (eclipse-temurin:25-jdk-noble)
#   - Copy Gradle wrapper + build files
#   - Download dependencies (cached)
#   - Copy source code
#   - ./gradlew bootJar

# Stage 2: Runtime (eclipse-temurin:25-jre-noble)
#   - Create non-root user 'flowero'
#   - Copy bootJar from Stage 1
#   - EXPOSE 8999 3999
#   - ENTRYPOINT: java -Xms128m -Xmx192m -XX:+UseZGC -jar app.jar
```

**Docker commands:**

```bash
# Build image
docker build -t flowero-discover .

# Run container
docker run -d \
  --name flowero-discover \
  -p 8999:8999 \
  -p 3999:8999 \
  flowero-discover

# Verify
curl http://localhost:8999/actuator/health
```

**JVM flags (per [[025_software_architecture_document|SAD]]):**

| Flag | Value | Why |
|------|-------|-----|
| `-Xms` | `128m` | Initial heap |
| `-Xmx` | `192m` | Max heap (256 MB total allocation per SAD) |
| `-XX:+UseZGC` | — | Z Garbage Collector — low-latency, Java 25 default |

## 6. Docker Compose Fragment

The `docker-compose.fragment.yml` is designed to be copied into the platform's main `docker-compose.yml`:

```yaml
flowero-discover:
  build:
    context: ./flowerodiscovery
    dockerfile: Dockerfile
  container_name: flowero-discover
  ports:
    - "8999:8999"   # BE: REST API
    - "3999:8999"    # FE: Dashboard (maps same Eureka server)
  networks:
    - panomete-net
  healthcheck:
    test: ["CMD", "curl", "-f", "http://localhost:8999/actuator/health"]
    interval: 30s
    timeout: 5s
    retries: 3
    start_period: 30s
  restart: unless-stopped
```

## 7. Build Output Verification

### Successful build output (2026-07-23):

```
$ ./gradlew clean build --no-daemon

BUILD SUCCESSFUL in 23s
8 actionable tasks: 8 executed

/buid/libs/
├── flowerodiscovery-0.0.1-SNAPSHOT.jar       (59 MB — fat JAR)
└── flowerodiscovery-0.0.1-SNAPSHOT-plain.jar  (2 KB — skinny JAR)
```

### Test output:

```
tests="6" failures="0" errors="0"
  ✓ eurekaDashboardIsReachable()
  ✓ eurekaAppsEndpointReturnsEmptyRegistry()
  ✓ actuatorHealthReturnsUp()
  ✓ doesNotRegisterWithItself()
  ✓ contextLoads()
  ✓ eurekaAppsEndpointReturnsJsonWithAcceptHeader()
```

---

## Related Documents

| Document | Relationship |
|----------|-------------|
| [[031_README_developer_guide]] | Developer setup instructions |
| [[033_dependency_manifest]] | Dependencies being built |
| [[034_commit_messages_changelog]] | Commit standards |
| [[052_deployment_plan]] | Deployment procedures |

---

> **Template Standard:** Based on SWEBOK v4, 12-Factor App
> **Usage:** All build commands verified working on 2026-07-23. The Gradle wrapper is the entry point — never use system Gradle.
