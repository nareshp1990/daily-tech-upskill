# Day 13 — Production LLM Architecture & Cost Optimisation
**Date:** 2026-03-20
**Phase:** 1 — Generative AI & Agentic AI
**Time Target:** 1–2 hours

---

## 1. Today's Focus Topic
**Production LLM Architecture & Cost Optimisation — Building for Scale Without Breaking the Budget**

LLMs are expensive. A naive implementation that routes every request to the most capable (most expensive) model, never caches, and has no fallback will either break the budget or break under load. Today you design a production-grade LLM architecture: model routing (match task complexity to model cost), semantic caching (don't pay for the same question twice), rate limiting (protect your API budget), fallback chains (survive provider outages), and cost attribution (know which feature is burning money). Analogy: this is the same discipline as database query optimisation — you already size connection pools, add indexes, and cache hot queries; today you apply the same rigour to LLM calls.

---

## 2. Key Concepts

### Model Routing (Tiered Models)
Not every request needs Claude Opus or GPT-4o. Route by complexity:
| Tier | Model (example) | Cost | Use for |
|------|----------------|------|---------|
| Fast/cheap | Claude Haiku / GPT-4o-mini | ~$0.25/M input | Classification, routing, simple Q&A |
| Mid-tier | Claude Sonnet | ~$3/M input | RAG answers, tool use, structured output |
| Powerful | Claude Opus | ~$15/M input | Complex reasoning, multi-step agents |

Route programmatically: classify the request first (cheap call) → send to appropriate tier.

### Semantic Caching
Cache not by exact string match, but by semantic similarity. If a user asks *"What's my order status?"* and another asks *"Can you check my order?"* — they're the same intent. Store embedding + response pairs in a vector DB; on each request, check if a semantically similar question was answered recently.

### Rate Limiting & Budget Guards
- Per-user rate limits: prevent one user from exhausting your API budget.
- Global token budget: hard-stop if daily spend exceeds threshold.
- Request queuing: buffer burst traffic rather than dropping requests.

### Fallback Chains
```
Primary: Claude Sonnet → fail → Fallback: GPT-4o-mini → fail → Static response
```
Resilience4j `CircuitBreaker` + `Fallback` is the natural fit — you already know this from Day 3.

### Cost Attribution
Tag every LLM call with: feature name, user tier, request type. Aggregate in Micrometer / Prometheus. Know your cost-per-feature before your finance team asks.

---

## 3. Hands-On Task

### Goal: Add model routing, semantic caching, and cost guardrails to the assistant

**Step 1 — Model router: classify request complexity**
```java
@Component
public class ModelRouter {

    private final ChatClient classifierClient;
    private final BeanOutputConverter<ComplexityScore> converter =
        new BeanOutputConverter<>(ComplexityScore.class);

    public record ComplexityScore(String tier) {} // SIMPLE, STANDARD, COMPLEX

    public ModelRouter(ChatClient.Builder builder) {
        // Haiku for classification — fast and cheap
        this.classifierClient = builder
            .defaultOptions(AnthropicChatOptions.builder()
                .withModel("claude-haiku-4-5-20251001")
                .withMaxTokens(20)
                .build())
            .build();
    }

    public String selectModel(String userMessage) {
        String result = classifierClient.prompt()
            .system("Classify the complexity of this request. " +
                    "SIMPLE: greeting, single fact lookup, yes/no question. " +
                    "STANDARD: multi-step lookup, RAG answer, structured extraction. " +
                    "COMPLEX: multi-turn reasoning, planning, synthesis of many sources. " +
                    converter.getFormat())
            .user(userMessage)
            .call()
            .content();

        return switch (converter.convert(result).tier()) {
            case "SIMPLE"   -> "claude-haiku-4-5-20251001";
            case "COMPLEX"  -> "claude-opus-4-6";
            default         -> "claude-sonnet-4-6";
        };
    }
}
```

**Step 2 — Semantic cache**
```java
@Service
public class SemanticCacheService {

    private final VectorStore vectorStore;
    private final EmbeddingModel embeddingModel;
    private static final double SIMILARITY_THRESHOLD = 0.92;
    private static final Duration CACHE_TTL = Duration.ofHours(1);

    public Optional<String> lookup(String question) {
        List<Document> results = vectorStore.similaritySearch(
            SearchRequest.query(question)
                .withTopK(1)
                .withSimilarityThreshold(SIMILARITY_THRESHOLD)
                .withFilterExpression("type == 'cache' AND expiresAt > '" + Instant.now() + "'")
        );
        return results.isEmpty() ? Optional.empty()
            : Optional.of(results.get(0).getMetadata().get("answer").toString());
    }

    public void store(String question, String answer) {
        Document cached = new Document(question, Map.of(
            "type",      "cache",
            "answer",    answer,
            "expiresAt", Instant.now().plus(CACHE_TTL).toString()
        ));
        vectorStore.add(List.of(cached));
    }
}
```

**Step 3 — Budget guard with Micrometer**
```java
@Component
public class TokenBudgetGuard {

    private static final long DAILY_TOKEN_BUDGET = 1_000_000L; // 1M tokens/day
    private final AtomicLong dailyTokensUsed = new AtomicLong(0);
    private final MeterRegistry meterRegistry;

    public TokenBudgetGuard(MeterRegistry meterRegistry) {
        this.meterRegistry = meterRegistry;
        // Reset daily at midnight via @Scheduled
    }

    public void checkBudget(long estimatedTokens) {
        long current = dailyTokensUsed.get();
        if (current + estimatedTokens > DAILY_TOKEN_BUDGET) {
            log.error("Daily token budget exhausted: {} / {}", current, DAILY_TOKEN_BUDGET);
            throw new BudgetExhaustedException("Daily LLM budget exceeded. Please try again tomorrow.");
        }
    }

    public void recordUsage(long tokensUsed, String featureName) {
        dailyTokensUsed.addAndGet(tokensUsed);
        meterRegistry.counter("llm.budget.tokens", "feature", featureName)
            .increment(tokensUsed);
    }

    @Scheduled(cron = "0 0 0 * * *")
    public void resetDailyBudget() {
        log.info("Resetting daily token budget (used: {})", dailyTokensUsed.getAndSet(0));
    }
}
```

**Step 4 — Fallback chain with Resilience4j**
```java
@Service
public class ResilientLlmService {

    private final ChatClient primaryClient;   // Claude Sonnet
    private final ChatClient fallbackClient;  // GPT-4o-mini

    @CircuitBreaker(name = "primary-llm", fallbackMethod = "callFallback")
    @RateLimiter(name = "llm-rate-limit")
    public String call(String systemPrompt, String userMessage) {
        return primaryClient.prompt()
            .system(systemPrompt)
            .user(userMessage)
            .call()
            .content();
    }

    public String callFallback(String systemPrompt, String userMessage, Throwable ex) {
        log.warn("Primary LLM unavailable ({}), using fallback", ex.getMessage());
        return fallbackClient.prompt()
            .system(systemPrompt)
            .user(userMessage)
            .call()
            .content();
    }
}
```

**application.yml additions:**
```yaml
resilience4j:
  circuitbreaker:
    instances:
      primary-llm:
        slidingWindowSize: 10
        failureRateThreshold: 50
        waitDurationInOpenState: 30s
  ratelimiter:
    instances:
      llm-rate-limit:
        limitForPeriod: 20
        limitRefreshPeriod: 1s
        timeoutDuration: 500ms
```

**Step 5 — Composed production service**
```java
@Service
public class ProductionAssistantService {

    private final ModelRouter router;
    private final SemanticCacheService cache;
    private final TokenBudgetGuard budgetGuard;
    private final ChatClient.Builder clientBuilder;

    public String assist(String featureName, String userMessage) {
        // 1. Check semantic cache
        Optional<String> cached = cache.lookup(userMessage);
        if (cached.isPresent()) {
            log.info("Cache HIT for: {}", userMessage);
            return cached.get();
        }

        // 2. Check budget (rough estimate: ~500 tokens average)
        budgetGuard.checkBudget(500);

        // 3. Route to appropriate model
        String model = router.selectModel(userMessage);

        // 4. Call LLM
        ChatClient client = clientBuilder
            .defaultOptions(AnthropicChatOptions.builder().withModel(model).build())
            .build();

        String response = client.prompt().user(userMessage).call().content();

        // 5. Record cost + cache result
        budgetGuard.recordUsage(500, featureName); // replace with actual token count from response
        cache.store(userMessage, response);

        return response;
    }
}
```

**Step 6 — Experiments**
1. Send 5 identical questions — verify the cache returns instantly after the first call (no LLM invocation on hits).
2. Send a simple question (`"Hello"`) vs a complex one — verify the router selects Haiku vs Sonnet.
3. Set `DAILY_TOKEN_BUDGET = 100` — trigger the budget guard and verify the error response.
4. Simulate a primary LLM failure (wrong API key) — verify the circuit breaker opens and the fallback responds.

---

## 4. Resources

- **[Spring AI Models Configuration](https://docs.spring.io/spring-ai/reference/api/chatmodel.html)** — How to configure multiple model providers and switch between them.
- **[Anthropic Model Pricing](https://www.anthropic.com/pricing)** — Current pricing for all Claude models; essential for cost modelling.
- **[Resilience4j Docs](https://resilience4j.readme.io/docs/circuitbreaker)** — Circuit breaker, rate limiter, retry configuration — you know this from Day 3; today you apply it to LLM specifically.

---

## 5. Trending Context

**Cost optimisation is now a primary concern for teams scaling LLM features.** Early-stage teams use the most capable model everywhere; at scale, this is unsustainable. The industry pattern: tiered routing (Haiku/Gemini Flash for triage, Sonnet/GPT-4o for standard tasks, Opus/o1 for complex reasoning) cuts costs by 60–80% without quality regression. Semantic caching adds another 20–40% reduction for consumer-facing features with repetitive queries. The infrastructure trend: **LLM gateways** (LiteLLM, Portkey, OpenRouter) that handle routing, caching, rate limiting, and cost tracking centrally across teams — the API gateway pattern applied to LLM traffic.

---

## 6. Community Tip

**Build your semantic cache with an intentional invalidation strategy from day one.** Cached LLM responses go stale when: your data changes (a cached order status is now wrong), your prompt changes (cached response doesn't reflect new behaviour), or TTL expires. Design for cache invalidation the same way you design for Redis invalidation: tag cached entries by data source (e.g., `"source: orders-db"`), and invalidate by tag when the underlying data changes via a Kafka consumer. A stale cache that silently returns wrong answers is worse than no cache at all — measure cache hit quality, not just hit rate.

---

## 7. Tomorrow's Preview

**Day 14** will cover **Fine-Tuning & Model Customisation** — when and why to fine-tune, dataset preparation, using the Anthropic/OpenAI fine-tuning APIs, and adapting pre-trained models for your domain with Spring AI.

---

*Built progressively. Each day connects to the next.*
