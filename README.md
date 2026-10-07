# Intelligent Data Integration & Processing API

A production-style Python backend built for the Office Solution AI Labs take-home assignment. It ingests enterprise data from JSON, XML, CSV, and external REST APIs; normalises it into a relational schema; and exposes a full analytics + CRUD API.

---

## Business Scenario

Office Solution AI Labs receives enterprise data from multiple source systems. This service ingests, validates, transforms, normalises, stores, queries, and analyses data flowing from JSON (customers, orders), XML (shipments), CSV (products), and a public REST API (country data).

---

## Architecture

```
clients
  │
  ▼
FastAPI (app/main.py)
  │  ├── Routers   (app/api/)          — HTTP layer only; no business logic
  │  ├── Services  (app/services/)     — business logic & transformation pipeline
  │  ├── Repos     (app/repositories/) — all SQL via SQLAlchemy
  │  ├── Models    (app/models/)       — ORM table definitions
  │  ├── Schemas   (app/schemas/)      — Pydantic request/response models
  │  └── Utils     (app/utils/)        — logging, exceptions, validation helpers
  │
  ▼
SQLite (app.db)   ←→  data/ (JSON · XML · CSV)
```

The transformation pipeline lives entirely in `app/services/transformation.py` — separate from route handlers.

---

## Tech Stack

| Layer | Library |
|---|---|
| Web framework | FastAPI 0.111 |
| Data validation | Pydantic v2 |
| ORM | SQLAlchemy 2.x |
| Database | SQLite (compatible with PostgreSQL) |
| CSV processing | Pandas |
| Async HTTP | httpx |
| ASGI server | Uvicorn |
| Testing | pytest + pytest-asyncio |

---

## Folder Structure

```
backend/
├── app/
│   ├── main.py              # FastAPI app, routers, startup
│   ├── database.py          # Engine, session factory, Base
│   ├── dependencies.py      # Dependency-injection helpers
│   ├── seed.py              # Table creation + full data ingestion
│   ├── api/                 # Route handlers (thin — delegate to services)
│   │   ├── customers.py
│   │   ├── orders.py
│   │   ├── shipments.py
│   │   ├── analytics.py
│   │   ├── external.py
│   │   └── ingestion.py
│   ├── models/              # SQLAlchemy ORM models
│   ├── schemas/             # Pydantic schemas
│   ├── services/            # Business logic
│   │   ├── ingestion.py     # Orchestrates the full pipeline
│   │   ├── transformation.py# Parsers, validators, metric calculators
│   │   ├── analytics.py     # Thin service over analytics repository
│   │   └── external_api.py  # Async REST Countries client
│   ├── repositories/        # All direct database access
│   └── utils/               # Logging, exceptions, validation helpers
├── data/                    # Source data files
├── tests/                   # pytest test suite
├── requirements.txt
├── .env.example
└── README.md
```

---

## Database Schema

```
customers       products
│               │
└── orders ─────┘ (via order_items.product_id)
      │
      ├── order_items (FK → products)
      └── shipments

ingestion_jobs  (standalone — tracks background jobs)
```

**Tables**

| Table | PK | Notable columns |
|---|---|---|
| customers | customer_id (str) | region, email (unique) |
| products | product_id (str) | cost_price |
| orders | order_id (str) | customer_id (FK), order_date (indexed) |
| order_items | id (int autoincrement) | order_id (FK), product_id (FK), quantity, unit_price |
| shipments | shipment_id (str) | order_id (FK), warehouse, dispatch_date, delivery_date, status |
| ingestion_jobs | job_id (UUID str) | status, message, created_at, completed_at |

---

## Data Transformation Pipeline

```
customers.json  orders.json  products.csv  shipments.xml
      │               │             │              │
   load &          load &        pandas        ElementTree
   flatten       flatten items   parse          parse
      │               │             │              │
      └───────────────┴─────────────┴──────────────┘
                       │
                  validate each record
                  (required fields, FKs, dates, numeric)
                       │
              skip/log invalid records
                       │
              insert into DB (upsert-safe)
                       │
              IngestionJob → completed / failed
```

---

## API Endpoints

| Method | Path | Description |
|---|---|---|
| GET | /health | Health check |
| GET | /customers | List customers (page, page_size, region, name) |
| GET | /customers/{id} | Get customer by ID |
| GET | /orders | List orders (page, customer_id, dates, sort) |
| GET | /orders/{id} | Get order with items, financials, shipment |
| GET | /shipments | List shipments (status, warehouse, dates) |
| GET | /shipments/{id} | Get shipment with delivery duration |
| GET | /analytics/customers/{id} | Customer revenue/profit summary |
| GET | /analytics/products | Per-product analytics |
| GET | /analytics/regions | Per-region analytics |
| GET | /analytics/top-products?n=10 | Top N products by revenue (SQL aggregation) |
| GET | /external/country/{name} | Async REST Countries lookup |
| POST | /ingest | Trigger background ingestion job |
| GET | /jobs/{job_id} | Poll ingestion job status |

---

## Business Calculations

For every order item:

```
revenue        = quantity × unit_price
product_cost   = quantity × cost_price
profit         = revenue − product_cost
profit_margin  = (profit / revenue) × 100    [0 when revenue = 0]
```

For shipments:

```
delivery_duration = delivery_date − dispatch_date   (days)
                  = null  when delivery_date is missing or result is negative
```

---

## External API Integration

- Endpoint: `GET /external/country/{country_name}`
- Source: [https://restcountries.com/v3.1/name/{country}](https://restcountries.com/v3.1/name/{country})
- Uses `httpx.AsyncClient` with a configurable timeout (`EXTERNAL_API_TIMEOUT` env var, default 10 s)
- Retries up to 2 times on connection/timeout errors
- Returns only useful fields: name, capital, region, population, currencies
- Isolated in `app/services/external_api.py`

---

## Error Handling

All errors return a consistent JSON shape:

```json
{ "error": "ERROR_CODE", "message": "Human-readable description" }
```

| Code | HTTP status | Meaning |
|---|---|---|
| RESOURCE_NOT_FOUND | 404 | Entity does not exist |
| CONFLICT | 409 | Duplicate record |
| VALIDATION_ERROR | 422 | Input failed Pydantic validation |
| INVALID_PARAMETER | 400 | Bad query parameter |
| EXTERNAL_API_ERROR | 502 | Upstream REST API failure |
| DATABASE_ERROR | 500 | SQLAlchemy failure |
| INTERNAL_ERROR | 500 | Unexpected exception |

Raw stack traces are never exposed to clients.

---

## Testing

```
tests/
├── conftest.py           # In-memory SQLite fixtures; transactional rollback per test
├── test_customers.py     # Customer CRUD API tests
├── test_transformation.py# Parser, metric calculation, and validation unit tests
├── test_analytics.py     # Analytics endpoints + top-products
└── test_errors.py        # 404s, 400s, external API mocking, health check
```

---

## Setup Instructions

```bash
# 1. Clone / navigate to the project
cd backend

# 2. Create and activate a virtual environment
python -m venv venv

# Windows
venv\Scripts\activate

# macOS/Linux
source venv/bin/activate

# 3. Install dependencies
pip install -r requirements.txt

# 4. Copy environment template (optional — defaults work out of the box)
cp .env.example .env

# 5. Seed the database
python -m app.seed
```

---

## Running the Server

```bash
uvicorn app.main:app --reload
```

Server starts at [http://127.0.0.1:8000](http://127.0.0.1:8000)

---

## Running Tests

```bash
pytest
```

---

## Example API Requests

```bash
# Health
curl http://127.0.0.1:8000/health

# List customers
curl "http://127.0.0.1:8000/customers?page=1&page_size=20&region=North"

# Get a customer
curl http://127.0.0.1:8000/customers/C001

# List orders with filters
curl "http://127.0.0.1:8000/orders?customer_id=C001&start_date=2026-01-01&end_date=2026-12-31&sort_by=order_date&sort_order=desc"

# Get order with financials
curl http://127.0.0.1:8000/orders/O1001

# Shipments
curl "http://127.0.0.1:8000/shipments?status=Delivered"

# Customer analytics
curl http://127.0.0.1:8000/analytics/customers/C001

# Product analytics
curl http://127.0.0.1:8000/analytics/products

# Regional analytics
curl http://127.0.0.1:8000/analytics/regions

# Top 5 products by revenue
curl "http://127.0.0.1:8000/analytics/top-products?n=5"

# External country lookup
curl http://127.0.0.1:8000/external/country/India

# Trigger ingestion job
curl -X POST http://127.0.0.1:8000/ingest

# Poll job status (replace with actual job_id from above)
curl http://127.0.0.1:8000/jobs/<job_id>
```

---

## Swagger / ReDoc

- Swagger UI: [http://127.0.0.1:8000/docs](http://127.0.0.1:8000/docs)
- ReDoc: [http://127.0.0.1:8000/redoc](http://127.0.0.1:8000/redoc)

---

## Design Decisions

- **SQLite with SQLAlchemy 2.x** — zero-config locally; switching to PostgreSQL requires only changing `DATABASE_URL`.
- **Repository pattern** — all SQL is isolated in `repositories/`; services never write raw queries.
- **Service layer** — business logic (calculations, validation, orchestration) lives in `services/`; routes are thin.
- **BackgroundTasks** — ingestion jobs run asynchronously without Celery or Redis, keeping the project runnable in one step.
- **Transactional rollback in tests** — each test wraps its session in a transaction that is rolled back, so tests are isolated and fast.
- **Pandas only for CSV** — used only in `load_products_csv` / `load_transactions_csv_safe`; not dragged into non-CSV paths.

---

## Performance Considerations

- Analytics endpoints (top-products, regions, products) perform all aggregation in SQL with `SUM`, `GROUP BY`, `ORDER BY`, `LIMIT` — Python never loads all rows.
- Pagination is applied at the database level with `OFFSET`/`LIMIT`.
- Indexes are defined on `customer_id`, `order_date`, `product_id`, `shipment.status`, and `shipment.warehouse`.
- The external API client uses a single `httpx.AsyncClient` per request with connection pooling and a hard timeout.

---

## Async Explanation

FastAPI runs on an ASGI server (Uvicorn). Route handlers declared with `async def` run on the event loop, allowing non-blocking I/O. The `/external/country/{name}` route is fully async: it uses `httpx.AsyncClient` so the event loop is free to serve other requests while waiting for the upstream REST Countries API. Regular CRUD routes use synchronous SQLAlchemy sessions (FastAPI handles them in a thread pool via `run_in_executor`).
