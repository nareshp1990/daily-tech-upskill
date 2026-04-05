# Day 24 — Service Discovery & Load Balancing
**Date:** 2026-03-31
**Phase:** 2 — System Design
**Time Target:** 1–2 hours

---

## 1. Today's Focus Topic
**Service Discovery & Load Balancing — How Microservices Find and Talk to Each Other**

In a monolith, method calls are resolved at compile time. In a microservices architecture with auto-scaling pods, service addresses change dynamically — containers start, stop, and move. Service discovery solves: "where is Service B right now?" Load balancing solves: "which instance of Service B should I send this request to?" Together they are the plumbing that makes dynamic, cloud-native architectures work.

---

## 2. Core Concepts

### 2.1 The Problem

```
Static configuration (broken):
  order-service.properties:
    payment.url=http://10.0.1.42:8080   ← What if this pod restarts?

Dynamic configuration (needed):
  Order Service asks: "Where are all healthy instances of payment-service?"
  Gets back: ["10.0.1.42:8080", "10.0.1.55:8080", "10.0.2.10:8080"]
  Picks one using load balancing logic.
```

### 2.2 Server-Side vs Client-Side Discovery

#### Server-Side Discovery
```
Client → Load Balancer (AWS ALB, Nginx, HAProxy)
              │
         Service Registry
              │
    ┌─────────┼─────────┐
    ▼         ▼         ▼
  Inst 1   Inst 2   Inst 3

Client is unaware of registry — LB handles discovery.
AWS ECS + ALB, Google Cloud Run, Kubernetes Services work this way.
```

#### Client-Side Discovery (Spring Cloud)
```
Client → reads registry → picks instance → calls directly

Client → Eureka/Consul
              │ returns [Inst1, Inst2, Inst3]
              ▼
Client applies load-balancing algorithm → calls Inst2 directly

Used by: Spring Cloud LoadBalancer, Netflix Ribbon (deprecated)
```

### 2.3 Service Registry Options

| Registry | CAP | Use Case |
|----------|-----|----------|
| **Eureka** (Netflix) | AP (available over consistent) | Spring Boot ecosystem, tolerates split-brain |
| **Consul** | CP | Multi-DC, health checks, K/V store, service mesh |
| **etcd** | CP | Kubernetes itself uses this internally |
| **Zookeeper** | CP | Older; Kafka still uses it (or KRaft now) |
| **Kubernetes DNS** | AP | Native K8s — `payment-service.default.svc.cluster.local` |

### 2.4 Spring Cloud Service Discovery Setup

**Eureka Server:**
```java
@SpringBootApplication
@EnableEurekaServer
public class ServiceRegistryApplication { }
```

```yaml
# eureka-server application.yml
server:
  port: 8761
eureka:
  client:
    register-with-eureka: false
    fetch-registry: false
```

**Microservice registration (client):**
```xml
<dependency>
  <groupId>org.springframework.cloud</groupId>
  <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
</dependency>
```

```yaml
# payment-service application.yml
spring:
  application:
    name: payment-service   ← This is the service name in the registry
eureka:
  client:
    service-url:
      defaultZone: http://localhost:8761/eureka/
  instance:
    prefer-ip-address: true
    lease-renewal-interval-in-seconds: 10
    lease-expiration-duration-in-seconds: 30
```

**Calling with client-side load balancing:**
```java
@Configuration
public class WebClientConfig {
    @Bean
    @LoadBalanced    // ← Magic annotation — wraps RestTemplate with Eureka lookup
    public RestTemplate restTemplate() {
        return new RestTemplate();
    }
}

@Service
@RequiredArgsConstructor
public class OrderService {
    private final RestTemplate restTemplate;

    public PaymentResponse processPayment(PaymentRequest req) {
        // "payment-service" is resolved from Eureka — not a real hostname
        return restTemplate.postForObject(
            "http://payment-service/v1/payments", req, PaymentResponse.class);
    }
}
```

---

## 3. Load Balancing Algorithms

### 3.1 Algorithm Comparison

| Algorithm | How It Works | Best For |
|-----------|-------------|---------|
| **Round Robin** | Cycle through instances sequentially | Homogeneous instances, uniform request cost |
| **Weighted Round Robin** | More requests to stronger instances | Mixed instance sizes |
| **Least Connections** | Send to instance with fewest active requests | Long-lived connections, variable request cost |
| **Random** | Pick randomly | Simple, low overhead |
| **IP Hash** | hash(clientIP) % N | Session affinity (sticky sessions) |
| **Least Response Time** | Pick fastest-responding instance | Latency-sensitive APIs |

### 3.2 Spring Cloud LoadBalancer Configuration

```java
@Configuration
@LoadBalancerClient(name = "payment-service",
                    configuration = CustomLoadBalancerConfig.class)
public class AppConfig { }

@Configuration
public class CustomLoadBalancerConfig {

    @Bean
    public ReactorLoadBalancer<ServiceInstance> randomLoadBalancer(
            Environment env,
            LoadBalancerClientFactory factory) {
        String name = env.getProperty(LoadBalancerClientFactory.PROPERTY_NAME);
        return new RandomLoadBalancer(
            factory.getLazyProvider(name, ServiceInstanceListSupplier.class), name);
    }
}
```

### 3.3 Health Checks — Only Route to Healthy Instances

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health, info
  endpoint:
    health:
      show-details: always

eureka:
  instance:
    health-check-url-path: /actuator/health
    status-page-url-path: /actuator/info
```

---

## 4. Kubernetes Service Discovery (Production Reality)

In Kubernetes, you rarely need Eureka — the platform provides DNS-based discovery:

```yaml
# payment-service Service definition
apiVersion: v1
kind: Service
metadata:
  name: payment-service
  namespace: production
spec:
  selector:
    app: payment-service    # Routes to pods with this label
  ports:
    - port: 80
      targetPort: 8080
  type: ClusterIP           # Internal DNS: payment-service.production.svc.cluster.local
```

```yaml
# order-service calls payment-service via DNS — no Eureka needed
# application.yml in order-service
payment:
  service:
    url: http://payment-service.production.svc.cluster.local
```

**Kubernetes load balancing is kube-proxy (iptables/IPVS) — it's transparent to the application.**

---

## 5. Health Check Patterns

```java
@Component
public class DatabaseHealthIndicator implements HealthIndicator {

    private final DataSource dataSource;

    @Override
    public Health health() {
        try (Connection conn = dataSource.getConnection()) {
            conn.prepareStatement("SELECT 1").execute();
            return Health.up()
                .withDetail("database", "responsive")
                .build();
        } catch (SQLException e) {
            return Health.down()
                .withDetail("database", "unreachable")
                .withException(e)
                .build();
        }
    }
}
```

```yaml
# Kubernetes liveness and readiness probes
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
  initialDelaySeconds: 10
  periodSeconds: 5
```

---

## 6. Hands-On Exercise

1. Set up a local Eureka server (Spring Boot Initializr → Eureka Server).
2. Create two instances of `payment-service` on ports 8081 and 8082, both registering with Eureka.
3. Create `order-service` using `@LoadBalanced RestTemplate` calling `http://payment-service`.
4. Send 10 requests to `order-service` — confirm it alternates between the two `payment-service` instances (add instance ID to the response).
5. Stop one `payment-service` instance — confirm `order-service` stops routing to it after the health check TTL expires.

---

## 7. Key Takeaways

1. **Server-side discovery (K8s Services, ALB) is simpler** — LB handles everything, clients are unaware.
2. **Client-side discovery (Eureka + Spring Cloud) gives more control** — useful outside Kubernetes.
3. **Always implement health checks** — the registry must know which instances are actually healthy.
4. **Least-connections beats round-robin for variable-cost requests** — when some requests are expensive, don't spray them equally.
5. **In production K8s, Eureka is often unnecessary** — DNS + Services + readiness probes is sufficient.

---

## 8. Connection to Previous Days

Scalability (Day 16) assumed a load balancer in front. Today you learned what it is. Rate limiting (Day 22) sits inside the load balancer or API gateway. Circuit breakers (coming tomorrow) complement service discovery — when a downstream service is detected unhealthy, the circuit opens and no more requests are routed to it.

---

## 9. Resources

- **[Spring Cloud Netflix Eureka docs](https://spring.io/projects/spring-cloud-netflix)** — Official reference.
- **[Kubernetes Services documentation](https://kubernetes.io/docs/concepts/services-networking/service/)** — K8s-native discovery.
- **[Consul service mesh](https://www.consul.io/docs/connect)** — HashiCorp's approach for multi-cloud.

---

*Day 24. In a world where services come and go, the registry is the phonebook — and the load balancer is the operator who picks up the call.*
