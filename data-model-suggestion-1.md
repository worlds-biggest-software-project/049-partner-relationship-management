# Data Model Suggestion 1: Entity-Centric Normalized Relational

> Project: Partner Relationship Management · Created: 2026-05-12

## Philosophy

This model follows classical normalized relational design principles. Every business concept — partners, deals, MDF claims, training courses, commissions — gets its own table with explicit foreign keys enforcing referential integrity. Junction tables handle many-to-many relationships (e.g., partners to programs, users to roles). The schema is wide, with 40+ tables, but each table has a clear single responsibility.

This approach mirrors how enterprise CRMs like Salesforce model partner data: separate objects for Accounts, Contacts, Opportunities, Leads, and Partners with explicit lookup relationships. It is the most familiar pattern for teams coming from relational database backgrounds and produces the most predictable query performance for complex cross-entity reports.

The normalized design is best suited for teams that prioritise data integrity, expect complex analytical queries across entities (e.g., "show me all partners in EMEA who registered deals above $50K but have not completed certification"), and are willing to accept more migration effort when schema changes occur.

**Best for:** Teams building a compliance-first PRM with complex reporting requirements and predictable query patterns.

**Trade-offs:**
- (+) Maximum data integrity via foreign keys and constraints
- (+) Well-understood by most backend engineers; rich ORM support
- (+) Excellent for complex JOIN-based analytics and ad hoc reporting
- (+) Schema-level documentation of business rules via constraints
- (-) High table count (40+) increases migration complexity
- (-) Adding jurisdiction-specific fields requires schema migrations
- (-) Many-to-many junction tables add query complexity
- (-) Rigid schema makes rapid prototyping slower

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| ISO 3166-1/2 | `jurisdictions` table stores ISO 3166 alpha-2 country codes and subdivision codes for partner geography |
| OAuth 2.0 (RFC 6749) | `oauth_tokens` table stores access/refresh tokens for CRM and hyperscaler integrations |
| SAML 2.0 | `sso_configurations` table stores IdP metadata per partner organisation for federated SSO |
| SCORM 1.2/2004 | `training_modules` table includes SCORM package references; `training_progress` tracks SCORM data model elements |
| OpenAPI 3.1 | API resources map 1:1 to tables; API documentation auto-generated from schema |
| RFC 7807 | Error responses reference entity types matching table names |
| GDPR / CCPA | `data_processing_consents` table tracks lawful basis per partner contact; `data_deletion_requests` for right-to-erasure |
| vCard 4.0 (RFC 6350) | Partner contact export uses vCard-compatible field mapping |

---

## Organisation & Tenancy

```sql
CREATE TABLE organisations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    domain          VARCHAR(255),
    logo_url        TEXT,
    billing_email   VARCHAR(255),
    plan            VARCHAR(50) NOT NULL DEFAULT 'free',  -- free, starter, professional, enterprise
    settings        JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email           VARCHAR(255) NOT NULL UNIQUE,
    full_name       VARCHAR(255) NOT NULL,
    avatar_url      TEXT,
    password_hash   TEXT,                                 -- null if SSO-only
    email_verified  BOOLEAN NOT NULL DEFAULT false,
    last_login_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE memberships (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    organisation_id UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    role            VARCHAR(50) NOT NULL DEFAULT 'member', -- owner, admin, manager, member, viewer
    status          VARCHAR(20) NOT NULL DEFAULT 'active', -- active, suspended, invited
    invited_at      TIMESTAMPTZ,
    joined_at       TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(user_id, organisation_id)
);

CREATE INDEX idx_memberships_org ON memberships(organisation_id);
CREATE INDEX idx_memberships_user ON memberships(user_id);
```

## Partner Management

```sql
CREATE TABLE partner_programs (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    program_type    VARCHAR(50) NOT NULL,                 -- reseller, referral, affiliate, technology, msp
    status          VARCHAR(20) NOT NULL DEFAULT 'active',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_partner_programs_org ON partner_programs(organisation_id);

CREATE TABLE partner_tiers (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    program_id      UUID NOT NULL REFERENCES partner_programs(id) ON DELETE CASCADE,
    name            VARCHAR(100) NOT NULL,                -- e.g., Platinum, Gold, Silver, Bronze, Registered
    rank            INT NOT NULL,                         -- numeric ordering (1 = highest)
    min_revenue     NUMERIC(15,2),                        -- auto-qualification threshold
    min_deals       INT,
    min_certifications INT,
    benefits        TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_partner_tiers_program ON partner_tiers(program_id);

CREATE TABLE partners (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    company_name    VARCHAR(255) NOT NULL,
    legal_name      VARCHAR(255),
    website         TEXT,
    logo_url        TEXT,
    primary_contact_name  VARCHAR(255),
    primary_contact_email VARCHAR(255),
    primary_contact_phone VARCHAR(50),
    address_line1   VARCHAR(255),
    address_line2   VARCHAR(255),
    city            VARCHAR(100),
    state_province  VARCHAR(100),
    postal_code     VARCHAR(20),
    country_code    CHAR(2),                              -- ISO 3166-1 alpha-2
    tier_id         UUID REFERENCES partner_tiers(id),
    program_id      UUID NOT NULL REFERENCES partner_programs(id),
    status          VARCHAR(30) NOT NULL DEFAULT 'prospect', -- prospect, onboarding, active, inactive, churned
    onboarded_at    TIMESTAMPTZ,
    partner_since   DATE,
    notes           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_partners_org ON partners(organisation_id);
CREATE INDEX idx_partners_program ON partners(program_id);
CREATE INDEX idx_partners_tier ON partners(tier_id);
CREATE INDEX idx_partners_status ON partners(organisation_id, status);
CREATE INDEX idx_partners_country ON partners(country_code);

CREATE TABLE partner_contacts (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    partner_id      UUID NOT NULL REFERENCES partners(id) ON DELETE CASCADE,
    full_name       VARCHAR(255) NOT NULL,
    email           VARCHAR(255) NOT NULL,
    phone           VARCHAR(50),
    job_title       VARCHAR(255),
    role            VARCHAR(50),                          -- primary, technical, billing, marketing
    is_portal_user  BOOLEAN NOT NULL DEFAULT false,
    user_id         UUID REFERENCES users(id),            -- linked user account if portal access granted
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_partner_contacts_partner ON partner_contacts(partner_id);
CREATE INDEX idx_partner_contacts_email ON partner_contacts(email);
```

## Deal Registration

```sql
CREATE TABLE deals (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    partner_id      UUID NOT NULL REFERENCES partners(id),
    registered_by   UUID NOT NULL REFERENCES users(id),
    deal_number     VARCHAR(50) NOT NULL,                 -- human-readable, e.g., DR-2026-00042
    customer_name   VARCHAR(255) NOT NULL,
    customer_email  VARCHAR(255),
    customer_phone  VARCHAR(50),
    customer_company VARCHAR(255),
    customer_website TEXT,
    customer_country CHAR(2),                             -- ISO 3166-1 alpha-2
    product_interest TEXT,
    estimated_value NUMERIC(15,2),
    currency        CHAR(3) NOT NULL DEFAULT 'USD',       -- ISO 4217
    expected_close_date DATE,
    actual_close_date DATE,
    stage           VARCHAR(50) NOT NULL DEFAULT 'submitted',
    -- submitted, under_review, approved, rejected, won, lost, expired
    approval_status VARCHAR(30) NOT NULL DEFAULT 'pending',
    -- pending, approved, rejected
    approved_by     UUID REFERENCES users(id),
    approved_at     TIMESTAMPTZ,
    rejection_reason TEXT,
    expiry_date     DATE,                                 -- deal protection expiry
    notes           TEXT,
    crm_opportunity_id VARCHAR(255),                      -- Salesforce/HubSpot opportunity ID
    co_sell_type    VARCHAR(50),                          -- aws_ace, microsoft_partner_center, google_cpa, none
    co_sell_ref     VARCHAR(255),                         -- external co-sell reference ID
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE UNIQUE INDEX idx_deals_number ON deals(organisation_id, deal_number);
CREATE INDEX idx_deals_org ON deals(organisation_id);
CREATE INDEX idx_deals_partner ON deals(partner_id);
CREATE INDEX idx_deals_stage ON deals(organisation_id, stage);
CREATE INDEX idx_deals_customer ON deals(organisation_id, customer_company);

-- Duplicate detection: check for matching customer + product within an org
CREATE INDEX idx_deals_dedup ON deals(organisation_id, customer_email, product_interest)
    WHERE approval_status != 'rejected';

CREATE TABLE deal_activities (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    deal_id         UUID NOT NULL REFERENCES deals(id) ON DELETE CASCADE,
    actor_id        UUID NOT NULL REFERENCES users(id),
    activity_type   VARCHAR(50) NOT NULL,                 -- status_change, note_added, document_attached, comment
    previous_value  TEXT,
    new_value       TEXT,
    comment         TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_deal_activities_deal ON deal_activities(deal_id);
CREATE INDEX idx_deal_activities_created ON deal_activities(deal_id, created_at);
```

## Market Development Funds (MDF)

```sql
CREATE TABLE mdf_budgets (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    partner_id      UUID REFERENCES partners(id),         -- null = org-wide budget
    fiscal_year     INT NOT NULL,
    fiscal_quarter  INT,                                  -- 1-4; null = annual budget
    total_amount    NUMERIC(15,2) NOT NULL,
    currency        CHAR(3) NOT NULL DEFAULT 'USD',
    allocated_amount NUMERIC(15,2) NOT NULL DEFAULT 0,
    spent_amount    NUMERIC(15,2) NOT NULL DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_mdf_budgets_org ON mdf_budgets(organisation_id);
CREATE INDEX idx_mdf_budgets_partner ON mdf_budgets(partner_id);

CREATE TABLE mdf_requests (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    partner_id      UUID NOT NULL REFERENCES partners(id),
    budget_id       UUID REFERENCES mdf_budgets(id),
    requested_by    UUID NOT NULL REFERENCES users(id),
    request_number  VARCHAR(50) NOT NULL,                 -- MDF-2026-00015
    title           VARCHAR(255) NOT NULL,
    description     TEXT,
    activity_type   VARCHAR(100) NOT NULL,                -- event, digital_campaign, content, webinar, trade_show
    planned_start   DATE,
    planned_end     DATE,
    requested_amount NUMERIC(15,2) NOT NULL,
    approved_amount NUMERIC(15,2),
    currency        CHAR(3) NOT NULL DEFAULT 'USD',
    status          VARCHAR(30) NOT NULL DEFAULT 'draft',
    -- draft, submitted, under_review, approved, rejected, completed, cancelled
    approved_by     UUID REFERENCES users(id),
    approved_at     TIMESTAMPTZ,
    rejection_reason TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE UNIQUE INDEX idx_mdf_requests_number ON mdf_requests(organisation_id, request_number);
CREATE INDEX idx_mdf_requests_org ON mdf_requests(organisation_id);
CREATE INDEX idx_mdf_requests_partner ON mdf_requests(partner_id);
CREATE INDEX idx_mdf_requests_status ON mdf_requests(organisation_id, status);

CREATE TABLE mdf_claims (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    request_id      UUID NOT NULL REFERENCES mdf_requests(id),
    partner_id      UUID NOT NULL REFERENCES partners(id),
    claim_number    VARCHAR(50) NOT NULL,
    claimed_amount  NUMERIC(15,2) NOT NULL,
    currency        CHAR(3) NOT NULL DEFAULT 'USD',
    status          VARCHAR(30) NOT NULL DEFAULT 'submitted',
    -- submitted, under_review, approved, rejected, paid
    proof_of_performance TEXT,                            -- description of proof submitted
    reviewed_by     UUID REFERENCES users(id),
    reviewed_at     TIMESTAMPTZ,
    paid_at         TIMESTAMPTZ,
    rejection_reason TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_mdf_claims_request ON mdf_claims(request_id);
CREATE INDEX idx_mdf_claims_partner ON mdf_claims(partner_id);

CREATE TABLE mdf_claim_documents (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    claim_id        UUID NOT NULL REFERENCES mdf_claims(id) ON DELETE CASCADE,
    file_name       VARCHAR(255) NOT NULL,
    file_url        TEXT NOT NULL,
    file_size_bytes BIGINT,
    mime_type       VARCHAR(100),
    document_type   VARCHAR(50),                          -- invoice, screenshot, attendee_list, report, receipt
    uploaded_by     UUID NOT NULL REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_mdf_claim_docs_claim ON mdf_claim_documents(claim_id);
```

## Training & Certification

```sql
CREATE TABLE training_modules (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    title           VARCHAR(255) NOT NULL,
    description     TEXT,
    module_type     VARCHAR(50) NOT NULL,                 -- scorm, video, document, quiz, external_link
    scorm_package_url TEXT,                               -- S3/GCS URL for SCORM zip
    scorm_version   VARCHAR(20),                          -- '1.2' or '2004_4th'
    content_url     TEXT,
    duration_minutes INT,
    is_required     BOOLEAN NOT NULL DEFAULT false,
    sort_order      INT NOT NULL DEFAULT 0,
    status          VARCHAR(20) NOT NULL DEFAULT 'draft', -- draft, published, archived
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_training_modules_org ON training_modules(organisation_id);

CREATE TABLE certifications (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    validity_months INT,                                  -- null = never expires
    required_modules INT NOT NULL DEFAULT 1,
    passing_score   NUMERIC(5,2),                         -- minimum score percentage
    badge_url       TEXT,
    status          VARCHAR(20) NOT NULL DEFAULT 'active',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_certifications_org ON certifications(organisation_id);

CREATE TABLE certification_modules (
    certification_id UUID NOT NULL REFERENCES certifications(id) ON DELETE CASCADE,
    module_id        UUID NOT NULL REFERENCES training_modules(id) ON DELETE CASCADE,
    is_required      BOOLEAN NOT NULL DEFAULT true,
    sort_order       INT NOT NULL DEFAULT 0,
    PRIMARY KEY (certification_id, module_id)
);

CREATE TABLE training_progress (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    partner_contact_id UUID NOT NULL REFERENCES partner_contacts(id) ON DELETE CASCADE,
    module_id       UUID NOT NULL REFERENCES training_modules(id),
    status          VARCHAR(30) NOT NULL DEFAULT 'not_started',
    -- not_started, in_progress, completed, failed
    score           NUMERIC(5,2),
    time_spent_seconds INT,
    attempts        INT NOT NULL DEFAULT 0,
    started_at      TIMESTAMPTZ,
    completed_at    TIMESTAMPTZ,
    scorm_data      JSONB,                                -- SCORM cmi data model suspend/bookmark data
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(partner_contact_id, module_id)
);

CREATE INDEX idx_training_progress_contact ON training_progress(partner_contact_id);
CREATE INDEX idx_training_progress_module ON training_progress(module_id);

CREATE TABLE certification_awards (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    partner_contact_id UUID NOT NULL REFERENCES partner_contacts(id) ON DELETE CASCADE,
    certification_id UUID NOT NULL REFERENCES certifications(id),
    awarded_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    expires_at      TIMESTAMPTZ,
    status          VARCHAR(20) NOT NULL DEFAULT 'active', -- active, expired, revoked
    certificate_url TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_cert_awards_contact ON certification_awards(partner_contact_id);
CREATE INDEX idx_cert_awards_cert ON certification_awards(certification_id);
```

## Commissions & Payouts

```sql
CREATE TABLE commission_plans (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    program_id      UUID REFERENCES partner_programs(id),
    name            VARCHAR(255) NOT NULL,
    plan_type       VARCHAR(50) NOT NULL,                 -- flat_rate, percentage, tiered_progressive, tiered_retroactive
    base_rate       NUMERIC(8,4),                         -- e.g., 0.10 for 10%
    currency        CHAR(3) NOT NULL DEFAULT 'USD',
    clawback_days   INT DEFAULT 90,                       -- days within which churn triggers clawback
    is_recurring    BOOLEAN NOT NULL DEFAULT false,       -- commissions on renewals
    status          VARCHAR(20) NOT NULL DEFAULT 'active',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_commission_plans_org ON commission_plans(organisation_id);

CREATE TABLE commission_tiers (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    plan_id         UUID NOT NULL REFERENCES commission_plans(id) ON DELETE CASCADE,
    min_revenue     NUMERIC(15,2) NOT NULL,               -- threshold start
    max_revenue     NUMERIC(15,2),                        -- null = unlimited
    rate            NUMERIC(8,4) NOT NULL,                -- commission rate for this tier
    sort_order      INT NOT NULL DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_commission_tiers_plan ON commission_tiers(plan_id);

CREATE TABLE commissions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    partner_id      UUID NOT NULL REFERENCES partners(id),
    deal_id         UUID REFERENCES deals(id),
    plan_id         UUID NOT NULL REFERENCES commission_plans(id),
    amount          NUMERIC(15,2) NOT NULL,
    currency        CHAR(3) NOT NULL DEFAULT 'USD',
    status          VARCHAR(30) NOT NULL DEFAULT 'pending',
    -- pending, approved, paid, clawed_back
    period_start    DATE,
    period_end      DATE,
    clawback_of     UUID REFERENCES commissions(id),      -- self-reference for clawback entries
    paid_at         TIMESTAMPTZ,
    notes           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_commissions_org ON commissions(organisation_id);
CREATE INDEX idx_commissions_partner ON commissions(partner_id);
CREATE INDEX idx_commissions_deal ON commissions(deal_id);
CREATE INDEX idx_commissions_status ON commissions(organisation_id, status);
```

## Content Library

```sql
CREATE TABLE content_folders (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    parent_id       UUID REFERENCES content_folders(id),
    name            VARCHAR(255) NOT NULL,
    sort_order      INT NOT NULL DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_content_folders_org ON content_folders(organisation_id);
CREATE INDEX idx_content_folders_parent ON content_folders(parent_id);

CREATE TABLE content_assets (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    folder_id       UUID REFERENCES content_folders(id),
    title           VARCHAR(255) NOT NULL,
    description     TEXT,
    asset_type      VARCHAR(50) NOT NULL,                 -- document, video, presentation, template, image, link
    file_url        TEXT,
    file_size_bytes BIGINT,
    mime_type       VARCHAR(100),
    external_url    TEXT,
    is_co_brandable BOOLEAN NOT NULL DEFAULT false,
    tags            TEXT[],                                -- PostgreSQL array for tag-based filtering
    status          VARCHAR(20) NOT NULL DEFAULT 'published',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_content_assets_org ON content_assets(organisation_id);
CREATE INDEX idx_content_assets_folder ON content_assets(folder_id);
CREATE INDEX idx_content_assets_tags ON content_assets USING GIN(tags);

-- RBAC for content: which tiers/programs can see which assets
CREATE TABLE content_access_rules (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    asset_id        UUID NOT NULL REFERENCES content_assets(id) ON DELETE CASCADE,
    tier_id         UUID REFERENCES partner_tiers(id),    -- null = all tiers
    program_id      UUID REFERENCES partner_programs(id), -- null = all programs
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_content_access_asset ON content_access_rules(asset_id);
```

## Integrations & SSO

```sql
CREATE TABLE crm_integrations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    provider        VARCHAR(50) NOT NULL,                 -- salesforce, hubspot, dynamics365, pipedrive
    instance_url    TEXT,
    access_token    TEXT,                                  -- encrypted at rest
    refresh_token   TEXT,                                  -- encrypted at rest
    token_expires_at TIMESTAMPTZ,
    sync_enabled    BOOLEAN NOT NULL DEFAULT true,
    last_sync_at    TIMESTAMPTZ,
    sync_status     VARCHAR(30),
    field_mapping   JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE UNIQUE INDEX idx_crm_integrations_org_provider ON crm_integrations(organisation_id, provider);

CREATE TABLE sso_configurations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    partner_id      UUID REFERENCES partners(id),         -- null = org-level SSO
    protocol        VARCHAR(20) NOT NULL,                 -- saml2, oidc
    idp_entity_id   VARCHAR(500),
    idp_sso_url     TEXT,
    idp_certificate TEXT,
    sp_entity_id    VARCHAR(500),
    is_enabled      BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_sso_config_org ON sso_configurations(organisation_id);

CREATE TABLE co_sell_integrations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    provider        VARCHAR(50) NOT NULL,                 -- aws_ace, microsoft_partner_center, google_cpa
    credentials     JSONB NOT NULL DEFAULT '{}',          -- encrypted; provider-specific auth config
    sync_enabled    BOOLEAN NOT NULL DEFAULT true,
    last_sync_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE UNIQUE INDEX idx_cosell_org_provider ON co_sell_integrations(organisation_id, provider);
```

## Notifications & Audit

```sql
CREATE TABLE notifications (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    user_id         UUID NOT NULL REFERENCES users(id),
    title           VARCHAR(255) NOT NULL,
    body            TEXT,
    notification_type VARCHAR(50) NOT NULL,               -- deal_registered, deal_approved, mdf_approved, training_completed
    entity_type     VARCHAR(50),                          -- deal, mdf_request, mdf_claim, training_module
    entity_id       UUID,
    is_read         BOOLEAN NOT NULL DEFAULT false,
    read_at         TIMESTAMPTZ,
    channel         VARCHAR(20) NOT NULL DEFAULT 'in_app', -- in_app, email, slack, teams
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_notifications_user ON notifications(user_id, is_read);
CREATE INDEX idx_notifications_org ON notifications(organisation_id);

CREATE TABLE audit_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id),
    actor_id        UUID REFERENCES users(id),
    actor_type      VARCHAR(20) NOT NULL DEFAULT 'user',  -- user, system, integration
    action          VARCHAR(100) NOT NULL,                -- e.g., deal.created, partner.tier_changed, mdf.approved
    entity_type     VARCHAR(50) NOT NULL,
    entity_id       UUID NOT NULL,
    changes         JSONB,                                -- {field: {old: x, new: y}}
    ip_address      INET,
    user_agent      TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_audit_log_org ON audit_log(organisation_id);
CREATE INDEX idx_audit_log_entity ON audit_log(entity_type, entity_id);
CREATE INDEX idx_audit_log_actor ON audit_log(actor_id);
CREATE INDEX idx_audit_log_created ON audit_log(organisation_id, created_at);
```

## Data Compliance

```sql
CREATE TABLE data_processing_consents (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    partner_contact_id UUID NOT NULL REFERENCES partner_contacts(id) ON DELETE CASCADE,
    consent_type    VARCHAR(50) NOT NULL,                 -- marketing, data_processing, analytics
    lawful_basis    VARCHAR(50) NOT NULL,                 -- consent, legitimate_interest, contract
    granted_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    revoked_at      TIMESTAMPTZ,
    ip_address      INET,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_consents_contact ON data_processing_consents(partner_contact_id);

CREATE TABLE data_deletion_requests (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id),
    requester_email VARCHAR(255) NOT NULL,
    request_type    VARCHAR(30) NOT NULL,                 -- erasure, export, rectification
    status          VARCHAR(30) NOT NULL DEFAULT 'pending',
    -- pending, in_progress, completed, rejected
    completed_at    TIMESTAMPTZ,
    notes           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_deletion_requests_org ON data_deletion_requests(organisation_id);
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Organisation & Tenancy | 3 | organisations, users, memberships |
| Partner Management | 5 | programs, tiers, partners, contacts, plus junction |
| Deal Registration | 2 | deals, deal_activities |
| MDF Management | 4 | budgets, requests, claims, claim_documents |
| Training & Certification | 5 | modules, certifications, junction, progress, awards |
| Commissions | 3 | plans, tiers, commissions |
| Content Library | 3 | folders, assets, access_rules |
| Integrations & SSO | 3 | crm_integrations, sso_configurations, co_sell_integrations |
| Notifications & Audit | 2 | notifications, audit_log |
| Data Compliance | 2 | consents, deletion_requests |
| **Total** | **32** | |

---

## Key Design Decisions

1. **UUID primary keys everywhere** — enables distributed ID generation, safe for multi-region deployments, and prevents sequential ID enumeration attacks on the partner portal API.

2. **organisation_id on every tenant-scoped table** — supports PostgreSQL Row-Level Security (RLS) policies for strict tenant isolation; every query can be filtered by organisation without complex joins.

3. **Separate partner_contacts from users** — a partner contact may or may not have portal access; linking them to a user account is optional. This avoids creating orphan user records for contacts who never log in.

4. **Deal duplicate detection via partial index** — the `idx_deals_dedup` index on (org, customer_email, product_interest) WHERE approval_status != 'rejected' enables fast duplicate detection at registration time without a separate dedup table.

5. **MDF budget / request / claim as three separate tables** — models the full MDF lifecycle (allocation, proposal, reimbursement) with distinct status machines rather than overloading a single table with polymorphic status values.

6. **Commission clawback as self-referencing row** — a clawback is modeled as a negative commission entry pointing to the original via `clawback_of`, keeping the commissions ledger append-only and auditable.

7. **JSONB used sparingly** — only for genuinely variable data (SCORM suspend data, CRM field mappings, integration credentials, audit log change diffs). Core business fields are always typed columns.

8. **PostgreSQL arrays for tags** — content asset tags use `TEXT[]` with a GIN index for fast containment queries (`WHERE tags @> ARRAY['sales-deck']`), avoiding a separate tags junction table for this simple use case.

9. **Audit log is append-only with JSONB changes** — captures before/after state for every mutation, enabling compliance reporting without reconstructing state from multiple tables.

10. **Content access rules use nullable foreign keys** — a rule with `tier_id = NULL` and `program_id = NULL` means "visible to all"; specific tier or program restrictions are additive filters.
