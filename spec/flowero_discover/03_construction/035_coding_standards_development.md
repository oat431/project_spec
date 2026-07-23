---
document_type: Coding Standards / Style Guide
version: "0.1"
status: Draft
author: "Dev Persona"
created: "2026-07-23"
last_updated: "2026-07-23"
project_name: "Flowero Discover"
project_id: "PAN-DISCOVER-001"
parent_platform: "Panomete Platform"
classification: "Internal"
tags: [coding-standards, style-guide, java, spring-boot, gradle, panomete]
standard_ref:
  - SWEBOK v4 — Construction
  - Google Java Style Guide
---

# Coding Standards — Flowero Discover

> **Service:** Flowero Discover (Spring Cloud Netflix Eureka)
> **Platform:** Panomete Platform
> **Version:** 0.1 | **Status:** Draft
> **Last Updated:** 2026-07-23

---

## 1. Purpose

> Coding standards for Flowero Discover. These rules apply to this service and serve as the reference for all Panomete Java services (Gate, Guard). Standards are **enforced** — if code doesn't follow them, it doesn't merge.

## 2. Language & Framework

| Aspect | Standard |
|--------|---------|
| Language | Java 25 LTS |
| Build System | Gradle 9.5.1 (wrapper committed) |
| Framework | Spring Boot 4.1.0 |
| Cloud | Spring Cloud 2025.1.2 (Oakwood) |
| Test Framework | JUnit 5 + AssertJ |
| Runtime | Eclipse Temurin JDK/JRE 25 |

## 3. Code Style

### 3.1 General Rules (Google Java Style Guide)

| Rule | Standard | Example |
|------|---------|---------|
| Indentation | 4 spaces (no tabs) | Standard Java convention |
| Line Length | 120 characters max | Wrapped at logical boundaries |
| Braces | Egyptian style (opening brace on same line) | `if (x) {` |
| Naming — Classes | PascalCase | `FlowerodiscoveryApplication` |
| Naming — Methods | camelCase | `eurekaDashboardIsReachable()` |
| Naming — Variables | camelCase | `restTemplate`, `port` |
| Naming — Constants | UPPER_SNAKE_CASE | `MAX_RETRY_COUNT` |
| Naming — Packages | lowercase, reversed domain | `panomete.flowerodiscovery` |
| Imports | No wildcard imports | `import org. ... .RestTemplate` not `import org. ... .*` |

### 3.2 Spring Boot Conventions

| Rule | Standard |
|------|---------|
| Main class | `@SpringBootApplication` + main method |
| Server annotations | `@EnableEurekaServer` (not hidden in config class) |
| Configuration | `application.yaml` (not `.properties`) |
| Dependency injection | Constructor injection preferred (field injection avoided) |
| Lombok | **Not used** — explicit getters/constructors over magic |
| Records | Preferred over POJOs for DTOs/value objects |
| Exception handling | Catch specific exceptions, not `Exception` |
| Logging | SLF4J via Lombok's `@Slf4j` or explicit `LoggerFactory` |

### 3.3 Test Conventions

| Rule | Standard | Example |
|------|---------|---------|
| Class naming | `*Tests` suffix | `FlowerodiscoveryApplicationTests` |
| Method naming | Descriptive, snake_case allowed | `eurekaDashboardIsReachable()` |
| Assertions | AssertJ (`assertThat(...)`) | `assertThat(response.getStatusCode()).isEqualTo(HttpStatus.OK)` |
| Test annotations | `@SpringBootTest(webEnvironment = RANDOM_PORT)` | Integration tests |
| REST client | `RestTemplate` (plain, not `TestRestTemplate` — compatibility) | `new RestTemplate()` |
| URL building | Helper method, not string concat | `url("/eureka/apps")` |
| Test organization | One test class per application, focused tests | Group related assertions logically |

### 3.4 Example — Good vs Bad

```java
// ✅ Good — clear test, AssertJ assertions, helper method
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
class FlowerodiscoveryApplicationTests {

    @LocalServerPort
    private int port;

    private RestTemplate restTemplate;

    @BeforeEach
    void setUp() {
        restTemplate = new RestTemplate();
    }

    private String url(String path) {
        return "http://localhost:" + port + path;
    }

    @Test
    void eurekaDashboardIsReachable() {
        ResponseEntity<String> response = restTemplate.getForEntity(
                url("/"), String.class);

        assertThat(response.getStatusCode()).isEqualTo(HttpStatus.OK);
        assertThat(response.getBody())
                .contains("Instances currently registered with Eureka");
    }
}

// ❌ Bad — vague names, field injection, no helper, raw strings
@SpringBootTest
class Tests {
    @Autowired RestTemplate t;    // field injection + bad name

    @Test
    void test1() {                 // meaningless name
        var r = t.getForEntity(
            "http://localhost:" + 8080 + "/eureka/apps", String.class);
        assert r.getStatusCodeValue() == 200;  // JUnit 4-style assertion
    }
}
```

## 4. Configuration Conventions

### application.yaml Structure

```yaml
# 1. Spring core settings
spring:
  application:
    name: flowero-discover       # kebab-case, matches Docker service name

# 2. Server settings
server:
  port: 8999

# 3. Eureka settings (grouped logically)
eureka:
  instance:
    hostname: flowero-discover
  client:
    register-with-eureka: false
    fetch-registry: false
  server:
    enable-self-preservation: true
    eviction-interval-timer-in-ms: 5000

# 4. Actuator / management
management:
  endpoints:
    web:
      exposure:
        include: health,info

# 5. Logging (last)
logging:
  level:
    com.netflix.eureka: INFO
```

**Rules:**
- YAML over `.properties` — hierarchical, readable
- kebab-case for all keys and values (`flowero-discover`, not `floweroDiscover`)
- Group by domain (spring → server → eureka → management → logging)
- Every non-obvious value has an inline comment explaining WHY

## 5. Project Structure

```
flowerodiscovery/
├── build.gradle                    # Dependencies + plugins + BOM
├── settings.gradle                 # rootProject.name = 'flowerodiscovery'
├── Dockerfile                      # Multi-stage (JDK → JRE)
├── docker-compose.fragment.yml     # Compose entry for platform
├── .dockerignore
├── gradlew / gradlew.bat           # Committed wrapper scripts
├── gradle/wrapper/                 # Committed wrapper JAR + properties
└── src/
    ├── main/
    │   ├── java/panomete/flowerodiscovery/
    │   │   └── FlowerodiscoveryApplication.java
    │   └── resources/
    │       └── application.yaml
    └── test/
        └── java/panomete/flowerodiscovery/
            └── FlowerodiscoveryApplicationTests.java
```

**Rules:**
- Package: `panomete.{servicename}` (lowercase, no hyphens)
- One main class at package root
- No `controller/`, `service/`, `config/` sub-packages — Eureka Server doesn't have custom controllers
- Tests mirror main package structure

## 6. Formatting & Linting

| Tool | Status | Notes |
|------|:---:|-------|
| Checkstyle | ❌ Not configured | Recommended: Google Java Style via `com.puppycrawl.tools:checkstyle` |
| Spotless | ❌ Not configured | Recommended: `com.diffplug.spotless` Gradle plugin |
| SpotBugs | ❌ Not configured | Recommended for static analysis |
| OWASP Dependency-Check | ❌ Not configured | Recommended for vulnerability scanning |

> ⚠️ No automated formatting/linting yet. All services are hand-reviewed. Add Checkstyle + Spotless when onboarding a second developer.

## 7. Git Conventions

| Rule | Standard |
|------|---------|
| Branch naming | `feature/discover-*`, `fix/discover-*` |
| Commit format | Conventional Commits — see [[034_commit_messages_changelog]] |
| Merge strategy | Squash-merge to `dev`, rebase-merge to `main` |
| `.gitignore` | `build/`, `.gradle/`, `.idea/`, `*.iml` |

---

## Related Documents

| Document | Relationship |
|----------|-------------|
| [[031_README_developer_guide]] | Developer onboarding |
| [[034_commit_messages_changelog]] | Commit format enforced |
| [[036_code_review_records]] | Standards enforced during review |

---

> **Template Standard:** Based on SWEBOK v4, Google Java Style Guide
> **Usage:** These standards apply to Flowero Discover and are the reference for all Panomete Java services. Adding Checkstyle + Spotless is the next step for automated enforcement.
