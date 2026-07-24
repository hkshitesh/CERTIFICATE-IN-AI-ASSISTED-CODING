# ExpenseFlow API

A small expense submission and approval API. **PoC, not production.**

One user journey: **submit** an expense in some currency → it is normalised to a
base currency (**INR**) on write → **approve or reject** it. A reporting endpoint
returns AI-generated spending insights.

> **Money is stored as integer minor units (paise/cents), never float.** Base
> currency is INR.

---

## What it does

- Submit an expense (description, amount in minor units, currency, category, submitter).
- List expenses, optionally filtered by `status` and/or `category`.
- Fetch a single expense, or the *n* most recent.
- Approve or reject a pending expense (only `pending` → `approved`/`rejected` is legal).
- Generate AI spending insights across all expenses (`/reports/insights`).

Two behaviours worth knowing, because they reflect the **real code**:

- **PII masking on read.** Free-text descriptions are passed through
  `app/sanitize.py` and redacted before being returned by the read endpoints.
  Numeric fields are left untouched. The stored row is never modified.
- **FX conversion is not implemented yet.** `create_expense` contains a
  `TODO(FX)`; for now `amount_base_minor` is set equal to the submitted
  `amount_minor` (correct only for INR submissions). There is no external FX call
  in the API today.

---

## Stack

- **Python** 3.10+ (project targets 3.12)
- **FastAPI** + **Uvicorn** (ASGI server)
- **SQLAlchemy 2.x** ORM on **SQLite** (file: `expenseflow.db`)
- **pydantic v2** for request/response models
- **python-dotenv** for configuration/secrets
- **anthropic** SDK for the insights endpoint
- **httpx** (used by the test client and the optional Streamlit UI)
- **pytest** for tests

---

## Layout

```
app/
  main.py       FastAPI app, /health, router wiring, table creation on startup
  db.py         engine (SQLite), SessionLocal, Base, get_db, init_db
  models.py     Expense ORM model + CHECK constraints
  schemas.py    pydantic v2 ExpenseCreate / ExpenseOut
  routes.py     the endpoints (expenses + reports)
  insights.py   Anthropic-backed insight generator (best-effort, never raises)
  sanitize.py   PII masking for descriptions
tests/
  test_api.py   TestClient tests against a fresh temporary SQLite DB
ui/
  app.py        optional Streamlit front-end (talks to the API over HTTP)
docs/
  ARCHITECTURE.md
```

---

## Setup (Windows)

From the project root (`expenseflow\`). Use the launcher `py` to pick the
interpreter; `py -3.12` if you have 3.12 installed, otherwise `py -3`.

### Create and activate the virtual environment

**Command Prompt (cmd.exe):**

```bat
py -3.12 -m venv .venv
.venv\Scripts\activate.bat
```

**PowerShell:**

```powershell
py -3.12 -m venv .venv
.venv\Scripts\Activate.ps1
```

> If PowerShell blocks the activation script, allow it for the current user once:
> `Set-ExecutionPolicy -Scope CurrentUser RemoteSigned`

### Install dependencies

There is no lockfile in the repo; install the packages directly:

```bat
python -m pip install --upgrade pip
pip install fastapi "uvicorn[standard]" sqlalchemy pydantic python-dotenv anthropic httpx pytest
```

Optional — only if you want to run the Streamlit UI:

```bat
pip install streamlit
```

To pin what you installed for later reuse:

```bat
pip freeze > requirements.txt
```

---

## Configure `.env`

Configuration and secrets are read from the environment via `python-dotenv`,
which loads a `.env` file from the project root if present. **Never hardcode
secrets.** Create `.env`:

```dotenv
# Required only for real AI insights. Without it, /reports/insights still works
# but returns a safe fallback object instead of model-generated text.
ANTHROPIC_API_KEY=sk-ant-...

# Optional. Defaults to sqlite:///expenseflow.db when unset.
DATABASE_URL=sqlite:///expenseflow.db
```

| Variable | Required | Default | Used by |
|---|---|---|---|
| `ANTHROPIC_API_KEY` | No (recommended for insights) | — | `app/insights.py` (`GET /reports/insights`) |
| `DATABASE_URL` | No | `sqlite:///expenseflow.db` | `app/db.py` |

> The insights endpoint is best-effort: on any API, network, or validation error
> (including a missing key) it logs and returns a fallback
> `{"summary": ..., "bullets": [...]}` rather than failing the request.

---

## Run the server

```bat
python -m uvicorn app.main:app --reload
```

- API base: `http://127.0.0.1:8000`
- Interactive docs (Swagger UI): `http://127.0.0.1:8000/docs`
- OpenAPI schema: `http://127.0.0.1:8000/openapi.json`

Tables are created automatically on startup (`init_db` in the app lifespan).

### Optional: run the Streamlit UI

The UI reads `API_BASE` (default `http://127.0.0.1:8000`). It also supports an
optional `API_KEY` env var that it sends as an `X-API-Key` header on writes — the
API in *this* repo does not require one, so leave it unset here.

```bat
streamlit run ui\app.py
```

---

## Run the tests

```bat
python -m pytest -q
```

`tests/test_api.py` drives the app with FastAPI's `TestClient` against a fresh
temporary SQLite database per test (the `get_db` dependency is overridden), so it
never touches `expenseflow.db`. It covers: create → `pending` with an id, listing
filtered by status, 404 on a missing id, approve flipping the status, and a 409
when re-approving an already-decided expense.

---

## Endpoint reference

Base URL: `http://127.0.0.1:8000`. All expense endpoints live under `/expenses`;
insights under `/reports`.

| Method | Path | Purpose | Success | Errors |
|---|---|---|---|---|
| GET | `/health` | Liveness probe | 200 `{"status":"ok"}` | — |
| POST | `/expenses` | Submit a new expense | 201 `ExpenseOut` | 422 validation |
| GET | `/expenses` | List expenses (filterable) | 200 `[ExpenseOut]` | — |
| GET | `/expenses/{expense_id}` | Fetch one expense | 200 `ExpenseOut` | 404 not found |
| GET | `/expenses/recent/{n}` | *n* most recent, newest first | 200 `[ExpenseOut]` | 422 if `n < 1` |
| POST | `/expenses/{expense_id}/approve` | Approve a pending expense | 200 `ExpenseOut` | 404 not found; 409 if not `pending` |
| POST | `/expenses/{expense_id}/reject` | Reject a pending expense | 200 `ExpenseOut` | 404 not found; 409 if not `pending` |
| GET | `/reports/insights` | AI spending insights | 200 `{"insight": {...}}` | — |

### Request body — `ExpenseCreate` (`POST /expenses`)

```json
{
  "description": "Team lunch",
  "amount_minor": 12345,
  "currency": "INR",
  "category": "meals",
  "submitted_by": "alice"
}
```

Validation (from `app/schemas.py` and the DB CHECK constraints):

- `description` — non-empty string.
- `amount_minor` — integer, **> 0**, in minor units of `currency`.
- `currency` — exactly 3 alphabetic letters, normalised to upper case (e.g. `USD`).
- `category` — non-empty string.
- `submitted_by` — non-empty string.

### Response body — `ExpenseOut`

Returned by all expense endpoints. Descriptions are PII-masked on read.

```json
{
  "id": 1,
  "description": "Team lunch",
  "amount_minor": 12345,
  "currency": "INR",
  "category": "meals",
  "submitted_by": "alice",
  "amount_base_minor": 12345,
  "status": "pending",
  "created_at": "2026-07-24T06:33:57.085839+00:00"
}
```

- `amount_base_minor` — amount in INR minor units, computed on write. **Currently
  equals `amount_minor`** (FX conversion is a `TODO`; see "What it does").
- `status` — one of `pending`, `approved`, `rejected` (starts as `pending`).
- `created_at` — ISO-8601 UTC timestamp string.

### Query parameters — `GET /expenses`

| Param | Type | Effect |
|---|---|---|
| `status` | string | Return only expenses with this status. |
| `category` | string | Return only expenses in this category. |

Both are optional and may be combined; results are ordered by `id`.

### Response — `GET /reports/insights`

```json
{
  "insight": {
    "summary": "One-sentence overview of spending.",
    "bullets": ["Insight one.", "Insight two.", "Insight three."]
  }
}
```

`bullets` is always exactly three strings. With no data (or on any error) a safe
fallback object is returned.

### Examples

Create (Command Prompt with `curl`):

```bat
curl -X POST http://127.0.0.1:8000/expenses ^
  -H "Content-Type: application/json" ^
  -d "{\"description\":\"Team lunch\",\"amount_minor\":12345,\"currency\":\"INR\",\"category\":\"meals\",\"submitted_by\":\"alice\"}"
```

List only pending expenses:

```bat
curl "http://127.0.0.1:8000/expenses?status=pending"
```

Approve expense #1:

```bat
curl -X POST http://127.0.0.1:8000/expenses/1/approve
```
