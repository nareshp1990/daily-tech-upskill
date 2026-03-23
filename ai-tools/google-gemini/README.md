# Google Gemini AI Ecosystem — Tutorial Series

> **Clarification:** This series covers **Google's Gemini AI ecosystem** — the family of
> multimodal AI models, developer tools, IDE integrations, APIs, SDKs, and cloud-native
> AI services that Google offers under the Gemini brand. If you arrived here searching for
> "Google Antigravity," you are in the right place — Gemini is Google's flagship AI
> platform for developers and enterprises.

---

## Who This Is For

Senior backend / full-stack engineers who already work with:
- Java 17+ / Spring Boot 3.x / Spring Cloud
- MySQL, Kafka, Docker, Kubernetes
- Azure (primary cloud), Terraform, GitHub Actions
- REST & gRPC APIs, microservices architecture

The tutorials assume you are comfortable with LLM fundamentals (tokens, context windows,
embeddings, RAG, tool use). They focus on **how to leverage Gemini in your daily engineering
workflow** — not on introductory AI theory.

---

## Learning Path

| # | File | Topic | Time |
|---|------|-------|------|
| 1 | [01-gemini-models-and-capabilities.md](01-gemini-models-and-capabilities.md) | Gemini model families, context windows, multimodal capabilities, AI Studio, model comparison | 60 min |
| 2 | [02-gemini-in-ide.md](02-gemini-in-ide.md) | Gemini Code Assist in VS Code & JetBrains, code completions, chat, debugging, comparison with Copilot & Claude Code | 60 min |
| 3 | [03-gemini-api-and-sdk.md](03-gemini-api-and-sdk.md) | Google AI SDK, Java/Spring AI integration, function calling, structured output, grounding, context caching, practical Spring Boot project | 90 min |
| 4 | [04-google-ai-studio-and-vertex-ai.md](04-google-ai-studio-and-vertex-ai.md) | AI Studio prompt design, Vertex AI platform, model tuning, evaluation, Agent Builder, RAG with Vertex AI Search | 75 min |
| 5 | [05-gemini-for-cloud-and-devops.md](05-gemini-for-cloud-and-devops.md) | Gemini in GCP Console, GKE, Cloud Logging, BigQuery, Security, Terraform — with Azure comparison | 60 min |
| 6 | [06-gemini-advanced-features.md](06-gemini-advanced-features.md) | Gems, Deep Research, Workspace integration, NotebookLM, Gemini Live, safety settings | 60 min |
| 7 | [07-daily-workflow-playbook.md](07-daily-workflow-playbook.md) | End-to-end daily workflow, quick-reference cheat sheet, Claude vs Copilot vs Gemini comparison matrix | 45 min |

**Total estimated time: ~7.5 hours** (spread across multiple sessions)

---

## Recommended Order

```
Start here
   |
   v
[01] Models & Capabilities ──> understand what Gemini can do
   |
   v
[02] IDE Integration ──────> use Gemini while writing code daily
   |
   v
[03] API & SDK ────────────> build services powered by Gemini
   |
   v
[04] AI Studio & Vertex AI ──> prototype, tune, and deploy models
   |
   v
[05] Cloud & DevOps ───────> integrate Gemini into ops workflows
   |
   v
[06] Advanced Features ────> power-user capabilities
   |
   v
[07] Daily Playbook ───────> tie it all together into a routine
```

---

## Prerequisites

- A Google account (free tier is sufficient for tutorials 1-3, 6-7)
- Google Cloud project with billing enabled (for tutorials 4-5, Vertex AI)
- IntelliJ IDEA or VS Code with Gemini Code Assist plugin installed
- Java 17+, Maven or Gradle, Spring Boot 3.2+
- Familiarity with REST APIs and JSON

---

## Quick Links

| Resource | URL |
|----------|-----|
| Google AI Studio | https://aistudio.google.com |
| Gemini API Docs | https://ai.google.dev/docs |
| Vertex AI Console | https://console.cloud.google.com/vertex-ai |
| Gemini Code Assist | https://cloud.google.com/gemini/docs/codeassist/overview |
| Spring AI Docs | https://docs.spring.io/spring-ai/reference/ |
| Gemini API Pricing | https://ai.google.dev/pricing |
| NotebookLM | https://notebooklm.google.com |
