# Data Model Suggestion 4: Analytics-First Star Schema with Operational Layer

> Project: Sales Forecasting Engine · Created: 2026-05-11

## Philosophy

The analytics-first approach separates the data model into two distinct layers: an **operational layer** (OLTP) that handles CRM sync, deal management, and ML scoring, and an **analytical layer** (OLAP) built as a star schema optimized for the dashboard and forecasting queries that dominate the engine's read workload. The analytical layer uses fact tables and dimension tables following Kimball dimensional modeling principles, with materialized views bridging the two layers.

Sales forecasting is fundamentally an analytics problem. The most critical queries are aggregations: "What is the total pipeline by stage?", "What is the forecast accuracy over the last 4 quarters?", "How does this rep's win rate compare to the team average?" These queries perform poorly on normalized OLTP schemas because they require multi-table joins across deals, accounts, users, stages, and time periods. A star schema pre-joins these dimensions into wide, denormalized fact tables that enable single-scan aggregations.

This architecture is how mature enterprise BI systems (Tableau, Looker, dbt-driven data warehouses) model sales data. PostgreSQL supports it efficiently via table partitioning (partition fact tables by date), materialized views (pre-compute expensive aggregations), and parallel query execution. The trade-off is maintaining the ETL pipeline between the operational and analytical layers, which adds engineering complexity but provides clean separation of concerns between transactional writes and analytical reads.

**Best for:** Organizations that prioritize dashboard performance, forecast accuracy analytics, and BI tool integration over schema simplicity.

**Trade-offs:**
- Pro: Fastest possible dashboard query performance — aggregations scan a single fact table
- Pro: Clean separation between write-path (OLTP) and read-path (OLAP) concerns
- Pro: Standard dimensional modeling patterns; compatible with any BI tool (Metabase, Superset, Tableau)
- Pro: Partitioned fact tables scale to millions of deal snapshots without query degradation
- Pro: Materialized views pre-compute expensive aggregations for sub-second dashboard loads
- Con: Two-layer architecture doubles the schema surface area and requires ETL maintenance
- Con: Analytical layer has data latency (minutes to hours depending on refresh frequency)
- Con: More complex to understand for developers unfamiliar with dimensional modeling
- Con: Storage cost is higher due to denormalization in fact tables
- Con: Schema changes require updates to both operational tables and fact/dimension tables

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| Salesforce Object Model | Operational layer maps Salesforce objects to canonical tables; analytical layer denormalizes into facts |
| HubSpot CRM Object Model | Same canonical mapping; CRM source tracked as a dimension attribute |
| MEDDIC / MEDDPICC | Qualification completeness modeled as a measurable metric in `fact_deal_snapshots` |
| ISO/IEC 5259-2:2024 | Data quality score as a fact table measure, enabling quality trend analysis over time |
| ASC 606 / GAAP | `dim_fiscal_period` dimension table models fiscal years, quarters, and months for GAAP-aligned reporting |
| ISO 4217 | Currency dimension with exchange rates for multi-currency pipeline consolidation |
| ISO 3166-1 | Geography dimension with country/region hierarchy for territory-based roll-ups |
| Kimball Dimensional Modeling | Star schema follows Kimball's bus architecture: conformed dimensions shared across fact tables |

---

## Operational Layer (OLTP)

```sql
-- =============================================================
-- OPERATIONAL LAYER — TRANSACTIONAL TABLES FOR CRM SYNC & SCORING
-- =============================================================
-- These tables handle writes: CRM data ingestion, ML model runs,
-- forecast submissions, and alert generation. They are normalized
-- for write performance and data integrity.

CREATE TABLE op_tenants (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    plan            VARCHAR(50) NOT NULL DEFAULT 'free',
    settings        JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE op_users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES op_tenants(id) ON DELETE CASCADE,
    email           VARCHAR(255) NOT NULL,
    name            VARCHAR(255) NOT NULL,
    role            VARCHAR(50) NOT NULL DEFAULT 'viewer',
    team            VARCHAR(255),
    manager_id      UUID REFERENCES op_users(id),
    quota           NUMERIC(18,2),
    last_login_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, email)
);

CREATE INDEX idx_op_users_tenant ON op_users(tenant_id);
CREATE INDEX idx_op_users_manager ON op_users(manager_id);

CREATE TABLE op_crm_connections (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES op_tenants(id) ON DELETE CASCADE,
    crm_type        VARCHAR(50) NOT NULL,
    instance_url    TEXT,
    access_token    TEXT NOT NULL,
    refresh_token   TEXT,
    token_expires_at TIMESTAMPTZ,
    sync_status     VARCHAR(50) NOT NULL DEFAULT 'pending',
    last_sync_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE op_accounts (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES op_tenants(id) ON DELETE CASCADE,
    crm_connection_id UUID NOT NULL REFERENCES op_crm_connections(id) ON DELETE CASCADE,
    external_id     VARCHAR(255) NOT NULL,
    name            VARCHAR(500) NOT NULL,
    industry        VARCHAR(255),
    employee_count  INTEGER,
    annual_revenue  NUMERIC(18,2),
    country_code    VARCHAR(2),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(crm_connection_id, external_id)
);

CREATE TABLE op_contacts (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES op_tenants(id) ON DELETE CASCADE,
    crm_connection_id UUID NOT NULL REFERENCES op_crm_connections(id) ON DELETE CASCADE,
    external_id     VARCHAR(255) NOT NULL,
    account_id      UUID REFERENCES op_accounts(id) ON DELETE SET NULL,
    first_name      VARCHAR(255),
    last_name       VARCHAR(255),
    email           VARCHAR(255),
    title           VARCHAR(255),
    seniority_level VARCHAR(50),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(crm_connection_id, external_id)
);

CREATE TABLE op_deals (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES op_tenants(id) ON DELETE CASCADE,
    crm_connection_id UUID NOT NULL REFERENCES op_crm_connections(id) ON DELETE CASCADE,
    external_id     VARCHAR(255) NOT NULL,
    account_id      UUID REFERENCES op_accounts(id) ON DELETE SET NULL,
    owner_user_id   UUID REFERENCES op_users(id) ON DELETE SET NULL,
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
    crm_fields      JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(crm_connection_id, external_id)
);

CREATE INDEX idx_op_deals_tenant ON op_deals(tenant_id);
CREATE INDEX idx_op_deals_open ON op_deals(tenant_id, is_closed) WHERE NOT is_closed;

CREATE TABLE op_activities (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES op_tenants(id) ON DELETE CASCADE,
    crm_connection_id UUID NOT NULL REFERENCES op_crm_connections(id) ON DELETE CASCADE,
    external_id     VARCHAR(255) NOT NULL,
    deal_id         UUID REFERENCES op_deals(id) ON DELETE SET NULL,
    contact_id      UUID REFERENCES op_contacts(id) ON DELETE SET NULL,
    activity_type   VARCHAR(50) NOT NULL,
    subject         VARCHAR(500),
    activity_date   TIMESTAMPTZ NOT NULL,
    duration_minutes INTEGER,
    direction       VARCHAR(20),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(crm_connection_id, external_id)
);

CREATE INDEX idx_op_activities_deal ON op_activities(deal_id);

CREATE TABLE op_deal_scores (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    deal_id         UUID NOT NULL REFERENCES op_deals(id) ON DELETE CASCADE,
    model_id        UUID NOT NULL,
    win_probability NUMERIC(5,4) NOT NULL,
    confidence_lower NUMERIC(5,4),
    confidence_upper NUMERIC(5,4),
    risk_level      VARCHAR(20) NOT NULL,
    explanation     TEXT,
    shap_values     JSONB,
    scored_at       TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_op_deal_scores_deal ON op_deal_scores(deal_id, scored_at DESC);

CREATE TABLE op_forecast_submissions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES op_tenants(id) ON DELETE CASCADE,
    period_label    VARCHAR(50) NOT NULL,
    fiscal_year     INTEGER NOT NULL,
    fiscal_quarter  INTEGER,
    submitted_by    UUID NOT NULL REFERENCES op_users(id),
    forecast_category VARCHAR(50) NOT NULL,
    submitted_amount NUMERIC(18,2) NOT NULL,
    ai_adjusted_amount NUMERIC(18,2),
    notes           TEXT,
    submitted_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

## Analytical Layer — Dimension Tables

```sql
-- =============================================================
-- ANALYTICAL LAYER — DIMENSION TABLES (SLOWLY CHANGING)
-- =============================================================
-- Conformed dimensions shared across all fact tables.
-- Updated via ETL from operational layer.

CREATE TABLE dim_date (
    date_key        INTEGER PRIMARY KEY,      -- YYYYMMDD integer for fast joins
    full_date       DATE NOT NULL UNIQUE,
    day_of_week     SMALLINT NOT NULL,        -- 1=Monday, 7=Sunday
    day_name        VARCHAR(10) NOT NULL,
    day_of_month    SMALLINT NOT NULL,
    day_of_year     SMALLINT NOT NULL,
    week_of_year    SMALLINT NOT NULL,
    month_number    SMALLINT NOT NULL,
    month_name      VARCHAR(10) NOT NULL,
    quarter_number  SMALLINT NOT NULL,
    year_number     INTEGER NOT NULL,
    is_weekend      BOOLEAN NOT NULL,
    is_business_day BOOLEAN NOT NULL
);

-- Pre-populate for 10 years (2020-2030)
-- INSERT INTO dim_date SELECT ... FROM generate_series('2020-01-01'::DATE, '2030-12-31'::DATE, '1 day');

CREATE TABLE dim_fiscal_period (
    period_key      SERIAL PRIMARY KEY,
    tenant_id       UUID NOT NULL,
    period_type     VARCHAR(20) NOT NULL,     -- weekly, monthly, quarterly, annual
    period_label    VARCHAR(50) NOT NULL,     -- 'Q2 2026', 'W19 2026', '2026-05', 'FY2026'
    fiscal_year     INTEGER NOT NULL,
    fiscal_quarter  SMALLINT,
    fiscal_month    SMALLINT,
    start_date      DATE NOT NULL,
    end_date        DATE NOT NULL,
    is_current      BOOLEAN DEFAULT FALSE,
    days_remaining  INTEGER,
    UNIQUE(tenant_id, period_type, start_date)
);

CREATE INDEX idx_dim_fiscal_period_tenant ON dim_fiscal_period(tenant_id, fiscal_year, fiscal_quarter);

CREATE TABLE dim_deal (
    deal_key        SERIAL PRIMARY KEY,       -- Surrogate key for fact table FK
    deal_id         UUID NOT NULL,            -- Natural key (op_deals.id)
    tenant_id       UUID NOT NULL,
    external_id     VARCHAR(255),
    name            VARCHAR(500) NOT NULL,
    deal_type       VARCHAR(100),
    lead_source     VARCHAR(255),
    pipeline_name   VARCHAR(255),
    crm_type        VARCHAR(50),
    -- SCD Type 2 fields
    effective_from  DATE NOT NULL,
    effective_to    DATE,                     -- NULL = current version
    is_current      BOOLEAN NOT NULL DEFAULT TRUE,
    UNIQUE(deal_id, effective_from)
);

CREATE INDEX idx_dim_deal_current ON dim_deal(deal_id) WHERE is_current;
CREATE INDEX idx_dim_deal_tenant ON dim_deal(tenant_id);

CREATE TABLE dim_account (
    account_key     SERIAL PRIMARY KEY,
    account_id      UUID NOT NULL,
    tenant_id       UUID NOT NULL,
    name            VARCHAR(500) NOT NULL,
    industry        VARCHAR(255),
    employee_count_band VARCHAR(50),          -- '1-50', '51-200', '201-1000', '1000+'
    annual_revenue_band VARCHAR(50),          -- '<$1M', '$1M-$10M', '$10M-$100M', '$100M+'
    country_code    VARCHAR(2),
    region          VARCHAR(100),
    crm_type        VARCHAR(50),
    effective_from  DATE NOT NULL,
    effective_to    DATE,
    is_current      BOOLEAN NOT NULL DEFAULT TRUE,
    UNIQUE(account_id, effective_from)
);

CREATE INDEX idx_dim_account_current ON dim_account(account_id) WHERE is_current;

CREATE TABLE dim_user (
    user_key        SERIAL PRIMARY KEY,
    user_id         UUID NOT NULL,
    tenant_id       UUID NOT NULL,
    name            VARCHAR(255) NOT NULL,
    email           VARCHAR(255),
    role            VARCHAR(50),
    team            VARCHAR(255),
    manager_name    VARCHAR(255),
    quota           NUMERIC(18,2),
    effective_from  DATE NOT NULL,
    effective_to    DATE,
    is_current      BOOLEAN NOT NULL DEFAULT TRUE,
    UNIQUE(user_id, effective_from)
);

CREATE INDEX idx_dim_user_current ON dim_user(user_id) WHERE is_current;

CREATE TABLE dim_stage (
    stage_key       SERIAL PRIMARY KEY,
    tenant_id       UUID NOT NULL,
    pipeline_name   VARCHAR(255),
    stage_name      VARCHAR(255) NOT NULL,
    display_order   INTEGER,
    default_probability NUMERIC(5,2),
    forecast_category VARCHAR(50),
    is_closed       BOOLEAN DEFAULT FALSE,
    is_won          BOOLEAN DEFAULT FALSE,
    UNIQUE(tenant_id, pipeline_name, stage_name)
);

CREATE TABLE dim_crm_source (
    crm_source_key  SERIAL PRIMARY KEY,
    crm_type        VARCHAR(50) NOT NULL,     -- salesforce, hubspot, pipedrive, dynamics365
    instance_url    TEXT,
    tenant_id       UUID NOT NULL,
    crm_connection_id UUID NOT NULL UNIQUE
);

CREATE TABLE dim_risk_level (
    risk_key        SERIAL PRIMARY KEY,
    risk_level      VARCHAR(20) NOT NULL UNIQUE,  -- low, medium, high, critical
    display_order   INTEGER NOT NULL,
    color_code      VARCHAR(7)                     -- #00FF00, #FFAA00, #FF5500, #FF0000
);

INSERT INTO dim_risk_level (risk_level, display_order, color_code) VALUES
    ('low', 1, '#22C55E'),
    ('medium', 2, '#F59E0B'),
    ('high', 3, '#EF4444'),
    ('critical', 4, '#DC2626');
```

## Analytical Layer — Fact Tables

```sql
-- =============================================================
-- FACT TABLES — THE ANALYTICAL CORE
-- =============================================================

-- ---------------------------------------------------------------
-- FACT: Deal Daily Snapshots
-- Grain: one row per deal per day
-- This is the primary fact table for pipeline analysis.
-- Partitioned by snapshot_date_key for efficient time-range scans.
-- ---------------------------------------------------------------
CREATE TABLE fact_deal_snapshots (
    snapshot_id     BIGSERIAL,
    snapshot_date_key INTEGER NOT NULL,        -- FK to dim_date
    deal_key        INTEGER NOT NULL,          -- FK to dim_deal
    account_key     INTEGER,                   -- FK to dim_account
    owner_user_key  INTEGER,                   -- FK to dim_user
    stage_key       INTEGER NOT NULL,          -- FK to dim_stage
    crm_source_key  INTEGER,                   -- FK to dim_crm_source
    risk_key        INTEGER,                   -- FK to dim_risk_level
    tenant_id       UUID NOT NULL,

    -- Measures
    amount          NUMERIC(18,2),
    amount_usd      NUMERIC(18,2),             -- Converted to USD for cross-currency roll-ups
    crm_probability NUMERIC(5,2),              -- CRM-native probability
    ml_win_probability NUMERIC(5,4),           -- ML-predicted probability
    ml_confidence_lower NUMERIC(5,4),
    ml_confidence_upper NUMERIC(5,4),
    weighted_amount NUMERIC(18,2),             -- amount * ml_win_probability
    dq_score        NUMERIC(5,2),
    meddic_completeness NUMERIC(5,2),          -- 0-100 qualification completeness

    -- Lifecycle measures
    days_in_pipeline INTEGER,
    days_in_current_stage INTEGER,
    days_since_last_activity INTEGER,
    close_date_slip_count INTEGER,
    stage_change_count INTEGER,
    activity_count_30d INTEGER,
    contact_count   INTEGER,
    call_count_30d  INTEGER,
    email_count_30d INTEGER,

    -- Deal status flags
    is_closed       BOOLEAN NOT NULL,
    is_won          BOOLEAN NOT NULL,
    close_date      DATE,
    forecast_category VARCHAR(50),

    PRIMARY KEY (snapshot_id, snapshot_date_key)
) PARTITION BY RANGE (snapshot_date_key);

-- Create monthly partitions (example for 2026)
CREATE TABLE fact_deal_snapshots_202601 PARTITION OF fact_deal_snapshots
    FOR VALUES FROM (20260101) TO (20260201);
CREATE TABLE fact_deal_snapshots_202602 PARTITION OF fact_deal_snapshots
    FOR VALUES FROM (20260201) TO (20260301);
CREATE TABLE fact_deal_snapshots_202603 PARTITION OF fact_deal_snapshots
    FOR VALUES FROM (20260301) TO (20260401);
CREATE TABLE fact_deal_snapshots_202604 PARTITION OF fact_deal_snapshots
    FOR VALUES FROM (20260401) TO (20260501);
CREATE TABLE fact_deal_snapshots_202605 PARTITION OF fact_deal_snapshots
    FOR VALUES FROM (20260501) TO (20260601);
CREATE TABLE fact_deal_snapshots_202606 PARTITION OF fact_deal_snapshots
    FOR VALUES FROM (20260601) TO (20260701);
-- (Continue for remaining months)

CREATE INDEX idx_fact_deal_snap_tenant_date ON fact_deal_snapshots(tenant_id, snapshot_date_key);
CREATE INDEX idx_fact_deal_snap_deal ON fact_deal_snapshots(deal_key, snapshot_date_key);
CREATE INDEX idx_fact_deal_snap_owner ON fact_deal_snapshots(owner_user_key, snapshot_date_key);
CREATE INDEX idx_fact_deal_snap_stage ON fact_deal_snapshots(stage_key, snapshot_date_key);

-- ---------------------------------------------------------------
-- FACT: Forecast Submissions
-- Grain: one row per forecast submission event
-- ---------------------------------------------------------------
CREATE TABLE fact_forecast_submissions (
    submission_id   BIGSERIAL PRIMARY KEY,
    submission_date_key INTEGER NOT NULL,      -- FK to dim_date
    period_key      INTEGER NOT NULL,          -- FK to dim_fiscal_period
    user_key        INTEGER NOT NULL,          -- FK to dim_user
    tenant_id       UUID NOT NULL,

    -- Measures
    forecast_category VARCHAR(50) NOT NULL,
    submitted_amount NUMERIC(18,2) NOT NULL,
    ai_adjusted_amount NUMERIC(18,2),
    adjustment_delta NUMERIC(18,2),            -- ai_adjusted - submitted
    submitted_at    TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_fact_forecast_sub_period ON fact_forecast_submissions(period_key, user_key);
CREATE INDEX idx_fact_forecast_sub_tenant ON fact_forecast_submissions(tenant_id, submission_date_key);

-- ---------------------------------------------------------------
-- FACT: Forecast Accuracy
-- Grain: one row per user per fiscal period (after period closes)
-- ---------------------------------------------------------------
CREATE TABLE fact_forecast_accuracy (
    accuracy_id     BIGSERIAL PRIMARY KEY,
    period_key      INTEGER NOT NULL,          -- FK to dim_fiscal_period
    user_key        INTEGER NOT NULL,          -- FK to dim_user
    tenant_id       UUID NOT NULL,

    -- Measures
    final_submitted_amount NUMERIC(18,2),      -- Last submission before period close
    ai_predicted_amount NUMERIC(18,2),         -- AI prediction at period close
    actual_amount   NUMERIC(18,2),             -- Actual closed-won revenue
    submitted_error_pct NUMERIC(8,4),          -- (submitted - actual) / actual
    ai_error_pct    NUMERIC(8,4),              -- (ai - actual) / actual
    submitted_abs_error NUMERIC(18,2),
    ai_abs_error    NUMERIC(18,2),
    ai_beat_human   BOOLEAN,                   -- TRUE if AI was more accurate
    computed_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(period_key, user_key)
);

CREATE INDEX idx_fact_accuracy_tenant ON fact_forecast_accuracy(tenant_id);

-- ---------------------------------------------------------------
-- FACT: Deal Stage Transitions
-- Grain: one row per stage change event
-- ---------------------------------------------------------------
CREATE TABLE fact_stage_transitions (
    transition_id   BIGSERIAL PRIMARY KEY,
    transition_date_key INTEGER NOT NULL,      -- FK to dim_date
    deal_key        INTEGER NOT NULL,          -- FK to dim_deal
    owner_user_key  INTEGER,                   -- FK to dim_user
    from_stage_key  INTEGER NOT NULL,          -- FK to dim_stage
    to_stage_key    INTEGER NOT NULL,          -- FK to dim_stage
    tenant_id       UUID NOT NULL,

    -- Measures
    days_in_from_stage INTEGER NOT NULL,
    amount_at_transition NUMERIC(18,2),
    probability_before NUMERIC(5,2),
    probability_after NUMERIC(5,2),
    is_forward_progression BOOLEAN NOT NULL,   -- TRUE if stage order increased
    is_close_event  BOOLEAN NOT NULL,          -- TRUE if transitioning to closed stage
    transitioned_at TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_fact_transitions_tenant ON fact_stage_transitions(tenant_id, transition_date_key);
CREATE INDEX idx_fact_transitions_deal ON fact_stage_transitions(deal_key);

-- ---------------------------------------------------------------
-- FACT: Activity Metrics (daily aggregates per deal)
-- Grain: one row per deal per day where activity occurred
-- ---------------------------------------------------------------
CREATE TABLE fact_deal_activity (
    activity_id     BIGSERIAL PRIMARY KEY,
    activity_date_key INTEGER NOT NULL,        -- FK to dim_date
    deal_key        INTEGER NOT NULL,          -- FK to dim_deal
    owner_user_key  INTEGER,                   -- FK to dim_user
    tenant_id       UUID NOT NULL,

    -- Measures
    call_count      INTEGER DEFAULT 0,
    email_count     INTEGER DEFAULT 0,
    meeting_count   INTEGER DEFAULT 0,
    task_count      INTEGER DEFAULT 0,
    note_count      INTEGER DEFAULT 0,
    total_activities INTEGER DEFAULT 0,
    total_duration_minutes INTEGER DEFAULT 0,
    inbound_count   INTEGER DEFAULT 0,
    outbound_count  INTEGER DEFAULT 0,
    unique_contacts_engaged INTEGER DEFAULT 0
);

CREATE INDEX idx_fact_activity_deal ON fact_deal_activity(deal_key, activity_date_key);
CREATE INDEX idx_fact_activity_tenant ON fact_deal_activity(tenant_id, activity_date_key);

-- ---------------------------------------------------------------
-- FACT: ML Score Changes
-- Grain: one row per scoring event per deal
-- ---------------------------------------------------------------
CREATE TABLE fact_score_changes (
    score_change_id BIGSERIAL PRIMARY KEY,
    score_date_key  INTEGER NOT NULL,          -- FK to dim_date
    deal_key        INTEGER NOT NULL,          -- FK to dim_deal
    risk_key        INTEGER,                   -- FK to dim_risk_level
    tenant_id       UUID NOT NULL,

    -- Measures
    win_probability NUMERIC(5,4) NOT NULL,
    previous_probability NUMERIC(5,4),
    probability_delta NUMERIC(5,4),
    confidence_lower NUMERIC(5,4),
    confidence_upper NUMERIC(5,4),
    confidence_width NUMERIC(5,4),             -- upper - lower
    dq_score        NUMERIC(5,2),
    model_version   VARCHAR(50),
    top_factor_positive VARCHAR(255),
    top_factor_negative VARCHAR(255),
    explanation     TEXT,
    scored_at       TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_fact_scores_deal ON fact_score_changes(deal_key, score_date_key);
CREATE INDEX idx_fact_scores_tenant ON fact_score_changes(tenant_id, score_date_key);
```

## Materialized Views (Pre-Computed Aggregates)

```sql
-- =============================================================
-- MATERIALIZED VIEWS — SUB-SECOND DASHBOARD QUERIES
-- =============================================================

-- Pipeline summary by stage (refreshed every 15 minutes)
CREATE MATERIALIZED VIEW mv_pipeline_by_stage AS
SELECT
    fds.tenant_id,
    fds.snapshot_date_key,
    dd.full_date AS snapshot_date,
    ds.pipeline_name,
    ds.stage_name,
    ds.display_order,
    ds.forecast_category,
    COUNT(*) AS deal_count,
    SUM(fds.amount_usd) AS total_amount,
    SUM(fds.weighted_amount) AS weighted_amount,
    AVG(fds.ml_win_probability) AS avg_win_probability,
    AVG(fds.dq_score) AS avg_dq_score,
    AVG(fds.days_in_current_stage) AS avg_days_in_stage
FROM fact_deal_snapshots fds
JOIN dim_date dd ON dd.date_key = fds.snapshot_date_key
JOIN dim_stage ds ON ds.stage_key = fds.stage_key
WHERE NOT fds.is_closed
GROUP BY fds.tenant_id, fds.snapshot_date_key, dd.full_date,
         ds.pipeline_name, ds.stage_name, ds.display_order, ds.forecast_category;

CREATE UNIQUE INDEX idx_mv_pipeline_stage ON mv_pipeline_by_stage(tenant_id, snapshot_date_key, pipeline_name, stage_name);

-- Rep performance summary (refreshed daily)
CREATE MATERIALIZED VIEW mv_rep_performance AS
SELECT
    fds.tenant_id,
    du.user_id,
    du.name AS rep_name,
    du.team,
    du.manager_name,
    du.quota,
    dfp.period_label,
    dfp.fiscal_year,
    dfp.fiscal_quarter,
    COUNT(DISTINCT fds.deal_key) AS active_deals,
    SUM(CASE WHEN NOT fds.is_closed THEN fds.amount_usd ELSE 0 END) AS open_pipeline,
    SUM(CASE WHEN fds.is_won THEN fds.amount_usd ELSE 0 END) AS closed_won,
    SUM(CASE WHEN fds.is_closed AND NOT fds.is_won THEN fds.amount_usd ELSE 0 END) AS closed_lost,
    AVG(fds.ml_win_probability) FILTER (WHERE NOT fds.is_closed) AS avg_open_deal_probability,
    AVG(fds.dq_score) FILTER (WHERE NOT fds.is_closed) AS avg_dq_score,
    CASE WHEN du.quota > 0 THEN SUM(CASE WHEN fds.is_won THEN fds.amount_usd ELSE 0 END) / du.quota ELSE NULL END AS quota_attainment
FROM fact_deal_snapshots fds
JOIN dim_user du ON du.user_key = fds.owner_user_key AND du.is_current
JOIN dim_fiscal_period dfp ON dfp.tenant_id = fds.tenant_id
    AND fds.snapshot_date_key BETWEEN
        (dfp.start_date - '2020-01-01'::DATE + 20200101)  -- Convert date to YYYYMMDD key
        AND (dfp.end_date - '2020-01-01'::DATE + 20200101)
WHERE dfp.period_type = 'quarterly'
GROUP BY fds.tenant_id, du.user_id, du.name, du.team, du.manager_name, du.quota,
         dfp.period_label, dfp.fiscal_year, dfp.fiscal_quarter;

-- Forecast accuracy trend (refreshed daily after period close)
CREATE MATERIALIZED VIEW mv_forecast_accuracy_trend AS
SELECT
    ffa.tenant_id,
    dfp.period_label,
    dfp.fiscal_year,
    dfp.fiscal_quarter,
    COUNT(*) AS rep_count,
    AVG(ffa.submitted_error_pct) AS avg_human_error_pct,
    AVG(ffa.ai_error_pct) AS avg_ai_error_pct,
    SUM(CASE WHEN ffa.ai_beat_human THEN 1 ELSE 0 END)::NUMERIC / COUNT(*) AS ai_win_rate,
    AVG(ffa.submitted_abs_error) AS avg_human_abs_error,
    AVG(ffa.ai_abs_error) AS avg_ai_abs_error
FROM fact_forecast_accuracy ffa
JOIN dim_fiscal_period dfp ON dfp.period_key = ffa.period_key
GROUP BY ffa.tenant_id, dfp.period_label, dfp.fiscal_year, dfp.fiscal_quarter;

-- Pipeline movement (deals entering/leaving stages per week)
CREATE MATERIALIZED VIEW mv_pipeline_movement AS
SELECT
    fst.tenant_id,
    dd.year_number,
    dd.week_of_year,
    MIN(dd.full_date) AS week_start,
    ds_from.stage_name AS from_stage,
    ds_to.stage_name AS to_stage,
    COUNT(*) AS transition_count,
    SUM(fst.amount_at_transition) AS total_amount_transitioned,
    AVG(fst.days_in_from_stage) AS avg_days_in_from_stage,
    SUM(CASE WHEN fst.is_forward_progression THEN 1 ELSE 0 END) AS forward_count,
    SUM(CASE WHEN NOT fst.is_forward_progression THEN 1 ELSE 0 END) AS backward_count
FROM fact_stage_transitions fst
JOIN dim_date dd ON dd.date_key = fst.transition_date_key
JOIN dim_stage ds_from ON ds_from.stage_key = fst.from_stage_key
JOIN dim_stage ds_to ON ds_to.stage_key = fst.to_stage_key
GROUP BY fst.tenant_id, dd.year_number, dd.week_of_year, ds_from.stage_name, ds_to.stage_name;
```

## ETL Refresh Configuration

```sql
-- =============================================================
-- ETL JOB TRACKING
-- =============================================================

CREATE TABLE etl_jobs (
    id              BIGSERIAL PRIMARY KEY,
    job_name        VARCHAR(100) NOT NULL,     -- 'refresh_fact_deal_snapshots', 'refresh_dim_deal', etc.
    job_type        VARCHAR(50) NOT NULL,      -- dimension_load, fact_load, mv_refresh
    status          VARCHAR(50) NOT NULL,      -- running, completed, failed
    started_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    completed_at    TIMESTAMPTZ,
    rows_processed  BIGINT,
    error_message   TEXT,
    tenant_id       UUID                       -- NULL for cross-tenant jobs
);

CREATE INDEX idx_etl_jobs_status ON etl_jobs(status, started_at DESC);

-- Example ETL schedule:
-- dim_date:                   Once (pre-populated 10 years)
-- dim_deal, dim_account:      Every 15 minutes (SCD Type 2 updates)
-- dim_user:                   Every hour
-- fact_deal_snapshots:        Daily at 2:00 AM (one row per open deal)
-- fact_stage_transitions:     Real-time (triggered by CRM sync)
-- fact_deal_activity:         Hourly (aggregate from op_activities)
-- fact_score_changes:         Real-time (triggered by ML scoring)
-- fact_forecast_submissions:  Real-time (triggered by user submission)
-- fact_forecast_accuracy:     Weekly (after period close)
-- mv_pipeline_by_stage:       Every 15 minutes
-- mv_rep_performance:         Daily at 3:00 AM
-- mv_forecast_accuracy_trend: Daily at 3:00 AM
-- mv_pipeline_movement:       Daily at 3:00 AM
```

## Example Analytical Queries

```sql
-- =============================================================
-- EXAMPLE: Pipeline trend over last 12 weeks
-- =============================================================
SELECT
    dd.year_number,
    dd.week_of_year,
    SUM(fds.amount_usd) AS total_pipeline,
    SUM(fds.weighted_amount) AS weighted_pipeline,
    COUNT(DISTINCT fds.deal_key) AS deal_count,
    AVG(fds.ml_win_probability) AS avg_win_probability
FROM fact_deal_snapshots fds
JOIN dim_date dd ON dd.date_key = fds.snapshot_date_key
WHERE fds.tenant_id = $1
  AND dd.day_of_week = 5  -- Friday snapshots
  AND dd.full_date >= CURRENT_DATE - INTERVAL '12 weeks'
  AND NOT fds.is_closed
GROUP BY dd.year_number, dd.week_of_year
ORDER BY dd.year_number, dd.week_of_year;

-- =============================================================
-- EXAMPLE: Rep forecast accuracy vs AI accuracy (last 4 quarters)
-- =============================================================
SELECT
    dfp.period_label,
    du.name AS rep_name,
    ffa.submitted_amount,
    ffa.ai_predicted_amount,
    ffa.actual_amount,
    ROUND(ffa.submitted_error_pct * 100, 1) AS human_error_pct,
    ROUND(ffa.ai_error_pct * 100, 1) AS ai_error_pct,
    ffa.ai_beat_human
FROM fact_forecast_accuracy ffa
JOIN dim_fiscal_period dfp ON dfp.period_key = ffa.period_key
JOIN dim_user du ON du.user_key = ffa.user_key AND du.is_current
WHERE ffa.tenant_id = $1
  AND dfp.fiscal_year >= 2025
ORDER BY dfp.fiscal_year, dfp.fiscal_quarter, du.name;

-- =============================================================
-- EXAMPLE: Deals at risk — declining probability + low activity
-- =============================================================
SELECT
    dd_deal.name AS deal_name,
    da.name AS account_name,
    du.name AS rep_name,
    fds.amount_usd,
    fds.ml_win_probability,
    fds.days_since_last_activity,
    fds.close_date_slip_count,
    fds.dq_score,
    dr.risk_level
FROM fact_deal_snapshots fds
JOIN dim_deal dd_deal ON dd_deal.deal_key = fds.deal_key AND dd_deal.is_current
JOIN dim_account da ON da.account_key = fds.account_key AND da.is_current
JOIN dim_user du ON du.user_key = fds.owner_user_key AND du.is_current
JOIN dim_risk_level dr ON dr.risk_key = fds.risk_key
WHERE fds.tenant_id = $1
  AND fds.snapshot_date_key = (SELECT MAX(date_key) FROM dim_date WHERE full_date <= CURRENT_DATE)
  AND NOT fds.is_closed
  AND dr.risk_level IN ('high', 'critical')
ORDER BY fds.amount_usd DESC;

-- =============================================================
-- EXAMPLE: Stage conversion rates by pipeline
-- =============================================================
SELECT
    ds_from.stage_name AS from_stage,
    ds_to.stage_name AS to_stage,
    COUNT(*) AS transitions,
    AVG(fst.days_in_from_stage) AS avg_days,
    SUM(CASE WHEN fst.is_forward_progression THEN 1 ELSE 0 END)::NUMERIC / COUNT(*) AS forward_rate
FROM fact_stage_transitions fst
JOIN dim_stage ds_from ON ds_from.stage_key = fst.from_stage_key
JOIN dim_stage ds_to ON ds_to.stage_key = fst.to_stage_key
WHERE fst.tenant_id = $1
  AND fst.transition_date_key >= 20260101
GROUP BY ds_from.stage_name, ds_to.stage_name, ds_from.display_order
ORDER BY ds_from.display_order;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Operational — Tenancy | 3 | op_tenants, op_users, op_crm_connections |
| Operational — CRM Data | 4 | op_accounts, op_contacts, op_deals, op_activities |
| Operational — Scoring & Forecasts | 2 | op_deal_scores, op_forecast_submissions |
| Dimensions | 7 | dim_date, dim_fiscal_period, dim_deal, dim_account, dim_user, dim_stage, dim_crm_source, dim_risk_level |
| Fact Tables | 5 | fact_deal_snapshots, fact_forecast_submissions, fact_forecast_accuracy, fact_stage_transitions, fact_deal_activity, fact_score_changes |
| Materialized Views | 4 | mv_pipeline_by_stage, mv_rep_performance, mv_forecast_accuracy_trend, mv_pipeline_movement |
| Infrastructure | 1 | etl_jobs |
| **Total** | **26** | 9 operational + 8 dimensions + 6 facts + 4 MVs + 1 infra |

---

## Key Design Decisions

1. **Two-layer architecture with clear separation.** Operational tables (prefixed `op_`) handle CRM sync and ML scoring with normalized structure optimized for writes. Analytical tables (prefixed `dim_` and `fact_`) handle dashboard queries with denormalized structure optimized for reads. This prevents the common problem of dashboard queries degrading CRM sync performance.

2. **`fact_deal_snapshots` as the primary analytical table.** One row per deal per day captures the complete state of every deal at each point in time. This enables pipeline trend charts, cohort analysis, and temporal comparisons without complex self-joins. The table is partitioned by `snapshot_date_key` for efficient time-range scans.

3. **Integer surrogate keys on dimensions.** Dimension tables use auto-incrementing integer keys (`deal_key`, `account_key`, `user_key`) instead of UUIDs for fact table foreign keys. Integer joins are faster than UUID joins, and integer keys compress better in the partitioned fact tables that will grow to millions of rows.

4. **SCD Type 2 on deal, account, and user dimensions.** When a deal's pipeline changes or a user's team changes, a new dimension row is created with `effective_from`/`effective_to` dates. This means historical fact table rows automatically reflect the dimension values that were true at the time of the snapshot, not the current values.

5. **Separate fact tables for different business processes.** Rather than one monolithic fact table, each business process (deal snapshots, forecast submissions, stage transitions, activities, score changes) has its own fact table at its own grain. This follows Kimball's bus matrix pattern and avoids fan-trap join problems.

6. **Materialized views for sub-second dashboards.** The four materialized views pre-compute the most expensive aggregations (pipeline by stage, rep performance, forecast accuracy, pipeline movement). These are refreshed on schedule (15-minute to daily) and serve dashboard queries in single-digit milliseconds.

7. **`dim_fiscal_period` per tenant.** Different tenants may have different fiscal year start months (January vs. February vs. April). The fiscal period dimension is tenant-specific, ensuring that "Q2 2026" means the correct date range for each organization.

8. **`fact_forecast_accuracy` with `ai_beat_human` flag.** Pre-computing whether the AI or the human was more accurate for each rep/period enables the "AI vs. Human accuracy" dashboard without runtime calculation. This directly supports the product's value proposition of demonstrating AI accuracy improvements over time.

9. **ETL job tracking for operational visibility.** The `etl_jobs` table provides observability into the data pipeline: when was the last refresh, how many rows were processed, did any jobs fail? This is essential for debugging data freshness issues in the analytical layer.

10. **Date dimension as integer keys (YYYYMMDD).** The `dim_date` table uses integer keys like `20260511` instead of DATE types. Integer comparison and partitioning is faster, and the format is human-readable for debugging. The `full_date` column provides the DATE type when needed.
