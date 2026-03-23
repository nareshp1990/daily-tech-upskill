# Tutorial 02: Claude Code Setup and Basics

## Table of Contents

1. [What is Claude Code?](#1-what-is-claude-code)
2. [Installation](#2-installation)
3. [Authentication](#3-authentication)
4. [First Run](#4-first-run)
5. [Permission Modes](#5-permission-modes)
6. [Slash Commands Reference](#6-slash-commands-reference)
7. [Reading Files and Understanding Your Codebase](#7-reading-files-and-understanding-your-codebase)
8. [Editing Code](#8-editing-code)
9. [Running Commands and Tests](#9-running-commands-and-tests)
10. [CLAUDE.md — Your Project Brain](#10-claudemd--your-project-brain)
11. [Practical Examples](#11-practical-examples)
12. [Try This Exercises](#12-try-this-exercises)
13. [Tips and Gotchas](#13-tips-and-gotchas)

---

## 1. What is Claude Code?

Claude Code is Anthropic's official CLI tool that brings Claude directly into your terminal. It is an **agentic coding assistant** that can:

- Read and understand your entire codebase
- Edit multiple files in a single operation
- Run terminal commands (build, test, deploy)
- Create git commits and pull requests
- Maintain context about your project conventions via CLAUDE.md
- Work in the background while you do other things

Unlike the web interface, Claude Code operates directly on your local files and has full access to your development environment.

---

## 2. Installation

### Prerequisites

- **Node.js 18+** (check with `node --version`)
- **npm** or compatible package manager
- An **Anthropic API key** or **Claude Max/Team/Enterprise subscription**

### Install via npm

```bash
npm install -g @anthropic-ai/claude-code
```

### Verify Installation

```bash
claude --version
```

### Update to Latest

```bash
npm update -g @anthropic-ai/claude-code
```

### Alternative: Run Without Installing

```bash
npx @anthropic-ai/claude-code
```

### System Requirements

- macOS, Linux, or Windows (via WSL2)
- Terminal with ANSI colour support
- At least 4GB RAM available
- Git (for version control features)

---

## 3. Authentication

### Option A: Anthropic API Key

```bash
# Set via environment variable
export ANTHROPIC_API_KEY="sk-ant-api03-..."

# Or let Claude Code prompt you on first run
claude auth login
```

Add to your shell profile for persistence:

```bash
# ~/.zshrc or ~/.bashrc
export ANTHROPIC_API_KEY="sk-ant-api03-your-key-here"
```

### Option B: Claude Max Subscription

If you have a Claude Max subscription ($100/month or $200/month), you can authenticate via the web:

```bash
claude auth login
# This opens a browser window for OAuth authentication
```

This gives you Claude Code usage included in your subscription without separate API charges.

### Option C: Claude Team / Enterprise

For organisational accounts, authentication is managed by your admin. Typically:

```bash
claude auth login --org your-org-id
```

### Verify Authentication

```bash
claude auth status
```

### Switching Accounts

```bash
claude auth logout
claude auth login
```

---

## 4. First Run

### Starting Claude Code

```bash
# Navigate to your project
cd ~/projects/order-service

# Start Claude Code
claude
```

On first run in a project, Claude Code will:
1. Scan your project structure
2. Read key configuration files (pom.xml, package.json, etc.)
3. Look for CLAUDE.md files for project instructions
4. Present an interactive prompt

### The Interface

```
╭─────────────────────────────────────────────────────╮
│ Claude Code                                          │
│                                                      │
│ Working directory: ~/projects/order-service           │
│ Model: claude-opus-4-0520                            │
│                                                      │
│ Type your message or use /help for commands           │
╰─────────────────────────────────────────────────────╯

>
```

### Your First Interaction

```
> What does this project do? Summarise the architecture and key components.
```

Claude Code will read your project files and provide a comprehensive summary, including:
- Project purpose and structure
- Key dependencies (from pom.xml or build.gradle)
- Main packages and their responsibilities
- Configuration and profiles
- Test structure

### Single-Shot Mode

For quick, one-off tasks without entering the interactive shell:

```bash
# Ask a question
claude "What Spring profiles are defined in this project?"

# Pipe input
cat error.log | claude "What's causing this error?"

# Process a file
claude "Add input validation to src/main/java/com/example/OrderController.java"
```

---

## 5. Permission Modes

Claude Code has a permission system that controls what actions it can take without asking.

### Permission Levels

| Mode | Behaviour |
|------|-----------|
| **Ask** (default) | Claude asks permission before every file edit and command execution |
| **Auto-accept edits** | File edits are applied automatically; commands still require approval |
| **Auto-accept all** | Both edits and commands run without asking (use with caution) |

### Setting Permission Mode

```bash
# Start with auto-accept for edits
claude --auto-accept-edits

# Start with full auto-accept (edits + commands)
claude --auto-accept-all

# Or use the shorter flags
claude -ae   # auto-accept edits
claude -aa   # auto-accept all
```

### Per-Session Permission Toggles

During a session, you can adjust permissions:

```
> /permissions
```

### Allowed and Denied Tools

You can configure which tools Claude Code can use in your settings:

```json
// ~/.claude/settings.json
{
  "permissions": {
    "allow": [
      "Read",
      "Glob",
      "Grep",
      "Bash(git *)",
      "Bash(mvn test *)",
      "Bash(docker *)"
    ],
    "deny": [
      "Bash(rm -rf *)",
      "Bash(git push --force *)"
    ]
  }
}
```

### Recommendation for Daily Use

Start with the default **Ask** mode until you are comfortable with Claude Code's behaviour. Then move to **auto-accept edits** for faster workflows. Reserve **auto-accept all** for well-understood, low-risk tasks.

---

## 6. Slash Commands Reference

### Essential Commands

| Command | Description |
|---------|-------------|
| `/help` | Show all available commands and usage tips |
| `/clear` | Clear conversation history and start fresh |
| `/compact` | Compress conversation context to save tokens (keeps key information) |
| `/cost` | Show token usage and estimated cost for the current session |
| `/doctor` | Diagnose issues with your Claude Code installation |
| `/init` | Create a CLAUDE.md file for the current project |
| `/review` | Review code changes (staged, unstaged, or specific files) |
| `/commit` | Stage changes and create a git commit with a generated message |

### Navigation and Context

| Command | Description |
|---------|-------------|
| `/compact` | Summarise and compress the conversation context |
| `/clear` | Reset the conversation completely |

### Development Workflow

| Command | Description |
|---------|-------------|
| `/review` | Review current code changes for issues |
| `/commit` | Create a git commit with auto-generated message |

### Session Management

| Command | Description |
|---------|-------------|
| `/cost` | Display current session cost and token usage |
| `/doctor` | Check installation health |
| `/help` | Show help and available commands |

### Using Slash Commands

```
> /compact
Conversation compressed. Context reduced from 45K to 8K tokens.

> /cost
Session cost: $0.23
Input tokens: 15,432
Output tokens: 3,211
Cache read tokens: 12,100

> /review
Reviewing changes...
Found 3 modified files:
- src/main/java/com/example/OrderService.java (modified)
- src/main/java/com/example/OrderController.java (modified)
- src/test/java/com/example/OrderServiceTest.java (new)
[Detailed review follows...]

> /commit
Staged 3 files. Generated commit message:
"Add order validation and unit tests for OrderService"
Proceed? (y/n)
```

---

## 7. Reading Files and Understanding Your Codebase

### How Claude Code Reads Your Codebase

Claude Code does **not** read your entire codebase upfront. Instead, it uses an intelligent approach:

1. **Initial scan:** Reads project structure (directory tree), key config files (pom.xml, etc.)
2. **On-demand reading:** Reads specific files only when needed for the current task
3. **Glob and Grep:** Searches for files by pattern and content as needed
4. **Context management:** Keeps relevant file contents in conversation context

### Asking About Your Codebase

```
> Show me all REST controllers in this project

> What database migrations exist?

> Find all places where we use Kafka producers

> What dependencies are in our pom.xml?

> Show me the error handling strategy across services
```

### Searching for Patterns

Claude Code can search your codebase using powerful tools:

```
> Find all classes annotated with @Transactional

> Search for any hardcoded passwords or API keys

> Show me all TODO comments in the codebase

> Find all Spring Boot configuration properties we're using
```

### Reading Specific Files

```
> Read the OrderService.java file and explain the business logic

> Show me the Docker configuration

> What's in our application.yml?
```

---

## 8. Editing Code

### Basic Editing

```
> Add a @NotNull validation annotation to the email field in UserDTO

> Add a health check endpoint to the OrderController

> Fix the NPE in OrderService.processOrder by adding null checks
```

Claude Code will:
1. Read the relevant file(s)
2. Show you the proposed changes (diff view)
3. Ask for permission to apply (in Ask mode)
4. Apply the changes to your local files

### Multi-File Editing

```
> Create a new endpoint POST /api/orders/{id}/cancel that:
> 1. Adds the endpoint to OrderController
> 2. Adds cancelOrder method to OrderService
> 3. Adds CANCELLED to the OrderStatus enum
> 4. Adds a test for the cancellation flow
```

Claude Code handles multi-file edits as a coordinated operation, maintaining consistency across files.

### Reviewing Changes Before Applying

In the default Ask mode, Claude shows you each edit before applying:

```
Edit: src/main/java/com/example/OrderController.java

 @PostMapping("/{id}/cancel")
 public ResponseEntity<OrderDTO> cancelOrder(@PathVariable Long id) {
     OrderDTO cancelled = orderService.cancelOrder(id);
     return ResponseEntity.ok(cancelled);
 }

Apply this edit? (y/n/e)
  y = yes, apply
  n = no, skip
  e = edit the change before applying
```

---

## 9. Running Commands and Tests

### Running Shell Commands

Claude Code can execute terminal commands on your behalf:

```
> Run the tests for the order module

Claude Code will execute:
  mvn test -pl order-service

Allow? (y/n)
```

### Common Development Commands

```
> Build the project and show me any compilation errors

> Run just the integration tests

> Start the application locally with the 'dev' profile

> Check if Docker is running and list containers

> Run a Maven dependency tree to check for conflicts
```

### Test-Driven Development

```
> Write a test for OrderService.calculateTotal, then implement
> the method to make the test pass.
```

Claude Code will:
1. Create the test file
2. Run the test (it fails)
3. Implement the method
4. Run the test again (it passes)

---

## 10. CLAUDE.md — Your Project Brain

### What is CLAUDE.md?

CLAUDE.md is a special file that Claude Code reads automatically at the start of every session. It contains project-specific instructions, conventions, and context that shape how Claude Code works with your codebase.

Think of it as a persistent system prompt for your project.

### Where to Place CLAUDE.md

Claude Code looks for CLAUDE.md files in multiple locations, with later ones taking priority:

| Location | Scope | Purpose |
|----------|-------|---------|
| `~/.claude/CLAUDE.md` | Global (all projects) | Personal coding preferences |
| `<project-root>/CLAUDE.md` | Project-wide | Team coding standards, architecture |
| `<project-root>/<subdir>/CLAUDE.md` | Directory-specific | Module-specific conventions |

### Global CLAUDE.md Example

```markdown
# ~/.claude/CLAUDE.md

## Personal Preferences
- I prefer concise responses with code-first explanations
- Use Java 21 features (records, sealed interfaces, pattern matching)
- Always include error handling in code examples
- I use IntelliJ IDEA and the terminal
- Preferred test framework: JUnit 5 + Mockito + Testcontainers
```

### Project CLAUDE.md Example

```markdown
# CLAUDE.md — Order Service

## Project Overview
E-commerce order processing microservice handling 10K orders/minute.
Part of a larger microservices ecosystem with 12 services.

## Tech Stack
- Java 21, Spring Boot 3.2.3
- PostgreSQL 16 (Azure Database)
- Apache Kafka 3.6 (Azure Event Hubs)
- Docker + Kubernetes (AKS)
- Gradle 8.5

## Architecture
- Hexagonal architecture (ports and adapters)
- Domain layer: src/main/java/com/example/order/domain/
- Application layer: src/main/java/com/example/order/application/
- Infrastructure layer: src/main/java/com/example/order/infrastructure/
- API layer: src/main/java/com/example/order/api/

## Coding Standards
- Use records for DTOs and value objects
- Use sealed interfaces for domain events
- Constructor injection only (no @Autowired on fields)
- All public methods require Javadoc
- Use Optional<T> for nullable returns, never return null
- Use @Valid on all request bodies
- Log at INFO level for business events, DEBUG for technical detail
- Exception hierarchy: DomainException → specific exceptions

## Build and Test
- Build: `./gradlew build`
- Unit tests: `./gradlew test`
- Integration tests: `./gradlew integrationTest`
- Integration tests use Testcontainers (PostgreSQL, Kafka)
- Minimum test coverage: 80% (enforced by JaCoCo)

## Database
- Migrations managed by Flyway
- Migration files: src/main/resources/db/migration/
- Naming: V{number}__{description}.sql
- Always add both up and down migrations

## API Conventions
- REST endpoints under /api/v1/
- Use DTOs for request/response (never expose domain entities)
- Pagination via Spring Pageable (default page size: 20)
- Error responses follow RFC 7807 (Problem Details)

## Kafka Topics
- order.created (OrderCreatedEvent)
- order.updated (OrderUpdatedEvent)
- order.cancelled (OrderCancelledEvent)
- Use Avro schema for events (schemas in /schemas/)

## Git Conventions
- Branch naming: feature/TICKET-123-short-description
- Commit messages: "TICKET-123: Verb in present tense"
- Always squash merge to main
```

### Creating CLAUDE.md with /init

```
> /init

This will analyse your project and generate a CLAUDE.md file with:
- Detected tech stack and dependencies
- Project structure overview
- Build and test commands
- Suggested coding conventions
```

### CLAUDE.md Best Practices

1. **Keep it focused.** Include what Claude Code needs to know, not everything about the project.
2. **Update it regularly.** As conventions change, update CLAUDE.md.
3. **Include build commands.** Claude Code uses these to verify its changes compile.
4. **Specify naming conventions.** Package structure, class naming, file naming.
5. **Document non-obvious patterns.** Custom annotations, base classes, utility methods that Claude should use.
6. **Commit it to version control.** The whole team benefits from consistent AI assistance.

---

## 11. Practical Examples

### Example 1: Explore a New Codebase

```
> I just cloned this repo. Give me a complete overview:
> 1. What does the project do?
> 2. What's the tech stack?
> 3. How is the code organised?
> 4. How do I build and run it?
> 5. What are the key entry points?
```

### Example 2: Add a New REST Endpoint

```
> Add a new endpoint GET /api/v1/orders/search that accepts:
> - query (String, optional) — searches order description
> - status (OrderStatus, optional) — filters by status
> - fromDate/toDate (LocalDate, optional) — date range filter
> - Pageable for pagination
>
> Follow the existing patterns in the codebase for the controller,
> service, and repository layers. Include validation and tests.
```

### Example 3: Debug a Failing Test

```
> The test OrderServiceTest.shouldCalculateOrderTotal is failing.
> Run it, show me the error, and fix the issue.
```

### Example 4: Review Before Committing

```
> /review

> I'm about to commit these changes. Review them for:
> 1. Any bugs or edge cases I missed
> 2. Missing test coverage
> 3. Performance concerns
> 4. Security issues

> /commit
```

### Example 5: Refactor with Confidence

```
> Refactor the OrderService to use the Strategy pattern for
> order processing. Currently there's a big switch statement
> in processOrder() that handles different order types.
> Keep all tests passing.
```

---

## 12. Try This Exercises

### Exercise 1: Setup and Explore
1. Install Claude Code
2. Navigate to one of your Spring Boot projects
3. Run `claude`
4. Ask: "Summarise this project's architecture and identify any code smells"

### Exercise 2: Create a CLAUDE.md
1. Run `/init` in your project
2. Review the generated CLAUDE.md
3. Add your team's specific coding standards
4. Start a new session and verify Claude Code follows the conventions

### Exercise 3: Test-Driven Feature
1. Ask Claude Code to write a test for a new feature first
2. Run the test (it should fail)
3. Ask Claude Code to implement the feature
4. Run the test again (it should pass)
5. Ask Claude Code to review the implementation

### Exercise 4: Code Review Workflow
1. Make some changes to a file manually (introduce a subtle bug)
2. Run `/review`
3. See if Claude Code catches the bug
4. Fix it and use `/commit` to commit

### Exercise 5: Cost Awareness
1. Start a session and work on a task for 15 minutes
2. Run `/cost` to see your usage
3. Run `/compact` and check cost again
4. Compare the before/after to understand token consumption

---

## 13. Tips and Gotchas

### Tips

1. **Start every project session with `/init` (once) to create CLAUDE.md.** This dramatically improves Claude Code's understanding of your project.

2. **Use `/compact` proactively.** Long conversations accumulate tokens. Compact before you hit context limits, not after.

3. **Be specific in your requests.** "Fix the bug" is vague. "Fix the NPE in OrderService.java line 45 where the customer address can be null" is precise.

4. **Let Claude Code run tests.** After making changes, always ask Claude Code to run the relevant tests. It can fix issues in a tight feedback loop.

5. **Use single-shot mode for quick tasks.** `claude "What Spring profiles exist in this project?"` is faster than starting an interactive session.

6. **Pipe error output directly.** `mvn test 2>&1 | claude "Fix these test failures"` is incredibly efficient.

7. **Check cost regularly.** `/cost` keeps you aware of spending. Opus is powerful but expensive.

### Gotchas

1. **Claude Code operates on your local files.** Unlike a web chat, edits are real. Use git to protect yourself — commit or stash before letting Claude Code make large changes.

2. **Auto-accept all is risky.** Claude Code might run commands like `mvn clean` or `docker-compose down` that affect your running environment. Use with care.

3. **Large projects slow down initial scanning.** If your project has thousands of files, the first query may take longer as Claude Code indexes the structure.

4. **CLAUDE.md in the repo is visible to the whole team.** Do not put personal preferences in the repo-level CLAUDE.md. Use `~/.claude/CLAUDE.md` for personal settings.

5. **Context window limits apply.** Even with 200K tokens, very large codebases cannot be held in memory entirely. Claude Code reads files on demand and drops old context.

6. **Git operations require a clean working directory.** `/commit` works best when you have a clear understanding of what is staged vs unstaged. Use `git status` before `/commit`.

7. **Network interruptions lose context.** If your connection drops mid-conversation, you may lose the current context. Use `/compact` periodically to create save points.

8. **Claude Code cannot access remote services.** It cannot SSH into servers, query production databases, or call APIs. It works within your local environment.

---

## Summary

| Concept | Key Takeaway |
|---------|-------------|
| Installation | `npm install -g @anthropic-ai/claude-code` |
| Auth | API key via env var or Claude Max subscription via OAuth |
| Permissions | Start with Ask mode, graduate to auto-accept edits |
| Slash Commands | `/review`, `/commit`, `/compact`, `/cost` are your essentials |
| CLAUDE.md | One file to rule them all — put your conventions here |
| Reading Code | Claude Code reads on-demand, not upfront |
| Editing | Shows diffs, asks permission, can handle multi-file edits |

**Next:** [Tutorial 03 — Claude Code Advanced Workflows](03-claude-code-advanced-workflows.md)
