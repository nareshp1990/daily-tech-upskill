# AI Daily News — 2026-04-22 (Wednesday)

## Headline: Claude Mythos Enterprise GA Ships with Agentic Controls, Google Unveils Gemini 3.2 with 5M-Token Context, EU AI Office Publishes Q1 Enforcement Summary

---

## 1. Frontier Model Updates

**Anthropic — Claude Mythos Enterprise GA**
One day after its public preview, Anthropic promoted Claude Mythos to Enterprise GA with a new set of agentic controls: tool-use budgets, approval gates, and per-tenant rate pools. Enterprise customers get dedicated throughput at 3M tokens/minute tier, and Mythos's "extended reasoning" mode is now governed by an admin-configurable token budget per request. Mythos also lands on AWS Bedrock, Google Vertex AI, and Azure AI Foundry simultaneously — a first for an Anthropic release.

**Google — Gemini 3.2 with 5M Token Context**
Google DeepMind released Gemini 3.2 Ultra with a 5M-token effective context window, backed by a new "Ring Attention v2" inference pipeline. The model scored 76.9% on SWE-bench Verified and 90.2% on MATH-500. Most notably, Gemini 3.2 introduces a native "agent graph" execution mode — the model emits a DAG of tool calls that the runtime executes in parallel, rather than one-at-a-time tool calls.

**OpenAI — GPT-5.1 Turbo Leak**
Unconfirmed benchmark numbers for an unreleased GPT-5.1 Turbo appeared on the lmsys-chatbot-arena preview board before being pulled: 1,428 Elo, placing it between GPT-5.1 and a hypothetical GPT-5.2. OpenAI has not commented. Turbo is widely expected to be the cost-optimised variant for the upcoming Responses API GA.

---

## 2. Open-Source Highlights

**Meta — Llama 4.5 Vision Released**
Meta released Llama 4.5 Vision, a 70B multimodal variant with native video understanding (up to 60 minutes per clip). Llama 4.5 Vision posts 81.3% on Video-MME and ties Gemini 3.1 Ultra on ActivityNet QA. Released under the Llama Community License 3.1; commercial use permitted below 900M MAU.

**Unsloth 2026.4 Released**
Unsloth 2026.4 adds single-H200 fine-tuning of Llama 4.5-70B via 2-bit LoRA, reducing memory footprint to 78GB peak. The release also includes a new "grokking scheduler" that automates learning-rate warm-up for small, high-quality datasets.

---

## 3. Agentic AI & Tooling

**LangGraph 0.3 — Durable Execution GA**
LangChain released LangGraph 0.3 with durable execution GA: long-running agent runs can now checkpoint to Postgres/Redis/SQLite and resume after crashes, deployments, or human approval waits. The release integrates with Temporal for enterprise-grade workflow orchestration.

**Windsurf Integrates Claude Mythos**
Windsurf (formerly Codeium) shipped Mythos integration across its IDE and agentic coding product. Windsurf reports a 31% improvement in repository-scale task completion on an internal benchmark versus GPT-5.1.

**CrewAI Enterprise 2.0 Launches**
CrewAI released Enterprise 2.0 with multi-tenant isolation, fine-grained audit logs, SSO via SAML and OIDC, and first-class observability exports to Datadog, New Relic, and Grafana Cloud. Pricing starts at $24k/year for the Team plan.

---

## 4. Funding & Business

**Figure AI Closes $3B Series E at $42B Valuation**
Figure AI closed a $3B Series E led by Microsoft and NVIDIA at a $42B post-money valuation. Funds will expand production of the Figure 03 humanoid to 12,000 units/year by end-2026. BMW, Mercedes, and a Tier-1 logistics customer have signed multi-year commercial deployments.

**Mistral Raises €1.4B at €18B Valuation**
Mistral AI closed a €1.4B Series D led by General Catalyst and a consortium of European sovereign funds, valuing the company at €18B. The round triples Mistral's valuation from its June 2025 raise and is earmarked for European infrastructure and open-weight research.

**Anthropic Doubles Down on Sovereign AI**
Anthropic signed sovereign deals with the UK, France, and Japan valued at a combined $4.2B over five years. Under the agreements, Anthropic will operate dedicated Mythos instances in-country and support each government's AI safety institute with red-team access.

---

## 5. Research & Breakthroughs

**DeepMind — AlphaProof 2 Solves 5 / 6 IMO 2026 Problems**
DeepMind disclosed that AlphaProof 2, a specialised proof-search model built on Gemini 3.2, solved 5 of 6 problems on the 2026 IMO qualifying set at gold-medal level. AlphaProof 2 couples an LLM policy with a Lean 4 proof verifier and a new "lemma retrieval" module over a 100M-theorem library.

**Stanford — "Cascade-Eval" Detects LLM Reasoning Collapse**
Stanford NLP published "Cascade-Eval", a method for detecting when LLM chain-of-thought reasoning breaks down on progressively harder problems. Across 12 frontier models, Cascade-Eval finds reasoning collapse thresholds at problem-difficulty levels where standard pass@1 scores are still above 70%.

**NVIDIA Research — GR00T N2 Foundation Model**
NVIDIA Research released GR00T N2, a humanoid-robotics foundation model trained on 2.1B video tokens + 58M simulated episodes. GR00T N2 supports zero-shot transfer across 11 humanoid platforms and is available via NVIDIA Isaac on Rubin GPUs.

---

## 6. Regulation & Policy

**EU AI Office — Q1 2026 Enforcement Summary**
The EU AI Office published its first quarterly enforcement summary. Highlights: 14 general-purpose AI providers formally notified of classification decisions, 3 under preliminary investigation for transparency-obligation gaps, and €42M in precautionary fines issued (none final). The office also published a draft template for the "sufficiently detailed training data summary" disclosure due in August.

**California SB-1047 Amendments Pass Committee**
Amendments to California SB-1047 cleared the Appropriations Committee, tightening compute-threshold reporting requirements for frontier developers and expanding whistleblower protections. A floor vote is expected before summer recess.

**UK AI Safety Institute — Renamed AI Security Institute, Mandate Expanded**
The UK AI Safety Institute was formally renamed the AI Security Institute (AISI) with an expanded remit that now includes national-security evaluations, CBRN uplift testing, and cyber-offense capability assessments. Budget rises to £300M over three years.

---

## 7. Quick Hits

- **Cursor:** Cursor 1.0 GA ships with multi-agent pair-programming
- **Perplexity:** Comet for Teams launches with shared workspaces and admin controls
- **Hugging Face:** Inference Endpoints adds Mythos in-console (one-click deploy)
- **Vercel:** AI Gateway adds failover routing across Mythos / GPT-5.1 / Gemini 3.2
- **Databricks:** Mosaic AI adds native Mythos model serving
- **Snowflake:** Cortex Agents framework supports MCP 1.1 tools
- **Apple:** Swift Assist adds on-device Llama 4.5-8B for private code completions
- **Adobe:** Firefly 5 integrates Runway Gen-4 video backbone
- **Amazon:** Rufus gets Mythos upgrade for shopping recommendations
- **xAI:** Grok 3.5 rumoured for May keynote at xAI Developer Day

---

## Sources

- Anthropic Newsroom, Google DeepMind Blog, OpenAI Blog
- Hugging Face, Unsloth GitHub, LangChain Blog, CrewAI Blog
- TechCrunch, The Information, Bloomberg, Reuters
- European Commission (AI Office), UK AI Security Institute
- DeepMind Publications, Stanford NLP, NVIDIA Research

---

*Next: 2026-04-23 — Mythos API pricing flash, OpenAI Responses API GA, first MCP security CVE disclosed*
