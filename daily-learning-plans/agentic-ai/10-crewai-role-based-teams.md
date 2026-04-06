# Agentic AI — Day 10: CrewAI — Role-Based Agent Teams
**Track:** Agentic AI (Implementation-Focused)
**Time Target:** 1–2 hours

---

## 1. Today's Focus
**CrewAI — High-Level Multi-Agent Framework with Roles, Goals, and Backstories**

LangGraph gives you full control over the graph. CrewAI takes a different approach: you define agents with roles, goals, and backstories, then define tasks and let CrewAI orchestrate the execution. It's higher-level and faster to prototype — think of it as the "Spring Boot" of multi-agent frameworks. Today you build a content creation crew and a code review crew.

---

## 2. CrewAI Core Concepts

```
Crew = a team of agents working together on tasks

Agent:
  - role: "Senior Java Developer"
  - goal: "Write clean, production-ready Java code"
  - backstory: "12 years of experience with Spring Boot..."
  - tools: [read_file, write_file, search_web]

Task:
  - description: "Review the OrderService code for issues"
  - expected_output: "A detailed code review with suggestions"
  - agent: java_developer (assigned specialist)

Crew:
  - agents: [researcher, developer, reviewer]
  - tasks: [research_task, coding_task, review_task]
  - process: sequential | hierarchical
```

---

## 3. Project: Technical Blog Writing Crew

```python
from crewai import Agent, Task, Crew, Process
from crewai_tools import SerperDevTool, WebsiteSearchTool
from langchain_anthropic import ChatAnthropic

llm = ChatAnthropic(model="claude-sonnet-4-20250514")

# --- Define Agents ---
researcher = Agent(
    role="Senior Technical Researcher",
    goal="Find the most accurate and up-to-date information about the given topic",
    backstory="""You are an expert technical researcher with a background in
    computer science. You excel at finding reliable sources, reading documentation,
    and extracting the most relevant information.""",
    tools=[SerperDevTool(), WebsiteSearchTool()],
    llm=llm,
    verbose=True
)

writer = Agent(
    role="Senior Technical Writer",
    goal="Write engaging, accurate technical blog posts that developers love to read",
    backstory="""You are a veteran technical writer who has written for major
    engineering blogs. You know how to explain complex topics simply, use good
    code examples, and structure content for maximum readability. Your target
    audience is senior backend engineers.""",
    llm=llm,
    verbose=True
)

editor = Agent(
    role="Technical Editor",
    goal="Ensure the blog post is accurate, well-structured, and free of errors",
    backstory="""You are a meticulous editor with deep technical knowledge.
    You verify code examples compile, check facts, ensure consistent formatting,
    and improve clarity without changing the author's voice.""",
    llm=llm,
    verbose=True
)

# --- Define Tasks ---
research_task = Task(
    description="""Research the topic: {topic}

    Find:
    1. Key concepts and how they work
    2. Current best practices (2025-2026)
    3. Real-world examples and case studies
    4. Common pitfalls and how to avoid them
    5. Code examples from official documentation""",
    expected_output="A structured research document with sources and key findings",
    agent=researcher
)

writing_task = Task(
    description="""Write a technical blog post based on the research provided.

    Structure:
    - Attention-grabbing introduction (why this matters)
    - Core concepts explained with diagrams (ASCII art)
    - Practical Java/Spring Boot code examples
    - Common mistakes section
    - Key takeaways (5 bullet points)
    - Further reading links

    Target: 1500-2000 words. Tone: professional but approachable.""",
    expected_output="A complete, well-formatted Markdown blog post",
    agent=writer,
    context=[research_task]  # Writer receives researcher's output
)

editing_task = Task(
    description="""Review and edit the blog post:
    1. Verify all code examples are syntactically correct
    2. Check factual accuracy against the research
    3. Improve sentence structure and clarity
    4. Ensure consistent formatting
    5. Add missing links or references
    6. Fix any grammar or spelling issues""",
    expected_output="The final, polished blog post ready for publication",
    agent=editor,
    context=[research_task, writing_task]  # Editor sees both research and draft
)

# --- Create and Run Crew ---
blog_crew = Crew(
    agents=[researcher, writer, editor],
    tasks=[research_task, writing_task, editing_task],
    process=Process.sequential,  # Research → Write → Edit
    verbose=True
)

result = blog_crew.kickoff(inputs={
    "topic": "Java Virtual Threads in Spring Boot 3 — Complete Guide"
})

print(result)
```

---

## 4. Project: Code Review Crew

```python
code_reader = Agent(
    role="Code Analyst",
    goal="Thoroughly read and understand code, identifying patterns and anti-patterns",
    backstory="Senior developer with expertise in Java, Spring Boot, and design patterns.",
    tools=[read_file_tool, search_in_files_tool],
    llm=llm
)

security_reviewer = Agent(
    role="Security Specialist",
    goal="Identify security vulnerabilities and suggest fixes",
    backstory="Application security expert familiar with OWASP Top 10 and Java security.",
    tools=[read_file_tool],
    llm=llm
)

perf_reviewer = Agent(
    role="Performance Engineer",
    goal="Identify performance bottlenecks and optimisation opportunities",
    backstory="Expert in JVM performance, database optimisation, and caching strategies.",
    tools=[read_file_tool],
    llm=llm
)

# Tasks
analyse_task = Task(
    description="Read and analyse {file_path}. Identify: design patterns used, "
                "code structure, potential issues, and readability concerns.",
    expected_output="Code analysis report with findings",
    agent=code_reader
)

security_task = Task(
    description="Review the code for security issues: injection risks, auth flaws, "
                "sensitive data exposure, insecure configurations.",
    expected_output="Security audit report with severity levels",
    agent=security_reviewer,
    context=[analyse_task]
)

perf_task = Task(
    description="Review the code for performance issues: N+1 queries, missing indexes, "
                "unnecessary object creation, inefficient algorithms.",
    expected_output="Performance review with optimisation suggestions",
    agent=perf_reviewer,
    context=[analyse_task]
)

review_crew = Crew(
    agents=[code_reader, security_reviewer, perf_reviewer],
    tasks=[analyse_task, security_task, perf_task],
    process=Process.sequential
)

result = review_crew.kickoff(inputs={"file_path": "src/main/java/com/example/OrderService.java"})
```

---

## 5. Hierarchical Process (Manager Mode)

```python
# CrewAI can use a "manager" that dynamically routes tasks
manager_crew = Crew(
    agents=[researcher, writer, editor, code_reader],
    tasks=[complex_task],
    process=Process.hierarchical,
    manager_llm=ChatAnthropic(model="claude-sonnet-4-20250514"),
    # The manager LLM decides which agents work on what
)
```

---

## 6. CrewAI vs LangGraph — When to Use Which

| Feature | CrewAI | LangGraph |
|---------|--------|-----------|
| **Setup time** | Minutes | Hours |
| **Customisation** | Limited (role/goal/backstory) | Full control over every edge |
| **Multi-agent** | Built-in (sequential, hierarchical) | Manual (build your own) |
| **Human-in-loop** | Basic | Advanced (interrupt/resume) |
| **Debugging** | Verbose logging | Graph visualisation + state inspection |
| **Production readiness** | Good for prototypes | Production-grade |
| **Best for** | Quick prototyping, clear role-based teams | Complex custom workflows |

**Recommendation:** Start with CrewAI for quick prototypes. Move to LangGraph when you need custom control flow, parallel execution, or human-in-the-loop.

---

## 7. Hands-On Exercise

1. Install CrewAI: `pip install crewai crewai-tools`
2. Build the **blog writing crew** and generate a post about a topic you're learning.
3. Build the **code review crew** and review one of your existing Java files.
4. Experiment with `Process.hierarchical` vs `Process.sequential` — observe the difference.
5. Compare: the same task done by CrewAI crew vs single LangGraph agent — which produces better results?

---

## 8. Key Takeaways

1. **CrewAI's role/goal/backstory pattern is powerful** — it creates focused, specialised agents with minimal code.
2. **Context chaining lets later agents build on earlier work** — writer sees researcher's output automatically.
3. **Sequential process for predictable workflows, hierarchical for dynamic routing.**
4. **CrewAI is the fastest path from idea to multi-agent prototype** — but LangGraph gives more control.

---

*Day 10. The right abstraction level matters — sometimes you need a StateGraph, sometimes you need a Crew.*
