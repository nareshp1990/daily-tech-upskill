# AI Daily News — 2026-04-19 (Sunday)

## Headline: Anthropic Closes $30B Series G at $380B Valuation, xAI Grok 3 Gets Persistent Memory, DeepSeek R2 Hits 92.7% on AIME

---

## 1. Frontier Model Updates

**Anthropic Claude Opus 4.6 Leads LMSYS Arena**
Claude Opus 4.6 retains the top position on the LMSYS Chatbot Arena leaderboard and has set a new record on SWE-bench Verified with a 65.3% solve rate, ahead of all competing models. The hybrid MoE architecture enables sustained throughput at lower inference cost compared to dense equivalents.

**xAI Grok 3 — Real-Time Image Generation & Persistent Memory**
xAI shipped two major Grok 3 capabilities this weekend: native real-time image generation powered by a proprietary diffusion model integrated directly into the chat interface, and Grok Memory — persistent cross-conversation context that allows the model to recall user preferences, ongoing projects, and prior decisions across sessions. Both features roll out to Premium subscribers first.

**Google Gemini 2.5 Pro Stable Released**
Google moved Gemini 2.5 Pro from experimental to stable on Google AI Studio and Vertex AI. The model features improved code generation quality and a confirmed 1M-token context window with sub-5-second time-to-first-token on standard queries.

---

## 2. Open-Source Highlights

**DeepSeek R2 Benchmarks Published**
DeepSeek's full evaluation report for R2 is now public. Highlights: 92.7% on AIME 2025, 89.4% on MATH-500, and 85.1% on HumanEval+. Pricing sits at roughly 70% below comparable Western reasoning models, continuing the trend of commoditising frontier-level reasoning. The model weights remain proprietary but API access is widely available.

**Mistral's Codestral 2.1 Released Under Apache 2.0**
Mistral open-sourced Codestral 2.1 this week under Apache 2.0, offering a 32K context code completion model fine-tuned on 80+ programming languages. Early benchmarks put it ahead of CodeLlama 70B on HumanEval at a fraction of the inference cost — a strong option for self-hosted IDE integrations.

---

## 3. Agentic AI & Tooling

**OpenAI Responses API Exits Beta**
OpenAI's Responses API — which provides persistent thread state, file search, code interpreter, and web search as built-in tools — has exited beta and is now generally available. It replaces the Assistants API for most agentic workflows and simplifies multi-turn agent development with automatic context management.

**LangGraph 0.4 Cloud GA**
LangGraph Cloud 0.4 reached general availability with native support for human-in-the-loop interrupts, durable execution (checkpointing to Postgres), and a visual debugger for stepping through agent graphs. The LangSmith integration now auto-traces every node execution without additional instrumentation.

---

## 4. Funding & Business

**Anthropic Raises $30B Series G at $380B Valuation**
Anthropic closed a $30 billion Series G led by GIC and Coatue, valuing the company at $380 billion post-money — up from $61.5 billion in its previous round. The capital will fund expanded compute capacity, safety research, and international expansion with a focus on European and Asia-Pacific enterprise markets.

**xAI Secures $20B Series E**
Elon Musk's xAI raised $20 billion in a Series E backed by Nvidia and Cisco among others. The round values the company at approximately $120 billion and will fund the Colossus 2 supercluster expansion (target: 2M GPUs by end of 2026) and Grok product development.

**Q1 2026 Foundational AI VC: Double All of 2025**
Crunchbase data shows Q1 2026 venture funding to foundational AI startups exceeded the entire 2025 total, driven by the three mega-rounds — OpenAI ($122B cumulative), Anthropic ($30B), and xAI ($20B). The capital concentration at the top five labs has never been higher.

---

## 5. Research & Breakthroughs

**Meta FAIR: Efficient Long-Context via Sparse Attention**
Meta's Fundamental AI Research team published "Sparse Flash Attention 3" achieving 4× throughput improvement for 128K+ context windows with less than 0.5% quality degradation on standard benchmarks. The technique uses learned sparsity masks that preserve the most relevant attention patterns while skipping low-signal token pairs.

**MIT: LLM Agents Solve Multi-Step Planning 40% Better with Structured Scratchpads**
A new MIT study found that giving LLM agents a constrained, schema-validated scratchpad (rather than free-form chain-of-thought) improves multi-step planning accuracy by 40% on the GAIA benchmark. The finding has direct implications for agentic framework design — structured intermediate outputs outperform verbose reasoning traces.

---

## 6. Regulation & Policy

**EU AI Act Article 6 Guidance Finalised**
The European AI Office published its final guidance on Article 6 high-risk AI system classification. Key clarifications: AI systems embedded in HR software for recruitment screening are now definitively classified as high-risk; B2B SaaS providers must maintain technical documentation for 10 years post-deployment. Enforcement begins August 2026.

**UK AI Safety Institute Rebranded to AI Security Institute**
The UK government renamed and expanded the AISI's mandate to explicitly cover adversarial AI threats including deepfakes, AI-enabled cyberattacks, and model theft. The institute will publish mandatory security baselines for frontier model providers operating in the UK by Q3 2026.

---

## 7. Quick Hits

- **Hugging Face** launched `transformers.js` v4 with native WebGPU support — run 7B models in-browser at near-native speed on consumer hardware
- **Nvidia** released CUDA 13.1 with improved profiling hooks for distributed training across heterogeneous GPU generations
- **GitHub Copilot** added multi-file edit mode to VS Code — accept/reject changes across a PR diff in a single review session
- **Perplexity** hit 100M monthly active users, up from 10M a year ago, driven by the AI search expansion into enterprise plans
- **AWS Bedrock** added Claude Opus 4.6 and DeepSeek R2 as managed models, available in us-east-1 and eu-west-1 from today
- **Weights & Biases** open-sourced their Weave evaluation framework under Apache 2.0 — structured LLM output eval with automatic trace capture

---

*Next: 2026-04-20 — Anthropic's safety research roadmap for 2026H2, OpenAI o4 release signals, and the EU's first AI Act enforcement action*
