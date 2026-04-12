# Graph Report - .  (2026-04-12)

## Corpus Check
- 112 files · ~190,420 words
- Verdict: corpus is large enough that graph structure adds value.

## Summary
- 128 nodes · 132 edges · 21 communities detected
- Extraction: 32% EXTRACTED · 68% INFERRED · 0% AMBIGUOUS · INFERRED: 90 edges (avg confidence: 0.82)
- Token cost: 0 input · 0 output

## Community Hubs (Navigation)
- [[_COMMUNITY_Claude Code Ecosystem|Claude Code Ecosystem]]
- [[_COMMUNITY_LLM Engineering Fundamentals|LLM Engineering Fundamentals]]
- [[_COMMUNITY_AI Tools & Resources Directory|AI Tools & Resources Directory]]
- [[_COMMUNITY_Backend Engineer Prompt Stack|Backend Engineer Prompt Stack]]
- [[_COMMUNITY_System Design Fundamentals|System Design Fundamentals]]
- [[_COMMUNITY_Agentic AI Projects|Agentic AI Projects]]
- [[_COMMUNITY_Copilot & Gemini IDE Tools|Copilot & Gemini IDE Tools]]
- [[_COMMUNITY_Distributed Systems Case Studies|Distributed Systems Case Studies]]
- [[_COMMUNITY_Multi-Agent Orchestration|Multi-Agent Orchestration]]
- [[_COMMUNITY_Frontier Model Releases|Frontier Model Releases]]
- [[_COMMUNITY_Physical AI & World Models|Physical AI & World Models]]
- [[_COMMUNITY_Anthropic Claude Roadmap|Anthropic Claude Roadmap]]
- [[_COMMUNITY_MCP & Agent Governance|MCP & Agent Governance]]
- [[_COMMUNITY_DevOps & Container Prompts|DevOps & Container Prompts]]
- [[_COMMUNITY_AI Daily News Apr 11|AI Daily News Apr 11]]
- [[_COMMUNITY_OpenAI Valuation Milestone|OpenAI Valuation Milestone]]
- [[_COMMUNITY_Anthropic Revenue Milestone|Anthropic Revenue Milestone]]
- [[_COMMUNITY_Gemma 4 Open Source|Gemma 4 Open Source]]
- [[_COMMUNITY_Claude Daily Workflow|Claude Daily Workflow]]
- [[_COMMUNITY_Gemini Daily Workflow|Gemini Daily Workflow]]
- [[_COMMUNITY_Document Processing Agent|Document Processing Agent]]

## God Nodes (most connected - your core abstractions)
1. `Prompt Library Overview` - 11 edges
2. `Anthropic Claude AI Tools Overview` - 9 edges
3. `Daily Tech Upskill Project` - 8 edges
4. `Google Gemini AI Tools Overview` - 6 edges
5. `GitHub Copilot Overview` - 5 edges
6. `LangChain4j Java Agents` - 4 edges
7. `Model Context Protocol (MCP) — Universal Tool Integration` - 4 edges
8. `Multi-Agent Orchestration (Supervisor, Swarm, Hierarchical)` - 4 edges
9. `CDN, Edge Caching & Global Distribution` - 4 edges
10. `LLM Providers (OpenAI, Anthropic, Google, Meta)` - 3 edges

## Surprising Connections (you probably didn't know these)
- `Daily Tech Upskill Project` --references--> `Prompt Library Overview`  [EXTRACTED]
  README.md → prompt-library/README.md
- `Senior Backend Engineer Learning Profile` --conceptually_related_to--> `Java & Spring Boot Prompts`  [INFERRED]
  ai-prompt.md → prompt-library/01-java-spring-boot.md
- `Senior Backend Engineer Learning Profile` --conceptually_related_to--> `Confluent Kafka Prompts`  [INFERRED]
  ai-prompt.md → prompt-library/04-confluent-kafka.md
- `Senior Backend Engineer Learning Profile` --conceptually_related_to--> `Azure Cloud Prompts`  [INFERRED]
  ai-prompt.md → prompt-library/06-azure-cloud.md
- `Daily Tech Upskill Project` --references--> `LLM Providers (OpenAI, Anthropic, Google, Meta)`  [EXTRACTED]
  README.md → ai-tools-directory.md

## Hyperedges (group relationships)
- **Frontier Model Competition (Anthropic, OpenAI, Google)** — news_08_claude_mythos5, news_10_openai_852b, news_12_gemini_ultra_2m, news_06_claude46_family [INFERRED 0.85]
- **Backend Engineer Prompt Stack** — prompt_lib_java_spring, prompt_lib_kafka, prompt_lib_mysql, prompt_lib_azure, prompt_lib_terraform [INFERRED 0.90]
- **Claude Code Complete Workflow Suite** — claude_setup_basics, claude_advanced_workflows, claude_cowork_multi_agent, claude_customisation, claude_daily_workflow [INFERRED 0.90]
- **AI Coding Assistants (Claude, Copilot, JetBrains)** — claude_code_in_ide, copilot_chat_edits, jetbrains_ai_mastery [INFERRED 0.85]
- **Agentic AI Framework Trio (Spring AI, LangChain4j, LangGraph)** — agentic_spring_ai_production, agentic_langchain4j, agentic_langgraph_python [INFERRED 0.88]
- **AI IDE Assistants (Copilot, Gemini, JetBrains)** — copilot_code_completion, gemini_in_ide, copilot_cli [INFERRED 0.85]
- **Agentic AI Production Projects** — agentic_code_review_project, agentic_customer_support_project, agentic_data_analysis_project [INFERRED 0.90]
- **Week 3 Production Agent Projects** — agentic_ecommerce_project, agentic_devops_project, agentic_doc_processing_project, agentic_research_assistant [INFERRED 0.90]
- **System Design Case Studies (Ride-Share, Payment, Social Feed, URL Shortener)** — sdplan_ride_sharing, sdplan_payment_system, sdplan_social_media_feed, sdplan_url_shortener [INFERRED 0.90]
- **Multi-Agent Orchestration Patterns** — agentic_multi_agent_orchestration, agentic_crewai, agentic_langgraph_multi_agent [INFERRED 0.88]
- **LLM Integration Pipeline (Embeddings → RAG → Agents → Memory)** — plan_embeddings_vector_search, plan_rag, plan_llm_agents, plan_llm_memory, plan_tool_calling [INFERRED 0.92]
- **System Design Infrastructure Core (Caching, DB, Queues, Rate Limiting)** — plan_caching, plan_db_design_indexing, plan_message_queues, plan_rate_limiting [INFERRED 0.90]
- **Distributed Systems Core (CAP, Hashing, Discovery, Service Mesh)** — plan_distributed_systems, plan_consistent_hashing, plan_service_discovery [INFERRED 0.88]

## Communities

### Community 0 - "Claude Code Ecosystem"
Cohesion: 0.14
Nodes (19): AI Tools Overview, Claude Code Advanced Workflows (multi-file, plan mode, subagents), Claude API & Spring AI Integration, Claude in CI/CD & Automation, Claude Code in IDE Integration, Claude Code Cowork & Multi-Agent Collaboration, Claude Code Customisation (CLAUDE.md, hooks, settings), Anthropic Latest Features & Releases (+11 more)

### Community 1 - "LLM Engineering Fundamentals"
Cohesion: 0.15
Nodes (14): Embeddings & Vector Search (pgvector, HNSW, Semantic Search), Generative AI Fundamentals (LLMs, Tokens, Inference, Context Window), LLM Agents & Multi-Step Reasoning (ReAct Loop, Spring AI Agent), LLM APIs & SDKs (Spring AI, Streaming, Retries), LLM Memory & Conversation State, LLM Observability & Evaluation (OTel Tracing, LLM-as-Judge), LLM Security & Prompt Injection Defense (OWASP LLM Top 10), Multi-Agent Systems & Orchestration (Orchestrator-Worker, Fan-Out) (+6 more)

### Community 2 - "AI Tools & Resources Directory"
Cohesion: 0.15
Nodes (13): Official AI Lab Blogs (Anthropic, OpenAI, Google DeepMind), AI Blogs, Newsletters & Resources, AI Newsletters (The Batch, TLDR AI, Ben's Bites), AI Code Editors & Assistants, AI Inference & API Platforms, LLM Providers (OpenAI, Anthropic, Google, Meta), Local LLM Tools (Ollama, LM Studio, llama.cpp), Daily Tech Upskill Project (+5 more)

### Community 3 - "Backend Engineer Prompt Stack"
Cohesion: 0.23
Nodes (12): Senior Backend Engineer Learning Profile, Azure Cloud Prompts, CI/CD & GitHub Actions Prompts, Communication & Productivity Prompts, Git & Code Review Prompts, Java & Spring Boot Prompts, Confluent Kafka Prompts, MySQL & Data Prompts (+4 more)

### Community 4 - "System Design Fundamentals"
Cohesion: 0.2
Nodes (12): API Design (REST, Versioning, Rate Limiting, Idempotency), Caching Strategies (Redis, Cache-Aside, Write-Through, Stampede), Consistent Hashing & Data Partitioning (Ring, VNodes), Database Design & Indexing (Schema, EXPLAIN, Composite Indexes), Distributed Systems Fundamentals (CAP, Raft, Sagas), Fine-Tuning & Model Customisation (LoRA, JSONL, Dataset Prep), Message Queues & Event-Driven Architecture (Kafka, CQRS, Event Sourcing), Phase 1 Capstone & Review (Full AI System Architecture) (+4 more)

### Community 5 - "Agentic AI Projects"
Cohesion: 0.24
Nodes (11): Agentic AI Capstone — Full Production Agent, Project: Code Review Agent, Custom Tool Development for Agents, Project: Customer Support System, Project: Data Analysis Agent, LangChain4j Java Agents, Python Agents with LangGraph, Model Context Protocol (MCP) — Universal Tool Integration (+3 more)

### Community 6 - "Copilot & Gemini IDE Tools"
Cohesion: 0.24
Nodes (10): GitHub Copilot CLI, GitHub Copilot Code Completion Mastery, GitHub Copilot Setup & Configuration, Gemini Advanced Features, Gemini API & SDK, Gemini for Cloud & DevOps, Google AI Studio & Vertex AI, Gemini in IDE (+2 more)

### Community 7 - "Distributed Systems Case Studies"
Cohesion: 0.27
Nodes (10): CDN, Edge Caching & Global Distribution, Circuit Breakers & Resilience Patterns (Resilience4j), Database Replication & Sharding (Primary-Replica, ShardingSphere), System Design: Payment System (Idempotency, Double-Entry, Reconciliation), Phase 2 Capstone & Interview Prep (Framework, Trade-offs), System Design: Search & Typeahead (Inverted Index, Trie, Elasticsearch), System Design: Social Media Feed (Fan-out Write vs Read, Redis), System Design: URL Shortener (Base62, Redis, CDN) (+2 more)

### Community 8 - "Multi-Agent Orchestration"
Cohesion: 0.25
Nodes (9): Agentic AI Architecture Deep Dive (ReAct, Plan-Execute, Patterns), CrewAI Role-Based Agent Teams, Project: DevOps Agent, Project: E-Commerce Agent, Human-in-the-Loop & Guardrails, LangGraph Multi-Agent Workflows, Multi-Agent Orchestration (Supervisor, Swarm, Hierarchical), Production Agent Architecture (scaling, cost, reliability) (+1 more)

### Community 9 - "Frontier Model Releases"
Cohesion: 0.67
Nodes (3): DeepSeek V4 Trains for $5.2M, Google Gemma 4 Open-Source Release, Gemini 3.1 Ultra — 2M Token Context Window

### Community 10 - "Physical AI & World Models"
Cohesion: 1.0
Nodes (2): NVIDIA Isaac GR00T — Language-Driven Robotics, World Models as 2026 AI Frontier

### Community 11 - "Anthropic Claude Roadmap"
Cohesion: 1.0
Nodes (2): Claude 4.6 Family — Opus and Sonnet, Claude Mythos 5 — 10T Parameter Model

### Community 12 - "MCP & Agent Governance"
Cohesion: 1.0
Nodes (2): Microsoft Agent Governance Toolkit, MCP at 97M Installs — Protocol Standard

### Community 13 - "DevOps & Container Prompts"
Cohesion: 1.0
Nodes (2): DevOps & Observability Prompts, Docker & Kubernetes Prompts

### Community 14 - "AI Daily News Apr 11"
Cohesion: 1.0
Nodes (1): AI Daily News 2026-04-11

### Community 15 - "OpenAI Valuation Milestone"
Cohesion: 1.0
Nodes (1): OpenAI $122B Round at $852B Valuation

### Community 16 - "Anthropic Revenue Milestone"
Cohesion: 1.0
Nodes (1): Anthropic Revenue Hits $30B Run Rate

### Community 17 - "Gemma 4 Open Source"
Cohesion: 1.0
Nodes (1): Gemma 4 Apache 2.0 — Four Model Sizes

### Community 18 - "Claude Daily Workflow"
Cohesion: 1.0
Nodes (1): Claude Code Daily Workflow Playbook

### Community 19 - "Gemini Daily Workflow"
Cohesion: 1.0
Nodes (1): Gemini Daily Workflow Playbook

### Community 20 - "Document Processing Agent"
Cohesion: 1.0
Nodes (1): Project: Document Processing Agent

## Knowledge Gaps
- **50 isolated node(s):** `Phase 1: Generative AI & Agentic AI`, `Phase 2: System Design`, `Phase 3: Microservices & Cloud-Native`, `Phase 4: DevOps & Platform Engineering`, `Phase 5: Emerging Backend Tech` (+45 more)
  These have ≤1 connection - possible missing edges or undocumented components.
- **Thin community `Physical AI & World Models`** (2 nodes): `NVIDIA Isaac GR00T — Language-Driven Robotics`, `World Models as 2026 AI Frontier`
  Too small to be a meaningful cluster - may be noise or needs more connections extracted.
- **Thin community `Anthropic Claude Roadmap`** (2 nodes): `Claude 4.6 Family — Opus and Sonnet`, `Claude Mythos 5 — 10T Parameter Model`
  Too small to be a meaningful cluster - may be noise or needs more connections extracted.
- **Thin community `MCP & Agent Governance`** (2 nodes): `Microsoft Agent Governance Toolkit`, `MCP at 97M Installs — Protocol Standard`
  Too small to be a meaningful cluster - may be noise or needs more connections extracted.
- **Thin community `DevOps & Container Prompts`** (2 nodes): `DevOps & Observability Prompts`, `Docker & Kubernetes Prompts`
  Too small to be a meaningful cluster - may be noise or needs more connections extracted.
- **Thin community `AI Daily News Apr 11`** (1 nodes): `AI Daily News 2026-04-11`
  Too small to be a meaningful cluster - may be noise or needs more connections extracted.
- **Thin community `OpenAI Valuation Milestone`** (1 nodes): `OpenAI $122B Round at $852B Valuation`
  Too small to be a meaningful cluster - may be noise or needs more connections extracted.
- **Thin community `Anthropic Revenue Milestone`** (1 nodes): `Anthropic Revenue Hits $30B Run Rate`
  Too small to be a meaningful cluster - may be noise or needs more connections extracted.
- **Thin community `Gemma 4 Open Source`** (1 nodes): `Gemma 4 Apache 2.0 — Four Model Sizes`
  Too small to be a meaningful cluster - may be noise or needs more connections extracted.
- **Thin community `Claude Daily Workflow`** (1 nodes): `Claude Code Daily Workflow Playbook`
  Too small to be a meaningful cluster - may be noise or needs more connections extracted.
- **Thin community `Gemini Daily Workflow`** (1 nodes): `Gemini Daily Workflow Playbook`
  Too small to be a meaningful cluster - may be noise or needs more connections extracted.
- **Thin community `Document Processing Agent`** (1 nodes): `Project: Document Processing Agent`
  Too small to be a meaningful cluster - may be noise or needs more connections extracted.

## Suggested Questions
_Questions this graph is uniquely positioned to answer:_

- **Why does `LLM Security & Prompt Injection Defense (OWASP LLM Top 10)` connect `LLM Engineering Fundamentals` to `System Design Fundamentals`?**
  _High betweenness centrality (0.034) - this node is a cross-community bridge._
- **What connects `Phase 1: Generative AI & Agentic AI`, `Phase 2: System Design`, `Phase 3: Microservices & Cloud-Native` to the rest of the system?**
  _50 weakly-connected nodes found - possible documentation gaps or missing edges._
- **Should `Claude Code Ecosystem` be split into smaller, more focused modules?**
  _Cohesion score 0.14 - nodes in this community are weakly interconnected._