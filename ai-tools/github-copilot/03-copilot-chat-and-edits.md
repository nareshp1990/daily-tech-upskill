# Tutorial 3: Copilot Chat & Edits

## Table of Contents

- [Copilot Chat Overview](#copilot-chat-overview)
- [Chat Participants](#chat-participants)
- [Slash Commands](#slash-commands)
- [Context References](#context-references)
- [Copilot Edits Mode](#copilot-edits-mode)
- [Inline Chat](#inline-chat)
- [Vision Capabilities](#vision-capabilities)
- [Practical Examples: Explain and Understand](#practical-examples-explain-and-understand)
- [Practical Examples: Generate and Fix](#practical-examples-generate-and-fix)
- [Practical Examples: Stack-Specific](#practical-examples-stack-specific)
- [Try This Exercises](#try-this-exercises)

---

## Copilot Chat Overview

Copilot Chat is a conversational AI assistant built into your IDE. Unlike code completions (which are automatic), Chat is interactive — you ask questions and get detailed responses.

### Opening Chat

| IDE | Shortcut | Notes |
|-----|----------|-------|
| VS Code — Chat Panel | `Ctrl + Alt + I` | Opens sidebar panel |
| VS Code — Inline Chat | `Ctrl + I` | Opens chat at cursor position |
| VS Code — Quick Chat | `Ctrl + Shift + I` | Floating chat box at top |
| IntelliJ — Chat Panel | `Alt + C` (or via Tool Windows) | Opens side panel |

### Chat Panel vs Inline Chat

- **Chat Panel**: Long-form conversations, explanations, multi-file questions, architecture discussions
- **Inline Chat**: Quick, focused edits at the cursor location — refactor a method, fix a bug, add a parameter

---

## Chat Participants

Chat participants scope your question to a specific domain. Use them with the `@` prefix.

### @workspace

Searches your entire workspace/project for context before answering.

```
@workspace How is authentication implemented in this project?

@workspace What services depend on the OrderRepository?

@workspace Show me all Kafka consumers and what topics they listen to

@workspace Where is the database connection configured?
```

**When to use**: Any question about your project's structure, architecture, or how things connect.

### @terminal

Has context about your terminal — recent commands, output, errors.

```
@terminal Explain the error in my terminal

@terminal Why did my Maven build fail?

@terminal What does this stack trace mean?
```

**When to use**: When you have a terminal error and want an explanation without copy-pasting.

### @vscode

Knows about VS Code settings, extensions, and configuration.

```
@vscode How do I configure the Java formatter?

@vscode What keyboard shortcut opens the terminal?

@vscode How do I set up a launch configuration for Spring Boot?
```

**When to use**: VS Code configuration questions.

### @github (Enterprise/Business)

Searches across your GitHub organization's repositories and knowledge bases.

```
@github How do other teams implement pagination?

@github Find examples of Kafka consumer configuration in our org

@github What's our standard error handling pattern?
```

**When to use**: Finding patterns and precedents across your organization's codebase.

---

## Slash Commands

Slash commands tell Copilot what type of task you want to perform.

### /explain

Explains selected code or a concept.

```
/explain

/explain this Kafka consumer configuration

/explain what this regex does: ^(?=.*[A-Z])(?=.*\d)[A-Za-z\d@$!%*?&]{8,}$
```

### /fix

Identifies and fixes bugs in the selected code.

```
/fix this method is throwing NullPointerException

/fix the SQL query returns duplicate results

/fix this test is flaky
```

### /tests

Generates unit tests for the selected code.

```
/tests

/tests using JUnit 5 and Mockito

/tests generate integration tests with Testcontainers
```

### /doc

Generates documentation for the selected code.

```
/doc

/doc generate Javadoc for this class

/doc add OpenAPI annotations to this controller
```

### /optimize

Suggests performance improvements.

```
/optimize this database query is slow

/optimize reduce memory usage in this method
```

### /simplify

Simplifies overly complex code.

```
/simplify

/simplify reduce the nesting in this method
```

### /new

Scaffolds new code.

```
/new Spring Boot REST controller for managing products

/new Kafka consumer for processing payment events

/new Flyway migration to add an index on orders.customer_id
```

### Combining Slash Commands with Participants

```
@workspace /explain how the order processing pipeline works

@workspace /tests generate tests for the PaymentService

@terminal /fix why is my Docker container crashing?
```

---

## Context References

Context references let you point Copilot to specific pieces of context using `#`.

### #file

Reference a specific file:

```
#file:OrderService.java explain the processOrder method

Write tests for the methods in #file:PaymentService.java

Refactor #file:UserController.java to use records for DTOs
```

### #selection

Reference the currently selected text in the editor:

```
/explain #selection

/fix #selection

Write tests for #selection
```

**Tip**: Select a block of code in the editor before typing in chat.

### #editor

Reference the entire visible content of the active editor:

```
/explain #editor

Find potential bugs in #editor
```

### #terminalLastCommand

Reference the last command run in the terminal:

```
Explain #terminalLastCommand

Why did #terminalLastCommand fail?
```

### #terminalSelection

Reference selected text in the terminal:

```
/explain #terminalSelection
```

### Combining Multiple References

```
Using the entity in #file:Order.java, write a repository interface
that matches the patterns in #file:UserRepository.java

Compare #file:OrderServiceImpl.java with #file:PaymentServiceImpl.java
and identify inconsistencies in error handling
```

---

## Copilot Edits Mode

Copilot Edits is a multi-file editing mode in VS Code that lets you describe changes in natural language and Copilot applies them across multiple files simultaneously.

### Opening Edits Mode

- Click the Copilot Edits icon in the sidebar (pencil icon)
- Or `Ctrl + Shift + I` and switch to Edits mode

### How It Works

1. Add files to the "Working Set" (files Copilot can read and modify)
2. Describe the change you want
3. Copilot shows a diff preview across all affected files
4. Review and accept/reject changes per file

### Example: Add Audit Fields to an Entity

**Working Set**: `Order.java`, `OrderRepository.java`, `OrderService.java`, `V004__add_audit_fields.sql`

**Prompt**:
```
Add audit fields (createdBy, updatedBy as String) to the Order entity.
Update the repository to support querying by createdBy.
Add a Flyway migration to add the columns to the orders table.
Update the service to set audit fields from the security context.
```

Copilot modifies all 4 files in a single operation.

### Example: Implement a New Endpoint

**Working Set**: `ProductController.java`, `ProductService.java`, `ProductServiceImpl.java`, `ProductRepository.java`

**Prompt**:
```
Add a search endpoint GET /api/v1/products/search that accepts query parameters:
name (partial match), minPrice, maxPrice, category.
Use Spring Data JPA Specification for dynamic filtering.
Return Page<ProductResponse>.
```

### Tips for Edits Mode

- Keep the working set small (3-8 files) for best results
- Be specific about what you want changed in each file
- Review diffs carefully before accepting
- Use Undo (`Ctrl+Z`) if edits are wrong — they are applied as regular edits

---

## Inline Chat

Inline Chat opens a small chat interface directly at your cursor position. It is ideal for quick, focused edits.

### Opening Inline Chat

`Ctrl + I` (VS Code)

### Common Uses

#### Add Error Handling

Select a method, press `Ctrl+I`:
```
Add try-catch with proper logging and throw a custom exception
```

#### Refactor a Method

Select the method, press `Ctrl+I`:
```
Extract the validation logic into a separate private method
```

#### Add Annotations

Place cursor on a method, press `Ctrl+I`:
```
Add @Cacheable with a 5-minute TTL and cache name "products"
```

#### Convert Code

Select code, press `Ctrl+I`:
```
Convert this for-loop to a Stream pipeline
```

#### Add Logging

Select a method, press `Ctrl+I`:
```
Add structured logging with SLF4J at entry, exit, and error points
```

### Inline Chat vs Chat Panel

| Use Case | Inline Chat (`Ctrl+I`) | Chat Panel (`Ctrl+Alt+I`) |
|----------|----------------------|--------------------------|
| Fix a single method | Yes | Overkill |
| Explain architecture | No | Yes |
| Add a parameter | Yes | No |
| Generate tests for a class | No | Yes |
| Refactor 2 lines | Yes | No |
| Multi-file changes | No | Use Edits mode |

---

## Vision Capabilities

Copilot Chat can understand images. You can paste screenshots directly into the chat.

### Use Cases

1. **UI Bug Reports**: Paste a screenshot of a broken UI, ask Copilot to identify the issue
2. **Architecture Diagrams**: Paste a diagram, ask Copilot to generate corresponding code
3. **Error Screenshots**: Paste a screenshot of a stack trace or error dialog
4. **Design Mockups**: Paste a wireframe, ask for HTML/CSS or component code

### How to Use

1. Take a screenshot or copy an image
2. Open Copilot Chat (`Ctrl+Alt+I`)
3. Paste the image (`Ctrl+V`) directly into the chat input
4. Add your question:

```
This is a screenshot of our monitoring dashboard showing high latency.
What could be causing the spike at 2pm? Suggest investigation steps.
```

```
This is our system architecture diagram. Generate a Docker Compose file
that includes all the services shown.
```

---

## Practical Examples: Explain and Understand

### Example 1: Explain a Complex Kafka Configuration

Select this code and use `/explain`:

```java
@Configuration
public class KafkaConsumerConfig {

    @Bean
    public ConcurrentKafkaListenerContainerFactory<String, OrderEvent> kafkaListenerContainerFactory(
            ConsumerFactory<String, OrderEvent> consumerFactory) {
        ConcurrentKafkaListenerContainerFactory<String, OrderEvent> factory =
                new ConcurrentKafkaListenerContainerFactory<>();
        factory.setConsumerFactory(consumerFactory);
        factory.setConcurrency(3);
        factory.getContainerProperties().setAckMode(ContainerProperties.AckMode.MANUAL_IMMEDIATE);
        factory.setCommonErrorHandler(new DefaultErrorHandler(
                new DeadLetterPublishingRecoverer(kafkaTemplate),
                new FixedBackOff(1000L, 3)));
        factory.getContainerProperties().setIdleEventInterval(60000L);
        return factory;
    }
}
```

Chat response explains: concurrency level, manual acknowledgment, dead letter queue configuration, retry policy with fixed backoff, and idle event monitoring.

### Example 2: Explain a Terraform Module

```
@workspace /explain How does our AKS Terraform module handle node pool scaling?
What variables control the min and max node counts?
```

### Example 3: Understand a Legacy Query

```
/explain this SQL query and suggest how to optimize it:

SELECT o.*, c.name, c.email,
       (SELECT COUNT(*) FROM order_items oi WHERE oi.order_id = o.id) as item_count,
       (SELECT SUM(oi.unit_price * oi.quantity) FROM order_items oi WHERE oi.order_id = o.id) as total
FROM orders o
JOIN customers c ON o.customer_id = c.id
WHERE o.created_at > DATE_SUB(NOW(), INTERVAL 30 DAY)
ORDER BY o.created_at DESC
```

---

## Practical Examples: Generate and Fix

### Example 4: Generate JUnit Tests

Select `OrderServiceImpl.java` and ask:

```
/tests Generate comprehensive JUnit 5 tests for the processOrder method.
Use Mockito for mocking OrderRepository, KafkaTemplate, and InventoryClient.
Include tests for: happy path, insufficient inventory, payment failure,
Kafka publish failure. Use AssertJ assertions.
```

Copilot generates:

```java
@ExtendWith(MockitoExtension.class)
class OrderServiceImplTest {

    @Mock private OrderRepository orderRepository;
    @Mock private KafkaTemplate<String, OrderEvent> kafkaTemplate;
    @Mock private InventoryClient inventoryClient;
    @InjectMocks private OrderServiceImpl orderService;

    @Test
    void processOrder_shouldCreateOrderAndPublishEvent() {
        // Arrange
        var request = new CreateOrderRequest(/* ... */);
        when(inventoryClient.checkAvailability(any())).thenReturn(true);
        when(orderRepository.save(any())).thenReturn(testOrder());

        // Act
        var result = orderService.processOrder(request);

        // Assert
        assertThat(result).isNotNull();
        assertThat(result.status()).isEqualTo(OrderStatus.CREATED);
        verify(kafkaTemplate).send(eq("order-created"), any(), any());
    }

    @Test
    void processOrder_shouldThrowWhenInventoryUnavailable() {
        // ...
    }
}
```

### Example 5: Fix a Bug

```
/fix This method sometimes returns stale data. We use @Cacheable but
the cache is not invalidated when orders are updated:

@Cacheable("orders")
public OrderResponse getOrder(Long id) {
    return orderRepository.findById(id)
        .map(orderMapper::toResponse)
        .orElseThrow(() -> new ResourceNotFoundException("Order", id));
}
```

Copilot suggests adding `@CacheEvict` to update/delete methods and using `@CachePut` on the update method.

### Example 6: Refactor to Clean Architecture

```
Refactor #file:OrderService.java to follow hexagonal architecture.
Extract the business logic into a domain service.
Create a port interface for the repository.
Keep the Spring annotations in the adapter layer only.
```

---

## Practical Examples: Stack-Specific

### Example 7: Explain a Kafka Consumer

```
/explain this Kafka consumer — what happens if processing fails?
How does the retry mechanism work? What goes to the DLQ?

@KafkaListener(
    topics = "payment-events",
    groupId = "order-service",
    containerFactory = "retryableKafkaListenerContainerFactory"
)
public void handlePaymentEvent(
        @Payload PaymentEvent event,
        @Header(KafkaHeaders.RECEIVED_KEY) String key,
        Acknowledgment ack) {
    try {
        log.info("Processing payment event: {}", key);
        orderService.updatePaymentStatus(event);
        ack.acknowledge();
    } catch (RetryableException e) {
        throw e; // Will be retried by container
    } catch (Exception e) {
        log.error("Non-retryable error processing payment event: {}", key, e);
        ack.acknowledge(); // Ack to avoid infinite retry
        deadLetterService.publish("payment-events-dlq", key, event, e);
    }
}
```

### Example 8: Generate Integration Test with Testcontainers

```
/tests Generate an integration test for OrderController using:
- @SpringBootTest with random port
- Testcontainers for MySQL and Kafka
- WebTestClient for HTTP calls
- Verify the full flow: create order → check DB → verify Kafka message
```

### Example 9: Docker Compose Help

```
I have a Spring Boot app that needs MySQL, Kafka (with Zookeeper),
and Redis. Generate a docker-compose.yml with:
- Health checks for all services
- Named volumes for data persistence
- A custom network
- Environment variables for Spring Boot to connect to all services
```

### Example 10: Terraform Debugging

```
@terminal My Terraform apply failed with this error. Explain why and
suggest a fix. The AKS cluster is failing to create because of subnet
configuration.
```

---

## Try This Exercises

### Exercise 1: Chat Exploration

1. Open your most complex Java class
2. Ask `@workspace /explain what is the responsibility of this class and how does it fit into the system?`
3. Follow up with: "What could go wrong in this code?"
4. Follow up with: "Suggest 3 improvements"

### Exercise 2: Test Generation

1. Select a service method in your project
2. Use `/tests` to generate unit tests
3. Review the generated tests — are mocks correct? Are edge cases covered?
4. Ask follow-up: "Add tests for concurrent access and race conditions"

### Exercise 3: Inline Chat Refactoring

1. Find a method longer than 30 lines
2. Select it and press `Ctrl+I`
3. Type: "Refactor this into smaller methods with clear names"
4. Review the suggestion before accepting

### Exercise 4: Multi-File Edits

1. Open Copilot Edits mode
2. Add 3-4 related files (Controller, Service, Repository, Entity)
3. Ask: "Add soft delete support. Add a 'deleted' boolean field, filter it in queries, and add a DELETE endpoint that sets deleted=true"
4. Review all diffs before accepting

### Exercise 5: Vision

1. Take a screenshot of a terminal error or monitoring dashboard
2. Paste it into Copilot Chat
3. Ask: "What does this error mean and how do I fix it?"

---

## Next Steps

You now know how to have productive conversations with Copilot. Next, learn how to use [Tutorial 4: Copilot CLI](04-copilot-cli.md) for command-line productivity.
