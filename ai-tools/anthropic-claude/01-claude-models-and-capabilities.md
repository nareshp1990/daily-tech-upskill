# Tutorial 01: Claude Models and Capabilities

## Table of Contents

1. [Claude Model Families](#1-claude-model-families)
2. [Model Selection Guide](#2-model-selection-guide)
3. [Context Windows and Token Counting](#3-context-windows-and-token-counting)
4. [Multimodal Capabilities](#4-multimodal-capabilities)
5. [Key API Parameters](#5-key-api-parameters)
6. [Claude.ai Web Interface Power Features](#6-claudeai-web-interface-power-features)
7. [Practical Examples for Backend Developers](#7-practical-examples-for-backend-developers)
8. [Try This Exercises](#8-try-this-exercises)
9. [Tips and Gotchas](#9-tips-and-gotchas)

---

## 1. Claude Model Families

Anthropic organises Claude into three model tiers, each optimised for different workloads.

### Claude Opus (Flagship)

| Property | Value |
|----------|-------|
| Latest Version | Claude Opus 4 (claude-opus-4-0520) |
| Model ID (API) | `claude-opus-4-0520` or alias `claude-opus-4-20250514` |
| Context Window | 200K tokens |
| Max Output | 32K tokens (default), up to 64K with extended thinking |
| Strengths | Complex reasoning, agentic coding, multi-step planning, research |
| Cost (approx.) | $15 / 1M input tokens, $75 / 1M output tokens |

**When to use Opus:** Architecture reviews, complex debugging sessions, multi-file refactoring, system design, agentic workflows that require deep reasoning. This is the model powering Claude Code by default.

### Claude Sonnet (Balanced)

| Property | Value |
|----------|-------|
| Latest Version | Claude Sonnet 4 (claude-sonnet-4-0520) |
| Model ID (API) | `claude-sonnet-4-0520` or alias `claude-sonnet-4-20250514` |
| Previous | Claude 3.5 Sonnet (`claude-3-5-sonnet-20241022`) |
| Context Window | 200K tokens |
| Max Output | 16K tokens (default), up to 64K with extended thinking |
| Strengths | Best balance of speed, cost, and capability |
| Cost (approx.) | $3 / 1M input tokens, $15 / 1M output tokens |

**When to use Sonnet:** Day-to-day coding assistance, code generation, API design, writing tests, documentation. The sweet spot for most development tasks.

### Claude Haiku (Fast & Lightweight)

| Property | Value |
|----------|-------|
| Latest Version | Claude 3.5 Haiku (`claude-3-5-haiku-20241022`) |
| Context Window | 200K tokens |
| Max Output | 8K tokens |
| Strengths | Speed, low cost, high throughput |
| Cost (approx.) | $0.80 / 1M input tokens, $4 / 1M output tokens |

**When to use Haiku:** Classification tasks, log analysis, simple Q&A, routing decisions, high-volume API calls where cost matters, real-time applications.

### Model ID Quick Reference

```
# Production-ready model IDs for API calls
claude-opus-4-0520          # Opus 4 — most capable
claude-sonnet-4-0520        # Sonnet 4 — balanced
claude-3-5-haiku-20241022   # Haiku 3.5 — fast & cheap

# Aliases (point to latest within family)
claude-opus-4-0520
claude-sonnet-4-0520
```

---

## 2. Model Selection Guide

### Decision Matrix

```
Need complex reasoning / multi-step agents?  ──── Opus
Need balanced speed + quality for coding?     ──── Sonnet
Need fast responses at scale / low cost?      ──── Haiku
```

### Cost Comparison (per 1M tokens)

| Model | Input | Output | Relative Cost |
|-------|-------|--------|---------------|
| Opus 4 | $15 | $75 | 5x Sonnet |
| Sonnet 4 | $3 | $15 | 1x (baseline) |
| Haiku 3.5 | $0.80 | $4 | ~0.27x Sonnet |

### Real-World Scenario Mapping

| Task | Recommended Model | Reasoning |
|------|-------------------|-----------|
| Complex system design discussion | Opus | Deep reasoning needed |
| Review a 500-line PR | Sonnet | Good balance of quality and cost |
| Generate unit tests for a service | Sonnet | Reliable code generation |
| Classify 10K support tickets | Haiku | High volume, simple task |
| Debug a race condition | Opus | Complex multi-step reasoning |
| Write a Dockerfile | Sonnet | Straightforward generation |
| Parse log files for patterns | Haiku | Fast, repetitive analysis |
| Refactor a monolith service | Opus | Multi-file understanding |
| Generate API documentation | Sonnet | Good writing + code understanding |
| Validate JSON schema inputs | Haiku | Simple validation at scale |

---

## 3. Context Windows and Token Counting

### What is a Context Window?

The context window is the total amount of text (measured in tokens) a model can process in a single conversation turn. All Claude models currently support **200,000 tokens** (~150K words or ~500 pages of text).

### Token Counting Rules of Thumb

```
1 token ≈ 4 characters in English
1 token ≈ 0.75 words
1,000 tokens ≈ 750 words

# Code tends to be more token-dense:
100 lines of Java ≈ 800-1200 tokens
A typical Spring Boot controller (200 lines) ≈ 1500-2500 tokens
A pom.xml with 50 dependencies ≈ 2000-3000 tokens
```

### What Fits in 200K Tokens?

```
~150,000 words of text
~500 pages of documentation
~300-500 source files (depending on size)
An entire medium-sized Spring Boot microservice codebase
```

### Counting Tokens via the API

```java
// The API response includes token usage
// In the response JSON:
{
  "usage": {
    "input_tokens": 2095,
    "output_tokens": 503,
    "cache_creation_input_tokens": 0,
    "cache_read_input_tokens": 0
  }
}
```

### Prompt Caching (Cost Optimisation)

Claude supports **prompt caching** for repeated prefixes. If your system prompt or context stays the same across calls, cached tokens cost significantly less:

- Cached input tokens: 90% cheaper than regular input
- Cache write: 25% more expensive on first call
- Cache TTL: 5 minutes (refreshed on each use)

This is critical for Spring Boot services making repeated API calls with the same system prompt.

---

## 4. Multimodal Capabilities

### Vision (Image Understanding)

Claude can analyse images passed via the API or pasted into the web interface.

**Supported formats:** JPEG, PNG, GIF, WebP
**Max image size:** 20MB per image

**Use cases for developers:**
- Screenshot-to-code: paste a UI mockup, get HTML/CSS
- Diagram analysis: paste architecture diagrams, get feedback
- Error screenshot debugging: paste error screenshots for diagnosis
- Whiteboard-to-requirements: convert whiteboard photos to user stories

```java
// API example: sending an image to Claude
Message message = Message.builder()
    .role("user")
    .content(List.of(
        ContentBlock.ofImage(ImageSource.builder()
            .type("base64")
            .mediaType("image/png")
            .data(base64EncodedImage)
            .build()),
        ContentBlock.ofText("What Spring Boot configuration issue do you see in this screenshot?")
    ))
    .build();
```

### PDF Processing

Claude can read and analyse PDF documents directly.

**Use cases:**
- Reviewing technical specifications
- Extracting requirements from PDF documents
- Analysing compliance documents

### Code Understanding

Claude excels at understanding code across many languages, with particular strength in:
- Java, Kotlin, Python, TypeScript, Go, Rust, C/C++
- Framework-specific patterns (Spring Boot, React, etc.)
- Build files (pom.xml, build.gradle, package.json)
- Configuration files (YAML, properties, JSON)
- Infrastructure as Code (Terraform, CloudFormation, K8s manifests)

---

## 5. Key API Parameters

### System Prompt

The system prompt sets Claude's behaviour for the entire conversation. It is processed before any user messages.

```
System prompt: "You are a senior Java architect reviewing code for a
high-traffic e-commerce platform. Focus on thread safety, performance,
and Spring Boot best practices. Always suggest concrete improvements
with code examples."
```

**Best practices:**
- Be specific about the role and context
- Include coding standards and conventions
- Specify output format preferences
- Include relevant tech stack details

### Temperature

Controls randomness in responses. Range: 0.0 to 1.0.

| Temperature | Behaviour | Use Case |
|-------------|-----------|----------|
| 0.0 | Deterministic, focused | Code generation, factual answers |
| 0.3-0.5 | Slightly varied | General coding assistance |
| 0.7-1.0 | Creative, diverse | Brainstorming, creative writing |

**For code generation, use temperature 0.0 to 0.3.** Higher values introduce unnecessary variation in code output.

### Max Tokens

Controls the maximum length of Claude's response.

```
Default: varies by model (8K-32K)
Maximum: 64K (with extended thinking on Opus/Sonnet)

# Rule of thumb for code generation:
- Simple function: 500-1000 tokens
- Full class: 2000-4000 tokens
- Multiple files: 8000-16000 tokens
```

### Extended Thinking

Opus and Sonnet support "extended thinking" where the model reasons step-by-step before responding. This is particularly useful for:
- Complex debugging
- Architecture decisions
- Multi-step refactoring plans

```json
{
  "model": "claude-opus-4-0520",
  "max_tokens": 16000,
  "thinking": {
    "type": "enabled",
    "budget_tokens": 10000
  },
  "messages": [...]
}
```

---

## 6. Claude.ai Web Interface Power Features

### Projects

Projects let you organise related conversations with shared context.

**Setup for a Spring Boot project:**
1. Create a new Project (e.g., "Order Service Rewrite")
2. Add project instructions (acts as a persistent system prompt)
3. Upload relevant files (architecture docs, API specs, key source files)
4. All conversations within the project share this context

**Project instructions example:**
```
This project is for the Order Service, a Spring Boot 3.2 microservice.
Tech stack: Java 21, Spring Boot 3.2, PostgreSQL, Kafka, Docker, K8s on Azure.
Follow our coding standards:
- Use records for DTOs
- Use Optional.ofNullable, never return null
- All public methods need Javadoc
- Use constructor injection, never @Autowired on fields
- Integration tests use Testcontainers
```

### Artifacts

Artifacts are standalone, rendered content blocks that Claude creates. They appear in a side panel and can be:
- Code files (syntax highlighted, copyable)
- HTML/React apps (live preview)
- SVG diagrams
- Markdown documents

**Power tip:** Ask Claude to create architecture diagrams as SVG artifacts. You get editable vector graphics of your system design.

### Styles

Custom styles let you control Claude's response format:
- **Concise:** Short, direct answers (great for quick coding questions)
- **Explanatory:** Detailed with reasoning (good for learning)
- **Formal:** Professional tone (documentation, reports)
- **Custom:** Define your own style

**Recommended custom style for backend development:**
```
Be concise and technical. Skip basic explanations unless asked.
Show code first, explain after. Use Java 21+ syntax.
Include error handling in all code examples.
Mention performance implications when relevant.
```

### Keyboard Shortcuts

| Shortcut | Action |
|----------|--------|
| `/` | Open command menu |
| `Ctrl+Shift+C` | Copy last code block |
| `Ctrl+/` | Toggle sidebar |

---

## 7. Practical Examples for Backend Developers

### Example 1: Code Review

```
Prompt: "Review this Spring Boot controller for security issues,
performance concerns, and adherence to REST best practices:

@RestController
@RequestMapping("/api/orders")
public class OrderController {

    @Autowired
    private OrderService orderService;

    @GetMapping
    public List<Order> getAllOrders() {
        return orderService.findAll();
    }

    @PostMapping
    public Order createOrder(@RequestBody Order order) {
        return orderService.save(order);
    }

    @DeleteMapping("/{id}")
    public void deleteOrder(@PathVariable String id) {
        orderService.delete(Long.parseLong(id));
    }
}"
```

Claude will identify: field injection, missing validation, unbounded list return, missing response status codes, lack of error handling, SQL injection risk in the delete path, and more.

### Example 2: Architecture Discussion

```
Prompt: "I'm designing an event-driven order processing system.
Requirements:
- 10K orders/minute peak
- Exactly-once processing guarantee
- Order states: CREATED → VALIDATED → PAID → SHIPPED → DELIVERED
- Need to integrate with payment gateway (async callback)
- Must handle partial failures (payment succeeds but inventory fails)

Current stack: Spring Boot, Kafka, PostgreSQL, Azure.
Propose an architecture with component diagram."
```

### Example 3: Debugging

```
Prompt: "My Spring Boot app throws this intermittent error under load:

org.springframework.dao.CannotAcquireConnectionException:
Failed to obtain JDBC Connection; nested exception is
java.sql.SQLTransientConnectionException: HikariPool-1 -
Connection is not available, request timed out after 30000ms.

HikariCP config:
  maximum-pool-size: 10
  minimum-idle: 5
  connection-timeout: 30000

The app handles ~500 requests/second with 3 replicas on K8s.
PostgreSQL is on Azure Database (4 vCores, 16GB RAM).
What's wrong and how do I fix it?"
```

---

## 8. Try This Exercises

### Exercise 1: Model Comparison
Open Claude.ai, create the same prompt, and compare responses between Sonnet and Opus:
```
"Design a rate limiter for a Spring Boot API gateway that supports:
- Per-user rate limits
- Per-endpoint rate limits
- Sliding window algorithm
- Redis-backed for distributed deployment
Show me the full implementation."
```
Notice the difference in depth and reasoning between models.

### Exercise 2: System Prompt Engineering
Create a Project in Claude.ai with these project instructions, then have a conversation about your current codebase:
```
You are a senior Java architect specialising in Spring Boot microservices.
When reviewing code: check thread safety, connection leaks, N+1 queries,
proper exception handling, and missing validation.
Always suggest the most idiomatic Java 21+ approach.
Format responses as: Problem → Impact → Solution → Code.
```

### Exercise 3: Multimodal Debugging
Take a screenshot of an error in your IDE or terminal and paste it into Claude.ai. Ask Claude to diagnose the issue. Compare this with typing out the error manually.

### Exercise 4: Architecture Diagram
Ask Claude to create an SVG artifact showing the architecture of your current microservices setup. Iterate on it by asking for changes (add a service, change a connection, etc.).

---

## 9. Tips and Gotchas

### Tips

1. **Be specific about your stack.** "Review this Java code" gives generic feedback. "Review this Spring Boot 3.2 / Java 21 controller for a high-traffic e-commerce API" gives targeted feedback.

2. **Use system prompts or project instructions** to avoid repeating context in every message. Set up your tech stack, coding standards, and preferences once.

3. **Paste error messages completely.** Include the full stack trace, not just the exception message. Claude can trace through the call stack.

4. **Ask for trade-offs, not just solutions.** "What are three approaches to implement this, with pros and cons?" gives you better architectural decisions than "How should I implement this?"

5. **Use Claude for rubber duck debugging.** Explain the problem step by step; Claude often catches the issue in your explanation.

6. **Leverage extended thinking for complex problems.** When using the API, enable extended thinking for debugging race conditions, designing distributed systems, or planning large refactors.

### Gotchas

1. **Claude does not have real-time information.** It cannot check if a Maven dependency version exists or fetch current API documentation. Always verify version numbers.

2. **Token limits apply to both input and output combined.** A 200K context window means your prompt + Claude's response must fit in 200K tokens total.

3. **Claude may hallucinate API methods or library features.** Always verify generated code compiles and the methods actually exist in the libraries you are using.

4. **Costs add up with Opus.** A single complex conversation with Opus can cost $1-5+. Use Sonnet for routine tasks and reserve Opus for the hard problems.

5. **Claude does not execute code.** It generates code but cannot run it (unless you are using Claude Code, which can execute commands). Always test generated code.

6. **Image quality matters.** Blurry screenshots or low-resolution architecture diagrams will get less accurate analysis. Use clear, readable images.

7. **Context window is not memory.** Each API call is stateless. Previous conversations are not remembered unless you include them in the prompt or use Claude.ai's conversation history.

---

## Summary

| Concept | Key Takeaway |
|---------|-------------|
| Models | Opus for complex reasoning, Sonnet for daily coding, Haiku for high-volume simple tasks |
| Context | 200K tokens across all models — enough for an entire microservice codebase |
| Multimodal | Send images, PDFs, and code for analysis |
| Parameters | Use low temperature (0-0.3) for code, system prompts for persistent context |
| Web Features | Projects for organised context, Artifacts for rendered output, Styles for format control |
| Cost | Sonnet is the sweet spot; use prompt caching for repeated system prompts |

**Next:** [Tutorial 02 — Claude Code Setup and Basics](02-claude-code-setup-and-basics.md)
