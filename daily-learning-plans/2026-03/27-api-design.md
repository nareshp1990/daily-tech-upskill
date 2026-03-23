# Day 20 — API Design
**Date:** 2026-03-27
**Phase:** 2 — System Design
**Time Target:** 1–2 hours

---

## 1. Today's Focus Topic
**API Design — REST Best Practices, Versioning, Rate Limiting, and Idempotency**

APIs are the contracts between systems. A well-designed API is easy to use correctly and hard to use incorrectly. Today you'll go beyond basic CRUD endpoints and learn the patterns that make APIs production-grade: versioning strategies, pagination, idempotency for safe retries, and rate limiting to protect your service.

---

## 2. Core Concepts

### 2.1 REST API Design Principles

**Resource-Oriented Design:**
```
Good (noun-based resources):          Bad (verb-based RPC-style):
GET    /orders                         GET    /getOrders
GET    /orders/{id}                    GET    /getOrderById?id=123
POST   /orders                         POST   /createOrder
PUT    /orders/{id}                    POST   /updateOrder
DELETE /orders/{id}                    POST   /deleteOrder
PATCH  /orders/{id}                    POST   /partialUpdateOrder

GET    /orders/{id}/items              GET    /getOrderItems?orderId=123
POST   /orders/{id}/items              POST   /addItemToOrder
```

**HTTP Methods and Their Semantics:**

| Method | Idempotent | Safe | Use Case |
|--------|-----------|------|----------|
| GET | Yes | Yes | Read resource |
| PUT | Yes | No | Full replace |
| DELETE | Yes | No | Remove resource |
| PATCH | No* | No | Partial update |
| POST | No | No | Create resource, trigger action |

*PATCH can be made idempotent with JSON Merge Patch.

### 2.2 Response Design

**Consistent envelope:**
```json
// Success
{
  "data": { "id": "ord-123", "status": "PENDING", "total": 59.98 },
  "meta": { "requestId": "req-abc-456" }
}

// Error
{
  "error": {
    "code": "ORDER_NOT_FOUND",
    "message": "Order ord-999 does not exist",
    "details": []
  },
  "meta": { "requestId": "req-abc-457" }
}
```

**HTTP Status Codes — Use Them Correctly:**

| Code | When |
|------|------|
| 200 | Success (GET, PUT, PATCH) |
| 201 | Created (POST that creates a resource) |
| 204 | No Content (DELETE, or PUT with no response body) |
| 400 | Bad request (validation failure, malformed JSON) |
| 401 | Unauthenticated (no/invalid token) |
| 403 | Forbidden (valid token, insufficient permissions) |
| 404 | Resource not found |
| 409 | Conflict (duplicate, version mismatch) |
| 422 | Unprocessable Entity (semantically invalid) |
| 429 | Too Many Requests (rate limited) |
| 500 | Internal Server Error (unexpected failure) |
| 503 | Service Unavailable (overloaded, maintenance) |

### 2.3 Pagination

**Offset-based (simple, common):**
```
GET /orders?page=2&size=20

Response:
{
  "data": [...],
  "pagination": {
    "page": 2,
    "size": 20,
    "totalElements": 156,
    "totalPages": 8
  }
}
```

**Cursor-based (scalable, consistent):**
```
GET /orders?cursor=eyJpZCI6MTAwfQ&size=20

Response:
{
  "data": [...],
  "pagination": {
    "nextCursor": "eyJpZCI6MTIwfQ",
    "hasMore": true
  }
}
```

| Feature | Offset | Cursor |
|---------|--------|--------|
| Deep pages | Slow (OFFSET 10000) | Fast (WHERE id > cursor) |
| Consistent during writes | No (items shift) | Yes |
| Random page access | Yes (jump to page 5) | No (sequential only) |
| Implementation | Simple | Moderate |

### 2.4 API Versioning

| Strategy | Example | Pros | Cons |
|----------|---------|------|------|
| URL path | `/v1/orders` | Explicit, easy to route | URL pollution |
| Header | `Accept: application/vnd.api.v2+json` | Clean URLs | Hidden, harder to test |
| Query param | `/orders?version=2` | Easy to add | Clutters params |

**Recommendation:** URL path versioning (`/v1/`) is the most pragmatic. It's explicit, works with every HTTP client, and is easy to route in Nginx/API gateway.

```java
@RestController
@RequestMapping("/v1/orders")
public class OrderControllerV1 {
    // ...
}

@RestController
@RequestMapping("/v2/orders")
public class OrderControllerV2 {
    // V2 returns a different response shape
}
```

### 2.5 Idempotency

The same request sent twice should produce the same result. Critical for payment APIs and any operation where retries happen (network timeouts, mobile apps).

```
Client                              Server
  │                                    │
  │── POST /payments ──────────────→  │  Process payment
  │   Idempotency-Key: pay-abc-123    │  Store result with key
  │                                    │
  │←── 201 Created ────────────────  │
  │                                    │
  │── POST /payments ──────────────→  │  Lookup key → already processed
  │   Idempotency-Key: pay-abc-123    │  Return stored result
  │                                    │
  │←── 201 Created (same response) ─  │  No duplicate charge
```

**Spring Boot implementation:**

```java
@RestController
@RequestMapping("/v1/payments")
@RequiredArgsConstructor
public class PaymentController {

    private final PaymentService paymentService;
    private final IdempotencyStore idempotencyStore;

    @PostMapping
    public ResponseEntity<PaymentResponse> createPayment(
            @RequestHeader("Idempotency-Key") String idempotencyKey,
            @RequestBody @Valid PaymentRequest request) {

        // Check for existing result
        return idempotencyStore.get(idempotencyKey)
            .map(cached -> ResponseEntity.status(cached.statusCode()).body(cached.body()))
            .orElseGet(() -> {
                PaymentResponse response = paymentService.processPayment(request);
                idempotencyStore.store(idempotencyKey, 201, response, Duration.ofHours(24));
                return ResponseEntity.status(201).body(response);
            });
    }
}

@Component
@RequiredArgsConstructor
public class RedisIdempotencyStore implements IdempotencyStore {

    private final RedisTemplate<String, IdempotencyRecord> redis;

    public Optional<IdempotencyRecord> get(String key) {
        return Optional.ofNullable(redis.opsForValue().get("idempotency:" + key));
    }

    public void store(String key, int status, Object body, Duration ttl) {
        redis.opsForValue().set("idempotency:" + key,
            new IdempotencyRecord(status, body), ttl);
    }
}
```

### 2.6 Rate Limiting

**Algorithms:**

| Algorithm | How | Pros | Cons |
|-----------|-----|------|------|
| Fixed Window | Count requests per time window (e.g., 100/min) | Simple | Burst at window boundaries |
| Sliding Window Log | Track timestamps of each request | Accurate | Memory-intensive |
| Sliding Window Counter | Weighted average of current + previous window | Good balance | Approximate |
| Token Bucket | Tokens added at fixed rate, each request consumes one | Allows bursts up to bucket size | Slightly complex |
| Leaky Bucket | Requests queue and process at fixed rate | Smooth output | Delays bursty traffic |

**Spring Boot rate limiting with Bucket4j:**

```java
@Component
public class RateLimitFilter extends OncePerRequestFilter {

    private final Map<String, Bucket> buckets = new ConcurrentHashMap<>();

    @Override
    protected void doFilterInternal(HttpServletRequest request,
            HttpServletResponse response, FilterChain chain)
            throws ServletException, IOException {

        String clientId = extractClientId(request);
        Bucket bucket = buckets.computeIfAbsent(clientId, this::createBucket);

        ConsumptionProbe probe = bucket.tryConsumeAndReturnRemaining(1);
        if (probe.isConsumed()) {
            response.setHeader("X-Rate-Limit-Remaining",
                String.valueOf(probe.getRemainingTokens()));
            chain.doFilter(request, response);
        } else {
            response.setStatus(429);
            response.setHeader("Retry-After",
                String.valueOf(probe.getNanosToWaitForRefill() / 1_000_000_000));
            response.getWriter().write("{\"error\":{\"code\":\"RATE_LIMITED\"}}");
        }
    }

    private Bucket createBucket(String clientId) {
        return Bucket.builder()
            .addLimit(Bandwidth.classic(100, Refill.intervally(100, Duration.ofMinutes(1))))
            .build();
    }
}
```

---

## 3. API Security Checklist

| Concern | Solution |
|---------|----------|
| Authentication | OAuth 2.0 / JWT Bearer tokens |
| Authorization | RBAC or ABAC at endpoint level |
| Input validation | Bean Validation (`@Valid`, `@NotNull`, `@Size`) |
| Rate limiting | Per-client token bucket |
| CORS | Whitelist allowed origins |
| TLS | HTTPS everywhere, HSTS header |
| Request size | `spring.servlet.multipart.max-file-size` |
| Sensitive data | Never return passwords, tokens, or internal IDs in responses |

---

## 4. Hands-On Exercise

### Task: Design a production-grade Order API

1. **Define the resource model:** Order, OrderItem, Payment.
2. **Design 8 endpoints** with proper HTTP methods, status codes, and response shapes.
3. **Implement cursor-based pagination** for `GET /v1/orders`.
4. **Add idempotency** to `POST /v1/payments` with Redis-backed store.
5. **Add rate limiting** — 100 requests/minute per API key.
6. **Document with OpenAPI/Swagger** using `springdoc-openapi`.

### Expected Endpoint Design:
```
GET    /v1/orders?cursor={cursor}&size=20     → 200 + paginated list
POST   /v1/orders                              → 201 + Location header
GET    /v1/orders/{id}                         → 200 or 404
PUT    /v1/orders/{id}                         → 200 or 404
DELETE /v1/orders/{id}                         → 204 or 404
GET    /v1/orders/{id}/items                   → 200 + list
POST   /v1/payments (Idempotency-Key header)   → 201 or 200 (replay)
GET    /v1/payments/{id}                       → 200 or 404
```

---

## 5. Key Takeaways

1. **Design resources, not procedures** — nouns in URLs, verbs in HTTP methods.
2. **Idempotency keys prevent duplicate side effects** — essential for payments and anything with retries.
3. **Cursor pagination scales better than offset** — use it for any list that could grow large.
4. **Rate limiting protects your service and your clients** — return `429` with `Retry-After`.
5. **Version your API from day one** — `/v1/` costs nothing upfront and saves painful migrations later.

---

## 6. Connection to Previous Days

Rate limiting appeared in Phase 1 Day 12 (LLM Security) to protect LLM endpoints from abuse. Idempotency connects to Day 19's event-driven patterns — idempotent consumers handle duplicate events the same way idempotent APIs handle duplicate requests. The cursor-based pagination pattern uses the same indexed-scan approach from Day 17's database indexing.

---

## 7. Resources

- **[Microsoft REST API Guidelines](https://github.com/microsoft/api-guidelines)** — Comprehensive API design standards.
- **[Stripe API — Idempotent Requests](https://docs.stripe.com/api/idempotent_requests)** — Gold standard implementation reference.
- **[Spring Boot + springdoc-openapi](https://springdoc.org/)** — Auto-generate OpenAPI docs from Spring MVC.

---

*Day 20. A good API is one you don't need documentation to guess.*
