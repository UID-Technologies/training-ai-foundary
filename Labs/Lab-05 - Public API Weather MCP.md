# Lab 05 — Build a Weather Assistant with a Public API

## Goal

Create an independent MCP server that uses Open-Meteo's public geocoding and forecast APIs. A user or AI host can ask for a city's current weather and receive a concise, structured summary.

## Practical scenario

A travel coordinator asks, “What is the weather in Pune right now?” The MCP tool resolves the city name to coordinates, retrieves current conditions, and returns only the useful fields. No API key is required for this learning exercise. Review Open-Meteo's current terms and attribution requirements before production use.

## What you will learn

- Wrap a REST API call in an MCP tool.
- Validate input, handle missing locations, and apply an HTTP timeout.
- Keep API integration logic behind a narrow tool interface.
- Test a tool independently from any language model.

## Time and prerequisites

60–75 minutes. Python 3.11+, internet access, and a terminal. This lab does not use Lab 01's files.

## Step 1 — Create a new project

```bash
mkdir mcp-weather-lab
cd mcp-weather-lab
uv init --python 3.11
uv add "mcp[cli]>=2,<3"
```

## Step 2 — Create `server.py`

```python
import json
from urllib.error import HTTPError, URLError
from urllib.parse import urlencode
from urllib.request import Request, urlopen

from mcp.server import MCPServer

mcp = MCPServer(
    "Weather MCP",
    instructions="Use get_current_weather to retrieve current weather for a city.",
)


def get_json(url: str) -> dict:
    """Fetch JSON from a public API with a timeout and a clear error."""
    request = Request(url, headers={"User-Agent": "python-mcp-weather-lab/1.0"})
    try:
        with urlopen(request, timeout=15) as response:
            return json.loads(response.read().decode("utf-8"))
    except HTTPError as exc:
        raise RuntimeError(f"Weather provider returned HTTP {exc.code}") from exc
    except URLError as exc:
        raise RuntimeError(f"Could not reach weather provider: {exc.reason}") from exc


@mcp.tool()
def get_current_weather(city: str) -> str:
    """Get current weather conditions for a city by name."""
    city = city.strip()
    if len(city) < 2 or len(city) > 100:
        raise ValueError("Enter a city name between 2 and 100 characters.")

    geo_query = urlencode(
        {"name": city, "count": 5, "language": "en", "format": "json"}
    )
    locations = get_json(
        f"https://geocoding-api.open-meteo.com/v1/search?{geo_query}"
    ).get("results", [])
    if not locations:
        return f"No location found for '{city}'. Check the spelling or include a region."

    # Prefer an exact case-insensitive city-name match, otherwise use the first
    # geocoding result. The country is returned so the user can confirm it.
    location = next(
        (item for item in locations if item.get("name", "").casefold() == city.casefold()),
        locations[0],
    )
    forecast_query = urlencode(
        {
            "latitude": location["latitude"],
            "longitude": location["longitude"],
            "current": (
                "temperature_2m,relative_humidity_2m,apparent_temperature,"
                "precipitation,weather_code,wind_speed_10m"
            ),
            "timezone": "auto",
        }
    )
    payload = get_json(f"https://api.open-meteo.com/v1/forecast?{forecast_query}")
    current = payload.get("current")
    if not current:
        raise RuntimeError("The weather provider returned no current conditions.")

    return json.dumps(
        {
            "city": location.get("name"),
            "region": location.get("admin1"),
            "country": location.get("country"),
            "local_time": current.get("time"),
            "temperature_c": current.get("temperature_2m"),
            "feels_like_c": current.get("apparent_temperature"),
            "relative_humidity_percent": current.get("relative_humidity_2m"),
            "precipitation_mm": current.get("precipitation"),
            "wind_speed_kmh": current.get("wind_speed_10m"),
            "weather_code": current.get("weather_code"),
            "units": payload.get("current_units", {}),
            "source": "Open-Meteo",
        },
        indent=2,
    )


if __name__ == "__main__":
    mcp.run()
```

## Step 3 — Understand the API sequence

1. The user supplies a city.
2. The tool validates the name and calls Open-Meteo geocoding.
3. It selects a matching place and receives latitude/longitude.
4. It calls the forecast endpoint for current conditions in that location's timezone.
5. It returns a small JSON result. The AI host can phrase this result naturally.

The tool only performs a read. It does not accept an arbitrary URL from the model, which limits SSRF-style risks and keeps the API surface focused.

## Step 4 — Test with MCP Inspector

```bash
uv run mcp dev server.py
```

In the Inspector's **Tools** tab:

1. Select `get_current_weather`.
2. Enter `New Delhi` and run it.
3. Confirm the result contains a city, country, local time, temperature, feels-like temperature, humidity, precipitation, wind speed, weather code, and source.
4. Try `Pune`, `London`, and an intentionally misspelled city.
5. Stop the Inspector with `Ctrl+C`.

Expected results vary as weather changes. The misspelled name should return a helpful “No location found” message rather than a Python traceback.

## Step 5 — Call the tool from Python

Create `client.py`:

```python
import asyncio
import sys
from pathlib import Path

from mcp import Client
from mcp.client.stdio import StdioServerParameters


async def main() -> None:
    server_file = Path(__file__).with_name("server.py").resolve()
    async with Client(
        StdioServerParameters(command=sys.executable, args=[str(server_file)])
    ) as client:
        result = await client.call_tool(
            "get_current_weather", {"city": "Pune"}
        )
        print(result.structured_content or result.content)


if __name__ == "__main__":
    asyncio.run(main())
```

Run:

```bash
uv run python client.py
```

This verifies the MCP server and API without needing a model. An AI host can use the same tool schema and decide when to call it.

## Step 6 — Optional Microsoft Foundry connection

For the Foundry agent to call this tool, deploy the server as a remote HTTPS MCP endpoint using Streamable HTTP and configure the endpoint in the Foundry agent as described in Lab 01. The endpoint must be reachable from Foundry and protected with an authentication method supported by the current Foundry MCP integration. Use approval while learning. Do not assume a local `localhost` server is accessible to a cloud agent.

To run the same server with HTTP during local development, replace `mcp.run()` with:

```python
    mcp.run(transport="streamable-http", host="127.0.0.1", port=8000)
```

Then use MCP Inspector or an HTTP MCP client against `http://127.0.0.1:8000/mcp`. Use HTTPS and an authenticated deployment before connecting remotely.

## Step 7 — Add a business-friendly tool (challenge)

Add `get_weather_brief(city: str) -> str` that calls the existing provider logic and returns one sentence suitable for a travel coordinator, while preserving the numeric values and city/country. Refactor the API logic into a helper so the two MCP tools do not duplicate calls or validation.

## Troubleshooting

- **Network timeout** — verify internet access; the example gives each API request 15 seconds.
- **Several cities with the same name** — inspect the returned country/region; extend the tool input with a country or region filter.
- **HTTP 429 or provider errors** — retry later, reduce call volume, and check Open-Meteo's current usage terms.
- `mcp` **import error** — use `uv run` or activate the virtual environment where the dependency was installed.
- **Inspector needs Node** — use the Python client instead.

## Cleanup

Stop the server/Inspector and delete the `mcp-weather-lab` directory when finished.
