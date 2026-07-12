---
document_type: Acceptance Criteria (ATDD/BDD)
version: "1.0"
status: Draft
author: "OraMesLita (AI)"
created: "2026-07-04"
last_updated: "2026-07-04"
project_name: "tiny mchwa"
project_id: "tiny-mchwa"
ba_owner: "OraMesLita (AI)"
qa_lead: "Panomete"
classification: "Internal"
tags: [acceptance-criteria, bdd, atdd, given-when-then]
standard_ref:
  - SWEBOK v4 — Requirements
  - ISO/IEC/IEEE 29119 — Software Testing
  - ISO/IEC/IEEE 29148 — Requirements Engineering
---

# Acceptance Criteria (ATDD/BDD)

> **Project:** tiny mchwa 🐜
> **Version:** 1.0 | **Status:** Draft
> **Last Updated:** 2026-07-04

---

## 1. Todolists

### FR-001: Create Todolist

| AC ID | Scenario | Given | When | Then | Priority |
|-------|---------|-------|------|------|----------|
| AC-001a | Happy path | User is authenticated | POST `/api/v1/todolists` with title, description, sourceService | 201 Created, todolist returned with id, status=pending | 🔴 |
| AC-001b | Missing title | User is authenticated | POST `/api/v1/todolists` without title | 400 Bad Request, validation error | 🔴 |
| AC-001c | Default status | User is authenticated | POST valid todolist | status is `pending` (no tasks yet) | 🔴 |

### FR-002: List Todolists

| AC ID | Scenario | Given | When | Then | Priority |
|-------|---------|-------|------|------|----------|
| AC-002a | Pagination | User has 50 todolists | GET `/api/v1/todolists?page=1&perPage=20` | 20 items returned, meta shows total=50, totalPages=3 | 🔴 |
| AC-002b | Filter by status | User has mixed status todolists | GET `/api/v1/todolists?status=inprogress` | Only inprogress todolists returned | 🟡 |
| AC-002c | Filter by source | User has todolists from blog and cookbook | GET `/api/v1/todolists?sourceService=blog` | Only blog todolists returned | 🟡 |
| AC-002d | Filter by title | User has "Grocery" and "Daily" lists | GET `/api/v1/todolists?title=grocery` | Only "Grocery" list returned | 🟡 |

### FR-003: Get Todolist

| AC ID | Scenario | Given | When | Then | Priority |
|-------|---------|-------|------|------|----------|
| AC-003a | Happy path | Todolist exists | GET `/api/v1/todolists/:id` | 200 OK, todolist returned | 🔴 |
| AC-003b | Not found | Todolist doesn't exist | GET `/api/v1/todolists/:id` | 404 Not Found | 🔴 |

### FR-004: Update Todolist

| AC ID | Scenario | Given | When | Then | Priority |
|-------|---------|-------|------|------|----------|
| AC-004a | Update title | Todolist exists | PUT `/api/v1/todolists/:id` with new title | 200 OK, title updated | 🔴 |
| AC-004b | Status immutable | Todolist exists | PUT with status field | Status field ignored, computed from tasks | 🔴 |

### FR-005: Delete Todolist

| AC ID | Scenario | Given | When | Then | Priority |
|-------|---------|-------|------|------|----------|
| AC-005a | Cascade delete | Todolist has 3 tasks | DELETE `/api/v1/todolists/:id` | 204 No Content, todolist and all 3 tasks deleted | 🔴 |
| AC-005b | Not found | Todolist doesn't exist | DELETE `/api/v1/todolists/:id` | 404 Not Found | 🔴 |

---

## 2. Tasks

### FR-101: Create Task

| AC ID | Scenario | Given | When | Then | Priority |
|-------|---------|-------|------|------|----------|
| AC-101a | Happy path | Todolist exists | POST `/api/v1/todolists/:id/tasks` with title | 201 Created, task with status=pending | 🔴 |
| AC-101b | Todolist not found | Todolist doesn't exist | POST task to it | 404 Not Found | 🔴 |

### FR-102: List Tasks

| AC ID | Scenario | Given | When | Then | Priority |
|-------|---------|-------|------|------|----------|
| AC-102a | Pagination | Todolist has 30 tasks | GET `/api/v1/todolists/:id/tasks?page=1&perPage=20` | 20 items, meta shows total=30 | 🔴 |
| AC-102b | Filter by status | Tasks have mixed status | GET with `?status=done` | Only done tasks returned | 🟡 |

### FR-103: Update Task

| AC ID | Scenario | Given | When | Then | Priority |
|-------|---------|-------|------|------|----------|
| AC-103a | Update status | Task exists | PUT `/api/v1/tasks/:id` with status=done | 200 OK, status updated | 🔴 |
| AC-103b | Triggers recompute | Task status changes | PUT succeeds | Parent todolist status recomputed | 🔴 |

### FR-104: Delete Task

| AC ID | Scenario | Given | When | Then | Priority |
|-------|---------|-------|------|------|----------|
| AC-104a | Delete task | Task exists | DELETE `/api/v1/tasks/:id` | 204 No Content | 🔴 |
| AC-104b | Triggers recompute | Task deleted | DELETE succeeds | Parent todolist status recomputed | 🔴 |

---

## 3. Status Computation

### BIZ-001: Todolist Status Rules

| AC ID | Scenario | Given | When | Then | Priority |
|-------|---------|-------|------|------|----------|
| AC-B001a | No tasks | Todolist has 0 tasks | View todolist | status = `pending` | 🔴 |
| AC-B001b | All pending | All tasks are `pending` | View todolist | status = `pending` | 🔴 |
| AC-B001c | Any inprogress | Any task is `inprogress` | View todolist | status = `inprogress` | 🔴 |
| AC-B001d | All done | All tasks are `done` | View todolist | status = `done` | 🔴 |

---

## 4. Acceptance Criteria Summary

| Requirement | Total ACs | 🔴 Must Have | 🟡 Should Have |
|------------|----------|-------------|---------------|
| FR-001 Create Todolist | 3 | 3 | 0 |
| FR-002 List Todolists | 4 | 1 | 3 |
| FR-003 Get Todolist | 2 | 2 | 0 |
| FR-004 Update Todolist | 2 | 2 | 0 |
| FR-005 Delete Todolist | 2 | 2 | 0 |
| FR-101 Create Task | 2 | 2 | 0 |
| FR-102 List Tasks | 2 | 1 | 1 |
| FR-103 Update Task | 2 | 2 | 0 |
| FR-104 Delete Task | 2 | 2 | 0 |
| BIZ-001 Status Rules | 4 | 4 | 0 |
| **Total** | **25** | **21** | **4** |

---

## Related Documents

| Document | Relationship |
|----------|-------------|
| [[012_user_stories]] | Stories these ACs verify |
| [[022_API_specification]] | API contract |
| [[042_test_cases]] | Test cases from these ACs |
