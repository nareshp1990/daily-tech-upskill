# Tutorial 09: Claude Code Skills & Slash Commands

## Table of Contents

1. [Overview: What Are Claude Code Skills?](#1-overview-what-are-claude-code-skills)
2. [Built-in Slash Commands Reference](#2-built-in-slash-commands-reference)
3. [User-Invokable Skills](#3-user-invokable-skills)
4. [Custom Slash Commands](#4-custom-slash-commands)
5. [Skill Composition & Workflows](#5-skill-composition--workflows)
6. [MCP-Powered Skills](#6-mcp-powered-skills)
7. [Creating a Skills Library](#7-creating-a-skills-library)
8. [Try This Exercises](#8-try-this-exercises)
9. [Tips and Gotchas](#9-tips-and-gotchas)

---

## 1. Overview: What Are Claude Code Skills?

Claude Code's skill system is the extensibility layer that turns an already capable AI assistant into one that is deeply customised for your workflow. Skills fall into three categories:

| Category | Examples | How They Work |
|----------|---------|---------------|
| **Built-in commands** | `/help`, `/clear`, `/model` | Hard-coded CLI commands executed by the Claude Code runtime. They manage the session itself. |
| **User-invokable skills** | `/commit`, `/review-pr` | Bundled prompt-based skills that Claude Code ships with. They are essentially curated prompt templates that invoke Claude's agent loop with a specific goal. |
| **Custom slash commands** | `/project:review-service` | Markdown files you author. Placed in `.claude/commands/` (project) or `~/.claude/commands/` (global). They inject the markdown content as a prompt when invoked. |

### How Skills Differ from Plain Prompts

A skill is more than a shortcut. When you type `/commit`, Claude Code:

1. Loads the skill's prompt template (including any `$ARGUMENTS` substitution).
2. Executes the resulting prompt within the agent loop -- meaning it can call tools (Bash, Read, Edit, Grep) autonomously.
3. Follows the skill's embedded constraints (e.g., the commit skill always runs `git status`, `git diff`, and `git log` before drafting a message).

This is fundamentally different from pasting a long prompt manually, because the skill also carries metadata about which tools it is allowed to invoke and what workflow it should follow.

### Mental Model

```
┌──────────────────────────────────────────┐
│              Claude Code CLI             │
├──────────┬──────────┬────────────────────┤
│ Built-in │  Bundled │  Custom Commands   │
│ Commands │  Skills  │  (.claude/commands)│
│ /help    │ /commit  │  /project:*        │
│ /clear   │ /review  │  ~/.claude/cmds/*  │
│ /model   │ /loop    │  your-team-cmds/*  │
├──────────┴──────────┴────────────────────┤
│          Agent Loop (tools, MCP)         │
└──────────────────────────────────────────┘
```

---

## 2. Built-in Slash Commands Reference

Built-in commands are executed directly by the Claude Code runtime. They do not go through the AI model -- they are immediate, deterministic operations on the session.

### Complete Reference

| Command | Purpose | Syntax | Key Details |
|---------|---------|--------|-------------|
| `/help` | Show available commands | `/help` | Lists all slash commands including custom ones. Good first command in any session. |
| `/clear` | Clear conversation history | `/clear` | Resets the context window. Your CLAUDE.md and memory files are re-loaded on next prompt. Does not affect files on disk. |
| `/compact` | Compact conversation history | `/compact [instructions]` | Summarises the conversation to free up context window space. Optionally pass instructions to guide what to preserve. |
| `/config` | View or edit configuration | `/config` | Opens an interactive view of your settings. Equivalent to inspecting `~/.claude/settings.json`. |
| `/cost` | Show token usage and cost | `/cost` | Displays input/output tokens consumed and estimated dollar cost for the current session. |
| `/doctor` | Diagnose installation issues | `/doctor` | Checks authentication, model access, MCP server connectivity, and configuration validity. Run this first when something feels broken. |
| `/init` | Initialise CLAUDE.md | `/init` | Creates a `CLAUDE.md` file in the current project root with sensible defaults. Safe to run on existing projects -- it will not overwrite. |
| `/login` | Authenticate | `/login` | Starts the OAuth flow or API key entry. Supports Anthropic direct, AWS Bedrock, and Google Vertex. |
| `/logout` | Sign out | `/logout` | Clears stored credentials. |
| `/memory` | View or edit memory | `/memory` | Opens your project and user memory files for viewing and editing. Memory persists across sessions. |
| `/model` | Switch model | `/model [model-name]` | Switch between available models mid-session. E.g., `/model claude-sonnet-4-20250514`. Without arguments, lists available models. |
| `/permissions` | Manage permissions | `/permissions` | View and modify tool permissions (allow/deny rules). Changes are written to settings files. |
| `/review` | Review code | `/review` | Triggers a code review of your current staged or unstaged changes. |
| `/status` | Show session status | `/status` | Displays current model, token usage, MCP server connections, and active permissions. |
| `/terminal-setup` | Set up terminal integration | `/terminal-setup` | Configures your terminal (iTerm2, VS Code terminal, etc.) for optimal Claude Code integration -- e.g., enabling Shift+Enter for multi-line input. |
| `/vim` | Toggle vim mode | `/vim` | Enables/disables vim-style keybindings in the Claude Code input area. |
| `/fast` | Toggle fast model | `/fast` | Switches between the default model and a faster (cheaper) model for quick tasks. Useful when you need a fast answer and don't need deep reasoning. |

### Practical Usage Patterns

**Starting a session right:**
```
> /status                    # Check model, tokens, MCP servers
> /doctor                    # Verify everything is healthy
> /model                     # Confirm which model is active
```

**Managing a long session:**
```
# After 30+ minutes of back-and-forth, context window fills up
> /cost                      # Check how much context is consumed
> /compact Preserve the OrderService refactoring plan and the list of files changed
> /cost                      # Verify context was freed
```

**Quick model switching for cost control:**
```
# Use the fast model for simple questions
> /fast
> What's the annotation for marking a Spring bean as request-scoped?

# Switch back for complex tasks
> /fast
> Now refactor the OrderService to use the Saga pattern
```

**Recovering from confusion:**
```
# Claude seems confused about the codebase state
> /clear
> Let's start fresh. Read the OrderService.java and explain the cancel flow.
```

---

## 3. User-Invokable Skills

These are skills that ship with Claude Code. They are prompt-based workflows that leverage the full agent loop -- reading files, running commands, and making edits.

### `/commit` -- Smart Git Commits

The commit skill automates the entire commit workflow: staging, diffing, writing a message, and creating the commit.

**How it works:**
1. Runs `git status` to see untracked and modified files.
2. Runs `git diff` (staged + unstaged) to understand changes.
3. Reads recent `git log` to match the repository's commit message style.
4. Drafts a commit message summarising the "why" not the "what."
5. Stages relevant files (avoids `.env`, credentials, binaries).
6. Creates the commit.

**Usage:**
```
> /commit
```

**With guidance:**
```
> /commit -m "focus on the Kafka consumer retry logic changes"
```

**Spring Boot example session:**
```
# You've just finished adding retry logic to KafkaConsumerConfig.java
# and updated application.yml with new retry properties

> /commit

# Claude Code will:
# 1. See KafkaConsumerConfig.java and application.yml are modified
# 2. Read the diffs
# 3. Generate: "Add exponential backoff retry to Kafka consumer configuration"
# 4. Stage both files and commit
```

### `/review-pr` -- Pull Request Review

Reviews a pull request, either the current branch's PR or a specific PR number.

**Usage:**
```
> /review-pr              # Review current branch's PR
> /review-pr 142          # Review PR #142
```

**What it checks:**
- Code quality and consistency with project conventions
- Potential bugs, null pointer risks, resource leaks
- Test coverage gaps
- API contract changes
- Security concerns

**Spring Boot context:**
```
> /review-pr 87

# Claude examines all changed files in PR #87, e.g.:
# - OrderController.java: new endpoint added
# - OrderService.java: business logic changes
# - OrderServiceTest.java: new test cases
#
# Output might include:
# "The new PUT /orders/{id}/status endpoint lacks @PreAuthorize.
#  The OrderService.updateStatus() doesn't validate state transitions.
#  Missing test for the COMPLETED -> CANCELLED transition (should throw)."
```

### `/simplify` -- Code Simplification

Analyses the current file or specified code and suggests simplifications.

**Usage:**
```
> /simplify                         # Simplify the file in context
> /simplify OrderService.java       # Target a specific file
```

**What it does:**
- Identifies overly complex methods (high cyclomatic complexity)
- Suggests modern Java alternatives (streams, records, pattern matching)
- Recommends extracting methods, reducing nesting
- Preserves behaviour -- never changes semantics

**Example output for a Spring Boot service:**
```
> /simplify src/main/java/com/example/order/service/OrderService.java

# Suggestions:
# 1. processOrder() has 6 levels of nesting -- extract validation into
#    a private validateOrder() method
# 2. The status-to-action switch block (lines 45-89) can be replaced
#    with an EnumMap<OrderStatus, Consumer<Order>>
# 3. Lines 102-115: manual null checks can use Optional.ofNullable() chain
# 4. The OrderResponse construction (lines 120-135) should use a
#    record + static factory method
```

### `/loop` -- Recurring Tasks

Sets up a task that Claude Code repeats on a schedule or trigger, useful for watch-style workflows during development.

**Usage:**
```
> /loop Run mvn test every time I save a file in src/main/java
> /loop Check for new error logs in target/app.log every 30 seconds
```

**Spring Boot development loop:**
```
> /loop Watch for compilation errors. When I say "check", run mvn compile
  and report any errors with suggested fixes.
```

### `/claude-api` -- Building with Claude API

Provides guidance and code generation for integrating the Claude API into your application.

**Usage:**
```
> /claude-api Generate a Spring Boot service that calls Claude for text summarisation
```

### `/update-config` -- Configure Settings

An interactive way to update Claude Code's configuration, hooks, and permissions.

**Usage:**
```
> /update-config Allow Bash(docker compose *) in project permissions
> /update-config Add a PostToolUse hook that runs checkstyle after Java file edits
```

### `/keybindings-help` -- Keyboard Customisation

Shows available keybindings and how to customise them.

**Usage:**
```
> /keybindings-help
```

**Key defaults to know:**

| Keybinding | Action |
|-----------|--------|
| `Enter` | Send message |
| `Shift+Enter` | New line (requires terminal setup) |
| `Ctrl+C` | Cancel current generation |
| `Ctrl+D` | Exit Claude Code |
| `Escape` | Clear current input / cancel |
| `Up Arrow` | Previous message (input history) |
| `Tab` | Accept autocomplete / command completion |

---

## 4. Custom Slash Commands

Custom slash commands are the most powerful extensibility mechanism. You write a markdown file, and Claude Code treats its content as a prompt template.

### Directory Structure

```
project-root/
└── .claude/
    └── commands/
        ├── review-service.md          # /project:review-service
        ├── generate-test.md           # /project:generate-test
        ├── check-security.md          # /project:check-security
        └── db-migration.md            # /project:db-migration

~/.claude/
└── commands/
    ├── morning-standup.md             # /user:morning-standup
    ├── end-of-day.md                  # /user:end-of-day
    └── review-my-prs.md              # /user:review-my-prs
```

### Naming Convention

- **Project commands** are prefixed with `/project:` when invoked.
- **User commands** are prefixed with `/user:` when invoked.
- The filename (minus `.md`) becomes the command name.
- Use kebab-case for filenames: `generate-test.md`, not `generateTest.md`.

### The `$ARGUMENTS` Placeholder

Any occurrence of `$ARGUMENTS` in the markdown file is replaced with whatever the user types after the command.

```markdown
# .claude/commands/generate-test.md
Generate comprehensive unit tests for: $ARGUMENTS

Follow these rules:
- Use JUnit 5 with @ExtendWith(MockitoExtension.class)
- Mock all dependencies using @Mock and @InjectMocks
- Test happy path, edge cases, and error scenarios
- Use AssertJ assertions (assertThat)
- Name tests using: should_expectedBehavior_when_condition
- Include @DisplayName for readability
```

**Invocation:**
```
> /project:generate-test OrderService
> /project:generate-test PaymentGatewayAdapter.processPayment()
```

### Practical Examples for Java/Spring Boot

#### Example 1: Service Layer Review

**`.claude/commands/review-service.md`:**
```markdown
Perform a thorough review of the Spring service class: $ARGUMENTS

Check for:

**Architecture:**
- Single Responsibility: does this service do too much?
- Are dependencies injected via constructor (not field injection)?
- Is the class annotated with @Service or @Component appropriately?
- Are interfaces used for external dependencies (ports)?

**Transaction Management:**
- Are write operations wrapped in @Transactional?
- Are read-only operations marked @Transactional(readOnly = true)?
- Is there any risk of the N+1 query problem?
- Are transactions kept short (no external HTTP calls inside)?

**Error Handling:**
- Are domain exceptions thrown (not generic RuntimeException)?
- Are infrastructure exceptions caught and wrapped?
- Is there proper logging before throwing?

**Thread Safety:**
- Is mutable shared state avoided?
- Are any fields non-final that shouldn't be?

**Testability:**
- Can this be tested without Spring context (pure unit test)?
- Are methods too long to test easily (>20 lines)?

Provide a summary with severity ratings: Critical / Warning / Suggestion.
For each finding, show the specific line and a concrete fix.
```

**Usage:**
```
> /project:review-service OrderService
> /project:review-service PaymentProcessingService
```

#### Example 2: Database Migration Generator

**`.claude/commands/db-migration.md`:**
```markdown
Generate a Flyway database migration for: $ARGUMENTS

Rules:
1. Name the file using the next version number in db/migration/
   (check existing files to determine the next V number)
2. Use the naming convention: V{n}__description.sql
3. Always include a backwards-compatible strategy:
   - Adding columns: use DEFAULT values or allow NULL initially
   - Renaming: add new column, NOT remove old (separate migration for removal)
   - Adding indexes: use CREATE INDEX CONCURRENTLY if PostgreSQL
4. Include a comment header with the change description
5. Include a rollback comment block (-- ROLLBACK: instructions)
6. Verify the migration does not break existing queries by checking
   repository interfaces and @Query annotations in the codebase

Generate the migration file and report any potential compatibility issues.
```

**Usage:**
```
> /project:db-migration Add a "cancelled_reason" column to the orders table
> /project:db-migration Create a customer_preferences table with JSON column
```

#### Example 3: Security Audit

**`.claude/commands/check-security.md`:**
```markdown
Perform a security audit of the current staged changes (git diff --cached).
If nothing is staged, audit the unstaged changes.

Check against the OWASP Top 10 and Spring-specific vulnerabilities:

1. **Injection** (SQL, JPQL, NoSQL, LDAP, OS command)
   - Any string concatenation in @Query annotations?
   - Any raw JDBC without parameterised queries?

2. **Broken Authentication**
   - New endpoints missing @PreAuthorize or security config?
   - Passwords handled in plaintext anywhere?

3. **Sensitive Data Exposure**
   - Entity fields being returned in API responses that shouldn't be?
   - Secrets in application.yml without ${ENV_VAR} references?

4. **Mass Assignment**
   - DTOs accepting fields that map to sensitive entity fields?
   - Missing @JsonIgnore on sensitive fields?

5. **Security Misconfiguration**
   - CORS configured too broadly?
   - Spring Actuator endpoints exposed without authentication?
   - Debug/dev profiles accidentally active?

6. **Insecure Deserialization**
   - Using ObjectMapper with default typing enabled?
   - Accepting polymorphic JSON without @JsonTypeInfo restrictions?

Output format:
- CRITICAL: Must fix before merge
- HIGH: Should fix before merge
- MEDIUM: Fix in follow-up
- LOW: Nice to have
- INFO: Observation, no action needed

For each finding, include the file, line number, and a concrete remediation.
```

#### Example 4: API Endpoint Generator

**`.claude/commands/generate-endpoint.md`:**
```markdown
Generate a complete REST API endpoint for: $ARGUMENTS

Create all layers following the project's existing patterns:

1. **Controller** (@RestController)
   - Use @Operation and @ApiResponse (OpenAPI annotations)
   - Input validation with @Valid
   - Proper HTTP status codes (201 for create, 204 for delete)
   - Use ResponseEntity

2. **Request/Response DTOs** (Java records)
   - @NotNull, @NotBlank, @Size validations
   - Never expose entity IDs in request DTOs

3. **Service** (@Service)
   - Constructor injection
   - @Transactional where needed
   - Domain exceptions for business rule violations

4. **Repository** (Spring Data JPA)
   - Only add custom queries if needed

5. **Unit Tests**
   - Service layer test with Mockito
   - Controller layer test with @WebMvcTest

6. **Integration Test**
   - @SpringBootTest with TestRestTemplate or MockMvc
   - Test happy path and error scenarios

Look at existing endpoints in the codebase to match conventions exactly.
```

**Usage:**
```
> /project:generate-endpoint CRUD for Product entity
> /project:generate-endpoint POST /api/v1/orders/{id}/refund
```

#### Example 5: User-Level Morning Routine

**`~/.claude/commands/morning-standup.md`:**
```markdown
Help me prepare for the morning standup. Perform these steps:

1. **Yesterday's work:**
   - Run: git log --since="yesterday 9am" --until="today 9am" --author="naresh" --oneline
   - Summarise what was done in 2-3 bullet points

2. **Open PRs:**
   - Run: gh pr list --author="@me" --state=open
   - List any that need attention (stale, review requested)

3. **PRs to review:**
   - Run: gh pr list --search "review-requested:@me"
   - List with title and author

4. **Current branch status:**
   - Run: git status
   - Any uncommitted work? Any stashed changes?

5. **CI status:**
   - Run: gh run list --limit 5
   - Any failures on my branches?

Format as a standup-ready summary:
**Yesterday:** [bullets]
**Today:** [inferred from open PRs, TODOs, current branch]
**Blockers:** [failing CI, stale PRs, etc.]
```

---

## 5. Skill Composition & Workflows

Individual skills become powerful when composed into workflows. You can chain them manually or create meta-commands that describe a multi-step process.

### The PR Workflow Chain

```
# 1. Review your changes
> /review

# 2. Fix any issues Claude identified
> Fix the null check issue in OrderService line 45 and add the missing test

# 3. Commit with a well-crafted message
> /commit

# 4. Push and create/update PR
> Push this branch and create a PR targeting main

# 5. Self-review the PR
> /review-pr
```

### Creating a Meta-Command for PR Workflow

**`.claude/commands/ship-it.md`:**
```markdown
Execute the full PR shipping workflow for the current branch:

1. **Pre-flight checks:**
   - Run `mvn verify` to ensure all tests pass
   - Run `mvn checkstyle:check` if configured
   - Check for any TODO/FIXME in changed files (git diff main...HEAD)

2. **Code review:**
   - Review all changes since branching from main
   - Flag any critical issues that must be fixed before merge

3. **If no critical issues found:**
   - Stage and commit any uncommitted changes with a descriptive message
   - Push the branch to origin
   - Create a PR with:
     - Title derived from branch name and changes
     - Body with a summary of changes, testing notes, and checklist
   - Report the PR URL

4. **If critical issues found:**
   - List all issues with severity
   - Do NOT commit or push
   - Ask what to fix first

Stop and report status after each major step.
```

**Usage:**
```
> /project:ship-it
```

### The Debugging Chain

**`.claude/commands/debug-issue.md`:**
```markdown
Help me debug this issue: $ARGUMENTS

Follow this systematic debugging approach:

1. **Reproduce:** Understand the symptom. What's the expected vs actual behaviour?

2. **Locate:** Search the codebase for relevant code:
   - Find the entry point (controller/consumer/scheduler)
   - Trace the call chain through service and repository layers
   - Identify the exact location where behaviour diverges

3. **Analyse:** For the suspect code:
   - Check for null safety issues
   - Check for concurrency problems
   - Check for incorrect query logic
   - Check for missing error handling
   - Check configuration (application.yml, bean definitions)

4. **Hypothesise:** List the top 3 most likely root causes, ranked by probability

5. **Verify:** For the top hypothesis:
   - Suggest a minimal test that would confirm/deny it
   - If possible, write and run that test

6. **Fix:** Propose a fix with:
   - The code change
   - A regression test
   - Any configuration changes needed

Present findings at each step before proceeding to the next.
```

**Usage:**
```
> /project:debug-issue Orders with status PENDING are not being picked up by the scheduler
> /project:debug-issue Kafka consumer stops processing after a deserialization error
```

### The Refactoring Workflow

**`.claude/commands/refactor-plan.md`:**
```markdown
Create a detailed refactoring plan for: $ARGUMENTS

1. **Current State Analysis:**
   - Read the relevant files and understand the current implementation
   - Identify code smells (long methods, god classes, feature envy, etc.)
   - Map dependencies (what calls this code, what does it call?)

2. **Target State Design:**
   - Describe the desired end state
   - Identify which design patterns apply
   - List all files that will change

3. **Step-by-Step Plan:**
   - Break the refactoring into small, individually committable steps
   - Each step must leave the codebase in a compilable, test-passing state
   - Order steps to minimise risk (add new before removing old)

4. **Risk Assessment:**
   - What could go wrong?
   - Which steps are reversible vs irreversible?
   - What tests need to be added before starting?

Do NOT make any changes yet. Present the plan for approval.
```

---

## 6. MCP-Powered Skills

MCP (Model Context Protocol) servers extend Claude Code's capabilities with external tools. Custom commands can leverage these tools for powerful workflows.

### How MCP and Skills Interact

When an MCP server is configured, its tools become available to Claude Code's agent loop. Your custom commands can reference these tools implicitly -- Claude Code will use them when needed.

```
Custom Command Prompt
        │
        ▼
  Agent Loop decides which tools to use
        │
        ├── Built-in tools (Bash, Read, Edit, Grep)
        └── MCP tools (Docker, PostgreSQL, GitHub, Jira)
```

### Example: Database-Aware Review Command

With a PostgreSQL MCP server configured:

**`.claude/commands/review-query-perf.md`:**
```markdown
Analyse the query performance of the repository: $ARGUMENTS

1. Find all custom @Query annotations in the repository interface
2. For each query:
   - Run EXPLAIN ANALYZE against the development database
   - Check if appropriate indexes exist
   - Identify any sequential scans on large tables
   - Check for N+1 patterns in related service code

3. Generate a report:
   - Queries sorted by estimated cost (worst first)
   - For each problematic query: the EXPLAIN output, the issue, and a fix
   - Suggested index additions (as Flyway migration SQL)

Note: Use the PostgreSQL MCP tools to run EXPLAIN queries.
```

### Example: Docker-Integrated Deployment Check

With the Docker MCP server configured:

**`.claude/commands/local-deploy-test.md`:**
```markdown
Test the current build in a local Docker environment:

1. Build the Docker image: docker build -t order-service:test .
2. Start dependencies with docker compose (if docker-compose.yml exists)
3. Run the container and wait for the health check to pass
4. Hit the /actuator/health endpoint
5. Run a basic smoke test:
   - POST /api/v1/orders with a sample payload
   - GET /api/v1/orders/{id} to verify it was created
6. Collect container logs
7. Stop and clean up containers

Report: build time, startup time, health check status, smoke test results.
If anything fails, show the relevant logs and suggest a fix.
```

### Example: GitHub-Integrated Issue Workflow

With the GitHub MCP server or `gh` CLI:

**`.claude/commands/work-on-issue.md`:**
```markdown
Start working on GitHub issue: $ARGUMENTS

1. Fetch the issue details (title, description, labels, comments)
2. Create a feature branch: feature/issue-{number}-{slug}
3. Analyse the issue requirements against the current codebase
4. Create a TODO checklist of implementation steps
5. Start with the first TODO item

If the issue is unclear, list questions that need answers before starting.
```

---

## 7. Creating a Skills Library

A skills library is a curated set of custom commands shared across your team. It encodes your team's development standards into executable workflows.

### Repository Structure

```
.claude/
└── commands/
    ├── README.md                      # Documents all available commands
    │
    ├── # Code generation
    ├── generate-endpoint.md           # Full CRUD endpoint scaffolding
    ├── generate-test.md               # Unit + integration test generation
    ├── generate-dto.md                # DTO + mapper generation
    ├── generate-consumer.md           # Kafka consumer scaffolding
    │
    ├── # Code quality
    ├── review-service.md              # Service layer review
    ├── review-controller.md           # Controller layer review
    ├── check-security.md              # Security audit
    ├── check-performance.md           # Performance review
    │
    ├── # Database
    ├── db-migration.md                # Generate Flyway migration
    ├── review-query-perf.md           # Query performance analysis
    │
    ├── # Workflow
    ├── ship-it.md                     # Full PR workflow
    ├── debug-issue.md                 # Systematic debugging
    ├── refactor-plan.md               # Refactoring plan (no changes)
    │
    └── # Documentation
        └── document-api.md            # Generate API documentation
```

### Versioning Commands with the Project

Since `.claude/commands/` lives in your project repo, commands evolve with the codebase:

```bash
# Commands are committed alongside code
git add .claude/commands/generate-endpoint.md
git commit -m "Add generate-endpoint custom command for consistent REST scaffolding"
```

This means:
- New team members get the commands when they clone the repo.
- Commands can be reviewed in PRs just like code.
- Old branches retain the commands that were valid at that point in time.

### Team Conventions File

Create a shared foundation that all commands can reference:

**`.claude/commands/_conventions.md`** (underscore prefix so it sorts first and is not invoked directly):
```markdown
# Team Conventions (referenced by other commands)

## Package Structure
com.example.{service-name}
├── controller/     # REST controllers
├── service/        # Business logic
├── repository/     # Data access
├── model/
│   ├── entity/     # JPA entities
│   ├── dto/        # Request/Response DTOs
│   └── event/      # Kafka events
├── config/         # Spring configuration
├── exception/      # Custom exceptions
└── util/           # Utilities (should be minimal)

## Naming Conventions
- Controllers: {Entity}Controller
- Services: {Entity}Service (interface) + {Entity}ServiceImpl
- Repositories: {Entity}Repository
- DTOs: Create{Entity}Request, Update{Entity}Request, {Entity}Response
- Events: {Entity}{Action}Event (e.g., OrderCreatedEvent)
- Exceptions: {Entity}{Reason}Exception (e.g., OrderNotFoundException)

## Test Naming
- Unit: {Class}Test
- Integration: {Class}IntegrationTest
- Method: should_{expectedBehavior}_when_{condition}

## Dependencies
- Validation: Jakarta Bean Validation
- Mapping: MapStruct (preferred) or static factory methods
- Testing: JUnit 5 + Mockito + AssertJ + Testcontainers
- Database: PostgreSQL + Flyway
- Messaging: Spring Kafka
```

Commands can reference this implicitly because Claude Code reads nearby files for context when executing in the `.claude/commands/` directory.

### User-Level Library for Cross-Project Commands

Your personal productivity commands go in `~/.claude/commands/`:

```
~/.claude/commands/
├── morning-standup.md         # Daily standup prep
├── end-of-day.md              # EOD summary and commit
├── review-my-prs.md           # Check all my open PRs across repos
├── learn-this.md              # Explain a concept with Java examples
├── write-adr.md               # Generate an Architecture Decision Record
└── estimate-task.md           # Break down and estimate a task
```

**`~/.claude/commands/estimate-task.md`:**
```markdown
Break down and estimate this task: $ARGUMENTS

1. Analyse the current codebase to understand the scope
2. Break into subtasks (max 4 hours each)
3. For each subtask:
   - Description of what changes
   - Files affected
   - Estimated time (in hours)
   - Risk level (Low/Medium/High) with explanation
   - Dependencies on other subtasks

4. Total estimate with a confidence range:
   - Optimistic (everything goes smoothly)
   - Realistic (normal pace with minor issues)
   - Pessimistic (unexpected complexity, unfamiliar area)

5. Suggest which subtasks can be parallelised across team members

Format as a table for easy copy-paste into Jira/Linear.
```

---

## 8. Try This Exercises

### Exercise 1: Build a Custom Review Command

Create `.claude/commands/review-pr-checklist.md` that reviews the current branch against a Spring Boot PR checklist:
- Are new endpoints documented with OpenAPI annotations?
- Do all new public methods have Javadoc?
- Are database changes backward-compatible?
- Is there adequate test coverage?
- Are new configuration properties documented in README?

Test it by making a small change and running `/project:review-pr-checklist`.

### Exercise 2: Create a Test Generation Command

Create `.claude/commands/generate-integration-test.md` that accepts a controller class name as `$ARGUMENTS` and generates:
- A `@SpringBootTest` integration test
- Uses `Testcontainers` for PostgreSQL
- Tests all endpoints in the controller
- Includes happy path and error scenarios
- Uses `RestAssured` or `MockMvc` (match existing project style)

Test with: `/project:generate-integration-test OrderController`

### Exercise 3: Session Management Drill

Practice built-in commands in a single session:
1. Start Claude Code in your project
2. Run `/status` -- note your model and token count
3. Ask Claude Code to explain a complex service class
4. Run `/cost` -- note the tokens consumed
5. Run `/compact Keep the explanation of OrderService`
6. Run `/cost` again -- compare token usage
7. Run `/fast`, ask a simple question, then `/fast` to switch back

### Exercise 4: Compose a Full Feature Workflow

Create three commands that work together:
1. `.claude/commands/plan-feature.md` -- Takes a feature description, analyses the codebase, and produces an implementation plan (no code changes)
2. `.claude/commands/implement-step.md` -- Takes a step number from the plan and implements it
3. `.claude/commands/verify-feature.md` -- Runs tests, checks code quality, and verifies the feature works

Use them in sequence:
```
> /project:plan-feature Add order cancellation with refund
> /project:implement-step Step 1: Add CANCELLED status and state machine validation
> /project:implement-step Step 2: Add refund service integration
> /project:verify-feature order cancellation with refund
```

### Exercise 5: Team Skills Library

Create a `.claude/commands/` directory with at least 5 commands tailored to your project. Commit them to your repo with a README that documents each command's purpose, syntax, and example usage. Have a teammate clone and try them out -- iterate based on their feedback.

---

## 9. Tips and Gotchas

### Do's

| Tip | Why |
|-----|-----|
| Keep custom command files focused on one task | Multi-purpose commands produce inconsistent results. One command, one job. |
| Use `$ARGUMENTS` for the variable part only | The command file should encode the fixed process; `$ARGUMENTS` supplies the target. |
| Include output format instructions | "Format as a markdown table" or "Output as a checklist" gives consistent, usable results. |
| Reference project conventions explicitly | "Follow the existing patterns in this codebase" helps Claude Code match your style. |
| Use `/compact` proactively | Don't wait for degraded responses. Compact when you shift to a new task within the same session. |
| Run `/doctor` when things feel off | It catches stale auth tokens, broken MCP connections, and config issues quickly. |
| Version your commands in git | They evolve with your codebase. PRs can include command updates alongside code changes. |

### Don'ts

| Gotcha | Details |
|--------|---------|
| Don't put secrets in command files | These are committed to git. Use `$ARGUMENTS` or environment variables for sensitive values. |
| Don't make commands too long | Commands over 2000 words dilute focus. Split into smaller, composable commands instead. |
| Don't rely on command execution order | Each `/project:*` invocation is independent. State from one does not carry to the next unless it is in files on disk. |
| Don't forget `/clear` resets everything | After `/clear`, Claude Code has no memory of the conversation. But it reloads CLAUDE.md and memory files, so persistent instructions survive. |
| Don't use `/compact` without guidance | Bare `/compact` may discard details you need. Always specify what to preserve: `/compact Keep the migration plan and list of affected tables`. |

### Common Mistakes

**Mistake 1: Overly vague custom commands**
```markdown
# BAD: Too vague
Review the code and make it better.
```
```markdown
# GOOD: Specific checklist
Review the code for:
1. Null safety (use Optional, @NonNull)
2. Resource leaks (try-with-resources for streams, connections)
3. Thread safety (mutable shared state in @Service beans)
...
```

**Mistake 2: Not using $ARGUMENTS when you should**
```markdown
# BAD: Hardcoded target
Review the OrderService class.
```
```markdown
# GOOD: Parameterised
Review the service class: $ARGUMENTS
```

**Mistake 3: Commands that both plan AND execute**
```markdown
# BAD: Plans and executes in one shot (risky for large changes)
Refactor the payment module to use Strategy pattern. Make all changes now.
```
```markdown
# GOOD: Plan first, execute separately
Create a refactoring plan for: $ARGUMENTS
Do NOT make any changes. Present the plan for approval.
```

**Mistake 4: Ignoring the context window in long sessions**
```
# After an hour of back-and-forth, Claude Code's responses degrade.
# You keep asking follow-ups and getting confused answers.

# FIX: Compact or clear
> /compact Keep the implementation plan for the refund feature
> Now let's implement step 3 of the plan.
```

### Performance Tips

1. **Use `/fast` for reconnaissance**: When you are exploring the codebase and asking "what does this do?" questions, the fast model is sufficient and cheaper.
2. **Switch to the full model for generation**: When you need Claude Code to write code, refactor, or make complex decisions, use the default (or explicitly set) model.
3. **Compact before complex tasks**: Free up context window space before asking Claude Code to tackle a large, multi-file change.
4. **Keep commands under 500 words**: Shorter commands produce more focused results. If you need more, link to conventions in CLAUDE.md rather than repeating them.

---

## Quick Reference Card

```
BUILT-IN COMMANDS (session management):
  /help          /clear         /compact [hint]    /config
  /cost          /doctor        /init              /login
  /logout        /memory        /model [name]      /permissions
  /review        /status        /terminal-setup    /vim
  /fast

BUNDLED SKILLS (agent-powered):
  /commit        /review-pr [#]    /simplify       /loop
  /claude-api    /update-config    /keybindings-help

CUSTOM COMMANDS:
  /project:*     .claude/commands/*.md     (project-level, committed)
  /user:*        ~/.claude/commands/*.md   (personal, all projects)

  File format:   Markdown with optional $ARGUMENTS placeholder
  Invocation:    /project:command-name [arguments]
```

---

*Next: [Tutorial 08 -- Daily Workflow Playbook](08-daily-workflow-playbook.md) for putting these skills into a daily routine.*
