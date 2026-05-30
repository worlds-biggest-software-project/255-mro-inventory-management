# MRO Inventory Management — Phased Development Plan

> Project: 255-mro-inventory-management · Created: 2026-05-29
> Purpose: Provide sufficient detail for Claude Code (Opus) to implement each phase end-to-end.

This plan synthesises `research.md`, `features.md`, `standards.md`, `README.md`, and `data-model-suggestion-1.md` (normalized relational, augmented with JSONB extensibility from `data-model-suggestion-3.md`). It targets an open-core, self-hostable, AI-native MRO inventory platform: real-time multi-location inventory, work-order/asset linkage, procurement, GS1 barcode workflows, and an AI demand-forecasting engine that emits confidence intervals — the transparency gap incumbents leave open.

---

## Technology Decisions

| Concern | Choice | Rationale |
|---------|--------|-----------|
| Primary language (backend) | Python 3.12 | The product is AI-heavy: intermittent-demand forecasting (Croston/SBA), ML catalogue dedup, supplier-risk scoring. Python has the mature stack (`statsforecast`, `scikit-learn`, `sentence-transformers`, `numpy`) and first-class LLM SDKs. One language for API + ML avoids a service-boundary split in the MVP. |
| API framework | FastAPI | Async, Pydantic-native, and auto-generates the **OpenAPI 3.1** spec required by standards.md for LLM tool-use and third-party integration. Dependency-injection model suits per-request tenant scoping. |
| Data validation / schemas | Pydantic v2 | Single source of truth for request/response models, config, and JSON Schema (Draft 2020-12) emission for wire payloads. |
| ORM / migrations | SQLAlchemy 2.0 (async) + Alembic | Mature async ORM; Alembic gives reproducible migrations for the normalized schema (Model 1) which explicitly requires migrations per its trade-offs. |
| Database | PostgreSQL 16 | Model 1 assumes PG features: `gen_random_uuid()`, `JSONB`, generated columns, partial indexes, GiST/`pg_trgm` for fuzzy part search, row-level security for multi-tenancy. Single DB serves OLTP + analytical roll-ups for the MVP. |
| Fuzzy / semantic search | `pg_trgm` (MVP) → `pgvector` (v1.1) | Trigram indexes power "natural language-ish" part lookup in MVP with zero new infra; `pgvector` + embeddings adds true semantic search and dedup in v1.1 without leaving Postgres. |
| Task queue | Celery + Redis | Forecasting runs, IoT-triggered work-order generation, barcode batch imports, and supplier-risk scoring are async/scheduled. Celery Beat handles nightly forecast and scoring jobs. Redis doubles as cache + broker. |
| LLM / AI gateway | `anthropic` SDK behind a thin `LLMClient` abstraction | NL part search, knowledge capture (v1.1+), and the MCP server need an LLM. Abstraction allows provider swap and prompt caching. Forecasting itself is statistical, not LLM — keeps it cheap and explainable. |
| Frontend | React 18 + TypeScript + Vite, TanStack Query, shadcn/ui + Tailwind | Dashboard-first SPA matching incumbent UX patterns. TanStack Query maps cleanly to the REST resources. Mobile barcode flows are responsive PWA, not a separate native app (MVP). |
| Barcode decoding (client) | `@zxing/browser` | GS1 EAN-13/Code-128/DataMatrix decoding in-browser for the PWA receiving/issue flows; no native build needed. |
| Auth | OAuth 2.0 Authorization Code + PKCE (RFC 7636) via Authlib; JWT access tokens | standards.md mandates PKCE for all client types in 2026. OIDC-ready for enterprise SSO later. |
| AuthZ | Role-based (admin/manager/planner/technician/viewer) + Postgres row-level security on `org_id` | Multi-tenant isolation; mitigates OWASP API1 (BOLA) at the database layer. |
| MCP server | Python MCP SDK (`mcp`), separate ASGI app | standards.md flags that **no incumbent ships an MCP server** — a differentiator. Exposes stock query, part search, PO creation, WO generation as tools. |
| Containerisation | Docker + docker-compose | README targets self-hosted + cloud. Compose wires api, worker, beat, postgres, redis, web for one-command local/self-host bring-up. |
| Testing | pytest + pytest-asyncio + httpx.AsyncClient; Testcontainers (Postgres) for integration; Vitest + Testing Library (frontend) | Standard Python stack; Testcontainers gives real-Postgres integration tests for RLS, constraints, generated columns. |
| Code quality | Ruff (lint+format), mypy (strict), pre-commit; ESLint + Prettier (frontend) | Fast, modern toolchain; mypy strict keeps the Pydantic/SQLAlchemy boundary honest. |
| Package manager | `uv` (backend), `pnpm` (frontend) | `uv` for fast reproducible Python installs; `pnpm` for the web workspace. |
| Key libraries | `statsforecast` (Croston/SBA/ADIDA), `scikit-learn`, `sentence-transformers` (v1.1), `python-barcode`/GS1 AI parser, `PyJWT`, `tenacity` | Domain-specific: intermittent-demand models, ML scoring, GS1 Application Identifier parsing, retry/backoff for external APIs (GS1 lookup, ERP). |

### Project Structure

```
mro-inventory/
├── pyproject.toml                  # uv-managed; ruff, mypy, pytest config
├── alembic.ini
├── Dockerfile                      # api/worker image
├── docker-compose.yml              # api, worker, beat, postgres, redis, web
├── .env.example
├── README.md
├── openapi/                        # exported OpenAPI 3.1 snapshots (CI artefact)
│
├── src/mro/
│   ├── main.py                     # FastAPI app factory, router registration
│   ├── config.py                   # Pydantic Settings (env-driven)
│   ├── db/
│   │   ├── base.py                 # async engine, session, Base
│   │   ├── models/                 # SQLAlchemy ORM (one module per domain)
│   │   │   ├── asset.py  parts.py  inventory.py  workorder.py
│   │   │   ├── procurement.py  forecasting.py  org.py  taxonomy.py
│   │   └── rls.py                  # row-level security helpers / tenant guc
│   ├── schemas/                    # Pydantic request/response models per domain
│   ├── api/
│   │   ├── deps.py                 # auth, current_user, tenant, pagination
│   │   ├── routers/                # FastAPI routers, one per resource
│   │   └── errors.py               # Problem+JSON error model
│   ├── services/                   # business logic (transaction orchestration)
│   │   ├── inventory_service.py  procurement_service.py
│   │   ├── workorder_service.py  transfer_service.py
│   ├── ai/
│   │   ├── forecasting/            # Croston/SBA/ensemble, ReorderRecommender
│   │   ├── dedup/                  # catalogue dedup (v1.1)
│   │   ├── supplier_risk/          # lead-time risk scoring (v1.1)
│   │   ├── nl_search/              # NL part search (v1.1)
│   │   └── llm_client.py
│   ├── integrations/
│   │   ├── gs1.py                  # GS1 AI parser + GS1 US lookup client
│   │   ├── iot/                    # sensor webhook ingest (v1.1)
│   │   └── erp/                    # ERP connector interface (v1.2)
│   ├── workers/
│   │   ├── celery_app.py  tasks.py  schedules.py
│   ├── mcp/
│   │   └── server.py               # MCP tool server (v1.1)
│   └── auth/
│       ├── oauth.py  jwt.py  rbac.py
│
├── migrations/                     # Alembic versions
├── seeds/                          # UNSPSC/eClass/UoM reference data loaders
├── tests/
│   ├── unit/  integration/  e2e/  fixtures/  conftest.py
│
└── web/
    ├── package.json
    └── src/
        ├── api/        # generated client from OpenAPI
        ├── routes/     # dashboard, inventory, parts, work-orders, po, forecasts
        ├── components/ # shadcn-based
        └── scan/       # PWA barcode receive/issue
```

---

## Phase 1: Foundation — Project Skeleton, Config, DB, Auth

### Purpose
Establish the runnable backbone: a FastAPI app with health checks, async Postgres via SQLAlchemy/Alembic, multi-tenant org model with row-level security, OAuth2+PKCE auth, RBAC, a Problem+JSON error contract, and Docker compose for one-command bring-up. Nothing domain-specific ships here, but every later phase depends on tenant scoping, auth, and migrations existing.

### Tasks

#### 1.1 — App factory, config, health, error contract

**What**: A FastAPI application with environment-driven settings, `/healthz` and `/readyz`, and a uniform error response.

**Design**:
```python
# config.py
class Settings(BaseSettings):
    database_url: PostgresDsn
    redis_url: RedisDsn
    jwt_secret: SecretStr
    jwt_algorithm: str = "HS256"
    access_token_ttl_seconds: int = 3600
    oauth_issuer: str = "mro-inventory"
    anthropic_api_key: SecretStr | None = None
    environment: Literal["dev", "test", "prod"] = "dev"
    model_config = SettingsConfigDict(env_file=".env", env_prefix="MRO_")

# api/errors.py  — RFC 9457 Problem Details
class ProblemDetail(BaseModel):
    type: str = "about:blank"
    title: str
    status: int
    detail: str | None = None
    instance: str | None = None
    errors: list[dict] | None = None   # field-level validation errors
```
- Register an exception handler mapping `RequestValidationError`, `HTTPException`, and a custom `DomainError` hierarchy to `ProblemDetail`.
- `/healthz` returns 200 unconditionally; `/readyz` checks DB + Redis connectivity.

**Testing**:
- `Unit: Settings loads from env with MRO_ prefix → fields populated, defaults applied`
- `Unit: missing DATABASE_URL → ValidationError naming database_url`
- `Integration: GET /healthz → 200 {"status":"ok"}`
- `Integration (Testcontainers): GET /readyz with DB+Redis up → 200; with DB down → 503 ProblemDetail`
- `Integration: POST body failing validation → 422 ProblemDetail with errors[].loc populated`

#### 1.2 — Async DB layer, Base model, Alembic

**What**: Async engine/session factory, a declarative `Base` with shared columns, and Alembic wired for autogenerate.

**Design**:
```python
# db/base.py
class Base(DeclarativeBase): pass

class TimestampMixin:
    created_at: Mapped[datetime] = mapped_column(server_default=func.now())
    updated_at: Mapped[datetime] = mapped_column(server_default=func.now(), onupdate=func.now())

class TenantMixin:
    org_id: Mapped[UUID] = mapped_column(ForeignKey("organisation.id"), index=True)
```
- `async_sessionmaker` bound to `create_async_engine(settings.database_url)`.
- `get_session()` FastAPI dependency yields a session, commits on success, rolls back on exception.
- Alembic `env.py` runs async, imports all model modules so autogenerate sees them.

**Testing**:
- `Integration (Testcontainers): alembic upgrade head then downgrade base → no errors, schema empty`
- `Integration: get_session commits on success; rolls back when handler raises`
- `Unit: TenantMixin/TimestampMixin produce expected columns on a sample model`

#### 1.3 — Organisation, user, audit log, row-level security

**What**: `organisation`, `app_user`, `audit_log` tables and an RLS scheme that scopes all tenant tables to the current org.

**Design** (from Model 1 §Multi-Tenant, with RLS):
```sql
-- organisation, app_user(role IN admin|manager|planner|technician|viewer), audit_log as in data-model-1
-- RLS pattern applied to every TenantMixin table:
ALTER TABLE <t> ENABLE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation ON <t>
  USING (org_id = current_setting('mro.current_org')::uuid);
```
- `db/rls.py`: `await session.execute(text("SET LOCAL mro.current_org = :org"), {"org": org_id})` at request start, set from the JWT claim.
- `audit_log` written by a SQLAlchemy event listener capturing INSERT/UPDATE/DELETE old/new JSONB on flagged models.

**Testing**:
- `Integration (Testcontainers): two orgs, set mro.current_org=A → query returns only A's rows`
- `Integration: insert without org guc set → 0 rows visible / write blocked`
- `Integration: update a part → audit_log row with action=UPDATE, old_values, new_values`
- `Unit: audit listener serialises only whitelisted columns`

#### 1.4 — OAuth2 Authorization Code + PKCE, JWT, RBAC

**What**: PKCE auth flow issuing JWT access tokens, plus a role-checking dependency.

**Design**:
- Endpoints: `GET /oauth/authorize` (validates `code_challenge`, `code_challenge_method=S256`), `POST /oauth/token` (verifies `code_verifier`), per RFC 7636.
- JWT claims: `sub` (user id), `org` (org id), `role`, `exp`, `iss`.
- `deps.py`:
```python
async def current_user(token=Depends(oauth2_scheme)) -> AuthUser: ...
def require_role(*roles: Role):
    async def _dep(user: AuthUser = Depends(current_user)):
        if user.role not in roles: raise Forbidden()
        return user
    return _dep
```
- Role hierarchy enforced at router level; write endpoints require ≥ planner, admin endpoints require admin.

**Testing**:
- `Unit: S256 challenge/verifier pair validates; mismatched verifier → invalid_grant`
- `Unit: expired JWT → 401; tampered signature → 401`
- `Integration: full authorize→token→call protected endpoint with Bearer → 200`
- `Integration: technician calls planner-only endpoint → 403 ProblemDetail`

### Phase 1 done when: app boots via compose, migrations apply, a user can complete PKCE login and call a protected stub endpoint scoped to their org, RLS verified across two orgs.

---

## Phase 2: Core Catalogue — Sites, Equipment, Parts, Taxonomy

### Purpose
Build the master data the whole system references: sites/functional locations, equipment classes, the parts catalogue, units of measure, and UNSPSC/eClass taxonomy with cross-mapping. After this phase the system can hold a real MRO catalogue and answer "what parts exist, classified how, for which equipment".

### Tasks

#### 2.1 — Sites, functional locations, equipment classes

**What**: CRUD for `site`, `functional_location` (self-referential hierarchy with materialised `path`), and `equipment_class` (ISO 14224 codes).

**Design**:
- ORM models per Model 1 §Asset & Equipment.
- `functional_location.path` maintained on insert/update as `<parent.path>/<code>`; expose tree via `GET /sites/{id}/locations?tree=true`.
- Endpoints (all tenant-scoped, paginated, RFC 8288 `Link` headers):
  `POST/GET/PATCH/DELETE /sites`, `/functional-locations`, `/equipment-classes`.

**Testing**:
- `Unit: path recomputed when parent changes`
- `Integration: create nested locations → tree endpoint returns correct nesting`
- `Integration: paginated list → Link: rel="next" present until last page`
- `Integration: equipment_class with duplicate ISO code → 409`

#### 2.2 — Parts catalogue + units of measure + alternates + asset BOM

**What**: `part`, `unit_of_measure`, `part_alternate`, `asset_bom` with trigram search on part name.

**Design** (Model 1 §Parts, plus a JSONB `attributes` column from Model 3 for industry-specific fields):
```python
class Part(Base, TenantMixin, TimestampMixin):
    part_number: Mapped[str]            # unique per org
    name: Mapped[str]
    gtin: Mapped[str | None]            # GS1 GTIN-14
    unspsc_commodity_id: Mapped[UUID | None]
    eclass_class_id: Mapped[UUID | None]
    unit_of_measure_id: Mapped[UUID]
    is_rotable: Mapped[bool] = False
    is_serialized: Mapped[bool] = False
    lead_time_days: Mapped[int | None]
    criticality: Mapped[str | None]     # critical|essential|standard|non_critical
    status: Mapped[str] = "active"
    attributes: Mapped[dict] = mapped_column(JSONB, default=dict)  # industry-specific
```
- GiST trigram index on `name`; `GET /parts?q=<text>` ranks by `similarity(name, q)`.
- `asset_bom` junction enforces `UNIQUE(asset_id, part_id, position_code)`.

**Testing**:
- `Unit: part_number uniqueness scoped to org (same number in two orgs OK)`
- `Integration: GET /parts?q="bearng" → returns "bearing" rows ranked by similarity`
- `Integration: link part to asset BOM; duplicate position_code → 409`
- `Integration: GTIN-14 with bad check digit rejected by validator`

#### 2.3 — Taxonomy tables + seed loaders (UNSPSC/eClass)

**What**: The four-level UNSPSC hierarchy, `eclass_class`, `taxonomy_crossmap`, and idempotent seed loaders.

**Design**:
- Tables per Model 1 §Taxonomy.
- `seeds/load_unspsc.py` ingests a CSV of segment/family/class/commodity codes; idempotent upsert by code.
- `GET /taxonomy/unspsc?level=commodity&parent=<code>` for drill-down; `GET /taxonomy/crossmap?unspsc=<code>` returns eClass candidates with `confidence`.

**Testing**:
- `Integration: load UNSPSC seed twice → no duplicates (idempotent)`
- `Integration: drill-down from segment → families → classes → commodities`
- `Unit: crossmap confidence constrained to [0,1]`
- `Fixture: 50-row UNSPSC sample committed under tests/fixtures`

### Phase 2 done when: a catalogue of parts can be created, classified to UNSPSC/eClass, linked to assets via BOM, and fuzzy-searched by name; reference data seeds load idempotently.

---

## Phase 3: Inventory Core — Storerooms, Balances, Transactions

### Purpose
The operational heart: track stock by storeroom/bin, record every movement as an append-only transaction, and keep `inventory_balance` consistent transactionally. This is the table-stakes capability every competitor has and the substrate the AI builds on.

### Tasks

#### 3.1 — Storerooms, bins, inventory balances

**What**: `storeroom`, `bin_location`, `inventory_balance` with reorder fields and low-stock partial index.

**Design** (Model 1 §Inventory):
- `inventory_balance` holds `quantity_on_hand/reserved/on_order`, `reorder_point`, `reorder_quantity`, `maximum_level`, `safety_stock`, `average_monthly_usage`, `UNIQUE(part_id, storeroom_id)`.
- Partial index `WHERE quantity_on_hand <= reorder_point` powers fast low-stock queries.
- `GET /inventory?storeroom=&low_stock=true` lists balances.

**Testing**:
- `Integration: create balance; CHECK quantity_on_hand >= 0 enforced`
- `Integration: low_stock=true returns only balances at/below reorder_point`
- `Integration: duplicate (part, storeroom) balance → 409`

#### 3.2 — Inventory transaction service (movement engine)

**What**: A service that records `inventory_transaction` rows and atomically updates `inventory_balance`.

**Design**:
```python
# services/inventory_service.py
async def post_transaction(session, *, part_id, storeroom_id, txn_type, quantity,
                           unit_cost=None, work_order_id=None, lot=None, serial=None,
                           performed_by) -> InventoryTransaction
```
- Transaction types: `receipt, issue, return, adjustment, transfer_out, transfer_in, cycle_count, scrap, reservation, unreservation`.
- Signed effect on `quantity_on_hand`/`quantity_reserved` per type, computed in one DB transaction with `SELECT ... FOR UPDATE` on the balance row to prevent oversell.
- Issue that would drive on-hand below zero → `InsufficientStock` domain error (mapped to 409).
- Updates `last_issue_date`/`last_receipt_date`/`last_count_date` accordingly.
- Endpoints: `POST /inventory/transactions`, `GET /inventory/transactions?part=&storeroom=&type=&from=&to=`.

**Testing**:
- `Unit: each txn_type maps to correct signed balance delta`
- `Integration: receipt 10 then issue 4 → on_hand 6, two txn rows`
- `Integration: issue 5 when on_hand 3 → 409 InsufficientStock, no txn written, balance unchanged`
- `Integration (concurrency): two simultaneous issues racing on_hand=1 → one succeeds, one 409 (FOR UPDATE holds)`
- `Integration: reservation increments quantity_reserved without changing on_hand`

#### 3.3 — Lot & serial tracking, cycle counts

**What**: `part_lot` (GS1 AI 10) and `part_serial` (GS1 AI 21) with lifecycle, plus cycle-count adjustment flow.

**Design**:
- `part_serial.status`: `in_stock|installed|in_repair|scrapped|in_transit`; `condition_code`: `new|serviceable|unserviceable|repairable|condemned`.
- Issuing a serialized part flips serial → `installed` and links `asset_id`.
- Cycle count posts a `cycle_count` transaction equal to (counted − on_hand) and sets `last_count_date`.

**Testing**:
- `Integration: issue serialized part → serial.status=installed, asset_id set, on_hand−1`
- `Integration: cycle count of 8 against on_hand 10 → adjustment txn of −2, balance 8`
- `Integration: lot expiration_date persisted; FEFO query orders by expiration`

#### 3.4 — Low-stock alerts

**What**: Alert generation when a movement drops on_hand to/below reorder_point.

**Design**:
- After each balance update, `inventory_service` emits a `low_stock` event (Redis pub/sub) and upserts an `alert` row (`type, part_id, storeroom_id, severity, status=open, created_at`).
- `GET /alerts?status=open&type=low_stock`; acknowledge via `PATCH /alerts/{id}`.

**Testing**:
- `Integration: issue crossing reorder_point → open low_stock alert created once (not duplicated on repeat)`
- `Integration: replenishment above reorder_point → alert auto-resolves`

### Phase 3 done when: stock can be received, issued, returned, transferred, counted, and reserved with a correct audit trail; balances never go negative; low-stock alerts fire and resolve.

---

## Phase 4: Procurement & Suppliers

### Purpose
Close the replenishment loop: suppliers, per-part pricing/lead times, purchase orders, and goods receipt that feeds back into inventory. After this phase the system can take a low-stock alert all the way to a received PO and recorded delivery performance (the data the v1.1 supplier-risk model needs).

### Tasks

#### 4.1 — Suppliers and supplier-part pricing

**What**: `supplier` and `supplier_part` (price, lead time, MOQ, preferred flag, validity window).

**Design** (Model 1 §Procurement):
- `UNIQUE(supplier_id, part_id)`; `is_preferred` partial index for default-supplier resolution.
- `GET /parts/{id}/suppliers` returns ranked options (preferred first, then price).

**Testing**:
- `Integration: two suppliers for a part, one preferred → preferred ranked first`
- `Integration: supplier rating outside [0,5] → 422`

#### 4.2 — Purchase orders + lines

**What**: `purchase_order` and `purchase_order_line` with generated `line_total` and a status state machine.

**Design**:
- States: `draft → submitted → approved → partially_received → received` (+ `cancelled` from any non-terminal).
- `line_total` is a Postgres `GENERATED ALWAYS AS (quantity_ordered*unit_price) STORED` column.
- Approval requires role ≥ manager; transitions validated by a state-machine guard rejecting illegal jumps (e.g. draft→received).
- Endpoints: `POST/GET /purchase-orders`, `POST /purchase-orders/{id}/submit|approve|cancel`, `GET/POST /purchase-orders/{id}/lines`.

**Testing**:
- `Unit: state machine allows draft→submitted→approved; rejects draft→received`
- `Integration: line_total computed by DB (read-only to API)`
- `Integration: technician attempts approve → 403`
- `Integration: cancel an approved-but-unreceived PO → status cancelled`

#### 4.3 — Goods receipt → inventory + delivery performance

**What**: Receiving a PO line posts a `receipt` inventory transaction and logs `supplier_delivery_performance`.

**Design**:
```python
# services/procurement_service.py
async def receive_line(session, *, po_line_id, quantity, lot=None, serial=None, performed_by):
    # 1. post_transaction(type=receipt, qty, link po_line)
    # 2. quantity_received += qty; recompute PO status (partial/received)
    # 3. insert supplier_delivery_performance(promised vs actual lead time, on_time, in_full)
```
- `actual_lead_time_days = receipt_date - order_date`; `on_time = receipt_date <= expected_delivery`; `in_full = quantity_received >= quantity_ordered`.

**Testing**:
- `Integration: receive 10 of 10 → balance +10, PO line/header → received, perf row on_time/in_full set`
- `Integration: receive 4 of 10 → PO partially_received, in_full=false`
- `Integration: receipt links inventory_transaction.purchase_order_line_id`

### Phase 4 done when: a PO can be drafted, approved, and received; receipts increment stock and record delivery performance; the full alert→PO→receipt loop works end to end.

---

## Phase 5: AI Demand Forecasting Engine (Core Differentiator)

### Purpose
Replace static min/max thresholds with intermittent-demand forecasting that emits **point estimates plus confidence intervals** and recommends reorder points/quantities — directly addressing the transparency gap in standards.md and the MVP "basic AI demand forecasting" requirement. This is the product's headline value.

### Tasks

#### 5.1 — Consumption history extraction

**What**: Build per-(part, storeroom) demand time series from `inventory_transaction` issue/scrap rows.

**Design**:
```python
# ai/forecasting/history.py
def build_demand_series(txns: list[Txn], bucket="W") -> pd.Series  # weekly buckets, gaps→0
```
- Aggregate `issue` + work-order consumption into period buckets; preserve intermittency (zeros matter for Croston).

**Testing**:
- `Unit: 3 issues across 2 weeks → correct weekly buckets, empty weeks = 0`
- `Unit: returns subtracted from demand`
- `Fixture: committed intermittent series replays deterministically`

#### 5.2 — Forecast models (Croston, SBA, ensemble) + confidence intervals

**What**: A `Forecaster` producing `point_estimate`, `confidence_lower/upper`, `confidence_level`.

**Design**:
```python
class ForecastResult(BaseModel):
    method: Literal["croston","sba","ml_ensemble","moving_average"]
    point_estimate: float
    confidence_lower: float
    confidence_upper: float
    confidence_level: float = 0.90
    model_version: str

class Forecaster:
    def fit_predict(self, series, horizon_days, method="auto") -> ForecastResult
```
- `method="auto"`: choose Croston/SBA for intermittent (ADI > 1.32 and CV² < 0.49 ⇒ smooth; else intermittent/lumpy — Syntetos-Boylan classification), moving_average for fast-movers.
- Intervals via empirical residual bootstrap (no LLM — explainable, cheap). Persist to `demand_forecast` (Model 1 §AI).

**Testing**:
- `Unit: intermittent series classified → Croston/SBA selected`
- `Unit: confidence_lower <= point_estimate <= confidence_upper`
- `Unit: zero-demand series → point_estimate≈0, finite interval`
- `Integration: forecast persisted with method + model_version`

#### 5.3 — Reorder-point recommender

**What**: Convert a forecast + lead time + target service level into recommended reorder point and quantity.

**Design**:
```python
# ai/forecasting/reorder.py
def recommend(forecast: ForecastResult, lead_time_days: int,
              service_level: float = 0.95) -> ReorderRecommendation
```
- `reorder_point = lead_time_demand + safety_stock`; safety stock from the forecast interval width scaled to `service_level` (z-factor). EOQ-style `reorder_quantity` from average demand and (configurable) holding/order cost ratio.
- Writes `recommended_reorder_point/qty` onto `demand_forecast`; a separate **apply** step updates `inventory_balance` only on user approval (never silently).

**Testing**:
- `Unit: longer lead time → higher reorder_point`
- `Unit: higher service_level → wider safety stock`
- `Integration: apply recommendation → inventory_balance.reorder_point updated, audit logged`

#### 5.4 — Nightly forecast job + API

**What**: Celery Beat job forecasting all active parts; endpoints to read forecasts and trigger ad-hoc runs.

**Design**:
- `workers/tasks.py: run_forecasts(org_id)` scheduled nightly; chunked over parts; idempotent per `forecast_date`.
- `GET /parts/{id}/forecast` (latest), `GET /forecasts?storeroom=&from=&to=`, `POST /parts/{id}/forecast:run`.
- Response surfaces interval + method + `model_version` so the UI can show confidence — the differentiator.

**Testing**:
- `Integration (mocked Celery eager): run_forecasts over 3 parts → 3 demand_forecast rows`
- `Integration: GET latest forecast returns interval fields`
- `Integration: re-run same day → upsert, no duplicate row`

### Phase 5 done when: every active part gets a nightly forecast with confidence intervals and a reorder recommendation; recommendations are explainable and applied only on approval.

---

## Phase 6: Web Application & Barcode PWA

### Purpose
Deliver the dashboard-first SPA and the mobile barcode flows that make the platform usable by managers and frontline technicians — matching incumbent UX while exposing AI confidence that incumbents hide.

### Tasks

#### 6.1 — App shell, auth, generated API client

**What**: React+Vite shell with PKCE login, route guards, and a typed client generated from the OpenAPI spec.

**Design**:
- `openapi-typescript` generates `web/src/api`; TanStack Query hooks per resource.
- Route guard reads role from token; planner+ sees write actions, viewer is read-only.

**Testing**:
- `Vitest: login redirect on 401; role guard hides write buttons for viewer`
- `Component: API hook error → ProblemDetail surfaced in toast`

#### 6.2 — Inventory, parts, work-order, PO screens

**What**: List/detail/edit screens for the core resources with low-stock and search.

**Design**:
- Inventory dashboard: low-stock table, storeroom filter, quick receive/issue.
- Parts: trigram search box, classification chips, BOM tab.
- PO: line editor with state-machine-driven action buttons.

**Testing**:
- `Component: low-stock table renders alert severities; ack removes row`
- `Component: PO in draft shows Submit; in approved shows Receive`
- `E2E (Playwright, mocked API): search part → open detail → view forecast chart`

#### 6.3 — Forecast visualisation with confidence band

**What**: Chart showing point estimate + shaded confidence interval + recommended reorder line.

**Design**:
- Recharts area for the interval band, line for point estimate, marker for current on-hand and recommended reorder point; method + model_version shown as caption.

**Testing**:
- `Component: given ForecastResult → band spans lower..upper, reorder line at recommended value`

#### 6.4 — Barcode receive/issue PWA

**What**: Mobile-responsive scan flow using the device camera to parse GS1 barcodes for receiving and issue.

**Design**:
- `@zxing/browser` decodes; `integrations/gs1` (Phase 7) parses AIs (01 GTIN, 10 lot, 21 serial); resolves to a part; posts receipt/issue transaction.
- Offline-tolerant: queue scans in IndexedDB, flush when online.

**Testing**:
- `Component: decoded GS1-128 with AI 01+10 → correct GTIN+lot parsed and shown`
- `E2E (mocked camera): scan → confirm → POST /inventory/transactions called`
- `Component: offline scan queued, flushed on reconnect`

### Phase 6 done when: managers operate inventory/PO/forecast screens in the browser and technicians can receive/issue parts by scanning on a phone, including offline.

---

## Phase 7: GS1 Integration & Catalogue Quality (v1.1)

### Purpose
Harden barcode handling and tackle the underserved catalogue-quality space: GS1 AI parsing/validation, GS1 US lookup, and ML-based near-duplicate SKU detection — Verdantis-class capability, open-source.

### Tasks

#### 7.1 — GS1 Application Identifier parser + GTIN validation + GS1 US lookup

**What**: Parse GS1-128/DataMatrix AI strings, validate GTIN check digits, and optionally enrich via GS1 US Product Lookup API.

**Design**:
```python
# integrations/gs1.py
def parse_gs1(payload: str) -> dict   # {"01": gtin, "10": lot, "21": serial, "17": expiry}
def validate_gtin(gtin: str) -> bool  # mod-10 check digit
class Gs1UsClient:                      # OpenAPI-based; tenacity retry/backoff
    async def lookup(self, gtin) -> ProductInfo | None
```

**Testing**:
- `Unit: parse "01093123456789031715... " → correct AI dict`
- `Unit: valid/invalid GTIN check digits`
- `Integration (mocked HTTP): lookup hit → ProductInfo; 404 → None; 5xx → retried then raised`

#### 7.2 — ML near-duplicate SKU detection

**What**: Detect duplicate/near-duplicate parts across the catalogue and flag `status='duplicate_suspect'`.

**Design**:
- Embed `name + manufacturer + manufacturer_part_number` with `sentence-transformers`; store vectors in `pgvector`.
- Candidate pairs via cosine similarity threshold; secondary check on manufacturer part number normalisation.
- `GET /parts/duplicates` returns clusters with similarity scores; merge endpoint `POST /parts/{id}/merge` re-points BOM/inventory/transactions to the survivor.

**Testing**:
- `Unit: "1/2in SS Bolt" vs "0.5\" Stainless Bolt" → high similarity (mocked embeddings)`
- `Integration: merge re-points asset_bom and inventory_balance, marks loser obsolete`
- `Integration: distinct parts below threshold not flagged`

### Phase 7 done when: GS1 barcodes are robustly parsed/validated, optionally enriched, and the catalogue surfaces and can merge near-duplicate SKUs.

---

## Phase 8: IoT Ingestion & Predictive Work Orders (v1.1)

### Purpose
Close the condition-to-parts loop: ingest sensor/condition events and auto-generate work orders with pre-populated parts lists from asset BOMs — the eMaint/Maximo capability, delivered open.

### Tasks

#### 8.1 — Sensor event ingestion

**What**: A webhook + queue for condition/anomaly events keyed to assets.

**Design**:
```python
class SensorEvent(BaseModel):
    asset_id: UUID; metric: str; value: float; severity: Literal["info","warn","critical"]; ts: datetime
# POST /iot/events  (HMAC-signed) → enqueue Celery task
```
- HMAC signature verification on the webhook (reject invalid → 401, no enqueue).

**Testing**:
- `Integration (mocked): valid HMAC → 202, task enqueued; bad HMAC → 401, not enqueued`
- `Unit: malformed payload → 422`

#### 8.2 — Predictive work-order generation

**What**: A `critical` anomaly creates a `predictive` work order with a recommended parts list from the asset BOM.

**Design**:
- `workorder_service.generate_from_anomaly(event)`: create WO (type=predictive, priority from severity), add `work_order_part` planned rows from `asset_bom` where `is_critical`, raise a notification.
- Dedup: suppress duplicate WO for the same asset+metric within a configurable window.

**Testing**:
- `Integration: critical event → predictive WO with planned parts from BOM`
- `Integration: second event within window → no duplicate WO`
- `Integration: info-severity event → no WO`

### Phase 8 done when: signed sensor events generate predictive work orders pre-populated with the right critical spares, with sane deduplication.

---

## Phase 9: Cross-Site Balancing & Supplier-Risk Intelligence (v1.1)

### Purpose
Deliver two underserved differentiators: automated surplus-to-shortfall transfer recommendations across sites, and ML supplier-lead-time risk scoring that flags at-risk parts before stockouts.

### Tasks

#### 9.1 — Cross-site balancing recommender

**What**: Recommend stock transfers moving surplus at one storeroom to cover forecast shortfall at another.

**Design**:
- For each part: compute per-storeroom projected shortfall (forecast demand over lead time vs on_hand) and surplus (on_hand > max_level). Match surpluses to shortfalls minimising transfer count/distance.
- `GET /recommendations/transfers` → list of `{part, from, to, qty, rationale}`; approve creates a `stock_transfer` (Phase 3 transfer flow).

**Testing**:
- `Unit: surplus at A, shortfall at B → transfer A→B for the gap`
- `Integration: approve recommendation → stock_transfer + lines created`
- `Unit: no surplus anywhere → empty recommendations`

#### 9.2 — Supplier lead-time risk scoring + alerts

**What**: ML model scoring supplier-part stockout risk from `supplier_delivery_performance` trends.

**Design**:
- Features: rolling on-time rate, lead-time mean/variance trend, in-full rate, open-PO exposure. Gradient-boosted classifier → risk score 0–100; nightly job writes scores and raises `supplier_risk` alerts above threshold.
- `GET /suppliers/{id}/risk`, `GET /alerts?type=supplier_risk`.

**Testing**:
- `Unit: degrading on-time trend → rising risk score`
- `Integration: nightly job → risk scores persisted, alerts above threshold`
- `Fixture: committed delivery-history sample yields deterministic score`

### Phase 9 done when: the system recommends inter-site transfers and proactively flags supplier-driven stockout risk with explainable scores.

---

## Phase 10: MCP Server, NL Search & Knowledge Capture (v1.1 → backlog)

### Purpose
Make the platform agent-accessible — an industry first per standards.md — and add the natural-language capabilities (part search, technician knowledge capture) that incumbents lack.

### Tasks

#### 10.1 — MCP server

**What**: An MCP server exposing inventory operations as LLM tools.

**Design**:
- Tools: `search_parts`, `get_stock_level`, `create_purchase_order`, `generate_work_order`, `get_part_forecast`.
- Auth: bearer token mapped to an org+role; tools enforce RBAC and RLS exactly as the REST API does. Audit every tool call.

**Testing**:
- `Integration: MCP search_parts tool → same results as REST search`
- `Integration: create_purchase_order via MCP respects role (viewer token → denied)`
- `Integration: every tool call writes an audit_log row`

#### 10.2 — Natural-language part search

**What**: Plain-English queries over the messy catalogue (e.g. "stainless half-inch bolt for pump 7").

**Design**:
- Hybrid: `pgvector` semantic similarity + trigram lexical, fused (reciprocal-rank fusion); optional LLM re-rank of top-N via `LLMClient` with prompt caching.
- `GET /parts/search?q=<nl>` returns ranked parts with match rationale.

**Testing**:
- `Integration (mocked embeddings/LLM): NL query → relevant parts ranked above noise`
- `Unit: fusion ranking stable and deterministic given fixed scores`

#### 10.3 — Technician knowledge capture (backlog)

**What**: Convert free-text/voice repair notes into structured records (failure mode/cause, parts used).

**Design**:
- `POST /work-orders/{id}/notes` accepts text; `LLMClient` extracts structured fields validated against ISO 14224 `failure_mode`/`failure_cause` lookups; low-confidence extractions queued for human review, never auto-applied.

**Testing**:
- `Integration (mocked LLM): note → structured failure_mode/cause + parts, validated against lookups`
- `Unit: low-confidence extraction routed to review queue, not applied`

### Phase 10 done when: an LLM agent can operate the platform via MCP under full RBAC/audit, users can search the catalogue in natural language, and technician notes can be structured with human-in-the-loop review.

---

## Phase Summary & Dependencies

```
Phase 1: Foundation (auth, DB, RLS, tenancy) ─── required by everything
    │
Phase 2: Catalogue (sites, parts, taxonomy) ──── requires P1
    │
Phase 3: Inventory core (balances, txns)    ──── requires P2
    ├── Phase 4: Procurement & suppliers     ── requires P3
    │
Phase 5: AI demand forecasting              ──── requires P3 (P4 enriches lead-time data)
    │
Phase 6: Web app & barcode PWA              ──── requires P3,P4,P5 (consumes their APIs)
    │
    ├── Phase 7: GS1 + catalogue dedup (v1.1)        ── requires P2,P3,P6
    ├── Phase 8: IoT + predictive work orders (v1.1) ── requires P3 (WO), parallel with P7
    ├── Phase 9: Cross-site balancing + supplier risk ─ requires P4,P5, parallel with P7,P8
    └── Phase 10: MCP + NL search + knowledge capture ─ requires P3,P4,P5; NL search needs pgvector (P7)
```

**Parallelism opportunities**
- Phase 4 (Procurement) and Phase 5 (Forecasting) can be built concurrently once Phase 3 lands — forecasting only needs the transaction history, not POs.
- Phases 7, 8, 9 are independent v1.1 tracks after Phase 6 and can be developed by separate contributors in parallel.
- Frontend work in Phase 6 can begin against the OpenAPI spec as soon as each backend resource (Phases 3–5) is contract-complete, even before backend polish.

**MVP cut line:** Phases 1–6 constitute the MVP (matching features.md "Must-have"). Phases 7–9 are v1.1 ("Should-have"). Phase 10 spans v1.1 (MCP, NL search) into backlog (knowledge capture).

---

## Definition of Done (per phase)

A phase is complete only when **all** of the following hold:

1. All tasks in the phase are implemented.
2. All unit and integration tests for the phase pass; coverage on new service/AI code ≥ 85%.
3. `ruff check` and `ruff format --check` pass; `mypy --strict` passes (backend); ESLint/Prettier pass (frontend).
4. `docker compose build` succeeds and `docker compose up` brings the stack to a healthy state (`/readyz` 200).
5. The phase's feature works end-to-end against a real Postgres (Testcontainers in CI) — not only against mocks.
6. New configuration options are added to `.env.example` and documented in the README.
7. New/changed API endpoints appear in the exported OpenAPI 3.1 spec (CI compares `openapi/` snapshot; drift fails the build).
8. Alembic migration(s) created, and `upgrade head` → `downgrade base` round-trips cleanly.
9. Row-level security verified for any new tenant-scoped table (cross-org isolation test present).
10. Any AI output is explainable: forecasts/scores persist `method`/`model_version` and confidence, and no AI recommendation mutates operational data without explicit user approval.
```
