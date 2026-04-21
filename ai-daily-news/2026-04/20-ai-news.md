# AI Daily News — 2026-04-20 (Monday)

## Headline: OpenAI Ships GPT-5.1 with Dynamic Reasoning Budget, Meta Releases Llama 4.5 Under Permissive License, Anthropic + AWS Expand Bedrock Partnership to $8B

---

## 1. Frontier Model Updates

**OpenAI GPT-5.1 — Dynamic Reasoning Budget**
OpenAI shipped GPT-5.1 to paid tiers on Monday with a new "reasoning budget" control that lets the model dynamically allocate thinking tokens per request based on question complexity. Developers can cap the budget (`reasoning_effort: low|medium|high|auto`) or let the model decide. OpenAI reports a 31% reduction in token cost on mixed workloads compared to GPT-5, while matching or exceeding GPT-5 on MATH-500, GPQA, and SWE-bench Verified.

**Anthropic Claude Opus 4.6 — Code Editor Mode**
Anthropic enabled a new "Code Editor Mode" in Claude Opus 4.6 via the API, providing structured file-edit primitives (apply_patch, multi-file atomic edits, rollback) as first-class tools. Early benchmarks on internal Anthropic agents show a 2.4× reduction in failed-edit retries compared to free-form diff generation.

**Google Gemini 3 Pro Trials Begin**
Google started private trials of Gemini 3 Pro with select enterprise customers on Vertex AI. The model reportedly features a redesigned mixture-of-experts architecture with 128 experts (up from 64 in Gemini 2.5) and a native 4M-token context window. General availability is targeted for late Q2 2026.

---

## 2. Open-Source Highlights

**Meta Releases Llama 4.5 Under Llama Community License 3.0**
Meta dropped Llama 4.5 on Monday in three sizes (8B, 70B, 405B) under an updated Llama Community License that removes the prior 700M MAU clause — now fully permissive for commercial use up to $100M ARR. Llama 4.5-405B tops the open-weight leaderboard on MMLU-Pro (86.1%) and GPQA (71.4%), narrowing the gap with Claude Opus 4.6 and Gemini 2.5 Pro.

**Hugging Face SmolLM3 Released**
Hugging Face published SmolLM3, a 3B-parameter model trained on 11T tokens with a focus on on-device inference. Quantised GGUF builds run at 40 tokens/sec on an M2 MacBook. The model ships with extended 64K context and native tool-calling support.

---

## 3. Agentic AI & Tooling

**LangChain Releases LangGraph Studio 2.0**
LangChain's LangGraph Studio 2.0 shipped with a visual graph editor for agent workflows, integrated LangSmith tracing, and one-click deploy to LangGraph Cloud. New features include time-travel debugging (replay any node from any checkpoint) and a multi-agent visualiser that renders supervisor/swarm topologies live.

**Microsoft Autogen 0.6 Adds Durable Execution**
Microsoft released Autogen 0.6 with durable execution via a Postgres-backed event store — agent runs now survive process restarts and can resume mid-conversation. The release also adds a new `GroupChat` primitive with built-in speaker-selection strategies (round-robin, LLM-selected, deterministic).

**Cursor Agent Mode Graduates to GA**
Cursor's Agent Mode exited preview and is now generally available to all paid tiers. New features include persistent project memory across sessions, a task queue UI for long-running refactors, and integration with GitHub Actions for post-commit review.

---

## 4. Funding & Business

**Anthropic + AWS — Bedrock Partnership Expands to $8B**
Anthropic and AWS expanded their strategic partnership to $8B on Monday, with AWS committing additional Trainium2 capacity and Anthropic making Bedrock the primary deployment channel for Claude enterprise customers. The deal includes a co-engineered inference stack targeting a 40% cost reduction by Q4 2026.

**Perplexity Closes $1.2B Series E at $20B Valuation**
Perplexity AI closed a $1.2B Series E led by IVP and NEA at a $20B post-money valuation. Proceeds will fund the rollout of Perplexity Comet (agentic browser) to enterprise customers and expansion into Japan, Germany, and Brazil.

**Scale AI Layoffs — 14% of Workforce Cut**
Scale AI cut approximately 14% of its workforce (roughly 900 employees), citing a shift from data-labelling services to its automated evaluation platform. CEO Alexandr Wang said the company is doubling down on frontier-model evaluation and red-teaming contracts with the US DoD.

---

## 5. Research & Breakthroughs

**DeepMind Gemini Robotics 1.5 Released**
DeepMind published Gemini Robotics 1.5, a vision-language-action model trained on 2.4B multi-embodiment trajectories. The model achieves 87% task success on real-world manipulation benchmarks across 14 different robot platforms, including humanoids and dexterous grippers.

**Stanford "Agent-R" — Self-Reflective Agents**
Stanford researchers published "Agent-R: Training Language Model Agents to Reflect via Iterative Self-Training", showing a 34% improvement on WebArena and ALFWorld by training agents to generate and critique their own reasoning traces. Code and weights are open under Apache 2.0.

---

## 6. Regulation & Policy

**EU AI Act — First General-Purpose AI Compliance Reports Published**
The EU AI Office published the first batch of compliance assessments for general-purpose AI providers (OpenAI, Anthropic, Google, Meta, Mistral). All five passed the initial review, but the report flagged gaps in training-data transparency and copyright opt-out mechanisms. Full enforcement begins 2 August 2026.

**UK AI Safety Institute Renamed "AI Security Institute"**
The UK government rebranded the AI Safety Institute as the AI Security Institute (AISI), reflecting a narrower mandate focused on national-security risks (CBRN misuse, cyber offence, autonomous replication). The institute's budget was increased to £100M per year through 2029.

**India Releases Draft AI Governance Framework**
India's Ministry of Electronics & IT published a draft AI governance framework with a risk-tiered approach modelled on the EU AI Act but with lighter-touch enforcement. Public consultation runs through 15 June 2026.

---

## 7. Quick Hits

- **Nvidia:** Blackwell Ultra GPUs ship to hyperscalers; export licenses approved for UAE and Saudi Arabia
- **Apple:** Apple Intelligence adds on-device Llama 4.5-8B as fallback for privacy-mode queries
- **Databricks:** DBRX 2 open-sourced — 132B parameters, Apache 2.0
- **Mistral:** Le Chat Enterprise adds SOC 2 Type II certification
- **Cohere:** Command R+ 2026.04 released — 128K context, multilingual RAG gains
- **xAI:** Grok 3 free tier rolls out to all X premium subscribers
- **Runway:** Gen-4 Turbo cuts video generation latency by 60%
- **Figure:** Figure 03 humanoid begins BMW plant trials in South Carolina
- **Waymo:** Robotaxi service expands to Chicago and Washington DC
- **Amazon:** Q Developer adds agentic code migration mode for Java 8 → 21 upgrades

---

*Next: 2026-04-21 — Anthropic Claude Mythos GA timing, Open-weight reasoning models ship, EU AI Act Q2 enforcement milestones*
