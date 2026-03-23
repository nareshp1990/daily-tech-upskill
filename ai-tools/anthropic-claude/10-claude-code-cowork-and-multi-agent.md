# Tutorial 10: Claude Code Cowork & Multi-Agent Collaboration

## Table of Contents

1. [Overview: What Is Claude Code Cowork?](#1-overview-what-is-claude-code-cowork)
2. [Parallel Agents (Subagents)](#2-parallel-agents-subagents)
3. [Plan Mode](#3-plan-mode)
4. [Task Management](#4-task-management)
5. [Headless Mode & Automation](#5-headless-mode--automation)
6. [Multi-Session Workflows](#6-multi-session-workflows)
7. [Coworking Patterns for Java/Spring Boot](#7-coworking-patterns-for-javaspring-boot)
8. [Git Worktree Integration](#8-git-worktree-integration)
9. [Best Practices & Anti-Patterns](#9-best-practices--anti-patterns)
10. [Try This Exercises](#10-try-this-exercises)
11. [Tips and Gotchas](#11-tips-and-gotchas)

---

## 1. Overview: What Is Claude Code Cowork?

### The Single-Agent Bottleneck

When you work with Claude Code in a standard session, you are talking to a single agent. That agent reads files, reasons about your request, writes code, and runs commands — all sequentially. For small tasks, this is perfectly fine. But for real-world backend work — migrating a monolith to microservices, implementing a feature across six Spring Boot modules, or reviewing a 2,000-line PR — sequential execution becomes the bottleneck.

### Multi-Agent Collaboration

Claude Code's cowork capabilities break that bottleneck by introducing **multiple agents that operate in parallel** within a single development workflow. Think of it like this:

| Single-Agent | Multi-Agent (Cowork) |
|---|---|
| One thread of execution | Multiple parallel threads |
| Reads one file at a time | Explores many files simultaneously |
| Plans and implements in the same context | Separate planning and implementation phases |
| Context window fills up quickly | Each agent has its own context window |
| One task at a time | Multiple tasks running concurrently |

### How It Differs from Running Multiple Terminals

You could open three terminal windows and run three separate `claude` sessions. That works, but cowork goes further:

- **Coordinated subagents**: Claude Code spawns child agents that report back to the parent, enabling a single orchestrated workflow.
- **Worktree isolation**: Parallel agents can edit files safely via git worktrees, avoiding merge conflicts.
- **Task tracking**: Work is broken into trackable pieces with explicit state (pending, in-progress, completed).
- **Shared context**: Agents within a cowork session share the same CLAUDE.md, project conventions, and high-level goals.

### When Multi-Agent Matters

For a senior backend engineer working on production Java/Spring Boot systems, multi-agent collaboration shines in these scenarios:

- **Large-scale refactoring**: Renaming a domain concept across 15 microservices.
- **Feature development**: One agent builds the service layer while another writes tests.
- **Code review + fix**: One agent reviews, another implements the fixes.
- **Investigation**: Exploring multiple modules simultaneously to understand a cross-cutting concern.
- **Migration**: Planning in one agent, executing in another, validating in a third.

---

## 2. Parallel Agents (Subagents)

### How Subagents Work Internally

When Claude Code determines that a task benefits from parallel execution, it spawns **subagents** — lightweight child agents that each get their own context window and tool access. The parent agent orchestrates, delegates, and synthesises results.

```
┌──────────────────────────────┐
│        Parent Agent          │
│  (orchestrates, delegates)   │
├──────┬──────────┬────────────┤
│      │          │            │
▼      ▼          ▼            ▼
┌────┐ ┌────┐  ┌────┐     ┌────┐
│Sub1│ │Sub2│  │Sub3│     │Sub4│
│Read│ │Read│  │Edit│     │Test│
│code│ │docs│  │file│     │run │
└────┘ └────┘  └────┘     └────┘
  │      │        │          │
  └──────┴────────┴──────────┘
           │
     ┌─────▼─────┐
     │  Parent    │
     │ synthesises│
     │  results   │
     └────────────┘
```

Each subagent:
- Has its own isolated context window (does not consume the parent's context).
- Can read files, search code, and run commands.
- Reports its findings back to the parent when complete.
- Cannot directly communicate with other subagents — all coordination goes through the parent.

### Types of Subagents

Claude Code uses different subagent strategies depending on the task:

| Subagent Type | Purpose | Typical Operations |
|---|---|---|
| **Explore** | Investigate code, find patterns, read documentation | `Read`, `Grep`, `Glob`, `Bash` (read-only) |
| **Plan** | Analyse and create implementation plans | `Read`, `Grep`, `Glob` (no edits) |
| **Edit** | Make targeted code changes | `Read`, `Edit`, `Write` |
| **Test** | Run tests and report results | `Bash` (test commands) |
| **General-purpose** | Any combination of the above | All tools available |

### When Claude Automatically Uses Subagents

Claude Code may spawn subagents on its own when it recognises that a task is parallelisable. Common triggers:

- **Multi-file investigation**: "Find all places where we use `@Transactional(readOnly = true)` and check if any of them modify state."
- **Broad search queries**: "How does the order lifecycle work from creation to fulfilment?"
- **Multiple independent subtasks**: "Add logging to OrderService, PaymentService, and InventoryService."

### Requesting Parallel Execution

You can explicitly request parallel work:

```
> Research these three things in parallel:
> 1. How OrderService handles cancellation
> 2. What events PaymentService publishes
> 3. How InventoryService manages reservations
```

The phrase "in parallel" signals Claude Code to spawn subagents rather than investigating sequentially.

Another effective pattern:

```
> Do these in parallel:
> - Add the new "priority" field to OrderEntity, OrderDTO, and OrderMapper
> - Write unit tests for the priority-based sorting logic
> - Update the OpenAPI spec with the new field
```

### Background vs Foreground Agents

**Foreground agents** block the parent — the parent waits for results before continuing. This is the default for subagent work.

**Background agents** run asynchronously. You can continue chatting with the main agent while background work proceeds. This is useful for:

- Long-running test suites
- Large file searches across the codebase
- Code generation that takes time

```
> Run the full integration test suite in the background while we
> work on the next feature.
```

### Worktree Isolation for Safe Parallel Edits

When multiple agents need to edit files simultaneously, Claude Code can use **git worktrees** to isolate changes. Each agent operates in its own worktree — a separate checkout of the same repository — so there is no risk of agents overwriting each other's work.

```
Main worktree:    /project/order-service/       (Agent 1: editing service layer)
Worktree 2:       /project/.worktrees/agent-2/  (Agent 2: editing test layer)
Worktree 3:       /project/.worktrees/agent-3/  (Agent 3: editing config)
```

Changes are merged back after each agent completes its work.

### Practical Examples

**Example 1: Researching Multiple Files Simultaneously**

```
> I'm seeing intermittent 409 Conflict errors in our order creation flow.
> In parallel, check:
> 1. The OrderService.createOrder() method for race conditions
> 2. The database schema for unique constraints on the orders table
> 3. Recent Kafka consumer logs for duplicate message processing
> 4. The idempotency key implementation in OrderController
```

Claude Code spawns four subagents, each investigating one area, and then synthesises a root cause analysis.

**Example 2: Running Tests While Editing**

```
> Fix the N+1 query issue in OrderRepository.findAllWithItems().
> While you're making the fix, run the existing OrderRepositoryTest
> in the background so we know the current baseline.
```

The parent agent edits the repository while a background agent runs the test suite, giving you before-and-after results.

**Example 3: Parallel Code Generation**

```
> Generate these three Spring Boot components in parallel:
> 1. NotificationEvent (domain event with type, recipient, channel, payload)
> 2. NotificationService (sends via email/SMS/push based on channel)
> 3. NotificationController (REST API for notification preferences)
> Follow existing patterns in the codebase.
```

---

## 3. Plan Mode

### What Is Plan Mode?

Plan mode is a **read-only collaboration mode** where Claude Code analyses your codebase and creates a detailed implementation plan without making any changes. It is the "think before you act" phase.

### Entering Plan Mode

There are two ways to enter plan mode:

1. **Slash command**: Type `/plan` followed by your task description.
2. **Keyboard shortcut**: Press `Shift+Tab` to toggle the current prompt into plan mode.

```
> /plan Refactor OrderService to use the Strategy pattern for different
> order types (standard, subscription, bulk, pre-order)
```

### How Plan Mode Works

In plan mode, Claude Code:

- **Reads** files, searches code, analyses dependencies — all the exploratory work.
- **Creates** a structured implementation plan with specific files to change, code patterns to follow, and the order of operations.
- **Does not** edit files, write code, or run destructive commands.

This gives you a chance to review the approach, catch misalignment, and steer the direction before any code is written.

### Creating Implementation Plans

A good plan from Claude Code includes:

```
## Implementation Plan: Strategy Pattern for Order Types

### Current State Analysis
- OrderService.processOrder() has a 200-line switch statement
- Four order types handled inline: standard, subscription, bulk, pre-order
- 47 references to order type checks across service layer

### Proposed Changes

#### Step 1: Define the Strategy Interface
- Create `OrderProcessingStrategy` interface in `com.example.order.strategy`
- Methods: `process(Order)`, `validate(Order)`, `calculatePrice(Order)`
- Supports: OrderType enum matching

#### Step 2: Implement Concrete Strategies
- `StandardOrderStrategy` — current default logic
- `SubscriptionOrderStrategy` — recurring billing, period tracking
- `BulkOrderStrategy` — volume discounts, split shipping
- `PreOrderStrategy` — reservation, release-date checks

#### Step 3: Create Strategy Registry
- `OrderStrategyRegistry` — Spring-managed, auto-discovers strategies via @Component
- Uses `Map<OrderType, OrderProcessingStrategy>` for O(1) lookup

#### Step 4: Refactor OrderService
- Remove switch statement
- Inject `OrderStrategyRegistry`
- Delegate to appropriate strategy

#### Step 5: Update Tests
- Add unit tests per strategy
- Refactor existing OrderServiceTest
- Add integration test for strategy resolution

### Files Affected
| File | Action | Risk |
|------|--------|------|
| OrderProcessingStrategy.java | Create | Low |
| StandardOrderStrategy.java | Create | Low |
| SubscriptionOrderStrategy.java | Create | Low |
| BulkOrderStrategy.java | Create | Low |
| PreOrderStrategy.java | Create | Low |
| OrderStrategyRegistry.java | Create | Low |
| OrderService.java | Modify | Medium |
| OrderServiceTest.java | Modify | Medium |
| OrderIntegrationTest.java | Modify | Low |

### Estimated Effort
~45 minutes with Claude Code assistance
```

### Reviewing and Iterating on Plans

Once Claude Code presents the plan, you can iterate:

```
> Good plan, but I want to add a few things:
> 1. Each strategy should also handle event publishing (different events per type)
> 2. Use @Order annotation to define a fallback chain if a strategy fails
> 3. Add a "custom" type that loads strategy from a config file
> Update the plan.
```

This back-and-forth happens entirely in read-only mode. No code has been written, no files touched.

### Converting Plans to Tasks

Once you are satisfied with the plan, you can execute it:

```
> The plan looks good. Implement Step 1 and Step 2 now.
```

Or convert the entire plan into sequential tasks:

```
> Implement the full plan, step by step. Pause after each step so I can review.
```

### When to Use Plan Mode vs Direct Execution

| Scenario | Use Plan Mode? |
|---|---|
| Adding a simple field to an entity | No — just do it |
| Refactoring a core service | Yes — plan first |
| Bug fix with known root cause | No — just fix it |
| Cross-module feature implementation | Yes — plan first |
| Database migration | Yes — always plan |
| Writing tests for existing code | No — usually straightforward |
| Introducing a new design pattern | Yes — plan first |
| Performance optimisation | Yes — plan the investigation and fixes |

**Rule of thumb**: If the change touches more than 5 files or involves architectural decisions, use plan mode.

### Practical Example: Planning a Spring Boot Microservice Refactor

```
> /plan We need to extract the notification logic from order-service into
> a standalone notification-service. Currently, OrderService directly calls
> EmailSender, SmsSender, and PushNotificationSender. We want:
> 1. A new notification-service Spring Boot app
> 2. order-service publishes NotificationRequestedEvent to Kafka
> 3. notification-service consumes the event and dispatches via the right channel
> 4. notification-service has its own Postgres DB for templates and history
> 5. Backwards-compatible rollout (feature flag to toggle old vs new path)
```

Claude Code will explore the existing codebase, identify all notification touchpoints, map out the Kafka event schema, and produce a step-by-step migration plan — all without changing a single line of code.

---

## 4. Task Management

### Creating Tasks

For complex, multi-step work, Claude Code can break the work into discrete, trackable tasks. You can request this explicitly:

```
> Break this feature into tasks:
> Implement a scheduled job that:
> 1. Finds all orders stuck in PROCESSING state for more than 1 hour
> 2. Retries payment processing up to 3 times
> 3. Moves to FAILED state after 3 retries
> 4. Sends a notification to the customer
> 5. Publishes an OrderFailedEvent for analytics
```

Claude Code creates a task list and works through it:

```
Tasks:
  [x] 1. Create StuckOrderScheduler with @Scheduled cron job
  [x] 2. Add findStuckOrders query to OrderRepository
  [>] 3. Implement retry logic in OrderRetryService
  [ ] 4. Add notification dispatch on final failure
  [ ] 5. Publish OrderFailedEvent
  [ ] 6. Write unit tests for retry logic
  [ ] 7. Write integration test for full flow
```

### Task States

| State | Symbol | Meaning |
|---|---|---|
| Pending | `[ ]` | Not yet started |
| In Progress | `[>]` | Currently being worked on |
| Completed | `[x]` | Done and verified |
| Blocked | `[!]` | Cannot proceed without input |

### Tracking Progress on Multi-Step Work

As Claude Code works through tasks, it reports progress. You can ask for a status update at any time:

```
> What's the status of our tasks?
```

And you will get a summary showing which tasks are complete, which is in progress, and what remains.

### Using Tasks for Complex Feature Implementation

Tasks are especially useful when combined with plan mode:

```
Step 1: Plan
> /plan Implement order export feature (CSV and PDF formats)

Step 2: Review and approve plan

Step 3: Execute as tasks
> Implement the plan as tasks. Work through each task and show me
> the results before moving to the next one.
```

### Practical Example: Implementing a New API Endpoint with Tests

```
> Create tasks for adding a GET /api/v1/orders/export endpoint that:
> - Accepts format=csv|pdf query parameter
> - Accepts date range filter (from, to)
> - Streams large result sets (don't load all into memory)
> - Requires ROLE_ADMIN authority
> - Has rate limiting (10 requests per minute)
>
> Work through each task.
```

Claude Code breaks this into granular tasks:

```
Tasks:
  [x] 1. Create OrderExportRequest DTO with validation
  [x] 2. Create OrderExportService interface + CsvExportStrategy
  [x] 3. Create PdfExportStrategy using iText
  [x] 4. Implement streaming with StreamingResponseBody
  [>] 5. Add OrderExportController with @PreAuthorize
  [ ] 6. Add rate limiting with Bucket4j
  [ ] 7. Write unit tests for export strategies
  [ ] 8. Write integration test for the endpoint
  [ ] 9. Update OpenAPI annotations
```

---

## 5. Headless Mode & Automation

### Running Claude Code Headlessly

Headless mode runs Claude Code non-interactively, making it a powerful building block for automated multi-agent workflows.

```bash
# Simple one-shot task
claude -p "List all Spring Boot @RestController classes and their base paths"

# With output format
claude -p "List all TODO comments in the codebase" --output-format json

# With specific tools allowed
claude -p "Review OrderService.java for potential bugs" \
  --allowedTools "Read,Grep,Glob"
```

### Piping Input/Output

Headless mode integrates with Unix pipes, enabling composition:

```bash
# Pipe a diff for review
git diff main..feature/order-export | claude -p "Review this diff for:
1. Spring Boot best practice violations
2. Missing error handling
3. Security concerns
4. Performance issues"

# Pipe test output for analysis
./gradlew test 2>&1 | claude -p "Analyse these test results.
Summarise failures and suggest fixes."

# Generate a changelog from commits
git log --oneline v2.3.0..HEAD | claude -p "Generate a user-facing
changelog from these commits. Group by: features, fixes, improvements."
```

### Using in Scripts and CI/CD

**Automated Code Review on PR**:

```bash
#!/bin/bash
# review-pr.sh — runs as a GitHub Actions step

PR_DIFF=$(gh pr diff "$PR_NUMBER")

REVIEW=$(echo "$PR_DIFF" | claude -p "You are a senior Java code reviewer.
Review this pull request diff for:
1. Spring Boot anti-patterns
2. Missing @Transactional annotations where needed
3. N+1 query risks in JPA relationships
4. Missing input validation
5. Hardcoded values that should be in application.yml

Format: markdown with severity (critical/warning/info) for each finding." \
  --allowedTools "Read,Grep,Glob")

gh pr comment "$PR_NUMBER" --body "$REVIEW"
```

**Batch Test Generation**:

```bash
#!/bin/bash
# generate-tests.sh — generate tests for all services missing coverage

for SERVICE in $(find . -name "*Service.java" -not -name "*Test*"); do
  TEST_FILE="${SERVICE%.java}Test.java"
  if [ ! -f "$TEST_FILE" ]; then
    echo "Generating tests for: $SERVICE"
    claude -p "Generate comprehensive unit tests for $SERVICE.
    Use JUnit 5, Mockito, AssertJ.
    Follow existing test patterns in the project.
    Cover happy paths, edge cases, and error scenarios." \
      --allowedTools "Read,Grep,Glob,Write"
  fi
done
```

### Combining with Parallel Execution

Run multiple headless agents in parallel using shell job control:

```bash
#!/bin/bash
# parallel-review.sh — review multiple services concurrently

SERVICES=("order-service" "payment-service" "inventory-service" "notification-service")

for SERVICE in "${SERVICES[@]}"; do
  (
    echo "Reviewing $SERVICE..."
    claude -p "Review all Java files in $SERVICE/src/main/java for:
    1. Unused imports
    2. Missing Javadoc on public methods
    3. Inconsistent exception handling
    Write a summary report." \
      --allowedTools "Read,Grep,Glob" \
      > "reports/${SERVICE}-review.md"
    echo "Done: $SERVICE"
  ) &
done

wait
echo "All reviews complete."
```

### Practical Examples

**Automated Migration — Javax to Jakarta Namespace**:

```bash
#!/bin/bash
# migrate-jakarta.sh

# Step 1: Plan (read-only agent)
PLAN=$(claude -p "Analyse the codebase and list every file that uses
javax.* imports that need migrating to jakarta.* for Spring Boot 3.
Output as a JSON array of file paths." \
  --allowedTools "Read,Grep,Glob" \
  --output-format json)

echo "$PLAN" > migration-plan.json

# Step 2: Execute migration per file (parallel agents)
echo "$PLAN" | jq -r '.[]' | while read FILE; do
  (
    claude -p "Migrate $FILE from javax.* to jakarta.* namespace.
    Only change import statements and annotations.
    Do not change any business logic." \
      --allowedTools "Read,Edit"
  ) &

  # Limit parallelism to 4 agents
  [ $(jobs -r | wc -l) -ge 4 ] && wait -n
done
wait

# Step 3: Validate (single agent)
claude -p "Run './gradlew compileJava' and report any compilation errors
from the javax-to-jakarta migration." \
  --allowedTools "Bash"
```

---

## 6. Multi-Session Workflows

### The Context Window Challenge

Every Claude Code session has a finite context window. For large tasks that span hours or days, you will exceed that window. Multi-session workflows let you break long-running work into manageable sessions.

### Using `/compact` to Manage Context

The `/compact` command summarises the current conversation, freeing context for new work while preserving the essential state.

**When to compact**:
- After completing a logical unit of work (e.g., finished the service layer, moving to tests).
- When Claude Code starts "forgetting" earlier instructions.
- When you see high token usage via `/cost`.

```
> We've finished implementing the OrderExportService and its tests.
> /compact

[Claude Code summarises: "Implemented OrderExportService with CSV and PDF
strategies, streaming support, and 12 unit tests all passing."]

> Now let's implement the controller layer.
```

### Continuing Work Across Sessions with Memory

Claude Code's memory system (`CLAUDE.md` and the memory files) persists across sessions. Use this to carry state:

```
> Remember: We are midway through the order export feature.
> Completed: service layer, strategies, tests.
> Remaining: controller, rate limiting, integration tests.
> The export service uses StreamingResponseBody for large results.
```

Next session:

```
> Let's continue the order export feature. Check your memory for
> where we left off.
```

### Splitting Large Tasks Across Sessions

**Session 1: Planning and Foundation**
```
> /plan Implement the full audit logging system
> [Review and approve plan]
> Implement the domain model and repository layer.
> /compact
```

**Session 2: Service and Event Layer**
```
> Continue the audit logging system. Check CLAUDE.md for the plan.
> Implement the service layer and Kafka event publishing.
> /compact
```

**Session 3: API and Integration**
```
> Continue the audit logging system.
> Implement the REST API and integration tests.
> Run the full test suite.
```

### CLAUDE.md as Shared Context Between Sessions

Your `CLAUDE.md` file acts as the persistent brain across sessions. For multi-session work, add project-level context:

```markdown
## In-Progress Work

### Audit Logging System (started 2026-03-20)
- **Status**: Service layer complete, controller pending
- **Branch**: feature/audit-logging
- **Design decisions**:
  - Using async Kafka publishing (not synchronous DB writes)
  - AuditEvent is immutable, stored in audit_events table
  - Retention: 90 days, then archived to S3
- **Files created so far**:
  - AuditEvent.java, AuditEventRepository.java
  - AuditService.java, AuditServiceTest.java
  - AuditEventPublisher.java (Kafka producer)
```

Any Claude Code session — interactive or headless — will read this file and pick up the context.

---

## 7. Coworking Patterns for Java/Spring Boot

### Pattern 1: Parallel Microservice Development

When you need to make the same type of change across multiple services:

```
> We need to add health check endpoints to all four microservices.
> Do these in parallel:
> 1. order-service: Add /actuator/health/readiness with DB and Kafka checks
> 2. payment-service: Add /actuator/health/readiness with DB and Stripe API checks
> 3. inventory-service: Add /actuator/health/readiness with DB and Redis checks
> 4. notification-service: Add /actuator/health/readiness with DB and SMTP checks
>
> Each service should follow the same HealthIndicator pattern but with
> service-specific checks.
```

Claude Code can use subagents to explore each service, understand its dependencies, and implement the appropriate health checks — all in parallel.

### Pattern 2: Test-Driven Development with Agents

Use two logical phases — one for writing tests, another for implementation:

```
Phase 1 — Test Agent:
> Write comprehensive unit tests for a PaymentRetryService that:
> - Retries failed payments up to 3 times with exponential backoff
> - Uses different retry strategies per payment method (card vs bank transfer)
> - Publishes PaymentRetriedEvent after each attempt
> - Publishes PaymentFailedEvent after max retries
> - Skips retry for non-retryable errors (invalid card, fraud)
>
> Write the tests first. Use descriptive method names. The implementation
> does not exist yet — use the expected API.

Phase 2 — Implementation Agent:
> Now implement PaymentRetryService to make all these tests pass.
> Run the tests after implementation.
```

### Pattern 3: Code Review Pipeline

A structured workflow using plan, implement, and review phases:

```
Step 1 — Explore:
> Explore the OrderController and identify all endpoints that are
> missing input validation, error handling, or proper HTTP status codes.

Step 2 — Plan:
> /plan Fix all the issues you found. Group by severity.

Step 3 — Implement:
> Implement the fixes for critical and high-severity issues.

Step 4 — Review:
> Review the changes you just made. Check for:
> - Consistent error response format (RFC 7807)
> - Proper use of @Valid and custom validators
> - No business logic leaking into the controller
```

### Pattern 4: Database Migration with Safety

Database changes are high-risk. Use a two-agent approach:

```
Agent 1 — Plan (read-only):
> /plan We need to split the "address" columns from the customer table
> into a separate addresses table (one-to-many relationship).
> Plan the Flyway migration, entity changes, and repository updates.
> Consider: zero-downtime deployment, data migration, rollback strategy.

[Review the plan carefully]

Agent 2 — Execute (after approval):
> Implement the migration plan. Start with the Flyway scripts, then
> update the entities and repositories. Run the tests at each step.
```

### Pattern 5: Documentation Generation While Coding

```
> Do these in parallel:
> 1. Implement the OrderArchiveService (moves old orders to cold storage)
> 2. Generate Javadoc for all public methods in the order package
> 3. Update the OpenAPI annotations on OrderController
> 4. Add a section to the README about the archival process
```

---

## 8. Git Worktree Integration

### How Worktrees Enable Parallel Edits

A git worktree is a separate working directory linked to the same repository. Each worktree can be on a different branch, allowing truly parallel development without branch switching.

```
project/
├── .git/                     # Shared repository
├── src/                      # Main worktree (your branch)
├── .worktrees/
│   ├── experiment-a/         # Worktree on branch experiment-a
│   └── experiment-b/         # Worktree on branch experiment-b
```

### Creating Isolated Worktrees for Experimental Changes

```bash
# Create a worktree for an experimental approach
git worktree add .worktrees/strategy-pattern feature/strategy-pattern

# Create another for an alternative approach
git worktree add .worktrees/visitor-pattern feature/visitor-pattern
```

Then run parallel Claude Code sessions:

```bash
# Terminal 1: Implement Strategy pattern approach
cd .worktrees/strategy-pattern
claude -p "Refactor OrderService.processOrder() using the Strategy pattern.
Implement all four order type strategies. Run tests."

# Terminal 2: Implement Visitor pattern approach
cd .worktrees/visitor-pattern
claude -p "Refactor OrderService.processOrder() using the Visitor pattern.
Implement visitors for all four order types. Run tests."
```

### Merging Worktree Results Back

After both agents complete:

```bash
# Compare the two approaches
diff -r .worktrees/strategy-pattern/src .worktrees/visitor-pattern/src

# Or use Claude Code to compare
claude -p "Compare the Strategy pattern implementation in
.worktrees/strategy-pattern/src/main/java/com/example/order/
with the Visitor pattern implementation in
.worktrees/visitor-pattern/src/main/java/com/example/order/

Evaluate: readability, extensibility, testability, and Spring Boot idiomaticness.
Recommend which approach to merge."
```

Merge the winner:

```bash
# Switch to main and merge
git merge feature/strategy-pattern

# Clean up worktrees
git worktree remove .worktrees/strategy-pattern
git worktree remove .worktrees/visitor-pattern
```

### Practical Example: Trying Two Approaches in Parallel

```
> We need to implement rate limiting for our REST API. I want to
> try two approaches in parallel:
>
> Approach A: Bucket4j with Redis backend (distributed)
> Approach B: Resilience4j RateLimiter (local, per-instance)
>
> Create two git worktrees and implement each approach.
> After both are done, compare them on:
> - Complexity
> - Performance overhead
> - Behaviour in a 3-node K8s cluster
> - Testability
```

---

## 9. Best Practices & Anti-Patterns

### When to Use Parallel Agents vs Sequential

| Use Parallel | Use Sequential |
|---|---|
| Independent file exploration | Changes that depend on previous steps |
| Same change across multiple services | Schema change then entity update then service update |
| Research + implementation simultaneously | When you need to review before proceeding |
| Test generation for multiple classes | Feature implementation that must compile at each step |
| Code review of separate modules | Debugging (need to see the result of each step) |

### Context Window Management

**Do:**
- Compact after each logical milestone.
- Use CLAUDE.md to persist key decisions and state.
- Start new sessions for unrelated tasks.
- Be specific in prompts — vague prompts cause Claude Code to read more files, consuming context.

**Don't:**
- Try to do everything in one session.
- Ask Claude Code to "read the entire codebase" — be targeted.
- Forget to compact before starting a new phase of work.

### Avoiding Redundant Work Between Agents

When using parallel agents, be explicit about scope:

```
# Bad — agents may overlap
> In parallel:
> 1. Improve the order service
> 2. Fix bugs in the order service

# Good — clear boundaries
> In parallel:
> 1. Add input validation to OrderController (only controller layer)
> 2. Fix the N+1 query in OrderRepository.findAllWithItems (only repository layer)
```

### Communication Patterns Between Agents

Agents within a cowork session communicate through the parent agent. But across sessions, you need explicit communication channels:

| Channel | Use Case | Example |
|---|---|---|
| **CLAUDE.md** | Persistent project context | Architecture decisions, conventions |
| **TODO comments** | In-code markers | `// TODO(agent-2): implement after schema migration` |
| **Task files** | Complex multi-session work | `tasks/audit-logging.md` with checklist |
| **Git commits** | Completed work handoff | Agent 1 commits, Agent 2 pulls and continues |
| **JSON files** | Structured data exchange | Migration plan, review findings |

### Anti-Patterns to Avoid

**Anti-Pattern 1: Over-Parallelisation**
```
# Don't do this — too many parallel concerns with dependencies
> In parallel:
> 1. Create the database migration
> 2. Update the entity to match
> 3. Update the repository queries
> 4. Update the service layer
> 5. Update the controller
```

The entity depends on the migration. The repository depends on the entity. This must be sequential.

**Anti-Pattern 2: Shared Mutable State**
```
# Don't let two agents edit the same file simultaneously
> In parallel:
> 1. Add a new method to OrderService
> 2. Refactor existing methods in OrderService
```

Both agents editing `OrderService.java` will create conflicts. Either sequence them or use worktree isolation.

**Anti-Pattern 3: Unbounded Exploration**
```
# Don't give an agent an open-ended research task with no scope
> Explore the entire codebase and tell me everything that could be improved.
```

This fills the context window with low-value information. Be specific: "Explore the order module for N+1 query patterns."

**Anti-Pattern 4: Ignoring Plan Mode for Risky Changes**
```
# Don't jump straight into implementation for database-touching changes
> Rename the 'customer_id' column to 'account_id' across all tables.
```

Always `/plan` first for schema changes. The plan will reveal foreign keys, indexes, and application code that references the column.

---

## 10. Try This Exercises

### Exercise 1: Parallel Exploration

Ask Claude Code to explore three unrelated areas of your codebase in parallel:
1. Find all REST endpoints and their HTTP methods.
2. List all Kafka topics referenced in the code.
3. Identify all scheduled jobs and their cron expressions.

Request the results in a unified summary table.

### Exercise 2: Plan-Then-Execute Workflow

Use `/plan` to plan the addition of request/response logging middleware to your Spring Boot application. The plan should cover:
- Filter vs interceptor approach decision
- What to log (method, URL, status, latency, correlation ID)
- Sensitive data masking (passwords, tokens)
- Log format (structured JSON)

Review the plan, iterate once, then execute.

### Exercise 3: Headless Batch Processing

Write a shell script that uses Claude Code in headless mode to:
1. Find all Java files without corresponding test files.
2. For each missing test file, generate a test skeleton.
3. Run the tests and report which pass.

Use parallel execution for the test generation step.

### Exercise 4: Multi-Session Feature Implementation

Implement a feature across three separate Claude Code sessions:
- **Session 1**: Plan and implement the domain model.
- **Session 2**: Implement the service and repository layers (use CLAUDE.md to carry context).
- **Session 3**: Implement the controller, integration tests, and run the full suite.

Track your progress in CLAUDE.md between sessions.

### Exercise 5: Worktree Comparison

Pick a design decision in your codebase (e.g., exception handling strategy). Use git worktrees to implement two different approaches. Then ask Claude Code to compare them and recommend the better option with justification.

---

## 11. Tips and Gotchas

### Tips

1. **Use "in parallel" as a trigger phrase.** When your prompt explicitly says "in parallel" or "do these simultaneously", Claude Code is more likely to spawn subagents rather than working sequentially.

2. **Plan mode is free exploration.** Since plan mode does not make changes, use it liberally. A 2-minute planning phase often saves 20 minutes of rework.

3. **Compact at phase boundaries.** After completing the service layer, compact before starting the controller. After finishing implementation, compact before writing tests.

4. **Use CLAUDE.md as your multi-session brain.** Record decisions, completed work, and next steps. This is the single most important practice for multi-session workflows.

5. **Headless mode + jq = structured automation.** Use `--output-format json` with `jq` to build pipelines that parse Claude Code's output programmatically.

6. **Worktrees are cheap.** Creating a git worktree is nearly instant and costs no extra disk space for the git objects. Use them freely for experimental parallel work.

7. **Scope subagents tightly.** "Explore the OrderService for thread safety issues" is better than "Explore the order module for problems." Tighter scope means faster, more useful results.

8. **Combine plan mode with task creation for large features.** `/plan` gives you the high-level design. Converting the plan into tasks gives you trackable progress.

9. **Use background agents for test suites.** While Claude Code implements changes, have it run the test suite in the background. You get real-time feedback without waiting.

10. **Headless agents in CI benefit from `--allowedTools`.** Restrict tools to `Read,Grep,Glob` for review jobs and `Read,Edit,Write,Bash` for code generation jobs. Never give CI agents unrestricted tool access.

### Gotchas

1. **Subagents do not share context with each other.** If Agent A discovers something Agent B needs to know, the parent agent must relay the information. You may need to explicitly ask: "Share the findings from the first exploration with the other agents."

2. **Parallel file edits without worktrees can conflict.** If two subagents try to edit the same file, one will overwrite the other. For parallel edits to the same file, use sequential execution or worktree isolation.

3. **Headless mode costs add up quickly.** Each headless invocation starts a fresh context. If you run 50 headless agents in a batch script, you pay for 50 separate context initialisations. Bundle related work into fewer, larger prompts when possible.

4. **`/compact` is lossy by design.** It summarises the conversation, discarding details. If a specific code pattern, variable name, or design decision is critical, restate it after compacting or record it in CLAUDE.md.

5. **Plan mode does not guarantee the plan is correct.** It is a proposal, not a guarantee. Always review plans critically, especially for database migrations, security changes, and public API modifications.

6. **Git worktrees share the same `.git` directory.** Running `git stash` in one worktree affects all worktrees. Use branches, not stashes, for worktree-based workflows.

7. **Background agents may finish after you have moved on.** If a background agent discovers an issue (e.g., failing tests), you might already be several steps ahead. Check background results before committing.

8. **Multiple headless agents writing to the same repo need coordination.** Without worktrees or branch isolation, parallel headless agents will create file conflicts. Always use worktree isolation or operate on separate file sets.

9. **Session memory is not automatic.** Claude Code does not automatically persist conversation state to CLAUDE.md. You must explicitly ask it to remember important context, or write it to CLAUDE.md yourself.

10. **Rate limits apply across all agents.** If you spawn 10 parallel headless agents, they all share your API rate limit. You may hit throttling. Limit parallelism to 3-5 concurrent agents for most plans.

---

## Summary

| Concept | Key Takeaway |
|---|---|
| Subagents | Parallel child agents for independent subtasks; each gets its own context window |
| Plan mode | Read-only exploration and planning; always use for changes touching 5+ files |
| Task management | Break complex work into trackable, stateful tasks |
| Headless mode | Non-interactive execution for scripts, CI/CD, and batch processing |
| Multi-session | Use `/compact` and CLAUDE.md to carry state across sessions |
| Coworking patterns | Match the pattern to the problem: parallel for independent work, sequential for dependent steps |
| Git worktrees | Cheap isolated checkouts for safe parallel experimentation |
| Anti-patterns | Avoid parallel edits to same file, unbounded exploration, and over-parallelisation of dependent tasks |

**Previous:** [Tutorial 08 — Daily Workflow Playbook](08-daily-workflow-playbook.md)
