---
document_type: Coding Standards / Style Guide
version: "0.1"
status: Active
author: "Dev Persona"
created: "2026-07-24"
last_updated: "2026-07-24"
project_name: "Flowero Gate"
project_id: "PAN-GATE-001"
parent_platform: "Panomete Platform"
classification: "Internal"
tags: [coding-standards, java, spring-boot, gateway, panomete]
standard_ref:
  - SWEBOK v4 — Construction
  - Google Java Style Guide
  - Spring Framework Documentation
---

# Coding Standards — Flowero Gate

> **Service:** Flowero Gate (Spring Cloud Gateway)
> **Platform:** Panomete Platform
> **Version:** 0.1 | **Status:** Active
> **Last Updated:** 2026-07-24

---

## 1. Purpose

> Coding standards ensure consistent, readable, maintainable code. Standards are **enforced**, not suggested. These standards reflect what the codebase actually follows — not aspirational guidelines.

## 2. Java / Spring Boot Standards

### Language & Build

| Aspect | Standard | Enforcement |
|--------|---------|-----------|
| Java Version | 25 LTS | `java.toolchain` in Gradle |
| Build System | Gradle 9.5.1 (wrapper committed) | `./gradlew` |
| DSL | Groovy (`build.gradle`) | Convention |
| Style Guide | Google Java Style Guide | Manual (no Checkstyle yet) |
| Formatting | 4 spaces, ~120 char line width | IDE default |
| Null Handling | `Assert.notNull()` for required params | Spring Framework convention |

### Naming Conventions

| Element | Convention | Example from codebase |
|---------|-----------|----------------------|
| Classes | PascalCase | `SecurityConfig`, `JwtClaimHeaderFilter`, `ResilientRedisRateLimiter` |
| Methods | camelCase | `securityWebFilterChain()`, `createValidJwt()` |
| Constants | UPPER_SNAKE_CASE | `private static WireMockServer authServer` (static fields) |
| Packages | lowercase, reversed domain | `panomete.flowerogate.config`, `panomete.flowerogate.filter` |
| Config properties | kebab-case | `app.post-login-redirect-url`, `cors.allowed-origins` |
| Environment vars | UPPER_SNAKE_CASE | `EUREKA_CLIENT_ENABLED`, `JWT_ISSUER_URI` |

### Dependency Injection

- **Constructor injection only** — no `@Autowired` on fields
- Constructor parameters use `@Value` for property injection

```java
// ✅ Good — constructor injection with @Value
public SecurityConfig(
        @Value("${app.post-login-redirect-url}") String postLoginRedirectUrl,
        CorsConfigurationSource corsConfigurationSource) {
    this.postLoginRedirectUrl = postLoginRedirectUrl;
    this.corsConfigurationSource = corsConfigurationSource;
}
```

### Configuration Classes

- Annotated with `@Configuration`
- Public methods annotated with `@Bean` (convention: `@Primary` when overriding auto-config)
- Descriptive Javadoc on class, not on every method

```java
// ✅ Good — clear class-level Javadoc
/**
 * Permissive security config for routing/filter tests.
 * Disables auth on all paths so WireMock-based tests can verify behavior.
 */
@TestConfiguration
public class TestSecurityConfig {
    @Bean
    @Primary
    SecurityWebFilterChain testSecurityFilterChain(ServerHttpSecurity http) { ... }
}
```

### Filter / WebFilter Classes

- Implement `WebFilter` interface
- Use `ServerWebExchange` (reactive), not `HttpServletRequest`
- Structured logging with `{}` placeholders

```java
// ✅ Good — reactive filter with structured logging
public class RequestLoggingFilter implements WebFilter {
    @Override
    public Mono<Void> filter(ServerWebExchange exchange, WebFilterChain chain) {
        long start = System.currentTimeMillis();
        return chain.filter(exchange).doFinally(s -> {
            long latency = System.currentTimeMillis() - start;
            log.info("method={} path={} status={} latency_ms={}",
                    exchange.getRequest().getMethod(), ...);
        });
    }
}
```

### Exception Handling

- Catch specific exceptions, not `Exception`
- Return standardized JSON error bodies
- Use `@ExceptionHandler` in `@RestControllerAdvice`

```java
// ✅ Good — specific exception classes, JSON error response
@RestControllerAdvice
public class GatewayExceptionHandler {
    @ExceptionHandler(ResponseStatusException.class)
    public ResponseEntity<Map<String, Object>> handle(ResponseStatusException ex) {
        return ResponseEntity.status(ex.getStatusCode())
                .body(Map.of("error", ex.getReason(), "status", ex.getStatusCode().value()));
    }
}
```

## 3. YAML Configuration Conventions

### Property Ordering

```yaml
server:       # Server config first
spring:       # Spring config
  application:
  cloud:
  security:
  data:
management:   # Actuator
eureka:       # Service discovery
logging:      # Logging last
```

### Profile Strategy

| File | Purpose | When Loaded |
|------|---------|------------|
| `application.yaml` | Base config — defaults for all environments | Always |
| `application-dev.yaml` | Dev overrides — localhost Keycloak, verbose logging | `spring.profiles.active=dev` |
| `application-prod.yaml` | Prod overrides — env vars, Eureka enabled, JSON logging | `spring.profiles.active=prod` |
| `src/test/resources/application.yaml` | Test config — excludes Eureka, Redis, OTLP auto-config | Test context only |

### Environment Variable Pattern

```yaml
# ✅ Good — env var with sensible default
host: ${REDIS_HOST:localhost}
port: ${REDIS_PORT:6379}

# ✅ Good — no default (must be provided)
redirect-url: ${app.post-login-redirect-url}
```

## 4. Test Conventions

### Test Class Structure

- `@SpringBootTest(webEnvironment = RANDOM_PORT)` for integration tests
- `@LocalServerPort int port` for dynamic port assignment
- `WebTestClient` for reactive endpoint testing
- WireMock for stubbing external services (JWKS endpoint, upstream services)

### Test Naming

| Pattern | Example |
|---------|---------|
| `methodName_condition_expectedBehavior` or descriptive camelCase | `protectedEndpointReturns401WithoutJwt()` |
| Test class named after the class under test + `Tests` suffix | `SecurityTests`, `RouteTests` |

### Test Assertions

```java
// ✅ Good — WebTestClient with expressive assertions
client.get().uri("/api/v1/users/me")
        .exchange()
        .expectStatus().isUnauthorized()
        .expectBody()
        .jsonPath("$.error").isEqualTo("Unauthorized")
        .jsonPath("$.status").isEqualTo(401);

// ✅ Good — value assertions with context
.expectStatus().value(status -> {
    assert status != 401 : "Expected non-401, got 401";
    assert status != 403 : "Expected non-403, got 403";
});
```

### Test Helpers

- `JwtTestHelper` — generates RSA keys, JWK sets, and signed JWTs for security tests
- `TestSecurityConfig` — `@TestConfiguration` with `@Primary` bean to override production security
- `@Import(TestSecurityConfig.class)` — used by routing tests that don't verify auth

## 5. Project Structure

```
src/
├── main/java/panomete/flowerogate/
│   ├── FlowerogateApplication.java          # @SpringBootApplication
│   ├── config/                               # @Configuration classes
│   │   ├── SecurityConfig.java               # WebFlux security
│   │   ├── CorsConfig.java                   # CORS with wildcard patterns
│   │   ├── GatewayConfig.java                # Route config
│   │   ├── RateLimiterConfig.java            # Key resolvers
│   │   ├── ResilientRedisRateLimiter.java    # Fail-open Valkey rate limiter
│   │   ├── CircuitBreakerConfig.java         # Resilience4j
│   │   └── ObservabilityConfig.java          # Micrometer tags
│   ├── filter/                               # WebFilter implementations
│   │   ├── JwtClaimHeaderFilter.java         # JWT claims → HTTP headers
│   │   ├── TraceIdFilter.java                # W3C traceparent
│   │   ├── RequestLoggingFilter.java         # Structured JSON logging
│   │   ├── RateLimitResponseFilter.java      # 429 response
│   │   ├── OAuth2RedirectParamFilter.java    # Post-login redirect
│   │   └── SensitiveDataMasker.java          # Secret masking
│   ├── controller/                           # @RestController classes
│   │   ├── FallbackController.java           # 404 + circuit breaker
│   │   └── RateLimitAdminController.java     # Rate limit admin
│   └── exception/                            # @RestControllerAdvice
│       └── GatewayExceptionHandler.java      # Global error handler
└── test/java/panomete/flowerogate/
    ├── FlowerogateApplicationTests.java      # Context load test
    ├── gateway/
    │   ├── RouteTests.java                   # Route matching tests
    │   ├── SecurityTests.java                # Auth enforcement tests
    │   └── TestSecurityConfig.java           # Permissive test security
    └── support/
        └── JwtTestHelper.java                # JWT factory for tests
```

## 6. Universal Rules (All Languages)

| Rule | Rationale |
|------|----------|
| **No hardcoded secrets** | Credentials → env vars or secret manager |
| **No commented-out code** | Delete it — git history preserves it |
| **No magic numbers** | Extract to named constants |
| **Functions do one thing** | If you need "and" in the name, split it |
| **Tests cover happy path + edge cases** | Error paths are as important as success paths |
| **Meaningful names** | `createValidJwt()` not `cvj()`, `jwkSetJson` not `j` |
| **Log at the right level** | INFO for requests, DEBUG for details, ERROR for failures |

## 7. Linting Status

| Tool | Status | Notes |
|------|:---:|-------|
| Checkstyle | Not configured | Recommended for consistent formatting |
| SpotBugs | Not configured | Recommended for static analysis |
| Spotless | Not configured | Recommended for auto-formatting |
| OWASP Dependency-Check | Not configured | Recommended for vulnerability scanning |

> **Recommendation:** Add Checkstyle + Spotless as a follow-up task. The codebase follows Google Java Style Guide conventions already — formal enforcement would prevent drift.

---

## Related Documents

| Document | Relationship |
|----------|-------------|
| [[031_README_developer_guide]] | Project overview and structure |
| [[036_code_review_records]] | Standards enforced during review |
| [[034_commit_messages_changelog]] | Commit message conventions |
