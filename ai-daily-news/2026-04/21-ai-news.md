# AI Daily News — 2026-04-21 (Tuesday)

## Headline: Anthropic Claude Mythos Public Preview Ships, NVIDIA Unveils Rubin Ultra Roadmap, OpenAI + Oracle Sign $15B Stargate Expansion

---

## 1. Frontier Model Updates

**Anthropic Claude Mythos — Public Preview GA**
Anthropic shipped Claude Mythos into public preview on Tuesday, ending six weeks of partner-only access through Project Glasswing. Mythos introduces a new "extended reasoning" mode that decouples thinking budget from output length, allowing up to 2M thinking tokens per turn on complex tasks. Benchmark highlights: 78.4% on SWE-bench Verified (new SOTA), 94.1% on MATH-500, and 91.7% on GPQA Diamond. Pricing matches Opus 4.6 tier. API access rolls out to all tiers over 72 hours.

**OpenAI GPT-5.1 Vision Upgrade**
OpenAI pushed an incremental upgrade to GPT-5.1's vision capabilities, landing 88.2% on MathVista (up from 84.1%) and introducing native PDF understanding without a conversion step — the model now ingests page layout, tables, and embedded charts directly. Document-heavy agentic workflows can drop the PDF-to-text preprocessing layer entirely.

**Mistral Large 3 Turbo Released**
Mistral released Large 3 Turbo, a distilled variant of Large 3 that matches 92% of the parent's MMLU-Pro score at roughly a quarter of the inference cost. Turbo targets the high-volume production tier — function calling, structured output, and RAG — and is available now on Le Plateforme and Azure AI Foundry.

---

## 2. Open-Source Highlights

**Qwen3 72B Released by Alibaba**
Alibaba released Qwen3 72B under Apache 2.0 with native 256K context and a dense transformer architecture. Benchmarks put Qwen3 72B ahead of Llama 4.5-70B on HumanEval+ (88.9%) and MBPP (83.2%), making it the strongest open-weight code model in its size class. GGUF quantisations are already on Hugging Face.

**vLLM 0.7 Adds Speculative Decoding for MoE Models**
The vLLM project shipped 0.7 with first-class speculative decoding support for mixture-of-experts models. Early benchmarks on Mixtral 8x22B show 2.1× throughput improvement with no quality loss. The release also adds prefix caching for multi-turn chat workloads.

---

## 3. Agentic AI & Tooling

**Model Context Protocol (MCP) 1.1 Released**
Anthropic released MCP 1.1 with three significant additions: streaming tool responses (partial results before completion), resource subscriptions (push updates when a resource changes), and first-class authentication via OAuth 2.1. The MCP registry at mcp.anthropic.com now lists 420+ community servers, up from 180 two months ago.

**Claude Code 3.0 Ships Plugin Ecosystem**
Anthropic launched Claude Code 3.0 with a public plugin marketplace. Plugins extend Claude Code with custom skills, slash commands, hooks, and status-line integrations. Top launch-partner plugins include Datadog (live metrics in the terminal), Sentry (error triage), and Linear (ticket sync).

**LangChain MultiAgent 0.2 — Swarm Primitive GA**
LangChain's MultiAgent package 0.2 promoted the `Swarm` primitive out of preview — a hand-off–based multi-agent topology where agents pass control peer-to-peer rather than through a central supervisor. Swarm ships with built-in cycle detection and cost budgeting.

---

## 4. Funding & Business

**OpenAI + Oracle — $15B Stargate Expansion**
OpenAI and Oracle announced a $15B expansion to the Stargate infrastructure project, adding three new data-centre sites in Texas, Arizona, and Wisconsin. The expansion brings total planned Stargate capacity to 9 GW by end of 2028, with the first Arizona site going live in Q3 2026. Oracle's share price rose 4.6% on the news.

**Anthropic Enterprise ARR Crosses $35B Run Rate**
Anthropic disclosed an enterprise ARR run rate of $35B on a strategic-investor call, up from $30B two weeks ago. Growth is driven by Claude Opus 4.6 adoption in financial services and healthcare, and early Mythos partner contracts.

**Cognition Labs Raises $800M Series D at $12B Valuation**
Cognition (maker of Devin) closed an $800M Series D led by Founders Fund at a $12B post-money valuation. Funds will expand Devin's enterprise offering and invest in an autonomous code-migration product targeting Java, .NET, and COBOL modernisation.

---

## 5. Research & Breakthroughs

**NVIDIA Unveils Rubin Ultra Roadmap at GTC Spring 2026**
At a surprise GTC Spring keynote on Tuesday, NVIDIA revealed the Rubin Ultra platform roadmap: 4× HBM4e per GPU (1TB memory), 3.5× FP8 throughput vs Blackwell Ultra, and a new NVLink 6 fabric supporting 576-GPU coherent domains. First Rubin Ultra systems ship to hyperscalers in Q4 2026, with broader availability in 2027.

**DeepMind "Evo-Agent" — Evolutionary Multi-Agent Self-Improvement**
DeepMind published "Evo-Agent", a framework where a population of agents self-improve via evolutionary search over prompts, tool selection, and decomposition strategies. On SWE-bench Verified, Evo-Agent applied to Claude Opus 4.6 achieves 79.1% — slightly ahead of Mythos public numbers — without any model weight changes.

**MIT Paper on Memory-Efficient LoRA for 100B+ Models**
MIT CSAIL published "QLoRA++: 4-bit LoRA Fine-Tuning for 100B+ Parameter Models on Single GPU", demonstrating full-rank fine-tuning of Llama 4.5-70B on a single H200 with novel blockwise quantisation. Reference code released under MIT license.

---

## 6. Regulation & Policy

**EU AI Act — August 2026 Enforcement Milestone**
The European Commission reconfirmed the 2 August 2026 enforcement milestone for general-purpose AI provisions. Providers with models >10^25 FLOPs of compute will face the highest tier of obligations (systemic risk assessments, red-teaming reports, cybersecurity certifications). Fines for non-compliance can reach 7% of global turnover.

**US — White House Issues AI Infrastructure Executive Order**
The White House issued an executive order expediting federal permitting for AI data centres meeting energy-efficiency and grid-stability criteria. The order also creates a new "AI Infrastructure Task Force" within the DOE to coordinate grid-capacity planning with hyperscaler build-outs.

**South Korea Passes AI Basic Act Implementation Rules**
South Korea's Ministry of Science and ICT published implementation rules for the AI Basic Act, effective 22 January 2027. The rules require pre-market risk assessments for high-impact AI systems and create a certification scheme operated by the Telecommunications Technology Association.

---

## 7. Quick Hits

- **Apple:** Apple Intelligence adds Claude Mythos as an optional backend for Siri advanced queries
- **Google:** Gemini Code Assist 3 adds multi-repo context and native GitHub Actions integration
- **Microsoft:** Copilot Studio adds Mythos and GPT-5.1 as selectable models
- **Meta:** Ray-Ban Display AR glasses ship with on-device Llama 4.5-8B
- **Databricks:** Mosaic AI Agent Framework 2.0 adds human-in-the-loop primitives
- **Hugging Face:** Inference Endpoints adds auto-scaling to zero for dev tier
- **Cursor:** Cursor Tab now predicts multi-file edits based on project history
- **Perplexity:** Comet browser launches enterprise tier with SSO and audit logs
- **Figure:** Figure 03 completes first fully autonomous 8-hour shift at BMW
- **xAI:** Grok 3 adds native voice mode with sub-300ms latency

---

*Next: 2026-04-22 — Claude Mythos enterprise GA pricing, GPT-5.1 Turbo benchmarks leak, EU AI Office publishes Q1 enforcement summary*
