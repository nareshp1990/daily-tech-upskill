# Day 15 — Phase 1 Capstone & Review
**Date:** 2026-03-22
**Phase:** 1 — Generative AI & Agentic AI
**Time Target:** 1–2 hours

---

## 1. Today's Focus Topic
**Phase 1 Capstone & Review — Connecting All 14 Days Into One System**

You've covered 14 topics across the full Generative AI & Agentic AI landscape. Today you pause building new things and instead: review the key patterns from each day, see how they compose into a production-grade system, identify gaps in your understanding, and plan your next phase. This is the "zoom out" day — the equivalent of a sprint retrospective where you consolidate learning before moving to the next challenge. You'll also sketch the architecture of a complete agentic system using everything you've learned.

---

## 2. Phase 1 Knowledge Map

### What You've Built — End-to-End

```
┌─────────────────────────────────────────────────────────┐
│                  User Request                           │
└───────────────────────┬─────────────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────────────┐
│  Security Layer (Day 12)                                │
│  • Input sanitisation (prompt injection defence)        │
│  • Rate limiting (Resilience4j)                         │
└───────────────────────┬─────────────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────────────┐
│  Production Gateway (Day 13)                            │
│  • Semantic cache (vector similarity lookup)            │
│  • Model router (Haiku / Sonnet / Opus)                 │
│  • Budget guard (daily token limit)                     │
│  • Fallback chain (primary → fallback → static)         │
└───────────────────────┬─────────────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────────────┐
│  Orchestrator (Day 11)                                  │
│  • Routes to specialist agents                          │
│  • Sequential or parallel fan-out                       │
└──────┬──────────────────────────────────────────────────┘
       │              │                │
       ▼              ▼                ▼
┌──────────┐   ┌──────────┐    ┌──────────────┐
│ Lookup   │   │ Pricing  │    │   Report     │
│ Agent    │   │ Agent    │    │   Agent      │
│ (Day 7)  │   │ (Day 6)  │    │ (Day 11)     │
└──────────┘   └──────────┘    └──────────────┘
       │              │
       ▼              ▼
┌──────────────────────────────────────────────┐
│  Tools Layer (Day 6)                         │
│  • @Tool methods with authorisation          │
│  • Input validation (Day 12)                 │
│  • Audit logging                             │
└──────────────────────┬───────────────────────┘
                       │
      ┌────────────────┼─────────────────┐
      ▼                ▼                 ▼
┌──────────┐    ┌──────────┐    ┌──────────────┐
│  Orders  │    │ pgvector │    │ External APIs│
│  DB      │    │ (RAG,    │    │              │
│          │    │  cache,  │    │              │
│          │    │  memory) │    │              │
└──────────┘    └──────────┘    └──────────────┘

Cross-cutting:
• Memory (Day 8): short-term (session) + long-term (vector)
• Observability (Day 9): OTel traces, token metrics, LLM-as-judge eval
• Structured Output (Day 10): BeanOutputConverter on all agent responses
• Fine-tuning (Day 14): domain-adapted model for format-critical paths
```

---

## 3. Day-by-Day Pattern Summary

| Day | Topic | Core Pattern | Spring AI API |
|-----|-------|-------------|--------------|
| 1 | Generative AI Fundamentals | Tokens, inference, context window | — |
| 2 | Prompt Engineering | System prompts, few-shot, CoT | `PromptTemplate` |
| 3 | LLM APIs & SDKs | Streaming, retries, resilience | `ChatClient`, Resilience4j |
| 4 | Embeddings & Vector Search | Semantic similarity, HNSW | `EmbeddingModel`, `VectorStore` |
| 5 | RAG | Ingest → Retrieve → Augment → Generate | `QuestionAnswerAdvisor` |
| 6 | Tool Use & Function Calling | LLM as reasoning layer over services | `@Tool`, `@ToolParam` |
| 7 | Agents & Multi-Step Reasoning | ReAct loop, planning | `ChatClient` + tools loop |
| 8 | Memory & Conversation State | Short-term + long-term memory | `MessageChatMemoryAdvisor` |
| 9 | Observability & Evaluation | OTel tracing, LLM-as-judge | `ChatModelObservationContext` |
| 10 | Structured Outputs | Typed extraction, schema enforcement | `BeanOutputConverter` |
| 11 | Multi-Agent Systems | Orchestrator-worker, fan-out/fan-in | Spring beans + CompletableFuture |
| 12 | LLM Security | Prompt injection defence, tool authz | Custom advisors + Spring Security |
| 13 | Production Architecture | Model routing, caching, budget | Resilience4j + VectorStore cache |
| 14 | Fine-Tuning | Dataset prep, behaviour alignment | OpenAI/Anthropic fine-tune API |

---

## 4. Capstone Exercise

### Goal: Design (on paper/whiteboard) a complete agentic system for your own domain

Pick a realistic use case from your work (e.g., a Kafka consumer troubleshooting assistant, a Spring Boot service health advisor, a code review agent). Sketch the architecture answering these questions:

**1. What does the user ask?**
Write 3 example user requests that cover the range of complexity.

**2. Which agents do you need?**
List 2–4 specialist agents. What is each agent's single responsibility? What tools does it have?

**3. How is the orchestrator structured?**
Sequential, parallel, or router? When would you use each for your use case?

**4. What data sources feed RAG?**
What documents/knowledge would you ingest? How would you chunk them? What metadata would you add for filtering?

**5. How do you handle memory?**
Short-term: how long should a session last? Long-term: what facts are worth persisting across sessions?

**6. What are the security boundaries?**
Which tools are dangerous (destructive/sensitive)? How do you enforce authorisation at the tool layer?

**7. What metrics matter?**
What does "this system is healthy" look like in Grafana? List 5 key metrics.

**8. Where would fine-tuning add value?**
Is there a specific behaviour (format, tone, domain jargon) that prompting alone won't reliably produce?

---

## 5. Self-Assessment Checklist

Go through each item. If you can't answer confidently, mark it for review and re-read that day's plan.

**Fundamentals**
- [ ] Can you explain why context window size matters for agent design?
- [ ] Can you write a system prompt that is robust to prompt injection?

**RAG & Embeddings**
- [ ] Can you explain the difference between HNSW and flat vector search?
- [ ] Can you implement chunking strategy choice (by size vs by semantic boundary) and explain the trade-offs?

**Agents & Tools**
- [ ] Can you explain the ReAct loop and implement it in Spring AI without looking at notes?
- [ ] Can you write a `@Tool` method with proper input validation and authorisation?

**Memory**
- [ ] Can you implement session-scoped memory with `MessageChatMemoryAdvisor`?
- [ ] Can you explain when to use in-context history vs vector-store memory?

**Production**
- [ ] Can you add LLM-as-judge evaluation to a test suite?
- [ ] Can you wire a model router that selects Haiku vs Sonnet based on complexity?
- [ ] Can you explain at least 3 OWASP LLM risks and the mitigations you'd implement?

**Fine-Tuning**
- [ ] Can you explain the 3 scenarios where RAG beats fine-tuning?
- [ ] Can you write a valid fine-tuning JSONL example?

---

## 6. Key Insights From Phase 1

1. **The LLM is a reasoning layer, not a data store.** Use RAG for knowledge, tools for actions, fine-tuning for behaviour.
2. **Tools are your security boundary.** Never trust LLM output as safe; validate and authorise at the tool level regardless of what the prompt says.
3. **Observability first.** You can't debug or improve what you can't measure. Add tracing and evaluation from day one.
4. **Prompt engineering before fine-tuning.** A well-crafted prompt solves 80% of problems at zero cost. Only fine-tune when prompting and RAG have genuinely hit their limits.
5. **Agents fail in unique ways.** They can loop, hallucinate tool arguments, and get confused by ambiguous instructions. Design with max-iterations, input validation, and graceful degradation from the start.

---

## 7. Phase 2 Preview — System Design

**Starting next week, Phase 2: System Design.** You'll apply the distributed systems thinking you've seen in LLM architecture to broader engineering challenges:

| Day | Topic |
|-----|-------|
| 16 | Scalability Fundamentals — horizontal scaling, load balancing, stateless design |
| 17 | Database Design & Indexing — schema design, query optimisation, index strategies |
| 18 | Caching Strategies — Redis, cache patterns (aside, write-through, write-behind) |
| 19 | Message Queues & Event-Driven Architecture — Kafka deep dive |
| 20 | API Design — REST best practices, versioning, rate limiting, idempotency |

The system design skills in Phase 2 will make you a better AI architect — you'll design LLM systems with the same rigour you apply to high-throughput microservices.

---

## 8. Resources for Phase 1 Review

- **[Spring AI Reference Docs](https://docs.spring.io/spring-ai/reference/)** — Full reference for everything you used.
- **[Anthropic's Cookbook](https://github.com/anthropics/anthropic-cookbook)** — Production patterns in code; read the agents and tool-use notebooks.
- **[OWASP LLM Top 10](https://owasp.org/www-project-top-10-for-large-language-model-applications/)** — Bookmark and re-read before any LLM feature goes to production.

---

*Phase 1 complete. 14 days. One complete agentic AI stack. On to Phase 2.*
