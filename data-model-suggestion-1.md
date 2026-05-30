# Data Model Suggestion 1: Entity-Centric Normalized Relational

> Project: MRO Inventory Management · Created: 2026-05-22

## Philosophy

This model follows the classic normalized relational approach: every concept gets its own table, relationships are enforced through foreign keys, and reference data is separated into lookup tables aligned with industry standards (UNSPSC, GS1, ISO 14224). The schema prioritises data integrity above all else — every inventory movement, every purchase order line, every parts-to-asset linkage is governed by constraints that prevent inconsistent state.

Real-world systems that use this pattern include SAP Plant Maintenance (SAP PM), IBM Maximo, and Infor EAM. These enterprise platforms maintain hundreds of tables with deep foreign key chains, enabling complex cross-entity queries (e.g., "show me all parts consumed against assets in Plant X that had unplanned failures in Q3, grouped by UNSPSC category"). The trade-off is structural rigidity — adding a new attribute or entity type requires a schema migration.

This approach aligns naturally with ISO 55001 asset management system requirements, which demand traceability of spare parts to asset maintenance activities, criticality assessment, and lifecycle cost accountability. Each of these requirements maps directly to a table or relationship in this model.

**Best for:** Organisations with well-defined, stable MRO processes that need strong referential integrity, complex cross-entity reporting, and regulatory compliance.

**Trade-offs:**
- Pro: Maximum data integrity — the database enforces business rules
- Pro: Complex analytical queries across entities are straightforward SQL joins
- Pro: Standards alignment is explicit — UNSPSC codes, GS1 identifiers, ISO 14224 taxonomies live in dedicated tables
- Pro: Well understood by database administrators and ORM frameworks
- Con: Schema migrations required for new entity types or attributes
- Con: High table count increases join complexity for simple operations
- Con: Multi-industry or multi-jurisdiction variation is hard to model without proliferating columns
- Con: Write-heavy operations (high-frequency sensor data) may require partitioning strategies

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| ISO 55001:2024 | Asset management system structure maps to `asset`, `asset_type`, `asset_criticality` tables; traceability via `work_order_part` junction |
| ISO 14224:2016 | Equipment taxonomy hierarchy in `equipment_class`, `equipment_subclass`; failure modes in `failure_mode`, `failure_cause` lookup tables |
| UNSPSC | Four-level classification hierarchy stored in `unspsc_segment`, `unspsc_family`, `unspsc_class`, `unspsc_commodity` tables; parts mapped via `part.unspsc_commodity_id` |
| eCl@ss | Parallel classification in `eclass_segment` through `eclass_commodity`; cross-mapped via `taxonomy_crossmap` table |
| GS1/GTIN | `part.gtin` column stores GTIN-14; `part_lot` and `part_serial` tables for GS1 AI 10 (batch/lot) and AI 21 (serial) |
| SAE JA1011 (RCM) | Criticality classification in `part_criticality` table drives reorder priority logic |
| OSHA 29 CFR 1910.147 | LOTO procedures linked to assets via `asset_loto_procedure` table with parts lists |
| GS1 Application Identifiers | Barcode scan data parsed into structured fields in `barcode_scan_log` |
| OpenAPI 3.1 | API resource design mirrors table structure 1:1 for predictable REST endpoints |

---

## Core Tables — Asset & Equipment Management

```sql
-- ============================================================
-- ASSET & EQUIPMENT MANAGEMENT
-- ============================================================

CREATE TABLE equipment_class (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    code            VARCHAR(20) NOT NULL UNIQUE,        -- ISO 14224 class code
    name            VARCHAR(200) NOT NULL,
    description     TEXT,
    parent_class_id UUID REFERENCES equipment_class(id),
    iso_14224_ref   VARCHAR(50),                        -- ISO 14224 reference number
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_equipment_class_parent ON equipment_class(parent_class_id);
CREATE INDEX idx_equipment_class_code ON equipment_class(code);

CREATE TABLE site (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    code            VARCHAR(50) NOT NULL UNIQUE,
    name            VARCHAR(200) NOT NULL,
    address_line1   VARCHAR(300),
    address_line2   VARCHAR(300),
    city            VARCHAR(100),
    state_province  VARCHAR(100),
    postal_code     VARCHAR(20),
    country_code    CHAR(2) NOT NULL,                   -- ISO 3166-1 alpha-2
    timezone        VARCHAR(50) NOT NULL DEFAULT 'UTC', -- IANA timezone
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE functional_location (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    site_id         UUID NOT NULL REFERENCES site(id),
    code            VARCHAR(100) NOT NULL,
    name            VARCHAR(200) NOT NULL,
    description     TEXT,
    parent_id       UUID REFERENCES functional_location(id),
    level           INT NOT NULL DEFAULT 0,             -- hierarchy depth
    path            TEXT,                               -- materialised path e.g. /plant/area/unit
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(site_id, code)
);

CREATE INDEX idx_floc_site ON functional_location(site_id);
CREATE INDEX idx_floc_parent ON functional_location(parent_id);
CREATE INDEX idx_floc_path ON functional_location USING gist (path gist_trgm_ops);

CREATE TABLE asset (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    asset_number        VARCHAR(50) NOT NULL UNIQUE,
    name                VARCHAR(200) NOT NULL,
    description         TEXT,
    equipment_class_id  UUID NOT NULL REFERENCES equipment_class(id),
    functional_location_id UUID REFERENCES functional_location(id),
    site_id             UUID NOT NULL REFERENCES site(id),
    manufacturer        VARCHAR(200),
    model_number        VARCHAR(100),
    serial_number       VARCHAR(100),
    install_date        DATE,
    warranty_expiry     DATE,
    status              VARCHAR(30) NOT NULL DEFAULT 'active'
                        CHECK (status IN ('active','inactive','decommissioned','in_repair')),
    criticality_rating  VARCHAR(10)
                        CHECK (criticality_rating IN ('A','B','C','D')),  -- SAE JA1011 aligned
    replacement_cost    NUMERIC(14,2),
    currency_code       CHAR(3) DEFAULT 'USD',          -- ISO 4217
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_asset_site ON asset(site_id);
CREATE INDEX idx_asset_floc ON asset(functional_location_id);
CREATE INDEX idx_asset_class ON asset(equipment_class_id);
CREATE INDEX idx_asset_status ON asset(status);
CREATE INDEX idx_asset_criticality ON asset(criticality_rating);
```

## Core Tables — Parts & Catalogue Management

```sql
-- ============================================================
-- TAXONOMY & CLASSIFICATION (UNSPSC / eCl@ss)
-- ============================================================

CREATE TABLE unspsc_segment (
    id      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    code    CHAR(2) NOT NULL UNIQUE,
    name    VARCHAR(200) NOT NULL
);

CREATE TABLE unspsc_family (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    segment_id  UUID NOT NULL REFERENCES unspsc_segment(id),
    code        CHAR(4) NOT NULL UNIQUE,
    name        VARCHAR(200) NOT NULL
);

CREATE TABLE unspsc_class (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    family_id   UUID NOT NULL REFERENCES unspsc_family(id),
    code        CHAR(6) NOT NULL UNIQUE,
    name        VARCHAR(200) NOT NULL
);

CREATE TABLE unspsc_commodity (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    class_id    UUID NOT NULL REFERENCES unspsc_class(id),
    code        CHAR(8) NOT NULL UNIQUE,
    name        VARCHAR(200) NOT NULL
);

CREATE TABLE eclass_class (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    code            VARCHAR(20) NOT NULL UNIQUE,        -- eCl@ss coded identifier
    name            VARCHAR(200) NOT NULL,
    version         VARCHAR(10) NOT NULL DEFAULT '14.0'
);

CREATE TABLE taxonomy_crossmap (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    unspsc_commodity_id UUID REFERENCES unspsc_commodity(id),
    eclass_class_id     UUID REFERENCES eclass_class(id),
    confidence          NUMERIC(3,2) CHECK (confidence BETWEEN 0 AND 1),
    source              VARCHAR(50),                    -- 'manual', 'ml_mapped', 'vendor'
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- ============================================================
-- PARTS CATALOGUE
-- ============================================================

CREATE TABLE unit_of_measure (
    id      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    code    VARCHAR(10) NOT NULL UNIQUE,                -- EA, KG, M, L, etc.
    name    VARCHAR(50) NOT NULL
);

CREATE TABLE part (
    id                      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    part_number             VARCHAR(100) NOT NULL UNIQUE,
    name                    VARCHAR(300) NOT NULL,
    description             TEXT,
    unspsc_commodity_id     UUID REFERENCES unspsc_commodity(id),
    eclass_class_id         UUID REFERENCES eclass_class(id),
    unit_of_measure_id      UUID NOT NULL REFERENCES unit_of_measure(id),
    gtin                    VARCHAR(14),                -- GS1 GTIN-14
    manufacturer_name       VARCHAR(200),
    manufacturer_part_number VARCHAR(100),
    is_rotable              BOOLEAN NOT NULL DEFAULT false,
    is_serialized           BOOLEAN NOT NULL DEFAULT false,
    weight_kg               NUMERIC(10,3),
    lead_time_days          INT,                        -- standard lead time
    unit_cost               NUMERIC(14,4),
    currency_code           CHAR(3) DEFAULT 'USD',
    status                  VARCHAR(20) NOT NULL DEFAULT 'active'
                            CHECK (status IN ('active','obsolete','pending_review','duplicate_suspect')),
    criticality             VARCHAR(10)
                            CHECK (criticality IN ('critical','essential','standard','non_critical')),
    created_at              TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at              TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_part_unspsc ON part(unspsc_commodity_id);
CREATE INDEX idx_part_gtin ON part(gtin) WHERE gtin IS NOT NULL;
CREATE INDEX idx_part_manufacturer ON part(manufacturer_name, manufacturer_part_number);
CREATE INDEX idx_part_status ON part(status);
CREATE INDEX idx_part_criticality ON part(criticality);
CREATE INDEX idx_part_name_trgm ON part USING gist (name gist_trgm_ops);

-- Part alternate / substitute relationships
CREATE TABLE part_alternate (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    part_id         UUID NOT NULL REFERENCES part(id),
    alternate_part_id UUID NOT NULL REFERENCES part(id),
    relationship    VARCHAR(20) NOT NULL
                    CHECK (relationship IN ('substitute','supersedes','equivalent')),
    notes           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    CHECK (part_id <> alternate_part_id)
);

CREATE INDEX idx_part_alt_part ON part_alternate(part_id);
CREATE INDEX idx_part_alt_alternate ON part_alternate(alternate_part_id);

-- Parts linked to assets (Bill of Materials)
CREATE TABLE asset_bom (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    asset_id        UUID NOT NULL REFERENCES asset(id),
    part_id         UUID NOT NULL REFERENCES part(id),
    quantity        NUMERIC(10,2) NOT NULL DEFAULT 1,
    position_code   VARCHAR(50),                        -- installation position
    is_critical     BOOLEAN NOT NULL DEFAULT false,
    notes           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(asset_id, part_id, position_code)
);

CREATE INDEX idx_asset_bom_asset ON asset_bom(asset_id);
CREATE INDEX idx_asset_bom_part ON asset_bom(part_id);
```

## Core Tables — Inventory & Storeroom Management

```sql
-- ============================================================
-- STOREROOM & INVENTORY
-- ============================================================

CREATE TABLE storeroom (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    site_id         UUID NOT NULL REFERENCES site(id),
    code            VARCHAR(50) NOT NULL,
    name            VARCHAR(200) NOT NULL,
    type            VARCHAR(30) NOT NULL DEFAULT 'central'
                    CHECK (type IN ('central','satellite','vehicle','consignment')),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(site_id, code)
);

CREATE TABLE bin_location (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    storeroom_id    UUID NOT NULL REFERENCES storeroom(id),
    code            VARCHAR(50) NOT NULL,               -- aisle-shelf-bin code
    description     VARCHAR(200),
    UNIQUE(storeroom_id, code)
);

CREATE TABLE inventory_balance (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    part_id         UUID NOT NULL REFERENCES part(id),
    storeroom_id    UUID NOT NULL REFERENCES storeroom(id),
    bin_location_id UUID REFERENCES bin_location(id),
    quantity_on_hand    NUMERIC(12,2) NOT NULL DEFAULT 0
                        CHECK (quantity_on_hand >= 0),
    quantity_reserved   NUMERIC(12,2) NOT NULL DEFAULT 0
                        CHECK (quantity_reserved >= 0),
    quantity_on_order   NUMERIC(12,2) NOT NULL DEFAULT 0
                        CHECK (quantity_on_order >= 0),
    reorder_point       NUMERIC(12,2),                  -- min threshold
    reorder_quantity    NUMERIC(12,2),                   -- how much to reorder
    maximum_level       NUMERIC(12,2),                   -- max threshold
    safety_stock        NUMERIC(12,2),
    last_count_date     DATE,
    last_issue_date     DATE,
    last_receipt_date   DATE,
    average_monthly_usage NUMERIC(12,2),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(part_id, storeroom_id)
);

CREATE INDEX idx_invbal_part ON inventory_balance(part_id);
CREATE INDEX idx_invbal_storeroom ON inventory_balance(storeroom_id);
CREATE INDEX idx_invbal_low_stock ON inventory_balance(quantity_on_hand, reorder_point)
    WHERE quantity_on_hand <= reorder_point;

-- Lot and serial tracking (GS1 AI 10 / AI 21)
CREATE TABLE part_lot (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    part_id         UUID NOT NULL REFERENCES part(id),
    lot_number      VARCHAR(50) NOT NULL,               -- GS1 AI 10
    expiration_date DATE,
    quantity        NUMERIC(12,2) NOT NULL DEFAULT 0,
    storeroom_id    UUID NOT NULL REFERENCES storeroom(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(part_id, lot_number, storeroom_id)
);

CREATE TABLE part_serial (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    part_id         UUID NOT NULL REFERENCES part(id),
    serial_number   VARCHAR(100) NOT NULL,              -- GS1 AI 21
    status          VARCHAR(20) NOT NULL DEFAULT 'in_stock'
                    CHECK (status IN ('in_stock','installed','in_repair','scrapped','in_transit')),
    storeroom_id    UUID REFERENCES storeroom(id),
    asset_id        UUID REFERENCES asset(id),          -- if installed
    condition_code  VARCHAR(20)
                    CHECK (condition_code IN ('new','serviceable','unserviceable','repairable','condemned')),
    total_flight_hours NUMERIC(10,1),                   -- aviation-specific, nullable
    total_cycles    INT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(part_id, serial_number)
);

CREATE INDEX idx_partserial_status ON part_serial(status);
CREATE INDEX idx_partserial_asset ON part_serial(asset_id) WHERE asset_id IS NOT NULL;

-- Inventory transactions (movement log)
CREATE TABLE inventory_transaction (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    part_id         UUID NOT NULL REFERENCES part(id),
    storeroom_id    UUID NOT NULL REFERENCES storeroom(id),
    transaction_type VARCHAR(30) NOT NULL
                    CHECK (transaction_type IN (
                        'receipt','issue','return','adjustment','transfer_out',
                        'transfer_in','cycle_count','scrap','reservation','unreservation'
                    )),
    quantity        NUMERIC(12,2) NOT NULL,
    unit_cost       NUMERIC(14,4),
    work_order_id   UUID,                               -- FK added after work_order table
    purchase_order_line_id UUID,                         -- FK added after po_line table
    lot_number      VARCHAR(50),
    serial_number   VARCHAR(100),
    reference_number VARCHAR(100),                       -- cross-reference
    notes           TEXT,
    performed_by    UUID NOT NULL,                       -- FK to user
    performed_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_invtxn_part ON inventory_transaction(part_id);
CREATE INDEX idx_invtxn_storeroom ON inventory_transaction(storeroom_id);
CREATE INDEX idx_invtxn_type ON inventory_transaction(transaction_type);
CREATE INDEX idx_invtxn_date ON inventory_transaction(performed_at);
CREATE INDEX idx_invtxn_wo ON inventory_transaction(work_order_id) WHERE work_order_id IS NOT NULL;

-- Inter-site stock transfers
CREATE TABLE stock_transfer (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    transfer_number     VARCHAR(50) NOT NULL UNIQUE,
    source_storeroom_id UUID NOT NULL REFERENCES storeroom(id),
    dest_storeroom_id   UUID NOT NULL REFERENCES storeroom(id),
    status              VARCHAR(20) NOT NULL DEFAULT 'draft'
                        CHECK (status IN ('draft','approved','in_transit','received','cancelled')),
    requested_by        UUID NOT NULL,
    approved_by         UUID,
    shipped_at          TIMESTAMPTZ,
    received_at         TIMESTAMPTZ,
    notes               TEXT,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    CHECK (source_storeroom_id <> dest_storeroom_id)
);

CREATE TABLE stock_transfer_line (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    stock_transfer_id   UUID NOT NULL REFERENCES stock_transfer(id),
    part_id             UUID NOT NULL REFERENCES part(id),
    quantity_requested  NUMERIC(12,2) NOT NULL,
    quantity_shipped    NUMERIC(12,2),
    quantity_received   NUMERIC(12,2),
    lot_number          VARCHAR(50),
    serial_number       VARCHAR(100),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_stline_transfer ON stock_transfer_line(stock_transfer_id);
CREATE INDEX idx_stline_part ON stock_transfer_line(part_id);
```

## Core Tables — Work Order Management

```sql
-- ============================================================
-- WORK ORDER MANAGEMENT
-- ============================================================

CREATE TABLE failure_mode (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    code            VARCHAR(20) NOT NULL UNIQUE,
    name            VARCHAR(200) NOT NULL,
    equipment_class_id UUID REFERENCES equipment_class(id),
    iso_14224_ref   VARCHAR(50),                        -- ISO 14224 failure mode code
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE failure_cause (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    code            VARCHAR(20) NOT NULL UNIQUE,
    name            VARCHAR(200) NOT NULL,
    iso_14224_ref   VARCHAR(50),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE work_order (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    work_order_number VARCHAR(50) NOT NULL UNIQUE,
    asset_id        UUID NOT NULL REFERENCES asset(id),
    site_id         UUID NOT NULL REFERENCES site(id),
    type            VARCHAR(30) NOT NULL
                    CHECK (type IN ('corrective','preventive','predictive','inspection','emergency')),
    priority        VARCHAR(10) NOT NULL DEFAULT 'medium'
                    CHECK (priority IN ('critical','high','medium','low')),
    status          VARCHAR(20) NOT NULL DEFAULT 'draft'
                    CHECK (status IN ('draft','planned','in_progress','on_hold','completed','cancelled')),
    description     TEXT NOT NULL,
    failure_mode_id UUID REFERENCES failure_mode(id),
    failure_cause_id UUID REFERENCES failure_cause(id),
    requested_by    UUID NOT NULL,
    assigned_to     UUID,
    planned_start   TIMESTAMPTZ,
    planned_end     TIMESTAMPTZ,
    actual_start    TIMESTAMPTZ,
    actual_end      TIMESTAMPTZ,
    downtime_minutes INT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_wo_asset ON work_order(asset_id);
CREATE INDEX idx_wo_site ON work_order(site_id);
CREATE INDEX idx_wo_status ON work_order(status);
CREATE INDEX idx_wo_type ON work_order(type);
CREATE INDEX idx_wo_dates ON work_order(planned_start, planned_end);

-- Parts consumed on work orders
CREATE TABLE work_order_part (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    work_order_id   UUID NOT NULL REFERENCES work_order(id),
    part_id         UUID NOT NULL REFERENCES part(id),
    storeroom_id    UUID NOT NULL REFERENCES storeroom(id),
    quantity_planned NUMERIC(10,2),
    quantity_used   NUMERIC(10,2) NOT NULL DEFAULT 0,
    unit_cost       NUMERIC(14,4),
    serial_number   VARCHAR(100),
    lot_number      VARCHAR(50),
    issued_at       TIMESTAMPTZ,
    issued_by       UUID,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_wopart_wo ON work_order_part(work_order_id);
CREATE INDEX idx_wopart_part ON work_order_part(part_id);

-- Add deferred FK to inventory_transaction
ALTER TABLE inventory_transaction
    ADD CONSTRAINT fk_invtxn_wo FOREIGN KEY (work_order_id) REFERENCES work_order(id);
```

## Core Tables — Procurement

```sql
-- ============================================================
-- SUPPLIER & PROCUREMENT
-- ============================================================

CREATE TABLE supplier (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    code            VARCHAR(50) NOT NULL UNIQUE,
    name            VARCHAR(300) NOT NULL,
    contact_name    VARCHAR(200),
    email           VARCHAR(200),
    phone           VARCHAR(50),
    website         VARCHAR(500),
    address_line1   VARCHAR(300),
    city            VARCHAR(100),
    country_code    CHAR(2),                            -- ISO 3166-1
    payment_terms_days INT DEFAULT 30,
    is_approved     BOOLEAN NOT NULL DEFAULT false,
    rating          NUMERIC(3,2) CHECK (rating BETWEEN 0 AND 5),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE supplier_part (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    supplier_id     UUID NOT NULL REFERENCES supplier(id),
    part_id         UUID NOT NULL REFERENCES part(id),
    supplier_part_number VARCHAR(100),
    unit_price      NUMERIC(14,4) NOT NULL,
    currency_code   CHAR(3) DEFAULT 'USD',
    min_order_qty   NUMERIC(10,2) DEFAULT 1,
    lead_time_days  INT,
    is_preferred    BOOLEAN NOT NULL DEFAULT false,
    contract_ref    VARCHAR(100),
    valid_from      DATE,
    valid_to        DATE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(supplier_id, part_id)
);

CREATE INDEX idx_suppart_supplier ON supplier_part(supplier_id);
CREATE INDEX idx_suppart_part ON supplier_part(part_id);
CREATE INDEX idx_suppart_preferred ON supplier_part(is_preferred) WHERE is_preferred = true;

CREATE TABLE purchase_order (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    po_number       VARCHAR(50) NOT NULL UNIQUE,
    supplier_id     UUID NOT NULL REFERENCES supplier(id),
    site_id         UUID NOT NULL REFERENCES site(id),
    status          VARCHAR(20) NOT NULL DEFAULT 'draft'
                    CHECK (status IN ('draft','submitted','approved','partially_received',
                                      'received','cancelled')),
    order_date      DATE NOT NULL DEFAULT CURRENT_DATE,
    expected_delivery DATE,
    total_amount    NUMERIC(14,2),
    currency_code   CHAR(3) DEFAULT 'USD',
    created_by      UUID NOT NULL,
    approved_by     UUID,
    notes           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_po_supplier ON purchase_order(supplier_id);
CREATE INDEX idx_po_status ON purchase_order(status);

CREATE TABLE purchase_order_line (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    purchase_order_id UUID NOT NULL REFERENCES purchase_order(id),
    line_number     INT NOT NULL,
    part_id         UUID NOT NULL REFERENCES part(id),
    storeroom_id    UUID NOT NULL REFERENCES storeroom(id),
    quantity_ordered NUMERIC(10,2) NOT NULL,
    quantity_received NUMERIC(10,2) NOT NULL DEFAULT 0,
    unit_price      NUMERIC(14,4) NOT NULL,
    line_total      NUMERIC(14,2) GENERATED ALWAYS AS (quantity_ordered * unit_price) STORED,
    expected_delivery DATE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(purchase_order_id, line_number)
);

CREATE INDEX idx_poline_po ON purchase_order_line(purchase_order_id);
CREATE INDEX idx_poline_part ON purchase_order_line(part_id);

-- Add deferred FK to inventory_transaction
ALTER TABLE inventory_transaction
    ADD CONSTRAINT fk_invtxn_poline FOREIGN KEY (purchase_order_line_id)
    REFERENCES purchase_order_line(id);
```

## Core Tables — AI Forecasting & Analytics

```sql
-- ============================================================
-- AI DEMAND FORECASTING
-- ============================================================

CREATE TABLE demand_forecast (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    part_id         UUID NOT NULL REFERENCES part(id),
    storeroom_id    UUID NOT NULL REFERENCES storeroom(id),
    forecast_date   DATE NOT NULL,                      -- date forecast was generated
    period_start    DATE NOT NULL,                       -- start of forecast period
    period_end      DATE NOT NULL,                       -- end of forecast period
    method          VARCHAR(30) NOT NULL
                    CHECK (method IN ('croston','sba','ml_ensemble','moving_average','manual')),
    point_estimate  NUMERIC(12,2) NOT NULL,             -- predicted demand quantity
    confidence_lower NUMERIC(12,2),                     -- lower bound (e.g. 5th percentile)
    confidence_upper NUMERIC(12,2),                     -- upper bound (e.g. 95th percentile)
    confidence_level NUMERIC(3,2) DEFAULT 0.90,         -- e.g. 0.90 for 90% CI
    recommended_reorder_point NUMERIC(12,2),
    recommended_reorder_qty   NUMERIC(12,2),
    model_version   VARCHAR(50),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_forecast_part ON demand_forecast(part_id, storeroom_id);
CREATE INDEX idx_forecast_date ON demand_forecast(forecast_date);

-- Part criticality scoring (dynamic, AI-driven)
CREATE TABLE part_criticality_score (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    part_id         UUID NOT NULL REFERENCES part(id),
    score_date      DATE NOT NULL,
    criticality_score NUMERIC(5,2) NOT NULL,            -- 0-100 scale
    business_impact_score NUMERIC(5,2),
    failure_probability_score NUMERIC(5,2),
    lead_time_risk_score NUMERIC(5,2),
    recommended_criticality VARCHAR(10)
                    CHECK (recommended_criticality IN ('critical','essential','standard','non_critical')),
    model_version   VARCHAR(50),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_critscore_part ON part_criticality_score(part_id, score_date DESC);

-- Supplier lead time tracking
CREATE TABLE supplier_delivery_performance (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    supplier_id     UUID NOT NULL REFERENCES supplier(id),
    part_id         UUID NOT NULL REFERENCES part(id),
    purchase_order_line_id UUID REFERENCES purchase_order_line(id),
    promised_lead_time_days INT,
    actual_lead_time_days   INT,
    quantity_ordered NUMERIC(10,2),
    quantity_received NUMERIC(10,2),
    delivery_date   DATE,
    on_time         BOOLEAN,
    in_full         BOOLEAN,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_supdeliv_supplier ON supplier_delivery_performance(supplier_id);
CREATE INDEX idx_supdeliv_part ON supplier_delivery_performance(part_id);
```

## Core Tables — Multi-Tenant & User Management

```sql
-- ============================================================
-- ORGANISATION & USER MANAGEMENT
-- ============================================================

CREATE TABLE organisation (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(300) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Add org_id to all major tables for multi-tenancy (row-level security)
-- Example: ALTER TABLE site ADD COLUMN org_id UUID NOT NULL REFERENCES organisation(id);

CREATE TABLE app_user (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisation(id),
    email           VARCHAR(300) NOT NULL,
    display_name    VARCHAR(200) NOT NULL,
    role            VARCHAR(30) NOT NULL DEFAULT 'technician'
                    CHECK (role IN ('admin','manager','planner','technician','viewer')),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(org_id, email)
);

CREATE INDEX idx_user_org ON app_user(org_id);

-- Audit log
CREATE TABLE audit_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    table_name      VARCHAR(100) NOT NULL,
    record_id       UUID NOT NULL,
    action          VARCHAR(10) NOT NULL CHECK (action IN ('INSERT','UPDATE','DELETE')),
    old_values      JSONB,
    new_values      JSONB,
    performed_by    UUID REFERENCES app_user(id),
    performed_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_audit_table_record ON audit_log(table_name, record_id);
CREATE INDEX idx_audit_date ON audit_log(performed_at);
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Asset & Equipment | 4 | equipment_class, site, functional_location, asset |
| Taxonomy & Classification | 6 | UNSPSC (4 levels), eclass_class, taxonomy_crossmap |
| Parts Catalogue | 4 | part, part_alternate, asset_bom, unit_of_measure |
| Inventory & Storeroom | 7 | storeroom, bin_location, inventory_balance, part_lot, part_serial, inventory_transaction, stock_transfer + lines |
| Work Orders | 4 | work_order, work_order_part, failure_mode, failure_cause |
| Procurement | 4 | supplier, supplier_part, purchase_order, purchase_order_line |
| AI & Forecasting | 3 | demand_forecast, part_criticality_score, supplier_delivery_performance |
| Users & Org | 3 | organisation, app_user, audit_log |
| **Total** | **~35** | |

---

## Key Design Decisions

1. **UUID primary keys throughout** — enables distributed ID generation for multi-site deployments and avoids sequential ID enumeration attacks on APIs.

2. **Separate UNSPSC hierarchy tables** — rather than a single flattened lookup, the four-level hierarchy (segment/family/class/commodity) is modelled as linked tables, enabling roll-up analytics and drill-down queries at any level.

3. **Dual taxonomy support (UNSPSC + eCl@ss)** — with a `taxonomy_crossmap` table allowing ML-generated or manual mappings between the two standards, reflecting real-world MRO catalogue requirements.

4. **Inventory balance as a materialised summary** — `inventory_balance` is the current-state snapshot, while `inventory_transaction` is the append-only movement log. The balance is updated transactionally when transactions are recorded.

5. **Separate lot and serial tracking** — `part_lot` for batch-tracked items (GS1 AI 10) and `part_serial` for individually serialized items including rotable spares, each with their own lifecycle status.

6. **ISO 14224 failure taxonomy** — `failure_mode` and `failure_cause` tables with ISO 14224 reference codes enable standardised reliability data collection that feeds back into demand forecasting.

7. **AI forecasting with confidence intervals** — the `demand_forecast` table stores not just point estimates but upper/lower bounds and confidence levels, addressing the transparency gap identified in competitor analysis.

8. **Criticality scoring as a time-series** — `part_criticality_score` stores dated scores so the system can track how criticality changes over time as equipment condition evolves.

9. **Supplier delivery performance tracking** — every delivery is logged with promised vs. actual lead times, enabling the ML-based supplier risk scoring described in the features specification.

10. **Row-level security ready** — the `organisation` table and `org_id` foreign key pattern enables PostgreSQL row-level security policies for multi-tenant isolation.
