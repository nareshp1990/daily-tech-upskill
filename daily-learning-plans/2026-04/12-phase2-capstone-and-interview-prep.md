# Day 36 — Phase 2 Capstone & System Design Interview Prep
**Date:** 2026-04-12
**Phase:** 2 — System Design
**Time Target:** 1–2 hours

---

## 1. Today's Focus Topic
**Phase 2 Capstone — Mastering the System Design Interview**

You've spent 3 weeks building deep knowledge of system design. Today is synthesis day: you review everything, practice the interview framework, and connect all the patterns into a repeatable process you can apply to any design problem. After today, you should be able to walk through any system design question with confidence.

---

## 2. Phase 2 Full Recap

### Week 1 (Days 16-22) — Foundations
| Day | Topic | Interview Pattern |
|-----|-------|-----------------|
| 16 | Scalability Fundamentals | "Scale this to 10M users" |
| 17 | Database Design & Indexing | "Why is this query slow?" |
| 18 | Caching Strategies | "Reduce database load by 10×" |
| 19 | Message Queues & Event-Driven | "Decouple these services" |
| 20 | API Design | "Design the REST API for X" |
| 21 | Distributed Systems Fundamentals | "Explain CAP theorem" |
| 22 | Rate Limiting & Backpressure | "Handle traffic spikes" |

### Week 2 (Days 23-29) — Infrastructure Deep Dive
| Day | Topic | Interview Pattern |
|-----|-------|-----------------|
| 23 | Consistent Hashing | "How do you partition data?" |
| 24 | Service Discovery & LB | "How do services find each other?" |
| 25 | Circuit Breakers | "Prevent cascading failures" |
| 26 | Distributed Tracing | "Debug this latency issue" |
| 27 | DB Replication & Sharding | "Scale the database tier" |
| 28 | CDN & Edge Caching | "Reduce global latency" |
| 29 | Week 2 Capstone | Full integration |

### Week 3 (Days 30-35) — Case Studies
| Day | Case Study | Core Pattern |
|-----|-----------|-------------|
| 30 | URL Shortener | Base62, read-heavy caching |
| 31 | Social Feed | Fan-out on write vs read |
| 32 | Ride-Sharing | Geospatial, WebSockets, state machine |
| 33 | Payment System | Idempotency, double-entry, reconciliation |
| 34 | Search & Typeahead | Inverted index, Trie, Elasticsearch |
| 35 | Video Streaming | Transcoding pipeline, HLS, CDN |

---

## 3. The System Design Interview Framework

Use this framework for every design question. Don't skip steps.

### Step 1: Clarify Requirements (5 min)
```
Ask before designing:
□ Functional: What must it do? What's out of scope?
□ Scale: How many users? TPS? Data volume? Geographic spread?
□ SLA: Latency targets? Availability? Consistency (strong vs eventual)?
□ Constraints: Budget? Existing tech stack? Timeline?

"Before I start designing, let me confirm what we need to build..."
```

### Step 2: Capacity Estimation (3 min)
```
Write numbers on the whiteboard:
□ DAU / MAU
□ Reads/second and writes/second
□ Storage: how much data per day, per year
□ Bandwidth: data transferred per second

These numbers drive every subsequent decision.
```

### Step 3: High-Level Design (5 min)
```
Draw the boxes:
Client → API Gateway → Services → Databases
Add:
□ CDN (if content-heavy)
□ Cache layer (if read-heavy)
□ Message queue (if async needed)
□ Search (if search functionality)

"Let me draw a rough architecture and we can drill into any part."
```

### Step 4: Deep Dive (20 min)
```
The interviewer will ask you to go deeper on specific components.
For each component, cover:
□ Data model / schema
□ API design
□ Scaling strategy
□ Failure modes and mitigations
```

### Step 5: Trade-offs & Bottlenecks (5 min)
```
Always discuss trade-offs:
□ Where are the bottlenecks in this design?
□ What would break first at 10× scale?
□ What would you change if you had more time?
□ Consistency vs availability trade-off for this use case
```

---

## 4. The Decision Tree — Choosing the Right Pattern

```
New design problem
       │
       ▼
Is it read-heavy (>10:1 read:write)?
  YES → Add caching (Redis), read replicas, CDN for static content
  NO  → Proceed

Is write throughput the bottleneck?
  YES → Consider sharding (partition by user_id or natural domain key)
  NO  → Single primary with read replicas usually sufficient

Are there slow, resource-intensive operations?
  YES → Move to async (Kafka + worker service)
  NO  → Synchronous is simpler

Do services need to find each other dynamically?
  YES (k8s) → Kubernetes DNS + Services
  YES (non-k8s) → Eureka/Consul
  NO → Static config OK for small teams

Can a downstream service failure cascade?
  YES (it's on your hot path) → Add circuit breaker + timeout + fallback
  NO (async consumer) → Dead letter queue is sufficient

Is latency critical for global users?
  YES → CDN for static, GeoDNS for app tier, multi-region for databases
  NO → Single region is operationally simpler

Is data correctness non-negotiable? (financial, healthcare)
  YES → ACID transactions, idempotency keys, event sourcing, reconciliation
  NO → Eventual consistency is fine, higher throughput possible
```

---

## 5. The Golden Questions Interviewers Always Ask

**"How would you handle 100× traffic?"**
> Horizontal scaling (stateless services), CDN, caching, Kafka to buffer writes, DB read replicas. If still not enough: sharding.

**"What happens if this service goes down?"**
> Circuit breaker opens, fallback returns degraded response, retry when service recovers. Critical data in Kafka (durable) so nothing is lost.

**"How do you ensure no data is lost?"**
> Kafka with replication factor 3, acks=all. Database with sync replication for critical data. Idempotency keys for API retries. Reconciliation jobs for eventual correctness.

**"How do you debug a latency issue in production?"**
> Check distributed traces in Jaeger/Tempo for the slow span. Check Prometheus metrics (p99 latency per service). Check logs correlated by traceId. Look at DB slow query log if DB is the bottleneck.

**"How would you scale the database?"**
> Layer 1: Indexes and query optimisation. Layer 2: Caching with Redis (80/20 rule). Layer 3: Read replicas. Layer 4: Sharding (only if previous layers are exhausted).

---

## 6. Phase 2 Self-Assessment — Final Checklist

### Can you explain to a non-engineer?
- [ ] Why we use caching and what cache invalidation problems exist
- [ ] What happens when a microservice goes down
- [ ] Why databases can be a bottleneck and how we fix it

### Can you design from scratch?
- [ ] URL shortener (30 min)
- [ ] Social media feed (45 min)
- [ ] Payment system (45 min)

### Can you answer in interviews?
- [ ] CAP theorem and when to pick CP vs AP
- [ ] Consistent hashing vs modulo hashing
- [ ] Fan-out on write vs fan-out on read trade-offs
- [ ] When to use Kafka vs direct service calls
- [ ] How to make an API idempotent
- [ ] What observability signals you'd set up for a new service

---

## 7. Phase 3 Preview — Microservices & Cloud-Native

Starting Day 37 (2026-04-13):

| Day | Topic |
|-----|-------|
| 37 | Microservices Architecture Patterns — Decomposition, boundaries, anti-patterns |
| 38 | API Gateway & Backend-for-Frontend (BFF) |
| 39 | Service Mesh — Istio, mTLS, traffic management |
| 40 | Docker Fundamentals & Containerisation |
| 41 | Kubernetes Core — Pods, Deployments, Services, ConfigMaps |
| 42 | Kubernetes Advanced — HPA, PDB, resource limits, health probes |
| 43 | Phase 3 Week 1 Capstone |

---

## 8. Key Takeaways

1. **Requirements first, design second** — a beautiful design for the wrong requirements is worthless.
2. **Numbers drive decisions** — capacity estimation is not optional; it justifies your architecture choices.
3. **Every trade-off has two sides** — strong consistency costs availability and latency; always name both sides.
4. **The interview is a conversation** — think out loud, invite feedback, be collaborative.
5. **Simple is better than clever** — interviewers want to see you can reason clearly, not that you know every technology.

---

## 9. Resources

- **[Grokking the System Design Interview](https://www.educative.io/courses/grokking-the-system-design-interview)** — The classic course.
- **[System Design Primer (GitHub)](https://github.com/donnemartin/system-design-primer)** — Free, comprehensive reference.
- **[ByteByteGo by Alex Xu](https://bytebytego.com/)** — Visual system design explanations.
- **[Designing Data-Intensive Applications](https://dataintensive.net/)** — The definitive technical reference.

---

*Day 36. Phase 2 complete. You don't just build systems now — you design them with intent, trade-offs fully understood.*
