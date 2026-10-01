# Sightline — SME Credit Copilot (Backend)

FastAPI backend for an explainable SME credit-analysis copilot. It reads a loan
file, extracts the financials, computes the credit ratios, scores a decision,
and drafts a credit memo — persisting every analysis as an audit trail.

## Architecture

The pipeline has four stages, and **which tool does which is the whole point**:

| Stage | Who does it | Where |
|-------|-------------|-------|
| 1. Extract financials from the document | **LLM** (Claude) | `app/llm.py` |
| 2. Compute ratios (DSCR, leverage, liquidity, CCC…) | **Code** | `app/finance.py` |
| 3. Score the decision + explain it | **Code** | `app/finance.py` |
| 4. Draft the credit memo | **LLM** (Claude) | `app/llm.py` |

The model only does *language* (reading messy docs, writing prose). All numbers
and the actual lending decision are computed in plain Python — so they're exact,
auditable, and explainable by design (EU AI Act, Art. 86 / Art. 14). The memo
model is handed the already-computed figures, so it can't invent numbers.

The Anthropic API key lives **only on the server** (`ANTHROPIC_API_KEY`). The
frontend never sees it — it calls this backend, and this backend calls Claude.

## Project layout

```
app/
  config.py       env-driven settings (DB, Anthropic, CORS)
  database.py     SQLAlchemy engine + session
  models.py       Analysis ORM model (the audit record)
  schemas.py      Pydantic request/response contracts
  finance.py      the deterministic engine (ratios, flags, scorecard)
  llm.py          the two Claude calls (extract, memo)
  routers/
    analyses.py   POST /api/analyze, GET /api/analyses, GET /api/analyses/{id}
  main.py         FastAPI app + CORS
requirements.txt
Procfile          Railway start command
.env.example
test_local.py     smoke test (LLM stubbed — no key needed)
```

## Run locally

```bash
python -m venv .venv && source .venv/bin/activate   # Windows: .venv\Scripts\activate
pip install -r requirements.txt
cp .env.example .env          # then paste your ANTHROPIC_API_KEY into .env
uvicorn app.main:app --reload
```

- API docs (interactive): http://localhost:8000/docs
- Health check: http://localhost:8000/api/health

With no `DATABASE_URL` set it uses a local SQLite file — zero setup. To run the
smoke test without any API key:

```bash
python test_local.py
```

## Endpoints

| Method | Path | Purpose |
|--------|------|---------|
| GET  | `/api/health` | liveness check |
| POST | `/api/analyze` | body `{ "text": "<loan file>" }` → full analysis |
| GET  | `/api/analyses` | recent analyses (summaries) |
| GET  | `/api/analyses/{id}` | one full analysis |

## Deploy to Railway (Postgres)

1. Push this folder to a GitHub repo.
2. In Railway: **New Project → Deploy from GitHub repo** (point it at this repo).
3. Add a **PostgreSQL** plugin — Railway sets `DATABASE_URL` automatically. The
   code normalises the `postgres://` URL it provides to the driver SQLAlchemy
   needs, so nothing else is required.
4. Add service variables:
   - `ANTHROPIC_API_KEY` — your key
   - `ANTHROPIC_MODEL` — e.g. `claude-sonnet-5` (default) or `claude-opus-4-8`
   - `CORS_ORIGINS` — your Vercel frontend URL, e.g. `https://sightline.vercel.app`
5. Railway uses the `Procfile` to start the server. Done.

The frontend (React/Vite, deployed on Vercel) comes next — it will call
`POST /api/analyze` and render the result.

## Notes

- Tables are created on startup for simplicity. When the schema starts evolving,
  switch to Alembic migrations.
- Auth/RBAC (JWT, officer roles) is the natural next backend layer, added
  deliberately rather than bundled in here.
- All borrower data in the demo is synthetic.


An SME credit assessment copilot for loan underwriting. Sightline reads a loan
application's supporting documents and returns a scored credit memo across nine
risk factors, so that two analysts reviewing the same borrower work from the same
framework rather than from their own spreadsheets.

## How it works

Documents go in. An extraction pass pulls the structured financial inputs out of
them, and a second pass scores the borrower against nine factors and drafts the
memo. The front end renders the decision, the score and the factor breakdown.

The API contract is generated from the backend schemas rather than written by
hand, so the two halves of the application cannot drift apart silently.

## Stack

| Layer | Technology |
| --- | --- |
| Backend | FastAPI, Python 3.11 |
| Frontend | React + Vite, TypeScript |
| Database | PostgreSQL in production, SQLite fallback locally |
| Extraction and analysis | Anthropic API |
| Types | OpenAPI generation, backend to frontend |
| Tests | pytest |
| Deployment | Railway |

## Layout

Backend and frontend live in the same repository.

    app/                    FastAPI backend
    sightline-frontend/     React frontend
    requirements.txt
    runtime.txt
    Procfile

## Running locally

Requires Node 18+, Python 3.11+ and an Anthropic API key.

**Backend**, from the repository root:

    python3 -m venv .venv
    source .venv/bin/activate
    pip install -r requirements.txt
    cp .env.example .env        # set ANTHROPIC_API_KEY and CORS_ORIGINS
    uvicorn app.main:app --port 8000 --reload

`DATABASE_URL` is optional locally — without it the app falls back to SQLite.
Check `http://localhost:8000/api/health` before going further.

**Frontend**, in a second terminal:

    cd sightline-frontend
    cp .env.example .env        # VITE_API_BASE=http://localhost:8000
    npm install
    npm run dev -- --host

`--host` is required: without it Vite binds to IPv6 only and forwarded ports
return a 404. `legacy-peer-deps` is set in the committed `.npmrc`, so a plain
`npm install` is enough.

The app runs at `http://localhost:5173`. Load the sample case and run an analysis
to confirm the full path end to end — allow 15–30 seconds on a cold start.

## Tests

    pytest test_schemas.py test_local.py

`test_local.py` calls the Anthropic API, so it needs a valid key and costs a few
cents to run.

## Regenerating API types

With the backend running locally:

    cd sightline-frontend
    npm run gen:types

## Status

Built July 2026 as sole developer, from requirements through to production
deployment.
