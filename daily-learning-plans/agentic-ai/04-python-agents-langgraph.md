# Agentic AI — Day 4: Python Agents with LangGraph
**Track:** Agentic AI (Implementation-Focused)
**Time Target:** 1–2 hours

---

## 1. Today's Focus
**LangGraph — Stateful Agent Graphs with Cycles, Conditions, and Human-in-the-Loop**

LangGraph is the production Python framework for building agentic applications. Unlike simple ReAct loops, LangGraph models agents as **state graphs** — nodes are functions, edges are transitions, and the graph can cycle (agent loops), branch (conditional routing), and pause (human approval). Today you build real agents in Python and understand why LangGraph is the go-to for complex agentic workflows.

---

## 2. LangGraph Core Concepts

```
┌───────────────────────────────────────────────┐
│                 StateGraph                     │
│                                                │
│  ┌────────┐    ┌──────────┐    ┌────────────┐ │
│  │  START  │───►│  Agent   │───►│  Tools     │ │
│  └────────┘    │  (LLM)   │◄───│  (Execute) │ │
│                └────┬─────┘    └────────────┘ │
│                     │                          │
│              should_continue?                  │
│              ┌──────┴──────┐                   │
│              ▼             ▼                   │
│         ┌────────┐   ┌────────┐               │
│         │  END   │   │ Tools  │ ──► back to   │
│         └────────┘   └────────┘     Agent     │
└───────────────────────────────────────────────┘

State = messages list (grows with each step)
Nodes = functions that transform state
Edges = transitions (conditional or unconditional)
```

---

## 3. Building a Research Agent

### 3.1 Simple ReAct Agent (Pre-built)

```python
from langchain_anthropic import ChatAnthropic
from langchain_core.tools import tool
from langgraph.prebuilt import create_react_agent

@tool
def search_web(query: str) -> str:
    """Search the web for information. Returns relevant results."""
    # In production: use Tavily, SerpAPI, or Brave Search
    import requests
    response = requests.get(
        "https://api.tavily.com/search",
        params={"query": query, "api_key": os.getenv("TAVILY_API_KEY")}
    )
    results = response.json().get("results", [])
    return "\n".join(f"- {r['title']}: {r['content'][:200]}" for r in results[:3])

@tool
def read_url(url: str) -> str:
    """Read and extract text content from a URL."""
    import requests
    from bs4 import BeautifulSoup
    response = requests.get(url, timeout=10)
    soup = BeautifulSoup(response.text, "html.parser")
    return soup.get_text()[:3000]  # Limit to 3000 chars

@tool
def save_note(title: str, content: str) -> str:
    """Save a research note for later reference."""
    notes_dir = Path("research_notes")
    notes_dir.mkdir(exist_ok=True)
    filepath = notes_dir / f"{title.replace(' ', '_')}.md"
    filepath.write_text(f"# {title}\n\n{content}")
    return f"Note saved: {filepath}"

model = ChatAnthropic(model="claude-sonnet-4-20250514")

research_agent = create_react_agent(
    model,
    tools=[search_web, read_url, save_note],
    prompt="You are a research assistant. Search for information, read sources, "
           "and compile your findings into structured notes. Always cite sources."
)

# Run
result = research_agent.invoke({
    "messages": [("user", "Research the latest developments in Java virtual threads and write a summary note")]
})
print(result["messages"][-1].content)
```

### 3.2 Custom StateGraph (Full Control)

```python
from typing import TypedDict, Annotated, Literal
from langgraph.graph import StateGraph, START, END
from langgraph.graph.message import add_messages
from langchain_core.messages import HumanMessage, AIMessage

# Define state schema
class ResearchState(TypedDict):
    messages: Annotated[list, add_messages]
    research_topic: str
    sources_found: list[str]
    notes: list[str]
    iteration: int

# Define nodes (each is a function: State → State)
def plan_research(state: ResearchState) -> dict:
    """LLM decides what to research next."""
    model = ChatAnthropic(model="claude-sonnet-4-20250514")
    response = model.invoke([
        ("system", "You are a research planner. Given the topic and any existing notes, "
                   "decide what to search for next. Return a search query."),
        ("user", f"Topic: {state['research_topic']}\n"
                 f"Notes so far: {state['notes']}\n"
                 f"Iteration: {state['iteration']}/3")
    ])
    return {"messages": [response]}

def execute_search(state: ResearchState) -> dict:
    """Execute the search based on LLM's plan."""
    last_message = state["messages"][-1].content
    results = search_web.invoke({"query": last_message})
    return {
        "sources_found": state.get("sources_found", []) + [results],
        "messages": [HumanMessage(content=f"Search results:\n{results}")]
    }

def synthesise(state: ResearchState) -> dict:
    """LLM synthesises findings into a note."""
    model = ChatAnthropic(model="claude-sonnet-4-20250514")
    all_sources = "\n\n".join(state["sources_found"])
    response = model.invoke([
        ("system", "Synthesise the search results into a clear, structured research note."),
        ("user", f"Topic: {state['research_topic']}\nSources:\n{all_sources}")
    ])
    return {
        "notes": state.get("notes", []) + [response.content],
        "messages": [response],
        "iteration": state["iteration"] + 1
    }

# Conditional edge: should we research more?
def should_continue(state: ResearchState) -> Literal["plan", "end"]:
    if state["iteration"] >= 3:
        return "end"
    return "plan"

# Build the graph
graph = StateGraph(ResearchState)

graph.add_node("plan", plan_research)
graph.add_node("search", execute_search)
graph.add_node("synthesise", synthesise)

graph.add_edge(START, "plan")
graph.add_edge("plan", "search")
graph.add_edge("search", "synthesise")
graph.add_conditional_edges("synthesise", should_continue, {
    "plan": "plan",     # Loop back
    "end": END          # Finish
})

research_graph = graph.compile()
```

### 3.3 Running the Graph

```python
result = research_graph.invoke({
    "messages": [],
    "research_topic": "Spring Boot 3.3 new features and virtual threads",
    "sources_found": [],
    "notes": [],
    "iteration": 0
})

# Print final research notes
for note in result["notes"]:
    print(note)
```

---

## 4. Visualising the Graph

```python
# Generate a Mermaid diagram of your graph
print(research_graph.get_graph().draw_mermaid())

# Or render as PNG (requires graphviz)
from IPython.display import Image
Image(research_graph.get_graph().draw_mermaid_png())
```

---

## 5. Adding Persistence (Checkpointing)

```python
from langgraph.checkpoint.memory import MemorySaver

# Add checkpointer — saves state after each node
checkpointer = MemorySaver()
research_graph = graph.compile(checkpointer=checkpointer)

# Run with a thread_id (conversation ID)
config = {"configurable": {"thread_id": "research-001"}}
result = research_graph.invoke(
    {"messages": [("user", "Research Java 21 features")],
     "research_topic": "Java 21", "sources_found": [], "notes": [], "iteration": 0},
    config=config
)

# Later: resume the same conversation
result2 = research_graph.invoke(
    {"messages": [("user", "Now compare with Java 17")]},
    config=config   # Same thread_id → loads previous state
)
```

---

## 6. Error Handling & Retries

```python
from langgraph.errors import NodeInterrupt

def execute_search_safe(state: ResearchState) -> dict:
    """Search with error handling."""
    try:
        last_message = state["messages"][-1].content
        results = search_web.invoke({"query": last_message})
        if not results:
            return {"messages": [HumanMessage(content="No results found. Try a different query.")]}
        return {
            "sources_found": state.get("sources_found", []) + [results],
            "messages": [HumanMessage(content=f"Search results:\n{results}")]
        }
    except Exception as e:
        # Return error as message — let the LLM adapt
        return {"messages": [HumanMessage(content=f"Search failed: {str(e)}. Please try a different approach.")]}
```

---

## 7. Hands-On Exercise

1. Set up Python environment with `langgraph` and `langchain-anthropic`.
2. Build the simple ReAct agent with `create_react_agent` and 3 tools.
3. Build the custom `ResearchState` graph with plan → search → synthesise → loop.
4. Run the graph and observe the 3-iteration research cycle.
5. Add persistence with `MemorySaver` and test conversation resumption.
6. Visualise your graph with `draw_mermaid()`.

---

## 8. Key Takeaways

1. **LangGraph = state machine for agents** — nodes are functions, edges are transitions, cycles enable agent loops.
2. **`create_react_agent` for quick agents, `StateGraph` for full control** — start simple, upgrade when needed.
3. **State is explicit and typed** — `TypedDict` schema makes state visible and debuggable.
4. **Checkpointing enables persistence** — conversations survive restarts, failures, and can be replayed.
5. **Conditional edges are the decision points** — the LLM decides (or rules decide) where to go next.

---

*Day 4. Graphs are better than loops — they make agent behaviour visible, testable, and controllable.*
