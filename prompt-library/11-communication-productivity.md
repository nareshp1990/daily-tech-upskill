# Communication & Productivity — Daily Prompts

> Tools: Outlook, Microsoft Teams, Confluence, Jira, Copilot, Markdown

---

## Email & Messages

### Write a technical status update email
```
Write a status update email for my manager / stakeholders.

Context:
- Project: [project name]
- Sprint: [sprint number/dates]
- Audience: [engineering manager / VP / cross-team]

Structure:
**Subject:** [Project] Status Update — [date]

**TL;DR** (3 bullet points max):
- What shipped / major milestone
- Key risk or blocker
- What's next

**Completed This Week:**
- [Feature/task] — impact in plain language
- [Feature/task] — metrics if applicable

**In Progress:**
- [Feature/task] — % complete, ETA
- [Feature/task] — dependency on [team/person]

**Blockers / Risks:**
- [Blocker] — impact if unresolved, who can help
- [Risk] — mitigation plan

**Next Week:**
- [Priority 1]
- [Priority 2]

Tone: professional but concise. No filler. Lead with what matters.
Fill this in with: [your details]
```

### Write a technical decision email
```
Write an email communicating a technical decision to stakeholders.

Decision: [what was decided]
Alternatives considered: [options A, B, C]
Chosen: [option] because [reasons]
Impact: [who/what is affected]

Structure:
**Subject:** [Decision]: [topic] — [chosen approach]

Hi [team/name],

**Decision:** We're going with [chosen approach] for [context].

**Why this approach:**
- [Reason 1 — most compelling]
- [Reason 2]
- [Reason 3]

**Alternatives considered:**
| Option | Pros | Cons | Why not |
|--------|------|------|---------|
| A | ... | ... | ... |
| B | ... | ... | ... |

**Impact:**
- [Team/service] will need to [change]
- Timeline: [when]
- Migration: [plan]

**Next steps:**
1. [Action] — [owner] — [date]
2. [Action] — [owner] — [date]

Questions? Let's discuss in [meeting/Slack channel].

Tone: confident, clear, assumes decision is final but open to questions.
```

### Reply to a production incident thread
```
Write a concise update for an ongoing incident thread (Slack / Teams / email).

Incident: [description]
Current status: [investigating / identified / mitigating / resolved]
Audience: engineering + product + leadership

Template by status:

**Investigating:**
🔍 Update: [service] — [symptom]
- Investigating since [time]
- Impact: [users/features affected]
- Team: [who's on it]
- Next update: [time]

**Identified:**
🎯 Root cause identified: [brief description]
- Cause: [what broke]
- Impact: [scope]
- Fix: [what we're doing]
- ETA: [estimated resolution]
- Next update: [time]

**Mitigating:**
🔧 Mitigation in progress
- Applied: [what fix was deployed]
- Status: [partial recovery / monitoring]
- Metrics: [key metric trending in right direction]
- Full resolution ETA: [time]

**Resolved:**
✅ Resolved: [service] back to normal
- Duration: [start] – [end] ([Xh Ym])
- Root cause: [one line]
- Fix: [one line]
- Post-mortem: scheduled for [date]

Fill in with: [your incident details]
```

---

## Meetings & Standups

### Daily standup update
```
Generate a concise standup update.

Yesterday: [what I worked on]
Today: [what I plan to do]
Blockers: [any blockers]

Format options:

**Async (Slack/Teams):**
📅 Standup — [date]
✅ Done: [completed items, one line each]
🔄 Today: [planned items]
🚫 Blocked: [blockers with who can help]

**Verbal (15-second version):**
"Yesterday I [finished/worked on X]. Today I'm [doing Y]. [No blockers / Blocked on Z, need help from person]."

Rules:
- No status padding — only real progress
- Blockers include who you need and what you need from them
- If nothing done, say why (meetings, context-switching, investigation)
```

### Prepare for a technical design review
```
Help me prepare for a technical design review meeting.

Design: [what you're proposing]
Audience: [senior engineers, architect, EM]
Duration: [30/60 min]

**Pre-read document outline:**
1. Problem statement (2-3 sentences)
2. Goals & non-goals
3. Proposed solution (architecture diagram description)
4. Key design decisions (with alternatives considered)
5. Data model changes
6. API changes
7. Migration plan
8. Risks & mitigations
9. Open questions for the group

**Meeting structure:**
- 0-5 min: Problem context (assume they read the doc)
- 5-15 min: Walk through the solution
- 15-25 min: Deep dive on controversial decisions
- 25-30 min: Open questions & action items

**Anticipate questions:**
Based on my design, what will senior engineers likely challenge?
- Performance concerns?
- Backward compatibility?
- Operational complexity?
- Why not simpler alternative?

Prepare 1-sentence answers for each likely challenge.
Fill in with: [your design details]
```

### Sprint retrospective facilitator
```
Help me facilitate a sprint retrospective.

Sprint: [number/dates]
Team size: [N]
Format: [remote/in-person]
Duration: [60 min]

**Agenda:**
- 0-5 min: Set the stage (prime directive reminder)
- 5-15 min: Data gathering (what happened this sprint — metrics, events)
- 15-35 min: Generate insights (what went well, what didn't, why)
- 35-50 min: Decide what to do (pick top 2-3 actions)
- 50-60 min: Close (assign owners, dates, retro feedback)

**Activity options for "Generate Insights":**
1. **Start/Stop/Continue** — classic, good for small teams
2. **4Ls** — Liked, Learned, Lacked, Longed For
3. **Sailboat** — Wind (helps), Anchor (slows), Rocks (risks), Island (goal)
4. **Timeline** — plot events on a mood line, discuss peaks/valleys

**Prompting questions:**
- What surprised you this sprint?
- What took longer than expected and why?
- Where did we waste time? What would you skip next sprint?
- What's one thing you'd change about how we work?
- What should we keep doing that's working?

Generate the Miro/Mural board template description for option [N].
```

---

## Documentation

### Write a technical design document
```
Write a technical design document for: [feature/system]

Template:

# [Feature/System Name] — Technical Design

## 1. Overview
**Problem:** What problem are we solving? (2-3 sentences)
**Goal:** What does success look like?
**Non-goals:** What are we explicitly NOT doing?

## 2. Background
Context needed to understand this design. Prior art, related systems.

## 3. Proposed Solution

### 3.1 Architecture
High-level architecture description (reference diagram).
Components and their responsibilities.

### 3.2 API Design
Endpoints, request/response schemas, error codes.

### 3.3 Data Model
New tables/collections, schema changes, migration plan.

### 3.4 Key Flows
Step-by-step for primary use cases (sequence diagram descriptions).

## 4. Design Decisions

| Decision | Options Considered | Chosen | Rationale |
|----------|-------------------|--------|-----------|
| [decision] | A, B, C | B | [why] |

## 5. Dependencies
External services, libraries, teams.

## 6. Rollout Plan
Feature flags, phased rollout, rollback plan.

## 7. Monitoring & Alerting
New metrics, dashboards, alerts.

## 8. Security Considerations
Auth, data access, PII handling.

## 9. Open Questions
Items needing input from reviewers.

Fill in for: [your feature details]
```

### Write an API documentation page
```
Write API documentation for this endpoint.

Endpoint: [METHOD /path]
Service: [service name]
Auth: [type — Bearer token, API key, etc.]

Structure:

## [METHOD] /api/v1/[resource]

**Description:** What this endpoint does (one sentence).

**Authentication:** Required. Bearer token with scope: [scope]

**Request:**
```json
{
  "field1": "string (required) — description",
  "field2": 123,
  "field3": {
    "nested": "value"
  }
}
```

**Path Parameters:**
| Name | Type | Required | Description |
|------|------|----------|-------------|

**Query Parameters:**
| Name | Type | Default | Description |
|------|------|---------|-------------|

**Response (200):**
```json
{
  "id": "uuid",
  "field1": "value",
  "createdAt": "ISO-8601"
}
```

**Error Responses:**
| Status | Code | Description | Example |
|--------|------|-------------|---------|
| 400 | VALIDATION_ERROR | Invalid input | {"error": "...", "details": [...]} |
| 401 | UNAUTHORIZED | Missing/invalid token | |
| 404 | NOT_FOUND | Resource doesn't exist | |
| 409 | CONFLICT | Duplicate resource | |
| 429 | RATE_LIMITED | Too many requests | Retry-After header |

**Example (cURL):**
```bash
curl -X POST https://api.example.com/api/v1/resource \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"field1": "value"}'
```

**Rate Limits:** 100 req/min per API key

**Notes:**
- Idempotency: use Idempotency-Key header for POST
- Pagination: use cursor-based (next_cursor in response)

Fill in for: [your endpoint details]
```

### Write a runbook / SOP
```
Write a Standard Operating Procedure (SOP) for: [operation]

Examples: database migration, service deployment, secret rotation, scaling event

Structure:

# SOP: [Operation Name]

**Owner:** [team]
**Last updated:** [date]
**Frequency:** [ad-hoc / weekly / per release]
**Estimated time:** [minutes]

## Prerequisites
- [ ] Access to [systems]
- [ ] Permissions: [roles needed]
- [ ] Notifications sent to [team/channel]

## Procedure

### Step 1: [Name]
**Purpose:** Why this step
**Command:**
```bash
[exact command]
```
**Expected output:** [what success looks like]
**If it fails:** [what to do]

### Step 2: [Name]
...

## Verification
- [ ] [Check 1]: [command] → expected result
- [ ] [Check 2]: [dashboard] → metric is normal
- [ ] [Check 3]: [smoke test] → passes

## Rollback
If anything goes wrong:
1. [Rollback step 1]
2. [Rollback step 2]
3. Notify: [who to tell]

## Troubleshooting
| Symptom | Likely Cause | Fix |
|---------|-------------|-----|
| [symptom] | [cause] | [fix] |

Fill in for: [your operation]
```

---

## Jira & Task Management

### Write a well-structured Jira ticket
```
Write a Jira ticket for: [feature/bug/task]

**Bug ticket:**
Summary: [BUG] [Component] — [short description of broken behavior]

Description:
**Environment:** [prod/staging, service version, browser if applicable]

**Steps to Reproduce:**
1. [Step 1]
2. [Step 2]
3. [Step 3]

**Expected Behavior:**
[What should happen]

**Actual Behavior:**
[What actually happens]

**Evidence:**
- Error log: [paste or link]
- Screenshot: [attach]
- Trace ID: [if applicable]

**Impact:**
- Users affected: [count/percentage]
- Workaround: [yes/no — describe if yes]

**Suggested Investigation:**
- Check: [specific area of code]
- Related: [linked tickets]

---

**Feature ticket:**
Summary: [FEATURE] [Component] — [short description]

Description:
**User Story:** As a [role], I want [capability] so that [benefit].

**Acceptance Criteria:**
- [ ] [Criterion 1 — specific and testable]
- [ ] [Criterion 2]
- [ ] [Criterion 3]

**Technical Notes:**
- Affected services: [list]
- API changes: [yes/no — describe]
- Database changes: [yes/no — describe]
- Feature flag: [name]

**Out of Scope:**
- [What this ticket does NOT include]

**Dependencies:**
- Blocked by: [tickets]
- Blocks: [tickets]

Fill in for: [your ticket details]
```

### Break down an epic into stories
```
Break down this epic into implementable user stories.

Epic: [title and description]
Goal: [what success looks like]
Timeline: [target completion]
Team: [N developers]

Guidelines:
1. Each story should be completable in 1-3 days
2. Stories should be independently deployable where possible
3. Include technical stories (infra, migration) alongside feature stories
4. Order by dependency (what must come first)
5. Tag: [API], [UI], [DB], [INFRA], [TEST] for each

Output:

| # | Story | Type | Est | Dependencies | Sprint |
|---|-------|------|-----|-------------|--------|
| 1 | [story title] | [type] | [S/M/L] | none | 1 |
| 2 | [story title] | [type] | [M] | #1 | 1 |
| ... | | | | | |

For the first 3 stories, write full Jira descriptions with acceptance criteria.
```

---

## Presentations & Knowledge Sharing

### Prepare a tech talk outline
```
Create an outline for a tech talk / lunch-and-learn.

Topic: [what you want to present]
Audience: [backend engineers / full team / leadership]
Duration: [15 / 30 / 45 min]
Goal: [inform / persuade / teach]

Structure:

**Title:** [catchy, specific title]

**1. Hook (2 min)**
- Start with a problem, story, or surprising fact
- Why should the audience care?

**2. Context (3 min)**
- Background needed to understand the talk
- Where we were before

**3. Main Content (N min)**
- [Point 1] — concept + example + demo
- [Point 2] — concept + example + demo
- [Point 3] — concept + example + demo
(Rule of 3: max 3 key points for a 30-min talk)

**4. Live Demo / Code Walkthrough (5 min)**
- Show, don't tell
- Have a backup (screenshots) in case demo fails

**5. Lessons Learned (3 min)**
- What surprised us
- What we'd do differently

**6. Q&A (5 min)**
- Prepare 3 questions you expect and answers

**Slides guideline:**
- 1 slide per minute (max)
- More diagrams, less text
- Code snippets: max 15 lines, syntax highlighted, annotated

Fill in for: [your topic]
```

### Explain a technical concept to non-technical stakeholders
```
Explain [technical concept] to a non-technical audience.

Concept: [what you need to explain]
Audience: [product manager / executive / client]
Context: [why they need to understand this]

Guidelines:
1. Start with the "why" — business impact first
2. Use an analogy from everyday life
3. Avoid jargon — if you must use a term, define it in parentheses
4. Focus on trade-offs they can make decisions about
5. End with what you need from them (decision, budget, priority)

Format:
- 3-sentence elevator pitch (for Slack/email)
- 1-paragraph explanation (for a meeting)
- 5-minute walkthrough with a simple diagram description

Example concepts:
- Why we need to migrate the database
- What technical debt means for feature velocity
- Why this integration will take 3 sprints not 1
- How caching works and why it saves money
```

---

## Productivity

### Manage context-switching
```
I'm juggling multiple priorities. Help me organise my day.

Current tasks:
1. [Task + urgency + effort estimate]
2. [Task + urgency + effort estimate]
3. [Task + urgency + effort estimate]
...

Constraints:
- Available hours: [N]
- Meetings at: [times]
- Deadline: [task] by [date]
- Energy: [morning peak / afternoon peak]

Create:
1. Prioritised list (Eisenhower matrix: urgent+important first)
2. Time-blocked schedule fitting around meetings
3. "Do not disturb" blocks for deep work (min 90 min)
4. Quick wins to batch together (< 15 min tasks)
5. What to delegate or defer

Rule: no task-switching within a focus block. Batch Slack/email to 2-3 check windows.
```

### Write a self-review / performance summary
```
Help me write a self-review for my performance cycle.

Period: [dates]
Role: Senior Backend Engineer

Achievements: [list your key work]
Impact: [metrics, outcomes]
Growth: [skills learned, mentoring done]

Structure:

**Impact Summary:**
[2-3 sentences: biggest impact this cycle, in business terms]

**Key Accomplishments:**
1. **[Project/Feature]** — [what you did] → [measurable impact]
   - Technical: [complexity, scale, innovation]
   - Business: [revenue, reliability, velocity improvement]
   - Team: [unblocked others, mentored, improved processes]

2. **[Project/Feature]** — ...

**Technical Growth:**
- [Skill/area] — how you grew and applied it
- [Skill/area] — ...

**Team Contributions:**
- Code reviews: [volume, quality focus]
- Mentoring: [who, what impact]
- Process improvements: [what you improved]
- On-call: [incidents handled, runbooks written]

**Areas for Growth:**
- [Area] — what you plan to do about it
- [Area] — ...

Tone: factual, specific, quantified where possible. No false modesty but no exaggeration.
Fill in with: [your details]
```

---

*Tailored for: Senior Backend Engineer / Java + Spring Boot team / Azure + AKS environment*
