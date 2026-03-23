# Mastering the Anthropic Claude Ecosystem

A comprehensive tutorial series for senior backend developers working with Java/Spring Boot, DevOps, and cloud-native technologies.

---

## Learning Path

```
Beginner ──────────────────────────────────────────────── Advanced

01 Models &        02 Claude Code      03 Advanced         04 Customisation
Capabilities ───── Setup & Basics ──── Workflows ──────── & MCP Servers
                                                               │
05 IDE             06 API &            07 CI/CD &          08 Daily Workflow
Integration ────── Spring AI ──────── Automation ──────── Playbook
```

## Tutorials

| # | File | Topic | Level |
|---|------|-------|-------|
| 01 | [01-claude-models-and-capabilities.md](01-claude-models-and-capabilities.md) | Claude model families, context windows, multimodal capabilities, web interface power features | Beginner |
| 02 | [02-claude-code-setup-and-basics.md](02-claude-code-setup-and-basics.md) | Installation, authentication, slash commands, CLAUDE.md, reading/editing code | Beginner |
| 03 | [03-claude-code-advanced-workflows.md](03-claude-code-advanced-workflows.md) | Multi-file edits, git workflows, plan mode, headless mode, parallel agents | Intermediate |
| 04 | [04-claude-code-customisation.md](04-claude-code-customisation.md) | CLAUDE.md deep dive, hooks, MCP servers, custom slash commands, settings | Intermediate |
| 05 | [05-claude-code-in-ide.md](05-claude-code-in-ide.md) | VS Code extension, JetBrains/IntelliJ integration, terminal workflows | Intermediate |
| 06 | [06-claude-api-and-spring-ai.md](06-claude-api-and-spring-ai.md) | Messages API, Spring AI ChatClient, tool use, structured outputs, building a chatbot | Advanced |
| 07 | [07-claude-in-ci-cd-and-automation.md](07-claude-in-ci-cd-and-automation.md) | GitHub Actions, automated PR review, test generation, Terraform review | Advanced |
| 08 | [08-daily-workflow-playbook.md](08-daily-workflow-playbook.md) | Morning routines, development workflows, DevOps tasks, cheat sheet | All Levels |

## Prerequisites

- Node.js 18+ (for Claude Code installation)
- An Anthropic API key or Claude Max subscription
- Java 21+ and Spring Boot 3.x
- Familiarity with Git, Docker, and terminal usage

## How to Use This Series

1. **New to Claude?** Start with Tutorial 01 to understand the model landscape, then move to 02 for hands-on setup.
2. **Already using Claude Code?** Jump to 03 and 04 for advanced workflows and customisation.
3. **Want to integrate Claude into your apps?** Tutorial 06 covers the API and Spring AI integration.
4. **Setting up CI/CD?** Tutorial 07 has ready-to-use GitHub Actions workflows.
5. **Want a daily reference?** Tutorial 08 is your go-to playbook.

## Quick Start

```bash
# Install Claude Code
npm install -g @anthropic-ai/claude-code

# Authenticate
claude auth login

# Start using in your project
cd your-spring-boot-project
claude
```

## Related Resources

- [Anthropic Documentation](https://docs.anthropic.com/)
- [Claude Code Documentation](https://docs.anthropic.com/en/docs/claude-code)
- [Spring AI Documentation](https://docs.spring.io/spring-ai/reference/)
- [Model Context Protocol](https://modelcontextprotocol.io/)
