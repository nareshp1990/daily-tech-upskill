# Day 38 — API Gateway & Backend for Frontend (BFF)
**Date:** 2026-04-14
**Phase:** 3 — Microservices & Cloud-Native
**Time Target:** 1–2 hours

---

## 1. Today's Focus Topic
**API Gateway & BFF Pattern — The Entry Point to Your Microservices**

Yesterday you decomposed a monolith into microservices. Today you solve the resulting problem: clients now need to talk to many services. The API Gateway is the single entry point that centralises cross-cutting concerns (auth, rate limiting, routing, TLS termination). The BFF (Backend for Frontend) pattern takes it further — tailoring the API surface to each client type. Both are used together in every production microservices deployment you'll encounter.

---

## 2. The Problem API Gateway Solves

### Without a Gateway

```
Mobile App  → Order Service   (port 8081)
           → Catalog Service  (port 8082)
           → Customer Service (port 8083)
           → Payment Service  (port 8084)
           → Each with its own auth, TLS, rate limit logic
```

**Problems:**
- Client must know all service addresses — tight coupling
- Repeated auth logic in every service
- TLS termination duplicated everywhere
- No single place for rate limiting, logging, analytics
- Hard to change service topology without updating all clients
- CORS and other cross-cutting concerns scattered

### With a Gateway

```
Mobile App  → API Gateway (single endpoint)
                ├── Route /api/orders     → Order Service
                ├── Route /api/catalog    → Catalog Service
                ├── Route /api/customers  → Customer Service
                └── Route /api/payments  → Payment Service
                
Cross-cutting (done once in gateway):
- JWT validation
- Rate limiting
- TLS termination
- Request logging
- Tracing injection
- CORS headers
```

---

## 3. Spring Cloud Gateway

Spring's first-class gateway solution. Built on Spring WebFlux (non-blocking, reactive).

### 3.1 Dependencies

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-gateway</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
</dependency>
```

### 3.2 Route Configuration

```yaml
# application.yml — Gateway configuration
spring:
  application:
    name: api-gateway
  cloud:
    gateway:
      routes:
        # Order Service routes
        - id: order-service
          uri: lb://order-service          # lb:// = load-balanced via Eureka
          predicates:
            - Path=/api/orders/**
          filters:
            - StripPrefix=1                # Remove /api prefix before forwarding
            - name: CircuitBreaker
              args:
                name: orderCircuitBreaker
                fallbackUri: forward:/fallback/orders
            - name: RequestRateLimiter
              args:
                redis-rate-limiter.replenishRate: 100
                redis-rate-limiter.burstCapacity: 200
                key-resolver: "#{@userKeyResolver}"

        # Catalog Service routes
        - id: catalog-service
          uri: lb://catalog-service
          predicates:
            - Path=/api/catalog/**
          filters:
            - StripPrefix=1
            - AddResponseHeader=X-Powered-By, catalog-service

        # Auth endpoint — no auth required
        - id: auth-service
          uri: lb://auth-service
          predicates:
            - Path=/api/auth/**
          filters:
            - StripPrefix=1
            # No authentication filter — public endpoint

      default-filters:
        - DedupeResponseHeader=Access-Control-Allow-Origin Access-Control-Allow-Credentials
```

### 3.3 JWT Authentication Filter

```java
@Component
public class JwtAuthenticationFilter implements GlobalFilter, Ordered {

    @Autowired
    private JwtTokenValidator jwtTokenValidator;

    private static final List<String> PUBLIC_PATHS = List.of(
        "/api/auth/login",
        "/api/auth/register",
        "/api/catalog/products"  // Public browse, no auth needed
    );

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        String path = exchange.getRequest().getPath().value();

        // Skip auth for public endpoints
        if (PUBLIC_PATHS.stream().anyMatch(path::startsWith)) {
            return chain.filter(exchange);
        }

        String authHeader = exchange.getRequest().getHeaders().getFirst("Authorization");

        if (authHeader == null || !authHeader.startsWith("Bearer ")) {
            exchange.getResponse().setStatusCode(HttpStatus.UNAUTHORIZED);
            return exchange.getResponse().setComplete();
        }

        String token = authHeader.substring(7);

        return jwtTokenValidator.validate(token)
            .flatMap(claims -> {
                // Inject user context as headers for downstream services
                ServerHttpRequest mutatedRequest = exchange.getRequest().mutate()
                    .header("X-User-Id", claims.getSubject())
                    .header("X-User-Roles", String.join(",", claims.getRoles()))
                    .build();

                return chain.filter(exchange.mutate().request(mutatedRequest).build());
            })
            .onErrorResume(ex -> {
                exchange.getResponse().setStatusCode(HttpStatus.UNAUTHORIZED);
                return exchange.getResponse().setComplete();
            });
    }

    @Override
    public int getOrder() {
        return -1; // Run before other filters
    }
}
```

### 3.4 Rate Limiting with Redis

```java
@Configuration
public class RateLimiterConfig {

    // Rate limit per authenticated user
    @Bean
    public KeyResolver userKeyResolver() {
        return exchange -> {
            String userId = exchange.getRequest().getHeaders().getFirst("X-User-Id");
            return Mono.just(userId != null ? userId : "anonymous");
        };
    }

    // Rate limit per IP (for unauthenticated endpoints)
    @Bean
    public KeyResolver ipKeyResolver() {
        return exchange -> Mono.just(
            Objects.requireNonNull(exchange.getRequest().getRemoteAddress())
                .getAddress().getHostAddress()
        );
    }
}
```

### 3.5 Fallback Controller

```java
@RestController
@RequestMapping("/fallback")
public class FallbackController {

    @GetMapping("/orders")
    public ResponseEntity<Map<String, String>> ordersFallback() {
        return ResponseEntity.status(HttpStatus.SERVICE_UNAVAILABLE)
            .body(Map.of(
                "status", "error",
                "message", "Order service is temporarily unavailable. Please try again.",
                "retryAfter", "30"
            ));
    }
}
```

---

## 4. Backend for Frontend (BFF) Pattern

### The Problem BFF Solves

A single "universal" API forces all clients to request too much or too little data.

```
Single API Problem:
- Mobile (slow connection, small screen) gets 50 fields — needs 8
- Web dashboard (fast connection, complex UI) gets 50 fields — needs 35
- Smart TV app gets 50 fields — needs 12

Everyone either over-fetches or needs multiple calls to stitch data.
```

### BFF Solution — One Backend Per Client Type

```
Mobile App  → Mobile BFF    → Aggregates services, returns mobile-optimized payload
Web App     → Web BFF       → Aggregates services, returns rich web payload
Smart TV    → TV BFF        → Returns TV-optimized content (images, video links)
Partner API → Partner BFF   → Returns B2B-formatted data with partner-specific auth
```

### BFF Architecture

```
                    ┌──────────────────────────────────────────┐
                    │            BFF Layer                     │
                    │                                          │
Mobile App   ──────►│ Mobile BFF  ──┐                         │
                    │               │                         │
Web SPA      ──────►│ Web BFF     ──┼──► Catalog Service      │
                    │               │    Order Service         │
Smart TV     ──────►│ TV BFF      ──┘    Customer Service     │
                    │                    Payment Service       │
                    └──────────────────────────────────────────┘
```

### BFF Implementation — Mobile BFF Example

```java
@RestController
@RequestMapping("/mobile/v1")
public class MobileBffController {

    @Autowired private CatalogServiceClient catalogClient;
    @Autowired private OrderServiceClient orderClient;
    @Autowired private CustomerServiceClient customerClient;

    /**
     * Mobile home screen — aggregates from 3 services, returns lean payload.
     * Web BFF would return 3x more data for the same logical screen.
     */
    @GetMapping("/home")
    public MobileHomeResponse getHomeScreen(@RequestHeader("X-User-Id") String userId) {
        
        // Parallel calls to downstream services (non-blocking)
        CompletableFuture<List<ProductSummary>> featuredFuture = 
            CompletableFuture.supplyAsync(() -> 
                catalogClient.getFeaturedProducts(8));  // Mobile: max 8 items
        
        CompletableFuture<List<OrderSummary>> recentOrdersFuture = 
            CompletableFuture.supplyAsync(() -> 
                orderClient.getRecentOrders(userId, 3));  // Mobile: last 3 only
        
        CompletableFuture<CustomerBadge> customerFuture = 
            CompletableFuture.supplyAsync(() -> 
                customerClient.getBadge(userId));  // Mobile: just tier + points

        CompletableFuture.allOf(featuredFuture, recentOrdersFuture, customerFuture).join();

        return MobileHomeResponse.builder()
            .featured(featuredFuture.join().stream()
                .map(this::toMobileProduct)  // Strip fields not needed on mobile
                .toList())
            .recentOrders(recentOrdersFuture.join())
            .badge(customerFuture.join())
            .build();
    }

    // Mobile-optimized product: imageUrl, name, price only (no description, specs, reviews)
    private MobileProductCard toMobileProduct(ProductSummary p) {
        return new MobileProductCard(p.getId(), p.getImageThumbnailUrl(), 
                                     p.getName(), p.getPrice());
    }
}
```

### Web BFF Example

```java
@RestController
@RequestMapping("/web/v1")
public class WebBffController {

    @GetMapping("/home")
    public WebHomeResponse getHomeScreen(@RequestHeader("X-User-Id") String userId) {
        
        // Web fetches more data — richer UI with recommendations, full descriptions
        List<ProductDetail> featured = catalogClient.getFeaturedProducts(20);  // More items
        List<OrderDetail> recentOrders = orderClient.getRecentOrders(userId, 10);  // More history
        CustomerProfile profile = customerClient.getFullProfile(userId);  // Full profile
        List<ProductRecommendation> recommendations = 
            recommendationClient.getPersonalized(userId, 10);

        return WebHomeResponse.builder()
            .featured(featured)
            .recentOrders(recentOrders)
            .profile(profile)
            .recommendations(recommendations)
            .build();
    }
}
```

---

## 5. Gateway vs BFF — When to Use Which

| Concern | API Gateway | BFF |
|---------|------------|-----|
| **Auth / JWT validation** | Gateway | — |
| **Rate limiting** | Gateway | — |
| **TLS termination** | Gateway | — |
| **Request routing** | Gateway | — |
| **Response aggregation** | — | BFF |
| **Client-specific payload shaping** | — | BFF |
| **Protocol translation (REST→gRPC)** | Gateway | BFF |
| **A/B testing / feature flags** | Either | BFF preferred |

**Typical production topology:**

```
Client → API Gateway (TLS, Auth, Rate Limit, Routing)
              ↓
         BFF Layer (Aggregation, Payload shaping per client)
              ↓
     Downstream Microservices
```

---

## 6. Cross-Cutting Concerns Reference

### 6.1 CORS in Gateway

```yaml
spring:
  cloud:
    gateway:
      globalcors:
        corsConfigurations:
          '[/**]':
            allowedOrigins:
              - "https://app.example.com"
              - "https://admin.example.com"
            allowedMethods:
              - GET
              - POST
              - PUT
              - DELETE
              - OPTIONS
            allowedHeaders:
              - "*"
            allowCredentials: true
            maxAge: 3600
```

### 6.2 Request/Response Logging Filter

```java
@Component
public class RequestLoggingFilter implements GlobalFilter {

    private static final Logger log = LoggerFactory.getLogger(RequestLoggingFilter.class);

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        long startTime = System.currentTimeMillis();
        String requestId = UUID.randomUUID().toString();
        
        exchange.getRequest().mutate()
            .header("X-Request-Id", requestId)
            .build();

        log.info("Incoming request: method={} path={} requestId={}",
            exchange.getRequest().getMethod(),
            exchange.getRequest().getPath(),
            requestId);

        return chain.filter(exchange).then(Mono.fromRunnable(() -> {
            long duration = System.currentTimeMillis() - startTime;
            log.info("Request completed: requestId={} status={} duration={}ms",
                requestId,
                exchange.getResponse().getStatusCode(),
                duration);
        }));
    }
}
```

### 6.3 Distributed Tracing Propagation

```java
// Gateway automatically propagates Trace IDs via Micrometer Tracing
// Add to application.yml:
management:
  tracing:
    sampling:
      probability: 1.0
  zipkin:
    tracing:
      endpoint: http://zipkin:9411/api/v2/spans
```

---

## 7. Deployment Topology

```yaml
# Docker Compose — Gateway + Services
version: '3.8'
services:
  api-gateway:
    image: api-gateway:latest
    ports:
      - "8080:8080"       # Public-facing
    environment:
      - EUREKA_URI=http://eureka:8761/eureka
      - REDIS_HOST=redis
    depends_on:
      - eureka
      - redis

  mobile-bff:
    image: mobile-bff:latest
    ports:
      - "8090:8090"       # Internal only — gateway routes to it
    environment:
      - EUREKA_URI=http://eureka:8761/eureka

  order-service:
    image: order-service:latest
    expose:
      - "8081"            # No direct public access
    environment:
      - EUREKA_URI=http://eureka:8761/eureka
```

---

## 8. Common Interview Questions

**Q: What's the difference between API Gateway and Load Balancer?**
A: A Load Balancer distributes traffic at L4/L7 across identical instances of the *same* service. An API Gateway routes to *different* services based on path/method, and adds application-layer logic (auth, rate limiting, request transformation). In practice, they're often stacked: Load Balancer → API Gateway → Services.

**Q: When do you use BFF vs a single gateway?**
A: When different client types have meaningfully different data needs. If mobile needs 8 fields and web needs 35, a single API forces over-fetching or multiple calls. BFF gives each client team ownership of their API contract. If all clients need the same data, a single gateway is simpler.

**Q: How do you secure service-to-service calls behind the gateway?**
A: The gateway injects the verified user context as trusted headers (`X-User-Id`, `X-User-Roles`). Services trust these headers since traffic only arrives via the gateway (network policy enforces this in K8s). For service-to-service, use mTLS with Istio or a service mesh — the gateway handles external auth, the mesh handles internal auth.

**Q: What happens if the gateway is down?**
A: The gateway is a critical path — it must be HA. Deploy at least 3 replicas, use health checks, and put a hardware/cloud load balancer in front. In K8s, use HPA and PodDisruptionBudgets. Circuit breaker at the gateway level prevents cascade failures to upstream services.

---

## 9. Today's Practice Exercise

Design the API Gateway + BFF layer for the **hotel booking platform** from yesterday:

1. Define gateway routes for: Search, Booking, Customer, Payment, Notifications
2. Which endpoints are public (no auth)? Which are authenticated?
3. Design the **Mobile BFF** response for the booking confirmation screen — what fields from Booking + Customer + Payment services would you aggregate?
4. How would you handle a slow Payment service response in the gateway? (Circuit breaker fallback)

---

## 10. Key Takeaways

- **API Gateway** = single entry point for all clients — centralises TLS, auth, rate limiting, routing
- **BFF** = per-client-type API layer — aggregates and shapes responses for mobile vs web vs TV
- **Spring Cloud Gateway** is the Spring-native solution — reactive, filter chain, Eureka-integrated
- **JWT auth** in the gateway: validate once, inject user context headers for downstream
- **Rate limiting** belongs in the gateway — per-user or per-IP, backed by Redis
- **BFF benefits:** each client team owns their contract, reduces over-fetching, enables independent evolution
- Always design the gateway as **HA** — it's the single point of entry

---

## 11. Resources

- [Spring Cloud Gateway Docs](https://docs.spring.io/spring-cloud-gateway/docs/current/reference/html/)
- [Sam Newman — BFF Pattern](https://samnewman.io/patterns/architectural/bff/)
- [Martin Fowler — BFF](https://martinfowler.com/articles/gateway-pattern.html)

---

*Next: Day 39 — Service Mesh with Istio (mTLS, Traffic Policies, Observability)*
