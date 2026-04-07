# Testing — Daily Prompts

> Tech stack: JUnit 5, Mockito, MockMvc, Testcontainers (MySQL), AssertJ, Spring Boot Test

---

## Unit Tests

### Test a service class
```
Write JUnit 5 + Mockito unit tests for this service class:
[paste service class]

Requirements:
- @ExtendWith(MockitoExtension.class), no Spring context
- @Mock for dependencies, @InjectMocks for the service
- Test cases:
  • Happy path for each public method
  • Null/empty input handling
  • Business rule violation scenarios
  • Exception propagation from dependencies
- Use AssertJ assertions (assertThat, assertThatThrownBy)
- Use @DisplayName with descriptive test names
- Use @ParameterizedTest where multiple inputs map to the same logic
- Verify mock interactions with verify() — no unnecessary verifyNoMoreInteractions
```

### Test a mapper/converter
```
Write unit tests for this mapper that converts [DTO ↔ Entity]:
[paste mapper class]

Cover:
- All fields mapped correctly
- Null source → appropriate handling (null or empty object)
- Null individual fields → no NPE
- Collection fields (empty list, populated list)
- Nested objects
- Edge cases: max-length strings, boundary dates, zero/negative numbers

Use AssertJ's recursive comparison: assertThat(actual).usingRecursiveComparison().isEqualTo(expected)
```

### Test exception scenarios
```
Write tests that verify [service/component] throws the correct exceptions:

Test these scenarios:
- [scenario 1] → should throw [ExceptionType] with message containing "[text]"
- [scenario 2] → should throw [ExceptionType]
- [scenario 3] → should NOT throw any exception

Use assertThatThrownBy() or assertThrows(). Verify the exception message,
error code, and that no side effects occurred (e.g., nothing was saved).
```

---

## Integration Tests

### Test a REST controller end-to-end
```
Write a Spring Boot integration test for this controller:
[paste controller class]

Setup:
- @SpringBootTest(webEnvironment = RANDOM_PORT) or @WebMvcTest([Controller].class)
- MockMvc for HTTP calls
- @MockBean for service layer (or real service with Testcontainers DB)
- ObjectMapper for JSON serialization

Test cases:
- POST /[resource] with valid body → 201, verify response body and Location header
- POST /[resource] with invalid body → 400, verify validation error details
- GET /[resource]/{id} → 200 with correct data
- GET /[resource]/{id} with non-existent id → 404
- GET /[resource]?page=0&size=10 → 200 with paginated response
- PUT /[resource]/{id} → 200, verify update applied
- DELETE /[resource]/{id} → 204
- Endpoints without auth token → 401

Show complete test class with @BeforeEach setup and helper methods.
```

### Test with Testcontainers (MySQL)
```
Write an integration test for [repository/service] using Testcontainers MySQL.

Setup:
- @Testcontainers + @Container with MySQLContainer
- @DynamicPropertySource to override datasource URL, username, password
- Flyway or Liquibase migration runs automatically
- @Transactional on test class for rollback after each test
  (or @Sql for explicit setup/teardown)

Test cases:
- Save and retrieve entity
- Find by [custom query method]
- Pagination and sorting
- Unique constraint violation → DataIntegrityViolationException
- Concurrent updates (optimistic locking with @Version)

Show the full test class including container setup.
```

### Test Kafka producer/consumer
```
Write an integration test for my Kafka [producer/consumer] using:
- EmbeddedKafka (@EmbeddedKafka annotation) or Testcontainers Kafka
- Spring Kafka Test (KafkaTestUtils)

Producer test:
- Send message → verify it lands on the correct topic with correct key/value
- Verify Avro serialization if using Schema Registry

Consumer test:
- Publish test message to topic → verify consumer processes it
- Verify database state after consumption
- Test deserialization error handling (poison pill)
- Test retry and DLT (Dead Letter Topic) flow

Show: test config, embedded broker setup, and assertions.
```

---

## MockMvc Patterns

### Reusable test helpers
```
Create a reusable MockMvc test utility class for my Spring Boot tests.

Include helper methods for:
- performPost(url, body) → ResultActions (with content-type, accept headers)
- performGet(url) → ResultActions
- performPut(url, body) → ResultActions
- performDelete(url) → ResultActions
- withAuth(token) → adds Authorization header
- expectValidationError(field, message) → checks error response body
- expectProblemDetail(status, title) → checks RFC 7807 response
- parseResponse(ResultActions, Class<T>) → deserialize response body

Show the utility class and example usage in a test.
```

---

## Test Data Builders

### Create test fixtures
```
Create a test data builder (Builder pattern) for [entity/DTO]:
[paste entity class]

Requirements:
- Default values for all fields (valid, realistic data)
- Fluent API: TestOrder.builder().withStatus(SHIPPED).withAmount(99.99).build()
- .random() method for fully random but valid data (use Instancio or manual)
- Support building related entities (e.g., order with line items)
- Reusable across unit and integration tests

Show: builder class and 3 example usages for different test scenarios.
```

---

## Test Strategy

### Coverage analysis
```
Analyze this class and tell me exactly which tests I need to write:
[paste class]

For each public method, list:
- Happy path scenarios
- Edge cases (null, empty, boundary)
- Error paths (exceptions, validation failures)
- Interaction verification (what should be called, what shouldn't)
- Which tests should be unit tests vs integration tests

Estimate total test count and approximate time to write them.
```

### What NOT to test
```
Review these test cases and identify any that are low-value or wasteful:
[paste test class]

Flag tests that:
- Only test framework behavior (e.g., testing that Spring DI works)
- Are tautological (mock returns X, assert X)
- Have excessive mocking that makes them brittle
- Test private methods indirectly through reflection
- Are duplicated across unit and integration tests

Suggest which to keep, remove, or refactor.
```

---

## Quick One-Liners

```
Write a @ParameterizedTest with @CsvSource for [method] with these input/output pairs:
[list pairs]
```

```
How do I mock a static method in [class] using Mockito 5.x? Show before/after.
```

```
Write a custom AssertJ assertion for [domain object] that checks [business rules].
```

```
Generate test data SQL (@Sql) for setting up [scenario] in my MySQL Testcontainer.
```

```
Convert this @SpringBootTest to a lighter @WebMvcTest. What do I need to mock?
```

```
Show me how to test a @Scheduled method without waiting for the cron trigger.
```

---

*Good tests catch bugs before production. Great tests document your system's behavior.*
