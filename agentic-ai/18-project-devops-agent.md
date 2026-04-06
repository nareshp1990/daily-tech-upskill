# Agentic AI — Day 18: Project — AI-Powered DevOps Assistant
**Track:** Agentic AI (Implementation-Focused)
**Time Target:** 1–2 hours

---

## 1. Today's Focus
**Build a DevOps Agent That Monitors, Diagnoses, and Helps Resolve Production Issues**

This agent is an on-call engineer's best friend: it can check deployment status, query logs, inspect Kubernetes pods, analyse error patterns, and suggest remediation steps. It combines infrastructure tools with LLM reasoning to accelerate incident response.

---

## 2. Architecture

```
On-call Engineer: "The payment service is returning 500 errors"
       │
       ▼
┌──────────────────┐
│ DevOps Agent     │
│ (LangGraph)      │
└───┬──────────────┘
    │
    ├── Tool: check_service_health(service) → HTTP health check
    ├── Tool: query_logs(service, time_range, level) → search logs
    ├── Tool: get_k8s_pods(namespace, service) → pod status
    ├── Tool: get_metrics(service, metric, duration) → Prometheus query
    ├── Tool: get_recent_deployments(service) → deployment history
    ├── Tool: run_kubectl(command) → safe kubectl commands
    │
    ▼
┌──────────────────┐
│ Output:          │
│ - Diagnosis      │
│ - Root cause     │
│ - Remediation    │
│ - Runbook link   │
└──────────────────┘
```

---

## 3. Tools

```python
import requests
import subprocess
import json
from datetime import datetime, timedelta
from langchain_core.tools import tool

@tool
def check_service_health(service_name: str, port: int = 8080) -> str:
    """Check the health endpoint of a service. Returns health status and details.
    Example: service_name='payment-service', port=8080"""
    try:
        url = f"http://{service_name}.production.svc.cluster.local:{port}/actuator/health"
        resp = requests.get(url, timeout=5)
        return f"Status: {resp.status_code}\nBody: {json.dumps(resp.json(), indent=2)}"
    except requests.exceptions.ConnectionError:
        return f"UNREACHABLE: Cannot connect to {service_name}:{port}"
    except requests.exceptions.Timeout:
        return f"TIMEOUT: {service_name}:{port} did not respond within 5s"

@tool
def query_logs(service_name: str, level: str = "ERROR", minutes: int = 30) -> str:
    """Query application logs from the logging system (Loki/ELK).
    level: DEBUG, INFO, WARN, ERROR. Returns last N minutes of matching logs.
    Example: service_name='payment-service', level='ERROR', minutes=15"""
    # In production: query Loki/Elasticsearch API
    # Simulated with kubectl logs
    try:
        result = subprocess.run(
            ["kubectl", "logs", "-l", f"app={service_name}",
             "-n", "production", "--since", f"{minutes}m",
             "--tail", "100"],
            capture_output=True, text=True, timeout=30
        )
        lines = result.stdout.strip().split("\n")
        filtered = [l for l in lines if level in l.upper()]
        if not filtered:
            return f"No {level} logs found in the last {minutes} minutes."
        return f"Found {len(filtered)} {level} entries:\n" + "\n".join(filtered[:30])
    except Exception as e:
        return f"Error querying logs: {e}"

@tool
def get_k8s_pods(service_name: str, namespace: str = "production") -> str:
    """Get Kubernetes pod status for a service. Shows running, pending, and crashed pods.
    Example: service_name='payment-service'"""
    try:
        result = subprocess.run(
            ["kubectl", "get", "pods", "-l", f"app={service_name}",
             "-n", namespace, "-o", "wide", "--no-headers"],
            capture_output=True, text=True, timeout=15
        )
        if not result.stdout.strip():
            return f"No pods found for {service_name} in {namespace}"
        return f"Pods for {service_name}:\n{result.stdout}"
    except Exception as e:
        return f"Error: {e}"

@tool
def get_metrics(service_name: str, metric: str, duration: str = "15m") -> str:
    """Query Prometheus metrics for a service.
    Common metrics: http_request_duration_seconds, http_requests_total,
    jvm_memory_used_bytes, process_cpu_usage, hikaricp_connections_active.
    duration: 5m, 15m, 1h, 3h.
    Example: service_name='payment-service', metric='http_requests_total', duration='15m'"""
    try:
        query = f'{metric}{{app="{service_name}"}}'
        resp = requests.get("http://prometheus:9090/api/v1/query_range", params={
            "query": query, "start": f"-{duration}", "end": "now", "step": "60s"
        }, timeout=10)
        data = resp.json()
        if data["status"] == "success" and data["data"]["result"]:
            results = data["data"]["result"]
            return json.dumps(results[:5], indent=2)  # Limit output
        return f"No data for metric {metric} on {service_name}"
    except Exception as e:
        return f"Prometheus query error: {e}"

@tool
def get_recent_deployments(service_name: str, count: int = 5) -> str:
    """Get recent deployment history for a service. Shows image version, time, and status.
    Use this to check if a recent deploy might have caused the issue."""
    try:
        result = subprocess.run(
            ["kubectl", "rollout", "history", f"deployment/{service_name}",
             "-n", "production"],
            capture_output=True, text=True, timeout=15
        )
        return result.stdout or "No deployment history found."
    except Exception as e:
        return f"Error: {e}"

@tool
def describe_pod(pod_name: str, namespace: str = "production") -> str:
    """Get detailed information about a specific pod including events.
    Use this when a pod is in CrashLoopBackOff or Pending state."""
    try:
        result = subprocess.run(
            ["kubectl", "describe", "pod", pod_name, "-n", namespace],
            capture_output=True, text=True, timeout=15
        )
        # Return last section (Events) which is most useful
        output = result.stdout
        events_idx = output.find("Events:")
        if events_idx > -1:
            return output[events_idx:]
        return output[-3000:]  # Last 3000 chars
    except Exception as e:
        return f"Error: {e}"
```

---

## 4. Agent

```python
from langgraph.prebuilt import create_react_agent
from langchain_anthropic import ChatAnthropic

model = ChatAnthropic(model="claude-sonnet-4-20250514", max_tokens=4096)

devops_agent = create_react_agent(
    model,
    tools=[check_service_health, query_logs, get_k8s_pods,
           get_metrics, get_recent_deployments, describe_pod],
    prompt="""You are a senior SRE / DevOps engineer assisting with incident response.

    When investigating an issue:
    1. Check service health first (is the service reachable?)
    2. Check pod status (are pods running, crashing, or pending?)
    3. Query recent error logs (what errors are occurring?)
    4. Check metrics (latency spikes, error rate increases, resource saturation?)
    5. Check recent deployments (did a deploy cause this?)

    Structure your diagnosis as:
    ## Diagnosis
    - **Status**: [healthy/degraded/down]
    - **Symptoms**: What's happening
    - **Root Cause**: Most likely cause based on evidence
    - **Evidence**: Specific logs, metrics, or pod status that support this
    - **Remediation**: Step-by-step actions to resolve
    - **Prevention**: What to add to prevent recurrence

    Be precise. Reference specific error messages, pod names, and metric values."""
)

# Run
result = devops_agent.invoke({
    "messages": [("user", "The payment service is returning 500 errors since about 30 minutes ago. "
                          "We also got PagerDuty alerts for high latency. Can you investigate?")]
})
print(result["messages"][-1].content)
```

---

## 5. Java Alternative (Spring Boot)

```java
@Service
public class DevOpsAgentService {

    public String investigate(String issue) {
        return chatClient.prompt()
            .system(DEVOPS_SYSTEM_PROMPT)
            .user(issue)
            .tools(
                new ServiceHealthTool(),
                new KubernetesTool(),
                new LogQueryTool(),
                new MetricsTool()
            )
            .call()
            .content();
    }
}
```

---

## 6. Hands-On Exercise

1. Implement the DevOps tools (mock the `kubectl` and Prometheus calls if you don't have a cluster).
2. Build the agent and test with these scenarios:
   - "Payment service returning 500 errors"
   - "Order service pods are in CrashLoopBackOff"
   - "Database connection pool is exhausted on user-service"
3. Test the agent's diagnostic reasoning — does it follow a logical investigation path?
4. Add a `rollback_deployment` tool with human approval (interrupt) before execution.

---

## 7. Key Takeaways

1. **Structured diagnosis beats ad-hoc troubleshooting** — the agent follows a consistent playbook.
2. **Never auto-remediate without approval** — read-only tools are safe; write tools need human confirmation.
3. **Context matters** — recent deployments are the most common root cause of sudden failures.
4. **This agent augments on-call, not replaces it** — it saves 15 minutes of initial investigation.

---

*Day 18. The best incident response starts with the right questions — and an agent that knows how to ask them.*
