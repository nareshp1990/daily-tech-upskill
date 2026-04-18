# Day 40 — Docker Deep-Dive

**Date:** 2026-04-18
**Phase:** 3 — Microservices & Cloud-Native
**Time Target:** 1–2 hours

---

## 1. Today's Focus Topic

Docker is far more than a packaging tool — it is a lightweight OS-level virtualisation platform built on Linux kernel primitives. Understanding its internals makes you a better architect: you write leaner images, debug container behaviour confidently, and design secure microservice deployments.

**Why it matters for your stack:**
- Every Spring Boot service you deploy to AKS runs inside a Docker container
- Image size directly affects cold-start latency in Kubernetes
- Multi-stage builds eliminate dev-only dependencies from production images
- Dockerfile patterns become shared contracts between Dev and Platform Engineering teams

---

## 2. Docker Internals

### Kernel Primitives Docker Uses

| Primitive | What It Does | Docker Usage |
|-----------|-------------|--------------|
| **Namespaces** | Isolate process view of system resources | PID, NET, MNT, UTS, IPC, USER namespaces per container |
| **cgroups** | Limit and account resource usage | CPU shares, memory limits, I/O throttling |
| **UnionFS** | Overlay filesystem for layered images | OverlayFS stacks read-only image layers + writable container layer |
| **seccomp** | Syscall filtering | Blocks dangerous syscalls (e.g., `ptrace`, `mount`) by default |

### Image Layer Architecture

```
┌─────────────────────────────┐
│   Container Layer (R/W)     │  ← Created at `docker run`
├─────────────────────────────┤
│   COPY target/app.jar ...   │  ← Layer 4 (your app)
├─────────────────────────────┤
│   RUN apt-get install ...   │  ← Layer 3
├─────────────────────────────┤
│   FROM eclipse-temurin:21   │  ← Layer 2 (JDK base)
├─────────────────────────────┤
│   FROM ubuntu:22.04         │  ← Layer 1 (OS base)
└─────────────────────────────┘
```

Each `RUN`, `COPY`, `ADD` instruction creates a new read-only layer. Layers are content-addressed (SHA256) and shared across images on the same host — pulling `eclipse-temurin:21` twice downloads it once.

**Layer cache invalidation:** Any change to a layer invalidates all layers below it. Place frequently-changing instructions (COPY app jar) **after** stable ones (dependency installation).

---

## 3. Multi-Stage Builds for Spring Boot

### Naive (Anti-Pattern) Dockerfile

```dockerfile
FROM eclipse-temurin:21-jdk
WORKDIR /app
COPY . .
RUN ./mvnw package -DskipTests
CMD ["java", "-jar", "target/service.jar"]
```

**Problems:** ~600 MB image, includes Maven, source code, test classes, and full JDK.

### Optimised Multi-Stage Dockerfile

```dockerfile
# ── Stage 1: Build ──────────────────────────────────────────────
FROM eclipse-temurin:21-jdk-alpine AS builder
WORKDIR /build

# Cache dependency layer separately from source
COPY pom.xml .
COPY .mvn .mvn
COPY mvnw .
RUN ./mvnw dependency:go-offline -q

COPY src src
RUN ./mvnw package -DskipTests -q

# Extract layered JAR (Spring Boot 2.3+)
RUN java -Djarmode=layertools -jar target/*.jar extract --destination extracted

# ── Stage 2: Runtime ─────────────────────────────────────────────
FROM eclipse-temurin:21-jre-alpine AS runtime
WORKDIR /app

# Non-root user for security
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser

# Copy layered JAR parts (most stable → least stable)
COPY --from=builder --chown=appuser:appgroup /build/extracted/dependencies/ ./
COPY --from=builder --chown=appuser:appgroup /build/extracted/spring-boot-loader/ ./
COPY --from=builder --chown=appuser:appgroup /build/extracted/snapshot-dependencies/ ./
COPY --from=builder --chown=appuser:appgroup /build/extracted/application/ ./

EXPOSE 8080
ENTRYPOINT ["java", "org.springframework.boot.loader.launch.JarLauncher"]
```

**Result:** ~180 MB vs ~600 MB. Re-deploying only app code changes the last layer only — dependencies layer (usually 100+ MB) is cached.

### Why Spring Boot Layered JARs?

Spring Boot's `layertools` splits the fat JAR into four ordered layers:

```
dependencies/           ← third-party libs (rarely change)
spring-boot-loader/     ← Spring Boot loader (rarely changes)
snapshot-dependencies/  ← SNAPSHOT versions (change occasionally)
application/            ← your compiled classes (change every build)
```

This maximises Docker layer cache reuse between deployments.

---

## 4. Image Optimisation Techniques

### Technique 1: Use Alpine or Distroless

```dockerfile
# Alpine (~5 MB OS)
FROM eclipse-temurin:21-jre-alpine

# Distroless (no shell, minimal attack surface)
FROM gcr.io/distroless/java21-debian12
```

| Base Image | Size | Shell | Use Case |
|------------|------|-------|----------|
| `eclipse-temurin:21-jdk` | ~470 MB | Yes | Build stage only |
| `eclipse-temurin:21-jre-alpine` | ~190 MB | sh | Dev/staging |
| `distroless/java21` | ~125 MB | No | Production |

### Technique 2: Minimise Layers

```dockerfile
# Bad: 3 layers
RUN apt-get update
RUN apt-get install -y curl
RUN rm -rf /var/lib/apt/lists/*

# Good: 1 layer
RUN apt-get update && apt-get install -y curl && rm -rf /var/lib/apt/lists/*
```

### Technique 3: .dockerignore

```
.git
target/
*.iml
.idea/
.mvn/wrapper/maven-wrapper.jar
**/*.md
**/__pycache__
```

### Technique 4: BuildKit Cache Mounts (Maven)

```dockerfile
RUN --mount=type=cache,target=/root/.m2 \
    ./mvnw package -DskipTests -q
```

Maven local repository is cached across builds on the same host — `dependency:go-offline` no longer needed.

---

## 5. Docker Compose for Microservices

A realistic `docker-compose.yml` for local development of a Spring Boot microservice stack:

```yaml
version: "3.9"

services:
  # ── Infrastructure ─────────────────────────────────────────────
  mysql:
    image: mysql:8.0
    environment:
      MYSQL_ROOT_PASSWORD: root
      MYSQL_DATABASE: orders_db
    ports: ["3306:3306"]
    volumes:
      - mysql_data:/var/lib/mysql
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost"]
      interval: 10s
      retries: 5

  redis:
    image: redis:7-alpine
    ports: ["6379:6379"]

  kafka:
    image: confluentinc/cp-kafka:7.6.0
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: PLAINTEXT:PLAINTEXT
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka:9092
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
      KAFKA_AUTO_CREATE_TOPICS_ENABLE: "true"
      CLUSTER_ID: "MkU3OEVBNTcwNTJENDM2Qk"
    ports: ["9092:9092"]
    depends_on:
      - zookeeper

  zookeeper:
    image: confluentinc/cp-zookeeper:7.6.0
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181

  # ── Services ────────────────────────────────────────────────────
  order-service:
    build:
      context: ./order-service
      target: runtime          # use runtime stage from multi-stage build
    environment:
      SPRING_DATASOURCE_URL: jdbc:mysql://mysql:3306/orders_db
      SPRING_KAFKA_BOOTSTRAP_SERVERS: kafka:9092
      SPRING_REDIS_HOST: redis
      SPRING_PROFILES_ACTIVE: docker
    ports: ["8081:8080"]
    depends_on:
      mysql:
        condition: service_healthy
      kafka:
        condition: service_started
    deploy:
      resources:
        limits:
          memory: 512m
          cpus: "0.5"

  notification-service:
    build:
      context: ./notification-service
      target: runtime
    environment:
      SPRING_KAFKA_BOOTSTRAP_SERVERS: kafka:9092
      SPRING_PROFILES_ACTIVE: docker
    ports: ["8082:8080"]
    depends_on:
      - kafka

volumes:
  mysql_data:
```

**Key patterns:**
- `condition: service_healthy` — waits for MySQL readiness check before starting app
- `target: runtime` — references named stage in multi-stage Dockerfile
- `deploy.resources.limits` — mirrors K8s resource limits for local parity
- `SPRING_PROFILES_ACTIVE: docker` — overrides `application.yml` with `application-docker.yml`

### Spring Boot `application-docker.yml`

```yaml
spring:
  datasource:
    url: jdbc:mysql://mysql:3306/orders_db
    username: root
    password: root
  kafka:
    bootstrap-servers: kafka:9092
  data:
    redis:
      host: redis

management:
  endpoints:
    web:
      exposure:
        include: health,info,prometheus
```

---

## 6. Docker Security Best Practices

### 1. Never Run as Root

```dockerfile
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser
```

### 2. Read-Only Filesystem

```yaml
# docker-compose.yml
services:
  order-service:
    read_only: true
    tmpfs:
      - /tmp            # Spring Boot temp files
      - /app/logs       # log output
```

### 3. Scan Images for CVEs

```bash
# Trivy (best-in-class, used in CI/CD)
trivy image --severity HIGH,CRITICAL myapp:latest

# Docker Scout (built into Docker Desktop)
docker scout cves myapp:latest
```

### 4. Sign Images with Docker Content Trust

```bash
export DOCKER_CONTENT_TRUST=1
docker push myregistry.azurecr.io/order-service:v1.2.0
```

### 5. Limit Capabilities

```yaml
# docker-compose.yml
security_opt:
  - no-new-privileges:true
cap_drop:
  - ALL
cap_add:
  - NET_BIND_SERVICE   # only if binding port < 1024
```

### 6. Use Specific Image Tags (Never `latest`)

```dockerfile
# Bad
FROM eclipse-temurin:latest

# Good
FROM eclipse-temurin:21.0.3_9-jre-alpine@sha256:abc123...
```

---

## 7. Useful Docker Commands for Daily Work

```bash
# Build with BuildKit enabled (faster, cache mounts)
DOCKER_BUILDKIT=1 docker build -t order-service:dev .

# Inspect layers and sizes
docker history order-service:dev

# Dive into image layer contents (install: brew install dive)
dive order-service:dev

# Check container resource usage
docker stats

# Exec into running container
docker exec -it <container_id> sh

# Copy files out of container (for debugging)
docker cp <container_id>:/app/logs/app.log ./

# Prune unused images/containers/volumes
docker system prune -af --volumes

# Multi-platform build (for ARM/x86 CI environments)
docker buildx build --platform linux/amd64,linux/arm64 -t order-service:v1 --push .
```

---

## 8. CI/CD Pipeline Integration (GitHub Actions)

```yaml
# .github/workflows/docker-build.yml
name: Build & Push Docker Image

on:
  push:
    branches: [main]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Login to Azure Container Registry
        uses: docker/login-action@v3
        with:
          registry: myregistry.azurecr.io
          username: ${{ secrets.ACR_USERNAME }}
          password: ${{ secrets.ACR_PASSWORD }}

      - name: Build and push
        uses: docker/build-push-action@v5
        with:
          context: .
          target: runtime
          push: true
          tags: myregistry.azurecr.io/order-service:${{ github.sha }}
          cache-from: type=gha
          cache-to: type=gha,mode=max    # GitHub Actions cache for layers
```

---

## 9. Interview Questions

1. **What are Docker namespaces and how do they provide isolation?** Name the six namespace types.
2. **Explain the difference between an image layer and a container layer.** Why are image layers read-only?
3. **Why use multi-stage builds?** What problem does the layered JAR approach solve specifically?
4. **What is the difference between `COPY` and `ADD`?** When should you use each?
5. **How does Docker's UnionFS (OverlayFS) work?** What happens when a container writes to a file that exists in an image layer?
6. **What security risks exist when running containers as root?** How do you mitigate them?
7. **Explain `depends_on` vs health checks in Docker Compose.** Why is `condition: service_healthy` important for Spring Boot apps?
8. **How does BuildKit's `--mount=type=cache` improve build performance?**
9. **What is a distroless image?** When would you choose it over Alpine?
10. **How do you handle secrets in Docker?** What's wrong with `ENV` for secrets?

---

## 10. Today's Practice Exercise

**Task:** Optimise the Dockerfile for a Spring Boot order service.

**Starting point:** An existing fat-JAR Dockerfile (600 MB image, runs as root, no layer caching).

**Steps:**
1. Create a multi-stage Dockerfile using `eclipse-temurin:21-jdk-alpine` for build and `eclipse-temurin:21-jre-alpine` for runtime
2. Enable Spring Boot layered JAR extraction
3. Add a non-root user
4. Create `.dockerignore` excluding `target/`, `.git/`, `.idea/`
5. Write a `docker-compose.yml` connecting the service to MySQL and Kafka with health checks
6. Add resource limits (`memory: 512m`, `cpus: 0.5`)
7. Measure image size before and after optimisation with `docker images`
8. Run `docker history` to verify layer structure

**Expected outcome:** Image size reduced from ~600 MB to ~180 MB, last layer contains only your compiled classes.

---

## 11. Key Takeaways

- Docker isolation relies on Linux **namespaces** (process/network/filesystem isolation) and **cgroups** (resource limits) — understanding this helps debug container networking and OOM issues
- **Multi-stage builds** are non-negotiable for production: build stage uses full JDK, runtime stage uses JRE-only Alpine (~3× size reduction)
- **Spring Boot layered JARs** maximise Docker layer cache reuse — only the `application/` layer changes on each code push, saving bandwidth and reducing pull time in K8s
- **Never run containers as root** — use `adduser` in Dockerfile, enforce `no-new-privileges`, and drop all capabilities
- **Docker Compose** local stacks should mirror K8s resource constraints and use health checks to avoid race conditions between services
- Always pin image tags to digests (`@sha256:...`) in production to prevent supply chain attacks from mutable tags

---

## 12. Resources

- [Docker Official Docs — Multi-stage builds](https://docs.docker.com/build/building/multi-stage/)
- [Spring Boot Docker best practices](https://spring.io/guides/gs/spring-boot-docker/)
- [BuildKit cache mounts](https://docs.docker.com/build/cache/optimize/)
- [Trivy container scanner](https://trivy.dev/)
- [Google Distroless images](https://github.com/GoogleContainerTools/distroless)
- [Docker Compose health checks](https://docs.docker.com/compose/how-tos/startup-order/)

---

*Next: Day 41 — Kubernetes Core (Pods, Deployments, Services, ConfigMaps, Probes)*
