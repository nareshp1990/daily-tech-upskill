# Day 11 — Multi-Agent Systems & Orchestration
**Date:** 2026-03-18
**Phase:** 1 — Generative AI & Agentic AI
**Time Target:** 1–2 hours

---

## 1. Today's Focus Topic
**Multi-Agent Systems & Orchestration — Divide, Specialise, Conquer**

A single agent with 10 tools becomes slow, unpredictable, and hard to debug. The solution: break the problem into specialised agents — each with a narrow tool set and a focused system prompt — coordinated by an orchestrator. This is the multi-agent pattern, and it maps directly to microservices thinking: single responsibility, loose coupling, independent deployability. Today you build an orchestrator that delegates to specialist sub-agents, handles their results, and synthesises a final response. Analogy: an engineering manager (orchestrator) who delegates front-end work to a UI engineer, back-end work to a backend engineer, and synthesises both into a project update.

---

## 2. Key Concepts

### Why Multi-Agent?
| Problem with single agent | Multi-agent solution |
|--------------------------|---------------------|
| Too many tools — LLM gets confused about which to use | Each agent has 2–4 tightly scoped tools |
| Long context — history grows unmanageable | Each agent has its own focused context |
| One failure kills the entire task | Agents fail independently; orchestrator retries |
| Hard to test | Each agent is independently unit-testable |

### Orchestration Patterns
**1. Sequential (pipeline):** Agent A → output → Agent B → output → Agent C.
**2. Parallel (fan-out/fan-in):** Orchestrator sends to A, B, C simultaneously; waits for all; merges results.
**3. Router:** Orchestrator classifies the request and routes to exactly one specialist.
**4. Hierarchical:** Orchestrator → sub-orchestrators → leaf agents. Useful for very complex tasks.

### Inter-Agent Communication
In Spring, agents are just Spring beans (services). The orchestrator calls them as method calls — no message broker needed for in-process multi-agent systems. For distributed multi-agent (across services), use Kafka or HTTP — the same patterns you already know.

### Shared Context vs. Independent Context
- **Shared:** Pass a context object (e.g., `AgentContext`) between agents carrying accumulated state.
- **Independent:** Each agent receives only what it needs; the orchestrator merges results.
- Prefer independent — it's easier to test, reason about, and parallelize.

---

## 3. Hands-On Task

### Goal: Build an order investigation system with three specialist agents orchestrated by a coordinator

**Agents:**
- `OrderLookupAgent` — fetches order details (tools: `getOrdersForCustomer`, `getOrderDetail`)
- `PricingAgent` — calculates costs and discounts (tool: `calculateDiscount`)
- `ReportAgent` — synthesises results into a customer-facing report (no tools, just summarisation)

**Step 1 — Specialist agent: OrderLookupAgent**
```java
@Service
public class OrderLookupAgent {

    private final ChatClient chatClient;

    public OrderLookupAgent(ChatClient.Builder builder, OrderTools orderTools) {
        this.chatClient = builder
            .defaultSystem("You are an order lookup specialist. Use the provided tools to retrieve " +
                           "complete order information. Return a structured summary of all orders found.")
            .defaultTools(orderTools)
            .build();
    }

    public String lookup(String query) {
        return chatClient.prompt().user(query).call().content();
    }
}
```

**Step 2 — Specialist agent: PricingAgent**
```java
@Service
public class PricingAgent {

    private final ChatClient chatClient;

    public PricingAgent(ChatClient.Builder builder, OrderTools pricingTools) {
        this.chatClient = builder
            .defaultSystem("You are a pricing specialist. Use the calculateDiscount tool to compute " +
                           "accurate pricing. Always show original price, discount, and final price.")
            .defaultTools(pricingTools)
            .build();
    }

    public String calculatePricing(String pricingQuery) {
        return chatClient.prompt().user(pricingQuery).call().content();
    }
}
```

**Step 3 — Specialist agent: ReportAgent**
```java
@Service
public class ReportAgent {

    private final ChatClient chatClient;

    public ReportAgent(ChatClient.Builder builder) {
        this.chatClient = builder
            .defaultSystem("You are a report writer. Given structured data from other agents, " +
                           "write a clear, concise customer-facing summary. Use plain language. " +
                           "Highlight any issues or action items at the top.")
            .build();
    }

    public String generateReport(String orderData, String pricingData, String originalRequest) {
        return chatClient.prompt()
            .user("Original request: " + originalRequest +
                  "\n\nOrder data:\n" + orderData +
                  "\n\nPricing data:\n" + pricingData +
                  "\n\nWrite a customer-facing summary.")
            .call()
            .content();
    }
}
```

**Step 4 — Orchestrator (sequential pattern)**
```java
@Service
public class InvestigationOrchestrator {

    private final OrderLookupAgent orderLookupAgent;
    private final PricingAgent pricingAgent;
    private final ReportAgent reportAgent;

    public InvestigationOrchestrator(OrderLookupAgent orderLookupAgent,
                                      PricingAgent pricingAgent,
                                      ReportAgent reportAgent) {
        this.orderLookupAgent = orderLookupAgent;
        this.pricingAgent = pricingAgent;
        this.reportAgent = reportAgent;
    }

    public String investigate(String customerRequest) {
        log.info("Orchestrator: starting investigation for request: {}", customerRequest);

        // Step 1 — gather order data
        String orderData = orderLookupAgent.lookup(
            "Look up all orders mentioned in: " + customerRequest);
        log.info("OrderLookupAgent result: {}", orderData);

        // Step 2 — gather pricing data (parallel candidate)
        String pricingData = pricingAgent.calculatePricing(
            "Calculate any pricing or discounts mentioned in: " + customerRequest);
        log.info("PricingAgent result: {}", pricingData);

        // Step 3 — synthesise report
        String report = reportAgent.generateReport(orderData, pricingData, customerRequest);
        log.info("ReportAgent result: {}", report);

        return report;
    }
}
```

**Step 5 — Parallel fan-out with CompletableFuture**
```java
@Service
public class ParallelOrchestrator {

    private final OrderLookupAgent orderLookupAgent;
    private final PricingAgent pricingAgent;
    private final ReportAgent reportAgent;

    public String investigateParallel(String customerRequest) throws Exception {
        // Fan out — both agents run concurrently
        CompletableFuture<String> orderFuture  = CompletableFuture.supplyAsync(
            () -> orderLookupAgent.lookup("Look up orders in: " + customerRequest));

        CompletableFuture<String> pricingFuture = CompletableFuture.supplyAsync(
            () -> pricingAgent.calculatePricing("Calculate pricing in: " + customerRequest));

        // Fan in — wait for both
        CompletableFuture.allOf(orderFuture, pricingFuture).join();

        return reportAgent.generateReport(
            orderFuture.get(), pricingFuture.get(), customerRequest);
    }
}
```

**Step 6 — REST endpoint**
```java
@RestController
@RequestMapping("/api/orchestrator")
public class OrchestratorController {

    private final InvestigationOrchestrator sequential;
    private final ParallelOrchestrator parallel;

    @PostMapping("/sequential")
    public Map<String, String> sequential(@RequestBody Map<String, String> body) {
        return Map.of("report", sequential.investigate(body.get("request")));
    }

    @PostMapping("/parallel")
    public Map<String, String> parallel(@RequestBody Map<String, String> body) throws Exception {
        return Map.of("report", parallel.investigateParallel(body.get("request")));
    }
}
```

**Step 7 — Experiments**
1. POST `{"request": "Check order ORD-102 for customer C-42 and give me a $150 item at 20% discount"}` — observe all three agents activate.
2. Compare `/sequential` vs `/parallel` latency — parallel should be ~40–50% faster when agent tasks are independent.
3. Force `OrderLookupAgent` to fail (throw an exception) — add a `try/catch` in the orchestrator and observe graceful degradation.
4. Add a router: classify the request first (billing/shipping/technical) and only invoke relevant agents.

---

## 4. Resources

- **[Anthropic: Multi-Agent Systems](https://www.anthropic.com/research/building-effective-agents#multi-agent-systems)** — Architectural patterns for orchestrator-subagent, parallelisation, and specialisation.
- **[LangGraph](https://langchain-ai.github.io/langgraph/)** — The leading graph-based multi-agent framework (Python). Read their pattern docs for conceptual depth; the patterns translate directly to Spring.
- **[CrewAI](https://docs.crewai.com/)** — Role-based multi-agent framework. Good for understanding how agent roles and backstories improve specialisation.

---

## 5. Trending Context

**Multi-agent systems are moving from research to production in 2025.** The key inflection point: models have become reliable enough at tool use that you can trust a sub-agent to complete a sub-task without human oversight. Early adopters (Cognition/Devin, GitHub Copilot Workspace, Salesforce Einstein) are all multi-agent under the hood. The emerging standard: **agent graphs** (LangGraph, Temporal workflows) where agents are nodes and data flows are edges — giving you all the observability and retry semantics of a distributed workflow. Expect Spring AI to add native graph-based orchestration in 2025/2026.

---

## 6. Community Tip

**Give each agent a unique name in its system prompt and log that name with every tool call.** When debugging a multi-agent run, the hardest problem is figuring out which agent made which decision. Include the agent name in every log statement: `"[OrderLookupAgent] Calling getOrderDetail for ORD-102"`. Also: give each agent run a shared `correlationId` (e.g., a UUID passed through every agent call) so you can group all agent logs for a single user request together in your log aggregator. This is the same distributed tracing discipline you already use in microservices — apply it to agents from day one.

---

## 7. Tomorrow's Preview

**Day 12** will cover **LLM Security & Prompt Injection Defense** — the OWASP Top 10 for LLM applications, prompt injection attack patterns, input/output sanitisation, and how to build a hardened LLM layer in Spring.

---

*Built progressively. Each day connects to the next.*
