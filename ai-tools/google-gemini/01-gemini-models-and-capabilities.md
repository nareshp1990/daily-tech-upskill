# 01 — Gemini Models and Capabilities

## Table of Contents

1. [Gemini Model Families](#1-gemini-model-families)
2. [When to Use Which Model](#2-when-to-use-which-model)
3. [Context Windows](#3-context-windows)
4. [Multimodal Capabilities](#4-multimodal-capabilities)
5. [Google AI Studio](#5-google-ai-studio)
6. [Gemini Advanced (Consumer Product)](#6-gemini-advanced-consumer-product)
7. [Comparison with Claude and GPT](#7-comparison-with-claude-and-gpt)
8. [Practical Examples](#8-practical-examples)
9. [Try This Exercises](#9-try-this-exercises)

---

## 1. Gemini Model Families

Google's Gemini lineup is organized into tiers based on capability and cost:

### Gemini 2.5 Pro (Flagship)

| Attribute | Value |
|-----------|-------|
| Model ID | `gemini-2.5-pro-preview-06-05` |
| Context Window | 1,048,576 tokens (1M) |
| Max Output | 65,536 tokens |
| Strengths | Best reasoning, coding, math, science, complex multi-step tasks |
| Multimodal | Text, image, audio, video input; text output |
| Thinking | Built-in "thinking" mode — model shows chain-of-thought reasoning |
| Price (input) | ~$1.25 / 1M tokens (<=200K), $2.50 / 1M tokens (>200K) |
| Price (output) | ~$10.00 / 1M tokens |

**Best for:** Complex code generation, architecture reviews, multi-file refactoring,
system design analysis, research synthesis.

### Gemini 2.5 Flash (Speed + Intelligence)

| Attribute | Value |
|-----------|-------|
| Model ID | `gemini-2.5-flash-preview-05-20` |
| Context Window | 1,048,576 tokens (1M) |
| Max Output | 65,536 tokens |
| Strengths | Excellent speed-to-intelligence ratio, "thinking budgets" |
| Thinking | Configurable — set thinking budget from 0 to 24,576 tokens |
| Price (input) | ~$0.15 / 1M tokens (<=200K) |
| Price (output) | ~$0.60 / 1M tokens (non-thinking), $3.50 / 1M tokens (thinking) |

**Best for:** High-volume API calls, real-time applications, chat, code completions,
summarization. Use the thinking budget dial to trade cost for reasoning depth.

### Gemini 2.0 Flash

| Attribute | Value |
|-----------|-------|
| Model ID | `gemini-2.0-flash` |
| Context Window | 1,048,576 tokens (1M) |
| Max Output | 8,192 tokens |
| Strengths | Fast, stable (GA), multimodal generation (images, audio output) |
| Special | Can generate images and audio natively |
| Price (input) | ~$0.10 / 1M tokens |
| Price (output) | ~$0.40 / 1M tokens |

**Best for:** Production workloads needing stability, image/audio generation,
agentic tool-use loops, lightweight tasks.

### Gemini 2.0 Flash Lite

| Attribute | Value |
|-----------|-------|
| Model ID | `gemini-2.0-flash-lite` |
| Context Window | 1,048,576 tokens (1M) |
| Max Output | 8,192 tokens |
| Strengths | Cheapest, fastest, lowest latency |
| Price (input) | ~$0.075 / 1M tokens |
| Price (output) | ~$0.30 / 1M tokens |

**Best for:** Classification, routing, simple extraction, high-QPS endpoints,
cost-sensitive batch processing.

### Gemini 1.5 Pro / 1.5 Flash (Legacy)

Still available but being superseded. Use 2.x models for new projects.

---

## 2. When to Use Which Model

```
Decision Tree:
                     ┌─ Need best reasoning/coding? ──> Gemini 2.5 Pro
                     │
Start ──> Complex? ──┤
                     │
                     └─ Good reasoning + fast? ──> Gemini 2.5 Flash (with thinking)

                     ┌─ Need image/audio output? ──> Gemini 2.0 Flash
                     │
Start ──> Simple? ───┤
                     │  ┌─ Need some smarts? ──> Gemini 2.0 Flash
                     └──┤
                        └─ Pure speed/cost? ──> Gemini 2.0 Flash Lite
```

### Model Selection by Task

| Task | Recommended Model | Why |
|------|-------------------|-----|
| Code review (complex PR) | 2.5 Pro | Deep reasoning over large diffs |
| Inline code completion | 2.5 Flash (low thinking) | Speed matters, moderate reasoning |
| Explain a stack trace | 2.0 Flash | Fast, sufficient reasoning |
| Batch classify 10K tickets | 2.0 Flash Lite | Cheapest per token |
| Analyze a 500-page PDF | 2.5 Pro | 1M context + best comprehension |
| Generate API docs | 2.5 Flash | Good writing, fast turnaround |
| Architecture diagram + explanation | 2.5 Pro | Complex multi-step output |
| Real-time chat assistant | 2.5 Flash | Balance of speed and quality |
| CI/CD log analysis | 2.0 Flash | Pattern matching, moderate reasoning |
| Generate test images | 2.0 Flash | Native image generation |

### Cost Comparison (per 1M tokens, input/output)

| Model | Input | Output | Relative Cost |
|-------|-------|--------|---------------|
| 2.5 Pro | $1.25 | $10.00 | $$$$$ |
| 2.5 Flash | $0.15 | $0.60 | $$ |
| 2.0 Flash | $0.10 | $0.40 | $ |
| 2.0 Flash Lite | $0.075 | $0.30 | $ |

> **Tip:** Gemini 2.5 Flash with a thinking budget of 0 behaves like a fast model.
> Increase the budget when you need better reasoning. This is a unique feature
> not available in Claude or GPT models.

---

## 3. Context Windows

All current Gemini models support a **1,048,576 token context window** (approximately
700K–800K words, or ~3,000 pages of text).

### What Fits in 1M Tokens

| Content | Approximate Size |
|---------|-----------------|
| An entire Spring Boot microservice (50 files) | ~50K tokens |
| A 300-page technical book | ~150K tokens |
| 1 hour of audio (transcribed) | ~15K tokens |
| 1 hour of video (with frames) | ~700K tokens |
| Full Kubernetes cluster manifests | ~20K tokens |
| Large database schema (500 tables) | ~100K tokens |

### Context Caching

For repeated queries against the same large context (e.g., a codebase), Gemini supports
**context caching** — you pay once to cache the context, then subsequent queries against
that cache are 75% cheaper on input tokens.

```
# Conceptual flow
1. Cache your codebase/docs (pay full input price once)
2. Ask 50 questions against it (pay reduced input price + output price)
3. Cache expires after TTL (default 1 hour, configurable)
```

> See [03-gemini-api-and-sdk.md](03-gemini-api-and-sdk.md) for implementation details.

---

## 4. Multimodal Capabilities

Gemini models natively understand multiple input types in a single request:

### Input Modalities

| Modality | Supported | Notes |
|----------|-----------|-------|
| Text | Yes | All models |
| Images | Yes | JPEG, PNG, GIF, WebP; up to 3,600 images per request |
| Audio | Yes | MP3, WAV, FLAC, AAC, OGG; up to ~9.5 hours |
| Video | Yes | MP4, AVI, MOV, MKV; up to ~1 hour |
| PDF | Yes | Treated as images (page by page) or extracted text |
| Code | Yes | Any programming language as text |

### Output Modalities

| Modality | 2.5 Pro | 2.5 Flash | 2.0 Flash | 2.0 Flash Lite |
|----------|---------|-----------|-----------|-----------------|
| Text | Yes | Yes | Yes | Yes |
| Images | No | No | Yes | No |
| Audio | No | No | Yes | No |
| JSON (structured) | Yes | Yes | Yes | Yes |

### Practical Multimodal Examples for Backend Devs

**1. Screenshot to code:**
Send a UI mockup screenshot, get HTML/CSS/Java template code back.

**2. Architecture diagram analysis:**
Upload a system architecture diagram image, ask Gemini to identify bottlenecks or
suggest improvements.

**3. Video code review:**
Record a screen walkthrough of your code, send the video, ask for a review.

**4. PDF spec to API:**
Upload a PDF API specification, ask Gemini to generate Spring Boot controller stubs.

**5. Log file analysis:**
Send a large log file as text, ask Gemini to identify error patterns.

---

## 5. Google AI Studio

**URL:** https://aistudio.google.com

Google AI Studio is the **free web-based playground** for Gemini models. Think of it as
your prototyping workbench before writing code.

### Key Features

| Feature | Description |
|---------|-------------|
| Chat mode | Multi-turn conversation with any Gemini model |
| Prompt gallery | Pre-built prompt templates for common tasks |
| System instructions | Set system prompts to control model behavior |
| Structured prompts | Build few-shot examples visually |
| File upload | Send images, audio, video, PDFs directly |
| Code execution | Model can write and run Python code |
| Grounding | Ground responses in Google Search results |
| API key management | Generate and manage API keys |
| Token counter | See exactly how many tokens your prompt uses |
| Export to code | Generate Python, JavaScript, Kotlin, Swift, cURL code |

### Getting an API Key

1. Go to https://aistudio.google.com/apikey
2. Click "Create API key"
3. Select or create a Google Cloud project
4. Copy the key — store it securely (never commit to Git)

### Free Tier Limits (Google AI Studio / Gemini API)

| Model | Free Tier RPM | Free Tier TPM | Free Tier RPD |
|-------|---------------|---------------|---------------|
| 2.5 Pro | 5 | 250K | 50 |
| 2.5 Flash | 10 | 250K | 500 |
| 2.0 Flash | 15 | 1M | 1,500 |
| 2.0 Flash Lite | 30 | 1M | 3,000 |

> RPM = requests per minute, TPM = tokens per minute, RPD = requests per day.
> Free tier is generous enough for prototyping and personal projects.

### Workflow: AI Studio for Prototyping

```
1. Open AI Studio → select model
2. Write your system prompt
3. Test with sample inputs
4. Iterate on prompt until output is good
5. Click "Get Code" → choose language
6. Paste generated code into your Spring Boot service
7. Replace hardcoded values with config properties
```

---

## 6. Gemini Advanced (Consumer Product)

Gemini Advanced is the **paid consumer product** available at https://gemini.google.com
with a Google One AI Premium subscription (~$20/month).

### What You Get

- Access to Gemini 2.5 Pro (full model)
- 1M token context window in chat
- Deep Research (automated multi-step research reports)
- Gems (custom AI personas)
- Gemini in Google Workspace (Docs, Sheets, Gmail, Slides)
- Gemini Live (real-time voice conversation)
- Priority access to new features
- NotebookLM Plus

### Gemini Advanced vs Free Gemini

| Feature | Free | Advanced |
|---------|------|----------|
| Model | 2.0 Flash | 2.5 Pro |
| Context window | Limited | 1M tokens |
| Deep Research | No | Yes |
| Gems | No | Yes |
| Workspace integration | No | Yes |
| File uploads | Limited | Large files |
| Image generation | Limited | Full |
| Priority access | No | Yes |

### Is It Worth It for Developers?

**Yes, if** you want Deep Research for technical learning, Workspace AI for documentation,
and a powerful 2.5 Pro chat interface without writing code.

**No, if** you only need the API — the free tier API access to 2.5 Pro may be sufficient
for testing, and you pay per token in production.

---

## 7. Comparison with Claude and GPT

### Model Tier Comparison

| Tier | Google | Anthropic | OpenAI |
|------|--------|-----------|--------|
| Flagship | Gemini 2.5 Pro | Claude Opus 4 | GPT-4.1 / o3 |
| Balanced | Gemini 2.5 Flash | Claude Sonnet 4 | GPT-4.1-mini |
| Fast/Cheap | Gemini 2.0 Flash Lite | Claude Haiku 3.5 | GPT-4.1-nano |

### Feature Comparison

| Feature | Gemini 2.5 Pro | Claude Opus 4 | GPT-4.1 |
|---------|----------------|---------------|---------|
| Max context | 1M tokens | 200K tokens | 1M tokens |
| Multimodal input | Text, image, audio, video | Text, image | Text, image |
| Audio input | Native | No | Yes (via Whisper) |
| Video input | Native (up to 1 hr) | No | No |
| Image output | 2.0 Flash only | No | DALL-E integration |
| Thinking/reasoning | Built-in (2.5 Pro/Flash) | Extended thinking | o3/o4-mini |
| Code execution | Yes (Python sandbox) | Yes (tool use) | Yes (Code Interpreter) |
| Grounding (web search) | Native | Tool use | Web browsing |
| Function calling | Yes | Yes (tool use) | Yes |
| Structured output | JSON schema | JSON mode | JSON schema |
| Streaming | Yes | Yes | Yes |
| Context caching | Yes (75% discount) | Yes (prompt caching) | No |

### Strengths by Provider

| Use Case | Best Choice | Why |
|----------|-------------|-----|
| Code generation & review | Claude Opus 4 / Gemini 2.5 Pro | Both excellent; Claude better at following complex instructions |
| Long document analysis | Gemini 2.5 Pro | 1M native context, multimodal |
| Video understanding | Gemini 2.5 Pro | Only model with native video input |
| Speed + cost at scale | Gemini 2.5 Flash | Thinking budget dial, cheapest mid-tier |
| Complex reasoning | Claude Opus 4 / o3 | Slight edge in nuanced reasoning |
| IDE integration | Gemini Code Assist / GitHub Copilot | Both mature; Copilot has wider ecosystem |
| Agentic workflows | Claude (tool use) / Gemini | Both strong; Claude's tool use is polished |
| Enterprise compliance | All three | Azure OpenAI, GCP Vertex AI, AWS Bedrock |

### Pricing Comparison (per 1M tokens)

| Model | Input | Output |
|-------|-------|--------|
| Gemini 2.5 Pro | $1.25 | $10.00 |
| Claude Opus 4 | $15.00 | $75.00 |
| GPT-4.1 | $2.00 | $8.00 |
| Gemini 2.5 Flash | $0.15 | $0.60 |
| Claude Sonnet 4 | $3.00 | $15.00 |
| GPT-4.1-mini | $0.40 | $1.60 |

> Gemini is typically the cheapest option at every tier. The trade-off is that
> Google's API ecosystem and documentation can be more fragmented than Anthropic's
> or OpenAI's.

---

## 8. Practical Examples

### Example 1: Code Review with Gemini

**In Google AI Studio:**

System instruction:
```
You are a senior Java code reviewer. Review the following code for:
1. Bugs and edge cases
2. Performance issues
3. Security vulnerabilities
4. Spring Boot best practices
5. Clean code principles

Be specific. Reference line numbers. Suggest fixes with code.
```

User prompt:
```
Review this Spring Boot REST controller:

@RestController
@RequestMapping("/api/users")
public class UserController {

    @Autowired
    private UserService userService;

    @GetMapping("/{id}")
    public User getUser(@PathVariable Long id) {
        return userService.findById(id);
    }

    @PostMapping
    public User createUser(@RequestBody User user) {
        return userService.save(user);
    }

    @DeleteMapping("/{id}")
    public void deleteUser(@PathVariable Long id) {
        userService.deleteById(id);
    }
}
```

**Gemini will identify:**
- Missing `@Valid` on `@RequestBody`
- No `ResponseEntity` wrappers (no HTTP status control)
- No error handling (what if user not found?)
- Field injection (`@Autowired`) instead of constructor injection
- No pagination on potential list endpoints
- No DTO layer — exposing entity directly
- Missing `@Transactional` considerations

### Example 2: Debugging with Gemini

Paste a stack trace along with relevant code:

```
I'm getting this exception in production. Here's the stack trace and the relevant
service code. What's the root cause and how do I fix it?

Exception:
org.springframework.dao.DataIntegrityViolationException:
could not execute statement; SQL [n/a]; constraint [uk_users_email]

[paste your service code here]
```

### Example 3: Architecture Analysis

Upload your system architecture diagram as an image to AI Studio, then ask:

```
Analyze this microservices architecture diagram. Identify:
1. Single points of failure
2. Missing patterns (circuit breaker, retry, etc.)
3. Potential bottlenecks under 10x traffic increase
4. Security gaps
5. Suggestions for improvement
```

---

## 9. Try This Exercises

### Exercise 1: Explore AI Studio (15 min)
1. Go to https://aistudio.google.com
2. Select Gemini 2.5 Pro
3. In system instructions, write: "You are a Spring Boot expert. Always provide code examples in Java 17+ with Spring Boot 3.x."
4. Ask it to review a piece of your own code
5. Try uploading a screenshot of an error and asking for help

### Exercise 2: Model Comparison (15 min)
1. In AI Studio, ask the same complex coding question to:
   - Gemini 2.5 Pro
   - Gemini 2.5 Flash
   - Gemini 2.0 Flash
2. Compare: response quality, speed, token usage
3. Note where 2.5 Flash gives you 90% of Pro's quality at 10% of the cost

### Exercise 3: Multimodal Test (10 min)
1. Take a screenshot of a database schema or ER diagram
2. Upload it to AI Studio
3. Ask Gemini to generate JPA entity classes from the diagram
4. Compare the output quality across models

### Exercise 4: Get Your API Key (5 min)
1. Go to https://aistudio.google.com/apikey
2. Create an API key
3. Store it in your password manager or `~/.env` (never in Git)
4. You will use this key in the API tutorial (03)

---

## Key Takeaways

1. **Gemini 2.5 Pro** is the flagship — best for complex tasks, 1M token context
2. **Gemini 2.5 Flash** is the sweet spot — tunable thinking budget, great price/performance
3. **Gemini 2.0 Flash** is fast and stable — good for production, can generate images/audio
4. **Google AI Studio** is your free prototyping playground — always start here
5. **Context caching** is a killer feature for codebase-level analysis
6. Gemini is the **cheapest option** at every tier compared to Claude and GPT
7. Gemini's unique advantage is **native multimodal** (especially video) and **thinking budgets**

---

Next: [02 — Gemini in the IDE](02-gemini-in-ide.md)
