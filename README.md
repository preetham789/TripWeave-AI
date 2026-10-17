# TripMate AI

TripMate AI is a FastAPI web application that turns a natural-language travel request into a practical trip plan. It uses a LangGraph workflow to collect live flight-status information, find hotel research, and generate a day-by-day itinerary with Groq.

## What it does

- Accepts travel-planning prompts through a simple browser interface or the API.
- Looks up live flight and route-status information through AviationStack.
- Searches for hotel information using Tavily.
- Uses Groq to create an itinerary and a polished final recommendation.
- Persists LangGraph conversation state in PostgreSQL, so requests can reuse a `thread_id`.

> AviationStack supplies flight status and schedule data, not ticket fares. The generated plan should be treated as planning guidance and checked with providers before booking.

## Tech stack

- Python, FastAPI, Uvicorn, and Jinja2
- LangGraph and LangChain
- Groq
- PostgreSQL with the LangGraph Postgres checkpointer
- Tavily and AviationStack

## Setup

1. Create and activate a virtual environment.

   ```powershell
   python -m venv .venv
   .\.venv\Scripts\Activate.ps1
   ```

2. Install dependencies.

   ```powershell
   pip install -r requirements.txt
   ```

3. Add a `.env` file in the project root. It must contain the following values:

   ```env
   GROQ_API_KEY=your_groq_api_key
   TAVILY_API_KEY=your_tavily_api_key
   AVIATIONSTACK_API_KEY=your_aviationstack_api_key
   DATABASE_URL=postgresql://user:password@host:5432/database

   # Optional
   GROQ_MODEL=openai/gpt-oss-120b
   DEFAULT_ORIGIN_IATA=DAC
   ```

   `DATABASE_URL` is required and the application adds `sslmode=require` automatically when it is absent.

## Run locally

```powershell
uvicorn app:app --reload
```

Open [http://127.0.0.1:8000](http://127.0.0.1:8000). The health endpoint is available at [http://127.0.0.1:8000/health](http://127.0.0.1:8000/health).

## API

Send a request to `POST /api/travel`:

```json
{
  "message": "Plan a 7-day Japan trip from India with flights, hotels, and sightseeing under 2 lakhs.",
  "thread_id": "optional-existing-thread-id"
}
```

The response includes the generated answer, the thread ID, raw flight and hotel research, the itinerary, and the number of LLM calls.

## Workflow

`flight_agent` → `hotel_agent` → `itinerary_agent` → `final_agent`

The workflow runs sequentially: it fetches flight data, gathers hotel research, produces an itinerary, then formats the final travel response.

## Project structure

```text
app.py                 FastAPI application and HTTP endpoints
backend.py             LangGraph workflow and PostgreSQL checkpointing
tools/flight_tool.py   AviationStack query and airport/route parsing
tools/tavily_tool.py   Tavily hotel research
templates/index.html   Web interface
static/                Browser styles and JavaScript
test.py                Interactive command-line smoke test
```

## Command-line smoke test

After configuring the environment, run:

```powershell
python test.py
```

Enter a travel request when prompted. This uses the same external services and PostgreSQL database as the web app.

## License

This project is distributed under the terms in [LICENSE](LICENSE).
