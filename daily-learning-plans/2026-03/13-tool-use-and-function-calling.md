# Day 6 — LLM Tool Use & Function Calling
**Date:** 2026-03-13
**Phase:** 1 — Generative AI & Agentic AI
**Time Target:** 1–2 hours

---

## 1. Today's Focus Topic
**LLM Tool Use & Function Calling — Giving the LLM Hands to Act with**

Days 4 and 5 taught the LLM to *know* things from your documents. Today you teach it to *do* things — call APIs, query databases, run calculations, send notifications. Function calling (also called "tool use") is the mechanism where an LLM reasons about which tool to invoke and what arguments to pass, and your code executes it. This is the core primitive of agentic AI. Analogy: the LLM is the brain, your functions are the hands — the brain decides what to do, the hands carry it out.

---

## 2. Key Concepts

### How Function Calling Works
1. You define functions (tools) with a name, description, and JSON Schema for parameters.
2. You send the user's message + tool definitions to the LLM.
3. The LLM responds with a `tool_call` instead of a text answer — specifying which function to call and with what args.
4. Your code executes the function and returns the result to the LLM.
5. The LLM uses the result to generate a final text response.

This loop can repeat: the LLM may call multiple tools in sequence before it has enough information to answer.

### Why This Matters
- LLMs can't natively access real-time data (stock prices, weather, your DB).
- Function calling makes the LLM a reasoning layer over your existing services — not a replacement for them.
- The LLM decides *when* to call a tool and *how to call it* — your job is to define the tools well.

### Tool Definition — What Makes a Good Tool
- **Name**: verb + noun, snake_case. `get_order_status`, not `orderInfo`.
- **Description**: be explicit — the LLM uses this to decide when to call the tool. Bad: `"Gets orders."` Good: `"Returns the current status and estimated delivery date for an order given its order ID. Only call this when the user is asking about a specific order's status."`
- **Parameters**: use JSON Schema — `type`, `description`, and `required` fields. The more descriptive, the better the LLM's argument generation.

### Spring AI — `@Tool` / `@Bean` Function Registration
Spring AI supports two styles:
1. **Method-level `@Tool` annotation** (Spring AI 1.0+) — cleanest approach.
2. **`Function<Input, Output>` bean** — compatible with older Spring AI versions.

### Multi-Turn Tool Use
- A single user message can trigger multiple sequential tool calls.
- The LLM maintains the conversation state and decides when it has enough information.
- Spring AI handles the multi-turn loop automatically via `ChatClient`.

---

## 3. Hands-On Task

### Goal: Build an order assistant that can look up order status and calculate delivery estimates

**Step 1 — Define domain objects**
```java
public record OrderStatus(String orderId, String status, LocalDate estimatedDelivery) {}
public record WeatherInfo(String city, String condition, int temperatureCelsius) {}
```

**Step 2 — Define tools with `@Tool`**
```java
@Component
public class OrderTools {

    // Simulated DB/service calls
    @Tool(description = "Returns the current status and estimated delivery date for a given order ID. " +
                        "Call this when the user asks about a specific order.")
    public OrderStatus getOrderStatus(
            @ToolParam(description = "The order ID, e.g. ORD-12345") String orderId) {
        // In real code: call your OrderService or DB
        return switch (orderId) {
            case "ORD-001" -> new OrderStatus(orderId, "SHIPPED", LocalDate.now().plusDays(2));
            case "ORD-002" -> new OrderStatus(orderId, "PROCESSING", LocalDate.now().plusDays(5));
            default -> new OrderStatus(orderId, "NOT_FOUND", null);
        };
    }

    @Tool(description = "Calculates a discount amount given an original price and discount percentage.")
    public Map<String, Object> calculateDiscount(
            @ToolParam(description = "Original price in USD") double originalPrice,
            @ToolParam(description = "Discount percentage, e.g. 20 for 20%") double discountPercent) {
        double discount = originalPrice * (discountPercent / 100);
        return Map.of(
            "original", originalPrice,
            "discount", discount,
            "final", originalPrice - discount
        );
    }
}
```

**Step 3 — Wire tools into ChatClient**
```java
@Configuration
public class AgentConfig {

    @Bean
    public ChatClient toolEnabledChatClient(ChatClient.Builder builder, OrderTools orderTools) {
        return builder
            .defaultTools(orderTools)
            .build();
    }
}
```

**Step 4 — Agent service**
```java
@Service
public class OrderAssistantService {

    private final ChatClient chatClient;

    public OrderAssistantService(ChatClient toolEnabledChatClient) {
        this.chatClient = toolEnabledChatClient;
    }

    public String assist(String userMessage) {
        return chatClient.prompt()
            .system("You are a helpful order assistant. Use the available tools to answer " +
                    "questions about orders and pricing. Always confirm the order ID with the user " +
                    "before calling tools if it's not provided.")
            .user(userMessage)
            .call()
            .content();
    }
}
```

**Step 5 — REST endpoint**
```java
@RestController
@RequestMapping("/api/assistant")
public class AssistantController {

    private final OrderAssistantService assistantService;

    public AssistantController(OrderAssistantService assistantService) {
        this.assistantService = assistantService;
    }

    @PostMapping
    public Map<String, String> chat(@RequestBody Map<String, String> body) {
        return Map.of("response", assistantService.assist(body.get("message")));
    }
}
```

**Step 6 — Experiments**
1. Send: `"Where is my order ORD-001?"` — observe the LLM calls `getOrderStatus` and synthesizes the answer.
2. Send: `"What's the final price of a $200 item with 15% discount?"` — observe `calculateDiscount` is called.
3. Send: `"Where is my order?"` (no ID) — observe the LLM asks for the order ID before calling the tool.
4. Send: `"Check order ORD-001 and also calculate discount for $150 at 20%"` — observe multi-tool call in one turn.
5. Add a log inside each tool method — verify when and how many times tools are called.

---

## 4. Resources

- **[Spring AI Function Calling](https://docs.spring.io/spring-ai/reference/api/tools.html)** — Official docs for `@Tool`, `@ToolParam`, and bean-based function registration.
- **[Anthropic Tool Use Guide](https://docs.anthropic.com/en/docs/build-with-claude/tool-use)** — Deep dive into the underlying protocol: tool definitions, tool_use blocks, tool_result blocks. Essential for debugging.
- **[OpenAI Function Calling Cookbook](https://cookbook.openai.com/examples/how_to_call_functions_with_chat_models)** — Model-agnostic patterns; the concepts apply directly to Spring AI regardless of provider.

---

## 5. Trending Context

**Tool use is now the standard way to build AI features in production systems.** The pattern has replaced brittle prompt-based JSON parsing for structured outputs. In 2025, models like Claude 3.5 Sonnet and GPT-4o have become highly reliable at choosing the right tool and generating correct arguments. The frontier has moved to *parallel tool calling* (calling multiple tools simultaneously), *tool chaining* (using the output of one tool as input to another), and *computer use* (tools that control a browser or desktop). Spring AI 1.0 supports parallel tool calls natively.

---

## 6. Community Tip

**Tools are a security boundary — treat them like a public API.** The LLM generates tool arguments based on user input. A malicious user can craft prompts to manipulate which tools get called and with what parameters (prompt injection). Apply the same defenses you would to any user-facing endpoint: validate and sanitize all tool inputs server-side, enforce authorization (don't let the LLM call `deleteOrder` unless the user is authorized), and log every tool invocation with the originating user. Never trust the LLM's argument generation as safe without validation — it's user-controlled input. Think of it as the LLM being a proxy between your user and your services.

---

## 7. Tomorrow's Preview

**Day 7** will cover **LLM Agents & Multi-Step Reasoning** — building autonomous agents that can plan, use tools iteratively, and complete multi-step tasks. You'll see how ReAct (Reason + Act) loops work and implement a simple agent using Spring AI's agent abstractions.

---

*Built progressively. Each day connects to the next.*
