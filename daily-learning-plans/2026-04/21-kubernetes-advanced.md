# Day 42 — Kubernetes Advanced

**Date:** 2026-04-21
**Phase:** 3 — Microservices & Cloud-Native
**Time Target:** 1–2 hours

---

## 1. Today's Focus Topic

Day 41 covered the core objects (Pods, Deployments, Services, ConfigMaps, probes). Day 42 moves to the production layer: **autoscaling under real load, templating with Helm, extending K8s with Operators/CRDs, tightening security with RBAC + NetworkPolicies + PodSecurity, and running stateful workloads (MySQL, Kafka) on AKS.** These are the gates between "it runs in K8s" and "it survives a Black Friday spike on a compliance-audited cluster."

**Why it matters for your stack:**
- HPA + KEDA let Kafka consumers auto-scale on **consumer-group lag**, not just CPU
- Helm is how every Spring Boot microservice at your scale is packaged and promoted across dev/uat/prod
- Operators run stateful infra (Postgres Operator, Strimzi for Kafka) the same way Deployments run stateless apps
- RBAC + NetworkPolicies are what auditors actually check — zero-trust inside the cluster
- StatefulSets + PVCs are the only way to run MySQL/Kafka on K8s without data loss

---

## 2. Autoscaling — HPA, VPA, KEDA, Cluster Autoscaler

### Horizontal Pod Autoscaler (HPA)

Scales replica count based on CPU, memory, or custom metrics.

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
  minReplicas: 3
  maxReplicas: 20
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
    - type: Pods
      pods:
        metric:
          name: http_requests_per_second
        target:
          type: AverageValue
          averageValue: "200"
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300   # avoid flapping
      policies:
        - type: Percent
          value: 50
          periodSeconds: 60
```

**Key rules:**
- HPA needs **metrics-server** (CPU/mem) or **Prometheus Adapter** (custom metrics)
- Always set `requests` on containers — HPA math is `currentUtilization = usage / request`
- Scale-down stabilisation window prevents thrashing during traffic dips

### KEDA — Event-Driven Autoscaling

HPA on CPU is blind to the backlog you actually care about. KEDA scales on external signals: **Kafka lag, RabbitMQ queue depth, Azure Service Bus messages, cron**.

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: payment-consumer-scaler
spec:
  scaleTargetRef:
    name: payment-consumer
  minReplicaCount: 2
  maxReplicaCount: 30
  pollingInterval: 15
  cooldownPeriod: 120
  triggers:
    - type: kafka
      metadata:
        bootstrapServers: kafka-bootstrap:9092
        consumerGroup: payment-processor
        topic: payments.created
        lagThreshold: "50"           # scale out when lag > 50 per partition
        offsetResetPolicy: latest
```

**Why it matters for you:** Your Kafka consumers idle 90% of the day and spike at settlement windows. KEDA takes them to zero and back to 30 pods without paying for always-on capacity. Works with AKS + Azure Event Hubs too.

### Vertical Pod Autoscaler (VPA)

Adjusts CPU/memory **requests** (not replicas) based on historical usage. Use in **recommendation mode** in prod — auto-update restarts pods.

### Cluster Autoscaler

Adds/removes **Nodes** when pending pods can't be scheduled. On AKS this is the built-in node pool autoscaler — set `min-count` and `max-count` per node pool. Pair spot node pools with on-demand for cost savings.

```bash
az aks nodepool update \
  --resource-group rg-prod \
  --cluster-name aks-prod \
  --name userpool \
  --enable-cluster-autoscaler \
  --min-count 3 --max-count 20
```

---

## 3. Helm — Packaging, Templating, Promotion

Raw YAML doesn't scale past two environments. Helm gives you **charts** (templated bundles) and **releases** (installed instances with values).

### Chart Structure

```
order-service-chart/
├── Chart.yaml
├── values.yaml              # defaults
├── values-dev.yaml
├── values-uat.yaml
├── values-prod.yaml
└── templates/
    ├── deployment.yaml
    ├── service.yaml
    ├── configmap.yaml
    ├── hpa.yaml
    └── _helpers.tpl
```

### Deployment Template (Spring Boot)

```yaml
# templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "order-service.fullname" . }}
  labels: {{- include "order-service.labels" . | nindent 4 }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels: {{- include "order-service.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      labels: {{- include "order-service.selectorLabels" . | nindent 8 }}
    spec:
      containers:
        - name: app
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          env:
            - name: SPRING_PROFILES_ACTIVE
              value: {{ .Values.springProfile | quote }}
          envFrom:
            - configMapRef:
                name: {{ include "order-service.fullname" . }}-config
            - secretRef:
                name: {{ include "order-service.fullname" . }}-secrets
          resources: {{- toYaml .Values.resources | nindent 12 }}
          livenessProbe:
            httpGet:
              path: /actuator/health/liveness
              port: 8080
            initialDelaySeconds: 30
          readinessProbe:
            httpGet:
              path: /actuator/health/readiness
              port: 8080
```

### Per-Environment Values

```yaml
# values-prod.yaml
replicaCount: 6
springProfile: prod
image:
  tag: "2026.04.21-a3f912"
resources:
  requests: { cpu: 500m, memory: 1Gi }
  limits:   { cpu: 2,    memory: 2Gi }
```

### Install / Upgrade

```bash
helm upgrade --install order-service ./order-service-chart \
  --namespace orders \
  --create-namespace \
  --values values-prod.yaml \
  --atomic --timeout 5m
```

`--atomic` rolls back on failure. `helm rollback order-service 1` reverts to a previous revision.

### Helm in CI/CD

- Chart repo (ChartMuseum / Azure Container Registry OCI) stores versioned charts
- Argo CD or Flux watches a GitOps repo of `values-*.yaml` and reconciles the cluster
- One chart, N releases per environment — the same binary everywhere

---

## 4. Operators & CRDs

A **Custom Resource Definition (CRD)** extends the Kubernetes API. An **Operator** is a controller that reconciles instances of that CRD toward the desired state — the same pattern K8s uses internally for Deployments.

### When to reach for an Operator

- **Use an existing operator** for anything stateful: Strimzi (Kafka), Zalando Postgres Operator, Elastic Cloud on K8s, Prometheus Operator
- **Build your own** only when you have domain-specific lifecycle logic that scripts + Helm hooks can't express (failovers, backups, schema migrations)

### Example: Strimzi Kafka Operator

```yaml
apiVersion: kafka.strimzi.io/v1beta2
kind: Kafka
metadata:
  name: payments-cluster
spec:
  kafka:
    replicas: 3
    listeners:
      - name: tls
        port: 9093
        type: internal
        tls: true
    storage:
      type: jbod
      volumes:
        - id: 0
          type: persistent-claim
          size: 200Gi
          class: managed-premium
  zookeeper:
    replicas: 3
    storage:
      type: persistent-claim
      size: 20Gi
```

The Strimzi operator watches this resource, creates the StatefulSets, manages rolling restarts, handles broker config changes, and runs topic/user CRs.

### Operator Maturity Model

| Level | Capability |
|-------|-----------|
| 1 | Basic install |
| 2 | Seamless upgrades |
| 3 | Full lifecycle (backup, restore) |
| 4 | Deep insights (metrics, alerts) |
| 5 | Auto-pilot (auto-scaling, auto-healing, auto-tuning) |

---

## 5. Ingress & Gateway API

### NGINX Ingress (classic)

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: api-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /$2
    cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  ingressClassName: nginx
  tls:
    - hosts: [api.example.com]
      secretName: api-tls
  rules:
    - host: api.example.com
      http:
        paths:
          - path: /orders(/|$)(.*)
            pathType: ImplementationSpecific
            backend:
              service:
                name: order-service
                port: { number: 80 }
```

### Gateway API (the successor)

Gateway API separates concerns: **GatewayClass** (infra), **Gateway** (listener, owned by platform team), **HTTPRoute** (routing rules, owned by app team). Supported by Istio, Contour, NGINX, Azure App Gateway for Containers.

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: orders-route
spec:
  parentRefs: [{ name: shared-gateway }]
  hostnames: ["api.example.com"]
  rules:
    - matches: [{ path: { type: PathPrefix, value: /orders } }]
      backendRefs:
        - name: order-service
          port: 80
```

**Use Ingress for single-team clusters. Use Gateway API when platform and app teams need separation of duties.**

---

## 6. Security — RBAC, NetworkPolicy, PodSecurity, Secrets

### RBAC

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: orders
  name: order-service-role
rules:
  - apiGroups: [""]
    resources: ["configmaps", "secrets"]
    verbs: ["get", "list", "watch"]
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: order-service
  namespace: orders
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: order-service-binding
  namespace: orders
subjects:
  - kind: ServiceAccount
    name: order-service
    namespace: orders
roleRef:
  kind: Role
  name: order-service-role
  apiGroup: rbac.authorization.k8s.io
```

Bind the Deployment's Pod template to `serviceAccountName: order-service`. **Never** use the `default` ServiceAccount in prod — and disable `automountServiceAccountToken` unless the pod talks to the K8s API.

### NetworkPolicy — Zero-Trust Inside the Cluster

Default K8s allows **all pod-to-pod traffic**. NetworkPolicies change that to deny-by-default.

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: order-service-policy
  namespace: orders
spec:
  podSelector:
    matchLabels: { app: order-service }
  policyTypes: [Ingress, Egress]
  ingress:
    - from:
        - podSelector: { matchLabels: { app: api-gateway } }
      ports: [{ port: 8080, protocol: TCP }]
  egress:
    - to:
        - podSelector: { matchLabels: { app: mysql } }
      ports: [{ port: 3306, protocol: TCP }]
    - to:
        - namespaceSelector: { matchLabels: { name: kube-system } }
      ports: [{ port: 53, protocol: UDP }]       # DNS
```

Requires a CNI that enforces NetPols — Calico, Cilium, or Azure CNI Overlay with NetworkPolicy enabled.

### Pod Security Standards (replaces PodSecurityPolicy)

Namespace labels enforce three levels: `privileged`, `baseline`, `restricted`.

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: orders
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/warn: restricted
```

`restricted` bans privileged containers, root users, hostPath mounts, and host namespace sharing.

### Secrets — Not Stored in Git

K8s Secrets are **base64**, not encrypted. Options for GitOps-friendly secret management:

| Tool | Trade-off |
|------|-----------|
| **Sealed Secrets** (Bitnami) | Encrypted manifest lives in Git, controller decrypts |
| **External Secrets Operator** | Pulls from Azure Key Vault / AWS Secrets Manager / HashiCorp Vault |
| **CSI Secret Store Driver** | Mounts secrets from Key Vault directly as a volume — no K8s Secret object |

For AKS, pair **Azure Key Vault CSI driver** with Workload Identity — no secrets ever land in etcd.

---

## 7. StatefulSets & Persistent Volumes

StatefulSet differences from Deployment:
- **Stable pod names:** `mysql-0`, `mysql-1`, `mysql-2`
- **Stable DNS:** `mysql-0.mysql-headless.db.svc.cluster.local`
- **Ordered startup & shutdown**
- **Persistent PVC per pod** (via `volumeClaimTemplates`)

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
spec:
  serviceName: mysql-headless
  replicas: 3
  selector:
    matchLabels: { app: mysql }
  template:
    metadata:
      labels: { app: mysql }
    spec:
      containers:
        - name: mysql
          image: mysql:8.4
          ports: [{ containerPort: 3306 }]
          volumeMounts:
            - name: data
              mountPath: /var/lib/mysql
  volumeClaimTemplates:
    - metadata: { name: data }
      spec:
        accessModes: [ReadWriteOnce]
        storageClassName: managed-premium
        resources:
          requests: { storage: 100Gi }
```

**Reality check:** prefer the managed service (Azure MySQL Flexible Server, Azure Cosmos, Confluent Cloud Kafka) over running stateful workloads in K8s unless you have a dedicated platform team. When you do run them in-cluster, use an Operator — never raw StatefulSets.

---

## 8. Interview Questions

1. **Your Kafka consumer pods are pegged at 90% CPU but lag is still growing. Would you use HPA on CPU, HPA on custom metrics, or KEDA? Why?**
   → KEDA on `lag per partition`. CPU is a lagging indicator; lag is the SLO. HPA on custom metrics works but KEDA handles scale-to-zero and has a native Kafka scaler.

2. **What's the difference between `requests` and `limits`, and what happens if I set limits without requests?**
   → Requests = scheduler reservation + HPA denominator. Limits = hard ceiling (CPU throttled, memory OOMKilled). If only limits are set, K8s sets requests = limits (Guaranteed QoS) — safe but you over-reserve.

3. **How does a StatefulSet pod keep the same PVC across restarts?**
   → `volumeClaimTemplates` creates one PVC per pod ordinal (`data-mysql-0`). The StatefulSet controller rebinds the same PVC when the pod is re-scheduled because the pod name is stable.

4. **Your cluster has 30 microservices. How do you enforce that `order-service` can only talk to `mysql` and `kafka`?**
   → Deny-all default NetworkPolicy per namespace + per-service allowlist ingress/egress policies. Requires a CNI that enforces policies (Calico, Cilium, Azure CNI Overlay + NetPol).

5. **When would you build a custom Operator vs use Helm + post-install hooks?**
   → Operator when state transitions are non-trivial (leader election, failover, rolling schema migrations) and must survive controller restarts. Helm hooks are fine for install-time one-shots but have no reconciliation loop.

6. **Explain the Pod-to-Service traffic path.**
   → Pod → cluster DNS resolves service name → stable ClusterIP → `kube-proxy` iptables/IPVS rule DNATs to a healthy Pod IP (from Endpoints/EndpointSlices) → CNI delivers.

7. **What's wrong with mounting Azure Key Vault secrets into a standard K8s Secret?**
   → The secret lands in etcd (base64, often unencrypted). Use CSI Secret Store Driver to mount directly as a volume so the secret never becomes a K8s Secret object. Pair with Workload Identity for auth.

8. **Your Helm upgrade left the release in `pending-upgrade` state. What happened and how do you recover?**
   → Timeout or failure mid-apply without `--atomic`. Recover with `helm rollback` to the last good revision or `helm history` + manual cleanup of stuck resources. Always use `--atomic --timeout` in CI.

---

## 9. Today's Practice Exercise

**Goal:** Package a Spring Boot service with Helm, autoscale it on Kafka lag, and lock it down with NetworkPolicy + RBAC.

### Step 1 — Helm chart
```bash
helm create order-service
# Replace templates/deployment.yaml with the template from §3
# Add templates/hpa.yaml and templates/keda-scaledobject.yaml
# Create values-dev.yaml and values-prod.yaml
helm install order-service ./order-service --values values-dev.yaml -n orders --create-namespace
```

### Step 2 — KEDA autoscaling
```bash
helm repo add kedacore https://kedacore.github.io/charts
helm install keda kedacore/keda -n keda --create-namespace
# Apply ScaledObject from §2 pointing at a Kafka topic
kubectl get hpa -n orders -w   # KEDA creates an HPA under the hood
```

### Step 3 — NetworkPolicy
```bash
# Default deny:
kubectl apply -f default-deny.yaml -n orders
# Targeted allows:
kubectl apply -f order-service-policy.yaml -n orders
# Verify:
kubectl run curl --rm -it --image=curlimages/curl -- sh
# curl http://order-service.orders  → should fail from default NS, succeed from api-gateway NS
```

### Step 4 — Load test & observe
```bash
# Produce 10k Kafka messages, watch KEDA scale consumer pods
kubectl get pods -n orders -w
kubectl describe scaledobject payment-consumer-scaler -n orders
```

**Success criteria:**
- Helm upgrades apply cleanly with `--atomic`
- Consumer scales from 2 → 20 pods when lag > 50, back to 2 within cooldown
- Curl from an unauthorised pod is blocked by NetworkPolicy
- Every Deployment has a non-default ServiceAccount with least-privilege RBAC

---

## 10. Key Takeaways

- **HPA for stateless web, KEDA for event-driven, VPA for right-sizing, Cluster Autoscaler for node capacity** — they compose, they don't overlap
- **Helm is mandatory** past two environments; pair with GitOps (Argo CD / Flux) for prod
- **Operators for stateful** — never run raw StatefulSets for Kafka/Postgres in production
- **Gateway API is the future** — Ingress still fine for single-team clusters
- **Zero-trust inside the cluster** = RBAC + NetworkPolicy + PodSecurity `restricted` + CSI Secret Store
- **Prefer managed services** (Azure MySQL, Cosmos, Confluent) over in-cluster stateful workloads unless you have a platform team

---

## 11. Resources

- **Kubernetes Autoscaling:** https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/
- **KEDA:** https://keda.sh/docs/latest/concepts/
- **Helm Best Practices:** https://helm.sh/docs/chart_best_practices/
- **Operator Framework:** https://operatorframework.io/
- **Gateway API:** https://gateway-api.sigs.k8s.io/
- **Pod Security Standards:** https://kubernetes.io/docs/concepts/security/pod-security-standards/
- **Azure Key Vault CSI Driver:** https://learn.microsoft.com/en-us/azure/aks/csi-secrets-store-driver
- **Strimzi Kafka Operator:** https://strimzi.io/docs/
- **Book:** *Kubernetes Patterns* (Ibryam & Huß, 2nd ed.)
- **Book:** *Production Kubernetes* (Dotson et al.)

---

*Next: Day 43 — Phase 3 Week 1 Capstone (Microservices → Gateway → Mesh → Docker → K8s Core → K8s Advanced)*
