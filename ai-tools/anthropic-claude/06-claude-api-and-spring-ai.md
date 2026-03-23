# Tutorial 06: Claude API and Spring AI Integration

## Table of Contents

1. [Claude API Fundamentals](#1-claude-api-fundamentals)
2. [Anthropic SDK Usage](#2-anthropic-sdk-usage)
3. [Spring AI Integration with Claude](#3-spring-ai-integration-with-claude)
4. [Building a Chatbot with Spring Boot + Claude](#4-building-a-chatbot-with-spring-boot--claude)
5. [Tool Use / Function Calling](#5-tool-use--function-calling)
6. [Structured Outputs](#6-structured-outputs)
7. [Embeddings Strategy](#7-embeddings-strategy)
8. [Error Handling, Retries, and Rate Limits](#8-error-handling-retries-and-rate-limits)
9. [Cost Management](#9-cost-management)
10. [Practical Project: AI-Powered Order Assistant](#10-practical-project-ai-powered-order-assistant)
11. [Try This Exercises](#11-try-this-exercises)
12. [Tips and Gotchas](#12-tips-and-gotchas)

---

## 1. Claude API Fundamentals

### Messages API

The Messages API is Claude's primary interface. Every interaction follows this pattern:

```
Request:
  - model (required)
  - messages[] (required) — conversation history
  - system (optional) — system prompt
  - max_tokens (required) — maximum response length
  - temperature (optional) — randomness (0.0-1.0)
  - tools[] (optional) — function definitions

Response:
  - content[] — response blocks (text, tool_use)
  - usage — token counts
  - stop_reason — why generation stopped
```

### Basic API Call (curl)

```bash
curl https://api.anthropic.com/v1/messages \
  -H "content-type: application/json" \
  -H "x-api-key: $ANTHROPIC_API_KEY" \
  -H "anthropic-version: 2023-06-01" \
  -d '{
    "model": "claude-sonnet-4-0520",
    "max_tokens": 1024,
    "system": "You are a Java expert. Provide concise, production-ready code.",
    "messages": [
      {
        "role": "user",
        "content": "Write a thread-safe singleton pattern in Java 21."
      }
    ]
  }'
```

### Response Structure

```json
{
  "id": "msg_01XFDUDYJgAACzvnptvVoYEL",
  "type": "message",
  "role": "assistant",
  "content": [
    {
      "type": "text",
      "text": "Here is a thread-safe singleton using an enum..."
    }
  ],
  "model": "claude-sonnet-4-0520",
  "stop_reason": "end_turn",
  "usage": {
    "input_tokens": 35,
    "output_tokens": 150
  }
}
```

### Streaming

For real-time responses, use Server-Sent Events (SSE):

```bash
curl https://api.anthropic.com/v1/messages \
  -H "content-type: application/json" \
  -H "x-api-key: $ANTHROPIC_API_KEY" \
  -H "anthropic-version: 2023-06-01" \
  -d '{
    "model": "claude-sonnet-4-0520",
    "max_tokens": 1024,
    "stream": true,
    "messages": [
      {"role": "user", "content": "Explain the Java Memory Model"}
    ]
  }'
```

Streaming events arrive as:
```
event: message_start
event: content_block_start
event: content_block_delta   (repeated — each with a text chunk)
event: content_block_stop
event: message_delta
event: message_stop
```

---

## 2. Anthropic SDK Usage

### Java SDK (Official)

As of 2025, Anthropic provides an official Java SDK:

```xml
<!-- pom.xml -->
<dependency>
    <groupId>com.anthropic</groupId>
    <artifactId>anthropic-java</artifactId>
    <version>1.1.0</version>
</dependency>
```

```java
import com.anthropic.AnthropicClient;
import com.anthropic.models.*;

public class ClaudeExample {

    public static void main(String[] args) {
        var client = AnthropicClient.builder()
            .apiKey(System.getenv("ANTHROPIC_API_KEY"))
            .build();

        var response = client.messages().create(MessageCreateParams.builder()
            .model("claude-sonnet-4-0520")
            .maxTokens(1024)
            .system("You are a helpful Java assistant.")
            .addUserMessage("What's the best way to handle nulls in Java 21?")
            .build());

        // Get the text response
        String text = response.content().stream()
            .filter(block -> block.type().equals("text"))
            .map(block -> block.text())
            .findFirst()
            .orElse("");

        System.out.println(text);
        System.out.printf("Tokens used: %d input, %d output%n",
            response.usage().inputTokens(),
            response.usage().outputTokens());
    }
}
```

### Streaming with Java SDK

```java
var stream = client.messages().createStreaming(MessageCreateParams.builder()
    .model("claude-sonnet-4-0520")
    .maxTokens(1024)
    .addUserMessage("Explain Spring Boot auto-configuration")
    .build());

stream.forEach(event -> {
    if (event instanceof ContentBlockDeltaEvent delta) {
        System.out.print(delta.delta().text());
    }
});
```

### Python SDK (for comparison / scripting)

```python
import anthropic

client = anthropic.Anthropic()  # Uses ANTHROPIC_API_KEY env var

message = client.messages.create(
    model="claude-sonnet-4-0520",
    max_tokens=1024,
    system="You are a helpful assistant.",
    messages=[
        {"role": "user", "content": "Write a Python decorator for retry logic"}
    ]
)

print(message.content[0].text)
```

---

## 3. Spring AI Integration with Claude

### Why Spring AI?

Spring AI provides a unified abstraction over multiple AI providers (OpenAI, Anthropic, Google, etc.) with Spring Boot auto-configuration. If you are already in the Spring ecosystem, this is the recommended approach.

### Dependencies

```xml
<!-- pom.xml -->
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

<dependencies>
    <dependency>
        <groupId>org.springframework.ai</groupId>
        <artifactId>spring-ai-anthropic-spring-boot-starter</artifactId>
    </dependency>
</dependencies>
```

For Gradle:
```groovy
// build.gradle
dependencies {
    implementation platform('org.springframework.ai:spring-ai-bom:1.0.0')
    implementation 'org.springframework.ai:spring-ai-anthropic-spring-boot-starter'
}
```

### Configuration

```yaml
# application.yml
spring:
  ai:
    anthropic:
      api-key: ${ANTHROPIC_API_KEY}
      chat:
        options:
          model: claude-sonnet-4-0520
          max-tokens: 4096
          temperature: 0.3
```

### Using ChatClient

Spring AI's `ChatClient` is the primary interface:

```java
@Service
@RequiredArgsConstructor
public class AiAssistantService {

    private final ChatClient.Builder chatClientBuilder;

    public String askQuestion(String question) {
        ChatClient chatClient = chatClientBuilder
            .defaultSystem("You are a helpful assistant for an e-commerce platform.")
            .build();

        return chatClient.prompt()
            .user(question)
            .call()
            .content();
    }
}
```

### ChatClient with System Prompt and Parameters

```java
@Service
public class CodeReviewService {

    private final ChatClient chatClient;

    public CodeReviewService(ChatClient.Builder builder) {
        this.chatClient = builder
            .defaultSystem("""
                You are a senior Java code reviewer.
                Focus on: security, performance, thread safety, and Spring Boot best practices.
                Format findings as: SEVERITY | ISSUE | SUGGESTION
                """)
            .build();
    }

    public String reviewCode(String code) {
        return chatClient.prompt()
            .user("Review this Java code:\n\n```java\n" + code + "\n```")
            .call()
            .content();
    }
}
```

### Streaming Responses

```java
@RestController
@RequestMapping("/api/v1/ai")
@RequiredArgsConstructor
public class AiController {

    private final ChatClient chatClient;

    @GetMapping(value = "/stream", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
    public Flux<String> streamResponse(@RequestParam String question) {
        return chatClient.prompt()
            .user(question)
            .stream()
            .content();
    }
}
```

### Conversation Memory

Spring AI supports conversation memory via advisors:

```java
@Service
public class ChatService {

    private final ChatClient chatClient;
    private final ChatMemory chatMemory;

    public ChatService(ChatClient.Builder builder, ChatMemory chatMemory) {
        this.chatMemory = chatMemory;
        this.chatClient = builder
            .defaultSystem("You are a helpful assistant.")
            .defaultAdvisors(new MessageChatMemoryAdvisor(chatMemory))
            .build();
    }

    public String chat(String conversationId, String message) {
        return chatClient.prompt()
            .user(message)
            .advisors(a -> a.param(ChatMemory.CONVERSATION_ID, conversationId))
            .call()
            .content();
    }
}
```

---

## 4. Building a Chatbot with Spring Boot + Claude

### Project Structure

```
chatbot-service/
├── src/main/java/com/example/chatbot/
│   ├── ChatbotApplication.java
│   ├── config/
│   │   └── AiConfig.java
│   ├── controller/
│   │   └── ChatController.java
│   ├── service/
│   │   └── ChatService.java
│   ├── model/
│   │   ├── ChatRequest.java
│   │   └── ChatResponse.java
│   └── memory/
│       └── JdbcChatMemory.java
├── src/main/resources/
│   ├── application.yml
│   └── db/migration/
│       └── V1__create_chat_memory.sql
└── pom.xml
```

### Configuration

```java
@Configuration
public class AiConfig {

    @Bean
    public ChatMemory chatMemory() {
        // In-memory for development
        return new InMemoryChatMemory();
    }

    // For production, use a persistent store:
    // @Bean
    // public ChatMemory chatMemory(JdbcTemplate jdbcTemplate) {
    //     return new JdbcChatMemory(jdbcTemplate);
    // }
}
```

### DTOs

```java
public record ChatRequest(
    @NotBlank String conversationId,
    @NotBlank @Size(max = 10000) String message
) {}

public record ChatResponse(
    String conversationId,
    String message,
    long inputTokens,
    long outputTokens
) {}
```

### Service

```java
@Service
@Slf4j
public class ChatService {

    private final ChatClient chatClient;

    public ChatService(ChatClient.Builder builder, ChatMemory chatMemory) {
        this.chatClient = builder
            .defaultSystem("""
                You are a helpful customer support assistant for an e-commerce platform.
                You can help with order status, returns, product questions, and general inquiries.
                Be concise and friendly. If you don't know something, say so.
                Never make up order numbers or tracking information.
                """)
            .defaultAdvisors(new MessageChatMemoryAdvisor(chatMemory))
            .build();
    }

    public ChatResponse chat(ChatRequest request) {
        log.info("Processing chat message for conversation: {}", request.conversationId());

        ChatResponse aiResponse = chatClient.prompt()
            .user(request.message())
            .advisors(a -> a.param(ChatMemory.CONVERSATION_ID, request.conversationId()))
            .call()
            .chatResponse();

        var result = aiResponse.getResult();
        var usage = aiResponse.getMetadata().getUsage();

        return new ChatResponse(
            request.conversationId(),
            result.getOutput().getText(),
            usage.getPromptTokens(),
            usage.getCompletionTokens()
        );
    }

    public Flux<String> chatStream(ChatRequest request) {
        return chatClient.prompt()
            .user(request.message())
            .advisors(a -> a.param(ChatMemory.CONVERSATION_ID, request.conversationId()))
            .stream()
            .content();
    }
}
```

### Controller

```java
@RestController
@RequestMapping("/api/v1/chat")
@RequiredArgsConstructor
@Validated
public class ChatController {

    private final ChatService chatService;

    @PostMapping
    public ResponseEntity<ChatResponse> chat(@Valid @RequestBody ChatRequest request) {
        ChatResponse response = chatService.chat(request);
        return ResponseEntity.ok(response);
    }

    @PostMapping(value = "/stream", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
    public Flux<String> chatStream(@Valid @RequestBody ChatRequest request) {
        return chatService.chatStream(request);
    }
}
```

### Testing with curl

```bash
# Regular chat
curl -X POST http://localhost:8080/api/v1/chat \
  -H "Content-Type: application/json" \
  -d '{
    "conversationId": "conv-001",
    "message": "What is your return policy?"
  }'

# Streaming chat
curl -X POST http://localhost:8080/api/v1/chat/stream \
  -H "Content-Type: application/json" \
  -d '{
    "conversationId": "conv-001",
    "message": "Can you explain that in more detail?"
  }'
```

---

## 5. Tool Use / Function Calling

### What is Tool Use?

Tool use (also called function calling) lets Claude call functions you define. Claude decides when to call a tool based on the conversation, sends the parameters, and you execute the function and return the result.

```
User: "What's the status of order #12345?"
  ↓
Claude decides to call: getOrderStatus(orderId: "12345")
  ↓
Your code executes the function, returns: {status: "SHIPPED", trackingNumber: "1Z999..."}
  ↓
Claude responds: "Order #12345 has been shipped! Your tracking number is 1Z999..."
```

### Spring AI Tool Definition

```java
@Service
public class OrderTools {

    private final OrderRepository orderRepository;
    private final InventoryService inventoryService;

    public OrderTools(OrderRepository orderRepository, InventoryService inventoryService) {
        this.orderRepository = orderRepository;
        this.inventoryService = inventoryService;
    }

    @Tool(description = "Get the current status and details of an order by its ID")
    public OrderDetails getOrderStatus(
            @ToolParam(description = "The order ID (e.g., ORD-12345)") String orderId) {

        return orderRepository.findByOrderId(orderId)
            .map(order -> new OrderDetails(
                order.getOrderId(),
                order.getStatus().name(),
                order.getTrackingNumber(),
                order.getEstimatedDelivery()
            ))
            .orElseThrow(() -> new OrderNotFoundException(orderId));
    }

    @Tool(description = "Check if a product is currently in stock and available for purchase")
    public StockInfo checkProductAvailability(
            @ToolParam(description = "The product SKU") String productSku,
            @ToolParam(description = "Desired quantity") int quantity) {

        var stock = inventoryService.checkStock(productSku);
        return new StockInfo(
            productSku,
            stock.available(),
            stock.available() >= quantity,
            stock.estimatedRestockDate()
        );
    }

    @Tool(description = "Initiate a return request for an order")
    public ReturnConfirmation initiateReturn(
            @ToolParam(description = "The order ID to return") String orderId,
            @ToolParam(description = "Reason for the return") String reason) {

        // Validate the order is eligible for return
        var order = orderRepository.findByOrderId(orderId)
            .orElseThrow(() -> new OrderNotFoundException(orderId));

        if (!order.isReturnEligible()) {
            return new ReturnConfirmation(orderId, false,
                "Order is not eligible for return (past 30-day window)", null);
        }

        var returnId = returnService.createReturn(orderId, reason);
        return new ReturnConfirmation(orderId, true,
            "Return initiated successfully", returnId);
    }

    // Record DTOs for tool responses
    public record OrderDetails(String orderId, String status,
                               String trackingNumber, LocalDate estimatedDelivery) {}
    public record StockInfo(String sku, int available,
                            boolean sufficient, LocalDate restockDate) {}
    public record ReturnConfirmation(String orderId, boolean approved,
                                     String message, String returnId) {}
}
```

### Registering Tools with ChatClient

```java
@Service
public class CustomerSupportService {

    private final ChatClient chatClient;

    public CustomerSupportService(ChatClient.Builder builder,
                                   ChatMemory chatMemory,
                                   OrderTools orderTools) {
        this.chatClient = builder
            .defaultSystem("""
                You are a customer support assistant.
                Use the available tools to look up order information,
                check inventory, and process returns.
                Always verify order IDs before making changes.
                """)
            .defaultAdvisors(new MessageChatMemoryAdvisor(chatMemory))
            .defaultTools(orderTools)  // Register tools
            .build();
    }

    public String handleCustomerQuery(String conversationId, String query) {
        return chatClient.prompt()
            .user(query)
            .advisors(a -> a.param(ChatMemory.CONVERSATION_ID, conversationId))
            .call()
            .content();
    }
}
```

### Tool Use via Raw API

```java
// Using the Anthropic SDK directly
var tools = List.of(
    Tool.builder()
        .name("getOrderStatus")
        .description("Get the status of an order by ID")
        .inputSchema(Map.of(
            "type", "object",
            "properties", Map.of(
                "orderId", Map.of("type", "string", "description", "The order ID")
            ),
            "required", List.of("orderId")
        ))
        .build()
);

var response = client.messages().create(MessageCreateParams.builder()
    .model("claude-sonnet-4-0520")
    .maxTokens(1024)
    .tools(tools)
    .addUserMessage("What's the status of order ORD-12345?")
    .build());

// Check if Claude wants to call a tool
for (var block : response.content()) {
    if (block.type().equals("tool_use")) {
        String toolName = block.name();       // "getOrderStatus"
        JsonNode input = block.input();        // {"orderId": "ORD-12345"}
        String toolUseId = block.id();

        // Execute the tool
        String result = executeToolCall(toolName, input);

        // Send the result back to Claude
        var followUp = client.messages().create(MessageCreateParams.builder()
            .model("claude-sonnet-4-0520")
            .maxTokens(1024)
            .tools(tools)
            .messages(List.of(
                Message.user("What's the status of order ORD-12345?"),
                Message.assistant(response.content()),
                Message.toolResult(toolUseId, result)
            ))
            .build());
    }
}
```

---

## 6. Structured Outputs

### Getting JSON Responses

Claude can return structured JSON using Spring AI's output converters:

```java
@Service
public class AnalysisService {

    private final ChatClient chatClient;

    public AnalysisService(ChatClient.Builder builder) {
        this.chatClient = builder.build();
    }

    // Return a single object
    public CodeReview reviewCode(String code) {
        return chatClient.prompt()
            .system("You are a code reviewer. Analyse the provided code.")
            .user("Review this code:\n" + code)
            .call()
            .entity(CodeReview.class);
    }

    // Return a list
    public List<SecurityIssue> findSecurityIssues(String code) {
        return chatClient.prompt()
            .system("You are a security auditor.")
            .user("Find security issues in:\n" + code)
            .call()
            .entity(new ParameterizedTypeReference<List<SecurityIssue>>() {});
    }

    // DTOs
    public record CodeReview(
        String summary,
        List<Finding> findings,
        int overallScore  // 1-10
    ) {}

    public record Finding(
        String severity,  // CRITICAL, HIGH, MEDIUM, LOW
        int lineNumber,
        String issue,
        String suggestion,
        String codeExample
    ) {}

    public record SecurityIssue(
        String cweId,
        String title,
        String description,
        String remediation
    ) {}
}
```

### Using BeanOutputConverter Directly

```java
@Service
public class StructuredOutputService {

    private final ChatClient chatClient;

    public StructuredOutputService(ChatClient.Builder builder) {
        this.chatClient = builder.build();
    }

    public ApiDesign designApi(String requirements) {
        var converter = new BeanOutputConverter<>(ApiDesign.class);

        String response = chatClient.prompt()
            .system("You are an API designer. Design REST APIs based on requirements. "
                + converter.getFormat())
            .user(requirements)
            .call()
            .content();

        return converter.convert(response);
    }

    public record ApiDesign(
        String resourceName,
        String basePath,
        List<Endpoint> endpoints
    ) {}

    public record Endpoint(
        String method,
        String path,
        String description,
        String requestBody,
        String responseBody,
        int expectedStatusCode
    ) {}
}
```

---

## 7. Embeddings Strategy

### Important: Claude Does Not Have an Embedding Model

Unlike OpenAI, Anthropic does not offer a dedicated embedding model. For embedding-based features (vector search, RAG, semantic similarity), you need a separate embedding provider.

### Recommended Embedding Options with Spring AI

```yaml
# Option 1: OpenAI Embeddings (most common)
spring:
  ai:
    openai:
      api-key: ${OPENAI_API_KEY}
      embedding:
        options:
          model: text-embedding-3-small

# Option 2: Azure OpenAI Embeddings
spring:
  ai:
    azure:
      openai:
        api-key: ${AZURE_OPENAI_KEY}
        endpoint: ${AZURE_OPENAI_ENDPOINT}

# Option 3: Ollama (local, free)
spring:
  ai:
    ollama:
      embedding:
        options:
          model: nomic-embed-text
```

### Mixed Provider Architecture

```java
@Configuration
public class AiConfig {

    // Claude for chat/generation
    @Bean
    public ChatClient chatClient(ChatClient.Builder builder) {
        return builder
            .defaultSystem("You are a helpful assistant.")
            .build();
    }

    // OpenAI or other provider for embeddings
    // (auto-configured by spring-ai-openai-spring-boot-starter)

    // VectorStore using embeddings
    @Bean
    public VectorStore vectorStore(EmbeddingModel embeddingModel, JdbcTemplate jdbc) {
        return new PgVectorStore(jdbc, embeddingModel);
    }
}
```

### RAG with Claude + External Embeddings

```java
@Service
public class RagService {

    private final ChatClient chatClient;
    private final VectorStore vectorStore;

    public RagService(ChatClient.Builder builder,
                      VectorStore vectorStore) {
        this.vectorStore = vectorStore;
        this.chatClient = builder
            .defaultSystem("Answer questions using the provided context. "
                + "If the context doesn't contain the answer, say so.")
            .defaultAdvisors(new QuestionAnswerAdvisor(vectorStore))
            .build();
    }

    public String ask(String question) {
        return chatClient.prompt()
            .user(question)
            .call()
            .content();
    }
}
```

---

## 8. Error Handling, Retries, and Rate Limits

### Anthropic API Error Codes

| HTTP Status | Error Type | Cause | Action |
|-------------|-----------|-------|--------|
| 400 | Invalid Request | Bad parameters | Fix the request |
| 401 | Authentication | Invalid API key | Check API key |
| 403 | Permission Denied | Insufficient permissions | Check account |
| 429 | Rate Limit | Too many requests | Retry with backoff |
| 500 | API Error | Server issue | Retry |
| 529 | Overloaded | API is overloaded | Retry with longer backoff |

### Resilience4j Integration

```java
@Configuration
public class ResilienceConfig {

    @Bean
    public RetryConfig aiRetryConfig() {
        return RetryConfig.custom()
            .maxAttempts(3)
            .waitDuration(Duration.ofSeconds(2))
            .retryOnException(e -> {
                if (e instanceof WebClientResponseException wcre) {
                    int status = wcre.getStatusCode().value();
                    return status == 429 || status == 500 || status == 529;
                }
                return e instanceof java.net.SocketTimeoutException;
            })
            .intervalFunction(IntervalFunction.ofExponentialBackoff(
                Duration.ofSeconds(2), 2.0))
            .build();
    }

    @Bean
    public CircuitBreakerConfig aiCircuitBreakerConfig() {
        return CircuitBreakerConfig.custom()
            .failureRateThreshold(50)
            .waitDurationInOpenState(Duration.ofSeconds(30))
            .slidingWindowSize(10)
            .build();
    }
}
```

### Service with Resilience

```java
@Service
@Slf4j
public class ResilientAiService {

    private final ChatClient chatClient;
    private final Retry retry;
    private final CircuitBreaker circuitBreaker;

    public ResilientAiService(ChatClient.Builder builder,
                               RetryConfig retryConfig,
                               CircuitBreakerConfig cbConfig) {
        this.chatClient = builder.build();
        this.retry = Retry.of("claude-api", retryConfig);
        this.circuitBreaker = CircuitBreaker.of("claude-api", cbConfig);
    }

    public String ask(String question) {
        Supplier<String> decorated = Decorators.ofSupplier(() ->
                chatClient.prompt()
                    .user(question)
                    .call()
                    .content()
            )
            .withRetry(retry)
            .withCircuitBreaker(circuitBreaker)
            .decorate();

        try {
            return decorated.get();
        } catch (Exception e) {
            log.error("AI service call failed after retries", e);
            return "I'm sorry, the AI service is temporarily unavailable. Please try again later.";
        }
    }
}
```

### Rate Limit Headers

Anthropic returns rate limit information in response headers:

```
anthropic-ratelimit-requests-limit: 1000
anthropic-ratelimit-requests-remaining: 999
anthropic-ratelimit-requests-reset: 2024-01-15T00:00:00Z
anthropic-ratelimit-tokens-limit: 100000
anthropic-ratelimit-tokens-remaining: 95000
anthropic-ratelimit-tokens-reset: 2024-01-15T00:01:00Z
```

### Handling Rate Limits Proactively

```java
@Component
public class RateLimitTracker {

    private final AtomicInteger remainingRequests = new AtomicInteger(1000);
    private final AtomicLong remainingTokens = new AtomicLong(100000);

    public void updateFromHeaders(HttpHeaders headers) {
        Optional.ofNullable(headers.getFirst("anthropic-ratelimit-requests-remaining"))
            .ifPresent(v -> remainingRequests.set(Integer.parseInt(v)));
        Optional.ofNullable(headers.getFirst("anthropic-ratelimit-tokens-remaining"))
            .ifPresent(v -> remainingTokens.set(Long.parseLong(v)));
    }

    public boolean canMakeRequest(int estimatedTokens) {
        return remainingRequests.get() > 0 && remainingTokens.get() > estimatedTokens;
    }
}
```

---

## 9. Cost Management

### Token Cost Tracking

```java
@Service
@Slf4j
public class CostTrackingService {

    private final MeterRegistry meterRegistry;

    // Pricing per 1M tokens (Sonnet 4)
    private static final double INPUT_COST_PER_MILLION = 3.0;
    private static final double OUTPUT_COST_PER_MILLION = 15.0;

    public CostTrackingService(MeterRegistry meterRegistry) {
        this.meterRegistry = meterRegistry;
    }

    public void trackUsage(String operation, long inputTokens, long outputTokens) {
        double inputCost = (inputTokens / 1_000_000.0) * INPUT_COST_PER_MILLION;
        double outputCost = (outputTokens / 1_000_000.0) * OUTPUT_COST_PER_MILLION;
        double totalCost = inputCost + outputCost;

        // Prometheus metrics
        meterRegistry.counter("ai.tokens.input", "operation", operation)
            .increment(inputTokens);
        meterRegistry.counter("ai.tokens.output", "operation", operation)
            .increment(outputTokens);
        meterRegistry.counter("ai.cost.usd", "operation", operation)
            .increment(totalCost);

        log.info("AI usage - operation={} input={} output={} cost=${:.4f}",
            operation, inputTokens, outputTokens, totalCost);
    }
}
```

### Budget Guard

```java
@Component
public class BudgetGuard {

    @Value("${ai.budget.daily-limit-usd:10.0}")
    private double dailyLimit;

    private final AtomicReference<Double> dailySpend = new AtomicReference<>(0.0);
    private final AtomicReference<LocalDate> currentDate = new AtomicReference<>(LocalDate.now());

    public void checkBudget(double estimatedCost) {
        // Reset daily counter if new day
        LocalDate today = LocalDate.now();
        if (!today.equals(currentDate.get())) {
            currentDate.set(today);
            dailySpend.set(0.0);
        }

        double newTotal = dailySpend.updateAndGet(v -> v + estimatedCost);
        if (newTotal > dailyLimit) {
            throw new BudgetExceededException(
                "Daily AI budget exceeded: $%.2f / $%.2f".formatted(newTotal, dailyLimit));
        }
    }
}
```

### Model Routing for Cost Optimisation

```java
@Service
public class ModelRouter {

    private final ChatClient sonnetClient;
    private final ChatClient haikuClient;
    private final ChatClient opusClient;

    public ModelRouter(
            @Qualifier("sonnetBuilder") ChatClient.Builder sonnetBuilder,
            @Qualifier("haikuBuilder") ChatClient.Builder haikuBuilder,
            @Qualifier("opusBuilder") ChatClient.Builder opusBuilder) {
        this.sonnetClient = sonnetBuilder.build();
        this.haikuClient = haikuBuilder.build();
        this.opusClient = opusBuilder.build();
    }

    public String route(String query, TaskComplexity complexity) {
        return switch (complexity) {
            case SIMPLE -> haikuClient.prompt().user(query).call().content();     // $0.80/M in
            case STANDARD -> sonnetClient.prompt().user(query).call().content();  // $3/M in
            case COMPLEX -> opusClient.prompt().user(query).call().content();     // $15/M in
        };
    }

    public enum TaskComplexity { SIMPLE, STANDARD, COMPLEX }
}
```

---

## 10. Practical Project: AI-Powered Order Assistant

### Overview

Build a Spring Boot service that provides an AI-powered order management assistant with:
- Natural language order queries
- Tool use for real-time order lookup
- Streaming responses
- Conversation memory
- Cost tracking

### Complete Implementation

**application.yml:**
```yaml
spring:
  application:
    name: order-assistant
  ai:
    anthropic:
      api-key: ${ANTHROPIC_API_KEY}
      chat:
        options:
          model: claude-sonnet-4-0520
          max-tokens: 2048
          temperature: 0.2
  datasource:
    url: jdbc:postgresql://localhost:5432/orderdb
    username: ${DB_USERNAME}
    password: ${DB_PASSWORD}

ai:
  budget:
    daily-limit-usd: 25.0
```

**OrderAssistantService.java:**
```java
@Service
@Slf4j
public class OrderAssistantService {

    private final ChatClient chatClient;
    private final CostTrackingService costTracker;

    public OrderAssistantService(ChatClient.Builder builder,
                                  ChatMemory chatMemory,
                                  OrderTools orderTools,
                                  CostTrackingService costTracker) {
        this.costTracker = costTracker;
        this.chatClient = builder
            .defaultSystem("""
                You are an order management assistant for our e-commerce platform.

                Capabilities:
                - Look up order status, details, and tracking information
                - Check product availability and stock levels
                - Process return requests (within 30-day window)

                Guidelines:
                - Always verify order IDs using the tools before responding
                - Be concise and helpful
                - If an order is not found, ask the customer to double-check the ID
                - For returns, explain the policy before processing
                - Never fabricate order details or tracking numbers
                """)
            .defaultAdvisors(new MessageChatMemoryAdvisor(chatMemory))
            .defaultTools(orderTools)
            .build();
    }

    public AssistantResponse chat(String conversationId, String message) {
        log.info("Chat request - conversation={}", conversationId);

        var chatResponse = chatClient.prompt()
            .user(message)
            .advisors(a -> a.param(ChatMemory.CONVERSATION_ID, conversationId))
            .call()
            .chatResponse();

        var usage = chatResponse.getMetadata().getUsage();

        costTracker.trackUsage("order-assistant",
            usage.getPromptTokens(), usage.getCompletionTokens());

        return new AssistantResponse(
            conversationId,
            chatResponse.getResult().getOutput().getText(),
            usage.getPromptTokens(),
            usage.getCompletionTokens()
        );
    }

    public Flux<String> chatStream(String conversationId, String message) {
        return chatClient.prompt()
            .user(message)
            .advisors(a -> a.param(ChatMemory.CONVERSATION_ID, conversationId))
            .stream()
            .content();
    }

    public record AssistantResponse(
        String conversationId,
        String message,
        long inputTokens,
        long outputTokens
    ) {}
}
```

**OrderAssistantController.java:**
```java
@RestController
@RequestMapping("/api/v1/assistant")
@RequiredArgsConstructor
@Validated
public class OrderAssistantController {

    private final OrderAssistantService assistantService;

    @PostMapping("/chat")
    public ResponseEntity<OrderAssistantService.AssistantResponse> chat(
            @Valid @RequestBody ChatRequest request) {
        var response = assistantService.chat(request.conversationId(), request.message());
        return ResponseEntity.ok(response);
    }

    @PostMapping(value = "/chat/stream", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
    public Flux<String> chatStream(@Valid @RequestBody ChatRequest request) {
        return assistantService.chatStream(request.conversationId(), request.message());
    }

    public record ChatRequest(
        @NotBlank String conversationId,
        @NotBlank @Size(max = 5000) String message
    ) {}
}
```

### Testing the Assistant

```bash
# Ask about an order
curl -X POST http://localhost:8080/api/v1/assistant/chat \
  -H "Content-Type: application/json" \
  -d '{"conversationId": "sess-001", "message": "Where is my order ORD-12345?"}'

# Follow-up (uses conversation memory)
curl -X POST http://localhost:8080/api/v1/assistant/chat \
  -H "Content-Type: application/json" \
  -d '{"conversationId": "sess-001", "message": "Can I return it?"}'

# Check stock
curl -X POST http://localhost:8080/api/v1/assistant/chat \
  -H "Content-Type: application/json" \
  -d '{"conversationId": "sess-002", "message": "Is the blue jacket (SKU-789) available in quantity 3?"}'
```

---

## 11. Try This Exercises

### Exercise 1: Basic API Call
Use curl to make a direct API call to Claude. Send a system prompt describing your project and ask for a code review of one of your controllers.

### Exercise 2: Spring AI Setup
Add Spring AI with the Anthropic starter to one of your Spring Boot projects. Create a simple `/api/ai/ask` endpoint that forwards questions to Claude.

### Exercise 3: Tool Use
Define a tool that queries your database (read-only) and register it with ChatClient. Ask Claude questions that require database lookups.

### Exercise 4: Structured Output
Create a service that analyses Java code and returns a structured `CodeQuality` record with metrics like complexity, test coverage suggestions, and improvement priorities.

### Exercise 5: Full Assistant
Build the Order Assistant from Section 10. Add at least one custom tool that queries your actual database. Test with a multi-turn conversation.

---

## 12. Tips and Gotchas

### Tips

1. **Use Spring AI over raw SDK for Spring Boot projects.** The auto-configuration, advisors, and integration patterns save significant boilerplate.

2. **Always set a system prompt.** It dramatically improves response quality and consistency. Include your domain context and constraints.

3. **Use streaming for user-facing features.** Nobody wants to wait 5+ seconds staring at a loading spinner. Stream the response.

4. **Track costs from day one.** Instrument token counting in your first implementation. Costs can surprise you.

5. **Use Sonnet for most API calls.** Reserve Opus for complex reasoning tasks. Use Haiku for classification and simple extraction.

6. **Tool descriptions matter enormously.** Claude decides whether to call a tool based on its description. Be precise and include examples.

7. **Validate tool inputs.** Claude may send unexpected values. Validate and sanitise all tool parameters before executing.

### Gotchas

1. **Claude does not have embeddings.** You need a separate provider (OpenAI, Cohere, Ollama) for vector search and RAG.

2. **Token limits include conversation history.** With memory advisors, long conversations accumulate tokens. Implement a sliding window or summary strategy.

3. **Tool use requires multiple API calls.** When Claude calls a tool, you need to: (a) get the tool call, (b) execute it, (c) send the result back. Spring AI handles this automatically, but be aware of the latency.

4. **Streaming and tool use together are complex.** When streaming, tool calls arrive as events that you need to handle mid-stream. Spring AI manages this, but custom implementations need care.

5. **API keys in environment variables, never in code.** Use `${ANTHROPIC_API_KEY}` in application.yml, never hardcode keys.

6. **Rate limits are per-organisation, not per-key.** Multiple services sharing an org share rate limits. Plan accordingly.

7. **Max tokens is for output only.** Setting `max-tokens: 4096` means Claude's response can be up to 4096 tokens. It does not limit input.

8. **Spring AI versions move fast.** Spring AI is under active development. Pin your version in the BOM and test upgrades carefully.

---

## Summary

| Concept | Key Takeaway |
|---------|-------------|
| Messages API | model + messages[] + max_tokens; streaming via SSE |
| Spring AI | Use `spring-ai-anthropic-spring-boot-starter`; ChatClient is the primary interface |
| Tool Use | Define tools with `@Tool`; register with `defaultTools()`; Claude calls them as needed |
| Structured Output | Use `.entity(MyClass.class)` for typed responses |
| Embeddings | Claude has no embedding model; use OpenAI, Cohere, or Ollama |
| Resilience | Resilience4j for retries and circuit breakers; handle 429 and 529 errors |
| Cost | Track tokens, implement budget guards, route to cheaper models for simple tasks |

**Next:** [Tutorial 07 — Claude in CI/CD and Automation](07-claude-in-ci-cd-and-automation.md)
