# Agentic AI — Day 7: Project — Intelligent Research Assistant Agent
**Track:** Agentic AI (Implementation-Focused)
**Time Target:** 1–2 hours

---

## 1. Today's Focus
**Week 1 Project: Build a Research Assistant That Searches, Reads, Analyses, and Writes Reports**

This is your first real project. You'll combine everything from Week 1 — Spring AI tools (Day 2), LangGraph graphs (Day 4), custom tools (Day 5), and memory (Day 6) — into a working research assistant that can take a topic, search the web, read sources, synthesise findings, and produce a structured Markdown report.

---

## 2. Project Requirements

**Input:** A research topic (e.g., "Compare Spring AI vs LangChain4j for building production agents")
**Output:** A structured Markdown report with:
- Executive summary
- Key findings (3-5 bullet points)
- Detailed analysis with citations
- Comparison table (if applicable)
- Recommendations

**Agent Capabilities:**
- Search the web for relevant sources (3-5 searches)
- Read and extract content from URLs
- Synthesise information from multiple sources
- Write structured reports
- Persist conversation state (resume if interrupted)

---

## 3. Option A: Python Implementation (LangGraph)

```python
import os
from typing import TypedDict, Annotated, Literal
from pathlib import Path
from datetime import datetime

from langchain_anthropic import ChatAnthropic
from langchain_core.tools import tool
from langchain_core.messages import HumanMessage
from langgraph.graph import StateGraph, START, END
from langgraph.graph.message import add_messages
from langgraph.checkpoint.memory import MemorySaver

# --- Tools ---
@tool
def search_web(query: str) -> str:
    """Search the web for information on a topic. Returns top 3 results with titles and snippets."""
    import requests
    response = requests.get("https://api.tavily.com/search", params={
        "query": query, "api_key": os.getenv("TAVILY_API_KEY"),
        "max_results": 3, "include_raw_content": False
    })
    results = response.json().get("results", [])
    return "\n\n".join(
        f"**{r['title']}**\nURL: {r['url']}\n{r['content'][:300]}"
        for r in results
    ) or "No results found."

@tool
def read_webpage(url: str) -> str:
    """Read and extract the main text content from a webpage URL."""
    import requests
    from bs4 import BeautifulSoup
    try:
        response = requests.get(url, timeout=10, headers={"User-Agent": "ResearchBot/1.0"})
        soup = BeautifulSoup(response.text, "html.parser")
        for tag in soup(["script", "style", "nav", "footer", "header"]):
            tag.decompose()
        text = soup.get_text(separator="\n", strip=True)
        return text[:4000]
    except Exception as e:
        return f"Error reading URL: {e}"

@tool
def save_report(filename: str, content: str) -> str:
    """Save the final research report as a Markdown file."""
    output_dir = Path("research_output")
    output_dir.mkdir(exist_ok=True)
    filepath = output_dir / f"{filename}.md"
    filepath.write_text(content)
    return f"Report saved: {filepath}"

# --- State ---
class ResearchState(TypedDict):
    messages: Annotated[list, add_messages]
    topic: str
    search_queries: list[str]
    sources: list[dict]
    findings: list[str]
    report: str
    phase: str  # planning, searching, reading, synthesising, writing

# --- Nodes ---
model = ChatAnthropic(model="claude-sonnet-4-20250514", max_tokens=4096)

def plan_research(state: ResearchState) -> dict:
    response = model.invoke([
        ("system", """You are a research planner. Given a topic, generate 3-4 specific
        search queries that will cover different angles of the topic.
        Return ONLY the queries, one per line."""),
        ("user", f"Research topic: {state['topic']}")
    ])
    queries = [q.strip() for q in response.content.strip().split("\n") if q.strip()]
    return {
        "search_queries": queries[:4],
        "phase": "searching",
        "messages": [response]
    }

def search_sources(state: ResearchState) -> dict:
    all_sources = []
    for query in state["search_queries"]:
        result = search_web.invoke({"query": query})
        all_sources.append({"query": query, "results": result})
    return {
        "sources": all_sources,
        "phase": "reading",
        "messages": [HumanMessage(content=f"Searched {len(all_sources)} queries")]
    }

def read_and_extract(state: ResearchState) -> dict:
    findings = []
    for source in state["sources"]:
        response = model.invoke([
            ("system", "Extract the 3 most important facts from these search results. Be specific and cite the source."),
            ("user", f"Query: {source['query']}\nResults:\n{source['results']}")
        ])
        findings.append(response.content)
    return {
        "findings": findings,
        "phase": "writing",
        "messages": [HumanMessage(content=f"Extracted findings from {len(findings)} sources")]
    }

def write_report(state: ResearchState) -> dict:
    all_findings = "\n\n---\n\n".join(state["findings"])
    response = model.invoke([
        ("system", """Write a comprehensive research report in Markdown format.
        Structure:
        # [Topic] — Research Report
        **Date:** [today]
        ## Executive Summary (2-3 sentences)
        ## Key Findings (3-5 bullet points)
        ## Detailed Analysis (3-4 paragraphs with citations)
        ## Comparison Table (if applicable)
        ## Recommendations
        ## Sources"""),
        ("user", f"Topic: {state['topic']}\n\nFindings:\n{all_findings}")
    ])

    filename = state["topic"].lower().replace(" ", "-")[:50]
    save_result = save_report.invoke({"filename": filename, "content": response.content})

    return {
        "report": response.content,
        "phase": "complete",
        "messages": [response, HumanMessage(content=save_result)]
    }

# --- Build Graph ---
graph = StateGraph(ResearchState)
graph.add_node("plan", plan_research)
graph.add_node("search", search_sources)
graph.add_node("read", read_and_extract)
graph.add_node("write", write_report)

graph.add_edge(START, "plan")
graph.add_edge("plan", "search")
graph.add_edge("search", "read")
graph.add_edge("read", "write")
graph.add_edge("write", END)

research_agent = graph.compile(checkpointer=MemorySaver())

# --- Run ---
if __name__ == "__main__":
    result = research_agent.invoke({
        "messages": [],
        "topic": "Spring AI vs LangChain4j for production Java agents in 2026",
        "search_queries": [],
        "sources": [],
        "findings": [],
        "report": "",
        "phase": "planning"
    }, {"configurable": {"thread_id": "research-001"}})

    print("\n" + "=" * 60)
    print(result["report"])
```

---

## 4. Option B: Java Implementation (Spring AI)

```java
@Service
@RequiredArgsConstructor
public class ResearchAgentService {

    private final ChatClient chatClient;
    private final WebSearchTool webSearchTool;

    public String research(String topic) {
        // Phase 1: Generate search queries
        String queries = chatClient.prompt()
            .system("Generate 3 specific search queries for this research topic. One per line.")
            .user(topic)
            .call().content();

        // Phase 2: Search and collect sources
        List<String> allResults = new ArrayList<>();
        for (String query : queries.split("\n")) {
            String results = webSearchTool.search(query.trim());
            allResults.add("Query: " + query + "\nResults: " + results);
        }

        // Phase 3: Synthesise and write report
        String sources = String.join("\n\n---\n\n", allResults);
        return chatClient.prompt()
            .system("""Write a research report in Markdown:
                # [Topic] — Research Report
                ## Executive Summary
                ## Key Findings
                ## Detailed Analysis
                ## Recommendations
                ## Sources""")
            .user("Topic: " + topic + "\n\nSources:\n" + sources)
            .call().content();
    }
}
```

---

## 5. Testing

```python
# Test scenarios
topics = [
    "Best practices for Java virtual threads in Spring Boot 3.3",
    "Comparing Redis vs Memcached for caching in microservices",
    "Model Context Protocol (MCP) for AI agent tool integration"
]

for topic in topics:
    print(f"\n{'='*60}")
    print(f"Researching: {topic}")
    result = research_agent.invoke({
        "messages": [], "topic": topic,
        "search_queries": [], "sources": [],
        "findings": [], "report": "", "phase": "planning"
    }, {"configurable": {"thread_id": f"test-{hash(topic)}"}})
    print(f"Report length: {len(result['report'])} chars")
    print(f"Queries used: {result['search_queries']}")
```

---

## 6. Enhancements to Try

- [ ] Add a "review" node that critiques the report and sends it back for revision
- [ ] Add URL reading capability for deeper source analysis
- [ ] Support follow-up questions ("Tell me more about the third finding")
- [ ] Generate a comparison table automatically when the topic involves comparison
- [ ] Add a "confidence score" for each finding based on source quality

---

## 7. Key Takeaways

1. **Multi-step agents are just state machines** — plan, execute, synthesise, output.
2. **LangGraph makes the flow explicit** — each node is testable independently.
3. **The quality of search queries determines report quality** — invest in the planning phase.
4. **Always save outputs** — research reports are valuable artifacts, not just chat responses.

---

*Day 7. Week 1 complete. You've built an agent that thinks, searches, reads, and writes — the foundation for everything ahead.*
