# Data Model Suggestion 3: Hybrid Relational + JSONB

> Project: Sales Forecasting Engine · Created: 2026-05-11

## Philosophy

The hybrid relational + JSONB approach uses strongly typed relational columns for the core fields that every deal, account, and forecast shares, while placing CRM-specific, jurisdiction-specific, and rapidly evolving fields in JSONB columns. This gives the engine the referential integrity of a relational model for its canonical data while providing the flexibility of a document store for the fields that vary across CRM sources, tenant configurations, and product iterations.

This architecture directly addresses the multi-CRM consolidation requirement. Salesforce Opportunities have fields like `ForecastCategoryName`, `HasOpportunityLineItem`, and `ContractId` that have no HubSpot equivalent. HubSpot Deals have properties like `hs_deal_stage_probability`, `hs_projected_amount`, and `hs_analytics_source` that Salesforce does not provide. Rather than discarding these CRM-specific fields or creating a sparse 200-column table, the hybrid model stores them in a `crm_fields` JSONB column that preserves the original CRM data while keeping the canonical fields in typed columns for efficient querying.

PostgreSQL's JSONB support is mature and production-proven: GIN indexing enables fast containment queries, `jsonb_path_query` supports SQL/JSON path expressions, and partial indexes on JSONB keys provide targeted performance optimization. This approach is widely used in multi-tenant SaaS platforms (Citus/Microsoft recommends it for tenant-specific custom fields) and is the standard pattern for CRM integration platforms that must accommodate unpredictable source schemas.

**Best for:** Rapid MVP development with multi-CRM flexibility, teams that expect frequent schema evolution, and products that need to preserve CRM-native field fidelity while maintaining a canonical query surface.

**Trade-offs:**
- Pro: Multi-CRM support without schema migrations — new CRM fields go into JSONB automatically
- Pro: Tenant-specific custom fields are trivially supported
- Pro: Faster MVP iteration — add new data points to JSONB before promoting them to typed columns
- Pro: Preserves full CRM-native field fidelity for debugging and data lineage
- Pro: Lower table count than fully normalized model; fewer joins for dashboard queries
- Con: JSONB fields lack database-level type enforcement; validation must happen in application code
- Con: JSONB queries are slower than typed column queries for large-scale analytics
- Con: Schema documentation is split between DDL (typed columns) and application code (JSONB structure)
- Con: GIN indexes on JSONB are larger and slower to update than B-tree indexes on typed columns
- Con: ORM integration is less ergonomic; JSONB fields require custom serialization

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| Salesforce Object Model | Salesforce-specific Opportunity fields stored in `crm_fields` JSONB preserving native field names |
| HubSpot CRM Object Model | HubSpot-specific Deal properties stored in `crm_fields` JSONB preserving native property names |
| Pipedrive / Dynamics 365 | Additional CRM source fields stored in `crm_fields` without schema changes |
| MEDDIC / MEDDPICC | Qualification data stored in `qualification` JSONB on deals, allowing flexible dimension sets per tenant |
| ISO/IEC 5259-2:2024 | Data quality assessment results stored in `quality` JSONB with structured flag arrays |
| ISO 4217 | Currency codes as typed VARCHAR(3) columns; multi-currency amounts in typed NUMERIC columns |
| ISO 3166-1 | Country codes as typed VARCHAR(2) columns on accounts |
| OAuth 2.0 (RFC 6749) | Token data in `auth` JSONB on crm_connections, accommodating different OAuth flows per CRM |
| ASC 606 / GAAP | Fiscal period configuration in `settings` JSONB on tenants, supporting custom fiscal year starts |
| JSON Schema 2020-12 | JSONB column structures documented as JSON Schema definitions in the `field_schemas` table |

---

## Core Tenancy & Configuration

```sql
-- =============================================================
-- TENANCY WITH JSONB SETTINGS
-- =============================================================

CREATE TABLE tenants (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    plan            VARCHAR(50) NOT NULL DEFAULT 'free',
    settings        JSONB NOT NULL DEFAULT '{}',
    -- settings example:
    -- {
    --   "fiscal_year_start_month": 1,
    --   "default_currency": "USD",
    --   "forecast_categories": ["pipeline", "best_case", "commit", "closed_won", "closed_lost"],
    --   "scoring_frequency": "daily",
    --   "alert_defaults": {
    --     "no_activity_days": 14,
    --     "probability_drop_threshold": 15
    --   },
    --   "custom_deal_fields": [
    --     {"key": "territory", "label": "Territory", "type": "string"},
    --     {"key": "product_line", "label": "Product Line", "type": "enum", "values": ["core", "premium", "enterprise"]}
    --   ]
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    email           VARCHAR(255) NOT NULL,
    name            VARCHAR(255) NOT NULL,
    role            VARCHAR(50) NOT NULL DEFAULT 'viewer',
    profile         JSONB NOT NULL DEFAULT '{}',
    -- profile example:
    -- {
    --   "avatar_url": "https://...",
    --   "team": "West Coast Enterprise",
    --   "quota": 500000.00,
    --   "manager_id": "uuid",
    --   "notification_preferences": {"slack": true, "email": false}
    -- }
    last_login_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, email)
);

CREATE INDEX idx_users_tenant ON users(tenant_id);

CREATE TABLE crm_connections (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    crm_type        VARCHAR(50) NOT NULL,
    instance_url    TEXT,
    auth            JSONB NOT NULL,
    -- auth example (Salesforce):
    -- {
    --   "access_token": "encrypted:...",
    --   "refresh_token": "encrypted:...",
    --   "token_expires_at": "2026-05-11T14:30:00Z",
    --   "instance_url": "https://mycompany.my.salesforce.com",
    --   "api_version": "v66.0",
    --   "scopes": ["api", "refresh_token"]
    -- }
    -- auth example (HubSpot):
    -- {
    --   "access_token": "encrypted:...",
    --   "refresh_token": "encrypted:...",
    --   "token_expires_at": "2026-05-11T12:30:00Z",
    --   "portal_id": "12345678",
    --   "api_version": "2026-03"
    -- }
    sync_config     JSONB NOT NULL DEFAULT '{}',
    -- sync_config example:
    -- {
    --   "sync_frequency_minutes": 15,
    --   "object_mapping": {
    --     "opportunity": {"enabled": true, "last_sync": "2026-05-11T12:00:00Z"},
    --     "account": {"enabled": true, "last_sync": "2026-05-11T12:00:00Z"},
    --     "contact": {"enabled": true, "last_sync": "2026-05-11T12:00:00Z"},
    --     "activity": {"enabled": true, "last_sync": "2026-05-11T12:00:00Z"}
    --   },
    --   "field_mapping_overrides": {
    --     "Amount": "amount",
    --     "Custom_ARR__c": "arr"
    --   }
    -- }
    sync_status     VARCHAR(50) NOT NULL DEFAULT 'pending',
    last_sync_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_crm_connections_tenant ON crm_connections(tenant_id);
```

## CRM Data Layer with JSONB Extensions

```sql
-- =============================================================
-- ACCOUNTS — TYPED CORE + JSONB EXTENSIONS
-- =============================================================

CREATE TABLE accounts (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    crm_connection_id UUID NOT NULL REFERENCES crm_connections(id) ON DELETE CASCADE,
    external_id     VARCHAR(255) NOT NULL,

    -- Canonical typed fields (common across all CRMs)
    name            VARCHAR(500) NOT NULL,
    industry        VARCHAR(255),
    employee_count  INTEGER,
    annual_revenue  NUMERIC(18,2),
    country_code    VARCHAR(2),               -- ISO 3166-1 alpha-2
    website         TEXT,
    owner_external_id VARCHAR(255),

    -- CRM-specific fields preserved as-is from the source system
    crm_fields      JSONB NOT NULL DEFAULT '{}',
    -- Salesforce example:
    -- {
    --   "Type": "Customer - Direct",
    --   "Rating": "Hot",
    --   "AccountSource": "Web",
    --   "Sic": "3674",
    --   "TickerSymbol": "ACME",
    --   "Ownership": "Public",
    --   "NumberOfEmployees": 5000,
    --   "Custom_Segment__c": "Enterprise"
    -- }
    -- HubSpot example:
    -- {
    --   "hs_analytics_source": "ORGANIC_SEARCH",
    --   "hs_analytics_num_page_views": 47,
    --   "lifecyclestage": "customer",
    --   "hs_target_account": true
    -- }

    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    synced_at       TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(crm_connection_id, external_id)
);

CREATE INDEX idx_accounts_tenant ON accounts(tenant_id);
CREATE INDEX idx_accounts_name ON accounts(tenant_id, name);
CREATE INDEX idx_accounts_crm_fields ON accounts USING GIN (crm_fields);

-- =============================================================
-- CONTACTS — TYPED CORE + JSONB EXTENSIONS
-- =============================================================

CREATE TABLE contacts (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    crm_connection_id UUID NOT NULL REFERENCES crm_connections(id) ON DELETE CASCADE,
    external_id     VARCHAR(255) NOT NULL,
    account_id      UUID REFERENCES accounts(id) ON DELETE SET NULL,

    -- Canonical typed fields
    first_name      VARCHAR(255),
    last_name       VARCHAR(255),
    email           VARCHAR(255),
    phone           VARCHAR(100),
    title           VARCHAR(255),
    seniority_level VARCHAR(50),

    -- CRM-specific fields
    crm_fields      JSONB NOT NULL DEFAULT '{}',

    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    synced_at       TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(crm_connection_id, external_id)
);

CREATE INDEX idx_contacts_tenant ON contacts(tenant_id);
CREATE INDEX idx_contacts_account ON contacts(account_id);
CREATE INDEX idx_contacts_email ON contacts(tenant_id, email);

-- =============================================================
-- DEALS — THE HEART OF THE MODEL: TYPED CORE + RICH JSONB
-- =============================================================

CREATE TABLE deals (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    crm_connection_id UUID NOT NULL REFERENCES crm_connections(id) ON DELETE CASCADE,
    external_id     VARCHAR(255) NOT NULL,
    account_id      UUID REFERENCES accounts(id) ON DELETE SET NULL,

    -- ====== CANONICAL TYPED FIELDS ======
    -- These fields exist in every CRM and are the primary query/filter targets.
    name            VARCHAR(500) NOT NULL,
    amount          NUMERIC(18,2),
    currency_code   VARCHAR(3) DEFAULT 'USD',
    stage_name      VARCHAR(255) NOT NULL,
    pipeline_name   VARCHAR(255),
    close_date      DATE,
    is_closed       BOOLEAN NOT NULL DEFAULT FALSE,
    is_won          BOOLEAN NOT NULL DEFAULT FALSE,
    deal_type       VARCHAR(100),
    lead_source     VARCHAR(255),
    owner_external_id VARCHAR(255),
    owner_user_id   UUID REFERENCES users(id) ON DELETE SET NULL,

    -- ====== CRM-NATIVE FIELDS (preserved from source) ======
    crm_fields      JSONB NOT NULL DEFAULT '{}',
    -- Salesforce-specific example:
    -- {
    --   "ForecastCategoryName": "Pipeline",
    --   "Probability": 30,
    --   "HasOpportunityLineItem": false,
    --   "ContractId": null,
    --   "CampaignId": "7015f00000ABC123",
    --   "NextStep": "Send proposal by Friday",
    --   "Description": "Enterprise license renewal",
    --   "LastActivityDate": "2026-05-10",
    --   "Custom_ARR__c": 180000,
    --   "Custom_Product_Interest__c": "Platform;Analytics"
    -- }
    -- HubSpot-specific example:
    -- {
    --   "hs_deal_stage_probability": 0.3,
    --   "hs_projected_amount": 150000,
    --   "hs_analytics_source": "PAID_SEARCH",
    --   "hs_analytics_source_data_1": "google-ads",
    --   "hs_closed_amount": 0,
    --   "hs_is_closed_won": false,
    --   "notes_last_updated": "2026-05-09T16:30:00Z"
    -- }

    -- ====== QUALIFICATION (flexible MEDDIC + custom) ======
    qualification   JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "methodology": "MEDDPICC",
    --   "metrics": "Customer needs 30% reduction in forecast error",
    --   "economic_buyer": "Sarah Chen, CFO",
    --   "decision_criteria": "Accuracy > 80%, CRM integration, self-hostable",
    --   "decision_process": "POC → Security review → Board approval",
    --   "identify_pain": "Current forecasts off by 40%, CFO losing trust",
    --   "champion": "Mike Torres, VP Sales Ops",
    --   "competition": "Clari (expensive), internal Excel model",
    --   "paper_process": "MSA required, 30-day review cycle",
    --   "completeness_score": 0.75,
    --   "last_updated": "2026-05-08"
    -- }

    -- ====== ML SCORING (latest score embedded for fast reads) ======
    scoring         JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "win_probability": 0.6823,
    --   "confidence_lower": 0.5901,
    --   "confidence_upper": 0.7745,
    --   "risk_level": "medium",
    --   "score_percentile": 62,
    --   "model_id": "uuid",
    --   "model_version": "xgboost-v3.2",
    --   "scored_at": "2026-05-11T06:00:00Z",
    --   "explanation": "Score at 68%: strong champion engagement offset by 2 close date slips",
    --   "top_factors": [
    --     {"feature": "has_champion", "impact": 0.089, "direction": "positive"},
    --     {"feature": "close_date_slip_count", "impact": -0.065, "direction": "negative"},
    --     {"feature": "contact_count", "impact": 0.041, "direction": "positive"},
    --     {"feature": "days_since_activity", "impact": -0.038, "direction": "negative"}
    --   ],
    --   "previous_probability": 0.7214,
    --   "trend": "declining"
    -- }

    -- ====== DATA QUALITY (computed by engine) ======
    quality         JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "score": 42.5,
    --   "flags": ["stale_close_date", "no_activity_30_days", "missing_champion"],
    --   "details": {
    --     "stale_close_date": {"days_past": 15, "severity": "warning"},
    --     "no_activity_30_days": {"days": 34, "severity": "critical"},
    --     "missing_champion": {"severity": "info"}
    --   },
    --   "last_assessed": "2026-05-11T06:00:00Z"
    -- }

    -- ====== LIFECYCLE METRICS (computed from activity/change history) ======
    lifecycle       JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "days_in_pipeline": 67,
    --   "days_in_current_stage": 12,
    --   "days_since_last_activity": 5,
    --   "close_date_slip_count": 2,
    --   "stage_change_count": 3,
    --   "activity_count_30d": 8,
    --   "contact_count": 4,
    --   "has_decision_maker": true,
    --   "has_champion": true,
    --   "email_count_30d": 5,
    --   "call_count_30d": 3,
    --   "meeting_count_30d": 2
    -- }

    -- ====== TENANT CUSTOM FIELDS ======
    custom_fields   JSONB NOT NULL DEFAULT '{}',
    -- Tenant-defined custom fields (configured in tenants.settings.custom_deal_fields):
    -- {
    --   "territory": "West Coast",
    --   "product_line": "enterprise",
    --   "partner_referral": true,
    --   "competitor_names": ["Clari", "Aviso"]
    -- }

    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    synced_at       TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(crm_connection_id, external_id)
);

-- Typed column indexes for common queries
CREATE INDEX idx_deals_tenant ON deals(tenant_id);
CREATE INDEX idx_deals_account ON deals(account_id);
CREATE INDEX idx_deals_stage ON deals(tenant_id, stage_name);
CREATE INDEX idx_deals_close_date ON deals(tenant_id, close_date);
CREATE INDEX idx_deals_open ON deals(tenant_id, is_closed) WHERE NOT is_closed;
CREATE INDEX idx_deals_pipeline ON deals(tenant_id, pipeline_name);

-- JSONB indexes for scoring and quality queries
CREATE INDEX idx_deals_risk ON deals((scoring->>'risk_level')) WHERE (scoring->>'risk_level') IN ('high', 'critical');
CREATE INDEX idx_deals_win_prob ON deals(((scoring->>'win_probability')::NUMERIC)) WHERE NOT is_closed;
CREATE INDEX idx_deals_dq_score ON deals(((quality->>'score')::NUMERIC)) WHERE NOT is_closed;

-- GIN index for CRM-specific field searches
CREATE INDEX idx_deals_crm_fields ON deals USING GIN (crm_fields);
CREATE INDEX idx_deals_custom_fields ON deals USING GIN (custom_fields);
```

## Activities & Relationships

```sql
-- =============================================================
-- ACTIVITIES — UNIFIED ACROSS CRMs
-- =============================================================

CREATE TABLE activities (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    crm_connection_id UUID NOT NULL REFERENCES crm_connections(id) ON DELETE CASCADE,
    external_id     VARCHAR(255) NOT NULL,
    deal_id         UUID REFERENCES deals(id) ON DELETE SET NULL,
    contact_id      UUID REFERENCES contacts(id) ON DELETE SET NULL,
    account_id      UUID REFERENCES accounts(id) ON DELETE SET NULL,

    -- Canonical typed fields
    activity_type   VARCHAR(50) NOT NULL,
    subject         VARCHAR(500),
    activity_date   TIMESTAMPTZ NOT NULL,
    duration_minutes INTEGER,
    direction       VARCHAR(20),
    status          VARCHAR(50),

    -- CRM-specific fields
    crm_fields      JSONB NOT NULL DEFAULT '{}',

    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    synced_at       TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(crm_connection_id, external_id)
);

CREATE INDEX idx_activities_deal ON activities(deal_id);
CREATE INDEX idx_activities_tenant_date ON activities(tenant_id, activity_date DESC);

-- =============================================================
-- DEAL-CONTACT RELATIONSHIPS
-- =============================================================

CREATE TABLE deal_contacts (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    deal_id         UUID NOT NULL REFERENCES deals(id) ON DELETE CASCADE,
    contact_id      UUID NOT NULL REFERENCES contacts(id) ON DELETE CASCADE,
    role            VARCHAR(100),
    is_primary      BOOLEAN DEFAULT FALSE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(deal_id, contact_id)
);

CREATE INDEX idx_deal_contacts_deal ON deal_contacts(deal_id);
```

## Pipeline Configuration

```sql
-- =============================================================
-- PIPELINES & STAGES
-- =============================================================

CREATE TABLE pipelines (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    crm_connection_id UUID REFERENCES crm_connections(id) ON DELETE SET NULL,
    external_id     VARCHAR(255),
    config          JSONB NOT NULL DEFAULT '{}',
    -- config example:
    -- {
    --   "is_default": true,
    --   "forecast_categories": ["pipeline", "best_case", "commit"],
    --   "stage_defaults": {
    --     "Qualification": {"probability": 20, "avg_days": 14},
    --     "Proposal": {"probability": 60, "avg_days": 21}
    --   }
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, name)
);

CREATE TABLE pipeline_stages (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    pipeline_id     UUID NOT NULL REFERENCES pipelines(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    display_order   INTEGER NOT NULL,
    default_probability NUMERIC(5,2),
    forecast_category VARCHAR(50),
    is_closed       BOOLEAN DEFAULT FALSE,
    is_won          BOOLEAN DEFAULT FALSE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(pipeline_id, name)
);
```

## ML Models & Score History

```sql
-- =============================================================
-- ML MODEL REGISTRY
-- =============================================================

CREATE TABLE ml_models (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    model_type      VARCHAR(50) NOT NULL,
    model_version   VARCHAR(50) NOT NULL,
    status          VARCHAR(50) NOT NULL DEFAULT 'training',
    config          JSONB NOT NULL DEFAULT '{}',
    -- config example:
    -- {
    --   "hyperparameters": {"n_estimators": 500, "max_depth": 6, "learning_rate": 0.05},
    --   "feature_names": ["days_in_stage", "activity_count_30d", "contact_count", ...],
    --   "training_deal_count": 2847,
    --   "training_started_at": "2026-05-11T02:00:00Z",
    --   "training_completed_at": "2026-05-11T02:15:00Z",
    --   "metrics": {"auc_roc": 0.87, "accuracy": 0.79, "f1": 0.81, "log_loss": 0.42}
    -- }
    model_artifact_path TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_ml_models_tenant_active ON ml_models(tenant_id, status) WHERE status = 'active';

-- =============================================================
-- SCORE HISTORY (append-only for audit trail)
-- =============================================================

CREATE TABLE score_history (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    deal_id         UUID NOT NULL REFERENCES deals(id) ON DELETE CASCADE,
    tenant_id       UUID NOT NULL,
    model_id        UUID NOT NULL REFERENCES ml_models(id),
    scored_at       TIMESTAMPTZ NOT NULL DEFAULT now(),
    result          JSONB NOT NULL
    -- result example:
    -- {
    --   "win_probability": 0.6823,
    --   "confidence_lower": 0.5901,
    --   "confidence_upper": 0.7745,
    --   "risk_level": "medium",
    --   "score_percentile": 62,
    --   "explanation": "Score at 68%: strong champion engagement offset by 2 close date slips",
    --   "shap_values": {
    --     "days_since_last_activity": -0.0823,
    --     "close_date_slip_count": -0.0654,
    --     "contact_count": 0.0412,
    --     "has_champion": 0.0389,
    --     "amount": 0.0234,
    --     "days_in_stage": -0.0198
    --   },
    --   "top_positive_factors": ["has_champion", "multi_threaded_contacts"],
    --   "top_negative_factors": ["close_date_slipped_2x", "no_exec_sponsor"]
    -- }
);

CREATE INDEX idx_score_history_deal ON score_history(deal_id, scored_at DESC);
CREATE INDEX idx_score_history_tenant ON score_history(tenant_id, scored_at DESC);
```

## Forecasting

```sql
-- =============================================================
-- FORECASTING
-- =============================================================

CREATE TABLE forecast_periods (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    period_type     VARCHAR(20) NOT NULL,
    period_label    VARCHAR(50) NOT NULL,
    fiscal_year     INTEGER NOT NULL,
    fiscal_quarter  INTEGER,
    start_date      DATE NOT NULL,
    end_date        DATE NOT NULL,
    is_current      BOOLEAN DEFAULT FALSE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, period_type, start_date)
);

CREATE TABLE forecast_submissions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    period_id       UUID NOT NULL REFERENCES forecast_periods(id) ON DELETE CASCADE,
    submitted_by    UUID NOT NULL REFERENCES users(id),
    submitted_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    data            JSONB NOT NULL
    -- data example:
    -- {
    --   "category": "commit",
    --   "submitted_amount": 1250000.00,
    --   "currency_code": "USD",
    --   "ai_adjusted_amount": 1105000.00,
    --   "ai_adjustment_reason": "AI reduced commit by $145K due to 4 high-risk deals",
    --   "notes": "Confident on Acme and BigCorp; watching DataFlow closely",
    --   "deal_breakdown": [
    --     {"deal_id": "uuid", "amount": 175000, "category": "commit"},
    --     {"deal_id": "uuid", "amount": 95000, "category": "best_case"}
    --   ],
    --   "pipeline_coverage": 3.2
    -- }
);

CREATE INDEX idx_forecast_submissions_period ON forecast_submissions(period_id, submitted_by);

CREATE TABLE forecast_snapshots (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    period_id       UUID NOT NULL REFERENCES forecast_periods(id) ON DELETE CASCADE,
    snapshot_date   DATE NOT NULL,
    data            JSONB NOT NULL,
    -- data example:
    -- {
    --   "total_pipeline": 4500000,
    --   "total_commit": 1250000,
    --   "total_best_case": 1850000,
    --   "total_most_likely": 1550000,
    --   "ai_predicted": {"total": 1380000, "lower": 1120000, "upper": 1640000},
    --   "deal_count": 47,
    --   "pipeline_coverage": 3.2,
    --   "by_rep": [
    --     {"user_id": "uuid", "name": "Alice", "commit": 350000, "best_case": 450000},
    --     {"user_id": "uuid", "name": "Bob", "commit": 280000, "best_case": 400000}
    --   ],
    --   "by_stage": {
    --     "Qualification": {"count": 15, "amount": 1200000},
    --     "Proposal": {"count": 12, "amount": 1400000}
    --   }
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, period_id, snapshot_date)
);

CREATE INDEX idx_forecast_snapshots_period ON forecast_snapshots(period_id, snapshot_date DESC);
```

## Conversation Intelligence

```sql
-- =============================================================
-- CONVERSATION INTELLIGENCE
-- =============================================================

CREATE TABLE call_recordings (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    deal_id         UUID REFERENCES deals(id) ON DELETE SET NULL,
    recording_url   TEXT NOT NULL,
    duration_seconds INTEGER,
    recorded_at     TIMESTAMPTZ NOT NULL,
    source          VARCHAR(50),
    processing      JSONB NOT NULL DEFAULT '{}',
    -- processing example:
    -- {
    --   "transcription_status": "completed",
    --   "whisper_model": "large-v3",
    --   "language": "en",
    --   "word_count": 4521,
    --   "participants": ["rep@company.com", "buyer@acme.com"],
    --   "speaker_segments": [
    --     {"speaker": "Rep", "start": 0.0, "end": 12.5, "text": "Thanks for joining..."}
    --   ],
    --   "insights": [
    --     {"type": "objection", "summary": "Budget concerns for Q3", "confidence": 0.92},
    --     {"type": "next_step", "summary": "Send revised proposal by Friday", "confidence": 0.95},
    --     {"type": "competitor_mention", "summary": "Buyer compared to Clari pricing", "confidence": 0.88}
    --   ],
    --   "llm_model": "claude-opus-4-20250514",
    --   "processed_at": "2026-05-11T08:15:00Z"
    -- }
    transcript_text TEXT,                     -- Full transcript for search
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_call_recordings_deal ON call_recordings(deal_id);
CREATE INDEX idx_call_recordings_tenant ON call_recordings(tenant_id, recorded_at DESC);

-- Full-text search on transcripts
CREATE INDEX idx_call_recordings_transcript ON call_recordings USING GIN (to_tsvector('english', transcript_text));
```

## Alerts

```sql
-- =============================================================
-- ALERTS
-- =============================================================

CREATE TABLE alerts (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    deal_id         UUID REFERENCES deals(id) ON DELETE CASCADE,
    alert_type      VARCHAR(50) NOT NULL,
    severity        VARCHAR(20) NOT NULL,
    message         TEXT NOT NULL,
    details         JSONB NOT NULL DEFAULT '{}',
    -- details example:
    -- {
    --   "threshold": 14,
    --   "actual_value": 21,
    --   "previous_probability": 0.72,
    --   "current_probability": 0.41,
    --   "channels_delivered": ["slack"],
    --   "slack_message_ts": "1715000000.000100"
    -- }
    is_dismissed    BOOLEAN DEFAULT FALSE,
    dismissed_by    UUID REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_alerts_tenant_active ON alerts(tenant_id, created_at DESC) WHERE NOT is_dismissed;
CREATE INDEX idx_alerts_deal ON alerts(deal_id);
```

## Field Schema Documentation

```sql
-- =============================================================
-- JSONB FIELD SCHEMA DOCUMENTATION
-- =============================================================
-- Documents the expected structure of each JSONB column,
-- enabling validation and API documentation generation.

CREATE TABLE field_schemas (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    table_name      VARCHAR(100) NOT NULL,
    column_name     VARCHAR(100) NOT NULL,
    json_schema     JSONB NOT NULL,           -- JSON Schema 2020-12 definition
    description     TEXT,
    version         INTEGER NOT NULL DEFAULT 1,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(table_name, column_name, version)
);
```

## Example Queries

```sql
-- =============================================================
-- JSONB QUERY EXAMPLES
-- =============================================================

-- Find deals where Salesforce custom field "Custom_ARR__c" exceeds $100K
SELECT id, name, amount, crm_fields->>'Custom_ARR__c' AS arr
FROM deals
WHERE tenant_id = $1
  AND NOT is_closed
  AND (crm_fields->>'Custom_ARR__c')::NUMERIC > 100000;

-- Find deals missing a champion (MEDDIC qualification)
SELECT id, name, amount, qualification->>'completeness_score' AS qual_score
FROM deals
WHERE tenant_id = $1
  AND NOT is_closed
  AND (qualification->>'champion') IS NULL;

-- Get deals with declining scores
SELECT id, name, amount,
       (scoring->>'win_probability')::NUMERIC AS prob,
       scoring->>'explanation' AS explanation,
       scoring->>'trend' AS trend
FROM deals
WHERE tenant_id = $1
  AND NOT is_closed
  AND scoring->>'trend' = 'declining'
ORDER BY (scoring->>'win_probability')::NUMERIC ASC;

-- Multi-CRM unified pipeline view
SELECT d.name, d.amount, d.stage_name, d.close_date,
       c.crm_type,
       (d.scoring->>'win_probability')::NUMERIC AS win_prob,
       (d.quality->>'score')::NUMERIC AS dq_score
FROM deals d
JOIN crm_connections c ON c.id = d.crm_connection_id
WHERE d.tenant_id = $1
  AND NOT d.is_closed
ORDER BY d.close_date;

-- Search call transcripts for competitor mentions
SELECT cr.id, cr.recorded_at, cr.deal_id, d.name AS deal_name,
       cr.processing->'insights' AS insights
FROM call_recordings cr
JOIN deals d ON d.id = cr.deal_id
WHERE cr.tenant_id = $1
  AND to_tsvector('english', cr.transcript_text) @@ to_tsquery('english', 'Clari | Gong | Aviso');
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Tenancy & Authentication | 3 | tenants, users, crm_connections |
| CRM Data Layer | 5 | accounts, contacts, deals, deal_contacts, activities |
| Pipeline Configuration | 2 | pipelines, pipeline_stages |
| ML Scoring | 2 | ml_models, score_history |
| Forecasting | 3 | forecast_periods, forecast_submissions, forecast_snapshots |
| Conversation Intelligence | 1 | call_recordings (insights embedded in JSONB) |
| Alerts | 1 | alerts |
| Schema Documentation | 1 | field_schemas |
| **Total** | **18** | |

---

## Key Design Decisions

1. **Six JSONB columns on the `deals` table, each with a clear purpose.** Rather than one monolithic JSONB column, the deals table has `crm_fields` (CRM-native data), `qualification` (MEDDIC), `scoring` (ML results), `quality` (data quality), `lifecycle` (computed metrics), and `custom_fields` (tenant-defined). This separation enables targeted GIN indexes on each and makes the schema self-documenting.

2. **Latest ML score embedded on the deal, history in `score_history`.** The `scoring` JSONB on deals gives the dashboard a single-table query for the current state. The `score_history` table provides the append-only audit trail. This denormalization eliminates a JOIN on the most frequent read query (deal list with scores).

3. **Call insights embedded in `call_recordings.processing` JSONB.** Rather than a separate `call_insights` table, insights are stored as an array within the recording's processing JSONB. This works well when insights are always read in the context of their recording and reduces table count. The trade-off is that querying "all objections across all deals" requires JSONB array traversal.

4. **`crm_fields` preserves source field names exactly.** Salesforce field `Custom_ARR__c` is stored as `"Custom_ARR__c"` in the JSONB, not renamed. This ensures the data lineage is clear and CRM-specific reports can reference native field names. The canonical typed columns (`amount`, `stage_name`, `close_date`) serve as the common query surface.

5. **Tenant custom fields separated from CRM fields.** `custom_fields` stores tenant-defined fields (configured in `tenants.settings.custom_deal_fields`), while `crm_fields` stores CRM-native fields. This prevents confusion about data provenance — a field in `crm_fields` came from the CRM; a field in `custom_fields` was defined by the tenant in the forecasting engine.

6. **`field_schemas` table documents JSONB structure.** Since JSONB columns lack database-level schema enforcement, the `field_schemas` table stores JSON Schema 2020-12 definitions for each JSONB column. These schemas drive application-level validation and API documentation generation, compensating for the flexibility/safety trade-off.

7. **Forecast snapshots use JSONB for rich, evolving aggregates.** The `forecast_snapshots.data` JSONB stores per-rep and per-stage breakdowns alongside top-level totals. This allows the snapshot structure to evolve (e.g., adding per-product breakdowns) without schema migrations, while the `snapshot_date` typed column enables efficient time-range queries.

8. **18 tables vs. 24 in the normalized model.** The JSONB hybrid consolidates tables where data is always read together (e.g., call insights with recordings, SHAP values with scores). This reduces join complexity for dashboard queries at the cost of JSONB traversal for cross-entity analytics.
