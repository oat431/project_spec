---
document_type: Stakeholder Analysis
version: "1.0"
status: Draft
author: "OraMesLita (AI)"
created: "2026-07-04"
last_updated: "2026-07-04"
project_name: "tiny mchwa"
project_id: "tiny-mchwa"
ba_owner: "OraMesLita (AI)"
classification: "Internal"
tags: [stakeholder-analysis, homelab]
standard_ref:
  - SWEBOK v4 — Requirements
  - BABOK v3 — BA Planning & Monitoring
---

# Stakeholder Analysis

> **Project:** tiny mchwa 🐜
> **Version:** 1.0 | **Status:** Draft
> **Last Updated:** 2026-07-04

---

## 1. Stakeholder Register

| Stakeholder | Role | Influence | Interest | Engagement |
|------------|------|----------|----------|------------|
| Panomete | Owner / Developer / User | High | High | Manage Closely |
| Blog Service | Consumer service | Medium | Medium | Keep Informed |
| Cookbook Service | Consumer service | Medium | Medium | Keep Informed |
| OraMesLita (AI) | Business Analyst | Low | High | Keep Informed |

---

## 2. Stakeholder Concerns

| Stakeholder | Primary Concerns | Success Criteria |
|------------|-----------------|-----------------|
| Panomete | Learn spec-driven development, working microservice | MVP complete, specs before code |
| Blog Service | Reliable API for draft lists | Can create/read todolists via API |
| Cookbook Service | Reliable API for grocery lists | Can create/read todolists via API |

---

## 3. Requirements Influence

| Stakeholder | Requirements Influenced | Priority Impact |
|------------|------------------------|----------------|
| Panomete | All requirements | Drives all priorities |
| Blog Service | FR-001, FR-002, sourceService filtering | 🟡 Should Have |
| Cookbook Service | FR-001, FR-002, sourceService filtering | 🟡 Should Have |

---

## Related Documents

| Document | Relationship |
|----------|-------------|
| [[011_business_objective]] | Objectives driven by stakeholders |
| [[012_user_stories]] | Stories from stakeholder needs |
