# Standards & API Reference

> Project: MRO Inventory Management · Generated: 2026-05-03

---

## Industry Standards & Specifications

### ISO Standards

**ISO 55000:2024 — Asset Management: Vocabulary, Overview and Principles**
- URL: https://www.iso.org/standard/83053.html
- The foundational vocabulary and principles standard for asset management systems. Defines concepts including asset, asset management system, and asset management objective. Provides the framework within which MRO inventory is treated as a supporting activity for physical asset lifecycle management. The 2024 edition revised principles and added guidance on data and people management.

**ISO 55001:2024 — Asset Management: Asset Management System Requirements**
- URL: https://www.iso.org/standard/83054.html
- Specifies requirements for establishing, implementing, maintaining, and improving an asset management system. An MRO inventory system used to support ISO 55001-compliant asset management must demonstrate traceability of spare parts to asset maintenance activities, appropriate inventory criticality assessment, and lifecycle cost accountability.

**ISO 14224:2016 — Collection and Exchange of Reliability and Maintenance Data for Equipment**
- URL: https://www.iso.org/standard/64076.html
- Defines a minimum data set and standardised format for collecting reliability and maintenance (RM) data in petroleum, petrochemical, and natural gas industries. Relevant to MRO systems as it specifies equipment taxonomy, failure mode classification, and data exchange formats for maintenance history and spare parts consumption records. Widely used beyond its original petroleum scope.

### W3C & IETF Standards

**RFC 9110 — HTTP Semantics**
- URL: https://www.rfc-editor.org/rfc/rfc9110
- Defines the semantics of HTTP/1.1 and HTTP/2, including methods (GET, POST, PUT, DELETE, PATCH), status codes, and content negotiation. Foundational to all REST API implementations in MRO CMMS platforms.

**RFC 8288 — Web Linking**
- URL: https://www.rfc-editor.org/rfc/rfc8288
- Defines the Link header field and link relations used in RESTful hypermedia APIs. Relevant when designing inventory resource APIs with pagination and resource discovery.

**RFC 7636 — Proof Key for Code Exchange (PKCE) by OAuth Public Clients**
- URL: https://www.rfc-editor.org/rfc/rfc7636
- Extends OAuth 2.0 authorisation code flow to prevent token interception in public clients (mobile apps, SPAs). The 2026 recommended authentication flow for all CMMS client types (web, mobile, desktop) is OAuth 2.0 Authorisation Code with PKCE.

**RFC 6749 — The OAuth 2.0 Authorization Framework**
- URL: https://www.rfc-editor.org/rfc/rfc6749
- Defines the OAuth 2.0 framework for delegated access. All reviewed MRO CMMS platforms use OAuth 2.0 for API authentication. Infor EAM routes API access through an ION API Gateway using OAuth 2.0; IBM Maximo, UpKeep, and others use Bearer Token or OAuth flows.

### Data Model & API Specifications

**OpenAPI Specification 3.1 (formerly Swagger)**
- URL: https://spec.openapis.org/oas/v3.1.0
- The industry standard for describing RESTful APIs in machine-readable YAML or JSON. UpKeep publishes an OpenAPI/Swagger spec as part of its developer portal. Fiix and Limble also provide Postman collections aligned with OpenAPI conventions. Any AI-native MRO platform should publish an OpenAPI 3.1 spec to enable LLM tool-use integration.

**UNSPSC — United Nations Standard Products and Services Code**
- URL: https://www.unspsc.org
- A classification taxonomy with over 1,500,000 codes jointly developed by the UN Development Programme. Used by asset-intensive industries (manufacturing, utilities, mining, oil & gas) as the primary MRO catalogue taxonomy for spend categorisation and analytics. Data deduplication tools such as Verdantis Harmonize classify all spare parts to UNSPSC. Belgium and France (2026) now require structured e-invoices via Peppol referencing UNSPSC codes.

**eCl@ss (eClass) Standard**
- URL: https://www.eclass.eu
- An international standard maintained by the eCl@ss e.V. industry association (Germany) for classification and description of products, materials, and services. Contains approximately 41,000 product classes and 17,000 properties, covering the majority of traded industrial goods. Commonly used in European manufacturing and engineering as an alternative or complement to UNSPSC. Cross-mapping between UNSPSC and eCl@ss is a frequent requirement for MRO data governance projects.

**GS1 GTIN / Application Identifier Standards**
- URL: https://www.gs1.org/standards
- GS1 defines barcode symbologies (EAN, UPC, ITF-14, GS1-128, DataMatrix) and the Global Trade Item Number (GTIN) used to uniquely identify products. GS1 Application Identifiers (AIs) encode additional attributes (lot number, expiry, serial number) in barcode or RFID payloads. All reviewed MRO platforms support GS1 barcode scanning for parts receiving and issue. GS1 US publishes a Product Lookup API (OpenAPI-based) for GTIN validation: https://www.gs1us.org/tools/gs1-us-data-hub/gs1-us-apis

**JSON Schema (Draft 2020-12)**
- URL: https://json-schema.org/specification
- Used to validate request and response payloads in MRO platform APIs. Relevant when defining the canonical data model for inventory records, purchase orders, and parts transactions in an AI-native platform.

### Security & Authentication Standards

**OAuth 2.0 (RFC 6749) + PKCE (RFC 7636)**
- URL: https://oauth.net/2/
- De-facto standard for API authentication across all reviewed CMMS and MRO platforms. Authorisation Code + PKCE is the recommended flow for all client types as of 2026.

**OpenID Connect Core 1.0**
- URL: https://openid.net/specs/openid-connect-core-1_0.html
- Authentication layer on top of OAuth 2.0 that adds ID Token (JWT), userinfo endpoint, and standardised scopes (openid, profile, email). Used by Infor EAM (via Infor OS IdP) and enterprise CMMS deployments for single-sign-on (SSO) integration with corporate identity providers.

**OWASP API Security Top 10 (2023)**
- URL: https://owasp.org/API-Security/editions/2023/en/0x00-header/
- Defines the top security risks for REST APIs including broken object-level authorisation (BOLA), excessive data exposure, and mass assignment. Essential reference for securing MRO inventory API endpoints, particularly parts consumption, purchase order creation, and cost data.

**NIST SP 800-53 Rev. 5 — Security and Privacy Controls**
- URL: https://csrc.nist.gov/publications/detail/sp/800-53/rev-5/final
- Relevant for MRO systems deployed in government, defence, and critical infrastructure sectors. Specifies supply chain risk management (SR) and configuration management (CM) controls applicable to maintenance parts tracking systems.

### Maintenance & Reliability Standards

**SAE JA1011 — Evaluation Criteria for Reliability-Centred Maintenance (RCM) Processes**
- URL: https://www.sae.org/standards/ja1011_202411-evaluation-criteria-reliability-centered-maintenance-rcm-processes
- Defines the minimum criteria a maintenance analysis process must meet to qualify as RCM. Drives criticality-based sparing strategies in MRO systems — classifying which parts are run-to-failure candidates versus critical-spare candidates. Updated November 2024.

**SAE JA1012 — A Guide to the Reliability-Centred Maintenance (RCM) Standard**
- URL: https://www.sae.org/standards/content/ja1012_202204/
- Companion document to JA1011 providing implementation guidance. Together with JA1011, forms the basis for spare parts criticality classification and reorder priority logic in MRO inventory systems.

**PAS 55 — Optimised Management of Physical Assets**
- Published by BSI (British Standards Institution). Precursor to ISO 55001; still widely referenced in UK utilities and infrastructure. Established the concept of asset criticality rating that informs MRO stocking level decisions.

**OSHA 29 CFR 1910.147 — The Control of Hazardous Energy (Lockout/Tagout)**
- URL: https://www.osha.gov/laws-regs/regulations/standardnumber/1910/1910.147
- US regulatory requirement governing safe maintenance procedures. MRO systems must support documentation of LOTO procedures linked to assets and associated maintenance parts lists to demonstrate compliance.

### MCP Server Specifications

**Model Context Protocol (MCP) — Specification 2025-11-05**
- URL: https://modelcontextprotocol.io/specification/2025-11-25
- Anthropic's open protocol enabling AI agents to access tools and data sources via a standardised server interface. An AI-native MRO inventory platform could expose an MCP server enabling LLMs to query stock levels, create purchase orders, search parts catalogues, and retrieve asset maintenance history through natural language agent interfaces. Enterprise MCP deployments in 2026 route through an MCP gateway for authentication, authorisation, rate limiting, and audit trails.

---

## Similar Products — Developer Documentation & APIs

### IBM Maximo (MRO Inventory Optimization)

- **Description:** Enterprise AI-driven MRO inventory optimisation module within the IBM Maximo Application Suite. Uses prescriptive analytics and criticality scoring to right-size stocking levels. Trusted by mining, utilities, oil & gas, and transportation operators globally.
- **API Documentation:** https://ibm-maximo-dev.github.io/maximo-restapi-documentation/
- **IBM API Hub:** https://developer.ibm.com/apis/catalog/maximo--maximo-manage-rest-api/
- **MRO Inventory Optimization Data REST API:** https://www.ibm.com/docs/en/mio?topic=api-data-rest-developers-guide
- **SDKs/Libraries:** No official SDKs; community Python and JavaScript wrappers available on GitHub.
- **Developer Guide:** https://ibm-maximo-dev.github.io/maximo-restapi-documentation/overview/overview/
- **Standards:** REST/JSON; OSLC (Open Services for Lifecycle Collaboration) for asset data interoperability
- **Authentication:** API Key and OAuth 2.0 Bearer Token

### Fiix CMMS (Rockwell Automation)

- **Description:** Cloud CMMS with AI-powered maintenance analytics (Fiix Foresight) and parts forecasting. Parts forecaster analyses historical data to predict future spare parts needs for up to 25 tracked SKUs simultaneously.
- **API Documentation:** https://fiixlabs.github.io/api-documentation/
- **Developer's Guide:** https://fiixlabs.github.io/api-documentation/guide.html
- **SDKs/Libraries:** Official Python SDK available; community Java client exists. API is available on Enterprise plan only.
- **Developer Guide:** https://fiixlabs.github.io/api-documentation/guide.html
- **Standards:** REST/JSON; CRUD operations over HTTPS; JSON payloads.
- **Authentication:** API Key (provided at account setup); Enterprise plan required for API access.

### MaintainX

- **Description:** Mobile-first industrial CMMS with AI-powered inventory predictions, work order management, and barcode scanning. Designed for frontline maintenance teams in manufacturing, facilities, and utilities.
- **API Documentation:** https://api.getmaintainx.com/v1/docs
- **API Tracker Reference:** https://apitracker.io/a/getmaintainx
- **SDKs/Libraries:** No official SDK published; REST API usable from any HTTP client. Zapier integration available for no-code workflows.
- **Developer Guide:** https://www.getmaintainx.com/blog/transform-your-operations-with-maintainx-integrations
- **Standards:** REST/JSON over HTTPS; Bearer Token authentication; OpenAPI-aligned endpoints.
- **Authentication:** Bearer Token; API access on Premium plan and above.

### Limble CMMS

- **Description:** Mid-market CMMS with unlimited-user pricing, multi-site inventory management, and inter-site transfer capabilities. REST API exposes assets, parts, work orders, purchase orders, vendors, and inventory endpoints.
- **API Documentation:** https://apidocs.limblecmms.com/
- **Postman Workspace:** https://www.postman.com/limbleapiqa/limble-solutions-llc-s-public-workspace/documentation/zskh2o7/limble-api-v2
- **SDKs/Libraries:** No official SDK; REST API accessible from any HTTP client.
- **Developer Guide:** https://help.limblecmms.com/en/collections/3322827-api
- **Standards:** REST/JSON; regional API endpoints (US, Canada, Australia, Europe, 21CFR). BASIC authentication.
- **Authentication:** HTTP Basic Auth (API key as credentials). Regional endpoint selection required based on account location.

### UpKeep

- **Description:** Mobile-first CMMS and EAM platform with inventory management, asset tracking, and work order management. 100+ pre-built integrations. REST API available on Enterprise plan.
- **API Documentation:** https://developers.onupkeep.com/
- **REST API Overview:** https://upkeep.com/integrations/rest-api/
- **SDKs/Libraries:** Postman collections and OpenAPI/Swagger spec available from developer portal. OAuth Playground available.
- **Developer Guide:** https://developers.onupkeep.com/ (includes webhooks, sandbox environment, changelog)
- **Standards:** REST/JSON; OpenAPI/Swagger spec published; Base endpoint: https://api.onupkeep.com/api/v2/
- **Authentication:** OAuth 2.0 and API Key. Enterprise plan required for REST API access.

### eMaint CMMS (Fluke Reliability / Fortive)

- **Description:** Cloud CMMS with native Fluke hardware sensor integration, unlimited stockroom inventory management, and (from March 2026) AI-generated work order creation from asset documentation and sensor data.
- **API Documentation:** https://www.emaint.com/emaint-cmms-api-unites-your-individual-business-processes/
- **Integration Overview:** https://www.emaint.com/emaint-integrations-your-cmms-connected/
- **SDKs/Libraries:** REST API with JSON payloads; no official SDK published.
- **Developer Guide:** Contact vendor for full API reference (not fully public-facing).
- **Standards:** REST/JSON; IIoT data ingestion from SCADA/PLC via standard OPC-UA and Modbus protocols.
- **Authentication:** API Key; enterprise agreement required for full API access.

### Infor EAM

- **Description:** Enterprise asset management platform with deep MRO inventory control, rotable spares management, and industry-specific pre-built configurations for utilities, transportation, healthcare, and government. API exposed through Infor ION API Gateway using OAuth 2.0.
- **API Catalogue:** https://developer.infor.com/hub/apicatalog
- **Developer Portal:** https://developer.infor.com/hub/apis
- **ION API Administration Guide:** https://docs.infor.com/ionapi/2021-x/en-us/ionapiag_cloud/default.html
- **GitHub Redistributable Artifacts:** https://github.com/infor-cloud/infor-eam
- **SDKs/Libraries:** WSDL artifacts for SOAP integration; REST JSON interface for modern integrations. Java and .NET client libraries generated from WSDL.
- **Standards:** SOAP (XML) and REST (JSON) interfaces; ION API Gateway with OAuth 2.0; Swagger documentation accessible via Hexagon documentation portal.
- **Authentication:** OAuth 2.0 via Infor OS identity provider (ION API Gateway). Authorised App registration required in Infor ION API.

### SAP Plant Maintenance (SAP PM / IBP for MRO)

- **Description:** SAP ERP module for plant maintenance and MRO inventory management. SAP IBP for MRO extends planning with intermittent demand forecasting specific to spare parts. Deep integration with SAP MM, PP, and FI-CO.
- **SAP API Business Hub:** https://api.sap.com/package/S4HANAOPAPI/odata
- **SAP S/4HANA Cloud OData APIs:** https://api.sap.com/package/SAPS4HANACloud/odata
- **Developer Guide:** https://www.apideck.com/blog/guide-to-sap-4-hana-rest-and-soap-api
- **SDKs/Libraries:** SAP Cloud Application Programming (CAP) model; ABAP SDK; various third-party connectors (MuleSoft, Boomi, etc.).
- **Standards:** OData V2 and V4 (preferred for new integrations); REST/JSON over HTTPS; SAP API Business Hub as central API catalogue.
- **Authentication:** OAuth 2.0 via SAP BTP (Business Technology Platform) identity services.

### Verdantis Harmonize

- **Description:** Specialised AI-powered MRO master data management platform. Performs catalogue deduplication (L2), data enrichment against verified supplier directories, classification to UNSPSC/eCl@ss, and SAP MRO master data integration. Not a CMMS — operates as a data foundation layer beneath CMMS and ERP.
- **API Documentation:** https://www.verdantis.com/ (limited public API documentation; vendor-managed integration)
- **SAP Integration Guide:** https://www.verdantis.com/sap-mro-master/
- **SDKs/Libraries:** Custom integration via vendor-provided connectors; SAP-native connector available.
- **Standards:** Internal UNSPSC and eCl@ss classification; REST/JSON integration layer with ERP systems.
- **Authentication:** Enterprise SaaS; authentication details via vendor agreement.

### GS1 US Product Lookup API

- **Description:** API for GTIN validation and product attribute lookup from the GS1 US Data Hub. Enables MRO inventory systems to validate part barcodes, retrieve manufacturer data, and support receiving workflows. Based on OpenAPI standard.
- **API Documentation:** https://www.gs1us.org/tools/gs1-us-data-hub/gs1-us-apis
- **Developer Portal Guide:** https://documents.gs1us.org/adobe/assets/deliver/urn:aaid:aem:b91e282f-2d95-4e37-b9db-05e31cd263bc/gs1-us-data-hub-api-developer-portal-user-guide.pdf
- **API Help:** https://www.help.gs1us.org/api-documentation
- **SDKs/Libraries:** No official SDK; OpenAPI spec enables client generation in any language.
- **Standards:** OpenAPI 3.x; REST/JSON; GS1 V5/V6 API versions maintained for legacy compatibility.
- **Authentication:** API Key (obtained via GS1 US subscription).

---

## Notes

**Intermittent Demand Forecasting Gap**: Standard inventory management data models and APIs (including SAP IBP and IBM Maximo) are still evolving in their support for probabilistic demand forecasting at the SKU level. Most platforms expose deterministic reorder point values rather than probability distributions, making it difficult for consuming systems to implement dynamic safety stock calculations. An AI-native open platform could standardise this through a well-defined forecast API returning confidence intervals alongside point estimates.

**MCP Integration Opportunity**: No reviewed CMMS or MRO platform currently publishes an official MCP server. This is an open opportunity — exposing inventory query, parts lookup, purchase order creation, and work order generation as MCP tools would allow large language model agents to perform MRO inventory operations through natural language, aligning with the 2026 enterprise MCP deployment trend.

**UNSPSC/eCl@ss Mapping**: There is no single official API for cross-mapping UNSPSC to eCl@ss codes at scale. Commercial services (Verdantis, Enventure, OptimizeMRO) provide this as a managed service rather than a self-service API. An open-source mapping dataset or API would be genuinely novel and valuable to the MRO domain.

**Barcode Standard Coverage**: All reviewed platforms support GS1 linear barcodes (EAN-13, Code 128) but RFID (EPC/GS1 Gen 2) support for storeroom operations is inconsistent and typically requires third-party hardware integration. This is an emerging area with no dominant standard API layer.
