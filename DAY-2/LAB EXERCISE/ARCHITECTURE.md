# ExpenseFlow — Architecture

A small expense submission + approval API (PoC). One user journey drives the design:
**submit an expense → normalise it to base currency (INR) → approve or reject it.**

Stack: Python 3.12, FastAPI, Uvicorn, SQLAlchemy ORM on SQLite (`expenseflow.db`), httpx for
the FX call, pydantic v2 for request/response models, pytest for tests. Money is stored as
integer minor units (never float). Base currency is INR; amounts are normalised to base on
write. Secrets come from environment variables via `python-dotenv`.

---

## 1. Schema — `expenses` table

Declared via SQLAlchemy ORM in `app/models.py`. SQLite is dynamically typed; the types below
are the SQLAlchemy column types (they also document intent).

| Column | Type | Null | Why it exists |
|---|---|---|---|
| `id` | `Integer` PK, autoincrement | no | Surrogate key. Stable handle for `/expenses/{id}` and decisions. |
| `employee` | `String` | no | Who submitted it. Free text — the brief has no auth/users, so no FK. |
| `description` | `String` | no | What the expense was for. |
| `category` | `String` | yes | Optional grouping (travel, meals…). Not every expense has one. |
| `original_amount_minor` | `Integer` | no | Amount **as submitted**, in *minor units of the original currency*. Integer per the "money is minor units, never float" rule. |
| `original_currency` | `String(3)` | no | ISO-4217 code of the submitted amount (e.g. `USD`, `JPY`). |
| `base_amount_minor` | `Integer` | **no** | Amount normalised to **INR paise**, computed at write time. `NOT NULL` structurally enforces the "normalised to base on write" invariant — see Edge case 1. |
| `base_currency` | `String(3)`, default `'INR'` | no | Records the base currency *at time of write*. Future-proofs a base change; keeps rows self-describing for audit. |
| `fx_rate` | `String` (decimal string, e.g. `"0.010852"`) | no | The exact original→INR rate used for this row. Stored as a **string**, not a float, so the conversion stays reproducible and auditable (no binary-float drift). `"1"` for same-currency. |
| `fx_rate_fetched_at` | `DateTime` | no | When the rate was obtained. Freshness + audit trail. |
| `fx_rate_source` | `String` | no | Provenance: `api` / `same-currency` / `cache`. Explains how the rate was derived. |
| `status` | `Enum(pending, approved, rejected)` (stored as text), default `pending` | no | The decision state machine. See Edge case 3. |
| `created_at` | `DateTime`, default now | no | Audit. |
| `decided_at` | `DateTime` | yes | Set once, when approved/rejected. `NULL` while pending. |
| `decided_by` | `String` | yes | Approver name (free text). `NULL` while pending. |

**Index:** on `status` (supports listing/filtering by state).

**Key invariant baked into the schema:** a row cannot exist without `base_amount_minor`.
There is no "unconverted expense" state — conversion is a precondition of persistence.

---

## 2. Endpoints

All handlers carry type hints and docstrings. Request/response bodies are pydantic v2 models
in `app/schemas.py`. Money crosses the wire as integer minor units.

| Method | Path | Request body | Success | Errors |
|---|---|---|---|---|
| `POST` | `/expenses` | `ExpenseCreate`: `{ employee, description, category?, amount_minor (>0), currency (ISO-4217) }` | `201` `ExpenseOut` | `422` invalid body; `502` FX unavailable |
| `GET` | `/expenses/{id}` | — | `200` `ExpenseOut` | `404` not found |
| `GET` | `/expenses` | — (optional `?status=` query filter) | `200` `list[ExpenseOut]` | — |
| `POST` | `/expenses/{id}/approve` | `DecisionIn`: `{ decided_by? }` | `200` `ExpenseOut` | `404` missing; `409` if not `pending` |
| `POST` | `/expenses/{id}/reject` | `DecisionIn`: `{ decided_by?, reason? }` | `200` `ExpenseOut` | `404` missing; `409` if not `pending` |
| `GET` | `/health` | — | `200` `{ "status": "ok" }` | — |

**`ExpenseOut`** returns the full row: original amount/currency, `base_amount_minor`,
`base_currency`, `fx_rate`, `fx_rate_source`, `status`, and timestamps.

- **`POST /expenses`** validates → obtains the FX rate (same-currency shortcut, else cache,
  else live httpx call) → converts to INR paise → persists with `status=pending`. Conversion
  happens **before** commit; if it can't be done, nothing is written (Edge case 1).
- **`approve` / `reject`** only act on a `pending` expense; they stamp `decided_at` +
  `decided_by` once and are otherwise `409` (Edge case 3).
- **`GET /health`** is a trivial liveness convenience, not a business endpoint.

---

## 3. File layout

- **`app/__init__.py`** — package marker.
- **`app/main.py`** — build the FastAPI app; `load_dotenv()`; a lifespan hook that calls `init_db()` (create tables) on startup; `include_router(...)` from `routes.py`.
- **`app/db.py`** — SQLAlchemy `engine` (`sqlite:///expenseflow.db`), `SessionLocal`, `Base`, a `get_db()` dependency, and `init_db()`.
- **`app/models.py`** — the `Expense` ORM model and the `Status` enum (`str, Enum`).
- **`app/schemas.py`** — pydantic v2 models: `ExpenseCreate`, `DecisionIn`, `ExpenseOut`, plus validators (ISO-4217 3-letter currency, `amount_minor > 0`).
- **`app/routes.py`** — one `APIRouter` with the endpoints; depends on `get_db` and calls into the FX helper.
- **`app/fx.py`** — the httpx FX client, the in-process rate cache, the currency-exponent map, and the pure conversion function. Isolating it keeps route handlers thin and makes the money math unit-testable without a network. *(Addition beyond the 5 files listed in `CLAUDE.md`; collapses into `routes.py` if strict layout is preferred.)*
- **`tests/`** — pytest suite (see Verification).

**Config** (via `python-dotenv`, no secrets hardcoded): `FX_API_URL`, `FX_API_KEY` (if the
provider needs one), `FX_TIMEOUT_SECONDS`, `FX_CACHE_TTL_SECONDS`. Base currency `INR` is a
constant. No new third-party dependency is introduced.

---

## 4. Edge-case decisions

### Edge case 1 — FX API is down at submit time: what happens to `base_amount_minor`?
**Decision: a row is never persisted without a successful conversion.** There is no expense
with a missing/null `base_amount_minor`.

`POST /expenses` resolves the rate in order:
1. **Same currency** (`currency == INR`): rate = `"1"`, source `same-currency`, **no network call**.
2. **Cache hit** (within `FX_CACHE_TTL_SECONDS`): use the cached rate, source `cache`.
3. **Live call**: httpx GET with a timeout and a small bounded retry.

If all fail, the handler **raises `502 Bad Gateway` and commits nothing** — the transaction is
rolled back, so no half-converted or null-base row is created. The client retries later. This
is enforced structurally by `base_amount_minor NOT NULL`: an unconverted expense is
unrepresentable.

*Rejected alternative:* persisting as `pending` with a null base and a "conversion_failed"
flag. That breaks the "normalise to base on write" invariant and would let an approver approve
an un-costed expense.

### Edge case 2 — Currencies don't all have 2 decimal places
Minor units are **not** universal: JPY has 0 decimals, most currencies 2, KWD/BHD 3. So
`original_amount_minor * rate` is **wrong** — ¥1000 (`amount_minor = 1000`, exp 0) is a very
different real amount than $10.00 (`amount_minor = 1000`, exp 2).

`app/fx.py` holds a small **currency-exponent map** and does the conversion in a single pure
function: original minor units → major units (using the source exponent) → multiply by the
rate → INR paise (using INR exponent 2), **rounding exactly once** to an integer. The rate is
kept as a decimal string and the result stored as an integer, so the computation is
deterministic and reproducible. Unknown currency codes are rejected at the `422` validation
layer before any math runs.

### Edge case 3 — Approving an already-decided expense: can you approve twice?
**Decision: no. The API rejects it.**

`status` is a state machine with a single terminal transition:
`pending → approved` **or** `pending → rejected`. Approved and rejected are terminal.

`approve` / `reject` first check the current status:
- if `pending` → apply the transition, stamp `decided_at` + `decided_by` **once**.
- if already `approved` or `rejected` → **`409 Conflict`**, with the current status in the
  message. No field changes; `decided_at` / `decided_by` are never overwritten.

This makes decisions idempotent-safe under client retries, prevents flipping
`approved → rejected` (or vice versa), and keeps a truthful audit trail. `404` is returned if
the expense doesn't exist.

---

## Verification

**Automated** (`python -m pytest -q`) — the FX API is faked with httpx's built-in
`MockTransport` (no new dependency), so tests never hit the network:
- Submit an **INR** expense → no network call, `fx_rate = "1"`, `base_amount_minor == original_amount_minor`.
- Submit a **foreign-currency** expense (mocked rate) → correct `base_amount_minor`.
- Submit a **JPY** (exp 0) expense → exponent handled correctly (Edge case 2).
- **FX down** (mock raises/times out) → `502` **and** assert no row was written (Edge case 1).
- Approve a `pending` → `200`, status `approved`, `decided_at` set.
- **Approve an already-approved** → `409`, fields unchanged (Edge case 3).
- Reject a `pending` → `200`; approve a `rejected` → `409`.
- `GET /expenses/{id}` unknown id → `404`.

**Manual smoke:**
- `python -m uvicorn app.main:app --reload`
- `curl` the journey: `POST /expenses` (foreign currency) → `GET /expenses/{id}` → `POST /expenses/{id}/approve` → re-`approve` and confirm `409`.
