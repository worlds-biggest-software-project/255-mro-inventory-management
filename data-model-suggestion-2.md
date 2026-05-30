# Data Model Suggestion 2: Event-Sourced / Audit-First (CQRS)

> Project: MRO Inventory Management · Created: 2026-05-22

## Philosophy

This model treats every state change as an immutable event appended to a central event store. The current state of any entity — inventory levels, work order status, part criticality — is derived by replaying events rather than stored as a mutable row. Read-optimised materialised views (projections) serve queries, while the event store serves as the single source of truth. This is the Command Query Responsibility Segregation (CQRS) pattern applied to MRO inventory.

The approach is grounded in real-world precedent: financial ledger systems have operated on append-only principles for centuries. In the MRO domain, IBM Maximo and SAP PM both maintain internal transaction logs, but they treat these as secondary artefacts — the "real" data is the mutable record. This model inverts that relationship: the event log *is* the data, and the readable tables are derived projections that can be rebuilt at any time.

For MRO inventory management, event sourcing is particularly compelling because: (a) regulatory environments require complete audit trails of every parts movement, (b) AI demand forecasting benefits from rich temporal data about *when* and *why* inventory changed, not just current levels, (c) temporal queries ("what was the stock level of part X at site Y on March 15th?") are trivially answered by replaying events to that point, and (d) the system can retroactively correct projections when events are found to be erroneous without destroying history.

**Best for:** Organisations in regulated industries (aviation, energy, oil & gas) requiring complete audit trails, temporal queries, and rich event data to train AI demand forecasting models.

**Trade-offs:**
- Pro: Complete, immutable audit trail — every inventory movement, every status change is permanently recorded
- Pro: Temporal queries are first-class ("what was true at time T?")
- Pro: AI training data is inherently rich — events carry context (who, why, from what trigger)
- Pro: Projections can be rebuilt or new ones added without migrating the source data
- Pro: Natural fit for IoT sensor event ingestion — sensor readings are already events
- Con: Higher storage requirements — events are never deleted, only compacted via snapshots
- Con: Eventual consistency between event store and projections requires careful handling
- Con: More complex application code — developers must think in events rather than CRUD
- Con: Simple "update a field" operations become multi-step event-publish-project cycles
- Con: Debugging requires understanding event replay, not just reading the current row

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| ISO 55001:2024 | Asset lifecycle events provide native traceability; maintenance activity events linked to parts consumption events |
| ISO 14224:2016 | Failure events carry ISO 14224 failure mode and cause codes as event metadata |
| UNSPSC / eCl@ss | Classification stored as reference data; parts events reference taxonomy codes |
| GS1/GTIN | Barcode scan events capture GTIN, lot (AI 10), and serial (AI 21) as event payload fields |
| SAE JA1011 (RCM) | Criticality re-assessment events trigger reorder parameter recalculation projections |
| OSHA 29 CFR 1910.147 | LOTO procedure events create a compliance-ready timeline per asset |
| RFC 9110 (HTTP) | CQRS command endpoints (POST) separated from query endpoints (GET) |
| OCSF | Event schema borrows from Open Cybersecurity Schema Framework for structured event metadata |

---

## Event Store — Core Infrastructure

```sql
-- ============================================================
-- EVENT STORE — THE SINGLE SOURCE OF TRUTH
-- ============================================================

-- Central event store table. ALL state changes across the system are
-- recorded here as immutable, append-only events.
CREATE TABLE event_store (
    event_id        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    stream_id       UUID NOT NULL,                      -- aggregate root ID
    stream_type     VARCHAR(50) NOT NULL,               -- 'inventory', 'work_order', 'asset', 'purchase_order', etc.
    event_type      VARCHAR(100) NOT NULL,              -- e.g. 'inventory.issued', 'work_order.completed'
    event_version   INT NOT NULL,                       -- sequence within stream for ordering
    payload         JSONB NOT NULL,                     -- event-specific data
    metadata        JSONB NOT NULL DEFAULT '{}',        -- who, why, correlation IDs
    correlation_id  UUID,                               -- links related events across streams
    causation_id    UUID,                               -- the event that caused this event
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    created_by      UUID NOT NULL,                      -- user who triggered the event
    UNIQUE(stream_id, event_version)
);

-- Partitioned by month for performance at scale
-- In production, use: PARTITION BY RANGE (created_at)

CREATE INDEX idx_event_stream ON event_store(stream_id, event_version);
CREATE INDEX idx_event_type ON event_store(event_type);
CREATE INDEX idx_event_created ON event_store(created_at);
CREATE INDEX idx_event_correlation ON event_store(correlation_id) WHERE correlation_id IS NOT NULL;
CREATE INDEX idx_event_stream_type ON event_store(stream_type, created_at);

-- Snapshot store for performance — avoids replaying all events
CREATE TABLE event_snapshot (
    stream_id       UUID NOT NULL,
    stream_type     VARCHAR(50) NOT NULL,
    snapshot_version INT NOT NULL,                      -- event_version at snapshot time
    state           JSONB NOT NULL,                     -- serialised aggregate state
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (stream_id, snapshot_version)
);
```

### Event Payload Examples

```sql
-- Example: inventory.received event payload
-- {
--   "part_id": "550e8400-e29b-41d4-a716-446655440001",
--   "part_number": "BRG-SKF-6205-2Z",
--   "storeroom_id": "550e8400-e29b-41d4-a716-446655440010",
--   "quantity": 25,
--   "unit_cost": 12.50,
--   "lot_number": "LOT-2026-03-4421",
--   "gtin": "00614141000012",
--   "purchase_order_id": "550e8400-e29b-41d4-a716-446655440020",
--   "po_line_number": 3,
--   "supplier_id": "550e8400-e29b-41d4-a716-446655440030",
--   "bin_location": "A3-S2-B7"
-- }

-- Example: inventory.issued event payload
-- {
--   "part_id": "550e8400-e29b-41d4-a716-446655440001",
--   "storeroom_id": "550e8400-e29b-41d4-a716-446655440010",
--   "quantity": 2,
--   "work_order_id": "550e8400-e29b-41d4-a716-446655440040",
--   "asset_id": "550e8400-e29b-41d4-a716-446655440050",
--   "issued_to": "550e8400-e29b-41d4-a716-446655440060",
--   "serial_numbers": ["SN-2026-001", "SN-2026-002"]
-- }

-- Example: asset.failure_reported event payload
-- {
--   "asset_id": "550e8400-e29b-41d4-a716-446655440050",
--   "failure_mode": "BRG_WEAR",
--   "failure_cause": "LUBRICATION_FAILURE",
--   "iso_14224_failure_mode": "1.3.2",
--   "iso_14224_failure_cause": "2.1.4",
--   "severity": "high",
--   "downtime_started_at": "2026-03-15T14:30:00Z",
--   "detected_by": "vibration_sensor",
--   "sensor_reading_id": "550e8400-e29b-41d4-a716-446655440070"
-- }

-- Example: forecast.generated event payload
-- {
--   "part_id": "550e8400-e29b-41d4-a716-446655440001",
--   "storeroom_id": "550e8400-e29b-41d4-a716-446655440010",
--   "method": "croston_sba",
--   "period_start": "2026-04-01",
--   "period_end": "2026-06-30",
--   "point_estimate": 8.5,
--   "confidence_lower": 3.0,
--   "confidence_upper": 15.0,
--   "confidence_level": 0.90,
--   "recommended_reorder_point": 12,
--   "model_version": "v2.3.1"
-- }
```

## Event Type Registry

```sql
-- ============================================================
-- EVENT TYPE REGISTRY — documents all valid event types
-- ============================================================

CREATE TABLE event_type_registry (
    event_type      VARCHAR(100) PRIMARY KEY,
    stream_type     VARCHAR(50) NOT NULL,
    description     TEXT NOT NULL,
    payload_schema  JSONB NOT NULL,                     -- JSON Schema for validation
    version         INT NOT NULL DEFAULT 1,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Seed with core event types:
INSERT INTO event_type_registry (event_type, stream_type, description, payload_schema) VALUES
    ('inventory.received',       'inventory',       'Parts received into storeroom from PO or return', '{}'),
    ('inventory.issued',         'inventory',       'Parts issued from storeroom to work order', '{}'),
    ('inventory.returned',       'inventory',       'Parts returned to storeroom from work order', '{}'),
    ('inventory.adjusted',       'inventory',       'Manual inventory adjustment (cycle count, correction)', '{}'),
    ('inventory.transferred_out','inventory',       'Parts shipped to another storeroom', '{}'),
    ('inventory.transferred_in', 'inventory',       'Parts received from another storeroom', '{}'),
    ('inventory.reserved',       'inventory',       'Parts reserved for a planned work order', '{}'),
    ('inventory.unreserved',     'inventory',       'Reservation released', '{}'),
    ('inventory.scrapped',       'inventory',       'Parts scrapped / condemned', '{}'),
    ('inventory.reorder_triggered', 'inventory',    'Stock fell below reorder point, PO recommended', '{}'),

    ('work_order.created',       'work_order',      'New work order created', '{}'),
    ('work_order.planned',       'work_order',      'Work order scheduled with parts and labour', '{}'),
    ('work_order.started',       'work_order',      'Work commenced on work order', '{}'),
    ('work_order.parts_issued',  'work_order',      'Parts issued against this work order', '{}'),
    ('work_order.completed',     'work_order',      'Work order marked complete', '{}'),
    ('work_order.cancelled',     'work_order',      'Work order cancelled', '{}'),

    ('asset.registered',         'asset',           'New asset registered in system', '{}'),
    ('asset.failure_reported',   'asset',           'Equipment failure reported with ISO 14224 codes', '{}'),
    ('asset.repaired',           'asset',           'Asset repair completed', '{}'),
    ('asset.decommissioned',     'asset',           'Asset removed from service', '{}'),
    ('asset.criticality_changed','asset',           'Asset criticality rating changed', '{}'),

    ('purchase_order.created',   'purchase_order',  'New PO drafted', '{}'),
    ('purchase_order.submitted', 'purchase_order',  'PO submitted for approval', '{}'),
    ('purchase_order.approved',  'purchase_order',  'PO approved', '{}'),
    ('purchase_order.line_received', 'purchase_order', 'PO line partially or fully received', '{}'),
    ('purchase_order.closed',    'purchase_order',  'PO fully received and closed', '{}'),

    ('forecast.generated',       'forecast',        'AI demand forecast produced', '{}'),
    ('forecast.accepted',        'forecast',        'Forecast recommendation accepted by planner', '{}'),
    ('forecast.overridden',      'forecast',        'Planner manually overrode forecast values', '{}'),

    ('sensor.reading_received',  'sensor',          'IoT sensor reading ingested', '{}'),
    ('sensor.anomaly_detected',  'sensor',          'Anomaly detected in sensor data', '{}'),
    ('sensor.threshold_breached','sensor',          'Sensor reading exceeded configured threshold', '{}');
```

## Read-Side Projections (Materialised Views)

```sql
-- ============================================================
-- PROJECTIONS — MATERIALISED READ MODELS
-- ============================================================

-- These tables are DERIVED from the event store. They can be dropped
-- and rebuilt by replaying events. They serve read queries only.

-- Reference data (not event-sourced — static lookups)
CREATE TABLE ref_site (
    id              UUID PRIMARY KEY,
    code            VARCHAR(50) NOT NULL UNIQUE,
    name            VARCHAR(200) NOT NULL,
    country_code    CHAR(2) NOT NULL,
    timezone        VARCHAR(50) NOT NULL DEFAULT 'UTC',
    is_active       BOOLEAN NOT NULL DEFAULT true
);

CREATE TABLE ref_storeroom (
    id              UUID PRIMARY KEY,
    site_id         UUID NOT NULL REFERENCES ref_site(id),
    code            VARCHAR(50) NOT NULL,
    name            VARCHAR(200) NOT NULL,
    type            VARCHAR(30) NOT NULL,
    UNIQUE(site_id, code)
);

CREATE TABLE ref_part (
    id              UUID PRIMARY KEY,
    part_number     VARCHAR(100) NOT NULL UNIQUE,
    name            VARCHAR(300) NOT NULL,
    description     TEXT,
    unspsc_code     CHAR(8),
    eclass_code     VARCHAR(20),
    gtin            VARCHAR(14),
    manufacturer_name VARCHAR(200),
    manufacturer_part_number VARCHAR(100),
    unit_of_measure VARCHAR(10) NOT NULL,
    is_rotable      BOOLEAN NOT NULL DEFAULT false,
    is_serialized   BOOLEAN NOT NULL DEFAULT false,
    criticality     VARCHAR(10),
    unit_cost       NUMERIC(14,4)
);

-- ============================================================
-- PROJECTION: Current inventory levels
-- Rebuilt by: replaying all inventory.* events
-- ============================================================

CREATE TABLE proj_inventory_balance (
    part_id             UUID NOT NULL,
    storeroom_id        UUID NOT NULL,
    quantity_on_hand    NUMERIC(12,2) NOT NULL DEFAULT 0,
    quantity_reserved   NUMERIC(12,2) NOT NULL DEFAULT 0,
    quantity_on_order   NUMERIC(12,2) NOT NULL DEFAULT 0,
    reorder_point       NUMERIC(12,2),
    reorder_quantity    NUMERIC(12,2),
    safety_stock        NUMERIC(12,2),
    last_receipt_event_id UUID,
    last_issue_event_id   UUID,
    last_updated_at     TIMESTAMPTZ NOT NULL,
    PRIMARY KEY (part_id, storeroom_id)
);

CREATE INDEX idx_proj_invbal_low ON proj_inventory_balance(quantity_on_hand, reorder_point)
    WHERE quantity_on_hand <= reorder_point;

-- ============================================================
-- PROJECTION: Work order current state
-- Rebuilt by: replaying all work_order.* events
-- ============================================================

CREATE TABLE proj_work_order (
    id              UUID PRIMARY KEY,
    work_order_number VARCHAR(50) NOT NULL UNIQUE,
    asset_id        UUID NOT NULL,
    site_id         UUID NOT NULL,
    type            VARCHAR(30) NOT NULL,
    priority        VARCHAR(10) NOT NULL,
    status          VARCHAR(20) NOT NULL,
    description     TEXT,
    failure_mode    VARCHAR(100),
    failure_cause   VARCHAR(100),
    assigned_to     UUID,
    planned_start   TIMESTAMPTZ,
    actual_start    TIMESTAMPTZ,
    actual_end      TIMESTAMPTZ,
    parts_cost      NUMERIC(14,2) DEFAULT 0,
    event_count     INT NOT NULL DEFAULT 0,
    last_event_id   UUID NOT NULL,
    last_updated_at TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_proj_wo_status ON proj_work_order(status);
CREATE INDEX idx_proj_wo_asset ON proj_work_order(asset_id);

-- ============================================================
-- PROJECTION: Asset current state with latest failure info
-- ============================================================

CREATE TABLE proj_asset (
    id                  UUID PRIMARY KEY,
    asset_number        VARCHAR(50) NOT NULL UNIQUE,
    name                VARCHAR(200) NOT NULL,
    equipment_class     VARCHAR(200),
    site_id             UUID NOT NULL,
    functional_location VARCHAR(200),
    status              VARCHAR(30) NOT NULL,
    criticality_rating  VARCHAR(10),
    last_failure_date   TIMESTAMPTZ,
    last_failure_mode   VARCHAR(100),
    total_failures      INT DEFAULT 0,
    total_downtime_minutes INT DEFAULT 0,
    last_event_id       UUID NOT NULL,
    last_updated_at     TIMESTAMPTZ NOT NULL
);

-- ============================================================
-- PROJECTION: Purchase order current state
-- ============================================================

CREATE TABLE proj_purchase_order (
    id              UUID PRIMARY KEY,
    po_number       VARCHAR(50) NOT NULL UNIQUE,
    supplier_id     UUID NOT NULL,
    supplier_name   VARCHAR(300),
    site_id         UUID NOT NULL,
    status          VARCHAR(20) NOT NULL,
    order_date      DATE NOT NULL,
    total_amount    NUMERIC(14,2),
    lines_ordered   INT DEFAULT 0,
    lines_received  INT DEFAULT 0,
    last_event_id   UUID NOT NULL,
    last_updated_at TIMESTAMPTZ NOT NULL
);

-- ============================================================
-- PROJECTION: Serialized part lifecycle (rotable spares)
-- ============================================================

CREATE TABLE proj_serial_lifecycle (
    part_id         UUID NOT NULL,
    serial_number   VARCHAR(100) NOT NULL,
    current_status  VARCHAR(20) NOT NULL,               -- in_stock, installed, in_repair, scrapped
    current_location_type VARCHAR(20),                  -- storeroom, asset, repair_shop
    current_location_id   UUID,
    condition_code  VARCHAR(20),
    total_install_count INT DEFAULT 0,
    total_repair_count  INT DEFAULT 0,
    last_event_id   UUID NOT NULL,
    last_updated_at TIMESTAMPTZ NOT NULL,
    PRIMARY KEY (part_id, serial_number)
);

-- ============================================================
-- PROJECTION: Supplier performance dashboard
-- ============================================================

CREATE TABLE proj_supplier_performance (
    supplier_id     UUID NOT NULL,
    part_id         UUID NOT NULL,
    total_orders    INT DEFAULT 0,
    total_on_time   INT DEFAULT 0,
    total_in_full   INT DEFAULT 0,
    avg_lead_time_days NUMERIC(6,1),
    last_delivery_date DATE,
    otif_percentage NUMERIC(5,2),                       -- On Time In Full %
    last_updated_at TIMESTAMPTZ NOT NULL,
    PRIMARY KEY (supplier_id, part_id)
);

-- ============================================================
-- PROJECTION: AI forecast dashboard (latest per part/storeroom)
-- ============================================================

CREATE TABLE proj_latest_forecast (
    part_id             UUID NOT NULL,
    storeroom_id        UUID NOT NULL,
    forecast_method     VARCHAR(30),
    point_estimate      NUMERIC(12,2),
    confidence_lower    NUMERIC(12,2),
    confidence_upper    NUMERIC(12,2),
    confidence_level    NUMERIC(3,2),
    recommended_reorder_point NUMERIC(12,2),
    recommended_reorder_qty   NUMERIC(12,2),
    model_version       VARCHAR(50),
    forecast_event_id   UUID NOT NULL,
    forecast_date       DATE NOT NULL,
    status              VARCHAR(20) DEFAULT 'pending',  -- pending, accepted, overridden
    PRIMARY KEY (part_id, storeroom_id)
);
```

## IoT Sensor Event Ingestion

```sql
-- ============================================================
-- SENSOR DATA — Uses TimescaleDB hypertable for time-series
-- ============================================================

-- If TimescaleDB extension is available:
-- CREATE EXTENSION IF NOT EXISTS timescaledb;

CREATE TABLE sensor_reading (
    time            TIMESTAMPTZ NOT NULL,
    sensor_id       UUID NOT NULL,
    asset_id        UUID NOT NULL,
    reading_type    VARCHAR(30) NOT NULL,               -- temperature, vibration, pressure, current
    value           DOUBLE PRECISION NOT NULL,
    unit            VARCHAR(20) NOT NULL,
    quality         VARCHAR(10) DEFAULT 'good'          -- good, suspect, bad
);

-- Convert to TimescaleDB hypertable for automatic partitioning:
-- SELECT create_hypertable('sensor_reading', 'time');

CREATE INDEX idx_sensor_asset ON sensor_reading(asset_id, time DESC);
CREATE INDEX idx_sensor_id ON sensor_reading(sensor_id, time DESC);

-- Sensor anomaly events flow into the main event_store with
-- stream_type='sensor' and event_type='sensor.anomaly_detected',
-- carrying the sensor_reading timestamps and values in the payload.
```

## Temporal Query Examples

```sql
-- ============================================================
-- EXAMPLE QUERIES — demonstrating event-sourced capabilities
-- ============================================================

-- Q1: What was the inventory level of part X at storeroom Y on March 15th?
-- (Replay inventory events for that stream up to the target date)
SELECT
    SUM(CASE
        WHEN event_type IN ('inventory.received', 'inventory.transferred_in', 'inventory.returned')
            THEN (payload->>'quantity')::NUMERIC
        WHEN event_type IN ('inventory.issued', 'inventory.transferred_out', 'inventory.scrapped')
            THEN -(payload->>'quantity')::NUMERIC
        WHEN event_type = 'inventory.adjusted'
            THEN (payload->>'adjustment_quantity')::NUMERIC
        ELSE 0
    END) AS quantity_at_date
FROM event_store
WHERE stream_type = 'inventory'
  AND payload->>'part_id' = '550e8400-e29b-41d4-a716-446655440001'
  AND payload->>'storeroom_id' = '550e8400-e29b-41d4-a716-446655440010'
  AND created_at <= '2026-03-15T23:59:59Z';

-- Q2: Complete timeline of a serialized rotable spare
SELECT
    event_type,
    created_at,
    payload->>'status' AS status,
    payload->>'location' AS location,
    metadata->>'performed_by_name' AS who,
    payload
FROM event_store
WHERE stream_type = 'inventory'
  AND payload->>'serial_number' = 'SN-2026-001'
ORDER BY created_at;

-- Q3: All events correlated to a single work order
-- (Uses correlation_id to link work_order, inventory, and purchase_order events)
SELECT
    event_type,
    stream_type,
    created_at,
    payload
FROM event_store
WHERE correlation_id = '550e8400-e29b-41d4-a716-446655440040'
ORDER BY created_at;

-- Q4: Demand pattern extraction for AI training
-- (Extract all issue events for a part with temporal context)
SELECT
    date_trunc('week', created_at) AS week,
    COUNT(*) AS issue_events,
    SUM((payload->>'quantity')::NUMERIC) AS total_issued,
    array_agg(DISTINCT payload->>'work_order_type') AS work_order_types
FROM event_store
WHERE event_type = 'inventory.issued'
  AND payload->>'part_id' = '550e8400-e29b-41d4-a716-446655440001'
  AND created_at >= now() - INTERVAL '2 years'
GROUP BY date_trunc('week', created_at)
ORDER BY week;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Event Store Infrastructure | 3 | event_store, event_snapshot, event_type_registry |
| Reference Data | 3 | ref_site, ref_storeroom, ref_part |
| Projections — Operational | 4 | proj_inventory_balance, proj_work_order, proj_asset, proj_purchase_order |
| Projections — Analytics | 3 | proj_serial_lifecycle, proj_supplier_performance, proj_latest_forecast |
| Sensor Data (TimescaleDB) | 1 | sensor_reading (hypertable) |
| **Total** | **~14** | Plus any additional projections added over time |

---

## Key Design Decisions

1. **Single event store table** — all domain events (inventory, work orders, assets, purchase orders, forecasts, sensors) flow into one `event_store` table, partitioned by `created_at`. This enables cross-domain correlation queries and simplifies event replay infrastructure.

2. **Stream ID + stream type + version** — each aggregate (e.g., a specific inventory position for a part at a storeroom) has a unique stream_id. Events within a stream are ordered by `event_version`, ensuring deterministic replay. The UNIQUE constraint on `(stream_id, event_version)` prevents concurrent writes from corrupting stream order.

3. **Correlation and causation IDs** — every event can reference the event that caused it (`causation_id`) and the broader business operation it belongs to (`correlation_id`). This enables queries like "show me everything that happened because of this sensor anomaly" — tracing from sensor alert through work order creation through parts issuance.

4. **Projections are disposable** — all `proj_*` tables can be dropped and rebuilt from events. This means new analytics requirements can be met by adding a new projection and replaying history, without migrating the source data.

5. **Snapshots for performance** — for streams with thousands of events (e.g., a high-volume part at a busy storeroom), `event_snapshot` stores periodic state snapshots so replay only needs to process events after the last snapshot.

6. **Event type registry** — the `event_type_registry` table documents all valid event types with JSON Schema for payload validation, enabling API consumers and projection builders to discover and validate event structures.

7. **IoT sensor data in TimescaleDB** — raw sensor readings use a separate time-series optimised table (TimescaleDB hypertable) rather than flowing through the event store, because sensor data volume (millions of readings/day) would overwhelm the event store. Anomaly detections and threshold breaches are promoted to events in the main store.

8. **Rich event metadata** — the `metadata` JSONB column on every event captures context: who triggered it, from which IP, via which API endpoint, which correlation/causation chain. This metadata is invaluable for audit, debugging, and AI feature engineering.

9. **Reference data is not event-sourced** — sites, storerooms, and part catalogue entries are managed as mutable reference data in `ref_*` tables. These change infrequently and do not benefit from event sourcing's overhead. This is a pragmatic hybrid that avoids dogmatic purity.

10. **AI forecasts as events** — forecast generation, acceptance, and override are all events. This means the system can audit why reorder parameters changed, who accepted or overrode a recommendation, and track forecast accuracy over time by comparing forecast events against subsequent issue events.
