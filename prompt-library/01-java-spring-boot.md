# Java & Spring Boot — Daily Prompts

> Tech stack: Java 17+, Spring Boot 3.x, Spring Data JPA, Spring Security, Spring Cloud, Maven/Gradle

---

## Feature Development

### Create a REST endpoint
```
Create a Spring Boot REST endpoint for [operation, e.g., "creating an order"].

Entity: [describe fields]
Requirements:
- Controller → Service → Repository layered architecture
- DTO with Jakarta Bean Validation (@NotNull, @Size, etc.)
- MapStruct or manual mapping between DTO and Entity
- Proper HTTP status codes (201 Created, 400, 409, 500)
- Global exception handler with Problem Details (RFC 7807)
- Slf4j logging at entry/exit/error points

Return all files with full package paths.
```

### Add a scheduled job
```
Implement a Spring Boot @Scheduled job that [describe task, e.g., "purges soft-deleted
records older than 30 days"].

Requirements:
- Configurable cron expression via application.yml
- ShedLock or database-based distributed lock (multiple pods in AKS)
- Graceful shutdown handling
- Batch processing with configurable page size
- Error handling — log and continue, don't abort entire batch
- Metrics counter for processed/failed records

Show the config, lock table DDL, and the scheduled class.
```

### Implement pagination and filtering
```
Add pagination, sorting, and dynamic filtering to the [entity] list endpoint.

Requirements:
- Spring Data JPA Specification for dynamic WHERE clauses
- Page<DTO> response with totalElements, totalPages, currentPage
- Default sort by [field] DESC
- Filter parameters as @RequestParam (optional)
- Cacheable if filter params haven't changed (ETag or Spring Cache)

Show: Specification class, repository, service, controller, and response wrapper.
```

### Implement async processing
```
Refactor [operation] to run asynchronously in Spring Boot.

Requirements:
- @Async with custom ThreadPoolTaskExecutor (core=5, max=20, queue=100)
- CompletableFuture return type
- Proper exception handling (AsyncUncaughtExceptionHandler)
- MDC context propagation for tracing
- Graceful shutdown (wait for in-flight tasks)
- Timeout handling

Show: async config, service method, and caller code.
```

---

## Configuration & Profiles

### Externalize configuration
```
I need to externalize these properties for [feature]: [list properties].

Current setup: Spring Boot 3.x on AKS with Azure App Configuration.
Show:
- @ConfigurationProperties class with validation
- application.yml with defaults
- Profile overrides for dev/staging/prod
- How to load from Azure App Configuration
- How to refresh without restart (@RefreshScope)
```

### Add feature flags
```
Implement a feature flag for [feature name] in Spring Boot.

Requirements:
- Toggle via config (Azure App Configuration or application.yml)
- @ConditionalOnProperty or custom annotation
- Logging when flag is checked
- Safe default (feature OFF) if config is missing
- A/B rollout support (percentage-based activation)

Show the implementation without adding a third-party feature flag library.
```

---

## Spring Security

### Secure an endpoint with JWT
```
Secure the [endpoint] with JWT-based authentication in Spring Boot 3.x.

Requirements:
- JwtDecoder bean configured for [issuer, e.g., Azure AD / Auth0]
- Role-based access: @PreAuthorize("hasRole('ADMIN')")
- Extract custom claims (userId, tenantId) into a SecurityContext holder
- 401 for missing/invalid token, 403 for insufficient role
- Permit /actuator/health without auth
- Unit test with @WithMockUser

Show: SecurityFilterChain config, JWT config, and test.
```

---

## Performance & Optimization

### Fix N+1 query problem
```
I suspect an N+1 query issue in this code:
[paste entity relationships and repository method]

Diagnose the problem and show fixes using:
1. @EntityGraph (preferred for simple cases)
2. JOIN FETCH in JPQL
3. Batch fetching (@BatchSize)

Compare the SQL generated before and after. Show how to verify with
spring.jpa.show-sql=true and Hibernate statistics.
```

### Optimize a slow endpoint
```
This endpoint is slow (~[X]ms, target <[Y]ms):
[paste controller/service code or describe the flow]

Analyze and suggest optimizations across all layers:
- Controller: async, compression, response caching
- Service: algorithm, unnecessary DB calls, N+1
- Repository: indexes, query rewrite, projection (interface-based)
- Infra: connection pool size, Redis cache, read replica
- Measure: how to pinpoint with Micrometer timer + Spring AOP

Rank suggestions by impact (high/medium/low) and effort.
```

### Connection pool tuning
```
Help me tune the HikariCP connection pool for my Spring Boot app.

Context:
- MySQL 8 on Azure Database for MySQL Flexible Server
- [N] pods in AKS, each pod gets its own pool
- Peak load: [X] concurrent requests per pod
- Average query time: [Y]ms
- MySQL max_connections: [Z]

Calculate optimal: maximumPoolSize, minimumIdle, connectionTimeout,
maxLifetime, idleTimeout. Explain the reasoning behind each value.
Show the application.yml configuration.
```

---

## Error Handling

### Global exception handling
```
Create a comprehensive global exception handler for my Spring Boot REST API.

Cover:
- Custom business exceptions (ResourceNotFoundException, DuplicateException,
  BusinessRuleViolationException) → 404, 409, 422
- Jakarta validation errors (MethodArgumentNotValidException) → 400
- Constraint violation (ConstraintViolationException) → 400
- Access denied → 403
- Unexpected errors → 500 with correlation ID, no stack trace in response
- Problem Details format (RFC 7807: type, title, status, detail, instance)
- Slf4j structured logging with correlation ID

Show: exception classes, @ControllerAdvice, error response DTO, and example log output.
```

---

## Spring Boot Upgrade / Migration

### Upgrade Spring Boot version
```
I need to upgrade from Spring Boot [current version] to [target version].

My app uses: [list dependencies — JPA, Security, Kafka, Actuator, etc.]

Provide:
1. Breaking changes I need to handle
2. Deprecated APIs to replace
3. New features I should adopt
4. Step-by-step upgrade checklist
5. Regex/sed commands for bulk renames if any packages changed
6. Test strategy to verify nothing broke
```

---

## Quick One-Liners

```
Show me the Spring Boot 3.x way to [do something]. I'm migrating from [old approach].
```

```
What's the difference between [Spring annotation A] and [Spring annotation B]?
When should I use each? Show examples.
```

```
Write a @ConfigurationProperties class for these YAML properties: [paste YAML block]
```

```
Convert this imperative Spring code to reactive using WebFlux: [paste code]
```

```
Generate the application.yml for connecting to [Azure MySQL / Azure Blob / Kafka / Redis]
with connection pooling, retry, and SSL. Include comments explaining each property.
```

---

*Copy → Replace placeholders → Paste into AI → Get production-ready code.*
