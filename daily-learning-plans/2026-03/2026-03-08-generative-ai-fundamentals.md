# Day 1 — Generative AI Fundamentals
**Date:** 2026-03-08
**Phase:** 1 — Generative AI & Agentic AI
**Time Target:** 1–2 hours

---

## 1. Today's Focus Topic
**How Large Language Models (LLMs) Work — From Tokens to Inference**

A solid mental model of LLMs is the foundation for everything else in AI integration, prompt engineering, RAG, and agentic systems. Before calling an API or building a pipeline, you need to understand what's happening under the hood.

---

## 2. Key Concepts

### Tokenization
- Text is split into **tokens** (subwords, not full words). "Spring Boot" ≈ 3 tokens.
- Token limits (context window) are your hard constraint — critical for backend API design.
- Java analogy: tokens are like bytes in a `ByteBuffer` — the model never sees raw strings.

### Transformer Architecture (High Level)
- **Attention mechanism** — every token attends to every other token; captures context.
- **Layers** — stacked transformer blocks refine representations.
- **Parameters** — weights learned during training; frozen at inference.
- You don't need the math. Know: more parameters ≠ always better; training data quality matters more.

### Inference Pipeline
1. Prompt → tokenize → embed → forward pass through layers → logits → sampling → token
2. **Temperature** controls randomness (0 = deterministic, 1+ = creative)
3. **Top-p / Top-k** — sampling strategies that affect output diversity
4. Tokens are generated **one at a time** (autoregressive) — latency scales with output length.

### Context Window
- The "working memory" of an LLM — everything it can see at once.
- Modern models: 128k–1M tokens. Older: 4k–8k.
- Cost and latency scale with tokens in + tokens out.

### Prompt Structure
- **System prompt** — sets role/behavior (like a constructor or config bean)
- **User message** — the actual request
- **Assistant turn** — model's response (can be pre-seeded for control)

---

## 3. Hands-On Task

### Goal: Call an LLM API from Java and experiment with parameters

**Option A — Anthropic Claude API (recommended)**
```java
// Add to pom.xml:
// No official Java SDK yet — use OkHttp or Spring's RestClient

// Using Spring Boot RestClient
@Service
public class ClaudeService {

    private final RestClient restClient;

    public ClaudeService() {
        this.restClient = RestClient.builder()
            .baseUrl("https://api.anthropic.com")
            .defaultHeader("x-api-key", System.getenv("ANTHROPIC_API_KEY"))
            .defaultHeader("anthropic-version", "2023-06-01")
            .defaultHeader("Content-Type", "application/json")
            .build();
    }

    public String chat(String userMessage) {
        var body = Map.of(
            "model", "claude-opus-4-6",
            "max_tokens", 1024,
            "messages", List.of(
                Map.of("role", "user", "content", userMessage)
            )
        );

        var response = restClient.post()
            .uri("/v1/messages")
            .body(body)
            .retrieve()
            .body(Map.class);

        var content = (List<Map<String, Object>>) response.get("content");
        return (String) content.get(0).get("text");
    }
}
```

**Option B — OpenAI-compatible API (if you have an OpenAI key)**
Same pattern, change base URL and model name.

**Experiments to run:**
1. Send the same prompt with `temperature=0` vs `temperature=1.0` — observe variance.
2. Count tokens manually vs what the API reports (`usage.input_tokens`).
3. Try a very long system prompt — observe latency increase.
4. Ask it to explain a Java Spring Boot concept — verify domain knowledge quality.

---

## 4. Resources

- **[Anthropic: Introduction to Claude](https://platform.claude.com/docs/en/docs/intro-to-claude)** — Official docs, clean and up-to-date. Start here.
- **[3Blue1Brown: But what is a GPT?](https://www.youtube.com/watch?v=wjZofJX0v4M)** — Best visual explanation of transformers (40 min, worth every minute).
- **[Tiktokenizer (browser tool)](https://tiktokenizer.vercel.app/)** — Paste any text and see how it tokenizes. Builds intuition fast.

---

## 5. Trending Context

**Reasoning Models are now mainstream (2025–2026):** Models like Claude Opus 4.6, GPT-o1/o3, and Gemini 2.0 Flash Thinking do extended internal reasoning before answering ("chain-of-thought" is now baked in). This changes prompt engineering — you no longer need to say "think step by step" for capable models. For backend engineers, this means: longer latency, higher token cost, but dramatically better accuracy on complex tasks like code generation and data extraction.

---

## 6. Community Tip

**Follow:** [Latent Space Podcast](https://www.latent.space/) — a podcast + Discord community run by ML engineers and builders. Heavy on practical AI engineering (not hype). Episodes like "The Anatomy of Autonomy" are gold for backend devs building AI systems.

Also subscribe to: **[TLDR AI newsletter](https://tldr.tech/ai)** — daily 5-minute digest, no fluff.

---

## 7. Tomorrow's Preview

**Day 2** will cover **Prompt Engineering** — system prompts, few-shot examples, structured output (JSON mode), and chain-of-thought. You'll build a reusable `PromptTemplate` abstraction in Java, similar to Spring's `JdbcTemplate`.

---

*Built progressively. Each day connects to the next.*
