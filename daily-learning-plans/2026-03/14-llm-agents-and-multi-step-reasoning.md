# Day 7 — LLM Agents & Multi-Step Reasoning
**Date:** 2026-03-14
**Phase:** 1 — Generative AI & Agentic AI
**Time Target:** 1–2 hours

---

## 1. Today's Focus Topic
**LLM Agents & Multi-Step Reasoning — From Single Calls to Autonomous Loops**

Day 6 taught the LLM to call a single tool and return a result. Today you wire that into a *loop*: the LLM reasons, decides which tool to use, observes the result, reasons again, and repeats — until it has enough information to produce a final answer. This loop is called **ReAct** (Reason + Act), and it is the foundation of every AI agent. Analogy: you're upgrading from a calculator (one operation) to a junior developer (plans work, takes steps, checks results, adjusts). The LLM acts as the planner; your tools are its actions; the loop is its work cycle.

---

## 2. Key Concepts

### The ReAct Loop (Reason → Act → Observe → Repeat)
ReAct stands for **Re**ason + **Act**. Each iteration:
1. **Thought** — the LLM reasons about what to do next (often visible in `<thinking>` or scratchpad tokens).
2. **Action** — the LLM emits a tool call.
3. **Observation** — your code executes the tool and returns the result.
4. **Repeat** — the LLM receives the observation and either calls another tool or produces a final answer.

This is not magic — it's a prompt-driven loop. The LLM's system prompt instructs it to think step-by-step, use tools as needed, and only answer when ready. Spring AI manages the loop automatically when tools are registered.

### Agent vs. Chain vs. Single Call
| Style | Description | When to Use |
|-------|-------------|-------------|
| Single call | One prompt → one response | Simple Q&A, classification |
| Chain | Fixed sequence of calls (A → B → C) | Predictable pipelines (RAG, summarise → translate) |
| Agent (ReAct) | Dynamic loop — LLM decides next step | Open-ended tasks, multi-tool, unknown number of steps |

### Planning — How Agents Decompose Tasks
A capable agent doesn't just react — it plans. Given a complex goal, a planning agent first produces a task list, then executes each task using tools, then synthesises. Spring AI doesn't have a built-in planner yet, but you can implement it by:
1. Making a first LLM call to produce a JSON plan (list of steps).
2. Iterating through the plan, calling tools per step.
3. Making a final LLM call to synthesise all results.

### Max Iterations & Termination
Agents need guardrails. Spring AI's `ChatClient` loop terminates when the LLM stops calling tools. You add safety by:
- Setting `maxToolCalls` (per-request limit).
- Defining a `ToolCallbackResolver` that returns an error for unknown tools.
- Adding a timeout to the underlying HTTP client.

### Spring AI Agent Abstractions
Spring AI 1.0 does not yet ship a named `Agent` class — the agent pattern is implemented via `ChatClient` + tools + a loop you control. The community-maintained `spring-ai-agents` extension (experimental) adds an `AgentExecutor` interface. For production use, compose `ChatClient` + tools + your loop logic.

---

## 3. Hands-On Task

### Goal: Build a research agent that autonomously plans and executes a multi-step investigation

**Scenario:** A user asks *"Summarise the order history for customer C-42 and flag any orders that are delayed."* The agent must: look up orders for the customer, check each order's status, identify delays, and synthesise a report — without the caller specifying each step.

**Step 1 — Domain objects**
```java
public record CustomerOrders(String customerId, List<String> orderIds) {}
public record OrderDetail(String orderId, String status, LocalDate expectedDelivery, boolean delayed) {}
```

**Step 2 — Tools (add to `OrderTools` from Day 6)**
```java
@Tool(description = "Returns all order IDs for a given customer ID. " +
                    "Call this first when the user asks about a customer's order history.")
public CustomerOrders getOrdersForCustomer(
        @ToolParam(description = "The customer ID, e.g. C-42") String customerId) {
    // Simulated data
    return switch (customerId) {
        case "C-42" -> new CustomerOrders(customerId, List.of("ORD-101", "ORD-102", "ORD-103"));
        default -> new CustomerOrders(customerId, List.of());
    };
}

@Tool(description = "Returns detailed status and delivery info for a specific order, " +
                    "including whether it is delayed past the expected delivery date.")
public OrderDetail getOrderDetail(
        @ToolParam(description = "The order ID to inspect") String orderId) {
    LocalDate today = LocalDate.now();
    return switch (orderId) {
        case "ORD-101" -> new OrderDetail(orderId, "DELIVERED", today.minusDays(1), false);
        case "ORD-102" -> new OrderDetail(orderId, "SHIPPED",   today.minusDays(3), true);  // overdue
        case "ORD-103" -> new OrderDetail(orderId, "PROCESSING", today.plusDays(2),  false);
        default        -> new OrderDetail(orderId, "NOT_FOUND", null, false);
    };
}
```

**Step 3 — Agent service with explicit loop control**
```java
@Service
public class ResearchAgentService {

    private final ChatClient chatClient;

    public ResearchAgentService(ChatClient.Builder builder, OrderTools orderTools) {
        this.chatClient = builder
            .defaultSystem("""
                You are an order research agent. When given a task:
                1. First call getOrdersForCustomer to get the order list.
                2. Then call getOrderDetail for each order individually.
                3. After gathering all information, produce a concise summary
                   highlighting any delayed orders.
                Never guess — always use tools to get real data.
                """)
            .defaultTools(orderTools)
            .build();
    }

    public String investigate(String task) {
        return chatClient.prompt()
            .user(task)
            .call()
            .content();
        // Spring AI handles the ReAct loop: it keeps calling tools
        // until the LLM stops issuing tool_call blocks.
    }
}
```

**Step 4 — REST endpoint**
```java
@RestController
@RequestMapping("/api/agent")
public class AgentController {

    private final ResearchAgentService agentService;

    public AgentController(ResearchAgentService agentService) {
        this.agentService = agentService;
    }

    @PostMapping("/investigate")
    public Map<String, String> investigate(@RequestBody Map<String, String> body) {
        return Map.of("report", agentService.investigate(body.get("task")));
    }
}
```

**Step 5 — Observe the multi-step loop**
1. POST `{"task": "Summarise order history for customer C-42 and flag delayed orders."}` — watch the log: Spring AI will call `getOrdersForCustomer` once, then `getOrderDetail` three times, then produce the summary.
2. Add `log.info("Tool called: {}", orderId)` inside each tool method — count invocations to see the loop in action.
3. Send a task with an unknown customer (`C-99`) — the agent should report no orders found, not hallucinate.
4. Send a two-customer task — observe the agent handling both in one conversation turn.

**Step 6 — Add a planning step (two-pass agent)**
```java
public String planThenExecute(String goal) {
    // Pass 1: produce a structured plan
    String plan = chatClient.prompt()
        .system("You are a planner. Given a goal, output a numbered JSON list of tool calls " +
                "needed to complete it. Do not call tools yet — just plan.")
        .user(goal)
        .call()
        .content();

    log.info("Agent plan: {}", plan);

    // Pass 2: execute with full tools
    return chatClient.prompt()
        .system("You are an executor. Follow this plan and use tools to complete each step:\n" + plan)
        .user(goal)
        .call()
        .content();
}
```

---

## 4. Resources

- **[Spring AI ChatClient + Tools](https://docs.spring.io/spring-ai/reference/api/chatclient.html)** — How `ChatClient` manages multi-turn tool loops.
- **[ReAct: Synergizing Reasoning and Acting in Language Models (paper)](https://arxiv.org/abs/2210.03629)** — The original ReAct paper (short, very readable). Essential background for understanding *why* the loop works.
- **[Anthropic: Building Effective Agents](https://www.anthropic.com/research/building-effective-agents)** — Practical agent patterns from Anthropic: augmented LLM, routing, orchestrator-subagent, and parallelisation.

---

## 5. Trending Context

**Agentic AI is the fastest-moving area of applied AI in 2025.** Every major cloud vendor now ships an agent runtime (AWS Bedrock Agents, Azure AI Agent Service, Google Vertex AI Agents). The frontier has shifted from "can LLMs use tools?" (solved) to "how do you make agents reliable, observable, and safe?" Key trends: (1) **structured outputs for tool calls** — models output JSON plans, not free-text reasoning, for more reliable parsing; (2) **agent memory** — giving agents a vector store to remember past interactions; (3) **multi-agent systems** — an orchestrator agent delegates to specialist sub-agents (each with their own tool set). Spring AI's roadmap includes an `AgentExecutor` and multi-agent support in upcoming releases.

---

## 6. Community Tip

**Always set a max-iteration guard in agents — the LLM can loop forever on ambiguous tasks.** In Spring AI, the loop terminates when the LLM stops calling tools, but a confused or poorly prompted LLM can call tools in circles. Add a bounded retry at the service layer using a simple counter or Resilience4j `TimeLimiter`. Also: log every tool call with a correlation ID so you can trace the full agent run in your logs — agent debugging without traces is extremely painful. Think of each ReAct iteration as a span in a distributed trace; tools are downstream service calls. Treat observability as a first-class requirement from day one.

---

## 7. Tomorrow's Preview

**Day 8** will cover **LLM Memory & Conversation State** — how to give agents persistent memory across sessions, implement short-term (in-context) and long-term (vector-store-backed) memory, and manage conversation history without blowing the context window.

---

*Built progressively. Each day connects to the next.*
