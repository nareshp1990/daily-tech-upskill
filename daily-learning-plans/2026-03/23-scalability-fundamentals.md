# Day 16 — Scalability Fundamentals
**Date:** 2026-03-23
**Phase:** 2 — System Design
**Time Target:** 1–2 hours

---

## 1. Today's Focus Topic
**Scalability Fundamentals — Horizontal Scaling, Load Balancing, Stateless Design**

Scalability is the ability of a system to handle growing workloads by adding resources. As a senior backend engineer you've scaled Spring Boot services behind load balancers — today you formalise the mental models behind those decisions. You'll learn to distinguish vertical from horizontal scaling, understand stateless design as the prerequisite for horizontal scaling, and see how load balancers distribute traffic across instances.

---

## 2. Core Concepts

### 2.1 Vertical vs Horizontal Scaling

| Dimension | Vertical (Scale Up) | Horizontal (Scale Out) |
|-----------|---------------------|----------------------|
| What changes | Bigger machine (CPU, RAM) | More machines |
| Ceiling | Hardware limits | Near-unlimited |
| Downtime | Often requires restart | Zero-downtime rolling deploys |
| Cost curve | Exponential (bigger boxes cost disproportionately more) | Linear |
| Complexity | Low (single node) | Higher (distributed state, networking) |

**Rule of thumb:** Start vertical for simplicity; move horizontal when you hit a single-node ceiling or need fault tolerance.

### 2.2 Stateless vs Stateful Services

```
Stateless Service                    Stateful Service
┌──────────────┐                    ┌──────────────┐
│  Request in  │                    │  Request in  │
│  Response out│                    │  Reads/writes │
│  No local    │                    │  local state  │
│  state       │                    │  (session,    │
└──────────────┘                    │   cache, file)│
                                    └──────────────┘
Any instance can serve              Requests must route to
any request                         the correct instance (sticky sessions)
```

**Why stateless wins for horizontal scaling:**
- Any instance handles any request — load balancer has full freedom.
- Instances are disposable — Kubernetes can kill/replace them freely.
- Auto-scaling is trivial — add pods, they start serving immediately.

**Common state you need to externalise:**
| State Type | Where to Move It |
|------------|-----------------|
| HTTP session | Redis (Spring Session) or JWT tokens |
| Cache | Redis / Memcached |
| File uploads | Object storage (Azure Blob, S3) |
| Scheduled jobs | Distributed scheduler (ShedLock, Quartz JDBC) |

### 2.3 Load Balancing

**Layer 4 (Transport) vs Layer 7 (Application):**

| Feature | L4 (TCP/UDP) | L7 (HTTP) |
|---------|-------------|-----------|
| Decision based on | IP + port | URL path, headers, cookies |
| Speed | Faster (no payload inspection) | Slightly slower |
| Use case | Database proxies, gRPC | REST APIs, path-based routing |
| Example | Azure Load Balancer, HAProxy TCP | Nginx, Envoy, AWS ALB |

**Load Balancing Algorithms:**

```
Round Robin          → Equal distribution, simplest
Weighted Round Robin → Some instances get more (bigger boxes)
Least Connections    → Route to least-busy instance
IP Hash              → Same client always hits same server (pseudo-sticky)
Random               → Surprisingly effective with many instances
```

**Health checks** ensure traffic only goes to healthy instances:
```yaml
# Kubernetes liveness + readiness probes
livenessProbe:
  httpGet:
    path: /actuator/health/liveness
    port: 8080
  initialDelaySeconds: 30
  periodSeconds: 10

readinessProbe:
  httpGet:
    path: /actuator/health/readiness
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 5
```

### 2.4 Scaling a Spring Boot Service — Before & After

**Before (stateful, single instance):**
```java
@RestController
public class OrderController {

    // BAD: in-memory session state
    private final Map<String, Cart> carts = new ConcurrentHashMap<>();

    @PostMapping("/cart/{userId}/add")
    public Cart addToCart(@PathVariable String userId, @RequestBody Item item) {
        carts.computeIfAbsent(userId, k -> new Cart()).addItem(item);
        return carts.get(userId);
    }
}
```

**After (stateless, horizontally scalable):**
```java
@RestController
@RequiredArgsConstructor
public class OrderController {

    private final RedisTemplate<String, Cart> redisTemplate;

    @PostMapping("/cart/{userId}/add")
    public Cart addToCart(@PathVariable String userId, @RequestBody Item item) {
        String key = "cart:" + userId;
        Cart cart = redisTemplate.opsForValue().get(key);
        if (cart == null) cart = new Cart();
        cart.addItem(item);
        redisTemplate.opsForValue().set(key, cart, Duration.ofHours(2));
        return cart;
    }
}
```

Now any pod can handle any cart request — scaling from 1 to 50 instances requires zero code changes.

---

## 3. Scaling Patterns

### 3.1 The Scale Cube (AKF Scale Cube)

```
                    ┌─────────────────────────┐
                   /                         /│
                  /    Z-axis: Data          / │
                 /     Partitioning         /  │
                /     (sharding by          /   │
               /       customer/region)    /    │
              ┌─────────────────────────┐      │
              │                         │      │
    Y-axis:   │                         │      /
    Functional│                         │     /
    Decomp    │                         │    /  X-axis:
    (micro-   │                         │   /   Horizontal
    services) │                         │  /    Cloning
              │                         │ /     (replicas)
              └─────────────────────────┘
```

| Axis | What | Example |
|------|------|---------|
| X | Clone the service, put a load balancer in front | 10 replicas of order-service |
| Y | Split by function into separate services | order-service, payment-service, inventory-service |
| Z | Split by data partition/shard | US orders on cluster A, EU orders on cluster B |

### 3.2 Auto-Scaling in Kubernetes

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: order-service-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: order-service
  minReplicas: 2
  maxReplicas: 20
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 60
    scaleDown:
      stabilizationWindowSeconds: 300  # slow scale-down to avoid flapping
```

---

## 4. Hands-On Exercise

### Task: Audit a Spring Boot service for horizontal scalability

1. **Pick a service** you've worked on (or imagine one).
2. **Identify stateful dependencies** — go through this checklist:
   - [ ] HTTP sessions stored in memory?
   - [ ] In-memory caches (Caffeine, ConcurrentHashMap)?
   - [ ] Local file reads/writes?
   - [ ] Scheduled tasks (only one instance should run)?
   - [ ] WebSocket connections (sticky sessions needed)?
3. **For each**, write the externalisation strategy:
   - Session → Spring Session + Redis
   - Cache → Redis / Spring Cache with Redis provider
   - Files → Azure Blob Storage
   - Scheduler → ShedLock with JDBC
   - WebSocket → Redis Pub/Sub for cross-instance broadcast
4. **Sketch the deployment** on paper:
   - 2+ pods behind a load balancer
   - External Redis for shared state
   - HPA configured on CPU/memory

### Stretch Goal
Write a `docker-compose.yml` that runs 3 instances of a Spring Boot app behind Nginx round-robin to see load distribution in action.

---

## 5. Key Takeaways

1. **Stateless design is the foundation** — you cannot scale horizontally if instances hold state.
2. **Externalise everything** — sessions to Redis, files to blob storage, locks to a distributed coordinator.
3. **Load balancers need health checks** — an unhealthy instance behind a load balancer is worse than fewer healthy instances.
4. **Auto-scaling needs stable metrics** — scale on CPU/memory for compute-bound, on queue depth for I/O-bound.
5. **The AKF Scale Cube gives three dimensions** — clone (X), decompose (Y), partition (Z) — use them in combination.

---

## 6. Connection to Phase 1

In Phase 1, you scaled LLM workloads with model routing (Day 13) and parallel agent fan-out (Day 11). The same principles apply: stateless agents are easy to parallelise, model routing is a form of Y-axis decomposition, and tenant-based prompt caches are Z-axis partitioning.

---

## 7. Resources

- **[The Art of Scalability (AKF Scale Cube)](https://akfpartners.com/growth-blog/scale-cube)** — Original framework for scaling dimensions.
- **[Spring Session + Redis](https://docs.spring.io/spring-session/reference/guides/boot-redis.html)** — Official guide for externalising sessions.
- **[Kubernetes HPA](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/)** — Auto-scaling reference.

---

*Phase 2, Day 1. The fundamentals that make everything else possible.*
