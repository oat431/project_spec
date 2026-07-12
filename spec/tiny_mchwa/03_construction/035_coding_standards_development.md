---
document_type: Coding Standards
version: "1.0"
status: Draft
author: "Panomete"
created: "2026-07-04"
last_updated: "2026-07-04"
project_name: "tiny mchwa"
project_id: "tiny-mchwa"
classification: "Internal"
tags: [coding-standards, go, homelab]
---

# Coding Standards

> **Project:** tiny mchwa 🐜
> **Status:** Draft

---

## Go Standards

| Rule | Description |
|------|-------------|
| Error handling | Always check errors, wrap with context |
| Naming | camelCase for unexported, PascalCase for exported |
| Package structure | Package-by-feature, not by-layer |
| Testing | testify assert + require |
| Validation | go-playground/validator |

---

## Linting

```bash
golangci-lint run
```

Enabled linters: errcheck, gosimple, govet, ineffassign, staticcheck, unused

---

## Related Documents

| Document | Relationship |
|----------|-------------|
| [[031_README_developer_guide]] | Developer setup |
