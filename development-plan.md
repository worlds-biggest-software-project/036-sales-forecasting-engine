# Sales Forecasting Engine — Development Plan

> Project: Sales Forecasting Engine (Candidate #36)
> Created: 2026-05-25
> Based on: research.md, features.md, standards.md, README.md, data-model-suggestion-1 through 4

---

## Technology Decisions & Rationale

### Data Model: Hybrid Relational + JSONB (Suggestion 3) with Event Sourcing Audit Layer (Suggestion 2)

**Decision:** Adopt data-model-suggestion-3 (Hybrid Relational + JSONB) as the primary operational schema, augmented with an append-only event log inspired by data-model-suggestion-2 for audit trail and ML training data.

**Rationale:**
- The hybrid model's 18-table schema balances MVP velocity with multi-CRM flexibility. CRM-specific fields go into `crm_fields` JSONB without schema migrations, directly addressing the multi-CRM consolidation requirement.
- MEDDIC qualification data in a `qualification` JSONB column supports per-tenant methodology customisation (MEDDIC vs. MEDDPICC vs. custom) without column explosion.
- Embedding the latest ML score in `deals.scoring` JSONB eliminates the JOIN on the most frequent dashboard query (deal list with scores), while `score_history` provides the audit trail.
- A lightweight `deal_events` append-only table (not full CQRS) captures stage changes, close date slips, and score changes for ML feature engineering and forecast accuracy tracking, without the operational complexity of full event sourcing.
- The analytics-first star schema (Suggestion 4) is deferred to Phase 8 as an optional analytical layer for customers with mature BI needs. The hybrid model's JSONB queries are sufficient for early dashboards.
- The fully normalized model (Suggestion 1) was rejected for MVP because schema migrations on every new CRM field would slow iteration. Its explicit MEDDIC columns are preserved conceptually via the `qualification` JSONB structure.

### Backend: Python (FastAPI)

**Rationale:** Python is the only viable choice given XGBoost, LightGBM, SHAP, and Whisper all have Python-first APIs. FastAPI provides async request handling, OpenAPI 3.1 spec generation (aligning with standards.md), and strong typing via Pydantic. The simple-salesforce and hubspot-api-client libraries are mature Python CRM connectors.

### Database: PostgreSQL 16+

**Rationale:** Native JSONB support with GIN indexing, declarative partitioning for score_history and deal_events tables, and `gen_random_uuid()` for primary keys. PostGIS is not needed. TimescaleDB is unnecessary at MVP scale. Supabase or Neon provide managed hosting options.

### ML Framework: XGBoost + SHAP

**Rationale:** XGBoost/LightGBM ensembles are the M5 competition accuracy leaders. SHAP values provide the model-agnostic feature importance explanations required for natural-language deal score justifications. Prophet is deferred to the pluggable ML backend (Phase 10).

### Frontend: React + TypeScript (Next.js)

**Rationale:** Next.js provides server-side rendering for dashboard performance, API routes for lightweight BFF patterns, and a large hiring pool. Recharts or Tremor for forecast visualisations. Tailwind CSS for rapid UI iteration.

### Task Queue: Celery + Redis

**Rationale:** CRM sync, ML model training, and batch scoring are all background jobs. Celery is the standard Python task queue with retry logic and scheduling. Redis serves double duty as Celery broker and cache for frequently accessed deal lists.

### Authentication: OAuth 2.0 (CRM connectors) + OIDC (user auth)

**Rationale:** CRM APIs require OAuth 2.0 Authorization Code flow (RFC 6749). User authentication uses OIDC via Auth0 or Clerk for multi-tenant SaaS. Aligns with standards.md specifications.

---

## Project Structure

```
sales-forecasting-engine/
├── backend/
│   ├── app/
│   │   ├── api/                    # FastAPI route handlers
│   │   │   ├── auth/               # Login, OAuth callbacks
│   │   │   ├── deals/              # Deal CRUD, scoring, search
│   │   │   ├── forecasts/          # Submissions, snapshots, accuracy
│   │   │   ├── crm/                # CRM connection management
│   │   │   ├── alerts/             # Alert rules and delivery
│   │   │   ├── scenarios/          # What-if scenario endpoints
│   │   │   └── conversations/      # Call recording & insights
│   │   ├── connectors/             # CRM-specific sync adapters
│   │   │   ├── base.py             # Abstract CRM connector interface
│   │   │   ├── salesforce.py       # Salesforce REST API connector
│   │   │   ├── hubspot.py          # HubSpot API v3 connector
│   │   │   └── pipedrive.py        # Pipedrive API v1 connector
│   │   ├── ml/                     # ML pipeline
│   │   │   ├── features.py         # Feature engineering from deal data
│   │   │   ├── training.py         # Model training pipeline
│   │   │   ├── scoring.py          # Batch and real-time scoring
│   │   │   ├── explainer.py        # SHAP explanation generation
│   │   │   └── models/             # Model registry and versioning
│   │   ├── services/               # Business logic layer
│   │   │   ├── deal_service.py
│   │   │   ├── forecast_service.py
│   │   │   ├── scoring_service.py
│   │   │   ├── alert_service.py
│   │   │   ├── quality_service.py  # CRM data quality scoring
│   │   │   └── scenario_service.py
│   │   ├── tasks/                  # Celery background tasks
│   │   │   ├── crm_sync.py
│   │   │   ├── batch_scoring.py
│   │   │   ├── model_training.py
│   │   │   ├── snapshot_capture.py
│   │   │   └── alert_evaluation.py
│   │   ├── models/                 # SQLAlchemy ORM models
│   │   ├── schemas/                # Pydantic request/response schemas
│   │   ├── core/                   # Config, security, dependencies
│   │   └── db/                     # Alembic migrations, session mgmt
│   ├── tests/
│   │   ├── unit/
│   │   ├── integration/
│   │   └── fixtures/               # CRM API mock data
│   ├── alembic/                    # Database migrations
│   ├── pyproject.toml
│   └── Dockerfile
├── frontend/
│   ├── src/
│   │   ├── app/                    # Next.js app router
│   │   ├── components/
│   │   │   ├── deals/              # Deal table, deal detail, score card
│   │   │   ├── forecasts/          # Forecast table, accuracy chart
│   │   │   ├── pipeline/           # Pipeline visualisation
│   │   │   ├── alerts/             # Alert feed, rule config
│   │   │   └── common/             # Layout, navigation, charts
│   │   ├── hooks/                  # React hooks for API calls
│   │   ├── lib/                    # API client, auth, utils
│   │   └── types/                  # TypeScript type definitions
│   ├── package.json
│   └── Dockerfile
├── docker-compose.yml              # Local dev: Postgres, Redis, API, worker, frontend
├── docs/
│   ├── api/                        # OpenAPI spec (auto-generated)
│   └── architecture/
└── .github/
    └── workflows/                  # CI/CD pipelines
```

---

## Phase Dependency Graph

```
Phase 1: Foundation
    │
    ├──► Phase 2: CRM Connectors ──► Phase 4: ML Scoring Pipeline
    │                                    │
    │                                    ├──► Phase 5: Forecast Engine
    │                                    │        │
    ├──► Phase 3: Frontend Shell ────────┤        ├──► Phase 7: Forecast Accuracy
    │                                    │        │
    │                                    ├──► Phase 6: Alerts & Notifications
    │                                    │
    │                                    └──► Phase 8: CRM Data Quality
    │
    Phase 9: What-If Scenarios (depends on Phase 5)
    │
    Phase 10: Pluggable ML Backend (depends on Phase 4)
    │
    Phase 11: Conversation Intelligence (depends on Phases 2, 4)
    │
    Phase 12: Enterprise & Scale (depends on all above)
```

---

## Phase 1: Foundation & Data Model

**Goal:** Establish the core backend, database schema, multi-tenant authentication, and CI/CD pipeline. No CRM integration yet — seed data via fixtures.

### Task 1.1: Project Scaffolding & Configuration

**What:** Create the monorepo structure, configure Python tooling (pyproject.toml, ruff, mypy, pytest), Docker Compose for local dev, and CI pipeline.

**Design:**

```python
# backend/app/core/config.py
from pydantic_settings import BaseSettings

class Settings(BaseSettings):
    DATABASE_URL: str = "postgresql+asyncpg://localhost:5432/forecast_engine"
    REDIS_URL: str = "redis://localhost:6379/0"
    SECRET_KEY: str
    CORS_ORIGINS: list[str] = ["http://localhost:3000"]
    LOG_LEVEL: str = "INFO"

    # Multi-tenancy
    DEFAULT_PLAN: str = "free"

    # ML
    MODEL_ARTIFACT_BUCKET: str = "models"
    SCORING_BATCH_SIZE: int = 500

    class Config:
        env_file = ".env"
```

```yaml
# docker-compose.yml
services:
  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: forecast_engine
      POSTGRES_USER: forecast
      POSTGRES_PASSWORD: localdev
    ports: ["5432:5432"]
    volumes: ["pgdata:/var/lib/postgresql/data"]

  redis:
    image: redis:7-alpine
    ports: ["6379:6379"]

  api:
    build: ./backend
    command: uvicorn app.main:app --reload --host 0.0.0.0 --port 8000
    ports: ["8000:8000"]
    depends_on: [db, redis]
    env_file: .env

  worker:
    build: ./backend
    command: celery -A app.tasks.celery_app worker --loglevel=info
    depends_on: [db, redis]
    env_file: .env

  frontend:
    build: ./frontend
    command: npm run dev
    ports: ["3000:3000"]

volumes:
  pgdata:
```

**Testing:**
- `docker compose up` starts all services without errors
- `pytest backend/tests/unit/test_config.py` validates settings loading from env
- Health check endpoint `GET /api/health` returns `{"status": "ok", "db": "connected", "redis": "connected"}`
- CI pipeline (GitHub Actions) runs lint, type check, and unit tests on every push

### Task 1.2: Database Schema & Migrations

**What:** Implement the hybrid relational + JSONB schema (data-model-suggestion-3) using SQLAlchemy 2.0 ORM models and Alembic migrations. Create tenants, users, crm_connections, accounts, contacts, deals, deal_contacts, activities, pipelines, pipeline_stages, ml_models, score_history, forecast_periods, forecast_submissions, forecast_snapshots, alerts, and field_schemas tables.

**Design:**

```python
# backend/app/models/deal.py
from sqlalchemy import (
    Column, String, Numeric, Date, Boolean, ForeignKey, Index, text
)
from sqlalchemy.dialects.postgresql import UUID, JSONB
from sqlalchemy.orm import relationship
from app.models.base import Base, TimestampMixin
import uuid

class Deal(Base, TimestampMixin):
    __tablename__ = "deals"

    id = Column(UUID(as_uuid=True), primary_key=True, default=uuid.uuid4)
    tenant_id = Column(UUID(as_uuid=True), ForeignKey("tenants.id", ondelete="CASCADE"), nullable=False)
    crm_connection_id = Column(UUID(as_uuid=True), ForeignKey("crm_connections.id", ondelete="CASCADE"), nullable=False)
    external_id = Column(String(255), nullable=False)
    account_id = Column(UUID(as_uuid=True), ForeignKey("accounts.id", ondelete="SET NULL"))
    owner_user_id = Column(UUID(as_uuid=True), ForeignKey("users.id", ondelete="SET NULL"))

    # Canonical typed fields
    name = Column(String(500), nullable=False)
    amount = Column(Numeric(18, 2))
    currency_code = Column(String(3), default="USD")
    stage_name = Column(String(255), nullable=False)
    pipeline_name = Column(String(255))
    close_date = Column(Date)
    is_closed = Column(Boolean, nullable=False, default=False)
    is_won = Column(Boolean, nullable=False, default=False)
    deal_type = Column(String(100))
    lead_source = Column(String(255))
    owner_external_id = Column(String(255))

    # JSONB extension columns
    crm_fields = Column(JSONB, nullable=False, server_default=text("'{}'"))
    qualification = Column(JSONB, nullable=False, server_default=text("'{}'"))
    scoring = Column(JSONB, nullable=False, server_default=text("'{}'"))
    quality = Column(JSONB, nullable=False, server_default=text("'{}'"))
    lifecycle = Column(JSONB, nullable=False, server_default=text("'{}'"))
    custom_fields = Column(JSONB, nullable=False, server_default=text("'{}'"))

    # Relationships
    account = relationship("Account", back_populates="deals")
    contacts = relationship("DealContact", back_populates="deal")
    activities = relationship("Activity", back_populates="deal")
    scores = relationship("ScoreHistory", back_populates="deal")

    __table_args__ = (
        Index("idx_deals_tenant", "tenant_id"),
        Index("idx_deals_open", "tenant_id", "is_closed", postgresql_where=text("NOT is_closed")),
        Index("idx_deals_stage", "tenant_id", "stage_name"),
        Index("idx_deals_close_date", "tenant_id", "close_date"),
        {"schema": None},
    )
```

```python
# backend/app/models/deal_event.py
# Lightweight event log for audit trail and ML feature engineering
class DealEvent(Base):
    __tablename__ = "deal_events"

    id = Column(UUID(as_uuid=True), primary_key=True, default=uuid.uuid4)
    deal_id = Column(UUID(as_uuid=True), ForeignKey("deals.id", ondelete="CASCADE"), nullable=False)
    tenant_id = Column(UUID(as_uuid=True), nullable=False)
    event_type = Column(String(100), nullable=False)  # stage_changed, amount_changed, close_date_changed, score_computed, etc.
    payload = Column(JSONB, nullable=False)
    occurred_at = Column(DateTime(timezone=True), nullable=False)
    recorded_at = Column(DateTime(timezone=True), nullable=False, server_default=text("now()"))

    __table_args__ = (
        Index("idx_deal_events_deal", "deal_id", "occurred_at"),
        Index("idx_deal_events_tenant_type", "tenant_id", "event_type", "occurred_at"),
    )
```

**Testing:**
- `alembic upgrade head` applies all migrations without errors on a fresh database
- `alembic downgrade base` followed by `alembic upgrade head` is idempotent
- Unit tests verify all 19 tables exist with correct column types, constraints, and indexes
- UNIQUE constraint on `(crm_connection_id, external_id)` prevents duplicate deal ingestion (test with conflicting INSERT)
- JSONB columns accept arbitrary JSON payloads and return them via ORM
- Foreign key cascades: deleting a tenant cascades to all child records

### Task 1.3: Multi-Tenant Authentication & User Management

**What:** Implement OIDC-based user authentication (via Auth0 or Clerk), tenant provisioning, user role assignment (admin, manager, rep, viewer), and row-level tenant isolation middleware.

**Design:**

```python
# backend/app/core/auth.py
from fastapi import Depends, HTTPException, Request
from fastapi.security import HTTPBearer
import jwt

security = HTTPBearer()

async def get_current_user(request: Request, token=Depends(security)) -> AuthUser:
    """Validate JWT, extract tenant_id and user_id, verify tenant membership."""
    try:
        payload = jwt.decode(token.credentials, settings.OIDC_PUBLIC_KEY, algorithms=["RS256"])
    except jwt.InvalidTokenError:
        raise HTTPException(status_code=401, detail="Invalid token")

    user = await user_service.get_by_auth_id(payload["sub"])
    if not user:
        raise HTTPException(status_code=403, detail="User not registered")

    request.state.tenant_id = user.tenant_id
    request.state.user = user
    return user

def require_role(*roles: str):
    """Dependency that checks user role against allowed roles."""
    async def checker(user: AuthUser = Depends(get_current_user)):
        if user.role not in roles:
            raise HTTPException(status_code=403, detail="Insufficient permissions")
        return user
    return checker
```

```python
# backend/app/core/tenant_middleware.py
class TenantIsolationMiddleware:
    """Ensures all database queries include tenant_id filter."""
    async def __call__(self, request: Request, call_next):
        # After auth, inject tenant_id into SQLAlchemy session execution options
        response = await call_next(request)
        return response
```

**Testing:**
- Valid JWT returns user object with correct tenant_id and role
- Expired JWT returns 401
- User from tenant A cannot access tenant B's deals (returns 403 or empty result)
- Role enforcement: `rep` cannot access admin-only endpoints (e.g., tenant settings)
- Tenant provisioning creates tenant, admin user, and default pipeline stages
- Rate limiting: 100 requests/minute per user, 1000/minute per tenant

### Task 1.4: Seed Data & Development Fixtures

**What:** Create realistic seed data for development and testing: 1 tenant, 5 users (1 admin, 1 manager, 3 reps), 50 accounts, 200 contacts, 150 deals (100 open, 50 closed), 1000 activities, and 2 pipeline configurations.

**Design:**

```python
# backend/app/db/seed.py
import random
from faker import Faker
from datetime import date, timedelta

fake = Faker()

STAGES = [
    {"name": "Prospecting", "probability": 10, "category": "pipeline"},
    {"name": "Qualification", "probability": 20, "category": "pipeline"},
    {"name": "Needs Analysis", "probability": 40, "category": "best_case"},
    {"name": "Proposal", "probability": 60, "category": "best_case"},
    {"name": "Negotiation", "probability": 80, "category": "commit"},
    {"name": "Closed Won", "probability": 100, "category": "closed_won", "is_closed": True, "is_won": True},
    {"name": "Closed Lost", "probability": 0, "category": "closed_lost", "is_closed": True},
]

async def seed_deals(session, tenant_id, accounts, users):
    deals = []
    for i in range(150):
        is_closed = i >= 100
        stage = random.choice(STAGES[:5]) if not is_closed else random.choice(STAGES[5:])
        deal = Deal(
            tenant_id=tenant_id,
            crm_connection_id=mock_crm_id,
            external_id=f"SEED-{i:04d}",
            account_id=random.choice(accounts).id,
            owner_user_id=random.choice(users).id,
            name=f"{fake.company()} - {fake.bs().title()}",
            amount=random.randint(10, 500) * 1000,
            stage_name=stage["name"],
            close_date=date.today() + timedelta(days=random.randint(-90, 180)),
            is_closed=is_closed,
            is_won=stage.get("is_won", False),
            lifecycle={"days_in_pipeline": random.randint(5, 120), "activity_count_30d": random.randint(0, 15)},
        )
        deals.append(deal)
    session.add_all(deals)
```

**Testing:**
- `python -m app.db.seed` completes without errors
- Seed data passes all database constraints (foreign keys, unique indexes)
- Deal distribution: approximately 67% open, 33% closed; stages distributed realistically
- API endpoints return paginated seed data with correct filtering

---

## Phase 2: CRM Connectors

**Goal:** Implement OAuth 2.0 connector framework and first two CRM integrations (Salesforce, HubSpot). Enable bidirectional sync of Opportunity/Deal, Account/Company, Contact, and Activity records.

**Depends on:** Phase 1

### Task 2.1: CRM Connector Abstract Interface

**What:** Define the abstract base class that all CRM connectors implement, including OAuth flow, data sync, field mapping, and rate limit handling.

**Design:**

```python
# backend/app/connectors/base.py
from abc import ABC, abstractmethod
from dataclasses import dataclass
from typing import AsyncIterator

@dataclass
class CRMRecord:
    external_id: str
    object_type: str  # opportunity, account, contact, activity
    fields: dict      # Raw CRM fields
    updated_at: str   # ISO timestamp

class BaseCRMConnector(ABC):
    def __init__(self, connection: CRMConnection):
        self.connection = connection

    @abstractmethod
    async def get_auth_url(self, redirect_uri: str) -> str:
        """Generate OAuth 2.0 authorization URL."""

    @abstractmethod
    async def exchange_code(self, code: str, redirect_uri: str) -> TokenPair:
        """Exchange authorization code for access/refresh tokens."""

    @abstractmethod
    async def refresh_token(self) -> TokenPair:
        """Refresh expired access token."""

    @abstractmethod
    async def fetch_records(
        self, object_type: str, since: datetime | None = None
    ) -> AsyncIterator[list[CRMRecord]]:
        """Fetch records incrementally since last sync. Yields batches."""

    @abstractmethod
    def map_to_canonical(self, record: CRMRecord) -> dict:
        """Map CRM-native fields to canonical schema + crm_fields JSONB."""

    @abstractmethod
    async def test_connection(self) -> bool:
        """Verify the connection is alive and tokens are valid."""
```

```python
# backend/app/connectors/field_mapping.py
SALESFORCE_DEAL_MAPPING = {
    "Id": "external_id",
    "Name": "name",
    "Amount": "amount",
    "CurrencyIsoCode": "currency_code",
    "StageName": "stage_name",
    "CloseDate": "close_date",
    "IsClosed": "is_closed",
    "IsWon": "is_won",
    "Type": "deal_type",
    "LeadSource": "lead_source",
    "OwnerId": "owner_external_id",
    "AccountId": "_account_external_id",  # Resolved to FK post-insert
    # All other fields → crm_fields JSONB
}

HUBSPOT_DEAL_MAPPING = {
    "hs_object_id": "external_id",
    "dealname": "name",
    "amount": "amount",
    "deal_currency_code": "currency_code",
    "dealstage": "stage_name",
    "closedate": "close_date",
    "hs_is_closed": "is_closed",
    "hs_is_closed_won": "is_won",
    "dealtype": "deal_type",
    "hs_analytics_source": "lead_source",
    "hubspot_owner_id": "owner_external_id",
    # All other properties → crm_fields JSONB
}
```

**Testing:**
- Abstract interface enforces implementation of all required methods (test with incomplete subclass raises TypeError)
- Field mapping correctly separates canonical fields from crm_fields JSONB for both Salesforce and HubSpot schemas
- `map_to_canonical` preserves all unmapped fields in `crm_fields` without data loss
- Mapping handles null/missing fields gracefully (e.g., HubSpot deal with no `amount`)

### Task 2.2: Salesforce Connector

**What:** Implement the Salesforce CRM connector using simple-salesforce library. Support OAuth 2.0 via External Client Apps (Spring '26), incremental sync via Salesforce Replication API or SOQL `WHERE LastModifiedDate > :since`, and field mapping for Opportunity, Account, Contact, and Task/Event objects.

**Design:**

```python
# backend/app/connectors/salesforce.py
from simple_salesforce import Salesforce as SFClient
from app.connectors.base import BaseCRMConnector, CRMRecord

class SalesforceConnector(BaseCRMConnector):
    OBJECT_MAP = {
        "opportunity": {"soql_object": "Opportunity", "model": "deals"},
        "account": {"soql_object": "Account", "model": "accounts"},
        "contact": {"soql_object": "Contact", "model": "contacts"},
        "activity": {"soql_object": "Task", "model": "activities"},
    }

    async def fetch_records(self, object_type: str, since=None) -> AsyncIterator[list[CRMRecord]]:
        sf = SFClient(
            instance_url=self.connection.instance_url,
            session_id=self._get_access_token(),
            version="66.0",
        )
        soql_obj = self.OBJECT_MAP[object_type]["soql_object"]
        query = f"SELECT FIELDS(ALL) FROM {soql_obj}"
        if since:
            query += f" WHERE LastModifiedDate > {since.isoformat()}"
        query += " ORDER BY LastModifiedDate ASC LIMIT 2000"

        result = sf.query(query)
        while True:
            records = [
                CRMRecord(
                    external_id=r["Id"],
                    object_type=object_type,
                    fields=r,
                    updated_at=r["LastModifiedDate"],
                )
                for r in result["records"]
            ]
            yield records
            if result.get("nextRecordsUrl"):
                result = sf.query_more(result["nextRecordsUrl"])
            else:
                break

    def map_to_canonical(self, record: CRMRecord) -> dict:
        canonical = {}
        crm_fields = {}
        mapping = SALESFORCE_DEAL_MAPPING  # Select based on record.object_type
        for sf_field, value in record.fields.items():
            if sf_field in mapping:
                canonical[mapping[sf_field]] = value
            elif sf_field not in ("attributes",):
                crm_fields[sf_field] = value
        canonical["crm_fields"] = crm_fields
        return canonical
```

**Testing:**
- OAuth flow: redirect to Salesforce login, callback receives code, exchange returns valid tokens
- Token refresh works before expiry and retries on 401
- Incremental sync: first sync fetches all records; subsequent syncs fetch only records modified since `last_sync_at`
- Field mapping: Salesforce Opportunity record maps to Deal with all canonical fields populated and custom fields (`Custom_ARR__c`, etc.) in `crm_fields`
- Pagination: handles Salesforce's `nextRecordsUrl` cursor correctly across 3+ pages
- Rate limit handling: 429 responses trigger exponential backoff with jitter
- Integration test with Salesforce sandbox (or mock): sync 100 Opportunities creates 100 deals in the database

### Task 2.3: HubSpot Connector

**What:** Implement the HubSpot CRM connector using the official hubspot-api-client Python SDK. Support OAuth 2.1-style token management (2026-03 API version), incremental sync via `properties` and `after` cursor, and field mapping for Deal, Company, Contact, and Engagement objects.

**Design:**

```python
# backend/app/connectors/hubspot.py
from hubspot import HubSpot
from hubspot.crm.deals import ApiException
from app.connectors.base import BaseCRMConnector, CRMRecord

class HubSpotConnector(BaseCRMConnector):
    OBJECT_MAP = {
        "opportunity": {"hs_object": "deals", "properties": ["dealname", "amount", "dealstage", "closedate", ...]},
        "account": {"hs_object": "companies", "properties": ["name", "industry", "numberofemployees", ...]},
        "contact": {"hs_object": "contacts", "properties": ["firstname", "lastname", "email", ...]},
        "activity": {"hs_object": "engagements", "properties": ["type", "timestamp", "duration", ...]},
    }

    async def fetch_records(self, object_type: str, since=None) -> AsyncIterator[list[CRMRecord]]:
        client = HubSpot(access_token=self._get_access_token())
        hs_config = self.OBJECT_MAP[object_type]

        if since:
            # Use search API with lastmodifieddate filter
            filter_groups = [{"filters": [
                {"propertyName": "hs_lastmodifieddate", "operator": "GTE", "value": int(since.timestamp() * 1000)}
            ]}]
            results = client.crm.deals.search_api.do_search(
                public_object_search_request={"filterGroups": filter_groups, "properties": hs_config["properties"], "limit": 100}
            )
        else:
            results = client.crm.deals.basic_api.get_page(
                properties=hs_config["properties"], limit=100
            )

        while True:
            records = [
                CRMRecord(
                    external_id=str(r.id),
                    object_type=object_type,
                    fields=r.properties,
                    updated_at=r.properties.get("hs_lastmodifieddate", ""),
                )
                for r in results.results
            ]
            yield records
            if results.paging and results.paging.next:
                results = client.crm.deals.basic_api.get_page(
                    properties=hs_config["properties"], limit=100, after=results.paging.next.after
                )
            else:
                break
```

**Testing:**
- OAuth flow: redirect to HubSpot authorization, callback receives code, exchange returns valid tokens (parameters in request body per 2026-03 spec)
- Token refresh: 30-minute access token expiry triggers automatic refresh
- Incremental sync with `hs_lastmodifieddate` filter correctly fetches only modified records
- Field mapping: HubSpot Deal properties map to Deal canonical fields; `hs_analytics_source`, `hs_projected_amount`, etc. land in `crm_fields`
- Association API: deal-contact relationships synced into `deal_contacts` table
- Rate limit: 429 responses with `Retry-After` header respected
- Integration test with HubSpot sandbox: sync 50 Deals creates 50 deals in the database

### Task 2.4: CRM Sync Orchestration

**What:** Build the Celery task that orchestrates periodic CRM sync: discovers what to sync, delegates to the appropriate connector, upserts records, records deal_events for changes, and updates `crm_connections.last_sync_at`.

**Design:**

```python
# backend/app/tasks/crm_sync.py
from celery import shared_task
from app.connectors import get_connector

@shared_task(bind=True, max_retries=3, default_retry_delay=60)
def sync_crm_connection(self, connection_id: str):
    """Sync all object types for a CRM connection."""
    connection = get_connection(connection_id)
    connector = get_connector(connection)

    for object_type in ["account", "contact", "opportunity", "activity"]:
        try:
            sync_object_type(connector, connection, object_type)
        except RateLimitError as e:
            self.retry(countdown=e.retry_after)
        except TokenExpiredError:
            connector.refresh_token()
            self.retry(countdown=5)

    connection.last_sync_at = utcnow()
    connection.sync_status = "active"
    db.commit()

def sync_object_type(connector, connection, object_type):
    """Upsert records and emit deal_events for changes."""
    for batch in connector.fetch_records(object_type, since=connection.last_sync_at):
        for record in batch:
            canonical = connector.map_to_canonical(record)
            existing = find_by_external_id(connection.id, canonical["external_id"])

            if existing:
                changes = detect_changes(existing, canonical)
                update_record(existing, canonical)
                for change in changes:
                    emit_deal_event(existing.id, change)
            else:
                create_record(connection, canonical)

@shared_task
def schedule_all_syncs():
    """Periodic task: enqueue sync for all active CRM connections."""
    connections = get_active_connections()
    for conn in connections:
        sync_crm_connection.delay(str(conn.id))
```

**Testing:**
- Full sync: first-time sync of 100 Salesforce Opportunities creates 100 deals
- Incremental sync: modifying 5 deals in CRM and re-syncing creates/updates only those 5
- Change detection: stage change from "Qualification" to "Proposal" emits a `stage_changed` deal_event with `previous_stage` and `new_stage`
- Close date slip: changing close date emits `close_date_changed` event with slip count
- Idempotency: running sync twice without CRM changes produces no new records or events
- Token refresh: expired token triggers refresh and retry; sync completes successfully
- Error handling: network timeout retries 3 times; persistent failure marks connection as `error`
- Celery beat: `schedule_all_syncs` runs every 15 minutes and enqueues one task per active connection

---

## Phase 3: Frontend Shell & Deal Dashboard

**Goal:** Build the core frontend application shell with navigation, authentication, and the primary deal dashboard view.

**Depends on:** Phase 1

### Task 3.1: Next.js Application Setup

**What:** Scaffold the Next.js 14+ application with app router, TypeScript, Tailwind CSS, and authentication integration (Auth0/Clerk).

**Design:**

```typescript
// frontend/src/app/layout.tsx
import { ClerkProvider } from "@clerk/nextjs";

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <ClerkProvider>
      <html lang="en">
        <body className="bg-gray-50 text-gray-900">
          <AppShell>{children}</AppShell>
        </body>
      </html>
    </ClerkProvider>
  );
}
```

```typescript
// frontend/src/components/common/AppShell.tsx
export function AppShell({ children }: { children: React.ReactNode }) {
  return (
    <div className="flex h-screen">
      <Sidebar items={[
        { label: "Pipeline", href: "/pipeline", icon: FunnelIcon },
        { label: "Deals", href: "/deals", icon: BriefcaseIcon },
        { label: "Forecasts", href: "/forecasts", icon: ChartBarIcon },
        { label: "Alerts", href: "/alerts", icon: BellIcon },
        { label: "Settings", href: "/settings", icon: CogIcon },
      ]} />
      <main className="flex-1 overflow-y-auto p-6">{children}</main>
    </div>
  );
}
```

**Testing:**
- Application starts on `http://localhost:3000` without errors
- Unauthenticated users redirected to login page
- Authenticated users see the sidebar navigation and main content area
- Navigation between pages updates URL and renders correct content
- Responsive layout: sidebar collapses on mobile viewports

### Task 3.2: Deal List & Detail Views

**What:** Build the deal list table with sorting, filtering, pagination, and inline score display. Build the deal detail view showing full deal information, score history chart, activity timeline, and qualification status.

**Design:**

```typescript
// frontend/src/components/deals/DealTable.tsx
interface DealRow {
  id: string;
  name: string;
  accountName: string;
  amount: number;
  stageName: string;
  closeDate: string;
  winProbability: number | null;
  riskLevel: string | null;
  dqScore: number | null;
  explanation: string | null;
  ownerName: string;
}

export function DealTable({ deals, sortBy, onSort, onFilter }: DealTableProps) {
  return (
    <table className="w-full">
      <thead>
        <tr>
          <SortableHeader field="name" label="Deal" sortBy={sortBy} onSort={onSort} />
          <SortableHeader field="amount" label="Amount" sortBy={sortBy} onSort={onSort} />
          <SortableHeader field="stageName" label="Stage" sortBy={sortBy} onSort={onSort} />
          <SortableHeader field="closeDate" label="Close Date" sortBy={sortBy} onSort={onSort} />
          <SortableHeader field="winProbability" label="Win %" sortBy={sortBy} onSort={onSort} />
          <th>Risk</th>
          <th>DQ Score</th>
          <th>Owner</th>
        </tr>
      </thead>
      <tbody>
        {deals.map(deal => (
          <tr key={deal.id} className="hover:bg-gray-50 cursor-pointer" onClick={() => navigate(`/deals/${deal.id}`)}>
            <td>{deal.name}</td>
            <td className="text-right">{formatCurrency(deal.amount)}</td>
            <td><StageBadge stage={deal.stageName} /></td>
            <td>{formatDate(deal.closeDate)}</td>
            <td><ProbabilityBar value={deal.winProbability} /></td>
            <td><RiskBadge level={deal.riskLevel} /></td>
            <td><QualityScore score={deal.dqScore} /></td>
            <td>{deal.ownerName}</td>
          </tr>
        ))}
      </tbody>
    </table>
  );
}
```

```typescript
// frontend/src/components/deals/ScoreExplanationCard.tsx
export function ScoreExplanationCard({ scoring }: { scoring: DealScoring }) {
  return (
    <div className="rounded-lg border p-4">
      <div className="flex items-center justify-between">
        <h3 className="text-sm font-medium text-gray-500">AI Score</h3>
        <ProbabilityBadge value={scoring.win_probability} trend={scoring.trend} />
      </div>
      <p className="mt-2 text-sm text-gray-700">{scoring.explanation}</p>
      <div className="mt-3 flex gap-4">
        <FactorList label="Helping" factors={scoring.top_factors.filter(f => f.direction === "positive")} color="green" />
        <FactorList label="Hurting" factors={scoring.top_factors.filter(f => f.direction === "negative")} color="red" />
      </div>
    </div>
  );
}
```

**Testing:**
- Deal list loads 150 seed deals with correct column data
- Sorting by amount (ascending/descending) reorders rows correctly
- Filtering by stage shows only matching deals; filter by risk level shows only high/critical
- Pagination: 20 deals per page; next/previous buttons work
- Deal detail page shows all canonical fields, MEDDIC qualification, and score explanation card
- Score history chart renders with Recharts (line chart of win_probability over time)
- Empty state: new tenant with no deals shows "Connect your CRM to get started" prompt

### Task 3.3: Pipeline Visualisation

**What:** Build the pipeline funnel/board view showing deals grouped by stage with aggregate metrics per stage.

**Design:**

```typescript
// frontend/src/components/pipeline/PipelineBoard.tsx
export function PipelineBoard({ stages, deals }: PipelineBoardProps) {
  return (
    <div className="flex gap-4 overflow-x-auto">
      {stages.map(stage => {
        const stageDeals = deals.filter(d => d.stageName === stage.name);
        const totalAmount = stageDeals.reduce((sum, d) => sum + (d.amount || 0), 0);
        return (
          <div key={stage.name} className="min-w-[280px] flex-shrink-0">
            <div className="rounded-t-lg bg-gray-100 p-3">
              <h3 className="font-medium">{stage.name}</h3>
              <p className="text-sm text-gray-500">
                {stageDeals.length} deals · {formatCurrency(totalAmount)}
              </p>
            </div>
            <div className="space-y-2 p-2">
              {stageDeals.map(deal => (
                <DealCard key={deal.id} deal={deal} />
              ))}
            </div>
          </div>
        );
      })}
    </div>
  );
}
```

**Testing:**
- Pipeline board displays all stages in correct order (by `display_order`)
- Each stage column shows deal count and total amount
- Deal cards show name, amount, close date, and risk level indicator
- Clicking a deal card navigates to deal detail page
- Empty stage shows "No deals" placeholder
- Pipeline board scrolls horizontally on narrow screens

---

## Phase 4: ML Scoring Pipeline

**Goal:** Build the ML model training pipeline, batch scoring, SHAP explanation generation, and score delivery to the frontend.

**Depends on:** Phase 2

### Task 4.1: Feature Engineering

**What:** Build the feature extraction pipeline that transforms raw deal data (canonical fields + lifecycle metrics + activity history + deal_events) into the feature vector used by the ML model.

**Design:**

```python
# backend/app/ml/features.py
from dataclasses import dataclass
import numpy as np

FEATURE_DEFINITIONS = [
    # Deal attributes
    "amount_log",                  # log(amount + 1) — normalised deal size
    "days_in_pipeline",            # Total days since deal entered pipeline
    "days_in_current_stage",       # Days in current stage
    "stage_order_normalised",      # Stage position / total stages (0.0 to 1.0)
    "crm_probability",            # CRM-native probability (0-100)

    # Activity signals
    "activity_count_30d",          # Total activities in last 30 days
    "call_count_30d",
    "email_count_30d",
    "meeting_count_30d",
    "days_since_last_activity",    # Days since most recent activity
    "activity_velocity",           # Activities per week over deal lifetime

    # Relationship signals
    "contact_count",               # Number of contacts associated with deal
    "has_decision_maker",          # Boolean: any contact with seniority C-level/VP/Director
    "has_champion",                # Boolean: MEDDIC champion field is populated

    # Momentum signals
    "close_date_slip_count",       # Number of times close date has been pushed
    "stage_change_count",          # Number of stage transitions
    "amount_change_count",         # Number of amount modifications
    "days_until_close",            # Calendar days until close_date (negative if past due)
    "is_close_date_past_due",      # Boolean: close_date < today and deal is still open

    # Data quality
    "dq_score",                    # CRM data quality score (0-100) — Phase 8 populates this

    # Historical context
    "account_win_rate",            # Historical win rate for this account
    "owner_win_rate",              # Historical win rate for this rep
    "pipeline_avg_deal_size",      # Average deal size in this pipeline
]

class FeatureExtractor:
    def __init__(self, tenant_id: str, db_session):
        self.tenant_id = tenant_id
        self.db = db_session

    async def extract(self, deal: Deal) -> np.ndarray:
        """Extract feature vector for a single deal."""
        lifecycle = deal.lifecycle or {}
        qualification = deal.qualification or {}

        features = {
            "amount_log": np.log1p(float(deal.amount or 0)),
            "days_in_pipeline": lifecycle.get("days_in_pipeline", 0),
            "days_in_current_stage": lifecycle.get("days_in_current_stage", 0),
            "stage_order_normalised": await self._stage_position(deal),
            "activity_count_30d": lifecycle.get("activity_count_30d", 0),
            "days_since_last_activity": lifecycle.get("days_since_last_activity", 999),
            "contact_count": lifecycle.get("contact_count", 0),
            "has_decision_maker": 1 if lifecycle.get("has_decision_maker") else 0,
            "has_champion": 1 if qualification.get("champion") else 0,
            "close_date_slip_count": lifecycle.get("close_date_slip_count", 0),
            "days_until_close": (deal.close_date - date.today()).days if deal.close_date else 999,
            "is_close_date_past_due": 1 if deal.close_date and deal.close_date < date.today() else 0,
            "account_win_rate": await self._account_win_rate(deal.account_id),
            "owner_win_rate": await self._owner_win_rate(deal.owner_user_id),
            # ... remaining features
        }
        return np.array([features[f] for f in FEATURE_DEFINITIONS])

    async def extract_batch(self, deals: list[Deal]) -> np.ndarray:
        """Extract features for a batch of deals. Returns (n_deals, n_features) array."""
        return np.array([await self.extract(d) for d in deals])
```

**Testing:**
- Feature extraction for a single deal returns a numpy array of length len(FEATURE_DEFINITIONS)
- `amount_log` correctly applies log1p transform (amount=100000 -> ~11.51)
- `days_since_last_activity` defaults to 999 when no activities exist
- `has_champion` returns 1 when `qualification.champion` is populated, 0 otherwise
- `stage_order_normalised` returns 0.0 for first stage, 1.0 for last open stage
- `is_close_date_past_due` returns 1 for deals with close_date before today
- Batch extraction of 150 deals returns shape (150, n_features) with no NaN values
- Feature extraction handles missing data gracefully (null amounts, no activities)

### Task 4.2: Model Training Pipeline

**What:** Build the model training pipeline that takes historical closed deals, trains an XGBoost binary classifier (win/loss), evaluates performance, and registers the trained model.

**Design:**

```python
# backend/app/ml/training.py
import xgboost as xgb
from sklearn.model_selection import StratifiedKFold, cross_val_score
from sklearn.metrics import roc_auc_score, f1_score, log_loss
import joblib

class ModelTrainer:
    DEFAULT_PARAMS = {
        "objective": "binary:logistic",
        "eval_metric": "logloss",
        "max_depth": 6,
        "learning_rate": 0.05,
        "n_estimators": 500,
        "subsample": 0.8,
        "colsample_bytree": 0.8,
        "min_child_weight": 5,
        "scale_pos_weight": 1.0,  # Adjusted based on class balance
        "random_state": 42,
    }

    async def train(self, tenant_id: str) -> MLModel:
        """Train a new model on historical closed deals."""
        # 1. Fetch closed deals with known outcomes
        closed_deals = await self.db.fetch_closed_deals(tenant_id, min_count=75)
        if len(closed_deals) < 75:
            raise InsufficientDataError(f"Need 75+ closed deals, have {len(closed_deals)}")

        # 2. Extract features and labels
        extractor = FeatureExtractor(tenant_id, self.db)
        X = await extractor.extract_batch(closed_deals)
        y = np.array([1 if d.is_won else 0 for d in closed_deals])

        # 3. Adjust class balance
        pos_ratio = y.sum() / len(y)
        params = {**self.DEFAULT_PARAMS, "scale_pos_weight": (1 - pos_ratio) / pos_ratio}

        # 4. Cross-validate
        model = xgb.XGBClassifier(**params)
        cv = StratifiedKFold(n_splits=5, shuffle=True, random_state=42)
        cv_scores = cross_val_score(model, X, y, cv=cv, scoring="roc_auc")

        # 5. Train final model on all data
        model.fit(X, y)

        # 6. Compute metrics
        y_pred_proba = model.predict_proba(X)[:, 1]
        metrics = {
            "auc_roc": float(roc_auc_score(y, y_pred_proba)),
            "auc_roc_cv_mean": float(cv_scores.mean()),
            "auc_roc_cv_std": float(cv_scores.std()),
            "f1": float(f1_score(y, (y_pred_proba > 0.5).astype(int))),
            "log_loss": float(log_loss(y, y_pred_proba)),
            "deal_count": len(closed_deals),
            "win_rate": float(pos_ratio),
        }

        # 7. Save model artifact
        artifact_path = f"models/{tenant_id}/xgboost-v{next_version}.joblib"
        joblib.dump(model, artifact_path)

        # 8. Register in database
        ml_model = MLModel(
            tenant_id=tenant_id,
            model_type="xgboost",
            model_version=f"v{next_version}",
            status="active",
            config={"hyperparameters": params, "feature_names": FEATURE_DEFINITIONS, "metrics": metrics},
            model_artifact_path=artifact_path,
        )
        # Retire previous active model
        await self.db.retire_active_models(tenant_id)
        self.db.add(ml_model)
        return ml_model
```

**Testing:**
- Training with 150 closed deals (100 won, 50 lost) completes in under 60 seconds
- Model achieves AUC-ROC > 0.65 on 5-fold cross-validation with seed data
- Training with fewer than 75 closed deals raises `InsufficientDataError`
- Model artifact is saved and can be loaded with `joblib.load`
- New model registered with status "active"; previous model status set to "retired"
- Metrics include `auc_roc`, `f1`, `log_loss`, `deal_count`
- Feature importance ranking is available via `model.feature_importances_`

### Task 4.3: Batch Scoring & SHAP Explanations

**What:** Build the batch scoring pipeline that applies the active model to all open deals, generates SHAP explanations, updates `deals.scoring` JSONB, appends to `score_history`, and emits `score_computed` deal_events.

**Design:**

```python
# backend/app/ml/scoring.py
import shap
import xgboost as xgb

class BatchScorer:
    async def score_all_open_deals(self, tenant_id: str):
        """Score all open deals for a tenant."""
        model_record = await self.db.get_active_model(tenant_id)
        if not model_record:
            return  # No model trained yet

        model = joblib.load(model_record.model_artifact_path)
        explainer = shap.TreeExplainer(model)

        open_deals = await self.db.get_open_deals(tenant_id)
        extractor = FeatureExtractor(tenant_id, self.db)

        for batch in chunked(open_deals, 500):
            X = await extractor.extract_batch(batch)
            probabilities = model.predict_proba(X)[:, 1]
            shap_values = explainer.shap_values(X)

            for deal, prob, shap_row in zip(batch, probabilities, shap_values):
                score_data = self._build_score(deal, prob, shap_row, model_record)
                deal.scoring = score_data
                self._append_score_history(deal, score_data, model_record)
                self._emit_score_event(deal, score_data)

    def _build_score(self, deal, probability, shap_row, model_record):
        feature_impacts = sorted(
            zip(FEATURE_DEFINITIONS, shap_row),
            key=lambda x: abs(x[1]),
            reverse=True,
        )
        top_factors = [
            {"feature": name, "impact": round(float(val), 4), "direction": "positive" if val > 0 else "negative"}
            for name, val in feature_impacts[:6]
        ]
        previous_prob = (deal.scoring or {}).get("win_probability")
        trend = self._compute_trend(probability, previous_prob)
        risk_level = self._compute_risk(probability, trend)

        explanation = self._generate_explanation(probability, previous_prob, top_factors, deal)

        return {
            "win_probability": round(float(probability), 4),
            "confidence_lower": round(float(probability - 0.1), 4),  # Placeholder; Phase 10 adds calibrated intervals
            "confidence_upper": round(float(probability + 0.1), 4),
            "risk_level": risk_level,
            "model_id": str(model_record.id),
            "model_version": model_record.model_version,
            "scored_at": utcnow().isoformat(),
            "explanation": explanation,
            "top_factors": top_factors,
            "previous_probability": previous_prob,
            "trend": trend,
        }

    def _generate_explanation(self, prob, prev_prob, factors, deal):
        """Generate natural-language explanation from SHAP values."""
        positives = [f for f in factors if f["direction"] == "positive"][:2]
        negatives = [f for f in factors if f["direction"] == "negative"][:2]

        parts = [f"Score at {prob*100:.0f}%"]
        if prev_prob is not None:
            delta = prob - prev_prob
            if abs(delta) > 0.02:
                direction = "up" if delta > 0 else "down"
                parts[0] += f" ({direction} {abs(delta)*100:.0f}pts)"

        if positives:
            pos_labels = [self._humanize_feature(f["feature"]) for f in positives]
            parts.append(f"helped by {' and '.join(pos_labels)}")
        if negatives:
            neg_labels = [self._humanize_feature(f["feature"]) for f in negatives]
            parts.append(f"hindered by {' and '.join(neg_labels)}")

        return "; ".join(parts)
```

**Testing:**
- Batch scoring of 100 open deals completes in under 30 seconds
- Each deal's `scoring` JSONB is updated with `win_probability`, `risk_level`, `explanation`, `top_factors`
- SHAP values sum to approximate the model's log-odds prediction (SHAP completeness property)
- Explanation text includes human-readable factor names (e.g., "no activity in 21 days" not "days_since_last_activity")
- `score_history` gets one new row per scored deal
- `deal_events` gets one `score_computed` event per deal
- Risk level assignment: probability >= 0.7 = low, 0.4-0.7 = medium, 0.2-0.4 = high, < 0.2 = critical
- Trend detection: probability dropped > 5pts from previous = "declining", increased > 5pts = "improving", else "stable"
- Scoring with no active model is a no-op (no errors)

### Task 4.4: Scheduled Scoring & Lifecycle Metric Refresh

**What:** Create Celery tasks to refresh deal lifecycle metrics (days_in_stage, days_since_activity, etc.) and run batch scoring daily.

**Design:**

```python
# backend/app/tasks/batch_scoring.py

@shared_task
def refresh_lifecycle_metrics(tenant_id: str):
    """Recompute lifecycle JSONB for all open deals."""
    open_deals = get_open_deals(tenant_id)
    for deal in open_deals:
        deal.lifecycle = {
            "days_in_pipeline": (date.today() - deal.created_at.date()).days,
            "days_in_current_stage": compute_days_in_stage(deal),
            "days_since_last_activity": compute_days_since_activity(deal),
            "close_date_slip_count": count_close_date_events(deal.id),
            "stage_change_count": count_stage_events(deal.id),
            "activity_count_30d": count_recent_activities(deal.id, days=30),
            "contact_count": count_deal_contacts(deal.id),
            "has_decision_maker": has_senior_contact(deal.id),
        }
    db.commit()

@shared_task
def run_daily_scoring():
    """Refresh metrics and score all tenants."""
    for tenant in get_active_tenants():
        refresh_lifecycle_metrics.delay(str(tenant.id))
        score_all_open_deals.apply_async(
            args=[str(tenant.id)],
            countdown=120,  # Wait for metrics refresh
        )
```

**Testing:**
- Lifecycle metrics update correctly (days_in_stage increments daily)
- Daily scoring runs for each active tenant
- Scoring runs after lifecycle refresh (120-second delay)
- Lifecycle refresh handles deals with no activities (days_since_last_activity = large number)
- Celery beat schedule: `run_daily_scoring` at 02:00 UTC daily

---

## Phase 5: Forecast Engine

**Goal:** Build the forecast submission, roll-up, snapshot, and period management system. Enable reps to submit forecasts and managers to view AI-adjusted team roll-ups.

**Depends on:** Phase 4

### Task 5.1: Forecast Period Management

**What:** Create and manage fiscal periods (weekly, monthly, quarterly) per tenant. Support custom fiscal year start months. Auto-detect current period.

**Design:**

```python
# backend/app/services/forecast_service.py

class ForecastService:
    async def generate_periods(self, tenant_id: str, fiscal_year_start_month: int = 1):
        """Generate quarterly and monthly periods for the fiscal year."""
        year = date.today().year
        for q in range(1, 5):
            start_month = fiscal_year_start_month + (q - 1) * 3
            if start_month > 12:
                start_month -= 12
                fy = year + 1
            else:
                fy = year
            start_date = date(fy if start_month >= fiscal_year_start_month else fy, start_month, 1)
            end_date = (start_date + timedelta(days=93)).replace(day=1) - timedelta(days=1)

            period = ForecastPeriod(
                tenant_id=tenant_id,
                period_type="quarterly",
                period_label=f"Q{q} FY{year}",
                fiscal_year=year,
                fiscal_quarter=q,
                start_date=start_date,
                end_date=end_date,
                is_current=(start_date <= date.today() <= end_date),
            )
            self.db.add(period)
```

**Testing:**
- Period generation creates 4 quarterly periods for a fiscal year
- Custom fiscal year start (e.g., February) correctly offsets quarter boundaries
- `is_current` flag set on exactly one period
- Duplicate period generation is idempotent (unique constraint on tenant_id + period_type + start_date)
- ASC 606 alignment: periods match standard quarterly accounting periods for January fiscal year

### Task 5.2: Forecast Submission & AI Adjustment

**What:** API for reps and managers to submit forecast amounts (commit, best_case, most_likely). AI adjustment computes an ML-adjusted amount based on the scoring of deals categorised under each forecast category.

**Design:**

```python
# backend/app/services/forecast_service.py

async def submit_forecast(self, tenant_id, user_id, period_id, category, amount):
    """Submit a forecast and compute AI adjustment."""
    # Get deals in this category for this period
    deals = await self.get_deals_for_category(tenant_id, period_id, category)
    ai_total = sum(
        float(d.amount or 0) * float((d.scoring or {}).get("win_probability", 0))
        for d in deals
    )

    adjustment_reason = None
    if abs(ai_total - amount) > amount * 0.05:
        high_risk = [d for d in deals if (d.scoring or {}).get("risk_level") in ("high", "critical")]
        adjustment_reason = (
            f"AI adjusted {category} to {format_currency(ai_total)} "
            f"({len(high_risk)} high-risk deals reduce confidence)"
        )

    submission = ForecastSubmission(
        tenant_id=tenant_id,
        period_id=period_id,
        submitted_by=user_id,
        data={
            "category": category,
            "submitted_amount": float(amount),
            "ai_adjusted_amount": round(ai_total, 2),
            "ai_adjustment_reason": adjustment_reason,
            "deal_breakdown": [
                {"deal_id": str(d.id), "amount": float(d.amount), "category": category}
                for d in deals
            ],
        },
    )
    self.db.add(submission)
    return submission
```

**Testing:**
- Forecast submission creates a record with both human and AI amounts
- AI adjustment: if 3 deals are "high risk", AI total is lower than submitted (weighted by probability)
- Adjustment reason explains the delta in plain language
- Deal breakdown included in submission data
- Multiple submissions for the same period + user + category: all preserved (append-only)
- Latest submission per category used for roll-up display

### Task 5.3: Team Roll-Up & Snapshot Capture

**What:** Aggregate individual rep forecasts into team and company roll-ups. Capture daily snapshots of aggregate forecast numbers for historical tracking.

**Design:**

```python
# backend/app/tasks/snapshot_capture.py

@shared_task
def capture_forecast_snapshot(tenant_id: str):
    """Capture daily snapshot of forecast aggregates for each active period."""
    current_periods = get_current_periods(tenant_id)
    for period in current_periods:
        open_deals = get_open_deals_in_period(tenant_id, period)
        submissions = get_latest_submissions(tenant_id, period.id)

        snapshot_data = {
            "total_pipeline": sum(d.amount or 0 for d in open_deals),
            "total_commit": sum_by_category(submissions, "commit"),
            "total_best_case": sum_by_category(submissions, "best_case"),
            "total_most_likely": sum_by_category(submissions, "most_likely"),
            "ai_predicted": {
                "total": sum(
                    float(d.amount or 0) * float((d.scoring or {}).get("win_probability", 0))
                    for d in open_deals
                ),
            },
            "deal_count": len(open_deals),
            "pipeline_coverage": calculate_coverage(open_deals, tenant_id),
        }

        snapshot = ForecastSnapshot(
            tenant_id=tenant_id,
            period_id=period.id,
            snapshot_date=date.today(),
            data=snapshot_data,
        )
        db.merge(snapshot)  # Upsert on (tenant_id, period_id, snapshot_date)
```

**Testing:**
- Daily snapshot captures total pipeline, commit, best_case, most_likely, and AI predicted values
- Pipeline coverage = total pipeline / team quota
- Snapshot for the same date updates (upsert), not duplicates
- Roll-up aggregation matches sum of individual rep submissions
- Snapshot data includes deal_count for denominator context
- Celery beat: `capture_forecast_snapshot` runs daily at 03:00 UTC for each active tenant

### Task 5.4: Forecast Dashboard Frontend

**What:** Build the forecast dashboard UI showing team roll-up table, forecast-over-time chart, and pipeline coverage gauge.

**Design:**

```typescript
// frontend/src/components/forecasts/ForecastRollup.tsx
export function ForecastRollup({ period, submissions, aiPrediction }: ForecastRollupProps) {
  return (
    <div className="grid grid-cols-4 gap-4">
      <MetricCard label="Pipeline" value={period.totalPipeline} />
      <MetricCard label="Commit" value={submissions.commit} aiValue={aiPrediction.commit} />
      <MetricCard label="Best Case" value={submissions.bestCase} aiValue={aiPrediction.bestCase} />
      <MetricCard label="Most Likely" value={submissions.mostLikely} aiValue={aiPrediction.mostLikely} />
    </div>
  );
}
```

```typescript
// frontend/src/components/forecasts/ForecastChart.tsx
export function ForecastOverTimeChart({ snapshots }: { snapshots: ForecastSnapshot[] }) {
  return (
    <ResponsiveContainer width="100%" height={300}>
      <LineChart data={snapshots}>
        <XAxis dataKey="snapshotDate" />
        <YAxis tickFormatter={formatCompactCurrency} />
        <Line dataKey="data.total_commit" stroke="#2563EB" name="Commit" />
        <Line dataKey="data.ai_predicted.total" stroke="#7C3AED" name="AI Predicted" strokeDasharray="5 5" />
        <Line dataKey="data.total_pipeline" stroke="#9CA3AF" name="Pipeline" />
        <Tooltip formatter={formatCurrency} />
        <Legend />
      </LineChart>
    </ResponsiveContainer>
  );
}
```

**Testing:**
- Forecast roll-up table displays all four categories with human and AI values side by side
- When AI value differs from human value by > 5%, the delta is highlighted with an explanation tooltip
- Forecast-over-time chart shows commit, AI predicted, and pipeline lines over 12 weeks
- Pipeline coverage gauge shows green (> 3x), yellow (2-3x), red (< 2x)
- Rep-level breakdown table shows each rep's submission and the AI adjustment
- Manager can drill down from team total to individual rep view

---

## Phase 6: Alerts & Notifications

**Goal:** Implement configurable alert rules that detect at-risk deals and deliver notifications via Slack and in-app feed.

**Depends on:** Phase 4

### Task 6.1: Alert Rule Engine

**What:** Build the rule evaluation engine that checks all open deals against configurable alert rules after each scoring cycle.

**Design:**

```python
# backend/app/services/alert_service.py

class AlertRule:
    RULE_TYPES = {
        "no_activity": lambda deal, threshold: (deal.lifecycle or {}).get("days_since_last_activity", 0) > threshold,
        "probability_drop": lambda deal, threshold: _probability_dropped(deal, threshold),
        "close_date_slip": lambda deal, threshold: (deal.lifecycle or {}).get("close_date_slip_count", 0) >= threshold,
        "stage_stagnation": lambda deal, threshold: (deal.lifecycle or {}).get("days_in_current_stage", 0) > threshold,
        "dq_score_low": lambda deal, threshold: float((deal.quality or {}).get("score", 100)) < threshold,
    }

class AlertEvaluator:
    async def evaluate_all_rules(self, tenant_id: str):
        """Evaluate all active alert rules against all open deals."""
        rules = await self.db.get_active_rules(tenant_id)
        open_deals = await self.db.get_open_deals(tenant_id)

        for rule in rules:
            evaluator = AlertRule.RULE_TYPES[rule.rule_type]
            for deal in open_deals:
                if evaluator(deal, rule.threshold_value):
                    existing = await self.db.find_recent_alert(deal.id, rule.id, hours=24)
                    if not existing:
                        alert = Alert(
                            tenant_id=tenant_id,
                            deal_id=deal.id,
                            alert_type=rule.rule_type,
                            severity=self._compute_severity(rule, deal),
                            message=self._format_message(rule, deal),
                            details={"threshold": rule.threshold_value, "actual_value": self._get_actual(rule, deal)},
                        )
                        self.db.add(alert)
                        await self._deliver(alert, rule.channels)
```

**Testing:**
- "No activity" rule fires for deals with 15+ days since last activity when threshold is 14
- "Probability drop" rule fires when win_probability drops by 16+ points when threshold is 15
- "Close date slip" rule fires when slip_count >= 2
- Duplicate suppression: same alert for same deal + rule not re-created within 24 hours
- Custom severity: no_activity > 30 days = critical, > 14 days = warning

### Task 6.2: Slack Integration

**What:** Implement Slack webhook delivery for alerts with formatted messages and deal links.

**Design:**

```python
# backend/app/services/slack_service.py
import httpx

class SlackService:
    async def send_alert(self, webhook_url: str, alert: Alert, deal: Deal):
        risk_emoji = {"critical": ":red_circle:", "warning": ":large_yellow_circle:", "info": ":large_blue_circle:"}
        payload = {
            "blocks": [
                {"type": "header", "text": {"type": "plain_text", "text": f"Deal Alert: {deal.name}"}},
                {"type": "section", "fields": [
                    {"type": "mrkdwn", "text": f"*Type:* {alert.alert_type.replace('_', ' ').title()}"},
                    {"type": "mrkdwn", "text": f"*Amount:* {format_currency(deal.amount)}"},
                    {"type": "mrkdwn", "text": f"*Stage:* {deal.stage_name}"},
                    {"type": "mrkdwn", "text": f"*Risk:* {risk_emoji.get(alert.severity, '')} {alert.severity.title()}"},
                ]},
                {"type": "section", "text": {"type": "mrkdwn", "text": alert.message}},
                {"type": "actions", "elements": [
                    {"type": "button", "text": {"type": "plain_text", "text": "View Deal"},
                     "url": f"{app_url}/deals/{deal.id}"}
                ]},
            ]
        }
        async with httpx.AsyncClient() as client:
            resp = await client.post(webhook_url, json=payload)
            return resp.status_code == 200
```

**Testing:**
- Slack message contains deal name, amount, stage, risk level, and alert type
- "View Deal" button links to the correct deal detail page
- Failed Slack delivery (non-200) does not prevent alert creation in database
- Slack webhook URL stored encrypted in tenant settings
- Rate limiting: max 1 Slack message per deal per rule per 24 hours

### Task 6.3: In-App Alert Feed

**What:** Build the frontend alert feed showing recent alerts with dismiss functionality.

**Testing:**
- Alert feed shows most recent 50 alerts, newest first
- Each alert shows deal name, alert type, severity badge, and timestamp
- Clicking an alert navigates to the deal detail page
- "Dismiss" button marks alert as dismissed; dismissed alerts hidden from feed
- Unread alert count shown as badge on sidebar navigation icon
- Filter by severity (critical, warning, info) and by alert type

---

## Phase 7: Forecast Accuracy Dashboard

**Goal:** Build the retrospective accuracy analysis that compares historical forecasts against actual results, measuring both human and AI prediction quality.

**Depends on:** Phase 5

### Task 7.1: Accuracy Computation

**What:** After a forecast period closes, compute the accuracy of both human submissions and AI predictions against actual closed-won revenue.

**Design:**

```python
# backend/app/services/accuracy_service.py

class AccuracyService:
    async def compute_period_accuracy(self, tenant_id: str, period_id: str):
        """Compute forecast accuracy for a closed period."""
        period = await self.db.get_period(period_id)
        if period.end_date >= date.today():
            return  # Period not yet closed

        # Actual closed-won revenue in this period
        actual = await self.db.sum_closed_won_in_period(tenant_id, period.start_date, period.end_date)

        # Get last submission per rep before period end
        submissions = await self.db.get_final_submissions(tenant_id, period_id)
        # Get AI prediction snapshot closest to period end
        ai_snapshot = await self.db.get_snapshot_near_date(tenant_id, period_id, period.end_date)

        for sub in submissions:
            submitted = float(sub.data.get("submitted_amount", 0))
            ai_predicted = float(sub.data.get("ai_adjusted_amount", 0))
            actual_for_rep = await self.db.sum_closed_won_by_owner(
                tenant_id, sub.submitted_by, period.start_date, period.end_date
            )

            accuracy = ForecastAccuracy(
                tenant_id=tenant_id,
                period_id=period_id,
                user_id=sub.submitted_by,
                submitted_amount=submitted,
                ai_predicted_amount=ai_predicted,
                actual_amount=actual_for_rep,
                submitted_error_pct=(submitted - actual_for_rep) / actual_for_rep if actual_for_rep else None,
                ai_error_pct=(ai_predicted - actual_for_rep) / actual_for_rep if actual_for_rep else None,
            )
            self.db.add(accuracy)
```

**Testing:**
- Accuracy computation runs only for closed periods (end_date < today)
- Error percentage is correctly computed: (forecast - actual) / actual
- Positive error = over-forecast, negative error = under-forecast
- AI prediction sourced from the last AI-adjusted amount before period close
- Rep with zero actual revenue: error_pct is NULL (avoid division by zero)
- Duplicate computation for same period + user is prevented (unique constraint)

### Task 7.2: Accuracy Dashboard Frontend

**What:** Build the accuracy dashboard showing human vs. AI forecast accuracy over time, per-rep accuracy leaderboard, and accuracy trend charts.

**Testing:**
- Bar chart comparing human error % vs. AI error % per quarter
- "AI beat human" percentage displayed prominently (e.g., "AI was more accurate in 73% of forecasts")
- Per-rep accuracy table sortable by error percentage
- Accuracy trend line chart over last 4 quarters
- Empty state: "Complete at least one forecast period to see accuracy data"

---

## Phase 8: CRM Data Quality Scoring

**Goal:** Implement automated CRM data quality assessment that flags stale close dates, missing activities, inflated amounts, and other hygiene issues before they pollute forecasts.

**Depends on:** Phase 4

### Task 8.1: Data Quality Rule Engine

**What:** Build a rule-based quality scoring engine that evaluates each open deal against a set of quality checks and produces a composite DQ score (0-100).

**Design:**

```python
# backend/app/services/quality_service.py

QUALITY_CHECKS = [
    {
        "name": "stale_close_date",
        "description": "Close date is in the past but deal is still open",
        "check": lambda d: d.close_date and d.close_date < date.today() and not d.is_closed,
        "penalty": 25,
        "severity": "critical",
    },
    {
        "name": "no_activity_14_days",
        "description": "No activity logged in 14+ days",
        "check": lambda d: (d.lifecycle or {}).get("days_since_last_activity", 999) >= 14,
        "penalty": 15,
        "severity": "warning",
    },
    {
        "name": "no_activity_30_days",
        "description": "No activity logged in 30+ days",
        "check": lambda d: (d.lifecycle or {}).get("days_since_last_activity", 999) >= 30,
        "penalty": 25,
        "severity": "critical",
    },
    {
        "name": "missing_close_date",
        "description": "No close date set",
        "check": lambda d: d.close_date is None,
        "penalty": 20,
        "severity": "warning",
    },
    {
        "name": "missing_amount",
        "description": "No deal amount specified",
        "check": lambda d: not d.amount or d.amount <= 0,
        "penalty": 20,
        "severity": "warning",
    },
    {
        "name": "missing_champion",
        "description": "No champion identified (MEDDIC)",
        "check": lambda d: not (d.qualification or {}).get("champion"),
        "penalty": 10,
        "severity": "info",
    },
    {
        "name": "single_threaded",
        "description": "Only one contact associated with deal",
        "check": lambda d: (d.lifecycle or {}).get("contact_count", 0) <= 1,
        "penalty": 10,
        "severity": "info",
    },
    {
        "name": "close_date_slipped_3x",
        "description": "Close date has slipped 3 or more times",
        "check": lambda d: (d.lifecycle or {}).get("close_date_slip_count", 0) >= 3,
        "penalty": 20,
        "severity": "warning",
    },
]

class QualityScorer:
    def score_deal(self, deal: Deal) -> dict:
        flags = []
        total_penalty = 0
        details = {}
        for check in QUALITY_CHECKS:
            if check["check"](deal):
                flags.append(check["name"])
                total_penalty += check["penalty"]
                details[check["name"]] = {"severity": check["severity"], "description": check["description"]}
        score = max(0, 100 - total_penalty)
        return {"score": score, "flags": flags, "details": details, "last_assessed": utcnow().isoformat()}
```

**Testing:**
- Deal with no issues scores 100
- Deal with stale close date + no activity 30 days scores 50 (100 - 25 - 25)
- Score never goes below 0 (clamped)
- Flags array contains correct check names for each failing check
- Quality scoring runs as part of the daily lifecycle refresh (Phase 4.4)
- Updated quality data appears in `deals.quality` JSONB and on the deal detail page

### Task 8.2: Data Quality Dashboard

**What:** Build a dashboard showing aggregate CRM data quality metrics: average DQ score, most common issues, deals needing attention.

**Testing:**
- Average DQ score displayed per team and per rep
- Bar chart of most common quality issues (e.g., "stale_close_date: 12 deals, no_activity_14_days: 23 deals")
- Table of deals with DQ score < 50, sorted by amount (highest value dirty deals first)
- Clicking a deal navigates to deal detail with quality issues highlighted
- DQ score trend over last 30 days (has data quality improved?)

---

## Phase 9: What-If Scenario Modelling

**Goal:** Enable managers to create hypothetical scenarios by adjusting deal amounts, probabilities, and close dates, then see the impact on the forecast.

**Depends on:** Phase 5

### Task 9.1: Scenario Engine

**What:** Build the backend for creating scenarios, applying adjustments (amount override, probability override, close date override, exclude), and computing the adjusted forecast.

**Design:**

```python
# backend/app/services/scenario_service.py

class ScenarioService:
    async def create_scenario(self, tenant_id, period_id, name, adjustments):
        """Create a scenario with deal-level adjustments."""
        scenario = Scenario(tenant_id=tenant_id, period_id=period_id, name=name, created_by=current_user.id)
        self.db.add(scenario)

        for adj in adjustments:
            self.db.add(ScenarioAdjustment(
                scenario_id=scenario.id,
                deal_id=adj["deal_id"],
                adjustment_type=adj["type"],  # amount_override, probability_override, exclude
                original_value=str(adj["original"]),
                adjusted_value=str(adj["adjusted"]),
            ))

        scenario.total_forecast = await self._compute_adjusted_forecast(scenario)
        return scenario

    async def _compute_adjusted_forecast(self, scenario):
        """Compute forecast with scenario adjustments applied."""
        deals = await self.get_period_deals(scenario.tenant_id, scenario.period_id)
        total = 0
        for deal in deals:
            adj = self._get_adjustment(scenario, deal.id)
            if adj and adj.adjustment_type == "exclude":
                continue
            amount = float(adj.adjusted_value) if adj and adj.adjustment_type == "amount_override" else float(deal.amount or 0)
            prob = float(adj.adjusted_value) if adj and adj.adjustment_type == "probability_override" else float((deal.scoring or {}).get("win_probability", 0))
            total += amount * prob
        return total
```

**Testing:**
- Creating a scenario with 3 deal adjustments produces correct adjusted forecast total
- Excluding a $100K deal reduces the forecast by its weighted amount
- Overriding a deal's probability from 0.8 to 0.3 reduces its contribution
- Baseline scenario (no adjustments) matches the current AI-predicted forecast
- Comparing two scenarios shows the delta clearly
- Scenario is immutable after creation (adjustments are append-only)

### Task 9.2: Scenario Comparison UI

**What:** Build the frontend for creating scenarios, adjusting deals, and comparing scenario outcomes side by side.

**Testing:**
- User can select deals and adjust amount or probability via inline editing
- "Exclude deal" toggle removes it from the scenario
- Side-by-side comparison shows baseline vs. scenario with delta highlighted
- Multiple scenarios can be compared simultaneously (up to 3)
- Scenario saved and retrievable from scenario list

---

## Phase 10: Pluggable ML Backend

**Goal:** Abstract the ML model interface to support swapping between XGBoost, LightGBM, Prophet (time-series), and future LLM-hybrid models without reconfiguring the product.

**Depends on:** Phase 4

### Task 10.1: Model Backend Abstraction

**What:** Refactor the ML pipeline to use a pluggable model interface. Each model backend implements training, prediction, and explanation methods.

**Design:**

```python
# backend/app/ml/backends/base.py
from abc import ABC, abstractmethod

class ModelBackend(ABC):
    @abstractmethod
    def train(self, X, y, params: dict) -> Any:
        """Train and return a model artifact."""

    @abstractmethod
    def predict(self, model, X) -> np.ndarray:
        """Return win probabilities for each row."""

    @abstractmethod
    def explain(self, model, X) -> list[dict]:
        """Return per-row feature importance explanations."""

    @abstractmethod
    def get_default_params(self) -> dict:
        """Return default hyperparameters."""
```

```python
# backend/app/ml/backends/xgboost_backend.py
class XGBoostBackend(ModelBackend):
    def train(self, X, y, params):
        model = xgb.XGBClassifier(**params)
        model.fit(X, y)
        return model

    def predict(self, model, X):
        return model.predict_proba(X)[:, 1]

    def explain(self, model, X):
        explainer = shap.TreeExplainer(model)
        return explainer.shap_values(X)

# backend/app/ml/backends/lightgbm_backend.py
class LightGBMBackend(ModelBackend):
    # Similar implementation using lightgbm.LGBMClassifier

# backend/app/ml/backends/prophet_backend.py
class ProphetBackend(ModelBackend):
    """Time-series forecasting at the pipeline level (not deal-level)."""
    # Uses Prophet for aggregate pipeline trend forecasting
```

**Testing:**
- XGBoost and LightGBM backends produce comparable AUC-ROC on the same training data
- Switching backend via tenant settings (`settings.ml_backend = "lightgbm"`) uses the correct implementation
- SHAP explanations work for both tree-based backends
- Prophet backend produces pipeline-level time-series forecast (different output shape)
- Model artifact serialization/deserialization works for each backend
- Feature importance ranking is consistent with SHAP values for tree backends

### Task 10.2: Probabilistic Confidence Intervals

**What:** Replace placeholder confidence intervals with calibrated prediction intervals using conformal prediction or quantile regression.

**Testing:**
- Confidence intervals are calibrated: approximately 80% of actual outcomes fall within the 80% interval
- Interval width varies by deal (uncertain deals have wider intervals)
- Interval endpoints stored in `deals.scoring.confidence_lower` and `confidence_upper`
- Forecast dashboard shows uncertainty bands on the forecast-over-time chart

---

## Phase 11: Conversation Intelligence

**Goal:** Implement call recording ingestion, Whisper-based transcription, LLM-based insight extraction (objections, next steps, competitor mentions), and integration of call signals into deal scoring.

**Depends on:** Phase 2, Phase 4

### Task 11.1: Call Recording Ingestion

**What:** Build the recording upload and video conferencing integration pipeline. Support manual file upload and Zoom/Google Meet webhook ingestion.

**Design:**

```python
# backend/app/api/conversations/routes.py
@router.post("/recordings")
async def upload_recording(
    file: UploadFile,
    deal_id: UUID | None = None,
    source: str = "manual_upload",
    user: AuthUser = Depends(get_current_user),
):
    """Upload a call recording for transcription and insight extraction."""
    # Store file in object storage (S3/GCS)
    recording_url = await storage.upload(file, prefix=f"recordings/{user.tenant_id}")

    recording = CallRecording(
        tenant_id=user.tenant_id,
        deal_id=deal_id,
        recording_url=recording_url,
        source=source,
        processing={"transcription_status": "pending"},
    )
    db.add(recording)
    db.commit()

    # Queue transcription
    transcribe_recording.delay(str(recording.id))
    return recording
```

**Testing:**
- Audio file upload (MP3, WAV, M4A) stores file and creates database record
- Recording linked to deal via `deal_id` parameter
- Transcription task queued asynchronously after upload
- File size limit enforced (500MB max)
- GDPR: recording consent metadata captured in `processing` JSONB

### Task 11.2: Whisper Transcription

**What:** Transcribe recordings using OpenAI Whisper (open-source model). Extract speaker segments and full transcript text.

**Design:**

```python
# backend/app/tasks/transcription.py
import whisper

@shared_task(bind=True, max_retries=2)
def transcribe_recording(self, recording_id: str):
    recording = get_recording(recording_id)
    recording.processing["transcription_status"] = "processing"
    db.commit()

    try:
        model = whisper.load_model("large-v3")
        audio_path = storage.download_temp(recording.recording_url)
        result = model.transcribe(audio_path, word_timestamps=True)

        recording.transcript_text = result["text"]
        recording.duration_seconds = int(result.get("duration", 0))
        recording.processing.update({
            "transcription_status": "completed",
            "whisper_model": "large-v3",
            "language": result.get("language", "en"),
            "word_count": len(result["text"].split()),
        })
        db.commit()

        # Queue insight extraction
        extract_insights.delay(str(recording.id))

    except Exception as e:
        recording.processing["transcription_status"] = "failed"
        recording.processing["error"] = str(e)
        db.commit()
        raise self.retry(exc=e)
```

**Testing:**
- 30-minute audio file transcribed in under 10 minutes on GPU (or 30 minutes on CPU)
- Transcript text stored in `call_recordings.transcript_text`
- Full-text search index enables searching transcripts for keywords
- Failed transcription retries twice, then marks status as "failed" with error message
- Language detection correctly identifies English transcripts

### Task 11.3: LLM Insight Extraction

**What:** Use an LLM to extract structured deal insights from call transcripts: objections, next steps, competitor mentions, champion signals, and risk signals.

**Design:**

```python
# backend/app/tasks/insight_extraction.py
from anthropic import Anthropic

EXTRACTION_PROMPT = """Analyze this sales call transcript and extract the following insights.
For each insight, provide:
- type: one of [objection, next_step, competitor_mention, champion_signal, risk_signal, pricing_discussion, positive_signal]
- summary: one-sentence summary of the insight
- confidence: 0.0 to 1.0 confidence score

Transcript:
{transcript}

Return JSON array of insights."""

@shared_task
def extract_insights(recording_id: str):
    recording = get_recording(recording_id)
    client = Anthropic()

    response = client.messages.create(
        model="claude-sonnet-4-20250514",
        max_tokens=2000,
        messages=[{"role": "user", "content": EXTRACTION_PROMPT.format(transcript=recording.transcript_text[:8000])}],
    )

    insights = parse_json(response.content[0].text)
    recording.processing["insights"] = insights
    recording.processing["llm_model"] = "claude-sonnet-4-20250514"
    recording.processing["processed_at"] = utcnow().isoformat()
    db.commit()
```

**Testing:**
- Extraction from a sample sales call transcript produces 3-8 insights
- Each insight has `type`, `summary`, and `confidence` fields
- Competitor mention extraction correctly identifies named competitors
- Objection extraction captures budget concerns, timeline pushback, etc.
- Next step extraction identifies specific action items
- Insights displayed on deal detail page under "Call Insights" section
- Insights feed into ML scoring as additional features (Phase 11.4)

### Task 11.4: Call Signal Integration with ML Scoring

**What:** Incorporate conversation intelligence signals as features in the ML scoring model.

**Testing:**
- New features added: `call_count_total`, `has_objection`, `has_champion_signal`, `competitor_mention_count`, `last_call_days_ago`
- Model retrained with call features achieves higher AUC-ROC than without (A/B comparison)
- Deals with call recordings score differently than deals without (call signals have non-zero SHAP values)
- Feature importance shows call-derived features contribute meaningfully to predictions

---

## Phase 12: Enterprise & Scale

**Goal:** Harden the platform for enterprise deployment: multi-CRM consolidation, self-hosted deployment, RBAC refinement, and performance optimization.

**Depends on:** All previous phases

### Task 12.1: Multi-CRM Consolidation

**What:** Enable a single tenant to connect both Salesforce and HubSpot simultaneously, with a unified pipeline view and consolidated forecast.

**Testing:**
- Tenant connects Salesforce (50 deals) and HubSpot (30 deals): pipeline shows all 80 deals
- CRM source column on deal table identifies which CRM each deal came from
- Forecast roll-up aggregates across both CRMs
- Duplicate detection: same real-world deal in both CRMs can be manually linked
- Disconnecting one CRM does not delete the other CRM's data

### Task 12.2: Self-Hosted Deployment (Docker/Helm)

**What:** Package the entire application as Docker images with Helm charts for Kubernetes deployment.

**Testing:**
- `docker compose up` starts the complete stack (Postgres, Redis, API, Worker, Frontend) on a single machine
- Helm chart deploys to a Kubernetes cluster with configurable replicas
- All data stays within the customer's infrastructure (no external API calls except CRM and optional LLM)
- Environment variables configure database, Redis, and auth provider
- Health checks and readiness probes configured for all services

### Task 12.3: Performance Optimization

**What:** Add database connection pooling, query optimization, Redis caching for hot data, and CDN for frontend assets.

**Testing:**
- Deal list API responds in < 200ms for 1000 deals with scoring and quality data
- Forecast dashboard loads in < 500ms
- Concurrent CRM syncs for 10 tenants do not degrade API response times
- Redis cache hit rate > 80% for deal list and pipeline queries
- Database query explain plans show index usage for all critical queries

### Task 12.4: Pipedrive and Dynamics 365 Connectors

**What:** Add CRM connectors for Pipedrive (REST API v1) and Microsoft Dynamics 365 (OData v4).

**Testing:**
- Pipedrive connector syncs Deals, Persons, Organizations, and Activities
- Dynamics 365 connector uses OData v4 with `$select`, `$filter`, `$expand` for efficient querying
- Both connectors implement the same `BaseCRMConnector` interface
- Field mapping tested with real CRM data exports

---

## Definition of Done (per phase)

A phase is considered **done** when:

1. **All tasks implemented** — every task in the phase has been coded, reviewed, and merged to the main branch.
2. **Tests passing** — all unit tests, integration tests, and the specific test cases listed per task pass in CI.
3. **No critical bugs** — no P0/P1 bugs open against the phase's functionality.
4. **API documentation** — all new API endpoints have OpenAPI specs auto-generated by FastAPI.
5. **Database migrations** — all schema changes have Alembic migration files that apply cleanly to a fresh database.
6. **Environment documentation** — any new environment variables or infrastructure requirements are documented.
7. **Performance baseline** — key queries and API endpoints have baseline response time measurements.
8. **Security review** — new endpoints enforce authentication and tenant isolation; no new SQL injection or XSS vectors introduced.

## Definition of Done (project-level MVP)

The MVP (Phases 1-7) is considered **done** when:

- A user can connect Salesforce or HubSpot via OAuth
- CRM deals sync automatically every 15 minutes
- ML model trains on historical closed deals and scores open deals daily
- Each deal has a win probability, risk level, and natural-language explanation
- Reps can submit forecasts; managers see team roll-ups with AI adjustments
- At-risk deals trigger Slack alerts
- Forecast accuracy is tracked and displayed after each quarter closes
- The application runs via `docker compose up` with seed data for demonstration
