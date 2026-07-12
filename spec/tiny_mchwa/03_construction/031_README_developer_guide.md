---
document_type: Developer Guide
version: "1.1"
status: Approved
author: "Panomete"
created: "2026-07-04"
last_updated: "2026-07-05"
project_name: "tiny mchwa"
project_id: "tiny-mchwa"
classification: "Internal"
tags: [developer-guide, setup, homelab]
standard_ref:
  - SWEBOK v4 — Construction
---

# Developer Guide

> **Project:** tiny mchwa 🐜
> **Version:** 1.1 | **Status:** Approved
> **Last Updated:** 2026-07-05

---

## 1. Prerequisites

### API

| Tool | Version | Purpose |
|------|---------|---------|
| Go | 1.25.3+ | Backend language |
| PostgreSQL | 15+ | Database |
| Air | latest | Hot reload |
| golangci-lint | latest | Linting |

### Web

| Tool | Version | Purpose |
|------|---------|---------|
| Bun | latest | Package manager + runtime |
| Node.js | 18+ | Fallback for tooling |

---

## 2. Setup

### API

```bash
# Clone repository
git clone https://github.com/oat431/tiny-mchwa-api.git
cd tiny-mchwa-api

# Install dependencies
go mod tidy

# Setup environment
cp .env.example .env
# Edit .env with your database credentials

# Run with hot reload
air

# Or run directly
go run cmd/server/main.go
```

### Web

```bash
# Clone repository
git clone https://github.com/oat431/tiny-mchwa-web.git
cd tiny-mchwa-web

# Install dependencies
bun install

# Start development server
bun dev
```

---

## 3. Environment Variables (API)

| Variable | Default | Description |
|----------|---------|-------------|
| DB_HOST | localhost | PostgreSQL host |
| DB_PORT | 5432 | PostgreSQL port |
| DB_USER | postgres | Database user |
| DB_PASSWORD | — | Database password |
| DB_NAME | tiny_mchwa | Database name |
| SERVER_PORT | 8005 | API server port |

---

## 4. Running Tests

### API

```bash
# Run all tests
go test ./...

# Run with coverage
go test -cover ./...

# Run specific package
go test ./internal/handler/...
```

### Web

```bash
# Run tests (when configured)
bun test
```

---

## 5. Ports

| Service | Port | URL |
|---------|------|-----|
| API | 8005 | http://localhost:8005 |
| Web | 3300 | http://localhost:3300 |
| PostgreSQL | 5432 | localhost:5432 |

---

## Related Documents

| Document | Relationship |
|----------|-------------|
| [[033_dependency_manifest]] | Full dependency list |
| [[035_coding_standards_development]] | Code style rules |
