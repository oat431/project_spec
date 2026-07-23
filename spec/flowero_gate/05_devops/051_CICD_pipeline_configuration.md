---
document_type: CI/CD Pipeline Configuration
version: "0.1"
status: Draft
author: "DevOps Persona"
created: "2026-07-23"
last_updated: "2026-07-23"
project_name: "Flowero Gate"
project_id: "PAN-GATE-001"
parent_platform: "Panomete Platform"
classification: "Internal"
tags: [ci-cd, pipeline, github-actions, spring-cloud-gateway, devops, panomete]
standard_ref:
  - SWEBOK v4 — Operations
---

# CI/CD Pipeline Configuration — Flowero Gate

> **Service:** Flowero Gate (Spring Cloud Gateway)
> **Platform:** Panomete Platform
> **Version:** 0.1 | **Status:** Draft — Awaiting PO Review
> **Last Updated:** 2026-07-23

---

## 1. Purpose

> Defines the CI/CD pipeline for Flowero Gate. Gate is a Java/Spring Boot reactive application (WebFlux/Netty). The pipeline compiles, tests, builds a JAR, packages as Docker image, and deploys to the homelab.

---

## 2. Service Profile

| Aspect | Detail |
|--------|--------|
| Language | Java 21 |
| Framework | Spring Boot 3.x + Spring Cloud Gateway (Reactive/WebFlux) |
| Build Tool | Maven (Maven Wrapper `mvnw`) |
| Artifact | Fat JAR → Docker image |
| Runtime Dependencies | Guard (JWKS), Discover (Eureka), Valkey (rate limiting) |
| Port | 8000 |
| Resource Profile | Medium — 0.5 vCPU, 512 MB heap |

---

## 3. Repository Structure

```
flowero-gate/
├── src/
│   ├── main/
│   │   ├── java/               # Gateway application, filters, security config
│   │   └── resources/
│   │       └── application.yml # Route config, Redis config, OAuth2 config
│   └── test/
│       └── java/               # Unit tests, route tests, JWT validation tests
├── Dockerfile
├── pom.xml
└── mvnw / mvnw.cmd
```

---

## 4. Pipeline Configuration

### 4.1 CI — Compile + Test (runs on PRs and main)

```yaml
# .github/workflows/gate-ci.yml
name: Gate CI

on:
  push:
    branches: [main, develop]
    paths: ['flowero-gate/**']
  pull_request:
    branches: [main]
    paths: ['flowero-gate/**']

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
        run: cd flowero-gate && ./mvnw compile -q

      - name: Lint (Checkstyle)
        run: cd flowero-gate && ./mvnw checkstyle:check -q

      - name: Lint (SpotBugs)
        run: cd flowero-gate && ./mvnw spotbugs:check -q

      - name: Unit Tests
        run: cd flowero-gate && ./mvnw test -q

      - name: Package
        run: cd flowero-gate && ./mvnw package -DskipTests -q

      - name: Upload JAR artifact
        uses: actions/upload-artifact@v4
        with:
          name: gate-jar
          path: flowero-gate/target/*.jar
```

### 4.2 CD — Build & Deploy Gate

```yaml
# .github/workflows/gate-deploy.yml
name: Gate Build & Deploy

on:
  push:
    branches: [main]
    paths: ['flowero-gate/**']

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: panomete/flowero-gate

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
        run: cd flowero-gate && ./mvnw package -DskipTests -q

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
          context: flowero-gate
          file: flowero-gate/Dockerfile
          push: true
          tags: |
            ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:latest
            ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }}

  deploy:
    name: Deploy Gate to Homelab
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
            docker compose -f docker-compose.platform.yml pull flowero-gate
            docker compose -f docker-compose.platform.yml up -d flowero-gate
            sleep 10

      - name: Smoke Test
        run: |
          # Gate health check
          curl -sf http://remote.panomete.com:8000/actuator/health || exit 1
          # Gate should reject unauthenticated requests
          CODE=$(curl -sf -o /dev/null -w '%{http_code}' https://api.panomete.com/api/blog/posts)
          if [ "$CODE" != "401" ]; then
            echo "❌ Expected 401 for no-token request, got $CODE"
            exit 1
          fi
          echo "✅ Gate correctly rejects unauthenticated requests"

      - name: Rollback on Failure
        if: failure()
        uses: appleboy/ssh-action@v1
        with:
          host: remote.panomete.com
          username: flowero
          key: ${{ secrets.HOMELAB_SSH_KEY }}
          script: |
            cd ~/platform
            docker compose -f docker-compose.platform.yml stop flowero-gate
            docker compose -f docker-compose.platform.yml up -d flowero-gate --pull never
```

---

## 5. Pipeline Stages

| Stage | Purpose | Duration (est.) | Failure Action |
|-------|---------|:---:|----------------|
| Compile | Java 21 compilation | < 30 sec | Block merge |
| Checkstyle | Code style enforcement | < 20 sec | Block merge |
| SpotBugs | Static analysis | < 30 sec | Block merge |
| Unit Test | JUnit + Mockito (route tests, JWT tests) | < 2 min | Block merge |
| Package | Build fat JAR | < 30 sec | Block deploy |
| Docker Build | Multi-stage build | < 2 min | Block deploy |
| Deploy | SSH + docker compose | < 1 min | Alert |
| Smoke Test | Health + 401 rejection | < 10 sec | Auto-rollback |

---

## 6. Gate Dockerfile

```dockerfile
# flowero-gate/Dockerfile
FROM maven:3.9-eclipse-temurin-21 AS builder
WORKDIR /build
COPY pom.xml .
RUN mvn dependency:go-offline -q
COPY src ./src
RUN mvn package -DskipTests -q

FROM eclipse-temurin:21-jre-alpine
WORKDIR /app
COPY --from=builder /build/target/*.jar app.jar

EXPOSE 8000

ENTRYPOINT ["java", \
  "-XX:MaxRAMPercentage=75", \
  "-Xmx384m", \
  "-jar", "app.jar"]
```

---

## 7. Configuration Reference

```yaml
# flowero-gate/src/main/resources/application.yml
server:
  port: 8000

spring:
  application:
    name: flowero-gate
  cloud:
    gateway:
      routes:
        - id: blog
          uri: lb://cute-gufo
          predicates: [Path=/api/blog/**]
          filters:
            - StripPrefix=1
            - name: RequestRateLimiter
              args:
                redis-rate-limiter.replenishRate: 100
                redis-rate-limiter.burstCapacity: 200
        # ... additional routes
    loadbalancer:
      eureka:
        enabled: true
  security:
    oauth2:
      resourceserver:
        jwt:
          issuer-uri: https://auth.panomete.com/realms/panomete
          jwk-set-uri: https://auth.panomete.com/realms/panomete/protocol/openid-connect/certs
  data:
    redis:
      host: ${VALKEY_HOST:local-valkey}
      port: ${VALKEY_PORT:6379}
      password: ${VALKEY_PASSWORD}

eureka:
  client:
    service-url:
      defaultZone: http://flowero-discover:8999/eureka/

logging:
  pattern:
    console: '{"timestamp":"%d{ISO8601}","level":"%p","message":"%m"}%n'
```

> **Critical:** Valkey host is `local-valkey` (container name on `db-network`), NOT `valkey` or `host.docker.internal`. Password is required. See MM04 DEC-003.

---

## 8. Secrets Management

| Secret | Location | Purpose | Rotation |
|--------|----------|---------|----------|
| `VALKEY_PASSWORD` | Homelab `~/platform/.env` | Rate limiting backend auth | On change |
| `HOMELAB_SSH_KEY` | GitHub Actions secrets | Deploy job SSH access | On team change |

> **Note:** Gate does NOT have its own OAuth2 client secret. It validates JWTs using Guard's public JWKS — no shared secret needed. Gate is a **resource server**, not a **client**.

---

## 9. Quality Gates

A pull request changing `flowero-gate/**` cannot merge unless:

- [ ] Compiles without errors (Java 21)
- [ ] Checkstyle passes (zero violations)
- [ ] SpotBugs passes (zero warnings)
- [ ] All unit tests pass (including route config tests and JWT validation tests)
- [ ] Code review approved (when team grows)

A deployment of Gate cannot proceed unless:

- [ ] CI pipeline passed on `main`
- [ ] Docker image built and pushed to GHCR
- [ ] Guard is healthy (JWKS must be reachable)
- [ ] Discover is healthy (Eureka registry must be available)
- [ ] Valkey is healthy (rate limiting backend)
- [ ] Smoke test passes (health + 401 rejection)
- [ ] Rollback plan is verified

---

## 10. Local Testing

```bash
cd flowero-gate

# Run locally (requires Guard + Discover + Valkey running)
./mvnw spring-boot:run

# Health check
curl http://localhost:8000/actuator/health

# Test 401 (no token)
curl -sf -o /dev/null -w '%{http_code}' http://localhost:8000/api/blog/posts
# Expected: 401

# Test with dummy JWT (for route testing without Keycloak)
TOKEN="eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCIs..." # Valid JWT from Keycloak
curl -H "Authorization: Bearer $TOKEN" http://localhost:8000/api/blog/posts
# Expected: proxied to backend (or 502 if backend down)
```

---

## 11. Security Scan (Phase 2)

> When observability lands in Phase 2, add Trivy image scanning to the pipeline:

```yaml
  security-scan:
    name: Security Scan
    needs: build
    runs-on: ubuntu-latest
    steps:
      - name: Run Trivy vulnerability scanner
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:latest
          format: 'table'
          exit-code: '1'    # Fail on critical
          severity: 'CRITICAL,HIGH'
```

---

## Related Documents

| Document | Relationship |
|----------|-------------|
| `panomete_platform/05_devops/051_CICD_pipeline_configuration` | Platform-level pipeline |
| [[052_deployment_plan]] | Gate-specific deployment procedure |
| `flowero_gate/02_design/022_API_specification` | Route table, auth behavior, error codes |
| `flowero_gate/02_design/021_architecture_decision_records` | Gate ADRs (JWT, rate limiting, no TLS) |
| `flowero_guard/02_design/022_API_specification` | JWKS endpoint Gate depends on |
| `flowero_discover/02_design/022_API_specification` | Eureka endpoints Gate depends on |

---

> **Template Standard:** Based on SWEBOK v4
> **Usage:** The pipeline is *the single path to production*. No hotfixes bypass the pipeline.
