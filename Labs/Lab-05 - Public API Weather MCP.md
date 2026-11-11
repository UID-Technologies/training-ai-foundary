# Lab 05 — Extended: Build a Weather Assistant with Streamlit UI + MCP Server

> **Course pack:** [UID-Technologies/training-ai-foundary — Labs](https://github.com/UID-Technologies/training-ai-foundary/tree/master/Labs)  
> **Prerequisite lab:** [Lab 05 — Public API Weather MCP](https://github.com/UID-Technologies/training-ai-foundary/blob/master/Labs/Lab-05%20-%20Public%20API%20Weather%20MCP.md) (standalone MCP server + CLI client)

## Goal

Extend the Weather MCP lab into a complete application where users enter a city in a **Streamlit** web UI, submit a weather request, and receive live weather from the MCP server.

Streamlit acts as an **MCP client** — it does **not** call Open-Meteo directly. This shows how a real frontend can use MCP tools **without** an LLM or AI agent.

## Practical scenario

A travel desk employee opens an internal weather page, types **Pune**, and clicks **Get Weather**. The browser talks to Streamlit; Streamlit starts the MCP server over stdio, calls `get_current_weather`, and renders a dashboard. Operations can verify the same flow with `client.py` without opening the browser.

## What you will learn

- Separate **UI**, **MCP client**, **MCP server**, and **external API** into clear layers.
- Use the Python MCP SDK (`MCPServer`, `ClientSession`, `stdio_client`) on MCP **2.x**.
- Expose multiple tools from one server (`get_current_weather`, `get_weather_brief`).
- Build a Streamlit form that invokes MCP tools asynchronously.
- Test end-to-end behavior including validation and error paths.

## Time and prerequisites

**90–120 minutes.** Python **3.11+**, internet access, and a terminal.  
**Optional:** complete base Lab 05 first (same `server.py` concepts).

Install **uv** (recommended) or use `venv` + `pip`. No Azure account, API key, or paid weather API is required.

## Application architecture

```text
Streamlit Web UI (app.py)
        │  city + response type
        ▼
MCP Python client (mcp_client.py) · stdio transport
        │  list_tools / call_tool
        ▼
Python MCP Server (server.py)
        │  get_current_weather · get_weather_brief
        ▼
Open-Meteo (geocoding + forecast) · HTTPS · JSON
```

| Layer             | Technology                          |
| ----------------- | ----------------------------------- |
| Frontend          | Streamlit                           |
| MCP client        | Python MCP SDK (`ClientSession`)    |
| MCP server        | `MCPServer` (MCP SDK v2)            |
| Transport         | stdio (local development)           |
| External provider | Open-Meteo                          |
| Data format       | JSON                                |
| AI model          | **Not required**                    |

**Important:** With stdio, the MCP client **launches** `server.py` as a subprocess. You do not need a separate terminal running the server for each UI request (this lab starts a new session per request).

## Project structure

```text
mcp-weather-lab/
│
├── server.py           # MCP server and weather tools
├── mcp_client.py       # Reusable MCP client
├── app.py              # Streamlit UI
├── client.py           # CLI test (no Streamlit)
├── pyproject.toml
├── lab.md              # This guide
├── README.md
└── .venv/
```

## Step 1 — Create project and install dependencies

Open PowerShell or a terminal:

```powershell
mkdir mcp-weather-lab
cd mcp-weather-lab

uv init --python 3.11
uv add "mcp[cli]>=2,<3" streamlit
```

If `uv` is not on PATH, install it from [https://docs.astral.sh/uv/getting-started/installation/](https://docs.astral.sh/uv/getting-started/installation/) and ensure `C:\Users\<you>\.local\bin` is on your user **Path**.

**Without uv:**

```powershell
python -m venv .venv
.\.venv\Scripts\Activate.ps1
python -m pip install "mcp[cli]>=2,<3" streamlit
```

On Linux/macOS: `source .venv/bin/activate`.

## Step 2 — Update the MCP server (`server.py`)

Implement shared `fetch_weather(city)` for geocoding + current conditions, then expose two tools:

| Tool                   | Returns                                              |
| ---------------------- | ---------------------------------------------------- |
| `get_current_weather`  | JSON string with structured fields (temp, wind, …)   |
| `get_weather_brief`    | One travel-friendly sentence (WMO weather code text) |

Design notes:

- `get_json()` uses `urllib` with a **15-second timeout** and a clear `User-Agent`.
- Validation: empty city, length 2–100 characters, unknown location.
- `ValueError` / `RuntimeError` from `fetch_weather` are converted to user-facing messages inside tools (so Streamlit gets warnings, not raw tracebacks).
- Entry point: `mcp.run()` for stdio (do not `print()` to stdout — it is the MCP wire).

Use the `server.py` in this repository as the reference implementation.

### Walk through the server flow

1. User supplies a city via MCP tool arguments.
2. Geocoding API resolves name → coordinates (prefer exact name match).
3. Forecast API returns `current` block for that timezone.
4. Tools return JSON or a brief string to the MCP client.

## Step 3 — Create the MCP client (`mcp_client.py`)

This module is the bridge between Streamlit and the server:

```python
from mcp import ClientSession, StdioServerParameters
from mcp.client.stdio import stdio_client
```

`call_weather_tool(city, tool_name)` should:

1. Build `StdioServerParameters(command=sys.executable, args=[path/to/server.py])`.
2. Open `stdio_client`, create `ClientSession`, `await session.initialize()`.
3. `await session.list_tools()` (discovery).
4. `await session.call_tool(tool_name, {"city": city})`.
5. Parse text content; for `get_current_weather`, `json.loads` and surface `{"error": ...}` as `ValueError`.

See `mcp_client.py` in this repo for the full implementation.

## Step 4 — Build the Streamlit UI (`app.py`)

The UI should include:

- Page title **MCP Weather Assistant** and caption: `Streamlit → MCP Client → MCP Server → Open-Meteo`
- Form: city text input (default `Pune`), response type radio (**Detailed Weather** / **Travel Weather Brief**)
- **Get Weather** button
- On success (detailed): metric cards, optional `st.map` from latitude/longitude, expandable raw JSON
- On success (brief): display the sentence in a success callout
- On failure: `st.warning` for validation/unknown city, `st.error` for network/runtime errors

Streamlit is synchronous; wrap the async MCP call:

```python
import asyncio

def run_mcp(city: str, tool_name: str):
    return asyncio.run(call_weather_tool(city, tool_name))
```

## Step 5 — Run the complete application

From the project directory:

```powershell
uv run streamlit run app.py
```

Or with an activated venv:

```powershell
streamlit run app.py
```

Open [http://localhost:8501](http://localhost:8501), enter a city, choose response type, and click **Get Weather**.

## Step 6 — Expected UI behavior

After a successful **Detailed Weather** request you should see:

- Location heading (city, country, region caption)
- Metrics: temperature, feels like, humidity, wind, precipitation, weather code
- A map pin at the resolved coordinates
- Expandable **Raw JSON from MCP tool**

**Travel Weather Brief** shows a single summary line (with an umbrella hint when precipitation &gt; 0).

Values are **live** from Open-Meteo via MCP (not hard-coded).

## Step 7 — Test the application

| Test | Input                | Expected behavior                         |
| ---- | -------------------- | ----------------------------------------- |
| 1    | Pune                 | Detailed metrics and map                  |
| 2    | New Delhi            | Weather for New Delhi, India              |
| 3    | London               | Weather with country shown                |
| 4    | Invalid city name    | Warning: location not found               |
| 5    | Empty input          | Warning: enter a city name                |
| 6    | Pune + Travel Brief  | One-sentence summary                      |
| 7    | Disconnect internet  | Connection / provider error message       |

### Verify Streamlit uses MCP (not Open-Meteo directly)

1. Temporarily rename `server.py` to `server_backup.py`.
2. Submit a weather request in Streamlit.
3. Expect an error — the client cannot launch the server.
4. Restore the filename and retry.

### Optional: MCP Inspector (base Lab 05)

You can still inspect tools in isolation:

```powershell
uv run mcp dev server.py
```

Use the **Tools** tab to run `get_current_weather` and `get_weather_brief`. Stop with `Ctrl+C`.

## Step 8 — CLI test (`client.py`)

Independent test without Streamlit:

```python
import asyncio
import json
from mcp_client import call_weather_tool

async def main() -> None:
    result = await call_weather_tool("Pune", "get_current_weather")
    print(json.dumps(result, indent=2))
    brief = await call_weather_tool("Pune", "get_weather_brief")
    print("\nTravel brief:")
    print(brief)

if __name__ == "__main__":
    asyncio.run(main())
```

Run:

```powershell
uv run python client.py
```

## Step 9 — Execution flow (checkpoint)

1. **User** enters a city and clicks **Get Weather**.
2. **Streamlit** calls `call_weather_tool(city, tool_name)`.
3. **MCP client** spawns `server.py`, initializes session, discovers tools, invokes the tool.
4. **MCP server** geocodes the city and calls Open-Meteo.
5. **Server** returns tool output to the client.
6. **Streamlit** renders metrics, map, or brief text.

## Checkpoint questions

1. Why should `app.py` avoid importing `fetch_weather` from `server.py`?
2. What transport does this lab use, and who starts the server process?
3. What is the difference between `get_current_weather` and `get_weather_brief` for the UI?
4. Why must debugging output avoid `print()` on stdout in stdio mode?

## Further enhancements

| Current application              | Future AI weather assistant              |
| -------------------------------- | ---------------------------------------- |
| User enters a city               | Natural-language question                |
| User selects MCP tool in UI      | Agent selects tool                       |
| Structured JSON / brief          | Natural-language explanation             |
| Streamlit + MCP                  | Streamlit + LLM + MCP                    |
| No model required                | LLM integration                          |

Possible next steps on the server side:

- `get_weather_forecast` for multi-day outlook
- Weather-code → human-readable labels in the detailed view
- City disambiguation (country/region parameter)
- Caching and rate limiting for Open-Meteo

**Production note:** stdio + one subprocess per request is fine for learning. For multiple users, use a **persistent** MCP service over **Streamable HTTP** with authentication instead of spawning `server.py` on every click.

Example local HTTP transport (comparison only — not used by this Streamlit lab):

```python
if __name__ == "__main__":
    mcp.run(transport="streamable-http", host="127.0.0.1", port=8000)
```

See [Lab 04](https://github.com/UID-Technologies/training-ai-foundary/blob/master/Labs/Lab-04%20-%20First%20MCP%20Server%20and%20Foundry.md) and base Lab 05 for Foundry remote endpoint guidance.

## Troubleshooting

| Issue | What to try |
| ----- | ----------- |
| `uv` not recognized | Install uv; add `%USERPROFILE%\.local\bin` to Path; open a new terminal |
| `ModuleNotFoundError: mcp` | Run commands with `uv run` or activate `.venv` |
| `ModuleNotFoundError: streamlit` | `uv add streamlit` or `pip install streamlit` |
| Streamlit hangs on first request | Normal — cold-start spawns MCP server; check firewall for Open-Meteo HTTPS |
| Network timeout | Verify internet; geocoding/forecast use 15s timeout |
| Several cities same name | Check country/region in JSON; extend tool with optional region |
| Generic MCP tool error | Ensure server tools catch `ValueError`/`RuntimeError` and return messages |
| Inspector needs Node | Use `client.py` or Streamlit instead |
| Stdio garbled / broken MCP | Remove `print()` from `server.py`; use `logging` instead |

## Cleanup

1. Stop Streamlit (`Ctrl+C` in the terminal).
2. Delete the `mcp-weather-lab` folder when finished, or keep it for Lab 06+ MCP patterns.

## Quick reference — one command start

```powershell
uv run streamlit run app.py
```

No Azure account, AI model, or paid weather API is required for this exercise.
