# IntelliJ IDEA AI Tools Mastery for Java/Spring Boot Engineers

> A comprehensive, practical guide to every AI-powered tool available in IntelliJ IDEA — tailored for senior Java/Spring Boot developers.

---

## Table of Contents

1. [Overview: AI Tools Landscape in IntelliJ IDEA](#1-overview-ai-tools-landscape-in-intellij-idea)
2. [JetBrains AI Assistant](#2-jetbrains-ai-assistant)
3. [GitHub Copilot in IntelliJ](#3-github-copilot-in-intellij)
4. [Claude Code in IntelliJ](#4-claude-code-in-intellij)
5. [Google Gemini Code Assist in IntelliJ](#5-google-gemini-code-assist-in-intellij)
6. [Amazon Q Developer in IntelliJ](#6-amazon-q-developer-in-intellij)
7. [Tabnine in IntelliJ](#7-tabnine-in-intellij)
8. [Comparison Matrix](#8-comparison-matrix)
9. [Recommended Setup for Java/Spring Boot Development](#9-recommended-setup-for-javaspring-boot-development)
10. [Keyboard Shortcuts Reference](#10-keyboard-shortcuts-reference)
11. [Try This Exercises](#11-try-this-exercises)
12. [Tips and Gotchas](#12-tips-and-gotchas)

---

## 1. Overview: AI Tools Landscape in IntelliJ IDEA

IntelliJ IDEA has become the primary battleground for AI coding assistants. As of 2025, six major AI tools integrate directly into the IDE, each with different strengths for Java/Spring Boot development.

### AI Tools Available in IntelliJ IDEA

| Tool | Type | Integration | Model(s) | Free Tier | Best For |
|------|------|-------------|-----------|-----------|----------|
| **JetBrains AI Assistant** | Native plugin | Deep IDE integration | JetBrains proprietary + GPT-4o, Claude | Limited free tier | Refactoring, inspections, commit messages |
| **GitHub Copilot** | Plugin | Ghost text + Chat panel | GPT-4o, Claude 3.5 Sonnet | Free for OSS / $10-$19/mo | Inline completions, broad language support |
| **Claude Code** | Terminal + Plugin | Terminal agent + JetBrains plugin | Claude Opus, Sonnet | Pay-per-use | Complex refactoring, multi-file changes, agentic tasks |
| **Google Gemini Code Assist** | Plugin | Completions + Chat | Gemini 2.0 | Free tier (individual) | GCP integration, large context window |
| **Amazon Q Developer** | Plugin | Completions + Chat + Security | Amazon proprietary | Free tier available | AWS integration, security scanning, Java transformations |
| **Tabnine** | Plugin | Completions + Chat | Tabnine proprietary | Free tier (basic) | Privacy-first, on-prem option, team learning |

### How They Integrate

```
IntelliJ IDEA
├── Editor Layer
│   ├── Ghost text completions (Copilot, AI Assistant, Gemini, Amazon Q, Tabnine)
│   ├── Inline suggestions on Tab
│   └── Multi-line block completions
├── Chat Panel (Tool Window)
│   ├── AI Assistant Chat
│   ├── Copilot Chat
│   ├── Gemini Chat
│   ├── Amazon Q Chat
│   └── Tabnine Chat
├── Context Menu (Right-click)
│   ├── Explain code
│   ├── Generate tests
│   ├── Refactor with AI
│   └── Find vulnerabilities
├── Terminal
│   ├── Claude Code CLI
│   └── AI-powered terminal commands
└── VCS Integration
    ├── AI commit messages
    ├── PR summaries
    └── Code review assistance
```

---

## 2. JetBrains AI Assistant

JetBrains AI Assistant is the native AI integration, meaning it has the deepest access to IntelliJ's code analysis engine — PSI trees, inspections, refactoring infrastructure, and project-wide indexing.

### Setup

1. **Install:** `Settings > Plugins > Marketplace > "AI Assistant"` (bundled in 2024.1+)
2. **Activate:** `Settings > Tools > AI Assistant > Enable AI Assistant`
3. **Subscription:** JetBrains AI Pro ($10/month, included with All Products Pack)
4. **Model selection:** `Settings > Tools > AI Assistant > Model` — choose between JetBrains models, GPT-4o, or Claude

### Core Features for Java Development

#### 2.1 Smart Code Completion

AI Assistant enhances IntelliJ's already-strong code completion with full-line and multi-line suggestions:

```java
@Service
@RequiredArgsConstructor
public class OrderService {

    private final OrderRepository orderRepository;
    private final KafkaTemplate<String, OrderEvent> kafkaTemplate;
    private final RedisTemplate<String, Order> redisTemplate;

    // Type: public Order cre...
    // AI completes the entire method:
    public Order createOrder(CreateOrderRequest request) {
        Order order = Order.builder()
                .customerId(request.getCustomerId())
                .items(request.getItems().stream()
                        .map(this::toOrderItem)
                        .toList())
                .status(OrderStatus.PENDING)
                .createdAt(Instant.now())
                .build();

        Order saved = orderRepository.save(order);

        kafkaTemplate.send("order-events",
                saved.getId().toString(),
                new OrderCreatedEvent(saved));

        redisTemplate.opsForValue().set(
                "order:" + saved.getId(), saved,
                Duration.ofHours(1));

        return saved;
    }
}
```

AI Assistant understands your field types and generates code using `KafkaTemplate` and `RedisTemplate` because it reads the class context.

#### 2.2 AI Chat Panel

Open with `Alt+Enter > AI Actions` or the AI tool window. The chat understands your full project structure.

**Practical prompts for Spring Boot:**

```
> Explain the transaction propagation in this @Transactional method
  and identify potential deadlock scenarios.

> Generate a Kafka consumer for OrderCreatedEvent that implements
  idempotent processing using a Redis deduplication key.

> Review this REST controller for OWASP Top 10 vulnerabilities.
```

#### 2.3 Generate Tests

Right-click any class or method > `AI Actions > Generate Tests`:

```java
// Input: your service method
public OrderSummary getOrderSummary(UUID orderId) {
    Order order = orderRepository.findById(orderId)
            .orElseThrow(() -> new OrderNotFoundException(orderId));
    List<Payment> payments = paymentClient.getPayments(orderId);
    return OrderSummary.of(order, payments);
}

// AI Assistant generates:
@ExtendWith(MockitoExtension.class)
class OrderServiceTest {

    @Mock private OrderRepository orderRepository;
    @Mock private PaymentClient paymentClient;
    @InjectMocks private OrderService orderService;

    @Test
    void getOrderSummary_existingOrder_returnsSummary() {
        UUID orderId = UUID.randomUUID();
        Order order = OrderFixtures.anOrder(orderId);
        List<Payment> payments = List.of(PaymentFixtures.aPayment());

        when(orderRepository.findById(orderId)).thenReturn(Optional.of(order));
        when(paymentClient.getPayments(orderId)).thenReturn(payments);

        OrderSummary result = orderService.getOrderSummary(orderId);

        assertThat(result.getOrderId()).isEqualTo(orderId);
        assertThat(result.getTotalPaid()).isEqualTo(payments.get(0).getAmount());
        verify(paymentClient).getPayments(orderId);
    }

    @Test
    void getOrderSummary_missingOrder_throwsException() {
        UUID orderId = UUID.randomUUID();
        when(orderRepository.findById(orderId)).thenReturn(Optional.empty());

        assertThatThrownBy(() -> orderService.getOrderSummary(orderId))
                .isInstanceOf(OrderNotFoundException.class);
    }
}
```

#### 2.4 AI-Powered Inspections

AI Assistant adds semantic inspections beyond IntelliJ's built-in ones:

- Detects potential **N+1 query problems** in JPA entity traversals
- Flags **thread-safety issues** in Spring singleton beans
- Identifies **missing error handling** in reactive chains
- Suggests **Spring Boot configuration best practices**

#### 2.5 Commit Message Generation

When you open the Commit dialog (`Cmd+K`), click the "Generate Commit Message with AI" button. It reads the diff and produces a conventional commit message:

```
feat(order): add idempotent order creation with Redis deduplication

- Add Redis-based idempotency key check before persisting order
- Publish OrderCreatedEvent to Kafka after successful save
- Cache order in Redis with 1-hour TTL for read optimization
```

#### 2.6 Refactoring Suggestions

Select a code block > `AI Actions > Suggest Refactoring`:

```java
// Before: AI detects this as a candidate for the Strategy pattern
public BigDecimal calculateDiscount(Order order) {
    if (order.getCustomerType() == CustomerType.VIP) {
        return order.getTotal().multiply(new BigDecimal("0.20"));
    } else if (order.getCustomerType() == CustomerType.PREMIUM) {
        return order.getTotal().multiply(new BigDecimal("0.10"));
    } else if (order.getCustomerType() == CustomerType.LOYALTY) {
        return order.getTotal().multiply(new BigDecimal("0.05"));
    }
    return BigDecimal.ZERO;
}

// After: AI suggests Strategy + Spring DI
public interface DiscountStrategy {
    CustomerType getCustomerType();
    BigDecimal calculate(Order order);
}

@Component
public class VipDiscountStrategy implements DiscountStrategy {
    public CustomerType getCustomerType() { return CustomerType.VIP; }
    public BigDecimal calculate(Order order) {
        return order.getTotal().multiply(new BigDecimal("0.20"));
    }
}

@Service
@RequiredArgsConstructor
public class DiscountService {
    private final Map<CustomerType, DiscountStrategy> strategies;

    @Autowired
    public DiscountService(List<DiscountStrategy> strategyList) {
        this.strategies = strategyList.stream()
                .collect(Collectors.toMap(
                        DiscountStrategy::getCustomerType,
                        Function.identity()));
    }

    public BigDecimal calculateDiscount(Order order) {
        return strategies.getOrDefault(order.getCustomerType(),
                o -> BigDecimal.ZERO).calculate(order);
    }
}
```

---

## 3. GitHub Copilot in IntelliJ

GitHub Copilot is the most widely adopted AI coding assistant. Its IntelliJ integration provides ghost text completions and a full chat panel.

### Setup

1. **Install:** `Settings > Plugins > Marketplace > "GitHub Copilot"`
2. **Sign in:** Click the Copilot icon in the status bar > Sign in to GitHub
3. **Subscription:** Free (for verified OSS maintainers/students), Individual ($10/mo), Business ($19/mo)
4. **Verify:** Status bar shows a Copilot icon (green = active, red = error)

### 3.1 Ghost Text Completions

Copilot provides inline suggestions as you type. Press `Tab` to accept, `Esc` to dismiss:

```java
@RestController
@RequestMapping("/api/v1/products")
@RequiredArgsConstructor
public class ProductController {

    private final ProductService productService;

    // Type: @GetMapping and Copilot suggests the full method
    @GetMapping("/{id}")
    public ResponseEntity<ProductResponse> getProduct(@PathVariable UUID id) {
        return productService.findById(id)
                .map(ProductResponse::from)
                .map(ResponseEntity::ok)
                .orElse(ResponseEntity.notFound().build());
    }

    // Type: @PostMapping and Copilot suggests based on patterns in your codebase
    @PostMapping
    public ResponseEntity<ProductResponse> createProduct(
            @Valid @RequestBody CreateProductRequest request) {
        Product created = productService.create(request);
        URI location = URI.create("/api/v1/products/" + created.getId());
        return ResponseEntity.created(location)
                .body(ProductResponse.from(created));
    }
}
```

**Trigger completions manually:** `Alt+\` (or `Option+\` on macOS)

### 3.2 Copilot Chat Panel

Open via `View > Tool Windows > GitHub Copilot Chat` or `Ctrl+Shift+C`.

**Slash commands available in IntelliJ:**

| Command | Purpose | Example |
|---------|---------|---------|
| `/explain` | Explain selected code | Select a complex stream pipeline and run `/explain` |
| `/tests` | Generate unit tests | `/tests` for the selected method |
| `/fix` | Fix a bug or error | Select code with a compilation error and run `/fix` |
| `/doc` | Generate documentation | `/doc` to create Javadoc for selected method |
| `/simplify` | Simplify complex code | Select nested ifs and run `/simplify` |

**Context-aware completions for Spring:**

Copilot reads `application.yml`, entity classes, and repository interfaces to generate contextually accurate completions. When you create a new `@Service` class that injects `JpaRepository<Order, UUID>`, Copilot knows the entity fields and generates CRUD methods accordingly.

### 3.3 Chat Participants (@workspace, @terminal)

```
@workspace How is error handling implemented across the REST controllers?
  -> Scans all controllers, finds @ControllerAdvice, summarises the pattern

@terminal What's the Maven command to run only integration tests?
  -> mvn verify -Dskip.unit.tests=true -Pfailsafe
```

### 3.4 Multi-File Awareness

Copilot considers open editor tabs as context. A practical pattern:

1. Open `Order.java` (entity), `OrderRepository.java`, and `OrderService.java`
2. Start typing in `OrderController.java`
3. Copilot uses all open files to provide accurate endpoint implementations

### 3.5 Keyboard Shortcuts (macOS / Windows-Linux)

| Action | macOS | Windows/Linux |
|--------|-------|---------------|
| Accept suggestion | `Tab` | `Tab` |
| Dismiss suggestion | `Esc` | `Esc` |
| Next suggestion | `Option+]` | `Alt+]` |
| Previous suggestion | `Option+[` | `Alt+[` |
| Trigger inline suggestion | `Option+\` | `Alt+\` |
| Open Copilot Chat | `Ctrl+Shift+C` | `Ctrl+Shift+C` |

---

## 4. Claude Code in IntelliJ

Claude Code is Anthropic's agentic coding tool that operates from the terminal. It excels at multi-file refactoring, complex reasoning, and tasks that require understanding large codebases.

### Setup Options

#### Option A: JetBrains Plugin

1. **Install:** `Settings > Plugins > Marketplace > "Claude Code"`
2. The plugin provides inline diff view and a dedicated tool window
3. Requires Claude Code CLI to be installed separately: `npm install -g @anthropic-ai/claude-code`

#### Option B: Terminal Integration (Always Works)

1. Install Claude Code CLI: `npm install -g @anthropic-ai/claude-code`
2. Open IntelliJ Terminal (`Alt+F12`)
3. Run `claude` to start an interactive session in your project directory
4. IntelliJ's terminal supports Claude Code's rich output (diffs, file trees)

### 4.1 Practical Spring Boot Workflows

#### Multi-file refactoring

```bash
# In IntelliJ terminal:
claude

> Refactor the OrderService to use the Outbox Pattern. Create an OutboxEvent
  entity, an OutboxRepository, replace direct KafkaTemplate.send() calls with
  outbox inserts inside the same @Transactional boundary, and create a scheduled
  OutboxPublisher that polls and publishes to Kafka.
```

Claude Code will:
- Read your existing `OrderService.java`, entity classes, and Kafka configuration
- Create `OutboxEvent.java`, `OutboxRepository.java`, `OutboxPublisher.java`
- Modify `OrderService.java` to write to the outbox table
- Create a Flyway migration `V*.sql` for the outbox table
- Show diffs for every file before applying

#### Architecture review

```bash
> Review the entire com.example.order package for Spring Boot best practices.
  Check for: proper use of @Transactional, error handling, DTO/entity separation,
  missing validation, and potential concurrency issues.
```

#### Database migration generation

```bash
> I added a 'priority' field (enum: LOW, MEDIUM, HIGH, CRITICAL) to the Order
  entity. Generate the Flyway migration, update the repository with a custom
  query to find orders by priority, and update the OrderResponse DTO.
```

### 4.2 Inline Diff (JetBrains Plugin)

When Claude Code proposes changes, the JetBrains plugin shows them as inline diffs directly in the editor — similar to a code review. You can:

- Accept individual hunks
- Reject specific changes
- Edit the proposed code before accepting

### 4.3 Claude Code with CLAUDE.md Project Context

Create a `CLAUDE.md` in your project root to give Claude Code persistent context:

```markdown
# CLAUDE.md
This is a Spring Boot 3.2 microservice using Java 21.
- Build: Maven (mvn clean verify)
- DB: PostgreSQL with Flyway migrations in src/main/resources/db/migration
- Messaging: Kafka with Spring Kafka
- Testing: JUnit 5 + Mockito + Testcontainers
- Code style: Google Java Style, max line length 120
- All REST endpoints must return ResponseEntity<T>
- All services must use constructor injection via @RequiredArgsConstructor
```

### 4.4 When to Use Claude Code vs Other Tools

| Scenario | Best Tool |
|----------|-----------|
| Quick single-line completion | Copilot or AI Assistant |
| Generate a test for one method | AI Assistant or Copilot |
| Refactor across 10+ files | Claude Code |
| Implement a design pattern across a package | Claude Code |
| Explain a single method | AI Assistant |
| Debug a complex multi-service issue | Claude Code |
| Write a Dockerfile or CI pipeline | Claude Code |

---

## 5. Google Gemini Code Assist in IntelliJ

Google's Gemini Code Assist brings Gemini's large context window (up to 1M tokens) to IntelliJ, with particularly strong GCP integration.

### Setup

1. **Install:** `Settings > Plugins > Marketplace > "Gemini Code Assist"`
2. **Sign in:** Google Cloud account or Google account (free tier available)
3. **Configure:** `Settings > Tools > Gemini Code Assist`
4. **Enterprise:** Available through Google Cloud with additional features (code customisation, enterprise context)

### 5.1 Code Completions

Gemini provides inline completions similar to Copilot. Its strength is in understanding large files and long context:

```java
@Configuration
public class KafkaConsumerConfig {

    @Value("${spring.kafka.bootstrap-servers}")
    private String bootstrapServers;

    // Gemini suggests the full consumer factory configuration
    // based on your application.yml and existing producer config
    @Bean
    public ConsumerFactory<String, OrderEvent> consumerFactory() {
        Map<String, Object> props = new HashMap<>();
        props.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, bootstrapServers);
        props.put(ConsumerConfig.GROUP_ID_CONFIG, "order-service");
        props.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG,
                StringDeserializer.class);
        props.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG,
                JsonDeserializer.class);
        props.put(JsonDeserializer.TRUSTED_PACKAGES,
                "com.example.order.event");
        props.put(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG, "earliest");
        props.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, false);
        return new DefaultKafkaConsumerFactory<>(props);
    }

    @Bean
    public ConcurrentKafkaListenerContainerFactory<String, OrderEvent>
            kafkaListenerContainerFactory() {
        ConcurrentKafkaListenerContainerFactory<String, OrderEvent> factory =
                new ConcurrentKafkaListenerContainerFactory<>();
        factory.setConsumerFactory(consumerFactory());
        factory.setConcurrency(3);
        factory.getContainerProperties()
                .setAckMode(ContainerProperties.AckMode.MANUAL_IMMEDIATE);
        return factory;
    }
}
```

### 5.2 Gemini Chat

The chat panel supports natural language queries with full project context:

```
> Generate a Spring Cloud GCP Pub/Sub configuration that replaces
  our current Kafka setup, maintaining the same event schema.

> Explain the memory implications of this Stream.collect() call
  when processing 10M records.
```

### 5.3 Cloud Integration

Gemini Code Assist has first-class support for GCP services:

- Auto-generates `application-gcp.yml` configuration
- Suggests Cloud SQL connection pooling settings
- Generates Cloud Run deployment descriptors
- Understands Secret Manager references in Spring config

### 5.4 Large Context Advantage

With up to 1M token context, Gemini can process your entire module at once. This is particularly useful for:

- Understanding complex inheritance hierarchies
- Tracing request flow across multiple layers
- Generating consistent code that matches patterns spread across many files

---

## 6. Amazon Q Developer in IntelliJ

Amazon Q Developer (formerly CodeWhisperer) provides AI assistance with a strong focus on AWS services, security scanning, and Java modernisation.

### Setup

1. **Install:** `Settings > Plugins > Marketplace > "Amazon Q"`
2. **Authenticate:** AWS Builder ID (free) or AWS IAM Identity Center (Pro)
3. **Tier:** Free (limited suggestions), Pro ($19/user/month through AWS)

### 6.1 Code Completions

Amazon Q provides inline completions with particular strength in AWS SDK usage:

```java
@Service
@RequiredArgsConstructor
public class S3DocumentService {

    private final S3Client s3Client;

    // Amazon Q excels at AWS SDK completions
    public String uploadDocument(MultipartFile file, String bucket) {
        String key = "documents/" + UUID.randomUUID() + "/" + file.getOriginalFilename();

        PutObjectRequest request = PutObjectRequest.builder()
                .bucket(bucket)
                .key(key)
                .contentType(file.getContentType())
                .serverSideEncryption(ServerSideEncryption.AES256)
                .metadata(Map.of(
                        "original-name", file.getOriginalFilename(),
                        "uploaded-at", Instant.now().toString()))
                .build();

        s3Client.putObject(request,
                RequestBody.fromInputStream(file.getInputStream(), file.getSize()));

        return key;
    }
}
```

### 6.2 Security Scanning

One of Amazon Q's standout features. Right-click a file or project > `Amazon Q > Run Security Scan`:

```java
// Amazon Q flags this:
@GetMapping("/users")
public List<User> searchUsers(@RequestParam String name) {
    // SECURITY: SQL Injection vulnerability detected
    String query = "SELECT * FROM users WHERE name = '" + name + "'";
    return jdbcTemplate.query(query, userRowMapper);
}

// Amazon Q suggests:
@GetMapping("/users")
public List<User> searchUsers(@RequestParam String name) {
    return jdbcTemplate.query(
            "SELECT * FROM users WHERE name = ?",
            userRowMapper,
            name);
}
```

Security scanning detects:
- SQL injection, XSS, SSRF
- Hardcoded credentials and secrets
- Insecure cryptographic usage
- OWASP Top 10 violations
- Vulnerable dependency patterns

### 6.3 Java Transformation (Upgrade)

Amazon Q can modernise Java codebases:

- **Java 8 to Java 17/21:** Upgrades language features, APIs, and dependencies
- **Spring Boot 2 to Spring Boot 3:** Handles `javax.*` to `jakarta.*` migration, property changes, removed APIs
- **JUnit 4 to JUnit 5:** Converts annotations, assertions, and runner patterns

```bash
# Trigger via Amazon Q Chat:
> /transform — starts the guided transformation wizard
```

### 6.4 Amazon Q Chat

```
> Generate a DynamoDB-backed session store for this Spring Boot application
  using the AWS SDK v2, with TTL-based expiration.

> Create an SQS listener with dead-letter queue configuration using
  Spring Cloud AWS Messaging.
```

---

## 7. Tabnine in IntelliJ

Tabnine differentiates through privacy, on-premise deployment, and team-level code learning.

### Setup

1. **Install:** `Settings > Plugins > Marketplace > "Tabnine AI Code Completion"`
2. **Account:** Create at tabnine.com (free tier available)
3. **Tier:** Basic (free), Pro ($12/mo), Enterprise (custom — on-prem available)

### 7.1 Code Completions

Tabnine learns from your codebase patterns. After indexing your project, completions match your team's style:

```java
// If your codebase consistently uses this pattern:
public record CreateOrderRequest(
        @NotNull UUID customerId,
        @NotEmpty List<OrderItemRequest> items,
        @Size(max = 500) String notes
) {}

// Tabnine learns and suggests the same pattern for new DTOs:
public record CreatePaymentRequest(
        @NotNull UUID orderId,
        @NotNull @Positive BigDecimal amount,
        @NotNull PaymentMethod method,
        @Size(max = 200) String reference
) {}
```

### 7.2 Team Learning (Enterprise)

In Enterprise mode, Tabnine builds a model of your team's codebase:
- Learns internal API patterns and naming conventions
- Suggests code consistent with your team's existing code
- Understands internal library usage
- Respects `.tabnine.json` configuration for included/excluded paths

### 7.3 Privacy Features

| Feature | Tabnine | Others |
|---------|---------|--------|
| On-premise deployment | Yes (Enterprise) | No (most) |
| Code never leaves your machine | Starter/Pro option | Varies |
| SOC 2 Type II certified | Yes | Varies |
| Train only on permissively licensed code | Yes (by default) | Varies |
| Per-repo opt-out | Yes | Limited |

### 7.4 Tabnine Chat

```
> Explain the transactional boundaries in this saga orchestrator
  and suggest improvements for failure handling.

> Generate a Spring Data JPA specification for dynamic filtering
  of orders by status, date range, and customer.
```

```java
// Tabnine generates specifications matching your existing patterns:
public class OrderSpecifications {

    public static Specification<Order> withFilters(OrderFilter filter) {
        return Specification.where(hasStatus(filter.getStatus()))
                .and(createdBetween(filter.getFrom(), filter.getTo()))
                .and(belongsToCustomer(filter.getCustomerId()));
    }

    private static Specification<Order> hasStatus(OrderStatus status) {
        return (root, query, cb) ->
                status == null ? null : cb.equal(root.get("status"), status);
    }

    private static Specification<Order> createdBetween(Instant from, Instant to) {
        return (root, query, cb) -> {
            if (from == null && to == null) return null;
            if (from != null && to != null)
                return cb.between(root.get("createdAt"), from, to);
            if (from != null)
                return cb.greaterThanOrEqualTo(root.get("createdAt"), from);
            return cb.lessThanOrEqualTo(root.get("createdAt"), to);
        };
    }

    private static Specification<Order> belongsToCustomer(UUID customerId) {
        return (root, query, cb) ->
                customerId == null ? null :
                        cb.equal(root.get("customerId"), customerId);
    }
}
```

---

## 8. Comparison Matrix

### Feature-by-Feature Comparison

| Feature | JetBrains AI | Copilot | Claude Code | Gemini | Amazon Q | Tabnine |
|---------|:------------:|:-------:|:-----------:|:------:|:--------:|:-------:|
| **Inline completions** | Yes | Yes | No | Yes | Yes | Yes |
| **Multi-line completions** | Yes | Yes | No | Yes | Yes | Yes |
| **Chat panel** | Yes | Yes | Terminal | Yes | Yes | Yes |
| **Explain code** | Yes | Yes | Yes | Yes | Yes | Yes |
| **Generate tests** | Yes | Yes | Yes | Yes | Yes | Yes |
| **Refactoring suggestions** | Yes | Limited | Yes | Limited | Limited | Limited |
| **Multi-file edits** | No | No | Yes | No | No | No |
| **Commit message gen** | Yes | Yes | Yes | Yes | No | No |
| **Security scanning** | Basic | No | Yes | No | Yes | No |
| **Code transformation** | No | No | Yes | No | Yes (Java) | No |
| **Cloud integration** | No | No | No | GCP | AWS | No |
| **On-premise option** | No | Enterprise | No | Enterprise | No | Yes |
| **Custom model training** | No | No | No | Enterprise | No | Enterprise |
| **IDE integration depth** | Deep | Good | Terminal | Good | Good | Good |

### Pricing Comparison (as of 2025)

| Tool | Free Tier | Individual | Business/Pro | Enterprise |
|------|-----------|------------|--------------|------------|
| **JetBrains AI** | Limited | $10/mo | $10/mo | Custom |
| **GitHub Copilot** | OSS/Student | $10/mo | $19/mo | $39/mo |
| **Claude Code** | None | Pay-per-use | Pay-per-use | Custom |
| **Gemini Code Assist** | Yes (limited) | Free | $19/mo | Custom |
| **Amazon Q** | Yes (limited) | N/A | $19/mo | Custom |
| **Tabnine** | Basic | $12/mo | Custom | Custom |

### Java/Spring Boot Specific Strengths

| Capability | Best Tool(s) |
|-----------|-------------|
| Spring Boot autoconfiguration awareness | JetBrains AI, Copilot |
| JPA/Hibernate query generation | Copilot, JetBrains AI |
| Kafka consumer/producer boilerplate | Copilot, Amazon Q |
| Complex multi-file refactoring | Claude Code |
| AWS SDK integration code | Amazon Q |
| GCP service integration | Gemini |
| Test generation (JUnit 5 + Mockito) | JetBrains AI, Copilot |
| Spring Boot 2 to 3 migration | Amazon Q, Claude Code |
| Security vulnerability detection | Amazon Q |
| Flyway/Liquibase migration generation | Claude Code |
| Architecture-level decisions | Claude Code |
| Quick boilerplate (DTOs, configs) | Copilot, JetBrains AI |

---

## 9. Recommended Setup for Java/Spring Boot Development

### The Optimal Combination

For a senior Java/Spring Boot engineer, the recommended stack is:

```
Primary:   GitHub Copilot        — daily inline completions and quick chat
Secondary: JetBrains AI Assistant — deep IDE refactoring, inspections, commit messages
Power:     Claude Code            — complex multi-file tasks, architecture work
Periodic:  Amazon Q               — security scanning, Java version upgrades
```

### Configuration to Avoid Conflicts

Running multiple AI plugins simultaneously causes conflicts in inline completions. Here is how to manage them.

#### Step 1: Set Completion Priority

`Settings > Tools > AI Assistant > Code Completion > Enable AI code completion suggestions`

Only enable inline completions for ONE tool at a time. Recommendation:

```
GitHub Copilot:      Inline completions ON
JetBrains AI:        Inline completions OFF (use for chat/refactoring only)
Gemini Code Assist:  Inline completions OFF (or uninstall if not using GCP)
Amazon Q:            Inline completions OFF (enable only for security scans)
Tabnine:             Uninstall if using Copilot
```

#### Step 2: Configure Each Tool's Role

```yaml
# In your head, not a real file - this is your usage strategy:

copilot:
  role: "Primary code completion and quick chat"
  when: "Writing new code, filling in boilerplate, quick Q&A"

jetbrains-ai:
  role: "Deep IDE integration"
  when: "Refactoring, inspections, commit messages, test generation via context menu"

claude-code:
  role: "Complex agent tasks"
  when: "Multi-file refactoring, architecture changes, large implementations"

amazon-q:
  role: "Security and modernisation"
  when: "Weekly security scans, Java/Spring Boot version upgrades"
```

#### Step 3: Memory and Performance

Each AI plugin consumes memory. Monitor with `Help > Diagnostic Tools > Activity Monitor`:

| Plugin | Typical Memory Overhead | Impact |
|--------|------------------------|--------|
| GitHub Copilot | ~200MB | Moderate |
| JetBrains AI | ~150MB (bundled) | Low |
| Gemini Code Assist | ~180MB | Moderate |
| Amazon Q | ~250MB | Moderate-High |
| Tabnine | ~300MB (local model) | High |

**Recommendation:** Keep a maximum of 2-3 AI plugins active simultaneously. Disable others via `Settings > Plugins` when not needed.

#### Step 4: `.gitignore` Additions

```gitignore
# AI tool configurations (add to .gitignore)
.tabnineignore
.tabnine_root
.amazonq/
copilot-debug.log
```

### Per-Project Configuration

Create project-specific instructions for tools that support it:

```
project-root/
├── CLAUDE.md              # Claude Code project instructions
├── .github/
│   └── copilot-instructions.md  # Copilot custom instructions
├── .amazonq/
│   └── rules/             # Amazon Q project rules
└── .tabnine.json          # Tabnine project config
```

---

## 10. Keyboard Shortcuts Reference

### All AI-Related Shortcuts in IntelliJ

#### JetBrains AI Assistant

| Action | macOS | Windows/Linux |
|--------|-------|---------------|
| Open AI Chat | `Cmd+Shift+A > AI Chat` | `Ctrl+Shift+A > AI Chat` |
| AI Actions on selection | `Alt+Enter > AI Actions` | `Alt+Enter > AI Actions` |
| Generate with AI | `Cmd+N > AI Generate` | `Alt+Insert > AI Generate` |
| Explain code | Select + `Alt+Enter > Explain` | Select + `Alt+Enter > Explain` |
| Generate commit message | In Commit dialog, click AI icon | In Commit dialog, click AI icon |
| Suggest refactoring | Select + `Alt+Enter > Suggest Refactoring` | Select + `Alt+Enter > Suggest Refactoring` |

#### GitHub Copilot

| Action | macOS | Windows/Linux |
|--------|-------|---------------|
| Accept suggestion | `Tab` | `Tab` |
| Dismiss suggestion | `Esc` | `Esc` |
| Next suggestion | `Option+]` | `Alt+]` |
| Previous suggestion | `Option+[` | `Alt+[` |
| Trigger suggestion | `Option+\` | `Alt+\` |
| Open Copilot Chat | `Ctrl+Shift+C` | `Ctrl+Shift+C` |
| Accept word | `Cmd+Right` | `Ctrl+Right` |

#### Claude Code (Terminal)

| Action | macOS | Windows/Linux |
|--------|-------|---------------|
| Open Terminal | `Option+F12` | `Alt+F12` |
| Start Claude Code | Type `claude` | Type `claude` |
| Send message | `Enter` | `Enter` |
| Multi-line input | `Shift+Enter` | `Shift+Enter` |
| Cancel generation | `Esc` | `Esc` |
| Exit Claude Code | `/exit` or `Ctrl+C` | `/exit` or `Ctrl+C` |
| Clear context | `/clear` | `/clear` |

#### Amazon Q

| Action | macOS | Windows/Linux |
|--------|-------|---------------|
| Accept suggestion | `Tab` | `Tab` |
| Dismiss suggestion | `Esc` | `Esc` |
| Trigger suggestion | `Option+C` | `Alt+C` |
| Open Q Chat | `Cmd+Shift+Q` | `Ctrl+Shift+Q` |
| Run security scan | Right-click > `Amazon Q > Scan` | Right-click > `Amazon Q > Scan` |

#### General IntelliJ (AI-Adjacent)

| Action | macOS | Windows/Linux |
|--------|-------|---------------|
| Quick documentation | `F1` | `Ctrl+Q` |
| Parameter info | `Cmd+P` | `Ctrl+P` |
| Show intention actions | `Option+Enter` | `Alt+Enter` |
| Find action | `Cmd+Shift+A` | `Ctrl+Shift+A` |
| Generate code | `Cmd+N` | `Alt+Insert` |

---

## 11. Try This Exercises

### Exercise 1: AI-Assisted REST Controller Development

**Goal:** Build a complete REST controller using AI completions.

1. Create a new class `InventoryController.java`
2. Type `@RestController` and let Copilot/AI Assistant suggest the class structure
3. Add the field `private final InventoryService inventoryService;`
4. Type method signatures one by one and observe how the AI completes each:
   - `GET /api/v1/inventory/{sku}` — single item lookup
   - `POST /api/v1/inventory/reserve` — stock reservation
   - `PUT /api/v1/inventory/{sku}/restock` — restock with quantity
5. Compare completions from Copilot vs JetBrains AI (toggle which is active)

**Evaluation:** Did the AI correctly use `ResponseEntity`, proper HTTP status codes, `@Valid`, and path variables?

### Exercise 2: Test Generation Showdown

**Goal:** Compare test generation quality across tools.

1. Take an existing service class with 3-4 methods (e.g., `OrderService`)
2. Generate tests using:
   - JetBrains AI: Right-click > `AI Actions > Generate Tests`
   - Copilot Chat: `/tests` with the class selected
   - Claude Code: `claude "Generate comprehensive tests for OrderService.java including edge cases"`
3. Compare the output on these criteria:

| Criterion | JetBrains AI | Copilot | Claude Code |
|-----------|:------------:|:-------:|:-----------:|
| Correct Mockito usage | ? | ? | ? |
| Edge cases covered | ? | ? | ? |
| Uses AssertJ fluent assertions | ? | ? | ? |
| Tests exception paths | ? | ? | ? |
| Follows existing test patterns | ? | ? | ? |

### Exercise 3: Complex Refactoring with Claude Code

**Goal:** Use Claude Code for a multi-file refactoring.

1. Open IntelliJ Terminal and start `claude`
2. Ask Claude Code to implement the **Repository Pattern** for an entity that currently uses direct JDBC:

```
> Refactor the UserDao class to use Spring Data JPA.
  Create the User entity with proper JPA annotations,
  a UserRepository interface extending JpaRepository,
  update UserService to use the repository instead of JdbcTemplate,
  and create a Flyway migration if the table structure changes.
```

3. Review the inline diffs before accepting
4. Run existing tests to verify nothing broke

### Exercise 4: Security Scan with Amazon Q

**Goal:** Find and fix security vulnerabilities.

1. Enable Amazon Q plugin
2. Right-click on `src/main/java` > `Amazon Q > Run Security Scan`
3. Review all findings — categorise them:
   - Critical (fix immediately)
   - Warning (fix in next sprint)
   - Info (nice to have)
4. Use Copilot Chat to explain each vulnerability: `@workspace /explain this security finding: [paste finding]`
5. Fix the top 3 critical issues

### Exercise 5: Full Feature Implementation with AI Collaboration

**Goal:** Build a complete feature using multiple AI tools together.

**Feature:** Add a "scheduled price drop" capability to a product service.

| Step | Tool | Task |
|------|------|------|
| 1 | Copilot | Scaffold `ScheduledPriceDropRequest` DTO with validation |
| 2 | JetBrains AI | Generate the `@Entity` for `PriceDrop` with JPA annotations |
| 3 | Claude Code | Implement `PriceDropScheduler` using `@Scheduled` with proper locking (`ShedLock`) |
| 4 | Copilot | Generate the REST endpoints in `PriceDropController` |
| 5 | JetBrains AI | Generate unit tests for `PriceDropService` |
| 6 | Claude Code | Add integration test with Testcontainers (Postgres + Redis) |
| 7 | Amazon Q | Run security scan on the new code |

---

## 12. Tips and Gotchas

### Common Issues

#### 1. Conflicting Inline Completions

**Problem:** Two plugins both show ghost text, causing flickering and incorrect acceptances.

**Fix:** Only enable inline completions for ONE plugin. Go to:
- Copilot: `Settings > Languages & Frameworks > GitHub Copilot > Disable completions`
- AI Assistant: `Settings > Tools > AI Assistant > Disable code completion suggestions`
- Tabnine: `Settings > Tools > Tabnine > Disable inline suggestions`

#### 2. Slow IDE After Installing Multiple AI Plugins

**Problem:** IntelliJ becomes sluggish with 3+ AI plugins active.

**Fix:**
- Increase IDE heap: `Help > Change Memory Settings` (set to 4096MB+)
- Disable background indexing for tools you are not actively using
- Close chat panels you are not using (each maintains a connection)
- Add to `idea.vmoptions`:
  ```
  -Xmx4096m
  -XX:+UseG1GC
  -XX:SoftRefLRUPolicyMSPerMB=50
  ```

#### 3. AI Suggests Deprecated Spring APIs

**Problem:** AI tools sometimes suggest Spring Boot 2.x patterns in a Spring Boot 3.x project.

**Fix:**
- Ensure your `pom.xml`/`build.gradle` is saved (AI tools read it for version context)
- In chat, prefix your query: "Using Spring Boot 3.2 and Java 21, ..."
- For Copilot, create `.github/copilot-instructions.md`:
  ```markdown
  This project uses Spring Boot 3.2 with Java 21.
  Use jakarta.* packages (not javax.*).
  Use constructor injection with @RequiredArgsConstructor.
  Use records for DTOs.
  ```

#### 4. AI-Generated Code Doesn't Match Project Style

**Problem:** Generated code uses different formatting, naming conventions, or patterns than your project.

**Fix:**
- JetBrains AI: It respects your `.editorconfig` and IntelliJ code style settings. Ensure they are configured.
- Copilot: Open related files in editor tabs (Copilot uses open tabs as context).
- Claude Code: Maintain a `CLAUDE.md` with explicit style rules.
- All tools: Run `Cmd+Option+L` (Reformat Code) after accepting AI suggestions.

#### 5. Copilot Suggestions Lag on Large Files

**Problem:** In files with 500+ lines, Copilot suggestions take 2-3 seconds.

**Fix:**
- Break large classes into smaller ones (this is good practice anyway)
- Close unnecessary editor tabs (reduces context sent to Copilot)
- Check network: `Settings > Languages & Frameworks > GitHub Copilot > Diagnostics`

### Performance Tips

| Tip | Impact |
|-----|--------|
| Keep only 5-8 editor tabs open | Reduces context payload, faster completions |
| Use `.copilotignore` to exclude test fixtures, generated code | Fewer irrelevant suggestions |
| Run Claude Code in a separate terminal window for heavy tasks | Prevents IntelliJ terminal from locking up |
| Disable AI completion in `application.yml` / XML files | These files confuse most AI tools |
| Restart AI plugins weekly | Clears accumulated memory leaks |

### When to Use Which Tool — Decision Flowchart

```
Need to write code?
├── Single line / autocomplete → Copilot (inline)
├── Full method body → Copilot (inline) or JetBrains AI
├── Multiple related files → Claude Code
└── AWS/GCP boilerplate → Amazon Q / Gemini

Need to understand code?
├── Single method → JetBrains AI (Alt+Enter > Explain)
├── Whole class → Copilot Chat (/explain)
├── Cross-module flow → Claude Code
└── Security implications → Amazon Q (security scan)

Need to refactor?
├── Rename/extract (IDE handles well) → IntelliJ built-in
├── Pattern application → JetBrains AI (suggest refactoring)
├── Cross-cutting concern → Claude Code
└── Java/Spring version upgrade → Amazon Q (/transform)

Need to test?
├── Unit test for one method → JetBrains AI (generate tests)
├── Test class with edge cases → Copilot Chat (/tests)
├── Integration test with Testcontainers → Claude Code
└── Security test → Amazon Q
```

### Privacy Considerations

| Concern | Recommendation |
|---------|---------------|
| Sending proprietary code to AI | Use Tabnine (on-prem) or enterprise tiers with data retention controls |
| API keys in code | All tools may inadvertently learn from `.env` files — add to `.gitignore` AND tool-specific ignore files |
| Compliance requirements (SOC2, HIPAA) | Tabnine Enterprise (on-prem), GitHub Copilot Business (with content exclusion), Amazon Q (AWS data policies) |
| Open source license contamination | Tabnine trains only on permissive licenses by default; Copilot has a "block matching public code" filter |

### Final Advice

1. **Start with Copilot + JetBrains AI.** This combination covers 90% of daily coding needs with minimal conflict.
2. **Add Claude Code for heavy tasks.** When you need to refactor a package, implement a design pattern across files, or reason about architecture, switch to Claude Code.
3. **Run Amazon Q security scans weekly.** Even if you do not use it for completions, the security scanning is valuable.
4. **Measure your productivity.** Track time spent on tasks before and after adopting AI tools. Most engineers see 20-40% improvement on boilerplate-heavy tasks.
5. **Stay updated.** These tools release updates monthly. Check changelogs for new features — the landscape is evolving rapidly.

---

*Last updated: 2025*
