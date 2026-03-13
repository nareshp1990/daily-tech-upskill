# Day 14 — Fine-Tuning & Model Customisation
**Date:** 2026-03-21
**Phase:** 1 — Generative AI & Agentic AI
**Time Target:** 1–2 hours

---

## 1. Today's Focus Topic
**Fine-Tuning & Model Customisation — Teaching the Model Your Domain**

Prompt engineering and RAG take you far — but there are tasks where you need the model itself to behave differently: adopt a specific tone, reliably output a proprietary format, or understand domain jargon that wasn't in its training data. Fine-tuning adjusts the model's weights on your data, baking in behaviour that prompts alone can't reliably achieve. Today you learn when fine-tuning is (and isn't) the right answer, how to prepare a quality dataset, how to kick off a fine-tuning job via API, and how to evaluate whether the fine-tuned model actually improves on the base model. Analogy: base model = a brilliant generalist new-hire; fine-tuning = the onboarding process that teaches them your company's conventions, terminology, and workflow.

---

## 2. Key Concepts

### When to Fine-Tune (and When NOT To)
**Fine-tune when:**
- You need consistent format/style that few-shot prompting can't reliably produce (e.g., your proprietary JSON schema).
- You have 100+ high-quality labelled examples of the target behaviour.
- You need lower latency via a smaller, specialised model vs. a large general-purpose one.
- The task requires deep domain knowledge (legal, medical, internal jargon) not well-represented in base training.

**Do NOT fine-tune when:**
- You haven't tried prompt engineering and RAG first — they're cheaper and faster to iterate.
- You have fewer than 50 examples — not enough signal; the model will overfit.
- You want to add new knowledge (use RAG instead — fine-tuning doesn't reliably inject facts).
- Your data changes frequently — fine-tuning is a one-time snapshot; RAG retrieves live data.

### Fine-Tuning vs. RAG vs. Prompting
| Approach | Best for | Data requirement | Cost | Iteration speed |
|----------|---------|-----------------|------|----------------|
| Prompt engineering | Behaviour guidance | None | Zero | Minutes |
| RAG | Dynamic knowledge injection | Documents | Low (storage) | Hours |
| Fine-tuning | Reliable format/style/tone | 50–10K labelled examples | Medium–High | Days |

### Dataset Preparation (the most important step)
The quality of your fine-tune is entirely determined by dataset quality. Rules:
1. **Format:** JSONL — one training example per line, `{"messages": [{"role": "system", ...}, {"role": "user", ...}, {"role": "assistant", ...}]}`.
2. **Quality over quantity:** 200 expert-curated examples beat 2000 mediocre ones.
3. **Diversity:** cover edge cases, not just the happy path.
4. **Consistency:** the assistant response must be exactly what you want the model to learn — clean up any inconsistencies.
5. **No PII:** scrub all sensitive data before submission.

### Fine-Tuning APIs
- **OpenAI:** `POST /v1/fine_tuning/jobs` — mature, supports GPT-4o-mini and GPT-3.5-turbo.
- **Anthropic:** Fine-tuning for Claude models via the API (available on enterprise plans).
- **Open-source (self-hosted):** LoRA/QLoRA on Llama/Mistral via Hugging Face `transformers` + PEFT — keeps data on-prem.

---

## 3. Hands-On Task

### Goal: Prepare a fine-tuning dataset for the order assistant and submit a fine-tuning job

**Step 1 — Define the target behaviour**
Our goal: the model should always respond in a structured format:
```
STATUS: [order status]
DELIVERY: [date or "unknown"]
ACTION: [what the customer should do next, or "none required"]
```
This format is hard to enforce reliably with prompting alone across thousands of requests.

**Step 2 — Generate training examples programmatically**
```java
@Component
public class FineTuneDatasetGenerator {

    record TrainingExample(List<Map<String, String>> messages) {}

    public List<TrainingExample> generate() {
        return List.of(
            example("You are an order assistant. Always respond in the exact format: STATUS/DELIVERY/ACTION.",
                    "Where is my order ORD-001?",
                    "STATUS: SHIPPED\nDELIVERY: 2026-03-15\nACTION: none required — your order is on its way."),

            example("You are an order assistant. Always respond in the exact format: STATUS/DELIVERY/ACTION.",
                    "Check ORD-002 for me",
                    "STATUS: PROCESSING\nDELIVERY: 2026-03-18\nACTION: none required — order is being prepared."),

            example("You are an order assistant. Always respond in the exact format: STATUS/DELIVERY/ACTION.",
                    "My order ORD-999 hasn't arrived",
                    "STATUS: NOT_FOUND\nDELIVERY: unknown\nACTION: please verify your order ID and contact support if the issue persists."),

            example("You are an order assistant. Always respond in the exact format: STATUS/DELIVERY/ACTION.",
                    "Is ORD-001 delayed?",
                    "STATUS: SHIPPED\nDELIVERY: 2026-03-15\nACTION: none required — delivery is on schedule.")

            // ... add 100+ diverse examples covering edge cases
        );
    }

    private TrainingExample example(String system, String user, String assistant) {
        return new TrainingExample(List.of(
            Map.of("role", "system",    "content", system),
            Map.of("role", "user",      "content", user),
            Map.of("role", "assistant", "content", assistant)
        ));
    }

    public void writeToJsonl(List<TrainingExample> examples, Path outputPath) throws IOException {
        ObjectMapper mapper = new ObjectMapper();
        try (BufferedWriter writer = Files.newBufferedWriter(outputPath)) {
            for (TrainingExample ex : examples) {
                writer.write(mapper.writeValueAsString(Map.of("messages", ex.messages())));
                writer.newLine();
            }
        }
        log.info("Wrote {} training examples to {}", examples.size(), outputPath);
    }
}
```

**Step 3 — Validate dataset quality**
```java
@Component
public class DatasetValidator {

    public void validate(Path datasetPath) throws IOException {
        ObjectMapper mapper = new ObjectMapper();
        List<String> errors = new ArrayList<>();
        int lineNumber = 0;

        try (BufferedReader reader = Files.newBufferedReader(datasetPath)) {
            String line;
            while ((line = reader.readLine()) != null) {
                lineNumber++;
                try {
                    JsonNode root = mapper.readTree(line);
                    JsonNode messages = root.get("messages");

                    if (messages == null || !messages.isArray())
                        errors.add("Line " + lineNumber + ": missing 'messages' array");

                    boolean hasSystem    = false;
                    boolean hasUser      = false;
                    boolean hasAssistant = false;

                    for (JsonNode msg : messages) {
                        String role = msg.get("role").asText();
                        if ("system".equals(role))    hasSystem = true;
                        if ("user".equals(role))      hasUser = true;
                        if ("assistant".equals(role)) hasAssistant = true;
                    }

                    if (!hasUser || !hasAssistant)
                        errors.add("Line " + lineNumber + ": missing user or assistant turn");

                } catch (Exception e) {
                    errors.add("Line " + lineNumber + ": invalid JSON — " + e.getMessage());
                }
            }
        }

        if (!errors.isEmpty()) throw new IllegalStateException("Dataset errors:\n" + String.join("\n", errors));
        log.info("Dataset validated: {} examples, no errors", lineNumber);
    }
}
```

**Step 4 — Submit fine-tuning job (OpenAI example)**
```java
@Service
public class FineTuningJobService {

    private final OpenAiApi openAiApi; // Spring AI OpenAI integration

    public String submitJob(Path datasetPath) {
        // Upload file
        FileSystemResource file = new FileSystemResource(datasetPath);
        // Use OpenAI client directly for fine-tuning API (Spring AI doesn't wrap this yet)
        // See: https://platform.openai.com/docs/api-reference/fine-tuning

        // Programmatic equivalent of:
        // curl https://api.openai.com/v1/files -F purpose=fine-tune -F file=@dataset.jsonl
        // curl https://api.openai.com/v1/fine_tuning/jobs -d '{"training_file": "<file_id>", "model": "gpt-4o-mini"}'

        log.info("Submit via CLI: openai api fine_tuning.jobs.create -t <file_id> -m gpt-4o-mini");
        return "See OpenAI Fine-Tuning dashboard for job status";
    }
}
```

**Step 5 — Evaluate fine-tuned model vs base model**
```java
@SpringBootTest
class FineTuneEvaluationTest {

    @Test
    void fineTunedModelFollowsFormatBetter() {
        // Test 20 prompts against both models
        // Score: % of responses that contain STATUS/DELIVERY/ACTION lines
        // Fine-tuned should score higher than base model + few-shot prompt
        List<String> testPrompts = List.of(
            "Where is ORD-001?",
            "Check my order ORD-002",
            "Is ORD-999 delivered yet?"
        );

        long baseModelConforming = testPrompts.stream()
            .map(p -> baseModelClient.prompt().user(p).call().content())
            .filter(r -> r.contains("STATUS:") && r.contains("DELIVERY:") && r.contains("ACTION:"))
            .count();

        long fineTunedConforming = testPrompts.stream()
            .map(p -> fineTunedClient.prompt().user(p).call().content())
            .filter(r -> r.contains("STATUS:") && r.contains("DELIVERY:") && r.contains("ACTION:"))
            .count();

        log.info("Base model format compliance: {}/{}", baseModelConforming, testPrompts.size());
        log.info("Fine-tuned format compliance: {}/{}", fineTunedConforming, testPrompts.size());

        // Fine-tuned model should be equal or better
        assertThat(fineTunedConforming).isGreaterThanOrEqualTo(baseModelConforming);
    }
}
```

**Step 6 — Experiments**
1. Generate 20 training examples, write to `dataset.jsonl`, run the validator — fix any format errors.
2. Test the target format compliance of a base model with zero-shot vs few-shot prompting — establish a baseline.
3. Review 5 of your training examples manually — is the assistant response exactly what you want the model to learn? Inconsistencies here directly hurt fine-tune quality.
4. Review Hugging Face's LoRA quickstart (`peft` + `transformers`) — understand the open-source alternative that keeps your data on-prem.

---

## 4. Resources

- **[OpenAI Fine-Tuning Guide](https://platform.openai.com/docs/guides/fine-tuning)** — End-to-end walkthrough; dataset format, job submission, evaluation. Most production-applicable guide available.
- **[Hugging Face PEFT / LoRA](https://huggingface.co/docs/peft/index)** — Open-source parameter-efficient fine-tuning. Key for on-prem/compliance scenarios.
- **[Anthropic Fine-Tuning](https://docs.anthropic.com/en/docs/build-with-claude/fine-tuning)** — Claude fine-tuning API docs (enterprise).

---

## 5. Trending Context

**Fine-tuning is shifting from "custom models" to "behaviour alignment."** In 2025, fine-tuning is less about teaching the model new facts (RAG does that better) and more about locking in reliable behaviour: output format compliance, tone consistency, domain-specific reasoning patterns. The trend: **LoRA adapters** — lightweight fine-tunes that add ~1% of the base model's parameters and can be swapped in/out per customer or use-case, enabling multi-tenant model customisation without running separate model instances. Cloud providers (AWS Bedrock, Azure AI) now offer hosted LoRA fine-tuning with data isolation guarantees — no data leaves your cloud account.

---

## 6. Community Tip

**Your fine-tuning dataset is a product — version it like one.** Store your `dataset.jsonl` in Git, tag each version, and maintain a changelog of what changed between versions (added edge cases, fixed inconsistencies, removed PII). When a fine-tuned model regresses, you need to bisect between dataset versions, just like you'd bisect between code commits. Also: never fine-tune on your production traffic directly — first anonymise and manually curate a sample. Production traffic contains rare errors and edge cases that, if trained on, will teach the model to replicate those mistakes.

---

## 7. Tomorrow's Preview

**Day 15** will be a **Phase 1 Capstone & Review** — synthesising all 14 days into a complete agentic AI system design, reviewing key patterns, and planning your Phase 2 transition to System Design.

---

*Built progressively. Each day connects to the next.*
