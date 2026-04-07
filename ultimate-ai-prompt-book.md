# The Ultimate AI Prompt Book

A comprehensive collection of **500+ ready-to-use AI prompts** organized by category — tailored for software engineers, developers, and tech professionals.

> **How to use:** Copy any prompt, replace the `[bracketed placeholders]` with your specifics, and paste into ChatGPT, Claude, Gemini, or any LLM.

---

## Table of Contents

1. [Software Development & Coding](#1-software-development--coding)
2. [System Design & Architecture](#2-system-design--architecture)
3. [Java & Spring Boot](#3-java--spring-boot)
4. [Database & SQL](#4-database--sql)
5. [DevOps, Docker & Kubernetes](#5-devops-docker--kubernetes)
6. [API Design & Integration](#6-api-design--integration)
7. [Debugging & Troubleshooting](#7-debugging--troubleshooting)
8. [Code Review & Refactoring](#8-code-review--refactoring)
9. [Testing & QA](#9-testing--qa)
10. [Cloud & Infrastructure (Azure/AWS/GCP)](#10-cloud--infrastructure)
11. [AI & LLM Development](#11-ai--llm-development)
12. [Prompt Engineering](#12-prompt-engineering)
13. [Security & DevSecOps](#13-security--devsecops)
14. [Technical Writing & Documentation](#14-technical-writing--documentation)
15. [Learning & Upskilling](#15-learning--upskilling)
16. [Career & Interview Prep](#16-career--interview-prep)
17. [Project Management & Agile](#17-project-management--agile)
18. [Marketing & Content Creation](#18-marketing--content-creation)
19. [SEO & Blogging](#19-seo--blogging)
20. [Business & Entrepreneurship](#20-business--entrepreneurship)
21. [Email & Communication](#21-email--communication)
22. [Data Analysis & Visualization](#22-data-analysis--visualization)
23. [Creative Writing & Storytelling](#23-creative-writing--storytelling)
24. [Education & Teaching](#24-education--teaching)
25. [Productivity & Personal Growth](#25-productivity--personal-growth)

---

## 1. Software Development & Coding

### Code Generation

```
Write a [language] function that [describe functionality]. Include input validation,
error handling, and inline comments explaining the logic. Use [specific patterns/style].
```

```
Generate a [language] class that implements [pattern/interface] for [use case].
Follow SOLID principles and include unit tests.
```

```
Create a complete CRUD module in [language/framework] for managing [entity].
Include: entity class, repository, service layer, controller, DTOs, validation,
exception handling, and pagination support.
```

```
Write a [language] implementation of [algorithm/data structure] optimized for
[specific constraint: memory, speed, concurrency]. Include time/space complexity
analysis in comments.
```

```
Convert this [source language] code to [target language], preserving the same
logic and handling language-specific idioms appropriately:
[paste code]
```

### Code Explanation

```
Explain this code line by line as if I'm a [junior/senior] developer.
Focus on the why, not just the what:
[paste code]
```

```
What design patterns are used in this code? Explain each pattern's role and
whether it's the right choice here:
[paste code]
```

```
Explain the time and space complexity of this code. Suggest optimizations:
[paste code]
```

### Clean Code

```
Rewrite this code following clean code principles (meaningful names, small functions,
single responsibility, no magic numbers). Explain each change:
[paste code]
```

```
Identify code smells in this code and suggest refactoring for each:
[paste code]
```

```
Apply the [SOLID/DRY/KISS/YAGNI] principle to improve this code. Show before/after:
[paste code]
```

---

## 2. System Design & Architecture

### High-Level Design

```
Design a [system name, e.g., URL shortener / chat app / payment system] that handles
[scale: e.g., 10M DAU, 1000 RPS]. Cover:
1. Functional & non-functional requirements
2. API design
3. Data model & database choice
4. High-level architecture diagram (describe components)
5. Deep dive into [specific component]
6. Scalability, reliability & monitoring strategy
Assume I'm in a system design interview — structure your answer accordingly.
```

```
Compare monolithic vs microservices architecture for [use case]. Include:
- When to choose each
- Migration strategy from monolith to microservices
- Trade-offs (complexity, latency, data consistency, deployment)
```

```
Design the database schema for [application]. Include:
- Entity-relationship diagram (describe tables and relationships)
- Indexing strategy
- Partitioning/sharding approach for [expected data volume]
- Read/write optimization trade-offs
```

### Scalability & Performance

```
How would you scale [system/service] from 1K to 10M users? Walk through each stage:
- Single server → Load balancer → Database replication → Caching → CDN → Sharding
Include specific technology choices and trade-offs at each stage.
```

```
Design a caching strategy for [application/use case]. Cover:
- Cache-aside vs write-through vs write-behind
- Cache invalidation approach
- Handling cache stampede
- Redis vs Memcached decision
- TTL strategy
```

```
Design a rate limiting system for [API/service]. Cover:
- Algorithm choice (token bucket, sliding window, fixed window)
- Distributed rate limiting across multiple servers
- Configuration per client/tier
- Handling burst traffic
```

### Trade-off Discussions

```
Compare [Technology A] vs [Technology B] for [use case]. Create a comparison table
covering: performance, scalability, learning curve, ecosystem, cost, and community
support. Give a final recommendation with reasoning.
```

```
What are the trade-offs between [approach A] and [approach B] for [problem]?
Structure as: Pros/Cons table, decision criteria, when to pick each, real-world
examples of companies using each approach.
```

---

## 3. Java & Spring Boot

### Spring Boot Development

```
Create a production-ready Spring Boot REST API for [entity] management using:
- Java 17+, Spring Boot 3.x
- Layered architecture (Controller → Service → Repository)
- JPA entity with proper annotations
- DTOs with Jakarta validation
- Global exception handler (@ControllerAdvice)
- Pagination and sorting
- Proper HTTP status codes
Include all files with package structure.
```

```
Implement [feature] in Spring Boot with:
- @Configuration class with @Bean definitions
- application.yml configuration
- Proper use of @Value or @ConfigurationProperties
- Profile-specific config (dev, staging, prod)
```

```
Write a Spring Boot integration test for [controller/service] using:
- @SpringBootTest or @WebMvcTest
- MockMvc for controller tests
- @MockBean for dependencies
- Testcontainers for database
- Assertions covering happy path and error scenarios
```

### Spring Cloud & Microservices

```
Implement service-to-service communication in Spring Boot microservices using:
- OpenFeign client with error handling
- Circuit breaker with Resilience4j (fallback, retry, bulkhead)
- Proper timeout configuration
- Distributed tracing with Micrometer + OpenTelemetry
Show complete code with configuration.
```

```
Set up an API Gateway using Spring Cloud Gateway for [microservices]. Include:
- Route configuration for each service
- Rate limiting filter
- Authentication filter (JWT validation)
- Load balancing
- CORS configuration
```

```
Implement the Saga pattern for [distributed transaction use case] in Spring Boot.
Cover: orchestration vs choreography approach, compensating transactions,
error handling, and idempotency.
```

### Kafka with Spring Boot

```
Implement a Kafka producer and consumer in Spring Boot for [use case]. Include:
- KafkaTemplate configuration
- Consumer with @KafkaListener
- Error handling and dead letter topic
- Retry configuration
- Serialization/deserialization (Avro/JSON)
- Idempotent consumer pattern
```

### Performance & Optimization

```
Review this Spring Boot application for performance issues. Check for:
- N+1 query problems
- Missing indexes
- Connection pool sizing
- Lazy vs eager loading
- Caching opportunities
- Thread pool configuration
[paste code or describe the application]
```

---

## 4. Database & SQL

### Query Writing & Optimization

```
Write an optimized SQL query to [describe requirement]. The tables are:
[describe schema]. Consider: indexes, joins, subqueries vs CTEs,
and explain the execution plan.
```

```
Optimize this slow SQL query. Analyze the execution plan, suggest indexes,
and rewrite if necessary. Current execution time: [X seconds], target: [Y ms]:
[paste query]
```

```
Write a database migration script to [add column / create table / modify constraint]
for [database: MySQL/PostgreSQL]. Include: rollback script, zero-downtime
considerations, and data backfill if needed.
```

### Schema Design

```
Design a database schema for [application] with [estimated scale].
Include: tables, relationships, indexes, constraints, and justify your choices
between normalization vs denormalization for each decision.
```

```
How should I partition/shard the [table name] table that currently has [X million]
rows and grows by [Y rows/day]? Consider: partition key selection, range vs hash
partitioning, query patterns, and migration strategy.
```

### Redis & NoSQL

```
Design a Redis data model for [use case: session store / leaderboard / cache /
rate limiter]. Include: key naming convention, data structures (String, Hash,
Sorted Set, etc.), TTL strategy, and memory estimation.
```

```
Compare SQL vs NoSQL for [use case]. Cover: data model, query patterns,
scalability, consistency requirements, and migration considerations.
When would you use MongoDB/DynamoDB/Cassandra vs PostgreSQL/MySQL?
```

---

## 5. DevOps, Docker & Kubernetes

### Docker

```
Write a production-optimized multi-stage Dockerfile for a [Java/Spring Boot/Node.js]
application. Include: minimal base image, layer caching, non-root user, health check,
security scanning, and .dockerignore file.
```

```
Create a docker-compose.yml for a development environment with:
[list services: app, database, Redis, Kafka, etc.]. Include: networking,
volumes, environment variables, health checks, and dependency ordering.
```

### Kubernetes

```
Write Kubernetes manifests (YAML) to deploy [application] with:
- Deployment (replicas, resource limits, readiness/liveness probes)
- Service (ClusterIP/LoadBalancer)
- ConfigMap and Secret
- HPA (Horizontal Pod Autoscaler)
- Ingress with TLS
Include best practices for production.
```

```
Debug this Kubernetes issue: [describe: pod CrashLoopBackOff / OOMKilled /
ImagePullBackOff / pending state]. Walk me through the diagnostic steps
(kubectl commands) and likely root causes.
```

```
Design a Kubernetes deployment strategy for [application] with zero downtime.
Compare: RollingUpdate, Blue-Green, and Canary. Include YAML configuration
for the recommended approach.
```

### CI/CD & GitHub Actions

```
Create a GitHub Actions CI/CD pipeline for a [Java/Spring Boot] application that:
1. Runs on push to main and PR creation
2. Builds with Maven/Gradle
3. Runs unit and integration tests
4. Performs SonarQube/code quality analysis
5. Builds Docker image and pushes to [registry]
6. Deploys to [environment] using [strategy]
Include caching, parallelization, and secret management.
```

### Terraform & IaC

```
Write Terraform configuration to provision [resource: AKS cluster / Azure App Service /
RDS database / VPC] on [cloud provider]. Include: variables, outputs, state management,
and modular structure. Follow infrastructure-as-code best practices.
```

---

## 6. API Design & Integration

```
Design a RESTful API for [resource/domain]. Include:
- Endpoints with proper HTTP methods and status codes
- Request/response JSON schemas
- Pagination, filtering, and sorting
- Error response format (RFC 7807 Problem Details)
- Versioning strategy
- Rate limiting headers
- OpenAPI/Swagger specification
```

```
Compare REST vs GraphQL vs gRPC for [use case]. When would I choose each?
Include: performance characteristics, tooling, learning curve, and real-world
examples of companies using each.
```

```
Design an idempotent API for [operation: payment / order creation]. Cover:
idempotency key implementation, database constraints, retry handling,
and distributed system considerations.
```

```
Write an OpenAPI 3.0 specification for [API]. Include: schemas, security
definitions (JWT/OAuth2), examples, and generate server/client stubs.
```

---

## 7. Debugging & Troubleshooting

```
Debug this error: [paste error message/stack trace]. I'm using [tech stack].
Walk me through:
1. What this error means
2. Most likely root causes (ranked by probability)
3. Diagnostic steps to identify the exact cause
4. The fix with code
5. How to prevent this in the future
```

```
My [application/service] is running slowly. Response time went from [X]ms to [Y]ms.
Tech stack: [list]. Walk me through a systematic performance investigation:
- What metrics to check first
- Common bottlenecks for this stack
- Profiling approach
- Quick wins vs long-term fixes
```

```
This code works locally but fails in production with [error/behavior].
Environment: [local setup] vs [production setup]. Help me identify
environment-specific issues:
[paste code and error]
```

```
Analyze this log output and identify the root cause of the issue.
Suggest fixes and monitoring improvements:
[paste logs]
```

---

## 8. Code Review & Refactoring

```
Review this code as a senior engineer. Check for:
- Bugs and edge cases
- Security vulnerabilities (OWASP Top 10)
- Performance issues
- Clean code violations
- Missing error handling
- Thread safety issues
Rate severity (Critical/Major/Minor) for each finding:
[paste code]
```

```
Refactor this [monolithic class / long method / complex function] into a cleaner
design. Apply appropriate design patterns. Show the refactored code with
explanation of each change:
[paste code]
```

```
This legacy code needs modernization from [old approach] to [new approach].
Create a step-by-step migration plan that maintains backward compatibility:
[paste code]
```

---

## 9. Testing & QA

```
Write comprehensive unit tests for this [class/function] using [JUnit 5 / Mockito /
pytest / Jest]. Cover:
- Happy path
- Edge cases (null, empty, boundary values)
- Error scenarios
- Parameterized tests for multiple inputs
Aim for >90% branch coverage:
[paste code]
```

```
Write integration tests for this [REST controller / Kafka consumer / database
repository] using [Testcontainers / MockMvc / WireMock]. Include setup,
teardown, and assertions.
```

```
Generate test cases (BDD format: Given-When-Then) for [feature/user story].
Cover: positive scenarios, negative scenarios, boundary conditions,
and concurrency edge cases.
```

```
Design a load testing scenario for [API/system] using [JMeter / Gatling / k6].
Target: [X RPS, Y concurrent users]. Include: ramp-up strategy, assertions,
and metrics to monitor.
```

---

## 10. Cloud & Infrastructure

### Azure

```
Design an Azure architecture for [application type] with:
- Compute: App Service vs AKS vs Container Apps
- Database: Azure SQL vs Cosmos DB vs PostgreSQL Flexible Server
- Messaging: Service Bus vs Event Hubs vs Event Grid
- Caching: Azure Cache for Redis
- Storage: Blob Storage tiers
Include: networking (VNet, Private Endpoints), security (Managed Identity,
Key Vault), monitoring (Application Insights), and cost estimation.
```

### AWS

```
Design an AWS architecture for [application]. Compare and recommend:
- ECS vs EKS vs Lambda for compute
- RDS vs DynamoDB vs Aurora for database
- SQS vs SNS vs EventBridge for messaging
Include: VPC design, IAM best practices, and cost optimization.
```

### General Cloud

```
Create a disaster recovery plan for [application] on [cloud]. Cover:
- RPO and RTO targets
- Backup strategy (automated, cross-region)
- Failover mechanism
- Data replication approach
- Runbook for DR activation
```

```
Estimate the monthly cloud cost for running [application] with [specs:
users, requests, storage, compute]. Compare across Azure/AWS/GCP.
Suggest cost optimization strategies.
```

---

## 11. AI & LLM Development

### Building AI Applications

```
Design a RAG (Retrieval-Augmented Generation) pipeline for [use case] using
[Spring AI / LangChain / LlamaIndex]. Include:
- Document ingestion and chunking strategy
- Embedding model selection
- Vector store setup (pgvector / Pinecone / Chroma)
- Retrieval with metadata filtering
- Prompt template with context injection
- Evaluation metrics (faithfulness, relevance)
```

```
Build an AI agent using [Spring AI / LangGraph / CrewAI] that can:
[describe capabilities]. Include:
- Tool definitions with @Tool annotations
- ReAct loop implementation
- Memory management (short-term and long-term)
- Error handling and max-iteration guards
- Cost tracking and rate limiting
```

```
Implement an MCP (Model Context Protocol) server in [language] that exposes
[tools/resources] to AI agents. Include: tool definitions, input validation,
error handling, and integration with Claude Code.
```

### LLM Integration

```
Compare [GPT-4 vs Claude vs Gemini vs open-source] for [use case].
Create a decision matrix covering: quality, cost per token, latency,
context window, tool use capability, and privacy considerations.
```

```
Design a production LLM architecture with:
- Model routing (simple queries → small model, complex → large model)
- Semantic caching to reduce costs
- Fallback chain (primary → secondary → local model)
- Token budget management
- Observability (traces, token usage, latency)
```

---

## 12. Prompt Engineering

### Prompt Techniques

```
You are an expert prompt engineer. Rewrite this prompt using advanced techniques
to get better results. Apply: role assignment, chain-of-thought, few-shot examples,
output format specification, and constraints:
Original prompt: [paste prompt]
```

```
Create a system prompt for an AI assistant that [describe role/behavior].
The assistant should: [list capabilities], avoid: [list restrictions],
output format: [specify], tone: [describe].
```

```
Generate 5 different prompt variations for [task] using these techniques:
1. Zero-shot
2. Few-shot (with examples)
3. Chain-of-thought
4. Role-based
5. Step-by-step decomposition
Compare which would give the best results and why.
```

### Prompt Templates

```
Create a reusable prompt template for [recurring task] that includes:
- Variables/placeholders for customization
- System instructions
- Output format specification
- Quality criteria
- Examples of good vs bad output
```

---

## 13. Security & DevSecOps

```
Perform a security audit on this code. Check for OWASP Top 10 vulnerabilities:
- SQL Injection
- XSS (Cross-Site Scripting)
- CSRF
- Broken Authentication
- Sensitive Data Exposure
- Insecure Deserialization
- Security Misconfiguration
Rate each finding by severity and provide fixes:
[paste code]
```

```
Design an authentication and authorization system for [application] using:
- JWT vs OAuth 2.0 vs Session-based (recommend and justify)
- Role-based access control (RBAC)
- Token refresh strategy
- Secure storage of secrets
Include Spring Security configuration.
```

```
Create a security checklist for deploying [application] to production. Cover:
- HTTPS/TLS configuration
- CORS policy
- Rate limiting
- Input validation
- Dependency vulnerability scanning
- Secret management
- Logging (without sensitive data)
- Container security
```

```
Explain and demonstrate how [attack type: SQL injection / XSS / CSRF / SSRF]
works against [type of application]. Then show the defense implementation
in [language/framework].
```

---

## 14. Technical Writing & Documentation

```
Write a README.md for [project] that includes:
- Project overview (what and why)
- Architecture diagram description
- Tech stack
- Prerequisites and setup instructions
- API documentation summary
- Environment variables
- Contributing guidelines
- License
```

```
Write an Architecture Decision Record (ADR) for choosing [technology/approach]
over [alternative] for [use case]. Follow the format:
Title, Status, Context, Decision, Consequences (positive and negative).
```

```
Create API documentation for [endpoint] in OpenAPI 3.0 format.
Include: description, parameters, request body, response schemas,
error codes, and curl examples.
```

```
Write a post-mortem / incident report for: [describe incident].
Structure: Summary, Impact, Timeline, Root Cause, Resolution,
Action Items, Lessons Learned.
```

```
Write a technical blog post about [topic] for a developer audience.
Include: introduction with hook, problem statement, solution walkthrough
with code snippets, diagrams, trade-offs discussed, and conclusion.
Target length: [X] words.
```

---

## 15. Learning & Upskilling

```
Create a [X]-day learning plan for [technology/topic] for a [experience level]
developer. I have [Y] hours per day. Include:
- Daily topics with learning objectives
- Recommended resources (docs, videos, articles)
- Hands-on exercises for each day
- A capstone project at the end
- Self-assessment checklist
My background: [describe your skills]
```

```
Explain [complex concept] to me using analogies from [domain I know well].
Then progressively increase the technical depth across 3 levels:
1. Beginner (analogy-based)
2. Intermediate (technical with examples)
3. Advanced (internals and edge cases)
```

```
I'm a [your role] and want to learn [new technology]. Create a comparison
mapping between [what I already know] and [what I want to learn].
Show me the equivalent concepts, patterns, and tools.
```

```
What are the most important concepts a [role] should know about [topic]
in 2026? Rank by importance and explain why each matters.
Skip the basics I'd already know as a [experience level] professional.
```

---

## 16. Career & Interview Prep

### System Design Interview

```
Mock system design interview: Ask me to design [system]. After I respond,
critique my answer as a senior interviewer. Point out what I missed,
what was strong, and how to improve. Score me on: requirements gathering,
high-level design, deep dive, scalability, and trade-off discussion.
```

```
Give me the top 20 system design questions asked at [FAANG / top startups].
For each, list: key components to cover, common pitfalls, and one
non-obvious insight that impresses interviewers.
```

### Coding Interview

```
Give me [N] [difficulty] [topic: arrays/trees/graphs/DP] problems to practice.
For each: problem statement, hints (progressive), optimal solution with
explanation, time/space complexity, and follow-up variations.
```

```
Review my solution to this coding problem. Analyze: correctness, edge cases
I missed, time/space complexity, code readability, and whether there's
a better approach:
Problem: [paste problem]
My solution: [paste code]
```

### Behavioral Interview

```
Prepare me for behavioral interviews at [company]. Generate 10 common
questions and help me structure STAR-format answers based on my experience as
a [role] who has [key achievements/projects].
```

```
I'm preparing for a [role] interview at [company]. Based on the job description
below, identify the top 10 skills they're looking for and suggest how to
demonstrate each with my background in [your experience]:
[paste job description]
```

---

## 17. Project Management & Agile

```
Break down this feature into user stories with acceptance criteria (Given-When-Then):
[describe feature]. Include: story points estimate, dependencies, and suggested
sprint allocation.
```

```
Write a technical RFC/proposal for [project/feature]. Include:
- Problem statement
- Proposed solution
- Alternatives considered
- Technical design
- Migration plan
- Rollback plan
- Timeline and milestones
- Open questions
```

```
Create a sprint retrospective summary from these notes: [paste notes].
Structure as: What went well, What didn't go well, Action items with owners
and deadlines.
```

```
Estimate the effort for [project/feature] using [T-shirt sizing / story points /
time-based]. Break into tasks, identify risks, and suggest a timeline.
Assume a team of [N] developers.
```

---

## 18. Marketing & Content Creation

```
Write [N] social media posts for [platform: LinkedIn/Twitter/Instagram] about
[topic/product]. Tone: [professional/casual/humorous]. Include: hooks,
CTAs, and relevant hashtags. Vary the formats (text, carousel ideas, poll).
```

```
Create an email marketing sequence ([N] emails) for [product/service].
Target audience: [describe]. Goals: awareness → consideration → conversion.
Include: subject lines (A/B variants), preview text, body copy, and CTAs.
```

```
Write a product launch announcement for [product/feature]. Create versions for:
1. Press release (formal)
2. Blog post (technical audience)
3. Social media thread (casual)
4. Email to existing customers
5. Internal Slack message to the team
```

```
Generate [N] headline variations for [article/landing page/ad] using:
- Numbers/lists approach
- Question-based approach
- How-to approach
- Fear of missing out (FOMO)
- Curiosity gap
A/B test recommendation included.
```

---

## 19. SEO & Blogging

```
Create an SEO-optimized blog post outline for "[keyword/topic]". Include:
- Title (60 chars max) with primary keyword
- Meta description (155 chars max)
- H2/H3 heading structure with keywords
- Internal/external linking suggestions
- Featured snippet optimization
- FAQ section (People Also Ask)
Target word count: [X].
```

```
Research long-tail keywords for [niche/topic]. Generate 20 keyword ideas with:
estimated search volume (low/medium/high), competition level, search intent
(informational/transactional/navigational), and suggested content format.
```

```
Rewrite this blog post for better SEO without sacrificing readability:
[paste content]. Optimize for keyword: [target keyword].
```

```
Create a content calendar for [blog/company] for [month]. [N] posts covering
[topics]. Include: publish dates, titles, target keywords, content type
(how-to, listicle, case study, tutorial), and promotion channels.
```

---

## 20. Business & Entrepreneurship

```
Write a business plan outline for [business idea]. Include:
- Executive summary
- Problem & solution
- Target market & TAM/SAM/SOM
- Business model & revenue streams
- Go-to-market strategy
- Competitive analysis
- Financial projections (3-year)
- Team requirements
```

```
Create a lean canvas for [startup idea]. Fill in all 9 blocks with specific,
actionable content. Then identify the riskiest assumption and suggest how
to validate it with a $[X] budget in [Y] weeks.
```

```
Analyze [industry/market] for startup opportunities in 2026. Identify:
- Emerging trends
- Underserved customer segments
- Technology enablers
- Business model innovations
- Potential moats/defensibility
```

```
Write a pitch deck script (10 slides) for [startup]. Include:
Problem, Solution, Market Size, Product Demo, Business Model,
Traction, Team, Competition, Financials, Ask.
```

---

## 21. Email & Communication

```
Write a professional email to [recipient/role] about [topic].
Tone: [formal/friendly/urgent]. Include: clear subject line,
context, specific ask, and next steps. Keep it under [N] sentences.
```

```
Rewrite this email to be more [concise/persuasive/professional/friendly]:
[paste email]
```

```
Draft a [type: follow-up / introduction / negotiation / feedback / escalation]
email for this situation: [describe context]. Include subject line and
2 tone variants.
```

```
Write a Slack message to [audience: team / manager / cross-functional stakeholders]
about [topic: project update / blocking issue / decision needed].
Be concise, use bullet points, and include clear action items.
```

---

## 22. Data Analysis & Visualization

```
Analyze this dataset and provide insights:
[paste data or describe the dataset]
Cover: trends, outliers, correlations, and actionable recommendations.
Suggest appropriate chart types for visualizing each insight.
```

```
Write a [Python/SQL] script to:
1. Load data from [source]
2. Clean and transform (handle nulls, duplicates, normalize)
3. Perform [specific analysis]
4. Generate visualizations using [matplotlib/seaborn/plotly]
5. Export results to [format]
```

```
Create a dashboard specification for [use case]. Define:
- KPIs and metrics with formulas
- Chart types for each metric
- Filter/drill-down capabilities
- Refresh frequency
- Alert thresholds
```

```
Explain this data in plain English for a non-technical stakeholder:
[paste data/chart description]. Create a 3-bullet executive summary
and a 1-paragraph detailed explanation.
```

---

## 23. Creative Writing & Storytelling

```
Write a [type: short story / poem / script / monologue] about [topic/theme].
Style: [author/genre reference]. Tone: [describe]. Length: [word count].
Include: [specific elements: dialogue, twist, metaphor, etc.].
```

```
Create a compelling narrative for [presentation / pitch / case study] about
[topic]. Use the [Hero's Journey / Problem-Solution-Result / Before-After-Bridge]
framework. Target audience: [describe].
```

```
Generate [N] creative concepts for [project: ad campaign / video series /
brand story / product name]. For each concept: title, one-line pitch,
target emotion, and execution approach.
```

```
Rewrite this technical content as an engaging story that a non-technical
audience would enjoy reading: [paste content]
```

---

## 24. Education & Teaching

```
Explain [complex topic] using the Feynman technique — as if teaching a
12-year-old. Use everyday analogies, avoid jargon, and build understanding
progressively. Then add a "going deeper" section for advanced learners.
```

```
Create a quiz with [N] questions on [topic] for [audience level]. Mix formats:
multiple choice, true/false, fill-in-the-blank, and short answer.
Include answer key with explanations for wrong answers.
```

```
Design a hands-on workshop/tutorial for [topic]. Duration: [X hours].
Include: learning objectives, prerequisites, step-by-step exercises,
common mistakes to watch for, and take-home challenge.
```

```
Create flashcards (Q&A format) for [N] key concepts in [topic].
Front: question or term. Back: concise answer with a memorable hook
or mnemonic. Group by subtopic.
```

---

## 25. Productivity & Personal Growth

```
Create a daily routine optimized for a [role] who wants to [goals].
Constraints: [work hours, commute, family time, exercise]. Include:
deep work blocks, learning time, breaks, and evening wind-down.
```

```
I'm feeling overwhelmed with [list of tasks/projects]. Help me:
1. Categorize using Eisenhower Matrix (urgent/important)
2. Identify what to delegate or drop
3. Create a prioritized action plan for this week
4. Suggest systems to prevent this in the future
```

```
Design a [30/60/90]-day plan for me as a new [role] at [type of company].
Include: learning milestones, relationship building, quick wins,
and long-term goals. My background: [describe].
```

```
I want to build the habit of [habit]. Create a plan using:
- Atomic Habits framework (cue, craving, response, reward)
- Implementation intention ("I will [behavior] at [time] in [location]")
- Habit stacking (attach to existing habit)
- Progress tracking method
- Recovery plan for missed days
```

---

## Bonus: Meta-Prompts (Prompts That Generate Better Prompts)

```
I want to achieve [goal] using AI. I'm not sure how to prompt for it.
Ask me 5 clarifying questions, then generate 3 prompt variations
ranked by likely effectiveness. Explain why each would work.
```

```
Act as a prompt engineering expert. Analyze this prompt and improve it:
[paste prompt]. Score the original (1-10) on: clarity, specificity,
context, output format, and constraint definition. Then provide the
improved version with explanations.
```

```
Create a reusable prompt library for my daily work as a [role].
Generate 10 prompts I'd use most frequently, each with:
- Template with placeholders
- Example filled-in version
- Expected output quality tips
```

---

## Prompt Writing Cheat Sheet

| Element | What It Does | Example |
|---------|-------------|---------|
| **Role** | Sets expertise context | "You are a senior Java architect..." |
| **Task** | Defines what to do | "Design a caching layer for..." |
| **Context** | Provides background | "We have 10M users, 5K RPS..." |
| **Constraints** | Limits the scope | "Use only Spring Boot 3.x, Java 17..." |
| **Format** | Specifies output structure | "Present as a comparison table..." |
| **Examples** | Shows expected quality | "Here's a sample: ..." |
| **Chain-of-Thought** | Forces reasoning | "Think step by step before answering..." |
| **Iteration** | Enables refinement | "First outline, then detail, then review..." |

### The 5-Part Prompt Formula

```
[ROLE] You are a [expert role] with expertise in [domain].
[CONTEXT] I'm working on [project] using [tech stack]. The challenge is [problem].
[TASK] Please [specific action] that [success criteria].
[CONSTRAINTS] Must follow [standards/limits]. Do not [exclusions].
[FORMAT] Structure your response as [format]. Include [specific elements].
```

---

## Sources & Further Reading

- [500+ Best Prompts for ChatGPT — God of Prompt](https://www.godofprompt.ai/blog/500-best-prompts-for-chatgpt-2024)
- [50 ChatGPT Prompts Every Developer Should Know](https://priygop.com/blog/50-chatgpt-prompts-every-developer-should-know)
- [Top 50 Best AI Coding Prompts for Developers 2026](https://www.logicweb.com/top-50-best-ai-coding-prompts-for-developers/)
- [ChatGPT Prompt Engineering for Developers — Strapi](https://strapi.io/blog/ChatGPT-Prompt-Engineering-for-Developers)
- [AI Prompt Starter Pack for Java & Spring Boot Developers — Medium](https://medium.com/@worksystemsai/ai-prompt-starter-pack-for-java-spring-boot-developers-stop-wasting-time-on-generic-ai-responses-4da5d048992d)
- [Spring AI Prompt Engineering Patterns — Spring.io](https://spring.io/blog/2025/04/14/spring-ai-prompt-engineering-patterns/)
- [30 ChatGPT Prompts for Software Engineers — KMS](https://kms-technology.com/blog/30-best-chatgpt-prompts-for-software-engineers/)
- [ChatGPT Prompts for Software Developers — GeeksforGeeks](https://www.geeksforgeeks.org/blogs/chatgpt-prompts-for-software-developers/)
- [The Ultimate Prompt Pack — There's An AI For That](https://theresanaiforthat.com/ebook/)

---

*Last updated: 2026-04-07. 500+ prompts across 25 categories. Copy, customize, and conquer.*
