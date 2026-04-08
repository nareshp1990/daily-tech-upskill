# DevOps & Observability — Daily Prompts

> Tech stack: Spring Boot Actuator, Micrometer, Prometheus, Grafana, ELK/EFK, Azure Monitor, PagerDuty

---

## Logging

### Design structured logging for Spring Boot
```
Design a structured logging strategy for my Spring Boot microservices.

Stack: Spring Boot 3.x, Logback, JSON format, EFK (Elasticsearch + Fluentd + Kibana)

Cover:
1. **Logback configuration** (logback-spring.xml):
   - JSON layout (LogstashEncoder) for prod
   - Console pattern for local dev
   - Profile-based switching (spring.profiles.active)

2. **MDC (Mapped Diagnostic Context)**:
   - traceId, spanId (from Micrometer Tracing)
   - userId, requestId, sessionId
   - serviceName, environment
   - Custom MDC filter for HTTP requests

3. **Log levels by package**:
   - com.myapp: DEBUG (dev), INFO (prod)
   - org.springframework: WARN
   - org.hibernate.SQL: DEBUG (dev only)
   - org.apache.kafka: WARN

4. **What to log at each level**:
   - ERROR: unrecoverable failures, data loss risk, circuit breaker open
   - WARN: retries, fallbacks, degraded service, slow queries (>500ms)
   - INFO: request/response summary, business events, state transitions
   - DEBUG: detailed flow, SQL, external API calls

5. **What NOT to log**:
   - PII (names, emails, SSN) — mask or exclude
   - Full request/response bodies in prod
   - Passwords, tokens, API keys

Show: logback-spring.xml, MDCFilter.java, and example log output.
```

### Log analysis prompts for debugging
```
I'm investigating a production issue. Help me write search queries.

Log system: Kibana (EFK stack)
Service: [service name]
Time range: [last N hours]
Symptom: [describe the issue]

Generate queries for:
1. **Error spike detection**:
   level:ERROR AND service:[name] | timechart count by exception.class

2. **Request tracing** (follow one request):
   traceId:"abc-123" | sort @timestamp asc

3. **Slow request identification**:
   response.duration:>2000 AND service:[name] | stats avg(duration) by endpoint

4. **Error correlation** (find common patterns):
   level:ERROR | top 10 exception.message
   level:ERROR | stats count by endpoint, exception.class

5. **Upstream/downstream failures**:
   (level:ERROR OR level:WARN) AND (message:*timeout* OR message:*connection refused* OR message:*circuit breaker*)

6. **Deployment correlation** (errors after deploy):
   level:ERROR AND @timestamp > "deploy-time" | timechart count

Adapt these to my specific issue and show KQL / Lucene syntax.
```

---

## Metrics & Monitoring

### Set up Micrometer metrics for Spring Boot
```
Set up comprehensive metrics for my Spring Boot microservice using Micrometer + Prometheus.

Include:

1. **RED metrics** (Rate, Errors, Duration) for every endpoint:
   - http.server.requests (auto from Actuator)
   - Custom timer for business operations

2. **Custom business metrics**:
   - Orders processed (counter, tagged by status: success/failure/cancelled)
   - Payment amount (distribution summary with percentile histograms)
   - Queue depth (gauge for Kafka consumer lag)
   - Cache hit ratio (counter for hits vs misses)
   - Active users (gauge, updated periodically)

3. **JVM metrics** (auto from Micrometer):
   - jvm.memory.used, jvm.gc.pause, jvm.threads.live
   - process.cpu.usage, system.cpu.usage

4. **Database metrics**:
   - hikaricp.connections.active / idle / pending
   - Custom query timer for slow queries

5. **Kafka metrics**:
   - kafka.consumer.lag
   - kafka.producer.record.send.rate

Show:
- MetricsConfig.java with custom MeterRegistryCustomizer
- Example @Timed and custom metric usage in service class
- application.yml Actuator config (endpoints, tags)
- Prometheus scrape config snippet
```

### Design Grafana dashboards
```
Design Grafana dashboard layouts for my Spring Boot microservices.

Create dashboard JSON/description for:

**Dashboard 1: Service Overview (team landing page)**
- Row 1: Request rate (req/s), Error rate (%), P50/P95/P99 latency
- Row 2: Active instances, CPU per pod, Memory per pod
- Row 3: HTTP status code distribution (2xx, 4xx, 5xx over time)
- Variables: service, namespace, time range

**Dashboard 2: Service Deep Dive**
- Row 1: Request rate and latency by endpoint
- Row 2: Error rate by endpoint + exception class
- Row 3: JVM heap, GC pauses, thread count
- Row 4: HikariCP pool (active, idle, pending, timeouts)
- Row 5: Kafka consumer lag, produce rate, error rate

**Dashboard 3: Business Metrics**
- Row 1: Orders/min, revenue/hour, conversion rate
- Row 2: Payment success rate, average processing time
- Row 3: Top errors affecting business flow
- Row 4: SLA compliance (% requests < 500ms)

For each panel, provide:
- PromQL query
- Visualization type (graph, stat, gauge, table, heatmap)
- Thresholds (green/yellow/red)
- Alert annotations
```

---

## Alerting

### Design an alerting strategy
```
Design an alerting strategy for my Spring Boot microservices on AKS.

Stack: Prometheus Alertmanager → PagerDuty (P1/P2) + Slack (P3/P4)

**Severity levels:**
- P1 (Critical): Customer-facing outage, data loss → PagerDuty (immediate page)
- P2 (High): Degraded service, error spike → PagerDuty (15min delay)
- P3 (Warning): Elevated latency, resource pressure → Slack #alerts
- P4 (Info): Approaching limits, anomaly detected → Slack #alerts-low

**Alert rules (with PromQL):**

P1:
- Service down (0 healthy pods for 2min)
- Error rate > 10% for 5min
- Database connection pool exhausted

P2:
- P99 latency > 5s for 10min
- Error rate > 5% for 10min
- Kafka consumer lag > 10,000 for 15min
- Disk usage > 90%

P3:
- P95 latency > 2s for 15min
- Memory usage > 80% for 10min
- Certificate expiry < 14 days
- Pod restart count > 3 in 30min

P4:
- CPU > 70% sustained 30min
- HikariCP pending > 0 for 5min
- GC pause > 500ms

For each alert:
- PromQL expression with appropriate thresholds and for-duration
- Runbook link template
- Recommended action

Show: prometheus-alerts.yml with all rules.
```

### Write a runbook for an alert
```
Write an on-call runbook for this alert: [alert name]

Structure:
## Alert: [name]
**Severity:** P[1-4]
**Fires when:** [condition in plain English]
**PromQL:** [query]

## Impact
- What's affected (users, services, data)
- Business impact (revenue, SLA)

## Quick Diagnosis (< 5 minutes)
1. Check: [specific dashboard link]
2. Run: kubectl get pods -n [namespace] | grep [service]
3. Check: [log query]
4. Verify: [dependency health]

## Common Causes & Fixes
### Cause 1: [most likely]
- Symptoms: [what you'll see]
- Fix: [exact commands]
- Verify: [how to confirm it's fixed]

### Cause 2: [next most likely]
- Symptoms: ...
- Fix: ...

## Escalation
- If not resolved in 15min → escalate to [team/person]
- If data loss suspected → notify [manager] immediately
- Slack: #incident-response

## Post-Incident
- [ ] Update incident timeline
- [ ] Write blameless post-mortem (template: [link])
- [ ] Create follow-up tickets
```

---

## Incident Response

### Investigate a production incident
```
I'm responding to a production incident. Help me investigate systematically.

Alert: [alert name/description]
Time started: [when]
Symptoms: [what users/monitoring show]
Recent changes: [deploys, config changes, infra updates]

Walk me through:

**Phase 1: Triage (0-5 min)**
1. Confirm scope: which services, which users, which regions?
2. Check: is this a partial or full outage?
3. Communicate: post to #incidents with initial assessment

**Phase 2: Diagnose (5-15 min)**
4. Check deployment history: was anything deployed recently?
   kubectl rollout history deployment/[service] -n [namespace]
5. Check dependent services: databases, Kafka, external APIs
6. Check resource pressure: CPU, memory, disk, connections
7. Check logs for error patterns:
   [Kibana query for errors in timeframe]

**Phase 3: Mitigate (15-30 min)**
8. If recent deploy: rollback
   kubectl rollout undo deployment/[service] -n [namespace]
9. If resource pressure: scale up
   kubectl scale deployment/[service] --replicas=[N]
10. If dependency failure: check circuit breakers, enable fallback

**Phase 4: Resolve & Document**
11. Confirm metrics returning to normal
12. Update incident channel with resolution
13. Schedule post-mortem within 48 hours

Adapt this to my specific incident.
```

### Write a post-mortem
```
Help me write a blameless post-mortem for this incident.

Incident: [title]
Date: [date]
Duration: [start - end]
Severity: P[1-4]
Impact: [users affected, revenue impact, SLA breach]

Structure:

## Incident Summary
One paragraph: what happened, impact, duration.

## Timeline (UTC)
| Time | Event |
|------|-------|
| HH:MM | [first indicator] |
| HH:MM | [alert fired] |
| HH:MM | [investigation started] |
| HH:MM | [root cause identified] |
| HH:MM | [mitigation applied] |
| HH:MM | [fully resolved] |

## Root Cause
What actually broke and why (technical detail, no blame).

## Contributing Factors
- What made this more likely to happen?
- What made detection/resolution slower?

## What Went Well
- Fast detection? Good alerting? Quick rollback?

## What Went Wrong
- Missing monitoring? Unclear runbook? Slow escalation?

## Action Items
| Priority | Action | Owner | Due Date |
|----------|--------|-------|----------|
| P1 | [prevent recurrence] | | |
| P2 | [improve detection] | | |
| P3 | [improve response] | | |

## Lessons Learned
Key takeaways for the team.

Fill in based on my incident details: [provide details]
```

---

## Health Checks & Readiness

### Design health check endpoints
```
Design comprehensive health check endpoints for my Spring Boot microservice.

Stack: Spring Boot Actuator, AKS, dependencies: MySQL, Redis, Kafka, external API

**Liveness probe** (/actuator/health/liveness):
- Is the JVM alive and responsive?
- NOT dependent on external services (don't cascade failures)
- Custom check: is the main event loop responsive? (deadlock detection)
- K8s config: initialDelaySeconds: 30, periodSeconds: 10, failureThreshold: 3

**Readiness probe** (/actuator/health/readiness):
- Can this instance serve traffic?
- Check: database connection pool has available connections
- Check: Kafka consumer is assigned partitions
- Check: Redis is reachable
- Check: feature flag service is loaded
- K8s config: initialDelaySeconds: 10, periodSeconds: 5, failureThreshold: 3

**Startup probe** (/actuator/health/liveness):
- Has the app finished initialising?
- K8s config: initialDelaySeconds: 0, periodSeconds: 5, failureThreshold: 30 (allows 150s startup)

**Deep health** (/actuator/health — restricted access):
- All dependency details
- Connection pool stats
- Kafka lag
- Cache connectivity
- External API latency

Show:
- HealthCheckConfig.java with custom health indicators
- application.yml actuator config
- Kubernetes deployment.yaml probes section
```

---

## Performance Debugging

### Diagnose a slow service
```
My Spring Boot service has degraded performance. Help me diagnose.

Symptoms: [P95 latency increased from 200ms to 2s]
When: [started at HH:MM, or after deploy X]
Traffic: [normal / spike / changed pattern]

Systematic diagnosis:

**Layer 1: Infrastructure**
- CPU throttling? kubectl top pods -n [namespace]
- Memory pressure / GC? Check jvm.gc.pause metrics
- Network latency between pods? Check service mesh metrics
- Node resource contention? kubectl describe node

**Layer 2: Database**
- Slow queries? Check Hikari metrics (active connections, wait time)
- Missing index? EXPLAIN ANALYZE on recent queries
- Lock contention? SHOW PROCESSLIST / pg_stat_activity
- Connection pool exhaustion? hikaricp.connections.pending > 0

**Layer 3: External Dependencies**
- Kafka lag increasing? kafka.consumer.lag metric
- Redis latency? redis.command.duration metric
- External API slow? Check circuit breaker state, timeout metrics
- DNS resolution issues?

**Layer 4: Application**
- Thread pool exhaustion? jvm.threads.live vs jvm.threads.peak
- Memory leak? Heap trending up between GC cycles
- N+1 queries? Check SQL log count per request
- Synchronous calls that should be async?

**Layer 5: Traffic**
- Changed request pattern? New heavy endpoint getting traffic?
- Missing pagination? Large result sets?
- Bot/scraper traffic?

For each finding, suggest the fix and expected improvement.
```

### Thread dump analysis
```
Analyse this thread dump from my Spring Boot application.

[paste thread dump or describe how to get it]

How to capture:
- kubectl exec -it [pod] -- jstack [pid]
- Or: curl localhost:8080/actuator/threaddump

Help me identify:
1. **Blocked threads**: waiting on locks (deadlock potential)
2. **WAITING threads**: what are they waiting for? (I/O, locks, queue)
3. **Thread pool saturation**: are Tomcat/async/scheduler pools full?
4. **Long-running threads**: stuck in business logic or external calls
5. **GC threads**: are they dominating?

For each problem area:
- Which thread pool is affected
- What it's blocked on
- Impact on service
- How to fix (pool sizing, timeout config, async conversion)
```

---

## Kubernetes Operations

### Debug a CrashLoopBackOff pod
```
My Spring Boot pod is in CrashLoopBackOff. Help me debug.

Service: [name]
Namespace: [namespace]
Cluster: AKS

Diagnostic steps:
1. kubectl describe pod [name] -n [namespace]
   → Check: Events, Exit Code, OOMKilled, liveness probe failures

2. kubectl logs [pod] -n [namespace] --previous
   → Check: startup errors, config issues, port conflicts

3. Common Spring Boot causes:
   - Exit Code 137: OOMKilled → increase memory limit or fix leak
   - Exit Code 1: application error → check logs for exception
   - Liveness probe failing: app starts too slowly → increase initialDelaySeconds
   - Config missing: missing environment variable or secret mount
   - Database unreachable: wrong connection string or network policy

4. kubectl get events -n [namespace] --sort-by='.lastTimestamp'
   → Check: image pull errors, resource quota exceeded, node pressure

5. Quick fixes:
   - OOM: kubectl set resources deployment/[name] --limits=memory=1Gi
   - Slow start: increase startupProbe.failureThreshold
   - Config: kubectl get configmap/secret and verify values

Show exact commands for each step.
```

### Kubernetes resource tuning
```
Help me right-size my Spring Boot application's Kubernetes resources.

Current config:
- Replicas: [N]
- CPU request/limit: [values]
- Memory request/limit: [values]
- JVM: -Xms[N] -Xmx[N]

Current metrics (from Prometheus):
- Average CPU usage: [value]
- P95 CPU usage: [value]
- Average memory: [value]
- Peak memory: [value]
- Request rate: [req/s]
- P95 latency: [value]

Help me:
1. Set CPU requests = P95 usage + 20% headroom
2. Set CPU limits = 2x requests (allow bursting)
3. Set memory requests = peak JVM heap + metaspace + stack + 25% headroom
4. Set memory limits = requests + 10% (prevent OOMKill with small buffer)
5. JVM heap = 75% of memory request (leave room for metaspace, stack, native memory)
6. HPA config: target CPU 70%, min/max replicas, scale-down stabilization

Show: updated deployment.yaml resources section and JVM args.
```

---

## On-Call Helpers

### Daily operational check prompt
```
Generate a daily operational health check routine for my on-call shift.

Morning check (5 min):
1. Check all dashboards — any overnight anomalies?
2. Review fired alerts in the last 12h — any unresolved?
3. Check deployment history — anything deployed overnight?
4. Verify: all pods Running, no restarts > 3
5. Kafka consumer lag — any queues backing up?
6. Certificate expiry — anything within 7 days?
7. Database: replication lag, disk usage, long-running queries

Generate kubectl and PromQL one-liners for each check.
```

---

*Tailored for: Spring Boot / AKS / Prometheus / Grafana / EFK / PagerDuty / On-Call*
