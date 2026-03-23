# Tutorial 04: Claude Code Customisation

## Table of Contents

1. [CLAUDE.md Deep Dive](#1-claudemd-deep-dive)
2. [Hooks System](#2-hooks-system)
3. [MCP (Model Context Protocol) Servers](#3-mcp-model-context-protocol-servers)
4. [Custom Slash Commands and Skills](#4-custom-slash-commands-and-skills)
5. [Settings and Configuration](#5-settings-and-configuration)
6. [Permission Configuration](#6-permission-configuration)
7. [Keybindings](#7-keybindings)
8. [Practical Setup Examples](#8-practical-setup-examples)
9. [Try This Exercises](#9-try-this-exercises)
10. [Tips and Gotchas](#10-tips-and-gotchas)

---

## 1. CLAUDE.md Deep Dive

### Advanced CLAUDE.md Patterns

Beyond basic project info, CLAUDE.md can encode sophisticated conventions that dramatically improve Claude Code's output.

### Pattern: Architecture Decision Records (ADRs)

```markdown
## Architecture Decisions

### ADR-001: Use Hexagonal Architecture
- Domain logic must never depend on infrastructure
- All external dependencies accessed through ports (interfaces)
- Adapters implement ports and live in the infrastructure package

### ADR-002: Event Sourcing for Order Aggregate
- Order state is derived from events, not stored directly
- Events stored in order_events table
- Snapshots created every 100 events for performance

### ADR-003: CQRS for Read-Heavy Endpoints
- Write operations go through the domain model
- Read operations use dedicated read models (projections)
- Read models stored in separate tables with denormalised data
```

### Pattern: Error Handling Contract

```markdown
## Error Handling

### Exception Hierarchy
```
BaseException (abstract)
├── ValidationException (400)
│   ├── InvalidInputException
│   └── BusinessRuleViolationException
├── NotFoundException (404)
│   ├── OrderNotFoundException
│   └── CustomerNotFoundException
├── ConflictException (409)
│   └── DuplicateOrderException
└── InfrastructureException (500)
    ├── DatabaseException
    └── ExternalServiceException
```

### Rules
- Never catch generic Exception unless re-throwing
- Always log at ERROR level before throwing InfrastructureException
- Business exceptions carry a machine-readable error code
- Error response format: RFC 7807 ProblemDetail
```

### Pattern: Code Generation Templates

```markdown
## Code Templates

### New Service Class
When creating a new service, follow this template:
```java
@Service
@RequiredArgsConstructor
@Slf4j
public class {Name}Service {

    private final {Name}Repository repository;
    private final ApplicationEventPublisher eventPublisher;

    @Transactional(readOnly = true)
    public {Name}DTO findById(Long id) {
        return repository.findById(id)
            .map({Name}Mapper::toDTO)
            .orElseThrow(() -> new {Name}NotFoundException(id));
    }

    @Transactional
    public {Name}DTO create(Create{Name}Request request) {
        log.info("Creating {name}: {}", request);
        var entity = {Name}Mapper.toEntity(request);
        var saved = repository.save(entity);
        eventPublisher.publishEvent(new {Name}CreatedEvent(saved.getId()));
        return {Name}Mapper.toDTO(saved);
    }
}
```

### New Integration Test
```java
@SpringBootTest(webEnvironment = RANDOM_PORT)
@Testcontainers
@ActiveProfiles("test")
class {Name}IntegrationTest {

    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16-alpine");

    @Autowired
    TestRestTemplate restTemplate;

    @DynamicPropertySource
    static void configureProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
    }

    // tests here
}
```
```

### Pattern: Module-Specific CLAUDE.md

Place CLAUDE.md files in subdirectories for module-specific context:

```
project-root/
├── CLAUDE.md                    # Project-wide conventions
├── order-service/
│   ├── CLAUDE.md                # Order service specifics
│   └── src/
├── payment-service/
│   ├── CLAUDE.md                # Payment service specifics
│   └── src/
└── infrastructure/
    ├── CLAUDE.md                # Terraform/K8s conventions
    └── terraform/
```

**`infrastructure/CLAUDE.md` example:**
```markdown
# Infrastructure Module

## Terraform Conventions
- All resources must have tags: environment, team, service, managed-by
- Use modules from our internal registry for standard resources
- State stored in Azure Blob Storage (configured per environment)
- Variable naming: snake_case
- Resource naming: {service}-{environment}-{resource_type}
- Always include cost estimation comments for expensive resources

## Kubernetes Conventions
- All deployments must have resource limits
- Use our base Helm chart (charts/base-service)
- Health checks: readiness at /actuator/health/readiness, liveness at /actuator/health/liveness
- HPA configured for CPU > 70%
```

---

## 2. Hooks System

### What are Hooks?

Hooks let you run custom code at specific points in Claude Code's workflow. They are scripts that execute before or after certain actions.

### Hook Types

| Hook | Trigger | Use Case |
|------|---------|----------|
| **PreToolUse** | Before Claude Code uses a tool (Read, Edit, Bash, etc.) | Validation, logging, blocking dangerous operations |
| **PostToolUse** | After Claude Code uses a tool | Notifications, logging, post-processing |
| **UserPromptSubmit** | When you submit a prompt | Input preprocessing, routing |
| **Stop** | When Claude Code finishes a response | Cleanup, notifications |

### Configuration

Hooks are configured in your settings files:

```json
// ~/.claude/settings.json (global hooks)
// or .claude/settings.json (project-level hooks)
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "/path/to/pre-bash-hook.sh"
          }
        ]
      }
    ],
    "PostToolUse": [
      {
        "matcher": "Edit",
        "hooks": [
          {
            "type": "command",
            "command": "/path/to/post-edit-hook.sh"
          }
        ]
      }
    ],
    "Stop": [
      {
        "matcher": "",
        "hooks": [
          {
            "type": "command",
            "command": "/path/to/on-stop.sh"
          }
        ]
      }
    ]
  }
}
```

### Hook Input/Output

Hooks receive JSON on stdin and can output JSON on stdout.

**PreToolUse hook input:**
```json
{
  "session_id": "abc123",
  "tool_name": "Bash",
  "tool_input": {
    "command": "rm -rf /tmp/test"
  }
}
```

**PreToolUse hook output (to block):**
```json
{
  "decision": "block",
  "reason": "Dangerous rm -rf command detected"
}
```

**PreToolUse hook output (to allow):**
```json
{
  "decision": "allow"
}
```

### Practical Hook Examples

**Hook 1: Block dangerous commands**

```bash
#!/bin/bash
# pre-bash-safety.sh — Block dangerous Bash commands

INPUT=$(cat)
COMMAND=$(echo "$INPUT" | jq -r '.tool_input.command')

# Block dangerous patterns
if echo "$COMMAND" | grep -qE 'rm -rf /|DROP TABLE|TRUNCATE|--force|--hard'; then
    echo '{"decision": "block", "reason": "Blocked: potentially dangerous command"}'
    exit 0
fi

echo '{"decision": "allow"}'
```

**Hook 2: Log all file edits**

```bash
#!/bin/bash
# post-edit-logger.sh — Log all file edits for audit

INPUT=$(cat)
FILE=$(echo "$INPUT" | jq -r '.tool_input.file_path // "unknown"')
TIMESTAMP=$(date -u +"%Y-%m-%dT%H:%M:%SZ")

echo "$TIMESTAMP | EDIT | $FILE" >> ~/.claude/edit-audit.log
```

**Hook 3: Auto-format after edits**

```bash
#!/bin/bash
# post-edit-format.sh — Run formatter after Java file edits

INPUT=$(cat)
FILE=$(echo "$INPUT" | jq -r '.tool_input.file_path // ""')

if [[ "$FILE" == *.java ]]; then
    # Run Google Java Format on the edited file
    google-java-format --replace "$FILE" 2>/dev/null
fi
```

**Hook 4: Send Slack notification on completion**

```bash
#!/bin/bash
# on-stop-notify.sh — Notify via Slack when Claude Code finishes

INPUT=$(cat)
STOP_REASON=$(echo "$INPUT" | jq -r '.stop_reason // "unknown"')

if [ "$STOP_REASON" = "end_turn" ]; then
    curl -s -X POST "$SLACK_WEBHOOK_URL" \
      -H 'Content-Type: application/json' \
      -d "{\"text\": \"Claude Code finished a task in $(basename $PWD)\"}" \
      > /dev/null 2>&1
fi
```

---

## 3. MCP (Model Context Protocol) Servers

### What is MCP?

The Model Context Protocol (MCP) is an open standard that allows AI assistants like Claude Code to interact with external tools and data sources. MCP servers expose capabilities (tools, resources, prompts) that Claude Code can use.

Think of MCP as a plugin system for Claude Code.

### How MCP Works

```
Claude Code ──── MCP Protocol ──── MCP Server ──── External System
                (JSON-RPC)         (Docker,         (Docker daemon,
                                    DB, GitHub,       PostgreSQL,
                                    etc.)             GitHub API)
```

### Configuring MCP Servers

MCP servers are configured in your settings:

```json
// ~/.claude/settings.json (global)
// or .claude/settings.json (project-level)
{
  "mcpServers": {
    "docker": {
      "command": "docker",
      "args": ["run", "-i", "--rm", "mcp/docker"],
      "env": {}
    },
    "postgres": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-postgres"],
      "env": {
        "DATABASE_URL": "postgresql://user:pass@localhost:5432/mydb"
      }
    },
    "github": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": {
        "GITHUB_TOKEN": "ghp_your_token"
      }
    }
  }
}
```

### Useful MCP Servers for Backend Developers

#### Docker MCP Server

Lets Claude Code interact with Docker directly.

```json
{
  "mcpServers": {
    "docker": {
      "command": "docker",
      "args": ["run", "-i", "--rm",
        "-v", "/var/run/docker.sock:/var/run/docker.sock",
        "mcp/docker"
      ]
    }
  }
}
```

**What you can do:**
```
> List all running containers
> Show the logs for the order-service container
> What's the resource usage of our Docker containers?
> Build and run the Dockerfile in the current directory
```

#### PostgreSQL MCP Server

Direct database access from Claude Code.

```json
{
  "mcpServers": {
    "postgres": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-postgres"],
      "env": {
        "DATABASE_URL": "postgresql://dev:devpass@localhost:5432/orderdb"
      }
    }
  }
}
```

**What you can do:**
```
> Show me the schema for the orders table
> What indexes exist on the orders table?
> Run an EXPLAIN ANALYZE on this query: SELECT * FROM orders WHERE status = 'PENDING'
> How many orders were created in the last 24 hours?
```

#### GitHub MCP Server

GitHub integration beyond git commands.

```json
{
  "mcpServers": {
    "github": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": {
        "GITHUB_TOKEN": "ghp_your_token"
      }
    }
  }
}
```

**What you can do:**
```
> Show me all open PRs assigned to me
> What are the latest issues labelled 'bug'?
> Create an issue for the performance problem we found
```

#### Filesystem MCP Server

Extended filesystem operations.

```json
{
  "mcpServers": {
    "filesystem": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-filesystem", "/Users/naresh/projects"]
    }
  }
}
```

#### Slack MCP Server

```json
{
  "mcpServers": {
    "slack": {
      "command": "npx",
      "args": ["-y", "@anthropic/mcp-server-slack"],
      "env": {
        "SLACK_BOT_TOKEN": "xoxb-your-token"
      }
    }
  }
}
```

**What you can do:**
```
> Post a message to #dev-team about the deployment
> What were the recent messages in #incidents?
```

### Verifying MCP Server Status

After configuring, start Claude Code and check:

```
> What MCP tools are available?
```

Claude Code will list all connected MCP servers and their available tools.

---

## 4. Custom Slash Commands and Skills

### Custom Slash Commands

You can create custom slash commands by placing markdown files in the `.claude/commands/` directory.

### Directory Structure

```
project-root/
└── .claude/
    └── commands/
        ├── review-security.md
        ├── generate-dto.md
        ├── check-coverage.md
        └── deploy-check.md
```

### Creating a Custom Command

**`.claude/commands/review-security.md`:**
```markdown
Review the current changes for security vulnerabilities.

Check for:
1. SQL injection (raw queries, string concatenation in queries)
2. XSS vulnerabilities in any response bodies
3. Missing authentication/authorization checks
4. Hardcoded secrets or API keys
5. Insecure deserialization
6. Missing input validation
7. SSRF vulnerabilities
8. Improper error handling that leaks internal details
9. Missing rate limiting on public endpoints
10. Insecure direct object references (IDOR)

For each issue found, provide:
- Severity (Critical/High/Medium/Low)
- File and line number
- Description of the vulnerability
- Concrete fix with code example

Format the output as a security review report.
```

**Usage:**
```
> /review-security
```

**`.claude/commands/generate-dto.md`:**
```markdown
Generate DTOs for the entity class specified by the user argument: $ARGUMENTS

For the given entity, create:
1. A request DTO (Create{Entity}Request) with validation annotations
2. A response DTO ({Entity}Response) as a Java record
3. An update DTO (Update{Entity}Request) with optional fields
4. A mapper class ({Entity}Mapper) using static methods

Follow these conventions:
- DTOs go in the dto/ package alongside the entity
- Use Jakarta Validation annotations (@NotNull, @NotBlank, @Size, etc.)
- Response DTOs should be Java records
- Mappers should be utility classes with static methods
- Never expose entity IDs in request DTOs
- Always include audit fields (createdAt, updatedAt) in response DTOs
```

**Usage:**
```
> /generate-dto Order
> /generate-dto Customer
```

**`.claude/commands/deploy-check.md`:**
```markdown
Perform a pre-deployment checklist for the current branch:

1. Check that all tests pass (run mvn test)
2. Verify no TODO or FIXME comments in changed files
3. Check for any hardcoded environment-specific values
4. Verify database migrations are forwards-compatible
5. Check that API changes are backward compatible
6. Verify Docker build succeeds
7. Check Kubernetes manifests for resource limits
8. Verify health check endpoints are configured
9. Check that logging levels are appropriate for production
10. Verify no debug/test code is committed

Report findings as a go/no-go checklist.
```

### User-Level Custom Commands

For personal commands that apply to all projects:

```
~/.claude/commands/
├── morning-standup.md
├── review-my-prs.md
└── end-of-day.md
```

**`~/.claude/commands/morning-standup.md`:**
```markdown
Help me prepare for the morning standup:

1. Show me what I committed yesterday (git log --since="yesterday" --author="naresh")
2. Show me any open PRs I need to review
3. Check if there are any failing CI builds on my branches
4. List any TODO comments I added recently

Summarise as bullet points I can read at standup:
- Yesterday: [what I did]
- Today: [based on open PRs and TODOs]
- Blockers: [any failing builds or issues]
```

---

## 5. Settings and Configuration

### Settings File Locations

| File | Scope | Shared? |
|------|-------|---------|
| `~/.claude/settings.json` | Global (all projects) | No (personal) |
| `<project>/.claude/settings.json` | Project-level | Yes (committed to repo) |
| `<project>/.claude/settings.local.json` | Project-level, local only | No (gitignored) |

### Complete Settings Reference

```json
// ~/.claude/settings.json
{
  // Permission configuration
  "permissions": {
    "allow": [
      "Read",
      "Glob",
      "Grep",
      "Bash(git *)",
      "Bash(mvn *)",
      "Bash(gradle *)",
      "Bash(docker *)",
      "Bash(kubectl get *)",
      "Bash(kubectl describe *)"
    ],
    "deny": [
      "Bash(rm -rf *)",
      "Bash(git push --force *)",
      "Bash(kubectl delete *)",
      "Bash(docker system prune *)"
    ]
  },

  // MCP servers
  "mcpServers": {
    // ... (see MCP section above)
  },

  // Hooks
  "hooks": {
    // ... (see Hooks section above)
  },

  // Environment variables for Claude Code
  "env": {
    "JAVA_HOME": "/usr/lib/jvm/java-21",
    "MAVEN_OPTS": "-Xmx2g"
  }
}
```

### Project Settings (Committed to Repo)

```json
// .claude/settings.json
{
  "permissions": {
    "allow": [
      "Bash(./gradlew *)",
      "Bash(docker compose *)"
    ]
  },
  "mcpServers": {
    // Project-specific MCP servers
  }
}
```

### Local Settings (Not Committed)

```json
// .claude/settings.local.json
{
  "mcpServers": {
    "postgres": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-postgres"],
      "env": {
        "DATABASE_URL": "postgresql://myuser:mypassword@localhost:5432/localdb"
      }
    }
  }
}
```

Use `.claude/settings.local.json` for anything that contains secrets or local paths.

---

## 6. Permission Configuration

### Understanding Permissions

Permissions control which tools Claude Code can use without asking. They use glob-style matching.

### Permission Patterns

```json
{
  "permissions": {
    "allow": [
      "Read",                      // All file reads
      "Glob",                      // File searching
      "Grep",                      // Content searching
      "Edit",                      // All file edits
      "Bash(git status)",          // Exact command
      "Bash(git diff *)",          // Wildcard match
      "Bash(mvn *)",               // All Maven commands
      "Bash(./gradlew *)",         // All Gradle commands
      "Bash(docker compose *)",    // Docker Compose commands
      "Bash(curl *)",              // HTTP requests
      "Bash(jq *)"                 // JSON processing
    ],
    "deny": [
      "Bash(rm -rf *)",            // Block recursive delete
      "Bash(git push --force *)",  // Block force push
      "Bash(git reset --hard *)",  // Block hard reset
      "Bash(kubectl delete *)",    // Block K8s deletions
      "Bash(docker rm -f *)",      // Block force container removal
      "Bash(DROP *)",              // Block SQL drops
      "Bash(TRUNCATE *)"           // Block SQL truncates
    ]
  }
}
```

### Recommended Permission Sets

**Conservative (new users):**
```json
{
  "permissions": {
    "allow": [
      "Read",
      "Glob",
      "Grep"
    ]
  }
}
```

**Developer (daily use):**
```json
{
  "permissions": {
    "allow": [
      "Read",
      "Glob",
      "Grep",
      "Edit",
      "Bash(git *)",
      "Bash(mvn *)",
      "Bash(./gradlew *)",
      "Bash(docker compose up *)",
      "Bash(docker compose down *)",
      "Bash(docker compose logs *)"
    ],
    "deny": [
      "Bash(rm -rf *)",
      "Bash(git push --force *)"
    ]
  }
}
```

**Power user (experienced):**
```json
{
  "permissions": {
    "allow": [
      "Read",
      "Glob",
      "Grep",
      "Edit",
      "Bash(git *)",
      "Bash(mvn *)",
      "Bash(./gradlew *)",
      "Bash(docker *)",
      "Bash(kubectl *)",
      "Bash(terraform *)",
      "Bash(az *)",
      "Bash(curl *)",
      "Bash(jq *)"
    ],
    "deny": [
      "Bash(rm -rf /)",
      "Bash(git push --force main)",
      "Bash(kubectl delete namespace *)",
      "Bash(terraform destroy *)"
    ]
  }
}
```

---

## 7. Keybindings

### Default Keybindings

| Key | Action |
|-----|--------|
| `Enter` | Submit prompt |
| `Shift+Enter` | New line in prompt |
| `Ctrl+C` | Cancel current generation / exit |
| `Ctrl+L` | Clear screen |
| `Up/Down` | Navigate prompt history |
| `Escape` | Cancel current input / dismiss |
| `Tab` | Accept suggestion or autocomplete |

### Terminal Integration

Claude Code works best in terminals that support:
- 256 colours or TrueColor
- Unicode characters
- Mouse events (for interactive elements)

**Recommended terminals:**
- macOS: iTerm2, Warp, built-in Terminal
- Linux: Kitty, Alacritty, GNOME Terminal
- Windows: Windows Terminal (via WSL2)

---

## 8. Practical Setup Examples

### Complete Setup for a Spring Boot Team

**Step 1: Global settings (`~/.claude/settings.json`):**
```json
{
  "permissions": {
    "allow": [
      "Read", "Glob", "Grep", "Edit",
      "Bash(git *)",
      "Bash(mvn *)",
      "Bash(./gradlew *)",
      "Bash(docker *)",
      "Bash(kubectl get *)",
      "Bash(kubectl describe *)",
      "Bash(kubectl logs *)"
    ],
    "deny": [
      "Bash(rm -rf *)",
      "Bash(git push --force *)",
      "Bash(kubectl delete *)"
    ]
  }
}
```

**Step 2: Global CLAUDE.md (`~/.claude/CLAUDE.md`):**
```markdown
## Personal Preferences
- Use Java 21 features
- Concise, code-first responses
- Always include error handling
- Use constructor injection
- Prefer records for DTOs
- Test with JUnit 5 + Mockito + AssertJ + Testcontainers
```

**Step 3: Project CLAUDE.md (committed to repo):**
(See the comprehensive example in Section 1)

**Step 4: Project settings (`.claude/settings.json`, committed):**
```json
{
  "permissions": {
    "allow": [
      "Bash(./gradlew *)",
      "Bash(docker compose *)"
    ]
  }
}
```

**Step 5: Local settings (`.claude/settings.local.json`, gitignored):**
```json
{
  "mcpServers": {
    "postgres": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-postgres"],
      "env": {
        "DATABASE_URL": "postgresql://dev:devpass@localhost:5432/orderdb"
      }
    }
  }
}
```

**Step 6: Custom commands (`.claude/commands/`):**
```
.claude/commands/
├── review-security.md
├── generate-dto.md
├── deploy-check.md
└── check-coverage.md
```

**Step 7: Hooks (`.claude/settings.json`):**
```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Edit",
        "hooks": [
          {
            "type": "command",
            "command": "bash -c 'FILE=$(cat | jq -r \".tool_input.file_path // empty\"); [ -n \"$FILE\" ] && [[ \"$FILE\" == *.java ]] && google-java-format --replace \"$FILE\" 2>/dev/null; true'"
          }
        ]
      }
    ]
  }
}
```

### Database Query Workflow with MCP

Once PostgreSQL MCP is configured:

```
> Show me the schema for the orders table

> How many orders per status in the last 7 days?

> Run EXPLAIN ANALYZE on this query and suggest index improvements:
> SELECT o.* FROM orders o JOIN customers c ON o.customer_id = c.id
> WHERE c.email LIKE '%@example.com' AND o.status = 'PENDING'
> ORDER BY o.created_at DESC LIMIT 50

> Based on the query plan, create a Flyway migration to add
> the recommended indexes.
```

### Docker Workflow with MCP

Once Docker MCP is configured:

```
> List all running containers and their health status

> Show me the logs from the order-service container (last 100 lines)

> The payment-service container keeps restarting. Show me
> the last few restarts and the error logs.

> Build the Dockerfile in the current directory and tag it
> as order-service:dev
```

---

## 9. Try This Exercises

### Exercise 1: Create Project CLAUDE.md
For one of your Spring Boot projects, create a comprehensive CLAUDE.md that includes:
- Architecture description
- Coding standards
- Build commands
- Test patterns
- API conventions

Start a new Claude Code session and verify it follows the conventions.

### Exercise 2: Set Up Custom Commands
Create at least three custom slash commands:
1. A security review command
2. A DTO generator command
3. A pre-commit checklist command

Test each one on your codebase.

### Exercise 3: Configure MCP Servers
Set up at least one MCP server:
- Docker MCP (easiest to start with)
- PostgreSQL MCP (if you have a local database)
- GitHub MCP (for PR management)

Verify the connection by asking Claude Code to use the MCP tools.

### Exercise 4: Create Safety Hooks
Write a PreToolUse hook that:
1. Blocks any Bash command containing `--force` or `-rf /`
2. Logs all commands to an audit file
3. Test it by asking Claude Code to run a blocked command

### Exercise 5: Team Configuration
Create a `.claude/` directory structure suitable for your team:
- `settings.json` with shared permissions
- Custom commands for common team workflows
- A CLAUDE.md with your team's coding standards

---

## 10. Tips and Gotchas

### Tips

1. **Start with CLAUDE.md, then add incrementally.** Get the basics working before adding hooks, MCP servers, and custom commands.

2. **Commit `.claude/settings.json` and `.claude/commands/` to your repo.** The whole team benefits from shared configuration.

3. **Use `.claude/settings.local.json` for secrets.** Add it to `.gitignore` immediately.

4. **Custom commands are markdown, not scripts.** They are prompts that Claude Code executes, not shell scripts. Think of them as saved prompt templates.

5. **MCP servers run as child processes.** They start when Claude Code starts and stop when it stops. They do not run as background daemons.

6. **Hooks should be fast.** They run synchronously. A slow hook makes every Claude Code action feel sluggish. Keep hooks under 1 second.

7. **Use deny permissions as guardrails.** Even if you trust yourself, deny rules prevent accidental damage from Claude Code's autonomy.

8. **Version your CLAUDE.md alongside your code.** As your architecture evolves, update CLAUDE.md. Review it during sprint retrospectives.

### Gotchas

1. **MCP servers need the tools installed.** The PostgreSQL MCP server needs `npx` available. Docker MCP needs Docker running. Verify prerequisites first.

2. **Hooks receive JSON and must output JSON.** If your hook outputs malformed JSON (or any extra text), Claude Code may ignore it or error.

3. **Custom commands inherit the current context.** A custom command run after a long conversation has access to all the context in that conversation.

4. **Setting priority: project > global.** If project settings conflict with global settings, project settings win. This can be confusing.

5. **MCP server credentials in settings files.** Even in `.local.json`, be careful with database passwords. Consider using environment variables referenced via `$ENV_VAR` syntax.

6. **Hook execution order matters.** Multiple hooks on the same trigger run sequentially. If the first hook blocks, subsequent hooks may not run.

7. **CLAUDE.md size affects token usage.** A very long CLAUDE.md consumes context window on every interaction. Keep it focused and concise (under 2000 words is a good target).

---

## Summary

| Concept | Key Takeaway |
|---------|-------------|
| CLAUDE.md | Encode architecture, conventions, and templates; use module-level files for specifics |
| Hooks | PreToolUse for safety, PostToolUse for formatting/logging, Stop for notifications |
| MCP Servers | Plugin system for Docker, databases, GitHub, Slack; configured in settings.json |
| Custom Commands | Markdown prompt templates in `.claude/commands/`; invoked with `/command-name` |
| Settings | Three levels: global, project (shared), local (secrets) |
| Permissions | Allow what you need, deny what is dangerous; use glob patterns |

**Next:** [Tutorial 05 — Claude Code in IDE](05-claude-code-in-ide.md)
