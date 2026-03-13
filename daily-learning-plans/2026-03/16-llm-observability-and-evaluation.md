# Day 9 — LLM Observability & Evaluation
**Date:** 2026-03-16
**Phase:** 1 — Generative AI & Agentic AI
**Time Target:** 1–2 hours

---

## 1. Today's Focus Topic
**LLM Observability & Evaluation — You Can't Improve What You Can't Measure**

You now have an agent that reasons, uses tools, and remembers. But when it gives a wrong answer in production — how do you debug it? How do you prove it's getting better? LLM systems fail in subtle ways: hallucinations, prompt regressions, latency spikes, cost overruns. Today you add the instrumentation layer: tracing every LLM call end-to-end, tracking latency and token cost, and evaluating answer quality using automated scoring. Analogy: this is the same discipline as APM (Application Performance Monitoring) for microservices — you already do this with Spring Boot Actuator and Micrometer; today you apply the same thinking to LLM pipelines.

---

## 2. Key Concepts

### What to Observe in LLM Systems
| Signal | What It Tells You |
|--------|-------------------|
| Latency (TTFT, total) | Where time is spent — model, retrieval, tools |
| Token usage (input/output) | Cost per request; prompt bloat detection |
| Tool call count | Agent loop efficiency; infinite loop detection |
| Retrieval quality (MRR, recall) | RAG pipeline health |
| Answer quality (LLM-as-judge) | Correctness, faithfulness, helpfulness |

**TTFT** = Time To First Token — critical for streaming UX.

### OpenTelemetry for LLM Calls
Spring AI 1.0 ships with built-in OpenTelemetry (OTel) instrumentation via Micrometer Tracing. Every `ChatClient` call automatically creates a span with:
- Model name and provider
- Input/output token counts
- Latency
- Tool calls made

Connect to any OTel-compatible backend: Zipkin, Jaeger, Grafana Tempo, Honeycomb, or Langfuse.

### LLM-as-Judge Evaluation
The most powerful evaluation pattern: use a second LLM call to score the first. A judge prompt receives the question, the expected answer (or rubric), and the actual answer — then outputs a score and reasoning. Used for:
- **Faithfulness** — did the answer stay grounded in the retrieved documents (no hallucination)?
- **Correctness** — is the answer factually right?
- **Helpfulness** — did it actually address the user's question?

Spring AI ships a built-in `RelevancyEvaluator` and `FaithfulnessEvaluator` for RAG pipelines.

### Prompt Regression Testing
Every time you change a prompt, run a test suite. Structure: a set of `(input, expected_output)` pairs evaluated by LLM-as-judge. Integrate into your CI pipeline — treat prompt changes the same as code changes.

---

## 3. Hands-On Task

### Goal: Add tracing, cost tracking, and automated evaluation to the order assistant from Day 6/7

**Step 1 — Dependencies**
```xml
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-opentelemetry</artifactId>
</dependency>
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-tracing-bridge-otel</artifactId>
</dependency>
<dependency>
    <groupId>io.opentelemetry.instrumentation</groupId>
    <artifactId>opentelemetry-spring-boot-starter</artifactId>
</dependency>
```

**Step 2 — application.yml**
```yaml
spring:
  ai:
    observation:
      include-prompt: true        # include prompt content in spans (disable in prod for PII)
      include-completion: true    # include completion content in spans

management:
  tracing:
    sampling:
      probability: 1.0            # 100% sampling in dev
  endpoints:
    web:
      exposure:
        include: health,metrics,prometheus
```

**Step 3 — Custom token cost metric**
```java
@Component
public class LlmCostObserver implements ObservationHandler<ChatModelObservationContext> {

    private static final Logger log = LoggerFactory.getLogger(LlmCostObserver.class);
    private final MeterRegistry meterRegistry;

    public LlmCostObserver(MeterRegistry meterRegistry) {
        this.meterRegistry = meterRegistry;
    }

    @Override
    public void onStop(ChatModelObservationContext context) {
        var usage = context.getResponse().getMetadata().getUsage();
        if (usage == null) return;

        long inputTokens  = usage.getPromptTokens();
        long outputTokens = usage.getGenerationTokens();

        // Claude Sonnet pricing example (adjust per model)
        double costUsd = (inputTokens / 1_000_000.0 * 3.0) + (outputTokens / 1_000_000.0 * 15.0);

        meterRegistry.counter("llm.tokens.input",  "model", context.getRequestOptions().getModel()).increment(inputTokens);
        meterRegistry.counter("llm.tokens.output", "model", context.getRequestOptions().getModel()).increment(outputTokens);
        meterRegistry.counter("llm.cost.usd",      "model", context.getRequestOptions().getModel()).increment(costUsd);

        log.info("LLM call — input: {} tokens, output: {} tokens, cost: ${}", inputTokens, outputTokens, String.format("%.6f", costUsd));
    }

    @Override
    public boolean supportsContext(Observation.Context context) {
        return context instanceof ChatModelObservationContext;
    }
}
```

**Step 4 — Faithfulness evaluator for RAG**
```java
@Service
public class RagEvaluationService {

    private final RelevancyEvaluator relevancyEvaluator;
    private final FaithfulnessEvaluator faithfulnessEvaluator;

    public RagEvaluationService(ChatClient.Builder builder) {
        ChatClient judgeClient = builder.build();
        this.relevancyEvaluator   = new RelevancyEvaluator(judgeClient);
        this.faithfulnessEvaluator = new FaithfulnessEvaluator(judgeClient);
    }

    public void evaluate(String question, String answer, List<Document> retrievedDocs) {
        EvaluationRequest request = new EvaluationRequest(question, retrievedDocs, answer);

        EvaluationResponse relevancy   = relevancyEvaluator.evaluate(request);
        EvaluationResponse faithfulness = faithfulnessEvaluator.evaluate(request);

        log.info("Relevancy: {} | Faithfulness: {} | Q: {}",
            relevancy.isPass(), faithfulness.isPass(), question);
    }
}
```

**Step 5 — Prompt regression test**
```java
@SpringBootTest
class OrderAssistantEvalTest {

    @Autowired OrderAssistantService assistantService;
    @Autowired ChatClient.Builder builder;

    record TestCase(String input, String expectedContains) {}

    @ParameterizedTest
    @MethodSource("testCases")
    void assistantAnswersCorrectly(TestCase tc) {
        String answer = assistantService.assist(tc.input());

        // LLM-as-judge: score answer against expected
        ChatClient judge = builder.build();
        String verdict = judge.prompt()
            .system("You are an evaluator. Reply with PASS or FAIL only. " +
                    "PASS if the answer adequately addresses the question and contains the expected information. " +
                    "FAIL otherwise.")
            .user("Question: " + tc.input() + "\nExpected to contain: " + tc.expectedContains() + "\nActual answer: " + answer)
            .call()
            .content();

        assertThat(verdict.trim().toUpperCase()).startsWith("PASS");
    }

    static Stream<TestCase> testCases() {
        return Stream.of(
            new TestCase("Where is order ORD-001?", "SHIPPED"),
            new TestCase("What is the discount on $100 at 10%?", "$90"),
            new TestCase("Where is my order?", "order ID")   // should ask for ID
        );
    }
}
```

**Step 6 — Experiments**
1. Run the app with a local Zipkin (`docker run -p 9411:9411 openzipkin/zipkin`) — observe traces with tool call spans nested under the chat call span.
2. Hit the `/actuator/metrics/llm.tokens.input` endpoint — verify token counts are accumulating.
3. Run the regression test suite — verify all cases pass; then deliberately break a prompt and watch a test fail.

---

## 4. Resources

- **[Spring AI Observability](https://docs.spring.io/spring-ai/reference/observabilty/index.html)** — Official docs for Spring AI OTel integration, span attributes, and metric names.
- **[Langfuse](https://langfuse.com/)** — Open-source LLM observability platform; Spring AI has a Langfuse exporter. Best-in-class for prompt versioning + evaluation dashboards.
- **[RAGAS](https://docs.ragas.io/)** — The standard framework for RAG evaluation metrics: faithfulness, answer relevancy, context precision. Useful for understanding what to measure even if you implement in Java.

---

## 5. Trending Context

**Observability is now a table-stakes requirement for production LLM features.** In 2025, the LLMOps space has consolidated around OpenTelemetry as the standard transport and Langfuse/Honeycomb/Arize as the major backends. The key insight that separates mature LLM teams from early-stage ones: **evaluation must be continuous, not one-off**. Teams with reliable LLM features run LLM-as-judge evaluations on a sample of every production request (async, post-response), track scores over time, and treat a score drop as an incident. This is the LLM equivalent of error rate alerting in your existing microservices.

---

## 6. Community Tip

**Log the full prompt+completion pair (with a sampling rate) to a separate audit store — not just metrics.** Metrics tell you something is wrong; logs tell you why. In production, log ~5–10% of requests (randomly sampled) to a database with the full prompt, completion, model, latency, and user ID. This dataset becomes invaluable for: debugging edge cases, building evaluation test suites from real traffic, and fine-tuning. Store PII-scrubbed versions; never log full prompts if they contain sensitive user data without consent. Think of this as your LLM black box recorder.

---

## 7. Tomorrow's Preview

**Day 10** will cover **Structured Outputs & Output Parsing** — how to reliably extract structured data (JSON, typed objects) from LLM responses, using JSON mode, function-calling-as-schema-enforcement, and Spring AI's `BeanOutputConverter`.

---

*Built progressively. Each day connects to the next.*
