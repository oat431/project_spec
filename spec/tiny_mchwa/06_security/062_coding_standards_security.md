---
document_type: Security Coding Standards
version: "1.0"
status: Draft
author: "[Author Name]"
created: "[YYYY-MM-DD]"
last_updated: "[YYYY-MM-DD]"
project_name: "tiny mchwa"
project_id: "tiny-mchwa"
classification: "Internal"
tags: [security, coding-standards, homelab]
---

# Security Coding Standards

> **Project:** tiny mchwa 🐜
> **Status:** Draft

---

## Rules

| Rule | Description |
|------|-------------|
| Input validation | Validate all user input |
| SQL injection | Use parameterized queries (sqlx) |
| Auth | Validate JWT on all protected endpoints |
| Secrets | Never commit secrets, use .env |

---

## Related Documents

| Document | Relationship |
|----------|-------------|
| [[061_security_test_report]] | Security testing |
