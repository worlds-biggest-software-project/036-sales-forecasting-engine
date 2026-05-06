# Standards & API Reference

> Project: Sales Forecasting Engine · Generated: 2026-05-06

## Industry Standards & Specifications

### ISO Standards

**ISO/IEC 5259-2:2024 — AI Data Quality Measures**
- URL: https://www.iso.org/standard/81860.html
- Defines a data quality model and a set of measurable characteristics for assessing and reporting on data quality in the context of analytics and machine learning. Directly applicable to the CRM data ingestion layer, where stale close dates, missing activities, and inflated amounts degrade forecast reliability. Builds on ISO/IEC 25012 and ISO 8000.

**ISO/IEC 5259-3:2024 — AI Data Quality Management Requirements**
- URL: https://www.iso.org/standard/81092.html
- Provides requirements and guidance for establishing, implementing, maintaining, and continually improving data quality management systems for analytics and ML. Relevant for defining data quality governance in the forecasting pipeline — particularly when ingesting CRM data from multiple sources. SGS certified the first compliant implementation in November 2025.

**ISO 8000 — Data Quality Standard**
- URL: https://en.wikipedia.org/wiki/ISO_8000
- Foundational standard for master data and transactional data quality. Establishes principles for data portability, provenance, and quality measurement. Referenced by ISO/IEC 5259 series and applicable to the CRM data normalisation layer when consolidating records across multiple CRM instances.

**ISO 20252:2019 — Market Research and Insights Data Vocabulary**
- URL: https://www.iso.org/standard/73671.html
- Defines vocabulary and service requirements for market, opinion, and social research, including data analytics. Relevant for aligning the terminology used in market analysis and demand forecasting components of the engine.

**ISO/IEC 25010 — Software Quality Model**
- URL: https://www.iso.org/standard/35733.html
- Defines quality characteristics including functional suitability, reliability, maintainability, security, and usability. Provides the quality framework for defining accuracy, reliability, and maintainability requirements of the forecasting engine itself, particularly for the ML model retraining pipeline and prediction serving layer.

### W3C & IETF Standards

**RFC 6749 — The OAuth 2.0 Authorization Framework**
- URL: https://datatracker.ietf.org/doc/html/rfc6749
- The core OAuth 2.0 specification defining the authorization framework that all major CRM APIs (Salesforce, HubSpot, Pipedrive, Zoho, Dynamics 365) require for third-party integrations. Any forecasting engine connector must implement the Authorization Code grant flow defined here.

**RFC 6750 — The OAuth 2.0 Authorization Framework: Bearer Token Usage**
- URL: https://www.ietf.org/rfc/rfc6750.txt
- Specifies how Bearer tokens obtained via OAuth 2.0 are transmitted in HTTP requests. All CRM API calls from the forecasting engine's connectors must implement Bearer token headers as specified in this RFC.

**OAuth 2.1 (IETF Draft) — Consolidated OAuth Framework**
- URL: https://datatracker.ietf.org/doc/draft-ietf-oauth-v2-1/
- Consolidates OAuth 2.0 and its most important extensions, deprecating implicit and resource owner password credential flows. HubSpot's 2026-03 API endpoints implement OAuth 2.1-style token management. New connectors should target OAuth 2.1 security practices.

**OAuth 2.0 Security Best Current Practice**
- URL: https://datatracker.ietf.org/doc/html/draft-ietf-oauth-security-topics
- Documents current best security practices for OAuth 2.0, extending the threat model from RFC 6749. Mandatory reference for implementing secure CRM connector authentication flows, particularly around PKCE and token storage.

**OpenID Connect Core 1.0**
- URL: https://openid.net/developers/how-connect-works/
- Identity layer built on top of OAuth 2.0 that provides authenticated user identity for multi-tenant SaaS deployment models. Relevant when building a cloud-hosted version of the forecasting engine that requires user authentication across enterprise tenants.

**OData v4 — Open Data Protocol (OASIS Standard)**
- URL: https://learn.microsoft.com/en-us/dynamics365/fin-ops-core/dev-itpro/data-entities/odata
- OASIS standard REST protocol underlying Microsoft Dynamics 365, Dataverse, and Power Platform APIs. The Dynamics 365 Web API is an OData v4 REST service; any Dynamics 365 connector must implement OData query syntax (`$select`, `$filter`, `$expand`, `$orderby`) for data retrieval. Also used by some Salesforce external services configurations.

### Data Model & API Specifications

**OpenAPI Specification v3.1.0 / v3.2.0**
- URL: https://spec.openapis.org/oas/v3.1.0.html and https://spec.openapis.org/oas/v3.2.0.html
- The de-facto standard for describing RESTful HTTP APIs in a machine-readable format (JSON or YAML). All major CRM vendors (Salesforce, Zoho, HubSpot) publish OpenAPI specifications for their APIs. The forecasting engine's own API surface should be described using OpenAPI 3.1+ for automated SDK generation and documentation.

**JSON Schema Draft 2020-12**
- URL: https://json-schema.org/draft/2020-12
- The specification underlying OpenAPI's data type definitions and the standard for JSON instance validation. Used to define and validate the request/response schemas for CRM webhook payloads, forecast submission formats, and deal-score output structures. OpenAPI 3.1 aligns its schema vocabulary with JSON Schema 2020-12.

**MEDDIC / MEDDPICC Sales Qualification Methodology**
- Not a formal technical standard, but the de-facto data schema convention in B2B sales forecasting. Dimensions: Metrics, Economic Buyer, Decision Criteria, Decision Process, Identify Pain, Champion, (Competition). Forecasting platforms model deal-health scoring dimensions around MEDDIC vocabulary; the engine's deal-scoring schema should map to these dimensions to align with buyer expectations.

**Salesforce Object Model (de-facto schema standard)**
- URL: https://developer.salesforce.com/docs/atlas.en-us.api_rest.meta/api_rest/
- Salesforce's standard object model — Opportunity, Account, Contact, Activity, ForecastingItem — is the de-facto CRM schema that any multi-CRM forecasting engine must map to. The Opportunity object's fields (Amount, CloseDate, StageName, ForecastCategoryName, Probability) define the canonical deal record schema for the industry.

### Security & Authentication Standards

**OWASP API Security Top 10**
- URL: https://owasp.org/www-project-api-security/
- Documents the most critical API security risks. Directly applicable to the forecasting engine's API surface and CRM connector token management. Key risks include broken object-level authorisation (BOLA), excessive data exposure, and insufficient rate limiting — all relevant when exposing deal score and forecast data via REST APIs.

**GDPR (General Data Protection Regulation) — EU 2016/679**
- URL: https://gdpr.eu/
- European data protection regulation governing the processing of personal data. CRM records ingested by the forecasting engine contain personal data (contact names, email addresses, communication history). GDPR requires lawful basis for processing, data minimisation, subject access rights, and data retention limits. GDPR is increasingly enforced as the primary regulatory tool for AI data processing in the EU (>€6.2B in fines since 2018, >60% issued since January 2023). Self-hosted deployment options help data-sovereign customers remain compliant.

**CCPA / CPRA (California Consumer Privacy Act / Privacy Rights Act)**
- URL: https://oag.ca.gov/privacy/ccpa
- US state privacy law with similar provisions to GDPR for California residents. Relevant for US-based SaaS deployments handling personal data from California-based sales contacts. Call recording features have specific consent implications under CCPA and California Penal Code § 632.

**ASC 606 / GAAP Revenue Recognition Standard**
- URL: https://asc.fasb.org/606
- US accounting standard governing when and how revenue is recognised. Finance and accounting teams use pipeline forecasts to model recognised revenue; forecast outputs must align with ASC 606 accounting periods (quarterly) to be usable by CFO/Finance personas. Comparable IFRS 15 standard applies in international contexts.

### MCP Server Specifications

**Model Context Protocol (MCP) — Anthropic**
- URL: https://modelcontextprotocol.io/
- Open protocol enabling AI models to interact with external tools and data sources via a standardised interface. Relevant for integrating the forecasting engine with AI assistant workflows: an MCP server exposing deal scores, pipeline summaries, and forecast data would allow Claude, Cursor, and other AI tools to query forecast data in natural-language conversational contexts. JSON Schema 2020-12 is being standardised as the default dialect for MCP tool definitions (SEP-1613).

**OpenTelemetry (OTEL) — CNCF Standard**
- URL: https://opentelemetry.io/
- Vendor-neutral open standard for collecting, processing, and exporting telemetry data (traces, metrics, logs) from cloud-native applications. The de-facto standard for ML model inference monitoring in production: traces model prediction latency, feature pipeline health, and data drift signals. OpenTelemetry's semantic conventions for generative AI (2024) extend to LLM-augmented forecasting pipelines. Supported by all major observability backends (Datadog, Grafana, Elastic, Google Cloud, Azure Monitor).

---

## Similar Products — Developer Documentation & APIs

### Salesforce CRM (REST API)

- **Description:** The dominant CRM platform for enterprise B2B sales; provides standard Opportunity, Account, Contact, and Activity objects that are the canonical deal record schema for the industry. Spring '26 release (v66.0) updated authentication to prefer External Client Apps over Connected Apps.
- **API Documentation:** https://developer.salesforce.com/docs/atlas.en-us.api_rest.meta/api_rest/
- **OpenAPI / PDF Reference:** https://resources.docs.salesforce.com/latest/latest/en-us/sfdc/pdf/api_rest.pdf (v66.0, April 2026)
- **Forecasts REST API:** https://developer.salesforce.com/docs/atlas.en-us.chatterapi.meta/chatterapi/connect_resources_forecasting.htm
- **Einstein Discovery REST API:** https://developer.salesforce.com/docs/atlas.en-us.bi_dev_guide_rest_sdd.meta/bi_dev_guide_rest_sdd/bi_rest_sdd_overview.htm
- **Authentication:** OAuth 2.0 via External Client Apps (Spring '26+); Authorization Code and Client Credentials flows; Bearer token in HTTP headers (RFC 6750)
- **Standards:** REST/JSON; OpenAPI 3.0 external services schema; SOQL query language for object traversal
- **SDKs:** JSForce (Node.js, community); simple-salesforce (Python); Salesforce SDK for Python, Java

### HubSpot CRM (API v3 / 2026-03)

- **Description:** Second-largest CRM platform; primary target for SMB and mid-market sales forecasting integrations. Provides Deal, Contact, Company, and Activity objects plus a native Forecast API (public beta).
- **API Documentation:** https://developers.hubspot.com/docs/api-reference/latest/
- **Forecasts API:** https://developers.hubspot.com/docs/api-reference/legacy/crm/objects/forecasts/guide
- **OAuth Token Management (2026-03):** https://developers.hubspot.com/docs/api-reference/latest/authentication/manage-oauth-tokens
- **CRM Pipelines API:** https://developers.hubspot.com/docs/api-reference/crm-pipelines-v3/guide
- **Authentication:** OAuth 2.1-style (2026-03 endpoints require parameters in request body, not query string); Private Apps for internal integrations; Bearer tokens expire after 30 minutes; refresh tokens are long-lived
- **Standards:** REST/JSON; OpenAPI descriptions available via developer portal
- **SDKs:** Official HubSpot Node.js, Python, PHP, Ruby, Go, Java SDKs; all open-source on GitHub

### Microsoft Dynamics 365 Sales (Dataverse Web API)

- **Description:** Enterprise CRM and ERP suite built on the Microsoft Power Platform; uses OData v4 as its primary API protocol. Native `msdyn_ForecastApi` custom action provides programmatic access to sales forecasting data, snapshots, and rollups.
- **API Documentation:** https://learn.microsoft.com/en-us/rest/dynamics365/
- **Forecasting API Reference:** https://learn.microsoft.com/en-us/dynamics365/sales/developer/reference/custom-actions/msdyn_forecastapi
- **Forecasting API Overview:** https://learn.microsoft.com/en-us/dynamics365/sales/developer/reference/custom-actions-manual-forecasting
- **Authentication:** Azure Active Directory / Microsoft Entra ID; OAuth 2.0 Authorization Code and Client Credentials flows; Bearer token in HTTP headers
- **Standards:** OData v4 (OASIS); REST/JSON; OpenAPI available via Power Platform API portal
- **SDKs:** Microsoft.Xrm.Data.PowerShell (PowerShell); Dataverse client libraries for .NET, Python, JavaScript

### Pipedrive (REST API v1)

- **Description:** CRM focused on pipeline-oriented sales teams; popular with SMB and early-stage startups. Provides Deal, Person, Organisation, and Activity objects; OAuth 2.0 apps can read and write all pipeline data.
- **API Documentation:** https://developers.pipedrive.com/docs/api/v1
- **Developer Portal:** https://developers.pipedrive.com/
- **Authentication Guide:** https://pipedrive.readme.io/docs/core-api-concepts-authentication
- **OAuth Endpoints:** https://developers.pipedrive.com/docs/api/v1/Oauth
- **Authentication:** OAuth 2.0 Authorization Code flow (for published apps); API token (for private integrations); access tokens expire after ~1 hour with proactive refresh required
- **Standards:** REST/JSON; OpenAPI specification available
- **SDKs:** Official Node.js client (GitHub: pipedrive/client-nodejs); official PHP client; community Python client

### Zoho CRM (REST API v8)

- **Description:** Affordable CRM with built-in AI (Zia) for deal probability scoring and anomaly detection; most affordable AI-powered forecasting of any reviewed platform. Provides standard CRM objects plus Zia prediction fields.
- **API Documentation:** https://www.zoho.com/crm/developer/docs/api/v8/
- **Developer Portal:** https://www.zoho.com/crm/developer/
- **OpenAPI Specification:** https://www.zoho.com/crm/developer/docs/api/v8/openapi-specification.html
- **Authentication:** OAuth 2.0 (Zoho Accounts server); supports Authorization Code, Client Credentials, and Refresh Token flows
- **Standards:** REST/JSON; OpenAPI 3.0 specification hosted on GitHub; COQL (CRM Object Query Language) for complex record retrieval; Bulk API for asynchronous large-dataset operations
- **SDKs:** Official Java, Python, PHP, Node.js, C#, Ruby, JavaScript, Android, iOS SDKs

### Clari (Revenue Intelligence API)

- **Description:** Enterprise revenue intelligence platform; provides REST APIs to retrieve and push deal scores, forecast roll-up data, and pipeline signals to/from external systems. Primarily used for BI tool integration and data export rather than CRM replacement.
- **API Documentation:** https://developer.clari.com/
- **API Reference:** https://developer.clari.com/documentation/external_spec
- **Copilot (Call Intelligence) API:** https://api-doc.copilot.clari.com/
- **Authentication:** API token in `apikey` header for standard endpoints; `partnerkey` header additionally required for partner/ingest APIs; Copilot uses `X-Api-Key` and `X-Api-Password` headers; OAuth 2.0 for published integrations
- **Standards:** REST/JSON; Swagger (OpenAPI) interactive specification published at developer portal; webhooks for real-time event processing
- **SDKs:** No official SDK; community Python data loader via dltHub (https://dlthub.com/context/source/clari)

### Gong (Conversation Intelligence API v2)

- **Description:** Conversation intelligence and forecasting platform; REST API provides access to call recordings, transcripts, deal likelihood scores, and engagement data. Two distinct APIs: Standard API (call data and CRM enrichment) and Engage API (outreach automation).
- **API Documentation:** https://help.gong.io/docs/what-the-gong-api-provides
- **Authentication Guide:** https://help.gong.io/docs/create-an-app-for-gong
- **Base URL:** `https://api.gong.io/v2/`
- **Authentication:** Basic Auth with Access Key and Access Key Secret for standard integrations; OAuth 2.0 for published marketplace apps; admin-level Gong account required to obtain API credentials
- **Rate Limits:** ~1,000 requests per hour for most endpoints; pagination required for list endpoints
- **Standards:** REST/JSON; consistent JSON response structure; standard HTTP status codes
- **SDKs:** No official SDK; community Python data loader via dltHub (https://dlthub.com/context/source/gong)

### Facebook Prophet (Open-Source Forecasting Library)

- **Description:** Open-source time series decomposition library (MIT licence) published by Meta; widely used as the time-series forecasting backend in bespoke sales forecasting pipelines. Scikit-learn-style API (fit/predict); supports trend, seasonality, and holiday effect decomposition.
- **Documentation:** https://facebook.github.io/prophet/
- **Quick Start:** https://facebook.github.io/prophet/docs/quick_start.html
- **GitHub Repository:** https://github.com/facebook/prophet
- **PyPI Package:** https://pypi.org/project/prophet/
- **Authentication:** N/A (local library)
- **Standards:** Python API following scikit-learn conventions; pandas DataFrame input/output; Stan/MCMC probabilistic backend; integrates with MLflow for experiment tracking
- **Licence:** MIT — fully permissive for commercial embedding
- **Related:** NeuralProphet (community fork extending Prophet with neural network components; more actively maintained)

### XGBoost + SHAP (Open-Source ML Libraries)

- **Description:** XGBoost (gradient-boosted trees, Apache 2.0) is the accuracy leader in real-world sales forecasting benchmarks (M5 competition). SHAP (SHapley Additive exPlanations) provides model-agnostic feature importance explanations — the primary mechanism for generating natural-language deal score justifications ("probability dropped because no exec sponsor and three close date slips").
- **XGBoost API Documentation:** https://xgboost.readthedocs.io/en/stable/python/python_api.html
- **SHAP Documentation:** https://shap.readthedocs.io/
- **SHAP GitHub:** https://github.com/shap/shap
- **Authentication:** N/A (local libraries)
- **Standards:** Python API following scikit-learn conventions; native `pred_contribs` parameter returns SHAP values directly; integrates with MLflow, Weights & Biases for experiment tracking; FastAPI/Flask for serving prediction endpoints
- **Licence:** XGBoost Apache 2.0 (includes patent grant); SHAP MIT
- **Related Libraries:** LightGBM (MIT), CatBoost (Apache 2.0), scikit-learn (BSD-3) — all permissively licensed

---

## Notes

**Salesforce API versioning:** Salesforce releases two major API versions per year (Spring and Fall). As of Spring '26, the REST API is at v66.0. Creating new Connected Apps is restricted; Salesforce now recommends External Client Apps for new integrations. CRM connector implementations should target the stable `/services/data/vXX.0/` versioned endpoint rather than `/latest/`.

**HubSpot Forecast API maturity:** HubSpot's native Forecast API was in public beta as of early 2026; developer access is available but the schema may change before GA. The legacy forecasts endpoint (`/crm/v3/objects/forecasts`) is stable for reading submitted forecasts, but programmatic forecast submission remains limited. Pipeline and Deal APIs (v3) are stable and GA.

**Dynamics 365 `msdyn_ForecastApi`:** This custom action (updated January 2025) supports snapshot storage and historical comparison for forecast data — making it particularly useful for the historical forecast accuracy tracking feature. The OData $filter syntax for querying forecast hierarchies can be complex; the Microsoft Learn documentation includes sample code.

**Call recording compliance gap:** GDPR Article 6 (lawful basis) and CCPA § 1798.100 both have implications for the conversation intelligence feature (call recording + transcription). EU and California law requires affirmative consent from all call participants for recording. Any implementation of the Whisper-based call signal extraction feature must include consent collection UI and configurable data retention policies to remain compliant.

**GAAP / ASC 606 alignment:** Finance teams expect forecast outputs to align with quarterly accounting periods and recognised-revenue line items. The forecasting engine should natively support period-based forecast views (quarter, fiscal year) rather than only rolling 90-day windows to be usable by CFO/Finance buyer personas.
