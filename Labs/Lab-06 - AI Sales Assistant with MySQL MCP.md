# Lab 06 — Build an AI Sales Assistant over MySQL with MCP

## Goal

Create a standalone read-only MCP server that lets a Microsoft Foundry agent answer business questions from a local MySQL sales database. The model does **not** receive database credentials and cannot generate or execute arbitrary SQL. It can call a small set of narrowly defined reporting tools.

## Practical scenario

A sales manager asks: “What were sales last month?”, “Which products sold the most?”, or “Show recent orders for customer CUST-1001.” The agent selects the appropriate MCP tool, the server runs a parameterized, allow-listed query, and the agent explains the returned data.

## What you will learn

- Seed a disposable MySQL database with sales data.
- Connect Python to MySQL using Connector/Python.
- Expose business-level read-only operations as MCP tools.
- Keep credentials on the server and enforce least privilege.
- Configure the MCP endpoint as a Foundry agent tool.

## Time and prerequisites

90–120 minutes. Python 3.11+, Docker Desktop/Engine, a terminal, and internet access for installation. To complete the Foundry portion, you also need an active Foundry project, a deployed chat model, and a reachable HTTPS MCP server deployment. The local database and stdio MCP server are for development; Foundry cannot reach your local machine.

## Architecture

```text
Sales manager → Microsoft Foundry agent → remote HTTPS MCP server → read-only MySQL user → sales_demo
                                             ↑
                                    MCP tools and results
```

The MCP server owns database access. The agent sees tool names, descriptions, arguments, and results; it never receives the password or a general-purpose SQL console.

## Step 1 — Create the independent project

```bash
mkdir mcp-mysql-sales-lab
cd mcp-mysql-sales-lab
uv init --python 3.11
uv add "mcp[cli]>=2,<3" mysql-connector-python
```

## Step 2 — Start a disposable MySQL instance

Run a local MySQL 8+ instance using Docker, or use an existing training database. The following command starts a local container and does not publish it to the internet:

```bash
docker run --name mcp-sales-mysql \
  -e MYSQL_ROOT_PASSWORD=LabOnly_Root_ChangeMe \
  -p 127.0.0.1:3307:3306 \
  -d mysql:8.4
```

Wait until MySQL is ready:

```bash
docker logs -f mcp-sales-mysql
```

When the log reports that MySQL is ready, press `Ctrl+C` to stop following the log; the container remains running. The lab password is for this disposable local container only. Replace it for any shared environment.

## Step 3 — Create the schema and least-privilege user

Create a file named `schema.sql` with this content:

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

Import it into the container:

```bash
docker exec -i mcp-sales-mysql mysql -uroot -pLabOnly_Root_ChangeMe < schema.sql
```

For a shared lab or production environment, use a secret manager and unique credentials. The reader account should have only `SELECT` grants on the required schema. Do not use the root account from the MCP server.

## Step 4 — Configure the Python server

Set these environment variables in the same terminal that will start the server.

**Windows PowerShell**

```powershell
$env:MYSQL_HOST = "127.0.0.1"
$env:MYSQL_PORT = "3307"
$env:MYSQL_DATABASE = "sales_demo"
$env:MYSQL_USER = "mcp_reader"
$env:MYSQL_PASSWORD = "LabOnly_Reader_ChangeMe"
```

**macOS/Linux**

```bash
export MYSQL_HOST=127.0.0.1
export MYSQL_PORT=3307
export MYSQL_DATABASE=sales_demo
export MYSQL_USER=mcp_reader
export MYSQL_PASSWORD=LabOnly_Reader_ChangeMe
```

For a reusable project, place these values in an ignored `.env` file and load them with a dotenv package. Never commit `.env` or real credentials.

## Step 5 — Create `server.py`

```python
import json
import os
from datetime import date
from decimal import Decimal

import mysql.connector
from mysql.connector import Error
from mcp.server import MCPServer

mcp = MCPServer(
    "Sales Reporting MCP",
    instructions=(
        "Answer sales questions only using the available read-only reporting tools. "
        "Do not invent values. Include the reporting period or customer in the answer."
    ),
)


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
        # Keep credentials and raw query details out of the model-visible error.
        raise RuntimeError(f"Database query failed: {exc.__class__.__name__}") from exc
    finally:
        if cursor is not None:
            cursor.close()
        if connection is not None and connection.is_connected():
            connection.close()


@mcp.tool()
def get_sales_summary(start_date: str, end_date: str) -> str:
    """Return order count and sales total for dates YYYY-MM-DD through YYYY-MM-DD, inclusive."""
    try:
        start = date.fromisoformat(start_date)
        end = date.fromisoformat(end_date)
    except ValueError as exc:
        raise ValueError("Dates must use YYYY-MM-DD format.") from exc
    if end < start:
        raise ValueError("end_date must be on or after start_date.")
    if (end - start).days > 366:
        raise ValueError("A report range cannot exceed 367 calendar days.")

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
    return json.dumps(
        {"start_date": str(start), "end_date": str(end), **rows[0]},
        default=json_default,
        indent=2,
    )


@mcp.tool()
def get_top_products(start_date: str, end_date: str, limit: int = 5) -> str:
    """List products ranked by quantity sold in the supplied inclusive date range."""
    try:
        start = date.fromisoformat(start_date)
        end = date.fromisoformat(end_date)
    except ValueError as exc:
        raise ValueError("Dates must use YYYY-MM-DD format.") from exc
    if end < start:
        raise ValueError("end_date must be on or after start_date.")
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
    return json.dumps(
        {"start_date": str(start), "end_date": str(end), "products": rows},
        default=json_default,
        indent=2,
    )


@mcp.tool()
def get_customer_orders(customer_id: str, limit: int = 5) -> str:
    """List a customer's most recent orders; customer_id looks like CUST-1001."""
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
    return json.dumps(
        {"customer_id": customer_id, "orders": rows},
        default=json_default,
        indent=2,
    )


if __name__ == "__main__":
    # Local development: stdio. A Foundry deployment needs a reachable HTTPS
    # Streamable HTTP endpoint and suitable authentication/network controls.
    mcp.run()
```

### Why these tools are safer than “ask AI to write SQL”

- Every query is authored and reviewed in the Python server.
- The user/model supplies only typed values, never SQL fragments.
- SQL values use parameter placeholders (`%s`).
- The database user is granted `SELECT` only.
- Date ranges and result limits are bounded.
- The database password exists only in the MCP server environment.

## Step 6 — Start the server and test each tool

In the terminal where you set the environment variables:

```bash
uv run mcp dev server.py
```

In the Inspector, invoke each tool:

1. `get_sales_summary`: start date `2026-10-01`, end date `2026-10-31`.
2. `get_top_products`: same dates, limit `5`.
3. `get_customer_orders`: customer ID `CUST-1001`, limit `5`.

Expected checks:

- October summary includes three orders and a total of `49292.00`.
- USB-C Dock and Wireless Keyboard tie at three units each; USB-C Dock ranks first by the revenue tie-breaker.
- Customer `CUST-1001` has orders `5003` and `5001`.
- A date range over the allowed limit or a limit of `1000` returns a validation error.

Stop the Inspector with `Ctrl+C`.

## Step 7 — Connect to Microsoft Foundry

The local stdio server is for development. A Foundry cloud agent cannot spawn your local Python process, and it cannot reach your `127.0.0.1` MySQL instance. To run the AI portion end to end:

1. **Deploy the MCP server** to an HTTPS-capable host reachable from Foundry (for example, an Azure-hosted app). Run it with Streamable HTTP, for example `mcp.run(transport="streamable-http", host="0.0.0.0", port=8000)`. Configure authentication supported by Foundry and restrict access to the intended agent/project.
2. **Place MySQL on a reachable, protected network.** In a realistic Azure setup, use managed MySQL or a protected private database and put the MCP service in a network path that can reach it. Do not open MySQL to the public internet for this lab. Use network rules/private networking, TLS where supported, Key Vault or equivalent for secrets, and a read-only DB identity.
3. **Expose only the MCP endpoint.** The Foundry agent should call the server; it should not connect directly to MySQL.
4. In the Foundry project, open the agent builder, choose **Add tool → Custom → Model Context Protocol**, enter the deployed HTTPS MCP endpoint, configure the matching authentication, and connect it. Portal labels can change; use the current official Foundry guide in the pack README.
5. Create an agent with instructions such as:
   > Answer sales questions using the available sales reporting MCP tools. Ask a clarifying question when a date range is missing. Use ISO dates when calling tools. Do not invent totals. Summarize the returned figures and state the period/customer used.
6. In the playground, ask: “How many orders and how much sales did we have from October 1 through October 31, 2026?” Review the proposed tool name and arguments. Approve the call, then verify the result matches the Inspector output.
7. Ask: “Which products sold the most in October 2026?” Verify the agent calls `get_top_products` and grounds its answer in the returned data.
8. Ask: “Show the latest orders for customer CUST-1001.” Verify the agent uses the customer-orders tool.

For a Foundry code-first agent, use Microsoft's current Python MCP integration guide (`AIProjectClient` and `MCPTool`) and its approval-response workflow. Foundry SDKs and agent APIs evolve; use the current official sample rather than copying an older blog's package versions. Begin with approval required while learning.

## Step 8 — Negative and security tests

1. Call `get_top_products` with `limit=1000`; the tool should reject it.
2. Call `get_sales_summary` with `2026-11-01` as start and `2026-10-01` as end; it should reject the inverted range.
3. Try an unknown customer ID; it should return an empty list, not disclose other customers.
4. Confirm `mcp_reader` cannot insert or delete records:

```bash
docker exec -it mcp-sales-mysql mysql -umcp_reader -pLabOnly_Reader_ChangeMe sales_demo
```

Then try `DELETE FROM orders WHERE order_id = 5001;`; MySQL should reject it. Exit with `exit`.

## Optional extension challenges

- Add a `get_sales_by_city` tool that groups sales by customer city.
- Add a `get_order_status_counts` tool for a bounded date range.
- Return typed structured objects using Pydantic models instead of serialized JSON strings.
- Add unit tests using an in-memory MCP client and a mocked database connector.
- Add an audit log with timestamp, tool name, and safe input values; never log credentials or sensitive customer data.

## Troubleshooting

- **Connection refused** — confirm the container is running (`docker ps`) and use host `127.0.0.1`, port `3307` from Python.
- **Access denied** — check the `MYSQL_USER`, `MYSQL_PASSWORD`, and that the grant was applied to `sales_demo.`*.
- **Unknown database/table** — rerun the `schema.sql` import and inspect with `SHOW TABLES;`.
- `mysql.connector` **not found** — run `uv add mysql-connector-python` in this project's directory.
- **Environment variables missing** — set them in the same terminal session that launches the server.
- **Foundry cannot connect** — confirm the endpoint is HTTPS, externally reachable or privately networked as configured, uses the right MCP path/transport, and authenticates using the selected method. `localhost` is never the remote deployment address.
- **Agent answers without calling tools** — strengthen its instruction to use the sales tools for any numeric sales answer; verify the MCP tools are visible in the agent and approval is handled.

## Cleanup

Stop the MCP server, then remove the MySQL container and its data:

```bash
docker rm -f mcp-sales-mysql
```

This removes the disposable database. Delete the project directory and any local environment file. If you deployed a remote MCP endpoint for the Foundry portion, remove that training deployment and its credentials when the exercise is complete.
