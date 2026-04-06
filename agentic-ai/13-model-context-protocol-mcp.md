# Agentic AI — Day 13: Model Context Protocol (MCP) — Universal Tool Integration
**Track:** Agentic AI (Implementation-Focused)
**Time Target:** 1–2 hours

---

## 1. Today's Focus
**MCP — The USB-C of AI Tool Integration**

Every AI framework (Spring AI, LangChain, CrewAI) has its own tool format. MCP (Model Context Protocol) by Anthropic standardises how AI applications connect to tools and data sources. Think of it as a universal adapter: build a tool server once, connect it from any MCP-compatible client. Today you build MCP servers (tool providers) and connect them to agents.

---

## 2. MCP Architecture

```
┌─────────────┐          ┌─────────────────┐
│  AI Agent   │          │  MCP Server     │
│  (Client)   │◄── MCP ─►│  (Tool Provider)│
│             │  protocol │                 │
│  Spring AI  │          │  - resources    │
│  or Claude  │          │  - tools        │
│  or LangChain│         │  - prompts      │
└─────────────┘          └─────────────────┘

MCP Server exposes:
  - Resources: data the agent can read (files, DB records, API data)
  - Tools: functions the agent can call (CRUD, search, compute)
  - Prompts: reusable prompt templates

Transport: stdio (local) or HTTP+SSE (remote)
```

---

## 3. Building an MCP Server (Python)

```python
from mcp.server import Server
from mcp.types import Tool, TextContent
import json

app = Server("database-tools")

# --- Define tools ---
@app.tool()
async def search_customers(query: str) -> str:
    """Search for customers by name or email.
    Returns matching customer records with id, name, email, and signup date."""
    # In production: query your actual database
    customers = [
        {"id": 1, "name": "Alice Johnson", "email": "alice@example.com", "signup": "2025-01-15"},
        {"id": 2, "name": "Bob Smith", "email": "bob@example.com", "signup": "2025-03-22"},
    ]
    results = [c for c in customers if query.lower() in c["name"].lower() or query.lower() in c["email"].lower()]
    return json.dumps(results, indent=2)

@app.tool()
async def get_order_history(customer_id: int, limit: int = 10) -> str:
    """Get order history for a customer.
    Returns recent orders with id, date, total, and status."""
    # Mock data
    orders = [
        {"id": 101, "date": "2025-12-01", "total": 59.99, "status": "delivered"},
        {"id": 102, "date": "2025-12-15", "total": 124.50, "status": "shipped"},
    ]
    return json.dumps(orders[:limit], indent=2)

@app.tool()
async def create_support_ticket(customer_id: int, subject: str, description: str, priority: str = "medium") -> str:
    """Create a customer support ticket.
    Priority: low, medium, high, urgent."""
    ticket_id = f"TICKET-{customer_id}-{hash(subject) % 10000}"
    return json.dumps({
        "ticket_id": ticket_id,
        "customer_id": customer_id,
        "subject": subject,
        "priority": priority,
        "status": "open",
        "message": f"Ticket {ticket_id} created successfully."
    })

# --- Define resources ---
@app.resource("config://support-policies")
async def get_support_policies() -> str:
    """Company support policies and SLA guidelines."""
    return """
    ## Support Policies
    - Response time SLA: urgent=1h, high=4h, medium=24h, low=72h
    - Refund policy: full refund within 30 days, 50% within 60 days
    - Escalation: 3+ tickets from same customer → auto-escalate to manager
    """

# --- Run server ---
if __name__ == "__main__":
    import asyncio
    from mcp.server.stdio import stdio_server

    async def main():
        async with stdio_server() as (read_stream, write_stream):
            await app.run(read_stream, write_stream)

    asyncio.run(main())
```

### Running the MCP Server

```bash
# Install MCP SDK
pip install mcp

# Run as stdio server (for local clients like Claude Desktop)
python database_tools_server.py

# Or run as HTTP server for remote access
# python -m mcp.server.http database_tools_server.py --port 8080
```

---

## 4. Building an MCP Server (Java / Spring Boot)

```java
@SpringBootApplication
public class McpServerApplication {
    public static void main(String[] args) {
        SpringApplication.run(McpServerApplication.class, args);
    }
}

@Component
public class CustomerToolProvider {

    @Bean
    public List<McpTool> mcpTools() {
        return List.of(
            McpTool.builder()
                .name("search_customers")
                .description("Search for customers by name or email")
                .inputSchema(Map.of(
                    "type", "object",
                    "properties", Map.of(
                        "query", Map.of("type", "string", "description", "Search term")
                    ),
                    "required", List.of("query")
                ))
                .handler(this::searchCustomers)
                .build()
        );
    }

    private String searchCustomers(Map<String, Object> params) {
        String query = (String) params.get("query");
        List<Customer> results = customerRepo.searchByNameOrEmail(query);
        return objectMapper.writeValueAsString(results);
    }
}
```

---

## 5. Connecting MCP to Your Agent (Python Client)

```python
from mcp import ClientSession, StdioServerParameters
from mcp.client.stdio import stdio_client
from langchain_anthropic import ChatAnthropic
from langchain_core.tools import StructuredTool

async def create_agent_with_mcp():
    # Connect to MCP server
    server_params = StdioServerParameters(
        command="python",
        args=["database_tools_server.py"]
    )

    async with stdio_client(server_params) as (read, write):
        async with ClientSession(read, write) as session:
            await session.initialize()

            # List available tools from MCP server
            tools_result = await session.list_tools()

            # Convert MCP tools to LangChain tools
            langchain_tools = []
            for mcp_tool in tools_result.tools:
                async def make_caller(tool_name):
                    async def call_tool(**kwargs):
                        result = await session.call_tool(tool_name, kwargs)
                        return result.content[0].text
                    return call_tool

                caller = await make_caller(mcp_tool.name)
                langchain_tools.append(StructuredTool(
                    name=mcp_tool.name,
                    description=mcp_tool.description,
                    func=caller,
                    args_schema=mcp_tool.inputSchema
                ))

            # Create agent with MCP tools
            model = ChatAnthropic(model="claude-sonnet-4-20250514")
            from langgraph.prebuilt import create_react_agent
            agent = create_react_agent(model, tools=langchain_tools)

            # Use the agent
            result = agent.invoke({
                "messages": [("user", "Find customer Alice and show her recent orders")]
            })
            print(result["messages"][-1].content)
```

---

## 6. MCP with Claude Desktop (Quickest Path)

```json
// ~/Library/Application Support/Claude/claude_desktop_config.json
{
  "mcpServers": {
    "database-tools": {
      "command": "python",
      "args": ["/path/to/database_tools_server.py"]
    },
    "github-tools": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": { "GITHUB_TOKEN": "ghp_xxx" }
    }
  }
}
```

---

## 7. Hands-On Exercise

1. Build a Python MCP server with 3 tools (search, get details, create ticket).
2. Test it locally with the MCP Inspector: `npx @modelcontextprotocol/inspector python database_tools_server.py`
3. Connect the MCP server to a LangGraph agent and test end-to-end.
4. Add a `resource` to your MCP server (e.g., support policies) and read it from the agent.
5. (Optional) Configure Claude Desktop to use your MCP server.

---

## 8. Key Takeaways

1. **MCP standardises tool integration** — build once, connect from any MCP-compatible client.
2. **Tools + Resources + Prompts** — MCP exposes all three, not just function calls.
3. **stdio for local, HTTP+SSE for remote** — choose transport based on deployment.
4. **MCP servers are microservices for AI** — each server is a focused tool provider.

---

*Day 13. Standards beat custom integrations — MCP is how the AI tool ecosystem scales.*
