# Agentic AI — Day 15: Production Agent Architecture
**Track:** Agentic AI (Implementation-Focused)
**Time Target:** 1–2 hours

---

## 1. Today's Focus
**Deploying Agents to Production — Observability, Scaling, Cost Control, and Reliability**

Building an agent that works locally is Day 1. Running it in production — handling 10K concurrent users, keeping latency under 5 seconds, staying within budget, and debugging when things go wrong — is a completely different challenge. Today covers the architecture and operational patterns that separate a demo from a product.

---

## 2. Production Architecture

```
                    Client Apps
                         │
                ┌────────▼────────┐
                │  API Gateway    │  Auth, rate limit, request routing
                │  (Kong/Nginx)   │
                └────────┬────────┘
                         │
                ┌────────▼────────┐
                │  Agent Service  │  Stateless Spring Boot / FastAPI
                │  (Horizontally  │  Pods behind K8s Service
                │   Scaled)       │
                └───┬────┬────┬───┘
                    │    │    │
          ┌─────────┘    │    └─────────┐
          ▼              ▼              ▼
   ┌────────────┐ ┌────────────┐ ┌────────────┐
   │ LLM API    │ │ Tool Svc   │ │ Vector DB  │
   │ (Claude /  │ │ (MCP /     │ │ (pgvector/ │
   │  OpenAI)   │ │  REST)     │ │  Pinecone) │
   └────────────┘ └────────────┘ └────────────┘
          │
   ┌──────┴──────┐
   │ Redis       │  Conversation memory, rate limits, caching
   └─────────────┘
   ┌─────────────┐
   │ Postgres    │  Agent logs, audit trail, user data
   └─────────────┘
   ┌─────────────┐
   │ OTel + Jaeger│  Traces, metrics, cost tracking
   └─────────────┘
```

---

## 3. Stateless Agent Design

```java
// Agents MUST be stateless for horizontal scaling
// State lives in Redis (memory) and Postgres (audit)

@RestController
@RequiredArgsConstructor
public class AgentController {

    private final ChatClient agentClient;
    private final ChatMemory redisMemory;     // Redis-backed
    private final AuditService auditService;

    @PostMapping("/v1/agent/chat")
    public AgentResponse chat(@RequestBody AgentRequest req,
                               @AuthenticationPrincipal UserPrincipal user) {
        // Load conversation from Redis
        String conversationId = req.conversationId();

        // Execute agent
        String response = agentClient.prompt()
            .system(s -> s.param("userId", user.getId()))
            .user(req.message())
            .advisors(new MessageChatMemoryAdvisor(redisMemory, conversationId, 20))
            .call()
            .content();

        // Audit log (async)
        auditService.logAsync(user.getId(), conversationId, req.message(), response);

        return new AgentResponse(conversationId, response);
    }
}
```

---

## 4. Cost Control

### 4.1 Model Routing (Cheap Model First)

```java
@Service
public class ModelRouter {

    private final ChatClient haikuClient;    // Fast, cheap: $0.25/$1.25 per 1M tokens
    private final ChatClient sonnetClient;   // Balanced: $3/$15 per 1M tokens

    public String route(String message, String complexity) {
        return switch (complexity) {
            case "simple" -> haikuClient.prompt().user(message).call().content();
            case "complex" -> sonnetClient.prompt().user(message).call().content();
            default -> {
                // Auto-classify complexity
                String classification = haikuClient.prompt()
                    .system("Classify: SIMPLE (factual, lookup) or COMPLEX (reasoning, multi-step). Reply one word.")
                    .user(message)
                    .call().content();
                yield classification.contains("SIMPLE")
                    ? haikuClient.prompt().user(message).call().content()
                    : sonnetClient.prompt().user(message).call().content();
            }
        };
    }
}
```

### 4.2 Response Caching

```java
@Service
@RequiredArgsConstructor
public class SemanticCache {

    private final VectorStore vectorStore;
    private final RedisTemplate<String, String> redis;

    public Optional<String> checkCache(String query) {
        // Semantic search: find similar past queries
        List<Document> similar = vectorStore.similaritySearch(
            SearchRequest.builder().query(query).topK(1).similarityThreshold(0.95).build());

        if (!similar.isEmpty()) {
            String cachedResponse = redis.opsForValue()
                .get("agent_cache:" + similar.get(0).getId());
            if (cachedResponse != null) return Optional.of(cachedResponse);
        }
        return Optional.empty();
    }

    public void cacheResponse(String query, String response) {
        Document doc = new Document(query);
        vectorStore.add(List.of(doc));
        redis.opsForValue().set("agent_cache:" + doc.getId(), response, Duration.ofHours(24));
    }
}
```

### 4.3 Budget Limits

```java
@Component
public class CostTracker {

    private final MeterRegistry metrics;

    public void trackLLMCall(String model, int inputTokens, int outputTokens) {
        double cost = calculateCost(model, inputTokens, outputTokens);
        metrics.counter("agent.llm.cost",
            "model", model).increment(cost);
        metrics.counter("agent.llm.tokens.input",
            "model", model).increment(inputTokens);
        metrics.counter("agent.llm.tokens.output",
            "model", model).increment(outputTokens);
    }

    // Prometheus alert: agent_llm_cost_total > 100 (daily budget $100)
}
```

---

## 5. Observability for Agents

### 5.1 Tracing Agent Steps

```java
@Service
@RequiredArgsConstructor
public class ObservableAgentService {

    private final Tracer tracer;
    private final ChatClient agent;

    public String executeWithTracing(String conversationId, String message) {
        Span agentSpan = tracer.nextSpan()
            .name("agent.execute")
            .tag("conversation.id", conversationId)
            .start();

        try (Tracer.SpanInScope ws = tracer.withSpan(agentSpan)) {
            String response = agent.prompt()
                .user(message)
                .call()
                .content();

            agentSpan.tag("response.length", String.valueOf(response.length()));
            return response;
        } catch (Exception e) {
            agentSpan.error(e);
            throw e;
        } finally {
            agentSpan.end();
        }
    }
}
```

### 5.2 Key Metrics to Monitor

| Metric | Alert Threshold | Why |
|--------|----------------|-----|
| `agent.request.latency.p99` | > 10s | User experience degrades |
| `agent.llm.cost.daily` | > budget | Prevent bill shock |
| `agent.tool.error.rate` | > 5% | Tools may be broken |
| `agent.iterations.per_request` | > 8 avg | Agent may be looping |
| `agent.fallback.rate` | > 10% | Guardrails triggering too often |
| `agent.memory.miss.rate` | > 50% | Memory not persisting correctly |

---

## 6. Streaming Responses (UX Improvement)

```java
@GetMapping(value = "/v1/agent/stream", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
public Flux<String> streamChat(@RequestParam String conversationId,
                                @RequestParam String message) {
    return agentClient.prompt()
        .user(message)
        .advisors(new MessageChatMemoryAdvisor(redisMemory, conversationId, 20))
        .stream()
        .content();
}
```

```python
# Python FastAPI streaming
from fastapi import FastAPI
from fastapi.responses import StreamingResponse

app = FastAPI()

@app.post("/v1/agent/stream")
async def stream_agent(request: AgentRequest):
    async def generate():
        async for chunk in agent.astream(
            {"messages": [("user", request.message)]},
            {"configurable": {"thread_id": request.conversation_id}}
        ):
            if "messages" in chunk:
                for msg in chunk["messages"]:
                    yield f"data: {msg.content}\n\n"
        yield "data: [DONE]\n\n"

    return StreamingResponse(generate(), media_type="text/event-stream")
```

---

## 7. Error Handling & Retries

```java
@Service
public class ResilientAgentService {

    @CircuitBreaker(name = "llm-api", fallbackMethod = "fallbackResponse")
    @Retry(name = "llm-api")
    @TimeLimiter(name = "llm-api")
    public CompletableFuture<String> executeAgent(String message) {
        return CompletableFuture.supplyAsync(() ->
            agentClient.prompt().user(message).call().content());
    }

    public CompletableFuture<String> fallbackResponse(String message, Throwable t) {
        log.error("Agent failed: {}", t.getMessage());
        return CompletableFuture.completedFuture(
            "I'm experiencing difficulties right now. Please try again in a moment.");
    }
}
```

---

## 8. Hands-On Exercise

1. Add Redis-backed `ChatMemory` to your Spring AI agent.
2. Implement the `ModelRouter` — route simple queries to Haiku, complex to Sonnet.
3. Add `CostTracker` with Micrometer metrics and observe token usage.
4. Implement response streaming via SSE endpoint.
5. Add Resilience4j circuit breaker around the LLM API call.

---

## 9. Key Takeaways

1. **Agents must be stateless** — state in Redis/Postgres, not in-memory.
2. **Model routing cuts cost 50-80%** — most queries don't need the expensive model.
3. **Semantic caching avoids redundant LLM calls** — cache near-duplicate questions.
4. **Stream responses for better UX** — users see progress instead of waiting.
5. **Observe everything** — cost, latency, tool errors, iteration counts, and fallback rates.

---

*Day 15. A production agent is an agent that can fail gracefully, scale horizontally, and stay within budget.*
