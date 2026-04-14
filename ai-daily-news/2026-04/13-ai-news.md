# AI Daily News — 2026-04-13 (Monday)

---

## Headline: OpenAI Launches GPT-5o with Native Code Execution, Apple Acquires Reka AI for $1.8B, EU Passes AI Liability Directive

---

## 1. Frontier Model Updates

### OpenAI GPT-5o — Native Code Interpreter at Inference Time
- **GPT-5o** (Omni) ships with a **native code execution sandbox** baked into the model inference pipeline.
- No tool-call round-trips — the model reasons, writes code, runs it, observes output, and incorporates results in a single forward pass.
- Benchmarks: **89.5% on SWE-Bench** (from 72% on GPT-4.5), **97.3% on MATH** — representing a step-change in agentic coding capability.
- API: available via `/v1/responses` endpoint (same as prior GPT-4.5o), new `execution_sandbox` parameter.

### Anthropic Releases Claude Sonnet 4.7 — Speed + Reasoning
- **Claude Sonnet 4.7** drops latency by 40% over Sonnet 4.6 with no benchmark regression.
- New **adaptive thinking budget**: Claude auto-scales reasoning tokens based on query complexity — short factual queries use 50 tokens; complex multi-step planning uses 4,000.
- Extended context to **500K tokens** with improved mid-context recall (critical for large codebases and legal documents).

### Mistral Codestral 2.0 — Polyglot Code Model
- Mistral ships **Codestral 2.0**: 35B parameters, trained on 5T code tokens across 120 programming languages.
- Best-in-class for: Java, Python, TypeScript, Rust, Go, SQL — competitive with GPT-5o on fill-in-the-middle tasks.
- **Apache 2.0 license** — unrestricted commercial use, local deployment on 2× A100s.

---

## 2. Open-Source Highlights

### Meta Llama 4 Scout — 17B MoE, 10M Token Context
- **Llama 4 Scout** (17B active, 109B total via MoE) hits **10 million token context** — the longest of any open model.
- Fits in 40GB VRAM (single H100) with 4-bit quantization — practical for enterprise on-premise deployment.
- Use cases: entire GitHub repos, legal document sets, months of conversation history — all in a single context.

### Microsoft Phi-4-Mini — 3.8B Reasoning Specialist
- **Phi-4-Mini** (3.8B) achieves Phi-3-Medium-level performance at 1/3 the size.
- Designed for **edge deployment**: runs on iPhone 15 Pro and Snapdragon X Elite at 25 tokens/second.
- Reasoning, structured output, and function calling — production-ready for on-device agentic workloads.

---

## 3. Agentic AI & Tooling

### Microsoft Copilot Studio — Visual Agent Workflow Designer
- **Copilot Studio** ships a no-code **visual workflow designer** for multi-agent pipelines.
- Drag-and-drop: connect Azure AI agents, assign tools, define handoff conditions, set approval gates.
- Targeting enterprise teams without ML expertise — already integrated with Azure DevOps, Teams, and SAP connectors.

### LangChain LangGraph Cloud — Managed Agent Runtime
- **LangGraph Cloud** launches: managed infrastructure for LangGraph-based agents — auto-scaling, persistence, checkpointing.
- Agents can be paused, resumed, and replayed from any checkpoint — enabling **long-running autonomous workflows** (hours/days).
- Spring AI Java SDK now has first-class LangGraph-compatible state graph builder.

### Warp AI Terminal — Inline Agent Debugging
- **Warp 2.0** integrates a resident AI agent that explains failing commands, proposes fixes, and can execute approved fixes autonomously.
- Works locally — no API call for known patterns; falls back to Claude Sonnet 4.7 for novel errors.
- Used by 2M developers; now the default terminal for GitHub Codespaces.

---

## 4. Funding & Business

### Apple Acquires Reka AI for $1.8B
- Apple closes its **Reka AI acquisition** at $1.8 billion — Reka's team (150 researchers from DeepMind, Google Brain, OpenAI) joins Apple Intelligence.
- Focus: **on-device multimodal agents** — combining vision, audio, and language natively in Apple Silicon inference.
- Expected impact: Apple Intelligence in iOS 21 (2027) will feature agentic capabilities running entirely on-device.

### Cohere Closes $500M Series E at $5.2B Valuation
- **Cohere** raises $500M led by Salesforce Ventures and NVIDIA — targeting enterprise private-cloud LLM deployment.
- Strong traction in regulated industries (banking, healthcare, legal) where data sovereignty prevents public cloud API use.
- New: **Cohere Private Cloud** — fully air-gapped deployment for Fortune 500 customers.

### Databricks AI Revenue Hits $3B ARR
- **Databricks** reports $3B ARR for AI/ML products — DBRX and Mosaic AI platform growing 220% YoY.
- Positioning: "the data+AI platform" — combine your data lakehouse with model fine-tuning and serving in one stack.

---

## 5. Research & Breakthroughs

### Scaling Laws Break Down Above 2T Parameters
- New research from Epoch AI: **scaling laws flatten above ~2T parameters** for autoregressive language models.
- Implication: raw parameter scaling is no longer the primary lever — **data quality, RLHF, reasoning training, and architecture improvements** matter more.
- Industry shift: labs moving from parameter-count races to "intelligence density" — doing more with smaller, better-trained models.

### Self-Play Reasoning — Models Teaching Themselves
- Anthropic publishes a paper on **self-play reasoning**: Claude generates challenging problems, solves them, critiques its own solutions, and fine-tunes on the best chains.
- No human-labeled data required past initialization — a path to continuous self-improvement with alignment guardrails.
- Early results: 15% improvement on mathematical reasoning benchmarks with 5 rounds of self-play.

### Multi-Agent Coordination — Theory of Mind Benchmarks
- Stanford releases **TAOM (Theory of Mind for Agents)** benchmark: evaluates whether agents can model other agents' beliefs and intentions.
- Current frontier models score 71–78% — humans at 94%.
- Key gap: agents struggle when other agents have **incorrect beliefs** — critical for deceptive adversary scenarios.

---

## 6. Regulation & Policy

### EU AI Liability Directive — Passed
- The **EU AI Liability Directive** officially passes, taking effect from 2027 for high-risk AI systems.
- Key provision: **reversed burden of proof** — if an AI system causes harm, the developer must prove the system was not at fault (rather than the victim proving it was).
- Covers: medical AI, autonomous vehicles, credit scoring, content moderation at scale.
- Impact on startups: liability insurance and audit trails become mandatory for EU market access.

### US Senate AI Accountability Act — Committee Vote
- **US Senate AI Accountability Act** advances through committee — proposes mandatory **pre-deployment safety evaluations** for frontier models above a compute threshold.
- Bipartisan support; Senate floor vote expected Q3 2026.
- Industry response: OpenAI, Anthropic, Google support the framework with minor proposed amendments; Meta and open-source coalition push for open-model exemptions.

---

## 7. Quick Hits

- **Cloudflare AI Gateway** now supports multi-provider routing: auto-failover from OpenAI → Anthropic → Cohere based on latency SLA
- **GitHub Models** adds Java (Spring AI) and Rust SDK clients for model experimentation in Codespaces
- **Perplexity Pro** integrates Claude Sonnet 4.7 as an alternative to GPT-5o for technical queries
- **Vercel AI SDK 4.0** adds streaming support for function calls and multi-turn structured output

---

*Next: 2026-04-14 — Llama 4 Maverick Tops Open-Source Leaderboard, NVIDIA Project Digits Ships, K8s AI Operator Goes GA*
