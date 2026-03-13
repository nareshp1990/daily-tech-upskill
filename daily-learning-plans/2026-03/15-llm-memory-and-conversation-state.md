# Day 8 — LLM Memory & Conversation State
**Date:** 2026-03-15
**Phase:** 1 — Generative AI & Agentic AI
**Time Target:** 1–2 hours

---

## 1. Today's Focus Topic
**LLM Memory & Conversation State — Giving Agents a Brain That Remembers**

Days 6 and 7 built agents that reason and act within a single request. The moment the request ends, all context is gone. Real assistants remember: they recall what you discussed last week, know your preferences, and don't make you repeat yourself. Today you implement two complementary memory types: **short-term memory** (in-context conversation history — what happened in this session) and **long-term memory** (vector-store-backed — what happened across sessions). Analogy: short-term memory is your working desk; long-term memory is your filing cabinet. The agent uses both to answer questions intelligently.

---

## 2. Key Concepts

### Why LLMs Have No Native Memory
Each LLM call is stateless — the model has no awareness of prior calls unless you include that history in the prompt. "Memory" is an application-layer concern: your code decides what to include in each prompt's context window.

### Short-Term Memory (In-Context / Conversation History)
The simplest form of memory: append previous messages (user + assistant turns) to every new prompt. Limitations:
- Bounded by the context window (e.g., 128K tokens for Claude, ~100K for GPT-4o).
- Costs more as history grows (you pay for all tokens in the window).
- Strategy: keep the last N messages, or summarise older history into a condensed block.

Spring AI provides `MessageChatMemoryAdvisor` which automatically manages a `ChatMemory` store (in-memory or JDBC-backed) and injects history into every prompt.

### Long-Term Memory (Vector-Store-Backed)
For cross-session persistence. Key ideas:
- After each conversation, extract important facts and embed + store them in a vector DB.
- On every new session, retrieve the most relevant memories via semantic search and inject them into the system prompt.
- This is essentially RAG applied to the agent's own past experiences.

### Memory Types Compared
| Type | Scope | Storage | Cost |
|------|-------|---------|------|
| In-context history | Current session | In-memory / DB | Token cost per turn |
| Summary memory | Current session | In-memory | Cheaper — compressed |
| Long-term (vector) | Cross-session | pgvector / Redis | Embedding + retrieval |
| Entity memory | Cross-session | Structured DB | Low — key-value lookups |

### Context Window Management
The biggest practical challenge. Strategies:
1. **Sliding window** — keep last N messages, drop older ones.
2. **Summarisation** — when history exceeds a threshold, ask the LLM to summarise it into one condensed message and replace the history with the summary.
3. **Selective retrieval** — store all history in a vector DB; retrieve only the top-K most relevant past messages for each new turn.

---

## 3. Hands-On Task

### Goal: Build a customer support assistant with session memory and cross-session long-term memory

**Step 1 — Add Spring AI memory dependency**
```xml
<!-- Spring AI JDBC memory (uses your existing DataSource) -->
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-advisors-vector-store</artifactId>
</dependency>
```

**Step 2 — In-memory conversation store (simplest start)**
```java
@Configuration
public class MemoryConfig {

    @Bean
    public ChatMemory chatMemory() {
        return new InMemoryChatMemory();
        // Swap for JdbcChatMemory for persistence across restarts
    }
}
```

**Step 3 — ChatClient with memory advisor**
```java
@Service
public class SupportAssistantService {

    private final ChatClient chatClient;
    private final ChatMemory chatMemory;

    public SupportAssistantService(ChatClient.Builder builder, ChatMemory chatMemory) {
        this.chatMemory = chatMemory;
        this.chatClient = builder
            .defaultSystem("""
                You are a helpful customer support assistant.
                You remember what the user told you earlier in this session.
                Use that context to avoid asking for information already provided.
                """)
            .defaultAdvisors(
                new MessageChatMemoryAdvisor(chatMemory)  // injects history automatically
            )
            .build();
    }

    public String chat(String sessionId, String userMessage) {
        return chatClient.prompt()
            .advisors(advisor -> advisor.param(
                AbstractChatMemoryAdvisor.CHAT_MEMORY_CONVERSATION_ID_KEY, sessionId))
            .user(userMessage)
            .call()
            .content();
    }
}
```

**Step 4 — REST endpoint with session ID**
```java
@RestController
@RequestMapping("/api/support")
public class SupportController {

    private final SupportAssistantService supportService;

    public SupportController(SupportAssistantService supportService) {
        this.supportService = supportService;
    }

    @PostMapping("/{sessionId}")
    public Map<String, String> chat(
            @PathVariable String sessionId,
            @RequestBody Map<String, String> body) {
        return Map.of("response", supportService.chat(sessionId, body.get("message")));
    }
}
```

**Step 5 — Long-term memory: extract and store facts after each session**
```java
@Service
public class LongTermMemoryService {

    private final ChatClient chatClient;
    private final VectorStore vectorStore;

    public LongTermMemoryService(ChatClient.Builder builder, VectorStore vectorStore) {
        this.chatClient = builder.build();
        this.vectorStore = vectorStore;
    }

    // Call this at end of session to extract memorable facts
    public void consolidateMemory(String customerId, List<Message> history) {
        String historyText = history.stream()
            .map(m -> m.getMessageType() + ": " + m.getContent())
            .collect(Collectors.joining("\n"));

        String facts = chatClient.prompt()
            .system("Extract 3-5 important facts about the customer from this conversation. " +
                    "Output as a bulleted list. Only include durable facts (preferences, issues, " +
                    "account details) — not transient details like today's date.")
            .user(historyText)
            .call()
            .content();

        // Store as a document associated with this customer
        Document memory = new Document(facts, Map.of("customerId", customerId,
                                                      "storedAt", LocalDate.now().toString()));
        vectorStore.add(List.of(memory));
    }

    // Call this at the start of a new session to recall relevant context
    public String recallContext(String customerId, String currentQuestion) {
        List<Document> memories = vectorStore.similaritySearch(
            SearchRequest.query(currentQuestion)
                .withFilterExpression("customerId == '" + customerId + "'")
                .withTopK(3)
        );

        if (memories.isEmpty()) return "";

        return memories.stream()
            .map(Document::getContent)
            .collect(Collectors.joining("\n---\n"));
    }
}
```

**Step 6 — Inject long-term memory into session start**
```java
public String startSession(String customerId, String sessionId, String firstMessage) {
    String pastContext = longTermMemoryService.recallContext(customerId, firstMessage);

    String systemPrompt = "You are a helpful support assistant.";
    if (!pastContext.isEmpty()) {
        systemPrompt += "\n\nWhat you know about this customer from past sessions:\n" + pastContext;
    }

    return chatClient.prompt()
        .system(systemPrompt)
        .advisors(advisor -> advisor.param(
            AbstractChatMemoryAdvisor.CHAT_MEMORY_CONVERSATION_ID_KEY, sessionId))
        .user(firstMessage)
        .call()
        .content();
}
```

**Step 7 — Experiments**
1. Start two turns with the same `sessionId`: tell the assistant your name in turn 1, ask it to use your name in turn 2 — verify it remembers.
2. Start a new session with a different `sessionId` — verify history does NOT bleed across sessions.
3. Call `consolidateMemory` after a session, then `recallContext` in a new session — verify facts are retrieved.
4. Grow the history to 20+ messages and inspect the prompt size — implement a sliding window limit of 10 messages by passing `maxMessages` to `MessageChatMemoryAdvisor`.

---

## 4. Resources

- **[Spring AI Chat Memory](https://docs.spring.io/spring-ai/reference/api/advisors.html#_chat_memory)** — `MessageChatMemoryAdvisor`, `InMemoryChatMemory`, `JdbcChatMemory`, and how advisors inject history.
- **[MemGPT / Letta Paper](https://arxiv.org/abs/2310.08560)** — The seminal paper on managing LLM memory beyond the context window. Introduces the paging metaphor for memory management.
- **[LangChain Memory Docs](https://python.langchain.com/docs/concepts/memory/)** — Language-agnostic conceptual breakdown of memory types: buffer, summary, entity, vector. Directly applicable to Spring AI patterns.

---

## 5. Trending Context

**Memory is the next frontier for production AI assistants.** In 2025, users expect AI to remember them — not just within a session but across days and months. OpenAI's ChatGPT memory, Claude's Projects feature, and Google's Gemini long-context approach all solve this differently. On the infrastructure side, the industry is converging on a pattern: Redis or pgvector for fast memory retrieval, async background jobs for memory consolidation (so it doesn't block the user response), and LLM-generated summaries to compress long histories. Spring AI's roadmap includes a `VectorStoreChatMemory` implementation that does this consolidation automatically.

---

## 6. Community Tip

**Never store raw conversation history in your vector DB — store extracted facts instead.** Raw history is noisy, verbose, and expensive to embed. Instead, run a "fact extraction" prompt at the end of each session (as shown in Step 5) and store the condensed bullet points. This reduces storage costs, improves retrieval relevance (signal-to-noise ratio is much higher), and avoids leaking sensitive mid-conversation details into long-term storage. Apply the same principle you'd use in a relational DB: normalise the data before storing. Also: always include a `userId` / `customerId` metadata field on every memory document so you can filter by user — without it, you'll retrieve other users' memories, which is both a privacy bug and a hallucination risk.

---

## 7. Tomorrow's Preview

**Day 9** will cover **LLM Observability & Evaluation** — how to trace, monitor, and evaluate LLM-powered systems in production: OpenTelemetry tracing for LLM calls, latency and cost metrics, prompt regression testing, and LLM-as-judge evaluation patterns.

---

*Built progressively. Each day connects to the next.*
