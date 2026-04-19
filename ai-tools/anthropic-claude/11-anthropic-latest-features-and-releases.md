# Tutorial 11: Anthropic Latest Features & Releases (2025–2026)

## Table of Contents

1. [Overview: Anthropic's 2025–2026 Product Evolution](#1-overview-anthropics-20252026-product-evolution)
2. [Claude Model Family Updates](#2-claude-model-family-updates)
3. [Claude Code: Major Feature Releases](#3-claude-code-major-feature-releases)
4. [Claude Cowork & Dispatch](#4-claude-cowork--dispatch)
5. [Claude Agent SDK](#5-claude-agent-sdk)
6. [Claude Code Plugins & Marketplace](#6-claude-code-plugins--marketplace)
7. [API & Platform New Features](#7-api--platform-new-features)
8. [MCP Ecosystem Expansion](#8-mcp-ecosystem-expansion)
9. [Claude Plans & Pricing Updates](#9-claude-plans--pricing-updates)
10. [Interactive Apps & Integrations](#10-interactive-apps--integrations)
11. [Timeline: Feature Launch Chronology](#11-timeline-feature-launch-chronology)
12. [What's Next: Roadmap Signals](#12-whats-next-roadmap-signals)
13. [Try This Exercises](#13-try-this-exercises)

---

## 1. Overview: Anthropic's 2025–2026 Product Evolution

Anthropic has transformed from an API-first model provider into a full-stack AI platform. Here's the high-level landscape of what shipped between late 2025 and March 2026:

| Category | Key Launches | Impact |
|----------|-------------|--------|
| **Models** | Opus 4.6, Sonnet 4.6, Haiku 4.5 | 1M context, stronger coding, extended thinking |
| **Developer Tools** | Agent Teams, Channels, Scheduled Tasks, Plugins | Claude Code becomes a full agentic platform |
| **Consumer/Enterprise** | Cowork, Dispatch, Projects, Interactive Apps | Desktop agent + mobile remote control |
| **SDK** | Claude Agent SDK (Python & TypeScript) | Programmatic agent building with full toolset |
| **API** | Web Search GA, Web Fetch, Files API, Skills API | Production-ready retrieval and file processing |
| **Ecosystem** | Plugin Marketplace, MCP integrations, Xcode | Broad IDE and tool ecosystem |

---

## 2. Claude Model Family Updates

### Current Model Lineup (April 2026)

| Model | ID | Context Window | Strengths | Pricing (Input/Output per MTok) |
|-------|-----|---------------|-----------|-------------------------------|
| **Claude Opus 4.7** ⭐ NEW | `claude-opus-4-7` | 1,000,000 tokens | Strongest coding (SWE-bench Pro 64.3%), adaptive thinking only, high-res vision, task budgets | $15 / $75 |
| **Claude Opus 4.6** | `claude-opus-4-6` | 1,000,000 tokens | Most capable — coding, planning, debugging, financial analysis, 14.5hr task horizon | $15 / $75 |
| **Claude Sonnet 4.6** | `claude-sonnet-4-6` | 200K (1M beta) | Balanced speed + intelligence, improved agentic search, fewer tokens consumed | $3 / $15 |
| **Claude Haiku 4.5** | `claude-haiku-4-5-20251001` | 200K | Fastest, near-frontier performance, real-time apps, cost-sensitive deployments | $0.80 / $4 |

### What Changed

**Opus 4.7 (April 16, 2026)** — GA:
- **SWE-bench Pro score: 64.3%** (up from 53.4% for Opus 4.6) — best coding performance yet
- **Adaptive thinking only** — `thinking: {type: "adaptive"}`; explicit `budget_tokens` removed (returns 400)
- **High-resolution vision** — supports images up to 2576px / 3.75MP (up from 1568px); pixel coordinates map 1:1 — no scale-factor math required
- **`xhigh` effort level** — new tier between `high` and `max`; recommended for coding and agentic tasks
- **Task budgets (beta)** — advisory token budget (`task_budget`) across a full agentic loop via beta header `task-budgets-2026-03-13`; model sees a running countdown; distinct from `max_tokens`
- **Enhanced vision capabilities** — low-level perception (pointing, measuring, counting), image localization, bounding-box detection
- **Breaking API changes vs 4.6:**
  - `temperature`, `top_p`, `top_k` non-default values now return 400 — omit entirely
  - Thinking content **omitted from responses by default** (set `"display": "summarized"` to restore visible reasoning)
  - New tokenizer: 1x–1.35x more tokens than Opus 4.6 (up to ~35% more, content-dependent)
- **Behavior changes** (prompt-tuning may be needed):
  - More literal instruction following at lower effort levels
  - Response length calibrates to task complexity rather than fixed verbosity
  - Fewer tool calls and subagents spawned by default
  - Real-time cybersecurity safeguards (Cyber Verification Program available for security research)

```java
// Using Opus 4.7 in Spring AI
@Bean
public ChatClient opus47Client(ChatClient.Builder builder) {
    return builder
        .defaultOptions(AnthropicChatOptions.builder()
            .model("claude-opus-4-7")
            .maxTokens(16384)
            // Do NOT set temperature/top_p — returns 400 error on 4.7
            .build())
        .build();
}
```

**Opus 4.6 (February 5, 2026)**:
- 1-million-token context window (up from 200K)
- Significantly stronger coding skills — better at multi-file refactors, debugging complex issues
- Improved planning and step-by-step reasoning
- Enhanced financial analysis and domain expertise
- 14.5-hour task completion time horizon for long-running agentic tasks
- Supports adaptive extended thinking (model decides when/how much to reason)

**Sonnet 4.6 (February 2026)**:
- Improved agentic search performance
- Consumes fewer tokens for equivalent tasks
- Extended thinking support
- 1M token context window (beta)

**Haiku 4.5 (October 2025)**:
- Most intelligent Haiku ever — near-frontier performance
- Ideal for real-time applications with latency-sensitive requirements

### Using in Spring AI

```java
@Configuration
public class ModelConfig {

    @Bean
    public ChatClient opusClient(ChatClient.Builder builder) {
        return builder
            .defaultOptions(AnthropicChatOptions.builder()
                .model("claude-opus-4-6")
                .maxTokens(8192)
                .build())
            .build();
    }

    @Bean
    public ChatClient sonnetClient(ChatClient.Builder builder) {
        return builder
            .defaultOptions(AnthropicChatOptions.builder()
                .model("claude-sonnet-4-6")
                .maxTokens(4096)
                .build())
            .build();
    }

    @Bean
    public ChatClient haikuClient(ChatClient.Builder builder) {
        return builder
            .defaultOptions(AnthropicChatOptions.builder()
                .model("claude-haiku-4-5-20251001")
                .maxTokens(2048)
                .build())
            .build();
    }
}
```

---

## 3. Claude Code: Major Feature Releases

### 3.1 Agent Teams (February 2026)

Agent Teams is the most significant Claude Code feature — fully independent Claude Code instances that communicate with each other via a shared mailbox.

**Key difference from subagents**:
- **Subagents**: Run within a single session, report back to the caller (worker → manager)
- **Agent Teams**: Fully independent sessions, message each other directly (colleague ↔ colleague)

**Enabling Agent Teams**:
```json
// In settings.json or .claude/settings.json
{
  "env": {
    "CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS": "1"
  }
}
```

**How it works**:
- Shared task list across all team members
- Built-in mailbox system for inter-agent communication
- Teammate lifecycle management (spawn, monitor, terminate)
- Each agent has its own context window and toolset
- Agents can work on different files/features simultaneously

**Practical example — parallel microservice development**:
```
You: "Refactor the order-service and payment-service to use the new event schema.
      Use agent teams — one agent per service."

Claude spawns:
  → Agent A: Works on order-service (own worktree)
  → Agent B: Works on payment-service (own worktree)
  → Both share a task list and communicate via mailbox
  → Results merged back when both complete
```

### 3.2 Channels (March 2026)

Channels let you push messages, alerts, and webhooks directly into a running Claude Code session from external platforms.

**Supported platforms** (research preview):
- Telegram
- Discord
- Custom HTTP endpoints

**What this enables**:
- Send Claude a message from your phone while it's running on your desktop
- Route CI/CD alerts directly into a Claude Code session
- Build custom notification pipelines that trigger Claude actions
- Two-way communication: Claude reads and responds through the same channel

**Setup**:
```bash
# Channels run through MCP servers
# Enable in Claude Code settings
claude --channels  # Start with channels enabled (research preview)
```

### 3.3 Scheduled Tasks (February 2026)

Save a prompt and have it run automatically on a recurring cadence — no more opening a new session every morning.

**Capabilities**:
- Full Claude Code session for each run (git access, MCP servers, tools)
- Recurring cadence configuration (hourly, daily, weekly, cron)
- Can create branches, commit changes, open PRs via GitHub MCP
- Access to all configured plugins and integrations

**Use cases for Java developers**:
```
# Daily error log review
Schedule: Every day at 9:00 AM
Prompt: "Check the error logs from the last 24 hours in the Spring Boot
         application. Summarise critical errors and create a GitHub issue
         for any new, untracked errors."

# Weekly dependency check
Schedule: Every Monday at 8:00 AM
Prompt: "Run ./gradlew dependencyUpdates. If there are security patches
         available, create a branch with the updates and open a PR."
```

### 3.4 Bare Mode

The `--bare` flag optimises Claude Code for scripted, non-interactive usage:

```bash
claude --bare -p "Analyse this codebase and output a JSON summary"
```

**What `--bare` skips**:
- Hooks
- LSP (Language Server Protocol)
- Plugin sync
- Skill directory walks
- Auto-memory (fully disabled)
- OAuth and keychain auth (requires `ANTHROPIC_API_KEY` or `apiKeyHelper`)

**When to use**: CI/CD pipelines, automated scripts, batch processing where you need raw speed.

### 3.5 Worktree Enhancements

```bash
# Start Claude Code in an isolated git worktree
claude --worktree
# or
claude -w
```

**New: Sparse checkout for monorepos**:
```json
// settings.json
{
  "worktree": {
    "sparsePaths": [
      "services/order-service/",
      "libs/common/",
      "build.gradle"
    ]
  }
}
```

This checks out only the directories you need — critical for large monorepos where a full checkout is expensive.

### 3.6 Enhanced Hooks

New hook types added to the existing system:

| Hook | Trigger | Use Case |
|------|---------|----------|
| `PreToolUse` | Before a tool runs | Validate, block, or modify tool calls |
| `PostToolUse` | After a tool completes | Log results, trigger follow-up actions |
| `Notification` | When Claude wants to notify user | Custom notification routing |
| **`Elicitation`** (new) | When Claude requests user input | Intercept and auto-respond to prompts |
| **`ElicitationResult`** (new) | After elicitation response | Override user responses programmatically |
| **`PostCompact`** (new) | After conversation compaction | Inject context, save summaries |

**Example: Auto-approve test runs**:
```json
// .claude/settings.json
{
  "hooks": {
    "Elicitation": [
      {
        "matcher": "run tests",
        "command": "echo '{\"approve\": true}'"
      }
    ]
  }
}
```

### 3.7 Auto Memory

Claude Code now maintains persistent project knowledge across sessions:

- Automatically saves important context (project patterns, decisions, architecture notes)
- Stored in `~/.claude/projects/<project>/memory/`
- Loaded at the start of each session
- Can be manually managed via `/memory` command
- Keeps an index file (`MEMORY.md`) for quick reference

### 3.8 Fast Mode (`/fast`) — April 2026

Fast mode enables faster output from Claude Opus 4.6 without downgrading to a smaller model.

```bash
# Toggle fast mode in a Claude Code session
/fast
```

- Uses Claude Opus 4.6 with optimised output speed (not a smaller model)
- Available exclusively on Opus 4.6 (not yet on 4.7)
- Ideal for quick edits, file reads, and repetitive tasks where full thinking depth is unnecessary
- Toggle on/off mid-session — persists until toggled again

### 3.9 Ultraplan (Research Preview — April 2026)

Ultraplan runs a planning agent in a remote cloud container while freeing your local terminal.

**Three modes:**
- **Simple** — Quick plan generation
- **Visual** — Web editor with inline comments, emoji reactions, and outline sidebar
- **Deep** — Comprehensive multi-angle planning using Opus 4.6 in a remote container (up to 30 minutes)

```bash
# Start Ultraplan from CLI
/ultraplan "Refactor the order-service to use event sourcing"
```

**Flow:**
1. Claude drafts a plan in a remote container
2. Review and annotate the plan in the web editor
3. Choose: run remotely (stays in container) or pull back locally
4. Auto-creates a cloud environment on first run (Team/Enterprise plans)

### 3.10 Auto Mode (GA for Max — April 2026)

Auto mode replaces manual permission prompts with an intelligent classifier:

- **Safe actions** run without interruption (file reads, searches, git status)
- **Risky actions** are automatically blocked (destructive deletes, force pushes)
- Middle ground between `--dangerously-skip-permissions` and full manual approval
- GA for Max subscribers (Opus 4.7); use `--auto` flag

```bash
claude --auto  # Enable auto mode
```

### 3.11 `/ultrareview` — Multi-Agent Cloud Code Review (Research Preview)

```bash
/ultrareview
```

- Runs a cloud-based two-stage multi-agent code review
  - Stage 1: Find potential bugs across the entire codebase
  - Stage 2: Confirm which bugs are real (separate verification agent)
- Far more thorough than single-agent review — catches issues that require cross-file reasoning
- Requires Team or Enterprise plan
- Results delivered as a structured report in the chat

### 3.12 New Commands & Features (2026)

| Command / Feature | Description |
|-------------------|-------------|
| `/team-onboarding` | Packages your Claude Code setup into a replayable onboarding guide for teammates |
| `/autofix-pr` | Enables PR auto-fix from the terminal |
| `/powerup` | Interactive animated lessons on Claude Code features |
| `/fewer-permission-prompts` | Scans session transcripts, proposes read-only command allowlist to reduce prompts |
| `/recap` | Generates a context summary when resuming a paused session |
| `/focus` | Toggles focus view (separates transcript from focus panel) |
| `/cost` (enhanced) | Now shows per-model breakdown and cache-hit rate |
| `Monitor` tool | Streams background script events live into the conversation; Claude tails logs and reacts |
| `PreCompact` hook | Blocking capability (exit code 2); background monitor support for plugins |
| `ENABLE_PROMPT_CACHING_1H` | Env var for extended 1-hour prompt cache TTL (beyond the standard 5-minute TTL) |
| Amazon Bedrock via Mantle | `CLAUDE_CODE_USE_MANTLE=1`; interactive Bedrock and Vertex AI setup wizards built in |
| PID namespace sandboxing | Linux: `CLAUDE_CODE_SUBPROCESS_ENV_SCRUB` for subprocess isolation |
| Read-only bash + globs | Bash commands with glob patterns that are read-only no longer trigger permission prompts |
| Plan file naming | Plan files auto-named after prompt content (e.g., `fix-auth-race-snug-otter.md`) |
| Push notification tool | Mobile alerts when Remote Control enabled |
| Windows PowerShell tool | Native PowerShell support (progressive rollout; toggle with `CLAUDE_CODE_USE_POWERSHELL_TOOL`) |

---

## 4. Claude Cowork & Dispatch

### 4.1 Claude Cowork (January 2026 — Research Preview)

Cowork transforms Claude from a chat assistant into a **desktop agent** that can automate multi-step tasks on your computer.

**What Cowork can do**:
- Navigate your file system and open applications
- Read and edit local files (spreadsheets, documents, code)
- Interact with desktop applications
- Execute multi-step workflows autonomously
- Access connected MCP integrations (Slack, Asana, Google Calendar, etc.)

**Available on**: Pro ($20/mo) and Max ($100–$200/mo) plans

**Projects in Cowork**:
- Attach existing local folders as project context
- Create new project folders during setup
- Context reusability across sessions
- Persistent memory within projects

### 4.2 Dispatch (March 17, 2026)

Dispatch is the mobile extension of Cowork — assign tasks to Claude from your phone and return to completed work on your desktop.

**How Dispatch works**:
```
1. Claude Desktop app running on your computer (macOS or Windows x64)
2. Claude mobile app on your phone
3. Pair the devices
4. Send tasks from phone → Claude executes on desktop
5. Return to your desktop to find work completed
```

**Key characteristics**:
- **Persistent thread**: Single conversation that doesn't reset between tasks
- **Cross-device**: Start on phone, finish on desktop (or vice versa)
- **Full access**: Claude can access local files, connected apps, and plugins on the desktop
- **Context retention**: Claude remembers previous tasks in the thread

**Practical examples**:
```
From your phone while commuting:
  "Prepare a summary of yesterday's git commits across all microservices"
  "Organise the Q2 planning spreadsheet by priority"
  "Analyse the error logs and draft an incident report"

Return to desktop → work is done.
```

**Requirements**:
- Latest Claude Desktop app (macOS or Windows x64)
- Claude mobile app
- Computer must be awake and app must be open
- Pro or Max subscription

---

## 5. Claude Agent SDK

### Overview

The Claude Agent SDK (formerly Claude Code SDK) enables you to programmatically build AI agents with Claude Code's full capabilities.

**Available in**:
- Python: `pip install claude-agent-sdk` (v0.2.81+)
- TypeScript/JavaScript: `npm install @anthropic-ai/claude-agent-sdk`

### Key Features

| Feature | Description |
|---------|-------------|
| **Built-in Tools** | Full Claude Code toolset (Read, Write, Edit, Bash, Glob, Grep) |
| **Custom Tools** | Define Python/TS functions as tools — runs as in-process MCP servers |
| **Hooks** | Intercept and modify agent behavior programmatically |
| **Adaptive Thinking** | Model decides when and how much to reason (Opus 4.6+) |
| **Token Budgets** | Set fixed thinking token limits or disable thinking |
| **Subagents** | Spawn child agents for parallel work |

### Python Example — Spring Boot Code Reviewer

```python
from claude_agent_sdk import ClaudeSDKClient, Tool

# Define a custom tool
@Tool(description="Run Spring Boot tests and return results")
def run_spring_tests(module: str) -> str:
    """Run tests for a specific Spring Boot module."""
    import subprocess
    result = subprocess.run(
        ["./gradlew", f":{module}:test"],
        capture_output=True, text=True, timeout=300
    )
    return f"Exit code: {result.returncode}\n{result.stdout}\n{result.stderr}"

# Create the agent
client = ClaudeSDKClient(
    model="claude-sonnet-4-6",
    tools=[run_spring_tests],
    thinking="adaptive",  # Opus 4.6+ feature
    max_tokens=8192
)

# Run the agent
result = client.run(
    prompt="""Review the recent changes in the order-service module.
    Run the tests first, then review code quality, security,
    and adherence to Spring Boot best practices."""
)

print(result.output)
```

### Xcode Integration (2026)

Apple's Xcode 26.3 natively integrates the Claude Agent SDK:
- Full Claude Code capabilities inside Xcode
- Subagents, background tasks, and plugins
- No separate IDE extension needed

---

## 6. Claude Code Plugins & Marketplace

### Official Plugin Marketplace (Early 2026)

Anthropic launched an official plugin directory with **72+ plugins across 24 categories**.

**What plugins bundle**:
- Slash commands
- Agent configurations
- MCP servers
- Hooks
- Skills

**Installing plugins**:
```bash
# Browse available plugins
claude plugins list

# Install a plugin
claude plugins install <plugin-name>

# Plugins from the official marketplace (claude-plugins-official)
# are automatically available
```

### Plugin Categories

| Category | Examples |
|----------|---------|
| **Development** | Code review, test generation, documentation |
| **DevOps** | Docker, Kubernetes, Terraform, CI/CD |
| **Database** | PostgreSQL, MongoDB, Redis tooling |
| **Cloud** | AWS, Azure, GCP management |
| **Productivity** | Slack, Asana, Jira, Google Calendar |
| **Financial** | 41 financial skills with MCP data integrations |
| **Knowledge Work** | 11 categories covering research, writing, analysis |
| **Biology/Science** | Specialised research plugins |

### Plugin Verification

| Badge | Meaning |
|-------|---------|
| **Anthropic Verified** | Reviewed by Anthropic for quality and safety |
| **Community** | Community-contributed, use at your own discretion |

### Creating Your Own Plugin

```bash
# Initialise a new plugin
mkdir my-spring-plugin && cd my-spring-plugin

# Create the plugin manifest
cat > .claude-plugin/manifest.json << 'EOF'
{
  "name": "spring-boot-toolkit",
  "version": "1.0.0",
  "description": "Spring Boot development utilities",
  "commands": ["commands/"],
  "mcp_servers": ["mcp/"],
  "hooks": ["hooks/"]
}
EOF
```

### Custom Marketplace

You can create and distribute your own plugin marketplace via a git repository:
```json
// .claude-plugin/marketplace.json
{
  "plugins": [
    {
      "name": "spring-boot-toolkit",
      "repo": "https://github.com/yourorg/spring-boot-toolkit",
      "description": "Spring Boot development utilities"
    }
  ]
}
```

---

## 7. API & Platform New Features

### 7.1 Web Search Tool (GA)

Previously in beta, web search is now generally available — no beta header required.

```java
// Spring AI with web search
@Bean
public ChatClient webSearchClient(ChatClient.Builder builder) {
    return builder
        .defaultOptions(AnthropicChatOptions.builder()
            .model("claude-sonnet-4-6")
            .tools(List.of(
                AnthropicTool.webSearch()
            ))
            .build())
        .build();
}
```

**New: Dynamic filtering** — uses code execution to filter results before they reach the context window, reducing token cost and improving relevance.

### 7.2 Web Fetch Tool (Beta)

Claude can now retrieve full content from specified web pages and PDF documents:

```java
// API request with web fetch
var response = anthropic.messages().create(
    MessageCreateParams.builder()
        .model("claude-sonnet-4-6")
        .tools(List.of(
            Tool.webFetch()
        ))
        .messages(List.of(
            Message.user("Fetch and summarise the Spring Boot 3.4 release notes
                          from the official documentation")
        ))
        .build()
);
```

**Free code execution**: API code execution is now free when used with web search or web fetch.

### 7.3 Files API & Skills API

**Files API** — Anthropic-managed skills for document processing:

| File Type | Capabilities |
|-----------|-------------|
| `.pptx` (PowerPoint) | Read, analyse, summarise presentations |
| `.xlsx` (Excel) | Parse spreadsheets, analyse data, generate charts |
| `.docx` (Word) | Read, edit, generate documents |
| `.pdf` | Parse, extract, summarise documents |

**Skills API** — Custom skills via `/v1/skills` endpoints:
```bash
# Create a custom skill
POST /v1/skills
{
  "name": "spring-boot-review",
  "description": "Review Spring Boot code for best practices",
  "instructions": "Check for proper use of dependency injection...",
  "tools": ["code_execution"]
}
```

### 7.4 Extended Thinking Display Control

New `display` field lets you omit thinking content from responses for faster streaming:

```java
// Omit thinking content but preserve signature for multi-turn
var response = anthropic.messages().create(
    MessageCreateParams.builder()
        .model("claude-opus-4-6")
        .thinking(ThinkingParams.builder()
            .type("enabled")
            .budgetTokens(10000)
            .display("omitted")  // New! Faster streaming
            .build())
        .messages(messages)
        .build()
);
// thinking blocks have empty `thinking` field but signature is preserved
// for multi-turn continuity
```

### 7.5 Data Residency Controls

Specify where model inference runs:
- **Default**: Anthropic's standard infrastructure
- **US-only inference**: Available at 1.1× pricing
- Useful for compliance requirements (HIPAA, SOC 2, government contracts)

---

## 8. MCP Ecosystem Expansion

### What's New in MCP (Model Context Protocol)

| Feature | Description |
|---------|-------------|
| **Interactive Dialogs** | MCP servers can request structured input mid-task (form fields or browser URL) |
| **Elicitation** | Servers can ask users questions during tool execution |
| **Plugin Bundling** | Plugins can bundle multiple MCP servers as a single install |
| **Channel Integration** | MCP powers the new Channels feature (Telegram, Discord) |
| **OAuth Support** | Streamlined authentication for third-party services |

### Official MCP Integrations

Anthropic has partnered with major platforms for first-party MCP support:

| Platform | Capabilities |
|----------|-------------|
| **Slack** | Read/send messages, manage channels |
| **Asana** | Create/manage tasks, projects |
| **Google Calendar** | View/create events, manage schedules |
| **Figma** | View designs, extract assets |
| **Box** | File management, document access |
| **Canva** | Create and edit designs |
| **monday.com** | Project management |
| **Hex** | Data analytics dashboards |
| **Intuit** | TurboTax, QuickBooks, Credit Karma, Mailchimp |

### Using MCP in Spring Boot Projects

```bash
# Add a database MCP server to your project
# .claude/settings.json
{
  "mcpServers": {
    "postgres": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-postgres"],
      "env": {
        "POSTGRES_URL": "postgresql://localhost:5432/mydb"
      }
    },
    "github": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": {
        "GITHUB_TOKEN": "${GITHUB_TOKEN}"
      }
    }
  }
}
```

---

## 9. Claude Plans & Pricing Updates

### Consumer Plans (March 2026)

| Plan | Price | Key Features |
|------|-------|-------------|
| **Free** | $0 | Limited usage, Sonnet 4.6, basic features |
| **Pro** | $20/mo | Full model access (Opus/Sonnet/Haiku), Cowork, Dispatch, Projects, ~45 msgs/5hr |
| **Max 5×** | $100/mo | 5× Pro usage (~225 msgs/5hr), priority access, early features |
| **Max 20×** | $200/mo | 20× Pro usage (~900 msgs/5hr), highest priority, all beta features |

### Enterprise Plans

| Plan | Features |
|------|----------|
| **Team** | $30/user/mo, Projects, admin controls, higher limits |
| **Enterprise** | Custom pricing, SSO, SCIM, data residency, audit logs |

### Claude Code Pricing

Claude Code usage is included in Max plans. For API usage:
- Billed per token (model-specific pricing)
- Code execution is **free** when used with web search or web fetch
- `--bare` mode reduces overhead costs (no hooks, no plugins)

---

## 10. Interactive Apps & Integrations

### Artifacts Evolution

Artifacts have evolved from simple code/text outputs to interactive applications:

| Capability | Description |
|------------|-------------|
| **Charts & Visualisations** | Custom charts, diagrams rendered inline |
| **MCP-Connected Apps** | Artifacts can read/write to external services |
| **Interactive Dashboards** | Live data from connected tools |
| **Design Canvases** | Create visual content inline |
| **Code Playgrounds** | Run and test code directly |

### Enterprise Integrations via Cowork

| Integration | What It Enables |
|-------------|----------------|
| **Slack** | Read channels, send messages, summarise threads |
| **Asana/monday.com** | Manage projects and tasks |
| **Google Workspace** | Access Docs, Sheets, Calendar, Gmail |
| **Figma/Canva** | View and interact with designs |
| **Hex** | Analytics dashboards and data exploration |
| **Box** | Enterprise file management |

---

## 11. Timeline: Feature Launch Chronology

| Date | Feature | Category |
|------|---------|----------|
| **Oct 2025** | Claude Haiku 4.5 | Model |
| **Nov 2025** | Skills API (beta), Agent Skills | API |
| **Dec 2025** | MCP Interactive Dialogs | Platform |
| **Jan 2026** | Claude Cowork (research preview) | Consumer |
| **Jan 2026** | Interactive Apps & MCP integrations | Consumer |
| **Jan 2026** | Claude Plugins marketplace | Developer |
| **Feb 2026** | Claude Opus 4.6 & Sonnet 4.6 | Model |
| **Feb 2026** | Agent Teams (experimental) | Developer |
| **Feb 2026** | Scheduled Tasks | Developer |
| **Feb 2026** | Auto Memory | Developer |
| **Feb 2026** | Parallel Agents, Worktree Sparse Checkout | Developer |
| **Feb 2026** | Web Search GA, Web Fetch beta | API |
| **Feb 2026** | Agent SDK rebrand (Code SDK → Agent SDK) | SDK |
| **Feb 2026** | Extended Thinking display control | API |
| **Mar 2026** | Dispatch for Cowork | Consumer |
| **Mar 2026** | Channels (Telegram, Discord) | Developer |
| **Mar 2026** | Bare mode (`--bare`) | Developer |
| **Mar 2026** | Elicitation & PostCompact hooks | Developer |
| **Mar 2026** | Data residency controls | Enterprise |
| **Mar 2026** | Xcode 26.3 Agent SDK integration | IDE |
| **Mar 2026** | Files API (PPTX, XLSX, DOCX) | API |
| **Apr 2026** | Claude Opus 4.7 (GA) | Model |
| **Apr 2026** | Fast mode (`/fast`) for Opus 4.6 | Developer |
| **Apr 2026** | Ultraplan (research preview) | Developer |
| **Apr 2026** | Auto mode (GA for Max) | Developer |
| **Apr 2026** | `/ultrareview` (multi-agent cloud review) | Developer |
| **Apr 2026** | Task budgets beta API | API |
| **Apr 2026** | Monitor tool, PreCompact hook | Developer |
| **Apr 2026** | `/team-onboarding`, `/powerup`, `/recap` | Developer |

---

## 12. What's Next: Roadmap Signals

Based on public statements, meetups, and research previews:

| Signal | Source | Expected |
|--------|--------|----------|
| **Swarming** | Claude Code Meetup (Tokyo) — "significant attention in 2026" | Mid 2026 |
| **Channels GA** | Currently research preview | Q2 2026 |
| **Agent Teams GA** | Currently experimental flag | Q2 2026 |
| **More MCP Integrations** | Anthropic blog — "expanding enterprise partnerships" | Ongoing |
| **Claude 5** | Industry speculation, benchmark leaks | H2 2026 |
| **Cowork GA** | Currently research preview | Q2 2026 |
| **Ultraplan GA** | Currently research preview | Q2–Q3 2026 |
| **`/ultrareview` GA** | Currently research preview (Team/Enterprise) | Q2–Q3 2026 |
| **Claude Haiku 4.7** | Likely next in the Haiku line | Q2–Q3 2026 |

---

## 13. Try This Exercises

### Exercise 1: Enable Agent Teams and Parallel Refactor

```bash
# 1. Enable agent teams
# Add to .claude/settings.json:
# { "env": { "CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS": "1" } }

# 2. Start Claude Code
claude

# 3. Ask for a parallel refactor:
# "Use agent teams to refactor the order-service and payment-service
#  simultaneously. Agent A handles order-service, Agent B handles
#  payment-service. Both should update the event DTOs to use the
#  new schema."
```

### Exercise 2: Set Up a Scheduled Task

```
# In Claude Code:
# "Create a scheduled task that runs every morning at 9 AM.
#  It should check for outdated dependencies in my Spring Boot
#  project and create a summary issue on GitHub."
```

### Exercise 3: Use Bare Mode in CI/CD

```yaml
# .github/workflows/claude-review.yml
name: Claude Code Review
on: [pull_request]
jobs:
  review:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Install Claude Code
        run: npm install -g @anthropic-ai/claude-code
      - name: Review PR
        env:
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
        run: |
          claude --bare -p "Review the changes in this PR. Focus on:
            1. Spring Boot best practices
            2. Security vulnerabilities
            3. Performance issues
            Output as a markdown summary."
```

### Exercise 4: Build a Custom Agent with Agent SDK

```python
# Install: pip install claude-agent-sdk
from claude_agent_sdk import ClaudeSDKClient, Tool

@Tool(description="Query the application database")
def query_db(sql: str) -> str:
    """Execute a read-only SQL query."""
    import psycopg2
    conn = psycopg2.connect("postgresql://localhost:5432/myapp")
    cur = conn.cursor()
    cur.execute(sql)
    return str(cur.fetchall())

client = ClaudeSDKClient(
    model="claude-sonnet-4-6",
    tools=[query_db]
)

result = client.run(
    prompt="Analyse the orders table. Find the top 10 customers by revenue "
           "and any anomalies in the last 30 days."
)
print(result.output)
```

### Exercise 5: Connect Dispatch and Work from Your Phone

```
1. Install Claude Desktop app on your Mac/PC
2. Install Claude mobile app on your phone
3. Pair the devices (Settings → Dispatch → Pair Device)
4. From your phone, send: "Check the Spring Boot error logs from today,
   summarise any critical issues, and draft a Slack message for the team"
5. Return to your desktop to review the results
```

---

## Related Resources

- [Anthropic Release Notes](https://docs.anthropic.com/en/release-notes/overview)
- [Claude Code Changelog](https://docs.anthropic.com/en/release-notes/claude-code)
- [Claude Code Docs — Agent Teams](https://code.claude.com/docs/en/agent-teams)
- [Claude Agent SDK — Python](https://github.com/anthropics/claude-agent-sdk-python)
- [Claude Plugins — Official Directory](https://github.com/anthropics/claude-plugins-official)
- [Claude Cowork Help Center](https://support.claude.com/en/articles/13947068)
- [Model Context Protocol](https://modelcontextprotocol.io/)
- [Claude Pricing](https://claude.com/pricing)

---

*Last updated: April 2026. Anthropic ships frequently — check the official release notes for the very latest.*
