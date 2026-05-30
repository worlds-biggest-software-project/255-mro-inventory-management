# Data Model Suggestion 3: Hybrid Relational + JSONB

> Project: MRO Inventory Management · Created: 2026-05-22

## Philosophy

This model uses PostgreSQL relational tables for core operational data (inventory balances, work orders, purchase orders) while leveraging JSONB columns for extensible, variable, and domain-specific attributes. The key insight is that MRO inventory management spans wildly different industries — aviation, oil & gas, manufacturing, utilities, facilities management — and each industry has unique attributes, compliance requirements, and classification schemes that a fixed relational schema cannot anticipate.

Rather than forcing all variation into the relational layer (which leads to hundreds of nullable columns or complex EAV patterns) or abandoning structure entirely (document-oriented), this hybrid approach gives you the best of both worlds: relational integrity for the data that matters most (quantities, costs, foreign keys, timestamps) and schema-on-write flexibility for everything else. PostgreSQL's JSONB type supports GIN indexing, containment queries, and partial path extraction, making it practical for both storage and retrieval.

This pattern is used by modern SaaS platforms that serve multiple industries from a single codebase. Shopify's product model, Stripe's metadata fields, and Salesforce's custom fields all follow variants of this approach. For MRO specifically, it means a single deployment can serve an aviation MRO shop tracking flight hours and cycle counts alongside a manufacturing plant tracking operating hours and vibration thresholds — without separate schemas or code paths.

**Best for:** Multi-industry SaaS deployments, rapid MVP development, organisations that need tenant-specific or industry-specific fields without schema migrations.

**Trade-offs:**
- Pro: Core data integrity maintained through relational constraints
- Pro: Industry-specific and tenant-specific attributes without schema migrations
- Pro: Faster time-to-market for MVP — new fields are configuration, not DDL
- Pro: GIN-indexed JSONB queries perform well for containment and equality checks
- Pro: Enables tenant-level customisation (custom fields per organisation)
- Con: JSONB fields are not type-checked at the database level — validation must happen in application code or via JSON Schema
- Con: Reporting tools may struggle with JSONB path queries compared to flat columns
- Con: Discipline required to avoid "JSONB as junk drawer" — must define and document JSONB schemas
- Con: Complex JSONB queries (range comparisons, joins on nested values) are slower than columnar equivalents
- Con: Foreign keys cannot reference values inside JSONB — referential integrity for JSONB data is application-enforced

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| ISO 55001:2024 | Core asset lifecycle tables are relational; asset-specific attributes (calibration data, regulatory data) in `asset.extended_attributes` JSONB |
| ISO 14224:2016 | Equipment taxonomy in relational hierarchy; industry-specific equipment attributes in `asset.extended_attributes` keyed by ISO 14224 level |
| UNSPSC / eCl@ss | Classification codes as relational columns on `catalogue_item`; classification-specific properties in `catalogue_item.classification_attributes` JSONB |
| GS1/GTIN | GTIN as relational column; GS1 Application Identifier payloads stored in `barcode_event.gs1_data` JSONB |
| SAE JA1011 (RCM) | Criticality scoring relational; RCM analysis details in `part_criticality.analysis_data` JSONB |
| OSHA 29 CFR 1910.147 | Compliance procedures relational; procedure-specific steps and checklists in `compliance_procedure.steps` JSONB |
| JSON Schema (Draft 2020-12) | Used to define and validate the structure of all JSONB columns via `field_definition` table |
| OpenAPI 3.1 | API schema references JSON Schema definitions for JSONB fields, enabling client code generation |

---

## Custom Field System

```sql
-- ============================================================
-- MULTI-TENANT ORGANISATION
-- ============================================================

CREATE TABLE organisation (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            TEXT NOT NULL,
    slug            TEXT NOT NULL UNIQUE,
    settings        JSONB NOT NULL DEFAULT '{}',
    -- settings example:
    -- {
    --   "industry": "aviation",
    --   "default_currency": "USD",
    --   "timezone": "America/Chicago",
    --   "features_enabled": ["rotable_spares", "iot_sensors", "ai_forecasting"]
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- ============================================================
-- CUSTOM FIELD DEFINITIONS (per-tenant schema configuration)
-- ============================================================

CREATE TABLE field_definition (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisation(id),
    entity_type     TEXT NOT NULL,
    -- Which entity this field applies to: 'asset', 'catalogue_item',
    -- 'work_order', 'storeroom', 'vendor', etc.
    field_key       TEXT NOT NULL,                          -- JSON path key
    display_name    TEXT NOT NULL,
    field_type      TEXT NOT NULL
                    CHECK (field_type IN ('text', 'number', 'boolean', 'date',
                                          'select', 'multi_select', 'url', 'json')),
    is_required     BOOLEAN NOT NULL DEFAULT false,
    default_value   JSONB,
    validation      JSONB,
    -- validation example:
    -- { "min": 0, "max": 100000, "regex": null }
    -- or for select: { "options": ["serviceable", "unserviceable", "repairable"] }
    sort_order      INTEGER NOT NULL DEFAULT 0,
    is_searchable   BOOLEAN NOT NULL DEFAULT false,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(org_id, entity_type, field_key)
);

CREATE INDEX idx_fielddef_org_entity ON field_definition(org_id, entity_type);
```

---

## Asset & Equipment Management

```sql
-- ============================================================
-- SITE & FUNCTIONAL LOCATION
-- ============================================================

CREATE TABLE site (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisation(id),
    code            TEXT NOT NULL,
    name            TEXT NOT NULL,
    country_code    CHAR(2) NOT NULL,                      -- ISO 3166-1
    timezone        TEXT NOT NULL DEFAULT 'UTC',
    address         JSONB NOT NULL DEFAULT '{}',
    -- address example:
    -- {
    --   "line1": "1200 Industrial Parkway",
    --   "line2": "Building C",
    --   "city": "Houston",
    --   "state": "TX",
    --   "postal_code": "77001",
    --   "country": "US",
    --   "coordinates": { "lat": 29.7604, "lng": -95.3698 }
    -- }
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(org_id, code)
);

CREATE TABLE functional_location (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    site_id         UUID NOT NULL REFERENCES site(id),
    parent_id       UUID REFERENCES functional_location(id),
    code            TEXT NOT NULL,
    name            TEXT NOT NULL,
    path            TEXT NOT NULL,                          -- materialised path: 'plant.area3.line2'
    depth           SMALLINT NOT NULL DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(site_id, code)
);

CREATE INDEX idx_floc_path ON functional_location USING gist (path gist_trgm_ops);

-- ============================================================
-- EQUIPMENT TAXONOMY (ISO 14224 hierarchy)
-- ============================================================

CREATE TABLE equipment_class (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisation(id),
    parent_id       UUID REFERENCES equipment_class(id),
    code            TEXT NOT NULL,
    name            TEXT NOT NULL,
    level           SMALLINT NOT NULL CHECK (level BETWEEN 1 AND 9),
    iso_14224_ref   TEXT,
    default_attributes JSONB NOT NULL DEFAULT '{}',
    -- default_attributes example (for a pump class):
    -- {
    --   "pump_type": null,
    --   "rated_flow_m3h": null,
    --   "rated_pressure_bar": null,
    --   "driver_type": null,
    --   "seal_type": null
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(org_id, code)
);

-- ============================================================
-- ASSET (relational core + JSONB extensions)
-- ============================================================

CREATE TABLE asset (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id              UUID NOT NULL REFERENCES organisation(id),
    site_id             UUID NOT NULL REFERENCES site(id),
    functional_location_id UUID REFERENCES functional_location(id),
    equipment_class_id  UUID REFERENCES equipment_class(id),
    asset_number        TEXT NOT NULL,
    name                TEXT NOT NULL,
    description         TEXT,
    -- Core relational fields (queried frequently, constrained)
    status              TEXT NOT NULL DEFAULT 'active'
                        CHECK (status IN ('active', 'inactive', 'decommissioned',
                                          'under_maintenance', 'standby')),
    criticality         TEXT CHECK (criticality IN ('critical', 'essential',
                                                    'desirable', 'run_to_failure')),
    manufacturer        TEXT,
    model               TEXT,
    serial_number       TEXT,
    install_date        DATE,
    warranty_expiry     DATE,
    replacement_cost    NUMERIC(14, 2),
    parent_asset_id     UUID REFERENCES asset(id),

    -- JSONB extension: industry-specific and custom attributes
    extended_attributes JSONB NOT NULL DEFAULT '{}',
    -- Aviation example:
    -- {
    --   "aircraft_type": "B737-800",
    --   "registration": "N12345",
    --   "total_flight_hours": 45230.5,
    --   "total_cycles": 22100,
    --   "next_c_check_hours": 48000,
    --   "aog_contact": "+1-555-0199"
    -- }
    --
    -- Manufacturing example:
    -- {
    --   "operating_hours": 18450,
    --   "vibration_baseline_mm_s": 2.8,
    --   "lubrication_schedule": "monthly",
    --   "rated_capacity_tons": 500,
    --   "plc_tag": "PUMP-A3-001"
    -- }

    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(org_id, asset_number)
);

CREATE INDEX idx_asset_site ON asset(site_id);
CREATE INDEX idx_asset_status ON asset(org_id, status);
CREATE INDEX idx_asset_criticality ON asset(criticality) WHERE criticality IS NOT NULL;
CREATE INDEX idx_asset_parent ON asset(parent_asset_id) WHERE parent_asset_id IS NOT NULL;
-- GIN index on extended_attributes for JSONB containment queries
CREATE INDEX idx_asset_ext_attrs ON asset USING gin (extended_attributes jsonb_path_ops);
```

---

## Catalogue & Inventory

```sql
-- ============================================================
-- CATALOGUE ITEM (relational core + JSONB for specifications)
-- ============================================================

CREATE TABLE catalogue_item (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id              UUID NOT NULL REFERENCES organisation(id),
    part_number         TEXT NOT NULL,
    name                TEXT NOT NULL,
    description         TEXT,

    -- Core relational fields
    unspsc_code         TEXT,                               -- UNSPSC 8-digit code
    eclass_code         TEXT,                               -- eClass IRDI
    gtin                TEXT,                               -- GS1 GTIN-14
    manufacturer        TEXT,
    manufacturer_part_number TEXT,
    unit_of_measure     TEXT NOT NULL DEFAULT 'EA',
    unit_cost           NUMERIC(14, 4),
    currency_code       CHAR(3) DEFAULT 'USD',
    weight_kg           NUMERIC(10, 4),
    is_rotable          BOOLEAN NOT NULL DEFAULT false,
    is_serialised       BOOLEAN NOT NULL DEFAULT false,
    criticality         TEXT CHECK (criticality IN ('critical', 'essential',
                                                    'desirable', 'run_to_failure')),
    status              TEXT NOT NULL DEFAULT 'active'
                        CHECK (status IN ('active', 'obsolete', 'pending_review',
                                          'duplicate_suspect')),

    -- JSONB: classification-specific properties
    classification_attributes JSONB NOT NULL DEFAULT '{}',
    -- Example for bearing (UNSPSC 31171500):
    -- {
    --   "bearing_type": "deep_groove_ball",
    --   "bore_diameter_mm": 25,
    --   "outer_diameter_mm": 52,
    --   "width_mm": 15,
    --   "dynamic_load_rating_kn": 14.8,
    --   "seal_type": "2Z",
    --   "material": "chrome_steel"
    -- }

    -- JSONB: custom fields defined by the tenant
    custom_fields JSONB NOT NULL DEFAULT '{}',
    -- Example:
    -- {
    --   "hazmat_class": "non_hazardous",
    --   "shelf_life_months": 36,
    --   "storage_temperature_max_c": 40,
    --   "customs_tariff_code": "8482.10.50"
    -- }

    -- JSONB: supplier catalogue cross-references
    supplier_references JSONB NOT NULL DEFAULT '[]',
    -- Example:
    -- [
    --   { "supplier": "SKF", "supplier_part": "6205-2Z", "price": 12.50 },
    --   { "supplier": "FAG", "supplier_part": "6205.2ZR", "price": 11.80 }
    -- ]

    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(org_id, part_number)
);

CREATE INDEX idx_cat_item_unspsc ON catalogue_item(unspsc_code) WHERE unspsc_code IS NOT NULL;
CREATE INDEX idx_cat_item_gtin ON catalogue_item(gtin) WHERE gtin IS NOT NULL;
CREATE INDEX idx_cat_item_mfr ON catalogue_item(org_id, manufacturer);
CREATE INDEX idx_cat_item_status ON catalogue_item(org_id, status);
CREATE INDEX idx_cat_item_class_attrs ON catalogue_item USING gin (classification_attributes jsonb_path_ops);
CREATE INDEX idx_cat_item_custom ON catalogue_item USING gin (custom_fields jsonb_path_ops);
-- Full-text search across name + description
CREATE INDEX idx_cat_item_fts ON catalogue_item USING gin (
    to_tsvector('english', name || ' ' || COALESCE(description, ''))
);

-- ============================================================
-- STOREROOM & INVENTORY BALANCE
-- ============================================================

CREATE TABLE storeroom (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    site_id         UUID NOT NULL REFERENCES site(id),
    code            TEXT NOT NULL,
    name            TEXT NOT NULL,
    storeroom_type  TEXT NOT NULL DEFAULT 'central'
                    CHECK (storeroom_type IN ('central', 'satellite', 'vehicle',
                                              'consignment', 'vendor_managed')),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    config          JSONB NOT NULL DEFAULT '{}',
    -- config example:
    -- {
    --   "auto_reorder": true,
    --   "cycle_count_frequency_days": 90,
    --   "default_bin_scheme": "aisle-rack-shelf",
    --   "supports_rfid": false
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(site_id, code)
);

CREATE TABLE bin_location (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    storeroom_id    UUID NOT NULL REFERENCES storeroom(id),
    code            TEXT NOT NULL,
    description     TEXT,
    UNIQUE(storeroom_id, code)
);

CREATE TABLE inventory_balance (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    catalogue_item_id   UUID NOT NULL REFERENCES catalogue_item(id),
    storeroom_id        UUID NOT NULL REFERENCES storeroom(id),
    bin_location_id     UUID REFERENCES bin_location(id),

    -- Core quantities (relational, constrained)
    quantity_on_hand    NUMERIC(12, 4) NOT NULL DEFAULT 0 CHECK (quantity_on_hand >= 0),
    quantity_reserved   NUMERIC(12, 4) NOT NULL DEFAULT 0 CHECK (quantity_reserved >= 0),
    quantity_on_order   NUMERIC(12, 4) NOT NULL DEFAULT 0 CHECK (quantity_on_order >= 0),

    -- Reorder parameters (may be AI-set or manual)
    reorder_point       NUMERIC(12, 4),
    reorder_quantity    NUMERIC(12, 4),
    max_level           NUMERIC(12, 4),
    safety_stock        NUMERIC(12, 4),
    reorder_source      TEXT DEFAULT 'manual'
                        CHECK (reorder_source IN ('manual', 'ai_forecast', 'system_default')),

    -- Cost tracking
    average_unit_cost   NUMERIC(14, 4),
    total_value         NUMERIC(14, 4) GENERATED ALWAYS AS (quantity_on_hand * average_unit_cost) STORED,

    -- Usage metrics
    last_issue_at       TIMESTAMPTZ,
    last_receipt_at     TIMESTAMPTZ,
    last_count_at       TIMESTAMPTZ,
    avg_monthly_usage   NUMERIC(12, 4),

    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(catalogue_item_id, storeroom_id)
);

CREATE INDEX idx_invbal_storeroom ON inventory_balance(storeroom_id);
CREATE INDEX idx_invbal_low_stock ON inventory_balance(catalogue_item_id)
    WHERE quantity_on_hand <= reorder_point AND reorder_point IS NOT NULL;

-- ============================================================
-- INVENTORY TRANSACTION LOG
-- ============================================================

CREATE TABLE inventory_transaction (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id              UUID NOT NULL REFERENCES organisation(id),
    inventory_balance_id UUID NOT NULL REFERENCES inventory_balance(id),
    transaction_type    TEXT NOT NULL
                        CHECK (transaction_type IN (
                            'receipt', 'issue', 'return', 'adjustment',
                            'transfer_out', 'transfer_in', 'cycle_count',
                            'scrap', 'reservation', 'unreservation'
                        )),
    quantity            NUMERIC(12, 4) NOT NULL,
    unit_cost           NUMERIC(14, 4),

    -- Flexible references via JSONB instead of multiple nullable FKs
    reference           JSONB NOT NULL DEFAULT '{}',
    -- Example for issue:
    -- {
    --   "work_order_id": "uuid-here",
    --   "work_order_number": "WO-2026-0421",
    --   "asset_id": "uuid-here",
    --   "asset_number": "PUMP-A3-001"
    -- }
    -- Example for receipt:
    -- {
    --   "purchase_order_id": "uuid-here",
    --   "po_number": "PO-2026-0088",
    --   "po_line": 3,
    --   "supplier_name": "SKF Distributor"
    -- }

    lot_number          TEXT,
    serial_number       TEXT,
    performed_by        UUID NOT NULL,
    notes               TEXT,
    transacted_at       TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (transacted_at);

CREATE INDEX idx_invtxn_balance ON inventory_transaction(inventory_balance_id);
CREATE INDEX idx_invtxn_type ON inventory_transaction(org_id, transaction_type);
CREATE INDEX idx_invtxn_date ON inventory_transaction(transacted_at);
CREATE INDEX idx_invtxn_ref ON inventory_transaction USING gin (reference jsonb_path_ops);

-- ============================================================
-- INTER-SITE TRANSFER
-- ============================================================

CREATE TABLE inventory_transfer (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id              UUID NOT NULL REFERENCES organisation(id),
    transfer_number     TEXT NOT NULL,
    from_storeroom_id   UUID NOT NULL REFERENCES storeroom(id),
    to_storeroom_id     UUID NOT NULL REFERENCES storeroom(id),
    status              TEXT NOT NULL DEFAULT 'requested'
                        CHECK (status IN ('requested', 'approved', 'in_transit',
                                          'received', 'cancelled')),
    requested_by        UUID NOT NULL,
    approved_by         UUID,
    shipped_at          TIMESTAMPTZ,
    received_at         TIMESTAMPTZ,
    lines               JSONB NOT NULL DEFAULT '[]',
    -- lines example:
    -- [
    --   {
    --     "catalogue_item_id": "uuid",
    --     "part_number": "BRG-6205-2Z",
    --     "qty_requested": 10,
    --     "qty_shipped": 10,
    --     "qty_received": 10
    --   },
    --   {
    --     "catalogue_item_id": "uuid",
    --     "part_number": "FLT-HYD-001",
    --     "qty_requested": 5,
    --     "qty_shipped": 5,
    --     "qty_received": null
    --   }
    -- ]
    notes               TEXT,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(org_id, transfer_number)
);
```

---

## Work Orders & Maintenance

```sql
-- ============================================================
-- WORK ORDER (relational core + JSONB extensions)
-- ============================================================

CREATE TABLE work_order (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id              UUID NOT NULL REFERENCES organisation(id),
    site_id             UUID NOT NULL REFERENCES site(id),
    wo_number           TEXT NOT NULL,
    asset_id            UUID REFERENCES asset(id),

    -- Core relational fields
    work_type           TEXT NOT NULL
                        CHECK (work_type IN ('corrective', 'preventive', 'predictive',
                                             'inspection', 'emergency', 'project')),
    priority            TEXT NOT NULL DEFAULT 'medium'
                        CHECK (priority IN ('critical', 'high', 'medium', 'low')),
    status              TEXT NOT NULL DEFAULT 'draft'
                        CHECK (status IN ('draft', 'awaiting_parts', 'scheduled',
                                          'in_progress', 'on_hold', 'completed',
                                          'closed', 'cancelled')),
    description         TEXT NOT NULL,
    assigned_to         UUID,
    scheduled_start     TIMESTAMPTZ,
    scheduled_end       TIMESTAMPTZ,
    actual_start        TIMESTAMPTZ,
    actual_end          TIMESTAMPTZ,
    downtime_minutes    INTEGER,

    -- JSONB: failure analysis data (ISO 14224)
    failure_data        JSONB NOT NULL DEFAULT '{}',
    -- Example:
    -- {
    --   "failure_mode_code": "BRG_WEAR",
    --   "failure_cause_code": "LUBRICATION_FAILURE",
    --   "iso_14224_failure_mode": "1.3.2",
    --   "iso_14224_failure_cause": "2.1.4",
    --   "detection_method": "vibration_monitoring",
    --   "consequence": "production_loss",
    --   "root_cause_analysis": "Grease degradation due to high ambient temperature"
    -- }

    -- JSONB: parts planned and consumed
    parts               JSONB NOT NULL DEFAULT '[]',
    -- Example:
    -- [
    --   {
    --     "catalogue_item_id": "uuid",
    --     "part_number": "BRG-6205-2Z",
    --     "name": "Deep Groove Ball Bearing 6205-2Z",
    --     "storeroom_id": "uuid",
    --     "qty_planned": 2,
    --     "qty_used": 2,
    --     "unit_cost": 12.50,
    --     "serial_numbers": ["SN-001", "SN-002"]
    --   }
    -- ]

    -- JSONB: labour records
    labour              JSONB NOT NULL DEFAULT '[]',
    -- Example:
    -- [
    --   {
    --     "user_id": "uuid",
    --     "name": "John Smith",
    --     "craft": "mechanic",
    --     "hours": 4.5,
    --     "rate": 65.00,
    --     "start": "2026-03-15T08:00:00Z",
    --     "end": "2026-03-15T12:30:00Z"
    --   }
    -- ]

    -- JSONB: custom fields per tenant
    custom_fields       JSONB NOT NULL DEFAULT '{}',

    -- Source tracking
    source              TEXT DEFAULT 'manual'
                        CHECK (source IN ('manual', 'pm_schedule', 'iot_alert',
                                          'ai_predicted', 'inspection')),
    source_ref          JSONB,
    -- Example for IoT-triggered:
    -- {
    --   "sensor_id": "uuid",
    --   "alert_type": "vibration_threshold",
    --   "reading_value": 12.5,
    --   "threshold": 10.0,
    --   "detected_at": "2026-03-15T14:22:00Z"
    -- }

    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(org_id, wo_number)
);

CREATE INDEX idx_wo_asset ON work_order(asset_id);
CREATE INDEX idx_wo_status ON work_order(org_id, status);
CREATE INDEX idx_wo_type ON work_order(work_type);
CREATE INDEX idx_wo_dates ON work_order(scheduled_start, scheduled_end);
CREATE INDEX idx_wo_failure ON work_order USING gin (failure_data jsonb_path_ops);
CREATE INDEX idx_wo_parts ON work_order USING gin (parts jsonb_path_ops);

-- ============================================================
-- PREVENTIVE MAINTENANCE SCHEDULE
-- ============================================================

CREATE TABLE pm_schedule (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id              UUID NOT NULL REFERENCES organisation(id),
    asset_id            UUID NOT NULL REFERENCES asset(id),
    name                TEXT NOT NULL,
    frequency           JSONB NOT NULL,
    -- Calendar-based:
    -- { "type": "calendar", "interval_days": 90 }
    -- Meter-based:
    -- { "type": "meter", "meter_type": "operating_hours", "interval": 500 }
    -- Condition-based:
    -- { "type": "condition", "sensor_type": "vibration", "threshold_mm_s": 10.0 }
    parts_list          JSONB NOT NULL DEFAULT '[]',
    -- [
    --   { "catalogue_item_id": "uuid", "part_number": "FLT-OIL-001", "quantity": 1 },
    --   { "catalogue_item_id": "uuid", "part_number": "GSK-PUMP-003", "quantity": 2 }
    -- ]
    instructions        JSONB NOT NULL DEFAULT '[]',
    -- [
    --   { "step": 1, "action": "Lock out equipment per LOTO procedure LP-042" },
    --   { "step": 2, "action": "Drain oil sump and dispose per EPA guidelines" },
    --   { "step": 3, "action": "Replace oil filter and gaskets" }
    -- ]
    is_active           BOOLEAN NOT NULL DEFAULT true,
    next_due            TIMESTAMPTZ,
    last_completed      TIMESTAMPTZ,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_pm_asset ON pm_schedule(asset_id);
CREATE INDEX idx_pm_next_due ON pm_schedule(next_due) WHERE is_active = true;
```

---

## Supplier & Procurement

```sql
-- ============================================================
-- VENDOR
-- ============================================================

CREATE TABLE vendor (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id              UUID NOT NULL REFERENCES organisation(id),
    code                TEXT NOT NULL,
    name                TEXT NOT NULL,
    contact             JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "name": "Jane Doe",
    --   "email": "jane@supplier.com",
    --   "phone": "+1-555-0100",
    --   "role": "account_manager"
    -- }
    address             JSONB NOT NULL DEFAULT '{}',
    payment_terms       TEXT,
    currency_code       CHAR(3) DEFAULT 'USD',
    is_active           BOOLEAN NOT NULL DEFAULT true,
    performance_metrics JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "on_time_delivery_rate": 0.92,
    --   "quality_reject_rate": 0.01,
    --   "avg_lead_time_days": 14,
    --   "risk_score": 22.5,
    --   "last_assessed": "2026-03-01"
    -- }
    custom_fields       JSONB NOT NULL DEFAULT '{}',
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(org_id, code)
);

-- ============================================================
-- PURCHASE ORDER (relational header + JSONB lines)
-- ============================================================

CREATE TABLE purchase_order (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id              UUID NOT NULL REFERENCES organisation(id),
    po_number           TEXT NOT NULL,
    vendor_id           UUID NOT NULL REFERENCES vendor(id),
    site_id             UUID NOT NULL REFERENCES site(id),
    status              TEXT NOT NULL DEFAULT 'draft'
                        CHECK (status IN ('draft', 'submitted', 'approved', 'sent',
                                          'partially_received', 'received',
                                          'cancelled', 'closed')),
    order_date          DATE NOT NULL DEFAULT CURRENT_DATE,
    expected_delivery   DATE,
    currency_code       CHAR(3) DEFAULT 'USD',
    total_amount        NUMERIC(14, 4),
    requested_by        UUID NOT NULL,
    approved_by         UUID,

    -- JSONB: order lines (denormalised for simplicity at MVP)
    lines               JSONB NOT NULL DEFAULT '[]',
    -- [
    --   {
    --     "line": 1,
    --     "catalogue_item_id": "uuid",
    --     "part_number": "BRG-6205-2Z",
    --     "description": "Deep Groove Ball Bearing",
    --     "quantity_ordered": 50,
    --     "quantity_received": 25,
    --     "unit_price": 11.80,
    --     "storeroom_id": "uuid",
    --     "storeroom_code": "MAIN-01"
    --   }
    -- ]

    notes               TEXT,
    custom_fields       JSONB NOT NULL DEFAULT '{}',
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(org_id, po_number)
);

CREATE INDEX idx_po_vendor ON purchase_order(vendor_id);
CREATE INDEX idx_po_status ON purchase_order(org_id, status);
CREATE INDEX idx_po_lines ON purchase_order USING gin (lines jsonb_path_ops);
```

---

## AI & Forecasting

```sql
-- ============================================================
-- AI DEMAND FORECAST
-- ============================================================

CREATE TABLE demand_forecast (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id              UUID NOT NULL REFERENCES organisation(id),
    catalogue_item_id   UUID NOT NULL REFERENCES catalogue_item(id),
    storeroom_id        UUID NOT NULL REFERENCES storeroom(id),
    forecast_date       DATE NOT NULL,

    -- Core prediction values (relational for easy querying)
    predicted_demand    NUMERIC(12, 4) NOT NULL,
    confidence_lower    NUMERIC(12, 4),
    confidence_upper    NUMERIC(12, 4),
    confidence_level    NUMERIC(3, 2) DEFAULT 0.90,
    recommended_rop     NUMERIC(12, 4),
    recommended_roq     NUMERIC(12, 4),

    -- JSONB: model details and explanation
    model_details       JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "model_version": "v2.3.1",
    --   "method": "croston_sba",
    --   "training_window_months": 24,
    --   "features_used": [
    --     "historical_consumption",
    --     "equipment_age",
    --     "seasonal_pattern",
    --     "sensor_vibration_trend"
    --   ],
    --   "explanation": "Demand increase driven by aging compressor fleet...",
    --   "accuracy_metrics": {
    --     "mae": 1.2,
    --     "mape": 0.15,
    --     "bias": 0.3
    --   }
    -- }

    status              TEXT NOT NULL DEFAULT 'pending'
                        CHECK (status IN ('pending', 'accepted', 'overridden', 'expired')),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_forecast_item ON demand_forecast(catalogue_item_id, storeroom_id, forecast_date DESC);

-- ============================================================
-- PART CRITICALITY (RCM-aligned with AI scoring details in JSONB)
-- ============================================================

CREATE TABLE part_criticality (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    catalogue_item_id   UUID NOT NULL REFERENCES catalogue_item(id),
    asset_id            UUID REFERENCES asset(id),
    criticality_class   TEXT NOT NULL
                        CHECK (criticality_class IN ('critical', 'essential',
                                                      'desirable', 'run_to_failure')),
    composite_score     NUMERIC(5, 2) NOT NULL,

    -- JSONB: detailed scoring breakdown
    analysis_data       JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "safety_impact": 4,
    --   "production_impact": 5,
    --   "environmental_impact": 2,
    --   "lead_time_risk": 3,
    --   "failure_frequency": 2.3,
    --   "mean_time_to_repair_hours": 8,
    --   "annual_cost_of_failure": 45000,
    --   "assessment_method": "ai_model_v1.2",
    --   "confidence": 0.87
    -- }

    assessed_at         TIMESTAMPTZ NOT NULL DEFAULT now(),
    assessed_by         UUID
);

CREATE INDEX idx_crit_item ON part_criticality(catalogue_item_id);
CREATE INDEX idx_crit_class ON part_criticality(criticality_class);

-- ============================================================
-- SUPPLIER RISK SCORING
-- ============================================================

CREATE TABLE supplier_risk_assessment (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    vendor_id           UUID NOT NULL REFERENCES vendor(id),
    risk_score          NUMERIC(5, 2) NOT NULL,
    risk_level          TEXT NOT NULL CHECK (risk_level IN ('low', 'medium', 'high', 'critical')),
    assessment_data     JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "on_time_rate": 0.85,
    --   "avg_lead_time_days": 18,
    --   "lead_time_variance_days": 5.2,
    --   "quality_reject_rate": 0.03,
    --   "financial_risk_indicators": {
    --     "credit_rating": "BBB",
    --     "payment_delays_90d": 0
    --   },
    --   "parts_at_risk": [
    --     { "part_number": "VLV-SOL-042", "reason": "single_source" },
    --     { "part_number": "FLT-HYD-001", "reason": "lead_time_increasing" }
    --   ]
    -- }
    model_version       TEXT,
    assessed_at         TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_risk_vendor ON supplier_risk_assessment(vendor_id, assessed_at DESC);
```

---

## IoT & Sensors

```sql
-- ============================================================
-- IoT SENSOR REGISTRATION
-- ============================================================

CREATE TABLE sensor (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    asset_id        UUID NOT NULL REFERENCES asset(id),
    sensor_type     TEXT NOT NULL,
    config          JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "manufacturer": "Fluke",
    --   "model": "3561 FC",
    --   "serial": "FLK-2026-4421",
    --   "unit": "mm/s",
    --   "sampling_interval_seconds": 60,
    --   "thresholds": {
    --     "warning": 7.0,
    --     "critical": 12.0
    --   }
    -- }
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_sensor_asset ON sensor(asset_id);

-- ============================================================
-- SENSOR READINGS (time-series, partitioned)
-- ============================================================

CREATE TABLE sensor_reading (
    sensor_id       UUID NOT NULL REFERENCES sensor(id),
    reading_at      TIMESTAMPTZ NOT NULL,
    value           DOUBLE PRECISION NOT NULL,
    is_anomaly      BOOLEAN NOT NULL DEFAULT false,
    anomaly_data    JSONB
    -- { "type": "spike", "baseline": 3.2, "deviation_sigma": 4.1 }
) PARTITION BY RANGE (reading_at);

CREATE INDEX idx_reading_sensor_time ON sensor_reading(sensor_id, reading_at DESC);
CREATE INDEX idx_reading_anomaly ON sensor_reading(sensor_id, reading_at)
    WHERE is_anomaly = true;
```

---

## Users, Compliance & Audit

```sql
-- ============================================================
-- USERS
-- ============================================================

CREATE TABLE app_user (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisation(id),
    email           TEXT NOT NULL,
    display_name    TEXT NOT NULL,
    role            TEXT NOT NULL DEFAULT 'technician'
                    CHECK (role IN ('admin', 'manager', 'planner', 'technician',
                                    'storekeeper', 'viewer')),
    permissions     JSONB NOT NULL DEFAULT '[]',
    -- ["inventory.issue", "work_order.create", "purchase_order.approve"]
    site_access     UUID[],                                 -- array of site IDs; null = all sites
    is_active       BOOLEAN NOT NULL DEFAULT true,
    preferences     JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(org_id, email)
);

-- ============================================================
-- COMPLIANCE PROCEDURE
-- ============================================================

CREATE TABLE compliance_procedure (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisation(id),
    name            TEXT NOT NULL,
    procedure_type  TEXT NOT NULL,
    regulation_ref  TEXT,
    steps           JSONB NOT NULL DEFAULT '[]',
    -- [
    --   { "step": 1, "action": "De-energize equipment", "requires_photo": false },
    --   { "step": 2, "action": "Apply lockout device", "requires_photo": true },
    --   { "step": 3, "action": "Verify zero energy state", "requires_photo": false }
    -- ]
    applicable_assets JSONB NOT NULL DEFAULT '{}',
    -- { "equipment_classes": ["rotating_machinery", "pressure_vessel"] }
    review_interval_days INTEGER,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- ============================================================
-- AUDIT LOG
-- ============================================================

CREATE TABLE audit_log (
    id              BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    org_id          UUID NOT NULL,
    user_id         UUID,
    action          TEXT NOT NULL,
    entity_type     TEXT NOT NULL,
    entity_id       UUID,
    changes         JSONB,
    -- {
    --   "old": { "quantity_on_hand": 25, "reorder_point": 10 },
    --   "new": { "quantity_on_hand": 23, "reorder_point": 12 }
    -- }
    context         JSONB NOT NULL DEFAULT '{}',
    -- { "ip": "10.0.1.50", "user_agent": "MRO-Mobile/2.1", "api_version": "v2" }
    performed_at    TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (performed_at);

CREATE INDEX idx_audit_entity ON audit_log(entity_type, entity_id);
CREATE INDEX idx_audit_user ON audit_log(user_id);
CREATE INDEX idx_audit_date ON audit_log(performed_at);
```

---

## JSONB Query Examples

```sql
-- ============================================================
-- EXAMPLE QUERIES LEVERAGING JSONB
-- ============================================================

-- Q1: Find all aviation assets with total flight hours > 40000
SELECT id, asset_number, name,
       extended_attributes->>'total_flight_hours' AS flight_hours,
       extended_attributes->>'next_c_check_hours' AS next_c_check
FROM asset
WHERE extended_attributes @> '{"aircraft_type": "B737-800"}'
  AND (extended_attributes->>'total_flight_hours')::NUMERIC > 40000;

-- Q2: Find catalogue items by classification attribute (bearing bore diameter)
SELECT part_number, name,
       classification_attributes->>'bearing_type' AS bearing_type,
       classification_attributes->>'bore_diameter_mm' AS bore_mm
FROM catalogue_item
WHERE classification_attributes @> '{"bearing_type": "deep_groove_ball"}'
  AND (classification_attributes->>'bore_diameter_mm')::NUMERIC BETWEEN 20 AND 30;

-- Q3: Find work orders with specific failure mode
SELECT wo_number, description, status,
       failure_data->>'failure_mode_code' AS failure_mode,
       failure_data->>'root_cause_analysis' AS root_cause
FROM work_order
WHERE failure_data @> '{"iso_14224_failure_mode": "1.3.2"}';

-- Q4: Cross-reference parts across suppliers in the JSONB array
SELECT part_number, name,
       ref->>'supplier' AS supplier,
       (ref->>'price')::NUMERIC AS price
FROM catalogue_item,
     jsonb_array_elements(supplier_references) AS ref
WHERE status = 'active'
  AND (ref->>'price')::NUMERIC < 15.00
ORDER BY price;

-- Q5: Tenant-specific custom field query
SELECT part_number, name,
       custom_fields->>'hazmat_class' AS hazmat,
       custom_fields->>'shelf_life_months' AS shelf_life
FROM catalogue_item
WHERE org_id = 'tenant-uuid'
  AND custom_fields @> '{"hazmat_class": "flammable"}';
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Organisation & Custom Fields | 2 | organisation, field_definition |
| Site & Location | 2 | site, functional_location |
| Equipment & Assets | 2 | equipment_class, asset (JSONB-extended) |
| Catalogue & Classification | 1 | catalogue_item (JSONB replaces 4+ taxonomy tables) |
| Storeroom & Inventory | 4 | storeroom, bin_location, inventory_balance, inventory_transaction (partitioned) |
| Transfers | 1 | inventory_transfer (JSONB lines) |
| Work Orders & PM | 2 | work_order (JSONB parts/labour/failure), pm_schedule (JSONB frequency/parts) |
| Vendors & Procurement | 2 | vendor, purchase_order (JSONB lines) |
| AI & Forecasting | 3 | demand_forecast, part_criticality, supplier_risk_assessment |
| IoT & Sensors | 2 | sensor, sensor_reading (partitioned) |
| Users & Compliance | 3 | app_user, compliance_procedure, audit_log (partitioned) |
| **Total** | **~24** | Significantly fewer than normalised model |

---

## Key Design Decisions

1. **Relational for quantities and status, JSONB for attributes and details** — inventory balances, reorder points, and quantities are relational columns with CHECK constraints. Industry-specific attributes (flight hours, vibration baselines) live in JSONB. This boundary is drawn by asking: "Does the database need to enforce integrity on this field?" If yes, relational. If no, JSONB.

2. **Custom field definitions table** — the `field_definition` table lets each tenant define custom fields per entity type with validation rules, display names, and data types. This replaces the need for schema-per-tenant or column proliferation.

3. **Work order parts and labour in JSONB arrays** — rather than separate `work_order_part` and `work_order_labour` junction tables, parts and labour are stored as JSONB arrays on the work order itself. This makes the work order a self-contained document for read operations, reducing joins. The trade-off is that aggregate queries across work orders ("total parts consumed this month") require JSONB array unpacking.

4. **Purchase order lines in JSONB** — same rationale as work order parts. A PO is typically read as a complete document. For aggregate spend analytics, a materialised view can flatten JSONB lines into a columnar format.

5. **Transfer lines in JSONB** — transfers are simple, low-frequency operations. JSONB lines eliminate a junction table without meaningful cost.

6. **Sensor config in JSONB** — sensor configuration varies dramatically by sensor type (vibration, temperature, pressure, current). A JSONB config column avoids a separate table per sensor type.

7. **UNSPSC/eClass as flat codes, not hierarchy tables** — the normalised model uses 4-6 tables for taxonomy hierarchies. This model stores the UNSPSC code as a simple text column on `catalogue_item` and queries the hierarchy via application-level lookup or a reference data API. Simpler schema, slightly more application logic.

8. **Vendor performance metrics in JSONB** — vendor performance data (on-time rate, quality reject rate, risk score) is stored as a JSONB field updated periodically by the AI engine. This avoids a separate time-series table for vendor metrics while still supporting dashboard display.

9. **GIN indexes on all JSONB columns** — every JSONB column that supports queries has a GIN index with `jsonb_path_ops`. This enables efficient containment queries (`@>`) and path-based lookups.

10. **24 tables vs. 35+** — the JSONB approach reduces total table count by ~30% compared to the normalised model. Each eliminated table is a junction or lookup table whose data is embedded in a parent entity's JSONB column. The reduction simplifies migrations, ORM mapping, and API design at the cost of some query flexibility.
