# Day 43 — Phase 3 Week 1 Capstone & Review

**Date:** 2026-04-24
**Phase:** 3 — Microservices & Cloud-Native
**Time Target:** 1–2 hours

---

## 1. Today's Focus Topic

**Phase 3 Week 1 Capstone — From a Monolith to a Cloud-Native Platform**

Week 1 of Phase 3 walked the full stack of what it takes to run Spring Boot microservices in production: how you decompose the domain (Day 37), where requests enter (Day 38), how services trust each other (Day 39), how you package the binary (Day 40), how you schedule it (Day 41), and how you scale, secure, and template it under load (Day 42). Today you stitch the six days into one coherent mental model, stress-test your understanding against interview-grade questions, and lock in the trade-offs before moving to Week 2.

**Why the capstone matters:** Every Week 1 topic in isolation is table-stakes. The senior-engineer interview question is never "what is an API Gateway"; it's "walk me through a request from the browser to MySQL through your cluster, and tell me what fails first when QPS doubles." That's what today builds.

---

## 2. Week 1 Recap

| Day | Topic | Core Takeaway |
|-----|-------|---------------|
| 37 | Microservices Patterns | DDD bounded contexts + Strangler Fig; DB-per-service; Sagas over 2PC |
| 38 | API Gateway & BFF | Spring Cloud Gateway = edge routing, auth, rate-limit; BFF = shape payloads per client |
| 39 | Service Mesh & Istio | Move cross-cutting concerns (mTLS, retry, canary, traces) out of app code |
| 40 | Docker Deep-Dive | Multi-stage builds, distroless base, non-root user, small layers |
| 41 | Kubernetes Core | Pod = atom; Deployment + Service + ConfigMap + Probes = minimum viable stack |
| 42 | Kubernetes Advanced | HPA/KEDA for elasticity, Helm for promotion, RBAC + NetPol for zero-trust |

---

## 3. End-to-End Architecture (Week 1 Composed)

```
                                Internet
                                    │
                         ┌──────────▼──────────┐
                         │   Azure Front Door  │  ← TLS, WAF, geo-routing
                         └──────────┬──────────┘
                                    │
                ┌───────────────────▼───────────────────┐
                │   Spring Cloud Gateway  (Day 38)      │
                │   • JWT filter  • Rate limit (Redis)  │
                │   • BFF aggregation for mobile/web    │
                └───────────────────┬───────────────────┘
                                    │
                    ┌───────────────┼───────────────┐
                    ▼               ▼               ▼
              ┌──────────┐    ┌──────────┐    ┌──────────┐
              │  order   │    │ payment  │    │ inventory │  ← Day 37: bounded contexts
              │ service  │    │ service  │    │  service  │    Day 40: multi-stage Dockerfile
              └────┬─────┘    └────┬─────┘    └────┬─────┘    Day 41: Deployment + Service
                   │               │               │          Day 42: HPA + NetworkPolicy
                   └───── Istio sidecars (Day 39) ─┘
                           mTLS · retries · traces
                                    │
                ┌───────────────────┼───────────────────┐
                ▼                   ▼                   ▼
         ┌────────────┐      ┌────────────┐      ┌────────────┐
         │  Kafka     │      │  MySQL     │      │  Redis     │
         │ (Strimzi)  │      │ (managed)  │      │  (managed) │
         └────────────┘      └────────────┘      └────────────┘
                                    │
                         ┌──────────▼──────────┐
                         │  Observability      │  ← Prometheus + Grafana
                         │  Loki · Tempo · OTel│    Jaeger via mesh
                         └─────────────────────┘

          Platform plane: Helm charts + Argo CD + AKS (Day 42)
          Security plane: Workload Identity + Key Vault CSI + RBAC
```

### Which concern lives where

| Concern | Owner | Why |
|---------|-------|-----|
| TLS termination at edge | Front Door | Offload before gateway |
| JWT verification | API Gateway | One place for auth, fail fast |
| Rate limit | API Gateway (Redis token bucket) | Global; per-user keys |
| mTLS between services | Istio sidecar | App code stays clean |
| Retries between services | Istio (idempotent) or Resilience4j (non-idempotent) | Choose per call |
| Canary 10/90 split | Istio VirtualService | Traffic-based, no redeploy |
| Autoscaling on Kafka lag | KEDA ScaledObject | CPU is a lagging signal |
| Promotion dev → prod | Helm values + Argo CD | Same chart, different values |
| Secrets | Azure Key Vault + CSI driver | Never lands in etcd |

---

## 4. Trace a Request End-to-End

**Scenario:** User taps "Place Order" in the mobile app.

1. **DNS + TLS** — `api.shop.example.com` resolves to Azure Front Door; TLS 1.3 handshake terminates at the edge.
2. **WAF + geo-routing** — Front Door rejects OWASP signatures, routes to the nearest AKS region.
3. **Ingress → Gateway** — NGINX/Gateway API forwards to `api-gateway` Service → one of N gateway pods.
4. **Gateway filters** — JWT validated, user claims extracted, rate-limit key `user:{id}` checked against Redis token bucket, request tagged with `X-User-Id`, `X-Tenant-Id`, and a `traceparent` header.
5. **BFF aggregation** (mobile route) — gateway fans out to `order-service` (create order), `inventory-service` (reserve stock), and `payment-service` (charge), merges the response.
6. **Sidecar intercept (Istio)** — each hop leaves the pod, the Envoy sidecar adds mTLS, enforces a 2s timeout with one retry (idempotent GETs only), and emits a trace span.
7. **Service logic (Spring Boot)** — `order-service` persists to MySQL, publishes `OrderPlaced` to Kafka outbox, returns 201.
8. **Event flow** — payment consumer picks up `PaymentRequested`, KEDA scales consumers on lag, settlement completes.
9. **Observability** — spans land in Tempo via OTel Collector, logs correlate by `trace_id` in Loki, Prometheus scrapes `/actuator/prometheus`.
10. **Response back** — gateway trims the internal fields, returns JSON to the client.

**What changes when QPS doubles?**
- Gateway pods scale first (HPA on CPU + RPS custom metric)
- Kafka consumer lag rises → KEDA adds payment/inventory consumers
- MySQL write IOPS saturates → writes back-pressure via outbox; reads move to replicas
- If still hot: shard orders by `user_id` (Day 27 playbook)

---

## 5. Self-Assessment Checklist

### Day 37 — Microservices Patterns
- [ ] Can you articulate the difference between *strategic* and *tactical* DDD?
- [ ] Given a module in a monolith, how do you decide whether to extract it?
- [ ] Sync vs async communication — name two tests you apply.
- [ ] Draw the saga for "place order → reserve stock → charge card → confirm" with the compensating path.

### Day 38 — API Gateway & BFF
- [ ] Why can't a service mesh replace an API gateway?
- [ ] Where do you put JWT verification: edge, gateway, or service? Why?
- [ ] How does Redis token-bucket rate limiting behave under a Redis failover?
- [ ] When do you add a BFF vs teach the client to aggregate?

### Day 39 — Service Mesh & Istio
- [ ] What is the sidecar injecting into your pod, and what happens if it crashes?
- [ ] mTLS strict vs permissive — when do you use each?
- [ ] Traffic split 90/10 in Istio — write the VirtualService.
- [ ] Name two things a mesh does not give you.

### Day 40 — Docker Deep-Dive
- [ ] What does multi-stage build buy you beyond image size?
- [ ] Why run containers as non-root, and how does the JVM react to that?
- [ ] What's in a distroless base image, and what isn't?
- [ ] How do you measure layer cache efficiency in CI?

### Day 41 — Kubernetes Core
- [ ] Explain the Pod lifecycle: Pending → Running → Terminating with hooks.
- [ ] What does `readinessProbe` being red mean vs `livenessProbe` being red?
- [ ] ConfigMap vs Secret vs Env from `valueFrom` — pick one for a JDBC URL with password.
- [ ] How does a Service route traffic when a Pod IP changes?

### Day 42 — Kubernetes Advanced
- [ ] HPA vs KEDA vs VPA vs Cluster Autoscaler — when do they compose?
- [ ] How does Helm `--atomic` change failure behaviour?
- [ ] Write the minimum NetworkPolicy that allows `order-service` → `mysql:3306` only.
- [ ] When do you run Kafka in-cluster (StatefulSet/Strimzi) vs managed?

---

## 6. Architecture Design Challenge

**Scenario:** A fintech startup (5M users, 500 TPS peak, regulated in EU + UK) asks you to replatform their Spring Boot monolith (single MySQL, single EC2 host) onto AKS over two quarters. Budget is tight, the team is 8 engineers, no platform team.

**Answer:**
1. What do you extract first and why? What do you keep in the monolith?
2. Which services get their own database and which share one initially?
3. Istio from day one, or later? Justify.
4. Where does JWT validation live? How do you rotate the signing key?
5. Kafka in AKS via Strimzi, or Confluent Cloud? What breaks your decision?
6. Helm chart layout — one chart per service, or one umbrella chart?
7. Which three metrics are on the CEO's dashboard? Which four alerts page oncall?
8. Disaster: AKS region is unavailable. What's your RTO, and what runs where?

**Sample answer (compressed):** Extract the highest-velocity bounded context first (payments) with DB-per-service + outbox; keep slow-changing domains (reporting, admin) in the monolith behind the Strangler Fig. Defer Istio until service count > 10 — mTLS via mesh has a real operational cost. JWT at Spring Cloud Gateway, Azure Key Vault for the signing key rotated every 90 days with a 48h overlap window. Confluent Cloud over Strimzi for an 8-person team — you don't have a Kafka on-call. One chart per service + one umbrella chart for environment promotion. Dashboard = end-to-end p99 latency, error rate, payment success rate. Alerts = p99 > SLO, 5xx > 0.5%, Kafka lag > 60s, DB connections > 80%. Region DR: multi-region AKS with active-active gateway, MySQL geo-replica with ≤30s RPO, RTO 10 min.

---

## 7. Common Interview Questions

**Q: Why run the API Gateway inside the cluster and not at the edge?**
Edge terminates TLS, does WAF, and geo-routes. Gateway needs JWT introspection, rate limit with Redis, header rewrites, and BFF aggregation — cheaper and safer next to the services. Often you have both.

**Q: A request times out intermittently between service A and B. Walk me through diagnosis.**
Traces first (is it A-side or B-side?), then logs joined by `trace_id`, then metrics (sidecar → upstream latency), then pod events (OOM? CrashLoop?), then NetworkPolicy (was it blocked?), then kube-proxy/iptables. Fix: tune timeout + retry at the mesh, bulkhead at the caller, increase replicas, cache if read-heavy.

**Q: How do you deliver a zero-downtime config change to 12 services?**
ConfigMap with a checksum annotation on the Deployment pod template, Helm upgrade --atomic, rolling restart driven by the checksum change, readiness probe gates traffic. For Spring Boot specifically, `spring.config.import=configtree:/config` re-reads mounted files without restart for non-startup properties.

**Q: Docker image is 900MB. What's your hit list?**
Multi-stage build (JRE not JDK in final); distroless or Alpine base; layer the jar as dependencies + classes (Spring Boot layertools); no dev tools in final stage; jlink custom runtime if you need sub-150MB; measure with `docker history`.

**Q: Service mesh sidecar adds latency. Is it worth it?**
Typically 1–3ms p99 added; you get mTLS, retries, traffic splitting, and L7 metrics for free. Worth it past ~10 services. Below that, do it in code with Spring Cloud LoadBalancer + Resilience4j.

---

## 8. Week 2 Preview — What's Next

| Day | Topic | Theme |
|-----|-------|-------|
| 44 | GitOps with Argo CD & Flux | Declarative delivery, drift detection |
| 45 | Event-Driven Microservices Deep Dive | Outbox, CDC (Debezium), exactly-once |
| 46 | Service Resilience in Depth | Bulkheads, hedged requests, chaos eng |
| 47 | Multi-Cluster & Multi-Region | Federation, failover, data locality |
| 48 | Cost-Aware Cloud-Native | Right-sizing, spot, FinOps basics |
| 49 | Microservices Testing Strategy | Contract tests (Pact), ephemeral envs |
| 50 | Phase 3 Week 2 Capstone | Production-ready platform review |

---

## 9. Key Takeaways

1. **The request path is the architecture.** If you can trace a request from DNS to MySQL and back, naming every component and failure mode, you own the system.
2. **Push cross-cutting concerns out of app code** in this order: Gateway (auth, rate limit) → Mesh (mTLS, retries, traces) → Platform (secrets, config, autoscaling).
3. **DB-per-service is a target, not a starting point** — share a DB by bounded context until write contention forces the split.
4. **Synchronous coupling is the enemy of availability** — move to events (outbox + Kafka) at every seam where you can tolerate eventual consistency.
5. **Use managed services until you can justify the alternative** — Confluent, Azure MySQL Flexible, Cosmos — before running stateful infra on K8s.
6. **Observability is not optional** at microservice scale; you cannot debug 20 services from `kubectl logs`.
7. **Promote with Helm + GitOps, not kubectl apply** — the delta between dev and prod must be a values file, nothing else.

---

## 10. Resources

- **Book:** *Building Microservices* (Sam Newman, 2nd ed.)
- **Book:** *Kubernetes Patterns* (Ibryam & Huß, 2nd ed.)
- **Book:** *Production Kubernetes* (Dotson et al.)
- **Spring Cloud Gateway:** https://spring.io/projects/spring-cloud-gateway
- **Istio Best Practices:** https://istio.io/latest/docs/ops/best-practices/
- **Helm Chart Best Practices:** https://helm.sh/docs/chart_best_practices/
- **KEDA scalers:** https://keda.sh/docs/latest/scalers/
- **Argo CD:** https://argo-cd.readthedocs.io/

---

*Next: Day 44 — GitOps with Argo CD & Flux (declarative delivery, drift detection, progressive rollouts)*
