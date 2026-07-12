---
document_type: Database Schema (DDL)
version: "1.0"
status: Approved
author: "OraMesLita (AI)"
created: "2026-07-01"
last_updated: "2026-07-04"
project_name: "tiny mchwa"
project_id: "tiny-mchwa"
tech_lead: "Panomete"
classification: "Internal"
tags: [database-schema, ddl, postgresql, homelab]
standard_ref:
  - SWEBOK v4 — Design
---

# Database Schema (DDL)

> **Project:** tiny mchwa 🐜
> **Version:** 1.0 | **Status:** Approved
> **Last Updated:** 2026-07-04

---

## 1. Database Overview

| Field | Detail |
|-------|--------|
| RDBMS | PostgreSQL 15+ |
| Character Set | UTF-8 |
| Schema | public |
| Naming Convention | snake_case, lowercase |

---

## 2. DDL Scripts

### 2.1 Todolists Table

```sql
CREATE TABLE todolists (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    title VARCHAR(255) NOT NULL,
    description VARCHAR(1000),
    owned_by UUID NOT NULL,
    source_service VARCHAR(100) NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_todolists_owned_by ON todolists(owned_by);
CREATE INDEX idx_todolists_source_service ON todolists(source_service);
CREATE INDEX idx_todolists_created_at ON todolists(created_at DESC);
```

**Note:** No `status` column — computed from tasks (see [[024_ERD]]).

---

### 2.2 Tasks Table

```sql
CREATE TABLE tasks (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    todolist_id UUID NOT NULL REFERENCES todolists(id) ON DELETE CASCADE,
    title VARCHAR(255) NOT NULL,
    description VARCHAR(1000),
    status VARCHAR(20) NOT NULL DEFAULT 'pending',
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),

    CONSTRAINT chk_tasks_status CHECK (status IN ('pending', 'inprogress', 'done'))
);

CREATE INDEX idx_tasks_todolist_id ON tasks(todolist_id);
CREATE INDEX idx_tasks_status ON tasks(status);
```

---

## 3. Triggers

### 3.1 Updated At Trigger

```sql
CREATE OR REPLACE FUNCTION update_updated_at_column()
RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = NOW();
    RETURN NEW;
END;
$$ language 'plpgsql';

CREATE TRIGGER trg_todolists_updated
    BEFORE UPDATE ON todolists
    FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();

CREATE TRIGGER trg_tasks_updated
    BEFORE UPDATE ON tasks
    FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();
```

---

## 4. Migrations

| Version | Date | Description | Script |
|---------|------|-------------|--------|
| v1.0 | 2026-07-04 | Initial schema | 001_initial.sql |

---

## Related Documents

| Document | Relationship |
|----------|-------------|
| [[024_ERD]] | Logical model this schema implements |
| [[022_API_specification]] | API using this schema |
