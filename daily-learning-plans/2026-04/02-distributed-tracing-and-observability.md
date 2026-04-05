# Day 26 — Distributed Tracing & Observability
**Date:** 2026-04-02
**Phase:** 2 — System Design
**Time Target:** 1–2 hours

---

## 1. Today's Focus Topic
**Distributed Tracing & Observability — Seeing the Whole Picture Across Services**

In a monolith, a stack trace tells you exactly what happened. In a microservices architecture, a single user request may touch 10 services. When it's slow — which service is the bottleneck? When it fails — which service dropped the ball? Observability is the property of a system that lets you answer these questions from its outputs alone. The three pillars are **logs**, **metrics**, and **traces**. Today you master distributed tracing and how to tie all three together.

---

## 2. Core Concepts

### 2.1 The Observability Pillars

```
┌──────────────────────────────────────────────────────────┐
│                    OBSERVABILITY                          │
│                                                          │
│  LOGS           METRICS           TRACES                 │
│  ─────          ───────           ──────                 │
│  "What          "How many?        "Why is this           │
│  happened?"     How fast?         request slow?"         │
│  Structured     Aggregated        Distributed            │
│  events         numbers           causality              │
│                                                          │
│  Loki/ELK       Prometheus        Jaeger/Zipkin          │
│                 Grafana           Tempo                  │
└──────────────────────────────────────────────────────────┘
```

### 2.2 Distributed Tracing Concepts

```
A trace = one end-to-end request journey
A span  = one unit of work within a trace (one service call, one DB query)

Trace ID: abc123 (same across all services for one user request)

User → [span: API Gateway]──────────────────────────────── 250ms
            └─→ [span: order-service]───────────────── 200ms
                    ├─→ [span: inventory-check]── 50ms
                    └─→ [span: payment-service]── 140ms
                              └─→ [span: DB query]── 120ms ← BOTTLENECK

Trace ID: abc123
  Span IDs: gateway(1), order(2), inventory(3), payment(4), db(5)
  Parent:   -(1), 1(2), 2(3), 2(4), 4(5)
```

### 2.3 W3C TraceContext Standard

```
HTTP header: traceparent: 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01
             │             │                                    │              │
             version       traceId (128-bit)                    spanId (64-bit) flags
```

---

## 3. OpenTelemetry (OTel) — The Modern Standard

OpenTelemetry is the vendor-neutral standard for producing and collecting telemetry. It replaces Sleuth, Brave, Zipkin client, and Jaeger client — use OTel and send to any backend.

### 3.1 Spring Boot 3 Setup

```xml
<dependency>
  <groupId>io.micrometer</groupId>
  <artifactId>micrometer-tracing-bridge-otel</artifactId>
</dependency>
<dependency>
  <groupId>io.opentelemetry.instrumentation</groupId>
  <artifactId>opentelemetry-spring-boot-starter</artifactId>
  <version>2.3.0</version>
</dependency>
<!-- Export to Jaeger/Tempo/etc -->
<dependency>
  <groupId>io.opentelemetry</groupId>
  <artifactId>opentelemetry-exporter-otlp</artifactId>
</dependency>
```

```yaml
management:
  tracing:
    sampling:
      probability: 1.0        # 100% in dev; use 0.01-0.1 (1-10%) in prod

otel:
  exporter:
    otlp:
      endpoint: http://localhost:4317   # Jaeger / OpenTelemetry Collector
  resource:
    attributes:
      service.name: order-service
      deployment.environment: production
```

### 3.2 Automatic Instrumentation

Spring Boot auto-instruments:
- HTTP requests/responses (Servlet, WebFlux)
- RestTemplate, WebClient calls (propagates trace context)
- JDBC queries
- Kafka producer/consumer
- Spring Data
- Scheduled tasks

```java
// Trace context propagates automatically in RestTemplate/@LoadBalanced
@Service
public class OrderService {
    // OTel auto-instruments this HTTP call — no manual code needed
    public void createOrder(Order order) {
        restTemplate.postForObject("http://payment-service/v1/payments", ...);
    }
}
```

### 3.3 Manual Custom Spans

```java
@Service
@RequiredArgsConstructor
public class OrderService {

    private final Tracer tracer;

    public OrderResult processOrder(Order order) {
        Span span = tracer.nextSpan()
            .name("order.process")
            .tag("order.id", order.getId())
            .tag("order.amount", String.valueOf(order.getAmount()))
            .start();

        try (Tracer.SpanInScope ws = tracer.withSpan(span)) {
            // All work here is associated with this span
            inventoryService.reserve(order);
            paymentService.charge(order);
            return OrderResult.success();
        } catch (Exception e) {
            span.error(e);
            throw e;
        } finally {
            span.end();
        }
    }
}
```

---

## 4. Structured Logging — Tying Logs to Traces

```xml
<dependency>
  <groupId>net.logstash.logback</groupId>
  <artifactId>logstash-logback-encoder</artifactId>
</dependency>
```

```json
// Structured log output — traceId and spanId auto-injected
{
  "timestamp": "2026-04-02T10:15:30.123Z",
  "level": "INFO",
  "logger": "OrderService",
  "message": "Order created successfully",
  "traceId": "4bf92f3577b34da6a3ce929d0e0e4736",
  "spanId": "00f067aa0ba902b7",
  "service": "order-service",
  "orderId": "ord-789",
  "userId": "usr-123"
}
```

Now in Grafana, you can:
1. See a slow trace in Tempo (tracing backend)
2. Click the traceId
3. Jump directly to the correlated logs in Loki
4. See the metric spike in Prometheus at the same timestamp

---

## 5. Metrics — The Numbers Behind the Story

### 5.1 The Four Golden Signals (Google SRE)

| Signal | Metric | Alert When |
|--------|--------|-----------|
| **Latency** | `http_server_requests_seconds_bucket` | p99 > 2s |
| **Traffic** | `http_server_requests_total` | Sudden drop = possible outage |
| **Errors** | `http_server_requests_total{status=~"5.."}` | Error rate > 1% |
| **Saturation** | `jvm_threads_live`, CPU, `hikaricp_connections_active` | > 80% utilisation |

### 5.2 Spring Boot Micrometer Metrics

```yaml
management:
  metrics:
    distribution:
      percentiles-histogram:
        http.server.requests: true   # Enable histogram for latency percentiles
      percentiles:
        http.server.requests: [0.5, 0.95, 0.99]
  endpoints:
    web:
      exposure:
        include: health, metrics, prometheus
```

```java
// Custom business metrics
@Service
@RequiredArgsConstructor
public class OrderService {

    private final MeterRegistry meterRegistry;
    private final Counter ordersCreated;
    private final Timer orderProcessingTime;

    public OrderService(MeterRegistry registry) {
        this.ordersCreated = Counter.builder("orders.created")
            .tag("region", "eu-west")
            .description("Total orders created")
            .register(registry);
        this.orderProcessingTime = Timer.builder("orders.processing.time")
            .description("Order processing duration")
            .register(registry);
    }

    public OrderResult processOrder(Order order) {
        return orderProcessingTime.record(() -> {
            OrderResult result = doProcess(order);
            ordersCreated.increment();
            return result;
        });
    }
}
```

---

## 6. Observability Stack Setup

```yaml
# docker-compose.yml — Local observability stack
services:
  # OpenTelemetry Collector — receives OTel, fans out to backends
  otel-collector:
    image: otel/opentelemetry-collector:latest
    ports:
      - "4317:4317"   # gRPC
      - "4318:4318"   # HTTP

  # Traces backend
  jaeger:
    image: jaegertracing/all-in-one:latest
    ports:
      - "16686:16686"  # Jaeger UI

  # Metrics backend
  prometheus:
    image: prom/prometheus:latest
    ports:
      - "9090:9090"

  # Visualisation
  grafana:
    image: grafana/grafana:latest
    ports:
      - "3000:3000"
```

---

## 7. Hands-On Exercise

1. Add OTel + Micrometer tracing to a Spring Boot 3 app.
2. Run Jaeger locally via Docker.
3. Create two services: `order-service` calling `payment-service` via RestTemplate.
4. Make a request and open Jaeger UI — find the trace, see both spans.
5. Add a custom span in `order-service.processOrder()` with tags.
6. Introduce a 500ms `Thread.sleep()` in `payment-service` — confirm Jaeger shows the bottleneck.
7. Enable structured JSON logging and verify `traceId` appears in log output.

---

## 8. Key Takeaways

1. **Traces answer "why is this slow"** — logs tell you what, metrics tell you how many, traces tell you where.
2. **OTel is the vendor-neutral choice** — instrument once, route to any backend.
3. **Correlate all three signals by traceId** — structured logs must include traceId and spanId.
4. **Sample in production** — 1–10% sampling is typical; head-based vs tail-based sampling trade-offs.
5. **The four golden signals are the minimum** — latency, traffic, errors, saturation cover 90% of production issues.

---

## 9. Connection to Previous Days

Day 9 (LLM Observability) introduced OTel tracing for AI pipelines. Today you apply the same principles across all microservices. Circuit breakers (Day 25) generate events that should be observed via metrics (circuit state changes). All the patterns from Phase 2 Week 1 need observability to validate in production.

---

## 10. Resources

- **[OpenTelemetry Java documentation](https://opentelemetry.io/docs/instrumentation/java/)** — Comprehensive instrumentation guide.
- **[Micrometer documentation](https://micrometer.io/docs)** — Metrics for JVM applications.
- **[Grafana LGTM stack](https://grafana.com/docs/grafana/latest/getting-started/get-started-grafana-prometheus/)** — Loki + Grafana + Tempo + Mimir.

---

*Day 26. You can't fix what you can't see — observability is the flashlight in the distributed darkness.*
