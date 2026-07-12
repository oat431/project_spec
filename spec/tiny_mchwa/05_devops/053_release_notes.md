---
document_type: Release Notes
version: "1.1"
status: Active
author: "Panomete"
created: "2026-07-04"
last_updated: "2026-07-05"
project_name: "tiny mchwa"
project_id: "tiny-mchwa"
classification: "Internal"
tags: [release-notes, homelab]
---

# Release Notes

> **Project:** tiny mchwa 🐜
> **Version:** 1.1 | **Status:** Active

---

## v1.0.0 — 2026-07-04 (MVP) 🐜

### Repositories

| Repo | URL | Branch |
|------|-----|--------|
| API | https://github.com/oat431/tiny-mchwa-api | main |
| Web | https://github.com/oat431/tiny-mchwa-web | main |

### Features

**API:**
- Todolists CRUD (create, list, get, update, delete)
- Tasks CRUD (create, list, get, update, delete)
- Pagination (page, perPage params)
- Filtering (status, sourceService, title)
- Computed todolist status from tasks
- Standardized `{data, error, meta}` response format
- Middleware: RequestID, Recover, Helmet, CORS, Logger
- Struct validation (go-playground/validator)
- Graceful shutdown
- Environment config via .env
- 12 handler tests passing

**Web:**
- Todolists page (list, create, edit, delete, pagination)
- Todolist detail page (tasks list, create, edit, delete, status toggle)
- TanStack Query for data fetching
- URL params for pagination + filters
- 404 page, ErrorBoundary, page titles
- Niramit font
- DaisyUI + Tailwind CSS styling

### Tech Stack

| Component | Technology | Version |
|-----------|-----------|---------|
| API Language | Go | 1.25.3 |
| API Framework | Fiber | v3.4.0 |
| Database | PostgreSQL | 15+ |
| DB Driver | sqlx + lib/pq | 1.4.0 |
| Web Framework | React | 19.2.7 |
| Web Language | TypeScript | 6.0.3 |
| UI Library | DaisyUI | 5.6.13 |
| Data Fetching | TanStack Query | 5.101.2 |
| Build Tool | Vite | 8.1.3 |

### Known Issues

- Auth: hardcoded UUID (Keycloak integration pending)
- Web: no tests yet
- API: no Dockerfile yet
- API: no health check endpoint

### Pending Work

**API:**
- #2 production readiness (migrations, Dockerfile, health check)
- #5 auth + observability (JWT, rate limiter, Swagger, OTel)

**Web:**
- #1 Vitest + React Testing Library tests
- #4 accessibility improvements

---

## Related Documents

| Document | Relationship |
|----------|-------------|
| [[034_commit_messages_changelog]] | Detailed changelog |
| [[072_meeting_minutes]] | Daily logs |
