# Data Model Suggestion 3: Hybrid Relational + JSONB

> Project: Partner Relationship Management · Created: 2026-05-12

## Philosophy

This model uses a relational core for the entities and relationships that are common across all PRM deployments (organisations, partners, deals, MDF) but delegates variable, jurisdiction-specific, and customer-customisable fields to JSONB columns. The result is a leaner table count than the fully normalised model, with the flexibility to support diverse partner program configurations without schema migrations.

The key insight driving this design is that PRM programs vary enormously by industry, geography, and company maturity. A hardware manufacturer's reseller program in Germany has different partner qualification fields than a SaaS company's referral program in the US. Rather than pre-defining every possible field or forcing customers into a rigid schema, the hybrid model defines the structural backbone relationally and makes the "flesh" — the program-specific, tier-specific, and jurisdiction-specific details — configurable via JSONB.

This pattern is used extensively by modern SaaS platforms like Notion (block content in JSONB), Linear (flexible issue metadata), and HubSpot (custom properties stored as JSON). It enables rapid MVP development because new field requirements can be shipped as configuration changes rather than database migrations.

**Best for:** Teams building an MVP-first PRM that must support diverse partner program types across multiple geographies without heavy migration overhead.

**Trade-offs:**
- (+) Lower table count — fewer tables to maintain, migrate, and back up
- (+) Highly flexible — new fields can be added via configuration, not migrations
- (+) Supports jurisdiction-specific fields (e.g., German tax ID, UK Companies House number) without separate tables
- (+) Rapid prototyping — JSONB columns can accommodate schema exploration
- (+) PostgreSQL JSONB has excellent query performance with GIN indexes
- (-) Weaker type safety for JSONB fields — validation must happen at application layer
- (-) JSONB fields are harder to query in ad hoc SQL than typed columns
- (-) Foreign key constraints cannot reference into JSONB — referential integrity for JSONB-stored references requires application enforcement
- (-) JSONB columns can become "junk drawers" without discipline
- (-) Reporting tools may struggle with JSONB field extraction

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| ISO 3166-1/2 | `country_code` is a typed column; jurisdiction-specific fields live in `custom_fields` JSONB |
| ISO 4217 | Currency codes are typed columns on all monetary tables |
| JSON Schema Draft 2020-12 | `custom_field_schemas` table stores JSON Schema definitions that validate `custom_fields` JSONB at the application layer |
| OAuth 2.0 (RFC 6749) | Integration credentials in `integrations` table with provider-specific config in JSONB |
| SAML 2.0 | SSO configuration stored as JSONB in `integrations` table (protocol=saml2) |
| SCORM 1.2/2004 | Training progress SCORM data model elements stored in JSONB `scorm_data` column |
| OpenAPI 3.1 | API schema auto-generated; JSONB fields documented via `x-json-schema` extension |
| GDPR / CCPA | `privacy` JSONB on partner_contacts stores consent records; `data_requests` table handles erasure |

---

## Organisation & Users

```sql
CREATE TABLE organisations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    domain          VARCHAR(255),
    plan            VARCHAR(50) NOT NULL DEFAULT 'free',
    settings        JSONB NOT NULL DEFAULT '{}',
    -- settings example:
    -- {
    --   "default_currency": "USD",
    --   "default_deal_expiry_days": 90,
    --   "mdf_requires_proof": true,
    --   "notifications": {"slack_webhook": "https://...", "teams_webhook": "https://..."},
    --   "branding": {"logo_url": "...", "primary_color": "#1a73e8"}
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email           VARCHAR(255) NOT NULL UNIQUE,
    full_name       VARCHAR(255) NOT NULL,
    avatar_url      TEXT,
    password_hash   TEXT,
    preferences     JSONB NOT NULL DEFAULT '{}',
    -- preferences example:
    -- {
    --   "timezone": "America/New_York",
    --   "notification_channels": ["email", "slack"],
    --   "language": "en"
    -- }
    last_login_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE memberships (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    organisation_id UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    role            VARCHAR(50) NOT NULL DEFAULT 'member',
    permissions     JSONB NOT NULL DEFAULT '[]',
    -- permissions example (fine-grained beyond role):
    -- ["deals.approve", "mdf.approve", "partners.manage", "content.publish"]
    status          VARCHAR(20) NOT NULL DEFAULT 'active',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(user_id, organisation_id)
);

CREATE INDEX idx_memberships_org ON memberships(organisation_id);
CREATE INDEX idx_memberships_user ON memberships(user_id);
```

## Custom Field Definitions

```sql
-- Defines which custom fields exist for each entity type per organisation.
-- This enables self-service field creation without migrations.
CREATE TABLE custom_field_schemas (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    entity_type     VARCHAR(50) NOT NULL,
    -- partner, deal, mdf_request, partner_contact
    field_key       VARCHAR(100) NOT NULL,                -- JSON path key, e.g., "tax_id", "vat_number"
    field_label     VARCHAR(255) NOT NULL,                -- display label
    field_type      VARCHAR(30) NOT NULL,                 -- text, number, date, boolean, select, multi_select, url
    is_required     BOOLEAN NOT NULL DEFAULT false,
    options         JSONB,                                -- for select/multi_select: ["Option A", "Option B"]
    validation      JSONB,                                -- JSON Schema fragment for validation
    -- validation example:
    -- {"type": "string", "pattern": "^DE\\d{9}$", "description": "German Tax ID"}
    sort_order      INT NOT NULL DEFAULT 0,
    visible_to      JSONB NOT NULL DEFAULT '["all"]',     -- ["all"] or ["tier:gold", "program:reseller"]
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(organisation_id, entity_type, field_key)
);

CREATE INDEX idx_custom_fields_org_entity ON custom_field_schemas(organisation_id, entity_type);
```

## Partner Management

```sql
CREATE TABLE partner_programs (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    program_type    VARCHAR(50) NOT NULL,                 -- reseller, referral, affiliate, technology, msp
    status          VARCHAR(20) NOT NULL DEFAULT 'active',
    config          JSONB NOT NULL DEFAULT '{}',
    -- config example:
    -- {
    --   "deal_registration_required": true,
    --   "auto_approve_deals_under": 10000,
    --   "default_deal_expiry_days": 90,
    --   "commission_model": "percentage",
    --   "default_commission_rate": 0.10,
    --   "tiers": [
    --     {"name": "Platinum", "rank": 1, "min_revenue": 500000, "min_deals": 20, "min_certs": 5},
    --     {"name": "Gold", "rank": 2, "min_revenue": 200000, "min_deals": 10, "min_certs": 3},
    --     {"name": "Silver", "rank": 3, "min_revenue": 50000, "min_deals": 5, "min_certs": 1},
    --     {"name": "Registered", "rank": 4, "min_revenue": 0, "min_deals": 0, "min_certs": 0}
    --   ]
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_programs_org ON partner_programs(organisation_id);

CREATE TABLE partners (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    program_id      UUID NOT NULL REFERENCES partner_programs(id),
    company_name    VARCHAR(255) NOT NULL,
    website         TEXT,
    country_code    CHAR(2),                              -- ISO 3166-1 alpha-2
    tier_name       VARCHAR(100),                         -- denormalised from program config
    tier_rank       INT,
    status          VARCHAR(30) NOT NULL DEFAULT 'prospect',
    partner_since   DATE,
    custom_fields   JSONB NOT NULL DEFAULT '{}',
    -- custom_fields example (varies by org and jurisdiction):
    -- {
    --   "legal_name": "Acme GmbH",
    --   "tax_id": "DE123456789",
    --   "companies_house_number": null,
    --   "vertical": "financial_services",
    --   "employee_count": 250,
    --   "salesforce_account_id": "001XXXXXXXXXXXX",
    --   "partner_manager": "Jane Smith",
    --   "annual_target": 500000
    -- }
    stats           JSONB NOT NULL DEFAULT '{}',
    -- stats example (computed, cached):
    -- {
    --   "total_deals": 15,
    --   "won_deals": 8,
    --   "total_revenue": 425000,
    --   "open_pipeline": 180000,
    --   "certifications": 4,
    --   "mdf_utilisation_pct": 72,
    --   "last_deal_at": "2026-04-20T14:30:00Z",
    --   "health_score": 82
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_partners_org ON partners(organisation_id);
CREATE INDEX idx_partners_program ON partners(program_id);
CREATE INDEX idx_partners_status ON partners(organisation_id, status);
CREATE INDEX idx_partners_country ON partners(country_code);
CREATE INDEX idx_partners_tier ON partners(organisation_id, tier_rank);
-- GIN index on custom_fields for flexible querying
CREATE INDEX idx_partners_custom ON partners USING GIN(custom_fields jsonb_path_ops);

CREATE TABLE partner_contacts (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    partner_id      UUID NOT NULL REFERENCES partners(id) ON DELETE CASCADE,
    full_name       VARCHAR(255) NOT NULL,
    email           VARCHAR(255) NOT NULL,
    phone           VARCHAR(50),
    job_title       VARCHAR(255),
    role            VARCHAR(50),                          -- primary, technical, billing, marketing
    user_id         UUID REFERENCES users(id),
    custom_fields   JSONB NOT NULL DEFAULT '{}',
    privacy         JSONB NOT NULL DEFAULT '{}',
    -- privacy example:
    -- {
    --   "marketing_consent": true,
    --   "consent_date": "2026-01-15",
    --   "consent_source": "portal_signup",
    --   "data_processing_basis": "legitimate_interest"
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_contacts_partner ON partner_contacts(partner_id);
CREATE INDEX idx_contacts_email ON partner_contacts(email);
```

## Deal Registration

```sql
CREATE TABLE deals (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    partner_id      UUID NOT NULL REFERENCES partners(id),
    registered_by   UUID NOT NULL REFERENCES users(id),
    deal_number     VARCHAR(50) NOT NULL,
    -- Customer info: core fields relational, extras in custom_fields
    customer_name   VARCHAR(255) NOT NULL,
    customer_email  VARCHAR(255),
    customer_company VARCHAR(255),
    customer_country CHAR(2),
    -- Deal details
    estimated_value NUMERIC(15,2),
    currency        CHAR(3) NOT NULL DEFAULT 'USD',
    expected_close_date DATE,
    actual_close_date DATE,
    stage           VARCHAR(50) NOT NULL DEFAULT 'submitted',
    approval_status VARCHAR(30) NOT NULL DEFAULT 'pending',
    approved_by     UUID REFERENCES users(id),
    approved_at     TIMESTAMPTZ,
    expiry_date     DATE,
    -- Co-sell info
    co_sell         JSONB NOT NULL DEFAULT '{}',
    -- co_sell example:
    -- {
    --   "type": "aws_ace",
    --   "opportunity_id": "OP-12345",
    --   "status": "submitted",
    --   "last_synced_at": "2026-04-20T10:00:00Z",
    --   "hyperscaler_stage": "prospect"
    -- }
    -- CRM sync
    crm_refs        JSONB NOT NULL DEFAULT '{}',
    -- crm_refs example:
    -- {
    --   "salesforce": {"opportunity_id": "006XXXX", "last_synced": "2026-04-20"},
    --   "hubspot": {"deal_id": "12345678", "last_synced": "2026-04-20"}
    -- }
    -- Custom fields
    custom_fields   JSONB NOT NULL DEFAULT '{}',
    -- custom_fields example:
    -- {
    --   "product_interest": "Enterprise Plan",
    --   "use_case": "Data analytics",
    --   "competition": "Competitor X",
    --   "decision_maker": "CTO",
    --   "budget_approved": true,
    --   "implementation_timeline": "Q3 2026"
    -- }
    notes           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE UNIQUE INDEX idx_deals_number ON deals(organisation_id, deal_number);
CREATE INDEX idx_deals_org ON deals(organisation_id);
CREATE INDEX idx_deals_partner ON deals(partner_id);
CREATE INDEX idx_deals_stage ON deals(organisation_id, stage);
CREATE INDEX idx_deals_custom ON deals USING GIN(custom_fields jsonb_path_ops);
-- Duplicate detection using JSONB containment
CREATE INDEX idx_deals_dedup ON deals(organisation_id, customer_email)
    WHERE approval_status != 'rejected';
```

## MDF Management

```sql
CREATE TABLE mdf_requests (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    partner_id      UUID NOT NULL REFERENCES partners(id),
    requested_by    UUID NOT NULL REFERENCES users(id),
    request_number  VARCHAR(50) NOT NULL,
    title           VARCHAR(255) NOT NULL,
    activity_type   VARCHAR(100) NOT NULL,
    requested_amount NUMERIC(15,2) NOT NULL,
    approved_amount NUMERIC(15,2),
    currency        CHAR(3) NOT NULL DEFAULT 'USD',
    status          VARCHAR(30) NOT NULL DEFAULT 'draft',
    budget_period   JSONB NOT NULL DEFAULT '{}',
    -- budget_period example:
    -- {"fiscal_year": 2026, "quarter": 2, "budget_id": "uuid"}
    schedule        JSONB NOT NULL DEFAULT '{}',
    -- schedule example:
    -- {"planned_start": "2026-06-01", "planned_end": "2026-06-30"}
    claim           JSONB,
    -- claim example (embedded, not separate table):
    -- {
    --   "claimed_amount": 4500.00,
    --   "status": "submitted",
    --   "submitted_at": "2026-07-05T10:00:00Z",
    --   "reviewed_by": "uuid",
    --   "reviewed_at": "2026-07-08T14:30:00Z",
    --   "proof_documents": [
    --     {"file_name": "invoice.pdf", "file_url": "s3://...", "type": "invoice"},
    --     {"file_name": "attendees.csv", "file_url": "s3://...", "type": "attendee_list"},
    --     {"file_name": "event-photo.jpg", "file_url": "s3://...", "type": "photo"}
    --   ],
    --   "paid_at": null,
    --   "rejection_reason": null
    -- }
    custom_fields   JSONB NOT NULL DEFAULT '{}',
    approved_by     UUID REFERENCES users(id),
    approved_at     TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE UNIQUE INDEX idx_mdf_number ON mdf_requests(organisation_id, request_number);
CREATE INDEX idx_mdf_org ON mdf_requests(organisation_id);
CREATE INDEX idx_mdf_partner ON mdf_requests(partner_id);
CREATE INDEX idx_mdf_status ON mdf_requests(organisation_id, status);
```

## Commissions

```sql
CREATE TABLE commissions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    partner_id      UUID NOT NULL REFERENCES partners(id),
    deal_id         UUID REFERENCES deals(id),
    entry_type      VARCHAR(30) NOT NULL,                 -- earned, clawback, adjustment, bonus
    amount          NUMERIC(15,2) NOT NULL,
    currency        CHAR(3) NOT NULL DEFAULT 'USD',
    status          VARCHAR(30) NOT NULL DEFAULT 'pending',
    calculation     JSONB NOT NULL DEFAULT '{}',
    -- calculation example:
    -- {
    --   "plan_name": "Reseller Standard",
    --   "plan_type": "tiered_progressive",
    --   "deal_value": 85000.00,
    --   "tier_applied": "Gold",
    --   "rate": 0.12,
    --   "base_commission": 10200.00,
    --   "bonus": 0,
    --   "clawback_window_days": 90,
    --   "is_recurring": false
    -- }
    clawback_of     UUID REFERENCES commissions(id),
    period_start    DATE,
    period_end      DATE,
    paid_at         TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_commissions_org ON commissions(organisation_id);
CREATE INDEX idx_commissions_partner ON commissions(partner_id);
CREATE INDEX idx_commissions_status ON commissions(organisation_id, status);
```

## Training & Certification

```sql
CREATE TABLE training_modules (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    title           VARCHAR(255) NOT NULL,
    module_type     VARCHAR(50) NOT NULL,
    status          VARCHAR(20) NOT NULL DEFAULT 'draft',
    config          JSONB NOT NULL DEFAULT '{}',
    -- config example:
    -- {
    --   "scorm_package_url": "s3://bucket/package.zip",
    --   "scorm_version": "2004_4th",
    --   "duration_minutes": 45,
    --   "passing_score": 80,
    --   "content_url": "https://...",
    --   "description": "Advanced sales methodology for enterprise deals"
    -- }
    sort_order      INT NOT NULL DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_training_org ON training_modules(organisation_id);

CREATE TABLE certifications (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    status          VARCHAR(20) NOT NULL DEFAULT 'active',
    config          JSONB NOT NULL DEFAULT '{}',
    -- config example:
    -- {
    --   "required_modules": ["uuid1", "uuid2", "uuid3"],
    --   "optional_modules": ["uuid4"],
    --   "min_required": 3,
    --   "passing_score": 75,
    --   "validity_months": 12,
    --   "badge_url": "https://...",
    --   "description": "Certified Sales Partner"
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_certs_org ON certifications(organisation_id);

CREATE TABLE training_progress (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    partner_contact_id UUID NOT NULL REFERENCES partner_contacts(id) ON DELETE CASCADE,
    module_id       UUID NOT NULL REFERENCES training_modules(id),
    status          VARCHAR(30) NOT NULL DEFAULT 'not_started',
    progress        JSONB NOT NULL DEFAULT '{}',
    -- progress example:
    -- {
    --   "score": 85,
    --   "time_spent_seconds": 2700,
    --   "attempts": 2,
    --   "started_at": "2026-03-01T10:00:00Z",
    --   "completed_at": "2026-03-01T10:45:00Z",
    --   "scorm_data": {
    --     "cmi.core.lesson_status": "completed",
    --     "cmi.core.score.raw": "85",
    --     "cmi.suspend_data": "..."
    --   }
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(partner_contact_id, module_id)
);

CREATE INDEX idx_progress_contact ON training_progress(partner_contact_id);

CREATE TABLE certification_awards (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    partner_contact_id UUID NOT NULL REFERENCES partner_contacts(id) ON DELETE CASCADE,
    certification_id UUID NOT NULL REFERENCES certifications(id),
    status          VARCHAR(20) NOT NULL DEFAULT 'active',
    awarded_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    expires_at      TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_cert_awards_contact ON certification_awards(partner_contact_id);
```

## Content Library

```sql
CREATE TABLE content_assets (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    title           VARCHAR(255) NOT NULL,
    asset_type      VARCHAR(50) NOT NULL,
    status          VARCHAR(20) NOT NULL DEFAULT 'published',
    metadata        JSONB NOT NULL DEFAULT '{}',
    -- metadata example:
    -- {
    --   "description": "Enterprise sales deck Q2 2026",
    --   "folder_path": "/sales/presentations",
    --   "file_url": "s3://bucket/sales-deck.pptx",
    --   "file_size_bytes": 2500000,
    --   "mime_type": "application/vnd.openxmlformats-officedocument.presentationml.presentation",
    --   "is_co_brandable": true,
    --   "tags": ["sales", "enterprise", "q2-2026"],
    --   "access": {
    --     "tiers": ["platinum", "gold"],
    --     "programs": ["reseller"]
    --   }
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_content_org ON content_assets(organisation_id);
CREATE INDEX idx_content_metadata ON content_assets USING GIN(metadata jsonb_path_ops);
```

## Integrations (Unified)

```sql
CREATE TABLE integrations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    provider        VARCHAR(50) NOT NULL,
    -- crm: salesforce, hubspot, dynamics365
    -- sso: saml2, oidc
    -- co_sell: aws_ace, microsoft_partner_center, google_cpa
    -- notifications: slack, teams
    category        VARCHAR(30) NOT NULL,                 -- crm, sso, co_sell, notifications, webhook
    is_enabled      BOOLEAN NOT NULL DEFAULT true,
    config          JSONB NOT NULL DEFAULT '{}',
    -- config varies by provider:
    -- Salesforce: {"instance_url": "...", "access_token": "...", "refresh_token": "...", "field_mapping": {...}}
    -- SAML2: {"idp_entity_id": "...", "idp_sso_url": "...", "idp_certificate": "...", "sp_entity_id": "..."}
    -- Slack: {"webhook_url": "...", "channel": "#partner-deals"}
    credentials     JSONB NOT NULL DEFAULT '{}',          -- encrypted at rest
    last_sync_at    TIMESTAMPTZ,
    sync_status     VARCHAR(30),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE UNIQUE INDEX idx_integrations_org_provider ON integrations(organisation_id, provider);
CREATE INDEX idx_integrations_category ON integrations(organisation_id, category);
```

## Activity Feed & Audit

```sql
CREATE TABLE activities (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id),
    actor_id        UUID REFERENCES users(id),
    actor_type      VARCHAR(20) NOT NULL DEFAULT 'user',
    action          VARCHAR(100) NOT NULL,
    entity_type     VARCHAR(50) NOT NULL,
    entity_id       UUID NOT NULL,
    data            JSONB NOT NULL DEFAULT '{}',
    -- data example:
    -- {
    --   "changes": {"stage": {"old": "submitted", "new": "approved"}},
    --   "ip_address": "203.0.113.42",
    --   "user_agent": "Mozilla/5.0...",
    --   "deal_number": "DR-2026-00042",
    --   "partner_name": "Acme Corp"
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_activities_org ON activities(organisation_id, created_at DESC);
CREATE INDEX idx_activities_entity ON activities(entity_type, entity_id);
CREATE INDEX idx_activities_actor ON activities(actor_id);
```

---

## Example JSONB Queries

### Find partners with a specific custom field value

```sql
-- All partners in Germany with annual_target above 200000
SELECT id, company_name, custom_fields->>'annual_target' AS target
FROM partners
WHERE organisation_id = '...'
  AND country_code = 'DE'
  AND (custom_fields->>'annual_target')::numeric > 200000;
```

### Query deals with co-sell on AWS ACE

```sql
-- All deals co-selling through AWS ACE that are not yet synced
SELECT d.id, d.deal_number, d.customer_company,
       d.co_sell->>'opportunity_id' AS ace_opp_id,
       d.co_sell->>'status' AS ace_status
FROM deals d
WHERE d.organisation_id = '...'
  AND d.co_sell->>'type' = 'aws_ace'
  AND d.co_sell->>'status' != 'synced';
```

### Content access filtering by tier

```sql
-- Assets visible to Gold-tier reseller partners
SELECT id, title, asset_type
FROM content_assets
WHERE organisation_id = '...'
  AND status = 'published'
  AND (
    metadata->'access' IS NULL
    OR metadata->'access'->'tiers' ? 'gold'
    OR NOT jsonb_exists(metadata->'access', 'tiers')
  );
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Organisation & Users | 3 | organisations, users, memberships |
| Custom Field Definitions | 1 | custom_field_schemas |
| Partner Management | 3 | programs, partners, contacts |
| Deal Registration | 1 | deals (co-sell, CRM refs, custom fields all in JSONB) |
| MDF Management | 1 | mdf_requests (claims embedded in JSONB) |
| Commissions | 1 | commissions (calculation details in JSONB) |
| Training & Certification | 4 | modules, certifications, progress, awards |
| Content Library | 1 | content_assets (folder path, access rules in JSONB) |
| Integrations | 1 | integrations (all providers unified) |
| Activity Feed | 1 | activities |
| **Total** | **17** | |

---

## Key Design Decisions

1. **custom_field_schemas as a meta-definition table** — rather than hard-coding jurisdiction-specific or program-specific fields, organisations define their own custom fields via `custom_field_schemas`. The application layer uses JSON Schema validation to enforce types, patterns, and required-ness against the `custom_fields` JSONB on each entity. This is the same pattern HubSpot uses for custom properties.

2. **Partner tiers embedded in program config JSONB** — tiers are stored as a JSON array inside `partner_programs.config` rather than a separate table. This eliminates a junction table and makes tier configuration a single document that can be versioned. The trade-off is that `tier_name` and `tier_rank` on the `partners` table are denormalised and must be updated when the program config changes.

3. **MDF claims embedded in mdf_requests** — rather than a separate `mdf_claims` table, the claim lifecycle (submission, review, proof of performance, payment) is stored as a JSONB object within the request. This works because a claim is tightly coupled to its request (1:1 in practice) and the claim lifecycle is short-lived. This cuts one table and simplifies the MDF query pattern.

4. **Unified integrations table** — instead of separate tables for CRM integrations, SSO configurations, and co-sell integrations, a single `integrations` table with `category` and provider-specific `config` JSONB handles all external connections. This reduces table count and makes it easy to add new integration types without migrations.

5. **Commission calculation stored as JSONB** — the `calculation` field on commissions captures the inputs and formula used to compute the commission amount. This serves as a receipt: if commission plan rates change, historical calculations remain correct and auditable.

6. **Content access rules inside metadata** — rather than a separate `content_access_rules` junction table, access control (which tiers and programs can see an asset) is stored inside the asset's `metadata` JSONB. This simplifies content queries at the cost of requiring application-layer enforcement.

7. **Partner stats cached in JSONB** — computed metrics (total deals, revenue, health score) are stored as a `stats` JSONB column on partners. A background job refreshes these periodically. This avoids expensive aggregate queries on every partner list page while keeping the data reasonably fresh.

8. **GIN indexes on JSONB columns** — `jsonb_path_ops` GIN indexes on `custom_fields`, `metadata`, and `co_sell` columns enable efficient containment queries (`@>`) without full table scans. This is critical for the "filter partners by custom field" use case.

9. **Privacy consent embedded per contact** — rather than a separate consents table, GDPR consent records are stored in the `privacy` JSONB column on `partner_contacts`. For most PRM deployments, consent tracking is per-contact and does not require relational querying across contacts.

10. **Folder paths as strings, not a tree** — content assets use a `folder_path` string (e.g., "/sales/presentations") in JSONB rather than a recursive `content_folders` table. This materialised-path approach is simpler to query (`WHERE metadata->>'folder_path' LIKE '/sales/%'`) and avoids recursive CTEs, at the cost of requiring application-layer path management.
