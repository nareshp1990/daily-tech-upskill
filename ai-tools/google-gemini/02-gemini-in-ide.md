# 02 — Gemini in the IDE

## Table of Contents

1. [Gemini Code Assist Overview](#1-gemini-code-assist-overview)
2. [Installation and Configuration](#2-installation-and-configuration)
3. [Code Completions](#3-code-completions)
4. [Chat Panel](#4-chat-panel)
5. [Code Generation, Explanation, and Debugging](#5-code-generation-explanation-and-debugging)
6. [Gemini in Android Studio](#6-gemini-in-android-studio)
7. [Gemini in Cloud Shell and Cloud Workstations](#7-gemini-in-cloud-shell-and-cloud-workstations)
8. [Firebase Integration with Gemini](#8-firebase-integration-with-gemini)
9. [Practical Examples for Java/Spring Boot](#9-practical-examples-for-javaspring-boot)
10. [Comparison: Gemini Code Assist vs Copilot vs Claude Code](#10-comparison-gemini-code-assist-vs-copilot-vs-claude-code)
11. [Try This Exercises](#11-try-this-exercises)

---

## 1. Gemini Code Assist Overview

**Gemini Code Assist** (formerly "Duet AI for Developers") is Google's AI-powered
coding assistant that integrates directly into your IDE. It provides:

- Inline code completions (like Copilot's ghost text)
- A chat panel for asking questions about code
- Code generation from natural language
- Code explanation and transformation
- Debugging assistance
- Test generation

### Editions

| Edition | Price | Features |
|---------|-------|----------|
| **Individual** (free) | $0 | Code completions, chat, limited usage per day |
| **Standard** | $19/user/month | Higher limits, enterprise code customization, admin controls |
| **Enterprise** | $45/user/month | Code customization with your codebase, full Vertex AI integration, Gemini 2.5 Pro |

> The free Individual tier is sufficient for trying out all features. The Enterprise
> tier adds "code customization" — the model learns your team's codebase patterns.

### Supported IDEs

- VS Code (via "Gemini Code Assist" extension)
- JetBrains IDEs — IntelliJ IDEA, PyCharm, GoLand, WebStorm, etc. (via plugin)
- Android Studio (built-in starting from Hedgehog+)
- Cloud Shell Editor (built-in)
- Cloud Workstations (built-in)
- Neovim / Vim (via third-party community plugins)

---

## 2. Installation and Configuration

### VS Code

```bash
# Step 1: Install the extension
# Open VS Code → Extensions (Ctrl+Shift+X) → Search "Gemini Code Assist" → Install

# Step 2: Sign in
# Click the Gemini icon in the sidebar → Sign in with Google
# For Individual: use your Google account
# For Enterprise: use your Google Cloud project

# Step 3: Verify
# Open a Java file → start typing → you should see ghost text suggestions
```

**Settings to configure (VS Code `settings.json`):**
```json
{
  "gemini.codeAssist.enableInlineCompletion": true,
  "gemini.codeAssist.enableChat": true,
  "gemini.codeAssist.languages": ["java", "kotlin", "python", "typescript", "yaml", "dockerfile"],
  "gemini.codeAssist.inlineSuggestionDelay": 300
}
```

### IntelliJ IDEA

```
Step 1: File → Settings → Plugins → Marketplace → Search "Gemini Code Assist"
Step 2: Install and restart IDE
Step 3: Tools → Gemini Code Assist → Sign in
Step 4: Verify: open a Java file, start typing, see inline suggestions
```

**IntelliJ-specific settings:**
- `Settings → Tools → Gemini Code Assist`
- Enable/disable inline completions
- Configure keyboard shortcuts
- Set up Google Cloud project (for enterprise features)

### Authentication Options

| Method | Use Case |
|--------|----------|
| Google account (personal) | Individual free tier |
| Google Cloud project | Enterprise features, billing |
| Service account | CI/CD, automated workflows |
| Workforce Identity Federation | SSO with your corporate IdP |

### Proxy Configuration (Corporate Networks)

If you are behind a corporate proxy:
```json
// VS Code settings.json
{
  "http.proxy": "http://proxy.corp.com:8080",
  "http.proxyStrictSSL": false
}
```

For IntelliJ:
```
Settings → Appearance & Behavior → System Settings → HTTP Proxy
```

---

## 3. Code Completions

### How Inline Completions Work

Gemini analyzes your current file, open files, and recent edits to suggest code as you type.
Suggestions appear as gray "ghost text."

```java
// Example: Type this and pause...
@Service
public class OrderService {

    private final OrderRepository orderRepository;
    private final KafkaTemplate<String, OrderEvent> kafkaTemplate;

    // Gemini suggests the constructor:
    public OrderService(OrderRepository orderRepository,
                        KafkaTemplate<String, OrderEvent> kafkaTemplate) {
        this.orderRepository = orderRepository;
        this.kafkaTemplate = kafkaTemplate;
    }

    // Type "public Order create" and Gemini suggests:
    public Order createOrder(CreateOrderRequest request) {
        Order order = Order.builder()
            .customerId(request.getCustomerId())
            .items(request.getItems())
            .status(OrderStatus.PENDING)
            .createdAt(Instant.now())
            .build();

        Order saved = orderRepository.save(order);
        kafkaTemplate.send("order-events", new OrderCreatedEvent(saved.getId()));
        return saved;
    }
}
```

### Keyboard Shortcuts

| Action | VS Code | IntelliJ |
|--------|---------|----------|
| Accept suggestion | `Tab` | `Tab` |
| Dismiss suggestion | `Esc` | `Esc` |
| Accept word-by-word | `Ctrl+Right` | `Ctrl+Right` |
| Trigger manually | `Ctrl+Space` (then wait) | `Alt+\` |
| Next suggestion | `Alt+]` | `Alt+]` |
| Previous suggestion | `Alt+[` | `Alt+[` |

### Tips for Better Completions

1. **Write clear method signatures first** — Gemini uses the method name, parameters,
   and return type as strong signals:
   ```java
   // Good: clear intent
   public List<OrderSummaryDto> findOrdersByCustomerIdAndDateRange(
       Long customerId, LocalDate from, LocalDate to) {
       // Gemini knows exactly what to generate
   }
   ```

2. **Add Javadoc before the method** — the comment guides the completion:
   ```java
   /**
    * Retries the failed Kafka message up to 3 times with exponential backoff.
    * If all retries fail, sends to the dead-letter topic.
    */
   public void retryFailedMessage(ConsumerRecord<String, String> record) {
       // Better completions because intent is documented
   }
   ```

3. **Keep related files open** — Gemini reads your open tabs for context.

4. **Use descriptive variable names** — `kafkaTemplate` gives more context than `kt`.

---

## 4. Chat Panel

### Opening the Chat

- **VS Code:** Click the Gemini icon in the sidebar, or `Ctrl+Shift+P` → "Gemini: Open Chat"
- **IntelliJ:** View → Tool Windows → Gemini

### What You Can Do in Chat

| Action | Example Prompt |
|--------|----------------|
| Explain code | "Explain what this Kafka consumer configuration does" |
| Generate code | "Write a Spring Security config with JWT authentication" |
| Refactor | "Refactor this method to use Java Streams" |
| Debug | "Why might this method throw ConcurrentModificationException?" |
| Generate tests | "Write unit tests for this service class using JUnit 5 and Mockito" |
| SQL help | "Write a MySQL query to find duplicate orders in the last 30 days" |
| DevOps | "Write a Dockerfile for this Spring Boot app with multi-stage build" |
| Architecture | "Is this a good approach for handling eventual consistency?" |

### Context-Aware Chat

The chat panel has access to:
- **Selected code** — highlight code, right-click → "Gemini: Explain This"
- **Current file** — the model sees your open file
- **Workspace context** — in Enterprise tier, the model indexes your repo

### Chat Slash Commands

In the Gemini chat panel, you can use:

| Command | What It Does |
|---------|--------------|
| `/explain` | Explain selected code |
| `/generate` | Generate code from description |
| `/fix` | Suggest a fix for selected code |
| `/tests` | Generate tests for selected code |
| `/doc` | Generate documentation |
| `/transform` | Transform code (e.g., to a different pattern) |

### Example Chat Session

```
You: /explain [selected code: a complex Kafka consumer with retry logic]

Gemini: This is a Kafka consumer configured with Spring's @RetryableTopic annotation...
[detailed explanation of each part]

You: Can you add dead-letter topic handling and make it idempotent?

Gemini: Here's the updated version with DLT and idempotency:
[generates updated code with @DltHandler and IdempotentRepository]

You: Write integration tests for this

Gemini: [generates @EmbeddedKafka test class with JUnit 5]
```

---

## 5. Code Generation, Explanation, and Debugging

### Code Generation

**From the editor (inline):** Write a comment describing what you want:

```java
// Generate a REST endpoint that accepts a CSV file upload,
// parses it using OpenCSV, validates each row against UserDto schema,
// and bulk-inserts valid rows into MySQL using JdbcTemplate batch update
@PostMapping("/import")
```

Gemini generates the full implementation below your comment.

**From chat:** Describe what you need in natural language:

```
Generate a complete Spring Boot configuration class that sets up:
1. HikariCP connection pool with MySQL (min 5, max 20 connections)
2. Redis cache with Lettuce client (TTL 15 minutes)
3. Kafka producer with JSON serializer and idempotence enabled
4. Resilience4j circuit breaker (50% failure rate, 10 sec wait)
Use @ConfigurationProperties for all external values.
```

### Code Explanation

Right-click on any code block → "Gemini: Explain This"

Works especially well for:
- Complex SQL queries
- Regex patterns
- Kubernetes manifests
- Terraform configurations
- Docker Compose files
- Build tool configurations (Gradle/Maven)

### Debugging Assistance

**Paste an error in chat:**
```
I'm getting this error when deploying to Kubernetes:

CrashLoopBackOff - Back-off restarting failed container

Here are my deployment.yaml, Dockerfile, and application.properties.
[paste files]
```

**Gemini will analyze:**
- Whether the container starts and crashes (check liveness probe)
- Environment variable issues
- Port mismatches
- Memory/CPU limits too low
- Missing config maps or secrets

---

## 6. Gemini in Android Studio

Android Studio (Hedgehog 2023.1.1+) has Gemini built in:

- **Studio Bot** — AI chat for Android development questions
- **Code completions** — Android-specific completions (Compose, Activities, etc.)
- **Crash analysis** — explain crash reports from Firebase Crashlytics
- **App Quality Insights** — AI-powered analysis of ANRs and crashes

> Relevance for backend devs: If you maintain backend APIs consumed by Android apps,
> you can use Android Studio's Gemini to understand client-side behavior and generate
> matching client code.

---

## 7. Gemini in Cloud Shell and Cloud Workstations

### Cloud Shell Editor

Google Cloud Shell (https://shell.cloud.google.com) includes a web-based VS Code editor
with Gemini Code Assist **built in** — no installation needed.

**Use case:** Quick edits to Terraform, YAML manifests, or scripts directly in GCP
without setting up a local environment.

### Cloud Workstations

Fully managed development environments on GCP (like GitHub Codespaces) with Gemini
Code Assist pre-installed.

```
# Set up a Cloud Workstation with Gemini
gcloud workstations configs create my-config \
  --region=us-central1 \
  --machine-type=e2-standard-4 \
  --enable-gemini-code-assist
```

> **Azure parallel:** Similar to Azure Dev Box + GitHub Copilot, but fully managed
> within GCP.

---

## 8. Firebase Integration with Gemini

Firebase now includes Gemini assistance for:

- **Firestore security rules** — generate and explain rules
- **Cloud Functions** — generate function boilerplate
- **Crashlytics** — AI-powered crash analysis
- **Performance monitoring** — explain performance regressions
- **Firebase Extensions** — discover and configure extensions

Access via: Firebase Console → look for Gemini sparkle icon, or Firebase CLI with
`firebase assist`.

---

## 9. Practical Examples for Java/Spring Boot

### Example 1: Generate a Complete REST Endpoint

In IntelliJ chat:
```
Generate a paginated search endpoint for Products with the following:
- Search by name (partial match), category, price range
- Sort by name, price, or createdAt
- Use Spring Data JPA Specifications for dynamic queries
- Return Page<ProductDto> using MapStruct mapper
- Include proper validation annotations
- Add OpenAPI annotations for Swagger docs
```

### Example 2: Generate Kafka Consumer with Error Handling

Select your Kafka configuration class, then in chat:
```
Based on this config, generate a Kafka consumer for "order-events" topic that:
1. Deserializes OrderEvent from JSON
2. Retries 3 times with exponential backoff (1s, 2s, 4s)
3. Sends to DLT after all retries exhausted
4. Logs each retry attempt with the attempt number
5. Is idempotent using a Redis-based IdempotentRepository
```

### Example 3: Refactor to Hexagonal Architecture

Select a service class:
```
/transform Refactor this OrderService into hexagonal architecture:
1. Extract a port interface (OrderPort)
2. Create an adapter implementation
3. Separate domain logic from infrastructure
4. Keep the Spring annotations only in the adapter layer
```

### Example 4: Generate Dockerfile

```
Generate a production Dockerfile for this Spring Boot 3.2 app with:
1. Multi-stage build (Maven build + JRE runtime)
2. Use Eclipse Temurin 21 JRE Alpine for runtime
3. Non-root user
4. Health check using actuator
5. JVM flags for containers: -XX:MaxRAMPercentage=75 -XX:+UseZGC
6. Layer extraction for better caching
```

### Example 5: Debug a Failing Integration Test

Paste the test, the error output, and the relevant service code:
```
This integration test fails intermittently. The test uses @SpringBootTest with
TestContainers for MySQL and embedded Kafka. Error: "Connection refused"
appears randomly. Here's the test class and docker-compose-test.yml:
[paste code]
What's causing the intermittent failure and how do I fix it?
```

---

## 10. Comparison: Gemini Code Assist vs Copilot vs Claude Code

### Feature Comparison

| Feature | Gemini Code Assist | GitHub Copilot | Claude Code |
|---------|-------------------|----------------|-------------|
| **Type** | IDE extension + chat | IDE extension + chat | CLI agent |
| **Inline completions** | Yes | Yes | No (not an IDE plugin) |
| **Chat in IDE** | Yes | Yes (Copilot Chat) | Terminal-based |
| **Agentic mode** | Limited | Copilot Workspace | Yes (primary mode) |
| **Multi-file editing** | No | Copilot Edits (preview) | Yes (core feature) |
| **Codebase-aware** | Enterprise tier | Copilot with @workspace | Yes (reads files on demand) |
| **Terminal integration** | No | Copilot in CLI | Yes (runs commands) |
| **Code execution** | No | No | Yes (runs tests, builds) |
| **Git integration** | No | PR summaries, reviews | Yes (commits, branches) |
| **Model** | Gemini 2.5 Flash/Pro | GPT-4.1 / Claude 3.5 | Claude Opus 4 / Sonnet 4 |
| **Free tier** | Yes (individual) | No ($10+/mo or free for students) | Free tier available |
| **Enterprise** | Yes ($45/user/mo) | Yes ($39/user/mo) | Yes (via Anthropic API) |
| **GCP integration** | Deep | None | None |
| **Azure integration** | None | Via Azure OpenAI | None |

### When to Use Each

| Scenario | Best Tool | Why |
|----------|-----------|-----|
| Writing code in IntelliJ | Gemini Code Assist or Copilot | Best inline completions |
| Large refactoring across 10+ files | Claude Code | Agentic multi-file editing |
| Quick code question while coding | Gemini Chat / Copilot Chat | In-IDE, fast |
| Debugging production issue | Claude Code | Can read logs, run commands |
| Writing Terraform for GCP | Gemini Code Assist | GCP-aware completions |
| Writing Terraform for Azure | Copilot | Azure-aware via context |
| Architecture review | Claude Code | Can analyze entire codebase |
| Generate unit tests | All three | All good; Claude Code can also run them |
| CI/CD pipeline work | Claude Code | Can edit YAML and run pipeline |
| GKE troubleshooting | Gemini (in Cloud Console) | Native GCP integration |
| Learning a new framework | Claude Code or Gemini | Both excellent for explanation |

### Recommendation: Use All Three

```
┌──────────────────┐
│  Daily Workflow   │
├──────────────────┤
│                  │
│  IDE (IntelliJ)  │──> Gemini Code Assist (completions + chat)
│                  │    OR Copilot (completions + chat)
│                  │
│  Terminal         │──> Claude Code (multi-file edits, debugging, git)
│                  │
│  Browser          │──> Google AI Studio (prompt prototyping)
│                  │    Gemini Advanced (research, docs)
│                  │
│  GCP Console      │──> Gemini Cloud Assist (infra troubleshooting)
│                  │
└──────────────────┘
```

> **Practical advice:** Install both Gemini Code Assist and Copilot in your IDE.
> Use whichever gives better suggestions for the current task. Use Claude Code
> from the terminal for complex, multi-step operations.

---

## 11. Try This Exercises

### Exercise 1: Install and Configure (15 min)
1. Install "Gemini Code Assist" in IntelliJ IDEA (or VS Code)
2. Sign in with your Google account
3. Open a Java file and verify inline suggestions work
4. Open the Gemini chat panel and ask: "What does this class do?"

### Exercise 2: Code Generation Challenge (20 min)
1. Create a new Spring Boot class (empty)
2. Write only the class-level comment:
   ```java
   /**
    * Service that manages user sessions. Stores sessions in Redis with
    * configurable TTL. Supports session refresh, invalidation, and
    * listing active sessions for a user. Publishes session events to Kafka.
    */
   ```
3. Let Gemini generate the entire class from the comment
4. Compare the output with what you would write manually
5. Rate: What did it get right? What would you change?

### Exercise 3: Debug a Real Issue (15 min)
1. Find a recent bug or error in one of your projects
2. Paste the error and relevant code into the Gemini chat panel
3. Evaluate: Did it identify the root cause? Was the suggested fix correct?
4. Compare: Ask the same question in Claude Code — which gave a better answer?

### Exercise 4: Test Generation (15 min)
1. Select a service class with business logic
2. In chat: `/tests Write comprehensive unit tests for this class with JUnit 5 and Mockito`
3. Create the test file with the generated code
4. Run the tests — do they pass? What's missing?

### Exercise 5: Side-by-Side Comparison (20 min)
1. Pick a moderately complex coding task (e.g., "write a rate limiter using Redis")
2. Ask Gemini Code Assist, GitHub Copilot (if available), and Claude Code
3. Compare: speed, code quality, explanation quality, correctness
4. Document your findings for future reference

---

## Key Takeaways

1. **Gemini Code Assist** is a capable IDE assistant — on par with Copilot for completions
2. The **free tier** is generous enough for individual developers
3. **Chat panel** is useful for quick questions without leaving your IDE
4. For Java/Spring Boot, it generates high-quality boilerplate and configuration code
5. **Enterprise tier** adds codebase-aware completions — valuable for large teams
6. Use Gemini Code Assist for **inline completions** and Claude Code for **multi-file agent tasks**
7. The best setup is **multiple tools for different scenarios** — not choosing just one

---

Next: [03 — Gemini API and SDK](03-gemini-api-and-sdk.md)
