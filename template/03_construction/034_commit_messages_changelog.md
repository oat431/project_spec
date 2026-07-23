---
document_type: Commit Messages / Changelog
version: "2.0"
status: Active
author: "[Technical Lead]"
created: "[YYYY-MM-DD]"
last_updated: "[YYYY-MM-DD]"
project_name: "[Project Name]"
project_id: "[Project-ID]"
classification: "Internal / Confidential"
tags: [commit-messages, changelog, conventional-commits, swebok]
standard_ref:
  - SWEBOK v4 — Construction
  - Conventional Commits v1.0
---

# Commit Messages / Changelog

> **Project:** [Project Name]
> **Version:** [X.Y] | **Status:** [Active]
> **Last Updated:** [YYYY-MM-DD]

---

## 1. Purpose

> Standardized commit messages and auto-generated changelogs. Good commit messages are documentation — "fix bug" is useless. "fix(auth): resolve token refresh race when user has multiple sessions" is useful.

## 2. Commit Message Format

Based on [Conventional Commits](https://www.conventionalcommits.org/):

```
<type>(<scope>): <description>

[optional body — explains WHY, not WHAT]

[optional footer(s)]
```

### Types

| Type | Description | Example |
|------|-----------|---------|
| `feat` | New feature | `feat(discover): add self-preservation config` |
| `fix` | Bug fix | `fix(gate): resolve NPE on missing JWT claims` |
| `docs` | Documentation only | `docs(readme): add Docker setup instructions` |
| `style` | Formatting, whitespace (no logic change) | `style: fix indentation in build.gradle` |
| `refactor` | Code restructuring (no feature/fix) | `refactor(guard): extract realm config to JSON` |
| `test` | Add or update tests | `test(discover): add standalone mode verification` |
| `chore` | Build, tooling, deps | `chore(deps): bump Spring Boot to 4.1.0` |
| `perf` | Performance improvement | `perf(gate): cache JWKS with 5min TTL` |
| `ci` | CI/CD changes | `ci: add Gradle build to GitHub Actions` |
| `build` | Build system or external deps | `build: configure Checkstyle plugin` |
| `revert` | Revert a previous commit | `revert: feat(discover): add clustering support` |

### Breaking Changes

Append `!` after type/scope, or add `BREAKING CHANGE:` footer:

```
feat(api)!: change registration response format

BREAKING CHANGE: Registration endpoint now returns JSON by default.
Clients must set Accept: application/xml for legacy XML format.
```

## 3. Scopes — Microservice Platform

| Scope | Applies To | Example |
|-------|-----------|---------|
| `discover` | Flowero Discover (Eureka) | `feat(discover): configure standalone mode` |
| `guard` | Flowero Guard (Keycloak) | `feat(guard): add panomete realm config` |
| `gate` | Flowero Gate (API Gateway) | `fix(gate): correct lb:// route resolution` |
| `gufo` | Cute Gufo (Blog) | `feat(gufo): add post pagination` |
| `mouton` | Fluffy Mouton (URL Shortener) | `feat(mouton): implement redirect endpoint` |
| `mchwa` | Tiny Mchwa (Todo) | `fix(mchwa): handle empty task lists` |
| `schwein` | Big Schwein (Ledger) | `feat(schwein): add transaction export` |
| `ardilla` | Shy Ardilla (Cook Book) | `feat(ardilla): add recipe search` |
| `jelen` | White Jelen (Hora) | `feat(jelen): implement scheduling` |
| `docker` | Container config | `chore(docker): add healthcheck to compose` |
| `nginx` | Reverse proxy config | `feat(nginx): route discovery.panomete.com` |
| `deps` | Cross-cutting dependencies | `chore(deps): update Spring Cloud to 2025.1.2` |
| `docs` | Cross-cutting documentation | `docs: update platform README` |
| `ci` | CI/CD pipelines | `ci: add multi-stack build matrix` |

## 4. Commit Guidelines

| Rule | Rationale |
|------|----------|
| Use imperative mood | "add" not "added" — matches `git merge` default |
| First line ≤ 72 characters | Readable in `git log --oneline` |
| Body explains WHY, not WHAT | Code shows *what* changed; commit explains *why* |
| Reference issues/tickets | `Closes #123` or `Refs PAN-42` |
| One logical change per commit | Atomic commits → easy to revert, bisect |
| No WIP commits on main | Squash before merge if needed |

## 5. Changelog Generation

### Tooling by Stack

| Stack | Tool | Command |
|-------|------|---------|
| Any (git-based) | [git-cliff](https://git-cliff.org/) | `git cliff -o CHANGELOG.md` |
| Node.js | [standard-version](https://github.com/conventional-changelog/standard-version) | `npx standard-version` |
| Node.js | [commitlint](https://commitlint.js.org/) | `npx commitlint --from HEAD~1` |
| Java/Gradle | [semantic-release](https://semantic-release.gitbook.io/) | CI-based, auto on merge to main |
| Go | [svu](https://github.com/caarlos0/svu) | `svu next` |

## 6. Changelog Format

```markdown
# Changelog

## [0.1.0] - 2026-07-23

### Features
- **discover:** configure Eureka standalone mode with self-preservation ([#1](link))
- **discover:** add Docker multi-stage build and compose fragment
- **discover:** add health check and dashboard verification tests

### Documentation
- **discover:** rewrite README with architecture and setup guide

## [0.0.1] - 2026-07-22

### Chores
- **discover:** initial Spring Initializr skeleton (Eureka Server + Gradle)
```

---

## Related Documents

| Document | Relationship |
|----------|-------------|
| [[035_coding_standards_development]] | Code standards these commits follow |
| [[031_README_developer_guide]] | Contribution guide |
| [[032_build_scripts]] | Build process these commits trigger |

---

> **Template Standard:** Based on SWEBOK v4, Conventional Commits v1.0
> **Usage:** Every commit tells a story. "fix bug" is a wasted commit. Use the scope table for multi-service platforms. Generate changelogs automatically — never write them by hand.
