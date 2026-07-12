---
document_type: Test Cases
version: "1.0"
status: Draft
author: "OraMesLita (AI)"
created: "2026-07-04"
last_updated: "2026-07-04"
project_name: "tiny mchwa"
project_id: "tiny-mchwa"
qa_lead: "Panomete"
classification: "Internal"
tags: [test-cases, testify, homelab]
standard_ref:
  - SWEBOK v4 — Testing
  - ISO/IEC/IEEE 29119 — Software Testing
---

# Test Cases

> **Project:** tiny mchwa 🐜
> **Version:** 1.0 | **Status:** Draft

---

## 1. Todolist Handler Tests

| TC ID | Test Name | Given | When | Then | Status |
|-------|-----------|-------|------|------|--------|
| TC-001 | Create todolist success | Valid request body | POST /todolists | 201, todolist returned | ✅ |
| TC-002 | Create todolist missing title | Empty title | POST /todolists | 400, validation error | ✅ |
| TC-003 | List todolists | User has todolists | GET /todolists | 200, array returned | ✅ |
| TC-004 | Get todolist | Todolist exists | GET /todolists/:id | 200, todolist returned | ✅ |
| TC-005 | Get todolist not found | ID doesn't exist | GET /todolists/:id | 404, error returned | ✅ |
| TC-006 | Update todolist | Todolist exists | PUT /todolists/:id | 200, updated returned | ✅ |
| TC-007 | Delete todolist | Todolist exists | DELETE /todolists/:id | 204, no content | ✅ |

---

## 2. Task Handler Tests

| TC ID | Test Name | Given | When | Then | Status |
|-------|-----------|-------|------|------|--------|
| TC-101 | Create task success | Todolist exists | POST /todolists/:id/tasks | 201, task returned | ✅ |
| TC-102 | Create task todolist not found | ID doesn't exist | POST /todolists/:id/tasks | 404, error | ✅ |
| TC-103 | List tasks | Todolist has tasks | GET /todolists/:id/tasks | 200, array returned | ✅ |
| TC-104 | Get task | Task exists | GET /tasks/:id | 200, task returned | ✅ |
| TC-105 | Update task | Task exists | PUT /tasks/:id | 200, updated returned | ✅ |
| TC-106 | Delete task | Task exists | DELETE /tasks/:id | 204, no content | ✅ |

---

## 3. Status Computation Tests

| TC ID | Test Name | Given | When | Then | Status |
|-------|-----------|-------|------|------|--------|
| TC-201 | No tasks = pending | 0 tasks | Compute status | `pending` | ✅ |
| TC-202 | All pending = pending | All tasks pending | Compute status | `pending` | ✅ |
| TC-203 | Any inprogress = inprogress | 1 task inprogress | Compute status | `inprogress` | ✅ |
| TC-204 | All done = done | All tasks done | Compute status | `done` | ✅ |

---

## 4. Validation Tests

| TC ID | Test Name | Given | When | Then | Status |
|-------|-----------|-------|------|------|--------|
| TC-301 | Empty title | title="" | POST /todolists | 400 error | ✅ |
| TC-302 | Title too long | title=256 chars | POST /todolists | 400 error | ✅ |
| TC-303 | Description too long | desc=1001 chars | POST /todolists | 400 error | ✅ |

---

## Related Documents

| Document | Relationship |
|----------|-------------|
| [[041_test_plan]] | Test strategy |
| [[013_acceptance_criteria]] | ACs these tests verify |
