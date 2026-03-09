# Day 2 — Prompt Engineering
**Date:** 2026-03-09
**Phase:** 1 — Generative AI & Agentic AI
**Time Target:** 1–2 hours

---

## 1. Today's Focus Topic
**Prompt Engineering — System Prompts, Few-Shot, JSON Mode, Chain-of-Thought**

Yesterday you learned what happens inside an LLM. Today you learn how to **talk to it effectively**. Prompt engineering is not magic — it's the discipline of structuring inputs to get reliable, parseable, production-grade outputs. For a backend engineer, think of it as designing an API contract with the model.

---

## 2. Key Concepts

### System Prompt
- The "constructor config" of the model's behavior for the entire conversation.
- Sets role, tone, constraints, output format, and domain context.
- Processed first — has higher influence than user messages.
- Java analogy: like a `@Configuration` bean that initializes behavior before any request arrives.

```
System: You are a JSON API. You only respond with valid JSON. Never include prose.
```

### Few-Shot Prompting
- Provide **examples** of input → output pairs before the actual task.
- Dramatically improves consistency and format adherence.
- Zero-shot: no examples. One-shot: 1 example. Few-shot: 2–5 examples (sweet spot).
- Java analogy: writing unit test fixtures that define expected behavior — the model infers the pattern.

```
User:
Classify the following support ticket sentiment.

Examples:
Input: "I can't believe this is broken again" → Output: {"sentiment": "negative"}
Input: "Works perfectly, thank you!" → Output: {"sentiment": "positive"}

Now classify:
Input: "It's okay I guess, does what I need"
```

### Structured Output (JSON Mode)
- Force the model to return machine-parseable JSON — essential for backend integrations.
- Two approaches:
  1. **Prompt-level**: Instruct in system prompt + use few-shot examples.
  2. **API-level**: Some APIs (OpenAI, Anthropic) support native JSON mode or tool-use schemas.
- Always validate the response with a JSON parser — models can still hallucinate structure under pressure.

### Chain-of-Thought (CoT)
- Ask the model to **reason step-by-step** before giving the final answer.
- Improves accuracy on multi-step tasks: classification, extraction, code review, math.
- Two styles:
  - **Explicit CoT**: "Think step by step before answering."
  - **Structured CoT**: Ask for `reasoning` field in JSON, then `answer` field.
- Modern reasoning models (Claude Opus 4.6, o3) do this internally — you may not need to prompt for it.

### Prompt Template Pattern
- Separate **template** (static structure) from **variables** (dynamic input).
- Enables reuse, testing, versioning — just like `JdbcTemplate` or `MessageFormat`.
- Critical for production: templates go in config/files, not hardcoded strings.

---

## 3. Hands-On Task

### Goal: Build a reusable `PromptTemplate` abstraction in Java

**Step 1 — Define the abstraction**
```java
public class PromptTemplate {

    private final String template;

    public PromptTemplate(String template) {
        this.template = template;
    }

    public String render(Map<String, String> variables) {
        String result = template;
        for (Map.Entry<String, String> entry : variables.entrySet()) {
            result = result.replace("{{" + entry.getKey() + "}}", entry.getValue());
        }
        return result;
    }

    // Load from classpath resource (e.g., prompts/classify.txt)
    public static PromptTemplate fromResource(String resourcePath) throws IOException {
        try (InputStream is = PromptTemplate.class.getClassLoader().getResourceAsStream(resourcePath)) {
            String content = new String(is.readAllBytes(), StandardCharsets.UTF_8);
            return new PromptTemplate(content);
        }
    }
}
```

**Step 2 — Create a prompt file** (`src/main/resources/prompts/classify-sentiment.txt`)
```
You are a sentiment classifier. Respond only with valid JSON in this exact format:
{"sentiment": "<positive|negative|neutral>", "confidence": "<high|medium|low>", "reasoning": "<one sentence>"}

Examples:
Input: "I can't believe this is broken again" → {"sentiment": "negative", "confidence": "high", "reasoning": "Strong frustration language."}
Input: "Works perfectly, thank you!" → {"sentiment": "positive", "confidence": "high", "reasoning": "Explicit satisfaction expressed."}

Now classify:
Input: {{user_input}}
```

**Step 3 — Wire it into your Claude service from Day 1**
```java
@Service
public class SentimentService {

    private final ClaudeService claudeService;
    private final PromptTemplate template;

    public SentimentService(ClaudeService claudeService) throws IOException {
        this.claudeService = claudeService;
        this.template = PromptTemplate.fromResource("prompts/classify-sentiment.txt");
    }

    public SentimentResult classify(String text) {
        String prompt = template.render(Map.of("user_input", text));
        String raw = claudeService.chat(prompt);

        // Parse JSON response
        ObjectMapper mapper = new ObjectMapper();
        return mapper.readValue(raw, SentimentResult.class);
    }
}

public record SentimentResult(String sentiment, String confidence, String reasoning) {}
```

**Step 4 — Experiments**
1. Try zero-shot vs few-shot on ambiguous inputs ("It's fine") — observe confidence difference.
2. Add a `reasoning` field to the JSON schema and compare accuracy on hard cases.
3. Break the system prompt intentionally (remove JSON instruction) — observe how it degrades.
4. Test with a Java stack trace as input — see if it correctly classifies as "negative/frustrated".

---

## 4. Resources

- **[Anthropic Prompt Engineering Guide](https://platform.claude.com/docs/en/docs/build-with-claude/prompt-engineering/overview)** — Authoritative, updated for Claude 4.x. Read the "Be clear and direct" and "Use examples" sections.
- **[OpenAI Prompt Engineering Guide](https://platform.openai.com/docs/guides/prompt-engineering)** — Complementary perspective, many techniques are model-agnostic.
- **[Learn Prompting](https://learnprompting.org/)** — Free open-source course, good for deepening specific techniques.

---

## 5. Trending Context

**Structured outputs are now a first-class API feature (2025–2026):** OpenAI's `response_format: {type: "json_schema"}` and Anthropic's tool-use API let you pass a JSON Schema and the model is **constrained** to output only valid JSON matching that schema. This is a game-changer for backend integration — no more fragile regex parsing. For Java devs: think of it as compile-time type safety for LLM responses. This is now the production-grade approach over prompt-level JSON instructions alone.

---

## 6. Community Tip

**Prompt versioning matters in production.** Treat prompts like code: version them in Git, A/B test changes, and log prompt + response pairs for debugging. Tools like **LangSmith**, **Braintrust**, and **Helicone** offer prompt tracing dashboards — similar to distributed tracing (Zipkin/Jaeger) but for LLM calls. Start simple: just log `(prompt_version, input, output, latency_ms)` to a DB table.

---

## 7. Tomorrow's Preview

**Day 3** will cover **LLM APIs & SDKs** — exploring the Anthropic Java SDK (or Spring AI), handling streaming responses, managing retries/timeouts, and building a production-ready `LlmClient` wrapper with proper error handling. Think of it as building a robust HTTP client layer for AI — the same discipline you'd apply to any external service integration.

---

*Built progressively. Each day connects to the next.*
