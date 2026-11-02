# Lab 04 — Build Your First MCP Server and Python Client

## Goal

Build a tiny MCP server in Python, connect to it from a separate Python client, and inspect the three MCP primitives: **tools**, **resources**, and **prompts**. Then understand what must change before the same server can be connected to Microsoft Foundry.

## What you will learn

- MCP server, client, and host roles.
- How typed Python functions become discoverable MCP tools.
- How a resource differs from a tool and a prompt.
- How a Python application can launch and call a local server over stdio.
- Why Foundry needs a remote HTTPS MCP endpoint.

## Time and prerequisites

45–60 minutes. Use Python 3.11+, a terminal, and an internet connection for installation. No API key or database is needed.

## The request flow

```text
User → Host/AI application → MCP client → MCP server → tool function
                                              ← result ←
```

The MCP server does not call the language model. It advertises capabilities and executes the functions a host/client asks it to run. A host can be a chat application, an IDE, or an agent service.

## Step 1 — Create an isolated project

```bash
mkdir mcp-first-lab
cd mcp-first-lab
uv init --python 3.11
uv add "mcp[cli]>=2,<3"
```

If you do not use `uv`, create and activate a virtual environment, then install the same SDK:

```bash
python -m venv .venv
# Windows PowerShell
.venv\Scripts\Activate.ps1
# macOS/Linux: source .venv/bin/activate
python -m pip install "mcp[cli]>=2,<3"
```

The `[cli]` extra supplies the `mcp` development command. `mcp dev` also needs Node.js/npm on PATH because the MCP Inspector is a web application.

## Step 2 — Create `server.py`

```python
from mcp.server import MCPServer

mcp = MCPServer(
    "First MCP Lab",
    instructions="A small training server. Use the greet tool for a greeting.",
)


@mcp.tool()
def greet(name: str) -> str:
    """Return a friendly greeting for the supplied name."""
    cleaned_name = name.strip()
    if not cleaned_name:
        raise ValueError("name must not be empty")
    return f"Hello, {cleaned_name}! Your Python MCP tool is working."


@mcp.resource("lab://about")
def about_lab() -> str:
    """Explain what this training MCP server demonstrates."""
    return (
        "This server demonstrates one tool (greet), one fixed resource "
        "(lab://about), and one reusable prompt (explain_mcp)."
    )


@mcp.prompt()
def explain_mcp(topic: str = "MCP") -> str:
    """Create a short learning prompt about a topic."""
    return f"Explain {topic} in simple words and give one practical example."


if __name__ == "__main__":
    # Default transport is stdio, which is ideal for a local host launching
    # this process. Do not print debugging text to stdout; stdout is the wire.
    mcp.run()
```

### Walk through the important lines

1. `MCPServer(...)` names the server and gives the host a short instruction.
2. `@mcp.tool()` registers a function the model/host may choose to call. The type hint and docstring provide its input schema and description.
3. `@mcp.resource("lab://about")` registers read-only context the application can load. It does not perform an action.
4. `@mcp.prompt()` registers a reusable message template a user can select.
5. `mcp.run()` defaults to stdio. The host starts the server process and exchanges MCP messages over standard input/output.

## Step 3 — Inspect the server

Run this from the project directory:

```bash
uv run mcp dev server.py
```

Open the Inspector URL printed in the terminal.

1. In **Tools**, select `greet`, provide `name = Varun`, and run it. Expect a greeting.
2. In **Resources**, read `lab://about`. Expect the server description.
3. In **Prompts**, select `explain_mcp`, keep or change the topic, and inspect the generated message.
4. Stop the Inspector with `Ctrl+C`.

If you do not have Node.js installed, skip this step and proceed to the Python client.

## Step 4 — Create `client.py`

This is a simple Python application acting as an MCP host/client. It launches `server.py` as a child process, discovers its tools, calls one tool, and reads a resource.

```python
import asyncio
import sys
from pathlib import Path

from mcp import Client
from mcp.client.stdio import StdioServerParameters
from pydantic import AnyUrl


async def main() -> None:
    server_file = Path(__file__).with_name("server.py").resolve()
    server = StdioServerParameters(
        command=sys.executable,
        args=[str(server_file)],
    )

    async with Client(server) as client:
        print(f"Connected to: {client.server_info.name}")

        tool_list = await client.list_tools()
        print("Discovered tools:", [tool.name for tool in tool_list.tools])

        result = await client.call_tool("greet", {"name": "Varun"})
        print("Tool result:", result.structured_content or result.content)

        resource_result = await client.read_resource(AnyUrl("lab://about"))
        print("Resource result:", resource_result)


if __name__ == "__main__":
    asyncio.run(main())
```

Run it:

```bash
uv run python client.py
```

Expected output includes the server name, `greet` among discovered tools, and a greeting. Depending on the SDK's result rendering, the returned text may appear in `structured_content` or the content blocks.

## Step 5 — Trace one call

When `client.py` calls `greet`, the sequence is:

1. The client launches the server process.
2. MCP initializes the connection and exchanges server capabilities.
3. `list_tools()` asks the server what tools are available.
4. `call_tool()` sends the tool name and JSON arguments.
5. The Python function runs and the result is returned to the client.
6. Exiting `async with` closes the process/connection.

The same protocol lets another host use the server. You do not need to rewrite the function for each host.

## Step 6 — Relate this to a Microsoft Foundry project

Foundry Agent Service can connect to an MCP server as an agent tool, but its cloud service needs a **remote MCP server endpoint** that it can reach. A local stdio process and `http://localhost` are not reachable from Foundry.

1. Keep the local stdio setup above for development and Inspector testing.
2. For Foundry, run/deploy the server with Streamable HTTP and give it a secure HTTPS URL. For example, the SDK supports `mcp.run(transport="streamable-http", host="0.0.0.0", port=8000)`. Do not expose a database or sensitive tool without authentication and network controls.
3. In the Foundry project, open the agent builder, choose **Add tool → Custom → Model Context Protocol**, enter the remote MCP endpoint, configure the endpoint's supported authentication, and connect it. Portal wording may change; follow the current Foundry MCP guide linked in the pack README.
4. Add the tool to an agent and test a prompt that should call `greet`. Review any approval request before allowing a tool call.

For code-based agents, Microsoft's current Python examples use the Azure AI Projects SDK and an `MCPTool` referencing `server_url`; the MCP server still needs to be remote. Follow the official Foundry guide for the current package versions and approval flow. The local client in Step 4 is intentionally a protocol learning harness, not the Foundry agent itself.

### Optional: switch this server to HTTP for a local transport comparison

Replace the last line of `server.py` with:

```python
    mcp.run(transport="streamable-http", host="127.0.0.1", port=8000)
```

Start it with `uv run python server.py`. In a second file, connect using:

```python
import asyncio
from mcp import Client


async def main() -> None:
    async with Client("http://127.0.0.1:8000/mcp") as client:
        result = await client.call_tool("greet", {"name": "HTTP learner"})
        print(result.structured_content or result.content)


asyncio.run(main())
```

Stop the server with `Ctrl+C`. Local HTTP is useful for learning the transport; it does not make the endpoint reachable from Foundry.

## Checkpoint questions

1. Which component contains the business function: host, client, or server?
2. Which primitive is appropriate for a function that looks up current information?
3. Which primitive is suitable for static reference content?
4. Why can the Foundry cloud agent not connect to this machine's `localhost`?

## Troubleshooting

- `ModuleNotFoundError: mcp` — activate the environment or use `uv run` so commands use the project environment.
- **Inspector does not start** — install Node.js/npm and ensure `npx` is on PATH, or skip Inspector and run `client.py`.
- **No tools found** — confirm decorators are on module-level functions and the server file imports without errors.
- **Unexpected text / connection error** — remove top-level `print()` calls. For stdio, stdout carries MCP protocol data; use logging for diagnostics.
- **Import errors from an older tutorial** — this lab uses SDK v2 (`from mcp.server import MCPServer`). Do not copy v1 imports into the same environment.

## Cleanup

Stop all running commands and remove the `mcp-first-lab` directory when you no longer need it.
