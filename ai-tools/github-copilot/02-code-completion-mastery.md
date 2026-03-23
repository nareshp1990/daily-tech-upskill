# Tutorial 2: Code Completion Mastery

## Table of Contents

- [How Inline Suggestions Work](#how-inline-suggestions-work)
- [Core Keyboard Shortcuts](#core-keyboard-shortcuts)
- [Partial Acceptance](#partial-acceptance)
- [Writing Comments to Guide Suggestions](#writing-comments-to-guide-suggestions)
- [Context Awareness](#context-awareness)
- [Multi-Line and Function-Level Completions](#multi-line-and-function-level-completions)
- [Copilot Completion Panel](#copilot-completion-panel)
- [Practical Examples: Spring Boot](#practical-examples-spring-boot)
- [Practical Examples: DevOps Files](#practical-examples-devops-files)
- [Tips for Better Suggestions](#tips-for-better-suggestions)
- [Try This Exercises](#try-this-exercises)

---

## How Inline Suggestions Work

Copilot uses a large language model (LLM) to predict what you are likely to type next. It sends the following context to the model:

1. **Current file content** — everything above and below your cursor
2. **Open tabs** — other files currently open in your IDE (neighboring tabs are weighted higher)
3. **File name and path** — `UserController.java` tells Copilot a lot about what to generate
4. **Language and framework** — detected from file extension and imports
5. **Project-level instructions** — from `.github/copilot-instructions.md`

The model generates one or more completions and displays the top suggestion as **ghost text** (gray, semi-transparent text after your cursor).

### The Suggestion Lifecycle

```
You type code → Copilot sends context to model → Model returns suggestions
→ Ghost text appears → You accept/reject/modify → Repeat
```

Suggestions typically appear after a brief delay (100-500ms) when you pause typing.

---

## Core Keyboard Shortcuts

### VS Code

| Action | Shortcut | Notes |
|--------|----------|-------|
| **Accept full suggestion** | `Tab` | Accepts all ghost text |
| **Dismiss suggestion** | `Esc` | Clears ghost text |
| **Next suggestion** | `Alt + ]` | Cycle to next alternative |
| **Previous suggestion** | `Alt + [` | Cycle to previous alternative |
| **Accept next word** | `Ctrl + Right Arrow` | Partial accept, word by word |
| **Accept next line** | `Ctrl + Down Arrow` (custom) | Accept one line at a time |
| **Open completions panel** | `Ctrl + Enter` | See up to 10 suggestions in a panel |
| **Trigger suggestion manually** | `Alt + \` | Force Copilot to suggest when it hasn't |

### IntelliJ IDEA

| Action | Shortcut | Notes |
|--------|----------|-------|
| **Accept full suggestion** | `Tab` | Accepts all ghost text |
| **Dismiss suggestion** | `Esc` | Clears ghost text |
| **Next suggestion** | `Alt + ]` | Cycle to next alternative |
| **Previous suggestion** | `Alt + [` | Cycle to previous alternative |
| **Accept next word** | `Ctrl + Right Arrow` | Partial accept |
| **Open completions panel** | Not available by default | Use Chat instead |

### Neovim (copilot.lua)

| Action | Shortcut | Notes |
|--------|----------|-------|
| **Accept suggestion** | `Alt + l` | Configurable |
| **Accept word** | `Alt + k` | Configurable |
| **Accept line** | `Alt + j` | Configurable |
| **Next suggestion** | `Alt + ]` | Configurable |
| **Previous suggestion** | `Alt + [` | Configurable |
| **Dismiss** | `Ctrl + ]` | Configurable |

---

## Partial Acceptance

Partial acceptance is one of the most powerful features. Instead of accepting an entire multi-line suggestion, you can accept it piece by piece.

### Word-by-Word Acceptance

```java
// You type:
public ResponseEntity<UserResponse> getUserBy

// Copilot suggests (ghost text):
// Id(@PathVariable Long id) {
//     return userService.findById(id)
//         .map(ResponseEntity::ok)
//         .orElse(ResponseEntity.notFound().build());
// }

// Press Ctrl+Right Arrow once:
public ResponseEntity<UserResponse> getUserById

// Press Ctrl+Right Arrow again:
public ResponseEntity<UserResponse> getUserById(@PathVariable

// Continue accepting word by word until you have what you want
// Press Esc to dismiss the rest if you want to diverge
```

### When to Use Partial Acceptance

- The suggestion is 80% right but you want to change a detail
- You want the method signature but not the body
- The variable names are wrong but the structure is correct
- You want to accept the first line of a multi-line suggestion

---

## Writing Comments to Guide Suggestions

Comments are the single most effective way to steer Copilot. Think of comments as prompts.

### Basic Pattern

```java
// Create a method that validates an email address using regex
public boolean isValidEmail(String email) {
    // Copilot generates the implementation
}
```

### Detailed Comments Give Better Results

```java
// REST endpoint: GET /api/v1/orders
// Query parameters: status (optional, enum: PENDING/SHIPPED/DELIVERED),
//                   page (default 0), size (default 20)
// Returns: Page<OrderResponse> with 200 OK
// Requires: ROLE_USER or ROLE_ADMIN
@GetMapping("/api/v1/orders")
// Copilot now has enough context to generate a complete, accurate method
```

### Step-by-Step Comments

```java
public OrderResponse processOrder(CreateOrderRequest request) {
    // 1. Validate the request
    // 2. Check inventory availability for each item
    // 3. Calculate total price with tax
    // 4. Create the order entity
    // 5. Publish OrderCreated event to Kafka
    // 6. Return the response DTO
}
```

Copilot will generate code for each numbered step.

### Architecture-Level Comments

```java
/**
 * Kafka consumer for processing payment events.
 * Listens to the "payment-completed" topic.
 * Updates order status to PAID.
 * Idempotent — uses event ID for deduplication.
 * Retries up to 3 times, then sends to DLQ.
 */
@KafkaListener(topics = "payment-completed")
// Copilot generates a robust consumer based on these requirements
```

---

## Context Awareness

### What Copilot Sees

Copilot's context includes far more than just the current line. Understanding this helps you provide better context.

#### 1. Current File (Highest Priority)

Everything in the file you are editing. Copilot sees:
- Imports at the top
- Class annotations
- Other methods in the class
- Field declarations

**Tip**: Keep relevant imports and class-level annotations visible. They dramatically improve suggestions.

#### 2. Open Tabs (High Priority)

Files currently open in your editor tabs. Copilot prioritizes:
- Files with similar names (e.g., `UserService.java` when editing `UserController.java`)
- Recently accessed files
- Files in the same package/directory

**Tip**: Before working on a Controller, open the corresponding Service, Repository, Entity, and DTO files in tabs.

#### 3. File Name and Path

The file path `src/main/java/com/example/order/service/OrderServiceImpl.java` tells Copilot:
- Language: Java
- Framework: Spring Boot (from path structure)
- Layer: Service implementation
- Domain: Order

**Tip**: Use descriptive, conventional file names.

#### 4. Neighboring Code

Code immediately above and below the cursor gets the most weight.

```java
@Service
@RequiredArgsConstructor
@Slf4j
public class OrderServiceImpl implements OrderService {

    private final OrderRepository orderRepository;
    private final KafkaTemplate<String, OrderEvent> kafkaTemplate;
    private final InventoryClient inventoryClient;

    // When you start typing here, Copilot knows about all three dependencies
    // and will use them appropriately in its suggestions
}
```

---

## Multi-Line and Function-Level Completions

### Triggering Multi-Line Completions

Copilot generates multi-line completions when:

1. You write a method signature and open the body:

```java
public List<OrderResponse> getOrdersByCustomer(Long customerId, Pageable pageable) {
    // Copilot suggests the entire method body
```

2. You write a descriptive comment:

```java
// Method to convert Order entity to OrderResponse DTO, including nested address mapping
```

3. You start a class/interface definition:

```java
public record CreateOrderRequest(
    // Copilot suggests all fields with validation annotations
```

### Getting Complete Class Generation

Start with the class declaration and a Javadoc comment:

```java
/**
 * JPA entity representing a customer order.
 * Fields: id, customerId, items (one-to-many), totalAmount, status, createdAt, updatedAt.
 * Uses optimistic locking with @Version.
 */
@Entity
@Table(name = "orders")
public class Order {
    // Copilot generates the entire entity with all annotations
```

---

## Copilot Completion Panel

### Opening the Panel

Press `Ctrl + Enter` (VS Code) to open the Copilot completions panel. This shows up to 10 different suggestions side by side.

### When to Use It

- You want to see multiple approaches to the same problem
- The default suggestion is not quite right
- You are generating boilerplate and want the best version
- You are writing tests and want to see different assertion styles

### Example Workflow

```java
// Write a method to calculate the total price of items with discount
public BigDecimal calculateTotal(List<OrderItem> items, BigDecimal discountPercent) {
    // Place cursor here
    // Press Ctrl+Enter
    // Panel shows 10 variations:
    //   Option 1: Stream with reduce
    //   Option 2: Stream with map + sum
    //   Option 3: For loop
    //   Option 4: Stream with BigDecimal.ZERO accumulator
    //   etc.
    // Click "Accept" on the one you prefer
```

---

## Practical Examples: Spring Boot

### Example 1: REST Controller

Open a file named `ProductController.java` and type:

```java
package com.example.catalog.controller;

import org.springframework.web.bind.annotation.*;
import org.springframework.http.ResponseEntity;
import com.example.catalog.dto.*;
import com.example.catalog.service.ProductService;
import jakarta.validation.Valid;
import lombok.RequiredArgsConstructor;

/**
 * REST controller for product catalog management.
 * Supports CRUD operations and search.
 */
@RestController
@RequestMapping("/api/v1/products")
@RequiredArgsConstructor
public class ProductController {

    private final ProductService productService;

    // GET all products with pagination and optional category filter
```

Copilot will suggest:

```java
    @GetMapping
    public ResponseEntity<Page<ProductResponse>> getProducts(
            @RequestParam(required = false) String category,
            @PageableDefault(size = 20, sort = "name") Pageable pageable) {
        return ResponseEntity.ok(productService.getProducts(category, pageable));
    }
```

Continue typing comments for each endpoint and Copilot will generate the entire controller.

### Example 2: Service Layer

```java
@Service
@RequiredArgsConstructor
@Slf4j
public class ProductServiceImpl implements ProductService {

    private final ProductRepository productRepository;
    private final ProductMapper productMapper;
    private final KafkaTemplate<String, ProductEvent> kafkaTemplate;

    // Create a new product, save to DB, publish event to Kafka topic "product-created"
```

Copilot suggests:

```java
    @Override
    @Transactional
    public ProductResponse createProduct(CreateProductRequest request) {
        log.info("Creating product: {}", request.name());
        Product product = productMapper.toEntity(request);
        Product saved = productRepository.save(product);
        kafkaTemplate.send("product-created", saved.getId().toString(),
                new ProductEvent(saved.getId(), ProductEventType.CREATED));
        return productMapper.toResponse(saved);
    }
```

### Example 3: Repository with Custom Queries

```java
public interface OrderRepository extends JpaRepository<Order, Long> {

    // Find orders by customer ID with status filter, ordered by creation date desc
```

Copilot suggests:

```java
    List<Order> findByCustomerIdAndStatusOrderByCreatedAtDesc(Long customerId, OrderStatus status);

    // Find total revenue for a date range
    @Query("SELECT SUM(o.totalAmount) FROM Order o WHERE o.createdAt BETWEEN :start AND :end")
    BigDecimal calculateRevenueBetween(@Param("start") LocalDateTime start, @Param("end") LocalDateTime end);

    // Find top 10 customers by order count
    @Query("SELECT o.customerId, COUNT(o) as orderCount FROM Order o GROUP BY o.customerId ORDER BY orderCount DESC")
    List<Object[]> findTopCustomersByOrderCount(Pageable pageable);
```

### Example 4: DTO as Record

```java
// Request DTO for creating a new product with validation
public record CreateProductRequest(
```

Copilot suggests:

```java
        @NotBlank(message = "Product name is required")
        @Size(max = 255, message = "Product name must not exceed 255 characters")
        String name,

        @Size(max = 1000, message = "Description must not exceed 1000 characters")
        String description,

        @NotNull(message = "Price is required")
        @Positive(message = "Price must be positive")
        BigDecimal price,

        @NotBlank(message = "Category is required")
        String category,

        @Min(value = 0, message = "Stock quantity cannot be negative")
        int stockQuantity
) {}
```

### Example 5: Exception Handler

```java
@ControllerAdvice
public class GlobalExceptionHandler {

    // Handle resource not found, return 404 with ProblemDetail
```

Copilot suggests:

```java
    @ExceptionHandler(ResourceNotFoundException.class)
    public ResponseEntity<ProblemDetail> handleNotFound(ResourceNotFoundException ex) {
        ProblemDetail problemDetail = ProblemDetail.forStatusAndDetail(
                HttpStatus.NOT_FOUND, ex.getMessage());
        problemDetail.setTitle("Resource Not Found");
        problemDetail.setProperty("timestamp", Instant.now());
        return ResponseEntity.status(HttpStatus.NOT_FOUND).body(problemDetail);
    }
```

---

## Practical Examples: DevOps Files

### Example 6: Dockerfile

Create a file named `Dockerfile` and type:

```dockerfile
# Multi-stage Dockerfile for Spring Boot application with Java 21
# Stage 1: Build with Maven
```

Copilot suggests:

```dockerfile
FROM eclipse-temurin:21-jdk AS build
WORKDIR /app
COPY pom.xml .
COPY src ./src
RUN ./mvnw clean package -DskipTests

# Stage 2: Run
FROM eclipse-temurin:21-jre
WORKDIR /app
COPY --from=build /app/target/*.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

### Example 7: Kubernetes Deployment

Create `k8s/deployment.yaml` and type:

```yaml
# Kubernetes deployment for order-service Spring Boot app
# 3 replicas, resource limits, health checks, config from ConfigMap
apiVersion: apps/v1
kind: Deployment
```

Copilot generates a complete deployment with readiness/liveness probes, resource limits, and ConfigMap references.

### Example 8: Terraform

Create `main.tf` and type:

```hcl
# Azure Kubernetes Service (AKS) cluster with 3 node pools
# System pool: 2 nodes, Standard_D4s_v3
# Application pool: 3-10 nodes (autoscale), Standard_D8s_v3
# Monitoring pool: 1 node, Standard_D2s_v3

resource "azurerm_kubernetes_cluster" "main" {
```

Copilot suggests the complete AKS resource with all specified node pools.

### Example 9: SQL Migration

Create `V003__add_order_items_table.sql` and type:

```sql
-- Create order_items table with foreign key to orders
-- Columns: id, order_id, product_id, quantity, unit_price, total_price
```

Copilot suggests:

```sql
CREATE TABLE order_items (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    order_id BIGINT NOT NULL,
    product_id BIGINT NOT NULL,
    quantity INT NOT NULL,
    unit_price DECIMAL(10, 2) NOT NULL,
    total_price DECIMAL(10, 2) NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    CONSTRAINT fk_order_items_order FOREIGN KEY (order_id) REFERENCES orders(id) ON DELETE CASCADE,
    INDEX idx_order_items_order_id (order_id),
    INDEX idx_order_items_product_id (product_id)
);
```

### Example 10: GitHub Actions Workflow

Create `.github/workflows/ci.yml` and type:

```yaml
# CI pipeline for Spring Boot application
# Steps: checkout, setup Java 21, cache Maven, run tests, build Docker image, push to ACR
name: CI Pipeline
```

Copilot generates the complete workflow with all specified steps.

---

## Tips for Better Suggestions

### 1. Name Things Well

```java
// Bad — vague name, vague suggestions
public void process(Object data) {

// Good — descriptive name, precise suggestions
public OrderConfirmation processPaymentAndConfirmOrder(PaymentRequest payment) {
```

### 2. Start with Imports

Adding the right imports at the top of the file primes Copilot:

```java
import org.springframework.kafka.annotation.KafkaListener;
import org.springframework.kafka.support.Acknowledgment;
import com.example.event.PaymentEvent;
// Now when you write a method, Copilot knows to use Kafka patterns
```

### 3. Open Related Files in Tabs

Before working on `OrderController.java`, open:
- `OrderService.java`
- `OrderRepository.java`
- `Order.java`
- `OrderResponse.java`

Copilot uses these as context.

### 4. Use Method Signatures as Anchors

Write the full method signature first, then let Copilot fill in the body:

```java
public Page<OrderResponse> searchOrders(
        String customerName,
        OrderStatus status,
        LocalDate fromDate,
        LocalDate toDate,
        Pageable pageable) {
    // Copilot now has all parameter context to generate the query
```

### 5. Write One Method, Then the Next Gets Better

Each method you write (or accept) adds to the context for subsequent suggestions. The 5th method in a class is usually much better than the 1st.

### 6. Use Copilot's Momentum

If Copilot starts generating good code, keep pressing Tab. It builds on its own suggestions and maintains consistency.

### 7. When Suggestions Are Wrong, Type a Few Characters

If the ghost text is wrong, type a few characters of the correct code and wait. Copilot will adjust.

---

## Try This Exercises

### Exercise 1: Comment-Driven Development

1. Create a new file `OrderService.java`
2. Write only comments describing 5 methods (no code)
3. Go back to each comment and let Copilot generate the implementation
4. Compare the generated code with what you would have written

### Exercise 2: Partial Acceptance Practice

1. Let Copilot suggest a full method
2. Accept it word-by-word using `Ctrl+Right Arrow`
3. At some point, dismiss with `Esc` and type your own variation
4. Practice this 5 times until it feels natural

### Exercise 3: Completion Panel Exploration

1. Write a method signature for `calculateShippingCost(Order order, Address destination)`
2. Press `Ctrl+Enter` to open the completions panel
3. Review all 10 suggestions — note the different approaches
4. Accept the one you prefer

### Exercise 4: DevOps File Generation

1. Create a new `docker-compose.yml` with a comment describing: Spring Boot app + MySQL + Kafka + Redis
2. Let Copilot generate the entire file
3. Review and fix any issues
4. Do the same for a `Dockerfile` and a Kubernetes `deployment.yaml`

### Exercise 5: Context Experiment

1. Close all editor tabs except one Java file
2. Generate a method — note the quality
3. Now open 5 related files (Entity, DTO, Repository, etc.) in tabs
4. Generate the same method again — compare the quality

---

## Next Steps

Now that you can get effective code completions, move to [Tutorial 3: Copilot Chat & Edits](03-copilot-chat-and-edits.md) to learn how to have conversations with Copilot for explanations, refactoring, and multi-file edits.
