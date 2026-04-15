# Day 39 — Service Mesh & Istio

**Date:** 2026-04-15
**Phase:** 3 — Microservices & Cloud-Native
**Time Target:** 1–2 hours

---

## 1. Today's Focus Topic

A **service mesh** is a dedicated infrastructure layer that handles service-to-service communication — providing traffic management, mutual TLS (mTLS), observability, and policy enforcement **without touching application code**.

**Istio** is the most widely adopted open-source service mesh, deployed as a set of control-plane components (`istiod`) and data-plane sidecar proxies (Envoy) injected alongside every pod.

**Why it matters for your stack:**
- AKS (Azure Kubernetes Service) supports Istio as a managed add-on
- Eliminates cross-cutting concerns (retries, timeouts, circuit breaking) from Spring Boot services
- Enables zero-trust networking between microservices in the same cluster

---

## 2. Core Architecture

### Control Plane vs. Data Plane

```
┌──────────────────────────────────────┐
│           Control Plane              │
│  ┌────────────────────────────────┐  │
│  │           istiod               │  │
│  │  Pilot (traffic mgmt)          │  │
│  │  Citadel (mTLS certs / PKI)    │  │
│  │  Galley (config validation)    │  │
│  └────────────────────────────────┘  │
└──────────────────────────────────────┘
             │ xDS API (gRPC)
             ▼
┌──────────────────────────────────────┐
│           Data Plane                 │
│  Pod A                  Pod B        │
│  ┌────────┐  sidecar    ┌────────┐  │
│  │ App    │◄──Envoy────►│ App    │  │
│  └────────┘             └────────┘  │
└──────────────────────────────────────┘
```

**Key components:**
| Component | Role |
|-----------|------|
| `istiod` | Unified control plane — distributes config to all Envoy sidecars |
| Envoy proxy | Sidecar injected into every pod — intercepts all inbound/outbound traffic |
| Kiali | Service mesh dashboard — topology, health, traffic flow |
| Jaeger/Zipkin | Distributed tracing (auto-populated by Envoy) |
| Prometheus + Grafana | Metrics scraping (Envoy emits `istio_requests_total`, latency histograms) |

---

## 3. Traffic Management

Istio introduces two CRDs for fine-grained routing:

### VirtualService — Define Routing Rules

```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: order-service
spec:
  hosts:
    - order-service          # Kubernetes service name
  http:
    # Canary: 10% to v2, 90% to v1
    - route:
        - destination:
            host: order-service
            subset: v1
          weight: 90
        - destination:
            host: order-service
            subset: v2
          weight: 10
```

### DestinationRule — Define Subsets & Load Balancing

```yaml
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: order-service
spec:
  host: order-service
  trafficPolicy:
    connectionPool:
      tcp:
        maxConnections: 100
      http:
        h2UpgradePolicy: UPGRADE
    outlierDetection:             # Circuit breaking at mesh level
      consecutive5xxErrors: 5
      interval: 10s
      baseEjectionTime: 30s
  subsets:
    - name: v1
      labels:
        version: v1
    - name: v2
      labels:
        version: v2
```

### Retry & Timeout Policy

```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: payment-service
spec:
  hosts:
    - payment-service
  http:
    - timeout: 3s
      retries:
        attempts: 3
        perTryTimeout: 1s
        retryOn: 5xx,reset,connect-failure,retriable-4xx
      route:
        - destination:
            host: payment-service
```

> **Key insight:** These retries/timeouts are enforced at the sidecar level. Your Spring Boot service sees a clean request — no Resilience4j duplication needed at this layer (though defence-in-depth is fine).

---

## 4. mTLS & Security

Istio's **PeerAuthentication** resource enables automatic mutual TLS between all services in a namespace:

```yaml
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: production
spec:
  mtls:
    mode: STRICT     # All traffic must use mTLS; plaintext rejected
```

**What this gives you:**
- Every service-to-service call is encrypted in transit
- Both sides present certificates managed by `istiod`'s built-in CA
- Certificates rotate automatically (default 24h)
- No code changes required in your Spring Boot services

### AuthorizationPolicy — Least-Privilege Access

```yaml
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: order-service-policy
  namespace: production
spec:
  selector:
    matchLabels:
      app: order-service
  action: ALLOW
  rules:
    - from:
        - source:
            principals:
              - cluster.local/ns/production/sa/api-gateway-sa  # only API gateway allowed
    - to:
        - operation:
            methods: ["GET", "POST"]
            paths: ["/api/orders/*"]
```

---

## 5. Observability (Zero-Code Instrumentation)

Envoy sidecars emit **all four golden signals** automatically:

| Signal | Istio Metric |
|--------|-------------|
| Latency | `istio_request_duration_milliseconds` (histogram) |
| Traffic | `istio_requests_total` (counter) |
| Errors | `istio_requests_total{response_code=~"5.."}` |
| Saturation | `envoy_cluster_upstream_cx_active` (gauge) |

### Useful PromQL Queries

```promql
# P99 latency per service
histogram_quantile(0.99,
  sum(rate(istio_request_duration_milliseconds_bucket[5m])) by (destination_service_name, le)
)

# Error rate per service
sum(rate(istio_requests_total{response_code=~"5.."}[1m])) by (destination_service_name)
/
sum(rate(istio_requests_total[1m])) by (destination_service_name)
```

### Distributed Tracing

Envoy injects trace headers (`x-b3-traceid`, `x-request-id`) automatically. Your Spring Boot service only needs to **forward** them:

```java
// Spring Cloud Sleuth / Micrometer Tracing handles this automatically
// Just add the dependency and traces propagate end-to-end through Istio
```

```xml
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-tracing-bridge-otel</artifactId>
</dependency>
<dependency>
    <groupId>io.opentelemetry</groupId>
    <artifactId>opentelemetry-exporter-zipkin</artifactId>
</dependency>
```

---

## 6. Advanced Patterns

### Fault Injection (Chaos Testing in Mesh)

```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: inventory-service
spec:
  hosts:
    - inventory-service
  http:
    - fault:
        delay:
          percentage:
            value: 10.0      # Inject 500ms delay on 10% of requests
          fixedDelay: 500ms
        abort:
          percentage:
            value: 5.0       # Return HTTP 503 on 5% of requests
          httpStatus: 503
      route:
        - destination:
            host: inventory-service
```

> Use this in staging/dev to verify your circuit breakers and fallbacks actually work.

### Traffic Mirroring (Shadow Traffic)

```yaml
http:
  - route:
      - destination:
          host: order-service
          subset: v1
    mirror:
      host: order-service
      subset: v2           # Shadow copy of all traffic sent to v2 (async, no impact on response)
    mirrorPercentage:
      value: 100.0
```

Perfect for validating v2 behaviour under real production traffic before switching over.

---

## 7. Istio vs. Spring Cloud Gateway (Your Stack)

| Concern | Istio (Mesh) | Spring Cloud Gateway |
|---------|-------------|----------------------|
| Layer | Infrastructure (L4/L7) | Application (L7) |
| Scope | All service-to-service | North-south (external → cluster) |
| mTLS | Automatic | Not provided |
| Auth (JWT) | Supported but limited | Full JWT + custom filters |
| BFF aggregation | Not its job | Yes |
| Config | Kubernetes CRDs | Java/YAML |
| **Use both?** | Yes — complementary | Yes — complementary |

**Best practice:** Use Istio for east-west (service-to-service) and Spring Cloud Gateway for north-south (external ingress + BFF). They do not overlap.

---

## 8. Istio on AKS (Your Azure Stack)

Azure offers **Istio as a managed AKS add-on** (GA since 2023):

```bash
# Enable Istio on existing AKS cluster
az aks mesh enable --resource-group myRG --name myAKS

# Enable sidecar injection for a namespace
kubectl label namespace production istio-injection=enabled

# Verify sidecars are injected
kubectl get pods -n production -o jsonpath='{.items[*].spec.containers[*].name}'
```

**Terraform resource:**
```hcl
resource "azurerm_kubernetes_cluster" "main" {
  # ...existing config...
  service_mesh_profile {
    mode                             = "Istio"
    internal_ingress_gateway_enabled = true
    external_ingress_gateway_enabled = true
  }
}
```

---

## 9. Interview Questions

1. **What problem does a service mesh solve that a library like Resilience4j doesn't?**
   - Service mesh is language-agnostic infrastructure — no SDK per service, consistent policy across polyglot stack, handles traffic from non-Java services too.

2. **How does Istio implement circuit breaking? Where does the state live?**
   - Outlier detection in `DestinationRule` — Envoy tracks consecutive errors per upstream host and ejects it from the load-balancing pool. State is local to each sidecar (no central state store).

3. **If mTLS STRICT is enabled and a legacy service sends plaintext — what happens?**
   - The receiving sidecar rejects the connection. Fix: set `PeerAuthentication` to `PERMISSIVE` for that namespace during migration, then flip to `STRICT` once all services are mesh-enrolled.

4. **VirtualService vs. DestinationRule — what's the difference?**
   - VirtualService: routing rules (match → route → weight). DestinationRule: upstream policies (load balancing, connection pools, TLS settings, subsets).

5. **How does Istio affect latency?**
   - Sidecar adds ~0.5–1ms per hop (Envoy processing). Acceptable for most microservice calls. For ultra-low-latency, consider ambient mesh mode (no sidecar, ztunnel node-level proxy).

6. **What is Istio Ambient Mesh?**
   - Next-gen mode (beta) that eliminates per-pod sidecars in favour of a per-node `ztunnel` proxy. Reduces resource overhead and simplifies upgrades. Worth knowing for interviews.

---

## 10. Today's Practice Exercise

**Goal:** Design an Istio traffic policy for a 3-service order flow: `api-gateway → order-service → payment-service`

Write the following YAML configs:

1. `VirtualService` for `order-service` with:
   - 5% traffic to v2 (canary)
   - 3s timeout, 2 retries on 5xx

2. `DestinationRule` for `order-service` with:
   - Subsets v1 (label `version: v1`) and v2 (label `version: v2`)
   - Outlier detection: eject after 3 consecutive 5xx errors, 30s ejection window

3. `PeerAuthentication` for the `production` namespace in STRICT mTLS mode

4. `AuthorizationPolicy` allowing only `order-service` to call `payment-service`

**Stretch:** Add a fault injection VirtualService that introduces a 200ms delay on 5% of calls to `payment-service` to test downstream resilience.

---

## 11. Key Takeaways

- Istio moves cross-cutting concerns (retries, timeouts, circuit breaking, mTLS, observability) **out of code and into the mesh**
- **VirtualService** = routing rules; **DestinationRule** = upstream policies; **PeerAuthentication** = mTLS mode; **AuthorizationPolicy** = RBAC
- Envoy sidecars emit all four golden signals with zero application code changes
- Fault injection (`VirtualService` faults) is the right way to chaos-test your microservices without needing Chaos Monkey
- On AKS, Istio is available as a first-class managed add-on — enable it via `az aks mesh enable` or Terraform `service_mesh_profile`
- Istio complements (not replaces) Spring Cloud Gateway: mesh handles east-west, gateway handles north-south

---

## 12. Resources

- [Istio Documentation](https://istio.io/latest/docs/)
- [Istio on AKS (Microsoft Docs)](https://learn.microsoft.com/en-us/azure/aks/istio-about)
- [Envoy Proxy Docs](https://www.envoyproxy.io/docs/envoy/latest/)
- [Kiali — Service Mesh Observability](https://kiali.io/)
- [Istio Ambient Mesh Overview](https://istio.io/latest/docs/ambient/)

---

*Next: Day 40 — Docker Deep-Dive (multi-stage builds, layer caching, distroless images, BuildKit)*
