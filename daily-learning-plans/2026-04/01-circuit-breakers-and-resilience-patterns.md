# Day 25 — Circuit Breakers & Resilience Patterns
**Date:** 2026-04-01
**Phase:** 2 — System Design
**Time Target:** 1–2 hours

---

## 1. Today's Focus Topic
**Circuit Breakers & Resilience Patterns — Failing Fast and Recovering Gracefully**

You know that distributed services fail. Yesterday you learned how to find them. Today you learn how to protect your service when they go down — and how to recover automatically. A circuit breaker is the electrical analogy: when current (errors) exceeds a threshold, it trips open and stops the flow before it causes damage. Combined with retries, timeouts, bulkheads, and fallbacks, you build services that degrade gracefully rather than cascade-failing.

---

## 2. Core Concepts

### 2.1 The Cascading Failure Problem

```
Without circuit breakers:
  Thread pool: 200 threads

  payment-service goes down → every request to it hangs for 30s (timeout)
  200 threads × 30s = 200 requests queued and hanging
  order-service thread pool exhausted
  order-service becomes unresponsive
  API gateway calls order-service → it hangs
  API gateway thread pool exhausts
  All services dead — one downstream failure killed everything ☠️

With circuit breaker:
  After 5 failures in 10s → circuit OPENS
  Subsequent calls fail immediately (no waiting)
  Thread pool freed for other work
  Fallback response returned to clients
  Circuit probes recovery after 30s → CLOSED again ✅
```

### 2.2 Circuit Breaker States

```
    Threshold         Probe
    exceeded          succeeds
CLOSED ──────────► OPEN ──────────► HALF-OPEN
  ▲                                    │
  │                                    │
  └────────────────────────────────────┘
         All probes succeed → CLOSED

CLOSED:    Normal operation. Count failures. If failure rate > threshold → OPEN.
OPEN:      Fail fast. Don't call downstream. Return fallback immediately.
HALF-OPEN: Send a limited number of test requests. If they succeed → CLOSED.
                                                   If they fail → OPEN again.
```

### 2.3 Resilience4j — The Java Standard

Resilience4j replaces Netflix Hystrix (deprecated). It's functional, modular, and native to Spring Boot.

#### Dependency
```xml
<dependency>
  <groupId>io.github.resilience4j</groupId>
  <artifactId>resilience4j-spring-boot3</artifactId>
</dependency>
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-aop</artifactId>
</dependency>
```

#### Configuration
```yaml
resilience4j:
  circuitbreaker:
    instances:
      payment-service:
        slidingWindowType: COUNT_BASED        # or TIME_BASED
        slidingWindowSize: 10                 # Last 10 calls
        failureRateThreshold: 50              # Open if >50% fail
        slowCallRateThreshold: 80             # Also open if >80% are slow
        slowCallDurationThreshold: 2000ms     # Calls >2s count as slow
        waitDurationInOpenState: 30s          # Wait before HALF-OPEN probe
        permittedNumberOfCallsInHalfOpenState: 3
        minimumNumberOfCalls: 5               # Min calls before evaluating
        recordExceptions:
          - java.io.IOException
          - org.springframework.web.client.HttpServerErrorException
        ignoreExceptions:
          - com.example.BusinessValidationException  # Don't count as failure
```

#### Usage
```java
@Service
@RequiredArgsConstructor
public class PaymentClient {

    private final RestTemplate restTemplate;

    @CircuitBreaker(name = "payment-service", fallbackMethod = "fallbackPayment")
    @Retry(name = "payment-service")
    @TimeLimiter(name = "payment-service")
    public CompletableFuture<PaymentResponse> processPayment(PaymentRequest req) {
        return CompletableFuture.supplyAsync(() ->
            restTemplate.postForObject(
                "http://payment-service/v1/payments", req, PaymentResponse.class));
    }

    // Fallback — same signature + Throwable param
    public CompletableFuture<PaymentResponse> fallbackPayment(
            PaymentRequest req, Throwable t) {
        log.warn("Circuit open for payment-service: {}", t.getMessage());
        // Queue for async processing, return pending status
        return CompletableFuture.completedFuture(
            PaymentResponse.pending("Payment queued for retry — ref: " + UUID.randomUUID()));
    }
}
```

---

## 3. Resilience Patterns

### 3.1 Retry with Exponential Backoff

```yaml
resilience4j:
  retry:
    instances:
      payment-service:
        maxAttempts: 3
        waitDuration: 500ms
        enableExponentialBackoff: true
        exponentialBackoffMultiplier: 2     # 500ms, 1000ms, 2000ms
        retryExceptions:
          - java.io.IOException
          - org.springframework.web.client.HttpServerErrorException$ServiceUnavailable
        ignoreExceptions:
          - org.springframework.web.client.HttpClientErrorException$BadRequest
```

**Retry + Jitter (avoid thundering herd):**
```java
RetryConfig config = RetryConfig.custom()
    .maxAttempts(3)
    .intervalFunction(IntervalFunction.ofExponentialRandomBackoff(
        Duration.ofMillis(500),   // initial interval
        2.0,                       // multiplier
        Duration.ofSeconds(5)      // max interval
    ))
    .build();
```

### 3.2 Timeout (TimeLimiter)

```yaml
resilience4j:
  timelimiter:
    instances:
      payment-service:
        timeoutDuration: 3s
        cancelRunningFuture: true
```

### 3.3 Bulkhead — Isolate Failure Domains

```
Without bulkhead:
  200 shared threads → payment slow → all threads consumed → orders, users all slow

With bulkhead:
  payment threads: 20  (isolated pool)
  order threads:   50
  user threads:    30
  Payment failure saturates only 20 threads — other services unaffected
```

```yaml
resilience4j:
  bulkhead:
    instances:
      payment-service:
        maxConcurrentCalls: 20
        maxWaitDuration: 100ms   # How long to wait if all 20 are busy
```

### 3.4 Rate Limiter (built into Resilience4j)

```yaml
resilience4j:
  ratelimiter:
    instances:
      payment-service:
        limitForPeriod: 100
        limitRefreshPeriod: 1s
        timeoutDuration: 100ms
```

### 3.5 Pattern Stacking Order

```
Request arrives
    │
    ▼
[RateLimiter] → throttle early
    │
    ▼
[Bulkhead] → isolate concurrency
    │
    ▼
[TimeLimiter] → timeout wrapper
    │
    ▼
[CircuitBreaker] → fail fast if open
    │
    ▼
[Retry] → retry on transient failure
    │
    ▼
Actual HTTP call
```

---

## 4. Observability Integration

```yaml
management:
  health:
    circuitbreakers:
      enabled: true    # Expose CB state in /actuator/health

# Metrics exposed (Micrometer → Prometheus):
# resilience4j_circuitbreaker_state{name="payment-service"}
# resilience4j_circuitbreaker_failure_rate{name="payment-service"}
# resilience4j_retry_calls_total{name="payment-service", kind="successful_without_retry"}
```

```java
// React to state transitions (for alerting/logging)
@Component
@RequiredArgsConstructor
public class CircuitBreakerEventListener {

    @EventListener
    public void onStateTransition(CircuitBreakerOnStateTransitionEvent event) {
        log.warn("Circuit breaker '{}' transitioned from {} to {}",
            event.getCircuitBreakerName(),
            event.getStateTransition().getFromState(),
            event.getStateTransition().getToState());
        // Alert on-call if transitioning to OPEN
        if (event.getStateTransition().getToState() == CircuitBreaker.State.OPEN) {
            alertingService.fire("Circuit open: " + event.getCircuitBreakerName());
        }
    }
}
```

---

## 5. Hands-On Exercise

1. Add Resilience4j to a Spring Boot project.
2. Create a `payment-service` stub that fails 60% of the time.
3. Configure a circuit breaker with:
   - Sliding window size: 5
   - Failure threshold: 50%
   - Open duration: 10 seconds
4. Send 20 requests and observe the circuit opening.
5. Wait 10 seconds and observe HALF-OPEN probe and recovery.
6. Add a retry (max 2 retries) and see total attempts per request.
7. Check `/actuator/health` to see circuit state.

---

## 6. Key Takeaways

1. **Circuit breakers prevent cascading failures** — one slow service doesn't kill everything.
2. **Fail fast is better than slow failure** — return a degraded response immediately rather than waiting for a 30s timeout.
3. **Bulkheads isolate blast radius** — dedicate thread budgets per downstream service.
4. **Retry only on idempotent operations** — never retry non-idempotent calls (payments!) without idempotency keys.
5. **All four work together** — Timeout + CircuitBreaker + Retry + Bulkhead = production-ready resilience.

---

## 7. Connection to Previous Days

Rate limiting (Day 22) controls how much traffic enters your service. Today's patterns protect your service from what happens to the services you call. Load shedding (Day 22) and circuit breakers are complementary: one protects you from being overwhelmed by requests, the other from being dragged down by failing dependencies.

---

## 8. Resources

- **[Resilience4j User Guide](https://resilience4j.readme.io/docs)** — Comprehensive official docs.
- **[Netflix Tech Blog — Hystrix](https://netflixtechblog.com/making-the-netflix-api-more-resilient-a8ec62159c2d)** — The origin story.
- **[Release It! book by Michael Nygard](https://pragprog.com/titles/mnee2/release-it-second-edition/)** — Where circuit breakers for software were popularised.

---

*Day 25. The circuit breaker doesn't fix the problem — it stops it from becoming everyone's problem.*
