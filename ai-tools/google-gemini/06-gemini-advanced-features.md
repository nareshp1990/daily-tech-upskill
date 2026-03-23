# 06 — Gemini Advanced Features

## Table of Contents

1. [Gemini Gems](#1-gemini-gems)
2. [Gemini Deep Research](#2-gemini-deep-research)
3. [Gemini in Google Workspace](#3-gemini-in-google-workspace)
4. [NotebookLM](#4-notebooklm)
5. [Gemini Live](#5-gemini-live)
6. [Extensions and Integrations](#6-extensions-and-integrations)
7. [Grounding and Citations](#7-grounding-and-citations)
8. [Safety Settings and Content Filtering](#8-safety-settings-and-content-filtering)
9. [Practical Use Cases for Senior Developers](#9-practical-use-cases-for-senior-developers)
10. [Try This Exercises](#10-try-this-exercises)

---

## 1. Gemini Gems

**Gems** are custom AI personas within Gemini Advanced. Think of them as reusable
system prompts with a personality.

### What Are Gems?

A Gem is a saved configuration of:
- Custom instructions (system prompt)
- Name and description
- Persistent across conversations

### Creating a Gem

```
1. Go to gemini.google.com → Gem Manager
2. Click "Create Gem"
3. Configure:
   - Name: "Spring Boot Architect"
   - Instructions: (see below)
4. Save
5. Start a new chat → select your Gem from the dropdown
```

### Useful Gems for Backend Developers

**Gem 1: Spring Boot Architect**
```
Name: Spring Boot Architect
Instructions:
You are a senior Spring Boot architect with 15+ years of Java experience.
You specialize in:
- Spring Boot 3.x with Java 17+
- Microservices architecture
- Event-driven systems with Kafka
- MySQL optimization and JPA/Hibernate
- Docker and Kubernetes deployments
- Azure cloud services

When reviewing code or architecture:
1. Always consider scalability (10x-100x current load)
2. Check for security (OWASP Top 10)
3. Evaluate observability (logging, metrics, tracing)
4. Consider cost implications
5. Suggest production-ready error handling

When writing code:
- Use constructor injection (never field injection)
- Use Java records for DTOs
- Include proper validation annotations
- Add Javadoc for public APIs
- Follow clean code principles

Be concise. Use bullet points. Provide code examples in Java 17+.
```

**Gem 2: Code Reviewer**
```
Name: Code Reviewer
Instructions:
You are a meticulous code reviewer. For every piece of code I share:

1. BUGS: Identify logical errors, null safety, race conditions
2. SECURITY: Check for injection, auth issues, data exposure
3. PERFORMANCE: Spot N+1 queries, missing indexes, unnecessary allocations
4. PATTERNS: Suggest better design patterns when applicable
5. TESTS: Point out untested edge cases

Format your review as:
- [CRITICAL] issues that will cause production incidents
- [HIGH] issues that should be fixed before merge
- [MEDIUM] improvements for code quality
- [LOW] style and preference suggestions

Always provide the fix, not just the problem.
```

**Gem 3: DevOps Troubleshooter**
```
Name: DevOps Troubleshooter
Instructions:
You are a DevOps expert specializing in:
- Kubernetes (AKS and GKE)
- Docker and container optimization
- Terraform (Azure and GCP)
- GitHub Actions CI/CD
- Monitoring and alerting (Prometheus, Grafana, Azure Monitor)

When I describe an issue:
1. Ask clarifying questions if needed
2. Provide a diagnostic checklist (most likely → least likely)
3. Give exact commands to run
4. Explain the fix step by step
5. Suggest preventive measures

Always consider both Azure and GCP contexts.
```

**Gem 4: System Design Interviewer**
```
Name: System Design Coach
Instructions:
You help me practice system design. When I describe a system:

1. Ask me clarifying questions about requirements
2. Guide me through: requirements → API → data model → high-level design
   → detailed design → bottlenecks → scaling
3. Challenge my decisions with "What happens if..."
4. Point out trade-offs I missed
5. Rate my design (1-10) with specific improvements

Use real numbers: QPS, storage, bandwidth estimates.
Focus on distributed systems patterns I should know.
```

---

## 2. Gemini Deep Research

Deep Research is Gemini's automated research capability that:
1. Takes your question
2. Creates a multi-step research plan
3. Browses the web extensively (multiple searches and pages)
4. Synthesizes findings into a comprehensive report
5. Includes citations and sources

### How It Works

```
You ask a question
       │
       v
Gemini creates a research plan (5-15 steps)
       │
       v
You approve/modify the plan
       │
       v
Gemini executes research (2-5 minutes)
- Searches Google multiple times
- Reads dozens of web pages
- Cross-references information
- Identifies contradictions
       │
       v
Delivers a structured report with citations
```

### Accessing Deep Research

```
1. Go to gemini.google.com (requires Gemini Advanced subscription)
2. Click the model selector → choose "Deep Research"
3. Ask your question
4. Review and approve the research plan
5. Wait 2-5 minutes for the report
```

### Use Cases for Developers

**1. Technology evaluation:**
```
"Research the current state of Java virtual threads (Project Loom) in Spring Boot 3.x.
Cover: adoption status, performance benchmarks vs traditional thread pools,
known issues, migration guide from ThreadPoolTaskExecutor, and which companies
are using it in production."
```

**2. Architecture decision records:**
```
"Research the trade-offs between Apache Kafka and Google Pub/Sub for an event-driven
microservices architecture. Include: throughput benchmarks, latency comparisons,
operational complexity, cost at 100K messages/second, exactly-once semantics,
and migration path from Kafka to Pub/Sub."
```

**3. Security audit preparation:**
```
"Research the latest OWASP Top 10 for LLM applications (2025 edition).
For each vulnerability, provide: description, real-world example,
how it applies to a Spring Boot service calling the Gemini API,
and specific mitigation code."
```

**4. Framework comparison:**
```
"Compare Spring AI vs LangChain4j for building LLM-powered Java applications.
Cover: API design, supported models, RAG support, tool/function calling,
streaming, community activity, production readiness, and code examples."
```

### Deep Research vs Regular Chat

| Aspect | Regular Chat | Deep Research |
|--------|-------------|---------------|
| Speed | Seconds | 2-5 minutes |
| Sources | Training data only | Live web search (dozens of pages) |
| Depth | Surface-level | Comprehensive report |
| Citations | None or limited | Full citations with URLs |
| Structure | Conversational | Structured document with sections |
| Best for | Quick questions | In-depth analysis, decision-making |

---

## 3. Gemini in Google Workspace

With Gemini Advanced (Google One AI Premium), Gemini integrates into Workspace apps:

### Gmail

| Feature | Example |
|---------|---------|
| Draft emails | "Write a professional email declining a meeting" |
| Summarize threads | "Summarize this 20-email thread" |
| Suggest replies | Context-aware reply suggestions |
| Tone adjustment | Make formal, casual, shorter, longer |
| Extract action items | "What are the action items from this thread?" |

**Developer use case:**
```
"Summarize this incident post-mortem email thread and extract:
1. Root cause
2. Impact (duration, affected users)
3. Action items with owners
4. Prevention measures"
```

### Google Docs

| Feature | Example |
|---------|---------|
| Generate content | "Write a technical design document for..." |
| Summarize documents | "Summarize this 30-page spec in 5 bullet points" |
| Rewrite sections | "Make this section more concise" |
| Generate tables | "Create a comparison table of..." |
| Proofread | Check grammar, clarity, tone |

**Developer use case:**
```
"Generate a technical design document for a new notification service:
- Requirements: multi-channel (email, SMS, push), templating, scheduling, tracking
- Architecture: Spring Boot, Kafka, Redis, MySQL
- Include: API design, data model, sequence diagrams, failure modes"
```

### Google Sheets

| Feature | Example |
|---------|---------|
| Generate formulas | "Calculate month-over-month growth rate" |
| Organize data | "Create a pivot table from this data" |
| Analyze trends | "What trends do you see in this data?" |
| Generate charts | Suggest and create visualizations |
| Create templates | "Build a project tracking spreadsheet" |

**Developer use case:**
```
"Create a capacity planning spreadsheet with:
- Current: 5000 RPS, 8 pods, 4 CPU / 8GB each
- Growth: 20% monthly
- Show: projected RPS, required pods, estimated cost for next 12 months
- Include formulas for CPU and memory scaling
- Add a chart showing cost growth"
```

### Google Slides

| Feature | Example |
|---------|---------|
| Generate slides | "Create a presentation about..." |
| Generate images | Create custom images for slides |
| Summarize to slides | Convert a doc into a presentation |
| Speaker notes | Generate speaker notes for each slide |

**Developer use case:**
```
"Generate a 10-slide architecture review presentation:
1. Title slide
2. Current architecture diagram description
3. Pain points (3 bullet points each)
4. Proposed changes
5. Migration plan (phased)
6-8. Technical details per phase
9. Timeline and resources
10. Q&A"
```

---

## 4. NotebookLM

**URL:** https://notebooklm.google.com

NotebookLM is an AI-powered research tool that lets you upload documents and have
AI conversations grounded entirely in your uploaded content.

### Key Features

| Feature | Description |
|---------|-------------|
| Source-grounded AI | Answers come ONLY from your uploaded documents |
| Multiple sources | Upload PDFs, Google Docs, websites, YouTube videos, text |
| Citations | Every answer includes inline citations to source material |
| Audio overviews | Generates a podcast-style audio summary of your sources |
| Notes | Save and organize AI-generated insights |
| Collaboration | Share notebooks with team members |

### How It Differs from Regular Gemini

| Aspect | Gemini Chat | NotebookLM |
|--------|-------------|------------|
| Knowledge source | General training data + web | ONLY your uploaded documents |
| Hallucination risk | Moderate | Very low (grounded in sources) |
| Citations | Optional, sometimes inaccurate | Always present, accurate |
| Best for | General questions | Deep analysis of specific documents |
| Privacy | Data may be used for training | Stronger data isolation |

### Use Cases for Developers

**1. Codebase onboarding:**
```
Upload:
- Architecture decision records (ADRs)
- README files
- API documentation
- Design documents

Then ask:
"How does the authentication flow work in this system?"
"What database schema supports the order management module?"
"Why was Kafka chosen over RabbitMQ?"
```

**2. RFC/Spec analysis:**
```
Upload: A 100-page API specification PDF

Ask:
"What are all the required fields for the createOrder endpoint?"
"What error codes can the payment service return?"
"Are there any inconsistencies between the sequence diagrams and the API spec?"
```

**3. Learning new technology:**
```
Upload:
- Spring AI documentation (PDF or URL)
- Spring Boot 3.3 release notes
- Relevant blog posts

Ask:
"How does Spring AI handle function calling with Gemini?"
"What changed in Spring Boot 3.3 that affects my Kafka configuration?"
"Compare the ChatClient API with the old ChatModel API"
```

**4. Incident investigation:**
```
Upload:
- Incident timeline document
- Relevant log exports
- Architecture diagrams
- Monitoring dashboards (screenshots)

Ask:
"What was the sequence of events that led to the outage?"
"Which services were affected and in what order?"
"Based on the architecture, what could prevent this in the future?"
```

### Audio Overviews

NotebookLM can generate a podcast-style audio discussion about your sources:
- Two AI hosts discuss your documents conversationally
- 10-30 minutes of audio
- Great for learning while commuting

```
Upload your team's architecture documents → Generate audio overview
→ Listen during your commute → Arrive with a better understanding
```

---

## 5. Gemini Live

Gemini Live is a real-time voice conversation mode:

### Features

| Feature | Description |
|---------|-------------|
| Real-time voice | Speak naturally, get instant voice responses |
| Interruption | Interrupt Gemini mid-sentence to redirect |
| Multi-modal | Show your camera/screen while talking |
| Always listening | Keeps conversation context across exchanges |
| Multiple voices | Choose from several voice options |

### Developer Use Cases

| Scenario | How to Use |
|----------|-----------|
| Rubber duck debugging | "I'm stuck on a race condition in my Kafka consumer..." |
| Architecture brainstorming | "Let's design a notification service — push back on my ideas" |
| Code explanation | Point camera at code on screen, ask questions verbally |
| Learning | "Explain the Saga pattern like I'm teaching it to a junior dev" |
| Interview prep | "Ask me system design questions and evaluate my answers" |

### Availability

- Gemini app on Android and iOS (Gemini Advanced required)
- Currently expanding to web interface

> **Practical tip:** Use Gemini Live for brainstorming and exploration sessions where
> typing feels slow. Switch to text for precise code generation.

---

## 6. Extensions and Integrations

Gemini supports extensions that connect it to external services:

### Built-in Extensions

| Extension | What It Does |
|-----------|-------------|
| Google Search | Ground responses in web results |
| Google Maps | Location, directions, places info |
| YouTube | Search and summarize videos |
| Google Flights | Find flight information |
| Google Hotels | Find hotel information |
| Google Workspace | Access Docs, Sheets, Drive, Gmail |

### Workspace Extension for Developers

With the Workspace extension enabled:

```
"Search my Google Drive for the architecture review document from last month"

"Find the email thread about the database migration and summarize the decision"

"Look at the spreadsheet 'Q1 Capacity Planning' and tell me when we'll need
to scale up our database"
```

### Custom Extensions (Vertex AI)

On Vertex AI, you can build custom extensions:

```json
{
  "name": "jira-extension",
  "description": "Interact with Jira for ticket management",
  "openApiSpec": {
    "uri": "gs://my-bucket/openapi/jira-api.yaml"
  },
  "authConfig": {
    "oauth2": {
      "clientId": "...",
      "clientSecret": "..."
    }
  }
}
```

This lets Gemini:
- Create Jira tickets from bug descriptions
- Look up ticket status
- Update ticket fields
- Link related tickets

---

## 7. Grounding and Citations

### How Grounding Works

```
Without grounding:
  User: "What's the latest Spring Boot version?"
  Model: "Spring Boot 3.2.0" (from training data — may be outdated)

With grounding:
  User: "What's the latest Spring Boot version?"
  Model: "Spring Boot 3.4.1 (released January 2026)" [source: spring.io]
  (searches web for current info)
```

### Types of Grounding

| Type | Source | Use Case |
|------|--------|----------|
| Google Search | Live web | Current info, real-time data |
| Enterprise data (Vertex AI) | Your documents | Internal knowledge base |
| None | Training data only | General knowledge |

### Citation Format in Responses

```
Grounded response:

"Spring Boot 3.4 introduced support for virtual threads as the default
thread model for embedded Tomcat [1]. This change, combined with the
new structured concurrency API [2], significantly simplifies reactive-style
programming.

[1] https://spring.io/blog/2025/11/spring-boot-3-4-release
[2] https://openjdk.org/jeps/462"
```

### Grounding via API

```java
// Spring AI with grounding tool
public String groundedQuery(String question) {
    return chatClient.prompt()
        .system("Always ground your answers in Google Search when asked about " +
                "current versions, recent changes, or latest best practices.")
        .user(question)
        // Grounding is configured in Vertex AI model options
        .call()
        .content();
}
```

---

## 8. Safety Settings and Content Filtering

### Safety Categories

Gemini models include built-in safety filters:

| Category | What It Filters |
|----------|----------------|
| HARM_CATEGORY_HARASSMENT | Bullying, threats |
| HARM_CATEGORY_HATE_SPEECH | Discriminatory content |
| HARM_CATEGORY_SEXUALLY_EXPLICIT | Sexual content |
| HARM_CATEGORY_DANGEROUS_CONTENT | Harmful instructions |

### Threshold Levels

| Level | Behavior |
|-------|----------|
| BLOCK_NONE | No filtering (use with caution) |
| BLOCK_ONLY_HIGH | Only block high-confidence harmful content |
| BLOCK_MEDIUM_AND_ABOVE | Block medium and high (default) |
| BLOCK_LOW_AND_ABOVE | Most aggressive filtering |

### Configuration

```json
{
  "safetySettings": [
    {
      "category": "HARM_CATEGORY_DANGEROUS_CONTENT",
      "threshold": "BLOCK_ONLY_HIGH"
    },
    {
      "category": "HARM_CATEGORY_HARASSMENT",
      "threshold": "BLOCK_MEDIUM_AND_ABOVE"
    }
  ]
}
```

### Spring AI Configuration

```yaml
spring:
  ai:
    vertex-ai:
      gemini:
        chat:
          options:
            safety-settings:
              - category: HARM_CATEGORY_DANGEROUS_CONTENT
                threshold: BLOCK_ONLY_HIGH
```

### Handling Blocked Responses

```java
// Check if response was blocked
public String safeGenerate(String prompt) {
    try {
        ChatResponse response = chatClient.prompt()
            .user(prompt)
            .call()
            .chatResponse();

        // Check finish reason
        String finishReason = response.getResult().getMetadata()
            .getFinishReason();

        if ("SAFETY".equals(finishReason)) {
            log.warn("Response blocked by safety filter for prompt: {}",
                prompt.substring(0, Math.min(50, prompt.length())));
            return "I cannot provide a response to this query due to safety guidelines.";
        }

        return response.getResult().getOutput().getContent();
    } catch (Exception e) {
        log.error("Gemini API error", e);
        return "An error occurred while processing your request.";
    }
}
```

### Best Practices for Production

1. **Do not set BLOCK_NONE** in production — always have some safety filtering
2. **Log blocked responses** — monitor for false positives
3. **Add your own content filtering** on top of Gemini's — model safety alone is not sufficient
4. **Rate limit users** to prevent abuse
5. **Audit API calls** for compliance

---

## 9. Practical Use Cases for Senior Developers

### Daily Productivity Gains

| Task | Time Without AI | Time With Gemini | Feature Used |
|------|-----------------|------------------|-------------|
| Research a new library | 2-3 hours | 15-30 min | Deep Research |
| Write ADR (Architecture Decision Record) | 1-2 hours | 20-30 min | Gems + Docs |
| Prepare design review slides | 2-3 hours | 30-45 min | Slides + Gems |
| Analyze incident logs | 1-2 hours | 15-30 min | NotebookLM |
| Write API documentation | 1-2 hours | 20-30 min | Gems + Docs |
| Capacity planning | 2-3 hours | 30-45 min | Sheets + Deep Research |
| Learn from a tech talk video | 1 hour viewing | 10 min summary | NotebookLM (YouTube) |

### Workflow: Writing an ADR

```
Step 1: Create "ADR Writer" Gem with your team's ADR template

Step 2: Start conversation:
"I need to write an ADR for choosing between Apache Kafka and AWS SQS
for our event-driven order processing system. We need:
- 50K events/second at peak
- At-least-once delivery
- Event replay for last 7 days
- Integration with Spring Boot and Docker/K8s on Azure"

Step 3: Gemini drafts the ADR with:
- Context and problem statement
- Decision drivers (requirements)
- Considered options (with pros/cons)
- Decision outcome (with justification)
- Consequences (positive, negative, risks)

Step 4: Review, edit, and finalize in Google Docs

Step 5: Use Gemini in Docs to refine:
"Make the consequences section more specific with metrics"
```

### Workflow: Technical Learning

```
Step 1: Use Deep Research:
"Research Java virtual threads: current state, Spring Boot integration,
performance characteristics, when NOT to use them, migration path"

Step 2: Upload the research report + relevant docs to NotebookLM

Step 3: Ask specific implementation questions:
"Based on these sources, how do I migrate my ThreadPoolTaskExecutor-based
Kafka consumer to virtual threads?"

Step 4: Generate Audio Overview for commute listening

Step 5: Use Gemini Gem (Spring Boot Architect) to write actual code:
"Generate a Spring Boot 3.3 configuration that uses virtual threads
for Tomcat, scheduled tasks, and Kafka consumers"
```

---

## 10. Try This Exercises

### Exercise 1: Create Your First Gem (10 min)
1. Go to gemini.google.com → Gem Manager
2. Create a "Code Reviewer" Gem (use the instructions from section 1)
3. Paste a piece of your code and get a review
4. Compare: is the Gem's review better than a regular Gemini chat?

### Exercise 2: Deep Research (15 min)
1. Open Gemini → select Deep Research
2. Ask: "Research the current best practices for securing Spring Boot
   microservices that call LLM APIs. Include: API key management,
   prompt injection prevention, output validation, rate limiting,
   and cost controls."
3. Review the research plan before approving
4. Read the report — identify 3 actionable items for your projects

### Exercise 3: NotebookLM for Learning (20 min)
1. Go to notebooklm.google.com
2. Create a new notebook
3. Upload: Spring AI documentation (URL) and a relevant blog post
4. Ask 5 questions about Spring AI features
5. Generate an Audio Overview and listen to it

### Exercise 4: Workspace Integration (15 min)
1. Open Google Docs → click the Gemini icon
2. Ask: "Generate a technical design document outline for a rate limiting service"
3. Let Gemini fill in each section
4. Use "Help me write" to refine the requirements section
5. Open Google Sheets → create a capacity planning spreadsheet with Gemini

### Exercise 5: Safety Settings Experiment (10 min)
1. In AI Studio, try the same prompt with different safety thresholds
2. Find a prompt that gets blocked at the default level
3. Understand why it was blocked (check safety ratings in the response)
4. Write a Java wrapper that gracefully handles blocked responses

---

## Key Takeaways

1. **Gems** are reusable system prompts — create one per role (reviewer, architect, debugger)
2. **Deep Research** replaces hours of manual research with a 5-minute automated report
3. **NotebookLM** is groundbreaking for source-grounded analysis — zero hallucination risk
4. **Workspace integration** saves time on documentation, planning, and communication
5. **Gemini Live** is best for brainstorming where typing feels slow
6. **Grounding** prevents outdated information — use it for anything time-sensitive
7. **Safety settings** are essential for production — always have filtering enabled
8. Combined, these features save **5-10 hours per week** for a senior developer

---

Next: [07 — Daily Workflow Playbook](07-daily-workflow-playbook.md)
