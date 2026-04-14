# Day 37 — Microservices Patterns & Decomposition
**Date:** 2026-04-13
**Phase:** 3 — Microservices & Cloud-Native
**Time Target:** 1–2 hours

---

## 1. Today's Focus Topic
**Microservices Architecture — Patterns, Decomposition, and Communication**

You've spent Phase 2 mastering system design at the component level. Phase 3 zooms in on how you actually build those components: decomposing a monolith into microservices, choosing the right communication patterns, ensuring data ownership, and avoiding the most common traps. This foundation is essential before diving into API Gateway, Service Mesh, and Kubernetes in the coming days.

---

## 2. Why Microservices?

### The Core Problem Microservices Solve

| Monolith Problem | Microservices Solution |
|------------------|----------------------|
| One team blocks others | Independent deployment per service |
| Scale the whole app for one hotspot | Scale only the bottleneck service |
| One bug can crash everything | Failure isolation via bulkheads |
| One tech stack forever | Polyglot — use the right tool per service |
| Slow build/deploy cycles | Small codebases, fast CI/CD per service |

### The Tradeoffs (be honest in interviews)

| Gain | Cost |
|------|------|
| Independent deployability | Distributed system complexity |
| Granular scalability | Network latency between services |
| Tech flexibility | Operational overhead (many services to monitor) |
| Team autonomy | Data consistency challenges |
| Fault isolation | Harder end-to-end testing |

---

## 3. Decomposition Strategies

### 3.1 Decompose by Domain (DDD Bounded Contexts)

The most reliable strategy. Domain-Driven Design (DDD) gives you a principled way to find service boundaries.

**Bounded Context:** A logical boundary within which a domain model is consistent and meaningful.

```
E-commerce Domain → Bounded Contexts:
├── Order Management     → Order Service
├── Product Catalog      → Catalog Service
├── Customer Identity    → Identity Service
├── Inventory Tracking   → Inventory Service
├── Payment Processing   → Payment Service
├── Shipping & Logistics → Fulfillment Service
└── Notifications        → Notification Service
```

**Key DDD Building Blocks:**
- **Entity:** Object with a unique identity (Order, Customer)
- **Value Object:** Immutable, no identity (Address, Money)
- **Aggregate:** Cluster of entities with one Aggregate Root (Order + OrderItems)
- **Domain Event:** Something meaningful that happened (`OrderPlaced`, `PaymentFailed`)
- **Repository:** Interface for aggregate persistence

**Rule:** Each service owns its Aggregate Root and communicates cross-boundary via Domain Events or APIs.

### 3.2 Decompose by Business Capability

Organize around what the business *does*, not the data.

```
Business Capabilities:
├── Customer Acquisition → User Service
├── Product Discovery    → Search + Catalog Service
├── Transaction          → Cart + Order + Payment Service
├── Post-Purchase        → Fulfillment + Returns Service
└── Customer Retention   → Loyalty + Notification Service
```

### 3.3 Strangler Fig Pattern (Migrating from Monolith)

The safest approach when decomposing an existing monolith.

```
Step 1: Keep monolith running
         Requests → Monolith

Step 2: Extract first service (e.g., Notifications)
         Requests → API Gateway → Notification Service
                              → Monolith (everything else)

Step 3: Iteratively extract more services
         Requests → API Gateway → Service A
                              → Service B
                              → Monolith (shrinking)

Step 4: Monolith is eventually gone (or irrelevant)
```

**Why it works:** No big-bang rewrite. Each extracted service is independently testable and deployable. Risk is incremental.

---

## 4. Communication Patterns

### 4.1 Synchronous — REST / gRPC

**Use when:** Caller needs an immediate answer (user-facing queries, blocking workflows).

```java
// Spring Boot REST — Order Service calls Inventory Service
@Service
public class OrderService {
    
    private final RestTemplate restTemplate;  // or WebClient for reactive
    
    public OrderResult placeOrder(OrderRequest request) {
        // Synchronous call — blocks until inventory responds
        InventoryCheckResponse inv = restTemplate.getForObject(
            "http://inventory-service/api/v1/check?sku=" + request.getSku() 
                + "&qty=" + request.getQuantity(),
            InventoryCheckResponse.class
        );
        
        if (!inv.isAvailable()) {
            throw new InsufficientStockException(request.getSku());
        }
        // Proceed with order creation...
    }
}
```

**gRPC advantages over REST:**
- Binary protocol (Protobuf) — 5–10× smaller payloads
- Strongly typed contracts via `.proto` files
- Bidirectional streaming
- Ideal for internal service-to-service (not public APIs)

### 4.2 Asynchronous — Kafka / Message Queues

**Use when:** Caller doesn't need immediate response, or event fan-out to multiple consumers.

```java
// Order Service publishes event (fire-and-forget)
@Service
public class OrderEventPublisher {
    
    @Autowired
    private KafkaTemplate<String, OrderPlacedEvent> kafkaTemplate;
    
    public void publishOrderPlaced(Order order) {
        OrderPlacedEvent event = OrderPlacedEvent.builder()
            .orderId(order.getId())
            .customerId(order.getCustomerId())
            .totalAmount(order.getTotal())
            .items(order.getItems())
            .occurredAt(Instant.now())
            .build();
        
        kafkaTemplate.send("order.placed", order.getId(), event);
    }
}

// Inventory Service consumes independently
@KafkaListener(topics = "order.placed", groupId = "inventory-group")
public void onOrderPlaced(OrderPlacedEvent event) {
    inventoryService.reserveStock(event.getItems());
}

// Notification Service also consumes
@KafkaListener(topics = "order.placed", groupId = "notification-group")
public void onOrderPlaced(OrderPlacedEvent event) {
    notificationService.sendOrderConfirmation(event.getCustomerId());
}
```

### 4.3 Choosing Sync vs Async

| Scenario | Pattern | Reasoning |
|----------|---------|-----------|
| Check inventory before ordering | Sync (REST) | Need answer to proceed |
| Notify customer after order | Async (Kafka) | No need to wait |
| Reserve stock after order | Async (Kafka) | Eventual consistency OK |
| Fetch product details for display | Sync (REST) | User is waiting |
| Process payment | Sync (REST) | Need confirmation |
| Update analytics after payment | Async (Kafka) | Fire and forget |

---

## 5. Data Ownership — One Database Per Service

**Golden Rule:** Each service owns its data. No shared databases.

```
WRONG — Shared Database (anti-pattern):
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│ Order Svc   │     │ Inventory   │     │ Customer    │
│             │     │ Service     │     │ Service     │
└──────┬──────┘     └──────┬──────┘     └──────┬──────┘
       │                   │                   │
       └─────────┬─────────┘                   │
                 │         ┌────────────────────┘
                 ▼         ▼
            ┌─────────────────┐
            │  Shared MySQL   │  ← Tight coupling!
            └─────────────────┘

RIGHT — Database Per Service:
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│ Order Svc   │     │ Inventory   │     │ Customer    │
│             │     │ Service     │     │ Service     │
└──────┬──────┘     └──────┬──────┘     └──────┬──────┘
       │                   │                   │
       ▼                   ▼                   ▼
  ┌─────────┐        ┌──────────┐        ┌──────────┐
  │ Orders  │        │Inventory │        │Customers │
  │  MySQL  │        │  MySQL   │        │PostgreSQL│
  └─────────┘        └──────────┘        └──────────┘
```

**Consequence:** Cross-service queries become cross-service API calls or event joins.

**Query patterns for cross-service data:**

```java
// Pattern 1: API Composition (real-time join via API calls)
@RestController
public class OrderDashboardController {
    
    public OrderDashboardResponse getDashboard(String orderId) {
        Order order = orderService.find(orderId);
        CustomerProfile customer = customerClient.getProfile(order.getCustomerId());
        List<Product> products = catalogClient.getProducts(order.getSkus());
        
        return OrderDashboardResponse.builder()
            .order(order)
            .customer(customer)
            .products(products)
            .build();
    }
}

// Pattern 2: CQRS Read Model (pre-joined materialized view)
// Order Service publishes events → Query Service builds a denormalized read model
// Query Service stores pre-joined data in its own DB for fast reads
```

---

## 6. Key Patterns Reference

### 6.1 Sidecar Pattern
Deploy a helper container alongside the main service container. The sidecar handles cross-cutting concerns without touching your business logic.

```yaml
# Kubernetes Pod — App + Sidecar
spec:
  containers:
  - name: order-service        # Your app
    image: order-service:1.0
    ports:
    - containerPort: 8080
  
  - name: envoy-proxy          # Sidecar: handles mTLS, tracing, circuit breaking
    image: envoy:1.28
    ports:
    - containerPort: 15001
```

**Common sidecar uses:** mTLS termination, distributed tracing, log shipping, config updates, health checks.

### 6.2 Saga Pattern (Distributed Transactions)
When an operation spans multiple services, use Sagas instead of 2PC.

```
Order Saga — Choreography Style (event-driven):

Order Svc → publishes OrderPlaced
                  ↓
Inventory Svc → reserves stock → publishes StockReserved
                                         ↓
Payment Svc → charges card → publishes PaymentCharged
                                    ↓
Fulfillment Svc → creates shipment → publishes ShipmentCreated

On failure → each service publishes compensating event to undo prior steps
```

### 6.3 Circuit Breaker (Resilience4j)

```java
@CircuitBreaker(name = "inventory", fallbackMethod = "inventoryFallback")
public InventoryCheckResponse checkInventory(String sku, int qty) {
    return inventoryClient.check(sku, qty);
}

public InventoryCheckResponse inventoryFallback(String sku, int qty, Exception ex) {
    // Return cached data or conservative "available" response
    return InventoryCheckResponse.available(); // Optimistic fallback
}
```

---

## 7. Microservices Anti-Patterns

| Anti-Pattern | Symptom | Fix |
|-------------|---------|-----|
| Distributed Monolith | Services must deploy together | Fix service boundaries (DDD) |
| Chatty Services | 10+ sync calls per user request | Async events, API composition, BFF |
| Shared Database | Two services write to same table | Migrate to DB-per-service |
| Too Fine-Grained | Nanoservices with 3 endpoints each | Merge into cohesive service |
| Synchronous Chains | A → B → C → D all sync | Introduce async messaging |
| No Circuit Breakers | Cascade failures across services | Add Resilience4j |
| Missing Idempotency | Duplicate events cause duplicate records | Idempotency keys + dedup |

---

## 8. Spring Boot Microservice Skeleton

```java
// Standard microservice structure
@SpringBootApplication
@EnableDiscoveryClient  // Register with Eureka/K8s
public class OrderServiceApplication {
    public static void main(String[] args) {
        SpringApplication.run(OrderServiceApplication.class, args);
    }
}

// application.yml
spring:
  application:
    name: order-service
  datasource:
    url: jdbc:mysql://order-db:3306/orders  # Own DB
  kafka:
    bootstrap-servers: kafka:9092

management:
  endpoints:
    web:
      exposure:
        include: health,info,prometheus,metrics
  tracing:
    sampling:
      probability: 1.0  # 100% trace sampling (reduce in prod)
```

---

## 9. Interview Questions

**Q: How do you decide service boundaries?**
A: Start with DDD bounded contexts — identify Aggregate Roots and the teams/domains that own them. Each bounded context typically maps to a service. Avoid splitting by technical layer (e.g., a "database service") — split by business capability.

**Q: How do you handle transactions across services?**
A: Use the Saga pattern — either choreography (event-driven, loose coupling) or orchestration (saga orchestrator coordinates steps). Avoid 2PC in microservices — it creates tight coupling and is impractical at scale.

**Q: How do you query data owned by multiple services?**
A: Two options — API Composition (real-time, simpler, higher latency) or CQRS with a read model (pre-joined, fast reads, eventual consistency). Choose based on freshness requirements and read volume.

**Q: What's the Strangler Fig pattern?**
A: A migration strategy — incrementally extract services from a monolith, routing traffic for each capability to the new service as it's ready. The monolith "strangles" over time. Zero big-bang rewrites, zero downtime.

---

## 10. Today's Practice Exercise

Design the microservices decomposition for a **hotel booking platform**:

1. Identify 5–7 bounded contexts and their aggregate roots
2. Choose sync vs async for: search availability, make reservation, send confirmation, charge payment
3. Draw the service dependency graph — flag any synchronous chains deeper than 2 hops
4. Design the Saga for "Book Room" — what are the steps and compensating actions?

---

## 11. Key Takeaways

- Decompose by **domain (DDD)** not by technical layer
- **Data ownership per service** is non-negotiable — no shared databases
- **Sync** for user-facing blocking calls; **async** for events and decoupled workflows
- Use **Strangler Fig** when migrating from monolith — incrementally, not big-bang
- **Sagas** replace distributed transactions — choreography or orchestration
- The **Sidecar** pattern handles cross-cutting concerns (mTLS, tracing) without polluting business code

---

## 12. Resources

- **Building Microservices** — Sam Newman (the definitive book)
- **Domain-Driven Design** — Eric Evans
- [Martin Fowler — Microservices](https://martinfowler.com/articles/microservices.html)
- Spring Boot Microservices: `spring-boot-starter-web` + `spring-cloud-starter-netflix-eureka-client`

---

*Next: Day 38 — API Gateway & Backend for Frontend (BFF) Pattern*
