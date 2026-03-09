# Day 3 — LLM APIs & SDKs
**Date:** 2026-03-10
**Phase:** 1 — Generative AI & Agentic AI
**Time Target:** 1–2 hours

---

## 1. Today's Focus Topic
**LLM APIs & SDKs — Spring AI, Streaming, Retries, Production-Ready Client**

Days 1 and 2 gave you the mental model and prompt craft. Today you build the **integration layer** — the code that sits between your Spring Boot app and any LLM provider. Done right, this layer is swappable, observable, and resilient. Done wrong, it's a fragile tangle of hardcoded HTTP calls. Think of it exactly like building a robust REST client for a third-party payment API.

---

## 2. Key Concepts

### Spring AI
- Spring's official abstraction over LLM providers (OpenAI, Anthropic, Ollama, Azure OpenAI, etc.).
- Key interfaces: `ChatClient`, `ChatModel`, `Prompt`, `Message`, `StreamingChatModel`.
- Java analogy: like Spring Data — you code to an interface, swap the provider via config.
- Add to `pom.xml`:
```xml
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-anthropic-spring-boot-starter</artifactId>
</dependency>
```
- Configure in `application.yml`:
```yaml
spring:
  ai:
    anthropic:
      api-key: ${ANTHROPIC_API_KEY}
      chat:
        options:
          model: claude-sonnet-4-6
          max-tokens: 1024
          temperature: 0.7
```

### Streaming Responses
- LLMs generate tokens one at a time — streaming lets you **push tokens to the client as they arrive** instead of waiting for the full response.
- Critical for UX in chat apps; also reduces perceived latency in backend pipelines.
- Spring AI returns `Flux<ChatResponse>` (Project Reactor) for streaming.
- Java analogy: like reading a `BufferedReader` line by line instead of loading the full file into memory.

### Retries & Timeouts
- LLM APIs are external services — they will fail, be slow, or rate-limit you.
- Same discipline as any HTTP dependency:
  - **Timeout:** set both connection and read timeouts.
  - **Retry:** exponential backoff on 429 (rate limit) and 5xx errors. Never retry 4xx client errors.
  - **Circuit breaker:** use Resilience4j to stop hammering a degraded provider.

### Provider Abstraction
- Abstract behind an interface so you can swap providers (Anthropic → OpenAI → local Ollama) without changing business logic.
- Store provider config externally (env vars / Vault) — never hardcode keys.

---

## 3. Hands-On Task

### Goal: Build a production-ready `LlmClient` wrapper using Spring AI

**Step 1 — Interface definition**
```java
public interface LlmClient {
    String chat(String systemPrompt, String userMessage);
    Flux<String> stream(String systemPrompt, String userMessage);
}
```

**Step 2 — Spring AI implementation**
```java
@Service
public class SpringAiLlmClient implements LlmClient {

    private final ChatClient chatClient;

    public SpringAiLlmClient(ChatClient.Builder builder) {
        this.chatClient = builder.build();
    }

    @Override
    public String chat(String systemPrompt, String userMessage) {
        return chatClient.prompt()
            .system(systemPrompt)
            .user(userMessage)
            .call()
            .content();
    }

    @Override
    public Flux<String> stream(String systemPrompt, String userMessage) {
        return chatClient.prompt()
            .system(systemPrompt)
            .user(userMessage)
            .stream()
            .content();
    }
}
```

**Step 3 — Add retry with Resilience4j**
```java
@Service
public class ResilientLlmClient implements LlmClient {

    private final LlmClient delegate;
    private final RetryRegistry retryRegistry;

    public ResilientLlmClient(SpringAiLlmClient delegate, RetryRegistry retryRegistry) {
        this.delegate = delegate;
        this.retryRegistry = retryRegistry;
    }

    @Override
    public String chat(String systemPrompt, String userMessage) {
        Retry retry = retryRegistry.retry("llm-client");
        return Retry.decorateSupplier(retry, () -> delegate.chat(systemPrompt, userMessage)).get();
    }

    @Override
    public Flux<String> stream(String systemPrompt, String userMessage) {
        // For streaming, retry at the subscription level
        return delegate.stream(systemPrompt, userMessage)
            .retryWhen(Retry.backoff(3, Duration.ofSeconds(1))
                .filter(ex -> ex instanceof HttpServerErrorException));
    }
}
```

**Resilience4j config in `application.yml`:**
```yaml
resilience4j:
  retry:
    instances:
      llm-client:
        max-attempts: 3
        wait-duration: 1s
        exponential-backoff-multiplier: 2
        retry-exceptions:
          - org.springframework.web.client.HttpServerErrorException
```

**Step 4 — Wire into your SentimentService from Day 2**
```java
// Replace ClaudeService with LlmClient interface
@Service
public class SentimentService {

    private final LlmClient llmClient;
    private final PromptTemplate template;

    public SentimentService(LlmClient llmClient) throws IOException {
        this.llmClient = llmClient;
        this.template = PromptTemplate.fromResource("prompts/classify-sentiment.txt");
    }

    public SentimentResult classify(String text) {
        String userPrompt = template.render(Map.of("user_input", text));
        String raw = llmClient.chat("You are a JSON-only sentiment classifier.", userPrompt);
        return new ObjectMapper().readValue(raw, SentimentResult.class);
    }
}
```

**Step 5 — Experiments**
1. Hit the streaming endpoint from a REST controller — observe tokens arriving in the browser via SSE.
2. Simulate a timeout by setting `read-timeout: 1ms` — verify retry kicks in.
3. Switch the provider from Anthropic to Ollama (local) by changing only `application.yml` — verify `LlmClient` interface holds.
4. Log `(model, input_tokens, output_tokens, latency_ms)` on every call — build observability from day one.

---

## 4. Resources

- **[Spring AI Reference Docs](https://docs.spring.io/spring-ai/reference/)** — Official docs, covers all providers. Start with the "Chat Model" and "Streaming" sections.
- **[Anthropic Java SDK (official)](https://github.com/anthropics/anthropic-sdk-java)** — Official Java SDK released in 2024; Spring AI wraps this under the hood for Anthropic.
- **[Resilience4j Docs — Retry](https://resilience4j.readme.io/docs/retry)** — Drop-in retry/circuit-breaker for Spring Boot. If you've used Hystrix before, this is its modern replacement.

---

## 5. Trending Context

**Spring AI hit 1.0 GA in 2025** and is now the de-facto standard for LLM integration in the Java ecosystem. It supports: chat, embedding, image generation, audio, vector stores, RAG pipelines, and function calling — all behind unified Spring abstractions. For backend engineers already on Spring Boot, this is the lowest-friction path to production AI. Avoid raw HTTP clients for LLM calls in new projects — Spring AI's abstractions will save you from reinventing retry logic, streaming handling, and provider-specific quirks.

---

## 6. Community Tip

**Instrument your LLM calls from day one.** Add a Spring `WebClient` filter (or Spring AI advisor) that logs token usage, latency, and model name on every call. LLM costs and latency are opaque until you measure them — and in production, a single prompt change can 3x your costs. Tools like **Helicone** and **LangSmith** provide dashboards for this, but even a simple `llm_call_log` DB table beats flying blind. Think of it as adding Micrometer metrics to your Kafka consumers — standard operational hygiene.

---

## 7. Tomorrow's Preview

**Day 4** will cover **Embeddings & Vector Search** — what embeddings are, how to generate them via API, and how to store/query them using a vector database (pgvector or Qdrant). This is the foundation of RAG (Retrieval-Augmented Generation) — the most important pattern for grounding LLMs in your own data.

---

*Built progressively. Each day connects to the next.*
