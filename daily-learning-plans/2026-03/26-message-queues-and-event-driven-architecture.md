# Day 19 — Message Queues & Event-Driven Architecture
**Date:** 2026-03-26
**Phase:** 2 — System Design
**Time Target:** 1–2 hours

---

## 1. Today's Focus Topic
**Message Queues & Event-Driven Architecture — Kafka Deep Dive, Patterns, and Spring Boot Integration**

You already use Kafka in production — today you go deeper. You'll formalise the patterns behind event-driven architecture, understand Kafka's guarantees at a protocol level, and learn when to use queues vs streams vs event sourcing. This is the system design perspective: not just "how to use Kafka" but "when and why to choose it."

---

## 2. Core Concepts

### 2.1 Synchronous vs Asynchronous Communication

```
Synchronous (REST):                Asynchronous (Event):
┌─────────┐  HTTP  ┌─────────┐    ┌─────────┐  Event  ┌───────┐  Event  ┌─────────┐
│ Order   │──────→│ Payment │    │ Order   │──────→│ Kafka │──────→│ Payment │
│ Service │←──────│ Service │    │ Service │       │       │       │ Service │
└─────────┘       └─────────┘    └─────────┘       └───────┘       └─────────┘
                                                      │
Coupled: both must                                    └──────→ ┌──────────┐
be up simultaneously                                           │ Inventory│
                                                               │ Service  │
                                  Decoupled: producer           └──────────┘
                                  doesn't know consumers
```

| Dimension | Synchronous | Asynchronous |
|-----------|-------------|--------------|
| Coupling | Tight (caller knows callee) | Loose (fire and forget) |
| Latency | Immediate response | Eventually processed |
| Failure handling | Caller retries or fails | Broker buffers, consumer retries |
| Scaling | Both sides scale together | Consumer scales independently |

### 2.2 Message Queue vs Event Stream

| Feature | Queue (RabbitMQ, SQS) | Stream (Kafka) |
|---------|----------------------|----------------|
| Consumption | Message consumed once, removed | Messages retained, replayable |
| Consumer model | Competing consumers | Consumer groups (each group gets all messages) |
| Ordering | Per-queue (FIFO) | Per-partition |
| Use case | Task distribution (work queue) | Event log, event sourcing, stream processing |

### 2.3 Kafka Architecture Deep Dive

```
                        Kafka Cluster
                 ┌──────────────────────────┐
                 │  Broker 1    Broker 2    Broker 3
                 │  ┌──────┐   ┌──────┐   ┌──────┐
  Topic:         │  │ P0   │   │ P1   │   │ P2   │   ← Partitions
  orders         │  │(lead)│   │(lead)│   │(lead)│
                 │  │      │   │      │   │      │
                 │  │ P1   │   │ P2   │   │ P0   │   ← Replicas
                 │  │(repl)│   │(repl)│   │(repl)│
                 │  └──────┘   └──────┘   └──────┘
                 └──────────────────────────┘

  Producer ────→ Partition (by key hash) ────→ Consumer Group
                                                ├── Consumer 1 → P0
                                                ├── Consumer 2 → P1
                                                └── Consumer 3 → P2
```

**Key guarantees:**
- **Ordering:** Guaranteed within a partition (not across partitions).
- **Durability:** Replicated across brokers. `acks=all` waits for all replicas.
- **At-least-once delivery:** Default. Consumer may see duplicates on rebalance.
- **Exactly-once:** Possible with idempotent producers + transactional consumers.

### 2.4 Kafka Delivery Semantics

| Setting | Guarantee | Trade-Off |
|---------|-----------|-----------|
| `acks=0` | Fire and forget | Fastest, may lose messages |
| `acks=1` | Leader acknowledged | Fast, may lose if leader crashes before replication |
| `acks=all` | All in-sync replicas acknowledged | Slowest, strongest durability |

**Consumer offset management:**
```
Partition: [msg0][msg1][msg2][msg3][msg4][msg5][msg6]
                              ↑                  ↑
                        committed offset    latest offset
                        (last processed)    (newest message)

Consumer lag = latest offset - committed offset = 3
```

### 2.5 Event-Driven Patterns

#### Pattern 1: Event Notification
Lightweight event signals something happened. Consumer fetches details if needed.

```json
{
  "eventType": "ORDER_PLACED",
  "orderId": "ord-123",
  "timestamp": "2026-03-26T10:30:00Z"
}
```

#### Pattern 2: Event-Carried State Transfer
Event contains the full state — consumers don't need to call back.

```json
{
  "eventType": "ORDER_PLACED",
  "orderId": "ord-123",
  "userId": "user-42",
  "items": [{"productId": "p-1", "quantity": 2, "price": 29.99}],
  "totalAmount": 59.98,
  "shippingAddress": { "city": "Bangalore", "zip": "560001" }
}
```

#### Pattern 3: Event Sourcing
Store events as the source of truth. Derive current state by replaying events.

```
Events:                              Current State (derived):
OrderCreated{items: [...]}           ┌──────────────────────┐
OrderItemAdded{item: X}       →     │ Order #123           │
OrderItemRemoved{item: Y}           │ Status: PAID         │
PaymentReceived{amount: 59.98}       │ Items: [A, X]        │
                                     │ Total: $59.98        │
                                     └──────────────────────┘
```

#### Pattern 4: CQRS (Command Query Responsibility Segregation)

```
  Commands (writes)              Queries (reads)
  ┌────────────┐                ┌────────────┐
  │ Order      │  events →      │ Order      │
  │ Command    │──────────→     │ Query      │
  │ Service    │   Kafka        │ Service    │
  │ (MySQL)    │                │ (Elastic-  │
  └────────────┘                │  search)   │
                                └────────────┘
Write model optimised            Read model optimised
for consistency                  for query patterns
```

---

## 3. Spring Boot + Kafka Integration

### 3.1 Producer

```java
@Service
@RequiredArgsConstructor
public class OrderEventPublisher {

    private final KafkaTemplate<String, OrderEvent> kafkaTemplate;

    public void publishOrderPlaced(Order order) {
        OrderEvent event = new OrderEvent(
            "ORDER_PLACED", order.getId(), order.getUserId(), Instant.now());

        kafkaTemplate.send("orders", order.getId(), event)   // key = orderId → same partition
            .whenComplete((result, ex) -> {
                if (ex != null) {
                    log.error("Failed to publish order event: {}", order.getId(), ex);
                } else {
                    log.info("Published ORDER_PLACED to partition {} offset {}",
                        result.getRecordMetadata().partition(),
                        result.getRecordMetadata().offset());
                }
            });
    }
}
```

### 3.2 Consumer with Error Handling

```java
@Component
@RequiredArgsConstructor
public class PaymentEventConsumer {

    private final PaymentService paymentService;

    @KafkaListener(
        topics = "orders",
        groupId = "payment-service",
        concurrency = "3"          // 3 consumer threads → 3 partitions
    )
    public void handleOrderEvent(
            @Payload OrderEvent event,
            @Header(KafkaHeaders.RECEIVED_PARTITION) int partition,
            @Header(KafkaHeaders.OFFSET) long offset) {

        log.info("Processing {} from partition {} offset {}", event.type(), partition, offset);

        if ("ORDER_PLACED".equals(event.type())) {
            paymentService.initiatePayment(event.orderId());
        }
    }
}

// Retry + DLT (Dead Letter Topic) configuration
@Configuration
public class KafkaConsumerConfig {

    @Bean
    public DefaultErrorHandler errorHandler(KafkaTemplate<String, Object> template) {
        DeadLetterPublishingRecoverer recoverer =
            new DeadLetterPublishingRecoverer(template);

        return new DefaultErrorHandler(recoverer,
            new FixedBackOff(1000L, 3));   // 3 retries, 1s apart, then DLT
    }
}
```

### 3.3 Idempotent Consumer

```java
@KafkaListener(topics = "orders", groupId = "inventory-service")
@Transactional
public void handleOrderEvent(@Payload OrderEvent event) {
    // Idempotency check: have we already processed this event?
    if (processedEventRepository.existsByEventId(event.eventId())) {
        log.info("Skipping duplicate event: {}", event.eventId());
        return;
    }

    inventoryService.reserveItems(event.orderId(), event.items());

    processedEventRepository.save(new ProcessedEvent(event.eventId(), Instant.now()));
}
```

---

## 4. Partitioning Strategy

| Key Strategy | Ordering Guarantee | Use Case |
|-------------|-------------------|----------|
| `orderId` | All events for an order in sequence | Order lifecycle |
| `userId` | All events for a user in sequence | User activity stream |
| `null` (round-robin) | No ordering, max throughput | Logs, metrics |
| `tenantId` | Tenant isolation | Multi-tenant SaaS |

**Partition count rule of thumb:** Start with `max(expected consumer instances, 6)`. You can increase partitions later but never decrease.

---

## 5. Hands-On Exercise

### Task: Design an event-driven order pipeline

1. **Define 4 events** for an order lifecycle: `OrderPlaced`, `PaymentProcessed`, `InventoryReserved`, `OrderShipped`.
2. **Choose the partition key** and justify why.
3. **Implement the producer** in Spring Boot that publishes `OrderPlaced`.
4. **Implement 2 consumers** in different consumer groups:
   - `payment-service`: listens for `OrderPlaced`, publishes `PaymentProcessed`.
   - `inventory-service`: listens for `OrderPlaced`, publishes `InventoryReserved`.
5. **Add idempotency** to both consumers.
6. **Add a Dead Letter Topic** for messages that fail after 3 retries.

### Stretch Goal
Implement a saga orchestrator that tracks the order through all 4 states and compensates (refund payment, release inventory) if any step fails.

---

## 6. Key Takeaways

1. **Events decouple services** — the producer doesn't know or care who consumes.
2. **Kafka ordering is per-partition** — choose your partition key to match your ordering needs.
3. **Idempotent consumers are mandatory** — at-least-once delivery means you will see duplicates.
4. **Dead Letter Topics are your safety net** — poison messages shouldn't block the pipeline.
5. **Event sourcing is powerful but complex** — use it when you need audit trails and time travel, not as a default.

---

## 7. Connection to Previous Days

In Phase 1 (Day 11), you used async fan-out with `CompletableFuture` for multi-agent orchestration. Event-driven architecture is the distributed equivalent — agents become services, function calls become events, and Kafka replaces in-process coordination. Yesterday's caching (Day 18) pairs naturally: cache the read model, invalidate on events.

---

## 8. Resources

- **[Kafka: The Definitive Guide](https://www.confluent.io/resources/kafka-the-definitive-guide-v2/)** — Free e-book from Confluent.
- **[Spring for Apache Kafka](https://docs.spring.io/spring-kafka/reference/)** — Official Spring Kafka reference.
- **[Martin Fowler: Event-Driven Architecture](https://martinfowler.com/articles/201701-event-driven.html)** — Patterns and trade-offs.

---

*Day 19. The best microservice communication is the one where neither service knows the other exists.*
