# Data Model Suggestion 2: Event-Sourced / Audit-First (CQRS)

> Project: Partner Relationship Management · Created: 2026-05-12

## Philosophy

This model treats every state change as an immutable event appended to a central event store. The event store is the single source of truth; all read-optimised views (partner profiles, deal pipelines, MDF balances, commission ledgers) are materialised projections rebuilt from events. This is the Command Query Responsibility Segregation (CQRS) pattern applied to PRM.

The approach is inspired by financial ledger systems, where every transaction is recorded and balances are derived, never stored directly. In the PRM context, this means a deal registration is not a row that gets updated through stages — it is a sequence of events: `DealRegistered`, `DealApproved`, `DealStageChanged`, `DealWon`, `CommissionCalculated`. The current state of any deal is computed by replaying its events.

This design is particularly powerful for PRM because channel programs have heavy audit and compliance requirements. Partners, vendors, and regulators all need to answer questions like "what was the deal status on March 15th?" or "who approved this MDF claim and when?" Event sourcing answers these questions natively, without retrofitting audit columns onto relational tables.

**Best for:** Teams prioritising complete audit trails, temporal queries, AI-powered analytics on partner behaviour patterns, and regulatory compliance.

**Trade-offs:**
- (+) 100% reliable audit trail — every change is permanently recorded
- (+) Temporal queries are trivial — reconstruct any entity state at any point in time
- (+) Enables AI/ML analysis of partner behaviour sequences (event patterns predict churn)
- (+) Schema evolution is easier — new event types can be added without migrations
- (+) Natural fit for webhook/notification systems — events are webhooks
- (-) Higher write amplification — every change creates a new event row
- (-) Read model complexity — projections must be maintained and kept in sync
- (-) Eventual consistency between event store and read models
- (-) More complex to query ad hoc — cannot simply SELECT * FROM deals
- (-) Team must learn event sourcing patterns; less familiar than CRUD

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| ISO 3166-1/2 | Event payloads include ISO country codes for partner and customer geography |
| ISO 4217 | Currency codes in all monetary event payloads |
| OAuth 2.0 (RFC 6749) | Integration events reference OAuth token lifecycle |
| SAML 2.0 | SSO configuration events carry IdP metadata per SAML spec |
| SCORM 1.2/2004 | Training events include SCORM cmi data model elements |
| Standard Webhooks | Event delivery to external systems follows Standard Webhooks payload and signature format |
| RFC 7807 | Command rejection responses use Problem Details format |
| GDPR / CCPA | Right-to-erasure implemented via crypto-shredding — encryption keys deleted, events become unreadable |

---

## Event Store (Core)

```sql
-- The central event store. All state changes across the entire system are appended here.
-- This table is APPEND-ONLY. No UPDATE or DELETE operations are permitted.
CREATE TABLE events (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL,
    aggregate_type  VARCHAR(50) NOT NULL,
    -- partner, deal, mdf_request, mdf_claim, commission, training, certification, content
    aggregate_id    UUID NOT NULL,                        -- the entity this event belongs to
    event_type      VARCHAR(100) NOT NULL,                -- e.g., DealRegistered, DealApproved, PartnerTierChanged
    event_version   INT NOT NULL DEFAULT 1,               -- schema version for this event type
    sequence_number BIGINT NOT NULL,                      -- per-aggregate ordering
    payload         JSONB NOT NULL,                       -- event-specific data
    metadata        JSONB NOT NULL DEFAULT '{}',          -- actor_id, ip_address, user_agent, correlation_id
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(aggregate_id, sequence_number)
);

-- Primary query pattern: load all events for an aggregate to rebuild state
CREATE INDEX idx_events_aggregate ON events(aggregate_id, sequence_number);

-- Secondary: query all events of a type within an org (for projections)
CREATE INDEX idx_events_org_type ON events(organisation_id, event_type, created_at);

-- Tertiary: global ordering for catch-up subscriptions
CREATE INDEX idx_events_global_order ON events(created_at, id);

-- Partition by month for manageability at scale
-- CREATE TABLE events PARTITION BY RANGE (created_at);

-- Aggregate version tracking for optimistic concurrency control
CREATE TABLE aggregates (
    id              UUID PRIMARY KEY,
    organisation_id UUID NOT NULL,
    aggregate_type  VARCHAR(50) NOT NULL,
    current_version BIGINT NOT NULL DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_aggregates_org_type ON aggregates(organisation_id, aggregate_type);
```

### Example Event Payloads

```sql
-- DealRegistered event payload:
-- {
--   "partner_id": "uuid",
--   "deal_number": "DR-2026-00042",
--   "customer_name": "Acme Corp",
--   "customer_email": "buyer@acme.com",
--   "customer_company": "Acme Corp",
--   "customer_country": "US",
--   "product_interest": "Enterprise Plan",
--   "estimated_value": 85000.00,
--   "currency": "USD",
--   "expected_close_date": "2026-09-15",
--   "co_sell_type": "aws_ace",
--   "registered_by": "uuid"
-- }

-- DealApproved event payload:
-- {
--   "approved_by": "uuid",
--   "expiry_date": "2026-12-15",
--   "notes": "Strong pipeline fit. Assigned to APAC team."
-- }

-- PartnerTierChanged event payload:
-- {
--   "previous_tier_id": "uuid",
--   "new_tier_id": "uuid",
--   "reason": "auto_qualification",
--   "qualifying_revenue": 250000.00,
--   "qualifying_deals": 12
-- }

-- MdfClaimApproved event payload:
-- {
--   "claim_id": "uuid",
--   "approved_amount": 5000.00,
--   "reviewed_by": "uuid",
--   "proof_documents": ["uuid1", "uuid2"]
-- }

-- CommissionCalculated event payload:
-- {
--   "deal_id": "uuid",
--   "plan_id": "uuid",
--   "deal_value": 85000.00,
--   "rate": 0.10,
--   "commission_amount": 8500.00,
--   "currency": "USD"
-- }
```

---

## Read Models (Materialised Projections)

These tables are derived from events and can be rebuilt at any time by replaying the event store. They exist purely for query performance.

### Partner Read Model

```sql
CREATE TABLE rm_partners (
    id              UUID PRIMARY KEY,
    organisation_id UUID NOT NULL,
    company_name    VARCHAR(255) NOT NULL,
    legal_name      VARCHAR(255),
    website         TEXT,
    logo_url        TEXT,
    primary_contact_name  VARCHAR(255),
    primary_contact_email VARCHAR(255),
    country_code    CHAR(2),
    program_id      UUID,
    program_name    VARCHAR(255),
    tier_id         UUID,
    tier_name       VARCHAR(100),
    tier_rank       INT,
    status          VARCHAR(30) NOT NULL,
    partner_since   DATE,
    total_deals     INT NOT NULL DEFAULT 0,
    total_revenue   NUMERIC(15,2) NOT NULL DEFAULT 0,
    open_deals      INT NOT NULL DEFAULT 0,
    won_deals       INT NOT NULL DEFAULT 0,
    certifications_count INT NOT NULL DEFAULT 0,
    last_deal_at    TIMESTAMPTZ,
    last_activity_at TIMESTAMPTZ,
    health_score    NUMERIC(5,2),                         -- AI-computed from event patterns
    churn_risk      VARCHAR(20),                          -- low, medium, high, critical
    projected_from  TIMESTAMPTZ NOT NULL DEFAULT now(),   -- when this projection was last rebuilt
    created_at      TIMESTAMPTZ NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_rm_partners_org ON rm_partners(organisation_id);
CREATE INDEX idx_rm_partners_status ON rm_partners(organisation_id, status);
CREATE INDEX idx_rm_partners_tier ON rm_partners(organisation_id, tier_rank);
CREATE INDEX idx_rm_partners_health ON rm_partners(organisation_id, health_score);
```

### Deal Pipeline Read Model

```sql
CREATE TABLE rm_deals (
    id              UUID PRIMARY KEY,
    organisation_id UUID NOT NULL,
    partner_id      UUID NOT NULL,
    partner_name    VARCHAR(255),
    deal_number     VARCHAR(50) NOT NULL,
    customer_name   VARCHAR(255) NOT NULL,
    customer_company VARCHAR(255),
    customer_country CHAR(2),
    estimated_value NUMERIC(15,2),
    currency        CHAR(3),
    stage           VARCHAR(50) NOT NULL,
    approval_status VARCHAR(30) NOT NULL,
    approved_by_name VARCHAR(255),
    co_sell_type    VARCHAR(50),
    co_sell_ref     VARCHAR(255),
    expected_close_date DATE,
    actual_close_date DATE,
    expiry_date     DATE,
    days_in_stage   INT,
    registered_by_name VARCHAR(255),
    crm_opportunity_id VARCHAR(255),
    projected_from  TIMESTAMPTZ NOT NULL DEFAULT now(),
    created_at      TIMESTAMPTZ NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_rm_deals_org ON rm_deals(organisation_id);
CREATE INDEX idx_rm_deals_partner ON rm_deals(partner_id);
CREATE INDEX idx_rm_deals_stage ON rm_deals(organisation_id, stage);
CREATE INDEX idx_rm_deals_dedup ON rm_deals(organisation_id, customer_company, stage)
    WHERE approval_status != 'rejected';
```

### MDF Balance Read Model

```sql
CREATE TABLE rm_mdf_balances (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL,
    partner_id      UUID,
    fiscal_year     INT NOT NULL,
    fiscal_quarter  INT,
    total_budget    NUMERIC(15,2) NOT NULL DEFAULT 0,
    allocated       NUMERIC(15,2) NOT NULL DEFAULT 0,
    pending_claims  NUMERIC(15,2) NOT NULL DEFAULT 0,
    approved_claims NUMERIC(15,2) NOT NULL DEFAULT 0,
    paid_claims     NUMERIC(15,2) NOT NULL DEFAULT 0,
    remaining       NUMERIC(15,2) NOT NULL DEFAULT 0,
    currency        CHAR(3) NOT NULL DEFAULT 'USD',
    projected_from  TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_rm_mdf_org ON rm_mdf_balances(organisation_id);
CREATE INDEX idx_rm_mdf_partner ON rm_mdf_balances(partner_id);

CREATE TABLE rm_mdf_requests (
    id              UUID PRIMARY KEY,
    organisation_id UUID NOT NULL,
    partner_id      UUID NOT NULL,
    partner_name    VARCHAR(255),
    request_number  VARCHAR(50) NOT NULL,
    title           VARCHAR(255) NOT NULL,
    activity_type   VARCHAR(100),
    requested_amount NUMERIC(15,2),
    approved_amount NUMERIC(15,2),
    currency        CHAR(3),
    status          VARCHAR(30) NOT NULL,
    planned_start   DATE,
    planned_end     DATE,
    claims_count    INT NOT NULL DEFAULT 0,
    total_claimed   NUMERIC(15,2) NOT NULL DEFAULT 0,
    projected_from  TIMESTAMPTZ NOT NULL DEFAULT now(),
    created_at      TIMESTAMPTZ NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_rm_mdf_requests_org ON rm_mdf_requests(organisation_id);
CREATE INDEX idx_rm_mdf_requests_partner ON rm_mdf_requests(partner_id);
```

### Commission Ledger Read Model

```sql
CREATE TABLE rm_commission_ledger (
    id              UUID PRIMARY KEY,
    organisation_id UUID NOT NULL,
    partner_id      UUID NOT NULL,
    partner_name    VARCHAR(255),
    deal_id         UUID,
    deal_number     VARCHAR(50),
    plan_name       VARCHAR(255),
    entry_type      VARCHAR(30) NOT NULL,                 -- earned, clawback, adjustment, payout
    amount          NUMERIC(15,2) NOT NULL,
    currency        CHAR(3) NOT NULL DEFAULT 'USD',
    status          VARCHAR(30) NOT NULL,
    period_start    DATE,
    period_end      DATE,
    paid_at         TIMESTAMPTZ,
    projected_from  TIMESTAMPTZ NOT NULL DEFAULT now(),
    created_at      TIMESTAMPTZ NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_rm_commissions_org ON rm_commission_ledger(organisation_id);
CREATE INDEX idx_rm_commissions_partner ON rm_commission_ledger(partner_id);
CREATE INDEX idx_rm_commissions_status ON rm_commission_ledger(organisation_id, status);

CREATE TABLE rm_commission_balances (
    partner_id      UUID PRIMARY KEY,
    organisation_id UUID NOT NULL,
    total_earned    NUMERIC(15,2) NOT NULL DEFAULT 0,
    total_paid      NUMERIC(15,2) NOT NULL DEFAULT 0,
    total_clawbacks NUMERIC(15,2) NOT NULL DEFAULT 0,
    balance_due     NUMERIC(15,2) NOT NULL DEFAULT 0,
    currency        CHAR(3) NOT NULL DEFAULT 'USD',
    projected_from  TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_rm_comm_bal_org ON rm_commission_balances(organisation_id);
```

### Training & Certification Read Model

```sql
CREATE TABLE rm_partner_certifications (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL,
    partner_id      UUID NOT NULL,
    partner_name    VARCHAR(255),
    contact_id      UUID NOT NULL,
    contact_name    VARCHAR(255),
    certification_name VARCHAR(255),
    certification_id UUID,
    status          VARCHAR(20) NOT NULL,
    modules_total   INT NOT NULL DEFAULT 0,
    modules_completed INT NOT NULL DEFAULT 0,
    awarded_at      TIMESTAMPTZ,
    expires_at      TIMESTAMPTZ,
    projected_from  TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_rm_certs_org ON rm_partner_certifications(organisation_id);
CREATE INDEX idx_rm_certs_partner ON rm_partner_certifications(partner_id);
```

---

## Projection Management

```sql
-- Tracks the position of each projector (read model builder) in the event stream
CREATE TABLE projection_checkpoints (
    projector_name  VARCHAR(100) PRIMARY KEY,
    last_event_id   UUID,
    last_event_at   TIMESTAMPTZ,
    events_processed BIGINT NOT NULL DEFAULT 0,
    status          VARCHAR(20) NOT NULL DEFAULT 'running', -- running, paused, rebuilding, error
    error_message   TEXT,
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Dead letter queue for events that failed projection
CREATE TABLE projection_failures (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    projector_name  VARCHAR(100) NOT NULL,
    event_id        UUID NOT NULL,
    error_message   TEXT NOT NULL,
    retry_count     INT NOT NULL DEFAULT 0,
    max_retries     INT NOT NULL DEFAULT 5,
    next_retry_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_proj_failures_retry ON projection_failures(projector_name, next_retry_at)
    WHERE retry_count < max_retries;
```

---

## Subscriptions & Webhooks

```sql
-- External webhook subscriptions for event delivery
CREATE TABLE webhook_subscriptions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL,
    url             TEXT NOT NULL,
    secret          TEXT NOT NULL,                        -- HMAC signing key (Standard Webhooks spec)
    event_types     TEXT[] NOT NULL,                      -- filter: ['deal.*', 'partner.tier_changed']
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_webhooks_org ON webhook_subscriptions(organisation_id);

-- Webhook delivery log
CREATE TABLE webhook_deliveries (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    subscription_id UUID NOT NULL REFERENCES webhook_subscriptions(id),
    event_id        UUID NOT NULL,
    payload         JSONB NOT NULL,
    response_status INT,
    response_body   TEXT,
    attempt         INT NOT NULL DEFAULT 1,
    delivered_at    TIMESTAMPTZ,
    next_retry_at   TIMESTAMPTZ,
    status          VARCHAR(20) NOT NULL DEFAULT 'pending', -- pending, delivered, failed, exhausted
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_webhook_del_sub ON webhook_deliveries(subscription_id);
CREATE INDEX idx_webhook_del_retry ON webhook_deliveries(next_retry_at)
    WHERE status = 'pending';
```

---

## Supporting Tables (Non-Event-Sourced)

Some reference data is not event-sourced because it is configuration, not business state:

```sql
CREATE TABLE organisations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    plan            VARCHAR(50) NOT NULL DEFAULT 'free',
    settings        JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email           VARCHAR(255) NOT NULL UNIQUE,
    full_name       VARCHAR(255) NOT NULL,
    password_hash   TEXT,
    last_login_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE memberships (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    organisation_id UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    role            VARCHAR(50) NOT NULL DEFAULT 'member',
    status          VARCHAR(20) NOT NULL DEFAULT 'active',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(user_id, organisation_id)
);

-- GDPR crypto-shredding: per-aggregate encryption keys
-- Deleting a key renders all events for that aggregate unreadable
CREATE TABLE encryption_keys (
    aggregate_id    UUID PRIMARY KEY,
    organisation_id UUID NOT NULL,
    encryption_key  BYTEA NOT NULL,                       -- AES-256 key, itself encrypted with master key
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    deleted_at      TIMESTAMPTZ                           -- set when right-to-erasure exercised
);

CREATE INDEX idx_encryption_keys_org ON encryption_keys(organisation_id);
```

---

## Example Temporal Query

Reconstruct a deal's state at a specific point in time:

```sql
-- What was the state of deal DR-2026-00042 on March 15, 2026?
SELECT
    e.event_type,
    e.payload,
    e.metadata,
    e.created_at
FROM events e
WHERE e.aggregate_type = 'deal'
  AND e.aggregate_id = '...'              -- deal's aggregate ID
  AND e.created_at <= '2026-03-15 23:59:59+00'
ORDER BY e.sequence_number ASC;

-- The application replays these events in order to reconstruct the deal
-- object as it existed on March 15.
```

## Example Partner Behaviour Pattern Query

Find partners whose event patterns suggest churn risk:

```sql
-- Partners who had active deal registrations but no new events in 60+ days
SELECT
    rm.id AS partner_id,
    rm.company_name,
    rm.total_deals,
    rm.last_activity_at,
    now() - rm.last_activity_at AS days_silent
FROM rm_partners rm
WHERE rm.organisation_id = '...'
  AND rm.status = 'active'
  AND rm.last_activity_at < now() - INTERVAL '60 days'
  AND rm.total_deals > 0
ORDER BY rm.last_activity_at ASC;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Event Store | 2 | events, aggregates |
| Read Models — Partners | 1 | rm_partners |
| Read Models — Deals | 1 | rm_deals |
| Read Models — MDF | 2 | rm_mdf_balances, rm_mdf_requests |
| Read Models — Commissions | 2 | rm_commission_ledger, rm_commission_balances |
| Read Models — Training | 1 | rm_partner_certifications |
| Projection Infrastructure | 2 | projection_checkpoints, projection_failures |
| Webhooks | 2 | webhook_subscriptions, webhook_deliveries |
| Supporting (Non-ES) | 4 | organisations, users, memberships, encryption_keys |
| **Total** | **17** | Plus read model tables grow as new projections are added |

---

## Key Design Decisions

1. **Single events table for all aggregate types** — rather than one event table per entity, a single `events` table simplifies infrastructure (one subscription, one partition strategy, one backup process). The `aggregate_type` column enables type-specific queries.

2. **JSONB payload per event** — each event type has a different payload shape stored as JSONB. This avoids wide nullable columns and allows new event types to be added without schema migrations. JSON Schema validation occurs at the application layer.

3. **Sequence number per aggregate** — `sequence_number` provides strict per-aggregate ordering with optimistic concurrency control. Before appending, the application checks `current_version` in the `aggregates` table and fails if another write has intervened.

4. **Crypto-shredding for GDPR** — instead of deleting events (which would break the immutable append-only guarantee), sensitive fields in event payloads are encrypted with a per-aggregate key. Right-to-erasure is implemented by deleting the key, rendering the data unrecoverable while preserving event structure.

5. **Read models are disposable** — every `rm_*` table can be dropped and rebuilt from the event store. The `projection_checkpoints` table tracks each projector's position so rebuilds can resume from a known point. This decouples read schema evolution from the write side.

6. **Denormalised read models** — read model tables include redundant data (e.g., `partner_name` on `rm_deals`) to avoid JOINs on read paths. This is intentional: read models exist to serve queries fast, not to enforce referential integrity.

7. **Webhook delivery as first-class citizen** — events naturally map to webhooks. The `webhook_subscriptions` table uses event type glob patterns (`deal.*`) for filtering, and deliveries follow the Standard Webhooks specification for signing and retry.

8. **Partner health score in read model** — `health_score` and `churn_risk` on `rm_partners` are computed by an AI projection that analyses event patterns (deal velocity, training completion, login frequency, MDF utilisation). This is a separate projector that runs asynchronously.

9. **Projection failure handling** — the `projection_failures` table implements a dead letter queue with exponential backoff retry, preventing a single bad event from stalling the entire projection pipeline.

10. **Time-partitioning readiness** — the events table comment notes partitioning by `created_at`. For production deployments with millions of events per month, PostgreSQL declarative partitioning by monthly ranges keeps individual partition sizes manageable and enables efficient time-range queries.
