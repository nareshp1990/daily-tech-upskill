# Day 29 — Phase 2 Week 2 Capstone & Review
**Date:** 2026-04-05
**Phase:** 2 — System Design
**Time Target:** 1–2 hours

---

## 1. Today's Focus Topic
**Phase 2 Week 2 Capstone — Connecting All the Infrastructure Pieces**

Week 2 covered the deep infrastructure of distributed systems: how data is partitioned (consistent hashing), how services find each other (discovery + load balancing), how they survive failures (circuit breakers + resilience), how you see what's happening (observability), how the database scales (replication + sharding), and how content reaches users fast (CDN + edge). Today you connect all these into one coherent architecture and test your understanding.

---

## 2. Week 2 Recap

| Day | Topic | Core Takeaway |
|-----|-------|---------------|
| 23 | Consistent Hashing & Data Partitioning | Adding 1 node remaps only 1/N keys, not all |
| 24 | Service Discovery & Load Balancing | Eureka/Consul + @LoadBalanced; K8s DNS in production |
| 25 | Circuit Breakers & Resilience | Fail fast; Resilience4j: CB + Retry + Bulkhead + TimeLimiter |
| 26 | Distributed Tracing & Observability | Logs + Metrics + Traces; OTel + Jaeger + Prometheus |
| 27 | Database Replication & Sharding | Replication = read scale; Sharding = write scale |
| 28 | CDN, Edge Caching & Global Distribution | Edge = physical proximity; Cache-Control contract |

---

## 3. Full System Architecture (Week 1 + Week 2)

```
                         Internet
                             │
                    ┌────────▼────────┐
                    │   CDN (Edge)    │  ← Day 28: Static assets, public APIs
                    │  CloudFlare     │    Cache-Control headers
                    └────────┬────────┘
                             │ Cache miss
                    ┌────────▼────────┐
                    │  API Gateway    │  ← Day 20: Rate limiting (token bucket)
                    │  (Kong/Nginx)   │    Auth, routing, TLS termination
                    └────────┬────────┘
                             │
              ┌──────────────┼──────────────┐
              ▼              ▼              ▼
       ┌─────────┐    ┌─────────┐    ┌─────────┐
       │ Order   │    │ Payment │    │ User    │  ← Day 16: Horizontal scaling
       │ Service │    │ Service │    │ Service │    Day 24: Service discovery
       └────┬────┘    └────┬────┘    └────┬────┘
            │              │              │
            │    ┌─────────▼────────┐     │
            │    │ Eureka / K8s DNS │     │  ← Day 24: Registry
            │    └──────────────────┘     │
            │                             │
            └──────────────┬──────────────┘
                           │
              ┌────────────┼────────────┐
              ▼            ▼            ▼
       ┌────────────┐ ┌────────┐ ┌────────────┐
       │ Redis Cache│ │ Kafka  │ │Circuit Bkr │  ← Day 18: Cache-aside
       │ L1+L2      │ │Cluster │ │Resilience4j│    Day 19: Event-driven
       └────────────┘ └────┬───┘ └────────────┘    Day 25: Resilience
                           │
              ┌────────────┼────────────┐
              ▼            ▼            ▼
       ┌────────────┐ ┌────────┐ ┌────────────┐
       │  DB Primary│ │DB Shard│ │DB Shard    │  ← Day 27: Replication+Sharding
       │  + Replicas│ │   0    │ │    1       │    Day 17: Indexing
       └────────────┘ └────────┘ └────────────┘

       ┌────────────────────────────────────────┐
       │          Observability Stack           │  ← Day 26: OTel + Jaeger
       │  Prometheus + Grafana + Jaeger + Loki  │    Logs + Metrics + Traces
       └────────────────────────────────────────┘
```

---

## 4. Self-Assessment Checklist

### Consistent Hashing (Day 23)
- [ ] Can you explain why `key % N` fails when N changes?
- [ ] Can you draw the consistent hash ring with VNodes?
- [ ] Can you implement `getNode(String key)` in Java?
- [ ] When would you choose range partitioning over hash partitioning?

### Service Discovery (Day 24)
- [ ] What's the difference between client-side and server-side discovery?
- [ ] How does `@LoadBalanced RestTemplate` work under the hood?
- [ ] What happens when a service instance fails a health check?
- [ ] When do you need Eureka in a Kubernetes environment?

### Circuit Breakers (Day 25)
- [ ] Draw the three states of a circuit breaker and their transitions.
- [ ] What is the difference between a bulkhead and a rate limiter?
- [ ] Why is Retry dangerous for non-idempotent operations?
- [ ] Configure Resilience4j with CB + Retry + TimeLimiter from memory.

### Observability (Day 26)
- [ ] What are the three pillars of observability? What question does each answer?
- [ ] How do you correlate a trace to its logs using structured logging?
- [ ] What are the four golden signals?
- [ ] What does `s-maxage` mean in `Cache-Control`?

### Database Replication & Sharding (Day 27)
- [ ] What is replication lag and when does it cause problems?
- [ ] How does `@Transactional(readOnly=true)` route to replicas?
- [ ] What makes a good vs bad shard key?
- [ ] How do you generate globally unique IDs across shards?

### CDN & Edge (Day 28)
- [ ] What is the difference between `max-age` and `s-maxage`?
- [ ] When should you NEVER cache a response on a CDN?
- [ ] What are surrogate keys / cache tags used for?
- [ ] What can you do with edge computing that you can't do with a CDN alone?

---

## 5. Architecture Design Challenge

Design a **global e-commerce platform** handling:
- 1 million users, 100,000 daily orders
- Serving users in US, EU, and APAC
- Product catalog: 10 million products
- Personalised recommendations
- SLA: 99.99% uptime, p99 latency < 200ms

**Answer these design questions:**
1. What do you put on the CDN? What headers do you set?
2. How do you shard the orders table? What's your shard key?
3. Where do you place circuit breakers? Configure the thresholds.
4. How does service discovery work for payment-service?
5. What metrics do you monitor? What alerts do you set up?
6. How do you handle a database primary failure?

---

## 6. Common Interview Questions

**Q: How would you design the data tier for Twitter at scale?**
- Time-series tweets → range partition by tweet_id (Snowflake-style)
- Fan-out on write for small accounts, fan-out on read for celebrities
- Redis for hot timelines
- CDN for media (images, videos)

**Q: Service A calls Service B, which is slow. What do you add?**
1. Timeout (TimeLimiter) — don't wait forever
2. Circuit breaker — stop calling when B is consistently failing
3. Retry with jitter — recover from transient failures
4. Bulkhead — isolate B's failures from affecting other calls
5. Fallback — degrade gracefully when B is down
6. Observability — see the problem before users complain

**Q: Your database is the bottleneck. Walk me through your options.**
1. Add indexes (Day 17)
2. Add caching (Day 18)
3. Add read replicas + read/write split (Day 27)
4. Consider denormalisation
5. Shard (last resort — operational complexity)

---

## 7. Week 3 Preview — System Design Case Studies

Next week you'll apply everything you've learned to classic system design interview problems:

| Day | Case Study | Key Systems Used |
|-----|-----------|-----------------|
| 30 | URL Shortener | Hash generation, Redis, read-heavy scale |
| 31 | Social Media Feed | Fan-out, Kafka, denormalisation |
| 32 | Ride-Sharing Platform | Real-time location, WebSockets, geo-indexing |
| 33 | Payment System | Idempotency, distributed transactions, saga |
| 34 | Search Engine & Typeahead | Inverted index, trie, Elasticsearch |
| 35 | Video Streaming Platform | CDN, adaptive bitrate, blob storage |
| 36 | Phase 2 Capstone & Interview Prep | End-to-end design + interview practice |

---

## 8. Key Takeaways

1. **No single pattern is a silver bullet** — every solution in Week 2 adds operational complexity and must be justified by scale requirements.
2. **Observability must be built in from the start** — it's not a feature you add later.
3. **Design for failure, not just for success** — circuit breakers, retries, bulkheads, and fallbacks are not optional in production.
4. **The database is not a monolith** — primary/replica for reads, sharding for writes, CDN for static content.
5. **Physical proximity always wins** — edge caching eliminates latency that no amount of optimisation can overcome from a distant origin.

---

*Day 29. The sum of the parts is a distributed system. This week you learned to name the parts — next week you draw the whole.*
