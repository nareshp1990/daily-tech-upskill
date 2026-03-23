# Tutorial 03: Claude Code Advanced Workflows

## Table of Contents

1. [Multi-File Editing Strategies](#1-multi-file-editing-strategies)
2. [Git Workflows](#2-git-workflows)
3. [Plan Mode for Complex Implementations](#3-plan-mode-for-complex-implementations)
4. [Memory System](#4-memory-system)
5. [Context Management](#5-context-management)
6. [Headless Mode and Scripting](#6-headless-mode-and-scripting)
7. [Claude Code in CI/CD](#7-claude-code-in-cicd)
8. [Parallel Agents and Subagents](#8-parallel-agents-and-subagents)
9. [Background Tasks](#9-background-tasks)
10. [Practical Multi-Step Workflows](#10-practical-multi-step-workflows)
11. [Try This Exercises](#11-try-this-exercises)
12. [Tips and Gotchas](#12-tips-and-gotchas)

---

## 1. Multi-File Editing Strategies

### Coordinated Multi-File Changes

Claude Code excels at making changes across multiple files while maintaining consistency. The key is to describe the full scope of the change upfront.

**Effective prompting for multi-file edits:**

```
> Add a new "cancel order" feature:
> 1. Add CANCELLED to OrderStatus enum
> 2. Add cancelOrder() to OrderService with validation
>    (can only cancel orders in CREATED or VALIDATED state)
> 3. Add PUT /api/v1/orders/{id}/cancel endpoint to OrderController
> 4. Add Kafka event publishing for OrderCancelledEvent
> 5. Add unit tests for the service layer
> 6. Add integration test for the endpoint
>
> Follow existing patterns in the codebase.
```

Claude Code will plan the changes across all files and execute them in the right order.

### Refactoring Across Files

```
> Rename the "Customer" entity to "Account" across the entire codebase:
> - Entity class, repository, service, controller, DTOs
> - Database migration (add a new Flyway migration)
> - Kafka event classes
> - Test files
> - Update all references
```

### Strategy: Incremental vs Big-Bang

**Incremental (recommended for large changes):**
```
> Step 1: Let's start by creating the new AccountEntity and AccountRepository.
> Don't touch the old Customer classes yet.

[Review and approve]

> Step 2: Create the AccountService that delegates to AccountRepository.
> Add tests for it.

[Review and approve]

> Step 3: Now update the controller layer to use Account instead of Customer.
> Update the DTOs too.

[Review and approve]

> Step 4: Remove the old Customer classes and update all remaining references.
```

**Big-Bang (fine for smaller, well-scoped changes):**
```
> Refactor the payment processing to use the Strategy pattern.
> Do all the changes at once and make sure all tests pass.
```

### Handling Merge Conflicts

```
> I have merge conflicts in these files after rebasing.
> Show me the conflicts and resolve them, keeping our branch's
> business logic but accepting main's structural changes.
```

---

## 2. Git Workflows

### Commit Workflow

```
> /commit
```

Claude Code will:
1. Run `git status` and `git diff` to understand changes
2. Analyse all modified and new files
3. Generate a descriptive commit message following your conventions
4. Ask for approval before committing

**With custom message guidance:**
```
> /commit
> The commit message should reference JIRA-456 and mention
> the performance improvement.
```

### Pull Request Creation

```
> Create a PR for this branch. The PR should:
> - Target the main branch
> - Include a summary of all changes
> - List any breaking changes
> - Include testing instructions
```

Claude Code will:
1. Analyse all commits on the branch vs main
2. Generate a PR title and description
3. Use `gh pr create` to create the PR on GitHub

### Code Review Workflow

**Review your own changes before pushing:**
```
> /review

> Review all changes since main. Check for:
> 1. Bugs and edge cases
> 2. Thread safety issues
> 3. Missing error handling
> 4. N+1 query problems
> 5. Proper validation
> 6. Test coverage gaps
```

**Review a specific PR:**
```
> Review PR #42. Focus on the database changes and any potential
> migration issues.
```

### Branch Management

```
> Create a new branch for JIRA-789 (adding order export feature),
> switch to it, and set up the initial structure.

> What branches exist and which ones are behind main?
```

---

## 3. Plan Mode for Complex Implementations

### What is Plan Mode?

For complex, multi-step tasks, you can ask Claude Code to create a plan before implementing. This gives you visibility into the approach and lets you make adjustments before any code changes.

### Using Plan Mode

```
> Plan how you would implement a distributed rate limiter for our API gateway.
> Don't write any code yet, just outline the approach.
```

Claude Code will respond with:
```
Plan:
1. Add Redis dependency to pom.xml / build.gradle
2. Create RateLimitConfig with configurable limits per endpoint
3. Implement SlidingWindowRateLimiter using Redis sorted sets
4. Create RateLimitFilter (Spring WebFilter) that intercepts requests
5. Add rate limit headers (X-RateLimit-Remaining, X-RateLimit-Reset)
6. Create RateLimitExceeded exception with 429 response
7. Add configuration properties for fine-tuning
8. Write unit tests for the limiter logic
9. Write integration tests with embedded Redis
10. Add to README / API documentation

Shall I proceed with implementation?
```

You can then:
- Approve the plan as-is
- Request modifications ("Use a token bucket algorithm instead")
- Ask for more detail on specific steps
- Tell it to skip certain steps

### Planning for Large Refactors

```
> I want to migrate our monolith's user module into a separate
> microservice. Plan the migration including:
> - What to extract
> - API contracts between services
> - Database separation strategy
> - Kafka events for synchronisation
> - Migration steps that allow zero downtime
>
> Create a detailed plan with phases. Don't implement yet.
```

---

## 4. Memory System

### Auto-Memory

Claude Code maintains automatic memory across sessions through the memory system. When you share important context, Claude Code can save it to `MEMORY.md` files for future reference.

**How it works:**
- Claude Code detects important project information during conversations
- It writes relevant facts to `~/.claude/projects/<project-path>/MEMORY.md`
- This file is automatically read at the start of each session

### What Gets Remembered

- Project structure and conventions
- Your preferences for code style
- Important architectural decisions
- Frequently referenced files or patterns
- Build and deployment procedures

### Manual Memory Management

You can explicitly tell Claude Code to remember things:

```
> Remember: our team uses feature flags via LaunchDarkly.
> The SDK is already configured in FeatureFlagService.java.
> Always check for feature flags when adding new features.

> Remember: the order-service connects to the legacy Oracle DB
> for reading customer data. This is read-only and will be
> migrated to PostgreSQL in Q3.
```

### Viewing and Editing Memory

The memory file is plain markdown and can be edited manually:

```bash
# View memory for current project
cat ~/.claude/projects/-Users-naresh-projects-order-service/MEMORY.md

# Edit manually
vim ~/.claude/projects/-Users-naresh-projects-order-service/MEMORY.md
```

### Memory Hierarchy

```
~/.claude/CLAUDE.md                          # Global personal preferences
<project>/CLAUDE.md                           # Project conventions (team)
~/.claude/projects/<path>/MEMORY.md          # Auto-memory (personal, per-project)
```

---

## 5. Context Management

### Understanding Context Limits

Even with 200K tokens, long conversations eventually fill the context window. Signs you are running low:

- Claude Code's responses become less accurate
- It "forgets" things you discussed earlier
- Responses feel more generic

### The /compact Command

`/compact` summarises the conversation history, preserving key information while reducing token count.

```
> /compact

Conversation compressed:
- Before: 87,432 tokens
- After: 12,651 tokens
- Key context preserved: file changes, project structure, current task
```

**When to use /compact:**
- After completing a major task, before starting a new one
- When you notice context degradation
- Proactively every 30-40 minutes in long sessions
- Before starting a complex multi-step task (to maximise available context)

### Custom Compact Instructions

```
> /compact Focus on preserving the database schema changes
> and the API contract decisions we made.
```

### Starting Fresh

```
> /clear
```

This completely resets the conversation. Use it when switching to an entirely different task. Your CLAUDE.md and memory files are still read on the next interaction.

### Context-Efficient Prompting

**Instead of:**
```
> Read all the files in the service layer and tell me about them
```

**Use:**
```
> What services handle payment processing? Show me just the class
> names and their key public methods.
```

This is more focused and uses fewer tokens.

---

## 6. Headless Mode and Scripting

### What is Headless Mode?

Headless mode runs Claude Code non-interactively, perfect for scripting and automation.

### Basic Headless Usage

```bash
# Single prompt, get response
claude -p "List all TODO comments in the codebase"

# With specific output format
claude -p "List all REST endpoints in this project as a markdown table" --output-format text

# JSON output for programmatic use
claude -p "What's the Java version in pom.xml?" --output-format json
```

### Piping Input

```bash
# Pipe a file for review
cat src/main/java/com/example/OrderService.java | claude -p "Review this code"

# Pipe error output for debugging
mvn test 2>&1 | claude -p "Explain these test failures and suggest fixes"

# Pipe git diff for review
git diff main | claude -p "Review these changes for bugs"
```

### Scripting with Claude Code

```bash
#!/bin/bash
# review-pr.sh — Automated PR review script

BRANCH=$(git branch --show-current)
BASE="main"

echo "Reviewing changes on branch: $BRANCH"

# Get the diff
DIFF=$(git diff $BASE...$BRANCH)

# Ask Claude to review
claude -p "Review this git diff for a Spring Boot project. Check for:
1. Bugs and edge cases
2. Security issues
3. Performance concerns
4. Missing tests

Diff:
$DIFF" --output-format text
```

### Chaining Commands

```bash
# Generate tests, then run them
claude -p "Write unit tests for OrderService.java" && mvn test

# Fix compilation errors in a loop
while ! mvn compile 2>/dev/null; do
    mvn compile 2>&1 | claude -p "Fix these compilation errors" --auto-accept-edits
done
```

### Using --allowedTools for Safety

In headless mode, restrict what Claude Code can do:

```bash
claude -p "Review this code" \
  --allowedTools "Read,Glob,Grep" \
  --output-format text
```

This allows reading but prevents any edits or command execution.

---

## 7. Claude Code in CI/CD

### GitHub Actions Integration

```yaml
# .github/workflows/claude-review.yml
name: Claude Code Review

on:
  pull_request:
    types: [opened, synchronize]

jobs:
  review:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      pull-requests: write
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - uses: actions/setup-node@v4
        with:
          node-version: '20'

      - name: Install Claude Code
        run: npm install -g @anthropic-ai/claude-code

      - name: Review PR
        env:
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
        run: |
          git diff origin/main...HEAD | claude -p \
            "Review this PR diff. Focus on bugs, security issues, \
             and Spring Boot best practices. Format as a concise \
             code review with line references." \
            --allowedTools "Read,Glob,Grep" \
            --output-format text > review.md

      - name: Post Review Comment
        uses: actions/github-script@v7
        with:
          script: |
            const fs = require('fs');
            const review = fs.readFileSync('review.md', 'utf8');
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: `## Claude Code Review\n\n${review}`
            });
```

### Automated Test Generation in CI

```yaml
# .github/workflows/generate-tests.yml
name: Generate Missing Tests

on:
  workflow_dispatch:
    inputs:
      target_class:
        description: 'Class to generate tests for'
        required: true

jobs:
  generate-tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: '20'

      - name: Install Claude Code
        run: npm install -g @anthropic-ai/claude-code

      - name: Generate Tests
        env:
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
        run: |
          claude -p "Generate comprehensive unit tests for \
            ${{ github.event.inputs.target_class }}. \
            Use JUnit 5, Mockito, and AssertJ. \
            Follow existing test patterns in the project." \
            --auto-accept-edits

      - name: Run Tests
        run: mvn test

      - name: Create PR with Tests
        run: |
          git checkout -b auto/tests-$(date +%s)
          git add -A
          git commit -m "Add generated tests for ${{ github.event.inputs.target_class }}"
          gh pr create --title "Add tests for ${{ github.event.inputs.target_class }}" \
            --body "Automatically generated tests using Claude Code"
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

---

## 8. Parallel Agents and Subagents

### What are Subagents?

Claude Code can spawn subagent tasks that run concurrently. This is useful for tasks that are independent and can be parallelised.

### How Claude Code Uses Parallel Execution

When you give Claude Code a task that has independent subtasks, it can use its internal agent tool to handle them in parallel:

```
> Review all 5 service classes in the service package.
> For each one, check for thread safety issues and
> suggest improvements.
```

Claude Code may internally spawn parallel read operations to examine all five files simultaneously, then synthesise the results.

### Practical Parallel Workflows

**Parallel code analysis:**
```
> Analyse these three microservices for consistency:
> - order-service/
> - payment-service/
> - inventory-service/
>
> Check that they all follow the same patterns for:
> error handling, logging, health checks, and API versioning.
```

**Parallel file generation:**
```
> Generate DTOs for all five database entities.
> Each DTO needs: request DTO, response DTO, and mapper.
```

### Running Multiple Claude Code Instances

For truly independent tasks, you can run multiple Claude Code instances in separate terminals:

```bash
# Terminal 1
cd order-service && claude -p "Add input validation to all controllers"

# Terminal 2
cd payment-service && claude -p "Add input validation to all controllers"

# Terminal 3
cd inventory-service && claude -p "Add input validation to all controllers"
```

---

## 9. Background Tasks

### Long-Running Tasks

For tasks that take a while (large refactors, comprehensive test generation), Claude Code continues working while you can observe progress.

```
> Generate integration tests for all repository classes.
> This will take a while — go ahead and do them all.
```

### Monitoring Progress

Claude Code shows progress as it works:
- Files being read
- Changes being made
- Commands being executed
- Errors encountered and resolved

### Detaching and Reattaching

If you need to close your terminal, you can use tools like `tmux` or `screen`:

```bash
# Start a tmux session
tmux new -s claude-refactor

# Run Claude Code
claude -p "Refactor all services to use constructor injection"

# Detach: Ctrl+B, then D

# Reattach later
tmux attach -t claude-refactor
```

---

## 10. Practical Multi-Step Workflows

### Workflow 1: Feature Development (End-to-End)

```
Step 1 — Plan:
> I need to add a "favourite orders" feature where users can bookmark
> orders for quick access. Plan the implementation.

Step 2 — Database:
> Create the Flyway migration for the favourites table.

Step 3 — Domain:
> Create the domain model, repository, and service layer.

Step 4 — API:
> Create the REST endpoints. Follow our existing patterns.

Step 5 — Tests:
> Write unit tests for the service and integration tests for the API.

Step 6 — Run Tests:
> Run all the tests and fix any failures.

Step 7 — Review:
> /review

Step 8 — Commit:
> /commit
```

### Workflow 2: Bug Fix

```
Step 1 — Reproduce:
> There's a bug where orders with more than 100 items throw an OOM error.
> Can you find where the issue might be?

Step 2 — Root Cause:
> Found it — the OrderMapper loads all items eagerly.
> Show me the relevant code.

Step 3 — Fix:
> Fix the eager loading issue. Use a paginated approach for large orders.

Step 4 — Test:
> Write a test that would have caught this bug.

Step 5 — Verify:
> Run the full test suite to make sure nothing else broke.

Step 6 — Commit:
> /commit
```

### Workflow 3: Performance Investigation

```
Step 1:
> Our order listing API is slow (2-3 seconds). Analyse the code path
> from OrderController.listOrders through to the database query.

Step 2:
> I see several N+1 queries. Show me all the JPA relationships
> that are fetched lazily but accessed in loops.

Step 3:
> Fix the N+1 queries using JOIN FETCH or @EntityGraph.

Step 4:
> Also add a database index for the common query patterns
> (status + createdDate sorting). Generate the Flyway migration.

Step 5:
> Run the tests and make sure the fixes don't break anything.
```

### Workflow 4: Microservice Scaffolding

```
> Create a new Spring Boot microservice called "notification-service" with:
> 1. Standard project structure (hexagonal architecture)
> 2. Spring Boot 3.2, Java 21, Gradle
> 3. Kafka consumer for notification events
> 4. REST API for notification preferences
> 5. PostgreSQL for storage with Flyway
> 6. Docker and K8s manifests
> 7. GitHub Actions CI/CD pipeline
> 8. Health check and Prometheus metrics
> 9. OpenAPI spec
> 10. CLAUDE.md with project conventions
>
> Follow the same patterns as our order-service.
```

---

## 11. Try This Exercises

### Exercise 1: Multi-File Feature
Ask Claude Code to add a new feature that touches at least 4 files (entity, repository, service, controller, test). Review the changes before applying.

### Exercise 2: Git Workflow
1. Make some changes manually
2. Use `/review` to get a code review
3. Use `/commit` to commit with a generated message
4. Create a branch, make more changes, and ask Claude Code to create a PR

### Exercise 3: Plan Mode
Ask Claude Code to plan a significant refactor (e.g., extracting a bounded context into its own module). Review the plan, ask for modifications, then implement step by step.

### Exercise 4: Headless Scripting
Write a bash script that:
1. Takes a Java class name as input
2. Uses Claude Code in headless mode to generate unit tests
3. Runs the tests
4. Reports pass/fail

### Exercise 5: Context Management
1. Start a long conversation (10+ interactions)
2. Check cost with `/cost`
3. Run `/compact`
4. Check cost again
5. Verify Claude Code still remembers the key context

---

## 12. Tips and Gotchas

### Tips

1. **Plan before implementing large changes.** "Plan how you would..." saves time by catching misalignment early.

2. **Use incremental multi-file edits for safety.** Let Claude Code make changes to one layer at a time, verify, then continue.

3. **Headless mode + Git hooks = powerful automation.** Use pre-commit hooks that run Claude Code for quick checks.

4. **Compact after completing sub-tasks.** Each time you finish a logical unit of work, compact to preserve context for the next task.

5. **Use `--allowedTools` in CI/CD to prevent writes.** For review pipelines, restrict Claude Code to read-only operations.

6. **Combine Claude Code with git worktrees** for parallel work on different features without branch switching.

7. **Keep Claude Code sessions focused.** One session per feature/task. Start fresh for unrelated work.

### Gotchas

1. **Multi-file edits can conflict with your uncommitted changes.** Always commit or stash before large refactors.

2. **Headless mode uses the same token costs.** Automation can run up costs quickly. Set budget limits.

3. **Parallel instances share the same CLAUDE.md but not conversation context.** Changes made by one instance are not visible to another until it reads the file again.

4. **CI/CD pipelines need the ANTHROPIC_API_KEY secret.** Store it securely in GitHub Secrets, never in code.

5. **`/compact` is lossy.** It summarises, so nuanced details may be lost. If a specific detail is critical, state it again after compacting.

6. **Auto-accept in CI/CD needs careful scoping.** Always restrict allowed tools in automated environments.

7. **Large diffs in headless mode may exceed token limits.** For very large PRs, split the review into per-file reviews.

---

## Summary

| Concept | Key Takeaway |
|---------|-------------|
| Multi-file edits | Describe the full scope upfront; use incremental approach for large changes |
| Git workflows | `/review` before committing, `/commit` for smart messages, Claude Code for PR creation |
| Plan mode | "Plan how you would..." before implementing complex features |
| Memory | MEMORY.md persists across sessions; explicitly tell Claude Code to remember important facts |
| Context | `/compact` regularly; start fresh sessions for new tasks |
| Headless mode | `-p` flag for scripting; `--allowedTools` for safety |
| CI/CD | GitHub Actions integration for automated reviews and test generation |
| Parallel work | Subagents for independent subtasks; multiple instances for separate services |

**Next:** [Tutorial 04 — Claude Code Customisation](04-claude-code-customisation.md)
