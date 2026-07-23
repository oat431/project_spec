---
document_type: Build Scripts
version: "2.0"
status: Draft
author: "[Technical Lead]"
created: "[YYYY-MM-DD]"
last_updated: "[YYYY-MM-DD]"
project_name: "[Project Name]"
project_id: "[Project-ID]"
classification: "Internal / Confidential"
tags: [build-scripts, ci-cd, automation, gradle, docker, swebok]
standard_ref:
  - SWEBOK v4 — Construction
  - 12-Factor App (Build, release, run)
---

# Build Scripts

> **Project:** [Project Name]
> **Version:** [X.Y] | **Status:** [Active]
> **Last Updated:** [YYYY-MM-DD]

---

## 1. Purpose

> Documents the build pipeline — scripts, stages, and automation that transform source code into deployable artifacts. The build pipeline IS the quality gate. If it's not in the pipeline, it doesn't happen.

## 2. Build Pipeline (Universal)

```mermaid
flowchart LR
    CODE[Source Code] --> LINT[Lint / Format]
    LINT --> COMPILE[Compile / Type-Check]
    COMPILE --> TEST[Test]
    TEST --> PACKAGE[Package Artifact]
    PACKAGE --> IMAGE[Docker Image]
    IMAGE --> PUSH[Push to Registry]
    PUSH --> DEPLOY[Deploy]

    style CODE fill:#2196F3,color:#fff
    style LINT fill:#FF9800,color:#fff
    style COMPILE fill:#FF9800,color:#fff
    style TEST fill:#4CAF50,color:#fff
    style PACKAGE fill:#9C27B0,color:#fff
    style IMAGE fill:#607D8B,color:#fff
    style DEPLOY fill:#4CAF50,color:#fff
```

## 3. Tech-Stack Variants

### 3.1 Java / Gradle / Spring Boot (Flowero Discover, Gate, Guard)

| Script | Command | Purpose |
|--------|---------|---------|
| Build (full) | `./gradlew build` | Compile + test + package bootJar |
| Build (skip tests) | `./gradlew build -x test` | Quick assemble during development |
| Compile only | `./gradlew compileJava` | Syntax check |
| Test | `./gradlew test` | Run all tests (JUnit 5) |
| Package | `./gradlew bootJar` | Create fat JAR (includes embedded server) |
| Run locally | `./gradlew bootRun` | Start with embedded server |
| Clean | `./gradlew clean` | Remove build artifacts |
| Dependency check | `./gradlew dependencies` | Print dependency tree |

**Gradle Wrapper:** Always use `./gradlew` (checked into VCS). Never rely on system Gradle.

```bash
# Generate wrapper (once per project)
gradle wrapper --gradle-version 9.5.1
```

**Key Files:**

| File | Purpose |
|------|---------|
| `build.gradle` | Dependencies, plugins, tasks |
| `settings.gradle` | Project name, module structure |
| `gradle/wrapper/gradle-wrapper.jar` | Wrapper binary (committed) |
| `gradle/wrapper/gradle-wrapper.properties` | Wrapper version pin |

### 3.2 Node.js / TypeScript / npm (Tiny Mchwa, Fluffy Mouton)

| Script | Command | Purpose |
|--------|---------|---------|
| Dev | `npm run dev` | Start development server with hot reload |
| Build | `npm run build` | Compile TypeScript → JavaScript |
| Test | `npm test` | Run all tests (Jest / Vitest) |
| Test:unit | `npm run test:unit` | Unit tests only |
| Test:integration | `npm run test:integration` | Integration tests |
| Lint | `npm run lint` | Run ESLint |
| Format | `npm run format` | Run Prettier |
| Type-check | `npm run type-check` | Run `tsc --noEmit` |
| Docker:build | `npm run docker:build` | Build Docker image |

**Key Files:**

| File | Purpose |
|------|---------|
| `package.json` | Dependencies + scripts |
| `package-lock.json` | Locked versions (committed) |
| `tsconfig.json` | TypeScript configuration |

### 3.3 Go (Cute Gufo)

| Script | Command | Purpose |
|--------|---------|---------|
| Build | `go build -o bin/app ./cmd/app` | Compile binary |
| Test | `go test ./...` | Run all tests |
| Test:verbose | `go test -v ./...` | Detailed output |
| Test:coverage | `go test -cover ./...` | Coverage report |
| Lint | `golangci-lint run` | Run linter |
| Format | `gofmt -w .` | Auto-format |
| Mod tidy | `go mod tidy` | Clean up dependencies |

**Key Files:**

| File | Purpose |
|------|---------|
| `go.mod` | Module path + dependencies |
| `go.sum` | Dependency checksums (committed) |

## 4. Docker Multi-Stage Build

Universal pattern — separate build and runtime stages:

```dockerfile
# ---- Stage 1: Build ----
FROM eclipse-temurin:25-jdk-noble AS build    # Java example
# FROM node:20-alpine AS build                 # Node.js example
# FROM golang:1.23-alpine AS build             # Go example
WORKDIR /app

# Copy dependency manifests first (cache layer)
COPY build.gradle settings.gradle gradlew ./
COPY gradle/ gradle/
RUN ./gradlew dependencies --no-daemon -q || true

# Copy source and build
COPY src/ src/
RUN ./gradlew bootJar --no-daemon

# ---- Stage 2: Runtime ----
FROM eclipse-temurin:25-jre-noble
# FROM node:20-alpine                        # Node.js
# FROM alpine:3.20                           # Go (scratch/distroless also works)
WORKDIR /app

# Create non-root user
RUN groupadd -r app && useradd -r -g app app

COPY --from=build /app/build/libs/*.jar app.jar
EXPOSE 8080
USER app

ENTRYPOINT ["java", "-Xms128m", "-Xmx192m", "-jar", "app.jar"]
```

**Key principles:**
- Dependency manifests are copied first → cached unless they change
- Source is copied last → fast incremental builds
- Non-root user in runtime stage
- Explicit JVM flags (`-Xms`, `-Xmx`)
- Multi-stage → final image is small (no JDK, no build tools)

## 5. CI/CD Pipeline (GitHub Actions — Multi-Stack)

```yaml
name: CI/CD

on:
  push:
    branches: [main, dev]
  pull_request:
    branches: [main]

jobs:
  # ---- Java Service ----
  build-java:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with:
          java-version: '25'
          distribution: 'temurin'
      - run: ./gradlew build --no-daemon
      - run: docker build -t ${{ github.repository }}:${{ github.sha }} .

  # ---- Node.js Service ----
  build-node:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
      - run: npm ci
      - run: npm run lint
      - run: npm test -- --coverage

  # ---- Go Service ----
  build-go:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-go@v5
        with:
          go-version: '1.23'
      - run: go test ./...
      - run: go build -o bin/app ./cmd/app
```

## 6. Build Artifacts

| Artifact | Format | Location | Retention |
|---------|--------|---------|----------|
| Fat JAR (Spring Boot) | `.jar` | `build/libs/` | Per build |
| Plain JAR | `.jar` | `build/libs/` | Per build |
| Docker Image | Docker | Container Registry | Tagged by SHA |
| Test Report | HTML/JUnit XML | `build/reports/tests/` | CI artifacts |
| Coverage Report | HTML/XML | `build/reports/coverage/` | CI artifacts |
| Go Binary | Executable | `bin/` or `dist/` | Per release |

---

## Related Documents

| Document | Relationship |
|----------|-------------|
| [[031_README_developer_guide]] | Build instructions for developers |
| [[033_dependency_manifest]] | Dependencies being built |
| [[034_commit_messages_changelog]] | Commit standards |
| [[052_deployment_plan]] | What happens after build |

---

> **Template Standard:** Based on SWEBOK v4, 12-Factor App (Build, Release, Run)
> **Usage:** Pick the variant matching your tech stack. The pipeline stages are universal — lint → compile → test → package → image → deploy. Automate everything.
