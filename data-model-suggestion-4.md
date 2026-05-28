# Data Model Suggestion 4: Graph-Relational Hybrid

> Project: Partner Relationship Management · Created: 2026-05-12

## Philosophy

This model combines a relational backbone for operational CRUD with a property graph layer for relationship-heavy queries. The relational tables handle day-to-day operations (deal registration, MDF claims, training progress), while a lightweight graph structure (`graph_nodes` and `graph_edges`) captures the complex web of relationships between partners, vendors, customers, deals, and people.

PRM is fundamentally a relationship management problem. Partners refer customers, customers buy from multiple partners, partners have sub-partners (distributors to resellers), individuals move between partner organisations, and deals involve multi-party co-sell arrangements across cloud hyperscalers. These relationship patterns are awkward to query in a purely relational model — they require recursive CTEs, multiple self-joins, or denormalised relationship tables. A graph layer makes these queries natural: "find all partners who share customers with Partner X," "trace the referral chain for this deal," or "identify conflict-of-interest overlaps between partner contacts and customer contacts."

The graph layer is implemented in PostgreSQL using two tables (`graph_nodes` and `graph_edges`) with JSONB properties, avoiding the need for a separate graph database. For teams that later need dedicated graph infrastructure, the same data can be projected into Neo4j or Amazon Neptune without changing the relational core.

**Best for:** Teams building a PRM where partner ecosystem mapping, co-sell network analysis, conflict detection, and relationship intelligence are strategic differentiators.

**Trade-offs:**

- (+) Natural model for partner ecosystem mapping and overlap analysis
- (+) Powerful for conflict-of-interest detection across partner and customer contacts
- (+) Enables co-sell network analysis — find partners with complementary customer bases
- (+) Graph traversal queries (shortest path, connected components) reveal hidden ecosystem patterns
- (+) Relational core keeps operational queries simple and performant
- (+) Graph layer can power AI recommendations ("partners like you also sell to...")
- (-) Two layers to maintain — relational tables and graph nodes/edges must stay in sync
- (-) Graph query patterns (recursive CTEs in PostgreSQL) have different performance characteristics than standard SQL
- (-) Team needs to learn graph thinking — not all engineers are comfortable with node/edge mental models
- (-) Dual-write concern: every partner or deal change may require updating both a relational table and graph edges
- (-) PostgreSQL graph queries are less performant than dedicated graph databases for deep traversals (6+ hops)

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| ISO 3166-1/2 | Country codes on partner and customer nodes, used for geographic graph partitioning |
| ISO 4217 | Currency codes on deal and commission edges |
| OAuth 2.0 (RFC 6749) | Integration credentials stored in relational `integrations` table |
| SAML 2.0 | SSO configuration per partner organisation node |
| SCORM 1.2/2004 | Training progress in relational tables; certification edges link contacts to certifications |
| OpenAPI 3.1 | REST API for relational CRUD; separate GraphQL endpoint for graph traversal queries |
| W3C RDF / Property Graph | Graph layer follows the Labeled Property Graph model (nodes with labels and properties, directed edges with types and properties) |
| GDPR / CCPA | Graph edges involving personal data respect deletion cascades; contact node deletion removes all incident edges |

---

## Organisation & Users (Relational)

```sql
CREATE TABLE organisations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    domain          VARCHAR(255),
    plan            VARCHAR(50) NOT NULL DEFAULT 'free',
    settings        JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email           VARCHAR(255) NOT NULL UNIQUE,
    full_name       VARCHAR(255) NOT NULL,
    avatar_url      TEXT,
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

CREATE INDEX idx_memberships_org ON memberships(organisation_id);
CREATE INDEX idx_memberships_user ON memberships(user_id);
```

## Graph Layer

```sql
-- Generic graph node table. Every entity that participates in relationships
-- gets a corresponding node. The node_type + entity_id pair links back to
-- the relational table for full CRUD.
CREATE TABLE graph_nodes (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    node_type       VARCHAR(50) NOT NULL,
    -- partner, customer, contact, deal, product, program, certification, hyperscaler
    entity_id       UUID NOT NULL,                        -- FK to the relational table (enforced at app layer)
    label           VARCHAR(255) NOT NULL,                -- display name for graph visualisation
    properties      JSONB NOT NULL DEFAULT '{}',
    -- properties vary by node_type:
    -- partner:  {"company_name": "Acme", "country": "US", "tier": "Gold", "status": "active"}
    -- customer: {"company_name": "BigCorp", "industry": "finance", "country": "GB", "arr": 120000}
    -- contact:  {"full_name": "Jane Doe", "email": "jane@acme.com", "role": "sales_director"}
    -- deal:     {"deal_number": "DR-2026-042", "value": 85000, "stage": "approved", "currency": "USD"}
    -- product:  {"name": "Enterprise Plan", "category": "saas"}
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(organisation_id, node_type, entity_id)
);

CREATE INDEX idx_graph_nodes_org ON graph_nodes(organisation_id);
CREATE INDEX idx_graph_nodes_type ON graph_nodes(organisation_id, node_type);
CREATE INDEX idx_graph_nodes_entity ON graph_nodes(entity_id);
CREATE INDEX idx_graph_nodes_props ON graph_nodes USING GIN(properties jsonb_path_ops);

-- Directed edges between nodes. Each edge has a type that defines the
-- relationship semantics.
CREATE TABLE graph_edges (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    source_node_id  UUID NOT NULL REFERENCES graph_nodes(id) ON DELETE CASCADE,
    target_node_id  UUID NOT NULL REFERENCES graph_nodes(id) ON DELETE CASCADE,
    edge_type       VARCHAR(100) NOT NULL,
    -- PARTNER_REFERS_CUSTOMER, PARTNER_REGISTERED_DEAL, DEAL_FOR_CUSTOMER,
    -- CONTACT_WORKS_AT, PARTNER_DISTRIBUTES_TO, PARTNER_CO_SELLS_WITH,
    -- DEAL_INVOLVES_PRODUCT, PARTNER_HAS_CERTIFICATION, CONTACT_MANAGES_DEAL,
    -- PARTNER_CHILD_OF (distributor hierarchy), CUSTOMER_OVERLAPS_WITH
    properties      JSONB NOT NULL DEFAULT '{}',
    -- properties vary by edge_type:
    -- PARTNER_REFERS_CUSTOMER: {"first_referral_date": "2026-01-15", "deals_count": 3, "total_value": 250000}
    -- PARTNER_REGISTERED_DEAL: {"registered_at": "2026-04-01", "commission_rate": 0.10}
    -- PARTNER_DISTRIBUTES_TO: {"since": "2024-06-01", "region": "EMEA", "agreement_url": "..."}
    -- CONTACT_WORKS_AT: {"role": "sales_director", "started": "2025-03-01"}
    -- PARTNER_CO_SELLS_WITH: {"hyperscaler": "aws", "shared_customers": 5}
    weight          NUMERIC(10,4) DEFAULT 1.0,            -- for weighted graph algorithms
    valid_from      TIMESTAMPTZ NOT NULL DEFAULT now(),   -- temporal: when relationship started
    valid_to        TIMESTAMPTZ,                          -- null = currently active
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_graph_edges_org ON graph_edges(organisation_id);
CREATE INDEX idx_graph_edges_source ON graph_edges(source_node_id);
CREATE INDEX idx_graph_edges_target ON graph_edges(target_node_id);
CREATE INDEX idx_graph_edges_type ON graph_edges(organisation_id, edge_type);
CREATE INDEX idx_graph_edges_temporal ON graph_edges(valid_from, valid_to)
    WHERE valid_to IS NULL;
CREATE INDEX idx_graph_edges_props ON graph_edges USING GIN(properties jsonb_path_ops);

-- Materialised view for bidirectional edge traversal
CREATE MATERIALIZED VIEW graph_edges_undirected AS
SELECT id, organisation_id, source_node_id AS node_a, target_node_id AS node_b,
       edge_type, properties, weight, valid_from, valid_to
FROM graph_edges
WHERE valid_to IS NULL
UNION ALL
SELECT id, organisation_id, target_node_id AS node_a, source_node_id AS node_b,
       edge_type, properties, weight, valid_from, valid_to
FROM graph_edges
WHERE valid_to IS NULL;

CREATE INDEX idx_edges_undir_node_a ON graph_edges_undirected(node_a);
CREATE INDEX idx_edges_undir_org ON graph_edges_undirected(organisation_id);
```

## Partner Management (Relational)

```sql
CREATE TABLE partner_programs (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    program_type    VARCHAR(50) NOT NULL,
    status          VARCHAR(20) NOT NULL DEFAULT 'active',
    config          JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_programs_org ON partner_programs(organisation_id);

CREATE TABLE partner_tiers (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    program_id      UUID NOT NULL REFERENCES partner_programs(id) ON DELETE CASCADE,
    name            VARCHAR(100) NOT NULL,
    rank            INT NOT NULL,
    min_revenue     NUMERIC(15,2),
    min_deals       INT,
    min_certifications INT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_tiers_program ON partner_tiers(program_id);

CREATE TABLE partners (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    program_id      UUID NOT NULL REFERENCES partner_programs(id),
    tier_id         UUID REFERENCES partner_tiers(id),
    company_name    VARCHAR(255) NOT NULL,
    legal_name      VARCHAR(255),
    website         TEXT,
    country_code    CHAR(2),
    status          VARCHAR(30) NOT NULL DEFAULT 'prospect',
    partner_since   DATE,
    graph_node_id   UUID REFERENCES graph_nodes(id),      -- link to graph layer
    custom_fields   JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_partners_org ON partners(organisation_id);
CREATE INDEX idx_partners_program ON partners(program_id);
CREATE INDEX idx_partners_status ON partners(organisation_id, status);

CREATE TABLE partner_contacts (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    partner_id      UUID NOT NULL REFERENCES partners(id) ON DELETE CASCADE,
    full_name       VARCHAR(255) NOT NULL,
    email           VARCHAR(255) NOT NULL,
    phone           VARCHAR(50),
    job_title       VARCHAR(255),
    role            VARCHAR(50),
    user_id         UUID REFERENCES users(id),
    graph_node_id   UUID REFERENCES graph_nodes(id),      -- link to graph layer
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_contacts_partner ON partner_contacts(partner_id);
CREATE INDEX idx_contacts_email ON partner_contacts(email);

-- Customers are tracked primarily for deal attribution and graph overlap analysis.
-- They are not full CRM records; the CRM integration provides the authoritative customer data.
CREATE TABLE customers (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    company_name    VARCHAR(255) NOT NULL,
    website         TEXT,
    industry        VARCHAR(100),
    country_code    CHAR(2),
    crm_account_id  VARCHAR(255),
    graph_node_id   UUID REFERENCES graph_nodes(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_customers_org ON customers(organisation_id);
CREATE INDEX idx_customers_crm ON customers(organisation_id, crm_account_id);
```

## Deal Registration (Relational)

```sql
CREATE TABLE deals (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    partner_id      UUID NOT NULL REFERENCES partners(id),
    customer_id     UUID REFERENCES customers(id),        -- link to customer for graph edges
    registered_by   UUID NOT NULL REFERENCES users(id),
    deal_number     VARCHAR(50) NOT NULL,
    customer_name   VARCHAR(255) NOT NULL,
    customer_email  VARCHAR(255),
    customer_company VARCHAR(255),
    customer_country CHAR(2),
    estimated_value NUMERIC(15,2),
    currency        CHAR(3) NOT NULL DEFAULT 'USD',
    expected_close_date DATE,
    actual_close_date DATE,
    stage           VARCHAR(50) NOT NULL DEFAULT 'submitted',
    approval_status VARCHAR(30) NOT NULL DEFAULT 'pending',
    approved_by     UUID REFERENCES users(id),
    approved_at     TIMESTAMPTZ,
    expiry_date     DATE,
    co_sell_type    VARCHAR(50),
    co_sell_ref     VARCHAR(255),
    crm_opportunity_id VARCHAR(255),
    graph_node_id   UUID REFERENCES graph_nodes(id),
    notes           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE UNIQUE INDEX idx_deals_number ON deals(organisation_id, deal_number);
CREATE INDEX idx_deals_org ON deals(organisation_id);
CREATE INDEX idx_deals_partner ON deals(partner_id);
CREATE INDEX idx_deals_customer ON deals(customer_id);
CREATE INDEX idx_deals_stage ON deals(organisation_id, stage);
CREATE INDEX idx_deals_dedup ON deals(organisation_id, customer_email)
    WHERE approval_status != 'rejected';
```

## MDF Management (Relational)

```sql
CREATE TABLE mdf_budgets (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    partner_id      UUID REFERENCES partners(id),
    fiscal_year     INT NOT NULL,
    fiscal_quarter  INT,
    total_amount    NUMERIC(15,2) NOT NULL,
    currency        CHAR(3) NOT NULL DEFAULT 'USD',
    allocated_amount NUMERIC(15,2) NOT NULL DEFAULT 0,
    spent_amount    NUMERIC(15,2) NOT NULL DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_mdf_budgets_org ON mdf_budgets(organisation_id);

CREATE TABLE mdf_requests (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    partner_id      UUID NOT NULL REFERENCES partners(id),
    budget_id       UUID REFERENCES mdf_budgets(id),
    requested_by    UUID NOT NULL REFERENCES users(id),
    request_number  VARCHAR(50) NOT NULL,
    title           VARCHAR(255) NOT NULL,
    activity_type   VARCHAR(100) NOT NULL,
    requested_amount NUMERIC(15,2) NOT NULL,
    approved_amount NUMERIC(15,2),
    currency        CHAR(3) NOT NULL DEFAULT 'USD',
    status          VARCHAR(30) NOT NULL DEFAULT 'draft',
    planned_start   DATE,
    planned_end     DATE,
    approved_by     UUID REFERENCES users(id),
    approved_at     TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE UNIQUE INDEX idx_mdf_number ON mdf_requests(organisation_id, request_number);
CREATE INDEX idx_mdf_org ON mdf_requests(organisation_id);
CREATE INDEX idx_mdf_partner ON mdf_requests(partner_id);

CREATE TABLE mdf_claims (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    request_id      UUID NOT NULL REFERENCES mdf_requests(id),
    claimed_amount  NUMERIC(15,2) NOT NULL,
    currency        CHAR(3) NOT NULL DEFAULT 'USD',
    status          VARCHAR(30) NOT NULL DEFAULT 'submitted',
    reviewed_by     UUID REFERENCES users(id),
    reviewed_at     TIMESTAMPTZ,
    paid_at         TIMESTAMPTZ,
    proof_documents JSONB NOT NULL DEFAULT '[]',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_mdf_claims_request ON mdf_claims(request_id);
```

## Training & Certification (Relational)

```sql
CREATE TABLE training_modules (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    title           VARCHAR(255) NOT NULL,
    module_type     VARCHAR(50) NOT NULL,
    scorm_package_url TEXT,
    content_url     TEXT,
    duration_minutes INT,
    passing_score   NUMERIC(5,2),
    status          VARCHAR(20) NOT NULL DEFAULT 'draft',
    sort_order      INT NOT NULL DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_training_org ON training_modules(organisation_id);

CREATE TABLE certifications (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    validity_months INT,
    required_modules INT NOT NULL DEFAULT 1,
    passing_score   NUMERIC(5,2),
    graph_node_id   UUID REFERENCES graph_nodes(id),      -- certifications are graph nodes too
    status          VARCHAR(20) NOT NULL DEFAULT 'active',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_certs_org ON certifications(organisation_id);

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
    score           NUMERIC(5,2),
    time_spent_seconds INT,
    attempts        INT NOT NULL DEFAULT 0,
    scorm_data      JSONB,
    started_at      TIMESTAMPTZ,
    completed_at    TIMESTAMPTZ,
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

## Commissions (Relational)

```sql
CREATE TABLE commission_plans (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    program_id      UUID REFERENCES partner_programs(id),
    name            VARCHAR(255) NOT NULL,
    plan_type       VARCHAR(50) NOT NULL,
    base_rate       NUMERIC(8,4),
    currency        CHAR(3) NOT NULL DEFAULT 'USD',
    clawback_days   INT DEFAULT 90,
    is_recurring    BOOLEAN NOT NULL DEFAULT false,
    status          VARCHAR(20) NOT NULL DEFAULT 'active',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_comm_plans_org ON commission_plans(organisation_id);

CREATE TABLE commissions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    partner_id      UUID NOT NULL REFERENCES partners(id),
    deal_id         UUID REFERENCES deals(id),
    plan_id         UUID NOT NULL REFERENCES commission_plans(id),
    entry_type      VARCHAR(30) NOT NULL,
    amount          NUMERIC(15,2) NOT NULL,
    currency        CHAR(3) NOT NULL DEFAULT 'USD',
    status          VARCHAR(30) NOT NULL DEFAULT 'pending',
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

## Integrations & Audit (Relational)

```sql
CREATE TABLE integrations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    provider        VARCHAR(50) NOT NULL,
    category        VARCHAR(30) NOT NULL,
    is_enabled      BOOLEAN NOT NULL DEFAULT true,
    config          JSONB NOT NULL DEFAULT '{}',
    credentials     JSONB NOT NULL DEFAULT '{}',
    last_sync_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE UNIQUE INDEX idx_integrations_org_provider ON integrations(organisation_id, provider);

CREATE TABLE audit_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id),
    actor_id        UUID REFERENCES users(id),
    actor_type      VARCHAR(20) NOT NULL DEFAULT 'user',
    action          VARCHAR(100) NOT NULL,
    entity_type     VARCHAR(50) NOT NULL,
    entity_id       UUID NOT NULL,
    changes         JSONB,
    ip_address      INET,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_audit_org ON audit_log(organisation_id, created_at DESC);
CREATE INDEX idx_audit_entity ON audit_log(entity_type, entity_id);
```

---

## Example Graph Queries

### Find all partners that share customers with a given partner

```sql
-- Partners that have referred or sold to the same customers as Partner X
WITH partner_customers AS (
    SELECT DISTINCT e.target_node_id AS customer_node_id
    FROM graph_edges e
    JOIN graph_nodes n ON n.id = e.source_node_id
    WHERE n.node_type = 'partner'
      AND n.entity_id = :partner_id                      -- Partner X
      AND e.edge_type IN ('PARTNER_REFERS_CUSTOMER', 'PARTNER_REGISTERED_DEAL')
      AND e.valid_to IS NULL
)
SELECT
    p_node.entity_id AS overlapping_partner_id,
    p_node.label AS partner_name,
    COUNT(DISTINCT pc.customer_node_id) AS shared_customers,
    array_agg(DISTINCT c_node.label) AS shared_customer_names
FROM partner_customers pc
JOIN graph_edges e2 ON e2.target_node_id = pc.customer_node_id
    AND e2.edge_type IN ('PARTNER_REFERS_CUSTOMER', 'PARTNER_REGISTERED_DEAL')
    AND e2.valid_to IS NULL
JOIN graph_nodes p_node ON p_node.id = e2.source_node_id
    AND p_node.node_type = 'partner'
    AND p_node.entity_id != :partner_id
JOIN graph_nodes c_node ON c_node.id = pc.customer_node_id
GROUP BY p_node.entity_id, p_node.label
ORDER BY shared_customers DESC;
```

### Detect deal registration conflicts (same customer, multiple partners)

```sql
-- Customers with active deal registrations from more than one partner
SELECT
    c_node.entity_id AS customer_id,
    c_node.label AS customer_name,
    array_agg(DISTINCT p_node.label) AS competing_partners,
    array_agg(DISTINCT d_node.properties->>'deal_number') AS deal_numbers
FROM graph_edges e
JOIN graph_nodes d_node ON d_node.id = e.source_node_id
    AND d_node.node_type = 'deal'
    AND (d_node.properties->>'stage') IN ('submitted', 'approved')
JOIN graph_nodes c_node ON c_node.id = e.target_node_id
    AND c_node.node_type = 'customer'
JOIN graph_edges pe ON pe.target_node_id = d_node.id
    AND pe.edge_type = 'PARTNER_REGISTERED_DEAL'
JOIN graph_nodes p_node ON p_node.id = pe.source_node_id
    AND p_node.node_type = 'partner'
WHERE e.edge_type = 'DEAL_FOR_CUSTOMER'
  AND e.organisation_id = :org_id
  AND e.valid_to IS NULL
GROUP BY c_node.entity_id, c_node.label
HAVING COUNT(DISTINCT p_node.entity_id) > 1;
```

### Trace the distributor hierarchy (multi-tier channel)

```sql
-- Recursive traversal: find all sub-partners under a distributor
WITH RECURSIVE partner_tree AS (
    -- Base: the top-level distributor
    SELECT
        n.entity_id AS partner_id,
        n.label AS partner_name,
        0 AS depth,
        ARRAY[n.label] AS path
    FROM graph_nodes n
    WHERE n.node_type = 'partner'
      AND n.entity_id = :distributor_id

    UNION ALL

    -- Recursive: children linked by PARTNER_CHILD_OF edges
    SELECT
        child.entity_id,
        child.label,
        pt.depth + 1,
        pt.path || child.label
    FROM partner_tree pt
    JOIN graph_nodes parent ON parent.entity_id = pt.partner_id
        AND parent.node_type = 'partner'
    JOIN graph_edges e ON e.target_node_id = parent.id
        AND e.edge_type = 'PARTNER_CHILD_OF'
        AND e.valid_to IS NULL
    JOIN graph_nodes child ON child.id = e.source_node_id
        AND child.node_type = 'partner'
    WHERE pt.depth < 10  -- safety limit
)
SELECT partner_id, partner_name, depth, path
FROM partner_tree
ORDER BY depth, partner_name;
```

### Find co-sell opportunities (partners with complementary customer bases)

```sql
-- Partners whose customers are in industries where Partner X has no presence
WITH partner_x_industries AS (
    SELECT DISTINCT c_node.properties->>'industry' AS industry
    FROM graph_edges e
    JOIN graph_nodes p_node ON p_node.id = e.source_node_id
        AND p_node.node_type = 'partner'
        AND p_node.entity_id = :partner_id
    JOIN graph_nodes c_node ON c_node.id = e.target_node_id
        AND c_node.node_type = 'customer'
    WHERE e.edge_type = 'PARTNER_REFERS_CUSTOMER'
      AND e.valid_to IS NULL
)
SELECT
    p_node.entity_id AS partner_id,
    p_node.label AS partner_name,
    c_node.properties->>'industry' AS industry,
    COUNT(*) AS customers_in_industry
FROM graph_edges e
JOIN graph_nodes p_node ON p_node.id = e.source_node_id
    AND p_node.node_type = 'partner'
    AND p_node.entity_id != :partner_id
JOIN graph_nodes c_node ON c_node.id = e.target_node_id
    AND c_node.node_type = 'customer'
WHERE e.edge_type = 'PARTNER_REFERS_CUSTOMER'
  AND e.valid_to IS NULL
  AND c_node.properties->>'industry' NOT IN (SELECT industry FROM partner_x_industries)
  AND e.organisation_id = :org_id
GROUP BY p_node.entity_id, p_node.label, c_node.properties->>'industry'
ORDER BY customers_in_industry DESC;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Organisation & Users | 3 | organisations, users, memberships |
| Graph Layer | 2 + 1 MV | graph_nodes, graph_edges, plus undirected materialised view |
| Partner Management | 5 | programs, tiers, partners, contacts, customers |
| Deal Registration | 1 | deals |
| MDF Management | 3 | budgets, requests, claims |
| Training & Certification | 5 | modules, certifications, junction, progress, awards |
| Commissions | 2 | plans, commissions |
| Integrations & Audit | 2 | integrations, audit_log |
| **Total** | **23 + 1 MV** | |

---

## Key Design Decisions

1. **graph_node_id on relational tables** — each relational entity that participates in the graph (partners, contacts, deals, customers, certifications) stores a back-reference to its corresponding `graph_nodes` row. This enables the application to navigate from relational to graph context without a separate lookup.

2. **Separate customers table** — unlike the normalised model (Suggestion 1) which stores customer info only on deal records, this model extracts customers into their own table and graph nodes. This is essential for customer overlap analysis: the graph layer needs a stable customer identity to connect multiple deals and partners.

3. **Temporal edges with valid_from/valid_to** — graph edges are temporal: a partner contact who moves to a different company gets a `valid_to` timestamp on the old CONTACT_WORKS_AT edge and a new edge with a fresh `valid_from`. This captures relationship history without deleting data, supporting queries like "who worked at this partner 6 months ago?"

4. **Weighted edges for algorithmic analysis** — the `weight` column on edges supports graph algorithms (PageRank for partner influence, shortest path for referral chains, community detection for ecosystem clusters). Default weight is 1.0; the application adjusts weights based on revenue, deal count, or recency.

5. **Materialised view for undirected traversal** — many graph queries need bidirectional traversal (e.g., "find all entities connected to X regardless of edge direction"). The `graph_edges_undirected` materialised view doubles the edge set to enable single-index lookups in either direction. It is refreshed periodically.

6. **Graph layer is a projection, not the source of truth** — relational tables own the data; graph nodes and edges are derived. If the graph becomes inconsistent, it can be rebuilt from relational data. This avoids the dual-write consistency problem at the cost of a rebuild mechanism.

7. **Edge types as strings, not enums** — using VARCHAR for `edge_type` rather than PostgreSQL ENUM allows new relationship types to be added without migrations. The application layer validates edge types.

8. **Customer identity resolution** — the `customers` table with `crm_account_id` serves as the identity resolution point. When a deal is registered with a customer email, the application matches or creates a customer record, enabling graph edges to accumulate on a stable customer node rather than fragmenting across deal records.

9. **PostgreSQL over Neo4j** — implementing the graph in PostgreSQL (rather than a dedicated graph DB) keeps the operational stack simpler. Recursive CTEs handle most PRM graph queries (typically 2-4 hops). If traversal depth or query volume outgrows PostgreSQL, the graph layer can be projected to Neo4j without changing the relational core.

10. **Certifications as graph nodes** — certifications participate in the graph via PARTNER_HAS_CERTIFICATION edges, enabling queries like "which partners are certified for Product X but have not registered any deals for it?" This connects enablement data to commercial activity through the graph.
