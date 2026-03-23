# 05 — Gemini for Cloud and DevOps

## Table of Contents

1. [Gemini in Google Cloud Console](#1-gemini-in-google-cloud-console)
2. [Gemini for GKE](#2-gemini-for-gke)
3. [Gemini for Cloud Logging and Monitoring](#3-gemini-for-cloud-logging-and-monitoring)
4. [Gemini for BigQuery](#4-gemini-for-bigquery)
5. [Gemini for Cloud Security](#5-gemini-for-cloud-security)
6. [Gemini Cloud Assist](#6-gemini-cloud-assist)
7. [Using Gemini with Terraform for GCP](#7-using-gemini-with-terraform-for-gcp)
8. [Practical Examples](#8-practical-examples)
9. [Azure Comparison and Transferable Concepts](#9-azure-comparison-and-transferable-concepts)
10. [Try This Exercises](#10-try-this-exercises)

---

## 1. Gemini in Google Cloud Console

Gemini is embedded throughout the Google Cloud Console as an AI assistant. Look for
the sparkle icon or the "Gemini" chat panel on the right side of the console.

### What It Can Do

| Capability | Example |
|------------|---------|
| Natural language queries | "Show me all VMs in us-central1 with > 80% CPU" |
| Explain services | "What is Cloud Run and when should I use it vs GKE?" |
| Generate commands | "Give me the gcloud command to create a GKE Autopilot cluster" |
| Troubleshoot errors | "Why is my Cloud Function returning 500 errors?" |
| Explain IAM | "Who has access to this Cloud Storage bucket?" |
| Cost analysis | "Why did my bill increase last month?" |
| Architecture advice | "What GCP services should I use for a real-time data pipeline?" |

### Accessing Gemini in Console

```
1. Open Google Cloud Console (https://console.cloud.google.com)
2. Click the Gemini sparkle icon (✨) in the top toolbar
3. A chat panel opens on the right side
4. Type your question in natural language
5. Gemini responds with explanations, commands, and links to docs
```

### Context Awareness

Gemini in the console is context-aware:
- **On the GKE page:** it knows you are asking about Kubernetes
- **On the IAM page:** it understands you are asking about permissions
- **On a VM page:** it can see the VM's configuration

```
Example:
[Viewing a specific GKE cluster page]
You: "Why are my pods not scheduling?"
Gemini: "Looking at your cluster, I see:
1. Node pool 'default' has 3 nodes, all at 95%+ CPU utilization
2. Your pending pods request 2 CPU cores each
3. Your cluster autoscaler max is set to 3 nodes

Recommendation: Increase the autoscaler max to 5 nodes or reduce
pod resource requests. Here's the command:
gcloud container clusters update my-cluster --max-nodes=5 ..."
```

---

## 2. Gemini for GKE

### Kubernetes Troubleshooting

Gemini in the GKE console can diagnose common issues:

| Issue | How Gemini Helps |
|-------|-----------------|
| CrashLoopBackOff | Analyzes container logs, identifies crash reason |
| ImagePullBackOff | Checks image name, registry permissions, network |
| Pending pods | Checks resource requests vs available capacity |
| OOMKilled | Analyzes memory usage patterns, suggests limits |
| Network issues | Explains NetworkPolicy effects, service routing |
| Certificate errors | Identifies TLS misconfigurations |

### Manifest Generation

Ask Gemini to generate Kubernetes manifests:

```
Prompt: "Generate a Kubernetes deployment for a Spring Boot app with:
- 3 replicas
- Rolling update strategy (25% max unavailable)
- Resource limits: 512Mi memory, 500m CPU
- Liveness probe on /actuator/health/liveness
- Readiness probe on /actuator/health/readiness
- Environment variables from a ConfigMap 'app-config'
- Secrets from a Secret 'app-secrets'
- Pod disruption budget (minAvailable: 2)
- Horizontal pod autoscaler (CPU 70%, min 3, max 10)"
```

**Gemini generates:**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
  labels:
    app: order-service
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 25%
      maxSurge: 25%
  selector:
    matchLabels:
      app: order-service
  template:
    metadata:
      labels:
        app: order-service
    spec:
      containers:
        - name: order-service
          image: gcr.io/my-project/order-service:latest
          ports:
            - containerPort: 8080
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
            failureThreshold: 3
          readinessProbe:
            httpGet:
              path: /actuator/health/readiness
              port: 8080
            initialDelaySeconds: 15
            periodSeconds: 5
          envFrom:
            - configMapRef:
                name: app-config
            - secretRef:
                name: app-secrets
---
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: order-service-pdb
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: order-service
---
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
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
```

### GKE Diagnostic Commands via Gemini

Ask Gemini for diagnostic commands:

```
You: "My Spring Boot app on GKE is throwing Connection refused to MySQL.
     Give me diagnostic commands."

Gemini: "Here are the commands to diagnose MySQL connectivity from GKE:

# 1. Check if the pod can reach MySQL
kubectl exec -it <pod-name> -- nc -zv mysql-host 3306

# 2. Check DNS resolution
kubectl exec -it <pod-name> -- nslookup mysql-host

# 3. Check if Cloud SQL Auth Proxy is running (if using Cloud SQL)
kubectl get pods -l app=cloud-sql-proxy

# 4. Check network policy
kubectl get networkpolicy -A | grep mysql

# 5. Check service account permissions for Cloud SQL
gcloud projects get-iam-policy $PROJECT_ID \
  --filter='bindings.members:serviceAccount:gke-sa@*' \
  --flatten='bindings[].members'

# 6. Check pod logs for connection details
kubectl logs <pod-name> --tail=50 | grep -i 'connection\|mysql\|refused'
"
```

---

## 3. Gemini for Cloud Logging and Monitoring

### Log Exploration

In Cloud Logging (https://console.cloud.google.com/logs), Gemini can:

| Feature | Description |
|---------|-------------|
| Explain log entries | "What does this error mean?" — click the Gemini icon on any log line |
| Generate queries | "Show me all ERROR logs from order-service in the last hour" |
| Summarize logs | "Summarize the most common errors in the last 24 hours" |
| Suggest fixes | "This log shows a connection timeout — here's how to fix it" |

### Natural Language to Logging Query

Instead of writing complex logging queries manually:

```
You: "Show me all 5xx errors from my Spring Boot services in the last 6 hours,
     grouped by service name"

Gemini generates:
resource.type="k8s_container"
resource.labels.cluster_name="my-cluster"
jsonPayload.severity >= "ERROR"
httpRequest.status >= 500
timestamp >= "2026-03-23T00:00:00Z"

-- Or as a Logs Explorer query:
severity >= ERROR
httpRequest.status >= 500
resource.labels.container_name =~ ".*-service"
```

### Cloud Monitoring Integration

In Cloud Monitoring, Gemini helps with:

| Task | How |
|------|-----|
| Create alerts | "Alert me when error rate exceeds 1% for 5 minutes" |
| Build dashboards | "Create a dashboard for my Spring Boot services with latency, errors, and throughput" |
| Explain metrics | "What does container/cpu/core_usage_time mean?" |
| Correlate issues | "My latency spiked at 2 PM — what changed?" |

### Example: Explain an Error

Click the Gemini icon next to a log entry:

```
Log entry:
java.lang.OutOfMemoryError: Java heap space
    at java.util.Arrays.copyOf(Arrays.java:3210)
    at org.hibernate.engine.spi.ActionQueue.executeActions(ActionQueue.java:604)
    at com.example.service.BatchProcessor.processBatch(BatchProcessor.java:45)

Gemini explains:
"This OutOfMemoryError occurs in your BatchProcessor class. The Hibernate
ActionQueue is trying to flush a large number of entities at once. This is
likely because:

1. You're loading a large batch without pagination
2. The Hibernate session is holding too many entities in its first-level cache

Fix:
- Process in smaller batches (e.g., 100 at a time)
- Call entityManager.flush() and entityManager.clear() between batches
- Add @Transactional with read-only where possible
- Consider using StatelessSession for batch operations
- Set spring.jpa.properties.hibernate.jdbc.batch_size=50"
```

---

## 4. Gemini for BigQuery

### SQL Generation

Gemini in the BigQuery console generates SQL from natural language:

```
You: "Find all customers who placed more than 5 orders in the last 30 days,
     with total spend over $500, sorted by total spend descending"

Gemini generates:
SELECT
  c.customer_id,
  c.email,
  c.name,
  COUNT(o.order_id) AS order_count,
  SUM(o.total_amount) AS total_spend
FROM `project.dataset.customers` c
JOIN `project.dataset.orders` o
  ON c.customer_id = o.customer_id
WHERE o.created_at >= DATE_SUB(CURRENT_DATE(), INTERVAL 30 DAY)
GROUP BY c.customer_id, c.email, c.name
HAVING
  COUNT(o.order_id) > 5
  AND SUM(o.total_amount) > 500
ORDER BY total_spend DESC;
```

### Data Analysis

| Task | Example Prompt |
|------|----------------|
| Anomaly detection | "Find any unusual spikes in order volume this week" |
| Schema explanation | "Explain the relationships between these tables" |
| Query optimization | "This query takes 30 seconds — how can I optimize it?" |
| Data quality | "Find rows with NULL values or inconsistent data in the orders table" |

### Practical Use for Backend Devs

```
You: "Generate a BigQuery query to analyze API latency by endpoint.
     The data is in a table called 'api_logs' with columns:
     timestamp, endpoint, method, status_code, latency_ms, user_id"

Gemini:
-- P50, P95, P99 latency by endpoint
SELECT
  endpoint,
  method,
  COUNT(*) AS request_count,
  APPROX_QUANTILES(latency_ms, 100)[OFFSET(50)] AS p50_ms,
  APPROX_QUANTILES(latency_ms, 100)[OFFSET(95)] AS p95_ms,
  APPROX_QUANTILES(latency_ms, 100)[OFFSET(99)] AS p99_ms,
  AVG(latency_ms) AS avg_ms,
  COUNTIF(status_code >= 500) AS error_count,
  COUNTIF(status_code >= 500) / COUNT(*) * 100 AS error_rate_pct
FROM `project.dataset.api_logs`
WHERE timestamp >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 24 HOUR)
GROUP BY endpoint, method
ORDER BY p99_ms DESC;
```

---

## 5. Gemini for Cloud Security

### Security Command Center Integration

Gemini helps analyze security findings in Security Command Center (SCC):

| Feature | Description |
|---------|-------------|
| Explain findings | "What does this 'PUBLIC_BUCKET_ACL' finding mean?" |
| Suggest remediation | "How do I fix this IAM overprivilege finding?" |
| Risk assessment | "How critical is this vulnerability?" |
| Attack path analysis | "Show me the attack path for this exposed service" |

### IAM Analysis

```
You: "Who can deploy to my production GKE cluster?"

Gemini: "Analyzing IAM policies for cluster 'prod-cluster':

1. Service accounts with container.admin:
   - deploy-sa@project.iam.gserviceaccount.com (CI/CD)
   - admin@project.iam.gserviceaccount.com (overprivileged — review)

2. Users with container.developer:
   - alice@company.com
   - bob@company.com

3. Groups:
   - devops@company.com (5 members) — has container.clusterAdmin

Recommendations:
- admin@ SA has project-level Owner role — reduce to container.admin
- Consider using Workload Identity instead of key-based SAs
- Enable Binary Authorization for production deployments"
```

### Vulnerability Detection

```
You: "Scan my GKE workloads for known vulnerabilities"

Gemini: "I found the following vulnerabilities in your cluster:

Critical:
- order-service:v2.1 → CVE-2024-22233 (Spring Framework RCE)
  Fix: Upgrade spring-framework to 6.1.4+

High:
- base image eclipse-temurin:17 → 3 CVEs in glibc
  Fix: Update to eclipse-temurin:17.0.10_7

Medium:
- user-service:v1.8 → CVE-2024-29025 (Netty HTTP/2 DoS)
  Fix: Upgrade netty to 4.1.108+

Run this command to see full scan results:
gcloud artifacts docker images scan gcr.io/project/order-service:v2.1"
```

---

## 6. Gemini Cloud Assist

Gemini Cloud Assist is a specialized assistant for infrastructure management:

### Capabilities

| Area | What It Does |
|------|-------------|
| **Design** | Recommend architectures, suggest services |
| **Operate** | Troubleshoot issues, explain errors |
| **Optimize** | Cost recommendations, performance tuning |
| **Secure** | IAM review, compliance checks |

### Infrastructure Recommendations

```
You: "I need to deploy a Spring Boot microservice that processes 10,000
     requests/second with sub-100ms latency. What GCP services should I use?"

Gemini Cloud Assist:
"For 10K RPS with sub-100ms latency, I recommend:

1. Compute: GKE Autopilot
   - Auto-scales pods and nodes
   - No cluster management overhead
   - Configure HPA based on custom metrics (latency)

2. Database: Cloud SQL for MySQL (if relational)
   - High availability with regional instance
   - Read replicas for read-heavy workloads
   - OR: Cloud Spanner for unlimited scale

3. Caching: Memorystore for Redis
   - Reduce DB load for hot data
   - Session storage
   - Rate limiting

4. Load Balancing: Cloud CDN + Global L7 LB
   - SSL termination
   - Connection draining

5. Messaging: Pub/Sub (instead of self-managed Kafka)
   - Fully managed
   - At-least-once delivery
   - Dead-letter topics

Estimated cost: $800-1,200/month for this workload

Here's a Terraform template to get started: [generates Terraform]"
```

---

## 7. Using Gemini with Terraform for GCP

### Generating Terraform with Gemini

Use AI Studio or Gemini Code Assist to generate Terraform for GCP resources:

```
Prompt: "Generate Terraform for a production GKE setup:
- Autopilot cluster in us-central1
- Private cluster with master authorized networks
- Workload Identity enabled
- Cloud NAT for outbound internet
- Cloud SQL MySQL 8.0 with high availability
- Memorystore Redis (6GB)
- Cloud Armor WAF policy
- All in a custom VPC with private subnets"
```

**Gemini generates:**

```hcl
# VPC
resource "google_compute_network" "main" {
  name                    = "prod-vpc"
  auto_create_subnetworks = false
}

resource "google_compute_subnetwork" "gke" {
  name          = "gke-subnet"
  ip_cidr_range = "10.0.0.0/20"
  region        = "us-central1"
  network       = google_compute_network.main.id

  secondary_ip_range {
    range_name    = "pods"
    ip_cidr_range = "10.4.0.0/14"
  }
  secondary_ip_range {
    range_name    = "services"
    ip_cidr_range = "10.8.0.0/20"
  }

  private_ip_google_access = true
}

# GKE Autopilot
resource "google_container_cluster" "primary" {
  name     = "prod-cluster"
  location = "us-central1"

  enable_autopilot = true

  network    = google_compute_network.main.name
  subnetwork = google_compute_subnetwork.gke.name

  ip_allocation_policy {
    cluster_secondary_range_name  = "pods"
    services_secondary_range_name = "services"
  }

  private_cluster_config {
    enable_private_nodes    = true
    enable_private_endpoint = false
    master_ipv4_cidr_block  = "172.16.0.0/28"
  }

  master_authorized_networks_config {
    cidr_blocks {
      cidr_block   = "10.0.0.0/8"
      display_name = "internal"
    }
  }

  workload_identity_config {
    workload_pool = "${var.project_id}.svc.id.goog"
  }

  release_channel {
    channel = "REGULAR"
  }
}

# Cloud NAT
resource "google_compute_router" "nat_router" {
  name    = "nat-router"
  region  = "us-central1"
  network = google_compute_network.main.id
}

resource "google_compute_router_nat" "nat" {
  name                               = "prod-nat"
  router                             = google_compute_router.nat_router.name
  region                             = "us-central1"
  nat_ip_allocate_option             = "AUTO_ONLY"
  source_subnetwork_ip_ranges_to_nat = "ALL_SUBNETWORKS_ALL_IP_RANGES"
}

# Cloud SQL MySQL
resource "google_sql_database_instance" "mysql" {
  name             = "prod-mysql"
  database_version = "MYSQL_8_0"
  region           = "us-central1"

  settings {
    tier              = "db-n1-standard-4"
    availability_type = "REGIONAL"

    backup_configuration {
      enabled            = true
      binary_log_enabled = true
      start_time         = "02:00"
    }

    ip_configuration {
      ipv4_enabled    = false
      private_network = google_compute_network.main.id
    }

    database_flags {
      name  = "slow_query_log"
      value = "on"
    }

    maintenance_window {
      day  = 7  # Sunday
      hour = 3
    }
  }

  deletion_protection = true
}

# Memorystore Redis
resource "google_redis_instance" "cache" {
  name           = "prod-redis"
  tier           = "STANDARD_HA"
  memory_size_gb = 6
  region         = "us-central1"

  authorized_network = google_compute_network.main.id
  connect_mode       = "PRIVATE_SERVICE_ACCESS"

  redis_version = "REDIS_7_0"

  maintenance_policy {
    weekly_maintenance_window {
      day = "SUNDAY"
      start_time {
        hours   = 3
        minutes = 0
      }
    }
  }
}

# Cloud Armor
resource "google_compute_security_policy" "waf" {
  name = "prod-waf"

  rule {
    action   = "deny(403)"
    priority = 1000
    match {
      expr {
        expression = "evaluatePreconfiguredExpr('sqli-v33-stable')"
      }
    }
    description = "Block SQL injection"
  }

  rule {
    action   = "deny(403)"
    priority = 1001
    match {
      expr {
        expression = "evaluatePreconfiguredExpr('xss-v33-stable')"
      }
    }
    description = "Block XSS"
  }

  rule {
    action   = "allow"
    priority = 2147483647
    match {
      versioned_expr = "SRC_IPS_V1"
      config {
        src_ip_ranges = ["*"]
      }
    }
    description = "Default allow"
  }
}
```

### Terraform Troubleshooting with Gemini

```
You: "My terraform apply is failing with this error:
     Error: Error creating Instance: googleapi: Error 403:
     Compute Engine API has not been activated"

Gemini: "This error means the Compute Engine API isn't enabled.
Run: gcloud services enable compute.googleapis.com --project=YOUR_PROJECT

Also ensure these APIs are enabled for GKE + Cloud SQL:
gcloud services enable container.googleapis.com
gcloud services enable sqladmin.googleapis.com
gcloud services enable redis.googleapis.com
gcloud services enable servicenetworking.googleapis.com"
```

---

## 8. Practical Examples

### Example 1: Troubleshoot a GKE Deployment

```
Scenario: Your Spring Boot app deploys but pods keep crashing.

Step 1: Open GKE console → click your cluster → Workloads
Step 2: Click the failing deployment → Events tab
Step 3: Click Gemini icon → "Why are my pods crashing?"

Gemini analyzes:
- Pod events (CrashLoopBackOff, OOMKilled, etc.)
- Container logs (startup errors, exception traces)
- Resource usage (memory exceeding limits)
- Probe failures (liveness/readiness)

And provides targeted fix recommendations.
```

### Example 2: Generate Cloud SQL Queries for Monitoring

```
You: "Write MySQL queries to check the health of my production database:
     connection count, slow queries, table sizes, replication lag"

Gemini:
-- Active connections
SELECT user, host, COUNT(*) as connections
FROM information_schema.processlist
GROUP BY user, host
ORDER BY connections DESC;

-- Slow queries in the last hour
SELECT * FROM mysql.slow_log
WHERE start_time > NOW() - INTERVAL 1 HOUR
ORDER BY query_time DESC
LIMIT 20;

-- Top 20 tables by size
SELECT
  table_schema AS 'Database',
  table_name AS 'Table',
  ROUND(data_length / 1024 / 1024, 2) AS 'Data (MB)',
  ROUND(index_length / 1024 / 1024, 2) AS 'Index (MB)',
  ROUND((data_length + index_length) / 1024 / 1024, 2) AS 'Total (MB)',
  table_rows AS 'Rows'
FROM information_schema.tables
WHERE table_schema NOT IN ('mysql', 'information_schema', 'performance_schema', 'sys')
ORDER BY (data_length + index_length) DESC
LIMIT 20;

-- InnoDB buffer pool hit ratio
SELECT
  (1 - (Innodb_buffer_pool_reads / Innodb_buffer_pool_read_requests)) * 100 AS hit_ratio
FROM (
  SELECT
    VARIABLE_VALUE AS Innodb_buffer_pool_reads
  FROM performance_schema.global_status
  WHERE VARIABLE_NAME = 'Innodb_buffer_pool_reads'
) a, (
  SELECT
    VARIABLE_VALUE AS Innodb_buffer_pool_read_requests
  FROM performance_schema.global_status
  WHERE VARIABLE_NAME = 'Innodb_buffer_pool_read_requests'
) b;
```

### Example 3: Incident Response with Gemini

```
Scenario: Production alert — 5xx errors spiking on order-service.

Step 1: Open Cloud Logging
  Prompt: "Show me all ERROR logs for order-service in the last 30 minutes"

Step 2: Click Gemini on a representative error log
  Gemini: "This is a ConnectionTimeoutException to your MySQL instance.
           The connection pool is exhausted."

Step 3: Open Cloud Monitoring
  Prompt: "Show me order-service database connection metrics"
  → See all connections at max, waiting queue growing

Step 4: Ask Gemini for fix
  "My HikariCP pool is exhausted (max 20). During peak traffic we need more.
   How do I fix this without redeploying?"

  Gemini: "Immediate fix — scale the deployment to spread connections:
  kubectl scale deployment order-service --replicas=6

  Then update your config to increase pool and reduce timeout:
  spring.datasource.hikari.maximum-pool-size=30
  spring.datasource.hikari.connection-timeout=5000
  spring.datasource.hikari.max-lifetime=1800000

  Long-term: Add connection pooling with PgBouncer/ProxySQL
  or switch to Cloud SQL Auth Proxy with connection limits"
```

---

## 9. Azure Comparison and Transferable Concepts

Since you primarily use Azure, here is how GCP Gemini features map to Azure equivalents:

### Service Mapping

| GCP + Gemini | Azure Equivalent | Notes |
|-------------|------------------|-------|
| Gemini in Cloud Console | Azure Copilot in Portal | Both provide NL queries for cloud resources |
| Gemini for GKE | Azure Copilot for AKS | Kubernetes troubleshooting |
| Cloud Logging + Gemini | Azure Monitor + Copilot | Log analysis, query generation |
| Cloud Monitoring + Gemini | Application Insights + Copilot | Metrics, alerts, dashboards |
| BigQuery + Gemini | Azure Synapse + Copilot | SQL generation, data analysis |
| Security Command Center + Gemini | Microsoft Defender for Cloud + Copilot | Security findings, remediation |
| Gemini Cloud Assist | Azure Advisor + Copilot | Recommendations, optimization |
| Vertex AI | Azure AI Studio / Azure OpenAI | LLM hosting, tuning, evaluation |
| Vertex AI Agent Builder | Azure AI Agent Service | Build AI agents with tools + data |
| Vertex AI Search | Azure AI Search | Managed search + RAG |
| Cloud SQL + Gemini | Azure SQL + Copilot | Database management |
| Terraform + Gemini | Terraform + Copilot | IaC generation and troubleshooting |

### Transferable Concepts

These patterns work identically regardless of cloud provider:

1. **NL-to-query translation** — Works for Cloud Logging queries, KQL (Azure), and SQL
2. **Incident troubleshooting** — Same diagnostic approach: logs → metrics → traces → fix
3. **IaC generation** — Gemini generates Terraform for GCP; Copilot does the same for Azure
4. **Security posture analysis** — IAM review, vulnerability scanning, compliance checks
5. **Cost optimization** — Both platforms have AI-powered cost recommendations
6. **Kubernetes management** — GKE and AKS troubleshooting follows the same patterns

### Multi-Cloud Strategy

```
Your stack (Azure primary, GCP for AI):

Azure (primary):
├── AKS (Kubernetes)
├── Azure SQL / Cosmos DB
├── Azure Service Bus (messaging)
├── Azure Monitor + App Insights
├── Azure DevOps / GitHub Actions
└── Azure Copilot (portal assistant)

GCP (for AI workloads):
├── Vertex AI (Gemini models)
├── Vertex AI Search (managed RAG)
├── Vertex AI Agent Builder (AI agents)
└── BigQuery (analytics, optional)

Your Spring Boot services:
├── Deploy on AKS (Azure)
├── Call Vertex AI Gemini API for AI features
├── Use Spring AI for provider abstraction
└── Terraform manages both clouds
```

**Cross-cloud connection:**
```hcl
# Terraform: Connect Azure AKS to GCP Vertex AI
# Option 1: API key (simple, for dev)
resource "kubernetes_secret" "gemini_api_key" {
  metadata {
    name      = "gemini-credentials"
    namespace = "default"
  }
  data = {
    api-key = var.gemini_api_key
  }
}

# Option 2: Workload Identity Federation (production)
# Your Azure-hosted service authenticates to GCP using Azure AD tokens
resource "google_iam_workload_identity_pool" "azure" {
  workload_identity_pool_id = "azure-pool"
}

resource "google_iam_workload_identity_pool_provider" "azure" {
  workload_identity_pool_id          = google_iam_workload_identity_pool.azure.workload_identity_pool_id
  workload_identity_pool_provider_id = "azure-provider"
  attribute_mapping = {
    "google.subject" = "assertion.sub"
  }
  oidc {
    issuer_uri = "https://sts.windows.net/${var.azure_tenant_id}/"
  }
}
```

---

## 10. Try This Exercises

### Exercise 1: Explore Cloud Console Gemini (15 min)
1. Open Google Cloud Console
2. Click the Gemini icon in the toolbar
3. Ask: "What services are running in this project?"
4. Navigate to GKE → ask "How do I create an Autopilot cluster?"
5. Navigate to Cloud SQL → ask "What's the recommended configuration for a production MySQL instance?"

### Exercise 2: Generate Terraform (20 min)
1. In AI Studio or Gemini Code Assist, ask Gemini to generate Terraform for:
   - A VPC with private subnet
   - A GKE Autopilot cluster
   - A Cloud SQL MySQL instance
2. Review the generated code for best practices
3. Compare with your Azure Terraform patterns — note similarities

### Exercise 3: Log Analysis (15 min)
1. If you have a GCP project with logs, open Cloud Logging
2. Use Gemini to write a query: "Show me all WARNING and ERROR logs from the last 24 hours"
3. Click the Gemini icon on a log entry to get an explanation
4. Compare: is this faster than manually reading the logs?

### Exercise 4: Kubernetes Manifest Generation (15 min)
1. Describe your most complex Kubernetes deployment to Gemini
2. Ask it to generate the full set of manifests (Deployment, Service, HPA, PDB, NetworkPolicy)
3. Compare with your existing manifests — what did Gemini add that you missed?
4. Ask Gemini to add Istio VirtualService configuration

### Exercise 5: Cross-Cloud Architecture (20 min)
1. Ask Gemini: "Design an architecture where Spring Boot services on Azure AKS
   call Gemini models on GCP Vertex AI. Include authentication, network security,
   and cost optimization."
2. Review the architecture
3. Ask follow-up: "Generate Terraform for the GCP side of this architecture"
4. Compare the authentication options: API key vs Workload Identity Federation

---

## Key Takeaways

1. **Gemini is embedded throughout GCP Console** — not a separate tool, it is context-aware in each service
2. **GKE troubleshooting** is where Gemini adds the most value for K8s users
3. **BigQuery SQL generation** saves significant time for data queries
4. **Cloud Logging + Gemini** turns log analysis from manual scanning to NL queries
5. **Security findings** become actionable with Gemini's remediation suggestions
6. **Terraform generation** works well for GCP resources — review before applying
7. **Azure users:** The concepts transfer directly — NL-to-query, troubleshooting, IaC generation
8. **Multi-cloud:** Use Workload Identity Federation to connect Azure workloads to GCP Vertex AI

---

Next: [06 — Gemini Advanced Features](06-gemini-advanced-features.md)
