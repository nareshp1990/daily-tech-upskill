# Agentic AI — Day 9: LangGraph Multi-Agent Workflows
**Track:** Agentic AI (Implementation-Focused)
**Time Target:** 1–2 hours

---

## 1. Today's Focus
**LangGraph Advanced — Sub-Graphs, Parallel Execution, and Human-in-the-Loop**

Yesterday you built a basic supervisor. Today you go deeper into LangGraph's advanced features: composing graphs inside graphs (sub-graphs), running agents in parallel, and pausing execution for human approval before proceeding. These are the patterns that make multi-agent systems production-ready.

---

## 2. Sub-Graphs — Agents as Composable Units

```python
from langgraph.graph import StateGraph, START, END

# --- Sub-graph: Research team ---
def build_research_subgraph():
    graph = StateGraph(ResearchState)
    graph.add_node("search", search_node)
    graph.add_node("analyse", analyse_node)
    graph.add_edge(START, "search")
    graph.add_edge("search", "analyse")
    graph.add_edge("analyse", END)
    return graph.compile()

# --- Sub-graph: Writing team ---
def build_writing_subgraph():
    graph = StateGraph(WritingState)
    graph.add_node("draft", draft_node)
    graph.add_node("review", review_node)
    graph.add_node("revise", revise_node)
    graph.add_edge(START, "draft")
    graph.add_edge("draft", "review")
    graph.add_conditional_edges("review", needs_revision, {
        "revise": "revise",
        "done": END
    })
    graph.add_edge("revise", "review")  # Re-review after revision
    return graph.compile()

# --- Parent graph: orchestrator ---
parent = StateGraph(OrchestratorState)
parent.add_node("research_team", build_research_subgraph())
parent.add_node("writing_team", build_writing_subgraph())
parent.add_edge(START, "research_team")
parent.add_edge("research_team", "writing_team")
parent.add_edge("writing_team", END)

full_pipeline = parent.compile()
```

---

## 3. Parallel Execution — Fan-Out/Fan-In

```python
from langgraph.graph import StateGraph, START, END
from typing import TypedDict, Annotated
import operator

class ParallelState(TypedDict):
    topic: str
    messages: Annotated[list, operator.add]
    research_results: Annotated[list, operator.add]  # Collects results from all branches

def research_academic(state: ParallelState) -> dict:
    model = ChatAnthropic(model="claude-sonnet-4-20250514")
    result = model.invoke([
        ("system", "Search academic sources (papers, journals) about the topic."),
        ("user", state["topic"])
    ])
    return {"research_results": [{"source": "academic", "content": result.content}]}

def research_industry(state: ParallelState) -> dict:
    model = ChatAnthropic(model="claude-sonnet-4-20250514")
    result = model.invoke([
        ("system", "Search industry blogs, tech talks, and engineering blogs about the topic."),
        ("user", state["topic"])
    ])
    return {"research_results": [{"source": "industry", "content": result.content}]}

def research_code(state: ParallelState) -> dict:
    model = ChatAnthropic(model="claude-sonnet-4-20250514")
    result = model.invoke([
        ("system", "Search GitHub repositories and code examples related to the topic."),
        ("user", state["topic"])
    ])
    return {"research_results": [{"source": "code", "content": result.content}]}

def synthesise_results(state: ParallelState) -> dict:
    model = ChatAnthropic(model="claude-sonnet-4-20250514")
    all_results = "\n\n---\n\n".join(
        f"[{r['source']}]: {r['content']}" for r in state["research_results"]
    )
    result = model.invoke([
        ("system", "Synthesise research from academic, industry, and code sources into a unified report."),
        ("user", f"Topic: {state['topic']}\n\nResults:\n{all_results}")
    ])
    return {"messages": [result.content]}

# Build with parallel branches
graph = StateGraph(ParallelState)
graph.add_node("academic", research_academic)
graph.add_node("industry", research_industry)
graph.add_node("code", research_code)
graph.add_node("synthesise", synthesise_results)

# Fan-out: START → all three in parallel
graph.add_edge(START, "academic")
graph.add_edge(START, "industry")
graph.add_edge(START, "code")

# Fan-in: all three → synthesise
graph.add_edge("academic", "synthesise")
graph.add_edge("industry", "synthesise")
graph.add_edge("code", "synthesise")
graph.add_edge("synthesise", END)

parallel_research = graph.compile()
```

---

## 4. Human-in-the-Loop (Interrupt & Approve)

```python
from langgraph.graph import StateGraph, START, END
from langgraph.checkpoint.memory import MemorySaver
from langgraph.types import interrupt, Command

class ApprovalState(TypedDict):
    messages: Annotated[list, add_messages]
    plan: str
    approved: bool

def create_plan(state: ApprovalState) -> dict:
    model = ChatAnthropic(model="claude-sonnet-4-20250514")
    response = model.invoke([
        ("system", "Create a detailed action plan for the task. List each step."),
        *state["messages"]
    ])
    return {"plan": response.content}

def await_approval(state: ApprovalState) -> dict:
    """Pause and wait for human approval."""
    # This will pause the graph and return to the caller
    answer = interrupt(
        {"question": f"Do you approve this plan?\n\n{state['plan']}\n\nRespond yes/no:"}
    )
    return {"approved": answer.lower().strip() in ("yes", "y", "approve")}

def execute_plan(state: ApprovalState) -> dict:
    if not state["approved"]:
        return {"messages": [("assistant", "Plan rejected. Please provide new instructions.")]}
    model = ChatAnthropic(model="claude-sonnet-4-20250514")
    response = model.invoke([
        ("system", "Execute the following plan step by step."),
        ("user", state["plan"])
    ])
    return {"messages": [response]}

graph = StateGraph(ApprovalState)
graph.add_node("plan", create_plan)
graph.add_node("approve", await_approval)
graph.add_node("execute", execute_plan)

graph.add_edge(START, "plan")
graph.add_edge("plan", "approve")
graph.add_edge("approve", "execute")
graph.add_edge("execute", END)

approval_agent = graph.compile(checkpointer=MemorySaver())

# --- Usage ---
config = {"configurable": {"thread_id": "task-001"}}

# Phase 1: runs until interrupt
result = approval_agent.invoke(
    {"messages": [("user", "Refactor the OrderService to use the repository pattern")],
     "plan": "", "approved": False},
    config
)
# Graph pauses at approve node, shows plan to user

# Phase 2: user approves, resume
result = approval_agent.invoke(
    Command(resume="yes"),
    config
)
# Graph continues from where it paused
```

---

## 5. Error Recovery in Multi-Agent Systems

```python
def safe_node(func):
    """Decorator: catch exceptions in agent nodes and recover gracefully."""
    def wrapper(state):
        try:
            return func(state)
        except Exception as e:
            return {
                "messages": [("assistant", f"[ERROR in {func.__name__}]: {str(e)}. Skipping.")],
                "errors": state.get("errors", []) + [{"node": func.__name__, "error": str(e)}]
            }
    wrapper.__name__ = func.__name__
    return wrapper

@safe_node
def risky_search(state):
    # If this fails, the error is caught and the graph continues
    results = search_web.invoke({"query": state["topic"]})
    return {"research_results": [results]}
```

---

## 6. Hands-On Exercise

1. Build the **parallel research graph** with 3 branches (academic, industry, code).
2. Run it on a topic and observe all 3 branches executing and merging results.
3. Implement the **human-in-the-loop approval** flow — pause, print plan, resume.
4. Create a **sub-graph** that encapsulates the research step and use it as a node in a parent graph.
5. Add error recovery using the `safe_node` decorator.

---

## 7. Key Takeaways

1. **Sub-graphs enable composition** — build small agents, compose them into larger systems.
2. **Parallel execution reduces latency** — 3 research agents in parallel = 1/3 the wall-clock time.
3. **Human-in-the-loop via interrupt is essential** — high-stakes decisions need human approval.
4. **Error recovery must be built in** — one failing agent shouldn't bring down the entire system.

---

*Day 9. Advanced agent workflows are just graphs with smarter edges — parallel for speed, interrupt for safety.*
