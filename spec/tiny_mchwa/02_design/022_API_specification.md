---
document_type: API Specification
version: "1.0"
status: Approved
author: "OraMesLita (AI)"
created: "2026-07-01"
last_updated: "2026-07-04"
project_name: "tiny mchwa"
project_id: "tiny-mchwa"
tech_lead: "Panomete"
classification: "Internal"
tags: [api-specification, rest, homelab]
standard_ref:
  - SWEBOK v4 — Design
  - OpenAPI Specification 3.0
---

# API Specification

> **Project:** tiny mchwa 🐜
> **Version:** 1.0 | **Status:** Approved
> **Last Updated:** 2026-07-04

---

## 1. API Overview

| Field | Detail |
|-------|--------|
| Base URL | `/api/v1` |
| Protocol | HTTP (HTTPS via gateway) |
| Format | JSON |
| Authentication | Bearer JWT token (Keycloak) |
| Gateway | Flowero Gate |

---

## 2. Response Format

### Success

```json
{
    "data": { ... },
    "error": null,
    "meta": null
}
```

### Error

```json
{
    "data": null,
    "error": {
        "code": 400,
        "message": "Validation failed",
        "details": ["title is required"]
    },
    "meta": null
}
```

### Pagination Meta

```json
{
    "meta": {
        "page": 1,
        "perPage": 20,
        "total": 150,
        "totalPages": 8
    }
}
```

---

## 3. Endpoints — Todolists

### POST /api/v1/todolists — Create Todolist

| Field | Detail |
|-------|--------|
| Auth | JWT required |
| Rate Limit | 100/min |

**Request:**
```json
{
    "title": "Grocery List",
    "description": "Weekly groceries",
    "sourceService": "todolist"
}
```

**Response (201):**
```json
{
    "data": {
        "id": "uuid",
        "title": "Grocery List",
        "description": "Weekly groceries",
        "status": "pending",
        "createdAt": "2026-07-01T10:00:00Z",
        "updatedAt": "2026-07-01T10:00:00Z",
        "ownedBy": "keycloak-uuid",
        "sourceService": "todolist"
    },
    "error": null,
    "meta": null
}
```

**Validation:**

| Field | Rule | Error |
|-------|------|-------|
| title | Required, max 255 chars | 400 |
| description | Optional, max 1000 chars | 400 |
| sourceService | Required, string | 400 |

---

### GET /api/v1/todolists — List Todolists

**Query Parameters:**

| Param | Type | Default | Description |
|-------|------|---------|-------------|
| page | number | 1 | Page number |
| perPage | number | 20 | Items per page |
| status | string | all | Filter by computed status |
| sourceService | string | all | Filter by source service |
| title | string | — | Filter by title (partial match) |

**Response (200):** Array of todolists with pagination meta.

---

### GET /api/v1/todolists/:id — Get Todolist

**Response (200):** Single todolist object.

**Error (404):**
```json
{
    "data": null,
    "error": { "code": 404, "message": "Todolist not found" },
    "meta": null
}
```

---

### PUT /api/v1/todolists/:id — Update Todolist

**Request:** `{ "title": "...", "description": "..." }`

**Note:** `status` field is ignored — computed from tasks.

---

### DELETE /api/v1/todolists/:id — Delete Todolist

**Response (204):** No content. Cascades to delete all tasks.

---

## 4. Endpoints — Tasks

### POST /api/v1/todolists/:todolistId/tasks — Create Task

**Request:**
```json
{
    "title": "Buy milk",
    "description": "2% fat, 1 gallon"
}
```

**Response (201):** Task object with status=pending.

---

### GET /api/v1/todolists/:id/tasks — List Tasks

**Query Parameters:**

| Param | Type | Default | Description |
|-------|------|---------|-------------|
| page | number | 1 | Page number |
| perPage | number | 20 | Items per page |
| status | string | all | Filter by status |
| title | string | — | Filter by title |

---

### GET /api/v1/tasks/:id — Get Task

**Response (200):** Single task object.

---

### PUT /api/v1/tasks/:id — Update Task

**Request:**
```json
{
    "title": "Buy milk",
    "description": "2% fat, 1 gallon",
    "status": "done"
}
```

**Note:** Updating status triggers todolist status recomputation.

---

### DELETE /api/v1/tasks/:id — Delete Task

**Response (204):** No content. Triggers todolist status recomputation.

---

## 5. Error Codes

| Code | HTTP | Description |
|------|------|-------------|
| VALIDATION_ERROR | 400 | Input validation failed |
| UNAUTHORIZED | 401 | Missing or invalid token |
| FORBIDDEN | 403 | Not owned by this user |
| NOT_FOUND | 404 | Resource not found |
| INTERNAL_ERROR | 500 | Server error |

---

## Related Documents

| Document | Relationship |
|----------|-------------|
| [[012_user_stories]] | Stories implementing these endpoints |
| [[013_acceptance_criteria]] | ACs for each endpoint |
| [[023_database_schema_DDL]] | Database backing these endpoints |
