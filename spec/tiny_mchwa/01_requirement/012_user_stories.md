---
document_type: User Stories
version: "1.0"
status: Draft
author: "OraMesLita (AI)"
created: "2026-07-04"
last_updated: "2026-07-04"
project_name: "tiny mchwa"
project_id: "tiny-mchwa"
ba_owner: "OraMesLita (AI)"
po_owner: "Panomete"
classification: "Internal"
tags: [user-stories, agile, backlog, homelab]
standard_ref:
  - SWEBOK v4 — Requirements
  - ISO/IEC/IEEE 29148 — Requirements Engineering
---

# User Stories

> **Project:** tiny mchwa 🐜
> **Version:** 1.0 | **Status:** Draft
> **Last Updated:** 2026-07-04

---

## 1. Epic Overview

| Epic ID | Epic Name | Stories | Total Points | Sprint |
|---------|-----------|---------|-------------|--------|
| E-01 | Todolist CRUD | 5 | 13 | Sprint 1 |
| E-02 | Task CRUD | 5 | 13 | Sprint 2 |
| E-03 | Computed Status | 2 | 8 | Sprint 2 |
| E-04 | Cross-Service Integration | 3 | 13 | Sprint 3 |

---

## 2. User Stories

### Epic E-01: Todolist CRUD

#### US-001: Create Todolist

**As a** user
**I want** to create a new todolist
**So that** I can organize my tasks

**Acceptance Criteria:**
- **AC-1:** Given I am authenticated, When I POST `/api/v1/todolists` with title and description, Then a new todolist is created with status `pending`
- **AC-2:** Given I omit the title, When I submit, Then I get a 400 error with validation message

**Story Points:** 3
**Priority:** 🔴 Must Have
**Epic:** E-01
**Status:** ✅ Complete

---

#### US-002: List Todolists

**As a** user
**I want** to view all my todolists
**So that** I can see what I need to do

**Acceptance Criteria:**
- **AC-1:** Given I have todolists, When I GET `/api/v1/todolists`, Then I see all my todolists with pagination
- **AC-2:** Given I have 50 todolists, When I request page 1 with perPage=20, Then I get 20 items with meta showing totalPages=3

**Story Points:** 3
**Priority:** 🔴 Must Have
**Epic:** E-01
**Status:** ✅ Complete

---

#### US-003: Get Todolist Detail

**As a** user
**I want** to view a single todolist
**So that** I can see its details and tasks

**Acceptance Criteria:**
- **AC-1:** Given a todolist exists, When I GET `/api/v1/todolists/:id`, Then I see the todolist details
- **AC-2:** Given the todolist doesn't exist, When I GET it, Then I get a 404 error

**Story Points:** 2
**Priority:** 🔴 Must Have
**Epic:** E-01
**Status:** ✅ Complete

---

#### US-004: Update Todolist

**As a** user
**I want** to update a todolist's title and description
**So that** I can correct or refine my lists

**Acceptance Criteria:**
- **AC-1:** Given a todolist exists, When I PUT `/api/v1/todolists/:id` with new title, Then the todolist is updated
- **AC-2:** Given I try to update status directly, When I submit, Then status is ignored (computed from tasks)

**Story Points:** 2
**Priority:** 🔴 Must Have
**Epic:** E-01
**Status:** ✅ Complete

---

#### US-005: Delete Todolist

**As a** user
**I want** to delete a todolist
**So that** I can remove lists I no longer need

**Acceptance Criteria:**
- **AC-1:** Given a todolist exists, When I DELETE `/api/v1/todolists/:id`, Then the todolist and all its tasks are deleted
- **AC-2:** Given the todolist doesn't exist, When I delete it, Then I get a 404 error

**Story Points:** 3
**Priority:** 🔴 Must Have
**Epic:** E-01
**Status:** ✅ Complete

---

### Epic E-02: Task CRUD

#### US-101: Create Task

**As a** user
**I want** to create a task under a todolist
**So that** I can track individual items

**Acceptance Criteria:**
- **AC-1:** Given a todolist exists, When I POST `/api/v1/todolists/:id/tasks` with title, Then a task is created with status `pending`
- **AC-2:** Given the todolist doesn't exist, When I create a task, Then I get a 404 error

**Story Points:** 3
**Priority:** 🔴 Must Have
**Epic:** E-02
**Status:** ✅ Complete

---

#### US-102: List Tasks

**As a** user
**I want** to view all tasks in a todolist
**So that** I can see what needs to be done

**Acceptance Criteria:**
- **AC-1:** Given a todolist has tasks, When I GET `/api/v1/todolists/:id/tasks`, Then I see all tasks with pagination
- **AC-2:** Given I filter by status=pending, When I request, Then I only see pending tasks

**Story Points:** 3
**Priority:** 🔴 Must Have
**Epic:** E-02
**Status:** ✅ Complete

---

#### US-103: Get Task Detail

**As a** user
**I want** to view a single task
**So that** I can see its details

**Acceptance Criteria:**
- **AC-1:** Given a task exists, When I GET `/api/v1/tasks/:id`, Then I see the task details

**Story Points:** 2
**Priority:** 🔴 Must Have
**Epic:** E-02
**Status:** ✅ Complete

---

#### US-104: Update Task

**As a** user
**I want** to update a task's title, description, and status
**So that** I can track progress

**Acceptance Criteria:**
- **AC-1:** Given a task exists, When I PUT `/api/v1/tasks/:id` with status=done, Then the task is updated
- **AC-2:** Given I update a task status, When the update succeeds, Then the parent todolist status is recomputed

**Story Points:** 3
**Priority:** 🔴 Must Have
**Epic:** E-02
**Status:** ✅ Complete

---

#### US-105: Delete Task

**As a** user
**I want** to delete a task
**So that** I can remove completed or irrelevant items

**Acceptance Criteria:**
- **AC-1:** Given a task exists, When I DELETE `/api/v1/tasks/:id`, Then the task is deleted
- **AC-2:** Given I delete a task, When the delete succeeds, Then the parent todolist status is recomputed

**Story Points:** 2
**Priority:** 🔴 Must Have
**Epic:** E-02
**Status:** ✅ Complete

---

### Epic E-03: Computed Status

#### US-201: Todolist Status from Tasks

**As a** user
**I want** todolist status to be automatically computed from tasks
**So that** I don't have to manually update list status

**Acceptance Criteria:**
- **AC-1:** Given all tasks are pending, When I view the todolist, Then status is `pending`
- **AC-2:** Given any task is inprogress, When I view the todolist, Then status is `inprogress`
- **AC-3:** Given all tasks are done, When I view the todolist, Then status is `done`

**Story Points:** 5
**Priority:** 🔴 Must Have
**Epic:** E-03
**Status:** ✅ Complete

---

#### US-202: Status Recomputation Triggers

**As a** developer
**I want** todolist status to recompute on task changes
**So that** status is always accurate

**Acceptance Criteria:**
- **AC-1:** Given I create/update/delete a task, When the operation completes, Then todolist status is recomputed

**Story Points:** 3
**Priority:** 🔴 Must Have
**Epic:** E-03
**Status:** ✅ Complete

---

### Epic E-04: Cross-Service Integration

#### US-301: Blog Draft Lists

**As a** blog service
**I want** to create draft todo lists via API
**So that** content planning is centralized

**Acceptance Criteria:**
- **AC-1:** Given blog service calls POST `/api/v1/todolists` with sourceService=blog, When successful, Then todolist is created and linked to blog

**Story Points:** 5
**Priority:** 🟡 Should Have
**Epic:** E-04
**Status:** ⬜ Not Started

---

#### US-302: Cookbook Grocery Lists

**As a** cookbook service
**I want** to create grocery lists via API
**So that** recipe ingredients are tracked

**Acceptance Criteria:**
- **AC-1:** Given cookbook service calls POST `/api/v1/todolists` with sourceService=cookbook, When successful, Then todolist is created and linked to cookbook

**Story Points:** 5
**Priority:** 🟡 Should Have
**Epic:** E-04
**Status:** ⬜ Not Started

---

#### US-303: Filter by Source Service

**As a** user
**I want** to filter todolists by source service
**So that** I can see lists from a specific service

**Acceptance Criteria:**
- **AC-1:** Given I have todolists from multiple services, When I filter by sourceService=blog, Then I only see blog-related lists

**Story Points:** 3
**Priority:** 🟡 Should Have
**Epic:** E-04
**Status:** ⬜ Not Started

---

## 3. Story Estimation Summary

| Epic | Stories | Total Points | Status |
|------|---------|-------------|--------|
| E-01 Todolist CRUD | 5 | 13 | ✅ Complete |
| E-02 Task CRUD | 5 | 13 | ✅ Complete |
| E-03 Computed Status | 2 | 8 | ✅ Complete |
| E-04 Cross-Service | 3 | 13 | ⬜ Not Started |
| **Total** | **15** | **47** | |

---

## Related Documents

| Document | Relationship |
|----------|-------------|
| [[013_acceptance_criteria]] | Detailed ACs for each story |
| [[022_API_specification]] | API contract these stories implement |
| [[011_business_objective]] | Stories trace to business objectives |
