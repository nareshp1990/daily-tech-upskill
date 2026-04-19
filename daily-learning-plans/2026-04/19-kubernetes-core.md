# Day 41 — Kubernetes Core

**Date:** 2026-04-19
**Phase:** 3 — Microservices & Cloud-Native
**Time Target:** 1–2 hours

---

## 1. Today's Focus Topic

Kubernetes is the de-facto runtime for production microservices. After Docker containers are built, K8s takes over: scheduling, self-healing, scaling, service discovery, and rolling updates. Understanding the core objects and control-plane mechanics is a prerequisite for everything else — AKS, Helm, GitOps, HPA, and K8s-native observability all sit on top of these fundamentals.

**Why it matters for your stack:**
- Every Spring Boot microservice you deploy to AKS runs in a Pod scheduled by K8s
- K8s ConfigMaps and Secrets replace application-level config files in production
- Liveness/readiness probes integrate directly with Spring Boot Actuator
- Deployment rolling-update strategy is your zero-downtime deploy mechanism
- Resource requests/limits are the foundation for cost control and bin-packing

---

## 2. Kubernetes Architecture

### Control Plane Components

```
┌───────────────────────── Control Plane (AKS-managed) ─────────────────────────┐
│                                                                                  │
│  ┌──────────────────┐  ┌──────────────────┐  ┌────────────────────────────┐   │
│  │   kube-apiserver │  │  kube-scheduler  │  │  kube-controller-manager   │   │
│  │  (REST gateway)  │  │ (Pod placement)  │  │ (ReplicaSet, Node, Endpoint)│   │
│  └──────────────────┘  └──────────────────┘  └────────────────────────────┘   │
│                                                                                  │
│  ┌──────────────────┐                                                           │
│  │      etcd        │  ← Distributed KV store — single source of truth         │
│  │  (cluster state) │                                                           │
│  └──────────────────┘                                                           │
└──────────────────────────────────────────────────────────────────────────────────┘

┌──────────────────────────── Worker Node ───────────────────────────────────────┐
│                                                                                  │
│  ┌──────────┐  ┌──────────┐  ┌────────────────────────────────────────────┐   │
│  │ kubelet  │  │ kube-    │  │                 Pods                        │   │
│  │(node     │  │ proxy    │  │  [container] [container]   [container]      │   │
│  │ agent)   │  │(iptables)│  │  [  shared network ns  ]   [  volumes  ]   │   │
│  └──────────┘  └──────────┘  └────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────────────────────────────────┘
```

### Request Flow: `kubectl apply -f deployment.yaml`

1. `kubectl` → HTTPS → **kube-apiserver** (authenticates, authorises, validates)
2. apiserver writes to **etcd**
3. **kube-controller-manager** watches etcd — detects new Deployment, creates ReplicaSet, then Pods
4. **kube-scheduler** watches for unscheduled Pods, assigns each to a Node based on resources and affinity
5. **kubelet** on the target Node pulls the container image and starts the container via containerd
6. kubelet reports Pod status back to apiserver

---

## 3. Core Kubernetes Objects

### Pod — The Smallest Deployable Unit

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: order-service-pod
  namespace: orders
  labels:
    app: order-service
    version: "1.0.0"
spec:
  containers:
    - name: order-service
      image: myregistry.azurecr.io/order-service:1.0.0
      ports:
        - containerPort: 8080
      env:
        - name: SPRING_PROFILES_ACTIVE
          value: "prod"
      # Resource management — always set in production
      resources:
        requests:
          memory: "256Mi"
          cpu: "250m"
        limits:
          memory: "512Mi"
          cpu: "500m"
      # Spring Boot Actuator integration
      livenessProbe:
        httpGet:
          path: /actuator/health/liveness
          port: 8080
        initialDelaySeconds: 30
        periodSeconds: 10
        failureThreshold: 3
      readinessProbe:
        httpGet:
          path: /actuator/health/readiness
          port: 8080
        initialDelaySeconds: 20
        periodSeconds: 5
        failureThreshold: 3
```

**Pods are ephemeral** — never create bare Pods in production. Use Deployments.

---

### Deployment — Desired State + Rolling Updates

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
  namespace: orders
spec:
  replicas: 3
  selector:
    matchLabels:
      app: order-service
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1        # Allow 1 extra Pod during rollout
      maxUnavailable: 0  # Never reduce below desired (zero-downtime)
  template:
    metadata:
      labels:
        app: order-service
        version: "1.2.0"
    spec:
      containers:
        - name: order-service
          image: myregistry.azurecr.io/order-service:1.2.0
          resources:
            requests:
              memory: "256Mi"
              cpu: "250m"
            limits:
              memory: "512Mi"
              cpu: "500m"
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
            initialDelaySeconds: 20
            periodSeconds: 5
          envFrom:
            - configMapRef:
                name: order-service-config
            - secretRef:
                name: order-service-secrets
```

**Rolling update mechanics:**
1. Create 1 new Pod (maxSurge=1) → wait for readiness probe to pass
2. Terminate 1 old Pod (maxUnavailable=0 maintained)
3. Repeat until all Pods updated
4. Traffic is only sent to Pods that pass the readiness probe

---

### Service — Stable Networking Endpoint

```yaml
# ClusterIP — internal traffic (microservice-to-microservice)
apiVersion: v1
kind: Service
metadata:
  name: order-service
  namespace: orders
spec:
  type: ClusterIP
  selector:
    app: order-service          # Routes to all Pods matching this label
  ports:
    - name: http
      port: 80
      targetPort: 8080
      protocol: TCP
```

| Service Type | Use Case | AKS |
|---|---|---|
| `ClusterIP` | Internal microservice-to-microservice | Default |
| `NodePort` | Dev/testing, direct Node access | Rarely in prod |
| `LoadBalancer` | Expose service externally | Creates Azure Load Balancer |
| `ExternalName` | Map to external DNS (e.g., Azure SQL) | DNS alias |

**DNS resolution in K8s:** `order-service.orders.svc.cluster.local` resolves to the ClusterIP from any Pod in the cluster.

---

### ConfigMap & Secret — Externalised Configuration

```yaml
# ConfigMap — non-sensitive config
apiVersion: v1
kind: ConfigMap
metadata:
  name: order-service-config
  namespace: orders
data:
  SPRING_DATASOURCE_URL: "jdbc:mysql://mysql-service:3306/orders_db"
  SPRING_KAFKA_BOOTSTRAP_SERVERS: "kafka-service:9092"
  SPRING_REDIS_HOST: "redis-service"
  MANAGEMENT_ENDPOINTS_WEB_EXPOSURE_INCLUDE: "health,info,prometheus"
  LOG_LEVEL: "INFO"
---
# Secret — sensitive config (base64-encoded in etcd)
apiVersion: v1
kind: Secret
metadata:
  name: order-service-secrets
  namespace: orders
type: Opaque
data:
  SPRING_DATASOURCE_PASSWORD: cGFzc3dvcmQxMjM=   # base64("password123")
  JWT_SECRET: c3VwZXJzZWNyZXRrZXkxMjM0NTY=
```

**In production on AKS:** Use Azure Key Vault + Secrets Store CSI Driver instead of K8s Secrets (avoids base64-in-etcd).

---

### Namespace — Logical Isolation

```bash
# Typical namespace layout for a microservices platform
kubectl create namespace orders
kubectl create namespace inventory
kubectl create namespace payments
kubectl create namespace monitoring    # Prometheus, Grafana
kubectl create namespace ingress-nginx
```

Namespaces enable:
- **Resource quotas** per team/environment
- **Network policies** to restrict cross-service traffic
- **RBAC** scoped to a namespace

---

## 4. Spring Boot on AKS — Full Deployment YAML

```yaml
# Full deployment spec for a production Spring Boot service on AKS
---
apiVersion: v1
kind: Namespace
metadata:
  name: orders
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: order-service-config
  namespace: orders
data:
  SPRING_PROFILES_ACTIVE: "prod"
  SPRING_DATASOURCE_URL: "jdbc:mysql://mysql.orders.svc.cluster.local:3306/orders_db"
  SPRING_KAFKA_BOOTSTRAP_SERVERS: "kafka.kafka.svc.cluster.local:9092"
  MANAGEMENT_ENDPOINTS_WEB_EXPOSURE_INCLUDE: "health,info,prometheus"
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
  namespace: orders
  annotations:
    deployment.kubernetes.io/revision: "1"
spec:
  replicas: 3
  selector:
    matchLabels:
      app: order-service
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    metadata:
      labels:
        app: order-service
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/path: "/actuator/prometheus"
        prometheus.io/port: "8080"
    spec:
      serviceAccountName: order-service-sa
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
      containers:
        - name: order-service
          image: myacr.azurecr.io/order-service:1.2.0
          imagePullPolicy: Always
          ports:
            - name: http
              containerPort: 8080
          envFrom:
            - configMapRef:
                name: order-service-config
            - secretRef:
                name: order-service-secrets
          resources:
            requests:
              memory: "256Mi"
              cpu: "250m"
            limits:
              memory: "512Mi"
              cpu: "500m"
          livenessProbe:
            httpGet:
              path: /actuator/health/liveness
              port: 8080
            initialDelaySeconds: 45    # JVM startup time
            periodSeconds: 10
            failureThreshold: 3
            timeoutSeconds: 5
          readinessProbe:
            httpGet:
              path: /actuator/health/readiness
              port: 8080
            initialDelaySeconds: 30
            periodSeconds: 5
            failureThreshold: 3
            timeoutSeconds: 3
          startupProbe:               # Prevents liveness from killing slow-starting JVM
            httpGet:
              path: /actuator/health/liveness
              port: 8080
            failureThreshold: 30      # 30 * 10s = 5 min max startup time
            periodSeconds: 10
      imagePullSecrets:
        - name: acr-pull-secret
---
apiVersion: v1
kind: Service
metadata:
  name: order-service
  namespace: orders
spec:
  type: ClusterIP
  selector:
    app: order-service
  ports:
    - name: http
      port: 80
      targetPort: 8080
```

---

## 5. Essential kubectl Commands

```bash
# Context & cluster
kubectl config get-contexts
kubectl config use-context aks-prod

# Namespace shorthand (set default namespace)
kubectl config set-context --current --namespace=orders

# Apply manifests
kubectl apply -f deployment.yaml
kubectl apply -f k8s/                  # Apply all files in directory

# Inspect
kubectl get pods -n orders -o wide
kubectl get pods -n orders --watch      # Live updates
kubectl describe pod order-service-xxx -n orders
kubectl logs order-service-xxx -n orders --follow
kubectl logs order-service-xxx -n orders --previous  # Crashed container logs

# Exec into a running container
kubectl exec -it order-service-xxx -n orders -- /bin/sh

# Deployment operations
kubectl rollout status deployment/order-service -n orders
kubectl rollout history deployment/order-service -n orders
kubectl rollout undo deployment/order-service -n orders       # Rollback
kubectl scale deployment order-service --replicas=5 -n orders

# Debug
kubectl get events -n orders --sort-by='.lastTimestamp'
kubectl top pods -n orders              # Requires metrics-server
kubectl top nodes

# Port-forward for local testing
kubectl port-forward svc/order-service 8080:80 -n orders
```

---

## 6. Resource Requests vs Limits

```
requests: What the Pod is guaranteed (used for scheduling)
limits:   Maximum the Pod can use (enforced by cgroups)
```

| Scenario | Requests | Limits | Effect |
|---|---|---|---|
| OOMKilled | Low | Too low | Container killed when it exceeds memory limit |
| CPU Throttling | Low | Too low | Container throttled — latency spikes |
| Eviction | Very low | None | Pod evicted during Node memory pressure |
| Unschedulable | Higher than Node capacity | — | Pod stays Pending |

**Best practice for Spring Boot (512 MB heap):**
```yaml
resources:
  requests:
    memory: "384Mi"   # < 512Mi heap → spring needs overhead too
    cpu: "250m"       # 0.25 cores guaranteed
  limits:
    memory: "640Mi"   # heap + metaspace + native memory
    cpu: "1000m"      # 1 core max
```

Set `-Xmx` in JVM args to stay comfortably under the memory limit:
```yaml
env:
  - name: JAVA_TOOL_OPTIONS
    value: "-Xmx448m -Xms256m -XX:+UseContainerSupport"
```

---

## 7. Liveness vs Readiness vs Startup Probes

| Probe | Failing Action | Use For |
|---|---|---|
| **Liveness** | Kill + restart container | Detect deadlocks, hung threads |
| **Readiness** | Remove from Service endpoints | Graceful startup, dependency unavailable |
| **Startup** | Kill + restart (until passed) | Slow-starting JVMs, avoid premature liveness kills |

**Spring Boot Actuator setup:**

```yaml
# application.yml
management:
  endpoint:
    health:
      probes:
        enabled: true        # Enables /actuator/health/liveness and /readiness
      show-details: always
  health:
    livenessState:
      enabled: true
    readinessState:
      enabled: true
```

Mark service as unready when downstream is unavailable (e.g., Kafka):
```java
@Component
public class KafkaReadinessIndicator implements ApplicationListener<KafkaEvent> {
    @Autowired
    private ApplicationContext context;

    @EventListener
    public void onKafkaDown(KafkaStoppedEvent event) {
        AvailabilityChangeEvent.publish(context, ReadinessState.REFUSING_TRAFFIC);
    }
}
```

---

## 8. Interview Questions

**Q1: What is the difference between a Pod and a Deployment?**
> A Pod is a single instance of one or more containers sharing network and storage. A Deployment is a controller that manages a desired number of Pod replicas, handles rolling updates, and enables rollback. You should never create bare Pods in production — if a bare Pod dies, it is not recreated. A Deployment's ReplicaSet ensures the desired count is always maintained.

**Q2: How does Kubernetes service discovery work?**
> K8s runs CoreDNS in the cluster. Every Service gets a stable DNS name: `<service>.<namespace>.svc.cluster.local`. When a Pod sends a request to `order-service`, CoreDNS resolves it to the ClusterIP. kube-proxy maintains iptables/IPVS rules that load-balance across all healthy Pod IPs (Endpoints). Pods are added/removed from Endpoints by the Endpoint controller based on readiness probe status.

**Q3: What happens when a Pod's readiness probe fails?**
> The Pod is removed from the Service's Endpoints list, so no new traffic is routed to it. The Pod is not restarted (that's the liveness probe's job). If the readiness probe starts passing again, the Pod is re-added to Endpoints. This is used for graceful startup (wait until connected to DB/Kafka) and graceful shutdown (drain in-flight requests before termination).

**Q4: How do requests and limits affect scheduling and runtime?**
> Requests are used by kube-scheduler to find a Node with sufficient available capacity — a Pod with 512Mi request is only scheduled on a Node with ≥512Mi allocatable memory free. Limits are enforced at runtime by cgroups: memory limits cause OOMKill if exceeded; CPU limits cause throttling. Setting limits lower than requests is invalid. Setting limits much higher than requests risks noisy-neighbour issues.

**Q5: What is a rolling update and how does `maxSurge`/`maxUnavailable` control it?**
> A rolling update replaces old Pods with new ones incrementally. `maxSurge` controls how many extra Pods can exist above the desired count during rollout (adds capacity headroom). `maxUnavailable` controls how many Pods can be unavailable below desired count. Setting `maxSurge=1, maxUnavailable=0` means we always maintain full capacity — the new Pod must pass readiness before an old Pod is terminated. This is the zero-downtime deployment strategy.

---

## 9. Today's Practice Exercise

Deploy a Spring Boot service to a local K8s cluster (Docker Desktop K8s or minikube):

**Step 1 — Enable local K8s**
```bash
# Docker Desktop: Settings → Kubernetes → Enable Kubernetes
# OR
minikube start --memory=4096 --cpus=4
```

**Step 2 — Build and load the image**
```bash
# Docker Desktop K8s (image already available locally)
docker build -t order-service:local .

# minikube (load image into minikube's Docker daemon)
minikube image load order-service:local
```

**Step 3 — Create namespace and apply manifests**
```bash
kubectl create namespace orders
kubectl apply -f k8s/configmap.yaml -n orders
kubectl apply -f k8s/deployment.yaml -n orders
kubectl apply -f k8s/service.yaml -n orders
```

**Step 4 — Verify**
```bash
kubectl get all -n orders
kubectl rollout status deployment/order-service -n orders
kubectl logs -l app=order-service -n orders --follow
```

**Step 5 — Test rolling update**
```bash
# Change image tag and apply
kubectl set image deployment/order-service \
  order-service=order-service:local-v2 -n orders
kubectl rollout status deployment/order-service -n orders
kubectl rollout history deployment/order-service -n orders
```

**Step 6 — Test rollback**
```bash
kubectl rollout undo deployment/order-service -n orders
kubectl rollout status deployment/order-service -n orders
```

**Step 7 — Observe probe behaviour**
```bash
# Temporarily break the readiness endpoint and watch the Pod go unready
kubectl exec -it order-service-xxx -n orders -- \
  curl -X POST http://localhost:8080/actuator/health/readiness/refuse
kubectl get endpoints order-service -n orders --watch
```

---

## 10. Key Takeaways

- **Pod** = smallest unit; **Deployment** = desired state controller; **Service** = stable network endpoint; **ConfigMap/Secret** = externalised config
- Rolling updates with `maxSurge=1, maxUnavailable=0` give zero-downtime deploys; readiness probe gates traffic
- Resource `requests` drive scheduling; `limits` enforce runtime caps — set both or face evictions and throttles
- Spring Boot's `/actuator/health/liveness` and `/actuator/health/readiness` are first-class K8s probe endpoints
- Add a **startup probe** for JVM services to prevent liveness from killing a container that's just slow to start
- K8s DNS (`<service>.<namespace>.svc.cluster.local`) replaces hardcoded IPs and Eureka-style service registries in cloud-native deployments

---

## 11. Resources

- [Kubernetes Concepts](https://kubernetes.io/docs/concepts/) — official docs
- [kubectl Cheat Sheet](https://kubernetes.io/docs/reference/kubectl/cheatsheet/)
- [Spring Boot on Kubernetes](https://spring.io/guides/gs/spring-boot-kubernetes/)
- [AKS Best Practices](https://learn.microsoft.com/en-us/azure/aks/best-practices)
- [K8s Resource Management](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/)

---

*Next: Day 42 — Kubernetes Advanced (HPA, RBAC, Network Policies, Helm, Persistent Volumes)*
