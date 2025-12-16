# Example: Weather Tools

A simple example demonstrating how to create and use tools in AEnvironment.

## Overview

This example creates a weather service environment with tools for:

- Getting current weather
- Getting weather forecasts
- Searching weather history

## Project Structure

```bash
weather-env/
├── config.json
├── Dockerfile
├── requirements.txt
└── src/
    └── tools.py
```

## Implementation

### config.json

```json
{
  "name": "weather-env",
  "version": "1.0.0",
  "description": "Weather information tools",
  "tags": ["weather", "api", "example"],
  "deployConfig": {
    "cpu": "500m",
    "memory": "512Mi",
    "os": "linux"
  }
}
```

### requirements.txt

```bash
aenvironment>=0.2.0
httpx>=0.24.0
```

### Dockerfile

```dockerfile
FROM python:3.12-slim

WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY src/ ./src/

CMD ["python", "-m", "aenv.server", "src/tools.py", "--port", "8081"]
```

### src/tools.py

```python
"""Weather tools for AEnvironment."""

from typing import Optional
from datetime import datetime, timedelta
import httpx

from aenv import register_tool


# Simulated weather data (replace with real API in production)
WEATHER_DATA = {
    "beijing": {"temp": 22, "condition": "sunny", "humidity": 45},
    "shanghai": {"temp": 25, "condition": "cloudy", "humidity": 65},
    "shenzhen": {"temp": 28, "condition": "rainy", "humidity": 80},
    "hangzhou": {"temp": 23, "condition": "partly_cloudy", "humidity": 55},
}


@register_tool
def get_weather(
    location: str,
    unit: str = "celsius"
) -> dict:
    """Get current weather for a location.

    Args:
        location: City name (e.g., "Beijing", "Shanghai")
        unit: Temperature unit, either "celsius" or "fahrenheit"

    Returns:
        Weather information including temperature, condition, and humidity

    Example:
        >>> get_weather("Beijing")
        {"location": "Beijing", "temperature": 22, "unit": "celsius", ...}
    """
    location_key = location.lower()

    if location_key not in WEATHER_DATA:
        return {
            "error": f"Weather data not available for {location}",
            "available_locations": list(WEATHER_DATA.keys())
        }

    data = WEATHER_DATA[location_key]
    temp = data["temp"]

    if unit == "fahrenheit":
        temp = temp * 9/5 + 32

    return {
        "location": location,
        "temperature": round(temp, 1),
        "unit": unit,
        "condition": data["condition"],
        "humidity": data["humidity"],
        "timestamp": datetime.now().isoformat()
    }


@register_tool
def get_forecast(
    location: str,
    days: int = 5,
    unit: str = "celsius"
) -> dict:
    """Get weather forecast for upcoming days.

    Args:
        location: City name
        days: Number of days to forecast (1-7)
        unit: Temperature unit

    Returns:
        List of daily forecasts
    """
    if days < 1 or days > 7:
        return {"error": "Days must be between 1 and 7"}

    location_key = location.lower()
    if location_key not in WEATHER_DATA:
        return {"error": f"Location not found: {location}"}

    base_data = WEATHER_DATA[location_key]
    forecasts = []

    conditions = ["sunny", "cloudy", "partly_cloudy", "rainy"]

    for i in range(days):
        date = datetime.now() + timedelta(days=i+1)
        temp_variation = (i % 3) - 1  # -1, 0, or 1
        temp = base_data["temp"] + temp_variation * 2

        if unit == "fahrenheit":
            temp = temp * 9/5 + 32

        forecasts.append({
            "date": date.strftime("%Y-%m-%d"),
            "temperature": round(temp, 1),
            "condition": conditions[i % len(conditions)],
            "humidity": base_data["humidity"] + (i * 2) % 20
        })

    return {
        "location": location,
        "unit": unit,
        "forecasts": forecasts
    }


@register_tool
def search_weather_history(
    location: str,
    start_date: str,
    end_date: Optional[str] = None
) -> dict:
    """Search historical weather data.

    Args:
        location: City name
        start_date: Start date in YYYY-MM-DD format
        end_date: End date in YYYY-MM-DD format (optional, defaults to start_date)

    Returns:
        Historical weather records
    """
    try:
        start = datetime.strptime(start_date, "%Y-%m-%d")
        end = datetime.strptime(end_date, "%Y-%m-%d") if end_date else start
    except ValueError:
        return {"error": "Invalid date format. Use YYYY-MM-DD"}

    if end < start:
        return {"error": "End date must be after start date"}

    # Simulated historical data
    records = []
    current = start
    while current <= end:
        records.append({
            "date": current.strftime("%Y-%m-%d"),
            "high": 25 + (current.day % 5),
            "low": 15 + (current.day % 3),
            "condition": "historical_data"
        })
        current += timedelta(days=1)

    return {
        "location": location,
        "period": {
            "start": start_date,
            "end": end_date or start_date
        },
        "records": records
    }


@register_tool
def compare_weather(
    locations: list[str],
    unit: str = "celsius"
) -> dict:
    """Compare weather across multiple locations.

    Args:
        locations: List of city names to compare
        unit: Temperature unit

    Returns:
        Comparison of weather across locations
    """
    results = []

    for location in locations:
        weather = get_weather(location, unit)
        if "error" not in weather:
            results.append(weather)

    if not results:
        return {"error": "No valid locations found"}

    # Find extremes
    hottest = max(results, key=lambda x: x["temperature"])
    coldest = min(results, key=lambda x: x["temperature"])

    return {
        "locations": results,
        "summary": {
            "hottest": {
                "location": hottest["location"],
                "temperature": hottest["temperature"]
            },
            "coldest": {
                "location": coldest["location"],
                "temperature": coldest["temperature"]
            },
            "average_temperature": round(
                sum(r["temperature"] for r in results) / len(results), 1
            )
        }
    }
```

## Usage

### Local Development

```bash
# Start the MCP server
cd weather-env
aenv serve src/tools.py --port 8081
```

### Python SDK

```python
import asyncio
from aenv import Environment

async def main():
    async with Environment("weather-env") as env:
        # List available tools
        tools = await env.list_tools()
        print("Available tools:")
        for tool in tools:
            print(f"  - {tool.name}: {tool.description}")

        # Get current weather
        weather = await env.call_tool(
            "get_weather",
            {"location": "Beijing", "unit": "celsius"}
        )
        print(f"\nCurrent weather: {weather.content}")

        # Get forecast
        forecast = await env.call_tool(
            "get_forecast",
            {"location": "Shanghai", "days": 3}
        )
        print(f"\nForecast: {forecast.content}")

        # Compare locations
        comparison = await env.call_tool(
            "compare_weather",
            {"locations": ["Beijing", "Shanghai", "Shenzhen"]}
        )
        print(f"\nComparison: {comparison.content}")

asyncio.run(main())
```

### With OpenAI Agent

```python
from openai import OpenAI
from aenv import Environment
import json

async def weather_agent():
    async with Environment("weather-env") as env:
        tools = await env.list_tools()

        # Convert to OpenAI format
        openai_tools = [
            {
                "type": "function",
                "function": {
                    "name": t.name,
                    "description": t.description,
                    "parameters": t.input_schema
                }
            }
            for t in tools
        ]

        client = OpenAI()

        messages = [
            {"role": "user", "content": "What's the weather like in Beijing and Shanghai? Which is warmer?"}
        ]

        response = client.chat.completions.create(
            model="gpt-4",
            messages=messages,
            tools=openai_tools
        )

        # Handle tool calls
        for tool_call in response.choices[0].message.tool_calls:
            result = await env.call_tool(
                tool_call.function.name,
                json.loads(tool_call.function.arguments)
            )
            print(f"Tool {tool_call.function.name}: {result.content}")
```

## Deployment

### Build and Push

```bash
# Validate configuration
aenv validate

# Build the image
aenv build

# Push to registry
aenv push

# Release version
aenv release 1.0.0
```

### Use in Production

```python
from aenv import Environment

env = Environment(
    "weather-env",
    aenv_url="https://aenv.production.com",
    api_key="your-api-key"
)
```

## Testing

### Unit Tests

```python
# tests/test_tools.py
import pytest
from src.tools import get_weather, get_forecast, compare_weather

def test_get_weather():
    result = get_weather("Beijing", "celsius")
    assert "temperature" in result
    assert result["location"] == "Beijing"
    assert result["unit"] == "celsius"

def test_get_weather_fahrenheit():
    result = get_weather("Beijing", "fahrenheit")
    assert result["temperature"] > 50  # Fahrenheit

def test_get_weather_unknown_location():
    result = get_weather("Unknown", "celsius")
    assert "error" in result

def test_get_forecast():
    result = get_forecast("Shanghai", days=3)
    assert len(result["forecasts"]) == 3

def test_compare_weather():
    result = compare_weather(["Beijing", "Shanghai"])
    assert "summary" in result
    assert "hottest" in result["summary"]
```

### Integration Tests

```python
# tests/test_integration.py
import pytest
from aenv import Environment

@pytest.mark.asyncio
async def test_weather_environment():
    async with Environment("weather-env") as env:
        # Test tool listing
        tools = await env.list_tools()
        tool_names = [t.name for t in tools]
        assert "get_weather" in tool_names

        # Test tool execution
        result = await env.call_tool(
            "get_weather",
            {"location": "Beijing"}
        )
        assert not result.is_error
```

## Next Steps

- Add real weather API integration
- Add caching for API responses
- Add more weather metrics (wind, pressure, UV index)
- Create weather alerts tool
