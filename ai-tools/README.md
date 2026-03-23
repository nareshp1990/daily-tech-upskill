# AI Tools Mastery Guide

A comprehensive tutorial series for mastering AI-powered developer tools — tailored for a senior backend developer working with Java/Spring Boot, microservices, DevOps, and cloud infrastructure.

---

## Tool Ecosystems

### 1. [Anthropic Claude](anthropic-claude/README.md)
Claude AI models, Claude Code (CLI), Claude in IDE, Claude API + Spring AI, CI/CD automation.

| # | Tutorial | Focus |
|---|----------|-------|
| 1 | [Models & Capabilities](anthropic-claude/01-claude-models-and-capabilities.md) | Opus/Sonnet/Haiku, pricing, when to use which |
| 2 | [Claude Code — Setup & Basics](anthropic-claude/02-claude-code-setup-and-basics.md) | Installation, commands, CLAUDE.md, permissions |
| 3 | [Claude Code — Advanced Workflows](anthropic-claude/03-claude-code-advanced-workflows.md) | Multi-file edits, git, plan mode, headless, CI/CD |
| 4 | [Claude Code — Customisation](anthropic-claude/04-claude-code-customisation.md) | Hooks, MCP servers, custom commands, settings |
| 5 | [Claude Code in IDE](anthropic-claude/05-claude-code-in-ide.md) | VS Code, IntelliJ, terminal workflows |
| 6 | [Claude API & Spring AI](anthropic-claude/06-claude-api-and-spring-ai.md) | Messages API, SDK, Spring AI integration |
| 7 | [Claude in CI/CD & Automation](anthropic-claude/07-claude-in-ci-cd-and-automation.md) | GitHub Actions, PR review, security scanning |
| 8 | [Daily Workflow Playbook](anthropic-claude/08-daily-workflow-playbook.md) | Practical routines, cheat sheet, power tips |

### 2. [GitHub Copilot](github-copilot/README.md)
Code completions, Copilot Chat, CLI, PR automation, advanced patterns.

| # | Tutorial | Focus |
|---|----------|-------|
| 1 | [Setup & Configuration](github-copilot/01-copilot-setup-and-configuration.md) | Plans, IDE setup, custom instructions |
| 2 | [Code Completion Mastery](github-copilot/02-code-completion-mastery.md) | Ghost text, shortcuts, context, practical examples |
| 3 | [Copilot Chat & Edits](github-copilot/03-copilot-chat-and-edits.md) | Chat, slash commands, inline edits, multi-file |
| 4 | [Copilot CLI](github-copilot/04-copilot-cli.md) | gh copilot suggest/explain, shell integration |
| 5 | [Copilot for Pull Requests](github-copilot/05-copilot-for-pull-requests.md) | PR summaries, auto review, Copilot Workspace |
| 6 | [Advanced Patterns & Tips](github-copilot/06-advanced-patterns-and-tips.md) | Prompt engineering, TDD, extensions, security |
| 7 | [Daily Workflow Playbook](github-copilot/07-daily-workflow-playbook.md) | Routines, cheat sheet, Copilot vs Claude comparison |

### 3. [Google Gemini](google-gemini/README.md)
Gemini models, Code Assist, API + SDK, AI Studio, Vertex AI, cloud DevOps.

| # | Tutorial | Focus |
|---|----------|-------|
| 1 | [Models & Capabilities](google-gemini/01-gemini-models-and-capabilities.md) | 2.5 Pro/Flash, 1M context, multimodal, pricing |
| 2 | [Gemini in IDE](google-gemini/02-gemini-in-ide.md) | Code Assist in VS Code/IntelliJ, completions, chat |
| 3 | [Gemini API & SDK](google-gemini/03-gemini-api-and-sdk.md) | Java SDK, Spring AI, function calling, grounding |
| 4 | [AI Studio & Vertex AI](google-gemini/04-google-ai-studio-and-vertex-ai.md) | Prompt design, fine-tuning, RAG, Agent Builder |
| 5 | [Gemini for Cloud & DevOps](google-gemini/05-gemini-for-cloud-and-devops.md) | GKE, logging, BigQuery, Terraform, Azure mapping |
| 6 | [Advanced Features](google-gemini/06-gemini-advanced-features.md) | Gems, Deep Research, NotebookLM, Workspace |
| 7 | [Daily Workflow Playbook](google-gemini/07-daily-workflow-playbook.md) | Routines, Claude vs Copilot vs Gemini matrix |

---

## Recommended Learning Path

```
Week 1: Pick your primary tool and go deep
  → Start with Anthropic Claude (tutorials 1–4) or GitHub Copilot (tutorials 1–3)

Week 2: Expand to API integration and automation
  → Claude API + Spring AI (tutorial 6) or Gemini API + SDK (tutorial 3)
  → CI/CD automation (Claude tutorial 7, Copilot tutorial 5)

Week 3: Add the second tool and compare
  → Learn the tool you didn't start with
  → Read the comparison in each playbook (tutorial 7/8)

Week 4: Optimise your daily workflow
  → Read all three Daily Workflow Playbooks
  → Build your personalised multi-tool setup
```

---

## Quick Decision Guide

| Task | Best Tool |
|------|-----------|
| Writing new code (inline completions) | GitHub Copilot |
| Complex multi-file changes | Claude Code |
| Code review and PR automation | Claude Code or Copilot |
| System design discussions | Claude (large context) or Gemini (1M context) |
| API/SDK integration in Spring Boot | Claude API (Spring AI) or Gemini API |
| DevOps (Docker, K8s, Terraform) | Claude Code or Copilot CLI |
| Research and learning | Gemini Deep Research, NotebookLM |
| Google Cloud operations | Gemini Code Assist |

---

*These tutorials are independent of the daily learning plans and can be explored at your own pace.*
