# Agentic AI — Day 21: Capstone — Full Production Multi-Agent System
**Track:** Agentic AI (Implementation-Focused)
**Time Target:** 1–2 hours

---

## 1. Today's Focus
**Capstone: Design and Build a Production-Ready Multi-Agent System**

This is your final project. You'll combine everything from the 21-day track into one comprehensive system. Choose one of the capstone options below, design the architecture, implement the core agents, add guardrails and observability, and deploy it as a working application.

---

## 2. Three-Week Recap

### Week 1: Foundations
| Day | What You Built | Key Skill |
|-----|---------------|-----------|
| 1 | Architecture overview | Pattern selection (ReAct, Plan-Execute, Supervisor) |
| 2 | Spring AI finance agent | Tool definitions, system prompts, guard rails |
| 3 | LangChain4j agent | AI Services, structured output, streaming |
| 4 | LangGraph research agent | State graphs, conditional edges, checkpointing |
| 5 | Custom tools (DB, API, file) | Tool design principles, output limits, safety |
| 6 | Memory systems | Short-term, long-term, semantic memory |
| 7 | Research assistant project | End-to-end single agent with multi-step reasoning |

### Week 2: Multi-Agent & Advanced
| Day | What You Built | Key Skill |
|-----|---------------|-----------|
| 8 | Supervisor & swarm patterns | Orchestration, routing, iteration limits |
| 9 | LangGraph sub-graphs, parallel | Fan-out/fan-in, human-in-the-loop |
| 10 | CrewAI role-based teams | Rapid prototyping, role/goal/backstory |
| 11 | Guardrails & approval gates | Input/output guards, cost limits, RBAC tools |
| 12 | RAG-powered agents | Agentic retrieval, self-reflective RAG |
| 13 | MCP tool integration | Standard protocol, universal tool servers |
| 14 | Customer support system | Full multi-agent with triage, specialists, escalation |

### Week 3: Production Projects
| Day | What You Built | Key Skill |
|-----|---------------|-----------|
| 15 | Production architecture | Stateless design, caching, cost tracking, streaming |
| 16 | Code review agent | GitHub integration, structured review output |
| 17 | Data analysis agent | SQL generation, chart creation, report writing |
| 18 | DevOps assistant | K8s tools, log analysis, incident diagnosis |
| 19 | E-commerce agent | Semantic product search, cart, conversation memory |
| 20 | Document processing pipeline | Classification, extraction, validation, routing |

---

## 3. Capstone Options (Choose One)

### Option A: Enterprise Knowledge Agent (Java/Spring Boot)

**Description:** A company-wide knowledge assistant that answers employee questions using internal documentation, Confluence wikis, Slack history, and Jira tickets.

**Architecture:**
```
Employee asks question
       │
       ▼
┌──────────────┐
│ Triage Agent │ → classify: HR, engineering, product, finance
└──────┬───────┘
       │
  ┌────┴──────────┬──────────────┐
  ▼               ▼              ▼
┌───────────┐ ┌──────────┐ ┌──────────┐
│ HR Agent  │ │ Eng Agent│ │ Product  │
│ (policies,│ │ (docs,   │ │ Agent    │
│  benefits)│ │  runbooks│ │ (roadmap │
└───────────┘ │  code)   │ │  specs)  │
              └──────────┘ └──────────┘
    │              │              │
    └──────────────┼──────────────┘
                   ▼
            ┌──────────┐
            │ RAG with │  pgvector / Elasticsearch
            │ multiple │  per-department indexes
            │ indexes  │
            └──────────┘
```

**Requirements:**
- [ ] Multi-department RAG with metadata filtering
- [ ] Conversation memory (Redis)
- [ ] Source citations in every answer
- [ ] Role-based access (HR docs only for HR team)
- [ ] Streaming responses via SSE
- [ ] Cost tracking and model routing (Haiku for simple, Sonnet for complex)
- [ ] Observability: OTel traces per agent step

### Option B: Autonomous Project Manager Agent (Python/LangGraph)

**Description:** An agent that monitors a GitHub project, tracks sprint progress, identifies blockers, writes status reports, and nudges developers about overdue tasks.

**Architecture:**
```
Scheduled trigger (daily)
       │
       ▼
┌──────────────────┐
│ Project Monitor  │
│ (Orchestrator)   │
└───┬──────────────┘
    │
    ├──► GitHub Agent: PRs, issues, reviews, commits
    ├──► Sprint Agent: Jira/Linear sprint progress
    ├──► Reporter Agent: compile daily status report
    └──► Nudger Agent: Slack DM for overdue tasks
```

**Requirements:**
- [ ] GitHub integration (PRs, issues, reviews)
- [ ] Sprint tracking (calculate velocity, burndown)
- [ ] Daily status report generation (Markdown)
- [ ] Slack notifications for blockers
- [ ] Human-in-the-loop before sending nudges
- [ ] Long-term memory (team velocity trends)
- [ ] Parallel execution (GitHub + Sprint agents run simultaneously)

### Option C: AI-Powered API Testing Agent (Java or Python)

**Description:** An agent that reads API documentation (OpenAPI spec), generates test cases, executes them, analyses results, and creates bug reports for failures.

**Architecture:**
```
Input: OpenAPI spec URL
       │
       ▼
┌──────────────┐
│ Spec Reader  │ → parses endpoints, schemas, auth
└──────┬───────┘
       ▼
┌──────────────┐
│ Test Planner │ → generates test cases per endpoint
└──────┬───────┘
       ▼
┌──────────────┐
│ Test Executor│ → runs HTTP requests, validates responses
└──────┬───────┘
       ▼
┌──────────────┐
│ Analyser     │ → identifies failures, classifies severity
└──────┬───────┘
       ▼
┌──────────────┐
│ Reporter     │ → generates bug report + test coverage report
└──────────────┘
```

---

## 4. Implementation Checklist (All Options)

### Core Requirements
- [ ] **Multi-agent orchestration** — supervisor or pipeline pattern
- [ ] **At least 5 custom tools** — real integrations, not mocks
- [ ] **Conversation or state memory** — Redis or checkpointing
- [ ] **Input + output guardrails** — safety screening
- [ ] **Structured outputs** — Pydantic models or Java records
- [ ] **Error handling** — graceful degradation, not crashes

### Production Requirements
- [ ] **Streaming responses** — SSE for real-time output
- [ ] **Cost tracking** — token counting, budget alerts
- [ ] **Observability** — trace each agent step, log tool calls
- [ ] **Rate limiting** — prevent abuse
- [ ] **Tests** — at least 3 integration tests

---

## 5. Evaluation Criteria

| Criterion | What to Assess |
|-----------|---------------|
| **Architecture** | Is the agent decomposition logical? Are specialists well-scoped? |
| **Tool Quality** | Are tool descriptions clear? Do they handle errors? Are outputs limited? |
| **Safety** | Are guardrails present? Can the agent be prompt-injected? |
| **Memory** | Does the agent remember context? Is memory persisted? |
| **Observability** | Can you trace a request through all agents? Can you see costs? |
| **Code Quality** | Is it clean, well-structured, and testable? |

---

## 6. Track Summary & What's Next

### What You Can Now Build
After 21 days, you have the skills to build:
- **Single agents** with tools, memory, and guardrails (Java + Python)
- **Multi-agent systems** with supervisor, swarm, and hierarchical patterns
- **RAG-powered agents** that search knowledge bases intelligently
- **Production agents** with cost control, observability, and streaming
- **Real-world applications** — code review, data analysis, DevOps, e-commerce, document processing

### Continuing Your Journey
| Next Step | Resource |
|-----------|---------|
| **LangGraph Cloud** | Deploy agents as hosted services |
| **Claude Agent SDK** | Anthropic's native agent framework |
| **Evaluation frameworks** | LangSmith, Braintrust — systematically test agent quality |
| **Fine-tuning for agents** | Train smaller models on your agent's tool-calling patterns |
| **Voice agents** | Combine with speech-to-text/text-to-speech for voice interfaces |
| **Computer use agents** | Claude's computer use for UI automation |

---

## 7. Key Takeaways

1. **Agents are software systems, not magic** — architecture, testing, observability, and deployment are all the same engineering you already know.
2. **Start simple, add complexity when earned** — a single ReAct agent with good tools beats a complex multi-agent system with bad tools.
3. **Tools make or break agents** — 80% of agent quality is tool design and descriptions.
4. **Guardrails are non-negotiable** — an agent without safety controls is a liability, not a feature.
5. **The LLM is the brain, but the system is the product** — memory, tools, routing, observability, and cost control are what make it production-ready.

---

*Day 21. You started with a prompt. You end with a system. The agents are ready — go build something real.*
