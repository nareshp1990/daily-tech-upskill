# Docker & Kubernetes — Daily Prompts

> Tech stack: Docker, Azure Kubernetes Service (AKS), Helm, kubectl

---

## Docker

### Production Dockerfile for Spring Boot
```
Write a production-optimized multi-stage Dockerfile for my Spring Boot application.

Build: Maven/Gradle, Java 17
Requirements:
- Stage 1: Build with Maven/Gradle (cache dependencies layer separately)
- Stage 2: Runtime with eclipse-temurin:17-jre-alpine
- Non-root user (appuser, UID 1001)
- HEALTHCHECK using /actuator/health
- EXPOSE 8080
- JVM flags: -XX:+UseContainerSupport, -XX:MaxRAMPercentage=75.0
- Labels: maintainer, version, build-date
- .dockerignore file

Also show: how to build, run locally, and scan for vulnerabilities (docker scout / trivy).
```

### Docker Compose for local development
```
Create a docker-compose.yml for local development of my Spring Boot microservices.

Services needed:
- app (my Spring Boot service, hot-reload with volume mount)
- mysql (8.0, with init script for schema)
- kafka + zookeeper (or KRaft mode) + schema-registry
- redis (for caching)
- [any other services]

Requirements:
- Named volumes for data persistence
- Health checks with depends_on condition: service_healthy
- Environment variables from .env file
- Network isolation (backend network)
- Port mappings that don't conflict with host services
- Profiles: "full" (all services) vs "light" (just app + db)

Show: docker-compose.yml, .env.example, and startup script.
```

### Debug container issues
```
My Docker container for [service] is [failing to start / crashing / slow].

Error: [paste docker logs output]

Dockerfile: [paste or describe]

Walk me through debugging:
1. docker logs [container] — interpret the output
2. docker exec -it [container] sh — what to check inside
3. Common causes: wrong base image, missing env vars, port mismatch,
   OOM killed, file permissions, timezone issues
4. The fix
5. How to prevent this (CI check, health checks, better logging)
```

---

## Kubernetes / AKS

### Deploy a Spring Boot app to AKS
```
Write Kubernetes manifests to deploy my Spring Boot application to AKS.

Include all resources:
1. Deployment:
   - replicas: [N]
   - Resource requests/limits (cpu: 250m/500m, memory: 512Mi/1Gi)
   - Readiness probe: /actuator/health/readiness (initialDelay: 30s)
   - Liveness probe: /actuator/health/liveness (initialDelay: 60s)
   - Startup probe for slow-starting apps
   - Environment variables from ConfigMap and Secret
   - Image pull policy and imagePullSecrets for ACR
   - Pod anti-affinity (spread across nodes)
   - Graceful shutdown: terminationGracePeriodSeconds: 30

2. Service (ClusterIP)
3. ConfigMap (non-sensitive config)
4. Secret (DB password, Kafka credentials — reference only, actual values from Key Vault)
5. HPA: min=2, max=10, target CPU 70%, target memory 80%
6. PodDisruptionBudget: minAvailable: 1
7. Ingress (nginx or Azure Application Gateway) with TLS
8. ServiceAccount with Azure Workload Identity annotation

Show all YAML files organized in a k8s/ directory structure.
```

### Helm chart
```
Create a Helm chart for my Spring Boot microservice.

Structure:
- Chart.yaml (name, version, appVersion)
- values.yaml with configurable: image, replicas, resources, env vars,
  ingress, probes, HPA, service account
- values-dev.yaml, values-staging.yaml, values-prod.yaml overrides
- templates/: deployment, service, configmap, secret, hpa, ingress, pdb
- helpers.tpl with common labels and selector

Show: chart structure, key templates with Go templating, and
helm install / helm upgrade commands for each environment.
```

### Debug a pod issue
```
My pod [pod-name] in namespace [namespace] on AKS is in [state:
CrashLoopBackOff / OOMKilled / Pending / ImagePullBackOff / Evicted].

Walk me through the diagnostic commands in order:
1. kubectl describe pod [pod] -n [ns] — what to look for
2. kubectl logs [pod] -n [ns] --previous — for crashed pods
3. kubectl get events -n [ns] --sort-by='.lastTimestamp'
4. kubectl top pod [pod] -n [ns] — resource usage
5. State-specific diagnostics:
   - CrashLoopBackOff: check exit code, app startup, health probes
   - OOMKilled: check limits, analyze heap, right-size
   - Pending: check node resources, PVC, affinity rules
   - ImagePullBackOff: check image name, ACR auth, imagePullSecrets
6. The likely fix for each scenario
```

### Rolling update with zero downtime
```
Configure zero-downtime deployment for [service] on AKS.

Show:
1. Deployment strategy:
   - rollingUpdate: maxSurge=1, maxUnavailable=0
   - Why this combination ensures zero downtime

2. Application-side:
   - Spring Boot graceful shutdown: server.shutdown=graceful
   - lifecycle.timeout-per-shutdown-phase=20s
   - preStop hook: sleep 5 (let endpoints deregister)

3. Readiness probe tuned correctly:
   - Don't report ready until app can serve traffic
   - Probe path, period, failure threshold

4. PodDisruptionBudget:
   - minAvailable: 50% or 1 (whichever makes sense)

5. Verification:
   - kubectl rollout status deployment/[name]
   - Test with continuous curl during rollout
```

### Scale and resource tuning
```
Help me right-size the resource requests/limits for my Spring Boot app on AKS.

Current settings: requests=[X cpu, Y memory], limits=[X cpu, Y memory]
Observed: kubectl top shows [actual cpu, actual memory]
Pod count: [N], HPA target: [CPU%]

Java heap: -Xms[X] -Xmx[Y]

Questions:
1. Are my requests/limits too high or too low?
2. What should memory limit be relative to Java heap? (heap + metaspace + threads + buffers)
3. CPU: should I set a limit or leave it unbounded? Trade-offs?
4. HPA: should I scale on CPU, memory, or custom metric (RPS)?
5. Show the updated YAML with recommended values and explain each choice.
```

---

## AKS-Specific

### Azure Container Registry integration
```
Set up the connection between AKS and Azure Container Registry (ACR).

Show:
1. Attach ACR to AKS: az aks update --attach-acr
2. Or: imagePullSecrets with ACR token
3. Docker build & push to ACR in GitHub Actions
4. Deployment YAML referencing ACR image
5. Image tag strategy: git SHA vs semver vs latest (recommend and why)
```

### Azure Workload Identity
```
Set up Azure Workload Identity for my Spring Boot app on AKS to access
[Azure Key Vault / Azure Blob Storage / Azure MySQL] without storing credentials.

Show:
1. Create Managed Identity in Azure
2. Create Kubernetes ServiceAccount with annotations
3. Federated credential between AKS OIDC and Managed Identity
4. Pod spec referencing the ServiceAccount
5. Spring Boot config to use DefaultAzureCredential
6. Terraform code for all Azure resources

This replaces storing connection strings in Kubernetes Secrets.
```

---

## Quick One-Liners

```
Show me the kubectl command to [restart all pods / view logs of previous crash /
exec into a running pod / port-forward to a pod / copy a file from pod].
```

```
My pod is using more memory than expected. How do I get a heap dump from a
running Java pod on AKS?
```

```
Write a Kubernetes CronJob that runs [task] every [schedule] in my AKS cluster.
```

```
How do I view AKS node resource utilization? Show kubectl and Azure Monitor commands.
```

```
Compare Kubernetes Deployment vs StatefulSet vs DaemonSet. When to use each?
```

```
Show me how to set up kubectl context for multiple AKS clusters and switch between them.
```

---

*In K8s: if you didn't set resource limits, you don't have resource limits. If you didn't set probes, K8s can't help you.*
