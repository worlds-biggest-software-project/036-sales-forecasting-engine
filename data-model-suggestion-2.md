# Data Model Suggestion 2: Event-Sourced / Audit-First

> Project: Sales Forecasting Engine · Created: 2026-05-11

## Philosophy

The event-sourced approach treats every change to a deal, forecast, or score as an immutable event appended to an event store. The current state of any entity is derived by replaying its event history. This architecture is paired with CQRS (Command Query Responsibility Segregation): writes go to the event store, while reads are served from materialized projections optimized for specific query patterns.

This philosophy directly addresses two critical requirements of a sales forecasting engine. First, forecast accuracy tracking requires knowing "what was the forecast on Week 3 of Q2?" — which is a temporal query that traditional CRUD databases handle poorly but event sourcing answers trivially. Second, deal score explainability benefits from a complete history of every change: "probability dropped from 72% to 41% because the close date slipped for the third time and no activity was logged for 21 days" is a narrative that emerges naturally from an event stream.

Event sourcing is used in financial trading systems (every order and fill is an event), audit-critical compliance platforms, and healthcare EHR systems where regulatory requirements demand a complete, immutable record of all changes. In the sales forecasting context, it provides the foundation for AI-driven pattern detection across deal lifecycles — the ML model can learn from sequences of events (stage changes, activity patterns, close date movements) rather than just point-in-time snapshots.

**Best for:** Organizations that need full audit trails, temporal queries ("what was true on date X?"), and AI pattern detection from deal change sequences.

**Trade-offs:**
- Pro: Complete, immutable audit trail of every change — impossible to lose history
- Pro: Temporal queries are trivial: replay events up to any point in time
- Pro: ML training data is richer — event sequences reveal patterns that snapshots miss
- Pro: Forecast accuracy tracking is built into the architecture, not bolted on
- Pro: Schema evolution is easier — new event types can be added without migrating existing data
- Con: Higher storage requirements (every change is stored, not just current state)
- Con: Read queries require materialized views / projections that must be maintained
- Con: Eventual consistency between event store and read models requires careful engineering
- Con: Debugging is harder — current state must be understood as the result of event replay
- Con: Team must understand event sourcing patterns; steeper learning curve than traditional CRUD

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| Salesforce Object Model | CRM sync events capture Salesforce Opportunity field changes as `DealFieldChanged` events |
| HubSpot CRM Object Model | HubSpot Deal property changes mapped to the same canonical event types |
| MEDDIC / MEDDPICC | Qualification updates modeled as `DealQualificationUpdated` events with dimension-level granularity |
| ISO/IEC 5259-2:2024 | Data quality assessments emitted as `DataQualityScored` events, creating a temporal quality audit trail |
| ASC 606 / GAAP | Forecast submissions and adjustments are events with timestamps, enabling exact period-end state reconstruction |
| ISO/IEC 25010 | Event store durability and projection consistency directly address reliability quality characteristics |
| OpenTelemetry (OTEL) | Event processing pipeline instrumented with OTEL traces for latency and throughput monitoring |

---

## Event Store Core

```sql
-- =============================================================
-- EVENT STORE — APPEND-ONLY, IMMUTABLE
-- =============================================================

-- Central event store table — all domain events are appended here.
-- This is the single source of truth for the entire system.
CREATE TABLE events (
    event_id        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    stream_id       UUID NOT NULL,            -- Aggregate root ID (deal_id, forecast_id, etc.)
    stream_type     VARCHAR(50) NOT NULL,      -- deal, forecast, account, contact, model, recording
    tenant_id       UUID NOT NULL,
    event_type      VARCHAR(100) NOT NULL,     -- e.g., DealCreated, DealStageChanged, ScoreComputed
    event_version   INTEGER NOT NULL,          -- Monotonically increasing per stream
    payload         JSONB NOT NULL,            -- Event-specific data
    metadata        JSONB NOT NULL DEFAULT '{}', -- Causation/correlation IDs, user agent, etc.
    occurred_at     TIMESTAMPTZ NOT NULL,      -- When the event happened in the real world
    recorded_at     TIMESTAMPTZ NOT NULL DEFAULT now(),  -- When it was stored
    UNIQUE(stream_id, event_version)
);

-- Partition by month for efficient time-range queries and storage management
-- In production, use PostgreSQL declarative partitioning:
-- CREATE TABLE events (...) PARTITION BY RANGE (recorded_at);

CREATE INDEX idx_events_stream ON events(stream_id, event_version ASC);
CREATE INDEX idx_events_tenant_type ON events(tenant_id, event_type, occurred_at DESC);
CREATE INDEX idx_events_tenant_time ON events(tenant_id, occurred_at DESC);
CREATE INDEX idx_events_type ON events(event_type, occurred_at DESC);

-- Optimistic concurrency control: prevent conflicting writes to the same stream
-- The UNIQUE(stream_id, event_version) constraint serves this purpose.
```

## Event Type Catalogue

```sql
-- =============================================================
-- EVENT TYPE REGISTRY
-- =============================================================

-- Documents all known event types and their payload schemas.
-- Used for validation, documentation, and schema evolution tracking.
CREATE TABLE event_type_registry (
    event_type      VARCHAR(100) PRIMARY KEY,
    stream_type     VARCHAR(50) NOT NULL,
    description     TEXT NOT NULL,
    payload_schema  JSONB NOT NULL,           -- JSON Schema defining the payload structure
    version         INTEGER NOT NULL DEFAULT 1,
    deprecated      BOOLEAN DEFAULT FALSE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Example event types and their payload structures:
--
-- DealCreated:
-- {
--   "external_id": "0065f00000ABC123",
--   "crm_type": "salesforce",
--   "name": "Acme Corp - Enterprise License",
--   "amount": 150000.00,
--   "currency_code": "USD",
--   "stage_name": "Qualification",
--   "close_date": "2026-09-30",
--   "owner_external_id": "0055f00000XYZ789",
--   "account_external_id": "0015f00000DEF456"
-- }
--
-- DealStageChanged:
-- {
--   "previous_stage": "Qualification",
--   "new_stage": "Proposal",
--   "previous_probability": 20.0,
--   "new_probability": 60.0,
--   "days_in_previous_stage": 14,
--   "change_source": "crm_sync"
-- }
--
-- DealAmountChanged:
-- {
--   "previous_amount": 150000.00,
--   "new_amount": 175000.00,
--   "currency_code": "USD",
--   "change_source": "crm_sync"
-- }
--
-- DealCloseDateChanged:
-- {
--   "previous_close_date": "2026-09-30",
--   "new_close_date": "2026-10-31",
--   "slip_count": 2,
--   "total_days_slipped": 31,
--   "change_source": "crm_sync"
-- }
--
-- DealClosed:
-- {
--   "is_won": true,
--   "final_amount": 175000.00,
--   "final_stage": "Closed Won",
--   "days_to_close": 87,
--   "close_date": "2026-09-15"
-- }
--
-- ScoreComputed:
-- {
--   "model_id": "uuid",
--   "model_version": "xgboost-v3.2",
--   "win_probability": 0.6823,
--   "previous_probability": 0.7214,
--   "probability_delta": -0.0391,
--   "confidence_lower": 0.5901,
--   "confidence_upper": 0.7745,
--   "risk_level": "medium",
--   "top_positive_factors": ["champion_identified", "multi_threaded"],
--   "top_negative_factors": ["close_date_slipped_2x", "no_exec_sponsor"],
--   "explanation": "Score dropped 4pts: close date slipped twice, no executive sponsor identified",
--   "shap_values": {
--     "days_since_last_activity": -0.0823,
--     "close_date_slip_count": -0.0654,
--     "contact_count": 0.0412,
--     "has_champion": 0.0389
--   }
-- }
--
-- ForecastSubmitted:
-- {
--   "period_type": "quarterly",
--   "period_label": "Q2 2026",
--   "fiscal_quarter": 2,
--   "fiscal_year": 2026,
--   "category": "commit",
--   "submitted_amount": 1250000.00,
--   "ai_adjusted_amount": 1105000.00,
--   "ai_adjustment_reason": "AI reduced commit by $145K due to 4 high-risk deals",
--   "deal_count": 23,
--   "pipeline_coverage": 3.2
-- }
--
-- DataQualityScored:
-- {
--   "dq_score": 42.5,
--   "flags": ["stale_close_date", "no_activity_30_days", "missing_champion"],
--   "previous_dq_score": 68.0,
--   "score_delta": -25.5
-- }
--
-- ActivityLogged:
-- {
--   "activity_type": "call",
--   "subject": "Discovery call with VP Engineering",
--   "duration_minutes": 45,
--   "direction": "outbound",
--   "contact_external_id": "0035f00000GHI012"
-- }
--
-- CallTranscribed:
-- {
--   "recording_id": "uuid",
--   "duration_seconds": 2700,
--   "word_count": 4521,
--   "whisper_model": "large-v3",
--   "language": "en"
-- }
--
-- CallInsightExtracted:
-- {
--   "recording_id": "uuid",
--   "insight_type": "objection",
--   "summary": "Prospect raised budget concerns for Q3 allocation",
--   "confidence": 0.92,
--   "llm_model": "claude-opus-4-20250514"
-- }
--
-- AlertTriggered:
-- {
--   "alert_type": "no_activity",
--   "threshold_days": 14,
--   "actual_days": 21,
--   "severity": "warning",
--   "message": "No activity logged for 21 days on Acme Corp deal"
-- }
```

## Tenant & Connection Projections (Read Models)

```sql
-- =============================================================
-- READ MODEL PROJECTIONS — MATERIALIZED FROM EVENTS
-- =============================================================
-- These tables are rebuilt/updated by event handlers (projections).
-- They can be dropped and rebuilt from the event store at any time.

CREATE TABLE rm_tenants (
    id              UUID PRIMARY KEY,
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    plan            VARCHAR(50) NOT NULL DEFAULT 'free',
    settings        JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL,
    -- Projection metadata
    last_event_id   UUID,
    projected_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE rm_users (
    id              UUID PRIMARY KEY,
    tenant_id       UUID NOT NULL,
    email           VARCHAR(255) NOT NULL,
    name            VARCHAR(255) NOT NULL,
    role            VARCHAR(50) NOT NULL,
    last_login_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL,
    last_event_id   UUID,
    projected_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_rm_users_tenant ON rm_users(tenant_id);

CREATE TABLE rm_crm_connections (
    id              UUID PRIMARY KEY,
    tenant_id       UUID NOT NULL,
    crm_type        VARCHAR(50) NOT NULL,
    instance_url    TEXT,
    sync_status     VARCHAR(50) NOT NULL,
    last_sync_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL,
    last_event_id   UUID,
    projected_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

## Deal Projection (Current State Read Model)

```sql
-- =============================================================
-- DEAL CURRENT STATE PROJECTION
-- =============================================================

-- The primary read model for deal data. Rebuilt from DealCreated,
-- DealStageChanged, DealAmountChanged, DealCloseDateChanged,
-- DealClosed, ScoreComputed, DataQualityScored events.
CREATE TABLE rm_deals (
    id              UUID PRIMARY KEY,
    tenant_id       UUID NOT NULL,
    crm_connection_id UUID NOT NULL,
    external_id     VARCHAR(255) NOT NULL,
    account_id      UUID,
    name            VARCHAR(500) NOT NULL,
    amount          NUMERIC(18,2),
    currency_code   VARCHAR(3) DEFAULT 'USD',
    stage_name      VARCHAR(255) NOT NULL,
    pipeline_name   VARCHAR(255),
    crm_probability NUMERIC(5,2),
    close_date      DATE,
    is_closed       BOOLEAN NOT NULL DEFAULT FALSE,
    is_won          BOOLEAN NOT NULL DEFAULT FALSE,
    deal_type       VARCHAR(100),
    lead_source     VARCHAR(255),
    owner_external_id VARCHAR(255),

    -- MEDDIC fields (projected from DealQualificationUpdated events)
    meddic_metrics          TEXT,
    meddic_economic_buyer   VARCHAR(255),
    meddic_decision_criteria TEXT,
    meddic_decision_process TEXT,
    meddic_identify_pain    TEXT,
    meddic_champion         VARCHAR(255),
    meddic_competition      TEXT,

    -- Latest ML score (projected from ScoreComputed events)
    current_win_probability NUMERIC(5,4),
    current_risk_level      VARCHAR(20),
    current_score_explanation TEXT,
    score_trend             VARCHAR(20),       -- improving, stable, declining
    last_scored_at          TIMESTAMPTZ,

    -- Data quality (projected from DataQualityScored events)
    dq_score                NUMERIC(5,2),
    dq_flags                TEXT[],

    -- Computed lifecycle metrics (derived from event sequence)
    days_in_current_stage   INTEGER,
    days_since_last_activity INTEGER,
    close_date_slip_count   INTEGER DEFAULT 0,
    total_days_slipped      INTEGER DEFAULT 0,
    stage_change_count      INTEGER DEFAULT 0,
    activity_count_30d      INTEGER DEFAULT 0,
    contact_count           INTEGER DEFAULT 0,
    days_in_pipeline        INTEGER,

    created_at      TIMESTAMPTZ NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL,
    last_event_id   UUID,
    last_event_version INTEGER,
    projected_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(crm_connection_id, external_id)
);

CREATE INDEX idx_rm_deals_tenant ON rm_deals(tenant_id);
CREATE INDEX idx_rm_deals_open ON rm_deals(tenant_id, is_closed) WHERE NOT is_closed;
CREATE INDEX idx_rm_deals_stage ON rm_deals(tenant_id, stage_name);
CREATE INDEX idx_rm_deals_close_date ON rm_deals(tenant_id, close_date);
CREATE INDEX idx_rm_deals_risk ON rm_deals(current_risk_level) WHERE current_risk_level IN ('high', 'critical');
```

## Forecast Projection

```sql
-- =============================================================
-- FORECAST READ MODELS
-- =============================================================

-- Current forecast state per period, rebuilt from ForecastSubmitted events
CREATE TABLE rm_forecasts (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    period_type     VARCHAR(20) NOT NULL,
    period_label    VARCHAR(50) NOT NULL,
    fiscal_year     INTEGER NOT NULL,
    fiscal_quarter  INTEGER,
    start_date      DATE NOT NULL,
    end_date        DATE NOT NULL,
    is_current      BOOLEAN DEFAULT FALSE,

    -- Latest aggregated numbers
    total_pipeline      NUMERIC(18,2),
    total_commit        NUMERIC(18,2),
    total_best_case     NUMERIC(18,2),
    total_most_likely   NUMERIC(18,2),
    ai_predicted_total  NUMERIC(18,2),
    ai_predicted_lower  NUMERIC(18,2),
    ai_predicted_upper  NUMERIC(18,2),
    deal_count          INTEGER,
    pipeline_coverage   NUMERIC(5,2),

    last_event_id   UUID,
    projected_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_rm_forecasts_tenant ON rm_forecasts(tenant_id, fiscal_year, fiscal_quarter);

-- Time-series of forecast values over time, rebuilt from ForecastSubmitted events.
-- One row per day per period — enables "forecast over time" charts.
CREATE TABLE rm_forecast_timeline (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    period_label    VARCHAR(50) NOT NULL,
    snapshot_date   DATE NOT NULL,
    user_id         UUID,                     -- NULL for company-level aggregates
    category        VARCHAR(50) NOT NULL,     -- commit, best_case, most_likely, ai_predicted
    amount          NUMERIC(18,2) NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, period_label, snapshot_date, user_id, category)
);

CREATE INDEX idx_rm_forecast_timeline_period ON rm_forecast_timeline(tenant_id, period_label, snapshot_date);
```

## Deal History Projection (Temporal Queries)

```sql
-- =============================================================
-- DEAL HISTORY PROJECTION — ENABLES "AS OF DATE" QUERIES
-- =============================================================

-- Captures the state of each deal at the end of each day where a change occurred.
-- Projected from the event stream by replaying events up to each day boundary.
CREATE TABLE rm_deal_history (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    deal_id         UUID NOT NULL,
    tenant_id       UUID NOT NULL,
    effective_date  DATE NOT NULL,
    amount          NUMERIC(18,2),
    stage_name      VARCHAR(255),
    close_date      DATE,
    win_probability NUMERIC(5,4),
    risk_level      VARCHAR(20),
    dq_score        NUMERIC(5,2),
    is_closed       BOOLEAN,
    is_won          BOOLEAN,
    UNIQUE(deal_id, effective_date)
);

CREATE INDEX idx_rm_deal_history_tenant_date ON rm_deal_history(tenant_id, effective_date DESC);
CREATE INDEX idx_rm_deal_history_deal ON rm_deal_history(deal_id, effective_date DESC);

-- Example: "What was the total pipeline on March 15?"
-- SELECT SUM(amount) FROM rm_deal_history
-- WHERE tenant_id = $1 AND effective_date = '2026-03-15' AND NOT is_closed;

-- Example: "How did deal X's probability change over time?"
-- SELECT effective_date, win_probability, stage_name
-- FROM rm_deal_history
-- WHERE deal_id = $1
-- ORDER BY effective_date;
```

## Score History Projection

```sql
-- =============================================================
-- SCORE HISTORY — ML AUDIT TRAIL
-- =============================================================

-- Every ScoreComputed event is projected here for score trend analysis.
CREATE TABLE rm_score_history (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    deal_id         UUID NOT NULL,
    tenant_id       UUID NOT NULL,
    scored_at       TIMESTAMPTZ NOT NULL,
    model_id        UUID,
    model_version   VARCHAR(50),
    win_probability NUMERIC(5,4),
    probability_delta NUMERIC(5,4),
    risk_level      VARCHAR(20),
    explanation     TEXT,
    top_positive_factors TEXT[],
    top_negative_factors TEXT[],
    event_id        UUID NOT NULL             -- Back-reference to source event
);

CREATE INDEX idx_rm_score_history_deal ON rm_score_history(deal_id, scored_at DESC);
CREATE INDEX idx_rm_score_history_tenant ON rm_score_history(tenant_id, scored_at DESC);
```

## Conversation Intelligence Read Models

```sql
-- =============================================================
-- CONVERSATION INTELLIGENCE PROJECTIONS
-- =============================================================

CREATE TABLE rm_call_recordings (
    id              UUID PRIMARY KEY,
    tenant_id       UUID NOT NULL,
    deal_id         UUID,
    recording_url   TEXT,
    duration_seconds INTEGER,
    recorded_at     TIMESTAMPTZ,
    participants    TEXT[],
    source          VARCHAR(50),
    transcription_status VARCHAR(50),
    insight_count   INTEGER DEFAULT 0,
    last_event_id   UUID,
    projected_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE rm_call_insights (
    id              UUID PRIMARY KEY,
    recording_id    UUID NOT NULL,
    deal_id         UUID,
    tenant_id       UUID NOT NULL,
    insight_type    VARCHAR(50) NOT NULL,
    summary         TEXT NOT NULL,
    confidence      NUMERIC(5,4),
    extracted_at    TIMESTAMPTZ NOT NULL,
    event_id        UUID NOT NULL
);

CREATE INDEX idx_rm_call_insights_deal ON rm_call_insights(deal_id);
CREATE INDEX idx_rm_call_insights_type ON rm_call_insights(tenant_id, insight_type);
```

## Projection Management

```sql
-- =============================================================
-- PROJECTION CHECKPOINTS
-- =============================================================

-- Tracks the last processed event for each projection, enabling
-- incremental rebuilds and ensuring exactly-once processing.
CREATE TABLE projection_checkpoints (
    projection_name VARCHAR(100) PRIMARY KEY,
    last_event_id   UUID NOT NULL,
    last_event_version INTEGER NOT NULL,
    last_processed_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    status          VARCHAR(50) NOT NULL DEFAULT 'active',  -- active, rebuilding, paused
    error_message   TEXT
);

-- Example projections:
-- 'deal_current_state'   -> rm_deals
-- 'deal_history'         -> rm_deal_history
-- 'forecast_timeline'    -> rm_forecast_timeline
-- 'score_history'        -> rm_score_history
-- 'call_insights'        -> rm_call_insights
```

## Event Replay Example Queries

```sql
-- =============================================================
-- EXAMPLE: Reconstruct deal state at a specific point in time
-- =============================================================

-- Get all events for a deal up to March 15, 2026
SELECT event_type, payload, occurred_at
FROM events
WHERE stream_id = '550e8400-e29b-41d4-a716-446655440000'  -- deal UUID
  AND stream_type = 'deal'
  AND occurred_at <= '2026-03-15T23:59:59Z'
ORDER BY event_version ASC;

-- =============================================================
-- EXAMPLE: Find deals whose close date slipped more than twice
-- =============================================================

SELECT stream_id AS deal_id,
       COUNT(*) AS slip_count,
       MIN((payload->>'previous_close_date')::DATE) AS original_close_date,
       MAX((payload->>'new_close_date')::DATE) AS latest_close_date
FROM events
WHERE tenant_id = $1
  AND event_type = 'DealCloseDateChanged'
  AND occurred_at >= '2026-01-01'
GROUP BY stream_id
HAVING COUNT(*) > 2
ORDER BY slip_count DESC;

-- =============================================================
-- EXAMPLE: Score change velocity — deals losing probability fastest
-- =============================================================

SELECT e.stream_id AS deal_id,
       d.name,
       COUNT(*) AS score_drops,
       SUM((e.payload->>'probability_delta')::NUMERIC) AS total_probability_loss
FROM events e
JOIN rm_deals d ON d.id = e.stream_id
WHERE e.tenant_id = $1
  AND e.event_type = 'ScoreComputed'
  AND (e.payload->>'probability_delta')::NUMERIC < 0
  AND e.occurred_at >= now() - INTERVAL '30 days'
  AND NOT d.is_closed
GROUP BY e.stream_id, d.name
ORDER BY total_probability_loss ASC
LIMIT 10;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Event Store | 2 | events (source of truth), event_type_registry |
| Tenant & Auth Projections | 3 | rm_tenants, rm_users, rm_crm_connections |
| Deal Projections | 2 | rm_deals (current state), rm_deal_history (temporal) |
| Forecast Projections | 2 | rm_forecasts, rm_forecast_timeline |
| Score Projections | 1 | rm_score_history |
| Conversation Projections | 2 | rm_call_recordings, rm_call_insights |
| Infrastructure | 1 | projection_checkpoints |
| **Total** | **13** | 1 event store + 10 read models + 2 infrastructure |

---

## Key Design Decisions

1. **Single event store table, not one per aggregate.** All events go into one `events` table, partitioned by `recorded_at`. This simplifies infrastructure (one table to back up, one to monitor) and enables cross-aggregate queries ("all events for this tenant in the last hour"). The `stream_type` column distinguishes deal events from forecast events.

2. **Event payload in JSONB, not in typed columns.** Each event type has a different payload shape. Storing payloads as JSONB avoids the need for hundreds of nullable columns or a complex type hierarchy. The `event_type_registry` table documents the expected JSON Schema for each event type, enabling validation without schema rigidity.

3. **Read models are expendable projections.** Every `rm_*` table can be dropped and rebuilt from the event store. This means schema changes to read models are trivial — add a column, rebuild the projection. The `projection_checkpoints` table tracks where each projection left off, enabling incremental rebuilds.

4. **`rm_deal_history` enables temporal queries without full event replay.** Rather than replaying the entire event stream to answer "what was the pipeline on March 15?", the deal history projection captures daily snapshots of deal state. This pre-computation trades storage for query speed — critical for dashboard responsiveness.

5. **Score history as a first-class projection.** `rm_score_history` stores every scoring event with its SHAP-derived explanation. This enables the "score over time" chart and retrospective analysis of model accuracy, supporting the forecast accuracy dashboard requirement.

6. **Optimistic concurrency via `(stream_id, event_version)` uniqueness.** When two processes try to append to the same deal's event stream simultaneously, the unique constraint ensures only one succeeds. The losing process must retry with the updated version, preventing lost updates without requiring locks.

7. **Separation of `occurred_at` and `recorded_at`.** `occurred_at` is when the event happened in the real world (e.g., when the stage changed in Salesforce). `recorded_at` is when the engine stored it. This bi-temporal distinction is critical for CRM sync — events may arrive out of order if a sync is delayed, but they are ordered correctly by `occurred_at` for replay.

8. **Event-driven ML training.** The ML model can be trained on event sequences rather than point-in-time snapshots. Features like "number of stage changes in last 30 days" or "close date slip velocity" are computed directly from event queries, providing richer training data than traditional feature engineering from a CRUD database.
