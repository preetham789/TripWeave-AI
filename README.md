# TripMate AI

> A multi-agent travel planning web application that converts a natural-language request into live flight-status context, hotel research, and an actionable itinerary.

[![Python](https://img.shields.io/badge/Python-3.11-3776AB?logo=python&logoColor=white)](https://www.python.org/)
[![FastAPI](https://img.shields.io/badge/FastAPI-0.136-009688?logo=fastapi&logoColor=white)](https://fastapi.tiangolo.com/)
[![LangGraph](https://img.shields.io/badge/Workflow-LangGraph-1C3C3C)](https://langchain-ai.github.io/langgraph/)
[![License](https://img.shields.io/badge/License-MIT-22C55E.svg)](LICENSE)

TripMate AI is a FastAPI application built around a LangGraph workflow. It coordinates purpose-built agents to interpret a trip request, collect available flight-status data and hotel research, create a budget-aware itinerary, and return a structured travel recommendation in a polished web interface.

## Contents

- [Key capabilities](#key-capabilities)
- [How it works](#how-it-works)
- [Technology](#technology)
- [Getting started](#getting-started)
- [Configuration](#configuration)
- [Run with Docker](#run-with-docker)
- [API reference](#api-reference)
- [Project structure](#project-structure)
- [Important limitations](#important-limitations)
- [License](#license)

## Key capabilities

- **Natural-language trip planning** — handles requests that include a destination, origin, duration, budget, or preferred activities.
- **Agent workflow** — sequential flight, hotel, itinerary, and final-response agents produce one cohesive result.
- **Flight route recognition** — resolves common cities, countries, aliases, and IATA codes to query AviationStack.
- **Hotel research** — uses Tavily search results as context for accommodation suggestions.
- **Persistent workflow state** — uses PostgreSQL-backed LangGraph checkpoints, with a thread ID retained in the browser's local storage.
- **Useful delivery UI** — renders Markdown responses, supports copy-to-clipboard, and can export the displayed plan as a PDF.
- **Container-ready** — includes a Dockerfile for consistent deployment.

## How it works

```text
Browser UI
    │  POST /api/travel
    ▼
FastAPI application
    │
    ▼
┌────────────────────────────────────────────────────────────────────┐
│                         LangGraph workflow                          │
│                                                                    │
│  Flight Agent → Hotel Agent → Itinerary Agent → Final Response     │
└────────────────────────────────────────────────────────────────────┘
    │                 │                         │
    ▼                 ▼                         ▼
AviationStack       Tavily                      Groq
    │
    ▼
PostgreSQL checkpoint store
```

1. The user submits a travel request in the web interface or through the API.
2. The flight agent extracts a route when possible and retrieves live flight/status data from AviationStack.
3. The hotel agent searches Tavily for accommodation-related research relevant to the request.
4. The itinerary agent combines the request and research into a practical itinerary.
5. The final-response agent formats the plan into trip summary, flight, hotel, itinerary, budget, and recommendation sections.

## Technology

| Layer | Implementation |
| --- | --- |
| Web application | FastAPI, Uvicorn, Jinja2, Pydantic |
| Agent orchestration | LangGraph and LangChain |
| Language model | Groq via `langchain-groq` |
| Flight data | AviationStack |
| Hotel research | Tavily |
| Persistence | PostgreSQL, Psycopg, LangGraph Postgres Saver |
| Client experience | HTML, CSS, JavaScript, Marked, html2pdf.js |
| Packaging | Docker |

## Getting started

### Prerequisites

- Python 3.11 or later
- A PostgreSQL database that is reachable from the application
- API keys for [Groq](https://console.groq.com/), [Tavily](https://tavily.com/), and [AviationStack](https://aviationstack.com/)

### Installation

1. Clone the repository and open the project directory.

   ```powershell
   git clone <repository-url>
   cd TripWeave-AI
   ```

2. Create and activate a virtual environment.

   ```powershell
   python -m venv .venv
   .\.venv\Scripts\Activate.ps1
   ```

3. Install the required packages.

   ```powershell
   pip install -r requirements.txt
   ```

4. Create a `.env` file in the project root using the configuration below.

5. Start the development server.

   ```powershell
   uvicorn app:app --reload
   ```

Open [http://127.0.0.1:8000](http://127.0.0.1:8000) in your browser. The health check is available at [http://127.0.0.1:8000/health](http://127.0.0.1:8000/health).

## Configuration

Create a `.env` file alongside `app.py`. Never commit this file; it is already covered by `.gitignore`.

```env
# Required
GROQ_API_KEY=your_groq_api_key
TAVILY_API_KEY=your_tavily_api_key
AVIATIONSTACK_API_KEY=your_aviationstack_api_key
DATABASE_URL=postgresql://username:password@host:5432/database

# Optional
GROQ_MODEL=openai/gpt-oss-120b
DEFAULT_ORIGIN_IATA=DAC
```

| Variable | Required | Description |
| --- | --- | --- |
| `GROQ_API_KEY` | Yes | Credential used to generate the itinerary and final answer. |
| `TAVILY_API_KEY` | Yes | Credential used for hotel-related web research. |
| `AVIATIONSTACK_API_KEY` | Yes | Credential used for live flight/status requests. |
| `DATABASE_URL` | Yes | PostgreSQL URL used by the LangGraph checkpointer. |
| `GROQ_MODEL` | No | Groq model identifier; defaults to `openai/gpt-oss-120b`. |
| `DEFAULT_ORIGIN_IATA` | No | Origin airport for destination-only requests; defaults to `DAC` (Dhaka). |

If `DATABASE_URL` does not already include an SSL mode, the application adds `sslmode=require` before connecting.

## Run with Docker

Build the image from the project root:

```powershell
docker build -t tripmate-ai .
```

Run the container using the local environment file:

```powershell
docker run --rm --env-file .env -p 8000:8000 tripmate-ai
```

The container serves the application on `0.0.0.0:8000`. Your database must be externally reachable from the container.

## API reference

### `POST /api/travel`

Generates a travel plan from a natural-language request.

**Request body**

```json
{
  "message": "Plan a 7-day Japan trip from India with flights, hotels, and sightseeing under 2 lakhs.",
  "thread_id": "optional-existing-thread-id"
}
```

| Field | Type | Required | Description |
| --- | --- | --- | --- |
| `message` | string | Yes | The trip request. Empty messages return `400 Bad Request`. |
| `thread_id` | string | No | Identifier for the graph checkpoint. One is generated when omitted. |

**Example response**

```json
{
  "success": true,
  "thread_id": "user_abc123",
  "answer": "## Trip Summary\n...",
  "flight_results": "Live flights from DEL to NRT\n...",
  "hotel_results": "1. **Example hotel**\n...",
  "itinerary": "Day 1: ...",
  "llm_calls": 4
}
```

| Response field | Description |
| --- | --- |
| `answer` | Formatted Markdown travel recommendation displayed by the frontend. |
| `flight_results` | Raw, formatted flight lookup context returned by the flight agent. |
| `hotel_results` | Raw, formatted hotel research returned by the hotel agent. |
| `itinerary` | Intermediate itinerary produced before final formatting. |
| `llm_calls` | Workflow counter; the current graph increments it once per agent. |

### `GET /health`

Returns a lightweight service status response:

```json
{
  "status": "ok",
  "message": "AI Travel Planner API is running"
}
```

## Local smoke test

After configuring the environment, run the interactive script:

```powershell
python test.py
```

Enter a trip request when prompted. This directly invokes the same LangGraph workflow used by the API and therefore requires working credentials and PostgreSQL access.

## Project structure

```text
TripWeave-AI/
├── app.py                  # FastAPI application, routes, static files, templates
├── backend.py              # Graph state, agents, Groq client, PostgreSQL checkpointer
├── Dockerfile              # Production container definition
├── requirements.txt        # Pinned Python dependencies
├── test.py                 # Interactive workflow smoke test
├── static/
│   ├── script.js           # API client, thread ID persistence, copy and PDF export
│   └── style.css           # Responsive web interface styling
├── templates/
│   └── index.html          # Single-page frontend shell
└── tools/
    ├── flight_tool.py      # Location parsing and AviationStack integration
    └── tavily_tool.py      # Tavily search integration
```

## Important limitations

- **This is not a booking engine.** The application does not make reservations or process payments.
- **Flight results are operational data.** AviationStack data may contain schedules, status, gates, terminals, and delays; it does not provide ticket prices.
- **Travel guidance is generated content.** Prices, availability, visa rules, local regulations, safety advice, and opening hours should be verified through official or provider sources before travel.
- **External services are required.** A missing or invalid Groq key prevents the application from starting; unavailable Tavily, AviationStack, or PostgreSQL services can cause planning requests to fail.

## License

Copyright © 2026 Preetham P. Distributed under the [MIT License](LICENSE).
