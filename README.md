# TripWeave AI

> A multi-agent travel-planning web app that turns a natural-language request into a weather-aware, research-backed itinerary.

[![Python](https://img.shields.io/badge/Python-3.11-3776AB?logo=python&logoColor=white)](https://www.python.org/)
[![FastAPI](https://img.shields.io/badge/FastAPI-0.136-009688?logo=fastapi&logoColor=white)](https://fastapi.tiangolo.com/)
[![LangGraph](https://img.shields.io/badge/Workflow-LangGraph-1C3C3C)](https://langchain-ai.github.io/langgraph/)
[![License](https://img.shields.io/badge/License-MIT-22C55E.svg)](LICENSE)

TripWeave AI (branded as **TripMate AI** in the current web interface) is a FastAPI application powered by a LangGraph workflow. It coordinates specialised agents to research flight options, accommodations, destination weather, and activities before producing a practical trip plan. The browser experience supports persistent conversation threads, Markdown rendering, copying results, and PDF export.

## Contents

- [What it does](#what-it-does)
- [Workflow](#workflow)
- [Architecture](#architecture)
- [Technology](#technology)
- [Getting started](#getting-started)
- [Configuration](#configuration)
- [Running the application](#running-the-application)
- [API](#api)
- [Testing and diagnostics](#testing-and-diagnostics)
- [Project structure](#project-structure)
- [Operational notes](#operational-notes)
- [License](#license)

## What it does

- Accepts free-form trip requests, such as “Plan a 7-day Japan trip from India under ₹2 lakh.”
- Generates flight guidance from airport and airline data supplied through an AviationStack MCP server.
- Researches hotel options using Tavily’s hosted MCP service.
- Retrieves current conditions and a short forecast through a local OpenWeather-backed MCP server.
- Produces a budget-conscious day-by-day itinerary and a final Markdown travel brief with flight, hotel, weather, budget, and recommendation sections.
- Stores LangGraph checkpoints in PostgreSQL so a browser can continue using the same `thread_id` across requests.
- Provides a responsive interface with example prompts, loading and error states, copy-to-clipboard, and client-side PDF download.

## Workflow

```text
Travel request
      |
      v
Flight agent  ──> airport and airline context via AviationStack MCP
      |
      v
Hotel agent   ──> accommodation research via Tavily MCP
      |
      v
Weather agent ──> current conditions + forecast via local Weather MCP
      |
      v
Itinerary agent ──> practical, weather-aware daily plan
      |
      v
Final-response agent ──> formatted Markdown recommendation
      |
      v
FastAPI response + PostgreSQL LangGraph checkpoint
```

Each request begins with a fresh state for the supplied user message. The persisted `thread_id` is used by the LangGraph checkpointer to retain workflow history; the frontend saves that identifier in browser local storage.

## Architecture

```text
Browser (HTML/CSS/JavaScript)
        |
        | POST /api/travel
        v
FastAPI application (app.py)
        |
        v
LangGraph workflow (backend.py)
        |
        +-- Groq LLM: flight guidance, itinerary, final response
        +-- Tavily MCP: hotel research
        +-- AviationStack MCP: airport and airline context
        +-- Local Weather MCP: OpenWeather current weather and forecast
        |
        v
PostgreSQL / LangGraph PostgresSaver: checkpoints
```

## Technology

| Area | Implementation |
| --- | --- |
| API and frontend delivery | FastAPI, Uvicorn, Jinja2, Pydantic |
| Agent orchestration | LangGraph, LangChain |
| Language model | Groq via `langchain-groq` |
| MCP integration | `langchain-mcp-adapters`, Tavily hosted MCP, AviationStack MCP, local FastMCP weather server |
| Weather data | OpenWeather API |
| State persistence | PostgreSQL, Psycopg, LangGraph Postgres Saver |
| Client experience | Vanilla HTML/CSS/JavaScript, Marked, html2pdf.js |
| Containerisation | Docker |

## Getting started

### Prerequisites

- Python 3.11 or later
- A reachable PostgreSQL database
- [uv](https://docs.astral.sh/uv/) on your `PATH` — the AviationStack MCP server is launched through `uvx`
- API credentials for [Groq](https://console.groq.com/), [Tavily](https://tavily.com/), [AviationStack](https://aviationstack.com/), and [OpenWeather](https://openweathermap.org/api)

### Installation

1. Clone the repository and enter it.

   ```powershell
   git clone <repository-url>
   cd TripWeave-AI
   ```

2. Create and activate a virtual environment.

   ```powershell
   python -m venv .venv
   .\.venv\Scripts\Activate.ps1
   ```

3. Install Python packages.

   ```powershell
   pip install -r requirements.txt
   ```

4. Create `.env` in the project root using the configuration below. Do not commit this file.

## Configuration

```env
# Required to start the API and generate plans
GROQ_API_KEY=your_groq_api_key
DATABASE_URL=postgresql://username:password@host:5432/database

# Required for the research integrations used during a planning request
TAVILY_API_KEY=your_tavily_api_key
AVIATIONSTACK_API_KEY=your_aviationstack_api_key
OPENWEATHER_API_KEY=your_openweather_api_key

# Optional
GROQ_MODEL=openai/gpt-oss-120b
```

| Variable | Required | Purpose |
| --- | --- | --- |
| `GROQ_API_KEY` | Yes | Authenticates Groq model calls. The application validates it when importing the backend. |
| `DATABASE_URL` | Yes | PostgreSQL connection used by the LangGraph checkpointer. `sslmode=require` is appended when no SSL mode is supplied. |
| `TAVILY_API_KEY` | For hotel research | Passed to Tavily’s hosted MCP endpoint. |
| `AVIATIONSTACK_API_KEY` | For flight research | Passed to the AviationStack MCP server. |
| `OPENWEATHER_API_KEY` | For weather research | Used by `custom_weather_mcp.py` for current weather and forecast data. |
| `GROQ_MODEL` | No | Groq model ID; defaults to `openai/gpt-oss-120b`. |

> Keep API keys out of source control. `.env` is already ignored by Git.

## Running the application

Start the development server:

```powershell
uvicorn app:app --reload
```

Open [http://127.0.0.1:8000](http://127.0.0.1:8000). The service health check is available at [http://127.0.0.1:8000/health](http://127.0.0.1:8000/health).

### Docker

Build the image:

```powershell
docker build -t tripweave-ai .
```

Run it with your environment file:

```powershell
docker run --rm --env-file .env -p 8000:8000 tripweave-ai
```

The image serves on `0.0.0.0:8000`. PostgreSQL must be reachable from the container. The current Dockerfile does **not** install `uv`; install it in the image before relying on AviationStack MCP flight research, or run the application outside Docker with `uv` installed.

## API

### `POST /api/travel`

Creates a travel plan from a natural-language request.

```json
{
  "message": "Plan a 7-day Japan trip from India with flights, hotels, and sightseeing under 2 lakhs.",
  "thread_id": "optional-existing-thread-id"
}
```

| Field | Type | Required | Description |
| --- | --- | --- | --- |
| `message` | string | Yes | The travel request. Blank messages return `400 Bad Request`. |
| `thread_id` | string | No | Existing checkpoint thread to use. A `user_<uuid>` ID is created if omitted. |

Successful responses include the final Markdown answer and intermediate workflow output:

```json
{
  "success": true,
  "thread_id": "user_abc123",
  "answer": "## Trip Summary\n...",
  "flight_results": "...",
  "hotel_results": "...",
  "itinerary": "...",
  "llm_calls": 4
}
```

`llm_calls` is the workflow counter for the flight, hotel, itinerary, and final-response stages; the weather stage does not increment it. Its current expected value is `4` for a completed run.

### `GET /health`

Returns:

```json
{
  "status": "ok",
  "message": "AI Travel Planner API is running"
}
```

## Testing and diagnostics

Run a syntax smoke check without making network calls:

```powershell
python -m py_compile app.py backend.py mcp_client.py custom_weather_mcp.py
```

`test.py` is currently a minimal Tavily MCP probe. It invokes a sample query when valid Tavily credentials are configured:

```powershell
python test.py
```

For a full end-to-end check, run the API and submit a request from the web UI or with PowerShell:

```powershell
Invoke-RestMethod -Method Post -Uri http://127.0.0.1:8000/api/travel `
  -ContentType 'application/json' `
  -Body '{"message":"Plan a 3-day trip to Dubai from Bengaluru."}'
```

## Project structure

```text
TripWeave-AI/
├── app.py                   # FastAPI routes, static files, and Jinja template setup
├── backend.py               # TravelState, five-agent LangGraph workflow, PostgreSQL checkpointing
├── mcp_client.py            # Tavily, AviationStack, and local Weather MCP clients
├── custom_weather_mcp.py    # FastMCP server backed by OpenWeather
├── mcp_client_test.py       # Standalone Tavily MCP test helper
├── test.py                  # Minimal Tavily MCP probe
├── requirements.txt         # Python dependencies
├── Dockerfile               # Application container definition
├── static/
│   ├── script.js            # API client, thread persistence, copy, and PDF export
│   └── style.css            # Responsive UI styles
├── templates/
│   └── index.html           # Single-page frontend
├── tools/
│   ├── flight_tool.py       # Earlier direct AviationStack utility and route parser
│   └── tavily_tool.py       # Earlier direct Tavily utility
└── workflow/
    └── Screenshot ...png    # Historical workflow illustration
```

The active application workflow uses `mcp_client.py`; the files in `tools/` are useful standalone utilities but are not imported by the current backend.

## Operational notes

- This application provides planning guidance; it does not book flights or hotels, process payments, or guarantee availability.
- AviationStack MCP data is used as flight context. Flight prices and inventory should be confirmed with a booking provider.
- Travel-related facts can change quickly. Verify visas, opening hours, safety guidance, rates, and reservation availability with official sources before travel.
- The workflow depends on external APIs and a PostgreSQL connection. A failing integration can cause the related agent or request to fail; the flight agent catches its own integration errors and returns an unavailable message.
- The current weather MCP server makes synchronous HTTP calls and returns OpenWeather’s error payloads where appropriate. Use a valid city-level destination for the most reliable results.

## License

Copyright © 2026 Preetham P. Distributed under the [MIT License](LICENSE).
