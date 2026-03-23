# Tutorial 7: Daily Workflow Playbook

## Table of Contents

- [Morning: Review and Triage](#morning-review-and-triage)
- [During Development: Completions + Chat Combo](#during-development-completions--chat-combo)
- [Testing: Generate Unit and Integration Tests](#testing-generate-unit-and-integration-tests)
- [DevOps: Docker, K8s, Terraform](#devops-docker-k8s-terraform)
- [Database: SQL Query Writing and Optimization](#database-sql-query-writing-and-optimization)
- [Documentation: Auto-Generate Javadoc and API Docs](#documentation-auto-generate-javadoc-and-api-docs)
- [Code Review: Copilot as a Reviewer](#code-review-copilot-as-a-reviewer)
- [Quick Reference: Shortcuts and Commands](#quick-reference-shortcuts-and-commands)
- [Comparison: Copilot vs Claude Code](#comparison-copilot-vs-claude-code)
- [Try This Exercises](#try-this-exercises)

---

## Morning: Review and Triage

### Review PRs with Copilot

When you have PRs to review in the morning:

#### 1. Understand the PR Quickly

Open the PR on GitHub.com. If Copilot has generated a summary, read it first. If not, use the Copilot chat on the PR page:

```
Summarize the changes in this PR.
What is the main purpose of this change?
Are there any breaking changes?
```

#### 2. Review Specific Files

In the PR diff view, select complex code blocks and ask:

```
/explain this change — why was the old approach replaced?

Is there a potential race condition in this code?

Does this change handle all error cases?
```

#### 3. Check Copilot's Automated Review

If Copilot has already reviewed the PR:
- Read Copilot's comments first — they catch mechanical issues
- Focus your review on business logic, architecture, and domain knowledge
- Flag anything Copilot missed that requires domain context

#### 4. CLI-Based PR Review

```bash
# List PRs awaiting your review
gh pr list --search "review-requested:@me"

# Check out a PR locally for testing
gh pr checkout 123

# View the PR diff
gh pr diff 123
```

### Triage Issues

```bash
# List open issues assigned to you
gh issue list --assignee @me --state open

# For each issue, ask Copilot for an implementation estimate
gh copilot suggest "estimate complexity of implementing user order history with pagination"
```

---

## During Development: Completions + Chat Combo

### The Flow

Your development workflow should alternate between completions and chat:

```
Comment → Tab (accept completion) → Review → Adjust → Comment → Tab → ...
Chat when stuck → Apply suggestion → Continue with completions
```

### Workflow Pattern: New Feature

#### Step 1: Plan with Chat

```
@workspace I need to add a product search feature with:
- Full-text search on name and description
- Filter by category, price range
- Sort by relevance, price, date
- Pagination

What files do I need to create/modify? Suggest the approach.
```

#### Step 2: Generate DTOs First

Open `SearchProductRequest.java` and type:

```java
// Search request with fulltext query, category filter, price range, sort, and pagination
public record SearchProductRequest(
```

Let Copilot complete the record fields.

#### Step 3: Write the Interface

```java
public interface ProductSearchService {

    // Search products with filters and pagination
    Page<ProductResponse> search(SearchProductRequest request);
```

#### Step 4: Implement with Completions

Open `ProductSearchServiceImpl.java`. With the interface tab open, type:

```java
@Service
@RequiredArgsConstructor
@Slf4j
public class ProductSearchServiceImpl implements ProductSearchService {

    private final ProductRepository productRepository;

    @Override
    public Page<ProductResponse> search(SearchProductRequest request) {
        // Build JPA Specification from search criteria
```

Copilot generates the implementation using the open interface and DTO as context.

#### Step 5: Ask Chat When Stuck

If the generated Specification logic is not quite right:

```
/fix The JPA Specification for price range filtering doesn't handle
the case where only minPrice is provided (no maxPrice).
Fix #selection to handle optional filters correctly.
```

### Workflow Pattern: Bug Fix

#### Step 1: Understand the Bug

```
@workspace /explain The order total is sometimes calculated incorrectly
when a coupon is applied. Show me the code path from OrderController
through to the total calculation.
```

#### Step 2: Find the Issue

```
@workspace Search for all places where order total or discount is calculated.
Are there any inconsistencies?
```

#### Step 3: Fix with Inline Chat

Select the buggy code, `Ctrl+I`:

```
Fix this calculation. The discount should be applied before tax,
not after. The correct formula is: (subtotal - discount) * (1 + taxRate)
```

#### Step 4: Verify with Tests

```
/tests Generate a test that verifies the order total calculation
with: subtotal=$100, discount=10%, tax=8%.
Expected total should be $97.20, not $100.80.
```

---

## Testing: Generate Unit and Integration Tests

### Unit Test Workflow

#### 1. Select the Method to Test

Select the method in your editor, then in Chat:

```
/tests Generate comprehensive JUnit 5 unit tests for #selection.
Use Mockito for all dependencies. Include:
- Happy path
- Null/empty input
- Edge cases (boundary values)
- Exception scenarios
Use AssertJ assertions and @Nested for grouping.
```

#### 2. Review and Customize

Copilot generates tests. Review and adjust:
- Verify mock setup is correct
- Check that assertions match expected behavior
- Add domain-specific edge cases Copilot missed

#### 3. Quick Test Patterns with Completions

After writing one test method manually, Copilot learns the pattern:

```java
@Nested
class CreateOrder {

    @Test
    void should_createOrder_when_inventoryIsAvailable() {
        // Write this one manually...
    }

    @Test
    void should_throwException_when_inventoryIsInsufficient() {
        // Copilot generates this following your pattern...
    }

    @Test
    void should_publishEvent_when_orderIsCreated() {
        // Copilot continues the pattern...
    }
}
```

### Integration Test Workflow

```
Generate an integration test for OrderController using:
- @SpringBootTest(webEnvironment = RANDOM_PORT)
- @Testcontainers with MySQL and Kafka containers
- WebTestClient for HTTP requests
- Test the full flow: POST /api/v1/orders → verify DB → verify Kafka message

Use our existing TestContainerConfig class for container setup.
```

### Test Data Generation

```
Generate a TestFixtures class for the Order domain:
- validOrder(): returns a fully populated Order entity
- orderWithStatus(OrderStatus): returns an order with specific status
- createOrderRequest(): returns a valid CreateOrderRequest record
- invalidOrderRequest(): returns a request that fails validation
Use the Builder pattern.
```

---

## DevOps: Docker, K8s, Terraform

### Docker Workflow

#### Dockerfile Generation

Open a new `Dockerfile` and type:

```dockerfile
# Optimized multi-stage build for Spring Boot 3 with Java 21
# Layer caching for dependencies, non-root user, health check, JVM tuning
```

Copilot generates an optimized Dockerfile.

#### Docker Compose Updates

In Chat:

```
Add a Prometheus and Grafana stack to our docker-compose.yml.
Prometheus should scrape our Spring Boot /actuator/prometheus endpoint.
Grafana should have the Prometheus datasource pre-configured.
Include volume mounts for config persistence.
```

#### Docker Troubleshooting from CLI

```bash
?? "why is my Docker container exiting with code 137"
# Explanation: OOMKilled — container exceeded memory limit

?? "find which Docker container is consuming the most memory right now"
# docker stats --no-stream --format "table {{.Name}}\t{{.MemUsage}}" | sort -k2 -rh | head -5
```

### Kubernetes Workflow

#### Generate Manifests

```
@workspace Generate Kubernetes manifests for deploying our order-service:
- Deployment with 3 replicas
- Service (ClusterIP)
- HPA: scale from 3 to 10 pods at 70% CPU
- ConfigMap from application-prod.yml
- Secret for database credentials (reference Azure Key Vault CSI)
- Ingress with TLS
Match our existing naming conventions from #file:k8s/user-service/deployment.yaml
```

#### Debug K8s Issues

```bash
?? "pod is stuck in Pending state, how to find out why"
# kubectl describe pod <pod-name> -n <namespace> | grep -A 5 "Events"

?? "check if Kubernetes node has enough resources for pending pods"
# kubectl describe nodes | grep -A 5 "Allocated resources"
```

### Terraform Workflow

#### Module Generation

```
Generate a Terraform module for Azure App Service with:
- Linux App Service Plan (P1v3)
- Web App with Java 21 runtime
- Deployment slots (staging, production)
- Application Insights integration
- Virtual network integration
- Managed identity for Azure Key Vault access
Follow our module structure from #file:terraform/modules/aks/main.tf
```

#### Terraform Troubleshooting

```
/explain This Terraform plan shows 3 resources being destroyed and recreated.
Why would changing the subnet address range force a recreate of the AKS cluster?
How can I change the subnet without destroying the cluster?
```

---

## Database: SQL Query Writing and Optimization

### Query Generation

In a `.sql` file or in Chat:

```
Write a MySQL query to find the top 10 customers by total order value
in the last 90 days, including their order count and average order value.
Exclude cancelled orders. Include customers who have at least 3 orders.
```

Copilot generates:

```sql
SELECT
    c.id,
    c.name,
    c.email,
    COUNT(o.id) AS order_count,
    SUM(o.total_amount) AS total_value,
    AVG(o.total_amount) AS avg_order_value
FROM customers c
INNER JOIN orders o ON c.id = o.customer_id
WHERE o.created_at >= DATE_SUB(CURRENT_DATE, INTERVAL 90 DAY)
  AND o.status != 'CANCELLED'
GROUP BY c.id, c.name, c.email
HAVING COUNT(o.id) >= 3
ORDER BY total_value DESC
LIMIT 10;
```

### Query Optimization

Select a slow query and ask:

```
/optimize This query takes 5 seconds on a table with 10M rows.
The orders table has indexes on: id (PK), customer_id, created_at.
Suggest optimizations including index changes.
```

### Migration Generation

```
Generate a Flyway migration V012__add_product_search_index.sql that:
1. Adds a fulltext index on products(name, description) for MySQL 8
2. Adds a composite index on products(category, price) for filter queries
3. Adds a partial index on orders(status) WHERE status = 'PENDING'
Make it idempotent and include a rollback comment.
```

### JPA Query Generation

In a Repository interface:

```java
// Find all orders placed by a customer in the last N days
// with a total amount greater than the specified threshold
// Sorted by creation date descending
// Use @Query with JPQL
```

---

## Documentation: Auto-Generate Javadoc and API Docs

### Javadoc Generation

Select a class or method, then `Ctrl+I`:

```
Add comprehensive Javadoc. Include @param, @return, @throws.
Explain the business purpose, not just the implementation.
```

### Batch Javadoc

In Chat:

```
Generate Javadoc for all public methods in #file:OrderService.java.
Focus on business context, parameters, return types, and exceptions.
Follow this format:

/**
 * Brief summary in one sentence.
 *
 * <p>Detailed description if needed.</p>
 *
 * @param paramName description
 * @return description
 * @throws ExceptionType when condition
 * @since 1.2.0
 */
```

### OpenAPI Documentation

```
Add OpenAPI/Swagger annotations to #file:OrderController.java.
Include:
- @Operation with summary and description
- @ApiResponse for 200, 400, 404, 500
- @Parameter descriptions
- @Schema on request/response DTOs
Use SpringDoc OpenAPI 3 annotations.
```

Copilot generates:

```java
@Operation(
    summary = "Create a new order",
    description = "Creates an order for the authenticated customer. Validates inventory availability."
)
@ApiResponses({
    @ApiResponse(responseCode = "201", description = "Order created",
        content = @Content(schema = @Schema(implementation = OrderResponse.class))),
    @ApiResponse(responseCode = "400", description = "Invalid request",
        content = @Content(schema = @Schema(implementation = ProblemDetail.class))),
    @ApiResponse(responseCode = "422", description = "Insufficient inventory",
        content = @Content(schema = @Schema(implementation = ProblemDetail.class)))
})
@PostMapping
public ResponseEntity<OrderResponse> createOrder(
        @Valid @RequestBody CreateOrderRequest request) {
```

### README and Architecture Docs

```
@workspace Generate a README for this microservice that includes:
- Service description
- Tech stack
- How to run locally (Docker Compose)
- API endpoints (from controllers)
- Configuration properties
- Architecture diagram (Mermaid syntax)
```

---

## Code Review: Copilot as a Reviewer

### Self-Review Before Pushing

Before creating a PR, use Copilot to review your own changes:

#### 1. Review in Chat

```
@workspace Review the changes I made today. Look for:
- Potential bugs
- Missing error handling
- Performance issues
- Security concerns
- Inconsistencies with existing code

Files I changed: #file:OrderService.java #file:OrderController.java
#file:OrderServiceImplTest.java
```

#### 2. Specific Reviews

```
# Security review
Review #file:AuthController.java for security vulnerabilities.
Check for: injection, broken auth, broken access control,
sensitive data exposure.

# Performance review
Review #file:ReportService.java for performance issues.
Are there N+1 queries? Unnecessary database calls? Missing caching?

# Concurrency review
Review #file:InventoryService.java for thread safety.
Multiple instances of this service run in production behind a load balancer.
```

### Review Checklist with Copilot

```
For my changes, verify:
1. All new public methods have Javadoc
2. All @Transactional annotations are at the correct level
3. All exceptions are handled (no unhandled RuntimeExceptions)
4. All new endpoints are secured with @PreAuthorize
5. All new database columns have indexes where appropriate
6. All new Kafka producers handle send failures
```

---

## Quick Reference: Shortcuts and Commands

### VS Code Keyboard Shortcuts

| Action | Shortcut | When to Use |
|--------|----------|-------------|
| Accept suggestion | `Tab` | Code completion looks right |
| Dismiss suggestion | `Esc` | Suggestion is wrong |
| Next suggestion | `Alt + ]` | Want to see alternatives |
| Previous suggestion | `Alt + [` | Went past the good one |
| Accept next word | `Ctrl + Right` | Partial acceptance |
| Completions panel | `Ctrl + Enter` | See all 10 options |
| Trigger manually | `Alt + \` | Force a suggestion |
| Inline chat | `Ctrl + I` | Quick edit at cursor |
| Chat panel | `Ctrl + Alt + I` | Long conversation |
| Quick chat | `Ctrl + Shift + I` | Fast question |

### Chat Commands Cheat Sheet

| Command | Example |
|---------|---------|
| `/explain` | `/explain this Kafka consumer` |
| `/fix` | `/fix the null pointer in this method` |
| `/tests` | `/tests JUnit 5 with Mockito` |
| `/doc` | `/doc add Javadoc to this class` |
| `/optimize` | `/optimize this SQL query` |
| `/simplify` | `/simplify reduce nesting` |
| `/new` | `/new Spring Boot REST controller for products` |

### Chat Participants

| Participant | When to Use |
|-------------|-------------|
| `@workspace` | Project-wide questions |
| `@terminal` | Terminal error help |
| `@vscode` | IDE configuration |
| `@github` | Cross-repo search (Enterprise) |

### Context References

| Reference | What It Points To |
|-----------|------------------|
| `#file:Name.java` | Specific file |
| `#selection` | Selected code |
| `#editor` | Visible editor content |
| `#terminalLastCommand` | Last terminal command |

### CLI Quick Reference

```bash
# Suggest a command
gh copilot suggest "your description"

# Explain a command
gh copilot explain "command to explain"

# Suggest with type hint
gh copilot suggest -t shell "..."
gh copilot suggest -t git "..."
gh copilot suggest -t gh "..."
```

### Aliases (~/.zshrc)

```bash
alias '??'='gh copilot suggest -t shell'
alias 'git?'='gh copilot suggest -t git'
alias 'gh?'='gh copilot suggest -t gh'
alias 'explain'='gh copilot explain'
```

---

## Comparison: Copilot vs Claude Code

Both tools are valuable. Here is when to use each.

### GitHub Copilot Strengths

| Strength | Details |
|----------|---------|
| **Inline completions** | Real-time ghost text as you type — Claude Code does not do this |
| **IDE integration** | Deeply integrated into VS Code, IntelliJ, Neovim |
| **PR workflow** | Summaries, reviews, Workspace — native GitHub integration |
| **Low friction** | Always on, no context switching |
| **Speed** | Completions appear in milliseconds |
| **Code review** | Automated PR reviews on GitHub.com |

### Claude Code Strengths

| Strength | Details |
|----------|---------|
| **Complex reasoning** | Better at multi-step, cross-file architectural changes |
| **Codebase understanding** | Searches and reads your entire codebase autonomously |
| **Multi-file edits** | Creates, modifies, deletes files across the project in one shot |
| **Terminal integration** | Runs commands, interprets output, iterates on failures |
| **Long conversations** | Maintains deep context across a long planning session |
| **Autonomous execution** | Can build features end-to-end: code, test, commit |

### Decision Matrix

| Task | Use Copilot | Use Claude Code |
|------|-------------|-----------------|
| Writing a new method | Tab completions | Overkill |
| Quick inline edit | `Ctrl+I` | Overkill |
| Explain a method | Copilot Chat `/explain` | Either works |
| Generate unit tests | Copilot Chat `/tests` | Either works |
| Refactor a single file | Copilot Chat | Either works |
| Refactor across 10 files | Copilot Edits (limited) | Claude Code (better) |
| Build a new feature end-to-end | Copilot Workspace | Claude Code |
| Debug a complex production issue | Copilot Chat | Claude Code (reads logs, traces) |
| Create a new microservice skeleton | Copilot Chat | Claude Code (creates all files) |
| Write a Dockerfile | Tab completions | Overkill |
| Design system architecture | Copilot Chat (basic) | Claude Code (deeper reasoning) |
| Review a PR | Copilot Review | Not applicable |
| CLI command help | `gh copilot suggest` | Not applicable |
| Commit and push changes | Manual | Claude Code handles it |
| Investigate a codebase you did not write | Copilot `@workspace` | Claude Code (more thorough) |

### The Combined Workflow

The best approach is to use both:

```
1. Start coding → Copilot completions (Tab)
2. Need a quick fix → Copilot inline chat (Ctrl+I)
3. Need tests → Copilot Chat (/tests)
4. Hit a complex problem → Switch to Claude Code for deep analysis
5. Need multi-file changes → Claude Code
6. Quick command lookup → gh copilot suggest
7. Ready to PR → Copilot for summary and review
```

### Cost Comparison

| Tool | Cost | Model |
|------|------|-------|
| Copilot Individual | $10/month | GPT-4o, Claude 3.5 Sonnet, others |
| Copilot Business | $19/user/month | Same + org features |
| Claude Code (Pro) | $20/month (Claude Pro subscription) | Claude 3.5 Sonnet, Claude 3 Opus |
| Claude Code (API) | Pay per token | Any Claude model |

---

## Try This Exercises

### Exercise 1: Full Morning Routine

1. Open your terminal and run `gh pr list --search "review-requested:@me"`
2. Pick one PR. Checkout and review it using Copilot Chat
3. Leave at least one review comment informed by Copilot's analysis
4. Check your issues: `gh issue list --assignee @me`

### Exercise 2: Feature Development Sprint

1. Pick a small feature (30-60 minutes of work)
2. Time yourself implementing it using the "Completions + Chat Combo" workflow
3. Track: how many suggestions you accepted, how many you rejected, how many times you used Chat
4. Compare with how long this would have taken without Copilot

### Exercise 3: Test Coverage Boost

1. Pick a service class with low test coverage
2. Use `/tests` to generate comprehensive tests
3. Review and fix the generated tests (they are never 100% correct)
4. Run coverage: `./mvnw test jacoco:report`
5. Track how much coverage improved

### Exercise 4: DevOps Day

1. Generate a Kubernetes manifest using Copilot Chat
2. Generate a Terraform module using completions
3. Use `gh copilot suggest` for 5 different CLI tasks
4. Generate a GitHub Actions workflow for a specific pipeline

### Exercise 5: Copilot vs Claude Code Challenge

1. Pick a task: "Add a caching layer to the product search endpoint"
2. Implement it first using Copilot only (completions + chat)
3. Revert the changes
4. Implement it again using Claude Code only
5. Compare: time, quality, number of iterations, correctness
6. Write down which tool was better for this specific task

### Exercise 6: Documentation Sprint

1. Pick your 3 most important service classes
2. Use Copilot to generate Javadoc for all public methods
3. Add OpenAPI annotations to your main REST controller
4. Generate a Mermaid architecture diagram using Chat
5. All in under 30 minutes

---

## Summary

This playbook gives you a structured approach to using Copilot throughout your entire workday. The key principles:

1. **Copilot completions for writing code** — Tab is your best friend
2. **Copilot Chat for understanding code** — `/explain`, `/fix`, `/tests`
3. **Copilot CLI for command-line tasks** — `??` alias for instant help
4. **Copilot for PRs** — summaries and reviews
5. **Claude Code for complex tasks** — multi-file, architecture, deep reasoning
6. **Always review generated code** — you own every line that ships

Master these patterns and you will be significantly more productive while maintaining code quality.
