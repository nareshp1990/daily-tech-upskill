# Agentic AI — Day 5: Custom Tool Development
**Track:** Agentic AI (Implementation-Focused)
**Time Target:** 1–2 hours

---

## 1. Today's Focus
**Building Production-Grade Tools — REST APIs, Databases, Files, and External Services**

An agent is only as useful as its tools. Days 2-4 used simple tools. Today you build the kinds of tools real agents need: querying databases, calling REST APIs, reading/writing files, executing code, and integrating with external services (Slack, Jira, GitHub). You'll learn tool design principles that determine whether the LLM uses your tools correctly — or hallucinates parameters.

---

## 2. Tool Design Principles

```
Good tool description:
  "Search for Jira issues by project key and status. Returns issue key, summary, and assignee.
   Example: project='BACKEND', status='In Progress'"

Bad tool description:
  "Search Jira"   ← LLM doesn't know what params to pass or what it returns

Rules:
  1. Describe what it does, what it returns, and give examples
  2. Parameter names should be self-explanatory
  3. Return structured data (not raw HTML/XML)
  4. Handle errors gracefully — return error messages, don't throw exceptions
  5. Limit output size — LLMs have context limits
```

---

## 3. Tool Categories & Implementations

### 3.1 Database Query Tools (Java)

```java
@Component
@RequiredArgsConstructor
public class DatabaseTools {

    private final JdbcTemplate jdbc;

    @Tool("Search for customers by name or email. Returns top 10 matches with id, name, email, and signup date.")
    public List<Map<String, Object>> searchCustomers(
            @ToolParam("Search term — matches name or email (partial match supported)") String searchTerm) {
        return jdbc.queryForList(
            "SELECT id, name, email, created_at FROM customers " +
            "WHERE name ILIKE ? OR email ILIKE ? LIMIT 10",
            "%" + searchTerm + "%", "%" + searchTerm + "%");
    }

    @Tool("Get order details including items, total, and status for a specific order ID")
    public Map<String, Object> getOrderDetails(
            @ToolParam("Order ID (numeric)") Long orderId) {
        Map<String, Object> order = jdbc.queryForMap(
            "SELECT o.id, o.status, o.total, o.created_at, c.name as customer_name " +
            "FROM orders o JOIN customers c ON o.customer_id = c.id WHERE o.id = ?",
            orderId);
        List<Map<String, Object>> items = jdbc.queryForList(
            "SELECT product_name, quantity, unit_price FROM order_items WHERE order_id = ?",
            orderId);
        order.put("items", items);
        return order;
    }

    @Tool("Run a read-only SQL query against the database. Only SELECT queries are allowed.")
    public String executeSqlQuery(
            @ToolParam("SQL SELECT query to execute. Must be a read-only query.") String sql) {
        // SECURITY: Only allow SELECT statements
        String trimmed = sql.trim().toUpperCase();
        if (!trimmed.startsWith("SELECT")) {
            return "ERROR: Only SELECT queries are allowed for safety.";
        }
        // SECURITY: Block dangerous keywords
        if (trimmed.contains("DROP") || trimmed.contains("DELETE") ||
            trimmed.contains("INSERT") || trimmed.contains("UPDATE")) {
            return "ERROR: Query contains forbidden keywords.";
        }
        try {
            List<Map<String, Object>> results = jdbc.queryForList(sql);
            // Limit output to prevent context overflow
            if (results.size() > 50) {
                return "Query returned " + results.size() + " rows. First 50:\n" +
                       results.subList(0, 50).toString();
            }
            return results.toString();
        } catch (Exception e) {
            return "Query error: " + e.getMessage();
        }
    }
}
```

### 3.2 REST API Tools (Java)

```java
@Component
@RequiredArgsConstructor
public class GitHubTools {

    private final RestTemplate restTemplate;

    @Tool("Search GitHub repositories by keyword. Returns name, description, stars, and URL.")
    public List<Map<String, Object>> searchRepositories(
            @ToolParam("Search keyword (e.g., 'spring-ai agent')") String query,
            @ToolParam(value = "Max results to return (default 5)", required = false) Integer limit) {
        int maxResults = limit != null ? Math.min(limit, 10) : 5;

        String url = "https://api.github.com/search/repositories?q={query}&sort=stars&per_page={limit}";
        Map<String, Object> response = restTemplate.getForObject(url, Map.class, query, maxResults);

        List<Map<String, Object>> items = (List<Map<String, Object>>) response.get("items");
        return items.stream().map(item -> Map.<String, Object>of(
            "name", item.get("full_name"),
            "description", item.getOrDefault("description", "N/A"),
            "stars", item.get("stargazers_count"),
            "url", item.get("html_url")
        )).toList();
    }

    @Tool("Get open issues for a GitHub repository")
    public List<Map<String, Object>> getIssues(
            @ToolParam("Repository full name (e.g., 'spring-projects/spring-ai')") String repo,
            @ToolParam(value = "Filter by label", required = false) String label) {
        String url = "https://api.github.com/repos/{repo}/issues?state=open&per_page=10";
        if (label != null) url += "&labels=" + label;

        List<Map<String, Object>> issues = restTemplate.exchange(
            url, HttpMethod.GET, null,
            new ParameterizedTypeReference<List<Map<String, Object>>>() {}, repo)
            .getBody();

        return issues.stream().map(issue -> Map.<String, Object>of(
            "number", issue.get("number"),
            "title", issue.get("title"),
            "state", issue.get("state"),
            "labels", issue.get("labels"),
            "created_at", issue.get("created_at")
        )).toList();
    }
}
```

### 3.3 File System Tools (Python)

```python
from pathlib import Path
from langchain_core.tools import tool

@tool
def read_file(filepath: str) -> str:
    """Read the contents of a file. Returns the file content as text.
    Use this to read code files, configuration files, or text documents.
    Example: filepath='src/main/java/App.java'"""
    path = Path(filepath)
    if not path.exists():
        return f"ERROR: File not found: {filepath}"
    if path.stat().st_size > 100_000:  # 100KB limit
        return f"ERROR: File too large ({path.stat().st_size} bytes). Read specific sections instead."
    return path.read_text()

@tool
def write_file(filepath: str, content: str) -> str:
    """Write content to a file. Creates the file if it doesn't exist.
    WARNING: This will overwrite existing files.
    Example: filepath='output/report.md', content='# Report\\n...'"""
    path = Path(filepath)
    path.parent.mkdir(parents=True, exist_ok=True)
    path.write_text(content)
    return f"File written: {filepath} ({len(content)} bytes)"

@tool
def list_directory(directory: str, pattern: str = "*") -> str:
    """List files in a directory matching a glob pattern.
    Example: directory='src', pattern='*.java'"""
    path = Path(directory)
    if not path.is_dir():
        return f"ERROR: Not a directory: {directory}"
    files = sorted(path.glob(pattern))[:50]  # Limit to 50 files
    return "\n".join(str(f) for f in files)

@tool
def search_in_files(directory: str, search_term: str, file_pattern: str = "*.py") -> str:
    """Search for a text pattern across files in a directory.
    Returns matching lines with file paths and line numbers.
    Example: directory='src', search_term='def process', file_pattern='*.py'"""
    results = []
    path = Path(directory)
    for file in path.rglob(file_pattern):
        try:
            for i, line in enumerate(file.read_text().splitlines(), 1):
                if search_term.lower() in line.lower():
                    results.append(f"{file}:{i}: {line.strip()}")
        except (UnicodeDecodeError, PermissionError):
            continue
    if not results:
        return f"No matches found for '{search_term}' in {directory}"
    return "\n".join(results[:30])  # Limit output
```

### 3.4 External Service Tools (Python — Slack & Jira)

```python
import requests
from langchain_core.tools import tool

@tool
def send_slack_message(channel: str, message: str) -> str:
    """Send a message to a Slack channel.
    Example: channel='#engineering', message='Deploy completed successfully'"""
    response = requests.post(
        "https://slack.com/api/chat.postMessage",
        headers={"Authorization": f"Bearer {os.getenv('SLACK_TOKEN')}"},
        json={"channel": channel, "text": message}
    )
    if response.json().get("ok"):
        return f"Message sent to {channel}"
    return f"ERROR: {response.json().get('error')}"

@tool
def create_jira_issue(project: str, summary: str, description: str,
                       issue_type: str = "Task") -> str:
    """Create a new Jira issue.
    Example: project='BACKEND', summary='Fix login bug', description='Users report...'"""
    response = requests.post(
        f"{os.getenv('JIRA_URL')}/rest/api/3/issue",
        auth=(os.getenv('JIRA_EMAIL'), os.getenv('JIRA_TOKEN')),
        json={
            "fields": {
                "project": {"key": project},
                "summary": summary,
                "description": {"type": "doc", "version": 1,
                    "content": [{"type": "paragraph",
                        "content": [{"type": "text", "text": description}]}]},
                "issuetype": {"name": issue_type}
            }
        }
    )
    data = response.json()
    if "key" in data:
        return f"Issue created: {data['key']} — {summary}"
    return f"ERROR: {data}"
```

---

## 4. Tool Safety Patterns

```python
# Pattern 1: Confirmation required for destructive actions
@tool
def delete_resource(resource_id: str, confirm: bool = False) -> str:
    """Delete a resource. Set confirm=True to actually delete.
    First call without confirm to see what will be deleted."""
    if not confirm:
        resource = get_resource(resource_id)
        return f"WILL DELETE: {resource}. Call again with confirm=True to proceed."
    # Actually delete
    ...

# Pattern 2: Rate limiting tools
from functools import lru_cache
import time

last_call_time = {}

@tool
def expensive_api_call(query: str) -> str:
    """Call an expensive external API. Rate limited to 1 call per 10 seconds."""
    now = time.time()
    if now - last_call_time.get("expensive_api", 0) < 10:
        return "ERROR: Rate limited. Please wait before calling again."
    last_call_time["expensive_api"] = now
    # Make the actual call...

# Pattern 3: Output truncation
@tool
def query_large_dataset(query: str) -> str:
    """Query a dataset. Results are truncated to 2000 characters."""
    results = execute_query(query)
    output = format_results(results)
    if len(output) > 2000:
        return output[:2000] + f"\n... (truncated, {len(output)} total chars)"
    return output
```

---

## 5. Hands-On Exercise

1. **Java:** Implement `DatabaseTools` with an H2 in-memory database. Create the `searchCustomers` and `getOrderDetails` tools.
2. **Java:** Implement `GitHubTools` to search repos and list issues (public API, no auth needed).
3. **Python:** Implement file system tools and test with the research agent from Day 4.
4. **Combine:** Build an agent with 5+ tools and test it with a complex task: "Find the top Spring AI repositories on GitHub, read their README, and write a comparison report to a file."
5. **Safety:** Add input validation to the SQL query tool — test with SQL injection attempts.

---

## 6. Key Takeaways

1. **Tool descriptions are the API documentation for the LLM** — invest time in clear, example-rich descriptions.
2. **Always limit output size** — a tool that returns 100KB of text will overflow the LLM's context.
3. **Security is your responsibility** — validate inputs, sanitise queries, restrict destructive operations.
4. **Return errors as strings, don't throw exceptions** — the LLM can adapt to error messages and try differently.
5. **Test tools independently before plugging into agents** — a broken tool will confuse the agent.

---

*Day 5. Tools are the bridge between the LLM's intelligence and the real world — build them with care.*
