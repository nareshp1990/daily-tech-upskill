# Day 10 — Structured Outputs & Output Parsing
**Date:** 2026-03-17
**Phase:** 1 — Generative AI & Agentic AI
**Time Target:** 1–2 hours

---

## 1. Today's Focus Topic
**Structured Outputs & Output Parsing — Making LLMs Return Machine-Readable Data**

LLMs natively produce free-form text. But most backend systems need structured data: a parsed order, a classified ticket, a typed API response. The naive approach — ask nicely for JSON and hope — breaks in production. Models hallucinate extra fields, use wrong types, or wrap JSON in markdown. Today you learn the reliable patterns: JSON mode, schema-constrained generation, and Spring AI's converter pipeline that maps LLM output directly to Java objects. This is one of the highest-ROI skills in applied AI — it's the bridge between the LLM and the rest of your Java microservices.

---

## 2. Key Concepts

### Why Free-Form Parsing Fails
- LLMs may wrap JSON in ```json ... ``` markdown fences.
- Field names or types may drift from the schema.
- Optional fields may be omitted or set to `null` when you expect a value.
- Nested objects are error-prone without explicit schema.

### Three Reliable Approaches

**1. JSON Mode** — instruct the model to output only valid JSON (supported by GPT-4o, Claude via system prompt, Gemini). Guarantees valid JSON syntax but not schema compliance.

**2. Function Calling as Schema Enforcement** — define a tool whose parameters match your output schema. The LLM fills in tool arguments (which are schema-validated). Spring AI's `BeanOutputConverter` uses this internally.

**3. Structured Outputs (OpenAI)** — strict JSON Schema enforcement at the model level; guarantees 100% schema compliance. Available on GPT-4o-mini and later. Claude achieves the same via `BeanOutputConverter`.

### Spring AI Output Converters
Spring AI ships three converters out of the box:

| Converter | Output Type | Use Case |
|-----------|------------|---------|
| `BeanOutputConverter<T>` | Single Java record/class | Typed extraction |
| `ListOutputConverter` | `List<String>` | Enumeration, tagging |
| `MapOutputConverter` | `Map<String, Object>` | Dynamic/unknown schema |

The converter appends format instructions to your prompt automatically and deserialises the response via Jackson.

---

## 3. Hands-On Task

### Goal: Build a ticket triage system that extracts structured data from free-form support messages

**Step 1 — Define output schema as Java records**
```java
public record TicketClassification(
    @JsonProperty("category")    String category,        // BILLING, SHIPPING, TECHNICAL, OTHER
    @JsonProperty("priority")    String priority,        // HIGH, MEDIUM, LOW
    @JsonProperty("sentiment")   String sentiment,       // POSITIVE, NEUTRAL, NEGATIVE
    @JsonProperty("summary")     String summary,         // 1-sentence summary
    @JsonProperty("tags")        List<String> tags,      // up to 3 relevant tags
    @JsonProperty("requiresEscalation") boolean requiresEscalation
) {}
```

**Step 2 — Triage service with BeanOutputConverter**
```java
@Service
public class TicketTriageService {

    private final ChatClient chatClient;
    private final BeanOutputConverter<TicketClassification> converter;

    public TicketTriageService(ChatClient.Builder builder) {
        this.converter = new BeanOutputConverter<>(TicketClassification.class);
        this.chatClient = builder.build();
    }

    public TicketClassification classify(String ticketText) {
        String format = converter.getFormat(); // appends JSON schema instructions to prompt

        String raw = chatClient.prompt()
            .system("""
                You are a customer support triage system. Analyse the incoming support message
                and classify it according to the required JSON schema.
                Categories: BILLING, SHIPPING, TECHNICAL, OTHER.
                Priorities: HIGH (urgent/angry/data loss), MEDIUM (inconvenient), LOW (minor/question).
                """ + format)
            .user(ticketText)
            .call()
            .content();

        return converter.convert(raw);
    }
}
```

**Step 3 — List extraction (tags and keywords)**
```java
@Service
public class KeywordExtractorService {

    private final ChatClient chatClient;
    private final ListOutputConverter converter = new ListOutputConverter(new DefaultConversionService());

    public KeywordExtractorService(ChatClient.Builder builder) {
        this.chatClient = builder.build();
    }

    public List<String> extractKeywords(String text) {
        String raw = chatClient.prompt()
            .system("Extract the top 5 keywords from the text. " + converter.getFormat())
            .user(text)
            .call()
            .content();

        return converter.convert(raw);
    }
}
```

**Step 4 — Validation layer (never trust raw LLM output)**
```java
@Component
public class ClassificationValidator {

    private static final Set<String> VALID_CATEGORIES = Set.of("BILLING", "SHIPPING", "TECHNICAL", "OTHER");
    private static final Set<String> VALID_PRIORITIES  = Set.of("HIGH", "MEDIUM", "LOW");
    private static final Set<String> VALID_SENTIMENTS  = Set.of("POSITIVE", "NEUTRAL", "NEGATIVE");

    public void validate(TicketClassification tc) {
        if (!VALID_CATEGORIES.contains(tc.category()))
            throw new IllegalStateException("Invalid category from LLM: " + tc.category());
        if (!VALID_PRIORITIES.contains(tc.priority()))
            throw new IllegalStateException("Invalid priority from LLM: " + tc.priority());
        if (!VALID_SENTIMENTS.contains(tc.sentiment()))
            throw new IllegalStateException("Invalid sentiment from LLM: " + tc.sentiment());
        if (tc.summary() == null || tc.summary().isBlank())
            throw new IllegalStateException("Missing summary from LLM");
    }
}
```

**Step 5 — REST endpoint**
```java
@RestController
@RequestMapping("/api/tickets")
public class TicketController {

    private final TicketTriageService triageService;
    private final ClassificationValidator validator;

    public TicketController(TicketTriageService triageService, ClassificationValidator validator) {
        this.triageService = triageService;
        this.validator = validator;
    }

    @PostMapping("/classify")
    public TicketClassification classify(@RequestBody Map<String, String> body) {
        TicketClassification result = triageService.classify(body.get("message"));
        validator.validate(result);
        return result;
    }
}
```

**Step 6 — Retry on parse failure**
```java
// Wrap the classify call with Resilience4j Retry (from Day 3) to handle
// rare JSON parse failures:
@Retry(name = "llm-parse", fallbackMethod = "classifyFallback")
public TicketClassification classifyWithRetry(String ticketText) {
    return classify(ticketText);
}

public TicketClassification classifyFallback(String ticketText, Exception ex) {
    log.error("LLM parse failed after retries: {}", ex.getMessage());
    return new TicketClassification("OTHER", "LOW", "NEUTRAL",
        "Unable to classify — manual review required", List.of(), false);
}
```

**Step 7 — Experiments**
1. POST `{"message": "My order ORD-999 hasn't arrived in 3 weeks and I'm furious!"}` — expect `SHIPPING`, `HIGH`, `NEGATIVE`, `requiresEscalation: true`.
2. POST `{"message": "Just wanted to say your app is great!"}` — expect `OTHER`, `LOW`, `POSITIVE`.
3. Intentionally break the schema (rename a field in the record) — observe the parse failure; verify the retry + fallback fires.
4. Add a `@NotNull` Bean Validation annotation to summary — run `validator.validate()` and see it catch a missing field.

---

## 4. Resources

- **[Spring AI Output Parsers](https://docs.spring.io/spring-ai/reference/api/output-parser.html)** — Official docs for `BeanOutputConverter`, `ListOutputConverter`, `MapOutputConverter`.
- **[OpenAI Structured Outputs](https://platform.openai.com/docs/guides/structured-outputs)** — The strictest form of schema enforcement; useful for understanding the state of the art even if using Claude.
- **[Instructor (Python)](https://python.useinstructor.com/)** — The most popular structured output library; reading their pattern docs gives excellent mental models applicable to Spring AI.

---

## 5. Trending Context

**Structured outputs have become the standard integration pattern between LLMs and backend systems in 2025.** The era of parsing free-form LLM text with regex is over. OpenAI's Structured Outputs (strict JSON Schema) and Anthropic's tool-use-as-schema pattern both guarantee valid output. The next wave: **multi-step structured pipelines** where each LLM call produces a typed intermediate result that feeds the next step — making LLM pipelines as predictable as typed function compositions. Spring AI's converters align exactly with this direction.

---

## 6. Community Tip

**Put your output schema in a separate, versioned file — not hardcoded in the Java record.** When you change the schema (add a field, rename a key), you're changing the implicit contract your prompt makes with the LLM. Treat schema changes like API versioning: bump a version field, run your regression test suite (Day 9) against the new schema before deploying, and keep the old schema active for in-flight requests. Also: always include an `"other"` escape hatch in enum fields (like `category: OTHER`) — the LLM will encounter inputs your schema designers didn't anticipate, and a safe default beats an exception.

---

## 7. Tomorrow's Preview

**Day 11** will cover **Multi-Agent Systems & Orchestration** — how to decompose complex tasks across multiple specialised agents, implement an orchestrator-worker pattern, and handle inter-agent communication in Spring AI.

---

*Built progressively. Each day connects to the next.*
