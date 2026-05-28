# Partner Relationship Management -- Development Plan

> Project: 049-partner-relationship-management
> Created: 2026-05-25
> Based on: research.md, features.md, standards.md, README.md, data-model-suggestion-1 through 4

---

## Table of Contents

1. [Technology Decisions](#technology-decisions)
2. [Project Structure](#project-structure)
3. [Data Model Selection](#data-model-selection)
4. [Phase Dependency Graph](#phase-dependency-graph)
5. [Phase 1: Foundation & Authentication](#phase-1-foundation--authentication)
6. [Phase 2: Partner Management Core](#phase-2-partner-management-core)
7. [Phase 3: Deal Registration Engine](#phase-3-deal-registration-engine)
8. [Phase 4: MDF Management](#phase-4-mdf-management)
9. [Phase 5: Training & Certification](#phase-5-training--certification)
10. [Phase 6: Partner Portal](#phase-6-partner-portal)
11. [Phase 7: CRM Integration](#phase-7-crm-integration)
12. [Phase 8: Notifications & Activity Feed](#phase-8-notifications--activity-feed)
13. [Phase 9: Commissions & Payouts](#phase-9-commissions--payouts)
14. [Phase 10: Co-Sell Integration](#phase-10-co-sell-integration)
15. [Phase 11: AI-Powered Intelligence](#phase-11-ai-powered-intelligence)
16. [Phase 12: MCP Server & API Ecosystem](#phase-12-mcp-server--api-ecosystem)
17. [Definition of Done](#definition-of-done)

---

## Technology Decisions

### Backend

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Language | **TypeScript (Node.js 22+)** | Target market is SaaS companies; TypeScript dominates their stack. Shared language with frontend reduces hiring friction. Strong ecosystem for API development (Hono, Drizzle, Zod). |
| Runtime | **Node.js 22 LTS** | Native TypeScript support via `--experimental-strip-types` in Node 22. LTS channel provides stability for enterprise customers. |
| API Framework | **Hono** | Lightweight, edge-compatible, supports OpenAPI 3.1 spec generation via `@hono/zod-openapi`. Faster cold starts than Express/Fastify for serverless deployments. |
| ORM / Query Builder | **Drizzle ORM** | Type-safe SQL queries, zero-overhead abstractions, first-class PostgreSQL support. Generates migration SQL from schema definitions. Avoids the runtime complexity of Prisma's query engine. |
| Validation | **Zod** | Runtime schema validation that generates TypeScript types. Powers OpenAPI spec generation via `@hono/zod-openapi`. Validates JSONB custom fields against stored JSON Schema definitions. |
| Database | **PostgreSQL 16** | Required by the data model (JSONB, GIN indexes, partial indexes, Row-Level Security). Strong multi-tenancy via RLS. The four data model suggestions all target PostgreSQL. |
| Cache / Queues | **Redis 7 (Valkey)** | Session caching, background job queues (BullMQ), real-time notification pubsub, rate limiting. Valkey is the open-source Redis fork. |
| Background Jobs | **BullMQ** | Reliable job processing for CRM sync, commission calculation, notification dispatch, webhook delivery. Built on Redis/Valkey. Supports retry, priority queues, and cron schedules. |
| File Storage | **S3-compatible (AWS S3 / MinIO)** | MDF proof-of-performance documents, SCORM packages, content assets, partner logos. MinIO for local development and self-hosted deployments. |
| Search | **PostgreSQL full-text search (initial), Meilisearch (v1.1)** | PostgreSQL `tsvector` is sufficient for partner and deal search at MVP scale. Meilisearch for content library full-text search when content volume warrants it. |

### Frontend

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Framework | **Next.js 15 (App Router)** | React Server Components for fast initial loads on the partner portal. Server Actions for form submissions. Middleware for tenant routing. Dominant framework in the SaaS ecosystem this product targets. |
| UI Library | **shadcn/ui + Tailwind CSS 4** | Copy-paste component library avoids dependency lock-in. Tailwind 4 provides design tokens via CSS custom properties. Consistent with modern SaaS design patterns. |
| State Management | **TanStack Query v5** | Server-state synchronisation for API data. Handles cache invalidation, optimistic updates, and background refetching for deal pipeline and partner list views. |
| Forms | **React Hook Form + Zod resolvers** | Performant form management with Zod schema validation shared between client and server. Essential for deal registration and MDF claim forms with dynamic custom fields. |
| Data Tables | **TanStack Table v8** | Headless table library for partner lists, deal pipelines, commission reports. Supports server-side sorting, filtering, and pagination. |
| Charts | **Recharts** | Declarative charting for partner dashboards, deal funnel visualisation, MDF utilisation graphs, commission trend charts. |

### Infrastructure & DevOps

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Containerisation | **Docker + Docker Compose** | Local development parity. Multi-service orchestration (API, PostgreSQL, Redis, MinIO). |
| CI/CD | **GitHub Actions** | Standard for open-source projects. Matrix testing across Node versions. Auto-deploy to staging on merge. |
| Deployment | **Docker on any cloud (initial), Kubernetes (scale)** | Cloud-agnostic deployment avoids vendor lock-in. Kubernetes via Helm chart when multi-region or >100 tenants. |
| Monitoring | **OpenTelemetry + Grafana stack** | Vendor-neutral observability. Traces, metrics, and logs via OTel SDK. Grafana Cloud or self-hosted Grafana/Loki/Tempo. |
| Auth Provider | **Custom (Lucia Auth) + SAML/OIDC federation** | Lucia provides session management without external dependency. SAML 2.0/OIDC federation via `saml2-js` and `openid-client` for enterprise partner SSO. Avoids Auth0/Clerk vendor lock-in and per-user costs that scale poorly with partner portal users. |

### Testing

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Unit / Integration | **Vitest** | Fast, native ESM support, compatible with Vite. Jest-compatible API for easy adoption. |
| API Testing | **Supertest + Vitest** | HTTP-level integration tests against the Hono API. Validates OpenAPI response schemas. |
| E2E | **Playwright** | Cross-browser E2E testing for the partner portal. Visual regression for portal branding. |
| Database Testing | **Testcontainers** | Ephemeral PostgreSQL containers per test suite. Real database behaviour without mocking. |
| Load Testing | **k6** | JavaScript-based load testing for API endpoints. Validate deal registration throughput under concurrent partner submissions. |

---

## Project Structure

```
partner-relationship-management/
├── apps/
│   ├── api/                          # Hono API server
│   │   ├── src/
│   │   │   ├── routes/               # Route modules by domain
│   │   │   │   ├── auth/
│   │   │   │   ├── partners/
│   │   │   │   ├── deals/
│   │   │   │   ├── mdf/
│   │   │   │   ├── training/
│   │   │   │   ├── commissions/
│   │   │   │   ├── content/
│   │   │   │   ├── integrations/
│   │   │   │   └── admin/
│   │   │   ├── middleware/            # Auth, tenant isolation, rate limiting
│   │   │   ├── services/             # Business logic layer
│   │   │   ├── jobs/                 # BullMQ job handlers
│   │   │   ├── webhooks/             # Outbound webhook delivery
│   │   │   ├── mcp/                  # MCP server implementation
│   │   │   └── index.ts
│   │   ├── drizzle/                  # Migrations
│   │   ├── tests/
│   │   └── package.json
│   │
│   └── web/                          # Next.js frontend
│       ├── src/
│       │   ├── app/
│       │   │   ├── (vendor)/         # Vendor admin views
│       │   │   │   ├── partners/
│       │   │   │   ├── deals/
│       │   │   │   ├── mdf/
│       │   │   │   ├── training/
│       │   │   │   ├── commissions/
│       │   │   │   ├── content/
│       │   │   │   ├── integrations/
│       │   │   │   └── settings/
│       │   │   ├── (portal)/         # Partner-facing portal
│       │   │   │   ├── dashboard/
│       │   │   │   ├── deals/
│       │   │   │   ├── mdf/
│       │   │   │   ├── training/
│       │   │   │   ├── content/
│       │   │   │   └── profile/
│       │   │   ├── (auth)/           # Login, SSO, signup
│       │   │   └── api/              # Next.js API routes (BFF)
│       │   ├── components/
│       │   │   ├── ui/               # shadcn/ui components
│       │   │   ├── partners/
│       │   │   ├── deals/
│       │   │   ├── mdf/
│       │   │   └── shared/
│       │   ├── hooks/
│       │   ├── lib/
│       │   └── styles/
│       ├── tests/
│       └── package.json
│
├── packages/
│   ├── db/                           # Drizzle schema + migrations (shared)
│   │   ├── src/
│   │   │   ├── schema/               # Table definitions by domain
│   │   │   │   ├── organisations.ts
│   │   │   │   ├── partners.ts
│   │   │   │   ├── deals.ts
│   │   │   │   ├── mdf.ts
│   │   │   │   ├── training.ts
│   │   │   │   ├── commissions.ts
│   │   │   │   ├── content.ts
│   │   │   │   ├── integrations.ts
│   │   │   │   ├── notifications.ts
│   │   │   │   └── audit.ts
│   │   │   ├── migrations/
│   │   │   └── index.ts
│   │   └── package.json
│   │
│   ├── shared/                       # Shared types, constants, validation schemas
│   │   ├── src/
│   │   │   ├── schemas/              # Zod schemas (shared client + server)
│   │   │   ├── types/
│   │   │   └── constants/
│   │   └── package.json
│   │
│   └── email/                        # Email templates (React Email)
│       ├── src/
│       │   └── templates/
│       └── package.json
│
├── docker-compose.yml
├── turbo.json                        # Turborepo configuration
├── pnpm-workspace.yaml
└── README.md
```

**Monorepo tooling:** pnpm workspaces + Turborepo for build orchestration. The `packages/db` package is shared between `apps/api` and `apps/web` to ensure schema types are consistent.

---

## Data Model Selection

**Selected: Hybrid approach combining Data Model Suggestion 1 (Entity-Centric Normalized) and Data Model Suggestion 3 (Hybrid Relational + JSONB)**

### Rationale

After evaluating all four data model suggestions against the project's positioning (lightweight, SaaS-native, fast-to-deploy PRM for 10-100 partner programs):

- **Suggestion 1 (Normalized)** provides the strongest data integrity and reporting capability, but 40+ tables add migration complexity that conflicts with the "fast deployment" differentiator.
- **Suggestion 2 (Event-Sourced/CQRS)** is architecturally powerful for audit trails, but introduces operational complexity (projection management, eventual consistency) that is premature for an MVP targeting mid-market SaaS companies.
- **Suggestion 3 (Hybrid Relational + JSONB)** has the right balance: relational backbone for core entities with JSONB flexibility for program-specific and jurisdiction-specific fields. 17 tables is manageable.
- **Suggestion 4 (Graph-Relational)** adds a compelling relationship-intelligence layer but the dual-write synchronisation overhead is unjustified until the partner ecosystem mapping feature is validated by customers.

**The selected hybrid approach:**

1. Uses Suggestion 1's normalized structure for the core entities that are common across all deployments: organisations, users, memberships, partners, partner_contacts, deals, deal_activities, mdf_budgets, mdf_requests, mdf_claims, training_modules, certifications, training_progress, certification_awards, commissions, content_assets, and audit_log.
2. Adopts Suggestion 3's `custom_fields JSONB` pattern on partners, deals, and MDF requests to support jurisdiction-specific and program-specific fields without schema migrations.
3. Adopts Suggestion 3's `custom_field_schemas` meta-definition table so organisations can define their own fields via configuration.
4. Adopts Suggestion 3's unified `integrations` table (instead of separate tables per integration category) to reduce table count and simplify adding new integration providers.
5. Adopts Suggestion 1's separate `mdf_claims` table (not embedded JSONB as in Suggestion 3) because MDF claims have their own approval workflow, document attachments, and audit requirements that justify a first-class entity.
6. Defers Suggestion 2's event sourcing to Phase 11 (AI Intelligence) where historical event patterns are needed for partner health scoring -- at that point, an append-only `partner_events` table is added alongside the relational tables, not replacing them.
7. Defers Suggestion 4's graph layer to a future release, after customer demand for ecosystem mapping is validated.

**Target table count at MVP: ~25 tables** (Phase 1-6 complete).

---

## Phase Dependency Graph

```
Phase 1: Foundation & Auth
    |
    v
Phase 2: Partner Management --------+
    |                                |
    v                                v
Phase 3: Deal Registration     Phase 5: Training & Cert
    |                                |
    v                                |
Phase 4: MDF Management             |
    |                                |
    +--------+-----------------------+
             |
             v
Phase 6: Partner Portal
    |
    +--------+--------+--------+
    |        |        |        |
    v        v        v        v
Phase 7  Phase 8  Phase 9  Phase 10
CRM Int  Notif.   Commiss. Co-Sell
    |        |        |        |
    +--------+--------+--------+
             |
             v
Phase 11: AI Intelligence
    |
    v
Phase 12: MCP Server & API Ecosystem
```

**Legend:**
- Phases 7, 8, 9, 10 can be developed in parallel after Phase 6.
- Phase 11 requires Phases 7 and 9 (CRM data and commission data feed the AI models).
- Phase 12 depends on Phase 11 (MCP server exposes AI capabilities to external agents).

---

## Phase 1: Foundation & Authentication

**Goal:** Bootable application with multi-tenant auth, organisation management, and CI/CD pipeline.

**Duration:** 3-4 weeks

### Task 1.1: Monorepo Scaffold & Tooling

**What:** Initialize the pnpm + Turborepo monorepo with `apps/api`, `apps/web`, `packages/db`, and `packages/shared`. Configure TypeScript, ESLint, Prettier. Set up Docker Compose for PostgreSQL 16, Redis 7, and MinIO.

**Design:**
```typescript
// turbo.json
{
  "$schema": "https://turbo.build/schema.json",
  "tasks": {
    "build": { "dependsOn": ["^build"], "outputs": ["dist/**", ".next/**"] },
    "dev": { "cache": false, "persistent": true },
    "test": { "dependsOn": ["^build"] },
    "db:migrate": { "cache": false },
    "lint": { "dependsOn": ["^build"] }
  }
}

// docker-compose.yml (services)
// postgres:16-alpine on port 5432
// valkey/valkey:7-alpine on port 6379
// minio/minio on port 9000
```

**Testing:**
- `pnpm install` succeeds without errors across all workspaces
- `pnpm turbo build` compiles all packages and apps successfully
- `docker compose up -d` starts PostgreSQL, Valkey, MinIO; all services respond to health checks
- `pnpm turbo test` runs an empty test suite in each package and passes

---

### Task 1.2: Database Schema -- Organisations, Users, Memberships

**What:** Define the Drizzle schema for `organisations`, `users`, and `memberships` tables. Generate and run the initial migration. Set up Row-Level Security (RLS) policies for tenant isolation.

**Design:**
```typescript
// packages/db/src/schema/organisations.ts
import { pgTable, uuid, varchar, text, jsonb, timestamp, boolean, uniqueIndex, index } from 'drizzle-orm/pg-core';

export const organisations = pgTable('organisations', {
  id: uuid('id').primaryKey().defaultRandom(),
  name: varchar('name', { length: 255 }).notNull(),
  slug: varchar('slug', { length: 100 }).notNull().unique(),
  domain: varchar('domain', { length: 255 }),
  logoUrl: text('logo_url'),
  billingEmail: varchar('billing_email', { length: 255 }),
  plan: varchar('plan', { length: 50 }).notNull().default('free'),
  settings: jsonb('settings').notNull().default({}),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
});

export const users = pgTable('users', {
  id: uuid('id').primaryKey().defaultRandom(),
  email: varchar('email', { length: 255 }).notNull().unique(),
  fullName: varchar('full_name', { length: 255 }).notNull(),
  avatarUrl: text('avatar_url'),
  passwordHash: text('password_hash'),
  emailVerified: boolean('email_verified').notNull().default(false),
  lastLoginAt: timestamp('last_login_at', { withTimezone: true }),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
});

export const memberships = pgTable('memberships', {
  id: uuid('id').primaryKey().defaultRandom(),
  userId: uuid('user_id').notNull().references(() => users.id, { onDelete: 'cascade' }),
  organisationId: uuid('organisation_id').notNull().references(() => organisations.id, { onDelete: 'cascade' }),
  role: varchar('role', { length: 50 }).notNull().default('member'),
  status: varchar('status', { length: 20 }).notNull().default('active'),
  invitedAt: timestamp('invited_at', { withTimezone: true }),
  joinedAt: timestamp('joined_at', { withTimezone: true }),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
}, (table) => ({
  uniqueUserOrg: uniqueIndex('idx_memberships_unique').on(table.userId, table.organisationId),
  orgIdx: index('idx_memberships_org').on(table.organisationId),
  userIdx: index('idx_memberships_user').on(table.userId),
}));

// RLS policy (applied via raw SQL migration):
// CREATE POLICY tenant_isolation ON memberships
//   USING (organisation_id = current_setting('app.current_org_id')::uuid);
```

**Testing:**
- Migration runs successfully against a clean database: `pnpm db:migrate` exits 0
- Migration is idempotent: running it twice does not error
- Drizzle can insert and query an organisation, a user, and a membership
- Unique constraint on `(user_id, organisation_id)` prevents duplicate memberships
- Unique constraint on `organisations.slug` prevents duplicate slugs
- RLS policy blocks queries when `app.current_org_id` is not set or mismatches

---

### Task 1.3: Authentication -- Email/Password + Session Management

**What:** Implement email/password registration, login, and session management using Lucia Auth v3. Sessions are stored in PostgreSQL. Middleware extracts and validates the session on every request, injecting the authenticated user and organisation into the Hono context.

**Design:**
```typescript
// apps/api/src/middleware/auth.ts
import { createMiddleware } from 'hono/factory';
import { lucia } from '../lib/lucia';
import type { Session, User } from 'lucia';

export const authMiddleware = createMiddleware(async (c, next) => {
  const sessionId = lucia.readSessionCookie(c.req.header('cookie') ?? '');
  if (!sessionId) {
    c.set('user', null);
    c.set('session', null);
    return next();
  }
  const { session, user } = await lucia.validateSession(sessionId);
  if (session?.fresh) {
    c.header('Set-Cookie', lucia.createSessionCookie(session.id).serialize(), { append: true });
  }
  if (!session) {
    c.header('Set-Cookie', lucia.createBlankSessionCookie().serialize(), { append: true });
  }
  c.set('user', user);
  c.set('session', session);
  return next();
});

// apps/api/src/routes/auth/register.ts
app.post('/auth/register', zValidator('json', registerSchema), async (c) => {
  const { email, password, fullName, organisationName } = c.req.valid('json');
  // 1. Hash password with Argon2id
  // 2. Create user
  // 3. Create organisation with auto-generated slug
  // 4. Create membership (role: 'owner')
  // 5. Create session
  // 6. Return session cookie + user data
});

// apps/api/src/routes/auth/login.ts
app.post('/auth/login', zValidator('json', loginSchema), async (c) => {
  const { email, password } = c.req.valid('json');
  // 1. Look up user by email
  // 2. Verify password with Argon2id
  // 3. Create session
  // 4. Return session cookie + user data
});
```

**Testing:**
- Register with valid email/password creates user, organisation, and membership; returns 201 with session cookie
- Register with duplicate email returns 409 Conflict
- Register with weak password (< 8 chars) returns 400 with validation error
- Login with correct credentials returns 200 with session cookie
- Login with incorrect password returns 401 Unauthorized
- Login with non-existent email returns 401 Unauthorized (no user enumeration)
- Authenticated request with valid session cookie returns 200 with user context
- Request with expired session cookie returns 401
- Request with no session cookie to protected endpoint returns 401
- Logout invalidates session; subsequent requests with the same cookie return 401

---

### Task 1.4: Organisation Management API

**What:** CRUD endpoints for organisations. Only owners/admins can update settings or invite members. Member invitation flow with email tokens.

**Design:**
```typescript
// apps/api/src/routes/admin/organisations.ts
app.get('/organisations/:orgId', requireAuth, requireRole(['owner', 'admin', 'manager', 'member']), async (c) => {
  // Return organisation details
});

app.patch('/organisations/:orgId', requireAuth, requireRole(['owner', 'admin']), zValidator('json', updateOrgSchema), async (c) => {
  // Update organisation name, domain, settings, branding
});

app.post('/organisations/:orgId/invitations', requireAuth, requireRole(['owner', 'admin']), zValidator('json', inviteSchema), async (c) => {
  // 1. Validate email not already a member
  // 2. Create invitation token (expires in 7 days)
  // 3. Create membership with status 'invited'
  // 4. Queue invitation email via BullMQ
  // 5. Return 201 with invitation details
});

app.post('/invitations/:token/accept', async (c) => {
  // 1. Validate token exists and is not expired
  // 2. Create user account if needed (or link existing)
  // 3. Update membership status to 'active'
  // 4. Create session
  // 5. Return 200 with session cookie
});
```

**Testing:**
- GET organisation returns 200 with correct data for authenticated member
- GET organisation returns 403 for user not in the organisation
- PATCH organisation succeeds for owner; returns 200 with updated fields
- PATCH organisation returns 403 for member role (insufficient permissions)
- POST invitation sends email and creates pending membership
- POST invitation with existing member email returns 409
- Accept invitation with valid token activates membership and creates session
- Accept invitation with expired token returns 410 Gone
- Accept invitation with already-used token returns 409

---

### Task 1.5: CI/CD Pipeline

**What:** GitHub Actions workflow for lint, typecheck, test, build. Database tests use Testcontainers. Auto-deploy to staging on merge to `main`.

**Design:**
```yaml
# .github/workflows/ci.yml
name: CI
on: [push, pull_request]
jobs:
  check:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16-alpine
        env:
          POSTGRES_DB: prm_test
          POSTGRES_USER: prm
          POSTGRES_PASSWORD: test
        ports: ['5432:5432']
      valkey:
        image: valkey/valkey:7-alpine
        ports: ['6379:6379']
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v4
      - uses: actions/setup-node@v4
        with: { node-version: 22 }
      - run: pnpm install --frozen-lockfile
      - run: pnpm turbo lint
      - run: pnpm turbo typecheck
      - run: pnpm turbo db:migrate
      - run: pnpm turbo test
      - run: pnpm turbo build
```

**Testing:**
- CI pipeline runs on PR creation and push to `main`
- Lint step catches ESLint violations and fails the build
- Typecheck step catches TypeScript errors and fails the build
- Database migration runs against the CI PostgreSQL service
- All test suites pass in CI environment
- Build step produces deployable artefacts

---

### Task 1.6: Audit Log Foundation

**What:** Implement the `audit_log` table and a reusable `auditLog()` helper function that records every mutation across the application. All subsequent phases use this helper.

**Design:**
```typescript
// packages/db/src/schema/audit.ts
export const auditLog = pgTable('audit_log', {
  id: uuid('id').primaryKey().defaultRandom(),
  organisationId: uuid('organisation_id').notNull().references(() => organisations.id),
  actorId: uuid('actor_id').references(() => users.id),
  actorType: varchar('actor_type', { length: 20 }).notNull().default('user'),
  action: varchar('action', { length: 100 }).notNull(),
  entityType: varchar('entity_type', { length: 50 }).notNull(),
  entityId: uuid('entity_id').notNull(),
  changes: jsonb('changes'),
  ipAddress: text('ip_address'),
  userAgent: text('user_agent'),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
});

// apps/api/src/services/audit.ts
export async function recordAudit(params: {
  db: DrizzleClient;
  organisationId: string;
  actorId: string;
  actorType?: 'user' | 'system' | 'integration';
  action: string;                        // e.g. 'partner.created', 'deal.approved'
  entityType: string;
  entityId: string;
  changes?: Record<string, { old: unknown; new: unknown }>;
  ipAddress?: string;
  userAgent?: string;
}) {
  await params.db.insert(auditLog).values(params);
}
```

**Testing:**
- `recordAudit()` inserts a row with all provided fields
- `changes` JSONB correctly stores old/new value diffs
- Audit log entries are queryable by organisation, entity type, entity ID, and date range
- Audit log entries are immutable (no UPDATE/DELETE operations succeed via application code)
- Index on `(organisation_id, created_at)` enables efficient time-range queries

---

## Phase 2: Partner Management Core

**Goal:** Full partner lifecycle management -- programs, tiers, partners, contacts, and custom fields.

**Duration:** 3-4 weeks

**Dependencies:** Phase 1 complete.

### Task 2.1: Partner Programs & Tiers Schema + API

**What:** Create the `partner_programs` and `partner_tiers` tables. CRUD API for programs and tiers. A program has a type (reseller, referral, affiliate, technology, MSP) and contains ordered tiers with qualification criteria.

**Design:**
```typescript
// packages/db/src/schema/partners.ts
export const partnerPrograms = pgTable('partner_programs', {
  id: uuid('id').primaryKey().defaultRandom(),
  organisationId: uuid('organisation_id').notNull().references(() => organisations.id, { onDelete: 'cascade' }),
  name: varchar('name', { length: 255 }).notNull(),
  description: text('description'),
  programType: varchar('program_type', { length: 50 }).notNull(),
  status: varchar('status', { length: 20 }).notNull().default('active'),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
});

export const partnerTiers = pgTable('partner_tiers', {
  id: uuid('id').primaryKey().defaultRandom(),
  programId: uuid('program_id').notNull().references(() => partnerPrograms.id, { onDelete: 'cascade' }),
  name: varchar('name', { length: 100 }).notNull(),
  rank: integer('rank').notNull(),
  minRevenue: numeric('min_revenue', { precision: 15, scale: 2 }),
  minDeals: integer('min_deals'),
  minCertifications: integer('min_certifications'),
  benefits: text('benefits'),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
});

// apps/api/src/routes/partners/programs.ts
// POST   /programs                 -- create program
// GET    /programs                 -- list programs for org
// GET    /programs/:id             -- get program with tiers
// PATCH  /programs/:id             -- update program
// DELETE /programs/:id             -- archive program (soft delete via status)
// POST   /programs/:id/tiers       -- add tier to program
// PATCH  /programs/:id/tiers/:tid  -- update tier
// DELETE /programs/:id/tiers/:tid  -- remove tier
```

**Testing:**
- Create program with valid data returns 201; program is queryable
- Create program with invalid `program_type` returns 400
- List programs returns only programs belonging to the authenticated org (tenant isolation)
- Create tier with rank 1 (Platinum) and rank 2 (Gold) on a program; listing tiers returns them in rank order
- Deleting a program with active partners returns 409 (cannot archive program with dependents)
- Deleting a program with no partners sets status to 'archived'
- Tier qualification thresholds (min_revenue, min_deals) are validated as non-negative

---

### Task 2.2: Partners CRUD with Custom Fields

**What:** Create the `partners` table with `custom_fields` JSONB column. Create `custom_field_schemas` meta-definition table. CRUD API for partners. Partners belong to a program and optionally a tier. Support filtering, sorting, and pagination.

**Design:**
```typescript
// packages/db/src/schema/partners.ts (continued)
export const partners = pgTable('partners', {
  id: uuid('id').primaryKey().defaultRandom(),
  organisationId: uuid('organisation_id').notNull().references(() => organisations.id, { onDelete: 'cascade' }),
  programId: uuid('program_id').notNull().references(() => partnerPrograms.id),
  tierId: uuid('tier_id').references(() => partnerTiers.id),
  companyName: varchar('company_name', { length: 255 }).notNull(),
  legalName: varchar('legal_name', { length: 255 }),
  website: text('website'),
  logoUrl: text('logo_url'),
  primaryContactName: varchar('primary_contact_name', { length: 255 }),
  primaryContactEmail: varchar('primary_contact_email', { length: 255 }),
  countryCode: char('country_code', { length: 2 }),
  status: varchar('status', { length: 30 }).notNull().default('prospect'),
  onboardedAt: timestamp('onboarded_at', { withTimezone: true }),
  partnerSince: date('partner_since'),
  customFields: jsonb('custom_fields').notNull().default({}),
  notes: text('notes'),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
});

// Custom field validation middleware:
// 1. Load custom_field_schemas for entity_type='partner' and current org
// 2. Validate incoming custom_fields JSONB against those schemas using Zod
// 3. Reject if required custom fields are missing or types mismatch

// apps/api/src/routes/partners/partners.ts
// POST   /partners              -- create partner
// GET    /partners              -- list with filters (status, program, tier, country, custom field values)
// GET    /partners/:id          -- get partner detail
// PATCH  /partners/:id          -- update partner
// PATCH  /partners/:id/status   -- change lifecycle status (prospect -> onboarding -> active -> inactive -> churned)
// PATCH  /partners/:id/tier     -- change partner tier (manual or auto-qualification)
```

**Testing:**
- Create partner with required fields returns 201 with generated UUID
- Create partner with custom_fields matching org's custom_field_schemas succeeds
- Create partner with custom_fields violating a required schema returns 400 with field-level errors
- List partners with `?status=active` returns only active partners
- List partners with `?country=US` returns only US-based partners
- List partners with `?program_id=...` returns only partners in that program
- List partners with pagination (`?page=2&limit=25`) returns correct page with total count
- Update partner status from 'prospect' to 'onboarding' sets `onboarded_at` timestamp
- Update partner status to 'active' sets `partner_since` date
- Invalid status transition (e.g. 'churned' to 'prospect') returns 400
- Tenant isolation: listing partners only returns partners from the authenticated org
- GIN index on custom_fields enables queries like `WHERE custom_fields @> '{"vertical": "financial_services"}'`

---

### Task 2.3: Partner Contacts

**What:** Create the `partner_contacts` table. CRUD API for contacts within a partner. A contact can optionally be linked to a user account for portal access.

**Design:**
```typescript
// packages/db/src/schema/partners.ts (continued)
export const partnerContacts = pgTable('partner_contacts', {
  id: uuid('id').primaryKey().defaultRandom(),
  partnerId: uuid('partner_id').notNull().references(() => partners.id, { onDelete: 'cascade' }),
  fullName: varchar('full_name', { length: 255 }).notNull(),
  email: varchar('email', { length: 255 }).notNull(),
  phone: varchar('phone', { length: 50 }),
  jobTitle: varchar('job_title', { length: 255 }),
  role: varchar('role', { length: 50 }),          // primary, technical, billing, marketing
  isPortalUser: boolean('is_portal_user').notNull().default(false),
  userId: uuid('user_id').references(() => users.id),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
});

// apps/api/src/routes/partners/contacts.ts
// POST   /partners/:partnerId/contacts       -- add contact
// GET    /partners/:partnerId/contacts       -- list contacts for partner
// PATCH  /partners/:partnerId/contacts/:id   -- update contact
// DELETE /partners/:partnerId/contacts/:id   -- remove contact
// POST   /partners/:partnerId/contacts/:id/invite  -- invite contact as portal user
```

**Testing:**
- Add contact to partner returns 201
- List contacts for a partner returns only that partner's contacts
- Contact with `role: 'primary'` is flagged appropriately
- Inviting a contact as portal user creates a user account and sends invitation email
- Inviting a contact who is already a portal user returns 409
- Deleting a contact who is a portal user also deactivates the linked user's portal access
- Contact email is validated as a valid email format

---

### Task 2.4: Custom Field Schema Management

**What:** API for organisation admins to define custom fields for partners, deals, and MDF requests. Supports text, number, date, boolean, select, and multi_select field types with optional validation rules.

**Design:**
```typescript
// apps/api/src/routes/admin/custom-fields.ts
// POST   /admin/custom-fields              -- create field definition
// GET    /admin/custom-fields?entity_type=partner  -- list field definitions
// PATCH  /admin/custom-fields/:id          -- update field definition
// DELETE /admin/custom-fields/:id          -- remove field definition

// Zod schema for custom field definition:
const customFieldSchemaInput = z.object({
  entityType: z.enum(['partner', 'deal', 'mdf_request', 'partner_contact']),
  fieldKey: z.string().regex(/^[a-z][a-z0-9_]{1,98}$/),   // snake_case, 2-99 chars
  fieldLabel: z.string().min(1).max(255),
  fieldType: z.enum(['text', 'number', 'date', 'boolean', 'select', 'multi_select', 'url']),
  isRequired: z.boolean().default(false),
  options: z.array(z.string()).optional(),                  // for select/multi_select
  validation: z.record(z.unknown()).optional(),             // JSON Schema fragment
  sortOrder: z.number().int().min(0).default(0),
  visibleTo: z.array(z.string()).default(['all']),          // tier/program visibility
});
```

**Testing:**
- Create custom field with `fieldType: 'select'` and `options: ['A', 'B', 'C']` succeeds
- Create custom field with `fieldType: 'select'` but no `options` returns 400
- Create custom field with duplicate `(entityType, fieldKey)` for same org returns 409
- `fieldKey` must be valid snake_case: 'my_field' passes, 'My Field' fails
- After creating a required custom field for 'partner', creating a partner without that field returns 400
- Deleting a custom field definition does not delete existing data in `custom_fields` JSONB columns
- Custom field definitions are scoped to the organisation (tenant isolation)

---

### Task 2.5: Partner Management UI (Vendor Admin)

**What:** Next.js pages for the vendor admin to manage partner programs, tiers, partners, and contacts. Includes partner list with filters, partner detail page, and partner onboarding workflow.

**Design:**
```
(vendor)/partners/page.tsx          -- Partner list with DataTable, filters, search
(vendor)/partners/[id]/page.tsx     -- Partner detail with tabs: Overview, Contacts, Deals, MDF, Training
(vendor)/partners/new/page.tsx      -- Create partner form
(vendor)/programs/page.tsx          -- Program list and management
(vendor)/programs/[id]/page.tsx     -- Program detail with tiers configuration
(vendor)/settings/custom-fields/    -- Custom field schema management
```

**Testing:**
- Partner list page loads and displays partners with correct columns (company, program, tier, status, country)
- Filters for status, program, tier, and country correctly narrow the list
- Search by company name returns matching partners
- Pagination controls work (next, previous, page size selector)
- Create partner form validates required fields before submission
- Partner detail page shows all tabs and correct data in each
- Custom field configuration page allows adding/editing/removing field definitions
- Playwright E2E: complete partner onboarding workflow from prospect to active

---

## Phase 3: Deal Registration Engine

**Goal:** Full deal registration workflow with duplicate detection, approval routing, and deal pipeline management.

**Duration:** 3-4 weeks

**Dependencies:** Phase 2 complete (partners exist to register deals).

### Task 3.1: Deals Schema & Duplicate Detection

**What:** Create the `deals` and `deal_activities` tables. Implement duplicate detection logic based on customer email + customer company within an organisation, excluding rejected deals.

**Design:**
```typescript
// packages/db/src/schema/deals.ts
export const deals = pgTable('deals', {
  id: uuid('id').primaryKey().defaultRandom(),
  organisationId: uuid('organisation_id').notNull().references(() => organisations.id, { onDelete: 'cascade' }),
  partnerId: uuid('partner_id').notNull().references(() => partners.id),
  registeredBy: uuid('registered_by').notNull().references(() => users.id),
  dealNumber: varchar('deal_number', { length: 50 }).notNull(),
  customerName: varchar('customer_name', { length: 255 }).notNull(),
  customerEmail: varchar('customer_email', { length: 255 }),
  customerCompany: varchar('customer_company', { length: 255 }),
  customerCountry: char('customer_country', { length: 2 }),
  estimatedValue: numeric('estimated_value', { precision: 15, scale: 2 }),
  currency: char('currency', { length: 3 }).notNull().default('USD'),
  expectedCloseDate: date('expected_close_date'),
  actualCloseDate: date('actual_close_date'),
  stage: varchar('stage', { length: 50 }).notNull().default('submitted'),
  approvalStatus: varchar('approval_status', { length: 30 }).notNull().default('pending'),
  approvedBy: uuid('approved_by').references(() => users.id),
  approvedAt: timestamp('approved_at', { withTimezone: true }),
  rejectionReason: text('rejection_reason'),
  expiryDate: date('expiry_date'),
  coSellType: varchar('co_sell_type', { length: 50 }),
  coSellRef: varchar('co_sell_ref', { length: 255 }),
  crmOpportunityId: varchar('crm_opportunity_id', { length: 255 }),
  customFields: jsonb('custom_fields').notNull().default({}),
  notes: text('notes'),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
});

// Duplicate detection service:
// apps/api/src/services/deal-dedup.ts
export async function checkDuplicateDeals(db: DrizzleClient, orgId: string, customerEmail: string, customerCompany: string): Promise<Deal[]> {
  return db.select().from(deals)
    .where(and(
      eq(deals.organisationId, orgId),
      ne(deals.approvalStatus, 'rejected'),
      or(
        eq(deals.customerEmail, customerEmail.toLowerCase()),
        eq(sql`lower(${deals.customerCompany})`, customerCompany.toLowerCase())
      )
    ));
}
```

**Testing:**
- Registering a deal with a unique customer email and company succeeds (201)
- Registering a deal with an email matching an existing non-rejected deal returns a duplicate warning in the response (but still allows creation with a `duplicate_acknowledged` flag)
- Registering a deal with a company name matching an existing deal (case-insensitive) triggers duplicate detection
- Rejected deals are excluded from duplicate detection
- Deal numbers are auto-generated in the format `DR-YYYY-NNNNN` and are unique within an organisation
- Deal number sequence is monotonically increasing per organisation

---

### Task 3.2: Deal Registration API & Approval Workflow

**What:** API endpoints for deal CRUD, stage transitions, and approval/rejection workflow. Deals follow a state machine: submitted -> under_review -> approved/rejected. Approved deals then progress through pipeline stages: approved -> won/lost/expired.

**Design:**
```typescript
// apps/api/src/routes/deals/deals.ts
// POST   /deals                  -- register new deal (partner or vendor)
// GET    /deals                  -- list deals with filters (partner, stage, status, date range)
// GET    /deals/:id              -- get deal detail with activity history
// PATCH  /deals/:id              -- update deal details
// POST   /deals/:id/approve      -- approve deal registration (vendor only)
// POST   /deals/:id/reject       -- reject deal registration (vendor only)
// POST   /deals/:id/stage        -- advance deal stage (approved -> won/lost/expired)

// State machine:
// submitted -> under_review -> approved -> won
//                           -> approved -> lost
//                           -> approved -> expired (automatic, via cron job)
//           -> under_review -> rejected

// apps/api/src/services/deal-lifecycle.ts
export const VALID_TRANSITIONS: Record<string, string[]> = {
  submitted: ['under_review'],
  under_review: ['approved', 'rejected'],
  approved: ['won', 'lost', 'expired'],
  // terminal states: won, lost, rejected, expired (no transitions out)
};

export function validateTransition(currentStage: string, targetStage: string): boolean {
  return VALID_TRANSITIONS[currentStage]?.includes(targetStage) ?? false;
}
```

**Testing:**
- Register deal as partner user returns 201 with deal_number and stage='submitted'
- Register deal as vendor admin returns 201
- Approve deal sets approval_status='approved', approved_by, approved_at, and calculates expiry_date based on org settings
- Reject deal sets approval_status='rejected' and requires rejection_reason
- Invalid stage transition (e.g. submitted -> won) returns 400 with error message
- Deal activities table records every stage change with actor, timestamp, and previous/new values
- List deals with `?stage=approved&partner_id=...` returns filtered results
- Deal expiry cron job transitions approved deals past expiry_date to 'expired' stage
- Expired deals are flagged in the list view
- Concurrent approval attempts are handled (optimistic locking via `updated_at`)

---

### Task 3.3: Deal Pipeline UI (Vendor & Partner)

**What:** Vendor admin deal management page and partner-facing deal registration form. Kanban board view for deal pipeline. Deal detail page with activity timeline.

**Design:**
```
# Vendor admin views
(vendor)/deals/page.tsx             -- Deal list with DataTable + Kanban toggle
(vendor)/deals/[id]/page.tsx        -- Deal detail: customer info, partner, stage, activities
(vendor)/deals/[id]/approve/        -- Approval dialog with notes field

# Partner portal views (built in Phase 6, wireframed here)
(portal)/deals/page.tsx             -- Partner's own deals list
(portal)/deals/new/page.tsx         -- Deal registration form with duplicate detection UI
(portal)/deals/[id]/page.tsx        -- Partner's deal detail (read-only except notes)
```

**Testing:**
- Deal list table shows correct columns: deal number, customer, partner, value, stage, registered date
- Kanban board groups deals by stage with correct counts
- Dragging a deal card in Kanban (where permitted) triggers a stage transition API call
- Approval dialog requires notes; approval posts to API and updates UI without full page reload
- Deal detail page shows activity timeline in reverse chronological order
- Duplicate detection warning appears in the deal registration form when matches are found
- Playwright E2E: register a deal -> approve it -> mark as won; verify all stage transitions

---

## Phase 4: MDF Management

**Goal:** Complete MDF lifecycle from budget allocation through request, approval, claim submission, and proof-of-performance.

**Duration:** 2-3 weeks

**Dependencies:** Phase 2 complete (partners exist for MDF allocation).

### Task 4.1: MDF Budget, Request, and Claim Schema

**What:** Create `mdf_budgets`, `mdf_requests`, `mdf_claims`, and `mdf_claim_documents` tables. Budget allocation per partner per fiscal quarter. Requests link to budgets.

**Design:**
```typescript
// packages/db/src/schema/mdf.ts
// Tables follow data-model-suggestion-1.md MDF section:
// mdf_budgets: org-wide or per-partner, fiscal_year + quarter, total/allocated/spent
// mdf_requests: request_number, title, activity_type, amount, status workflow
// mdf_claims: claimed_amount, proof_of_performance, status, reviewed_by
// mdf_claim_documents: file uploads linked to claims

// MDF request status machine:
// draft -> submitted -> under_review -> approved -> completed
//                    -> under_review -> rejected
//                                    -> cancelled

// MDF claim status machine:
// submitted -> under_review -> approved -> paid
//           -> under_review -> rejected
```

**Testing:**
- Create MDF budget for partner with fiscal_year=2026, quarter=2, total_amount=50000 succeeds
- Create MDF request linking to budget; allocated_amount on budget updates
- Request amount exceeding remaining budget returns 400
- Approve MDF request sets approved_amount (can be less than requested_amount)
- Submit claim against approved request; claimed_amount cannot exceed approved_amount
- Upload proof-of-performance documents (PDF, images) to claim; files stored in S3/MinIO
- Approve claim updates budget's spent_amount
- Mark claim as paid sets paid_at timestamp
- Budget utilisation query returns total/allocated/spent/remaining per partner per quarter

---

### Task 4.2: MDF API Endpoints

**What:** REST API for MDF budget management, request lifecycle, claim submission, and document upload.

**Design:**
```typescript
// apps/api/src/routes/mdf/
// Budget endpoints (vendor admin only):
// POST   /mdf/budgets                    -- allocate budget
// GET    /mdf/budgets                    -- list budgets with filters
// PATCH  /mdf/budgets/:id               -- adjust budget

// Request endpoints (partner + vendor):
// POST   /mdf/requests                  -- create MDF request (partner)
// GET    /mdf/requests                  -- list requests
// PATCH  /mdf/requests/:id             -- update request
// POST   /mdf/requests/:id/submit      -- submit for review (partner)
// POST   /mdf/requests/:id/approve     -- approve request (vendor)
// POST   /mdf/requests/:id/reject      -- reject request (vendor)

// Claim endpoints (partner + vendor):
// POST   /mdf/requests/:id/claims      -- submit claim with proof documents
// POST   /mdf/claims/:id/documents     -- upload proof-of-performance document
// POST   /mdf/claims/:id/approve       -- approve claim (vendor)
// POST   /mdf/claims/:id/reject        -- reject claim (vendor)
// POST   /mdf/claims/:id/pay           -- mark claim as paid (vendor)
```

**Testing:**
- Full MDF lifecycle: allocate budget -> create request -> submit -> approve -> submit claim -> upload proof -> approve claim -> mark paid
- Budget remaining amount decreases correctly at each step
- Cannot submit request against a fully allocated budget
- Claim proof documents accept PDF, PNG, JPG, CSV file types
- Document upload to MinIO/S3 returns a presigned URL
- Rejecting a claim returns the allocated amount to the budget
- Cancelling a request returns the allocated amount to the budget
- MDF request and claim numbers are auto-generated (MDF-YYYY-NNNNN, CLM-YYYY-NNNNN)

---

### Task 4.3: MDF Management UI

**What:** Vendor admin MDF dashboard showing budget utilisation, pending requests, and claims. Partner-facing MDF request form.

**Design:**
```
(vendor)/mdf/page.tsx               -- MDF dashboard: budget overview, utilisation charts
(vendor)/mdf/budgets/page.tsx       -- Budget allocation management
(vendor)/mdf/requests/page.tsx      -- Request list with filters and approval actions
(vendor)/mdf/claims/page.tsx        -- Claims list with proof review and approval
(portal)/mdf/page.tsx              -- Partner's MDF: available budget, requests, claims
(portal)/mdf/new/page.tsx          -- MDF request submission form
(portal)/mdf/[id]/claim/page.tsx   -- Claim submission with document upload
```

**Testing:**
- Budget utilisation chart shows correct percentages (allocated/total)
- Request list filters by status, partner, and date range
- Approval action opens a dialog with approved_amount field (defaults to requested_amount)
- Claim review page displays uploaded proof documents with image preview and PDF viewer
- Playwright E2E: partner submits MDF request -> vendor approves -> partner submits claim with proof -> vendor approves claim -> vendor marks paid

---

## Phase 5: Training & Certification

**Goal:** Partner training module management with SCORM-compatible content, certification paths, and progress tracking.

**Duration:** 2-3 weeks

**Dependencies:** Phase 2 complete (partner contacts exist for training progress).

### Task 5.1: Training & Certification Schema

**What:** Create `training_modules`, `certifications`, `certification_modules` (junction), `training_progress`, and `certification_awards` tables.

**Design:**
```typescript
// packages/db/src/schema/training.ts
// Follow data-model-suggestion-1.md Training & Certification section.
// training_modules: title, module_type (scorm/video/document/quiz/external_link),
//   scorm_package_url, content_url, duration_minutes, is_required, status
// certifications: name, validity_months, required_modules count, passing_score
// certification_modules: junction linking certification to required/optional modules
// training_progress: per partner_contact per module status tracking, SCORM data in JSONB
// certification_awards: per partner_contact, awarded_at, expires_at, status
```

**Testing:**
- Create training module with type 'scorm' and package URL succeeds
- Create certification requiring modules A, B, C with passing_score=80
- Record training progress: not_started -> in_progress -> completed with score=85
- Completing all required modules with passing scores auto-awards certification
- Failing a module (score below passing_score) sets status='failed', does not award certification
- Certification with validity_months=12 sets expires_at to 12 months from awarded_at
- Expired certifications are flagged by a cron job
- SCORM suspend data is persisted in JSONB and returned on next module load

---

### Task 5.2: Training API & SCORM Integration

**What:** API endpoints for training module management, SCORM package upload and launch URL generation, progress tracking, and certification management.

**Design:**
```typescript
// apps/api/src/routes/training/
// Module management (vendor admin):
// POST   /training/modules                -- create module
// GET    /training/modules                -- list modules
// PATCH  /training/modules/:id           -- update module
// POST   /training/modules/:id/scorm     -- upload SCORM package to S3

// Certification management (vendor admin):
// POST   /training/certifications                     -- create certification
// GET    /training/certifications                     -- list certifications
// PATCH  /training/certifications/:id                -- update certification
// POST   /training/certifications/:id/modules        -- add module to certification

// Progress tracking (partner portal):
// GET    /training/my/modules                        -- list available modules for current partner
// POST   /training/modules/:id/launch                -- get SCORM launch URL
// POST   /training/modules/:id/progress              -- update progress (SCORM API endpoint)
// GET    /training/my/certifications                 -- list certification status

// Admin views:
// GET    /training/partners/:partnerId/progress      -- partner's training overview
```

**Testing:**
- Upload SCORM 1.2 package (ZIP); API extracts manifest and stores package in S3
- Launch URL returns a signed URL to the SCORM player with correct parameters
- SCORM progress API accepts `cmi.core.lesson_status`, `cmi.core.score.raw`, `cmi.suspend_data`
- Completing a module updates the certification progress calculation
- Partner training overview returns modules with status and completion percentage
- Certification awards API returns active and expired certifications per contact

---

### Task 5.3: Training & Certification UI

**What:** Vendor admin training management pages and partner-facing training portal.

**Design:**
```
(vendor)/training/modules/page.tsx         -- Module list, SCORM upload
(vendor)/training/certifications/page.tsx  -- Certification paths management
(vendor)/training/partners/[id]/page.tsx   -- Partner training progress overview

(portal)/training/page.tsx                 -- Available modules and certifications
(portal)/training/[moduleId]/page.tsx      -- Module launch page (SCORM player embedded)
(portal)/training/certifications/page.tsx  -- Certification progress and badges
```

**Testing:**
- Module list displays correct status badges (draft/published/archived)
- SCORM package upload shows progress bar and success/error state
- Certification path page shows modules with completion checkmarks
- SCORM player launches in an iframe and communicates progress via postMessage API
- Progress bar on certification page updates after module completion
- Certificate badge displays after all requirements are met
- Playwright E2E: publish module -> partner launches module -> complete with passing score -> verify certification awarded

---

## Phase 6: Partner Portal

**Goal:** Unified partner-facing portal with self-service deal registration, MDF, training, content access, and dashboard.

**Duration:** 3-4 weeks

**Dependencies:** Phases 2, 3, 4, 5 complete.

### Task 6.1: Portal Authentication & Partner Routing

**What:** Separate authentication flow for partner portal users. Portal routes use the partner's organisation context. Support both email/password login and SSO (SAML 2.0/OIDC) for partner organisations.

**Design:**
```typescript
// apps/web/src/app/(portal)/layout.tsx
// - Detects partner context from session
// - Loads partner's branding (logo, colors) from organisation settings
// - Restricts navigation to partner-scoped views

// SSO configuration per partner (in integrations table):
// Provider: saml2 or oidc
// Config: { idp_entity_id, idp_sso_url, idp_certificate, sp_entity_id }

// apps/api/src/routes/auth/sso.ts
// GET  /auth/sso/:partnerId/login   -- initiate SSO flow
// POST /auth/sso/callback           -- handle SAML/OIDC callback
```

**Testing:**
- Partner portal user logs in with email/password and sees only their partner's data
- Partner portal user cannot access vendor admin routes
- SSO login initiates SAML AuthnRequest to configured IdP
- SSO callback creates/updates user session and links to partner contact record
- Portal loads partner's branding (logo, primary color) from organisation settings
- Unauthenticated access to portal routes redirects to login page
- Partner user with expired session is redirected to re-authenticate

---

### Task 6.2: Partner Dashboard

**What:** Main dashboard page for partner portal users showing deal pipeline summary, MDF balance, training progress, and recent activity.

**Design:**
```typescript
// apps/web/src/app/(portal)/dashboard/page.tsx
// Dashboard widgets:
// 1. Deal Pipeline Summary: submitted/approved/won counts + total pipeline value
// 2. MDF Balance: remaining budget this quarter, pending claims
// 3. Training Progress: certifications earned/in-progress, upcoming expirations
// 4. Recent Activity: last 10 activities related to this partner
// 5. Quick Actions: Register Deal, Submit MDF Request, Start Training

// API endpoint:
// GET /portal/dashboard  -- returns aggregated dashboard data for authenticated partner
```

**Testing:**
- Dashboard loads within 2 seconds for a partner with 50 deals, 10 MDF requests, 5 certifications
- Deal pipeline widget shows correct counts per stage
- MDF balance widget shows remaining = total - allocated for current quarter
- Training widget shows progress bars for in-progress certifications
- Activity feed shows the 10 most recent activities with correct timestamps
- Quick action buttons navigate to the correct forms
- Dashboard data refreshes when the user returns to the page (TanStack Query stale-while-revalidate)

---

### Task 6.3: Content Library (Portal)

**What:** Partner-facing content library with role-based access control. Partners see only content assets visible to their tier and program. Search and filter by folder, type, and tags.

**Design:**
```typescript
// packages/db/src/schema/content.ts
export const contentAssets = pgTable('content_assets', {
  id: uuid('id').primaryKey().defaultRandom(),
  organisationId: uuid('organisation_id').notNull().references(() => organisations.id, { onDelete: 'cascade' }),
  folderId: uuid('folder_id').references(() => contentFolders.id),
  title: varchar('title', { length: 255 }).notNull(),
  description: text('description'),
  assetType: varchar('asset_type', { length: 50 }).notNull(),
  fileUrl: text('file_url'),
  fileSizeBytes: bigint('file_size_bytes', { mode: 'number' }),
  mimeType: varchar('mime_type', { length: 100 }),
  isCoBrandable: boolean('is_co_brandable').notNull().default(false),
  tags: text('tags').array(),
  status: varchar('status', { length: 20 }).notNull().default('published'),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
});

export const contentAccessRules = pgTable('content_access_rules', {
  id: uuid('id').primaryKey().defaultRandom(),
  assetId: uuid('asset_id').notNull().references(() => contentAssets.id, { onDelete: 'cascade' }),
  tierId: uuid('tier_id').references(() => partnerTiers.id),
  programId: uuid('program_id').references(() => partnerPrograms.id),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
});

// Access control logic:
// 1. If no access rules exist for an asset -> visible to all partners
// 2. If access rules exist -> partner must match at least one rule's tier or program
```

**Testing:**
- Vendor admin uploads content asset with file to S3; returns 201 with signed download URL
- Vendor admin sets access rules: asset visible only to Gold-tier resellers
- Gold-tier reseller partner sees the asset in their content library
- Silver-tier reseller partner does not see the asset
- Content library search by title and tags returns matching results
- Folder navigation works (nested folders up to 5 levels)
- Download generates a time-limited presigned S3 URL
- Co-brandable assets are flagged in the UI

---

### Task 6.4: Portal Branding & Customisation

**What:** Organisation-level portal branding: custom logo, primary/secondary colours, custom welcome message. Applied via CSS custom properties derived from organisation settings.

**Design:**
```typescript
// Branding settings stored in organisations.settings JSONB:
// {
//   "branding": {
//     "logo_url": "https://...",
//     "favicon_url": "https://...",
//     "primary_color": "#1a73e8",
//     "secondary_color": "#ea4335",
//     "welcome_message": "Welcome to the Acme Partner Portal"
//   }
// }

// apps/web/src/app/(portal)/layout.tsx
// Reads branding from API, applies as CSS custom properties:
// --prm-primary: #1a73e8;
// --prm-secondary: #ea4335;
// shadcn/ui components reference these variables via Tailwind config
```

**Testing:**
- Portal renders with organisation's logo in the header
- Primary colour is applied to buttons, links, and active navigation
- Default branding is applied when no custom branding is configured
- Branding changes are reflected immediately after saving (no cache stale data)
- Branding preview in vendor admin settings page shows live preview

---

## Phase 7: CRM Integration

**Goal:** Bidirectional sync with Salesforce and HubSpot CRMs for deal and partner data.

**Duration:** 3-4 weeks

**Dependencies:** Phase 6 complete (partner portal provides the operational context for sync).

### Task 7.1: Integration Framework

**What:** Create the unified `integrations` table and an abstract integration provider interface. Implement OAuth 2.0 flow for CRM connection. Store tokens securely (encrypted at rest).

**Design:**
```typescript
// packages/db/src/schema/integrations.ts
export const integrations = pgTable('integrations', {
  id: uuid('id').primaryKey().defaultRandom(),
  organisationId: uuid('organisation_id').notNull().references(() => organisations.id, { onDelete: 'cascade' }),
  provider: varchar('provider', { length: 50 }).notNull(),
  category: varchar('category', { length: 30 }).notNull(),
  isEnabled: boolean('is_enabled').notNull().default(true),
  config: jsonb('config').notNull().default({}),
  credentials: jsonb('credentials').notNull().default({}),   // AES-256 encrypted
  lastSyncAt: timestamp('last_sync_at', { withTimezone: true }),
  syncStatus: varchar('sync_status', { length: 30 }),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
});

// apps/api/src/services/integrations/provider.ts
export interface IntegrationProvider {
  connect(orgId: string): Promise<{ authUrl: string }>;
  handleCallback(orgId: string, code: string): Promise<void>;
  syncDeals(orgId: string): Promise<SyncResult>;
  syncPartners(orgId: string): Promise<SyncResult>;
  disconnect(orgId: string): Promise<void>;
}
```

**Testing:**
- OAuth 2.0 connection flow redirects to Salesforce/HubSpot auth page
- Callback stores encrypted access_token and refresh_token
- Token refresh occurs automatically when access_token expires
- Disconnecting removes credentials and marks integration as disabled
- Integration status is visible on the vendor admin integrations page

---

### Task 7.2: Salesforce CRM Sync

**What:** Bidirectional sync between PRM deals and Salesforce Opportunities. Map PRM partner records to Salesforce Account/Partner objects. Configurable field mapping.

**Design:**
```typescript
// apps/api/src/services/integrations/salesforce.ts
// Sync logic:
// 1. On deal creation in PRM -> create/update Salesforce Opportunity
// 2. On deal stage change in PRM -> update Salesforce Opportunity stage
// 3. On Salesforce Opportunity update -> update PRM deal (via webhook or polling)
// 4. Field mapping stored in integration config JSONB:
//    { "deal.estimated_value": "Opportunity.Amount",
//      "deal.stage": "Opportunity.StageName",
//      "deal.customer_company": "Opportunity.Account.Name" }
//
// BullMQ job: 'salesforce-sync' runs every 5 minutes for each connected org
// Conflict resolution: last-write-wins with audit log entry noting the conflict
```

**Testing:**
- Creating a deal in PRM creates a corresponding Salesforce Opportunity
- Updating a deal stage in PRM updates the Salesforce Opportunity StageName
- Salesforce Opportunity amount change syncs back to PRM deal.estimated_value
- Field mapping configuration determines which fields sync in each direction
- Sync handles API rate limits (429 responses) with exponential backoff
- Sync handles Salesforce token expiry by refreshing before retry
- Sync conflict is logged in audit_log with both old and new values
- Sync status dashboard shows last sync time, records synced, and errors

---

### Task 7.3: HubSpot CRM Sync

**What:** Same bidirectional sync for HubSpot CRM. Map PRM deals to HubSpot Deals, PRM partners to HubSpot Companies.

**Design:**
```typescript
// apps/api/src/services/integrations/hubspot.ts
// Uses HubSpot CRM API v3: https://developers.hubspot.com/docs/reference/api
// Maps: PRM Deal -> HubSpot Deal, PRM Partner -> HubSpot Company
// Uses HubSpot webhook subscriptions for real-time inbound sync
// Configurable field mapping same pattern as Salesforce
```

**Testing:**
- OAuth 2.0 connection with HubSpot succeeds
- PRM deal creation creates HubSpot Deal with correct properties
- HubSpot Deal property change triggers webhook that updates PRM deal
- HubSpot API rate limits are handled with retry and backoff
- Field mapping UI allows mapping PRM fields to HubSpot properties

---

### Task 7.4: Integration Management UI

**What:** Vendor admin integration settings page for connecting CRMs, configuring field mappings, and monitoring sync status.

**Design:**
```
(vendor)/integrations/page.tsx              -- Integration catalog with connect buttons
(vendor)/integrations/[id]/page.tsx         -- Integration detail: field mapping, sync history
(vendor)/integrations/[id]/mapping/page.tsx -- Drag-and-drop field mapping editor
```

**Testing:**
- Integration catalog shows available integrations with connection status
- Connect button initiates OAuth flow and redirects to provider
- After successful connection, integration shows as 'Connected' with last sync time
- Field mapping editor shows PRM fields on the left and CRM fields on the right
- Saving field mapping updates the integration config
- Sync history table shows last 50 sync operations with status and record counts

---

## Phase 8: Notifications & Activity Feed

**Goal:** Multi-channel notifications (in-app, email, Slack, Teams) and a unified activity feed.

**Duration:** 2-3 weeks

**Dependencies:** Phase 6 complete.

### Task 8.1: Notification System

**What:** Create the `notifications` table. Implement a notification service that dispatches via multiple channels based on user preferences and organisation configuration.

**Design:**
```typescript
// packages/db/src/schema/notifications.ts
export const notifications = pgTable('notifications', {
  id: uuid('id').primaryKey().defaultRandom(),
  organisationId: uuid('organisation_id').notNull().references(() => organisations.id, { onDelete: 'cascade' }),
  userId: uuid('user_id').notNull().references(() => users.id),
  title: varchar('title', { length: 255 }).notNull(),
  body: text('body'),
  notificationType: varchar('notification_type', { length: 50 }).notNull(),
  entityType: varchar('entity_type', { length: 50 }),
  entityId: uuid('entity_id'),
  isRead: boolean('is_read').notNull().default(false),
  readAt: timestamp('read_at', { withTimezone: true }),
  channel: varchar('channel', { length: 20 }).notNull().default('in_app'),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
});

// apps/api/src/services/notifications.ts
// Notification events:
// - deal.registered     -> notify vendor admin
// - deal.approved       -> notify partner contact
// - deal.rejected       -> notify partner contact
// - mdf.submitted       -> notify vendor admin
// - mdf.approved        -> notify partner contact
// - mdf.claim_approved  -> notify partner contact
// - training.completed  -> notify partner contact + vendor admin
// - partner.tier_changed -> notify partner contacts
//
// Dispatch channels:
// - in_app: insert into notifications table + real-time via WebSocket/SSE
// - email: queue email via BullMQ + React Email templates
// - slack: POST to Slack webhook URL from org settings
// - teams: POST to Teams webhook URL from org settings
```

**Testing:**
- Deal approved event creates in-app notification for the registering partner contact
- In-app notification is retrievable via GET /notifications API
- Mark notification as read updates `is_read` and `read_at`
- Email notification sends formatted email using React Email template
- Slack notification posts to configured webhook with correct message format
- Teams notification posts to configured webhook with Adaptive Card format
- Notification preferences allow users to opt out of specific channels per notification type
- Notification bell in the UI shows unread count badge

---

### Task 8.2: Webhook System

**What:** Outbound webhook delivery for external integrations. Organisations can subscribe to event types and receive signed webhook payloads.

**Design:**
```typescript
// packages/db/src/schema/notifications.ts (continued)
export const webhookSubscriptions = pgTable('webhook_subscriptions', {
  id: uuid('id').primaryKey().defaultRandom(),
  organisationId: uuid('organisation_id').notNull().references(() => organisations.id, { onDelete: 'cascade' }),
  url: text('url').notNull(),
  secret: text('secret').notNull(),         // HMAC-SHA256 signing key
  eventTypes: text('event_types').array().notNull(),
  isActive: boolean('is_active').notNull().default(true),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
});

// Delivery follows Standard Webhooks specification:
// - HMAC-SHA256 signature in Webhook-Signature header
// - Webhook-Id and Webhook-Timestamp headers
// - Retry with exponential backoff (5 attempts: 30s, 2m, 15m, 1h, 6h)
// - Dead letter after max retries
```

**Testing:**
- Create webhook subscription for `deal.*` event types
- When a deal is created, webhook POST is sent to subscriber URL
- Webhook payload includes `Webhook-Id`, `Webhook-Timestamp`, and `Webhook-Signature` headers
- Signature verification succeeds with the subscription secret
- Failed delivery (non-2xx response) triggers retry with exponential backoff
- After 5 failed attempts, delivery is moved to dead letter queue
- Webhook delivery log shows attempt count, response status, and timing
- Disabling a subscription stops future deliveries

---

### Task 8.3: Notification & Webhook UI

**What:** In-app notification center, notification preference management, and webhook subscription management for vendor admins.

**Design:**
```
(vendor)/settings/notifications/page.tsx    -- Notification preferences per event type
(vendor)/settings/webhooks/page.tsx         -- Webhook subscription management
(vendor)/settings/webhooks/[id]/page.tsx    -- Webhook detail with delivery log
```

**Testing:**
- Notification center dropdown shows unread notifications with dismiss action
- Notification preferences page shows a matrix of event types vs channels (email, Slack, Teams)
- Toggling a preference updates immediately
- Webhook management page shows active subscriptions with endpoint URL and event types
- Create webhook subscription form validates URL format and generates a secret
- Webhook detail page shows delivery history with response codes and retry status

---

## Phase 9: Commissions & Payouts

**Goal:** Commission plan management, automated commission calculation on deal close, and payout tracking.

**Duration:** 2-3 weeks

**Dependencies:** Phase 3 complete (deals with stage='won' trigger commission calculation).

### Task 9.1: Commission Plans & Calculation

**What:** Create `commission_plans`, `commission_tiers`, and `commissions` tables. Implement commission calculation engine supporting flat rate, percentage, and tiered progressive/retroactive models.

**Design:**
```typescript
// packages/db/src/schema/commissions.ts
// Follow data-model-suggestion-1.md Commissions section.
// commission_plans: plan_type (flat_rate, percentage, tiered_progressive, tiered_retroactive),
//   base_rate, clawback_days, is_recurring
// commission_tiers: per plan, min/max revenue thresholds with rates
// commissions: per partner per deal, amount, status (pending/approved/paid/clawed_back),
//   clawback_of self-reference

// apps/api/src/services/commission-engine.ts
export function calculateCommission(plan: CommissionPlan, tiers: CommissionTier[], dealValue: number): number {
  switch (plan.planType) {
    case 'flat_rate':
      return plan.baseRate;
    case 'percentage':
      return dealValue * plan.baseRate;
    case 'tiered_progressive':
      // Each tier applies only to revenue within its range
      return tiers.reduce((total, tier) => {
        const applicable = Math.min(dealValue, tier.maxRevenue ?? Infinity) - tier.minRevenue;
        return total + (applicable > 0 ? applicable * tier.rate : 0);
      }, 0);
    case 'tiered_retroactive':
      // Highest qualifying tier rate applies to entire deal value
      const qualifyingTier = tiers.filter(t => dealValue >= t.minRevenue).sort((a, b) => b.minRevenue - a.minRevenue)[0];
      return dealValue * (qualifyingTier?.rate ?? 0);
  }
}

// BullMQ job: when a deal transitions to 'won', calculate and record commission
// BullMQ cron: clawback check -- if a deal's customer churns within clawback_days,
//   create a negative commission entry with clawback_of pointing to the original
```

**Testing:**
- Flat rate commission: plan with base_rate=500, deal won -> commission=500
- Percentage commission: plan with base_rate=0.10, deal value=85000 -> commission=8500
- Tiered progressive: tiers [0-50K@8%, 50K-100K@10%, 100K+@12%], deal=85000 -> commission=(50000*0.08)+(35000*0.10)=7500
- Tiered retroactive: same tiers, deal=85000 -> highest qualifying tier is 50K-100K@10% -> commission=85000*0.10=8500
- Commission entry created with status='pending' on deal won
- Commission approved sets status='approved'
- Commission paid sets status='paid' and paid_at timestamp
- Clawback: deal customer churns within 90 days -> negative commission entry created with clawback_of referencing original
- Recurring commission: if is_recurring=true, commission calculated on each renewal deal

---

### Task 9.2: Commission API & Reports

**What:** API endpoints for commission plan management, commission approval, payout tracking, and reporting.

**Design:**
```typescript
// apps/api/src/routes/commissions/
// Plan management (vendor admin):
// POST   /commissions/plans                -- create plan
// GET    /commissions/plans                -- list plans
// PATCH  /commissions/plans/:id           -- update plan
// POST   /commissions/plans/:id/tiers     -- add tier to plan

// Commission records:
// GET    /commissions                      -- list commissions with filters
// POST   /commissions/:id/approve         -- approve commission (vendor)
// POST   /commissions/:id/pay            -- mark as paid (vendor)

// Reports:
// GET    /commissions/report/partner/:id  -- partner commission statement
// GET    /commissions/report/summary      -- org-wide commission summary
```

**Testing:**
- Commission plan creation with tiered_progressive type and 3 tiers succeeds
- Listing commissions with `?partner_id=...&status=pending` returns filtered results
- Partner commission statement shows: earned, paid, clawed back, and balance due
- Org-wide summary shows total commissions by period, by partner, by plan
- Commission approval requires vendor admin role
- Commission payout API validates that commission is in 'approved' status

---

### Task 9.3: Commission UI

**What:** Vendor admin commission management and partner-facing commission dashboard.

**Design:**
```
(vendor)/commissions/page.tsx               -- Commission overview: pending, approved, paid totals
(vendor)/commissions/plans/page.tsx         -- Commission plan management with tier editor
(vendor)/commissions/payouts/page.tsx       -- Payout queue: approve and mark paid
(portal)/commissions/page.tsx               -- Partner's commission statement and history
```

**Testing:**
- Commission plan editor allows adding/removing tiers with visual rate bracket display
- Payout queue shows pending commissions sortable by amount and partner
- Batch approve action processes multiple commissions at once
- Partner commission statement shows a ledger view with earned, clawback, and payout entries
- Playwright E2E: create plan -> win deal -> verify commission calculated -> approve -> mark paid

---

## Phase 10: Co-Sell Integration

**Goal:** Integration with cloud hyperscaler co-sell programs (AWS ACE, Microsoft Partner Center, Google Cloud Partner Advantage).

**Duration:** 3-4 weeks

**Dependencies:** Phase 3 complete (deals exist to link with co-sell opportunities).

### Task 10.1: Co-Sell Integration Framework

**What:** Create the `co_sell_integrations` table (or entries in the unified `integrations` table with category='co_sell'). Abstract co-sell provider interface for AWS ACE, Microsoft Partner Center, and Google CPA APIs.

**Design:**
```typescript
// apps/api/src/services/integrations/cosell/provider.ts
export interface CoSellProvider {
  name: 'aws_ace' | 'microsoft_partner_center' | 'google_cpa';
  connect(orgId: string, credentials: Record<string, string>): Promise<void>;
  submitOpportunity(orgId: string, deal: Deal): Promise<{ externalRef: string }>;
  syncStatus(orgId: string, externalRef: string): Promise<CoSellStatus>;
  listOpportunities(orgId: string, filters?: CoSellFilters): Promise<CoSellOpportunity[]>;
}

// apps/api/src/services/integrations/cosell/aws-ace.ts
// Uses AWS Partner Central API for ACE (APN Customer Engagements)
// Submits opportunity with customer info, estimated value, and product interest
// Syncs status: prospect -> qualified -> won/lost

// apps/api/src/services/integrations/cosell/microsoft-pc.ts
// Uses Microsoft Partner Center API for co-sell referrals
// Submits referral with customer profile and deal details
// Syncs status: new -> accepted -> won/lost/expired

// apps/api/src/services/integrations/cosell/google-cpa.ts
// Uses Google Cloud Partner Advantage API
// Submits engagement with customer and solution details
// Syncs status: pending -> active -> closed
```

**Testing:**
- AWS ACE integration: submit deal as ACE opportunity; receive external reference ID
- AWS ACE status sync: poll for status changes; update deal co_sell_ref and deal stage
- Microsoft Partner Center: submit referral; receive referral ID; sync status
- Google CPA: submit engagement; receive engagement ID; sync status
- Co-sell status changes trigger notifications to the deal registering partner
- Co-sell integration handles authentication failures gracefully (credential rotation)
- Unified co-sell dashboard shows deals across all three hyperscalers with normalised status

---

### Task 10.2: Co-Sell UI

**What:** Co-sell management page showing unified view of hyperscaler co-sell opportunities.

**Design:**
```
(vendor)/co-sell/page.tsx                   -- Unified co-sell dashboard
(vendor)/co-sell/[id]/page.tsx             -- Co-sell opportunity detail with sync status
(vendor)/integrations/co-sell/page.tsx     -- Co-sell provider connection management
```

**Testing:**
- Unified dashboard shows co-sell opportunities from all connected hyperscalers
- Status filter works across providers (normalised status values)
- Clicking a co-sell opportunity shows deal details with hyperscaler-specific metadata
- Connection management allows configuring credentials per hyperscaler
- Sync status indicator shows last successful sync time per provider

---

## Phase 11: AI-Powered Intelligence

**Goal:** AI-driven deal coaching, partner health scoring, personalised onboarding paths, and predictive analytics.

**Duration:** 4-5 weeks

**Dependencies:** Phases 7 and 9 complete (CRM data and commission data feed AI models).

### Task 11.1: Partner Health Scoring Engine

**What:** Compute a health score (0-100) for each partner based on deal velocity, training completion, MDF utilisation, login frequency, and engagement recency. Store scores on partner records, refreshed daily.

**Design:**
```typescript
// apps/api/src/services/ai/partner-health.ts
export interface HealthScoreInputs {
  dealVelocity: number;           // deals registered in last 90 days / historical average
  winRate: number;                // won deals / total closed deals (last 12 months)
  trainingCompletion: number;     // completed modules / required modules (0-1)
  mdfUtilisation: number;         // MDF spent / MDF allocated current year (0-1)
  daysSinceLastActivity: number;  // days since last deal, MDF, or training activity
  certificationCoverage: number;  // contacts with active certifications / total contacts (0-1)
}

export function computeHealthScore(inputs: HealthScoreInputs): { score: number; risk: string; factors: string[] } {
  // Weighted formula:
  // score = (dealVelocity * 0.25) + (winRate * 0.20) + (trainingCompletion * 0.15)
  //       + (mdfUtilisation * 0.10) + (recencyScore * 0.20) + (certCoverage * 0.10)
  // recencyScore: 100 if < 7 days, decaying to 0 at 120 days
  // risk: >= 70 = low, 40-69 = medium, 20-39 = high, < 20 = critical
}

// BullMQ cron job: daily at 02:00 UTC
// Computes health score for every active partner
// Updates partners.custom_fields with { health_score, churn_risk, score_factors }
// Appends to partner_events (append-only event table for AI analysis):
// { event_type: 'health_score_computed', payload: { score, risk, factors, inputs } }
```

**Testing:**
- Partner with recent deal activity, high win rate, completed training scores > 80
- Partner with no activity in 90+ days, no training completion scores < 30
- Health score computation runs in < 5 seconds for 1000 partners
- Risk level thresholds produce correct classifications
- Score factors array correctly identifies the weakest areas
- Daily cron job updates all partner health scores without errors
- Historical health scores are queryable via partner_events table

---

### Task 11.2: AI Deal Coaching

**What:** At deal registration time, analyse the deal against historical win/loss patterns and provide coaching: win probability estimate, missing qualification fields, and suggested next actions.

**Design:**
```typescript
// apps/api/src/services/ai/deal-coaching.ts
// When a deal is registered (POST /deals), after successful creation:
// 1. Gather historical deal data for this org: win/loss rates by customer country,
//    deal size, product, partner tier, co-sell involvement
// 2. Prompt an LLM (Claude API) with the deal context and historical patterns:
//    "Given these historical deal patterns and this new deal registration,
//     estimate win probability, identify missing qualification information,
//     and suggest next actions for the partner."
// 3. Store coaching response on the deal as deal_activities with type='ai_coaching'
// 4. Return coaching suggestions in the API response

// LLM prompt includes:
// - Deal details (customer, value, product, country)
// - Historical win rate for similar deals
// - Partner's track record (health score, win rate)
// - Missing fields compared to won deals (qualification gaps)
// - Co-sell opportunity (is the customer on a hyperscaler?)
```

**Testing:**
- Deal registration response includes `coaching` field with win_probability, gaps, and suggestions
- Win probability is a number between 0 and 1 with explanation
- Gaps array identifies fields that are present on won deals but missing on this deal
- Suggestions array provides 2-5 actionable next steps
- Coaching is generated within 3 seconds of deal submission
- Coaching is stored in deal_activities for future reference
- If LLM API is unavailable, deal registration still succeeds (coaching is best-effort)
- Coaching suggestions are specific to the deal context (not generic advice)

---

### Task 11.3: Personalised Onboarding Paths

**What:** When a new partner is onboarded, generate a personalised training and enablement sequence based on the partner's business profile, geography, product certifications, and program type.

**Design:**
```typescript
// apps/api/src/services/ai/onboarding.ts
// When a partner transitions to status='onboarding':
// 1. Gather: partner profile (company size, country, vertical), program type, tier
// 2. Gather: available training modules and certifications
// 3. Gather: onboarding patterns from successful partners (partners who reached
//    time-to-first-deal < 60 days)
// 4. Prompt LLM: "Given this partner profile and these available training modules,
//    create a prioritised onboarding sequence that will minimise time-to-first-deal."
// 5. Store recommended sequence on partner as onboarding_plan in custom_fields
// 6. Optionally auto-enrol partner contacts in the recommended modules

// Output format:
// { recommended_modules: [{ id, title, reason, priority }],
//   estimated_days_to_first_deal: number,
//   focus_areas: string[] }
```

**Testing:**
- Partner onboarding generates a personalised module sequence within 5 seconds
- Recommended modules are a subset of the org's published training modules
- Each recommendation includes a reason explaining why it is relevant to this partner
- Partners in different verticals receive different module sequences
- Partners in different countries receive localised content recommendations (where available)
- Estimated time-to-first-deal is derived from historical data of similar partners

---

### Task 11.4: AI Intelligence UI

**What:** Partner health dashboard with churn risk indicators, deal coaching display in deal detail, and onboarding plan visualisation.

**Design:**
```
(vendor)/analytics/health/page.tsx         -- Partner health heatmap: all partners with score/risk
(vendor)/partners/[id]/health/page.tsx     -- Individual partner health detail with score factors
(vendor)/deals/[id] -> coaching panel      -- AI coaching suggestions alongside deal details
(portal)/onboarding/page.tsx               -- Partner's personalised onboarding plan with progress
```

**Testing:**
- Health heatmap colours partners by risk level (green/yellow/orange/red)
- Clicking a partner shows health score breakdown with radar chart of factors
- Deal detail page shows AI coaching panel with win probability and action items
- Onboarding plan page shows recommended modules in priority order with completion status
- All AI-generated content is clearly labelled as AI-generated

---

## Phase 12: MCP Server & API Ecosystem

**Goal:** Expose PRM capabilities via Model Context Protocol (MCP) for AI agent consumption and publish a comprehensive public API with OpenAPI 3.1 documentation.

**Duration:** 3-4 weeks

**Dependencies:** Phase 11 complete (MCP server exposes AI capabilities).

### Task 12.1: MCP Server Implementation

**What:** Implement an MCP server (JSON-RPC 2.0 over stdio/SSE) that exposes PRM resources and tools to AI agents. Agents can query partners, deals, MDF status, and trigger deal registration.

**Design:**
```typescript
// apps/api/src/mcp/server.ts
// MCP server following https://modelcontextprotocol.io/specification/2025-11-25
// Transport: SSE (Server-Sent Events) for web-based agents, stdio for local agents

// Resources (read-only data access):
// - prm://partners              -- list partners with filters
// - prm://partners/{id}         -- partner detail with health score
// - prm://deals                 -- list deals with pipeline status
// - prm://deals/{id}           -- deal detail with coaching suggestions
// - prm://mdf/budgets          -- MDF budget balances
// - prm://training/progress    -- partner training completion

// Tools (actions):
// - register_deal(partner_id, customer_info, deal_details) -- register a new deal
// - approve_deal(deal_id, notes)                           -- approve a deal registration
// - submit_mdf_request(partner_id, request_details)        -- submit MDF request
// - get_partner_health(partner_id)                         -- get health score and risk factors
// - find_co_sell_opportunities(partner_id)                 -- AI-powered co-sell suggestions

// Prompts:
// - deal_coaching(deal_id)      -- get AI coaching for a specific deal
// - partner_analysis(partner_id) -- comprehensive partner performance analysis
// - program_health()            -- overall channel program health summary
```

**Testing:**
- MCP server responds to `initialize` request with capabilities
- `resources/list` returns all available PRM resources
- `resources/read` for `prm://partners` returns partner list in JSON format
- `tools/list` returns all available PRM tools with parameter schemas
- `tools/call` for `register_deal` creates a deal and returns deal_number
- Authentication: MCP server validates API key from MCP client
- Tenant isolation: MCP requests are scoped to the authenticated organisation
- Error handling: invalid tool parameters return structured error per MCP spec
- Rate limiting: MCP requests are subject to the same rate limits as REST API

---

### Task 12.2: Public API & OpenAPI Documentation

**What:** Publish the REST API with versioned URLs (/api/v1/...), OpenAPI 3.1 spec auto-generated from Zod schemas, interactive API documentation, and API key management.

**Design:**
```typescript
// apps/api/src/routes/index.ts
// All routes mounted under /api/v1/
// OpenAPI spec generated via @hono/zod-openapi
// Served at /api/v1/openapi.json

// apps/api/src/routes/admin/api-keys.ts
// POST   /admin/api-keys          -- generate API key for integration
// GET    /admin/api-keys          -- list active API keys
// DELETE /admin/api-keys/:id      -- revoke API key

// API key authentication middleware:
// Bearer token in Authorization header -> look up in api_keys table -> inject org context
```

**Testing:**
- OpenAPI spec is valid according to OpenAPI 3.1.1 specification
- All API endpoints are documented with request/response schemas
- API documentation UI (Scalar or Stoplight Elements) is accessible at /docs
- API key generation returns a key that authenticates subsequent requests
- API key revocation immediately blocks requests with that key
- Rate limiting returns 429 with `Retry-After` header
- Error responses follow RFC 7807 Problem Details format
- SDK auto-generation from OpenAPI spec produces working TypeScript client

---

### Task 12.3: API Ecosystem & Developer Portal

**What:** Developer portal with API documentation, getting started guides, webhook reference, and MCP server documentation. Zapier integration via Zapier CLI app.

**Design:**
```
Developer portal pages (deployed as part of the marketing site or standalone):
- Getting Started guide with authentication and first API call
- API Reference (auto-generated from OpenAPI spec)
- Webhook Event Reference with payload schemas
- MCP Server Documentation with resource and tool descriptions
- SDKs: auto-generated TypeScript, Python client libraries

Zapier integration:
- Triggers: deal.registered, deal.approved, deal.won, mdf.submitted, mdf.approved
- Actions: register_deal, create_partner, submit_mdf_request
- Searches: find_partner, find_deal
```

**Testing:**
- Developer portal loads and renders API documentation correctly
- Getting Started guide code samples execute successfully against the API
- Zapier triggers fire correctly when corresponding events occur in PRM
- Zapier actions create records successfully in PRM
- Auto-generated TypeScript SDK compiles and passes type checking
- Auto-generated Python SDK installs and passes basic integration tests

---

## Definition of Done

Each phase is considered complete when ALL of the following criteria are met:

### Code Quality
- [ ] All code passes TypeScript strict mode type checking (`pnpm turbo typecheck`)
- [ ] All code passes ESLint with project configuration (`pnpm turbo lint`)
- [ ] No `any` types except in explicitly justified cases with inline comments
- [ ] All public functions and API endpoints have JSDoc comments
- [ ] Database schema changes are in Drizzle migration files, not manual SQL

### Testing
- [ ] Unit test coverage >= 80% for business logic (services, commission engine, health scoring)
- [ ] Integration tests cover all API endpoints with success and error cases
- [ ] Database tests run against real PostgreSQL via Testcontainers
- [ ] E2E tests cover the critical user journey for the phase (Playwright)
- [ ] All tests pass in CI (`pnpm turbo test` exits 0)

### Security
- [ ] All API endpoints enforce authentication (except public auth routes)
- [ ] Tenant isolation verified: no cross-org data leakage in any API response
- [ ] Passwords hashed with Argon2id; no plaintext secrets in code or config
- [ ] OAuth tokens and API keys encrypted at rest in the database
- [ ] Input validation via Zod on all API endpoints; no raw user input in SQL queries
- [ ] OWASP API Security Top 10 checklist reviewed for new endpoints

### Documentation
- [ ] API endpoints documented in OpenAPI spec (auto-generated from Zod schemas)
- [ ] Database schema changes documented in migration files with comments
- [ ] README updated if new setup steps are required
- [ ] Architecture Decision Records (ADRs) written for non-obvious technical decisions

### Operations
- [ ] Database migrations are backward-compatible (no destructive changes in production)
- [ ] Background jobs have retry logic and dead letter handling
- [ ] Logging follows structured JSON format with correlation IDs
- [ ] Health check endpoint returns 200 with database and Redis connectivity status
- [ ] OpenTelemetry traces emit for all API requests and background jobs

### Review
- [ ] Code reviewed by at least one other engineer
- [ ] Demo conducted for stakeholders at end of phase
- [ ] Known issues and tech debt documented in backlog
