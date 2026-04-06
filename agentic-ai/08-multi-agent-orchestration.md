# Agentic AI — Day 8: Multi-Agent Orchestration Deep Dive
**Track:** Agentic AI (Implementation-Focused)
**Time Target:** 1–2 hours

---

## 1. Today's Focus
**Multi-Agent Systems — Supervisor, Swarm, and Hierarchical Patterns in Code**

Week 1 built single agents. Week 2 is about teams of agents. In the real world, complex tasks need specialists: one agent reads code, another writes tests, a third reviews quality. The question is: who decides what each agent does? Today you implement three orchestration patterns — supervisor, swarm/handoff, and hierarchical — and learn when each shines.

---

## 2. Pattern 1: Supervisor Agent (LangGraph)

The supervisor decides which specialist to call next.

```python
from typing import Literal
from langchain_anthropic import ChatAnthropic
from langgraph.graph import StateGraph, START, END
from langgraph.graph.message import add_messages
from typing import TypedDict, Annotated
from pydantic import BaseModel

# --- Specialist agents ---
def code_analyst(state: dict) -> dict:
    model = ChatAnthropic(model="claude-sonnet-4-20250514")
    response = model.invoke([
        ("system", "You are a code analyst. Analyse the code provided and identify issues, "
                   "patterns, and improvement opportunities."),
        *state["messages"]
    ])
    return {"messages": [("assistant", f"[Code Analyst] {response.content}")]}

def test_writer(state: dict) -> dict:
    model = ChatAnthropic(model="claude-sonnet-4-20250514")
    response = model.invoke([
        ("system", "You are a test writer. Write comprehensive unit tests for the code discussed. "
                   "Use JUnit 5 for Java or pytest for Python."),
        *state["messages"]
    ])
    return {"messages": [("assistant", f"[Test Writer] {response.content}")]}

def doc_writer(state: dict) -> dict:
    model = ChatAnthropic(model="claude-sonnet-4-20250514")
    response = model.invoke([
        ("system", "You are a documentation writer. Write clear, concise documentation "
                   "for the code and analysis discussed."),
        *state["messages"]
    ])
    return {"messages": [("assistant", f"[Doc Writer] {response.content}")]}

# --- Supervisor ---
class SupervisorDecision(BaseModel):
    next_agent: Literal["code_analyst", "test_writer", "doc_writer", "FINISH"]
    reasoning: str

def supervisor(state: dict) -> dict:
    model = ChatAnthropic(model="claude-sonnet-4-20250514")
    decision = model.with_structured_output(SupervisorDecision).invoke([
        ("system", """You are a supervisor managing a team:
        - code_analyst: analyses code quality, patterns, and issues
        - test_writer: writes unit tests
        - doc_writer: writes documentation

        Based on the conversation so far, decide who should work next.
        Choose FINISH when the task is complete.
        Typical flow: analyst first → test writer → doc writer → FINISH"""),
        *state["messages"]
    ])
    return {
        "messages": [("assistant", f"[Supervisor] Next: {decision.next_agent} — {decision.reasoning}")],
        "next_agent": decision.next_agent
    }

# --- Build graph ---
class TeamState(TypedDict):
    messages: Annotated[list, add_messages]
    next_agent: str

def route_to_agent(state: TeamState) -> str:
    return state.get("next_agent", "supervisor")

graph = StateGraph(TeamState)
graph.add_node("supervisor", supervisor)
graph.add_node("code_analyst", code_analyst)
graph.add_node("test_writer", test_writer)
graph.add_node("doc_writer", doc_writer)

graph.add_edge(START, "supervisor")
graph.add_conditional_edges("supervisor", route_to_agent, {
    "code_analyst": "code_analyst",
    "test_writer": "test_writer",
    "doc_writer": "doc_writer",
    "FINISH": END
})
# After each specialist, go back to supervisor
graph.add_edge("code_analyst", "supervisor")
graph.add_edge("test_writer", "supervisor")
graph.add_edge("doc_writer", "supervisor")

team = graph.compile()

# --- Run ---
result = team.invoke({
    "messages": [("user", """Analyse this Java code, write tests, and document it:

    @Service
    public class OrderService {
        public Order createOrder(CreateOrderRequest req) {
            if (req.items().isEmpty()) throw new IllegalArgumentException("No items");
            BigDecimal total = req.items().stream()
                .map(i -> i.price().multiply(BigDecimal.valueOf(i.quantity())))
                .reduce(BigDecimal.ZERO, BigDecimal::add);
            return new Order(UUID.randomUUID(), req.userId(), req.items(), total, Instant.now());
        }
    }""")],
    "next_agent": "supervisor"
})
```

---

## 3. Pattern 2: Swarm / Handoff Pattern

Agents hand off to each other directly — no central supervisor.

```python
from langgraph.prebuilt import create_react_agent
from langchain_core.tools import tool

# Each agent can hand off to another
@tool
def transfer_to_billing() -> str:
    """Transfer this conversation to the billing specialist.
    Use this when the user has billing or payment questions."""
    return "Transferring to billing agent..."

@tool
def transfer_to_technical() -> str:
    """Transfer this conversation to the technical support specialist.
    Use this when the user has technical issues or needs troubleshooting."""
    return "Transferring to technical agent..."

@tool
def transfer_to_triage() -> str:
    """Transfer back to the triage agent if the issue doesn't fit your speciality."""
    return "Transferring back to triage..."

# Triage agent
triage_agent = create_react_agent(
    model,
    tools=[transfer_to_billing, transfer_to_technical],
    prompt="You are a customer service triage agent. Determine the user's issue "
           "and transfer to the appropriate specialist."
)

# Billing agent
billing_agent = create_react_agent(
    model,
    tools=[check_balance, process_refund, transfer_to_triage, transfer_to_technical],
    prompt="You are a billing specialist. Help with payment issues, refunds, and billing questions."
)

# Technical agent
technical_agent = create_react_agent(
    model,
    tools=[check_system_status, reset_password, transfer_to_triage, transfer_to_billing],
    prompt="You are a technical support specialist. Help with system issues and troubleshooting."
)
```

---

## 4. Pattern 3: Java Supervisor (Spring AI)

```java
@Service
@RequiredArgsConstructor
public class AgentSupervisor {

    private final ChatClient supervisorClient;
    private final Map<String, ChatClient> specialists;

    public String executeTask(String task) {
        List<String> conversationHistory = new ArrayList<>();
        conversationHistory.add("Task: " + task);

        for (int i = 0; i < 5; i++) { // Max 5 iterations
            // Supervisor decides next action
            String decision = supervisorClient.prompt()
                .system("""You manage: code_analyst, test_writer, doc_writer.
                    Return JSON: {"agent": "name_or_FINISH", "instruction": "what to do"}""")
                .user(String.join("\n", conversationHistory))
                .call().content();

            var parsed = objectMapper.readValue(decision, SupervisorDecision.class);
            if ("FINISH".equals(parsed.agent())) break;

            // Route to specialist
            ChatClient specialist = specialists.get(parsed.agent());
            String result = specialist.prompt()
                .user(parsed.instruction() + "\n\nContext:\n" + String.join("\n", conversationHistory))
                .call().content();

            conversationHistory.add("[" + parsed.agent() + "]: " + result);
        }

        return String.join("\n\n", conversationHistory);
    }
}
```

---

## 5. When to Use Which Pattern

| Pattern | Best For | Complexity | Control |
|---------|----------|------------|---------|
| **Supervisor** | Well-defined workflows, clear specialist roles | Medium | High — supervisor controls flow |
| **Swarm/Handoff** | Conversational agents, customer support | Low | Low — agents decide themselves |
| **Hierarchical** | Very complex tasks needing sub-teams | High | High — tree of supervisors |
| **Parallel Fan-out** | Independent sub-tasks that can run concurrently | Medium | Medium |

---

## 6. Hands-On Exercise

1. Implement the **Supervisor pattern** in LangGraph with 3 specialists.
2. Test with a real code snippet — observe the supervisor routing sequence.
3. Implement the **Swarm pattern** with triage → billing → technical agents.
4. Compare: same task, supervisor vs single-agent — observe quality difference.
5. Add iteration limits and timeout guards to prevent infinite loops.

---

## 7. Key Takeaways

1. **Supervisor pattern is the most controllable** — you can log, audit, and limit every routing decision.
2. **Swarm is the most natural for conversations** — users don't notice agent switches.
3. **Always set maximum iterations** — without limits, agents can loop forever.
4. **Specialists should have narrow, focused system prompts** — broad prompts produce mediocre results.

---

*Day 8. One agent is a worker. A team of agents is an organisation — and every organisation needs structure.*
