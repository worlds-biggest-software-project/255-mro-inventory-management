# MRO Inventory Management

> Part of the [worlds-biggest-software-project](https://github.com/worlds-biggest-software-project) initiative.
>
> An AI-native, open-source MRO inventory platform that replaces static min/max thresholds with demand-sensing optimisation grounded in equipment condition data.

Maintenance, Repair and Operations (MRO) inventory management determines whether industrial assets stay running or sit idle waiting for parts. This project targets maintenance managers, reliability engineers, and MRO procurement teams who need accurate spare parts availability without the cost and complexity of enterprise EAM suites or the feature gaps of mid-market CMMS tools.

---

## Why MRO Inventory Management?

- Enterprise platforms with prescriptive AI (IBM Maximo, SAP PM, Infor EAM) use custom pricing that places them out of reach for mid-market operators and require specialist partners to implement.
- Mid-market CMMS tools (MaintainX, Limble, UpKeep, Fiix) gate advanced analytics, REST APIs, and inventory automation behind higher-tier plans, with Fiix's parts forecaster capped at just 25 parts.
- No open-source MRO inventory platform of comparable capability was identified in the landscape survey — the entire category is proprietary SaaS.
- Existing tools rely on static min/max reorder points and do not expose AI confidence intervals, leaving buyers unable to evaluate or trust recommendations.
- Cross-site inventory balancing, supplier lead-time risk scoring, and natural-language parts search across messy catalogue data are underserved across every incumbent reviewed.

---

## Key Features

### Inventory Tracking and Movement

- Real-time multi-location inventory tracking with low-stock alerts and reorder point management
- Parts linked to work orders and assets with full consumption audit trail
- Barcode scanning support for parts receiving and issue on mobile devices
- Vendor and supplier information management per SKU, with purchase order creation

### AI-Driven Demand and Forecasting

- AI demand forecasting using historical consumption data to replace static min/max thresholds
- IoT/sensor data ingestion to trigger predictive parts requirements based on equipment condition
- Criticality scoring per part tied to asset criticality and failure impact
- Supplier lead-time risk scoring surfaced as alerts before stockouts occur

### Catalogue Quality and Search

- Natural language parts search and catalogue management
- NLP-powered deduplication of near-duplicate SKUs across supplier feeds
- Cross-site inventory balancing with automated surplus transfer recommendations

### Knowledge and Compliance (Backlog)

- Technician voice/text observation capture with AI conversion to structured maintenance records
- AI confidence transparency — explanations and confidence intervals surfaced alongside recommendations
- Rotable spares lifecycle tracking (issue, repair, refurbish, restock)
- Industry-specific compliance workflow templates (aviation AOG, oil & gas LOTO, utilities regulatory)

---

## AI-Native Advantage

Incumbents either offer prescriptive AI only at enterprise pricing or expose AI as a fixed-capacity add-on. This project applies AI across the workflow: demand-sensing reorder points that combine equipment condition data with usage history, ML-driven catalogue deduplication across supplier feeds, predictive work order generation from IoT anomalies with pre-populated parts lists, supplier lead-time intelligence that flags at-risk parts before stockouts, and natural-language knowledge capture from technicians post-repair.

---

## Tech Stack & Deployment

The project targets an open-core or fully open-source distribution with self-hosted and cloud deployment modes. Integration scope includes IoT and sensor data ingestion (SCADA/PLC equivalents), barcode scanner and label printer support, ERP connectors for procurement workflows, and a REST API for third-party extension. Standards alignment includes ISO 55000/55001 for asset management, GS1/GTIN for product identification, UNSPSC/eClass for MRO taxonomy, and SAE JA1011 (Reliability-Centred Maintenance) for criticality-based stocking.

---

## Market Context

The global MRO market was valued at approximately USD 717 billion in 2026, projected to reach USD 882 billion by 2035 (Market Reports World, 2026). The AI-driven predictive maintenance software segment was valued at USD 1.18–17 billion in 2026 depending on scope, with 15–30% CAGR projections through 2030 (Research and Markets, 2026). Mid-market CMMS pricing ranges from ~$16–$70/user/month; enterprise platforms run into hundreds of thousands to millions annually. Primary buyers are maintenance managers and plant operations directors at manufacturing, energy, utilities, and facilities organisations, plus MRO procurement managers and reliability engineers.

The candidate is rated complexity 6/10, demand Medium, domain availability Low.

---

## Project Status

> This project is in the **research and specification phase**.  
> Contributions, feedback, and domain expertise are welcome.

---

## Contributing

We welcome contributions from developers, domain experts, and potential users.
See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

**Important:** All contributions must be your own original work or clearly attributed
open-source material with a compatible licence. Copyright infringement and licence
violations will not be tolerated and will result in immediate removal of the offending
contribution. If you are unsure whether a piece of code, text, or other material is
safe to contribute, open an issue and ask before submitting.

---

## Licence

Licence to be determined. See [discussion](#) for context.
