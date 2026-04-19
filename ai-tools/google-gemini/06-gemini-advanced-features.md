# 06 — Gemini Advanced Features

## Table of Contents

1. [Gemini Model Lineup (2025–2026)](#1-gemini-model-lineup-20252026)
2. [Gemini Gems](#2-gemini-gems)
3. [Gemini Deep Research](#3-gemini-deep-research)
4. [Gemini Canvas](#4-gemini-canvas)
5. [Gemini Scheduled Actions](#5-gemini-scheduled-actions)
6. [Gemini in Google Workspace](#6-gemini-in-google-workspace)
7. [NotebookLM](#7-notebooklm)
8. [Gemini Live](#8-gemini-live)
9. [Gemini for Mac & Gemini in Chrome](#9-gemini-for-mac--gemini-in-chrome)
10. [Gemini Code Assist — 2026 Updates](#10-gemini-code-assist--2026-updates)
11. [Extensions and Integrations](#11-extensions-and-integrations)
12. [Grounding and Citations](#12-grounding-and-citations)
13. [Safety Settings and Content Filtering](#13-safety-settings-and-content-filtering)
14. [Deprecations](#14-deprecations)
15. [Practical Use Cases for Senior Developers](#15-practical-use-cases-for-senior-developers)
16. [Try This Exercises](#16-try-this-exercises)

---

## 1. Gemini Model Lineup (2025–2026)

### Gemini 2.5 Family (GA — June 2025)

| Model | Context | Strengths | Pricing (Input/Output per MTok) |
|-------|---------|-----------|-------------------------------|
| **Gemini 2.5 Pro** | 1M (2M coming) | Adaptive thinking, best reasoning quality | ~$1.25 / $10 (doubles above 200K tokens) |
| **Gemini 2.5 Flash** | 1M | Speed + thinking balance, 30 HD voices | Low cost |
| **Gemini 2.5 Flash-Lite** | 1M | Lowest cost in 2.5 family | Lowest |
| **Gemini 2.5 Deep Think** | 1M | Enhanced iterative thinking (Ultra tier only) | — |

### Gemini 3.x Family (Nov 2025 – Apr 2026)

| Model | Released | Strengths |
|-------|----------|-----------|
| **Gemini 3 Pro** | Nov 18, 2025 | PhD-level reasoning, enhanced multimodal |
| **Gemini 3 Flash** | Dec 17, 2025 | Default app model; replaced 2.5 Flash |
| **Gemini 3 Deep Think** | Dec 4, 2025 | Ultra-exclusive iterative reasoning |
| **Gemini 3.1 Pro** | Feb 19, 2026 | Complex problem-solving, visual explanations, multi-step planning |
| **Gemini 3.1 Flash** | Preview | High-performance, low-latency |
| **Gemini 3.1 Flash-Lite** | Mar 3, 2026 | Most cost-efficient option |
| **Gemini 3.1 Flash-Live** | Mar 26, 2026 | Real-time audio-to-audio streaming |
| **Gemini 3.1 Flash-Image** | Feb 26, 2026 | Speed-optimized image generation |
| **Gemini 3.1 Flash TTS** | Apr 15, 2026 | Expressive, steerable text-to-speech; multi-speaker, 24+ languages |

### Specialized Models

| Model | Released | Capability |
|-------|----------|-----------|
| **Imagen 4** Ultra/Standard/Fast | Aug 14, 2025 | Improved text-in-image, instruction-following, typography |
| **Veo 3.1 / 3.1-fast** | Oct 15, 2025 | Video generation with audio (dialogue, SFX, music); 4K output |
| **Veo 3.1-lite** | Mar 31, 2026 | Most cost-efficient video generation |
| **Lyria 3** clip + pro | Mar 25, 2026 | Music generation from text and images |
| **Gemini Embedding 001** | Jul 14, 2025 GA | Text embeddings |
| **Gemini Embedding 2** | Mar 10, 2026 preview | First multimodal embedding: text, image, video, audio, PDF |
| **Gemini Robotics ER-1.6** | Apr 14, 2026 | Instrument reading, spatial reasoning |

### New API Platform Features

| Feature | Date | Description |
|---------|------|-------------|
| **Inference tiers** | Apr 1, 2026 | Flex tier (cost-optimize) vs Priority tier (latency-optimize) |
| **Computer Use tool** | Jan 29, 2026 | Gemini can interact with desktop UIs |
| **Built-in Tools + Function Calling** | Mar 18, 2026 | Combine built-in tools (search, code) with custom function calling in one request |
| **Google Maps grounding** | Mar 18, 2026 | Extended to Gemini 3 models |
| **File API: 100MB limit** | Jan 8, 2026 | Up from 20MB; Cloud Storage bucket and pre-signed URL support |
| **File Search API** | Nov 6, 2025 | Search within uploaded files |
| **Live API: 24hr sessions** | — | Session resumption, configurable VAD, context window compression, 30 HD voices, async function calls |
| **Batch API** | Jul 7, 2025 | Async processing; embeddings support added Sep 10, 2025 |
| **Context caching** | Apr 2025 | Available for Gemini 2.0 Flash and above |
| **Multimodal function responses** | Dec 17, 2025 | Functions can return images, audio, video |
| **Pre/Postpay billing** | Mar 23, 2026 | Project-level and account-level spend caps |
| **Gemini Embedding 2** | Mar 10, 2026 | Multimodal embeddings for text, image, video, audio, PDF |

### Subscription Tiers (2026)

| Plan | Price | Key Access |
|------|-------|-----------|
| **Free** | $0 | Gemini 3 Flash, limited usage, Deep Research (free tier) |
| **Google AI Pro** | ~$20/mo (rebranded from Gemini Advanced, May 2025) | Gemini 3.1 Pro, Canvas, Workspace integration, Gems, higher limits |
| **Google AI Ultra** | ~$250/mo | 1M context, priority access, Deep Think, Veo 3, all preview features |

---

## 2. Gemini Gems

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

### Shareable Gems (September 2025)

Gems can now be shared with team members — similar to Google Drive sharing:

```
1. Open Gem Manager → select a Gem
2. Click "Share" → add team members by email
3. Set permission: View only or Edit
4. Share the link — teammates can start conversations with the shared Gem
```

**Team use case**: Create a "Code Reviewer" Gem with your team's standards, share it with all engineers. Consistent review criteria across the team without copying instructions.

---

## 3. Gemini Deep Research (Updated — Gemini 3.x Thinking)

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

**2026 Updates to Deep Research:**
- Upgraded to **Gemini 3.x Flash Thinking** model (faster, more thorough)
- **Free tier access** — no longer requires paid subscription
- **Canvas integration** — output can be opened directly in Canvas for interactive editing

---

## 4. Gemini Canvas (March 2025)

Canvas is a collaborative document co-creation space within Gemini — a living workspace where AI and the user work together on documents and code.

### What Canvas Can Do

| Capability | Description |
|-----------|-------------|
| **Document co-creation** | Draft, edit, and refine long documents iteratively |
| **Code iteration** | Write, test, and refine code with real-time preview |
| **Web page generation** | Generate complete HTML/CSS/JS pages with live preview |
| **Infographic creation** | Generate visual infographics from data or descriptions |
| **Quiz generation** | Create interactive quizzes from uploaded content |
| **Audio overview** | Generate podcast-style audio summary of the document |
| **Vibe coding** | Build apps conversationally, iterate in real time |
| **Cross-user data sharing** | Share Canvas documents with teammates (Drive-style) |

### Using Canvas for Technical Documentation

```
1. Open Gemini → type your request with "use Canvas" or click Canvas icon

2. Example prompts:
   "In Canvas, write a technical design document for a rate limiting service.
    Include: requirements, API design, data model, failure modes."

   "In Canvas, create an API reference for our order service with endpoints,
    request/response examples, and error codes."

3. Iterate interactively:
   "Add a section on Redis implementation"
   "Make the architecture diagram section more detailed"
   "Generate an audio overview of this doc"

4. Share the Canvas link with your team for collaborative review
```

### Canvas + Deep Research Workflow

```
Step 1: Use Deep Research for a technology evaluation
Step 2: Click "Open in Canvas" from the research report
Step 3: Interactively shape the output into an ADR or design doc
Step 4: Share with the team via the Canvas link
```

---

## 5. Gemini Scheduled Actions (June 2025)

Scheduled Actions let you set up recurring or one-time automated tasks in the Gemini app:

### Setting Up Scheduled Actions

```
1. In Gemini app → click the clock/schedule icon
2. Configure:
   - Prompt: "Summarize the top 5 Java/Spring Boot news items from this week"
   - Frequency: Daily at 8:00 AM / Weekly Monday / One-time
   - Delivery: Notification or email
3. Gemini runs the task automatically at the scheduled time
```

### Developer Use Cases

```
# Daily standup prep
"Every weekday at 8:30 AM:
 Summarize my Google Calendar for today, highlight any new items
 added since yesterday, and list my top 3 priorities."

# Weekly tech digest
"Every Monday at 9:00 AM:
 Research and summarize: 1) major Java/Spring releases or CVEs from last week,
 2) Kubernetes updates, 3) Azure announcements relevant to AKS."

# Monthly architecture review reminder
"On the 1st of each month:
 Create a checklist of architecture review items for a microservices system
 running on AKS with Kafka and MySQL."
```

---

## 6. Gemini in Google Workspace

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

## 7. NotebookLM

**URL:** https://notebooklm.google.com

NotebookLM is an AI-powered research tool that lets you upload documents and have
AI conversations grounded entirely in your uploaded content.

**2026 Update:** NotebookLM is now integrated into the Gemini app — notebooks appear as chat projects with deep NotebookLM integration. You can pull notebooks as sources directly into Gemini conversations.

### Key Features

| Feature | Description |
|---------|-------------|
| Source-grounded AI | Answers come ONLY from your uploaded documents |
| Multiple sources | Upload PDFs, Google Docs, websites, YouTube videos, text |
| Citations | Every answer includes inline citations to source material |
| Audio overviews | Generates a podcast-style audio summary of your sources |
| Notes | Save and organize AI-generated insights |
| Collaboration | Share notebooks with team members |
| **Gemini app integration** | Access notebooks directly from Gemini chat (2026) |
| **Temporary Chat** | Chat sessions not saved to history, excluded from training (Aug 2025) |
| **Chat History Search** | Full-text search across all NotebookLM conversations (Aug 2025) |

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

## 8. Gemini Live

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

## 9. Gemini for Mac & Gemini in Chrome

### Gemini for Mac (April 15, 2026)

A native macOS app — no browser tab required.

| Feature | Details |
|---------|---------|
| **Global shortcut** | `Option + Space` — invoke Gemini from anywhere on your Mac |
| **Window sharing** | Share any app window with Gemini for context |
| **System context** | Gemini can see what's on your screen when you share a window |
| **Availability** | macOS 15+, requires Google AI Pro or Ultra subscription |

**Developer use case:**
```
1. Press Option+Space with your IDE in focus
2. Share the IDE window with Gemini
3. Ask: "What's wrong with the method on line 47?"
4. Gemini reads your code directly — no copy-paste needed
```

### Gemini in Chrome (May 2025)

Context-aware web analysis without switching tabs:

- **Access**: Side panel in Chrome (Windows and macOS)
- **Capability**: Analyse the current web page — documentation, articles, GitHub PRs
- **Requires**: Google AI Pro or Ultra subscription

```
While reading a Java Spring Security documentation page:
1. Open Gemini side panel
2. Ask: "How does this OAuth2 configuration compare to our current setup?"
3. Gemini reads the page and answers in context
```

---

## 10. Gemini Code Assist — 2026 Updates

Gemini Code Assist (the IDE developer tool) received major updates in 2025–2026.

### Finish Changes (GA — March 2026)

AI pair programmer that completes in-progress work — tell it what you want and it finishes the code:

```
# In IntelliJ or VS Code, write a pseudocode comment or partial implementation:

// TODO: add retry logic with exponential backoff, max 3 retries, delay starting at 1s
public void callExternalService(String url) {
    // stub
}

# Then ask Gemini Code Assist: "Finish this"
# Result: complete implementation with Resilience4j Retry or manual retry loop
```

**Works from:**
- TODO comments and pseudocode
- Half-written code
- Natural language descriptions alongside partial implementations

**GA dates:** VS Code (March 4, 2026), IntelliJ (February 24, 2026)

### Outline Feature (GA — 2026)

Automatically generates English summaries of code blocks in the Outline tab:

- Opens alongside the file in the IDE's outline view
- Shows natural-language description of each method and class
- Helps orient quickly in unfamiliar codebases
- Available in both VS Code and IntelliJ

### Next Edit Predictions (Preview — September 2025)

Forecasts the next code change you're likely to make based on the current edit:

```java
// You rename this field:
private final OrderService orderService;  →  private final OrderManager orderManager;

// Next Edit Prediction offers:
// → Update constructor parameter name
// → Update all usages within the class
// → Update the @Bean method name in config
```

**Status:** Preview in VS Code (September 26, 2025)

### Agent Mode in Code Assist (2025–2026)

| Feature | Date | Description |
|---------|------|-------------|
| **Agent mode GA — IntelliJ** | Jul 31, 2025 | Full agent mode (plan + execute) in JetBrains |
| **Auto-approve with rollback** | Jul 24, 2025 | Agent mode changes can be approved automatically; rollback on failure |
| **MCP server integration** | Oct 14, 2025 | Connect external services to Code Assist agent via MCP |
| **`/deploy` command** | Sep 10, 2025 | Direct Cloud Run deployment from agent mode |
| **`@` remote repo mentions** | Sep 3, 2025 | Prioritize specific repository context in chat |
| **Gemini on GitHub** | Nov 10, 2025 | Enterprise Code Assist with multi-repo support and persistent memory for GitHub |
| **Gemini 3.1 Pro in Code Assist** | Mar 13, 2026 (preview) | Agent mode, chat, code gen using Gemini 3.1 Pro |

### Code Customization

- **Repository indexing** — Index your organization's repos for project-aware completions (configured via Google Cloud Console)
- **Expanded to CLI and agent mode** (November 12, 2025)
- **Customer-managed encryption keys** for indexed code (December 12, 2025)
- **Data residency support** (March 12, 2025)

---

## 14. Deprecations

| Model / Feature | Shutdown Date |
|----------------|---------------|
| `gemini-2.0-flash` variants | June 1, 2026 |
| Gemini 1.5 models (all) | September 29, 2025 |
| `text-embedding-004` | January 14, 2026 |
| `gemini-robotics-er-1.5-preview` | April 30, 2026 |

**Migration path:**
- `gemini-1.5-pro` → `gemini-2.5-pro` (drop-in compatible for most use cases)
- `gemini-2.0-flash` → `gemini-3-flash` or `gemini-2.5-flash`
- `text-embedding-004` → `gemini-embedding-001` or `gemini-embedding-2` (multimodal)

---

## 11. Extensions and Integrations

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

## 12. Grounding and Citations

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

## 13. Safety Settings and Content Filtering

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

## 15. Practical Use Cases for Senior Developers

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

## 16. Try This Exercises

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

### Exercise 6: Canvas for a Technical Design Doc (15 min)
1. Open Gemini → start a Canvas session
2. Ask: "In Canvas, write a technical design document for a cache invalidation strategy using Redis for a multi-service Spring Boot system"
3. Iterate: "Add a section covering the Spring Boot implementation with `@CacheEvict`"
4. Export to Google Docs and share with a teammate

### Exercise 7: Set Up a Scheduled Action (10 min)
1. Open the Gemini app → click the schedule icon
2. Create a weekly action: "Every Monday at 9 AM, research and summarize: top Spring Boot CVEs or updates from last week, and any Kubernetes AKS announcements"
3. Verify the first scheduled run

### Exercise 8: Gemini Code Assist — Finish Changes (15 min)
1. In IntelliJ or VS Code with Gemini Code Assist installed
2. Write a TODO comment in Java: `// TODO: implement rate limiting using token bucket algorithm, max 100 req/sec per user, backed by Redis`
3. Ask Gemini Code Assist: "Finish this"
4. Review the generated implementation
5. Ask it to also generate a test class for the rate limiter

---

## Key Takeaways

1. **Gemini 3.x** is the current model family — Gemini 3.1 Pro is state-of-the-art reasoning as of April 2026
2. **Gems + Shareable Gems** — create team-standard personas, share via Drive-style links
3. **Deep Research** replaces hours of manual research; now upgraded to 3.x Thinking + Canvas output
4. **Canvas** is a collaborative workspace for iterative document/code creation — key for ADRs and design docs
5. **NotebookLM** is groundbreaking for source-grounded analysis — zero hallucination risk; now integrated in Gemini app
6. **Scheduled Actions** automate recurring research and digests — no scripting required
7. **Workspace integration** saves time on documentation, planning, and communication
8. **Gemini Code Assist** — Finish Changes, Agent Mode, and Next Edit Predictions are major productivity gains for IntelliJ/VS Code
9. **Deprecation alert**: Gemini 1.5 models shut down Sep 29, 2025; `gemini-2.0-flash` ends Jun 1, 2026 — migrate now
10. Combined, these features save **5-10 hours per week** for a senior developer

*Last updated: April 2026.*

---

Next: [07 — Daily Workflow Playbook](07-daily-workflow-playbook.md)
