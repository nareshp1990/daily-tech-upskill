# Agentic AI — Day 16: Project — Automated Code Review Agent
**Track:** Agentic AI (Implementation-Focused)
**Time Target:** 1–2 hours

---

## 1. Today's Focus
**Build a Code Review Agent That Reads PRs, Analyses Code, and Writes Review Comments**

This agent integrates with GitHub, reads pull request diffs, analyses code for quality/security/performance issues, and posts structured review comments. It's a practical agent you can actually deploy — combining file reading tools, LLM reasoning, and API integration.

---

## 2. Architecture

```
GitHub Webhook (PR opened/updated)
       │
       ▼
┌──────────────────┐
│ Code Review Agent│
│ (LangGraph)      │
└───┬──────────────┘
    │
    ├── Tool: fetch_pr_diff(repo, pr_number)
    ├── Tool: read_file(repo, path, ref)
    ├── Tool: search_codebase(repo, query)
    ├── Tool: post_review_comment(repo, pr, file, line, body)
    │
    ▼
┌──────────────────┐
│ Review Output:   │
│ - Inline comments│
│ - Summary review │
│ - Approval/      │
│   Request changes│
└──────────────────┘
```

---

## 3. GitHub Tools

```python
import os
import requests
from langchain_core.tools import tool

GITHUB_TOKEN = os.getenv("GITHUB_TOKEN")
HEADERS = {"Authorization": f"token {GITHUB_TOKEN}", "Accept": "application/vnd.github.v3+json"}

@tool
def fetch_pr_diff(repo: str, pr_number: int) -> str:
    """Fetch the diff of a pull request. Returns the changed files with line-by-line diffs.
    Example: repo='owner/repo', pr_number=42"""
    url = f"https://api.github.com/repos/{repo}/pulls/{pr_number}/files"
    response = requests.get(url, headers=HEADERS)
    files = response.json()
    output = []
    for f in files:
        output.append(f"### {f['filename']} ({f['status']}, +{f['additions']}/-{f['deletions']})")
        if f.get("patch"):
            output.append(f"```diff\n{f['patch'][:3000]}\n```")  # Limit per file
    return "\n\n".join(output)

@tool
def read_file_from_repo(repo: str, filepath: str, ref: str = "main") -> str:
    """Read a file's full content from a GitHub repository at a specific branch/commit.
    Use this to understand the full context around changed code."""
    url = f"https://api.github.com/repos/{repo}/contents/{filepath}?ref={ref}"
    response = requests.get(url, headers=HEADERS)
    data = response.json()
    import base64
    content = base64.b64decode(data["content"]).decode("utf-8")
    return content[:5000]

@tool
def post_review_comment(repo: str, pr_number: int, body: str, event: str = "COMMENT") -> str:
    """Post a review on a pull request. Event can be: COMMENT, APPROVE, REQUEST_CHANGES.
    Body should be a structured review summary in Markdown."""
    url = f"https://api.github.com/repos/{repo}/pulls/{pr_number}/reviews"
    response = requests.post(url, headers=HEADERS, json={
        "body": body,
        "event": event
    })
    if response.status_code == 200:
        return f"Review posted: {event}"
    return f"Error posting review: {response.json()}"
```

---

## 4. Code Review Agent (LangGraph)

```python
from typing import TypedDict, Annotated
from langgraph.graph import StateGraph, START, END
from langgraph.graph.message import add_messages
from langchain_anthropic import ChatAnthropic
from pydantic import BaseModel

model = ChatAnthropic(model="claude-sonnet-4-20250514", max_tokens=4096)

class ReviewState(TypedDict):
    messages: Annotated[list, add_messages]
    repo: str
    pr_number: int
    diff: str
    issues: list[dict]
    summary: str

def fetch_changes(state: ReviewState) -> dict:
    diff = fetch_pr_diff.invoke({"repo": state["repo"], "pr_number": state["pr_number"]})
    return {"diff": diff, "messages": [("assistant", f"Fetched PR diff ({len(diff)} chars)")]}

class CodeIssue(BaseModel):
    file: str
    line: int
    severity: str       # critical, warning, suggestion
    category: str       # security, performance, bug, style, logic
    description: str
    suggestion: str

class ReviewResult(BaseModel):
    issues: list[CodeIssue]
    overall_quality: str   # good, needs_work, critical_issues

def analyse_code(state: ReviewState) -> dict:
    result = model.with_structured_output(ReviewResult).invoke([
        ("system", """You are a senior code reviewer (12+ years Java/Spring Boot experience).
        Review this PR diff for:
        1. **Bugs**: Logic errors, null pointer risks, off-by-one errors
        2. **Security**: SQL injection, XSS, auth bypass, hardcoded secrets
        3. **Performance**: N+1 queries, unnecessary allocations, missing indexes
        4. **Design**: SOLID violations, missing error handling, unclear naming
        5. **Style**: Inconsistent formatting, dead code, missing tests

        Be specific: cite exact line numbers, explain why it's an issue, suggest a fix.
        Only report real issues — don't nitpick."""),
        ("user", f"PR diff:\n{state['diff']}")
    ])
    return {"issues": [issue.dict() for issue in result.issues]}

def write_review(state: ReviewState) -> dict:
    issues = state["issues"]
    if not issues:
        summary = "## Code Review: Looks Good!\n\nNo significant issues found. LGTM."
        event = "APPROVE"
    else:
        critical = [i for i in issues if i["severity"] == "critical"]
        warnings = [i for i in issues if i["severity"] == "warning"]
        suggestions = [i for i in issues if i["severity"] == "suggestion"]

        parts = ["## Code Review Summary\n"]
        if critical:
            parts.append(f"### Critical Issues ({len(critical)})")
            for i in critical:
                parts.append(f"- **{i['file']}:{i['line']}** [{i['category']}] {i['description']}\n  > Fix: {i['suggestion']}")
        if warnings:
            parts.append(f"\n### Warnings ({len(warnings)})")
            for i in warnings:
                parts.append(f"- **{i['file']}:{i['line']}** [{i['category']}] {i['description']}")
        if suggestions:
            parts.append(f"\n### Suggestions ({len(suggestions)})")
            for i in suggestions:
                parts.append(f"- **{i['file']}:{i['line']}** {i['description']}")

        summary = "\n".join(parts)
        event = "REQUEST_CHANGES" if critical else "COMMENT"

    return {"summary": summary, "messages": [("assistant", summary)]}

def post_review(state: ReviewState) -> dict:
    has_critical = any(i["severity"] == "critical" for i in state["issues"])
    event = "REQUEST_CHANGES" if has_critical else ("APPROVE" if not state["issues"] else "COMMENT")
    result = post_review_comment.invoke({
        "repo": state["repo"], "pr_number": state["pr_number"],
        "body": state["summary"], "event": event
    })
    return {"messages": [("assistant", result)]}

# Build graph
graph = StateGraph(ReviewState)
graph.add_node("fetch", fetch_changes)
graph.add_node("analyse", analyse_code)
graph.add_node("write", write_review)
graph.add_node("post", post_review)

graph.add_edge(START, "fetch")
graph.add_edge("fetch", "analyse")
graph.add_edge("analyse", "write")
graph.add_edge("write", "post")
graph.add_edge("post", END)

review_agent = graph.compile()

# --- Run ---
result = review_agent.invoke({
    "messages": [], "repo": "youruser/yourrepo", "pr_number": 42,
    "diff": "", "issues": [], "summary": ""
})
```

---

## 5. Webhook Integration (FastAPI)

```python
from fastapi import FastAPI, Request

app = FastAPI()

@app.post("/webhook/github")
async def github_webhook(request: Request):
    payload = await request.json()
    if payload.get("action") in ("opened", "synchronize"):
        pr = payload["pull_request"]
        repo = payload["repository"]["full_name"]
        pr_number = pr["number"]

        # Run review agent async
        review_agent.invoke({
            "messages": [], "repo": repo, "pr_number": pr_number,
            "diff": "", "issues": [], "summary": ""
        })
    return {"status": "ok"}
```

---

## 6. Hands-On Exercise

1. Set up a GitHub personal access token with `repo` scope.
2. Implement the 3 GitHub tools (fetch diff, read file, post review).
3. Build the review agent graph and test on a real PR in your repository.
4. Test with deliberately bad code (SQL injection, null pointer risk) and verify detection.
5. (Optional) Set up the FastAPI webhook to auto-review new PRs.

---

## 7. Key Takeaways

1. **Structured output (Pydantic) makes review results machine-readable** — severity, category, line numbers.
2. **Separate fetch → analyse → write → post** — each step is testable and debuggable.
3. **Limit diff size per file** — LLMs have context limits; truncate large diffs.
4. **Don't auto-approve** — use COMMENT or REQUEST_CHANGES; human makes final approval.

---

*Day 16. A code review agent doesn't replace reviewers — it catches the obvious so humans can focus on the subtle.*
