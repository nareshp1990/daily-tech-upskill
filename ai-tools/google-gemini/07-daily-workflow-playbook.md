# 07 — Daily Workflow Playbook

## Table of Contents

1. [Morning Routine](#1-morning-routine)
2. [During Development](#2-during-development)
3. [System Design and Architecture](#3-system-design-and-architecture)
4. [DevOps and Operations](#4-devops-and-operations)
5. [Research and Learning](#5-research-and-learning)
6. [Documentation](#6-documentation)
7. [API Exploration and Prototyping](#7-api-exploration-and-prototyping)
8. [Quick Reference Card](#8-quick-reference-card)
9. [Claude vs Copilot vs Gemini — Comprehensive Comparison](#9-claude-vs-copilot-vs-gemini--comprehensive-comparison)
10. [Try This Exercises](#10-try-this-exercises)

---

## 1. Morning Routine

### Email Triage with Gemini Workspace (5-10 min)

```
Step 1: Open Gmail → Gemini sidebar
Step 2: "Summarize my unread emails from the last 12 hours.
         Categorize as: Urgent, Action Required, FYI, Newsletter"
Step 3: For long threads → "Summarize this thread and list action items for me"
Step 4: Draft replies → "Write a reply confirming I'll review the PR by EOD"
```

### Code Review Queue (15-20 min)

```
Step 1: Open IDE (IntelliJ) with Gemini Code Assist
Step 2: For each PR:
  a. Open the diff
  b. Select changed code → /explain (for context)
  c. Select changed code → right-click → "Gemini: Review This"
  d. Check: bugs, security, performance, patterns
Step 3: For complex PRs:
  a. Copy the entire diff
  b. Open Claude Code → paste the diff → ask for comprehensive review
  c. Claude can see the full codebase context
```

### Standup Prep (5 min)

```
Open Gemini chat:
"Based on my git commits from yesterday (I worked on the order-service
retry logic and fixed the Kafka consumer offset issue), help me draft
a brief standup update: what I did, what I'm doing today, any blockers."
```

---

## 2. During Development

### Writing New Code

```
Primary tool: Gemini Code Assist (IntelliJ) for inline completions

Workflow:
1. Write the method signature + Javadoc
2. Let Gemini suggest the implementation (Tab to accept)
3. Review suggestion — modify as needed
4. For complex logic: open Gemini chat → describe what you need
5. For multi-file refactoring: switch to Claude Code in terminal
```

### Example: Building a New Feature

```
Task: Add rate limiting to the API gateway

Step 1 — Design (Gemini chat):
"Design a rate limiting solution for a Spring Boot API gateway.
Requirements: per-user, sliding window, Redis-backed, configurable
per endpoint. Include the class structure."

Step 2 — Generate config (Gemini Code Assist):
Write: // Spring Boot configuration for Redis-based rate limiter
→ Tab through completions

Step 3 — Implement core logic (Gemini Code Assist + Chat):
Write method signatures, let completions handle boilerplate
For complex algorithm: ask in chat, paste result, refine

Step 4 — Write tests (Claude Code):
"Write comprehensive tests for the RateLimiter class including:
edge cases, concurrent access, Redis failure fallback.
Use JUnit 5, Mockito, and Testcontainers for Redis."

Step 5 — Generate Kubernetes config (Gemini chat):
"Generate a ConfigMap for rate limiting thresholds per endpoint"
```

### Debugging

```
Scenario: Unexpected behavior in production

Tool selection:
├── Simple error → Gemini Code Assist chat (stay in IDE)
├── Complex error requiring log analysis → Claude Code (can read files)
├── Infrastructure error → Gemini in Cloud Console
└── Unknown root cause → AI Studio with full context (large input)

Step 1: Paste error + relevant code into Gemini chat
Step 2: If Gemini identifies the issue → fix it
Step 3: If not clear → switch to Claude Code:
  "Read src/main/java/com/.../OrderService.java and explain why
   processOrder might throw ConcurrentModificationException when
   called from the Kafka consumer"
Step 4: Claude Code can also read test files, configs, and logs
```

### Quick Code Tasks

| Task | Tool | How |
|------|------|-----|
| Generate a DTO from entity | Gemini Code Assist | Select entity → /generate "Create DTO" |
| Add validation annotations | Gemini Code Assist | Select fields → /transform "Add Bean Validation" |
| Convert to builder pattern | Gemini Code Assist | Select class → /transform "Add builder" |
| Write regex | Gemini chat | "Write regex for validating email with + aliases" |
| Optimize SQL query | Gemini chat | Paste query → "Optimize for MySQL 8" |
| Convert YAML to properties | Gemini chat | Paste YAML → "Convert to application.properties" |

---

## 3. System Design and Architecture

### Using Gemini's Large Context Window

Gemini 2.5 Pro's 1M token context window is uniquely suited for architecture analysis:

```
Step 1: Concatenate your project source into a text file
  find src -name "*.java" -exec cat {} + > codebase.txt

Step 2: Upload to AI Studio (or use context caching via API)

Step 3: Ask architecture questions:
  - "Identify all circular dependencies between services"
  - "Map out the data flow for a complete order lifecycle"
  - "Find all places where we violate the single responsibility principle"
  - "Generate a component diagram from this codebase"
  - "What would break if we split the user-service into auth and profile?"
```

### Architecture Reviews

```
Tool: Gemini Advanced (Gem: Spring Boot Architect)

"Review this architecture for a food delivery platform:
- 12 microservices on AKS
- MySQL for orders and users, MongoDB for menus
- Kafka for inter-service events
- Redis for caching and session
- Nginx ingress controller

Traffic: 5000 orders/hour peak, 500K daily active users

Evaluate: scalability, fault tolerance, observability, security,
and suggest improvements."
```

### Design Document Generation

```
Tool: Gemini in Google Docs

Step 1: Create a new Doc
Step 2: Use Gemini → "Generate a technical design document for..."
Step 3: Fill in each section with Gemini assistance:
  - Requirements (from your notes)
  - API design (generate from requirements)
  - Data model (generate schema)
  - Sequence diagrams (describe in Mermaid syntax)
  - Non-functional requirements (Gemini suggests based on scale)
  - Trade-offs and alternatives (Deep Research for current best practices)
```

---

## 4. DevOps and Operations

### Daily DevOps Tasks

| Task | Tool | Command/Prompt |
|------|------|----------------|
| Check failing CI builds | Gemini Code Assist | "Why is this GitHub Action failing?" |
| Generate Dockerfile | Gemini Code Assist | /generate "Production Dockerfile for Spring Boot" |
| Write K8s manifest | Gemini Cloud Console | "Generate deployment for order-service with HPA" |
| Troubleshoot pod issues | Gemini (GKE page) | "Why are my pods CrashLoopBackOff?" |
| Write Terraform | Gemini Code Assist | Inline completions for .tf files |
| Analyze logs | Gemini (Cloud Logging) | "Show me all OOM errors in last 24h" |
| Create alert | Gemini (Cloud Monitoring) | "Alert when error rate > 1% for 5 min" |

### Incident Response Workflow

```
Step 1: Alert fires (PagerDuty / Azure Monitor)
Step 2: Open Gemini in Cloud Console (or Azure Portal with Copilot)
  "Show me all errors for order-service in the last 30 minutes"
Step 3: Click Gemini on the error log
  → Get explanation + suggested fix
Step 4: If complex → open Claude Code:
  "I'm seeing this error in production: [paste error]
   Read the relevant service code and identify the root cause.
   Also check the recent git commits for related changes."
Step 5: Fix, deploy, verify
Step 6: Use Gemini in Docs to draft the incident report
```

### Terraform Workflow

```
Step 1: In IntelliJ with Gemini Code Assist, write Terraform:
  - Inline completions for resource blocks
  - Chat for complex modules

Step 2: Review with Claude Code:
  "Review my Terraform code in /infra/terraform/ for:
   security misconfigurations, missing tags, cost optimization,
   and best practices for Azure/GCP resources"

Step 3: In AI Studio, ask:
  "Given this Terraform code [paste], generate the corresponding
   documentation for our infrastructure wiki"
```

---

## 5. Research and Learning

### Learning New Technology

```
Tool: Gemini Deep Research + NotebookLM

Step 1: Deep Research
  "Research the state of gRPC in Spring Boot 3.x:
   setup guide, performance vs REST, bidirectional streaming,
   error handling, integration testing, production best practices"

Step 2: Upload research report to NotebookLM
  + Add official Spring gRPC documentation URLs
  + Add relevant blog posts

Step 3: Ask implementation questions:
  "How do I add gRPC to my existing Spring Boot REST service
   without breaking the existing REST endpoints?"

Step 4: Generate Audio Overview for commute

Step 5: Implement with Gemini Code Assist:
  Write the gRPC service definitions and let completions
  generate the Spring Boot integration code
```

### Staying Current

```
Weekly routine (30 min):

Monday: Deep Research — "What's new in the Java/Spring ecosystem this week?"
Wednesday: NotebookLM — Upload 2-3 technical blog posts, Q&A with them
Friday: Gemini Gem (System Design Coach) — Practice one system design question
```

---

## 6. Documentation

### API Documentation

```
Tool: Gemini Code Assist + Google Docs

Step 1: In IntelliJ, select your REST controller
Step 2: Gemini chat → "Generate OpenAPI documentation for this controller"
Step 3: For user-facing docs:
  Gemini in Docs → "Write developer-friendly API documentation for:
  [paste OpenAPI spec]
  Include: authentication, common use cases, code examples in Java and cURL"
```

### Runbook Generation

```
Tool: Gemini Gem (DevOps Troubleshooter)

"Generate a runbook for handling a database connection pool exhaustion
incident on our order-service (Spring Boot, HikariCP, MySQL, running on AKS).

Include:
1. How to detect (alerts, logs, metrics)
2. Immediate mitigation steps
3. Root cause investigation
4. Permanent fixes
5. Prevention measures
6. Relevant kubectl, mysql, and monitoring commands"
```

### Code Comments and Javadoc

```
Tool: Gemini Code Assist

Select a class → /doc → generates Javadoc for all public methods

Or in chat:
"Generate comprehensive Javadoc for this service class.
Include: class-level description, @param, @return, @throws,
and @see references to related classes"
```

---

## 7. API Exploration and Prototyping

### Prompt Prototyping in AI Studio

```
Workflow for building an AI feature:

Step 1: Open AI Studio
Step 2: Experiment with different prompts:
  - Try system instructions
  - Test with structured prompts (few-shot)
  - Evaluate different models (Pro vs Flash)
  - Test edge cases

Step 3: Once satisfied:
  - Click "Get Code" → cURL
  - Adapt to Spring AI ChatClient
  - Add error handling, retries, monitoring

Step 4: Iterate:
  - Monitor production outputs
  - Return to AI Studio to refine
  - Update prompt in config (no code changes)
```

### Prototyping Cycle

```
AI Studio (minutes)     →   Spring Boot service (hours)   →   Production (days)
├── Test prompt          ├── ChatClient integration         ├── Vertex AI endpoint
├── Try models           ├── Error handling                 ├── Monitoring
├── Evaluate quality     ├── Rate limiting                  ├── Cost tracking
├── Export code          ├── Tests                          ├── A/B testing
└── Get API key          └── Docker/K8s                     └── Model routing
```

---

## 8. Quick Reference Card

### Which Tool for What

| I need to... | Use |
|-------------|-----|
| Write code in IDE | Gemini Code Assist (inline completions) |
| Ask a quick code question | Gemini Code Assist (chat panel) |
| Refactor multiple files | Claude Code (terminal) |
| Debug with full codebase context | Claude Code |
| Prototype an AI prompt | Google AI Studio |
| Build a production AI service | Spring AI + Vertex AI |
| Research a technology deeply | Gemini Deep Research |
| Analyze specific documents | NotebookLM |
| Write documentation | Gemini in Google Docs |
| Plan capacity in spreadsheet | Gemini in Google Sheets |
| Present architecture | Gemini in Google Slides |
| Troubleshoot Kubernetes | Gemini in Cloud Console |
| Write Terraform | Gemini Code Assist (IDE) |
| Generate SQL queries | Gemini in BigQuery |
| Review security posture | Gemini in Security Command Center |
| Brainstorm architecture | Gemini Live (voice) |
| Learn while commuting | NotebookLM Audio Overview |
| Get real-time info | Gemini with Google Search grounding |

### Keyboard Shortcuts (IDE)

| Action | VS Code | IntelliJ |
|--------|---------|----------|
| Accept suggestion | `Tab` | `Tab` |
| Dismiss | `Esc` | `Esc` |
| Open Gemini chat | `Ctrl+Shift+P` → "Gemini" | `View → Gemini` |
| Explain selected code | Right-click → Gemini → Explain | Right-click → Gemini → Explain |
| Generate tests | Right-click → Gemini → Generate Tests | Right-click → Gemini → Generate Tests |

### Model Selection Cheat Sheet

| Need | Model | Why |
|------|-------|-----|
| Best quality, complex task | Gemini 2.5 Pro | Best reasoning |
| Good quality, fast | Gemini 2.5 Flash | Thinking budget control |
| Fast and cheap | Gemini 2.0 Flash | Production workloads |
| Cheapest possible | Gemini 2.0 Flash Lite | Batch processing |
| Image/audio output | Gemini 2.0 Flash | Multimodal generation |

---

## 9. Claude vs Copilot vs Gemini — Comprehensive Comparison

### Overall Comparison Matrix

| Dimension | Claude (Anthropic) | GitHub Copilot | Gemini (Google) |
|-----------|-------------------|----------------|-----------------|
| **Primary interface** | CLI (Claude Code), API, Web | IDE plugin, Web, CLI | IDE plugin, Web, API, Cloud Console |
| **Flagship model** | Claude Opus 4 | GPT-4.1 + Claude 3.5 | Gemini 2.5 Pro |
| **Fast model** | Claude Sonnet 4 | GPT-4.1-mini | Gemini 2.5 Flash |
| **Budget model** | Claude Haiku 3.5 | GPT-4.1-nano | Gemini 2.0 Flash Lite |

### IDE Experience

| Feature | Claude Code | GitHub Copilot | Gemini Code Assist |
|---------|------------|----------------|-------------------|
| Inline completions | No | Excellent | Very Good |
| Chat in IDE | No (terminal) | Yes (sidebar) | Yes (sidebar) |
| Multi-file editing | Yes (core strength) | Yes (Copilot Edits) | No |
| Terminal commands | Yes (runs them) | Yes (Copilot CLI) | No |
| Git operations | Yes (commits, branches) | Yes (PR summaries) | No |
| Test execution | Yes (runs tests) | No | No |
| Code explanation | Yes | Yes | Yes |
| Code generation | Yes (multi-file) | Yes (single file focus) | Yes (single file focus) |
| Codebase awareness | Full (reads any file) | @workspace (indexed) | Enterprise tier (indexed) |
| Autonomous mode | Yes (agent loop) | Yes (Copilot Workspace) | Limited |

### Coding Quality

| Task | Claude | Copilot | Gemini | Notes |
|------|--------|---------|--------|-------|
| Simple code completion | Good | Excellent | Very Good | Copilot has most training data for completions |
| Complex algorithms | Excellent | Good | Very Good | Claude Opus 4 edges out on reasoning |
| Spring Boot boilerplate | Excellent | Excellent | Very Good | All three know Spring well |
| Bug detection | Excellent | Good | Good | Claude's analysis is most thorough |
| Refactoring | Excellent | Good | Good | Claude Code can refactor across files |
| Test generation | Excellent | Very Good | Good | Claude Code also runs the tests |
| Kotlin/Gradle | Good | Very Good | Very Good | Copilot has slight edge |
| Terraform | Good | Very Good | Good (GCP bias) | Copilot best for Azure Terraform |
| SQL optimization | Good | Good | Very Good | Gemini's BigQuery integration helps |

### API and SDK

| Feature | Claude API | OpenAI API (Copilot backend) | Gemini API |
|---------|-----------|------------------------------|-----------|
| Spring AI support | Yes | Yes | Yes |
| Function calling | Yes (tool use) | Yes | Yes |
| Structured output | JSON mode | JSON schema | JSON schema |
| Streaming | Yes | Yes | Yes |
| Context window | 200K tokens | 1M tokens (GPT-4.1) | 1M tokens |
| Context caching | Yes (prompt caching) | No | Yes (75% discount) |
| Multimodal input | Text, image | Text, image | Text, image, audio, video |
| Image output | No | DALL-E | Yes (2.0 Flash) |
| Batch API | Yes | Yes | Yes (Vertex AI) |
| Free tier | Limited | No | Yes (generous) |
| Grounding (web search) | Via tool use | Via browsing | Native Google Search |

### Pricing Comparison

| Use Case | Claude | Copilot | Gemini |
|----------|--------|---------|--------|
| IDE assistant | Claude Code (usage-based or $20/mo via Max) | $10-39/user/mo | Free (individual) or $19-45/user/mo |
| API — cheap model | Haiku: $0.25/$1.25 per 1M tok | GPT-4.1-nano: $0.10/$0.40 | Flash Lite: $0.075/$0.30 |
| API — mid model | Sonnet: $3/$15 per 1M tok | GPT-4.1-mini: $0.40/$1.60 | 2.5 Flash: $0.15/$0.60 |
| API — flagship | Opus: $15/$75 per 1M tok | GPT-4.1: $2/$8 | 2.5 Pro: $1.25/$10 |
| Consumer chat (paid) | Claude Pro ($20/mo) | N/A | Gemini Advanced ($20/mo) |
| Enterprise platform | Anthropic API + custom | GitHub Enterprise | Vertex AI |

### Cloud Integration

| Feature | Claude | Copilot | Gemini |
|---------|--------|---------|--------|
| AWS integration | Bedrock | None | None |
| Azure integration | None | Azure OpenAI, Azure DevOps | None |
| GCP integration | Vertex AI (Model Garden) | None | Deep (console, GKE, logging, etc.) |
| Kubernetes assist | Via CLI | Via chat | Cloud Console + GKE native |
| Logging/monitoring | None | None | Cloud Logging, Monitoring native |
| Security analysis | None | Code scanning (Copilot) | Security Command Center |
| Terraform | Via CLI | Via IDE | Via IDE + Cloud Console |

### Unique Strengths

| Tool | Unique Strengths |
|------|-----------------|
| **Claude Code** | Agentic multi-file editing, terminal integration, runs tests and commands, reads entire codebase on demand, best instruction following |
| **GitHub Copilot** | Best inline completions (most training data), deep GitHub integration (PR reviews, issues), widest IDE support, enterprise code search |
| **Gemini** | 1M token context, native multimodal (video, audio), cheapest pricing, Google Search grounding, deep GCP integration, NotebookLM, Deep Research, Workspace integration, context caching |

### Weakness Comparison

| Tool | Weaknesses |
|------|-----------|
| **Claude Code** | No IDE inline completions, terminal-only (steeper learning curve), no cloud console integration |
| **GitHub Copilot** | No agentic multi-file editing (yet), no terminal command execution, no multimodal, no free tier |
| **Gemini** | Code Assist weaker than Copilot for completions, no agentic terminal mode, GCP-biased (limited Azure/AWS awareness), newer ecosystem (fewer community resources) |

### Decision Matrix: When to Use What

| Scenario | Best Choice | Runner-Up | Why |
|----------|-------------|-----------|-----|
| Writing Java code in IntelliJ | Copilot or Gemini | — | Best inline completions |
| Large refactoring (10+ files) | Claude Code | Copilot Workspace | True agentic editing |
| Quick code question in IDE | Copilot Chat or Gemini Chat | — | No context switch |
| Debugging with stack trace | Claude Code | Gemini Chat | Can read logs and run commands |
| Code review | Claude Code | Gemini 2.5 Pro | Best analysis depth |
| Analyzing a 500-page PDF | Gemini 2.5 Pro | — | 1M context + native PDF |
| Building AI-powered API | Gemini API (cheapest) | Claude API (best quality) | Cost vs quality trade-off |
| Prototype AI prompts | Google AI Studio | — | Free, visual, export to code |
| GCP infrastructure | Gemini Cloud Console | — | Native integration |
| Azure infrastructure | Copilot | Claude Code | Azure awareness |
| Research a technology | Gemini Deep Research | Claude Code + web search | Automated multi-step research |
| Learn from documents | NotebookLM | — | Source-grounded, audio summaries |
| Write documentation | Gemini Workspace | Claude Code | Integrated into Docs/Sheets |
| CI/CD troubleshooting | Claude Code | Copilot | Can read and fix pipeline files |
| System design practice | Gemini (Gem: System Design) | Claude | Persistent persona |
| Interview preparation | Claude | Gemini | Better conversation quality |
| Cost-sensitive batch processing | Gemini Flash Lite | — | Cheapest per token |
| Maximum accuracy, cost no object | Claude Opus 4 | Gemini 2.5 Pro | Best reasoning |

### Recommended Setup for Senior Backend Engineer

```
┌─────────────────────────────────────────────────────────┐
│                    YOUR DAILY TOOLKIT                     │
├─────────────────────────────────────────────────────────┤
│                                                          │
│  IntelliJ IDEA                                          │
│  ├── Gemini Code Assist (free) ──> inline completions   │
│  └── OR GitHub Copilot ($10/mo) ──> inline completions  │
│      (install both, use whichever is better per task)    │
│                                                          │
│  Terminal                                                │
│  └── Claude Code ──> multi-file editing, debugging,      │
│                      git operations, running tests        │
│                                                          │
│  Browser                                                 │
│  ├── Google AI Studio ──> prompt prototyping (free)      │
│  ├── Gemini Advanced ──> Deep Research, Gems ($20/mo)    │
│  ├── NotebookLM ──> document analysis (free)             │
│  └── Claude.ai ──> complex conversations                 │
│                                                          │
│  Google Workspace                                        │
│  ├── Docs + Gemini ──> documentation                     │
│  ├── Sheets + Gemini ──> capacity planning               │
│  └── Gmail + Gemini ──> email triage                     │
│                                                          │
│  Cloud Consoles                                          │
│  ├── GCP + Gemini ──> GCP infrastructure                 │
│  └── Azure + Copilot ──> Azure infrastructure            │
│                                                          │
│  API (Production Services)                               │
│  ├── Spring AI (abstraction layer)                       │
│  ├── Gemini API ──> cost-sensitive workloads             │
│  └── Claude API ──> quality-sensitive workloads          │
│                                                          │
│  Estimated Monthly Cost:                                 │
│  ├── Gemini Code Assist: $0 (free)                      │
│  ├── Claude Code: $20 (Max plan) or usage-based          │
│  ├── Gemini Advanced: $20 (optional, for Deep Research)  │
│  ├── API usage: variable                                 │
│  └── Total: $20-60/month for AI tools                    │
│                                                          │
└─────────────────────────────────────────────────────────┘
```

### Summary: One-Sentence Verdict

| Tool | Verdict |
|------|---------|
| **Claude Code** | Best for complex, multi-step engineering tasks that require codebase understanding and autonomous execution |
| **GitHub Copilot** | Best for inline code completions and staying in flow while writing code in the IDE |
| **Gemini** | Best value for money with the broadest feature set — from IDE to cloud console to research to workspace |

---

## 10. Try This Exercises

### Exercise 1: Morning Routine Trial (20 min)
1. Tomorrow morning, try the full routine:
   - Gmail triage with Gemini (5 min)
   - Code review with Gemini Code Assist (15 min)
2. Track time spent vs your normal routine
3. Note what was faster and what was not helpful

### Exercise 2: Feature Development with AI Tools (30 min)
1. Pick a small feature to build (e.g., a new endpoint)
2. Use this workflow:
   - Design in Gemini chat (5 min)
   - Write code with Gemini Code Assist completions (15 min)
   - Generate tests with Claude Code (10 min)
3. Track how much code you wrote vs how much AI wrote
4. Rate the quality: would you merge this code as-is?

### Exercise 3: Tool Comparison Sprint (20 min)
1. Pick a moderately complex task: "Generate a Spring Boot service that
   implements the Saga pattern for order processing with compensation"
2. Ask the same question to:
   - Gemini Code Assist (IDE chat)
   - Claude Code (terminal)
   - Google AI Studio (web)
3. Compare: code quality, completeness, explanation quality
4. Document your preference and why

### Exercise 4: Full DevOps Cycle (25 min)
1. Generate a Dockerfile with Gemini Code Assist
2. Generate Kubernetes manifests with Gemini chat
3. Generate Terraform with Gemini Code Assist
4. Review all generated code with Claude Code:
   "Review the Dockerfile, K8s manifests, and Terraform in /deploy/ for
   security issues, best practices, and production readiness"
5. Note which tool was best for generation vs review

### Exercise 5: Build Your Tool Matrix (15 min)
1. Open a spreadsheet (use Gemini in Sheets)
2. List your 20 most common daily tasks
3. For each task, assign the best AI tool
4. Create your personal "Which tool for what" reference card
5. Tape it to your monitor (or pin it in your notes app)

---

## Key Takeaways

1. **No single tool wins everything** — the best setup uses Claude Code + Gemini Code Assist (or Copilot) + AI Studio
2. **Claude Code** excels at complex, multi-step tasks that need codebase context
3. **Gemini Code Assist** (or Copilot) excels at keeping you in flow with inline completions
4. **Gemini's ecosystem** (Deep Research, NotebookLM, Workspace, Cloud Console) is unmatched in breadth
5. **Gemini API is the cheapest** — use it for cost-sensitive production workloads
6. **Spring AI** lets you switch between providers without code changes
7. **The 80/20 rule:** You will use inline completions 80% of the time — make sure those work great
8. **Invest time in Gems** — reusable personas save more time than one-off prompts
9. **Deep Research + NotebookLM** together replace hours of manual research weekly
10. **Budget approximately $20-60/month** for AI tools — ROI is 10x+ in saved time

---

## Final Checklist

Before you close this series, make sure you have:

- [ ] Google AI Studio account with API key
- [ ] Gemini Code Assist installed in IntelliJ (or VS Code)
- [ ] At least one Gem created (Code Reviewer or Spring Boot Architect)
- [ ] Tried Deep Research on one technical topic
- [ ] Uploaded documents to NotebookLM
- [ ] Made at least one API call to Gemini from Java
- [ ] Identified your top 5 daily tasks where AI helps most
- [ ] Created your personal "Which tool for what" reference card
- [ ] Decided on your monthly AI tool budget

---

Back to: [README — Tutorial Index](README.md)
