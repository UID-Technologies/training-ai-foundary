# Lab 06 — Build an AI Sales Assistant over MySQL with MCP

> Training series: [UID Technologies — Microsoft Foundry Labs](https://github.com/UID-Technologies/training-ai-foundary/tree/master/Labs)  
> Canonical lab name in the pack: `Lab-06 - AI Sales Assistant with MySQL MCP.md`

## Goal

Create a standalone read-only MCP server that lets a Microsoft Foundry agent answer business questions from a MySQL sales database. The model does **not** receive database credentials and cannot generate or execute arbitrary SQL. It can call a small set of narrowly defined reporting tools.

Add a **Streamlit application** so end users can chat with a Foundry-backed assistant and browse the same reports in a browser.

## Practical scenario

A sales manager asks: “What were sales last month?”, “Which products sold the most?”, or “Show recent orders for customer CUST-1001.” The agent selects the appropriate MCP tool, the server runs a parameterized, allow-listed query, and the agent explains the returned data.

An analyst opens the **Reports** page to verify figures. An end user opens the **Sales assistant** page to ask questions in natural language (when Foundry is configured) or uses quick-action buttons for demo queries.

## What you will learn

- Seed a `sales_demo` database on **local MySQL** with realistic sales data.
- Connect Python to MySQL using Connector/Python.
- Separate **database access** (`db.py`), **business reports** (`reports.py`), and **MCP tools** (`server.py`).
- Expose business-level read-only operations as MCP tools.
- Keep credentials on the server and enforce least privilege (`mcp_reader`).
- Build a user-facing Streamlit UI (chat + dashboards) that reuses the same report logic.
- Optionally call a **Foundry chat deployment** from Python with function tools that mirror MCP tool names.
- Configure a remote MCP endpoint as a Foundry agent tool for production-shaped deployments.

## Time and prerequisites

**90–120 minutes.** Python 3.11+, [uv](https://docs.astral.sh/uv/), a **local MySQL 8+** server, a terminal, and internet access for installation.

This lab uses a local MySQL admin account for setup (for example `root` / `root` on port `3306`). Docker is optional (see appendix).

To complete the **Foundry cloud agent** portion, you also need an active Foundry project, a deployed chat model, and a reachable **HTTPS** MCP server deployment. Local stdio MCP and local MySQL are for development; Foundry cannot reach your `127.0.0.1` services.

## Architecture

```text
Sales manager → Microsoft Foundry agent → remote HTTPS MCP server → read-only MySQL user → sales_demo
                                             ↑
                                    MCP tools and results

End user → Streamlit (Sales assistant) → Foundry chat model (optional) → function tools → reports.py → db.py → MySQL
Analyst  → Streamlit (Reports)         → reports.py → db.py → MySQL

Developer → uv run mcp dev server.py   → server.py → reports.py → db.py → MySQL (stdio MCP)
```

The MCP server and Streamlit app **own** database access. Agents and models see tool names, descriptions, arguments, and results; they never receive the database password or a general-purpose SQL console.

## Project layout

Flat project root (no nested Python package):

| File | Purpose |
|------|---------|
| `pyproject.toml` | Dependencies; `[tool.uv] package = false` |
| `schema.sql` | Schema, seed data, `mcp_reader` user |
| `.env.example` | MySQL settings template (local `root` / `root`) |
| `.env.foundry.example` | Optional Foundry + MySQL settings |
| `db.py` | Connection helper and parameterized queries |
| `reports.py` | Validation and allow-listed SQL |
| `server.py` | MCP tools (JSON for hosts/agents) |
| `openai_tools.py` | Function-tool schemas for Streamlit / Foundry chat |
| `sales_assistant.py` | Foundry chat + tool execution loop |
| `streamlit_app.py` | Streamlit entry (navigation) |
| `ui_assistant.py` | Sales assistant chat UI |
| `ui_reports.py` | Reports dashboard UI |

---

## Step 1 — Create the independent project

Clone or create the project folder, then install dependencies.

```bash
cd mcp-mysql-sales-lab
uv sync
```

If you build from scratch:

```bash
mkdir mcp-mysql-sales-lab
cd mcp-mysql-sales-lab
uv init --python 3.11
```

Add dependencies (or copy `pyproject.toml` from this repo):

```bash
uv add "mcp[cli]>=2,<3" mysql-connector-python python-dotenv streamlit pandas
uv add azure-ai-projects azure-identity
```

Set `[tool.uv] package = false` in `pyproject.toml` so all modules live at the project root—no `src/` package folder.

---

## Step 2 — Prepare local MySQL

Use MySQL **8+** installed on your machine (Windows MySQL Installer, Homebrew, apt, etc.).

| Setting | Lab default |
|---------|-------------|
| Host | `127.0.0.1` |
| Port | `3306` |
| Admin user | `root` |
| Admin password | `root` (use your actual password if different) |

Verify connectivity:

```bash
mysql -h 127.0.0.1 -P 3306 -uroot -proot -e "SELECT VERSION();"
```

---

## Step 3 — Create the schema and least-privilege user

Create `schema.sql` in the project root (this file is included in the repo):

```sql
CREATE DATABASE IF NOT EXISTS sales_demo;
USE sales_demo;

CREATE TABLE IF NOT EXISTS customers (
    customer_id VARCHAR(20) PRIMARY KEY,
    customer_name VARCHAR(100) NOT NULL,
    city VARCHAR(80) NOT NULL
);

CREATE TABLE IF NOT EXISTS products (
    product_id INT PRIMARY KEY,
    product_name VARCHAR(120) NOT NULL,
    category VARCHAR(80) NOT NULL,
    unit_price DECIMAL(10, 2) NOT NULL
);

CREATE TABLE IF NOT EXISTS orders (
    order_id INT PRIMARY KEY,
    customer_id VARCHAR(20) NOT NULL,
    order_date DATE NOT NULL,
    status VARCHAR(30) NOT NULL,
    FOREIGN KEY (customer_id) REFERENCES customers(customer_id)
);

CREATE TABLE IF NOT EXISTS order_items (
    order_id INT NOT NULL,
    product_id INT NOT NULL,
    quantity INT NOT NULL,
    unit_price DECIMAL(10, 2) NOT NULL,
    PRIMARY KEY (order_id, product_id),
    FOREIGN KEY (order_id) REFERENCES orders(order_id),
    FOREIGN KEY (product_id) REFERENCES products(product_id)
);

INSERT INTO customers (customer_id, customer_name, city) VALUES
('CUST-1001', 'Asha Sharma', 'Pune'),
('CUST-1002', 'Rohan Mehta', 'Mumbai'),
('CUST-1003', 'Neha Iyer', 'Bengaluru')
ON DUPLICATE KEY UPDATE customer_name = VALUES(customer_name), city = VALUES(city);

INSERT INTO products (product_id, product_name, category, unit_price) VALUES
(101, 'Wireless Keyboard', 'Accessories', 2499.00),
(102, 'USB-C Dock', 'Accessories', 6999.00),
(103, '27-inch Monitor', 'Displays', 18999.00),
(104, 'Laptop Stand', 'Accessories', 1799.00)
ON DUPLICATE KEY UPDATE product_name = VALUES(product_name), category = VALUES(category), unit_price = VALUES(unit_price);

INSERT INTO orders (order_id, customer_id, order_date, status) VALUES
(5001, 'CUST-1001', '2026-10-03', 'Delivered'),
(5002, 'CUST-1002', '2026-10-12', 'Shipped'),
(5003, 'CUST-1001', '2026-10-20', 'Delivered'),
(5004, 'CUST-1003', '2026-11-01', 'Processing')
ON DUPLICATE KEY UPDATE customer_id = VALUES(customer_id), order_date = VALUES(order_date), status = VALUES(status);

INSERT INTO order_items (order_id, product_id, quantity, unit_price) VALUES
(5001, 101, 2, 2499.00),
(5001, 104, 1, 1799.00),
(5002, 103, 1, 18999.00),
(5002, 102, 1, 6999.00),
(5003, 101, 1, 2499.00),
(5003, 102, 2, 6999.00),
(5004, 103, 2, 18999.00)
ON DUPLICATE KEY UPDATE quantity = VALUES(quantity), unit_price = VALUES(unit_price);

CREATE USER IF NOT EXISTS 'mcp_reader'@'%' IDENTIFIED BY 'LabOnly_Reader_ChangeMe';
GRANT SELECT ON sales_demo.* TO 'mcp_reader'@'%';
FLUSH PRIVILEGES;
```

Import with your admin account.

**macOS / Linux**

```bash
mysql -h 127.0.0.1 -P 3306 -uroot -proot < schema.sql
```

**Windows PowerShell**

```powershell
Get-Content schema.sql | mysql -h 127.0.0.1 -P 3306 -uroot -proot
```

For production-shaped practice, run the MCP server and Streamlit with `mcp_reader` instead of `root` after import. For local development, this lab’s `.env.example` uses `root` / `root`.

---

## Step 4 — Configure environment variables

Copy templates and edit as needed:

```powershell
Copy-Item .env.example .env
```

**`.env` (MySQL — required)**

```ini
MYSQL_HOST=127.0.0.1
MYSQL_PORT=3306
MYSQL_DATABASE=sales_demo
MYSQL_USER=root
MYSQL_PASSWORD=root
```

**Optional — Foundry chat in Streamlit** (merge from `.env.foundry.example`):

```ini
FOUNDRY_PROJECT_ENDPOINT=https://<resource>.services.ai.azure.com/api/projects/<project>
FOUNDRY_MODEL_NAME=gpt-4.1-mini
```

Run `az login` (or configure another credential that `DefaultAzureCredential` can use) before using AI chat.

Never commit `.env` or real credentials.

**Windows PowerShell** (same session, if not using `.env` for MCP):

```powershell
$env:MYSQL_HOST = "127.0.0.1"
$env:MYSQL_PORT = "3306"
$env:MYSQL_DATABASE = "sales_demo"
$env:MYSQL_USER = "root"
$env:MYSQL_PASSWORD = "root"
```

**macOS/Linux**

```bash
export MYSQL_HOST=127.0.0.1
export MYSQL_PORT=3306
export MYSQL_DATABASE=sales_demo
export MYSQL_USER=root
export MYSQL_PASSWORD=root
```

---

## Step 5 — Create the Python modules

### 5a — `db.py`

```python
import os
from datetime import date
from decimal import Decimal

import mysql.connector
from mysql.connector import Error


def db_connection():
    """Open a short-lived database connection using server-side secrets."""
    required = ["MYSQL_HOST", "MYSQL_DATABASE", "MYSQL_USER", "MYSQL_PASSWORD"]
    missing = [name for name in required if not os.getenv(name)]
    if missing:
        raise RuntimeError(f"Missing required environment variables: {', '.join(missing)}")

    return mysql.connector.connect(
        host=os.environ["MYSQL_HOST"],
        port=int(os.getenv("MYSQL_PORT", "3306")),
        database=os.environ["MYSQL_DATABASE"],
        user=os.environ["MYSQL_USER"],
        password=os.environ["MYSQL_PASSWORD"],
        connection_timeout=5,
    )


def json_default(value):
    if isinstance(value, (date, Decimal)):
        return str(value)
    raise TypeError(f"Cannot serialize {type(value).__name__}")


def query_rows(sql: str, params: tuple = ()) -> list[dict]:
    """Run one server-authored query; never accept SQL from a tool argument."""
    connection = None
    cursor = None
    try:
        connection = db_connection()
        cursor = connection.cursor(dictionary=True)
        cursor.execute(sql, params)
        return cursor.fetchall()
    except Error as exc:
        raise RuntimeError(f"Database query failed: {exc.__class__.__name__}") from exc
    finally:
        if cursor is not None:
            cursor.close()
        if connection is not None and connection.is_connected():
            connection.close()
```

### 5b — `reports.py`

```python
import json
from datetime import date

from db import json_default, query_rows


def parse_date_range(start_date: str, end_date: str) -> tuple[date, date]:
    try:
        start = date.fromisoformat(start_date)
        end = date.fromisoformat(end_date)
    except ValueError as exc:
        raise ValueError("Dates must use YYYY-MM-DD format.") from exc
    if end < start:
        raise ValueError("end_date must be on or after start_date.")
    if (end - start).days > 366:
        raise ValueError("A report range cannot exceed 367 calendar days.")
    return start, end


def fetch_sales_summary(start_date: str, end_date: str) -> dict:
    start, end = parse_date_range(start_date, end_date)
    rows = query_rows(
        """
        SELECT COUNT(DISTINCT o.order_id) AS order_count,
               COALESCE(SUM(oi.quantity * oi.unit_price), 0) AS sales_total
        FROM orders o
        LEFT JOIN order_items oi ON oi.order_id = o.order_id
        WHERE o.order_date >= %s AND o.order_date <= %s
        """,
        (start, end),
    )
    return {"start_date": str(start), "end_date": str(end), **rows[0]}


def fetch_top_products(start_date: str, end_date: str, limit: int = 5) -> dict:
    start, end = parse_date_range(start_date, end_date)
    if not 1 <= limit <= 10:
        raise ValueError("limit must be between 1 and 10.")
    rows = query_rows(
        """
        SELECT p.product_id, p.product_name, p.category,
               SUM(oi.quantity) AS units_sold,
               SUM(oi.quantity * oi.unit_price) AS sales_total
        FROM order_items oi
        JOIN products p ON p.product_id = oi.product_id
        JOIN orders o ON o.order_id = oi.order_id
        WHERE o.order_date >= %s AND o.order_date <= %s
        GROUP BY p.product_id, p.product_name, p.category
        ORDER BY units_sold DESC, sales_total DESC
        LIMIT %s
        """,
        (start, end, limit),
    )
    return {"start_date": str(start), "end_date": str(end), "products": rows}


def fetch_customer_orders(customer_id: str, limit: int = 5) -> dict:
    customer_id = customer_id.strip()
    if not customer_id or len(customer_id) > 20:
        raise ValueError("Provide a valid customer ID, such as CUST-1001.")
    if not 1 <= limit <= 20:
        raise ValueError("limit must be between 1 and 20.")
    rows = query_rows(
        """
        SELECT o.order_id, o.order_date, o.status,
               COALESCE(SUM(oi.quantity * oi.unit_price), 0) AS order_total
        FROM orders o
        LEFT JOIN order_items oi ON oi.order_id = o.order_id
        WHERE o.customer_id = %s
        GROUP BY o.order_id, o.order_date, o.status
        ORDER BY o.order_date DESC, o.order_id DESC
        LIMIT %s
        """,
        (customer_id, limit),
    )
    return {"customer_id": customer_id, "orders": rows}


def dumps_report(payload: dict) -> str:
    return json.dumps(payload, default=json_default, indent=2)
```

### 5c — `server.py`

```python
from mcp.server import MCPServer

from reports import dumps_report, fetch_customer_orders, fetch_sales_summary, fetch_top_products

mcp = MCPServer(
    "Sales Reporting MCP",
    instructions=(
        "Answer sales questions only using the available read-only reporting tools. "
        "Do not invent values. Include the reporting period or customer in the answer."
    ),
)


@mcp.tool()
def get_sales_summary(start_date: str, end_date: str) -> str:
    """Return order count and sales total for dates YYYY-MM-DD through YYYY-MM-DD, inclusive."""
    return dumps_report(fetch_sales_summary(start_date, end_date))


@mcp.tool()
def get_top_products(start_date: str, end_date: str, limit: int = 5) -> str:
    """List products ranked by quantity sold in the supplied inclusive date range."""
    return dumps_report(fetch_top_products(start_date, end_date, limit))


@mcp.tool()
def get_customer_orders(customer_id: str, limit: int = 5) -> str:
    """List a customer's most recent orders; customer_id looks like CUST-1001."""
    return dumps_report(fetch_customer_orders(customer_id, limit))


if __name__ == "__main__":
    # Local development: stdio. A Foundry deployment needs a reachable HTTPS
    # Streamable HTTP endpoint and suitable authentication/network controls.
    mcp.run()
```

### Why these tools are safer than “ask AI to write SQL”

- Every query is authored and reviewed in `reports.py` / `db.py`.
- The user/model supplies only typed values, never SQL fragments.
- SQL values use parameter placeholders (`%s`).
- The database user can be limited to `SELECT` only (`mcp_reader`).
- Date ranges and result limits are bounded.
- The database password exists only in the server environment.

---

## Step 6 — Start the MCP server and test each tool

In a terminal where MySQL environment variables are set (or loaded for your shell):

```bash
uv run mcp dev server.py
```

In the **Inspector**, invoke each tool:

1. `get_sales_summary`: start date `2026-10-01`, end date `2026-10-31`.
2. `get_top_products`: same dates, limit `5`.
3. `get_customer_orders`: customer ID `CUST-1001`, limit `5`.

**Expected checks**

- October summary includes three orders and a total of `49292.00`.
- USB-C Dock and Wireless Keyboard tie at three units each; USB-C Dock ranks first by the revenue tie-breaker.
- Customer `CUST-1001` has orders `5003` and `5001`.
- A date range over the allowed limit or a limit of `1000` returns a validation error.

Stop the Inspector with `Ctrl+C`.

---

## Step 7 — Run the Streamlit user application

```bash
uv run streamlit run streamlit_app.py
```

Open the URL in the browser (usually `http://localhost:8501`).

### Sales assistant (default page)

- **Quick actions** work without Foundry: October 2026 summary, top products, orders for `CUST-1001`.
- **Chat** (optional): set `FOUNDRY_PROJECT_ENDPOINT` and `FOUNDRY_MODEL_NAME` in `.env`, then ask e.g. “How many orders from October 1 through October 31, 2026?”
- Expand **Tool calls** to verify the assistant used reporting tools with parameters—not user-supplied SQL.

### Reports page

1. Set sidebar dates **2026-10-01** → **2026-10-31**.
2. Run summary, top products, and customer orders; confirm results match Step 6.

Supporting modules: `streamlit_app.py`, `ui_assistant.py`, `ui_reports.py`, `sales_assistant.py`, `openai_tools.py` (included in this repo).

Stop Streamlit with `Ctrl+C`.

---

## Step 8 — Connect to Microsoft Foundry (cloud agent + remote MCP)

The local stdio server is for development. A Foundry cloud agent cannot spawn your local Python process, and it cannot reach your `127.0.0.1` MySQL instance. To run the AI portion end to end:

1. **Deploy the MCP server** to an HTTPS-capable host reachable from Foundry. Run with Streamable HTTP, for example `mcp.run(transport="streamable-http", host="0.0.0.0", port=8000)`. Configure authentication and network restrictions.
2. **Place MySQL** on a reachable, protected network (managed MySQL, private networking, Key Vault secrets, read-only DB identity). Do not expose MySQL to the public internet.
3. **Expose only the MCP endpoint** to the agent—not the database.
4. In the Foundry project, choose **Add tool → Custom → Model Context Protocol**, enter the HTTPS MCP URL, and configure authentication. Portal labels may change; use the [current Foundry MCP guide](https://learn.microsoft.com/en-us/azure/foundry/agents/how-to/tools/model-context-protocol).
5. Create an agent with instructions such as:

   > Answer sales questions using the available sales reporting MCP tools. Ask a clarifying question when a date range is missing. Use ISO dates when calling tools. Do not invent totals. Summarize the returned figures and state the period/customer used.

6. In the playground, verify:
   - “How many orders and how much sales did we have from October 1 through October 31, 2026?” → `get_sales_summary`
   - “Which products sold the most in October 2026?” → `get_top_products`
   - “Show the latest orders for customer CUST-1001.” → `get_customer_orders`

For a **code-first** agent, use `AIProjectClient`, `PromptAgentDefinition`, and `MCPTool` with `server_url` pointing at your deployed sales MCP. Handle `mcp_approval_request` with `McpApprovalResponse`. Use Microsoft’s current Python samples and package versions; start with **approval required**.

**Any Foundry chat model:** set the deployment name in `PromptAgentDefinition(model=...)` or in Streamlit’s `FOUNDRY_MODEL_NAME`.

---

## Step 9 — Negative and security tests

1. Call `get_top_products` with `limit=1000`; the tool should reject it.
2. Call `get_sales_summary` with `2026-11-01` as start and `2026-10-01` as end; reject the inverted range.
3. Use an unknown customer ID; return an empty `orders` list without disclosing other customers.
4. Confirm `mcp_reader` cannot delete data:

```bash
mysql -h 127.0.0.1 -P 3306 -umcp_reader -pLabOnly_Reader_ChangeMe sales_demo
```

```sql
DELETE FROM orders WHERE order_id = 5001;
```

MySQL should reject the statement. Exit with `exit`.

---

## Checkpoint questions

1. Where do database credentials live—the Foundry agent, the MCP server, or the browser?
2. Why should tools accept **dates** and **customer IDs** instead of raw SQL strings?
3. What is the difference between the Streamlit **Reports** page and the **Sales assistant** page?
4. Why must Foundry use an **HTTPS** MCP URL instead of `uv run mcp dev server.py` on your laptop?
5. What three MCP tools exist in `server.py`, and what does each return?

---

## Optional extension challenges

- Add a `get_sales_by_city` tool that groups sales by customer city.
- Add a `get_order_status_counts` tool for a bounded date range.
- Return typed structured objects using Pydantic models instead of JSON strings.
- Add unit tests for `reports.py` with a mocked `query_rows`.
- Add an audit log with timestamp, tool name, and safe inputs (never log passwords).
- Deploy MCP to Azure and attach it with `MCPTool` in a Foundry Prompt Agent sample script.

---

## Troubleshooting

- **Connection refused** — confirm MySQL is running; host `127.0.0.1`, port `3306` (or your `.env` port).
- **Access denied** — check `MYSQL_USER`, `MYSQL_PASSWORD`; re-import `schema.sql` if `mcp_reader` is missing.
- **Unknown database/table** — re-run `schema.sql`; verify with `SHOW TABLES` in `sales_demo`.
- **`mysql.connector` not found** — run `uv sync` in the project directory.
- **Environment variables missing** — use `.env` for Streamlit; export vars in the same terminal as `mcp dev`.
- **Streamlit cannot connect** — `.env` in project root; MySQL service running.
- **Foundry chat fails in UI** — `az login`; verify `FOUNDRY_PROJECT_ENDPOINT` and `FOUNDRY_MODEL_NAME`.
- **Foundry cannot connect to MCP** — HTTPS, reachable URL, correct transport/auth; not `localhost` from the cloud.
- **Agent answers without tools** — strengthen instructions; require tool use; handle MCP approval.
- **MCP import errors** — use `mcp[cli]>=2,<3`; `from mcp.server import MCPServer`.

---

## Cleanup

Stop Streamlit and any MCP processes. Remove lab data from local MySQL if desired:

```sql
DROP DATABASE IF EXISTS sales_demo;
```

Delete local `.env` files with secrets. Remove any remote MCP deployment and credentials used for Foundry testing.

---

## Optional appendix — Docker instead of local MySQL

If you prefer an isolated MySQL container:

```bash
docker run --name mcp-sales-mysql \
  -e MYSQL_ROOT_PASSWORD=LabOnly_Root_ChangeMe \
  -p 127.0.0.1:3307:3306 \
  -d mysql:8.4
```

Import: `docker exec -i mcp-sales-mysql mysql -uroot -pLabOnly_Root_ChangeMe < schema.sql`  
Set `MYSQL_PORT=3307` in `.env`. Cleanup: `docker rm -f mcp-sales-mysql`.

Docker is **not** required for this repository’s default path.
