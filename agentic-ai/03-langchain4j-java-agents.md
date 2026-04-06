# Agentic AI — Day 3: LangChain4j — Java-Native Agent Framework
**Track:** Agentic AI (Implementation-Focused)
**Time Target:** 1–2 hours

---

## 1. Today's Focus
**LangChain4j Deep Dive — AI Services, Agents, and Memory in Pure Java**

Spring AI is great for Spring Boot integration. LangChain4j is a different philosophy: a rich, Java-native library with AI Services (type-safe agent interfaces), built-in RAG, streaming, and multi-provider support. Today you rebuild yesterday's finance agent using LangChain4j and learn what it does differently — and when to prefer it over Spring AI.

---

## 2. LangChain4j Core Concepts

```
┌──────────────────────────────────────────────┐
│              AI Service Interface             │  ← Type-safe, declarative
│  (Java interface + annotations)               │
└────────────────┬─────────────────────────────┘
                 │ proxied by LangChain4j
    ┌────────────┼────────────┐
    ▼            ▼            ▼
┌────────┐  ┌────────┐  ┌──────────┐
│  Chat  │  │ Tools  │  │  Memory  │
│ Model  │  │ (Java  │  │ (Message │
│(Claude)│  │methods)│  │  Store)  │
└────────┘  └────────┘  └──────────┘
```

---

## 3. Building the Agent

### 3.1 AI Service Interface (Declarative Agent)

```java
public interface FinanceAssistant {

    @SystemMessage("""
        You are a personal finance assistant. Help users understand their
        spending, check balances, and manage budgets.

        Rules:
        - Always verify account existence before querying transactions.
        - Format amounts as currency with 2 decimal places.
        - Default to last 30 days for spending queries.
        - Be concise. Use bullet points for summaries.
        """)
    String chat(@MemoryId String conversationId, @UserMessage String message);
}
```

### 3.2 Tool Definitions

```java
public class AccountTools {

    private final AccountRepository accountRepo;
    private final TransactionRepository txnRepo;

    @Tool("Get all accounts with balances for a user")
    public List<Account> getAccounts(@P("User ID") Long userId) {
        return accountRepo.findByUserId(userId);
    }

    @Tool("Get recent transactions, optionally filtered by category")
    public List<Transaction> getTransactions(
            @P("Account ID") Long accountId,
            @P("Category filter or 'ALL'") String category,
            @P("Days to look back") int days) {
        LocalDate since = LocalDate.now().minusDays(days);
        if ("ALL".equals(category)) {
            return txnRepo.findByAccountIdAndDateAfter(accountId, since);
        }
        return txnRepo.findByAccountIdAndCategoryAndDateAfter(accountId, category, since);
    }

    @Tool("Get spending summary grouped by category")
    public List<SpendingSummary> getSpendingSummary(
            @P("Account ID") Long accountId,
            @P("Start date YYYY-MM-DD") String startDate,
            @P("End date YYYY-MM-DD") String endDate) {
        return txnRepo.getSpendingSummary(accountId,
            LocalDate.parse(startDate), LocalDate.parse(endDate));
    }

    @Tool("Calculate percentage change between two amounts")
    public String calculateChange(
            @P("Previous amount") double previous,
            @P("Current amount") double current) {
        double change = ((current - previous) / previous) * 100;
        return String.format("%.1f%% %s", Math.abs(change),
            change >= 0 ? "increase" : "decrease");
    }
}
```

### 3.3 Wiring It Together

```java
@Configuration
public class LangChain4jConfig {

    @Bean
    public ChatLanguageModel chatModel() {
        return AnthropicChatModel.builder()
            .apiKey(System.getenv("ANTHROPIC_API_KEY"))
            .modelName("claude-sonnet-4-20250514")
            .maxTokens(4096)
            .build();
    }

    @Bean
    public ChatMemoryProvider memoryProvider() {
        // Per-conversation memory, max 20 messages
        return conversationId -> MessageWindowChatMemory.builder()
            .maxMessages(20)
            .id(conversationId)
            .build();
    }

    @Bean
    public FinanceAssistant financeAssistant(
            ChatLanguageModel model,
            ChatMemoryProvider memoryProvider,
            AccountTools accountTools) {

        return AiServices.builder(FinanceAssistant.class)
            .chatLanguageModel(model)
            .chatMemoryProvider(memoryProvider)
            .tools(accountTools)
            .build();
    }
}
```

### 3.4 REST Controller

```java
@RestController
@RequestMapping("/v1/finance")
@RequiredArgsConstructor
public class FinanceController {

    private final FinanceAssistant assistant;

    @PostMapping("/chat")
    public Map<String, String> chat(@RequestBody ChatRequest request) {
        String response = assistant.chat(request.conversationId(), request.message());
        return Map.of("response", response, "conversationId", request.conversationId());
    }
}
```

---

## 4. Streaming Responses

```java
public interface FinanceAssistantStreaming {

    @SystemMessage("You are a personal finance assistant...")
    TokenStream chat(@MemoryId String conversationId, @UserMessage String message);
}

// Build streaming version
FinanceAssistantStreaming streamingAssistant = AiServices.builder(FinanceAssistantStreaming.class)
    .streamingChatLanguageModel(streamingModel)
    .chatMemoryProvider(memoryProvider)
    .tools(accountTools)
    .build();

// SSE endpoint
@GetMapping(value = "/chat/stream", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
public Flux<String> chatStream(@RequestParam String conversationId,
                                @RequestParam String message) {
    TokenStream stream = streamingAssistant.chat(conversationId, message);

    return Flux.create(sink -> {
        stream.onNext(sink::next)
              .onComplete(token -> sink.complete())
              .onError(sink::error)
              .start();
    });
}
```

---

## 5. Structured Output Extraction

```java
// LangChain4j can extract structured objects directly
public interface TransactionClassifier {

    @SystemMessage("Classify the following transaction description into a category.")
    @UserMessage("Transaction: {{description}}, Amount: {{amount}}")
    TransactionCategory classify(
        @V("description") String description,
        @V("amount") BigDecimal amount);
}

public record TransactionCategory(
    String category,        // FOOD, TRANSPORT, ENTERTAINMENT, etc.
    double confidence,      // 0.0 - 1.0
    String reasoning        // Why this category
) {}

// Usage — no JSON parsing needed!
TransactionCategory result = classifier.classify("Uber ride to airport", new BigDecimal("45.00"));
// → TransactionCategory("TRANSPORT", 0.95, "Uber is a ride-sharing/transport service")
```

---

## 6. RAG Integration (Built-In)

```java
// LangChain4j has built-in RAG support
public interface FinanceAdvisor {

    @SystemMessage("You are a financial advisor. Use the provided context to answer questions.")
    String advise(@MemoryId String id, @UserMessage String question);
}

// Wire with RAG content retriever
EmbeddingModel embeddingModel = new AllMiniLmL6V2EmbeddingModel();
EmbeddingStore<TextSegment> store = new InMemoryEmbeddingStore<>();

ContentRetriever retriever = EmbeddingStoreContentRetriever.builder()
    .embeddingStore(store)
    .embeddingModel(embeddingModel)
    .maxResults(5)
    .minScore(0.7)
    .build();

FinanceAdvisor advisor = AiServices.builder(FinanceAdvisor.class)
    .chatLanguageModel(model)
    .contentRetriever(retriever)   // Automatically injects retrieved context
    .build();
```

---

## 7. Spring AI vs LangChain4j — When to Use Which

| Feature | Spring AI | LangChain4j |
|---------|-----------|-------------|
| **Spring Boot integration** | Native, auto-config | Good (starter available) |
| **Agent interface style** | ChatClient fluent API | AI Services (interface-based) |
| **Structured output** | BeanOutputConverter | Automatic from return type |
| **Streaming** | Flux-based | TokenStream callbacks |
| **RAG** | Advisors pattern | ContentRetriever built-in |
| **Memory** | ChatMemory + Advisors | ChatMemoryProvider per-conversation |
| **Multi-provider** | One provider per starter | All providers in one dependency |
| **Maturity** | Newer, rapidly evolving | More mature, broader features |

**Recommendation:** Use Spring AI when you're already in Spring Boot and want seamless integration. Use LangChain4j when you need richer agent abstractions, multi-provider support, or the AI Services pattern.

---

## 8. Hands-On Exercise

1. Add LangChain4j dependencies to your project.
2. Create the `FinanceAssistant` AI Service interface.
3. Implement `AccountTools` with mock data.
4. Wire everything in `LangChain4jConfig`.
5. Test multi-turn conversations:
   - "Show my accounts" → "What did I spend on food last week?" → "Compare to the previous week"
6. Add the `TransactionClassifier` structured output extractor and test it.
7. Compare: same question, Spring AI vs LangChain4j — note API differences.

---

## 9. Key Takeaways

1. **AI Services are interface-driven** — define what you want, LangChain4j builds the implementation.
2. **Structured output is automatic** — return a record from your interface, LangChain4j extracts it.
3. **Memory is per-conversation** — `ChatMemoryProvider` creates isolated memory per `@MemoryId`.
4. **Both Spring AI and LangChain4j can coexist** — use each for its strengths in the same project.

---

*Day 3. The best abstraction is the one that lets you think about the problem, not the framework.*
