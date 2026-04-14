# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

---

## Repository Purpose

A structured daily learning journal for a **Senior Backend Engineer** (12+ years, Java/Spring Boot/Spring Cloud/MySQL/Kafka/Docker/K8s/Azure/Terraform/GitHub Actions). One topic per day, 1–2 hours, committed and pushed to GitHub daily.

---

## Directory Structure

```
daily-learning-plans/YYYY-MM/DD-<slug>.md   ← Main learning plan files
agentic-ai/NN-<slug>.md                     ← Parallel 21-day agentic AI track
ai-tools/                                   ← Deep-dive guides per AI tool (Claude, Copilot, Gemini, IntelliJ)
ai-daily-news/YYYY-MM/DD-ai-news.md         ← Daily AI industry news digest
prompt-library/                             ← Stack-specific prompt collections (Java, Kafka, K8s, etc.)
ai-tools-directory.md                       ← 200+ AI tools reference
graphify-out/                               ← Knowledge graph outputs (do not edit manually)
ai-prompt.md                                ← The original system prompt that defines the user context
```

---

## Daily Workflow

When the user asks to "complete today's tasks" or "what's the plan today", the full workflow is:

1. Check `README.md` to find the last completed day and determine today's day number and topic
2. Create the **learning plan** at `daily-learning-plans/YYYY-MM/DD-<slug>.md`
3. Create the **AI daily news** at `ai-daily-news/YYYY-MM/DD-ai-news.md`
4. Update **README.md** — add the new learning plan row to the correct Phase section and add the news row to the AI Daily News table
5. `git add` the specific files, `git commit`, `git push origin main`

---

## Learning Plan Format

Every learning plan file follows this structure (see any existing file for reference):

```
# Day N — <Topic Title>
**Date:** YYYY-MM-DD
**Phase:** N — <Phase Name>
**Time Target:** 1–2 hours

## 1. Today's Focus Topic
## 2. <Core Concept Section(s)>   ← varies by topic
## N-2. Interview Questions        ← always include
## N-1. Today's Practice Exercise  ← always include
## N. Key Takeaways               ← always include
## N+1. Resources                 ← always include

*Next: Day N+1 — <Next Topic>*
```

- Code examples must be **Java/Spring Boot** unless the topic is inherently Python (LangGraph, CrewAI)
- Tie examples to the user's existing stack: Spring Boot 3.x, Spring Cloud, Kafka, MySQL, Redis, K8s/AKS

---

## Learning Phases & Current Progress

| Phase | Focus | Status |
|-------|-------|--------|
| 1 | Generative AI & Agentic AI (Days 1–15) | Complete |
| 2 | System Design (Days 16–36) | Complete |
| 3 | Microservices & Cloud-Native | **In Progress** — Day 38 done (2026-04-14) |
| 4 | DevOps & Platform Engineering | Not started |
| 5 | Emerging Backend Tech | Not started |

**Phase 3 Week 1 remaining topics:** Service Mesh/Istio → Docker deep-dive → Kubernetes Core → K8s Advanced → Week 1 Capstone

**Parallel track:** `agentic-ai/` — 21-day project-driven track (Days 1–21 complete).

---

## AI Daily News Format

```
# AI Daily News — YYYY-MM-DD (Weekday)

## Headline: <3 major stories joined with commas>

## 1. Frontier Model Updates
## 2. Open-Source Highlights
## 3. Agentic AI & Tooling
## 4. Funding & Business
## 5. Research & Breakthroughs
## 6. Regulation & Policy
## 7. Quick Hits   ← bullet list of minor items

*Next: YYYY-MM-DD — <teaser headline>*
```

---

## README Index Rules

- **Phase sections** in README must stay in order (Phase 1 → 2 → 3 → AI Daily News → Agentic Track)
- Each learning plan gets one table row: `| YYYY-MM-DD | Full Topic Title — subtitle | [View](path) |`
- Each news entry gets one row: `| YYYY-MM-DD | Three-story headline | [View](path) |`
- When a new Phase starts, add a new `### Phase N — <Name>` heading with a `#### YYYY-MM` sub-heading

---

## Knowledge Graph (graphify RAG)

A knowledge graph of this codebase lives at `graphify-out/graph.json`.

**Before answering questions about content in this repo** (learning plans, AI tools, agentic-ai topics, system design, prompt library), query the graph first:

```
/graphify query "<your question>"
/graphify path "RAG" "Multi-Agent Orchestration"
/graphify explain "Model Context Protocol"
```

The graph covers 128 nodes across 21 communities. Key hub nodes: Prompt Library Overview, Anthropic Claude AI Tools, Daily Tech Upskill Project, MCP, Multi-Agent Orchestration.

**Rebuild after adding new files:**
```
/graphify . --update
```
