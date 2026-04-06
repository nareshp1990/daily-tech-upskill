# Agentic AI — Day 2: Spring AI Agents — Production-Ready Single Agent
**Track:** Agentic AI (Implementation-Focused)
**Time Target:** 1–2 hours

---

## 1. Today's Focus
**Building a Production-Ready Agent with Spring AI**

Yesterday was setup and overview. Today you build a real agent in Spring Boot: a **Personal Finance Assistant** that can check balances, categorise transactions, and provide spending insights — using Spring AI's ChatClient with function calling. You'll implement proper tool definitions, error handling, conversation memory, and structured responses.

---

## 2. Project: Personal Finance Assistant Agent

### 2.1 Architecture

```
User → REST API → ChatClient (Claude/GPT)
                       │
                       │ ReAct loop (automatic in Spring AI)
                       │
              ┌────────┴────────────────────┐
              │         Tool Belt            │
              ├─────────┬──────────┬─────────┤
              │ Account │Transaction│ Budget  │
              │ Tools   │ Tools     │ Tools   │
              └─────────┴──────────┴─────────┘
                   │          │          │
                   ▼          ▼          ▼
              AccountDB  TxnService  BudgetSvc
```

### 2.2 Domain Model

```java
public record Account(
    Long id,
    String name,
    String type,          // CHECKING, SAVINGS, CREDIT_CARD
    BigDecimal balance,
    String currency
) {}

public record Transaction(
    Long id,
    Long accountId,
    BigDecimal amount,
    String category,       // FOOD, TRANSPORT, ENTERTAINMENT, etc.
    String merchant,
    LocalDate date,
    String description
) {}

public record SpendingSummary(
    String category,
    BigDecimal totalSpent,
    int transactionCount,
    BigDecimal avgTransaction
) {}
```

### 2.3 Agent Tools

```java
@Component
@RequiredArgsConstructor
public class AccountTools {

    private final AccountRepository accountRepo;
    private final TransactionRepository txnRepo;

    @Tool("Get all accounts with their current balances for the user")
    public List<Account> getAccounts(@ToolParam("User ID") Long userId) {
        return accountRepo.findByUserId(userId);
    }

    @Tool("Get the current balance of a specific account")
    public BigDecimal getBalance(
            @ToolParam("Account ID") Long accountId) {
        return accountRepo.findById(accountId)
            .map(Account::balance)
            .orElseThrow(() -> new IllegalArgumentException("Account not found: " + accountId));
    }

    @Tool("Get recent transactions for an account, optionally filtered by category")
    public List<Transaction> getTransactions(
            @ToolParam("Account ID") Long accountId,
            @ToolParam(value = "Category filter (e.g. FOOD, TRANSPORT). Pass 'ALL' for no filter", required = false) String category,
            @ToolParam(value = "Number of days to look back, default 30", required = false) Integer days) {

        int lookbackDays = days != null ? days : 30;
        LocalDate since = LocalDate.now().minusDays(lookbackDays);

        if (category != null && !category.equals("ALL")) {
            return txnRepo.findByAccountIdAndCategoryAndDateAfter(accountId, category, since);
        }
        return txnRepo.findByAccountIdAndDateAfter(accountId, since);
    }

    @Tool("Get spending summary grouped by category for a date range")
    public List<SpendingSummary> getSpendingSummary(
            @ToolParam("Account ID") Long accountId,
            @ToolParam("Start date (YYYY-MM-DD)") String startDate,
            @ToolParam("End date (YYYY-MM-DD)") String endDate) {

        LocalDate start = LocalDate.parse(startDate);
        LocalDate end = LocalDate.parse(endDate);
        return txnRepo.getSpendingSummary(accountId, start, end);
    }

    @Tool("Set a monthly budget for a spending category")
    public String setBudget(
            @ToolParam("User ID") Long userId,
            @ToolParam("Spending category (e.g. FOOD, TRANSPORT)") String category,
            @ToolParam("Monthly budget amount") BigDecimal amount) {
        budgetService.setBudget(userId, category, amount);
        return "Budget set: " + category + " = $" + amount + "/month";
    }
}
```

### 2.4 Agent Configuration

```java
@Configuration
public class FinanceAgentConfig {

    @Bean
    public ChatClient financeAgent(ChatClient.Builder builder) {
        return builder
            .defaultSystem("""
                You are a personal finance assistant. You help users understand
                their spending, check balances, and manage budgets.

                Rules:
                - Always verify account existence before querying transactions.
                - When showing amounts, use proper currency formatting.
                - For spending analysis, default to the last 30 days unless specified.
                - Never disclose sensitive account numbers — use account names.
                - If a user asks about budgets, check current spending first.
                - Be concise but helpful. Use bullet points for summaries.

                Current user ID: {userId}
                """)
            .defaultTools(new AccountTools(accountRepo, txnRepo, budgetService))
            .build();
    }
}
```

### 2.5 REST API with Conversation Memory

```java
@RestController
@RequestMapping("/v1/finance-agent")
@RequiredArgsConstructor
public class FinanceAgentController {

    private final ChatClient financeAgent;
    private final ChatMemory chatMemory;

    @PostMapping("/chat")
    public AgentResponse chat(@RequestBody AgentRequest request) {
        String response = financeAgent.prompt()
            .system(s -> s.param("userId", request.userId()))
            .user(request.message())
            .advisors(new MessageChatMemoryAdvisor(chatMemory,
                request.conversationId(), 20))   // Keep last 20 messages
            .call()
            .content();

        return new AgentResponse(request.conversationId(), response);
    }
}

public record AgentRequest(String conversationId, Long userId, String message) {}
public record AgentResponse(String conversationId, String message) {}
```

---

## 3. Conversation Memory

```java
@Configuration
public class MemoryConfig {

    // In-memory for dev — use Redis for production
    @Bean
    @Profile("dev")
    public ChatMemory inMemoryChatMemory() {
        return new InMemoryChatMemory();
    }

    // Redis-backed for production
    @Bean
    @Profile("prod")
    public ChatMemory redisChatMemory(RedisTemplate<String, Object> redisTemplate) {
        return new RedisChatMemory(redisTemplate, Duration.ofHours(24));
    }
}
```

---

## 4. Error Handling & Guard Rails

```java
@Component
public class AgentGuardRails {

    // Wrap tool calls with safety checks
    @Tool("Transfer money between accounts")
    public String transferMoney(
            @ToolParam("Source account ID") Long fromAccount,
            @ToolParam("Destination account ID") Long toAccount,
            @ToolParam("Amount to transfer") BigDecimal amount) {

        // Guard: max single transfer limit
        if (amount.compareTo(new BigDecimal("10000")) > 0) {
            return "ERROR: Transfer amount exceeds $10,000 limit. " +
                   "Please contact support for large transfers.";
        }

        // Guard: never transfer to external accounts via agent
        if (!accountRepo.isInternalAccount(toAccount)) {
            return "ERROR: Agent cannot transfer to external accounts. " +
                   "Please use the secure banking portal.";
        }

        // Guard: require explicit confirmation
        return "CONFIRMATION REQUIRED: Transfer $" + amount +
               " from account " + fromAccount + " to " + toAccount +
               ". Please confirm by saying 'yes, transfer'.";
    }
}
```

### Tool Execution Timeout

```java
@Configuration
public class ToolTimeoutConfig {

    @Bean
    public FunctionCallbackWrapper<?> withTimeout(AccountTools tools) {
        // Wrap tool execution with timeout
        return FunctionCallbackWrapper.builder(tools)
            .withExecutionTimeout(Duration.ofSeconds(5))
            .withFallback(e -> "Tool execution timed out. Please try again.")
            .build();
    }
}
```

---

## 5. Testing the Agent

```java
@SpringBootTest
class FinanceAgentTest {

    @Autowired ChatClient financeAgent;

    @Test
    void shouldCheckBalanceAndSummariseSpending() {
        String response = financeAgent.prompt()
            .system(s -> s.param("userId", "1"))
            .user("What's my checking account balance and how much did I spend on food this month?")
            .call()
            .content();

        assertThat(response).containsAnyOf("balance", "Balance", "$");
        assertThat(response).containsAnyOf("food", "Food", "FOOD");
    }

    @Test
    void shouldRefuseHighValueTransfer() {
        String response = financeAgent.prompt()
            .system(s -> s.param("userId", "1"))
            .user("Transfer $50,000 to account 999")
            .call()
            .content();

        assertThat(response).containsAnyOf("exceeds", "limit", "cannot");
    }
}
```

---

## 6. Hands-On Exercise

1. Create a new Spring Boot project with Spring AI.
2. Implement the `AccountTools` class with mock data (in-memory lists or H2 database).
3. Configure the finance agent with system prompt and tools.
4. Test these conversations:
   - "What are my account balances?"
   - "Show me my food spending this month"
   - "How does my spending compare to last month?"
   - "Set a $500 budget for dining"
5. Add conversation memory and verify the agent remembers context across messages.

---

## 7. Key Takeaways

1. **Tools are the agent's hands** — well-designed tool descriptions determine whether the LLM picks the right tool.
2. **Guard rails are non-negotiable** — limits, confirmations, and scope restrictions prevent dangerous actions.
3. **Conversation memory enables multi-turn interactions** — without it, every message starts from scratch.
4. **The system prompt is your most important configuration** — it defines the agent's personality, rules, and constraints.

---

*Day 2. A tool well-described is a tool well-used — the LLM only knows what you tell it.*
