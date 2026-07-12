---
document_type: Dependency Manifest
version: "1.1"
status: Approved
author: "OraMesLita (AI)"
created: "2026-07-04"
last_updated: "2026-07-05"
project_name: "tiny mchwa"
project_id: "tiny-mchwa"
classification: "Internal"
tags: [dependencies, go, react, homelab]
standard_ref:
  - SWEBOK v4 — Construction
---

# Dependency Manifest

> **Project:** tiny mchwa 🐜
> **Version:** 1.1 | **Status:** Approved
> **Last Updated:** 2026-07-05

---

## API Dependencies (go.mod)

| Package | Version | Purpose |
|---------|---------|---------|
| github.com/gofiber/fiber/v3 | v3.4.0 | HTTP framework |
| github.com/jmoiron/sqlx | v1.4.0 | SQL extensions |
| github.com/lib/pq | v1.12.3 | PostgreSQL driver |
| github.com/go-playground/validator/v10 | v10.30.3 | Struct validation |
| github.com/joho/godotenv | v1.5.1 | .env loading |
| github.com/google/uuid | v1.6.0 | UUID generation |
| github.com/stretchr/testify | v1.11.1 | Testing assertions |

## API Dev Dependencies

| Package | Version | Purpose |
|---------|---------|---------|
| github.com/air-verse/air | latest | Hot reload |
| github.com/golangci/golangci-lint | latest | Linting |

---

## Web Dependencies (package.json)

| Package | Version | Purpose |
|---------|---------|---------|
| react | ^19.2.7 | UI framework |
| react-dom | ^19.2.7 | React DOM renderer |
| react-router-dom | ^7.18.1 | Client-side routing |
| @tanstack/react-query | ^5.101.2 | Data fetching + caching |
| @tailwindcss/vite | ^4.3.2 | Tailwind CSS integration |
| tailwindcss | ^4.3.2 | Utility-first CSS |
| daisyui | ^5.6.13 | Component library |

## Web Dev Dependencies

| Package | Version | Purpose |
|---------|---------|---------|
| typescript | ^6.0.3 | Type checking |
| vite | ^8.1.3 | Build tool |
| @vitejs/plugin-react | ^6.0.3 | React plugin for Vite |
| @types/react | ^19.2.17 | React type definitions |
| @types/react-dom | ^19.2.3 | ReactDOM type definitions |
| autoprefixer | ^10.5.2 | CSS autoprefixer |
| postcss | ^8.5.16 | CSS processing |

---

## Related Documents

| Document | Relationship |
|----------|-------------|
| [[031_README_developer_guide]] | Setup instructions |
| [[025_software_architecture_document]] | Architecture overview |
