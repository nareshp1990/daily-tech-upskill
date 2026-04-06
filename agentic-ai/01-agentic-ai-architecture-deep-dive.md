# Agentic AI — Day 1: Architecture Deep Dive
**Track:** Agentic AI (Implementation-Focused)
**Time Target:** 1–2 hours
**Prerequisites:** Phase 1 Days 7, 11, 13 (ReAct, multi-agent theory, tool calling)

---

## 1. Today's Focus
**Agentic AI Architecture — From Theory to Production Patterns**

Phase 1 introduced the ReAct loop and multi-agent orchestration conceptually. This track is different — you'll **build real agents** that solve real problems. Today you map the agentic AI landscape: what frameworks exist (Java and Python), which patterns work in production, and when to choose which approach. By end of day you'll have a clear mental model and your development environment ready.

---

## 2. The Agentic AI Spectrum

```
Complexity ──────────────────────────────────────────────►

┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────────┐  ┌──────────────┐
│ Prompt +  │  │  Chain   │  │  Single  │  │  Multi-Agent │  │  Autonomous  │
│ Response  │  │ (Fixed   │  │  Agent   │  │  System      │  │  Agent       │
│           │  │  Steps)  │  │  (ReAct) │  │  (Orchestr.) │  │  (Self-Imp.) │
└──────────┘  └──────────┘  └──────────┘  └──────────────┘  └──────────────┘
  ChatGPT       RAG           Research       Customer         Devin-like
  Wrapper       Pipeline      Assistant      Support Team     Code Agent

  You covered this in Phase 1 ──────►  This track builds this ──────────►
```

---

## 3. Agent Architecture Patterns

### 3.1 ReAct Agent (Single Loop)
```
Best for: Tasks solvable with 1 agent + N tools
Example: "Look up this customer and check their order status"

┌──────────────────────────────────────┐
│              LLM (Brain)             │
│  System prompt: "You are a customer  │
│  service agent. Use tools to help."  │
└──────────┬───────────────────────────┘
           │ reason → act → observe → repeat
    ┌──────┴───────────────────┐
    │         Tool Belt         │
    ├──────────┬──────┬────────┤
    │ DB Query │ API  │ Email  │
    │          │ Call │ Send   │
    └──────────┴──────┴────────┘
```

### 3.2 Plan-and-Execute Agent
```
Best for: Complex multi-step tasks where upfront planning matters
Example: "Write a comprehensive report on market trends"

Step 1: LLM creates a plan (list of sub-tasks)
Step 2: Executor runs each sub-task (may be a ReAct agent itself)
Step 3: Replanner adjusts remaining plan based on completed steps

┌──────────┐     ┌──────────┐     ┌──────────┐
│ Planner  │────►│ Executor │────►│Replanner │──► back to Executor
│ (LLM)    │     │ (Agent)  │     │ (LLM)    │
└──────────┘     └──────────┘     └──────────┘
```

### 3.3 Supervisor Multi-Agent
```
Best for: Diverse sub-tasks requiring different specialisations
Example: "Analyse this codebase and write a technical report"

┌───────────────────────┐
│    Supervisor Agent    │  Decides who works next
└───────┬───────────────┘
        │ routes to specialist
   ┌────┼────────────┐
   ▼    ▼            ▼
┌──────┐ ┌─────────┐ ┌────────┐
│Code  │ │Research │ │Writer  │  Each is a ReAct agent
│Reader│ │Agent    │ │Agent   │  with its own tools
└──────┘ └─────────┘ └────────┘
```

### 3.4 Swarm / Handoff Pattern
```
Best for: Conversational agents that transfer between specialists
Example: Customer calls → Triage → Billing Agent → Technical Agent

Agent A (Triage): "This is a billing issue" → handoff to Agent B
Agent B (Billing): resolves billing → "There's also a tech issue" → handoff to Agent C
Agent C (Technical): resolves technical issue → done
```

### 3.5 Reflection / Self-Critique
```
Best for: Tasks where quality matters (code generation, writing)
Example: "Write production-quality code for this feature"

┌──────────┐     ┌──────────┐     ┌──────────┐
│Generator │────►│  Critic  │────►│Generator │──► Final output
│ (Draft)  │     │ (Review) │     │ (Revise) │
└──────────┘     └──────────┘     └──────────┘
```

---

## 4. Framework Landscape

### Java Ecosystem

| Framework | Strengths | When to Use |
|-----------|-----------|-------------|
| **Spring AI** | Native Spring Boot, auto-config, enterprise-ready | Spring Boot projects, enterprise apps |
| **LangChain4j** | Rich agent/chain/memory abstractions, multiple LLM providers | Complex agent logic, multi-provider needs |
| **Semantic Kernel (Java)** | Microsoft-backed, Azure integration | Azure-heavy environments |

### Python Ecosystem

| Framework | Strengths | When to Use |
|-----------|-----------|-------------|
| **LangGraph** | Stateful graphs, cycles, human-in-loop, production-grade | Complex multi-agent workflows, state machines |
| **CrewAI** | Role-based teams, easy multi-agent, built on LangChain | Multi-agent teams, quick prototyping |
| **AutoGen (Microsoft)** | Multi-agent conversations, code execution | Research, conversational multi-agent |
| **Claude Agent SDK** | Anthropic-native, tool use, agentic loops | Claude-specific applications |
| **OpenAI Agents SDK** | OpenAI-native, handoffs, guardrails | OpenAI-specific applications |

### Choosing Java vs Python for This Track

```
Use Java (Spring AI / LangChain4j) when:
  ✅ Building production backend services
  ✅ Integrating agents into existing Spring Boot microservices
  ✅ Enterprise/compliance requirements
  ✅ You need strong typing, DI, and Spring ecosystem

Use Python (LangGraph / CrewAI) when:
  ✅ Rapid prototyping of agent logic
  ✅ Complex multi-agent graphs with cycles
  ✅ ML/data-heavy pipelines
  ✅ Broader ecosystem of AI libraries (transformers, etc.)

This track covers BOTH — you'll build some projects in Java, some in Python.
```

---

## 5. Environment Setup

### Java (Spring Boot + Spring AI + LangChain4j)

```xml
<!-- pom.xml base dependencies -->
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>3.3.0</version>
</parent>

<dependencies>
    <dependency>
        <groupId>org.springframework.ai</groupId>
        <artifactId>spring-ai-openai-spring-boot-starter</artifactId>
    </dependency>
    <!-- Or for Anthropic Claude -->
    <dependency>
        <groupId>org.springframework.ai</groupId>
        <artifactId>spring-ai-anthropic-spring-boot-starter</artifactId>
    </dependency>

    <!-- LangChain4j (alternative/complementary) -->
    <dependency>
        <groupId>dev.langchain4j</groupId>
        <artifactId>langchain4j-spring-boot-starter</artifactId>
        <version>0.36.0</version>
    </dependency>
    <dependency>
        <groupId>dev.langchain4j</groupId>
        <artifactId>langchain4j-anthropic-spring-boot-starter</artifactId>
        <version>0.36.0</version>
    </dependency>
</dependencies>
```

```yaml
# application.yml
spring:
  ai:
    anthropic:
      api-key: ${ANTHROPIC_API_KEY}
      chat:
        options:
          model: claude-sonnet-4-20250514
          max-tokens: 4096
```

### Python (LangGraph + CrewAI)

```bash
# Create virtual environment
python -m venv agentic-ai-env
source agentic-ai-env/bin/activate

# Core dependencies
pip install langgraph langchain-anthropic langchain-openai
pip install crewai crewai-tools
pip install anthropic openai
pip install python-dotenv

# .env file
ANTHROPIC_API_KEY=sk-ant-xxx
OPENAI_API_KEY=sk-xxx
```

---

## 6. Your First Agent — Quick Verification (Both Languages)

### Java (Spring AI)

```java
@SpringBootApplication
public class AgentApp {
    public static void main(String[] args) {
        SpringApplication.run(AgentApp.class, args);
    }
}

@RestController
@RequiredArgsConstructor
public class AgentController {

    private final ChatClient chatClient;

    @PostMapping("/agent/ask")
    public String ask(@RequestBody String question) {
        return chatClient.prompt()
            .system("You are a helpful assistant. Think step by step.")
            .user(question)
            .tools(new TimeTools(), new MathTools())
            .call()
            .content();
    }
}

// Simple tool
class TimeTools {
    @Tool("Get the current date and time")
    public String getCurrentTime() {
        return LocalDateTime.now().toString();
    }
}
```

### Python (LangGraph)

```python
from langchain_anthropic import ChatAnthropic
from langchain_core.tools import tool
from langgraph.prebuilt import create_react_agent

@tool
def get_current_time() -> str:
    """Get the current date and time."""
    from datetime import datetime
    return datetime.now().isoformat()

@tool
def calculate(expression: str) -> str:
    """Evaluate a mathematical expression. Example: '2 + 2'"""
    return str(eval(expression))

model = ChatAnthropic(model="claude-sonnet-4-20250514")
agent = create_react_agent(model, tools=[get_current_time, calculate])

result = agent.invoke({"messages": [("user", "What's 15% of today's date day number?")]})
print(result["messages"][-1].content)
```

---

## 7. Track Roadmap

| Week | Focus | Projects |
|------|-------|---------|
| **Week 1** (Days 1-7) | Foundations & Single Agent | Research Assistant |
| **Week 2** (Days 8-14) | Multi-Agent & Advanced Patterns | Customer Support System |
| **Week 3** (Days 15-21) | Production Projects & Capstone | Code Review, DevOps, E-Commerce agents |

---

## 8. Hands-On Exercise

1. Set up both Java (Spring AI) and Python (LangGraph) development environments.
2. Run the verification agents above with your API keys.
3. Add a third tool to each (e.g., `searchWeb` or `readFile`) and test multi-tool usage.
4. Compare how Spring AI and LangGraph handle the ReAct loop differently.

---

## 9. Key Takeaways

1. **Agentic AI is a spectrum** — from simple chains to autonomous multi-agent systems; pick the right level.
2. **Java for production integration, Python for rapid prototyping** — you'll use both in this track.
3. **Pattern choice matters more than framework choice** — ReAct, Plan-and-Execute, Supervisor, Swarm each solve different problems.
4. **Phase 1 gave you theory; this track gives you implementation** — every day ends with running code.

---

*Day 1. The agent landscape is vast — but every complex system is built from simple patterns composed together.*
