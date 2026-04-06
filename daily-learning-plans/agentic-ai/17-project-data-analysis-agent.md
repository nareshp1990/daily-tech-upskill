# Agentic AI — Day 17: Project — Data Analysis & Reporting Agent
**Track:** Agentic AI (Implementation-Focused)
**Time Target:** 1–2 hours

---

## 1. Today's Focus
**Build an Agent That Queries Databases, Analyses Data, Generates Charts, and Writes Reports**

This agent is a "data analyst on demand" — give it a question like "What were our top-selling products last quarter and how does revenue compare to the previous quarter?" and it writes SQL, executes queries, analyses results, generates visualisations, and compiles a report. Perfect for internal dashboards and ad-hoc business intelligence.

---

## 2. Architecture

```
User: "Show me revenue trends by region for Q1"
       │
       ▼
┌──────────────────┐
│ Data Agent       │
│ (LangGraph)      │
└───┬──────────────┘
    │
    ├── Tool: execute_sql(query) → runs read-only SQL
    ├── Tool: generate_chart(data, chart_type) → creates PNG/SVG
    ├── Tool: save_report(title, content) → writes Markdown
    │
    ▼
┌──────────────────┐
│ Output:          │
│ - SQL results    │
│ - Charts (PNG)   │
│ - Markdown report│
└──────────────────┘
```

---

## 3. Tools

```python
import sqlite3
import json
from pathlib import Path
from langchain_core.tools import tool

DB_PATH = "sample_data.db"

@tool
def execute_sql(query: str) -> str:
    """Execute a read-only SQL query against the analytics database.
    Only SELECT queries are allowed. Returns results as a formatted table.
    Available tables: orders(id, customer_id, product, category, amount, region, order_date),
    customers(id, name, email, signup_date, plan), products(id, name, category, price).
    Example: 'SELECT region, SUM(amount) as revenue FROM orders GROUP BY region'"""
    if not query.strip().upper().startswith("SELECT"):
        return "ERROR: Only SELECT queries are allowed."
    try:
        conn = sqlite3.connect(DB_PATH)
        conn.row_factory = sqlite3.Row
        cursor = conn.execute(query)
        rows = cursor.fetchall()
        if not rows:
            return "Query returned 0 rows."
        columns = rows[0].keys()
        header = " | ".join(columns)
        separator = " | ".join("---" for _ in columns)
        data = "\n".join(" | ".join(str(row[col]) for col in columns) for row in rows[:50])
        conn.close()
        return f"| {header} |\n| {separator} |\n| {' |\n| '.join(data.split(chr(10)))} |\n\n({len(rows)} rows)"
    except Exception as e:
        return f"SQL Error: {e}"

@tool
def generate_chart(data_json: str, chart_type: str, title: str, x_label: str, y_label: str) -> str:
    """Generate a chart from JSON data and save as PNG.
    chart_type: bar, line, pie, scatter
    data_json: JSON array like [{"label": "US", "value": 1000}, ...]
    Returns the file path of the saved chart."""
    import matplotlib
    matplotlib.use('Agg')
    import matplotlib.pyplot as plt

    data = json.loads(data_json)
    labels = [d["label"] for d in data]
    values = [d["value"] for d in data]

    fig, ax = plt.subplots(figsize=(10, 6))
    if chart_type == "bar":
        ax.bar(labels, values)
    elif chart_type == "line":
        ax.plot(labels, values, marker='o')
    elif chart_type == "pie":
        ax.pie(values, labels=labels, autopct='%1.1f%%')
    elif chart_type == "scatter":
        ax.scatter(labels, values)

    ax.set_title(title)
    if chart_type != "pie":
        ax.set_xlabel(x_label)
        ax.set_ylabel(y_label)
        plt.xticks(rotation=45, ha='right')

    output_dir = Path("reports/charts")
    output_dir.mkdir(parents=True, exist_ok=True)
    filepath = output_dir / f"{title.replace(' ', '_').lower()}.png"
    plt.tight_layout()
    plt.savefig(filepath, dpi=150)
    plt.close()
    return f"Chart saved: {filepath}"

@tool
def save_report(filename: str, content: str) -> str:
    """Save a Markdown analysis report to the reports directory."""
    output_dir = Path("reports")
    output_dir.mkdir(exist_ok=True)
    filepath = output_dir / f"{filename}.md"
    filepath.write_text(content)
    return f"Report saved: {filepath}"

@tool
def get_table_schema() -> str:
    """Get the database schema — all tables and their columns.
    Use this first to understand what data is available before writing queries."""
    conn = sqlite3.connect(DB_PATH)
    cursor = conn.execute("SELECT sql FROM sqlite_master WHERE type='table'")
    schemas = [row[0] for row in cursor.fetchall() if row[0]]
    conn.close()
    return "\n\n".join(schemas)
```

---

## 4. Agent (LangGraph)

```python
from langgraph.prebuilt import create_react_agent
from langchain_anthropic import ChatAnthropic

model = ChatAnthropic(model="claude-sonnet-4-20250514", max_tokens=4096)

data_agent = create_react_agent(
    model,
    tools=[execute_sql, generate_chart, save_report, get_table_schema],
    prompt="""You are a senior data analyst. When the user asks a business question:

    1. First, check the database schema to understand available tables.
    2. Write and execute SQL queries to get the relevant data.
    3. If comparing periods, run queries for each period.
    4. Generate charts to visualise key findings.
    5. Write a concise analysis report with:
       - Executive summary (2-3 sentences)
       - Key metrics (table format)
       - Charts (reference the saved chart files)
       - Insights and recommendations

    Rules:
    - Always use GROUP BY for aggregations.
    - Limit results to top 20 unless the user asks for more.
    - Format currency as $ with commas.
    - Include percentage changes when comparing periods."""
)

# Run
result = data_agent.invoke({
    "messages": [("user",
        "What were our top 5 products by revenue last quarter? "
        "Compare with the previous quarter and show a chart.")]
})
```

---

## 5. Sample Data Setup

```python
import sqlite3

def create_sample_db():
    conn = sqlite3.connect("sample_data.db")
    conn.executescript("""
        CREATE TABLE IF NOT EXISTS customers (
            id INTEGER PRIMARY KEY, name TEXT, email TEXT,
            signup_date TEXT, plan TEXT
        );
        CREATE TABLE IF NOT EXISTS products (
            id INTEGER PRIMARY KEY, name TEXT, category TEXT, price REAL
        );
        CREATE TABLE IF NOT EXISTS orders (
            id INTEGER PRIMARY KEY, customer_id INTEGER, product TEXT,
            category TEXT, amount REAL, region TEXT, order_date TEXT,
            FOREIGN KEY (customer_id) REFERENCES customers(id)
        );

        INSERT INTO products VALUES
            (1,'Widget Pro','Electronics',49.99),
            (2,'Gadget Plus','Electronics',89.99),
            (3,'Service Plan A','Services',29.99),
            (4,'Service Plan B','Services',59.99),
            (5,'Accessory Kit','Accessories',19.99);

        -- Generate 500 sample orders
    """)
    import random
    from datetime import datetime, timedelta
    products = [("Widget Pro","Electronics",49.99),("Gadget Plus","Electronics",89.99),
                ("Service Plan A","Services",29.99),("Service Plan B","Services",59.99),
                ("Accessory Kit","Accessories",19.99)]
    regions = ["US-East","US-West","EU","APAC"]
    for i in range(500):
        p = random.choice(products)
        qty = random.randint(1, 5)
        date = (datetime(2026,1,1) + timedelta(days=random.randint(0,180))).strftime("%Y-%m-%d")
        conn.execute("INSERT INTO orders (customer_id, product, category, amount, region, order_date) VALUES (?,?,?,?,?,?)",
            (random.randint(1,50), p[0], p[1], p[2]*qty, random.choice(regions), date))
    conn.commit()
    conn.close()
```

---

## 6. Hands-On Exercise

1. Run `create_sample_db()` to generate sample data.
2. Build the data agent with all 4 tools.
3. Test these queries:
   - "What's our total revenue by region?"
   - "Show me the monthly revenue trend with a line chart"
   - "Which product category is growing fastest?"
   - "Generate a full Q1 business report with charts"
4. Verify the agent generates correct SQL and meaningful charts.
5. Save a complete report as Markdown and review it.

---

## 7. Key Takeaways

1. **Schema discovery first** — the agent should always check `get_table_schema` before writing SQL.
2. **Read-only SQL is a hard requirement** — never let the agent modify production data.
3. **Charts make data actionable** — a number in a table is forgettable; a chart is memorable.
4. **The agent is a data analyst replacement for ad-hoc queries** — not for building dashboards.

---

*Day 17. Data without analysis is noise. An agent that analyses, visualises, and reports turns data into decisions.*
