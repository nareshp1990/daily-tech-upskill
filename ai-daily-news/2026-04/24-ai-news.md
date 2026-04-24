# AI Daily News — 2026-04-24 (Friday)

## Headline: Anthropic Publishes MCP 1.2 RFC with Agent-to-Agent Protocol, Gemini 3.2 Flash Thinking Ships, Fortune 500 Mythos Adoption Crosses 60%

---

## 1. Frontier Model Updates

**Google — Gemini 3.2 Flash Thinking GA**
Google released Gemini 3.2 Flash Thinking, a cost-optimised reasoning variant that matches 94% of Ultra's performance on reasoning benchmarks at one-fifth the inference cost. Flash Thinking is priced at $1.50/M input and $6/M output, placing it aggressively against Claude Mythos Turbo and GPT-5.1 Turbo. Native agent-graph parallel tool use is included.

**Anthropic — Mythos Thinking Mode Extensions**
Anthropic released two extensions to Mythos thinking mode: interruptible thinking (the model can pause, accept new tool results, and resume the same chain) and "shared thinking" across parallel tool calls, letting a single thinking trace inform a fan-out of tool invocations. Both roll out to API and Claude.ai over the next week.

**Meta — Llama 5 Teaser at LlamaCon**
Meta AI teased Llama 5 at LlamaCon, showing architecture slides for a dense 450B + MoE 900B model with native multimodal input (text, image, video, audio) and native agent graph output. Ship date not announced; Mark Zuckerberg hinted at "summer release window." Open weights expected under a revised community licence.

---

## 2. Open-Source Highlights

**Qwen3.5 Released by Alibaba**
Alibaba released Qwen3.5 in 7B, 32B, and 110B variants under Apache 2.0. Qwen3.5-110B scores 84.7% on MATH-500 and 67.1% on SWE-bench Verified, making it the strongest truly-open-weight code model. Native 1M-token context via a new RoPE scaling scheme; ships with a vision-language variant.

**Ollama 0.9 Ships Native MCP Client + Agent Runtime**
Ollama 0.9 introduces a native MCP client so locally running open-weight models can consume tools from any MCP server. The release also adds a lightweight agent runtime with tool routing, retry policy, and parallel tool execution — everything runs on-device.

---

## 3. Agentic AI & Tooling

**Anthropic — MCP 1.2 RFC Opens for Comment**
Anthropic published the MCP 1.2 RFC, whose headline feature is a new "agent-to-agent" (A2A) protocol layer that lets MCP-exposed agents discover, negotiate with, and delegate to other agents across organisational boundaries. The RFC also formalises audit-log exports, mandatory-field tool schemas, and a capability-based permission model. Comment period runs 30 days.

**Enterprise Mythos Adoption Crosses 60% in Fortune 500**
IDC and Gartner released a joint flash note: Mythos has now been deployed in production workloads at 61% of Fortune 500 firms, up from 38% two months ago, driven largely by enterprise-tier agentic coding, contact-centre automation, and document-processing use cases. Customer overlap between Mythos and GPT-5.1 is estimated at 43%.

**Browser Use 2.0 Released**
The Browser Use project released 2.0 with a 3× speedup on standard web-navigation benchmarks and new support for "high-risk" page-interaction detection (checkout forms, MFA prompts) that trigger approval gates in the host agent.

---

## 4. Funding & Business

**Databricks Acquires Snorkel AI for $2.6B**
Databricks announced the acquisition of Snorkel AI for $2.6B, bringing programmatic data-labelling and weak-supervision tooling into the Mosaic AI platform. The deal strengthens Databricks's fine-tuning and RAG-evaluation story against Snowflake and Hugging Face.

**ElevenLabs Raises $500M at $9B Valuation**
ElevenLabs closed a $500M Series D at a $9B valuation led by a16z. The round funds an enterprise voice-agent platform and a partnership with Duolingo for conversational language tutors. ElevenLabs now claims a 71% share of the synthetic-voice API market.

**Palantir + Anthropic Extend Federal Partnership**
Palantir and Anthropic extended their partnership with a $2.1B, five-year deal for deploying Mythos-powered agents across U.S. federal civilian agencies. The deal follows successful pilots at the VA and SSA that cut claims-processing backlogs by 34%.

---

## 5. Research & Breakthroughs

**Anthropic — "Constitutional AI 3" Paper**
Anthropic's safety team published "Constitutional AI 3", formalising a multi-round constitutional critique pipeline where the model iteratively challenges and revises its own reasoning against a constitution of principles. On internal helpfulness + harmlessness evaluations, CAI-3 shows a 42% reduction in policy violations without helpfulness regression.

**Meta FAIR — "World Model Planning" Benchmark**
Meta FAIR released a new benchmark, WM-Bench, for evaluating world-model planning across 18 procedurally-generated environments. Gemini 3.2 Ultra tops the leaderboard at 64.2%, with Mythos at 62.7% and GPT-5.1 at 58.4%. Human baseline is 81%.

**Caltech — Lightspeed Inference on Photonic Chips**
Caltech researchers, in collaboration with Lightmatter, demonstrated 1.2 petaFLOPS/W Llama 4.5-8B inference on a new photonic tensor core, an 11× energy-efficiency improvement over H200. The result, published in *Nature*, opens a path to in-datacentre photonic accelerators by late 2027.

---

## 6. Regulation & Policy

**EU AI Office — Mythos & GPT-5.1 Pre-Market Risk Tier Classifications**
The EU AI Office published pre-market classifications for Mythos and GPT-5.1, both designated as general-purpose AI with systemic risk. The classifications carry the full suite of Article 55 obligations: model evaluation, adversarial testing, cybersecurity protection, and serious-incident reporting.

**US Senate — AI Transparency Act Reintroduced**
A bipartisan group of U.S. senators reintroduced the AI Transparency Act, which would require frontier developers above a compute threshold to publish training-data summaries, pre-deployment evaluation results, and capability reports within 30 days of model release. The bill is co-sponsored by senators from both parties and industry groups have signalled cautious support.

**Australia — Voluntary AI Safety Standard v2**
Australia's National AI Centre published Version 2 of the Voluntary AI Safety Standard, expanding the guardrails section to cover agentic systems and adding guidance on shadow AI governance within enterprises.

---

## 7. Quick Hits

- **Microsoft:** Azure AI Foundry adds Gemini 3.2 Flash Thinking same-day
- **AWS:** Bedrock announces Mythos "extended thinking batch" tier for batch-mode discounts
- **Google Cloud:** Vertex AI Agent Builder adds MCP 1.1 client + A2A preview
- **Zed:** Zed Editor hits 1.0 GA with native MCP + multi-model agents
- **Replit:** Replit Ghostwriter 3 ships with Mythos-powered refactor preview
- **Cursor:** Cursor Web launches — browser-based IDE with same agent features
- **Perplexity:** Comet adds cross-tab memory spanning 10,000 pages per user
- **OpenAI:** ChatGPT Enterprise adds memory controls with per-conversation opt-outs
- **Vercel:** v0 adds multi-page app generation with Mythos backbone
- **Anthropic:** Claude Desktop 2.0 adds native Microsoft 365 + Google Workspace connectors

---

## Sources

- Anthropic Newsroom, Google DeepMind Blog, Meta AI (LlamaCon press kit)
- Alibaba Cloud, Ollama GitHub, MCP RFC repository
- TechCrunch, Bloomberg, Reuters, The Information
- European Commission (AI Office), U.S. Senate press releases, Australian DISR
- IDC flash note, Gartner insight brief, Nature (Caltech paper)

---

*Next: 2026-04-25 — Weekend deep dive: MCP A2A reference implementations, OpenAI o-series 2.0 rumours, agent-cost benchmarks*
