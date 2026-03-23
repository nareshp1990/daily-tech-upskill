# Tutorial 08: Daily Workflow Playbook

A practical, action-oriented guide for using Claude Code throughout your workday as a senior backend developer.

## Table of Contents

1. [Morning Routine](#1-morning-routine)
2. [During Development](#2-during-development)
3. [Code Review Workflows](#3-code-review-workflows)
4. [System Design Discussions](#4-system-design-discussions)
5. [DevOps Tasks](#5-devops-tasks)
6. [Database Work](#6-database-work)
7. [API Design and Documentation](#7-api-design-and-documentation)
8. [Incident Response and Debugging](#8-incident-response-and-debugging)
9. [Quick Reference Cheat Sheet](#9-quick-reference-cheat-sheet)
10. [Power User Tips and Tricks](#10-power-user-tips-and-tricks)

---

## 1. Morning Routine

### Review Open PRs

```
> Show me all modified files and a summary of the changes in the
> current branch compared to main.

# Or review someone else's PR:
> Review PR #42. Focus on the service layer changes and any
> potential breaking changes to the API contract.
```

### Check What You Did Yesterday

```
> Show me my git commits from yesterday and summarise what I worked on.
> Format as bullet points for the standup.
```

### Assess Today's Work

```
> What TODOs and FIXMEs exist in the codebase? Prioritise the
> critical ones. Also check if any TODO references a JIRA ticket.
```

### Quick Codebase Health Check

```
> Are there any compilation warnings in the project?
> Run: mvn compile -q

> Check if all tests pass:
> Run: mvn test -q
```

---

## 2. During Development

### Writing New Code

**Create a new feature:**
```
> Create a notification preferences API:
> - Entity: NotificationPreference (userId, channel, enabled, frequency)
> - Repository with custom query for active preferences by user
> - Service with CRUD + bulk update
> - REST controller with validation
> - Unit and integration tests
> Follow existing patterns in the codebase.
```

**Implement a specific method:**
```
> Implement the calculateShippingCost method in OrderService.
> Rules:
> - Free shipping for orders over $100
> - Flat rate $5.99 for domestic
> - Weight-based for international ($2 per kg, minimum $15)
> - Express shipping adds 50% surcharge
> Include edge case handling and tests.
```

### Debugging

**Fix a specific error:**
```
> I'm getting this error when processing large orders:
>
> java.lang.OutOfMemoryError: Java heap space
>   at com.example.service.OrderExportService.exportToExcel(OrderExportService.java:87)
>
> The method loads all order items into a list. Orders can have up to 50K items.
> Fix it using a streaming approach.
```

**Trace a bug:**
```
> Users report that order totals are sometimes wrong. The discount
> calculation in OrderService.applyDiscount seems correct to me.
> Trace through the code path from the controller to the final
> total calculation and find where the bug might be.
```

**Fix failing tests:**
```
> Run mvn test and fix any failures.
```

Or more specifically:
```
> The test OrderServiceTest.shouldApplyBulkDiscount fails
> intermittently. Find the flaky test issue and fix it.
```

### Refactoring

**Extract a pattern:**
```
> I notice we have similar validation logic in OrderController,
> PaymentController, and ShippingController. Extract a common
> validation utility or use a shared approach. Show me the plan first.
```

**Modernise code:**
```
> Refactor this class to use Java 21 features:
> - Replace anonymous classes with lambdas
> - Use records where appropriate
> - Use pattern matching for instanceof
> - Use switch expressions
> - Use text blocks for multi-line strings
> Keep all tests passing.
```

**Clean up dead code:**
```
> Find all unused methods, classes, and imports in the
> com.example.order package. List them and then remove them.
> Run tests after removal to ensure nothing breaks.
```

### Writing Tests

**Unit tests:**
```
> Write unit tests for PaymentService. Cover:
> - Successful payment processing
> - Insufficient funds scenario
> - Payment gateway timeout
> - Duplicate payment detection
> - Concurrent payment requests for the same order
> Use JUnit 5, Mockito, and AssertJ.
```

**Integration tests:**
```
> Write an integration test for the full order creation flow:
> 1. Create a customer
> 2. Add items to cart
> 3. Submit order
> 4. Verify order status and inventory update
> Use Testcontainers for PostgreSQL and Kafka.
```

**Test edge cases:**
```
> What edge cases are missing from OrderServiceTest?
> Generate tests for any gaps you find.
```

---

## 3. Code Review Workflows

### Self-Review Before Pushing

```
> /review

> I'm about to push these changes. Give me a brutally honest review.
> Flag anything that would fail a senior engineer's code review.
```

### Review a PR Diff

```
> Review the diff between my branch and main. Focus on:
> 1. Correctness of the business logic
> 2. Thread safety in the service layer
> 3. API backward compatibility
> 4. Missing error handling
> 5. Test adequacy
```

### Review Specific Concerns

```
> Review the database migration files I've added.
> Check for: data loss risks, missing rollback, performance impact
> on the orders table (5M rows), and locking concerns.
```

```
> Review the Kafka consumer changes for exactly-once semantics.
> Are there any scenarios where we could process a message twice
> or lose a message?
```

### Commit with Context

```
> /commit

# Or with guidance:
> Stage the changes and create a commit. The message should reference
> JIRA-567 and explain why we changed the retry strategy.
```

---

## 4. System Design Discussions

### Design a New System

```
> I need to design a real-time notification system with these requirements:
> - Support email, SMS, push notifications, and in-app
> - Handle 50K notifications/minute at peak
> - Support user preferences (channel, frequency, quiet hours)
> - Template-based content with variable substitution
> - Delivery tracking and retry for failures
> - Priority levels (critical alerts bypass quiet hours)
>
> Tech stack: Spring Boot, Kafka, PostgreSQL, Azure.
>
> Design the architecture with:
> 1. Component diagram
> 2. Data model
> 3. Kafka topic design
> 4. Key API endpoints
> 5. Failure handling strategy
```

### Evaluate Trade-offs

```
> We're deciding between these approaches for the order processing pipeline:
>
> Option A: Synchronous processing with database transactions
> Option B: Event-driven with Kafka and eventual consistency
> Option C: Saga pattern with orchestrator
>
> Requirements: 10K orders/minute, involves payment + inventory + shipping.
> What are the trade-offs? Which do you recommend and why?
```

### Scale an Existing System

```
> Our order service currently handles 1K requests/second on 3 pods.
> We need to scale to 10K requests/second.
> Current bottleneck: PostgreSQL (CPU at 80%, connection pool exhausted).
>
> What changes do we need? Consider:
> - Read replicas
> - Caching strategy (Redis)
> - Connection pooling tuning
> - Query optimisation
> - Sharding considerations
> - CDN for static content
```

### API Contract Design

```
> Design the API contract for a multi-tenant SaaS billing service.
> Requirements:
> - Subscription plans (monthly, yearly)
> - Usage-based pricing add-ons
> - Invoice generation
> - Payment method management
> - Webhook callbacks for payment events
>
> Show me the OpenAPI spec with example requests/responses.
```

---

## 5. DevOps Tasks

### Dockerfile Review and Optimisation

```
> Review our Dockerfile and optimise it:
> - Reduce image size
> - Improve build cache efficiency
> - Fix any security issues
> - Use multi-stage build if not already
> - Ensure the JVM is tuned for containers

> Here's our current Dockerfile: [read it from the project]
```

### Kubernetes Manifest Review

```
> Review our K8s deployment manifest for the order-service.
> Check for:
> - Appropriate resource limits/requests
> - Health check configuration
> - Security context
> - HPA configuration
> - Pod disruption budget
> - Rolling update strategy
```

**Generate new K8s resources:**
```
> Create a K8s deployment, service, HPA, and PDB for a new
> microservice called notification-service. Base it on our
> existing order-service manifests but adjust:
> - Lower CPU (500m request, 1000m limit)
> - Higher memory (512Mi request, 1Gi limit)
> - Scale 2-10 pods based on Kafka lag metric
```

### Terraform Work

```
> Create a Terraform module for an Azure Service Bus namespace
> with two queues (orders, notifications) and dead letter queues.
> Follow our existing Terraform conventions.

> Review this Terraform plan output. Any concerns about cost,
> security, or availability?
```

### GitHub Actions

```
> Our CI/CD pipeline takes 12 minutes. Analyse the workflow
> file and suggest optimisations:
> - Parallelisation opportunities
> - Caching improvements
> - Unnecessary steps to remove
> - Better matrix strategy

> Create a new GitHub Actions workflow for deploying to
> staging when a PR is merged to the develop branch.
```

### Docker Compose for Local Development

```
> Create a docker-compose.yml for local development that includes:
> - PostgreSQL 16 with init script
> - Kafka (using Redpanda for simplicity)
> - Redis
> - Our order-service (build from Dockerfile)
> - Prometheus + Grafana for local monitoring
> Include health checks and proper networking.
```

---

## 6. Database Work

### Query Optimisation

```
> This query takes 3 seconds on a table with 5M rows:
>
> SELECT o.*, c.name, c.email,
>        COUNT(oi.id) as item_count,
>        SUM(oi.price * oi.quantity) as total
> FROM orders o
> JOIN customers c ON o.customer_id = c.id
> JOIN order_items oi ON oi.order_id = o.id
> WHERE o.status IN ('PENDING', 'PROCESSING')
>   AND o.created_at > NOW() - INTERVAL '30 days'
> GROUP BY o.id, c.id
> ORDER BY o.created_at DESC
> LIMIT 50
>
> Current indexes: orders(id), orders(customer_id), order_items(order_id)
>
> Suggest optimisations: indexes, query rewriting, or schema changes.
```

### Migration Generation

```
> Create a Flyway migration to:
> 1. Add a 'priority' column to orders (enum: LOW, MEDIUM, HIGH, CRITICAL)
> 2. Default to MEDIUM for existing rows
> 3. Add an index on (status, priority, created_at)
> 4. This table has 5M rows — make the migration non-blocking.
```

### Schema Review

```
> Review our database schema for the order domain:
> - orders, order_items, order_events, customers, addresses
>
> Check for:
> - Missing indexes for common query patterns
> - Normalisation issues
> - Potential performance bottlenecks at scale
> - Missing constraints
> - Data type appropriateness
```

### JPA / Hibernate Optimisation

```
> Find all N+1 query problems in our JPA entities.
> Check @OneToMany, @ManyToOne relationships for:
> - Eager loading that should be lazy
> - Lazy loading that causes N+1 in loops
> - Missing @BatchSize or @Fetch annotations
> - Missing @EntityGraph usage in repository methods
```

---

## 7. API Design and Documentation

### Design a New API

```
> Design a REST API for order management following our conventions:
> - Prefix: /api/v1/orders
> - Support: CRUD, search, status transitions, bulk operations
> - Use HATEOAS links
> - Use RFC 7807 for errors
> - Support pagination, sorting, filtering
> - Include rate limit headers
> Show the OpenAPI 3.0 spec.
```

### Generate Documentation

```
> Generate API documentation for all controllers in this project.
> Include:
> - Endpoint URL and method
> - Request/response examples with realistic data
> - Authentication requirements
> - Error responses
> - Rate limits
> Format as Markdown.
```

### API Review

```
> Review our public API for backward compatibility issues.
> Check for:
> - Required fields that were previously optional
> - Changed field types or names
> - Removed endpoints
> - Changed error codes
> - Modified pagination behaviour
```

---

## 8. Incident Response and Debugging

### Analyse Errors

```
> Here's a stack trace from production:

[paste full stack trace]

> What's the root cause? What's the impact? How do we fix it?
> Also suggest what monitoring we should add to catch this earlier.
```

### Log Analysis

```bash
# Pipe logs directly to Claude Code
cat /tmp/order-service-errors.log | claude -p \
  "Analyse these error logs. Find patterns, root causes, and suggest fixes. Group by error type and frequency."
```

### Performance Investigation

```
> Our P99 latency for POST /api/v1/orders spiked from 200ms to 2s
> in the last hour. The change happened at 14:30 UTC.
>
> Here's what we know:
> - No deployments since yesterday
> - Database CPU is normal (30%)
> - Kafka consumer lag is increasing
> - Pod count is normal (3 replicas)
>
> Help me build a debugging checklist and investigate the code paths
> that could cause this.
```

### Post-Incident Review

```
> We had a 30-minute outage caused by a database connection pool
> exhaustion. Help me write a post-mortem that covers:
> - Timeline
> - Root cause analysis
> - Impact
> - What went well
> - What went poorly
> - Action items (with owners and deadlines)
>
> Here are the details: [describe the incident]
```

---

## 9. Quick Reference Cheat Sheet

### Essential Slash Commands

```
/help           — Show all commands
/clear          — Reset conversation
/compact        — Compress context (save tokens)
/cost           — Show session cost and token usage
/review         — Review code changes
/commit         — Create a git commit
/init           — Generate CLAUDE.md for the project
/doctor         — Diagnose installation issues
```

### Common One-Liners

```bash
# Quick question without entering interactive mode
claude "What Spring profiles are defined in this project?"

# Review a file
claude "Review src/main/java/com/example/OrderService.java for bugs"

# Pipe errors for analysis
mvn test 2>&1 | claude -p "Fix these test failures"

# Git diff review
git diff main | claude -p "Review these changes"

# Generate from template
claude "Create a new service class following our CLAUDE.md conventions for a 'Payment' domain"
```

### Flags Reference

```bash
claude                      # Interactive mode
claude "question"           # Single-shot mode
claude -p "prompt"          # Headless/pipe mode (non-interactive)
claude -ae                  # Auto-accept edits
claude -aa                  # Auto-accept all (edits + commands)
claude --output-format json # JSON output (for scripting)
claude --output-format text # Plain text output
claude --allowedTools "Read,Glob,Grep"  # Restrict tools
claude --model claude-sonnet-4-0520     # Specify model
```

### Daily Task Quick Commands

| Task | Command |
|------|---------|
| Start coding session | `cd project && claude` |
| Quick code review | `claude "Review the changes in this branch vs main"` |
| Fix build | `mvn compile 2>&1 \| claude -p "Fix these errors"` |
| Fix tests | `mvn test 2>&1 \| claude -p "Fix failing tests"` |
| Commit | `/commit` (in interactive mode) |
| Create PR | `"Create a PR for this branch targeting main"` |
| Check cost | `/cost` |
| Reset context | `/compact` or `/clear` |
| Explore codebase | `"Give me an overview of this project"` |
| Find pattern | `"Find all places where we handle authentication"` |

### Context Management

| Situation | Action |
|-----------|--------|
| Long conversation (30+ min) | Run `/compact` |
| Switching tasks | Run `/clear` |
| Claude forgets earlier context | Run `/compact` with custom focus |
| Context feels stale | Start a new session |
| Large codebase | Ask focused questions, not "read everything" |

---

## 10. Power User Tips and Tricks

### Tip 1: Use Claude Code as a Pair Programmer

Instead of asking Claude Code to write everything, describe your approach and ask for feedback:

```
> I'm thinking of implementing the retry logic using Resilience4j
> with an exponential backoff starting at 1 second, max 3 retries.
> The circuit breaker should open after 5 consecutive failures.
>
> Does this make sense for our Kafka consumer that processes payments?
> Any concerns with these parameters?
```

### Tip 2: Batch Related Changes

Instead of:
```
> Add validation to OrderController
> [wait for response]
> Now add validation to PaymentController
> [wait for response]
> Now add validation to ShippingController
```

Do:
```
> Add @Valid annotation and validation constraints to all request
> DTOs across OrderController, PaymentController, and ShippingController.
> Use consistent validation rules: all IDs must be positive, names
> max 100 chars, email must be valid format.
```

### Tip 3: Reference Specific Files and Lines

```
> In OrderService.java, the processOrder method (around line 87)
> has a race condition. Two threads can read the same inventory
> count and both succeed. Add optimistic locking using @Version.
```

### Tip 4: Ask for Alternatives

```
> Show me three different approaches to implement distributed
> rate limiting. For each, show a code sketch and list pros/cons.
> Then recommend which one for our scale (5K requests/second).
```

### Tip 5: Use Claude Code for Code Archaeology

```
> Why was the OrderValidator class implemented this way?
> Look at the git history, related test files, and any comments.
> Is there a simpler approach that preserves the same behaviour?
```

### Tip 6: Pre-Load Context Efficiently

At the start of a session, give Claude Code targeted context:

```
> I'm working on the payment processing module today.
> The key files are:
> - PaymentService.java
> - PaymentController.java
> - PaymentGatewayClient.java
> - PaymentServiceTest.java
>
> Read these files so you understand the current state.
> Then I'll ask you to make some changes.
```

### Tip 7: Use Claude for Learning

```
> Explain how Spring Boot's auto-configuration works internally.
> Show me the actual source code flow from @SpringBootApplication
> to a bean being created. Use our project as a concrete example.
```

### Tip 8: Create Reusable Custom Commands

Set up custom slash commands for your most common tasks (see Tutorial 04):

```
/review-security     — Security-focused code review
/generate-dto        — Generate DTOs from entity
/deploy-check        — Pre-deployment checklist
/morning-standup     — Prepare standup notes
```

### Tip 9: Measure and Optimise Your Usage

Track your Claude Code costs weekly:
```
> /cost

# Keep a log:
# Week 1: $12 (mostly Opus, lots of architecture discussions)
# Week 2: $8 (switched to Sonnet for routine coding)
# Week 3: $5 (better prompts, more focused questions)
```

### Tip 10: Know When NOT to Use Claude Code

Sometimes traditional tools are faster:

| Task | Better Tool |
|------|-------------|
| Simple rename refactor | IntelliJ refactor (Shift+F6) |
| Find a specific string | `grep` or IntelliJ search (Ctrl+Shift+F) |
| Quick variable rename | IDE rename |
| Format code | IDE auto-format or Prettier |
| Git operations you know | Direct git commands |
| Checking build status | `mvn compile` directly |

Use Claude Code for tasks that benefit from reasoning, multi-file understanding, or code generation. Use traditional tools for mechanical, well-defined operations.

---

## Daily Workflow Summary

```
Morning (15 min):
├── Check PRs to review
├── Run tests / build
└── Plan today's work

Development (main work hours):
├── Write features with Claude Code
├── Debug with Claude Code + IDE debugger
├── Write tests with Claude Code
├── /compact periodically
└── /review before committing

Review & Commit (end of task):
├── /review (self-review)
├── /commit (smart commit message)
└── Create PR when ready

DevOps (as needed):
├── Dockerfile / K8s review
├── Terraform changes
├── CI/CD pipeline updates
└── Database migrations

End of Day:
├── /cost (check daily spend)
├── Note any learnings
└── Push work in progress
```

---

**This playbook is a living document. Update it as you discover new workflows and patterns that work for your team.**

**Start of the series:** [README — Index and Learning Path](README.md)
