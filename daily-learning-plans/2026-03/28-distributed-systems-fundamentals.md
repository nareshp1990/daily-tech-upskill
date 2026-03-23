# Day 21 — Distributed Systems Fundamentals
**Date:** 2026-03-28
**Phase:** 2 — System Design
**Time Target:** 1–2 hours

---

## 1. Today's Focus Topic
**Distributed Systems Fundamentals — CAP Theorem, Consistency Models, Consensus, and Failure Modes**

Every microservice system is a distributed system. Today you formalise the theory behind the trade-offs you make daily: why you can't have both perfect consistency and perfect availability, what "eventual consistency" actually means, how distributed consensus works, and how to design for the failures that will happen.

---

## 2. Core Concepts

### 2.1 The CAP Theorem

```
         Consistency
            /\
           /  \
          /    \
         / CA   \
        /  (RDBMS\
       /  single  \
      /   node)    \
     /──────────────\
    / CP          AP \
   / (ZooKeeper,  (Cassandra, \
  /  etcd, HBase)  DynamoDB)   \
 /________________________________\
Partition Tolerance   Availability
```

**CAP says:** In a network partition, you must choose between Consistency (every read gets the latest write) and Availability (every request gets a response).

**In practice:**
- Network partitions **will** happen — so you're always choosing between CP and AP.
- **CP systems** (e.g., ZooKeeper, etcd): Refuse to serve stale data during a partition. Prefer correctness.
- **AP systems** (e.g., Cassandra, DynamoDB): Continue serving data during a partition, but it may be stale. Prefer availability.

**The real question for your system:** "What happens when the network between service A and service B breaks?" Do you return an error (CP) or return possibly-stale data (AP)?

### 2.2 Consistency Models

| Model | Guarantee | Example |
|-------|-----------|---------|
| **Strong (Linearisable)** | Reads always see the latest write | Single MySQL instance, ZooKeeper |
| **Sequential** | All clients see operations in the same order | Transactions on a single DB |
| **Causal** | Causally related operations are seen in order | "Reply appears after the message it replies to" |
| **Eventual** | All replicas converge given enough time | DNS, Cassandra, DynamoDB |
| **Read-your-writes** | You see your own writes immediately | Session-scoped guarantee |

**Most microservice systems use eventual consistency** between services (via events/Kafka) and strong consistency within a service (via its own database).

### 2.3 The PACELC Theorem (CAP Extended)

```
If there's a Partition:             Else (normal operation):
  Choose Availability or             Choose Latency or
  Consistency (PA or PC)             Consistency (EL or EC)

Examples:
  Cassandra: PA/EL  → Available during partition, low latency normally (eventually consistent)
  MySQL:     PC/EC  → Consistent during partition, consistent normally (higher latency)
  DynamoDB:  PA/EL  → Same as Cassandra
  ZooKeeper: PC/EC  → Same as MySQL
```

### 2.4 Failure Modes in Distributed Systems

| Failure | What Happens | Impact |
|---------|-------------|--------|
| **Crash failure** | Node stops responding | Simplest — health check detects, LB removes |
| **Omission failure** | Messages lost (network) | Timeout → retry → idempotency required |
| **Timing failure** | Response too slow | Timeout triggers fallback |
| **Byzantine failure** | Node behaves incorrectly (bugs, corruption) | Hardest — needs checksums, consensus |
| **Network partition** | Nodes can't communicate | CAP trade-off kicks in |

### 2.5 Distributed Consensus — Raft

When multiple nodes need to agree on a value (leader election, config change), they use a consensus algorithm.

```
Raft Leader Election:

  Node A (Follower)    Node B (Leader)    Node C (Follower)
       │                    │                    │
       │    heartbeat       │    heartbeat       │
       │←───────────────────│───────────────────→│
       │                    │                    │
       │                    │ ✗ (crashes)        │
       │                    │                    │
       │  election timeout  │                    │
       │  ... no heartbeat  │                    │
       │                    │                    │
  Node A: "I'm candidate,  │                    │
           vote for me"     │                    │
       │─────────────────────────────────────────→│
       │                                          │
       │←─────────── "yes, you're leader" ───────│
       │                                          │
  Node A (Leader) ←────── new leader ────────────→ Node C (Follower)
```

**Where you see Raft in practice:**
- **etcd** (Kubernetes control plane) — cluster state consensus.
- **Consul** — service discovery and config.
- **CockroachDB** — distributed SQL consensus.

### 2.6 Clocks and Ordering

**Wall clocks are unreliable** in distributed systems — machines drift (milliseconds to seconds).

**Solutions:**
| Approach | How | Used By |
|----------|-----|---------|
| **Logical clocks (Lamport)** | Counter incremented on each event | Most distributed systems |
| **Vector clocks** | Per-node counters detect concurrent events | DynamoDB, Riak |
| **Hybrid logical clocks** | Physical time + logical counter | CockroachDB |
| **TrueTime** | GPS + atomic clocks with bounded uncertainty | Google Spanner |

---

## 3. Patterns for Handling Distributed Failures

### 3.1 Saga Pattern (Distributed Transactions)

Since you can't use a single ACID transaction across microservices, use a saga: a sequence of local transactions with compensating actions.

```
Order Saga (Choreography):

  OrderService          PaymentService       InventoryService
       │                      │                     │
  1. Create order             │                     │
  2. Publish OrderCreated ──→ │                     │
       │                 3. Charge card              │
       │                 4. Publish PaymentDone ───→ │
       │                      │                5. Reserve stock
       │                      │                6. Publish StockReserved
       │                      │                     │

Compensation (if stock reservation fails):
       │                      │                     │
       │                      │                5. Publish StockFailed ──→
       │                 6. Refund card ←──────────  │
       │                 7. Publish PaymentRefunded   │
  8. Cancel order ←──────────  │                     │
```

### 3.2 Circuit Breaker (from Day 3, now generalised)

```java
// Resilience4j circuit breaker — same pattern from Phase 1 Day 3
@CircuitBreaker(name = "payment-service", fallbackMethod = "paymentFallback")
public PaymentResult processPayment(PaymentRequest request) {
    return paymentClient.charge(request);
}

public PaymentResult paymentFallback(PaymentRequest request, Exception ex) {
    // Queue for retry, return pending status
    retryQueue.enqueue(request);
    return PaymentResult.pending("Payment queued for retry");
}
```

### 3.3 Outbox Pattern (Reliable Event Publishing)

Ensures that a database write and an event publish happen atomically.

```
1. Single DB transaction:
   INSERT INTO orders (id, status) VALUES ('ord-123', 'CREATED');
   INSERT INTO outbox (event_type, payload) VALUES ('OrderCreated', '{...}');

2. Outbox poller (or CDC via Debezium):
   Read outbox → Publish to Kafka → Mark as published

Result: If the transaction commits, the event WILL be published.
        If it rolls back, no event is published.
        No dual-write problem.
```

---

## 4. Hands-On Exercise

### Task: Design failure handling for an order system

1. **Draw the architecture:** 3 services (Order, Payment, Inventory) communicating via Kafka.
2. **Design the saga** for order placement with compensation for each failure point:
   - What if payment fails?
   - What if inventory reservation fails after payment succeeds?
   - What if the Kafka broker is down?
3. **Implement the outbox pattern** for OrderService:
   - Outbox table schema.
   - Spring `@Scheduled` poller that reads unpublished events and sends to Kafka.
   - Mark events as published after successful send.
4. **Add a circuit breaker** on the Payment client with 3 failure types:
   - Timeout → retry.
   - 4xx → don't retry (bad request).
   - 5xx → circuit breaker → fallback to queue.

### Outbox Table Schema:
```sql
CREATE TABLE outbox (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    aggregate_type VARCHAR(100) NOT NULL,   -- 'Order'
    aggregate_id VARCHAR(100) NOT NULL,     -- 'ord-123'
    event_type VARCHAR(100) NOT NULL,       -- 'OrderCreated'
    payload JSON NOT NULL,
    created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    published_at DATETIME NULL,
    INDEX idx_unpublished (published_at, created_at)
);
```

---

## 5. Key Takeaways

1. **CAP is about partitions** — in normal operation, you can have both consistency and availability. The choice matters only when things break.
2. **Eventual consistency is the default between services** — embrace it and design for it (idempotency, compensation).
3. **Sagas replace distributed transactions** — each service owns its own transaction, compensation handles failures.
4. **The outbox pattern solves the dual-write problem** — never write to a database and a message broker in two separate operations.
5. **Consensus (Raft/Paxos) is expensive** — use it only where you truly need agreement (leader election, config), not for every operation.

---

## 6. Connection to Previous Days

The circuit breaker pattern appeared in Phase 1 Day 3 (LLM API resilience). The saga pattern mirrors the multi-agent orchestration from Day 11 — each agent is like a saga step with its own failure mode. The outbox pattern ensures reliable event publishing, complementing Day 19's Kafka patterns with transactional guarantees.

---

## 7. Resources

- **[Designing Data-Intensive Applications, Ch. 8–9](https://dataintensive.net/)** — The definitive chapters on distributed systems trouble and consistency.
- **[The Raft Consensus Algorithm (visualisation)](https://thesecretlivesofdata.com/raft/)** — Interactive Raft explanation.
- **[Microservices Patterns (Chris Richardson), Ch. 4](https://microservices.io/patterns/data/saga.html)** — Saga pattern in depth.

---

*Day 21. In a distributed system, everything that can fail will fail — design for it.*
