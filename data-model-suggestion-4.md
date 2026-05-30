# Data Model Suggestion 4: Graph-Relational Hybrid

> Project: MRO Inventory Management · Created: 2026-05-22

## Philosophy

This model combines a traditional relational layer for operational CRUD (inventory balances, purchase orders, work orders) with a property graph layer for relationship-heavy queries that relational databases struggle with. The graph layer uses PostgreSQL's `ltree` extension for hierarchical relationships and a generic `graph_node`/`graph_edge` pattern for arbitrary typed relationships between entities.

MRO inventory management is fundamentally about relationships: assets contain subassemblies, subassemblies use parts, parts have substitutes and supersessions, parts come from suppliers, suppliers have lead-time dependencies, sensors monitor assets, work orders consume parts from storerooms, failures cascade through dependency chains. Traditional relational models handle direct (one-hop) relationships well but become unwieldy for multi-hop traversals ("which parts are affected if supplier X fails?", "what is the full dependency chain for this critical asset?", "show me all assets that share a common failure-prone part").

The graph-relational hybrid is inspired by real-world implementations: LinkedIn uses a graph layer for social connections over relational data, Amazon uses graph databases for product recommendation engines, and in the industrial space, Siemens MindSphere uses graph-based digital twins to model asset hierarchies and dependency chains. For MRO specifically, a graph layer enables supply chain risk analysis (single-source part → all dependent assets), failure propagation analysis (failed component → all affected systems), and cross-site optimisation (find the nearest storeroom with available stock via graph traversal).

**Best for:** Organisations with complex asset hierarchies, multi-tier supply chains, substitute/supersession part networks, and requirements for supply chain risk analysis or failure propagation queries.

**Trade-offs:**
- Pro: Multi-hop relationship queries (risk propagation, dependency chains) are natural and performant
- Pro: Asset hierarchy traversal (plant → area → unit → component) is first-class
- Pro: Part substitute/supersession networks are explicit and queryable
- Pro: Supply chain vulnerability analysis ("if vendor X fails, which assets are at risk?") is a simple graph traversal
- Pro: Failure correlation discovery across shared components
- Con: Two conceptual models (relational + graph) increase developer cognitive load
- Con: Graph edge tables grow large for densely connected domains
- Con: Transactional consistency across relational and graph layers requires careful handling
- Con: Graph queries (recursive CTEs, ltree operations) are less familiar to most SQL developers
- Con: BI/reporting tools have limited native support for graph query patterns

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| ISO 55001:2024 | Asset lifecycle modelled as relational entities; asset-to-asset relationships (parent/child, depends-on, protects) as graph edges |
| ISO 14224:2016 | Equipment taxonomy modelled as ltree path hierarchy; failure propagation via graph edges between equipment classes |
| UNSPSC / eCl@ss | Classification codes on part nodes; graph edges link parts to their taxonomy position |
| GS1/GTIN | GTIN stored on part nodes; supply chain graph connects GTIN → manufacturer → distributor → storeroom |
| SAE JA1011 (RCM) | Criticality analysis uses graph traversal to identify all assets affected by a critical part's unavailability |
| OSHA 29 CFR 1910.147 | LOTO procedures linked via graph edges to assets and the specific isolation points (modelled as nodes) |
| ISO 3166 | Jurisdiction hierarchy modelled as ltree for multi-region site management |

---

## Graph Infrastructure

```sql
-- ============================================================
-- GRAPH LAYER — Generic property graph on PostgreSQL
-- ============================================================

CREATE EXTENSION IF NOT EXISTS ltree;

-- Graph node: represents any entity in the relationship graph
CREATE TABLE graph_node (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL,
    node_type       TEXT NOT NULL,
    -- Node types:
    -- 'asset', 'part', 'storeroom', 'vendor', 'site', 'sensor',
    -- 'work_order', 'equipment_class', 'functional_location',
    -- 'purchase_order', 'failure_mode', 'compliance_procedure'
    entity_id       UUID NOT NULL,     -- FK to the relational entity table
    label           TEXT NOT NULL,      -- human-readable label
    properties      JSONB NOT NULL DEFAULT '{}',
    -- Cached key properties for fast graph-only queries:
    -- { "part_number": "BRG-6205-2Z", "criticality": "critical", "status": "active" }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(org_id, node_type, entity_id)
);

CREATE INDEX idx_gnode_type ON graph_node(org_id, node_type);
CREATE INDEX idx_gnode_entity ON graph_node(entity_id);
CREATE INDEX idx_gnode_props ON graph_node USING gin (properties jsonb_path_ops);

-- Graph edge: represents a typed, directed relationship between two nodes
CREATE TABLE graph_edge (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL,
    source_node_id  UUID NOT NULL REFERENCES graph_node(id) ON DELETE CASCADE,
    target_node_id  UUID NOT NULL REFERENCES graph_node(id) ON DELETE CASCADE,
    edge_type       TEXT NOT NULL,
    -- Edge types:
    -- Asset hierarchy:    'contains', 'parent_of', 'depends_on', 'protects'
    -- Part relationships: 'substitutes', 'supersedes', 'compatible_with', 'used_in'
    -- Supply chain:       'supplied_by', 'manufactured_by', 'stored_in'
    -- Maintenance:        'requires_part', 'monitored_by', 'failed_with'
    -- Compliance:         'governed_by', 'isolated_by'
    -- Forecasting:        'predicted_demand_for', 'risk_assessed'
    properties      JSONB NOT NULL DEFAULT '{}',
    -- Edge-specific metadata:
    -- For 'substitutes': { "fit_confidence": 0.95, "notes": "Same OD/ID, different seal" }
    -- For 'supplied_by': { "lead_time_days": 14, "unit_price": 12.50, "is_preferred": true }
    -- For 'depends_on':  { "dependency_type": "critical", "failure_propagation": true }
    weight          NUMERIC(8, 4) DEFAULT 1.0,
    -- Traversal weight for shortest-path and risk propagation algorithms
    is_active       BOOLEAN NOT NULL DEFAULT true,
    valid_from      TIMESTAMPTZ DEFAULT now(),
    valid_to        TIMESTAMPTZ,       -- null = currently valid
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_gedge_source ON graph_edge(source_node_id);
CREATE INDEX idx_gedge_target ON graph_edge(target_node_id);
CREATE INDEX idx_gedge_type ON graph_edge(org_id, edge_type);
CREATE INDEX idx_gedge_props ON graph_edge USING gin (properties jsonb_path_ops);
CREATE INDEX idx_gedge_active ON graph_edge(source_node_id, edge_type) WHERE is_active = true;

-- Edge type registry
CREATE TABLE edge_type_registry (
    edge_type       TEXT PRIMARY KEY,
    source_node_type TEXT NOT NULL,
    target_node_type TEXT NOT NULL,
    description     TEXT NOT NULL,
    is_bidirectional BOOLEAN NOT NULL DEFAULT false,
    inverse_type    TEXT,              -- e.g. 'supplied_by' inverse is 'supplies'
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

INSERT INTO edge_type_registry (edge_type, source_node_type, target_node_type, description, is_bidirectional, inverse_type) VALUES
    ('contains',         'asset',     'asset',     'Parent asset contains child asset/component', false, 'contained_in'),
    ('depends_on',       'asset',     'asset',     'Asset depends on another asset for operation', false, 'depended_on_by'),
    ('protects',         'asset',     'asset',     'Safety/protection relationship between assets', false, 'protected_by'),
    ('used_in',          'part',      'asset',     'Part is used in asset (BOM relationship)', false, 'uses_part'),
    ('substitutes',      'part',      'part',      'Part can substitute for another', true, 'substitutes'),
    ('supersedes',       'part',      'part',      'Newer part supersedes older part', false, 'superseded_by'),
    ('compatible_with',  'part',      'part',      'Parts are cross-compatible', true, 'compatible_with'),
    ('supplied_by',      'part',      'vendor',    'Part is supplied by vendor', false, 'supplies'),
    ('manufactured_by',  'part',      'vendor',    'Part is manufactured by vendor', false, 'manufactures'),
    ('stored_in',        'part',      'storeroom', 'Part has inventory in storeroom', false, 'stores'),
    ('monitored_by',     'asset',     'sensor',    'Asset is monitored by IoT sensor', false, 'monitors'),
    ('failed_with',      'asset',     'part',      'Asset failure involved this part', false, null),
    ('requires_part',    'work_order','part',       'Work order requires this part', false, null),
    ('governed_by',      'asset',     'compliance_procedure', 'Asset governed by compliance procedure', false, 'governs'),
    ('isolated_by',      'asset',     'asset',     'LOTO isolation point relationship', false, 'isolates');
```

---

## Relational Layer — Assets & Equipment

```sql
-- ============================================================
-- ORGANISATION & SITE
-- ============================================================

CREATE TABLE organisation (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            TEXT NOT NULL,
    slug            TEXT NOT NULL UNIQUE,
    settings        JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE site (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisation(id),
    code            TEXT NOT NULL,
    name            TEXT NOT NULL,
    country_code    CHAR(2) NOT NULL,
    timezone        TEXT NOT NULL DEFAULT 'UTC',
    address         JSONB NOT NULL DEFAULT '{}',
    -- ltree path for geographic hierarchy
    geo_path        ltree,
    -- e.g. 'us.tx.houston.industrial_park_north'
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(org_id, code)
);

CREATE INDEX idx_site_geo ON site USING gist (geo_path);

-- ============================================================
-- EQUIPMENT TAXONOMY (ltree hierarchy)
-- ============================================================

CREATE TABLE equipment_class (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisation(id),
    code            TEXT NOT NULL,
    name            TEXT NOT NULL,
    -- ltree path encodes the ISO 14224 hierarchy
    taxonomy_path   ltree NOT NULL,
    -- e.g. 'oil_gas.upstream.rotating.pump.centrifugal'
    -- Level 1: Industry
    -- Level 2: Business category
    -- Level 3: Equipment category
    -- Level 4: Equipment class
    -- Level 5: Equipment type
    iso_14224_ref   TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(org_id, code)
);

CREATE INDEX idx_eqclass_path ON equipment_class USING gist (taxonomy_path);

-- ============================================================
-- FUNCTIONAL LOCATION (ltree hierarchy)
-- ============================================================

CREATE TABLE functional_location (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    site_id         UUID NOT NULL REFERENCES site(id),
    code            TEXT NOT NULL,
    name            TEXT NOT NULL,
    -- ltree path for functional hierarchy
    location_path   ltree NOT NULL,
    -- e.g. 'plant_a.area_3.production_line_2.station_7'
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(site_id, code)
);

CREATE INDEX idx_floc_path ON functional_location USING gist (location_path);

-- ============================================================
-- ASSET
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
    -- ltree path encodes full asset hierarchy position
    asset_path          ltree,
    -- e.g. 'plant_a.compressor_train_1.gearbox.input_shaft'
    extended_attributes JSONB NOT NULL DEFAULT '{}',
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(org_id, asset_number)
);

CREATE INDEX idx_asset_site ON asset(site_id);
CREATE INDEX idx_asset_status ON asset(org_id, status);
CREATE INDEX idx_asset_path ON asset USING gist (asset_path);
CREATE INDEX idx_asset_ext ON asset USING gin (extended_attributes jsonb_path_ops);
```

---

## Relational Layer — Catalogue & Inventory

```sql
-- ============================================================
-- CATALOGUE ITEM
-- ============================================================

CREATE TABLE catalogue_item (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id              UUID NOT NULL REFERENCES organisation(id),
    part_number         TEXT NOT NULL,
    name                TEXT NOT NULL,
    description         TEXT,
    unspsc_code         TEXT,
    eclass_code         TEXT,
    gtin                TEXT,
    manufacturer        TEXT,
    manufacturer_part_number TEXT,
    unit_of_measure     TEXT NOT NULL DEFAULT 'EA',
    unit_cost           NUMERIC(14, 4),
    currency_code       CHAR(3) DEFAULT 'USD',
    is_rotable          BOOLEAN NOT NULL DEFAULT false,
    is_serialised       BOOLEAN NOT NULL DEFAULT false,
    criticality         TEXT CHECK (criticality IN ('critical', 'essential',
                                                    'desirable', 'run_to_failure')),
    status              TEXT NOT NULL DEFAULT 'active'
                        CHECK (status IN ('active', 'obsolete', 'pending_review')),
    specifications      JSONB NOT NULL DEFAULT '{}',
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(org_id, part_number)
);

CREATE INDEX idx_cat_item_gtin ON catalogue_item(gtin) WHERE gtin IS NOT NULL;
CREATE INDEX idx_cat_item_mfr ON catalogue_item(org_id, manufacturer);
CREATE INDEX idx_cat_item_status ON catalogue_item(org_id, status);
CREATE INDEX idx_cat_item_specs ON catalogue_item USING gin (specifications jsonb_path_ops);
CREATE INDEX idx_cat_item_fts ON catalogue_item USING gin (
    to_tsvector('english', name || ' ' || COALESCE(description, ''))
);

-- ============================================================
-- STOREROOM
-- ============================================================

CREATE TABLE storeroom (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    site_id         UUID NOT NULL REFERENCES site(id),
    code            TEXT NOT NULL,
    name            TEXT NOT NULL,
    storeroom_type  TEXT NOT NULL DEFAULT 'central'
                    CHECK (storeroom_type IN ('central', 'satellite', 'vehicle',
                                              'consignment')),
    is_active       BOOLEAN NOT NULL DEFAULT true,
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

-- ============================================================
-- INVENTORY BALANCE
-- ============================================================

CREATE TABLE inventory_balance (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    catalogue_item_id   UUID NOT NULL REFERENCES catalogue_item(id),
    storeroom_id        UUID NOT NULL REFERENCES storeroom(id),
    bin_location_id     UUID REFERENCES bin_location(id),
    quantity_on_hand    NUMERIC(12, 4) NOT NULL DEFAULT 0 CHECK (quantity_on_hand >= 0),
    quantity_reserved   NUMERIC(12, 4) NOT NULL DEFAULT 0 CHECK (quantity_reserved >= 0),
    quantity_on_order   NUMERIC(12, 4) NOT NULL DEFAULT 0 CHECK (quantity_on_order >= 0),
    reorder_point       NUMERIC(12, 4),
    reorder_quantity    NUMERIC(12, 4),
    max_level           NUMERIC(12, 4),
    safety_stock        NUMERIC(12, 4),
    average_unit_cost   NUMERIC(14, 4),
    last_issue_at       TIMESTAMPTZ,
    last_receipt_at     TIMESTAMPTZ,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(catalogue_item_id, storeroom_id)
);

CREATE INDEX idx_invbal_storeroom ON inventory_balance(storeroom_id);
CREATE INDEX idx_invbal_low ON inventory_balance(catalogue_item_id)
    WHERE quantity_on_hand <= reorder_point AND reorder_point IS NOT NULL;

-- ============================================================
-- INVENTORY TRANSACTION
-- ============================================================

CREATE TABLE inventory_transaction (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id              UUID NOT NULL,
    inventory_balance_id UUID NOT NULL REFERENCES inventory_balance(id),
    transaction_type    TEXT NOT NULL
                        CHECK (transaction_type IN (
                            'receipt', 'issue', 'return', 'adjustment',
                            'transfer_out', 'transfer_in', 'cycle_count',
                            'scrap', 'reservation', 'unreservation'
                        )),
    quantity            NUMERIC(12, 4) NOT NULL,
    unit_cost           NUMERIC(14, 4),
    reference_type      TEXT,
    reference_id        UUID,
    lot_number          TEXT,
    serial_number       TEXT,
    performed_by        UUID NOT NULL,
    notes               TEXT,
    transacted_at       TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (transacted_at);

CREATE INDEX idx_invtxn_balance ON inventory_transaction(inventory_balance_id);
CREATE INDEX idx_invtxn_type ON inventory_transaction(org_id, transaction_type);
CREATE INDEX idx_invtxn_date ON inventory_transaction(transacted_at);
```

---

## Relational Layer — Work Orders & Procurement

```sql
-- ============================================================
-- VENDOR
-- ============================================================

CREATE TABLE vendor (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisation(id),
    code            TEXT NOT NULL,
    name            TEXT NOT NULL,
    contact_name    TEXT,
    contact_email   TEXT,
    country_code    CHAR(2),
    payment_terms   TEXT,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(org_id, code)
);

-- ============================================================
-- VENDOR CATALOGUE ITEM (relational for pricing; graph for relationships)
-- ============================================================

CREATE TABLE vendor_catalogue_item (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    vendor_id           UUID NOT NULL REFERENCES vendor(id),
    catalogue_item_id   UUID NOT NULL REFERENCES catalogue_item(id),
    vendor_part_number  TEXT,
    unit_price          NUMERIC(14, 4) NOT NULL,
    currency_code       CHAR(3) DEFAULT 'USD',
    lead_time_days      INTEGER,
    min_order_qty       NUMERIC(12, 4) DEFAULT 1,
    is_preferred        BOOLEAN NOT NULL DEFAULT false,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(vendor_id, catalogue_item_id)
);

CREATE INDEX idx_vci_item ON vendor_catalogue_item(catalogue_item_id);

-- ============================================================
-- WORK ORDER
-- ============================================================

CREATE TABLE work_order (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id              UUID NOT NULL REFERENCES organisation(id),
    site_id             UUID NOT NULL REFERENCES site(id),
    wo_number           TEXT NOT NULL,
    asset_id            UUID REFERENCES asset(id),
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
    failure_mode        TEXT,
    failure_cause       TEXT,
    assigned_to         UUID,
    scheduled_start     TIMESTAMPTZ,
    scheduled_end       TIMESTAMPTZ,
    actual_start        TIMESTAMPTZ,
    actual_end          TIMESTAMPTZ,
    downtime_minutes    INTEGER,
    source              TEXT DEFAULT 'manual'
                        CHECK (source IN ('manual', 'pm_schedule', 'iot_alert',
                                          'ai_predicted', 'inspection')),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(org_id, wo_number)
);

CREATE INDEX idx_wo_asset ON work_order(asset_id);
CREATE INDEX idx_wo_status ON work_order(org_id, status);

-- ============================================================
-- WORK ORDER PARTS
-- ============================================================

CREATE TABLE work_order_part (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    work_order_id       UUID NOT NULL REFERENCES work_order(id) ON DELETE CASCADE,
    catalogue_item_id   UUID NOT NULL REFERENCES catalogue_item(id),
    storeroom_id        UUID NOT NULL REFERENCES storeroom(id),
    quantity_planned    NUMERIC(12, 4) NOT NULL,
    quantity_actual     NUMERIC(12, 4),
    unit_cost           NUMERIC(14, 4),
    serial_number       TEXT,
    issued_at           TIMESTAMPTZ,
    issued_by           UUID
);

CREATE INDEX idx_wopart_wo ON work_order_part(work_order_id);
CREATE INDEX idx_wopart_item ON work_order_part(catalogue_item_id);

-- ============================================================
-- PURCHASE ORDER
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
    total_amount        NUMERIC(14, 4),
    currency_code       CHAR(3) DEFAULT 'USD',
    requested_by        UUID NOT NULL,
    approved_by         UUID,
    notes               TEXT,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(org_id, po_number)
);

CREATE TABLE purchase_order_line (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    purchase_order_id   UUID NOT NULL REFERENCES purchase_order(id) ON DELETE CASCADE,
    catalogue_item_id   UUID NOT NULL REFERENCES catalogue_item(id),
    storeroom_id        UUID NOT NULL REFERENCES storeroom(id),
    quantity_ordered    NUMERIC(12, 4) NOT NULL,
    quantity_received   NUMERIC(12, 4) NOT NULL DEFAULT 0,
    unit_price          NUMERIC(14, 4) NOT NULL,
    expected_delivery   DATE
);

CREATE INDEX idx_poline_po ON purchase_order_line(purchase_order_id);
CREATE INDEX idx_poline_item ON purchase_order_line(catalogue_item_id);
```

---

## AI, Forecasting & Sensors

```sql
-- ============================================================
-- AI DEMAND FORECAST
-- ============================================================

CREATE TABLE demand_forecast (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id              UUID NOT NULL,
    catalogue_item_id   UUID NOT NULL REFERENCES catalogue_item(id),
    storeroom_id        UUID NOT NULL REFERENCES storeroom(id),
    forecast_date       DATE NOT NULL,
    predicted_demand    NUMERIC(12, 4) NOT NULL,
    confidence_lower    NUMERIC(12, 4),
    confidence_upper    NUMERIC(12, 4),
    confidence_level    NUMERIC(3, 2) DEFAULT 0.90,
    recommended_rop     NUMERIC(12, 4),
    recommended_roq     NUMERIC(12, 4),
    model_details       JSONB NOT NULL DEFAULT '{}',
    status              TEXT NOT NULL DEFAULT 'pending'
                        CHECK (status IN ('pending', 'accepted', 'overridden', 'expired')),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_forecast_item ON demand_forecast(catalogue_item_id, storeroom_id, forecast_date DESC);

-- ============================================================
-- SUPPLIER LEAD TIME HISTORY
-- ============================================================

CREATE TABLE supplier_lead_time (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    vendor_catalogue_item_id UUID NOT NULL REFERENCES vendor_catalogue_item(id),
    purchase_order_line_id UUID REFERENCES purchase_order_line(id),
    promised_days       INTEGER NOT NULL,
    actual_days         INTEGER,
    variance_days       INTEGER GENERATED ALWAYS AS (actual_days - promised_days) STORED,
    delivered_at        TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_slt_vci ON supplier_lead_time(vendor_catalogue_item_id);

-- ============================================================
-- IoT SENSOR
-- ============================================================

CREATE TABLE sensor (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    asset_id        UUID NOT NULL REFERENCES asset(id),
    sensor_type     TEXT NOT NULL,
    unit_of_measure TEXT NOT NULL,
    config          JSONB NOT NULL DEFAULT '{}',
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_sensor_asset ON sensor(asset_id);

CREATE TABLE sensor_reading (
    sensor_id       UUID NOT NULL REFERENCES sensor(id),
    reading_at      TIMESTAMPTZ NOT NULL,
    value           DOUBLE PRECISION NOT NULL,
    is_anomaly      BOOLEAN NOT NULL DEFAULT false
) PARTITION BY RANGE (reading_at);

CREATE INDEX idx_sreading_sensor ON sensor_reading(sensor_id, reading_at DESC);
```

---

## Users & Audit

```sql
-- ============================================================
-- USERS
-- ============================================================

CREATE TABLE app_user (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisation(id),
    email           TEXT NOT NULL,
    display_name    TEXT NOT NULL,
    role            TEXT NOT NULL DEFAULT 'technician',
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(org_id, email)
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
    old_values      JSONB,
    new_values      JSONB,
    performed_at    TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (performed_at);

CREATE INDEX idx_audit_entity ON audit_log(entity_type, entity_id);
CREATE INDEX idx_audit_user ON audit_log(user_id);
CREATE INDEX idx_audit_date ON audit_log(performed_at);
```

---

## Graph Query Examples

```sql
-- ============================================================
-- GRAPH TRAVERSAL QUERIES
-- ============================================================

-- Q1: SUPPLY CHAIN RISK — If vendor X fails, which assets are at risk?
-- Traversal: vendor → (supplied_by) → parts → (used_in) → assets
WITH vendor_parts AS (
    SELECT e.target_node_id AS vendor_node, e.source_node_id AS part_node,
           pn.entity_id AS part_id, pn.label AS part_name,
           e.properties->>'is_preferred' AS is_preferred
    FROM graph_edge e
    JOIN graph_node pn ON pn.id = e.source_node_id
    JOIN graph_node vn ON vn.id = e.target_node_id
    WHERE e.edge_type = 'supplied_by'
      AND vn.entity_id = 'vendor-uuid-here'  -- the failing vendor
      AND e.is_active = true
),
affected_assets AS (
    SELECT DISTINCT
           an.entity_id AS asset_id,
           an.label AS asset_name,
           an.properties->>'criticality' AS asset_criticality,
           vp.part_name,
           vp.is_preferred
    FROM vendor_parts vp
    JOIN graph_edge e2 ON e2.source_node_id = vp.part_node
    JOIN graph_node an ON an.id = e2.target_node_id
    WHERE e2.edge_type = 'used_in'
      AND e2.is_active = true
)
SELECT * FROM affected_assets
ORDER BY
    CASE asset_criticality
        WHEN 'critical' THEN 1
        WHEN 'essential' THEN 2
        WHEN 'desirable' THEN 3
        ELSE 4
    END;

-- Q2: PART SUBSTITUTE NETWORK — Find all substitutes for a part (multi-hop)
-- Traversal: part → (substitutes/compatible_with) → parts → recurse
WITH RECURSIVE part_network AS (
    -- Start with the target part
    SELECT
        gn.id AS node_id,
        gn.entity_id AS part_id,
        gn.label AS part_name,
        0 AS depth,
        ARRAY[gn.id] AS path
    FROM graph_node gn
    WHERE gn.entity_id = 'target-part-uuid'
      AND gn.node_type = 'part'

    UNION ALL

    -- Traverse substitute and compatible edges
    SELECT
        gn2.id,
        gn2.entity_id,
        gn2.label,
        pn.depth + 1,
        pn.path || gn2.id
    FROM part_network pn
    JOIN graph_edge ge ON (ge.source_node_id = pn.node_id OR ge.target_node_id = pn.node_id)
    JOIN graph_node gn2 ON gn2.id = CASE
        WHEN ge.source_node_id = pn.node_id THEN ge.target_node_id
        ELSE ge.source_node_id
    END
    WHERE ge.edge_type IN ('substitutes', 'compatible_with', 'supersedes')
      AND ge.is_active = true
      AND gn2.id <> ALL(pn.path)  -- prevent cycles
      AND pn.depth < 3            -- limit traversal depth
)
SELECT part_id, part_name, depth
FROM part_network
WHERE depth > 0
ORDER BY depth;

-- Q3: ASSET HIERARCHY — Find all components under a top-level asset using ltree
SELECT a.asset_number, a.name, a.criticality, a.status,
       nlevel(a.asset_path) AS hierarchy_depth
FROM asset a
WHERE a.asset_path <@ 'plant_a.compressor_train_1'  -- all descendants
ORDER BY a.asset_path;

-- Q4: EQUIPMENT CLASS SIBLINGS — Find all equipment in the same ISO 14224 class
SELECT a.asset_number, a.name, a.site_id
FROM asset a
JOIN equipment_class ec ON ec.id = a.equipment_class_id
WHERE ec.taxonomy_path ~ 'oil_gas.upstream.rotating.pump.*'
  AND a.status = 'active';

-- Q5: FAILURE PROPAGATION — What parts commonly fail together?
-- (Parts connected to the same asset via 'failed_with' edges)
SELECT
    p1.label AS part_a,
    p2.label AS part_b,
    COUNT(*) AS co_failure_count
FROM graph_edge e1
JOIN graph_edge e2 ON e1.source_node_id = e2.source_node_id  -- same asset
    AND e1.target_node_id < e2.target_node_id                 -- avoid duplicates
JOIN graph_node p1 ON p1.id = e1.target_node_id
JOIN graph_node p2 ON p2.id = e2.target_node_id
WHERE e1.edge_type = 'failed_with'
  AND e2.edge_type = 'failed_with'
GROUP BY p1.label, p2.label
HAVING COUNT(*) >= 3
ORDER BY co_failure_count DESC;

-- Q6: CROSS-SITE STOCK FINDER — Find nearest storeroom with available stock
-- Combines graph traversal with relational inventory data
WITH part_storerooms AS (
    SELECT
        sn.entity_id AS storeroom_id,
        sn.label AS storeroom_name,
        ge.properties->>'site_code' AS site_code
    FROM graph_node pn
    JOIN graph_edge ge ON ge.source_node_id = pn.id
    JOIN graph_node sn ON sn.id = ge.target_node_id
    WHERE pn.entity_id = 'target-part-uuid'
      AND ge.edge_type = 'stored_in'
      AND ge.is_active = true
)
SELECT
    ps.storeroom_name,
    ps.site_code,
    ib.quantity_on_hand,
    ib.quantity_on_hand - ib.quantity_reserved AS available
FROM part_storerooms ps
JOIN inventory_balance ib ON ib.storeroom_id = ps.storeroom_id
    AND ib.catalogue_item_id = 'target-part-uuid'
WHERE ib.quantity_on_hand - ib.quantity_reserved > 0
ORDER BY available DESC;

-- Q7: GEOGRAPHIC HIERARCHY QUERY — All sites in Texas using ltree
SELECT id, code, name
FROM site
WHERE geo_path <@ 'us.tx'
  AND is_active = true;
```

---

## Graph Synchronisation

```sql
-- ============================================================
-- TRIGGER: Sync relational entities to graph nodes
-- ============================================================

-- When an asset is inserted, automatically create a graph node
CREATE OR REPLACE FUNCTION sync_asset_to_graph()
RETURNS TRIGGER AS $$
BEGIN
    INSERT INTO graph_node (org_id, node_type, entity_id, label, properties)
    VALUES (
        NEW.org_id,
        'asset',
        NEW.id,
        NEW.asset_number || ': ' || NEW.name,
        jsonb_build_object(
            'asset_number', NEW.asset_number,
            'status', NEW.status,
            'criticality', NEW.criticality,
            'site_id', NEW.site_id
        )
    )
    ON CONFLICT (org_id, node_type, entity_id) DO UPDATE
    SET label = EXCLUDED.label,
        properties = EXCLUDED.properties;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_asset_graph_sync
    AFTER INSERT OR UPDATE ON asset
    FOR EACH ROW EXECUTE FUNCTION sync_asset_to_graph();

-- Similar triggers for: catalogue_item, vendor, storeroom, sensor, work_order

-- When a vendor_catalogue_item is inserted, create a 'supplied_by' edge
CREATE OR REPLACE FUNCTION sync_vendor_part_to_graph()
RETURNS TRIGGER AS $$
DECLARE
    part_node_id UUID;
    vendor_node_id UUID;
BEGIN
    SELECT id INTO part_node_id FROM graph_node
    WHERE node_type = 'part' AND entity_id = NEW.catalogue_item_id LIMIT 1;

    SELECT id INTO vendor_node_id FROM graph_node
    WHERE node_type = 'vendor' AND entity_id = NEW.vendor_id LIMIT 1;

    IF part_node_id IS NOT NULL AND vendor_node_id IS NOT NULL THEN
        INSERT INTO graph_edge (org_id, source_node_id, target_node_id, edge_type, properties, weight)
        VALUES (
            (SELECT org_id FROM vendor WHERE id = NEW.vendor_id),
            part_node_id,
            vendor_node_id,
            'supplied_by',
            jsonb_build_object(
                'unit_price', NEW.unit_price,
                'lead_time_days', NEW.lead_time_days,
                'is_preferred', NEW.is_preferred,
                'vendor_part_number', NEW.vendor_part_number
            ),
            CASE WHEN NEW.is_preferred THEN 0.5 ELSE 1.0 END
        )
        ON CONFLICT DO NOTHING;
    END IF;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_vendor_part_graph_sync
    AFTER INSERT OR UPDATE ON vendor_catalogue_item
    FOR EACH ROW EXECUTE FUNCTION sync_vendor_part_to_graph();
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Graph Infrastructure | 3 | graph_node, graph_edge, edge_type_registry |
| Organisation & Site | 2 | organisation, site (with ltree geo_path) |
| Equipment & Location | 2 | equipment_class (ltree), functional_location (ltree) |
| Assets | 1 | asset (with ltree asset_path) |
| Catalogue & Inventory | 5 | catalogue_item, storeroom, bin_location, inventory_balance, inventory_transaction (partitioned) |
| Vendors & Procurement | 5 | vendor, vendor_catalogue_item, purchase_order, purchase_order_line |
| Work Orders | 2 | work_order, work_order_part |
| AI & Forecasting | 2 | demand_forecast, supplier_lead_time |
| IoT & Sensors | 2 | sensor, sensor_reading (partitioned) |
| Users & Audit | 2 | app_user, audit_log (partitioned) |
| **Total** | **~26** | Plus graph_node and graph_edge which mirror relational entities |

---

## Key Design Decisions

1. **Generic graph overlay, not a replacement** — the `graph_node`/`graph_edge` tables mirror entities from the relational layer rather than replacing them. Operational CRUD (update a quantity, create a PO) happens against relational tables. Relationship-heavy queries (risk propagation, substitute networks, hierarchy traversals) use the graph layer. This avoids the "graph-only" pitfall where simple queries become unnecessarily complex.

2. **ltree for bounded hierarchies** — equipment taxonomy (ISO 14224, max 9 levels), functional locations, asset hierarchies, and geographic site hierarchies use PostgreSQL's `ltree` extension. ltree is ideal for bounded, relatively static hierarchies: it supports ancestor/descendant queries, pattern matching, and sorting by hierarchy position natively. The alternative (recursive CTEs on adjacency lists) works but is slower for deep trees.

3. **Graph edges for unbounded/dynamic relationships** — part substitute networks, supply chain relationships, failure correlations, and LOTO isolation points use the generic `graph_edge` table. These relationships are dynamic (substitutes change, vendors are added/removed), potentially cyclic (A substitutes B, B substitutes C, C compatible with A), and multi-hop (risk propagation traverses vendor → part → asset → downstream asset).

4. **Edge type registry** — the `edge_type_registry` table documents valid edge types, their source/target node types, and whether they are bidirectional. This prevents arbitrary edge creation and enables the application layer to validate graph mutations.

5. **Temporal edges** — `valid_from`/`valid_to` timestamps on edges enable point-in-time graph queries ("who supplied this part on March 15th?") and soft-deletion of relationships without losing history.

6. **Weight on edges** — the `weight` column enables weighted graph algorithms: shortest path for supply chain analysis (preferred vendors = lower weight), risk propagation scoring, and cross-site transfer cost optimisation.

7. **Trigger-based synchronisation** — PostgreSQL triggers keep the graph layer in sync with relational inserts/updates. This ensures consistency without requiring the application to maintain two models manually. The triggers are idempotent (ON CONFLICT DO NOTHING/DO UPDATE) to handle re-runs safely.

8. **Properties cache on graph nodes** — the `properties` JSONB column on `graph_node` caches frequently-needed attributes (criticality, status, part_number) so that graph-only queries can filter without joining back to relational tables. This is a deliberate denormalization for traversal performance.

9. **Depth-limited recursive CTEs** — all recursive graph traversal queries include a depth limit (typically 3-5 hops) to prevent runaway queries on densely connected graphs. The `path` array prevents cycles.

10. **Graph layer is optional** — the relational layer is fully functional without the graph tables. An organisation that does not need risk propagation or substitute network analysis can ignore the graph entirely. The triggers can be disabled, and the `graph_node`/`graph_edge` tables can be empty without affecting core inventory operations.
