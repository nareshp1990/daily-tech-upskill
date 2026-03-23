# Day 22 — Rate Limiting, Backpressure & Load Shedding
**Date:** 2026-03-29
**Phase:** 2 — System Design
**Time Target:** 1–2 hours

---

## 1. Today's Focus Topic
**Rate Limiting, Backpressure & Load Shedding — Protecting Services Under Load**

Yesterday you learned that distributed systems fail. Today you learn how to fail gracefully. Rate limiting controls how much traffic enters. Backpressure tells upstream to slow down. Load shedding drops excess work to protect the system. Together, these three patterns keep your services alive when traffic spikes, downstream services degrade, or resources run out.

---

## 2. Core Concepts

### 2.1 The Three Defences

```
          Incoming Traffic
               │
               ▼
    ┌─────────────────────┐
    │    Rate Limiting     │  "Only 1000 req/s per client"
    │   (at the gate)      │  Excess → 429 Too Many Requests
    └──────────┬──────────┘
               │
               ▼
    ┌─────────────────────┐
    │    Backpressure      │  "I'm slow, stop sending"
    │   (between services) │  Signal upstream to slow down
    └──────────┬──────────┘
               │
               ▼
    ┌─────────────────────┐
    │    Load Shedding     │  "I'm overloaded, dropping low-priority work"
    │   (last resort)      │  Protect core functionality
    └──────────┬──────────┘
               │
               ▼
         Service processes
         remaining requests
```

### 2.2 Rate Limiting — Deep Dive

#### Token Bucket Algorithm

```
Bucket capacity: 10 tokens
Refill rate: 2 tokens/second

Time 0:  [●●●●●●●●●●]  10 tokens — 5 requests arrive → 5 tokens left
Time 1:  [●●●●●●●]      7 tokens (5 + 2 refilled)
Time 2:  [●●●●●●●●●]    9 tokens — burst of 12 requests:
                          9 accepted, 3 rejected (429)
Time 3:  [●●]            2 tokens (0 + 2 refilled)
```

**Properties:** Allows short bursts (up to bucket size), smooths out over time. This is what most API gateways use.

#### Sliding Window Counter

```
Window: 1 minute, Limit: 100 requests

Previous window (40 requests)    Current window (70 requests)
|========================|======|==================|
                         ^      ^
                    70% of      30% of
                    prev window  current window

Weighted count = (40 × 0.70) + 70 = 98  → Under limit ✅
Next request:     (40 × 0.70) + 71 = 99  → Under limit ✅
Next request:     (40 × 0.70) + 72 = 100 → AT limit ⚠️
Next request:     Rejected → 429
```

#### Distributed Rate Limiting with Redis

```java
@Component
@RequiredArgsConstructor
public class RedisRateLimiter {

    private final StringRedisTemplate redis;

    // Sliding window counter using Redis sorted set
    public boolean isAllowed(String clientId, int maxRequests, Duration window) {
        String key = "ratelimit:" + clientId;
        long now = Instant.now().toEpochMilli();
        long windowStart = now - window.toMillis();

        // Atomic pipeline: remove old entries, add current, count
        return redis.execute(new SessionCallback<Boolean>() {
            @Override
            public Boolean execute(RedisOperations operations) {
                operations.multi();
                operations.opsForZSet().removeRangeByScore(key, 0, windowStart);
                operations.opsForZSet().add(key, String.valueOf(now), now);
                operations.opsForZSet().zCard(key);
                operations.expire(key, window.plusSeconds(10));
                List<Object> results = operations.exec();

                long count = (Long) results.get(2);
                return count <= maxRequests;
            }
        });
    }
}
```

#### Rate Limiting Dimensions

| Dimension | Example | Use Case |
|-----------|---------|----------|
| Per API key | 1000 req/min per key | SaaS API plans |
| Per user | 100 req/min per user | Prevent abuse |
| Per IP | 50 req/min per IP | Anonymous traffic |
| Per endpoint | 10 req/min on `/export` | Protect heavy endpoints |
| Global | 50,000 req/min total | Protect infrastructure |

### 2.3 Backpressure

Backpressure is a signal from a consumer to a producer: "Slow down, I can't keep up."

#### Where Backpressure Appears

| Layer | Mechanism |
|-------|-----------|
| **TCP** | Receive window shrinks → sender slows down |
| **Kafka** | Consumer lag grows → alert, auto-scale consumers |
| **Thread pool** | Queue fills → reject policy kicks in |
| **Reactive streams** | `Flux.onBackpressureDrop()` |
| **gRPC** | Flow control window |

#### Thread Pool Backpressure in Spring Boot

```java
@Configuration
public class AsyncConfig {

    @Bean
    public ThreadPoolTaskExecutor taskExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(10);
        executor.setMaxPoolSize(50);
        executor.setQueueCapacity(100);

        // Backpressure: when queue is full, reject new work
        executor.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
        // CallerRunsPolicy → calling thread does the work → natural slowdown
        // AbortPolicy → throws RejectedExecutionException → caller handles
        return executor;
    }
}
```

#### Kafka Consumer Backpressure

```yaml
spring:
  kafka:
    consumer:
      max-poll-records: 50          # Process 50 records per poll (smaller batches)
      fetch-max-wait-ms: 500
      max-poll-interval-ms: 300000  # If processing takes > 5min, consumer is kicked
```

**Monitor consumer lag** — if lag grows consistently, consumers can't keep up:
```bash
kafka-consumer-groups.sh --bootstrap-server localhost:9092 \
  --group payment-service --describe
```

### 2.4 Load Shedding

When the system is overloaded beyond what rate limiting and backpressure can handle, shed load to protect critical functionality.

#### Priority-Based Load Shedding

```java
@Component
@RequiredArgsConstructor
public class LoadSheddingFilter extends OncePerRequestFilter {

    private final LoadMonitor loadMonitor;

    @Override
    protected void doFilterInternal(HttpServletRequest request,
            HttpServletResponse response, FilterChain chain)
            throws ServletException, IOException {

        double cpuLoad = loadMonitor.getCpuUsage();
        RequestPriority priority = classifyRequest(request);

        if (cpuLoad > 0.9 && priority == RequestPriority.LOW) {
            // Shed low-priority requests when CPU > 90%
            response.setStatus(503);
            response.setHeader("Retry-After", "30");
            response.getWriter().write(
                "{\"error\":{\"code\":\"SERVICE_OVERLOADED\",\"message\":\"Try again later\"}}");
            return;
        }

        if (cpuLoad > 0.95 && priority != RequestPriority.CRITICAL) {
            // Only serve critical requests when CPU > 95%
            response.setStatus(503);
            response.setHeader("Retry-After", "60");
            return;
        }

        chain.doFilter(request, response);
    }

    private RequestPriority classifyRequest(HttpServletRequest request) {
        String path = request.getRequestURI();
        if (path.startsWith("/health")) return RequestPriority.CRITICAL;
        if (path.startsWith("/v1/payments")) return RequestPriority.HIGH;
        if (path.startsWith("/v1/orders")) return RequestPriority.MEDIUM;
        if (path.startsWith("/v1/recommendations")) return RequestPriority.LOW;
        return RequestPriority.MEDIUM;
    }
}
```

#### Graceful Degradation Strategies

| Strategy | How | Example |
|----------|-----|---------|
| **Feature flags** | Disable non-essential features | Turn off recommendations during peak |
| **Simplified responses** | Return cached/static data instead of computing | Return last-known product list |
| **Queue overflow** | Drop oldest or lowest-priority items | Event processing skips analytics events |
| **Connection limits** | Cap max concurrent connections | Tomcat `server.tomcat.max-connections` |
| **Timeout reduction** | Lower timeouts under load | 5s → 2s to free threads faster |

---

## 3. Combining All Three in Production

```
Internet → API Gateway (rate limit per key)
                │
                ▼
         Load Balancer
          │         │
          ▼         ▼
     ┌─────────┐ ┌─────────┐
     │ Pod 1   │ │ Pod 2   │
     │ ┌─────┐ │ │ ┌─────┐ │
     │ │Load │ │ │ │Load │ │  ← Load shedding (CPU-based)
     │ │Shed │ │ │ │Shed │ │
     │ └──┬──┘ │ │ └──┬──┘ │
     │    ▼    │ │    ▼    │
     │ ┌─────┐ │ │ ┌─────┐ │
     │ │Svc  │ │ │ │Svc  │ │  ← Thread pool backpressure
     │ │Logic│ │ │ │Logic│ │
     │ └──┬──┘ │ │ └──┬──┘ │
     └────┼────┘ └────┼────┘
          │           │
          ▼           ▼
      ┌───────────────────┐
      │   Kafka            │  ← Consumer lag = backpressure signal
      └────────┬──────────┘
               │
          ┌────┴────┐
          ▼         ▼
     ┌─────────┐ ┌─────────┐
     │Consumer │ │Consumer │  ← Auto-scale on lag
     │ Pod 1   │ │ Pod 2   │
     └─────────┘ └─────────┘
```

---

## 4. Hands-On Exercise

### Task: Build a resilient request pipeline

1. **Rate limiting:** Implement a Redis-based sliding window rate limiter (100 req/min per API key).
2. **Backpressure:** Configure a Spring `ThreadPoolTaskExecutor` with `CallerRunsPolicy` and queue capacity of 50.
3. **Load shedding:** Create a servlet filter that:
   - At 80% CPU: log a warning.
   - At 90% CPU: reject `/v1/recommendations` requests with 503.
   - At 95% CPU: reject everything except `/health` and `/v1/payments`.
4. **Test it:** Use a load testing tool (wrk, k6, or JMeter) to push the service to its limits and verify each defence layer activates.

### Load Test with k6:
```javascript
import http from 'k6/http';
import { check } from 'k6';

export const options = {
    stages: [
        { duration: '30s', target: 50 },    // ramp up
        { duration: '1m',  target: 200 },   // sustained load
        { duration: '30s', target: 500 },   // spike
        { duration: '30s', target: 0 },     // ramp down
    ],
};

export default function () {
    const res = http.get('http://localhost:8080/v1/orders', {
        headers: { 'X-API-Key': 'test-key-1' },
    });
    check(res, {
        'not rate limited': (r) => r.status !== 429,
        'not shed': (r) => r.status !== 503,
    });
}
```

---

## 5. Key Takeaways

1. **Rate limiting is the first line of defence** — applied at the API gateway, per client, before any business logic.
2. **Backpressure keeps the system honest** — if a consumer can't keep up, the producer must slow down or work queues up.
3. **Load shedding is a last resort** — when overwhelmed, protect critical paths by dropping non-essential work.
4. **All three work together** — rate limiting controls inflow, backpressure regulates internal flow, load shedding prevents collapse.
5. **503 with Retry-After is better than a crash** — a controlled rejection is always better than an uncontrolled failure.

---

## 6. Connection to Previous Days

Rate limiting appeared in Day 12 (LLM Security) and Day 20 (API Design). Backpressure connects to Day 19's Kafka consumer management. Load shedding is the generalisation of the fallback chain from Day 13 (Production LLM Architecture) — when the primary path is overloaded, shed load gracefully rather than letting everything fail.

---

## 7. Resources

- **[Google SRE Book, Ch. 22: Handling Overload](https://sre.google/sre-book/handling-overload/)** — How Google does load shedding at scale.
- **[Stripe's rate limiting blog](https://stripe.com/blog/rate-limiters)** — Practical production rate limiting.
- **[Bucket4j](https://github.com/bucket4j/bucket4j)** — Java rate limiting library with Redis support.

---

*Day 22. The system that survives isn't the fastest — it's the one that knows when to say no.*
