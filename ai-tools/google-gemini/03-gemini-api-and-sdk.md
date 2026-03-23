# 03 — Gemini API and SDK

## Table of Contents

1. [SDK Overview](#1-sdk-overview)
2. [Gemini API Fundamentals](#2-gemini-api-fundamentals)
3. [Java SDK for Gemini](#3-java-sdk-for-gemini)
4. [Spring AI Integration with Gemini](#4-spring-ai-integration-with-gemini)
5. [Function Calling / Tool Use](#5-function-calling--tool-use)
6. [Structured Output (JSON Mode)](#6-structured-output-json-mode)
7. [Grounding with Google Search](#7-grounding-with-google-search)
8. [Context Caching](#8-context-caching)
9. [Multimodal Inputs](#9-multimodal-inputs)
10. [Error Handling, Retries, and Rate Limits](#10-error-handling-retries-and-rate-limits)
11. [Practical Project: Spring Boot Gemini Service](#11-practical-project-spring-boot-gemini-service)
12. [Try This Exercises](#12-try-this-exercises)

---

## 1. SDK Overview

Google provides official SDKs for accessing Gemini models:

| Language | Package | API Target |
|----------|---------|------------|
| Python | `google-genai` | Google AI (free tier) or Vertex AI |
| JavaScript/TypeScript | `@google/genai` | Google AI or Vertex AI |
| Java/Kotlin | `google-cloud-vertexai` | Vertex AI |
| Go | `google.golang.org/genai` | Google AI or Vertex AI |
| Swift | `google-generative-ai-swift` | Google AI |
| Dart/Flutter | `google_generative_ai` | Google AI |
| REST | cURL / any HTTP client | Both |

### Two API Endpoints

| Endpoint | Base URL | Auth | Best For |
|----------|----------|------|----------|
| **Google AI** (Gemini API) | `https://generativelanguage.googleapis.com/v1beta/` | API key | Free tier, prototyping, personal projects |
| **Vertex AI** | `https://{region}-aiplatform.googleapis.com/` | OAuth / Service Account | Production, enterprise, VPC, compliance |

> **For Java developers:** The official Java SDK (`google-cloud-vertexai`) targets Vertex AI.
> For the free Google AI endpoint, use the REST API directly or Spring AI.

---

## 2. Gemini API Fundamentals

### Core Operations

| Operation | Endpoint | Use Case |
|-----------|----------|----------|
| `generateContent` | POST `/models/{model}:generateContent` | Single request/response |
| `streamGenerateContent` | POST `/models/{model}:streamGenerateContent` | Streaming response |
| `countTokens` | POST `/models/{model}:countTokens` | Count tokens before sending |
| `embedContent` | POST `/models/{model}:embedContent` | Generate embeddings |
| `batchEmbedContents` | POST `/models/{model}:batchEmbedContents` | Bulk embeddings |

### Request Structure

```json
{
  "contents": [
    {
      "role": "user",
      "parts": [
        { "text": "Explain the CAP theorem for a senior backend developer" }
      ]
    }
  ],
  "systemInstruction": {
    "parts": [
      { "text": "You are a distributed systems expert. Be concise." }
    ]
  },
  "generationConfig": {
    "temperature": 0.7,
    "topP": 0.95,
    "topK": 40,
    "maxOutputTokens": 8192,
    "responseMimeType": "text/plain"
  },
  "safetySettings": [
    {
      "category": "HARM_CATEGORY_DANGEROUS_CONTENT",
      "threshold": "BLOCK_ONLY_HIGH"
    }
  ]
}
```

### Response Structure

```json
{
  "candidates": [
    {
      "content": {
        "parts": [
          { "text": "The CAP theorem states that..." }
        ],
        "role": "model"
      },
      "finishReason": "STOP",
      "safetyRatings": [...]
    }
  ],
  "usageMetadata": {
    "promptTokenCount": 25,
    "candidatesTokenCount": 150,
    "totalTokenCount": 175
  }
}
```

### Multi-Turn Chat

```json
{
  "contents": [
    { "role": "user", "parts": [{ "text": "What is CQRS?" }] },
    { "role": "model", "parts": [{ "text": "CQRS stands for Command Query..." }] },
    { "role": "user", "parts": [{ "text": "How does it relate to event sourcing?" }] }
  ]
}
```

### cURL Example

```bash
# Simple text generation
curl -s "https://generativelanguage.googleapis.com/v1beta/models/gemini-2.5-flash:generateContent?key=${GEMINI_API_KEY}" \
  -H "Content-Type: application/json" \
  -d '{
    "contents": [{
      "parts": [{"text": "Write a Java record for a Kafka message header with timestamp, correlationId, and source service name"}]
    }]
  }' | jq '.candidates[0].content.parts[0].text'
```

---

## 3. Java SDK for Gemini

### Option A: Vertex AI Java SDK (Official)

**Maven dependency:**
```xml
<dependency>
  <groupId>com.google.cloud</groupId>
  <artifactId>google-cloud-vertexai</artifactId>
  <version>1.15.0</version>
</dependency>
```

**Basic usage:**
```java
import com.google.cloud.vertexai.VertexAI;
import com.google.cloud.vertexai.api.GenerateContentResponse;
import com.google.cloud.vertexai.generativemodel.GenerativeModel;
import com.google.cloud.vertexai.generativemodel.ResponseStream;

public class GeminiExample {

    public static void main(String[] args) throws Exception {
        // Requires GOOGLE_APPLICATION_CREDENTIALS or gcloud auth
        try (VertexAI vertexAI = new VertexAI("your-project-id", "us-central1")) {

            GenerativeModel model = new GenerativeModel
                .Builder()
                .setModelName("gemini-2.5-flash")
                .setVertexAi(vertexAI)
                .build();

            // Simple text generation
            GenerateContentResponse response = model.generateContent(
                "Explain the Circuit Breaker pattern in microservices"
            );
            System.out.println(response.getCandidates(0)
                .getContent().getParts(0).getText());
        }
    }
}
```

**Streaming:**
```java
try (VertexAI vertexAI = new VertexAI("your-project-id", "us-central1")) {
    GenerativeModel model = new GenerativeModel
        .Builder()
        .setModelName("gemini-2.5-flash")
        .setVertexAi(vertexAI)
        .build();

    ResponseStream<GenerateContentResponse> stream =
        model.generateContentStream("Write a Spring Boot health check endpoint");

    stream.stream().forEach(chunk -> {
        String text = chunk.getCandidates(0)
            .getContent().getParts(0).getText();
        System.out.print(text); // Print as chunks arrive
    });
}
```

**System instructions and generation config:**
```java
import com.google.cloud.vertexai.api.GenerationConfig;
import com.google.cloud.vertexai.generativemodel.ContentMaker;

GenerationConfig config = GenerationConfig.newBuilder()
    .setTemperature(0.3f)
    .setMaxOutputTokens(4096)
    .setTopP(0.95f)
    .build();

GenerativeModel model = new GenerativeModel.Builder()
    .setModelName("gemini-2.5-pro")
    .setVertexAi(vertexAI)
    .setGenerationConfig(config)
    .setSystemInstruction(ContentMaker.fromString(
        "You are a senior Java architect. Provide production-ready code with error handling."))
    .build();
```

### Option B: REST API with Spring WebClient (Google AI Endpoint)

If you prefer the free Google AI endpoint with an API key:

```java
@Service
public class GeminiRestClient {

    private final WebClient webClient;

    @Value("${gemini.api-key}")
    private String apiKey;

    public GeminiRestClient(WebClient.Builder webClientBuilder) {
        this.webClient = webClientBuilder
            .baseUrl("https://generativelanguage.googleapis.com/v1beta")
            .build();
    }

    public String generate(String prompt) {
        Map<String, Object> request = Map.of(
            "contents", List.of(
                Map.of("parts", List.of(Map.of("text", prompt)))
            ),
            "generationConfig", Map.of(
                "temperature", 0.7,
                "maxOutputTokens", 4096
            )
        );

        Map<String, Object> response = webClient.post()
            .uri("/models/gemini-2.5-flash:generateContent?key={key}", apiKey)
            .bodyValue(request)
            .retrieve()
            .bodyToMono(new ParameterizedTypeReference<Map<String, Object>>() {})
            .block();

        // Extract text from response
        List<Map<String, Object>> candidates =
            (List<Map<String, Object>>) response.get("candidates");
        Map<String, Object> content =
            (Map<String, Object>) candidates.get(0).get("content");
        List<Map<String, Object>> parts =
            (List<Map<String, Object>>) content.get("parts");
        return (String) parts.get(0).get("text");
    }

    // Streaming version
    public Flux<String> generateStream(String prompt) {
        Map<String, Object> request = Map.of(
            "contents", List.of(
                Map.of("parts", List.of(Map.of("text", prompt)))
            )
        );

        return webClient.post()
            .uri("/models/gemini-2.5-flash:streamGenerateContent?alt=sse&key={key}", apiKey)
            .bodyValue(request)
            .retrieve()
            .bodyToFlux(String.class)
            .filter(chunk -> !chunk.isEmpty())
            .map(this::extractTextFromChunk);
    }
}
```

---

## 4. Spring AI Integration with Gemini

Spring AI provides a clean abstraction over multiple LLM providers, including
Google Vertex AI / Gemini.

### Setup

**Maven dependencies:**
```xml
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-vertex-ai-gemini-spring-boot-starter</artifactId>
</dependency>

<!-- BOM -->
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.springframework.ai</groupId>
            <artifactId>spring-ai-bom</artifactId>
            <version>1.0.0</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>

<!-- Spring AI milestone repo (if needed) -->
<repositories>
    <repository>
        <id>spring-milestones</id>
        <url>https://repo.spring.io/milestone</url>
    </repository>
</repositories>
```

**application.yml:**
```yaml
spring:
  ai:
    vertex-ai:
      gemini:
        project-id: ${GCP_PROJECT_ID}
        location: us-central1
        chat:
          options:
            model: gemini-2.5-flash
            temperature: 0.7
            max-output-tokens: 4096
```

### Basic Chat with Spring AI

```java
import org.springframework.ai.chat.client.ChatClient;
import org.springframework.ai.chat.prompt.Prompt;
import org.springframework.ai.chat.messages.SystemMessage;
import org.springframework.ai.chat.messages.UserMessage;

@Service
public class GeminiChatService {

    private final ChatClient chatClient;

    public GeminiChatService(ChatClient.Builder chatClientBuilder) {
        this.chatClient = chatClientBuilder
            .defaultSystem("You are a senior Java architect. Be concise and practical.")
            .build();
    }

    // Simple call
    public String ask(String question) {
        return chatClient.prompt()
            .user(question)
            .call()
            .content();
    }

    // With parameters
    public String askWithContext(String question, String codeContext) {
        return chatClient.prompt()
            .system("You are reviewing Java code. Point out bugs and improvements.")
            .user(u -> u.text("""
                Context code:
                ```java
                {code}
                ```

                Question: {question}
                """)
                .param("code", codeContext)
                .param("question", question))
            .call()
            .content();
    }

    // Streaming
    public Flux<String> askStream(String question) {
        return chatClient.prompt()
            .user(question)
            .stream()
            .content();
    }
}
```

### REST Controller

```java
@RestController
@RequestMapping("/api/ai")
public class GeminiController {

    private final GeminiChatService chatService;

    public GeminiController(GeminiChatService chatService) {
        this.chatService = chatService;
    }

    @PostMapping("/chat")
    public ResponseEntity<Map<String, String>> chat(@RequestBody Map<String, String> request) {
        String answer = chatService.ask(request.get("question"));
        return ResponseEntity.ok(Map.of("answer", answer));
    }

    @GetMapping(value = "/chat/stream", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
    public Flux<String> chatStream(@RequestParam String question) {
        return chatService.askStream(question);
    }
}
```

### Switching Between Providers

One of Spring AI's biggest advantages — switch from Gemini to Claude to GPT by
changing configuration, not code:

```yaml
# Switch to Anthropic Claude
spring:
  ai:
    anthropic:
      api-key: ${ANTHROPIC_API_KEY}
      chat:
        options:
          model: claude-sonnet-4-20250514
          temperature: 0.7
          max-tokens: 4096

# Switch to OpenAI GPT
spring:
  ai:
    openai:
      api-key: ${OPENAI_API_KEY}
      chat:
        options:
          model: gpt-4.1
          temperature: 0.7
```

> The `ChatClient` interface stays the same. Only the starter dependency and config change.

---

## 5. Function Calling / Tool Use

Gemini supports function calling — the model can invoke your Java methods.

### How It Works

```
User prompt ──> Gemini ──> "I need to call getOrderStatus(orderId=123)"
                              │
                              v
                   Your Java code executes getOrderStatus(123)
                              │
                              v
                   Result sent back to Gemini
                              │
                              v
                   Gemini generates final response using the result
```

### Spring AI Tool Use with Gemini

```java
// Define a tool
import org.springframework.ai.tool.annotation.Tool;
import org.springframework.ai.tool.annotation.ToolParam;
import org.springframework.stereotype.Component;

@Component
public class OrderTools {

    private final OrderRepository orderRepository;

    public OrderTools(OrderRepository orderRepository) {
        this.orderRepository = orderRepository;
    }

    @Tool(description = "Look up the current status of an order by its ID. " +
                         "Returns order status, last updated timestamp, and tracking URL.")
    public OrderStatusDto getOrderStatus(
            @ToolParam(description = "The numeric order ID") Long orderId) {

        Order order = orderRepository.findById(orderId)
            .orElseThrow(() -> new OrderNotFoundException(orderId));

        return new OrderStatusDto(
            order.getId(),
            order.getStatus().name(),
            order.getUpdatedAt(),
            order.getTrackingUrl()
        );
    }

    @Tool(description = "Search orders by customer email. Returns up to 10 recent orders.")
    public List<OrderSummaryDto> searchOrders(
            @ToolParam(description = "Customer email address") String email,
            @ToolParam(description = "Maximum number of results (default 10)") int limit) {

        return orderRepository.findByCustomerEmail(email, PageRequest.of(0, limit))
            .stream()
            .map(this::toSummaryDto)
            .toList();
    }
}
```

```java
// Use tools in ChatClient
@Service
public class OrderAssistantService {

    private final ChatClient chatClient;
    private final OrderTools orderTools;

    public OrderAssistantService(ChatClient.Builder builder, OrderTools orderTools) {
        this.orderTools = orderTools;
        this.chatClient = builder
            .defaultSystem("You are an order support assistant. Use the available tools to look up order information.")
            .build();
    }

    public String handleQuery(String userMessage) {
        return chatClient.prompt()
            .user(userMessage)
            .tools(orderTools)
            .call()
            .content();
    }
}
```

### Using Vertex AI SDK Directly for Function Calling

```java
import com.google.cloud.vertexai.api.*;
import com.google.cloud.vertexai.generativemodel.*;

// Define function declarations
FunctionDeclaration getWeatherFunc = FunctionDeclaration.newBuilder()
    .setName("getWeather")
    .setDescription("Get current weather for a city")
    .setParameters(Schema.newBuilder()
        .setType(Type.OBJECT)
        .putProperties("city", Schema.newBuilder()
            .setType(Type.STRING)
            .setDescription("City name")
            .build())
        .addRequired("city")
        .build())
    .build();

Tool tool = Tool.newBuilder()
    .addFunctionDeclarations(getWeatherFunc)
    .build();

GenerativeModel model = new GenerativeModel.Builder()
    .setModelName("gemini-2.5-flash")
    .setVertexAi(vertexAI)
    .setTools(List.of(tool))
    .build();

// Send message
GenerateContentResponse response = model.generateContent("What's the weather in Tokyo?");

// Check if model wants to call a function
Part responsePart = response.getCandidates(0).getContent().getParts(0);
if (responsePart.hasFunctionCall()) {
    FunctionCall functionCall = responsePart.getFunctionCall();
    String functionName = functionCall.getName(); // "getWeather"
    String city = functionCall.getArgs().getFieldsMap().get("city").getStringValue(); // "Tokyo"

    // Execute your function
    String weatherResult = getWeather(city);

    // Send result back to Gemini
    Content functionResponse = ContentMaker.fromMultiModalData(
        Part.newBuilder().setFunctionResponse(
            FunctionResponse.newBuilder()
                .setName(functionName)
                .setResponse(Struct.newBuilder()
                    .putFields("weather", Value.newBuilder()
                        .setStringValue(weatherResult).build())
                    .build())
                .build()
        ).build()
    );

    GenerateContentResponse finalResponse = model.generateContent(
        List.of(userContent, modelResponseContent, functionResponse)
    );
}
```

---

## 6. Structured Output (JSON Mode)

### Method 1: Response MIME Type

```json
{
  "contents": [...],
  "generationConfig": {
    "responseMimeType": "application/json"
  }
}
```

### Method 2: JSON Schema (Recommended)

```json
{
  "contents": [...],
  "generationConfig": {
    "responseMimeType": "application/json",
    "responseSchema": {
      "type": "object",
      "properties": {
        "severity": {
          "type": "string",
          "enum": ["LOW", "MEDIUM", "HIGH", "CRITICAL"]
        },
        "summary": {
          "type": "string",
          "description": "One-line summary of the issue"
        },
        "affectedFiles": {
          "type": "array",
          "items": { "type": "string" }
        },
        "suggestedFix": {
          "type": "string"
        }
      },
      "required": ["severity", "summary", "affectedFiles", "suggestedFix"]
    }
  }
}
```

### Spring AI Structured Output with Gemini

```java
// Define your output record
public record CodeReviewResult(
    String severity,
    String summary,
    List<String> affectedFiles,
    String suggestedFix,
    List<String> relatedPatterns
) {}

// Use BeanOutputConverter
@Service
public class CodeReviewService {

    private final ChatClient chatClient;

    public CodeReviewService(ChatClient.Builder builder) {
        this.chatClient = builder.build();
    }

    public CodeReviewResult reviewCode(String code) {
        return chatClient.prompt()
            .system("You are a code reviewer. Analyze the code and return structured results.")
            .user("Review this code:\n" + code)
            .call()
            .entity(CodeReviewResult.class);  // Automatic JSON parsing
    }

    // List output
    public List<CodeReviewResult> reviewMultipleFiles(List<String> files) {
        String allFiles = files.stream()
            .map(f -> "```\n" + f + "\n```")
            .collect(Collectors.joining("\n---\n"));

        return chatClient.prompt()
            .user("Review each file:\n" + allFiles)
            .call()
            .entity(new ParameterizedTypeReference<List<CodeReviewResult>>() {});
    }
}
```

---

## 7. Grounding with Google Search

Grounding lets Gemini access real-time information from Google Search.

### Via REST API

```json
{
  "contents": [{
    "parts": [{"text": "What are the latest Spring Boot 3.x security vulnerabilities?"}]
  }],
  "tools": [{
    "googleSearch": {}
  }]
}
```

The response includes `groundingMetadata` with search results and citations:

```json
{
  "candidates": [{
    "content": { "parts": [{ "text": "Recent Spring Boot vulnerabilities include..." }] },
    "groundingMetadata": {
      "searchEntryPoint": { "renderedContent": "..." },
      "groundingChunks": [
        { "web": { "uri": "https://spring.io/security/...", "title": "..." } }
      ],
      "groundingSupports": [
        {
          "segment": { "startIndex": 0, "endIndex": 150 },
          "groundingChunkIndices": [0],
          "confidenceScores": [0.95]
        }
      ]
    }
  }]
}
```

### Use Cases for Grounding

| Use Case | Why Grounding Helps |
|----------|-------------------|
| Check latest library versions | Model's training data is outdated |
| Find recent CVEs | Security info changes daily |
| Current best practices | Frameworks evolve rapidly |
| Verify deprecated APIs | Models may suggest removed methods |
| Check cloud service availability | Regions and features change |

> **Note:** Grounding costs extra (~$35 per 1,000 grounded requests). Use selectively.

---

## 8. Context Caching

Context caching lets you store large contexts (codebases, documentation) and query
them repeatedly at reduced cost.

### How It Works

```
Step 1: Create cache (pay full input price for cached content)
         Cache large context ──> Cache ID returned
         TTL: 1 hour (default), configurable

Step 2: Use cache (pay ~25% of normal input price for cached portion)
         Send cache ID + new query ──> Response
         Only the new query tokens are charged at full price
```

### REST API

```bash
# Create a cache
curl -X POST \
  "https://generativelanguage.googleapis.com/v1beta/cachedContents?key=${GEMINI_API_KEY}" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "models/gemini-2.5-flash",
    "displayName": "my-spring-boot-codebase",
    "contents": [{
      "parts": [{"text": "'"$(cat all-source-files.txt)"'"}],
      "role": "user"
    }],
    "systemInstruction": {
      "parts": [{"text": "You are a code reviewer for this Spring Boot project."}]
    },
    "ttl": "3600s"
  }'

# Response includes cache name: "cachedContents/abc123"

# Query against the cache
curl -X POST \
  "https://generativelanguage.googleapis.com/v1beta/models/gemini-2.5-flash:generateContent?key=${GEMINI_API_KEY}" \
  -H "Content-Type: application/json" \
  -d '{
    "cachedContent": "cachedContents/abc123",
    "contents": [{
      "parts": [{"text": "Find all potential SQL injection vulnerabilities in this codebase"}],
      "role": "user"
    }]
  }'
```

### Java Implementation

```java
@Service
public class CachedCodeAnalysisService {

    private final WebClient webClient;
    private final String apiKey;
    private String cachedContentName;

    // Cache the codebase
    public void cacheCodebase(String allSourceCode) {
        Map<String, Object> request = Map.of(
            "model", "models/gemini-2.5-flash",
            "displayName", "project-codebase",
            "contents", List.of(Map.of(
                "parts", List.of(Map.of("text", allSourceCode)),
                "role", "user"
            )),
            "systemInstruction", Map.of(
                "parts", List.of(Map.of("text",
                    "You are reviewing a Spring Boot microservice. " +
                    "The full source code is provided as context."))
            ),
            "ttl", "3600s"
        );

        Map<String, Object> response = webClient.post()
            .uri("/cachedContents?key={key}", apiKey)
            .bodyValue(request)
            .retrieve()
            .bodyToMono(new ParameterizedTypeReference<Map<String, Object>>() {})
            .block();

        this.cachedContentName = (String) response.get("name");
    }

    // Query against cached codebase (75% cheaper on cached tokens)
    public String analyzeCode(String question) {
        Map<String, Object> request = Map.of(
            "cachedContent", cachedContentName,
            "contents", List.of(Map.of(
                "parts", List.of(Map.of("text", question)),
                "role", "user"
            ))
        );

        // ... send request and extract response
    }
}
```

### When to Use Context Caching

| Scenario | Cache? | Why |
|----------|--------|-----|
| Analyzing a codebase with multiple questions | Yes | Pay once for 200K+ tokens |
| Single one-off question | No | Cache creation cost > savings |
| Daily standup summary (same format) | Yes | Same system prompt + template daily |
| Long documentation reference | Yes | Upload once, query many times |
| Interactive chat (small context) | No | Cache overhead not worth it |

> **Break-even:** Caching is cheaper when you make ~4+ queries against the same context.

---

## 9. Multimodal Inputs

### Sending Images

```java
// Vertex AI SDK
import com.google.cloud.vertexai.generativemodel.PartMaker;

byte[] imageBytes = Files.readAllBytes(Path.of("architecture-diagram.png"));
Content content = ContentMaker.fromMultiModalData(
    PartMaker.fromMimeTypeAndData("image/png", imageBytes),
    "Analyze this architecture diagram. Identify single points of failure."
);

GenerateContentResponse response = model.generateContent(content);
```

### Sending PDFs

```java
// PDF as inline data
byte[] pdfBytes = Files.readAllBytes(Path.of("api-spec.pdf"));
Content content = ContentMaker.fromMultiModalData(
    PartMaker.fromMimeTypeAndData("application/pdf", pdfBytes),
    "Generate Spring Boot controller stubs from this API specification."
);
```

### Sending Video (via URI)

For large files, upload to Google Cloud Storage first:

```java
// Video from GCS
Content content = ContentMaker.fromMultiModalData(
    Part.newBuilder().setFileData(FileData.newBuilder()
        .setMimeType("video/mp4")
        .setFileUri("gs://my-bucket/code-walkthrough.mp4")
        .build()
    ).build(),
    "Watch this code walkthrough and identify any bugs or improvements."
);
```

### REST API with Base64 Image

```bash
# Encode image to base64
IMAGE_BASE64=$(base64 -i screenshot.png)

curl -X POST \
  "https://generativelanguage.googleapis.com/v1beta/models/gemini-2.5-flash:generateContent?key=${GEMINI_API_KEY}" \
  -H "Content-Type: application/json" \
  -d '{
    "contents": [{
      "parts": [
        {"text": "What error is shown in this screenshot? How do I fix it?"},
        {
          "inlineData": {
            "mimeType": "image/png",
            "data": "'"${IMAGE_BASE64}"'"
          }
        }
      ]
    }]
  }'
```

---

## 10. Error Handling, Retries, and Rate Limits

### Common Error Codes

| HTTP Code | Error | Cause | Action |
|-----------|-------|-------|--------|
| 400 | INVALID_ARGUMENT | Bad request format | Fix request |
| 403 | PERMISSION_DENIED | Invalid API key or no access | Check credentials |
| 429 | RESOURCE_EXHAUSTED | Rate limit hit | Retry with backoff |
| 500 | INTERNAL | Server error | Retry |
| 503 | UNAVAILABLE | Service overloaded | Retry with backoff |

### Rate Limits (Paid Tier)

| Model | RPM | TPM |
|-------|-----|-----|
| Gemini 2.5 Pro | 1,000 | 4M |
| Gemini 2.5 Flash | 2,000 | 4M |
| Gemini 2.0 Flash | 2,000 | 4M |

### Resilience4j Retry Configuration

```java
@Configuration
public class GeminiResilienceConfig {

    @Bean
    public RetryRegistry retryRegistry() {
        RetryConfig config = RetryConfig.custom()
            .maxAttempts(3)
            .waitDuration(Duration.ofMillis(500))
            .intervalFunction(IntervalFunction.ofExponentialBackoff(500, 2.0))
            .retryOnResult(response -> response == null)
            .retryExceptions(
                WebClientResponseException.TooManyRequests.class,
                WebClientResponseException.InternalServerError.class,
                WebClientResponseException.ServiceUnavailable.class
            )
            .ignoreExceptions(
                WebClientResponseException.BadRequest.class,
                WebClientResponseException.Forbidden.class
            )
            .build();

        return RetryRegistry.of(config);
    }
}

@Service
public class ResilientGeminiClient {

    private final GeminiRestClient geminiClient;
    private final Retry retry;

    public ResilientGeminiClient(GeminiRestClient geminiClient, RetryRegistry registry) {
        this.geminiClient = geminiClient;
        this.retry = registry.retry("gemini");
    }

    public String generate(String prompt) {
        return Retry.decorateSupplier(retry, () -> geminiClient.generate(prompt)).get();
    }
}
```

### Rate Limiter with Resilience4j

```java
@Bean
public RateLimiter geminiRateLimiter() {
    RateLimiterConfig config = RateLimiterConfig.custom()
        .limitForPeriod(50)               // 50 requests
        .limitRefreshPeriod(Duration.ofMinutes(1))  // per minute
        .timeoutDuration(Duration.ofSeconds(10))    // wait up to 10s
        .build();
    return RateLimiter.of("gemini", config);
}
```

---

## 11. Practical Project: Spring Boot Gemini Service

Build a complete code review assistant microservice.

### Project Structure

```
gemini-review-service/
├── src/main/java/com/example/review/
│   ├── GeminiReviewApplication.java
│   ├── config/
│   │   ├── GeminiConfig.java
│   │   └── ResilienceConfig.java
│   ├── controller/
│   │   └── ReviewController.java
│   ├── service/
│   │   ├── CodeReviewService.java
│   │   └── GeminiClient.java
│   ├── model/
│   │   ├── ReviewRequest.java
│   │   ├── ReviewResponse.java
│   │   └── ReviewIssue.java
│   └── tool/
│       └── GitHubTools.java
├── src/main/resources/
│   └── application.yml
└── pom.xml
```

### Key Files

**application.yml:**
```yaml
spring:
  ai:
    vertex-ai:
      gemini:
        project-id: ${GCP_PROJECT_ID}
        location: us-central1
        chat:
          options:
            model: gemini-2.5-flash
            temperature: 0.3
            max-output-tokens: 8192

gemini:
  review:
    system-prompt: |
      You are a senior Java code reviewer. Analyze code for:
      1. Bugs and logic errors
      2. Security vulnerabilities (OWASP Top 10)
      3. Performance issues
      4. Spring Boot best practices
      5. Clean code violations
      Return results as structured JSON.
    max-file-size-kb: 500
```

**ReviewController.java:**
```java
@RestController
@RequestMapping("/api/reviews")
@Validated
public class ReviewController {

    private final CodeReviewService reviewService;

    public ReviewController(CodeReviewService reviewService) {
        this.reviewService = reviewService;
    }

    @PostMapping
    public ResponseEntity<ReviewResponse> reviewCode(
            @Valid @RequestBody ReviewRequest request) {
        ReviewResponse response = reviewService.review(request);
        return ResponseEntity.ok(response);
    }

    @PostMapping(value = "/stream", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
    public Flux<String> reviewCodeStream(@Valid @RequestBody ReviewRequest request) {
        return reviewService.reviewStream(request);
    }

    @PostMapping("/github")
    public ResponseEntity<ReviewResponse> reviewGitHubPR(
            @RequestParam String repoOwner,
            @RequestParam String repoName,
            @RequestParam int prNumber) {
        ReviewResponse response = reviewService.reviewGitHubPR(repoOwner, repoName, prNumber);
        return ResponseEntity.ok(response);
    }
}
```

**CodeReviewService.java:**
```java
@Service
public class CodeReviewService {

    private final ChatClient chatClient;
    private final String systemPrompt;

    public CodeReviewService(
            ChatClient.Builder builder,
            @Value("${gemini.review.system-prompt}") String systemPrompt,
            GitHubTools gitHubTools) {
        this.systemPrompt = systemPrompt;
        this.chatClient = builder
            .defaultSystem(systemPrompt)
            .defaultTools(gitHubTools)
            .build();
    }

    public ReviewResponse review(ReviewRequest request) {
        return chatClient.prompt()
            .user(u -> u.text("""
                Review the following {language} code:

                File: {filename}
                ```{language}
                {code}
                ```

                Focus on: {focusAreas}
                """)
                .param("language", request.language())
                .param("filename", request.filename())
                .param("code", request.code())
                .param("focusAreas", String.join(", ", request.focusAreas())))
            .call()
            .entity(ReviewResponse.class);
    }

    public Flux<String> reviewStream(ReviewRequest request) {
        return chatClient.prompt()
            .user("Review this code:\n```\n" + request.code() + "\n```")
            .stream()
            .content();
    }

    public ReviewResponse reviewGitHubPR(String owner, String repo, int prNumber) {
        String userMessage = String.format(
            "Fetch and review the pull request #%d from %s/%s. " +
            "Look at all changed files and provide a comprehensive review.",
            prNumber, owner, repo
        );

        return chatClient.prompt()
            .user(userMessage)
            .call()
            .entity(ReviewResponse.class);
    }
}
```

**Model classes:**
```java
public record ReviewRequest(
    @NotBlank String code,
    @NotBlank String language,
    String filename,
    List<String> focusAreas
) {
    public ReviewRequest {
        if (focusAreas == null) focusAreas = List.of("bugs", "security", "performance");
    }
}

public record ReviewResponse(
    String summary,
    String overallSeverity,
    List<ReviewIssue> issues,
    List<String> positives,
    String refactoredCode
) {}

public record ReviewIssue(
    String severity,     // LOW, MEDIUM, HIGH, CRITICAL
    int lineNumber,
    String category,     // BUG, SECURITY, PERFORMANCE, STYLE
    String description,
    String suggestedFix
) {}
```

### Test It

```bash
# Start the service
mvn spring-boot:run

# Submit a review
curl -X POST http://localhost:8080/api/reviews \
  -H "Content-Type: application/json" \
  -d '{
    "code": "public void deleteUser(Long id) { userRepo.deleteById(id); }",
    "language": "java",
    "filename": "UserService.java",
    "focusAreas": ["security", "error-handling"]
  }'
```

---

## 12. Try This Exercises

### Exercise 1: Hello Gemini API (15 min)
1. Get an API key from https://aistudio.google.com/apikey
2. Make a cURL request to `gemini-2.5-flash` using the REST API
3. Ask it to generate a Spring Boot health check endpoint
4. Parse the JSON response — extract just the generated code

### Exercise 2: Spring AI Setup (20 min)
1. Create a new Spring Boot 3.2+ project
2. Add the `spring-ai-vertex-ai-gemini-spring-boot-starter` dependency
3. Configure your GCP project ID and location
4. Create a simple `/api/chat` endpoint
5. Test with a question about Spring Boot

### Exercise 3: Structured Output (20 min)
1. Define a `BugReport` record: `severity`, `description`, `file`, `lineNumber`, `fix`
2. Use Spring AI's `.entity(BugReport.class)` to get structured responses
3. Send a code snippet with an obvious bug
4. Verify the response deserializes correctly into your record

### Exercise 4: Function Calling (25 min)
1. Create a `@Tool` method that looks up data from your database
2. Wire it into the `ChatClient` with `.tools()`
3. Ask the model a question that requires calling your tool
4. Verify the model calls the tool, gets the result, and incorporates it

### Exercise 5: Context Caching (20 min)
1. Concatenate all Java source files from one of your projects into a single text file
2. Create a context cache using the REST API
3. Ask 5 different questions against the cached codebase
4. Compare costs: cached vs uncached (check `usageMetadata` in responses)

---

## Key Takeaways

1. **Spring AI** is the cleanest way to use Gemini from Java — same API for all providers
2. **Vertex AI SDK** is best for production GCP deployments with IAM and VPC
3. **REST API** with an API key is simplest for prototyping and personal projects
4. **Function calling** lets Gemini invoke your Java methods — powerful for assistants
5. **Structured output** with JSON schema ensures reliable parsing
6. **Context caching** saves ~75% on input costs for repeated queries
7. **Resilience4j** retry + rate limiter is essential for production API calls
8. Gemini API is cheaper than Claude/GPT at every tier, with competitive quality

---

Next: [04 — Google AI Studio and Vertex AI](04-google-ai-studio-and-vertex-ai.md)
