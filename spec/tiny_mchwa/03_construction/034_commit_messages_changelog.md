---
document_type: Commit Messages & Changelog
version: "1.1"
status: Active
author: "Panomete"
created: "2026-07-04"
last_updated: "2026-07-05"
project_name: "tiny mchwa"
project_id: "tiny-mchwa"
classification: "Internal"
tags: [changelog, commits, homelab]
---

# Commit Messages & Changelog

> **Project:** tiny mchwa 🐜
> **Version:** 1.1 | **Status:** Active
> **Last Updated:** 2026-07-05

---

## Commit Convention

```
type(scope): description

feat(api): add todolist CRUD
fix(db): resolve connection pool leak
test(handler): add todolist handler tests
docs(spec): update API specification
```

---

## Changelog

### 2026-07-04 — MVP Launch 🐜

#### API (tiny-mchwa-api)

**PRs Merged:**
- #1 feat: mvp polish (validation, CORS, lint, Air) — merged
- #6 feat: tasks CRUD + computed status — merged
- #7 feat: add tests + refactor services to interfaces — merged

**Features:**
- ✅ Todolists CRUD (create, list, get, update, delete)
- ✅ Tasks CRUD (create, list, get, update, delete)
- ✅ Pagination + filtering (status, sourceService, title)
- ✅ Computed todolist status from tasks
- ✅ Standardized `{data, error, meta}` response format
- ✅ Middleware: RequestID, Recover, Helmet, CORS, Logger
- ✅ Struct validation (go-playground/validator)
- ✅ Graceful shutdown
- ✅ godotenv for .env loading
- ✅ 12 handler tests passing (mock-based)
- ✅ Services refactored to interfaces for testability

**Issues Pending:**
- #2 production readiness (migrations, Dockerfile, health check)
- #5 auth + observability (JWT, rate limiter, Swagger, OTel)

#### Web (tiny-mchwa-web)

**PRs Merged:**
- #6 404 page, error handling, page titles — merged
- #7 URL params for pagination + filters — merged
- #8 TanStack Query for data fetching — merged

**Features:**
- ✅ Todolists page (list, create, edit, delete, pagination)
- ✅ Todolist detail page (tasks list, create, edit, delete, status toggle)
- ✅ TanStack Query for data fetching
- ✅ URL params for pagination + filters
- ✅ 404 page, ErrorBoundary, page titles
- ✅ Niramit font
- ✅ Vite proxy to API at localhost:8005

**Issues Pending:**
- #1 Vitest + React Testing Library tests
- #4 accessibility improvements

---

## Related Documents

| Document | Relationship |
|----------|-------------|
| [[053_release_notes]] | Release notes |
| [[072_meeting_minutes]] | Daily logs |
