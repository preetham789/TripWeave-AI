# TripMate AI

> An AI-powered travel planning application that transforms a simple trip request into flight intelligence, hotel research, and a practical day-by-day itinerary.

[![Python](https://img.shields.io/badge/Python-3.10%2B-3776AB?logo=python&logoColor=white)](https://www.python.org/)
[![FastAPI](https://img.shields.io/badge/FastAPI-0.136-009688?logo=fastapi&logoColor=white)](https://fastapi.tiangolo.com/)
[![LangGraph](https://img.shields.io/badge/Orchestration-LangGraph-1C3C3C)](https://langchain-ai.github.io/langgraph/)
[![License](https://img.shields.io/badge/License-MIT-green.svg)](LICENSE)

TripMate AI is a full-stack, multi-agent travel planner built with FastAPI and LangGraph. It coordinates specialized agents to retrieve live flight-status information, research hotel options, and synthesize the results into a clear, budget-aware travel plan.

## Highlights

- **Natural-language planning** — accepts requests such as “Plan a 7-day Japan trip from India under 2 lakhs.”
- **Multi-agent workflow** — dedicated flight, hotel, itinerary, and response-generation agents work in sequence.
- **Live travel research** — integrates AviationStack for flight status and Tavily for hotel discovery.
- **Persistent conversations** — stores LangGraph checkpoints in PostgreSQL and supports reusable thread IDs.
- **Browser and API access** — includes a responsive Jinja2 frontend and a documented JSON API.

> **Note:** AviationStack provides operational flight and schedule data, not fare pricing. Verify all availability, prices, visas, and bookings with the relevant provider before travelling.

## Architecture

```text
User / Web UI
      │
      ▼
 FastAPI  ── POST /api/travel
      │
      ▼
┌─────────────────────────────────────────────────────────────┐
│                     LangGraph workflow                       │
│                                                             │
│  Flight Agent → Hotel Agent → Itinerary Agent → Final Agent │
└─────────────────────────────────────────────────────────────┘
      │              │                    │
      ▼              ▼                    ▼
AviationStack      Tavily                Groq
      │
      ▼
PostgreSQL checkpoint store
```

## Technology

| Area | Tools |
| --- | --- |
| API and server | FastAPI, Uvicorn, Pydantic |
| Agent orchestration | LangGraph, LangChain |
| AI model | Groq |
| Data sources | AviationStack, Tavily |
| State persistence | PostgreSQL, Psycopg, LangGraph Postgres Saver |
| Frontend | Jinja2, HTML, CSS, JavaScript |

## Getting started

### Prerequisites

- Python 3.10 or later
- A PostgreSQL database reachable from your environment
- API keys for [Groq](https://console.groq.com/), [Tavily](https://tavily.com/), and [AviationStack](https://aviationstack.com/)

### Installation

1. Clone the repository and enter the project directory.

   ```powershell
   git clone <repository-url>
   cd TripWeave-AI
   ```

2. Create and activate a virtual environment.

   ```powershell
   python -m venv .venv
   .\.venv\Scripts\Activate.ps1
   ```

3. Install the dependencies.

   ```powershell
   pip install -r requirements.txt
   ```

4. Create a `.env` file in the project root.

   ```env
   GROQ_API_KEY=your_groq_api_key
   TAVILY_API_KEY=your_tavily_api_key
   AVIATIONSTACK_API_KEY=your_aviationstack_api_key
   DATABASE_URL=postgresql://username:password@host:5432/database

   # Optional configuration
   GROQ_MODEL=openai/gpt-oss-120b
   DEFAULT_ORIGIN_IATA=DAC
   ```

   The application automatically appends `sslmode=require` to `DATABASE_URL` when it is not already present.

### Run the application

```powershell
uvicorn app:app --reload
```

Visit [http://127.0.0.1:8000](http://127.0.0.1:8000) to use the web application. Confirm the service is available at [http://127.0.0.1:8000/health](http://127.0.0.1:8000/health).

## API reference

### `POST /api/travel`

Creates a travel plan from a natural-language request.

**Request body**

```json
{
  "message": "Plan a 7-day Japan trip from India with flights, hotels, and sightseeing under 2 lakhs.",
  "thread_id": "optional-existing-thread-id"
}
```

| Field | Type | Required | Description |
| --- | --- | --- | --- |
| `message` | string | Yes | The travel requirement or question. |
| `thread_id` | string | No | An existing ID used to continue a saved LangGraph conversation. |

**Successful response**

```json
{
  "success": true,
  "thread_id": "user_abc123",
  "answer": "...",
  "flight_results": "...",
  "hotel_results": "...",
  "itinerary": "...",
  "llm_calls": 4
}
```

### `GET /health`

Returns a simple service-health response.

```json
{
  "status": "ok",
  "message": "AI Travel Planner API is running"
}
```

## Project structure

```text
TripWeave-AI/
├── app.py                  # FastAPI application, frontend, and API routes
├── backend.py              # LangGraph definition and PostgreSQL checkpointing
├── requirements.txt        # Python dependencies
├── test.py                 # Interactive command-line smoke test
├── static/
│   ├── script.js           # Frontend behavior
│   └── style.css           # Frontend styles
├── templates/
│   └── index.html          # Main web interface
└── tools/
    ├── flight_tool.py      # Route parsing and AviationStack integration
    └── tavily_tool.py      # Tavily hotel-research integration
```

## Local smoke test

Once environment variables are configured, run:

```powershell
python test.py
```

Enter a travel request when prompted. The script exercises the same workflow, API providers, and PostgreSQL checkpointer used by the web application.

## Environment variables

| Variable | Required | Purpose |
| --- | --- | --- |
| `GROQ_API_KEY` | Yes | Authenticates requests to the Groq model API. |
| `TAVILY_API_KEY` | Yes | Enables hotel and destination research. |
| `AVIATIONSTACK_API_KEY` | Yes | Enables live flight-status lookups. |
| `DATABASE_URL` | Yes | PostgreSQL connection string for LangGraph checkpoints. |
| `GROQ_MODEL` | No | Groq model name; defaults to `openai/gpt-oss-120b`. |
| `DEFAULT_ORIGIN_IATA` | No | Default departure airport for destination-only queries; defaults to `DAC`. |

## License

Released under the [MIT License](LICENSE).
