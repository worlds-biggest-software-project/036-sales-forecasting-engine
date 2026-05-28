# Data Model Suggestion 1: Entity-Centric Normalized Relational

> Project: Sales Forecasting Engine · Created: 2026-05-11

## Philosophy

The entity-centric normalized relational approach models every domain concept as a distinct table with explicit foreign key relationships between them. Each CRM source (Salesforce, HubSpot, Pipedrive) is mapped into a canonical internal schema where deals, accounts, contacts, activities, and forecasts each have dedicated tables with well-defined column types and constraints. This produces a highly structured, query-friendly database that enforces referential integrity at the schema level.

This approach mirrors how Salesforce itself structures its standard object model (Opportunity, Account, Contact, Activity) and how HubSpot organizes its Deal, Contact, Company, and Activity objects. By normalizing CRM data into a canonical schema on ingestion, the engine decouples itself from any single CRM's data model while maintaining the relationships that make cross-entity queries possible — for example, "show me all deals owned by reps who have not logged an activity in 14 days."

This is the most conventional architecture for a CRM-adjacent product and is best suited when data integrity is paramount, complex cross-entity queries are frequent, and the team values schema-as-documentation. The trade-off is rigidity: adding CRM-specific fields or supporting a new CRM requires schema migrations.

**Best for:** Teams that want a predictable, well-documented schema with strong referential integrity and SQL-native querying.

**Trade-offs:**
- Pro: Maximum data integrity via foreign keys and constraints
- Pro: Familiar to any developer who knows SQL; easy to reason about
- Pro: Excellent query performance for known access patterns with proper indexing
- Pro: Schema itself documents the domain model
- Con: Adding CRM-specific or jurisdiction-specific fields requires schema migrations
- Con: Multi-CRM field mapping may require "lowest common denominator" canonical fields, losing CRM-specific metadata
- Con: Higher table count increases join complexity for dashboard queries
- Con: Less flexible for rapid prototyping during MVP phase

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| Salesforce Object Model | Canonical deal/account/contact schema mirrors Salesforce Opportunity, Account, Contact, Activity standard objects |
| HubSpot CRM Object Model | Deal, Company, Contact, Activity objects mapped to canonical tables via ingestion pipeline |
| MEDDIC / MEDDPICC | Deal qualification dimensions (Metrics, Economic Buyer, Decision Criteria, etc.) modeled as explicit columns on the deal table |
| OAuth 2.0 (RFC 6749) | CRM connector credentials stored in `crm_connections` table with encrypted token fields |
| ISO/IEC 5259-2:2024 | Data quality scores stored as explicit columns on deal records, aligning with AI data quality measurement standard |
| ASC 606 / GAAP | Forecast periods align with fiscal quarters; `forecast_periods` table uses fiscal year/quarter identifiers |
| OpenTelemetry (OTEL) | ML model inference metrics stored in `model_runs` table for pipeline health monitoring |

---

## Core Identity & Multi-Tenancy

```sql
-- =============================================================
-- TENANCY & AUTHENTICATION
-- =============================================================

CREATE TABLE tenants (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    plan            VARCHAR(50) NOT NULL DEFAULT 'free',  -- free, pro, enterprise
    settings        JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    email           VARCHAR(255) NOT NULL,
    name            VARCHAR(255) NOT NULL,
    role            VARCHAR(50) NOT NULL DEFAULT 'viewer',  -- admin, manager, rep, viewer
    avatar_url      TEXT,
    last_login_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, email)
);

CREATE INDEX idx_users_tenant ON users(tenant_id);

CREATE TABLE crm_connections (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    crm_type        VARCHAR(50) NOT NULL,  -- salesforce, hubspot, pipedrive, dynamics365, zoho
    instance_url    TEXT,                   -- e.g., https://mycompany.my.salesforce.com
    access_token    TEXT NOT NULL,          -- encrypted at rest
    refresh_token   TEXT,                   -- encrypted at rest
    token_expires_at TIMESTAMPTZ,
    scopes          TEXT[],
    sync_status     VARCHAR(50) NOT NULL DEFAULT 'pending',  -- pending, syncing, active, error
    last_sync_at    TIMESTAMPTZ,
    sync_cursor     JSONB,                 -- CRM-specific sync state (e.g., Salesforce replication ID)
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_crm_connections_tenant ON crm_connections(tenant_id);
```

## CRM Canonical Data Layer

```sql
-- =============================================================
-- CANONICAL CRM ENTITIES
-- =============================================================

CREATE TABLE accounts (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    crm_connection_id UUID NOT NULL REFERENCES crm_connections(id) ON DELETE CASCADE,
    external_id     VARCHAR(255) NOT NULL,  -- CRM-native ID (Salesforce 18-char ID, HubSpot numeric ID)
    name            VARCHAR(500) NOT NULL,
    industry        VARCHAR(255),
    employee_count  INTEGER,
    annual_revenue  NUMERIC(18,2),
    website         TEXT,
    billing_country VARCHAR(100),           -- ISO 3166-1 alpha-2
    billing_state   VARCHAR(100),
    account_type    VARCHAR(100),           -- Customer, Prospect, Partner, etc.
    owner_external_id VARCHAR(255),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    synced_at       TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(crm_connection_id, external_id)
);

CREATE INDEX idx_accounts_tenant ON accounts(tenant_id);
CREATE INDEX idx_accounts_name ON accounts(tenant_id, name);
CREATE INDEX idx_accounts_industry ON accounts(tenant_id, industry);

CREATE TABLE contacts (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    crm_connection_id UUID NOT NULL REFERENCES crm_connections(id) ON DELETE CASCADE,
    external_id     VARCHAR(255) NOT NULL,
    account_id      UUID REFERENCES accounts(id) ON DELETE SET NULL,
    first_name      VARCHAR(255),
    last_name       VARCHAR(255),
    email           VARCHAR(255),
    phone           VARCHAR(100),
    title           VARCHAR(255),            -- Job title
    department      VARCHAR(255),
    is_decision_maker BOOLEAN DEFAULT FALSE,
    seniority_level VARCHAR(50),             -- C-level, VP, Director, Manager, Individual Contributor
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    synced_at       TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(crm_connection_id, external_id)
);

CREATE INDEX idx_contacts_tenant ON contacts(tenant_id);
CREATE INDEX idx_contacts_account ON contacts(account_id);
CREATE INDEX idx_contacts_email ON contacts(tenant_id, email);

CREATE TABLE deals (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    crm_connection_id UUID NOT NULL REFERENCES crm_connections(id) ON DELETE CASCADE,
    external_id     VARCHAR(255) NOT NULL,   -- Salesforce Opportunity ID or HubSpot Deal ID
    account_id      UUID REFERENCES accounts(id) ON DELETE SET NULL,
    name            VARCHAR(500) NOT NULL,
    amount          NUMERIC(18,2),
    currency_code   VARCHAR(3) DEFAULT 'USD', -- ISO 4217
    stage_name      VARCHAR(255) NOT NULL,
    pipeline_name   VARCHAR(255),
    probability     NUMERIC(5,2),             -- CRM-native probability (0-100)
    close_date      DATE,
    is_closed       BOOLEAN NOT NULL DEFAULT FALSE,
    is_won          BOOLEAN NOT NULL DEFAULT FALSE,
    deal_type       VARCHAR(100),             -- New Business, Renewal, Expansion, etc.
    lead_source     VARCHAR(255),
    next_step       TEXT,
    description     TEXT,
    owner_external_id VARCHAR(255),
    owner_user_id   UUID REFERENCES users(id) ON DELETE SET NULL,

    -- MEDDIC / MEDDPICC qualification fields
    meddic_metrics          TEXT,             -- Quantified business value
    meddic_economic_buyer   VARCHAR(255),     -- Name/role of economic buyer
    meddic_decision_criteria TEXT,            -- How the buyer will evaluate
    meddic_decision_process TEXT,             -- Steps to close
    meddic_identify_pain    TEXT,             -- Key pain points
    meddic_champion         VARCHAR(255),     -- Internal champion name/role
    meddic_competition      TEXT,             -- Competitive landscape

    -- Data quality signals (computed by engine)
    dq_score                NUMERIC(5,2),     -- 0-100 data quality score
    dq_flags                TEXT[],           -- Array of quality issue codes
    days_in_stage           INTEGER,
    days_since_activity     INTEGER,
    close_date_slip_count   INTEGER DEFAULT 0,

    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    synced_at       TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(crm_connection_id, external_id)
);

CREATE INDEX idx_deals_tenant ON deals(tenant_id);
CREATE INDEX idx_deals_account ON deals(account_id);
CREATE INDEX idx_deals_stage ON deals(tenant_id, stage_name);
CREATE INDEX idx_deals_close_date ON deals(tenant_id, close_date);
CREATE INDEX idx_deals_is_open ON deals(tenant_id, is_closed) WHERE NOT is_closed;
CREATE INDEX idx_deals_pipeline ON deals(tenant_id, pipeline_name);

CREATE TABLE deal_contacts (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    deal_id         UUID NOT NULL REFERENCES deals(id) ON DELETE CASCADE,
    contact_id      UUID NOT NULL REFERENCES contacts(id) ON DELETE CASCADE,
    role            VARCHAR(100),             -- Decision Maker, Influencer, Champion, Blocker, End User
    is_primary      BOOLEAN DEFAULT FALSE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(deal_id, contact_id)
);

CREATE INDEX idx_deal_contacts_deal ON deal_contacts(deal_id);
CREATE INDEX idx_deal_contacts_contact ON deal_contacts(contact_id);

CREATE TABLE activities (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    crm_connection_id UUID NOT NULL REFERENCES crm_connections(id) ON DELETE CASCADE,
    external_id     VARCHAR(255) NOT NULL,
    deal_id         UUID REFERENCES deals(id) ON DELETE SET NULL,
    contact_id      UUID REFERENCES contacts(id) ON DELETE SET NULL,
    account_id      UUID REFERENCES accounts(id) ON DELETE SET NULL,
    activity_type   VARCHAR(50) NOT NULL,     -- call, email, meeting, task, note
    subject         VARCHAR(500),
    description     TEXT,
    activity_date   TIMESTAMPTZ NOT NULL,
    duration_minutes INTEGER,
    direction       VARCHAR(20),              -- inbound, outbound (for calls/emails)
    status          VARCHAR(50),              -- completed, scheduled, canceled
    owner_external_id VARCHAR(255),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    synced_at       TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(crm_connection_id, external_id)
);

CREATE INDEX idx_activities_deal ON activities(deal_id);
CREATE INDEX idx_activities_tenant_date ON activities(tenant_id, activity_date DESC);
CREATE INDEX idx_activities_type ON activities(tenant_id, activity_type);
```

## Pipeline & Stage Configuration

```sql
-- =============================================================
-- PIPELINE CONFIGURATION
-- =============================================================

CREATE TABLE pipelines (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    crm_connection_id UUID REFERENCES crm_connections(id) ON DELETE SET NULL,
    external_id     VARCHAR(255),
    is_default      BOOLEAN DEFAULT FALSE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, name)
);

CREATE TABLE pipeline_stages (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    pipeline_id     UUID NOT NULL REFERENCES pipelines(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    display_order   INTEGER NOT NULL,
    default_probability NUMERIC(5,2),         -- 0-100 default win probability for this stage
    forecast_category VARCHAR(50),            -- pipeline, best_case, commit, closed_won, closed_lost, omit
    is_closed       BOOLEAN DEFAULT FALSE,
    is_won          BOOLEAN DEFAULT FALSE,
    avg_days_in_stage NUMERIC(8,2),           -- Computed: historical average time deals spend here
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(pipeline_id, name)
);

CREATE INDEX idx_pipeline_stages_pipeline ON pipeline_stages(pipeline_id, display_order);
```

## ML Scoring & Predictions

```sql
-- =============================================================
-- ML MODEL MANAGEMENT & DEAL SCORING
-- =============================================================

CREATE TABLE ml_models (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    model_type      VARCHAR(50) NOT NULL,     -- xgboost, lightgbm, prophet, llm_hybrid
    model_version   VARCHAR(50) NOT NULL,
    status          VARCHAR(50) NOT NULL DEFAULT 'training',  -- training, active, retired, failed
    training_started_at TIMESTAMPTZ,
    training_completed_at TIMESTAMPTZ,
    training_deal_count INTEGER,
    feature_names   TEXT[] NOT NULL,           -- Ordered list of features used
    hyperparameters JSONB,
    metrics         JSONB,                    -- { "auc_roc": 0.87, "accuracy": 0.79, "f1": 0.81 }
    model_artifact_path TEXT,                 -- S3/GCS path to serialized model
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_ml_models_tenant_active ON ml_models(tenant_id, status) WHERE status = 'active';

CREATE TABLE deal_scores (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    deal_id         UUID NOT NULL REFERENCES deals(id) ON DELETE CASCADE,
    model_id        UUID NOT NULL REFERENCES ml_models(id) ON DELETE CASCADE,
    win_probability NUMERIC(5,4) NOT NULL,    -- 0.0000 to 1.0000
    score_percentile NUMERIC(5,2),            -- How this deal compares to historical deals at same stage (0-100)
    confidence_lower NUMERIC(5,4),            -- Lower bound of prediction interval
    confidence_upper NUMERIC(5,4),            -- Upper bound of prediction interval
    risk_level      VARCHAR(20) NOT NULL,     -- low, medium, high, critical

    -- Top contributing SHAP features (denormalized for query speed)
    top_positive_factors TEXT[],              -- ['strong_champion_engagement', 'multi_threaded_3_contacts']
    top_negative_factors TEXT[],              -- ['close_date_slipped_3x', 'no_activity_21_days']
    explanation_text TEXT,                    -- "Probability dropped from 72% to 41% because..."

    scored_at       TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_deal_scores_deal ON deal_scores(deal_id, scored_at DESC);
CREATE INDEX idx_deal_scores_model ON deal_scores(model_id);
CREATE INDEX idx_deal_scores_risk ON deal_scores(risk_level) WHERE risk_level IN ('high', 'critical');

CREATE TABLE deal_score_features (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    deal_score_id   UUID NOT NULL REFERENCES deal_scores(id) ON DELETE CASCADE,
    feature_name    VARCHAR(255) NOT NULL,
    feature_value   NUMERIC(18,6),
    shap_value      NUMERIC(18,6) NOT NULL,   -- Positive = pushes toward win; negative = pushes toward loss
    feature_rank    INTEGER NOT NULL           -- 1 = most impactful feature
);

CREATE INDEX idx_deal_score_features_score ON deal_score_features(deal_score_id, feature_rank);
```

## Forecasting

```sql
-- =============================================================
-- FORECASTING & SUBMISSIONS
-- =============================================================

CREATE TABLE forecast_periods (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    period_type     VARCHAR(20) NOT NULL,     -- weekly, monthly, quarterly
    period_label    VARCHAR(50) NOT NULL,     -- 'Q2 2026', 'W19 2026', '2026-05'
    fiscal_year     INTEGER NOT NULL,
    fiscal_quarter  INTEGER,                  -- 1-4 (for quarterly periods)
    start_date      DATE NOT NULL,
    end_date        DATE NOT NULL,
    is_current      BOOLEAN DEFAULT FALSE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, period_type, start_date)
);

CREATE INDEX idx_forecast_periods_current ON forecast_periods(tenant_id, is_current) WHERE is_current;

CREATE TABLE forecast_submissions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    period_id       UUID NOT NULL REFERENCES forecast_periods(id) ON DELETE CASCADE,
    submitted_by    UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    forecast_category VARCHAR(50) NOT NULL,   -- commit, best_case, most_likely, pipeline
    submitted_amount NUMERIC(18,2) NOT NULL,
    currency_code   VARCHAR(3) DEFAULT 'USD',
    ai_adjusted_amount NUMERIC(18,2),         -- ML-adjusted amount
    ai_adjustment_reason TEXT,                -- "AI reduced commit by $45K due to 3 high-risk deals"
    notes           TEXT,
    submitted_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_forecast_submissions_period ON forecast_submissions(period_id, submitted_by);
CREATE INDEX idx_forecast_submissions_tenant ON forecast_submissions(tenant_id, submitted_at DESC);

CREATE TABLE forecast_snapshots (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    period_id       UUID NOT NULL REFERENCES forecast_periods(id) ON DELETE CASCADE,
    snapshot_date   DATE NOT NULL,
    total_pipeline  NUMERIC(18,2),
    total_commit    NUMERIC(18,2),
    total_best_case NUMERIC(18,2),
    total_most_likely NUMERIC(18,2),
    ai_predicted_total NUMERIC(18,2),
    ai_predicted_lower NUMERIC(18,2),
    ai_predicted_upper NUMERIC(18,2),
    deal_count      INTEGER,
    pipeline_coverage NUMERIC(5,2),           -- pipeline / quota ratio
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, period_id, snapshot_date)
);

CREATE INDEX idx_forecast_snapshots_period ON forecast_snapshots(period_id, snapshot_date DESC);

CREATE TABLE forecast_accuracy (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    period_id       UUID NOT NULL REFERENCES forecast_periods(id) ON DELETE CASCADE,
    user_id         UUID REFERENCES users(id) ON DELETE SET NULL,
    submitted_amount NUMERIC(18,2),
    ai_predicted_amount NUMERIC(18,2),
    actual_amount   NUMERIC(18,2),
    submitted_error_pct NUMERIC(8,4),         -- (submitted - actual) / actual
    ai_error_pct    NUMERIC(8,4),             -- (ai_predicted - actual) / actual
    computed_at     TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_forecast_accuracy_period ON forecast_accuracy(period_id);
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
    participants    TEXT[],                   -- Email addresses of participants
    source          VARCHAR(50),             -- zoom, google_meet, teams, manual_upload
    transcription_status VARCHAR(50) DEFAULT 'pending',  -- pending, processing, completed, failed
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_call_recordings_deal ON call_recordings(deal_id);
CREATE INDEX idx_call_recordings_status ON call_recordings(transcription_status) WHERE transcription_status = 'pending';

CREATE TABLE call_transcripts (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    recording_id    UUID NOT NULL REFERENCES call_recordings(id) ON DELETE CASCADE,
    full_text       TEXT NOT NULL,
    speaker_segments JSONB,                  -- [{"speaker": "Rep", "start": 0.0, "end": 12.5, "text": "..."}]
    language        VARCHAR(10) DEFAULT 'en',
    whisper_model   VARCHAR(50),             -- model used for transcription
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE call_insights (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    recording_id    UUID NOT NULL REFERENCES call_recordings(id) ON DELETE CASCADE,
    deal_id         UUID REFERENCES deals(id) ON DELETE SET NULL,
    insight_type    VARCHAR(50) NOT NULL,     -- objection, next_step, competitor_mention, pricing_discussion,
                                              -- champion_signal, risk_signal, positive_signal
    summary         TEXT NOT NULL,
    confidence      NUMERIC(5,4),             -- 0-1 confidence of LLM extraction
    raw_excerpt     TEXT,                      -- Relevant transcript excerpt
    llm_model       VARCHAR(100),             -- Model used for extraction
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_call_insights_deal ON call_insights(deal_id);
CREATE INDEX idx_call_insights_type ON call_insights(insight_type);
```

## Alerts & Notifications

```sql
-- =============================================================
-- ALERTS & NOTIFICATIONS
-- =============================================================

CREATE TABLE alert_rules (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    rule_type       VARCHAR(50) NOT NULL,     -- no_activity, probability_drop, close_date_slip,
                                              -- stage_stagnation, dq_score_low
    threshold_value NUMERIC(18,4),            -- e.g., 14 (days), 15 (percentage points)
    channels        TEXT[] NOT NULL,           -- ['slack', 'email']
    is_active       BOOLEAN DEFAULT TRUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE alerts (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    rule_id         UUID NOT NULL REFERENCES alert_rules(id) ON DELETE CASCADE,
    deal_id         UUID NOT NULL REFERENCES deals(id) ON DELETE CASCADE,
    alert_type      VARCHAR(50) NOT NULL,
    message         TEXT NOT NULL,
    severity        VARCHAR(20) NOT NULL,     -- info, warning, critical
    is_dismissed    BOOLEAN DEFAULT FALSE,
    dismissed_by    UUID REFERENCES users(id),
    delivered_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_alerts_tenant_active ON alerts(tenant_id, created_at DESC) WHERE NOT is_dismissed;
CREATE INDEX idx_alerts_deal ON alerts(deal_id);
```

## What-If Scenarios

```sql
-- =============================================================
-- SCENARIO MODELING
-- =============================================================

CREATE TABLE scenarios (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    period_id       UUID NOT NULL REFERENCES forecast_periods(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    created_by      UUID NOT NULL REFERENCES users(id),
    is_baseline     BOOLEAN DEFAULT FALSE,
    total_forecast  NUMERIC(18,2),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE scenario_adjustments (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    scenario_id     UUID NOT NULL REFERENCES scenarios(id) ON DELETE CASCADE,
    deal_id         UUID REFERENCES deals(id) ON DELETE CASCADE,
    adjustment_type VARCHAR(50) NOT NULL,     -- amount_override, probability_override, close_date_override, exclude
    original_value  TEXT,
    adjusted_value  TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_scenario_adjustments_scenario ON scenario_adjustments(scenario_id);
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Tenancy & Authentication | 3 | tenants, users, crm_connections |
| Canonical CRM Entities | 5 | accounts, contacts, deals, deal_contacts, activities |
| Pipeline Configuration | 2 | pipelines, pipeline_stages |
| ML Scoring & Predictions | 3 | ml_models, deal_scores, deal_score_features |
| Forecasting | 4 | forecast_periods, forecast_submissions, forecast_snapshots, forecast_accuracy |
| Conversation Intelligence | 3 | call_recordings, call_transcripts, call_insights |
| Alerts & Notifications | 2 | alert_rules, alerts |
| Scenario Modeling | 2 | scenarios, scenario_adjustments |
| **Total** | **24** | |

---

## Key Design Decisions

1. **Canonical CRM schema over CRM-native mirroring.** Salesforce Opportunity fields and HubSpot Deal properties are mapped into a single `deals` table with common columns. This means CRM-specific fields are either mapped to canonical columns (e.g., Salesforce `StageName` and HubSpot `dealstage` both map to `stage_name`) or dropped. This simplifies multi-CRM queries but requires careful mapping logic in the ingestion layer.

2. **MEDDIC fields as explicit columns on `deals`.** Rather than storing qualification data in a JSONB column or a separate key-value table, each MEDDIC dimension has its own column. This makes qualification completeness scoring a simple `COUNT(NULLIF(...))` query and enables direct filtering ("show me deals missing a champion").

3. **SHAP values stored in a child table.** `deal_score_features` stores per-feature SHAP values for each scoring event, enabling both real-time explanation generation and retrospective model analysis. The top positive/negative factors are denormalized onto `deal_scores` for dashboard query speed.

4. **Forecast snapshots for accuracy tracking.** `forecast_snapshots` captures a daily snapshot of aggregate forecast numbers, enabling the historical forecast accuracy dashboard that compares submissions-over-time against actuals. This supports ASC 606 / GAAP quarterly alignment.

5. **Separate `forecast_accuracy` table.** Rather than computing accuracy on-the-fly from snapshots, pre-computed accuracy metrics per user per period allow rapid display of "who forecasts accurately" leaderboards.

6. **Multi-CRM support via `crm_connection_id` foreign keys.** Every CRM entity (account, contact, deal, activity) references the specific CRM connection it came from, enabling a single tenant to have both Salesforce and HubSpot data coexisting. The `external_id + crm_connection_id` unique constraint prevents duplicate ingestion.

7. **Call recordings and insights as first-class entities.** Conversation intelligence is modeled with three separate tables (recordings, transcripts, insights) rather than a single denormalized table, enabling independent processing pipelines (upload, transcribe, extract insights) and allowing insights to reference both a recording and a deal.

8. **Alert rules as configurable entities.** Rather than hard-coding alert thresholds, `alert_rules` allows per-tenant configuration of alert types and thresholds, supporting the "no activity in 14 days" and "probability dropped >15 points" patterns described in the feature spec.
