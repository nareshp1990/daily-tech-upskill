# daily-tech-upskill — Project Instructions

## Knowledge Graph (graphify RAG)

A knowledge graph of this entire codebase lives at `graphify-out/graph.json`.

**Before answering questions about content in this repo** (learning plans, AI tools, agentic-ai topics, system design topics, prompt library), query the graph first:

```
/graphify query "<your question>"
```

Or use path traversal for concept-to-concept questions:
```
/graphify path "RAG" "Multi-Agent Orchestration"
```

Or explain a specific concept:
```
/graphify explain "Model Context Protocol"
```

### What the graph contains
- 128 nodes across 21 communities
- Covers: daily learning plans (Phase 1 AI + Phase 2 System Design), agentic-ai track, AI tools (Claude/Copilot/Gemini), prompt library, AI daily news
- Key god nodes (most connected): Prompt Library Overview, Anthropic Claude AI Tools, Daily Tech Upskill Project, MCP, Multi-Agent Orchestration

### When to use
- "What topics are covered in Phase 1?" → query the graph
- "How does RAG connect to agents?" → path query
- "What's in the prompt library?" → explain query
- "Which system design case studies exist?" → query the graph

### Rebuild the graph
If new files are added, run `/graphify . --update` to incrementally re-extract only changed files.
