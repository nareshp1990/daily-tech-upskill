# Tutorial 6: Advanced Patterns & Tips

## Table of Contents

- [Custom Instructions Deep Dive](#custom-instructions-deep-dive)
- [Prompt Engineering for Better Completions](#prompt-engineering-for-better-completions)
- [Copilot with Test-Driven Development](#copilot-with-test-driven-development)
- [Learning New Frameworks with Copilot](#learning-new-frameworks-with-copilot)
- [Copilot Extensions and Agents](#copilot-extensions-and-agents)
- [Model Selection](#model-selection)
- [Copilot Metrics and Usage Analytics](#copilot-metrics-and-usage-analytics)
- [Security Considerations](#security-considerations)
- [When NOT to Use Copilot](#when-not-to-use-copilot)
- [Performance Tips](#performance-tips)
- [Try This Exercises](#try-this-exercises)

---

## Custom Instructions Deep Dive

### File Location and Discovery

Copilot reads instructions from `.github/copilot-instructions.md` in the root of your repository. This file is used for:
- Code completions
- Chat responses
- PR summaries (on some plans)

### Instruction Anatomy

Structure your instructions for maximum impact:

```markdown
# Copilot Instructions

## Language and Runtime
- Java 21, Spring Boot 3.3.x, Spring Cloud 2024.x
- Use virtual threads (Project Loom) for blocking I/O operations
- Prefer records for immutable data carriers
- Use sealed interfaces for type hierarchies

## Architecture
- Domain-Driven Design with hexagonal architecture
- Aggregates own their invariants — no anemic domain models
- Use domain events for cross-aggregate communication
- Ports (interfaces) in domain layer, adapters (implementations) in infrastructure

## Code Generation Rules
- Constructor injection only — never @Autowired on fields
- Use @Slf4j for logging, always use parameterized messages: log.info("Order {} created", orderId)
- Favor composition over inheritance
- Methods should do one thing — max 20 lines
- No utility classes with static methods — use injectable services

## Error Handling
- Use a sealed exception hierarchy: DomainException > OrderException, PaymentException, etc.
- Return ProblemDetail (RFC 7807) from REST endpoints
- Log stack traces only at ERROR level
- Wrap checked exceptions at service boundaries

## Testing Patterns
- Test class: {ClassName}Test (unit), {ClassName}IntegrationTest (integration)
- Test method naming: should_{expectedBehavior}_when_{condition}
- Use @Nested for grouping related tests
- Use TestFixtures class for shared test data builders
- AssertJ for all assertions

## Naming Conventions
- Packages: com.example.{service}.{layer}.{feature}
- REST paths: /api/v1/{resource} (plural nouns)
- Kafka topics: {domain}.{event-type}.v{version} (e.g., order.created.v1)
- Config properties: app.{feature}.{setting}

## Do NOT generate
- Lombok annotations except @Slf4j
- Field-level @Autowired
- Raw SQL string concatenation
- System.out.println or System.err.println
- Wildcard imports (import java.util.*)
```

### Workspace-Level Instructions (VS Code)

In addition to repository-level instructions, you can create workspace-level instructions in VS Code:

**`.vscode/copilot-instructions.md`** (local to workspace, can be gitignored):

```markdown
# Local Development Instructions

## Database
- Local MySQL runs on port 3307 (Docker)
- Schema: order_service_dev
- User: dev / password: dev

## Kafka
- Local Kafka broker: localhost:9092
- No authentication in dev mode

## Testing
- Run integration tests with: ./mvnw test -Pintegration
- Testcontainers auto-detects Docker
```

### Per-File Type Instructions

Use section headers to scope instructions:

```markdown
## When generating Java code
- Use Java 21 features (records, sealed classes, pattern matching)

## When generating SQL
- Target MySQL 8.x syntax
- Use InnoDB engine
- Include IF NOT EXISTS for CREATE TABLE

## When generating Dockerfile
- Use eclipse-temurin base images
- Multi-stage builds always
- Non-root user in final stage

## When generating Terraform
- Use Azure provider (azurerm) 3.x
- Backend: azurerm (Azure Storage Account)
- Tag all resources with: environment, team, cost-center

## When generating YAML (Kubernetes)
- Include resource requests and limits
- Include readiness and liveness probes
- Use rolling update strategy
```

---

## Prompt Engineering for Better Completions

### Pattern 1: The Anchor Comment

Write a detailed comment that anchors Copilot's understanding:

```java
/**
 * Processes a batch of order events from Kafka.
 *
 * Algorithm:
 * 1. Deduplicate events by event ID (idempotency)
 * 2. Group events by order ID
 * 3. For each order, apply events in chronological order
 * 4. Persist updated order states in a single batch
 * 5. Acknowledge the Kafka offsets
 *
 * Error handling:
 * - Transient errors: throw to trigger Kafka retry
 * - Permanent errors: log, send to DLQ, continue with next order
 *
 * Performance: processes ~5000 events/second on a single partition.
 */
public void processBatch(List<ConsumerRecord<String, OrderEvent>> records) {
```

### Pattern 2: The Example-Driven Comment

Show an example of input/output:

```java
/**
 * Parses a shipping address from a single-line format.
 *
 * Input:  "123 Main St, Apt 4B, Springfield, IL, 62701, US"
 * Output: Address(street="123 Main St", unit="Apt 4B", city="Springfield",
 *         state="IL", zip="62701", country="US")
 *
 * Rules:
 * - Unit/apartment is optional
 * - Country defaults to "US" if not provided
 * - Throws InvalidAddressException if street or city is missing
 */
public Address parseAddress(String rawAddress) {
```

### Pattern 3: The Specification Comment

Describe the contract precisely:

```java
// Calculate shipping cost.
// Rules:
// - Base rate: $5.99
// - Per kg: $1.50 (round up to nearest kg)
// - Express multiplier: 2.5x
// - Free shipping for orders over $100
// - Maximum shipping cost: $49.99
// - Returns BigDecimal rounded to 2 decimal places
public BigDecimal calculateShippingCost(Order order, ShippingType type) {
```

### Pattern 4: The Peer Code Pattern

Write one example, and Copilot infers the pattern for subsequent ones:

```java
// Mapper method 1 — write this manually
public OrderResponse toResponse(Order order) {
    return new OrderResponse(
        order.getId(),
        order.getCustomerId(),
        order.getStatus().name(),
        order.getTotalAmount(),
        order.getCreatedAt()
    );
}

// Mapper method 2 — Copilot follows the pattern
public CustomerResponse toResponse(Customer customer) {
    // Copilot generates this following the same style
```

### Pattern 5: The Negative Constraint

Tell Copilot what NOT to do:

```java
// Convert entity to DTO
// Do NOT use ModelMapper or MapStruct
// Do NOT use reflection
// Use explicit field-by-field mapping
public ProductResponse toResponse(Product product) {
```

---

## Copilot with Test-Driven Development

### The TDD + Copilot Workflow

TDD and Copilot work extremely well together. The pattern:

1. **You write the test** (Red)
2. **Copilot writes the implementation** (Green)
3. **You refactor together** (Refactor)

### Step 1: Write the Test First

```java
@Test
void should_calculateDiscount_when_customerIsGoldTier() {
    // Arrange
    var customer = TestFixtures.goldCustomer();
    var order = TestFixtures.orderWithTotal(new BigDecimal("200.00"));

    // Act
    BigDecimal discount = discountService.calculateDiscount(customer, order);

    // Assert
    assertThat(discount).isEqualByComparingTo(new BigDecimal("20.00")); // 10% for gold
}

@Test
void should_returnZeroDiscount_when_customerIsStandardTier() {
    var customer = TestFixtures.standardCustomer();
    var order = TestFixtures.orderWithTotal(new BigDecimal("200.00"));

    BigDecimal discount = discountService.calculateDiscount(customer, order);

    assertThat(discount).isEqualByComparingTo(BigDecimal.ZERO);
}

@Test
void should_capDiscountAt50_when_calculatedDiscountExceedsCap() {
    var customer = TestFixtures.platinumCustomer(); // 20% discount
    var order = TestFixtures.orderWithTotal(new BigDecimal("500.00")); // 20% = $100

    BigDecimal discount = discountService.calculateDiscount(customer, order);

    assertThat(discount).isEqualByComparingTo(new BigDecimal("50.00")); // Capped at $50
}
```

### Step 2: Let Copilot Implement

Open `DiscountServiceImpl.java`. Copilot sees the tests (if the test file is in an open tab) and generates an implementation that passes all three tests.

```java
@Service
public class DiscountServiceImpl implements DiscountService {

    private static final BigDecimal MAX_DISCOUNT = new BigDecimal("50.00");

    @Override
    public BigDecimal calculateDiscount(Customer customer, Order order) {
        // Copilot generates this based on your tests
        BigDecimal rate = switch (customer.tier()) {
            case PLATINUM -> new BigDecimal("0.20");
            case GOLD -> new BigDecimal("0.10");
            case SILVER -> new BigDecimal("0.05");
            case STANDARD -> BigDecimal.ZERO;
        };

        BigDecimal discount = order.totalAmount().multiply(rate);
        return discount.min(MAX_DISCOUNT).setScale(2, RoundingMode.HALF_UP);
    }
}
```

### Step 3: Refactor

Use inline chat: "Extract the discount rates into a configuration map" or "Add caching for tier-based discount rates."

### Why TDD + Copilot Works

- Tests provide **precise specifications** — Copilot knows exactly what to build
- Tests constrain the solution — fewer ways to go wrong
- You maintain design control by writing tests first
- Copilot speeds up the implementation phase significantly
- The feedback loop (run tests) validates Copilot's output immediately

---

## Learning New Frameworks with Copilot

### Pattern: Ask, See, Understand

When learning something new (e.g., Spring Modulith, jOOQ, Quarkus):

#### Step 1: Ask in Chat

```
@workspace /explain What is Spring Modulith and how would I use it
in this project to enforce module boundaries?
```

#### Step 2: Generate an Example

```
Generate an example of a Spring Modulith application module for
the "order" domain. Show the module structure, the @ApplicationModule
annotation, and how to publish domain events to other modules.
```

#### Step 3: Write Tests to Understand Behavior

```
/tests Generate a test using @ApplicationModuleTest that verifies
the order module publishes an OrderCreated event when a new order is placed.
```

#### Step 4: Compare with Your Existing Patterns

```
Compare Spring Modulith's event system with our current Kafka-based
event publishing. What are the pros and cons for our use case?
```

### Using Copilot as a Learning Companion

```
# In Chat:
I'm learning about virtual threads in Java 21.
Explain when I should use them in a Spring Boot application.
Show me how to configure Spring Boot to use virtual threads
for HTTP request handling and Kafka consumers.
What are the pitfalls I should watch out for?
```

---

## Copilot Extensions and Agents

### What Are Copilot Extensions?

Copilot Extensions are third-party integrations that add specialized capabilities to Copilot Chat. You invoke them using `@` in chat.

### Available Extensions (as of 2025)

| Extension | What It Does |
|-----------|-------------|
| `@docker` | Docker-specific assistance, Dockerfile generation, compose help |
| `@azure` | Azure resource management, ARM/Bicep template generation |
| `@sentry` | Error investigation using Sentry data |
| `@datadog` | Metrics and monitoring queries |
| `@hashicorp` | Terraform and Vault assistance |
| `@octopus` | Octopus Deploy pipeline help |
| `@mermaid` | Generate Mermaid diagrams from code |

### Using Extensions

```
@docker Generate a multi-stage Dockerfile optimized for a Spring Boot 3
application with Java 21. Include a non-root user and health check.

@azure Create a Bicep template for an AKS cluster with 3 node pools,
container insights enabled, and private cluster networking.

@hashicorp Write a Terraform module for Azure Key Vault with
access policies for my AKS cluster's managed identity.
```

### Installing Extensions

```
1. Go to GitHub.com → Your profile → Settings → Copilot → Extensions
2. Browse the marketplace
3. Install and authorize the extension
4. It becomes available as @extension-name in Copilot Chat
```

### Copilot Agents (Preview)

Copilot Agents are autonomous agents that can perform multi-step tasks:

- **@workspace agent**: Can search code, read files, and propose changes across multiple files
- **Custom agents**: Organizations can build custom agents using the Copilot Extensions API

```
@workspace Create a new microservice for inventory management.
It should have:
- A REST API for checking and updating stock levels
- A Kafka consumer for order events
- MySQL for persistence
- Standard project structure matching our other services
```

The agent creates multiple files, configures dependencies, and generates a working skeleton.

---

## Model Selection

### Available Models in Copilot

As of 2025, Copilot supports multiple models:

| Model | Best For | Speed |
|-------|----------|-------|
| **GPT-4o** | General coding, chat, explanations | Fast |
| **Claude 3.5 Sonnet** | Complex reasoning, refactoring, long code | Medium |
| **GPT-4o mini** | Quick completions, simple edits | Very fast |
| **o1-preview** | Complex logic, algorithm design | Slow |
| **o1-mini** | Math, logical reasoning | Medium |

### Switching Models

**VS Code Chat**:
1. Open Copilot Chat
2. Click the model name at the top of the chat panel
3. Select a different model from the dropdown

**Keyboard**: No shortcut — use the dropdown.

### When to Use Which Model

```
Simple code completion → GPT-4o mini (fastest)
Chat explanations → GPT-4o (good balance)
Complex refactoring → Claude 3.5 Sonnet (best reasoning)
Algorithm design → o1-preview (deep thinking)
Quick inline chat → GPT-4o mini
```

### Model Selection Strategy

```
Morning code review:     GPT-4o (fast, good explanations)
Active development:      GPT-4o mini (fast completions, low latency)
Complex debugging:       Claude 3.5 Sonnet (better at tracing logic)
Architecture planning:   o1-preview (deep reasoning)
Test generation:         GPT-4o (good balance of speed and quality)
```

---

## Copilot Metrics and Usage Analytics

### Individual Metrics

In VS Code, check your personal usage:

```
1. Ctrl+Shift+P → "GitHub Copilot: Show Usage"
2. View: acceptance rate, suggestions per day, languages used
```

### Organization Metrics (Business/Enterprise)

Admins can view at: `github.com/organizations/{org}/settings/copilot`

Available metrics:
- **Acceptance rate**: % of suggestions accepted (target: 25-35%)
- **Active users**: How many developers are using Copilot
- **Suggestions per day**: Volume of completions offered
- **Languages**: Which languages generate the most suggestions
- **Top users**: Who benefits most from Copilot
- **Lines of code accepted**: Total LOC from Copilot suggestions

### Interpreting Metrics

| Metric | Good | Investigate |
|--------|------|-------------|
| Acceptance rate | 25-35% | Below 15% (poor context) or above 50% (over-reliance) |
| Active users | 80%+ of licensed users | Below 50% (need training) |
| Daily suggestions | 50-200 per user | Below 20 (not engaged) |

### Improving Team Adoption

Low acceptance rates often mean:
1. Developers do not know the shortcuts (training needed)
2. No `.github/copilot-instructions.md` (add project context)
3. Copilot is disabled for key file types (check settings)
4. Developers dismiss too quickly (encourage partial acceptance)

---

## Security Considerations

### What Copilot Sends to the Cloud

- Current file content (for completions)
- Open tab contents (for context)
- Chat messages and referenced code
- File paths and names

### What Copilot Does NOT Send

- Your entire repository
- Files you have not opened
- Git history
- Environment variables
- Contents of `.gitignore`d files (though file names may be sent)

### Data Handling by Plan

| Aspect | Individual | Business | Enterprise |
|--------|-----------|----------|------------|
| Code used for training | Opt-out available | Never | Never |
| Data retained | 28 days | Not retained | Not retained |
| Telemetry | Standard | Controlled | Controlled |
| Content exclusions | No | Yes | Yes |
| IP indemnity | No | Yes | Yes |
| Audit logs | No | Yes | Yes |

### Content Exclusions (Business/Enterprise)

Exclude sensitive files from Copilot:

```
Organization Settings → Copilot → Content Exclusions:

Add paths:
  - "**/.env*"
  - "**/secrets/**"
  - "**/credentials/**"
  - "**/*-secret.yaml"
  - "**/terraform.tfvars"
  - "**/key-vault/**"
```

### Best Practices

1. **Enable content exclusions** for files containing secrets
2. **Never type secrets in comments** — Copilot may echo them in suggestions
3. **Review all generated code** — Copilot may suggest insecure patterns
4. **Disable for sensitive repositories** if your security policy requires it
5. **Use the Business plan** for professional work — IP indemnity matters

---

## When NOT to Use Copilot

### Do Not Use Copilot For:

1. **Secrets and credentials**: Never type passwords, API keys, tokens, or connection strings near Copilot. It may memorize and suggest them in other contexts.

2. **Cryptographic implementations**: Do not let Copilot generate encryption, hashing, or key management code. Use established libraries (Bouncy Castle, Tink).

3. **Security-critical authorization logic**: Write access control logic manually. Copilot might miss edge cases that create privilege escalation.

4. **Compliance-sensitive code**: HIPAA, PCI-DSS, SOX-regulated logic should be written and reviewed manually.

5. **Complex business rules you do not fully understand**: If you cannot verify the output, do not use Copilot. You are responsible for every line that ships.

6. **Legal/licensing-sensitive code**: If your organization has strict IP policies, be aware that Copilot may suggest code similar to open-source projects.

### Signs You Are Over-Relying on Copilot

- Accepting suggestions without reading them
- Unable to write code when Copilot is down
- Skipping code review because "Copilot wrote it"
- Not writing tests because the code "looks right"
- Accepting 80%+ of suggestions (healthy is 25-35%)

---

## Performance Tips

### Speed Up Suggestions

1. **Close unnecessary tabs**: Fewer open files = faster context assembly
2. **Keep files small**: Copilot processes the entire current file
3. **Disable for file types you do not need**: Reduces background processing
4. **Use GPT-4o mini for completions**: Faster model, good enough for most completions

### Improve Suggestion Quality

1. **Be in the right file**: Copilot uses the file name and path heavily
2. **Write the method signature first**: Parameters give strong context
3. **Keep related files open**: Entity + Repository + Service in tabs
4. **Write a good first method**: Copilot learns the pattern for the rest
5. **Use specific types**: `List<Order>` is better than `List<Object>` for context

### Reduce Noise

1. **Do not fight Copilot**: If suggestions are consistently wrong, the context is misleading. Fix the context (imports, class structure, comments) rather than cycling through suggestions.
2. **Dismiss quickly**: Press `Esc` immediately if the suggestion starts wrong. Type a few more characters and wait for a new suggestion.
3. **Use Tab stops**: In VS Code, Copilot suggestions work with tab stops in snippets. Accept the snippet first, then fill in the blanks.

---

## Try This Exercises

### Exercise 1: Custom Instructions Optimization

1. Create `.github/copilot-instructions.md` in your main project
2. Add sections for: Language, Architecture, Naming, Error Handling, Testing, Do NOT Generate
3. Open a Java file and generate a new method — observe if instructions are followed
4. Refine the instructions based on results
5. Iterate 3 times until suggestions consistently match your style

### Exercise 2: TDD with Copilot

1. Pick a feature to implement (e.g., "calculate order discount based on customer tier")
2. Write 5 test cases first (happy path + edge cases)
3. Open the implementation file — let Copilot generate the code
4. Run the tests — do they pass?
5. If not, use Chat to fix: `/fix these tests are failing because...`

### Exercise 3: Model Comparison

1. Open a complex method in Chat
2. Ask "Refactor this to be more readable" with GPT-4o
3. Switch to Claude 3.5 Sonnet and ask the same question
4. Compare the two suggestions — which is better for your use case?
5. Try o1-preview for an algorithm problem

### Exercise 4: Security Audit

1. Review your `.github/copilot-instructions.md` for security rules
2. Check that content exclusions are set up for `.env` files
3. Try typing a comment like `// database password:` and see if Copilot suggests anything sensitive
4. Add explicit rules: "Never generate hardcoded credentials"

### Exercise 5: Framework Exploration

1. Pick a framework you want to learn (e.g., Spring Modulith, jOOQ, Quarkus)
2. Ask Copilot Chat: "Explain [framework] and how it compares to what we use"
3. Ask: "Generate a minimal working example in our project structure"
4. Ask: "Write tests to demonstrate the key features"
5. Ask: "What are the migration steps from our current approach?"

---

## Next Steps

You now know advanced Copilot patterns. Complete the series with [Tutorial 7: Daily Workflow Playbook](07-daily-workflow-playbook.md) — a practical reference for using Copilot throughout your workday.
