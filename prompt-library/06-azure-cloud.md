# Azure Cloud — Daily Prompts

> Tech stack: AKS, Azure Blob Storage, Azure Key Vault, Azure App Configuration, Azure MySQL, Managed Identity, Azure Monitor

---

## Architecture & Design

### Design an Azure architecture
```
Design an Azure architecture for [application/feature] with these requirements:
- [functional requirements]
- [non-functional: availability, latency, throughput, data residency]

Include:
1. Compute: AKS (existing) — namespace strategy, node pool sizing
2. Database: Azure MySQL Flexible Server — tier, HA, read replicas
3. Messaging: Confluent Kafka (managed) — integration pattern
4. Storage: Azure Blob Storage — tier (Hot/Cool/Archive), lifecycle policy
5. Caching: Azure Cache for Redis — tier, eviction policy
6. Secrets: Azure Key Vault — rotation policy
7. Config: Azure App Configuration — feature flags, dynamic refresh
8. Networking: VNet, Private Endpoints, NSGs, Azure Front Door/AGW
9. Identity: Managed Identity for all service-to-service auth
10. Monitoring: Azure Monitor, Application Insights, Log Analytics

Draw the architecture as a text-based diagram showing data flow.
Estimate monthly cost for [expected scale].
```

### Disaster recovery plan
```
Design a DR plan for my application running on Azure:
- AKS cluster in [region]
- Azure MySQL Flexible Server
- Azure Blob Storage
- Confluent Kafka (managed)

Requirements: RPO=[X hours], RTO=[Y hours]

Cover:
1. What to replicate and how (geo-redundant storage, read replicas, cross-region)
2. AKS: multi-region vs cold standby
3. MySQL: geo-redundant backup vs read replica in secondary region
4. Blob Storage: GRS vs RA-GRS
5. DNS failover with Azure Traffic Manager or Front Door
6. Runbook: step-by-step DR activation procedure
7. DR testing: how often, what to verify
8. Cost of DR setup (idle vs active-active)
```

---

## Azure Blob Storage

### Implement file upload/download
```
Implement Azure Blob Storage file upload and download in Spring Boot.

Requirements:
- Azure SDK for Java (azure-storage-blob)
- Authentication via Managed Identity (DefaultAzureCredential)
- Container per [tenant / feature / date]
- Upload: multipart file → Blob with metadata (uploadedBy, contentType, originalName)
- Download: generate SAS token URL (time-limited, read-only)
- List blobs with prefix filtering and pagination
- Delete with soft-delete enabled (recovery window)
- Max file size: [X MB], allowed types: [list]
- application.yml config for container names and connection

Show: BlobStorageService, Controller endpoint, config, and error handling.
```

### Blob lifecycle management
```
Set up Azure Blob Storage lifecycle management for [use case].

Policy:
- Move to Cool tier after [X] days
- Move to Archive after [Y] days
- Delete after [Z] days
- Apply only to blobs with prefix [path/] or tag [key=value]

Show:
1. Azure CLI command to create the policy
2. Terraform resource (azurerm_storage_management_policy)
3. How to verify the policy is working
4. Cost comparison: Hot vs Cool vs Archive for [storage volume]
```

---

## Azure Key Vault

### Integrate Key Vault with Spring Boot
```
Integrate Azure Key Vault with my Spring Boot app on AKS.

Approach: Azure Key Vault as a property source (secrets appear as Spring properties).

Show:
1. Maven dependency (azure-spring-cloud-starter-appconfiguration-config or
   azure-identity + Key Vault SDK)
2. Managed Identity setup (pod → ServiceAccount → Workload Identity → Key Vault)
3. Key Vault access policy or RBAC role assignment
4. application.yml configuration
5. Referencing secrets: spring.datasource.password=${kv-db-password}
6. Secret rotation: how the app picks up rotated secrets (restart vs refresh)
7. Local development: how to access Key Vault from local machine
   (az login + DefaultAzureCredential fallback)
```

---

## Azure App Configuration

### Dynamic configuration with feature flags
```
Set up Azure App Configuration with Spring Boot for dynamic config and feature flags.

Requirements:
- Spring Cloud Azure App Configuration
- Key-value pairs for: [list config keys]
- Feature flags for: [list features]
- Label-based environment separation (dev, staging, prod)
- Dynamic refresh without restart (@RefreshScope, polling interval)
- Sentinel key for triggering refresh
- Fallback to local application.yml if App Config is unreachable

Show: pom.xml dependencies, bootstrap.yml, application.yml, and a
@ConfigurationProperties class consuming the values.
```

---

## Azure Monitor & Observability

### Set up Application Insights
```
Integrate Azure Application Insights with my Spring Boot app on AKS.

Requirements:
- Auto-instrumentation via OTEL Java agent (applicationinsights-agent.jar)
- Custom metrics: [list business metrics, e.g., orders.processed, kafka.consumer.lag]
- Custom dimensions on traces: tenantId, userId, correlationId
- Dependency tracking: MySQL, Kafka, Blob Storage, external HTTP calls
- Live metrics stream
- Availability test for health endpoint
- Alerts:
  • Error rate > [X]% → email + Teams notification
  • Response time P95 > [Y]ms → warning
  • Pod restart count > [N] in 5 min → critical

Show: Dockerfile modification (add agent), env vars, and Terraform for alerts.
```

### KQL queries for troubleshooting
```
Write Azure Monitor KQL queries for these scenarios:

1. Find all 5xx errors in the last hour, grouped by endpoint:
2. Show the slowest 10 requests in the last 24 hours with full trace:
3. Track dependency failures (MySQL, Kafka, external APIs) over time:
4. Show request volume and error rate per minute for the last 6 hours:
5. Find exceptions with stack traces matching [keyword]:
6. Correlate a single request across all services using correlationId:
7. Show memory and CPU trends for my pods:
8. Calculate P50, P95, P99 latency for [endpoint]:

For each, show: the KQL query, where to run it (Logs blade), and how to pin to a dashboard.
```

---

## Cost Optimization

### Analyze and reduce Azure spend
```
Help me optimize Azure costs for my application. Current monthly spend: ~$[X].

Resources:
- AKS: [node count, VM size]
- Azure MySQL: [tier, size]
- Blob Storage: [volume, tier]
- Other: [list]

Analyze and suggest:
1. Right-size VMs (check utilization with Azure Advisor)
2. Reserved Instances vs Pay-as-you-go for stable workloads
3. Spot instances for non-critical workloads
4. Blob lifecycle policies (tiering)
5. MySQL: pause dev/staging during off-hours? Burstable tier?
6. AKS: cluster autoscaler tuning, node auto-provisioning
7. Unused resources (unattached disks, idle IPs, empty resource groups)
8. Azure Cost Management alerts and budgets

Prioritize by savings potential (high/medium/low).
```

---

## Quick One-Liners

```
Show me the az CLI command to [list resources / check AKS status / view Key Vault secrets /
download blob / check MySQL metrics].
```

```
How do I connect to Azure MySQL from my local machine through a private endpoint?
Show the SSH tunnel approach.
```

```
Generate a SAS token for Azure Blob container [name] with read-only access for 24 hours.
Show both az CLI and Java SDK approaches.
```

```
What Azure RBAC role does my Managed Identity need to [read Key Vault / write Blob Storage /
connect to MySQL / pull from ACR]? Show the role assignment command.
```

```
Compare Azure App Service vs AKS vs Container Apps for hosting my Spring Boot app.
Decision matrix with cost, complexity, scaling, and team skill requirements.
```

---

*In Azure: Managed Identity > connection strings. Private Endpoints > public access. Always.*
