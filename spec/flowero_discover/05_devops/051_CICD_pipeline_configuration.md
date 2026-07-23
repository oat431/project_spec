---
document_type: CI/CD Pipeline Configuration
version: "0.1"
status: Draft
author: "DevOps Persona"
created: "2026-07-23"
last_updated: "2026-07-23"
project_name: "Flowero Discover"
project_id: "PAN-DISCOVER-001"
parent_platform: "Panomete Platform"
classification: "Internal"
tags: [ci-cd, pipeline, github-actions, eureka, devops, panomete]
standard_ref:
  - SWEBOK v4 — Operations
---

# CI/CD Pipeline Configuration — Flowero Discover

> **Service:** Flowero Discover (Spring Cloud Netflix Eureka)
> **Platform:** Panomete Platform
> **Version:** 0.1 | **Status:** Draft — Awaiting PO Review
> **Last Updated:** 2026-07-23

---

## 1. Purpose

> Defines the CI/CD pipeline for Flowero Discover. Discover is a standard Java/Spring Boot application — the pipeline compiles, tests, builds a JAR, packages as Docker image, and deploys to the homelab.

---

## 2. Service Profile

| Aspect | Detail |
|--------|--------|
| Language | Java 21 |
| Framework | Spring Boot 3.x + Spring Cloud Netflix Eureka Server |
| Build Tool | Maven (Maven Wrapper `mvnw`) |
| Artifact | Fat JAR → Docker image |
| Dependencies | None at runtime (standalone Eureka server) |
| Ports | 8999 (REST API), 3999 (Dashboard) |
| Resource Profile | Lightweight — 0.25 vCPU, 256 MB heap |

---

## 3. Repository Structure

```
flowero-discover/
├── src/
│   ├── main/
│   │   ├── java/           # Eureka server application
│   │   └── resources/
│   │       └── application.yml
│   └── test/
│       └── java/           # Unit tests
├── Dockerfile
├── pom.xml
└── mvnw / mvnw.cmd
```

---

## 4. Pipeline Configuration

### 4.1 CI — Compile + Test (runs on PRs and main)

```yaml
# .github/workflows/discover-ci.yml
name: Discover CI

on:
  push:
    branches: [main, develop]
    paths: ['flowero-discover/**']
  pull_request:
    branches: [main]
    paths: ['flowero-discover/**']

jobs:
  build-test:
    name: Compile + Test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Set up JDK 21
        uses: actions/setup-java@v4
        with:
          java-version: '21'
          distribution: 'temurin'
          cache: maven

      - name: Compile
        run: cd flowero-discover && ./mvnw compile -q

      - name: Lint (Checkstyle)
        run: cd flowero-discover && ./mvnw checkstyle:check -q

      - name: Lint (SpotBugs)
        run: cd flowero-discover && ./mvnw spotbugs:check -q

      - name: Unit Tests
        run: cd flowero-discover && ./mvnw test -q

      - name: Package (skip tests — already ran)
        run: cd flowero-discover && ./mvnw package -DskipTests -q

      - name: Upload JAR artifact
        uses: actions/upload-artifact@v4
        with:
          name: discover-jar
          path: flowero-discover/target/*.jar
```

### 4.2 CD — Build & Deploy Discover

```yaml
# .github/workflows/discover-deploy.yml
name: Discover Build & Deploy

on:
  push:
    branches: [main]
    paths: ['flowero-discover/**']

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: panomete/flowero-discover

jobs:
  build:
    name: Build Docker Image
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Set up JDK 21
        uses: actions/setup-java@v4
        with:
          java-version: '21'
          distribution: 'temurin'
          cache: maven

      - name: Build JAR
        run: cd flowero-discover && ./mvnw package -DskipTests -q

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Log in to GHCR
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Build & Push
        uses: docker/build-push-action@v5
        with:
          context: flowero-discover
          file: flowero-discover/Dockerfile
          push: true
          tags: |
            ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:latest
            ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }}

  deploy:
    name: Deploy Discover to Homelab
    needs: build
    runs-on: ubuntu-latest
    environment: homelab
    steps:
      - uses: actions/checkout@v4

      - name: Deploy via SSH
        uses: appleboy/ssh-action@v1
        with:
          host: remote.panomete.com
          username: flowero
          key: ${{ secrets.HOMELAB_SSH_KEY }}
          script: |
            cd ~/platform
            docker compose -f docker-compose.platform.yml pull flowero-discover
            docker compose -f docker-compose.platform.yml up -d flowero-discover
            sleep 10

      - name: Smoke Test
        run: |
          # Discover health check
          curl -sf http://remote.panomete.com:8999/actuator/health || exit 1
          # Dashboard accessible via Nginx
          curl -sf -o /dev/null -w '%{http_code}' https://discovery.panomete.com/ || exit 1

      - name: Rollback on Failure
        if: failure()
        uses: appleboy/ssh-action@v1
        with:
          host: remote.panomete.com
          username: flowero
          key: ${{ secrets.HOMELAB_SSH_KEY }}
          script: |
            cd ~/platform
            docker compose -f docker-compose.platform.yml stop flowero-discover
            docker compose -f docker-compose.platform.yml up -d flowero-discover --pull never
```

---

## 5. Pipeline Stages

| Stage | Purpose | Duration (est.) | Failure Action |
|-------|---------|:---:|----------------|
| Compile | Java 21 compilation | < 30 sec | Block merge |
| Checkstyle | Code style enforcement | < 20 sec | Block merge |
| SpotBugs | Static analysis | < 30 sec | Block merge |
| Unit Test | JUnit + Mockito | < 1 min | Block merge |
| Package | Build fat JAR | < 30 sec | Block deploy |
| Docker Build | Multi-stage build | < 2 min | Block deploy |
| Deploy | SSH + docker compose | < 1 min | Alert |
| Smoke Test | Health + dashboard | < 10 sec | Auto-rollback |

---

## 6. Discover Dockerfile

```dockerfile
# flowero-discover/Dockerfile
FROM maven:3.9-eclipse-temurin-21 AS builder
WORKDIR /build
COPY pom.xml .
RUN mvn dependency:go-offline -q
COPY src ./src
RUN mvn package -DskipTests -q

FROM eclipse-temurin:21-jre-alpine
WORKDIR /app
COPY --from=builder /build/target/*.jar app.jar

EXPOSE 8999 3999

ENTRYPOINT ["java", \
  "-XX:MaxRAMPercentage=75", \
  "-Xmx192m", \
  "-jar", "app.jar"]
```

---

## 7. Configuration Reference

```yaml
# flowero-discover/src/main/resources/application.yml
server:
  port: 8999   # REST API for service registration

eureka:
  instance:
    hostname: flowero-discover
  client:
    register-with-eureka: false    # Standalone — don't register self
    fetch-registry: false          # Standalone — don't fetch registry
  server:
    enable-self-preservation: true # Keep evicted services during network partitions
    eviction-interval-timer-in-ms: 5000

spring:
  application:
    name: flowero-discover

management:
  endpoints:
    web:
      exposure:
        include: health,info
```

---

## 8. Secrets Management

> Discover has **no secrets**. It is a standalone Eureka server with no database, no external dependencies, and no authentication on the dashboard (trusted internal network per ADR-D003).

| Secret | Needed? | Notes |
|--------|:---:|-------|
| Database credentials | ❌ | No database — fully in-memory |
| OAuth2 client secret | ❌ | Not behind Gate — direct Nginx access |
| API keys | ❌ | No external service calls |
| `HOMELAB_SSH_KEY` | ✅ | GitHub Actions deploy job only |

---

## 9. Quality Gates

A pull request changing `flowero-discover/**` cannot merge unless:

- [ ] Compiles without errors (Java 21)
- [ ] Checkstyle passes (zero violations)
- [ ] SpotBugs passes (zero warnings)
- [ ] All unit tests pass
- [ ] Code review approved (when team grows)

A deployment of Discover cannot proceed unless:

- [ ] CI pipeline passed on `main`
- [ ] Docker image built and pushed to GHCR
- [ ] Port 8999 and 3999 are free
- [ ] Smoke test passes (health + dashboard)

---

## 10. Local Testing

```bash
cd flowero-discover

# Run locally
./mvnw spring-boot:run

# Health check
curl http://localhost:8999/actuator/health

# Test registration (simulate a service registering)
curl -X POST http://localhost:8999/eureka/apps/TEST-SERVICE \
  -H "Content-Type: application/json" \
  -d '{
    "instance": {
      "instanceId": "test-service:8080",
      "hostName": "test-service",
      "app": "TEST-SERVICE",
      "ipAddr": "127.0.0.1",
      "status": "UP",
      "port": {"$": 8080, "@enabled": "true"}
    }
  }'

# Verify registration
curl -H "Accept: application/json" http://localhost:8999/eureka/apps | jq .
```

---

## Related Documents

| Document | Relationship |
|----------|-------------|
| `panomete_platform/05_devops/051_CICD_pipeline_configuration` | Platform-level pipeline |
| [[052_deployment_plan]] | Discover-specific deployment |
| `flowero_discover/02_design/022_API_specification` | Eureka REST API reference |
| `flowero_discover/02_design/021_architecture_decision_records` | Discover ADRs |

---

> **Template Standard:** Based on SWEBOK v4
> **Usage:** The pipeline is *the single path to production*. No hotfixes bypass the pipeline.
